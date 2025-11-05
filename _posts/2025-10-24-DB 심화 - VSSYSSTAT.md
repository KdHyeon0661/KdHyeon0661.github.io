---
layout: post
title: DB 심화 - V$SYSSTAT
date: 2025-10-24 17:25:23 +0900
category: DB 심화
---
# Oracle `V$SYSSTAT` 완전 가이드  
— **시스템 수행 통계 수집 & 분석** + **Ratio 기반 성능 분석** (예제 풍부, 함정·해석법까지)

> 목표  
> - `V$SYSSTAT`이 제공하는 **시스템 전역(인스턴스 단위) 누적 통계**를 **정확히 수집**하고, **시간구간별 델타**로 **추세/부하 타입**을 판독한다.  
> - **Ratio 기반 성능 분석**(Buffer Cache Hit Ratio, Soft-Parse %, Logical/Physical mix, Sort In-Memory %, Redo/Txn 등)의 **정의·수식·SQL**을 제시하고, **해석 시 주의점(함정)** 을 함께 제공한다.  
> - **운영 절차(샘플링 → 델타 계산 → Ratio/절대치 동시 관찰 → 원인 추적)** 를 **템플릿**과 **사례**로 정리한다.

---

## 0) 큰 그림: `V$SYSSTAT`를 왜, 어떻게 보나?

