---
layout: post
title: DB - GROUP BY & HAVING
date: 2025-02-25 19:20:23 +0900
category: DB
---
# GROUP BY & HAVING

## 개념과 실행 순서: SQL의 논리적 처리 파이프라인

**GROUP BY**는 동일한 값을 가진 행을 하나의 그룹으로 묶어 **집계 함수**를 적용합니다. **HAVING**은 그룹화된 결과를 대상으로 조건을 적용하여 필터링합니다.

SQL 쿼리의 **논리적 실행 순서**는 다음과 같습니다:

1. **FROM 및 JOIN**: 테이블에서 데이터를 읽고 조인을 수행
2. **WHERE**: 개별 행을 필터링
3. **GROUP BY**: 행을 그룹으로 묶음
4. **집계 함수 계산**: 각 그룹에 대해 SUM, AVG, COUNT 등의 집계를 수행
5. **HAVING**: 그룹 수준에서 결과를 필터링
6. **SELECT**: 컬럼과 표현식을 계산하고 별칭을 생성
7. **ORDER BY**: 결과를 정렬
8. **LIMIT/OFFSET**: 페이지네이션 적용

**핵심 원칙**: `WHERE`은 그룹화 이전에 **개별 행을 필터링**하고, `HAVING`은 그룹화 이후에 **집계 결과를 기준으로 필터링**합니다.

---

## GROUP BY 핵심 규칙과 흔한 함정

### 비집계 컬럼 규칙: SELECT에 나온 모든 비집계 컬럼은 GROUP BY에 포함되어야 함

ANSI SQL 표준에 따르면, `SELECT` 절에 나열된 모든 비집계 컬럼은 반드시 `GROUP BY` 절에도 포함되어야 합니다.

```sql
-- 올바른 예: SELECT에 나온 모든 비집계 컬럼이 GROUP BY에 포함됨
SELECT department_id, AVG(salary) as avg_salary
FROM employees
GROUP BY department_id;
```

일부 데이터베이스(특히 MySQL의 특정 설정)에서는 이 규칙이 완화되어 비집계 컬럼이 GROUP BY에 없어도 임의의 값을 반환할 수 있습니다. 이는 **데이터 무결성 문제**를 초래할 수 있으므로 피해야 합니다.

```sql
-- MySQL에서 안전한 설정 (항상 활성화 권장)
SET sql_mode = 'ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_DATE,NO_ENGINE_SUBSTITUTION';
```

### GROUP BY만 사용하는 의미

집계 함수 없이 `GROUP BY`만 사용하는 것은 일반적으로 의미가 없거나 오류를 발생시킵니다. 단순히 중복을 제거하고 싶다면 `DISTINCT`를 사용하는 것이 더 명확합니다.

```sql
-- 중복 제거 목적이라면 DISTINCT 사용이 더 명시적입니다
SELECT DISTINCT department_id FROM employees;
```

### 표현식으로 그룹핑하기

`GROUP BY`에는 단순한 컬럼 이름뿐만 아니라 계산된 표현식도 사용할 수 있습니다.

```sql
-- 날짜 부분만으로 그룹핑
SELECT DATE(order_date) as order_day, COUNT(*) as order_count
FROM orders
GROUP BY DATE(order_date)
ORDER BY order_day;
```

**성능 주의사항**: 컬럼을 함수로 감싸면 인덱스를 사용하지 못할 수 있습니다. 성능이 중요한 경우에는 계산된 컬럼을 미리 생성하고 인덱스를 걸어두는 것이 좋습니다.

---

## HAVING 핵심 규칙: 행 필터링과 그룹 필터링의 차이

- **WHERE**: 개별 행을 필터링합니다. 집계 함수를 사용할 수 없습니다.
- **HAVING**: 그룹 수준에서 결과를 필터링합니다. 집계 함수를 사용할 수 있습니다.

```sql
-- 부서별 평균 급여가 300 이상인 부서만 조회
SELECT department_id, AVG(salary) as avg_salary
FROM employees
GROUP BY department_id
HAVING AVG(salary) >= 300;
```

**성능 최적화 패턴**: 가능한 많은 조건을 `WHERE` 절로 이동시켜 그룹화 전에 행 수를 줄이는 것이 성능에 좋습니다. `HAVING`은 그룹화 이후에 최종적으로 그룹을 필터링할 때만 사용하세요.

---

## NULL 처리, DISTINCT, 조건부 집계 패턴

