---
layout: post
title: DB 심화 - 문장 수준 읽기 일관성과 Consistent 모드 블록 읽기
date: 2025-10-15 22:25:23 +0900
category: DB 심화
---
# Oracle의 **문장 수준 읽기 일관성(Statement-Level Read Consistency)** 과 **Consistent 모드 블록 읽기(CR Read)** — 내부 동작·예제·튜닝 총정리 (19c 기준)

> 목표
> - 오라클이 **기본 격리 수준**에서 제공하는 **문장 수준(read consistency at statement level)** 을 정확히 정의하고,
> - 이를 구현하는 핵심 메커니즘인 **Consistent 모드 블록 읽기(= CR Read, consistent get)** 의 **세부 원리**를 단계별로 설명하며,
> - **Undo / ITL / Row Lock Byte / 커밋 SCN / 지연 클린아웃** 과의 연결을 실제 **세션 A/B 시나리오**와 **진단 SQL**로 확인.
> - **ORA-01555(snapshot too old)**, **consistent gets 과다**, **read by other session** 등 자주 만나는 증상과 튜닝 포인트까지 정리.

---

## 한눈에 보는 결론

- 오라클은 **기본 SELECT** 에 대해 **“문장 시작 시점의 데이터 일관성”** 을 **항상 보장**한다.
- 문장 시작 시 오라클은 내부적으로 **쿼리 SCN(= S)** 을 잡는다. 그 문장이 끝날 때까지,
  $$\text{보여줄 데이터의 커밋 SCN} \le S$$
  를 만족하도록 **Consistent Read(CR)** 를 수행한다.
- 어떤 블록(혹은 행)의 변경이 **쿼리 SCN 이후에 발생**했다면, 해당 블록의 **Undo 체인**을 따라가 **과거 버전**을 재구성한 **CR 버퍼(복제본)** 를 만들어서 반환한다.
- 이를 위해 **데이터 블록 헤더의 ITL**(Interested Transaction List), **행 헤더의 Row Lock Byte**, **Undo 세그먼트(트랜잭션 테이블 슬롯)**, **커밋 SCN**, **지연 클린아웃**이 함께 동작한다.

---

## 문장 수준 읽기 일관성(Statement-Level Read Consistency)란?

### 정의

- **문장 시작 시점**(파서/실행기 진입 직후) **고정된 SCN** 을 기준으로,
- **SELECT** 는 **해당 문장이 끝날 때까지** **그 시점에 커밋되어 있던 상태**만 보이도록 결과를 반환한다.
- 문장 실행 중에 **다른 세션이 커밋**하더라도, 그 **새 커밋 결과는 보이지 않는다.**

> 이는 PostgreSQL의 “트랜잭션 수준 MVCC”와 달리, **오라클의 기본은 문장 수준**이다. 트랜잭션 전체에 대해 동일한 스냅샷을 강제하려면 **SERIALIZABLE** 로 올려야 한다.

### 직관적 이해

- SELECT가 시작될 때 **스냅샷(=쿼리 SCN)** 을 찍고, 그 이후의 변경은 **Undo** 를 통해 **과거 버전**으로 되돌려 읽는다.
- 이렇게 하면 **긴 스캔 중간에 들어온 커밋** 때문에 **부분적으로 새로운 값/예전 값이 섞이는 문제**가 없다.

---

## Consistent 모드 블록 읽기(Consistent Get, CR Read)의 큰 그림

### Current vs Consistent

- **Current 모드**(db block gets): **현재 버전**을 그대로 본다. 주로
  - DML 수행,
  - `SELECT ... FOR UPDATE` 로 **행 잠금**을 취득할 때,
  - 일부 인덱스/세그먼트 헤더 관리 시 사용.
- **Consistent 모드**(consistent gets): **쿼리 SCN 기준의 과거 버전**을 보기 위해
  - **Undo 레코드**를 적용한 **CR 블록**을 만들어 읽는다.

통계 확인:

```sql
SELECT name, value
FROM   v$sysstat
WHERE  name IN ('consistent gets','db block gets','physical reads')
ORDER  BY name;
```

### 왜 CR 블록을 만드나?

- 문장 수준 일관성을 지키려면 **쿼리 SCN보다 새로운 변경**은 **보여주면 안 된다**.
- 해당 블록의 **Row Lock Byte → ITL → 트랜잭션 슬롯(XID) → Undo 체인**을 따라 **과거 상태**로 재조립한 **CR 버퍼**를 만들어서 사용한다.

---

## 블록 내부 구조와 일관성 — ITL · Row Lock Byte · 트랜잭션 슬롯

- **블록 헤더의 ITL 슬롯**:
  “이 블록을 현재/최근 변경한 **트랜잭션(XID)**” 의 자리표.
  - XID(usn.slot.sqn), **Undo 포인터**, **플래그**, (클린아웃 후) **커밋 SCN** 저장.
- **Row Lock Byte(행 헤더의 잠금 바이트)**:
  “이 행을 변경한 ITL 슬롯 번호(인덱스)” 를 기록.
  - 행 → ITL → XID → Undo 슬롯 으로 연결.
- **Undo 세그먼트의 트랜잭션 테이블 슬롯**:
  XID가 가리키는 집. **Undo 체인**의 헤드, **커밋 상태/SCN** 등을 보관.

> **지연 클린아웃(Deferred Cleanout)**: 커밋 직후 ITL에 **커밋 SCN**을 즉시 못 쓰면, **나중 읽기** 시 그 정보를 채운다. 이후 같은 블록 재읽기는 Undo 조회 없이 “커밋됨”을 빠르게 판단.

---

## CR Read의 세부 알고리즘(개념적 단계)

SELECT가 블록을 읽을 때, 내부적으로 아래와 같은 절차를 밟는다(요약):

1. **쿼리 SCN = S** 를 확보(문장 시작 시).
2. 블록을 버퍼 캐시에서 찾는다(없으면 물리 I/O).
3. 블록의 **변경 힌트**(행 헤더의 **Row Lock Byte**) 를 본다.
   - 이 행을 변경한 **ITL 슬롯 k** 를 찾는다.
4. **ITL[k]** 를 조사:
   - **커밋 SCN** 이 **기록**되어 있고 **커밋 SCN ≤ S** → 해당 변경은 **문장 시점 이전에 커밋**됨 → **현재 값** 사용.
   - **커밋 SCN > S** (혹은 커밋 안 됨/미기록) → **Undo 체인**을 따라가서 **과거 버전**을 재조립 → **CR 블록** 생성 후 사용.
   - **커밋 SCN 미기록(클린아웃 전)** 이면 트랜잭션 슬롯을 확인해 **커밋/SCN**을 가져와 ITL에 채워 넣는 **지연 클린아웃** 수행. 이후 다시 4단계 판정.
5. 필요한 모든 행에 대해 3~4를 반복.
6. 결과 반환.

수식 감각(의사조건):

$$
\text{보여줄 행 버전} =
\begin{cases}
\text{현재값} & \text{if } \text{commit\_scn} \le S \\
\text{Undo 적용값(CR)} & \text{if } \text{commit\_scn} > S \ \text{or unknown}
\end{cases}
$$

---

## 세션 시나리오로 보는 문장 수준 일관성

### 데이터 준비

```sql
DROP TABLE t_cr PURGE;

CREATE TABLE t_cr (
  id    NUMBER PRIMARY KEY,
  grp   NUMBER NOT NULL,
  val   NUMBER NOT NULL,
  note  VARCHAR2(50)
);

BEGIN
  FOR i IN 1..10000 LOOP
    INSERT INTO t_cr VALUES (i, MOD(i,10), 0, RPAD('x',20,'x'));
    IF MOD(i,1000)=0 THEN COMMIT; END IF;
  END LOOP;
  COMMIT;
END;
/
CREATE INDEX ix_t_cr_grp ON t_cr(grp);
```

### 세션 A — 긴 스캔(의도적으로 느리게)

