---
layout: post
title: DB 심화 - Consistent vs Current 모드 읽기
date: 2025-10-15 23:25:23 +0900
category: DB 심화
---
# Consistent vs Current 모드 읽기 — 동작 차이, 갱신 시 상호작용, “일관성 없이(update anomaly)” 보이는 사례까지 (Oracle 19c 기준)

> 목표
> - **Consistent 모드(= Consistent Read, consistent gets)** 와 **Current 모드(= Current Read, db block gets)** 의 **개념/차이/내부 절차**를 명확히 구분
> - “**Consistent로 갱신 대상을 식별하고, Current로 실제 갱신**”하는 **일반적인 DML 경로**를 타임라인으로 해부
> - **Consistent로 갱신할 때 생기는 현상?** / **Current로 갱신할 때 생기는 현상?**을 예제와 함께 **정확히** 설명
> - **Read Committed(기본)**, **Serializable**에서 각각 발생 가능한 **갱신 이상(anomaly)** 을 실습 시나리오로 보여주고, **예방책(SELECT FOR UPDATE, 버전 컬럼, 체크 제약/락 설계)** 제시

---

## 한눈 요약

- **Consistent 모드 읽기**: **문장 시작 시점의 SCN**으로 **과거 버전**을 재구성하여 읽는 것.
  - 내부적으로 **Undo** 를 적용해 **CR(Consistent Read) 버퍼**를 만들어서 반환.
  - 통계: `consistent gets` 증가.
- **Current 모드 읽기**: **현재 버전**을 그대로 읽는 것(락/수정 전제).
  - DML(UPDATE/DELETE/INSERT) 시 **수정 대상 블록/행**에 대해 **Current** 로 잡고 **행 락(TX enqueue)** + **버퍼 pin** 후 변경.
  - 통계: `db block gets` 증가.

오라클의 일반 DML 경로는 **“Consistent로 후보행을 찾고 → Current로 바꾼다”** 이다.

---

## Consistent vs Current — 개념과 내부 동작

### Consistent Read (CR, consistent gets)

- **문장(SELECT 혹은 DML 내부의 읽기 단계) 시작 시점의 SCN**(스냅샷)을 기준으로,
- 해당 시점 이후 커밋된 변경은 **Undo** 를 통해 **과거 버전**으로 되돌려(재조립) 읽는다.
- **블록 헤더의 ITL / Row Lock Byte / 트랜잭션 테이블 슬롯** 을 따라 **커밋 SCN 판정 → Undo 체인 추적**으로 구현.
- 목적: **Statement-level Read Consistency** 보장.
- **갱신 자체는 하지 않음**(읽기 단계).
- 지표: `consistent gets`(논리 일관 읽기).

### Current Read (current, db block gets)

- **현재 버전** 블록을 **그대로** 읽는다(= 일관성 복원 없이).
- 주로 **DML 수행**(UPDATE/DELETE/INSERT), **SELECT FOR UPDATE**(행 잠금 취득) 시 사용.
- **행 잠금(TX enqueue)**, **ITL 슬롯 점유**, **버퍼 pin**, **Dirty** 마킹 등 **쓰기를 위한 준비/보호**가 따라온다.
- 지표: `db block gets`(논리 현재 읽기).

---

## “Consistent로 갱신 대상을 식별하고, Current로 갱신” — 표준 DML 경로

UPDATE/DELETE 는 두 단계로 이해하면 쉽다.

1) **읽기 단계(Consistent)**: WHERE 조건으로 **대상 행을 찾는다**
2) **쓰기 단계(Current)**: **대상 행을 현재 버전으로 재확인**하고 **락 후 변경**한다

### 단계별 내부 흐름(요약)

1. **Consistent 읽기(찾기)**:
   - 쿼리 SCN = S 를 잡고 인덱스/테이블을 스캔하여 후보 **ROWID** 목록을 만든다(필요 시 Undo 적용).
2. **Current 재확인 & 갱신**:
   - 각 ROWID에 대해 **현재 버전 블록**을 읽는다(Current),
   - **행 잠금**(TX) 및 **ITL 슬롯** 확보,
   - (필요 시) **칸 충돌/변경감지**(Serializable 등) 후 **실제 칼럼 업데이트** → **Dirty**.
3. **커밋**: LGWR가 **Redo** 를 온라인 리두 로그에 동기 flush → **Durability 보장**. DBWn은 나중에 데이터파일에 기록.

