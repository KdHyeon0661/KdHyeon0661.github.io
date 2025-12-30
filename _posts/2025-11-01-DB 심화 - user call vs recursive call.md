---
layout: post
title: DB 심화 - user call vs recursive call
date: 2025-11-01 17:25:23 +0900
category: DB 심화
---
# User Call vs Recursive Call: 오라클 호출 개념 완벽 이해

> **핵심 요약**
> - **User Call**: 클라이언트 애플리케이션이 오라클 서버에 보내는 OCI/드라이버 레벨 호출로, 파싱, 실행, 페치 등의 작업을 포함합니다.
> - **Recursive Call**: 오라클이 사용자의 SQL을 처리하는 과정에서 내부적으로 실행하는 추가 SQL 호출로, 데이터 딕셔너리 조회, 공간 관리, 시퀀스 처리, 제약 검증 등의 작업을 담당합니다.
> 
> 이 두 개념을 명확히 이해하면, 단순한 SELECT 문이 왜 느린지, AWR 리포트에서 Recursive CPU가 높게 나타나는 이유는 무엇인지, 파싱 횟수는 적은데 디스크 I/O가 많은 원인은 무엇인지와 같은 성능 문제에 대한 체계적인 답을 찾을 수 있습니다.

---

## 개념 정의: User Call과 Recursive Call

### User Call (사용자 호출)

**정의**
User Call은 클라이언트 애플리케이션이 오라클 데이터베이스 서버와 통신하기 위해 OCI(Oracle Call Interface) 또는 데이터베이스 드라이버를 통해 보내는 호출입니다. 이는 사용자가 직접 요청한 작업을 나타냅니다.

**주요 통계 지표**
- `V$SYSSTAT` / `V$SESSTAT` 뷰의 `user calls`
- `bytes received via SQL*Net from client`
- `bytes sent via SQL*Net to client`

**발생 시점 예시**
- SQL 문장 파싱 요청(`OCIPrepare`)
- SQL 실행 요청(`OCIExecute`)
- 결과 페치 요청(`OCIDefineByPos` / Fetch)
- 데이터베이스 연결/해제(Logon/Logoff)
- 메타데이터 조회 요청(Describe)

**주요 활용 목적**
애플리케이션의 "채팅성(Chattiness)"을 진단하는 데 사용됩니다. 즉, 너무 빈번한 소량 페치, 불필요한 파싱/실행 반복, 과도한 커밋 호출 등의 비효율적인 패턴을 식별합니다.

### Recursive Call (재귀 호출)

**정의**
Recursive Call은 오라클 데이터베이스 엔진이 사용자의 SQL 문장을 처리하는 과정에서 내부적으로 실행하는 추가 SQL 호출입니다. 사용자는 이 호출들을 직접 의도하지 않았지만, 데이터베이스의 정상적인 동작을 위해 필요합니다.

**주요 발생 원인**

1. **데이터 딕셔너리/메타데이터 조회**
   - 동의어(Synonym) 해석
   - 객체 권한 검증
   - 뷰(View) 확장
   - 파티션 정보 확인

2. **세그먼트 및 공간 관리**
   - 익스텐트(Extent) 할당
   - 세그먼트 헤더 갱신
   - ASSM(Automatic Segment Space Management) 내부 작업

3. **시퀀스 처리**
   - `SEQ.NEXTVAL` 호출 시 내부 `SEQ$` 테이블 접근
   - 시퀀스 캐시 보충

4. **제약 조건 및 트리거 실행**
   - CHECK 제약 조건 검증
   - FOREIGN KEY 제약 조건 검증
   - 트리거 내부 SQL 실행
   - 물질화된 뷰 로그 관리

5. **통계 및 실행 계획 관련 작업**
   - 동적 샘플링(Dynamic Sampling)
   - Outline/SQL 프로파일 관리
   - SQL 계획 관리자(SPM) 작업
   - 규칙 기반 옵티마이저 평가

6. **DDL 및 DDL 유사 작업**
   - DDL 문장 실행 시 내부 데이터 딕셔너리 갱신

**주요 통계 지표**
- `recursive calls`
- `recursive cpu usage`
- `recursive elapsed time` (버전에 따라 명칭 상이)

