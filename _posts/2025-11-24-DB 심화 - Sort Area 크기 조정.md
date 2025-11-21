---
layout: post
title: DB 심화 - Sort Area 크기 조정
date: 2025-11-24 15:25:23 +0900
category: DB 심화
---
# Sort Area 크기 조정 : PGA 메모리 관리 방식 & AUTO 모드의 크기 결정 원리

## 핵심 요약(먼저 큰 그림)

- **정렬/해시 작업영역(workarea)** 는 PGA에서 **동적으로** 배정된다.
- `WORKAREA_SIZE_POLICY=AUTO`(권장)일 때 **개별 workarea**가 받는 메모리 =
  $$\boxed{\textbf{Grant}=\min(\textit{ExpectedOptimalSize},\ \textit{GlobalMemoryBound})}$$
  - **ExpectedOptimalSize**: 옵티마이저/실행 중 추정한 “**디스크 I/O 없이 끝낼**” 이상적 크기
  - **GlobalMemoryBound**: 시스템 전체의 **수요**(동시 활성 workarea 개수)와 **예산**(`PGA_AGGREGATE_TARGET`, `PGA_AGGREGATE_LIMIT`)을 반영한 **전역 상한**
- Grant가 **Optimal ≥** 이면 **메모리 정렬/해시(디스크 I/O 없음)**,
  **One-pass ≥** 이면 **한 번의 디스크 스필**로 처리,
  그보다 작으면 **Multi-pass**(여러 번 스필/리로드)로 처리된다.
- **정확한 공식은 비공개(내부 알고리즘)**지만, 실무에선 **전역 상한을 관측**(`V$PGASTAT.global memory bound`)하고 **활성 workarea** 상태(`V$SQL_WORKAREA_ACTIVE`)로 **실제 배정량/예상 임계치**를 확인해 **의사결정을 내린다.**

---

## PGA 메모리 관리 방식 선택

### vs 자동(AUTO) vs AMM(메모리 타깃 기반)

- **MANUAL**
  - 파라미터: `WORKAREA_SIZE_POLICY=MANUAL`
  - 세션/시스템 단위로 `SORT_AREA_SIZE`, `HASH_AREA_SIZE`(및 `_..._AREA_SIZE` 계열)로 **작업영역 크기 직접 지정**
  - 병렬(PX)까지 고려하면 관리가 어려움, 현대 버전에서는 **권장하지 않음**
- **AUTO (권장)**
  - `WORKAREA_SIZE_POLICY=AUTO`, 예산은 **`PGA_AGGREGATE_TARGET`**
  - Oracle이 동시 세션/작업영역 수요를 기반으로 **전역 상한(글로벌 바운드)**를 주기적으로 조정
  - 각 workarea는 **예상 최적 크기**와 **글로벌 바운드** 중 **작은 값**을 부여받음
- **AMM(Automatic Memory Management)**
  - `MEMORY_TARGET` 사용 시 **SGA/PGA 총량을 동적 분배**
  - 이때 `PGA_AGGREGATE_TARGET`은 **조언치(advisory target)** 성격, 실제 배정은 AMM의 균형에 좌우
  - ASMM(SGA_TARGET) + Auto PGA(별도 `PGA_AGGREGATE_TARGET`) 조합을 현업에서 더 선호하기도 함

> 기본 권장:
> - **`WORKAREA_SIZE_POLICY=AUTO` + (ASMM) `SGA_TARGET` + `PGA_AGGREGATE_TARGET`**
> - 또는 **AMM**(`MEMORY_TARGET`)을 사용할 땐 PGA 변동이 더 가변적임을 이해하고 모니터링 강화.

설정 예:
```sql
-- 권장: 자동 PGA 메모리 관리
ALTER SYSTEM SET workarea_size_policy = AUTO;
ALTER SYSTEM SET pga_aggregate_target = 8G;      -- 환경/동접에 맞춰 조정
ALTER SYSTEM SET pga_aggregate_limit  = 16G;     -- 상한(초과 시 ORA-4036 등 보호)

-- (참고) 수동 모드
ALTER SYSTEM SET workarea_size_policy = MANUAL;
ALTER SYSTEM SET sort_area_size = 128M;          -- 구버전 호환/특수 케이스 외 비권장
ALTER SYSTEM SET hash_area_size = 128M;
```

