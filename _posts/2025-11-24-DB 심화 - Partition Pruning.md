---
layout: post
title: DB 심화 - Partition Pruning
date: 2025-11-24 19:25:23 +0900
category: DB 심화
---
# Oracle Partition Pruning: 파티션 프루닝의 원리와 실전 활용

## 파티션 프루닝의 중요성

파티션 테이블의 가장 큰 가치는 **"필요 없는 파티션을 아예 읽지 않는 것"**에 있습니다. 이는 인덱스 최적화, 조인 튜닝, 캐시 관리 등 다른 최적화 기법들보다 근본적으로 성능에 더 큰 영향을 미칩니다. 왜냐하면 물리적으로 접근해야 할 데이터의 양 자체를 줄여주기 때문입니다.

**파티션 프루닝의 효과**:
- **비용 절감**: 전체 테이블 스캔 비용 → 필요한 파티션만 스캔하는 비용으로 축소
- **확장성**: 데이터 양이 증가할수록 성능 향상 효과가 기하급수적으로 커짐
- **병렬 처리 최적화**: PX SEND/RECEIVE 연산, TEMP 스필, 인터커넥트 트래픽을 획기적으로 감소

---

## 파티션 프루닝의 종류와 실행계획 신호

### 정적 프루닝(Static Pruning)

정적 프루닝은 쿼리 파싱 단계에서 파티션 범위가 이미 결정되는 경우입니다. 상수값이나 바인드 변수로 파티션 키가 명확히 지정될 때 발생합니다.

**실행계획 신호**:
- `PARTITION RANGE SINGLE` 또는 `PARTITION RANGE ITERATOR`
- `PSTART`와 `PSTOP`이 상수값으로 표시 (예: `PSTART=2 PSTOP=2`)

```sql
-- 정적 프루닝 예제
EXPLAIN PLAN FOR
SELECT SUM(amount)
FROM sales
WHERE sales_dt >= DATE '2025-02-01'
  AND sales_dt < DATE '2025-03-01';

-- 실행계획 결과
-- PARTITION RANGE SINGLE
-- PSTART=2 PSTOP=2 (p2025m02 파티션만 스캔)
```

### 동적 프루닝(Dynamic Pruning)

동적 프루닝은 실행 시점에 파티션 범위가 결정되는 경우로, 두 가지 주요 유형이 있습니다.

#### 1. 동적 서브쿼리 프루닝(Dynamic Subquery Pruning)
파티션 키의 범위가 서브쿼리 결과로 결정될 때 발생합니다.

#### 2. 조인 필터(블룸) 기반 프루닝(Join Filter/Bloom-based Pruning)
조인 과정에서 생성된 블룸 필터를 이용해 런타임에 불필요한 파티션을 건너뜁니다. 특히 병렬 처리 환경에서 강력한 효과를 발휘합니다.

**실행계획 신호**:
- `PSTART/PSTOP=KEY` 또는 `PSTART/PSTOP=KEY(INLIST)`
- `JOIN FILTER CREATE` / `JOIN FILTER USE`
- 병렬 처리 시 `PX JOIN FILTER` 또는 `PX BLOOM FILTER`

---

## 실습 환경 구성