- `V$SYSSTAT` = 인스턴스 기동 이후 누적된 **시스템 전체** 통계(“statistic#”, “name”, “value”).  
- **샘플링 시점 A, B의 차이(Δ)** 를 내면 **해당 구간의 활동량**을 얻는다.  
- 이 델타들을 **합리적으로 결합**하면 **Ratio**(비율)들이 도출되고, 이를 통해 **캐시 효율, 파싱 행태, I/O mix, 정렬 스필** 등 **시스템 전반 상태**를 판단할 수 있다.  
- 단, **Ratio는 보조지표**이다. **절대치(초당, 분당)** 와 함께 봐야 해석이 안전하다.

---

## 1) 준비: 자주 쓰는 주요 지표 이름(요지)

| 범주 | 대표 통계명(`name`) | 의미(누적) |
|---|---|---|
| 메모리/논리 읽기 | `consistent gets`, `db block gets`, `session logical reads` | CR/현재 블록 논리 읽기 수, 합(logical) |
| 물리 I/O | `physical reads`, `physical writes`, `physical read bytes`, `physical write bytes` | 디스크 I/O 수/바이트 |
| 파싱/실행 | `parse count (total)`, `parse count (hard)`, `execute count` | 총 파싱/하드파싱/실행 호출 수 |
| 정렬 | `sorts (memory)`, `sorts (disk)` | PGA 내 정렬, TEMP 스필 정렬 수 |
| 네트워크 | `bytes sent via SQL*Net to client`, `bytes received via SQL*Net from client` | 클라이언트 왕복 바이트 |
| Redo | `redo size`, `redo writes`, `redo entries` | 로그 생성량/쓰기 횟수/엔트리 수 |
| 트랜잭션 | `user commits`, `user rollbacks` | 사용자 커밋/롤백 수 |
| DDL/딕셔너리 | `recursive calls` | 내부(리커시브) 호출 수 |
| 기타 | `opened cursors cumulative` | 누적 열린 커서 수 |

> 버전에 따라 통계명이 추가/변경될 수 있음. 분석 전 **실제 인스턴스의 name 목록**을 먼저 확인하자.

```sql
-- 내가 가진 통계 이름 훑기(부분일치)
SELECT name FROM v$sysstat
WHERE LOWER(name) LIKE '%consistent gets%' OR LOWER(name) LIKE '%parse count%';
```

---

## 2) 수집 원칙: **누적 → 델타 → 단위 시간 보정** (핵심 템플릿 제공)

### 2.1 스냅샷 테이블(임시) 만들기

```sql
-- 1) 샘플 저장용 GTT(세션 단위 유지)
CREATE GLOBAL TEMPORARY TABLE snap_sysstat (
  snap_ts  TIMESTAMP DEFAULT SYSTIMESTAMP,
  name     VARCHAR2(64),
  value    NUMBER
) ON COMMIT PRESERVE ROWS;

-- 2) 스냅샷 채우기(현재 시점)
INSERT INTO snap_sysstat(name, value)
SELECT name, value FROM v$sysstat;

COMMIT;
```

### 2.2 N초 후 델타 계산

```sql
-- N초 대기(예: 60s) 후 비교
-- (SQL*Plus/클라이언트 타이머 사용; 여기선 개념만)

WITH prev AS (
  SELECT name, value AS v1
  FROM   snap_sysstat
),
curr AS (
  SELECT name, value AS v2
  FROM   v$sysstat
)
SELECT c.name,
       c.v2 - p.v1 AS delta
FROM   curr c JOIN prev p USING(name)
ORDER  BY 2 DESC;
```

> **Tip**: 델타를 **초당 값**으로 환산하려면 $$ \text{per\_sec}=\frac{\Delta}{\Delta t} $$.  
> 수집 스크립트에 **경과시간(초)** 를 변수로 넣어두면 편하다.

---

## 3) Ratio 라이브러리(수식 + SQL)

### 3.1 Buffer Cache Hit Ratio (BCHR) — (주의: 과신 금물)

**아이디어**: 논리읽기 중 물리읽기가 차지하는 비율.  
가볍게는 다음 근사 사용:

$$
\text{BCHR} \approx 1 - \frac{\Delta\text{physical reads}}{\Delta\text{session logical reads}}
$$

- `session logical reads = consistent gets + db block gets`(일반적으로)

```sql
WITH d AS (
  SELECT
    MAX(CASE WHEN name='physical reads' THEN delta END) pr,
    MAX(CASE WHEN name='consistent gets' THEN delta END) cg,
    MAX(CASE WHEN name='db block gets'  THEN delta END) dbg
  FROM (
    -- 델타 준비(위 2.2 예제를 CTE로 래핑했다고 가정)
    SELECT c.name, (c.value - p.value) delta
    FROM   v$sysstat c JOIN snap_sysstat p USING(name)
  )
)
SELECT 1 - (pr / NULLIF((cg+dbg),0)) AS bchr
FROM d;
```

**해석 주의**  
- BCHR이 높아도(예: 99% 이상) **I/O 대기**가 클 수 있다(핫셋 외 범위 스캔/직접경로/대형 정렬 등).  
- 반드시 **절대치(초당 물리읽기량)** 와 **Top Wait/Top SQL** 을 함께 보라.

---

### 3.2 Soft-Parse Ratio

**정의**: 전체 파싱 중 하드 파싱이 차지하는 비율.  
소프트파싱 비율(Soft %):

$$
\text{SoftParse\%} = 1 - \frac{\Delta\text{parse count (hard)}}{\Delta\text{parse count (total)}}
$$

```sql
WITH d AS (
  SELECT
    MAX(CASE WHEN name='parse count (total)' THEN delta END) p_total,
    MAX(CASE WHEN name='parse count (hard)'  THEN delta END) p_hard
  FROM (
    SELECT c.name, (c.value - p.value) delta
    FROM   v$sysstat c JOIN snap_sysstat p USING(name)
  )
)
SELECT 1 - (p_hard / NULLIF(p_total,0)) AS soft_parse_ratio
FROM d;
```

**해석**  
- SoftParse%가 낮다면 **바인드 미사용/커서 재사용 실패** 가능성.  
- 함께 볼 지표: `v$sqlarea`의 `parse_calls`, `executions`, `version_count`, `v$sql_shared_cursor`(child 폭증 이유), `session_cached_cursors`.

---

### 3.3 논리/물리 Mix (Logical per Physical)

$$
\text{Logical per Physical} = \frac{\Delta\text{session logical reads}}{\Delta\text{physical reads}}
$$

```sql
WITH d AS (
  SELECT
    MAX(CASE WHEN name='session logical reads' THEN delta END) lr,
    MAX(CASE WHEN name='physical reads'       THEN delta END) pr
  FROM (
    SELECT c.name, (c.value - p.value) delta
    FROM   v$sysstat c JOIN snap_sysstat p USING(name)
  )
)
SELECT lr / NULLIF(pr,0) AS logical_per_physical
FROM d;
```

**해석**  
- 값이 크면 상대적으로 캐시 적중이 좋아 보이지만, **읽는 양 자체가 과도**하면 여전히 느릴 수 있다.  
- **Top SQL/ASH Wait** 로 실제 병목이 **I/O인지, CPU인지, 락인지** 교차 확인.

---

### 3.4 Sort In-Memory Ratio (정렬 스필 감시)

$$
\text{SortInMem\%} = \frac{\Delta\text{sorts (memory)}}{\Delta\text{sorts (memory)}+\Delta\text{sorts (disk)}}
$$

```sql
WITH d AS (
  SELECT
    MAX(CASE WHEN name='sorts (memory)' THEN delta END) sm,
    MAX(CASE WHEN name='sorts (disk)'   THEN delta END) sd
  FROM (
    SELECT c.name, (c.value - p.value) delta
    FROM   v$sysstat c JOIN snap_sysstat p USING(name)
  )
)
SELECT (sm / NULLIF(sm+sd,0)) AS sort_in_mem_ratio
FROM d;
```

**해석 & 개선**  
- In-Memory%가 낮으면 **PGA 부족/카디널리티 오판/대형 해시&정렬 전략** 가능.  
- **조치**: `pga_aggregate_target` 검토, **카디널리티 정확화(히스토그램/조인순서)**, 인덱스로 정렬 회피.

---

### 3.5 Redo per Transaction / Commit Rate

```sql
WITH d AS (
  SELECT
    MAX(CASE WHEN name='redo size'    THEN delta END) redo_bytes,
    MAX(CASE WHEN name='user commits' THEN delta END) commits,
    MAX(CASE WHEN name='user rollbacks' THEN delta END) rollbacks
  FROM (
    SELECT c.name, (c.value - p.value) delta
    FROM   v$sysstat c JOIN snap_sysstat p USING(name)
  )
)
SELECT redo_bytes / NULLIF((commits+rollbacks),0) AS redo_per_tx_bytes,
       commits/(commits+rollbacks)                AS commit_ratio
FROM d;
```

**해석**  
- Tx당 Redo가 과도하면 **대형 DML/배치/LOB** 가능.  
- `log file sync` 가 상위 이벤트로 보이면 **커밋 빈도**/네트워크/스토리지 지연 검토.

---

### 3.6 Execute per Parse (실행/파싱 효율)

$$
\text{Exec/Parse} = \frac{\Delta\text{execute count}}{\Delta\text{parse count (total)}}
$$

```sql
WITH d AS (
  SELECT
    MAX(CASE WHEN name='execute count'        THEN delta END) execs,
    MAX(CASE WHEN name='parse count (total)'  THEN delta END) ptotal
  FROM (
    SELECT c.name, (c.value - p.value) delta
    FROM   v$sysstat c JOIN snap_sysstat p USING(name)
  )
)
SELECT execs / NULLIF(ptotal,0) AS exec_per_parse
FROM d;
```

**해석**  
- 값이 낮으면 **파싱이 잦은** 애플리케이션(리터럴 남발/커서 재사용 실패) 가능.

---

### 3.7 Current vs Consistent Reads Mix

```sql
WITH d AS (
  SELECT
    MAX(CASE WHEN name='consistent gets' THEN delta END) cg,
    MAX(CASE WHEN name='db block gets'   THEN delta END) dbg
  FROM (
    SELECT c.name, (c.value - p.value) delta
    FROM   v$sysstat c JOIN snap_sysstat p USING(name)
  )
)
SELECT cg, dbg,
       dbg / NULLIF((cg+dbg),0) AS current_read_ratio
FROM d;
```

**해석**  
- `db block gets` 비중↑ = **현재 모드 읽기**(갱신 경로/세그먼트 헤더 등) 증가.  
- DML 집중/핫블록 경합 시 함께 증가할 수 있다.

---

## 4) “원클릭” 델타 & Ratio 리포트(예제 뷰)

아래는 **이전 스냅샷 vs 현재**의 델타와 주요 Ratio를 **한 번에** 보는 예시.

```sql
-- 1) 스냅샷 갱신(이전 삭제 → 새로 저장)
TRUNCATE TABLE snap_sysstat;
INSERT INTO snap_sysstat(name, value)
SELECT name, value FROM v$sysstat;
COMMIT;

-- (N초 후) 2) 리포트
WITH d AS (
  SELECT c.name, (c.value - p.value) delta
  FROM   v$sysstat c JOIN snap_sysstat p USING(name)
),
m AS (
  SELECT
    MAX(CASE WHEN name='session logical reads' THEN delta END) lr,
    MAX(CASE WHEN name='physical reads'        THEN delta END) pr,
    MAX(CASE WHEN name='consistent gets'       THEN delta END) cg,
    MAX(CASE WHEN name='db block gets'         THEN delta END) dbg,
    MAX(CASE WHEN name='parse count (total)'   THEN delta END) ptotal,
    MAX(CASE WHEN name='parse count (hard)'    THEN delta END) phard,
    MAX(CASE WHEN name='execute count'         THEN delta END) execs,
    MAX(CASE WHEN name='sorts (memory)'        THEN delta END) s_mem,
    MAX(CASE WHEN name='sorts (disk)'          THEN delta END) s_disk,
    MAX(CASE WHEN name='redo size'             THEN delta END) redo_b,
    MAX(CASE WHEN name='user commits'          THEN delta END) commits,
    MAX(CASE WHEN name='user rollbacks'        THEN delta END) rollbacks
  FROM d
)
SELECT
  -- 절대치
  lr   AS logical_reads,
  pr   AS physical_reads,
  execs,
  ptotal, phard,
  s_mem, s_disk,
  redo_b, commits, rollbacks,
  -- Ratio
  CASE WHEN lr>0 THEN 1 - (pr/lr) END                           AS bchr_approx,
  1 - (phard/NULLIF(ptotal,0))                                  AS soft_parse_ratio,
  (s_mem/NULLIF(s_mem+s_disk,0))                                AS sort_in_mem_ratio,
  (lr/NULLIF(pr,0))                                             AS logical_per_physical,
  (redo_b/NULLIF((commits+rollbacks),0))                        AS redo_per_tx_bytes,
  (commits/NULLIF((commits+rollbacks),0))                       AS commit_ratio,
  (dbg/NULLIF((cg+dbg),0))                                      AS current_read_ratio
FROM m;
```

> **현장 팁**  
> - 이 리포트를 **분당** 또는 **5초/10초** 간격으로 자동 수집해 그래프화하면 **변화점**이 바로 보인다.  
> - **Top Waits/SQL**(AWR/ASH)와 함께 **대시보드**에 붙이면 **원인 추적 속도**가 빨라진다.

---

## 5) 해석의 기술: Ratio만 보지 말고 “함께” 보라

1) **Ratio + 절대치(per second)** 를 함께 본다.  
   - BCHR 높아도 **초당 물리 I/O**가 큰데 느릴 수 있다.  
   - SoftParse%가 99%라도 **Exec/Parse**가 1.1이면 “거의 매 실행마다 parse”일 수 있다.  
