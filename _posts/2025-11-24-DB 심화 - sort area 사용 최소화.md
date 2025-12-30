---
layout: post
title: DB 심화 - sort area 사용 최소화
date: 2025-11-24 14:25:23 +0900
category: DB 심화
---
# Oracle에서 **Sort Area(TEMP/PGA) 사용을 최소화**하는 SQL 작성법

> 핵심 목표
1. **정렬 입력을 얇게 만들기**: 정렬 전에 정렬키와 식별자만 투영하여 정렬 대상 행의 폭을 최소화한다.
2. **부분 정렬(Stopkey) 활용하기**: 상위 N개만 필요한 경우 전체 정렬을 피하고 힙 정렬을 통해 조기 종료하도록 유도한다.
3. **인덱스 순서 활용하기**: 가능한 경우 인덱스가 제공하는 순서를 그대로 사용하여 정렬 작업 자체를 회피한다.

정렬 비용은 **행 수 × 행 폭**에 비례합니다. 따라서 Top-N 쿼리를 작성할 때는 "전체를 정렬한 뒤 N개를 자르는" 방식이 아니라, "정렬을 시작하기 전부터 N개만 확보하는" 방식으로 접근해야 합니다. 이는 실행계획에 `SORT ORDER BY STOPKEY`로 나타나며, PGA와 TEMP 사용량을 획기적으로 줄입니다.

---

## 1. 실습 환경 구성

### 샘플 테이블 및 데이터

```sql
-- 제품 마스터 테이블
CREATE TABLE d_product (
  prod_id   NUMBER PRIMARY KEY,
  category  VARCHAR2(16) NOT NULL,
  brand     VARCHAR2(16) NOT NULL
);

-- 매출 실적 테이블 (대용량 CLOB 컬럼 포함)
CREATE TABLE f_sales (
  sales_id  NUMBER PRIMARY KEY,
  prod_id   NUMBER NOT NULL REFERENCES d_product(prod_id),
  sales_dt  DATE   NOT NULL,
  qty       NUMBER NOT NULL,
  amount    NUMBER(12,2) NOT NULL,
  big_note  CLOB  -- 정렬 단계에서 행폭을 폭증시키는 대표 컬럼
);

-- 정렬 및 Top-N 쿼리에 사용할 핵심 인덱스
CREATE INDEX ix_sales_amount_desc ON f_sales(amount DESC);
CREATE INDEX ix_sales_dt_amount   ON f_sales(sales_dt, amount DESC);
CREATE INDEX ix_sales_prod_amt    ON f_sales(prod_id, amount DESC);

-- 샘플 데이터 입력
INSERT INTO d_product VALUES (10,'ELEC','BR0');
INSERT INTO d_product VALUES (20,'HOME','BR1');

INSERT INTO f_sales VALUES (1001,10,DATE '2025-02-03',1,10000, RPAD('a',2000,'a'));
INSERT INTO f_sales VALUES (1002,20,DATE '2025-02-05',2,15000, RPAD('b',2000,'b'));
INSERT INTO f_sales VALUES (1003,10,DATE '2025-02-10',1, 9000, RPAD('c',2000,'c'));
COMMIT;

EXEC DBMS_STATS.GATHER_TABLE_STATS(USER,'D_PRODUCT');
EXEC DBMS_STATS.GATHER_TABLE_STATS(USER,'F_SALES');
```

### 성능 측정 루틴

```sql
-- 최근 실행된 쿼리의 실제 실행계획 및 통계 확인
SELECT *
FROM   TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,
       'ALLSTATS LAST +PREDICATE +NOTE +ALIAS'));

-- 세션의 정렬 및 Workarea 관련 실시간 통계 모니터링
SELECT sn.name, ms.value
FROM   v$mystat ms JOIN v$statname sn ON sn.stat#=ms.stat#
WHERE  sn.name IN ('sorts (memory)','sorts (disk)',
                   'workarea executions - optimal',
                   'workarea executions - onepass',
                   'workarea executions - multipass');
```

