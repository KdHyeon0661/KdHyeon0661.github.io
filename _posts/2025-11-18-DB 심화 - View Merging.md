---
layout: post
title: DB 심화 - View Merging
date: 2025-11-18 19:25:23 +0900
category: DB 심화
---
# View Merging 완전 가이드 (Oracle CBO 관점)

**주제 요약**
- **View Merging이란?**: 인라인 뷰(서브쿼리·WITH 쿼리)를 상위 쿼리로 **병합**하여 **조인 순서·액세스 경로** 탐색공간을 넓히는 **비용기반 쿼리 변환(CQT)**
- **단수(Simple) 뷰 머징**: 단일 테이블·단순 조작(프로젝션/필터) 중심의 뷰 병합
- **복합(Complex) 뷰 머징**: `GROUP BY/AGG/DISTINCT/UNION ALL` 등을 포함한 **집계/합성** 인라인 뷰를 병합(집계 푸시다운/조인 재배치 포함)
- **왜 CQT가 필요한가**: 병합 여부가 성능을 **수배~수십배** 갈라놓음. 정답은 **데이터 분포·통계·카디널리티**에 달림
- **머징 불가 뷰의 처리**: `NO_MERGE/MATERIALIZE`/`INLINE`/임시테이블·MV·결과 캐시 등 **대안 전략**

---

## 실습 스키마 (요약)

```sql
-- 차원/사실 구조
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

-- 인덱스 (예)
CREATE INDEX ix_fs_cust_dt  ON f_sales(cust_id, sales_dt);
CREATE INDEX ix_fs_prod_dt  ON f_sales(prod_id, sales_dt);
CREATE INDEX ix_prod_cat_br ON d_product(category, brand, prod_id);

-- 샘플 데이터는 생략 가능. 실행계획 비교 중심으로 진행.
```

---

# View Merging이란?

**정의**: 옵티마이저가 실행계획 생성 전에 **쿼리 변환(Query Transformation)** 단계에서 **인라인 뷰를 상위 쿼리에 통합**하여 **조인 그래프를 평탄화**(Flatten) 하는 것.
- **효과**
  1) 상위/하위 경계를 없애 **조인 순서·방법(NL/HJ/SMJ)** 선택폭 확대
  2) **프레디킷 푸시다운**/전이·인덱스 사용/파티션 프루닝 촉진
  3) **집계 푸시다운**(복합 머징)으로 **I/O·CPU 대폭 절감**
- **제어 힌트**
  - 강제: `MERGE` / 금지: `NO_MERGE` (QB 이름과 함께 사용 권장)
  - WITH 인라인/물리화: `INLINE` / `MATERIALIZE`
  - 전역 금지: `NO_QUERY_TRANSFORMATION` (디버깅용)

> 용어: Oracle 문맥에서 *View Merging*은 **인라인 뷰·서브쿼리·WITH 쿼리**까지 포괄.

---

# 단수(Simple) View Merging

단일 테이블 기반, 단순 프로젝션/필터/조인만 포함하는 뷰의 병합.

## 기본 예시 — 단순 필터 뷰

```sql
-- 인라인 뷰 v: ELEC/B0 필터
SELECT /*+ qb_name(main) */
       s.sales_id, s.amount
FROM   f_sales s
JOIN  (SELECT /*+ qb_name(v) */
              p.prod_id
       FROM   d_product p
       WHERE  p.category='ELEC'
       AND    p.brand   ='B0') v
ON     v.prod_id = s.prod_id
WHERE  s.sales_dt BETWEEN DATE '2024-03-01' AND DATE '2024-03-31';
```

- **병합 전**(강제 비머지):
```sql
-- 물리화 유도
SELECT /*+ qb_name(main) */
       s.sales_id, s.amount
FROM   f_sales s
JOIN  (SELECT /*+ qb_name(v) NO_MERGE(v) MATERIALIZE */
              p.prod_id
       FROM   d_product p
       WHERE  p.category='ELEC' AND p.brand='B0') v
ON v.prod_id = s.prod_id
WHERE s.sales_dt BETWEEN DATE '2024-03-01' AND DATE '2024-03-31';
```
  - 계획: `VIEW`(TEMP TABLE TRANSFORMATION) → v를 먼저 만들어 디스크(Temp)에 저장 → s와 조인
  - 장점: v가 매우 작으면 유리할 수 있음(한번 만들고 여러 번 사용 시 특히)
  - 단점: **조인 순서 제한**, 추가 I/O(Temp)

