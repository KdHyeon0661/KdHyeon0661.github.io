---
layout: post
title: DB 심화 - Oracle LOCK
date: 2025-10-19 21:25:23 +0900
category: DB 심화
---
# Oracle **LOCK**

> 목표
> - Oracle의 **락 레이어**(Row/TX, Table/TM, 기타 Enqueue)를 개념부터 동작까지 정확히 설명합니다.
> - **TX 락**이 왜, 언제, 어떻게 잡히는지 — 특히 **무결성 제약 위배 가능성**, **비트맵 인덱스 엔트리 갱신**, **ITL 슬롯 부족**, **인덱스 분할** 상황을 실무 관점에서 해부합니다.
> - **DML Row Lock / Table Lock**의 상호작용과 모드 호환성, **데드락**과 진단, **해결(커밋/롤백)** 을 예제와 함께 제시합니다.
> - `V$LOCK`, `V$SESSION`, `DBA_BLOCKERS/WAITERS`, `V$TRANSACTION` 등 **진단 SQL**과 **튜닝 가이드라인**을 제공합니다.

---

## 큰 그림 — Oracle 락 레이어 맵

Oracle의 동시성 제어는 여러 계층(Layer)의 락(Lock) 메커니즘이 협력하여 이루어집니다. 이해를 돕기 위해 주요 락의 계층을 살펴보겠습니다.

- **Row-Level Lock**: 개별 행에 대한 잠금입니다. 이는 물리적으로 데이터 블록의 ITL 슬롯과 행 헤더의 Lock Byte로 구현되지만, 논리적으로는 다음 계층의 **TX Enqueue**로 표현되어 관리됩니다.
- **TX Enqueue**: 트랜잭션 자체에 대한 락입니다. 특정 트랜잭션이 변경 중인 행에 대한 소유권을 나타내며, **행 충돌 시 관찰되는 대기 이벤트**는 주로 `enq: TX - row lock contention` 입니다.
- **TM Enqueue**: **테이블(또는 다른 오브젝트)** 레벨의 락입니다. DML과 DDL 작업 간의 충돌을 제어합니다. (예: RS/RX/SSX/X 등의 모드)
- **기타 Enqueue**: `HW`(High Water Mark 할당), `SQ`(시퀀스 캐시), `ST`(공간 트랜잭션), `CF`(컨트롤 파일) 등 특수한 목적의 락들이 있습니다.
- **래치(Latch) / 뮤텍스(Mutex)**: **SGA 내 공유 메모리 구조체를 보호**하는 초단기 락입니다. 사용자 데이터 보호를 위한 **엔큐(Enqueue)** 와는 목적과 수명이 다른 별도의 레이어입니다.

---

## 락의 핵심 개념: 구조, 이름, 모드

### 유형(Type)과 모드(Mode)

- `V$LOCK` 뷰의 `TYPE` 컬럼은 락의 유형을 나타냅니다. (예: **TX**(트랜잭션), **TM**(테이블), **UL**(사용자 정의), **HW**, **SQ** 등)
- **모드(MODE)** 는 락의 강도(또는 목적)를 정수로 표현합니다. 일반적인 매핑은 다음과 같습니다(버전별 세부 명칭은 다를 수 있음).
  - **0**: None
  - **1**: Null
  - **2**: Row-S (SS)
  - **3**: Row-X (SX 또는 RX)
  - **4**: Share (S)
  - **5**: Share/Row-X (SSX)
  - **6**: Exclusive (X)

### 호환성의 기본 개념

락 모드 간의 호환성은 동시 작업 가능 여부를 결정합니다. 기본 원칙은 다음과 같습니다.
- **Exclusive(X)** 와 **Exclusive(X)** 는 서로 **비호환**합니다. 한쪽이 대기해야 합니다.
- **Share(S)** 와 **Share(S)** 는 일반적으로 **호환**합니다. 여러 세션이 동시에 읽을 수 있습니다.
- **Row-X(RX)** 는 Share(S) 모드와 **부분적으로 호환**할 수 있지만, Exclusive(X) 모드와는 **비호환**합니다.
- 구체적인 **TM 락 호환성 매트릭스**는 별도의 표로 정리됩니다.

