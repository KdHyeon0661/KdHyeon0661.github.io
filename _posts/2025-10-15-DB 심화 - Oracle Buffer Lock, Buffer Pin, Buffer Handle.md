---
layout: post
title: DB 심화 - Oracle Buffer Lock / Buffer Pin / Buffer Handle
date: 2025-10-15 19:25:23 +0900
category: DB 심화
---
# Oracle Buffer Lock / Buffer Pin / Buffer Handle — 개념·내부 동작·실습·튜닝 총정리 (19c 기준)

> 목표  
> - “**버퍼 락(buffer lock)**”이라는 **비공식적** 표현이 실제로는 어떤 **내부 동기화 메커니즘**(pin, latch/mutex, enqueue)과 **블록 단위 동작**을 가리키는지 정확히 정리  
> - **Buffer Handle(= Buffer Header/X$BH 엔트리)**, **Pinning(고정)** 의 필요성과 작동 방식  
> - **대기 이벤트**(`buffer busy waits`, `read by other session`, `write complete waits`, `latch: cache buffers chains` 등)와의 연결  
> - **실습 시나리오**와 **튜닝 패턴**으로 “현장에서 보이는 현상 ↔ 내부 구조”를 대응

---

## 0) 용어 정리 — “락(lock)”, “래치(latch)/뮤텍스(mutex)”, “핀(pin)”, “엔큐(enqueue)”

Oracle에서 **동시성 제어**는 서로 다른 레벨의 원자성을 가진 여러 메커니즘이 겹쳐 작동합니다. “버퍼 락”이라는 표현은 주로 **버퍼 블록을 보호/고정**하는 내부 동작을 통칭하는 **관용적 표현**입니다. 정확히는 아래처럼 구분합니다.

