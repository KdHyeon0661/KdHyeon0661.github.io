---
layout: post
title: DB 심화 - 서브쿼리 Unnesting (1)
date: 2025-11-18 16:25:23 +0900
category: DB 심화
---
# 서브쿼리 Unnesting 완전 가이드

## 서브쿼리 Unnesting의 개념과 필요성

**서브쿼리 Unnesting(언네스팅)**은 옵티마이저가 **실행계획 생성 전에** 상관 또는 비상관 서브쿼리를 조인 형태로 재작성하는 **쿼리 변환(Query Transformation)** 기법입니다. 이 변환은 성능 최적화에 중요한 역할을 하며, 다음과 같은 핵심 효과를 제공합니다:

1. **실행 전략의 다양화**: 조인, 파티션 프루닝, 인덱스 사용 등 더 많은 실행 전략 후보를 검토할 수 있게 합니다.
2. **CPU 및 I/O 부하 감소**: 스칼라 서브쿼리의 대량 호출(row-by-row)을 한 번의 조인 또는 집계 작업으로 변환하여 시스템 자원 사용을 크게 줄입니다.
3. **집합 연산 최적화**: `EXISTS/IN/NOT EXISTS/NOT IN`을 SEMI/ANTI JOIN으로 변환하여 중복 제거와 부정 조건 처리를 데이터베이스 엔진 수준에서 효율적으로 수행합니다.

---

## 서브쿼리의 분류 체계 이해

### 상관성 기준 분류

- **상관 서브쿼리(Correlated Subquery)**: 내부 쿼리가 외부 쿼리의 컬럼을 참조합니다.
  ```sql
  SELECT c.cust_id
  FROM   d_customer c
  WHERE  EXISTS (SELECT 1 FROM f_sales s WHERE s.cust_id = c.cust_id);
  ```

- **비상관 서브쿼리(Uncorrelated Subquery)**: 내부 쿼리가 외부 컬럼을 참조하지 않습니다.
  ```sql
  SELECT * FROM d_product
  WHERE  prod_id IN (SELECT prod_id FROM f_sales WHERE sales_dt >= DATE '2024-03-01');
  ```

### 반환 형태 기준 분류

- **스칼라 서브쿼리(Scalar Subquery)**: 단일 행과 단일 컬럼 값을 반환합니다.
- **멀티로우 서브쿼리**: `IN`, `EXISTS`, `ANY/ALL`, `NOT EXISTS/NOT IN` 등 집합을 반환합니다.

### 위치 기준 분류

- **SELECT 절 서브쿼리**: 스칼라 서브쿼리 형태로 주로 사용됩니다.
- **WHERE/HAVING 절 서브쿼리**: 필터 조건으로 사용됩니다.
- **FROM 절 서브쿼리**: 인라인 뷰 또는 CTE 형태로, 주로 뷰 머지(View Merge) 대상이 됩니다.

### 연산자 유형 분류

- **EXISTS / NOT EXISTS**: SEMI / ANTI JOIN 후보
- **IN / NOT IN**: SEMI / ANTI JOIN 후보  
- **= ANY / > ALL 등**: 쿼리 재작성 후 조인 또는 집계로 변환 가능
- **스칼라 서브쿼리**: LEFT/INNER JOIN + GROUP BY/집계로 변환 가능

**중요 참고사항**: `NOT IN`은 서브쿼리 결과에 NULL이 포함되면 전체 결과가 공집합이 되는 3값 논리 특성을 가집니다. 실무에서는 **`NOT EXISTS`** 사용을 권장합니다.

---

## Unnesting의 의미와 이점 분석

### 개념적 의미
서브쿼리를 **조인 형태로 재작성**함으로써 옵티마이저가 NL/HJ/SMJ 조인 방식, 인덱스 및 파티션 선택 등 **넓은 탐색 공간**에서 비용 비교를 수행할 수 있도록 합니다.

### 주요 이점

1. **대량 호출 제거**: 수만 번 호출되는 스칼라 서브쿼리를 **한 번의 조인 또는 집계 작업**으로 대체합니다.
2. **액세스 경로 최적화**: `EXISTS/IN`을 **SEMI JOIN**으로 변환하여 중복 탐색과 반복 스캔을 방지합니다.
3. **프루닝 및 푸시다운 활성화**: 조인 전에 프레디킷 푸시다운과 전이성을 활용하여 읽어야 할 블록 수를 최소화합니다.
4. **조인 순서 최적화**: 언네스팅 후 `LEADING`, `ORDERED`, `USE_*` 힌트를 사용하여 구체적인 조인 순서를 조정할 수 있습니다.

