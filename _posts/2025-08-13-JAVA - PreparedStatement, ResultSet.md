---
layout: post
title: Java - PreparedStatement, ResultSet
date: 2025-08-13 23:25:23 +0900
category: Java
---
# PreparedStatement, ResultSet / 트랜잭션 처리 — 실전 가이드

JDBC로 안전하고 빠른 DB 코드를 작성하려면 **`PreparedStatement`**(파라미터 바인딩), **`ResultSet`**(결과 순회/매핑), **트랜잭션**(원자성/격리)을 정확히 다뤄야 합니다. 아래는 개념→핵심 API→베스트 프랙티스→실전 예제로 정리한 내용입니다.

---

## 1) `PreparedStatement` — 파라미터 바인딩과 성능

### 1.1 개념
- **미리 컴파일된 SQL**에 **`?` 자리표시자**로 파라미터를 바인딩하는 문장.
- 장점
  - **SQL 인젝션 방지** (문자열 연결 금지)
  - **성능**: 문장 재사용 시 DB가 파싱/플랜 캐시를 재사용
  - **타입 안전**: 드라이버가 JDBC 타입 ↔ DB 타입 변환

### 1.2 기본 사용
```java
String sql = "SELECT id, name, email FROM users WHERE status = ? AND created_at >= ?";
try (PreparedStatement ps = conn.prepareStatement(sql)) {
    ps.setString(1, "ACTIVE");             // 인덱스는 1부터 시작
    ps.setTimestamp(2, Timestamp.valueOf("2025-01-01 00:00:00"));
    try (ResultSet rs = ps.executeQuery()) { /* ... */ }
}
```

### 1.3 주요 바인딩 메서드
- 원시/래퍼: `setInt`, `setLong`, `setDouble`, `setBoolean`, `setBigDecimal`…
- 문자/이진: `setString`, `setBytes`, `setBinaryStream`, `setCharacterStream`
- 날짜/시간: `setDate`, `setTime`, `setTimestamp`, (`setObject(LocalDate/LocalDateTime)`도 지원하는 드라이버 多)
- **NULL**: `setNull(index, java.sql.Types.INTEGER)` 처럼 **타입 명시**
- 재사용: `clearParameters()`로 파라미터 초기화 후 다시 `setXxx` 가능

### 1.4 배치(Batch) 업데이트
- 대량 INSERT/UPDATE에 효과적. 트랜잭션과 함께 사용 권장.
```java
String sql = "INSERT INTO users(name, email) VALUES(?, ?)";
try (PreparedStatement ps = conn.prepareStatement(sql)) {
    for (var u : users) {
        ps.setString(1, u.getName());
        ps.setString(2, u.getEmail());
        ps.addBatch();
    }
    int[] counts = ps.executeBatch();
}
```

### 1.5 생성된 키 받기 (AUTO_INCREMENT/시퀀스)
```java
String sql = "INSERT INTO users(name, email) VALUES(?, ?)";
try (PreparedStatement ps = conn.prepareStatement(sql, java.sql.Statement.RETURN_GENERATED_KEYS)) {
    ps.setString(1, "Alice");
    ps.setString(2, "a@ex.com");
    ps.executeUpdate();
    try (ResultSet keys = ps.getGeneratedKeys()) {
        if (keys.next()) {
            long newId = keys.getLong(1);
        }
    }
}
```

> 팁: 일부 DB/드라이버는 **배치 + 생성키** 동시 사용에 제한이 있으니 문서 확인.

---

## 2) `ResultSet` — 결과 순회, 타입 매핑, 커서

### 2.1 기본 순회/추출
```java
try (ResultSet rs = ps.executeQuery()) {
    while (rs.next()) {
        long id    = rs.getLong("id");     // 이름 또는 1-base 인덱스 사용 가능
        String nm  = rs.getString("name");
        String em  = rs.getString(3);
        if (rs.wasNull()) { /* 직전 값이 NULL인지 확인 */ }
    }
}
```

### 2.2 타입 매핑 포인트
- `getXxx("col")` 또는 `getXxx(index)`
- `getObject("col", LocalDate.class)`처럼 **java.time** 매핑 지원(드라이버 의존)
- **NULL 처리**  
  - 원시형 `getInt` 등은 0과 구분 어려움 → `wasNull()` 또는 래퍼 `getObject(Integer.class)` 사용
