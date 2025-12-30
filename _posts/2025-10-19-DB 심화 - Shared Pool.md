---
layout: post
title: DB 심화 - Shared Pool
date: 2025-10-19 16:25:23 +0900
category: DB 심화
---
# Oracle **Shared Pool** - **Dictionary Cache(= Row Cache)**, **Library Cache** (19c 기준)

> 목표
> - **Shared Pool**의 구조와 역할을 큰 그림으로 정리하고, 그 핵심인 **Dictionary Cache(=Row Cache)** 와 **Library Cache** 를 내부 동작까지 깊이 있게 파헤칩니다.
> - **Hard Parse와 Soft Parse의 차이**, **Child Cursor 분기**, **Bind Peeking 및 Adaptive Cursor Sharing**, **Invalidation**, **Pin/Lock** 메커니즘의 상호작용을 예제와 함께 재현합니다.
> - 실무에서 자주 마주치는 **대기 이벤트**(예: `library cache: lock/pin`, `cursor: mutex S/X`, `row cache mutex`, `row cache lock`)에 대한 **진단 SQL**과 **튜닝 지침**을 제공합니다.
> - **ORA-04031(shared pool memory)** 에러 증상, **Shared Pool 자동/수동 사이징**, **DBMS_SHARED_POOL.KEEP** 전략까지 포함합니다.

---

## 큰 그림 — Shared Pool의 역할

**SGA의 Shared Pool** 은 SQL과 PL/SQL 실행에 필요한 공용 메타데이터, 코드, 실행 계획을 캐싱하는 핵심 메모리 영역입니다. 이를 통해 다음과 같은 성능 향상을 도모합니다.
- **파싱 비용의 감소**: Hard Parse를 Soft Parse로 최소화합니다.
- **메타데이터 조회 비용 감소**: 자주 참조되는 사전 정보를 메모리에서 빠르게 제공합니다.
- **동시성 제어**: Pin, Lock, Mutex 메커니즘을 통해 다수 세션이 안전하게 공유 리소스를 사용할 수 있도록 합니다.

**주요 하위 구성 요소**
1. **Dictionary Cache (= Row Cache)**
   - 데이터베이스 사전의 메타데이터(사용자, 오브젝트, 세그먼트, 권한, 동의어, 파라미터 등)를 키-값 형태로 캐싱합니다.
   - SQL 파싱 및 실행 중 필요한 사전 정보를 디스크 I/O 없이 빠르게 조회할 수 있도록 합니다.
   - 관련 대기 이벤트: `row cache mutex`, `row cache lock`

2. **Library Cache**
   - **SQL 커서(Shared Cursor)**, **PL/SQL 코드**(패키지, 함수, 프로시저), **Java 소스**, **오브젝트 정의**의 구문 트리, 실행 계획, 실행 코드를 캐싱합니다.
   - 최초 컴파일(Hard Parse) 없이 동일한 SQL/PLSQL을 재사용(Soft Parse)할 수 있게 합니다.
   - 관련 대기 이벤트: `library cache: lock`, `library cache: pin`, `cursor: mutex S/X`, `library cache load lock` 등

---

## Shared Pool 사이징 및 상태 점검

Shared Pool의 건강 상태를 확인하는 기본적인 쿼리입니다.

```sql
-- Shared Pool 내부의 다양한 풀과 그 사용량 확인
SELECT pool, name, bytes
FROM   v$sgastat
WHERE  pool = 'shared pool'
ORDER  BY bytes DESC;

-- 큰 메모리 청크 요청을 위한 Reserved 영역 상태 확인
SELECT * FROM v$shared_pool_reserved;

-- 관련 파라미터 확인
SHOW PARAMETER sga_target;
SHOW PARAMETER shared_pool_size;
SHOW PARAMETER shared_pool_reserved_size;
```