---

## 실습 환경 준비

다음은 예제 실습을 위한 기본 인덱스 설정입니다(기존 스키마가 있는 경우 재생성 불필요).

```sql
-- 핵심 인덱스 예시
CREATE INDEX ix_prod_cat_brand ON d_product(category, brand, prod_id);
CREATE INDEX ix_fs_cust_dt     ON f_sales(cust_id, sales_dt);
CREATE INDEX ix_fs_prod_dt     ON f_sales(prod_id, sales_dt);
```

---

## 기본 예제를 통한 Unnesting 이해

### `EXISTS` → **SEMI JOIN** 변환

**원본 쿼리(상관 서브쿼리)**
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

**언네스팅 효과**
- `EXISTS`는 매칭되는 첫 번째 행만 찾으면 되므로, **SEMI JOIN**으로 변환됩니다.
- 예상 실행계획: `NESTED LOOPS SEMI` 또는 `HASH JOIN SEMI`
- `ix_fs_cust_dt` 인덱스를 활용한 빠른 매칭이 기대됩니다.

**변환 제어 힌트**
```sql
SELECT /*+ UNNEST */ c.cust_id ...  -- 언네스팅 강제
SELECT /*+ NO_UNNEST */ c.cust_id ...  -- 언네스팅 금지
```

**핵심 포인트**
- `SEMI JOIN`은 자동으로 중복 제거를 수행합니다(외부 행당 내부 다중 매칭이 있어도 한 번만 처리).
- `s.sales_dt >= :d`와 같은 고선택도 필터일수록 성능 향상 효과가 큽니다.

### `NOT EXISTS` → **ANTI JOIN** 변환

**원본 쿼리**
```sql
SELECT c.cust_id
FROM   d_customer c
WHERE  NOT EXISTS (
  SELECT 1 FROM f_sales s
  WHERE  s.cust_id = c.cust_id
  AND    s.sales_dt >= DATE '2024-03-01'
);
```

**언네스팅 효과**
- `HASH JOIN ANTI` 또는 `NESTED LOOPS ANTI`로 변환되어 **일치하지 않는 행을 효율적으로 식별**합니다.

**주의사항**: `NOT IN`은 서브쿼리 결과에 NULL이 하나라도 있으면 전체 결과가 UNKNOWN이 됩니다. 실무에서는 **`NOT EXISTS`** 사용을 강력히 권장합니다.

### `IN` → **SEMI JOIN** 변환

**원본 쿼리(비상관 서브쿼리)**
```sql
SELECT *
FROM   d_product p
WHERE  p.prod_id IN (SELECT s.prod_id
                     FROM   f_sales s
                     WHERE  s.sales_dt BETWEEN DATE '2024-03-01' AND DATE '2024-03-31');
```

**언네스팅 효과**
- 내부 서브쿼리 결과의 중복을 제거한 후 **SEMI JOIN**으로 변환됩니다.
- `ix_fs_prod_dt` 인덱스가 있으면 해시 집계 → 해시 세미 조인 패턴이 빈번하게 사용됩니다.

**실무 팁**: 내부 서브쿼리에 `GROUP BY prod_id`를 명시적으로 추가하면 옵티마이저가 중복 제거 비용을 더 정확히 평가할 수 있습니다.

### 스칼라 서브쿼리 → **조인 + 집계** 변환

**비효율적인 원본 패턴**
```sql
-- 각 상품의 3월 총매출을 스칼라 서브쿼리로 계산
SELECT p.prod_id,
       (SELECT SUM(s.amount)
        FROM   f_sales s
        WHERE  s.prod_id = p.prod_id
        AND    s.sales_dt BETWEEN DATE '2024-03-01' AND DATE '2024-03-31') AS m_sum
FROM   d_product p
WHERE  p.category = 'ELEC';
```
- 상품 테이블에 2만 행이 있다면 **스칼라 서브쿼리가 2만 번 호출**됩니다. 캐싱이 일부 도움이 되지만, 값의 다양성이 높으면 캐시 미스가 빈번합니다.

