---
layout: post
title: DB 심화 - Sort Area 크기 조정
date: 2025-11-24 15:25:23 +0900
category: DB 심화
---
# Sort Area 크기 조정 — Oracle PGA 메모리 관리 방식 & AUTO 모드의 크기 결정 원리

> 목표  
> - Oracle의 **정렬(SORT)·해시(HASH) 작업영역(workarea)** 이 PGA에서 **어떻게 배정·회수·스필(spill)** 되는지 “실무에서 관측 가능한 모델”로 정리한다.  
> - `WORKAREA_SIZE_POLICY=AUTO`에서 **Global Memory Bound(GMB)** 가 어떻게 변하고(동시성·PX·AMM 영향), 각 workarea가 **ExpectedOptimal / ExpectedOnePass** 대비 어느 등급(Optimal/One-pass/Multi-pass)으로 수행되는지 **실측으로 판단**할 수 있게 한다.  
> - **정렬 자체를 회피/축소하는 SQL 재작성**과, 불가피할 때의 **PGA 타깃 조정/병렬·동시성 제어** 순서로 튜닝하는 “정석 절차”를 제공한다.

---

## 0) 핵심 요약 — 먼저 큰 그림

- Oracle의 정렬/해시 등 **workarea** 는 **PGA**에서 **동적으로 할당·반납**된다.
- `WORKAREA_SIZE_POLICY=AUTO`일 때, **개별 workarea가 받는 실제 메모리(Grant)** 는 다음의 **최소값**으로 결정된다.

  $$\boxed{\textbf{Grant}=\min(\textit{ExpectedOptimalSize},\ \textit{GlobalMemoryBound})}$$

  - **ExpectedOptimalSize(EOS)**: 옵티마이저/실행 엔진이 추정한 “**디스크 스필 없이 메모리만으로 끝낼 이상적 크기**”
  - **GlobalMemoryBound(GMB)**: **동시 활성 workarea 수요**와 **PGA 예산(`PGA_AGGREGATE_TARGET`, `PGA_AGGREGATE_LIMIT`)**을 반영한 “**workarea 1개당 전역 상한**”
- 실행 등급은 **Grant가 기대 임계치들을 넘는지**로 판정된다.

  ```text
  IF Grant >= ExpectedOptimalSize      → Optimal (메모리 내 정렬/해시)
  ELSE IF Grant >= ExpectedOnepassSize → One-pass (1회 스필)
  ELSE                                 → Multi-pass (여러 번 스필/리로드)
  ```

- GMB의 **정확한 내부 공식은 비공개**이며 버전·부하·히든 파라미터에 따라 달라진다.  
  실무에서는  
  1) `V$PGASTAT.global memory bound`로 **현재 GMB를 관측**하고  
  2) `V$SQL_WORKAREA_ACTIVE`에서 **EXPECTED_* vs LAST_MEMORY_USED**를 비교해  
  **One-pass/Multi-pass 위험을 증거 기반으로 판단**한다.

---

## 1) 작업영역(workarea)과 Sort Area의 정체

### 1.1 workarea가 쓰이는 연산들

workarea는 “정렬 메모리”라는 좁은 의미를 넘어 **PGA 기반의 모든 대형 중간 연산 버퍼**다. 대표적으로:

- **SORT 계열**
  - `SORT ORDER BY`, `SORT GROUP BY`, `SORT UNIQUE`
  - `WINDOW SORT` (분석 함수, `SUM OVER`, `ROW_NUMBER`, `RANK` 등)
  - `MERGE JOIN` 전 정렬
- **HASH 계열**
  - `HASH JOIN` 의 빌드(build) 테이블 해시 테이블
  - `HASH GROUP BY` 의 해시 버킷/집계 테이블
  - `DISTINCT`의 해시 집합
- **기타**
  - Bitmap merge/creation, 일부 XML/JSON 처리, 병렬 PX 교환 버퍼 일부 등

즉, **Sort Area를 키우면 정렬뿐 아니라 해시 조인/집계도 함께 안정화**된다.

---

### 1.2 정렬/해시 관점의 3단계 처리 모델

