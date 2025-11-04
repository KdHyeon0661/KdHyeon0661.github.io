---
layout: post
title: DB 심화 - Oracle LOCK
date: 2025-10-19 21:25:23 +0900
category: DB 심화
---
# Oracle **LOCK** 완전 가이드  
— **Enqueue(엔큐) Lock**, **TX Lock**(무결성 제약·비트맵 인덱스 갱신·ITL 부족·인덱스 분할), **기타 트랜잭션 락**, **DML Row Lock / Table Lock**, 그리고 **락을 푸는 열쇠 = COMMIT**  

> 목표  
> - Oracle의 **락 레이어**(Row/TX, Table/TM, 기타 Enqueue)를 개념부터 동작까지 정확히 설명  
> - **TX 락**이 왜, 언제, 어떻게 잡히는지 — 특히 **무결성 제약 위배 가능성**, **비트맵 인덱스 엔트리 갱신**, **ITL 슬롯 부족**, **인덱스 분할** 상황을 실무 관점에서 해부  
> - **DML Row Lock / Table Lock**의 상호작용과 모드 호환성, **데드락**과 진단, **해결(커밋/롤백)** 을 예제와 함께 제시  
> - `V$LOCK`, `V$SESSION`, `DBA_BLOCKERS/WAITERS`, `V$TRANSACTION` 등 **진단 SQL**과 **튜닝 체크리스트** 제공

---

## 0) 큰 그림 — Oracle 락 레이어 맵

- **Row-Level Lock**: 개별 행에 대한 잠금(비가시적 구조) — *세션은 실제론 **TX enqueue**를 통해 보유*.  
- **TX Enqueue**: 트랜잭션(세그먼트 헤더/UNDO)에 대한 락. **행 변경의 소유권**을 표현, **행 충돌 시 대기 이벤트**는 보통 `enq: TX - row lock contention`.  
- **TM Enqueue**: **테이블(오브젝트)** 레벨. DML/DDL 충돌을 제어(모드: RS/RX/SSX/… → 호환성 표 참조).  
- **그 외 Enqueue**: `HW`(High Water Mark), `SQ`(Sequence), `ST`(Space Transaction), `CF`(Control File) 등.  
- **래치/뮤텍스**는 **SGA 보호**용(동시성 제어)으로, **엔큐(사용자 데이터 보호)** 와 다른 레이어.

---

## 1) Enqueue(엔큐) Lock 핵심

### 1.1 구조·이름·모드
- `V$LOCK.TYPE` 예: **TX, TM, UL, HW, SQ, ST, CF…**  
- **모드(MODE)** 값(정수) — 일반적으로:  
  - **0**: None, **1**: Null, **2**: Row-S(=SS), **3**: Row-X(=SX), **4**: S, **5**: S/Row-X, **6**: X  
  - 제품 버전에 따라 표기가 다를 수 있으나 **호환성 매트릭스**는 동일 개념.

### 1.2 호환성 개념(요지)
- **X** vs **X**: 비호환 → 한쪽 대기  
- **S** vs **S**: 호환  
- **SX(=RX)**: S와는 부분 호환, X와는 비호환 … (**TM 호환성 표**는 §4.2)

### 1.3 관측·진단 기본
```sql
-- 세션/락
SELECT s.sid, s.serial#, s.username, l.type, l.id1, l.id2, l.lmode, l.request, l.block
FROM   v$lock l JOIN v$session s ON s.sid = l.sid
ORDER  BY s.sid;

-- 블로커/웨이터
SELECT * FROM dba_blockers;
SELECT * FROM dba_waiters;

-- 트랜잭션 핸들
SELECT s.sid, t.xidusn, t.xidslot, t.xidsqn, t.status
FROM   v$transaction t JOIN v$session s ON s.taddr = t.addr;
```

---

## 2) **TX Lock** — 행 변경의 주연 배우

> **핵심**: 모든 DML(UPDATE/DELETE/INSERT)은 **행 레벨 락**을 잡고, 그 소유권이 **TX Enqueue**로 표현된다.  
> 다른 세션이 **같은 행**을 바꾸려 하면 **TX 락 충돌**로 대기(`enq: TX - row lock contention`).

