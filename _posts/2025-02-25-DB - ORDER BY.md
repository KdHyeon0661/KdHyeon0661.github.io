---
layout: post
title: DB - ORDER BY
date: 2025-02-25 20:20:23 +0900
category: DB
---
# ORDER BY

## 핵심 한 줄 요약

- **정확성**: 동일 키 동점(tie)이 있을 때 **보조 정렬 키**를 넣어 **결정성**을 보장하라.
- **성능**: 정렬은 비싸다. **인덱스 정렬 재활용**(index order)과 **키셋 페이징**을 최우선 고려하라.
- **이식성**: **NULL 순서·Collation·Locale**은 RDBMS마다 다르다. **명시**하라.

---

## 개념과 실행 순서(논리 파이프라인)

`ORDER BY`는 **결과 집합의 최종 정렬** 단계다. SQL의 논리적 실행 순서는 개념적으로 다음과 같다.

1. `FROM` … `JOIN`
2. `WHERE`          — **행 필터**
3. `GROUP BY`       — 그룹 생성
4. 집계 계산
5. `HAVING`         — **그룹 필터**
6. `SELECT`         — 투영/표현식 계산
7. **`ORDER BY`**   — **정렬**
8. `LIMIT/OFFSET` or `FETCH` — **Top-N/페이징**

정렬 비용은 일반적으로 $$O(n \log n)$$이며(메모리/디스크 정렬 혼합), 정렬 컬럼에 맞는 **인덱스**가 있으면 **정렬 단계 자체를 생략**하거나 크게 줄일 수 있다.

---

## 기본 문법과 결정성(Determinism)

```sql
SELECT col1, col2
FROM T
ORDER BY col1 [ASC|DESC], col2 [ASC|DESC];
```

### 동점(tie)과 결정성

동일한 `col1` 값이 다수면 **결과 순서가 비결정적**일 수 있다. **항상 보조 키**를 넣어 결정적으로 만든다.

```sql
-- 비결정적 (col1이 같을 때 순서 미정)
ORDER BY col1;

-- 결정적 (동점 시 col2로 재정렬)
ORDER BY col1, col2;
```

**주의**: `ORDER BY`가 없는 서브쿼리/뷰의 내부 순서를 **상위 쿼리가 보장받지 못한다**. 최종 쿼리에서 **반드시** `ORDER BY`를 명시하라.

---

## 컬럼 번호 정렬, 별칭, 표현식, CASE 정렬

### 컬럼 번호 정렬

```sql
SELECT name, age, salary
FROM Employee
ORDER BY 2 DESC;      -- 2번째 열(age)
```
> 실무 권장: **가독성/유지보수** 때문에 **컬럼명(또는 별칭)**을 쓰자.

### 별칭 정렬

```sql
SELECT name, salary * 1.1 AS salary_new
FROM Employee
ORDER BY salary_new DESC;
```

### 표현식 정렬

```sql
SELECT order_id, order_dt
FROM Orders
ORDER BY DATE(order_dt), order_id;
```

### CASE로 사용자 정의 순서

```sql
-- 상태의 임의 우선순위 지정: PENDING > PAID > REFUND > 그 외
SELECT order_id, status
FROM Orders
ORDER BY
  CASE status
    WHEN 'PENDING' THEN 1
    WHEN 'PAID'    THEN 2
    WHEN 'REFUND'  THEN 3
    ELSE 9
  END, order_id;
```

### 불리언/조건식 정렬

```sql
-- 최근 7일 이내 주문을 상단에 올리고, 그 안에서는 최신순
SELECT order_id, order_dt
FROM Orders
ORDER BY (order_dt >= CURRENT_DATE - INTERVAL '7 DAY') DESC,
         order_dt DESC;
```

---

## NULL 정렬과 Collation/Locale

### NULL 순서

DBMS별 기본 NULL 위치가 다르다(사용자 초안 요약과 동일). **명시적으로 제어**하라.

```sql
-- PostgreSQL/Oracle 등
ORDER BY score DESC NULLS LAST;

-- MySQL (명시적 제어가 필요하면 CASE)
ORDER BY (score IS NULL), score DESC;  -- NULL이 마지막
```

### Collation/Locale

문자 정렬은 **Collation**에 따라 다르다(대소문자, 악센트, 한글 초성/종성 등).

```sql
-- MySQL: 특정 Collation 강제
SELECT name
FROM Member
ORDER BY name COLLATE utf8mb4_0900_as_cs;  -- 악센트/대소문자 구분
```