Oracle은 Workarea 크기에 따라 **입력 데이터의 처리 단계가 바뀐다.**

- **Optimal**
  - 모든 작업이 **메모리에서 끝남**
  - 디스크 임시(TEMP) I/O 없음
- **One-pass**
  - 입력을 1회 읽으며 **한 번만 스필**하고, 스필 파일을 1회 재읽어 종료
  - TEMP I/O는 있으나 “제어 가능한 수준”
- **Multi-pass**
  - 스필/리로드가 여러 번
  - TEMP 랜덤 I/O 폭증, CPI↑, 응답시간 “급락”

**현업 목표는 “Multi-pass 근절, One-pass 최소화, Optimal 최대화”**다.

---

## 2) PGA 메모리 관리 방식: MANUAL vs AUTO vs AMM

### 2.1 MANUAL

- 파라미터
  ```sql
  WORKAREA_SIZE_POLICY = MANUAL
  SORT_AREA_SIZE
  HASH_AREA_SIZE
  ```
- **세션/시스템이 직접 할당 크기를 고정**한다.
- 병렬(PX), 동시 접속, 혼합 워크로드에서
  - 어느 세션이 언제 workarea를 쓰는지 예측이 불가
  - 고정값이 과대면 PGA 폭주, 과소면 Multi-pass 연쇄
- 현대 버전(12c+)에선 **특수한 레거시/버그 회피 외 실무 비권장**.

---

### 2.2 AUTO (권장 표준)

- 파라미터
  ```sql
  WORKAREA_SIZE_POLICY = AUTO
  PGA_AGGREGATE_TARGET
  PGA_AGGREGATE_LIMIT
  ```
- Oracle이
  1) **현재 동시 활성 workarea 수**를 추적하고  
  2) PGA 예산을 고려해 **GMB를 산출**하고  
  3) 각 workarea에 `Grant = min(EOS, GMB)` 를 배정한다.
- **동시성에 따라 GMB가 계속 변한다.**  
  그래서 “같은 SQL도 시간대/부하에 따라 One-pass로 떨어질 수 있음”을 항상 염두에 둬야 한다.

---

### 2.3 AMM(Automatic Memory Management)

- 파라미터
  ```sql
  MEMORY_TARGET
  MEMORY_MAX_TARGET
  ```
- SGA + PGA 총량을 **한 풀(pool)** 로 보고 자동 분배한다.
- 이때
  - `PGA_AGGREGATE_TARGET`은 실질적 타깃이 아니라 **조언치(advisory)** 에 가까워짐
  - SGA가 급히 필요하면 PGA가 줄어들 수 있고, 반대도 가능
- 실무에선
  - **ASMM(SGA_TARGET) + Auto PGA(PGA_AGGREGATE_TARGET)** 조합을 더 선호하는 조직도 많다.
  - AMM을 쓰면 **PGA 변동 폭이 크다**는 전제로 모니터링 강도를 높여야 한다.

---

### 2.4 권장 설정 예

```sql
-- 권장 표준
ALTER SYSTEM SET workarea_size_policy = AUTO;
ALTER SYSTEM SET pga_aggregate_target = 8G;
ALTER SYSTEM SET pga_aggregate_limit  = 16G;

-- (레거시/특수 사유에 한해)
ALTER SYSTEM SET workarea_size_policy = MANUAL;
ALTER SYSTEM SET sort_area_size = 128M;
ALTER SYSTEM SET hash_area_size = 128M;
```

---

## 3) AUTO 모드에서 크기 결정 원리 — “준-공식”과 실측 모델

### 3.1 전역 상한(Global Memory Bound, GMB)의 의미

GMB는 “**workarea 1개가 가질 수 있는 현재 최대 허용치**”다.

- PGA 예산이 넉넉하고 workarea가 적게 돌면 GMB가 크다 → Optimal↑
- 동시 workarea가 급증하거나 PGA가 모자라면 GMB가 줄어든다 → One/Multi↑

실무적 개념 모델:

$$ \text{GMB} \approx \min\Big(\text{PerWorkareaCap},\ \frac{\text{AvailablePGA}}{\text{ActiveWorkareas}}\Big) $$

