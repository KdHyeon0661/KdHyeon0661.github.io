---
layout: post
title: DB 심화 - Partition Pruning
date: 2025-11-24 19:25:23 +0900
category: DB 심화
---
# Oracle **Partition Pruning** 총정리

**주제**: 기본 파티션 프루닝 → **서브쿼리 프루닝**(Dynamic Subquery Pruning) → **조인 필터 프루닝**(Join Filter / Bloom-Based Pruning) → **SQL 조건절 작성 시 주의사항**
**목표**: “**언제/어떻게** 프루닝이 일어나며, **플랜에서 무엇을 확인**할지와 **현업 쿼리 작성 요령**”을 예제와 함께 끝까지 이해하기

> 용어
> - **Partition Pruning(프루닝)**: 쿼리에 필요 없는 **파티션/서브파티션을 아예 읽지 않도록** 잘라내는 최적화.
> - **Static(정적) Pruning**: 파싱 시점에 상수/바인드로 범위를 확정하여 잘라냄.
> - **Dynamic(동적) Pruning**: 실행 중 **서브쿼리/조인에서 나온 값**으로 파티션을 잘라냄.
>   - **Subquery Pruning**: 스칼라 서브쿼리/인라인 뷰 결과로 파티션 키 범위를 동적으로 확정.
>   - **Join Filter Pruning**: 조인 쪽에서 만든 **조인 필터(= Bloom Filter)**를 이용해 **런타임에 파티션을 건너뜀**(특히 PQ에서 강력).

---

## 실습용 스키마(공통)

```sql
ALTER SESSION SET nls_date_format = 'YYYY-MM-DD';

-- 월별 RANGE 파티션: 판매 테이블
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

-- 고객 차원 (비파티션)
DROP TABLE dim_customer PURGE;
CREATE TABLE dim_customer (
  cust_id      NUMBER PRIMARY KEY,
  region_group VARCHAR2(10),
  grade        VARCHAR2(10),
  active_yn    CHAR(1)
);

-- 날짜 차원(일 단위, 최근 기간만 필터링에 사용)
DROP TABLE dim_calendar PURGE;
CREATE TABLE dim_calendar (
  cal_dt       DATE PRIMARY KEY,
  y            NUMBER,
  m            NUMBER,
  d            NUMBER,
  is_weekend   CHAR(1)
);

-- 샘플 데이터 (간단 예시)
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

**플랜 확인 습관**
- `EXPLAIN PLAN` → `DBMS_XPLAN.DISPLAY` 또는
- 실제 실행 후 `DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'BASIC +ALIAS +PARTITION +OUTLINE')`
- **프루닝 여부**는 `PSTART/PSTOP`에 드러남
  - `PSTART/PSTOP = 2 2`(단일 파티션) → 완전 프루닝
  - `KEY` 또는 `KEY(INLIST)` → **동적/조인 기반 프루닝** 신호

---

# 기본(정적) 파티션 프루닝

### 상수/바인드 기반 프루닝 (RANGE)

```sql
-- (A) 상수 리터럴: 2025-02 한 달만 읽기
EXPLAIN PLAN FOR
SELECT /* sales: 2월만 */ SUM(amount)
FROM   sales
WHERE  sales_dt >= DATE '2025-02-01'
AND    sales_dt <  DATE '2025-03-01';

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY(NULL,NULL,'BASIC +PARTITION'));
/*
예상: PARTITION RANGE SINGLE, PSTART/PSTOP = 2 2 (p2025m02만)
*/

-- (B) 바인드 변수(타입이 DATE 여야 함)
VAR :d1 DATE; VAR :d2 DATE;
EXEC :d1 := DATE '2025-02-01'; EXEC :d2 := DATE '2025-03-01';

