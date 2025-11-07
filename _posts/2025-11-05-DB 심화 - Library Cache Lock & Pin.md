---
layout: post
title: DB 심화 - Library Cache Lock & Pin
date: 2025-11-05 20:25:23 +0900
category: DB 심화
---
# Oracle **Library Cache Lock & Pin** 완전 해설  

> **핵심 요약**  
> - **Library Cache Lock**: *오브젝트의 “정의/메타데이터”*(예: SQL 커서, PL/SQL, 테이블 사전정보 등)가 **변경되지 않도록** 보호하는 **이름 수준(Name/Handle) 직렬화 장치**. (DDL·리네이밍·인밸리데이션·로드 등과 충돌 방지)  
> - **Library Cache Pin**: **이미 로드된 실행체(Heap/Child Cursor/코드 조각)** 를 **실행 중에 교체/에이징/재컴파일**하지 못하도록 **실행체 수준**에서 잡아두는 **별도 직렬화 장치**.  
> - **두 장치를 분리한 이유**:  
>   1) **동시성**: 다수 세션이 같은 코드(Child)를 **동시에 PIN** 하고 **병렬 실행** 가능. 한편 소수의 DDL/컴파일이 정의를 바꿀 때만 **LOCK** 경합.  
>   2) **세분화된 보호**: “정의 보호(LOCK)”와 “실행체 보호(PIN)”의 **범위/수명/모드**가 다르다. 분리하면 **재컴파일·에이징**으로부터 실행을 안전하게 지키면서 **메타 변경**도 독립적으로 직렬화할 수 있다.  
>   3) **성능/스케일링**: 하나의 전역 락으로 모든 것을 보지 않고, **필요 최소 단위**로 보호해 **파싱/실행** 경합을 줄인다.  
> - 대표 대기 이벤트:  
>   - `library cache lock`(정의 변경/획득 충돌)  
>   - `library cache pin`(실행체 보호 중 재컴파일/인밸리데이션 충돌)  
>   - (11g+) `cursor: pin S`, `cursor: pin S wait on X`, `cursor: mutex S/X` 등 **Mutex 기반** 직렬화 이벤트(LOCK/PIN을 더 미세한 Mutex로 대체/보완)

---

## 1. Library Cache의 객체와 직렬화 개요

### 1.1 Library Cache 안에 뭐가 있나
- **SQL AREA**: Parent/Child Cursor(텍스트·파싱 정보·실행계획·바인드 민감도 등)  
- **PL/SQL**: 패키지/프로시저/함수의 컴파일 산출물(코드·메타)  
- **데이터 딕셔너리 캐시 연계 정보**: 테이블/인덱스 메타 참조  
- **기타 네임스페이스**: `TABLE/INDEX/VIEW/SEQUENCE/PROCEDURE/PACKAGE/BODY/TRIGGER/TYPE` …, `SQL AREA`, `CURSOR STATS` 등

### 1.2 KGL 개념(개요)
- **KGL Lock(=Library Cache Lock)**: **오브젝트 핸들(정의/이름)**에 대한 공유/배타 모드 획득  
- **KGL Pin(=Library Cache Pin)**: **오브젝트 힙(실행체, child heap)** 에 대해 **고정(PIN)**  
- **수명**:  
  - **LOCK**: 파싱/DDL/로드/인밸리데이션에 걸쳐 비교적 **짧게 혹은 특정 동작 동안** 보유  
  - **PIN**: 실행·결과 반환 동안 **더 길게** 보유(실행 중 교체/aging 방지)  
- **11g+ Mutex 전환**: 많은 경우 **Latch/LOCK/PIN** 일부가 **Cursor Mutex**로 세분화. 대기 이벤트 명칭이 달라질 수 있음.

---

## 2. 왜 **LOCK**과 **PIN**을 따로 두었나? (설계 동기)

### 2.1 “정의”와 “실행체”의 **보호 대상이 다르다**
- **정의(Definition/Name)**: SQL 텍스트·오브젝트 이름·DDL 구조 등.  
  → **LOCK** 으로 보호.  
  → 예: `ALTER PACKAGE …`, `ALTER TABLE …`, `CREATE OR REPLACE FUNCTION …`, 커서 **하드 파싱/로드** 등
