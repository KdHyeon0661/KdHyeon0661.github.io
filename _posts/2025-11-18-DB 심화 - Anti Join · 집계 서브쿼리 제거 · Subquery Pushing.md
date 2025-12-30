---
layout: post
title: DB 심화 - Anti Join · 집계 서브쿼리 제거 · Subquery Pushing
date: 2025-11-18 18:25:23 +0900
category: DB 심화
---
# Anti Join · 집계 서브쿼리 제거 · Subquery Pushing

## 목표
이 글은 실행계획에서 **"실제로 어떤 작업이 수행되는지"**를 기준으로 다음과 같은 세 가지 핵심 최적화 기법을 심층적으로 다룹니다:

1. **Anti Join** - 데이터 부재 증명을 위한 최적화
2. **상관 집계 서브쿼리 제거(Aggregate Subquery Elimination)** - 반복 집계 호출 제거
3. **Subquery Pushing(PUSH_SUBQ)** - 서브쿼리 조기 평가를 통한 성능 개선

각 기법을 **논리적 의미 → 옵티마이저 변환 → 실행계획 해석 → 힌트 활용 → 주의사항**의 순서로 체계적으로 설명합니다.

## 핵심 전제
- Oracle의 CBO는 **의미 보존(semantic correctness)**이 확보된 경우에만 변환을 수행합니다.
- 변환이 실패하거나 회피되면 실행계획에 **FILTER, SORT, 반복 스칼라 서브쿼리** 형태가 남아 성능 저하를 초래합니다.
- 모든 튜닝은 **DBMS_XPLAN 실측(ALLSTATS LAST)**을 기준으로 검증해야 합니다.

---

## 실습 환경 준비: 스키마 및 데이터

아래는 Anti Join, Semi Join, 집계 제거 실습에 적합한 "차원(작은 테이블)-사실(큰 테이블)" 구조의 스키마입니다.

```sql
ALTER SESSION SET nls_date_format = 'YYYY-MM-DD';

DROP TABLE d_customer PURGE;
DROP TABLE d_product PURGE;
DROP TABLE f_sales PURGE;

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
  amount   NUMBER(12,2) NOT NULL,
  CONSTRAINT fk_sales_cust FOREIGN KEY (cust_id) REFERENCES d_customer(cust_id),
  CONSTRAINT fk_sales_prod FOREIGN KEY (prod_id) REFERENCES d_product(prod_id)
);

-- 사실 테이블 인덱스
CREATE INDEX ix_fs_cust_dt ON f_sales(cust_id, sales_dt);
CREATE INDEX ix_fs_prod_dt ON f_sales(prod_id, sales_dt);
CREATE INDEX ix_prod_cat_brand ON d_product(category, brand, prod_id);

-- 샘플 데이터 삽입
INSERT INTO d_customer VALUES (101,'USA','VIP');
INSERT INTO d_customer VALUES (202,'DEU','STD');
INSERT INTO d_customer VALUES (303,'FRA','STD');

INSERT INTO d_product VALUES (10,'ELEC','B0');
INSERT INTO d_product VALUES (20,'HOME','B1');
INSERT INTO d_product VALUES (30,'ELEC','B0');

INSERT INTO f_sales VALUES (1,101,10,DATE '2024-03-01',1, 100.00);
INSERT INTO f_sales VALUES (2,101,20,DATE '2024-03-03',2,  50.00);
INSERT INTO f_sales VALUES (3,202,10,DATE '2024-03-10',1, 200.00);
COMMIT;

-- 통계 수집
BEGIN
  DBMS_STATS.GATHER_TABLE_STATS(USER,'D_CUSTOMER');
  DBMS_STATS.GATHER_TABLE_STATS(USER,'D_PRODUCT');
  DBMS_STATS.GATHER_TABLE_STATS(USER,'F_SALES');
END;
/
```

### 실행계획 및 통계 확인 루틴

