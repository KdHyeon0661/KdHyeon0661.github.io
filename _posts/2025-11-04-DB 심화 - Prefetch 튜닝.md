---
layout: post
title: DB 심화 - Prefetch 튜닝
date: 2025-11-04 18:25:23 +0900
category: DB 심화
---
# Prefetch 튜닝: 인덱스와 테이블 프리페치 최적화

> **핵심 원리**
> 프리페치는 데이터베이스 성능 최적화의 중요한 기법입니다. 인덱스 프리페치는 인덱스 범위 스캔 시 인접 블록을 미리 읽어 연속 탐색의 지연을 줄이고, 테이블 프리페치는 인덱스에서 수집한 ROWID들을 모아 배치로 테이블 블록을 읽어 랜덤 I/O를 최소화합니다. 두 기법 모두 I/O 효율성을 극대화하여 응답 시간을 개선합니다.

---

## 실습 환경 구성

```sql
-- 테이블 생성 및 샘플 데이터 적재
DROP TABLE sales PURGE;
DROP TABLE dim_customer PURGE;

-- 판매 테이블
CREATE TABLE sales (
  sale_id     NUMBER PRIMARY KEY,
  cust_id     NUMBER NOT NULL,
  sale_dt     DATE   NOT NULL,
  amount      NUMBER(12,2) NOT NULL,
  status      VARCHAR2(8)  NOT NULL,
  pad         VARCHAR2(100)
);

-- 고객 정보 테이블
CREATE TABLE dim_customer (
  cust_id     NUMBER PRIMARY KEY,
  region      VARCHAR2(8),
  grade       VARCHAR2(8)
);

-- 샘플 데이터 적재 (150만 건)
INSERT /*+ APPEND */ INTO sales
SELECT level                               AS sale_id,
       MOD(level, 500000)+1                AS cust_id,
       DATE '2025-01-01' + MOD(level, 365) AS sale_dt,
       ROUND(DBMS_RANDOM.VALUE(10,1000),2) AS amount,
       CASE MOD(level,4)
            WHEN 0 THEN 'OK'
            WHEN 1 THEN 'OK'
            WHEN 2 THEN 'CXL'
            ELSE 'HOLD' END                AS status,
       RPAD('x',100,'x')                   AS pad
FROM dual
CONNECT BY level <= 1500000;
COMMIT;

-- 고객 데이터 적재 (50만 건)
INSERT INTO dim_customer
SELECT level, 
       CASE MOD(level,4)
            WHEN 0 THEN 'APAC'
            WHEN 1 THEN 'EMEA'
            WHEN 2 THEN 'AMER'
            ELSE 'OTHR' END,
       CASE MOD(level,3)
            WHEN 0 THEN 'VIP'
            WHEN 1 THEN 'GOLD'
            ELSE 'SILV' END
FROM dual CONNECT BY level <= 500000;
COMMIT;

-- 인덱스 설계
CREATE INDEX ix_sales_cust_dt_amt
  ON sales(cust_id, sale_dt DESC, sale_id DESC, amount);

CREATE INDEX ix_sales_status ON sales(status);

-- 통계 정보 수집
BEGIN
  DBMS_STATS.GATHER_TABLE_STATS(USER,'SALES',cascade=>TRUE);
  DBMS_STATS.GATHER_TABLE_STATS(USER,'DIM_CUSTOMER',cascade=>TRUE);
END;
/
```

---

## 인덱스 프리페치 이해와 최적화

### 인덱스 프리페치의 개념
인덱스 프리페치는 인덱스 범위 스캔 시 다음 블록을 미리 읽어오는 최적화 기법입니다. 이는 연속적인 인덱스 탐색에서 발생할 수 있는 대기 시간을 줄여 전체적인 I/O 효율성을 향상시킵니다.

