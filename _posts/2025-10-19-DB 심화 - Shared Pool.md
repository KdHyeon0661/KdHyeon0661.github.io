---
layout: post
title: DB 심화 - Shared Pool
date: 2025-10-19 16:25:23 +0900
category: DB 심화
---
# Oracle **Shared Pool** 완전 가이드 — **Dictionary Cache(= Row Cache)**, **Library Cache** (19c 기준)

> 목표
> - **Shared Pool**의 구조·역할을 큰 그림으로 정리하고, 그 핵심인 **Dictionary Cache(=Row Cache)** 와 **Library Cache** 를 **내부 동작**까지 깊게 파헤칩니다.
> - **Hard Parse vs Soft Parse**, **Child Cursor 분기**, **Bind Peeking / Adaptive Cursor Sharing**, **Invalidation**, **Pin/Lock** 의 상호작용을 예제와 함께 재현합니다.
> - 실무에서 자주 보는 **대기 이벤트**(예: `library cache: lock/pin`, `cursor: mutex S/X`, `row cache mutex`, `row cache lock`)와 **진단 SQL**, **튜닝 체크리스트**를 제공합니다.
> - **ORA-04031(shared pool memory)** 류 증상, **Shared Pool 자동/수동 사이징**, **DBMS_SHARED_POOL.KEEP** 전략까지 포함합니다.

---

## 큰 그림 — Shared Pool이 하는 일

- **SGA의 Shared Pool** 은 **SQL/PLSQL 실행에 필요한 공용 메타데이터**와 **코드/실행 계획**을 캐시하여,
  - **파싱 비용 감소**(Hard Parse → Soft Parse),
  - **메타데이터 조회 비용 감소**,
  - **동시성 제어**(Pin/Lock/Mutex) 를 통해 **수행 효율과 일관성**을 보장합니다.

**주요 하위 구성**
1) **Dictionary Cache (= Row Cache)**
   - **데이터 사전 메타데이터**(사용자/오브젝트/세그먼트/권한/동의어/파라미터 등)의 **키-값 캐시**
   - 파싱/실행 중 필요한 **사전 정보를 메모리에서 빠르게 제공**
   - 관련 이벤트: `row cache mutex`, `row cache lock`
2) **Library Cache**
   - **SQL 커서(Shared Cursor)**, **PL/SQL 코드**(패키지/함수/프로시저), **Java 소스**, **오브젝트 정의** 의 **구문 트리·실행계획·코드** 캐시
   - **Hard Parse(최초 컴파일) 없이 재사용** → **Soft Parse**
   - 관련 이벤트: `library cache: lock`, `library cache: pin`, `cursor: mutex S/X`, `library cache load lock` 등

---

## Shared Pool 사이징·상태 점검

```sql
-- Shared Pool 전체 상태(메모리, 프리, Reserved)
SELECT pool, name, bytes
FROM   v$sgastat
WHERE  pool = 'shared pool'
ORDER  BY bytes DESC;

-- Reserved 영역(큰 청크 요청 대응)
SELECT * FROM v$shared_pool_reserved;

-- 파라미터 (자동 메모리 사용 시 SGA_TARGET/INMEMORY 등과 상호작용)
SHOW PARAMETER sga_target;
SHOW PARAMETER shared_pool_size;
SHOW PARAMETER shared_pool_reserved_size;
```

> **TIP**
> - **자동 메모리(AMM/ASMM)** 환경에서는 `shared_pool_size`를 직접 강제하기보다 **상위 총량(SGA/PGA)** 와 **워크로드**에 맞게 **자동 튜너**가 움직이게 하는 것이 일반적입니다.
> - **ORA-04031**(shared pool memory) 발생 시: **커서 폭주/리터럴 남발** 제거, **바인드 변수**, **세션/커서 캐시**, **KEEP/핀 전략**, **패키지 컴파일 빈도 관리** 등을 먼저 점검하세요.

---

## Dictionary Cache (Row Cache)

### 무엇을 담나?

