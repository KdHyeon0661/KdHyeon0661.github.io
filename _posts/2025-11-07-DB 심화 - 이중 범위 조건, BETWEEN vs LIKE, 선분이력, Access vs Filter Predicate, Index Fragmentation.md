---
layout: post
title: DB 심화 - 이중 범위 조건, BETWEEN vs LIKE, 선분이력, Access vs Filter Predicate, Index Fragmentation
date: 2025-11-07 22:25:23 +0900
category: DB 심화
---
# 같은 컬럼 이중 범위, `BETWEEN` vs `LIKE`, 선분이력, Access/Filter, Index Fragmentation

> 주제  
> - 같은 컬럼에 **범위 조건을 두 번** 줄 때 옵티마이저가 스캔 범위를 어떻게 잡는지  
> - `BETWEEN` 과 `LIKE 'prefix%'` 가 **어떤 인덱스 범위**를 만드는지  
> - **선분 이력(유효기간)** 스키마에서 인덱스를 어떻게 설계하고 스캔을 줄이는지  
> - 실행계획에서 **Access Predicate vs Filter Predicate** 를 어떻게 읽고, SARGability를 통해 Access로 끌어올리는지  
> - **Index Fragmentation** 이 언제 실제로 문제가 되고, COALESCE/REBUILD 를 언제 할지

---

## 0. 실습 공통 준비

```sql
ALTER SESSION SET nls_date_format  = 'YYYY-MM-DD HH24:MI:SS';
ALTER SESSION SET statistics_level = ALL;  -- DBMS_XPLAN ALLSTATS LAST 용
```

실행계획 확인용 템플릿:

```sql
SELECT * FROM TABLE(
  DBMS_XPLAN.DISPLAY_CURSOR(
    NULL, NULL,
    'ALLSTATS LAST +PREDICATE +PEEKED_BINDS +ALIAS +IOSTATS +MEMSTATS'
  )
);
```

---

## 1. 같은 컬럼에 **이중 범위 조건**을 줄 때의 함정

### 1.1 실습 스키마

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
      DATE '2024-01-01' + MOD(i, 540), -- 약 18개월 구간 순환
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
/
```

이제 `order_dt` 에 **서로 겹치는 범위**를 두 번 줄 때를 보자.

---

### 1.2 교집합(AND) — 같은 컬럼에 범위를 두 번 쓰는 나쁜 패턴

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

SELECT * FROM TABLE(
  DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST +PREDICATE +PEEKED_BINDS')
);
```

이 WHERE 절은 논리적으로는

$$
\text{order\_dt} \in [\text{b1},\text{b2}] \cap [\text{2025-10-15}, +\infty)
$$

이고, 교집합이니 실제로는

$$
\text{order\_dt} \in [\text{2025-10-15}, \text{b2}]
$$

와 같다. 즉, **단일 범위**로 쓸 수 있다.

문제는 옵티마이저가 항상 이걸 완벽하게 단순화해 주지는 않는다는 점이다.  
특히 바인드/함수 혼합될 때, 조건이 다음처럼 나뉠 수 있다.

- Access Predicate: `order_dt BETWEEN :b1 AND :b2`
- Filter Predicate: `order_dt >= DATE '2025-10-15'`

Access 로 이미 넓게 잡아놓고, Filter 로 한 번 더 거르는 꼴이 되면  
인덱스 스캔 범위가 **불필요하게 넓어질 수 있다.**

#### 올바른 형태 — 단일 범위로 정규화

```sql
SELECT /* A1 GOOD */
       COUNT(*)
FROM   ord
WHERE  order_dt >= DATE '2025-10-15'
AND    order_dt <  DATE '2025-11-01';   -- 좌포우개
```

날짜는 가능하면 `BETWEEN` 대신

- **좌측 포함 / 우측 배제** 형태로 쓰는 것이 시분초 경계에서 안전하다.

$$
\text{order\_dt} \in [\text{2025-10-15 00:00:00}, \text{2025-11-01 00:00:00})
$$

---