- **PerWorkareaCap**: 버전/플랫폼이 정한 1개 workarea 상한  
  (히든 파라미터, 내부 safeguard)
- **AvailablePGA**: 타깃 주변에서 **실제로 가용한 PGA**  
  (`PGA_AGGREGATE_TARGET`, 최근 사용량, Limit, 압박 신호를 반영)
- **ActiveWorkareas**: 현재 동시 활성 정렬/해시 작업 영역의 개수

> 내부 로직은 더 복잡하지만, 이 모델만으로도  
> **“왜 지금 One-pass가 늘었는가?”** 를 사실상 설명할 수 있다.

---

### 3.2 개별 workarea Grant의 결정

다시 한 번 핵심 공식:

$$ \textbf{Grant}=\min(\textit{ExpectedOptimalSize},\ \textit{GlobalMemoryBound}) $$

- EOS는 연산 유형별로 계산 방식이 다르다.
  - **SORT**: 입력행 수 × 행폭 × 정렬 키 구조 × 계수  
  - **HASH JOIN**: 빌드 입력의 행 수·행폭·해시 버킷 추정  
  - **HASH GROUP BY**: 그룹 NDV 추정·엔트리 크기 × 계수
- GMB보다 EOS가 크면 EOS가 **잘려서(Clamp)** One/Multi 위험이 생김.

---

### 3.3 ExpectedOnePassSize(E1S)란?

EOS는 “완전 메모리 처리” 임계치지만, Oracle은  
**“최소 One-pass로 끝낼 수 있는 크기(E1S)”**도 별도로 계산한다.

- `Grant >= E1S` 이면 **One-pass 보장 가능성↑**
- `Grant < E1S` 이면 **Multi-pass 거의 확정**

따라서 실무 판단은:

1) EOS vs Grant → Optimal 가능?
2) E1S vs Grant → 최소 One-pass는 가능?

---

### 3.4 관측 포인트(증거 기반 튜닝)

다음 뷰/통계는 “Oracle이 내부에서 계산한 값 + 실제 배정 결과”를 전부 노출한다.

- `V$PGASTAT`
  - `global memory bound`
  - `total PGA allocated / inuse`
  - `over allocation count` 등
- `V$SQL_WORKAREA_ACTIVE`
  - 현재 돌고 있는 workarea별  
    `EXPECTED_OPTIMAL_SIZE`, `EXPECTED_ONEPASS_SIZE`, `LAST_MEMORY_USED`
- `V$SQL_WORKAREA`
  - 완료된 작업의 과거 실행 결과(Optimal/One/Multi 카운트)
- `V$PGA_TARGET_ADVICE[_HISTOGRAM]`
  - 타깃 상향 시 Optimal 비율/스필 감소 예측
- `V$SYSSTAT`, `V$MYSTAT`
  - `workarea executions - optimal/onepass/multipass`
  - `sorts (memory/disk)`

---

### 3.5 실측 SQL 모음

```sql
-- (1) 전역상태: GMB 및 PGA 압박 확인
SELECT name, value
FROM   v$pgastat
WHERE  name IN (
  'total PGA allocated',
  'total PGA inuse',
  'global memory bound',
  'over allocation count',
  'pga aggregate target',
  'pga aggregate limit'
);

-- (2) 현재 활성 workarea의 기대치 vs 실제 배정
SELECT sid, operation_type, policy,
       expected_optimal_size/1024/1024  AS exp_opt_mb,
       expected_onepass_size/1024/1024  AS exp_one_mb,
       last_memory_used/1024/1024       AS last_mb,
       active_time/100                  AS active_sec
FROM   v$sql_workarea_active
ORDER  BY active_time DESC;

-- (3) 완료된 workarea의 실행등급 통계(과거)
SELECT operation_type,
       optimal_executions,
       onepass_executions,
       multipasses,
       last_degree
FROM   v$sql_workarea
ORDER BY (optimal_executions + onepass_executions + multipasses) DESC;

-- (4) 세션별 sort/워크에어리어 통계
SELECT sn.name, ms.value
FROM   v$mystat ms JOIN v$statname sn ON sn.stat# = ms.stat#
WHERE  sn.name IN (
  'workarea executions - optimal',
  'workarea executions - onepass',
  'workarea executions - multipass',
  'sorts (memory)',
  'sorts (disk)'
);
```