SELECT /* 바인드 기반 프루닝 */ SUM(amount)
FROM   sales
WHERE  sales_dt >= :d1
AND    sales_dt <  :d2;
```

> 팁
> - **묵시적 형변환**(예: `sales_dt = '2025-02-10'`)은 **프루닝을 깨뜨릴 위험**. **타입 일치**(DATE ↔ DATE)가 중요.
> - 날짜 문자열은 반드시 `DATE 'YYYY-MM-DD'` 리터럴, 또는 **DATE 바인드** 사용.

### LIST/HASH 프루닝

```sql
-- LIST 파티션이라면: WHERE region_cd IN ('KR','US') → 해당 파티션만
-- HASH 파티션이라면: WHERE cust_id IN (101,202) → 해시 결과에 해당하는 파티션만
```

> 프루닝 성립 전제: **파티션 키 컬럼 그대로** 비교(=, IN), **가공/함수 적용** 금지.
> 함수가 필요하면 **가상 컬럼**(+ 그 컬럼 기반 파티셔닝)이나 **FBI** 고려.

### 프루닝을 막는 패턴(정석 회피법)

| 나쁜 예 | 이유 | 좋은 예 |
|---|---|---|
| `WHERE TO_CHAR(sales_dt,'YYYY-MM')='2025-02'` | 파티션 키에 함수 사용 → 프루닝 불가 | `WHERE sales_dt >= DATE '2025-02-01' AND sales_dt < DATE '2025-03-01'` |
| `WHERE sales_dt BETWEEN TO_DATE(:s,'YYYY-MM-DD') AND TO_DATE(:e,'YYYY-MM-DD')` | 바인드가 VARCHAR2 → 런타임 변환/인덱스/프루닝 악영향 | **바인드 타입 DATE** 사용 |
| `WHERE TRUNC(sales_dt)=DATE '2025-02-10'` | 함수로 파티션 키 가공 | `WHERE sales_dt >= DATE '2025-02-10' AND sales_dt < DATE '2025-02-11'` |

---

# **서브쿼리 프루닝**(Dynamic Subquery Pruning)

> **개념**: 파티션 키의 **범위/값**을 **서브쿼리**가 결정하는 경우, 옵티마이저가 서브쿼리를 **선평가**하거나 실행 중 결과를 이용해 **PSTART/PSTOP=KEY** 형태로 **동적 프루닝**한다.
> - 단일값/상수화(SCALAR SUBQUERY, MIN/MAX)일수록 유리
> - 플랜에 `PSTART/PSTOP: KEY`, **FILTER** 단계, 또는 `PARTITION RANGE ITERATOR`가 보인다.

### 예제: **날짜 경계가 서브쿼리 결과**

```sql
-- dim_calendar 에서 “최근 평일 구간”을 서브쿼리로 계산
EXPLAIN PLAN FOR
SELECT SUM(amount)
FROM   sales
WHERE  sales_dt >= (SELECT MIN(cal_dt) FROM dim_calendar WHERE cal_dt>=DATE '2025-02-01' AND is_weekend='N')
AND    sales_dt <  (SELECT MAX(cal_dt)+1 FROM dim_calendar WHERE cal_dt< DATE '2025-03-01' AND is_weekend='N');

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY(NULL,NULL,'BASIC +ALIAS +PARTITION'));
/*
관찰 포인트:
- PSTART/PSTOP: KEY  (서브쿼리 결과를 키로 사용한 동적 프루닝)
- SCALAR SUBQUERY가 먼저 평가되어 상수화될 가능성
*/
```

### 예제: **IN-LIST를 서브쿼리로** (LIST/HASH에도 응용)

```sql
-- region_cd 리스트를 서브쿼리로 얻기 (LIST 파티셔닝 가정)
EXPLAIN PLAN FOR
SELECT COUNT(*)
FROM   sales
WHERE  region_cd IN (
  SELECT CASE WHEN region_group='APAC' THEN 'KR' END
  FROM   dim_customer
  WHERE  grade IN ('GOLD','SILVER') AND active_yn='Y'
);

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY(NULL,NULL,'BASIC +PARTITION'));
/*
PSTART/PSTOP: KEY(INLIST) 형태가 보이면, 서브쿼리로 결정된 IN-LIST를 바탕으로 동적 프루닝
*/
```

> 팁
> - **SCALAR SUBQUERY**가 **한 번 평가(상수화)**되면 정적 프루닝처럼 동작.
> - 결과가 **다중 값**이라면 `KEY(INLIST)`로 표시되는 경우가 많다.
> - `WITH ...` 인라인 뷰로 **서브쿼리 결과를 먼저 축소/상수화**시키면 더 잘 먹힌다.

---

# **조인 필터 프루닝**(Join Filter / Bloom-Based Pruning)

> **개념**: **조인하는 다른 테이블**에서 얻은 키 집합을 **조인 필터**(블룸 필터)로 만들어, **사실상 “해당 키가 포함되지 않는 파티션은 건너뛰는”** 런타임 프루닝.
> - **Parallel Query(PQ)**에서 강력, 최근 버전은 **직렬**에서도 일부 동작.
> - 플랜에 **`JOIN FILTER CREATE` / `JOIN FILTER USE`** 또는 **`PSTART/PSTOP: KEY`**가 찍힘.
> - 조인 키 = 파티션 키(또는 동등 변환 가능)일 때 효과.

### 시나리오: **고객 등급으로 조인 후, 해당 고객만 있는 파티션만 스캔**

```sql
-- 고객 차원에서 GOLD/SILVER만 선택 → sales와 cust_id로 조인
-- sales는 RANGE(sales_dt)이지만, 조인 필터는 '행 수준'뿐 아니라
-- "월별 파티션 중 해당 cust_id 흔적이 없는 파티션"도 건너뛰게 최적화될 수 있음(버전/플랜에 따라 다름).
EXPLAIN PLAN FOR
SELECT /*+ LEADING(c) USE_HASH(s) */ SUM(s.amount)
FROM   dim_customer c
JOIN   sales       s
ON     s.cust_id = c.cust_id
WHERE  c.grade IN ('GOLD','SILVER')
AND    s.sales_dt >= DATE '2025-02-01'
AND    s.sales_dt <  DATE '2025-04-01';

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY(NULL,NULL,'BASIC +ALIAS +PARTITION +NOTE'));
/*
관찰 포인트:
- JOIN FILTER CREATE on C
- JOIN FILTER USE on S
- PARTITION RANGE [JOIN]/[ITERATOR], PSTART/PSTOP: KEY
*/
```

> 실무 팁
> - **해시 조인 + LEADING/USE_HASH**로 **차원 → 사실(Fact)** 순서가 되면 조인 필터 생성이 잘 된다.
> - **PQ**(예: `/*+ PQ_DISTRIBUTE() */`) 환경에서 **더 자주/강하게** 관찰.
> - 컬럼 타입/스케일이 미묘하게 다르면 필터가 비활성화될 수 있으므로 **정합한 데이터 타입**을 유지.

### 예제: **달력 차원으로 기간 좁힌 뒤 조인(동적 범위 프루닝)**

```sql
-- 달력에서 특정 주중만 선택 → sales와 날짜로 조인
EXPLAIN PLAN FOR
SELECT /*+ LEADING(d) USE_HASH(s) */
       COUNT(*)
