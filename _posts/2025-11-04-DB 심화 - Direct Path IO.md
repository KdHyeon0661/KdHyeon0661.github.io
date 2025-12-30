---
layout: post
title: DB 심화 - Direct Path I/O
date: 2025-11-04 19:25:23 +0900
category: DB 심화
---
# Direct Path I/O 심층 이해: direct path read/write temp, direct path read, direct path write

> **핵심 요약**
> Direct Path I/O는 오라클 데이터베이스에서 대량 데이터 처리를 최적화하기 위해 버퍼 캐시를 우회하는 메커니즘입니다. 읽기 작업의 경우 `direct path read`(데이터 파일에서 직접 읽기)와 `direct path read temp`(임시 테이블스페이스에서 읽기)로 구분되며, 쓰기 작업은 `direct path write`(데이터 파일에 직접 쓰기)와 `direct path write temp`(임시 테이블스페이스에 쓰기)로 구분됩니다. 이 메커니즘은 대용량 순차 작업에서 처리량을 극대화하고 버퍼 캐시 오염을 방지하는 장점이 있지만, OLTP 환경에서는 캐시 재사용성을 저하시키고 I/O 경합을 유발할 수 있으므로 신중한 적용이 필요합니다.

---

## 실습 환경 구성

### 테스트 스키마 생성

```sql
-- 대용량 실습 테이블 생성
DROP TABLE big_sales PURGE;

CREATE TABLE big_sales (
  sale_id     NUMBER PRIMARY KEY,
  cust_id     NUMBER NOT NULL,
  sale_dt     DATE   NOT NULL,
  amount      NUMBER(12,2) NOT NULL,
  status      VARCHAR2(8)  NOT NULL,
  pad         VARCHAR2(200)
);

-- 300만 행의 샘플 데이터 삽입
INSERT /*+ APPEND */ INTO big_sales
SELECT level,
       MOD(level, 2000000)+1,
       DATE '2025-01-01' + MOD(level, 365),
       ROUND(DBMS_RANDOM.VALUE(1,1000),2),
       CASE MOD(level,4) 
         WHEN 0 THEN 'OK' 
         WHEN 1 THEN 'OK' 
         WHEN 2 THEN 'CXL' 
         ELSE 'HOLD' 
       END,
       RPAD('x',200,'x')
FROM dual
CONNECT BY level <= 3000000;  -- 300만행
COMMIT;

-- 조회 패턴을 위한 인덱스 생성
CREATE INDEX ix_bs_cust_dt ON big_sales(cust_id, sale_dt DESC, sale_id DESC);

-- 통계 정보 수집
BEGIN
  DBMS_STATS.GATHER_TABLE_STATS(
    USER,
    'BIG_SALES',
    cascade => TRUE, 
    method_opt => 'for all columns size skewonly'
  );
END;
/
```

### 모니터링 도구 설정

Direct Path I/O를 분석하기 위한 다양한 모니터링 도구들이 있습니다:

```sql
-- 1. 세션별 이벤트 모니터링
SELECT event, total_waits, time_waited_micro/1000000 as seconds
FROM v$session_event 
WHERE sid = SYS_CONTEXT('USERENV','SID')
AND event LIKE 'direct path%';

-- 2. 시스템 통계 확인
SELECT name, value
FROM v$sysstat
WHERE name IN (
  'physical reads direct',
  'physical writes direct',
  'physical reads direct temporary',
  'physical writes direct temporary'
);

-- 3. TEMP 공간 사용 현황
SELECT tablespace_name, 
       ROUND(used_blocks * block_size / 1024 / 1024, 2) as used_mb,
       ROUND(max_used_blocks * block_size / 1024 / 1024, 2) as max_used_mb
FROM v$tempseg_usage
WHERE session_addr = (SELECT saddr FROM v$session WHERE sid = SYS_CONTEXT('USERENV','SID'));

-- 4. Active Session History (ASH) 분석
SELECT event, COUNT(*) as sample_count
FROM v$active_session_history
WHERE sample_time > SYSTIMESTAMP - INTERVAL '10' MINUTE
  AND session_id = SYS_CONTEXT('USERENV','SID')
  AND event LIKE 'direct path%'
GROUP BY event
ORDER BY sample_count DESC;
```

---

## Direct Path I/O의 기본 개념

### 버퍼 캐시 우회 메커니즘

