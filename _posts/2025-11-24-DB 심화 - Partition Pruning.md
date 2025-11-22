---
layout: post
title: DB 심화 - Partition Pruning
date: 2025-11-24 19:25:23 +0900
category: DB 심화
---
# Oracle Partition Pruning

## 왜 Partition Pruning이 “성능 절반”을 먹고 들어가는가

파티션 테이블의 핵심 가치는 **“필요 없는 파티션을 읽지 않는 자유”**다.
인덱스, 조인, 병렬도(DOP), 캐시, I/O 튜닝 이전에 **물리적으로 읽을 대상 자체를 줄이는** 최적화이기 때문에 효과가 가장 크다.

- 파티션 프루닝이 성공하면
  **(전체 테이블 스캔 비용)** → **(필요 파티션만 스캔 비용)** 으로 축소된다.
- **데이터가 커질수록** 차이는 선형이 아니라 **눈덩이처럼 커진다.**
- 병렬(PQ)에서는 프루닝이 곧 **PX SEND/RECEIVE, TEMP 스필, interconnect 트래픽**을 줄이는 지름길이다.

---

## Partition Pruning의 분류와 플랜 신호

### Static(정적) Pruning

- **파싱 시점**에 파티션 범위가 **상수/바인드**로 확정되어 잘린다.
- 플랜 신호
  - `PARTITION RANGE SINGLE / ITERATOR`
  - `PSTART/PSTOP = 상수값`
  - `PSTART/PSTOP = 2 2`처럼 **명확히 숫자로 고정**

### Dynamic(동적) Pruning

- **실행 중**에 값이 결정되어 잘린다.
- 두 갈래가 실무에서 중요하다.

1) **Dynamic Subquery Pruning**
   - 파티션 키 범위가 **서브쿼리 결과**로 결정될 때
   - 서브쿼리가 상수화되면 정적처럼 동작,
     상수화가 안 되면 실행 중 `KEY`, `KEY(INLIST)`로 프루닝

2) **Join Filter(블룸) 기반 Pruning**
   - 조인에서 얻은 키 집합으로 **Bloom Filter(조인 필터)**를 만들어
     **런타임에 파티션/서브파티션을 건너뛴다.**
   - 특히 **PQ에서 강력**하다.

- 플랜 신호
  - `PSTART/PSTOP=KEY` 또는 `KEY(INLIST)`
  - `JOIN FILTER CREATE` / `JOIN FILTER USE`
  - PQ면 `PX JOIN FILTER` 혹은 `PX BLOOM FILTER`

---

## 실습 스키마(공통)

```sql
ALTER SESSION SET nls_date_format = 'YYYY-MM-DD';

DROP TABLE sales PURGE;
CREATE TABLE sales (
  sales_id   NUMBER       PRIMARY KEY,
  sales_dt   DATE         NOT NULL,
  cust_id    NUMBER       NOT NULL,
  region_cd  VARCHAR2(6)  NOT NULL,
  amount     NUMBER(12,2) NOT NULL
)
PARTITION BY RANGE (sales_dt) (
  PARTITION p2025m01 VALUES LESS THAN (DATE '2025-02-01'),
  PARTITION p2025m02 VALUES LESS THAN (DATE '2025-03-01'),
  PARTITION p2025m03 VALUES LESS THAN (DATE '2025-04-01'),
  PARTITION pmax     VALUES LESS THAN (MAXVALUE)
);

DROP TABLE dim_customer PURGE;
CREATE TABLE dim_customer (
  cust_id      NUMBER PRIMARY KEY,
  region_group VARCHAR2(10),
  grade        VARCHAR2(10),
  active_yn    CHAR(1)
);

DROP TABLE dim_calendar PURGE;
CREATE TABLE dim_calendar (
  cal_dt       DATE PRIMARY KEY,
  y            NUMBER,
  m            NUMBER,
  d            NUMBER,
  is_weekend   CHAR(1)
);

INSERT INTO dim_customer VALUES (101, 'APAC', 'GOLD', 'Y');
INSERT INTO dim_customer VALUES (202, 'AMER', 'SILVER', 'Y');
INSERT INTO dim_customer VALUES (303, 'EMEA', 'BRONZE', 'N');

INSERT INTO dim_calendar VALUES (DATE '2025-02-10', 2025, 2, 10, 'N');
INSERT INTO dim_calendar VALUES (DATE '2025-02-11', 2025, 2, 11, 'N');
INSERT INTO dim_calendar VALUES (DATE '2025-03-01', 2025, 3, 1 , 'Y');

INSERT INTO sales VALUES (1, DATE '2025-02-10', 101, 'KR',  100.00);
INSERT INTO sales VALUES (2, DATE '2025-03-01', 202, 'US',  250.00);
INSERT INTO sales VALUES (3, DATE '2025-02-11', 202, 'US',  170.00);
COMMIT;

EXEC DBMS_STATS.GATHER_TABLE_STATS(USER, 'SALES');
EXEC DBMS_STATS.GATHER_TABLE_STATS(USER, 'DIM_CUSTOMER');
EXEC DBMS_STATS.GATHER_TABLE_STATS(USER, 'DIM_CALENDAR');
```

