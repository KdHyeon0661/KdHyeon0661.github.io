---
layout: post
title: DB 심화 - View Merging
date: 2025-11-18 19:25:23 +0900
category: DB 심화
---
# View Merging

## 목표와 핵심 개념

### 학습 목표
View Merging(뷰 병합)을 단순한 정의를 넘어, **CBO(비용 기반 옵티마이저)가 언제, 왜, 어떻게 이를 결정하는지** 깊이 이해하는 것입니다. 각 유형별로 의미 보존 조건, 탐색 공간 확대 효과, 비용 역전 가능성, 그리고 힌트를 통한 제어 방법을 실전 관점에서 정리해보겠습니다.

### 기본 용어 정리
*   **Query Block (QB)**: 옵티마이저가 독립적으로 최적화하는 기본 단위인 `SELECT ... FROM ... WHERE ...` 구문입니다. `qb_name()` 힌트로 이름을 지정할 수 있습니다.
*   **Inline View**: `FROM (subquery) v` 형태의 서브쿼리 뷰입니다. `WITH` 절(서브쿼리 팩토링)로 정의된 것도 결국 인라인 뷰로 간주됩니다.
*   **Flattening**: 상위와 하위 Query Block 사이의 경계를 제거하여 하나의 단일 Query Block으로 "평탄화"하는 과정입니다.
*   **CQT (Cost-Based Query Transformation)**: 변환이 항상 수행되는 것이 아니라, 변환 전후의 **비용을 비교하여 최종 결정**되는 변환 유형을 말합니다. Complex View Merging이 대표적인 예입니다.

---

## 옵티마이저 파이프라인에서의 View Merging 위치

옵티마이저는 대략 다음 순서로 작업을 진행합니다(버전별 세부 차이는 있으나 큰 흐름은 동일합니다).

1.  **파싱 및 의미 분석(Semantic Analysis)**
2.  **쿼리 변환 단계(Query Transformations)**
    *   **View Merging**이 바로 이 단계에서 발생합니다.
    *   Subquery Unnesting, Predicate Pushdown/Pullup, OR-Expansion, Star Transformation 등 다른 중요한 변환도 함께 진행됩니다.
3.  **카디널리티 추정(Cardinality Estimation)**
4.  **실행 계획 탐색 및 비용 계산(Join Order / Access Path / Join Method Search)**
5.  **실행 계획 선택 및 저장(Child Cursor)**

**핵심은 2단계인 쿼리 변환 단계입니다.** View Merging이 성공하면 **Query Block의 구조 자체가 변경**되며, 이는 3~4단계의 비용 계산과 탐색 가능한 실행 계획 후보군을 근본적으로 바꿔놓습니다. 즉, 병합 여부는 사용될 플랜의 종류를 결정하여 성능에 수십 배의 차이를 만들어낼 수 있습니다.

---

## View Merging의 세 가지 유형

Oracle의 공식 분류에 따라 세 가지 유형을 살펴보겠습니다.

1.  **Simple View Merging (SVM)**
    *   **SPJ(Select-Project-Join)** 형태의 비교적 단순한 뷰를 병합합니다.
2.  **Outer-Join View Merging (OJVM)**
    *   외부 조인(LEFT/RIGHT/FULL JOIN)에 참여하는 뷰를, **NULL 보존 의미가 훼손되지 않는 범위 내에서** 병합합니다.
3.  **Complex View Merging (CVM)**
    *   `GROUP BY`, `DISTINCT` 등 "복잡한" 연산을 포함한 뷰를 병합하여 **집계나 중복 제거의 위치를 변경**하는 변환입니다.
    *   **항상 유리한 것이 아니므로 비용 비교(CQT)를 통해 최종 결정**됩니다.

이제 각 유형을 의미 보존 조건, 비용 효과, 실전 패턴과 함께 자세히 알아보겠습니다.

---

## Simple View Merging - SPJ 뷰 병합의 핵심

### Simple View Merging의 목적
Simple 뷰는 일반적으로 조인과 필터 조건을 묶어 놓은 서브쿼리일 뿐이므로, 병합하면 다음과 같은 이점이 있습니다.
*   **조인 그래프가 평탄화**되어 옵티마이저가 **전체 조인 순서를 한꺼번에 탐색**할 수 있게 됩니다. 이는 Query Block별로 분리된 탐색보다 훨씬 많은 조인 순서와 방법을 고려할 수 있게 해줍니다.
*   인덱스 네스티드 루프 조인(NL), 조인 필터, 파티션 프루닝 등 다양한 최적화 기회가 증가합니다.

