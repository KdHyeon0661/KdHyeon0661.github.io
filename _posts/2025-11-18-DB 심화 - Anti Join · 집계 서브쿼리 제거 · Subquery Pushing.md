---
layout: post
title: DB 심화 - Anti Join · 집계 서브쿼리 제거 · Subquery Pushing
date: 2025-11-18 18:25:23 +0900
category: DB 심화
---
# Anti Join · 집계 서브쿼리 제거 · Subquery Pushing (Oracle CBO 심화)

**목표:**
- **Anti Join**: `NOT EXISTS`/`NOT IN`/집합 차집합 형태와 실행계획(`HASH/NL/MERGE ANTI`)
- **Aggregate Subquery Elimination**: 상관 **집계 서브쿼리 제거**(사전 집계+조인으로 재작성)
- **Subquery Pushing**: `PUSH_SUBQ`(및 `NO_PUSH_SUBQ`)로 서브쿼리 **조기 평가·축소 → 조인**

---

## 0) 준비 스키마(요약)

```sql
-- 고객(1) : 매출(M)
CREATE TABLE d_customer(
  cust_id NUMBER PRIMARY KEY,
  region  VARCHAR2(8) NOT NULL,
  tier    VARCHAR2(8) NOT NULL
);

CREATE TABLE d_product(
  prod_id  NUMBER PRIMARY KEY,
  category VARCHAR2(16) NOT NULL,
  brand    VARCHAR2(16) NOT NULL
);

CREATE TABLE f_sales(
  sales_id NUMBER PRIMARY KEY,
  cust_id  NUMBER NOT NULL,
  prod_id  NUMBER NOT NULL,
  sales_dt DATE   NOT NULL,
  qty      NUMBER NOT NULL,
  amount   NUMBER(12,2) NOT NULL
);

-- 사실 테이블 인덱스(비고유 포함)
CREATE INDEX ix_fs_cust_dt ON f_sales(cust_id, sales_dt);  -- non-unique
CREATE INDEX ix_fs_prod_dt ON f_sales(prod_id, sales_dt);
CREATE INDEX ix_prod_cat_brand ON d_product(category, brand, prod_id);
```

---

# 1) Anti Join: “없음을” 가장 싸게 증명하기

**정의**: “A에는 있지만 B에는 **없는** 행만” 뽑는 조인.
Oracle 실행계획 용어로 `… JOIN ANTI`(Hash/Nested Loops/Merge)가 나타난다.

### 표현식과 의미 차이

```sql
-- (1) NOT EXISTS  : NULL 안전, 실무 표준
SELECT c.cust_id
FROM   d_customer c
WHERE  NOT EXISTS (
  SELECT 1 FROM f_sales s
  WHERE  s.cust_id = c.cust_id
  AND    s.sales_dt >= DATE '2024-03-01'
);

-- (2) NOT IN (subquery) : 서브쿼리 결과에 NULL 있으면 전체가 UNKNOWN(거의 공집합)
SELECT c.cust_id
FROM   d_customer c
WHERE  c.cust_id NOT IN (SELECT s.cust_id FROM f_sales s WHERE s.sales_dt >= DATE '2024-03-01');

-- (3) 차집합(MINUS) : 세트 관점
SELECT c.cust_id
FROM   d_customer c
MINUS
SELECT s.cust_id
FROM   f_sales s
WHERE  s.sales_dt >= DATE '2024-03-01';
```
- **권장:** 실무는 **`NOT EXISTS`**가 정석. `NOT IN`은 **NULL** 포함 시 결과가 달라진다.
- 옵티마이저는 보통 `NOT EXISTS`/차집합을 **ANTI JOIN**으로 변환한다.

### Anti Join 방식

- **Hash Anti Join**: 대량/광범위 조건에 강함. **Join Filter(Bloom)**로 바깥 스캔 조기 컷 가능.
- **Nested Loops Anti**: 바깥 소량 + 내부 고선택도 인덱스 탐색 시 유리.
- **Merge Anti Join**: 두 입력이 조인키로 정렬되어 있을 때 후보.