> **운영 팁**
> - **자동 메모리 관리(AMM/ASMM)** 환경에서는 `shared_pool_size`를 고정하기보다는 `sga_target`을 적절히 설정하여 Oracle이 워크로드에 맞게 조정하도록 하는 것이 일반적입니다.
> - **ORA-04031** 에러는 Shared Pool 메모리 부족을 의미합니다. 이는 커서 폭주, 리터럴 SQL 남발, 큰 PL/SQL 객체 로드 등이 원인일 수 있습니다. 바인드 변수 사용, 세션 커서 캐시 조정, 불필요한 객체 핀 제거 등을 우선 점검하세요.

---

## Dictionary Cache (Row Cache) 상세 분석

### 저장하는 정보

Dictionary Cache는 데이터베이스 운영에 필수적인 메타데이터를 행(Row) 단위로 캐싱합니다. 파싱, 권한 검사, 오브젝트 확인 시 매번 디스크 기반의 데이터 사전을 읽지 않도록 합니다.

주요 캐시 항목 예시:
- **`dc_objects`**: 오브젝트(테이블, 인덱스 등) 정의 정보
- **`dc_users`**: 사용자 정보
- **`dc_segments`**: 세그먼트(데이터 저장 단위) 정보
- **`dc_constraints`**: 제약 조건 정보
- **`dc_synonyms`**: 동의어 정보
- **`dc_sequences`**: 시퀀스 정보

> **특징**
> - **Library Cache**보다 접근 빈도가 훨씬 높고, 각 접근 시간은 매우 짧습니다.
> - SQL 파싱의 첫 단계에서 필수적으로 참조됩니다.

### 상태 모니터링 쿼리

```sql
-- Row Cache의 효율성(Hit/Miss 비율) 확인
SELECT parameter, gets, getmisses, getmisses/NULLIF(gets,0) AS miss_ratio,
       scans, scanmisses
FROM   v$rowcache
ORDER  BY miss_ratio DESC NULLS LAST;

-- Row Cache의 부모/자식 엔트리 수준에서의 상세 통계
SELECT cache#, type, parameter, count, usage, getmisses
FROM   v$rowcache_parent
ORDER  BY getmisses DESC;
```

### 관련 대기 이벤트

- **`row cache mutex`**
  - Row Cache 엔트리에 대한 접근을 직렬화하는 뮤텍스 경합입니다. Oracle 19c 이후 주요 동기화 메커니즘입니다.
  - **원인**: 매우 빈번한 Hard Parse, 집중적인 DDL 작업(오브젝트 생성/삭제), 권한 검사 폭주 등.
- **`row cache lock`**
  - Row Cache 엔트리의 내용을 수정할 때 필요한 락입니다.
  - **원인**: DDL 실행, 시퀀스 캐시 조정, 프로파일 변경 등.

### 실습: DDL 폭주로 인한 Row Cache 경합 재현

```sql
-- 주의: 테스트 환경에서만 실행하세요.
BEGIN
  FOR i IN 1..5000 LOOP
    EXECUTE IMMEDIATE 'CREATE TABLE t_dc_'||i||'(id NUMBER)';
    EXECUTE IMMEDIATE 'DROP TABLE t_dc_'||i;
  END LOOP;
END;
/

-- 발생한 대기 이벤트 확인
SELECT event, total_waits, time_waited_micro/1e6 sec
FROM   v$system_event
WHERE  event IN ('row cache mutex','row cache lock')
ORDER  BY sec DESC;
```

> **튜닝 방향**
> - 불필요하게 빈번한 **DDL 실행을 최소화**합니다. 임시 오브젝트는 재사용을 고려하세요.
> - **시퀀스의 CACHE 크기**를 적절히 늘려 Row Cache에 대한 갱신 빈도를 낮춥니다.
> - 불필요한 동의어나 복잡한 권한 구조를 정리하여 Row Cache 조회 부하를 줄입니다.
> - 근본적으로 **Hard Parse를 억제**(바인드 변수 사용)하면 Row Cache 조회 자체가 감소합니다.

