---
layout: post
title: DB - GROUP BY & HAVING
date: 2025-02-25 19:20:23 +0900
category: DB
---
# GROUP BY & HAVING

## 0. 빠른 로드맵

1. 개념·실행 순서(논리적 단계)
2. GROUP BY 핵심 규칙(함정 포함)
3. HAVING 핵심 규칙(행 vs 그룹, 재필터링)
4. NULL·DISTINCT·조건부 집계 패턴
5. 다중 컬럼·표현식 그룹핑과 날짜 버킷팅(분/시/일/월/주)
6. 고급 집계: ROLLUP, CUBE, GROUPING SETS, GROUPING_ID
7. 집계 + 윈도 함수 경계(언제 무엇을 쓰는가)
8. 성능 최적화: 인덱스/프리카드·카디널리티/Pushdown/Partial Aggregation
9. 실전 시나리오 10선(코드 완비)
10. 다이얼렉트(MySQL/SQL Server/PostgreSQL/Oracle) 차이 요약
11. 체크리스트

---

## 1. 개념과 실행 순서(논리적 파이프라인)

**GROUP BY**는 동일 키를 가진 행을 묶어 **집계 함수**(`SUM/AVG/COUNT/MIN/MAX/STDDEV/VARIANCE …`)를 계산한다.
**HAVING**은 **그룹** 단위로 **집계 결과**를 필터링한다.

SQL의 **논리적 실행 순서(개념적)**

1. `FROM` … `JOIN`
2. `WHERE` (행 필터)
3. `GROUP BY` (그룹 생성)
4. 집계 함수 계산
5. `HAVING` (그룹 필터)
6. `SELECT` (표현식 산출, 별칭 생성)
7. `ORDER BY` (정렬)
8. `LIMIT/OFFSET` (페이지)

> **핵심**: `WHERE`은 그룹 이전에 **행을 줄이는 필터**, `HAVING`은 그룹 이후에 **집계 결과로 필터**.

---

## 2. GROUP BY 핵심 규칙과 흔한 함정

### 2.1 비집계 컬럼 규칙
- ANSI SQL에서 **`SELECT`에 나오는 비집계 컬럼은 전부 `GROUP BY`에 포함**되어야 한다.

```sql
-- 올바른 예
SELECT dept_id, AVG(salary)
FROM Employee
GROUP BY dept_id;
```

- 일부 DB(옛 MySQL의 ONLY_FULL_GROUP_BY 비활성)는 비집계 컬럼이 GROUP BY에 없어도 **임의 값**을 반환할 수 있어 **위험**.
  → **항상** `ONLY_FULL_GROUP_BY` 활성(또는 표준 준수).

```sql
-- 권장 세팅(MySQL)
SET sql_mode = 'ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_DATE,NO_ENGINE_SUBSTITUTION';
```

### 2.2 집계 없는 GROUP BY는 불가
- **집계 함수 없이** `GROUP BY`만 쓰면 보통 의미가 없거나 오류.
  (PostgreSQL은 `GROUP BY`만으로도 “중복 제거 + 정렬된 유사 효과”가 나지만 **집계가 목적**이 아니라면 `DISTINCT`가 더 명확.)

```sql
-- 중복 제거 목적이면 DISTINCT를 쓰는 편이 명시적
SELECT DISTINCT dept_id FROM Employee;
```

### 2.3 표현식으로 그룹핑 가능
- `GROUP BY`에는 **컬럼만**이 아니라 **표현식**도 올 수 있다(다이얼렉트별 제한 약간).

```sql
SELECT DATE(order_dt) AS d, COUNT(*)
FROM Orders
GROUP BY DATE(order_dt)
ORDER BY d;
```

> **성능 주의**: 인덱스 컬럼을 함수로 감싸면 인덱스를 못 타는 경우가 많다.
> 날짜 버킷팅은 **반열림 구간**이나 **계산 열/함수 기반 인덱스**로 인덱스 친화적으로 바꾸는 게 정석(8장 참조).

---

## 3. HAVING 핵심 규칙 — 행 vs 그룹

- `WHERE`은 **개별 행 조건**. 집계 함수 사용 불가.
- `HAVING`은 **그룹 조건**. 집계 함수 사용 가능.