힌트 예시:
```sql
-- 해시 안티 유도
SELECT /*+ USE_HASH(s) LEADING(s c) */ c.cust_id
FROM   d_customer c
WHERE  NOT EXISTS (SELECT 1 FROM f_sales s
                   WHERE s.cust_id=c.cust_id
                     AND s.sales_dt>=DATE '2024-03-01');

-- NL 안티 유도(바깥 소량일 때)
SELECT /*+ USE_NL(s) LEADING(c s) */ c.cust_id
FROM   d_customer c
WHERE  NOT EXISTS (SELECT 1 FROM f_sales s
                   WHERE s.cust_id=c.cust_id
                     AND s.sales_dt>=DATE '2024-03-01');
```

### `COUNT(*)` 기반 “부재” 판정 → Anti Join으로 제거

```sql
-- 느린 패턴: 부재를 COUNT=0로 검사(행마다 집계 호출)
SELECT c.cust_id
FROM   d_customer c
WHERE  (SELECT COUNT(*)
        FROM   f_sales s
        WHERE  s.cust_id=c.cust_id
        AND    s.sales_dt>=DATE '2024-03-01') = 0;

-- 좋은 패턴: NOT EXISTS (ANTI JOIN 변환)
SELECT c.cust_id
FROM   d_customer c
WHERE  NOT EXISTS (SELECT 1
                   FROM f_sales s
                   WHERE s.cust_id=c.cust_id
                     AND s.sales_dt>=DATE '2024-03-01');
```
- **Aggregate Subquery Elimination**의 한 형태: `COUNT(*)=0` → **반존재성** → **ANTI JOIN**.

### Anti Join 주의점

- `NOT IN`은 **NULL 조심**(서브쿼리 집합에 NULL이면 전체 결과가 사라질 수 있음).
- 외부조인과 섞일 땐 **NULL 보존** 의미가 변형될 수 있어, 옵티마이저가 안티 변환을 회피하기도 한다.
- 의미 보존이 애매하면 해당 **QB에 `NO_QUERY_TRANSFORMATION`** 또는 표현을 **명시적 조인**으로 수동 재작성.

---

# 2) Aggregate Subquery Elimination (상관 집계 서브쿼리 제거)

**문제 유형**: SELECT-list 또는 WHERE 절의 **상관 집계**(예: `SUM/COUNT/MAX/MIN`) 때문에 **행마다** 집계를 수행 → CPU/I/O 과다.
**해법**: “집계를 **한 번만** 계산”하는 **사전 집계 + 조인**으로 바꾼다(= 서브쿼리 언네스트 + 집계 풀기).

## SELECT-list 스칼라 집계 제거

```sql
-- 느린 원형: 각 상품의 3월 매출 합을 스칼라 집계로(행마다 SUM 호출)
SELECT p.prod_id,
       (SELECT SUM(s.amount)
        FROM   f_sales s
        WHERE  s.prod_id = p.prod_id
        AND    s.sales_dt BETWEEN DATE '2024-03-01' AND DATE '2024-03-31') AS m_sum
FROM   d_product p
WHERE  p.category='ELEC';

-- 개선: 사전 집계 → 조인 (언네스트 기대)
SELECT /*+ UNNEST */ p.prod_id, ss.m_sum
FROM   d_product p
LEFT JOIN (
  SELECT s.prod_id, SUM(s.amount) m_sum
  FROM   f_sales s
  WHERE  s.sales_dt BETWEEN DATE '2024-03-01' AND DATE '2024-03-31'
  GROUP  BY s.prod_id
) ss
ON ss.prod_id = p.prod_id
WHERE p.category='ELEC';
```
- 효능: 스칼라 서브쿼리 **대량 호출 제거**.
- 계획: `HASH GROUP BY` → `HASH JOIN OUTER` 또는 `NL OUTER`.
- 스칼라 캐시가 있더라도 값 영역이 넓으면 미스가 많음 → 조인 방식이 보통 더 빠름.

## WHERE 절 집계 제거 (존재/비존재·상계/하계)

```sql
-- 느림: 고객별 3월 매출 합이 100만 이상
SELECT c.cust_id
FROM   d_customer c
WHERE  (SELECT SUM(s.amount)
        FROM   f_sales s
        WHERE  s.cust_id=c.cust_id
        AND    s.sales_dt BETWEEN :d1 AND :d2) >= 1000000;

-- 개선: 사전 집계 → HAVING → 조인
WITH agg AS (
  SELECT s.cust_id, SUM(s.amount) sum_amt
  FROM   f_sales s
  WHERE  s.sales_dt BETWEEN :d1 AND :d2
  GROUP  BY s.cust_id
  HAVING SUM(s.amount) >= 1000000
)
SELECT c.cust_id
FROM   d_customer c
JOIN   agg
ON     agg.cust_id = c.cust_id;
```
- `COUNT(*)=0/ >0` 류는 **(NOT) EXISTS/SEMI/ANTI JOIN**으로,
  `SUM/MAX/MIN` 비교류는 **사전 집계 + 조인**으로 풀자.

