---
layout: post
title: DB 심화 - 이중 범위 조건, BETWEEN vs LIKE, 선분이력, Access vs Filter Predicate, Index Fragmentation
date: 2025-11-07 22:25:23 +0900
category: DB 심화
---
# 같은 컬럼 이중 범위, `BETWEEN` vs `LIKE`, 선분이력, Access/Filter, Index Fragmentation 재정리

> 핵심 주제  
> - 동일 컬럼에 **중복된 범위 조건**을 사용할 때 옵티마이저가 인덱스 스캔 범위를 어떻게 설정하는지  
> - `BETWEEN`과 `LIKE 'prefix%'`가 **인덱스에서 생성하는 범위**의 차이와 실제 수행 방식  
> - **선분 이력(유효기간) 모델**에서 인덱스를 효과적으로 설계하고 스캔 효율을 높이는 방법  
> - 실행계획에서 **Access Predicate와 Filter Predicate**를 구분하여 읽고, SARGability를 통해 Access 조건으로 끌어올리는 기법  
> - **인덱스 단편화(Index Fragmentation)**가 실제 성능에 미치는 영향과 COALESCE/REBUILD 적용 시기

---

## 0. 실습 환경 설정

```sql
ALTER SESSION SET nls_date_format  = 'YYYY-MM-DD HH24:MI:SS';
ALTER SESSION SET statistics_level = ALL;
```

실행계획 확인을 위한 표준 템플릿:

```sql
SELECT * FROM TABLE(
  DBMS_XPLAN.DISPLAY_CURSOR(
    NULL, NULL,
    'ALLSTATS LAST +PREDICATE +PEEKED_BINDS +ALIAS +IOSTATS +MEMSTATS'
  )
);
```

---

## 1. 동일 컬럼에 중복 범위 조건 사용의 문제점

### 실습 테이블 생성

```sql
DROP TABLE ord PURGE;

CREATE TABLE ord (
  order_id  NUMBER       PRIMARY KEY,
  cust_id   NUMBER       NOT NULL,
  order_dt  DATE         NOT NULL,
  amount    NUMBER(12,2) NOT NULL,
  status    VARCHAR2(8)  NOT NULL
);

BEGIN
  FOR i IN 1..300000 LOOP
    INSERT INTO ord
    VALUES (
      i,
      MOD(i,60000)+1,
      DATE '2024-01-01' + MOD(i, 540), -- 약 18개월 범위 내 순환
      ROUND(DBMS_RANDOM.VALUE(1, 200000), 2),
      CASE MOD(i,5)
        WHEN 0 THEN 'NEW'
        WHEN 1 THEN 'PAID'
        WHEN 2 THEN 'SHIP'
        WHEN 3 THEN 'DONE'
        ELSE 'CANC'
      END
    );
  END LOOP;
  COMMIT;
END;
/

CREATE INDEX ix_ord_dt  ON ord(order_dt, order_id);
CREATE INDEX ix_ord_cdt ON ord(cust_id, order_dt, order_id);

BEGIN
  DBMS_STATS.GATHER_TABLE_STATS(
    USER, 'ORD', cascade=>TRUE,
    method_opt=>'for all columns size skewonly'
  );
END;
```

---

### 교집합(AND) 조건의 비효율성

동일한 `order_dt` 컬럼에 서로 겹치는 두 개의 범위 조건을 AND로 연결하는 것은 흔히 발생하는 실수입니다.

```sql
VAR b1 VARCHAR2(19);
VAR b2 VARCHAR2(19);
EXEC :b1 := '2025-10-01 00:00:00';
EXEC :b2 := '2025-10-31 23:59:59';

SELECT /* A1: BAD - 이중 범위 */
       COUNT(*)
FROM   ord
WHERE  order_dt BETWEEN TO_DATE(:b1,'YYYY-MM-DD HH24:MI:SS')
                   AND TO_DATE(:b2,'YYYY-MM-DD HH24:MI:SS')
AND    order_dt >= DATE '2025-10-15';
```

이 쿼리의 WHERE 절은 논리적으로 다음 두 구간의 교집합을 의미합니다:
- 구간 1: `[:b1, :b2]`
- 구간 2: `[2025-10-15, 무한대)`