```sql
-- 예: 부서별 평균 급여가 300 이상인 부서
SELECT dept_id, AVG(salary) AS avg_salary
FROM Employee
GROUP BY dept_id
HAVING AVG(salary) >= 300;
```

**재필터링 패턴**
- 우선 `WHERE`로 **행 수를 줄이고**, 이후 `HAVING`으로 그룹을 **다시 줄이는** 것이 일반적 성능 패턴.

---

## 4. NULL, DISTINCT, 조건부 집계

### 4.1 NULL과 COUNT의 차이
- `COUNT(*)` : NULL 포함 **전체 행수**
- `COUNT(col)` : **NULL 제외** 카운트

```sql
SELECT COUNT(*) AS rows_all,
       COUNT(email) AS rows_with_email
FROM Customer;
```

### 4.2 DISTINCT와 집계
- `COUNT(DISTINCT col)` : 고유 값 개수
- 다중 컬럼 고유 개수: `COUNT(DISTINCT col1, col2)`(MySQL),
  SQL Server/PostgreSQL은 `COUNT(DISTINCT CONCAT(...))` 또는 구조에 따라 `COUNT(*)` + `DISTINCT` 서브쿼리.

```sql
-- MySQL: 복합 고유 수
SELECT COUNT(DISTINCT customer_id, product_id)
FROM Orders;

-- SQL Server/PostgreSQL: 서브쿼리
SELECT COUNT(*) FROM (
  SELECT DISTINCT customer_id, product_id
  FROM Orders
) d;
```

### 4.3 조건부 집계(리포트의 기본)
```sql
-- 상태별 카운트/금액 합을 한 번에
SELECT
  SUM(CASE WHEN status = 'PAID'   THEN 1 ELSE 0 END) AS cnt_paid,
  SUM(CASE WHEN status = 'REFUND' THEN 1 ELSE 0 END) AS cnt_refund,
  SUM(amount) AS amt_total
FROM Orders;
```

---

## 5. 다중 컬럼·표현식 그룹핑 & 날짜 버킷팅

### 5.1 다중 컬럼 그룹핑
```sql
-- 부서+직무 별 인원
SELECT dept_id, job, COUNT(*) AS emp_count
FROM Employee
GROUP BY dept_id, job;
```

### 5.2 날짜 버킷팅(일/주/월)

> **권장 패턴**: “**반열림 구간**” 범위 조건 + **원본 컬럼**으로 그룹 키를 만들어 **인덱스 타기**.

**일별**
```sql
-- MySQL
SELECT DATE(order_dt) AS d, COUNT(*) AS cnt
FROM Orders
WHERE order_dt >= '2025-10-01'
  AND order_dt <  '2025-11-01'
GROUP BY DATE(order_dt)
ORDER BY d;
```

**월별(반열림)**
```sql
-- SQL Server (반열림 구간)
SELECT
  FORMAT(order_dt, 'yyyy-MM') AS ym,  -- 주의: FORMAT은 느림(보고서 한정)
  COUNT(*) AS cnt
FROM dbo.Orders
WHERE order_dt >= DATEFROMPARTS(2025, 10, 01)
  AND order_dt <  DATEFROMPARTS(2025, 11, 01)
GROUP BY FORMAT(order_dt, 'yyyy-MM')
ORDER BY ym;
```

> **고성능 대안**: 계산 열(예: `ym_first` = 월 첫날) + 인덱스, 또는 미리 **달력 차원 테이블**을 조인(데이터 웨어하우스 표준).

---

## 6. 고급 집계 — ROLLUP, CUBE, GROUPING SETS

### 6.1 ROLLUP (계층 소계 + 총계)
```sql
-- 부서/직무별 인원 + 부서 소계 + 전체 합계
SELECT dept_id, job, COUNT(*) AS emp_count
FROM Employee
GROUP BY ROLLUP (dept_id, job);
```
- (dept_id, job) → (dept_id, NULL) → (NULL, NULL) 순으로 소계/합계 행이 추가.

### 6.2 CUBE (모든 조합의 소계)
```sql
-- (부서), (직무), (부서+직무), (전체) 소계/합계 모두
SELECT dept_id, job, COUNT(*) AS emp_count
FROM Employee
GROUP BY CUBE (dept_id, job);
```

