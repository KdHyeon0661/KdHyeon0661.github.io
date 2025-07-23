---
layout: post
title: DB - WHERE 절
date: 2025-02-21 20:20:23 +0900
category: DB
---
# 🔧 SQL 함수(Function) 정리 (MySQL / MSSQL 기준)

## 📌 함수란?

SQL에서 **함수(Function)**는 데이터를 가공하거나 계산하기 위해 제공되는 **미리 정의된 연산 로직**입니다.

- SELECT, WHERE, GROUP BY, ORDER BY 등 다양한 절에서 활용 가능  
- 데이터 조회, 요약, 필터링, 전처리 등에 필수  
- 대부분의 DBMS는 자체 내장 함수를 제공하며, 사용자 정의 함수(UDF)도 가능

## 📂 SQL 함수 분류

| 분류 | 설명 | 예시 |
|------|------|------|
| **단일 행 함수 (Scalar Function)** | 각 행에 대해 개별적으로 작동 | `UPPER(name)`, `ROUND(price, 1)` |
| **그룹 함수 (집계 함수, Aggregate Function)** | 여러 행을 하나로 집계 | `SUM(salary)`, `AVG(score)` |

## 🔹 단일 행 함수

### 🧵 1. 문자열 함수

| 함수 | MySQL | MSSQL | 설명 | 예시 |
|-------|--------|--------|-------------|--------------|
| 대문자 변환 | `UPPER(str)` | `UPPER(str)` | 문자열을 대문자로 변환 | `UPPER('kim') → 'KIM'` |
| 소문자 변환 | `LOWER(str)` | `LOWER(str)` | 문자열을 소문자로 변환 | `LOWER('SEOUL') → 'seoul'` |
| 문자열 길이 | `LENGTH(str)` (바이트 수)<br>`CHAR_LENGTH(str)` (문자 수) | `LEN(str)` | 문자열 길이 | `LENGTH('hello') → 5` / `LEN('hello') → 5` |
| 문자열 자르기 | `SUBSTRING(str, pos, len)` | `SUBSTRING(str, pos, len)` | 문자열 일부 반환 | `SUBSTRING('ABCDEF', 2, 3) → 'BCD'` |
| 문자열 합치기 | `CONCAT(str1, str2, ...)` | `CONCAT(str1, str2, ...)` 또는 `str1 + str2` | 문자열 연결 | `CONCAT('Hello', 'World') → 'HelloWorld'` |
| 공백 제거 | `TRIM(str)` | `LTRIM(RTRIM(str))` 또는 `TRIM(str)` (SQL Server 2017+) | 문자열 앞뒤 공백 제거 | `TRIM(' abc ') → 'abc'` |
| 문자열 교체 | `REPLACE(str, from_str, to_str)` | `REPLACE(str, from_str, to_str)` | 문자열 내 특정 문자 교체 | `REPLACE('hello world', 'world', 'SQL') → 'hello SQL'` |

### 🔢 2. 숫자 함수

| 함수 | MySQL | MSSQL | 설명 | 예시 |
|-------------|--------------|--------------|--------------|-------------|
| 반올림 | `ROUND(x, n)` | `ROUND(x, n)` | 소수 n번째 자리 반올림 | `ROUND(123.456, 1) → 123.5` |
| 자르기(내림) | `TRUNCATE(x, n)` | `FLOOR(x * POWER(10, n)) / POWER(10, n)` 또는 `ROUND(x, n, 1)` (옵션) | 소수점 n자리까지 자르기 | `TRUNCATE(123.456, 1) → 123.4` |
| 나머지 | `MOD(x, y)` 또는 `x % y` | `x % y` | 나머지 계산 | `MOD(10, 3) → 1` |
| 절댓값 | `ABS(x)` | `ABS(x)` | 절댓값 반환 | `ABS(-10) → 10` |
| 올림 | `CEIL(x)` 또는 `CEILING(x)` | `CEILING(x)` | 소수점 올림 | `CEIL(2.3) → 3` |

### 📅 3. 날짜 함수

| 함수 | MySQL | MSSQL | 설명 | 예시 |
|-----------|----------------|----------------|-----------------|--------------|
| 현재 날짜 및 시간 | `NOW()` | `GETDATE()` | 현재 날짜 + 시간 반환 | `NOW() → '2025-07-20 15:00:00'` |
| 현재 날짜 | `CURRENT_DATE` | `CAST(GETDATE() AS DATE)` | 현재 날짜 반환 | `CURRENT_DATE → '2025-07-20'` |
| 특정 날짜에 월 더하기 | `DATE_ADD(date, INTERVAL n MONTH)` | `DATEADD(MONTH, n, date)` | 날짜 기준 월 단위 더하기 | `DATE_ADD('2024-01-01', INTERVAL 3 MONTH) → '2024-04-01'` |
| 월 차이 계산 | `PERIOD_DIFF(EXTRACT(YEAR_MONTH FROM d1), EXTRACT(YEAR_MONTH FROM d2))` (MySQL) <br>혹은 유사 계산 by TIMESTAMPDIFF | `DATEDIFF(month, d2, d1)` (SQL Server는 별도 함수 없음, 날짜 계산 방법 다름) | 두 날짜 월 차이 | MySQL: `TIMESTAMPDIFF(MONTH, '2024-01-01', '2025-01-01') → 12` |
| 시분초 제거 | `DATE()` | `CAST(date AS DATE)` | datetime에서 날짜만 반환 | `DATE(NOW()) → '2025-07-20'` |
| 날짜 차이 (일) | `DATEDIFF(date1, date2)` | `DATEDIFF(day, date2, date1)` | 두 날짜 사이 일 수 차이 | `DATEDIFF('2025-07-21', '2025-07-20') → 1` |