- **엔큐(Enqueue, 잠금; TX/TM 등)**: 비교적 **장수명**의 대기 가능 잠금. 트랜잭션(row-level: TX), 오브젝트(테이블: TM) 등에 사용.
- **래치(Latch)**/**뮤텍스(Mutex)**: **극단적으로 짧은** 임계구역 보호(스핀락 계열). 예:  
  - `cache buffers chains` 래치: **버퍼 헤더 체인** 접근 보호 (해시 버킷)  
  - `cache buffer lru chain` 래치: **LRU 체인** 조작 보호  
  - 라이브러리 캐시 뮤텍스: 파싱/커서 보호 등
- **버퍼 핀(Buffer Pin)** / **핀 카운트(refcount)**:  
  - **특정 버퍼 프레임(= 한 DB 블록)** 을 **대체(evict)·쓰기** 못 하도록 그 순간 **고정**하는 내부 메커니즘.  
  - **다중 세션이 동시에 같은 블록을 읽는 동안** 각자 **공유 pin**을 들고 있을 수 있고, **갱신 시점**엔 **배타 pin**이 필요합니다(실제 내부 구현은 버전에 따라 다르지만, 개념적으로 “동시에 건드리면 안 되는 상태를 pin으로 지킨다”로 이해).

> 즉, “**버퍼 락**”이라는 표현은 보통 **버퍼에 대한 핀(pin)** 또는 이를 얻기 위한 **래치/뮤텍스 대기**와 **연계된 상태**를 가리키는 말로 쓰입니다.  
> 엄밀히는 **락(enqueue)** 과 **버퍼 pin**은 **다른 레이어**입니다.

---

## 1) Buffer Handle(=Buffer Header) — 무엇이고 어디에 있나?

**버퍼 캐시**는 “**버퍼 프레임**(8KB 등 블록 1개를 담는 메모리) + **버퍼 헤더**”로 구성됩니다. 이 **버퍼 헤더**(내부적으로 **Buffer Handle**로 불리기도 함)는 다음 메타를 가집니다.

- (파일#, 블록#) 식별자
- **상태**: xcur/scur/cr/free/read 등 (현재 또는 일관 버전)
- **Dirty 플래그**, 최근 **SCN/버전**, **ITL/락 정보**(블록 헤더 측면), **터치 카운트(tch)**  
- **핀 카운트(refcount)**: 누가/몇 명이 이 버퍼를 **현재 고정(pin)** 하고 있는지
- **체인 포인터**: **Cache Buffer Chains**(해시 버킷 체인)과 **LRU 체인**에 연결되는 링크

관측용 뷰(참고: X$BH는 내부 뷰로 버전별 차이가 있을 수 있음):

```sql
-- 핫(자주 참조) 버퍼 헤더 관측: tch 높은 블록
-- (SYS 권한 필요; 운영에선 과도한 X$ 조회는 지양)
SELECT file#, dbablk, tch
FROM   x$bh
WHERE  tch >= 64
ORDER  BY tch DESC FETCH FIRST 50 ROWS ONLY;

-- v$bh 로는 상태/클래스를 요약 확인
SELECT status, class, COUNT(*) AS cnt
FROM   v$bh
GROUP  BY status, class
ORDER  BY cnt DESC;
```

> 실무 의미: **tch** 가 비정상적으로 높은 블록은 **핫 블록** 후보 → **CBC 래치 경합**이나 **buffer busy waits** 관측과 함께 조사.

---

## 2) Pinning(핀) — 왜 필요한가? (필요성 · 의미 · 수명)

### 2.1 필요성

- **동일 블록**에 대해 **동시에 여러 세션이 접근**할 수 있습니다.  
- 누군가 블록을 **읽는 동안** 또는 **갱신 중**인데, 다른 스레드가 그 블록을 **LRU로 내보내거나** **동시에 바꿔버리면** **일관성이 깨짐**.  
- 따라서 접근하는 동안 **버퍼를 pin** 해서 “**나 이 블록 쓰고 있음, 내보내지 마/겹쳐 쓰지 마**”를 보장합니다.

### 2.2 수명과 형태(개념적)

- **공유 pin(읽기)**: 여러 세션이 **동일 블록을 동시에 읽는** 동안 **동시 pin 가능**.  
- **배타 pin(갱신)**: **현재 버전(current)** 블록에 **변경을 반영**하려면 다른 공유 pin과 충돌 조정이 필요(대기 발생 가능).  
- **CR 버전 핀**: Undo로 재조립된 **Consistent Read 버퍼**도 **일시적으로 pin** 됩니다(쿼리 결과 반환 동안).

> 내부 구현 디테일은 버전별로 조금씩 다를 수 있으나, **“읽는 동안 고정, 쓰는 동안 더 강하게 고정”**이라는 직관은 변하지 않습니다.

### 2.3 Pin과 다른 동기화의 관계

- **pin을 얻거나 해제**하려면 **버퍼 헤더 체인**에 들어갔다 나와야 하므로, 그 순간 **CBC 래치**를 잠깐 쥡니다.  
- **LRU 위치 갱신/evict** 시에는 **LRU 래치/뮤텍스**가 개입.  
- **행 잠금(row-level)** 은 **TX enqueue** 로 별도 관리. 즉, **버퍼 pin**과 **TX 락**은 **서로 다른 목적**을 위해 동시 사용됩니다.

---

## 3) “버퍼 락”과 대표 대기 이벤트

실전에서 “버퍼 락 걸렸다”라고 말할 때 보통 아래 **대기 이벤트**들이 등장합니다.

- **`buffer busy waits`**  
  - **같은 블록**을 **다른 세션이 사용 중**이라 당장 접근/변경할 수없는 상태.  
  - 원인: **다른 세션이 읽는 중/쓰는 중**, **블록 핀 충돌**, **ITL 부족** 등 여러 하위 케이스.  
- **`read by other session`**  
  - **다른 세션이 디스크에서 블록을 읽어오는 중**이며, 그 완료를 **공유**하려고 **대기**. (다중 세션이 동일 블록을 동시에 미스 → 한 세션이 읽고 나머지는 기다림)  
- **`write complete waits`**  
  - DBWn이 **해당 버퍼 쓰기 완료**해야 읽을 수 있는 상황에서, **쓰기 완료 대기**.  
- **`free buffer waits`**  
  - **LRU에서 free 버퍼를 확보**하지 못해 대기(버퍼 캐시 압박/DBWn 지연 증상).  
- **`latch: cache buffers chains`**  
  - **CBC 래치** 경합. **많은 세션이 같은 해시 버킷/핫 블록**을 동시에 노리면 발생.

관측 예시:

```sql
-- 시스템 상위 대기
SELECT * FROM (
  SELECT event, total_waits, time_waited_micro/1e6 AS sec_waited
  FROM   v$system_event
  ORDER  BY time_waited_micro DESC
) WHERE ROWNUM <= 20;

-- 현재 세션 대기
SELECT sid, serial#, username, event, wait_class, state, seconds_in_wait
FROM   v$session
WHERE  username IS NOT NULL
AND    state <> 'WAITED SHORT TIME'
ORDER  BY seconds_in_wait DESC;

-- 래치(특히 CBC) 경합
SELECT name, gets, misses, sleeps
FROM   v$latch
WHERE  LOWER(name) LIKE '%cache buffers chain%'
ORDER  BY misses DESC;
```

---

## 4) 블록 레벨에서 무슨 일이? — Current vs CR, 핀/변경/대체

### 4.1 Current(현재) 버전: 쓰기 전용 보호

- **DML(INSERT/UPDATE/DELETE)** 이 **Current 버전 버퍼**를 수정 → **Dirty** 플래그 세팅  
- 이때 **배타적 성격의 pin** 이 필요(다른 세션의 동시 변경 금지)  
- 커밋 시점에 **Redo flush(LGWR)** 로 **내구성** 확보, 이후 적절한 타이밍에 **DBWn** 이 **데이터파일로 기록**

### 4.2 CR(Consistent Read) 버전: 읽기 일관성

- SELECT가 **쿼리 시작 SCN** 기준으로 읽어야 할 때, 해당 블록이 더 신규라면 **Undo** 로 **CR 버전**을 **재조립**  
- **CR 버퍼**도 **pin** 되지만 **일시적**이며 **LRU 콜드 측**에서 쉽게 밀려날 수 있음  
- Undo 부족 시 **ORA-01555(snapshot too old)**

### 4.3 Pin과 Eviction(대체)

- **pin된 버퍼는 LRU에서 밀어낼 수 없음** → **free buffer waits** 방지 위해 DBWn이 부지런히 dirty flush, LRU는 콜드 말단 정리  
- **많은 세션이 동일 블록에 핀**을 걸면, 해당 버킷/체인에서 **CBC 래치 대기** 증가

---

## 5) 실습 시나리오 — “버퍼 pin/락 대기를 직접 만들어 보고 완화하기”

> 아래 실습은 **테스트 환경**에서만 실행하세요. (권한: 일반적으로 DBA 권한 필요)

### 5.1 준비: 샘플 테이블과 인덱스

```sql
DROP TABLE t_hot PURGE;

CREATE TABLE t_hot (
  id   NUMBER PRIMARY KEY,
  k    NUMBER NOT NULL,
  pad  VARCHAR2(200)
);

-- 단일 블록에 많은 행이 몰리도록 작은 행 크기 + 연속 id 삽입
BEGIN
  FOR i IN 1..50000 LOOP
    INSERT INTO t_hot (id, k, pad)
    VALUES (i, MOD(i,10), RPAD('x', 50, 'x'));
    IF MOD(i,1000)=0 THEN COMMIT; END IF;
  END LOOP;
  COMMIT;
END;
/

CREATE INDEX ix_t_hot_k ON t_hot(k);
```

### 5.2 “같은 블록/행에 대한 경합” 유발

두 개 이상의 세션에서 동시에 다음을 반복합니다.

```sql
-- 세션 A/B/C ... 동시에
UPDATE t_hot
   SET pad = RPAD('y',50,'y')
 WHERE k = 0
   AND id BETWEEN 1 AND 2000;  -- 비교적 좁은 범위, 같은 블록 빈번히 접근
COMMIT;
```

관측:

```sql
-- CBC 래치/버퍼 관련 대기 확인
SELECT event, total_waits, time_waited_micro/1e6 AS sec
FROM   v$system_event
WHERE  event IN ('buffer busy waits','read by other session','write complete waits')
   OR  event LIKE 'latch: cache buffers chains'
ORDER  BY sec DESC;

-- 핫 블록 후보(tch 상위) (SYS)
SELECT file#, dbablk, tch
FROM   x$bh
ORDER  BY tch DESC FETCH FIRST 20 ROWS ONLY;
```

**예상 현상**  
- `buffer busy waits` 증가(동일 블록 충돌)  
- 경우에 따라 `latch: cache buffers chains` 상승(같은 해시 버킷에 집중)  
- 물리 미스가 겹치면 일부 `read by other session` 관측

### 5.3 완화 1 — 액세스 분산(Reverse/Partition/샤딩)

```sql
-- 1) Reverse Key Index로 삽입/접근 분산(키 특성 따라 효과 상이)
DROP INDEX ix_t_hot_k;
CREATE INDEX ix_t_hot_k ON t_hot(k) REVERSE;

-- 2) 해시 파티셔닝으로 블록 집적도 분산
DROP TABLE t_hot PURGE;

CREATE TABLE t_hot (
  id  NUMBER,
  k   NUMBER,
  pad VARCHAR2(200),
  CONSTRAINT pk_t_hot PRIMARY KEY (id)
)
PARTITION BY HASH (k)
PARTITIONS 16;

BEGIN
  FOR i IN 1..50000 LOOP
    INSERT INTO t_hot (id, k, pad)
    VALUES (i, MOD(i,10), RPAD('x', 50, 'x'));
    IF MOD(i,1000)=0 THEN COMMIT; END IF;
  END LOOP;
  COMMIT;
END;
/

CREATE INDEX ix_t_hot_k ON t_hot(k) LOCAL;
```

효과 관측:

```sql
-- 경합 이벤트 감소 여부
SELECT event, total_waits, time_waited_micro/1e6 AS sec
FROM   v$system_event
WHERE  event IN ('buffer busy waits','read by other session')
ORDER  BY sec DESC;

-- tch 상위 블록 감소 확인
SELECT file#, dbablk, tch
FROM   x$bh
WHERE  tch >= 64
ORDER  BY tch DESC;
```

### 5.4 완화 2 — “단일 카운터” 패턴을 샤딩

```sql
-- 나쁜 예시: 단일 카운터(한 블록/한 행 과열)
DROP TABLE t_counter PURGE;
CREATE TABLE t_counter (id NUMBER PRIMARY KEY, cnt NUMBER);
INSERT INTO t_counter VALUES (1, 0);
COMMIT;

-- 여러 세션에서
-- UPDATE t_counter SET cnt = cnt + 1 WHERE id = 1; COMMIT;
-- => buffer busy / CBC latch 증가

-- 개선: N개 버킷으로 분산
DROP TABLE t_counter_shard PURGE;
CREATE TABLE t_counter_shard (k NUMBER PRIMARY KEY, cnt NUMBER);

BEGIN
  FOR i IN 0..63 LOOP
    INSERT INTO t_counter_shard VALUES (i, 0);
  END LOOP;
  COMMIT;
END;
/

-- 세션별로 서로 다른 k 사용
-- UPDATE t_counter_shard SET cnt = cnt + 1 WHERE k = :bucket; COMMIT;

-- 최종 합계
SELECT SUM(cnt) FROM t_counter_shard;
```

### 5.5 읽기 공존 최적화 — 커버링 인덱스/선택 컬럼

읽기 세션들이 **동일 블록**을 자주 스캔하는 경우(특히 좁은 인덱스 리프에 집중) `read by other session`/`buffer busy waits`가 보일 수 있습니다. **커버링 인덱스**(필요 컬럼을 인덱스에 포함)로 **테이블 블록 접근 자체를 줄이기**도 유효합니다.

```sql
-- 커버링 인덱스 예: 조회에 필요한 컬럼 포함
CREATE INDEX ix_t_hot_k_cover ON t_hot(k, pad);

-- 조회시 테이블 접근 최소화
SELECT /*+ INDEX(t_hot ix_t_hot_k_cover) */
       pad