### 6.3 GROUPING SETS (원하는 조합만)
```sql
-- 부서 소계와 전체 합계만 원할 때
SELECT dept_id, COUNT(*) AS emp_count
FROM Employee
GROUP BY GROUPING SETS ((dept_id), ());
```

### 6.4 GROUPING / GROUPING_ID로 “소계 행” 식별
```sql
-- SQL Server/PostgreSQL/Oracle: GROUPING, GROUPING_ID 지원
SELECT
  dept_id, job,
  COUNT(*) AS emp_count,
  GROUPING(dept_id) AS g_dept,
  GROUPING(job)     AS g_job,
  GROUPING_ID(dept_id, job) AS gid
FROM Employee
GROUP BY ROLLUP (dept_id, job)
ORDER BY dept_id, job;
```

- `GROUPING(col)=1` 이면 그 컬럼은 **소계/합계 레벨**에서 **NULL을 의미하는 자리 표시자**.

---

## 7. 집계 vs 윈도 함수 — 언제 무엇을 쓰나

| 요구 | 사용 |
|---|---|
| **그룹당 1행**으로 요약(부서별 합계 등) | `GROUP BY` + 집계 |
| 요약값을 **행별로** 보존(누계/이동합/그룹 내 순위) | **윈도 함수** `SUM() OVER(...)`, `ROW_NUMBER()` |
| 그룹 요약 + 원본 행 동시 표시 | 서브쿼리/CTE 또는 윈도 집계 |

```sql
-- 윈도: 카테고리별 누계
SELECT
  category,
  product,
  amount,
  SUM(amount) OVER (PARTITION BY category ORDER BY amount DESC) AS cum_amt
FROM sales_by_product;
```

---

## 8. 성능 최적화 핵심

### 8.1 WHERE로 먼저 줄여라
- `HAVING` 대신 **가능한 조건은 `WHERE`**로 내려 **행 수 줄이기**.

```sql
-- 비권장: HAVING에 행 조건(날짜) 넣으면 전체 스캔 후 그룹
SELECT customer_id, SUM(amount) AS s
FROM Orders
GROUP BY customer_id
HAVING MIN(order_dt) >= '2025-10-01';

-- 권장: WHERE로 먼저 필터
SELECT customer_id, SUM(amount) AS s
FROM Orders
WHERE order_dt >= '2025-10-01'
GROUP BY customer_id;
```

### 8.2 인덱스와 집계
- **그룹 키(=GROUP BY 컬럼)**에 **인덱스**가 있으면 **정렬/해시 작업 감소** 가능.
- MySQL InnoDB는 **인덱스 순회 + 조기 집계**가 이득인 경우가 많다(핵심: **선택도/카디널리티**).

```sql
-- 예: (customer_id, order_dt) 복합 인덱스는 고객/기간 집계에 유리
CREATE INDEX ix_orders_customer_dt ON Orders (customer_id, order_dt);
```

### 8.3 DISTINCT vs GROUP BY 비용
- “고유 목록”만 필요하면 `SELECT DISTINCT`가 명시적.
- 다만 인덱스 유무/옵티마이저 계획에 따라 비용 차이가 있으므로 **실행계획 확인**.

### 8.4 계산 열/함수 기반 인덱스
- 날짜 버킷팅 키(월/주/일)와 같이 **표현식 그룹핑**이 잦다면,
  **계산 열(또는 생성 열) + 인덱스**로 **SARGable**하게.

```sql
-- SQL Server: 월첫날 계산 열 + 인덱스
ALTER TABLE dbo.Orders
ADD ym_first AS DATEFROMPARTS(YEAR(order_dt), MONTH(order_dt), 1) PERSISTED;
CREATE INDEX ix_orders_ym_first ON dbo.Orders(ym_first);
```

---

## 9. 실전 시나리오 10선

### 9.1 부서별 평균 급여 + 평균 300 이상만 (기본기)
```sql
SELECT dept_id, AVG(salary) AS avg_salary
FROM Employee
GROUP BY dept_id
HAVING AVG(salary) >= 300
ORDER BY avg_salary DESC;
```