**간단한 비유**
- **User Call**: 레스토랑에서 손님이 웨이터에게 주문하는 횟수
- **Recursive Call**: 웨이터가 주방에 전달한 주문을 처리하기 위해 주방 직원들이 내부적으로 수행하는 작업 횟수

---

## 실습 환경 구성

### 테스트 스키마 설정

```sql
-- 실험용 테이블 생성
DROP TABLE demo_call PURGE;
CREATE TABLE demo_call (
  id        NUMBER PRIMARY KEY,
  c_region  VARCHAR2(10),
  c_status  VARCHAR2(10),
  amt       NUMBER(12,2),
  dte       DATE
);

-- 샘플 데이터 삽입 (500,000행)
BEGIN
  INSERT /*+ APPEND */ INTO demo_call
  SELECT level,
         CASE MOD(level,4) 
           WHEN 0 THEN 'APAC' 
           WHEN 1 THEN 'EMEA' 
           WHEN 2 THEN 'AMER' 
           ELSE 'OTHER' 
         END,
         CASE MOD(level,5) 
           WHEN 0 THEN 'X' 
           ELSE 'OK' 
         END,
         ROUND(DBMS_RANDOM.VALUE(10,1000),2),
         TRUNC(SYSDATE) - MOD(level, 365)
  FROM dual CONNECT BY level <= 500000;
  COMMIT;
END;
/

-- 인덱스 생성 및 통계 수집
CREATE INDEX ix_demo_call_r_d ON demo_call(c_region, dte);
BEGIN 
  DBMS_STATS.GATHER_TABLE_STATS(USER,'DEMO_CALL', cascade=>TRUE); 
END;
/

-- 시퀀스 및 트리거 생성 (재귀 호출 데모용)
DROP SEQUENCE demo_seq;
CREATE SEQUENCE demo_seq CACHE 100;

-- BEFORE INSERT 트리거 생성
CREATE OR REPLACE TRIGGER demo_call_bi
BEFORE INSERT ON demo_call
FOR EACH ROW
BEGIN
  -- ID가 NULL인 경우 시퀀스에서 값 할당
  IF :NEW.id IS NULL THEN
    :NEW.id := demo_seq.NEXTVAL; -- 내부적으로 SEQ$ 테이블 접근 (재귀 호출 발생)
  END IF;
END;
/
```

### 실시간 모니터링을 위한 유틸리티 함수

```sql
-- 세션 통계 스냅샷 추출 함수
CREATE OR REPLACE FUNCTION get_session_stats 
RETURN SYS_REFCURSOR
IS
  v_cursor SYS_REFCURSOR;
BEGIN
  OPEN v_cursor FOR
    SELECT sn.name AS stat_name, 
           ss.value AS stat_value
    FROM   v$sesstat ss
    JOIN   v$statname sn ON sn.statistic# = ss.statistic#
    WHERE  ss.sid = SYS_CONTEXT('USERENV','SID')
    AND    sn.name IN ('user calls',
                      'recursive calls',
                      'recursive cpu usage',
                      'parse count (total)',
                      'parse count (hard)',
                      'bytes sent via SQL*Net to client',
                      'bytes received via SQL*Net from client')
    ORDER BY sn.name;
    
  RETURN v_cursor;
END;
/

-- 사용 예시: 실행 전후 통계 비교
DECLARE
  v_before_stats SYS_REFCURSOR;
  v_after_stats  SYS_REFCURSOR;
  v_stat_name    VARCHAR2(100);
  v_before_val   NUMBER;
  v_after_val    NUMBER;
BEGIN
  -- 실행 전 통계 수집
  v_before_stats := get_session_stats();
  
  -- 테스트 쿼리 실행
  FOR i IN 1..100 LOOP
    INSERT INTO demo_call(c_region, c_status, amt, dte)
    VALUES ('TEST', 'OK', 100, SYSDATE);
  END LOOP;
  COMMIT;
  
  -- 실행 후 통계 수집
  v_after_stats := get_session_stats();
  
  -- 결과 비교 출력
  DBMS_OUTPUT.PUT_LINE('통계 비교 (After - Before):');
  DBMS_OUTPUT.PUT_LINE('----------------------------------------');
  
  -- 통계 비교 로직 (실제 구현에서는 더 정교한 비교 로직 필요)
END;
/
```

