---
layout: post
title: DB 심화 - 서브쿼리 Unnesting (2)
date: 2025-11-18 17:25:23 +0900
category: DB 심화
---
# 서브쿼리 언네스팅 심층 분석: FILTER vs SEMI JOIN의 본질적 차이

## 서브쿼리 최적화의 핵심 도전

데이터베이스의 가장 일반적인 관계 패턴은 1:M(일대다) 관계입니다. 고객과 주문, 제품과 판매 기록, 부서와 직원과 같은 관계에서 "다(M)" 측 테이블은 동일한 키 값이 반복되는 구조를 가집니다. 이러한 환경에서 EXISTS나 IN을 사용한 서브쿼리의 성능 최적화는 데이터베이스 엔진에게 특별한 도전을 제기합니다.

## 실습 환경 구성

```sql
-- 환경 설정
ALTER SESSION SET nls_date_format = 'YYYY-MM-DD';
ALTER SESSION SET statistics_level = ALL;

-- 차원 테이블: 고객 정보
CREATE TABLE customers (
    customer_id NUMBER PRIMARY KEY,
    region      VARCHAR2(20) NOT NULL,
    tier        VARCHAR2(20) NOT NULL,
    join_date   DATE NOT NULL
);

-- 팩트 테이블: 판매 기록 (M측)
CREATE TABLE sales (
    sale_id     NUMBER PRIMARY KEY,
    customer_id NUMBER NOT NULL,
    product_id  NUMBER NOT NULL,
    sale_date   DATE NOT NULL,
    quantity    NUMBER NOT NULL,
    amount      NUMBER(10,2) NOT NULL
);

-- 비고유 인덱스 생성 (실제 M측 테이블의 전형적인 구조)
CREATE INDEX idx_sales_customer_date ON sales(customer_id, sale_date);
CREATE INDEX idx_sales_customer ON sales(customer_id);

-- 샘플 데이터 생성
DECLARE
    v_customer_count CONSTANT NUMBER := 10000;
    v_sale_count     CONSTANT NUMBER := 500000;
    v_date_range     CONSTANT NUMBER := 365;
BEGIN
    -- 고객 데이터 생성
    FOR i IN 1..v_customer_count LOOP
        INSERT INTO customers VALUES (
            i,
            CASE MOD(i, 8)
                WHEN 0 THEN '서울' WHEN 1 THEN '경기' 
                WHEN 2 THEN '부산' WHEN 3 THEN '대구'
                WHEN 4 THEN '인천' WHEN 5 THEN '광주'
                WHEN 6 THEN '대전' ELSE '울산'
            END,
            CASE MOD(i, 100)
                WHEN 0 THEN 'VIP'
                WHEN 1 THEN 'GOLD'
                WHEN 2 THEN 'SILVER'
                ELSE 'STANDARD'
            END,
            DATE '2020-01-01' + MOD(i, 1000)
        );
    END LOOP;
    
    -- 판매 데이터 생성 (M측: 한 고객당 여러 판매 기록)
    FOR i IN 1..v_sale_count LOOP
        INSERT INTO sales VALUES (
            i,
            MOD(i, v_customer_count) + 1,  -- 고객 ID (반복됨)
            MOD(i, 5000) + 1,              -- 제품 ID
            DATE '2024-01-01' + MOD(i, v_date_range),  -- 판매일
            1 + MOD(i, 10),                -- 수량
            ROUND(DBMS_RANDOM.VALUE(1000, 100000), 2)  -- 금액
        );
    END LOOP;
    
    COMMIT;
    DBMS_OUTPUT.PUT_LINE('데이터 생성 완료: 고객 ' || v_customer_count || '명, 판매 ' || v_sale_count || '건');
END;
/

-- 통계 수집
BEGIN
    DBMS_STATS.GATHER_TABLE_STATS(USER, 'CUSTOMERS', cascade => TRUE);
    DBMS_STATS.GATHER_TABLE_STATS(
        USER, 'SALES', 
        estimate_percent => DBMS_STATS.AUTO_SAMPLE_SIZE,
        method_opt => 'FOR ALL COLUMNS SIZE AUTO',
        cascade => TRUE
    );
END;
/
```

## FILTER 연산의 작동 메커니즘과 한계

### FILTER의 기본 동작 방식

