---
layout: post
title: DB 심화 - Single Block vs Multiblock I/O
date: 2025-11-04 17:25:23 +0900
category: DB 심화
---
# Single Block vs Multiblock I/O

> **핵심 요약**
> - **Single Block I/O**: **한 번에 1 블록**(일반적으로 8KB)을 읽는다. 전형적으로 **인덱스 Range/Unique Scan → 테이블 BY ROWID** 같은 **랜덤 읽기(OLTP)** 경로에서 발생. **대기 이벤트**는 주로 `db file sequential read`.
> - **Multiblock I/O**: **한 번에 여러 블록**(멀티블록)을 읽는다. **Table Full Scan / Partition Full Scan / Index Fast Full Scan** 같은 **순차 대량 읽기(DW/리포트)** 경로에서 발생. **대기 이벤트**는 `db file scattered read`(버퍼드) 또는 `direct path read`(직접 경로).
> - 튜닝의 핵심은 워크로드 성격에 맞춰 **Single Block I/O 발생량을 구조적으로 줄이고**(커버링 인덱스·클러스터링 팩터·조인 방식 전환), **Multiblock I/O의 효율을 최대화**(프루닝·병렬·Direct Path·I/O 크기)하는 것이다.

---

## 실습 환경 공통: 샘플 스키마 & 통계 수집

```sql
-- 블록 사이즈 기본 8KB 가정
-- 큰 테이블과 인덱스를 만들어 실험
DROP TABLE big_orders PURGE;

CREATE TABLE big_orders (
  order_id     NUMBER PRIMARY KEY,
  cust_id      NUMBER NOT NULL,
  order_dt     DATE   NOT NULL,
  status       VARCHAR2(10),
  amount       NUMBER(12,2),
  pad          VARCHAR2(200)
)
-- 일자 파티션으로 분할(프루닝/순차 스캔 실험용)
PARTITION BY RANGE (order_dt) (
  PARTITION p2025q1 VALUES LESS THAN (DATE '2025-04-01'),
  PARTITION p2025q2 VALUES LESS THAN (DATE '2025-07-01'),
  PARTITION p2025q3 VALUES LESS THAN (DATE '2025-10-01'),
  PARTITION p2025q4 VALUES LESS THAN (DATE '2026-01-01')
);

-- 대량 적재(예: 1천만 행; 데모에서는 규모를 조정)
INSERT /*+ APPEND */ INTO big_orders
SELECT level,
       MOD(level, 2000000)+1,
       DATE '2025-01-01' + MOD(level, 365),
       CASE MOD(level,5) WHEN 0 THEN 'NEW' WHEN 1 THEN 'OK' WHEN 2 THEN 'RTN'
                         WHEN 3 THEN 'CXL' ELSE 'HOLD' END,
       ROUND(DBMS_RANDOM.VALUE(1,1000),2),
       RPAD('x',200,'x')
FROM dual
CONNECT BY level <= 1000000;  -- 데모 크기: 100만
COMMIT;

-- 인덱스
CREATE INDEX ix_bo_cust_dt ON big_orders(cust_id, order_dt DESC, order_id DESC);
CREATE INDEX ix_bo_status  ON big_orders(status);

BEGIN
  DBMS_STATS.GATHER_TABLE_STATS(USER, 'BIG_ORDERS', cascade=>TRUE, method_opt=>'for all columns size skewonly');
END;
/
```

---

## 개념 정리: Single vs Multiblock I/O

### Single Block I/O

- **정의**: 한 I/O 호출에 **딱 1개의 데이터 블록**을 읽는다(보통 8KB).
- **주요 용도**:
  - **인덱스 Range/Unique Scan**이 반환한 **ROWID** 로 테이블 **행 한 건씩** 접근
  - **랜덤 읽기** 중심의 OLTP(계좌조회, 상품상세 등)