FROM   t_hot
WHERE  k = 7
AND    id BETWEEN 10000 AND 12000;
```

---

## 6) “버퍼 핀”과 “행 락(TX)”의 차이 — 헷갈리기 쉬운 포인트

- **버퍼 핀**: **블록 프레임 자체**를 고정(대체·동시 변경 방지). **내부 메커니즘**, **래치/뮤텍스**와 결합.  
- **TX(Row-level) 락**: **특정 행**의 논리적 동시 업데이트 방지. **엔큐** 기반, 커밋/롤백까지 지속.

둘은 **독립적**입니다. 예를 들어:

1. 세션 A가 블록 B의 행 r1을 업데이트 → **TX 락**(해당 행) + **버퍼 핀**(블록)  
2. 세션 B가 **동일 블록 B의 다른 행 r2** 업데이트 시도 →  
   - **TX 락** 경쟁이 없더라도, **버퍼 핀/버퍼 상태** 때문에 `buffer busy waits` 같은 **버퍼 레벨 대기**가 발생할 수 있음(특히 ITL 부족/블록 형상 문제 등)

관측 팁:

```sql
-- 세그먼트 통계에서 버퍼 관련 대기 상위 오브젝트
SELECT o.owner, o.object_name, o.object_type,
       SUM(CASE WHEN s.statistic_name='buffer busy waits' THEN s.value ELSE 0 END) AS buf_busy,
       SUM(CASE WHEN s.statistic_name='read by other session' THEN s.value ELSE 0 END) AS read_other