FILTER 연산은 서브쿼리를 조인으로 변환하지 않고, 외부 쿼리의 각 행에 대해 서브쿼리를 개별적으로 평가합니다. 이는 상관 서브쿼리가 "행 단위"로 처리되는 전통적인 방식을 따릅니다.

```sql
-- FILTER 연산이 사용되는 전형적인 패턴
EXPLAIN PLAN FOR
SELECT /*+ NO_UNNEST */ 
       c.customer_id,
       c.region
FROM customers c
WHERE EXISTS (
    SELECT 1
    FROM sales s
    WHERE s.customer_id = c.customer_id
    AND s.sale_date BETWEEN DATE '2024-03-01' AND DATE '2024-03-31'
);

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);
```

**FILTER 연산의 실행 계획 예시:**
```
OPERATION           | OPTIONS       | OBJECT_NAME
-------------------|---------------|-------------
SELECT STATEMENT   |               |
 FILTER            |               |
  TABLE ACCESS     | FULL          | CUSTOMERS
  INDEX            | RANGE SCAN    | IDX_SALES_CUSTOMER_DATE
   TABLE ACCESS    | BY INDEX ROWID| SALES
```

### 서브쿼리 캐싱의 제한적 효과

Oracle은 FILTER 연산에서 일정 수준의 캐싱을 수행하지만, 이는 제한적인 효율성만 제공합니다:

```sql
-- 서브쿼리 캐싱 효과 관찰을 위한 쿼리
WITH customer_sample AS (
    SELECT customer_id FROM customers 
    WHERE region = '서울'
    AND ROWNUM <= 1000
)
SELECT /*+ NO_UNNEST */ 
       cs.customer_id
FROM customer_sample cs
WHERE EXISTS (
    SELECT 1
    FROM sales s
    WHERE s.customer_id = cs.customer_id
    AND s.sale_date >= DATE '2024-03-01'
);

-- 캐시 히트율이 낮은 경우의 성능 문제
SELECT /*+ NO_UNNEST */ 
       c.customer_id
FROM customers c
WHERE EXISTS (
    SELECT 1
    FROM sales s
    WHERE s.customer_id = c.customer_id
    AND s.amount > 50000
);
```

**FILTER의 주요 제한사항:**
1. **캐시 크기 제한**: 서브쿼리 캐시는 제한된 크기를 가지며, 많은 고유 키 값이 있는 경우 효율이 급격히 떨어집니다.
2. **비고유 인덱스 오버헤드**: M측 테이블의 비고유 인덱스는 동일 키 값에 대한 여러 엔트리를 가지며, FILTER는 매번 첫 번째 매칭을 찾기 위해 인덱스 체인을 탐색해야 합니다.
3. **반복적 평가**: 외부 테이블의 각 행에 대해 서브쿼리를 재평가하므로, 대량 데이터 처리 시 비효율적입니다.

## SEMI JOIN: 언네스팅의 성능 혁신

### SEMI JOIN의 기본 개념

SEMI JOIN은 서브쿼리를 조인으로 변환하여, "존재 여부"를 집합 단위로 효율적으로 확인합니다. 이 변환은 서브쿼리 언네스팅(Subquery Unnesting)의 핵심 기법입니다.

```sql
-- SEMI JOIN으로 변환된 쿼리
EXPLAIN PLAN FOR
SELECT /*+ UNNEST */ 
       c.customer_id,
       c.region
FROM customers c
WHERE EXISTS (
    SELECT 1
    FROM sales s
    WHERE s.customer_id = c.customer_id
    AND s.sale_date BETWEEN DATE '2024-03-01' AND DATE '2024-03-31'
);

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);
```

**SEMI JOIN의 실행 계획 예시:**
```
OPERATION           | OPTIONS       | OBJECT_NAME
-------------------|---------------|-------------
SELECT STATEMENT   |               |
 HASH JOIN         | SEMI          |
  TABLE ACCESS     | FULL          | CUSTOMERS
  TABLE ACCESS     | FULL          | SALES
   INDEX           | RANGE SCAN    | IDX_SALES_CUSTOMER_DATE
```

### NL SEMI JOIN: 조기 종료 최적화

