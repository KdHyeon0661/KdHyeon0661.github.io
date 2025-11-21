---
layout: post
title: DB 심화 - Oracle Database 기본 아키텍처
date: 2025-10-15 17:25:23 +0900
category: DB 심화
---
# Oracle Database 기본 아키텍처 (19c 기준, 실무 관점 종합 정리)

> 대상: 오라클을 처음 체계적으로 정리하려는 분부터 실무 튜닝/운영 지식을 연결하고 싶은 분까지
> 버전 가정: Oracle Database 19c (12c/21c/23ai와 차이점은 각 섹션에 주석)

---

## ↔ 데이터베이스(파일)”

오라클은 **인스턴스(Instance)** 와 **데이터베이스(Database)** 로 분리된 **논리 모델**을 갖습니다.

- **인스턴스**: SGA(+옵션 In-Memory Column Store), PGA, 그리고 수십 개의 **백그라운드 프로세스**
  - 인스턴스는 “메모리와 프로세스의 집합”으로, 가동 중일 때만 존재합니다.
- **데이터베이스**: **데이터 파일**, **컨트롤 파일**, **리두 로그(온라인/아카이브)**, **임시 파일**, **파라미터/패스워드 파일**, **ADR(진단 디렉터리)** 등 **물리 파일 집합**입니다.
  - 인스턴스가 내려가도 파일은 디스크에 남습니다.

**핵심 상호 작용**:

1. 클라이언트가 SQL을 보내면 서버 프로세스가 파싱/최적화/실행을 수행.
2. **Buffer Cache** 에 블록을 읽고, DML 시 변경 → **Dirty Buffer** 로 표시.
3. 변경 사항은 **Redo Log Buffer** 에 리두 레코드로 기록되고, **커밋 시 LGWR가 플러시**.
4. **DBWn** 이 적절한 시점에 Dirty Buffer를 **데이터 파일**로 기록.
5. **CKPT** 가 체크포인트 정보를 컨트롤 파일/헤더에 갱신.
6. **SMON/PMON** 등 백그라운드가 회복/정리/모니터링을 수행.

---

## 프로세스 아키텍처 — 백그라운드/서버/리스너

### 프로세스 유형

- **서버 프로세스(Server Process)**
  - **전용(Dedicated)**: 클라이언트 세션당 서버 프로세스 1:1. 단순하고 예측 가능.
  - **공유(Shared Server, MTS)**: 다수 세션이 **디스패처(Dispatcher)** 를 통해 **공유 서버** 프로세스 풀을 공유. 고동시성+짧은 트랜잭션 환경에 유리.
- **백그라운드(Background) 프로세스** — 필수/옵션
  - **PMON**(Process Monitor): 비정상 종료 세션 정리, 리소스 회수.
  - **SMON**(System Monitor): **인스턴스/크래시 리커버리**, 임시 세그먼트 청소.
  - **DBWn**(Database Writer): Dirty Buffer → 데이터 파일.
  - **LGWR**(Log Writer): Redo Log Buffer → 온라인 리두 로그 파일에 순차 기록(**커밋 지연 최소화**).
  - **CKPT**(Checkpoint): 체크포인트 발생 시 데이터파일 헤더/컨트롤 파일의 SCN 갱신.
  - **ARCn**(Archiver): ARCHIVELOG 모드에서 **온라인 리두 로그 → 아카이브 로그** 복제.
  - **MMON/MMNL**: AWR 스냅샷, 통계/얼럿 관리.
  - **LREG**: 서비스/리스너와 등록.
  - **CJQ0**: 스케줄러 잡 관리(및 **Jnnn** 잡 슬레이브).
  - **RVWR**: (옵션) Flashback Database용 리두 버퍼 기록.
  - **RECO**: 분산 트랜잭션 복구.
  - **QMNC/Qnnn**: AQ(Advanced Queuing).
  - **Dnnn**: Shared Server 디스패처.
  - **FBDA**: Flashback Data Archive(옵션) 백그라운드.
  - **RVWR/FBDA/MMON 등은 기능 활성화 시만 등장**.

> RAC/ASM 환경에서는 **LMON/LMDn/LMSn/LCKn**(글로벌 캐시/락) 및 **RBAL/ARBx**(ASM 리밸런스) 등 추가 프로세스가 존재합니다.

### 리스너와 접속 흐름(Oracle Net)