---

## 4) 실습 1 — 큰 SORT로 One-pass ↔ Optimal 전환 관측

### 4.1 준비: 큰 정렬 대상 테이블

> 환경에 따라 행 수를 조절한다.  
> 핵심은 **정렬 키가 충분히 커서 EOS/E1S가 의미 있게 나오도록** 만드는 것.

```sql
DROP TABLE t_big PURGE;

CREATE TABLE t_big AS
SELECT rownum AS id,
       TO_CHAR(5000000-rownum, 'FM0000000') AS k, -- 역순 키
       RPAD('x', 100, 'x') AS payload
FROM   dual
CONNECT BY LEVEL <= 2000000;

EXEC DBMS_STATS.GATHER_TABLE_STATS(USER, 'T_BIG');
```

---

### 4.2 케이스 A: 낮은 PGA 타깃 → One-pass/Multi 유발

```sql
ALTER SYSTEM SET workarea_size_policy = AUTO;
ALTER SYSTEM SET pga_aggregate_target = 512M;

-- 실행 전 통계
SELECT sn.name, ms.value
FROM   v$mystat ms JOIN v$statname sn ON sn.stat#=ms.stat#
WHERE  sn.name IN (
  'workarea executions - optimal',
  'workarea executions - onepass',
  'workarea executions - multipass',
  'sorts (memory)',
  'sorts (disk)'
);

-- 대형 정렬
SELECT /*+ MONITOR */
       k, COUNT(*)
FROM   t_big
GROUP  BY k
ORDER  BY k;

-- 실행 직후 관측
SELECT name, value FROM v$pgastat
WHERE name IN ('global memory bound','total PGA inuse','pga aggregate target');

SELECT operation_type,
       expected_optimal_size/1024/1024 exp_opt_mb,
       expected_onepass_size /1024/1024 exp_one_mb,
       last_memory_used       /1024/1024 last_mb
FROM   v$sql_workarea
WHERE  operation_type LIKE 'SORT%'
ORDER  BY last_memory_used DESC;

-- 실행 후 통계
SELECT sn.name, ms.value
FROM   v$mystat ms JOIN v$statname sn ON sn.stat#=ms.stat#
WHERE  sn.name IN (
  'workarea executions - optimal',
  'workarea executions - onepass',
  'workarea executions - multipass',
  'sorts (memory)',
  'sorts (disk)'
);
```

**기대 관찰**
- `global memory bound` 감소
- `exp_opt_mb > last_mb >= exp_one_mb` → One-pass 가능성↑
- `sorts (disk)` 증가, One-pass 카운트 증가

---

### 4.3 케이스 B: 높은 PGA 타깃 → Optimal 유도

```sql
ALTER SYSTEM SET pga_aggregate_target = 4G;

SELECT /*+ MONITOR */
       k, COUNT(*)
FROM   t_big
GROUP  BY k
ORDER  BY k;

SELECT name, value FROM v$pgastat
WHERE name IN ('global memory bound','total PGA inuse','pga aggregate target');

SELECT operation_type,
       expected_optimal_size/1024/1024 exp_opt_mb,
       expected_onepass_size /1024/1024 exp_one_mb,
       last_memory_used       /1024/1024 last_mb
FROM   v$sql_workarea
WHERE  operation_type LIKE 'SORT%'
ORDER  BY last_memory_used DESC;
```

**기대 관찰**
- GMB 상승으로 `Grant >= EOS`
- Optimal 비율 증가, 디스크 스필 0에 근접

---

## 5) 실습 2 — HASH JOIN에서의 스필 관측

정렬만이 아니라 해시에서도 동일 원리가 적용된다.

### 5.1 준비: 큰 조인 대상