```sql
SELECT *
FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(
  NULL,NULL,'ALLSTATS LAST +PREDICATE +ALIAS +NOTE'));
```

---

## Anti Join — 데이터 "부재"를 효율적으로 증명하는 방법

### Anti Join의 논리적 의미

Anti Join은 "A 테이블에는 존재하지만 B 테이블에는 존재하지 않는 행"을 반환하는 연산입니다. SQL 관점에서는 `NOT EXISTS`, `NOT IN`(주의 필요), `MINUS`, 또는 `LEFT JOIN ... IS NULL` 형태로 표현됩니다.

관계대수 관점에서의 정의:
- **Anti-Semi Join**: 
  $$A \ \text{ANTI-JOIN}\ B = \{a \in A\ |\ \nexists b \in B,\ a.key=b.key\}$$

즉, **B 테이블에서 "존재하지 않음"을 효율적으로 증명**하는 비용을 최소화하는 것이 Anti Join 튜닝의 핵심입니다.

### 다양한 SQL 표현식과 의미 차이(특히 NULL 처리)

```sql
-- (1) NOT EXISTS: NULL 안전성 보장, 실무 표준
SELECT c.cust_id
FROM   d_customer c
WHERE  NOT EXISTS (
  SELECT 1 FROM f_sales s
  WHERE  s.cust_id = c.cust_id
  AND    s.sales_dt >= DATE '2024-03-01'
);

-- (2) NOT IN: 서브쿼리 결과에 NULL이 있으면 전체 결과가 UNKNOWN
SELECT c.cust_id
FROM   d_customer c
WHERE  c.cust_id NOT IN (
  SELECT s.cust_id FROM f_sales s
  WHERE s.sales_dt >= DATE '2024-03-01'
);

-- (3) MINUS: 집합 차집합(중복 제거 포함)
SELECT c.cust_id FROM d_customer c
MINUS
SELECT s.cust_id FROM f_sales s
WHERE  s.sales_dt >= DATE '2024-03-01';

-- (4) LEFT JOIN + IS NULL: 순수 동등 조인 조건에서 Anti Join과 동등
SELECT c.cust_id
FROM   d_customer c
LEFT JOIN f_sales s
  ON s.cust_id = c.cust_id
 AND s.sales_dt >= DATE '2024-03-01'
WHERE s.cust_id IS NULL;
```

**핵심 차이점**
- `NOT EXISTS`는 **서브쿼리 결과에 NULL이 포함되어도 의미가 변하지 않습니다**.
- `NOT IN`은 **IN 집합에 NULL이 하나라도 있으면 전체 결과가 UNKNOWN**이 되어 예기치 않게 결과 행 수가 감소할 수 있습니다.
- 따라서 실무에서 Anti Join 패턴은 **항상 `NOT EXISTS`를 기준으로 작성**하는 것이 안전합니다.

### Null-Aware Anti Join(NA-AJ) — NOT IN의 안전한 변환 메커니즘

Oracle은 `NOT IN`이나 `!= ALL`과 같은 NULL 민감 표현식을 Anti Join으로 변환할 때 의미 보존을 위해 **Null-Aware Anti Join**이라는 특별한 메커니즘을 사용합니다.

**변환 원리**
- 일반 Anti Join은 "키가 동일한 행이 없으면 통과"하는 단순한 로직입니다.
- NULL 값이 포함되면 "존재하지 않음"의 의미가 모호해지므로, Oracle은 **추가적인 NULL 체크 로직이 결합된 특수한 Anti Join**을 사용합니다.

**실행계획 특징**
- `HASH JOIN ANTI` 또는 `MERGE JOIN ANTI` 라인에 **NULL 보존을 위한 추가 연산이 결합된 형태**로 나타납니다.

**실무적 결론**
- 특별한 이유가 없다면, **처음부터 `NOT EXISTS`를 사용**하는 것이 실행계획 안정성과 의미적 정확성 모두에서 유리합니다.