### 1.3 합집합(OR) — 여러 달, 여러 기간을 한 번에 조회할 때

```sql
SELECT /* A2: 10월 OR 12월 */
       COUNT(*)
FROM   ord
WHERE  (order_dt BETWEEN DATE '2025-10-01'
                     AND DATE '2025-10-31' + (1 - 1/86400))
   OR  (order_dt BETWEEN DATE '2025-12-01'
                     AND DATE '2025-12-31' + (1 - 1/86400));
```

옵티마이저는 보통

- `INDEX RANGE SCAN` 두 번 + `BITMAP OR`  
  또는  
- `UNION-ALL View` 로 내부 변형해서 처리한다.

두 달 정도면 괜찮지만  
**달이 5개, 10개로 늘어나면** SQL 텍스트가 지저분해지고

- 파싱/플랜 캐시
- 카디널리티 추정
- Bind 변수 재사용

이 복잡해진다.

#### 확장 가능한 방식 — “월 목록 테이블” 조인

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

- 월이 늘어나면 **M**에만 Row를 늘리면 된다.
- 옵티마이저는 M을 작은 드라이빙 집합으로 보고  
  각 월에 대해 **짧은 범위 Range Scan** 을 반복 수행할 수 있다.

---

### 1.4 정리 — “같은 컬럼에 범위 두 번”의 일반 원칙

1. 교집합(AND)은 **항상 단일 범위로 정규화**하라.
2. 합집합(OR)이 많아지면 **목록 테이블 + 조인**으로 바꾸어라.
3. 실행계획에서 **Access/Filter Predicate** 를 보고  
   실제 인덱스 스캔 범위가 어떻게 잡히는지 확인하라.

---

## 2. `BETWEEN` vs `LIKE` — 접두사 검색의 스캔 범위

### 2.1 실습용 코드 테이블

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
/
```

---

### 2.2 `LIKE 'ABC%'` 가 만드는 인덱스 범위

B-Tree 인덱스는 **사전순** 키 정렬이다.  
`LIKE 'ABC%'` 는 내부적으로 대략 다음 범위를 의미한다.

$$
\text{sku} \in [\text{'ABC'}, \text{'ABD'}) 
$$

즉,

- **하한**: `'ABC'`  
- **상한**: `'ABD'` (다음 접두사) 바로 직전까지

#### 예제

```sql
-- LIKE 접두사
SELECT /* B1: LIKE prefix */ COUNT(*)
FROM   item
WHERE  sku LIKE 'ABC%';

-- 같은 스캔 범위의 BETWEEN
SELECT /* B2: BETWEEN lower ~ upper */ COUNT(*)
FROM   item
WHERE  sku >= 'ABC'
AND    sku <  'ABD';
```

위 두 쿼리는

- **동일한 인덱스 상의 Leaf 범위**를 스캔한다.
- 실행계획을 보면 둘 다 `INDEX RANGE SCAN IX_ITEM_SKU` 가 나오는 것을 볼 수 있다.

> 일반화하면,
> $$ \text{col LIKE 'prefix%'} \iff \text{col} \in [\text{prefix}, \text{prefix\_next}) $$  
> 여기서 prefix\_next 는 “다음 사전순 접두사”다.

---

### 2.3 접미사/부분 일치 — 범위를 만들 수 없는 경우

```sql
-- 접미사 검색
SELECT /* 나쁜 예 */ COUNT(*)
FROM   item
WHERE  sku LIKE '%000123';

-- 부분 일치
SELECT COUNT(*)
FROM   item
WHERE  sku LIKE '%ABC%';
```

- 이런 패턴은 인덱스 키 범위를  
  $$[\text{?},\text{?})$$  
  형태로 만들 수 없다.
- 보통 `INDEX FULL SCAN` + 필터, 혹은 아예 TABLE FULL SCAN 으로 간다.

필요하다면 다음 전략을 쓴다.

1. **역방향 문자열 인덱스** (예: `REVERSE(sku)` 나 `substr(reverse(sku),1,N)` 에 인덱스)
2. Oracle Text / 검색 엔진 등 별도의 전문 인덱스

---

### 2.4 날짜/시간의 접두사 `LIKE` → `BETWEEN` 변환

나쁜 예:

```sql
-- 문자열 변환 + LIKE → 인덱스 못 탐
SELECT /* C0 BAD */
       COUNT(*)