### 기본적인 관측과 진단 쿼리

```sql
-- 현재 세션과 연결된 락 정보 확인
SELECT s.sid, s.serial#, s.username, l.type, l.id1, l.id2, l.lmode, l.request, l.block
FROM   v$lock l JOIN v$session s ON s.sid = l.sid
ORDER  BY s.sid;

-- 블로킹 체인을 쉽게 확인 (DBA_BLOCKERS/WAITERS 뷰 활용)
SELECT * FROM dba_blockers;
SELECT * FROM dba_waiters;

-- 활성 트랜잭션과 연결된 세션 정보
SELECT s.sid, t.xidusn, t.xidslot, t.xidsqn, t.status
FROM   v$transaction t JOIN v$session s ON s.taddr = t.addr;
```

---

## **TX Lock** — 행 변경의 핵심 락

> **핵심 원리**: 모든 DML 작업(UPDATE, DELETE, INSERT, SELECT FOR UPDATE)은 대상 **행에 대한 락(Row-Level Lock)** 을 획득하며, 이 락의 소유권은 논리적으로 **TX Enqueue**로 표현됩니다.
> 다른 세션이 **이미 락이 걸린 동일한 행**을 변경하려고 시도하면 **TX 락 충돌**이 발생하며, 이때 `enq: TX - row lock contention` 대기 이벤트를 보게 됩니다.

### 기본적인 행 락 충돌 시나리오

```sql
-- 테스트 환경 준비
DROP TABLE t_tx PURGE;
CREATE TABLE t_tx(id NUMBER PRIMARY KEY, v NUMBER);
INSERT INTO t_tx SELECT LEVEL, 0 FROM dual CONNECT BY LEVEL <= 10;
COMMIT;

-- 세션 A: 특정 행에 대한 락 획득
UPDATE t_tx SET v = v+1 WHERE id=1;
-- 아직 COMMIT하지 않은 상태

-- 세션 B: 동일한 행 변경 시도 → 세션 A의 커밋/롤백까지 대기
UPDATE t_tx SET v = v+10 WHERE id=1;
-- 세션 B는 'enq: TX - row lock contention' 이벤트를 대기하게 됩니다.
```

**해결 방법**: **세션 A가 COMMIT 또는 ROLLBACK을 수행**하면 해당 행에 대한 락이 해제되고, 대기 중이던 세션 B의 작업이 진행됩니다. 락을 푸는 유일한 방법은 트랜잭션을 종료하는 것입니다.

---

## TX Lock 보유 시간이 길어지는 4가지 주요 원인

TX 락이 예상보다 오래 유지되면, 그만큼 다른 세션들의 대기 시간이 길어져 전체 시스템 성능에 영향을 미칩니다. 다음은 보유 시간을 증가시키는 대표적인 원인들입니다.

### **원인 1: 무결성 제약 위배 가능성 (특히, 인덱스 없는 외래키)**

- **상황**: 부모-자식 관계에서 **부모 테이블의 행을 삭제하거나 기본키를 수정**할 때, 자식 테이블의 외래키(FK) 컬럼에 **인덱스가 존재하지 않는 경우**.
- **문제점**: Oracle은 자식 테이블에 해당 부모 키를 참조하는 행이 있는지 확인하기 위해 **자식 테이블 전체를 스캔**해야 할 수 있습니다. 이 과정에서 자식 테이블에 대한 광범위한 **TM(테이블) 락**과 **TX 락**이 발생하며, 이로 인해 동시에 실행되는 다른 자식 테이블 DML 작업들이 장시간 대기할 수 있습니다.
- **해결 지침**: **모든 외래키(FK) 컬럼에 인덱스를 생성**하는 것은 동시성과 성능을 위한 필수 사항입니다.
```sql
-- 자식 테이블의 FK 컬럼에 인덱스 생성
CREATE INDEX ix_child_parent_id ON child_table(parent_id);
```

