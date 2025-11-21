---
layout: post
title: DB 심화 - PQ_DISTRIBUTE
date: 2025-11-25 15:25:23 +0900
category: DB 심화
---
# Oracle **PQ_DISTRIBUTE** 힌트

**주제**
1) PQ_DISTRIBUTE 힌트의 **용도(왜 쓰는가)**
2) **구문**과 의미 정확히 이해하기
3) 지원되는 **분배 방식 지정법**과 조합
4) **튜닝 사례**: 브로드캐스트/해시/파티션-와이즈 강제, 스큐 완화, RAC 대기 절감 등

> 기준: Oracle 11g 이상(12c/19c에서도 그대로 통함).
> 목표: 실행계획의 `PX SEND/RECEIVE`, `IN-OUT`, `TQ####`를 **읽고**, 힌트로 **원하는 데이터 흐름**을 **재현/강제**해 성능을 안정화한다.

---

## 실습 공통 스키마(한 번만 실행)

```sql
ALTER SESSION SET nls_date_format = 'YYYY-MM-DD';
ALTER SESSION SET statistics_level = ALL;  -- DBMS_XPLAN.DISPLAY_CURSOR 통계 풍부

-- 사실 테이블 2개: 해시 파티셔닝(고객 기준) → Full Partition-Wise Join 시연
DROP TABLE fact_sales PURGE;
CREATE TABLE fact_sales (
  sales_id   NUMBER       NOT NULL,
  cust_id    NUMBER       NOT NULL,
  sales_dt   DATE         NOT NULL,
  region_cd  VARCHAR2(6)  NOT NULL,
  amount     NUMBER(12,2) NOT NULL,
  CONSTRAINT pk_fact_sales PRIMARY KEY (sales_id)
)
PARTITION BY HASH (cust_id)
PARTITIONS 8;

DROP TABLE fact_clicks PURGE;
CREATE TABLE fact_clicks (
  click_id   NUMBER       NOT NULL,
  cust_id    NUMBER       NOT NULL,
  click_dt   DATE         NOT NULL,
  source_cd  VARCHAR2(10) NOT NULL,
  cost       NUMBER(12,2) NOT NULL,
  CONSTRAINT pk_fact_clicks PRIMARY KEY (click_id)
)
PARTITION BY HASH (cust_id)
PARTITIONS 8;

-- 비파티션 큰 테이블(Partial/Dynamic 용)
DROP TABLE orders_np PURGE;
CREATE TABLE orders_np (
  order_id   NUMBER       NOT NULL,
  cust_id    NUMBER       NOT NULL,
  order_dt   DATE         NOT NULL,
  amt        NUMBER(12,2) NOT NULL,
  CONSTRAINT pk_orders_np PRIMARY KEY (order_id)
);

-- 소형 차원(브로드캐스트 용)
DROP TABLE dim_customer PURGE;
CREATE TABLE dim_customer (
  cust_id      NUMBER PRIMARY KEY,
  region_group VARCHAR2(10),
  grade        VARCHAR2(10),
  active_yn    CHAR(1)
);

-- 샘플 데이터(요약). 실제 벤치는 APPEND로 대량 적재 권장
INSERT INTO dim_customer VALUES (101,'APAC','GOLD','Y');
INSERT INTO dim_customer VALUES (202,'AMER','SILVER','Y');
INSERT INTO dim_customer VALUES (303,'EMEA','BRONZE','N');

INSERT INTO fact_sales  VALUES (1,101, DATE '2025-02-10','KR',100);
INSERT INTO fact_sales  VALUES (2,202, DATE '2025-02-11','US',170);
INSERT INTO fact_sales  VALUES (3,202, DATE '2025-03-01','US',250);

INSERT INTO fact_clicks VALUES (1,101, DATE '2025-02-10','AD', 1.2);
INSERT INTO fact_clicks VALUES (2,202, DATE '2025-02-11','SEO',0.7);
INSERT INTO fact_clicks VALUES (3,202, DATE '2025-03-01','AD', 1.1);

INSERT INTO orders_np   VALUES (10,101, DATE '2025-02-10', 300);
INSERT INTO orders_np   VALUES (20,202, DATE '2025-02-11', 500);
INSERT INTO orders_np   VALUES (30,202, DATE '2025-03-01', 700);

COMMIT;

BEGIN
  DBMS_STATS.GATHER_TABLE_STATS(USER,'FACT_SALES');
  DBMS_STATS.GATHER_TABLE_STATS(USER,'FACT_CLICKS');
  DBMS_STATS.GATHER_TABLE_STATS(USER,'ORDERS_NP');
  DBMS_STATS.GATHER_TABLE_STATS(USER,'DIM_CUSTOMER');
END;
/
```