```sql
DROP TABLE t_build PURGE;
DROP TABLE t_probe PURGE;

CREATE TABLE t_build AS
SELECT rownum id,
       MOD(rownum, 100000) k,
       RPAD('b', 200, 'b') payload
FROM dual CONNECT BY LEVEL <= 800000;

CREATE TABLE t_probe AS
SELECT rownum id,
       MOD(rownum, 100000) k,
       RPAD('p', 50, 'p') payload
FROM dual CONNECT BY LEVEL <= 2000000;

EXEC DBMS_STATS.GATHER_TABLE_STATS(USER, 'T_BUILD');
EXEC DBMS_STATS.GATHER_TABLE_STATS(USER, 'T_PROBE');
```

---

### 5.2 낮은 PGA에서 해시 One/Multi 발생

```sql
ALTER SYSTEM SET pga_aggregate_target = 512M;

SELECT /*+ MONITOR USE_HASH(b) */
       COUNT(*)
FROM   t_probe p
JOIN   t_build b ON b.k = p.k;

-- 해시 워크에어리어 관측
SELECT sid, operation_type,
       expected_optimal_size/1024/1024 exp_opt_mb,
       expected_onepass_size /1024/1024 exp_one_mb,
       last_memory_used      /1024/1024 last_mb
FROM   v$sql_workarea_active
WHERE  operation_type LIKE 'HASH%'
ORDER BY last_memory_used DESC;
```

**관측 논리**
- 해시 빌드가 EOS에 못 미치면 One/Multi 스필  
- TEMP 사용량이 늘고, CPU 대비 I/O 비중이 급상승한다.

---

### 5.3 높은 PGA에서 Optimal 유도

```sql
ALTER SYSTEM SET pga_aggregate_target = 4G;

SELECT /*+ MONITOR USE_HASH(b) */
       COUNT(*)
FROM   t_probe p
JOIN   t_build b ON b.k = p.k;

SELECT operation_type,
       optimal_executions, onepass_executions, multipasses
FROM   v$sql_workarea
WHERE  operation_type LIKE 'HASH%';
```

---

## 6) 동시성이 GMB를 깎는 방식 — “왜 밤엔 빠른데 낮엔 느린가?”

### 6.1 GMB는 동시 활성 workarea에 반비례한다

**동일 PGA 타깃이라도**, 아래 상황에서는 곧바로 GMB가 떨어진다.

- 대시보드/리포트가 동시에 몰림  
- 배치가 낮 시간에 겹침  
- 병렬 PX가 늘어 workarea 개수가 폭증  
- AMM이 SGA로 메모리를 가져감

즉:

$$ \text{ActiveWorkareas} \uparrow \Rightarrow \text{GMB} \downarrow \Rightarrow \text{One/Multi} \uparrow $$

---

### 6.2 실습: 두 세션 동시 정렬로 GMB 하락 관측

**세션 1**

```sql
ALTER SESSION SET workarea_size_policy = AUTO;

SELECT /*+ MONITOR */
       k, COUNT(*)
FROM   t_big
GROUP  BY k
ORDER  BY k;
```

**세션 2(동시에 실행)**

```sql
SELECT /*+ MONITOR */
       k, COUNT(*)
FROM   t_big
GROUP  BY k
ORDER  BY k;
```

**제3 세션에서 관측**

```sql
SELECT name, value
FROM   v$pgastat
WHERE  name = 'global memory bound';

SELECT sid, operation_type,
       expected_optimal_size/1024/1024 exp_opt_mb,
       last_memory_used/1024/1024 last_mb
FROM   v$sql_workarea_active
ORDER BY last_memory_used DESC;
```

**기대**
- 단일 실행보다 **GMB가 즉시 하락**
- 각 세션의 Last_MB가 EOS보다 작아져 One-pass로 떨어질 수 있음

---

## 7) 병렬(PX)과 workarea — “DOP는 메모리를 곱하기로 먹는다”

### 7.1 PX는 workarea를 세션 수만큼 복제한다

예를 들어:
- DOP=8인 해시 조인은  
  **빌드 해시 테이블이 8개로 분산**될 수 있으며
- 정렬도 각 슬레이브별로 **로컬 sort run을 만든 뒤 merge**한다.

따라서 PX에서는:

$$ \text{ActiveWorkareas} \approx \text{DOP} \times \text{연산 수} $$

결과:
- DOP가 높아질수록 GMB가 깎여 One/Multi 위험이 커진다.

