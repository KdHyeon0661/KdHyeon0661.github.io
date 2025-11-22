---
layout: post
title: DB 심화 - View Merging
date: 2025-11-18 19:25:23 +0900
category: DB 심화
---
# View Merging

## 0. 이 글의 목표와 전제

### 0.1 목표
- **View Merging(=뷰 병합)**을 “정의 암기”가 아니라  
  **CBO(비용기반 옵티마이저)가 왜/언제/어떻게 결정하는지**까지 이해한다.
- **Simple / Outer-join / Complex View Merging**을 각각
  - **의미 보존 조건(semantic correctness)**
  - **탐색공간 확장 효과**
  - **비용 역전(merge가 더 느려지는 경우)**
  - **힌트/통계/물리설계로 제어하는 법**
  으로 쪼개서 실전 튜닝 관점으로 정리한다.

### 0.2 기본 용어
- **Query Block(QB)**: 옵티마이저가 독립적으로 최적화하는 단위 `SELECT ... FROM ... WHERE ...`  
  `qb_name()` 힌트로 이름을 붙일 수 있다.
- **Inline View**: `FROM (subquery) v` 형태의 뷰. `WITH`(서브쿼리 팩터링)도 결국 인라인 뷰로 볼 수 있다.
- **Flattening**: 상·하위 QB 경계를 제거해 하나의 QB로 “평탄화(merge)”하는 것.
- **CQT(Cost-Based Query Transformation)**: 옵티마이저 변환이 “항상”이 아니라 **비용 비교로 결정**되는 변환군. Complex View Merging이 대표적이다. :contentReference[oaicite:0]{index=0}

---

## 1. View Merging이 일어나는 위치: 옵티마이저 파이프라인

옵티마이저는 대략 다음 순서로 움직인다(버전별 세부 차이는 있지만 큰 흐름은 동일).

1) **파싱/정규화(semantic analysis)**  
2) **쿼리 변환 단계(Query Transformations)**  
   - View Merging  
   - Subquery Unnesting  
   - Predicate Pushdown / Pullup  
   - OR-Expansion  
   - Star Transformation 등  
3) **카디널리티 추정(Cardinality Estimation)**  
4) **플랜 탐색/비용 계산(Join order / Access path / Join method search)**  
5) **플랜 선택/저장(Child cursor)**

**핵심은 2단계**다.  
View Merging이 되면 **QB 구조가 바뀌어** 3~4단계의 탐색공간 자체가 달라진다.  
즉, “병합 여부”는 **플랜 후보군을 바꿔** 성능을 수배 이상 갈라놓는다. :contentReference[oaicite:1]{index=1}

---

## 2. View Merging의 3가지 타입

Oracle 공식 분류는 아래 3개다. :contentReference[oaicite:2]{index=2}

1) **Simple View Merging (SVM)**  
   - **SPJ(Select-Project-Join)** 중심 “가벼운” 뷰 병합
2) **Outer-Join View Merging (OJVM)**  
   - 외부조인에 등장하는 뷰를 **NULL 보존 의미가 깨지지 않는 한도에서** 병합
3) **Complex View Merging (CVM)**  
   - `GROUP BY`, `DISTINCT` 등 “무거운” 뷰를 병합해  
     **집계/중복제거 위치를 바꾸는** 변환  
   - **항상 이득이 아니므로 비용 비교(CQT)**로 결정된다. :contentReference[oaicite:3]{index=3}

이제 각각을 **의미 보존 조건 + 비용 효과 + 실전 패턴**으로 깊게 들어가보자.

---

## 3. 단수(Simple) View Merging - SPJ 뷰 병합의 정석

### 3.1 왜 Simple 뷰를 병합하나?

Simple 뷰는 보통 “조인·필터를 묶어둔 서브쿼리”일 뿐이라, 병합하면:

- **조인 그래프가 평탄화**되어
  - 조인 순서 탐색이 **QB별 분리 탐색 → 전역 탐색**으로 바뀜
  - 더 많은 조인 순서/방법 후보를 비용화 가능
- **인덱스 NL 조인·조인 필터·파티션 프루닝** 같은 기회가 늘어남 :contentReference[oaicite:4]{index=4}

