---
layout: post
title: DB - 함수(Function)
date: 2025-02-21 20:20:23 +0900
category: DB
---
# SQL 함수(Function)

## 함수 분류와 3값 논리

### 함수 분류(재정의)

| 분류 | 설명 | 예시 |
|---|---|---|
| 단일 행 함수(Scalar) | 행마다 **하나의 값**을 반환 | `LOWER(email)`, `ROUND(price, 2)` |
| 집계 함수(Aggregate) | 여러 행 → **단일 값** 집계 | `SUM(amount)`, `AVG(score)` |
| 윈도 함수(Analytic) | 여러 행 → **행별** 집계/순위/누계 | `SUM(amount) OVER(...)`, `ROW_NUMBER()` |
| JSON/형변환/기타 | JSON·형변환·보안·유틸 | `JSON_EXTRACT`, `TRY_CONVERT`, `HASHBYTES` |

> 윈도 함수는 **GROUP BY로는 표현하기 어려운 “행별 집계”**를 제공한다.

### SQL의 3값 논리(Three-Valued Logic)

- 비교 결과는 `TRUE / FALSE / UNKNOWN(NULL)` 중 하나.
- `NULL = NULL`은 `TRUE`가 아니라 **UNKNOWN**.
- WHERE 절에서 `UNKNOWN`은 필터링되어 **제외**된다.
- NULL 비교는 **`IS NULL`/`IS NOT NULL`**을 사용.

---

## 문자열 함수 — MySQL vs SQL Server

### 길이와 바이트: `LENGTH` vs `CHAR_LENGTH` vs `LEN` vs `DATALENGTH`

| DB | 문자 길이 | 바이트 길이 | 주의 |
|---|---|---|---|
| MySQL | `CHAR_LENGTH(s)` | `LENGTH(s)` | UTF8에서 한글은 3바이트(utf8mb4 3~4) |
| SQL Server | `LEN(s)` | `DATALENGTH(s)` | `LEN`은 **후행 공백 미포함** |

```sql
-- MySQL
SELECT CHAR_LENGTH('가'), LENGTH('가');  -- 1, 3 (utf8) 또는 4(utf8mb4)
-- SQL Server
SELECT LEN(N'abc  '), DATALENGTH(N'abc  ');  -- 3, 10 (NVARCHAR는 2바이트/문자)
```

### 대/소문자, 트림/패딩/치환

| 기능 | MySQL | SQL Server | 예시 |
|---|---|---|---|
| 대문자 | `UPPER(s)` | `UPPER(s)` | `UPPER('kim') → 'KIM'` |
| 소문자 | `LOWER(s)` | `LOWER(s)` |  |
| 트림 | `TRIM([LEADING|TRAILING|BOTH] x FROM s)` / `TRIM(s)` | `LTRIM`/`RTRIM`/`TRIM`(2017+) |  |
| 패딩 | `LPAD(s,n,p)`/`RPAD(s,n,p)` | 직접 조합(STRING_AGG/REPLICATE) |  |
| 치환 | `REPLACE(s,from,to)` | `REPLACE(s,from,to)` |  |

```sql
-- MySQL: 공백/특정 문자 제거
SELECT TRIM(BOTH '/' FROM '/a/b/');  -- 'a/b'
-- SQL Server: 공백 제거
SELECT TRIM('  abc  '), LTRIM(RTRIM('  abc  '));  -- 'abc'
```

### 부분 문자열, 검색, 위치

| 기능 | MySQL | SQL Server |
|---|---|---|
| 부분 | `SUBSTRING(s,pos,len)` | `SUBSTRING(s,pos,len)` |
| 찾기 | `INSTR(s,substr)` / `LOCATE(substr, s)` | `CHARINDEX(substr, s)` |
| 분할 | `SUBSTRING_INDEX(s, delim, count)` | `STRING_SPLIT(s, delim)`(2016+, 옵션) |

```sql
-- MySQL: 2번째 슬래시까지 왼쪽
SELECT SUBSTRING_INDEX('a/b/c', '/', 2);  -- 'a/b'
-- SQL Server: 분할 후 집계
SELECT value FROM STRING_SPLIT('a,b,c', ',');
```

### 패턴/정규식

- MySQL: `LIKE`(전방 고정 인덱스 가능), `REGEXP_LIKE/REGEXP_REPLACE`(8.0+).
- SQL Server: 정규식 내장 X → CLR/외부/LIKE 조합, 또는 `TRANSLATE`, `REPLACE` 조합.

