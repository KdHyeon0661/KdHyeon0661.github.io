---
layout: post
title: DB 심화 - SQL Trace
date: 2025-10-24 15:25:23 +0900
category: DB 심화
---
# Oracle **SQL Trace**

## 0) SQL Trace 한눈 요약

- **무엇?** Oracle에서 커서의 **PARSE/EXEC/FETCH 호출**, **대기 이벤트**(waits), **바인드 값**(binds) 등을 **세부 로그**로 파일에 남기는 기능. (이벤트 번호 **10046**)
- **왜?** “**어디서 얼마나** 시간이 쓰였는지”를 **근거**로 파악하기 위해. AUTOTRACE/AWR보다 **호출 단위**로 상세.
- **어떻게?**  
  1) **자기 세션**: `DBMS_MONITOR.SESSION_TRACE_ENABLE(waits=>TRUE, binds=>TRUE)`  
  2) **다른 세션**: 같은 프로시저에 **(sid, serial#)** 전달 또는 `DBMS_SYSTEM.SET_SQL_TRACE_IN_SESSION`  
  3) **Service/Module/Action**: `DBMS_MONITOR.SERV_MOD_ACT_TRACE_ENABLE` 로 **그룹 단위** 일괄 추적
- **어디에 남나?** `V$DIAG_INFO` → **Default Trace File** / **Diag Trace 디렉토리**
- **어떻게 읽나?** **TKPROF** 로 포맷팅 → SQL별 **Parse/Exec/Fetch 시간/대기** 요약

---

## 1) 준비 — 트레이스 파일 위치/이름 파악

```sql
-- 현재 세션의 기본 트레이스 파일 전체 경로
SELECT value AS trace_file
FROM   v$diag_info
WHERE  name = 'Default Trace File';

-- 트레이스 디렉터리(ADR)
SELECT value AS diag_trace_dir
FROM   v$diag_info
WHERE  name = 'Diag Trace';

-- 파일 이름에 식별자 덧붙이기(가독성↑)
ALTER SESSION SET tracefile_identifier = 'PAYROLL_JUN_RUN';
```

> 팁  
> - `tracefile_identifier` 를 세팅하면 파일명이 `…_PAYROLL_JUN_RUN.trc`처럼 생성되어 **찾기 쉬움**  
> - OS에서 `ls -ltr`(최신순)로 빠르게 찾는다.

---

## 2) 자기 세션에 트레이스 걸기 (가장 안전·기본)

### 2.1 `DBMS_MONITOR` 권장 방식

```sql
-- (1) 바인드/대기 포함하여 시작
EXEC DBMS_MONITOR.SESSION_TRACE_ENABLE(waits => TRUE, binds => TRUE, plan_stat => 'ALL_EXECUTIONS');
-- plan_stat 옵션: 'FIRST_EXECUTION', 'ALL_EXECUTIONS', NULL

-- 문제 SQL 실행...
SELECT /* target */ COUNT(*) 
FROM   orders o JOIN customers c ON c.id=o.customer_id
WHERE  c.region = :r AND o.order_date BETWEEN :d1 AND :d2;

-- (2) 중지
EXEC DBMS_MONITOR.SESSION_TRACE_DISABLE;
```

**설명 & 베스트 프랙티스**
- `waits=>TRUE` : `db file sequential read` 같은 **대기 이벤트**를 기록  
- `binds=>TRUE` : 바인드 값 기록(**PII/보안 주의**)  
- `plan_stat` : 실행 중 **Plan statistics**(라인별 A-rows/Starts/Temp 등)도 함께 캡처(버전에 따라 10046 + 10132/SQLTraceExt 항목 반영)
- 트레이스는 **필요 구간**만 켜서 **오버헤드 최소화**(보통 5~15%대, 상황·옵션에 따라 더 큼)

### 2.2 `ALTER SESSION SET EVENTS '10046'` 고전 방식

