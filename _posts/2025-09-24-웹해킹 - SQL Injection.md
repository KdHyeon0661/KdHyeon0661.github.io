---
layout: post
title: 웹해킹 - SQL Injection
date: 2025-09-24 15:25:23 +0900
category: 웹해킹
---
# — 기법과 방어 (Prepared Statement, ORM 중심 완전 가이드)

## 한눈에 보는 핵심

- **SQL Injection(이하 SQLi)** = *사용자 입력이 SQL “데이터”가 아닌 “코드”로 해석*되게 만들어 쿼리의 **의도**를 바꾸는 취약점. 로그인 우회, 데이터 열람/수정/삭제, 계정 탈취, 심하면 DB/서버 장악까지 이어짐.
- **최고(그리고 표준) 방어** = **준비된 문장(Prepared Statement, 파라미터 바인딩)** + **ORM의 파라미터화 쿼리**. 문자열 결합/보간 금지. **식별자(컬럼/테이블명)** 같은 *구조*는 파라미터화 대상이 아니므로 **화이트리스트 매핑**으로만 처리.
- **보조 방어** = 입력 허용리스트, 에러 메시지 제한, 최소 권한, 다중 문장 실행 비활성화, 로깅/모니터링, WAF. (보조는 *보완*일 뿐, 파라미터화의 대체가 아님)

---

## SQLi가 발생하는 지점과 “쿼리 의도” 붕괴

### 쿼리 조립의 두 얼굴

- **좋은 방식**:
  ```sql
  SELECT * FROM users WHERE email = ? AND pwd_hash = ?
  ```
  — SQL(코드)과 데이터(값)가 **분리**되고, 값은 **파라미터**로 전달됨.
- **나쁜 방식(취약)**:
  ```sql
  "SELECT * FROM users WHERE email = '" + email + "' AND pwd_hash = '" + h + "'"
  ```
  — 입력이 문자열 결합으로 쿼리 **구조**를 변경 가능.

> SQLi의 본질은 “데이터가 코드가 되지 못하게” 하는 것. 이를 가장 확실히 보장하는 표준 해법이 **Prepared Statement**다.

### SQLi 대표 유형(요약)

- **에러 기반**: 에러 메시지로 스키마/DB 정보 유출.
- **UNION 기반**: 결과 집합 합쳐서 데이터 추출.
- **블라인드(Boolean/Time)**: 참/거짓, 지연시간으로 참조 데이터 추출.
- **스택드 쿼리(다중 문장)**: `;`로 추가 명령 실행(드라이버/서버 설정 의존).
- **2차(Second-Order)**: 악성 입력이 **저장**되었다가 향후 다른 쿼리에서 **재해석**되며 발생.
- **OOB(Out-of-Band)**: DB가 **DNS/HTTP 등 외부 채널**로 결과를 전송하도록 유도.

---

## 공격 관점 “시나리오”로 이해하기 (방어 역설계에 필요)

> ✅ 모든 시나리오는 **학습용 로컬 랩**(DVWA, Juice Shop 등)에서만. 질의 패턴과 방어 포인트를 **반대로** 생각해보면 실무 설계가 훨씬 탄탄해집니다.

### 시나리오 A — 로그인 우회 (에러/UNION/블라인드로 변주)

- **취약 코드 (Node/Express + 가상 드라이버)**:
  ```javascript
  // ❌ 취약: 문자열 보간
  app.post("/login", async (req, res) => {
    const { email, password } = req.body;
    const sql = `SELECT id, role FROM users
                 WHERE email='${email}' AND pwd_hash=SHA2('${password}', 256)`;
    const rows = await db.query(sql);
    if (rows.length) return res.send("ok");
    res.status(401).send("nope");
  });
  ```
- **문제**: `email`에 `' OR '1'='1` 등을 주입하면 WHERE 절 전체가 참. (방어는 §4 참고)
  SQLi 정의·로그인 우회 랩 예시는 PortSwigger 문서가 체계적입니다.

