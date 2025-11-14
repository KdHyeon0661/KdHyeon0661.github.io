---
layout: post
title: DB 심화 - 이중 범위 조건, BETWEEN vs LIKE, 선분이력, Access vs Filter Predicate, Index Fragmentation
date: 2025-11-07 22:25:23 +0900
category: DB 심화
---
# 같은 컬럼 **이중 범위 조건**, `BETWEEN` vs `LIKE` 스캔 범위, **선분이력** 인덱스 스캔 효율, **Access vs Filter Predicate**, **Index Fragmentation**까지 한 번에 정리

## 실습 공통 준비

```sql
ALTER SESSION SET nls_date_format = 'YYYY-MM-DD HH24:MI:SS';
ALTER SESSION SET statistics_level = ALL; -- ALLSTATS LAST 확인용
```

---

# 같은 컬럼에 **두 개의 범위 검색 조건**을 걸 때의 주의 사항

> B-Tree는 **선행 키**부터 **사전순**으로 탐색합니다.
> **같은 컬럼**에 범위가 **두 개** 붙으면, 옵티마이저는 **합집합(OR)** 또는 **교집합(AND)**을 만들어 **여러 구간**을 스캔하거나, **Full Scan**으로 빠지기도 합니다.
> **핵심**은 “**정규화된 단일 범위**”로 바꿔 **스캔 구간을 최소화**하는 것.

## 스키마 & 데이터

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
      DATE '2024-01-01' + MOD(i, 540), -- 약 18개월
      ROUND(DBMS_RANDOM.VALUE(1, 200000), 2),
      CASE MOD(i,5) WHEN 0 THEN 'NEW' WHEN 1 THEN 'PAID'
                    WHEN 2 THEN 'SHIP' WHEN 3 THEN 'DONE' ELSE 'CANC' END
    );
  END LOOP;
  COMMIT;
END;
/

-- 인덱스: 날짜 중심, 고객 중심
CREATE INDEX ix_ord_dt  ON ord(order_dt, order_id);
CREATE INDEX ix_ord_cdt ON ord(cust_id, order_dt, order_id);

BEGIN
  DBMS_STATS.GATHER_TABLE_STATS(USER, 'ORD', cascade=>TRUE,
    method_opt=>'for all columns size skewonly');
END;
/
```

## **같은 컬럼** 두 범위: “교집합(AND)”로 더 좁히는 경우

```sql
-- 예: 월말 실적만 뽑겠다고 날짜를 두 범위로 겹쳐 쓰는 코드
VAR b1 VARCHAR2; VAR b2 VARCHAR2;
EXEC :b1 := '2025-10-01 00:00:00'; EXEC :b2 := '2025-10-31 23:59:59';

SELECT /* A1: 두 범위의 교집합 */
       COUNT(*)
FROM   ord
WHERE  order_dt BETWEEN TO_DATE(:b1,'YYYY-MM-DD HH24:MI:SS') AND TO_DATE(:b2,'YYYY-MM-DD HH24:MI:SS')
AND    order_dt >= DATE '2025-10-15';  -- 같은 컬럼에 또 범위

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST +PEEKED_BINDS +ALIAS'));
```

**주의**
- 위 WHERE는 사실상 `order_dt BETWEEN '2025-10-15 00:00:00' AND '2025-10-31 23:59:59'` **단일 범위**로 **정규화 가능**.
- **두 개의 범위**가 들어가면 옵티마이저가 단순화하지 못하고, **Predicate Pushdown/Combine**이 어정쩡해져 **불필요 스캔**이 생길 수 있습니다.
- **항상** 같은 컬럼의 범위는 **한 번만**: **최대/최소 경계값**으로 **정규화**하세요.

### 바른 형태

```sql
SELECT /* A1’ */
       COUNT(*)