- **대표 대기 이벤트**: `db file sequential read`
  - 이름의 “sequential”은 “I/O 호출을 순차적으로 기다린다”는 의미에 가깝고, **연속 블록**이라는 뜻이 아니다.

### Multiblock I/O

- **정의**: 한 I/O 호출에 **여러 블록**(멀티블록)을 **한꺼번에** 읽는다.
- **주요 용도**:
  - **Table Full Scan / Partition Full Scan** (버퍼 캐시 사용) ⇒ `db file scattered read`
  - **Direct Path Read** (버퍼 캐시 우회, 대량 읽기 최적화) ⇒ `direct path read` / `direct path read temp`
  - **Index Fast Full Scan**(인덱스 전체를 빠르게 훑음)
- **장점**: **처리량(throughput)** 유리. 연속 블록을 묶어서 읽기 때문에 **호출 횟수**가 줄고 대역폭을 잘 활용한다.
- **주의**: **읽는 양이 많을수록** 빠르다는 뜻은 아니다. 프루닝/필터로 **읽을 범위 자체**를 줄이는 게 우선.

### 블록/MBRC(멀티블록 리드 카운트) 근사

- 데이터베이스 블록 크기: 보통 **8KB**.
- 멀티블록 I/O 크기 = **MBRC × 블록 크기**.
  - 예: MBRC≈16이면 **16 × 8KB = 128KB/IO** 근사.
- 실제 MBRC는 **버전/플랫폼/스토리지/옵티마이저 판단**에 따라 달라지며, **자동**으로 결정되는 경향(과거 파라미터 `db_file_multiblock_read_count`는 현대엔 힌트적 의미).

**간단 수식**
$$
\text{Single Block IO Calls} \approx \text{방문해야 할 ROWID 수}
$$
$$
\text{Multiblock IO Calls} \approx \frac{\text{스캔해야 할 블록 수}}{\text{MBRC}}
$$

---

## 언제 Single, 언제 Multiblock인가? — 실행계획 관점

### Single Block 지향(OLTP)

- **패턴**: `INDEX RANGE/UNIQUE SCAN` → `TABLE ACCESS BY ROWID`
- **장점**: **필요한 소수 행**만 빠르게 읽는다.
- **단점**: 행이 **너무 많아지면** 랜덤 I/O 폭증 → 비효율.

**예: 고객의 “특정 주문” 찾기(단건 조회)**
```sql
SELECT /*+ index(b ix_bo_cust_dt) */ order_id, amount
FROM   big_orders b
WHERE  cust_id = :cust
AND    order_dt >= :d1
AND    order_dt <  :d2
AND    ROWNUM <= 1;
```
- 결과가 소수이면 **Single Block**이 최적.

### Multiblock 지향(DW/리포트/대량 범위)

- **패턴**: `TABLE FULL SCAN` / `PARTITION RANGE SCAN` / `INDEX FAST FULL SCAN`
- **장점**: 대량 범위를 **연속 블록**으로 빠르게 읽음.
- **단점**: 불필요한 블록까지 많이 읽을 우려 ⇒ **파티션 프루닝**·**프레디케이트 푸시다운**으로 범위를 **작게** 만들 것.

**예: 최근 분기 주문 합계(대량 집계)**
```sql
SELECT /*+ full(b) use_hash(b) */
       SUM(amount)
FROM   big_orders b
WHERE  order_dt >= ADD_MONTHS(TRUNC(SYSDATE,'Q'), -3);
```
- 프루닝으로 **작은 전체만** 읽고, **멀티블록**으로 처리량 극대화.

---

## 대기 이벤트로 구분/관찰하기

### 시스템/세션 레벨 카운터

