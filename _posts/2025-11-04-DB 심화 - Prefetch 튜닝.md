---
layout: post
title: DB 심화 - Prefetch 튜닝
date: 2025-11-04 18:25:23 +0900
category: DB 심화
---
# Prefetch 튜닝 — **인덱스 프리페치**와 **테이블 프리페치**

> **핵심 요약**
> - **인덱스 프리페치(Index Prefetch)**: 인덱스 **범위 스캔** 시 **다음(이웃) 리프 블록**을 **읽어둘 준비(리드어헤드/멀티블록·병렬 발주)** 를 해서 **연속 탐색**의 지연을 줄이는 최적화. 특히 **순방향 키 스캔**·**정렬 일치**·**클러스터링 팩터 양호**한 데이터에서 효과가 큼. 실행 시 주로 `db file scattered read`(인덱스 FFS) 또는 **배치된 단일 블록 읽기**(`db file parallel read`) 패턴으로 관찰됨.
> - **테이블 프리페치(Table Prefetch)**: 인덱스에서 수집한 **ROWID들을 모아** **중복 제거 및 정렬** 후 **테이블 블록을 배치로 미리 읽어두는**(= **Batched Table Access**) 최적화. 실행계획에 **`TABLE ACCESS BY INDEX ROWID BATCHED`** 로 나타나며, 대기 이벤트에 **`db file parallel read`** 가 보이는 경우가 많다. 목적은 **랜덤 I/O(단일 블록) 횟수와 왕복을 줄이는 것**.
>
> **효과 극대화 포인트**
> - (1) **인덱스 설계**(필터+정렬 일치, 커버링), (2) **클러스터링 팩터 개선**(CTAS 재적재), (3) **부분범위처리/Stopkey**, (4) **배열 페치/배치 바인딩**과 함께 쓰면 **IO 호출 수/대기 시간**이 눈에 띄게 감소한다.

---

## 실습 환경 준비

> 아래 스키마는 **인덱스 범위 스캔 → 테이블 접근** 경로에서 **프리페치 효과**를 관찰해보기 위한 예다.

```sql
-- 정리
DROP TABLE sales PURGE;
DROP TABLE dim_customer PURGE;

-- 본문 테이블(수십~수백만 행 가정)
CREATE TABLE sales (
  sale_id     NUMBER PRIMARY KEY,
  cust_id     NUMBER NOT NULL,
  sale_dt     DATE   NOT NULL,
  amount      NUMBER(12,2) NOT NULL,
  status      VARCHAR2(8)  NOT NULL,
  pad         VARCHAR2(100)
);

-- 고객
CREATE TABLE dim_customer (
  cust_id     NUMBER PRIMARY KEY,
  region      VARCHAR2(8),
  grade       VARCHAR2(8)
);

-- 샘플 데이터(데모 규모는 적당히)
INSERT /*+ APPEND */ INTO sales
SELECT level                               AS sale_id,
       MOD(level, 500000)+1                AS cust_id,
       DATE '2025-01-01' + MOD(level, 365) AS sale_dt,
       ROUND(DBMS_RANDOM.VALUE(10,1000),2) AS amount,
       CASE MOD(level,4)
            WHEN 0 THEN 'OK'
            WHEN 1 THEN 'OK'
            WHEN 2 THEN 'CXL'
            ELSE 'HOLD' END                AS status,
       RPAD('x',100,'x')                   AS pad
FROM dual
CONNECT BY level <= 1500000;
COMMIT;

INSERT INTO dim_customer
SELECT level, CASE MOD(level,4)
                 WHEN 0 THEN 'APAC'
                 WHEN 1 THEN 'EMEA'
                 WHEN 2 THEN 'AMER'
                 ELSE 'OTHR' END,
              CASE MOD(level,3)
                 WHEN 0 THEN 'VIP'
                 WHEN 1 THEN 'GOLD'
                 ELSE 'SILV' END
FROM dual CONNECT BY level <= 500000;
COMMIT;

-- 인덱스 설계: 필터+정렬 일치 & 커버링 후보
CREATE INDEX ix_sales_cust_dt_amt
  ON sales(cust_id, sale_dt DESC, sale_id DESC, amount);

CREATE INDEX ix_sales_status ON sales(status);

BEGIN
  DBMS_STATS.GATHER_TABLE_STATS(USER,'SALES',cascade=>TRUE);
  DBMS_STATS.GATHER_TABLE_STATS(USER,'DIM_CUSTOMER',cascade=>TRUE);
END;
/
```

---

## 인덱스 프리페치(Index Prefetch)

