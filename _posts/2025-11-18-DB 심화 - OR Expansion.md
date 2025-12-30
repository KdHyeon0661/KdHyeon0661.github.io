---
layout: post
title: DB 심화 - OR Expansion
date: 2025-11-18 23:25:23 +0900
category: DB 심화
---
# OR Expansion: OR 조건의 족쇄를 풀고 최적화의 자유를 되찾는 기술

## OR 조건의 근본적인 문제: 옵티마이저의 딜레마

Oracle 옵티마이저(CBO)가 SQL 실행 계획을 수립할 때, OR 조건은 특별한 도전을 제기합니다. 기본적으로 옵티마이저는 하나의 쿼리 블록에 대해 단일한 조인 순서, 단일한 접근 경로, 단일한 조인 방법을 선택해야 합니다. 그러나 OR 조건은 종종 서로 상충되는 최적화 요구사항을 동시에 제시합니다.

**실제 시나리오에서의 OR 문제:**

```sql
SELECT *
FROM sales s
JOIN customers c ON c.customer_id = s.customer_id
WHERE (c.region = '서울' AND c.membership_tier = 'VIP')
   OR (s.sales_date >= DATE '2024-01-01' AND s.amount > 1000000);
```

이 쿼리에서:
- 첫 번째 조건: 소수의 VIP 고객을 대상으로 하는 고선택적 필터
- 두 번째 조건: 특정 기간 이후의 고액 거래를 대상으로 하는 광범위한 필터

이 두 조건은 서로 다른 인덱스, 서로 다른 조인 순서, 심지어 서로 다른 조인 방법을 요구할 수 있습니다. OR을 하나의 블록으로 처리하면 옵티마이저는 "두 조건을 모두 만족시키는 최소 공통 분모"를 선택하게 되어, 종종 비효율적인 전체 테이블 스캔이나 적절하지 않은 인덱스 사용으로 이어집니다.

## OR Expansion의 개념: 분할 정복 접근법

OR Expansion은 이러한 딜레마를 해결하기 위해 OR 조건을 가진 쿼리 블록을 여러 개의 독립적인 쿼리 블록으로 분해하는 기술입니다. 이 변환은 비용 기반 쿼리 변환(Cost-based Query Transformation)의 일종으로, Oracle 12cR2 이후 더욱 정교해졌습니다.

**OR Expansion의 기본 아이디어:**

```sql
-- 원본 쿼리
SELECT *
FROM table1 t1
JOIN table2 t2 ON t2.id = t1.id
WHERE (condition_A) OR (condition_B);

-- OR Expansion 변환 후
SELECT * FROM table1 t1 JOIN table2 t2 ON t2.id = t1.id WHERE condition_A
UNION ALL
SELECT * FROM table1 t1 JOIN table2 t2 ON t2.id = t1.id WHERE condition_B
AND NOT (condition_A); -- 중복 방지
```

각 분기(브랜치)는 이제 독립적인 쿼리 블록이 되어, 자신에게 최적화된 실행 계획을 가질 수 있습니다. 이는 데이터베이스 최적화의 "분할 정복(divide and conquer)" 전략에 해당합니다.

## OR Expansion의 발전: LORE에서 ORE까지

### LORE (Legacy OR-Expansion)
과거의 OR Expansion 방식은 "Concatenation"으로 알려져 있으며, 주로 모든 분기에서 인덱스 접근이 가능한 경우에 잘 작동했습니다. 이 방식은 `USE_CONCAT` 힌트로 유도할 수 있으며, 옵티마이저 트레이스에서 "CONCATENATION" 또는 "LORE"로 표시됩니다.

### ORE (Cost-based OR-Expansion)
Oracle 12cR2 이후 도입된 새로운 방식은 더욱 유연하고 비용 기반의 결정을 내립니다. ORE는 분기별로 서로 다른 조인 전략과 접근 경로를 허용하며, 더 복잡한 시나리오에서도 잘 작동합니다.

**버전 업그레이드 시 주의사항:**
11g에서 19c/23ai로 업그레이드하는 경우, 기존에 LORE로 잘 작동하던 쿼리가 새로운 ORE 알고리즘에서 다르게 처리될 수 있습니다. 때로는 자동 확장이 발생하지 않아 성능이 저하되는 경우도 있습니다. 이러한 경우 `USE_CONCAT` 힌트를 사용하여 레거시 방식을 강제할 수 있습니다.