Oracle 공식 예시도 “4개 후보 → 6개 후보”로 탐색공간이 늘어  
NL 기반 인덱스 접근이 가능해져 비용이 떨어졌다는 구조다. :contentReference[oaicite:5]{index=5}

### 3.2 Simple View Merging의 “의미 보존” 조건

Simple 뷰는 아래가 성립하면 거의 항상 병합 가능:

- 뷰 내부가 **SPJ** (단순 SELECT, 컬럼 투영, WHERE 필터, 조인)
- **ROWNUM, 분석함수, CONNECT BY, MODEL, FLASHBACK, SAMPLE** 등  
  **순서/상태 의존 요소가 없음**
- 외부조인의 NULL 보존을 깨지 않음(외부조인 케이스는 OJVM 파트에서)  

이 조건이 깨지면 “Simple이라도” 병합이 막힐 수 있다.

### 3.3 기본 예시(사용자 스키마 기반 확장)

#### 3.3.1 원형 쿼리

```sql
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

**의미상** “ELEC/B0 제품을 산 3월 매출”을 찾는 전형적 SPJ 뷰다.

#### 3.3.2 비병합(물리화) 강제

```sql
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

**옵티마이저 관점**
- `v`를 **TEMP로 먼저 만들어 저장**한 뒤 조인할 수 있음
- 장점:  
  - `v`가 **극소 집합**이고  
  - 동일 뷰를 **여러 번 재사용**하면  
  **“한 번 만들고 재사용”**하는 게 싸질 수 있다.
- 단점:
  - 조인 순서가 **`(v 생성) → s 조인`**으로 고정  
  - temp I/O + 메모리 비용 증가

이게 바로 **“병합이 항상 이득이 아닌 이유”**의 첫 번째 유형이다.

#### 3.3.3 병합 강제

```sql
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

**기대되는 플랜 변화**
- `VIEW` 노드가 사라지고
- `d_product`의 필터가 `f_sales` 접근과 결합되며
- `ix_fs_prod_dt` 또는 `ix_prod_cat_br`를 활용한  
  `NL`이나 `HASH JOIN` 후보가 열림

### 3.4 병합 후 “일반 힌트가 제대로 먹는 이유”

병합되면 `v`가 하나의 QB에 포함되니  
`LEADING/USE_HASH/USE_NL/INDEX` 같은 **일반 힌트가 QB 경계 없이 적용**된다.  
Oracle도 이 “탐색공간 확장”을 Simple VM의 핵심 가치로 본다. :contentReference[oaicite:6]{index=6}

```sql
SELECT /*+ MERGE(v) LEADING(v s) USE_HASH(s) */
       s.sales_id, s.amount
FROM   f_sales s
JOIN  (SELECT /*+ qb_name(v) */ p.prod_id
       FROM d_product p
       WHERE p.category='ELEC' AND p.brand='B0') v