Direct Path I/O의 핵심은 전통적인 버퍼드 I/O와 달리 데이터베이스 버퍼 캐시(SGA)를 거치지 않고, 세션의 PGA(Private Global Area) 메모리에서 직접 스토리지 장치로 데이터를 읽고 쓰는 방식입니다. 이 접근법은 다음과 같은 특징을 가집니다:

1. **대용량 블록 처리**: 일반적으로 1MB 이상의 큰 I/O 단위로 작업을 수행합니다.
2. **비동기 I/O 지원**: I/O 요청과 처리를 분리하여 CPU 활용도를 높입니다.
3. **병렬 처리 최적화**: 여러 프로세스가 동시에 독립적인 I/O 작업을 수행할 수 있습니다.

### 성능 이점의 원리

Direct Path I/O가 빠를 수 있는 주요 이유는 다음과 같습니다:

1. **I/O 호출 횟수 감소**: 큰 블록 크기로 I/O를 수행하면 동일한 데이터량에 대해 더 적은 수의 I/O 호출이 발생합니다.
2. **컨텍스트 스위칭 최소화**: I/O 요청과 완료 사이의 컨텍스트 전환 횟수가 줄어듭니다.
3. **버퍼 캐시 경합 방지**: 대량 작업이 SGA 버퍼 캐시를 채우지 않으므로 OLTP 워크로드에 대한 영향을 최소화합니다.
4. **메모리 복사 감소**: 데이터가 PGA에서 직접 처리되므로 SGA와 PGA 사이의 불필요한 메모리 복사가 제거됩니다.

### Direct Path I/O가 선택되는 조건

오라클 옵티마이저는 다음과 같은 조건에서 Direct Path I/O를 선택할 가능성이 높습니다:

1. **대량 풀 스캔 작업**: 테이블의 상당 부분을 스캔해야 하는 경우
2. **낮은 데이터 재사용성**: 캐시에 유지될 가능성이 낮은 데이터에 대한 접근
3. **병렬 쿼리 실행**: 병렬 실행 서버를 사용하는 쿼리
4. **PGA 초과 작업**: 정렬이나 해시 조인 작업이 PGA 메모리를 초과하는 경우
5. **대량 데이터 적재**: `INSERT /*+ APPEND */`나 CTAS(Create Table As Select) 작업

---

## TEMP 기반 Direct I/O: direct path read/write temp

### 개념 이해

`direct path read temp`와 `direct path write temp` 이벤트는 오라클이 임시 테이블스페이스(TEMP)를 사용할 때 발생합니다. 이는 주로 다음과 같은 상황에서 발생합니다:

1. **정렬 작업**: ORDER BY, GROUP BY, DISTINCT 등의 작업에서 정렬 버퍼(PGA)를 초과할 때
2. **해시 조인**: 해시 테이블이 PGA 메모리를 초과할 때
3. **윈도우 함수**: 분석 함수 실행 시 중간 결과 저장이 필요할 때
4. **병렬 쿼리**: 병렬 실행 서버 간 데이터 교환 시

### TEMP 스필 현상 재현

의도적으로 TEMP 공간을 사용하도록 유도하는 쿼리 예제:

```sql
-- PGA 메모리 제한을 낮게 설정하여 TEMP 스필 유도
ALTER SESSION SET workarea_size_policy = MANUAL;
ALTER SESSION SET sort_area_size = 1048576;  -- 1MB로 제한

-- 대량 정렬 및 그룹화 작업 실행
SELECT /*+ MONITOR GATHER_PLAN_STATISTICS */
       cust_id, 
       COUNT(*) as transaction_count,
       SUM(amount) as total_amount,
       AVG(amount) as average_amount
FROM   big_sales
WHERE  sale_dt >= DATE '2025-01-01'
  AND  sale_dt < DATE '2025-02-01'
GROUP BY cust_id
ORDER BY total_amount DESC, cust_id;

-- 실행 계획 및 통계 확인
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(
  NULL, NULL, 'ALLSTATS LAST +MEMSTATS +PEEKED_BINDS'
));
```

### TEMP 사용 모니터링