---

## AUTO 모드에서 크기 결정 원리(“준-공식”)

### 개념 모델(실무 친화적 수식)

- **전역 상한(Global Memory Bound; GMB)**:
  전체 PGA 예산과 동시 **활성 workarea 수**(정렬·해시·윈도우 등)를 고려해 계산되는 **1개 workarea당 최대 허용치**
  $$ \text{GMB} \approx \min\Big(\text{_smm\_max\_size},\ \frac{\text{TargetPGA(가용)}}{\text{ActiveWorkareas}}\Big) $$
  - `TargetPGA(가용)`: `PGA_AGGREGATE_TARGET` 근처에서 **이미 사용 중인 메모리**, 최근 히스토리, `PGA_AGGREGATE_LIMIT` 등 **가용성**을 반영
  - `_smm_max_size`: **workarea 1개가 가질 수 있는 상한**(버전/플랫폼/패치에 따라 다름, 히든 파라미터)
  - **정확한 수식은 비공개**이므로 `V$PGASTAT.global memory bound`로 **결과값을 관측**한다.

- **개별 workarea Grant**:
  $$ \textbf{Grant}=\min(\textit{ExpectedOptimalSize},\ \textit{GlobalMemoryBound}) $$
  - `ExpectedOptimalSize` : **입력 크기·카디널리티/분포**로 추정한 **디스크 스필 없이 끝낼** 메모리(작업별 계산식 상이)
  - Grant가 `ExpectedOnePassSize` 이상이면 **One-pass** 이상을 확보
  - 그 미만이면 **Multipass**로 예상(여러 번 스필)

- **실제 판단 로직(관측 가능한 조건)**
  ```text
  IF Grant >= ExpectedOptimalSize      → Optimal (메모리 내 처리)
  ELSE IF Grant >= ExpectedOnepassSize → One-pass (1회 스필)
  ELSE                                 → Multi-pass (다중 스필)
  ```

### 관측 포인트(증거 기반 튜닝)

- `V$PGASTAT.global memory bound` : **현재 GMB**(KB)
- `V$SQL_WORKAREA_ACTIVE` : **활성 workarea별**
  - `EXPECTED_OPTIMAL_SIZE`, `EXPECTED_ONEPASS_SIZE`, `EXPECTED_MULTIPASSES`,
    `LAST_MEMORY_USED`, `ACTIVE_TIME`, `OPERATION_TYPE(SORT/HASH GROUP BY 등)`
- `V$SQL_WORKAREA` : **완료된 작업**의 과거 기록(Optimal/One/Multi 통계)
- `V$PGA_TARGET_ADVICE`, `V$PGA_TARGET_ADVICE_HISTOGRAM` : 타깃 조정 효과 예측
- `V$MYSTAT/V$SYSSTAT` : `workarea executions - optimal / onepass / multipass`, `sorts (memory/disk)`

조회 예:
```sql
-- 전역 상한(글로벌 바운드) 확인
SELECT name, value
FROM   v$pgastat
WHERE  name IN ('total PGA allocated','total PGA inuse','global memory bound',
                'over allocation count','pga aggregate target','pga aggregate limit');

-- 활성 workarea(현재 돌고 있는 정렬/해시)의 예상 임계치와 실제 배정 확인
SELECT sid, operation_type, policy,
       expected_optimal_size/1024/1024 AS exp_opt_mb,
       expected_onepass_size /1024/1024 AS exp_one_mb,
       last_memory_used       /1024/1024 AS last_mb,
       active_time/100 AS active_sec
FROM   v$sql_workarea_active
ORDER  BY active_time DESC;

-- 완료된 작업의 통계(과거)
SELECT operation_type,
       optimal_executions, onepass_executions, multipasses,
       last_degree
FROM   v$sql_workarea
ORDER  BY (optimal_executions + onepass_executions + multipasses) DESC;

-- 세션/시스템 통계(전/후 비교)
SELECT sn.name, ms.value
FROM   v$mystat ms JOIN v$statname sn ON sn.stat# = ms.stat#
WHERE  sn.name IN ('workarea executions - optimal',
                   'workarea executions - onepass',
                   'workarea executions - multipass',
                   'sorts (memory)','sorts (disk)');
```

---

