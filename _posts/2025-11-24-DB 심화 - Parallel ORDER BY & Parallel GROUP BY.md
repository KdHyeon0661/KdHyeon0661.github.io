---
layout: post
title: DB 심화 - Parallel ORDER BY & Parallel GROUP BY
date: 2025-11-24 23:25:23 +0900
category: DB 심화
---
# Oracle 병렬 실행: Parallel ORDER BY & Parallel GROUP BY

## 0) 왜 Parallel ORDER BY / GROUP BY가 까다로운가

병렬(PQ)은 “똑같은 일을 N명이 나눠 하는 것”처럼 보이지만, **정렬·집계는 ‘전역 결과’가 필요**하다는 점에서 단순 분할이 끝이 아니다.

- **ORDER BY**
  전역 순서를 맞추려면 “각자 정렬 → 다시 모아서 순서 재조정”이 필수다.
  즉 **로컬 정렬(Local Sort)** 과 **전역 병합(Global Merge/Sort)** 을 거치며, 중간에 **재분배(PX SEND/RECEIVE + TQ)** 가 들어간다.
- **GROUP BY**
  각 PX가 부분적으로 집계(Partial)한 뒤, **같은 그룹 키끼리 한 곳으로 모으는 재분배(PX SEND HASH)** 가 필요하고,
  그 다음 **최종 집계(Final)** 로 마무리한다.

정리하면:
- PQ는 **스캔 병렬화(쉽다)** 보다
  **리오더링(ORDER BY) / 동일키 수렴(GROUP BY)** 에서 **네트워크·메모리·스큐**가 성능을 좌우한다.

---

## 1) Parallel Execution 빠른 리캡 (플랜을 ‘읽을 수 있는 언어’로 만들기)

### QC, PX 서버, DFO, TQ

- **QC(Query Coordinator)**: SQL을 파싱/최적화하고, PX 서버들을 지휘하며 최종 결과를 조립.
- **PX Servers(Slaves)**: 실제 스캔/조인/정렬/집계 작업 수행.
- **DFO(Data Flow Operation)**: 병렬 실행의 “작업 단계 파이프라인”.
  DFO 사이를 연결하는 통로가 **TQ(Table Queue)**.
- **TQ(Table Queue)**: PX ↔ PX 또는 PX ↔ QC 사이에 레코드를 흘리는 큐.
  플랜에서 `PX SEND ...` / `PX RECEIVE`가 TQ 경계다.

### IN-OUT 컬럼 해석

- `P->P`: PX 서버끼리 레코드 재분배(네트워크 이동 필수)
- `P->S`: PX → QC로 최종 전달
- `PCWP/PCWC`: 같은 집합 중에서 Consume/Produce 관계를 표현.

**핵심**:
`P->P`가 많아질수록 **재분배(네트워크, TQ 스큐, PX Deq 대기)** 가 성능을 결정한다.

### Granule (BRG/PG/SPG)

- **BRG(Block Range Granule)**: PX 서버가 테이블을 블록 범위로 잘라 가져가는 기본 방식. **로드밸런싱(동적 배분)**에 강함.
- **PG(Partition Granule)**: 파티션 단위로 작업 배정.
  파티션-wise 조인/집계에서 중요.
- **SPG(Subpartition Granule)**: 서브파티션 단위.

---

## 2) Parallel ORDER BY — “2단계 정렬 + RANGE 재분배 + 전역 병합”

### 전형적인 플랜 구조

병렬 ORDER BY는 거의 항상 다음 패턴이 보인다.

1) **스캔/필터 단계 (PX BLOCK ITERATOR 등)**
2) **로컬 정렬(Local Sort)**
3) **정렬키 기반 RANGE 재분배**: `PX SEND RANGE` → `PX RECEIVE`
4) **전역 병합/정렬(Global)**: `SORT ORDER BY` 또는 `MERGE SORT`
5) QC로 `P->S`

