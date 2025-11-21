---
layout: post
title: DB 심화 - 조건절 Pushing
date: 2025-11-18 20:25:23 +0900
category: DB 심화
---
# 조건절 Pushing 완전 가이드 (Oracle CBO 관점)

**주제:**
- **Predicate Pushdown(조건절 푸시다운)**: 필터를 **더 아래 단계**(뷰/인라인 뷰/서브쿼리/베이스 테이블/파티션)로 밀어 넣어 읽는 데이터 량을 최소화
- **Predicate Pullup(조건절 풀업/호이스트)**: 안전할 때 **상위 단계**로 끌어올려 **변환·조인 재배치** 여지를 넓힘
- **Join Predicate Pushdown(조인 조건 푸시다운)**: 조인 동등조건·전이성으로 **상대 테이블**에 동일 필터를 **전파**하거나, **Join Filter(Bloom)**로 스캔 단계에서 **조기차단**

---

## 샘플 스키마(요약)

```sql
-- 차원/사실 모델
CREATE TABLE d_customer (
  cust_id NUMBER PRIMARY KEY,
  region  VARCHAR2(8) NOT NULL,
  tier    VARCHAR2(8) NOT NULL
);

CREATE TABLE d_product (
  prod_id  NUMBER PRIMARY KEY,
  category VARCHAR2(16) NOT NULL,
  brand    VARCHAR2(16) NOT NULL
);

CREATE TABLE f_sales (
  sales_id NUMBER PRIMARY KEY,
  cust_id  NUMBER NOT NULL,
  prod_id  NUMBER NOT NULL,
  sales_dt DATE   NOT NULL,
  qty      NUMBER NOT NULL,
  amount   NUMBER(12,2) NOT NULL
);

-- 주요 인덱스(예시)
CREATE INDEX ix_fs_cust_dt  ON f_sales(cust_id, sales_dt);
CREATE INDEX ix_fs_prod_dt  ON f_sales(prod_id, sales_dt);
CREATE INDEX ix_prod_cat_br ON d_product(category, brand, prod_id);
```

관찰은 항상:
```sql
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(
  NULL, NULL, 'ALLSTATS LAST +PREDICATE +ALIAS +NOTE'));
```
- `Predicate Information`에 **pushed predicates**가 명시됩니다.
- `Notes`에 **Predicate Move-Around/Pushdown** 흔적이 보일 수 있습니다.

---

# Predicate Pushdown — 개념과 이득
### 정의

- **Pushdown**: WHERE/HAVING/ON의 조건을 **가능한 한 데이터 원천 가까이** 이동.
  - 인라인 뷰/서브쿼리 **안쪽**, 베이스 테이블 **액세스 전**, 파티션 **프루닝** 지점으로 밀어 넣음.
- **효과**
  1) **읽는 블록 감소** → I/O 절감
  2) **카디널리티 축소** → 이후 조인·집계가 가벼워짐
  3) **인덱스/파티션 프루닝** 촉진
  4) (해시) **Join Filter(Bloom)**와 결합 시 **프로브 전 조기 차단**

### 제어 힌트(대표)

- `PUSH_PRED` / `NO_PUSH_PRED` — (인라인 뷰/서브쿼리) **필터 이동** 선호/억제
- `MERGE(qb)` / `NO_MERGE(qb)` — **View Merging**(병합) 허용/금지(병합 후 푸시다운 여지↑)
- `PUSH_SUBQ` / `NO_PUSH_SUBQ` — **서브쿼리**를 먼저 계산/축소(푸시)
- 조합: `LEADING/ORDERED` + `USE_HASH/USE_NL`(조인 전략 고정), `SWAP_JOIN_INPUTS` 등

---

# Pushdown 기본 예제 (인라인 뷰)
### 병합 없이 뷰에 조건 남김 (덜 좋은 패턴)

```sql
-- ELEC/B0 제품만 뽑아 s와 조인하되, 상위 WHERE만 사용
SELECT s.sales_id, s.amount
FROM   f_sales s
JOIN  (SELECT p.prod_id
       FROM   d_product p
       WHERE  p.category='ELEC' AND p.brand='B0') v
ON     v.prod_id = s.prod_id
WHERE  s.sales_dt BETWEEN DATE '2024-03-01' AND DATE '2024-03-31';
```
- 옵티마이저가 자동 병합·푸시를 못하면, 플랜에 `VIEW v`가 남고 s 조인이 비효율일 수 있음.