### 개념

- **목표**: 인덱스 리프를 **키 순서대로 쭉 훑을 때** 매번 “다음 리프 블록”을 기다리느라 생기는 **왕복 지연**을 줄이기.
- **방법**: 엔진이 **인접 리프 블록** 또는 **연속 리프 범위**를 **멀티블록/병렬로 미리 발주**(read-ahead)하여,
  커서가 다음 키로 진행할 때 이미 **버퍼 캐시** 또는 **세션 IO 큐**에 블록이 존재하도록 만든다.
- **관찰 포인트**
  - 인덱스가 **Fast Full Scan**으로 선택되면 **멀티블록**(`db file scattered read` / `direct path read`) 기반.
  - **범위 스캔**에서도 상황에 따라 **여러 단일블록 IO를 묶어** `db file parallel read` 로 보일 수 있다(플랫폼/버전 의존·오라클 내부 최적화).

> 직관: **인덱스 키 순서**가 **물리 리프 순서**에 가깝고, 조건이 **연속 범위**라면 **프리페치로 효과**가 더 커진다.

### 실습 1 — 인덱스 범위 스캔 + Top-N(Stopkey)

```sql
-- "고객별 최근 매출 Top-100" : 스캔 키가 정렬 인덱스와 일치 → 인접 리프 순회
SELECT /*+ index(s ix_sales_cust_dt_amt) */
       s.sale_id, s.sale_dt, s.amount
FROM   sales s
WHERE  s.cust_id = :cust
ORDER  BY s.sale_dt DESC, s.sale_id DESC
FETCH FIRST 100 ROWS ONLY;
```

- **왜 프리페치가 잘 먹히나?**
  - 인덱스 정의가 **(cust_id, sale_dt DESC, sale_id DESC)** 이고, 정렬도 동일 → **리프 체인 순회가 연속적**.
  - 엔진은 다음 리프 블록을 **미리 발주**해서 커서 전진 시 대기 최소화.
- **관찰**:
  - `DBMS_XPLAN.DISPLAY_CURSOR` 에서 `INDEX RANGE SCAN` + `STOPKEY` 확인.
  - 세션 이벤트에 `db file sequential read`(단일) 비중이 여전히 보이지만, **호출 간 평균 대기**가 작아지고,
    상황에 따라 `db file parallel read` 로 **여러 블록 발주** 흔적이 함께 나타날 수 있다.

### 실습 2 — 인덱스 Fast Full Scan(FFS)로 읽기량 자체 줄이기

```sql
-- 상태별 건수: 테이블 컬럼을 방문하지 않아도 된다면 인덱스만 훑기
SELECT /*+ index_ffs(s ix_sales_status) */
       s.status, COUNT(*)
FROM   sales s
GROUP  BY s.status;
```

- **FFS 특징**: 인덱스 리프 블록을 **멀티블록/순차**로 **쓸어 담음** → **인덱스 프리페치가 사실상 기본**.
- **장점**: **테이블 BY ROWID 접근 자체가 없음** → 이후 테이블 프리페치 필요도 없음.
- **관찰**: `db file scattered read` 또는 `direct path read` 비중↑, `db file sequential read`↓.

### 인덱스 프리페치 효과를 **극대화**하는 조건

- **정렬을 인덱스로 해결**(ORDER BY와 인덱스 컬럼 순서 일치) → **STOPKEY**로 앞부분만.
- **클러스터링 팩터**가 좋다(인덱스 키 순서가 물리적 근접) → **리프/테이블 블록 모두 근접**.
- **커버링 인덱스**면 테이블 방문 없음 → 인덱스 읽기만 최적화하면 끝.
- **카디널리티가 적절**(너무 드물지도, 너무 광범위하지도 않음)하고 **범위성이 있음**.

---

## — Batched Table Access

### 개념

- **문제 배경**: 인덱스 범위 스캔 후 **ROWID마다** 테이블 블록을 **한 블록씩** 랜덤 접근하면
  **`db file sequential read` 호출**이 폭증하고 지연이 누적된다.
- **해법**: 인덱스에서 ROWID를 **일단 모아서**(버퍼링), **중복 제거/정렬** 후,
  **해당 테이블 블록들을 한 번에(또는 몇 묶음으로) 미리 읽어두는** 전략.
  Oracle 실행계획은 이를 **`TABLE ACCESS BY INDEX ROWID BATCHED`** 로 표시한다.