**주의**: 서로 다른 Collation 컬럼을 합쳐 정렬하면 **임시 변환/디스크 정렬** 발생 가능 → **동일 Collation 정규화**가 성능/일관성에 좋다.

---

## 자연 정렬(Natural Sort)과 숫자형 문자열

문자열 `"2" < "10"` 문제를 해결하려면 **숫자 캐스팅** 또는 **패딩**을 사용한다.

```sql
-- 숫자 캐스팅
SELECT code
FROM Items
ORDER BY CAST(code AS UNSIGNED);

-- 고정 폭 패딩(문자 기반 정렬 유지)
SELECT code
FROM Items
ORDER BY LPAD(code, 6, '0');
```

---

## 정렬과 인덱스 — “정렬 생략”이 최강의 최적화

### 인덱스 순서 재사용(Index order)

`WHERE`의 **선택도 높은 선행 컬럼**과 `ORDER BY`의 컬럼 순서/방향이 **인덱스 정의와 일치**하면 **정렬 없이** 스캔만으로 정렬된 결과를 얻는다.

```sql
-- 인덱스: (dept_id ASC, salary DESC)
CREATE INDEX ix_emp_dept_salary ON Employee(dept_id ASC, salary DESC);

-- 정렬 비용 0 (인덱스 순서 그대로)
SELECT name, dept_id, salary
FROM Employee
WHERE dept_id IN (10, 20)
ORDER BY dept_id ASC, salary DESC;
```

**규칙 요약**
- `WHERE` 조건이 인덱스 **왼쪽 접두(prefix)** 를 잘 사용하고,
- `ORDER BY`가 인덱스 **키 순서/방향과 동일**,
- 필요 시 **커버링 인덱스**(SELECT 컬럼이 모두 인덱스에 존재)면 **랜덤 접근도 감소**.

### 정렬 방향 혼합

서로 다른 방향(ASC/DESC) 혼합도 **인덱스 정의에서 방향까지 일치**해야 정렬 생략 가능(DBMS별 제약 다름).

```sql
-- 혼합 방향 인덱스
CREATE INDEX ix_orders_dt_desc_amt_asc ON Orders(order_dt DESC, amount ASC);

SELECT order_id
FROM Orders
WHERE order_dt >= '2025-01-01'
ORDER BY order_dt DESC, amount ASC;
```

### 함수 적용 시 인덱스 무력화

`ORDER BY DATE(order_dt)`처럼 **표현식**을 감싸면 인덱스 정렬을 재사용하지 못한다.
→ **계산(생성) 열 + 인덱스** 또는 **반열림 구간 + 원본 컬럼 정렬**을 고려.

```sql
-- SQL Server: 계산 열 + 인덱스
ALTER TABLE dbo.Orders
ADD order_date DATE
  PERSISTED AS CAST(order_dt AS DATE);
CREATE INDEX ix_orders_order_date ON dbo.Orders(order_date);

SELECT *
FROM dbo.Orders
WHERE order_date BETWEEN '2025-10-01' AND '2025-10-31'
ORDER BY order_date, order_id; -- 정렬 생략 가능
```

---

## Top-N, 페이징, WITH TIES, 키셋 페이징

### Top-N

```sql
-- Oracle 12c+
SELECT name, salary
FROM Employee
ORDER BY salary DESC
FETCH FIRST 5 ROWS ONLY;

-- MySQL/PostgreSQL
SELECT name, salary
FROM Employee
ORDER BY salary DESC
LIMIT 5;
```

### WITH TIES (동점 포함)

```sql
-- SQL Server / Oracle 12c+ / PostgreSQL(호환 구문 상이)
SELECT name, salary
FROM Employee
ORDER BY salary DESC
FETCH FIRST 5 ROWS WITH TIES;  -- 5위와 동점 모두 포함
```

### OFFSET의 비용과 대안

`OFFSET 100000`은 **앞선 10만 건을 스킵**해야 하므로 매우 비싸다.
**키셋 페이징(Keyset/Cursor Paging)**으로 전환하라.

```sql
-- 키셋 페이징 (마지막 키 기준)
SELECT order_id, order_dt
FROM Orders
WHERE (order_dt, order_id) < (:last_dt, :last_id)   -- DB별 튜플 비교 문법 상이
ORDER BY order_dt DESC, order_id DESC
LIMIT 50;
```

**장점**: 큰 OFFSET 없이 **바로 다음 페이지**를 가져온다. 인덱스 순서 재사용으로 매우 빠르다.
**주의**: **결정적 정렬 키 조합**(유니크 근사)이 필수.

---

## ORDER BY와 집계/윈도 함수의 경계

### 집계 + 정렬