```sql
-- 실시간 TEMP 사용량 모니터링
WITH temp_usage AS (
  SELECT 
    s.sid,
    s.serial#,
    s.username,
    s.program,
    u.tablespace,
    u.contents,
    u.segtype,
    ROUND(u.blocks * t.block_size / 1024 / 1024, 2) as used_mb,
    s.sql_id,
    s.event
  FROM v$session s
  JOIN v$tempseg_usage u ON s.saddr = u.session_addr
  JOIN dba_tablespaces t ON u.tablespace = t.tablespace_name
  WHERE s.type = 'USER'
)
SELECT * FROM temp_usage
ORDER BY used_mb DESC;

-- TEMP 관련 대기 이벤트 분석
SELECT 
  event,
  COUNT(*) as wait_count,
  ROUND(SUM(time_waited_micro)/1000000, 2) as total_wait_seconds,
  ROUND(AVG(time_waited_micro)/1000, 2) as avg_wait_ms
FROM v$session_event
WHERE event IN (
  'direct path read temp',
  'direct path write temp',
  'temp file write',
  'temp file read'
)
GROUP BY event
ORDER BY total_wait_seconds DESC;
```

### TEMP 스필 최적화 전략

1. **PGA 메모리 최적화**
   ```sql
   -- PGA 통계 모니터링
   SELECT 
     ROUND(pga_target_for_estimate/1024/1024) as pga_target_mb,
     estd_pga_cache_hit_percentage as cache_hit_percent,
     estd_extra_bytes_rw/1024/1024 as estd_extra_mb_rw,
     estd_overalloc_count
   FROM v$pga_target_advice
   ORDER BY pga_target_for_estimate;
   
   -- 최적 PGA 크기 설정
   ALTER SYSTEM SET pga_aggregate_target = 4G;
   ALTER SYSTEM SET pga_aggregate_limit = 8G;
   ```

2. **쿼리 재작성**
   ```sql
   -- 비효율적인 쿼리
   SELECT customer_id, SUM(amount)
   FROM big_sales
   GROUP BY customer_id
   ORDER BY SUM(amount) DESC;
   
   -- 개선된 쿼리 (TOP-N 제한)
   SELECT *
   FROM (
     SELECT customer_id, SUM(amount) as total
     FROM big_sales
     WHERE sale_dt >= ADD_MONTHS(TRUNC(SYSDATE), -12)
     GROUP BY customer_id
     ORDER BY SUM(amount) DESC
   )
   WHERE ROWNUM <= 100;
   ```

3. **인덱스 활용**
   ```sql
   -- 정렬 작업을 인덱스로 대체
   CREATE INDEX ix_sales_cust_amount ON big_sales(cust_id, amount DESC);
   
   -- 인덱스 범위 스캔 활용
   SELECT /*+ INDEX(b ix_sales_cust_amount) */
          cust_id, amount
   FROM   big_sales b
   WHERE  cust_id BETWEEN 1000 AND 2000
   ORDER BY amount DESC;
   ```

---

## 데이터 파일 직접 읽기: direct path read

### 개념 및 발생 조건

`direct path read` 이벤트는 오라클이 데이터 파일에서 직접 데이터를 읽을 때 발생합니다. 이는 다음과 같은 상황에서 주로 관찰됩니다:

1. **직렬 대량 스캔**: `_serial_direct_read` 파라미터가 TRUE이거나 자동으로 결정된 경우
2. **병렬 쿼리 실행**: 병렬 실행 서버가 데이터를 분할하여 읽을 때
3. **Exadata 환경**: Smart Scan이 활성화된 경우
4. **대용량 풀 스캔**: 테이블의 상당 부분을 액세스해야 하는 경우

### 직렬 Direct Path Read 테스트

```sql
-- Direct Path Read 강제 활성화 (테스트 목적)
ALTER SESSION SET "_serial_direct_read" = TRUE;

-- 대량 데이터 스캔 쿼리
SELECT /*+ MONITOR FULL(b) */
       COUNT(*) as row_count,
       SUM(amount) as total_amount,
       MIN(sale_dt) as earliest_date,
       MAX(sale_dt) as latest_date
FROM   big_sales b
WHERE  sale_dt BETWEEN DATE '2025-01-01' AND DATE '2025-03-31';

-- 이벤트 통계 확인
SELECT 
  event,
  total_waits,
  total_timeouts,
  time_waited_micro,
  average_wait_micro
FROM v$session_event
WHERE sid = SYS_CONTEXT('USERENV','SID')
  AND event LIKE 'direct path read%';
```