- 파싱·실행 중 필요한 **데이터 사전 테이블의 메타데이터**를 **키 기반 캐시**로 보관합니다.
- 예: **`dc_objects`**, **`dc_users`**, **`dc_segments`**, **`dc_constraints`**, **`dc_synonyms`**, **`dc_sequences`**, **`dc_profiles`**, **`dc_histogram_defs`** 등.

> **특징**
> - **행 단위(Row)** 로 캐시(그래서 **Row Cache**)
> - **Library Cache** 보다 **더 자주·짧게** 접근됨(파싱/권한체크/오브젝트 조회)

### 관측 뷰

```sql
-- 캐시 품질, Get/Miss 비율
SELECT parameter, gets, getmisses, getmisses/NULLIF(gets,0) AS miss_ratio,
       scans, scanmisses
FROM   v$rowcache
ORDER  BY miss_ratio DESC NULLS LAST;

-- 부모/자식 엔트리 수준
SELECT cache#, type, parameter, count, usage, getmisses
FROM   v$rowcache_parent
ORDER  BY getmisses DESC;
```

### 관련 대기 이벤트

- **`row cache mutex`**
  - Row Cache 접근 **뮤텍스** 경합(19c 이후 뮤텍스 기반)
  - **많은 동시 Hard Parse**, **오브젝트 생성/DDL 빈발**, **권한 체크 폭주** 등에서 상승
- **`row cache lock`**
  - **Row Cache 엔트리 수정** 시 보호(예: DDL, sequence cache 조정)

### 실습: DDL 폭주로 Row Cache 경합 유도(테스트 환경)

```sql
-- 주의: 테스트 전용
BEGIN
  FOR i IN 1..5000 LOOP
    EXECUTE IMMEDIATE 'CREATE TABLE t_dc_'||i||'(id NUMBER)';
    EXECUTE IMMEDIATE 'DROP TABLE t_dc_'||i;
  END LOOP;
END;
/

-- 진단
SELECT event, total_waits, time_waited_micro/1e6 sec
FROM   v$system_event
WHERE  event IN ('row cache mutex','row cache lock')
ORDER  BY sec DESC;
```

> **튜닝 힌트**
> - 불필요한 **DDL 빈발** 제거(임시 오브젝트 재사용)
> - **시퀀스 cache** 적절히 확장(DDL적 접근 감소)
> - **권한/동의어 정리**로 **row cache lookups** 최소화
> - Hard Parse 억제(바인드 변수, 커서 재사용) → **Row Cache 조회 수** 자체 감소

---

## Library Cache

### 무엇을 담나?

- **SQL 커서 공유**(파싱 트리, 최적화 결과/실행계획, 실행 상태 일부)
- **PL/SQL 오브젝트**(패키지/프로시저/함수/트리거) **코드와 메타**
- **Java 소스/클래스**
- **DDL 오브젝트 정의**(의존성 그래프 포함)

### Hard Parse vs Soft Parse

- **Hard Parse**:
  1) 구문 해석 → 2) 메타데이터 검사(**Row Cache** 사용) → 3) **최적화(플랜 생성)** → 4) **Library Cache**에 **커서 생성·등록**
- **Soft Parse**:
  - 동일 SQL(정규화된 텍스트 + 바인드 자리수 + NLS 등 **동등 조건**)을 **Library Cache에서 찾아** 재사용

> **효과**: Hard Parse 줄이면 **CPU·Row Cache·Latch/Mutex 경합**이 동시에 줄어듭니다.

### 관측 뷰

```sql
-- 커서 재사용/미스
SELECT namespace, gets, gethits, pins, pinhits, reloads, invalidations
FROM   v$librarycache;

-- SQL 커서 현황
SELECT sql_id, plan_hash_value, executions, parse_calls,
       version_count, loaded_versions, invalidations
FROM   v$sqlarea
ORDER  BY parse_calls DESC FETCH FIRST 20 ROWS ONLY;

-- Child Cursor 분기 사유
SELECT sql_id, child_number, reason
FROM   v$sql_shared_cursor
WHERE  sql_id = :sql_id;
```

