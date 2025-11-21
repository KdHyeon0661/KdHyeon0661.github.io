---
layout: post
title: DB 심화 - Parallel ORDER BY & Parallel GROUP BY
date: 2025-11-24 23:25:23 +0900
category: DB 심화
---
# Oracle 병렬 실행: **Parallel ORDER BY** & **Parallel GROUP BY** 완전 정리

**목표**: 대용량 정렬/집계를 병렬로 처리할 때 **어떤 단계에서 무엇이 일어나는지**(IN-OUT, PX SEND/RECEIVE, TQ, Granule), **어떤 대기 이벤트가 터지는지**, **어떻게 힌트/통계/메모리/분배전략으로 개선하는지**를 실습 가능한 예제와 함께 끝까지 설명합니다.
기준 버전: 11g 이상(12c+ 용어 일부 포함). 스토리지/Temp/PGA는 기본값 가정.

---

## 공통 실습 스키마 & 데이터(한 번 실행)

```sql
ALTER SESSION SET nls_date_format = 'YYYY-MM-DD';

-- ① 시계열 RANGE 파티션 팩트
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

-- ② 고객 차원
DROP TABLE dim_customer PURGE;
CREATE TABLE dim_customer (
  cust_id      NUMBER PRIMARY KEY,
  region_group VARCHAR2(10),
  grade        VARCHAR2(10),
  active_yn    CHAR(1)
);

-- 샘플
INSERT INTO dim_customer VALUES (101,'APAC','GOLD','Y');
INSERT INTO dim_customer VALUES (202,'AMER','SILVER','Y');
INSERT INTO dim_customer VALUES (303,'EMEA','BRONZE','N');

INSERT INTO fact_sales VALUES (1, DATE '2025-02-10', 101, 'KR', 100.00);
INSERT INTO fact_sales VALUES (2, DATE '2025-02-11', 202, 'US', 170.00);
INSERT INTO fact_sales VALUES (3, DATE '2025-03-01', 202, 'US', 250.00);
COMMIT;

BEGIN
  DBMS_STATS.GATHER_TABLE_STATS(USER,'FACT_SALES');
  DBMS_STATS.GATHER_TABLE_STATS(USER,'DIM_CUSTOMER');
END;
/
```

> **Tip**: 실제 벤치 시엔 `INSERT /*+ APPEND */ SELECT`로 수백만~수천만 로우를 채우고, `PGA_AGGREGATE_TARGET`, TEMP 용량을 충분히 확보하세요.

---

# 병렬 **ORDER BY**: 2단계(로컬 정렬 → 글로벌 병합/정렬)

## 작동 원리(플랜에서 읽는 포인트)

- **로컬 단계(Local Sort)**: 각 PX 서버가 자신에게 배정된 **granule**(BRG/PG)을 스캔하고 **부분 정렬** 수행
- **재분배/병합(Global)**:
  - **전역 순서를 유지**하려면 보통 **`PX SEND RANGE` → `PX RECEIVE`**가 끼고,
    **`SORT ORDER BY`**(또는 **`MERGE SORT`**)가 상위 단계에서 **전역 병합**을 수행
  - **Top-N**이면 각 PX가 **부분 Top-N 유지**(메모리 절약) 후 최종 단계에서 **STOPKEY**로 빠르게 종료
- **IN-OUT** 열: `P->P`(PX↔PX 재분배)와 `P->S`(PX→QC) 흐름 확인
- **대기 이벤트**: `direct path read/write temp`, `PX Deq Credit: send blkd` 등

## 기본 예제: 월 범위 **전역 정렬**

```sql
EXPLAIN PLAN FOR
SELECT /*+ parallel(f 8) monitor */
       f.sales_id, f.sales_dt, f.amount
FROM   fact_sales f
WHERE  f.sales_dt BETWEEN DATE '2025-02-01' AND DATE '2025-03-01'
ORDER  BY f.amount DESC;

SELECT *
FROM   TABLE(DBMS_XPLAN.DISPLAY(NULL,NULL,'BASIC +PARALLEL +ALIAS +PARTITION +PREDICATE +NOTE'));
```
**플랜 해석 가이드**
- `PX BLOCK ITERATOR`/`PX PARTITION RANGE SINGLE` : granule 타입
- 중간 `PX SEND RANGE` / `PX RECEIVE` : **정렬키 기준 범위 분배**
- 상단 `SORT ORDER BY` : 글로벌 병합/정렬
- `IN-OUT` 열: 스캔은 `PCWP/PCWC`(동일 집합 결합), 중간은 `P->P`, 마지막 `P->S`

