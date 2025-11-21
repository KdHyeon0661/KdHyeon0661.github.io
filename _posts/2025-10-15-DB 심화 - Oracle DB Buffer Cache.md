---
layout: post
title: DB 심화 - Oracle DB Buffer Cache
date: 2025-10-15 18:25:23 +0900
category: DB 심화
---
# Oracle DB Buffer Cache — 블록 단위 I/O, Cache Buffer Chains, LRU Chains 완전 정리 (19c 기준)

> 목표: **버퍼 캐시 내부 구조**(블록/헤더), **Cache Buffer Chains(CBC)** 해시·래치, **LRU 체인**(터치 카운트 기반 냉/온 리스트)까지 “왜 그런 현상이 보이는지”를 **실습 쿼리**와 함께 설명합니다.
> 환경 가정: Oracle 19c, 블록 크기 8KB(기본), 단일 인스턴스(필요 시 RAC 주석), 권한: 일부 X$ 뷰는 SYS 권한 필요.

---

## 큰 그림 — “블록” 단위 I/O와 캐시 생태계

오라클 **버퍼 캐시(Buffer Cache)** 는 디스크의 **데이터파일 블록**을 **메모리**에 적재해 **논리적 I/O**(consistent/current)로 빠르게 접근하게 하는 영역입니다.
핵심 키워드:

- **버퍼(Buffer Frame)**: 한 개의 **데이터베이스 블록**을 담는 메모리 슬롯(보통 8KB).
- **버퍼 헤더(Buffer Header)**: 이 프레임을 설명하는 메타데이터(파일#, 블록#, 상태, SCN, 핀(pin) 카운트, 터치(tch) 카운트 등).
- **해시 버킷(Hash Buckets)** + **Cache Buffer Chains(CBC) 래치**: “(파일#, 블록#)” → 해시 → **체인**(링크드 리스트)에서 **버퍼 헤더**를 탐색/고정.
- **LRU 체인**: 오래 안 쓰인 버퍼를 내보내고(free/make free) 자주 쓰인 버퍼를 남기는 **냉/온 리스트**(터치 카운트로 근사).
- **Dirty 리스트**: 변경(Dirty)된 버퍼는 **DBWn**이 적절한 시점에 디스크로 기록.
- **CR(Current/Consistent Read)**: 쿼리 **SCN** 기준의 일관 읽기를 위해 **Undo** 로 **CR 버퍼** 재구성.

> **논리 I/O**는 **버퍼 캐시**에서의 블록 접근, **물리 I/O**는 디스크에서 블록을 읽어오는 작업입니다.
> 캐시 히트율은 참고 지표일 뿐(“무조건 높을수록 좋다”는 오해), **상위 대기 이벤트/경합** 기반으로 병목을 판단해야 합니다.

---

## 버퍼 캐시 내부 구조 — “버퍼 프레임 + 버퍼 헤더”

### 버퍼 프레임(Buffer Frame)과 클래스

- 프레임 한 칸이 **DB 블록(8KB 등)** 하나를 담습니다.
- 프레임은 **상태**(free, xcur, scur, cr 등)와 **클래스**(data block, segment header, undo 등)로 구분됩니다.
- Oracle은 내부적으로 **클래스별 LRU 통계**를 추적합니다.

```sql
-- 프레임 요약/상태 확인
SELECT status, COUNT(*) AS cnt
FROM   v$bh
GROUP  BY status;

-- 클래스(블록 종류) 분포
SELECT class, COUNT(*) AS cnt
FROM   v$bh
GROUP  BY class
ORDER  BY cnt DESC;
```

- **status** 대략:
  - **xcur**: Exclusive Current(현재 변경 소유),
  - **scur**: Shared Current(공유 읽기),
  - **cr**: Consistent Read(Undo 기반 snapshot),
  - **free**: 할당 가능,
  - **read**: 디스크에서 읽는 중 등.

> RAC에선 **gc current/cr** 상태와 **GCS** 통신이 추가됩니다(§7).

### — 무엇을 들고 있나?

- (파일#, 블록#), **SCN/버전**, **Dirty 여부**, **핀 카운트**(고정 중이면 제거 금지), **Touch Count(tch)**, **체인 포인터** 등.
- **버퍼 헤더**는 **해시 버킷 체인**에 연결되며, **LRU 체인**에도 링크로 연결됩니다.

```sql
-- X$BH는 내부(비공식) 뷰. SYS 권한 필요.
-- 터치 카운트(tch)가 높은 블록(핫 블록) 살펴보기
SELECT /*+ RULE */ file#, dbablk, tch
FROM   x$bh
WHERE  tch > 100
ORDER  BY tch DESC FETCH FIRST 20 ROWS ONLY;
```

> `tch`(touch count)는 “최근 참조 빈도”의 근사치입니다. **핫 블록** 진단에 유용합니다.

---

## — 해시 버킷과 래치 경합

### CBC 동작 원리

1. **(파일#, 블록#)** 조합을 **해시** → **해시 버킷** 선택.
2. 해당 버킷의 **체인(링크드 리스트)** 에서 **버퍼 헤더**를 탐색.
3. 탐색/수정 동안 **CBC 래치**(latch: cache buffers chains)로 **단기 보호**.
4. 찾으면 **버퍼 핀(pin)** → 데이터 접근, 필요 시 **CR 재구성**.

> **장점**: O(1)에 근접한 탐색.
> **단점**: **특정 버킷/블록에 접근이 집중**되면 **래치 경합**.

### CBC 경합이 왜 생기나? (Hot Block 패턴)

- **단일 수치 증가 카운터 테이블**(모든 트랜잭션이 같은 블록/행 갱신)
- **편향된 인덱스 액세스**(예: 증가하는 시퀀스로 **인덱스 한쪽 리프**가 과열)
- **작은 룩업 테이블**의 고정된 몇 개 행만 반복 조회(특정 블록만 과히트)

```sql
-- CBC 래치 통계 관측 (상위 래치 미스)
SELECT name, gets, misses, sleep
FROM   v$latch
WHERE  LOWER(name) LIKE '%cache buffers chain%'
ORDER  BY misses DESC;

-- 세션 수준 현재 대기(래치 대기 포함)
SELECT sid, event, state, seconds_in_wait
FROM   v$session
WHERE  state <> 'WAITED SHORT TIME'
AND    (event LIKE 'latch:%' OR event LIKE 'buffer busy%');
```

> 과거/단일 인스턴스: `latch: cache buffers chains`
> RAC: 글로벌 캐시 관련 `gc buffer busy`로 나타나기도.

### CBC 경합 완화 전략(핵심)

- **액세스 분산**:
  - **Reverse Key Index** 로 **순차 삽입 핫 리프** 분산.
  - **Hash 파티셔닝**/**범위+해시 합성**으로 키 분포 분산.
  - 단일 **카운터 테이블** → **샤딩된 카운터**(N개 행) 사용 후 합산.
- **논리 I/O 저감**: 필요한 **커버링 인덱스** 설계, **필요 컬럼만 선택**, **Batch 처리**.
- **버퍼 캐시 크기 조정**: 너무 작으면 재활용 빈번 → 체인 스캔 기회 증가.
- **(신중)** 해시 버킷 증가(숨은 파라미터) 등은 **권장 X** — 릴리즈·환경별 리스크.

```sql
-- Reverse Key Index 예시 (운영 영향 확인 필수)
CREATE INDEX ix_sales_demo_rev ON sales_demo(sale_id) REVERSE;

-- Hash 파티셔닝 예시
CREATE TABLE counters (
  k NUMBER, v NUMBER,
  CONSTRAINT pk_counters PRIMARY KEY (k)
)
PARTITION BY HASH (k)
PARTITIONS 16;

-- 샤딩된 카운터 패턴(예시)
-- update counters set v = v + 1 where k = MOD(:session_id, 16);
```

---

## LRU 체인 — 냉/온 리스트와 터치 카운트

### 개념 업데이트: “순수 LRU”가 아니다

- 오래전 **단일 LRU**가 아니라, 현재는 **터치 카운트(tch)**, **핫/콜드 리스트**, **중간 삽입(midpoint)**, **버퍼 풀(KEEP/RECYCLE/DEFAULT)** 등 **복합 정책**을 사용합니다.
- 목적: **대용량 Full Scan** 같은 **일시적 접근**이 **핵심 워킹셋**을 **몰아내지 않도록**.

> 정확한 알고리즘은 버전에 따라 달라질 수 있고 **공식 문서에 완전 공개되지 않습니다**. 여기서는 **관찰로 추정 가능한 동작**을 설명합니다.

### 냉/온 리스트, midpoint 삽입, touch count

- 새로 읽은 블록은 **콜드(냉) 영역** 중간 부근에 삽입.
- **반복 참조**되면 **tch 증가**, 상대적으로 **온(핫) 영역**에 잔류.
- **배출(evict)** 은 주로 **콜드 말단**에서.
- 목적: **일시적 스캔**은 콜드로 흘러가다 쉽게 배출, **핵심 워킹셋**은 핫에 오래 남김.

```sql
-- LRU 관련 통계(버퍼 풀별)
SELECT * FROM v$buffer_pool;
SELECT * FROM v$buffer_pool_statistics;

-- Keep/Recycle 풀 크기/사용 확인
SHOW PARAMETER db_keep_cache_size;
SHOW PARAMETER db_recycle_cache_size;
```

### Keep/Recycle Buffer Pool — 워킹셋 힌팅

- **KEEP**: 항상 메모리에 남기고 싶은 “작은 핵심 테이블/인덱스”.
- **RECYCLE**: **일시적 접근**(예: 로그성 대테이블) → 캐시 오염 최소화.

```sql
-- 버퍼 풀 크기(초기화 파라미터)
ALTER SYSTEM SET db_keep_cache_size    = 512M SCOPE=BOTH;
ALTER SYSTEM SET db_recycle_cache_size = 256M SCOPE=BOTH;

-- 오브젝트를 KEEP 풀로
ALTER TABLE dim_branch STORAGE (BUFFER_POOL KEEP);
ALTER INDEX ix_dim_branch_code STORAGE (BUFFER_POOL KEEP);

-- 오브젝트를 RECYCLE 풀로
ALTER TABLE fact_tx STORAGE (BUFFER_POOL RECYCLE);
```

---

## CR(Consistent Read)와 버퍼 — 같은 블록의 “여러 버전”

### 왜 CR가 필요한가?

- 쿼리 시작 시 **SCN**을 고정. 읽는 블록의 변경 SCN이 크면 **Undo** 로 과거 버전을 재조립 → **CR 버퍼** 생성.
- 결과: **트랜잭션 격리/일관** 보장.

```sql
-- Undo 관련 파라미터/테이블스페이스
SHOW PARAMETER undo_tablespace;
SHOW PARAMETER undo_retention;

-- CR 접근 증가가 의심될 때(관찰 지표 예시)
SELECT name, value
FROM   v$sysstat
WHERE  name IN ('consistent gets', 'db block gets', 'physical reads');
```

### CR 버퍼 수명과 LRU

- **CR 버퍼**도 버퍼 캐시에 존재합니다. **일시적** 성격이 강해 LRU **콜드 측**에서 빨리 밀려나가기 쉬움.
- **장시간 쿼리 + 대규모 DML** → Undo 부족 시 **ORA-01555(snapshot too old)**.

---

## 실습: 눈으로 보는 버퍼 캐시

### 준비 — 샘플 테이블/인덱스

```sql
-- 샘플: 판매 테이블 (앞 절의 예시와 유사)
CREATE TABLE sales_demo (
  sale_id     NUMBER GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
  cust_id     NUMBER NOT NULL,
  sale_dt     DATE   NOT NULL,
  amount      NUMBER(12,2) NOT NULL,
  channel     VARCHAR2(20),
  payload     VARCHAR2(100),
  created_at  DATE DEFAULT SYSDATE
);

CREATE INDEX ix_sales_demo_cust ON sales_demo(cust_id);
CREATE INDEX ix_sales_demo_dt   ON sales_demo(sale_dt);

BEGIN
  FOR i IN 1..200000 LOOP
    INSERT INTO sales_demo (cust_id, sale_dt, amount, channel, payload)
    VALUES (TRUNC(DBMS_RANDOM.VALUE(1, 40000)),
            TRUNC(SYSDATE - DBMS_RANDOM.VALUE(0, 365)),
            ROUND(DBMS_RANDOM.VALUE(10, 1000), 2),
            CASE MOD(i,4) WHEN 0 THEN 'web' WHEN 1 THEN 'store' WHEN 2 THEN 'app' ELSE 'call' END,
            RPAD('x', 50, 'x'));
    IF MOD(i,2000)=0 THEN COMMIT; END IF;
  END LOOP;
  COMMIT;
END;
/
```

### 캐시 히트·물리 읽기 감각잡기

```sql
-- 수행 전 베이스라인
SELECT name, value
FROM   v$sysstat
WHERE  name IN ('db block gets','consistent gets','physical reads');

-- 인덱스 범위 스캔(논리 I/O 위주)
SELECT /*+ index(s ix_sales_demo_cust) */
       SUM(amount)
FROM   sales_demo s
WHERE  cust_id BETWEEN 1000 AND 2000;

-- 다시 통계 비교
SELECT name, value
FROM   v$sysstat
WHERE  name IN ('db block gets','consistent gets','physical reads');
```

> 인덱스 범위 스캔은 **단일 블록 읽기**가 많고 캐시 히트율이 비교적 높게 나옵니다.

### Full Scan이 캐시에 미치는 영향(RECYCLE 활용)

```sql
-- 대량 Full Scan
SELECT /*+ FULL(s) */ COUNT(*)
FROM   sales_demo s
WHERE  sale_dt BETWEEN ADD_MONTHS(TRUNC(SYSDATE),-6) AND SYSDATE;

-- 이후 v$bh 분포를 확인
SELECT class, status, COUNT(*) cnt
FROM   v$bh
GROUP  BY class, status
ORDER  BY cnt DESC;

-- Full Scan 대상 테이블을 RECYCLE로 보내 캐시 오염 완화
ALTER TABLE sales_demo STORAGE (BUFFER_POOL RECYCLE);
```

> Full Scan은 과거엔 `db file scattered read`, 11g+ 에선 조건에 따라 **direct path read** 경로도 탑니다(바이패스/부분적 사용). **RECYCLE 풀**은 워킹셋 보호에 도움.

### CBC 핫 블록 재현(소규모)과 경감

```sql
-- "카운터 1행" 패턴(비추천: 핫 블록의 전형)
CREATE TABLE t_counter (id NUMBER PRIMARY KEY, cnt NUMBER);
INSERT INTO t_counter VALUES (1, 0);
COMMIT;

-- 세션 A~N에서 동시에:
-- UPDATE t_counter SET cnt = cnt + 1 WHERE id = 1; COMMIT;
-- 이런 패턴은 CBC 래치 경합(또는 RAC에선 gc busy) 유발

-- 개선: 샤딩형 카운터 (16개의 버킷으로 분산)
CREATE TABLE t_counter_shard (k NUMBER PRIMARY KEY, cnt NUMBER);
BEGIN
  FOR i IN 0..15 LOOP
    INSERT INTO t_counter_shard VALUES (i, 0);
  END LOOP; COMMIT;
END;
/

-- 각 세션은 서로 다른 k 버킷만 업데이트 → 경합 급감
-- 최종 카운트는 SUM(cnt)로 조회
```

```sql
-- "증가 시퀀스 + 리프 한쪽 과열" 완화: Reverse Key Index
DROP INDEX ix_sales_demo_dt;
CREATE INDEX ix_sales_demo_dt ON sales_demo(sale_dt) REVERSE;

-- 또는 파티셔닝/해시 파티셔닝으로 키 분산
```

```sql
-- CBC 래치 관측(전/후 비교)
SELECT name, gets, misses, sleep
FROM   v$latch
WHERE  LOWER(name) LIKE '%cache buffers chain%';
```

### KEEP 풀로 핵심 룩업 고정

```sql
-- 소형 코드 테이블을 KEEP로 항상 메모리에
ALTER SYSTEM SET db_keep_cache_size = 256M SCOPE=BOTH;

CREATE TABLE dim_code (
  code VARCHAR2(20) PRIMARY KEY,
  name VARCHAR2(100)
);

INSERT INTO dim_code SELECT 'C'||LEVEL, 'Name'||LEVEL FROM dual CONNECT BY LEVEL <= 10000;
COMMIT;

ALTER TABLE dim_code STORAGE (BUFFER_POOL KEEP);

-- 자주 참조하는 보고서 쿼리에서 논리 I/O 히트가 개선되는지 관측
```

---

## 운영·튜닝 관점 — 어떤 지표를 보나?

### 상위 대기 이벤트·세션 관측

```sql
-- 시스템 Top Waits
SELECT * FROM (
  SELECT event, total_waits, time_waited_micro/1e6 AS sec_waited
  FROM   v$system_event
  ORDER  BY time_waited_micro DESC
) WHERE ROWNUM <= 20;

-- 세션 현재 대기
SELECT sid, serial#, username, event, wait_class, state, seconds_in_wait
FROM   v$session
WHERE  username IS NOT NULL
AND    state <> 'WAITED SHORT TIME'
ORDER  BY seconds_in_wait DESC;
```

- CBC 관련이면 **`latch: cache buffers chains`** 가 상위에 보일 수 있습니다.
- **`free buffer waits`**: LRU에서 **free 버퍼** 확보가 지연(버퍼 캐시 압박 / DBWn 지연).
- **`buffer busy waits`**: 버퍼 헤더 단의 다양한 경합(로드/쓰기 완료 대기 포함).
- **`read by other session`**: 다른 세션이 같은 블록을 디스크에서 읽는 중, 완료 대기.

### 버퍼 풀·LRU 상태

```sql
SELECT * FROM v$buffer_pool;
SELECT * FROM v$buffer_pool_statistics;

-- v$bh 분포/상태
SELECT class, status, COUNT(*) cnt
FROM   v$bh
GROUP  BY class, status
ORDER  BY cnt DESC;

-- X$BH로 tch 높은 핫 블록 탐지(SYS)
SELECT file#, dbablk, tch
FROM   x$bh
WHERE  tch >= 64
ORDER  BY tch DESC FETCH FIRST 50 ROWS ONLY;
```

### 수학적 감각 (지표는 참고용)

- **캐시 히트률(참고용)**
  $$ \text{Hit Ratio} \approx 1 - \frac{\text{physical reads}}{\text{db block gets} + \text{consistent gets}} $$
  > 히트율이 높아도 **CBC 경합**이나 **Undo/Redo 대기**가 병목이면 성능은 나빠질 수 있습니다.
- **Redo/Undo/Temp 상호영향**: 대량 DML은 **Redo 폭증** → LGWR/아카이브 부담, 대형 정렬/해시는 **PGA/Temp** 압박 → 간접적으로 LRU에도 영향.

---

## RAC 관점 한 줄 — 글로벌 캐시와 핫 블록

- RAC에서는 **같은 데이터베이스**를 **여러 인스턴스**가 공유. 블록 소유권 전환에 **GCS/GES** 개입.
- **핫 블록**은 **인스턴스 간** 지속 왕복으로 `gc buffer busy`, `gc cr request` 등의 **gc 대기**로 표출.
- 완화는 **데이터/워크로드 분산(서비스·파티셔닝)**, **인덱스 설계(Reverse/Hash)**, **시퀀스 캐시/증가폭 조정** 등 **단일 인스턴스와 유사**하되 **인스턴스별 국소성**을 추가 고려.

---

## 자주 하는 질문(FAQ)

**Q1. LRU 체인을 “강제로” 조정할 수 있나?**
- 직접 조정은 불가. 대신 **KEEP/RECYCLE 풀**, **오브젝트 설계**, **쿼리 패턴**을 조정해 **LRU 결과**를 바꿉니다.

**Q2. x$ 뷰로 많이 보면 되나?**
- **운영에선 최소화**가 원칙. 릴리즈별로 컬럼·의미가 달라질 수 있고 **지원 대상이 아닙니다**. **v$ 뷰와 AWR/ASH** 중심으로 관측하세요.

**Q3. 캐시 히트율이 99%인데도 느리다?**
- **정상입니다**. CBC 래치, Undo, Redo, Temp, I/O 대기 등 **다른 병목**을 확인해야 합니다.

**Q4. Full Scan은 항상 캐시를 오염시키나?**
- **아닙니다**. 버전/상황에 따라 **direct path read** 로 **부분 바이패스**합니다. 그래도 **RECYCLE/병렬/파티셔닝** 설계로 안전망을 깔아두는 게 좋습니다.

---

## 체크리스트 — 버퍼 캐시 튜닝 12선

1. **Top Waits** 에 `latch: cache buffers chains` 있는가?
2. **핫 블록**(x$bh.tch가 비정상 고카운트) 존재? 해당 오브젝트/인덱스 리프/루트 편향?
3. **Reverse Key Index / Hash 파티셔닝 / 샤딩 카운터** 등 액세스 분산 시도?
4. **KEEP/RECYCLE** 적절 적용? 워킹셋 보존되고 대테이블 오염 억제?
5. **db_cache_size** 충분? `free buffer waits` 발생 여부 확인.
6. **DBWn** 쓰기 지연? `write complete waits`/redo·checkpoint 빈발?
7. **Full Scan** 빈도/패턴 점검: 불필요한 SELECT * 제거/커버링 인덱스.
8. **바인드 변수** 사용으로 파싱/라이브러리 캐시 경합 완화.
9. **대형 정렬/해시**로 Temp 남용? PGA 타깃/쿼리 리라이트/배치 튜닝.
10. **Undo 보존/용량** (`undo_retention`, TS 사이즈) 점검(ORA-01555 방지).
11. RAC면 **인스턴스 로컬리티**(서비스 라우팅) 설계.
12. **AWR/ASH** 로 “사실 기반” 원인 규명 → 처방.

---

## 통합 실습 시나리오 — “문제 만들고, 관측하고, 고친다”

### CBC 핫 블록 만들기

```sql
-- 1) 단일-카운터(나쁜 예시)
DROP TABLE t_counter PURGE;
CREATE TABLE t_counter (id NUMBER PRIMARY KEY, cnt NUMBER);
INSERT INTO t_counter VALUES (1, 0);
COMMIT;

-- 2) 동시 업데이트(여러 세션에서 반복)
-- UPDATE t_counter SET cnt = cnt + 1 WHERE id = 1; COMMIT;

-- 3) 관측
SELECT name, gets, misses, sleep
FROM   v$latch
WHERE  LOWER(name) LIKE '%cache buffers chain%';

-- (가능시) X$BH에서 tch 상위 블록
SELECT file#, dbablk, tch
FROM   x$bh
ORDER  BY tch DESC FETCH FIRST 10 ROWS ONLY;
```

### 개선: 샤딩 카운터 + 커버링 인덱스

```sql
DROP TABLE t_counter_shard PURGE;
CREATE TABLE t_counter_shard (
  k   NUMBER PRIMARY KEY,
  cnt NUMBER
);

BEGIN
  FOR i IN 0..63 LOOP
    INSERT INTO t_counter_shard VALUES (i, 0);
  END LOOP; COMMIT;
END;
/

-- 각 세션은 서로 다른 k에 업데이트
-- UPDATE t_counter_shard SET cnt = cnt + 1 WHERE k = :my_bucket; COMMIT;

-- 최종 합계
SELECT SUM(cnt) FROM t_counter_shard;

-- CBC 래치 미스/슬립이 감소했는지 비교
SELECT name, gets, misses, sleep
FROM   v$latch
WHERE  LOWER(name) LIKE '%cache buffers chain%';
```

### Full Scan 오염 완화: RECYCLE & 보고서 쿼리 분리

```sql
-- 대테이블을 RECYCLE로
ALTER TABLE sales_demo STORAGE (BUFFER_POOL RECYCLE);

-- 보고서용 소테이블을 KEEP로
ALTER SYSTEM SET db_keep_cache_size = 256M SCOPE=BOTH;
ALTER TABLE dim_code STORAGE (BUFFER_POOL KEEP);

-- 보고서 쿼리/일괄 배치 분리 후 v$bh, v$buffer_pool_statistics로 효과 관측
SELECT * FROM v$buffer_pool_statistics;
```

### CR/Undo 관측: 장시간 쿼리 + 동시 DML

```sql
-- 세션 A: 장시간 집계
SELECT /*+ FULL(s) PARALLEL(4) */ channel, SUM(amount)
FROM   sales_demo s
GROUP  BY channel;

-- 세션 B: 대규모 UPDATE
UPDATE sales_demo SET amount = amount * 1.1 WHERE sale_dt >= TRUNC(SYSDATE) - 180;
COMMIT;

-- Undo 보존/용량이 부족하면 세션 A에서 ORA-01555 위험
SHOW PARAMETER undo_retention;
```

---

## 참고 SQL 묶음 — 진단 단축키

```sql
-- 11.1 v$bh: 오브젝트별 블록 분포
SELECT /*+ LEADING(b) USE_NL(o) */
       o.owner, o.object_name, o.object_type, b.status, COUNT(*) AS blocks
FROM   v$bh b
JOIN   dba_objects o
  ON   o.data_object_id = b.objd
GROUP  BY o.owner, o.object_name, o.object_type, b.status
ORDER  BY blocks DESC, o.object_name;

-- 11.2 latch 상위
SELECT name, gets, misses, sleeps, round(misses/nullif(gets,0),6) AS miss_ratio
FROM   v$latch
ORDER  BY misses DESC;

-- 11.3 buffer busy / read by other session 상위 세그먼트
SELECT /*+ RULE */
       o.owner, o.object_name, o.object_type, SUM(s.total_waits) AS waits
FROM   v$segment_statistics s
JOIN   dba_objects o
  ON   o.owner = s.owner AND o.object_name = s.object_name
WHERE  s.statistic_name IN ('buffer busy waits','read by other session')
GROUP  BY o.owner, o.object_name, o.object_type
ORDER  BY waits DESC FETCH FIRST 20 ROWS ONLY;

-- 11.4 버퍼 풀 통계
SELECT * FROM v$buffer_pool_statistics;

-- 11.5 물리 읽기 핫 세그먼트
SELECT owner, object_name, object_type, SUM(physical_reads) pr
FROM   v$segment_statistics
GROUP  BY owner, object_name, object_type
ORDER  BY pr DESC FETCH FIRST 20 ROWS ONLY;
```

---

## 결론 요약

- **CBC**: (파일#, 블록#) → **해시 버킷** → **체인 탐색**. **핫 블록**은 `latch: cache buffers chains`로 드러남 — **접근 분산/쿼리 리라이트/인덱스 설계**가 핵심 처방.
- **LRU**: 순수 LRU가 아니라 **터치 카운트 기반 핫/콜드**. **KEEP/RECYCLE**로 워킹셋 의도적 보존/격리.
- **CR/Undo**: 일관 읽기 구현. 장시간 쿼리 + 대규모 DML은 **Undo 보존**을 체크.
- **실전 접근**: **AWR/ASH/v$**로 **사실 기반** 원인 규명 → **데이터/인덱스/파티션/버퍼풀** 설계와 **쿼리 패턴**을 함께 조정.