### Anti Join 실행계획의 주요 형태

Anti Join 변환이 성공하면 실행계획에 명확히 표시됩니다.

- **HASH JOIN ANTI**
  - 내부(Build) 입력의 키 집합을 해시 테이블로 구성한 후, 외부(Probe) 입력을 스캔하며 **매칭되지 않는 행을 효율적으로 필터링**합니다.
  - 데이터 웨어하우스 환경과 대량 범위 스캔에서 표준 선택입니다.

- **NESTED LOOPS ANTI**
  - 외부 입력이 작고, 내부 입력에 선택도 높은 인덱스가 있을 때 유리합니다.
  - "외부 행마다 내부의 존재 여부만 확인"하고 즉시 다음 행으로 진행합니다.

- **MERGE JOIN ANTI**
  - 양쪽 입력이 조인 키 순서로 제공될 때 후보가 됩니다.
  - 정렬된 상태(또는 인덱스 순서)로 이미 제공된다면 비용이 매우 낮습니다.

**변환 실패 시 증상**
- 실행계획에 `FILTER` 연산이 남고, 외부 행마다 내부 서브쿼리가 반복 실행됩니다.
- 이는 **Anti Join 변환 실패의 가장 명확한 신호**입니다.

### Anti Join 방식 선택에 영향을 미치는 요인

CBO는 다음 요인들을 고려하여 Anti Join 방식을 결정합니다:

| 요인 | Hash Anti 적합성 | NL Anti 적합성 | Merge Anti 적합성 |
|------|------------------|----------------|-------------------|
| 외부 입력 크기 | 큼 | 작음 | 중간~큼 |
| 내부 입력 크기 | 중간~큼 | 작고 인덱스 강함 | 정렬/순서 보장됨 |
| 조인 키 분포 | 고르게 분산 | 높은 선택도 | 정렬된 상태 |
| 파티션/병렬 | Bloom/Join Filter 결합 유리 | OLTP 패턴 유리 | 정렬 비용 제로 시 최적 |

### 힌트를 활용한 Anti Join 방식 제어

```sql
-- Hash Anti Join 유도
SELECT /*+ USE_HASH(s) LEADING(s) */ c.cust_id
FROM d_customer c
WHERE NOT EXISTS (
  SELECT 1 FROM f_sales s
  WHERE s.cust_id = c.cust_id
    AND s.sales_dt >= DATE '2024-03-01'
);

-- NL Anti Join 유도(외부 입력이 작은 경우)
SELECT /*+ USE_NL(s) LEADING(c) */ c.cust_id
FROM d_customer c
WHERE NOT EXISTS (
  SELECT 1 FROM f_sales s
  WHERE s.cust_id = c.cust_id
    AND s.sales_dt >= DATE '2024-03-01'
);

-- Merge Anti Join 유도(정렬된 입력 활용)
SELECT /*+ USE_MERGE(s) LEADING(c) */ c.cust_id
FROM d_customer c
WHERE NOT EXISTS (
  SELECT 1 FROM f_sales s
  WHERE s.cust_id = c.cust_id
);
```

**특히 중요한 힌트**
- Anti Join 변환 자체를 조인 형태로 재작성하려면 `UNNEST` 사용
- 변환을 차단하려면 `NO_UNNEST` 사용
- 모든 쿼리 변환을 차단하여 의미와 실행계획을 고정하려면 `NO_QUERY_TRANSFORMATION` 사용

### Hash Anti Join과 Join Filter(Bloom Filter) 결합

Hash 계열 Anti Join에서는 내부 Build 입력을 기반으로 **Bloom Filter(Join Filter)**를 생성하고, Probe 테이블 스캔 단계에서 **조기 차단**을 수행할 수 있습니다.

실행계획 신호:
- `JOIN FILTER CREATE` (Build 측)
- `JOIN FILTER USE` (Probe 스캔 측)