---

## Library Cache 상세 분석

### 저장하는 정보

Library Cache는 실행 가능한 코드와 그 실행 계획을 저장하여 재사용성을 극대화합니다.

- **SQL 커서**: 파싱된 SQL 텍스트, 실행 계획, 실행 컨텍스트.
- **PL/SQL 오브젝트**: 컴파일된 패키지, 프로시저, 함수, 트리거의 코드와 메타데이터.
- **Java 클래스**: 데이터베이스 내 로드된 Java 소스 및 클래스.
- **DDL 오브젝트 정의**: 테이블, 뷰 등의 정의와 의존성 정보.

### Hard Parse vs Soft Parse

- **Hard Parse (하드 파싱)**:
  SQL이 최초로 실행되거나 Library Cache에서 찾을 수 없을 때 발생하는 전체 컴파일 과정입니다.
  1. **구문 해석(Syntax Check)**
  2. **의미 검증(Semantic Check) - 이때 Dictionary Cache 활발히 사용**
  3. **최적화(Optimization) - 실행 계획 생성**
  4. **로드(Load) - 생성된 커서를 Library Cache에 등록**
  이 과정은 CPU와 메모리, 래치/뮤텍스 자원을 상당히 소모합니다.

- **Soft Parse (소프트 파싱)**:
  동일한 SQL(정규화된 텍스트, 바인드 변수 위치, NLS 설정 등이 완전히 일치)이 Library Cache에 이미 존재할 때 발생합니다. 저장된 실행 계획과 컨텍스트를 그대로 재사용하므로 Hard Parse에 비해 비용이 극히 적습니다.

> **핵심 목표**: 애플리케이션 설계와 튜닝의 중요한 목표는 **Hard Parse 비율을 최소화**하는 것입니다.

### 상태 모니터링 쿼리

```sql
-- Library Cache 전체 통계 (Get, Pin, Reload, Invalidation)
SELECT namespace, gets, gethits, pins, pinhits, reloads, invalidations
FROM   v$librarycache;

-- 가장 많이 파싱된 SQL 확인
SELECT sql_id, plan_hash_value, executions, parse_calls,
       version_count, loaded_versions, invalidations
FROM   v$sqlarea
ORDER  BY parse_calls DESC FETCH FIRST 20 ROWS ONLY;

-- 특정 SQL의 Child Cursor가 분기된 원인 분석
SELECT sql_id, child_number, reason
FROM   v$sql_shared_cursor
WHERE  sql_id = :sql_id;
```

### Child Cursor 분기의 원인

동일한 SQL 텍스트라도 여러 개의 Child Cursor(자식 커서)가 생성될 수 있습니다. 이는 성능 최적화(Adaptive Cursor Sharing) 또는 환경 차이 때문일 수 있습니다. 주요 분기 원인:
- **바인드 변수 데이터 타입 또는 길이의 불일치**
- **NLS(국가 언어 지원) 설정 차이**
- **옵티마이저 관련 파라미터 또는 통계 변경**
- **Bind Peeking과 Adaptive Cursor Sharing(ACS)** 으로 인한 바인드 값 분포 기반의 최적 플랜 생성

> **문제점**: 과도한 `version_count`는 Shared Pool 메모리 낭비와 `cursor: mutex S/X` 경합을 유발할 수 있습니다.

### 관련 대기 이벤트

- **`library cache: lock`**: 오브젝트(커서, PL/SQL 유닛 등)의 정합성을 보장하기 위한 락. DDL이나 컴파일 작업 시 발생합니다.
- **`library cache: pin`**: 오브젝트가 실행 중이거나 컴파일 중일 때 메모리에서 사라지지 않도록 고정(Pin)하는 메커니즘. 동시 실행 시 경합 발생.
- **`cursor: mutex S/X`**: 커서 헤더에 대한 접근을 직렬화하는 뮤텍스. **하드 파싱 폭주**와 **Child Cursor 과다 생성** 시 가장 흔히 발생하는 이벤트입니다.
- **`library cache load lock`**: 새 커서를 Library Cache에 로드하는 과정에서의 동기화 락.