FROM   ord
WHERE  order_dt BETWEEN DATE '2025-10-15' AND DATE '2025-10-31' + 0.999994; -- 23:59:59 근사
```

## 같은 컬럼 두 범위: **합집합(OR)**일 때

```sql
-- 10월 혹은 12월 구간(두 범위 OR) : 두 "연속 구간"의 합집합
SELECT /* A2 */
       COUNT(*)
FROM   ord
WHERE  (order_dt BETWEEN DATE '2025-10-01' AND DATE '2025-10-31' + (1 - 1/86400))
   OR  (order_dt BETWEEN DATE '2025-12-01' AND DATE '2025-12-31' + (1 - 1/86400));

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST'));
```

- Oracle은 이 경우 **INDEX RANGE SCAN 두 번** + **BITMAP OR** 또는 **UNION ALL** 전략을 취할 수 있습니다.
- **월이 더 많아지면** `OR`가 많아져 **파싱/플랜 캐시/카디널리티 추정**이 꼬일 수 있음 →
  **캘린더/임시 테이블**로 월 목록을 만들고 **조인**으로 바꾸면 **확장성**이 좋습니다.

```sql
-- 권장: 월 테이블 조인
-- WITH M AS (SELECT DATE '2025-10-01' d FROM dual UNION ALL SELECT DATE '2025-12-01' FROM dual)
WITH M AS (
  SELECT DATE '2025-10-01' AS base_dt FROM dual
  UNION ALL SELECT DATE '2025-12-01' FROM dual
)
SELECT /* A2’ */
       COUNT(*)
FROM   ord o
JOIN   M   m
ON     o.order_dt >= m.base_dt
   AND o.order_dt <  ADD_MONTHS(m.base_dt,1);
```

---

# `BETWEEN` vs `LIKE` **스캔 범위 비교** (접두사 검색을 날짜/코드 범위로 변환)

> `LIKE 'ABC%'` 는 **사전순 범위**를 하나 만들어 B-Tree 스캔이 가능합니다.
> 날짜/코드의 “접두사”를 `BETWEEN` **하·상한**으로 바꾸면 **같은 효과**를 얻습니다.
> 핵심은 **SARGable**(인덱스가 먹히도록) 만들기.

## 코드 접두사 예

```sql
DROP TABLE item PURGE;
CREATE TABLE item(
  item_id NUMBER PRIMARY KEY,
  sku     VARCHAR2(20) NOT NULL,
  price   NUMBER(10,2) NOT NULL
);
BEGIN
  FOR i IN 1..200000 LOOP
    INSERT INTO item VALUES ( i
      , CASE MOD(i,5) WHEN 0 THEN 'ABC'||TO_CHAR(i,'FM000000')
                      WHEN 1 THEN 'ABD'||TO_CHAR(i,'FM000000')
                      WHEN 2 THEN 'ACD'||TO_CHAR(i,'FM000000')
                      WHEN 3 THEN 'XYZ'||TO_CHAR(i,'FM000000')
                      ELSE 'Z'||TO_CHAR(i,'FM000000') END
      , ROUND(DBMS_RANDOM.VALUE(1,999),2) );
  END LOOP;
  COMMIT;
END;
/
CREATE INDEX ix_item_sku ON item(sku);
BEGIN DBMS_STATS.GATHER_TABLE_STATS(USER,'ITEM', cascade=>TRUE); END;
/

-- LIKE 접두사
SELECT /* B1: LIKE prefix */ COUNT(*)
FROM item WHERE sku LIKE 'ABC%';

-- 같은 효과의 BETWEEN (문자열 상한은 접미 \xFF 비슷한 높은 문자로)
SELECT /* B2: BETWEEN lower ~ upper */
       COUNT(*)