### NULL과 COUNT의 미묘한 차이

- `COUNT(*)`: NULL 값을 포함한 **모든 행의 수**를 셉니다.
- `COUNT(column)`: 해당 컬럼에서 **NULL이 아닌 값의 수**를 셉니다.

```sql
SELECT 
    COUNT(*) as total_rows,
    COUNT(email) as rows_with_email,
    COUNT(*) - COUNT(email) as rows_without_email
FROM customers;
```

### DISTINCT와 집계 함수 조합

- `COUNT(DISTINCT column)`: 컬럼의 고유한 값 개수를 셉니다.
- 다중 컬럼의 고유 조합 개수를 세는 방법은 데이터베이스마다 다릅니다.

```sql
-- MySQL: 직접적인 다중 컬럼 DISTINCT 지원
SELECT COUNT(DISTINCT customer_id, product_id) as unique_combinations
FROM orders;

-- 다른 데이터베이스: 서브쿼리 사용
SELECT COUNT(*) as unique_combinations FROM (
    SELECT DISTINCT customer_id, product_id
    FROM orders
) as distinct_combinations;
```

### 조건부 집계: 한 번의 스캔으로 여러 통계 계산하기

```sql
-- 주문 상태별 통계를 한 번에 계산
SELECT
    SUM(CASE WHEN status = 'PAID' THEN amount ELSE 0 END) as total_paid,
    SUM(CASE WHEN status = 'PENDING' THEN amount ELSE 0 END) as total_pending,
    SUM(CASE WHEN status = 'REFUNDED' THEN amount ELSE 0 END) as total_refunded,
    COUNT(CASE WHEN status = 'PAID' THEN 1 END) as count_paid,
    COUNT(CASE WHEN status = 'PENDING' THEN 1 END) as count_pending
FROM orders;
```

---

## 다중 컬럼·표현식 그룹핑과 날짜 버킷팅

### 다중 컬럼 그룹핑

```sql
-- 부서와 직무별 직원 수 계산
SELECT department_id, job_title, COUNT(*) as employee_count
FROM employees
GROUP BY department_id, job_title
ORDER BY department_id, job_title;
```

### 날짜 버킷팅 (일/주/월/분기/년)

날짜를 기준으로 그룹화하는 것은 실무에서 매우 흔한 패턴입니다. 성능을 위해 **반열림 구간**을 사용하고 원본 컬럼을 그대로 사용하여 인덱스를 활용할 수 있도록 하는 것이 중요합니다.

**일별 집계**:
```sql
-- MySQL
SELECT DATE(order_date) as order_day, COUNT(*) as daily_orders
FROM orders
WHERE order_date >= '2025-01-01'
  AND order_date < '2025-02-01'
GROUP BY DATE(order_date)
ORDER BY order_day;
```

**월별 집계 (성능 최적화)**:
```sql
-- 인덱스 친화적인 월별 집계 (SQL Server)
SELECT
    YEAR(order_date) as order_year,
    MONTH(order_date) as order_month,
    COUNT(*) as monthly_orders,
    SUM(amount) as monthly_revenue
FROM orders
WHERE order_date >= '2025-01-01'
  AND order_date < '2026-01-01'
GROUP BY YEAR(order_date), MONTH(order_date)
ORDER BY order_year, order_month;
```

**성능 팁**: 빈번한 날짜 기반 집계가 필요하다면, 미리 계산된 월 첫날 컬럼을 추가하고 인덱스를 생성하는 것이 좋습니다.

```sql
-- SQL Server: 계산된 컬럼과 인덱스
ALTER TABLE orders
ADD month_start AS DATEFROMPARTS(YEAR(order_date), MONTH(order_date), 1) PERSISTED;

CREATE INDEX ix_orders_month_start ON orders(month_start);
```

---

## 고급 집계 기능: ROLLUP, CUBE, GROUPING SETS

### ROLLUP: 계층적 소계와 총계 생성

```sql
-- 부서와 직무별 인원 수에 부서 소계와 전체 총계 추가
SELECT 
    department_id,
    job_title,
    COUNT(*) as employee_count,
    SUM(salary) as total_salary
FROM employees
GROUP BY ROLLUP (department_id, job_title)
ORDER BY department_id, job_title;
```
- 결과에는 `(부서, 직무)` 상세 데이터, `(부서, NULL)` 부서별 소계, `(NULL, NULL)` 전체 총계가 포함됩니다.

