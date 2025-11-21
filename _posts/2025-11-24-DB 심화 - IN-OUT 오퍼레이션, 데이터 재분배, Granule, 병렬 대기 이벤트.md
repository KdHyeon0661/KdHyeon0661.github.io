---
layout: post
title: DB 심화 - IN-OUT 오퍼레이션, 데이터 재분배, Granule, 병렬 대기 이벤트
date: 2025-11-24 22:25:23 +0900
category: DB 심화
---
# Oracle 병렬 실행 심화: **IN-OUT 오퍼레이션**, **데이터 재분배**, **Granule**, **병렬 대기 이벤트** 총정리

**목표**: 실행계획과 뷰( `DBMS_XPLAN`, `V$PQ_TQSTAT`, `GV$PX_*` )로 **보이는 것**을 기준으로, 병렬 실행의 내부 흐름을 **끝까지** 이해하고, 실제 **힌트/DDL/모니터링** 예제로 병목을 찾아내고 해결하는 방법을 정리한다.

---

## 실습 공통 스키마 & 데이터

```sql
ALTER SESSION SET nls_date_format = 'YYYY-MM-DD';

-- ① 사실 테이블(월 RANGE 파티션)
DROP TABLE fact_sales PURGE;
CREATE TABLE fact_sales (
  sales_id   NUMBER       NOT NULL,
  sales_dt   DATE         NOT NULL,
  cust_id    NUMBER       NOT NULL,
  region_cd  VARCHAR2(6)  NOT NULL,
  amount     NUMBER(12,2) NOT NULL,
  CONSTRAINT pk_fact_sales PRIMARY KEY (sales_id)
)
PARTITION BY RANGE (sales_dt) (
  PARTITION p2025m01 VALUES LESS THAN (DATE '2025-02-01'),
  PARTITION p2025m02 VALUES LESS THAN (DATE '2025-03-01'),
  PARTITION p2025m03 VALUES LESS THAN (DATE '2025-04-01'),
  PARTITION pmax     VALUES LESS THAN (MAXVALUE)
);

-- ② 고객/달력 차원
DROP TABLE dim_customer PURGE;
CREATE TABLE dim_customer (
  cust_id      NUMBER PRIMARY KEY,
  region_group VARCHAR2(10),
  grade        VARCHAR2(10),
  active_yn    CHAR(1)
);

DROP TABLE dim_calendar PURGE;
CREATE TABLE dim_calendar(
  cal_dt     DATE PRIMARY KEY,
  y          NUMBER,
  m          NUMBER,
  is_weekend CHAR(1)
);

-- 샘플
INSERT INTO dim_customer VALUES (101,'APAC','GOLD','Y');
INSERT INTO dim_customer VALUES (202,'AMER','SILVER','Y');
INSERT INTO dim_customer VALUES (303,'EMEA','BRONZE','N');

INSERT INTO dim_calendar VALUES (DATE '2025-02-10', 2025, 2, 'N');
INSERT INTO dim_calendar VALUES (DATE '2025-02-11', 2025, 2, 'N');
INSERT INTO dim_calendar VALUES (DATE '2025-03-01', 2025, 3, 'Y');

INSERT INTO fact_sales VALUES (1, DATE '2025-02-10', 101, 'KR', 100.00);
INSERT INTO fact_sales VALUES (2, DATE '2025-02-11', 202, 'US', 170.00);
INSERT INTO fact_sales VALUES (3, DATE '2025-03-01', 202, 'US', 250.00);
COMMIT;

BEGIN
  DBMS_STATS.GATHER_TABLE_STATS(USER,'FACT_SALES');
  DBMS_STATS.GATHER_TABLE_STATS(USER,'DIM_CUSTOMER');
  DBMS_STATS.GATHER_TABLE_STATS(USER,'DIM_CALENDAR');
END;
/
```

---

# **IN-OUT 오퍼레이션**: 실행계획의 `IN-OUT` 열 해석

