---
layout: post
title: Java - PreparedStatement, ResultSet
date: 2025-08-13 23:25:23 +0900
category: Java
---
# PreparedStatement, ResultSet / 트랜잭션 처리

## 0. 빠른 준비 — 커넥션/리소스 관리 기본 틀

가장 안전한 **try-with-resources** 패턴:

```java
DataSource ds = /* HikariCP 등으로 초기화 */;

try (Connection conn = ds.getConnection()) {
    conn.setReadOnly(false);
    conn.setAutoCommit(false);

    String sql = "UPDATE accounts SET balance = balance - ? WHERE id = ?";
    try (PreparedStatement ps = conn.prepareStatement(sql)) {
        ps.setBigDecimal(1, new BigDecimal("50.00"));
        ps.setLong(2, 101L);
        int n = ps.executeUpdate();
        if (n != 1) throw new SQLException("Not found: 101");
    }

    conn.commit();
} catch (SQLException e) {
    // 커넥션이 try-with-resources 블록을 벗어날 때 자동으로 닫히더라도,
    // 중간에 실패 시 즉시 롤백하는 것이 좋다.
    // (아래 트랜잭션 섹션에서 재시도/분류 전략을 다룬다)
    e.printStackTrace();
}
```

> **원칙**: `ResultSet` → `Statement` → `Connection` 순으로 **자동 닫힘**이 보장되도록 항상 `try-with-resources`를 사용한다.

---

## 1. PreparedStatement — 파라미터 바인딩/재사용/배치

### 1.1 개념 & 장점
- `?` **자리표시자**로 값과 SQL을 분리 → **SQL 인젝션** 방지
- 동일 SQL을 **재사용**할 때 파싱/플랜 캐시 **히트**
- JDBC 타입 ↔ DB 타입 **변환 책임**을 드라이버에 위임

### 1.2 기본 사용

```java
String sql = """
  SELECT id, name, email
  FROM users
  WHERE status = ? AND created_at >= ?
  ORDER BY id
""";

try (PreparedStatement ps = conn.prepareStatement(sql)) {
    ps.setString(1, "ACTIVE");
    ps.setTimestamp(2, Timestamp.valueOf("2025-01-01 00:00:00"));
    try (ResultSet rs = ps.executeQuery()) {
        while (rs.next()) {
            long id = rs.getLong("id");
            String name = rs.getString("name");
            String email = rs.getString("email");
            // ...
        }
    }
}
```

### 1.3 바인딩 메서드 필수 정리

| 카테고리 | 대표 메서드 | 메모 |
|---|---|---|
| 수치 | `setInt`, `setLong`, `setDouble`, `setBigDecimal` | 정밀도 필요한 금액은 **BigDecimal** |
| 문자열/이진 | `setString`, `setBytes`, `setBinaryStream`, `setCharacterStream` | 대용량은 스트림으로 |
| 날짜/시간 | `setDate`, `setTime`, `setTimestamp`, `setObject(LocalDate/LocalDateTime)` | 최신 드라이버는 `java.time`을 잘 지원 |
| NULL | `setNull(index, Types.XXX)` | **타입 명시 필수** |
| 기타 | `setObject(idx, value)` | 드라이버가 적절한 매핑 수행 |

> **TIP**: 원시형 `getXxx`는 NULL 구분이 어려우니, 읽을 때는 **래퍼/`getObject`**를 적절히 활용한다(2.1 참고).

### 1.4 배치(Batch) — 대량 DML 가속

```java
String sql = "INSERT INTO users(name, email, status) VALUES(?, ?, ?)";
try (PreparedStatement ps = conn.prepareStatement(sql)) {
    for (User u : users) {
        ps.setString(1, u.name());
        ps.setString(2, u.email());
        ps.setString(3, "ACTIVE");
        ps.addBatch();

        if (/* 청크 기준 */ false) {
            ps.executeBatch();
        }
    }
    int[] counts = ps.executeBatch();
    // counts 배열로 적용 결과 확인
}
```

- **트랜잭션**과 함께 사용(오류 시 일괄 롤백)
- DB/드라이버에 따라 **배치 재작성**(멀티 values) 최적화가 자동으로 적용되기도 함
- 초대량은 **청크 분할**(`executeBatch()` 주기 조절)로 메모리/락 경합 완화

