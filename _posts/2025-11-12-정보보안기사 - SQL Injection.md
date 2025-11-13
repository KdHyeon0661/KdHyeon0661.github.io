---
layout: post
title: 정보보안기사 - SQL Injection
date: 2025-11-12 20:25:23 +0900
category: 정보보안기사
---
# SECTION 08 웹 애플리케이션 취약점 — 01. SQL Injection 취약점 완전 정리 (원리·유형·DBMS 차이·실습용 최소 재현·방어 전략·점검/모니터링·체크리스트·예상문제)

## 개요 — “쿼리와 데이터의 경계가 무너지면”
**SQL Injection(SQI)**은 애플리케이션이 **사용자 입력을 SQL 구문과 안전하게 분리하지 못할 때** 발생한다. 공격자는 입력값을 이용해 **쿼리 구조를 변형**하여 인증 우회, 임의 조회/수정, 시스템 명령 실행(일부 DBMS)까지 도달할 수 있다.
핵심은 단 하나: **쿼리와 데이터 사이에 ‘파라미터 경계’를 만든다(=준비문/바인딩)**.

---

## 위협 모델과 원리

### 데이터 흐름
```
[클라이언트 입력] -> [웹서버] -> [쿼리 생성] -> [DB서버] -> [결과 반환]
                               ^    (문자열 연결로 SQL 생성 시 취약)
```

### 형식적 관점(간단)
사용자 입력 `x`가 쿼리 템플릿 `Q(·)`에 삽입될 때,
- 안전: $$ \text{safe\_sql} = Q(\text{bind}(x)) $$
- 취약: $$ \text{unsafe\_sql} = Q(\text{concat}(x)) $$
여기서 `bind`는 **파라미터 바인딩**, `concat`은 **문자열 병합**을 뜻한다.

---

## 취약점 유형 분류

### 1. In-band(직접 반사/결과 수집)
- **Union-based**: 결과셋을 **UNION SELECT**로 합쳐 민감 정보 노출
- **Error-based**: DB 오류 메시지에 스키마/데이터가 반사

### 2. Blind(응답에 데이터 직접 없음)
- **Boolean-based**: 조건식 참/거짓에 따라 응답 길이나 상태 차이 관찰
- **Time-based**: 지연 함수(`SLEEP`, `pg_sleep`, `WAITFOR DELAY`, `DBMS_LOCK.SLEEP`)로 참/거짓 구분

### 3. Out-of-band(OoB)
- DB가 **외부 채널**(DNS/HTTP 등)로 데이터를 밀어내는 케이스(일부 환경/권한에서 가능)

### 4. 특수 상황
- **Stacked queries**: 여러 쿼리를 `;`로 이어 실행(드라이버/DB 설정에 따라 차이)
- **2차(Stored) 인젝션**: 저장된 악성 입력이 **나중에** 다른 쿼리 문맥에서 실행
- **ORM/빌더 인젝션**: ORM을 쓴다고 안전이 아님. **문자열 보간**·Raw 쿼리로 취약
- **NoSQL/OS Query 계열**: SQL 이외의 질의 언어도 입력이 구조에 섞이면 같은 원리로 취약

---

## DBMS별 차이(요약)

| 항목 | MySQL/MariaDB | PostgreSQL | SQL Server | Oracle |
|---|---|---|---|---|
| 주석 | `-- `, `#`, `/* */` | `-- `, `/* */` | `-- `, `/* */` | `-- `, `/* */` |
| 지연 | `SLEEP(n)` | `pg_sleep(n)` | `WAITFOR DELAY '0:0:n'` | `DBMS_LOCK.SLEEP(n)` |
| 에러 기반 | `updatexml()`, `extractvalue()`(낙후) | type cast 오류 등 | `CONVERT` 오류 등 | to_number 오류, XML함수 등 |
| 다중 쿼리 | 드라이버 설정 영향 | 기본 분리, 기능적 제한 | 가능(설정 영향) | 기본 1스테이트먼트 |

> 공통: **준비문(Prepared Statement) + 바인딩**을 쓰면 DB별 차이가 대부분 상쇄된다.

---

## 안전한 vs 취약한 코드 (언어별 미니 예시)

> 아래 **취약 예시**는 **로컬 랩/교육 목적**으로만 사용하라. 운영 코드에 **반드시 안전 예시**를 적용할 것.