```sql
-- 세션 A
SET TIMING ON
SELECT /*+ FULL(t_cr) */ SUM(val)
FROM   t_cr
WHERE  grp IN (0,1,2,3,4);  -- 전체의 절반쯤 스캔
-- 의도적으로 fetch 사이 사이에 DBMS_LOCK.SLEEP(0.5) 등으로 느리게 만들 수 있음(PL/SQL에서)
```

### 세션 B — 중간에 UPDATE + COMMIT

```sql
-- 세션 B
UPDATE t_cr SET val = 100 WHERE grp = 0;
COMMIT;
```

### 관찰 포인트

- 세션 A의 SELECT 는 **시작 시점의 SCN** 으로 **전 구간을 읽는다**.
- 세션 B가 중간에 커밋해도, 세션 A는 **그 커밋 이후 변경을 보지 않는다**.
- A가 **이미 읽은 블록**은 그대로이고, **아직 안 읽은 블록**을 읽을 때 **해당 변경이 S 이후**라면 **Undo** 를 따라 **CR 블록**을 만들어서 **변경 전 값(=0)** 을 본다.

> 결과적으로 세션 A의 SUM(val)은 **전부 0**으로 계산된다(문장 시작 시점 기준).
> 세션 B의 커밋은 **세션 A의 다음 문장부터** 보이게 된다.

---

## 인덱스/테이블 조인 시 CR — “인덱스는 현재 버전, 테이블은 CR로”

- 오라클은 **인덱스 구조 탐색**에서 대부분 **현재 버전**을 사용하되,
- **테이블 행 데이터** 를 가져올 때 그 행의 **커밋 시점** 을 확인해 **필요 시 CR** 로 되돌린다.
- 즉 **조인/스캔** 도중 새로운 커밋이 들어와도, **행 단위** 로 **쿼리 SCN** 기준의 **과거 버전**을 재구성해 가져온다.

실험(아이디어):

```sql
-- 인덱스 스캔 + 테이블 페치
SELECT /*+ index(t ix_t_cr_grp) */ COUNT(*)
FROM   t_cr t
WHERE  grp = 7;
```

- 인덱스 엔트리는 현재 기준으로 탐색하더라도, 테이블 페치 시 해당 행이 **S 이후 커밋**으로 갱신되었으면 **Undo** 로 **CR 값**을 읽는다.

---

## SELECT FOR UPDATE와 Current Read

- `SELECT ... FOR UPDATE` 는 **행 잠금**을 위해 **현재 버전**(current)을 본다(= **db block gets**).
- 이는 문장 수준 **읽기 일관성** 과는 다른 목적(락 취득)이므로 주의.
- 이후 UPDATE/DELETE로 이어질 수 있으니 **CR** 이 아닌 **Current** 가 필요한 것이다.

간단 확인:

```sql
SELECT COUNT(*) FROM t_cr WHERE grp = 0;              -- 일반 SELECT(일관성)
SELECT COUNT(*) FROM t_cr WHERE grp = 0 FOR UPDATE;   -- 행 잠금 + current read
```

통계는 `v$sysstat` 의 `db block gets` 증가로 관찰 가능.

---

## ORA-01555(snapshot too old) — 왜 터지나?

### 원인

- SELECT 가 **아주 오래** 걸리고, 그동안 **다른 세션의 DML/COMMIT** 로 인해 **필요한 Undo 레코드**가 **재사용**(순환)되어 사라졌다.
- 그러면 해당 행을 **쿼리 SCN 시점** 으로 **되돌릴** 수 없으므로 ORA-01555.

### 대응

- **UNDO 테이블스페이스 용량** / **`UNDO_RETENTION`** 확보
- **장시간 보고서** ↔ **대규모 DML/배치** 시간대 분리
- 필요 시 **쿼리 재작성**(조기 필터링/인덱스 활용), **페치 전략 조정**(더 빠르게 끝내기)

진단:

```sql
SHOW PARAMETER undo_tablespace;
SHOW PARAMETER undo_retention;

SELECT begin_time, undoblks, txncount, ssolderrcnt, tuned_undoretention
FROM   v$undostat
ORDER  BY begin_time DESC FETCH FIRST 24 ROWS ONLY;
```

- `ssolderrcnt` : snapshot too old 발생 횟수
- `tuned_undoretention` : 오라클이 동적으로 계산한 보존 목표

---

## 지연 클린아웃(Deferred Cleanout)과 CR 비용

- 트랜잭션 **커밋 직후**, 모든 관련 블록의 ITL에 **커밋 SCN** 을 **즉시** 기록하지 못할 수 있다(부담/타이밍 문제).
- **다음에 해당 블록을 읽는 세션** 이 **트랜잭션 테이블** 을 확인해 **커밋 SCN** 을 알아내고, **ITL 엔트리** 에 그것을 적어 넣는다(**클린아웃**).
- 이후 같은 블록의 읽기는 Undo를 안 타도 되므로 **CR 비용 감소**.

관찰(개념):

```sql
-- 클린아웃이 자주 보인다면, 해당 세그먼트/블록에 커밋 직후 접근 패턴이 겹치는지,
-- 또는 ITL/헤더 여유 등 다른 경합 요인이 없는지 함께 본다.
SELECT event, total_waits, time_waited_micro/1e6 AS sec
FROM   v$system_event
WHERE  event IN ('buffer busy waits','read by other session','enq: TX - allocate ITL entry')
ORDER  BY sec DESC;
```

---

## “Consistent gets” 가 높다는 건 나쁜가?

- **아니다.** 일관성을 지키려면 필요한 비용이다.
- 다만, **불필요한 스캔/조인/중복 읽기** 로 **consistent gets** 가 과도하게 많다면 **쿼리 튜닝 신호**일 수 있다.
- 튜닝 포인트:
  - **인덱스 설계/커버링 인덱스**
  - **조기 필터링**(Predicate Pushdown)
  - **조인 순서/카디널리티** 교정
  - **필요 컬럼만 SELECT**(SELECT * 지양)

---

## RAC에서의 CR — GC 대기와의 연동(한 줄 요약)

- RAC에선 블록이 **인스턴스 간** 이동/요청되며, **CR 이미지를 원격에서 조립**/전송하는 경로도 있다.
- `gc cr request`, `gc buffer busy` 등 **GC 대기**가 표출될 수 있으며, **서비스 기반 파티셔닝/로컬리티** 설계가 중요.

---

## 실습: CR 동작을 체감하는 미니 데모

### 준비

```sql
DROP TABLE t_demo_cr PURGE;

CREATE TABLE t_demo_cr (
  id   NUMBER PRIMARY KEY,
  g    NUMBER NOT NULL,
  v    NUMBER NOT NULL
);

BEGIN
  FOR i IN 1..2000 LOOP
    INSERT INTO t_demo_cr VALUES (i, MOD(i,5), 0);
  END LOOP;
  COMMIT;
END;
/
CREATE INDEX ix_demo_cr_g ON t_demo_cr(g);
```

### 세션 A — 느린 SELECT

```sql
-- 세션 A
SET AUTOTRACE OFF
SET TIMING ON

-- (선택) 느리게 만들기 위한 의도적 대기: PL/SQL로 CHUNK별 SLEEP 가능
SELECT /*+ FULL(t) */ COUNT(*)
FROM   t_demo_cr t
WHERE  g IN (0,1,2);   -- 넓은 범위
```

### 세션 B — 중간에 값 변경 후 커밋

```sql
-- 세션 B
UPDATE t_demo_cr SET v = 999 WHERE g = 0;
COMMIT;
```

### 기대 관찰

- 세션 A의 **COUNT(*)** 결과는 **변경 이전 기준**(문장 시작 시점)과 일치.
- 세션 A가 **아직 읽지 못한 블록**을 읽는 시점에 g=0 행들이 이미 커밋되어 있더라도, A는 **Undo 적용(CR)** 으로 **변경 전 버전**을 본다.

---