### 1.5 생성된 키 회수 (AUTO_INCREMENT/시퀀스)

```java
String sql = "INSERT INTO users(name, email) VALUES(?, ?)";
try (PreparedStatement ps = conn.prepareStatement(sql, Statement.RETURN_GENERATED_KEYS)) {
    ps.setString(1, "Alice");
    ps.setString(2, "a@ex.com");
    ps.executeUpdate();

    try (ResultSet keys = ps.getGeneratedKeys()) {
        if (keys.next()) {
            long newId = keys.getLong(1);
            // newId 사용
        }
    }
}
```

> 일부 DB는 **배치 + 생성키**를 동시에 제한한다. 드라이버 릴리즈 노트 확인.

### 1.6 동적 IN 절 유틸 (가독성 개선)

```java
static class SqlWithParams {
    final String sql;
    final List<Object> params;
    SqlWithParams(String sql, List<Object> params){ this.sql=sql; this.params=params; }
}

static SqlWithParams inClause(String base, String col, List<?> vals) {
    String q = String.join(",", java.util.Collections.nCopies(vals.size(), "?"));
    String sql = base + " WHERE " + col + " IN (" + q + ")";
    return new SqlWithParams(sql, List.copyOf(vals));
}

// 사용
var built = inClause("SELECT id,name FROM users", "id", List.of(1L,2L,3L));
try (PreparedStatement ps = conn.prepareStatement(built.sql)) {
    int i=1; for (Object v: built.params) ps.setObject(i++, v);
    try (ResultSet rs = ps.executeQuery()) { /* ... */ }
}
```

### 1.7 (선택) 특수 타입 매핑 힌트
- **PostgreSQL 배열**: `ps.setArray(idx, conn.createArrayOf("text", arr))`
- **JSON**(PG): `ps.setObject(idx, "{\"k\":\"v\"}", java.sql.Types.OTHER)`
- **ENUM**: 문자열로 저장/매핑하거나 DB ENUM과 1:1 매핑(드라이버별 지원 확인)

---

## 2. ResultSet — 안전한 읽기/스트리밍/커서 제어

### 2.1 안전한 읽기 패턴(NUll 처리)

```java
try (ResultSet rs = ps.executeQuery()) {
    while (rs.next()) {
        Long id = rs.getObject("id", Long.class); // NULL 안전
        String name = rs.getString("name");       // NULL이면 null 반환
        Integer age = rs.getObject("age", Integer.class);
        // 원시형 사용 시 직전 호출에 대해 wasNull()로 구분 가능
    }
}
```

- `getXxx(1)`처럼 **인덱스 기반**은 빠르지만 컬럼 순서 변경에 취약 → 가능하면 **컬럼명 기반** 권장
- 대용량 BLOB/CLOB은 `getBinaryStream`/`getCharacterStream`으로 스트리밍

### 2.2 커서 타입/동시성/홀더빌리티

```java
try (PreparedStatement ps = conn.prepareStatement(
        sql,
        ResultSet.TYPE_FORWARD_ONLY,
        ResultSet.CONCUR_READ_ONLY)) {

    // 커밋 후 커서 유지 여부(드라이버 의존)
    conn.setHoldability(ResultSet.HOLD_CURSORS_OVER_COMMIT);

    try (ResultSet rs = ps.executeQuery()) { /* ... */ }
}
```

| 옵션 | 의미 | 메모 |
|---|---|---|
| `TYPE_FORWARD_ONLY` | 앞으로만 이동(기본/가장 빠름) | 대량 스트리밍 적합 |
| `TYPE_SCROLL_INSENSITIVE` | 앞뒤 스크롤 가능, 다른 트랜잭션 변경 미반영 | 드라이버 구현 차 |
| `TYPE_SCROLL_SENSITIVE` | 변경 반영 | 지원 드묾 |
| `CONCUR_UPDATABLE` | 결과 셋에서 `updateRow()` | 제약 多, 신중히 사용 |

### 2.3 페치 사이즈/스트리밍

```java
ps.setFetchSize(500);   // 네트워크 왕복/메모리 균형점 찾기
```