### 효율적인 인덱스 프리페치 조건
1. **정렬 순서 일치**: 쿼리의 ORDER BY 절이 인덱스 컬럼 순서와 일치할 때
2. **연속 범위 스캔**: 조건이 연속적인 키 범위를 지정할 때
3. **좋은 클러스터링 팩터**: 인덱스 키 순서가 물리적 저장 순서와 유사할 때

### 실전 예제: 고객별 최근 거래 조회
```sql
-- 효율적인 인덱스 프리페치 예제
SELECT /*+ index(s ix_sales_cust_dt_amt) */
       s.sale_id, s.sale_dt, s.amount
FROM sales s
WHERE s.cust_id = :cust_id
ORDER BY s.sale_dt DESC, s.sale_id DESC
FETCH FIRST 100 ROWS ONLY;
```
이 쿼리는 인덱스의 정렬 순서와 완벽히 일치하므로 인덱스 프리페치가 최적으로 동작합니다.

### 인덱스 Fast Full Scan 활용
테이블 접근 없이 인덱스만으로 쿼리를 완결할 수 있을 때 Fast Full Scan을 활용하면 멀티블록 읽기로 효율성을 극대화할 수 있습니다.

```sql
-- 상태별 거래 건수 집계
SELECT /*+ index_ffs(s ix_sales_status) */
       s.status, COUNT(*)
FROM sales s
GROUP BY s.status;
```

---

## 테이블 프리페치(Batched Table Access) 이해와 최적화

### 테이블 프리페치의 개념
테이블 프리페치는 인덱스 스캔으로 얻은 ROWID들을 모아 중복을 제거하고 정렬한 후, 테이블 블록을 배치로 읽어오는 기법입니다. 이는 실행 계획에서 `TABLE ACCESS BY INDEX ROWID BATCHED`로 표시됩니다.

### 테이블 프리페치 동작 메커니즘
1. 인덱스 범위 스캔을 통해 ROWID 수집
2. ROWID들을 블록 단위로 그룹화하고 중복 제거
3. 정렬된 블록 목록을 기준으로 배치 I/O 수행
4. 결과를 클라이언트에 반환

### 실전 예제: 지역별 최근 거래 상세 조회
```sql
-- 테이블 프리페치가 동작하는 예제
SELECT s.sale_id, s.sale_dt, s.amount, s.status
FROM sales s
JOIN dim_customer c ON c.cust_id = s.cust_id
WHERE c.region = 'APAC'
  AND s.sale_dt >= SYSDATE - 14
ORDER BY s.sale_dt DESC, s.sale_id DESC;
```

### 실행 계획 분석
```sql
-- BATCHED 접근 방식 확인
SELECT *
FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST +PEEKED_BINDS'));
```
효과적인 테이블 프리페치가 동작하면 `TABLE ACCESS BY INDEX ROWID BATCHED`가 실행 계획에 나타납니다.

---

## 프리페치 최적화를 위한 설계 패턴

### 정렬 순서와 인덱스 일치화
쿼리의 정렬 요구사항을 인덱스 설계에 반영해야 합니다.

```sql
-- 정렬 순서가 인덱스와 일치하는 예제
CREATE INDEX ix_sales_composite ON sales(
  cust_id, 
  sale_date DESC, 
  amount DESC, 
  sale_id
);

-- 해당 인덱스를 활용하는 쿼리
SELECT sale_id, sale_date, amount
FROM sales
WHERE cust_id = :cust_id
ORDER BY sale_date DESC, amount DESC
FETCH FIRST 50 ROWS ONLY;
```

### 커버링 인덱스 설계
자주 사용되는 컬럼을 인덱스에 포함시켜 테이블 접근을 최소화합니다.

```sql
-- 커버링 인덱스 예제
CREATE INDEX ix_sales_covering ON sales(
  cust_id,
  sale_date DESC
)
INCLUDE (amount, status);  -- SQL Server/PostgreSQL 스타일
-- Oracle에서는 컬럼을 인덱스 정의에 포함
CREATE INDEX ix_sales_covering_oracle ON sales(
  cust_id,
  sale_date DESC,
  amount,
  status,
  sale_id
);
```

