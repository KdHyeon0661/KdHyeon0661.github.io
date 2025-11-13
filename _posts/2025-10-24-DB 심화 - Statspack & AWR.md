---
layout: post
title: DB 심화 - Statspack & AWR
date: 2025-10-24 20:25:23 +0900
category: DB 심화
---
# Oracle **Statspack & AWR**

## 0. 큰 그림: Statspack vs AWR

| 항목 | Statspack | AWR (Automatic Workload Repository) |
|---|---|---|
| 설치/라이선스 | **무료**(스크립트로 설치) | EE + **Diagnostics Pack** 라이선스 필요 |
| 수집 | 수동/잡 스냅샷 | 자동 스냅샷(기본 1시간/7일 보관), API 제어 |
| 리포트 | `spreport.sql` (스냅샷 범위 지정) | `awrrpt.sql`(인스턴스), `awrrpti.sql`(다중 인스턴스/RAC), `awrsqrpt.sql`(특정 SQL), `awrdiff.sql`(비교) 등 |
| 데이터 범위 | 시스템 전반(Top5 events, SQL top 등) | +**Time Model**, +**ASH** 연동(보조), +세부 통계 다양 |
| 비교/베이스라인 | 수동 | **베이스라인 생성/고정/관리** API |
| 권장 | **SE/비DIAG** 환경, 과거 시스템 | **표준**(가능하면 AWR 권장) |

> 본 문서는 **둘 다** 다룬다. **기법/해석은 동일**하며, AWR이 **더 풍부**하다.

---

## 1. Statspack: 설치 → 스냅샷 → 리포트

### 1.1 설치(최초 1회)
```sql
-- SYS로 접속
@?/rdbms/admin/spcreate.sql
-- PERFSTAT 스키마/오브젝트 생성
```
> 제거 시 `spdrop.sql`. 설치 중 **테이블스페이스**/패스워드 지정 단계가 있다.

### 1.2 스냅샷(수동/잡)
```sql
-- 즉시 스냅샷
EXEC perfstat.statspack.snap;

-- 잡으로 주기 수집(예: 30분)
VARIABLE jobno NUMBER
BEGIN
  DBMS_JOB.SUBMIT(:jobno, 'perfstat.statspack.snap;', SYSDATE, 'SYSDATE + 30/1440');
  COMMIT;
END;
/
PRINT jobno;
```

### 1.3 스냅샷 목록 확인 → 리포트 생성
```sql
-- 스냅샷 목록
SELECT snap_id, begin_interval_time, end_interval_time
FROM   perfstat.stats$snapshot
ORDER  BY snap_id DESC;

-- 리포트
@?/rdbms/admin/spreport.sql
-- 프롬프트: DBID/Instance/Start snap/End snap/출력 파일명
```

---

## 2. AWR: 설정 → 리포트 → 비교/SQL 전용/다중 인스턴스

### 2.1 스냅샷 주기/보관 기간 조정
```sql
-- 현 설정
SELECT snap_interval, retention FROM dba_hist_wr_control;

-- 30분마다, 14일 보관
BEGIN
  DBMS_WORKLOAD_REPOSITORY.MODIFY_SNAPSHOT_SETTINGS(
    interval  => 30,     -- 분
    retention => 14*24*60 -- 분
  );
END;
/
```

### 2.2 수동 스냅샷
```sql
EXEC DBMS_WORKLOAD_REPOSITORY.CREATE_SNAPSHOT();
```

### 2.3 표준 리포트
```sql
-- 단일 인스턴스
@?/rdbms/admin/awrrpt.sql
-- RAC/다중 인스턴스(시간대 동일 구간 병합)
@?/rdbms/admin/awrrpti.sql

-- 특정 SQL 레포트(해당 구간)
@?/rdbms/admin/awrsqrpt.sql

-- 두 구간 비교(전/후)
@?/rdbms/admin/awrdiff.sql
-- RAC 비교
@?/rdbms/admin/awrddrpt.sql
```
> 리포트는 **텍스트/HTML** 출력 가능. HTML이 가독성↑.

---

## 3. 리포트 읽는 순서(탑다운 루틴)