FROM   v$segment_statistics s
JOIN   dba_objects o ON o.owner=s.owner AND o.object_name=s.object_name
GROUP  BY o.owner, o.object_name, o.object_type
ORDER  BY buf_busy DESC FETCH FIRST 20 ROWS ONLY;
```

---

## 7) ITL 부족과 버퍼 대기 — “버퍼 락처럼 보이는” 전형적 사례

**ITL(Interested Transaction List)** 은 **블록 헤더** 안에 있는 **트랜잭션 슬롯**입니다. 동시에 여러 트랜잭션이 하나의 블록을 갱신하려면 **충분한 ITL** 이 필요합니다.

- **ITL 부족** → `enq: TX - allocate ITL entry` 대기  
- **동시에** 해당 블록이 **고정(pin)** 되어 **버퍼 busy** 와 엮여 보이기도 함

완화:

```sql
-- 테이블/인덱스 ITL 증설 (주의: 공간 trade-off)
ALTER TABLE  t_hot  STORAGE (INITRANS 8 MAXTRANS 255);
ALTER INDEX  ix_t_hot_k  STORAGE (INITRANS 8 MAXTRANS 255);

-- 재구성(재빌드/재생성) 또는 PCTFREE 조정으로 헤더 여유 확보
ALTER TABLE t_hot MOVE;  -- (로우 체인지, 인덱스 재빌드 필요)
ALTER INDEX ix_t_hot_k REBUILD;
```

---

## 8) LRU·DBWn·Pin 상호작용 — `free buffer waits` / `write complete waits`

- **free buffer waits**: **LRU에서 비어있는 버퍼를 얻지 못해** 대기.  
  - 원인: **Dirty가 많은데 DBWn 쓰기 지연**, **버퍼 캐시가 너무 작아 evict 불가**(핀 다수) 등.  
  - 처방: **db_cache_size 증설**, **DBWn/스토리지 쓰기 성능** 개선, **Batch DML 패턴 조정**.
- **write complete waits**: **읽으려는 버퍼가 쓰기 중** → **DBWn 완료 대기**.

점검:

```sql
SELECT event, total_waits, time_waited_micro/1e6 AS sec
FROM   v$system_event
WHERE  event IN ('free buffer waits','write complete waits')
ORDER  BY sec DESC;

