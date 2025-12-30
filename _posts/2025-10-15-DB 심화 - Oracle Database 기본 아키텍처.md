---
layout: post
title: DB 심화 - Oracle Database 기본 아키텍처
date: 2025-10-15 17:25:23 +0900
category: DB 심화
---
# Oracle Database 기본 아키텍처 (19c 기준)

## 인스턴스와 데이터베이스

오라클은 **인스턴스(Instance)** 와 **데이터베이스(Database)** 로 분리된 **논리 모델**을 갖습니다.

- **인스턴스**: SGA(옵션인 In-Memory Column Store 포함), PGA, 그리고 수십 개의 **백그라운드 프로세스**로 구성됩니다. 인스턴스는 “메모리와 프로세스의 집합”으로, 가동 중일 때만 존재합니다.
- **데이터베이스**: **데이터 파일**, **컨트롤 파일**, **리두 로그(온라인/아카이브)**, **임시 파일**, **파라미터/패스워드 파일**, **ADR(진단 디렉터리)** 등 **물리 파일의 집합**입니다. 인스턴스가 내려가도 파일은 디스크에 남아 있습니다.

**핵심 상호 작용**은 다음과 같습니다.

1. 클라이언트가 SQL을 보내면 서버 프로세스가 파싱, 최적화, 실행을 수행합니다.
2. **Buffer Cache** 에서 블록을 읽고, DML 시 변경하여 **Dirty Buffer** 로 표시합니다.
3. 변경 사항은 **Redo Log Buffer** 에 리두 레코드로 기록되고, **커밋 시 LGWR가 이를 플러시**합니다.
4. **DBWn** 이 적절한 시점에 Dirty Buffer를 **데이터 파일**로 기록합니다.
5. **CKPT** 가 체크포인트 정보를 컨트롤 파일과 데이터 파일 헤더에 갱신합니다.
6. **SMON/PMON** 등 백그라운드 프로세스가 회복, 정리, 모니터링을 수행합니다.

---

## 프로세스 아키텍처 — 백그라운드, 서버, 리스너

### 프로세스 유형

- **서버 프로세스(Server Process)**
  - **전용(Dedicated)**: 클라이언트 세션당 서버 프로세스가 1:1로 할당됩니다. 구조가 단순하고 동작이 예측 가능합니다.
  - **공유(Shared Server, MTS)**: 다수의 세션이 **디스패처(Dispatcher)** 를 통해 **공유 서버** 프로세스 풀을 공유합니다. 고동시성과 짧은 트랜잭션 환경에 유리합니다.
- **백그라운드(Background) 프로세스** — 필수 및 옵션 프로세스로 구성됩니다.
  - **PMON**(Process Monitor): 비정상 종료된 세션을 정리하고 리소스를 회수합니다.
  - **SMON**(System Monitor): **인스턴스/크래시 리커버리**를 수행하고 임시 세그먼트를 청소합니다.
  - **DBWn**(Database Writer): Dirty Buffer를 데이터 파일에 기록합니다.
  - **LGWR**(Log Writer): Redo Log Buffer의 내용을 온라인 리두 로그 파일에 순차 기록하며, **커밋 지연을 최소화**하는 핵심 역할을 합니다.
  - **CKPT**(Checkpoint): 체크포인트 발생 시 데이터파일 헤더와 컨트롤 파일의 SCN을 갱신합니다.
  - **ARCn**(Archiver): ARCHIVELOG 모드에서 **온라인 리두 로그를 아카이브 로그**로 복제합니다.
  - **MMON/MMNL**: AWR 스냅샷 생성 및 통계, 얼럿 관리를 담당합니다.
  - **LREG**: 서비스 정보를 리스너에 동적으로 등록합니다.
  - **CJQ0**: 스케줄러 잡을 관리하며, **Jnnn** 잡 슬레이브를 생성합니다.
  - **RVWR**: (옵션) Flashback Database를 위한 리두 버퍼 기록을 담당합니다.
  - **RECO**: 분산 트랜잭션 복구를 수행합니다.
  - **QMNC/Qnnn**: AQ(Advanced Queuing) 관련 작업을 처리합니다.
  - **Dnnn**: Shared Server 모드에서 디스패처 역할을 합니다.
  - **FBDA**: Flashback Data Archive(옵션) 백그라운드 프로세스입니다.
  - **RVWR, FBDA, MMON 등은 해당 기능이 활성화되었을 때만 등장**합니다.

> RAC/ASM 환경에서는 **LMON/LMDn/LMSn/LCKn**(글로벌 캐시/락 관리) 및 **RBAL/ARBx**(ASM 리밸런싱) 등 추가 프로세스가 존재합니다.

### 리스너와 접속 흐름(Oracle Net)