### 9.2 최근 6개월, 고객별 총액 500 이상 (반열림 + 재필터)
```sql
-- MySQL
SELECT customer_id, SUM(total_amount) AS total
FROM Orders
WHERE order_date >= DATE_SUB(DATE_FORMAT(CURDATE(), '%Y-%m-01'), INTERVAL 6 MONTH)
  AND order_date <  DATE_FORMAT(CURDATE(), '%Y-%m-01')
GROUP BY customer_id
HAVING SUM(total_amount) >= 500
ORDER BY total DESC;
```

### 9.3 월별/상태별 매출(조건부 집계) + 월 버킷팅
```sql
-- SQL Server
WITH base AS (
  SELECT
    order_dt,
    status,
    amount,
    DATEFROMPARTS(YEAR(order_dt), MONTH(order_dt), 1) AS ym_first
  FROM dbo.Orders
  WHERE order_dt >= DATEADD(MONTH, -12, DATEFROMPARTS(YEAR(GETDATE()), MONTH(GETDATE()), 1))
)
SELECT
  CONVERT(char(7), ym_first, 126) AS ym,
  SUM(CASE WHEN status='PAID'   THEN amount ELSE 0 END) AS amt_paid,
  SUM(CASE WHEN status='REFUND' THEN amount ELSE 0 END) AS amt_refund,
  SUM(amount) AS amt_total
FROM base
GROUP BY ym_first
ORDER BY ym_first;
```

### 9.4 상품·지역 이중 축 집계 + ROLLUP (소계/합계)
```sql
SELECT region, product, SUM(amount) AS amt
FROM Sales
GROUP BY ROLLUP (region, product)
ORDER BY region, product;
```

### 9.5 원하는 소계만 — GROUPING SETS
```sql
-- 상품 소계 + 지역 소계 + 전체 합계
SELECT
  region,
  product,
  SUM(amount) AS amt
FROM Sales
GROUP BY GROUPING SETS (
  (region, product), -- 상세
  (region),          -- 지역 소계
  (product),         -- 상품 소계
  ()                 -- 전체 합계
);
```

### 9.6 DAU/WAU/MAU(일/주/월 활성 사용자) — DISTINCT + 버킷팅
```sql
-- MySQL 예시: 월별 활성 사용자
SELECT DATE_FORMAT(event_dt, '%Y-%m') AS ym,
       COUNT(DISTINCT user_id) AS mau
FROM user_events
WHERE event_dt >= DATE_SUB(CURDATE(), INTERVAL 12 MONTH)
GROUP BY DATE_FORMAT(event_dt, '%Y-%m')
ORDER BY ym;
```

### 9.7 반품 비율 — 안전 나눗셈
```sql
-- 반품수/전체주문수
SELECT
  100.0 * SUM(CASE WHEN status='REFUND' THEN 1 ELSE 0 END)
         / NULLIF(COUNT(*), 0) AS refund_rate_pct
FROM Orders;
```

### 9.8 키셋 페이징용 그룹 집계(Stable sort)
```sql
-- 월별 합계를 구하고, 마지막 ym 이후 다음 페이지 가져오기
SELECT ym, amt
FROM monthly_sales
WHERE ym < :last_ym
ORDER BY ym DESC
LIMIT 50;
```

### 9.9 JSON 속성별 집계(가상/계산 열)
```sql
-- MySQL: 생성 열 + 인덱스
ALTER TABLE events
  ADD category VARCHAR(20)
    GENERATED ALWAYS AS (JSON_UNQUOTE(JSON_EXTRACT(payload, '$.category'))) STORED,
  ADD INDEX ix_events_category (category);

SELECT category, COUNT(*) AS cnt
FROM events
GROUP BY category
HAVING COUNT(*) >= 100;
```

### 9.10 고액 고객 상위 3등급만 — GROUP BY + 서브쿼리
```sql
-- 고객별 총액 산출 후 상위 등급 필터
WITH agg AS (
  SELECT customer_id, SUM(amount) AS total_amt
  FROM Orders
  WHERE order_dt >= '2025-01-01'
  GROUP BY customer_id
)
SELECT *
FROM agg
WHERE total_amt >= 10000; -- 예: VIP 기준
```

---

## 10. 다이얼렉트 차이 요약

