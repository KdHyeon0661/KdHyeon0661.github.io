---
layout: post
title: DB 심화 - Cursor Sharing
date: 2025-11-05 21:25:23 +0900
category: DB 심화
---
# Oracle **Cursor Sharing** 완전 가이드

> **핵심 요약**
> - **Cursor** = 파싱된 SQL의 **공유 가능한 실행 단위**(라이브러리 캐시 객체).
> - **Cursor Sharing** = **동일/유사 SQL**이 **같은 커서(Parent/Child)** 를 **재사용**하도록 하는 모든 메커니즘(바인드 변수, `CURSOR_SHARING`, Adaptive Cursor Sharing 등).
> - 목표: **하드 파싱 감소**, **Shared Pool 절약**, **라이브러리 캐시 경합·멀텍스/락 감소**, **일관된 플랜**.
> - 위험: **값 편중**(히스토그램) 상황에서 **무차별 바인드화**는 **잘못된 플랜 고착** → 성능 급락.
> - 해법: 바인드 변수 **기본 원칙**, **Adaptive Cursor Sharing(ACS)**, **히스토그램**, 필요한 곳엔 **리터럴 분리**, **SQL Plan Baseline**까지.

---

## 실습 환경 준비

```sql
-- 스키마 정리
DROP TABLE cs_demo PURGE;

-- 편중된 데이터 생성: status 컬럼에 'HOT' 1%, 'COLD' 99%
CREATE TABLE cs_demo (
  id      NUMBER PRIMARY KEY,
  status  VARCHAR2(10),
  payload VARCHAR2(200)
);

BEGIN
  FOR i IN 1..100000 LOOP
    INSERT INTO cs_demo
    VALUES (
      i,
      CASE WHEN DBMS_RANDOM.VALUE(0,100) < 1 THEN 'HOT' ELSE 'COLD' END,
      RPAD('x',100,'x')
    );
  END LOOP;
  COMMIT;
END;
/

CREATE INDEX ix_cs_demo_status ON cs_demo(status);

-- 통계 + 히스토그램(편중 정확화)
BEGIN
  DBMS_STATS.GATHER_TABLE_STATS(
    ownname    => USER,
    tabname    => 'CS_DEMO',
    method_opt => 'for columns status size 254',
    cascade    => TRUE
  );
END;
/
```

---

## 커서 구조 이해 — Parent/Child, 공유의 단위

### Parent vs Child

- **Parent Cursor**: **SQL 텍스트** 단위의 상위 핸들(정규화된 텍스트+env).
- **Child Cursor**: **실제 실행체(플랜/환경)**; 바인드/환경/NLS/오브젝트 상태에 따라 **복수(child#)** 생성 가능.
  - 예: 동일 텍스트라도 **바인드 값 편중/환경 차이** → 서로 다른 Child 플랜 생성.

### 공유 조건(대표)

- 텍스트 동일(공백·대소문자·힌트 포함), 스키마/권한/NLS *등* 환경 동일, 파라미터/outline 등 영향 동일.
- **바인드 변수 사용** 시, **리터럴 값 차이**가 있어도 Parent 공유 가능(Child는 ACS로 갈라질 수 있음).

### 관련 뷰

```sql
SELECT sql_id, child_number, plan_hash_value, executions, parsing_schema_name
FROM   v$sql
WHERE  sql_text LIKE 'SELECT /* CS_DEMO */%';

SELECT sql_id, version_count, loads, parse_calls
FROM   v$sqlarea
WHERE  sql_text LIKE 'SELECT /* CS_DEMO */%';

-- Child가 갈라진 이유들(플래그)
SELECT * FROM v$sql_shared_cursor WHERE sql_id = :sql_id;
```

---

## 바인드 변수 vs 리터럴 — “공유”의 출발점

### 리터럴 난사(나쁜 예)

```sql
-- 각기 다른 리터럴 → 서로 다른 텍스트 → Parent 자체가 공유 불가
SELECT /* CS_DEMO */ COUNT(*) FROM cs_demo WHERE status='HOT';
SELECT /* CS_DEMO */ COUNT(*) FROM cs_demo WHERE status='COLD';
SELECT /* CS_DEMO */ COUNT(*) FROM cs_demo WHERE status='H0T';  -- 오타도 다른 커서
```
- 결과: **Parent/Child 폭증**, **파싱 오버헤드↑**, **Shared Pool 메모리↑**, **library cache load lock/mutex** 경합 위험.

### 바인드 변수(정석)

```sql
VAR b VARCHAR2(10)
EXEC :b := 'HOT';

SELECT /* CS_DEMO */ COUNT(*)
FROM cs_demo
WHERE status = :b;
```
- **장점**: 파싱 재사용, 커서 공유, 보안(SQL Injection 리스크↓).
- **주의**: **값 편중**이 큰 컬럼에서는 **한 가지 플랜**이 모두에게 적합하지 않을 수 있음(→ 4장).

---

## `CURSOR_SHARING` 파라미터 — **강제 유사-텍스트 공유**

```sql
SHOW PARAMETER cursor_sharing;
-- EXACT(기본), FORCE, SIMILAR(10g에서 사실상 폐기/비권장)
```

- **EXACT**: 텍스트가 **정확히 같아야** 공유(권장 기본).
- **FORCE**: 엔진이 리터럴을 **자동 바인드 치환**해 공유 폭을 키움.
  - 장점: 레거시 앱(바인드 미사용)을 **즉시** 구제(파싱/커서 폭발 차단).
  - 단점: **값 편중** 무시 → **부정확한 선택도**로 **나쁜 플랜 고착**(히스토그램 무력화되는 경우 존재), 일부 함수 기반·상수 폴딩 로직에 부작용 가능.
- **SIMILAR**: 과거 “선택도 비슷하면 공유” 취지였으나 **비일관성** 문제로 **비권장/사장**.

> **권장 원칙**:
> 1) 애플리케이션 **바인드 변수**가 **정답**.
> 2) 불가피할 때만 **FORCE** 를 **임시 안전망**으로 사용(핫 타임 그레이스풀).
> 3) 근본 해결 후 **EXACT** 복귀.

