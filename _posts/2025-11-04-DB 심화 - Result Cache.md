---
layout: post
title: DB 심화 - Result Cache
date: 2025-11-04 21:25:23 +0900
category: DB 심화
---
# Oracle **Result Cache** 완전 가이드

— **SQL Result Cache**와 **PL/SQL Function Result Cache**, 메모리/무효화/제약/튜닝/진단/사례

> **핵심 요약**
> - **Result Cache(서버 결과 캐시)**는 “**동일한 입력 → 동일한 결과**”인 읽기 중심 워크로드에서, **쿼리 결과** 또는 **함수 반환값**을 **SGA의 전용 메모리 영역**에 저장해 **재사용**하도록 하는 기능이다.
> - 두 축:
>   1) **SQL Result Cache** — `SELECT` 결과 집합을 캐시(힌트 `/*+ RESULT_CACHE */` 또는 파라미터).
>   2) **PL/SQL Function Result Cache** — 함수 호출(인자/세션컨텍스트)에 따른 **반환값** 캐시(`RESULT_CACHE` 프라그마).
> - **자동 무효화**: 결과를 만든 **테이블/뷰/시노님/시퀀스 등 의존 객체**가 변경되면 **캐시 무효화**.
> - **적합한 곳**: 변경 빈도 낮고, 동일 쿼리/호출이 **반복**되며 계산 비용이 큰 경우(집계/참조 테이블, 변환 함수 등).

---

## 준비 체크: 파라미터·뷰·패키지

### 핵심 파라미터

```sql
-- 현재 설정 확인
SHOW PARAMETER RESULT_CACHE

-- 주요 항목
-- RESULT_CACHE_MAX_SIZE            : SGA 내 Result Cache 사이즈(바이트). 0이면 기능 사실상 비활성.
-- RESULT_CACHE_MAX_RESULT         : 하나의 결과가 캐시에서 차지할 수 있는 최대 비율(%) 예: 5
-- RESULT_CACHE_MODE               : MANUAL | FORCE
-- RESULT_CACHE_REMOTE_EXPIRATION  : 원격(데이터베이스 링크) 결과 TTL(분). 0이면 원격 결과 캐시 안 함.
```
- **MODE**
  - `MANUAL` : 힌트가 있는 경우만 캐시
  - `FORCE`  : 가능하면 자동 캐시(제약/비적합 상황이면 캐시 제외). 운영에선 **MANUAL 권장**(명시적 관리).

### 관리·통계 뷰/패키지

```sql
-- 상태/통계
SELECT * FROM V$RESULT_CACHE_STATISTICS;  -- 전역 통계(히트율, 메모리, 해시버킷 등)
SELECT * FROM V$RESULT_CACHE_MEMORY;      -- 메모리 세부
SELECT * FROM V$RESULT_CACHE_OBJECTS;     -- 캐시 객체(결과/Dependency/기타)
SELECT * FROM V$RESULT_CACHE_DEPENDENCY;  -- 결과 ↔ 의존 객체 매핑

-- 패키지
-- - DBMS_RESULT_CACHE.FLUSH: 전체 플러시
-- - DBMS_RESULT_CACHE.BYPASS(TRUE/FALSE): 일시 우회
-- - DBMS_RESULT_CACHE.MEMORY_REPORT: 가독성 있는 메모리 리포트
BEGIN
  DBMS_RESULT_CACHE.MEMORY_REPORT;  -- 리포트 출력(서버 출력)
END;
/
```

---

## SQL Result Cache

### 개념

- 특정 `SELECT`의 **결과 집합**을 캐시에 저장 → **동일 SQL(바인드 포함), 동일 세션 컨텍스트**로 재실행 시 **결과를 메모리에서 즉시 반환**.
- **무효화**: 결과에 **의존하는 객체(테이블/뷰 등)** 에 DML/DDL이 발생하면 자동 무효화.
- **키 구성 요소(간단화)**: **정규화된 SQL 텍스트**, **바인드 값**, **NLS/환경 일부**, **현재 스키마**, **ROLE/권한 영향** 등.