ON v.prod_id = s.prod_id
WHERE s.sales_dt BETWEEN :d1 AND :d2;
```

### 3.5 Simple VM이 **자동으로 안 될 때** 체크 포인트

“SPJ인데 VIEW가 안 사라진다?” → 다음을 의심:

1) **뷰 내부에 병합 방해 요소가 숨어 있음**
   - 예: `ROWNUM`, `ORDER BY`, `FETCH FIRST`, 분석함수, CONNECT BY 등
2) **외부조인 의미 때문에 OJVM이 제한됨**
3) **`NO_MERGE`가 암묵적으로 걸림**
   - SQL Profile/Outline/patch, view definition hint 등
4) **카디널리티 오판으로 CQT가 회피**
   - 드물지만 통계가 심하게 틀리면 Simple이어도 회피하는 경우가 있다.

---

## 4. Outer-Join View Merging - NULL 의미 보존의 싸움

### 4.1 왜 별도 타입인가?

외부조인(LEFT/RIGHT/FULL)은 **“매칭 실패 행을 NULL로 보존”**하는 의미가 있다.  
뷰를 병합하다가 조인 순서/방법이 바뀌면  
**NULL 보존 대상이 달라져 결과가 바뀔 수 있으므로**,  
Oracle은 OJVM을 **의미 보존 가능할 때만 제한적으로 수행**한다. :contentReference[oaicite:7]{index=7}

### 4.2 대표 패턴

#### 4.2.1 병합 가능한 케이스(보통)

```sql
SELECT c.cust_id, s.amount
FROM   d_customer c
LEFT JOIN (
  SELECT /* SPJ */
         cust_id, amount
  FROM   f_sales
  WHERE  sales_dt >= :d1
) v
ON v.cust_id = c.cust_id
WHERE c.region = 'KOR';
```

- `v`가 SPJ이고
- NULL 보존(LEFT JOIN의 “c 행 보존”)이 깨지지 않는 범위라면  
  병합 후에도 의미 동일.

#### 4.2.2 병합이 제한되는 케이스

```sql
SELECT c.cust_id, v.sum_amt
FROM   d_customer c
LEFT JOIN (
  SELECT cust_id, SUM(amount) sum_amt
  FROM   f_sales
  GROUP  BY cust_id
) v
ON v.cust_id = c.cust_id
WHERE c.region='KOR';
```

- `GROUP BY`(Complex VM 요소) + LEFT JOIN이 결합됨
- 병합 시 **집계 위치 이동 → NULL 보존 해석이 바뀔 위험**  
  → Oracle이 비용/의미 조건을 따져 병합을 보류할 수 있음.

---

## 5. 복합(Complex) View Merging - 집계/중복제거 위치 최적화

### 5.1 CVM의 본질

Complex VM은 단순히 “뷰를 없애는 것”이 아니다.  
**집계(GROUP BY)나 DISTINCT의 “위치”를 재배치**하는 변환이다.

Oracle 공식 설명 그대로:

- 뷰를 병합하면  
  **GROUP BY/DISTINCT 평가를 “조인 뒤로 늦출 수도 있고(Delayed)”**  
  반대로 “조인 전에 유지할 수도 있다(Early)”
- 둘 중 무엇이 싸로운지는  
  **데이터 특성과 조인 필터링/폭증 여부에 달려**  
  → 그래서 **항상 비용 비교(CQT)**다. :contentReference[oaicite:8]{index=8}

### 5.2 CVM이 유리한 전형

#### 5.2.1 “조인이 강하게 필터링하는” 케이스

원형:

```sql
SELECT c.cust_id, a.sum_amt
FROM   d_customer c
JOIN  (SELECT /*+ qb_name(agg) */
              s.cust_id, SUM(s.amount) sum_amt
       FROM   f_sales s
       WHERE  s.sales_dt BETWEEN :d1 AND :d2
       GROUP  BY s.cust_id) a
ON     a.cust_id = c.cust_id
WHERE  c.region='KOR';
```

병합 후(Oracle 공식 예시와 동일한 구조): :contentReference[oaicite:9]{index=9}

```sql
SELECT /*+ MERGE(agg) */
       c.cust_id, SUM(s.amount) sum_amt
FROM   d_customer c
JOIN   f_sales s
ON     s.cust_id = c.cust_id
WHERE  s.sales_dt BETWEEN :d1 AND :d2
AND    c.region = 'KOR'
GROUP  BY c.cust_id;
```

**왜 싸지나?**
- `c.region='KOR'` 조인이 `f_sales`를 **대폭 줄이는 필터**라면  
  **집계 대상 행 자체가 줄어** Hash Group By 비용이 감소한다. :contentReference[oaicite:10]{index=10}

즉, **집계를 “늦추는(Delayed Group By)”** 게 이득.

#### 5.2.2 DISTINCT 뷰 병합

원형:

```sql
SELECT s.sales_id
FROM   f_sales s
JOIN  (SELECT DISTINCT p.prod_id
       FROM d_product p
       WHERE p.category='ELEC') v