## “집계 + DISTINCT” 제거 (중복 제거의 위치 변경)

```sql
-- 원형: 주문 라인에서 고객별 상품종류 수(= DISTINCT prod_id) 필요
SELECT c.cust_id
FROM   d_customer c
WHERE  (SELECT COUNT(DISTINCT s.prod_id)
        FROM   f_sales s
        WHERE  s.cust_id=c.cust_id
        AND    s.sales_dt>=:d1) >= 10;

-- 개선: 먼저 중복 제거 → 고객별 집계 → 조인
WITH prod_set AS (
  SELECT DISTINCT s.cust_id, s.prod_id
  FROM   f_sales s
  WHERE  s.sales_dt>=:d1
),
agg AS (
  SELECT cust_id, COUNT(*) prod_cnt
  FROM   prod_set
  GROUP  BY cust_id
  HAVING COUNT(*) >= 10
)
SELECT c.cust_id
FROM   d_customer c
JOIN   agg ON agg.cust_id=c.cust_id;
```
- **DISTINCT의 위치**를 바꿔 입력량 축소 → 집계 비용 감소.

## 옵티마이저 주도 vs 수동 재작성

- Oracle은 많은 경우 **자동**으로 집계 서브쿼리를 언네스트/제거한다(버전/통계 의존).
- 자동이 불안정하면 **수동 재작성** + 힌트(`UNNEST`, `NO_UNNEST`, `MATERIALIZE/INLINE`, `LEADING/USE_*`)로 안정화.

---

# 3) Subquery Pushing: `PUSH_SUBQ`로 “먼저 좁히고” 조인하기

**아이디어**: 큰 테이블 `T`에 대해 `T.key IN (SELECT ... FROM S WHERE ...)` 등의 **서브쿼리 결과를 먼저 계산**하여
**작은 집합**(키 리스트)로 만든 뒤, 이걸 기준으로 **조인**하게 유도한다. 즉 **필터를 앞당겨** I/O를 줄인다.

## 기본 예제

```sql
-- 원형: 큰 F_SALES에서 브랜드/기간 조건을 'IN (subquery)'로 필터
SELECT s.*
FROM   f_sales s
WHERE  s.prod_id IN (
  SELECT p.prod_id
  FROM   d_product p
  WHERE  p.category='ELEC'
  AND    p.brand   ='B0'
)
AND    s.sales_dt BETWEEN DATE '2024-03-01' AND DATE '2024-03-31';

-- Pushing 유도
SELECT /*+ PUSH_SUBQ */ s.*
FROM   f_sales s
WHERE  s.prod_id IN (
  SELECT p.prod_id
  FROM   d_product p
  WHERE  p.category='ELEC'
  AND    p.brand   ='B0'
)
AND    s.sales_dt BETWEEN DATE '2024-03-01' AND DATE '2024-03-31';
```
- 효과: `p` 조건을 먼저 평가해 **작은 prod_id 집합**을 만들고,
  이를 이용해 `s`에 **세미조인 방식**으로 빠르게 진입(해시 세미/인덱스) → **랜덤 I/O 감소**.
- 비슷한 효과를 `UNNEST`로도 얻지만, `PUSH_SUBQ`는 “서브쿼리를 **더 먼저** 평가하라”는 **우선순위 신호**에 가깝다.
- 반대로 원치 않으면 `NO_PUSH_SUBQ`.

## View/Inline-View와 결합

```sql
-- 인라인 뷰 + 서브쿼리: 먼저 축소하고 조인
SELECT /*+ PUSH_SUBQ */
       s.sales_id, s.amount
FROM   f_sales s
JOIN  (SELECT p.prod_id
       FROM   d_product p
       WHERE  p.category='ELEC' AND p.brand='B0') v
ON     v.prod_id = s.prod_id
WHERE  s.sales_dt BETWEEN :d1 AND :d2;
```
- 옵티마이저가 자동으로 잘 하기도 하지만, 데이터 분포/통계상 오판 시 `PUSH_SUBQ`로 “**먼저 좁히기**”를 강조.