### 뷰 병합 + 푸시다운 유도 (좋은 패턴)

```sql
SELECT /*+ MERGE(v) PUSH_PRED(v) USE_HASH(v) LEADING(v s) */
       s.sales_id, s.amount
FROM   f_sales s
JOIN  (SELECT p.prod_id
       FROM   d_product p
       WHERE  p.category='ELEC' AND p.brand='B0') v
ON     v.prod_id = s.prod_id
WHERE  s.sales_dt BETWEEN :d1 AND :d2;
```
- `MERGE(v)`로 인라인 뷰 평탄화 → `PUSH_PRED(v)`로 v의 조건을 s 접근 전에도 활용(인덱스/프루닝).
- 결과: `HASH JOIN [SEMI/INNER]`, `JOIN FILTER CREATE/USE`가 보이면 성공 신호.

---

# Subquery Pushdown (PUSH_SUBQ) vs Unnesting
### PUSH_SUBQ — “먼저 좁혀라”

```sql
-- 큰 사실 테이블 s에 진입하기 전에 prod 서브쿼리를 먼저 좁힘
SELECT /*+ PUSH_SUBQ UNNEST USE_HASH(p) LEADING(p s) */
       s.*
FROM   f_sales s
WHERE  s.prod_id IN (
  SELECT p.prod_id
  FROM   d_product p
  WHERE  p.category='ELEC' AND p.brand='B0'
)
AND    s.sales_dt BETWEEN :d1 AND :d2;
```
- **PUSH_SUBQ**: 서브쿼리 결과(키 집합)를 **먼저** 만들고 **세미조인/해시**로 붙임.
- 보통 `HASH JOIN SEMI` + `JOIN FILTER`가 생성되어 s 스캔량을 크게 줄입니다.

### Unnesting과의 관계

- `UNNEST`는 **서브쿼리를 조인 형태**로 **재작성**.
- `PUSH_SUBQ`는 “그 서브쿼리 **평가를 앞**으로 당겨” **먼저 축소**하라는 힌트.
- 함께 쓰면 “**앞에서 축소 → 조인**” 전략을 안정화.

---

# Join Predicate Pushdown (전이성/조인 필터/블룸)
### 전이성(Transitivity)으로 상호 전파

```sql
-- 전이성: c.cust_id = s.cust_id, c.cust_id BETWEEN 100 AND 200
-- → s.cust_id BETWEEN 100 AND 200 으로도 전파(푸시)
SELECT /*+ USE_NL(s) LEADING(c s) */
       s.sales_id
FROM   d_customer c
JOIN   f_sales s
ON     s.cust_id = c.cust_id
WHERE  c.cust_id BETWEEN 100 AND 200
AND    s.sales_dt BETWEEN :d1 AND :d2;
```
- 플랜의 Predicate에 `access("S"."CUST_ID" BETWEEN 100 AND 200)`가 보이면 **푸시 성공**.
- 이로써 s의 **인덱스 범위**가 좁아지고 **파티션 프루닝**이 유발될 수 있습니다.

### — 해시 조인에서의 스캔 차단

```sql
-- 해시 조인 빌드 입력의 키 집합으로 Bloom Filter 생성 → 프로브 테이블 스캔 중 조기 차단
SELECT /*+ USE_HASH(p) LEADING(p s) */
       COUNT(*)
FROM   f_sales s
JOIN   d_product p
ON     p.prod_id = s.prod_id
WHERE  p.category='ELEC';
```
- 실행계획에 `JOIN FILTER CREATE`(p), `JOIN FILTER USE`(s)가 보이면 **블룸 필터**가 생성/사용됨.
- 효과: s 스캔 중 **가능성 낮은 행**을 **인덱스/테이블 접근 전에** 걸러냄 → 물리·논리 읽기 절감.

---

# Pullup(호이스트) — 언제, 왜 끌어올리나
### Pullup의 의의

- Pushdown이 항상 정답은 아님. 어떤 조건은 상위에서 처리해야 **의미 보존/조인 재배치**가 쉬움.
- 옵티마이저는 **Predicate Move-Around**에서 **Push/Pull**을 **비용기반**으로 결정.

### 예: 외부조인과 WHERE 절 조건