> **핵심**: “**느리다**”를 **시간**으로 해체 → **CPU vs WAIT** → **어떤 WAIT?** → **누가 발생? (SQL/세그먼트/파일/라인)**

1) **Report Summary / Time Period** 확인
   - 구간 길이, 스냅샷 시각, 인스턴스, RAC 여부.
2) **Load Profile** / **DB Time / DB CPU**
   - 초당 logical/physical reads, Executes, Parses, Redo, Commits 등.
   - **DB Time**과 **DB CPU**의 관계로 **대기 비중** 확인.
3) **Top Timed Events**(Statspack) / **Top Events** + **Time Model**(AWR)
   - Wait Class 구도(User I/O/Concurrency/Commit/Cluster/Library/Network/CPU).
4) **SQL ordered by Elapsed/CPU/Reads/Execs/Cluster/Version count**
   - 상위 **핫 SQL** 식별(최대 기여 순).
5) **Instance Activity / System Statistics**
   - session logical reads, physical reads/writes, sorts(memory/disk) 등 **볼륨** 체크.
6) **I/O Statistics / File I/O / Temp**
   - 파일/테이블스페이스 별 **read/write time**, Temp 사용.
7) **Segment Statistics**(Top logical/physical gets, buffer busy, itl waits)
   - **핫 세그먼트/인덱스**(핫블록/경합/분포).
8) **Advisory/Memory**(AWR)
   - PGA/Buffer Cache/Shared Pool 어드바이저로 **메모리 방향성** 확인.
9) **RAC**: Global Cache(Interconnect) 대기/ 인스턴스별 부하 분포.
10) **상관 확인**: 필요한 SQL을 **DBMS_XPLAN.DISPLAY_CURSOR('ALLSTATS LAST')** 또는 **SQL Monitor**로 **Plan 라인**까지 내려가 최종 결론.

---

## 4. 예제 시나리오 & 리포트 읽기

> 아래 예제는 “월말 보고서가 **느려진 피크 30분**” 구간을 AWR로 분석한다는 가정. (Statspack도 거의 동일)

### 4.1 구간 선택(스냅샷 범위)
```sql
-- 후보 스냅샷 범위 조회
SELECT snap_id, begin_interval_time, end_interval_time
FROM   dba_hist_snapshot
WHERE  begin_interval_time BETWEEN
       TO_TIMESTAMP('2025-10-31 10:00:00','YYYY-MM-DD HH24:MI:SS')
   AND TO_TIMESTAMP('2025-10-31 12:00:00','YYYY-MM-DD HH24:MI:SS')
ORDER  BY snap_id;
```
> 10:30~11:00 구간이 피크로 가정하여 **Start Snap=1201, End Snap=1202** 선택.

### 4.2 리포트 생성
```sql
@?/rdbms/admin/awrrpt.sql
-- DBID/Inst 선택 → 1201 ~ 1202 → HTML
```

### 4.3 핵심 섹션 해석(요약 샘플)

#### (1) Load Profile (요지)
```
Per Second
  DB Time(s):            120.3
  DB CPU(s):              52.9
  Redo size(bytes):     12,3 MB
  Logical reads:        85,000
  Block changes:         5,200
  Executes:              1,120
  Parses:                  220
  Hard parses:              12
  ...
```
- **DB Time > DB CPU** 큰 격차 → **대기 비중 높음**
- Executes/sec ↑ , Parses/sec 적당, Hard parses 낮음(좋음)

#### (2) Top Timed Events / Time Model
```
Top Timed Events
  Event                            %DB time   Wait Class
  direct path read temp              38.1     User I/O
  db file sequential read            21.4     User I/O
  log file sync                      10.7     Commit
  ...
Time Model Statistics
  DB time                          216,540 s
  DB CPU                            95,240 s
  sql execute elapsed time         200,xxx s
  parse time elapsed                 1,4xx s
```
- **Temp 읽기(정렬/해시 스필)** 1위, 랜덤 I/O 2위, 커밋 지연 3위 → **정렬/해시 과다 + 랜덤 I/O 많음 + 커밋 빈도/지연** 혼합.