- BLOB/CLOB은 스트리밍으로 처리: `getBinaryStream`, `getCharacterStream`

### 2.3 커서/동시성/홀더빌리티
`Statement`/`PreparedStatement` 생성 시 지정:
```java
PreparedStatement ps = conn.prepareStatement(
    sql,
    ResultSet.TYPE_FORWARD_ONLY,         // 또는 TYPE_SCROLL_INSENSITIVE / SENSITIVE
    ResultSet.CONCUR_READ_ONLY           // 또는 CONCUR_UPDATABLE
);
// 커밋 시 커서 유지 여부
conn.setHoldability(ResultSet.HOLD_CURSORS_OVER_COMMIT);
```
- **TYPE_FORWARD_ONLY**: 앞으로만 이동(기본, 가장 빠름)
- **SCROLL_INSENSITIVE**: 스크롤 가능, 다른 트랜잭션 변경 미반영
- **SCROLL_SENSITIVE**: 스크롤 가능, 변경 반영(드라이버 지원 여부 확인)
- **CONCUR_UPDATABLE**: `rs.updateXxx(...); rs.updateRow();` 같은 **업데이트 가능한 ResultSet**

### 2.4 성능 옵션
- **페치 사이즈**: 대량 조회 시 네트워크 왕복 감소
```java
ps.setFetchSize(500);         // 드라이버/DB별 의미 다를 수 있음
```
- **최대 행수/쿼리 타임아웃**
```java
ps.setMaxRows(10_000);
ps.setQueryTimeout(30);       // 초 단위
```

---

## 3) 트랜잭션 처리 — 원자성/일관성 보장

### 3.1 기본 흐름
- JDBC 기본은 **auto-commit = true** (각 SQL이 자동 커밋)
- **여러 SQL을 하나의 단위**로 묶으려면:
```java
conn.setAutoCommit(false);
try {
    // 다수 DML/쿼리
    // ...
    conn.commit();                    // 성공 시 커밋
} catch (SQLException e) {
    conn.rollback();                  // 실패 시 롤백
    throw e;
} finally {
    conn.setAutoCommit(true);         // 원복(풀 반환 전 권장)
}
```

### 3.2 격리 수준 (Isolation Levels)
```java
conn.setTransactionIsolation(Connection.TRANSACTION_READ_COMMITTED);
```
| 상수 | 의미 | 현상 방지 |
|---|---|---|
| `READ_UNCOMMITTED` | 가장 낮음 | 없음(Dirty Read 허용) |
| `READ_COMMITTED`   | 커밋된 것만 읽음 | Dirty Read 방지 |
| `REPEATABLE_READ`  | 반복 조회 일관 | + Non-Repeatable Read 방지 |
| `SERIALIZABLE`     | 가장 높음 | + Phantom Read 방지 |

> 실제 동작은 DBMS에 따라 차이. 기본값은 보통 `READ_COMMITTED`(예: PostgreSQL, Oracle).

### 3.3 세이브포인트 (부분 롤백)
```java
conn.setAutoCommit(false);
try {
    // op1
    Savepoint sp = conn.setSavepoint();
    // op2
    if (somethingWrong) conn.rollback(sp); // op2만 되돌림
    conn.commit();
} catch (SQLException e) {
    conn.rollback();
}
```

### 3.4 커밋/롤백 시 주의
- `ResultSet`은 커밋/롤백 후 **닫힐 수 있음**(holdability/드라이버 설정에 좌우)
- 긴 트랜잭션은 **락 경합**과 **타임아웃**을 유발 → 가능한 **짧게** 유지

---

## 4) 실전 예제

### 4.1 조회: DTO 매핑 & 자원 관리 (try-with-resources)
```java
public record User(long id, String name, String email) {}

public List<User> findActiveUsers(Connection conn) throws SQLException {
    String sql = """
        SELECT id, name, email
        FROM users
        WHERE status = ? AND created_at >= ?
        ORDER BY id
    """;
    List<User> out = new ArrayList<>();
    try (PreparedStatement ps = conn.prepareStatement(sql)) {
        ps.setString(1, "ACTIVE");
        ps.setTimestamp(2, Timestamp.valueOf("2025-01-01 00:00:00"));
        ps.setFetchSize(500);
        try (ResultSet rs = ps.executeQuery()) {
            while (rs.next()) {
                out.add(new User(
                    rs.getLong("id"),
                    rs.getString("name"),
                    rs.getString("email")
                ));
            }
        }
    }
    return out;
}
```

