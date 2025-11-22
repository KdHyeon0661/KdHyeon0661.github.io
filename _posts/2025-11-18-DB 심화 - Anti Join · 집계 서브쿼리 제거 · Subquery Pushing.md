---
layout: post
title: DB 심화 - Anti Join · 집계 서브쿼리 제거 · Subquery Pushing
date: 2025-11-18 18:25:23 +0900
category: DB 심화
---
# Anti Join · 집계 서브쿼리 제거 · Subquery Pushing

**목표**
이 글은 실행계획에서 **“무엇이 실제로 일어났는지”**를 기준으로,
1) **Anti Join**(부재 증명),
2) **상관 집계 서브쿼리 제거(Aggregate Subquery Elimination)**,
3) **Subquery Pushing(PUSH_SUBQ)**
를 **논리 의미 → 옵티마이저 변환 → 실행계획 판독 → 힌트/재작성 → 반례/함정** 순서로 끝까지 정리한다.

> 전제
> - Oracle의 CBO는 **의미 보존(semantic correctness)**이 증명될 때만 변환을 수행한다.
> - 변환이 실패/회피되면 플랜은 **FILTER / SORT / 반복 스칼라 서브쿼리** 형태로 남아 “느린 계획”이 된다.
> - 이 글의 모든 튜닝은 **DBMS_XPLAN 실측(ALLSTATS LAST)**을 기준으로 검증한다.

---

## 준비 스키마 & 데이터(확장 버전)

아래는 Anti/Semi/집계 제거 실습에 쓰기 좋은 “차원(작음)–사실(큼)” 구조다.

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

-- 사실 테이블 인덱스(비고유 포함)
CREATE INDEX ix_fs_cust_dt ON f_sales(cust_id, sales_dt);
CREATE INDEX ix_fs_prod_dt ON f_sales(prod_id, sales_dt);
CREATE INDEX ix_prod_cat_brand ON d_product(category, brand, prod_id);

-- 샘플 데이터
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

BEGIN
  DBMS_STATS.GATHER_TABLE_STATS(USER,'D_CUSTOMER');
  DBMS_STATS.GATHER_TABLE_STATS(USER,'D_PRODUCT');
  DBMS_STATS.GATHER_TABLE_STATS(USER,'F_SALES');
END;
/
```

관찰은 항상 아래로 한다.

```sql
SELECT *
FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(
  NULL,NULL,'ALLSTATS LAST +PREDICATE +ALIAS +NOTE'));
```

---

## Anti Join — “없음을” 가장 싸게 증명하는 방법

### Anti Join의 논리 의미

Anti Join은 “A에는 있지만 B에는 없는 행”을 반환하는 연산이다.
SQL 관점에서는 `NOT EXISTS`, `NOT IN`(주의), `MINUS`, 혹은 `LEFT JOIN … IS NULL` 형태로 표현된다.

관계대수 관점(직관):

- **Anti-Semi Join**
  $$A \ \text{ANTI-JOIN}\ B = \{a \in A\ |\ \nexists b \in B,\ a.key=b.key\}$$

즉, **B에서 “존재하지 않음”을 증명**하는 비용을 가장 싸게 만들면 곧 Anti Join 튜닝이다.

---

### SQL 표현식별 의미 차이(특히 NULL)

```sql
-- (1) NOT EXISTS : NULL 안전, 실무 표준
SELECT c.cust_id
FROM   d_customer c
WHERE  NOT EXISTS (
  SELECT 1 FROM f_sales s
  WHERE  s.cust_id = c.cust_id
  AND    s.sales_dt >= DATE '2024-03-01'
);

-- (2) NOT IN : 서브쿼리 결과에 NULL이 있으면 전체가 UNKNOWN
SELECT c.cust_id
FROM   d_customer c
WHERE  c.cust_id NOT IN (
  SELECT s.cust_id FROM f_sales s
  WHERE s.sales_dt >= DATE '2024-03-01'
);