---

# PQ_DISTRIBUTE 힌트의 **용도**

병렬 실행에서 오퍼레이터(특히 **조인/집계**) 사이에 **데이터가 어떻게 분배/전송**되는지는 성능을 **좌우**합니다.
옵티마이저는 **기본 규칙**(통계/카디널리티/오브젝트 크기)에 따라 **HASH 재분배, BROADCAST 복제, PARTITION-WISE** 등을 선택하는데,

- 통계 부정확/스큐/설계 제약 등으로 **비효율적인 분배**가 선택되면
  → `PX Deq Credit: send blkd`(메시지 큐 역압), `direct path write/read temp`(해시/소트 Temp 스필), RAC의 `gc cr/current request` 등이 크게 늘어납니다.

**PQ_DISTRIBUTE**는 이런 **데이터 분배 전략을 강제**하여,
- **작은 쪽 브로드캐스트**(작을 때 진리)
- **대형↔대형 해시-해시**(키 기준 균등 분배)
- **파티션-와이즈 조인**(재분배 최소화)
로 유도할 수 있게 해줍니다.

---

# **구문** 이해하기

```sql
/*+ PQ_DISTRIBUTE( table_alias  method_in  method_out ) */
```
- `table_alias` : 힌트를 **적용할 테이블/뷰의 별칭**. (반드시 별칭 사용 권장)
- `method_in`, `method_out` : 이 **row source**가 병렬 파이프라인에서 **다른 집합과 상호작용**할 때의 **분배 전략**을 지정.
  관행적으로 다음과 같이 사용합니다.
  - **해시-해시**: `PQ_DISTRIBUTE(t HASH HASH)`
    → 조인 키 기준으로 **양 입력**을 해시 재분배하여 **동일 키를 같은 PX**에 모읍니다.
  - **브로드캐스트**: `PQ_DISTRIBUTE(dim BROADCAST NONE)`
    → `dim`을 **모든 PX로 복제**, 반대편 큰 집합은 **재분배 없이 로컬 처리**.
  - **파티션-와이즈**: `PQ_DISTRIBUTE(a PARTITION PARTITION)`
    → 조인 키가 파티션 키이고 **동일한 파티션 방식/경계**일 때 **Full Partition-Wise Join** 지향.
  - **NONE**: 재분배를 **하지 않음**(가능한 경우에 한함). 보통 브로드캐스트의 **반대편**이나, 파티션-와이즈에서 사용.

> 실무 팁: 문법상 두 토큰을 모두 적지만, 한쪽만 **핵심 지정**하고 반대쪽은 **NONE**으로 명시하는 패턴이 가장 읽기 쉽고 안전합니다.
> (예: 브로드캐스트는 **복제 대상**에만 `BROADCAST NONE`을 주고, 큰쪽은 생략/기본 유지)

---

# **분배 방식** 지정(지원 조합)

일반적으로 다음 **네 가지**를 사용합니다.

| 키워드 | 의미 | 사용 맥락 |
|---|---|---|
| `HASH` | 조인키 **해시**로 **균등 재분배** | 대형↔대형(사실↔사실), DISTINCT/GROUP BY의 키 분산 |
| `BROADCAST` | 작은 집합을 **모든 PX에 복제** | 스타/스노우플레이크 차원, 작을 때 강력 |
| `PARTITION` | **파티션-와이즈**(같은 파티션 방식/경계/키) | Full/Partial PWJ 강제 |
| `NONE` | **재분배 없음**(가능한 경우에 한해) | 브로드캐스트 반대편, 파티션-와이즈 |

> 참고: 내부적으로 `RANGE/ROUND-ROBIN` 등의 분배도 존재하지만, **조인 튜닝**에서 명시적으로 쓰는 힌트 키워드는 **위 네 가지**가 실전 표준입니다.

---

# **예제로 배우는** PQ_DISTRIBUTE

## 대형↔대형: **해시-해시**

```sql
EXPLAIN PLAN FOR
SELECT /*+ leading(s) use_hash(c)
           parallel(s 16) parallel(c 16)
           pq_distribute(s HASH HASH)
           pq_distribute(c HASH HASH)
           monitor */
       s.cust_id, SUM(s.amount) amt, SUM(c.cost) cost
FROM   fact_sales s
JOIN   fact_clicks c
   ON  c.cust_id = s.cust_id
GROUP  BY s.cust_id;

SELECT *
FROM TABLE(DBMS_XPLAN.DISPLAY(NULL,NULL,'BASIC +PARALLEL +ALIAS +NOTE'));
```
**해석 포인트**
- 플랜 중간에 `PX SEND HASH` / `PX RECEIVE`( `IN-OUT = P->P` )가 나타나며, **양쪽**에서 해시 재분배를 수행.
- 동일 키가 한 PX로 **수렴** → 해시빌드/프로브 **지역화**.
- **주의**: 키 스큐가 심하면 `PX Deq Credit: send blkd` 및 어느 한 슬레이브 과부하.
  → **SALT 컬럼**(예: `MOD(ORA_HASH(cust_id), 8)`)로 분산도 보정, 또는 브로드캐스트 전환 고려.