### 클러스터링 팩터 개선
물리적 데이터 저장 순서를 인덱스 키 순서와 유사하게 재구성합니다.

```sql
-- 클러스터링 팩터 개선을 위한 재구성
-- 1. 기존 테이블 백업
CREATE TABLE sales_backup AS SELECT * FROM sales;

-- 2. 새로운 테이블 생성 (정렬된 순서로)
CREATE TABLE sales_new AS
SELECT * FROM sales
ORDER BY cust_id, sale_date, sale_id;

-- 3. 인덱스 재생성
CREATE INDEX ix_sales_new_cust_dt ON sales_new(cust_id, sale_date DESC, sale_id DESC);

-- 4. 통계 수집
BEGIN
  DBMS_STATS.GATHER_TABLE_STATS(USER,'SALES_NEW',cascade=>TRUE);
END;
/
```

---

## 성능 모니터링과 진단

### I/O 패턴 분석
```sql
-- 시스템 I/O 이벤트 분석
SELECT event, 
       total_waits, 
       time_waited_micro/1000000 as seconds_waited,
       ROUND(time_waited_micro/total_waits/1000, 2) as avg_ms_per_wait
FROM v$system_event
WHERE event LIKE 'db file%read%'
ORDER BY time_waited_micro DESC;
```

### SQL별 성능 프로파일링
```sql
-- 상위 I/O 소비 SQL 식별
SELECT sql_id, 
       plan_hash_value, 
       executions,
       buffer_gets,
       disk_reads,
       ROUND(buffer_gets/NULLIF(executions,0)) as avg_buffer_gets,
       ROUND(disk_reads/NULLIF(executions,0)) as avg_disk_reads,
       elapsed_time/1000000/NULLIF(executions,0) as avg_elapsed_sec
FROM v$sql
WHERE disk_reads > 1000
ORDER BY avg_disk_reads DESC
FETCH FIRST 20 ROWS ONLY;
```

### 실시간 세션 모니터링
```sql
-- 현재 실행 중인 세션의 I/O 패턴
SELECT s.sid, s.serial#, s.username, s.program,
       se.event, se.wait_time, se.seconds_in_wait,
       sq.sql_text
FROM v$session s
JOIN v$session_event se ON s.sid = se.sid
LEFT JOIN v$sql sq ON s.sql_id = sq.sql_id
WHERE se.event LIKE 'db file%read%'
  AND s.status = 'ACTIVE'
ORDER BY se.wait_time DESC;
```

---

## 일반적인 문제 패턴과 해결책

### 문제 1: 정렬 불일치로 인한 성능 저하
**증상**: 실행 계획에 SORT ORDER BY 작업이 포함되고, 디스크 정렬 발생

**해결책**:
```sql
-- 인덱스 재설계로 정렬 요구사항 반영
CREATE INDEX ix_sales_cust_amount ON sales(cust_id, amount DESC, sale_id);

-- 정렬 순서에 맞는 쿼리 작성
SELECT sale_id, amount, sale_date
FROM sales
WHERE cust_id = :cust_id
ORDER BY amount DESC  -- 인덱스 정렬 순서와 일치
FETCH FIRST 100 ROWS ONLY;
```

### 문제 2: 과도한 테이블 접근
**증상**: 많은 수의 `db file sequential read` 이벤트, 높은 평균 대기 시간

**해결책**:
```sql
-- 커버링 인덱스로 테이블 접근 제거
CREATE INDEX ix_sales_cust_cover ON sales(
  cust_id, 
  sale_date DESC, 
  sale_id, 
  amount, 
  status
);

-- 필요한 컬럼만 선택 (인덱스 컬럼만)
SELECT sale_id, sale_date, amount
FROM sales
WHERE cust_id = :cust_id
  AND sale_date >= SYSDATE - 30;
```

