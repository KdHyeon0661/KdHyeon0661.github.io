---
layout: post
title: DB 심화 - SQL Trace
date: 2025-10-24 15:25:23 +0900
category: DB 심화
---
# Oracle SQL Trace

## SQL Trace란 무엇인가?

SQL Trace는 Oracle 데이터베이스의 **가장 상세한 성능 진단 도구**로, 특정 세션이나 그룹의 모든 SQL 실행 내역을 파일에 기록합니다. 이를 통해 단순히 '느리다'라는 증상을 넘어, **정확히 어디서(어느 SQL, 어느 호출 단계), 무엇을(어떤 대기 이벤트), 왜(어떤 바인드 값과 실행 계획으로)** 시간이 소요되었는지를 미시적으로 파악할 수 있습니다.

### AWR, ASH, SQL Trace 비교
- **AWR**: 1시간 단위 통계 요약 (거시적 분석)
- **ASH**: 1초 단위 세션 활동 샘플링 (중간 수준 분석)
- **SQL Trace**: 개별 SQL 실행의 모든 호출과 대기 상세 기록 (미시적 분석)

**핵심 가치**: SQL Trace는 성능 문제의 **법의학적 증거**를 제공합니다. 특히 재현이 어려운 간헐적 문제나 특정 시나리오에서만 발생하는 문제를 해결할 때 필수적입니다.

---

## SQL Trace의 작동 원리와 아키텍처

### 내부 메커니즘 이해
SQL Trace는 Oracle의 이벤트 추적 시스템을 기반으로 동작합니다. 특히 이벤트 번호 **10046**이 SQL Trace를 제어합니다.

```
[애플리케이션 요청]
    ↓
[Oracle 세션에서 SQL 실행]
    ↓
[Trace 활성화 시] → [Trace 버퍼에 상세 기록]
    ↓                    ↓
[정상 처리]        [주기적으로 디스크에 쓰기]
    ↓                    ↓
[결과 반환]        [.trc 파일 생성]
```

### Trace 파일 저장 위치와 관리
Trace 파일은 Automatic Diagnostic Repository(ADR) 구조 하에 저장됩니다.

```sql
-- 현재 세션의 Trace 파일 위치 확인
SELECT value AS trace_file_path,
       LENGTH(value) AS path_length
FROM   v$diag_info 
WHERE  name = 'Default Trace File';

-- ADR 베이스 디렉토리 확인
SELECT value AS adr_home
FROM   v$diag_info 
WHERE  name = 'ADR Home';

-- 최근 생성된 Trace 파일 목록 (DBA 권한)
SELECT d.value || '/' || i.instance_name || '_ora_' || p.spid || '.trc' AS trace_file
FROM   v$diag_info d,
       v$instance i,
       v$process p,
       v$session s
WHERE  d.name = 'Diag Trace'
  AND  s.sid = SYS_CONTEXT('USERENV', 'SID')
  AND  p.addr = s.paddr;
```

**실전 팁**: Trace 파일 명명 규칙은 `{INSTANCE_NAME}_ora_{SPID}_{TRACE_IDENTIFIER}.trc` 형식입니다. `TRACEFILE_IDENTIFIER` 세션 파라미터를 설정하면 파일명에 식별자가 추가되어 찾기 쉽습니다.

---

## 다양한 Trace 활성화 방법과 실전 활용 시나리오

### 1. 자기 세션 추적 (가장 안전한 기본 방법)

#### DBMS_MONITOR 패키지 사용 (권장)
```sql
-- 고급 옵션 포함 전체 추적 시작
BEGIN
    DBMS_MONITOR.SESSION_TRACE_ENABLE(
        session_id     => NULL,      -- NULL: 현재 세션
        serial_num     => NULL,
        waits          => TRUE,      -- 대기 이벤트 기록
        binds          => TRUE,      -- 바인드 변수 값 기록
        plan_stat      => 'ALL_EXECUTIONS', -- 모든 실행 통계 수집
        timeout_sec    => 1800       -- 30분 후 자동 종료 (안전장치)
    );
END;
/

-- 예제: 성능 문제가 의심되는 SQL 실행
DECLARE
    l_start_time TIMESTAMP;
    l_end_time   TIMESTAMP;
    l_order_count NUMBER;
BEGIN
    l_start_time := SYSTIMESTAMP;
    
    -- 문제가 되는 복잡한 쿼리
    SELECT /*+ ORDERED USE_NL(o) INDEX(c CUSTOMERS_REGION_IDX) */ 
           COUNT(DISTINCT o.order_id)
    INTO   l_order_count
    FROM   customers c,
           orders o,
           order_items oi
    WHERE  c.customer_id = o.customer_id
      AND  o.order_id = oi.order_id
      AND  c.region = 'APAC'
      AND  o.order_date BETWEEN DATE '2024-01-01' AND DATE '2024-06-30'
      AND  oi.product_category = 'ELECTRONICS'
    GROUP BY c.country;
    
    l_end_time := SYSTIMESTAMP;
    DBMS_OUTPUT.PUT_LINE('처리시간: ' || 
        EXTRACT(SECOND FROM (l_end_time - l_start_time)) || '초');
END;
/

-- Trace 종료
EXEC DBMS_MONITOR.SESSION_TRACE_DISABLE();
```