- **리스너(listener.ora)** 는 특정 포트(기본 1521)에서 대기하며, 접속 요청을 받아 **서버 프로세스**를 붙여줍니다.
- 전용 서버 모드에선 클라이언트가 곧바로 서버 프로세스와 세션을 형성합니다.
- 공유 서버 모드에선 **디스패처**가 요청을 큐에 적재하고, **공유 서버 프로세스**가 처리합니다.

```bash
# 리스너 상태 확인

lsnrctl status

# 데이터베이스 서비스가 리스너에 동적 등록되었는지 확인

lsnrctl services
```

---

## In-Memory

### 구성

- **Shared Pool**
  - **Library Cache**: 파싱된 SQL, 실행계획(커서) 캐시.
  - **Data Dictionary Cache(Row Cache)**: 오브젝트/사용자/권한 메타데이터 캐시.
  - **Server Result Cache**(옵션).
- **Database Buffer Cache**
  - 데이터 파일 블록의 버퍼. **Clean/Dirty** 상태, **LRU(또는 버킷/핫/콜드 리스트)** 정책.
  - 각 버퍼는 **SCN, ITL(Interested Transaction List), 버전** 관련 메타 포함.
- **Redo Log Buffer**
  - 리두 엔트리(변경 이력) 임시 버퍼. **LGWR**가 **커밋 시 즉시** 혹은 주기적으로 플러시.
- **Large Pool**(옵션)
  - 대용량 메모리 연산(병렬실행 메시지 버퍼, RMAN 채널, Shared Server UGA 등).
- **Java Pool**(옵션), **Streams Pool**(옵션).
- **In-Memory Column Store(IM column store)** — 12c R2+ 옵션
  - 컬럼 형식으로 데이터를 메모리에 보관, 벡터화/인메모리 스캔 가속.

```sql
-- SGA 요약
SELECT * FROM v$sga;
SELECT * FROM v$sgainfo ORDER BY name;

-- Shared Pool/Buffer Cache 사용량(요약)
SELECT pool, name, bytes/1024/1024 AS mb
FROM v$sgastat
ORDER BY pool, name;

-- Redo Log Buffer 크기
SHOW PARAMETER log_buffer;
```

### PGA(Process Global Area)

- **서버 프로세스별** 사설 메모리. 정렬(sort), 해시 조인, 세션 변수 등 작업 공간.
- 파라미터 `pga_aggregate_target`(수동/자동), `pga_aggregate_limit`(상한).
- **UGA**(User Global Area)는 전용서버에선 PGA 내, **Shared Server**에선 **SGA(Large Pool)** 로 이동.

```sql
-- PGA 사용량(세션/인스턴스 레벨)
SELECT name, round(value/1024/1024) AS mb
FROM v$pgastat;

SELECT sid, name, value
FROM v$sesstat s JOIN v$statname n ON s.statistic#=n.statistic#
WHERE n.name LIKE 'session pga memory%';
```

### 메모리 파라미터(AMM/A-SGA)

- **MEMORY_TARGET / SGA_TARGET / PGA_AGGREGATE_TARGET**
  - AMM(메모리 타깃 기반) 또는 ASMM(SGA 자동 관리) 구성에 따라 동적 조절.
- 실무에선 **ASMM(SGA_TARGET + PGA_AGGREGATE_TARGET)** 조합이 보편적.

```sql
SHOW PARAMETER sga_target;
SHOW PARAMETER pga_aggregate_target;
SHOW PARAMETER memory_target;
```

---

## 저장소 아키텍처 — 데이터/컨트롤/리두/아카이브/임시/UNDO

### 컨트롤 파일(Control Files)

- **데이터베이스 메타데이터** (DB 이름/ID, 체크포인트 SCN, 데이터파일/리두로그 목록 등).
- 보통 **다중화**(복수 경로)에 배치.
```sql
SELECT name FROM v$controlfile;
SELECT * FROM v$database;     -- DBID, ARCHIVELOG 모드 등
```

### & 테이블스페이스(TS)

- **테이블스페이스**는 논리 단위; 그 안에 **세그먼트(테이블, 인덱스)** 들이 저장.
- **데이터 파일**은 실제 OS 파일. 각 TS는 하나 이상의 데이터 파일 보유.
```sql
SELECT tablespace_name, file_name, bytes/1024/1024 AS mb
FROM dba_data_files
ORDER BY tablespace_name, file_id;

-- TEMP 파일
SELECT tablespace_name, file_name, bytes/1024/1024 AS mb
FROM dba_temp_files;
```