### Child Cursor(버전) 분기 — 왜 늘어나는가?

- **바인드 타입/길이 상이**, **NLS 설정 차이**, **환경 파라미터 차이**
- **Bind Peeking** → **Adaptive Cursor Sharing(ACS)** 로 **바인드 값 분포** 따라 다른 플랜
- **Object 상태 변경/통계 변경** → **Invalidation** → 재파싱

> **증상**: `version_count` 고도화, `cursor: mutex S/X`, `library cache: lock/pin`, `mutex spin` 증가

### 대기 이벤트

- **`library cache: lock`**: 오브젝트(커서/PLSQL/테이블 정의 등) **정합성 보장** 위한 Lock
- **`library cache: pin`**: **실행/컴파일 중** 오브젝트 **변경 금지**를 위한 Pin
- **`cursor: mutex S/X`**: 커서 헤더 접근 직렬화(공유/배타) — **하드 파싱 폭주**, **리터럴 남발**, **Child Cursor 폭증** 시 상승
- **`library cache load lock`**: 커서를 캐시에 로드하는 **초기 로드 직전 보호**

---

## 예제 시나리오 — 리터럴 남발 vs 바인드 변수

### 준비

```sql
DROP TABLE t_sales PURGE;
CREATE TABLE t_sales (
  id    NUMBER PRIMARY KEY,
  cid   NUMBER NOT NULL,
  amt   NUMBER NOT NULL,
  note  VARCHAR2(30)
);
BEGIN
  FOR i IN 1..100000 LOOP
    INSERT INTO t_sales VALUES (i, MOD(i,1000), MOD(i,100), RPAD('x', 20, 'x'));
    IF MOD(i,1000)=0 THEN COMMIT; END IF;
  END LOOP;
  COMMIT;
END;
/
CREATE INDEX ix_sales_cid ON t_sales(cid);
```

### **리터럴 남발** (나쁜 예)

```sql
-- (의도적으로) 서로 다른 상수 값으로 같은 형태를 수천 번 실행
BEGIN
  FOR k IN 1..5000 LOOP
    EXECUTE IMMEDIATE 'SELECT /* lit */ COUNT(*) FROM t_sales WHERE cid='||MOD(k,1000);
  END LOOP;
END;
/

-- 관측
SELECT sql_text, executions, parse_calls, version_count
FROM   v$sqlarea
WHERE  sql_text LIKE 'SELECT /* lit */ COUNT(*)%'
ORDER  BY parse_calls DESC;
```

**현상**
- **같은 형태**인데 리터럴 값이 달라 **정규화 실패** → **Hard Parse 폭주**
- **Library Cache/Row Cache** 경합, `cursor: mutex S/X`, `library cache: lock` 상승
- **Shared Pool 파편화**/메모리 압박 → **ORA-04031** 위험

### **바인드 변수** (좋은 예)

```sql
VARIABLE b NUMBER
BEGIN
  FOR k IN 1..5000 LOOP
    :b := MOD(k,1000);
    EXECUTE IMMEDIATE 'SELECT /* bind */ COUNT(*) FROM t_sales WHERE cid=:1' USING :b;
  END LOOP;
END;
/

-- 관측
SELECT sql_text, executions, parse_calls, version_count
FROM   v$sqlarea
WHERE  sql_text LIKE 'SELECT /* bind */ COUNT(*)%'
ORDER  BY parse_calls DESC;
```

**효과**
- **하나의 커서**를 재사용 → **Soft Parse 비율↑**, **Row/Library Cache 경합↓**
- `version_count` 도 **ACS 분기** 정도로 제한

> **추가 팁**
> - 리터럴 유입이 통제 어려우면, 일시적으로 **`CURSOR_SHARING = FORCE`** 로 **대부분의 리터럴을 바인드로 대체**(부작용 가능성 주의).

---

## Bind Peeking & Adaptive Cursor Sharing(ACS)