실제 교집합은 `[2025-10-15, :b2]`로 단일 범위로 표현 가능합니다. 그러나 옵티마이저가 항상 이렇게 최적화해주지는 않습니다. 특히 바인드 변수와 리터럴이 혼합된 경우, 실행계획에서 다음과 같이 나눠질 수 있습니다:
- Access Predicate: `order_dt BETWEEN :b1 AND :b2`
- Filter Predicate: `order_dt >= DATE '2025-10-15'`

이 경우 인덱스 스캔은 `[:b1, :b2]`의 넓은 범위를 모두 읽은 후, 필터 조건으로 10월 15일 이전 데이터를 제거하는 비효율이 발생합니다.

#### 올바른 단일 범위 정규화

```sql
SELECT /* A1 GOOD */
       COUNT(*)
FROM   ord
WHERE  order_dt >= DATE '2025-10-15'
AND    order_dt <  DATE '2025-11-01';   -- 좌포우개 패턴
```

날짜 범위 조건은 `BETWEEN` 대신 **좌측 포함/우측 배제(Left-Inclusive, Right-Exclusive)** 패턴을 사용하는 것이 안전하고 효율적입니다. 이 패턴은 시분초 경계 문제를 방지하면서도 옵티마이저가 단일 범위 스캔을 쉽게 계획할 수 있게 합니다.

---

### 합집합(OR) 조건의 확장성 문제

```sql
SELECT /* A2: 10월 OR 12월 */
       COUNT(*)
FROM   ord
WHERE  (order_dt BETWEEN DATE '2025-10-01'
                     AND DATE '2025-10-31' + (1 - 1/86400))
   OR  (order_dt BETWEEN DATE '2025-12-01'
                     AND DATE '2025-12-31' + (1 - 1/86400));
```

옵티마이저는 일반적으로 두 개의 `INDEX RANGE SCAN`을 수행한 후 `BITMAP OR`이나 `CONCATENATION` 연산으로 결과를 합칩니다. 2-3개의 기간은 괜찮지만, 5-10개 이상의 기간이 OR로 연결되면 SQL 문장이 복잡해지고 파싱 오버헤드가 증가하며, 카디널리티 추정이 어려워집니다.

#### 확장 가능한 해결책: 목록 테이블 조인

```sql
WITH M AS (
  SELECT DATE '2025-10-01' AS base_dt FROM dual
  UNION ALL
  SELECT DATE '2025-12-01' FROM dual
)
SELECT /* A2 GOOD */
       COUNT(*)
FROM   ord o
JOIN   M   m
ON     o.order_dt >= m.base_dt
   AND o.order_dt <  ADD_MONTHS(m.base_dt,1);
```

이 패턴의 장점:
1. 기간이 추가될 때 M 서브쿼리만 수정하면 됨
2. 옵티마이저가 M을 작은 드라이빙 집합으로 인식하여 각 기간별로 짧은 범위 스캔 반복 수행
3. 파싱 계획 재사용성 향상

---

## 2. `BETWEEN`과 `LIKE`의 인덱스 범위 비교

### 실습 테이블 생성

```sql
DROP TABLE item PURGE;

CREATE TABLE item (
  item_id NUMBER       PRIMARY KEY,
  sku     VARCHAR2(20) NOT NULL,
  price   NUMBER(10,2) NOT NULL
);

BEGIN
  FOR i IN 1..200000 LOOP
    INSERT INTO item VALUES (
      i,
      CASE MOD(i,5)
        WHEN 0 THEN 'ABC'||TO_CHAR(i,'FM000000')
        WHEN 1 THEN 'ABD'||TO_CHAR(i,'FM000000')
        WHEN 2 THEN 'ACD'||TO_CHAR(i,'FM000000')
        WHEN 3 THEN 'XYZ'||TO_CHAR(i,'FM000000')
        ELSE       'Z'  ||TO_CHAR(i,'FM000000')
      END,
      ROUND(DBMS_RANDOM.VALUE(1,999),2)
    );
  END LOOP;
  COMMIT;
END;
/

CREATE INDEX ix_item_sku ON item(sku);

BEGIN
  DBMS_STATS.GATHER_TABLE_STATS(USER,'ITEM', cascade=>TRUE);
END;
```

---

### `LIKE 'prefix%'`의 실제 인덱스 범위

B-Tree 인덱스는 키 값을 사전식 순서로 정렬합니다. `LIKE 'ABC%'` 조건은 내부적으로 다음 범위와 동일합니다:

