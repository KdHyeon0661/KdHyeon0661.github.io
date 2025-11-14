---
layout: post
title: DB 심화 - DETERMINISTIC` 함수 사용 시 주의사항
date: 2025-11-05 22:25:23 +0900
category: DB 심화
---
# Oracle **`DETERMINISTIC` 함수 사용 시 주의사항**

> **핵심 요약**
> - `DETERMINISTIC`는 “**동일 입력 ⇒ 항상 동일 출력**”을 **개발자가 보증**한다는 *선언*이다.
> - 이 선언은 **캐싱을 자동으로 켜주지 않는다**. (캐시는 `RESULT_CACHE`가 담당)
> - 그러나 `DETERMINISTIC`는 **함수기반 인덱스(FBI)**, **가상 컬럼**, **쿼리 재작성**에서 **필수 전제**라 **거짓으로 붙이면 인덱스/결과가 틀려질 수 있다**.
> - 다음에 단 하나라도 해당되면 **절대 `DETERMINISTIC` 붙이면 안 된다**: `SYSDATE/SYSTIMESTAMP`, `DBMS_RANDOM`, `SYS_GUID`, **SEQUENCE**, **세션/NLS/TimeZone/ROLE** 의존, **패키지 전역변수** 의존, **테이블 조회**(데이터가 바뀔 수 있음) 등.
> - “같은 입력에 대해 정말 변하지 않음”을 **증명**할 수 없으면 `DETERMINISTIC` 대신 `RESULT_CACHE`·스칼라 서브쿼리 캐시·조인/집합 재작성 등 **안전한 대안**을 쓰는 게 정석.

---

## `DETERMINISTIC`의 정확한 의미

```sql
CREATE OR REPLACE FUNCTION fx_norm(p_txt IN VARCHAR2)
RETURN VARCHAR2
DETERMINISTIC
IS
BEGIN
  RETURN TRIM(LOWER(p_txt));
END;
/
```

- 이 함수는 **입력이 문자열 하나뿐**이고, 내부에서 **외부 상태(시간/세션/테이블)** 를 참조하지 않으므로 “**진짜** 결정적”.
- **오해 주의**: `DETERMINISTIC`은 **엔진이 캐시한다**는 뜻이 아니다.
  - 캐시를 원하면 `RESULT_CACHE`(PL/SQL) 또는 **SQL Result Cache**(힌트) 사용.

---

## 왜 위험한가: “거짓 결정성”이 **틀린 결과**를 만든다

### 나쁜 예 1 — 시간 의존

```sql
CREATE OR REPLACE FUNCTION fx_today()
RETURN DATE
DETERMINISTIC
IS
BEGIN
  RETURN TRUNC(SYSDATE);  -- 시간에 따라 바뀜!
END;
/
```
- **문제**: 동일 입력(없음)인데 **출력이 날마다 바뀜**.
- FBI/가상컬럼에 쓰면 **인덱스 키가 데이터 수정 없이 달라지는 모순** → 잘못된 액세스/결과.

### 나쁜 예 2 — 세션/NLS 의존

```sql
CREATE OR REPLACE FUNCTION fx_title(p_txt VARCHAR2)
RETURN VARCHAR2
DETERMINISTIC
IS
BEGIN
  -- NLS 설정에 따라 UPPER/LOWER 결과 달라질 수 있음 (언어별 대소문 규칙)
  RETURN INITCAP(p_txt);
END;
/
```
- 세션마다 `NLS_SORT`, `NLS_COMP` 등 **문화권 설정**이 다르면 **같은 입력**도 **출력이 달라질 수 있음**.

### 나쁜 예 3 — 테이블/패키지 상태 의존

```sql
CREATE OR REPLACE FUNCTION fx_rate(p_ccy VARCHAR2)
RETURN NUMBER
DETERMINISTIC
IS
  v NUMBER;
BEGIN
  SELECT rate INTO v FROM rates WHERE ccy=p_ccy; -- 데이터가 바뀌면?
  RETURN v;