### 2.1 기본 시나리오 — 행 락 충돌
```sql
-- 준비
DROP TABLE t_tx PURGE;
CREATE TABLE t_tx(id NUMBER PRIMARY KEY, v NUMBER);
INSERT INTO t_tx SELECT LEVEL, 0 FROM dual CONNECT BY LEVEL <= 10;
COMMIT;

-- 세션 A: 행 잠금
UPDATE t_tx SET v = v+1 WHERE id=1;

-- 세션 B: 같은 행 변경 시도 → A 커밋까지 대기
UPDATE t_tx SET v = v+10 WHERE id=1;
-- B는 'enq: TX - row lock contention'
```

**해결**: **A가 COMMIT/ROLLBACK** 하면 B가 진행 → **락을 푸는 열쇠는 커밋**(§7).

---

## 3) TX Lock이 “예상 밖”으로 길어지는 4가지 대표 원인

### 3.1 **무결성 제약 위배 가능성**(FK 미인덱스 등)
- **외래키(FK)** 가 **인덱스 없음** 상태에서 **부모 삭제/수정** 시:  
  - Oracle은 **자식 테이블 전체 범위를 확인**해야 하므로 **자식 테이블 TM/TX 잠금**이 **광범위/장시간** 발생.  
  - 동시에 다른 세션이 자식에 DML → **교착/대기** 증가.

**해결 가이드**  
- **모든 FK 컬럼에 인덱스** 생성 (정합성 + 동시성 필수 교양).
```sql
CREATE INDEX ix_child_fk ON child(parent_id);
```

### 3.2 **비트맵 인덱스 엔트리 갱신**
- **Bitmap Index** 는 **값 하나당 다수 행**을 한 비트맵으로 관리.  
- 특정 값 업데이트/삽입은 **넓은 범위의 행 잠금**(또는 인덱스 엔트리 잠금)을 유발 → **TX 대기** 커짐.  
- **OLTP/높은 동시성 DML**엔 부적합. **카디널리티 낮은 컬럼이라도** 갱신이 잦으면 **B*Tree** 권장.

**해결 가이드**  
- 갱신 빈도가 높은 테이블은 **비트맵 인덱스 지양**. 보고서/집계 전용(읽기 위주)에만 사용.

### 3.3 **ITL 슬롯 부족(Interested Transaction List)**
- 동시 갱신이 **같은 블록**에 몰리면, 블록 헤더의 **ITL 슬롯**이 모자라서 **추가 TX가 기다림**  
  - 대기 이벤트 예: `enq: TX - allocate ITL entry`  
- **원인**: `INITRANS` 너무 낮음, `PCTFREE` 낮아 헤더 확장 여지 없음, **핫 블록 설계**.

**해결 가이드**  
```sql
ALTER TABLE t_tx MOVE INITRANS 8 PCTFREE 20; -- 블록 헤더 여유↑
ALTER INDEX pk_t_tx REBUILD INITRANS 8;
-- 키 분산(Reverse Key, Hash 파티션, 샤딩)으로 '같은 블록' 집중 완화
```

### 3.4 **인덱스 분할(Index Split) & 핫블록**
- 증가형 키(시퀀스)로 **우측 리프** 집중 → **분할 잦음** + **버퍼 경합**(`buffer busy waits`) → DML 지연  
- 지연 동안 **TX Lock 보유시간**이 늘어나 **연쇄 대기** 확대.

**해결 가이드**  
- **Reverse Key Index**(범위 스캔 없다면), **Hash 파티셔닝**, **Hi/Lo**(랜덤화)로 **삽입 분산**.

---

## 4) 기타 트랜잭션 Lock & **TM(Table) Lock** — DML/DDL 충돌의 교차점

### 4.1 DML이 잡는 Table Lock(TM)
- DML(INSERT/UPDATE/DELETE)은 대상 테이블에 보통 **TM=RX(Row-Exclusive)** 를 건다.  
- SELECT … FOR UPDATE도 대상 테이블 **TM=RX**.  
- DDL은 **TM** 을 더 강한 모드로 요구 → DML과 충돌.

### 4.2 TM 호환성(요지표)
| 요청 | 보유 중 |
|---|---|
| **DDL(Exclusive)** vs DML(RX/RS/SSX) → **비호환** |
| **DML(RX)** vs **DML(RX)** → **호환**(대부분) |
| **LOCK TABLE … SHARE** 등은 그에 맞는 호환성 매트릭스 고려 |