**모니터링 팁:** 특정 작업을 실행하기 전후로 통계 값을 각각 기록하여 델타(변화량)를 계산하면, 해당 작업이 어떤 호출을 얼마나 발생시켰는지 정확히 측정할 수 있습니다.

---

## TKPROF와 SQL Trace에서의 관찰

### 트레이스 설정 방법

```sql
-- 상세 통계 수집 활성화
ALTER SESSION SET statistics_level = ALL;

-- 레벨 8 트레이스 시작 (대기 이벤트 포함, 바인드 변수 값 제외)
ALTER SESSION SET events '10046 trace name context forever, level 8';

-- 테스트 쿼리 실행
SELECT /*+ MONITOR */ COUNT(*)
FROM   demo_call
WHERE  c_region = 'APAC'
AND    dte >= TRUNC(SYSDATE) - 30;

-- 트레이스 중지
ALTER SESSION SET events '10046 trace name context off';
```

### TKPROF 결과 분석

TKPROF로 변환한 트레이스 파일의 상단에는 다음과 같은 요약 정보가 포함됩니다:

```
Misses in library cache during parse: 0
Optimizer mode: ALL_ROWS
Parsing user id: 107
Number of plan statistics captured: 1

Recursive calls: 42
user  calls: 23
```

**관찰 포인트:**

1. **User Calls 값 분석**
   - 클라이언트에서 서버로 보낸 호출 총수
   - 일반적으로 Parse(1) + Execute(1) + Fetch(n) 횟수의 합
   - 값이 비정상적으로 높다면 애플리케이션의 비효율적인 호출 패턴을 의심

2. **Recursive Calls 값 분석**
   - 오라클 내부에서 발생한 추가 호출 수
   - 비정상적으로 높은 값은 다양한 문제를 시사:
     - 빈번한 DDL 실행
     - 작은 시퀀스 캐시로 인한 잦은 보충
     - 트리거나 제약 조건의 비효율적인 내부 SQL
     - 공간 관리 오버헤드

---

## 사례 연구: 다양한 패턴별 호출 분석

### 사례 1: 단순 SELECT 쿼리 (User Call 중심)

```sql
-- 바인드 변수 사용
VAR r VARCHAR2(10); 
EXEC :r := 'APAC';

-- 효율적인 페치 크기로 실행
SELECT /*+ CASE1_SIMPLE_SELECT */ COUNT(*)
FROM   demo_call
WHERE  c_region = :r
AND    dte >= TRUNC(SYSDATE) - 30
AND    dte < TRUNC(SYSDATE);
```

**예상 호출 패턴:**
- `user calls`: Parse(1) + Execute(1) + Fetch(1) = 약 3회
- `recursive calls`: 딕셔너리 캐시가 워밍업된 상태라면 0에 가까움

**성능 문제 시나리오:**
만약 애플리케이션이 페치 배열 크기를 매우 작게 설정했다면(예: 10), 1000행을 조회할 때 100번의 Fetch 호출이 발생하여 `user calls`가 불필요하게 증가합니다.

**해결 방안:**
- JDBC: `setFetchSize(1000)` 설정
- ODP.NET: `FetchSize` 속성 조정
- Python: `cursor.arraysize = 1000` 설정

### 사례 2: 시퀀스와 트리거를 사용하는 INSERT (Recursive Call 증가)

```sql
BEGIN
  -- 5000건의 INSERT 실행
  FOR i IN 1..5000 LOOP
    INSERT INTO demo_call(c_region, c_status, amt, dte)
    VALUES ('APAC', 'OK', 100, SYSDATE);
    -- 트리거에서 시퀀스 호출 발생 → 내부 SEQ$ 테이블 접근
  END LOOP;
  COMMIT;
END;
/
```

**예상 호출 패턴:**
- `user calls`: 루프 반복 횟수에 따라 수천 회 발생
- `recursive calls`: 시퀀스 캐시 보충, 트리거 실행, 제약 검증 등으로 상당히 증가

**문제 분석:**
시퀀스의 `CACHE` 값이 20으로 작게 설정된 경우, 5000번의 INSERT 중 약 250번의 시퀀스 캐시 보충이 발생하여 동일한 수의 재귀 호출이 추가됩니다.

