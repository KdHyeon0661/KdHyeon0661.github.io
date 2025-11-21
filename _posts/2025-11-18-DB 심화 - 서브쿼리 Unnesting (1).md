---
layout: post
title: DB 심화 - 서브쿼리 Unnesting (1)
date: 2025-11-18 16:25:23 +0900
category: DB 심화
---
# 서브쿼리 **Unnesting** 완전 가이드

## 0) 왜 “서브쿼리 Unnesting”인가?

- **정의:** 옵티마이저가 **실행계획 생성 전에**, 상관(또는 비상관) **서브쿼리를 조인 형태로 재작성**하는 **쿼리 변환(Query Transformation)** 기법.
- **핵심 효과:**
  1) 조인·파티션 프루닝·인덱스 사용 등 **더 많은 실행전략 후보**를 열어준다.
  2) 스칼라 서브쿼리 **대량 호출**(row-by-row)을 **한 번의 조인/집계**로 바꿔 **CPU/I-O**를 크게 줄인다.
  3) `EXISTS/IN/NOT EXISTS/NOT IN`을 **SEMI/ANTI JOIN**으로 치환해 **중복 제거·부정 조건**을 엔진 레벨에서 효율 처리한다.

---

## 1) 서브쿼리의 **분류**

### 상관성(correlation) 기준

- **상관 서브쿼리**(Correlated): 내부가 **바깥 쿼리의 컬럼**을 참조
  ```sql
  SELECT c.cust_id
  FROM   d_customer c
  WHERE  EXISTS (SELECT 1 FROM f_sales s WHERE s.cust_id = c.cust_id);
  ```
- **비상관 서브쿼리**(Uncorrelated): 내부가 바깥 컬럼을 참조하지 않음
  ```sql
  SELECT * FROM d_product
  WHERE  prod_id IN (SELECT prod_id FROM f_sales WHERE sales_dt >= DATE '2024-03-01');
  ```

### 반환 형태 기준

- **스칼라 서브쿼리**(Scalar): **단일 값**(single row, single column) 반환을 기대
- **멀티로우 서브쿼리**: `IN`, `EXISTS`, `ANY/ALL`, `NOT EXISTS/NOT IN` 등 **집합** 반환

### 위치 기준

- **SELECT-list**(스칼라 서브쿼리), **WHERE/HAVING**(필터), **FROM**(인라인 뷰/CTE)
  - FROM의 인라인 뷰는 주로 **뷰 머지(View Merge)** 대상이지만, 내부 상관/필터가 얽힌 경우 **언네스트 연쇄**가 관여한다.

### 연산자 유형

- **EXISTS / NOT EXISTS**  → (SEMI / ANTI JOIN 후보)
- **IN / NOT IN**          → (SEMI / ANTI JOIN 후보)
- **= ANY / > ALL** 등     → (쿼리 재작성 후 조인/집계로 변환 가능)
- **스칼라**(SELECT-list)  → (LEFT/INNER JOIN + GROUP BY/집계로 변환)

> **주의:** `NOT IN`은 **NULL** 포함 시 **결과가 공집합**이 되는 3값 논리 특성. 실무에선 **`NOT EXISTS`** 권장.

---

## 2) **Unnesting**의 의미 & 이점

### 의미

- 서브쿼리를 **조인 형태로 재작성**해, 옵티마이저가 **NL/HJ/SMJ**, **인덱스/파티션** 선택 등 **넓은 탐색공간**에서 비용을 비교할 수 있게 해준다.

### 이점

1) **대량 호출 제거**: 스칼라 서브쿼리 수십만 번 호출 ↦ **한 번의 조인/집계**.
2) **더 나은 액세스 경로**: `EXISTS/IN`을 **SEMI JOIN**으로 바꿔 **중복 탐색/반복 스캔**을 차단.
3) **프루닝/푸시다운** 촉진: 조인 앞에서 프레디킷 푸시/전이성으로 **읽는 블록 최소화**.
4) **조인 순서 최적화**: 언네스트 후 **LEADING/ORDERED/USE_* 힌트**로 구체 조정 가능.

---

## 3) **준비 테이블**(예시용)

> 이미 이전 글에서 사용한 스키마가 있다면 **재생성 불필요**합니다. 이하 예제는 `D_CUSTOMER`, `D_PRODUCT`, `F_SALES`를 가정합니다.

```sql
-- 핵심 인덱스(예)
CREATE INDEX ix_prod_cat_brand ON d_product(category, brand, prod_id);
CREATE INDEX ix_fs_cust_dt     ON f_sales(cust_id, sales_dt);
CREATE INDEX ix_fs_prod_dt     ON f_sales(prod_id, sales_dt);
```