$$
\text{sku} \in [\text{'ABC'}, \text{'ABD'}) 
$$

즉, 'ABC'로 시작하는 모든 값은 'ABC'와 'ABD' 사이의 연속된 구간에 위치합니다.

```sql
-- LIKE 접두사
SELECT /* B1: LIKE prefix */ COUNT(*)
FROM   item
WHERE  sku LIKE 'ABC%';

-- 동일한 인덱스 범위를 사용하는 BETWEEN
SELECT /* B2: BETWEEN lower ~ upper */ COUNT(*)
FROM   item
WHERE  sku >= 'ABC'
AND    sku <  'ABD';
```

두 쿼리는 **동일한 인덱스 리프 블록 범위**를 스캔하며, 실행계획도 동일한 `INDEX RANGE SCAN`을 보여줍니다.

일반적으로 다음 등식이 성립합니다:
$$ \text{col LIKE 'prefix%'} \iff \text{col} \in [\text{prefix}, \text{prefix\_next}) $$
여기서 `prefix_next`는 "다음 사전순 접두사"입니다.

---

### 접미사 및 부분 일치 검색의 한계

```sql
-- 인덱스 범위 스캔 불가능한 패턴
SELECT COUNT(*)
FROM   item
WHERE  sku LIKE '%000123';  -- 접미사 검색

SELECT COUNT(*)
FROM   item
WHERE  sku LIKE '%ABC%';    -- 부분 일치 검색
```

이런 패턴은 B-Tree 인덱스의 선두 컬럼 조건이 아니므로 범위 스캔을 할 수 없습니다. 대신 `INDEX FULL SCAN`이나 `TABLE FULL SCAN` 후 필터링이 발생합니다.

해결 전략:
1. **역방향 인덱스**: `REVERSE(sku)` 컬럼에 인덱스 생성 후 `LIKE REVERSE('321000%')` 검색
2. **함수 기반 인덱스**: `SUBSTR(sku, -6)` 등의 함수 기반 인덱스 활용
3. **전문 검색 엔진**: Oracle Text 등 전문 검색 기능 도입

---

### 날짜 컬럼의 잘못된 LIKE 사용

```sql
-- 나쁜 예: 함수 적용으로 인덱스 사용 불가
SELECT /* C0 BAD */
       COUNT(*)
FROM   ord
WHERE  TO_CHAR(order_dt,'YYYY-MM-DD') LIKE '2025-10-%';

-- 좋은 예: 범위 조건으로 인덱스 활용
SELECT /* C1 GOOD */
       COUNT(*)
FROM   ord
WHERE  order_dt >= DATE '2025-10-01'
AND    order_dt <  DATE '2025-11-01';
```

날짜 컬럼에 `TO_CHAR` 함수를 적용하면 인덱스 정렬 구조를 활용할 수 없습니다. 항상 원본 컬럼을 범위 조건으로 사용해야 합니다.

---

## 3. 선분 이력(유효기간) 모델의 인덱스 설계

### 기본 스키마

```sql
DROP TABLE hist PURGE;

CREATE TABLE hist (
  emp_id   NUMBER       NOT NULL,
  start_dt DATE         NOT NULL,
  end_dt   DATE         NOT NULL,
  grade    VARCHAR2(10) NOT NULL,
  CONSTRAINT ck_hist_range CHECK (start_dt <= end_dt)
);

BEGIN
  FOR e IN 1..50000 LOOP
    DECLARE
      s   DATE := DATE '2024-01-01';
      cnt PLS_INTEGER := TRUNC(DBMS_RANDOM.VALUE(3, 10));
    BEGIN
      FOR i IN 1..cnt LOOP
        INSERT INTO hist VALUES (
          e,
          s,
          s + TRUNC(DBMS_RANDOM.VALUE(20, 120)),
          CASE MOD(i,4)
            WHEN 0 THEN 'A'
            WHEN 1 THEN 'B'
            WHEN 2 THEN 'C'
            ELSE 'D'
          END
        );
        s := s + TRUNC(DBMS_RANDOM.VALUE(121, 240));
      END LOOP;
    END;
  END LOOP;
  COMMIT;
END;
/

-- 인덱스 후보
CREATE INDEX ix_hist_e_s ON hist(emp_id, start_dt);
CREATE INDEX ix_hist_e_s_e ON hist(emp_id, start_dt, end_dt);

BEGIN
  DBMS_STATS.GATHER_TABLE_STATS(USER,'HIST', cascade=>TRUE);
END;
```