### **원인 2: 비트맵 인덱스 엔트리 갱신**

- **상황**: **비트맵 인덱스(Bitmap Index)** 는 하나의 인덱스 엔트리가 **동일한 값을 가진 여러 행**을 가리킵니다.
- **문제점**: 특정 값을 가진 한 행을 갱신하거나 삽입하면, Oracle은 해당 비트맵 인덱스 엔트리(및 관련된 다른 엔트리)를 잠가야 합니다. 이는 **논리적으로 광범위한 행들에 대한 락**을 효과적으로 걸게 만들며, 높은 동시성 DML 환경에서 심각한 경합을 유발합니다.
- **해결 지침**: **OLTP 환경**이나 **갱신이 빈번한 테이블**에서는 비트맵 인덱스를 사용하지 않는 것이 좋습니다. 비트맵 인덱스는 **주로 읽기 전용 또는 배치성 보고서 쿼리**에 적합합니다.

### **원인 3: ITL 슬롯 부족**

- **상황**: `INITRANS` 값이 너무 낮거나 `PCTFREE`가 부족하여 블록 헤더 공간이 모자라면, 동일한 블록을 동시에 변경하려는 여러 트랜잭션이 **ITL(Interested Transaction List) 슬롯을 할당받지 못할** 수 있습니다.
- **문제점**: ITL 슬롯을 할당받지 못한 트랜잭션은 `enq: TX - allocate ITL entry` 이벤트를 대기하게 되며, 이는 결국 행 락 획득의 지연으로 이어집니다.
- **해결 지침**:
```sql
-- 테이블의 INITRANS를 늘리고 헤더 확장 공간(PCTFREE)을 확보
ALTER TABLE t_tx MOVE INITRANS 8 PCTFREE 20;
-- 관련 인덱스도 함께 조정
ALTER INDEX pk_t_tx REBUILD INITRANS 8;
-- 근본적으로는 파티셔닝, Reverse Key, 해시 함수 등을 이용해 '핫 블록'을 분산시킵니다.
```

### **원인 4: 인덱스 리프 블록 경합 (핫블록)**

- **상황**: 순차적으로 증가하는 값(시퀀스 등)을 기본키로 사용하는 테이블에서, 모든 삽입이 **인덱스 트리의 가장 오른쪽 리프 블록**으로 집중됩니다.
- **문제점**: 이로 인해 해당 리프 블록에서 빈번한 **분할(split)** 이 발생하고, `buffer busy waits` 경합이 심화됩니다. 이러한 물리적 경합은 DML 작업 자체를 지연시키고, 결과적으로 **TX 락의 보유 시간을 증가**시켜 연쇄 대기를 유발합니다.
- **해결 지침**: **Reverse Key 인덱스**(범위 스캔이 없는 경우), **해시 파티셔닝**, **Hi-Lo 알고리즘**을 통한 키 값 랜덤화 등을 통해 삽입 작업을 여러 블록에 분산시킵니다.

---

## **TM Lock** — DML과 DDL의 충돌 관리자

### DML 작업이 걸는 테이블 락(TM)

- 일반적인 DML 작업(INSERT, UPDATE, DELETE)은 대상 테이블에 **TM 모드 RX(Row-Exclusive)** 락을 겁니다.
- `SELECT ... FOR UPDATE` 문도 마찬가지로 **TM 모드 RX** 락을 겁니다.
- DDL 작업(예: `ALTER TABLE`)은 일반적으로 더 강한 모드(예: **X, Exclusive**)의 TM 락을 요구합니다. 이는 진행 중인 DML 작업과 충돌하여 DDL이 대기하거나, DML이 DDL을 기다리게 만들 수 있습니다.

### TM 락 호환성 요약