---

## 4) **기본 예시**로 이해하는 Unnesting

### `EXISTS` → **SEMI JOIN** (가장 고전적)

**원형(상관 서브쿼리)**
```sql
SELECT /*+ qb_name(main) */
       c.cust_id
FROM   d_customer c
WHERE  EXISTS (
  SELECT 1
  FROM   f_sales s
  WHERE  s.cust_id = c.cust_id
  AND    s.sales_dt >= DATE '2024-03-01'
);
```

**언네스트 기대 효과**
- `EXISTS`는 **매칭 1건만** 찾으면 되므로, 조인 시 **SEMI JOIN**으로 변환.
- 실행계획 예시(개념):
  `NESTED LOOPS SEMI` 또는 `HASH JOIN SEMI`
- `ix_fs_cust_dt`로 **빠른 매칭** 기대.

**강제/제어 힌트**
```sql
SELECT /*+ UNNEST */ c.cust_id ...
-- 반대로 금지: /*+ NO_UNNEST */
```

**포인트**
- `SEMI JOIN`은 **중복 제거를 자연스럽게** 수행(외부 c 1행당 내부 s 다수여도 1번만 OK).
- 필터가 `s.sales_dt >= :d` 등 **고선택도**일수록 효과↑.

---

### `NOT EXISTS` → **ANTI JOIN**

**원형**
```sql
SELECT c.cust_id
FROM   d_customer c
WHERE  NOT EXISTS (
  SELECT 1 FROM f_sales s
  WHERE  s.cust_id = c.cust_id
  AND    s.sales_dt >= DATE '2024-03-01'
);
```

**언네스트 후**
- `HASH JOIN ANTI`/`NESTED LOOPS ANTI` 등으로 변환 → **일치하는 것이 없음을** 효율 판단.

> **주의:** `NOT IN (subq)`는 **NULL**이 하나라도 존재하면 전체가 **UNKNOWN**(거의 공집합). → 실무는 **`NOT EXISTS`** 추천.

---

### `IN` → **SEMI JOIN**

**원형(비상관)**
```sql
SELECT *
FROM   d_product p
WHERE  p.prod_id IN (SELECT s.prod_id
                     FROM   f_sales s
                     WHERE  s.sales_dt BETWEEN DATE '2024-03-01' AND DATE '2024-03-31');
```

**언네스트 후**
- 내부 `s`를 **중복 제거**(필요 시)한 뒤 **SEMI JOIN**.
- 인덱스 `ix_fs_prod_dt`가 있으면 **해시 집계 → 해시세미조인**이 빈번.

**팁**
- 내부 서브쿼리에 **GROUP BY prod_id**를 명시하면 **중복 제거 비용**을 옵티마이저가 더 정확히 판단하기도 한다.

---

### **스칼라 서브쿼리**(SELECT-list) → **조인 + 그룹 집계**

**원형(느린 패턴)**
```sql
-- 각 상품의 3월 총매출을 스칼라 서브쿼리로
SELECT p.prod_id,
       (SELECT SUM(s.amount)
        FROM   f_sales s
        WHERE  s.prod_id = p.prod_id
        AND    s.sales_dt BETWEEN DATE '2024-03-01' AND DATE '2024-03-31') AS m_sum
FROM   d_product p
WHERE  p.category = 'ELEC';
```
- `p`가 2만 행이라면 **스칼라 서브쿼리 2만 번** 호출. 캐시가 있어도 **값군 다양**하면 미스 빈번.

**언네스트/개선(조인 + 사전 집계)**
```sql
SELECT /*+ UNNEST */ p.prod_id, ss.m_sum
FROM   d_product p
LEFT JOIN (
  SELECT s.prod_id, SUM(s.amount) m_sum
  FROM   f_sales s
  WHERE  s.sales_dt BETWEEN DATE '2024-03-01' AND DATE '2024-03-31'
  GROUP  BY s.prod_id
) ss
ON ss.prod_id = p.prod_id
WHERE p.category = 'ELEC';
```
- 내부에서 **한 번만** 집계 → 외부와 **조인**.
- 실행계획: `HASH GROUP BY` → `HASH JOIN OUTER` 등.
- **대량 호출 제거**가 포인트.

---

### `ANY/ALL` 변환(요점만)

```sql
-- 가격이 하위집합의 "모든 값보다 큰" 상품
SELECT p.prod_id
FROM   d_product p
WHERE  p.prod_id > ALL (SELECT s.prod_id FROM f_sales s WHERE s.sales_dt >= :d);
```
- 내부를 **최대/최소** 등으로 **집계**한 뒤 **스칼라 비교**로 단순화하거나, **조인 + 필터**로 재작성 가능.
- 옵티마이저가 자동 시도하며, 난해하면 `NO_UNNEST`로 보호.