-- DBWn 활동/체크포인트 관련(요약)
SELECT * FROM v$instance_recovery;
```

---

## 9) RAC에서의 “버퍼 락” 체감 — 글로벌 캐시(GCS)와 핫 블록

- RAC는 **블록 소유권/버전**을 인스턴스 간 교환.  
- 동일 블록에 접근이 집중되면 **gc 관련 대기**(`gc buffer busy`, `gc current request`, `gc cr request`)가 다발.  
- 완화는 **서비스 기반 워크로드 분할**, **파티셔닝으로 데이터 로컬리티 확보**, **Reverse/Hash 인덱스**, **시퀀스 캐시/증가폭 조정** 등 단일 인스턴스와 유사+α.

---

## 10) 관측·진단 SQL 모음(현업 단축키)

```sql
-- 10.1 상위 이벤트/세션
SELECT * FROM (
  SELECT event, total_waits, time_waited_micro/1e6 sec
  FROM   v$system_event
  ORDER  BY sec DESC
) WHERE ROWNUM <= 20;

SELECT sid, serial#, username, event, wait_class, state, seconds_in_wait, sql_id
FROM   v$session
WHERE  username IS NOT NULL
AND    state <> 'WAITED SHORT TIME'
ORDER  BY seconds_in_wait DESC;