#### (3) SQL ordered by Elapsed Time
```
SQL ID       Elapsed(s)  Executions  Elap/Exec  CPU(s)  Buffer Gets  Disk Reads
9pqs1m..         42,300        25      1,692     5,900    1.2e9       58,000,000
3a7nr8..         21,900      1,200        18     7,300    0.4e9       22,000,000
...
```
- **9pqs1m..** 상위 1 SQL이 **총 응답시간의 20%** 기여. 선순위 타겟.

#### (4) Segment Statistics (logical/physical/ITL/buffer busy 상위)
```
Top Segments by Logical Reads
  OWNER.TABLE / INDEX ...   Logical Reads  %Total
  APP.ORDERS_IDX1           2.9e8          25%
  APP.ORDERS                1.8e8          15%
...
Top Segments by Physical Reads
  TEMP (TS#12)              5.2e7
  APP.ORDERS                3.1e7
...
```
- Temp 스필 많음(예상 일치). `ORDERS_IDX1`/`ORDERS` 핫.

#### (5) I/O Stats / Temp
```
IOStat by Filetype Summary
  Temp: Read 55,000,000 blocks (avg read time 8.2 ms)
  Data: Read 31,000,000 blocks ...
```
- Temp I/O 지배적 → **PGA/정렬/해시/카디널리티** 점검 필요.

### 4.4 타겟 SQL 라인 통계(Plan 연결)
```sql
-- SQL ID 9pqs1m.. 의 실제 라인 통계(마지막/최근)
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR('9pqs1m..', NULL,
  'ALLSTATS LAST +PREDICATE +PROJECTION +PEEKED_BINDS +OUTLINE +NOTE'));
```
**관찰 포인트**
- `SORT GROUP BY` / `HASH JOIN` 라인의 **TempSpc** 수백 MB~수 GB, **Time**↑
- 상위 입력 라인에서 `A-Rows` 과다(카디널리티 오판)
- `TABLE ACCESS FULL ORDERS` 또는 `INDEX RANGE SCAN + BY ROWID` 후 **대량 유입**

**가설 → 조치**
1) **카디널리티 정확화**: 히스토그램 생성(스큐 컬럼), 바인드 값/ACS 확인(+PEEKED_BINDS)
2) **정렬 회피**: 커버링 인덱스(필요 컬럼 포함) 설계로 `ORDER BY/GROUP BY` 비용 절감
3) **PGA↑/workarea policy** 검토(스필 줄이기)
4) 조인전략 전환(해시→NL, 소량 먼저), 파티션 프루닝 강화

**Before/After 확인**: 동일 구간 AWR 또는 `awrdiff.sql`로 비교(§6).

---

## 5. Statspack 리포트 분석(동일 루틴 적용)

`spreport.sql` 결과의 핵심 섹션도 **Top 5 Timed Events**, **Load Profile**, **SQL ordered by…**, **Instance Activity** 등 **유사**하다.

- **Top 5 Timed Events**에서 `db file scattered read`↑ → FTS/범위 과다 → **Plan에서 FTS 라인** 확인
- **SQL ordered by Gets/Reads** 상위 SQL의 **실행계획**을 `DBMS_XPLAN.DISPLAY_CURSOR`로 검증(Statspack만 쓴다 해도 SQL은 캐시에 존재)
- **Buffer busy/ITL waits** 세그먼트가 보이면 인덱스 편향/INITRANS/키분산 대책 검토

> 차이: Statspack은 **Time Model/Advisory**가 제한적. **해석 원칙**은 동일.

---

## 6. **비교 리포트**로 “튜닝 효과/회귀” 증명

### 6.1 AWR Diff Report
```sql
@?/rdbms/admin/awrdiff.sql
-- DBID/Inst → Before(Start1,End1) vs After(Start2,End2) 선택
```
**읽는 법**
- **DB Time/DB CPU/Top Events**의 Before/After **증감**
- **SQL Diff** 섹션에서 **새로 튄 SQL**/개선된 SQL 식별
- **I/O/Temp/Redo** 볼륨 비교로 작업성격 변화 확인