---

## 5) 언네스트 **적용/제한** 요약

- **잘 되는 경우**
  - `EXISTS/IN/NOT EXISTS` 기반의 **필터** 서브쿼리
  - SELECT-list **스칼라** 서브쿼리(특히 SUM/COUNT/MIN/MAX)
  - 상관 컬럼이 **등치**로 연결되고, 내부가 **SARGABLE**(인덱스 활용 가능)

- **어려운/제한되는 경우**
  - 외부조인 **NULL 보존** 의미에 영향(푸시/전이로 결과 변경 위험)
  - `NOT IN` + **NULL** 포함 가능성
  - 서브쿼리 내부에 **ROWNUM/샘플링/분석함수** 등 변환 저해 요소
  - 복잡한 OR/CASE로 **조건 분기**가 많아 **플랜 폭발** 위험

- **제어 힌트**
  - 강제/금지: `UNNEST` / `NO_UNNEST`
  - (연쇄 변환 관련) `MERGE/NO_MERGE`, `USE_CONCAT/NO_EXPAND`
  - 변환 범위 축소: `QB_NAME`, `NO_QUERY_TRANSFORMATION(@qb)`(필요 시)

---

## 6) **언네스트된 쿼리의 조인 순서 조정** (핵심 실전)

언네스트 후에는 “서브쿼리”가 **하나의 테이블(또는 집합)**처럼 **JOIN 트리**에 들어옵니다. 이때 **조인 순서/방법**은 일반 조인과 똑같이 **비용 기반**으로 결정됩니다. 필요하면 힌트로 조정하세요.

### 기본: 옵티마이저의 비용 기반 결정

- NL/Hash/Merge **어떤 조인 방식**이 최적인지, 누가 **드라이빙**(LEADING)일지 **통계/히스토그램/카디널리티**로 선택.

### 조정 힌트 모음

- **조인 순서:**
  - `LEADING(table_or_view …)` : 지정한 순서로 **드라이빙**
  - `ORDERED` : FROM에 **나열된 순서**를 따르도록(단, 뷰 머지/언네스트 뒤의 순서를 염두)
- **조인 방법:**
  - `USE_NL(t)` / `USE_HASH(t)` / `USE_MERGE(t)`
  - `SWAP_JOIN_INPUTS(t)` : 해시조인 빌드/프로브 **역전**(메모리/카디널리티상 유리)
- **세미/안티 조인에서도 유효**: `USE_HASH`를 붙이면 `HASH JOIN SEMI/ANTI`가 유도되는 식

### 예제: `EXISTS` 언네스트 후 **드라이빙** 바꾸기

```sql
-- 원형: EXISTS (언네스트 → SEMI JOIN)
SELECT /*+ qb_name(main) */ c.cust_id
FROM   d_customer c
WHERE  EXISTS (SELECT 1 FROM f_sales s
               WHERE s.cust_id = c.cust_id
               AND   s.sales_dt >= DATE '2024-03-01');

-- 조정 1) 고객을 드라이빙, 내부는 해시세미
SELECT /*+ UNNEST LEADING(c) USE_HASH(s) */ c.cust_id
FROM   d_customer c
JOIN   f_sales s
ON     s.cust_id = c.cust_id
AND    s.sales_dt >= DATE '2024-03-01'
-- 세미 조인 의미(중복 제거)는 옵티가 유지

-- 조정 2) 반대로, 매출을 드라이빙(매출이 소수일 때 유리)
SELECT /*+ UNNEST LEADING(s c) USE_NL(c) SWAP_JOIN_INPUTS(c) */
       c.cust_id
FROM   d_customer c
JOIN   f_sales s
ON     s.cust_id = c.cust_id
AND    s.sales_dt >= DATE '2024-03-01';
```

### 예제: 스칼라 서브쿼리 언네스트 후 조인 전략

```sql
-- 스칼라 언네스트(사전 집계 → 조인)
SELECT /*+ UNNEST LEADING(ss p) USE_HASH(p) */
       p.prod_id, ss.m_sum
FROM   d_product p
LEFT JOIN (
  SELECT s.prod_id, SUM(s.amount) m_sum
  FROM   f_sales s
  WHERE  s.sales_dt BETWEEN DATE '2024-03-01' AND DATE '2024-03-31'
  GROUP  BY s.prod_id
) ss
ON ss.prod_id = p.prod_id
WHERE p.category = 'ELEC';
```
- `LEADING(ss p)`로 **집계 결과**(작은 집합)를 먼저 잡고 `p`를 붙이는 전략을 강제.
- `USE_HASH(p)`로 **해시 아우터 조인** 유도.