#### 고급 옵션 상세 설명
- **`plan_stat` 옵션**:
  - `'FIRST_EXECUTION'`: 첫 번째 실행 계획 통계만 수집
  - `'ALL_EXECUTIONS'`: 모든 실행의 통계 수집 (반복 실행 분석에 유용)
  - `NULL`: 실행 통계 미수집
  
- **`timeout_sec`**: 실수로 Trace를 끄지 않아도 자동 종료되는 안전장치

- **`max_size_mb`**: Trace 파일 최대 크기 제한 (디스크 공간 보호)

### 2. 다른 세션 추적 (운영 환경 문제 분석)

#### 대상 세션 식별과 안전한 추적
```sql
-- 문제 세션 식별을 위한 진단 쿼리
SELECT s.sid,
       s.serial#,
       s.username,
       s.program,
       s.module,
       s.action,
       s.sql_id,
       s.prev_sql_id,
       s.event AS current_wait,
       s.state,
       s.seconds_in_wait,
       s.blocking_session,
       ROUND(p.pga_used_mem/1024/1024, 2) AS pga_used_mb,
       ROUND(p.pga_alloc_mem/1024/1024, 2) AS pga_alloc_mb
FROM   v$session s,
       v$process p
WHERE  s.type = 'USER'
  AND  s.username NOT IN ('SYS', 'SYSTEM')
  AND  p.addr = s.paddr
  AND  s.status = 'ACTIVE'
  AND  s.state != 'WAITING'  -- CPU 사용 중인 세션
ORDER BY s.last_call_et DESC
FETCH FIRST 10 ROWS ONLY;

-- 특정 모듈/액션을 실행 중인 세션 찾기
SELECT sid, serial#, sql_id, client_info
FROM   v$session
WHERE  module = 'PAYROLL_PROCESSING'
  AND  action = 'CALCULATE_BONUS'
  AND  status = 'ACTIVE';

-- 선택한 세션에 Trace 활성화
BEGIN
    -- 먼저 세션이 아직 존재하는지 확인
    FOR rec IN (SELECT 1 FROM v$session 
                WHERE sid = &SID AND serial# = &SERIAL#)
    LOOP
        DBMS_MONITOR.SESSION_TRACE_ENABLE(
            session_id => &SID,
            serial_num => &SERIAL#,
            waits      => TRUE,
            binds      => FALSE,  -- 운영 환경에서는 보안상 바인드 값 제외
            plan_stat  => 'ALL_EXECUTIONS'
        );
        DBMS_OUTPUT.PUT_LINE('Trace started for SID: ' || &SID || 
                            ', SERIAL#: ' || &SERIAL#);
        EXIT;
    END LOOP;
END;
/

-- Trace 모니터링 (실시간으로 파일 크기 확인)
SELECT sid, serial#, sql_trace, sql_trace_waits, sql_trace_binds
FROM   v$session
WHERE  sid = &SID;

-- 지정 시간 후 또는 문제 재현 후 Trace 종료
BEGIN
    DBMS_LOCK.SLEEP(300);  -- 5분 대기
    DBMS_MONITOR.SESSION_TRACE_DISABLE(&SID, &SERIAL#);
    DBMS_OUTPUT.PUT_LINE('Trace stopped for SID: ' || &SID);
END;
/
```

### 3. 서비스/모듈/액션 단위 추적 (애플리케이션 기능 수준 분석)