---

### 7.2 PX에서의 실무적 처방

- DOP를 무작정 키우지 말고 **workarea 스필 여부를 먼저 확인**
- PX가 필요하면
  - **PGA 타깃을 PX 워크로드 기준으로 산정**
  - **병렬 동시 개수**(Resource Manager, Job 스케줄) 제어
- 특정 보고 SQL만 PX 허용, 나머지 OLTP는 PX 제한이 흔한 운영 패턴

---

## 8) “메모리로 해결하기 전에” 정렬/해시 자체를 줄이는 SQL 재작성

### 8.1 정렬 회피가 최고의 튜닝

정렬 I/O를 줄이는 방법은 “메모리 확대”보다 **항상 우선**이다.

#### 8.1.1 인덱스로 ORDER BY 제거

```sql
-- amount DESC 정렬을 인덱스가 대체
CREATE INDEX ix_sales_amt_desc ON f_sales(amount DESC);

SELECT /*+ INDEX_DESC(f_sales ix_sales_amt_desc) */
       sales_id, amount
FROM   f_sales
WHERE  sales_dt BETWEEN :lo AND :hi
FETCH FIRST 200 ROWS ONLY;  -- Stopkey
```

- Sort Area가 아니라 **Stopkey + 인덱스 역순 스캔**이 해결
- KPI/Top-N 대시보드에 가장 강력

---

#### 8.1.2 “정렬 후 가공” 대신 “가공 후 정렬”

행폭을 줄이면 EOS가 줄어든다.

```sql
WITH k_only AS (
  SELECT /*+ MATERIALIZE */
         key_col
  FROM   big_fact
  WHERE  cond = :b1
)
SELECT key_col
FROM   k_only
ORDER  BY key_col;
```

- 정렬 대상의 **row width 축소** → ExpectedOptimalSize 감소
- 특히 Wide row(LOB/JSON/대형 문자열)가 포함된 정렬에 치명적 효과

---

#### 8.1.3 GROUP BY에서 SORT 대신 HASH 유도

```sql
SELECT /*+ USE_HASH_AGGREGATION */
       k, SUM(amount)
FROM   big_fact
GROUP  BY k;
```

- 해시 집계가 가능한 구조라면 SORT GROUP BY를 피할 수 있다.
- 단, NDV가 매우 크면 해시도 EOS가 커질 수 있으므로 workarea 관측 필요.

---

### 8.2 해시 조인 빌드 입력을 줄여 스필 제거

해시 조인의 EOS는 **빌드 입력 크기**에 선형 비례한다.

- **드라이빙/필터 강한 테이블을 빌드로** 만들면 EOS가 줄어든다.
- 빌드가 작으면 One/Multi 위험이 급감.

예:

```sql
SELECT /*+ LEADING(small) USE_HASH(big) */
       ...
FROM   small_dim small
JOIN   big_fact  big ON big.k = small.k
WHERE  small.flag = 'Y';
```

---

## 9) PGA 타깃 산정/증설의 실무 절차

### 9.1 먼저 “스필이 실제로 병목인지” 확인

- One-pass가 약간 있다고 해서 무조건 늘릴 필요는 없다.
- Multi-pass가 **지속적/대량**이면 거의 확실히 PGA 병목.

확인 순서:

1) `V$SQL_WORKAREA`에서 One/Multi 비율 확인  
2) `TEMP` I/O 증가(`sorts (disk)`, AWR TEMP read/write) 확인  
3) SQL 응답시간과 TEMP burst가 상관되는지 확인

---

### 9.2 `V$PGA_TARGET_ADVICE`로 증설 효과 예측

```sql
SELECT pga_target_for_estimate/1024/1024 AS target_mb,
       estd_extra_bytes_rw/1024/1024     AS extra_io_mb,
       estd_pga_cache_hit_percentage     AS hit_pct
FROM   v$pga_target_advice
ORDER  BY target_mb;
```

해석:

- **hit_pct가 깔끔히 올라가고 extra_io가 줄어드는 구간**이 증설의 “효과 구간”
- hit_pct가 거의 안 오르면  
  → SQL 재작성/카디널리티 개선이 먼저