- **실행체(Heaps/Child Cursor)**: **하드파싱 결과물**(실행계획, 코드, 최적화 정보, 힙).  
  → **PIN** 으로 보호.  
  → 예: 실행 도중 **에이징(Shared Pool 단편화/압박)**, **재컴파일**로 인한 교체 방지

### 2.2 **동시성 극대화**
- 수백/수천 세션이 **동일 Child** 를 동시에 실행해도 **각자 PIN** 을 들고 있으면 **자유롭게 실행**할 수 있다.  
- 이때 DDL 같은 **정의 변경**은 LOCK 레벨에서만 직렬화되어 **실행 경합**이 불필요하게 커지지 않는다.

### 2.3 **비용 제어/수명 최적화**
- 실행이 길면 **PIN** 을 오래 들고 있어도 **정의 변경(LOCK)** 은 다른 통로로 제어된다.  
- 반대로 파싱 순간에만 **LOCK** 으로 짧게 보호하고, 실행 들어가면 **PIN** 으로 전환해 **락 홀드 시간 최소화**.

---

## 3. 대기 이벤트로 보는 차이

| 이벤트 | 언제 발생 | 의미/상황 |
|---|---|---|
| `library cache lock` | 오브젝트 정의/이름 수준 LOCK이 필요하나 **다른 세션이 상충** | DDL·컴파일·하드파싱 간 충돌, Dependent 객체 인밸리데이션 직전 |
| `library cache pin` | 실행체 PIN 필요하나 **재컴파일/인밸리데이션/에이징 충돌** | 실행 중 누군가가 동일 Child를 바꾸려 할 때 |
| `library cache load lock` | **로드/리로드(aging 후 재적재)** 직렬화 | 동일 오브젝트를 **동시에** 로드하지 않도록 직렬화 |
| (11g+) `cursor: pin S` / `cursor: pin S wait on X` | Cursor Mutex 충돌 | 다중 세션 실행/하드파싱이 **같은 Child** 를 두고 경합 |
| (11g+) `cursor: mutex S/X` | Mutex 기반 보호 경합 | Cursor 구조/통계 접근 직렬화 |

> 버전에 따라 동일 상황이 **Mutex 이벤트**로 관측되는 경우가 많다.

---

## 4. 재현 시나리오 (실습용)

> 아래 실습은 **두 세션**(세션 A, B)을 사용한다. 실제로는 SYS 권한 없이도 충분히 관찰 가능(일부 내부 X$ 뷰 제외).

### 4.1 **PIN 경합**: 패키지 실행 중 재컴파일 시
**세션 A** – 패키지/함수 실행(오래 걸리게 만들기)
```sql
-- 세션 A
CREATE OR REPLACE PACKAGE rc_demo AS
  FUNCTION work(p INT) RETURN NUMBER;
END rc_demo;
/

CREATE OR REPLACE PACKAGE BODY rc_demo AS
  FUNCTION work(p INT) RETURN NUMBER IS
    v NUMBER := 0;
  BEGIN
    -- 5초 동안 루프 (실행 중 Child PIN 보유)
    FOR i IN 1..5 LOOP
      v := v + p;
      DBMS_LOCK.SLEEP(1);
    END LOOP;
    RETURN v;
  END;
END rc_demo;
/

-- 실제 실행 (동시에 B에서 재컴파일을 시도할 예정)
SELECT rc_demo.work(10) FROM dual;
```

**세션 B** – 동일 패키지 **컴파일**(정의 변경)
```sql
-- 세션 B (A가 실행 중일 때)
ALTER PACKAGE rc_demo COMPILE PACKAGE;     -- 또는 BODY
-- 또는 CREATE OR REPLACE PACKAGE/BODY ...
```

**기대 현상**
- 세션 B가 **컴파일**/정의 변경을 하려면 **LOCK**과 **PIN** 단계가 필요하다.  
- 세션 A가 **Child를 PIN** 하고 실행 중이므로, B는 **`library cache pin`** (혹은 Mutex 이벤트) **대기**가 발생할 수 있다.  
- 세션 A 종료/반환 → PIN 해제 → 세션 B 컴파일 진행.