ON v.prod_id = s.prod_id;
```

병합 시 Oracle은 상황에 따라:

- DISTINCT를 **세미조인/해시세미**로 바꿔  
  `v`를 “중복 제거 집합” 대신 “존재성 체크”로 처리하거나
- DISTINCT 위치를 조인 뒤로 늦춰  
  **필요 행만 distinct**하도록 바꿀 수 있다. :contentReference[oaicite:11]{index=11}

### 5.3 CVM이 불리해지는 전형(“조인 폭증”)

아래처럼 **집계가 매우 강하게 줄여주는 뷰**를  
병합해 집계를 뒤로 미루면 오히려 망한다.

```sql
-- a가 sales를 1000만행→1000행으로 줄여준다 가정
WITH a AS (
  SELECT cust_id, SUM(amount) sum_amt
  FROM f_sales
  WHERE sales_dt BETWEEN :d1 AND :d2
  GROUP BY cust_id
)
SELECT c.cust_id, a.sum_amt
FROM d_customer c
JOIN a ON a.cust_id = c.cust_id
WHERE c.tier='VIP';
```

- 조인 전에 a가 “초소형 집합”이 되어 NL/Hash probe 비용이 낮다.
- 병합하면 **sales 원본과 조인 → 중간행 폭증 → 그 뒤 GROUP BY**  
  → 조인이 더 비싸질 수 있다. :contentReference[oaicite:12]{index=12}

**요약**
- **필터링 조인 → Delayed Group By가 유리**
- **폭증 조인/집계가 축소 역할 → Early Group By가 유리**
- 옵티마이저는 이 둘을 **비용으로 비교해 결정**한다. :contentReference[oaicite:13]{index=13}

### 5.4 UNION ALL 뷰와 CVM + OR-Expansion

```sql
SELECT s.sales_id
FROM   f_sales s
JOIN (
  SELECT prod_id FROM d_product WHERE category='ELEC'
  UNION ALL
  SELECT prod_id FROM d_product WHERE category='HOME'
) v
ON v.prod_id = s.prod_id;
```

- `UNION ALL`은 OR-Expansion과 결합될 수 있고,
- 각 분기별로 **개별 병합 → 개별 최적 조인 순서**가 열려  
  큰 성능 차이를 만든다. :contentReference[oaicite:14]{index=14}

튜닝 시:
- 분기별 선택도가 매우 다르면 `OR_EXPAND` 유도
- 분기가 너무 많아 폭증하면 반대로 물리화/IN-LIST 처리 고려

---

## 6. “머징 불가(Non-mergeable)”의 정확한 이유

Oracle이 View Merging을 포기하는 이유는 2종류다.

### 6.1 **의미상 불가(semantic danger)**

- 뷰 내부에 **순서/상태 의존 요소**
  - `ROWNUM`, `ORDER BY ... FETCH`, `SAMPLE`, `CONNECT BY`, `MODEL`,
    Flashback(`AS OF`) 등
- 뷰 내부 **분석함수(윈도우)**가 결과 의미의 핵심  
  - 파티션/정렬 의미가 깨질 위험
- 외부조인에서 **NULL 보존 의미가 달라질 위험**
- DISTINCT/GROUP BY가 상위와 얽혀  
  병합 시 **중복제거 범위 자체가 달라질 위험** :contentReference[oaicite:15]{index=15}

→ 이런 경우는 힌트로 강제해도 **Oracle이 거부하거나(ignored hint)**  
결과가 바뀔 수 있으니 **수동 재작성**이 정석.

### 6.2 **비용상 회피(CQT decision)**

- 병합(Delayed) vs 비병합(Early) 비용을 둘 다 계산한 뒤  
  **비병합이 싸면 그대로 둔다.** :contentReference[oaicite:16]{index=16}
- 통계가 틀리면 이 판단이 틀어져  
  “병합해야 하는데 안 하는/하면 안 되는데 하는” 사고가 난다.

---

## 7. 병합 판단의 핵심 변수: 카디널리티 & 통계

### 7.1 CBO는 “중간행 크기”를 비용으로 본다
- View Merging은 조인/집계 위치를 바꾸므로  
  **중간행(row-source) 크기 추정**이 정확해야 한다.
- 추정이 틀리면 탐색공간이 바뀌는 만큼  
  결과도 극단적으로 틀어진다. :contentReference[oaicite:17]{index=17}

### 7.2 통계 보정 레버

1) **히스토그램**
   - 편향 컬럼(예: region, tier, category)의 선택도 오판 방지
2) **확장 통계(컬럼 그룹)**
   - `cust_id + sales_dt` 같은 결합 선택도 보정
3) **파티션 통계 / 증분 통계**
4) **동적 샘플링 / SQL Plan Directive**
   - 통계 미비 시 임시 보정으로 동작할 수 있음

실전에서는 “병합이 이상하다” 싶으면  
**2~3단계의 카디널리티 추정부터 의심**하는 게 빠르다.

---

## 8. 제어 수단: 힌트/파라미터/재작성

### 8.1 힌트 요약표

| 목적 | 힌트 | 의미 |
|---|---|---|
| 병합 강제 | `MERGE(qb)` | 인라인 뷰/서브쿼리 QB를 상위 QB로 병합 유도 :contentReference[oaicite:18]{index=18} |
| 병합 금지 | `NO_MERGE(qb)` | QB를 유지(필요시 Temp 물리화) :contentReference[oaicite:19]{index=19} |
| WITH 인라인화 | `INLINE` | WITH를 인라인 뷰로 펼쳐 병합 가능성↑ |
| WITH 물리화 | `MATERIALIZE` | WITH를 Temp로 생성해 재사용 유도 |
| 변환 전체 금지 | `NO_QUERY_TRANSFORMATION` | 디버깅/보호용 |
| 조인 순서 | `LEADING/ORDERED` | 병합 후 전역 조인 순서 고정 |
| 조인 방법 | `USE_NL/HASH/MERGE` | NL/HJ/SMJ 유도 |

**주의**
- `NO_MERGE`는 Simple/Complex 모두 막을 수 있다(버전/케이스에 따라 다르게 보이기도 함). :contentReference[oaicite:20]{index=20}
- QB 이름 없이 쓰면 다른 뷰까지 막는 사고가 많다 → **`qb_name()` 필수.**

### 8.2 “물리화가 더 좋은” 재사용 패턴

```sql
WITH a AS (
  SELECT /*+ qb_name(a) */
         s.cust_id, SUM(s.amount) sum_amt
  FROM f_sales s
  WHERE s.sales_dt BETWEEN :d1 AND :d2
  GROUP BY s.cust_id
)
SELECT /*+ MATERIALIZE */
       c.cust_id, a.sum_amt