`DBMS_XPLAN.DISPLAY[_CURSOR]`에 `+PARALLEL`을 붙이면, 각 오퍼레이터의 **데이터 흐름 방향**이 `IN-OUT` 컬럼으로 나온다.

### 표기 종류

- **S->P**: **Serial → Parallel** (QC 또는 직렬 단계가 **PX 서버 집합** 쪽으로 보냄)
- **P->S**: **Parallel → Serial** (PX 서버 집합 → QC(Serial)로 반환)
- **P->P**: **Parallel → Parallel** (**두 PX 집합** 간의 데이터 스트림; 중간에 **TQ(테이블 큐)** 사용)
- **PCWP / PCWC**: *Parallel Combined With Parent/Child*.
  - **PCWP**: 부모 오퍼레이터와 **같은** PX 서버에서 실행(테이블 큐를 생략해 **메시지 비용↓**).
  - **PCWC**: 자식 오퍼레이터와 **같은** PX 서버에서 실행.

> 해석 팁: **P->P** 라인이 보이면 **데이터 재분배**가 일어났다는 강력한 신호다(아래 §2).

### 예시: 단순 병렬 스캔

```sql
EXPLAIN PLAN FOR
SELECT /*+ parallel(f 8) */ COUNT(*)
FROM   fact_sales f
WHERE  f.sales_dt BETWEEN DATE '2025-02-01' AND DATE '2025-03-01';

SELECT *
FROM   TABLE(DBMS_XPLAN.DISPLAY(NULL,NULL,'BASIC +PARALLEL +PARTITION +ALIAS'));
```
- 기대: `PARTITION RANGE SINGLE`, `PX BLOCK ITERATOR`(BRG, 아래 §3)
- `IN-OUT` 예: `S->P`(QC→PX로 파라미터 전달), 스캔 `PCWP/PCWC` 조합(오버헤드 감소), 집계 결과 `P->S`(PX→QC).

### 예시: 조인 파이프라인(두 PX 집합)

```sql
EXPLAIN PLAN FOR
SELECT /*+ leading(c) use_hash(f) parallel(c 8) parallel(f 8) */
       SUM(f.amount)
FROM   dim_customer c
JOIN   fact_sales  f
  ON   f.cust_id = c.cust_id
WHERE  f.sales_dt BETWEEN DATE '2025-02-01' AND DATE '2025-03-01'
AND    c.grade IN ('GOLD','SILVER');

SELECT *
FROM   TABLE(DBMS_XPLAN.DISPLAY(NULL,NULL,'BASIC +PARALLEL +PARTITION +ALIAS +NOTE'));
```
- 기대: 한 집합이 **Producer**, 다른 집합이 **Consumer**.
- 중간에 **`PX SEND`/`PX RECEIVE`** 라인과 **`P->P`** 흐름이 보이면 **데이터 재분배** 발생(해시/브로드캐스트 등, §2).

---

# **데이터 재분배(Distribution)**: `PX SEND/RECEIVE`와 `TQ`

병렬 조인/집계는 **키 균형**을 맞추려고 **데이터를 섞는다**. 실행계획에는 `PX SEND XXX` / `PX RECEIVE`와 함께 `TQ####`가 표시된다.

## 재분배 방식

- **HASH**: 조인 키(또는 그룹 키)를 **해시**해 **균등 분배**.
  - 장점: 균형 좋음. 단점: **Skew** 키에 취약 → 한 PX만 과부하.
- **BROADCAST**: 작은 집합을 **모든 PX**에 **복제** 전송.
  - 장점: 큰 집합 **재분배 비용 없음**. 단점: 작은 집합이 충분히 **작아야** 함.
