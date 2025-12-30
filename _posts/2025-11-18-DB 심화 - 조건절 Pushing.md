---
layout: post
title: DB 심화 - 조건절 Pushing
date: 2025-11-18 20:25:23 +0900
category: DB 심화
---
# 조건절 Pushing 완전 가이드

## 주제와 핵심 개념

이 가이드에서는 조건절(Predicate)의 위치를 옵티마이저가 어떻게 재배치하는지, 그 세 가지 주요 메커니즘을 다룹니다.

*   **Predicate Pushdown(조건절 푸시다운)**: 필터 조건을 **가능한 한 데이터 원천(테이블, 인덱스)에 가깝게 아래로 밀어 넣어**, 읽어야 하는 데이터 양과 처리해야 할 연산량을 최소화합니다.
*   **Predicate Pullup(조건절 풀업/호이스트)**: 쿼리의 의미를 훼손하지 않는 안전한 범위에서 조건을 **위로 끌어올려**, 조인 순서 변경이나 뷰 병합(View Merging) 같은 다른 최적화 변환이 적용될 수 있는 공간을 확보합니다.
*   **Join Predicate Pushdown(조인 조건 푸시다운)**: 조인 조건의 동등성이나 전이성(Transitivity)을 이용하여 **필터를 조인의 반대편 테이블로 전파**하거나, **Join Filter(Bloom Filter)**를 생성하여 프로브(Probe) 측 테이블 스캔 시 조기에 불필요한 데이터를 차단합니다.

**핵심 직관**
옵티마이저(CBO)는 **"쿼리의 원래 의미가 보장되는 범위 내에서, 전체 실행 비용을 가장 많이 줄일 수 있는 방향으로"** 조건절을 위아래로 이동시킵니다.
*   **Pushdown**의 목표는 **I/O와 중간 결과 집합의 크기를 줄이는 것**입니다.
*   **Pullup**의 목표는 **쿼리 의미를 보호하고, 더 넓은 범위의 최적화 변환을 가능하게 하는 것**입니다.
*   **Join Predicate Pushdown**은 필터 효과를 **조인 상대편 테이블이나 심지어 스토리지 스캔 단계까지 확장**시키는 고급 최적화입니다.

---

## 실습 환경 설정 및 관찰 방법

다음은 전형적인 차원-사실 모델 기반의 실습용 스키마입니다.

```sql
-- 차원 테이블
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

-- 사실 테이블
CREATE TABLE f_sales (
  sales_id NUMBER PRIMARY KEY,
  cust_id  NUMBER NOT NULL,
  prod_id  NUMBER NOT NULL,
  sales_dt DATE   NOT NULL,
  qty      NUMBER NOT NULL,
  amount   NUMBER(12,2) NOT NULL
);

-- 인덱스 생성
CREATE INDEX ix_fs_cust_dt  ON f_sales(cust_id, sales_dt);
CREATE INDEX ix_fs_prod_dt  ON f_sales(prod_id, sales_dt);
CREATE INDEX ix_prod_cat_br ON d_product(category, brand, prod_id);

-- 기본 통계 수집
BEGIN
  DBMS_STATS.GATHER_TABLE_STATS(USER,'D_CUSTOMER');
  DBMS_STATS.GATHER_TABLE_STATS(USER,'D_PRODUCT');
  DBMS_STATS.GATHER_TABLE_STATS(USER,'F_SALES');
END;
/
```

**실행 계획 분석 루틴**
조건절 이동의 효과를 확인하려면 항상 아래 방법으로 상세 실행 계획을 확인하세요.

```sql
SELECT *
FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(
  NULL, NULL,
  'ALLSTATS LAST +PREDICATE +ALIAS +NOTE +PROJECTION'
));
```

분석 포인트:
*   **`Predicate Information` 섹션**: 조건이 `access`(스캔 입력 단계에서 사용)로 적용되었는지, `filter`(읽은 후 상위에서 걸러짐)로 적용되었는지, 혹은 `pushed`(푸시다운됨)로 표시되는지 확인합니다.
*   **`Notes` 섹션**: `view merging`, `predicate moved`, `join filter` 등 옵티마이저의 변환 결정에 대한 힌트를 제공합니다.

---

## Predicate Pushdown의 본질과 이점