---

## 7) **고급 케이스**

### OR 조건과 언네스트의 **연쇄 변환**

- OR 때문에 인덱스/언네스트가 막히면 **`USE_CONCAT`(OR-EXPAND)**로 **분기별** 최적 경로 + 언네스트를 유도.
```sql
SELECT /*+ USE_CONCAT */ *
FROM   f_sales s
WHERE  (EXISTS (SELECT 1 FROM d_product p
                WHERE p.prod_id=s.prod_id AND p.brand='B0'))
    OR (EXISTS (SELECT 1 FROM d_customer c
                WHERE c.cust_id=s.cust_id AND c.region='KOR'));
-- → UNION ALL 두 분기로 쪼개진 뒤 각 분기에서 EXISTS 언네스트(세미조인)
```

### HAVING 상관 서브쿼리

```sql
-- 카테고리별 합이 “그 카테고리의 3월 최대값 이상인 상품만”
SELECT p.prod_id
FROM   d_product p
GROUP  BY p.prod_id, p.category
HAVING SUM( (SELECT s.amount
             FROM f_sales s
             WHERE s.prod_id = p.prod_id
               AND s.sales_dt BETWEEN :d1 AND :d2) ) >=
       (SELECT MAX(x.sum_amt)
        FROM (SELECT s.prod_id, SUM(s.amount) sum_amt
              FROM f_sales s
              WHERE s.sales_dt BETWEEN :d1 AND :d2
              GROUP BY s.prod_id) x
        JOIN d_product p2 ON p2.prod_id = x.prod_id
        WHERE p2.category = p.category);
```
- 실무에선 이렇게 복잡하면 **서브쿼리를 먼저 집계 뷰/CTE로 만들고**, `MATERIALIZE`/`INLINE`과 `UNNEST` 조합으로 풀어 **명시적 조인 구조**로 단순화하는 편이 낫다.

### `NOT IN`의 NULL 함정과 언네스트

```sql
-- 주의: subq에 NULL이 있으면 전체가 UNKNOWN → 결과 거의 0
SELECT p.prod_id
FROM   d_product p
WHERE  p.prod_id NOT IN (SELECT s.prod_id FROM f_sales s);
```
- 옵티마이저가 **ANTI JOIN**으로 변환하려면 **NULL 제거 조건**(예: `WHERE s.prod_id IS NOT NULL`)이 필요.
- 실무는 애초에 **`NOT EXISTS`**로 작성:
```sql
SELECT p.prod_id
FROM   d_product p
WHERE  NOT EXISTS (SELECT 1 FROM f_sales s WHERE s.prod_id = p.prod_id);
```

### 언네스트 불가/비효율 상황에서의 대안

- **`NO_UNNEST`**로 보호하고, 스칼라면 **함수 기반**(가벼운 경우) 유지 + **결과 캐시/스칼라 캐시** 기대
- 가능하면 **집계/조인으로 재작성**해서 명시적 언네스트를 사람 손으로 구현

---

## 8) **관찰/진단** 방법

```sql
-- 실측 실행계획 + 변환 노트/프레디킷 이동 확인
SELECT *
FROM   TABLE(DBMS_XPLAN.DISPLAY_CURSOR(
         NULL, NULL, 'ALLSTATS LAST +NOTE +PREDICATE +ALIAS'));
```
- 계획에서 `NESTED LOOPS SEMI/ANTI`, `HASH JOIN SEMI/ANTI`가 보이면 **언네스트 성공**.
- `Predicate Information`에서 **전이 프레디킷/푸시다운** 여부를 확인.
- 일부 버전은 `Notes`에 **Subquery Unnesting** 관련 문구가 찍힌다.
- 필요 시 10053 trace(전문/비상)로 변환 결정을 추적.

---

## 9) 성능 체감 **시나리오**(OLTP/리포트)

### OLTP: 스칼라 다건 호출 → 조인

- 증상: CPU 높음, `BUFFER_GETS` 폭증, `ROW_NUMBER`(스칼라 캐시 miss) 비율 높은 SQL
- 처방: **스칼라 언네스트**로 한 번 집계 후 조인 → 실행시간 **수배~수십배** 단축 사례 빈번

### 리포트: 다중 `EXISTS` 필터

- 증상: 필터별 인덱스 존재하나 **조건 결합**이 약함
- 처방: **언네스트 + OR-EXPAND**로 각 분기에 적합 인덱스 경로 선택 → I/O 급감