- **리스너(listener.ora)** 는 특정 포트(기본 1521)에서 대기하며, 접속 요청을 받아 **서버 프로세스**를 연결해줍니다.
- 전용 서버 모드에서는 클라이언트가 곧바로 서버 프로세스와 세션을 형성합니다.
- 공유 서버 모드에서는 **디스패처**가 요청을 큐에 적재하고, **공유 서버 프로세스**가 이를 처리합니다.

```bash
# 리스너 상태 확인
lsnrctl status

# 데이터베이스 서비스가 리스너에 동적 등록되었는지 확인
lsnrctl services
```

---

## 메모리 아키텍처 — SGA와 PGA

### SGA(시스템 글로벌 영역) 구성

- **Shared Pool**
  - **Library Cache**: 파싱된 SQL과 실행계획(커서)을 캐시합니다.
  - **Data Dictionary Cache(Row Cache)**: 오브젝트, 사용자, 권한 등 메타데이터를 캐시합니다.
  - **Server Result Cache**(옵션)도 포함됩니다.
- **Database Buffer Cache**
  - 데이터 파일 블록의 버퍼를 관리합니다. **Clean/Dirty** 상태로 구분되며, **LRU(또는 버킷/핫/콜드 리스트)** 정책에 따라 관리됩니다.
  - 각 버퍼는 **SCN, ITL(Interested Transaction List), 버전** 관련 메타데이터를 포함합니다.
- **Redo Log Buffer**
  - 리두 엔트리(변경 이력)를 위한 임시 버퍼입니다. **LGWR**가 **커밋 시 즉시** 혹은 주기적으로 이를 플러시합니다.
- **Large Pool**(옵션)
  - 대용량 메모리 연산(병렬 실행 메시지 버퍼, RMAN 채널, Shared Server UGA 등)에 사용됩니다.
- **Java Pool**(옵션), **Streams Pool**(옵션)도 있습니다.
- **In-Memory Column Store(IM column store)** — 12c R2부터 제공되는 옵션 기능으로, 데이터를 컬럼 형식으로 메모리에 보관하여 벡터화 및 인메모리 스캔 성능을 가속합니다.

```sql
-- SGA 요약 정보 확인
SELECT * FROM v$sga;
SELECT * FROM v$sgainfo ORDER BY name;

-- Shared Pool과 Buffer Cache 사용량 확인(요약)
SELECT pool, name, bytes/1024/1024 AS mb
FROM v$sgastat
ORDER BY pool, name;

-- Redo Log Buffer 크기 확인
SHOW PARAMETER log_buffer;
```

### PGA(프로세스 글로벌 영역)

- **서버 프로세스별**로 할당되는 사설 메모리 영역입니다. 정렬(sort), 해시 조인, 세션 변수 등 작업 공간으로 사용됩니다.
- `pga_aggregate_target`(수동 또는 자동 관리), `pga_aggregate_limit`(상한 설정) 파라미터로 관리됩니다.
- **UGA**(User Global Area)는 전용 서버 모드에서는 PGA 내부에, **Shared Server** 모드에서는 **SGA(Large Pool)** 로 이동됩니다.

```sql
-- PGA 사용량 확인(인스턴스 및 세션 레벨)
SELECT name, round(value/1024/1024) AS mb
FROM v$pgastat;

SELECT sid, name, value
FROM v$sesstat s JOIN v$statname n ON s.statistic#=n.statistic#
WHERE n.name LIKE 'session pga memory%';
```

### 메모리 파라미터(AMM/A-SGA)

- **MEMORY_TARGET / SGA_TARGET / PGA_AGGREGATE_TARGET**
  - AMM(메모리 타깃 기반 자동 관리) 또는 ASMM(SGA 자동 관리) 구성에 따라 동적으로 메모리가 조절됩니다.
- 실무에서는 **ASMM(SGA_TARGET + PGA_AGGREGATE_TARGET)** 조합이 예측 가능성과 안정성 측면에서 보편적으로 사용됩니다.

```sql
-- 관련 파라미터 확인
SHOW PARAMETER sga_target;
SHOW PARAMETER pga_aggregate_target;
SHOW PARAMETER memory_target;
```

---

## 저장소 아키텍처 — 데이터, 컨트롤, 리두, 아카이브, 임시, UNDO

### 컨트롤 파일(Control Files)

- **데이터베이스 메타데이터** (DB 이름/ID, 체크포인트 SCN, 데이터파일/리두로그 목록 등)를 저장합니다.
- 가용성을 위해 보통 **다중화**(복수 경로)되어 배치됩니다.
```sql
-- 컨트롤 파일 경로 확인
SELECT name FROM v$controlfile;
-- 데이터베이스 기본 정보 확인 (DBID, ARCHIVELOG 모드 등)
SELECT * FROM v$database;
```