**언네스팅을 통한 개선**
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
- 내부에서 **한 번만 집계** 수행 후 외부와 조인합니다.
- 실행계획: `HASH GROUP BY` → `HASH JOIN OUTER`
- **대량 호출 제거**가 핵심 성과입니다.

### `ANY/ALL` 조건의 변환

```sql
-- 가격이 하위집합의 "모든 값보다 큰" 상품 조회
SELECT p.prod_id
FROM   d_product p
WHERE  p.prod_id > ALL (SELECT s.prod_id FROM f_sales s WHERE s.sales_dt >= :d);
```
- 내부 집합을 최대값 또는 최소값으로 **집계한 후 스칼라 비교**로 단순화하거나, **조인 + 필터** 형태로 재작성할 수 있습니다.
- 옵티마이저가 자동으로 변환을 시도하며, 복잡한 경우 `NO_UNNEST`로 원형을 보호할 수 있습니다.

---

## Unnesting 적용 조건과 제한사항

### 효과적인 적용 조건
- `EXISTS/IN/NOT EXISTS` 기반의 필터 서브쿼리
- SELECT 절의 스칼라 서브쿼리(특히 SUM/COUNT/MIN/MAX 집계 함수 사용 시)
- 상관 컬럼이 등치 조건으로 연결되고 내부 쿼리가 SARGABLE(인덱스 활용 가능)한 경우

### 어려운 변환 조건 및 제한사항
- 외부 조인의 NULL 보존 의미에 영향을 미치는 경우(푸시다운이나 전이성으로 결과 변경 위험)
- `NOT IN`에 NULL 포함 가능성이 있는 경우
- 서브쿼리 내부에 ROWNUM, 샘플링, 분석 함수 등 변환을 저해하는 요소가 있는 경우
- 복잡한 OR/CASE 조건으로 인해 실행계획 폭발 위험이 있는 경우

### 변환 제어 힌트
- 강제/금지: `UNNEST` / `NO_UNNEST`
- 연쇄 변환 관련: `MERGE/NO_MERGE`, `USE_CONCAT/NO_EXPAND`
- 변환 범위 축소: `QB_NAME`, `NO_QUERY_TRANSFORMATION(@qb)` (필요 시)

---

## 언네스팅된 쿼리의 조인 순서 및 방법 조정

언네스팅 후 서브쿼리는 **하나의 테이블(또는 집합)**처럼 JOIN 트리에 통합됩니다. 이때 조인 순서와 방법은 일반 조인과 동일하게 **비용 기반**으로 결정되며, 필요 시 힌트로 세밀하게 조정할 수 있습니다.

### 기본 원칙: 비용 기반 결정
- NL/Hash/Merge 조인 방식 중 어떤 것이 최적인지
- 어떤 테이블이 드라이빙(LEADING) 역할을 할지
- 통계, 히스토그램, 카디널리티 정보를 기반으로 결정

### 조정 힌트 모음
- **조인 순서 제어**:
  - `LEADING(table_or_view …)`: 지정한 순서로 드라이빙
  - `ORDERED`: FROM 절에 나열된 순서를 따르도록 강제(뷰 머지/언네스팅 후 순서 고려)
- **조인 방법 제어**:
  - `USE_NL(t)` / `USE_HASH(t)` / `USE_MERGE(t)`
  - `SWAP_JOIN_INPUTS(t)`: 해시 조인의 빌드/프로브 역할 역전(메모리 또는 카디널리티 조건에 유리)
- **세미/안티 조인에도 유효**: `USE_HASH`를 사용하면 `HASH JOIN SEMI/ANTI`가 유도됩니다.

### 실전 예제: `EXISTS` 언네스팅 후 드라이빙 변경

```sql
-- 원본: EXISTS (언네스팅 → SEMI JOIN)
SELECT /*+ qb_name(main) */ c.cust_id
FROM   d_customer c
WHERE  EXISTS (SELECT 1 FROM f_sales s
               WHERE s.cust_id = c.cust_id
               AND   s.sales_dt >= DATE '2024-03-01');

-- 조정 1) 고객을 드라이빙, 내부는 해시 세미 조인
SELECT /*+ UNNEST LEADING(c) USE_HASH(s) */ c.cust_id
FROM   d_customer c
JOIN   f_sales s
ON     s.cust_id = c.cust_id
AND    s.sales_dt >= DATE '2024-03-01'
-- 세미 조인의 중복 제거 의미는 옵티마이저가 유지

-- 조정 2) 반대로 매출을 드라이빙(매출 데이터가 적을 때 유리)
SELECT /*+ UNNEST LEADING(s c) USE_NL(c) SWAP_JOIN_INPUTS(c) */
       c.cust_id
FROM   d_customer c
JOIN   f_sales s
ON     s.cust_id = c.cust_id
AND    s.sales_dt >= DATE '2024-03-01';
```

