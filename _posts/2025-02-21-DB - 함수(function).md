---
layout: post
title: DB - WHERE 절
date: 2025-02-21 20:20:23 +0900
category: DB
---
# 🔧 SQL 함수(Function) 정리

## 📌 함수란?

SQL에서 **함수(Function)**는 데이터를 가공하거나 계산하기 위해 제공되는 **미리 정의된 연산 로직**입니다.

- SELECT, WHERE, GROUP BY, ORDER BY 등 다양한 절에서 활용 가능
- 데이터 조회, 요약, 필터링, 전처리 등에 필수
- 대부분의 DBMS는 자체 내장 함수를 제공하며, 사용자 정의 함수(UDF)도 가능

---

## 📂 SQL 함수 분류

| 분류 | 설명 | 예시 |
|------|------|------|
| **단일 행 함수 (Scalar Function)** | 각 행에 대해 개별적으로 작동 | `UPPER(name)`, `ROUND(price)` |
| **그룹 함수 (집계 함수, Aggregate Function)** | 여러 행을 하나로 집계 | `SUM(salary)`, `AVG(score)` |

---

## 🔹 단일 행 함수

### 📁 1. 문자열 함수

| 함수 | 설명 | 예시 |
|------|------|------|
| `UPPER()` | 대문자 변환 | `UPPER('kim') → 'KIM'` |
| `LOWER()` | 소문자 변환 | `LOWER('SEOUL') → 'seoul'` |
| `LENGTH()` | 문자열 길이 | `LENGTH('hello') → 5` |
| `SUBSTR()` | 문자열 자르기 | `SUBSTR('ABCDEF', 2, 3) → 'BCD'` |
| `CONCAT()` | 문자열 합치기 | `CONCAT('Hello', 'World') → 'HelloWorld'` |
| `TRIM()` | 앞뒤 공백 제거 | `TRIM(' abc ') → 'abc'` |

---

### 📁 2. 숫자 함수

| 함수 | 설명 | 예시 |
|------|------|------|
| `ROUND(x, n)` | 소수점 반올림 | `ROUND(123.456, 1) → 123.5` |
| `TRUNC(x, n)` | 소수점 자르기 | `TRUNC(123.456, 1) → 123.4` |
| `MOD(x, y)` | 나머지 계산 | `MOD(10, 3) → 1` |
| `ABS(x)` | 절댓값 | `ABS(-10) → 10` |

---

### 📁 3. 날짜 함수

| 함수 | 설명 | 예시 |
|------|------|------|
| `SYSDATE` | 현재 날짜와 시간 | `SYSDATE → '2025-07-20 15:00'` |
| `CURRENT_DATE` | 현재 날짜 | `CURRENT_DATE → '2025-07-20'` |
| `ADD_MONTHS(date, n)` | 월 단위 더하기 | `ADD_MONTHS('2024-01-01', 3) → '2024-04-01'` |
| `MONTHS_BETWEEN(d1, d2)` | 월 차이 계산 | `MONTHS_BETWEEN('2025-01-01', '2024-01-01') → 12` |
| `TRUNC(date)` | 날짜의 시분초 제거 | `TRUNC(SYSDATE) → '2025-07-20'` |

---

## 🔹 그룹 함수 (집계 함수)

### 📁 1. 대표 함수

| 함수 | 설명 | 예시 |
|------|------|------|
| `COUNT(*)` | 전체 행 수 | `SELECT COUNT(*) FROM emp` |
| `SUM(col)` | 합계 | `SUM(salary)` |
| `AVG(col)` | 평균 | `AVG(score)` |
| `MAX(col)` | 최댓값 | `MAX(salary)` |
| `MIN(col)` | 최솟값 | `MIN(score)` |

### 🔸 실무 팁:
- `NULL` 값은 집계에서 제외됨 (예: `SUM`, `AVG`)
- `COUNT(*)`는 NULL 포함하지만, `COUNT(col)`은 제외

---

## 🧪 실무 예제

```sql
-- 이름을 대문자로 조회
SELECT UPPER(name) FROM Employee;

-- 최근 3개월 이내 입사자 조회
SELECT * FROM Employee
WHERE hire_date >= ADD_MONTHS(SYSDATE, -3);

-- 부서별 평균 급여
SELECT dept_id, AVG(salary)
FROM Employee
GROUP BY dept_id;

-- 이메일이 NULL이 아닌 사용자 수
SELECT COUNT(email) FROM Customer WHERE email IS NOT NULL;
```

---

## 🛠 실무 주의사항 및 성능 고려

### ✅ 인덱스 사용에 주의
- 함수가 사용된 컬럼은 일반적으로 **인덱스를 타지 못함**
```sql
-- 인덱스 비사용
WHERE UPPER(name) = 'KIM'

-- 인덱스 사용 가능
WHERE name = 'kim' (단, name 컬럼에 UPPER 인덱스를 만들 경우 가능)
```

### ✅ GROUP BY 없이 집계 함수는 전체 집계
```sql
SELECT SUM(salary) FROM Employee; -- 전체 총합
```

---

## 🎯 확장 주제: 사용자 정의 함수 (UDF)

- 실무에서는 복잡한 로직이 자주 반복되면 **사용자 정의 함수**를 만들어 재사용
- PL/SQL, T-SQL 등에서 직접 생성 가능

```sql
CREATE FUNCTION get_tax(salary NUMBER)
RETURN NUMBER
IS
BEGIN
  RETURN salary * 0.03;
END;
```

---

## ✅ 정리

| 항목 | 설명 |
|------|------|
| 단일 행 함수 | 행별 가공: 문자열, 숫자, 날짜 |
| 그룹 함수 | 다수 행 → 하나 결과 |
| 주의점 | NULL 처리, 인덱스 영향, GROUP BY 적절 사용 |
| 실무 활용 | 필터링, 전처리, 집계, 시계열 분석 등 |

> SQL 함수는 단순 조회 이상의 **업무 로직 캡슐화 및 데이터 품질 향상**에 핵심 도구입니다.
> 특히 대량 데이터 환경에서는 성능을 고려한 **함수 최적화**와 **적재적소 사용**이 중요합니다.