### 힌트 사용법

```sql
-- 명시적 캐시
SELECT /*+ RESULT_CACHE */ c.cust_id, SUM(o.amount) AS amt
FROM   orders o JOIN customers c ON c.cust_id = o.cust_id
WHERE  c.region = :r
GROUP  BY c.cust_id;

-- 캐시 금지
SELECT /*+ NO_RESULT_CACHE */ ...
```
> `RESULT_CACHE_MODE=FORCE` 인 환경에서도 **NO_RESULT_CACHE** 로 제외 가능.

### 캐시 적합/부적합 판별(주요 규칙·예시)

- **적합**
  - 변경이 드문 **참조성 테이블**(코드/마스터), **집계/랭킹** 등 반복된 조회
  - **바인드 패턴**이 제한적(예: `:region` 이 소수 값)
- **부적합(자동 제외/효과 없음)**
  - **비결정적 함수**: `DBMS_RANDOM`, `SYS_GUID`, 일부 `SYS_CONTEXT`, `SYSTIMESTAMP` 등
  - `CURRENT_DATE`, `SYSDATE` 등 **시간 의존**(상황 따라 제외/TTL 필요)
  - **FOR UPDATE**, **병렬 쿼리의 일부 단계**, **Long/LOB 스트리밍 특성**, **세션 컨텍스트 민감성 매우 큼**
  - 결과셋이 **아주 큼** → `RESULT_CACHE_MAX_RESULT` 제한에 걸려 미캐시

### 예제: “리포트 상단 배너용” Top-N 집계

> 시나리오: 홈 대시보드가 **같은 Top-N**을 분당 수십~수백회 반복 조회. 소스 테이블은 10분에 1회 배치 반영.

```sql
-- 가정: orders, customers 테이블 존재 (읽기 잦고, 변경은 배치 위주)
SELECT /*+ RESULT_CACHE */
       c.region, c.segment,
       SUM(o.amount) AS total_amt
FROM   orders o
JOIN   customers c ON c.cust_id = o.cust_id
WHERE  o.order_dt >= TRUNC(SYSDATE) - 7  -- 최근 7일
GROUP  BY c.region, c.segment
ORDER  BY total_amt DESC
FETCH FIRST 10 ROWS ONLY;
```

**효과 관찰**
```sql
-- 캐시 적중률
SELECT * FROM V$RESULT_CACHE_STATISTICS;

-- 어떤 결과가 올라왔는지
SELECT ID, TYPE, NAME, STATUS, CREATION_TIMESTAMP, SERIAL#
FROM   V$RESULT_CACHE_OBJECTS
ORDER  BY CREATION_TIMESTAMP DESC;

-- 의존성 확인(어떤 테이블에 의존?)
SELECT rco.ID, rco.NAME, rcd.OBJECT_NAME, rcd.OBJECT_TYPE
FROM   V$RESULT_CACHE_OBJECTS rco
JOIN   V$RESULT_CACHE_DEPENDENCY rcd
  ON   rco.ID = rcd.RESULT_ID
WHERE  rco.TYPE = 'Result';
```

> **무효화 규칙**: 예를 들어 `orders` 에서 **INSERT/UPDATE/DELETE** 가 발생하면 **해당 결과가 INVALID로 표기되고 재사용되지 않음**(개별 결과 단위 무효화).

### 결과 TTL

```sql
ALTER SYSTEM SET RESULT_CACHE_REMOTE_EXPIRATION = 10; -- 10분 TTL
-- 원격 테이블이 참조되는 결과는 TTL 경과 후 자동 만료(무효화 이벤트 없더라도).
```

### 운영 팁

- `RESULT_CACHE_MODE=MANUAL` + **핵심 질의에만 힌트**로 **정밀 제어** 권장.
- `RESULT_CACHE_MAX_SIZE` 는 너무 작으면 **교체/할당 실패**↑, 너무 크면 **SGA 압박**. AWR/메모리 리포트로 적정치 산정.
- **바인드 상수화**(리터럴) 남발 시 동일 SQL로 인식되지 않아 **캐시 파편화**. **바인드 변수** 사용 권장.