- **병합 후**(기본/강제):
```sql
-- 병합 강제
SELECT /*+ MERGE(v) qb_name(main) */
       s.sales_id, s.amount
FROM   f_sales s
JOIN  (SELECT /*+ qb_name(v) */
              p.prod_id
       FROM   d_product p
       WHERE  p.category='ELEC' AND p.brand='B0') v
ON v.prod_id = s.prod_id
WHERE s.sales_dt BETWEEN DATE '2024-03-01' AND DATE '2024-03-31';
```
  - 계획: `HASH JOIN [SEMI/INNER]` 또는 `NESTED LOOPS` 직접 구성 (뷰 노드 소거)
  - **프레디킷 푸시**로 s 쪽 인덱스/파티션 프루닝 결합이 용이

> **포인트**: 단순 뷰는 대개 자동 머징. `NO_MERGE`가 없는데 `VIEW` 노드가 보이면 **다른 제약(외부조인/분석함수 등)**을 의심.

## 단순 뷰 + 조인 순서 최적화

```sql
-- ELEC/B0 프로덕트를 먼저 드라이빙하여 s 프로브(해시세미/조인필터 기대)
SELECT /*+ MERGE(v) USE_HASH(v) LEADING(v s) */
       s.sales_id, s.amount
FROM   f_sales s
JOIN  (SELECT /*+ qb_name(v) */ p.prod_id
       FROM   d_product p
       WHERE  p.category='ELEC' AND p.brand='B0') v
ON v.prod_id = s.prod_id
WHERE s.sales_dt BETWEEN :d1 AND :d2;
```
- 병합 후 `LEADING`/`USE_HASH` 등 **일반 힌트**가 **그대로 적용** 가능 → 탐색 공간 확대가 핵심 가치.

---

# 복합(Complex) View Merging

**집계/Distinct/Union All** 등 “무거운” 인라인 뷰를 병합해 **집계 푸시다운**/**조인 재배치**를 가능하게 하는 고급 변환.

## — 핵심 사례

```sql
-- 원형: 3월 기간 고객별 매출합을 사전 집계 후 고객과 조인
SELECT c.cust_id, a.sum_amt
FROM   d_customer c
JOIN  (SELECT /*+ qb_name(agg) */
              s.cust_id, SUM(s.amount) sum_amt
       FROM   f_sales s
       WHERE  s.sales_dt BETWEEN DATE '2024-03-01' AND DATE '2024-03-31'
       GROUP  BY s.cust_id) a
ON     a.cust_id = c.cust_id
WHERE  c.region='KOR';
```

- **비머지(물리화) 강제**
```sql
SELECT c.cust_id, a.sum_amt
FROM   d_customer c
JOIN  (SELECT /*+ qb_name(agg) NO_MERGE(agg) MATERIALIZE */
              s.cust_id, SUM(s.amount) sum_amt
       FROM   f_sales s
       WHERE  s.sales_dt BETWEEN :d1 AND :d2
       GROUP  BY s.cust_id) a
ON a.cust_id = c.cust_id
WHERE c.region='KOR';
```
  - 장점: a가 작고 **여러 번 재사용**되면 유리
  - 단점: a 생성 비용(Temp I/O), **조인 순서·푸시다운 제약**

- **복합 뷰 머징(집계 푸시)**
```sql
-- 병합 유도: agg를 상위로 끌어올리거나, 조건을 아래로 내려 보정
SELECT /*+ MERGE(agg) */
       c.cust_id, SUM(s.amount) sum_amt
FROM   d_customer c
JOIN   f_sales s
ON     s.cust_id = c.cust_id
WHERE  s.sales_dt BETWEEN :d1 AND :d2
AND    c.region = 'KOR'
GROUP  BY c.cust_id;
```
  - 의미 동일(조심: 의미 보존 확인 필요)
  - 효과: `c.region='KOR'`가 **조인 전부터 반영**되어 s 액세스 축소(프루닝/조인필터),
    집계는 **필요 최소 행**만 대상으로 수행 → **I/O·CPU 감소**

> **핵심**: **복합 뷰 머징(CVM)**은 집계/Distinct가 얽힌 뷰에서도 **의미를 보존**하는 한도에서 **집계 위치**를 조정한다.
> Oracle은 버전대마다 집계 푸시/조인 재배치 범위를 확장해 왔다(카디널리티·통계 정확성이 필수).

## DISTINCT/UNION ALL 포함 뷰

```sql
-- DISTINCT 뷰: prod_id 집합을 만들고 조인
SELECT s.*
FROM   f_sales s
JOIN  (SELECT /*+ qb_name(v) */
              DISTINCT p.prod_id
       FROM   d_product p
       WHERE  p.category='ELEC') v