```sql
-- 기본 테이블 구조
DROP TABLE sales PURGE;
DROP TABLE dim_customer PURGE;
DROP TABLE dim_calendar PURGE;

-- 파티션 테이블 생성 (월별 RANGE 파티션)
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

-- 차원 테이블들
CREATE TABLE dim_customer (
    cust_id      NUMBER PRIMARY KEY,
    region_group VARCHAR2(10),
    grade        VARCHAR2(10),
    active_yn    CHAR(1)
);

CREATE TABLE dim_calendar (
    cal_dt     DATE PRIMARY KEY,
    y          NUMBER,
    m          NUMBER,
    d          NUMBER,
    is_weekend CHAR(1)
);

-- 샘플 데이터 삽입
INSERT INTO dim_customer VALUES (101, 'APAC', 'GOLD', 'Y');
INSERT INTO dim_customer VALUES (202, 'AMER', 'SILVER', 'Y');
INSERT INTO dim_customer VALUES (303, 'EMEA', 'BRONZE', 'N');

INSERT INTO dim_calendar VALUES (DATE '2025-02-10', 2025, 2, 10, 'N');
INSERT INTO dim_calendar VALUES (DATE '2025-02-11', 2025, 2, 11, 'N');
INSERT INTO dim_calendar VALUES (DATE '2025-03-01', 2025, 3, 1, 'Y');

INSERT INTO sales VALUES (1, DATE '2025-02-10', 101, 'KR', 100.00);
INSERT INTO sales VALUES (2, DATE '2025-03-01', 202, 'US', 250.00);
INSERT INTO sales VALUES (3, DATE '2025-02-11', 202, 'US', 170.00);

COMMIT;

-- 통계 수집
EXEC DBMS_STATS.GATHER_TABLE_STATS(USER, 'SALES');
EXEC DBMS_STATS.GATHER_TABLE_STATS(USER, 'DIM_CUSTOMER');
EXEC DBMS_STATS.GATHER_TABLE_STATS(USER, 'DIM_CALENDAR');
```

### 실행계획 분석 루틴

```sql
-- 성능 통계 수집 활성화
ALTER SESSION SET statistics_level = ALL;

-- 쿼리 실행 후
SELECT *
FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(
    NULL, NULL,
    'BASIC +ALIAS +PARTITION +PREDICATE +NOTE +OUTLINE'
));
```

---

## 기본 정적 프루닝 활용

### RANGE 파티션에서의 정적 프루닝

```sql
-- 상수값을 이용한 정적 프루닝
EXPLAIN PLAN FOR
SELECT SUM(amount)
FROM sales
WHERE sales_dt >= DATE '2025-02-01'
  AND sales_dt < DATE '2025-03-01';

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY(NULL, NULL, 'BASIC +PARTITION'));
```

**실행계획 해석**:
- `PARTITION RANGE SINGLE`: 단일 파티션만 스캔
- `PSTART=2 PSTOP=2`: 두 번째 파티션(p2025m02)만 접근
- 이는 2025년 2월 데이터만 처리함을 의미

### 바인드 변수를 활용한 정적 프루닝

```sql
-- DATE 타입 바인드 변수 사용
VARIABLE v_start_date DATE;
VARIABLE v_end_date DATE;

BEGIN
    :v_start_date := DATE '2025-02-01';
    :v_end_date := DATE '2025-03-01';
END;
/

SELECT SUM(amount)
FROM sales
WHERE sales_dt >= :v_start_date
  AND sales_dt < :v_end_date;
```

**중요 포인트**:
- DATE 타입 바인드 변수는 정적 프루닝이 잘 동작함
- VARCHAR2 타입에 `TO_DATE()` 함수를 적용하면 프루닝이 불안정해질 수 있음

### LIST/HASH 파티션의 프루닝

- **LIST 파티션**: `WHERE key IN (...) ` 조건에서 해당 파티션만 접근
- **HASH 파티션**: `WHERE key = :value` 또는 `WHERE key IN (...)` 조건에서 해시 결과에 해당하는 파티션만 접근

**기본 원칙**: 파티션 키 컬럼을 그대로 비교해야 프루닝이 효과적으로 작동합니다.

---

## 정적 프루닝을 방해하는 안티패턴들

프루닝 효과를 최대화하려면 다음 안티패턴들을 피해야 합니다.

| 안티패턴 | 문제점 | 개선안 |
|---------|--------|--------|
| `TO_CHAR(sales_dt,'YYYY-MM')='2025-02'` | 파티션 키에 함수 적용 | `sales_dt >= DATE '2025-02-01' AND sales_dt < DATE '2025-03-01'` |
| `sales_dt = '2025-02-10'` | 묵시적 형변환 발생 | `sales_dt = DATE '2025-02-10'` |
| `TRUNC(sales_dt) = :date` | 함수로 키 변형 | `sales_dt >= :date AND sales_dt < :date + 1` |
| `BETWEEN :start AND :end` | 상한값 포함 문제 | `>= :start AND < :end` |

---

## 동적 서브쿼리 프루닝

### 개념 이해

동적 서브쿼리 프루닝은 파티션 키의 범위가 서브쿼리 실행 결과에 의해 결정될 때 발생합니다. 옵티마이저는 두 가지 방식으로 처리합니다:

1. **서브쿼리 선평가(상수화)**: 서브쿼리 결과가 단일값이거나 작은 범위일 때, 실행 전에 평가하여 정적 프루닝처럼 처리
2. **런타임 동적 프루닝**: 실행 중에 결과가 결정되면 `KEY` 또는 `KEY(INLIST)` 형태로 프루닝

### 스칼라 서브쿼리를 이용한 동적 프루닝

```sql
-- 최소/최대값을 서브쿼리로 결정
EXPLAIN PLAN FOR
SELECT SUM(amount)
FROM sales
WHERE sales_dt >= (
    SELECT MIN(cal_dt)
    FROM dim_calendar
    WHERE cal_dt >= DATE '2025-02-01'
      AND is_weekend = 'N'
)
AND sales_dt < (
    SELECT MAX(cal_dt) + 1
    FROM dim_calendar
    WHERE cal_dt < DATE '2025-03-01'
      AND is_weekend = 'N'
);

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY(NULL, NULL, 'BASIC +ALIAS +PARTITION'));
```

**관찰 포인트**:
- `PSTART/PSTOP=KEY`로 표시됨
- 서브쿼리 결과가 상수화되면 숫자로 고정될 수 있음

**실무 팁**: `MIN`, `MAX` 같은 집계 함수는 상수화 확률이 높으므로 동적 프루닝에 유리합니다.

### IN-LIST 서브쿼리 기반 프루닝

```sql
-- 서브쿼리 결과를 IN-LIST로 활용
EXPLAIN PLAN FOR
SELECT COUNT(*)
FROM sales
WHERE cust_id IN (
    SELECT cust_id
    FROM dim_customer
    WHERE grade IN ('GOLD', 'SILVER')
      AND active_yn = 'Y'
);

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY(NULL, NULL, 'BASIC +PARTITION'));
```

**실행계획 특징**:
- HASH 파티션의 경우 `KEY(INLIST)` 형태로 표시
- 해당 고객 ID의 해시값이 매핑되는 파티션만 스캔

**주의사항**: 서브쿼리 결과가 너무 많으면 `KEY(INLIST)` 프루닝의 효과가 감소합니다. 이 경우 조인 필터 프루닝이 더 효과적일 수 있습니다.

### 동적 프루닝 실패 케이스와 해결책

**문제 상황**:
- 서브쿼리 결과가 대량이라 상수화/IN-LIST 프루닝 부담
- 비결정적 함수(`SYSDATE`, `DBMS_RANDOM`) 사용
- 데이터 타입이나 스케일 불일치

**해결 전략**:
1. 서브쿼리에서 결과 집합을 사전에 축소
2. 데이터 타입 정합성 확보
3. 필요한 경우 `WITH ... MATERIALIZE`로 결과 고정

---

## 조인 필터(블룸) 기반 프루닝

### 작동 원리

조인 필터 프루닝은 조인 연산 중 드라이빙(빌드) 테이블에서 추출한 키값으로 블룸 필터를 생성하고, 이 필터를 이용해 프로브 테이블의 불필요한 파티션을 런타임에 건너뛰는 기법입니다.

**특징**:
- **확률적 필터링**: 블룸 필터는 false positive(거짓 양성) 가능성이 있지만, 메모리 효율이 뛰어남
- **런타임 최적화**: 실행 중에 동적으로 적용
- **병렬 처리 친화적**: PX 서버 간 효율적인 필터 공유 가능

**실행계획 신호**:
- `JOIN FILTER CREATE` (빌드 측)
- `JOIN FILTER USE` (프로브 측)
- `PSTART/PSTOP=KEY` 또는 `PARTITION RANGE JOIN`

### 해시 조인을 활용한 조인 필터 프루닝

```sql
-- 차원 → 사실 테이블 조인에서의 조인 필터
EXPLAIN PLAN FOR
SELECT /*+ LEADING(c) USE_HASH(s) */
       SUM(s.amount)
FROM dim_customer c
JOIN sales s ON s.cust_id = c.cust_id
WHERE c.grade IN ('GOLD', 'SILVER')
  AND s.sales_dt >= DATE '2025-02-01'
  AND s.sales_dt < DATE '2025-04-01';

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY(NULL, NULL, 'BASIC +ALIAS +PARTITION +NOTE'));
```