- **RANGE**: 범위로 분배(주로 **Sort-Merge**나 **Order** 관련).
- **ROUND-ROBIN**: 키 상관없이 라운드 로빈(부담 분산, 조인 키 일치가 필요없는 단계).
- **PARTITION / PARTITION HASH**: 파티션(또는 서브파티션) **소유에 맞춰** 분배(**Partition-wise Join** 달성).
  - 장점: **재분배 최소화**. 전제: **두 입력이 동일한 파티션키/경계**를 가짐.

## 힌트로 제어: `PQ_DISTRIBUTE`

```sql
-- (A) 해시-해시(대형↔대형 조인, 비교적 균형)
SELECT /*+ leading(f) use_hash(c) parallel(f 16) parallel(c 16)
           pq_distribute(f HASH HASH) pq_distribute(c HASH HASH) */
       COUNT(*)
FROM   fact_sales f
JOIN   dim_customer c
  ON   c.cust_id = f.cust_id;

-- (B) 브로드캐스트(작은 차원 → 큰 사실로 복제)
SELECT /*+ leading(c) use_hash(f) parallel(f 16) parallel(c 16)
           pq_distribute(c BROADCAST NONE) */
       SUM(f.amount)
FROM   dim_customer c
JOIN   fact_sales  f
  ON   f.cust_id = c.cust_id;

-- (C) 파티션-와이즈(두 테이블이 같은 파티션 키/경계일 때)
--    pq_distribute 둘다 PARTITION 이면 재분배 거의 없음(풀/부분 P-W Join)
```

> **실전 규칙**
> - **소-대** 조인: **BROADCAST** 채택(‘소’ 기준 수십~수백 MB 이내가 일반적 감)
> - **대-대** 조인: **HASH-HASH** 기본, 스큐 있으면 **SALT**·**복합 키**·**파티션 정렬** 고려
> - **파티션 정렬** 가능: **PARTITION-WISE**로 최고 효율

## 감지

```sql
-- 각 TQ(테이블 큐)의 송수신 로우/바이트 분포 확인
SELECT dfo_number, tq_id, server_type, MIN(num_rows), MAX(num_rows),
       ROUND(100*(MAX(num_rows)-MIN(num_rows))/NULLIF(MAX(num_rows),0),1) skew_pct
FROM   v$pq_tqstat
GROUP  BY dfo_number, tq_id, server_type
ORDER  BY dfo_number, tq_id, server_type;

-- PX 세션-프로세스 매칭
SELECT sid, qcsid, server_set, server#
FROM   gv$px_session
ORDER  BY qcsid, server_set, server#;
```
- **`skew_pct`가 높으면** 일부 PX에 로우가 몰린다 → **HASH 키 개선/소금(SALT)/분해 UNION ALL** 등.

---

# **Granule(그라뉼)**: PX가 나눠서 먹는 **작업 조각**

병렬 스캔/집계는 데이터를 **Granule** 단위로 나누어 **PX 서버에 동적 할당**한다.

## Granule 유형

- **BRG(Block Range Granule)**: 세그먼트를 **블록 범위**로 쪼갬.
  - 플랜에 **`PX BLOCK ITERATOR`** 표시.
  - 장점: **동적 로드밸런싱**(빠른 서버가 더 많은 granule을 가져감).
  - 일반 **Full Scan**의 표준.
- **PG(Partition Granule)**: **파티션 단위**로 할당.
  - 플랜에 **`PX PARTITION RANGE`**, **`PX PARTITION HASH/LIST`** 등 표기.
  - **Partition-wise** 작업, 파티션 단위 **I/O 지역성** 확보.
- **SPG(Subpartition Granule)**: **서브파티션** 단위.

> Granule 수는 **DOP보다 충분히 많게** 잡혀 **동적 균형**을 돕는다(큰 파티션 하나가 한 PX만 오래 잡아먹지 않게).

## 예시로 보는 구분