---

### 특정 시점 유효 행 조회의 일반적 패턴

```sql
VAR id   NUMBER;
VAR asof DATE;
EXEC :id   := 12345;
EXEC :asof := DATE '2025-10-15';

SELECT /* D1: 유효한 선분 */
       grade, start_dt, end_dt
FROM   hist
WHERE  emp_id   = :id
AND    start_dt <= :asof
AND    end_dt   >= :asof;
```

이 쿼리의 일반적인 실행계획:
- Access Predicate: `emp_id = :id AND start_dt <= :asof`
- Filter Predicate: `end_dt >= :asof`

문제점:
1. `end_dt >= :asof` 조건이 필터로 처리되므로, `start_dt <= :asof`인 모든 행을 먼저 읽어야 함
2. `(emp_id, start_dt, end_dt)` 인덱스를 사용하면 커버링 인덱스 효과로 테이블 액세스를 줄일 수 있음

---

### 최적화 패턴: 최신 start_dt 한 건 조회

많은 시스템에서 한 엔티티의 특정 시점에는 단 하나의 유효 선분만 존재합니다. 이 경우 다음 패턴이 효과적입니다:

```sql
VAR id NUMBER; VAR t DATE;
EXEC :id := 777; EXEC :t := DATE '2025-10-10';

SELECT /* D2: 포인트 시점 최적화 */
       grade, start_dt, end_dt
FROM (
  SELECT /*+ INDEX_DESC(hist ix_hist_e_s) */
         grade, start_dt, end_dt
  FROM   hist
  WHERE  emp_id   = :id
  AND    start_dt <= :t
  ORDER  BY start_dt DESC
) WHERE ROWNUM = 1
  AND end_dt >= :t;
```

이 패턴의 장점:
1. `INDEX_DESC` 힌트로 인덱스를 역방향 스캔하여 `start_dt <= :t` 중 가장 큰 값 한 건만 찾음
2. `ROWNUM = 1`로 인해 첫 번째 행 발견 후 바로 스캔 중단
3. 읽은 한 건에 대해서만 `end_dt >= :t` 조건 확인
4. 엔티티당 선분 수가 많아도 일정한 성능 유지

---

## 4. Access Predicate vs Filter Predicate 이해

### 개념적 차이

실행계획의 `PREDICATE INFORMATION` 섹션은 조건식을 두 가지로 분류합니다:

- **Access Predicate**
  - 인덱스나 파티션에 접근하는 범위를 결정하는 조건
  - **스캔 범위를 줄이는** 역할
  - 예: `emp_id = 100 AND start_dt <= :asof`

- **Filter Predicate**
  - 이미 읽은 행에 대해 추가로 적용하는 조건
  - **읽은 후 버리는 비용**을 발생시킴
  - 예: `end_dt >= :asof`

튜닝의 핵심 목표는 **Filter Predicate를 가능한 한 Access Predicate로 승격**시키는 것입니다.

---

### 함수 적용으로 인한 Access 기회 상실

```sql
-- 나쁜 예: 컬럼에 함수 적용
SELECT /* E1 BAD */
       COUNT(*)
FROM   ord
WHERE  TRUNC(order_dt) = DATE '2025-10-15';
```

이 쿼리는 `TRUNC(order_dt)`가 인덱스 키 순서를 파괴하므로, `TRUNC(order_dt) = DATE '2025-10-15'` 조건이 필터로 처리됩니다.

#### 해결책 1: 가상 컬럼과 함수 기반 인덱스

```sql
ALTER TABLE ord ADD (order_dt_d AS (TRUNC(order_dt)));
CREATE INDEX ix_ord_dt_d ON ord(order_dt_d);

SELECT /* E1 GOOD - Access 승격 */
       COUNT(*)
FROM   ord
WHERE  order_dt_d = DATE '2025-10-15';
```

#### 해결책 2: 범위 조건으로 재작성

```sql
SELECT /* E1 ALT */
       COUNT(*)
FROM   ord
WHERE  order_dt >= DATE '2025-10-15'
AND    order_dt <  DATE '2025-10-16';
```

---

### 복합 인덱스에서의 조건 분배