2) **시간모델/대기**와 교차 확인(보조)  
   - `V$SYS_TIME_MODEL`(DB CPU, SQL execute elapsed time)  
   - `V$SYSTEM_EVENT`(Top Waits), `V$ACTIVE_SESSION_HISTORY`(샘플 기반)  
3) **업무 패턴** 고려  
   - 배치 시간의 Ratio는 OLTP와 다르게 나오는 게 정상.  
4) **지속성 vs 스파이크** 구분  
   - 1~2 스냅으로 결론 내리지 말고 **추세**로 판단.

---

## 6) 사례 연구

### 6.1 “BCHR 99.5%인데 느리다”
- **관측**: `bchr_approx=0.995`, 하지만 `direct path read temp`/`log file sync`가 Top Wait.  
- **원인 가설**: 대형 정렬 스필 또는 커밋 과다.  
- **검증**: `sort_in_mem_ratio`↓, `redo_per_tx_bytes`/`commits per sec` 확인, SQL Monitor에서 TempSpc↑ 확인.  
- **조치**: PGA 상향/카디널리티 보정(스필 감소) 또는 커밋 정책/스토리지 지연 점검.

### 6.2 Soft-Parse% 낮음 + Exec/Parse 낮음
- **관측**: `soft_parse_ratio=0.7`, `exec_per_parse≈1.3`.  
- **원인**: 리터럴 남발/커서 재사용 실패.  
- **검증**: `v$sql_shared_cursor`, `version_count`↑, `session_cached_cursors` 낮음.  
- **조치**: 바인드 도입, 커서 캐시 설정, SQL 표준화.