이때 “읽기 단계”와 “쓰기 단계” 사이에 **다른 세션**이 동일 행을 **먼저 변경/커밋**했으면…
- 기본 **Read Committed**에서는 **최신(Current) 행**을 기준으로 계속 진행한다(필요 시 **재평가**로 인해 **선택 행이 줄거나 늘 수 있음**).
- **Serializable**에선 **ORA-08177**(can’t serialize)로 실패할 수 있다.

---

## “Consistent로 갱신할 때 생기는 현상”은?

**정확히 말해 ‘Consistent로 갱신’이라는 동작은 없다.** Consistent 모드는 **읽기에서만** 사용된다. 다만 DML 구문도 내부적으로 **읽기 단계(Consistent)** 를 수행해 **대상 행을 식별**한다.

**그럼 “Consistent로 갱신할 때 생기는 현상”의 의도는?**
→ **DML의 “읽기 단계”가 Consistent이기 때문에 생기는 현상**을 뜻한다고 해석한다:

- **READ COMMITTED**:
  - UPDATE/DELETE의 **WHERE 평가**가 **Consistent(문장 시작 시점)** 이다.
  - 그러나 **실제 갱신은 Current로** 하므로, **갱신 시점에 행이 바뀌었거나 사라졌으면** 재확인 결과 **해당 행이 갱신 대상에서 빠질 수 있다**.
  - 즉, “WHERE로 걸렸던 것 같은데 마지막에 갱신 카운트가 줄었다”가 가능.
- **SERIALIZABLE**:
  - 트랜잭션 전체에 대해 **스냅샷 일관성**을 유지하려 하므로,
  - **읽기 단계 이후 행이 바뀐 사실**을 감지하면 **ORA-08177**(serialization failure)이 발생해서 “재시도”가 필요할 수 있다.

---

## “Current로 갱신할 때 생기는 현상”

- **현재 버전**을 대상으로 **행 잠금(TX)** 과 **ITL 슬롯** 확보 후 **즉시** 갱신한다.
- **동일 행 경쟁** 시:
  - 상대가 **먼저 락 보유** → 후행 트랜잭션은 **`enq: TX - row lock contention`** 대기.
  - 교착 가능 케이스(교차 순서로 여러 행을 서로 락) → Oracle이 감지 시 한쪽 **ORA-00060(deadlock detected)**.
- **갱신 후 커밋까지**:
  - 해당 블록/행은 **Dirty** 상태. 커밋은 **Redo flush** 로 보장; **DBWn**이 나중에 데이터파일 반영.
  - 다른 SELECT는 **쿼리 SCN** 기준으로 필요한 경우 **Undo** 에서 **과거 버전**을 읽는다(문장 수준 일관성 유지).

---

## 예제: READ COMMITTED에서 “읽기는 Consistent, 쓰기는 Current”

### 준비

```sql
DROP TABLE t_rc PURGE;

CREATE TABLE t_rc (
  id   NUMBER PRIMARY KEY,
  g    NUMBER NOT NULL,
  val  NUMBER NOT NULL
);

BEGIN
  FOR i IN 1..10 LOOP
    INSERT INTO t_rc VALUES (i, 1, 0);
  END LOOP;
  COMMIT;
END;
/
```

### 세션 A — 느린 UPDATE(읽기-쓰기 사이 틈 만들기)

```sql
-- 세션 A
-- 읽기 단계(Consistent)가 시작되게 WHERE 범위를 넓게
UPDATE t_rc
   SET val = val + 1
 WHERE g = 1;

-- (의도적으로 지연: SQL*Plus에선 어렵지만, PL/SQL 블록으로 row-by-row 처리하며 DBMS_LOCK.SLEEP 넣어도 됨)
-- 실제로는 같은 문장 안에서 Consistent 읽기가 끝나고 Current 수정이 진행되는 동안 시간차가 발생할 수 있다.
```

### 세션 B — 틈새에 일부 행을 먼저 바꾸고 커밋

```sql
-- 세션 B
UPDATE t_rc SET val = 100 WHERE id BETWEEN 1 AND 3;
COMMIT;
```

### 결과 관찰(핵심 포인트)

- 세션 A의 UPDATE는 **Consistent 읽기**로 잡아둔 후보 중, **Current 갱신 시점**에 이미 값/상태가 바뀐 행(1~3)에 대해
  - **행 락 충돌**로 기다리거나(순서에 따라)
  - 혹은 **재확인 결과** 최종 갱신 대상에서 빠져 **A의 영향 행 수가 줄어든다**(마지막 결과 카운트가 예상보다 작을 수 있음).