-- 10.2 래치(CBC) 경합
SELECT name, gets, misses, sleeps,
       ROUND(misses/NULLIF(gets,0),6) AS miss_ratio
FROM   v$latch
WHERE  LOWER(name) LIKE '%cache buffers chain%'
ORDER  BY misses DESC;

-- 10.3 세그먼트 단위 버퍼 관련 대기
SELECT o.owner, o.object_name, o.object_type,
       SUM(CASE WHEN s.statistic_name='buffer busy waits' THEN s.value ELSE 0 END) AS buf_busy,
       SUM(CASE WHEN s.statistic_name='read by other session' THEN s.value ELSE 0 END) AS read_other
FROM   v$segment_statistics s
JOIN   dba_objects o ON o.owner=s.owner AND o.object_name=s.object_name
GROUP  BY o.owner, o.object_name, o.object_type
ORDER  BY buf_busy DESC FETCH FIRST 15 ROWS ONLY;

-- 10.4 버퍼 헤더 상태 요약
SELECT status, class, COUNT(*) cnt
FROM   v$bh
GROUP  BY status, class
ORDER  BY cnt DESC;

-- 10.5 핫 블록(x$bh.tch 상위)  ※SYS 권한
SELECT file#, dbablk, tch
FROM   x$bh
WHERE  tch >= 64
ORDER  BY tch DESC FETCH FIRST 30 ROWS ONLY;
```

---

## 11) 튜닝 체크리스트 — “버퍼 락/핀 병목”을 볼 때 무엇을 바꿀 것인가?

1. **핫 블록 패턴** 탐지: `x$bh.tch` 상위, `v$segment_statistics`(`buffer busy waits`) 상위  
2. **CBC 래치** 경합 유무: `latch: cache buffers chains` 미스/슬립 비정상 증가?  
3. **접근 분산 설계**: Reverse Key / Hash 파티셔닝 / 단일 카운터 → 샤딩 카운터 / 시퀀스 캐시+증가폭  
4. **커버링 인덱스**: 동일 블록 테이블 액세스 줄여 `read by other session`/`buffer busy` 완화  
5. **INITRANS/ITL**: 업데이트 동시성 높으면 ITL 부족 체크(`enq: TX - allocate ITL entry`)  
6. **db_cache_size / db_keep_cache_size / db_recycle_cache_size**: LRU 압박/오염 완화  
7. **DBWn/스토리지 쓰기 성능**: `free buffer waits`/`write complete waits` 감시  
8. **배치 DML 패턴 조정**: 대량 업데이트를 **파티션·키 범위별**로 분산, 커밋 간격 튜닝  
9. **RAC**: 서비스/파티션으로 **인스턴스 로컬리티** 유지, gc 대기 완화  
10. **AWR/ASH**: 상위 SQL·대기·시간대별 추세로 **정량** 판별(감 아닌 데이터)

---

## 12) 미니 실전 예제 — “버퍼 핀 줄이기로 보고서 가속”

### 상황
- 보고서 쿼리가 **같은 소수 블록**의 차원 테이블을 **수천 세션이 동시 조회** → `read by other session`·`buffer busy waits` 증가  
- 일부 세션은 같은 블록의 다른 행을 **UPDATE**(희소하지만 존재) → 간헐적 지연

### 처방
1. **차원 테이블을 KEEP 풀**에 올려 **재사용 핫셋**을 메모리에 고정  
2. 보고서 쿼리에 **커버링 인덱스** 적용 → 테이블 블록 접근 최소화  
3. 희소 UPDATE는 **야간 배치**로 이관(가능 시)

```sql
ALTER SYSTEM SET db_keep_cache_size = 512M SCOPE=BOTH;

