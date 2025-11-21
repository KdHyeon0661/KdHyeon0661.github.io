---
layout: post
title: DB 심화 - Static SQL vs Dynamic SQL
date: 2025-10-28 23:25:23 +0900
category: DB 심화
---
# Static SQL vs Dynamic SQL

> **핵심 요약**
> - **Static SQL**: 쿼리 *텍스트*가 **컴파일 시점에 고정**. *PL/SQL의 정적 SQL, 커서, 프로시저 내부의 고정 쿼리*가 대표적.
> - **Dynamic SQL**: 쿼리 *텍스트*가 **런타임에 구성**·실행. *EXECUTE IMMEDIATE, DBMS_SQL, 애플리케이션에서 문자열로 조립한 SQL*이 대표적.
> - **성능/안정성/보안의 본질**은 **Static vs Dynamic** 구분이 아니라 **바인드 변수 사용 여부**다.
>   - 바인드를 쓰면 **커서 공유↑, 파싱 비용↓, 라이브러리 캐시 경합↓, SQL 인젝션 위험↓**
>   - 리터럴/문자열 연결로 값을 끼워 넣으면 (Static이든 Dynamic이든) **동일한 문제**가 발생한다.

---

## Static SQL과 Dynamic SQL — 정확한 정의

### Static SQL

- **정의**: *쿼리 텍스트*가 **변하지 않고** 코드에 **고정**되어 있으며, **컴파일 시점**에 구문 검증, 바인드 확인이 가능한 형태.
- **예**
  - PL/SQL 블록 안의 `SELECT ... INTO ...`
  - 미리 정의된 커서(`CURSOR c IS SELECT ...`)
  - 패키지/프로시저 본문에 **상수 텍스트**로 박혀 있는 SQL
- **장점**
  - 컴파일 타임 검증(문법·권한·바인드 자리수)
  - PL/SQL 최적화(Static SQL은 런타임 오버헤드 적음)
  - 보통 바인드 패턴이 자연스럽다
- **제약**
  - 테이블/컬럼/힌트/WHERE 조건의 **구조 자체**가 바뀌면 적용 어려움(런타임 변경 불가)

### Dynamic SQL

- **정의**: *쿼리 텍스트*를 **런타임에 문자열로 생성**해 실행(`EXECUTE IMMEDIATE`, `DBMS_SQL`, 애플리케이션에서 문자열 조립).
- **사용 이유**
  - 조건/대상 오브젝트/컬럼/힌트/ORDER BY 등을 **동적으로 바꿔야** 할 때
  - DDL/권한/스키마 명을 런타임에 정해야 할 때
  - **IN 목록 길이 가변**, PIVOT/UNPIVOT, 동적 컬럼 투영 등
- **주의**
  - 문자열 조립 시 **반드시 바인드**를 사용해 값은 분리
  - 오브젝트 명(테이블/컬럼)은 바인드 불가 → 화이트리스트 검증/인용자 처리 필요
  - Static과 동일하게 바인드만 잘 지키면 성능·보안 모두 무리 없다

> **정리**: Static/Dynamic 여부 자체가 선악을 가르는 기준이 아님. *“값”은 바인드로, “구조”는 동적이면 문자열로* — 이 원칙을 지켜야 한다.

---

## 성능·동시성·보안의 본질 = **바인드 변수**

### 커서 공유와 파싱

- **바인드 사용**: 텍스트가 같아 **Parent 커서** 1개로 모이고 **Child**도 재사용 → **소프트 파싱** 중심.
- **리터럴 남발**: 값마다 텍스트가 달라 **Parent** 폭증, **하드 파싱** 증가, `library cache: mutex X/S`, `cursor: pin S wait on X` 대기↑.

### 보안 — SQL Injection

- **문자열 연결**로 값 삽입 → 공격자가 **구문을 깨고** 임의 SQL 실행
- **바인드 사용**: 값이 **데이터**로만 전달, 구문은 **고정**되어 인젝션 차단

### 바인드 피킹/스큐 — 예외 관리

- 값 분포 스큐가 크면 **ACS(Adaptive Cursor Sharing)**, **히스토그램**으로 보정
- 극단적이면 **값대별 SQL 분리** 또는 **특정값 리터럴 분리(예외)**

---

## PL/SQL에서의 Static vs Dynamic — 예제들과 모범 패턴

### Static SQL (권장 기본)