### 문제 3: 나쁜 클러스터링 팩터
**증상**: 인덱스 범위 스캔 후 많은 테이블 블록 접근, 높은 논리적 읽기

**해결책**:
```sql
-- 파티셔닝을 통한 물리적 그룹화
CREATE TABLE sales_partitioned
PARTITION BY RANGE (sale_date) (
  PARTITION p_2024_q1 VALUES LESS THAN (DATE '2024-04-01'),
  PARTITION p_2024_q2 VALUES LESS THAN (DATE '2024-07-01'),
  PARTITION p_2024_q3 VALUES LESS THAN (DATE '2024-10-01'),
  PARTITION p_2024_q4 VALUES LESS THAN (DATE '2025-01-01')
) AS SELECT * FROM sales;

-- 파티션 키를 활용한 효율적 접근
SELECT * FROM sales_partitioned
WHERE sale_date >= DATE '2024-10-01'
  AND sale_date < DATE '2025-01-01';
```

---

## 고급 최적화 기법

### 조건부 프리페치 전략
다양한 데이터 접근 패턴에 따라 다른 프리페치 전략을 적용합니다.

```sql
-- 작은 결과 집합: 네스티드 루프 조인
SELECT /*+ leading(c) use_nl(s) index(s) */ 
       c.cust_id, c.name, s.sale_id, s.amount
FROM customers c
JOIN sales s ON s.cust_id = c.cust_id
WHERE c.region = :region
  AND c.signup_date >= SYSDATE - 90
FETCH FIRST 100 ROWS ONLY;

-- 큰 결과 집합: 해시 조인
SELECT /*+ leading(c) use_hash(s) full(s) */ 
       c.region, COUNT(*) as transaction_count,
       SUM(s.amount) as total_amount
FROM customers c
JOIN sales s ON s.cust_id = c.cust_id
WHERE c.signup_date >= SYSDATE - 365
GROUP BY c.region;
```

### 적응적 실행 계획
데이터 분포에 따라 동적으로 최적의 접근 방식을 선택합니다.

```sql
-- 힌트를 통한 실행 계획 제어
SELECT /*+ 
    OPT_PARAM('_optimizer_adaptive_plans' 'false')
    OPT_PARAM('_optimizer_use_feedback' 'false')
    LEADING(t1 t2) 
    USE_HASH(t2) 
    FULL(t2) 
*/ 
    t1.id, t1.name, t2.value
FROM table1 t1
JOIN table2 t2 ON t2.table1_id = t1.id
WHERE t1.category = :category;
```

---

## 성능 테스트와 검증 방법

### A/B 테스트 프레임워크
```sql
-- 테스트 환경 설정
ALTER SESSION SET statistics_level = ALL;
ALTER SESSION SET events '10046 trace name context forever, level 8';

-- 기존 방식 테스트
SELECT /*+ TEST_A */ s.sale_id, s.sale_dt, s.amount
FROM sales s
WHERE s.cust_id = :cust_id
ORDER BY s.amount DESC
OFFSET 0 ROWS FETCH NEXT 50 ROWS ONLY;

-- 최적화된 방식 테스트
SELECT /*+ TEST_B index(s ix_sales_cust_dt_amt) */ 
       s.sale_id, s.sale_dt, s.amount
FROM sales s
WHERE s.cust_id = :cust_id
ORDER BY s.sale_dt DESC, s.sale_id DESC
FETCH FIRST 50 ROWS ONLY;

-- 트레이스 종료
ALTER SESSION SET events '10046 trace name context off';

-- 결과 비교
SELECT sql_id, plan_hash_value, executions,
       buffer_gets, disk_reads,
       elapsed_time/1000000 as elapsed_seconds
FROM v$sql
WHERE sql_text LIKE '%TEST_%'
ORDER BY sql_id;
```