#### 애플리케이션 계측 설정
```sql
-- 애플리케이션에서 모듈/액션 설정 예시 (PL/SQL)
BEGIN
    -- 배치 작업 시작 시
    DBMS_APPLICATION_INFO.SET_MODULE(
        module_name => 'MONTHLY_REPORT',
        action_name => 'GENERATE_SALES_SUMMARY'
    );
    
    -- 작업 단계별 업데이트
    DBMS_APPLICATION_INFO.SET_ACTION('EXTRACT_DATA');
    -- 데이터 추출 작업 수행...
    
    DBMS_APPLICATION_INFO.SET_ACTION('TRANSFORM_DATA');
    -- 데이터 변환 작업 수행...
    
    DBMS_APPLICATION_INFO.SET_ACTION('LOAD_DATA');
    -- 데이터 적재 작업 수행...
    
    -- 작업 완료 시
    DBMS_APPLICATION_INFO.SET_MODULE(NULL, NULL);
END;
/

-- Java 애플리케이션 예시
// OracleDataSource 설정 시
OracleDataSource ds = new OracleDataSource();
ds.setURL("jdbc:oracle:thin:@localhost:1521:ORCL");
ds.setUser("app_user");
ds.setPassword("password");

// 연결 후 모듈/액션 설정
Connection conn = ds.getConnection();
((OracleConnection)conn).setEndToEndMetrics(
    new String[]{"MONTHLY_REPORT"},  // 모듈
    new int[]{OracleConnection.END_TO_END_STATE_INDEX_MAX}, 
    15  // 우선순위
);
```

#### 서비스 수준 Trace 활성화
```sql
-- 특정 서비스의 모든 활동 추적
BEGIN
    DBMS_MONITOR.SERV_MOD_ACT_TRACE_ENABLE(
        service_name  => 'ERP_SERVICE',
        waits         => TRUE,
        binds         => FALSE,  -- 보안상 생산 환경에서는 FALSE
        instance_name => NULL     -- NULL: 모든 인스턴스
    );
END;
/

-- 특정 모듈의 특정 액션만 추적 (정밀 타겟팅)
BEGIN
    DBMS_MONITOR.SERV_MOD_ACT_TRACE_ENABLE(
        service_name  => 'ERP_SERVICE',
        module_name   => 'ORDER_PROCESSING',
        action_name   => 'VALIDATE_PAYMENT',
        waits         => TRUE,
        binds         => TRUE,    -- 개발 환경에서만
        instance_name => 'ORCL1'  -- 특정 RAC 인스턴스만
    );
END;
/

-- Trace 상태 확인
SELECT trace_type,
       primary_id,
       qualifier_id1,
       qualifier_id2,
       waits,
       binds
FROM   dba_enabled_traces
ORDER BY trace_type, primary_id;

-- Trace 종료
BEGIN
    DBMS_MONITOR.SERV_MOD_ACT_TRACE_DISABLE(
        service_name  => 'ERP_SERVICE',
        module_name   => 'ORDER_PROCESSING',
        action_name   => 'VALIDATE_PAYMENT'
    );
END;
/
```

### 4. 클라이언트 식별자 기반 추적 (멀티테넌트 환경)

```sql
-- 클라이언트별 세션 구분
BEGIN
    -- 애플리케이션에서 고객별 식별자 설정
    DBMS_SESSION.SET_IDENTIFIER('CUSTOMER_12345_ORDER_67890');
    
    -- 또는 테넌트별 식별자
    DBMS_SESSION.SET_IDENTIFIER('TENANT_ACME_USER_JSMITH');
END;
/

-- 특정 클라이언트의 모든 활동 추적
BEGIN
    DBMS_MONITOR.CLIENT_ID_TRACE_ENABLE(
        client_id => 'CUSTOMER_12345_ORDER_67890',
        waits     => TRUE,
        binds     => TRUE
    );
END;
/

-- 클라이언트 식별자별 세션 확인
SELECT sid, serial#, username, client_identifier
FROM   v$session
WHERE  client_identifier IS NOT NULL
ORDER BY client_identifier;

-- Trace 종료
EXEC DBMS_MONITOR.CLIENT_ID_TRACE_DISABLE('CUSTOMER_12345_ORDER_67890');
```

---

## TKPROF 활용: Trace 파일 분석과 리포트 생성

### TKPROF 기본 사용법
```bash
# 기본 명령어
tkprof source_trace_file.trc output_report.prf sys=no waits=yes sort=exeela

# 고급 옵션
tkprof app_trace.trc detailed_report.txt \
  sys=no \
  waits=yes \
  sort='(exeela,fchela,prsela)' \
  print=100 \
  explain=apps/apps_password@prod_db \
  table=sys.plan_table \
  record=problem_sql.sql
```

### TKPROF 옵션 상세 설명
- **`sys=no`**: SYS 사용자의 SQL을 리포트에서 제외 (일반적으로 권장)
- **`waits=yes`**: 대기 이벤트 정보 포함
- **`sort=exeela,fchela`**: 실행 시간, 페치 시간 순으로 정렬
- **`print=N`**: 상위 N개의 SQL만 출력
- **`explain`**: 실행 계획 자동 생성 (실제 실행 계획이 아닌 예측 계획)
- **`aggregate`**: 동일 SQL의 여러 실행을 집계
- **`record`**: 문제 SQL을 별도 파일로 추출