---

## 예제 시나리오 — 리터럴 SQL의 폐해 vs 바인드 변수의 효용

### 테스트 환경 준비

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

### **안티패턴: 리터럴 SQL 남발**

```sql
-- 서로 다른 상수 값을 가진 동일 형태의 SQL을 수천 번 실행
BEGIN
  FOR k IN 1..5000 LOOP
    EXECUTE IMMEDIATE 'SELECT /* lit */ COUNT(*) FROM t_sales WHERE cid='||MOD(k,1000);
  END LOOP;
END;
/

-- 결과 관측
SELECT sql_text, executions, parse_calls, version_count
FROM   v$sqlarea
WHERE  sql_text LIKE 'SELECT /* lit */ COUNT(*)%'
ORDER  BY parse_calls DESC;
```

**발생 현상**
- SQL 텍스트가 매번 달라져 정규화 키가 다르게 생성됩니다.
- 결과적으로 **매번 Hard Parse**가 발생합니다.
- **Library Cache와 Row Cache에 대한 극심한 경합**이 발생하며, `cursor: mutex S/X`, `library cache: lock` 이벤트가 급증합니다.
- Shared Pool이 수천 개의 유사 커서로 파편화되고, 메모리 부족(ORA-04031) 위험이 높아집니다.

### **베스트 프랙티스: 바인드 변수 사용**

```sql
VARIABLE b NUMBER
BEGIN
  FOR k IN 1..5000 LOOP
    :b := MOD(k,1000);
    EXECUTE IMMEDIATE 'SELECT /* bind */ COUNT(*) FROM t_sales WHERE cid=:1' USING :b;
  END LOOP;
END;
/

-- 결과 관측
SELECT sql_text, executions, parse_calls, version_count
FROM   v$sqlarea
WHERE  sql_text LIKE 'SELECT /* bind */ COUNT(*)%'
ORDER  BY parse_calls DESC;
```

**개선 효과**
- 동일한 SQL 텍스트(`cid=:1`)를 재사용하므로 **대부분 Soft Parse**로 처리됩니다.
- 하나의 부모 커서와 소수의 Child Cursor만 생성됩니다.
- Row Cache와 Library Cache에 대한 부하 및 경합이 현저히 감소합니다.

> **응급 조치**: 레거시 애플리케이션 변경이 어려울 경우, 세션 또는 시스템 레벨에서 `CURSOR_SHARING = FORCE` 파라미터를 일시적으로 적용하여 리터럴을 시스템 생성 바인드로 대체할 수 있습니다. 단, 이로 인한 실행 계획 변경 리스크를 인지해야 합니다.

---

## Bind Peeking과 Adaptive Cursor Sharing(ACS)

- **Bind Peeking**: SQL이 처음 Hard Parse될 때 전달된 바인드 변수의 실제 값을 "엿보아" 통계 정보와 결합하여 실행 계획을 결정합니다.
- **Adaptive Cursor Sharing(ACS)**: 이후 다양한 바인드 값으로 동일 SQL이 실행되면서, Oracle이 값의 분포(선택도)를 학습합니다. 서로 다른 선택도를 가진 값 범주에 대해 **별도의 최적화된 Child Cursor(실행 계획)** 를 생성합니다. 이는 `V$SQL_CS_SELECTIVITY`와 `V$SQL_CS_HISTOGRAM` 뷰에서 확인할 수 있습니다.

```sql
-- 특정 SQL이 왜 여러 Child Cursor를 가지게 되었는지 구체적인 원인 확인
SELECT child_number, reason
FROM   v$sql_shared_cursor
WHERE  sql_id = :sql_id;
```