### CUBE: 모든 조합의 소계 생성

```sql
-- 모든 가능한 그룹 조합에 대한 소계 생성
SELECT 
    department_id,
    job_title,
    gender,
    COUNT(*) as employee_count
FROM employees
GROUP BY CUBE (department_id, job_title, gender);
```

### GROUPING SETS: 원하는 조합만 지정

```sql
-- 특정 그룹 조합만 필요한 경우
SELECT 
    department_id,
    job_title,
    COUNT(*) as employee_count
FROM employees
GROUP BY GROUPING SETS (
    (department_id, job_title),  -- 부서-직무 상세
    (department_id),             -- 부서별 소계
    ()                           -- 전체 총계
);
```

### GROUPING 및 GROUPING_ID 함수: 소계 행 식별

```sql
-- 소계 행을 식별하기 위한 함수 사용
SELECT
    department_id,
    job_title,
    COUNT(*) as employee_count,
    GROUPING(department_id) as is_department_subtotal,  -- 1이면 소계 행
    GROUPING(job_title) as is_job_subtotal,            -- 1이면 소계 행
    GROUPING_ID(department_id, job_title) as grouping_level
FROM employees
GROUP BY ROLLUP (department_id, job_title)
ORDER BY department_id, job_title;
```

---

## 집계 함수 vs 윈도 함수: 언제 무엇을 사용해야 할까?

| 요구사항 | 적합한 기능 | 예시 |
|---------|------------|------|
| 그룹당 하나의 요약 행 생성 | **GROUP BY + 집계 함수** | 부서별 평균 급여 |
| 각 행에 그룹 요약값을 표시 | **윈도 함수** | 각 직원에 부서 평균 급여 표시 |
| 그룹 내 순위 또는 누적 합계 | **윈도 함수** | 부서 내 급여 순위, 월별 누계 매출 |
| 복잡한 그룹화와 개별 데이터 모두 필요 | **서브쿼리/CTE + JOIN** | 상세 데이터와 집계 데이터를 함께 표시 |

```sql
-- 윈도 함수 예시: 부서별 급여 순위와 부서 평균 표시
SELECT
    employee_id,
    department_id,
    salary,
    ROW_NUMBER() OVER (PARTITION BY department_id ORDER BY salary DESC) as dept_rank,
    AVG(salary) OVER (PARTITION BY department_id) as dept_avg_salary
FROM employees;
```

---

## 성능 최적화 핵심 전략

### 1. WHERE 절로 먼저 데이터 양 줄이기

가능한 모든 필터링 조건을 `WHERE` 절에 배치하여 그룹화하기 전에 데이터 양을 최소화하세요.

```sql
-- 비효율적: 전체 데이터를 그룹화한 후 HAVING으로 필터링
SELECT customer_id, SUM(amount) as total_spent
FROM orders
GROUP BY customer_id
HAVING MIN(order_date) >= '2025-01-01';

-- 효율적: 먼저 WHERE로 데이터를 필터링한 후 그룹화
SELECT customer_id, SUM(amount) as total_spent
FROM orders
WHERE order_date >= '2025-01-01'
GROUP BY customer_id;
```

### 2. 적절한 인덱스 활용

GROUP BY에 사용되는 컬럼에 인덱스를 생성하면 정렬이나 해시 작업의 부하를 줄일 수 있습니다.

```sql
-- GROUP BY에 자주 사용되는 컬럼 조합에 인덱스 생성
CREATE INDEX ix_orders_customer_date ON orders(customer_id, order_date);
```

### 3. 계산된 컬럼과 함수 기반 인덱스

자주 사용하는 그룹핑 표현식에 대해 미리 계산된 컬럼을 만들고 인덱스를 생성하세요.

```sql
-- PostgreSQL: 함수 기반 인덱스
CREATE INDEX idx_orders_year_month ON orders(EXTRACT(YEAR FROM order_date), EXTRACT(MONTH FROM order_date));

-- MySQL: 생성 컬럼과 인덱스
ALTER TABLE orders
ADD COLUMN order_year_month VARCHAR(7) AS (DATE_FORMAT(order_date, '%Y-%m')) STORED,
ADD INDEX idx_order_year_month (order_year_month);
```

### 4. 카디널리티 고려

- **높은 카디널리티** 컬럼(고유 값이 많은 경우): 해시 집계가 효율적
- **낮은 카디널리티** 컬럼(고유 값이 적은 경우): 정렬 기반 집계가 효율적