**관찰 포인트**:
- `JOIN FILTER CREATE`가 `DIM_CUSTOMER`에 적용
- `JOIN FILTER USE`가 `SALES`에 적용
- `PSTART/PSTOP=KEY` 또는 `PARTITION RANGE JOIN` 표시

**해시 조인의 중요성**: 해시 조인은 빌드 집합을 한 번에 모아 블룸 필터 생성에 최적화되어 있습니다. NL 조인은 루프 기반이므로 조인 필터 생성이 어렵습니다.

### 날짜 차원을 이용한 조인 필터 프루닝

```sql
-- 달력 차원과의 조인을 통한 런타임 프루닝
EXPLAIN PLAN FOR
SELECT /*+ LEADING(d) USE_HASH(s) */
       COUNT(*)
FROM dim_calendar d
JOIN sales s ON s.sales_dt = d.cal_dt
WHERE d.cal_dt BETWEEN DATE '2025-02-01' AND DATE '2025-03-10'
  AND d.is_weekend = 'N';

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY(NULL, NULL, 'BASIC +ALIAS +PARTITION +NOTE'));
```

**동작 원리**: `dim_calendar` 테이블에서 추출한 날짜 집합을 기반으로 `sales` 테이블의 파티션을 런타임에 걸러냅니다.

### 조인 필터 프루닝 최적화 힌트

```sql
-- 효과적인 조인 필터 유도
SELECT /*+ 
    LEADING(dimension_table)  -- 차원 테이블 선행
    USE_HASH(fact_table)      -- 해시 조인 사용
    PARALLEL(fact_table 8)    -- 병렬 처리
*/
FROM dimension_table
JOIN fact_table ON ...
```

**핵심 전략**:
1. **차원 → 사실 방향**: 작은 차원 테이블을 드라이빙으로 설정
2. **해시 조인 선호**: 블룸 필터 생성에 최적화된 조인 방식
3. **데이터 타입 정합**: 조인 키의 데이터 타입이 정확히 일치해야 함

---

## 복합 파티션에서의 프루닝

복합(Composite) 파티션은 상위 파티션과 하위 서브파티션 모두에서 프루닝이 발생할 때 최대 효과를 발휘합니다.

### RANGE × HASH 복합 파티션 예제

```sql
-- 복합 파티션 테이블 생성
DROP TABLE sales_composite PURGE;

CREATE TABLE sales_composite (
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

-- 복합 프루닝 예제 쿼리
EXPLAIN PLAN FOR
SELECT SUM(amount)
FROM sales_composite
WHERE sales_dt >= DATE '2025-02-01'
  AND sales_dt < DATE '2025-03-01'
  AND cust_id IN (101, 202);

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY(NULL, NULL, 'BASIC +PARTITION'));
```

**프루닝 효과**:
1. **상위 RANGE 프루닝**: 2025년 2월 파티션만 선택
2. **하위 HASH 프루닝**: 고객 ID 101, 202에 해당하는 서브파티션만 선택

**실무 적용**: 복합 파티션은 다차원 접근 패턴이 있는 데이터에 특히 효과적입니다 (예: 시간 + 고객, 지역 + 제품 등).

---

## 쿼리 작성 모범 사례

### SARGable 조건 작성

파티션 키에 직접적인 조건을 적용해야 프루닝이 효과적으로 작동합니다.

```sql
-- 나쁜 예: 함수 적용
SELECT * FROM sales WHERE TO_CHAR(sales_dt, 'YYYYMM') = '202502';

-- 좋은 예: 직접 범위 지정
SELECT * FROM sales 
WHERE sales_dt >= DATE '2025-02-01' 
  AND sales_dt < DATE '2025-03-01';
```

**가상 컬럼 활용**: 함수 기반 접근이 필요할 때는 가상 컬럼을 이용한 파티셔닝을 고려하세요.