FROM   ord
WHERE  TO_CHAR(order_dt,'YYYY-MM-DD') LIKE '2025-10-%';
```

- `order_dt` 에 **함수**를 걸어서 인덱스를 깨버린다.

좋은 예:

```sql
-- 10월 한 달 범위
SELECT /* C1 GOOD */
       COUNT(*)
FROM   ord
WHERE  order_dt >= DATE '2025-10-01'
AND    order_dt <  DATE '2025-11-01';
```

- 이건 B-Tree 의 사전순 정렬을 그대로 활용하는 **정상적인 Range Scan 패턴**이다.
- 시계열 중심 쿼리는 항상 **좌포우개 범위**로 통일하는 습관이 좋다.

---

## 3. 선분 이력(유효기간 이력)과 인덱스 스캔 효율

### 3.1 기본 스키마

**패턴**

- 한 엔티티(사원, 고객, 계약 등)에 대해  
  여러 개의 **비겹치는 기간 혹은 겹칠 수 있는 기간**을 기록
- 스키마:  
  `(entity_id, start_dt, end_dt, other columns …)`

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
```

인덱스 후보:

```sql
-- 후보1: 자주 떠올리는 형태
CREATE INDEX ix_hist_e_s ON hist(emp_id, start_dt);

-- 후보2: end_dt까지 포함 (커버링 효과 기대)
CREATE INDEX ix_hist_e_s_e ON hist(emp_id, start_dt, end_dt);

BEGIN
  DBMS_STATS.GATHER_TABLE_STATS(USER,'HIST', cascade=>TRUE);
END;
/
```

---

### 3.2 “특정 시점에 유효한 행” 찾기 — 전형적인 질의

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

SELECT * FROM TABLE(
  DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST +PREDICATE')
);
```

보통 실행계획에서는

- Access Predicate  
  `emp_id = :id AND start_dt <= :asof`
- Filter Predicate  
  `end_dt >= :asof`

이렇게 나온다.

즉,

1. `(emp_id, start_dt)` 인덱스를 사용해  
   `emp_id = :id` 이고 `start_dt <= :asof` 인 **모든 후보 선분**을 스캔
2. 그 중에서 `end_dt >= :asof` 인 것만 **필터**로 거른다.

여기서 **핵심 포인트** 두 가지:

1. **후행 컬럼(end\_dt)** 에 조건이 있어도,  
   B-Tree 정렬 축이 `(emp_id, start_dt)` 라면  
   `end_dt >= :asof` 를 Access 로 승격하기 어렵다.
2. 그래도 `(emp_id, start_dt, end_dt)` 인덱스로 만들면  
   **커버링 인덱스**가 되어 테이블 미방문이 가능해진다.

---

### 3.3 “하나만 유효하다” 가정이 있을 때 — 가장 최근 start_dt 한 건만 읽기

많은 시스템에서 같은 `emp_id` 에서 주어진 시점에는  
**유효한 선분이 하나만 존재하도록** 보장한다.

이때 가장 빠른 패턴은:

1. `start_dt <= :asof` 인 선분 중 **가장 늦게 시작한(start_dt 가장 큰)** 것 **1건**만 가져오고
2. 그 한 건에 대해서만 `end_dt >= :asof` 를 체크한다.

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

- `(emp_id, start_dt)` 인덱스를 **DESC** 로 스캔하면서  
  `start_dt <= :t` 중 가장 큰 값을 가지는 한 건을 찾는다.
- `ROWNUM = 1` 이므로 **맨 앞 몇 블록만 읽고 종료**한다.
- 그 한 건에 대해서만 `end_dt >= :t` 를 필터로 확인.

이 패턴은

- `emp_id` 당 선분 개수가 많아도 효과가 유지되고
- **랜덤 I/O 최소**, **선형 탐색 없음** 이라는 장점이 있다.

---

### 3.4 범위 조회(“이 시점에 유효한 모든 사람”) 에서는?

예:

```sql
SELECT /* D3 */
       COUNT(*)