### 왜 RANGE 분배인가?

- 전역 순서를 맞추려면 **정렬키 범위를 분할**해 **각 PX가 disjoint한 구간**을 담당 →
  최종 단계는 **단순 병합**(run merge)만 수행 가능 → **Temp I/O 절감**.

## → 병렬 최적화

```sql
EXPLAIN PLAN FOR
SELECT /*+ parallel(f 8) monitor */
       f.sales_id, f.sales_dt, f.amount
FROM   fact_sales f
WHERE  f.sales_dt >= DATE '2025-02-01' AND f.sales_dt < DATE '2025-04-01'
ORDER  BY f.amount DESC
FETCH  FIRST 100 ROWS ONLY;

SELECT *
FROM   TABLE(DBMS_XPLAN.DISPLAY(NULL,NULL,'BASIC +PARALLEL +NOTE +OUTLINE'));
```
**기대 포인트**
- `SORT ORDER BY STOPKEY` 표기(최상단 또는 QC 인근)
- 각 PX가 **Top-N heap**을 유지하며 상위만 전달 → **메모리/네트워크/Temp 감소**
- 상위 병합 단계에서 **100개 충족 즉시 종료** → 응답시간↓(OLAP 대시보드에 유리)

> **주의**: Top-N의 **전역 순서 보장**을 위해서도 보통 `PX SEND RANGE`가 필요.
> 다만 각 PX의 **부분 Top-N 유지**가 전체 데이터 이동량을 크게 줄여줍니다.

## ORDER BY + 조인(차원 선행, 브로드캐스트 vs 해시)

```sql
-- 소형 차원 → 브로드캐스트 → 사실집합만 크게 스캔/정렬
EXPLAIN PLAN FOR
SELECT /*+ leading(c) use_hash(f) parallel(f 8) parallel(c 8)
           pq_distribute(c BROADCAST NONE) monitor */
       f.cust_id, SUM(f.amount) amt
FROM   dim_customer c
JOIN   fact_sales  f
  ON   f.cust_id = c.cust_id
WHERE  c.grade IN ('GOLD','SILVER')
GROUP  BY f.cust_id
ORDER  BY amt DESC;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'BASIC +PARALLEL +NOTE'));
```
- **브로드캐스트**: 작은 집합만 복제 → **재분배 비용↓**, 정렬 단계 집중
- 대형↔대형이면 **HASH-HASH** 분배가 일반적(`PX SEND HASH`) → 정렬키가 집계 결과에 종속이면 **최종 단계 정렬**로 귀결

## ORDER BY 병목과 대기 이벤트

- **`direct path write temp` / `read temp`**: SORT가 메모리 초과로 TEMP 사용
  - **대책**: `PGA_AGGREGATE_TARGET`/`WORKAREA_SIZE_POLICY=AUTO` 확대, **Top-N**/프루닝/선행필터
- **`PX Deq Credit: send blkd`**: Consumer가 느려 Producer가 큐 크레딧을 못 받음
  - **대책**: **RANGE 분배 구간** 조정(스큐 완화), DOP 적정화, 브로드캐스트/해시 전략 재검토
- **Admission Queue**(`PX Queuing: statement queue`): 동시 병렬 한도 초과
  - **대책**: 작업 창 분산, 리소스 매니저 우선순위

---

# 병렬 **GROUP BY**: 2단계(부분 집계 → 최종 집계)

## 작동 원리(핵심)

- **Partial Aggregation(로컬)**: 각 PX가 자신에게 온 레코드에서 **그룹별 부분 집계** 수행
  - **Hash Group By**가 기본(메모리 적합), 부족하면 **one-pass/multi-pass**로 TEMP 사용
- **재분배(키 기준)**: `PX SEND HASH`로 **그룹키 해시** 기준 재분배 → 동일 키는 **같은 PX**로 수렴
- **Final Aggregation(글로벌)**: 각 PX가 받은 동일 키 묶음을 **최종 집계**하여 QC에 전달

> 플랜에는 보통 **두 번의 집계 오퍼레이터**(PARTIAL / FINAL)와 **한 번의 PX SEND HASH**가 등장합니다.

## 기본 예제: 고객별 합계(병렬 해시 집계)

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
**플랜 해석 가이드**
- 하위: `HASH GROUP BY (PARTIAL)` 또는 `GROUP BY (PUSHED DOWN)`
- 중간: `PX SEND HASH` / `PX RECEIVE` (**P->P**)
- 상위: `HASH GROUP BY`(FINAL) → `P->S`로 QC 전달