---

## 소형↔대형: **브로드캐스트**

```sql
EXPLAIN PLAN FOR
SELECT /*+ leading(d) use_hash(s)
           parallel(d 8) parallel(s 16)
           pq_distribute(d BROADCAST NONE)
           monitor */
       s.cust_id, SUM(s.amount) amt
FROM   dim_customer d
JOIN   fact_sales   s
   ON  s.cust_id = d.cust_id
WHERE  d.grade IN ('GOLD','SILVER')
GROUP  BY s.cust_id;

SELECT *
FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'BASIC +PARALLEL +NOTE'));
```
**효과**
- `dim_customer`가 `PX SEND BROADCAST`로 **복제**, `fact_sales`는 **재분배 없이** 로컬 병렬 스캔→조인.
- **네트워크/큐잉**이 크지 않고, 해시 재분배를 피할 수 있어 **응답/스루풋↑**.
- **조건**: 복제 대상이 **충분히 작아야** 함(환경에 따라 수십~수백 MB 내).
  - 통계가 틀려서 **작지 않은데 브로드캐스트** 되면 오히려 **폭증** → 통계 정합/필터 선적용 중요.

---

## Full Partition-Wise Join: **PARTITION PARTITION**

```sql
EXPLAIN PLAN FOR
SELECT /*+ leading(s) use_hash(c)
           parallel(s 16) parallel(c 16)
           pq_distribute(s PARTITION PARTITION)
           pq_distribute(c PARTITION PARTITION)
           monitor */
       s.cust_id, SUM(s.amount + c.cost) sum_amt
FROM   fact_sales  s
JOIN   fact_clicks c
   ON  c.cust_id = s.cust_id
GROUP  BY s.cust_id;

SELECT *
FROM TABLE(DBMS_XPLAN.DISPLAY(NULL,NULL,'BASIC +PARALLEL +NOTE +PARTITION'));
```
**전제**
- **조인 키 = 파티션 키**, **동일 방식/개수/경계** (여기선 둘 다 `PARTITION BY HASH(cust_id) PARTITIONS 8`)
**효과**
- `PX SEND` **최소/없음**, 파티션별 **지역 조인** → 네트워크/큐잉/RAC GC 대기 **극소화**.
- **RAC**에선 파티션-서비스 **affinity**까지 잡으면 **scalability 최고**.

---

## Partial PWJ + 동적 재분배(비파티션 보정)

비파티션 `orders_np`와 파티션 `fact_sales`를 조인.
```sql
-- (A) 기본: 옵티마이저 판단에 맡김 → 때로는 비효율 분배
EXPLAIN PLAN FOR
SELECT /*+ leading(s) use_hash(o)
           parallel(s 16) parallel(o 16)
           monitor */
       s.cust_id, SUM(s.amount), SUM(o.amt)
FROM   fact_sales s
JOIN   orders_np  o
  ON   o.cust_id = s.cust_id
GROUP  BY s.cust_id;

-- (B) 비파티션을 조인키 해시로 재분배하여 파티션쪽으로 "맞춘다"
EXPLAIN PLAN FOR
SELECT /*+ leading(s) use_hash(o)
           parallel(s 16) parallel(o 16)
           pq_distribute(o HASH HASH)
           monitor */
       s.cust_id, SUM(s.amount), SUM(o.amt)
FROM   fact_sales s
JOIN   orders_np  o
  ON   o.cust_id = s.cust_id
GROUP  BY s.cust_id;

SELECT *
FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'BASIC +PARALLEL +NOTE +ALIAS'));
```
**핵심**
- (B)는 `orders_np`를 **해시 재분배**해 `fact_sales`의 **파티션과 유사한 구간**으로 맞춰 **교차 통신**을 줄입니다.
- **스큐**가 있으면 여전히 불균형 → **SALT/복합키**로 분산도 개선.

---

# 튜닝 사례(현장형)

## 해시-해시 vs 브로드캐스트 선택