ON v.prod_id = s.prod_id;

-- 병합 유도 후 플랜 관찰
SELECT /*+ MERGE(v) */
       s.*
FROM   f_sales s
JOIN  (SELECT DISTINCT p.prod_id
       FROM   d_product p
       WHERE  p.category='ELEC') v
ON v.prod_id = s.prod_id;
```
- 경우에 따라 `DISTINCT`가 **조인 전 후**로 이동 가능.
- `UNION ALL` 뷰는 **OR-EXPAND**와 결합되어 각 분기로 나뉜 뒤 **머징**/조인 순서 선택.

## 분석함수/외부조인과의 상호작용

- **분석함수(윈도우)**가 뷰 내부에 있으면 **병합을 막는** 경우 많음(순서 보존/파티션 의미).
- **외부조인(LEFT/RIGHT/FULL)**은 **NULL 보존** 의미 때문에 머징이 **제한적**.
  - 단, **의미 보존 가능한 범위**에서 일부 푸시/병합이 이뤄질 수 있음(버전·패턴 의존).
  - 이런 경우는 보통 **비용기반 판단**에 의존 → 자동 머징 실패 시 **수동 재작성**이 안전.

---

# 비용기반 쿼리 변환(CQT)의 필요성

**왜 항상 병합하지 않을까?**
- 병합은 **탐색공간 확장**이지만, **항상 이득이 아니다**. **데이터 분포**/**카디널리티** 오판 시
  - 조인 순서가 **역효과**,
  - 집계 위치 이동이 **중간행 폭증** 유발,
  - 파티션 프루닝 기회 상실/증가 등 다양한 **트레이드오프**가 존재.

### 동일 의미, 다른 비용 — 사례

```sql
-- (A) 물리화(비머지)가 유리한 경우: a가 매우 작고, 여러번 재사용
WITH a AS (
  SELECT s.cust_id, SUM(s.amount) sum_amt
  FROM   f_sales s
  WHERE  s.sales_dt BETWEEN :d1 AND :d2
  GROUP  BY s.cust_id
)
SELECT /*+ MATERIALIZE */
       c.cust_id, a.sum_amt
FROM   d_customer c
JOIN   a ON a.cust_id=c.cust_id
WHERE  c.tier='VIP';

-- (B) 병합이 유리한 경우: 조인 전 필터(예: c.tier='VIP')를 s에 푸시 가능
WITH a AS (
  SELECT s.cust_id, SUM(s.amount) sum_amt
  FROM   f_sales s
  WHERE  s.sales_dt BETWEEN :d1 AND :d2
  GROUP  BY s.cust_id
)
SELECT /*+ INLINE MERGE(a) */
       c.cust_id, SUM(s.amount) sum_amt