| 요청 모드 \ 보유 모드 | RX (Row-X) | S (Share) | X (Exclusive) |
| :--- | :--- | :--- | :--- |
| **RX (Row-X)** | **호환** | **호환** | **비호환** |
| **S (Share)** | **호환** | **호환** | **비호환** |
| **X (Exclusive)** | **비호환** | **비호환** | **비호환** |

> **실무 감각**
> - **"ALTER TABLE이 왜 안 끝나지?"** → 해당 테이블을 대상으로 하는 DML 트랜잭션이 아직 완료되지 않아 **TM 락 충돌**이 발생한 경우입니다.
> - **"갑자기 DML이 매우 느려졌다."** → 다른 세션이 `LOCK TABLE ... IN EXCLUSIVE MODE` 명령을 수행했거나, DDL 작업이 진행 중일 수 있습니다.

### TM 락 진단 예시

```sql
-- 현재 특정 테이블을 잠그고 있는 TM 락 확인
SELECT s.sid, s.serial#, o.owner, o.object_name, l.lmode, l.request
FROM   v$lock l
JOIN   v$session s ON s.sid = l.sid
JOIN   dba_objects o ON o.object_id = l.id1
WHERE  l.type = 'TM' AND o.object_name = 'YOUR_TABLE_NAME';

-- 현재 발생 중인 블로킹 체인 확인
SELECT * FROM dba_waiters;
SELECT * FROM dba_blockers;
```

---

## **DML Row Lock**과 애플리케이션 제어

- **행 수준 락**은 물리적으로 데이터 블록에 구현되지만, 애플리케이션 개발 및 문제 진단 측면에서는 **TX 락**을 통해 그 상태를 파악하고 제어할 수 있습니다.

### NOWAIT / WAIT / SKIP LOCKED 옵션 활용

애플리케이션 설계에 따라 락 대기 방식을 세밀하게 제어할 수 있습니다.

```sql
-- 1. NOWAIT: 락을 즉시 획득할 수 없으면 오류(ORA-00054)를 발생시킵니다.
--    빠른 실패(Fail-fast)가 필요한 UI 요청에 적합합니다.
SELECT * FROM t_tx WHERE id = 1 FOR UPDATE NOWAIT;

-- 2. WAIT N: 지정된 시간(초) 동안 락 획득을 시도합니다. 시간 내 실패하면 오류를 발생시킵니다.
SELECT * FROM t_tx WHERE id = 1 FOR UPDATE WAIT 5;

-- 3. SKIP LOCKED: 이미 잠긴 행은 무시하고, 잠금 가능한 행들만 즉시 반환합니다.
--    다중 워커(Multiple Workers)가 작업 큐(Queue)에서 태스크를 가져갈 때 유용합니다.
SELECT id FROM jobs WHERE status='READY'
FOR UPDATE SKIP LOCKED
FETCH FIRST 100 ROWS ONLY;
```

---

## 데드락(Deadlock)과 장시간 대기 구분하기

### 교착 상태(Deadlock)의 발생과 처리

데드락은 두 개 이상의 트랜잭션이 서로가 가진 리소스의 락을 기다리며 무한정 대기하는 상태를 말합니다. Oracle은 **데드락 감지기(Deadlock Detector)** 가 주기적으로 이를 탐지하고, 한 트랜잭션을 강제로 롤백시켜(`ORA-00060`) 다른 트랜잭션이 진행될 수 있도록 합니다.

```sql
-- 세션 A
UPDATE t_tx SET v=v+1 WHERE id=1; -- 행(id=1)에 대한 락 획득
-- 세션 B
UPDATE t_tx SET v=v+1 WHERE id=2; -- 행(id=2)에 대한 락 획득

-- 이 상태에서:
-- 세션 A가 행(id=2)을 변경하려고 시도 → 세션 B의 커밋을 대기
-- 세션 B가 행(id=1)을 변경하려고 시도 → 세션 A의 커밋을 대기
-- → Wait-For Graph에서 사이클 발생 → Oracle이 한 세션(예: 세션 B)을 ORA-00060으로 종료
```