### 실전 예제: 스칼라 서브쿼리 언네스팅 후 조인 전략

```sql
-- 스칼라 언네스팅(사전 집계 → 조인)
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
- `LEADING(ss p)`로 **집계 결과**(일반적으로 작은 집합)를 먼저 처리하고 `p`를 조인하는 전략을 강제합니다.
- `USE_HASH(p)`로 **해시 아우터 조인**을 유도합니다.

---

## 고급 언네스팅 케이스 분석

### OR 조건과 언네스팅의 연쇄 변환
OR 조건으로 인해 인덱스 사용이나 언네스팅이 제한될 경우 **`USE_CONCAT`(OR-EXPAND)** 힌트를 사용하여 분기별 최적 경로와 언네스팅을 유도할 수 있습니다.

```sql
SELECT /*+ USE_CONCAT */ *
FROM   f_sales s
WHERE  (EXISTS (SELECT 1 FROM d_product p
                WHERE p.prod_id=s.prod_id AND p.brand='B0'))
    OR (EXISTS (SELECT 1 FROM d_customer c
                WHERE c.cust_id=s.cust_id AND c.tier='VIP'));
-- → UNION ALL 형태의 두 분기로 분할된 후 각 분기에서 EXISTS 언네스팅(세미조인) 수행
```

### HAVING 절의 상관 서브쿼리
```sql
-- 카테고리별 합이 "해당 카테고리의 3월 최대값 이상인" 상품만 조회
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
- 실무에서는 이렇게 복잡한 구조를 **서브쿼리를 먼저 집계 뷰 또는 CTE로 생성**하고, `MATERIALIZE`/`INLINE`과 `UNNEST` 조합으로 풀어 **명시적 조인 구조**로 단순화하는 접근이 권장됩니다.

### `NOT IN`의 NULL 함정과 언네스팅
```sql
-- 주의: 서브쿼리에 NULL이 있으면 전체 결과가 UNKNOWN → 거의 공집합 반환
SELECT p.prod_id
FROM   d_product p
WHERE  p.prod_id NOT IN (SELECT s.prod_id FROM f_sales s);
```
- 옵티마이저가 ANTI JOIN으로 변환하려면 **NULL 제거 조건**(예: `WHERE s.prod_id IS NOT NULL`)이 필요합니다.
- 실무에서는 처음부터 **`NOT EXISTS`** 패턴을 사용하는 것이 안전합니다:
```sql
SELECT p.prod_id
FROM   d_product p
WHERE  NOT EXISTS (SELECT 1 FROM f_sales s WHERE s.prod_id = p.prod_id);
```

### 언네스팅 불가 또는 비효율 상황에서의 대안
- **`NO_UNNEST`**로 원형을 보호하고, 스칼라 서브쿼리의 경우 **결과 캐시/스칼라 캐시** 효과를 기대합니다.
- 가능하다면 **집계/조인으로 명시적으로 재작성**하여 사람이 직접 언네스팅을 구현합니다.

---

## 언네스팅 성공 여부 관찰 및 진단 방법

```sql
-- 실측 실행계획 + 변환 노트/프레디킷 이동 확인
SELECT *
FROM   TABLE(DBMS_XPLAN.DISPLAY_CURSOR(
         NULL, NULL, 'ALLSTATS LAST +NOTE +PREDICATE +ALIAS'));
```
- 실행계획에서 `NESTED LOOPS SEMI/ANTI`, `HASH JOIN SEMI/ANTI`가 확인되면 **언네스팅 성공**입니다.
- `Predicate Information` 섹션에서 **전이 프레디킷/푸시다운** 여부를 확인합니다.
- 일부 Oracle 버전에서는 `Notes` 섹션에 **Subquery Unnesting** 관련 메시지가 기록됩니다.
- 필요 시 10053 트레이스(전문가용/비상 상황)를 통해 변환 결정 과정을 추적할 수 있습니다.