### Java (JDBC)
```java
// ❌ 취약: 문자열 연결
String sql = "SELECT * FROM users WHERE id = '" + request.getParameter("id") + "'";
ResultSet rs = stmt.executeQuery(sql);
```

```java
// ✅ 안전: PreparedStatement + 바인딩
String sql = "SELECT * FROM users WHERE id = ?";
PreparedStatement ps = conn.prepareStatement(sql);
ps.setString(1, request.getParameter("id"));
ResultSet rs = ps.executeQuery();
```

### Python (psycopg2, PostgreSQL)
```python
# ❌ 취약
cur.execute(f"SELECT * FROM products WHERE name LIKE '%{q}%'")
```
```python
# ✅ 안전
cur.execute("SELECT * FROM products WHERE name LIKE %s", (f"%{q}%",))
```

### C# (.NET, Dapper)
```csharp
// ❌ 취약
var sql = $"SELECT * FROM Orders WHERE OrderId = {orderId}";
var rows = conn.Query(sql);
```
```csharp
// ✅ 안전
var rows = conn.Query("SELECT * FROM Orders WHERE OrderId = @id", new { id = orderId });
```

### PHP (PDO)
```php
// ❌ 취약
$sql = "SELECT * FROM users WHERE email = '".$_GET['email']."'";
$rows = $pdo->query($sql);
```
```php
// ✅ 안전
$stmt = $pdo->prepare("SELECT * FROM users WHERE email = :email");
$stmt->execute([':email' => $_GET['email']]);
$rows = $stmt->fetchAll(PDO::FETCH_ASSOC);
```

### Node.js (pg)
```js
// ❌ 취약
const sql = `SELECT * FROM blog WHERE slug = '${req.params.slug}'`;
const { rows } = await client.query(sql);
```
```js
// ✅ 안전
const { rows } = await client.query(
  "SELECT * FROM blog WHERE slug = $1",
  [req.params.slug]
);
```

> 원칙: **1) 항상 준비문, 2) 식별자(테이블/컬럼)까지 동적으로 바인딩해야 하면 화이트리스트만 허용**.

---

## 입력 문맥별 위험 포인트

- **문자열 리터럴 문맥**: `'` 종료 → 조건 삽입 → 주석으로 뒷부분 무효화
- **숫자 문맥**: 숫자라서 안전? → **숫자처럼 보이는 문자열**이 그대로 병합되면 여전히 취약
- **식별자 문맥**(컬럼/테이블 이름): **바인딩 불가**가 일반적 → **화이트리스트**만
- **LIMIT/OFFSET, ORDER BY**: 프레임워크가 종종 문자열 결합 → 화이트리스트/정수 검증 필수

---

## 미니 실습(로컬 교육용) — “취약 → 악용 관찰 → 즉시 패치”

> 목적: **취약 동작을 눈으로 확인**하고, **바로 패치**하는 훈련.
> 대상 DB: SQLite(내장). 운영망/실서버 금지.

### 1. 최소 Flask 앱 (취약 라우트 + 안전 라우트)
```python
# app.py (학습용)
from flask import Flask, request, jsonify
import sqlite3

app = Flask(__name__)

def q(sql):
    conn = sqlite3.connect(":memory:")
    c = conn.cursor()
    c.execute("CREATE TABLE users(id INTEGER PRIMARY KEY, name TEXT, role TEXT)")
    c.executemany("INSERT INTO users(name, role) VALUES (?,?)",
                  [("admin","admin"),("alice","user"),("bob","user")])
    rows = c.execute(sql).fetchall()   # 위험: 그대로 실행
    conn.close()
    return rows

@app.get("/vuln")
def vuln():
    name = request.args.get("name","")
    # ❌ 취약: 문자열 연결
    sql = f"SELECT id,name,role FROM users WHERE name = '{name}'"
    try:
        return jsonify(q(sql))
    except Exception as e:
        return {"error": str(e)}, 500

@app.get("/safe")
def safe():
    name = request.args.get("name","")
    # ✅ 안전: 바인딩
    conn = sqlite3.connect(":memory:")
    c = conn.cursor()
    c.execute("CREATE TABLE users(id INTEGER PRIMARY KEY, name TEXT, role TEXT)")
    c.executemany("INSERT INTO users(name, role) VALUES (?,?)",
                  [("admin","admin"),("alice","user"),("bob","user")])
    rows = c.execute("SELECT id,name,role FROM users WHERE name = ?", (name,)).fetchall()
    conn.close()
    return jsonify(rows)

if __name__ == "__main__":
    app.run(debug=True)
```