**예방 및 대응 방안**:
- **리소스 잠금 순서 표준화**: 여러 테이블이나 행을 잠가야 한다면, 모든 트랜잭션에서 **동일한 순서**로 잠금을 요청하도록 설계합니다.
- **트랜잭션 길이 최소화**: 불필요하게 긴 트랜잭션은 데드락 가능성을 높입니다.
- **애플리케이션 오류 처리**: `ORA-00060` 오류가 발생하면 트랜잭션을 롤백하고 사용자에게 알리거나, 안전하게 작업을 **재시도(Retry)** 하는 로직을 구현합니다.

### 장시간 대기(Long Waits)와의 차이

`enq: TX - row lock contention` 이벤트로 장시간 대기하는 것이 항상 데드락은 아닙니다. 단순히 **상대방 트랜잭션이 아주 오래 실행 중**이기 때문에 대기 시간이 길어질 뿐일 수 있습니다. 이 경우 `V$SESSION` 뷰의 `SECONDS_IN_WAIT` 컬럼이나 ASH/AWR 리포트를 통해 **블로킹 세션이 실제로 무엇을 하고 있는지**를 조사해야 합니다.

---

## **락을 해제하는 유일한 열쇠: COMMIT과 ROLLBACK**

- **TX 락(행 락)** 은 해당 락을 획득한 **트랜잭션이 COMMIT 또는 ROLLBACK으로 종료되는 순간**에만 해제됩니다.
- **TM 락** 은 해당 DML/DDL 작업 구간이 끝날 때 해제됩니다. 일반적인 DML의 경우 트랜잭션 종료 시점과 일치합니다.
- **주의사항**: `SELECT ... FOR UPDATE` 로 명시적으로 획득한 행 락도 마찬가지로 **COMMIT/ROLLBACK 전까지 유지**됩니다.

```sql
-- 세션 A
SELECT * FROM t_tx WHERE id=1 FOR UPDATE; -- 행(id=1)에 락 설정
-- ... (락을 유지한 채 다른 작업 수행) ...

COMMIT; -- 이 명령이 실행되는 순간 행(id=1)에 대한 락이 해제됩니다.
-- 이를 기다리던 다른 세션들의 작업이 이제 진행될 수 있습니다.
```

> **개념적 이해**
> $$ \text{대기 시간(Blocked Time)} \approx \sum \text{락 보유 시간(Hold Time)} + \text{I/O 및 래치 지연 시간} $$
> 성능 튜닝의 핵심은 **락 보유 시간(Hold Time)** 을 줄이는 것이며, 이에 가장 큰 영향을 미치는 요소는 **트랜잭션의 길이**와 **COMMIT의 타이밍**입니다.

---

## 특수 TX 락 상황에 대한 상세 분석

### **무결성 제약 위배 가능성 심화 분석**

- **메커니즘**: 자식 테이블에 FK 인덱스가 없을 때 부모 행을 삭제하면, Oracle은 자식 테이블을 **전체 테이블 스캔(FTS) 또는 풀 인덱스 스캔**하여 참조 무결성을 검증합니다. 이 검증 과정에서 자식 테이블에 **높은 수준의 TM 락**이 걸리게 되어, 동시에 실행되는 다른 자식 테이블 DML 작업들과의 충돌 가능성이 크게 증가합니다.
- **징후**: 부모 테이블의 단순 삭제 작업이 예상보다 오래 걸리고, 동시에 자식 테이블을 사용하는 세션들에서 `enq: TM - contention` 대기 이벤트가 관찰됩니다.
- **근본 해결**: **FK 컬럼에 반드시 인덱스를 생성**하여, 무결성 검증을 "전체 스캔"에서 "인덱스 범위 스캔"으로 변경합니다.

### **비트맵 인덱스 갱신 문제 심화 분석**