특히 병렬 해시 조인/안티 조인에서 이 조기 차단 효과가 매우 큽니다.

### Anti Join의 주요 함정 및 주의사항

1. **NOT IN과 NULL 문제**
   - 결과 자체가 변경될 수 있어 의미와 성능 모두 위험합니다.
   - 실무 표준은 `NOT EXISTS` 사용입니다.

2. **외부 조인과 결합된 부재 판정**
   - NULL 보존 의미가 변형될 수 있어 Oracle이 Anti Join 변환을 회피할 수 있습니다.
   - 필요 시 `LEFT JOIN ... IS NULL` 형태로 고정하거나, 쿼리 블록 단위로 변환을 차단합니다.

3. **조인 키 가공/형변환**
   - 등가 증명이 실패하여 Anti Join 변환이 실패할 수 있습니다.
   - 조인 키는 **순수 컬럼 동등 조건**으로 유지하고, 필요한 가공은 **가상 컬럼이나 함수 기반 인덱스**로 이전합니다.

---

## 집계 서브쿼리 제거(Aggregate Subquery Elimination) — 상관 집계 반복 호출 최적화

### 성능 문제의 근본 원인

상관 집계 서브쿼리는 "외부 행마다 집계 작업을 반복 수행"하는 비효율을 초래합니다.

```sql
-- 비효율 패턴: 외부 행마다 SUM 함수 호출
SELECT p.prod_id,
       (SELECT SUM(s.amount)
        FROM f_sales s
        WHERE s.prod_id = p.prod_id
          AND s.sales_dt BETWEEN DATE '2024-03-01' AND DATE '2024-03-31') AS m_sum
FROM d_product p
WHERE p.category = 'ELEC';
```

CBO가 제거 또는 언네스팅을 수행하지 못하면 실행계획에:
- `SCALAR SUBQUERY` 또는 `FILTER` 연산이 남고
- **A-Rows(외부 반복 횟수)**만큼 집계 작업이 실행됩니다.

### 최적화 목표: 사전 집계 + 조인 패턴

```sql
-- 개선 패턴: 사전 집계(한 번) + 조인
SELECT /*+ UNNEST */
       p.prod_id, ss.m_sum
FROM d_product p
LEFT JOIN (
  SELECT s.prod_id, SUM(s.amount) m_sum
  FROM f_sales s
  WHERE s.sales_dt BETWEEN DATE '2024-03-01' AND DATE '2024-03-31'
  GROUP BY s.prod_id
) ss
ON ss.prod_id = p.prod_id
WHERE p.category='ELEC';
```

기대 실행계획:
- `HASH GROUP BY` (ss 사전 집계 1회 수행)
- `HASH JOIN OUTER` 또는 `NESTED LOOPS OUTER`
- `SCALAR SUBQUERY` 연산 제거

### WHERE 절 상관 집계 제거 패턴

```sql
-- 비효율 패턴: 고객마다 SUM 호출 후 비교
SELECT c.cust_id
FROM d_customer c
WHERE (SELECT SUM(s.amount)
       FROM f_sales s
       WHERE s.cust_id=c.cust_id
         AND s.sales_dt BETWEEN :d1 AND :d2) >= 100000;

-- 개선 패턴: 사전 집계 + HAVING + 조인
WITH agg AS (
  SELECT s.cust_id, SUM(s.amount) sum_amt
  FROM f_sales s
  WHERE s.sales_dt BETWEEN :d1 AND :d2
  GROUP BY s.cust_id
  HAVING SUM(s.amount) >= 100000
)
SELECT c.cust_id
FROM d_customer c
JOIN agg ON agg.cust_id = c.cust_id;
```

**패턴 분류 가이드**
- `COUNT(*) = 0` 또는 `COUNT(*) > 0` 패턴은 존재/부존재 문제이므로 **SEMI/ANTI JOIN으로 변환**하는 것이 더 자연스럽습니다.
- `SUM/MAX/MIN/AVG` 비교 패턴은 **사전 집계 + 조인(또는 APPLY)**으로 해결해야 합니다.