```sql
SELECT dept_id, COUNT(*) AS emp_count
FROM Employee
GROUP BY dept_id
ORDER BY emp_count DESC, dept_id ASC;
```

### 윈도 함수의 `ORDER BY`

윈도 함수의 `ORDER BY`는 **파티션 내부의 계산 순서**를 정의한다.
**결과의 표시 순서**를 바꾸려면 **외부 `ORDER BY`**가 별도로 필요하다.

```sql
-- 파티션 내 최근 순위
SELECT
  customer_id,
  order_id,
  order_dt,
  ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_dt DESC) AS rn
FROM Orders
ORDER BY customer_id, rn;  -- 최종 표시 순서
```

---

## 날짜/시간 정렬 — 반열림 구간과 안정 키

```sql
-- 월 범위(반열림) + 안정 정렬 키
SELECT order_id, order_dt
FROM Orders
WHERE order_dt >= '2025-10-01'
  AND order_dt <  '2025-11-01'
ORDER BY order_dt, order_id;
```

- 반열림 구간은 경계 포함/배제 혼선을 줄이고 **인덱스 사용**에 유리하다.
- 같은 시각(order_dt 동점)에는 `order_id` 같은 **보조 키**로 결정성 보장.

---

## 정렬 안정성(Stable Sort)과 타 DB 영향

일부 엔진은 내부적으로 **안정 정렬이 아닐 수 있다**. 즉, 동점의 상대 순서가 입력 순서와 같다는 보장이 없다.
→ **언제나 보조 키를 포함**해 결정적으로 하라.

---

## 대용량 정렬: 메모리 vs 디스크(Temp)와 근사해법

- 정렬 메모리를 초과하면 디스크(Temp)로 spill. I/O 급증.
- **인덱스 순서 재활용**, **Top-N with index**, **키셋 페이징**으로 회피.
- 분석용 근사순위/샘플링 등(엔진별 기능)도 검토.

---

## 다이얼렉트 차이 요약

| 항목 | MySQL | PostgreSQL | SQL Server | Oracle |
|---|---|---|---|---|
| LIMIT/OFFSET | `LIMIT n OFFSET m` | 동일 | `OFFSET m ROWS FETCH NEXT n ROWS ONLY` | `FETCH FIRST n ROWS [WITH TIES]` |
| NULLS 제어 | 직접 구문 없음 → `CASE` | `NULLS FIRST/LAST` | `NULLS FIRST/LAST` 미지원(대체 로직 필요) | `NULLS FIRST/LAST` |
| Collation | `COLLATE` 지원 | `COLLATE`/ICU | 데이터베이스/열/표현식 수준 | NLS 기반, `NLSSORT` 등 |
| 혼합 방향 인덱스 | 8.0+ 지원 | 지원 | 지원 | 지원 |
| WITH TIES | 8.0.31+ `LIMIT ... WITH TIES` (주의: 버전별) | 13+ `FETCH WITH TIES` | `TOP ... WITH TIES` / `FETCH ... WITH TIES` | `FETCH ... WITH TIES` |

> 실제 지원 버전은 환경에 따라 다르므로, 프로젝트 표준 DB/버전을 기준으로 확인할 것.

---

## 실무 시나리오와 예제

### 고객 등급 > 이름 순

```sql
SELECT customer_name, grade
FROM Customers
ORDER BY grade DESC, customer_name ASC;
```

### 최근 주문순 + NULL 주문일 뒤로

```sql
-- PostgreSQL/Oracle
SELECT order_id, customer_id, order_dt
FROM Orders
ORDER BY order_dt DESC NULLS LAST, order_id DESC;
```

### 집계 결과 정렬(부서 인원 Top-K)

```sql
-- MySQL/PostgreSQL
SELECT dept_id, COUNT(*) AS emp_count
FROM Employee
GROUP BY dept_id
ORDER BY emp_count DESC, dept_id
LIMIT 10;
```

### 사용자 지정 정렬(카테고리 그룹 우선)

```sql
SELECT product_id, category
FROM Products
ORDER BY
  CASE category
    WHEN 'Premium' THEN 1
    WHEN 'Standard' THEN 2
    WHEN 'Trial' THEN 3
    ELSE 9
  END,
  product_id;
```

### 문자열 숫자 자연 정렬

```sql
-- MySQL
SELECT version
FROM Releases
ORDER BY CAST(SUBSTRING_INDEX(version, '.', 1) AS UNSIGNED),
         CAST(SUBSTRING_INDEX(SUBSTRING_INDEX(version, '.', 2), '.', -1) AS UNSIGNED),
         CAST(SUBSTRING_INDEX(version, '.', -1) AS UNSIGNED);
```