---

## 10) **체크리스트**

- [ ] `EXISTS/IN/NOT EXISTS`는 **언네스트**로 **SEMI/ANTI** 유도
- [ ] `NOT IN` 금지(또는 **NULL 제거**) → `NOT EXISTS` 선호
- [ ] 스칼라 서브쿼리는 **사전 집계 + 조인**으로 바꿔 **대량 호출 제거**
- [ ] 언네스트 후 **조인 순서/방식**은 `LEADING/ORDERED/USE_*`로 조정
- [ ] OR로 인덱스가 막히면 **`USE_CONCAT`(OR-EXPAND)** + 언네스트 연쇄
- [ ] 외부조인·NULL 보존 의미가 섞이면 **`NO_UNNEST`**로 보호하고 수동 재작성 고려
- [ ] 검증은 항상 **E-Rows vs A-Rows** + `DBMS_XPLAN.DISPLAY_CURSOR('ALLSTATS LAST …')`

---

## 부록 A. **미니 실습 모음**

### A-1) EXISTS → SEMI JOIN + 조인 순서 지정

```sql
EXPLAIN PLAN FOR
SELECT /*+ UNNEST LEADING(c s) USE_NL(s) */
       c.cust_id
FROM   d_customer c
WHERE  EXISTS (
  SELECT 1 FROM f_sales s
  WHERE  s.cust_id = c.cust_id
  AND    s.sales_dt >= DATE '2024-03-01'
);
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);
```

### A-2) 스칼라 언네스트(집계→조인) + 해시조인 유도

```sql
EXPLAIN PLAN FOR
SELECT /*+ UNNEST */ p.prod_id, ss.m_sum
FROM   d_product p
LEFT JOIN (
  SELECT /*+ USE_HASH(s) */ s.prod_id, SUM(s.amount) m_sum
  FROM   f_sales s
  WHERE  s.sales_dt BETWEEN DATE '2024-03-01' AND DATE '2024-03-31'
  GROUP  BY s.prod_id
) ss
ON ss.prod_id = p.prod_id
WHERE p.category = 'ELEC';
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);
```

### A-3) OR-EXPAND + 언네스트

```sql
EXPLAIN PLAN FOR
SELECT /*+ USE_CONCAT */ *
FROM   f_sales s
WHERE  EXISTS (SELECT 1 FROM d_product  p WHERE p.prod_id=s.prod_id AND p.brand='B0')
   OR  EXISTS (SELECT 1 FROM d_customer c WHERE c.cust_id=s.cust_id AND c.tier='VIP');
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);
```

### A-4) NOT IN → NOT EXISTS로 교체

```sql
-- 나쁜 예(서브쿼리에 NULL 있으면 거의 공집합)
SELECT p.prod_id
FROM   d_product p
WHERE  p.prod_id NOT IN (SELECT s.prod_id FROM f_sales s);

-- 좋은 예
SELECT p.prod_id
FROM   d_product p
WHERE  NOT EXISTS (SELECT 1 FROM f_sales s WHERE s.prod_id = p.prod_id);
```

---

## 부록 B. **FAQ**

**Q1. 언네스트가 항상 빠른가요?**
A. 대부분 이득이지만, **극저량**(수행 건수 수십) + **스칼라 캐시 적중률↑**이면 원형이 더 빠를 수도 있습니다. → 실측으로 판단.

**Q2. 왜 언네스트 후 플랜이 더 나빠졌죠?**
A. **카디널리티 오판/히스토그램 부재**로 비용이 잘못 평가된 케이스. 통계 정비 후 `LEADING/USE_*`로 보정하세요.

**Q3. 외부조인과 섞이면 결과가 달라질 수 있나요?**
A. 예. **NULL 보존** 의미 때문에 무리한 프레디킷 전이/푸시는 금물. 해당 **QB에 `NO_UNNEST`**로 방어하거나 수동 재작성.

---

### 결론

- **서브쿼리 Unnesting**은 **“row-by-row”를 “set-based”로** 바꿔주는 강력한 자동 재작성이다.
- `EXISTS/IN/NOT EXISTS`는 **SEMI/ANTI**, 스칼라는 **집계→조인**으로 바뀌며, 그 순간부터는 **모든 조인 최적화 기법**을 동원할 수 있다.
- 마지막 한 끗은 **조인 순서/방법 제어(LEADING/USE_*)**와 **정확한 통계**다.
- 언제나 **실측 플랜**으로 확인하자. **E-Rows vs A-Rows**가 정답을 말해준다.