- **Bind Peeking**: 최초 하드 파싱 시 **바인드 값**을 엿보고 통계·선택도를 기반으로 **플랜 결정**
- 이후 **바인드 값 분포**가 다양하면, **ACS** 가 **바인드 셀렉티비티 히스토리**를 학습해 **다른 Child Cursor** 를 생성(= 값 범주별 최적 플랜)

```sql
-- 특정 커서의 Child 분기 사유 확인
SELECT child_number, reason
FROM   v$sql_shared_cursor
WHERE  sql_id = :sql_id;
```

> **튜닝 포인트**
> - **히스토그램 통계**(column usage)와 **ACS 분기**의 **과도화** 확인
> - 값 분포에 민감한 조건은 **플랜 기반 안정화**(SPM) 또는 **강제 힌트** 고려
> - **불필요한 바인드 길이/타입 다변성**(예: NUMBER vs VARCHAR) 제거

---

## Invalidation(무효화) — 언제/왜 생기나?

- **DDL**(테이블 구조 변경, 인덱스 생성/삭제)
- **통계 갱신**(표본/히스토그램)으로 선택도 변동
- **권한/동의어 변경**, **환경 파라미터 변경**
→ 관련 커서/오브젝트가 **무효화**되어, 다음 실행 시 **Hard Parse(또는 Reload)** 필요

```sql
-- 라이브러리 캐시 무효화 카운트
SELECT namespace, invalidations
FROM   v$librarycache
WHERE  namespace IN ('SQL AREA','TABLE/PROCEDURE','BODY','TRIGGER');
```

> **증상**: `library cache: lock/pin` 일시 상승, 첫 실행 지연
> **대응**: 필요 이상의 **빈번한 DDL/통계 갱신** 지양, **배치 시간**으로 이전

---

## Library Cache/Row Cache 관련 대기 이벤트 빠르게 읽기

| 이벤트 | 의미 | 원인/대응 |
|---|---|---|
| `cursor: mutex S/X` | 커서 헤더 접근 직렬화 | 하드 파싱 폭주, 리터럴 남발 → 바인드/커서 재사용 |
| `library cache: lock/pin` | 오브젝트 정합성/실행 보호 | DDL/컴파일/Invalidation 동반, 파싱 폭주 |
| `library cache load lock` | 커서 로드 보호 | 새로운 커서 로드 남발(리터럴) |
| `row cache mutex` | Row Cache 접근 직렬화 | DDL 빈발/권한체크 폭주/하드 파싱 과다 |
| `row cache lock` | Row Cache 엔트리 수정 | DDL/시퀀스/프로파일 변경 |

---

## DBMS_SHARED_POOL.KEEP — 핫 오브젝트 핀(Pin)

- 자주 사용하는 **패키지/PLSQL/커서 템플릿** 등을 **Shared Pool에서 축출되지 않게 고정**
- **주의**: 과도한 KEEP은 **메모리 고정 낭비** → **엄선** 필요

```sql
-- 패키지/프로시저 핀
EXEC DBMS_SHARED_POOL.KEEP('SCOTT.PKG_BILLING','P');  -- P=PL/SQL

-- 커서(정규화 SQL) 핀 예: 커서의 주소/해시값을 찾아 KEEP
SELECT address, hash_value, sql_text
FROM   v$sql
WHERE  sql_text LIKE 'SELECT /* bind */ COUNT(*) FROM t_sales WHERE cid=:1';

-- (고급) KEEP by Address/Hash (버전/에디션 호환성 주의)
-- EXEC DBMS_SHARED_POOL.KEEP('<ADDRESS>,<HASH_VALUE>');
```

---

## 세션/커서 캐시 — 소프트 파싱 보조

```sql
-- 세션 단 캐시
SHOW PARAMETER session_cached_cursors;

-- 파싱/재사용 지표
SELECT name, value
FROM   v$sysstat
WHERE  name IN ('parse count (total)','parse count (hard)','session cursor cache hits')
ORDER  BY name;
```

- **`session_cached_cursors`** 로 **세션당 커서 재오픈** 비용 절감
- **하드 파싱 비율**( `parse count (hard)` / `parse count (total)` )을 **낮게** 유지

---