Oracle 문서/사례에서도 “ORDER BY는 정렬키 범위를 나눠 RANGE 분배 후 병합”이 병렬 표준 패턴으로 설명된다.

---

### 기본 예제: 전역 정렬 (당신 초안 확장)

```sql
EXPLAIN PLAN FOR
SELECT /*+ parallel(f 8) monitor */
       f.sales_id, f.sales_dt, f.amount
FROM   fact_sales f
WHERE  f.sales_dt BETWEEN DATE '2025-02-01' AND DATE '2025-03-01'
ORDER  BY f.amount DESC;

SELECT *
FROM   TABLE(DBMS_XPLAN.DISPLAY(NULL,NULL,
         'BASIC +PARALLEL +ALIAS +PARTITION +PREDICATE +NOTE'));
```

#### 플랜에서 꼭 찾아야 할 줄

- `PX BLOCK ITERATOR` / `PX PARTITION RANGE SINGLE`
  → **Granule 단위 스캔**
- `PX SEND RANGE` / `PX RECEIVE`
  → **정렬키(amount) 기준 RANGE 재분배**
- 상단 `SORT ORDER BY`
  → **전역 병합(최종 순서 맞추기)**

#### 왜 RANGE 분배인가?

전역 ORDER BY는 “정렬키가 disjoint한 구간으로 나뉘어야” 전역 순서를 유지할 수 있다.

- PX1이 “amount 상위 10%”, PX2가 “다음 10%”…처럼 구간이 안 겹치면
  최종 단계는 **단순 병합(run merge)** 만으로 끝난다.
- 따라서 Oracle은 ORDER BY에서 **`PX SEND RANGE`** 를 주로 사용한다.

---

### Top-N이 들어가면 무엇이 달라지나? (STOPKEY)

```sql
EXPLAIN PLAN FOR
SELECT /*+ parallel(f 8) monitor */
       f.sales_id, f.sales_dt, f.amount
FROM   fact_sales f
WHERE  f.sales_dt >= DATE '2025-02-01' AND f.sales_dt < DATE '2025-04-01'
ORDER  BY f.amount DESC
FETCH  FIRST 100 ROWS ONLY;

SELECT *
FROM   TABLE(DBMS_XPLAN.DISPLAY(NULL,NULL,'BASIC +PARALLEL +NOTE'));
```

Top-N이면 플랜에 다음이 보인다.

- `SORT ORDER BY STOPKEY`
- 여전히 `PX SEND RANGE` → `PX RECEIVE`가 있을 가능성이 높음
- 다만 각 PX 서버가 **부분 Top-N 힙**을 유지
  → **전달되는 row 수가 급감**
  → TQ 네트워크/Temp I/O가 크게 줄어듦

즉 Top-N 병렬의 본질은:

> **각 PX가 “자기 로컬 상위 N’만” 들고 있다가,
> 최종 병합 단계에서 전역 상위 N을 확정하며 STOPKEY로 조기 종료.**

---

### ORDER BY에서 터지는 대표 병목/대기 이벤트

#### (1) TEMP spill

- `direct path write temp` / `direct path read temp`
  로컬 또는 전역 정렬이 PGA를 초과하면 TEMP로 spill.
  병렬이면 spill이 **PX 개수만큼 동시에** 발생한다.

**해결 축**
- 정렬 입력을 줄인다(Top-N, 기간 축소, 프루닝).
- 정렬 대상 행폭을 줄인다(정렬키+PK만 먼저 정렬).
- PGA/WORKAREA를 합리적으로 확장한다(AUTO 전제).

#### (2) TQ/네트워크 병목

- `PX Deq Credit: send blkd`
  Producer(PX SEND 측)가 Consumer가 느려 **TQ 크레딧을 못 받고 대기**.

**원인**
- RANGE 분배 구간이 비대칭(= **정렬키 스큐**)
- QC 또는 상위 DFO가 느림
- 너무 큰 DOP로 TQ가 과밀

**대책**
- 정렬키 스큐 완화(분배 버킷 증가 / 정렬키 설계 개선)
- DOP 현실화
- Top-N으로 이동량 자체를 줄임