```sql
-- LEFT OUTER JOIN에서, 내부 테이블(s)에 대한 WHERE 필터는 위치에 따라 "내부를 INNER로 바꿔버릴" 수 있음
-- (1) 안전한 위치: ON 절에 두기(LEFT 유지)
SELECT c.cust_id, s.amount
FROM   d_customer c
LEFT JOIN f_sales s
  ON   s.cust_id = c.cust_id
 AND  s.sales_dt BETWEEN :d1 AND :d2
WHERE c.region='KOR';

-- (2) 위험: s.amount > 100 을 WHERE에 두면 LEFT→INNER로 변질
--    → 옵티마이저는 이를 막기 위해 해당 필터를 "끌어올리거나(재배치)" 위치를 조정
SELECT c.cust_id, s.amount
FROM   d_customer c
LEFT JOIN f_sales s
  ON   s.cust_id = c.cust_id
WHERE c.region='KOR'
  AND s.amount > 100; -- NULL 보존 깨짐
```
- **Pullup/재배치**는 이렇게 **의미 변경**을 피하고 최적화를 가능케 하려는 안전장치 역할도 합니다.
- 외부조인일 때는 **필터 위치**가 매우 중요: 내부 테이블 필터는 **ON절**에 두고, 외부 테이블 필터는 **WHERE절**.

### 예: GROUP BY/HAVING과의 상호작용

```sql
-- HAVING SUM(s.amount) > 0 을 푸시하려면 "사전집계"로 형태를 바꿔야
-- (복합 뷰 머징/집계 푸시다운과 결합)
SELECT c.cust_id
FROM   d_customer c
JOIN  (SELECT s.cust_id, SUM(s.amount) sum_amt
       FROM   f_sales s
       WHERE  s.sales_dt BETWEEN :d1 AND :d2
       GROUP  BY s.cust_id) a
ON     a.cust_id = c.cust_id
WHERE  c.region='KOR'
AND    a.sum_amt > 0;
```
- 옵티마이저가 판단해 **집계 위치 재배치(병합) + 조건 푸시/풀업**을 함께 적용할 수 있습니다.
- 불가능/위험하면 **그대로 유지**하거나 **임시 물리화**로 처리.

---

# Pushdown의 수준들(어디까지 내려가나)

1) **뷰 내부**: 인라인 뷰/CTE 안쪽으로 이동 (`PUSH_PRED`, `MERGE`)
2) **베이스 테이블 접근 직전**: 인덱스/테이블 액세스 **access predicate**로 전환
3) **파티션 프루닝**: 파티션 키 조건을 **access predicate**로 만들어 **불필요 파티션 배제**
4) **스토리지/엔진 레벨**: 해시 조인 **Join Filter(Bloom)**로 **스캔 단계**에서 **조기 차단**
5) **DB 링크/외부 테이블**: 가능한 필터는 **원격/스토리지로 푸시**(네트워크·I/O 감소) — 상황/옵션 의존

---

# Pushdown이 막히는 경우 (제약 & 주의)

- **외부조인 NULL 보존**: 내부쪽 필터를 WHERE에 두면 의미 변질 → **ON절**로 이동/재배치 필요
- **분석함수/ROWNUM/CONNECT BY**: **순서/상태 의존** → 푸시 제한
- **비결정적 함수**(예: `DBMS_RANDOM`, `SYSDATE` 등)나 **부작용 있는 함수**: 푸시 제한
- **데이터 타입 변환/함수 가공**: `WHERE TO_CHAR(s.sales_dt,'YYYY')='2024'`처럼 **컬럼 가공**은 **인덱스/푸시** 막음
  - 해결: **SARGABLE**(인덱스 친화) 형태로 변환: `WHERE s.sales_dt >= DATE '2024-01-01' AND s.sales_dt < DATE '2025-01-01'`
- **OR 묶임**: 하나의 분기가 푸시를 막으면 전체가 막힘 → `USE_CONCAT`(OR-EXPAND)로 분기 분해 후 각 분기에서 푸시
- **DISTINCT/집계/UNION**: 의미 보존 가능한 범위에서만 가능(복합 뷰 머징 주제와 연결)

---

# 실전 튜닝 시나리오

## 파티션 프루닝을 노린 푸시