| DB/드라이버 | 흔한 팁 |
|---|---|
| PostgreSQL | `setFetchSize(N)`이 서버 커서 기반 스트리밍으로 동작 |
| MySQL | 최신 드라이버 + `useCursorFetch=true` 설정 후 `setFetchSize(N)` 사용. (구식 방식은 `Integer.MIN_VALUE`) |
| Oracle | `setFetchSize(N)`으로 네트워크 왕복 조절. LOB 스트리밍은 드라이버 가이드 참고 |

> **주의**: 스트리밍 시 커서를 닫기 전에 **ResultSet을 끝까지 소모**하거나 명시적으로 `close()` 하자. (풀링 커넥션의 리소스 누수 방지)

### 2.4 메타데이터 기반 매핑(가벼운 RowMapper)

```java
static Map<String, Object> rowToMap(ResultSet rs) throws SQLException {
    ResultSetMetaData md = rs.getMetaData();
    int n = md.getColumnCount();
    Map<String, Object> m = new java.util.LinkedHashMap<>(n);
    for (int i=1; i<=n; i++) {
        m.put(md.getColumnLabel(i), rs.getObject(i));
    }
    return m;
}
```

---

## 3. 트랜잭션 — 경계·격리·복구

### 3.1 기본 경계

```java
boolean old = conn.getAutoCommit();
conn.setAutoCommit(false);
try {
    // ... 여러 DML/조회
    conn.commit();
} catch (SQLException e) {
    conn.rollback();
    throw e;
} finally {
    conn.setAutoCommit(old);
}
```

- **짧고 명확한 범위**로 유지 (락 경합/타임아웃 최소화)
- `conn.setReadOnly(true)`는 옵티마이저 힌트로 작동하는 DB도 있다(선택)

### 3.2 격리 수준과 현상

```java
conn.setTransactionIsolation(Connection.TRANSACTION_READ_COMMITTED);
```

| 상수 | 방지되는 현상 |
|---|---|
| `READ_UNCOMMITTED` | (거의 없음) |
| `READ_COMMITTED` | **Dirty Read** 방지 |
| `REPEATABLE_READ` | + **Non-Repeatable Read** 방지 |
| `SERIALIZABLE` | + **Phantom Read** 방지(가장 엄격) |

> 실제 의미/기본값은 DBMS마다 조금씩 다르다. **요구 수준**과 **성능 비용**을 함께 고려하자.

### 3.3 세이브포인트 — 부분 롤백

```java
conn.setAutoCommit(false);
try {
    step1();
    Savepoint sp = conn.setSavepoint();

    step2(); // 실패할 수도 있는 구간
    conn.commit();
} catch (Exception e) {
    try { conn.rollback(sp); } catch (SQLException ignore) {}
    // step1은 유지, step2만 되돌아감
    conn.commit();
} finally {
    conn.setAutoCommit(true);
}
```

### 3.4 잠금/동시성 — `SELECT ... FOR UPDATE` 등

```sql
-- 행 잠금 예
SELECT * FROM accounts WHERE id = ? FOR UPDATE;
```

- 큐 처리에선 `FOR UPDATE SKIP LOCKED`(지원 DB 한정)로 **경합 완화**
- 인덱스 순서/범위를 사용해 **락 순서**를 일관되게 가져가면 교착상태 감소

### 3.5 실패 분류 & 재시도(Deadlock/Serialization)

```java
static boolean isRetryable(SQLException e) {
    String sqlState = e.getSQLState(); // 예: "40001" (serialization), "40P01" (PG deadlock)
    if (sqlState == null) return false;
    return sqlState.startsWith("40") || // 트랜잭션 롤백
           sqlState.startsWith("08");   // 연결 문제
}

<T> T withTxRetry(DataSource ds, int max, Callable<T> body) throws Exception {
    int attempt = 0;
    while (true) {
        attempt++;
        try (Connection c = ds.getConnection()) {
            boolean old = c.getAutoCommit();
            c.setAutoCommit(false);
            try {
                T out = body.call(); // 내부에서 c를 ThreadLocal로 꺼내 쓰는 구조 등
                c.commit();
                return out;
            } catch (SQLException e) {
                c.rollback();
                if (attempt >= max || !isRetryable(e)) throw e;
                // 지수 백오프
                Thread.sleep(Math.min(1000L, 50L * (1L << attempt)));
            } finally {
                c.setAutoCommit(old);
            }
        }
    }
}
```