FROM   hist
WHERE  start_dt <= :t
AND    end_dt   >= :t;
```

이건 특정 `emp_id` 조건 없이 **전체 선분 중 겹치는 것**을 찾는 것이다.

전략:

1. `(start_dt)` 또는 `(start_dt, end_dt)` 인덱스로  
   `start_dt <= :t` 조건을 Access 로 쓰고  
   `end_dt >= :t` 는 Filter 로 쓰는 것
2. 혹은 선분 데이터가 매우 크면, 기간별 파티셔닝으로  
   `start_dt` 범위를 줄인 뒤 로컬 인덱스를 쓰는 것
3. 분석계(DW)라면 **Bitmap** 인덱스 + 조합도 검토 가능

---

## 4. Access Predicate vs Filter Predicate

### 4.1 개념 정리

실행계획에서 `PREDICATE INFORMATION` 를 보면 다음처럼 나뉜다.

- **Access Predicate**  
  - 인덱스/파티션 **접근 키**를 만드는 데 사용  
  - **스캔 범위를 줄이는 조건**
- **Filter Predicate**  
  - 이미 읽어온 Row 에 대해 **추가로 거르는 조건**  
  - 스캔 범위는 줄이지 못하고, **읽은 뒤 버리는 비용**만 만든다.

따라서 **튜닝의 1차 목표**는

> “조건을 **Filter** 에서 **Access** 로 가능한 많이 승격시키는 것”

이다.

---

### 4.2 함수 때문에 Access 기회를 날리는 예

```sql
-- 나쁜 예: 컬럼에 함수
SELECT /* E1 BAD */
       COUNT(*)
FROM   ord
WHERE  TRUNC(order_dt) = DATE '2025-10-15';
```

- 실행계획을 보면 보통 `TRUNC(order_dt)` 조건은
  - Table/Index Scan 이후의 **Filter** 로 잡힌다.
- B-Tree 의 정렬 축은 `order_dt` 인데  
  우리가 **함수 적용된 값**으로 비교를 걸어 버렸기 때문이다.

#### 해결 1) 가상 컬럼 + 함수 기반 인덱스

```sql
ALTER TABLE ord ADD (order_dt_d AS (TRUNC(order_dt)));
CREATE INDEX ix_ord_dt_d ON ord(order_dt_d);

SELECT /* E1 GOOD - Access 승격 */
       COUNT(*)
FROM   ord
WHERE  order_dt_d = DATE '2025-10-15';

SELECT * FROM TABLE(
  DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST +PREDICATE')
);
```

지금은

- Access Predicate: `order_dt_d = :b1`
- Filter Predicate: (없거나 최소한으로)

로 내려가고, `ix_ord_dt_d` 에 대해 Range Scan 이 가능하다.

#### 해결 2) 범위로 리라이트

```sql
SELECT /* E1 ALT */
       COUNT(*)
FROM   ord
WHERE  order_dt >= DATE '2025-10-15'
AND    order_dt <  DATE '2025-10-16';
```

- 이 방법도 `order_dt` 인덱스를 제대로 탈 수 있다.
- 시계열 조회 전체에 좌포우개 패턴을 쓰면, 튜닝/검증도 쉬워진다.

---

### 4.3 복합 인덱스에서 Access/Filter 분배 읽는 법

예: 인덱스 `(cust_id, order_dt, order_id)`

```sql
SELECT /* E2 */
       order_id