- READ COMMITTED에선 **이런 “최종 영향 행 수 변동”이 정상 동작**이다(문장 중 Consistent, 갱신은 Current).

**요약**: **RC에서 UPDATE/DELETE는 “읽기는 Consistent, 쓰기는 Current”이므로, 사이에 다른 커밋이 끼면 최종 갱신 결과가 변한다.**

---

## “잃어버린 갱신(Lost Update)” 가능?

**Lost Update** 정의: 두 세션이 **같은 행의 “옛 값”을 읽고** 그 값을 바탕으로 **서로 덮어쓴다** → **먼저 쓴 값이 마지막 커밋에 의해 사라짐**.

Oracle의 **READ COMMITTED** 에서는 **Lost Update가 가능**하다(기본적으로 방지하지 않음).
방지하려면 **`SELECT ... FOR UPDATE`**, **버전 칼럼(낙관적 잠금)**, **체크 제약/트리거** 등 별도 장치가 필요.

### 준비

```sql
DROP TABLE t_lu PURGE;
CREATE TABLE t_lu (id NUMBER PRIMARY KEY, val NUMBER);
INSERT INTO t_lu VALUES (1, 0);
COMMIT;
```

### 타임라인

- **세션 A**:
  ```sql
  -- 옛 값 읽기(Consistent)
  SELECT val FROM t_lu WHERE id = 1;  -- 0
  -- 복잡한 로직... (시간 경과)
  ```

- **세션 B**:
  ```sql
  UPDATE t_lu SET val = val + 10 WHERE id = 1;  -- 0 -> 10
  COMMIT;
  ```

- **세션 A**(늦게 도착):
  ```sql
  -- A는 여전히 "처음 읽은 0"을 기준으로 계산했다고 가정
  UPDATE t_lu SET val = 0 + 1 WHERE id = 1;  -- 1로 덮어씀
  COMMIT;
  ```

**결과**: 최종 `val = 1` 이다. 세션 B의 변경(10)이 **유실**되었다.

**해결**:
- **`SELECT ... FOR UPDATE`** 로 행 락을 잡고 시작
- **버전 칼럼(예: `ver` 증가)** 또는 **ORA_ROWSCN/해시 등**을 WHERE에 포함해 **동시 수정 감지**
- 애플리케이션 계층에서 **낙관적 락 실패 시 재시도**

---

## 예제: Serializable에서 Consistent→Current 갱신 시 **ORA-08177**

**Serializable** 은 트랜잭션 전체를 스냅샷 기준으로 보장하려고 시도한다.
이때 **읽은 후** 같은 데이터에 **경쟁 변경이 커밋**되면, 나중에 **Current로 갱신**하려는 순간 **ORA-08177**(can’t serialize)로 실패할 수 있다.

### 준비

```sql
ALTER SESSION SET ISOLATION_LEVEL = SERIALIZABLE;

DROP TABLE t_ser PURGE;
CREATE TABLE t_ser (id NUMBER PRIMARY KEY, val NUMBER);
INSERT INTO t_ser VALUES (1, 0);
COMMIT;
```

### 타임라인

- **세션 A**(Serializable):
  ```sql
  SELECT val FROM t_ser WHERE id = 1;  -- 스냅샷 시점: 0
  -- ...
  ```

- **세션 B**(일반 RC):
  ```sql
  UPDATE t_ser SET val = 100 WHERE id = 1;
  COMMIT;
  ```

- **세션 A**(계속):
  ```sql
  UPDATE t_ser SET val = val + 1 WHERE id = 1;
  -- ORA-08177: can't serialize access for this transaction
  ROLLBACK;
  ```

**요지**: Serializable은 **“읽은 스냅샷에 맞춰 쓸 수 있어야 한다”** 를 보장하려고 해서, 사이 경쟁이 있으면 **실패로 되돌린다**(재시도 필요).

---

## “Consistent로 갱신 대상을 식별하고 Current로 갱신” — 정밀 타임라인

다음은 UPDATE가 내부적으로 거치는 과정을 **행 단위**로 그린 것이다.

1) **Consistent 읽기 단계**
- 인덱스/테이블에서 WHERE 조건으로 **ROWID 후보 리스트** 생성(Undo 기반 복원 포함).
- 이 단계에서 **문장 수준** 일관성 보장.