### TKPROF 리포트 해석 실전 예제
```
SQL ID: 8a5d9sjh23k4m
SELECT /*+ INDEX(c) */ c.customer_name, o.order_date, SUM(oi.quantity)
FROM customers c, orders o, order_items oi
WHERE c.customer_id = o.customer_id
  AND o.order_id = oi.order_id
  AND c.region = :region
  AND o.order_date BETWEEN :start_date AND :end_date
GROUP BY c.customer_name, o.order_date

call     count       cpu    elapsed     disk    query current    rows
------- ------  -------- ---------- -------- -------- ------- ------
Parse        1      0.01       0.01        0        0       0      0
Execute      1      0.00       0.00        0        0       0      0
Fetch     1000     12.45      45.23   125000   150000     100  50000
------- ------  -------- ---------- -------- -------- ------- ------
total     1002     12.46      45.24   125000   150000     100  50000

Misses in library cache during parse: 1
Optimizer mode: ALL_ROWS
Parsing user id: 108
Number of plan statistics captured: 1

Rows (1st) Rows (avg) Rows (max) Row Source Operation
---------- ---------- ---------- ---------------------------------------------------
     50000      50000      50000 HASH GROUP BY (cr=150000 pr=125000 pw=0 time=45230 us)
   1000000    1000000    1000000  HASH JOIN (cr=120000 pr=100000 pw=0 time=40210 us)
   1000000    1000000    1000000   TABLE ACCESS FULL CUSTOMERS (cr=50000 pr=40000 pw=0 time=15020 us)
  50000000   50000000   50000000   TABLE ACCESS FULL ORDERS (cr=70000 pr=60000 pw=0 time=25190 us)

Elapsed times include waiting on following events:
  Event waited on                             Times   Max. Wait  Total Waited
  ----------------------------------------   Waited  ----------  ------------
  db file scattered read                        250        0.12         18.45
  direct path read temp                         100        0.08          6.32
  enq: TX - row lock contention                   5        0.85          2.14
```

**핵심 분석 포인트:**
1. **Fetch 시간(45.23초)이 전체 시간의 99%** → 클라이언트에 결과 전달이 병목
2. **디스크 읽기(125,000 블록) 과다** → 버퍼 캐시 효율성 문제
3. **Temp 영역 사용(`direct path read temp`)** → 정렬/해시 작업의 PGA 부족
4. **행 잠금 경합(`enq: TX`)** → 동시성 문제

---

## 고급 Trace 기법과 최적화

### 1. 계층적 Trace: 문제 범위 점진적 좁히기
```sql
-- 1단계: 서비스 수준에서 문제 발생 패턴 관찰
EXEC DBMS_MONITOR.SERV_MOD_ACT_TRACE_ENABLE('ERP_SERVICE', NULL, NULL, TRUE, FALSE);

-- 2단계: 문제 모듈 식별 후 모듈 수준 추적
EXEC DBMS_MONITOR.SERV_MOD_ACT_TRACE_DISABLE('ERP_SERVICE', NULL, NULL);
EXEC DBMS_MONITOR.SERV_MOD_ACT_TRACE_ENABLE('ERP_SERVICE', 'ORDER_PROCESSING', NULL, TRUE, TRUE);

-- 3단계: 특정 액션으로 집중
EXEC DBMS_MONITOR.SERV_MOD_ACT_TRACE_DISABLE('ERP_SERVICE', 'ORDER_PROCESSING', NULL);
EXEC DBMS_MONITOR.SERV_MOD_ACT_TRACE_ENABLE('ERP_SERVICE', 'ORDER_PROCESSING', 'CALCULATE_TAX', TRUE, TRUE);

-- 4단계: 최종적으로 문제 세션만 상세 추적
SELECT sid, serial# FROM v$session 
WHERE module='ORDER_PROCESSING' AND action='CALCULATE_TAX';

EXEC DBMS_MONITOR.SESSION_TRACE_ENABLE(&PROBLEM_SID, &PROBLEM_SERIAL#, TRUE, TRUE, 'ALL_EXECUTIONS');
```