---

## 실전 시나리오 10가지

### 1. 기본적인 그룹화와 필터링
```sql
-- 부서별 평균 급여가 50000 이상인 부서 조회
SELECT department_id, AVG(salary) as avg_salary
FROM employees
WHERE hire_date >= '2020-01-01'  -- 신입 직원만
GROUP BY department_id
HAVING AVG(salary) >= 50000
ORDER BY avg_salary DESC;
```

### 2. 시간 기반 그룹화 (최근 6개월)
```sql
-- 최근 6개월 간 월별 매출 집계
SELECT
    DATE_TRUNC('month', order_date) as month,
    COUNT(*) as order_count,
    SUM(amount) as total_revenue,
    AVG(amount) as avg_order_value
FROM orders
WHERE order_date >= CURRENT_DATE - INTERVAL '6 months'
GROUP BY DATE_TRUNC('month', order_date)
ORDER BY month DESC;
```

### 3. 다중 조건 집계 (한 번의 스캔으로 여러 통계)
```sql
-- 제품 카테고리별 다양한 통계 계산
SELECT
    category,
    COUNT(*) as total_products,
    SUM(CASE WHEN price > 100 THEN 1 ELSE 0 END) as premium_count,
    AVG(price) as avg_price,
    MIN(price) as min_price,
    MAX(price) as max_price,
    SUM(stock_quantity) as total_stock
FROM products
GROUP BY category
ORDER BY total_products DESC;
```

### 4. 계층적 보고서 (ROLLUP 활용)
```sql
-- 지역-도시별 매출에 지역 소계와 전체 총계 포함
SELECT
    COALESCE(region, '전체') as region,
    COALESCE(city, CASE WHEN region IS NULL THEN '전체' ELSE '지역 소계' END) as city,
    COUNT(*) as order_count,
    SUM(amount) as total_amount
FROM sales
GROUP BY ROLLUP (region, city)
ORDER BY region, city;
```

### 5. 고유 사용자 기반 집계
```sql
-- 일별 활성 사용자 수 (DAU)
SELECT
    DATE(login_time) as login_date,
    COUNT(DISTINCT user_id) as daily_active_users
FROM user_sessions
WHERE login_time >= CURRENT_DATE - INTERVAL '30 days'
GROUP BY DATE(login_time)
ORDER BY login_date DESC;
```

### 6. 비율 계산 (0으로 나누기 방지)
```sql
-- 제품별 환불률 계산
SELECT
    product_id,
    COUNT(*) as total_orders,
    SUM(CASE WHEN status = 'REFUNDED' THEN 1 ELSE 0 END) as refunded_orders,
    ROUND(
        100.0 * SUM(CASE WHEN status = 'REFUNDED' THEN 1 ELSE 0 END) 
        / NULLIF(COUNT(*), 0), 2
    ) as refund_rate_percent
FROM orders
GROUP BY product_id
HAVING COUNT(*) >= 10  -- 통계적 유의성을 위한 최소 주문 수
ORDER BY refund_rate_percent DESC;
```

### 7. 이동 평균 계산 (윈도 함수와의 조합)
```sql
-- 7일 이동 평균 매출 계산
WITH daily_sales AS (
    SELECT
        DATE(order_date) as sale_date,
        SUM(amount) as daily_total
    FROM orders
    GROUP BY DATE(order_date)
)
SELECT
    sale_date,
    daily_total,
    AVG(daily_total) OVER (
        ORDER BY sale_date 
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) as moving_avg_7d
FROM daily_sales
ORDER BY sale_date;
```

### 8. 상위 N개 그룹 식별
```sql
-- 매출 상위 10개 고객 식별
SELECT
    customer_id,
    customer_name,
    COUNT(*) as order_count,
    SUM(amount) as total_spent
FROM orders
JOIN customers USING (customer_id)
WHERE order_date >= '2025-01-01'
GROUP BY customer_id, customer_name
ORDER BY total_spent DESC
LIMIT 10;
```