END;
/
```
- **데이터가 바뀔 수 있는** 외부 테이블에 의존 → 결정성이 **보장되지 않음**.
- FBI/가상컬럼/재작성에서 **틀린 인덱스**·**스테일 결과**를 양산.

> **원칙**: `DETERMINISTIC`은 **수학적 순수함**(Pure function) 수준일 때만 붙인다.
> 외부 세계(시간·세션·테이블·패키지 상태·IO)에 *조금이라도* 의존하면 **금지**.

---

## `DETERMINISTIC` vs `RESULT_CACHE` (둘의 차이)

| 항목 | `DETERMINISTIC` | `RESULT_CACHE` (PL/SQL) |
|---|---|---|
| 의미 | “같은 입력 → 같은 출력” **개발자 보증** | **SGA**에 결과를 캐시하여 **재사용** |
| 캐싱 | ❌ 자동 캐시 없음 | ✅ 캐시 및 **의존 객체 변경 시 무효화**(자동) X (※ PL/SQL 함수 자체의 의존성은 자동 추적 X, 보통 함수 내부 테이블 참조 시 RC가 **안전하지 않을 수 있어** 지양) |
| FBI/가상컬럼 | ✅ 필요(요건) | 무관 |
| 위험 | 거짓 선언 시 **틀린 결과** | 캐시 일관성/무효화 정책 이해 필요 |
| 권장 | **진짜 순수 함수**에만 | “비싼 계산” + “입력으로만 결정”일 때 |

> **중요**: `RESULT_CACHE`도 **외부 테이블 변경을 자동 인지**하지 못하는 PL/SQL 함수 패턴이 있다(함수 내부에서 SELECT). **의존성 추적 불가**라면 RC도 위험할 수 있다. 이 경우 **머티리얼라이즈드 뷰/조인 재작성** 등 **집합적 해결**을 우선 검토.

---

## 함수기반 인덱스(FBI)/가상컬럼에서의 필수 요건

### 안전한 예 — 문자열 정규화 인덱스

```sql
CREATE OR REPLACE FUNCTION fx_norm_ascii(p_txt VARCHAR2)
RETURN VARCHAR2
DETERMINISTIC
IS
BEGIN
  RETURN TRIM(LOWER(REPLACE(p_txt, ' ', '')));
END;
/

CREATE TABLE t_customers (
  cust_id NUMBER PRIMARY KEY,
  name    VARCHAR2(200),
  name_norm GENERATED ALWAYS AS (fx_norm_ascii(name)) VIRTUAL
);

CREATE INDEX ix_customers_name_norm ON t_customers(name_norm);

-- SARGable 검색
SELECT * FROM t_customers
WHERE name_norm = fx_norm_ascii(:q);  -- 인덱스 사용
```
- **조건**: 함수는 **오직 입력 인자만** 사용, 외부 상태 0%.

### 위험한 예 — 환율 인덱스

```sql
CREATE OR REPLACE FUNCTION fx_rate_det(p_ccy VARCHAR2)
RETURN NUMBER
DETERMINISTIC
IS
  v NUMBER;
BEGIN
  SELECT rate INTO v FROM rates WHERE ccy=p_ccy; -- 외부 테이블
  RETURN v;
END;
/

-- ❌ 이런 함수로 FBI/가상컬럼 만들면 데이터 변경 시 인덱스가 현실과 불일치
```

> **테스트 팁**: **DML 없이** 외부 테이블 내용만 바꾸고 질의가 바뀌면 **거짓 결정성**. FBI에 쓰면 **인덱스가 거짓말**을 한다.

---

## “결정성”을 깨는 요소 체크리스트

- [ ] `SYSDATE`/`SYSTIMESTAMP`/`CURRENT_DATE`
- [ ] `DBMS_RANDOM`, `SYS_GUID`, `SEQUENCE.NEXTVAL`
- [ ] **세션 상태**: `NLS_*`, `TIME_ZONE`, `CURRENT_SCHEMA`, `ROLE`, `CLIENT_IDENTIFIER`, `MODULE/ACTION` 등 `SYS_CONTEXT`
- [ ] **패키지·전역 변수**(상태 유지)
- [ ] **테이블/뷰 쿼리**(데이터가 바뀜)
- [ ] **외부 자원**(파일/네트워크/DB Link)

하나라도 **예**면 `DETERMINISTIC` **금지**.

---

## 성능 관점: 행단위 함수 호출의 비용과 대안

> `SELECT ... WHERE fx(col)=:x` 는 **행마다 함수 호출**. 함수가 비싸면 **I/O보다 CPU가 병목**.

### 집합 재작성(가장 안전/빠름)

```sql
-- 나쁜 패턴
SELECT * FROM sales WHERE fx_rate_det(ccy) > 1300;

-- 안전한 집합형(조인으로 변환)  ← fx_rate_det가 rates를 조회했다면 원래 이게 본질
SELECT s.*
FROM sales s
JOIN rates r ON r.ccy = s.ccy
WHERE r.rate > 1300;
```

### 스칼라 서브쿼리 캐시 (Oracle이 자동 캐시)

```sql
-- 같은 키에 대해 여러번 평가되면 엔진이 결과를 캐시
SELECT *
FROM sales s
WHERE (SELECT r.rate FROM rates r WHERE r.ccy=s.ccy) > 1300;
```
- **주의**: 그래도 `DETERMINISTIC`처럼 FBI 요건은 충족 못함. 단, **함수 호출 폭탄**은 완화.

### `RESULT_CACHE` (PL/SQL, 결정성 보장 가능할 때만)

```sql
CREATE OR REPLACE FUNCTION fx_tax(p_region VARCHAR2)
RETURN NUMBER
RESULT_CACHE  -- 캐시하고 싶을 때
IS
  v NUMBER;