### 플랜 확인 루틴

```sql
ALTER SESSION SET statistics_level = ALL;

-- SQL 실행

SELECT *
FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(
  NULL, NULL,
  'BASIC +ALIAS +PARTITION +PREDICATE +NOTE +OUTLINE'
));
```

- `PARTITION RANGE SINGLE/ITERATOR/JOIN`
- `PSTART/PSTOP`
- `JOIN FILTER CREATE/USE`
- `PX SEND/RECEIVE`(PQ라면)
이 네 가지가 프루닝 독해의 전부다.

---

## 기본(정적) Partition Pruning

### RANGE 파티션: 상수/바인드 기반

```sql
EXPLAIN PLAN FOR
SELECT SUM(amount)
FROM   sales
WHERE  sales_dt >= DATE '2025-02-01'
AND    sales_dt <  DATE '2025-03-01';

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY(NULL,NULL,'BASIC +PARTITION'));
```

**기대 플랜**
- `PARTITION RANGE SINGLE`
- `PSTART/PSTOP = 2 2`
→ p2025m02만 읽는다.

#### 바인드 버전(중요)

```sql
VAR d1 DATE; VAR d2 DATE;
EXEC :d1 := DATE '2025-02-01';
EXEC :d2 := DATE '2025-03-01';

SELECT SUM(amount)
FROM   sales
WHERE  sales_dt >= :d1
AND    sales_dt <  :d2;
```

- 바인드 타입이 **DATE**일 때만 깔끔한 정적 프루닝.
- VARCHAR2 바인드에 `TO_DATE(:x)` 형태를 쓰면
  프루닝이 “될 수도 / 안 될 수도” 있는 불안정 상태로 간다.

---

### LIST/HASH 파티션 프루닝 개념

- LIST면 `WHERE key IN (...)`일 때 해당 파티션만
- HASH면 `WHERE key = :b` 또는 `IN-LIST`일 때 **해시 결과에 해당**하는 파티션만

**전제**
- 파티션 키 컬럼을 **그대로** 비교해야 한다.
  (함수, 표현식, 묵시적 변환이 끼면 깨질 가능성이 높다)

---

### 정적 프루닝을 깨는 대표 안티패턴

| 안티패턴 | 왜 깨지나 | 정석 |
|---|---|---|
| `TO_CHAR(sales_dt,'YYYY-MM')='2025-02'` | 파티션 키에 함수 → 프루닝 불가 | `sales_dt >= DATE '2025-02-01' AND sales_dt < DATE '2025-03-01'` |
| `sales_dt = '2025-02-10'` | 문자열 ↔ DATE 묵시적 변환 | `sales_dt = DATE '2025-02-10'` 또는 DATE 바인드 |
| `TRUNC(sales_dt)=:d` | 함수로 키 가공 | `sales_dt >= :d AND sales_dt < :d+1` |
| `BETWEEN :d1 AND :d2`(상한 포함) | upper-bound 포함으로 경계와 어긋나기 쉬움 | `>= lower AND < upper` |