### 테이블스페이스(Tablespace)와 데이터 파일

- **테이블스페이스**는 데이터 저장의 논리적 단위이며, 그 안에 **세그먼트(테이블, 인덱스 등)** 가 저장됩니다.
- **데이터 파일**은 실제 OS 파일입니다. 각 테이블스페이스는 하나 이상의 데이터 파일을 보유합니다.
```sql
-- 테이블스페이스와 데이터 파일 정보 확인
SELECT tablespace_name, file_name, bytes/1024/1024 AS mb
FROM dba_data_files
ORDER BY tablespace_name, file_id;

-- TEMP 파일 정보 확인
SELECT tablespace_name, file_name, bytes/1024/1024 AS mb
FROM dba_temp_files;
```

### 리두 로그(Redo Log)와 아카이브(Archive)

- **LGWR**가 **Redo Log Buffer → 온라인 리두로그 파일**에 변경 사항을 순차적으로 기록합니다.
- **리두 로그 그룹/멤버**로 구성되며(다중화 권장), **로그 스위치** 시 다음 그룹으로 전환합니다.
- **ARCHIVELOG 모드**에서는 스위치된 로그를 **ARCn**이 **아카이브 로그**로 복제하여 보관합니다.
```sql
-- 리두 로그 그룹 상태 확인
SELECT group#, bytes/1024/1024 AS size_mb, members, archived, status
FROM v$log
ORDER BY group#;
-- 리두 로그 멤버(파일) 경로 확인
SELECT group#, member FROM v$logfile ORDER BY group#, member;
-- 아카이브 모드 및 대상 경로 확인
ARCHIVE LOG LIST;
```

### UNDO와 임시(Temp) 테이블스페이스

- **UNDO 테이블스페이스**: 일관된 읽기(Read Consistency), 롤백, 플래시백 쿼리의 기반이 되는 Undo 세그먼트/레코드를 저장합니다.
- **TEMP 테이블스페이스**: 정렬, 해시 조인 등 PGA 공간을 초과하는 큰 작업 시 **Temp Spill** 공간으로 사용됩니다.
```sql
-- 현재 사용 중인 UNDO 테이블스페이스 확인
SHOW PARAMETER undo_tablespace;
-- UNDO 테이블스페이스 상태 확인
SELECT tablespace_name, status FROM dba_tablespaces WHERE contents='UNDO';
-- Temp 세그먼트 사용 현황 확인
SELECT * FROM v$tempseg_usage;
```

### 파라미터 파일, 패스워드 파일, ADR

- **spfile**(바이너리) / **pfile**(텍스트). 동적 변경이 유지되는 spfile 사용을 권장합니다.
- **orapwd** 유틸리티로 **패스워드 파일**을 생성하여 SYSDBA 권한의 원격 인증을 가능하게 합니다.
- **ADR**(Automatic Diagnostic Repository): 알럿, 트레이스, 코어 덤프 등 진단 데이터를 위한 표준 디렉터리 구조입니다.

```sql
-- 현재 사용 중인 파라미터 파일 출처 확인
SHOW PARAMETER spfile;
-- pfile과 spfile 간 변환 (관리 작업용)
CREATE PFILE='/tmp/initORCL.ora' FROM SPFILE;
CREATE SPFILE FROM PFILE='/tmp/initORCL.ora';
```

---

## 일관성, SCN, Redo, Undo — 읽기와 쓰기의 핵심 메커니즘

### SCN(System Change Number)

- 데이터베이스 전역의 **논리적 시계** 역할을 합니다. 커밋, 체크포인트 등 **일관성의 경계**를 표시하는 데 사용됩니다.
- **일관 읽기(Read Consistency)**: 쿼리가 시작된 시점의 SCN을 기준으로, 필요시 Undo 데이터를 이용해 **CR(Consistent Read) 블록**을 재구성하여 반환합니다.

### DML 처리 흐름(핵심)

1. 서버 프로세스가 **Buffer Cache** 에서 대상 블록을 핀(pin)하고 래치를 획득한 후 변경합니다.
2. 변경 전/후 정보를 기반으로 **Redo**(재현 로그)를 생성하여 **Redo Log Buffer** 에 기록합니다.
3. 변경된 버퍼는 **Dirty** 상태로 표시됩니다(아직 데이터 파일에는 반영되지 않음).
4. **COMMIT** 명령이 실행되면, **LGWR** 가 해당 트랜잭션의 Redo 레코드를 **온라인 리두 로그 파일**로 **플러시**(fsync)합니다.
   - 커밋의 **지연 시간**은 기본적으로 LGWR 플러시 I/O 왕복 시간과 로그 파일 동기화 비용에 의해 결정됩니다.
