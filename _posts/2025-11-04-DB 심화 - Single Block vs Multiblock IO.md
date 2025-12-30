---
layout: post
title: DB 심화 - Single Block vs Multiblock I/O
date: 2025-11-04 17:25:23 +0900
category: DB 심화
---
# Single Block vs Multiblock I/O: 데이터베이스 I/O 최적화의 핵심 이해

> **핵심 요약**
> 데이터베이스 성능 최적화에서 Single Block I/O와 Multiblock I/O의 차이를 이해하는 것은 매우 중요합니다. Single Block I/O는 한 번에 하나의 블록(일반적으로 8KB)만 읽는 방식으로, 인덱스를 통한 랜덤 접근에 적합합니다. 반면 Multiblock I/O는 한 번에 여러 블록을 동시에 읽는 방식으로, 대량 데이터의 순차적 스캔에 효율적입니다. 올바른 I/O 전략 선택은 워크로드의 특성에 따라 결정되어야 합니다.

---

## 실습 환경 설정

```sql
-- 대용량 테이블 생성 (파티셔닝 포함)
DROP TABLE big_orders PURGE;

CREATE TABLE big_orders (
  order_id     NUMBER PRIMARY KEY,
  cust_id      NUMBER NOT NULL,
  order_dt     DATE   NOT NULL,
  status       VARCHAR2(10),
  amount       NUMBER(12,2),
  pad          VARCHAR2(200)
)
PARTITION BY RANGE (order_dt) (
  PARTITION p2025q1 VALUES LESS THAN (DATE '2025-04-01'),
  PARTITION p2025q2 VALUES LESS THAN (DATE '2025-07-01'),
  PARTITION p2025q3 VALUES LESS THAN (DATE '2025-10-01'),
  PARTITION p2025q4 VALUES LESS THAN (DATE '2026-01-01')
);

-- 샘플 데이터 적재 (100만 건)
INSERT /*+ APPEND */ INTO big_orders
SELECT level,
       MOD(level, 2000000)+1,
       DATE '2025-01-01' + MOD(level, 365),
       CASE MOD(level,5) WHEN 0 THEN 'NEW' WHEN 1 THEN 'OK' WHEN 2 THEN 'RTN'
                         WHEN 3 THEN 'CXL' ELSE 'HOLD' END,
       ROUND(DBMS_RANDOM.VALUE(1,1000),2),
       RPAD('x',200,'x')
FROM dual
CONNECT BY level <= 1000000;
COMMIT;

-- 인덱스 생성
CREATE INDEX ix_bo_cust_dt ON big_orders(cust_id, order_dt DESC, order_id DESC);
CREATE INDEX ix_bo_status  ON big_orders(status);

-- 통계 정보 수집
BEGIN
  DBMS_STATS.GATHER_TABLE_STATS(
    USER, 
    'BIG_ORDERS', 
    cascade=>TRUE, 
    method_opt=>'for all columns size skewonly'
  );
END;
/
```

---

## 기본 개념 이해

### Single Block I/O
Single Block I/O는 데이터베이스가 한 번에 하나의 블록만 읽는 방식입니다. 이 방식의 특징은 다음과 같습니다:

- **블록 크기**: 일반적으로 8KB (데이터베이스 설정에 따라 다를 수 있음)
- **주요 사용 사례**: 인덱스 범위 스캔, 고유 인덱스 스캔, ROWID를 통한 테이블 접근
- **대기 이벤트**: `db file sequential read`
- **적합한 워크로드**: OLTP(Online Transaction Processing) 환경의 랜덤 접근

Single Block I/O의 핵심 장점은 필요한 데이터만 정확하게 읽을 수 있다는 점입니다. 하지만 대량의 데이터를 처리할 때는 많은 I/O 호출이 발생하여 성능이 저하될 수 있습니다.

### Multiblock I/O
Multiblock I/O는 데이터베이스가 한 번에 여러 블록을 동시에 읽는 방식입니다:

- **I/O 크기**: MBRC(Multiblock Read Count) × 블록 크기 (예: 16 × 8KB = 128KB)
- **주요 사용 사례**: 테이블 풀 스캔, 파티션 풀 스캔, 인덱스 Fast Full Scan
- **대기 이벤트**: `db file scattered read` (버퍼 캐시 사용), `direct path read` (직접 경로)
- **적합한 워크로드**: 데이터 웨어하우징, 리포트 생성, 대량 배치 처리