#### (3) Admission/Queue

- `PX Queuing: statement queue`
  동시 병렬 한도 초과 시 대기.

**대책**
- Resource Manager로 PQ 허용량/우선순위 조절
- 배치 윈도우 분리

---

### ORDER BY 실전 튜닝 패턴

#### 패턴 A) “정렬 전에 얇게 만들고 Top-N 확정 후 확장”

정렬 비용은 **행 수 × 행 폭**에 비례한다.
따라서:

1) 정렬키 + 식별자(ROWID/PK)만 정렬해 얇게 만든다.
2) Top-N 확정 후 상세를 조인/가공한다.

```sql
WITH topn AS (
  SELECT /*+ parallel(f 8) index_desc(f ix_sales_amount_desc) */
         sales_id, amount
  FROM   f_sales f
  WHERE  sales_dt >= :d1 AND sales_dt < :d2
  FETCH FIRST 100 ROWS ONLY
)
SELECT /*+ parallel(s 8) */
       s.sales_id, s.prod_id, s.sales_dt, s.qty, s.amount, s.big_note
FROM   topn t
JOIN   f_sales s ON s.sales_id = t.sales_id;
```

**효과**
- Local/Global sort에 들어가는 row width가 최소화 → PGA/TEMP 대폭 감소.
- PQ에서 특히 체감이 큼.

#### 패턴 B) 인덱스 순서로 “정렬 자체 제거/축소”

- `(sales_dt, amount DESC)` 같은 인덱스가 있으면
  **RANGE 스캔 자체가 정렬 순서를 제공**한다.
- 병렬이라도 입력이 거의 정렬 상태면 전역 단계가 단순 병합이 되어 spill이 줄어든다.

#### 패턴 C) 파티션-wise ORDER BY

- 테이블이 파티션되어 있고 “파티션 내부 순서만 필요”하거나
  “파티션별 Top-N 후 병합”이 비즈니스적으로 허용되면

> **각 파티션에서 STOPKEY로 N’개만 뽑고 → 작은 집합만 최종 병합**

---

## 3) Parallel GROUP BY — “Partial 집계 → HASH 재분배 → Final 집계”

### 전형적인 플랜 구조

GROUP BY는 대부분 다음 2-Stage Aggregation 패턴:

1) **로컬 부분 집계(Partial Hash Group By)**
2) **그룹키 기준 재분배**: `PX SEND HASH` → `PX RECEIVE`
3) **최종 집계(Final Hash Group By)**
4) QC로 `P->S`

Oracle 실행계획 예제에서도 `HASH GROUP BY (PARTIAL)` 아래에 `PX SEND HASH`가 나타나는 것이 표준 흐름이다.

---

### 기본 예제: 고객별 합계

```sql
EXPLAIN PLAN FOR
SELECT /*+ parallel(f 16) monitor */
       f.cust_id, SUM(f.amount) amt
FROM   fact_sales f
WHERE  f.sales_dt BETWEEN DATE '2025-02-01' AND DATE '2025-03-01'
GROUP  BY f.cust_id;

SELECT *
FROM   TABLE(DBMS_XPLAN.DISPLAY(NULL,NULL,'BASIC +PARALLEL +ALIAS +PARTITION +NOTE'));
```

#### 플랜 관찰 포인트

- 하위 `HASH GROUP BY (PARTIAL)`
  → 각 PX가 자기 granule에서 그룹별 부분 합계를 만듦.
- 중간 `PX SEND HASH` / `PX RECEIVE`
  → `cust_id` 해시 기준 재분배 → 같은 cust_id는 같은 PX로 수렴.
- 상위 `HASH GROUP BY`(FINAL)
  → 수렴된 그룹끼리 최종 합산.

**효과**
- Partial 단계에서 row 수가 확 줄어든다.
  즉 네트워크로 보내야 할 데이터가 대폭 감소한다.

---

### GROUP BY에서의 주요 병목/대기