5. 이후 적절한 시점(체크포인트 또는 버퍼 프레셔 등)에 **DBWn** 이 Dirty Buffer를 **데이터파일**에 기록합니다.

> **원자성(Atomicity)**과 **내구성(Durability)**은 **Redo** 메커니즘으로 보장되며, **일관성(Consistency)**은 **Undo**와 **SCN**에 의해 보장됩니다.

```sql
-- DML 수행 후 커밋 시 리두 생성량 관찰
SELECT name, value
FROM v$sysstat
WHERE name IN ('redo size','user commits')
ORDER BY name;

-- 간단한 DML로 실습
CREATE TABLE t_demo (id NUMBER PRIMARY KEY, payload VARCHAR2(100));
INSERT INTO t_demo VALUES (1, 'hello');
COMMIT;
-- 리두 크기 증가 확인
SELECT name, value FROM v$sysstat WHERE name = 'redo size';
```

### Undo 기반의 일관 읽기

- 쿼리가 시작될 때 **쿼리 SCN** 을 확보합니다. 읽으려는 블록의 변경 SCN이 쿼리 SCN보다 크면, **Undo 레코드**를 재적용하여 **CR 블록**을 생성해 반환합니다.
- **Undo 보관 기간**(`undo_retention`)이 너무 짧거나 급격한 DML로 인해 필요한 Undo 데이터가 재사용되면, **ORA-01555(snapshot too old)** 오류가 발생할 수 있습니다.

```sql
-- Undo 보관 기간 확인
SHOW PARAMETER undo_retention;
-- 일관 읽기와 관련된 시스템 통계 확인
SELECT name, value FROM v$sysstat
WHERE name LIKE 'table scan rows gotten' OR name LIKE 'consistent gets';
```

---

## 체크포인트와 리커버리 — 장애 및 재시작 시나리오

### 체크포인트(Checkpoint)

- 특정 시점의 **SCN을 데이터파일 헤더와 컨트롤 파일**에 기록하여 **복구의 시작 지점**을 설정합니다.
- **주요 목적**: 크래시(Crash) 발생 후 재생해야 할 리두(REDO)의 범위를 단축하여 **인스턴스 리커버리 시간을 최소화**하는 것입니다.
- **CKPT** 가 체크포인트를 트리거하면 DBWn은 Dirty Buffer를 가능한 한 디스크에 기록하고, 데이터 파일 헤더의 SCN을 갱신합니다.

```sql
-- 최근 체크포인트 정보 확인
SELECT checkpoint_change# FROM v$database;
-- 인스턴스 리커버리 관련 파라미터 및 상태 확인
SELECT * FROM v$instance_recovery;
```

### 인스턴스 리커버리(Instance/Crash Recovery)

- **전제 조건**: ARCHIVELOG 모드 여부와 관계없이 수행됩니다. 인스턴스가 **비정상 종료**된 후 재시작 시 **SMON** 프로세스에 의해 자동으로 수행됩니다.
- **REDO Roll Forward**: 마지막 체크포인트부터 인스턴스 종료 시점까지의 모든 리두 변경 사항을 데이터 파일에 재적용합니다.
- **Undo Rollback**: 커밋되지 않은 트랜잭션의 변경 사항을 Undo 데이터를 이용해 되돌립니다.

### 미디어 리커버리(Media Recovery)

- **데이터 파일의 손상 또는 유실** 시, **백업 파일과 아카이브 로그**를 사용하여 **시점복구(PITR)** 등을 수행합니다.
- 미디어 리커버리를 위해서는 반드시 **ARCHIVELOG 모드**로 운영되어야 합니다(무정지 백업, PITR, Data Guard 구축의 기본 전제 조건).

```sql
-- 아카이브 모드 및 FRA(플래시 리커버리 영역) 설정 확인
ARCHIVE LOG LIST;
SHOW PARAMETER db_recovery_file_dest;
SHOW PARAMETER db_recovery_file_dest_size;
```

---

## 동시성 제어 — 래치, 뮤텍스, 엔큐, 트랜잭션 락

### 뮤텍스(Mutex)와 래치(Latch)

- **뮤텍스(Mutex)**: **초단기, 매우 경량화된 스핀락(Spinlock)** 계열의 동기화 메커니즘입니다. **라이브러리 캐시, 버퍼 헤더** 등 **핫(Hot)한 내부 구조체**를 보호하는 데 사용됩니다. Oracle 11g부터 라이브러리 캐시 보호에 본격적으로 도입되었습니다.
- **엔큐(Enqueue)**: **래치보다 오래 지속될 수 있는 큐(Queue) 기반의 락** 메커니즘입니다. 공유 리소스에 대한 순서적 접근을 관리합니다.