### 의미 보존 조건
Simple 뷰는 다음 조건을 만족하면 거의 항상 병합 가능합니다.
*   뷰 내부가 순수 **SPJ** 형태입니다(SELECT, 컬럼 프로젝션, WHERE 필터, 조인만 존재).
*   **ROWNUM, 분석 함수, CONNECT BY, MODEL, FLASHBACK, SAMPLE** 등 **순서나 상태에 의존적인 요소가 없습니다**.
*   외부 조인의 NULL 보존 의미를 훼손하지 않습니다(이 경우는 OJVM에서 다룹니다).

이 조건이 충족되지 않으면, 아무리 단순해 보이는 뷰라도 병합이 차단될 수 있습니다.

### 기본 예제 및 힌트 사용

#### 원본 쿼리 (병합 가능한 SPJ 뷰)
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
이 쿼리는 'ELEC' 및 'B0' 제품의 3월 매출을 조회하는 전형적인 SPJ 뷰를 포함합니다.

#### 병합 강제 (`MERGE` 힌트)
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
**기대 효과**: `VIEW` 연산자가 사라지고, `d_product`와 `f_sales` 테이블이 하나의 Query Block에서 최적화됩니다. `d_product`의 인덱스(`ix_prod_cat_br`)와 `f_sales`의 인덱스(`ix_fs_prod_dt`)를 활용한 다양한 조인 방법(NL, Hash)이 고려될 수 있습니다.

#### 병합 방지 (`NO_MERGE` 및 `MATERIALIZE` 힌트)
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
**옵티마이저의 관점**: 뷰 `v`를 먼저 실행하여 임시 세그먼트(TEMP)에 저장한 후, 그 결과를 `f_sales`와 조인합니다.
*   **장점**: 뷰 `v`의 결과 집합이 **매우 작고**, 동일한 뷰가 쿼리 내에서 **여러 번 재사용**된다면, 한 번만 계산하여 재사용하는 것이 효율적일 수 있습니다.
*   **단점**: 조인 순서가 **`(v 생성) → (s 조인)`**으로 사실상 고정됩니다. 또한 임시 영역 I/O 및 메모리 사용에 대한 추가 비용이 발생합니다.

이것이 바로 **"View Merging이 항상 좋은 것만은 아니다"**라는 사실을 보여주는 첫 번째 사례입니다.

### Simple View Merging이 자동으로 발생하지 않을 때 점검 사항
뷰가 SPJ 형태인데도 `VIEW` 연산자가 사라지지 않는다면 다음을 의심해 보아야 합니다.
1.  뷰 내부에 병합을 방해하는 요소(예: `ROWNUM`, `ORDER BY`, 분석 함수)가 숨어 있습니다.
2.  외부 조인 의미로 인해 OJVM 제약이 적용되고 있습니다.
3.  SQL 프로파일, Outline, 패치 등에 의해 암시적으로 `NO_MERGE` 힌트가 적용되었습니다.
4.  통계 정보가 심하게 부정확하여, 옵티마이저가 비용 기반 판단을 잘못 내렸습니다(드문 경우).

---

## Outer-Join View Merging - NULL 보존 의미의 한계 내에서

### 별도 유형으로 분류되는 이유
외부 조인은 매칭되지 않는 행을 NULL로 보존한다는 독특한 의미를 가집니다. 뷰를 병합하는 과정에서 조인 순서나 방법이 변경되면, **어느 테이블의 행이 보존되어야 하는지** 그 의미가 달라질 위험이 있습니다. 따라서 Oracle은 Outer-Join View Merging을 **의미가 완벽하게 보존될 수 있을 때만 제한적으로 수행**합니다.

### 병합 가능 케이스와 제한 케이스