---

## PL/SQL **Function Result Cache**

### 개념

- PL/SQL 함수에 `RESULT_CACHE` 프라그마를 붙여, **함수 인자**(+ 일부 세션 컨텍스트)를 키로 **반환값**을 캐시.
- **무효화**: 함수가 참조하는 **데이터베이스 객체** 변경 시 관련 결과 **자동 무효화**.
- 과거 문법의 `RELIES_ON (obj ...)` 는 버전에 따라 **권장되지 않음**(자동 의존성 추적을 사용).

### 함수 선언/정의

```sql
CREATE OR REPLACE FUNCTION fx_currency_rate(
    p_from IN VARCHAR2,
    p_to   IN VARCHAR2
) RETURN NUMBER
RESULT_CACHE
IS
    v_rate NUMBER;
BEGIN
    -- 계산 비용이 꽤 큰(또는 원격/HTTP 호출 래핑) 경우라고 가정
    SELECT rate
    INTO   v_rate
    FROM   currency_rates
    WHERE  from_ccy = p_from
    AND    to_ccy   = p_to;

    RETURN v_rate;
END;
/
```

**호출 예**
```sql
-- 같은 입력이면 SGA에서 곧바로 복원(무효화 전까지)
SELECT fx_currency_rate('USD','KRW') FROM dual;
SELECT fx_currency_rate('USD','KRW') FROM dual;  -- 캐시 적중 기대
```

**관찰**
```sql
SELECT * FROM V$RESULT_CACHE_OBJECTS WHERE TYPE='Dependency'; -- fx_currency_rate가 의존하는 오브젝트
SELECT * FROM V$RESULT_CACHE_OBJECTS WHERE TYPE='Result' AND NAME LIKE 'FX_CURRENCY_RATE%';
```

### 세션 컨텍스트 고려

- 함수 결과 키는 **인자** 외에도 일부 **세션 상태**에 민감할 수 있다(예: NLS 설정에 따라 숫자 포맷/라운딩 로직이 달라지는 경우).
- **순수 결정성**을 유지해야 캐시 효율이 높다.
  - 부적합: 내부에서 `SYSDATE`, `SYSTIMESTAMP`, `DBMS_RANDOM` 사용, **변화하는 외부 상태** 의존 등.

### 고비용 변환/규칙 함수 예

```sql
CREATE OR REPLACE FUNCTION fx_vat_rate(p_region IN VARCHAR2)
RETURN NUMBER
RESULT_CACHE
IS
  v NUMBER;
BEGIN
  SELECT vat_rate INTO v FROM region_vat WHERE region = p_region;
  RETURN v;
END;
/

-- 대량 처리 SQL에서 함수 호출 최소화 효과(캐시 적중)
SELECT o.order_id,
       o.amount,
       o.amount * fx_vat_rate(c.region) AS amount_vat
FROM   orders o
JOIN   customers c ON c.cust_id = o.cust_id
WHERE  o.order_dt >= TRUNC(SYSDATE)-30;
```
> **주의**: 대량 SQL에서 **함수 호출 자체 비용**(컨텍스트 스위칭)이 크다면, **스칼라 서브쿼리 캐싱** or **집합 연산으로 재작성**이 더 우수할 수 있다. Result Cache는 **함수 자체의 반복 호출**을 줄이지는 못하고 **결과 재사용**만 해주기 때문.

---

## 메모리·무효화·삽입/교체 정책

### 메모리 구조(요약)

- Result Cache는 **SGA 내 전용 영역**(보통 Shared Pool 한 켠)에서 **해시 버킷**으로 관리.
- **오브젝트 유형**
  - `Result`  : SQL 결과셋/함수 반환값
  - `Dependency` : Result가 참조하는 DB 오브젝트(테이블/뷰 등)
  - `Other` : 기타 메타
- **교체 정책**: 공간 부족 시 **LRU 유사 정책**으로 추출(세부는 버전별 상이).