- `/vuln?name=alice` → 정상 행
- `/safe?name=alice` → 정상 행
- `/vuln?name=...`에 악성 입력을 넣으면 **쿼리 구조가 변형**됨을 확인 가능(로컬 학습 전용).
  “왜 깨지는가?”를 관찰한 뒤 즉시 `/safe` 접근법으로 **수정(Prepared Statement)**.

> 운영 코드에서는 **에러 메시지/스택 트레이스 출력 금지**, 표준화된 오류 응답 사용.

---

## 방어 전략 — “바인딩 만능론 + 최소권한 + 검증/로깅”

### 1. 1차 방어선: **준비문(Prepared Statement) + 파라미터 바인딩**
- 모든 **데이터 값**은 바인딩. 문자열을 **절대** 쿼리에 직접 넣지 않는다.
- LIKE 검색: `"name LIKE ?"` + `('%' + term + '%')` 식으로 값만 바인딩.

### 2. 2차: **화이트리스트 검증**
- **식별자/키워드**(ORDER BY 컬럼, 정렬 방향, 테이블명 등)는 **사전에 정한 집합**에서만 허용.
- 정수·범위·리스트 등 **스키마 기반** 입력 검증(JSON 스키마, Joi, FluentValidation 등).

### 3. 3차: **최소 권한 DB 계정**
- 애플리케이션 DB 계정은 **SELECT/INSERT/UPDATE/DELETE** 등 필요한 권한만.
- **DDL/EXECUTE/OS 명령 호출** 권한 제거. **xp_cmdshell**(MSSQL) 같은 위험 기능 비활성.
- **스키마 분리**: 관리·집계·배치 계정 분리.

### 4. 추가: **저장 프로시저는 만능 아님**
- 프로시저 내부에서 **동적 SQL 문자열**을 만들면 **똑같이 취약**.
- 프로시저도 **파라미터 바인딩**을 사용할 것.

### 5. 에러/로깅/모니터링
- **DB 오류 메시지 노출 금지**(스택 트레이스 숨김, 일반화 응답).
- **SIEM/로그**에 실패 쿼리 패턴, **SLEEP/pg_sleep/WAITFOR** 호출, **주석/UNION 키워드** 빈도 등 탐지 룰.
- **WAF/프록시 룰**: 공격 징후 차단(완전한 방어 X, **방어망의 한겹**으로 사용).

### 6. 빌드 파이프라인 보안
- **SAST**: 문자열 연결 쿼리 탐지 룰(예: `ExecuteQuery(string concatenation)`).
- **IAST/RASP/DAST**: 런타임/동적 분석으로 인젝션 시도 탐지.
- **보안 코드 리뷰 체크리스트** 준수.

---

## 진단/테스트(실무 팁 중심)

- **테스트 범위**: 모든 데이터 입력 경로(Form, QueryString, JSON, Header, Cookie, URL segment).
- **패턴 힌트(하이레벨)**: `'`·`")`·`)--`·`/*`·`UNION`·`SELECT`·지연 함수 호출 등 입력 시 **응답/로그**의 변화.
- **Blind 시나리오**: 응답 시간 변곡점, 글자 수, 정렬/페이징 이상, 500/200 비율 변화.
- **자동화 도구**: 사내 규정/승인 하에 **테스트 환경에서만** 활용(DAST 등).
  결과는 **재현 절차·영향·완화책**을 포함한 보고서로 귀결.

> 중요: 실서버 무단 테스트 금지. **승인된 범위**의 QA/스테이징에서만.

---

## 방어 레시피(프레임워크별 관용구)

### Spring(Data/JPA)
- `@Query` + `:param` 바인딩, `JpaRepository` 메서드 쿼리 사용
- **Native Query**가 꼭 필요하면 **파라미터**로만, 식별자 화이트리스트
- `JdbcTemplate`도 `?` 바인딩 사용