```sql
-- 시스템 이벤트: Single vs Multi 대기량 비교
SELECT event, total_waits, time_waited_micro/1e6 AS sec
FROM   v$system_event
WHERE  event IN ('db file sequential read',
                 'db file scattered read',
                 'direct path read',
                 'direct path read temp')
ORDER BY sec DESC;

-- 세션 이벤트
SELECT event, total_waits, time_waited_micro/1e6 AS sec
FROM   v$session_event
WHERE  sid = :sid
AND    event IN ('db file sequential read','db file scattered read','direct path read');

-- SQL별 물리/논리 읽기
SELECT sql_id, plan_hash_value, executions,
       buffer_gets, disk_reads,
       ROUND(buffer_gets/NULLIF(executions,0)) avg_buf,
       ROUND(disk_reads/NULLIF(executions,0))  avg_disk
FROM   v$sql
ORDER  BY avg_disk DESC FETCH FIRST 20 ROWS ONLY;
```

### ASH로 실시간/샘플 기반 확인

```sql
-- 최근 5분간 IO 이벤트 분포: Single vs Multi
SELECT event, COUNT(*) samples
FROM   v$active_session_history
WHERE  sample_time > SYSTIMESTAMP - INTERVAL '5' MINUTE
AND    session_type = 'FOREGROUND'
AND    event IN ('db file sequential read','db file scattered read','direct path read')
GROUP  BY event
ORDER  BY samples DESC;
```

### 실행계획에서 경로 확인

```sql
-- 실제 실행계획 + 통계
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST'));
```
- `TABLE ACCESS BY ROWID` + `INDEX RANGE SCAN` → **Single Block** 패턴
- `TABLE ACCESS FULL` / `PARTITION RANGE` + `STOPKEY` → **Multiblock** 가능
- `PX`(병렬) + `FULL` + `HASH JOIN` → 대량 멀티블록/Direct Path 가능성 큼

---

## 실전 시나리오: 같은 질의, 서로 다른 I/O 전략

### “최근 3개월, 고객별 주문 합계 Top-50”

**A안 (잘못된 설계: Single 폭증)**
```sql
SELECT /* bad: NL + Random IO 폭증 */
       c.cust_id, SUM(b.amount) s
FROM   (SELECT DISTINCT cust_id FROM big_orders WHERE order_dt >= ADD_MONTHS(TRUNC(SYSDATE,'MM'),-3)) c
JOIN   big_orders b
  ON   b.cust_id = c.cust_id
WHERE  b.order_dt >= ADD_MONTHS(TRUNC(SYSDATE,'MM'),-3)
GROUP  BY c.cust_id
ORDER  BY s DESC
FETCH FIRST 50 ROWS ONLY;
```
- 옵티마이저가 `NESTED LOOPS`로 가면 `ROWID` 랜덤 접근 → **Single Block** 대기 급증.

**B안 (권장: Multiblock 중심 + 프루닝 + Stopkey)**
```sql
WITH c AS (
  SELECT /*+ MATERIALIZE */ cust_id
  FROM big_orders
  WHERE order_dt >= ADD_MONTHS(TRUNC(SYSDATE,'MM'),-3)
  GROUP BY cust_id
)
SELECT /*+ use_hash(b) full(b) */
       b.cust_id, SUM(b.amount) s
FROM   c
JOIN   big_orders b
  ON   b.cust_id = c.cust_id
WHERE  b.order_dt >= ADD_MONTHS(TRUNC(SYSDATE,'MM'),-3)
GROUP  BY b.cust_id
ORDER  BY s DESC
FETCH FIRST 50 ROWS ONLY;  -- STOPKEY
```
- 큰 테이블 쪽을 **Full/Partition Scan**으로 읽어 **Multiblock I/O** 경로 확보.
- 프루닝으로 “작은 전체”만 스캔하여 효율적.

---

## Single Block I/O **줄이는** 패턴 (OLTP 개선)

### 커버링 인덱스(테이블 BY ROWID 제거)