## 🔹 그룹 함수 (집계 함수)

| 함수 | 설명 | MySQL | MSSQL | 예시 |
|------|------|-------|-------|----------------|
| 전체 행 수 | NULL 포함 여부 다름 | `COUNT(*)` | `COUNT(*)` | `SELECT COUNT(*) FROM emp;` |
| 특정 컬럼 NULL 제외 카운트 | NULL 제외 | `COUNT(col)` | `COUNT(col)` | `COUNT(email)` |
| 합계 | NULL 제외 | `SUM(col)` | `SUM(col)` | `SUM(salary)` |
| 평균 | NULL 제외 | `AVG(col)` | `AVG(col)` | `AVG(score)` |
| 최댓값 | NULL 제외 | `MAX(col)` | `MAX(col)` | `MAX(salary)` |
| 최솟값 | NULL 제외 | `MIN(col)` | `MIN(col)` | `MIN(score)` |

## 🧪 실무 예제 (MySQL / MSSQL)

```sql
-- 1. 이름을 대문자로 조회
-- MySQL
SELECT UPPER(name) AS upper_name FROM Employee;

-- MSSQL
SELECT UPPER(name) AS upper_name FROM Employee;

-- 2. 최근 3개월 이내 입사자 조회
-- MySQL
SELECT * FROM Employee WHERE hire_date >= DATE_SUB(NOW(), INTERVAL 3 MONTH);

-- MSSQL
SELECT * FROM Employee WHERE hire_date >= DATEADD(MONTH, -3, GETDATE());

-- 3. 부서별 평균 급여 조회
SELECT dept_id, AVG(salary) AS avg_salary FROM Employee GROUP BY dept_id;

-- 4. 이메일이 NULL이 아닌 고객 수
SELECT COUNT(email) FROM Customer WHERE email IS NOT NULL;

-- 5. 월별 매출 합계 (MySQL 예시)
SELECT DATE_FORMAT(sale_date, '%Y-%m') AS sale_month, SUM(amount) AS total_amount
FROM Sales
GROUP BY sale_month
ORDER BY sale_month;

-- 6. 문자열 내 특정 단어 교체 (MSSQL)
SELECT REPLACE(description, 'old', 'new') AS new_description FROM Products;

-- 7. 날짜 시분초 제거 후 비교
-- MySQL
SELECT * FROM Orders WHERE DATE(order_date) = CURRENT_DATE();

-- MSSQL
SELECT * FROM Orders WHERE CAST(order_date AS DATE) = CAST(GETDATE() AS DATE);
```

## 🛠 실무 주의사항 및 성능 고려

### ✅ 인덱스 사용 주의
- 함수가 적용된 컬럼은 기본적으로 인덱스를 타지 못함

```sql
-- 인덱스 미사용 사례
WHERE UPPER(name) = 'KIM'

-- 인덱스 사용 가능 예 (인덱스를 별도로 만들어야 함)
WHERE name = 'kim'
```

- 필요한 경우 대문자 변환 인덱스 또는 계산된 컬럼을 만들어 성능을 개선

### ✅ NULL 값 처리
- 집계 함수는 NULL을 제외하고 계산  
- COUNT(*)는 NULL 포함, COUNT(col)은 NULL 제외

### ✅ GROUP BY 없는 집계 함수는 전체 집계

```sql
SELECT SUM(salary) FROM Employee; -- 전체 합계
```

## 🎯 확장 주제: 사용자 정의 함수 (UDF)

- 복잡한 반복 로직은 함수로 만들어 재사용 가능

### MySQL 예시 (저장 함수)

```sql
DELIMITER //
CREATE FUNCTION get_tax(salary DECIMAL(10,2))
RETURNS DECIMAL(10,2)
DETERMINISTIC
BEGIN
  RETURN salary * 0.03;
END;
//
DELIMITER ;
```

### MSSQL 예시

```sql
CREATE FUNCTION dbo.get_tax (@salary MONEY)
RETURNS MONEY
AS
BEGIN
  RETURN @salary * 0.03;
END;
```

## ✅ 정리

| 항목 | 설명 |
|------|------|
| 단일 행 함수 | 행별 데이터 가공 (문자열, 숫자, 날짜) |
| 그룹 함수 | 여러 행 묶어 집계 결과 도출 |
| DBMS별 차이 | 함수 명명과 동작 차이를 숙지 필요 |
| 주의사항 | 함수 적용 시 인덱스 비사용 가능성 & NULL 처리 주의 |
| 실무 활용 | 필터링, 전처리, 집계, 시계열 분석, 사용자 함수 통한 업무 효율화 |

> SQL 함수는 데이터 가공뿐 아니라 업무 로직 분리, 성능 최적화를 위해 꼭 익혀야 할 핵심 도구입니다.  
> DBMS별 차이를 이해하고, 적합한 함수 선택 및 인덱스 영향을 고려해야 안정적이고 빠른 쿼리를 작성할 수 있습니다.