**장점**
- **중간 단계에서 데이터 감소**(부분 집계) → 네트워크/TQ 부하 절감
- 키 스큐만 없다면 **균형적으로** 분산되어 **대기 이벤트** 감소

## 분배전략 강제: `PQ_DISTRIBUTE`

```sql
-- 대형↔대형 집계: 해시-해시(기본)
SELECT /*+ parallel(f 16) pq_distribute(f HASH HASH) monitor */
       cust_id, SUM(amount)
FROM   fact_sales f
GROUP  BY cust_id;

-- 소형 차원을 조인 후 집계하는 경우: 브로드캐스트로 작은쪽 복제
SELECT /*+ leading(c) use_hash(f) parallel(f 16) parallel(c 16)
           pq_distribute(c BROADCAST NONE) monitor */
       f.region_cd, SUM(f.amount)
FROM   dim_customer c
JOIN   fact_sales  f
  ON   f.cust_id = c.cust_id
GROUP  BY f.region_cd;
```
- **HASH-HASH**: 일반적인 그룹키 기반 재분배
- **BROADCAST**: 작은 집합을 복제해 **재분배 비용↓**(집계 대상이 큰 사실 테이블일 때 유리)

## ROLLUP/CUBE/GROUPING SETS 병렬화

```sql
EXPLAIN PLAN FOR
SELECT /*+ parallel(f 8) monitor */
       f.region_cd, f.cust_id, SUM(f.amount) amt
FROM   fact_sales f
GROUP  BY ROLLUP (f.region_cd, f.cust_id);

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'BASIC +PARALLEL +NOTE'));
```
**포인트**
- **ROLLUP/CUBE**는 **계층적 집계**를 생성 → **정렬 기반**(Sort Group By) 필요성이 커질 수 있음
- 해시 집계 가능 구간은 부분적으로 사용, 상위 레벨 계산 시 **추가 Sort**가 등장할 수 있음
- 대용량이면 **TEMP I/O**가 증가 → **PGA/TEMP/프루닝** 신경쓰기

## 병렬화

```sql
EXPLAIN PLAN FOR
SELECT /*+ parallel(f 8) monitor */ DISTINCT f.cust_id
FROM   fact_sales f
WHERE  f.sales_dt >= DATE '2025-02-01' AND f.sales_dt < DATE '2025-04-01';

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'BASIC +PARALLEL +NOTE'));
```
- 내부적으로 **GROUP BY**로 구현되는 경우가 많다 → 상동의 2단계(부분→최종) 흐름
- 키 스큐가 크면 `PX Deq Credit: send blkd`(재분배 병목) 증가

## GROUP BY 병목과 대기 이벤트

- **`direct path write/read temp`**: 해시 테이블이 **메모리 초과** → TEMP spill
  - **대책**: PGA/WORKAREA 확대, **부분 집계 위치**를 소스 가까이에(가능하면 **GBY pushdown**), 불필요 레코드 사전 필터
- **`PX Deq Credit: send blkd`**: 한 PX가 특정 키에 몰려 느림(**스큐**)
  - **대책**: 키 **SALT 컬럼** 추가/복합키 해시, DOP 조절, **브로드캐스트**로 전환(조건 충족 시)
- **RAC**: `gc cr/current request` 증가 시 **Partition-wise** 집계 설계(파티션키 정렬)

---

# Granule & 분배가 정렬/집계에 미치는 영향

## Granule

- **BRG**(Block Range Granule): Full Scan 정렬/집계의 **기본**. **동적 로드밸런싱**에 유리
- **PG/SPG**(Partition/Subpartition Granule): 파티션 단위 처리. **Partition-wise 집계**시 필수

## 분배(Distribution)

- **ORDER BY**: 전역 순서 필요 → 보통 **`PX SEND RANGE`**
- **GROUP BY**: 동일 키 수렴 → **`PX SEND HASH`**
- **소형 차원**: **BROADCAST**가 해시 재분배보다 유리(네트워크/정렬 부담↓)

---

# 모니터링: 플랜/통계/대기

## 플랜과 TQ(테이블 큐)

```sql
-- 실제 실행 후 플랜
SELECT *
FROM   TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'BASIC +PARALLEL +ALIAS +PREDICATE +NOTE'));

-- TQ 분포/스큐 확인
SELECT dfo_number, tq_id, server_type, inst_id, process, num_rows, bytes
FROM   v$pq_tqstat
ORDER  BY dfo_number, tq_id, server_type, inst_id, process;
```
- `PX SEND RANGE/HASH` 라인과 `P->P` 흐름: **재분배 유무/종류**
- `v$pq_tqstat`에서 num_rows **최대/최소 차이**가 크면 **스큐**