## OR Expansion의 핵심 장점

### 1. 분기별 최적 인덱스 활용
OR 조건의 각 부분이 서로 다른 인덱스를 최적으로 사용할 수 있게 합니다.

```sql
-- OR Expansion 없음: 단일 인덱스 선택의 딜레마
SELECT /*+ NO_EXPAND */ *
FROM orders
WHERE (customer_id = 100) OR (order_date > DATE '2024-01-01');
-- 옵티마이저는 customer_id 인덱스나 order_date 인덱스 중 하나만 선택

-- OR Expansion 적용: 각 조건에 최적의 인덱스 사용
SELECT /*+ USE_CONCAT */ *
FROM orders
WHERE customer_id = 100
UNION ALL
SELECT *
FROM orders
WHERE order_date > DATE '2024-01-01'
AND customer_id != 100; -- 중복 제거
```

### 2. 파티션 프루닝 최적화
OR 조건이 파티션 키에 영향을 미칠 때, 확장을 통해 각 분기가 서로 다른 파티션 세트를 정확히 타겟팅할 수 있습니다.

```sql
-- 월별 파티션된 sales 테이블
SELECT /*+ USE_CONCAT */ 
       product_id, SUM(amount)
FROM sales
WHERE (sales_date BETWEEN '2024-01-01' AND '2024-01-31')
   OR (sales_date BETWEEN '2024-03-01' AND '2024-03-31')
GROUP BY product_id;

-- 확장 후 각 분기는 정확히 하나의 월 파티션만 접근
```

### 3. 조인 전략 다양화
각 분기가 서로 다른 드라이빙 테이블과 조인 방법을 사용할 수 있습니다.

```sql
-- 복잡한 조인 쿼리에서의 OR Expansion
SELECT /*+ USE_CONCAT */ 
       s.sales_id, c.customer_name, p.product_name
FROM sales s
JOIN customers c ON c.customer_id = s.customer_id
JOIN products p ON p.product_id = s.product_id
WHERE (c.region = '서울' AND c.membership = 'VIP')
   OR (p.category = '전자제품' AND s.amount > 500000);

-- 확장 후:
-- 첫 번째 분기: customers를 드라이빙 테이블로, Nested Loops 조인
-- 두 번째 분기: products를 드라이빙 테이블로, Hash 조인
```

## OR Expansion 제어 힌트

Oracle은 OR Expansion을 제어하기 위한 다양한 힌트를 제공합니다:

```sql
-- OR Expansion 강제
SELECT /*+ USE_CONCAT */ * FROM table WHERE condition_A OR condition_B;

-- 보다 공격적인 OR Expansion (12cR2+)
SELECT /*+ OR_EXPAND */ * FROM table WHERE condition_A OR condition_B;

-- 특정 쿼리 블록에서 OR Expansion 방지
SELECT /*+ NO_OR_EXPAND(@qb1) */ * FROM table WHERE condition_A OR condition_B;

-- 모든 형태의 확장 방지 (전통적 방식)
SELECT /*+ NO_EXPAND */ * FROM table WHERE condition_A OR condition_B;
```

## 실습 환경 구성

OR Expansion의 효과를 직접 확인하기 위한 실습 환경을 구성해 보겠습니다:

```sql
-- 세션 설정
ALTER SESSION SET nls_date_format = 'YYYY-MM-DD';
ALTER SESSION SET statistics_level = ALL;

-- 차원 테이블 생성
CREATE TABLE dim_customers (
    customer_id NUMBER PRIMARY KEY,
    region      VARCHAR2(20) NOT NULL,
    membership_tier VARCHAR2(20) NOT NULL,
    signup_date DATE NOT NULL
);

CREATE TABLE dim_products (
    product_id  NUMBER PRIMARY KEY,
    category    VARCHAR2(30) NOT NULL,
    brand       VARCHAR2(30) NOT NULL,
    price_range VARCHAR2(20) NOT NULL
);

-- 팩트 테이블 생성 (월별 파티션)
CREATE TABLE fact_sales (
    sale_id     NUMBER,
    customer_id NUMBER NOT NULL,
    product_id  NUMBER NOT NULL,
    sale_date   DATE NOT NULL,
    quantity    NUMBER NOT NULL,
    amount      NUMBER(12,2) NOT NULL
)
PARTITION BY RANGE (sale_date) (
    PARTITION sales_2024_q1 VALUES LESS THAN (DATE '2024-04-01'),
    PARTITION sales_2024_q2 VALUES LESS THAN (DATE '2024-07-01'),
    PARTITION sales_2024_q3 VALUES LESS THAN (DATE '2024-10-01'),
    PARTITION sales_2024_q4 VALUES LESS THAN (DATE '2025-01-01'),
    PARTITION sales_future  VALUES LESS THAN (MAXVALUE)
);

-- 인덱스 생성
CREATE INDEX idx_sales_customer_date ON fact_sales(customer_id, sale_date);
CREATE INDEX idx_sales_product_date ON fact_sales(product_id, sale_date);
CREATE INDEX idx_sales_date ON fact_sales(sale_date);
CREATE INDEX idx_customers_region_tier ON dim_customers(region, membership_tier);
CREATE INDEX idx_products_category ON dim_products(category, product_id);

-- 샘플 데이터 삽입
DECLARE
    v_customer_id NUMBER;
    v_product_id  NUMBER;
    v_sale_date   DATE;
BEGIN
    -- 10,000명의 고객 데이터
    FOR i IN 1..10000 LOOP
        INSERT INTO dim_customers VALUES (
            i,
            CASE MOD(i, 5) 
                WHEN 0 THEN '서울' WHEN 1 THEN '경기' 
                WHEN 2 THEN '부산' WHEN 3 THEN '대구' ELSE '인천' 
            END,
            CASE MOD(i, 10)
                WHEN 0 THEN 'VIP' WHEN 1 THEN 'GOLD'
                WHEN 2 THEN 'SILVER' WHEN 3 THEN 'BRONZE' ELSE 'STANDARD'
            END,
            DATE '2020-01-01' + MOD(i, 1460)
        );
    END LOOP;
    
    -- 5,000개의 제품 데이터
    FOR i IN 1..5000 LOOP
        INSERT INTO dim_products VALUES (
            i,
            CASE MOD(i, 8)
                WHEN 0 THEN '전자제품' WHEN 1 THEN '의류'
                WHEN 2 THEN '식품' WHEN 3 THEN '가구'
                WHEN 4 THEN '서적' WHEN 5 THEN '스포츠용품'
                WHEN 6 THEN '화장품' ELSE '기타'
            END,
            '브랜드_' || TO_CHAR(MOD(i, 50) + 1),
            CASE 
                WHEN MOD(i, 100) < 10 THEN '고가'
                WHEN MOD(i, 100) < 40 THEN '중가'
                ELSE '저가'
            END
        );
    END LOOP;
    
    -- 100,000건의 판매 데이터
    FOR i IN 1..100000 LOOP
        v_customer_id := MOD(i, 10000) + 1;
        v_product_id  := MOD(i, 5000) + 1;
        v_sale_date   := DATE '2024-01-01' + MOD(i, 365);
        
        INSERT INTO fact_sales VALUES (
            i,
            v_customer_id,
            v_product_id,
            v_sale_date,
            1 + MOD(i, 10),
            ROUND(DBMS_RANDOM.VALUE(1000, 1000000), 2)
        );
    END LOOP;
    
    COMMIT;
    DBMS_OUTPUT.PUT_LINE('샘플 데이터 생성 완료');
END;
/

-- 통계 수집
BEGIN
    DBMS_STATS.GATHER_TABLE_STATS(USER, 'DIM_CUSTOMERS', cascade => TRUE);
    DBMS_STATS.GATHER_TABLE_STATS(USER, 'DIM_PRODUCTS', cascade => TRUE);
    DBMS_STATS.GATHER_TABLE_STATS(
        USER, 'FACT_SALES', 
        estimate_percent => DBMS_STATS.AUTO_SAMPLE_SIZE,
        method_opt => 'FOR ALL COLUMNS SIZE AUTO',
        cascade => TRUE
    );
END;
/
```

## 실전 예제: OR Expansion의 실제 효과

### 예제 1: 서로 다른 인덱스 활용 최적화