### 2. 자동화된 Trace 수집 스크립트
```sql
-- Trace 자동 수집 패키지
CREATE OR REPLACE PACKAGE trace_manager AS
    PROCEDURE start_service_trace(
        p_service_name  VARCHAR2,
        p_duration_sec  NUMBER DEFAULT 300
    );
    
    PROCEDURE stop_all_traces;
    
    PROCEDURE analyze_trace_files(
        p_days_back NUMBER DEFAULT 1
    );
END trace_manager;
/

CREATE OR REPLACE PACKAGE BODY trace_manager AS
    
    g_trace_start_time TIMESTAMP;
    
    PROCEDURE start_service_trace(
        p_service_name  VARCHAR2,
        p_duration_sec  NUMBER DEFAULT 300
    ) IS
    BEGIN
        -- 기존 Trace 중지
        DBMS_MONITOR.SERV_MOD_ACT_TRACE_DISABLE(p_service_name, NULL, NULL);
        
        -- 새 Trace 시작
        DBMS_MONITOR.SERV_MOD_ACT_TRACE_ENABLE(
            service_name => p_service_name,
            waits        => TRUE,
            binds        => FALSE,
            instance_name => NULL
        );
        
        g_trace_start_time := SYSTIMESTAMP;
        
        -- 자동 종료 작업 예약
        DBMS_SCHEDULER.CREATE_JOB(
            job_name        => 'STOP_TRACE_' || p_service_name,
            job_type        => 'PLSQL_BLOCK',
            job_action      => 'BEGIN trace_manager.stop_service_trace(''' || 
                              p_service_name || '''); END;',
            start_date      => SYSTIMESTAMP + NUMTODSINTERVAL(p_duration_sec, 'SECOND'),
            enabled         => TRUE,
            auto_drop       => TRUE
        );
        
        DBMS_OUTPUT.PUT_LINE('Trace started for service: ' || p_service_name);
    END start_service_trace;
    
    PROCEDURE stop_service_trace(p_service_name VARCHAR2) IS
    BEGIN
        DBMS_MONITOR.SERV_MOD_ACT_TRACE_DISABLE(p_service_name, NULL, NULL);
        DBMS_OUTPUT.PUT_LINE('Trace stopped for service: ' || p_service_name);
    END stop_service_trace;
    
    PROCEDURE stop_all_traces IS
    BEGIN
        FOR rec IN (SELECT primary_id FROM dba_enabled_traces)
        LOOP
            BEGIN
                DBMS_MONITOR.SERV_MOD_ACT_TRACE_DISABLE(rec.primary_id, NULL, NULL);
            EXCEPTION
                WHEN OTHERS THEN
                    NULL;  -- 다른 Trace 유형은 무시
            END;
        END LOOP;
        DBMS_OUTPUT.PUT_LINE('All traces stopped.');
    END stop_all_traces;
    
    PROCEDURE analyze_trace_files(p_days_back NUMBER DEFAULT 1) IS
        l_trace_dir VARCHAR2(4000);
    BEGIN
        SELECT value INTO l_trace_dir
        FROM v$diag_info
        WHERE name = 'Diag Trace';
        
        -- Trace 파일 분석용 외부 테이블 생성
        EXECUTE IMMEDIATE '
            CREATE OR REPLACE DIRECTORY trace_dir AS ''' || l_trace_dir || '''
        ';
        
        -- 추가 분석 로직 구현...
    END analyze_trace_files;
    
END trace_manager;
/
```

### 3. RAC 환경에서의 분산 Trace 관리
```sql
-- RAC 전체 Trace 활성화
BEGIN
    FOR inst IN (SELECT instance_number FROM gv$instance)
    LOOP
        DBMS_MONITOR.SERV_MOD_ACT_TRACE_ENABLE(
            service_name  => 'GLOBAL_SERVICE',
            instance_name => 'ORCL' || inst.instance_number,
            waits         => TRUE,
            binds         => FALSE
        );
    END LOOP;
END;
/

-- 특정 인스턴스의 Trace 파일만 수집
SELECT inst_id, 
       LISTAGG(trace_file, ', ') WITHIN GROUP (ORDER BY trace_file) AS trace_files
FROM (
    SELECT i.inst_id,
           d.value || '/' || i.instance_name || '_ora_' || p.spid || '.trc' AS trace_file
    FROM   gv$diag_info d,
           gv$instance i,
           gv$process p,
           gv$session s
    WHERE  d.name = 'Diag Trace'
      AND  d.inst_id = i.inst_id
      AND  s.inst_id = i.inst_id
      AND  s.sid = SYS_CONTEXT('USERENV', 'SID')
      AND  p.inst_id = s.inst_id
      AND  p.addr = s.paddr
      AND  i.inst_id = &TARGET_INSTANCE
)
GROUP BY inst_id;
```

---

## Trace 관련 성능 영향과 모니터링