> **성능**: `%abc`(전방 와일드)는 인덱스 사용 곤란. **전방 고정** `abc%`는 유리.

---

## 숫자/수학 함수 — 반올림·절삭·안전 계산

| 기능 | MySQL | SQL Server | 메모 |
|---|---|---|---|
| 반올림 | `ROUND(x, n)` | `ROUND(x, n)` | SQL Server `ROUND(x, n, 1)` = **내림** 모드 |
| 절삭 | `TRUNCATE(x, n)` | 없지만 `ROUND(x, n, 1)`/수식 |  |
| 올림/내림 | `CEIL/CEILING`, `FLOOR` | `CEILING`, `FLOOR` |  |
| 나머지 | `MOD(x,y)`/`x % y` | `%` |  |
| 절댓값 | `ABS(x)` | `ABS(x)` |  |

```sql
-- MySQL: 소수 1자리 절삭
SELECT TRUNCATE(123.456, 1);  -- 123.4

-- SQL Server: 내림 대체
SELECT ROUND(123.456, 1, 1);  -- 123.4
```

**안전 나눗셈(0 나누기 방지)**

```sql
-- MySQL
SELECT IFNULL(numerator / NULLIF(denominator, 0), 0);
-- SQL Server
SELECT ISNULL(1.0 * numerator / NULLIF(denominator, 0), 0);
```

부동소수 오차를 줄이려면 **DECIMAL** 사용(가격/금액).

---

## 날짜/시간 함수 — 경계, 타임존, 리포트

| 기능 | MySQL | SQL Server |
|---|---|---|
| 현재 | `NOW()` | `GETDATE()` |
| 날짜만 | `CURDATE()`/`DATE(dt)` | `CAST(GETDATE() AS DATE)` |
| 더하기 | `DATE_ADD(d, INTERVAL n X)` | `DATEADD(X, n, d)` |
| 차이 | `TIMESTAMPDIFF(unit, d1,d2)`/`DATEDIFF(d1,d2)`(MySQL은 d1-d2) | `DATEDIFF(unit, d1,d2)`(경계 카운트 방식) |
| 포맷 | `DATE_FORMAT(dt, fmt)` | `FORMAT(dt, fmt, culture)`(주의: 느림) |

> **경계**: **반열림 구간** `>= start AND < next_start` 권장(중복/누락 방지).

```sql
-- 최근 월별 합계 (MySQL)
SELECT DATE_FORMAT(order_dt, '%Y-%m') AS ym, SUM(amount)
FROM orders
WHERE order_dt >= DATE_SUB(CURDATE(), INTERVAL 12 MONTH)
GROUP BY ym
ORDER BY ym;

-- 월 경계(반열림, SQL Server)
WHERE order_dt >= DATEFROMPARTS(2025, 01, 01)
  AND order_dt <  DATEFROMPARTS(2025, 02, 01);
```

**윈도: 이동 7일 합(후술 6절 참고)**

---

## NULL/조건/집계 — 조건부 집계의 정석

### NULL 처리를 통일

| 목적 | MySQL | SQL Server |
|---|---|---|
| 첫 번째 비NULL | `COALESCE(a,b,...)` | `COALESCE(a,b,...)` |
| 단항 대체 | `IFNULL(expr, alt)` | `ISNULL(expr, alt)` |
| 안전 나눗셈 | `NULLIF(den,0)` | `NULLIF(den,0)` |

> **집계**: `COUNT(col)`은 NULL 제외, `COUNT(*)`는 NULL 포함.

### 조건부 집계(보고서 핵심)

```sql
-- 월별 상태별 건수(양 테이블 동일)
SELECT
  DATE_FORMAT(order_dt, '%Y-%m') AS ym,
  SUM(CASE WHEN status = 'PAID'    THEN 1 ELSE 0 END) AS cnt_paid,
  SUM(CASE WHEN status = 'REFUND'  THEN 1 ELSE 0 END) AS cnt_refund,
  COUNT(*) AS cnt_total
FROM orders
GROUP BY ym
ORDER BY ym;

-- SQL Server 날짜 포맷(권장: 변환 대신 그룹 키 별도)
SELECT
  CONVERT(char(7), order_dt, 126) AS ym,  -- 'YYYY-MM'
  SUM(CASE WHEN status = 'PAID'   THEN 1 ELSE 0 END) AS cnt_paid,
  SUM(CASE WHEN status = 'REFUND' THEN 1 ELSE 0 END) AS cnt_refund,
  COUNT(*) AS cnt_total
FROM dbo.orders
GROUP BY CONVERT(char(7), order_dt, 126)
ORDER BY ym;
```

