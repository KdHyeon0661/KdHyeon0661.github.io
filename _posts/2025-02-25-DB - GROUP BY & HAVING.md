---
layout: post
title: DB - GROUP BY & HAVING
date: 2025-02-25 19:20:23 +0900
category: DB
---
# 📊 GROUP BY & HAVING 절 정리

## 📌 개요

- `GROUP BY`: 같은 값을 가진 행들끼리 묶어 그룹을 형성하고, 집계함수(SUM, AVG, COUNT 등)와 함께 사용
- `HAVING`: 그룹에 대한 조건을 필터링 (집계 결과에 조건을 걸 때 사용)

---

## 🔹 GROUP BY 문법

```sql
SELECT 컬럼명1, 집계함수(컬럼명2)
FROM 테이블명
GROUP BY 컬럼명1;
```

### ✅ 특징
- SELECT절에 있는 **비집계 컬럼**은 반드시 `GROUP BY`에 포함되어야 함
- 집계함수 없이 `GROUP BY`만 쓰는 건 오류

---

## 🔹 HAVING 문법

```sql
SELECT 컬럼명1, 집계함수(컬럼명2)
FROM 테이블명
GROUP BY 컬럼명1
HAVING 집계함수(컬럼명2) 조건;
```

### ✅ 특징
- `WHERE`은 **행(row)** 필터링, `HAVING`은 **그룹(group)** 필터링
- `HAVING`은 `GROUP BY` 이후 동작

---

## 🧪 예제

### 🟡 1. 부서별 평균 급여

```sql
SELECT dept_id, AVG(salary) AS avg_salary
FROM Employee
GROUP BY dept_id;
```

### 🟠 2. 평균 급여가 300 이상인 부서만 조회

```sql
SELECT dept_id, AVG(salary) AS avg_salary
FROM Employee
GROUP BY dept_id
HAVING AVG(salary) >= 300;
```

### 🔵 3. 부서별 직원 수

```sql
SELECT dept_id, COUNT(*) AS emp_count
FROM Employee
GROUP BY dept_id;
```

---

## 🧩 GROUP BY + WHERE + HAVING 흐름

1. `WHERE`: 행 필터링
2. `GROUP BY`: 그룹 형성
3. `HAVING`: 그룹 조건 필터링
4. `SELECT`: 결과 반환
5. `ORDER BY`: 정렬

```sql
SELECT dept_id, AVG(salary)
FROM Employee
WHERE job = 'SALES'
GROUP BY dept_id
HAVING AVG(salary) > 500
ORDER BY AVG(salary) DESC;
```

---

## 🛠 실무 팁

### ✅ 집계 없는 GROUP BY → 오류

```sql
-- 오류 발생
SELECT dept_id, name
FROM Employee
GROUP BY dept_id;
```

> 해결법: name에 집계함수 사용하거나 GROUP BY에 포함해야 함

---

### ✅ GROUP BY 다중 컬럼

```sql
SELECT dept_id, job, COUNT(*)
FROM Employee
GROUP BY dept_id, job;
```

→ 부서+직무 조합별 인원 수

---

### ✅ GROUP BY + ROLLUP (누계 계산 등)

```sql
SELECT dept_id, job, COUNT(*)
FROM Employee
GROUP BY ROLLUP(dept_id, job);
```

> 부서·직무별 + 소계 + 전체 합계까지 제공 (OLAP 분석용)

---

### ✅ HAVING과 WHERE 차이

| 항목 | WHERE | HAVING |
|------|-------|--------|
| 대상 | 개별 행(row) | 그룹(group) |
| 시점 | GROUP BY 전에 작동 | GROUP BY 후에 작동 |
| 집계함수 사용 | 불가능 | 가능 |

```sql
-- WHERE은 불가능
WHERE COUNT(*) > 10  ❌

-- HAVING은 가능
HAVING COUNT(*) > 10 ✅
```

---

## 🧠 실무 예시

### 🔸 각 상품별 판매 건수가 100건 이상인 경우만 조회

```sql
SELECT product_id, COUNT(*) AS sales_count
FROM Orders
GROUP BY product_id
HAVING COUNT(*) >= 100;
```

### 🔸 최근 6개월 주문 중, 고객별 총액 500 이상인 사람

```sql
SELECT customer_id, SUM(total_amount) AS total
FROM Orders
WHERE order_date >= ADD_MONTHS(SYSDATE, -6)
GROUP BY customer_id
HAVING SUM(total_amount) >= 500;
```

---

## ✅ 정리 요약

| 구분 | 설명 |
|------|------|
| GROUP BY | 같은 값을 가진 행을 하나의 그룹으로 묶음 |
| HAVING | 그룹 단위 조건 지정 (집계 결과 필터링) |
| WHERE vs HAVING | WHERE은 개별 행, HAVING은 그룹 대상 |
| 실무 활용 | 매출 집계, 방문자 분석, 부서 통계, 거래 요약 등 |

> 실무에서는 **GROUP BY + HAVING** 조합이 매출, 고객 분석, 사용자 활동 요약 등에 매일같이 쓰이며,
> **서브쿼리**나 **윈도우 함수**와 결합되어 복잡한 리포트를 구성하는 데 중요한 핵심 기능입니다.