---

## 성능 개선 시나리오별 적용 전략

### OLTP 환경: 스칼라 서브쿼리 다건 호출 최적화
- **증상**: 높은 CPU 사용률, `BUFFER_GETS` 폭증, `ROW_NUMBER`(스칼라 캐시 미스) 비율이 높은 SQL
- **처방**: **스칼라 언네스팅**을 통해 한 번의 집계 후 조인으로 변환 → 실행 시간 **수배~수십배 단축** 사례 빈번

### 리포트 환경: 다중 `EXISTS` 필터 최적화
- **증상**: 개별 필터에 인덱스가 존재하지만 **조건 결합**이 비효율적
- **처방**: **언네스팅 + OR-EXPAND**를 활용하여 각 분기에 적합한 인덱스 경로 선택 → I/O 부하 급감

---

## 실습 예제 모음

### EXISTS → SEMI JOIN + 조인 순서 지정
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

### 스칼라 언네스팅(집계→조인) + 해시조인 유도
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

### OR-EXPAND + 언네스팅
```sql
EXPLAIN PLAN FOR
SELECT /*+ USE_CONCAT */ *
FROM   f_sales s
WHERE  EXISTS (SELECT 1 FROM d_product  p WHERE p.prod_id=s.prod_id AND p.brand='B0')
   OR  EXISTS (SELECT 1 FROM d_customer c WHERE c.cust_id=s.cust_id AND c.tier='VIP');
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);
```

### NOT IN → NOT EXISTS로의 안전한 전환
```sql
-- 위험 패턴(서브쿼리에 NULL 있으면 거의 공집합)
SELECT p.prod_id
FROM   d_product p
WHERE  p.prod_id NOT IN (SELECT s.prod_id FROM f_sales s);

-- 안전 패턴
SELECT p.prod_id
FROM   d_product p
WHERE  NOT EXISTS (SELECT 1 FROM f_sales s WHERE s.prod_id = p.prod_id);
```

---

## 자주 묻는 질문(FAQ)

### Q1. 언네스팅이 항상 성능 향상을 보장하나요?
**A.** 대부분의 경우 성능 이득을 제공하지만, **극소량 데이터**(수십 건 이하) + **스칼라 캐시 적중률이 높은** 상황에서는 원형 쿼리가 더 빠를 수 있습니다. 반드시 실측 성능으로 판단해야 합니다.

### Q2. 언네스팅 후 실행계획이 더 나빠진 이유는 무엇인가요?
**A.** **카디널리티 추정 오류**나 **히스토그램 부재**로 인해 비용 평가가 잘못된 경우입니다. 통계 정보를 정비한 후 `LEADING` 및 `USE_*` 힌트로 실행계획을 보정해야 합니다.

### Q3. 외부 조인과 결합된 경우 결과가 달라질 수 있나요?
**A.** 예, **NULL 보존** 의미 때문에 무리한 프레디킷 전이나 푸시다운은 위험합니다. 해당 쿼리 블록에 `NO_UNNEST`를 적용하거나 수동으로 쿼리를 재작성해야 합니다.

---

## 결론

서브쿼리 Unnesting은 **"행 단위 처리(row-by-row)"를 "집합 기반 처리(set-based)"로 전환**하는 강력한 자동 재작성 기법입니다. `EXISTS/IN/NOT EXISTS`는 **SEMI/ANTI JOIN**으로, 스칼라 서브쿼리는 **집계 후 조인**으로 변환되며, 이 시점부터는 **모든 조인 최적화 기법**을 활용할 수 있게 됩니다.

성공적인 언네스팅 튜닝의 마지막 관문은 **조인 순서와 방법의 정밀한 제어(`LEADING`, `USE_*` 힌트)**와 **정확한 통계 정보 관리**에 있습니다. 언네스팅이 적용된 후에도 지속적인 모니터링과 조정이 필요합니다.

모든 최적화 작업의 최종 검증은 **실측 실행계획 분석**을 통해 이루어져야 합니다. **E-Rows(예상 행 수)와 A-Rows(실제 행 수)의 비교**는 옵티마이저의 판단 정확도를 평가하는 가장 명확한 지표입니다. 이러한 데이터 기반 접근법을 통해 복잡한 서브쿼리 구조를 효율적인 조인 패턴으로 변환하여 데이터베이스 성능을 극대화할 수 있습니다.