**최적화 방안:**

1. **시퀀스 캐시 크기 증가**
   ```sql
   ALTER SEQUENCE demo_seq CACHE 1000;
   ```

2. **벌크 INSERT 사용**
   ```sql
   -- PL/SQL FORALL 구문 활용
   DECLARE
     TYPE t_ids IS TABLE OF NUMBER;
     v_ids t_ids := t_ids();
   BEGIN
     -- 대량 데이터 준비
     FOR i IN 1..5000 LOOP
       v_ids.EXTEND;
       v_ids(v_ids.COUNT) := i;
     END LOOP;
     
     -- 벌크 INSERT 실행
     FORALL i IN 1..v_ids.COUNT
       INSERT INTO demo_call(id, c_region, c_status, amt, dte)
       VALUES (v_ids(i), 'APAC', 'OK', 100, SYSDATE);
       
     COMMIT;
   END;
   ```

### 사례 3: DDL 실행 시의 Recursive Call 급증

```sql
-- 인덱스 생성 (내부적으로 다수의 딕셔너리 테이블 갱신 발생)
CREATE INDEX ix_demo_call_status ON demo_call(c_status);
```

**호출 패턴 특성:**
DDL 문장은 내부적으로 여러 시스템 테이블(`obj$`, `tab$`, `ind$`, `col$` 등)을 갱신해야 하므로 상당한 수의 재귀 호출을 발생시킵니다.

**문제가 되는 패턴:**
- 런타임 중 빈번한 임시 테이블 생성/삭제
- 동적 권한 부여/회수
- 세션별 임시 객체 관리

**대체 솔루션:**
1. **전역 임시 테이블(GTT) 활용**
   ```sql
   CREATE GLOBAL TEMPORARY TABLE temp_data (
     id NUMBER,
     name VARCHAR2(100)
   ) ON COMMIT DELETE ROWS;
   ```

2. **컬렉션 변수 사용** (PL/SQL 내 처리 시)
   ```plsql
   DECLARE
     TYPE t_data IS TABLE OF demo_call%ROWTYPE;
     v_data t_data;
   BEGIN
     -- 컬렉션 변수에 데이터 저장 및 처리
     SELECT * BULK COLLECT INTO v_data
     FROM demo_call
     WHERE c_region = 'APAC';
     
     -- 메모리 내 처리
     FOR i IN 1..v_data.COUNT LOOP
       -- 처리 로직
     END LOOP;
   END;
   ```

### 사례 4: 대량 데이터 작업 시의 공간 관리 오버헤드

```sql
-- 대량 INSERT (익스텐트 확장 발생)
INSERT /*+ APPEND */ INTO demo_call
SELECT * FROM demo_call WHERE ROWNUM <= 100000;
COMMIT;
```

**발생 현상:**
대량 데이터 삽입 시 세그먼트의 익스텐트 확장이 발생하며, 이 과정에서 세그먼트 헤더 갱신과 딕셔너리 테이블 업데이트를 위한 재귀 호출이 증가합니다.

**최적화 전략:**

1. **사전 공간 할당**
   ```sql
   ALTER TABLE demo_call ALLOCATE EXTENT (
     SIZE 100M
   );
   ```

2. **적절한 스토리지 파라미터 설정**
   ```sql
   CREATE TABLE large_table (
     id NUMBER,
     data VARCHAR2(4000)
   ) TABLESPACE users
   STORAGE (
     INITIAL 100M
     NEXT 50M
     MINEXTENTS 1
     MAXEXTENTS UNLIMITED
   );
   ```

### 사례 5: 복잡한 객체 계층 구조의 메타데이터 조회

```sql
-- 뷰 생성
CREATE OR REPLACE VIEW v_demo_filtered AS
SELECT * 
FROM demo_call 
WHERE c_status = 'OK';

-- Public 동의어 생성
CREATE PUBLIC SYNONYM syn_demo FOR v_demo_filtered;

-- 권한 부여
GRANT SELECT ON syn_demo TO public;

-- 복잡한 경로를 통한 조회
SELECT COUNT(*) 
FROM syn_demo 
WHERE c_region = 'APAC';
```

**초기 실행 시 발생 현상:**
- 동의어 해석을 위한 딕셔너리 조회
- 뷰 텍스트 확장
- 권한 검증
- 객체 종속성 확인