### 6.3 Sort In-Memory% 저조, 물리읽기 폭증
- **관측**: `sort_in_mem_ratio=0.4`, `physical_reads/sec` 큼.  
- **원인**: 대형 해시/정렬 스필.  
- **조치**: 인덱스 설계/조인순서 조정, PGA↑, 파티션 프루닝.

---

## 7) 빠른 동일-시각 스냅 비교(“전/후” 실험)

튜닝 전후 **같은 기간**에 아래처럼 두 스냅을 비교해 **효과**를 수치로 증명한다.

```sql
-- 스냅샷 A 저장
TRUNCATE TABLE snap_sysstat;
INSERT INTO snap_sysstat(name, value)
SELECT name, value FROM v$sysstat;
COMMIT;

-- (튜닝 적용 후 N분간 부하)

-- 스냅샷 B와의 델타
WITH delta AS (
  SELECT c.name, c.value - p.value AS delta
  FROM   v$sysstat c JOIN snap_sysstat p USING(name)
)
SELECT * FROM delta
WHERE name IN ('session logical reads','physical reads','parse count (total)','parse count (hard)',
               'sorts (memory)','sorts (disk)','redo size','user commits','user rollbacks')
ORDER BY name;
```

**리포트 포맷팅**: 위 4장 “원클릭 리포트”를 함께 출력해 **Ratio 변화**도 보인다.