#### (1) Hash spill → TEMP

- `direct path write temp` / `read temp`
  해시 테이블이 PGA를 초과하면 spill.

**대책**
- PGA/WORKAREA_SIZE_POLICY=AUTO에서 target 보정
- 불필요 row/컬럼을 사전 제거

#### (2) 그룹키 스큐 (가장 악명 높은 병목)

스큐가 있으면:

- 한 PX가 “거의 모든 동일 키”를 받아
  Final 단계가 그 PX에 몰림
- 나머지 PX는 놀고, 느린 PX를 기다리며
  `PX Deq Credit: send blkd` 상승

**스큐의 정량 감각**
- 재분배 후 각 PX가 받는 row가 고르게 분산되는 게 이상적.
- `v$pq_tqstat`에서 `num_rows`의 max/min 차이가 크면 스큐. (뒤에서 쿼리 제공)

#### (3) RAC 환경(있다면)

- `gc cr/current request`가 늘면
  Partition-wise 설계를 고려.

---

### GROUP BY 실전 튜닝 패턴

#### 패턴 A) Partial 집계를 “소스 가까이”에서

가능하면 Partial을 최대한 아래로 내린다.

- **조인 후 집계**보다
- **필터/프루닝 후 집계**가 먼저

즉 **집계 입력 자체를 줄여** spill/네트워크를 축소한다.

#### 패턴 B) 키 스큐 완화(SALTING)

특정 키가 압도적으로 많을 때:

1) “키 + salt”로 먼저 분산 집계
2) 바깥에서 salt 제거 후 재집계

```sql
SELECT /*+ parallel(f 16) monitor */ cust_id, SUM(amt) amt
FROM (
  SELECT f.cust_id,
         f.amount amt,
         MOD(ORA_HASH(f.cust_id), 8) salt
  FROM   fact_sales f
)
GROUP BY cust_id;
```

보다 명확한 2단계 버전:

```sql
WITH salted AS (
  SELECT /*+ parallel(f 16) */
         cust_id,
         MOD(ORA_HASH(cust_id), 8) salt,
         SUM(amount) amt
  FROM   fact_sales f
  GROUP  BY cust_id, MOD(ORA_HASH(cust_id), 8)
)
SELECT cust_id, SUM(amt) amt
FROM   salted
GROUP  BY cust_id;
```

#### 패턴 C) 소형 차원은 BROADCAST

```sql
SELECT /*+ leading(c) use_hash(f) parallel(f 16) parallel(c 16)
           pq_distribute(c BROADCAST NONE) monitor */
       f.region_cd, SUM(f.amount)
FROM   dim_customer c
JOIN   fact_sales  f ON f.cust_id = c.cust_id
GROUP  BY f.region_cd;
```

- 작은 집합을 복제하는 게
  해시 재분배보다 싸다.

#### 패턴 D) ROLLUP/CUBE는 “정렬 기반이 섞인다”를 전제로 설계

- ROLLUP/CUBE는 계층 집계를 만들기 때문에
  내부적으로 Sort Group By가 나타날 가능성이 높다.
- 입력량·행폭을 반드시 줄여야 한다.

---

## 4) 병렬 정렬/집계 모니터링: 무엇을 어떻게 보나

### DBMS_XPLAN

```sql
ALTER SESSION SET statistics_level = ALL;

SELECT /*+ monitor parallel(f 16) */
       cust_id, SUM(amount)
FROM   fact_sales f
GROUP  BY cust_id;

SELECT *
FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,
     'BASIC +PARALLEL +ALIAS +PARTITION +PREDICATE +NOTE'));
```

**필독 포인트**
- `PX SEND RANGE`(ORDER BY), `PX SEND HASH`(GROUP BY)
- `SORT ORDER BY STOPKEY`(Top-N)
- `HASH GROUP BY (PARTIAL)`/`FINAL`
- `IN-OUT` 흐름

### TQ 스큐/분배 확인: v$pq_tqstat