### 시나리오 B — 검색창에서 스키마 덤프 (UNION 기반)

- **취약 코드 (Python/Flask + SQLite 가정)**:
  ```python
  @app.get("/search")
  def search():
      q = request.args.get("q","")
      # ❌ 취약
      rows = db.execute(f"SELECT title,price FROM products WHERE title LIKE '%{q}%'").fetchall()
      return jsonify(rows)
  ```
- **변주 포인트**
  - `ORDER BY <n>` 증가시켜 컬럼 수 추정 → `UNION SELECT` 구성.
  - DB별 문자열 연결/주석 구문(예: `--`, `#`, `/* */`) 차이를 기억(치트시트 참고).

### 시나리오 C — “다중 문장”이 켜진 드라이버

- 일부 클라이언트/드라이버는 **기본적으로 다중 문장 비활성화**(보안상 권장). 실수로 활성화하면 `; DROP TABLE ...` 류 스택드 쿼리가 가능. Node의 `mysql`/`mysql2`는 기본 **비활성**이며, 활성화 옵션은 신중히.

### 시나리오 D — 2차 SQLi

- **흐름**: ① 프로필 “닉네임”을 저장(정상) → ② 나중에 **관리자 검색 기능**이 `... WHERE name = '입력값'`을 문자열 결합해 실행 → ③ 저장된 값이 **그때** 코드로 작동. (진단/개념은 PortSwigger/Oracle 튜토리얼 참고)

### 블라인드

- SQL Server의 특정 확장 프로시저(예: `xp_dirtree`) 호출 유도 등으로 **DNS 쿼리** 발생 → 공격자 도메인에서 수신해 값 추출. (연구/랩 자료 다수)

---

## **준비된 문장(Prepared Statement)**: 왜 “표준”인가

### 개념

- 쿼리를 **미리 컴파일**(파싱/검증)하고, **값**은 **별도로 바인딩**. 데이터는 **절대 SQL 구조가 되지 않음**. 대부분의 DB/드라이버가 1급 기능으로 제공.

### DB/언어별 “정석” 예제

#### **MySQL** — 서버측 Prepared Statements

```sql
-- 서버 측: PREPARE / EXECUTE / DEALLOCATE
PREPARE s FROM 'SELECT id FROM users WHERE email=? AND pwd_hash=?';
SET @e='a@b.c', @p=SHA2('secret',256);
EXECUTE s USING @e, @p;
DEALLOCATE PREPARE s;
```
- MySQL 공식 문서: PREPARE/EXECUTE/DEALLOCATE 구조와 장점 명시.

#### **PostgreSQL** — PREPARE/EXECUTE, 드라이버 자동 파라미터화

```sql
PREPARE get_user(text, text) AS
  SELECT id FROM users WHERE email = $1 AND pwd_hash = $2;
EXECUTE get_user('a@b.c', '...hash...');
```
- 개념·동작은 공식 문서의 PREPARE 설명 참고.

#### **Java (JDBC)** — `PreparedStatement`

```java
String sql = "SELECT id, role FROM users WHERE email = ? AND pwd_hash = ?";
try (PreparedStatement ps = conn.prepareStatement(sql)) {
  ps.setString(1, email);
  ps.setString(2, hash(password));
  try (ResultSet rs = ps.executeQuery()) {
    ...
  }
}
```
- `PreparedStatement` 는 **사전 컴파일된 SQL**로 다회 실행/보안성 이점. (오라클/자바 공식)

#### **Python (psycopg)** — 자리표시자

```python
cur.execute(
  "SELECT id FROM users WHERE email=%s AND pwd_hash=%s",
  (email, pwd_hash)
)
```
- 파라미터는 SQL과 **분리해서** 전송. (psycopg3 문서의 파라미터 전송 설명)

#### **C# / ADO.NET**