```sql
-- f_sales가 RANGE PARTITION(sales_dt)라고 가정
SELECT /*+ MERGE(v) PUSH_PRED(v) */
       COUNT(*)
FROM   f_sales s
JOIN  (SELECT p.prod_id
       FROM   d_product p
       WHERE  p.brand='B0') v
ON     v.prod_id = s.prod_id
WHERE  s.sales_dt BETWEEN :d1 AND :d2;
```
- `MERGE + PUSH_PRED`로 v의 필터와 WHERE의 기간 조건을 s 접근 전 **access predicate**로 만들어
  **필요 파티션만** 스캔하도록 유도.

## 전이성 + 인덱스 범위 최적화

```sql
-- c에서 cust_id 범위를 주면 s에도 동일 범위가 푸시되어 ix_fs_cust_dt 활용
SELECT /*+ USE_NL(s) LEADING(c s) */
       COUNT(*)
FROM   d_customer c
JOIN   f_sales s
ON     s.cust_id = c.cust_id
WHERE  c.cust_id BETWEEN :lo AND :hi
AND    s.sales_dt BETWEEN :d1 AND :d2;
```

## 블룸 필터에 의한 스캔 차단

```sql
-- ELEC 카테고리만 허용하는 Join Filter를 만들어 s 스캔 중 조기 차단
SELECT /*+ USE_HASH(p) LEADING(p s) */
       SUM(s.amount)
FROM   f_sales s
JOIN   d_product p
ON     p.prod_id = s.prod_id
WHERE  p.category='ELEC'
AND    s.sales_dt BETWEEN :d1 AND :d2;
```
- `JOIN FILTER CREATE/USE`가 보이면 **Pushdown이 스캔 단계**까지 내려간 것.

## OR-EXPAND로 푸시 회복

```sql
-- OR가 푸시를 막으면 분기 분해
SELECT /*+ USE_CONCAT */
       s.*
FROM   f_sales s
WHERE  (s.prod_id IN (SELECT p.prod_id FROM d_product p WHERE p.brand='B0'))
   OR  (s.cust_id IN (SELECT c.cust_id FROM d_customer c WHERE c.tier='VIP'));
```
- `USE_CONCAT`로 **UNION ALL 분기** 생성 → 각 분기에서 **독립적 푸시** 가능.

---

# Pullup이 유리할 때 (재배치로 의미·비용 보호)

- 외부조인에서 **WHERE의 내부 필터**를 **ON절**로 되돌리기(LEFT→INNER 변질 방지)
- 뷰 내부 필터가 조인 순서를 망가뜨리면, 상위에서 처리하도록 **끌어올려** 재배치
- 집계·DISTINCT와 얽힌 경우 **사전집계 위치**를 바꾸기 위해 **조건을 상하로 이동**(복합 뷰 머징)

예:
```sql
-- 잘못된 위치(의미 변경 위험)
SELECT c.cust_id
FROM   d_customer c LEFT JOIN f_sales s
ON     s.cust_id = c.cust_id
WHERE  s.amount > 0;    -- LEFT → INNER 변질

-- 수정(풀업/재배치 개념 반영)
SELECT c.cust_id
FROM   d_customer c LEFT JOIN f_sales s
ON     s.cust_id = c.cust_id
AND    s.amount > 0;
```

---

# 관찰·검증 루틴

```sql
-- 실행 전후 세션 통계 체크(예: logical/physical reads)
SELECT sn.name, ms.value
FROM   v$mystat ms JOIN v$statname sn ON ms.stat#=sn.stat#
WHERE  sn.name IN ('session logical reads','physical reads','consistent gets');

-- 쿼리 실행 (힌트/작성법 변경 전후 비교)

-- 실측 실행계획
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(
  NULL, NULL, 'ALLSTATS LAST +PREDICATE +NOTE +ALIAS'));
```
- **Access/Filter Predicates**에서 **어디까지 내려갔는지** 확인
- `JOIN FILTER`/`Predicate Pushdown` 노트 확인
- `E-Rows vs A-Rows`로 **카디널리티 오판**(=비용오판)을 점검

---

# 체크리스트