### 병렬 쿼리에서의 Direct Path Read

```sql
-- 병렬도 설정
ALTER SESSION SET parallel_degree_policy = MANUAL;

-- 병렬 쿼리 실행
SELECT /*+ PARALLEL(b, 8) FULL(b) MONITOR */
       status,
       COUNT(*) as count_per_status,
       SUM(amount) as total_per_status
FROM   big_sales b
WHERE  sale_dt >= ADD_MONTHS(TRUNC(SYSDATE, 'MM'), -6)
GROUP BY status
ORDER BY total_per_status DESC;

-- 병렬 실행 모니터링
SELECT 
  dfo_number,
  tq_id,
  server_type,
  process,
  num_rows,
  bytes,
  waits,
  timeout
FROM v$pq_tqstat
ORDER BY dfo_number, tq_id, server_type;
```

### Direct Path Read 최적화

1. **I/O 크기 조정**
   ```sql
   -- 다중 블록 읽기 크기 확인
   SHOW PARAMETER db_file_multiblock_read_count;
   
   -- I/O 크기 최적화
   ALTER SESSION SET db_file_multiblock_read_count = 128;
   ```

2. **파티셔닝 활용**
   ```sql
   -- 파티션된 테이블 생성
   CREATE TABLE sales_partitioned
   PARTITION BY RANGE (sale_dt) (
     PARTITION p202401 VALUES LESS THAN (DATE '2024-02-01'),
     PARTITION p202402 VALUES LESS THAN (DATE '2024-03-01'),
     PARTITION p202403 VALUES LESS THAN (DATE '2024-04-01'),
     PARTITION p202404 VALUES LESS THAN (DATE '2024-05-01')
   ) AS SELECT * FROM big_sales;
   
   -- 파티션 프루닝 활용
   SELECT SUM(amount)
   FROM sales_partitioned
   WHERE sale_dt BETWEEN DATE '2024-01-15' AND DATE '2024-02-15';
   ```

3. **인덱스 최적화**
   ```sql
   -- 커버링 인덱스 생성
   CREATE INDEX ix_sales_covering ON big_sales(sale_dt, cust_id, amount);
   
   -- 인덱스만 스캔으로 처리
   SELECT /*+ INDEX(b ix_sales_covering) */
          sale_dt, cust_id, SUM(amount)
   FROM   big_sales b
   WHERE  sale_dt >= DATE '2025-01-01'
   GROUP BY sale_dt, cust_id;
   ```

---

## 데이터 파일 직접 쓰기: direct path write

### 개념 및 활용 시나리오

`direct path write` 이벤트는 대량 데이터 쓰기 작업에서 발생하며, 다음과 같은 작업에서 주로 사용됩니다:

1. **CTAS (Create Table As Select)**
2. **INSERT /*+ APPEND */ 작업**
3. **병렬 DML 작업**
4. **SQL*Loader Direct Path 모드**
5. **인덱스 재구성 작업**

### Direct Path Write 작업 예제

```sql
-- NOLOGGING 모드로 CTAS 수행
CREATE TABLE big_sales_archive NOLOGGING
COMPRESS FOR QUERY HIGH
PARALLEL 8
AS 
SELECT * 
FROM big_sales 
WHERE sale_dt < DATE '2024-01-01';

-- APPEND 힌트를 사용한 대량 삽입
INSERT /*+ APPEND PARALLEL(big_sales, 4) */ 
INTO big_sales
SELECT 
  sale_id + 3000000,
  cust_id,
  sale_dt + INTERVAL '1' YEAR,
  amount,
  status,
  pad
FROM big_sales 
WHERE sale_dt >= DATE '2024-01-01'
  AND sale_dt < DATE '2024-02-01';
COMMIT;

-- Direct Path Write 모니터링
SELECT 
  event,
  total_waits,
  time_waited_micro,
  ROUND(time_waited_micro/1000000, 2) as wait_seconds
FROM v$session_event
WHERE sid = SYS_CONTEXT('USERENV','SID')
  AND event LIKE 'direct path write%';
```

### Direct Path Write 최적화 전략

1. **적절한 병렬도 설정**
   ```sql
   -- 테이블 병렬도 설정
   ALTER TABLE big_sales PARALLEL 8;
   
   -- 세션 병렬도 설정
   ALTER SESSION FORCE PARALLEL DML PARALLEL 4;
   ```