FROM d_customer c
JOIN a ON a.cust_id=c.cust_id
WHERE c.tier='VIP'
UNION ALL
SELECT /*+ MATERIALIZE */
       c.cust_id, a.sum_amt
FROM d_customer c
JOIN a ON a.cust_id=c.cust_id
WHERE c.region='KOR';
```

- a를 **두 번 재사용** → 물리화가 이득 가능.
- 단, a가 커지면 temp 비용이 더 커질 수 있으니  
  **A-Rows 기준 실측**이 답.

### 8.3 “수동 재작성”이 안전한 경우

- 외부조인 + 집계/분석함수
- 병합 금지 요소가 섞여 있고, 결과 보존이 필요할 때

이때는:
1) 뷰를 풀어 명시 조인/집계로 재작성
2) `LEADING/USE_*`로 의도한 조인 순서/방법을 고정
3) 통계를 보정

---

## 9. 디버깅/진단: “병합됐는지”와 “왜 그렇게 됐는지”

### 9.1 DBMS_XPLAN 실측 플랜

```sql
SELECT *
FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(
  NULL, NULL,
  'ALLSTATS LAST +PREDICATE +NOTE +ALIAS +PEEKED_BINDS'
));
```

- **NOTE**: `VIEW MERGING`, `COMPLEX VIEW MERGING` 흔적 확인 :contentReference[oaicite:21]{index=21}
- **PREDICATE**: 필터가 어디로 푸시/전이됐는지 확인
- `VIEW` 연산자 유무:
  - 없음 → 병합
  - `VW_NWVW_*` → 병합 과정에서 새 “내부 뷰”가 생성된 케이스(특정 보호/재구성 목적)

### 9.2 23ai에서 더 좋아진 힌트/분석 리포트

23ai는 `DBMS_XPLAN`에 **SQL Analysis/Hints Report**가 개선되어  
“힌트가 왜 안 먹었는지” 확인이 훨씬 쉬워졌다. :contentReference[oaicite:22]{index=22}

### 9.3 10053 Trace(전문가용)

```sql
ALTER SESSION SET events '10053 trace name context forever, level 1';
-- 쿼리 실행
ALTER SESSION SET events '10053 trace name context off';
```

- 변환 단계에서 **merge 후보, 비용 비교, 거부 사유**가 상세히 찍힌다.
- “왜 CBO가 merge를 포기했는가?”의 최종 근거.

---

## 10. 실전 튜닝 시나리오 모음(확장)

### 10.1 시나리오 A: 병합이 프루닝을 만든다

**상황**
- `f_sales`가 월별 RANGE 파티션
- `d_customer.region='KOR'`가 5% 선택도
- `f_sales.cust_id`로 조인

**전략**
- Complex VM으로 집계를 뒤로 늦춰  
  **KOR 고객만 먼저 조인/필터 → 그 다음 집계**가 되게 만든다.

```sql
SELECT /*+ MERGE(agg) LEADING(c s) USE_HASH(s) */
       c.cust_id, SUM(s.amount) sum_amt