---

### 9.3 타깃과 리밋을 함께 설계

- `PGA_AGGREGATE_TARGET` = **평상시 운영 타깃**
- `PGA_AGGREGATE_LIMIT` = **폭주 시 보호선**

실무 관측:

- 타깃에 근접하자 `over allocation count`가 자주 증가하면
  - 타깃 증설
  - 또는 동시 workarea를 줄이는 운영 스케줄 조정 필요

---

## 10) 특수 케이스: MANUAL을 고려할 수 있는 상황

일반적으로 비권장이지만, 다음 상황에 “일시적으로” 고려될 수 있다.

- 레거시 버전/특정 버그로 Auto PGA가 비정상 동작
- 배치 전용 인스턴스에서 **항상 동일한 정렬/해시 부하**가 재현되는 경우
- 시스템 정책상 PGA 타깃을 크게 키우기 곤란한데,  
  특정 세션만 sort/hashing이 반드시 Optimal이어야 하는 경우

이 때도 원칙은:
- 전체를 MANUAL로 고정하지 말고
- 가능하면 **배치 전용 인스턴스 / 리소스 그룹 / 서비스 분리**로 해결

---

## 11) 운영에서 자주 마주치는 증상 ↔ 원인 ↔ 처방

| 증상 | 원인(관측) | 처방(우선순위) |
|---|---|---|
| 낮 시간 One/Multi 급증 | 동시 리포트·PX 증가로 GMB 하락 | (1) 스케줄 분산 (2) DOP 제한 (3) PGA 증설 |
| 특정 SQL만 One/Multi | EOS가 과도(행폭/정렬량) | (1) 정렬 회피/행폭 축소/Stopkey (2) 빌드 입력 축소 |
| HASH JOIN만 스필 | 빌드 입력 과대/순서 불량 | (1) 조인 순서 재작성 (2) 힌트 최소 고정 |
| TEMP 폭주 | Multi-pass 연쇄 | (1) Multi 근절이 목표 (2) PGA+동시성 동시 조정 |
| AMM에서 변동성 큼 | SGA↔PGA 재분배 | (1) AMM 정책 점검 (2) 변동성 전제 모니터링 |

---

## 12) 최종 체크리스트(현장용)

- [ ] **AUTO 정책**인가? (`WORKAREA_SIZE_POLICY=AUTO`)
- [ ] `global memory bound`를 **직접 보고 있는가**
- [ ] One-pass/Multi-pass가 **언제(시간대/부하/배치)에 늘어나는지** 파악했는가
- [ ] **정렬/해시 자체를 줄이는 SQL 재작성**을 먼저 했는가
- [ ] 해시/정렬의 **빌드 입력·행폭·정렬량**을 줄였는가
- [ ] `V$PGA_TARGET_ADVICE`로 **증설 효과 구간**을 확인했는가
- [ ] PX(DOP)와 동시성 스케줄이 **GMB를 깎지 않도록 제어**했는가
- [ ] 타깃/리밋이 **폭주와 안정성**을 동시에 만족하는가
- [ ] 마지막으로 `WORKAREA`/`SYSSTAT`로 **실측 개선**을 확인했는가

---

## 결론

- AUTO 모드에서 workarea 메모리는  
  **“필요한 이상적 크기(EOS)”와 “시스템 전역 상한(GMB)”의 최소값**으로 배정된다.
- GMB는 **동시성·PX·총 PGA 가용량**에 의해 **실시간으로 변한다.**  
  그래서 “하드웨어만큼이나 운영 패턴(동시 실행 구조)이 성능을 결정한다.”
- 튜닝의 정석은

  1) **정렬/해시 자체를 줄이는 SQL 재작성**  
  2) **빌드 입력·행폭·조인 순서 개선**  
  3) 그래도 Multi-pass가 남으면 **PGA 타깃 증설 + 동시성/PX 제어**  
  4) `ALLSTATS + IOSTATS + MEMSTATS`와 `WORKAREA` 뷰로 **팩트 기반 검증**

이 순서를 지키면 Sort Area/PGA 튜닝은 “감(感)”이 아니라 **증거 기반 엔지니어링**이 된다.