```sql
-- 래치 통계 개요 확인 (빈번한 미스가 성능 병목일 수 있음)
SELECT name, gets, misses, immediate_gets, immediate_misses
FROM v$latch
ORDER BY misses DESC FETCH FIRST 20 ROWS ONLY;
```

### TM 락과 TX 락

- **TM 락**(Table-level Lock): 오브젝트(테이블) 레벨의 락입니다. DDL 실행 시나 다른 트랜잭션과의 DML 충돌을 조정합니다.
- **TX 락**(Transaction-level Lock): 트랜잭션(행) 레벨의 락입니다. 동일한 행을 변경하려는 트랜잭션 간의 충돌 시 대기 상태를 만듭니다.
- **Row-level Lock** 은 **버퍼 헤더의 ITL(Interested Transaction List) 슬롯**에 기록되며, **Wait Event**를 통해 그 대기 상황을 관찰할 수 있습니다.

```sql
-- 시스템 레벨 상위 대기 이벤트 확인
SELECT * FROM (
  SELECT event, total_waits, time_waited_micro/1e6 AS sec_waited
  FROM v$system_event
  ORDER BY time_waited_micro DESC
) WHERE ROWNUM <= 20;

-- 현재 세션별 대기 이벤트 확인
SELECT sid, event, state, wait_time, seconds_in_wait
FROM v$session
WHERE state <> 'WAITED SHORT TIME';
```

---

## 옵티마이저, 실행계획, 버퍼 캐시 동작 (아키텍처 관점 요약)

- **코스트 기반 옵티마이저(CBO)**: 통계(카디널리티, 히스토그램, NDV)와 시스템 상태를 기반으로 데이터 액세스 경로 및 조인 전략을 선택합니다.
- **버퍼 캐시**:
  - **캐시 히트율(Cache Hit Ratio)**은 중요한 **하나의 지표**일 뿐이며, 무조건 높은 것이 최선은 아닙니다.
  - 개념상 히트율은 $$ \text{Hit Ratio} \approx 1 - \frac{\text{physical reads}}{\text{logical reads}} $$ 공식으로 설명할 수 있습니다.
  - 실전 성능 튜닝은 **상위 대기 이벤트, 핫 블록 경합, 리두/UNDO 사용 패턴** 등을 종합적으로 분석하는 접근이 더 정확합니다.

```sql
-- 실행계획 및 실제 수행 통계 확인 예시
EXPLAIN PLAN FOR
SELECT /*+ index(t_demo pk_t_demo) */ *
FROM t_demo
WHERE id = 1;
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);

-- AWR/ASH 리포트가 있다면 상위 SQL 및 대기 이벤트를 통해 병목 지점을 확인할 수 있습니다.
```

---

## 인스턴스 라이프사이클 — STARTUP, SHUTDOWN 및 NOMOUNT, MOUNT, OPEN 단계

### 시작 단계

1. **STARTUP NOMOUNT**
   - 파라미터 파일을 읽고 **SGA, PGA, 백그라운드 프로세스**를 초기화합니다. 이 단계에서는 **컨트롤 파일은 아직 오픈되지 않습니다**.
2. **STARTUP MOUNT**
   - **컨트롤 파일을 오픈**하여 데이터파일과 리두로그 파일의 목록 및 상태를 인식합니다. 복구 필요성 여부를 판단할 수 있는 상태가 됩니다.
3. **ALTER DATABASE OPEN**
   - 모든 데이터파일과 리두로그 파일을 오픈하고, 데이터베이스를 정상 서비스 상태로 전환합니다.

```sql
-- SYSDBA 세션에서 단계별 시작
STARTUP NOMOUNT;
ALTER DATABASE MOUNT;
ALTER DATABASE OPEN;

-- 데이터베이스 종료 (일반적으로 권장되는 방식)
SHUTDOWN IMMEDIATE;
-- SHUTDOWN TRANSACTIONAL / ABORT 등도 상황에 따라 사용 가능합니다.
```

### 주요 관리 명령어

```sql
-- 주요 파라미터 값 확인
SHOW PARAMETER db_name;
SHOW PARAMETER control_files;
SHOW PARAMETER pga_aggregate_target;
SHOW PARAMETER sga_target;

-- 로그 스위치 수동 발생 (테스트 용도)
ALTER SYSTEM SWITCH LOGFILE;
```

---

## 네트워킹, 서비스, 리스너 — 다중 서비스와 로드밸런싱 개요