FROM   dim_calendar d
JOIN   sales       s
ON     s.sales_dt = d.cal_dt
WHERE  d.cal_dt BETWEEN DATE '2025-02-01' AND DATE '2025-03-10'
AND    d.is_weekend = 'N';

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY(NULL,NULL,'BASIC +ALIAS +PARTITION'));
/*
핵심: d에서 뽑아낸 날짜 집합이 '조인 필터'로 반영되어 s의 파티션 범위를 런타임에 잘라낼 수 있음.
PSTART/PSTOP: KEY(INLIST) 또는 KEY 로 표시될 수 있음.
*/
```

### Join Filter 프루닝을 돕는 힌트/조건

- **LEADING(차원) USE_HASH(사실)**: 조인 필터 생성 유도
- **PARTITION-WISE JOIN**이 가능한 데이터 모델(동일 키/경계/개수)일수록 **강력**
- **바인드/함수 가공 금지**, **타입 정합**(NUMBER↔NUMBER, DATE↔DATE)

---

# SQL 조건절 작성 시 **주의사항(체크리스트)**

## **SARGable**하게 쓰기 (프루닝/인덱스 모두에 유익)

- **함수/표현식**을 파티션 키에 직접 적용하지 말 것
  - 나쁨: `TRUNC(sales_dt)=:x` → 좋음: `sales_dt >= :x AND sales_dt < :x+1`
  - 나쁨: `TO_CHAR(sales_dt,'YYYYMM')='202502'` → 좋음: `BETWEEN` 범위
- **묵시적 형변환 금지**: 리터럴 ‘문자열’ 대신 **DATE/NUMBER 리터럴 또는 동일 타입 바인드**
- **IN-LIST**가 파티션 키이면 **정확한 프루닝** → `KEY(INLIST)`

## **OR** 조건과 프루닝

- 같은 파티션 키에 대한 **OR**은 옵티마이저가 **OR Expansion**(UNION ALL 변환)을 시도
  - 성공하면 **브랜치별 프루닝**이 가능 → 오히려 **빠를 수 있음**
  - 실패하면 넓은 범위 스캔 → 느릴 수 있음
- 직접 **`UNION ALL`로 분해**하면 **브랜치별 힌트/조건** 제공이 쉬워짐

```sql
-- OR 대신 UNION ALL (브랜치별 프루닝 보장)
SELECT SUM(amount) FROM sales
 WHERE sales_dt >= DATE '2025-02-01' AND sales_dt < DATE '2025-03-01'