-- (3) MINUS : 집합 차집합(중복 제거 포함)
SELECT c.cust_id FROM d_customer c
MINUS
SELECT s.cust_id FROM f_sales s
WHERE  s.sales_dt >= DATE '2024-03-01';

-- (4) LEFT JOIN + IS NULL : Anti Join과 동등(조건이 순수 동등조인일 때)
SELECT c.cust_id
FROM   d_customer c
LEFT JOIN f_sales s
  ON s.cust_id = c.cust_id
 AND s.sales_dt >= DATE '2024-03-01'
WHERE s.cust_id IS NULL;
```

**핵심**
- `NOT EXISTS`는 **서브쿼리 집합에 NULL이 있어도 의미가 변하지 않는다**.
- `NOT IN`은 **IN 집합에 NULL이 하나라도 있으면 전체가 UNKNOWN**이 되어 결과가 비정상적으로 줄어들 수 있다.
- 따라서 실무에서 Anti Join은 **무조건 `NOT EXISTS`를 기준으로 작성**하는 습관이 안전하다.

---

### Null-Aware Anti Join(NA-AJ) — NOT IN을 안전하게 변환하는 내부 장치

Oracle은 `NOT IN`/`!= ALL` 같은 NULL 민감 표현을 Anti Join으로 변환할 때,
의미 보존을 위해 **Null-Aware Anti Join**을 사용한다.

- 변환 요지
  - 일반 Anti Join은 “키가 같은 row가 없으면 통과”인데,
  - NULL이 섞이면 “없음”의 의미가 애매해져서 **추가 NULL 체크 로직을 결합한 Anti Join**으로 실행한다.
- 실행계획 특징
  - `HASH JOIN ANTI` 또는 `MERGE JOIN ANTI` 라인에
    **NULL 보존 목적의 연산이 결합된 형태**로 나타난다.

**튜닝 실무 결론**
- `NOT IN`을 써야만 하는 상황이 아니라면,
  **처음부터 `NOT EXISTS`로 명시하는 것이 플랜·의미·성능 모두 안정적**이다.

---

### Anti Join 실행계획 형태

Anti Join으로 변환이 성공하면 오퍼레이터 이름이 명확히 드러난다.

- `HASH JOIN ANTI`
  - 내부(Build) 입력에서 키 집합을 해시 테이블로 구성한 뒤,
    외부(Probe) 입력을 스캔하며 **매칭이 없음을 빠르게 거른다**.
  - DW/대량 범위 스캔에서 표준 선택.
- `NESTED LOOPS ANTI`
  - 바깥 소량, 안쪽 인덱스 고선택도일 때 유리.
  - “바깥 row마다 안쪽 존재성만 확인”하고 바로 다음 row로 넘어간다.
- `MERGE JOIN ANTI`
  - 양쪽이 조인키 순서로 공급될 때 후보.
  - 정렬(또는 인덱스 순서)이 이미 있다면 비용이 매우 낮다.

**변환 실패 시**
- 플랜에 `FILTER` 라인이 남고
- 바깥 row마다 안쪽 서브쿼리가 반복 실행된다.
이는 **Anti Join 실패의 가장 명확한 신호**다.

---

### 어떤 Anti Join이 선택되는가? (비용 요인)

CBO는 아래 요인으로 Anti Join 방식을 택한다.

| 요인 | Hash Anti | NL Anti | Merge Anti |
|---|---|---|---|
| 외부 입력 크기 | 큼 | 작음 | 중간~큼 |
| 내부 입력 크기 | 중간~큼 | 작고 인덱스 강함 | 정렬/순서가 이미 있음 |
| 조인키 분포 | 고르게 분산 | 고선택도 | 정렬되어 있거나 인덱스 순서 |
| 파티션/병렬 | Bloom/Join Filter 결합 유리 | OLTP 패턴 유리 | 정렬 비용이 0이면 최강 |

---

### 힌트로 Anti Join 제어

```sql
-- Hash Anti 유도
SELECT /*+ USE_HASH(s) LEADING(s) */ c.cust_id
FROM d_customer c
WHERE NOT EXISTS (
  SELECT 1 FROM f_sales s
  WHERE s.cust_id = c.cust_id
    AND s.sales_dt >= DATE '2024-03-01'
);