```csharp
using var cmd = new SqlCommand(
  "SELECT Id FROM Users WHERE Email=@email AND PwdHash=@h", conn);
cmd.Parameters.AddWithValue("@email", email);
cmd.Parameters.Add("@h", SqlDbType.VarChar, 64).Value = hash;
using var reader = cmd.ExecuteReader();
```
- `SqlCommand.Parameters` 사용이 정석. (MS Docs)

#### **PHP PDO**

```php
$stmt = $pdo->prepare("SELECT id FROM users WHERE email=? AND pwd_hash=?");
$stmt->execute([$email, $hash]);
```
- mysqli/PDO 모두 **준비된 문장** 권장. (PHP/MySQL 문서)

---

## **ORM로 안전하게** — 프레임워크별 포인트

> ORM은 보통 **기본이 파라미터화**입니다. 단, **Raw SQL**, **문자열 보간**, **동적 식별자(컬럼/ORDER BY)**를 섞는 순간 취약해질 수 있습니다.

### SQLAlchemy (Python)

```python
# ✅ ORM 스타일 (자동 파라미터화)

user = session.query(User).filter(User.email == email).one_or_none()

# ✅ 텍스트 SQL도 바인딩

from sqlalchemy import text
session.execute(text("UPDATE users SET active=:a WHERE id=:id"),
                {"a": True, "id": uid})
```
- **항상 바인딩 파라미터**를 사용. 문자열 인라인은 위험 경고.

### Django ORM

```python
# ✅ 안전: QuerySet API

User.objects.filter(email=email)

# ❌ 위험: 문자열 결합 raw SQL
# User.objects.raw(f"SELECT * FROM users WHERE email = '{email}'")

```
- Django는 기본적으로 **파라미터화**. Raw를 써도 **params 전달** 필수. (Django 보안 문서)

### Sequelize (Node.js)

```js
// ✅ 바인드 파라미터
await sequelize.query(
  "SELECT * FROM users WHERE email = $email",
  { bind: { email } }
);

// ✅ Replacements (라이브러리 측 escaping) 또는 bind 권장
await sequelize.query(
  "SELECT * FROM products WHERE price > :p",
  { replacements: { p: 100 } }
);
```
- **bind vs replacements** 차이를 공식 문서가 설명. (둘 다 raw 보간보다 안전)

### Prisma (Node.js)

```ts
// ✅ 가능하면 ORM API 사용 (자동 파라미터화)
const user = await prisma.user.findUnique({ where: { email } });

// ✅ 불가피한 Raw: 반드시 $queryRaw / Tagged Template 사용
const rows = await prisma.$queryRaw`SELECT * FROM users WHERE email = ${email}`;
```
- Prisma는 **ORM 경로 권장**, raw 사용 시 안전 패턴 가이드 제공.

### EF Core (C#)

```csharp
// ✅ LINQ
var user = await ctx.Users.SingleOrDefaultAsync(u => u.Email == email);

// ✅ Raw가 필요하면 FromSql(Interpolated) 사용
var r = ctx.Users
  .FromSqlInterpolated($"SELECT * FROM Users WHERE Email = {email}");
```
- `FromSql`/인터폴레이션도 내부적으로 **파라미터화**. (MS Docs)

---

## “틀리기 쉬운” 케이스들: **식별자/IN/LIKE/다중문장**

### **식별자(컬럼/정렬키/테이블명)는 파라미터화 불가**

- 구조(Identifier)는 값이 아니라 **SQL 문법 요소**라 바인딩 불가.
  → 사용자 입력을 **그대로 식별자로 쓰지 말고**, **화이트리스트 매핑**을 적용:
  ```javascript
  // 정렬 키 허용 목록
  const ALLOWED = { name: "name", price: "price", created: "created_at" };
  const sort = ALLOWED[req.query.sort] || "created_at"; // 안전한 기본값
  const sql = `SELECT id,name,price FROM products ORDER BY ${sort} DESC`;
  ```