### & 아카이브

- **LGWR**가 **Redo Log Buffer → 온라인 리두로그 파일**에 순차 기록.
- **리두 로그 그룹/멤버**로 구성(다중화 권장). **로그 스위치** 시 다음 그룹으로 전환.
- **ARCHIVELOG 모드**에선 스위치된 로그를 **ARCn**이 **아카이브 로그**로 복제.
```sql
SELECT group#, bytes/1024/1024 AS size_mb, members, archived, status
FROM v$log
ORDER BY group#;

SELECT group#, member FROM v$logfile ORDER BY group#, member;

ARCHIVE LOG LIST;  -- 아카이브 모드/대상 경로 확인
```

### UNDO/임시(작업공간)

- **UNDO 테이블스페이스**: 일관 읽기/롤백/플래시백 쿼리의 기반(Undo 세그먼트/레코드 저장).
- **TEMP 테이블스페이스**: 정렬/해시 등 PGA 초과 작업 시 **Temp Spill** 용.
```sql
-- 현재 UNDO TS
SHOW PARAMETER undo_tablespace;

SELECT tablespace_name, status FROM dba_tablespaces WHERE contents='UNDO';

-- Temp 사용량
SELECT * FROM v$tempseg_usage;
```

### 파라미터 파일/패스워드 파일/ADR

- **spfile**(바이너리) / **pfile**(텍스트). spfile 권장(동적 변경/퍼시스트).
- **orapwd** 로 **패스워드 파일** 생성(SYSDBA 원격 인증).
- **ADR**: 알럿/트레이스/코어 덤프 등 진단 데이터 표준 디렉터리.

```sql
-- 현재 파라미터 파일 출처
SHOW PARAMETER spfile;

-- pfile ↔ spfile 변환
CREATE PFILE='/tmp/initORCL.ora' FROM SPFILE;
CREATE SPFILE FROM PFILE='/tmp/initORCL.ora';
```

---

## 일관성/SCN/Redo/Undo — 읽기/쓰기의 핵심 메커니즘

### SCN(System Change Number)

- DB 전역의 **논리적 시계**. 커밋/체크포인트 등 **일관성 경계**를 표시.
- **일관 읽기(Read Consistency)**: 쿼리 시작 시점의 SCN을 기준으로, Undo로부터 **CR(Consistent Read) 블록**을 재구성.

### DML 흐름(핵심)

1. 서버 프로세스가 **Buffer Cache** 에서 대상 블록을 핀(pin)·래치 후 변경.
2. 변경 전(after/before) 정보에 기반해 **Redo**(재현 로그) 생성 → **Redo Log Buffer** 에 기록.
3. 변경된 버퍼는 **Dirty** 로 표시(아직 데이터 파일엔 미반영).
4. **COMMIT** → **LGWR** 가 해당 트랜잭션의 Redo를 **온라인 리두 로그**로 **플러시**(fsync).
   - 커밋의 **지연 시간** ≒ LGWR 플러시 I/O 왕복 + 로그 파일 동기화 비용.
5. 이후 적당한 시점(체크포인트/버퍼 프레셔 등)에 **DBWn** 이 Dirty Buffer를 **데이터파일**에 기록.

> **원자성/내구성**은 **Redo** 로 보장, **일관성**은 **Undo+SCN** 으로 보장.

```sql
-- 세션에서 DML → 커밋 → 리두 생성량 관찰(요약)
SELECT name, value
FROM v$sysstat
WHERE name IN ('redo size','user commits')
ORDER BY name;

-- 간단한 DML 실습
CREATE TABLE t_demo (id NUMBER PRIMARY KEY, payload VARCHAR2(100));
INSERT INTO t_demo VALUES (1, 'hello');
COMMIT;

-- 리두 증가 확인
SELECT name, value FROM v$sysstat WHERE name = 'redo size';
```

### Undo 기반 일관 읽기

- 쿼리 시작 시 **쿼리 SCN** 을 확보. 읽어온 블록의 변경 SCN이 더 크면, **Undo 레코드**를 따라가 **CR 블록**을 만들어 반환.
- **Undo 보관 기간**(`undo_retention`)이 짧거나 급격한 DML로 Undo가 재사용되면, **ORA-01555(snapshot too old)** 발생.