---

## 2. 기본 원리: Stopkey와 부분 정렬

Oracle은 `FETCH FIRST N ROWS ONLY` 또는 `WHERE ROWNUM <= N` 구문을 만나면, 상위 N개 레코드만 확보하면 된다는 것을 인지합니다. 이때 옵티마이저는 전체 데이터 세트를 완전히 정렬하지 않고, 최소 힙(Min-Heap) 또는 최대 힙(Max-Heap) 자료구조를 PGA에 유지하며 N개의 후보만 관리하는 **부분 정렬(Partial Sort)** 을 수행합니다. 이 최적화가 성공하면 실행계획에 `SORT ORDER BY STOPKEY`가 표시되며, 이는 대량의 정렬 작업을 N개로 제한함으로써 PGA 메모리 사용량과 TEMP 스필 가능성을 크게 낮춥니다.

**주의사항**: `ROWNUM`은 `ORDER BY` 이전에 부여됩니다. 따라서 `WHERE ROWNUM <= N ORDER BY ...`와 같은 잘못된 구문은 무작위 N개의 레코드를 선택한 후 정렬하는 결과를 초래합니다. 항상 정렬이 먼저 적용된 서브쿼리 내에서 `ROWNUM` 또는 `FETCH`를 사용해야 합니다.

---

## 3. 핵심 패턴: 정렬을 끝낸 후 가공하기(Late Materialization)

이 패턴의 핵심은 정렬 작업이 발생하는 내부 단계에서는 가능한 한 적은 양의 데이터(정렬키와 행을 식별할 수 있는 값)만 다루고, 상위 N개가 확정된 이후에야 비로소 나머지 컬럼이나 조인을 수행하는 것입니다.

### 안티패턴: 무거운 컬럼을 포함한 정렬

```sql
-- CLOB 컬럼까지 정렬 대상에 포함시켜 행폭이 극도로 비대해짐
SELECT *
FROM   f_sales
WHERE  sales_dt >= DATE '2025-02-01'
ORDER  BY amount DESC
FETCH  FIRST 100 ROWS ONLY;
```
*문제점*: 정렬 작업은 전체 행의 모든 선택된 컬럼을 PGA에 로드합니다. `big_note`(CLOB)와 같은 대용량 컬럼이 포함되면 정렬 영역이 급격히 팽창하고, 메모리 부족 시 TEMP 테이블스페이스로의 디스크 스필이 발생하여 성능이 크게 저하됩니다.

### 개선 패턴: 정렬키 + 식별자만으로 Top-N 선정

```sql
WITH topn_keys AS (
  -- 1단계: 정렬키(amount)와 행 식별자(sales_id)만으로 얇은 정렬 수행
  SELECT /*+ INDEX_DESC(f_sales ix_sales_dt_amount) */
         sales_id,
         amount
  FROM   f_sales
  WHERE  sales_dt >= DATE '2025-02-01'
  ORDER  BY amount DESC
  FETCH FIRST 100 ROWS ONLY  -- Stopkey 최적화 유도
)
-- 2단계: 확정된 100개 Row에 대해서만 상세 컬럼 조회 및 조인 수행
SELECT s.sales_id, s.prod_id, s.sales_dt, s.qty, s.amount, s.big_note, p.brand
FROM   topn_keys t
JOIN   f_sales   s ON s.sales_id = t.sales_id
JOIN   d_product p ON p.prod_id  = s.prod_id
ORDER  BY t.amount DESC; -- 이미 정렬된 키 순서로 출력
```
*효과*: 내부 쿼리의 정렬 작업은 두 개의 컬럼(`sales_id`, `amount`)만을 처리하므로 행폭이 최소화됩니다. 결과적으로 정렬을 위한 PGA 메모리 요구량이 크게 감소하고, Stopkey 최적화로 인해 정렬 자체도 최소한으로 수행됩니다. 바깥쪽 조인은 100개의 행에 대해서만 발생하므로 전체적인 부하가 낮습니다.