- OWASP도 *식별자엔 이스케이프/파라미터화가 통하지 않는다*고 명시.

### **IN 절의 가변 길이 파라미터**

- ORM/드라이버가 **자동 확장**을 제공하거나, 안전히 **자리표시자 반복** 생성.
  ```python
  # SQLAlchemy: 자동 바인딩 확장
  ids = [5, 6, 7]
  session.execute(text("SELECT * FROM t WHERE id IN :ids").bindparams(bindparam("ids", expanding=True),), {"ids": ids})
  ```
  - “리스트 파라미터를 각 자리표시자로 펼친다”는 패턴이 흔함.

### **LIKE 검색과 와일드카드 이스케이프**

- `LIKE '%...?%'`의 `%`/`_`는 **와일드카드** — *인젝션*과는 별개로, 사용자가 `_%` 등을 입력하면 과도한 매칭이 일어남.
- 파라미터화 + **와일드카드 이스케이프**를 병행:
  ```sql
  -- PostgreSQL: ESCAPE 지정 가능
  SELECT * FROM notes
   WHERE title LIKE '%' || REPLACE(REPLACE(:q, '%', '\%'), '_', '\_') || '%' ESCAPE '\';
  ```
  - ESCAPE/백슬래시 동작은 DB별 차이가 있으니 공식 문서 확인.

### 금지**

- Node `mysql/mysql2` 등은 **기본 비활성화**(권장). 활성화 시 공격 면이 커짐. 가능하면 **항상 비활성** 유지. (옵션/경고 참고)

---

## “방어 공방”으로 보는 **기법별 요령**

### UNION/에러 기반에 대한 설계 포인트

- **파라미터화**가 끝. 불필요한 상세 에러 **노출 금지**(스택 트레이스/SQL 오류문).
- DB 권한 최소화(리드온리 역할, `INFORMATION_SCHEMA` 접근 제한 등).
- **로깅**: 비정상 구문/경고 패턴을 구조화 로그로 남기기.

### 블라인드(Boolean/Time)

- **시간 지연 함수**(예: `SLEEP`, `pg_sleep`) 호출 자체를 막을 수는 없으나, 파라미터화로 **구조 변경**을 원천 차단.
- 과도한 지연/오류율 감지 알람.

### OOB/2차

- DB 기능/확장 프로시저(예: `xp_cmdshell`/`xp_dirtree`) 비활성/권한 최소화.
- 저장된 데이터가 **다른 곳**에서 문자열로 재조립되지 않게 **일관되게 파라미터화**.

---

## — 언어별 패치**

### Node.js (mysql2 / node-postgres)

```javascript
// ❌ before
const sql = `SELECT * FROM users WHERE email='${email}'`;
const rows = await conn.query(sql);

// ✅ after (mysql2)
const [rows] = await conn.execute(
  "SELECT * FROM users WHERE email = ?",
  [email]
);

// ✅ after (pg)
const { rows } = await client.query(
  "SELECT * FROM users WHERE email = $1",
  [email]
);
```
- `multipleStatements` 켜지지 않게 기본값 유지 권장.

### Python (psycopg/SQLAlchemy)

```python
# ❌ before

cur.execute(f"UPDATE users SET name='{name}' WHERE id={uid}")

# ✅ after (psycopg)

cur.execute("UPDATE users SET name=%s WHERE id=%s", (name, uid))

# ✅ after (SQLAlchemy Core)

from sqlalchemy import text
session.execute(text("UPDATE users SET name=:n WHERE id=:id"),
                {"n": name, "id": uid})
```
- SQLAlchemy는 **바인딩 파라미터**가 기본. 문자열 인라인은 경고.

### Java (JDBC)

```java
// ❌ Statement + 문자열 결합
Statement st = conn.createStatement();
ResultSet rs = st.executeQuery("SELECT * FROM u WHERE email='"+email+"'");

// ✅ PreparedStatement
PreparedStatement ps = conn.prepareStatement("SELECT * FROM u WHERE email=?");
ps.setString(1, email);
ResultSet rs2 = ps.executeQuery();
```
- `PreparedStatement`는 표준 해결책.