```sql
SHOW PARAMETER undo_retention;

-- Undo 소비가 많은 쿼리 패턴 탐지(예: 큰 정렬/해시 조인과 교차)
SELECT name, value FROM v$sysstat
WHERE name LIKE 'table scan rows gotten' OR name LIKE 'consistent gets';
```

---

## 체크포인트 & 리커버리 — 장애/재시작 시나리오

### 체크포인트(Checkpoint)

- 특정 시점의 **SCN을 데이터파일 헤더/컨트롤 파일**에 기록하여 **복구 경계** 설정.
- **목적**: 크래시 후 재생해야 할 리두(REDO) 범위를 단축 → **인스턴스 리커버리 시간 축소**.
- **CKPT** 가 트리거되면 DBWn이 Dirty Buffer를 가능한 한 밀어내고 헤더 SCN 갱신.

```sql
-- 최근 체크포인트 정보
SELECT checkpoint_change# FROM v$database;
SELECT * FROM v$instance_recovery;
```

### 인스턴스 리커버리(Instance/Crash Recovery)

- **전제**: ARCHIVELOG 여부와 무관. 인스턴스가 **비정상 종료**되면 **SMON** 이 자동 수행.
- **REDO Roll Forward**: 종료 시점까지의 리두 재생.
- **Undo Rollback**: 커밋되지 않은 트랜잭션을 되돌림.

### 미디어 리커버리(Media Recovery)

- **데이터 파일 손상/유실** 시, **백업 + 아카이브 로그** 로 **시점복구(PITR)** 등 수행.
- **ARCHIVELOG 모드**가 필수(무정지 백업, PITR, Data Guard 등 전제).

```sql
-- 아카이브 모드 확인
ARCHIVE LOG LIST;

-- FRA(플래시 리커버리 영역) 설정 확인
SHOW PARAMETER db_recovery_file_dest;
SHOW PARAMETER db_recovery_file_dest_size;
```

---

## 동시성 제어 — 래치/뮤텍스/엔큐/트랜잭션 락

### & 뮤텍스(Mutex)

- **초단기, 매우 경량**의 **스핀락** 계열. **라이브러리 캐시/버퍼 헤더** 등 **핫 구조체** 보호.
- 11g+ 부터 라이브러리 캐시 보호에 **뮤텍스** 도입.

```sql
-- 래치 통계 개요
SELECT name, gets, misses, immediate_gets, immediate_misses
FROM v$latch
ORDER BY misses DESC FETCH FIRST 20 ROWS ONLY;
```

### & TM/TX 락

- 엔큐는 **좀 더 오래 지속**될 수 있는 **큐형 락**.
- **TM**: 오브젝트(테이블) 레벨 락. **DDL/DML 충돌** 조정.
- **TX**: 트랜잭션(행) 락. **행 레벨 충돌** 시 대기.
- **Row-level Lock** 은 **버퍼 헤더(ITL 슬롯)** 에 기록되며, **Wait Event** 로 관찰.

```sql
-- 대기 이벤트 상위 (세션/시스템)
SELECT * FROM (
  SELECT event, total_waits, time_waited_micro/1e6 AS sec_waited
  FROM v$system_event
  ORDER BY time_waited_micro DESC
) WHERE ROWNUM <= 20;

-- 세션별 현재 대기
SELECT sid, event, state, wait_time, seconds_in_wait
FROM v$session
WHERE state <> 'WAITED SHORT TIME';
```

---

## 옵티마이저/실행계획/버퍼 캐시 동작(아키텍처 관점 요약)

- **코스트 기반 옵티마이저(CBO)**: 통계(카디널리티/히스토그램/NDV)와 시스템 상태를 기반으로 액세스/조인 전략 선택.
- **버퍼 캐시**:
  - **cache hit ratio** 는 **하나의 지표**일 뿐이며, 무조건 높다고 좋은 게 아님.
  - 단, 개념상 비율은 $$ \text{Hit Ratio} \approx 1 - \frac{\text{physical reads}}{\text{logical reads}} $$ 로 설명됩니다.
  - 실제 튜닝은 **상위 대기 이벤트/핫 블록 경합/리두/UNDO 사용 패턴**으로 접근해야 정확합니다.