**문제**: 차원 테이블이 **작아 보이지만** 통계가 엉켜 옵티마이저가 **HASH-HASH**를 선택 → 네트워크/큐잉 폭증
**해결**
```sql
SELECT /*+ leading(d) use_hash(f)
           parallel(d 8) parallel(f 16)
           pq_distribute(d BROADCAST NONE) monitor */
       SUM(f.amount)
FROM   dim_customer d
JOIN   fact_sales   f
  ON   f.cust_id = d.cust_id
WHERE  d.active_yn = 'Y';
```
- `dim_customer`를 **브로드캐스트**로 강제하여, `fact_sales`는 **재분배 없이** 지역 조인.
- 이후 `V$PQ_TQSTAT`에서 `PX SEND BROADCAST` 바이트가 **작고**, `PX Deq Credit: send blkd`가 **감소**했는지 확인.

---

## 스큐 키(편중)로 인한 슬레이브 과부하

**증상**: `PX Deq Credit: send blkd`가 상위 1위, `v$pq_tqstat`에서 **num_rows 편차**가 매우 큼
**해결(두 단계 집계 예시)**
```sql
-- ① 해시 분산용 SALT 칼럼을 만들어 내부 분배 균등화
WITH S AS (
  SELECT cust_id,
         amount,
         MOD(ORA_HASH(cust_id), 8) salt  -- 8-way salt
  FROM   fact_sales
)
SELECT /*+ parallel(16) monitor */
       cust_id, SUM(amount)
FROM (
  SELECT /* partial aggregation by (cust_id, salt) */
         cust_id, salt, SUM(amount) amount
  FROM   S
  GROUP  BY cust_id, salt
)
GROUP  BY cust_id;
```
- **PX 재분배 → 부분 집계 → 최종 집계**로 바이트/큐잉을 안정화.
- 조인에서도 동일 아이디어 적용 가능(내부에서 `cust_id, salt`로 분산 후 바깥에서 다시 합산).

---

## RAC에서 GC 대기 급증

**증상**: 병렬 조인 시 `gc cr request`, `gc current request` 대기 많음
**전략**: **Full Partition-Wise Join + 서비스-파티션 affinity**
```sql
-- 둘 다 HASH(cust_id) 동일 파티션 → PARTITION PARTITION
SELECT /*+ leading(s) use_hash(c)
           parallel(s 16) parallel(c 16)
           pq_distribute(s PARTITION PARTITION)
           pq_distribute(c PARTITION PARTITION) monitor */
       COUNT(*)
FROM fact_sales s JOIN fact_clicks c ON c.cust_id = s.cust_id;

-- RAC: 서비스(Instance)별로 파티션(또는 서브파티션) 범위를 배치
-- 세션을 서비스 X로 붙이고, 그 서비스가 담당하는 파티션 범위만 읽도록 WHERE 절/파티션 프루닝 설계
```
- **재분배 최소화**로 **인스턴스 간 블록 이동↓** → GC 대기 급감.

---

## Partial PWJ에서 비파티션 쪽 반복 스캔

**증상**: 파티션 쪽은 빠르지만, 비파티션 큰 테이블이 **모든 파티션과 반복 조인**되어 I/O 과다
**해결(선결 조건 필터 → 브로드캐스트 전환)**
```sql
-- 비파티션 orders_np에 강한 필터(최근 한 달 등)를 먼저 적용해 충분히 소형화
WITH O AS (
  SELECT /*+ materialize */ *
  FROM   orders_np
  WHERE  order_dt >= ADD_MONTHS(TRUNC(SYSDATE,'MM'), -1)
)
SELECT /*+ leading(O) use_hash(s)
           parallel(s 16) parallel(O 8)
           pq_distribute(O BROADCAST NONE) monitor */
       s.cust_id, SUM(s.amount), SUM(O.amt)
FROM   O
JOIN   fact_sales s
  ON   s.cust_id = O.cust_id
GROUP  BY s.cust_id;
```
- **소형화된** 비파티션 집합을 **브로드캐스트**로 전환 → 반복 조인/재분배 비용 제거.

---

## ORDER BY/ROLLUP 동반 쿼리에서 정렬 대기

정렬/집계가 큰 경우 `direct path write/read temp` 증가. **분배 전략 + Top-N/Pushdown** 병행.
```sql
-- 조인 후 집계 + 정렬: 작은 쪽 브로드캐스트로 재분배 비용 제거
SELECT /*+ leading(d) use_hash(f)
           parallel(d 8) parallel(f 16)
           pq_distribute(d BROADCAST NONE) monitor */
       f.cust_id, SUM(f.amount) amt
FROM   dim_customer d JOIN fact_sales f
  ON   f.cust_id = d.cust_id
WHERE  d.active_yn = 'Y'
GROUP  BY f.cust_id
ORDER  BY amt DESC FETCH FIRST 100 ROWS ONLY;  -- Top-N으로 Temp 부담 축소
```