```sql
-- Level: 12 = waits(8)+binds(4), 8=waits만, 4=binds만, 1=기본
ALTER SESSION SET EVENTS '10046 trace name context forever, level 12';

-- SQL 실행 구간...

ALTER SESSION SET EVENTS '10046 trace name context off';
```

**언제 쓰나?** 스크립트/도구 제약으로 `DBMS_MONITOR` 호출이 어려울 때 **대체**로 사용.

---

## 3) 다른 세션에 트레이스 걸기 (운영 이슈 재현·현장점검)

> 전제: **권한** 필요(보통 DBA). 대상 세션의 **SID, SERIAL#** 확보가 우선.

### 3.1 대상 세션 식별

```sql
-- 최근 10분간 활동/대기 많은 세션 파악(예시)
SELECT s.sid, s.serial#, s.username, s.module, s.action, s.program,
       s.machine, s.sql_id, s.event, s.state, s.seconds_in_wait
FROM   v$session s
WHERE  s.type = 'USER'
ORDER  BY s.seconds_in_wait DESC;

-- 특정 사용자/모듈로 좁히기
SELECT sid, serial#, module, action, sql_id
FROM   v$session
WHERE  username = 'APPUSER'
  AND  module   = 'PAYROLL-BATCH';
```

### 3.2 `DBMS_MONITOR.SESSION_TRACE_ENABLE` (권장)

```sql
-- SYS/DBA 권한 세션에서 실행
EXEC DBMS_MONITOR.SESSION_TRACE_ENABLE(
  session_id => :sid,     -- 대상 세션 SID
  serial_num => :serial#, -- 대상 세션 SERIAL#
  waits      => TRUE,
  binds      => TRUE,
  plan_stat  => 'ALL_EXECUTIONS'  -- 필요시
);

-- 문제 재현/지속 관찰...

EXEC DBMS_MONITOR.SESSION_TRACE_DISABLE(:sid, :serial#);
```

**장점**  
- **해당 세션만** 정밀 추적, **서비스 전체**에 비해 **오버헤드/로그량 최소화**

### 3.3 `DBMS_SYSTEM.SET_SQL_TRACE_IN_SESSION` (레거시)

```sql
-- 켜기
EXEC DBMS_SYSTEM.SET_SQL_TRACE_IN_SESSION(:sid, :serial#, TRUE);
-- 끄기
EXEC DBMS_SYSTEM.SET_SQL_TRACE_IN_SESSION(:sid, :serial#, FALSE);
```

**언제?** 구버전/호환성 요구. 단, **waits/binds 옵션 직접 제공 불가** → 세밀 제어는 `DBMS_MONITOR` 사용 권장.

> 운영 팁  
> - **지속 시간** 제한(타이머) 또는 **문제 구간만** ON/OFF 습관화  
> - 대상 세션이 **끊기면** 트레이스도 함께 종료(파일은 남음)  
> - **TKPROF**로 **즉시 가공**해 공유 가능한 리포트를 만든다

---

## 4) Service / Module / Action 단위로 트레이스 걸기 (대상 그룹 지정)

> 대규모 애플리케이션에서 “**특정 서비스나 화면/배치**”에 속한 **모든 세션**을 자동 추적하는 방법.  
> **오버헤드/로그 폭증** 가능 — **범위를 좁혀** 사용!

### 4.1 애플리케이션에서 Module/Action 세팅

```sql
-- (PL/SQL/SQL*Plus 예시) 현재 세션의 모듈/액션 지정
BEGIN
  DBMS_APPLICATION_INFO.SET_MODULE(module_name => 'PAYROLL-BATCH',
                                   action_name => 'CLOSE_JUNE');
END;
/
-- Java/JDBC는 setClientInfo / OCI은 OCIAttrSet 등으로 설정 가능
```

> **Service** 는 보통 **Listener/DB Service 설정** 또는 `ALTER SYSTEM/DBMS_SERVICE` 로 운용.

### 4.2 `DBMS_MONITOR.SERV_MOD_ACT_TRACE_ENABLE`