**영향 관찰**
```sql
-- 전/후 비교: parse_calls, loads, version_count
SELECT sql_id, executions, parse_calls, loads, version_count
FROM   v$sqlarea
ORDER  BY parse_calls DESC FETCH FIRST 20 ROWS ONLY;
```

---

## **Bind Peeking** & **Adaptive Cursor Sharing(ACS)**

### 개념

- **Bind Peeking**: 커서 최초 하드 파싱 때 **바인드 값**을 **엿보고(peek)** 그 값 기준으로 **선택도/플랜** 결정.
  - 문제: 최초 값이 **비대표적**(예: `status='HOT'` 1%)이면, 이후 `COLD`(99%)에도 그 플랜을 **재사용** → 성능 붕괴.
- **ACS**: 실행 중 **카디널리티 피드백**을 통해 “**바인드 범위 구간**”을 학습, **Child를 여러 개**로 갈라 각 값 범위에 **다른 플랜**을 매칭.
  - **Bind-Sensitive → Bind-Aware** 전환, `v$sql_cs_selectivity` 등으로 관찰.

### 실습 — 편중 컬럼 + 바인드

```sql
-- 바인드 기반 동일 SQL 두 번: HOT → COLD
VAR s VARCHAR2(10)

EXEC :s := 'HOT';
SELECT /* CS_DEMO */ COUNT(*) FROM cs_demo WHERE status=:s;

EXEC :s := 'COLD';
SELECT /* CS_DEMO */ COUNT(*) FROM cs_demo WHERE status=:s;

-- 실제 수행 통계/플랜 확인
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST +PEEKED_BINDS +NOTE'));
```
**기대**
- 1차(HOT) 하드 파싱: 인덱스 범위 스캔(적은 행) 선택.
- 2차(COLD) 실행 후 옵티마이저가 **카디널리티 괴리**를 감지 → **Bind-Aware** 전환 → **Child 분기** 생성(Full Scan 선택).
- `v$sql_shared_cursor`에서 Child 분기 이유(바인드/환경) 플래그 확인.

```sql
SELECT sql_id, child_number, is_bind_sensitive, is_bind_aware, is_shareable
FROM   v$sql
WHERE  sql_text LIKE 'SELECT /* CS_DEMO */%';
```

> **전제**: 히스토그램·통계가 있어야 편중 인식이 정확(ACS가 더 빨리/정확히 분기).

---

## 세션 커서 캐시 vs 라이브러리 캐시 — “Soft Parse 줄이기”

- **라이브러리 캐시**: DB 전체 공유, Parent/Child 커서가 **SGA**에 존재.
- **세션 커서 캐시(session cached cursors)**: **세션 프로세스**(UGA/PGA) 레벨에서 **최근 사용 커서 핸들**을 보관 → **REPARSE 비용 감소**, `parse count (total)`↓.
  - 파라미터: `SESSION_CACHED_CURSORS` (일반 50~200 적정; 트래픽·SQL 다양도에 따라 조정).
  - 진단: 세션 통계 **`session cursor cache hits`** 상승 기대.