```sql
SELECT dfo_number, tq_id, server_type,
       process, num_rows, bytes
FROM   v$pq_tqstat
ORDER  BY dfo_number, tq_id, server_type, process;
```

- 같은 TQ에서 **process별 num_rows가 비슷해야** 이상적.
- max/min이 크게 벌어지면 스큐 → salting/분배전략/키 설계 점검.

### Workarea / PGA / TEMP

```sql
-- 활성 workarea(정렬/해시)
SELECT sid, operation_type, policy,
       expected_optimal_size/1024/1024 exp_opt_mb,
       last_memory_used/1024/1024 last_mb,
       active_time/100 active_sec
FROM   v$sql_workarea_active
ORDER  BY active_time DESC;

-- TEMP 사용
SELECT tablespace, SUM(blocks)*8192/1024/1024 temp_mb
FROM   v$tempseg_usage
GROUP  BY tablespace;
```

### 병렬 관련 대기(ASH)

```sql
SELECT event, COUNT(*) samples
FROM   v$active_session_history
WHERE  sample_time > SYSDATE - 1/288
AND    (event LIKE 'PX Deq%' OR event LIKE 'direct path%')
GROUP  BY event
ORDER  BY samples DESC;
```

---

## 5) 파라미터/메모리/분배전략의 균형

### DOP와 메모리의 곱셈 효과

병렬 환경에서 정렬/해시 영역은 **대략 DOP에 비례해 곱으로 커진다.**

$$
\text{총 Workarea 요구량} \approx \text{Workarea/Slave} \times \text{DOP} \times \text{동시 Workarea 개수}
$$

- DOP=16, 동시에 Sort 2개라면
  한 쿼리만으로도 32배가 된다.
- 따라서 “DOP만 크게”는 대부분 TEMP/PGA 병목으로 귀결.

### WORKAREA_SIZE_POLICY=AUTO 전제 베스트

- PQ 환경에서는 AUTO가 **전역 상한(Global Memory Bound)** 을 계산해
  spill/메모리 폭주를 자동으로 완충한다.
- ORDER BY / GROUP BY가 잦다면 PGA 타깃을 **관측 기반으로 증감**해 최적점을 찾는다.

---

## 6) 현업 체크리스트(ORDER BY & GROUP BY 공통)

1. **플랜에서 PX SEND 종류를 확인**
   - ORDER BY → RANGE
   - GROUP BY → HASH
2. **Top-N이면 반드시 STOPKEY**
3. **정렬/집계 입력량을 먼저 줄여라**
   - 프루닝, 선필터, 투영 최소화
4. **스큐는 v$pq_tqstat로 눈으로 확인**
5. **DOP·PGA·TEMP는 한 덩어리로 튜닝**
   - DOP↑ 하면 PGA/TEMP/네트워크 요구도 같이 ↑
6. **소형 차원은 BROADCAST, 대형은 HASH**
7. **ROLLUP/CUBE는 정렬 비용이 섞임을 가정**
8. **실증**
   - XPLAN + TQSTAT + Workarea + ASH
     네 가지를 같이 봐야 원인이 보인다.

---

## 7) 요약

- **Parallel ORDER BY**
  - **Local Sort → PX SEND RANGE → Global Merge/Sort**
  - Top-N은 **STOPKEY + 부분 Top-N 유지**가 승부.
  - 병목은 주로 **TEMP spill**과 **RANGE 스큐**.

- **Parallel GROUP BY**
  - **Partial Hash GBY → PX SEND HASH → Final GBY**
  - 병목은 **그룹키 스큐**와 **해시 spill**.
  - **salting/partial pushdown/broadcast**가 핵심 무기.

한 줄 결론:
**PX SEND/RECEIVE와 TQ 스큐만 정확히 읽고,
정렬·집계 입력(행수×행폭)과 DOP·PGA·TEMP를 동시에 맞추면
병렬 ORDER BY/GROUP BY의 성능 문제는 대부분 구조적으로 해결된다.**