### 무효화 트리거

- 의존 오브젝트에 **DML/DDL** 발생 → 관련 **Result** 를 **INVALID** 로 표기 → 다음 접근 시 **재계산**.
- **원격 오브젝트**는 `RESULT_CACHE_REMOTE_EXPIRATION` (TTL) 정책.
- **권한/오브젝트 접근 가능성**이 달라지면 해당 결과 재사용 불가.

### 크기 제한

- `RESULT_CACHE_MAX_RESULT` (%) : 단일 결과가 전체 캐시에서 차지 가능한 상한. 너무 큰 결과는 삽입 자체가 **거부**.
- 실습/운영에서 **큰 결과셋**(수만~수십만 행)을 캐시시키는 것은 비현실적. **Top-N·요약·lookup** 중심으로.

---

## 제약/금지/주의 리스트

- **비결정성**: `DBMS_RANDOM`, `SYS_GUID`, 일부 `SYS_CONTEXT`, `CURRENT_SCHEMA` 변동 등 **세션/시간 의존** → 캐시 제외/무용.
- **시간 함수**: `SYSDATE`/`SYSTIMESTAMP` 포함 질의는 일반적으로 **캐시 부적합**(TTL로 보완 가능하나 주의).
- **FOR UPDATE**, DML/DDL 문, 일부 LOB 스트림/커서 변형 → 대상 아님.
- **결과 크기 과대**: `MAX_RESULT` 한계 초과 → 삽입 거부.
- **보안 컨텍스트**: VPD/ROW LEVEL SECURITY, Fine-Grained Access Control 등 **보안 정책**이 있으면 **세션별 결과 달라짐** → 캐시 효과 크게 감소/제외.

---

## RAC에서의 Result Cache

- **인스턴스 로컬 캐시**: 각 인스턴스의 SGA에 **별도 보유**(글로벌 공유 아님).
- **무효화**: 의존 오브젝트 변경 시 **모든 인스턴스에 무효화 브로드캐스트** → 각 인스턴스의 관련 결과 **INVALID**.
- **튜닝 포인트**
  - **읽기 전용/리포팅 서비스**를 특정 인스턴스에 라우팅하면 그 인스턴스의 Result Cache **히트율**을 높일 수 있음.
  - 인스턴스 간 **동일 질의**가 계속 반복되더라도 **각자 따로 저장**한다는 점 고려(메모리 계획).

---

## 실전 튜닝 절차

### 후보 발굴

```sql
-- 반복 호출이 많은 SQL Top-N
SELECT sql_id, parsing_schema_name, executions, buffer_gets, rows_processed
FROM   v$sql
WHERE  command_type = 3 -- SELECT
ORDER  BY executions DESC FETCH FIRST 50 ROWS ONLY;

-- 변경 빈도 낮은 참조 테이블 목록(예: 코드/마스터)
SELECT owner, table_name, last_analyzed, num_rows
FROM   dba_tables
WHERE  owner IN ('APP','DIM')
AND    num_rows < 100000
ORDER  BY num_rows;
```

### 실험

1) **기존 응답시간/CPU** 기준선 채취(AWR/ASH/SQL Monitor).
2) 대상 SQL에 `/*+ RESULT_CACHE */` 부여.
3) 재실행 후 **V$RESULT_CACHE_OBJECTS**/STATISTICS/메모리 리포트 점검.
4) **히트율/응답시간/리소스** 개선 확인.
5) DML 빈도/배치 타이밍에 따른 **무효화 영향** 모니터링.

### 실패/비효과 원인과 대응

| 현상 | 원인 | 대응 |
|---|---|---|
| 캐시가 안 생김 | 비결정성/제약, 결과 과대 | 쿼리/함수 재작성, Top-N/집계 요약, 파라미터 조정 |
| 히트율 낮음 | 바인드 값 다양/세션 컨텍스트 차이 큼 | 바인드 정규화, 호출 패턴 단순화, 리포팅 서비스 분리 |
| 잦은 무효화 | 대상 테이블 변경 잦음 | 더 작은 집합/스냅샷 테이블로 분리, 머티리얼라이즈드 뷰 고려 |
| 메모리 압박 | MAX_SIZE/RESULT 한계 | 사이즈 상향, 후보 축소, 불필요 캐시 제거(FLUSH) |