### Trace 오버헤드 측정
```sql
-- Trace 활성화 전후 성능 비교
CREATE TABLE trace_impact_test AS
SELECT level AS id,
       'Sample Data ' || level AS data,
       SYSDATE + level/86400 AS timestamp
FROM dual
CONNECT BY level <= 100000;

-- Trace 비활성화 상태 성능 측정
SET TIMING ON
SELECT /*+ NO_TRACE */ COUNT(*) FROM trace_impact_test WHERE id BETWEEN 1 AND 50000;
-- 결과: 0.45초

-- Trace 활성화 후 성능 측정
EXEC DBMS_MONITOR.SESSION_TRACE_ENABLE(waits=>TRUE, binds=>TRUE);
SELECT /*+ WITH_TRACE */ COUNT(*) FROM trace_impact_test WHERE id BETWEEN 1 AND 50000;
-- 결과: 0.52초 (약 15% 오버헤드)
EXEC DBMS_MONITOR.SESSION_TRACE_DISABLE;

-- 오버헤드 요인 분석
SELECT stat_name,
       value AS trace_off,
       LEAD(value) OVER (ORDER BY stat_name) AS trace_on,
       ROUND((LEAD(value) OVER (ORDER BY stat_name) - value) / NULLIF(value, 0) * 100, 2) AS overhead_pct
FROM (
    SELECT 'CPU_TIME' AS stat_name, SUM(cpu_time) AS value
    FROM v$sql
    WHERE sql_text LIKE '%NO_TRACE%'
    UNION ALL
    SELECT 'CPU_TIME_TRACE' AS stat_name, SUM(cpu_time) AS value
    FROM v$sql
    WHERE sql_text LIKE '%WITH_TRACE%'
);
```

### 실시간 Trace 모니터링
```sql
-- 활성 Trace 모니터링 대시보드
WITH trace_info AS (
    SELECT 
        s.sid,
        s.serial#,
        s.username,
        s.program,
        s.module,
        s.action,
        s.sql_id,
        s.sql_trace,
        s.sql_trace_waits,
        s.sql_trace_binds,
        p.spid,
        t.trace_type,
        t.primary_id,
        t.qualifier_id1,
        t.qualifier_id2
    FROM v$session s
    LEFT JOIN v$process p ON p.addr = s.paddr
    LEFT JOIN dba_enabled_traces t ON (
        (t.trace_type = 'SESSION' AND t.primary_id = s.sid AND t.qualifier_id1 = s.serial#) OR
        (t.trace_type = 'CLIENT_ID' AND s.client_identifier = t.primary_id) OR
        (t.trace_type = 'SERVICE_MODULE_ACTION' AND 
         s.service_name = t.primary_id AND
         (t.qualifier_id1 IS NULL OR s.module = t.qualifier_id1) AND
         (t.qualifier_id2 IS NULL OR s.action = t.qualifier_id2))
    )
    WHERE s.type = 'USER'
)
SELECT 
    ti.sid,
    ti.serial#,
    ti.username,
    ti.program,
    ti.module,
    ti.action,
    ti.sql_id,
    CASE WHEN ti.sql_trace = 'ENABLED' THEN '✓' ELSE '✗' END AS trace_enabled,
    ti.trace_type,
    ti.primary_id,
    f.value AS trace_file
FROM trace_info ti
LEFT JOIN v$diag_info f ON f.name = 'Default Trace File'
WHERE ti.sql_trace = 'ENABLED' OR ti.trace_type IS NOT NULL
ORDER BY ti.sid;
```

---

## Trace 데이터 분석과 문제 해결 워크플로우

### 종합적인 문제 진단 워크플로우