```plsql
DECLARE
  v_sum NUMBER;
BEGIN
  SELECT /* static */ SUM(sal) INTO v_sum
  FROM   emp
  WHERE  deptno = :b1;  -- 바인드! (PL/SQL에서는 자동 바인드 자리)
  DBMS_OUTPUT.PUT_LINE('sum='||v_sum);
END;
/
```
- **특징**: 컴파일 타임 검증, 커서 재사용 좋음.

### Dynamic SQL (Native Dynamic SQL: EXECUTE IMMEDIATE)

> 구조(테이블/컬럼/ORDER BY)가 바뀔 수 있을 때만 동적화
```plsql
DECLARE
  v_sql  VARCHAR2(4000);
  v_sum  NUMBER;
  v_tab  VARCHAR2(30) := 'EMP';           -- 오브젝트 명: 바인드 불가 → 화이트리스트 필수
  v_col  VARCHAR2(30) := 'SAL';
  v_dept NUMBER := 10;
BEGIN
  -- 오브젝트/컬럼은 안전 인용 처리(화이트리스트 검증 후 더블쿼트 인용)
  v_sql := 'SELECT /* dynamic */ SUM("'||v_col||'") FROM "'||v_tab||'" WHERE deptno = :b1';

  EXECUTE IMMEDIATE v_sql INTO v_sum USING v_dept;  -- 값은 바인드!
  DBMS_OUTPUT.PUT_LINE('sum='||v_sum);
END;
/
```
**안전 수칙**
- 오브젝트/컬럼/ORDER BY 키워드 → **화이트리스트** 또는 **ALLOWLIST** 검증 (사전에 허용된 집합만 선택)
- 값은 항상 `USING` **바인드**로 전달
- `RETURNING INTO`도 바인드 가능:
```plsql
DECLARE
  v_sql VARCHAR2(4000);
  v_id  NUMBER := 1001;
  v_new NUMBER := 1200;
  v_old NUMBER;
BEGIN
  v_sql := 'UPDATE emp SET sal=:new WHERE empno=:id RETURNING sal INTO :old';
  EXECUTE IMMEDIATE v_sql USING v_new, v_id RETURNING INTO v_old;
  DBMS_OUTPUT.PUT_LINE('old='||v_old);
END;
/
```

### DBMS_SQL (아주 복잡한 동적 시나리오)

- 컬럼 수/타입이 **완전히 가변**일 때
- 일반적으로는 **EXECUTE IMMEDIATE**로 충분, DBMS_SQL은 특수 케이스 사용

---

## 일반 프로그래밍 언어에서의 작성법 — “바인드가 전부다”

### Java (JDBC)

#### **좋은 예** — PreparedStatement(바인드)

```java
String sql = "SELECT /* good */ COUNT(*) FROM emp WHERE deptno=? AND hiredate BETWEEN ? AND ?";
try (PreparedStatement ps = conn.prepareStatement(sql)) {
  ps.setInt(1, 10);
  ps.setDate(2, Date.valueOf("2025-10-01"));
  ps.setDate(3, Date.valueOf("2025-10-31"));
  try (ResultSet rs = ps.executeQuery()) {
    if (rs.next()) System.out.println(rs.getInt(1));
  }
}
```

#### **나쁜 예** — Statement + 문자열 연결(인젝션/성능 문제)

```java
// 절대 금지
String sql = "SELECT /* bad */ COUNT(*) FROM emp WHERE deptno=" + userDept +
             " AND hiredate BETWEEN '" + from + "' AND '" + to + "'";
try (Statement st = conn.createStatement();
     ResultSet rs = st.executeQuery(sql)) { ... }  // 인젝션/리터럴 파싱 난립
```

> **성능 팁**: Statement Cache/PreparedStatement Cache 활성화(드라이버) + 서버 `SESSION_CACHED_CURSORS` 병행

---

### Python (oracledb / cx_Oracle)

#### **좋은 예** — 바인드 딕셔너리

```python
sql = """SELECT /* good */ COUNT(*) FROM emp
         WHERE deptno=:dept AND hiredate BETWEEN :d1 AND :d2"""
cur.execute(sql, dept=10, d1=date(2025,10,1), d2=date(2025,10,31))
print(cur.fetchone()[0])
```

#### **나쁜 예** — f-string/format로 문자열 합치기

```python
# 금지

sql = f"SELECT COUNT(*) FROM emp WHERE deptno={dept} AND ename='{user_input}'"
cur.execute(sql)  # 인젝션/커서 공유 실패
```

---

### .NET (ODP.NET)