- **서비스(Service)** 개념: 하나의 물리적 데이터베이스에서 여러 개의 **서비스 이름**을 노출할 수 있습니다. 이는 워크로드 분리, 로드 밸런싱, Failover 정책 수립에 활용됩니다.
- 클라이언트는 **TNS** 별칭을 통해 서비스명에 접속하거나, **EZCONNECT**(`host:port/service_name`) 방식을 흔히 사용합니다.

```sql
-- 데이터베이스에서 제공 중인 서비스 확인 (리스너 동적 등록 기준)
SELECT name, network_name FROM v$services;
-- 현재 세션이 어떤 서비스로 접속했는지 확인
SELECT sid, serial#, username, service_name
FROM v$session
WHERE username IS NOT NULL;
```

---

## RAC, ASM, 메타 아키텍처 개요

- **RAC**(Real Application Clusters): 여러 인스턴스가 **하나의 공유 스토리지 상의 데이터베이스**를 **동시에 오픈**하여 사용하는 클러스터 구성입니다.
  - **GCS/GES**(Global Cache/Enqueue Service)를 통해 버퍼 캐시와 락의 일관성을 유지합니다.
  - 관련 백그라운드 프로세스: **LMON, LMD, LMS, LCK** 등이 추가됩니다.
- **ASM**(Automatic Storage Management): 오라클 전용의 볼륨 관리자 겸 파일 시스템입니다.
  - **디스크 그룹** 단위로 스토리지를 관리하며, 내부적으로 **스트라이핑과 미러링**을 제공합니다.
  - 관련 백그라운드 프로세스: **RBAL, ARBx** (리밸런싱 작업) 등이 추가됩니다.
- **Data Guard**: **REDO 전송 및 적용** 메커니즘을 기반으로 **물리적 또는 논리적 스탠바이 데이터베이스**를 구성하는 고가용성/재해복구 솔루션입니다. 아키텍처상 LGWR/ARCn 프로세스와 긴밀하게 통합되어 동작합니다.

---

## 모니터링과 진단 — 동시성, I/O, 리두, Undo, 메모리

### 시스템 핵심 지표 확인

```sql
-- 상위 시스템 통계 확인
SELECT * FROM (
  SELECT name, value
  FROM v$sysstat
  ORDER BY value DESC
) WHERE ROWNUM <= 30;

-- 상위 대기 이벤트 확인
SELECT * FROM (
  SELECT event, total_waits, time_waited_micro/1e6 AS seconds
  FROM v$system_event
  ORDER BY time_waited_micro DESC
) WHERE ROWNUM <= 20;
```

### 버퍼 캐시, 리두, 아카이브 로그 상태 관측

```sql
-- 버퍼 캐시 내 블록 상태 분포 확인
SELECT status, COUNT(*)
FROM v$bh
GROUP BY status;

-- 현재까지 생성된 리두 양 확인
SELECT VALUE AS redo_bytes
FROM v$sysstat
WHERE name = 'redo size';

-- 최근 아카이브 로그 생성 이력 확인
SELECT sequence#, first_time, next_time, blocks, archived, status
FROM v$archived_log
ORDER BY sequence# DESC FETCH FIRST 10 ROWS ONLY;
```

### PGA와 Temp 테이블스페이스 사용 현황

```sql
-- PGA 통계 요약 확인
SELECT name, value
FROM v$pgastat;

-- 현재 Temp 세그먼트를 사용 중인 세션 및 사용량 확인
SELECT s.sid, u.tablespace, u.blocks*ts.block_size/1024/1024 AS mb_used
FROM v$tempseg_usage u
JOIN v$session s ON s.saddr = u.session_addr
JOIN dba_tablespaces ts ON ts.tablespace_name = u.tablespace
ORDER BY mb_used DESC;
```

---

## 실습 시나리오 — “직접 만들어 보고 아키텍처 관측하기”

### 준비: 샘플 스키마 및 테이블 생성

```sql
-- 샘플 테이블과 인덱스 생성
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

-- 더미 데이터 주입 (간소화된 버전)
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

### DML/커밋과 리두 생성량 관찰

```sql
-- 현재 리두 사이즈 확인
SELECT name, value FROM v$sysstat WHERE name = 'redo size';

-- 대량 DML 수행
UPDATE sales_demo SET amount = amount * 1.05 WHERE sale_dt >= TRUNC(SYSDATE) - 30;
COMMIT;

-- 리두 사이즈 증가 확인
SELECT name, value FROM v$sysstat WHERE name = 'redo size';
```

**개념 점검**: `UPDATE` 문 수행 시 변경 내역이 **Redo Log Buffer**에 기록되고, **COMMIT** 시 **LGWR**가 이를 플러시합니다. 데이터파일에의 실제 반영(DBWn 작업)은 이후에 일어날 수 있습니다.

### 읽기 일관성과 Undo의 역할 확인

```sql
-- 긴 실행 시간이 예상되는 집계 쿼리 (세션 A에서 실행 가정)
SELECT /*+ FULL(s) PARALLEL(4) */ channel, SUM(amount)
FROM sales_demo s
GROUP BY channel;