인덱스 `(cust_id, order_dt, order_id)`가 있을 때:

```sql
SELECT /* E2 */
       order_id
FROM   ord
WHERE  cust_id  = :cid
AND    order_dt >= :d1
AND    status   = 'PAID';
```

실행계획:
- Access: `cust_id = :cid AND order_dt >= :d1`
- Filter: `status = 'PAID'`

`status`는 인덱스 키에 없으므로 필터가 됩니다. 하지만 인덱스를 `(cust_id, status, order_dt)`로 재설계하면:
- Access: `cust_id = :cid AND status = 'PAID' AND order_dt >= :d1`

**핵심 원칙**:
1. 선두 컬럼에 등호(=) 조건이 있으면 후행 컬럼도 Access로 승격될 가능성 높아짐
2. 인덱스 설계 시 자주 사용되는 필터 조건을 인덱스 중간에 포함시키는 것 고려

---

## 5. 인덱스 단편화 관리

### 현대 환경에서의 인덱스 단편화 영향

최신 SSD 저장장치는 높은 랜덤 I/O 성능을 제공하지만, 여전히 지나친 단편화는 성능에 영향을 미칠 수 있습니다. Oracle B-Tree 인덱스는 자동 리밸런싱을 수행하지만, 특정 상황에서는 수동 정리가 필요합니다.

---

### 단편화 의심 상황

1. **대량 삭제 후**: 많은 삭제된 리프 행(`del_lf_rows`)이 재사용되지 않는 경우
2. **편향 삽입 패턴**: 단조 증가하는 키(시퀀스 PK)로 인한 마지막 리프 블록 집중
3. **인덱스 높이 증가**: `BLEVEL` 증가로 인한 탐색 비용 상승

---

### 모니터링 지표

```sql
-- 인덱스 구조 검증 (주의: 일시적 락 발생 가능)
ALTER INDEX ix_ord_dt VALIDATE STRUCTURE;

SELECT name, height, lf_rows, del_lf_rows,
       lf_blks, br_blks, pct_used
FROM   index_stats
WHERE  name = 'IX_ORD_DT';

-- 기본 통계 확인
SELECT index_name, blevel, leaf_blocks,
       distinct_keys, clustering_factor
FROM   user_indexes
WHERE  table_name = 'ORD';
```

주요 판단 기준:
- `del_lf_rows / lf_rows` 비율이 20-30% 이상일 때
- `pct_used`가 50% 미만으로 낮을 때
- `blevel` 증가 또는 `leaf_blocks`의 비정상적 증가

---

### 정리 작업 유형

#### COALESCE (병합)

```sql
ALTER INDEX ix_ord_dt COALESCE;
```
- 인접한 리프 블록의 빈 공간을 병합하는 가벼운 작업
- 온라인 상태에서 수행 가능, 인덱스 사용 중단 없음
- 트리 구조는 변경하지 않음

#### REBUILD (재구축)

```sql
ALTER INDEX ix_ord_dt REBUILD ONLINE;
```
- 인덱스를 완전히 새로 구축
- 공간 효율성과 구조 최적화 제공
- 더 많은 리소스(임시 공간, 리두, 언두) 사용

#### REVERSE KEY (역방향 키)

```sql
DROP INDEX ix_ord_dt;
CREATE INDEX ix_ord_dt ON ord(order_dt, order_id) REVERSE;
```
- 단조 증가 키의 "핫 블록" 경합 분산에 효과적
- 단점: 범위 스캔/정렬 작업에 부적합
- **포인트 조회 전용 인덱스에만 사용 권장**

---

### 정리 작업 효과 검증

```sql
-- 정리 전후 비교
SELECT index_name, blevel, leaf_blocks
FROM   user_indexes
WHERE  index_name = 'IX_ORD_DT';

-- COALESCE 또는 REBUILD 수행
ALTER INDEX ix_ord_dt COALESCE;

-- 정리 후 다시 확인
SELECT index_name, blevel, leaf_blocks
FROM   user_indexes
WHERE  index_name = 'IX_ORD_DT';
```

중요한 것은 **숫자 개선 여부 확인**입니다. 인덱스 정리 후 해당 인덱스를 사용하는 대표 쿼리의 다음 지표를 비교해야 합니다:
- `Buffers/Reads` 감소 여부
- `Elapsed Time` 개선 여부
- 관련 대기 이벤트(`buffer busy waits`, `latch free` 등) 감소 여부