#### **좋은 예** — OracleCommand + Parameters

```csharp
var cmd = conn.CreateCommand();
cmd.BindByName = true;
cmd.CommandText = @"SELECT /* good */ SUM(sal) FROM emp
                    WHERE deptno=:dept AND hiredate BETWEEN :d1 AND :d2";
cmd.Parameters.Add("dept", OracleDbType.Int32).Value = 10;
cmd.Parameters.Add("d1", OracleDbType.Date).Value = new DateTime(2025,10,1);
cmd.Parameters.Add("d2", OracleDbType.Date).Value = new DateTime(2025,10,31);
var sum = Convert.ToDecimal(cmd.ExecuteScalar());
```

#### **나쁜 예** — 문자열 연결

```csharp
// 금지
cmd.CommandText = "SELECT SUM(sal) FROM emp WHERE ename='" + user + "'";
```

---

### Node.js (node-oracledb)

```js
const sql = `SELECT /* good */ COUNT(*) FROM emp
             WHERE deptno=:dept AND hiredate BETWEEN :d1 AND :d2`;
const result = await conn.execute(sql, { dept: 10, d1: new Date('2025-10-01'), d2: new Date('2025-10-31') });
console.log(result.rows[0][0]);
```

---

## “동적 WHERE”를 **안전하게** 구성하는 법 (값은 바인드, 구조만 조합)

### 패턴: 조건적 필터 조합

```plsql
DECLARE
  v_sql CLOB := 'SELECT empno, ename, sal FROM emp WHERE 1=1';
  v_binds DBMS_SQL.VARCHAR2_TABLE; -- 설명용
  v_idx PLS_INTEGER := 0;
  v_min_sal NUMBER := 2000;
  v_dept    NUMBER := NULL;
BEGIN
  IF v_min_sal IS NOT NULL THEN
    v_sql := v_sql || ' AND sal >= :b'||TO_CHAR(v_idx+1);
    v_idx := v_idx + 1;
  END IF;
  IF v_dept IS NOT NULL THEN
    v_sql := v_sql || ' AND deptno = :b'||TO_CHAR(v_idx+1);
    v_idx := v_idx + 1;
  END IF;

  -- EXECUTE IMMEDIATE + USING ... 순서대로 바인드 전달
  IF v_idx = 1 THEN
    EXECUTE IMMEDIATE v_sql USING v_min_sal;
  ELSIF v_idx = 2 THEN
    EXECUTE IMMEDIATE v_sql USING v_min_sal, v_dept;
  ELSE
    EXECUTE IMMEDIATE v_sql;
  END IF;
END;
/
```
- **원칙**: WHERE 조각은 **문자열**로 연결하되, **값은 바인드**로만 지정.

### IN 목록 가변 (바인드 배열)

- Oracle 12c+에서 **PL/SQL 바인드 배열**로 안전하게 구현 가능(언어별 드라이버도 배열 바인드 지원)
```plsql
DECLARE
  TYPE t_num IS TABLE OF NUMBER;
  v_list t_num := t_num(10, 20, 30);
  v_cnt  NUMBER;
BEGIN
  SELECT /* in-bind */ COUNT(*)
  INTO   v_cnt
  FROM   emp
  WHERE  deptno IN (SELECT COLUMN_VALUE FROM TABLE(v_list)); -- 안전
  DBMS_OUTPUT.PUT_LINE('cnt='||v_cnt);
END;
/
```
- 애플리케이션에서도 OracleCollection/배열 바인드를 활용(프레임워크별 상이).

---

## 성능/안정성 비교 시나리오

### 시나리오 A — Static SQL + 바인드 (권장 기준)

- **특징**: 파싱 안정, 커서 공유 극대화, 인젝션 불가
- **적용**: 대부분의 OLTP/배치

### 시나리오 B — Dynamic SQL + 바인드 (안전)

- **특징**: WHERE/ORDER/오브젝트 동적화 가능, 성능/보안 양호
- **주의**: 오브젝트 명 화이트리스트, 값은 전부 `USING` 바인드

### 리터럴 남발 (위험)

- **결과**: Parent 폭증, 하드 파싱↑, 뮤텍스 경합↑, 인젝션 위험
- **대응**: 전면 바인드 전환, 텍스트 템플릿 고정, 캐시(서버/클라이언트) 병행

---

## 바인드가 성능을 바꾸는 **지표** 확인(요약 스크립트)