### C# (EF Core / ADO.NET)

```csharp
// ✅ LINQ 우선
var users = await ctx.Users.Where(u => u.Email == email).ToListAsync();

// ✅ Raw 필요 시
var list = await ctx.Users.FromSqlInterpolated(
  $"SELECT * FROM Users WHERE Email={email}").ToListAsync();
```
- `FromSql(Interpolated)`도 내부적으로 **파라미터화**.

### PHP (PDO)

```php
// ✅ PDO prepared
$stmt = $pdo->prepare("INSERT INTO logs(message, level) VALUES (?,?)");
$stmt->execute([$msg, $level]);
```
- mysqli도 동일 콘셉트(prepare/bind/execute).

---

## **Stored Procedure**는 만능? — “안전하게 쓰면” OK, 그 자체가 해답은 아님

- 안전하게 작성된 SP는 **파라미터화**와 동일한 효과를 낼 수 있으나, SP 내부에서 문자열 **동적 결합**(예: `EXEC('...'+@userInput)`) 하면 SQLi 여지 존재. OWASP도 “SP는 **항상 안전한 것이 아니다**”라고 명시.
- 동적 SQL이 필요하면 `sp_executesql` + **매개변수 바인딩** 사용. (EXEC보다는 안전/플랜 재사용)

---

## **테스트 전략** — 막았다는 “증거”를 남기는 방법

### 단위 테스트(예시: 로그인)

```javascript
// Jest 예시
await request(app).post("/login").send({
  email: "a@b.c' OR '1'='1",
  password: "x"
}).expect(401); // ✅ 파라미터화라면 실패해야 정상
```

### 페이로드 스모크 테스트

- 입력값 세트: `"' OR '1'='1"`, `"') OR 1=1 -- "`, `") OR SLEEP(2) --"`, `"; DROP TABLE x; --"` 등
- **결과 기대치**: 에러/성공이 아닌 “그냥 **문자열**로 취급”되어 **의도 변화 없음**. (자세한 SQL 구문은 치트시트 참고)

### 로깅/모니터링

- “예외적 따옴표/주석/지연” 패턴 알림.
- 실패 비율/지연 급증시 탐지.

---

## **운영 수칙 체크리스트**

1. **파라미터화 100%**: 모든 DML/SELECT, 보고/검색 API 포함. (ORM/드라이버가 지원)
2. **식별자 화이트리스트**: 정렬/컬럼 선택/테이블 스위치. (파라미터화 불가 영역)
3. **에러/스택 숨김**: 사용자 응답은 일반화, 내부 로깅은 상세.
4. **최소 권한**: 앱 계정은 필요한 권한만. 위험 프로시저/함수 차단.
5. **다중 문장 비활성화**: 드라이버 옵션 점검. (Node mysql2 기본 꺼짐)
6. **LIKE 와일드카드 이스케이프**: 필요한 경우만, ESCAPE 활용.
7. **정적 분석/코드 리뷰**: 문자열 결합 쿼리 금지 규칙.
8. **로그/탐지/알림**: 이상 패턴 초기에 대응.

---

## **현장에서 자주 묻는 것들 (FAQ)**

### Q1. “우리는 ORM 쓰는데 안전한가요?”

- **대부분의 ORM은 기본적으로 파라미터화**에 기반. 하지만 **Raw SQL + 문자열 보간**을 쓰는 순간 똑같이 취약합니다. (Django/SQLAlchemy/Sequelize/Prisma 문서 모두 같은 취지)

### Q2. “LIKE 검색은 파라미터화하면 끝 아닌가요?”

- 인젝션은 막지만, 사용자가 `%`/`_`로 의도치 않은 **범위 확장**을 유발할 수 있으므로 **와일드카드 이스케이프**를 병행하세요. (DB별 ESCAPE 규칙 참고)