```sql
-- 실행계획/실제 수행 통계
EXPLAIN PLAN FOR
SELECT /*+ index(t_demo pk_t_demo) */ *
FROM t_demo
WHERE id = 1;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);

-- AWR/ASH가 있다면 상위 SQL/대기 이벤트로 병목을 확인 (라이선스/에디션 주의)
```

---

## 인스턴스 라이프사이클 — STARTUP/SHUTDOWN & NOMOUNT/MOUNT/OPEN

### 단계

1. **STARTUP NOMOUNT**
   - 파라미터 파일 읽고 **SGA/PGA/백그라운드** 초기화. **컨트롤 파일은 아직 미개방**.
2. **STARTUP MOUNT**
   - **컨트롤 파일 오픈**, 데이터파일/리두로그 목록 인식. 복구 필요성 판단.
3. **ALTER DATABASE OPEN**
   - 데이터파일/리두로그 오픈, 정상 서비스 시작.

```sql
-- SYSDBA 세션에서
STARTUP NOMOUNT;
ALTER DATABASE MOUNT;
ALTER DATABASE OPEN;

-- 종료
SHUTDOWN IMMEDIATE;  -- 일반적 권장
-- SHUTDOWN TRANSACTIONAL / ABORT 등도 상황에 따라 사용
```

### 파라미터/서비스 등록/로그 스위치

```sql
-- 주요 파라미터 조회
SHOW PARAMETER db_name;
SHOW PARAMETER control_files;
SHOW PARAMETER pga_aggregate_target;
SHOW PARAMETER sga_target;

-- 로그 스위치 유발(테스트 용)
ALTER SYSTEM SWITCH LOGFILE;
```

---

## 네트워킹/서비스/리스너 — 다중 서비스와 로드밸런싱(개요)

- **서비스(Service)** 개념: 하나의 DB에서 여러 개 **서비스 명**을 노출. 워크로드 분리/로드 밸런싱/Failover 정책에 활용.
- **TNS** 이름 → **서비스명** 매핑. 클라이언트에서 **EZCONNECT**(`host:port/service`) 도 흔히 사용.

```sql
-- 서비스 확인(동적 등록)
SELECT name, network_name FROM v$services;

-- 세션이 어떤 서비스로 접속했는지
SELECT sid, serial#, username, service_name
FROM v$session
WHERE username IS NOT NULL;
```

---

## RAC/ASM/메타 아키텍처(개요)

- **RAC**(Real Application Clusters): 여러 인스턴스가 **하나의 데이터베이스**(공유 스토리지)를 **동시에** 오픈.
  - **GCS/GES**(Global Cache/Enqueue Service)로 버퍼/락 일관성 유지.
  - 백그라운드: **LMON/LMD/LMS/LCK** 등.
- **ASM**(Automatic Storage Management): 오라클 전용 볼륨/파일시스템.
  - **디스크 그룹** 으로 묶어 **스트라이핑/미러링** 제공.
  - 백그라운드: **RBAL/ARBx** (리밸런스).
- **Data Guard**: **REDO 전송/적용**으로 **물리/논리 스탠바이** 구성. (아키텍처상 LGWR/ARCn와 통합)

---

## 모니터링/진단 — 동시성, I/O, 리두, Undo, 메모리

### 시스템 지표 훑어보기

```sql
-- 시스템 통계 상위
SELECT * FROM (
  SELECT name, value
  FROM v$sysstat
  ORDER BY value DESC
) WHERE ROWNUM <= 30;

-- 대기 이벤트 상위
SELECT * FROM (
  SELECT event, total_waits, time_waited_micro/1e6 AS seconds
  FROM v$system_event
  ORDER BY time_waited_micro DESC
) WHERE ROWNUM <= 20;
```

### 버퍼 캐시/리두/아카이브 관측

```sql
-- 버퍼 캐시 블록 상태
SELECT status, COUNT(*)
FROM v$bh
GROUP BY status;

-- 리두 생성률(초당)
SELECT VALUE AS redo_bytes
FROM v$sysstat
WHERE name = 'redo size';

-- 아카이브 로그 생성 이력
SELECT sequence#, first_time, next_time, blocks, archived, status
FROM v$archived_log
ORDER BY sequence# DESC FETCH FIRST 10 ROWS ONLY;
```

### PGA/Temp 사용