---

## Dynamic Subquery Pruning

### 개념

파티션 키의 범위/리스트가 **서브쿼리 결과**로 결정될 때
옵티마이저는 다음 중 하나를 한다.

1) **서브쿼리 선평가(상수화)**
   - 결과가 단일값(또는 아주 작은 범위)이고
     실행 전에 구할 수 있으면
     → 정적처럼 프루닝.

2) **런타임 동적 프루닝**
   - 실행 중에 결과가 결정되면
     → `PSTART/PSTOP=KEY` 혹은 `KEY(INLIST)`
     → `PARTITION RANGE ITERATOR`와 `FILTER`가 조합됨.

---

### SCALAR 서브쿼리로 경계 결정

```sql
EXPLAIN PLAN FOR
SELECT SUM(amount)
FROM   sales
WHERE  sales_dt >= (
         SELECT MIN(cal_dt)
         FROM dim_calendar
         WHERE cal_dt >= DATE '2025-02-01' AND is_weekend='N'
       )
AND    sales_dt <  (
         SELECT MAX(cal_dt)+1
         FROM dim_calendar
         WHERE cal_dt < DATE '2025-03-01' AND is_weekend='N'
       );

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY(NULL,NULL,'BASIC +ALIAS +PARTITION'));
```

**관찰 포인트**
- `PSTART/PSTOP=KEY`
- 또는 SCALAR가 상수화되면 숫자가 고정될 수 있다.

**실무 요령**
- `MIN/MAX` 같은 집약은 상수화 확률이 높다.
- 서브쿼리 결과를 먼저 **작게 만들고** 메인 쿼리에 넣을수록 동적 프루닝이 안정된다.

---

### IN-LIST 서브쿼리 기반 프루닝

```sql
EXPLAIN PLAN FOR
SELECT COUNT(*)
FROM   sales
WHERE  cust_id IN (
  SELECT cust_id
  FROM dim_customer
  WHERE grade IN ('GOLD','SILVER') AND active_yn='Y'
);

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY(NULL,NULL,'BASIC +PARTITION'));
```

**기대**
- HASH 파티션이라면 `KEY(INLIST)` 형태로
  **해당 고객 해시가 있는 파티션만 스캔**.

**주의**
- 서브쿼리 결과가 너무 크면 KEY(INLIST) 프루닝의 효과가 약해진다.
  이때는 **조인 필터 프루닝**(다음 장)이 더 유효하다.

---

### Subquery Pruning이 잘 안 먹는 케이스

- 서브쿼리 결과가 **대량**이라 상수화/IN-LIST 프루닝이 부담스러울 때
- **비결정적 함수(SYSDATE, DBMS_RANDOM 등)**가 경계에 개입할 때
- 파티션 키와 서브쿼리 결과 사이의 **데이터 타입/스케일 불일치**

**해결**
- 먼저 서브쿼리에서 후보 범위를 **강하게 축소**
- 데이터 타입 정합
- 필요하면 결과를 `WITH ... MATERIALIZE`로 고정

---

## Join Filter / Bloom-Based Pruning

### 개념

조인 시 **드라이빙(빌드) 집합**에서 얻은 키로
블룸 필터를 만들어 **프로브(사실) 집합의 파티션/서브파티션을 실행 중에 건너뛴다.**

- 특징
  - **런타임 프루닝**이다.
  - 필터는 **완벽한 집합이 아니라 확률적**(false positive 가능).
    그래서 “**읽을 파티션을 확정**”하는 정적 프루닝보다
    “**읽지 말아도 되는 파티션을 빠르게 거르기**”에 강하다.
  - **PQ에서 특히 강력**하다. (PX 서버가 필터를 나눠 갖고 병렬 적용)

- 플랜 신호
  - `JOIN FILTER CREATE` (빌드 쪽)
  - `JOIN FILTER USE` (프로브 쪽)
  - `PSTART/PSTOP=KEY` 또는 `PARTITION RANGE JOIN`