```sql
SHOW PARAMETER session_cached_cursors;

-- 세션 통계 확인(현재 세션)
SELECT name, value
FROM   v$sesstat st JOIN v$statname sn ON st.statistic#=sn.statistic#
WHERE  sn.name IN ('parse count (total)','parse count (hard)','session cursor cache hits')
AND    st.sid = SYS_CONTEXT('USERENV','SID');
```

---

## Child Cursor 폭증(Versioning) — 원인과 해법

### 원인(일부)

- **바인드 타입/길이/정확성 불일치**(예: 숫자 컬럼에 문자열 바인드)
- **Optimizer 환경 차이**(parameter/NLS/`optimizer_features_enable`)
- **권한/Edition/Shared Pool 상태 등**
- **ACS** 로 의도적으로 분기(정상 케이스)

### 진단

```sql
-- Version Count 많은 SQL 찾기
SELECT sql_id, version_count, loaded_versions, kept_versions, executions
FROM   v$sqlarea
ORDER  BY version_count DESC FETCH FIRST 30 ROWS ONLY;

-- Child가 나뉜 사유 플래그
SELECT * FROM v$sql_shared_cursor WHERE sql_id = :sql_id;
```

### 해법

- **바인드 타입 일치**(숫자↔숫자, 날짜↔날짜), **길이 안정화**.
- **세션 파라미터 표준화**(Connection Pool에서 통일).
- 필요 시 **플랜 기준 통제**: SQL Plan Baseline / Outline / 힌트.
- **과도한 ACS 분기**는 통계/히스토그램 조정, 리터럴 분리(소수의 인기값만)로 완화.

---

## 애플리케이션 레이어에서의 Cursor Sharing 베스트프랙티스

### JDBC

```java
// ✅ PreparedStatement 사용(바인드)
String sql = "SELECT /* CS_DEMO */ COUNT(*) FROM cs_demo WHERE status = ?";
try (PreparedStatement ps = conn.prepareStatement(sql)) {
    ps.setString(1, "HOT");
    try (ResultSet rs = ps.executeQuery()) {
        if (rs.next()) System.out.println(rs.getInt(1));
    }
}
```

### Python (cx_Oracle / oracledb)

```python
import oracledb
conn = oracledb.connect(dsn="...", user="...", password="...")
cur = conn.cursor()
cur.execute("SELECT /* CS_DEMO */ COUNT(*) FROM cs_demo WHERE status=:s", s="HOT")
print(cur.fetchone()[0])
```

### ODP.NET

```csharp
using var cmd = new OracleCommand(
    "SELECT /* CS_DEMO */ COUNT(*) FROM cs_demo WHERE status=:s", conn);
cmd.Parameters.Add(new OracleParameter("s", OracleDbType.Varchar2, "HOT", ParameterDirection.Input));
var count = (decimal)cmd.ExecuteScalar();
```

### PL/SQL

```plsql
DECLARE
  v_cnt NUMBER;
BEGIN
  EXECUTE IMMEDIATE
    'SELECT /* CS_DEMO */ COUNT(*) FROM cs_demo WHERE status=:s'
    INTO v_cnt USING 'HOT';
  DBMS_OUTPUT.PUT_LINE(v_cnt);
END;
/
```

> **공통 원칙**:
> - 바인드 **타입/길이**를 **컬럼과 일치**시켜라.
> - 커넥션 풀에서 **세션 파라미터**를 **일관화**하라.
> - 대량 DML은 **배치 바인드**(array DML)로 **call 수↓**.

---

## `CURSOR_SHARING=FORCE` 의 사용 가이드(임시처방)

### 언제 유용한가

- 레거시 앱이 **모든 SQL을 리터럴로** 발행 → 하드 파싱 폭발/Shared Pool 압박.
- 긴급 상황에서 **즉각 파싱 부하**를 낮춰야 할 때.

### 리스크

- 히스토그램/편중 반영 **약화** → **나쁜 플랜 고정**.
- 상수 폴딩/함수 기반 인덱스/결정성 의존 로직 **부작용** 가능.

### 전략

- **단기 적용** + **Top SQL 정규화**(바인드화)를 **병행**.
- **핫 값**(예: 매우 흔한 `COLD`)과 **드문 값**(예: `HOT`)을 **선별 리터럴 분리**(동적 SQL)하여 **플랜 이원화**.
- **장기**: `EXACT` 복귀 + **ACS/히스토그램**으로 해결.

---

## 바인드 + 리터럴 **하이브리드 전략**(현실적)

- 대부분의 조건은 **바인드**로 **공유 극대화**.
- **극단적 편중 컬럼**(상위 몇 값)만 **리터럴 분리**:
  - 앱 레벨에서 if-else: 인기값이면 **리터럴 SQL**(특화 인덱스/플랜), 그 외는 **바인드 SQL**.
  - 또는 **SQL Profile/Baseline**으로 값 별 플랜 고정.