---

## 6. 통합 실습: 나쁜 패턴 → 좋은 패턴

### 중복 범위 조건 정규화

```sql
VAR a DATE; VAR b DATE; VAR c DATE;
EXEC :a := DATE '2025-10-01';
EXEC :b := DATE '2025-10-31' + (1 - 1/86400);
EXEC :c := DATE '2025-10-15';

-- 비효율적인 이중 범위
SELECT /* F1 BAD */
       COUNT(*)
FROM   ord
WHERE  order_dt BETWEEN :a AND :b
AND    order_dt >= :c;

-- 효율적인 단일 범위
SELECT /* F1 GOOD */
       COUNT(*)
FROM   ord
WHERE  order_dt >= :c
AND    order_dt <= :b;
```

비교 포인트: 실행계획에서 Access Predicate의 차이와 `Buffers/Reads` 수치

---

### 월 접두사 검색 최적화

```sql
-- 문자열 변환으로 인덱스 사용 불가
SELECT /* F2 BAD */
       COUNT(*)
FROM   ord
WHERE  TO_CHAR(order_dt,'YYYY-MM') = '2025-10';

-- 범위 조건으로 인덱스 활용
SELECT /* F2 GOOD */
       COUNT(*)
FROM   ord
WHERE  order_dt >= DATE '2025-10-01'
AND    order_dt <  DATE '2025-11-01';
```

---

### 선분 이력 최적화 패턴

```sql
VAR id NUMBER; VAR t DATE;
EXEC :id := 777; EXEC :t := DATE '2025-10-10';

SELECT /* F3 */
       grade
FROM (
  SELECT /*+ INDEX_DESC(hist ix_hist_e_s) */
         grade, end_dt
  FROM   hist
  WHERE  emp_id   = :id
  AND    start_dt <= :t
  ORDER  BY start_dt DESC
) WHERE ROWNUM = 1
  AND end_dt >= :t;
```

---

## 결론

데이터베이스 쿼리 최적화는 단순히 실행 속도를 높이는 것을 넘어, 시스템 자원을 효율적으로 사용하는 방법을 이해하는 과정입니다. 본 문서에서 다룬 주요 개념들을 정리하면 다음과 같습니다:

1. **중복 범위 조건은 단일 범위로 통합하라**  
   같은 컬럼에 여러 범위 조건을 사용하면 옵티마이저가 넓은 범위를 스캔하게 만들 수 있습니다. 항상 논리적으로 동등한 단일 범위로 재작성하세요.

2. **LIKE 접두사 검색은 범위 검색으로 이해하라**  
   `LIKE 'ABC%'`는 `BETWEEN 'ABC' AND 'ABD'`와 같은 인덱스 범위를 사용합니다. 이 원리를 이해하면 문자열 패턴 검색도 인덱스를 효과적으로 활용할 수 있습니다.

3. **선분 이력 조회는 "최신 한 건 먼저 찾기" 패턴을 활용하라**  
   유효기간 모델에서 특정 시점의 데이터를 조회할 때는 역방향 인덱스 스캔과 `ROWNUM = 1`을 조합하여 최소한의 블록만 읽도록 최적화하세요.

4. **Access 조건을 최대화하는 인덱스 설계를 하라**  
   실행계획의 Access/Filter 구분을 이해하고, 가능한 많은 조건이 Access로 처리되도록 인덱스를 설계하세요. 함수 기반 인덱스와 가상 컬럼은 이 목표를 달성하는 강력한 도구입니다.

5. **인덱스 단편화 관리는 데이터 변경 패턴에 기반하라**  
   단편화는 항상 문제가 되지는 않지만, 대량 삭제나 편향 삽입이 발생한 후에는 주기적으로 모니터링하고 필요시 COALESCE나 REBUILD를 수행하세요. 단, 작업 전후로 실제 성능 개선 여부를 반드시 측정하세요.

최적화의 궁극적 목표는 "데이터베이스가 작업을 수행하는 방식을 이해하고, 그 방식을 최대한 효율적으로 만드는 것"입니다. 각 패턴의 이론적 배경을 이해하고, 실제 실행계획과 성능 지표를 꾸준히 분석하는 습관이 가장 효과적인 튜닝의 시작점입니다.