### Django/SQLAlchemy
- Django ORM QuerySet / SQLAlchemy Core/ORM 바인딩
- Raw SQL은 `text()` + `bindparams()`(SQLAlchemy), Django는 `params` 인자 사용

### ASP.NET EF Core/Dapper
- LINQ → SQL 변환(안전) / Dapper는 **이름 바인딩**
- FromSqlRaw는 변수 직접 결합 금지, 반드시 파라미터화 API 사용

### Laravel/Node ORM(Prisma/TypeORM/Sequelize)
- Query Builder/ORM 메서드 우선, Raw 쿼리는 바인딩 자리표시자 사용
- 마이그레이션/시드에서 문자열 결합 금지

---

## 운영·모니터링 룰(아이디어)

- **지연 함수 호출 탐지**: SLEEP/pg_sleep/WAITFOR/DBMS_LOCK 패턴 알림
- **Union/stacked 키워드 빈도** 모니터
- **500 오류 급증** + **특정 파라미터** 상관관계 경보
- **WAF**: SQLi 시그니처 룰, **레이트 리미트**(파라미터 조합 기준)

---

## 체크리스트(실무용)

- [ ] 모든 데이터 경로에서 **준비문 + 바인딩** 사용(검색/페이징/정렬 포함)
- [ ] **식별자/정렬/컬럼/OrderBy** 등은 **화이트리스트**
- [ ] DB 계정 **최소 권한**, 위험 기능 비활성(xp_cmdshell 등)
- [ ] **에러 메시지 외부 노출 금지**, 표준화된 오류 응답
- [ ] **로그/모니터링**: 의심 키워드/지연/오류 패턴 탐지 룰
- [ ] **테스트/리뷰**: SAST/DAST/코드리뷰 통과, 취약 경로 회귀 테스트
- [ ] **ORM 사용 지침** 수립(문자열 보간 금지, Raw 최소화)
- [ ] **보안 게이트**: PR 템플릿에 “SQL 문자열 결합 없음” 확인란
- [ ] **교육**: 신규 입사자/외주 개발자 대상 분기별 트레이닝

---

## 예상문제(실기 중심 미니 문항)

1) **문항**: 다음 중 SQL Injection 방어에 **가장 근본적인** 방법은?
   a) 입력 필터링(블랙리스트) b) 준비문/파라미터 바인딩 c) WAF 적용 d) 에러 숨김
   **정답**: b

2) **문항**: 아래 코드의 문제점과 수정안을 간단히 서술하시오.
```python
# Flask + SQLite
q = request.args.get("q","")
sql = f"SELECT * FROM emp WHERE name LIKE '%{q}%'"
rows = db.execute(sql)
```
**모범답안 요지**: 문자열 결합으로 취약. `"name LIKE ?"` 와 값 `("%"+q+"%",)`를 바인딩.

3) **문항**: DB 계정 권한 설계 시 최소 권한 원칙에 따라 제거할 항목 2가지를 쓰시오.
   **예시 답**: DDL 권한, OS 명령 실행 권한(확장 프로시저/외부 스크립트 호출 등)

4) **문항**: Blind Time-based 탐지에 사용되는 DB별 지연 함수 하나씩 예를 들라.
   **예시 답**: MySQL `SLEEP`, PostgreSQL `pg_sleep`, SQL Server `WAITFOR DELAY`, Oracle `DBMS_LOCK.SLEEP`

5) **문항**: ORM을 사용해도 SQLi가 발생하는 전형적 상황을 한 가지 서술하시오.
   **예시 답**: Raw SQL 문자열 보간, 동적 ORDER BY/컬럼명을 사용자가 지정하고 화이트리스트 검증 없이 그대로 결합하는 경우.

---

## 요약
- SQLi는 **데이터가 쿼리 구조를 침식**할 때 발생한다.
- **항상 ‘바인딩’으로 경계**를 만들고, 식별자/정렬 등은 **화이트리스트**로만.
- DB 권한 최소화, 에러 노출 금지, 로깅/모니터링으로 **조기 탐지**.
- SAST/DAST/코드리뷰/교육으로 **파이프라인 전반**에서 예방하라.
- 본 문서의 **안전 예시**를 프로젝트 템플릿에 반영하면 실무에서 대부분의 SQLi를 차단할 수 있다.