```sql
-- BRG: 일반 병렬 Full Scan
EXPLAIN PLAN FOR
SELECT /*+ parallel(f 8) */ SUM(amount) FROM fact_sales f;
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY(NULL,NULL,'BASIC +PARALLEL'));

-- PG: 파티션 범위를 명확히 줄이거나 파티션-와이즈 조인 가능할 때
EXPLAIN PLAN FOR
SELECT /*+ parallel(f 8) */ SUM(amount)
FROM   fact_sales f
WHERE  f.sales_dt BETWEEN DATE '2025-02-01' AND DATE '2025-03-01';
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY(NULL,NULL,'BASIC +PARTITION +PARALLEL'));
```
- **해석**:
  - `PX BLOCK ITERATOR` → **BRG**
  - `PX PARTITION RANGE [SINGLE/ITERATOR]` → **PG**
  - `PSTART/PSTOP`가 단일/키 범위면 프루닝도 함께 확인 가능

---

# 병렬 처리 과정에서 발생하는 **대기 이벤트**

병렬 실행은 **두 종류**의 대기를 주로 본다:
1) **메시징/큐잉(PX Deq 계열, Admission Queue)**
2) **I/O(Direct Path)** + RAC라면 **GC(글로벌 캐시) 대기**

## 메시징/큐잉(Producer ↔ Consumer)

- **`PX Deq Credit: send blkd`**
  - **Producer** 가 **Consumer 크레딧**(버퍼/메시지 큐) 없어서 **전송을 못 하고** 막혀 있음.
  - 원인: Consumer가 느림(해시/소트/필터로 **처리 지연**), **네트워크/RAC** 지연.
  - 대책: **스큐 완화**, Consumer 측 **메모리/워크에어리어** 확대(정렬/해시 onepass 방지), **브로드캐스트/해시** 선택 재검토.
- **`PX Deq: Execute Reply`**
  - **QC** 또는 상위 단계가 **슬레이브 응답**을 기다림(작업 분배/완료 통지).
  - 정상적일 수 있으나, 과다하면 **슬레이브 작업이 길다**는 뜻(아래 I/O/스큐 확인).
- **`PX Deq: Signal ACK` / `PX Deq: Parse Reply` 등**
  - 제어 메시지 왕복 중(일반적 오버헤드, 과도하면 통신 병목 의심).

### Admission/큐잉

- **`PX Queuing: statement queue`**
  - 시스템이 허용한 **동시 병렬 한도**를 넘어서 **대기열**에서 대기 중.
  - 파라미터/리소스매니저로 **동시 DOP** 제한: 대기 자체는 정상적 동작.
- **`enq: PS - contention`**
  - 병렬 문장(enqueue) 자원 경합. **병렬 서버 부족** 등.

## & Temp

- **`direct path read` / `direct path read temp`**
  - 병렬 스캔/소트/해시에서 **버퍼 캐시 우회** **직접 I/O**.
  - 대책: **스토리지 대역폭**, **파티션/프루닝**, **DOP 적정화**, **선행 필터/조인필터**.
- **`direct path write temp`**
  - 해시/소트 **메모리 초과**로 TEMP 쓰기(원패스/멀티패스).
  - 대책: **PGA/WORKAREA** 증가, 입력 감소(필터링/프루닝), DOP 조정.

## RAC(캐시 퓨전)

- **`gc cr request` / `gc current request`** 등
  - 병렬 분산이 **인스턴스 간** 발생 → **블록 전달** 대기.
  - 대책: **Partition-wise** 설계(동일 파티션 키/경계), **instance affinity**(서비스/파티션 배치), 재분배 최소화.

---

# 종합 실습: 플랜/통계로 흐름 읽기

## 조인 + 재분배 패턴 비교