### 안전 평균/비율

```sql
-- 비율 = 분자 / 분모, 0 나눗셈 방지
SELECT
  100.0 * SUM(CASE WHEN status='PAID' THEN 1 ELSE 0 END)
  / NULLIF(COUNT(*), 0) AS rate_paid
FROM orders;
```

---

## 윈도 함수 — 누계/이동 집계/그룹 Top-N

> MySQL 8.0+, SQL Server 2012+ 지원

### 누계/이동합

```sql
-- MySQL: 최근 90일, 일자별 매출과 이동 7일 합
SELECT
  sale_date,
  SUM(amount) AS daily_amount,
  SUM(SUM(amount)) OVER (
    ORDER BY sale_date
    ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
  ) AS rolling_7d_amount
FROM sales
WHERE sale_date >= CURDATE() - INTERVAL 90 DAY
GROUP BY sale_date
ORDER BY sale_date;

-- SQL Server: 동일
SELECT
  sale_date,
  SUM(amount) AS daily_amount,
  SUM(SUM(amount)) OVER (
    ORDER BY sale_date
    ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
  ) AS rolling_7d_amount
FROM dbo.sales
WHERE sale_date >= DATEADD(DAY, -90, CAST(GETDATE() AS DATE))
GROUP BY sale_date
ORDER BY sale_date;
```

### 파티션별 순위 / Top-N per group

```sql
-- 카테고리별 매출 Top 3 상품
SELECT *
FROM (
  SELECT
    category_id, product_id, SUM(amount) AS amt,
    ROW_NUMBER() OVER (PARTITION BY category_id ORDER BY SUM(amount) DESC) AS rn
  FROM order_items
  GROUP BY category_id, product_id
) t
WHERE rn <= 3
ORDER BY category_id, rn;
```

### 누적/비율

```sql
-- 카테고리 내 누적 점유율
SELECT
  category_id, product_id, amt,
  SUM(amt) OVER (PARTITION BY category_id ORDER BY amt DESC) AS cum_amt,
  SUM(amt) OVER (PARTITION BY category_id) AS total_amt,
  1.0 * SUM(amt) OVER (PARTITION BY category_id ORDER BY amt DESC)
      / NULLIF(SUM(amt) OVER (PARTITION BY category_id),0) AS cum_ratio
FROM (
  SELECT category_id, product_id, SUM(amount) AS amt
  FROM order_items
  GROUP BY category_id, product_id
) s
ORDER BY category_id, cum_amt DESC;
```

---

## 형변환·타입 검사 — 안전 변환과 성능

| 기능 | MySQL | SQL Server | 메모 |
|---|---|---|---|
| 캐스팅 | `CAST(x AS TYPE)`/`CONVERT(TYPE, x)` | `CAST(x AS TYPE)`/`CONVERT(TYPE, x)` |  |
| 실패-무해 변환 | `CAST` 실패 시 에러(엄격 모드) | `TRY_CONVERT(TYPE, x)`/`TRY_PARSE` | 실패 시 NULL |
| 숫자화 | `CAST(JSON_UNQUOTE(...) AS DECIMAL)` | `TRY_CONVERT(NUMERIC, JSON_VALUE(...))` |  |

```sql
-- SQL Server: TRY_CONVERT로 실패 시 NULL
SELECT TRY_CONVERT(INT, '123'), TRY_CONVERT(INT, '12A');  -- 123, NULL
```

---

## JSON 함수 — 반정규화/스키마 유연성 대처

### MySQL

- 추출: `JSON_EXTRACT(doc, '$.a.b')` 또는 `doc->'$.a.b'`
- 스칼라화: `JSON_UNQUOTE(JSON_EXTRACT(...))`
- 수정: `JSON_SET`, `JSON_REPLACE`, `JSON_REMOVE`
- 검색: **가급적 “가상 컬럼 + 인덱스”**(8.0+)로 SARGable

```sql
-- 가상 컬럼 + 함수기반 인덱스
ALTER TABLE events
  ADD item_id VARCHAR(32) GENERATED ALWAYS AS (JSON_UNQUOTE(JSON_EXTRACT(payload, '$.itemId'))) STORED,
  ADD INDEX ix_events_item_id (item_id);

SELECT * FROM events WHERE item_id = 'A001';
```

### SQL Server