Nested Loops SEMI JOIN은 첫 번째 매칭을 발견하면 즉시 다음 외부 행으로 이동하는 "조기 종료(Early-out)" 메커니즘을 사용합니다.

```sql
-- NL SEMI JOIN 유도
SELECT /*+ UNNEST USE_NL(s) LEADING(c s) */
       c.customer_id,
       c.tier
FROM customers c
WHERE EXISTS (
    SELECT 1
    FROM sales s
    WHERE s.customer_id = c.customer_id
    AND s.sale_date >= DATE '2024-01-01'
    AND s.quantity > 5
)
AND c.region = '서울';
```

**NL SEMI JOIN의 장점:**
- **조기 종료**: 첫 매칭 발견 시 즉시 중단하여 불필요한 검색 제거
- **인덱스 효율**: 비고유 인덱스에서도 첫 매칭만 찾으면 되므로 효율적
- **소량 데이터에 적합**: 외부 테이블이 작을 때 유리

### HASH SEMI JOIN: 집합 기반 최적화

Hash SEMI JOIN은 내부 테이블의 고유 키 값을 해시 테이블에 저장하여 멤버십 테스트를 수행합니다.

```sql
-- HASH SEMI JOIN 유도
SELECT /*+ UNNEST USE_HASH(s) LEADING(s c) */
       c.customer_id,
       c.region
FROM customers c
WHERE EXISTS (
    SELECT 1
    FROM sales s
    WHERE s.customer_id = c.customer_id
    AND s.sale_date BETWEEN DATE '2024-01-01' AND DATE '2024-03-31'
    AND s.amount > 100000
);
```

**HASH SEMI JOIN의 고급 기능:**

```sql
-- JOIN FILTER(Bloom Filter) 활용
SELECT /*+ UNNEST USE_HASH(s) OPT_PARAM('_bloom_filter_enabled', 'true') */
       c.customer_id,
       COUNT(*) as sales_count
FROM customers c
WHERE EXISTS (
    SELECT 1
    FROM sales s
    WHERE s.customer_id = c.customer_id
    AND s.sale_date >= DATE '2024-01-01'
)
GROUP BY c.customer_id
HAVING COUNT(*) > 10;
```

**HASH SEMI JOIN의 장점:**
1. **해시 캐시**: 내부 테이블의 키 집합을 해시 테이블에 캐싱
2. **상수 시간 검색**: 멤버십 테스트가 O(1) 시간 복잡도
3. **JOIN FILTER**: Bloom Filter를 사용한 조기 필터링으로 I/O 감소
4. **대량 데이터 적합**: 외부 테이블이 클 때 효율적

## 성능 비교 실험

### 실험 1: FILTER vs SEMI JOIN 직접 비교

```sql
-- 성능 측정을 위한 준비
SET TIMING ON
SET AUTOTRACE TRACEONLY STATISTICS

-- 테스트 1: FILTER 연산 (언네스팅 금지)
SELECT /*+ NO_UNNEST GATHER_PLAN_STATISTICS */ 
       COUNT(*) as customer_count
FROM customers c
WHERE EXISTS (
    SELECT 1
    FROM sales s
    WHERE s.customer_id = c.customer_id
    AND s.sale_date BETWEEN DATE '2024-03-01' AND DATE '2024-03-31'
    AND s.amount > 50000
);

-- 테스트 2: SEMI JOIN (언네스팅 허용)
SELECT /*+ UNNEST GATHER_PLAN_STATISTICS */ 
       COUNT(*) as customer_count
FROM customers c
WHERE EXISTS (
    SELECT 1
    FROM sales s
    WHERE s.customer_id = c.customer_id
    AND s.sale_date BETWEEN DATE '2024-03-01' AND DATE '2024-03-31'
    AND s.amount > 50000
);

-- 통계 비교
SELECT 
    'FILTER' as operation_type,
    sql_id,
    executions,
    elapsed_time,
    buffer_gets,
    disk_reads,
    cpu_time
FROM v$sql
WHERE sql_text LIKE '%NO_UNNEST%'
AND sql_text LIKE '%customer_count%'
UNION ALL
SELECT 
    'SEMI_JOIN' as operation_type,
    sql_id,
    executions,
    elapsed_time,
    buffer_gets,
    disk_reads,
    cpu_time
FROM v$sql
WHERE sql_text LIKE '%UNNEST%'
AND sql_text NOT LIKE '%NO_UNNEST%'
AND sql_text LIKE '%customer_count%';
```