2. **스트라이핑 및 공간 관리**
   ```sql
   -- 테이블스페이스 스트라이핑 확인
   SELECT tablespace_name, file_name, bytes/1024/1024 as size_mb
   FROM dba_data_files
   WHERE tablespace_name = 'USERS'
   ORDER BY file_name;
   
   -- 균등한 공간 할당을 위한 초기 크기 설정
   CREATE TABLE large_table (
     id NUMBER,
     data VARCHAR2(4000)
   ) STORAGE (
     INITIAL 100M
     NEXT 50M
     MINEXTENTS 1
     MAXEXTENTS UNLIMITED
   ) TABLESPACE users;
   ```

3. **NOLOGGING 옵션 활용 (주의 필요)**
   ```sql
   -- NOLOGGING 모드 설정
   ALTER TABLE big_sales NOLOGGING;
   
   -- NOLOGGING 작업 수행
   INSERT /*+ APPEND NOLOGGING */ INTO big_sales
   SELECT * FROM staging_table;
   COMMIT;
   
   -- 로깅 모드 복원
   ALTER TABLE big_sales LOGGING;
   ```

---

## 혼합 환경에서의 Direct Path I/O 관리

### OLTP와 배치 작업의 균형

혼합 워크로드 환경에서는 Direct Path I/O의 적절한 사용이 중요합니다:

```sql
-- 시간대별 I/O 패턴 분석
SELECT 
  TO_CHAR(sample_time, 'HH24') as hour,
  event,
  COUNT(*) as sample_count,
  ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER(PARTITION BY TO_CHAR(sample_time, 'HH24')), 2) as percentage
FROM v$active_session_history
WHERE sample_time > SYSDATE - 1
  AND event LIKE 'direct path%'
  AND session_type = 'FOREGROUND'
GROUP BY TO_CHAR(sample_time, 'HH24'), event
ORDER BY hour, sample_count DESC;

-- 리소스 그룹을 통한 워크로드 관리
BEGIN
  DBMS_RESOURCE_MANAGER.CREATE_PENDING_AREA();
  
  -- 배치 작업 그룹 생성
  DBMS_RESOURCE_MANAGER.CREATE_CONSUMER_GROUP(
    consumer_group => 'BATCH_GROUP',
    comment => '대량 배치 작업용'
  );
  
  -- OLTP 작업 그룹 생성
  DBMS_RESOURCE_MANAGER.CREATE_CONSUMER_GROUP(
    consumer_group => 'OLTP_GROUP',
    comment => '온라인 트랜잭션 처리용'
  );
  
  -- 리소스 할당 계획 설정
  DBMS_RESOURCE_MANAGER.CREATE_PLAN(
    plan => 'MIXED_WORKLOAD_PLAN',
    comment => '혼합 워크로드 관리 계획'
  );
  
  DBMS_RESOURCE_MANAGER.SUBMIT_PENDING_AREA();
END;
/
```

### Adaptive Direct Path Read 조정

오라클 12c부터는 Adaptive Direct Path Read 기능이 도입되어 시스템이 자동으로 Direct Path Read 사용 여부를 결정합니다:

```sql
-- Adaptive Direct Path Read 상태 확인
SELECT 
  name,
  value,
  isdefault,
  description
FROM v$parameter
WHERE name LIKE '%direct%read%'
   OR name LIKE '%serial%direct%';

-- 통계 기반 의사 결정 모니터링
SELECT 
  sql_id,
  executions,
  buffer_gets,
  disk_reads,
  direct_writes,
  ROUND(disk_reads/NULLIF(executions,0), 2) as reads_per_exec,
  ROUND(buffer_gets/NULLIF(executions,0), 2) as gets_per_exec
FROM v$sql
WHERE executions > 100
  AND disk_reads > 10000
ORDER BY disk_reads DESC
FETCH FIRST 10 ROWS ONLY;
```

---

## 성능 문제 진단 및 해결 절차

### 체계적인 진단 방법

1. **기본 정보 수집**
   ```sql
   -- 시스템 기본 정보
   SELECT * FROM v$version;
   SHOW PARAMETER db_block_size;
   SHOW PARAMETER pga_aggregate_target;
   SHOW PARAMETER sga_target;
   
   -- I/O 관련 파라미터
   SHOW PARAMETER db_file_multiblock_read_count;
   SHOW PARAMETER parallel_degree_policy;
   SHOW PARAMETER workarea_size_policy;
   ```