> **튜닝 관점**
> - ACS는 바인드 변수 사용 시 선택도 차이로 인한 성능 문제를 완화하지만, 지나치게 많은 Child Cursor를 생성하면 Shared Pool 부담이 될 수 있습니다.
> - 컬럼 히스토그램 통계가 적절히 수집되어야 ACS가 올바르게 동작합니다.
> - 매우 민감한 쿼리의 경우, **SQL Plan Management(SPM)** 를 통해 안정적인 플랜을 고정하거나 **SQL 프로파일**을 적용하는 것을 고려할 수 있습니다.

---

## Invalidation — 캐시 무효화

Library Cache에 캐시된 객체가 더 이상 유효하지 않게 되어 무효화되는 상황입니다. 무효화된 객체는 다음 실행 시 Reload(재파싱 및 재로드)를 필요로 합니다.

**주요 원인:**
- **DDL 작업**: 테이블 컬럼 추가/삭제, 인덱스 생성/삭제 등.
- **객체 통계 갱신**: `DBMS_STATS.GATHER_TABLE_STATS` 실행.
- **권한 변경**: 객체에 대한 권한 부여/회수.
- **글로벌 파라미터 변경**: 옵티마이저 관련 파라미터 변경.

```sql
-- 라이브러리 캐시 무효화 횟수 모니터링
SELECT namespace, invalidations
FROM   v$librarycache
WHERE  namespace IN ('SQL AREA','TABLE/PROCEDURE','BODY','TRIGGER');
```

> **운영 영향**: Invalidation은 관련 SQL의 다음 실행을 지연시키고(`library cache: lock/pin` 대기), 일시적인 성능 저하를 유발할 수 있습니다. 따라서 핵심 업무 시간대의 불필요한 DDL이나 통계 수집을 지양해야 합니다.

---

## Library Cache/Row Cache 관련 대기 이벤트 진단표

| 대기 이벤트 | 의미 | 주요 원인 및 대응 방안 |
| :--- | :--- | :--- |
| **`cursor: mutex S/X`** | 커서 헤더 접근 직렬화 경합 | **원인**: 하드 파싱 폭주, 리터럴 SQL 남발. <br> **대응**: 바인드 변수 사용, 커서 재사용 유도. |
| **`library cache: lock`** | 오브젝트 정합성 유지 락 경합 | **원인**: 동시 DDL, 객체 컴파일, Invalidation. <br> **대응**: DDL 작업 시간대 분리, 불필요한 컴파일 최소화. |
| **`library cache: pin`** | 오브젝트 실행 중 변경 방지 Pin 경합 | **원인**: 동일 객체에 대한 매우 빈번한 실행/참조. <br> **대응**: 코드 설계 검토(과도한 동시 실행 완화). |
| **`library cache load lock`** | 새 커서 Library Cache 로드 락 | **원인**: 새로운 커서 로드 폭주(리터럴 SQL). <br> **대응**: 바인드 변수 사용, `CURSOR_SHARING` 고려. |
| **`row cache mutex`** | Row Cache 엔트리 접근 뮤텍스 경합 | **원인**: 하드 파싱 폭주, DDL 빈발, 권한 검사 집중. <br> **대응**: Hard Parse 감소, DDL 통합, 시퀀스 CACHE 크기 증가. |
| **`row cache lock`** | Row Cache 엔트리 수정 락 | **원인**: DDL 실행, 시퀀스 NEXTVAL 호출, 프로파일 변경. <br> **대응**: DDL 작업 최적화, 시퀀스 캐시 적정화. |

---

## DBMS_SHARED_POOL.KEEP — 핵심 객체 고정

자주 사용되고 크기가 큰 PL/SQL 패키지, 프로시저 또는 핵심 SQL 커서가 Shared Pool에서 쫓겨나지 않도록 메모리에 고정(Pin)할 수 있습니다. 이는 라이브러리 캐시 로드 경합과 재파싱 오버헤드를 방지합니다.