```sql
-- 1) 특정 Service 전체
EXEC DBMS_MONITOR.SERV_MOD_ACT_TRACE_ENABLE(
  service_name => 'PAYROLL_SVC',
  waits        => TRUE,
  binds        => TRUE
);

-- 2) 특정 Service + Module
EXEC DBMS_MONITOR.SERV_MOD_ACT_TRACE_ENABLE(
  service_name => 'PAYROLL_SVC',
  module_name  => 'PAYROLL-BATCH',
  waits        => TRUE,
  binds        => TRUE
);

-- 3) Service + Module + Action (가장 좁은 타겟)
EXEC DBMS_MONITOR.SERV_MOD_ACT_TRACE_ENABLE(
  service_name => 'PAYROLL_SVC',
  module_name  => 'PAYROLL-BATCH',
  action_name  => 'CLOSE_JUNE',
  waits        => TRUE,
  binds        => TRUE
);
```

- **효과**: **해당 서비스/모듈/액션으로 접속하거나 변경되는 모든 세션**에 자동으로 트레이스 적용  
- **인스턴스 한정**이 필요하면 `instance_name`(RAC 환경) 지정

```sql
-- 해제(완전히 끄기)
EXEC DBMS_MONITOR.SERV_MOD_ACT_TRACE_DISABLE(
  service_name => 'PAYROLL_SVC',
  module_name  => 'PAYROLL-BATCH',
  action_name  => 'CLOSE_JUNE'
);
```

### 4.3 Client Identifier 기반(참고)

```sql
-- 클라이언트 아이디(예: 고객/테넌트) 단위
BEGIN
  DBMS_SESSION.SET_IDENTIFIER('TENANT#ACME'); -- 애플리케이션에서 설정
END;
/

EXEC DBMS_MONITOR.CLIENT_ID_TRACE_ENABLE(client_id => 'TENANT#ACME', waits => TRUE, binds => FALSE);
EXEC DBMS_MONITOR.CLIENT_ID_TRACE_DISABLE(client_id => 'TENANT#ACME');
```

---

## 5) 실전 시나리오

### 5.1 배치 작업 “6월 마감” 느림 — 모듈/액션 트레이스
**상황**: `PAYROLL-BATCH / CLOSE_JUNE` 단계가 느려짐 (다수 세션/워커 사용)

1) **애플리케이션**에서 해당 단계 진입 시 `SET_MODULE/SET_ACTION` 호출 보장  
2) DBA가 **모듈/액션 트레이스** ON
```sql
EXEC DBMS_MONITOR.SERV_MOD_ACT_TRACE_ENABLE(
  service_name => 'PAYROLL_SVC',
  module_name  => 'PAYROLL-BATCH',
  action_name  => 'CLOSE_JUNE',
  waits => TRUE, binds => TRUE
);
```
3) 작업 수행 → **다수 트레이스 파일** 생성(세션별)  
4) 종료 후 **OFF**  
5) 파일을 **TKPROF** 로 묶어 주요 SQL/대기 요약

### 5.2 특정 사용자 화면 느림 — Service 단위(짧게) 트레이스
**상황**: 사용자 불특정, 특정 서비스(웹) 레이어 전체에서 간헐적 느림

1) **짧은 시간 창**으로 service trace ON  
2) N분 뒤 OFF, 생성된 트레이스에서 **Top SQL/이벤트** 추출  
3) 문제 SQL을 타겟으로 **세션 단위/자기 세션 재현**으로 심화 분석

---

## 6) TKPROF로 읽기 (요약 가이드)

```bash
# 서버(또는 파일 접근 가능 위치)에서
tkprof input.trc output.prf sys=no sort=exeela,fchela

# 옵션
# sys=no  : SYS 호출 제외
# waits=yes : 대기 이벤트 표시(일부 버전 기본)
# sort=... : 실행시간/페치시간/파스시간 등 정렬
```