```sql
-- 가상 컬럼 기반 파티셔닝
ALTER TABLE transactions
ADD (transaction_month GENERATED ALWAYS AS (TRUNC(transaction_date, 'MM')));

CREATE INDEX idx_trans_month ON transactions(transaction_month);
```

### 데이터 타입 정합성 유지

묵시적 형변환은 프루닝 실패의 주요 원인입니다.

```sql
-- 나쁜 예: 문자열 리터럴 (묵시적 변환)
SELECT * FROM sales WHERE sales_dt = '2025-02-10';

-- 좋은 예: DATE 리터럴
SELECT * FROM sales WHERE sales_dt = DATE '2025-02-10';

-- 좋은 예: DATE 타입 바인드 변수
SELECT * FROM sales WHERE sales_dt = :target_date;
```

### OR 조건의 신중한 처리

OR 조건은 프루닝을 복잡하게 만들 수 있습니다. 가능하면 UNION ALL로 분리하는 것이 안전합니다.

```sql
-- 위험한 OR 조건
SELECT SUM(amount) FROM sales
WHERE (sales_dt >= DATE '2025-02-01' AND sales_dt < DATE '2025-03-01')
   OR (sales_dt >= DATE '2025-03-01' AND sales_dt < DATE '2025-04-01');

-- 안전한 UNION ALL 분리
SELECT SUM(amount) FROM sales
WHERE sales_dt >= DATE '2025-02-01' AND sales_dt < DATE '2025-03-01'
UNION ALL
SELECT SUM(amount) FROM sales
WHERE sales_dt >= DATE '2025-03-01' AND sales_dt < DATE '2025-04-01';
```

### 범위 조건 작성 시 주의사항

RANGE 파티션은 상한값을 포함하지 않는 구조이므로, `>= AND <` 패턴이 가장 안전합니다.

```sql
-- 권장 패턴
WHERE sales_dt >= :start_date AND sales_dt < :end_date

-- 피해야 할 패턴 (상한값 포함 문제)
WHERE sales_dt BETWEEN :start_date AND :end_date
```

---

## 프루닝 효과 검증과 문제 해결

### 1단계: 실행계획 분석

프루닝이 제대로 작동하는지 확인하려면 실행계획에서 다음 신호들을 찾아보세요:

```sql
-- 상세 실행계획 확인
SELECT *
FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(
    NULL, NULL,
    'ALLSTATS LAST +ALIAS +PARTITION +PREDICATE +NOTE'
));
```

**확인 포인트**:
- `PARTITION RANGE SINGLE/ITERATOR/JOIN` 여부
- `PSTART/PSTOP` 값 (숫자, KEY, KEY(INLIST))
- `JOIN FILTER CREATE/USE` 존재 여부
- 실제 읽은 파티션 수와 블록 수

### 2단계: 성능 통계 비교

```sql
-- 프루닝 효과 측정
ALTER SESSION SET statistics_level = ALL;

-- 쿼리 실행
SELECT /*+ MONITOR */ SUM(s.amount)
FROM dim_customer c
JOIN sales s ON s.cust_id = c.cust_id
WHERE c.grade IN ('GOLD', 'SILVER')
  AND s.sales_dt >= DATE '2025-02-01'
  AND s.sales_dt < DATE '2025-04-01';

-- 통계 확인
SELECT 
    sn.name AS metric_name,
    ms.value AS value
FROM v$mystat ms
JOIN v$statname sn ON sn.stat# = ms.stat#
WHERE sn.name IN (
    'physical reads',
    'session logical reads',
    'table scan blocks gotten',
    'table scan rows gotten'
);
```

### 3단계: 문제 진단

프루닝이 예상대로 작동하지 않을 때 체크할 사항들:

1. **파티션 키 조건 검토**
   - 함수나 표현식이 적용되지 않았는가?
   - 데이터 타입이 정확히 일치하는가?

2. **조인 방식 확인**
   - 차원 테이블이 드라이빙되는가?
   - 해시 조인이 사용되는가?

3. **통계 정확성**
   - 파티션 통계와 글로벌 통계가 최신인가?
   - 카디널리티 추정이 정확한가?

4. **OR 조건 처리**
   - OR Expansion이 발생하는가?
   - UNION ALL로 변환 가능한가?

---

## 실전 시나리오 비교