### 6.2 RAC 다중 인스턴스 비교
```sql
@?/rdbms/admin/awrddrpt.sql
```
- 인스턴스 간 **부하 불균형/인터커넥트 대기** 비교

---

## 7. **베이스라인**(AWR)으로 “좋을 때 상태” 고정

### 7.1 베이스라인 생성/관리
```sql
-- 스냅샷 범위 베이스라인
DECLARE
  l_baseline_name VARCHAR2(30) := 'EOM_GOOD_2025OCT';
BEGIN
  DBMS_WORKLOAD_REPOSITORY.CREATE_BASELINE(
    start_snap_id => 1101,
    end_snap_id   => 1103,
    baseline_name => l_baseline_name,
    expiration    => NULL -- 영구
  );
END;
/
-- 조회/삭제
SELECT baseline_id, baseline_name, start_snap_id, end_snap_id
FROM   dba_hist_baseline;
BEGIN
  DBMS_WORKLOAD_REPOSITORY.DROP_BASELINE('EOM_GOOD_2025OCT', TRUE);
END;
/
```

### 7.2 이동 윈도 베이스라인(피크 시간 자동 고정)
```sql
BEGIN
  DBMS_WORKLOAD_REPOSITORY.CREATE_BASELINE_TEMPLATE(
    baseline_name          => 'EOM_10AM_12PM',
    template_type          => 'MOVING_WINDOW',
    start_time             => TO_TIMESTAMP_TZ('10:00:00 +09:00','HH24:MI:SS TZH:TZM'),
    end_time               => TO_TIMESTAMP_TZ('12:00:00 +09:00','HH24:MI:SS TZH:TZM'),
    day_of_week_mask       => '1111100' -- 월~금
  );
END;
/
```

> **사용법**: 문제 상황을 “좋을 때 베이스라인”과 비교하여 **회귀**를 신속히 검출.

---

## 8. **특정 SQL** 리포트/해석(AWR)

### 8.1 `awrsqrpt.sql` — SQL 한 건 집중
```sql
@?/rdbms/admin/awrsqrpt.sql
-- DBID/Inst/Snap 범위/SQL_ID 입력
```
- **실행 횟수, 평균 elapsed/CPU/reads**, child cursor 현황, 계획 변동(스냅샷 시점별)
- **계획 변동**이 응답시간 변화를 만든 경우 → **통계/히스토그램/바인드/프로파일** 조치

### 8.2 라인 통계(실행 후)
```sql
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR('&&SQL_ID', NULL,
  'ALLSTATS LAST +PEEKED_BINDS +PREDICATE +PROJECTION +OUTLINE +NOTE'));
```
- **A-Rows vs E-Rows**, **TempSpc**, **Buffers/Reads**로 병목 라인 결정
- **대기 이벤트**는 **ASH/SQL Monitor** 타임라인으로 상관 확인

---

## 9. **리포트 해석 체크리스트**(암기 카드)

1) **구간 고정**: 피크 30분 등 **일관된** 범위
2) **DB Time vs DB CPU**: 대기 비중 판단
3) **Top Events**: User I/O? Concurrency? Commit? Cluster? Library?
4) **Top SQL**: Elapsed/CPU/Reads/Execs별 상위 **3~5개** 선정
5) **I/O/Temp**: 파일/TS/오브젝트 누가 I/O/Temp를 잡아먹나
6) **Segments**: 핫 인덱스/테이블, buffer busy/ITL waits 대상
7) **Plan 라인**: `ALLSTATS LAST`로 **시간/Temp/버퍼** 큰 라인
8) **조치**: 인덱스/카디널리티/조인/정렬 회피/PGA/커밋/키분산/로컬리티
9) **비교 보고**: `awrdiff.sql`로 **효과** 수치화
10) **재발 방지**: 베이스라인/모니터링 임계치 구성

---

## 10. 자동화/스크립트(샘플)

### 10.1 최신 6시간 스냅샷 범위에서 AWR HTML 리포트 생성
```sql
-- SQL*Plus
DEF hours = 6
COL begin_snap NEW_VALUE bsn;
COL end_snap   NEW_VALUE esn;

SELECT MIN(snap_id) bsn, MAX(snap_id) esn
FROM   dba_hist_snapshot
WHERE  begin_interval_time > SYSDATE - &&hours/24;

HOST echo Generating AWR report for &&bsn to &&esn ...

@?/rdbms/admin/awrrpt.sql
-- 프롬프트에서 bsn/esn 사용
```