### COUNT(DISTINCT) 상관 집계 제거 전략

```sql
-- 비효율 패턴: 행마다 DISTINCT 집계 수행
SELECT c.cust_id
FROM d_customer c
WHERE (SELECT COUNT(DISTINCT s.prod_id)
       FROM f_sales s
       WHERE s.cust_id=c.cust_id
         AND s.sales_dt>=:d1) >= 10;

-- 개선 패턴: (1) 중복 제거 → (2) 집계 → (3) 조인
WITH prod_set AS (
  SELECT DISTINCT s.cust_id, s.prod_id
  FROM f_sales s
  WHERE s.sales_dt>=:d1
),
agg AS (
  SELECT cust_id, COUNT(*) prod_cnt
  FROM prod_set
  GROUP BY cust_id
  HAVING COUNT(*) >= 10
)
SELECT c.cust_id
FROM d_customer c
JOIN agg ON agg.cust_id = c.cust_id;
```

**최적화 의도**
- `DISTINCT`의 위치를 조정하여 **집계 입력 크기를 최소화**합니다.

### 자동 제거가 차단되는 일반적 조건

| 차단 요인 | 차단 이유 | 해결 방안 |
|-----------|----------|-----------|
| 외부 조인 + 상관 집계 | NULL 보존 의미 변경 가능성 | 수동 재작성, ON 절로 필터 이동 |
| 분석 함수/ROWNUM/Top-N | 순서/프레임 의존성으로 재배치 위험 | 인라인 뷰 물리화(MATERIALIZE) 후 최적화 |
| 비결정적 함수 | 동일 입력에 대해 결과 변동 가능성 | 함수 제거 또는 대체 |
| 조인 키 가공/형변환 | 등가 증명 실패 | 가상 컬럼/FBI 사용, 데이터 타입 일치화 |
| 카디널리티 추정 오류 | 제거 후 조인이 더 비싸다고 판단 | 통계/히스토그램/확장 통계 보정 |

### 집계 제거 관련 핵심 힌트

- `UNNEST` / `NO_UNNEST`: 상관 서브쿼리의 조인 형태 변환 허용/차단
- `PUSH_SUBQ`: 서브쿼리를 **먼저 계산하여 작은 집합으로 만든 후 조인**하도록 우선순위 부여
- `MATERIALIZE` / `INLINE`: WITH/인라인 뷰를 중간 결과로 고정하거나 병합
- `NO_QUERY_TRANSFORMATION`: 쿼리 블록 단위로 모든 변환 차단(실행계획 고정용)

---

## Subquery Pushing — `PUSH_SUBQ`를 통한 "조기 축소 후 조인"

### Pushing의 개념적 의미

`PUSH_SUBQ`는 **"서브쿼리를 더 먼저 평가하라"**는 옵티마이저 힌트입니다. 일반적으로 `IN/EXISTS` 서브쿼리를:

1. 조인으로 변환하고(`UNNEST`)
2. 그 결과를 기반으로 큰 테이블을 효율적으로 탐색하거나
3. Hash Semi/Anti + Bloom Filter로 **스캔 작업 자체를 감소**시키도록 유도합니다.

### 기본 예제: IN 서브쿼리의 조기 평가

```sql
-- 원본 쿼리
SELECT s.*
FROM f_sales s
WHERE s.prod_id IN (
  SELECT p.prod_id
  FROM d_product p
  WHERE p.category='ELEC'
    AND p.brand='B0'
)
AND s.sales_dt BETWEEN DATE '2024-03-01' AND DATE '2024-03-31';

-- PUSH_SUBQ 적용: prod_id 집합 먼저 축소 → 세미 조인
SELECT /*+ PUSH_SUBQ UNNEST USE_HASH(p) LEADING(p s) */
       s.*
FROM f_sales s
WHERE s.prod_id IN (
  SELECT p.prod_id
  FROM d_product p
  WHERE p.category='ELEC'
    AND p.brand='B0'
)
AND s.sales_dt BETWEEN DATE '2024-03-01' AND DATE '2024-03-31';
```