```sql
-- 1. 문제 범위 식별
SELECT wait_class, event, COUNT(*) * 10 AS est_seconds  -- 샘플링 기반 추정
FROM v$active_session_history
WHERE sample_time > SYSDATE - INTERVAL '15' MINUTE
  AND session_type = 'FOREGROUND'
GROUP BY wait_class, event
ORDER BY est_seconds DESC;

-- 2. 대상 선정 및 Trace 활성화
BEGIN
    -- CASE 1: 특정 대기 이벤트가 많을 때
    IF &TOP_EVENT LIKE 'enq: TX%' THEN
        DBMS_MONITOR.SERV_MOD_ACT_TRACE_ENABLE(
            service_name => &PROBLEM_SERVICE,
            waits => TRUE,
            binds => TRUE
        );
    -- CASE 2: 특정 SQL이 문제일 때
    ELSIF &TOP_SQL_ID IS NOT NULL THEN
        FOR rec IN (SELECT sid, serial# FROM v$session WHERE sql_id = &TOP_SQL_ID)
        LOOP
            DBMS_MONITOR.SESSION_TRACE_ENABLE(
                rec.sid, rec.serial#, TRUE, TRUE, 'ALL_EXECUTIONS'
            );
        END LOOP;
    END IF;
END;
/

-- 3. Trace 파일 수집 및 분석 자동화
DECLARE
    l_trace_files DBMS_SQLCLOUD_OBJECTLIST;
BEGIN
    -- Trace 파일 찾기
    SELECT directory_path || '/' || filename
    BULK COLLECT INTO l_trace_files
    FROM (
        SELECT d.value AS directory_path,
               i.instance_name || '_ora_' || p.spid || '.trc' AS filename
        FROM v$diag_info d,
             v$instance i,
             v$process p,
             v$session s
        WHERE d.name = 'Diag Trace'
          AND s.sid = SYS_CONTEXT('USERENV', 'SID')
          AND p.addr = s.paddr
    );
    
    -- 각 파일 분석
    FOR i IN 1..l_trace_files.COUNT LOOP
        -- TKPROF 실행 (OS 명령어 호출)
        DBMS_SCHEDULER.CREATE_JOB(
            job_name => 'ANALYZE_TRACE_' || i,
            job_type => 'EXECUTABLE',
            job_action => '/usr/bin/tkprof',
            number_of_arguments => 6,
            enabled => FALSE
        );
        
        DBMS_SCHEDULER.SET_JOB_ARGUMENT_VALUE('ANALYZE_TRACE_' || i, 1, l_trace_files(i));
        DBMS_SCHEDULER.SET_JOB_ARGUMENT_VALUE('ANALYZE_TRACE_' || i, 2, 
            SUBSTR(l_trace_files(i), 1, LENGTH(l_trace_files(i))-4) || '.prf');
        DBMS_SCHEDULER.SET_JOB_ARGUMENT_VALUE('ANALYZE_TRACE_' || i, 3, 'sys=no');
        DBMS_SCHEDULER.SET_JOB_ARGUMENT_VALUE('ANALYZE_TRACE_' || i, 4, 'waits=yes');
        DBMS_SCHEDULER.SET_JOB_ARGUMENT_VALUE('ANALYZE_TRACE_' || i, 5, 'sort=exeela');
        DBMS_SCHEDULER.SET_JOB_ARGUMENT_VALUE('ANALYZE_TRACE_' || i, 6, 'print=100');
        
        DBMS_SCHEDULER.ENABLE('ANALYZE_TRACE_' || i);
    END LOOP;
END;
/

-- 4. 분석 결과 기반 조치 결정
/*
Trace 분석 결과 패턴별 조치:
1. Parse 시간 높음 → 바인드 변수 적용, 커서 공유 최적화
2. Execute 시간 높음 → 실행 계획 최적화, 인덱스 튜닝
3. Fetch 시간 높음 → 페이징 적용, 불필요한 컬럼 제거
4. 특정 대기 이벤트 집중 → 해당 리소스 최적화
*/
```

---

## 보안과 규정 준수 고려사항