### 10.2 특정 서비스/모듈의 SQL Top-N(ASH) — 보조
```sql
SELECT sql_id,
       COUNT(*) samples,
       ROUND(100*RATIO_TO_REPORT(COUNT(*)) OVER (),1) pct
FROM   v$active_session_history
WHERE  sample_time > SYSTIMESTAMP - INTERVAL '30' MINUTE
  AND  service_hash = (SELECT service_hash FROM v$active_services WHERE name='PAYROLL_SVC')
  AND  module = 'PAYROLL-BATCH'
GROUP  BY sql_id
ORDER  BY samples DESC FETCH FIRST 10 ROWS ONLY;
```

---

## 11. 일반 함정 & 베스트 프랙티스

**함정**
- 스냅샷 **구간 길이**가 너무 길면 스파이크가 **희석**된다. 너무 짧으면 **노이즈**. (보통 15~30분 추천)
- Top SQL이 **총 DB Time의 5% 미만**이면 **시스템 현상**(락/로그/클러스터)일 수 있다.
- “BCHR 99%”만 보고 결론 금지 → **Temp/Commit/Concurrency**가 지배할 수 있음.
- RAC에서 인스턴스별 **부하/서비스 라우팅**을 무시하면 오판.

**베스트 프랙티스**
1) **Top Events → Top SQL → Plan 라인** 3단 점프
2) **전/후 비교**로 수치 증명(awrdiff)
3) **베이스라인**으로 “정상 상태” 고정 → 회귀 탐지
4) **Advisory**는 방향성으로만 사용(정답 아님)
5) AWR + **ASH**(타임라인/상관) + **SQL Monitor**(장수행/라인 시간) **병행**

---

## 12. 미니 실습(끝까지 따라하기)

### 12.1 피크 구간 리포트 → 병목 SQL → 라인
1) **AWR 리포트**: 10:30~11:00
   - Top Event: `direct path read temp` 1위
   - SQL Top1: `9pqs1m..`
2) **해당 SQL 라인 통계**
```sql
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR('9pqs1m..', NULL,
 'ALLSTATS LAST +PREDICATE +PROJECTION +PEEKED_BINDS +OUTLINE +NOTE'));
```
3) **조치**: 히스토그램 + 커버링 인덱스 + PGA↑
4) **비교 리포트**
```sql
@?/rdbms/admin/awrdiff.sql
-- Before: 1201~1202 / After: 1203~1204
```
5) **효과**: DB Time 30%↓, Temp I/O 80%↓, SQL Top1 Elapsed 42,300→3,100s

---

## 13. 수학(개념 리마인드)

- **응답시간 분해**
  $$ T_{\text{elapsed}} = T_{\text{CPU}} + \sum_k T_{\text{wait}_k} $$
- **평균 활성 세션(AAS)**
  $$ \text{AAS} \approx \frac{\text{DB Time}}{\text{Wall-clock Time}} $$

> 해석은 **시간** 기준으로: 어느 **이벤트**(이름표)가, 어떤 **SQL/세그먼트/라인**에서 시간을 **태우는가**.

---

## 14. 결론

- **Statspack/AWR**는 **시계열 성능 DNA**를 남기는 표준 도구다.
- 리포트는 **Top Events → Top SQL → Plan 라인 → 세그먼트/I/O** 순서로 읽는다.
- **비교 리포트**로 전/후를 수치로 증명하고, **베이스라인**으로 정상 상태를 고정하라.
- 문제의 **이름표(대기 이벤트)** 를 찾고, **좌표(Plan 라인)** 를 확인하고, **작고 정확한 변경**으로 해결하라.

> 한 줄 정리
> **AWR/Statspack**은 “문제의 시간과 장소”를 알려준다.
> 그 지도를 따라 **Plan 라인**까지 내려가면, **튜닝의 답**이 거기에 있다.