### 정의
Predicate Pushdown은 필터 조건을 실행 계획 트리에서 **더 아래쪽, 즉 데이터를 읽는 오퍼레이터에 더 가깝게 옮기는 변환**입니다. 이 변환은 주로 다음 위치로 조건을 이동시킵니다.
1.  **인라인 뷰 또는 CTE(Common Table Expression) 내부**
2.  **서브쿼리(주로 `IN`, `EXISTS`와 사용) 내부**
3.  **베이스 테이블이나 인덱스를 액세스하는 바로 앞 단계**
4.  **파티션 프루닝(Partition Pruning)이 발생하는 지점**
5.  **해시 조인에서 생성되는 Join Filter(Bloom Filter)로**

### Pushdown의 주요 이점
*   **I/O 감소**: 조건에 맞지 않는 데이터 블록을 아예 읽지 않게 되어 디스크 또는 버퍼 캐시 I/O가 줄어듭니다.
*   **중간 결과 집합 축소**: 조인, 정렬, 집계와 같은 후속 작업에 입력되는 행의 수가 줄어들어 CPU 및 메모리 사용량이 감소합니다.
*   **인덱스 활용도 향상**: 조건이 인덱스 액세스 단계로 내려가면(`access predicate`), 인덱스 범위 스캔이 더 효율적으로 작동합니다.
*   **파티션 프루닝 유도**: 파티션 키에 대한 조건이 명확해지면 관련 없는 파티션 전체를 스캔에서 제외할 수 있습니다.
*   **Join Filter 효율**: 해시 조인 시 빌드(Build) 측의 키로 생성된 Bloom Filter를 프로브(Probe) 측 스캔에 사용하여, 조인에 성공할 수 없는 행을 아주 초기 단계에서 걸러낼 수 있습니다.

---

## Pushdown이 적용되는 다양한 수준과 패턴

### 레벨 1: 뷰/인라인 뷰 내부로의 Pushdown
인라인 뷰를 사용할 때, 외부 쿼리의 조건이 뷰 내부로 푸시다운되지 않으면 비효율이 발생할 수 있습니다.

**비효율적인 패턴 (병합 및 푸시다운 차단)**
```sql
SELECT s.sales_id, s.amount
FROM   f_sales s
JOIN  (SELECT p.prod_id
       FROM   d_product p
       WHERE  p.category='ELEC' AND p.brand='B0') v
ON     v.prod_id = s.prod_id
WHERE  s.sales_dt BETWEEN :d1 AND :d2;
```
*   `NO_MERGE`가 암시적일 경우, `VIEW v` 연산자가 실행 계획에 남습니다.
*   외부 조건(`s.sales_dt`)이 뷰 `v` 내부로 푸시다운되지 않아, `v`의 결과 집합 전체와 `s`를 조인한 후에야 날짜 필터가 적용될 수 있습니다.

**효율적인 패턴 (병합 및 푸시다운 유도)**
```sql
SELECT /*+ MERGE(v) PUSH_PRED(v) */
       s.sales_id, s.amount
FROM   f_sales s
JOIN  (SELECT p.prod_id
       FROM   d_product p
       WHERE  p.category='ELEC' AND p.brand='B0') v
ON     v.prod_id = s.prod_id
WHERE  s.sales_dt BETWEEN :d1 AND :d2;
```
*   `MERGE(v)`: 인라인 뷰 `v`를 외부 쿼리와 평탄화하여 하나의 Query Block으로 만듭니다.
*   `PUSH_PRED(v)`: 조건절을 가능한 한 아래로 푸시다운하도록 옵티마이저에 힌트를 줍니다.
*   **기대 효과**: `d_product`의 필터와 `f_sales`의 날짜 조건이 함께 고려되어, 최적의 인덱스(`ix_fs_prod_dt`)를 사용한 조인 계획이 세워질 가능성이 높아집니다.

### 레벨 2: 서브쿼리 Pushdown (`PUSH_SUBQ`)
`IN` 또는 `EXISTS` 서브쿼리는 세미조인(Semijoin)으로 변환될 수 있습니다. 서브쿼리의 결과 집합을 먼저 축소하는 것이 유리하다고 판단되면 옵티마이저는 그 평가를 앞당깁니다(`PUSH_SUBQ`).