-- 동시에 다른 세션(B)에서 특정 기간 데이터를 대규모로 UPDATE 또는 DELETE 수행
-- 충분한 undo_retention 시간과 Undo 테이블스페이스 공간이 확보되지 않으면 세션 A에서 ORA-01555 오류 발생 가능
```

**핵심 원리**: 세션 A는 쿼리가 시작된 시점의 **SCN**을 기준으로 데이터를 읽습니다. 따라서 세션 B가 데이터를 변경하더라도, 세션 A는 **Undo 데이터**를 활용해 **CR 블록**을 재구성하여 일관된 결과를 보게 됩니다. 필요한 Undo 정보가 보관되지 못하면 **snapshot too old** 오류가 발생합니다.

### 체크포인트 및 로그 스위치 관찰

```sql
-- 로그 스위치 강제 수행
ALTER SYSTEM SWITCH LOGFILE;

-- 현재 리두 로그 그룹 상태 및 시퀀스 확인
SELECT sequence#, first_change#, next_change#, archived, status
FROM v$log
ORDER BY sequence#;

-- 최근 아카이브 로그 정보 확인
SELECT sequence#, applied, first_time, next_time
FROM v$archived_log
ORDER BY sequence# DESC FETCH FIRST 5 ROWS ONLY;

-- DBWn 및 CKPT 활동은 주로 alert 로그, ASH, AWR 리포트를 통해 관측할 수 있습니다.
```

---

## 성능 및 용량 감각을 위한 개념적 수식

> **참고**: 아래 수식들은 기본적인 "감각 형성"을 위한 개념적 지표로 제공됩니다. 실전 튜닝은 반드시 **대기 이벤트 분석, 프로파일링, 실제 측정값**을 중심으로 진행해야 합니다.

- **Redo 생성률 추정**:
  $$ \text{redo\_rate (MB/s)} \approx \frac{\text{redo size (MB)}}{\text{interval (s)}} $$
  - LGWR 대기 이벤트 분석, 로그 파일 I/O 성능 평가, 아카이브 처리량 계획 수립 시 참고됩니다.
- **캐시 히트율 개념**:
  $$ \text{Hit Ratio} \approx 1 - \frac{\text{Physical Reads}}{\text{DB Block Gets + Consistent Gets}} $$
  - 단, 히트율이 높더라도 래치 경합, 핫블록, Redo/Undo 관련 대기 이벤트가 주요 병목일 수 있음을 유의해야 합니다.
- **PGA/Temp Spill 임계 판단 감각**:
  $$ \text{Spill Ratio} \approx \frac{\text{temp read/write blocks}}{\text{total sort/merge operations}} $$
  - Spill 비율이 지속적으로 높다면 `pga_aggregate_target` 증설, 워크로드 튜닝, 쿼리 재작성을 고려해볼 수 있습니다.

---

## 운영 관점의 핵심 설정과 모범 사례

1.  **ARCHIVELOG 모드와 FRA 구성**: 무정지 백업, 시점 복구(PITR), Data Guard 구축의 기본 토대를 마련합니다.
2.  **리두 로그 크기 및 개수 설계**: 너무 잦은 로그 스위치는 체크포인트를 빈번하게 발생시켜 성능에 부하를 줄 수 있습니다. 일반적으로 **20~30분 주기로 로그 스위치**가 발생하도록 크기를 조정하는 것이 흔한 목표입니다(실제 워크로드에 따라 조정 필요).
3.  **Undo 및 Temp 용량 사전 계획**: 장시간 실행되는 OLAP 쿼리 또는 대용량 배치 작업 시나리오를 고려하여 충분한 공간을 할당하고, `undo_retention`과 Temp 디스크 I/O 성능을 고려합니다.
4.  **SGA/PGA 목표치 명시적 설정**: AMM보다는 **ASMM(SGA_TARGET) + PGA_AGGREGATE_TARGET** 조합을 사용하면 메모리 사용이 더 예측 가능해집니다.
5.  **통계 및 히스토그램 주기적 관리(DBMS_STATS)**: 옵티마이저가 최적의 실행 계획을 수립할 수 있도록 정확한 카디널리티 정보를 제공하는 기반을 유지합니다.
6.  **바인드 변수 적극 사용**: 하드 파싱(Hard Parsing) 빈도와 라이브러리 캐시 관련 경합을 현저히 감소시킵니다.
7.  **AWR/ASH 리포트를 활용한 과학적 관측**: 상위 SQL, 대기 이벤트, 세션 활동 프로파일을 분석하여 성능 병목을 체계적으로 식별하고 해결합니다.
8.  **물리적 파일의 다중화 구현**: 데이터 파일, 리두 로그, 컨트롤 파일을 스토리지 또는 ASM 정책과 연계하여 다중화하면 가용성이 크게 향상됩니다.

---

## 기본 아키텍처 점검 포인트

1.  `ARCHIVELOG` 모드 운영 여부와 아카이브 대상(FRA 등)의 용량 상태를 정기적으로 점검하세요.
2.  온라인 리두 로그의 **그룹 수, 개별 크기, 배치 위치의 I/O 성능**이 현재 워크로드에 적합한지 검토하세요.
3.  `undo_tablespace`의 크기, `undo_retention` 설정 값, 그리고 실제 Undo 공간 사용 추이를 모니터링하세요.
4.  TEMP 테이블스페이스의 총 용량과 `v$tempseg_usage` 뷰를 통한 실시간 사용 현황을 감시하세요.
5.  `sga_target`, `pga_aggregate_target` 설정값과 `v$pgastat`, `v$sgastat`의 실제 사용량을 비교 분석하세요.
6.  시스템 전반의 병목을 식별하기 위해 `v$system_event` 또는 AWR 리포트의 Top Events를 상시 확인하세요.
7.  성능에 가장 큰 영향을 미치는 쿼리를 파악하기 위해 AWR Top SQL 또는 SQL Monitor 리포트를 활용하세요.
8.  데이터 파일, 컨트롤 파일, 리두 로그의 다중화 구성과 함께 RMAN을 이용한 백업/복구 전략이 수립되어 있는지 확인하세요.
9.  리스너 설정(`listener.ora`), 데이터베이스 서비스 구성, 그리고 전용/공유 서버 모드 정책을 검토하세요.
10. 래치/뮤텍스 대기, 버퍼 버시(Busy) 대기, 핫블록 경합 등의 지표와 파싱 관련 통계를 주기적으로 점검하세요.

---

## 부록 — 자주 참조하는 시스템 뷰 (빠른 참조)

```sql
-- 인스턴스 및 데이터베이스 기본 정보
SELECT * FROM v$instance;
SELECT * FROM v$database;

