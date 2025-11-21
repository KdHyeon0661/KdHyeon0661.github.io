---
layout: post
title: DB 심화 - Direct Path I/O
date: 2025-11-04 19:25:23 +0900
category: DB 심화
---
# Direct Path I/O — `direct path read/write temp`, `direct path read`, `direct path write`

> **한 줄 요약**
> - **Direct Path I/O**는 **버퍼 캐시를 우회**(bypass)하여 **세션의 PGA 또는 프로세스 버퍼 ↔ 데이터파일/임시파일**로 **대량 I/O**를 수행하는 모드다.
> - **읽기 계열**
>   - `direct path read temp` : **TEMP 공간**(임시 테이블스페이스)에서 **정렬/해시/스풀 결과**를 **직접** 읽을 때.
>   - `direct path read` : **데이터파일**에서 **테이블/인덱스/세그먼트 블록**을 **직접** 읽을 때(주로 **대량 스캔**, PX, Smart Scan 등).
> - **쓰기 계열**
>   - `direct path write temp` : TEMP에 **정렬·해시·스풀** 결과를 **직접** 쓸 때.
>   - `direct path write` : 데이터파일에 **CTAS/INSERT APPEND/병렬 로드** 등 **대량 적재/변경**을 **직접** 쓸 때.
>
> **언제 유리?**
> - **대용량 순차 작업**(Full/Partition Scan, 대규모 Sort/Hash, 병렬 실행, APPEND 적재)에서 **버퍼 캐시 오염 방지**와 **대역폭 극대화**로 **처리량↑**
> **주의점**
> - OLTP 혼합환경에서 무분별한 강제는 **캐시 재사용성↓**, I/O 폭증시 **스토리지 큐 포화** 가능.
> - TEMP 스필이 과하면 `direct path read/write temp` 지배 → **PGA/Workarea** 설정·실행계획 개선 필요.

---

## 실습 스키마 & 공통 환경

```sql
-- 실습용 큰 테이블 생성
DROP TABLE big_sales PURGE;

CREATE TABLE big_sales (
  sale_id     NUMBER PRIMARY KEY,
  cust_id     NUMBER NOT NULL,
  sale_dt     DATE   NOT NULL,
  amount      NUMBER(12,2) NOT NULL,
  status      VARCHAR2(8)  NOT NULL,
  pad         VARCHAR2(200)
);

INSERT /*+ APPEND */ INTO big_sales
SELECT level,
       MOD(level, 2000000)+1,
       DATE '2025-01-01' + MOD(level, 365),
       ROUND(DBMS_RANDOM.VALUE(1,1000),2),
       CASE MOD(level,4) WHEN 0 THEN 'OK' WHEN 1 THEN 'OK' WHEN 2 THEN 'CXL' ELSE 'HOLD' END,
       RPAD('x',200,'x')
FROM dual
CONNECT BY level <= 3000000;  -- 300만행
COMMIT;

CREATE INDEX ix_bs_cust_dt ON big_sales(cust_id, sale_dt DESC, sale_id DESC);

BEGIN
  DBMS_STATS.GATHER_TABLE_STATS(USER,'BIG_SALES',cascade=>TRUE, method_opt=>'for all columns size skewonly');
END;
/
```

> **실험 팁**
> - 세션별 이벤트/통계 관찰: `V$SESSION_EVENT`, `V$SESSTAT`, `V$SQL`, `DBMS_XPLAN.DISPLAY_CURSOR('…','…','ALLSTATS LAST +PEEKED_BINDS')`
> - 시스템/ASH 관찰: `V$SYSTEM_EVENT`, `V$ACTIVE_SESSION_HISTORY`, `V$TEMPSEG_USAGE`
> - TEMP 사용량: `V$TEMPSEG_USAGE`, `DBA_TEMP_FREE_SPACE`
> - I/O 계열 통계: `V$SYSSTAT`(physical reads direct, physical writes direct, physical reads/writes direct temporary)

---