```sql
SELECT /*+ PUSH_SUBQ UNNEST */
       SUM(s.amount)
FROM   f_sales s
WHERE  s.prod_id IN (
  SELECT p.prod_id
  FROM   d_product p
  WHERE  p.category='ELEC' AND p.brand='B0'
)
AND    s.sales_dt BETWEEN :d1 AND :d2;
```
*   `UNNEST`: 서브쿼리를 일반 조인 형태로 풀어냅니다.
*   `PUSH_SUBQ`: 풀어낸 서브쿼리(세미조인)를 가능한 한 먼저 실행하여 결과 집합을 축소하도록 유도합니다.
*   **성공 신호**: 실행 계획에 `HASH JOIN SEMI` 또는 `NESTED LOOPS SEMI`가 나타나고, `d_product` 테이블이 먼저 처리되는 드라이빙 테이블이 됩니다.

### 레벨 3: 베이스 테이블 Access Predicate로의 Pushdown
조건이 인덱스 컬럼을 가공 없이 참조할 때(`SARGABLE`), 옵티마이저는 이를 `access predicate`로 만들어 인덱스 스캔 범위를 직접 제한합니다.

**비SARGABLE 쿼리 (푸시다운 실패)**
```sql
SELECT COUNT(*)
FROM f_sales s
WHERE TO_CHAR(s.sales_dt,'YYYY')='2024'; -- 컬럼 가공
```
*   `sales_dt` 컬럼에 함수가 적용되어 인덱스 범위 스캔을 사용할 수 없습니다. 풀 테이블 스캔 후 모든 행에 대해 필터링이 발생합니다.

**SARGABLE 쿼리 (푸시다운 성공)**
```sql
SELECT COUNT(*)
FROM f_sales s
WHERE s.sales_dt >= DATE '2024-01-01'
  AND s.sales_dt <  DATE '2025-01-01';
```
*   컬럼이 가공되지 않았으므로, `ix_fs_cust_dt` 인덱스의 선두 컬럼이 `sales_dt`가 아니라면 사용되지 않을 수 있지만, `sales_dt`만의 인덱스가 있다면 범위 스캔으로 푸시다운됩니다.

### 레벨 4: 파티션 프루닝으로의 Pushdown
대용량 파티션 테이블에서 조건을 파티션 키 컬럼으로 푸시다운하면, 관련 없는 파티션 전체를 접근에서 제외할 수 있습니다.

```sql
-- f_sales가 sales_dt 기준 RANGE 파티션 테이블이라고 가정
SELECT /*+ MERGE(v) */
       COUNT(*)
FROM   f_sales s
JOIN  (SELECT p.prod_id FROM d_product p WHERE p.brand='B0') v
ON     v.prod_id = s.prod_id
WHERE  s.sales_dt BETWEEN :d1 AND :d2; -- 파티션 키 조건
```
*   `MERGE(v)`로 뷰가 평탄화되면, `sales_dt` 조건이 `f_sales` 테이블 액세스 단계로 명확히 내려갑니다.
*   **성공 신호**: 실행 계획의 `PARTITION RANGE` 오퍼레이터 아래에 `PSTART`와 `PSTOP`이 특정 파티션 번호로 한정되어 나타납니다.

### 레벨 5: Join Filter(Bloom Filter) 생성
해시 조인에서 작은 빌드(Build) 테이블의 조인 키를 기반으로 Bloom Filter를 생성하면, 큰 프로브(Probe) 테이블을 스캔할 때 이 필터를 사용하여 조인에 실패할 행을 아주 초기에 걸러낼 수 있습니다.

```sql
SELECT /*+ USE_HASH(p) LEADING(p s) */
       SUM(s.amount)
FROM   f_sales s
JOIN   d_product p ON p.prod_id = s.prod_id
WHERE  p.category='ELEC'
AND    s.sales_dt BETWEEN :d1 AND :d2;
```
*   `d_product`(빌드 측)에서 `category='ELEC'`인 행들의 `prod_id`로 해시 테이블과 Bloom Filter가 생성됩니다.
*   `f_sales`(프로브 측)를 스캔할 때 각 행의 `prod_id`가 Bloom Filter를 통과하지 못하면 즉시 버려집니다.
*   **성공 신호**: 실행 계획에 `JOIN FILTER CREATE` 및 `JOIN FILTER USE` 오퍼레이터가 나타나고, `Predicate Information`에 `SYS_OP_BLOOM_FILTER` 조건이 추가됩니다.

---

## Join Predicate Pushdown: 전이성과 필터 전파