2) **Current 갱신 단계**(ROWID별 루프)
- 해당 ROWID가 속한 **현재 버전 블록**을 **Current** 로 읽음(`db block gets`).
- **버퍼 pin**, **ITL 슬롯 확보**(필요 시 `enq: TX - allocate ITL entry`), **TX 락 취득**.
- **해당 행이 여전히 WHERE 조건 만족하는지 재확인**(필요 시 옵티마이저/실행기 전략에 따라 다를 수 있으나 보통 필터 재평가):
  - 만족 → 변경 적용, **Redo** 레코드 생성, **Dirty** 플래그.
  - 불만족 → 스킵(최종 영향 행 수 감소).
- 모든 ROWID 처리 후 **COMMIT** → **LGWR flush**.

이 재확인/충돌 처리에서 발생하는 대기/거절이 **read committed와 serializable에서 다르게 표정**으로 나타난다.

---

## “오라클에서 일관성 없게 값을 갱신하는 사례” — 어떤 장면을 의미?

문장 수준 일관성은 **읽기**를 위한 개념이다.
**갱신의 일관성 부족(= 업데이트 anomaly)** 은 보통 다음과 같은 장면을 가리킨다.

### **Lost Update** (앞서 데모) — RC에서 예방 없이 읽고 나중에 덮어쓰기

- 해결: `SELECT ... FOR UPDATE`, 버전 컬럼, 재시도 정책

### **Write Skew** (스냅샷/Serializable 류에서 전형)

- **두 트랜잭션**이 동일 **전역 제약**을 **각자 Consistent 스냅샷** 기반으로 검사 후,
- **서로 다른 행**을 갱신하여 **전역 제약 위반**을 만들어 내는 현상.
- 오라클 **Serializable** 은 **스냅샷 격리 성격**을 보이므로, **같은 행을 직접 충돌하지 않으면** **ORA-08177 없이** **Write Skew** 가 통과할 **가능성**이 있다(전역 제약이 DB 레벨에서 강제되지 않으면).
- 해결: **체크 제약/유니크/외래키** 등 **DB 제약**으로 전역 불변식을 강제하거나, **락 테이블** 등으로 **인위적 충돌** 유발.

**예시 — 당직 의사 최소 1명 규칙(전역 제약)**

```sql
DROP TABLE duty PURGE;
CREATE TABLE duty(
  doctor VARCHAR2(30) PRIMARY KEY,
  oncall CHAR(1) CHECK (oncall IN ('Y','N'))
);

INSERT INTO duty VALUES ('Alice','Y');
INSERT INTO duty VALUES ('Bob','Y');
COMMIT;

-- 제약: 항상 'Y'가 최소 1명 있어야 함 (현재는 두 명)
-- 아래는 DB에 '한 명 이상'을 강제하는 체크 제약이 없다고 가정
```

- **T1**(Serializable):
  ```sql
  -- 스냅샷에서 'Y'가 2명 보인다
  SELECT COUNT(*) FROM duty WHERE oncall='Y';  -- 2
  -- 논리: "내가 Bob을 N으로 바꿔도 Alice가 Y니까 OK"
  UPDATE duty SET oncall='N' WHERE doctor='Bob';
  -- 아직 커밋 안 함
  ```

- **T2**(Serializable):
  ```sql
  -- T1 커밋 전 스냅샷에도 'Y'가 2명
  SELECT COUNT(*) FROM duty WHERE oncall='Y';  -- 2
  -- 논리: "내가 Alice를 N으로 바꿔도 Bob이 Y니까 OK"
  UPDATE duty SET oncall='N' WHERE doctor='Alice';
  -- 커밋!
  COMMIT;   -- 성공
  ```

- **T1** 커밋:
  ```sql
  COMMIT;   -- 성공해 버릴 수도 있음 → 최종적으로 'Y'가 0명(전역 제약 위반)
  ```

**설명**
- 두 트랜잭션이 **서로 다른 행**을 바꾸어 **전역 제약**을 깨는 **Write Skew**.
- 해결은 **DB에 전역 제약(예: CHECK + 서브쿼리 불가 → 트리거/락 테이블/앱단 보완)** 을 **강하게** 두는 것.

### **Non-deterministic UPDATE**

- 동일 문장 내부에서도 **Non-deterministic 함수**(예: `DBMS_RANDOM.VALUE()`), **ROWNUM 순서 업데이트**, **병렬 실행** 등으로 **결과가 실행마다 달라지는** 사례(일관성의 의미가 *트랜잭션 일관성*이 아닌, *결과 결정성* 관점).
- 비즈니스 “일관성”을 말할 때 혼동되므로 주의.