기대 실행계획:
- `HASH JOIN SEMI`
- `JOIN FILTER CREATE/USE`가 포함되면 **스캔 단계까지 푸시가 적용된 것**을 의미합니다.

### FILTER vs SEMI JOIN — Pushing 성공 여부 판단

- **실패한 실행계획**
  - `FILTER` 연산 존재
  - 외부 행 수만큼 서브쿼리 반복 실행
- **성공한 실행계획**
  - `HASH JOIN SEMI` 또는 `NESTED LOOPS SEMI`
  - 서브쿼리 라인 제거(조인으로 평탄화)

따라서 Pushing 튜닝의 본질은:

> **"FILTER를 SEMI/ANTI JOIN으로 변환하는 것"**

이라고 이해할 수 있습니다.

### NO_PUSH_SUBQ가 필요한 상황

모든 Pushing이 항상 성능 향상을 보장하는 것은 아닙니다. 특히 다음과 같은 상황에서는 주의가 필요합니다:

- 서브쿼리 결과가 **너무 크거나 변동성이 높은 경우**
  - "먼저 생성하는 비용"이 오히려 손실일 수 있습니다.
- Pushing 이후 큰 테이블이 **비효율적인 탐색 경로로 고정되는 경우**

이런 상황에서는 다음과 같이 원래 형태를 유지하는 것이 안정적입니다:

```sql
SELECT /*+ NO_PUSH_SUBQ NO_UNNEST */
...
```

---

## 세 가지 기법의 통합 적용: Anti Join + 집계 제거 + Pushing

### 복합 시나리오: "2024년 3월에 ELEC/B0 제품을 전혀 구매하지 않은 VIP 고객 조회"

```sql
-- 비효율 원본: COUNT(*)=0 + IN 서브쿼리 + 상관 집계
SELECT c.cust_id
FROM d_customer c
WHERE c.tier='VIP'
  AND (SELECT COUNT(*)
       FROM f_sales s
       WHERE s.cust_id=c.cust_id
         AND s.prod_id IN (
              SELECT p.prod_id
              FROM d_product p
              WHERE p.category='ELEC' AND p.brand='B0'
           )
         AND s.sales_dt BETWEEN :d1 AND :d2) = 0;
```

**문제점 분석**
- VIP 고객 수만큼 `COUNT(*)` 상관 집계 반복 실행
- 각 집계 내부에서 다시 `IN (서브쿼리)` 반복 실행
- 결과적으로 **중첩 FILTER + 스칼라 집계 호출** 구조

### 최적화 재작성

```sql
SELECT /*+ PUSH_SUBQ UNNEST USE_HASH(p) LEADING(p c) */
       c.cust_id
FROM d_customer c
WHERE c.tier='VIP'
  AND NOT EXISTS (
    SELECT 1
    FROM f_sales s
    JOIN (SELECT p.prod_id
          FROM d_product p
          WHERE p.category='ELEC' AND p.brand='B0') p
      ON p.prod_id = s.prod_id
    WHERE s.cust_id = c.cust_id
      AND s.sales_dt BETWEEN :d1 AND :d2
  );
```

**변화 요약**
1. `COUNT(*)=0` → `NOT EXISTS` 변환
   - 상관 집계 제거
   - Anti Join 변환 가능
2. 제품 조건 사전 축소(`PUSH_SUBQ`)
   - 작은 prod_id 집합으로 최적화
3. Hash Anti + Bloom Filter 활용
   - 사실 테이블 스캔량 급감

### 실행계판 판독 체크리스트

- **Anti Join 변환 성공 여부**
  - `HASH JOIN ANTI` / `NL ANTI` / `MERGE ANTI` 확인