## 대기 이벤트 샘플(ASH/Monitor)

```sql
-- 최근 세션의 병렬 관련 대기 추이(개념적 예시)
SELECT event, COUNT(*) samples
FROM   v$active_session_history
WHERE  session_type = 'FOREGROUND'
AND    sample_time > SYSDATE - 1/288
AND    (event LIKE 'PX Deq%' OR event LIKE 'direct path%' OR event LIKE 'gc %')
GROUP  BY event
ORDER  BY samples DESC;
```

---

# 성능 최적화 체크리스트

## ORDER BY

- **Top-N**로 줄일 수 있으면 반드시 `FETCH FIRST n ROWS ONLY` 적용
- 전역 순서가 꼭 필요하지 않다면 **소트 자체를 제거**(인덱스 정렬 대체, 클러스터링 팩터 개선)
- **RANGE 분배 구간**이 **심하게 비대칭**이면 스큐 완화(버킷 늘리기/정렬키 설계 재검토)
- **PGA/TEMP/DOP** 밸런스: 너무 큰 DOP는 Temp 경합만 키울 수 있음

## GROUP BY

- **Partial → Final** 2단계를 **최대한 효과적으로**: 소스 가까이에서 **부분 집계**
- 키 **스큐** 완화: **SALT**(예: `cust_id*1000+mod(hash(sales_dt),10)`), **복합키** 해시
- **소형 차원 브로드캐스트** 적극 활용
- **ROLLUP/CUBE**는 정렬 기반이 섞일 수 있으므로 **메모리/Temp/프루닝**을 우선 설계

---

# 손에 잡히는 실습 시나리오

## 병렬 ORDER BY vs Top-N

```sql
-- (A) 전체 정렬
SELECT /*+ parallel(f 8) monitor */ *
FROM   fact_sales f
WHERE  f.sales_dt BETWEEN DATE '2025-02-01' AND DATE '2025-03-01'
ORDER  BY f.amount DESC;

-- (B) Top-100
SELECT /*+ parallel(f 8) monitor */ *
FROM   fact_sales f
WHERE  f.sales_dt BETWEEN DATE '2025-02-01' AND DATE '2025-03-01'
ORDER  BY f.amount DESC
FETCH  FIRST 100 ROWS ONLY;

-- 실행 후 두 쿼리의 DBMS_XPLAN, V$PQ_TQSTAT, TEMP/대기 비교
```

## vs 스큐 완화

```sql
-- (A) 기본 병렬 해시 집계
SELECT /*+ parallel(f 16) monitor */ cust_id, SUM(amount)
FROM   fact_sales f
GROUP  BY cust_id;

-- (B) 키 스큐 완화(예시: salt 8-way)
SELECT /*+ parallel(f 16) monitor */ cust_id, SUM(amt)
FROM (
  SELECT f.cust_id, f.amount amt, MOD(ORA_HASH(f.cust_id), 8) salt
  FROM   fact_sales f
)
GROUP  BY cust_id;
-- 또는 GROUP BY cust_id, salt 한 뒤 밖에서 다시 GROUP BY cust_id로 SUM
```
- (B)는 내부 분배 균형을 맞춘 뒤 **외부에서 한 번 더 집계**하여 정확한 결과를 만든다.

---

# 운영 파라미터/힌트 베스트 프랙티스(요약)

- `WORKAREA_SIZE_POLICY=AUTO`, `PGA_AGGREGATE_TARGET`(또는 `PGA_AGGREGATE_LIMIT`) 적정화
- 힌트:
  - `PARALLEL(t, DOP)` / `MONITOR`
  - `PQ_DISTRIBUTE(alias HASH HASH | BROADCAST NONE | RANGE RANGE)`
  - `USE_HASH`, `LEADING`(조인 순서)
  - `PX_JOIN_FILTER(alias)`(재분배 전 필터링 강화)
- 파티션/서브파티션 설계로 **Partition-wise** 처리 기회 극대화

---

## 한 줄 결론

- **ORDER BY**는 **RANGE 분배 + 병합/STOPKEY**가 핵심,
- **GROUP BY**는 **Partial→Hash 재분배→Final**이 핵심입니다.
- 플랜의 **`PX SEND/RECEIVE`**, **`IN-OUT`**, **`V$PQ_TQSTAT` 스큐**만 정확히 읽고
  **PGA/Temp/DOP/분배전략**을 맞추면, 대부분의 **병렬 정렬/집계 병목**은 깔끔하게 정리됩니다.