---

## 고급 주제

### TTL 기반 제어(원격/시간 민감)

- 원격 테이블이 포함된 SQL은 `RESULT_CACHE_REMOTE_EXPIRATION` 값(분)만큼 **유효**.
- **시간 의존** 로직은 Result Cache보다 **머티리얼라이즈드 뷰 REFRESH** 또는 **애플리케이션 레벨 캐시**가 더 적합한 경우가 많다.

### 머티리얼라이즈드 뷰/Server-Side Cache 비교

- **Result Cache**: **쿼리 결과/함수 반환**을 **SGA**에 보관, **메모리 기반**(DML 시 자동 무효화). 즉시성/간단성 장점.
- **MView**: **디스크 기반 사본** 저장, **스케줄/온디맨드 리프레시**. 변경이 빈번해도 **비동기 갱신**으로 조회 분리 가능.
- **둘의 목적 다름**: **짧은 재사용 창구** vs **물리적 사본/오프로드**.

### Client Result Cache(참고)

- OCI/ODP.NET 등 **클라이언트 프로세스 내** 캐시(네트워크 왕복도 절감).
- 서버 Result Cache와는 별도. 애플리케이션 아키텍처 관점에서 **양쪽 계층의 캐시 전략**을 **중복/경합없이** 설계.

---

## 종합 예제 — “대시보드 가속”

### 스키마

```sql
CREATE TABLE dim_region (
  region   VARCHAR2(8) PRIMARY KEY,
  vat_rate NUMBER(5,2)
);

CREATE TABLE fct_sales (
  sale_id  NUMBER PRIMARY KEY,
  region   VARCHAR2(8) NOT NULL,
  amount   NUMBER(12,2) NOT NULL,
  sale_dt  DATE NOT NULL
);

CREATE INDEX ix_fct_sales_region_dt ON fct_sales(region, sale_dt);

BEGIN
  DBMS_STATS.GATHER_TABLE_STATS(USER,'DIM_REGION',cascade=>TRUE);
  DBMS_STATS.GATHER_TABLE_STATS(USER,'FCT_SALES',cascade=>TRUE);
END;
/
```

### PL/SQL 함수 결과 캐시

```sql
CREATE OR REPLACE FUNCTION fx_vat(p_region IN VARCHAR2)
RETURN NUMBER
RESULT_CACHE
IS
  v NUMBER;
BEGIN
  SELECT vat_rate INTO v
  FROM   dim_region
  WHERE  region = p_region;
  RETURN v;
END;
/
```

### SQL Result Cache 적용 쿼리

```sql
-- 대시보드 상단: 최근 7일 지역별 매출+VAT 합계(반복 조회)
SELECT /*+ RESULT_CACHE */
       region,
       SUM(amount)                                  AS amt,
       SUM(amount * fx_vat(region))                 AS amt_vat
FROM   fct_sales
WHERE  sale_dt >= TRUNC(SYSDATE) - 7
GROUP  BY region
ORDER  BY amt DESC;
```

**운영 시나리오**
- **조회 빈번**, **기초 테이블 갱신은 배치 5분/10분 간격** → 그 사이 **Result Cache 적중**으로 **응답시간 단축**.
- 배치가 반영되면 **무효화 → 첫 조회가 재계산** 후 다시 캐시.

### 관찰/리포트

```sql
SELECT name, value
FROM   v$result_cache_statistics;

SELECT id, type, name, status, creation_timestamp
FROM   v$result_cache_objects
WHERE  type = 'Result'
ORDER  BY creation_timestamp DESC;

BEGIN
  DBMS_RESULT_CACHE.MEMORY_REPORT;
END;
/
```

---

## 운영 플레이북(체크리스트)