- 효과: **공유**와 **선택도 정확성**을 동시에.

예시:
```sql
-- 바인드 기본 + 드문 값은 리터럴로(힌트/플랜 고정)
-- 1) 기본 바인드 SQL
SELECT /* CS_BASE */ COUNT(*) FROM cs_demo WHERE status = :s;

-- 2) 드문 값(예: 'HOT')만 리터럴 분리(플랜을 인덱스 강제)
SELECT /* CS_HOT */ /*+ index(cs_demo ix_cs_demo_status) */ COUNT(*)
FROM cs_demo WHERE status='HOT';
```

---

## 모니터링 & 진단 레시피

### 파싱/커서 지표

```sql
-- 시스템/세션 파싱 지표
SELECT name, value FROM v$sysstat WHERE name LIKE 'parse count%';
SELECT name, value FROM v$sesstat st JOIN v$statname sn ON st.statistic#=sn.statistic#
WHERE sn.name IN ('parse count (total)','parse count (hard)','session cursor cache hits')
AND st.sid = SYS_CONTEXT('USERENV','SID');

-- Top 파싱 SQL
SELECT sql_id, parse_calls, executions, version_count, sql_text
FROM   v$sqlarea
ORDER  BY parse_calls DESC FETCH FIRST 30 ROWS ONLY;
```

### 커서 공유 상태

```sql
-- Parent/Child 현황
SELECT sql_id, child_number, plan_hash_value, is_bind_sensitive, is_bind_aware, executions
FROM   v$sql
WHERE  sql_text LIKE 'SELECT /* CS_% */%';

-- Child 분리 사유
SELECT * FROM v$sql_shared_cursor WHERE sql_id = :sql_id;
```

### Mutex/라이브러리 캐시 경합

```sql
SELECT event, COUNT(*) samples
FROM   v$active_session_history
WHERE  sample_time > SYSTIMESTAMP - INTERVAL '10' MINUTE
AND    (event LIKE 'library cache%' OR event LIKE 'cursor:%mutex%')
GROUP  BY event ORDER  BY samples DESC;
```

---

## 트러블슈팅 시나리오

### “바인드화 했더니 느려졌다”

- **현상**: `CURSOR_SHARING=FORCE` 또는 앱 바인드 도입 후 **전체 느려짐**.
- **원인**: 편중 컬럼(히스토그램)에서 **대표성 없는 플랜**이 모든 값에 **재사용**.
- **조치**:
  1) **히스토그램 수집/검증**(`method_opt`)
  2) **ACS가 작동**하는지 확인(Child 분기)
  3) 인기 값만 **리터럴 분리** or **히ント/플랜 고정**
  4) 장기: **EXACT** + 바인드 + **ACS** 정착

### Child 폭증으로 경합

- **현상**: `version_count` ≫, `cursor: pin S wait on X` 증가.
- **조치**: 바인드 **타입/길이** 통일, **세션 파라미터 표준화**, 필요 시 **플랜 고정**.

### 파싱 폭발

- **현상**: `parse count (total/hard)` 급증, shared pool 압박.
- **조치**: 바인드 적용, `SESSION_CACHED_CURSORS` ↑, 상위 리터럴 SQL 정규화, 임시로 `CURSOR_SHARING=FORCE`.

---

## 실전 체크리스트

- [ ] **바인드 변수**를 **항상** 기본값으로(타입/길이 일치).
- [ ] `CURSOR_SHARING`은 **EXACT** 유지, 불가피할 때만 **FORCE(임시)**.
- [ ] **히스토그램**으로 편중 인식, **ACS** 활성(바인드-어웨어 분기).
- [ ] **세션 커서 캐시**(50~200)로 리파스 비용↓.
- [ ] **Child 이유 분석**(`v$sql_shared_cursor`)로 폭증 원인 제거.
- [ ] **핫 값**은 **리터럴 분리**·플랜 고정으로 이원화.
- [ ] 지표로 검증: `parse count`, `version_count`, `session cursor cache hits`, `mutex/library cache` 이벤트.

---

## 요약

- Cursor Sharing은 **파싱된 SQL을 최대한 재사용**하는 기술/관행의 총합이다.
- **바인드 변수**가 정석, **ACS+히스토그램**이 편중 문제를 해결한다.
- `CURSOR_SHARING=FORCE`는 **응급 처방**일 뿐, 장기적으로는 **EXACT**+**정상적 바인딩**으로 복귀해야 한다.
- 모든 판단은 **계측으로**: Parent/Child 수, 파싱 카운트, 대기 이벤트, 실제 실행 통계로 **전/후 비교**하라.