---

# **실행계획/동작** 검증 루틴

## 플랜에서 꼭 볼 것

```sql
SELECT *
FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'BASIC +PARALLEL +ALIAS +PREDICATE +NOTE'));
```
- **`PX SEND`** 라인: `HASH`/`BROADCAST`/`PARTITION`가 **정말 선택**됐는지
- **`IN-OUT`**: `P->P`(재분배), `P->S`(QC로 반환), `PCWP/PCWC`(동일 슬레이브 집합 결합)
- **NOTE**: *partition-wise join* 표기 등

## 분배/스큐 확인

```sql
SELECT dfo_number, tq_id, server_type, inst_id, process,
       num_rows, bytes
FROM   v$pq_tqstat
ORDER  BY dfo_number, tq_id, server_type, inst_id, process;
```
- **최대/최소 `num_rows` 편차**가 크면 **스큐**.
- 브로드캐스트는 **작은 바이트**로 고르게 복제되는지 확인.

## 대기 이벤트 요약(ASH 예시)

```sql
SELECT event, COUNT(*) samples
FROM   v$active_session_history
WHERE  sample_time > SYSDATE - 1/288   -- 최근 5분
AND    session_type = 'FOREGROUND'
AND    (event LIKE 'PX Deq%' OR event LIKE 'direct path%' OR event LIKE 'gc %')
GROUP  BY event
ORDER  BY samples DESC;
```
- `PX Deq Credit: send blkd`(역압), `direct path write/read temp`(Temp), `gc %`(RAC) 추적.

---

# **주의/함정** 체크리스트

1. **통계가 틀리면** 잘못된 분배를 강제할 수 있다
   - 브로드캐스트 대상이 실제로 **크면** 복제 비용이 폭증
   - 먼저 **필터/프루닝**으로 실제 전송량을 줄이고, 통계를 **갱신**
2. **스큐 키**
   - 해시-해시에서 **편중**은 **한 PX**를 멈춘다 → `SALT`/복합키로 분산도 확보
3. **파티션-와이즈 전제**
   - **동일 파티션 방식/경계/키**가 아니면 효과 없음(Partial/Dynamic으로 절충)
4. **조인 순서**
   - `LEADING`, `USE_HASH/USE_MERGE` 등으로 **조인 순서**가 원하는 분배 전략과 **맞물리게**
5. **DOP 과대**
   - DOP가 과도하면 Temp/네트워크/큐잉 경합 ↑. 스토리지/CPU/네트워크 **밸런스** 맞추기
6. **RAC**
   - PWJ + **서비스-파티션 affinity** 없으면 GC 대기가 되살아남

---

# 요약 표 — 언제 무엇을 강제할까?

| 상황 | 힌트(예) | 기대 효과 |
|---|---|---|
| 소형 차원 vs 대형 사실 | `pq_distribute(dim BROADCAST NONE)` | 해시 재분배 제거, 네트워크/큐잉↓ |
| 대형↔대형(균등 분배) | `pq_distribute(a HASH HASH)` + `pq_distribute(b HASH HASH)` | 동일 키 수렴, 스루풋↑ |
| Full PWJ 가능(동일 파티션) | `pq_distribute(t PARTITION PARTITION)`(양쪽) | 재분배 최소, RAC GC↓, 선형 확장 |
| Partial PWJ + 보정 | `pq_distribute(np HASH HASH)` | 비파티션을 키 해시로 정렬, 교차 통신↓ |
| 스큐 심함 | (해시 전략 유지 +) SALT/복합키 + 부분집계 | 슬레이브 과부하 해소 |
| 정렬/Top-N 대시보드 | (조인 브로드캐스트) + `FETCH FIRST n` | Temp/네트워크 절감, 응답 빠름 |

---

## 결론

- **PQ_DISTRIBUTE**는 병렬 실행에서 **데이터가 어디로 어떻게 이동**하는지 **직접 조종**하는 힌트입니다.
- 네 가지 키워드( **HASH / BROADCAST / PARTITION / NONE** )만 정확히 써도,
  **대형↔대형**, **소형↔대형**, **RAC**, **스큐**, **Partial/Dynamic** 등 **대부분의 현장 케이스**를 **의도한 흐름**으로 안정화할 수 있습니다.
- 항상 **실행계획(`PX SEND/RECEIVE`, `IN-OUT`) + V$PQ_TQSTAT 스큐 + 대기 이벤트**를 함께 확인하세요. 그게 곧 정답입니다.
