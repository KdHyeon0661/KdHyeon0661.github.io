---
layout: post
title: DB - 함수(Function)
date: 2025-02-21 20:20:23 +0900
category: DB
---
# SQL 함수(Function) 완전 정리

## 함수의 분류와 기본 개념

SQL 함수는 그 동작과 반환 값에 따라 네 가지 주요 범주로 구분할 수 있습니다.

| 분류 | 설명 | 예시 |
|---|---|---|
| **단일 행 함수** | 각 행마다 하나의 결과 값을 반환합니다. | `UPPER(name)`, `ROUND(price, 2)` |
| **집계 함수** | 여러 행의 값을 그룹화하여 단일 결과 값을 반환합니다. | `SUM(sales)`, `AVG(score)` |
| **윈도 함수** | 행마다 계산을 수행하지만, 결과 집합의 다른 행을 참조할 수 있습니다. | `ROW_NUMBER()`, `SUM(amount) OVER(...)` |
| **전문 함수** | JSON 처리, 형변환, 보안 등 특수한 작업을 수행합니다. | `JSON_EXTRACT()`, `TRY_CAST()` |

윈도 함수는 GROUP BY로 표현하기 어려운 "행별 집계"나 "순위 계산"을 가능하게 해주는 강력한 도구입니다.

### NULL과 3값 논리
SQL에서는 비교 연산의 결과가 `TRUE`, `FALSE`, 그리고 `UNKNOWN`(NULL) 세 가지 중 하나일 수 있습니다. 이는 중요한 함의를 가집니다.
- `NULL = NULL`은 `TRUE`가 아니라 `UNKNOWN`입니다.
- WHERE 절에서 `UNKNOWN`은 `FALSE`로 처리되어 결과에서 제외됩니다.
- NULL 값과의 비교는 항상 `IS NULL` 또는 `IS NOT NULL` 연산자를 사용해야 합니다.

---

## 문자열 함수: MySQL vs SQL Server 비교

### 문자열 길이 측정
문자 수와 바이트 수는 다를 수 있으며, 이는 데이터베이스마다 다른 함수로 측정합니다.

```sql
-- MySQL: 문자 수 vs 바이트 수
SELECT CHAR_LENGTH('가'), LENGTH('가');  -- 결과: 1, 3 (UTF-8에서 한글은 3바이트)

-- SQL Server: 문자 수 vs 바이트 수 (후행 공백 주의)
SELECT LEN('abc  '), DATALENGTH('abc  ');  -- 결과: 3, 5
SELECT LEN(N'abc  '), DATALENGTH(N'abc  ');  -- 결과: 3, 10 (NVARCHAR는 유니코드)
```

SQL Server의 `LEN()` 함수는 문자열의 후행 공백을 제거한 후의 길이를 반환하므로 주의가 필요합니다.

### 대소문자 변환과 공백 제거
```sql
-- MySQL: 다양한 TRIM 옵션
SELECT TRIM(BOTH '/' FROM '/path/to/file/');  -- 결과: 'path/to/file'

-- SQL Server: TRIM 함수 (2017 이후)
SELECT TRIM('  Hello  ');  -- 결과: 'Hello'
-- 이전 버전에서는 LTRIM(RTRIM('  Hello  ')) 사용
```

### 부분 문자열 추출과 검색
```sql
-- MySQL: 구분자로 문자열 분할
SELECT SUBSTRING_INDEX('a/b/c/d', '/', 2);  -- 결과: 'a/b'

-- SQL Server: 문자열 분할 (2016 이후)
SELECT value FROM STRING_SPLIT('apple,banana,cherry', ',');
```

### 성능 고려사항
문자열 패턴 매칭에서 `LIKE` 연산자를 사용할 때:
- `'abc%'` (전방 고정): 인덱스를 효과적으로 사용할 수 있습니다.
- `'%abc'` (후방 와일드카드): 인덱스 사용이 어렵고 성능이 저하될 수 있습니다.
- `'%abc%'` (양방향 와일드카드): 일반적으로 풀 테이블 스캔이 필요합니다.

---

## 숫자 함수: 정확한 계산을 위한 도구

### 반올림과 절삭
```sql
-- MySQL
SELECT ROUND(123.456, 2);    -- 결과: 123.46
SELECT TRUNCATE(123.456, 2); -- 결과: 123.45

-- SQL Server
SELECT ROUND(123.456, 2);       -- 결과: 123.460
SELECT ROUND(123.456, 2, 1);    -- 결과: 123.450 (세 번째 매개변수 1은 절삭)
```