```sql
-- PGA 사용 추이
SELECT name, value
FROM v$pgastat;

-- Temp 사용 현황
SELECT s.sid, u.tablespace, u.blocks*ts.block_size/1024/1024 AS mb_used
FROM v$tempseg_usage u
JOIN v$session s ON s.saddr = u.session_addr
JOIN dba_tablespaces ts ON ts.tablespace_name = u.tablespace
ORDER BY mb_used DESC;
```

---

## 실습 시나리오 — “작게 만들어 보고 아키텍처 관측하기”

### 준비: 샘플 스키마/테이블 생성

```sql
-- 샘플 테이블/인덱스 생성
CREATE TABLE sales_demo (
  sale_id     NUMBER GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
  cust_id     NUMBER NOT NULL,
  sale_dt     DATE   NOT NULL,
  amount      NUMBER(12,2) NOT NULL,
  channel     VARCHAR2(20),
  created_at  DATE DEFAULT SYSDATE
);

CREATE INDEX ix_sales_demo_cust ON sales_demo(cust_id);
CREATE INDEX ix_sales_demo_dt   ON sales_demo(sale_dt);

-- 더미 데이터 주입(간단 버전)
BEGIN
  FOR i IN 1..100000 LOOP
    INSERT INTO sales_demo (cust_id, sale_dt, amount, channel)
    VALUES (TRUNC(DBMS_RANDOM.VALUE(1, 10000)),
            TRUNC(SYSDATE - DBMS_RANDOM.VALUE(0, 365)),
            ROUND(DBMS_RANDOM.VALUE(10, 1000), 2),
            CASE MOD(i,4) WHEN 0 THEN 'web' WHEN 1 THEN 'store' WHEN 2 THEN 'app' ELSE 'call' END);
    IF MOD(i,1000)=0 THEN COMMIT; END IF;
  END LOOP;
  COMMIT;
END;
/
```

### DML/커밋과 리두 관측

```sql
-- 현재 리두 사이즈
SELECT name, value FROM v$sysstat WHERE name = 'redo size';

-- 대량 DML
UPDATE sales_demo SET amount = amount * 1.05 WHERE sale_dt >= TRUNC(SYSDATE) - 30;
COMMIT;

-- 증가 확인
SELECT name, value FROM v$sysstat WHERE name = 'redo size';
```

**개념 체크**: `UPDATE` 로 인해 **Redo Log Buffer** 에 변경 내역이 기록되고, **COMMIT** 시 **LGWR** 가 플러시. 데이터파일 반영(DBWn)은 나중에 일어날 수 있습니다.

### 읽기 일관성과 Undo

```sql
-- 긴 쿼리 세션 A (예: 전체 집계)
SELECT /*+ FULL(s) PARALLEL(4) */ channel, SUM(amount)
FROM sales_demo s
GROUP BY channel;

-- 동시에 세션 B에서 특정 기간의 데이터를 대규모 UPDATE/DELETE
-- 충분한 undo_retention과 undo 공간이 없으면 세션 A에서 ORA-01555 발생 가능
```

**핵심**: 세션 A는 쿼리 시작 시점의 **SCN** 기준으로 읽기 때문에, 세션 B가 데이터를 바꾸더라도 **Undo** 를 따라가 **CR 블록**을 재구성합니다. Undo 보관이 부족하면 **snapshot too old**.

### 체크포인트/로그 스위치 관찰

```sql
-- 로그 스위치 강제
ALTER SYSTEM SWITCH LOGFILE;

-- 최근 아카이브 로그/로그 시퀀스 확인
SELECT sequence#, first_change#, next_change#, archived, status
FROM v$log
ORDER BY sequence#;

SELECT sequence#, applied, first_time, next_time
FROM v$archived_log
ORDER BY sequence# DESC FETCH FIRST 5 ROWS ONLY;

-- DBWn/CKPT 활동은 alert 로그/ASH/AWR로 관측 가능
```

---

## 성능·용량 감각을 위한 간단 수식(개념적 가이드)

> **주의**: 아래 수식들은 “감각 형성”을 위한 개념적 지표입니다. 실전 튜닝은 **대기 이벤트/프로파일링/실측** 중심으로 하세요.

- **Redo 생성률**:
  $$ \text{redo\_rate (MB/s)} \approx \frac{\text{redo size (MB)}}{\text{interval (s)}} $$
  - LGWR 대기, 로그 파일 I/O 성능, 아카이브 처리량 추산에 활용.