```sql
-- PL/SQL 객체(패키지)를 Shared Pool에 고정
EXEC DBMS_SHARED_POOL.KEEP('SCOTT.PKG_BILLING', 'P');  -- 'P'는 PL/SQL 객체 타입

-- 고정할 SQL 커서 확인 (ADDRESS와 HASH_VALUE 필요)
SELECT address, hash_value, sql_text
FROM   v$sql
WHERE  sql_text LIKE 'SELECT /* critical_query */%';

-- 확인된 ADDRESS와 HASH_VALUE를 사용하여 고정 (고급 방법, 주의 필요)
-- EXEC DBMS_SHARED_POOL.KEEP('00000008A3C01F80, 1234567890', 'C');
```

> **주의사항**: 과도한 KEEP은 Shared Pool의 유연한 메모리 관리를 방해하고, 실제로 자주 사용되지 않는 객체가 메모리를 점유하게 할 수 있습니다. **정말 핵심적이고 자주 호출되는 대형 객체에만 선택적으로 적용**해야 합니다.

---

## 세션 커서 캐싱

세션 커서 캐시는 각 세션 내에서 반복적으로 파싱되는 SQL에 대한 추가적인 최적화 계층입니다. 이는 라이브러리 캐시 조회보다도 빠른 재사용을 가능하게 합니다.

```sql
-- 세션 커서 캐시 크기 파라미터 확인
SHOW PARAMETER session_cached_cursors;

-- 파싱 관련 시스템 통계 확인
SELECT name, value
FROM   v$sysstat
WHERE  name IN ('parse count (total)','parse count (hard)','session cursor cache hits')
ORDER  BY name;
```

- **`session_cached_cursors`**: 각 세션이 캐시할 수 있는 커서 수를 결정합니다. 적절히 증가시키면 동일 세션 내의 반복적인 Soft Parse 성능이 향상됩니다.
- **핵심 지표**: `parse count (hard) / parse count (total)` 비율을 모니터링하여 Hard Parse 비율을 낮추는 것이 목표입니다. `session cursor cache hits`가 증가하는지 확인합니다.

---

## ORA-04031 에러 대응을 위한 점검 사항

Shared Pool 메모리 부족 오류(ORA-04031)를 예방하고 대응하기 위한 실무 지침입니다.

1.  **SQL 표준화**: 리터럴 SQL을 바인드 변수 기반 SQL로 전환합니다. `CURSOR_SHARING=FORCE`는 임시 방편으로 고려할 수 있습니다.
2.  **Child Cursor 관리**: 바인드 변수의 데이터 타입과 길이를 일관되게 유지하고, 불필요한 NLS 또는 환경 설정 차이를 제거합니다. Adaptive Cursor Sharing으로 인한 과도한 분기가 있는지 점검합니다.
3.  **KEEP 전략 검토**: `DBMS_SHARED_POOL.KEEP`으로 고정된 객체 목록을 정기 검토하여 불필요한 고정을 해제합니다.
4.  **DDL/컴파일 전략**: 빈번한 애플리케이션 객체 재컴파일을 지양하고, 주요 DDL 작업은 사용량이 낮은 시간대에 배치 처리합니다.
5.  **Shared Pool 모니터링**: `V$SGASTAT`, `V$SHARED_POOL_RESERVED`를 통해 메모리 사용량과 예약 영역의 효율성을 정기적으로 확인합니다.
6.  **성능 진단 활용**: AWR/ASH 리포트에서 `cursor: mutex`, `library cache:`, `row cache` 관련 대기 이벤트가 상위를 차지하는 원인 SQL을 식별하고 최적화합니다.
7.  **워크로드 분산**: 많은 수의 세션이 동시에 하드 파싱을 유발하는 작업을 수행하지 않도록 애플리케이션 스케줄링을 조정합니다.

---

## 종합 실습 — “나쁜 패턴 진단 → 개선 → 효과 검증”

이 실습을 통해 리터럴 SQL의 문제점을 직접 관측하고, 바인드 변수 전환 후의 개선 효과를 확인할 수 있습니다.