### 실험 2: 다양한 시나리오별 성능 분석

```sql
-- 시나리오 A: 고선택도 조건 (소수 고객만 매칭)
SELECT /*+ UNNEST USE_NL(s) LEADING(c s) */
       c.customer_id,
       c.tier,
       (SELECT MAX(s.amount) 
        FROM sales s 
        WHERE s.customer_id = c.customer_id) as max_amount
FROM customers c
WHERE c.region = '서울'
AND c.tier = 'VIP'
AND EXISTS (
    SELECT 1
    FROM sales s
    WHERE s.customer_id = c.customer_id
    AND s.amount > 900000  -- 매우 높은 금액
);

-- 시나리오 B: 저선택도 조건 (많은 고객 매칭)
SELECT /*+ UNNEST USE_HASH(s) LEADING(s c) */
       c.region,
       COUNT(DISTINCT c.customer_id) as active_customers
FROM customers c
WHERE EXISTS (
    SELECT 1
    FROM sales s
    WHERE s.customer_id = c.customer_id
    AND s.sale_date >= DATE '2024-01-01'  -- 광범위한 기간
)
GROUP BY c.region
ORDER BY active_customers DESC;

-- 시나리오 C: 복합 조건
SELECT /*+ UNNEST */ 
       c.customer_id,
       c.region,
       c.tier
FROM customers c
WHERE EXISTS (
    SELECT 1
    FROM sales s
    WHERE s.customer_id = c.customer_id
    AND s.sale_date >= DATE '2024-01-01'
    AND (s.amount > 50000 OR s.quantity > 10)
)
AND c.join_date >= DATE '2023-01-01';
```

## 스칼라 서브쿼리의 최적화 전략

### 스칼라 서브쿼리의 캐싱 메커니즘

```sql
-- 전통적 스칼라 서브쿼리 (행 단위 평가)
SELECT 
    c.customer_id,
    c.region,
    (SELECT SUM(s.amount)
     FROM sales s
     WHERE s.customer_id = c.customer_id
     AND s.sale_date BETWEEN DATE '2024-01-01' AND DATE '2024-03-31') as q1_sales,
    (SELECT COUNT(*)
     FROM sales s
     WHERE s.customer_id = c.customer_id
     AND s.sale_date BETWEEN DATE '2024-01-01' AND DATE '2024-03-31') as q1_count
FROM customers c
WHERE c.region = '서울';

-- 최적화된 버전: 사전 집계 + 조인
WITH sales_agg AS (
    SELECT 
        customer_id,
        SUM(amount) as total_sales,
        COUNT(*) as sale_count
    FROM sales
    WHERE sale_date BETWEEN DATE '2024-01-01' AND DATE '2024-03-31'
    GROUP BY customer_id
)
SELECT 
    c.customer_id,
    c.region,
    COALESCE(sa.total_sales, 0) as q1_sales,
    COALESCE(sa.sale_count, 0) as q1_count
FROM customers c
LEFT JOIN sales_agg sa ON sa.customer_id = c.customer_id
WHERE c.region = '서울';
```

### 스칼라 서브쿼리 캐시의 효율성 분석

```sql
-- 캐시 히트율 실험
DECLARE
    TYPE number_array IS TABLE OF NUMBER;
    v_customer_ids number_array;
    v_hit_count NUMBER := 0;
    v_total_count NUMBER := 0;
    v_start_time NUMBER;
    v_end_time NUMBER;
BEGIN
    -- 고객 ID 샘플링 (반복 패턴 생성)
    SELECT customer_id 
    BULK COLLECT INTO v_customer_ids
    FROM (
        SELECT customer_id FROM customers 
        WHERE region = '서울'
        ORDER BY DBMS_RANDOM.VALUE
    ) WHERE ROWNUM <= 1000;
    
    -- 같은 ID를 반복하여 캐시 효과 극대화
    v_start_time := DBMS_UTILITY.GET_TIME;
    
    FOR i IN 1..10 LOOP  -- 10번 반복
        FOR j IN 1..v_customer_ids.COUNT LOOP
            FOR rec IN (
                SELECT SUM(amount) as total
                FROM sales
                WHERE customer_id = v_customer_ids(j)
                AND sale_date >= DATE '2024-01-01'
            ) LOOP
                v_total_count := v_total_count + 1;
                -- 캐시 히트 카운팅 로직 (개념적)
                IF j <= 100 THEN  -- 처음 100개 ID는 반복적으로 사용
                    v_hit_count := v_hit_count + 1;
                END IF;
            END LOOP;
        END LOOP;
    END LOOP;
    
    v_end_time := DBMS_UTILITY.GET_TIME;
    
    DBMS_OUTPUT.PUT_LINE('실행 시간: ' || (v_end_time - v_start_time)/100 || '초');
    DBMS_OUTPUT.PUT_LINE('캐시 히트율(추정): ' || 
                         ROUND(v_hit_count/NULLIF(v_total_count, 0) * 100, 2) || '%');
END;
/
```