---

## 8) 자동화 스크립트(간단 샘플)

> 크론/스케줄러로 주기 실행하여 CSV로 떨구기(개념).

```sql
-- SQL*Plus
SET HEADING OFF FEEDBACK OFF PAGES 0 LINESIZE 200
SPOOL sysstat_&yyyymmdd._&hh24miss..csv

WITH d AS (
  SELECT name, value FROM v$sysstat
),
m AS (
  SELECT
    MAX(CASE WHEN name='session logical reads' THEN value END) lr,
    MAX(CASE WHEN name='physical reads'        THEN value END) pr,
    MAX(CASE WHEN name='parse count (total)'   THEN value END) ptotal,
    MAX(CASE WHEN name='parse count (hard)'    THEN value END) phard,
    MAX(CASE WHEN name='sorts (memory)'        THEN value END) s_mem,
    MAX(CASE WHEN name='sorts (disk)'          THEN value END) s_disk,
    MAX(CASE WHEN name='redo size'             THEN value END) redo_b,
    MAX(CASE WHEN name='user commits'          THEN value END) commits,
    MAX(CASE WHEN name='user rollbacks'        THEN value END) rollbacks
  FROM d
)
SELECT TO_CHAR(SYSDATE,'YYYY-MM-DD HH24:MI:SS')||','||
       lr||','||pr||','||ptotal||','||phard||','||s_mem||','||s_disk||','||
       redo_b||','||commits||','||rollbacks
FROM m;

SPOOL OFF
```