FROM   ord
WHERE  cust_id  = :cid
AND    order_dt >= :d1
AND    status   = 'PAID';
```

실행계획의 Predicate 정보:

- Access:  
  `cust_id = :cid AND order_dt >= :d1`
- Filter:  
  `status = 'PAID'`

이 경우, `status` 는 인덱스 키에 없으므로 어쩔 수 없이 필터다.

하지만 인덱스를 `(cust_id, status, order_dt)` 로 만들면

- Access:  
  `cust_id = :cid AND status = 'PAID' AND order_dt >= :d1`  
  (혹은 status 까지 Access, order_dt 는 Range Access)

로 바꿀 수 있다.

**핵심 공식**:

1. **선두 컬럼**에 **등호(=)** 조건이 있으면 Access 로 잘 올라간다.
2. 후행 컬럼도 되도록 **함수/형변환 없이** 조건을 주자.
3. 가상 컬럼/함수 기반 인덱스로  
   Access 로 끌어올 수 있는 부분은 최대한 끌어올려라.

---

## 5. Index Fragmentation — 언제 신경 써야 하나

### 5.1 Oracle B-Tree 의 특성

일반적인 현대 엔터프라이즈 SSD 는 4KB 랜덤 읽기 기준 **수십만~수백만 IOPS** 를 제공한다.:contentReference[oaicite:0]{index=0}  
Oracle B-Tree 인덱스는

- **리밸런싱**(Split/Merge)을 자동으로 수행하고
- 논리적인 “조각화”는 디스크가 HDD이든 SSD이든  
  **항상 큰 성능 병목**이 되는 것은 아니다.

그렇다고 **아무 때나 무시해도 된다**는 뜻은 아니다.

다음과 같은 상황에서는 **리빌드/Coalesce** 가 의미를 가진다.

---

### 5.2 파편화가 의심되는 패턴

1. **대량 삭제 후** 재사용이 거의 없는 인덱스

   - Leaf Block 에 삭제 마커가 가득  
   - `del_lf_rows` 많고 `pct_used` 낮음

2. **편향 삽입(오른쪽 증가 키)**

   - PK/Sequence가 단조 증가하는 경우  
   - 마지막 Leaf Block 에 I/O/락 경합 집중
   - 상위 Branch 구조도 한쪽으로 치우칠 수 있음

3. **BLEVEL 증가**

   - 인덱스 높이가 늘어나 탐색에 필요한 I/O 증가
   - BLEVEL 3 → 4 → 5 로 올라가는 순간에 영향이 생길 수 있다.

---

### 5.3 지표 확인

```sql
-- 1) VALIDATE STRUCTURE (주의: 일시적 락/부하)
ALTER INDEX ix_ord_dt VALIDATE STRUCTURE;

SELECT name, height, lf_rows, del_lf_rows,
       lf_blks, br_blks, pct_used
FROM   index_stats
WHERE  name = 'IX_ORD_DT';

-- 2) 요약 통계
SELECT index_name, blevel, leaf_blocks,
       distinct_keys, clustering_factor
FROM   user_indexes
WHERE  table_name = 'ORD';
```

감각적인 기준(현업에서 자주 쓰는):

- `del_lf_rows / lf_rows` 비율이 상당히 클 때
- `leaf_blocks` 가 과거보다 **의미 있게** 늘어났을 때
- `pct_used` 가 **매우 낮을 때**
- `blevel` 이 증가했거나, 같은 blevel 이라도 **leaf_blocks 증가 폭**이 큰 경우

이런 상황에서 해당 인덱스를 사용하는 쿼리의

- Buffers / Reads
- Elapsed Time
- latch / buffer busy wait

등이 실제로 나빠졌다면 조치를 고려한다.

---

### 5.4 COALESCE vs REBUILD vs REVERSE KEY

#### COALESCE

```sql
ALTER INDEX ix_ord_dt COALESCE;
```

- 인접한 Leaf Block 의 빈여유를 병합하는 **가벼운 정리** 작업
- 인덱스를 UNUSABLE 로 만들지 않고,  
  ONLINE 상태에서 빠르게 수행 가능
- **Tree 구조 자체는 크게 바꾸지 않는다.**

#### REBUILD

```sql
ALTER INDEX ix_ord_dt REBUILD ONLINE;
```

- 인덱스를 **새로 구축**한 뒤 교체
- Leaf/Branch 구조, PCTFREE, 압축 등을 새롭게 적용
- 공간 낭비가 심하거나 blevel 이 커졌을 때 유용
- REBUILD 시 TEMP/REDO/Undo 자원 사용 고려 필요

#### REVERSE KEY

```sql
DROP INDEX ix_ord_dt;
CREATE INDEX ix_ord_dt ON ord(order_dt, order_id) REVERSE;
```

- PK/Sequence 와 같이 **단조 증가하는 키** 에서  
  마지막 Leaf Block 에 집중되는 경합을 완화하기 위해 쓰는 기법
- 단점: **범위 스캔/정렬**에는 매우 불리하다.  
  (키 비트가 뒤집혀 있으니 순차 범위를 잡을 수 없음)
- **포인트 조회/동시성 분산**에만 사용해야 한다.

---

### 5.5 조치 전/후 비교

```sql
-- 조치 전
SELECT index_name, blevel, leaf_blocks
FROM   user_indexes
WHERE  index_name = 'IX_ORD_DT';