```sql
-- 자주 쓰는 조회의 SELECT-LIST/조건/정렬을 인덱스에 포함
CREATE INDEX ix_bo_cover ON big_orders(cust_id, order_dt DESC, order_id DESC, amount, status);

-- 커버링 인덱스만으로 해결 → TABLE ACCESS 불필요 ⇒ BY ROWID 제거 ⇒ Single Block 감소
SELECT /*+ index(b ix_bo_cover) */
       order_id, order_dt, amount, status
FROM   big_orders b
WHERE  cust_id = :cust
ORDER  BY order_dt DESC, order_id DESC
FETCH FIRST 50 ROWS ONLY;
```

### 클러스터링 팩터 개선(준-순차화)

- 테이블 물리 순서를 **인덱스 키 순**에 가깝게 재배치(CTAS) → Range Scan의 `BY ROWID` 방문이 **근접**해져 **Single Block I/O의 랜덤성**이 감소.
```sql
-- 재배치
CREATE TABLE big_orders_sorted NOLOGGING AS
SELECT * FROM big_orders
ORDER BY cust_id, order_dt, order_id;

ALTER TABLE big_orders RENAME TO big_orders_old;
ALTER TABLE big_orders_sorted RENAME TO big_orders;

-- 인덱스 재생성 & 통계
DROP INDEX ix_bo_cust_dt;
CREATE INDEX ix_bo_cust_dt ON big_orders(cust_id, order_dt DESC, order_id DESC);

BEGIN
  DBMS_STATS.GATHER_TABLE_STATS(USER,'BIG_ORDERS',cascade=>TRUE, method_opt=>'for all columns size skewonly');
END;
/
```

### 조건/조인 재작성으로 후보 축소(세미조인/IN/EXISTS)

```sql
SELECT /*+ SEMIJOIN */ b.*
FROM   big_orders b
WHERE  EXISTS (
  SELECT 1 FROM some_filter f
  WHERE  f.cust_id = b.cust_id
  AND    f.key = :k
);
```
- **테이블 가기 전** 후보를 줄여 **불필요한 BY ROWID** 방문 감소.

---

## Multiblock I/O **효율을 키우는** 패턴 (DW/리포트 개선)

### 파티션 프루닝 + 해시 조인 + 병렬

```sql
SELECT /*+ full(b) use_hash(b) parallel(b 8) */
       status, COUNT(*), SUM(amount)
FROM   big_orders b
WHERE  order_dt >= DATE '2025-10-01'
GROUP  BY status;
```
- **최근 파티션만** 읽음(프루닝) → **작은 전체**.
- 병렬로 **대역폭** 활용, 멀티블록 크기 활용.

### Direct Path Read 유도(대량 스캔)

```sql
-- 세션 단위 direct path read 유도(대량 작업에서 유리)
ALTER SESSION SET "_serial_direct_read" = TRUE;

SELECT /*+ full(b) */
       COUNT(*)
FROM   big_orders b
WHERE  order_dt >= DATE '2025-10-01';
```
- Direct Path Read는 **버퍼 캐시 우회** → 캐시 오염 방지, 대량 스캔에 이점.
- 주의: OLTP 혼합 환경에서는 무조건적 강제는 지양(실측으로 판단).

### 인덱스 Fast Full Scan(순차 인덱스 스캔)

```sql
SELECT /*+ index_ffs(b ix_bo_status) */
       status, COUNT(*)
FROM   big_orders b
GROUP  BY status;
```
- 인덱스 전체를 멀티블록으로 빠르게 훑음.
- 필요한 컬럼이 인덱스에 있다면 **테이블 방문 없이** 집계 가능.

---

## MBRC(멀티블록 리드)와 I/O 크기

### MBRC의 현실