## Direct Path의 큰 그림

### 무엇을 “우회(bypass)”하나?

- **버퍼 캐시(SGA) 우회**: 전통적 버퍼드 읽기/쓰기(캐시에 블록 적재)를 건너뛰고,
- **세션/프로세스 버퍼(주로 PGA)** ↔ **스토리지(데이터파일/TEMP 파일)** 로 **큰 I/O 청크**를 직접 발주.

### 왜 빠를 수 있나?

- **큰 I/O 사이즈**(연속 블록 읽기/쓰기)로 **요청 수↓, 대역폭↑, 컨텍스트 스위칭↓**
- **버퍼 캐시 오염 방지**: 대용량 작업이 캐시를 밀어내지 않음 → OLTP 캐시 보호.
- **병렬 실행(PX)** 과 결합 시 스토리지 파이프라인을 **동시에 폭 넓게 사용**.

### 언제 선택되나? (일반적 휴리스틱)

- **대량 Full/Partition Scan**(카디널리티 크고 재사용성 낮음)
- **정렬/해시 연산**이 PGA를 초과하여 **TEMP 스필** 발생
- **CTAS/INSERT /*+ APPEND */ / 병렬 적재**, 일부 `ALTER INDEX REBUILD` 등
- **Exadata**: Smart Scan 시 `cell smart table/index scan` + direct path 계열 이벤트 동반

> **주의**: 버전/플랫폼/스토리지/파라미터에 따라 옵티마이저·I/O 레이어 판단이 달라질 수 있다.
> - 참고 세션 파라미터(상황 진단용):
>   - `workarea_size_policy` (=AUTO 권장)
>   - `pga_aggregate_target` / `pga_aggregate_limit`
>   - `parallel_degree_policy`, `parallel_min_time_threshold`
>   - (테스트) `"_serial_direct_read"`(TRUE 시 직렬에서도 direct read 유도)

---

## `direct path read temp` / `direct path write temp` — TEMP 기반 Direct I/O

### 개념

- **정렬(SORT)**, **해시 조인/해시 집계**, **Analytic** 윈도우, **ORDER BY + FETCH** 등 **작업 영역(workarea)** 이 **PGA 용량을 초과**하면,
  엔진은 중간 결과를 **TEMP 테이블스페이스**로 **스필(spill)** 한다.
- 이때 TEMP 파일로의 I/O는 **버퍼 캐시와 무관** → `direct path write temp` / `direct path read temp`.

### 실습: 의도적으로 TEMP 스필 유도

```sql
-- (테스트 세션에서) PGA를 작게 잡아 스필 유도
ALTER SESSION SET workarea_size_policy = AUTO;
ALTER SESSION SET sort_area_size = 65536; -- 구버전 호환; AUTO여도 영향 있을 수 있음(환경차)
-- 또는 전체 DB 차원의 PGA가 작아도 스필 발생

-- 큰 Sort + Group By
SELECT /* 대량정렬/집계 */
       cust_id, COUNT(*) cnt, SUM(amount) amt
FROM   big_sales
GROUP  BY cust_id
ORDER  BY amt DESC
FETCH FIRST 1000 ROWS ONLY;
```

**관찰 포인트**
```sql
-- 세션 이벤트
SELECT event, total_waits, time_waited_micro/1e6 sec
FROM   v$session_event
WHERE  sid = SYS_CONTEXT('USERENV','SID')
AND    event LIKE 'direct path % temp'
ORDER BY sec DESC;

-- TEMP 사용 실시간
SELECT s.sid, u.tablespace, u.segfile#, u.segblk#, u.blocks
FROM   v$session s
JOIN   v$tempseg_usage u ON s.saddr = u.session_addr
WHERE  s.sid = SYS_CONTEXT('USERENV','SID');

-- 시스템 통계
SELECT name, value
FROM   v$sysstat
WHERE  name IN ('physical writes direct temporary',
                'physical reads direct temporary');
```

