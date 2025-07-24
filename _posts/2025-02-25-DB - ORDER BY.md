---
layout: post
title: DB - ORDER BY
date: 2025-02-25 20:20:23 +0900
category: DB
---
# 🔽 ORDER BY 절 정리

## 📌 개요

- `ORDER BY`는 SQL 결과를 **특정 컬럼 기준으로 정렬**하기 위한 절
- SELECT 쿼리에서 **가장 마지막에 실행**
- 오름차순(ASC, 기본값) 또는 내림차순(DESC) 지정 가능

---

## 🔹 기본 문법

```sql
SELECT 컬럼1, 컬럼2
FROM 테이블명
ORDER BY 컬럼1 [ASC|DESC], 컬럼2 [ASC|DESC];
```

---

## 🔹 예제

### 1. 오름차순 정렬 (기본값)

```sql
SELECT name, age
FROM Employee
ORDER BY age;
```

> 나이 기준으로 작은 순으로 정렬

---

### 2. 내림차순 정렬

```sql
SELECT name, salary
FROM Employee
ORDER BY salary DESC;
```

> 급여 기준으로 높은 순부터 정렬

---

### 3. 다중 정렬

```sql
SELECT name, dept_id, salary
FROM Employee
ORDER BY dept_id ASC, salary DESC;
```

> 부서번호 오름차순 → 급여 내림차순 정렬

---

### 4. 컬럼 번호로 정렬

```sql
SELECT name, age, salary
FROM Employee
ORDER BY 2 DESC;
```

> SELECT 절의 두 번째 컬럼(age)을 기준으로 내림차순

> ❗ 실무에서는 가독성/유지보수 문제로 **컬럼명 사용 권장**

---

### 5. 정렬 + LIMIT(상위 N건)

```sql
-- Oracle (12c 이상)
SELECT name, salary
FROM Employee
ORDER BY salary DESC
FETCH FIRST 5 ROWS ONLY;

-- MySQL / PostgreSQL
SELECT name, salary
FROM Employee
ORDER BY salary DESC
LIMIT 5;
```

> 급여 상위 5명

---

## 🧩 정렬과 NULL 처리

SQL에서는 **NULL 값의 정렬 순서**가 DBMS마다 다름:

| DBMS | 오름차순 | 내림차순 |
|------|----------|-----------|
| Oracle | NULL이 마지막 | NULL이 첫 번째 |
| PostgreSQL | NULL이 첫 번째 | NULL이 마지막 |
| MySQL | NULL이 첫 번째 | NULL이 마지막 |

### 명시적 제어 예시 (PostgreSQL 등)

```sql
ORDER BY score DESC NULLS LAST;
```

---

## 🧠 실무 활용 예시

### 🔸 고객 등급별 정렬 후, 이름순 정렬

```sql
SELECT customer_name, grade
FROM Customers
ORDER BY grade DESC, customer_name ASC;
```

### 🔸 최근 주문순 정렬

```sql
SELECT order_id, customer_id, order_date
FROM Orders
ORDER BY order_date DESC;
```

### 🔸 집계 + 정렬

```sql
SELECT dept_id, COUNT(*) AS emp_count
FROM Employee
GROUP BY dept_id
ORDER BY emp_count DESC;
```

---

## 🛠 실무 주의사항

| 항목 | 설명 |
|------|------|
| 정렬은 느림 | 데이터가 많을수록 정렬 비용 큼 (디스크 정렬 발생 가능) |
| 인덱스 활용 여부 | 정렬 대상 컬럼에 인덱스 있으면 성능 향상 가능 |
| OFFSET 사용 주의 | OFFSET 10000처럼 큰 수를 쓰면 성능 급저하 가능 |
| NULL 정렬 | DBMS마다 순서 다르므로 `NULLS FIRST/LAST` 명시 추천 |

---

## ✅ 정리 요약

| 항목 | 설명 |
|------|------|
| 기본 정렬 | `ORDER BY 컬럼 [ASC|DESC]` |
| 다중 정렬 | 여러 컬럼 기준 우선순위 지정 |
| NULL 처리 | `NULLS FIRST`, `NULLS LAST` 명시 가능 |
| 정렬 + LIMIT | Top-N 추출에 자주 사용 |
| 성능 고려 | 대량 정렬 시 인덱스와 실행계획 분석 필요 |

> 실무에서는 **조회 결과를 순위화하거나, Top-N 추출하거나, 사용자 정렬 옵션을 구현할 때** 필수로 사용되며,
> **정렬 비용을 줄이기 위해 인덱스를 적절히 활용하거나, 페이징 전략과 함께 사용**하는 경우가 많습니다.