## FILTER → PUSH_SUBQ → HASH SEMI(또는 NL SEMI)

```sql
-- FILTER 패턴(비-언네스트)을 강제로 언네스트/푸시하여 조인으로
SELECT /*+ PUSH_SUBQ UNNEST USE_HASH(p) LEADING(p s) */
       s.*
FROM   f_sales s
WHERE  s.prod_id IN (SELECT p.prod_id
                     FROM   d_product p
                     WHERE  p.category='ELEC' AND p.brand='B0')
AND    s.sales_dt BETWEEN :d1 AND :d2;
```
- 계획 후보: `HASH JOIN SEMI`(p가 build, s가 probe)
- 빌드 입력이 작아 해시/블룸 필터가 잘 먹는다 → s 스캔 중 **조기 필터링**.

## 주의점

- **외부조인/NULL 보존**이 얽히면 푸시로 결과가 바뀔 수 있다 → 안전하지 않으면 옵티마이저가 회피.
- 뷰 머지/OR-EXPAND와의 **충돌**: 변환 우선순위에 따라 예상치 못한 플랜이 나올 수 있으므로,
  **QB 이름(`QB_NAME`) + `NO_MERGE/NO_EXPAND`**로 범위를 좁혀 단계별 검증.

---

# 4) 세 가지를 한 번에: Anti Join + 집계 제거 + Push
### 시나리오: “3월 동안 **구매 이력이 전혀 없는** ELEC/B0 고객”

```sql
-- 느린/복잡한 원형: 집계/서브쿼리 혼재
SELECT c.cust_id
FROM   d_customer c
WHERE  c.tier='VIP'
AND    (SELECT COUNT(*)
        FROM   f_sales s
        WHERE  s.cust_id=c.cust_id
        AND    s.prod_id IN (SELECT p.prod_id
                             FROM   d_product p
                             WHERE  p.category='ELEC' AND p.brand='B0')
        AND    s.sales_dt BETWEEN :d1 AND :d2) = 0;

-- 최적 재작성: (a) 제품 서브쿼리 PUSH_SUBQ로 먼저 축소
--               (b) COUNT(*)=0 → NOT EXISTS(ANTI JOIN)로 집계 제거
SELECT /*+ PUSH_SUBQ UNNEST USE_HASH(p) LEADING(p c) */
       c.cust_id
FROM   d_customer c
WHERE  c.tier='VIP'
AND    NOT EXISTS (
  SELECT 1
  FROM   f_sales s
  JOIN  (SELECT p.prod_id
         FROM   d_product p
         WHERE  p.category='ELEC' AND p.brand='B0') p
    ON  p.prod_id = s.prod_id
  WHERE  s.cust_id = c.cust_id
  AND    s.sales_dt BETWEEN :d1 AND :d2
);
```
- **핵심 변화**
  - `COUNT(*)=0` 제거 → **ANTI JOIN**
  - 제품 필터를 **먼저** 평가 → 작은 prod 집합으로 **푸시**
  - 필요하면 `JOIN FILTER`까지 활성 → `f_sales` 스캔량 급감

---

# 5) 실행계획·진단 포인트

```sql
-- 실측 계획 + 프레디킷 이동/노트 확인
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(
  NULL,NULL,'ALLSTATS LAST +PREDICATE +ALIAS +NOTE'));
```
- Anti Join 성공: `HASH JOIN ANTI` / `NESTED LOOPS ANTI` / `MERGE JOIN ANTI`.
- 집계 제거 성공: 스칼라 서브쿼리 사라지고 `HASH GROUP BY`(사전집계) + 조인으로 나타남.
- Subquery Pushing: Notes/PREDICATE에 **push/unnest/transform** 흔적, `JOIN FILTER CREATE/USE`가 보일 수 있다.
- 항상 **E-Rows vs A-Rows**로 비용 오판을 체크하고, 필요 시 **통계(히스토그램/확장 통계)** 보정.

---

# 6) 체크리스트(실무)