> 실무 감각  
> - “**왜 ALTER TABLE이 안 되지?**” → 동시에 DML이 테이블을 만지고 **TM 충돌**.  
> - “**왜 DML이 멈추지?**” → 누군가 `LOCK TABLE …` 이나 DDL을 잡은 경우.

### 4.3 진단 예시
```sql
-- TM를 보는 간단 예
SELECT s.sid, o.owner, o.object_name, l.type, l.lmode, l.request
FROM   v$lock l
JOIN   v$session s ON s.sid = l.sid
JOIN   dba_objects o ON o.object_id = l.id1
WHERE  l.type = 'TM';

-- 현재 블로킹 체인
SELECT * FROM dba_waiters;
SELECT * FROM dba_blockers;
```

---

## 5) **DML Row Lock** — 실제 충돌은 여기서 발생

- **행 락 자체**는 데이터 블록의 **ITL/Lock Byte** 등 **로우 헤더**로 표현(물리적)되지만,  
  애플리케이션/진단 레벨에선 **TX 락**으로 관측·대기 관리.

### 5.1 NOWAIT / WAIT / SKIP LOCKED
```sql
-- 즉시 실패(UX 빠른 피드백)
SELECT * FROM t_tx WHERE id = 1 FOR UPDATE NOWAIT;   -- ORA-00054

-- 제한 대기
SELECT * FROM t_tx WHERE id = 1 FOR UPDATE WAIT 5;

-- 잠긴 행은 건너뛰고 즉시 반환(큐/병렬 워커)
SELECT id FROM jobs WHERE status='READY'
FOR UPDATE SKIP LOCKED
FETCH FIRST 100 ROWS ONLY;
```

---

## 6) 데드락(ORA-00060)과 “무한 대기” 구분

### 6.1 교착 상황 예
```sql
-- 세션 A
UPDATE t_tx SET v=v+1 WHERE id=1;  -- A holds row 1
-- 세션 B
UPDATE t_tx SET v=v+1 WHERE id=2;  -- B holds row 2

-- A가 id=2를, B가 id=1을 추가로 갱신 시도하면
-- Wait-For Graph 사이클 → Oracle이 한쪽을 ORA-00060으로 강제 종료
```

**대응**  
- **항상 같은 순서**로 다중 자원 잠금  
- **트랜잭션 짧게**(락 보유시간 최소)  
- **오류 핸들링**: 00060 발생 시 **재시도** 로직

### 6.2 무한 대기? 실제로는 “상대가 오래 걸리는 것”
- 상대 트랜잭션이 **긴 작업/분할/ITL 확장 실패/Redo flush 지연** → **대기가 길어짐**  
- 모니터링: `V$SESSION`(EVENT/SECONDS_IN_WAIT), `ASH/AWR` 로 상대 세션의 **현재 작업** 파악.

---

## 7) **락을 푸는 열쇠 = COMMIT** (+ ROLLBACK)

- **TX 락/행 락**은 **해당 트랜잭션의 COMMIT/ROLLBACK 순간**에 해제.  
- **TM 락**은 DML/DDL 구간 종료 시 해제(트랜잭션 경계와 일치하는 경우가 많음).  
- **주의**: `SELECT … FOR UPDATE` 로 잡은 행 잠금도 **커밋/롤백**으로만 해제.

```sql
-- 세션 A
SELECT * FROM t_tx WHERE id=1 FOR UPDATE;
-- 잠금 유지 중 …

COMMIT;  -- 이 순간 잠금 해제 → 대기 중인 세션이 진행
```

> **수학적 감각(개념)**  
> $$ \text{Blocked\ Time} \approx \sum \text{Hold\ Time(TX/TM)} + \text{I/O/Latch\ Delays} $$
> **Hold Time** 을 줄이는 가장 큰 지렛대는 **COMMIT 타이밍**과 **트랜잭션 길이**.

---

## 8) “특수” TX 상황 — 사례별 자세한 해설