### 최적화 패턴: ROWID를 활용한 최소화

`ROWID`는 행을 찾는 가장 직접적인 주소이며, 크기도 매우 작습니다. 이를 활용하면 정렬 입력을 극단적으로 줄일 수 있습니다.

```sql
WITH topn_rids AS (
  SELECT /*+ INDEX_DESC(f_sales ix_sales_dt_amount) */
         ROWID AS rid, -- PK 대신 더 가벼운 ROWID 사용
         amount
  FROM   f_sales
  WHERE  sales_dt >= DATE '2025-02-01'
  FETCH FIRST 200 ROWS ONLY
)
SELECT s.*, p.brand
FROM   topn_rids t
JOIN   f_sales   s ON s.ROWID = t.rid
JOIN   d_product p ON p.prod_id = s.prod_id
ORDER  BY t.amount DESC;
```

---

## 4. Top-N 적용 시점 전략: 조인과 가공보다 먼저

복잡한 조인이나 집계 작업을 수행한 결과에서 Top-N을 추출하는 것은 종종 비효율적입니다. 가능하다면, Top-N 후보를 먼저 추출한 후 필요한 확장 작업을 하는 것이 유리합니다.

### 안티패턴: 조인 후 정렬

```sql
-- 모든 조인이 완료된 큰 결과 집합을 정렬
SELECT s.*, p.category
FROM   f_sales s
JOIN   d_product p ON p.prod_id = s.prod_id
WHERE  p.category = 'ELEC'
ORDER  BY s.amount DESC
FETCH  FIRST 100 ROWS ONLY;
```

### 개선 패턴: Top-N 후 조인 (비즈니스 의미 확인 필수)

```sql
WITH topn_candidates AS (
  -- 'ELEC' 제품이라는 조건 없이, 전체에서 금액 기준 Top-N 후보 선정
  SELECT /*+ INDEX_DESC(f_sales ix_sales_amount_desc) */
         sales_id,
         prod_id,
         amount
  FROM   f_sales
  FETCH  FIRST 100 ROWS ONLY
)
-- 후보 100건에 대해서만 조인 및 필터링 적용
SELECT s.*, p.category
FROM   topn_candidates t
JOIN   f_sales   s ON s.sales_id = t.sales_id
JOIN   d_product p ON p.prod_id  = t.prod_id
WHERE  p.category = 'ELEC'; -- 여기서 일부 후보가 탈락할 수 있음
```
**중요**: 이 패턴은 **"전체 상위 100개 중 'ELEC' 제품인 것"** 을 찾습니다. 만약 요구사항이 **"'ELEC' 제품 중 상위 100개"** 라면, 이는 잘못된 결과를 초래합니다. 반드시 비즈니스 로직을 검토한 후 적용해야 합니다. 후자의 경우, `WHERE p.category = 'ELEC'` 조건을 내부 쿼리로 옮기고, `(prod_id, amount DESC)` 또는 `(category, amount DESC)` 인덱스를 활용해야 합니다.

---

## 5. 분석함수(윈도우 함수)의 정렬 최적화

`ROW_NUMBER()`, `RANK()` 등을 사용한 Top-N 쿼리는 편리하지만, 내부적으로 `WINDOW SORT` 작업을 유발하여 전체 데이터를 정렬할 수 있습니다.

### 표준 방식 (정렬 부담 큼)

```sql
SELECT *
FROM (
  SELECT s.*,
         ROW_NUMBER() OVER (ORDER BY amount DESC) AS rn
  FROM   f_sales s
) WHERE rn <= 100;
```

### 최적화 방식 1: 인덱스 순서 + ROWNUM으로 대체 (전역 Top-N)

분석함수가 필수가 아니라면, 인덱스를 이용해 정렬 자체를 회피하는 것이 가장 효과적입니다.