**최적화 접근법:**

1. **딕셔너리 캐시 최적화**
   ```sql
   -- 공유 풀 크기 적정화
   ALTER SYSTEM SET SHARED_POOL_SIZE = 2G SCOPE=SPFILE;
   
   -- 라이브러리 캐시 히트율 모니터링
   SELECT namespace,
          gethitratio,
          pinhitratio,
          reloads,
          invalidations
   FROM   v$librarycache
   WHERE  namespace IN ('SQL AREA', 'TABLE/PROCEDURE', 'BODY', 'TRIGGER');
   ```

2. **객체 설계 단순화**
   - 불필요한 뷰 중첩 회피
   - 과도한 동의어 사용 자제
   - 권한 계층 구조 단순화

---

## AWR과 ASH 리포트에서의 분석 방법

### AWR 리포트 분석 포인트

1. **Load Profile 섹션**
   - `User calls per sec`: 초당 사용자 호출 수
   - `Recursive calls per sec`: 초당 재귀 호출 수

2. **Instance Efficiency Percentages 섹션**
   - `Dictionary Cache Hit %`: 딕셔너리 캐시 적중률
   - `Library Cache Hit %`: 라이브러리 캐시 적중률

3. **SQL Ordered by Recursive CPU Time 섹션**
   - 재귀 SQL 중 CPU 소비가 큰 순서로 정렬
   - 내부 시스템 SQL(`obj$`, `seg$`, `seq$` 접근) 식별

### ASH를 활용한 실시간 분석

```sql
-- 최근 15분간 재귀 호출 관련 상위 SQL 조회
SELECT 
    sql_id,
    COUNT(*) AS sample_count,
    ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER(), 2) AS percentage
FROM   v$active_session_history
WHERE  sample_time > SYSTIMESTAMP - INTERVAL '15' MINUTE
  AND  sql_id IS NOT NULL
GROUP  BY sql_id
ORDER  BY sample_count DESC
FETCH FIRST 10 ROWS ONLY;

-- 특정 재귀 SQL의 상세 정보 확인
SELECT 
    sql_text,
    executions,
    elapsed_time,
    cpu_time,
    buffer_gets,
    disk_reads
FROM   v$sql
WHERE  sql_id = '확인된_SQL_ID'
  AND  sql_text LIKE '%obj$%' OR sql_text LIKE '%seq$%'; -- 시스템 테이블 패턴
```

---

## 원인별 진단 및 해결 체계

### User Call이 과다한 경우

**증상**
- `user calls` 통계가 비정상적으로 높음
- 네트워크 트래픽(`bytes sent/received via SQL*Net`) 증가
- 애플리케이션 응답 시간 지연

**주요 원인**
1. **작은 페치 배열 크기**: 많은 행을 조회할 때 너무 작은 배치로 나누어 페치
2. **비효율적인 파싱 패턴**: 동일 SQL의 반복적인 파싱
3. **과도한 커밋**: 각 DML 후 불필요한 커밋 호출
4. **불필요한 왕복**: 논리적으로 한 번에 처리 가능한 작업을 여러 호출로 분할

**해결 방안**
1. **페치 크기 최적화**
   ```java
   // JDBC 예시
   statement.setFetchSize(1000);
   ```
   
2. **커서 캐싱 활성화**
   ```sql
   -- 세션 커서 캐시 크기 증가
   ALTER SESSION SET SESSION_CACHED_CURSORS = 100;
   ```

3. **배치 처리 구현**
   ```java
   // JDBC 배치 처리 예시
   connection.setAutoCommit(false);
   PreparedStatement ps = connection.prepareStatement(
       "INSERT INTO demo_call VALUES (?, ?, ?, ?, ?)");
   
   for (int i = 0; i < 1000; i++) {
       ps.setInt(1, i);
       // ... 다른 파라미터 설정
       ps.addBatch();
       
       if (i % 100 == 0) {
           ps.executeBatch();
       }
   }
   ps.executeBatch();
   connection.commit();
   ```

### Recursive Call이 과다한 경우

**증상**
- `recursive calls` 통계가 비정상적으로 높음
- `recursive cpu usage` 증가
- 딕셔너리 캐시 미스율 상승