2. **성능 문제 특정화**
   ```sql
   -- 최근 성능 이슈 있는 SQL 식별
   SELECT 
     sql_id,
     executions,
     elapsed_time,
     cpu_time,
     buffer_gets,
     disk_reads,
     direct_writes,
     ROUND(elapsed_time/NULLIF(executions,0)/1000, 2) as avg_ms_per_exec
   FROM v$sqlstats
   WHERE elapsed_time > 10000000  -- 10초 이상 소요
   ORDER BY elapsed_time DESC
   FETCH FIRST 20 ROWS ONLY;
   ```

3. **상세 실행 계획 분석**
   ```sql
   -- 특정 SQL의 실행 계획 확인
   SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(
     '&sql_id', 
     NULL, 
     'ADVANCED ALLSTATS LAST +MEMSTATS +PEEKED_BINDS'
   ));
   ```

4. **I/O 패턴 분석**
   ```sql
   -- 데이터 파일별 I/O 통계
   SELECT 
     file#,
     name,
     phyrds as physical_reads,
     phywrts as physical_writes,
     ROUND(phyblkrd/NULLIF(phyrds,0), 2) as avg_blocks_per_read,
     ROUND(phyblkwrt/NULLIF(phywrts,0), 2) as avg_blocks_per_write
   FROM v$datafile
   ORDER BY phyrds + phywrts DESC;
   ```

### 일반적인 문제 패턴 및 해결책

**패턴 1: 과도한 TEMP 사용**
- **증상**: `direct path read/write temp` 이벤트가 지속적으로 높음
- **원인**: PGA 메모리 부족, 비효율적인 정렬/해시 작업
- **해결**: 
  ```sql
  -- PGA 크기 조정
  ALTER SYSTEM SET pga_aggregate_target = 8G;
  
  -- 쿼리 재작성
  SELECT /*+ NO_USE_HASH_AGGREGATION */ ...
  ```

**패턴 2: 직렬 Direct Path Read 과다**
- **증상**: OLTP 쿼리에서 `direct path read` 발생
- **원인**: `_serial_direct_read` 파라미터 잘못 설정
- **해결**:
  ```sql
  -- 파라미터 조정
  ALTER SYSTEM SET "_serial_direct_read" = FALSE;
  
  -- 인덱스 최적화
  CREATE INDEX ...;
  ```

**패턴 3: 병렬 작업 경합**
- **증상**: 병렬 쿼리 실행 시 성능 저하
- **원인**: 과도한 병렬도, I/O 서브시스템 병목
- **해결**:
  ```sql
  -- 적절한 병렬도 설정
  ALTER SESSION SET parallel_degree_limit = 8;
  
  -- 병렬 조정
  SELECT /*+ PARALLEL(4) OPT_PARAM('parallel_degree_policy', 'MANUAL') */ ...
  ```

---

## 모범 사례와 권장 설정

### 일반적인 권장 사항

1. **PGA 메모리 설정**
   ```sql
   -- PGA 통계 기반 크기 결정
   SELECT 
     ROUND(pga_target_for_estimate/1024/1024/1024, 2) as target_gb,
     estd_pga_cache_hit_percentage as hit_percent,
     estd_extra_bytes_rw/1024/1024/1024 as extra_gb
   FROM v$pga_target_advice
   ORDER BY pga_target_for_estimate;
   
   -- 일반적인 규칙: SGA의 20-50% 크기로 PGA 설정
   ALTER SYSTEM SET pga_aggregate_target = 4G;
   ALTER SYSTEM SET pga_aggregate_limit = 8G;
   ```

2. **병렬 실행 설정**
   ```sql
   -- 자동 병렬도 결정
   ALTER SYSTEM SET parallel_degree_policy = AUTO;
   ALTER SYSTEM SET parallel_min_time_threshold = 30; -- 30초 이상 쿼리만 병렬화
   
   -- 병렬 서버 풀 크기
   ALTER SYSTEM SET parallel_servers_target = 64;
   ALTER SYSTEM SET parallel_max_servers = 128;
   ```