**해석 포인트**
- SQL별 **Parse/Execute/Fetch** 각각의 **Elapsed/CPU/Calls/Rows**  
- **대기 이벤트**별 총합 및 비중  
- **바인드 값**(bind capture on)  
- “어떤 SQL이 가장 느리고, 그 중 **Exec vs Fetch** 어디에 시간이 드는지” → **락인지 I/O인지**와 연결

> 보다 세밀한 **라인별 통계/Plan**은 실행 후 `DBMS_XPLAN.DISPLAY_CURSOR('ALLSTATS LAST +PEEKED_BINDS')` 혹은 **SQL Monitor Report** 병행

---

## 7) 운영 시 주의/정책

1) **오버헤드**: waits/binds/plan_stat ON은 로그량·CPU 증가 → **범위/기간 최소화**  
2) **보안/개인정보**: `binds=>TRUE` 는 **민감 값**이 남는다 → **마스킹/보관 정책**  
3) **용량 관리**: 대량 트레이스는 ADR/파일시스템 **가득참** 위험 → **롤링 수집/압축**  
4) **끝나면 반드시 끄기**: 서비스/모듈 단위는 **영향 범위 큼**  
5) **식별자 부여**: `tracefile_identifier` 로 **파일 분류**를 습관화  
6) **시간 상관**: 문제가 생긴 **시각대와 매칭**하여 수집

---

## 8) API·옵션 치트시트

### 8.1 켜기/끄기

| 목적 | 권장 API | 예 |
|---|---|---|
| 자기 세션 | `DBMS_MONITOR.SESSION_TRACE_ENABLE` | `EXEC DBMS_MONITOR.SESSION_TRACE_ENABLE(TRUE,TRUE,'ALL_EXECUTIONS');` |
| 다른 세션 | `DBMS_MONITOR.SESSION_TRACE_ENABLE(sid,serial#)` | `EXEC DBMS_MONITOR.SESSION_TRACE_ENABLE(123,456,TRUE,TRUE);` |
| 레거시 | `DBMS_SYSTEM.SET_SQL_TRACE_IN_SESSION` | `EXEC DBMS_SYSTEM.SET_SQL_TRACE_IN_SESSION(123,456,TRUE);` |
| 이벤트 | `ALTER SESSION SET EVENTS '10046 …'` | `… level 12` / `… off` |
| 서비스/모듈/액션 | `DBMS_MONITOR.SERV_MOD_ACT_TRACE_ENABLE` | `EXEC …(service,module,action,TRUE,TRUE);` |
| 클라이언트 ID | `DBMS_MONITOR.CLIENT_ID_TRACE_ENABLE` | `EXEC …('TENANT#ACME',TRUE,FALSE);` |

### 8.2 10046 레벨

| Level | 의미 |
|---|---|
| 1 | 기본(SQL Trace) |
| 4 | **Binds** 캡처 |
| 8 | **Waits** 캡처 |
| 12 | **Waits + Binds** (가장 많이 사용) |

---

## 9) 문제 해결 루틴(현장 템플릿)

1) **증상/대상 식별**: 서비스? 모듈? 특정 사용자/세션?  
2) **범위 결정**: 세션 단위가 가능한지(최소 범위 우선)  
3) **Trace ON**: 적절한 API로 waits/binds/plan_stat 선택  
4) **재현/관측**: 충분히 데이터 수집(짧고 확실하게)  
5) **Trace OFF**: 범위 대량 수집 방지  
6) **TKPROF**: Top SQL, Parse/Exec/Fetch, 대기 이벤트 파악  
7) **심화**: 해당 SQL `DISPLAY_CURSOR(ALLSTATS LAST +PEEKED_BINDS)` / SQL Monitor  
8) **조치**: 인덱스/조인/통계/락/커밋/네트워크/파라미터…  
9) **재검증**: 동일 시나리오 재측정

---

## 10) 미니 예제 세트(끝까지 따라하기)

### 10.1 자기 세션 — 바인드·대기 포함 + 식별자