---

### 차원 → 사실 해시 조인에서 Join Filter 생성

```sql
EXPLAIN PLAN FOR
SELECT /*+ LEADING(c) USE_HASH(s) */
       SUM(s.amount)
FROM   dim_customer c
JOIN   sales       s
ON     s.cust_id = c.cust_id
WHERE  c.grade IN ('GOLD','SILVER')
AND    s.sales_dt >= DATE '2025-02-01'
AND    s.sales_dt <  DATE '2025-04-01';

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY(NULL,NULL,'BASIC +ALIAS +PARTITION +NOTE'));
```

**관찰**
- `JOIN FILTER CREATE` on `DIM_CUSTOMER`
- `JOIN FILTER USE` on `SALES`
- `PSTART/PSTOP=KEY` 또는 `PARTITION RANGE JOIN`

**왜 해시 조인이 중요하나?**
- 해시 조인은 빌드 집합을 한 번 모아 키 집합을 만들기 쉬워
  블룸 필터 생성과 푸시다운에 최적이다.
- NL 조인(인덱스 루프)으로 가면
  조인 필터 프루닝은 거의 기대하기 어렵다.

---

### 달력 차원 조인으로 “날짜 집합 기반” 런타임 프루닝

```sql
EXPLAIN PLAN FOR
SELECT /*+ LEADING(d) USE_HASH(s) */
       COUNT(*)
FROM   dim_calendar d
JOIN   sales s
ON     s.sales_dt = d.cal_dt
WHERE  d.cal_dt BETWEEN DATE '2025-02-01' AND DATE '2025-03-10'
AND    d.is_weekend = 'N';

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY(NULL,NULL,'BASIC +ALIAS +PARTITION +NOTE'));
```

- `dim_calendar`에서 뽑힌 날짜 집합으로
  `sales`가 **런타임에 파티션을 건너뛰는지** 확인.
- `KEY(INLIST)`가 뜨면 더 강한 신호.

---

### Join Filter 프루닝을 돕는 실무 힌트

- 조인 순서/방식
  - `LEADING(차원) USE_HASH(사실)`
  - 차원(작은 쪽) → 사실(큰 쪽) 방향이 필터 생성의 정석

- PQ 결합(더 강하게)
  - `PARALLEL(sales 16)` 등으로 사실 테이블에 PQ 부여
  - 필요하면 `PQ_DISTRIBUTE`로 차원 브로드캐스트(작을 때)
    → 차원 빌드가 더 빨라지고 필터 생성이 빨라진다

- 타입 정합
  - `cust_id NUMBER` ↔ `cust_id NUMBER`
  - `sales_dt DATE` ↔ `cal_dt DATE`
  작은 차이(스케일/타입)가 블룸 필터 적용을 끊어버리는 경우가 많다.

---

## Composite(서브파티션) 프루닝까지

Composite 파티션은 “상위 파티션 프루닝 + 하위 서브파티션 프루닝”이 **둘 다** 일어날 때 진짜 이득이다.

### RANGE(월) × HASH(cust_id) 예시

```sql
DROP TABLE sales_cp PURGE;
CREATE TABLE sales_cp (
  sales_id NUMBER,
  sales_dt DATE NOT NULL,
  cust_id  NUMBER NOT NULL,
  amount   NUMBER
)
PARTITION BY RANGE (sales_dt)
SUBPARTITION BY HASH (cust_id)
SUBPARTITIONS 8
(
  PARTITION p2025m02 VALUES LESS THAN (DATE '2025-03-01'),
  PARTITION p2025m03 VALUES LESS THAN (DATE '2025-04-01'),
  PARTITION pmax     VALUES LESS THAN (MAXVALUE)
);
```

```sql
EXPLAIN PLAN FOR
SELECT SUM(amount)
FROM sales_cp
WHERE sales_dt >= DATE '2025-02-01' AND sales_dt < DATE '2025-03-01'
AND   cust_id IN (101,202);

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY(NULL,NULL,'BASIC +PARTITION'));
```