-- 물리적 파일 정보
SELECT * FROM v$datafile;
SELECT * FROM v$controlfile;
SELECT * FROM v$log;
SELECT * FROM v$logfile;

-- 메모리 영역 정보
SELECT * FROM v$sga;
SELECT * FROM v$sgainfo;
SELECT * FROM v$sgastat;
SELECT * FROM v$pgastat;

-- 세션, 프로세스, 대기 이벤트 정보
SELECT sid, serial#, username, status, machine, program
FROM v$session
WHERE username IS NOT NULL;

SELECT * FROM v$process;

SELECT sid, event, wait_class, state, seconds_in_wait
FROM v$session
WHERE state <> 'WAITED SHORT TIME';

-- 옵티마이저 통계 정보 (예시)
SELECT * FROM dba_tab_statistics WHERE table_name='SALES_DEMO';
```

---

## 결론

오라클 데이터베이스의 기본 아키텍처는 **"인스턴스(메모리와 프로세스의 집합)"** 가 **"데이터베이스(물리적 파일의 집합)"** 를 관리하고 서비스하는 명확한 분리 구조에 기반합니다. 이 구조 안에서 **Redo, Undo, SCN**이라는 세 가지 핵심 메커니즘이 서로 협력하여 트랜잭션의 **원자성(Atomicity), 일관성(Consistency), 내구성(Durability)**을 실현합니다. **LGWR**(커밋 성능과 내구성 담당), **DBWn**(효율적인 쓰기 담당), **CKPT**(복구 지점 관리), **SMON/PMON**(자동 복구 및 정리)와 같은 백그라운드 프로세스들이 이 메커니즘을 구동하는 엔진 역할을 합니다.

실무에서 안정적이고 성능良好的인 데이터베이스를 운영하려면, SGA/PGA, Undo, Temp, Redo/Archive 저장 공간에 대한 충분한 용량 계획과 함께, 단순한 지표보다는 **상위 대기 이벤트에 기반한 성능 병목 분석**에 집중하는 접근이 필수적입니다. 아키텍처에 대한 깊은 이해는 복잡한 문제를 해결하고, 시스템을 체계적으로 튜닝하며, 장애에 효과적으로 대응하는 데 반드시 필요한 기초입니다. 본 문서가 오라클 데이터베이스의 핵심 구조와 동작 원리를 실무 관점에서 정리하는 데 유용한 길잡이가 되기를 바랍니다.