- **집계 제거 성공 여부**
  - `SCALAR SUBQUERY` 라인 부재 확인
  - 사전 집계(`HASH GROUP BY`) + 조인 구조 확인
- **Pushing 성공 여부**
  - `HASH JOIN SEMI/ANTI` 확인
  - `FILTER` 연산 부재 확인
  - `JOIN FILTER CREATE/USE` 존재 시 최적 상태

---

## 실전 미니 실습 세트

```sql
-- A. Hash Anti vs NL Anti 비교
EXPLAIN PLAN FOR
SELECT /*+ USE_HASH(s) LEADING(s c) */
       c.cust_id
FROM d_customer c
WHERE NOT EXISTS (
  SELECT 1 FROM f_sales s
  WHERE s.cust_id=c.cust_id
);
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);

EXPLAIN PLAN FOR
SELECT /*+ USE_NL(s) LEADING(c s) */
       c.cust_id
FROM d_customer c
WHERE NOT EXISTS (
  SELECT 1 FROM f_sales s
  WHERE s.cust_id=c.cust_id
);
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);

-- B. 상관 SUM 제거 비교
EXPLAIN PLAN FOR
SELECT p.prod_id,
       (SELECT SUM(s.amount)
        FROM f_sales s
        WHERE s.prod_id=p.prod_id) AS sum_amt
FROM d_product p;
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);

EXPLAIN PLAN FOR
SELECT /*+ UNNEST */
       p.prod_id, ss.sum_amt
FROM d_product p
LEFT JOIN (
  SELECT prod_id, SUM(amount) sum_amt
  FROM f_sales
  GROUP BY prod_id
) ss ON ss.prod_id=p.prod_id;
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);

-- C. PUSH_SUBQ 효과 확인
EXPLAIN PLAN FOR
SELECT /*+ PUSH_SUBQ UNNEST USE_HASH(p) LEADING(p s) */
       s.*
FROM f_sales s
WHERE s.prod_id IN (
  SELECT p.prod_id
  FROM d_product p
  WHERE p.category='ELEC'
);
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);
```

---

## 결론

1. **Anti Join**은 데이터 "부재"를 증명하는 가장 효율적인 경로입니다. `NOT EXISTS`를 사용하면 의미적 안전성과 CBO 변환 안정성을 모두 확보할 수 있습니다. NULL 민감성을 이해하고 적절한 패턴을 선택하는 것이 중요합니다.

2. **집계 서브쿼리 제거**는 상관 집계의 행 단위 반복 호출을 제거하여 "한 번의 집계 + 조인"으로 성능을 극대화합니다. 사전 집계 패턴과 적절한 힌트 사용이 성공적인 최적화의 핵심입니다.

3. **Subquery Pushing**은 작은 키 집합을 먼저 생성하여 큰 테이블을 효율적으로 탐색하도록 유도합니다. `FILTER` 연산을 `SEMI/ANTI JOIN`으로 변환하는 것이 성능 향상의 지표가 됩니다.

4. 이러한 세 가지 기법을 통합적으로 적용하면 복잡한 쿼리의 성능을 극적으로 개선할 수 있습니다. 특히 중첩된 서브쿼리와 상관 집계가 결합된 패턴에서는 구조적 재작성이 필수적입니다.

5. 모든 튜닝의 최종 검증은 **실측 실행계획과 통계 분석**을 통해 이루어져야 합니다. 실행계획에서 `FILTER`가 사라지고 `SEMI/ANTI JOIN`과 사전 집계 패턴이 나타나면 튜닝이 성공적으로 완료된 것입니다.

이러한 고급 최적화 기법을 숙달하면 복잡한 비즈니스 로직을 가진 쿼리에서도 안정적이고 효율적인 성능을 달성할 수 있습니다. 각 기법의 동작 원리를 깊이 이해하고 상황에 맞게 적절히 조합하는 것이 전문가의 역량입니다.