- **관찰 포인트**
  - 대기 이벤트에 **`db file parallel read`**(여러 블록 동시 발주 후 기다림)가 **눈에 띄게 증가**할 수 있다.
  - 전체적인 **`db file sequential read` 호출 수가 감소**(왕복/락/컨텍스트 전환 ↓).

> 직관: **“ROWID 하나당 1 IO”** → **“ROWID 50~100개 모아 1~몇 번의 복합 IO”** 로 바꾸는 것.

### 실행계획에서 BATCHED 확인

```sql
-- 최근 N일 동안 특정 고객군의 상세 조회(행 수가 조금 많은 편)
SELECT s.sale_id, s.sale_dt, s.amount, s.status
FROM   sales s
JOIN   dim_customer c
  ON   c.cust_id = s.cust_id
WHERE  c.region = 'APAC'
AND    s.sale_dt >= SYSDATE - 14
ORDER  BY s.sale_dt DESC, s.sale_id DESC;
```

- **계획 확인**
```sql
SELECT *
FROM   TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST +PEEKED_BINDS'));
```
- 좋은 경우:
  - `INDEX RANGE SCAN` (sales 인덱스)
  - `TABLE ACCESS BY INDEX ROWID **BATCHED**` (sales)
- **이때** `V$SESSION_EVENT` / `V$SYSTEM_EVENT` 에서 `db file parallel read` 가 보이면,
  **테이블 프리페치가 동작**하며 **배치로 블록을 읽는 중**일 가능성이 크다.

### Batched가 **안** 잡힐 때의 원인과 대안

- **히스토그램/통계 오판**으로 옵티마이저가 **NL + 단건 접근**이 싸다고 믿는 경우
  → 통계를 정확히, 힌트(`LEADING`/`USE_HASH`/`FULL`/`INDEX`), 또는 SQL 재작성으로 후보 축소.
- **클러스터링 팩터가 나쁨**: ROWID가 너무 흩어져 있어 **배치 이득↓**
  → **CTAS 재적재**로 물리 순서 개선 또는 **IOT/클러스터 테이블** 검토.
- **부분범위처리/Stopkey 누락**: 너무 많은 로우를 가져오면 배치해도 총 IO↑
  → 정렬 포함 인덱스 + `FETCH FIRST N` 적용.
- **SELECT-LIST 과다**: 많은 열 때문에 **테이블 방문** 비중 자체가 높음
  → **커버링 인덱스**로 일부 질의를 “테이블 무방문”화.

---

## 인덱스·테이블 프리페치 **시나리오별** 실전 예

### + 상세 혼합 흐름

**목록(Top-N) — 인덱스 프리페치가 핵심**

```sql
-- 고객 화면 첫 페이지: 정렬 포함 인덱스 + Stopkey → 인덱스 리프 연속 스캔
SELECT /*+ index(s ix_sales_cust_dt_amt) */
       s.sale_id, s.sale_dt, s.amount
FROM   sales s
WHERE  s.cust_id = :cust
ORDER  BY s.sale_dt DESC, s.sale_id DESC
FETCH FIRST 50 ROWS ONLY;   -- STOPKEY
```

- 인덱스에서 필요한 앞부분만 **연속 스캔**.
- **인덱스 프리페치**로 **대기 최소화**, 테이블은 **커버링 인덱스 컬럼이면 무방문**.

**상세(선택된 50건) — 테이블 프리페치가 핵심**

```sql
-- 선택된 50건 상세(추가 컬럼 필요 → 테이블 방문)
SELECT /* 상세 페이지 */
       s.sale_id, s.sale_dt, s.amount, s.status, s.pad
FROM   sales s
WHERE  s.sale_id IN (:id1, :id2, ... :id50);
```

- 옵티마이저가 **IN LIST** 를 정렬된 ROWID로 묶어 **BATCHED** 테이블 접근을 시도.
- **`TABLE ACCESS BY INDEX ROWID BATCHED`** 확인 → **`db file parallel read`** 증가 경향.

### 조건이 넓은 범위(배치 이득 vs 해시 조인 비교)

```sql
-- 최근 3개월, 특정 등급 고객의 상세 목록 (로우가 제법 많음)
SELECT s.sale_id, s.cust_id, s.sale_dt, s.amount
FROM   sales s
JOIN   dim_customer c
  ON   c.cust_id = s.cust_id
WHERE  s.sale_dt >= ADD_MONTHS(TRUNC(SYSDATE,'MM'), -3)
AND    c.grade = 'GOLD'
ORDER  BY s.sale_dt DESC, s.sale_id DESC
FETCH FIRST 1000 ROWS ONLY;
```