## ORA-04031 대비 체크리스트

1. **리터럴 유입 통제**: 바인드 변수, `CURSOR_SHARING=FORCE`(임시), SQL 표준화
2. **Child Cursor 폭증 관리**: 타입/길이 일관성, NLS/환경 일치, ACS 과도화 점검
3. **KEEP/Pin 최소화·정제**: 꼭 필요한 것만 고정
4. **PL/SQL 재컴파일 빈발 억제**: 빈번한 DDL/컴파일 배치화
5. **Shared Pool 크기/Reserved**: `v$sgastat`, `v$shared_pool_reserved` 모니터
6. **AWR/ASH** 로 `cursor: mutex`/`library cache:`/`row cache` 상위 원인 파악
7. **패턴 자체 개선**: 많은 세션이 같은 시각에 **하드 파싱**/DDL 하지 않도록 **스케줄링**

---

## 종합 실습 — “나쁜 패턴 → 개선 → 지표 비교”

### 나쁜 패턴 실행

- 리터럴 남발 루프(§4.2)
- 동시에 DDL/리컴파일 루프(§2.4)

### 지표 스냅샷

```sql
-- 전/후 비교용 스냅샷
SELECT event, total_waits, time_waited_micro/1e6 AS sec
FROM   v$system_event
WHERE  event LIKE 'library cache%' OR event LIKE 'cursor:%'
   OR  event LIKE 'row cache%';

SELECT name, value
FROM   v$sysstat
WHERE  name IN ('parse count (total)','parse count (hard)','session cursor cache hits');

SELECT namespace, gets, gethits, pins, pinhits, reloads, invalidations
FROM   v$librarycache;

SELECT parameter, gets, getmisses
FROM   v$rowcache;
```

### 개선책 적용

- 바인드 변수로 전환(§4.3), `session_cached_cursors` 상향, KEEP 최소화, DDL 배치화

### 재측정

- **Hard Parse↓**, **cursor mutex/row cache 대기↓**, **Library/Row cache 미스↓** 기대

---

## 수학적 감각(개념식)

- **평균 파싱 비용**
  $$ \mathrm{Parse\ Cost} \approx \alpha \cdot \mathrm{HardParse} + \beta \cdot \mathrm{SoftParse} $$
  여기서 \( \alpha \gg \beta \). **Hard Parse 비율을 낮추는 것**이 관건.

- **Shared Pool 압력**
  $$ \mathrm{SP\ Pressure} \approx f(\text{#cursors}, \text{child versions}, \text{pin/lock time}, \text{reserved misses}) $$

---

## 마무리 요약

- **Dictionary Cache(=Row Cache)** 는 **메타데이터**를 행 단위로 캐시해 **파싱/권한체크**를 빠르게 합니다.
  - 경합 시 **`row cache mutex/lock`**, DDL/권한 변경/Hard Parse 폭주가 흔한 원인.
- **Library Cache** 는 **SQL/PLSQL 커서**와 **코드/플랜**을 캐시해 **Hard Parse를 Soft Parse로 치환**합니다.
  - 리터럴 남발·Child Cursor 폭증 시 **`cursor: mutex S/X`**, **`library cache: lock/pin`** 이 두드러집니다.
- **핵심 튜닝 원리**
  1) **바인드 변수**로 커서 재사용 극대화
  2) **Child 분기 원인 제거**(타입/길이/NLS/ACS)
  3) **DDL/통계/컴파일** 은 **배치화**로 Invalidation 폭주 방지
  4) **세션/커서 캐시**, **KEEP 최소화**, **Shared Pool 모니터링**
- 이것이 **Shared Pool 안정화 → 파싱/메타 대기 감소 → 전체 응답 개선**으로 이어지는 가장 확실한 길입니다.

> 한 줄 정리:
> **Row Cache** 는 “사전을 빨리 찾게” 하고, **Library Cache** 는 “같은 말을 다시 안 물어보게” 합니다.
> 두 캐시를 **덜 흔들고 더 재사용**하게 설계하는 것이, Oracle 성능의 절반입니다.