### 안전한 나눗셈 (0으로 나누기 방지)
```sql
-- MySQL
SELECT COALESCE(amount / NULLIF(quantity, 0), 0) AS unit_price FROM products;

-- SQL Server
SELECT ISNULL(amount * 1.0 / NULLIF(quantity, 0), 0) AS unit_price FROM products;
```

금액이나 가격 계산에는 부동소수점 오차를 피하기 위해 `DECIMAL` 또는 `NUMERIC` 데이터 타입을 사용하는 것이 좋습니다.

---

## 날짜와 시간 함수

### 현재 시간과 날짜 조작
```sql
-- MySQL
SELECT NOW();                              -- 현재 날짜와 시간
SELECT DATE_ADD(NOW(), INTERVAL 7 DAY);    -- 7일 후
SELECT DATEDIFF('2024-12-31', '2024-01-01'); -- 일수 차이

-- SQL Server
SELECT GETDATE();                          -- 현재 날짜와 시간
SELECT DATEADD(DAY, 7, GETDATE());         -- 7일 후
SELECT DATEDIFF(DAY, '2024-01-01', '2024-12-31'); -- 일수 차이
```

### 날짜 범위 쿼리: 반열림 구간 패턴
날짜 범위를 쿼리할 때는 "반열림 구간" 패턴을 사용하는 것이 안전합니다. 이는 경계 값의 중복이나 누락을 방지합니다.

```sql
-- 2024년 1월 데이터 조회 (잘못된 방법)
SELECT * FROM orders WHERE MONTH(order_date) = 1 AND YEAR(order_date) = 2024;

-- 2024년 1월 데이터 조회 (권장 방법 - 반열림 구간)
SELECT * FROM orders 
WHERE order_date >= '2024-01-01' 
  AND order_date < '2024-02-01';
```

두 번째 방법은 인덱스를 효과적으로 사용할 수 있고, 1월 31일 23:59:59.999 같은 경계 값 문제를 피할 수 있습니다.

---

## NULL 처리와 조건부 집계

### NULL 값 다루기
```sql
-- COALESCE: 첫 번째 NULL이 아닌 값 반환
SELECT COALESCE(middle_name, 'N/A') AS display_name FROM users;

-- NULLIF: 두 값이 같으면 NULL 반환 (0으로 나누기 방지에 유용)
SELECT amount / NULLIF(quantity, 0) FROM sales;
```

### 조건부 집계: 다차원 보고서 작성
조건부 집계는 단일 쿼리로 여러 차원의 통계를 생성할 수 있는 강력한 기법입니다.

```sql
-- 월별, 상태별 주문 통계
SELECT 
    DATE_FORMAT(order_date, '%Y-%m') AS month,
    -- 조건부 카운트
    SUM(CASE WHEN status = 'COMPLETED' THEN 1 ELSE 0 END) AS completed_orders,
    SUM(CASE WHEN status = 'CANCELLED' THEN 1 ELSE 0 END) AS cancelled_orders,
    
    -- 조건부 금액 합계
    SUM(CASE WHEN status = 'COMPLETED' THEN amount ELSE 0 END) AS completed_amount,
    SUM(CASE WHEN status = 'CANCELLED' THEN amount ELSE 0 END) AS cancelled_amount,
    
    -- 총계
    COUNT(*) AS total_orders,
    SUM(amount) AS total_amount
    
FROM orders
GROUP BY DATE_FORMAT(order_date, '%Y-%m')
ORDER BY month;
```

---

## 윈도 함수: 고급 분석을 위한 도구

### 이동 평균과 누계 계산
```sql
-- 7일 이동 평균 계산
SELECT 
    sale_date,
    daily_sales,
    AVG(daily_sales) OVER (
        ORDER BY sale_date 
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS moving_avg_7d,
    SUM(daily_sales) OVER (
        ORDER BY sale_date 
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS running_total
FROM daily_sales
ORDER BY sale_date;
```

### 그룹 내 순위 매기기
```sql
-- 부서별 급여 순위
SELECT 
    department_id,
    employee_name,
    salary,
    ROW_NUMBER() OVER (PARTITION BY department_id ORDER BY salary DESC) AS rank_in_dept,
    DENSE_RANK() OVER (PARTITION BY department_id ORDER BY salary DESC) AS dense_rank_in_dept
FROM employees
ORDER BY department_id, salary DESC;
```