조인 조건의 동등성(=)을 이용하면, 한 테이블에 적용된 필터를 조인 상대편 테이블로 전파(푸시다운)할 수 있습니다. 이를 **전이성(Transitivity)** 최적화라고 합니다.

```sql
SELECT /*+ USE_NL(s) LEADING(c s) */
       COUNT(*)
FROM   d_customer c
JOIN   f_sales s ON s.cust_id = c.cust_id
WHERE  c.cust_id BETWEEN :lo AND :hi -- 이 조건이
AND    s.sales_dt BETWEEN :d1 AND :d2;
```
*   `c.cust_id BETWEEN :lo AND :hi` 조건은 조인 조건(`s.cust_id = c.cust_id`)을 통해 `s.cust_id BETWEEN :lo AND :hi`로 **전이**될 수 있습니다.
*   **성공 신호**: `Predicate Information`에서 `s` 테이블에 `access("S"."CUST_ID" BETWEEN :LO AND :HI)`라는 조건이 추가됩니다. 이로 인해 `ix_fs_cust_dt` 인덱스를 효율적으로 범위 스캔할 수 있게 됩니다.

**전이성 푸시다운이 효과적이기 위한 조건**
*   조인 조건이 **등치 조건**(=)이어야 합니다.
*   양쪽 컬럼의 데이터 타입이 동일해야 합니다.
*   통계 정보가 정확해야 합니다. 카디널리티를 과대평가하면 옵티마이저가 전이성 푸시다운의 이점을 과소평가하여 적용하지 않을 수 있습니다.

---

## Predicate Pullup: 때로는 조건을 위로 올리는 것이 최선이다

### Pullup의 역할과 필요성
모든 조건을 아래로만 밀어넣는 것이 최선은 아닙니다. 옵티마이저는 다음과 같은 이유로 조건을 일부 위로 끌어올릴(Pullup) 때가 있습니다.
1.  **의미 보존, 특히 외부 조인에서**: 외부 조인에서 내부 테이블에 대한 필터를 WHERE 절에 두면 외부 조인의 의미가 내부 조인으로 바뀝니다. 이를 방지하기 위해 조건을 ON 절 쪽으로 "올리는" 재배치가 필요할 수 있습니다.
2.  **다른 최적화 변환의 공간 확보**: 조건이 특정 위치에 있으면 뷰 병합(View Merging)이나 조인 순서 재배치 같은 더 중요한 변환이 방해받을 수 있습니다. 조건을 일시적으로 위로 올려 이러한 변환이 먼저 일어날 수 있도록 합니다.

옵티마이저의 이러한 전체적인 조건 재배치 활동을 **Predicate Move-Around**라고 합니다.

### 외부 조인에서의 주의사항
```sql
-- 안전한 패턴: 내부 테이블 필터를 ON 절에 명시
SELECT c.cust_id, s.amount
FROM   d_customer c
LEFT JOIN f_sales s
  ON   s.cust_id = c.cust_id
 AND   s.sales_dt BETWEEN :d1 AND :d2 -- ON 절에 필터
WHERE  c.region='USA';

-- 위험한 패턴: 내부 테이블 필터를 WHERE 절에 둠 (LEFT JOIN 의미 손상)
SELECT c.cust_id, s.amount
FROM   d_customer c
LEFT JOIN f_sales s ON s.cust_id = c.cust_id
WHERE  c.region='USA'
  AND  s.amount > 100; -- WHERE 절 필터는 NULL인 행을 제거하여 LEFT JOIN을 INNER JOIN으로 만듦
```
옵티마이저는 `WHERE s.amount > 100` 조건을 `s` 테이블 액세스 단계로 푸시다운하면 쿼리 의미가 바뀌므로, 이를 방지하기 위해 조건의 위치를 조정(Pullup 또는 재배치)하거나 변환을 포기합니다.

---

## Pushdown을 방해하는 장벽과 해결책