UNION ALL
SELECT SUM(amount) FROM sales
 WHERE sales_dt >= DATE '2025-03-01' AND sales_dt < DATE '2025-04-01';
```

## **BETWEEN** vs **>= AND <** (RANGE 경계)

- 파티션 경계가 보통 `VALUES LESS THAN (upper)`이므로,
  `>= lower AND < upper` 형태가 **오프바이원/시간대 이슈**를 피하기 쉽다.
- `BETWEEN`은 양끝 **포함** → 상한을 **하루 뒤**로 바꿔 쓰지 않으면 의도 미스매치 위험.

## **서브쿼리/조인**으로 프루닝 유도

- 서브쿼리는 **상수화**(SCALAR)되도록 **집약**한다(예: `MIN/MAX`, `LISTAGG → IN-LIST`)
- 조인은 **차원 선행(LEADING)**, **해시 조인(USE_HASH)**로 **Join Filter** 생성 유도
- **타입/스케일 정합**(NUMBER(10) ↔ NUMBER, DATE ↔ DATE)

## **가상 컬럼/함수기반 파티셔닝(FBP)** 고려

- 불가피하게 가공이 필요하면 **가상 컬럼 생성 → 그 컬럼으로 파티셔닝**
- 또는 **함수기반 파티셔닝**(Function-Based Partitioning) 도입

```sql
-- 계정 앞 3자리로 LIST 파티션을 하고 싶다 → 가상 컬럼
ALTER TABLE accounts ADD (branch_code GENERATED ALWAYS AS (SUBSTR(account_id,1,3)) VIRTUAL);
-- 이후 branch_code로 LIST 파티셔닝 생성(또는 재생성)
```

## **통계/카디널리티**의 정확성

- 프루닝 자체는 조건식에서 결정되지만, **플랜 선택**(조인 순서/방식/필터 생성)은 통계의 영향을 받음
- **파티션/서브파티션 통계** + **글로벌 통계**를 **균형 있게** 수집

---

# 프루닝 **검증 절차** (플랜/실행에서 확인)

### DBMS_XPLAN으로 PSTART/PSTOP, Join Filter 확인

```sql
-- 실제 실행 후 확인(가장 권장): GATHER_PLAN_STATISTICS 또는 /*+ MONITOR */
ALTER SESSION SET statistics_level = ALL;

SELECT /*+ MONITOR */ SUM(s.amount)
FROM   dim_customer c
JOIN   sales s ON s.cust_id = c.cust_id
WHERE  c.grade IN ('GOLD','SILVER')
AND    s.sales_dt >= DATE '2025-02-01'
AND    s.sales_dt <  DATE '2025-04-01';

SELECT *
FROM   TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL, NULL, 'BASIC +ALIAS +PARTITION +NOTE +OUTLINE'));