```sql
-- 시스템 파싱 지표
SELECT name, value
FROM   v$sysstat
WHERE  name IN ('parse count (total)','parse count (hard)','parse time elapsed',
                'session cursor cache hits');

-- 특정 SQL의 Child/바인드 민감도(ACS)
SELECT sql_id, child_number, is_bind_sensitive, is_bind_aware, executions
FROM   v$sql
WHERE  sql_text LIKE 'SELECT /* good */%'
ORDER  BY child_number;

-- 공유 실패 이유(환경/바인드 타입 불일치 등)
SELECT child_number, reason
FROM   v$sql_shared_cursor
WHERE  sql_text LIKE 'SELECT /* good */%'
  AND  reason IS NOT NULL AND reason <> 'N';
```

---

## 보안 — 인젝션 방어 체크리스트

- [ ] **값은 100% 바인드 변수**로 전달(문자열 연결 금지)
- [ ] 오브젝트/컬럼/ORDER BY 키워드 등은 **화이트리스트**에서만 선택
- [ ] LIKE 검색 시 와일드카드(`%_`)는 **애플리케이션에서 이스케이프** 또는 **ESCAPE**절 사용
- [ ] 권한은 **최소 권한 원칙**(PROCEDURE + Definer’s Rights / Invoker’s Rights 적절히)
- [ ] 동적 SQL은 **로깅/감사**(누가 어떤 텍스트로 실행했는지)

**나쁜 예 → 좋은 예**
```plsql
-- ❌ 나쁜 예
v_sql := 'SELECT * FROM emp WHERE ename='''||v_name||'''';

-- ✅ 좋은 예
v_sql := 'SELECT * FROM emp WHERE ename=:b1';
EXECUTE IMMEDIATE v_sql USING v_name;
```

---

## “Static vs Dynamic” 설계 기준 트리

1) **쿼리 구조(테이블/컬럼/조인/정렬)가 고정**인가?
   - 예 → **Static SQL + 바인드**
   - 아니오 → 2

2) 구조를 일부 조합해야 하는가?
   - 예 → **Dynamic SQL + 바인드**(오브젝트는 화이트리스트)
   - 아니오 → Static

3) 값 분포 스큐로 플랜이 흔들리는가?
   - 예 → **히스토그램 + ACS** → 그래도 안 되면 **값대별 SQL 분리**
   - 아니오 → 기본 유지

4) 일부 단일 핫값만 특별히 다루고 싶은가?
   - 예외적으로 **리터럴 분리** 검토(Parent 분리) — *철저한 운영 규율 필수*

---

## 종합 예제 — “동적 Where + 안전 바인드 + 정렬 동적화”

### PL/SQL 패키지

```plsql
CREATE OR REPLACE PACKAGE emp_query_pkg AS
  TYPE t_emp_tab IS TABLE OF emp%ROWTYPE;
  FUNCTION list_emps(
     p_min_sal  NUMBER   DEFAULT NULL,
     p_deptno   NUMBER   DEFAULT NULL,
     p_order_by VARCHAR2 DEFAULT 'ENAME'  -- ENAME|SAL|HIREDATE only
  ) RETURN SYS_REFCURSOR;
END emp_query_pkg;
/

CREATE OR REPLACE PACKAGE BODY emp_query_pkg AS
  FUNCTION list_emps(p_min_sal NUMBER, p_deptno NUMBER, p_order_by VARCHAR2)
    RETURN SYS_REFCURSOR
  IS
    v_sql   CLOB := 'SELECT empno, ename, sal, hiredate, deptno FROM emp WHERE 1=1';
    v_rc    SYS_REFCURSOR;
    v_safe  VARCHAR2(30);
  BEGIN
    -- 화이트리스트 검증
    CASE UPPER(p_order_by)
      WHEN 'ENAME'   THEN v_safe := 'ENAME'
      WHEN 'SAL'     THEN v_safe := 'SAL'
      WHEN 'HIREDATE' THEN v_safe := 'HIREDATE'
      ELSE v_safe := 'ENAME'
    END CASE;

    -- 동적 WHERE
    IF p_min_sal IS NOT NULL THEN
      v_sql := v_sql || ' AND sal >= :b1';
    END IF;
    IF p_deptno IS NOT NULL THEN
      v_sql := v_sql || CASE WHEN p_min_sal IS NULL THEN ' AND deptno = :b1'
                             ELSE ' AND deptno = :b2' END;
    END IF;

    -- 동적 ORDER BY (오브젝트/컬럼만 문자열, 값은 바인드)
    v_sql := v_sql || ' ORDER BY '||v_safe;

    IF     p_min_sal IS NOT NULL AND p_deptno IS NOT NULL THEN
      OPEN v_rc FOR v_sql USING p_min_sal, p_deptno;
    ELSIF  p_min_sal IS NOT NULL THEN
      OPEN v_rc FOR v_sql USING p_min_sal;
    ELSIF  p_deptno IS NOT NULL THEN
      OPEN v_rc FOR v_sql USING p_deptno;
    ELSE
      OPEN v_rc FOR v_sql;
    END IF;

    RETURN v_rc;
  END;
END emp_query_pkg;
/
```
- **포인트**:
  - `ORDER BY`는 화이트리스트에서만 문자열 삽입
  - WHERE 값은 모두 **바인드**
  - **Static vs Dynamic** 혼용 가능(이 함수 내부는 동적이지만 모두 안전)