## 실습 시나리오 — 정렬 한 번으로 One-pass ↔ Optimal 만들기

> **아이디어**: `PGA_AGGREGATE_TARGET`을 다르게 설정하고 같은 **대형 정렬**을 실행,
> `V$PGASTAT.global memory bound`와 `V$SQL_WORKAREA_ACTIVE`의 **EXPECTED\_*** 대비 **실제 배정(last\_memory\_used)** 를 비교한다.

### 준비: 큰 정렬 대상 만들기(예시)

```sql
-- 실습 테이블: 약 수백만행 권장(환경에 맞게 조정)
CREATE TABLE t_big AS
SELECT /*+ NO_MERGE */
       rownum AS id,
       TO_CHAR(1000000 - rownum, 'FM000000') AS k,  -- 역순 정렬 대상
       RPAD('x', 100, 'x') AS payload
FROM   dual
CONNECT BY LEVEL <= 2000000;  -- 환경에 맞게 조정

EXEC DBMS_STATS.GATHER_TABLE_STATS(USER, 'T_BIG');
```

### 케이스 A: 작은 PGA 타깃(One-pass/Multi 유발)

```sql
ALTER SYSTEM SET workarea_size_policy = AUTO;
ALTER SYSTEM SET pga_aggregate_target = 512M;  -- 작게

-- 세션 통계 스냅샷(전)
SELECT sn.name, ms.value
FROM   v$mystat ms JOIN v$statname sn ON sn.stat#=ms.stat#
WHERE  sn.name IN ('workarea executions - optimal','workarea executions - onepass',
                   'workarea executions - multipass','sorts (memory)','sorts (disk)');

-- 대형 정렬 실행(정렬 키만 투영하면 행폭이 작아짐 → 여기선 실험 위해 일부 컬럼 포함)
SELECT /*+ MONITOR */
       k, COUNT(*)
FROM   t_big
GROUP  BY k
ORDER  BY k;  -- 정렬 유도

-- 활성/과거 workarea 관측(실행 중/실행 직후)
SELECT name, value FROM v$pgastat
WHERE name IN ('global memory bound','total PGA inuse','pga aggregate target');

SELECT operation_type,
       expected_optimal_size/1024/1024 exp_opt_mb,
       expected_onepass_size /1024/1024 exp_one_mb,
       last_memory_used       /1024/1024 last_mb
FROM   v$sql_workarea
WHERE  operation_type LIKE 'SORT%'
ORDER  BY last_memory_used DESC;

-- 세션 통계 스냅샷(후)
SELECT sn.name, ms.value
FROM   v$mystat ms JOIN v$statname sn ON sn.stat#=ms.stat#
WHERE  sn.name IN ('workarea executions - optimal','workarea executions - onepass',
                   'workarea executions - multipass','sorts (memory)','sorts (disk)');
```

**기대 관찰**
- `global memory bound`가 낮게 잡힘
- `expected_optimal_size > last_memory_used >= expected_onepass_size` → **One-pass**
- `workarea executions - onepass` 증가, `sorts (disk)`도 약간 동반 가능

### 케이스 B: 큰 PGA 타깃(Optimal 유도)

```sql
ALTER SYSTEM SET pga_aggregate_target = 4G; -- 크게

-- 동일 작업 재실행
SELECT /*+ MONITOR */
       k, COUNT(*)
FROM   t_big
GROUP  BY k
ORDER  BY k;

-- 관측
SELECT name, value FROM v$pgastat
WHERE name IN ('global memory bound','total PGA inuse','pga aggregate target');

SELECT operation_type,
       expected_optimal_size/1024/1024 exp_opt_mb,
       expected_onepass_size /1024/1024 exp_one_mb,
       last_memory_used       /1024/1024 last_mb
FROM   v$sql_workarea
WHERE  operation_type LIKE 'SORT%'
ORDER  BY last_memory_used DESC;

-- 세션 통계 비교
SELECT sn.name, ms.value
FROM   v$mystat ms JOIN v$statname sn ON sn.stat#=ms.stat#
WHERE  sn.name IN ('workarea executions - optimal','workarea executions - onepass',
                   'workarea executions - multipass','sorts (memory)','sorts (disk)');
```