**관찰**
```sql
-- 대기 관찰 (둘 다에서)
SELECT sid, event, wait_class, blocking_session
FROM   v$session
WHERE  sid = SYS_CONTEXT('USERENV','SID');

-- ASH로 5분 윈도우에서 필터
SELECT event, COUNT(*) samples
FROM   v$active_session_history
WHERE  sample_time > SYSTIMESTAMP - INTERVAL '5' MINUTE
  AND  session_id = <세션B SID>
GROUP  BY event ORDER BY samples DESC;
```

---

### 4.2 **LOCK 경합**: 테이블 DDL과 쿼리 파싱 충돌
**세션 A** – 테이블 DDL(정의 변경)
```sql
-- 세션 A
CREATE TABLE lck_t (id NUMBER PRIMARY KEY, val VARCHAR2(30));
INSERT INTO lck_t VALUES (1,'A'); COMMIT;

-- DDL로 정의 변경(인덱스 추가/컬럼 수정 등)
ALTER TABLE lck_t ADD (val2 VARCHAR2(10));
-- 이 때 해당 오브젝트 네임스페이스에 대한 library cache lock 필요
```

**세션 B** – 해당 테이블을 참조하는 SQL을 **하드 파싱**(바인드 없이)
```sql
-- 세션 B (동시에)
SELECT /* 하드파싱 유도: 리터럴 */
       COUNT(*) FROM lck_t WHERE val = 'A';
```

**기대 현상**
- 세션 A의 **DDL**은 `lck_t` 정의를 바꾸므로 **오브젝트 핸들 LOCK** 필요.  
- 세션 B는 **파싱 중 사전정보 참조/커서 로드** 위해 동일 정의에 대한 LOCK 필요.  
- 타이밍에 따라 B 또는 A가 **`library cache lock`** 대기를 보일 수 있다.

---

### 4.3 **LOAD LOCK**: 에이징된 커서를 동시에 재적재
**세션 A/B** – shared pool 압박 또는 `ALTER SYSTEM FLUSH SHARED_POOL` 후 동일 SQL 동시 수행
```sql
-- (운영 주의) 데모 목적으로만
ALTER SYSTEM FLUSH SHARED_POOL;

-- 두 세션에서 동시에
SELECT /* big literal to avoid share */ COUNT(*) 
FROM   dual CONNECT BY LEVEL <= 100000;
```
**현상**
- 동일 오브젝트/커서를 **동시에 로드**하려 들면 **`library cache load lock`** 으로 **한쪽이 직렬화**된다.

---

## 5. 내부 흐름(직관 모델)

1) **파싱/하드파싱 시점**  
   - 이름 해석 → **Library Cache Handle** 찾기/만들기 → **LOCK 획득**(정의 충돌 방지)  
   - 필요한 힙(Child) 로드/컴파일 → 완료 후 필요 LOCK 해제

2) **실행(Execute/Fetch) 시점**  
   - 대상 **Child Heap을 PIN**(실행 중 교체 금지)  
   - 실행 완료 후 **PIN 해제** (오브젝트는 캐시에 남음, 에이징 후보)

3) **정의 변경/재컴파일/인밸리데이션**  
   - 변경자 세션이 **LOCK/LOAD LOCK** 필요  
   - 다른 세션이 해당 **Child를 PIN** 하고 있으면 변경자는 **PIN 대기**  
   - 실행이 끝나 PIN이 풀리면, 변경을 적용·인밸리데이트

> **포인트**: 파싱/정의 변경은 **LOCK** 레이어, 실행 중 안정성은 **PIN** 레이어.  
> 분리 덕분에 **대부분의 실행**은 서로 **같은 Child를 “읽기-공유”** 하며 달릴 수 있다.

---

## 6. 흔한 원인 & 증상 & 해법

### 6.1 잦은 DDL/컴파일(== 정의 변경)
- **증상**: `library cache lock`/`library cache pin`/`library cache load lock` 증가, 파싱 지연/실행 중 단기 스톨  
- **원인 예**  
  - 배포 스크립트가 **업무시간**에 자주 수행(PL/SQL 재컴파일, Synonym 교체 등)  
  - 빈번한 `CREATE OR REPLACE`(뷰/펑션/패키지)  
  - 자동 코드 생성기/ORM이 스키마를 **빈번히** 건드림  
- **해법**  
  - **DDL 윈도우** 분리(야간 배포)  
  - Edition 기반 롤링, **Editioned Objects** 활용  
  - 불필요 DDL 제거, **변경 묶음** 처리  
  - `DBMS_RESULT_CACHE`/MView 등으로 **정의 의존 감소**