- [ ] 가능한 한 **SARGABLE**하게 작성(컬럼 가공 금지, 범위는 컬럼 좌측)
- [ ] 인라인 뷰/서브쿼리는 **`MERGE` + `PUSH_PRED`/`PUSH_SUBQ`**로 **앞에서 축소**
- [ ] 전이성으로 **조인 상대 테이블**에 필터 **전파**되는지 확인
- [ ] **외부조인**은 **NULL 보존** 깨지지 않게 필터 위치 점검(내부 필터는 **ON절**)
- [ ] OR은 `USE_CONCAT`로 분해해 각 분기별 **푸시 회복**
- [ ] 해시 조인에서는 **Join Filter(Bloom)**가 생성되는지 관찰
- [ ] 통계/히스토그램/확장 통계로 **카디널리티 오판 방지**
- [ ] 결과/성능은 **실측 플랜**·**세션 통계**로 검증

---

## 부록 A. 미니 실습 묶음

### A-1. 뷰 푸시다운 vs 비푸시

```sql
-- (1) 비머지/비푸시
SELECT /*+ NO_MERGE(v) NO_PUSH_PRED(v) */
       COUNT(*)
FROM   f_sales s
JOIN  (SELECT p.prod_id FROM d_product p
       WHERE  p.category='ELEC' AND p.brand='B0') v
ON     v.prod_id = s.prod_id
WHERE  s.sales_dt BETWEEN :d1 AND :d2;

-- (2) 머지+푸시
SELECT /*+ MERGE(v) PUSH_PRED(v) USE_HASH(v) LEADING(v s) */
       COUNT(*)
FROM   f_sales s
JOIN  (SELECT p.prod_id FROM d_product p
       WHERE  p.category='ELEC' AND p.brand='B0') v
ON     v.prod_id = s.prod_id
WHERE  s.sales_dt BETWEEN :d1 AND :d2;
```

### A-2. 서브쿼리 푸시 + 언네스트

```sql
SELECT /*+ PUSH_SUBQ UNNEST USE_HASH(p) LEADING(p s) */
       SUM(s.amount)
FROM   f_sales s
WHERE  s.prod_id IN (SELECT p.prod_id
                     FROM   d_product p
                     WHERE  p.category='ELEC' AND p.brand='B0')
AND    s.sales_dt BETWEEN :d1 AND :d2;
```

### A-3. 전이성(Join Predicate Pushdown)

```sql
SELECT /*+ USE_NL(s) LEADING(c s) */
       COUNT(*)
FROM   d_customer c
JOIN   f_sales s
ON     s.cust_id = c.cust_id
WHERE  c.cust_id BETWEEN :lo AND :hi
AND    s.sales_dt BETWEEN :d1 AND :d2;
-- s.cust_id BETWEEN :lo AND :hi 가 access predicate로 내려갔는지 확인
```

### A-4. 외부조인 필터 위치 교정

```sql
-- 안전한 위치: ON
SELECT c.cust_id, s.amount
FROM   d_customer c
LEFT JOIN f_sales s
  ON   s.cust_id = c.cust_id
 AND   s.amount  > 100
WHERE  c.region  = 'KOR';
```

---

## 부록 B. FAQ

**Q1. `PUSH_PRED`와 `PUSH_SUBQ` 차이는?**
- `PUSH_PRED`: **인라인 뷰/서브쿼리 내부로 필터 이동**을 선호.
- `PUSH_SUBQ`: **서브쿼리 자체를 먼저 평가**해 작은 키 집합으로 축소 후 조인.

**Q2. 전이성은 언제 동작하나요?**
- 주로 **등치 조인** + 한쪽에 **범위/상수 조건**이 있을 때. 통계/형변환/함수 가공에 막힐 수 있습니다.

**Q3. 왜 내 쿼리는 푸시가 안 되나요?**
- 외부조인/분석함수/ROWNUM/CONNECT BY/컬럼 가공/NULL 보존 등 **제약** 때문.
  → **작성 교정**(SARGABLE), **OR-EXPAND**, **힌트**, **수동 재작성**을 고려하세요.

---

### 결론

- **Predicate Pushdown**은 I/O와 행 개수를 **근본적으로 줄이는** 1차 최적화입니다.
- **Pullup/재배치**는 의미를 지키면서 **변환·조인 재배치**를 가능하게 하는 보완축입니다.
- **Join Predicate Pushdown**(전이성·블룸)은 조인 단계에서의 **조기 차단**으로 **스캔 비용**을 급감시킵니다.
- 최종 답은 항상 **실측**: `DBMS_XPLAN … ALLSTATS LAST +PREDICATE +NOTE`와 세션 통계로 확인하세요.