```sql
ALTER SESSION SET tracefile_identifier = 'HOT_REPORT';
EXEC DBMS_MONITOR.SESSION_TRACE_ENABLE(waits=>TRUE, binds=>TRUE, plan_stat=>'FIRST_EXECUTION');

VAR r VARCHAR2(10); EXEC :r := 'APAC';
VAR d1 DATE;        EXEC :d1 := DATE '2024-06-01';
VAR d2 DATE;        EXEC :d2 := DATE '2024-06-30';

SELECT /* HOT_REPORT */ c.region, SUM(o.amount)
FROM   orders o JOIN customers c ON c.id=o.customer_id
WHERE  c.region=:r AND o.order_date BETWEEN :d1 AND :d2
GROUP  BY c.region;

EXEC DBMS_MONITOR.SESSION_TRACE_DISABLE;

-- 파일 경로 확인
SELECT value FROM v$diag_info WHERE name='Default Trace File';
```

### 10.2 다른 세션 — SID/SERIAL#로

```sql
-- 대상 찾기
SELECT sid, serial#, username, module, action, sql_id
FROM   v$session
WHERE  username='APPUSER' AND module='PAYROLL-BATCH';

-- 켜기
EXEC DBMS_MONITOR.SESSION_TRACE_ENABLE( :sid, :serial#, waits=>TRUE, binds=>FALSE );

-- 잠시 후 끄기
EXEC DBMS_MONITOR.SESSION_TRACE_DISABLE( :sid, :serial# );
```

### 10.3 서비스/모듈/액션 — 배치 구간만

```sql
-- 앱이 진입 시 다음 호출 보장
BEGIN
  DBMS_APPLICATION_INFO.SET_MODULE('PAYROLL-BATCH','CLOSE_JUNE');
END;
/

-- DBA: 좁은 타겟으로 ON
EXEC DBMS_MONITOR.SERV_MOD_ACT_TRACE_ENABLE(
  service_name=>'PAYROLL_SVC',
  module_name =>'PAYROLL-BATCH',
  action_name =>'CLOSE_JUNE',
  waits=>TRUE, binds=>TRUE
);

-- … 배치 수행 …

EXEC DBMS_MONITOR.SERV_MOD_ACT_TRACE_DISABLE(
  service_name=>'PAYROLL_SVC',
  module_name =>'PAYROLL-BATCH',
  action_name =>'CLOSE_JUNE'
);
```

---

## 11) 자주 묻는 질문(FAQ)

**Q1. 트레이스 파일이 너무 많아요.**  
A. **범위를 줄이거나 기간을 짧게**, `tracefile_identifier` 로 켜고 끄는 구간을 명확히. 서비스 단위는 **최후 수단**.

**Q2. 바인드 값은 꼭 필요할까요?**  
A. **플랜 오차/카디널리티/ACS 문제**를 파악하려면 유용하나, **민감정보**라면 **가급적 끄고**(binds=>FALSE) 다른 지표로 분석.

**Q3. AUTOTRACE/EXPLAIN과 뭐가 달라요?**  
A. SQL Trace는 **호출·대기 단위의 원시 로그**. AUTOTRACE는 **세션 통계 요약/예상 계획**, `DISPLAY_CURSOR`는 **실제 플랜 라인 통계**.

**Q4. RAC에서 특정 인스턴스만?**  
A. `SERV_MOD_ACT_TRACE_ENABLE` 의 `instance_name` 활용. 또는 **그 인스턴스로 라우팅된 서비스**에서 ON.

---

## 12) 결론

- **세션 단위**가 기본, **서비스/모듈/액션**은 “정말 필요할 때만” **좁게**.  
- **waits/binds/plan_stat** 조합으로 **원인(락/I/O/네트워크/파싱/플랜)** 을 **증거**로 잡아라.  
- **Trace → TKPROF → (필요시) DISPLAY_CURSOR/SQL Monitor** 3단 루틴이 **실패 없는 정석**이다.

> 한 줄 정리  
> **늘 최소 범위/최소 시간으로**, **증거를 남기고**(Trace), **요약을 읽고**(TKPROF), **라인을 고쳐라**(Plan 튜닝).