### 6.2 대량 하드 파싱/리터럴 난사
- **증상**: `library cache load lock`, Mutex 핀 대기, Shared pool 단편화  
- **원인**: 바인드 미사용, SQL 텍스트 변종 폭증 → 커서 수 폭증·재로드  
- **해법**  
  - **바인드 변수** 적용(가능한 한), **커서 공유**  
  - `SESSION_CACHED_CURSORS`, `OPEN_CURSORS` 적정  
  - 상위 트래픽 SQL **정규화**(일부 인기 리터럴은 전략적 분리)

### 6.3 통계/DDL에 의한 **인밸리데이션 폭발**
- **증상**: 특정 시간대에 `library cache pin` 스파이크  
- **원인**  
  - `DBMS_STATS` 수행 시 **즉시 무효화**(많은 커서 재하드파싱 유발)  
- **해법**  
  - `DBMS_STATS` 의 **`no_invalidate => DBMS_STATS.AUTO_INVALIDATE`**(또는 TRUE) 활용: **점진적** 무효화  
  - 통계 작업 **분할/스케줄링**(저부하 시간)

### 6.4 Shared Pool 압박/에이징
- **증상**: `library cache load lock`/PIN 대기, 자주 리로드  
- **원인**: Shared pool 크기 부족, 거대 패키지/커서  
- **해법**  
  - Shared pool **사이징** 재평가, 큰 패키지 분할  
  - **PL/SQL Native**/Fat Package 최적화, 불필요 cursor 폐기  
  - 빈번한 `FLUSH SHARED_POOL` 지양

---

## 7. 모니터링/진단 스크립트

### 7.1 현재 대기 & 블로킹 체인
```sql
SELECT sid, serial#, username, event, wait_class,
       blocking_session, seconds_in_wait, state
FROM   v$session
WHERE  event LIKE 'library cache%'
   OR  event LIKE 'cursor:%mutex%'
ORDER BY seconds_in_wait DESC;
```

### 7.2 ASH로 최근 15분 집중 분석
```sql
SELECT event, COUNT(*) samples
FROM   v$active_session_history
WHERE  sample_time > SYSTIMESTAMP - INTERVAL '15' MINUTE
  AND  (event LIKE 'library cache%' OR event LIKE 'cursor:%mutex%')
GROUP  BY event
ORDER  BY samples DESC;
```

### 7.3 어떤 오브젝트/커서가 문제?
> **DBA 권한**이면 `X$KGL` 계열로 해시값/주소→오브젝트 매핑 가능.  
권한이 제한적이라면 **시간 상관**으로 Top SQL/최근 컴파일/DDL 로그를 함께 본다.
```sql
-- Top 컴파일(최근 1시간) 근사치: DBA_OBJECTS LAST_DDL_TIME
SELECT owner, object_name, object_type, last_ddl_time
FROM   dba_objects
WHERE  last_ddl_time > SYSDATE-1/24
ORDER  BY last_ddl_time DESC;

-- Top 하드 파싱 SQL
SELECT sql_id, executions, loads, parse_calls, version_count, sql_text
FROM   v$sqlarea
ORDER  BY parse_calls DESC FETCH FIRST 30 ROWS ONLY;
```

### 7.4 커서 버전/Mutex 신호
```sql
-- child 커서 다중화(버전 폭증) 감시
SELECT sql_id, version_count, loaded_versions, kept_versions
FROM   v$sqlarea
ORDER  BY version_count DESC FETCH FIRST 30 ROWS ONLY;
```

---

## 8. 실전 튜닝 플레이북

1) **증상 확인**: `library cache lock/pin/load lock`, `cursor: pin S` 등 **이벤트 분포** 수집  
2) **원인 유형 분류**  
   - DDL/컴파일/배포? → **LOCK/PIN 스파이크**  
   - 바인드 미사용/하드 파싱 폭증? → **LOAD LOCK/Mutex**  
   - 통계/인밸리데이션? → **PIN 스파이크**  
   - Shared pool 압박? → **LOAD LOCK/리로드**  