### 8.1 **무결성 제약 위배 가능성**  
- **상황**: 부모/자식 관계에서 **부모 변경**(DELETE/UPDATE PK), 자식 FK에 **인덱스 없음**.  
- **동작**: Oracle은 자식의 FK 검사 위해 **테이블 스캔/잠금** → **TM/RX 확대** + **TX 대기 증가**.  
- **징후**: 부모 DML 느림, 자식 테이블에 **TM 잠금 다수**, 대기 이벤트 `enq: TM - contention`.  
- **해결**: FK 인덱스 생성, 외래키 정합성 검사 비용을 **지점화**.

### 8.2 **비트맵 인덱스 엔트리 갱신**  
- **상황**: 값 하나에 **여러 행** 연결(비트맵). UPDATE/INSERT가 많으면 **광범위 동시성 저하**.  
- **징후**: TX 대기 증가, 인덱스 블록 경합, DML TPS 하락.  
- **해결**: **B*Tree로 전환**, 비트맵은 **읽기 중심** 환경에 한정.

### 8.3 **ITL 슬롯 부족**  
- **상황**: 동일 블록에 동시 갱신(핫 블록), `INITRANS` 낮음.  
- **징후**: `enq: TX - allocate ITL entry`, `buffer busy waits`, 세그먼트 통계 **ITL waits** 상승.  
- **해결**: `INITRANS/PCTFREE` 확대, 키/파티션 설계로 핫블록 분산.

### 8.4 **인덱스 분할(leaf split)**  
- **상황**: 증가형 키 우측 집중.  
- **징후**: User I/O/Concurrency 대기 동반, TX 보유시간 증가.  
- **해결**: Reverse Key, Hash 파티션, Hi/Lo(랜덤화), RAC에선 파티션-서비스 로컬리티.

---

## 9) 실습 세트 — 락 관측·차단·해소

### 9.1 블로킹 체인 보기
```sql
-- 지금 막힌 세션/막고 있는 세션
SELECT w.waiting_session   AS waiter,
       h.holding_session   AS blocker,
       w.lock_type, w.lock_id1, w.lock_id2
FROM   dba_waiters w JOIN dba_blockers h
     ON w.lock_id1=h.lock_id1 AND w.lock_id2=h.lock_id2;

-- 현재 대기 이벤트
SELECT sid, event, state, seconds_in_wait, blocking_session
FROM   v$session
WHERE  state = 'WAITING'
ORDER  BY seconds_in_wait DESC;
```

### 9.2 특정 객체의 TM 락
```sql
SELECT s.sid, s.serial#, o.owner, o.object_name, l.lmode, l.request
FROM   v$lock l
JOIN   v$session s ON s.sid = l.sid
JOIN   dba_objects o ON o.object_id = l.id1
WHERE  l.type='TM' AND o.object_name = UPPER(:obj);
```

### 9.3 “강제 해소”는 최후수단
```sql
-- 매우 주의! 트랜잭션 날아감
ALTER SYSTEM KILL SESSION '<sid,serial#>' IMMEDIATE;
```
- **권장**: 먼저 **상대 세션과 커뮤니케이션**, 또는 **타임아웃/에러 핸들링**으로 자연 해소.

---

## 10) 설계·튜닝 체크리스트

1) **트랜잭션 짧게**: UI 대기/외부 콜동안 트랜잭션 열지 말 것(“출력-커밋-입력-커밋”).  
2) **커밋 정책**: 너무 잦은 커밋은 `log file sync`↑, 너무 드문 커밋은 **락 보유시간↑** — 업무별 최적화.  
3) **FK 인덱스 전수 점검**: 자식 FK에 **반드시 인덱스**.  
4) **인덱스 전략**: 증가키 핫스팟 완화(Reverse/Hash/파티션/HiLo).  
5) **ITL 여유**: 동시 갱신 테이블/인덱스는 `INITRANS/PCTFREE` 상향.  
6) **비트맵 인덱스 사용처**: 읽기 전용/배치 전용. OLTP 갱신 테이블에 사용 금지.  
7) **NOWAIT/WAIT/SKIP LOCKED**: UX/배치 특성에 맞게 대기 정책 명시.  
8) **데드락 회피**: **잠금 순서 표준화**, 다중 리소스는 **항상 동일 순서**로 접근.  
9) **모니터링**: `DBA_BLOCKERS/WAITERS`, `V$LOCK`, 세그먼트 통계 **ITL waits/buffer busy waits**, AWR/ASH로 상위 이벤트 확인.  
10) **DDL 윈도우**: DDL은 비혼잡 시간, **세션 롱런 DML**과 충돌하지 않게 운영.