### 키셋 페이징(무한 스크롤)

```sql
-- 마지막 행의 (created_at, id)를 기억
SELECT id, created_at, title
FROM Posts
WHERE (created_at, id) < (:last_created_at, :last_id)
ORDER BY created_at DESC, id DESC
LIMIT 30;
```

### 인덱스와 정렬 재활용

```sql
-- 인덱스 준비
CREATE INDEX ix_emp_dept_salary ON Employee(dept_id, salary DESC);

-- 정렬 생략(스트림 정렬)
SELECT emp_id, dept_id, salary
FROM Employee
WHERE dept_id BETWEEN 10 AND 20
ORDER BY dept_id, salary DESC;
```

### 날짜 버킷 행렬: 일자 오름·금액 내림

```sql
SELECT order_date, SUM(amount) AS amt
FROM Orders
WHERE order_date >= '2025-10-01'
  AND order_date <  '2025-11-01'
GROUP BY order_date
ORDER BY order_date ASC;   -- 집계 후 표 형식용
```

### WITH TIES로 동점 포함 Top-N

```sql
-- Oracle/SQL Server/PostgreSQL (구문 차이 있음)
SELECT name, sales
FROM Salesperson
ORDER BY sales DESC
FETCH FIRST 3 ROWS WITH TIES;
```

### 다국어 정렬(한국어/영문 혼재)

```sql
-- MySQL 예시
SELECT name
FROM Directory
ORDER BY name COLLATE 'utf8mb4_0900_ai_ci', id;  -- 악센트/대소문자 무시 정렬 + 보조키
```

---

## 성능 체크리스트

- [ ] **정렬 컬럼 인덱스** 보유 여부 확인(가능하면 정렬 생략)
- [ ] `WHERE` 선택도 높은 컬럼이 **인덱스 선두**에 있는가
- [ ] `ORDER BY` 방향/순서가 **인덱스 정의와 일치**하는가
- [ ] **표현식/함수**가 인덱스 사용을 막지 않는가(계산 열/함수 인덱스 고려)
- [ ] **OFFSET 큰 값** 사용 지양, **키셋 페이징**으로 대체
- [ ] **NULLS FIRST/LAST** 등 **명시적 제어**로 이식성 확보
- [ ] Collation/Locale 혼재로 인한 **임시 변환/디스크 정렬** 발생 여부
- [ ] **WITH TIES** 요구사항(리포트 정확성) 반영 여부
- [ ] 실행계획에서 **External Sort/Temp Spill** 발생 여부 점검

---

## 자주 하는 실수와 예방

| 실수 | 영향 | 예방 |
|---|---|---|
| 동점 시 보조 키 없음 | 비결정 순서 → UI 깜빡임/불안정 | 항상 2차 키 추가 |
| OFFSET 10만 페이지 | 극심한 성능 저하 | 키셋 페이징 전환 |
| 함수 감싼 정렬 컬럼 | 인덱스 무력화 | 계산 열/함수 인덱스/반열림 범위 |
| Collation 섞인 정렬 | 디스크 정렬/비일관 | 스키마 Collation 정규화 |
| NULL 순서 가정 | DBMS별 상이 동작 | `NULLS FIRST/LAST` 또는 `CASE` 명시 |

---

## 실행계획 힌트(개념)

- **Using index for order by**: 인덱스 스캔으로 정렬 생략(스트림 정렬)
- **Filesort/Sort**: 정렬 연산 발생(메모리/디스크)
- **Top-N sort**: 필요 건수만 유지하는 최적화(있으면 좋음)
- **Parallel sort**: MPP/병렬 엔진에서 정렬 병렬화

---

## 수학적 비용 감각

정렬은 일반적으로 $$O(n \log n)$$, **외부정렬(external sort)** 시 I/O 왕복이 추가된다.
**인덱스 순서 재사용**은 사실상 **$$O(n)$$** 스캔으로 대체되어 **압도적 이득**을 준다.

---

# 결론

- `ORDER BY`는 단순한 문법이 아니라 **결과의 정확성(결정성)과 체감 성능**을 좌우하는 핵심이다.
- **보조 키로 결정성 확보**, **인덱스 재사용으로 정렬 생략**, **키셋 페이징으로 대용량 대응**, **NULL/Collation 명시**—이 네 가지가 실무 품질을 가른다.
- 최종적으로, **실행계획 확인**과 **인덱스/표현식 설계**가 성패를 좌우한다. 오늘 바로 쿼리에서 **보조 정렬 키**와 **키셋 페이징**을 적용해 보자.