### 개인정보 보호를 위한 Trace 관리
```sql
-- 민감정보 마스킹을 위한 Trace 정책
CREATE OR REPLACE PACKAGE secure_trace AS
    
    PROCEDURE enable_trace_with_masking(
        p_sid      NUMBER,
        p_serial   NUMBER,
        p_masking_patterns DBMS_SQL.VARCHAR2_TABLE
    );
    
    FUNCTION mask_sensitive_data(
        p_sql_text VARCHAR2,
        p_patterns DBMS_SQL.VARCHAR2_TABLE
    ) RETURN VARCHAR2;
    
END secure_trace;
/

CREATE OR REPLACE PACKAGE BODY secure_trace AS
    
    g_masking_patterns DBMS_SQL.VARCHAR2_TABLE := DBMS_SQL.VARCHAR2_TABLE(
        '\d{4}-\d{4}-\d{4}-\d{4}',  -- 신용카드
        '\d{3}-\d{2}-\d{4}',        -- SSN
        '\d{3}-\d{3}-\d{4}',        -- 전화번호
        '[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}'  -- 이메일
    );
    
    PROCEDURE enable_trace_with_masking(
        p_sid      NUMBER,
        p_serial   NUMBER,
        p_masking_patterns DBMS_SQL.VARCHAR2_TABLE
    ) IS
        l_context VARCHAR2(100);
    BEGIN
        -- 민감정보를 마스킹한 컨텍스트 생성
        l_context := 'TRACE_WITH_MASKING_' || DBMS_RANDOM.STRING('X', 10);
        
        -- Trace 활성화 (바인드 값은 기록하지 않음)
        DBMS_MONITOR.SESSION_TRACE_ENABLE(
            p_sid, p_serial,
            waits => TRUE,
            binds => FALSE,  -- 보안상 바인드 값 제외
            plan_stat => 'ALL_EXECUTIONS'
        );
        
        -- 추가적인 보안 로깅
        INSERT INTO trace_audit_log (
            trace_id, sid, serial#, start_time, 
            masking_enabled, audit_user
        ) VALUES (
            l_context, p_sid, p_serial, SYSTIMESTAMP,
            'Y', SYS_CONTEXT('USERENV', 'SESSION_USER')
        );
        
        COMMIT;
    END enable_trace_with_masking;
    
    FUNCTION mask_sensitive_data(
        p_sql_text VARCHAR2,
        p_patterns DBMS_SQL.VARCHAR2_TABLE
    ) RETURN VARCHAR2 IS
        l_result VARCHAR2(32767) := p_sql_text;
    BEGIN
        FOR i IN 1..p_patterns.COUNT LOOP
            l_result := REGEXP_REPLACE(
                l_result,
                p_patterns(i),
                '***MASKED***',
                1, 0, 'i'
            );
        END LOOP;
        RETURN l_result;
    END mask_sensitive_data;
    
END secure_trace;
/

-- Trace 감사 로그 테이블
CREATE TABLE trace_audit_log (
    trace_id          VARCHAR2(50) PRIMARY KEY,
    sid               NUMBER,
    serial#           NUMBER,
    start_time        TIMESTAMP,
    end_time          TIMESTAMP,
    trace_file        VARCHAR2(500),
    masking_enabled   CHAR(1),
    audit_user        VARCHAR2(30),
    purpose           VARCHAR2(200),
    CONSTRAINT chk_masking CHECK (masking_enabled IN ('Y', 'N'))
);

-- 정기적인 Trace 파일 정리
CREATE OR REPLACE PROCEDURE purge_old_trace_files (
    p_days_to_keep NUMBER DEFAULT 30
) IS
    l_trace_dir VARCHAR2(4000);
BEGIN
    SELECT value INTO l_trace_dir
    FROM v$diag_info
    WHERE name = 'Diag Trace';
    
    -- 오래된 Trace 파일 삭제 (OS 명령어)
    DBMS_SCHEDULER.CREATE_JOB(
        job_name => 'PURGE_OLD_TRACES',
        job_type => 'EXECUTABLE',
        job_action => '/bin/find',
        number_of_arguments => 4,
        enabled => FALSE
    );
    
    DBMS_SCHEDULER.SET_JOB_ARGUMENT_VALUE('PURGE_OLD_TRACES', 1, l_trace_dir);
    DBMS_SCHEDULER.SET_JOB_ARGUMENT_VALUE('PURGE_OLD_TRACES', 2, '-name');
    DBMS_SCHEDULER.SET_JOB_ARGUMENT_VALUE('PURGE_OLD_TRACES', 3, '"*.trc"');
    DBMS_SCHEDULER.SET_JOB_ARGUMENT_VALUE('PURGE_OLD_TRACES', 4, 
        '-mtime +' || p_days_to_keep || ' -delete');
    
    DBMS_SCHEDULER.ENABLE('PURGE_OLD_TRACES');
    
    -- 감사 로그 정리
    DELETE FROM trace_audit_log
    WHERE start_time < SYSTIMESTAMP - p_days_to_keep;
    
    COMMIT;
END purge_old_trace_files;
/
```

---

## 결론: SQL Trace의 전략적 활용

SQL Trace는 Oracle 성능 문제 해결의 **궁극적인 증거 수집 도구**입니다. 그러나 그 힘은 책임감 있는 사용에서 비롯됩니다. 효과적인 Trace 활용을 위한 핵심 원칙은 다음과 같습니다:

1. **최소 권한 원칙**: 항상 가능한 가장 좁은 범위(세션 → 모듈 → 서비스 순)로 Trace를 활성화하세요.
2. **시간 제한 설정**: 무한정 Trace를 실행하지 말고, 명시적인 타임아웃을 설정하세요.
3. **보안 고려**: 운영 환경에서는 바인드 값 기록을 피하고, 필요시 마스킹을 적용하세요.
4. **체계적 분석**: Trace → TKPROF → 실행 계획 분석의 체계적인 워크플로우를 따르세요.
5. **자동화 활용**: 반복적인 Trace 수집 작업은 패키지와 스케줄러로 자동화하세요.

가장 중요한 것은 **Trace가 목적이 아니라 수단**이라는 점입니다. Trace를 통해 발견한 통찰을 실제 성능 개선으로 연결할 때, 비로소 그 진정한 가치가 발현됩니다. 데이터베이스 성능 튜닝은 예술과 과학의 조화이며, SQL Trace는 그 과정에서 가장 정밀한 측정 도구 역할을 합니다.