```sql
-- OR Expansion 없이
EXPLAIN PLAN FOR
SELECT /*+ NO_EXPAND */ 
       s.sale_id, s.amount, c.customer_name
FROM fact_sales s
JOIN dim_customers c ON c.customer_id = s.customer_id
WHERE (c.region = '서울' AND c.membership_tier = 'VIP')
   OR (s.sale_date BETWEEN DATE '2024-01-01' AND DATE '2024-01-31');

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);

-- OR Expansion 적용
EXPLAIN PLAN FOR
SELECT /*+ USE_CONCAT */ 
       s.sale_id, s.amount, c.customer_name
FROM fact_sales s
JOIN dim_customers c ON c.customer_id = s.customer_id
WHERE (c.region = '서울' AND c.membership_tier = 'VIP')
   OR (s.sale_date BETWEEN DATE '2024-01-01' AND DATE '2024-01-31');

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);
```

**관찰 포인트:**
- 첫 번째 쿼리: 단일 실행 계획으로 두 조건을 모두 만족시켜야 함
- 두 번째 쿼리: 두 개의 분기로 나뉘어 각각 최적의 인덱스 사용
- 각 분기의 `Pstart`/`Pstop` 값 확인 (파티션 프루닝 차이)

### 예제 2: 중복 제거를 위한 상호배타적 조건

```sql
-- OR Expansion with mutual exclusion
SELECT /*+ USE_CONCAT */ 
       SUM(s.amount) as total_sales
FROM fact_sales s
JOIN dim_customers c ON c.customer_id = s.customer_id
JOIN dim_products p ON p.product_id = s.product_id
WHERE (c.membership_tier = 'VIP' AND p.category = '전자제품')
   OR (s.amount > 500000 AND p.price_range = '고가')
ORDER BY total_sales DESC;

-- 확장 및 중복 제거 적용
SELECT /*+ USE_CONCAT */ 
       SUM(s.amount) as total_sales
FROM fact_sales s
JOIN dim_customers c ON c.customer_id = s.customer_id
JOIN dim_products p ON p.product_id = s.product_id
WHERE c.membership_tier = 'VIP' 
  AND p.category = '전자제품'

UNION ALL

SELECT SUM(s.amount)
FROM fact_sales s
JOIN dim_customers c ON c.customer_id = s.customer_id
JOIN dim_products p ON p.product_id = s.product_id
WHERE s.amount > 500000 
  AND p.price_range = '고가'
  AND NOT (c.membership_tier = 'VIP' AND p.category = '전자제품');
```

## OR Expansion의 제약 조건과 주의사항

### 자동 확장이 발생하지 않는 경우

1. **의미 보존 불가능**: 변환 후 쿼리 의미가 변경되는 경우
2. **비용 이점 없음**: 확장 오버헤드가 이점보다 큰 경우
3. **기술적 제약**: 복잡한 중첩 구조, 윈도우 함수, 외부 조인 포함 시
4. **통계 부정확**: 분기별 비용 추정이 불안정한 경우

### 분기 수 증가 문제

```sql
-- 분기 수 폭발 예시 (비추천)
SELECT /*+ USE_CONCAT */ *
FROM orders
WHERE status IN ('A','B','C','D','E','F','G','H','I','J');

-- 10개 값 -> 10개 분기 생성
-- 대안: IN-LIST iterator 사용
SELECT /*+ NO_EXPAND */ *
FROM orders
WHERE status IN ('A','B','C','D','E','F','G','H','I','J');
```

너무 많은 분기는 파싱 오버헤드, 실행 계획 캐시 부담, UNION ALL 병합 비용을 증가시킵니다.

## IN-LIST Iterator vs OR Expansion vs Bitmap OR

Oracle은 OR 조건 처리에 세 가지 주요 전략을 제공합니다:

| 상황 | 권장 전략 | 이유 |
|------|-----------|------|
| 동일 컬럼, 값 수 적음, 분포 유사 | IN-LIST Iterator | 동일 인덱스 반복 스캔이 가장 효율적 |
| 서로 다른 컬럼, 각기 다른 인덱스 | OR Expansion | 분기별 최적 인덱스 동시 활용 |
| 비트맵 인덱스 환경, 다중 조건 | Bitmap OR | 비트맵 연산이 가장 저비용 |
| 분기별 조인 전략 상이 필요 | OR Expansion | 분기마다 다른 실행 계획 허용 |
| 간단한 OR, 확장 오버헤드 큼 | OR 조건 유지 | 변환 비용 > 이익 |

## 실무 적용 패턴

### 패턴 1: 동적 조건 처리