## 실무 적용 가이드라인

### 상황별 최적 전략 선택

```sql
-- 1. 소량 외부 테이블 + 고선택도 조건: NL SEMI JOIN
SELECT /*+ UNNEST USE_NL(s) LEADING(c s) INDEX(s IDX_SALES_CUSTOMER_DATE) */
       c.customer_id,
       c.tier
FROM customers c
WHERE c.region = '서울'
AND c.tier = 'VIP'
AND EXISTS (
    SELECT 1
    FROM sales s
    WHERE s.customer_id = c.customer_id
    AND s.sale_date >= DATE '2024-03-01'
    AND s.amount > 800000
);

-- 2. 대량 외부 테이블 + 저선택도 조건: HASH SEMI JOIN
SELECT /*+ UNNEST USE_HASH(s) LEADING(s c) FULL(s) */
       c.region,
       COUNT(*) as customer_count
FROM customers c
WHERE EXISTS (
    SELECT 1
    FROM sales s
    WHERE s.customer_id = c.customer_id
    AND s.sale_date >= DATE '2024-01-01'
)
GROUP BY c.region;

-- 3. 복잡한 비즈니스 로직: 서브쿼리 팩토링
WITH recent_customers AS (
    SELECT DISTINCT customer_id
    FROM sales
    WHERE sale_date >= DATE '2024-01-01'
    GROUP BY customer_id
    HAVING SUM(amount) > 1000000
),
high_value_sales AS (
    SELECT customer_id, SUM(amount) as total_amount
    FROM sales
    WHERE sale_date >= DATE '2024-03-01'
    GROUP BY customer_id
)
SELECT 
    c.customer_id,
    c.region,
    c.tier,
    COALESCE(hvs.total_amount, 0) as recent_sales
FROM customers c
INNER JOIN recent_customers rc ON rc.customer_id = c.customer_id
LEFT JOIN high_value_sales hvs ON hvs.customer_id = c.customer_id
WHERE c.join_date >= DATE '2023-01-01';
```

### 성능 모니터링과 튜닝 프레임워크

```sql
-- 서브쿼리 성능 모니터링 대시보드
SELECT 
    sql_id,
    plan_hash_value,
    executions,
    ROUND(elapsed_time/1000000, 2) as elapsed_sec,
    buffer_gets,
    disk_reads,
    rows_processed,
    SUBSTR(sql_text, 1, 100) as sql_snippet,
    CASE 
        WHEN INSTR(sql_text, 'NO_UNNEST') > 0 THEN 'FILTER'
        WHEN INSTR(sql_text, 'UNNEST') > 0 THEN 'SEMI_JOIN'
        ELSE 'AUTO'
    END as subquery_type
FROM v$sql
WHERE sql_text LIKE '%EXISTS%'
   OR sql_text LIKE '%IN (SELECT%'
   OR sql_text LIKE '%SELECT.*SELECT%'
AND executions > 0
AND last_active_time > SYSDATE - 1
ORDER BY elapsed_time DESC
FETCH FIRST 20 ROWS ONLY;

-- 실행 계획 상세 분석
SELECT 
    operation,
    options,
    object_name,
    cardinality,
    bytes,
    cost,
    access_predicates,
    filter_predicates,
    time
FROM v$sql_plan
WHERE sql_id = :target_sql_id
AND (operation LIKE '%JOIN%' OR operation = 'FILTER')
ORDER BY id;
```