> 수집물은 나중에 **델타 처리**(이전 행과 차이)를 해 Ratio 시계열로 만든다(스프레드시트/파이썬 등).

---

## 9) 보완 지표(교차 확인 추천)

- `V$SYSTEM_EVENT` / `V$SESSION`(EVENT, BLOCKING_SESSION) → **Top Waits**  
- `V$SYS_TIME_MODEL` → **DB CPU vs Elapsed**  
- `V$FILESTAT`, `V$TEMPSEG_USAGE` → **I/O 파일/Temp 사용**  
- `V$SQLAREA`, `V$SQL`, `V$SQL_MONITOR` → **핫 SQL**  
- `DBA_HIST_SYSSTAT`(AWR) → **과거 구간** 비교(프로덕션 권장)

---

## 10) 함정 & 베스트 프랙티스 요약

**함정**  
- Ratio는 **맥락 없이 절대값**이 아니다. **절대치/Top Wait/Top SQL** 을 같이 보라.  
- BCHR만 보고 “디스크 문제”로 결론 내지 말 것.  
- 샘플링 간격이 너무 길면 **스파이크**를 놓친다. 너무 짧으면 **노이즈**가 커진다.

**베스트 프랙티스**  
1) **델타 + per-sec** 로 정규화.  
2) **Ratio 다발**을 한 화면에: BCHR, SoftParse%, SortInMem%, Exec/Parse, Logical/Physical, Redo/Tx.  
3) **해석 트리** 를 습관화:  
   - 느리다 → **Top Wait** 무엇? → `sysstat`로 mix 확인 → **문제 SQL** → **실행계획/행통계**.  
4) **전/후 검증**을 자동화(같은 부하/같은 시간대).

---

## 11) 미니 실습 시나리오

### 11.1 정렬 스필 개선 효과 보기
1) 60초 구간 스냅 A → 쿼리(대형 ORDER BY) 수행 → 스냅 B  
2) `sort_in_mem_ratio`와 `physical reads`/sec 비교  
3) 인덱스 추가(정렬 회피) 또는 `PGA` 상향 → 동일 부하 재측정  
4) Ratio↑, 물리 I/O↓ 확인

### 11.2 커서 재사용 개선
1) 초기 SoftParse% 낮음, Exec/Parse 낮음  
2) 바인드 도입 및 `session_cached_cursors` 조정 → 동일 부하 재측정  
3) SoftParse%↑, Exec/Parse↑, `parse count (hard)` 델타↓ 확인

---

## 12) 수학 한 줄(개념 정리)

- **초당 값**:  
  $$ \text{per\_sec} = \frac{\Delta \text{value}}{\Delta t\ (\text{sec})} $$
- **BCHR 근사**:  
  $$ \text{BCHR} \approx 1 - \frac{\Delta \text{physical reads}}{\Delta \text{session logical reads}} $$
- **SoftParse%**:  
  $$ \text{SoftParse\%} = 1 - \frac{\Delta \text{parse hard}}{\Delta \text{parse total}} $$
- **Sort In-Memory%**:  
  $$ \frac{\Delta \text{sorts (memory)}}{\Delta \text{sorts (memory)}+\Delta \text{sorts (disk)}} $$

---

## 13) 결론

- `V$SYSSTAT`는 **시스템 전역의 “맥박”** 이다.  
- **누적→델타→시간정규화** 로 **구간 활동**을 만들고, **Ratio** 로 **상대 효율**을 읽되, **절대치와 대기/핫SQL**로 **현상→원인**을 닫아라.  
- 최종 진단은 **DBMS_XPLAN(ALLSTATS)**, **SQL Monitor**, **ASH/AWR** 와 **교차 확인**할 때 정확해진다.

> 한 줄 정리  
> **V$SYSSTAT = 숫자 창고**. 델타로 꺼내고, 초당으로 정리하고, Ratio로 색칠하되, **대기/플랜**으로 증명하라.