```sql
-- NVL 안티패턴
SELECT *
FROM orders
WHERE status = NVL(:input_status, status);

-- OR Expansion 패턴
SELECT /*+ USE_CONCAT */ *
FROM orders
WHERE :input_status IS NULL
UNION ALL
SELECT *
FROM orders
WHERE status = :input_status
AND :input_status IS NOT NULL;
```

### 패턴 2: 복잡한 비즈니스 규칙

```sql
-- 다양한 할인 조건 적용
SELECT /*+ USE_CONCAT */ 
       product_id,
       SUM(CASE WHEN sale_date >= ADD_MONTHS(SYSDATE, -3) 
                THEN amount ELSE 0 END) as recent_sales,
       SUM(amount) as total_sales
FROM sales
WHERE (product_category = '신제품' AND sale_date >= ADD_MONTHS(SYSDATE, -1))
   OR (customer_tier = 'VIP' AND amount > 100000)
   OR (sale_date BETWEEN '2024-01-01' AND '2024-03-31' 
       AND promotion_flag = 'Y')
GROUP BY product_id;
```

### 패턴 3: 성능 모니터링과 튜닝

```sql
-- OR Expansion 효과 분석
SELECT 
    sql_id,
    plan_hash_value,
    executions,
    elapsed_time,
    buffer_gets,
    disk_reads,
    CASE 
        WHEN INSTR(sql_text, 'USE_CONCAT') > 0 THEN 'OR_EXPAND'
        WHEN INSTR(sql_text, 'NO_EXPAND') > 0 THEN 'NO_EXPAND'
        ELSE 'AUTO'
    END as expansion_type
FROM v$sql
WHERE sql_text LIKE '%WHERE%(condition_A OR condition_B)%'
AND executions > 0
ORDER BY elapsed_time DESC;
```

## OR Expansion 디버깅과 문제 해결

### 실행 계획 분석

```sql
-- 상세 실행 계획 분석
SELECT *
FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(
    sql_id => :target_sql_id,
    format => 'ALLSTATS LAST +PREDICATE +ALIAS +NOTE +PEEKED_BINDS'
));

-- OR Expansion 관련 정보 확인
SELECT 
    operation,
    options,
    object_name,
    cardinality,
    bytes,
    cost,
    access_predicates,
    filter_predicates
FROM v$sql_plan
WHERE sql_id = :target_sql_id
AND operation LIKE '%UNION%' 
   OR options LIKE '%CONCAT%'
ORDER BY id;
```

### 성능 비교 스크립트

```sql
-- OR Expansion 전후 성능 비교
DECLARE
    v_start_time NUMBER;
    v_end_time   NUMBER;
    v_buffer_gets NUMBER;
    v_disk_reads  NUMBER;
BEGIN
    -- OR Expansion 없이
    EXECUTE IMMEDIATE 'ALTER SYSTEM FLUSH BUFFER_CACHE';
    
    SELECT value INTO v_buffer_gets 
    FROM v$mystat 
    WHERE statistic# = (SELECT statistic# FROM v$statname WHERE name = 'session logical reads');
    
    SELECT value INTO v_disk_reads
    FROM v$mystat 
    WHERE statistic# = (SELECT statistic# FROM v$statname WHERE name = 'physical reads');
    
    v_start_time := DBMS_UTILITY.GET_TIME;
    
    FOR rec IN (
        SELECT /*+ NO_EXPAND */ COUNT(*) as cnt
        FROM fact_sales s
        JOIN dim_customers c ON c.customer_id = s.customer_id
        WHERE (c.region = '서울' AND c.membership_tier = 'VIP')
           OR (s.sale_date BETWEEN DATE '2024-01-01' AND DATE '2024-01-31')
    ) LOOP
        DBMS_OUTPUT.PUT_LINE('NO_EXPAND 결과: ' || rec.cnt);
    END LOOP;
    
    v_end_time := DBMS_UTILITY.GET_TIME;
    
    DBMS_OUTPUT.PUT_LINE('NO_EXPAND 실행 시간: ' || 
                        (v_end_time - v_start_time)/100 || '초');
    
    -- OR Expansion 적용
    EXECUTE IMMEDIATE 'ALTER SYSTEM FLUSH BUFFER_CACHE';
    
    v_start_time := DBMS_UTILITY.GET_TIME;
    
    FOR rec IN (
        SELECT /*+ USE_CONCAT */ COUNT(*) as cnt
        FROM fact_sales s
        JOIN dim_customers c ON c.customer_id = s.customer_id
        WHERE (c.region = '서울' AND c.membership_tier = 'VIP')
           OR (s.sale_date BETWEEN DATE '2024-01-01' AND DATE '2024-01-31')
    ) LOOP
        DBMS_OUTPUT.PUT_LINE('USE_CONCAT 결과: ' || rec.cnt);
    END LOOP;
    
    v_end_time := DBMS_UTILITY.GET_TIME;
    
    DBMS_OUTPUT.PUT_LINE('USE_CONCAT 실행 시간: ' || 
                        (v_end_time - v_start_time)/100 || '초');
END;
/
```