/*
확인 포인트
- PARTITION RANGE [SINGLE/ITERATOR/JOIN]
- PSTART/PSTOP = 2 3  (정적 범위), 또는 KEY / KEY(INLIST) (동적)
- JOIN FILTER CREATE / JOIN FILTER USE  (노트/라인)
*/
```

### “프루닝이 안 된다” 체크리스트

- [ ] 파티션 키에 **함수/형변환**이 걸렸는가?
- [ ] 리터럴/바인드의 **데이터 타입**이 일치하는가?
- [ ] **OR**로 넓게 감싸지 않았는가? → **UNION ALL**로 브랜치 분리
- [ ] 조인 기반 프루닝이면 **LEADING/USE_HASH** 등으로 **차원 선행**을 만들 수 있는가?
- [ ] 통계/카디널리티 부정확으로 플랜이 **필터 생성**을 포기하지 않았는가?

---

# 심화: Composite/서브파티션까지 프루닝

> Composite(RANGE-HASH, RANGE-LIST 등)에서는 **상위 파티션 프루닝** 후,
> 하위 **서브파티션 프루닝**까지 일어나야 진짜 I/O 절감이 크다.

```sql
-- 예시: RANGE(월) × HASH(cust_id) 구조에서
-- WHERE sales_dt BETWEEN ... AND ...  → 상위 RANGE 프루닝
--   + WHERE cust_id IN (101,202)     → 하위 HASH 프루닝(= KEY(INLIST))
```

**플랜**에서 `PSTART/PSTOP`가 **서브파티션 레벨**로 나타나거나, 접근 오퍼레이터에 **SUBPARTITION** 힌트/표기가 보이면 성공.

---

# 미니 벤치 시나리오(아이디어)

1. **정적 vs 동적 프루닝 비교**
   - 같은 범위를 **상수**로 줄 때와 **서브쿼리 MIN/MAX**로 줄 때 **PSTART/PSTOP** 차이를 확인.
2. **OR vs UNION ALL** 비교
   - 같은 월 두 개를 OR로 묶은 버전 vs UNION ALL로 나눈 버전 성능 비교.
3. **조인 필터 on / off**
   - `LEADING/USE_HASH`와 `NO_USE_HASH`(혹은 NL 조인) 플랜 비교 → **JOIN FILTER** 생성 여부 차이 체감.
4. **타입 미스매치**
   - `sales_dt = '2025-02-10'` vs `sales_dt = DATE '2025-02-10'` **블록 읽기량** 차이를 눈으로 확인.

---

## 요약

- **기본 프루닝**: **상수/바인드**로 파티션 키를 직접 제한(함수/형변환 금지, `>= AND <` 권장).
- **서브쿼리 프루닝**: **서브쿼리 결과**(SCALAR/IN-LIST)가 키가 되어 **`KEY`/`KEY(INLIST)`**로 표시.
- **조인 필터 프루닝**: 차원→사실 방향 **해시 조인**에서 **JOIN FILTER CREATE/USE**가 보이면 성공(특히 PQ에서 강력).
- **작성 주의사항**: SARGable, 타입 정합, OR→UNION ALL, 가상 컬럼/함수기반 파티셔닝, 통계 정확성.
- **검증**: `DBMS_XPLAN.DISPLAY_CURSOR(... '+PARTITION')`로 **PSTART/PSTOP**을 반드시 확인.

---

## 부록: 자주 쓰는 힌트/검증 스니펫

```sql
-- 조인 필터/동적 프루닝을 돕는 전형
SELECT /*+ LEADING(d) USE_HASH(s) GATHER_PLAN_STATISTICS */
       COUNT(*)
FROM   dim_calendar d
JOIN   sales s
ON     s.sales_dt = d.cal_dt
WHERE  d.cal_dt BETWEEN :d1 AND :d2
AND    d.is_weekend = 'N';

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'BASIC +PARTITION +ALIAS +NOTE'));

-- OR 확장을 유도하고 싶을 때(버전에 따라 자동 OR Expansion 가능)
-- 안 먹히면 직접 UNION ALL로 분해
SELECT /*+ NO_EXPAND */ SUM(amount)
FROM   sales
WHERE  sales_dt BETWEEN DATE '2025-02-01' AND DATE '2025-02-28'
OR     sales_dt BETWEEN DATE '2025-03-01' AND DATE '2025-03-31';
```

> 결론: 프루닝은 “**읽지 않을 자유**”를 확보하는 기술입니다.
> 쿼리 조건을 **파티션 키에 직결**하고, **서브쿼리/조인 필터**를 통해 **런타임에도 잘라낼 수 있게** 만들면,
> “데이터가 커질수록” **차이가 눈덩이처럼 커집니다**.