FROM   item
WHERE  sku >= 'ABC'
AND    sku <  'ABD'; -- 문자열 상한=다음 접두사
```

**요점**
- `LIKE 'ABC%'` ↔ `sku BETWEEN 'ABC' AND 'ABD'-ε` 는 **동일한 인덱스 범위**를 스캔.
- **문자열 상한**은 보통 “다음 접두사”로 잡습니다(여기선 `'ABD'`).
- `LIKE '%ABC'`(접미사)는 **상한/하한**을 만들 수 없어 **Full Scan** 위험 → **역인덱스/함수기반 인덱스** 등 별도 설계 필요.

## 날짜 접두사 예 (`TRUNC`/포맷 문자열 주의)
### (실패 예) SARGability 깨지는 패턴

```sql
-- 암시적 형변환/함수 적용 → 인덱스 비활성화 위험
SELECT /* C0: 나쁜 예 */
       COUNT(*)
FROM   ord
WHERE  TO_CHAR(order_dt,'YYYY-MM-DD') LIKE '2025-10-%';
```
- `TO_CHAR(order_dt, ...)`는 **컬럼에 함수** → **인덱스 못탐(일반적으로)**.

### (성공 예) 범위로 변환

```sql
-- 10월 한 달 = [2025-10-01 00:00:00, 2025-11-01 00:00:00)
SELECT /* C1: 좋은 예 */
       COUNT(*)
FROM   ord
WHERE  order_dt >= DATE '2025-10-01'
AND    order_dt <  DATE '2025-11-01';
```
- `BETWEEN`보다 **좌측 포함/우측 배제** 형태가 **시분초**/경계값 처리에 안전.

---

# **선분이력(유효기간 이력)**의 인덱스 스캔 효율

> 전형 스키마: `(entity_id, start_dt, end_dt)`
> 질의: “시점 `:asof`에 **유효한 행**을 찾아라”
> WHERE: `entity_id = :id AND start_dt <= :asof AND end_dt >= :asof`

## 베이스 스키마

```sql
DROP TABLE hist PURGE;
CREATE TABLE hist (
  emp_id   NUMBER       NOT NULL,
  start_dt DATE         NOT NULL,
  end_dt   DATE         NOT NULL,
  grade    VARCHAR2(10) NOT NULL,
  CONSTRAINT ck_hist_range CHECK (start_dt <= end_dt)
);
-- 중복/겹침 방지는 애플리케이션/제약/프로시저로 보장(여기선 단순화)

BEGIN
  FOR e IN 1..50000 LOOP
    -- 각 사원: 3~10개의 비겹침 선분
    DECLARE
      s DATE := DATE '2024-01-01';
      cnt PLS_INTEGER := TRUNC(DBMS_RANDOM.VALUE(3, 10));
    BEGIN
      FOR i IN 1..cnt LOOP
        INSERT INTO hist VALUES ( e, s, s + TRUNC(DBMS_RANDOM.VALUE(20, 120)),
                                  CASE MOD(i,4) WHEN 0 THEN 'A' WHEN 1 THEN 'B' WHEN 2 THEN 'C' ELSE 'D' END );
        s := s + TRUNC(DBMS_RANDOM.VALUE(121, 240));
      END LOOP;
    END;
  END LOOP;
  COMMIT;
END;
/
```

## 인덱스 설계 후보와 **Access vs Filter** 차이

```sql
-- 후보1: (emp_id, start_dt)   -- 흔히 먼저 떠올리는 구조
CREATE INDEX ix_hist_e_s ON hist(emp_id, start_dt);

-- 후보2: (emp_id, start_dt, end_dt) -- 끝값까지 인덱스에 포함
CREATE INDEX ix_hist_e_s_e ON hist(emp_id, start_dt, end_dt);

BEGIN
  DBMS_STATS.GATHER_TABLE_STATS(USER,'HIST', cascade=>TRUE);
END;
/
```

### 질의 & 실행계획 관찰

```sql
VAR id NUMBER; VAR asof DATE;
EXEC :id := 12345; EXEC :asof := DATE '2025-10-15';

SELECT /* D1: 유효 선분 찾기 */
       grade, start_dt, end_dt