**해석**
- `direct path write temp` 가 증가: 정렬/해시 중간 결과를 TEMP에 **직접 쓰는 중**
- 이후 결과를 소비할 때 `direct path read temp` 로 **직접 읽음**
- TEMP가 과도하면 I/O 지연 → **PGA 증가**, **쿼리 재작성**, **병렬/파티션 전략** 등으로 개선

### 튜닝 포인트

- **PGA/Workarea**: `workarea_size_policy=AUTO` + **적절한 `pga_aggregate_target`** (AWR에서 PGA Target Hit% 관찰)
- **실행계획**: JOIN 순서/방법을 바꿔 **해시 테이블 크기↓**, **프루닝**으로 입력량↓, **조기 필터링**
- **TOP-N/Stopkey**: ORDER BY 전체 정렬 ↓ → `FETCH FIRST N ROWS ONLY` + 정렬일치 인덱스
- **쿼리 구조**: 윈도우 함수 범위 축소/사전 집계/리스트 제한

---

## `direct path read` — 데이터파일 직접 읽기

### 개념

- 테이블/인덱스 **세그먼트**에서 대량 연속 I/O가 필요할 때, **버퍼 캐시를 거치지 않고** 세션 버퍼로 **직접 읽음**.
- 전형 사례
  - **대량 Full/Partition Scan**(특히 **재사용성 낮은** 블록)
  - **병렬 실행(PX)** 의 스캔 단계
  - **Exadata Smart Scan** 동반(이벤트는 `cell smart table scan` 등으로 표기, 함께 나타날 수 있음)

### 실습: 직렬 세션에서 direct path read 유도

```sql
-- 테스트 세션: 직렬 direct read 유도 (주의: 테스트 용도)
ALTER SESSION SET "_serial_direct_read" = TRUE;

-- 대량 범위 집계
SELECT /*+ full(b) */
       COUNT(*) , SUM(amount)
FROM   big_sales b
WHERE  sale_dt >= DATE '2025-10-01';
```

**관찰**
```sql
SELECT event, total_waits, time_waited_micro/1e6 sec
FROM   v$session_event
WHERE  sid = SYS_CONTEXT('USERENV','SID')
AND    event IN ('db file scattered read','direct path read','direct path read temp')
ORDER  BY sec DESC;

SELECT name, value
FROM   v$sysstat
WHERE  name LIKE 'physical reads direct%';
```

- `direct path read` 가 증가하면 **데이터파일에서 캐시 우회 읽기** 수행 중.
- `db file scattered read` 대비 **호출 수↓/I/O 크기↑** 경향(스토리지 상황에 따라 상이).

### 병렬 실행(PX)에서의 direct path read

```sql
-- 병렬 스캔
ALTER SESSION SET parallel_degree_policy = MANUAL;

SELECT /*+ full(b) parallel(b 8) use_hash(b) */
       status, COUNT(*), SUM(amount)
FROM   big_sales b
WHERE  sale_dt >= ADD_MONTHS(TRUNC(SYSDATE,'MM'), -3)
GROUP  BY status;
```

- PX 슬레이브가 각 파티션/범위를 나눠 **동시에 direct path read**.
- 스토리지가 받쳐주면 **대역폭 극대화**, 그렇지 않으면 **큐 포화**로 RT↑.

### 언제 `direct path read` 가 덜 유리한가?

- **작은 랜덤 조회**(OLTP)에서는 **버퍼 캐시 재사용**이 더 낫다.
- **혼합 워크로드**에서 무차별 direct read는 **캐시 재사용** 기회를 놓치고, **I/O 경쟁**을 유발할 수 있다.
- 판단 기준은 항상 **실측 RT/대기 이벤트**.

---

## `direct path write` — 데이터파일 직접 쓰기

### 개념

- **대량 적재/변경** 시, 세션 버퍼에서 데이터파일로 **직접 쓰기** 수행.
- 전형 사례
  - **CTAS (CREATE TABLE AS SELECT)**
  - **INSERT /*+ APPEND */** (APPEND 힌트)
  - **병렬 INSERT/병렬 로드**(SQL*Loader Direct Path 등)
  - **인덱스 재구성/리빌드**의 일부 단계