---

## 11) 예제 모음 — 실전 패턴

### 11.1 FK 미인덱스로 인한 TM/TX 대기
```sql
-- 부모/자식 준비
DROP TABLE c PURGE; DROP TABLE p PURGE;
CREATE TABLE p(id NUMBER PRIMARY KEY);
CREATE TABLE c(id NUMBER PRIMARY KEY, p_id NUMBER,
               CONSTRAINT fk_c_p FOREIGN KEY (p_id) REFERENCES p(id));
-- FK 인덱스는 만들지 않는다(의도)
INSERT INTO p SELECT LEVEL FROM dual CONNECT BY LEVEL <= 1000;
INSERT INTO c SELECT LEVEL, MOD(LEVEL,1000) FROM dual CONNECT BY LEVEL <= 100000;
COMMIT;

-- 세션 A: 부모 삭제
DELETE FROM p WHERE id = 1; -- 자식 검사로 장시간

-- 세션 B: 자식 삽입/갱신 → TM/TX 대기 폭증
INSERT INTO c VALUES(999999, 1);
```
**개선**:
```sql
CREATE INDEX ix_c_fk ON c(p_id);
```

### 11.2 ITL 부족 재현(개념)
```sql
CREATE TABLE t_itl(id NUMBER, g NUMBER, pad VARCHAR2(100))
INITRANS 1 PCTFREE 0;
INSERT /*+ append */ INTO t_itl
SELECT LEVEL, MOD(LEVEL,5), rpad('x',100,'x')
FROM dual CONNECT BY LEVEL <= 100000;
COMMIT;

-- 여러 세션이 같은 g(=0) 범위를 UPDATE → ITL 부족 대기
UPDATE t_itl SET pad='y' WHERE g=0;  -- 병렬로 실행
-- 관측: 'enq: TX - allocate ITL entry'

-- 개선
ALTER TABLE t_itl MOVE INITRANS 8 PCTFREE 20;
```

### 11.3 증가키 인덱스 분할/핫스팟
```sql
DROP TABLE t_ins PURGE;
CREATE TABLE t_ins(id NUMBER PRIMARY KEY, payload VARCHAR2(50));
-- 기본 인덱스: 우측 리프 집중

-- 대량 동시 삽입 → 분할/경합
INSERT INTO t_ins VALUES(seq.NEXTVAL, 'x'); -- 병렬 워커

-- 개선: Reverse Key (범위 스캔 없다면)
CREATE INDEX pk_t_ins_rev ON t_ins(id) REVERSE;
```

---

## 12) 요약

- **TX Lock** 은 **행 변경 충돌**의 실질 주체이며, **커밋/롤백**으로만 해제된다.  
- **TM Lock** 은 **테이블 레벨**의 DML/DDL 충돌을 조정한다.  
- **TX 보유가 길어지는 4대 요인**:  
  1) **무결성 제약 검사 비용(특히 FK 미인덱스)**  
  2) **비트맵 인덱스 갱신**(OLTP엔 부적합)  
  3) **ITL 슬롯 부족**(INITRANS/PCTFREE/핫블록)  
  4) **인덱스 분할·핫스팟**(증가키 → Reverse/Hash/파티션)  
- **락을 푸는 진짜 열쇠는 COMMIT**. 트랜잭션을 **짧게**, **커밋 타이밍**을 현명하게.  
- **진단**은 `V$LOCK/SESSION`, `DBA_BLOCKERS/WAITERS`, 세그먼트 통계, AWR/ASH로 **탑다운**으로 접근.  
- **설계·운영**으로 충돌 자체를 줄이면(인덱스/파티션/키분산/ITL여유/대기정책) 락 문제의 80%가 사라진다.

> 한 줄 결론  
> **락은 문제의 원인이 아니라 “문제가 드러난 증상”**이다.  
> **TX/TM 레이어**를 분해해서 **왜 오래 잡고 있었는지**를 찾고, **커밋·인덱스·ITL·키 분산**으로 **원인을 치료**하라.