**기대 관찰**
- `global memory bound`가 높아져서 **Grant ≥ ExpectedOptimalSize**
- **Optimal 실행**(디스크 스필 없음), `workarea executions - optimal` 증가

> 팁: **동시 세션이 많아져 활성 workarea 수가 늘면** 같은 타깃이라도 **GMB가 떨어져** One-pass/Multi로 쉽게 변한다.
> 즉, **최적 크기는 부하(동시성)에 의존**한다.

---

## 자동 PGA 메모리 관리 하에서의 “결정 공식”을 실무 언어로

정렬/해시 **workarea**가 배정받는 크기는 다음과 같이 **결정**된다.

1. **작업 자체의 필요 크기 추정**
   - 옵티마이저/실행 중 통계로 **입력 크기**·**카디널리티**·**키 분포** 등을 평가
   - 그 결과로 `EXPECTED_OPTIMAL_SIZE`, `EXPECTED_ONEPASS_SIZE`, `EXPECTED_MULTIPASSES` 산출

2. **전역 상한(Global Memory Bound; GMB) 계산**
   - **현재/근래의 활성 workarea 수요**를 집계
   - `PGA_AGGREGATE_TARGET`(또는 AMM 하의 가용 PGA), `PGA_AGGREGATE_LIMIT`, 히스토리·피드백을 반영
   - **히든 한도**(예: `_smm_max_size`)도 적용
   - 결과는 `V$PGASTAT.global memory bound`로 관측 가능

3. **Grant 결정**
   $$ \text{Grant}=\min(\textit{ExpectedOptimalSize},\ \textit{GlobalMemoryBound}) $$

4. **실행 등급 판정**
   - `Grant ≥ ExpectedOptimalSize` → **Optimal**
   - `ExpectedOnePassSize ≤ Grant < ExpectedOptimalSize` → **One-pass**
   - `Grant < ExpectedOnePassSize` → **Multi-pass**

> **정확한 내부 수식은 공개되지 않았으며 버전에 따라 달라질 수 있다.**
> 그러나 위와 같은 **관측 가능 요소**(GMB, EXPECTED\_*, LAST\_MEMORY\_USED)를 통해 **사실상 동일한 결론**을 얻고 **튜닝**할 수 있다.

---

## 튜닝 절차(체크리스트)

1. **현재 상태 진단**
   - `V$SQL_WORKAREA[_ACTIVE]`에서 **SORT/HASH 작업의 Optimal/One/Multi 비율** 확인
   - `V$PGASTAT.global memory bound`와 `total PGA allocated/inuse` 확인
   - `V$PGA_TARGET_ADVICE`(히트/스필 예측)로 타깃 상향 효과 검토
2. **쿼리/플랜 재검토**(메모리보다 더 중요한 기본기)
   - **정렬 발생 자체를 줄이는 설계**(인덱스가 정렬 대체, Top-N STOPKEY, “정렬 후 가공” 패턴 등)
   - 해시 집계/조인 스필을 줄이는 **카디널리티 정확화**, **조인 순서/방식** 개선
3. **PGA 예산 조정**
   - `PGA_AGGREGATE_TARGET`을 **점진적으로 상향** → `V$PGA_TARGET_ADVICE`로 효과 예측
   - **동시성**도 고려(부하 테스트) : 동접이 많으면 **GMB 하락**
4. **상한/보호선 점검**
   - `PGA_AGGREGATE_LIMIT`(전체 상한)으로 **메모리 폭주 보호**
   - (버전 의존) **workarea per-process 상한**(`_smm_max_size`, `_pga_max_size`)이 과도하게 낮지 않은지 점검(변경 시는 신중)
5. **검증**
   - 동일 워크로드로 **전/후** `workarea executions - optimal/onepass/multipass`와 **응답시간** 비교
   - `sorts (disk)`·`workarea executions - multipass`가 **의미 있게 감소**해야 성공

---

## 예제: 같은 쿼리, 다른 결과(옵션 조합)

### “정렬 자체”를 줄여 메모리 의존도를 낮추기(권장)