> **주의**: **비멱등(POST-like) 작업**은 재시도 시 **중복 처리 방지 키**(idempotency key)나 **고유 제약**을 결합하자.

### 3.6 분산 트랜잭션이 필요하다면
- 서로 다른 DB/서비스를 하나의 트랜잭션으로 묶으려면 **XA/JTA** 같은 **분산 트랜잭션** 기술이 필요
- 가능하면 **사건 저장/보상 트랜잭션**(Saga) 등 **비동기/파편화** 패턴 고려

---

## 4. 실전 레포지토리 — 조회/삽입/배치/트랜잭션

```java
public record User(long id, String name, String email, String status) {}

public final class UserRepository {
    private final DataSource ds;

    public UserRepository(DataSource ds) { this.ds = ds; }

    public List<User> findActiveSince(LocalDate since) throws SQLException {
        String sql = """
          SELECT id, name, email, status
          FROM users
          WHERE status = ? AND created_at >= ?
          ORDER BY id
        """;
        try (Connection conn = ds.getConnection();
             PreparedStatement ps = conn.prepareStatement(sql)) {

            ps.setString(1, "ACTIVE");
            ps.setObject(2, since.atStartOfDay()); // 최신 드라이버는 java.time 매핑 OK
            ps.setFetchSize(500);

            List<User> out = new ArrayList<>();
            try (ResultSet rs = ps.executeQuery()) {
                while (rs.next()) {
                    out.add(new User(
                        rs.getLong("id"),
                        rs.getString("name"),
                        rs.getString("email"),
                        rs.getString("status")
                    ));
                }
            }
            return out;
        }
    }

    public long create(Connection conn, String name, String email) throws SQLException {
        String sql = "INSERT INTO users(name, email, status) VALUES(?,?, 'ACTIVE')";
        try (PreparedStatement ps = conn.prepareStatement(sql, Statement.RETURN_GENERATED_KEYS)) {
            ps.setString(1, name);
            ps.setString(2, email);
            ps.executeUpdate();
            try (ResultSet keys = ps.getGeneratedKeys()) {
                if (!keys.next()) throw new SQLException("No key");
                return keys.getLong(1);
            }
        }
    }

    public int[] bulkUpsert(Connection conn, List<User> users) throws SQLException {
        // 예: PostgreSQL 기준 upsert (DB별 문법 차이)
        String sql = """
          INSERT INTO users(id, name, email, status)
          VALUES(?, ?, ?, ?)
          ON CONFLICT (id) DO UPDATE
            SET name = EXCLUDED.name,
                email = EXCLUDED.email,
                status = EXCLUDED.status
        """;
        try (PreparedStatement ps = conn.prepareStatement(sql)) {
            int chunk = 0;
            for (User u : users) {
                ps.setLong(1, u.id());
                ps.setString(2, u.name());
                ps.setString(3, u.email());
                ps.setString(4, u.status());
                ps.addBatch();
                if (++chunk % 1000 == 0) ps.executeBatch();
            }
            return ps.executeBatch();
        }
    }
}
```

**서비스 계층에서 트랜잭션 경계**

```java
public final class UserService {
    private final DataSource ds;
    private final UserRepository repo;
    public UserService(DataSource ds) { this.ds = ds; this.repo = new UserRepository(ds); }

    public long register(String name, String email) throws SQLException {
        try (Connection conn = ds.getConnection()) {
            boolean old = conn.getAutoCommit();
            conn.setAutoCommit(false);
            try {
                long id = repo.create(conn, name, email);
                audit(conn, id, "REGISTER");
                conn.commit();
                return id;
            } catch (SQLException e) {
                conn.rollback();
                throw e;
            } finally {
                conn.setAutoCommit(old);
            }
        }
    }

    private void audit(Connection conn, long userId, String action) throws SQLException {
        try (PreparedStatement ps = conn.prepareStatement(
                "INSERT INTO audit(user_id, action) VALUES(?, ?)")) {
            ps.setLong(1, userId);
            ps.setString(2, action);
            ps.executeUpdate();
        }
    }
}
```

---

## 5. 시간 제한/취소/연결 문제 — 운영 팁