```sql
-- ① HASH-HASH (대-대)
SELECT /*+ leading(c) use_hash(f) parallel(c 8) parallel(f 8)
           pq_distribute(c HASH HASH) pq_distribute(f HASH HASH)
           monitor */
       SUM(f.amount)
FROM   dim_customer c
JOIN   fact_sales  f
  ON   f.cust_id = c.cust_id
WHERE  f.sales_dt BETWEEN DATE '2025-02-01' AND DATE '2025-03-01';

-- ② BROADCAST (소-대)
SELECT /*+ leading(c) use_hash(f) parallel(c 8) parallel(f 8)
           pq_distribute(c BROADCAST NONE) monitor */
       SUM(f.amount)
FROM   dim_customer c
JOIN   fact_sales  f
  ON   f.cust_id = c.cust_id
WHERE  c.grade IN ('GOLD','SILVER')
AND    f.sales_dt BETWEEN DATE '2025-02-01' AND DATE '2025-03-01';
```

### 모니터링/플랜 확인

```sql
-- 실제 실행 후 플랜: IN-OUT, TQ, 재분배 방식 확인
SELECT *
FROM   TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'BASIC +PARALLEL +ALIAS +PROJECTION +NOTE'));

-- TQ 분포/스큐 확인
SELECT * FROM v$pq_tqstat ORDER BY dfo_number, tq_id, server_type, inst_id, process;

-- PX 세션/프로세스
SELECT qcsid, sid, server_set, server#, REQUESTED_DOP, STATUS
FROM   gv$px_session
ORDER  BY qcsid, server_set, server#;

-- (ASH/AWR) 대기 이벤트 요약(대략)
-- select * from v$active_session_history where ...
```

**판독 체크리스트**
- `PX SEND XXX` 종류: **HASH / BROADCAST / RANGE / ROUND-ROBIN / PARTITION**
- `IN-OUT = P->P`: **재분배 존재**
- `PCWP/PCWC`: **같은 PX 집합**으로 묶어 **TQ 생략**(오버헤드↓)
- `V$PQ_TQSTAT` 스큐: **최대/최소 row 차이**가 큰지
- 대기: `PX Deq Credit: send blkd`(Backpressure), `direct path read/write temp`(I/O/Temp)

---

# 병목 원인별 **해결 전략**

## `PX Deq Credit: send blkd`(Producer 막힘)

- **원인**: Consumer가 느리다(해시 빌드, 정렬, Temp I/O, 스큐 키).
- **대응**
  1) **브로드캐스트**로 전환(작은 테이블일 때): `pq_distribute(dim BROADCAST NONE)`
  2) **스큐 완화**: 조인키 **복합화**(예: `(cust_id, trunc(sales_dt,'DD'))`), **SALT 컬럼** 도입
  3) 워크에어리어 확대:
     ```sql
     ALTER SESSION SET workarea_size_policy = AUTO;
     ALTER SYSTEM  SET pga_aggregate_target = <증가>;
     ```
  4) **조인 필터** 활성(미리 걸러내기):
     ```sql
     SELECT /*+ px_join_filter(f) */ ... FROM dim d JOIN fact f ON ...
     ```

## Temp 과다(`direct path write/read temp`)

- **원인**: 메모리 부족으로 해시/소트가 디스크 사용.
- **대응**: **PGA/WORKAREA 상향**, **입력 축소**(프루닝/선행필터), **DOP 조정**(너무 큰 DOP는 오히려 Temp 경합↑).

## Admission Queue(대기열)

- **원인**: 시스템 병렬 한도 도달.
- **대응**: 배치 시간 분산, **리소스 매니저**로 중요 작업 우선, 필요 시 **PARALLEL_SERVERS_TARGET/MAX** 조정.

## RAC GC 대기

- **원인**: 인스턴스 간 **블록 공유** 증가(재분배/일관성).
- **대응**: **Partition-wise**(같은 키/경계), **서비스-파티션 친화 배치**, **GPI 대신 로컬 인덱스** 고려.

---

# 설계 팁: **IN-OUT**/재분배를 줄이는 방향