-- NL Anti 유도(바깥이 작은 경우)
SELECT /*+ USE_NL(s) LEADING(c) */ c.cust_id
FROM d_customer c
WHERE NOT EXISTS (
  SELECT 1 FROM f_sales s
  WHERE s.cust_id = c.cust_id
    AND s.sales_dt >= DATE '2024-03-01'
);

-- Merge Anti 유도(입력이 정렬 순서로 공급되게 설계)
SELECT /*+ USE_MERGE(s) LEADING(c) */ c.cust_id
FROM d_customer c
WHERE NOT EXISTS (
  SELECT 1 FROM f_sales s
  WHERE s.cust_id = c.cust_id
);
```

**특히 중요**
- Anti 변환 자체를 **풀어 조인으로 재작성**하려면 `UNNEST`, 막으려면 `NO_UNNEST`.
- 변환 전체를 막아 의미/플랜을 고정하려면 `NO_QUERY_TRANSFORMATION`.

---

### Hash Anti + Join Filter(Bloom) 결합

Hash 계열 Anti Join에서는 내부 Build 입력으로 **Bloom Filter(Join Filter)**를 만들고,
Probe 테이블 스캔 단계에서 **조기 차단**할 수 있다.

플랜 신호:
- `JOIN FILTER CREATE` (Build 쪽)
- `JOIN FILTER USE` (Probe 스캔 쪽)

특히 병렬 해시 조인/안티 조인에서 이 조기 차단 효과가 크다.

---

### Anti Join의 대표 반례/함정

1) **NOT IN + NULL 문제**
   - 결과 자체가 바뀐다 → 의미/성능 모두 위험.
   - 실무 표준은 `NOT EXISTS`.

2) **외부조인과 결합된 부재 판정**
   - NULL 보존 의미가 변형될 수 있어 Oracle이 Anti 변환을 회피할 수 있다.
   - 필요 시 수동으로 `LEFT JOIN … IS NULL`로 형태를 고정하거나, QB 단위로 변환을 막는다.

3) **조인키 가공/형변환**
   - 등가 증명이 실패 → Anti 변환 실패(FILTER로 남음).
   - 조인키는 **순수 컬럼 동등**으로 두고, 필요한 가공은 **가상 컬럼/FBI**로 옮긴다.

---

## Aggregate Subquery Elimination — 상관 집계 반복을 “한 번의 집계 + 조인”으로

### 왜 느린가?

상관 집계 서브쿼리는 “바깥 row마다 집계를 다시 수행”한다.

```sql
-- 느린 패턴: 바깥 row마다 SUM이 호출
SELECT p.prod_id,
       (SELECT SUM(s.amount)
        FROM f_sales s
        WHERE s.prod_id = p.prod_id
          AND s.sales_dt BETWEEN DATE '2024-03-01' AND DATE '2024-03-31') AS m_sum
FROM d_product p
WHERE p.category = 'ELEC';
```

CBO가 제거/언네스트를 못하면 플랜에:
- `SCALAR SUBQUERY`
- 또는 `FILTER`
가 남고, **A-Rows(바깥 반복)** 만큼 집계가 실행된다.
Jonathan Lewis가 정리한 스칼라 서브쿼리 반복·캐싱·언네스트 조건이 이 문제의 핵심이다.

---

### 제거의 목표: 사전 집계 + 조인

```sql
-- 개선: 사전 집계(한 번) + 조인
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

기대 플랜:
- `HASH GROUP BY` (ss 사전 집계 1회)
- `HASH JOIN OUTER` 또는 `NESTED LOOPS OUTER`
- `SCALAR SUBQUERY` 사라짐

---

### WHERE 절 상관 집계 제거