- **문 레벨**: `ps.setQueryTimeout(seconds)` — 드라이버/DB에 따라 **서버/클라이언트 중 어느 쪽에서** 시간 제한이 적용되는지 다름
- **소켓 레벨**: 드라이버 연결 문자열의 `socketTimeout`/`loginTimeout` 등의 **전송 계층** 타임아웃
- **취소**: 별도 스레드에서 `statement.cancel()` 가능(캔슬 지원 여부는 드라이버 의존)
- **예외 분류**:  
  - `SQLState` `"40xxx"`: 트랜잭션 롤백(교착/직렬화 실패) → **재시도 후보**  
  - `"08xxx"`: 연결 문제 → 재연결/재시도  
  - `"23xxx"`: 무결성 위반 → **로직 수정/사용자 피드백**

---

## 6. 커넥션 풀(DataSource) 스니펫 (HikariCP 예)

```java
import com.zaxxer.hikari.HikariConfig;
import com.zaxxer.hikari.HikariDataSource;

HikariConfig cfg = new HikariConfig();
cfg.setJdbcUrl("jdbc:postgresql://localhost:5432/app");
cfg.setUsername("app");
cfg.setPassword("secret");
cfg.setMaximumPoolSize(20);
cfg.setMinimumIdle(2);
cfg.setAutoCommit(false);          // 기본 false 권장(명시적인 트랜잭션)
cfg.setConnectionTimeout(5000);    // 연결 획득 타임아웃
cfg.setLeakDetectionThreshold(0);  // 필요 시 누수 감지 ms
DataSource ds = new HikariDataSource(cfg);
```

> **주의**: 풀로 얻은 커넥션은 **항상 원상 복구**(autoCommit/isolation/readOnly 등) 후 반환되도록 하자. 실패 시 풀의 `reset`/`initSQL` 옵션 검토.

---

## 7. 성능/안정성 체크리스트

- **안전성**
  - [ ] 문자열 연결 금지 → **PreparedStatement** 필수
  - [ ] NULL → `setNull(idx, Types.XXX)` 명시
  - [ ] `PreparedStatement`/`ResultSet`은 **스레드-세이프 아님**(한 스레드에서 사용)
- **성능**
  - [ ] 대량 DML → **배치 + 청크 + 트랜잭션**
  - [ ] 대량 조회 → **`setFetchSize` + 스트리밍**
  - [ ] 자주 쓰는 문장 → 같은 커넥션에서 **재사용**(드라이버/DB 플랜 캐시 히트)
- **트랜잭션**
  - [ ] 범위를 **짧게**, 실패 시 **롤백**, `finally`에서 상태 복원
  - [ ] 격리 수준은 **요건/비용**을 저울질
  - [ ] **세이브포인트**로 부분 롤백 설계
  - [ ] 교착/직렬화 실패는 **재시도 정책**(지수 백오프) 적용
- **리소스**
  - [ ] 항상 **try-with-resources**로 닫힘 보장
  - [ ] 풀 설정(최대/최소/타임아웃)과 **메트릭** 모니터링
- **설계**
  - [ ] SQL은 레포지토리 계층에 캡슐화, 매핑은 메서드/RowMapper로 분리
  - [ ] 동적 `IN`/업서트/락 쿼리 등 **유틸**로 재사용

---

## 8. (부록) 문제 재현/테스트 팁

- **단위 테스트**: JDBC 호출을 인터페이스로 추상화 후 **Fake/Mock** 주입  
- **통합 테스트**: Testcontainers로 DB 띄우고 실제 JDBC 실행  
- **대량/스트리밍**: Mock 서버 대신 **실제 DB**에서 페치 사이즈/네트워크 왕복을 체감  
- **타임아웃/캔슬**: 느린 함수/락을 고의로 만들어 시나리오 검증

---

### 마무리

- `PreparedStatement`는 **보안과 성능의 기본기**다.  
- `ResultSet`은 **NULL/타입/스트리밍** 포인트만 잡으면 안전해진다.  
- 트랜잭션은 **짧고 명확하게**, 실패는 **분류하고 재시도**하자.  

이 가이드의 패턴(try-with-resources, 배치+트랜잭션, 세이브포인트, 재시도, 페치 사이즈)은 **어떤 DBMS/드라이버에서도 그대로 재사용** 가능하다. 실전 환경에 맞춰 **격리/락/타임아웃**만 조정하면 된다.