### 자동화된 성능 모니터링
```sql
-- 성능 기준선 설정
CREATE TABLE performance_baseline (
    test_name VARCHAR2(100),
    sql_id VARCHAR2(13),
    avg_buffer_gets NUMBER,
    avg_disk_reads NUMBER,
    avg_elapsed_ms NUMBER,
    test_timestamp DATE DEFAULT SYSDATE
);

-- 정기적인 성능 측정
DECLARE
    v_buffer_gets NUMBER;
    v_disk_reads NUMBER;
    v_elapsed_ms NUMBER;
BEGIN
    -- 테스트 쿼리 실행
    SELECT buffer_gets, disk_reads, elapsed_time/1000
    INTO v_buffer_gets, v_disk_reads, v_elapsed_ms
    FROM v$sql
    WHERE sql_id = '&sql_id_to_test';
    
    -- 기준선 저장
    INSERT INTO performance_baseline 
    VALUES ('Daily Check', '&sql_id', 
            v_buffer_gets, v_disk_reads, v_elapsed_ms, SYSDATE);
    COMMIT;
END;
/
```

---

## 결론: 프리페치 최적화의 체계적 접근법

프리페치 최적화는 단순한 기술 적용이 아니라 데이터베이스 설계와 쿼리 작성의 근본적인 변화를 요구합니다. 효과적인 프리페치 구현을 위해 다음 원칙을 준수하세요:

### 1. 인덱스 설계 최적화
인덱스는 단순한 검색 도구가 아니라 데이터 접근 경로의 핵심입니다. 쿼리 패턴을 분석하여 가장 빈번하게 사용되는 필터 조건과 정렬 요구사항을 인덱스에 반영하세요. 특히:
- 자주 사용되는 조인 조건과 필터 컬럼을 선두 컬럼으로 구성
- ORDER BY, GROUP BY에 사용되는 컬럼을 인덱스에 포함
- 커버링 인덱스로 테이블 접근 최소화

### 2. 물리적 데이터 레이아웃 관리
클러스터링 팩터는 프리페치 효율성에 결정적인 영향을 미칩니다. 데이터의 물리적 저장 순서를 논리적 접근 패턴과 일치시켜야 합니다:
- 자주 함께 접근되는 데이터를 물리적으로 근접 저장
- 파티셔닝을 통한 데이터 세그먼트화
- 정기적인 재구성을 통한 클러스터링 팩터 유지

### 3. 쿼리 작성 패턴 개선
데이터베이스 엔진이 최적의 실행 계획을 선택할 수 있도록 쿼리를 명확하게 작성하세요:
- SARGable 조건 사용으로 인덱스 활용 보장
- 불필요한 컬럼 선택 회피
- 적절한 조인 방법 선택 (네스티드 루프 vs 해시 조인)

### 4. 지속적인 모니터링과 튜닝
성능 최적화는 일회성 작업이 아닌 지속적인 프로세스입니다:
- 성능 기준선 설정과 정기적 모니터링
- 실행 계획 변화 추적
- 실제 워크로드 패턴에 따른 동적 조정

### 5. 측정 기반 의사결정
직관이 아닌 데이터에 기반한 최적화 결정이 중요합니다:
- 모든 변경 사항 전후 성능 측정
- 다양한 시나리오에서의 성능 테스트
- 프로덕션 환경에서의 실제 성능 영향 평가

프리페치 최적화를 성공적으로 구현하면 I/O 효율성이 극대화되어 응답 시간이 단축되고, 시스템 자원 사용률이 개선되며, 사용자 경험이 전반적으로 향상됩니다. 이러한 개선은 단순한 기술적 성과를 넘어 비즈니스 가치 창출에 직접적으로 기여합니다.

최종적으로, 프리페치 최적화는 데이터베이스 성능 관리의 한 요소일 뿐이라는 점을 기억하세요. 인덱스 설계, 쿼리 최적화, 시스템 구성, 애플리케이션 아키텍처 등 다양한 요소가 조화를 이루어야 진정한 성능 개선을 달성할 수 있습니다.