```sql
-- 느림: 고객마다 SUM 호출 후 비교
SELECT c.cust_id
FROM d_customer c
WHERE (SELECT SUM(s.amount)
       FROM f_sales s
       WHERE s.cust_id=c.cust_id
         AND s.sales_dt BETWEEN :d1 AND :d2) >= 100000;

-- 개선: 사전 집계 + HAVING + 조인
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

**패턴 분류**
- `COUNT(*) = 0 / > 0`
  → 존재/비존재의 문제다.
  → **(NOT) EXISTS → SEMI/ANTI JOIN**으로 푸는 게 더 자연스럽다.
- `SUM/MAX/MIN/AVG` 비교
  → 사전 집계 + 조인(또는 APPLY)로 풀어야 한다.

---

### COUNT(DISTINCT) 상관 집계 제거

```sql
-- 느림: row마다 DISTINCT 집계
SELECT c.cust_id
FROM d_customer c
WHERE (SELECT COUNT(DISTINCT s.prod_id)
       FROM f_sales s
       WHERE s.cust_id=c.cust_id
         AND s.sales_dt>=:d1) >= 10;

-- 개선: (1) 먼저 중복 제거, (2) 집계, (3) 조인
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

**의도**
- `DISTINCT`의 위치를 바꿔 **집계 입력을 최소화**한다.

---

### 자동 제거가 “막히는” 전형적 조건

| 차단 요인 | 왜 막히나 | 해결 |
|---|---|---|
| 외부조인 + 상관집계 | NULL 보존 의미가 바뀔 수 있음 | 수동 재작성, ON절로 필터 이동 |
| 분석함수/ROWNUM/Top-N | 순서/프레임 의존이라 재배치 위험 | 인라인뷰 물리화(MATERIALIZE) 후 최적화 |
| 비결정 함수 | 같은 입력이라도 결과가 달라질 수 있어 의미 불명 | 함수 제거/대체 |
| 조인키 가공/형변환 | 등가 증명이 실패 | 가상컬럼/FBI 사용, 타입 일치 |
| 카디널리티 오판 | 제거 후 조인이 더 비싸다고 판단 | 통계/히스토그램/확장통계 보정 |

---

### 집계 제거 관련 핵심 힌트

- `UNNEST` / `NO_UNNEST` : 상관 서브쿼리의 조인 형태 변환을 허용/차단
- `PUSH_SUBQ` : 서브쿼리를 **먼저 계산해 작은 집합으로 만든 뒤 조인**하도록 우선순위 부여
- `MATERIALIZE` / `INLINE` : WITH/인라인뷰를 중간 결과로 고정하거나 병합
- `NO_QUERY_TRANSFORMATION` : QB 단위로 변환 전체 차단(플랜 고정용)

---

## Subquery Pushing — `PUSH_SUBQ`로 “먼저 좁혀서” 조인하기

### Push의 의미

`PUSH_SUBQ`는 **“서브쿼리를 더 먼저 평가해라”**는 옵티마이저 힌트다.
보통 `IN/EXISTS` 서브쿼리를:

1) 조인으로 풀어내고(`UNNEST`)
2) 그 결과를 기반으로 큰 테이블을 탐색하게 하거나
3) Hash Semi/Anti + Bloom Filter로 **스캔부터 줄이도록** 유도한다.

---

### 기본 예제: IN 서브쿼리의 조기 평가