3) **즉각**  
   - 배포/DDL **중지/연기**, 통계 **no_invalidate 조정**  
   - 최상위 리터럴 SQL **바인드화**(리스크 낮은 것부터)  
   - 병목 SQL **세션 캐시 커서/커서 재사용** 검토  
4) **근본**  
   - **배포 윈도우 분리**, Edition 기반 교체  
   - SQL 정규화, **Session/Client Result Cache**  
   - Shared pool 사이즈/패키지 크기/설계 개선  
5) **검증**: AWR/ASH/`DBMS_XPLAN` 전후 비교 — 대기·파싱·로드·응답시간 **숫자** 확인

---

## 9. 추가 예제: “동일 Child 커서에 대한 Mutex 경합”

**세션 A** – 대량 실행(Child 핀 공유)
```sql
-- 세션 A
VAR b NUMBER; EXEC :b := 1000;
SELECT /* child 공유 */ COUNT(*) 
FROM orders 
WHERE cust_id = :b;
-- (루프 실행 또는 애플리케이션에서 다중 스레드 호출)
```

**세션 B** – 같은 SQL에 **다른 환경/바인드 패턴**으로 **새 Child 강제**
```sql
-- 세션 B (옵티마이저 환경/세션 파라미터 살짝 변경)
ALTER SESSION SET optimizer_features_enable='19.1.0'; -- 예시
SELECT COUNT(*) FROM orders WHERE cust_id = :b2;
```

**현상**: Child 생성/핀 전환 시 **`cursor: pin S wait on X`** 등 **Mutex 경합**이 발생 가능.  
**대응**: 환경/바인드/스태츠 일관화, **버전 폭증 방지**, 필요 시 힌트로 강제 단일화.

---

## 10. FAQ

**Q1. `library cache lock` 과 `enq: TX - …` 같은 DML 락과의 차이는?**  
- **범위가 다름**: LC Lock은 **Library Cache 내부 메타/코드** 직렬화, TX는 **데이터(행/트랜잭션)** 직렬화.

**Q2. `library cache pin` 은 왜 실행 중에 길게 잡히나?**  
- 실행체(Child Heap)가 **에이징/교체/재컴파일**되는 순간 **메모리 일관성**이 깨질 수 있어 **실행 동안** PIN을 보유.

**Q3. 11g 이후에 왜 Mutex 이벤트가 더 많이 보이나?**  
- Latch 기반 직렬화를 **더 미세한 Mutex**로 대체/보완해 **경합 범위를 축소**. 이벤트 이름만 달라진 경우가 많다.

**Q4. 통계 수집 때문에 핀 대기가 튄다면?**  
- `DBMS_STATS` 의 **`no_invalidate => DBMS_STATS.AUTO_INVALIDATE`** 또는 지연 인밸리데이션 정책 사용, **야간 분산**.

---

## 11. 운영 체크리스트 (요약)

- [ ] 업무시간 DDL/컴파일 금지(배포 윈도우/Edition)  
- [ ] 바인드 변수·커서 재사용(세션 커서 캐시, SQL 정규화)  
- [ ] 통계 수집 시 **점진적 인밸리데이션**  
- [ ] Shared Pool 사이징·거대 패키지 분할·불필요 `FLUSH` 지양  
- [ ] Top 파싱/로드 SQL 확인(리터럴 난사 차단)  
- [ ] ASH/AWR로 `library cache %`/`cursor:%` 이벤트 **지속 모니터링**

---

## 12. 마무리

- **LOCK**은 “**정의/이름을 바꾸는 순간**의 안전벨트”, **PIN**은 “**실행 중 코드/힙을 지키는 안전핀**”.  
- 두 장치를 **분리**했기 때문에 Oracle은 **대규모 동시 실행**과 **드문 정의 변경**을 **서로 방해 없이** 처리할 수 있다.  
- 실전에서는 **DDL·통계·하드파싱** 타이밍을 관리하고, **바인드/커서 재사용**을 지키며, **Shared Pool 건강**을 유지하는 것이 핵심이다.  
- 모든 판단은 **숫자**로: `v$session`/`ASH`/`AWR`/`v$sqlarea`/`DBMS_XPLAN` 으로 **전/후**를 비교하라.  
  그렇게 하면 `library cache lock` & `pin` 문제는 **재현 가능**하게 설명되고 **예측 가능한 방식**으로 줄어든다.