```sql
-- ix_sales_amount_desc 인덱스가 amount DESC 순서를 제공
SELECT /*+ INDEX_DESC(f_sales ix_sales_amount_desc) */
       sales_id, prod_id, sales_dt, qty, amount
FROM   f_sales
WHERE  ROWNUM <= 100;
-- FETCH FIRST 100 ROWS ONLY; -- 12c 이상에서 동일
```

### 최적화 방식 2: 인덱스로 윈도우 정렬 최소화 (그룹별 Top-K)

그룹별 상위 K개를 뽑을 때, 분석함수의 정렬 부담을 줄이려면 인덱스 순서를 활용해야 합니다.

```sql
-- (prod_id, amount DESC) 순서를 제공하는 인덱스가 있다고 가정
CREATE INDEX ix_sales_prod_amt_desc ON f_sales(prod_id, amount DESC);

SELECT /*+ INDEX(f_sales ix_sales_prod_amt_desc) */
       *
FROM (
  SELECT s.*,
         ROW_NUMBER() OVER (PARTITION BY prod_id ORDER BY amount DESC) AS rn
  FROM   f_sales s
) WHERE rn <= 3;
```
이 경우, 데이터가 이미 인덱스 순서(`prod_id`별로 정렬되고, 각 그룹 내에서 `amount` 기준 내림차순)로 접근되므로, `WINDOW SORT` 연산은 각 파티션 내에서의 정렬을 최소화하거나 생략할 수 있습니다. 실행계획에서 `WINDOW NOSORT` 또는 `WINDOW BUFFER`가 나타나는지 확인하세요.

### 최적화 방식 3: LATERAL 조인(CROSS APPLY) 활용

드라이빙 테이블(그룹 목록)의 크기가 작을 때 매우 효과적인 방법입니다. 각 그룹에 대해 독립적으로 인덱스 탐색과 Stopkey를 적용합니다.

```sql
-- 각 제품(prod_id)별로 매출액 Top-3 조회
SELECT p.prod_id, p.category, s.sales_id, s.amount
FROM   d_product p
CROSS APPLY (
  SELECT /*+ INDEX_DESC(s ix_sales_prod_amt_desc) */
         sales_id, amount
  FROM   f_sales s
  WHERE  s.prod_id = p.prod_id -- 외부 조인 변수(p.prod_id) 참조
  ORDER BY s.amount DESC
  FETCH FIRST 3 ROWS ONLY
) s
WHERE p.category = 'ELEC';
```
*장점*: 글로벌한 대규모 정렬(`WINDOW SORT`)이 전혀 발생하지 않습니다. 각 제품별로 인덱스를 통해 최대 3건만 빠르게 추출합니다.
*단점*: 드라이빙 테이블(여기서는 `d_product`)의 행 수가 매우 많고, 각 그룹에 대한 인덱스 탐색 비용이 높다면 NL 조인의 반복 비용이 커질 수 있습니다.

### 최적화 방식 4: Top-1은 KEEP 구문으로 해결

그룹별로 정렬 기준 1위의 값만 필요하다면, 분석함수보다 효율적인 `KEEP (DENSE_RANK FIRST/LAST)` 집계 함수를 사용할 수 있습니다.

```sql
-- 제품별 최고 매출액과 그 때의 판매일자
SELECT prod_id,
       MAX(amount) AS max_amount,
       MIN(sales_dt) KEEP (DENSE_RANK FIRST ORDER BY amount DESC) AS dt_of_max
FROM   f_sales
GROUP BY prod_id;
```
이 쿼리는 `SORT GROUP BY` 작업은 필요하지만, `WINDOW SORT` 작업을 피할 수 있습니다. `DENSE_RANK FIRST`는 동률 처리 시 모든 1위 행을 고려합니다.

---

## 6. 종합 실전 예제 및 효과 검증

**요구사항**: 2025년 2월 동안 발생한 거래 중, 매출액(amount)이 가장 높은 100건의 상세 내역과 제품 브랜드를 조회하시오. (CLOB 컬럼 포함)

### A안: 분석함수 사용 (단순하지만 비용 높음)