## 고급 최적화 기법

### 파티션 프루닝과의 연동

```sql
-- 월별 파티션된 sales 테이블 가정
SELECT /*+ UNNEST USE_HASH(s) */
       c.customer_id,
       c.region,
       COUNT(*) as monthly_sales_count
FROM customers c
WHERE EXISTS (
    SELECT 1
    FROM sales_partitioned s
    WHERE s.customer_id = c.customer_id
    AND s.sale_date BETWEEN DATE '2024-01-01' AND DATE '2024-03-31'
    AND s.amount > 50000
)
GROUP BY c.customer_id, c.region
HAVING COUNT(*) >= 3;

-- 파티션 프루닝 확인
EXPLAIN PLAN FOR
SELECT /*+ UNNEST */ 
       c.customer_id
FROM customers c
WHERE EXISTS (
    SELECT 1
    FROM sales_partitioned s
    WHERE s.customer_id = c.customer_id
    AND s.sale_date BETWEEN DATE '2024-02-01' AND DATE '2024-02-28'
);

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);
```

### 적응형 쿼리 최적화

```sql
-- 적응형 조인 방법 활용
SELECT /*+ UNNEST ADAPTIVE */ 
       c.customer_id,
       c.tier,
       (SELECT MAX(s.amount) 
        FROM sales s 
        WHERE s.customer_id = c.customer_id
        AND s.sale_date >= DATE '2024-01-01') as max_sale_2024
FROM customers c
WHERE c.region = '서울'
AND EXISTS (
    SELECT 1
    FROM sales s
    WHERE s.customer_id = c.customer_id
    AND s.sale_date >= DATE '2024-03-01'
);

-- 동적 샘플링과의 조합
SELECT /*+ UNNEST DYNAMIC_SAMPLING(4) */ 
       c.customer_id,
       c.region
FROM customers c
WHERE EXISTS (
    SELECT 1
    FROM sales s
    WHERE s.customer_id = c.customer_id
    AND s.sale_date >= ADD_MONTHS(SYSDATE, -6)
);
```

## 결론: 현명한 서브쿼리 최적화 전략

서브쿼리 언네스팅은 Oracle 데이터베이스의 핵심 최적화 기술 중 하나로, FILTER 연산의 행 단위 처리 한계를 극복하고 집합 기반의 효율적인 처리로 전환합니다. 이 기술의 성공적 적용을 위한 핵심 원칙은 다음과 같습니다:

### 1. 패턴 인식과 전략 선택
- **소량-고선택도**: NL SEMI JOIN으로 조기 종료 이점 활용
- **대량-저선택도**: HASH SEMI JOIN으로 해시 캐싱 효과 극대화
- **복잡한 비즈니스 로직**: 서브쿼리 팩토링과 CTE 활용

### 2. 데이터 특성 이해
- M측 테이블의 비고유 인덱스 특성을 고려한 접근법 선택
- 데이터 분포 편향과 카디널리티에 따른 최적화 전략 수립
- 파티션 구조를 활용한 프루닝 효과 확보

### 3. 측정 기반 결정
- 추상적 이론보다 실제 실행 통계에 의존
- FILTER vs SEMI JOIN의 실제 성능 차이 측정
- 다양한 시나리오에서의 성능 패턴 분석

### 4. 점진적 최적화
- 자동 언네스팅에 의존하면서도 필요 시 힌트로 세부 조정
- 성능 모니터링을 통한 지속적 개선
- 비즈니스 요구사항과 성능 목표의 균형 유지

서브쿼리 최적화는 단순한 기술 적용이 아니라 데이터 패턴, 쿼리 특성, 시스템 환경을 종합적으로 이해하는 과정입니다. FILTER와 SEMI JOIN의 근본적 차이를 이해하고, 상황에 맞는 최적의 전략을 선택할 때, 복잡한 비즈니스 요구사항을 효율적으로 처리하는 강력한 데이터베이스 시스템을 구축할 수 있습니다.

최종적으로 모든 최적화 결정은 실제 성능 측정으로 검증되어야 합니다. 실행 계획 분석, 자원 사용량 모니터링, 응답 시간 측정을 통해 데이터 기반의 튜닝 결정을 내리는 것이 장기적인 성능 안정성을 보장하는 핵심입니다.