### 실습: APPEND 적재와 CTAS

```sql
-- APPEND: High Water Mark(HWM) 위쪽에 연속 공간을 확보해 직접 쓰기 (버퍼 캐시 오염 최소)
CREATE TABLE big_sales_copy NOLOGGING AS
SELECT /*+ APPEND */ * FROM big_sales WHERE sale_dt >= DATE '2025-01-01';

-- 또는
INSERT /*+ APPEND */ INTO big_sales_copy
SELECT * FROM big_sales WHERE amount > 500;
COMMIT;
```

**관찰**
```sql
SELECT event, total_waits, time_waited_micro/1e6 sec
FROM   v$session_event
WHERE  sid = SYS_CONTEXT('USERENV','SID')
AND    event LIKE 'direct path write%';

SELECT name, value
FROM   v$sysstat
WHERE  name IN ('physical writes direct', 'physical writes direct temporary');
```

- `direct path write` 증가: **데이터파일로 직접 쓰기**
- APPEND는 **연속 공간** 할당 → **멀티블록 쓰기** 유리, **캐시 오염 방지**

### 주의와 팁

- **REDO/UNDO 정책**: NOLOGGING + APPEND는 **REDO 최소화** 가능(복구 전략 고려 필수).
- **병렬도**: 과도한 병렬 쓰기는 **스토리지 큐 포화** → RT 급증 가능.
- **세그먼트 공간 관리**(ASSM 등)와 **확장(extent) 크기**가 **연속성**에 영향.

---

## Direct Path I/O와 실행계획/트레이스 진단

### 플랜·실행 통계 확인

```sql
ALTER SESSION SET statistics_level = ALL;

-- 대상 SQL 수행 후
SELECT *
FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL, NULL, 'ALLSTATS LAST +PEEKED_BINDS +ALIAS'));
```
- **노드 해석**
  - `TABLE ACCESS FULL` + (PX/병렬) → `direct path read` 유력
  - `CREATE TABLE AS SELECT` / `INSERT /*+ APPEND */` → `direct path write` 유력
  - 대량 Sort/Hash → TEMP 사용 → `direct path write temp` / `direct path read temp`
- **Row Source stats** 에서 **`rows`, `bytes`, `starts`, `A-Rows`** 로 각 단계의 **데이터 양** 파악

### 세션/시스템 이벤트, 통계

```sql
-- 세션 이벤트
SELECT event, total_waits, time_waited_micro/1e6 sec
FROM   v$session_event
WHERE  sid = SYS_CONTEXT('USERENV','SID')
AND    event LIKE 'direct path%';

-- 시스템 통계 요약
SELECT name, value
FROM   v$sysstat
WHERE  name LIKE 'physical % direct%';

-- TEMP 사용
SELECT s.sid, u.tablespace, u.blocks
FROM   v$session s
JOIN   v$tempseg_usage u ON s.saddr=u.session_addr
WHERE  s.sid = SYS_CONTEXT('USERENV','SID');
```

### ASH로 최근 10분 관찰

```sql
SELECT event, COUNT(*) samples
FROM   v$active_session_history
WHERE  sample_time > SYSTIMESTAMP - INTERVAL '10' MINUTE
AND    session_type='FOREGROUND'
AND    event LIKE 'direct path%'
GROUP  BY event
ORDER  BY samples DESC;
```

---

## 튜닝 전략 총정리 — 언제/어떻게 Direct Path를 취할 것인가?

### 읽기 측면

- **DW/보고·배치**:
  - **해시 조인 + Full/Partition Scan** 중심 → **direct path read** 자연 발생
  - **프루닝/블룸**으로 **스캔 범위 자체 축소**(작은 전체)
  - **병렬도**는 스토리지/네트워크/CPU와 균형(“느릴수록 더 높은 병렬”은 위험)