FROM   d_customer c
JOIN   f_sales s ON s.cust_id = c.cust_id
WHERE  c.tier='VIP'
AND    s.sales_dt BETWEEN :d1 AND :d2
GROUP  BY c.cust_id;
```
- **정확한 통계/히스토그램/확장통계(결합 선택도)**가 없으면 **CBO 오판** → 병합/비병합 판단이 빗나감.
- 따라서 **CQT는 비용 기반**이어야 하며, **실측**으로 검증해야 한다.

---

# 머징되지 않는(View가 병합 불가) 경우의 처리

병합이 **의미상 불가**하거나 **옵티마이저가 회피**하는 패턴이 있다.

## 머징이 어렵거나 금지되는 대표 패턴

- 뷰 내부에 **`ROWNUM`/샘플링/SAMPLE/CONNECT BY/MODEL/Flashback** 등 **순서/상태 의존** 요소
- **분석함수(윈도우)**가 결과 의미에 필수
- **외부조인**으로 인해 **NULL 보존** 의미가 깨질 위험
- **DISTINCT/GROUP BY**가 상위와 얽혀 **의미 보존**이 어려운 경우
- **MERGE/NO_MERGE** 힌트로 **명시 금지**
- 버전/버그/통계 이슈로 **자동 변환 실패**

## 처리 전략(대안)

1) **물리화 유도**:
   - `NO_MERGE(qb)` 또는 `MATERIALIZE`(WITH에서), 필요시 `RESULT_CACHE`
   - 장점: **재사용**/중복 제거, 조인 순서 **안정화**
   - 단점: Temp I/O/메모리 부담
2) **명시적 재작성**:
   - 복잡 뷰를 **명시 조인**/집계로 풀고, `LEADING/USE_*`로 **조인 순서·방법 고정**
   - 외부조인이라면 **필요 부분만 인라인**, 나머지는 **물리화**로 타협
3) **단계적 파이프라이닝**:
   - 큰 집합을 **여러 단계**(필터→집계→조인)로 나누고, 각 단계의 **주요 필터·프루닝**이 일어나게 설계
4) **통계 보정**:
   - 히스토그램/확장 통계/컬럼 그룹 통계를 수집해 **카디널리티 오판**을 교정
5) **물리 설계 보조**:
   - 적합 인덱스·파티션 설계를 통해 **병합 없이도** 효율적 경로 확보
   - **물질화 뷰(MV)**/Query Rewrite로 **사전 집계** 사용

---

# 실전 시나리오 & 실행계획 비교

## “머징으로 프루닝 촉진” (집계 뷰 → 병합)

```sql
-- 원형: 집계 뷰 a를 고객과 조인
EXPLAIN PLAN FOR
SELECT c.cust_id, a.sum_amt
FROM   d_customer c
JOIN  (SELECT s.cust_id, SUM(s.amount) sum_amt
       FROM   f_sales s
       WHERE  s.sales_dt BETWEEN DATE '2024-03-01' AND DATE '2024-03-31'
       GROUP  BY s.cust_id) a
ON a.cust_id = c.cust_id
WHERE c.region='KOR';
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY());
```
- (가능) `VIEW` 노드(비병합) 또는 `HASH JOIN` 평탄화(병합).
- 병합 시 `c.region='KOR'`가 s로 **전이/푸시**되어 s 스캔량 감소.

```sql
-- 병합 강제 + 조인 순서/방법 지정
EXPLAIN PLAN FOR
SELECT /*+ MERGE(a) USE_HASH(s) LEADING(c s) */
       c.cust_id, SUM(s.amount) sum_amt
FROM   d_customer c
JOIN   f_sales s ON s.cust_id=c.cust_id
WHERE  c.region='KOR'
AND    s.sales_dt BETWEEN DATE '2024-03-01' AND DATE '2024-03-31'
GROUP  BY c.cust_id;
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY());
```

## “물리화가 더 좋은” 경우 (재사용·소집합)

```sql
-- a를 여러 번 조인/필터에 재사용
WITH a AS (
  SELECT s.cust_id, SUM(s.amount) sum_amt
  FROM   f_sales s
  WHERE  s.sales_dt BETWEEN :d1 AND :d2
  GROUP  BY s.cust_id
)
SELECT /*+ MATERIALIZE */ c.cust_id, a.sum_amt
FROM   d_customer c
JOIN   a ON a.cust_id=c.cust_id
WHERE  c.tier='VIP'
UNION ALL
SELECT /*+ MATERIALIZE */ c.cust_id, a.sum_amt
FROM   d_customer c
JOIN   a ON a.cust_id=c.cust_id
WHERE  c.region='KOR';
```
- a가 **작고 재사용**되면 물리화가 **더 빠를 수** 있다.
- 반대로 a가 크고 단발이면 병합이 유리.

## DISTINCT/UNION ALL의 병합 관찰

```sql
EXPLAIN PLAN FOR
SELECT s.sales_id
FROM   f_sales s
JOIN  (SELECT DISTINCT p.prod_id
       FROM   d_product p
       WHERE  p.category='ELEC') v
ON v.prod_id = s.prod_id;
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY());

EXPLAIN PLAN FOR
SELECT /*+ MERGE(v) */ s.sales_id
FROM   f_sales s
JOIN  (SELECT DISTINCT p.prod_id
       FROM   d_product p
       WHERE  p.category='ELEC') v
ON v.prod_id = s.prod_id;
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY());
```
- `VIEW` 노드 소거/존재, `HASH JOIN`/`JOIN FILTER` 생성 여부를 비교.

---

# 디버깅/진단 방법

```sql
-- 실측 실행계획(변환 노트/프레디킷 이동 확인)
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(
  NULL, NULL, 'ALLSTATS LAST +PREDICATE +NOTE +ALIAS'));