FROM   d_customer c
JOIN   f_sales s
ON     s.cust_id=c.cust_id
WHERE  c.region='KOR'
AND    s.sales_dt BETWEEN DATE '2024-03-01' AND DATE '2024-03-31'
GROUP  BY c.cust_id;
```

**관찰 포인트**
- 파티션 프루닝 + KOR 조인 필터가  
  `HASH GROUP BY` 입력을 크게 줄이는지.

### 10.2 시나리오 B: 병합이 폭증을 만든다

**상황**
- `f_sales`는 1억행
- 기간 필터가 넓고
- 고객별 집계가 10만행으로 축소됨
- 이후 VIP만 1% 필터

**전략**
- 뷰를 **먼저 집계(Early)**해서 줄여둔 뒤 VIP 조인

```sql
WITH agg AS (
  SELECT /*+ qb_name(agg) MATERIALIZE */
         cust_id, SUM(amount) sum_amt
  FROM f_sales
  WHERE sales_dt BETWEEN :d1 AND :d2
  GROUP BY cust_id
)
SELECT /*+ NO_MERGE(agg) */
       c.cust_id, agg.sum_amt
FROM d_customer c
JOIN agg ON agg.cust_id=c.cust_id
WHERE c.tier='VIP';
```

**관찰 포인트**
- 병합하면 조인이 폭증하는지  
  (A-Rows가 급증하면 병합 금지 쪽이 정답일 확률↑)

---

## 11. 베스트 프랙티스 체크리스트(최종)

- [ ] **Simple SPJ 뷰는 기본적으로 병합 허용**  
      (`NO_MERGE` 남발 금지)
- [ ] **Complex 뷰는 “집계 위치”의 비용 비교**가 핵심  
      (Filtered join이면 Delayed가 유리, Explosion이면 Early가 유리) :contentReference[oaicite:23]{index=23}
- [ ] `MERGE/NO_MERGE`는 **QB 이름과 함께** 정밀 타겟팅
- [ ] “작고 재사용多” = `MATERIALIZE/RESULT_CACHE/MV` 고려
- [ ] **카디널리티 오판**이 보이면  
      히스토그램/확장통계/파티션 통계부터 보정
- [ ] 외부조인/분석함수/ROWNUM 등 병합 난이도↑ 패턴은  
      **수동 재작성 + 조인 힌트 고정**이 안전
- [ ] 최종 판단은 언제나  
      **E-Rows vs A-Rows(실측)**, IO/CPU 통계, SQL Monitor로 한다.

---

## 12. 맺음말

View Merging은 “뷰를 없애는 기능”이 아니라  
**조인 그래프를 재구성해 CBO의 탐색공간과 비용구조를 바꾸는 핵심 변환**이다. :contentReference[oaicite:24]{index=24}

- Simple VM은 **탐색공간 확장**
- Complex VM은 **집계/Distinct 위치 최적화(CQT)**
- 이득/손해는 **데이터 분포·카디널리티·통계 정확도**에 달려 있으며
- 성능의 최종 근거는 **실측 플랜**이다.

이걸 기준으로 플랜을 읽고,  
병합/물리화/재작성/통계보정의 레버를 정확히 쓰면  
복잡한 인라인 뷰가 섞인 실전 SQL에서도  
의도한 실행계획을 안정적으로 만들 수 있다.