## 진단/관측 SQL 단축키

```sql
-- 시스템 전반 일관성/현재 읽기/물리I/O 감
SELECT name, value
FROM   v$sysstat
WHERE  name IN ('consistent gets','db block gets','physical reads','redo size','user commits')
ORDER  BY name;

-- 현재 대기(읽기/버퍼 관련)
SELECT sid, event, wait_class, state, seconds_in_wait
FROM   v$session
WHERE  state <> 'WAITED SHORT TIME'
AND    (event LIKE 'read by other session'
    OR  event LIKE 'buffer busy waits'
    OR  event LIKE 'enq: TX%')
ORDER  BY seconds_in_wait DESC;

-- Undo 보존/에러 추이
SELECT begin_time, undoblks, txncount, ssolderrcnt, tuned_undoretention
FROM   v$undostat
ORDER  BY begin_time DESC FETCH FIRST 24 ROWS ONLY;
```

---

## 성능/안정성 체크리스트 (CR 관점)

1. **ORA-01555** 가 보이면
   - `UNDO_RETENTION` / Undo TS 용량 / DML·보고서 시간 분리 / 쿼리 튜닝
2. **consistent gets 과다**
   - 스캔 폭/조인 순서/인덱스 설계/커버링 인덱스/SELECT 컬럼 축소
3. **read by other session**
   - 같은 블록을 다수 세션이 동시에 미스 → 한 세션이 읽는 동안 나머지는 대기
   - 인덱스/파티셔닝/캐시 정책(KEEP/RECYCLE)으로 히트율 개선
4. **buffer busy waits**
   - 핫 블록/ITL 부족/쓰기경합 → 키 분산, INITRANS/PCTFREE, 샤딩 카운터 등
5. **지연 클린아웃 빈발**
   - 해당 세그먼트에서 커밋 직후 잦은 읽기/쓰기 패턴 → ITL/헤더 여유, 경합 완화, 파티셔닝 검토
6. **SELECT FOR UPDATE** 사용 구간
   - Current read 증가로 경합/대기 양상 변동 → 필요한 곳에서만 사용

---

## 보너스: 수식으로 보는 CR 판정 규칙(개념)

- 쿼리 시작 시 **SCN = S**
- 각 행의 “유효 커밋 SCN” 을 **C** 라고 하면, 보여줄 값의 조건은

$$
\text{Visible}(row) \iff
\begin{cases}
C \le S & \text{(이미 커밋된 버전)} \\
\text{CR(row, S)} & \text{(C > S이거나 unknown → Undo 통한 과거 이미지)}
\end{cases}
$$

- **CR(row, S)**: Undo 체인에서 **SCN \(\le S\)** 를 만족하는 **직전 버전**을 찾는 함수(개념).
- Undo 체인이 부족하면 **CR(row, S)** 가 실패 → **ORA-01555**.

---

## 마무리 요약

- 오라클은 **기본적으로 모든 SELECT** 에 대해 **문장 수준 일관성** 을 제공한다.
- 구현의 핵심은 **Consistent 모드 블록 읽기(CR Read)** 이며, **Undo/ITL/Row Lock Byte/커밋 SCN/지연 클린아웃** 이 맞물려 동작한다.
- **인덱스는 현재 버전으로 탐색**하더라도, **테이블 행은 필요 시 Undo로 과거 버전(CR)을 재조립** 하여 **쿼리 SCN** 기준으로 보여준다.
- 문제의 주범은 보통 **Undo 부족(ORA-01555)**, **핫 블록/ITL 부족**, **비효율 쿼리로 인한 consistent gets 과다** 이며,
  적절한 **Undo 보존/용량**, **키 분산/ITL 조정**, **쿼리/인덱스 설계** 로 안정성과 성능을 함께 달성할 수 있다.

> 한 줄 정리:
> **오라클의 SELECT 는 “시작 시 찍은 스냅샷”을 끝까지 지킨다. 바뀐 건 Undo로 되돌려 읽는 것이고, 그 기술이 바로 Consistent Read다.**