- [ ] **부재 판정**은 `NOT EXISTS`(→ **ANTI JOIN**)로. `NOT IN`은 NULL 주의.
- [ ] **COUNT(*)=0/ >0** 패턴은 **집계 서브쿼리 제거**로 변환.
- [ ] SELECT-list 상관 집계는 **사전 집계 + 조인**으로 한 번만 계산.
- [ ] 큰 테이블 앞에 서브쿼리로 **키 집합 축소**가 가능하면 `PUSH_SUBQ`(+`UNNEST`) 고려.
- [ ] 조인 방식/순서 제어: `LEADING/ORDERED`, `USE_HASH/USE_NL/USE_MERGE`, `SWAP_JOIN_INPUTS`.
- [ ] 파티션 프루닝/인덱스 선두 정합성 점검(특히 기간 조건).
- [ ] 의미 변화 위험(외부조인·NULL 보존) 시 `NO_QUERY_TRANSFORMATION` 또는 수동 재작성.
- [ ] 결과/성능 검증은 **실측 계획**과 **세션 통계**(logical/physical reads)로.

---

## 부록 A. 미니 실습 스니펫

### A-1) Anti Join 강제/비교

```sql
-- 1) Hash Anti
EXPLAIN PLAN FOR
SELECT /*+ USE_HASH(s) LEADING(s c) */
       c.cust_id
FROM   d_customer c
WHERE  NOT EXISTS (SELECT 1 FROM f_sales s
                   WHERE s.cust_id=c.cust_id
                     AND s.sales_dt>=DATE '2024-03-01');
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);

-- 2) NL Anti
EXPLAIN PLAN FOR
SELECT /*+ USE_NL(s) LEADING(c s) */
       c.cust_id
FROM   d_customer c
WHERE  NOT EXISTS (SELECT 1 FROM f_sales s
                   WHERE s.cust_id=c.cust_id
                     AND s.sales_dt>=DATE '2024-03-01');
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);
```

### A-2) 집계 서브쿼리 제거(SELECT-list)

```sql
EXPLAIN PLAN FOR
SELECT /*+ UNNEST */
       p.prod_id, ss.m_sum
FROM   d_product p
LEFT JOIN (
  SELECT s.prod_id, SUM(s.amount) m_sum
  FROM   f_sales s
  WHERE  s.sales_dt BETWEEN DATE '2024-03-01' AND DATE '2024-03-31'
  GROUP  BY s.prod_id
) ss
ON ss.prod_id = p.prod_id
WHERE p.category='ELEC';
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);
```

### A-3) Subquery Pushing

```sql
EXPLAIN PLAN FOR
SELECT /*+ PUSH_SUBQ UNNEST USE_HASH(p) LEADING(p s) */
       s.*
FROM   f_sales s
WHERE  s.prod_id IN (
  SELECT p.prod_id
  FROM   d_product p
  WHERE  p.category='ELEC' AND p.brand='B0'
)
AND    s.sales_dt BETWEEN DATE '2024-03-01' AND DATE '2024-03-31';
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);
```

---

## 부록 B. FAQ

**Q1. `PUSH_SUBQ`와 `UNNEST`의 차이는?**
- `UNNEST`는 **서브쿼리를 조인으로 풀어내는 변환 자체**를 지시.
- `PUSH_SUBQ`는 “그 서브쿼리를 **먼저 평가**(= 앞단에서 축소)하려는 의도”를 더 강하게 전달.
  실무에선 **둘을 함께** 써서 *“먼저 축소 → 조인”* 전략을 안정화하는 경우가 많다.

**Q2. Anti Join에서 `JOIN FILTER`는 언제 생기나?**
- 보통 **Hash Anti**에서 내부(Build) 입력의 키 집합으로 **Bloom Filter**를 만들어
  바깥(Probe) 스캔 중 **조기 필터링**할 때 `JOIN FILTER CREATE/USE`가 보인다.

**Q3. 왜 자동 변환이 가끔 실패하나?**
- 통계/히스토그램 미비로 **카디널리티 오판**, 외부조인·NULL 보존 의미 충돌, 변환 간 **우선순위 충돌** 때문.
  → **QB 이름**을 붙여 변환 범위를 좁히고, **힌트로 안정화**하며, **실측**으로 확인하라.

---

### 결론

- **Anti Join**은 “없음을 증명”하는 최단 경로다. `COUNT(*)=0` 류는 **집계 제거 → Anti**로 바꿔라.
- **Aggregate Subquery Elimination**은 상관 집계의 **행단위 호출**을 **한 번의 집계 + 조인**으로 끝낸다.
- **Subquery Pushing**은 **먼저 좁혀** 대상을 작게 만들고, 그 뒤에 **세미/안티/일반 조인**으로 빠르게 간다.
- 언제나 **실측 플랜**과 **세션 통계**로 개선 폭을 확인하라. 그게 정답이다.