### 누적 비율 계산 (파레토 분석)
```sql
-- 제품별 매출 누적 비율
WITH product_sales AS (
    SELECT 
        product_id,
        SUM(amount) AS total_sales
    FROM order_items
    GROUP BY product_id
)
SELECT 
    product_id,
    total_sales,
    SUM(total_sales) OVER (ORDER BY total_sales DESC) AS running_total,
    100.0 * SUM(total_sales) OVER (ORDER BY total_sales DESC) 
        / SUM(total_sales) OVER () AS cumulative_percentage
FROM product_sales
ORDER BY total_sales DESC;
```

---

## 형변환과 타입 안전성

### 안전한 형변환
```sql
-- SQL Server: 변환 실패 시 NULL 반환
SELECT TRY_CAST('123' AS INT);      -- 결과: 123
SELECT TRY_CAST('ABC' AS INT);      -- 결과: NULL (에러 대신 NULL 반환)

-- MySQL: 엄격 모드에서 변환 실패 시 에러 발생
SELECT CAST('123' AS UNSIGNED);     -- 결과: 123
```

### 암시적 형변환 피하기
암시적 형변환은 성능 저하와 예기치 않은 결과를 초래할 수 있습니다.

```sql
-- 비효율적: 인덱스 사용 불가
SELECT * FROM users WHERE CAST(created_at AS DATE) = '2024-01-01';

-- 효율적: 인덱스 사용 가능
SELECT * FROM users 
WHERE created_at >= '2024-01-01' 
  AND created_at < '2024-01-02';
```

---

## JSON 데이터 다루기

### JSON 값 추출과 검색
```sql
-- MySQL
SELECT 
    JSON_EXTRACT(user_data, '$.address.city') AS city,
    JSON_UNQUOTE(JSON_EXTRACT(user_data, '$.name')) AS user_name
FROM users
WHERE JSON_EXTRACT(user_data, '$.active') = 'true';

-- SQL Server
SELECT 
    JSON_VALUE(user_data, '$.address.city') AS city,
    JSON_QUERY(user_data, '$.preferences') AS preferences
FROM users
WHERE JSON_VALUE(user_data, '$.active') = 'true';
```

### JSON 데이터에 인덱스 생성하기
JSON 필드를 자주 검색한다면, 가상 컬럼(MySQL) 또는 계산 열(SQL Server)을 생성하고 인덱스를 추가하는 것이 성능에 도움이 됩니다.

```sql
-- MySQL: 가상 컬럼과 인덱스
ALTER TABLE products
ADD COLUMN category_name VARCHAR(100)
GENERATED ALWAYS AS (JSON_UNQUOTE(JSON_EXTRACT(specs, '$.category'))) STORED;

CREATE INDEX idx_products_category ON products(category_name);

-- SQL Server: 계산 열과 인덱스
ALTER TABLE products
ADD category_name AS JSON_VALUE(specs, '$.category') PERSISTED;

CREATE INDEX idx_products_category ON products(category_name);
```

---

## 성능 최적화: SARGable 쿼리 작성

SARGable(Search ARGument ABLE) 쿼리는 인덱스를 효과적으로 사용할 수 있는 쿼리를 의미합니다.

### SARGable하지 않은 패턴과 개선 방법
```sql
-- 비SARGable: 컬럼을 함수로 감싸면 인덱스 사용 불가
SELECT * FROM orders WHERE YEAR(order_date) = 2024 AND MONTH(order_date) = 1;

-- SARGable: 인덱스 사용 가능
SELECT * FROM orders 
WHERE order_date >= '2024-01-01' 
  AND order_date < '2024-02-01';

-- 비SARGable: 계산식에 컬럼 사용
SELECT * FROM products WHERE price * 1.1 > 100;

-- SARGable: 계산식을 재정렬
SELECT * FROM products WHERE price > 100 / 1.1;
```

### 함수 기반 인덱스 활용
자주 사용하는 함수 표현식에 인덱스를 생성할 수 있습니다.

```sql
-- MySQL: 함수 기반 인덱스
CREATE INDEX idx_users_email_lower ON users((LOWER(email)));

-- 이제 이 쿼리는 인덱스를 사용할 수 있습니다
SELECT * FROM users WHERE LOWER(email) = 'user@example.com';
```

---

## 실무 적용 예제