FROM   hist
WHERE  emp_id = :id
AND    start_dt <= :asof
AND    end_dt   >= :asof;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST +PREDICATE'));
```

- **(emp_id, start_dt)** 인덱스에서는 보통
  - **Access Predicate**: `emp_id = :id` **AND** `start_dt <= :asof`
  - **Filter Predicate**: `end_dt >= :asof`
  즉, **start_dt** 기준으로 **:asof** 이전까지 내려가며 후보를 읽고, **end_dt**는 **필터**로 거릅니다.

- **(emp_id, start_dt, end_dt)**이면
  - Access: `emp_id = :id` **AND** `start_dt <= :asof` (동일)
  - Filter: `end_dt >= :asof` (여전히 Filter인 경우가 많음)
  → **후행 end_dt 등치/범위가 Access로 승격되긴 어렵다**(B-Tree의 정렬축은 선행→후행).
  → 하지만 **커버링 효과**(인덱스만으로 선분 조건 검사)를 기대할 수 있어 **테이블 미방문**이 가능해집니다(선택적).

## 더 빠른 한 건 찾기(포인트 시점): **가장 최근 start_dt** 한 건만

> 같은 `emp_id` 내에서 `start_dt <= :asof` 중 **가장 큰 start_dt** 1건을 가져오고, 그 **end_dt ≥ :asof**만 확인.

```sql
SELECT /* D2: 포인트 시점 최적화 */
       grade, start_dt, end_dt
FROM (
  SELECT /*+ INDEX_DESC(hist ix_hist_e_s) */
         grade, start_dt, end_dt
  FROM   hist
  WHERE  emp_id = :id
  AND    start_dt <= :asof
  ORDER  BY start_dt DESC
) WHERE ROWNUM = 1
  AND end_dt >= :asof;
```

- `(emp_id, start_dt)`에 **DESC 스캔 + Stopkey**로 **맨 앞 한 블록**만 읽고 **필터** 1회 확인 → **최소 I/O**.
- **일반 포인트 질의**라면 이 패턴이 가장 안정적으로 빠릅니다.

## 선분 겹침/조회가 많을 때의 전략

- **파티셔닝**(기간/emp_id 해시) + **로컬 인덱스**로 범위를 줄임.
- **엔티티별 유효행 단 1건**만 유지(과거는 별도 테이블) → 온라인 조회 경로 단순화.
- **비교적 큰 범위 조회**(예: `:asof` 구간에 겹치는 모든 선분)라면
  `(start_dt)` 단독 인덱스 + **Bitmap AND**로 `(end_dt)` 인덱스와 결합도 고려(OLAP 성).

---

# **Access Predicate vs Filter Predicate** — 계획에서 “먹히는 조건”과 “읽고 거르는 조건”

> `DBMS_XPLAN.DISPLAY_CURSOR(..., 'ALLSTATS LAST +PREDICATE')`를 보면
> - **access**: 인덱스/파티션 접근 **키로 사용**, **스캔 범위 축소**
> - **filter**: 접근 후 **읽고 나서 거름**(스캔량에는 포함)

## 예제: 함수로 SARGability를 깨면 → **filter**로 떨어진다

```sql
-- 나쁜 예: 컬럼에 함수
SELECT /* E1: BAD - filter로 떨어짐 */
       COUNT(*)
FROM   ord
WHERE  TRUNC(order_dt) = DATE '2025-10-15';
```

**해결 1) 가상 컬럼 + 함수기반 인덱스**
```sql
ALTER TABLE ord ADD (order_dt_d AS (TRUNC(order_dt)));
CREATE INDEX ix_ord_dt_d ON ord(order_dt_d);

SELECT /* E1’: GOOD - access로 승격 */
       COUNT(*)
FROM   ord
WHERE  order_dt_d = DATE '2025-10-15';

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST +PREDICATE'));
```

**해결 2) 범위로 변환(좌포우개)**
```sql
SELECT /* E1’’ */
       COUNT(*)