### 1단계: 나쁜 패턴(리터럴 SQL) 실행 및 베이스라인 측정

먼저, 리터럴 SQL을 폭발적으로 실행하는 스크립트를 수행하기 전후로 시스템 지표를 측정합니다.

```sql
-- (1) 실행 전 '나쁜 패턴' 실행 전 스냅샷 저장 (가상)
-- SELECT event, total_waits... FROM v$system_event WHERE ... (위 진단쿼리 참고)
-- SELECT name, value FROM v$sysstat WHERE ... (위 진단쿼리 참고)

-- (2) 나쁜 패턴 실행 (리터럴 루프, 필요시 DDL 루프도 추가)
BEGIN
  FOR k IN 1..5000 LOOP
    EXECUTE IMMEDIATE 'SELECT /* BAD_PATTERN */ COUNT(*) FROM t_sales WHERE cid='||MOD(k,1000);
  END LOOP;
END;
/

-- (3) 실행 후 스냅샷 저장 및 비교
```

### 2단계: 개선 조치 적용

- 모든 SQL을 바인드 변수 형태로 변환합니다.
- `session_cached_cursors` 파라미터 값을 적절히 상향 조정합니다(예: 50).
- 불필요한 `KEEP`은 제거하고, 진짜 필요한 핵심 객체만 고정합니다.
- 대량 DDL 작업이 있다면, 별도의 배치 창으로 이동시킵니다.

### 3단계: 개선 후 측정 및 비교

개선 조치 후 동일한 부하 테스트를 수행하고 지표를 다시 측정합니다.

**기대되는 개선 효과:**
- **`parse count (hard)`** 통계 값의 현저한 감소
- **`cursor: mutex S/X`**, **`row cache mutex`** 등의 대기 시간 감소
- **`V$LIBRARYCACHE`** 뷰의 `RELOADS` 및 `INVALIDATIONS` 감소
- **`V$ROWCACHE`** 뷰의 `GETMISSES` 감소

---

## 결론

Oracle Shared Pool은 데이터베이스 성능의 심장부와 같은 역할을 합니다. **Dictionary Cache(Row Cache)** 는 SQL 실행을 위한 메타데이터 검색을, **Library Cache** 는 SQL과 PL/SQL 코드의 실행 계획과 컴파일 결과를 재사용할 수 있게 하여 시스템의 전반적인 효율성을 결정짓습니다.

성능 문제의 상당수는 Shared Pool의 비효율적 사용에서 비롯됩니다. **리터럴 SQL의 폭주**는 Hard Parse를 유발하여 `cursor: mutex` 경합을 만들고, **빈번한 DDL**은 `row cache lock`과 무효화(Invalidation)를 유발하며, **과도한 Child Cursor 분기**는 Shared Pool 메모리를 고갈시킬 수 있습니다.

따라서 안정적인 성능을 보장하기 위한 핵심 원칙은 다음과 같습니다.
1.  **최우선으로 바인드 변수를 사용**하여 커서 재사용률을 극대화하세요.
2.  **Child Cursor가 불필요하게 분기되지 않도록** 환경 설정과 바인드 변수 사용을 일관되게 유지하세요.
3.  **DDL, 통계 수집, 대형 컴파일 작업은 시스템 부하가 낮은 시간대로 계획**하여 캐시 무효화의 영향을 최소화하세요.
4.  **체계적인 모니터링**을 통해 `V$LIBRARYCACHE`, `V$ROWCACHE`, 관련 대기 이벤트를 지속적으로 관찰하고, 문제의 조기 징후를 포착하세요.

Shared Pool을 효율적으로 관리하는 것은 단순한 메모리 설정을 넘어, **애플리케이션 설계 패턴에서부터 운영 관행에 이르는 종합적인 접근**이 필요합니다. 본 가이드가 이러한 종합적인 이해와 실무 적용에 도움이 되기를 바랍니다.