Multiblock I/O의 주요 장점은 I/O 호출 횟수를 줄여 처리량(throughput)을 향상시킨다는 점입니다. 하지만 불필요한 데이터까지 읽을 수 있다는 단점도 있습니다.

---

## I/O 패턴별 실행 계획 분석

### Single Block I/O 패턴의 실행 계획
Single Block I/O는 일반적으로 인덱스 기반 접근에서 발생합니다:

```sql
-- 단일 고객의 최근 주문 조회
EXPLAIN PLAN FOR
SELECT order_id, amount
FROM big_orders
WHERE cust_id = 12345
  AND order_dt >= DATE '2025-09-01';

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);
```

예상 실행 계획:
```
-------------------------------------------------------------------------------
| Id  | Operation                           | Name           | Rows  | Bytes |
-------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                    |                |    10 |   200 |
|   1 |  TABLE ACCESS BY INDEX ROWID BATCHED| BIG_ORDERS     |    10 |   200 |
|   2 |   INDEX RANGE SCAN                  | IX_BO_CUST_DT  |    10 |       |
-------------------------------------------------------------------------------
```

이 실행 계획에서 `INDEX RANGE SCAN`과 `TABLE ACCESS BY INDEX ROWID BATCHED`는 Single Block I/O 패턴을 나타냅니다.

### Multiblock I/O 패턴의 실행 계준
Multiblock I/O는 풀 스캔이나 파티션 스캔에서 발생합니다:

```sql
-- 분기별 판매 통계 조회
EXPLAIN PLAN FOR
SELECT status, COUNT(*), SUM(amount)
FROM big_orders
WHERE order_dt >= DATE '2025-07-01'
GROUP BY status;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);
```

예상 실행 계획:
```
--------------------------------------------------------------------------
| Id  | Operation           | Name       | Rows  | Bytes | Cost | Time   |
--------------------------------------------------------------------------
|   0 | SELECT STATEMENT    |            |     5 |    65 | 1234 | 00:00:01|
|   1 |  HASH GROUP BY      |            |     5 |    65 | 1234 | 00:00:01|
|   2 |   PARTITION RANGE ALL|            | 250K |  3175K| 1230 | 00:00:01|
|   3 |    TABLE ACCESS FULL| BIG_ORDERS | 250K |  3175K| 1230 | 00:00:01|
--------------------------------------------------------------------------
```

`TABLE ACCESS FULL` 또는 `PARTITION RANGE ALL` 뒤에 `TABLE ACCESS FULL`이 있는 실행 계획은 Multiblock I/O 패턴을 나타냅니다.

---

## 성능 모니터링과 진단

### I/O 이벤트 모니터링
현재 시스템의 I/O 패턴을 분석하려면 다음 쿼리를 사용합니다:

```sql
-- 시스템 전체 I/O 이벤트 분석
SELECT event, 
       total_waits, 
       ROUND(time_waited_micro/1000000, 2) as seconds_waited,
       ROUND(time_waited_micro/total_waits/1000, 2) as avg_ms_per_wait
FROM v$system_event
WHERE event IN ('db file sequential read', 
                'db file scattered read', 
                'direct path read',
                'direct path read temp')
ORDER BY time_waited_micro DESC;
```

### 세션별 I/O 분석
특정 세션의 I/O 활동을 모니터링하려면:

```sql
-- 활성 세션의 I/O 이벤트
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

### SQL별 I/O 프로파일링
상위 I/O를 소비하는 SQL을 식별하려면:

```sql
-- 상위 I/O 소비 SQL
SELECT sql_id, 
       plan_hash_value, 
       executions,
       buffer_gets,
       disk_reads,
       ROUND(buffer_gets/NULLIF(executions,0)) as avg_logical_reads,
       ROUND(disk_reads/NULLIF(executions,0)) as avg_physical_reads,
       elapsed_time/1000000/NULLIF(executions,0) as avg_elapsed_seconds
FROM v$sql
WHERE disk_reads > 1000
  AND parsing_schema_name = USER