- **메커니즘**: 비트맵 인덱스는 하나의 비트맵 엔트리가 많은 행을 대표하므로, 한 행을 갱신해도 해당 비트맵 세그먼트에 배타적 락이 필요합니다. 이는 OLTP 환경에서 동일 인덱스 키 값을 갱신하려는 다수 트랜잭션 간의 심각한 직렬화(Serialization)를 초래합니다.
- **징후**: DML 성능(TPS)이 저하되고, `enq: TX - row lock contention`과 함께 인덱스 블록 관련 `buffer busy waits`가 증가합니다.
- **근본 해결**: 갱신이 빈번한 컬럼에는 **B*Tree 인덱스**를 사용합니다. 비트맵 인덱스는 데이터 웨어하우스처럼 **갱신이 거의 없는 읽기 중심 환경**에만 제한적으로 적용합니다.

### **ITL 슬롯 부족 문제 심화 분석**

- **메커니즘**: 각 데이터 블록 헤더에는 동시에 해당 블록을 변경할 수 있는 트랜잭션 수를 제한하는 ITL 슬롯이 있습니다. `INITRANS`는 초기 슬롯 수를, `PCTFREE`는 슬롯이 확장될 수 있는 헤더 공간을 결정합니다. 슬롯이 부족하면 새로운 트랜잭션은 블록 변경을 시작조차 할 수 없어 대기합니다.
- **징후**: `enq: TX - allocate ITL entry` 대기 이벤트가 빈번히 발생하며, `V$SEGMENT_STATISTICS` 뷰에서 해당 오브젝트의 "ITL waits" 통계가 높게 나타납니다.
- **근본 해결**: `INITRANS`와 `PCTFREE`를 적절히 상향 조정하고, 파티셔닝 등을 통해 워크로드를 여러 블록에 분산시킵니다.

### **인덱스 리프 블록 분할 경합 심화 분석**

- **메커니즘**: 시퀀스 등으로 생성된 단조 증가(Monotonically Increasing) 값을 PK로 사용하면, 모든 새 행이 인덱스의 "가장 오른쪽" 리프 블록에 삽입됩니다. 이 블록이 가득 차면 분할(Split)이 발생하는데, 이 작업 자체가 락을 필요로 하여 동시 삽입들을 일시적으로 직렬화합니다.
- **징후**: 삽입 성능이 저하되고, `buffer busy waits`, `read by other session` 등의 이벤트와 함께 인덱스 분할 관련 대기가 관찰됩니다.
- **근본 해결**: **Reverse Key 인덱스**를 사용하거나(단, 범위 스캔이 불가능해짐), **해시 파티셔닝**을 도입하여 삽입 부하를 여러 인덱스 구조에 분산시킵니다.

---

## 실습 세트 — 락 관측, 차단 확인, 문제 해소

### 현재 시스템의 블로킹 체인 조회

```sql
-- DBA_WAITERS와 DBA_BLOCKERS 뷰를 조인하여 명확한 블로킹 관계 확인
SELECT w.waiting_session   AS waiter,
       h.holding_session   AS blocker,
       w.lock_type, w.lock_id1, w.lock_id2
FROM   dba_waiters w JOIN dba_blockers h
     ON w.lock_id1=h.lock_id1 AND w.lock_id2=h.lock_id2;

-- 대기 중인 세션들의 상세 정보 (블로킹 세션 ID 포함)
SELECT sid, event, state, seconds_in_wait, blocking_session
FROM   v$session
WHERE  state = 'WAITING'
ORDER  BY seconds_in_wait DESC;
```

### 강제 세션 종료 (최후의 수단)

```sql
-- 특정 세션을 강제로 종료합니다. 해당 세션의 진행 중인 트랜잭션은 롤백됩니다.
-- 운영 환경에서 사용 시 각별한 주의가 필요합니다.
ALTER SYSTEM KILL SESSION '<sid,serial#>' IMMEDIATE;
```
**권장 사항**: 강제 종료는 문제의 원인을 해결하지 않습니다. 가능하다면 **해당 세션의 사용자와 소통**하여 정상적으로 커밋/롤백을 유도하거나, 애플리케이션 레벨에서 **타임아웃과 오류 처리** 로직을 강화하는 것이 바람직합니다.