### 4.2 트랜잭션 + INSERT + 생성키 회수
```java
public long createUser(Connection conn, String name, String email) throws SQLException {
    String sql = "INSERT INTO users(name, email, status) VALUES(?, ?, 'ACTIVE')";
    boolean old = conn.getAutoCommit();
    conn.setAutoCommit(false);
    try (PreparedStatement ps = conn.prepareStatement(sql, java.sql.Statement.RETURN_GENERATED_KEYS)) {
        ps.setString(1, name);
        ps.setString(2, email);
        ps.executeUpdate();
        long newId;
        try (ResultSet keys = ps.getGeneratedKeys()) {
            if (!keys.next()) throw new SQLException("No generated key");
            newId = keys.getLong(1);
        }
        // 예: 다른 테이블에도 기록
        String logSql = "INSERT INTO audit(user_id, action) VALUES(?, ?)";
        try (PreparedStatement ps2 = conn.prepareStatement(logSql)) {
            ps2.setLong(1, newId);
            ps2.setString(2, "CREATE");
            ps2.executeUpdate();
        }
        conn.commit();
        return newId;
    } catch (SQLException e) {
        conn.rollback();
        throw e;
    } finally {
        conn.setAutoCommit(old);
    }
}
```

### 4.3 배치 + 트랜잭션 (대량 처리)
```java
public int[] bulkInsertUsers(Connection conn, List<User> users) throws SQLException {
    String sql = "INSERT INTO users(name, email) VALUES(?, ?)";
    boolean old = conn.getAutoCommit();
    conn.setAutoCommit(false);
    try (PreparedStatement ps = conn.prepareStatement(sql)) {
        int batch = 0;
        for (User u : users) {
            ps.setString(1, u.name());
            ps.setString(2, u.email());
            ps.addBatch();
            if (++batch % 1000 == 0) ps.executeBatch(); // 청크 실행
        }
        int[] res = ps.executeBatch();
        conn.commit();
        return res;
    } catch (SQLException e) {
        conn.rollback();
        throw e;
    } finally {
        conn.setAutoCommit(old);
    }
}
```

---

## 5) 베스트 프랙티스 체크리스트

- **안전성**
  - [ ] **문자열 연결 금지** → 반드시 `PreparedStatement` 사용
  - [ ] NULL은 `setNull(idx, Types.XXX)`로 명시
  - [ ] 재사용 시 `clearParameters()`로 초기화
  - [ ] `PreparedStatement`/`ResultSet`은 **스레드-세이프 아님**(한 스레드에서만 사용)

- **성능**
  - [ ] 대량 작업: **배치 + 트랜잭션**
  - [ ] 대량 조회: **`setFetchSize`**/스트리밍 사용
  - [ ] 자주 쓰는 문장은 같은 커넥션 안에서 **재사용**

- **트랜잭션**
  - [ ] 필요한 범위에서만 `setAutoCommit(false)` → **짧고 명확하게**
  - [ ] 실패 시 `rollback()` 확실히, `finally`에서 `autoCommit` 복구
  - [ ] 격리 수준은 비즈니스 요건과 DB 비용을 고려해 선택

- **리소스 관리**
  - [ ] 항상 **try-with-resources**로 `ResultSet` → `Statement` → `Connection` 순서로 닫힘 보장
  - [ ] 커넥션 풀(`DataSource`) 사용(HikariCP 등), `DriverManager` 직사용 최소화

- **설계**
  - [ ] SQL은 상수/리포지토리 계층으로 분리, 매핑 로직을 메서드로 캡슐화
  - [ ] `IN (?)` 동적 리스트는 자리표시자 확장(`IN (?,?,?)`) 유틸로 처리

---

### 마무리
`PreparedStatement`는 **보안과 성능의 기본**, `ResultSet`은 **정확한 타입/NULL 처리**가 핵심입니다.  
트랜잭션은 **짧고 명확하게** 관리하고, 대량 처리에는 **배치**를 결합하세요.  
이 원칙만 지켜도 대부분의 JDBC 코드가 **안전·빠름·유지보수 용이**한 수준을 달성합니다.