| 장벽 | 원인 | 해결 방안 |
| :--- | :--- | :--- |
| **외부 조인 NULL 보존** | 내부 테이블 필터 푸시다운이 LEFT JOIN 의미를 훼손 | 내부 테이블 조건은 ON 절에 작성 |
| **비결정적 함수** | `SYSDATE`, `ROWNUM`, 사용자 함수 등 실행 시점마다 결과가 달라 의미 보존 어려움 | 함수 제거 또는 사전 계산된 값 사용 |
| **컬럼 가공/형변환** | `WHERE UPPER(name)='A'` 형태로 인덱스 액세스 불가 | 가공 없이 컬럼 참조(SARGABLE)하도록 재작성, 함수 기반 인덱스 고려 |
| **복잡한 OR 조건** | OR 조건의 한쪽이 푸시다운 불가하면 전체가 차단 | `USE_CONCAT` 힌트로 OR-Expansion 유도, 각 분기별 최적화 |
| **집계/윈도우 함수** | `GROUP BY`, `DISTINCT`, 분석 함수는 연산 순서 변경 어려움 | 뷰 병합 또는 사전 집계 설계 검토 |

**OR-EXPAND를 통한 회복 예시**
```sql
-- OR로 인해 푸시다운이 제한될 수 있음
SELECT s.*
FROM   f_sales s
WHERE  (s.prod_id IN (SELECT p.prod_id FROM d_product p WHERE p.brand='B0'))
   OR  (s.cust_id IN (SELECT c.cust_id FROM d_customer c WHERE c.tier='VIP'));

-- USE_CONCAT 힌트로 OR 분기 확장
SELECT /*+ USE_CONCAT */
       s.*
FROM   f_sales s
WHERE  (s.prod_id IN (SELECT p.prod_id FROM d_product p WHERE p.brand='B0'))
   OR  (s.cust_id IN (SELECT c.cust_id FROM d_customer c WHERE c.tier='VIP'));
```
`USE_CONCAT` 힌트는 OR 조건을 `UNION ALL` 형태의 분기로 풀어, 각 분기 내에서 독립적인 Pushdown 최적화가 발생하도록 합니다.

---

## 힌트를 이용한 세밀한 제어

옵티마이저의 조건 재배치 방향을 조정하는 주요 힌트는 다음과 같습니다. 이 힌트들의 의미는 오랜 기간 안정적으로 유지되어 왔습니다.

### 핵심 제어 힌트
| 힌트 | 목적 |
| :--- | :--- |
| `PUSH_PRED(qb)` | 지정된 Query Block(인라인 뷰) 내부로 조건을 푸시다운하도록 **선호**합니다. |
| `NO_PUSH_PRED(qb)` | 지정된 Query Block 내부로 조건을 푸시다운하는 것을 **억제**합니다. |
| `MERGE(qb)` | 인라인 뷰를 외부 쿼리와 병합(View Merging)하도록 합니다. 병합은 푸시다운의 전제 조건이 될 수 있습니다. |
| `NO_MERGE(qb)` | 인라인 뷰 병합을 방지합니다. |
| `PUSH_SUBQ` | 서브쿼리를 가능한 한 빨리 실행하여 결과를 축소하도록 합니다. |
| `NO_PUSH_SUBQ` | 서브쿼리의 실행을 늦추도록 합니다. |
| `PX_JOIN_FILTER(alias)` | (병렬 쿼리에서) 지정된 테이블에 대해 Join Filter(Bloom Filter) 생성을 **강제**합니다. |

**사용 팁**: 힌트는 정확한 대상을 지정하기 위해 `qb_name()` 힌트와 함께 사용하는 것이 좋습니다. `NO_MERGE`를 QB 이름 없이 사용하면 의도치 않게 다른 뷰까지 병합이 막힐 수 있습니다.

---

## 실전 시나리오 분석

### 시나리오 1: 뷰 병합과 푸시다운의 시너지 효과 비교
```sql
-- Case 1: 병합과 푸시다운을 모두 방지
SELECT /*+ NO_MERGE(v) NO_PUSH_PRED(v) */
       COUNT(*)
FROM   f_sales s
JOIN  (SELECT p.prod_id FROM d_product p WHERE p.category='ELEC') v
ON v.prod_id = s.prod_id
WHERE s.sales_dt BETWEEN :d1 AND :d2;

-- Case 2: 병합과 푸시다운을 모두 유도
SELECT /*+ MERGE(v) PUSH_PRED(v) */
       COUNT(*)
FROM   f_sales s
JOIN  (SELECT p.prod_id FROM d_product p WHERE p.category='ELEC') v
ON v.prod_id = s.prod_id
WHERE s.sales_dt BETWEEN :d1 AND :d2;
```
**분석 포인트**:
*   두 실행 계획에서 `VIEW` 오퍼레이터의 유무를 확인합니다.
*   `Predicate Information`에서 조건(`sales_dt`, `category`)이 어느 단계(`access`/`filter`)에 적용되었는지 비교합니다.
*   `A-Rows`(실제 행수)와 `Buffers`(버퍼 읽기 수)를 확인하여 I/O 및 처리량 감소 효과를 측정합니다.