BEGIN
  -- 테이블 읽는 함수의 RC는 무효화/일관성이 설계에 달림(주의)
  SELECT rate INTO v FROM tax_table WHERE region=p_region;
  RETURN v;
END;
/
```
- **의존성 추적**이 자동이 아니라면 RC도 **스테일 위험**. 보통 **MV/스냅샷** 설계가 더 안전.

---

## 병렬/분산/에디션·권한 관점의 함정

- **병렬 실행**: 슬레이브 프로세스들이 **세션 상태**가 다르거나(예: NLS) 환경이 달라지면 결정성이 깨질 수 있다.
- **DB Link**: 원격 DB의 데이터/세션 상태에 의존 → 결과가 시간/장소에 따라 달라질 수 있다.
- **Editioning**: `EDITIONING VIEW` 하에서 함수가 참조하는 객체 구현이 **에디션별로 다르면** 출력이 달라질 수 있다.
- **Invoker Rights**(`AUTHID CURRENT_USER`) 함수가 권한/보안 정책(VPD)에 따라 **보는 데이터가 변하면** 결정성 위배.

**결론**: 이런 상황은 `DETERMINISTIC` **금지**. 필요하면 **조인/재작성/스냅샷**으로 구조적 해결.

---

## 실제로 “틀린 결과”가 나오는 시연

```sql
-- 외부 테이블 rates
CREATE TABLE rates (ccy VARCHAR2(3) PRIMARY KEY, rate NUMBER);
INSERT INTO rates VALUES ('USD', 1300);
COMMIT;

CREATE OR REPLACE FUNCTION fx_rate_bad(p_ccy VARCHAR2)
RETURN NUMBER
DETERMINISTIC
IS
  v NUMBER;
BEGIN
  SELECT rate INTO v FROM rates WHERE ccy=p_ccy;
  RETURN v;
END;
/

-- FBI/가상컬럼
CREATE TABLE txn (
  id  NUMBER PRIMARY KEY,
  ccy VARCHAR2(3),
  usd_rate GENERATED ALWAYS AS (fx_rate_bad(ccy)) VIRTUAL
);
CREATE INDEX ix_txn_rate ON txn(usd_rate);

INSERT INTO txn(id, ccy) VALUES (1,'USD'); COMMIT;

-- 지금은 1300이므로 다음이 TRUE
SELECT * FROM txn WHERE usd_rate = 1300;

-- 외부 테이블만 갱신(인덱스/행 DML 없음)
UPDATE rates SET rate=1400 WHERE ccy='USD'; COMMIT;

-- 논리적으로는 1400이어야 하지만, 인덱스/가상컬럼은 1300으로 고정되어 있을 수 있다!
SELECT * FROM txn WHERE usd_rate = 1400; -- ❌ 못 찾을 수 있음(틀린 결과)
SELECT * FROM txn WHERE usd_rate = 1300; -- ❌ 찾을 수 있음(오래된 값)
```

> 요점: **인덱스 키는 행의 데이터에서 결정**돼야 한다. 외부 테이블/시간/세션에 의존하는 함수에 `DETERMINISTIC`을 주면, **인덱스와 현실이 어긋난다**.

---

## 안전한 사용 패턴

1) **순수 계산**: 수학적 변환, 포맷 정규화(공백/문장부호 제거), 고정 규칙 매핑(단, 룩업 테이블 X).
2) **바이트/비트 조작**: `HASH`, `CRC`(단, 해시 알고리즘 Seed가 고정이고 NLS/캐릭터셋 영향 없는 전제).
3) **언어·세션 독립**: NLS에 영향받는 `INITCAP/UPPER/LOWER` 대신 **NLS 고정** 또는 **ASCII 전용 로직**.
4) **가상컬럼/FBI**: 입력 컬럼만 변환, 외부 참조 금지.

**예시(OK)**:
```sql
CREATE OR REPLACE FUNCTION fx_phone_norm(p_txt VARCHAR2)
RETURN VARCHAR2
DETERMINISTIC
IS
BEGIN
  -- 숫자만 남기기 (로케일 무관)
  RETURN REGEXP_REPLACE(p_txt, '[^0-9]', '');