---

## 설계 및 운영을 위한 핵심 가이드라인

1.  **트랜잭션 설계**: 트랜잭션은 필요한 최소 시간만 유지하도록 설계합니다. 사용자 입력 대기나 외부 API 호출과 같은 장시간 작업은 트랜잭션 외부에서 처리하도록 합니다.
2.  **커밋 정책 최적화**: 너무 잦은 커밋은 `log file sync` 대기를 증가시키고, 너무 드문 커밋은 락 보유 시간을 늘립니다. 업무 특성(OLTP vs 배치)에 맞는 적절한 커밋 단위를 찾습니다.
3.  **외래키 인덱스 필수화**: 모든 외래키 제약 조건이 걸린 컬럼에는 성능과 동시성을 위해 반드시 인덱스를 생성합니다.
4.  **인덱스 전략 수립**: 순차 증가 키로 인한 핫스팟을 완화하기 위해 Reverse Key, 해시 파티셔닝, Hi-Lo 알고리즘 등을 적절히 활용합니다.
5.  **ITL 슬롯 용량 계획**: 동시 갱신이 많은 테이블과 인덱스는 `INITRANS`와 `PCTFREE` 값을 충분히 높게 설정합니다.
6.  **비트맵 인덱스 사용 제한**: 비트맵 인덱스는 갱신이 거의 없는 데이터 웨어하우스 환경에 한정하여 사용합니다. OLTP 테이블에는 적용을 지양합니다.
7.  **명시적 락 대기 정책 활용**: 애플리케이션 요구사항에 따라 `FOR UPDATE NOWAIT/WAIT/SKIP LOCKED` 옵션을 적극 사용합니다.
8.  **데드락 예방 설계**: 여러 리소스를 잠가야 하는 로직에서는 항상 **동일한 순서**로 잠금을 요청하도록 표준을 정합니다.
9.  **체계적 모니터링**: `DBA_BLOCKERS`, `V$LOCK`, `V$SEGMENT_STATISTICS` 의 "ITL waits", "buffer busy waits" 통계를 정기적으로 점검하고, AWR/ASH 리포트를 통해 상위 대기 이벤트를 분석합니다.
10. **DDL 작업 관리**: 주요 DDL 작업은 사용자 트래픽이 적은 시간대(메인터넌스 윈도우)에 수행하며, 장시간 실행되는 DML 트랜잭션과의 충돌 가능성을 사전에 검토합니다.

---

## 실전 패턴 예제

### 패턴 1: FK 미인덱스로 인한 광역 락 대기

```sql
-- 테스트 환경 구성
DROP TABLE child PURGE; DROP TABLE parent PURGE;
CREATE TABLE parent(id NUMBER PRIMARY KEY);
CREATE TABLE child(id NUMBER PRIMARY KEY, parent_id NUMBER,
                   CONSTRAINT fk_child_parent FOREIGN KEY (parent_id) REFERENCES parent(id));
-- FK 컬럼(parent_id)에 인덱스를 의도적으로 생성하지 않음
INSERT INTO parent SELECT LEVEL FROM dual CONNECT BY LEVEL <= 1000;
INSERT INTO child SELECT LEVEL, MOD(LEVEL,1000) FROM dual CONNECT BY LEVEL <= 100000;
COMMIT;

-- 세션 A: 부모 테이블에서 한 행 삭제 (자식 테이블 전체 스캔 발생)
DELETE FROM parent WHERE id = 1; -- 이 작업은 시간이 걸림

-- 세션 B: 자식 테이블에 새로운 행 삽입 시도 → TM/TX 락 충돌로 대기
INSERT INTO child VALUES(999999, 1);
```
**개선 조치**:
```sql
-- 자식 테이블의 FK 컬럼에 인덱스 생성
CREATE INDEX ix_child_parent_id ON child(parent_id);
```