### 예제 1: 이메일 도메인별 사용자 통계
```sql
-- 이메일에서 도메인 추출 및 통계
SELECT 
    SUBSTRING_INDEX(email, '@', -1) AS email_domain,
    COUNT(*) AS user_count,
    AVG(TIMESTAMPDIFF(YEAR, birth_date, CURDATE())) AS avg_age,
    SUM(CASE WHEN active = 1 THEN 1 ELSE 0 END) AS active_users
FROM users
WHERE email LIKE '%@%'
GROUP BY SUBSTRING_INDEX(email, '@', -1)
ORDER BY user_count DESC;
```

### 예제 2: 계층형 데이터 순회 (재귀 쿼리)
```sql
-- 조직도 조회 (SQL Server)
WITH OrgHierarchy AS (
    -- 기준점: 최상위 관리자
    SELECT 
        employee_id,
        employee_name,
        manager_id,
        1 AS level,
        CAST(employee_name AS VARCHAR(1000)) AS hierarchy_path
    FROM employees
    WHERE manager_id IS NULL
    
    UNION ALL
    
    -- 재귀적으로 하위 직원 추가
    SELECT 
        e.employee_id,
        e.employee_name,
        e.manager_id,
        oh.level + 1,
        CAST(oh.hierarchy_path + ' > ' + e.employee_name AS VARCHAR(1000))
    FROM employees e
    INNER JOIN OrgHierarchy oh ON e.manager_id = oh.employee_id
)
SELECT 
    employee_id,
    employee_name,
    level,
    hierarchy_path
FROM OrgHierarchy
ORDER BY hierarchy_path;
```

### 예제 3: 간격 찾기 (Gaps and Islands)
```sql
-- 연속된 날짜 구간 찾기
WITH DateGroups AS (
    SELECT 
        event_date,
        event_date - INTERVAL (ROW_NUMBER() OVER (ORDER BY event_date)) DAY AS grp
    FROM events
    GROUP BY event_date
)
SELECT 
    MIN(event_date) AS range_start,
    MAX(event_date) AS range_end,
    COUNT(*) AS consecutive_days
FROM DateGroups
GROUP BY grp
HAVING COUNT(*) >= 3  -- 3일 이상 연속된 구간만
ORDER BY range_start;
```

---

## 결론

SQL 함수는 데이터 처리와 분석을 위한 강력한 도구이지만, 올바르게 사용하지 않으면 성능 문제를 초래할 수 있습니다. 효과적인 SQL 함수 사용을 위한 핵심 원칙은 다음과 같습니다.

1. **NULL 이해하기**: SQL의 3값 논리를 이해하고, NULL 값은 항상 `IS NULL` 또는 `IS NOT NULL`로 비교하세요.

2. **SARGable 쿼리 작성**: 가능한 한 컬럼을 함수나 연산으로 감싸지 말고, 인덱스를 활용할 수 있는 형태로 쿼리를 작성하세요. 필요한 경우 함수 기반 인덱스나 계산 열을 활용하세요.

3. **윈도 함수 활용**: 복잡한 분석 요구사항이 있다면 윈도 함수를 고려하세요. 이동 평균, 누계, 순위 계산 등을 간결하게 표현할 수 있습니다.

4. **안전한 형변환**: 데이터 타입 변환이 필요할 때는 암시적 변환에 의존하기보다 명시적 변환 함수를 사용하세요. SQL Server의 `TRY_CAST`나 `TRY_CONVERT`처럼 실패 시 NULL을 반환하는 안전한 함수를 활용하세요.

5. **날짜 범위는 반열림 구간으로**: 날짜 범위를 쿼리할 때는 `>= 시작일 AND < 다음일` 패턴을 사용하여 경계 값 문제를 피하세요.

6. **조건부 집계로 다차원 리포트 작성**: `SUM(CASE WHEN ... THEN ... END)` 패턴을 활용하여 단일 쿼리로 다양한 차원의 통계를 생성하세요.

7. **JSON 데이터는 인덱스와 함께**: JSON 필드를 자주 검색한다면 가상 컬럼이나 계산 열을 생성하고 인덱스를 추가하여 성능을 개선하세요.

8. **데이터베이스별 차이 이해하기**: MySQL과 SQL Server는 비슷한 기능을 다른 함수 이름으로 제공할 수 있습니다. 프로젝트의 데이터베이스에 맞는 함수를 사용하세요.

함수의 힘은 데이터를 변환하고 분석하는 능력에 있지만, 그 힘은 성능 고려사항과 결합되어야 합니다. 올바른 함수 사용은 데이터 작업을 단순화하고, 복잡한 비즈니스 로직을 명확하게 표현하며, 시스템 성능을 유지하는 데 도움이 됩니다.