## 고급 주제: 병렬 처리와 OR Expansion

병렬 처리 환경에서 OR Expansion은 추가적인 고려사항이 있습니다:

```sql
-- 병렬 OR Expansion
SELECT /*+ USE_CONCAT PARALLEL(s 4) PARALLEL(c 4) */ 
       /*+ PQ_DISTRIBUTE(s HASH HASH) */
       s.sale_id, c.customer_name, s.amount
FROM fact_sales s
JOIN dim_customers c ON c.customer_id = s.customer_id
WHERE (c.region = '서울' AND s.amount > 1000000)
   OR (c.membership_tier = 'VIP' AND s.sale_date >= DATE '2024-06-01')
ORDER BY s.amount DESC;

-- 분기별 다른 병렬 전략
SELECT /*+ USE_CONCAT */ 
       /*+ PQ_DISTRIBUTE(s BROADCAST NONE) */  -- 첫 번째 분기
       s.sale_id, s.amount
FROM fact_sales s
JOIN dim_customers c ON c.customer_id = s.customer_id
WHERE c.region = '서울' AND s.amount > 1000000

UNION ALL

SELECT /*+ PQ_DISTRIBUTE(s HASH HASH) */  -- 두 번째 분기
       s.sale_id, s.amount
FROM fact_sales s
JOIN dim_customers c ON c.customer_id = s.customer_id
WHERE c.membership_tier = 'VIP' 
  AND s.sale_date >= DATE '2024-06-01'
  AND NOT (c.region = '서울' AND s.amount > 1000000)
ORDER BY amount DESC;
```

## 결론: OR Expansion의 현명한 활용

OR Expansion은 Oracle 옵티마이저의 강력한 기능 중 하나로, OR 조건으로 인한 최적화 제약을 해결하는 효과적인 방법입니다. 그러나 이 기술은 만능 해결책이 아니며, 신중한 적용이 필요합니다.

### 핵심 교훈:

1. **적용 대상 식별**: OR Expansion은 서로 다른 인덱스, 파티션 범위, 조인 전략을 요구하는 OR 조건에서 가장 큰 효과를 발휘합니다.

2. **비용-편익 분석**: 분기 생성 오버헤드와 성능 이득을 신중히 비교해야 합니다. 간단한 OR 조건이나 IN-LIST로 처리 가능한 경우에는 OR Expansion이 불필요할 수 있습니다.

3. **중복 관리**: OR 조건 간 중복이 있는 경우 상호배타적 조건을 추가하거나 UNION을 사용하여 정확한 결과를 보장해야 합니다.

4. **통계 정확성**: OR Expansion의 성공은 각 분기의 선택도 추정 정확도에 크게 의존합니다. 정확한 히스토그램과 통계 정보가 필수적입니다.

5. **점진적 적용**: 프로덕션 환경에서는 먼저 테스트 환경에서 효과를 검증하고, 필요한 경우 힌트를 사용하여 점진적으로 적용하는 것이 안전합니다.

OR Expansion을 효과적으로 활용하려면 단순히 기술을 적용하는 것을 넘어, 데이터 패턴, 쿼리 특성, 시스템 환경을 종합적으로 이해해야 합니다. 이는 옵티마이저와의 협업 과정으로, 올바른 이해와 적용을 통해 복잡한 비즈니스 요구사항을 효율적으로 처리할 수 있는 강력한 도구가 될 수 있습니다.

최종적으로, 어떤 최적화 기법이든 실제 실행 결과와 성능 측정으로 효과를 입증하는 것이 가장 중요합니다. OR Expansion 적용 전후의 실행 계획, 자원 사용량, 응답 시간을 꼼꼼히 비교하여 데이터 기반의 결정을 내리는 것이 성공적인 데이터베이스 튜닝의 핵심입니다.