ORDER BY avg_physical_reads DESC
FETCH FIRST 10 ROWS ONLY;
```

---

## Single Block I/O 최적화 전략

### 커버링 인덱스 활용
커버링 인덱스를 사용하면 테이블 접근 없이 인덱스만으로 쿼리를 완결할 수 있습니다:

```sql
-- 커버링 인덱스 생성
CREATE INDEX ix_bo_cust_cover ON big_orders(
  cust_id, 
  order_dt DESC, 
  order_id DESC, 
  amount, 
  status
);

-- 커버링 인덱스를 활용한 쿼리
SELECT order_id, order_dt, amount, status
FROM big_orders
WHERE cust_id = :cust_id
  AND order_dt >= :start_date
ORDER BY order_dt DESC
FETCH FIRST 100 ROWS ONLY;
```

### 클러스터링 팩터 개선
데이터의 물리적 저장 순서를 논리적 접근 패턴과 일치시키면 I/O 효율성이 향상됩니다:

```sql
-- 클러스터링 팩터 개선을 위한 테이블 재구성
CREATE TABLE big_orders_reorganized AS
SELECT * FROM big_orders
ORDER BY cust_id, order_dt, order_id;

-- 인덱스 재생성
CREATE INDEX ix_bo_cust_dt_new ON big_orders_reorganized(
  cust_id, 
  order_dt DESC, 
  order_id DESC
);

-- 통계 재수집
BEGIN
  DBMS_STATS.GATHER_TABLE_STATS(
    USER, 
    'BIG_ORDERS_REORGANIZED', 
    cascade=>TRUE
  );
END;
/
```

### 효율적인 조인 전략
적절한 조인 방법 선택으로 Single Block I/O를 최소화합니다:

```sql
-- 네스티드 루프 조인 (소규모 데이터셋에 적합)
SELECT /*+ leading(c) use_nl(o) index(o) */ 
       c.cust_id, c.customer_name, o.order_id, o.amount
FROM customers c
JOIN big_orders o ON o.cust_id = c.cust_id
WHERE c.region = 'APAC'
  AND c.signup_date >= SYSDATE - 90
FETCH FIRST 100 ROWS ONLY;
```

---

## Multiblock I/O 최적화 전략

### 파티션 프루닝 활용
파티션 프루닝을 통해 필요한 데이터만 읽도록 제한합니다:

```sql
-- 파티션 프루닝이 적용되는 쿼리
SELECT status, COUNT(*), SUM(amount)
FROM big_orders
WHERE order_dt >= DATE '2025-10-01'
  AND order_dt < DATE '2026-01-01'
GROUP BY status;
```

### Direct Path Read 최적화
대량 데이터 처리 시 Direct Path Read를 활용하면 버퍼 캐시 오염을 방지할 수 있습니다:

```sql
-- Direct Path Read 활성화 (세션 레벨)
ALTER SESSION SET "_serial_direct_read" = ALWAYS;

-- 대량 집계 쿼리
SELECT /*+ full(b) parallel(b 4) */ 
       cust_id, COUNT(*), SUM(amount)
FROM big_orders b
WHERE order_dt >= DATE '2025-01-01'
GROUP BY cust_id
HAVING COUNT(*) > 100;
```

### 병렬 처리 활용
대용량 데이터 처리를 위해 병렬 실행을 적절히 활용합니다:

```sql
-- 병렬 처리 활용
SELECT /*+ parallel(b 8) full(b) */ 
       EXTRACT(YEAR FROM order_dt) as order_year,
       EXTRACT(MONTH FROM order_dt) as order_month,
       status,
       COUNT(*) as order_count,
       SUM(amount) as total_amount
FROM big_orders b
WHERE order_dt >= DATE '2025-01-01'
GROUP BY EXTRACT(YEAR FROM order_dt),
         EXTRACT(MONTH FROM order_dt),
         status
ORDER BY order_year, order_month, status;
```

---

## 실전 시나리오: I/O 전략 선택 가이드

### 시나리오 1: 실시간 주문 조회 시스템 (OLTP)

**요구사항**:
- 고객별 최근 주문 50건 조회
- 평균 응답 시간 < 100ms
- 초당 1000건 이상 처리

**최적화 방안**:
```sql
-- Single Block I/O 최적화 전략
CREATE INDEX ix_orders_oltp ON orders(
  customer_id, 
  order_date DESC, 
  order_id DESC, 
  amount, 
  status
);