```sql
-- 인덱스가 정렬을 대체(ORDER BY amount DESC)
CREATE INDEX ix_sales_amt_desc ON f_sales(amount DESC);

-- Top-N: 정렬키 + 식별자만 뽑은 뒤 나중에 가공(행폭↓ → sort area↓)
WITH topn AS (
  SELECT /*+ INDEX_DESC(f_sales ix_sales_amt_desc) */
         sales_id, amount
  FROM   f_sales
  WHERE  sales_dt BETWEEN :lo AND :hi
  FETCH FIRST 200 ROWS ONLY
)
SELECT s.sales_id, s.amount, p.brand
FROM   topn t
JOIN   f_sales   s ON s.sales_id = t.sales_id
JOIN   d_product p ON p.prod_id  = s.prod_id;
```

- **메모리 사이징에 앞서** 정렬 발생량/행폭을 줄이는 구조가 더 강력하고 견고

### 메모리만으로 해결(가끔 필요한 경우)

```sql
-- 대형 해시 집계/조인이 빈번, 정렬 회피가 어렵다면:
ALTER SYSTEM SET workarea_size_policy = AUTO;
ALTER SYSTEM SET pga_aggregate_target = 12G;
ALTER SYSTEM SET pga_aggregate_limit  = 24G;

-- 관측: 글로벌 바운드 상승 → Optimal 비율↑
SELECT name, value FROM v$pgastat
WHERE name IN ('global memory bound','pga aggregate target','total PGA allocated');
```

---

## 병렬 실행(PX) 주의

- PX 환경에선 **슬레이브마다 별도 workarea**가 생긴다.
- 같은 쿼리라도 DOP가 높으면 **활성 workarea 수가 폭증** → **GMB 하락** → One/Multi 증가
- 해결: **적절한 DOP**, **PGA 예산 상향**, **PX 메모리 관련 파라미터**(버전별 가이드) 점검

---

## 자주 묻는 질문(FAQ)

- **Q. AUTO에서 특정 세션/쿼리만 크게 주고 싶다?**
  A. 원칙적으론 **전역 알고리즘**이 공정하게 나눔. 개별적으로 조정하긴 어렵고,
     보통 **플랜/힌트/구조**로 정렬·스필 자체를 줄이거나 **작업을 나누는** 쪽이 현실적.

- **Q. `PGA_AGGREGATE_LIMIT`을 넘으면?**
  A. Oracle이 **메모리 회수**를 시도하고, 필요 시 **에러(ORA-4036 등)**로 보호한다.
     보고서·배치가 동시에 몰리는 창구를 **분산**하고, **타깃/상한**을 현실화해야 함.

- **Q. 정확한 공식이 왜 없나?**
  A. Oracle은 **피드백 기반 적응형**(부하·히스토리·버전별 개선) 알고리즘을 사용.
     **관측 가능한 값**(GMB·EXPECTED\_*)을 바탕으로 **팩트 기반** 튜닝을 하는 것이 실무 표준.

---

## 최종 체크리스트

- [ ] `WORKAREA_SIZE_POLICY=AUTO` 사용 중인가?
- [ ] `V$PGASTAT.global memory bound`가 **너무 낮지 않은가**?
- [ ] `V$SQL_WORKAREA[_ACTIVE]`에서 **One/Multi** 비중이 높은가?
- [ ] **정렬 회피 설계**(인덱스·STOPKEY·“정렬 후 가공”)를 먼저 시행했는가?
- [ ] `PGA_AGGREGATE_TARGET` 상향의 **효과**(`V$PGA_TARGET_ADVICE`)를 검토했는가?
- [ ] 동시성/병렬도(PX)가 **GMB를 과도하게 낮추는** 상황 아닌가?
- [ ] 상한(`PGA_AGGREGATE_LIMIT`)과 **히든 per-workarea 상한**이 병목을 만들고 있지 않은가?
- [ ] 전/후 **응답시간**과 `workarea executions - multipass`/`sorts (disk)`가 **유의미하게 감소**했는가?

---

## 결론

- AUTO 모드에서 **workarea 배정**은 “**필요한 최적 크기**”와 “**시스템 전역 상한(GMB)**” 중 **작은 값**으로 결정된다.
- 공식은 비공개지만, **관측 가능한 지표**(GMB, EXPECTED\_*, One/Multi 비율)로 **동작을 추정**하고,
  **정렬 회피 설계** → **PGA 예산 조정** → **병렬/동시성 제어**의 순서로 접근하면
  **Sort area**를 적절히 제어하면서 **응답시간과 안정성**을 동시에 확보할 수 있다.