### 시나리오 2: Join Filter의 성능 영향 평가
```sql
SELECT /*+ LEADING(p s) USE_HASH(p) */
       COUNT(*)
FROM   f_sales s
JOIN   d_product p ON p.prod_id = s.prod_id
WHERE  p.category='ELEC';
```
**분석 포인트**:
*   실행 계획에 `JOIN FILTER CREATE` (빌드 측)와 `JOIN FILTER USE` (프로브 측)가 나타나는지 확인합니다.
*   `f_sales` 테이블 스캔 시 `A-Rows`가 Bloom Filter에 의해 얼마나 줄어드는지 관찰합니다.
*   대용량 파티션 테이블인 경우, `PX PARTITION HASH JOIN-FILTER`와 같은 형태로 파티션 단위 프루닝까지 발생하는지 확인합니다.

---

## 통계 정보의 중요성: 모든 결정의 근간

옵티마이저의 Pushdown/Pullup 결정은 궁극적으로 **조건 적용 후 남을 행의 수(카디널리티)에 대한 비용 추정**에 기반합니다. 따라서 통계 정보가 부정확하면 옵티마이저는 잘못된 결정을 내릴 수 있습니다.

**반드시 점검해야 할 통계 항목**:
*   **기본 테이블 통계**: 행수, 블록 수 등
*   **히스토그램**: `region`, `category` 같이 데이터 분포가 균일하지 않은(편향된) 컬럼의 선택도 추정 정확도를 높입니다.
*   **확장 통계**: `(cust_id, sales_dt)` 처럼 함께 자주 사용되어 상관관계가 높은 컬럼들의 결합 선택도를 정확히 추정합니다.

```sql
-- 히스토그램 및 확장 통계 수집 예시
BEGIN
  DBMS_STATS.GATHER_TABLE_STATS(
    USER, 'F_SALES',
    METHOD_OPT => 'FOR ALL COLUMNS SIZE AUTO' -- 히스토그램 자동 생성
  );
  DBMS_STATS.CREATE_EXTENDED_STATS(
    USER, 'F_SALES', '(cust_id, sales_dt)' -- 확장 통계 생성
  );
END;
/
```
**핵심 원리**: **카디널리티 추정이 정확할수록, 옵티마이저의 조건 재배치 결정도 더욱 최적화에 가까워집니다.**

---

## 결론

조건절 Pushdown, Pullup, 그리고 Join Predicate Pushdown은 Oracle 옵티마이저가 쿼리 성능을 극대화하기 위해 사용하는 근본적이고 강력한 최적화 도구들입니다.

*   **Predicate Pushdown**은 **"데이터를 읽기 전에 가능한 한 많이 줄여라"**는 최적화의 첫 번째 원칙을 구현합니다. I/O와 CPU 부하를 근본적으로 감소시키는 가장 효과적인 방법 중 하나입니다.
*   **Predicate Pullup**은 이 원칙에 대한 **현명한 예외**를 제공합니다. 쿼리의 정확한 의미를 지키고, 뷰 병합이나 조인 재배치 같은 더 큰 최적화의 문을 열기 위해 조건의 위치를 유연하게 조정합니다.
*   **Join Predicate Pushdown** (전이성, Join Filter)은 Pushdown의 효과를 **조인 관계와 스토리지 액세스 레벨까지 확장**시킵니다. 한 테이블의 필터가 조인을 타고 반대편 테이블의 스캔 범위를 좁히거나, Bloom Filter를 통해 블록/파티션 단위의 불필요한 I/O를 제거합니다.

이 모든 메커니즘의 효과는 궁극적으로 **정확한 통계 정보**와 **옵티마이저가 제공하는 실행 계획 정보를 정확히 해석하는 능력**에 달려 있습니다. `DBMS_XPLAN`을 통해 조건이 어디에 어떻게 적용되었는지(`Predicate Information`), 어떤 변환이 일어났는지(`Notes`)를 꼼꼼히 읽고 분석하는 습관이, 복잡한 SQL의 성능을 진단하고 튜닝하는 데 있어 가장 중요한 실무 역량이 될 것입니다.