### 시나리오 1: 정적 vs 동적 프루닝 비교

```sql
-- 정적 프루닝
SELECT /*+ MONITOR */ 
    SUM(amount),
    COUNT(*) AS row_count
FROM sales
WHERE sales_dt >= DATE '2025-02-01'
  AND sales_dt < DATE '2025-03-01';

-- 동적 프루닝 (서브쿼리)
SELECT /*+ MONITOR */ 
    SUM(amount),
    COUNT(*) AS row_count
FROM sales
WHERE sales_dt >= (SELECT MIN(cal_dt) FROM dim_calendar WHERE m = 2)
  AND sales_dt < (SELECT MAX(cal_dt) + 1 FROM dim_calendar WHERE m = 2);
```

**비교 항목**:
- 실행계획의 `PSTART/PSTOP` 값
- 읽은 파티션 수
- 처리 시간
- 물리적 읽기 수

### 시나리오 2: 조인 필터 효과 측정

```sql
-- 조인 필터 활성 (해시 조인)
SELECT /*+ LEADING(c) USE_HASH(s) MONITOR */
    SUM(s.amount) AS total_amount
FROM dim_customer c
JOIN sales s ON s.cust_id = c.cust_id
WHERE c.grade IN ('GOLD', 'SILVER');

-- 조인 필터 비활성 (NL 조인)
SELECT /*+ LEADING(c) USE_NL(s) MONITOR */
    SUM(s.amount) AS total_amount
FROM dim_customer c
JOIN sales s ON s.cust_id = c.cust_id
WHERE c.grade IN ('GOLD', 'SILVER');
```

**관찰 포인트**:
- `JOIN FILTER CREATE/USE` 유무
- `PSTART/PSTOP=KEY` 유무
- 처리된 블록 수 차이
- 총 실행 시간 비교

---

## 결론: 파티션 프루닝 마스터하기

파티션 프루닝은 대용량 데이터 환경에서 필수적인 최적화 기술입니다. 효과적인 프루닝을 위해 다음 원칙들을 기억하세요:

### 핵심 원칙 3가지

1. **정적 프루닝 기반 설계**
   - 파티션 키를 직접적으로, 올바른 타입으로 조건 지정
   - `>= AND <` 패턴으로 범위 명확히 정의
   - 실행계획에서 `PSTART/PSTOP=상수` 확인

2. **동적 프루닝 전략적 활용**
   - 서브쿼리 결과를 이용한 런타임 프루닝 적극 활용
   - 조인 필터를 통한 블룸 기반 프루닝은 대용량 조인에서 강력
   - 병렬 처리 환경에서 조인 필터 효과 극대화

3. **쿼리 작성 규칙 준수**
   - SARGable 조건 유지 (함수 적용 회피)
   - 데이터 타입 정합성 확보
   - OR 조건은 UNION ALL로 안전하게 분리

### 실무 적용 체크리스트

- [ ] 파티션 키에 함수나 표현식이 적용되지 않았는가?
- [ ] 데이터 타입이 정확히 일치하는가?
- [ ] 범위 조건이 `>= AND <` 패턴으로 작성되었는가?
- [ ] 조인 시 차원 → 사실 방향으로 설계되었는가?
- [ ] 해시 조인이 적절히 활용되는가?
- [ ] 실행계획에서 프루닝 신호가 명확한가?
- [ ] 실제 성능 통계로 효과가 입증되는가?

### 지속적인 최적화 사이클

1. **분석**: 실행계획으로 프루닝 현황 진단
2. **설계**: 쿼리와 인덱스 구조를 프루닝 친화적으로 개선
3. **구현**: 최적의 조인 순서와 방식 적용
4. **검증**: 성능 통계와 실행계획으로 효과 확인
5. **모니터링**: 프로덕션 환경에서 지속적 관찰

파티션 프루닝은 단순한 성능 최적화 기술을 넘어, 대용량 데이터 시스템의 설계 철학을 반영합니다. "필요한 데이터만 정확하게 접근한다"는 이 원칙은 데이터 양이 증가할수록 그 가치가 더욱 빛납니다. 체계적인 접근과 과학적인 검증을 통해 예측 가능한 성능을 구현하세요.