- **대략적 캐시 히트 감각**:
  $$ \text{Hit Ratio} \approx 1 - \frac{\text{Physical Reads}}{\text{DB Block Gets + Consistent Gets}} $$
  - 단, 히트율이 높아도 경합/핫블록/Redo·Undo·Latch 대기가 병목일 수 있음.
- **PGA/Temp 임계 판단**:
  $$ \text{Spill Ratio} \approx \frac{\text{temp read/write blocks}}{\text{total sort/merge operations}} $$
  - Spill 비율이 높으면 `pga_aggregate_target` 증설/워크로드 튜닝/쿼리 리라이트 고려.

---

## 운영 팁 — 기본 아키텍처를 살리는 설정/관행

1. **ARCHIVELOG + 적절한 FRA**: 무정지 백업/PITR/Data Guard 가능성 확보.
2. **리두 로그 크기/개수 설계**: 너무 잦은 로그 스위치는 체크포인트 빈발/아카이브 부하.
   - 일반적 권장: **20~30분 주기의 로그 스위치**가 흔한 타깃(워크로드에 따라 조정).
3. **Undo/Temp 용량 계획**: 긴 OLAP 쿼리/배치 시나리오 고려. `undo_retention`/`temp` 디스크 I/O 감안.
4. **SGA/PGA 목표치 설정**: AMM보다는 **ASMM + PGA 타깃** 조합이 예측 가능.
5. **통계/히스토그램 관리(DBMS_STATS)**: 옵티마이저의 올바른 카디널리티 추정 보장.
6. **바인드 변수 사용**: 하드 파싱/라이브러리 캐시 경합 감소.
7. **AWR/ASH로 관측**: 상위 SQL/이벤트/세션 프로파일로 병목을 과학적으로 추적.
8. **데이터파일/리두/컨트롤 다중화**: 스토리지/ASM 정책과 함께 가용성 강화.

---

## 체크리스트 — “기본 아키텍처 점검 10선”

1. `ARCHIVELOG` 모드 여부/아카이브 대상/FRA 용량.
2. 온라인 리두 로그 **그룹 수/크기/IOPS** 적정성.
3. `undo_tablespace`, `undo_retention` 과 Undo 공간.
4. TEMP TS 용량/`v$tempseg_usage` 감시.
5. `sga_target`, `pga_aggregate_target` 및 `pgastat`/`sgastat` 관측.
6. 상위 대기 이벤트( `v$system_event` / AWR Top Events ).
7. 상위 SQL(쿼리) 프로파일(AWR Top SQL / SQL Monitor).
8. 데이터파일/컨트롤/리두 다중화와 백업/복구 전략(RMAN).
9. 리스너/서비스 설정, 전용/공유 서버 정책.
10. 경합 지표(래치/뮤텍스/핫블록)와 파싱/바인드 사용 현황.

---

## 부록 — 자주 보는 뷰(빠른 탐색)

```sql
-- 인스턴스/DB
SELECT * FROM v$instance;
SELECT * FROM v$database;

-- 파일
SELECT * FROM v$datafile;
SELECT * FROM v$controlfile;
SELECT * FROM v$log;
SELECT * FROM v$logfile;

-- 메모리
SELECT * FROM v$sga;
SELECT * FROM v$sgainfo;
SELECT * FROM v$sgastat;
SELECT * FROM v$pgastat;

-- 세션/프로세스/대기
SELECT sid, serial#, username, status, machine, program
FROM v$session
WHERE username IS NOT NULL;

SELECT * FROM v$process;

SELECT sid, event, wait_class, state, seconds_in_wait
FROM v$session
WHERE state <> 'WAITED SHORT TIME';

-- 옵티마이저/통계
SELECT * FROM dba_tab_statistics WHERE table_name='SALES_DEMO';
```

---

## 마무리 요약

- 오라클의 **기본 아키텍처**는 “**인스턴스(메모리+프로세스)** 가 **데이터베이스(파일)** 를 관리”하는 구조입니다.
- **Redo/Undo/SCN** 삼각 구도가 **원자성/일관성/내구성**을 구현합니다.
- **LGWR(커밋 지연)**, **DBWn(지연 쓰기)**, **CKPT(체크포인트)**, **SMON/PMON(리커버리/정리)** 가 핵심 역할.
- **SGA/PGA/Undo/Temp/Redo/Archive** 의 용량 계획과 **상위 대기 이벤트 기반 관측**이 실무 안정성/성능의 핵심입니다.