FROM   ord
WHERE  order_dt >= DATE '2025-10-15'
AND    order_dt <  DATE '2025-10-16';
```

## 두 조건 중 하나라도 **access**로 끌어올려라

- 복합 인덱스에서 **선행 컬럼 = 등치**가 **access**를 보장.
- 후행은 범위라도 **선행 =** 덕분에 **짧은 연속 구간**으로 탐색 가능.
- **형변환 통일**, **함수기반 인덱스**, **가상 컬럼**으로 access 승격 기회를 적극 만들 것.

---

# **Index Fragmentation**(인덱스 파편화) — 언제, 무엇을, 어떻게 조치?

> Oracle B-Tree는 **리밸런스**가 자동으로 일어나 실질적 “조각화”가 OS 파일 시스템처럼 크게 문제 되지 않는 경우가 많습니다.
> 다만 아래 상황에서는 **리빌드/Coalesce**가 의미가 있습니다.

## 파편화/과대 성장의 전형

- **대량 삭제 후 미회수 공간**: Leaf에 **del row** 다수, **leaf_blocks↑**.
- **편향 삽입(오른쪽 증가 키)**: 상위/특정 Leaf **핫 블록 경합** + **여유 공간 불균형**.
- **BLEVEL(트리 높이) 과도**: 인덱스가 **높아져** 탐색 I/O가 늘어남(대용량·낮은 PCTFREE).

## 상태 확인

```sql
-- 1) INDEX_STATS (VALIDATE STRUCTURE는 일시적인 잠금/비용 주의)
ALTER INDEX ix_ord_dt  VALIDATE STRUCTURE;
SELECT name, height, lf_rows, del_lf_rows, lf_blks, br_blks, pct_used
FROM   index_stats WHERE name = 'IX_ORD_DT';

-- 2) USER_INDEXES 요약
SELECT index_name, blevel, leaf_blocks, distinct_keys, clustering_factor
FROM   user_indexes
WHERE  table_name = 'ORD';
```

**가이드라인(현장 감각)**
- `del_lf_rows / lf_rows` 비율이 **크고**, `leaf_blocks`가 **과대**, `pct_used`가 **낮을 때** 조치 고려.
- `blevel` 4 이상이면(환경/버퍼에 따라 다름) 탐색 비용 체크.

## 조치: **COALESCE** vs **REBUILD** vs **REVERSE KEY**

```sql
-- COALESCE: 인접 빈 Leaf를 병합(온라인/빠름/UNUSABLE 안 됨)
ALTER INDEX ix_ord_dt COALESCE;

-- REBUILD: 새로 만든 뒤 교체(공간 재활용·구조 정리, ONLINE 가능/리빌드 시간·용량 필요)
ALTER INDEX ix_ord_dt REBUILD ONLINE;

-- 편향 삽입 핫블록이면: REVERSE KEY로 분산 (PK·순차키에서 고려)
DROP INDEX ix_ord_dt;
CREATE INDEX ix_ord_dt ON ord(order_dt, order_id) REVERSE;
```

**주의**
- **REVERSE KEY**는 **범위 스캔/정렬**에 불리 → **포인트 조회/동시성 분산**에만 씁니다.
- Fragmentation 자체가 **항상** 성능 문제는 아닙니다. **실제 Buffers/Reads/경합**이 개선되는지 **숫자**로 확인하세요.

---

# 통합 실습 시나리오 · “나쁜 예 → 좋은 예” 비교

## 같은 컬럼 이중 범위(AND) 정규화

```sql
VAR a DATE; VAR b DATE; VAR c DATE;
EXEC :a := DATE '2025-10-01'; EXEC :b := DATE '2025-10-31' + 0.999994; EXEC :c := DATE '2025-10-15';

-- 나쁜 예
SELECT /* F1 BAD */
       COUNT(*)
FROM   ord
WHERE  order_dt BETWEEN :a AND :b
AND    order_dt >= :c;

-- 좋은 예(정규화)
SELECT /* F1 GOOD */
       COUNT(*)