#### 병합 가능한 일반적인 케이스
```sql
SELECT c.cust_id, s.amount
FROM   d_customer c
LEFT JOIN (
  SELECT cust_id, amount -- 순수 SPJ 뷰
  FROM   f_sales
  WHERE  sales_dt >= :d1
) v
ON v.cust_id = c.cust_id
WHERE c.region = 'KOR';
```
뷰 `v`가 SPJ 형태이고, LEFT JOIN의 왼쪽 테이블(`d_customer`)의 모든 행을 보존해야 한다는 의미가 병합 후에도 유지될 수 있다면 병합이 가능합니다.

#### 병합이 제한될 수 있는 케이스
```sql
SELECT c.cust_id, v.sum_amt
FROM   d_customer c
LEFT JOIN (
  SELECT cust_id, SUM(amount) sum_amt -- GROUP BY 포함 (Complex 뷰)
  FROM   f_sales
  GROUP  BY cust_id
) v
ON v.cust_id = c.cust_id
WHERE c.region='KOR';
```
뷰 `v`에 `GROUP BY`가 포함되어 Complex View Merging의 영역에 들어갑니다. 병합 시 집계 위치가 이동하면 NULL 보존의 해석이 모호해질 수 있어, 옵티마이저가 비용과 의미 보존 조건을 종합적으로 판단하여 병합을 보류할 수 있습니다.

---

## Complex View Merging - 집계와 중복 제거의 위치 최적화

### CVM의 본질
Complex View Merging은 단순히 뷰를 없애는 것이 아니라, **`GROUP BY`나 `DISTINCT` 같은 "무거운" 연산의 평가 시점을 조정**하는 변환입니다. 옵티마이저는 이 연산을 조인 전에 평가할지(Early), 조인 후에 평가할지(Delayed)를 비용 비교를 통해 결정합니다.

### CVM이 유리한 전형적인 패턴

#### 조인 필터링 효과가 강한 경우 (Delayed Group By 유리)
**원본 쿼리**:
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
**병합 후 쿼리** (Delayed Group By):
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
**이점**: `c.region='KOR'` 조건으로 `d_customer` 테이블이 크게 필터링된 후, 그 결과와 `f_sales`를 조인합니다. **집계해야 할 `f_sales` 행의 수가 원본 대비 현저히 줄어들어** Hash Group By의 비용이 감소합니다.

### CVM이 불리할 수 있는 패턴 (Early Group By 유리)