- 과거엔 `db_file_multiblock_read_count` 로 직관적 제어를 기대했지만, **현대 버전**에선 스토리지/OS/옵티마이저가 I/O 크기를 **자동** 조정하는 경향.
- Exadata/ASM/스토리지에 따라 **Smart Scan**/**Cell** 레벨 최적화가 개입.

### I/O 크기와 처리량 직관

- 블록 8KB, MBRC=16 ⇒ 호출당 128KB.
- 스캔해야 할 블록 수가 1,280,000개라면,
  $$\text{Multiblock IO Calls} \approx \frac{1,280,000}{16} = 80,000$$
- **호출 수**가 줄어들수록 컨텍스트 전환·락/래치·OS 호출 비용이 함께 줄어든다.

---

## Single ↔ Multi 전략 선택 가이드

| 상황 | 권장 경로 | 근거 |
|---|---|---|
| **소수 행**(PK/Unique/높은 선택도) | **Single Block** (인덱스 + BY ROWID) | 필요한 것만 최소로 읽겠다 |
| **대량 범위**(최근 분기/연속 날짜 등) | **Multiblock** (Full/Partition Scan, HJ) | 연속 블록을 묶어 읽는 게 빠르다 |
| **탐색+상세** 혼합(목록→상세) | 목록=Multiblock + Stopkey, 상세=Single | 목록에서 **작게 읽고 멈춤**, 상세는 **정밀 조회** |
| **보고/집계** | Multiblock + 병렬 + Direct Path | 처리량 극대화, 캐시 오염 최소 |
| **OLTP 핫 루프** | Single 줄이기(커버링/클러스터링) | 랜덤 I/O를 최소화해 지연 줄임 |

---

## 관찰/검증 절차(필수)

### 트레이스 & 플랜

```sql
ALTER SESSION SET statistics_level = ALL;
ALTER SESSION SET events '10046 trace name context forever, level 8';

-- 테스트 SQL 실행

ALTER SESSION SET events '10046 trace name context off';

-- SQL Monitor 또는 DBMS_XPLAN으로 확인
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST +PEEKED_BINDS'));
```

### 시스템/세션/SQL 지표

```sql
-- Single vs Multi 이벤트 비중
SELECT event, total_waits, time_waited_micro/1e6 sec
FROM   v$system_event
WHERE  event IN ('db file sequential read','db file scattered read','direct path read')
ORDER  BY sec DESC;

-- SQL별 평균 disk_reads
SELECT sql_id, plan_hash_value, executions,
       ROUND(disk_reads/NULLIF(executions,0)) avg_disk
FROM   v$sql
ORDER BY avg_disk DESC FETCH FIRST 20 ROWS ONLY;
```

### 세그먼트 핫스팟

```sql
SELECT owner, object_name, statistic_name, value
FROM   v$segment_statistics
WHERE  statistic_name IN ('physical reads','physical reads direct','buffer busy waits')
ORDER  BY value DESC FETCH FIRST 20 ROWS ONLY;
```

---

## 안티패턴과 교정

| 안티패턴 | 증상 | 교정 |
|---|---|---|
| 넓은 범위를 **NL + BY ROWID**로 | `db file sequential read` 폭증 | **Hash Join + (Partition) Full Scan**로 Multiblock 경로 |
| OFFSET 페이지(앞부분 버리기) | 불필요 읽기 증가 | Keyset + **STOPKEY** 로 필요한 만큼만 |
| 커버링 인덱스 부재 | BY ROWID 잦음 | **커버링/복합 인덱스** 설계 |
| 클러스터링 팩터 나쁨 | Range Scan도 랜덤화 | **CTAS 재정렬** + 인덱스 재생성 |
| 프루닝 실패 | Full Scan 범위 과다 | 파티션 키 조건/함수 적용 금지/식 변환 |

---

## 추가 고급 주제

### Direct Path Read vs Buffered Read

- **Buffered**: 버퍼 캐시에 적재 → **재사용** 가능(OLTP에 유리)
- **Direct Path**: 캐시 우회 → **대량 작업**에서 효율(캐시 오염 방지)
- 옵티마이저는 **세그먼트 크기·통계·작업 특성**을 보고 자동 선택(버전·파라미터 영향)

### 병렬 실행(PX)과 I/O

- PX는 스캔 범위를 슬라이스해 **여러 서버 프로세스**가 동시에 멀티블록 읽기를 수행
- **주의**: 스토리지/네트워크가 받쳐줘야 효과. 지나친 PX는 스토리지 큐 포화로 역효과.

### 저장소/파일시스템의 영향

- ASM 스트라이핑·세그먼트 레이아웃은 멀티블록의 **연속성**과 **대역폭**에 영향
- NFS/dNFS/ZFS(ARC) 등은 **대역폭·RTT·캐시 정책**이 Multiblock 효율에 결정적

---

## 요약 레시피

1) **쿼리 목적**을 보고 **Single vs Multiblock** 전략을 먼저 선택한다.
2) **Single를 줄여야** 하는 OLTP라면: **커버링 인덱스**, **클러스터링 팩터**, **세미조인/필터 선행**, **부분범위처리**.
3) **Multiblock을 키워야** 하는 DW라면: **파티션 프루닝**, **해시 조인**, **병렬/Direct Path**, **INDEX FFS**, **정렬 포함 인덱스 + STOPKEY**.
4) 모든 변경은 **AWR/ASH/SQL Monitor/Trace** 로 **단위 시간당 I/O·대기·RT**가 실제 줄었는지 확인한다.

---

## 부록 A: 실습용 전/후 비교 쿼리

**A.1 (단건/소량 조회) Single Block 최적화**
```sql
-- BEFORE: 범위 넓고 BY ROWID 과다
SELECT order_id, amount
FROM   big_orders
WHERE  cust_id = :cust
AND    order_dt BETWEEN :d1 AND :d2;

-- AFTER: 커버링 인덱스 + Stopkey (목록 화면 50건)
SELECT /*+ index(b ix_bo_cust_dt) */
       order_id, order_dt, amount