| 항목 | MySQL | SQL Server | PostgreSQL | Oracle |
|---|---|---|---|---|
| ONLY_FULL_GROUP_BY | 모드로 제어 | 표준 준수 | 표준 준수 | 표준 준수 |
| 날짜 버킷 | `DATE_FORMAT`, 함수 기반 인덱스 | 계산 열(PERSISTED) | `date_trunc('month', ts)` | `TRUNC(dt, 'MM')` |
| 다중 DISTINCT | `COUNT(DISTINCT a,b)` 지원 | 서브쿼리 필요 | 서브쿼리/표현식 | 서브쿼리/표현식 |
| ROLLUP/CUBE/GROUPING SETS | 지원(8.0+) | 전부 지원 | 전부 지원 | 전부 지원 |
| GROUPING_ID | 일부제공(버전 주의) | 지원 | 지원 | 지원 |

> **PostgreSQL**의 `date_trunc`, **Oracle**의 `TRUNC(date, 'MM')`는 날짜 버킷팅이 간결하고 빠르다.
> **MySQL/SQL Server**는 **계산(생성) 열 + 인덱스** 패턴을 적극 활용.

---

## 11. 체크리스트

- [ ] **비집계 컬럼 = 전부 GROUP BY** (표준 준수)
- [ ] 행 필터는 **WHERE**, 그룹 필터는 **HAVING**
- [ ] **조건부 집계**: `SUM(CASE WHEN ...)` 패턴 숙지
- [ ] **NULL 의미** 숙지: `COUNT(*)` vs `COUNT(col)`
- [ ] 날짜 집계는 **반열림 구간** + **계산 열/함수 인덱스**
- [ ] 다차원 소계는 **ROLLUP/CUBE/GROUPING SETS**
- [ ] **윈도 함수**는 “행별 누계/순위/이동합”, GROUP BY는 “그룹당 1행”
- [ ] **WHERE로 먼저 줄이고** HAVING으로 재필터
- [ ] **인덱스**: 그룹 키/프리카드 고려, 필요 시 함수 기반/계산 열
- [ ] 실행계획으로 **정렬/해시/스트림 집계** 비용 확인

---

## 부록) 추가 예제 모음

### A. 집계 결과에 별칭 사용 (ORDER BY, HAVING)
```sql
-- 대부분 DB에서 SELECT 별칭은 ORDER BY에서 사용 가능
SELECT dept_id, AVG(salary) AS avg_salary
FROM Employee
GROUP BY dept_id
HAVING AVG(salary) > 500
ORDER BY avg_salary DESC;
```

### B. 조건부 평균(0 나눗셈 방지)
```sql
SELECT
  1.0 * SUM(CASE WHEN status='PAID' THEN amount ELSE 0 END)
  / NULLIF(SUM(CASE WHEN status='PAID' THEN 1 ELSE 0 END), 0) AS avg_paid_amount
FROM Orders;
```

### C. 상위 N 그룹 (서브쿼리로 두 단계)
```sql
-- 월별 합계 중 상위 5개월
WITH m AS (
  SELECT DATE_FORMAT(order_dt, '%Y-%m') AS ym, SUM(amount) AS amt
  FROM Orders
  GROUP BY DATE_FORMAT(order_dt, '%Y-%m')
)
SELECT *
FROM m
ORDER BY amt DESC
LIMIT 5;
```

### D. 근사 고유 수(빅데이터)
- **MySQL**: HyperLogLog 내장 없음 → 앱/ETL에서 계산 또는 ClickHouse/Redis HLL 활용
- **PostgreSQL/BigQuery/ClickHouse**: `approx_count_distinct`류 제공
- 대용량 N:M 고유 집계는 **近似**를 고려(대시보드·랭킹 등).

---

# 결론

- `GROUP BY`/`HAVING`은 **정확한 통계·대시보드·요약 리포트**의 중심이다.
- **WHERE로 행을 줄이고 → GROUP BY로 요약 → HAVING으로 재필터**가 성능 골든루트.
- 날짜 버킷팅·조건부 집계·다중 축 소계(ROLLUP/CUBE/GROUPING SETS)·윈도 함수와의 경계까지 익히면 **대부분의 분석성 쿼리**를 고성능으로 구현할 수 있다.
- 마지막으로, **인덱스/계산 열/함수 기반 인덱스**와 **실행계획 확인**을 습관화하라.
  그 한 줄이 **수십 배 성능 차이**를 만든다.