### Q3. “Stored Procedure로만 만들면 안전?”

- **아닙니다.** SP 내부에서 동적 SQL을 조립하면 SQLi가 가능합니다. SP도 **파라미터화**를 지키세요.

### Q4. “IN 절에 리스트를 안전하게 넣는 방법?”

- 드라이버/ORM이 **자리표시자 확장**을 지원하거나, 리스트 길이에 맞춰 **파라미터를 생성**하세요. (SQLAlchemy 등 사례)

### Q5. “왜 모두가 Prepared Statement를 ‘표준’이라 하나요?”

- **의도 분리(코드≠데이터)**를 **언어/드라이버 수준**에서 강제하기 때문. 주요 DB/언어에서 **공식 1급 기능**으로 제공되고, 보안/성능(플랜 재사용) 이점도 큽니다.

---

## 부록 — **언어별 안전 레시피 모음**

### Node.js (mysql2 / pg)

```javascript
// mysql2
const [rows] = await pool.execute(
  "SELECT * FROM posts WHERE author_id = ? AND created_at >= ?",
  [authorId, since]
);

// pg (node-postgres)
const res = await client.query(
  "UPDATE posts SET title = $1 WHERE id = $2 AND author_id = $3",
  [title, id, authorId]
);
```
- (Tip) Node-MySQL은 다중문장 **기본 OFF**. 켜지 않기!

### Python (SQLAlchemy)

```python
# ORM

session.query(Post).filter(Post.author_id == uid, Post.created_at >= since)

# Core + text()

from sqlalchemy import text
session.execute(
  text("INSERT INTO audit(actor, action) VALUES (:a, :x)"),
  {"a": actor, "x": action}
)
```
- “바인딩을 인라인으로 렌더링하지 말라”는 경고.

### Java (JDBC)

```java
try (PreparedStatement ps = con.prepareStatement(
    "SELECT * FROM orders WHERE user_id=? ORDER BY created_at DESC LIMIT ?")) {
  ps.setLong(1, userId);
  ps.setInt(2, limit);
  try (ResultSet rs = ps.executeQuery()) { ... }
}
```
- `PreparedStatement`의 취지는 오라클 문서에 명확.

### C# (EF Core/ADO.NET)

```csharp
var q = ctx.Orders.Where(o => o.UserId == userId).OrderByDescending(o => o.CreatedAt).Take(limit);
```
- Raw SQL 필요시 `FromSqlInterpolated`/`SqlParameter` 사용.

### PHP (PDO)

```php
$stmt = $pdo->prepare("UPDATE users SET last_login=NOW() WHERE id=?");
$stmt->execute([$id]);
```

---

## 보너스 — **LIKE 안전 검색**(DB별 스니펫)

```sql
-- PostgreSQL
SELECT * FROM notes
WHERE title LIKE '%' || REPLACE(REPLACE(:q, '%', '\%'), '_', '\_') || '%'
ESCAPE '\';
```
- ESCAPE/백슬래시 규칙은 공식 문서를 참고.

```sql
-- MySQL
SELECT * FROM notes
WHERE title LIKE CONCAT('%', REPLACE(REPLACE(?,'%','\\%'),'_','\\_'), '%');
```
- MySQL의 백슬래시/ESCAPE 동작 차이 유의.

---

## **요약**

- **SQLi의 본질**: 입력이 쿼리 **코드**가 되지 못하게 하는 것.
- **항상**: Prepared Statement / 파라미터화. ORM도 **Raw+보간 금지**.
- **예외영역**(식별자/정렬/IN/LIKE/다중문장)은 별도 패턴으로 처리.
- **보조수단**(입력 허용리스트/권한 제한/오류 은닉/로그·탐지)은 “두께”를 더하는 역할.
- 참고: OWASP SQL Injection Prevention & Query Parameterization 치트시트, PortSwigger Web Security Academy.