### 9. 여러 그룹화 수준을 동시에 보고 (GROUPING SETS)
```sql
-- 다양한 그룹화 수준으로 매출 분석
SELECT
    COALESCE(CAST(year AS VARCHAR), 'All Years') as year_group,
    COALESCE(CAST(quarter AS VARCHAR), 'All Quarters') as quarter_group,
    COUNT(*) as order_count,
    SUM(amount) as total_amount
FROM (
    SELECT 
        EXTRACT(YEAR FROM order_date) as year,
        EXTRACT(QUARTER FROM order_date) as quarter,
        amount
    FROM orders
    WHERE order_date >= '2024-01-01'
) as subquery
GROUP BY GROUPING SETS (
    (year, quarter),  -- 연도-분기 상세
    (year),           -- 연도별 소계
    ()                -- 전체 총계
)
ORDER BY year_group, quarter_group;
```

### 10. 성능 최적화된 대량 데이터 집계
```sql
-- 대량 데이터를 효율적으로 집계하기 위한 전략
WITH filtered_data AS (
    -- 먼저 필요한 데이터만 필터링
    SELECT customer_id, amount, order_date
    FROM orders
    WHERE order_date >= '2025-01-01'
      AND status = 'COMPLETED'
),
aggregated AS (
    -- 필터링된 데이터로 집계 수행
    SELECT
        customer_id,
        COUNT(*) as order_count,
        SUM(amount) as total_spent,
        MAX(order_date) as last_order_date
    FROM filtered_data
    GROUP BY customer_id
)
-- 최종 결과만 선택
SELECT *
FROM aggregated
WHERE total_spent >= 1000
ORDER BY total_spent DESC;
```

---

## 데이터베이스별 구현 차이 요약

| 기능 | MySQL | PostgreSQL | SQL Server | Oracle |
|------|-------|------------|------------|--------|
| **ONLY_FULL_GROUP_BY** | sql_mode로 제어 | 기본 적용 | 기본 적용 | 기본 적용 |
| **다중 컬럼 DISTINCT** | `COUNT(DISTINCT a,b)` 지원 | 서브쿼리 필요 | 서브쿼리 필요 | 서브쿼리 필요 |
| **날짜 버킷팅** | `DATE_FORMAT()`, `YEAR()`, `MONTH()` | `date_trunc()`, `EXTRACT()` | `DATEPART()`, `FORMAT()` | `TRUNC()`, `EXTRACT()` |
| **ROLLUP/CUBE** | 8.0+ 지원 | 지원 | 지원 | 지원 |
| **GROUPING SETS** | 8.0+ 지원 | 지원 | 지원 | 지원 |
| **함수 기반 인덱스** | 생성 컬럼(STORED)으로 시뮬레이션 | 직접 지원 | 계산된 컬럼(PERSISTED) | 함수 기반 인덱스 |
| **근사 집계** | 기본 제공 안됨 | `approx_count_distinct()` | `APPROX_COUNT_DISTINCT()` | `APPROX_COUNT_DISTINCT()` |

---

## 결론: GROUP BY와 HAVING의 효과적인 활용

1. **명확한 설계 원칙**: 
   - `WHERE`은 개별 행 필터링, `HAVING`은 그룹 결과 필터링
   - SELECT의 모든 비집계 컬럼은 GROUP BY에 포함해야 함

2. **성능 최적화 전략**:
   - 가능한 많은 필터링을 WHERE로 선처리하여 데이터 양 최소화
   - GROUP BY 컬럼에 적절한 인덱스 활용
   - 자주 사용하는 그룹핑 표현식은 계산된 컬럼으로 미리 생성

3. **적절한 도구 선택**:
   - 단순 그룹 요약: GROUP BY + 집계 함수
   - 행별 그룹 정보 표시: 윈도 함수
   - 계층적 보고서: ROLLUP, CUBE, GROUPING SETS

4. **실무 패턴 숙지**:
   - 날짜 버킷팅은 반열림 구간 사용
   - 조건부 집계로 한 번의 스캔으로 여러 통계 계산
   - NULL 처리와 0 나누기 방지를 위한 방어적 코딩

5. **데이터베이스 특성 이해**:
   - 사용 중인 데이터베이스의 GROUP BY 구현 특성 이해
   - 데이터베이스별 최적화 기법 적용

GROUP BY와 HAVING은 데이터 분석과 비즈니스 인텔리전스의 핵심 도구입니다. 기본 원칙을 이해하고 적절한 최적화 기법을 적용하면 대규모 데이터에서도 효율적으로 통계와 인사이트를 도출할 수 있습니다. 가장 중요한 것은 실제 데이터와 쿼리 패턴을 분석하여 상황에 맞는 최적의 접근 방식을 선택하는 것입니다.