**기대**
- 상위 RANGE `PSTART/PSTOP = p2025m02`
- 하위 HASH는 `KEY(INLIST)`로 서브파티션 프루닝
→ “월도 줄고, 그 월 안에서 고객 해시 서브파티션도 줄어든다.”

---

## SQL 조건절 작성 체크리스트(프루닝/인덱스 공통)

### SARGable하게 쓰기

- 파티션 키에 **함수/표현식 직접 적용 금지**
- 필요하면 “가상 컬럼 → 그걸로 파티셔닝” 또는 FBI

```sql
-- 가상 컬럼 기반 파티셔닝 아이디어
ALTER TABLE accounts
  ADD (yyyymm GENERATED ALWAYS AS (TO_CHAR(open_dt,'YYYYMM')) VIRTUAL);
```

---

### 데이터 타입 정합(묵시적 변환 금지)

```sql
-- 나쁨: 문자열 리터럴
WHERE sales_dt = '2025-02-10'

-- 좋음: DATE 리터럴
WHERE sales_dt = DATE '2025-02-10'

-- 좋음: DATE 바인드
WHERE sales_dt = :d1
```

---

### OR 조건은 신중히

- 같은 파티션 키에 대한 OR은 **OR Expansion(UNION ALL 변환)**이 되면 각 브랜치별 프루닝
- 안 되면 넓게 스캔

**가장 안전한 방법**

```sql
SELECT SUM(amount) FROM sales
 WHERE sales_dt >= DATE '2025-02-01' AND sales_dt < DATE '2025-03-01'
UNION ALL
SELECT SUM(amount) FROM sales
 WHERE sales_dt >= DATE '2025-03-01' AND sales_dt < DATE '2025-04-01';
```

---

### BETWEEN보다 `>= AND <`

- RANGE 경계(`VALUES LESS THAN`)는 상한 미포함 구조
- `>= lower AND < upper`가 **시간/밀리초/타임존** 이슈에 훨씬 안전

---

### 통계

프루닝은 조건식이 결정하지만,
**조인 순서/방식/필터 생성 여부**는 통계가 결정한다.

- 파티션 통계 + 글로벌 통계를 **균형 있게**
- 조인 필터 프루닝을 기대한다면
  차원 테이블 카디널리티/선택도가 정확해야 `LEADING/USE_HASH` 플랜이 자연스럽게 나온다.

---

## 프루닝 검증/트러블슈팅 런북

### 1단계: 플랜에서 “프루닝 신호” 찾기

- `PARTITION RANGE SINGLE/ITERATOR/JOIN`
- `PSTART/PSTOP = 숫자 / KEY / KEY(INLIST)`
- `JOIN FILTER CREATE/USE`

### 2단계: 실제 실행 후 통계 포함 플랜 확인

```sql
ALTER SESSION SET statistics_level = ALL;

SELECT /*+ MONITOR */ SUM(s.amount)
FROM dim_customer c
JOIN sales s ON s.cust_id = c.cust_id
WHERE c.grade IN ('GOLD','SILVER')
AND   s.sales_dt >= DATE '2025-02-01'
AND   s.sales_dt <  DATE '2025-04-01';

SELECT *
FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(
  NULL,NULL,'ALLSTATS LAST +ALIAS +PARTITION +PREDICATE +NOTE'
));
```

- `PSTART/PSTOP`가 KEY인데 실제 읽기량이 크면
  블룸 false positive가 많거나(키가 너무 넓음),
  프루닝 신호만 있고 효과는 미미한 케이스일 수 있다.

### 3단계: “안 되는 이유” 체크

- 파티션 키 가공/함수?
- 타입 불일치?
- OR 폭탄?
- 조인 필터라면 차원 선행 + 해시 조인이 맞는가?
- 통계가 왜곡돼 빌드/프로브 순서가 뒤집혔는가?

---

## 미니 벤치 시나리오(현장 체감용)

### 정적 vs 서브쿼리 동적