- **옵션 A**: 인덱스 범위 스캔 + **BATCHED ROWID** → **테이블 프리페치**로 랜덤 IO 감소
- **옵션 B**: **해시 조인 + Partition/Full Scan** → **멀티블록 순차 IO** 중심
- **실측**으로 **어느 쪽이 더 빠른지** 판단(데이터 분포/스토리지 특성 영향 큼).

---

## 프리페치가 **잘 먹히는 구조** 만들기

### 정렬 일치 & Stopkey

- ORDER BY와 인덱스 컬럼 순서가 일치하면, **인덱스 리프 체인**을 **한 방향**으로 순회 가능 → **리드어헤드** 효율↑.
- Stopkey로 **앞부분만** 필요하면, 프리페치로 **짧은 구간**을 **끊김 없이** 가져와 **왕복 최소화**.

### 커버링 인덱스

- SELECT-LIST/필터/정렬 열이 인덱스에 **다 들어있으면**, **테이블 방문 제거** → **테이블 프리페치 자체가 불필요**.
- 커버링이 어려우면, 적어도 **테이블 방문을 작은 세트**에만 하도록 **뷰 머지 방지 + Stopkey**.

### 클러스터링 팩터 개선

- 같은 키(또는 인접 키)의 로우가 **근접 블록**에 모여 있으면,
  **BATCHED ROWID** 가 **적은 IO로 많은 로우**를 끌어온다.
```sql
-- 재적재(CTAS)로 물리 순서 개선
CREATE TABLE sales_sorted NOLOGGING AS
SELECT * FROM sales
ORDER BY cust_id, sale_dt, sale_id;

ALTER TABLE sales RENAME TO sales_old;
ALTER TABLE sales_sorted RENAME TO sales;

-- 인덱스 재생성 후 통계 수집
DROP INDEX ix_sales_cust_dt_amt;
CREATE INDEX ix_sales_cust_dt_amt
  ON sales(cust_id, sale_dt DESC, sale_id DESC, amount);

BEGIN
  DBMS_STATS.GATHER_TABLE_STATS(USER,'SALES',cascade=>TRUE);
END;
/
```

### IOT/클러스터 테이블

- **IOT**: PK 기반 조회는 **데이터 자체가 인덱스 구조** → **랜덤 IO 최소화**, 프리페치 효과도 높음.
- **클러스터 테이블**: 동일 클러스터 키의 로우가 **같은 블록/근접 블록** → BATCHED가 **큰 이득**.

---

## 프리페치 관찰·진단·계량

### 실행계획

```sql
SELECT *
FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST +PEEKED_BINDS +ALIAS'));
```
- **중요 노드**
  - `INDEX RANGE SCAN ...` + `STOPKEY` (인덱스 프리페치 기대)
  - `INDEX FAST FULL SCAN ...` (멀티블록 기반 인덱스 프리페치)
  - `TABLE ACCESS BY INDEX ROWID **BATCHED**` (테이블 프리페치)

### 대기 이벤트/통계

```sql
-- 세션/시스템 이벤트: 배치가 잘 되면 아래 경향이 보임
SELECT event, total_waits, time_waited_micro/1e6 AS sec
FROM   v$system_event
WHERE  event IN ('db file sequential read', 'db file scattered read', 'db file parallel read', 'direct path read')
ORDER  BY sec DESC;

-- SQL별 I/O 프로파일
SELECT sql_id, plan_hash_value, executions,
       buffer_gets, disk_reads,
       ROUND(disk_reads/NULLIF(executions,0)) avg_disk
FROM   v$sql
ORDER  BY avg_disk DESC FETCH FIRST 20 ROWS ONLY;
```
- **테이블 프리페치**가 동작하면, **`db file sequential read` 호출 수 감소**, **`db file parallel read` 일부 증가** 경향.

### ASH로 최근 분포

```sql
SELECT event, COUNT(*) samples
FROM   v$active_session_history
WHERE  sample_time > SYSTIMESTAMP - INTERVAL '10' MINUTE
AND    session_type='FOREGROUND'
AND    event IN ('db file sequential read','db file parallel read','db file scattered read')
GROUP  BY event
ORDER  BY samples DESC;
```

---

## 프리페치 관련 **자주 하는 질문(FAQ)**

### Q1. 프리페치를 강제하는 힌트가 있나요?

- **직접적인 “프리페치 힌트”**는 거의 없다.
- 대신 프리페치가 **잘 일어나는 경로**(정렬 일치 인덱스 범위 스캔 + Stopkey, 인덱스 FFS, Batched Table Access)가
  **자연스럽게 선택되도록** 힌트/통계/SQL 구조를 설계한다.