### 애플리케이션 호출 예 (Python)

```python
sql = "BEGIN :rc := emp_query_pkg.list_emps(:min_sal, :deptno, :order_by); END;"
rc = cur.var(oracledb.CURSOR)
cur.execute(sql, rc=rc, min_sal=2000, deptno=None, order_by="SAL")
for row in rc.getvalue():
    print(row)
```

---

## 트러블슈팅 FAQ

- **Q. Static인데도 커서가 공유가 안 됩니다.**
  **A.** 세션 파라미터/NLS/바인드 타입 불일치, 힌트/주석 차이, DDL/통계 변경으로 인한 Invalidation을 확인하세요.
  `V$SQL_SHARED_CURSOR.reason`, `is_bind_sensitive/aware` 등 점검.

- **Q. Dynamic으로 바꾸니 느려졌어요.**
  **A.** 값 바인드가 빠졌거나, 텍스트가 매 호출 달라졌을 가능성. 문자열 조합을 템플릿+바인드로 정규화하세요.
  Statement Cache/Session Cursor Cache도 활성화.

- **Q. IN 목록이 길어 문자열로 이어붙이면 빨라 보이던데요?**
  **A.** 잠깐입니다. 커서 공유가 끊어져 전체 시스템 성능을 갉아먹습니다. **배열 바인드/임시 테이블**/GTT를 고려하세요.

- **Q. 특정 값만 유독 느려요.**
  **A.** 스큐/바인드 피킹 이슈. **히스토그램+ACS**를 먼저, 그래도 크면 **값대별 SQL 분리**를.

---

## 체크리스트(운영 기준)

- [ ] **Static/Dynamic** 선택은 *업무 요구(구조 가변성)* 로 결정한다.
- [ ] **값은 100% 바인드**, 구조(오브젝트/컬럼/ORDER)는 **화이트리스트 문자열**.
- [ ] SQL 템플릿 **고정**(주석/공백/힌트 랜덤 생성 금지).
- [ ] 드라이버 **Statement Cache** + 서버 **SESSION_CACHED_CURSORS** 적정화.
- [ ] 지표로 검증: `parse count`, `session cursor cache hits`, 라이브러리 캐시 대기, RT.
- [ ] 스큐 컬럼 **히스토그램**/ACS/필요시 **SQL 분리**.
- [ ] 배포/통계는 **저부하 창**에 순차 적용(Invalidation 스파이크 방지).
- [ ] 보안 로깅/감사(동적 SQL 텍스트/호출자 기록).

---

## 결론 — “Static vs Dynamic”이 아니라 **Bind vs Literal**

- 대부분의 성능/보안 이슈는 **Static/Dynamic의 선택** 때문이 아니라, **값을 문자열로 끼워 넣는 습관**에서 발생한다.
- **원칙**:
  1) 구조가 고정이면 **Static SQL + 바인드**
  2) 구조가 가변이면 **Dynamic SQL + (화이트리스트) + 바인드**
  3) 스큐가 심하면 **히스토그램/ACS/SQL 분리**
  4) 항상 **Statement Cache + Session Cursor Cache**로 파싱 비용 최소화
- 이 네 가지만 지키면 대부분의 OLTP/보고 시스템에서 **안정적이고 빠른** SQL 실행 기반을 갖출 수 있다.

> **한 줄 요약**
> “Static vs Dynamic”은 **도구 선택**의 문제, **성능/보안**의 본질은 **바인드 변수**다.
> *값은 바인드, 구조는 화이트리스트 문자열* — 이 룰을 코드베이스 전체에 표준화하라.