- **OLTP/혼합**:
  - 기본은 **버퍼 캐시 재사용** 경로가 유리
  - 무차별 `"_serial_direct_read"=TRUE` 금물
  - **정렬 일치 인덱스 + Stopkey**, **커버링 인덱스**로 **읽을 양 자체 감소**

### 측면

- **PGA 여유**와 **AUTO Workarea**로 스필 최소화 → `direct path read/write temp` 감소
- 쿼리 구조 변경: **사전 집계/조기 필터/윈도우 범위 축소**
- **Top-N/Keyset**으로 불필요 전체정렬 방지

### 쓰기 측면(대량 적재/변경)

- **APPEND/CTAS/병렬**로 **direct path write** 활용
- **NOLOGGING** + 백업/복구 정책 수립(필요 시 FORCE LOGGING 환경 주의)
- **분할 정복**: 파티션 단위 CTAS/교체(Exchange Partition)로 **가동중 적재** 최적화

---

## BEFORE → AFTER 시나리오

### 최근 분기 집계 보고서

**Before (버퍼드 + 랜덤 혼재)**
```sql
SELECT c.region, COUNT(*), SUM(b.amount)
FROM   big_sales b
JOIN   (SELECT DISTINCT cust_id, 'APAC' region FROM big_sales WHERE cust_id BETWEEN 1 AND 500000) c
  ON   c.cust_id = b.cust_id
WHERE  b.sale_dt >= ADD_MONTHS(TRUNC(SYSDATE,'Q'), -1)
GROUP  BY c.region;
```
- 실행 중 `db file scattered read`/`sequential read` 혼재, TEMP 스필 일부

**After (direct path read 중심 + 프루닝/병렬)**
```sql
ALTER SESSION SET parallel_degree_policy = MANUAL;

SELECT /*+ full(b) use_hash(b) parallel(b 8) */
       'APAC' region, COUNT(*), SUM(b.amount)
FROM   big_sales b
WHERE  b.sale_dt >= ADD_MONTHS(TRUNC(SYSDATE,'Q'), -1)
AND    b.cust_id BETWEEN 1 AND 500000
GROUP  BY 'APAC';
```
- **`direct path read` 비중↑**, 호출 수↓, 총 시간 단축
- TEMP 스필 줄이려면 Top-N/사전 필터 추가

### 하루치 증분 적재

**Before**
```sql
INSERT INTO big_sales
SELECT * FROM staging_sales_day;  -- 버퍼드 쓰기, redo↑, 캐시 오염
COMMIT;
```

**After (direct path write)**
```sql
INSERT /*+ APPEND */ INTO big_sales
SELECT /* 필요 컬럼 변환/검증 */ * FROM staging_sales_day;
COMMIT;
```
- **`direct path write`** 관찰, **연속 공간**에 쓰기로 **I/O 호출 효율↑**, **버퍼 캐시 오염↓**
- 필요 시 **NOLOGGING** + 백업 전략 병행

### TEMP 스필 줄이기

**Before (스필 과다)**
```sql
SELECT cust_id, SUM(amount) amt
FROM   big_sales
GROUP  BY cust_id
ORDER  BY amt DESC
FETCH FIRST 5000 ROWS ONLY; -- 전체 집계 후 정렬 → TEMP 스필 큼
```

**After (사전 필터/Top-N 구조 개선 + 인덱스 활용)**
```sql
-- 예: 최근 3개월만
SELECT /*+ index(b ix_bs_cust_dt) */
       b.cust_id, SUM(b.amount) amt
FROM   big_sales b
WHERE  b.sale_dt >= ADD_MONTHS(TRUNC(SYSDATE,'MM'), -3)
GROUP  BY b.cust_id
ORDER  BY amt DESC
FETCH FIRST 5000 ROWS ONLY;
```
- 입력량 축소 → TEMP 스필/`direct path write/read temp` 감소

---

## 체크리스트