- **Partition-wise Join**: 테이블을 **같은 파티션키/경계**로 설계 → `P->P` **최소화**
- **로컬 인덱스(Prefixed/Non-Prefixed)**로 **프루닝+리딩컬럼**을 동시에 잡는다
- **소형 차원**은 **브로드캐스트** 기본값으로(해시 재분배보다 싸다)
- **스큐 내재** 키는 **SALT**(해시함수로 분산), **복합키**로 분해
- Granule은 **DOP보다 넉넉**히(Oracle이 자동), 대형 파티션은 **서브파티션**으로 더 쪼개 동적 균형 보조

---

# 미니 랩: 스큐 유발 → 개선 비교

```sql
-- 1) 스큐 유발: cust_id=202 비중이 매우 큰 상태라고 가정
-- (데이터 준비 과정 생략: 202가 80% 이상)

-- A. HASH-HASH(스큐 악화)
SELECT /*+ leading(f) use_hash(c) parallel(f 16) parallel(c 16)
           pq_distribute(f HASH HASH) pq_distribute(c HASH HASH) monitor */
  COUNT(*)
FROM fact_sales f JOIN dim_customer c ON c.cust_id=f.cust_id;

--      → V$PQ_TQSTAT 에서 한 PX에 row가 몰림, PX Deq Credit: send blkd↑

-- B. 브로드캐스트로 개선(차원 소형 가정)
SELECT /*+ leading(c) use_hash(f) parallel(f 16) parallel(c 16)
           pq_distribute(c BROADCAST NONE) monitor */
  COUNT(*)
FROM fact_sales f JOIN dim_customer c ON c.cust_id=f.cust_id;

--      → 해시 재분배 제거, Producer/Consumer backpressure↓
```

---

# 부록: 유용한 뷰/힌트 리스트

- **뷰**
  - `V$PQ_TQSTAT` : TQ별 송수신 로우/바이트(스큐 확인)
  - `GV$PX_SESSION`, `GV$PX_PROCESS`, `GV$PX_PROCESS_SYSSTAT` : PX 세션/프로세스
  - `V$SQL_MONITOR`, `V$ACTIVE_SESSION_HISTORY(ASH)` : 단계별 대기/진행
- **주요 힌트**
  - `PARALLEL(table|index, DOP)` / `NOPARALLEL`
  - `PQ_DISTRIBUTE(table_alias METHOD1 METHOD2)`
  - `LEADING()/USE_HASH()/USE_MERGE()` 조합으로 **조인 순서/방식** 고정
  - `PX_JOIN_FILTER(table_alias)` : 런타임 필터로 **불필요 재분배 감소**

---

## 한눈 요약

- **IN-OUT**: `S->P / P->P / P->S / PCWC/PCWP` → **데이터 흐름**과 **TQ 유무**를 읽는 창
- **재분배**: `PX SEND {HASH|BROADCAST|RANGE|ROUND-ROBIN|PARTITION}`
  - **소-대**는 **BROADCAST**, **대-대**는 **HASH**, 가능하면 **Partition-wise**
- **Granule**: **BRG**(블록 범위) vs **PG/SPG**(파티션/서브파티션) → **동적 균형/지역성**
- **대기 이벤트**:
  - 메시징: `PX Deq Credit: send blkd`, `PX Deq: Execute Reply`, 큐잉
  - I/O: `direct path read/write(temp)`
  - RAC: `gc cr/current request`
- **튜닝 키**: 스큐 제거, 조인/분배 방식 선택, 워크에어리어 확보, 파티션-와이즈 설계, 적정 DOP

> 결론: 병렬 실행은 **“데이터를 어떻게 나누고(Granule), 어디로 섞어 보내며(재분배), 어느 집합에서 처리하는가(IN-OUT)”** 의 문제다.
> 실행계획의 `IN-OUT`/`PX SEND`/`TQ`와 `V$PQ_TQSTAT` 스큐 지표만 정확히 읽어도, **대기 이벤트의 8할**은 **원인-대응**이 바로 잡힌다.