-- 커버링 인덱스 활용 쿼리
SELECT /*+ index(o ix_orders_oltp) */ 
       order_id, order_date, amount, status
FROM orders o
WHERE customer_id = :customer_id
ORDER BY order_date DESC
FETCH FIRST 50 ROWS ONLY;
```

### 시나리오 2: 월간 판매 리포트 (배치)

**요구사항**:
- 월별 판매 통계 생성
- 백만 건 이상 처리
- 야간 배치 실행

**최적화 방안**:
```sql
-- Multiblock I/O 최적화 전략
SELECT /*+ parallel(o 8) full(o) */ 
       EXTRACT(YEAR FROM order_date) as year,
       EXTRACT(MONTH FROM order_date) as month,
       product_category,
       COUNT(*) as order_count,
       SUM(amount) as total_sales,
       AVG(amount) as avg_order_value
FROM orders o
WHERE order_date >= ADD_MONTHS(TRUNC(SYSDATE, 'MM'), -12)
GROUP BY EXTRACT(YEAR FROM order_date),
         EXTRACT(MONTH FROM order_date),
         product_category
ORDER BY year, month, product_category;
```

---

## 일반적인 문제와 해결책

### 문제 1: 과도한 Single Block I/O

**증상**:
- 높은 `db file sequential read` 대기 시간
- 느린 OLTP 응답 시간
- 인덱스 스캔 후 많은 테이블 접근

**해결책**:
```sql
-- 1. 커버링 인덱스 도입
CREATE INDEX ix_customer_orders_cover ON orders(
  customer_id, 
  order_date, 
  order_id, 
  total_amount, 
  status
);

-- 2. 클러스터링 팩터 개선
-- 데이터 재구성 또는 IOT(Index Organized Table) 고려

-- 3. 배치 처리 최적화
-- 여러 ROWID를 모아 배치 처리
```

### 문제 2: 비효율적인 Multiblock I/O

**증상**:
- 불필요한 전체 테이블 스캔
- 파티션 프루닝 실패
- 적절하지 않은 병렬 처리

**해결책**:
```sql
-- 1. 파티션 프루닝 검증
EXPLAIN PLAN FOR
SELECT * FROM orders
WHERE order_date >= DATE '2025-01-01'
  AND order_date < DATE '2025-02-01';

-- 2. 적절한 병렬도 설정
ALTER SESSION FORCE PARALLEL DML PARALLEL 4;
ALTER SESSION FORCE PARALLEL QUERY PARALLEL 4;

-- 3. Direct Path Read 최적화
ALTER SESSION SET "_serial_direct_read" = AUTO;
```

---

## 성능 테스트와 측정 방법

### A/B 테스트 프레임워크
```sql
-- 성능 측정 환경 설정
ALTER SESSION SET statistics_level = ALL;
ALTER SESSION SET events '10046 trace name context forever, level 8';

-- Single Block I/O 전략 테스트
SELECT /*+ MONITOR SINGLE_BLOCK_TEST */ 
       order_id, amount
FROM big_orders
WHERE cust_id = 12345
  AND order_dt BETWEEN DATE '2025-09-01' AND DATE '2025-09-30';

-- Multiblock I/O 전략 테스트  
SELECT /*+ MONITOR MULTIBLOCK_TEST full(b) */ 
       status, COUNT(*), SUM(amount)
FROM big_orders b
WHERE order_dt >= DATE '2025-09-01'
  AND order_dt < DATE '2025-10-01'
GROUP BY status;

-- 트레이스 종료
ALTER SESSION SET events '10046 trace name context off';

-- 결과 비교
SELECT sql_id, 
       SUBSTR(sql_text, 1, 50) as sql_snippet,
       executions,
       buffer_gets,
       disk_reads,
       elapsed_time/1000000 as elapsed_seconds
FROM v$sql
WHERE sql_text LIKE '%MONITOR%'
ORDER BY sql_id;
```

### 자동화된 성능 모니터링
```sql
-- 성능 기준선 테이블 생성
CREATE TABLE performance_baseline_io (
    test_name VARCHAR2(100),
    sql_id VARCHAR2(13),
    test_type VARCHAR2(20), -- 'SINGLE' or 'MULTI'
    buffer_gets NUMBER,
    disk_reads NUMBER,
    elapsed_ms NUMBER,
    test_timestamp TIMESTAMP DEFAULT SYSTIMESTAMP
);