### Q2. 왜 `db file parallel read` 가 늘었는데 체감은 빨라지죠?

- 여러 단일블록 IO를 **한 묶음으로 발주** → **왕복 횟수·락/래치·컨텍스트 스위칭**이 줄어서 **총 지연이 감소**.
- **호출 수**와 **평균 대기**를 함께 보자. **총 시간**이 줄었으면 **성공**이다.

### Q3. 항상 BATCHED가 좋은가요?

- 대부분의 “많은 로우를 읽는 인덱스 경로”에서 유리.
- 하지만 **해시 조인 + Full Scan(멀티블록)** 이 더 나은 경우도 많다(특히 **넓은 범위**).
- **실측**으로 비교하자.

---

## 안티패턴 → 교정

| 안티패턴 | 증상 | 교정 |
|---|---|---|
| 정렬이 인덱스와 불일치 | SORT + 많은 ROWID → 랜덤 IO 폭증 | **정렬 일치 인덱스** 설계 + **Stopkey** |
| 커버링 부족 | 테이블 BY ROWID 과다 | **커버링 인덱스** 또는 SELECT-LIST 축소 |
| 클러스터링 팩터 나쁨 | BATCHED 이득 적음 | **CTAS 재적재**로 물리 순서 개선 |
| 프루닝 실패 | 읽기량 과다 | 파티션 키 조건 명확화, 함수 적용 금지 |
| 뷰 병합으로 조기 함수 평가 | 대량 행에 함수 호출 | **NO_MERGE** + 인라인뷰 Stopkey 후 **소수 행**에만 함수 |

---

## 전/후 비교 스크립트(측정 루틴)

```sql
ALTER SESSION SET statistics_level = ALL;
ALTER SESSION SET events '10046 trace name context forever, level 8';

-- BEFORE: 정렬 불일치 + OFFSET (나쁨)
SELECT s.sale_id, s.sale_dt, s.amount
FROM   sales s
WHERE  s.cust_id = :cust
ORDER  BY s.amount DESC
OFFSET :skip ROWS FETCH NEXT :take ROWS ONLY;

-- AFTER: 정렬 일치 인덱스 + Stopkey (인덱스 프리페치 기대)
SELECT /*+ index(s ix_sales_cust_dt_amt) */
       s.sale_id, s.sale_dt, s.amount
FROM   sales s
WHERE  s.cust_id = :cust
ORDER  BY s.sale_dt DESC, s.sale_id DESC
FETCH FIRST :take ROWS ONLY;

-- 상세: IN LIST → BATCHED TABLE ACCESS 기대
SELECT s.sale_id, s.sale_dt, s.amount, s.status, s.pad
FROM   sales s
WHERE  s.sale_id IN (:id1, :id2, :id3, ...);

ALTER SESSION SET events '10046 trace name context off';

-- 실행계획·대기 확인
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST'));
```

---

## 요약 체크리스트

- [ ] **ORDER BY** 와 **인덱스 순서** 일치 → **STOPKEY** 로 앞부분만?
- [ ] **커버링 인덱스** 로 테이블 방문을 없앨 수 있는가? (불가하면 소수 행만 테이블 방문)
- [ ] 실행계획에 **`BATCHED`** 가 잡혔는가? (없다면 플랜/통계를 점검)
- [ ] **클러스터링 팩터** 를 개선했는가? (Range Scan·BATCHED 품질↑)
- [ ] **해시 조인 + Full/Partition Scan** 과 **BATCHED 경로** 를 **실측 비교**했는가?
- [ ] 대기 이벤트에서 **`sequential read`↓**, **`parallel read`↗** + **총 시간↓** 인가?

---

## 결론

- **인덱스 프리페치**는 **연속 리프 접근**의 지연을 줄이고, **테이블 프리페치(BATCHED)** 는 **ROWID 랜덤 IO**를 **묶음 IO**로 바꾸어 **왕복·락·컨텍스트 전환**을 줄인다.
- 프리페치의 진짜 가치는 **설계**(정렬 일치 인덱스·커버링·클러스터링)와 **질의 구조**(Stopkey·뷰 머지 방지), **옵티마이저 유도**(힌트/통계)와 결합할 때 폭발한다.
- 항상 **실행계획/대기/IO 프로파일**을 보고 **전/후 실측**으로 검증하라. “프리페치가 보이는가?”보다 **응답시간이 줄었는가?**가 최종 판단 기준이다.