```sql
-- 정적
SELECT /*+ MONITOR */ SUM(amount)
FROM sales
WHERE sales_dt >= DATE '2025-02-01'
AND   sales_dt <  DATE '2025-03-01';

-- 동적(서브쿼리)
SELECT /*+ MONITOR */ SUM(amount)
FROM sales
WHERE sales_dt >= (SELECT MIN(cal_dt) FROM dim_calendar WHERE m=2)
AND   sales_dt <  (SELECT MAX(cal_dt)+1 FROM dim_calendar WHERE m=2);
```

**비교 포인트**
- `PSTART/PSTOP=2 2` vs `KEY`
- 읽은 파티션 수, 블록 수, elapsed 비교

---

### OR vs UNION ALL

```sql
-- OR 버전
SELECT /*+ MONITOR */ SUM(amount)
FROM sales
WHERE (sales_dt >= DATE '2025-02-01' AND sales_dt < DATE '2025-03-01')
   OR (sales_dt >= DATE '2025-03-01' AND sales_dt < DATE '2025-04-01');

-- UNION ALL 버전
SELECT /*+ MONITOR */ SUM(amount) FROM sales
 WHERE sales_dt >= DATE '2025-02-01' AND sales_dt < DATE '2025-03-01'
UNION ALL
SELECT /*+ MONITOR */ SUM(amount) FROM sales
 WHERE sales_dt >= DATE '2025-03-01' AND sales_dt < DATE '2025-04-01';
```

- OR Expansion 실패 시 OR 버전은 넓게 스캔.

---

### Join Filter on/off

```sql
-- Join Filter 유도(차원 선행 + 해시)
SELECT /*+ LEADING(c) USE_HASH(s) MONITOR */
       SUM(s.amount)
FROM dim_customer c JOIN sales s
ON s.cust_id=c.cust_id
WHERE c.grade IN ('GOLD','SILVER');

-- Join Filter 억제(NL)
SELECT /*+ LEADING(c) USE_NL(s) MONITOR */
       SUM(s.amount)
FROM dim_customer c JOIN sales s
ON s.cust_id=c.cust_id
WHERE c.grade IN ('GOLD','SILVER');
```

**비교**
- `JOIN FILTER CREATE/USE` 유무
- fact 쪽 `PSTART/PSTOP=KEY` 유무
- 읽은 블록/시간

---

### 타입 미스매치

```sql
-- (나쁨)
SELECT /*+ MONITOR */ COUNT(*)
FROM sales
WHERE sales_dt = '2025-02-10';

-- (좋음)
SELECT /*+ MONITOR */ COUNT(*)
FROM sales
WHERE sales_dt = DATE '2025-02-10';
```

읽기량 차이를 바로 체감 가능.

---

## 결론 — 프루닝은 “읽지 않을 자유”를 설계하는 기술

1. **정적 프루닝**
   - 파티션 키를 **그대로, 같은 타입으로, 범위로** 제한하라.
   - 플랜에서 `PSTART/PSTOP=숫자` 확인.

2. **서브쿼리 동적 프루닝**
   - 경계를 서브쿼리가 결정하면 `KEY/KEY(INLIST)`로 런타임 프루닝.
   - **SCALAR/MIN/MAX/선축소**로 상수화 확률을 높여라.

3. **조인 필터(블룸) 프루닝**
   - 차원→사실, 해시조인, PQ에서 최강.
   - `JOIN FILTER CREATE/USE`, `PARTITION RANGE JOIN`, `KEY` 신호를 읽어라.

4. **현업 SQL의 핵심**
   - **SARGable**, **타입 정합**, **OR 관리(UNION ALL)**,
     **>= AND < 경계**, **통계 정합**,
     이 여섯 가지가 프루닝 성공률을 사실상 결정한다.

> 한 줄 요약:
> **“파티션 키를 직접 제한하는 정적 프루닝이 기본,
> 서브쿼리/조인으로 런타임 프루닝을 덧씌우면 데이터가 클수록 차이는 폭발한다.”**