```sql
-- 원형
SELECT s.*
FROM f_sales s
WHERE s.prod_id IN (
  SELECT p.prod_id
  FROM d_product p
  WHERE p.category='ELEC'
    AND p.brand='B0'
)
AND s.sales_dt BETWEEN DATE '2024-03-01' AND DATE '2024-03-31';

-- PUSH_SUBQ로 먼저 prod_id 집합 축소 → 세미조인
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

기대 플랜:
- `HASH JOIN SEMI`
- `JOIN FILTER CREATE/USE`가 붙으면 **스캔 단계까지 푸시가 내려간 것**.

---

### FILTER vs SEMI JOIN — Push 성공/실패 판독

- **실패 플랜**
  - `FILTER`
  - 바깥 row만큼 서브쿼리 반복
- **성공 플랜**
  - `HASH JOIN SEMI` 또는 `NESTED LOOPS SEMI`
  - 서브쿼리 라인이 사라짐(Join으로 평탄화)

그래서 Push 튜닝은 결국:

> “FILTER를 SEMI/ANTI로 바꾸는 것”

이라고 생각해도 된다.

---

### NO_PUSH_SUBQ가 왜 필요한가?

모든 Push가 항상 이득은 아니다. 특히:

- 서브쿼리 결과가 **너무 크거나 변동 폭이 크면**
  “먼저 만들기 비용”이 오히려 손해일 수 있다.
- Push 이후 큰 테이블이 **비효율적 탐색 경로**로 고정될 위험이 있다.

이때는:

```sql
SELECT /*+ NO_PUSH_SUBQ NO_UNNEST */
...
```

로 “원래대로” 두는 것이 안정적인 경우가 있다.

---

## 세 기법의 결합: Anti + 집계 제거 + Push

### 시나리오

“2024년 3월에 **ELEC/B0 제품을 전혀 사지 않은 VIP 고객**”

```sql
-- 느린 원형: COUNT(*)=0 + IN 서브쿼리 + 상관집계
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

문제:
- VIP 고객 수만큼 `COUNT(*)` 상관집계가 반복
- 그 안에서 다시 `IN (subquery)`가 반복
- 결과적으로 **중첩 FILTER + 스칼라 집계 호출**이 된다.

---

### 최적 재작성

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
1) `COUNT(*)=0` → `NOT EXISTS`
   - 상관 집계 제거
   - Anti Join으로 변환 가능
2) 제품 조건을 미리 축소(`PUSH_SUBQ`)
   - 작은 prod_id 집합으로 먼저 최적화
3) Hash Anti + Bloom Filter 가능
   - 사실 테이블 스캔량 급감

---

### 플랜 판독 체크리스트

- Anti 변환 성공
  - `HASH JOIN ANTI` / `NL ANTI` / `MERGE ANTI`
- 집계 제거 성공
  - `SCALAR SUBQUERY` 라인 없음
  - 사전 집계(`HASH GROUP BY`) + 조인
- Push 성공
  - `HASH JOIN SEMI/ANTI`
  - `FILTER` 없음
  - `JOIN FILTER CREATE/USE` 존재 시 최상

---

## 운영 튜닝 체크리스트(실무)

- [ ] **부재 판정은 `NOT EXISTS`**로 작성한다. (`NOT IN`은 NULL 위험)
- [ ] `COUNT(*)=0` / `COUNT(*)>0` 패턴은 **Anti/Semi Join으로 변환**한다.
- [ ] SELECT-list/WHERE 상관 집계는 **사전 집계 + 조인**으로 바꾼다.
- [ ] 서브쿼리 결과가 작아질 수 있으면 `PUSH_SUBQ(+UNNEST)`로 **먼저 축소 후 조인**한다.
- [ ] 플랜에 `FILTER`가 남아있다면 “변환 실패”를 의심하고 재작성/힌트를 검토한다.
- [ ] 최종 검증은 **DBMS_XPLAN 실측(ALLSTATS LAST)**과 `A-Rows`, 세션 통계로 한다.

---

## 부록. 미니 실습 묶음

```sql
-- A. Hash Anti vs NL Anti
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

-- B. 상관 SUM 제거
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

-- C. PUSH_SUBQ
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

## 맺음말

- **Anti Join**은 “없음”을 증명하는 최단 경로다. `NOT EXISTS`를 쓰면 의미가 안전하고 CBO 변환도 안정적이다.
- **Aggregate Subquery Elimination**은 상관 집계의 **행단위 반복 호출을 제거**해 “한 번의 집계 + 조인”으로 끝낸다.
- **Subquery Pushing**은 작은 키 집합을 **먼저 만들고**, 그걸 기반으로 큰 테이블을 **세미/안티/일반 조인**으로 빠르게 탐색하게 한다.
- 최종 정답은 항상 **실측 계획과 통계**다. 플랜에서 `FILTER`가 사라지고 `SEMI/ANTI + 사전집계`가 보이면, 튜닝은 성공한 것이다.