**주요 원인별 해결 전략**

1. **시퀀스 관련 문제**
   ```sql
   -- 캐시 크기 증가 (기본값은 20)
   ALTER SEQUENCE demo_seq CACHE 1000;
   
   -- NOORDER 옵션 (RAC 환경에서 성능 최적화)
   CREATE SEQUENCE fast_seq 
     START WITH 1 
     INCREMENT BY 1 
     CACHE 1000 
     NOORDER;
   ```

2. **DDL 오버헤드**
   - 런타임 DDL 실행 회피
   - 임시 테이블 대신 GTT 또는 컬렉션 변수 사용
   - 객체 생성/삭제는 배치 시간에 집중 실행

3. **공간 관리 최적화**
   ```sql
   -- 테이블스페이스 자동 확장 설정
   ALTER DATABASE DATAFILE '/u01/app/oracle/oradata/DB/users01.dbf'
   AUTOEXTEND ON NEXT 100M MAXSIZE 10G;
   
   -- 세그먼트 사전 확장
   EXEC DBMS_SPACE.ADMIN_TABLESPACE_GROWTH('USERS', 'AUTO');
   ```

4. **딕셔너리 캐시 효율화**
   ```sql
   -- 공유 풀 크기 모니터링 및 조정
   SELECT *
   FROM   v$sgastat
   WHERE  pool = 'shared pool'
   AND    name LIKE '%cache%';
   
   -- 핫 오브젝트 식별
   SELECT owner, name, type, executions, loads, invalidations
   FROM   v$db_object_cache
   WHERE  executions > 1000
   ORDER BY executions DESC;
   ```

---

## 성능 지표 해석을 위한 휴리스틱

### 유용한 성능 비율 지표

1. **행 당 사용자 호출 비율**
   ```
   Rows per User Call = 총 페치 행 수 ÷ user calls
   ```
   - **낮을수록**: 왕복이 많음 → 페치 배열 크기 증가 필요
   - **적정 값**: 애플리케이션 특성에 따라 다르며, 일반적으로 100 이상을 목표

2. **재귀/사용자 호출 비율**
   ```
   Recursive/User Ratio = recursive calls ÷ user calls
   ```
   - **비정상적 상승**: 내부 오버헤드 증가 원인 조사 필요
   - **일반적 범위**: 작업 유형에 크게 의존 (DDL 시 높음, 단순 SELECT 시 낮음)

3. **실행 당 파싱 비율**
   ```
   Parse per Execution = parse count (total) ÷ execute count
   ```
   - **1에 가까울수록**: 파싱 재사율 낮음 → 커서 캐싱 최적화 필요
   - **목표 값**: 0.1 이하 (10번 실행에 1번 파싱)

### 모니터링 쿼리 예시

```sql
-- 실시간 성능 지표 모니터링
SELECT 
    'User Calls' AS metric,
    (SELECT value FROM v$mystat s, v$statname n 
     WHERE s.statistic# = n.statistic# AND n.name = 'user calls') AS value
FROM dual
UNION ALL
SELECT 
    'Recursive Calls',
    (SELECT value FROM v$mystat s, v$statname n 
     WHERE s.statistic# = n.statistic# AND n.name = 'recursive calls')
FROM dual
UNION ALL
SELECT 
    'Rows per User Call',
    ROUND(
        (SELECT value FROM v$mystat s, v$statname n 
         WHERE s.statistic# = n.statistic# AND n.name = 'session logical reads') /
        NULLIF(
            (SELECT value FROM v$mystat s, v$statname n 
             WHERE s.statistic# = n.statistic# AND n.name = 'user calls'), 0
        ), 2
    )
FROM dual;
```

---

## 종합 최적화 전략

### 애플리케이션 레벨 최적화

1. **연결 관리**
   - 연결 풀링(Connection Pooling) 구현
   - 적절한 풀 크기 유지 (과다/부족 모두 문제)

2. **문장 처리 최적화**
   ```java
   // PreparedStatement 재사용 예시
   private static final String INSERT_SQL = 
       "INSERT INTO demo_call VALUES (?, ?, ?, ?, ?)";
   
   public void bulkInsert(List<Data> dataList) throws SQLException {
       try (PreparedStatement ps = connection.prepareStatement(INSERT_SQL)) {
           connection.setAutoCommit(false);
           
           for (Data data : dataList) {
               ps.setInt(1, data.getId());
               // ... 다른 파라미터 설정
               ps.addBatch();
           }
           
           ps.executeBatch();
           connection.commit();
       }
   }
   ```