3. **I/O 최적화 설정**
   ```sql
   -- 다중 블록 읽기 크기
   ALTER SYSTEM SET db_file_multiblock_read_count = 128;
   
   -- 비동기 I/O 활성화
   ALTER SYSTEM SET filesystemio_options = SETALL;
   ALTER SYSTEM SET disk_asynch_io = TRUE;
   ```

### 환경별 최적화 전략

**데이터 웨어하우스 환경**
```sql
-- 대용량 배치 작업에 최적화
ALTER SYSTEM SET "_serial_direct_read" = TRUE;
ALTER SYSTEM SET parallel_degree_policy = AUTO;
ALTER SYSTEM SET parallel_adaptive_multi_user = FALSE;
```

**OLTP 환경**
```sql
-- OLTP 워크로드에 최적화
ALTER SYSTEM SET "_serial_direct_read" = FALSE;
ALTER SYSTEM SET parallel_degree_policy = MANUAL;
ALTER SYSTEM SET parallel_min_servers = 0;
```

**혼합 워크로드 환경**
```sql
-- 시간대별 자동 조정
BEGIN
  -- 업무 시간에는 OLTP 모드
  DBMS_SCHEDULER.CREATE_JOB(
    job_name => 'DAYTIME_SETTINGS',
    job_type => 'PLSQL_BLOCK',
    job_action => 'BEGIN
      EXECUTE IMMEDIATE ''ALTER SYSTEM SET "_serial_direct_read" = FALSE'';
      EXECUTE IMMEDIATE ''ALTER SYSTEM SET parallel_degree_policy = MANUAL'';
    END;',
    start_date => TRUNC(SYSDATE) + INTERVAL '8' HOUR,
    repeat_interval => 'FREQ=DAILY',
    enabled => TRUE
  );
  
  -- 야간에는 배치 모드
  DBMS_SCHEDULER.CREATE_JOB(
    job_name => 'NIGHTTIME_SETTINGS',
    job_type => 'PLSQL_BLOCK',
    job_action => 'BEGIN
      EXECUTE IMMEDIATE ''ALTER SYSTEM SET "_serial_direct_read" = TRUE'';
      EXECUTE IMMEDIATE ''ALTER SYSTEM SET parallel_degree_policy = AUTO'';
    END;',
    start_date => TRUNC(SYSDATE) + INTERVAL '20' HOUR,
    repeat_interval => 'FREQ=DAILY',
    enabled => TRUE
  );
END;
/
```

---

## 결론

Direct Path I/O는 오라클 데이터베이스에서 대량 데이터 처리 성능을 극대화하는 강력한 메커니즘입니다. `direct path read/write`와 `direct path read/write temp`는 각각 데이터 파일과 임시 테이블스페이스에 대한 직접 I/O 작업을 나타냅니다.

이러한 메커니즘의 효과적인 활용을 위해서는 작업의 특성과 환경을 정확히 이해해야 합니다. 데이터 웨어하우스 환경에서는 Direct Path I/O를 적극적으로 활용하여 처리량을 극대화하는 것이 유리한 반면, OLTP 환경에서는 캐시 재사용성을 고려하여 신중하게 적용해야 합니다.

성공적인 Direct Path I/O 최적화를 위해서는 다음과 같은 접근법이 필요합니다:

1. **정량적 측정**: AWR, ASH, SQL 모니터 등을 활용하여 실제 성능 영향을 측정합니다.
2. **단계적 최적화**: PGA 메모리 조정, 쿼리 재작성, 인덱스 최적화 순으로 접근합니다.
3. **환경 고려**: 데이터베이스 용도(OLTP, DW, 혼합)에 맞는 설정을 적용합니다.
4. **지속적 모니터링**: 성능 변화를 지속적으로 추적하고 필요시 조정합니다.

가장 중요한 원칙은 "모든 상황에 맞는 단일 최적 설정은 없다"는 점입니다. 각 환경의 특성, 워크로드 패턴, 하드웨어 구성에 맞춰 유연하게 접근하고, 데이터 기반 의사결정을 통해 지속적으로 최적화를 진행해야 합니다.

Direct Path I/O는 성능 최적화 도구 상자에서 강력한 도구 중 하나일 뿐, 이것이 모든 문제의 해결책은 아닙니다. 인덱스 설계, 쿼리 최적화, 파티셔닝 전략 등과 함께 종합적으로 접근할 때 진정한 성능 향상을 달성할 수 있습니다.