1) **대상 선정**: “변경 드문+반복 조회+계산 비싼” 질의·함수.
2) **수동 모드**: `RESULT_CACHE_MODE=MANUAL` 유지, 후보 SQL/함수에 **명시 적용**.
3) **메모리 한계치**: `RESULT_CACHE_MAX_SIZE`, `MAX_RESULT` 로 **과대 결과 삽입 방지**.
4) **무효화 영향**: 배치 스케줄과 조회 패턴을 맞춰 **효율** 극대화.
5) **진단 루틴**: 뷰/패키지로 **히트율·오브젝트·의존성** 상시 점검.
6) **RAC**: 리포팅 트래픽은 **특정 인스턴스 라우팅**으로 **로컬 히트율** 제고.
7) **제약 회피**: 비결정성/시간 의존/보안 컨텍스트 민감 SQL은 **다른 캐싱/물리화** 고려.
8) **전/후 측정**: AWR/ASH/SQL Monitor로 **응답시간·CPU·버퍼/물리 읽기**의 **객관적 개선** 확인.

---

## 자주 하는 질문(FAQ)

- **Q. 왜 힌트를 줬는데도 캐시가 안 생기나요?**
  **A.** 비결정적 요소가 있거나 결과가 너무 크거나, 의존 오브젝트가 캐시 부적합일 수 있습니다. `V$RESULT_CACHE_OBJECTS` 를 확인하고, 쿼리/함수를 **순수 결정적**으로 재작성하세요.

- **Q. 캐시가 자주 무효화됩니다.**
  **A.** 대상 테이블 변경이 잦은 구조입니다. **스냅샷 테이블/MView** 로 조회 대상을 분리하거나, 요약 테이블을 만들어 **변경 경로**와 **조회 경로**를 분리하세요.

- **Q. 바인드 값이 너무 다양해 히트율이 낮습니다.**
  **A.** **Top-N/범주화**로 결과 유형을 줄이거나, **애플리케이션 레벨 캐시**(예: 멱등 키)와 병행하세요.

- **Q. RAC에서 글로벌 공유는 안 되나요?**
  **A.** 서버 Result Cache는 **인스턴스 로컬**입니다. 서비스 라우팅으로 **특정 인스턴스**에 리포팅을 집중하면 효과가 큽니다.

---

## 빠른 진단 스니펫 모음

```sql
-- 전체 통계
SELECT name, value FROM v$result_cache_statistics;

-- 최근 생성된 결과
SELECT id, type, name, bytes, status, creation_timestamp
FROM   v$result_cache_objects
WHERE  type='Result'
ORDER  BY creation_timestamp DESC;

-- 결과 ↔ 의존 오브젝트
SELECT rco.id, rco.name, rcd.object_name, rcd.object_type
FROM   v$result_cache_objects rco
JOIN   v$result_cache_dependency rcd ON rco.id = rcd.result_id
WHERE  rco.type = 'Result';

-- 캐시 플러시(주의: 운영 주의)
BEGIN
  DBMS_RESULT_CACHE.FLUSH;
END;
/

-- 일시 우회(배포/점검 중)
BEGIN
  DBMS_RESULT_CACHE.BYPASS(TRUE);
  -- ... 테스트 ...
  DBMS_RESULT_CACHE.BYPASS(FALSE);
END;
/
```

---

## 결론

- **Result Cache**는 “**같은 것을 또 계산하지 말자**”는 원칙을 **DB 엔진 차원**에서 실현하는 경량 가속 장치다.
- 성공 조건은 **결정성**, **반복성**, **변경 드묾**.
- **SQL Result Cache**와 **PL/SQL Function Result Cache**를 **명시적/선별적**으로 적용하고,
  **메모리/무효화/RAC 특성**을 이해해 **읽기 워크로드**를 **손쉽게 가속**하라.
- 모든 변경은 **전/후 지표**로 검증하고, 캐시가 부적합한 경우엔 **머티리얼라이즈드 뷰/애플리케이션 캐시**로 전략을 전환한다.