-- 성능 측정 프로시저
CREATE OR REPLACE PROCEDURE measure_io_performance AS
    v_buffer_gets NUMBER;
    v_disk_reads NUMBER;
    v_elapsed_ms NUMBER;
    v_sql_id VARCHAR2(13);
BEGIN
    -- Single Block I/O 테스트
    SELECT buffer_gets, disk_reads, elapsed_time/1000, sql_id
    INTO v_buffer_gets, v_disk_reads, v_elapsed_ms, v_sql_id
    FROM v$sql
    WHERE sql_text LIKE '%SINGLE_BLOCK_TEST%'
      AND ROWNUM = 1;
    
    INSERT INTO performance_baseline_io 
    VALUES ('Daily Single Block Test', v_sql_id, 'SINGLE',
            v_buffer_gets, v_disk_reads, v_elapsed_ms, SYSTIMESTAMP);
    
    -- Multiblock I/O 테스트
    SELECT buffer_gets, disk_reads, elapsed_time/1000, sql_id
    INTO v_buffer_gets, v_disk_reads, v_elapsed_ms, v_sql_id
    FROM v$sql
    WHERE sql_text LIKE '%MULTIBLOCK_TEST%'
      AND ROWNUM = 1;
    
    INSERT INTO performance_baseline_io 
    VALUES ('Daily Multiblock Test', v_sql_id, 'MULTI',
            v_buffer_gets, v_disk_reads, v_elapsed_ms, SYSTIMESTAMP);
    
    COMMIT;
END;
/
```

---

## 결론: 데이터베이스 I/O 최적화의 종합적 접근

효율적인 데이터베이스 성능 관리를 위해서는 Single Block과 Multiblock I/O의 특성을 이해하고, 워크로드에 맞게 최적의 전략을 선택해야 합니다.

### 핵심 원칙

1. **워크로드 특성에 맞는 전략 선택**
   - OLTP 시스템: Single Block I/O 최적화에 중점
   - 데이터 웨어하우스: Multiblock I/O 효율 극대화
   - 하이브리드 시스템: 두 전략의 적절한 조합

2. **근본적인 설계 최적화**
   - 인덱스 설계: 커버링 인덱스, 복합 인덱스 활용
   - 테이블 설계: 파티셔닝, 클러스터링, IOT 고려
   - 물리적 저장: 데이터 배치 최적화

3. **실행 계획 관리**
   - 통계 정보 정확성 유지
   - 실행 계획 안정성 확보
   - 필요 시 힌트를 통한 실행 계획 제어

4. **시스템 수준 최적화**
   - 버퍼 캐시 크기 적절화
   - I/O 서브시스템 최적화
   - 병렬 처리 적절한 활용

### 성공적인 I/O 최적화의 지표

- **응답 시간 개선**: 사용자 경험 향상
- **처리량 증가**: 단위 시간당 더 많은 작업 처리
- **자원 효율성**: CPU, 메모리, 디스크 I/O의 균형적 활용
- **확장성**: 데이터 증가에 따른 성능 저하 최소화

### 지속적인 개선 사이클

1. **측정**: 현재 성능 기준선 확립
2. **분석**: 병목 지점 및 개선 가능성 식별
3. **실험**: 다양한 최적화 기법 테스트
4. **적용**: 검증된 개선사항 운영 환경 반영
5. **모니터링**: 변경 효과 지속적 추적

데이터베이스 I/O 최적화는 단순한 기술적 작업을 넘어 비즈니스 요구사항과 시스템 특성을 종합적으로 고려한 의사결정 과정입니다. 올바른 I/O 전략 선택과 지속적인 최적화를 통해 안정적이고 효율적인 데이터베이스 시스템을 구축할 수 있습니다.

최종적으로, I/O 최적화의 성공은 기술적 지표뿐만 아니라 비즈니스 가치 창출이라는 관점에서 평가되어야 합니다. 더 빠른 응답 시간, 더 높은 처리량, 더 낮은 운영 비용은 모두 궁극적으로 더 나은 비즈니스 성과로 이어집니다.