---

## SELECT FOR UPDATE — Consistent vs Current의 연결 고리

- `SELECT ... FOR UPDATE` 는 **행 잠금**을 얻는 **Current Read** 이다.
- 이 구문으로 **갱신 대상을 식별**하면, 이어지는 UPDATE/DELETE에서 **동일 행을 즉시 수정**할 수 있으며, **Lost Update** 를 방지한다.

**데모**

```sql
-- 세션 A
SELECT val FROM t_lu WHERE id=1 FOR UPDATE;  -- current read + TX 락
UPDATE t_lu SET val = val + 1 WHERE id=1;    -- 즉시 갱신 가능
COMMIT;

-- 세션 B (동일 행 시도)
UPDATE t_lu SET val = val + 10 WHERE id=1;   -- A의 커밋까지 TX row lock contention 대기
```

---

## 진단/관측에 유용한 뷰

```sql
-- 읽기/쓰기 패턴 감: 일관/현재/물리 I/O
SELECT name, value
FROM   v$sysstat
WHERE  name IN ('consistent gets','db block gets','physical reads','user commits','redo size')
ORDER  BY name;

-- 현재 대기(락/버퍼)
SELECT sid, event, wait_class, state, seconds_in_wait
FROM   v$session
WHERE  state <> 'WAITED SHORT TIME'
AND    (event LIKE 'enq: TX%'
    OR  event LIKE 'buffer busy waits'
    OR  event LIKE 'read by other session')
ORDER  BY seconds_in_wait DESC;

-- ORA-08177(Serializable) 빈도는 AWR/알러트 로그/애플리케이션 레벨에서 관찰
```

---

## 설계 및 선택 시 고려사항

1.  **Lost Update 방지**가 필요한가?
    - 예/금액/포인트 등 누적형 → `SELECT ... FOR UPDATE` / 버전 컬럼(낙관적 잠금) / MQ 단일 처리
2.  **전역 제약(Write Skew 위험)** 이 있는가?
    - 스냅샷/Serializable에서도 어긋날 수 있음 → **DB 제약**(유니크/참조무결성/트리거/락 테이블)로 **강제**
3.  **일관성의 범위**
    - 문장 수준으로 충분? 트랜잭션 전체 스냅샷 필요? → READ COMMITTED vs SERIALIZABLE 선택
4.  **성능 트레이드오프**
    - `FOR UPDATE`는 **락 경합**과 **db block gets** 증가를 동반
    - 낙관적 잠금은 **충돌 시 재시도 비용** 증가
5.  **재현 불가/비결정성 업데이트** 금지
    - ROWNUM 기반 임의 업데이트, DBMS_RANDOM, 병렬 무질서 업데이트 → **안정된 ORDER BY + 키 기반 처리**로 통제

---

## 결론

오라클에서 데이터를 읽고 갱신하는 과정은 **Consistent 모드**와 **Current 모드**라는 두 개의 축으로 명확히 구분됩니다. Consistent Read는 문장 수준의 일관된 읽기를 보장하기 위해 Undo를 활용한 과거 버전 재구성을 수행하며, Current Read는 실제 데이터 변경을 위한 현재 버전 접근과 락 획득을 담당합니다.

표준 DML 경로인 **“Consistent로 대상 식별 → Current로 갱신”** 은 강력한 일관성 모델의 기초이지만, 두 단계 사이에 발생할 수 있는 경쟁 상태는 다양한 갱신 이상(Update Anomaly)을 초래합니다. **Read Committed**에서는 Lost Update가, **Serializable**(스냅샷 격리)에서는 Write Skew가 발생할 가능성이 있으며, 각각 `SELECT FOR UPDATE`, 버전 컬럼, 데이터베이스 수준의 강한 제약 조건 등을 통해 방지할 수 있습니다.

결국 오라클의 동시성 제어는 **“읽기는 과거(Consistent), 쓰기는 현재(Current)”** 라는 원칙 아래, 개발자가 트랜잭션 격리 수준과 적절한 잠금 기법을 이해하고 설계에 반영할 때 그 위력을 발휘합니다. 동시성 문제를 해결하는 핵심은 이 두 모드의 동작 차이와 그 사이에서 벌어지는 경쟁의 본질을 정확히 아는 데 있습니다.