ALTER TABLE dim_product STORAGE (BUFFER_POOL KEEP);
CREATE INDEX ix_dim_product_cover ON dim_product(product_code, product_name, category);

-- 보고서 쿼리
SELECT /*+ INDEX(dp ix_dim_product_cover) */
       dp.product_code, dp.product_name, dp.category, SUM(f.sales)
FROM   fact_sales f
JOIN   dim_product dp
  ON   dp.product_code = f.product_code
WHERE  f.sale_dt BETWEEN ADD_MONTHS(TRUNC(SYSDATE),-1) AND SYSDATE
GROUP  BY dp.product_code, dp.product_name, dp.category;
```

**효과**  
- 차원 블록을 **항상 메모리**에서 즉시 제공 → **`read by other session` 감소**  
- 인덱스만으로 필요한 컬럼 획득 → **테이블 블록 접근·핀 감소**

---

## 13) 수학적 감각(보조 지표) — “핀/경합의 간접 신호”

> 지표는 어디까지나 **참고용**입니다. 결론은 **AWR/ASH의 대기/SQL 근거**로 내리세요.

- **논리/물리 읽기 대비 경합 비율**  
  $$ \text{Contention Ratio} \approx \frac{\text{buffer busy waits (time or count)}}{\text{consistent gets} + \text{db block gets}} $$
- **CBC 래치 미스율**  
  $$ \text{CBC Miss Ratio} \approx \frac{\text{misses}}{\text{gets}} $$
- **Free Buffer Pressure**  
  $$ \text{FreeBuf Pressure} \approx \frac{\text{free buffer waits (time)}}{\text{DB time}} $$

**주의**: 히트율이 높아도 **CBC/버퍼 busy**가 심하면 **응답시간은 여전히 나쁠 수 있음**.

---

## 14) 요약

- “**버퍼 락**”은 정확히는 **버퍼 pin(고정)** 과 이를 둘러싼 **래치/뮤텍스/엔큐** 상호작용을 **관용적으로 묶어 부르는 말**입니다.  
- **Buffer Handle(=Buffer Header/X$BH)** 는 각 **버퍼 프레임**의 메타(상태/핀카운트/링크/tch 등)를 보유하며, **Cache Buffer Chains**(해시 버킷)과 **LRU 체인**에 연결됩니다.  
- **Pinning** 은 **읽기/쓰기 시 블록 대체·경합을 방지**하기 위한 핵심 메커니즘.  
- 현업 병목은 `buffer busy waits`, `read by other session`, `write complete waits`, `latch: cache buffers chains` 등으로 표출됩니다.  
- **접근 분산(Reverse/Hash/파티셔닝/샤딩)**, **커버링 인덱스/선택 컬럼**, **ITL/INITRANS 조정**, **KEEP/RECYCLE/캐시 크기/DBWn 성능** 등으로 완화합니다.  
- 결론은 항상 **AWR/ASH/v$**로 **정량 진단 → 설계/쿼리/파라미터**를 함께 조정하는 방향이어야 합니다.