3. **트랜잭션 관리**
   - 논리적 작업 단위별 커밋
   - 불필요한 자동 커밋 비활성화
   - 적절한 격리 수준 설정

### 데이터베이스 레벨 최적화

1. **메모리 구성 최적화**
   ```sql
   -- 공유 풀 크기 모니터링
   SELECT component, current_size, min_size, max_size
   FROM   v$memory_dynamic_components
   WHERE  component LIKE '%Shared Pool%';
   
   -- 라이브러리 캐시 효율 분석
   SELECT namespace,
          gets,
          gethits,
          ROUND(gethitratio * 100, 2) AS hit_ratio_pct,
          pins,
          pinhits,
          ROUND(pinhitratio * 100, 2) AS pin_hit_ratio_pct
   FROM   v$librarycache;
   ```

2. **시퀀스 관리**
   ```sql
   -- 시퀀스 사용 현황 모니터링
   SELECT sequence_owner,
          sequence_name,
          cache_size,
          last_number,
          increment_by
   FROM   dba_sequences
   WHERE  sequence_owner = USER
   ORDER BY cache_size;
   ```

3. **객체 설계 최적화**
   - 불필요한 뷰 중첩 회피
   - 적절한 인덱스 전략 수립
   - 파티셔닝 고려 (대량 데이터 처리 시)

---

## 문제 해결 워크플로우

### 체계적인 진단 절차

1. **증상 식별**
   - 응답 시간 지연
   - CPU 사용률 상승
   - 특정 시간대 성능 저하

2. **데이터 수집**
   ```sql
   -- AWR 스냅샷 생성
   EXEC DBMS_WORKLOAD_REPOSITORY.CREATE_SNAPSHOT();
   
   -- 세션 통계 수집
   ALTER SESSION SET statistics_level = ALL;
   ```

3. **분석 수행**
   - AWR 리포트 분석 (User/Recursive Calls 추이)
   - ASH 데이터 검토 (실시간 대기 이벤트)
   - TKPROF 분석 (상세 호출 패턴)

4. **근본 원인 식별**
   - 패턴 기반 원인 분류
   - 비정상 지표 상관관계 분석

5. **해결책 구현 및 검증**
   - 타겟팅된 최적화 적용
   - 성능 개선 효과 측정
   - 회귀 테스트 수행

---

## 결론

User Call과 Recursive Call의 개념을 명확히 이해하는 것은 오라클 데이터베이스 성능 튜닝의 핵심입니다. 이 두 지표는 데이터베이스 시스템의 건강 상태를 진단하는 중요한 체온계 역할을 합니다.

User Call이 높은 경우, 이는 일반적으로 애플리케이션과 데이터베이스 간의 통신 비효율성을 나타냅니다. 해결책은 주로 클라이언트 측에 있으며, 페치 배열 크기 최적화, 커서 캐싱 활성화, 배치 처리 구현 등이 효과적입니다.

반면 Recursive Call이 높은 경우, 이는 데이터베이스 내부의 오버헤드가 증가했음을 의미합니다. 시퀀스 캐시 크기 조정, DDL 실행 패턴 최적화, 공간 관리 전략 개선, 딕셔너리 캐시 효율화 등 데이터베이스 레벨의 조정이 필요합니다.

가장 효과적인 성능 최적화는 이러한 두 가지 호출 유형을 균형 있게 관리하는 데 있습니다. 정기적인 모니터링을 통해 baseline을 설정하고, 비정상적인 변화를 조기에 감지하며, 데이터 기반의 과학적 접근법으로 문제를 해결하는 것이 지속 가능한 고성능 시스템 운영의 핵심입니다.

성능 문제는 단일 원인보다는 여러 요소가 복합적으로 작용하는 경우가 많습니다. User Call과 Recursive Call 분석을 출발점으로 삼아, 실행 계획, 인덱스 효율성, 메모리 구성, I/O 패턴 등 종합적인 관점에서 시스템을 진단하고 최적화하시기 바랍니다.