- [ ] **보고/배치**: `direct path read`로 **순차 대량 읽기**를 가져왔는가? (프루닝/병렬/블룸)
- [ ] **적재/변경**: APPEND/CTAS로 **`direct path write`**를 활용하는가? (연속성/NOLOGGING 정책)
- [ ] **정렬/해시**: **PGA 충분**? 스필 최소화되어 `direct path read/write temp` 과다 발생을 막았는가?
- [ ] **혼합 워크로드**: OLTP 경로에 **직렬 direct read 강제**를 남발하지 않았는가?
- [ ] **계측**: AWR/ASH/SQL Monitor로 **이벤트 비중·총 시간**이 실제로 개선되었는가?

---

## 수식으로 보는 의사 의사결정

- **대량 스캔 처리량 근사**
  $$ \text{Throughput} \approx \frac{\text{Read Size per IO} \times \text{IOPS}}{\text{Concurrency Penalty}} $$
  - `direct path read`는 **Read Size per IO↑** 로 **호출 수↓**를 유도
- **TEMP 스필 임계**
  $$ \text{Workarea Demand} > \text{PGA Grant} \Rightarrow \text{direct path write temp} \to \text{direct path read temp} $$
  - `PGA Grant`(작업 영역 할당) 확대 또는 **입력량/카디널리티 감소**로 회피

---

## 명령·질의 스니펫 모음 (복사/붙여넣기)

**10.1 이벤트·통계 한눈에**
```sql
-- 세션 이벤트(Direct Path)
SELECT event, total_waits, time_waited_micro/1e6 sec
FROM   v$session_event
WHERE  sid = SYS_CONTEXT('USERENV','SID')
AND    event LIKE 'direct path%';

-- 시스템 통계(Direct IO)
SELECT name, value
FROM   v$sysstat
WHERE  name LIKE 'physical % direct%';

-- TEMP 사용
SELECT s.sid, u.tablespace, u.blocks*vt.block_size/1024/1024 mb
FROM   v$session s
JOIN   v$tempseg_usage u ON s.saddr = u.session_addr
JOIN   v$tablespace t    ON t.ts#    = u.ts#
JOIN   v$tempfile vt     ON vt.ts#   = t.ts#
WHERE  s.sid = SYS_CONTEXT('USERENV','SID');
```

**10.2 직렬 direct read 테스트(주의: 테스트용)**
```sql
ALTER SESSION SET "_serial_direct_read" = TRUE;
SELECT /*+ full(b) */ COUNT(*) FROM big_sales b WHERE sale_dt >= DATE '2025-10-01';
```

**10.3 APPEND / CTAS**
```sql
INSERT /*+ APPEND */ INTO big_sales
SELECT * FROM big_sales WHERE amount > 900;
COMMIT;

CREATE TABLE big_sales_part NOLOGGING PARALLEL 8 AS
SELECT * FROM big_sales WHERE sale_dt >= DATE '2025-10-01';
```

**10.4 병렬 스캔**
```sql
ALTER SESSION SET parallel_degree_policy = MANUAL;
SELECT /*+ full(b) parallel(b 8) */ COUNT(*) FROM big_sales b;
```

---

## 결론

- **Direct Path I/O**는 **대량 작업의 처리량**을 크게 높이는 핵심 메커니즘이다.
  - **읽기**: `direct path read` / `direct path read temp`
  - **쓰기**: `direct path write` / `direct path write temp`
- **언제나 유리한 것은 아니다.**
  - **DW/보고/배치**엔 강력,
  - **OLTP/혼합**에서는 **캐시 재사용**·**랜덤 지연**·**스토리지 큐** 등을 고려해 **선택적으로** 적용.
- 성공 기준은 **히트율**이 아니라 **응답시간/총 시간/처리량**이다.
  - AWR/ASH/SQL Monitor/TKPROF 로 **전/후 계측**을 통해,
  - **TEMP 스필**, **읽기/쓰기 Direct Path 이벤트 비중**, **병렬 대역폭**을 **숫자로** 확인하라.