FROM   big_orders b
WHERE  cust_id = :cust
ORDER  BY order_dt DESC, order_id DESC
FETCH FIRST 50 ROWS ONLY;
```

**A.2 (대량 집계) Multiblock 최적화**
```sql
-- BEFORE: NL + 랜덤 I/O 다발 가능
SELECT c.cust_id, SUM(b.amount)
FROM   (SELECT DISTINCT cust_id FROM big_orders WHERE order_dt>=ADD_MONTHS(TRUNC(SYSDATE,'Q'),-1)) c
JOIN   big_orders b ON b.cust_id=c.cust_id
WHERE  b.order_dt>=ADD_MONTHS(TRUNC(SYSDATE,'Q'),-1)
GROUP  BY c.cust_id;

-- AFTER: FULL + HASH + 프루닝
SELECT /*+ full(b) use_hash(b) */
       b.cust_id, SUM(b.amount)
FROM   big_orders b
WHERE  b.order_dt>=ADD_MONTHS(TRUNC(SYSDATE,'Q'),-1)
GROUP  BY b.cust_id;
```

**A.3 Direct Path Read 확인**
```sql
SELECT name, value
FROM   v$sysstat
WHERE  name LIKE 'physical reads direct%';  -- direct path read/ temp

SELECT * FROM v$session_event
WHERE  event LIKE 'direct path read%';
```

---

## 결론

- **Single Block I/O** 는 **정밀한 소량 조회**에 강하고, **Multiblock I/O** 는 **대량 순차 읽기**에 강하다.
- 고성능 시스템은 둘 중 하나만 밀어붙이지 않는다. **OLTP 경로**에서는 Single Block I/O를 **최소화**하고, **DW/리포트 경로**에서는 Multiblock I/O의 **효율**을 극대화한다.
- **프루닝/커버링/클러스터링/조인 전략/병렬/Direct Path** 를 조합해 **읽을 양 자체를 줄이고**, **읽을 때는 잘 읽는** 설계를 하라.
- 최종 판단은 **실측 데이터**(AWR/ASH/Trace/SQL Monitor)로 한다 — **히트율**이 아니라 **응답시간과 처리량**이 목표다.