- 추출: `JSON_VALUE(doc, '$.a')`(스칼라) / `JSON_QUERY(doc, '$.arr')`(JSON)
- 펼치기: `OPENJSON(doc)` (CROSS APPLY)
- 인덱스: **계산 열(PERSISTED) + 인덱스** 권장

```sql
-- 계산 열 + 인덱스
ALTER TABLE dbo.events
ADD item_id AS JSON_VALUE(payload, '$.itemId') PERSISTED;

CREATE INDEX ix_events_item_id ON dbo.events(item_id);

SELECT * FROM dbo.events WHERE item_id = 'A001';
```

---

## 보안·암호화·마스킹(간단 정리)

- MySQL: `AES_ENCRYPT/AES_DECRYPT`(주의: 키 관리), `SHA2`, `MD5`(비권장).
- SQL Server: `HASHBYTES`, `ENCRYPTBYKEY/DECRYPTBYKEY`, 동적 데이터 마스킹(Dynamic Data Masking), Always Encrypted.
- **권한/키 관리/컴플라이언스**는 DB 함수보다 **플랫폼 설계** 이슈임을 명시.

---

## 인덱스와 함수 — SARGability 핵심

### 함수가 인덱스를 막는 경우

- **컬럼을 함수/연산으로 감싸면** B-tree 인덱스 활용이 어려워짐.

```sql
-- 비권장: 인덱스 무력화 가능
WHERE DATE(order_dt) = CURDATE();        -- MySQL
WHERE CAST(order_dt AS DATE) = CAST(GETDATE() AS DATE); -- SQL Server

-- 권장: 반열림 구간
WHERE order_dt >= CURDATE() AND order_dt < CURDATE() + INTERVAL 1 DAY;
WHERE order_dt >= CAST(GETDATE() AS DATE) AND order_dt < DATEADD(DAY, 1, CAST(GETDATE() AS DATE));
```

### 함수 기반 인덱스 / 계산 열

- **MySQL 8.0**: 함수 기반 인덱스 지원.

```sql
CREATE INDEX ix_user_lower_email ON users ((LOWER(email)));
SELECT * FROM users WHERE LOWER(email) = 'alpha@example.com';
```

- **SQL Server**: 계산 열 + PERSISTED + 인덱스.

```sql
ALTER TABLE dbo.users ADD email_lower AS LOWER(email) PERSISTED;
CREATE INDEX ix_users_email_lower ON dbo.users(email_lower);
```

> **원칙**: “함수는 인덱스로 끌어내리고, WHERE는 컬럼 원형 비교”가 SARGable.

---

## UDF(사용자 정의 함수) — 성능·보안·가이드

### 유형

| DB | 스칼라 UDF | 테이블 반환 UDF |
|---|---|---|
| MySQL | Stored Function | (주로 Stored Procedure/뷰/테이블로 대체) |
| SQL Server | `CREATE FUNCTION ... RETURNS scalar` | Inline TVF(권장)/Multi-Statement TVF(주의) |

### 성능

- SQL Server **스칼라 UDF**는 **행 단위 호출**로 느릴 수 있음.
  2019+의 **UDF 인라이닝**(제약 조건하에)으로 대폭 개선 가능.
- **Inline TVF**는 옵티마이저가 조인/필터링 최적화에 유리 → **선호**.
- MySQL Stored Function은 트랜잭션/결정성/보안 고려 필요.

### 결정성/인덱스

- 계산 열 인덱스(PERSISTED)에는 **결정적(Deterministic)** 함수만 허용(SQL Server).
- MySQL 함수 기반 인덱스는 표현식이 인덱싱 가능해야 한다.

### 보안/권한

- 권한 에스컬레이션, 임의 파일/네트워크 접근 금지(서버 설정).
- SQL Server는 `SCHEMABINDING`/권한 최소화 원칙.

---

## 실전 시나리오 8종

### “Gmail 사용자만, 이메일 소문자 매칭 + 인덱스”

```sql
-- MySQL: 가상 컬럼 + 인덱스(성능)
ALTER TABLE users
  ADD email_lower VARCHAR(255) GENERATED ALWAYS AS (LOWER(email)) STORED,
  ADD INDEX ix_users_email_lower (email_lower);

SELECT id, email
FROM users
WHERE email_lower LIKE '%@gmail.com';

-- SQL Server: 계산 열 + 인덱스
ALTER TABLE dbo.users
ADD email_lower AS LOWER(email) PERSISTED;
CREATE INDEX ix_users_email_lower ON dbo.users(email_lower);

SELECT id, email
FROM dbo.users
WHERE email_lower LIKE '%@gmail.com';
```