**상황**: 뷰 내부의 집계가 원본 데이터를 매우 크게 축소시키는 경우.
```sql
-- a가 f_sales(1000만 행)를 고객별 집계하여 1000행으로 줄인다고 가정
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
*   **비병합(Early)의 이점**: 먼저 집계(`a`)를 수행하면 그 결과는 매우 작은 집합(1000행)이 됩니다. 이 작은 집합을 `d_customer`와 조인하면 NL 조인이 매우 효율적일 수 있습니다.
*   **병합(Delayed)의 위험**: `a`를 병합하면 `f_sales`의 원본 1000만 행과 `d_customer`를 먼저 조인하게 되어 중간 결과 집합이 폭발적으로 증가한 후, 마지막에 집계를 수행해야 합니다. 이는 훨씬 더 비쌀 가능성이 높습니다.

**핵심 요약**
*   **조인 조건이 데이터를 강력히 필터링한다면** → 집계를 **늦추는(Delayed)** 것이 유리.
*   **뷰 내부 집계가 데이터를 강력히 축소시킨다면** → 집계를 **먼저(Early)** 수행하는 것이 유리.
*   옵티마이저는 이 두 가지 접근법의 비용을 추정하여 더 저렴한 방식을 선택합니다.

---

## View Merging이 불가능하거나 회피되는 이유

옵티마이저가 View Merging을 포기하는 이유는 크게 두 가지입니다.

### 1. 의미론적 위험(Semantic Danger)
뷰를 병합하면 쿼리의 원래 의미가 훼손될 수 있는 경우입니다.
*   **순서/상태 의존적 요소**: `ROWNUM`, `ORDER BY ... FETCH FIRST`, `SAMPLE`, `CONNECT BY`, `MODEL`, Flashback Query(`AS OF`) 등.
*   **분석 함수**: 윈도우 함수의 파티셔닝 및 정렬 의미가 조인 순서 변경으로 인해 깨질 위험이 있습니다.
*   **외부 조인의 NULL 보존 의미 변경 위험**.
*   **`DISTINCT`/`GROUP BY` 범위 변경 위험**: 병합 시 중복 제거나 집계의 범위 자체가 달라질 수 있습니다.
*   **이 경우, 힌트로 강제해도 Oracle이 무시하거나(Ignored Hint), 결과가 잘못될 수 있으므로 수동으로 쿼리를 재작성하는 것이 안전합니다.**

### 2. 비용 기반 판단(Cost-Based Decision)
*   Complex View Merging의 본질인 비용 비교(CQT) 결과, **병합하지 않는 쪽의 비용 추정치가 더 낮은 경우**입니다.
*   통계 정보가 부정확하면 이 비용 비교가 잘못되어 "병합해야 하는데 안 하거나", "병합하면 안 되는데 하는" 오류가 발생할 수 있습니다.

---

## 병합 판단의 핵심: 카디널리티와 통계의 정확성

### 중간 결과 집합 크기 추정의 중요성
View Merging은 조인과 집계의 순서를 바꾸므로, 각 단계에서 생성되는 **중간 결과 집합(Row Source)의 크기 추정이 정확해야** 옵티마이저가 올바른 결정을 내릴 수 있습니다. 추정이 틀리면 선택 가능한 플랜 후보군 자체가 달라져 결과적으로 매우 비효율적인 실행 계획을 선택할 수 있습니다.

### 정확한 추정을 위한 통계 보정 도구
1.  **히스토그램**: `region`, `category` 같은 편향된 데이터 분포를 가진 컬럼의 선택도 추정 오류를 방지합니다.
2.  **확장 통계(컬럼 그룹)**: `cust_id`와 `sales_dt` 같이 함께 자주 사용되며 상관관계가 있는 컬럼들의 결합 선택도를 정확히 추정합니다.
3.  **파티션 통계 / 증분 통계**: 대규모 파티션 테이블에서 각 파티션의 특성을 반영합니다.
4.  **동적 샘플링 / SQL Plan Directive**: 통계 정보가 부족할 때 런타임에 샘플링을 통해 추정을 보정합니다.

실전에서 "View Merging이 예상대로 동작하지 않는다"고 느껴지면, 가장 먼저 **카디널리티 추정(`E-Rows`)이 현실(`A-Rows`)과 맞는지** 확인하는 것이 진단의 첫걸음입니다.

---

## View Merging 제어를 위한 실전 도구

### 힌트 정리
| 목적 | 힌트 | 설명 및 주의사항 |
| :--- | :--- | :--- |
| **병합 강제** | `MERGE(qb)` | 지정된 QB(인라인 뷰)를 상위 QB와 병합하도록 유도합니다. |
| **병합 방지** | `NO_MERGE(qb)` | 지정된 QB를 독립적으로 유지합니다. 필요시 임시 저장(Materialize)될 수 있습니다. |
| **WITH 절 병합** | `INLINE` | `WITH` 절로 정의된 서브쿼리를 인라인 뷰로 풀어 병합 가능성을 높입니다. |
| **WITH 절 물리화** | `MATERIALIZE` | `WITH` 절의 결과를 임시 세그먼트에 저장하여 재사용합니다. |
| **전체 변환 금지** | `NO_QUERY_TRANSFORMATION` | 디버깅 또는 특수 목적으로 모든 쿼리 변환을 비활성화합니다. |
| **QB 이름 지정** | `qb_name(qb)` | 힌트의 적용 대상을 명확히 하기 위해 반드시 사용하는 것이 좋습니다. |

**주의**: `NO_MERGE` 힌트를 QB 이름 없이 사용하면 의도치 않은 다른 뷰까지 병합이 막힐 수 있습니다. **`qb_name()` 힌트와 함께 사용하여 정확히 타겟팅하세요.**

### 수동 재작성이 최선의 해결책인 경우
의미 보존 조건이 까다로워 힌트로 제어하기 어려운 경우(예: 외부 조인 + 분석 함수가 혼합된 복잡한 뷰)에는, 뷰를 풀어 명시적인 조인과 집계로 쿼리를 재작성한 후, `LEADING`, `USE_NL` 등의 힌트로 원하는 실행 계획을 직접 고정하는 것이 가장 안정적입니다.

---

## 디버깅: "병합되었는가" 그리고 "왜 그렇게 되었는가"

### 실행 계획 분석 (`DBMS_XPLAN`)
```sql
SELECT *
FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(
  NULL, NULL,
  'ALLSTATS LAST +PREDICATE +NOTE +ALIAS +PEEKED_BINDS'
));
```
*   **`NOTE` 섹션**: `VIEW MERGED`, `COMPLEX VIEW MERGING` 등의 메시지를 확인합니다.
*   **`VIEW` 연산자**: 실행 계획에 `VIEW` 라인이 나타난다면 병합되지 않은 것입니다. `VW_NWVW_*`와 같은 특수 내부 뷰 이름이 보일 수 있는데, 이는 병합 과정에서 생성된 보호/재구성 목적의 뷰일 수 있습니다.
*   **`PREDICATE` 섹션**: 필터 조건이 어디로 "푸시 다운"되었는지 확인하여 병합 효과를 간접적으로 확인할 수 있습니다.

### 고급 트레이스 (10053 이벤트)
```sql
ALTER SESSION SET events '10053 trace name context forever, level 1';
-- 쿼리 실행
ALTER SESSION SET events '10053 trace name context off';
```
생성된 트레이스 파일에는 옵티마이저가 각 뷰를 병합 후보로 고려했는지, 비용 비교는 어떻게 했는지, 최종적으로 병합을 포기했다면 그 이유가 상세히 기록됩니다. "왜 병합이 안 됐는가?"에 대한 가장 확실한 근거를 제공합니다.

---

## 실전 튜닝 시나리오

### 시나리오 A: 병합을 통한 파티션 프루닝 유도
**상황**: 대용량 월별 파티션 테이블 `f_sales`와 `d_customer`를 조인할 때, `d_customer`의 `region` 조건이 강력한 필터 역할을 합니다.
**전략**: Complex View Merging을 통해 집계를 조인 뒤로 미뤄, 먼저 `region` 조건으로 필터링된 고객들만 관련된 `f_sales` 파티션과 조인하도록 유도합니다. 이로 인해 불필요한 파티션 액세스를 피하고(프루닝), 집계 대상 데이터 양도 줄일 수 있습니다.

### 시나리오 B: 병합 방지를 통한 조인 폭증 방지
**상황**: `f_sales` 테이블을 고객별로 집계하면 행 수가 크게 줄어들지만, 이를 병합하여 조인 순서를 변경하면 중간 결과가 폭발적으로 증가합니다.
**전략**: `NO_MERGE` 힌트를 사용하여 뷰를 먼저 집계(Early Group By)하도록 고정합니다. 작아진 집계 결과 집합을 다른 테이블과 조인하면 NL 조인 등 효율적인 접근이 가능해집니다. 실행 계획의 `A-Rows`를 확인하여 중간 집합 크기가 예상대로 줄어드는지 반드시 검증해야 합니다.

---

## 결론

View Merging은 단순한 "뷰 제거" 기능이 아닙니다. 이것은 옵티마이저가 **쿼리의 구조 자체를 재구성하여 탐색 가능한 실행 계획의 공간을 근본적으로 확장하거나 변경하는 핵심 메커니즘**입니다.

*   **Simple View Merging**은 주로 **탐색 공간을 확장**하여 더 많은 조인 순서와 방법을 고려할 기회를 제공합니다.
*   **Complex View Merging**은 **집계나 중복 제거 같은 무거운 연산의 평가 시점을 최적화**하는 비용 기반 결정입니다.
*   **Outer-Join View Merging**은 **쿼리의 의미, 특히 NULL 보존을 해치지 않는 선**에서 제한적으로 적용됩니다.

이러한 변환의 유익 여부는 전적으로 **데이터의 분포, 통계의 정확도, 그리고 카디널리티 추정**에 달려 있습니다. 따라서 효과적인 튜닝을 위해서는 힌트로 병합을 강제하거나 방지하는 기술적 숙련도와 더불어, **실행 계획의 추정 행수(`E-Rows`)와 실제 행수(`A-Rows`)를 꾸준히 비교하며 통계의 정확성을 관리하는 근본적인 실무 능력**이 동반되어야 합니다.

이 원리를 이해하고 적용한다면, 복잡한 인라인 뷰와 서브쿼리로 구성된 실전 SQL에서도 의도한 최적의 실행 경로를 안정적으로 구현할 수 있게 될 것입니다.