-- COALESCE 또는 REBUILD
ALTER INDEX ix_ord_dt COALESCE;
-- 또는
-- ALTER INDEX ix_ord_dt REBUILD ONLINE;

-- 조치 후
SELECT index_name, blevel, leaf_blocks
FROM   user_indexes
WHERE  index_name = 'IX_ORD_DT';
```

그리고 해당 인덱스를 사용하는 대표 쿼리를 다시 실행하여

- Buffers/Reads 감소 여부
- Elapsed Time 개선 여부
- 대기 이벤트 감소 여부

를 확인해야 한다.  
**조치를 했는데 숫자가 그대로거나 더 나쁘면, 그 조치는 잘못된 것**이다.

---

## 6. 통합 실습: 나쁜 예 → 좋은 예

### 6.1 같은 컬럼 이중 범위 정규화

```sql
VAR a DATE; VAR b DATE; VAR c DATE;
EXEC :a := DATE '2025-10-01';
EXEC :b := DATE '2025-10-31' + (1 - 1/86400);  -- 23:59:59 근사
EXEC :c := DATE '2025-10-15';

-- 나쁜 예
SELECT /* F1 BAD */
       COUNT(*)
FROM   ord
WHERE  order_dt BETWEEN :a AND :b
AND    order_dt >= :c;

-- 좋은 예
SELECT /* F1 GOOD */
       COUNT(*)
FROM   ord
WHERE  order_dt >= :c
AND    order_dt <= :b;

SELECT * FROM TABLE(
  DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST')
);
```

비교 포인트:

- 인덱스 스캔 시 Access Predicate 가 어떻게 달라지는지
- Buffers/Reads 수치 차이

---

### 6.2 월 접두사 처리: `LIKE` → 범위

```sql
-- 나쁜 예: 문자열 변환
SELECT /* F2 BAD */
       COUNT(*)
FROM   ord
WHERE  TO_CHAR(order_dt,'YYYY-MM') = '2025-10';

-- 좋은 예: 범위
SELECT /* F2 GOOD */
       COUNT(*)
FROM   ord
WHERE  order_dt >= DATE '2025-10-01'
AND    order_dt <  DATE '2025-11-01';

SELECT * FROM TABLE(
  DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST +PREDICATE')
);
```

---

### 6.3 선분 이력 포인트 조회 최적화

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

SELECT * FROM TABLE(
  DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST +PREDICATE')
);
```

- Range Scan + Stopkey 로 **선분 후보를 1건으로 줄이고**  
  Filter 로 end\_dt 를 확인하는 패턴을 체득할 수 있다.

---

### 6.4 Access 승격용 가상 컬럼

```sql
ALTER TABLE ord ADD (order_dt_day AS (TRUNC(order_dt)));
CREATE INDEX ix_ord_day ON ord(order_dt_day);

SELECT /* F4 */
       COUNT(*)
FROM   ord
WHERE  order_dt_day = DATE '2025-10-15';

SELECT * FROM TABLE(
  DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST +PREDICATE')
);
```

실행계획에서

- Access 에 `order_dt_day = :b1` 이 올라오는지
- Range Scan 이 되는지