FROM   ord
WHERE  order_dt BETWEEN :c AND :b;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST'));
```

## LIKE 접두사 ↔ BETWEEN 변환

```sql
-- 나쁜 예(암시적 형변환)
SELECT /* F2 BAD */
       COUNT(*)
FROM   ord
WHERE  TO_CHAR(order_dt,'YYYY-MM') = '2025-10';

-- 좋은 예(범위)
SELECT /* F2 GOOD */
       COUNT(*)
FROM   ord
WHERE  order_dt >= DATE '2025-10-01'
AND    order_dt <  DATE '2025-11-01';
```

## 선분이력 포인트 최적화

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

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST +PREDICATE'));
```

## Access vs Filter 승격(가상 컬럼)

```sql
ALTER TABLE ord ADD (order_dt_day AS (TRUNC(order_dt)));
CREATE INDEX ix_ord_day ON ord(order_dt_day);

SELECT /* F4: access */
       COUNT(*)
FROM   ord
WHERE  order_dt_day = DATE '2025-10-15';

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST +PREDICATE'));
```

## Fragmentation 조치 전/후 비교

```sql
-- 전 상태
SELECT index_name, blevel, leaf_blocks FROM user_indexes WHERE index_name='IX_ORD_DT';
ALTER INDEX ix_ord_dt COALESCE;
-- 또는 ALTER INDEX ix_ord_dt REBUILD ONLINE;

-- 후 상태
SELECT index_name, blevel, leaf_blocks FROM user_indexes WHERE index_name='IX_ORD_DT';
```

---

# 체크리스트 (현업 적용용)

- [ ] **같은 컬럼**에 **범위 두 번** 쓰지 말고 **단일 범위**로 **정규화**했는가?
- [ ] `LIKE 'ABC%'`는 **BETWEEN 하·상한**으로 변환 가능한가(문자열·날짜 모두)?
- [ ] **선분이력**은 **(엔티티, start_dt)** DESC + **Stopkey**로 **최근 1건**을 먼저 찾고 **end_dt**만 필터로 확인하는가?
- [ ] 실행계획에서 **access**와 **filter**를 구분해, 가능한 **access**로 승격(가상 컬럼/함수기반 인덱스/형변환 제거)했는가?
- [ ] **Fragmentation**은 숫자로 확인하고(blevel/leaf_blocks/del_lf_rows), **COALESCE/REBUILD**로 실제 개선되는지 검증했는가?
- [ ] **정렬/Top-N**이 많다면 **정렬 일치 인덱스(ASC/DESC)** + **좌포우개 범위**를 쓰는가?
- [ ] `OR`가 많아지면 **목록 테이블 조인**(UNION ALL/OR Expansion 대체)으로 바꿨는가?

---

## 요약

- **같은 컬럼**에 범위 조건을 **여러 번** 사용하면 **스캔 구간이 불명확**해져 **불필요 I/O**가 늘 수 있습니다. ⇒ **단일 범위로 정규화**.
- `LIKE 'prefix%'`는 **BETWEEN 하·상한**으로 바꿔 **동일한 인덱스 범위**로 안전하게 스캔하세요.
- **선분이력**은 “**최근 시작 한 건 + 끝값 확인**”이 가장 효율적이며, 인덱스는 보통 **(엔티티, start_dt)**를 사용합니다.
- **Access Predicate**는 **스캔량을 줄이는 조건**, **Filter**는 **읽고 나서 거르는 조건**입니다. **SARGability**로 Access 승격을 노리세요.
- **Index Fragmentation**은 **항상 문제는 아니나**, 삭제/편향 삽입 후에는 **COALESCE/REBUILD**로 구조/공간을 정리하고, **숫자**로 개선을 검증하세요.

> 항상 `DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST +PREDICATE')`로
> **전/후 Buffers, Reads, Predicate 유형(access/filter)**를 확인하세요. 체감이 아니라 **수치**가 답입니다.