END;
/
```

---

## `DETERMINISTIC`과 옵티마이저 상호작용

- 옵티마이저는 `DETERMINISTIC`을 보고 **함수 호출 축소**(공통 값 중복 제거) 등을 시도할 수 있다.
- 하지만 **비용 모델상 유의미한 축소**가 아닐 때는 그대로 호출한다.
- FBI/가상컬럼/SARGable 경로가 생기면 **I/O가 크게 줄어** 성능 향상 여지가 큼.

---

## 단위 테스트/감사 스크립트 (결정성 검증)

### 반복 호출 일관성

```sql
WITH t(val) AS (
  SELECT :x FROM dual
)
SELECT CASE WHEN COUNT(DISTINCT fx_candidate(val))=1
            THEN 'CONSISTENT' ELSE 'INCONSISTENT' END as verdict
FROM t CONNECT BY LEVEL <= 1000;  -- 동일 입력 1000회 평가
```

### 세션/NLS 변화 시 검증

```sql
ALTER SESSION SET NLS_COMP=BINARY;
SELECT fx_candidate(:x) FROM dual;

ALTER SESSION SET NLS_COMP=LINGUISTIC;
ALTER SESSION SET NLS_SORT=GERMAN;
SELECT fx_candidate(:x) FROM dual;  -- 결과가 달라지면 DETERMINISTIC 금지
```

### 시간/패키지 상태 검증

```sql
-- 시간
SELECT fx_candidate(:x) FROM dual;
EXEC DBMS_LOCK.SLEEP(2);
SELECT fx_candidate(:x) FROM dual; -- 값이 바뀌면 금지

-- 패키지 상태
BEGIN pkg_stateful.g_flag := 1; END;
SELECT fx_candidate(:x) FROM dual;
BEGIN pkg_stateful.g_flag := 0; END;
SELECT fx_candidate(:x) FROM dual; -- 바뀌면 금지
```

---

## 실무 가이드라인(체크리스트)

- [ ] 함수가 **입력 인자 외 아무것도** 보지 않는가? (테이블/시간/세션/패키지 상태 X)
- [ ] **NLS/Locale/Timezone** 영향이 없는가? (있다면 고정/격리)
- [ ] FBI/가상컬럼에 쓸 계획이라면 **수학적 순수성**을 *증명*할 수 있는가?
- [ ] 캐시가 필요하면 `RESULT_CACHE`·SQL Result Cache·스칼라 서브쿼리 캐시 등 **안전한 캐시**를 먼저 고려했는가?
- [ ] 이미 `DETERMINISTIC` 붙인 함수가 외부 테이블을 읽는다면 **즉시 제거/재설계**(조인/뷰/스냅샷).
- [ ] 배포 전 **결정성 테스트**(세션/NLS/시간 변화)와 **리뷰**를 거쳤는가?

---

## 대안 설계 예제

### 룩업을 함수로 하지 말고 **조인**으로

```sql
-- ❌ (위험) 룩업 함수 + DETERMINISTIC
SELECT * FROM sales WHERE fx_rate_det(ccy) > 1300;

-- ✅ 조인으로 본질화
SELECT s.*
FROM sales s
JOIN rates r ON r.ccy = s.ccy
WHERE r.rate > 1300;
```

### “정규화 + FBI” 안전 패턴

```sql
CREATE OR REPLACE FUNCTION fx_email_norm(p VARCHAR2)
RETURN VARCHAR2
DETERMINISTIC
IS
BEGIN
  RETURN LOWER(REGEXP_REPLACE(p, '\s', ''));
END;
/

ALTER TABLE users ADD email_norm GENERATED ALWAYS AS (fx_email_norm(email)) VIRTUAL;
CREATE INDEX ix_users_email_norm ON users(email_norm);

-- 이제 빠른 탐색 가능
SELECT * FROM users WHERE email_norm = fx_email_norm(:q);
```

---

## 요약

- `DETERMINISTIC`은 **캐시 기능이 아니라 “절대 변하지 않음”을 보증하는 선언**이다.
- 이 선언을 믿고 엔진은 **함수기반 인덱스/가상컬럼/재작성** 등을 허용한다.
- **거짓 선언**은 **잘못된 인덱스/틀린 결과**를 만든다.
- **진짜 순수 함수**에만 붙이고, 그 외에는 **조인/집합 재작성/Result Cache/스칼라 서브쿼리 캐시** 같은 **안전한 대안**을 사용하라.
- 마지막으로, 배포 전 **결정성 테스트 스위트**로 NLS/세션/시간/상태 의존을 반드시 털어내라. 숫자가 말해준다.