```sql
WITH ranked_sales AS (
  SELECT sales_id,
         ROW_NUMBER() OVER (ORDER BY amount DESC) AS rn
  FROM   f_sales
  WHERE  sales_dt >= DATE '2025-02-01' AND sales_dt < DATE '2025-03-01'
)
SELECT r.sales_id, s.prod_id, s.amount, s.sales_dt, s.big_note, p.brand
FROM   ranked_sales r
JOIN   f_sales   s ON s.sales_id = r.sales_id
JOIN   d_product p ON p.prod_id  = s.prod_id
WHERE  r.rn <= 100
ORDER BY s.amount DESC;
```

### B안: 정렬 최소화 패턴 적용 (권장)

```sql
-- 기간 필터와 정렬 키를 모두 커버하는 인덱스 생성 (사전 준비)
CREATE INDEX ix_f_sales_dt_amt ON f_sales(sales_dt, amount DESC);

WITH topn_keys AS (
  -- 1. 얇은 정렬: 기간 필터 적용 후, 식별자와 정렬키만으로 Stopkey Top-N 수행
  SELECT /*+ INDEX(f_sales ix_f_sales_dt_amt) */
         sales_id,
         amount
  FROM   f_sales
  WHERE  sales_dt >= DATE '2025-02-01' AND sales_dt < DATE '2025-03-01'
  ORDER BY amount DESC
  FETCH FIRST 100 ROWS ONLY -- SORT ORDER BY STOPKEY 기대
)
-- 2. 사후 확장: 확정된 100건에 대해 상세 컬럼과 조인 수행
SELECT s.sales_id, s.prod_id, s.amount, s.sales_dt, s.big_note, p.brand
FROM   topn_keys t
JOIN   f_sales   s ON s.sales_id = t.sales_id
JOIN   d_product p ON p.prod_id  = s.prod_id
ORDER BY t.amount DESC;
```

**성능 차이 검증**:
B안을 실행한 후 제공된 측정 루틴으로 다음을 확인하세요.
*   실행계획: `SORT ORDER BY STOPKEY`가 나타나고, `WINDOW SORT`는 보이지 않아야 합니다.
*   통계치(`v$mystat`): `sorts (disk)`의 증가가 0에 가깝고, `workarea executions - optimal` 비율이 높아야 합니다. `onepass`/`multipass`는 발생하지 않거나 매우 적어야 합니다.

---

## 결론

Oracle에서 정렬 관련 성능 문제와 PGA/TEMP 부족 현상은 근본적인 **쿼리 작성 패턴**을 변경함으로써 크게 개선할 수 있습니다. 핵심은 세 가지입니다.

1.  **정렬 대상을 가볍게 하라**: 정렬이 발생하는 내부 단계에서는 `ROWID` 또는 `PK`와 `정렬키`만을 대상으로 하여 행폭을 최소화하십시오.
2.  **전체 정렬을 피하라**: 상위 N개만 필요한 경우, 반드시 `FETCH FIRST N ROWS ONLY` 또는 `ROWNUM <= N`을 사용해 옵티마이저가 `STOPKEY` 최적화를 수행하도록 유도하십시오. 가능하면 인덱스 순서를 활용해 정렬 작업 자체를 생략하십시오.
3.  **무거운 작업은 뒤로 미루라**: 조인, CLOB 접근, 복잡한 계산 등은 Top-N 대상이 확정된 이후의 외부 쿼리에서 수행하십시오.

분석함수는 강력한 도구이지만, 특히 대량 데이터의 Top-N 처리에서는 숨겨진 정렬 비용을 유발할 수 있습니다. 항상 "인덱스 + Stopkey"라는 더 간단하고 효율적인 대안이 가능한지 먼저 고민하십시오. 이러한 원칙을 적용하면 디스크 정렬(`sorts (disk`)을 극소화하고, 메모리 내 최적 실행(`workarea executions - optimal`) 비율을 높여, 시스템의 전반적인 안정성과 응답 속도를 동시에 향상시킬 수 있습니다.