```
- `Note`에서 **View Merging/Subquery Unnesting/Predicate Pushdown** 흔적 확인
- `Predicate Information`에서 **푸시/전이**·뷰 내부→외부 이동 확인
- `VIEW` 연산자의 유무로 병합 여부 직관 체크
- 필요 시 **10053 Trace**(전문)로 CBO 변환 결정을 정밀 추적

---

# 베스트프랙티스 체크리스트

- [ ] **단순 뷰**는 기본적으로 **병합 허용** (`NO_MERGE` 남발 금지)
- [ ] **복합 뷰**(집계/Distinct/Union All)는 **의미 보존** 되는 범위에서 **집계 위치 최적화**
- [ ] `MERGE(qb)`/`NO_MERGE(qb)`는 **QB 이름**으로 **정밀 타겟팅**
- [ ] 재사용·작은 집합 = **물리화** 고려(`MATERIALIZE`/RESULT CACHE/MV)
- [ ] **조인 순서/방법**은 `LEADING/ORDERED`, `USE_NL/HASH/MERGE`로 보정
- [ ] **통계/히스토그램/확장통계** 정비(카디널리티 오판 방지)
- [ ] 외부조인/윈도우 함수/ROWNUM 등은 **병합 난이도↑** → **수동 재작성** 고려
- [ ] 항상 **E-Rows vs A-Rows**(실측)로 판단. “빠를 것 같다” 금지.

---

## 부록 A. Hint & 패턴 요약표

| 목적 | 힌트/기법 | 설명 |
|---|---|---|
| 병합 강제 | `MERGE(qb)` | 인라인 뷰를 상위로 병합(의미 보존되는 한) |
| 병합 금지 | `NO_MERGE(qb)` | 인라인 뷰 물리화 유지(Temp 가능) |
| WITH 인라인 | `INLINE` | WITH를 인라인으로 확장(병합 가능성↑) |
| WITH 물리화 | `MATERIALIZE` | WITH를 Temp로 물리화(재사용 유리) |
| 변환 전체 금지 | `NO_QUERY_TRANSFORMATION` | 디버깅/보호용 |
| 조인 순서 | `LEADING/ORDERED` | 병합 후에도 적용 |
| 조인 방법 | `USE_NL/HASH/MERGE` | NL/HJ/SMJ 유도 |
| 빌드/프로브 전환 | `SWAP_JOIN_INPUTS` | 해시 빌드 입력 전환 |

---

## 부록 B. 자주 묻는 질문(FAQ)

**Q1. View Merging과 Subquery Unnesting 차이는?**
- 둘 다 **쿼리 변환**이지만, *View Merging*은 **FROM의 인라인 뷰를 상위로 평탄화**하는 개념.
- *Unnesting*은 **WHERE/SELECT-list** 서브쿼리를 **조인/집계** 형태로 재작성.
- 실제로는 **결합**되어 동작: 뷰 병합 후 **조인 순서 최적화**, 또는 언네스트 후 뷰 머징.

**Q2. 병합이 항상 이득인가?**
- 아니요. a) 뷰가 **작고 재사용多**면 물리화가 이득, b) 병합으로 **중간행 폭증**시 비용↑.
- **CQT+실측**이 답.

**Q3. DISTINCT/AGG 있는 뷰도 병합되나요?**
- **가능**하나 **의미 보존** 하에서만. Oracle은 버전에 따라 범위가 확대되었고, 카디널리티/통계에 민감.

**Q4. 왜 내 쿼리는 병합이 안 되나요?**
- 외부조인/윈도우/ROWNUM/CONNECT BY/샘플링 등 **제약 요소** 때문일 수 있음.
- `DBMS_XPLAN`의 `NOTE`/`PREDICATE`로 변환 흔적을 확인하고, **수동 재작성** 고려.

---

## 맺음말

- **View Merging**은 **조인 순서·액세스 경로** 최적화를 위한 **핵심 CQT**다.
- **단수 머징**은 탐색공간을 넓히고, **복합 머징**은 **집계 푸시**로 비용을 크게 낮춘다.
- 그러나 **항상 이득은 아니며**, 데이터 분포·통계·의미 보존이 관건.
- 실제 성능은 **실측 실행계획(ALLSTATS LAST)**과 **세션 통계**로 검증하라 — 그것이 정답이다.