### 패턴 2: ITL 슬롯 부족 재현

```sql
-- ITL 슬롯을 제한하기 위한 테이블 생성
CREATE TABLE t_itl_test(id NUMBER, grp NUMBER, pad VARCHAR2(100))
INITRANS 1 PCTFREE 0; -- 매우 낮은 설정

INSERT /*+ append */ INTO t_itl_test
SELECT LEVEL, MOD(LEVEL,5), rpad('x',100,'x')
FROM dual CONNECT BY LEVEL <= 100000;
COMMIT;

-- 여러 세션에서 동일한 그룹(grp=0)의 데이터를 동시에 갱신
-- 세션 A, B, C... 에서 동시 실행:
UPDATE t_itl_test SET pad='y' WHERE grp=0;
-- `enq: TX - allocate ITL entry` 대기 이벤트 관찰 가능

-- 개선: INITRANS와 PCTFREE 상향 조정
ALTER TABLE t_itl_test MOVE INITRANS 8 PCTFREE 20;
```

### 패턴 3: 순차 증가 키 핫스팟

```sql
DROP TABLE t_hot_insert PURGE;
CREATE TABLE t_hot_insert(id NUMBER PRIMARY KEY, payload VARCHAR2(50));
-- 기본 PK 인덱스는 우측 리프 블록에 삽입이 집중됨

-- 다중 세션/애플리케이션에서 동시 삽입 수행 시 핫블록 경합 발생
INSERT INTO t_hot_insert VALUES(some_sequence.NEXTVAL, 'data');

-- 개선: Reverse Key 인덱스 적용 (단, 범위 스캔 불가)
DROP INDEX SYS_C0012345; -- 기존 PK 인덱스 삭제
CREATE UNIQUE INDEX pk_t_hot_insert_rev ON t_hot_insert(id) REVERSE;
ALTER TABLE t_hot_insert ADD CONSTRAINT pk_t_hot_insert PRIMARY KEY (id) USING INDEX pk_t_hot_insert_rev;
```

---

## 결론

오라클 데이터베이스의 락 메커니즘은 데이터의 정확성과 일관성을 유지하면서도 높은 동시성을 달성하기 위한 정교한 시스템입니다. **TX 락**은 행 수준의 변경 충돌을 관리하는 실질적인 주체이며, **TM 락**은 오브젝트 레벨에서 DML과 DDL 작업의 조화를 맞춥니다. 성능 문제로 나타나는 `enq: TX - row lock contention`은 단순한 증상일 뿐, 그 이면에는 **FK 인덱스 부재, 비트맵 인덱스 남용, ITL 슬롯 부족, 인덱스 핫스팟**과 같은 구조적 원인이 자리 잡고 있는 경우가 많습니다.

따라서 락 문제를 해결하는 근본적인 접근법은 락 자체를 조작하는 것이 아니라, **락이 오래 보유되게 만드는 조건을 제거**하는 데 있습니다. 트랜잭션을 짧게 유지하고, 적시에 커밋하며, 인덱스 전략을 합리적으로 수립하고, 테이블과 인덱스의 물리적 저장 속성을 워크로드에 맞게 설계하는 것이 모든 것의 시작입니다. 체계적인 모니터링을 통해 문제를 조기에 발견하고, 본 가이드에서 제시한 실전 패턴과 해결 지침을 참고하여 안정적이고 성능良好的인 데이터베이스 환경을 구축하시기 바랍니다.

> **한 줄 요약**
> **락은 원인이 아니라 결과적인 증상입니다.** `TX/TM` 락의 표면을 벗겨내어 **"왜 이 락이 오래 잡혀 있는가?"** 라는 근본 질문에 답을 찾고, **커밋 전략, 인덱스 설계, ITL 관리, 키 값 분산**이라는 네 가지 축에서 해결책을 모색하세요.