를 확인해보자.

---

## 7. 요약 체크리스트

마지막으로 실무에 바로 써먹을 수 있는 체크리스트를 모으면:

### 7.1 같은 컬럼 이중 범위

- [ ] 같은 컬럼에 **범위 조건을 두 번**(교집합 AND) 쓰지 않았는가?  
      → 항상 **단일 범위**로 정규화.
- [ ] 달/분기 등 여러 구간을 OR 로 나열하지 않았는가?  
      → **목록 테이블 + 조인**으로 치환 가능한가?

### 7.2 `BETWEEN` vs `LIKE`

- [ ] `LIKE 'prefix%'` 를 **BETWEEN 하·상한**으로 표현할 수 있는가?  
      (문자열 상한은 **다음 접두사**)
- [ ] 날짜/시간 접두사는 `TO_CHAR` + LIKE 가 아니라  
      **좌포우개 범위**로 모두 통일했는가?
- [ ] 접미사/부분 일치는 필요한 경우에만 쓰고,  
      역방향/함수 기반 인덱스 또는 전문 인덱스 도입을 검토했는가?

### 7.3 선분 이력

- [ ] 유효 시점 조회에 “**가장 최근 start_dt 한 건 + end_dt 확인**” 패턴을 쓸 수 있는가?  
      → `(엔티티, start_dt)` 인덱스 + `INDEX_DESC` + Stopkey.
- [ ] 범위 조회가 크면 파티셔닝/로컬 인덱스로 범위를 줄였는가?

### 7.4 Access vs Filter

- [ ] 실행계획에서 **Access Predicate** / **Filter Predicate** 를 구분해서 보고 있는가?
- [ ] 컬럼에 **함수/형변환**을 적용해서 Access 기회를 날리지 않았는가?
- [ ] 가상 컬럼/함수 기반 인덱스로 Filter 를 Access 로 승격할 수 있는 부분은 없는가?

### 7.5 Index Fragmentation

- [ ] `blevel`, `leaf_blocks`, `del_lf_rows`, `pct_used` 를 주기적으로 확인하는가?
- [ ] 대량 삭제/편향 삽입 후 COALESCE/REBUILD 를 **숫자 기준**으로 판단하는가?
- [ ] REVERSE KEY 는 **포인트 조회/동시성 분산**에만 쓰고  
      범위/정렬이 필요한 인덱스에는 쓰지 않았는가?

---

## 8. 결론

- 같은 컬럼에 범위 조건을 여러 번 쓰면,  
  **스캔 구간이 불명확해지고 불필요 I/O**가 생긴다.  
  → 항상 **단일 범위**로 정규화하라.
- `LIKE 'prefix%'` 와 `BETWEEN lower AND upper` 는  
  B-Tree 상에서 **동일한 범위를 스캔**한다.  
  문자열/날짜 접두사를 전부 **범위식**으로 치환하라.
- 선분 이력(유효기간)은 “**최근 시작 한 건**을 먼저 찾고,  
  그 한 건에 대해 끝값을 확인하는” 패턴이 가장 효율적이다.
- 실행계획의 Access/Filter 구분은  
  **“어디까지가 스캔을 줄이는 조건인지”** 를 보여준다.  
  가능한 한 많은 조건을 **Access** 로 끌어올리자.
- Index Fragmentation 은 **언제나 문제**는 아니지만,  
  삭제/편향 삽입 후에는 COALESCE/REBUILD 가  
  실제 **I/O/경합/응답시간**을 줄이는지 **숫자**로 검증해야 한다.

결국, 모든 규칙은 다음 하나로 압축된다.

> **“조건의 모양을 B-Tree가 좋아하는 형태로 만들고,  
> 그 효과를 DBMS_XPLAN 의 숫자로 확인할 것.”**

이 글의 예제들을 직접 실행해 보면,  
같은 데이터에서도 **조건 표현 방식**에 따라  
스캔 범위·Buffers·Reads 가 얼마나 달라지는지 눈으로 확인할 수 있을 것이다.