### “월별·상태별 매출 리포트 — 조건부 집계 + 날짜 경계”

```sql
-- MySQL
SELECT DATE_FORMAT(order_dt, '%Y-%m') AS ym,
       SUM(CASE WHEN status='PAID'   THEN amount ELSE 0 END) AS amt_paid,
       SUM(CASE WHEN status='REFUND' THEN amount ELSE 0 END) AS amt_refund,
       SUM(amount) AS amt_total
FROM orders
WHERE order_dt >= DATE_SUB(DATE_FORMAT(CURDATE(), '%Y-%m-01'), INTERVAL 11 MONTH)
GROUP BY ym
ORDER BY ym;

-- SQL Server
WITH base AS (
  SELECT
    CAST(EOMONTH(order_dt, 0) - (DAY(EOMONTH(order_dt, 0)) - 1) AS DATE) AS ym_first, -- 월 1일
    status, amount
  FROM dbo.orders
  WHERE order_dt >= DATEADD(MONTH, -11, DATEFROMPARTS(YEAR(GETDATE()), MONTH(GETDATE()), 1))
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

### “이동 7일 합(윈도) + 주말 제외(조건부)”

```sql
-- MySQL: 주말 제외는 CASE로 0 처리
SELECT dt,
       SUM(CASE WHEN DAYOFWEEK(dt) IN (1,7) THEN 0 ELSE amount END) AS amt_wd,
       SUM(SUM(CASE WHEN DAYOFWEEK(dt) IN (1,7) THEN 0 ELSE amount END))
       OVER (ORDER BY dt ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS rolling_7d
FROM daily_sales
WHERE dt >= CURDATE() - INTERVAL 60 DAY
GROUP BY dt
ORDER BY dt;

-- SQL Server: DATEPART(WEEKDAY, ...)
SELECT dt,
       SUM(CASE WHEN DATEPART(WEEKDAY, dt) IN (1,7) THEN 0 ELSE amount END) AS amt_wd,
       SUM(SUM(CASE WHEN DATEPART(WEEKDAY, dt) IN (1,7) THEN 0 ELSE amount END))
       OVER (ORDER BY dt ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS rolling_7d
FROM dbo.daily_sales
WHERE dt >= DATEADD(DAY, -60, CAST(GETDATE() AS DATE))
GROUP BY dt
ORDER BY dt;
```

### “그룹별 Top-N — ROW_NUMBER”

```sql
-- 카테고리별 판매 상위 3개 상품
WITH ranker AS (
  SELECT category_id, product_id, SUM(amount) AS amt,
         ROW_NUMBER() OVER (PARTITION BY category_id ORDER BY SUM(amount) DESC) AS rn
  FROM order_items
  GROUP BY category_id, product_id
)
SELECT *
FROM ranker
WHERE rn <= 3
ORDER BY category_id, rn;
```

### “JSON 속성 인덱싱 — 가상/계산 열”

(8절 예시 참고 — 반복 생략)

### “안전한 평균단가 — 0 나눗셈 방지 + 반올림”

```sql
-- MySQL
SELECT product_id,
       ROUND(SUM(amount) / NULLIF(SUM(qty), 0), 2) AS avg_price
FROM order_items
GROUP BY product_id;

-- SQL Server
SELECT product_id,
       ROUND(1.0 * SUM(amount) / NULLIF(SUM(qty), 0), 2) AS avg_price
FROM dbo.order_items
GROUP BY product_id;
```

### “정규식 치환(이메일 도메인 표준화) — MySQL”

```sql
-- MySQL 8.0: REGEXP_REPLACE
UPDATE users
SET email = CONCAT(SUBSTRING_INDEX(email, '@', 1), '@', LOWER(SUBSTRING_INDEX(email, '@', -1)));
```

(SQL Server는 정규식 미지원 → 분해/REPLACE로 조합 또는 CLR)

### “기간 페이징 — 키셋 페이징”

```sql
-- 마지막 (dt, id)을 알고 있을 때 다음 페이지
SELECT id, dt, amount
FROM payments
WHERE (dt, id) < ( :last_dt, :last_id )
ORDER BY dt DESC, id DESC
LIMIT 100;

-- SQL Server는 튜플 비교 대신 논리식
SELECT TOP (100) id, dt, amount
FROM dbo.payments
WHERE (dt < @last_dt) OR (dt = @last_dt AND id < @last_id)
ORDER BY dt DESC, id DESC;
```

---

## 체크리스트(요약·실무 가드)

- **NULL**: `= NULL` 금지 → `IS NULL`. `NOT IN` + NULL 함정 피하고 `NOT EXISTS` 사용.
- **SARGability**: 컬럼을 함수/연산으로 감싸지 말고, **함수는 인덱스로 끌어내리기**(계산 열/함수 인덱스).
- **날짜 경계**: **반열림 구간** `>= start AND < next` 습관화.
- **문자 길이**: SQL Server `LEN`은 후행 공백 미포함, `DATALENGTH`는 바이트.
- **안전 나눗셈**: `NULLIF(den,0)`로 0 나눗셈 방지.
- **조건부 집계**: `SUM(CASE WHEN ... THEN 1 ELSE 0 END)` 패턴 정석.
- **윈도 함수**: 이동합/누계/Top-N per group 문제를 GROUP BY 없이 해결.
- **JSON**: 추출 후 스칼라화·가상/계산 열 + 인덱스.
- **UDF**: SQL Server는 **Inline TVF 우선**, 스칼라 UDF는 2019+ 인라이닝 활용.
- **성능 모니터링**: 실행계획/통계 최신화, 리라이트(OR → UNION ALL, NOT → 긍정 범위).

---

## 부록 A) 빠른 참조 테이블

### A.1 문자열 핵심

| 목적 | MySQL | SQL Server |
|---|---|---|
| 대/소문자 | `LOWER/UPPER` | `LOWER/UPPER` |
| 길이(문자/바이트) | `CHAR_LENGTH` / `LENGTH` | `LEN` / `DATALENGTH` |
| 공백 제거 | `TRIM` | `TRIM`(2017+)/`LTRIM`/`RTRIM` |
| 분할/위치 | `SUBSTRING_INDEX`/`LOCATE` | `STRING_SPLIT`/`CHARINDEX` |
| 정규식 | `REGEXP_LIKE/REPLACE` | (내장 X, CLR/LIKE 조합) |

### A.2 날짜 핵심

| 목적 | MySQL | SQL Server |
|---|---|---|
| 현재 | `NOW()` | `GETDATE()` |
| 더하기 | `DATE_ADD(dt, INTERVAL n X)` | `DATEADD(X, n, dt)` |
| 차이 | `TIMESTAMPDIFF(...)` | `DATEDIFF(unit, d1,d2)` |
| 포맷 | `DATE_FORMAT` | `FORMAT`(주의: 느림) |

### A.3 안전 계산

```sql
-- MySQL
SELECT IFNULL(num / NULLIF(den,0), 0);

-- SQL Server
SELECT ISNULL(1.0 * num / NULLIF(den,0), 0);
```

---

## 부록 B) 샘플 스키마(공통)

```sql
-- MySQL
CREATE TABLE orders (
  id BIGINT PRIMARY KEY,
  customer_id BIGINT NOT NULL,
  order_dt DATETIME NOT NULL,
  status ENUM('PENDING','PAID','SHIPPED','REFUND','CANCELLED') NOT NULL,
  amount DECIMAL(12,2) NOT NULL
);

CREATE TABLE order_items (
  order_id BIGINT NOT NULL,
  product_id BIGINT NOT NULL,
  qty INT NOT NULL,
  price DECIMAL(12,2) NOT NULL,
  PRIMARY KEY(order_id, product_id)
);

-- SQL Server
CREATE TABLE dbo.orders (
  id BIGINT PRIMARY KEY,
  customer_id BIGINT NOT NULL,
  order_dt DATETIME2 NOT NULL,
  status VARCHAR(20) NOT NULL,
  amount DECIMAL(12,2) NOT NULL
);

CREATE TABLE dbo.order_items (
  order_id BIGINT NOT NULL,
  product_id BIGINT NOT NULL,
  qty INT NOT NULL,
  price DECIMAL(12,2) NOT NULL,
  CONSTRAINT PK_order_items PRIMARY KEY (order_id, product_id)
);
```

---

# 마무리

- **함수는 강력하지만 성능을 해칠 수도 있다**: 인덱스와 잘 결합되도록 **SARGable 쿼리**를 우선하며, 필요한 경우 **계산 열/함수 기반 인덱스**로 인덱스 측으로 “끌어내려라”.
- **윈도 함수**로 리포트/대시보드 요구사항을 간결·고성능으로 해결하고, **조건부 집계**로 다차원 지표를 한 번에 만든다.
- **JSON/형변환/보안**은 “작동”보다 “운영 가능성(성능·가시성·컴플라이언스)”이 중요하다.
