---
layout: post
title: Java - Checked vs Unchecked 예외
date: 2025-07-29 15:20:23 +0900
category: Java
---
# Checked vs Unchecked 예외

## 큰 그림 — 개념 요약

| 분류 | 상속 루트 | 컴파일러 강제 | 의도/의미 | 대표 예 |
|---|---|---|---|---|
| **Checked** | `Exception` (단, `RuntimeException` 제외) | **처리**(`try-catch`) **또는 선언**(`throws`) **강제** | 호출자가 **복구 가능**한 상황(I/O, 네트워크, 파일, 외부시스템) | `IOException`, `SQLException`, `ClassNotFoundException` |
| **Unchecked** | `RuntimeException` | 강제 **없음** | **프로그래밍 오류/계약 위반**(잘못된 인수, 상태 위반, NPE) | `NullPointerException`, `IllegalArgumentException`, `IllegalStateException`, `IndexOutOfBoundsException` |
| **Error** | `Error` | 다루지 않음 | JVM/시스템 치명 오류 | `OutOfMemoryError`, `StackOverflowError` |

예외 계층(요약):

```
Throwable
 ├─ Error                  // 시스템 치명 오류, 일반적으로 catch 대상 아님
 └─ Exception
     ├─ RuntimeException   // 언체크
     └─ (Checked)          // 체크드
```

---

## 기본 문법과 동작

### 처리 vs 선언

```java
// Checked 예외: 반드시 처리하거나(try-catch) 선언해야 함
void read() throws IOException {
    try (var br = new java.io.BufferedReader(new java.io.FileReader("in.txt"))) {
        System.out.println(br.readLine());
    }
}

// Unchecked 예외: 선언/처리 강제 없음
int divide(int a, int b) {            // throws 선언 없어도 됨
    return a / b;                     // b==0 → ArithmeticException
}
```

### 다중 catch & 멀티-catch

```java
try {
    mightThrow();
} catch (java.io.IOException | java.sql.SQLException e) {
    // 공통 처리
}
```

### finally와 자원 해제

```java
// 전통 패턴
java.io.BufferedReader br = null;
try {
    br = new java.io.BufferedReader(new java.io.FileReader("in.txt"));
    System.out.println(br.readLine());
} catch (java.io.IOException e) {
    // 처리
} finally {
    if (br != null) try { br.close(); } catch (java.io.IOException ignore) {}
}

// 권장: try-with-resources (Java 7+)
try (var br = new java.io.BufferedReader(new java.io.FileReader("in.txt"))) {
    System.out.println(br.readLine());
}
```

---

## 억제(Suppressed) 예외 — try-with-resources의 숨은 핵심

자원 닫기(`close`) 중 발생한 예외는 **억제(suppressed)** 되어 **주 예외**에 연결됩니다.

```java
class Closer implements AutoCloseable {
    @Override public void close() { throw new RuntimeException("close failed"); }
}

static void mainLogic() {
    try (var c = new Closer()) {
        throw new IllegalStateException("main failed");
    } catch (Exception e) {
        System.out.println(e); // IllegalStateException: main failed (주 예외)
        for (Throwable s : e.getSuppressed()) {
            System.out.println("suppressed: " + s); // close failed
        }
    }
}
```

> finally에서 직접 `close()`를 호출하며 새 예외를 던지면 **원래 예외가 유실**될 수 있습니다. 억제 예외를 자동으로 관리하는 **try-with-resources**를 사용하세요.

---

## 오버라이딩과 `throws` 제약

- 하위 클래스는 **상위 메서드가 선언한 checked 예외보다 더 넓은**(상위 타입의) **checked 예외**를 **던질 수 없음**
- 더 **좁히거나(하위 타입)**, **unchecked만 던지는 것**은 허용

```java
class Base {
    void work() throws java.io.IOException {}
}
class Child extends Base {
    @Override void work() /* throws Exception */ { /* 컴파일 오류: 더 넓어짐 */ }
    @Override void work() /* OK: throws 없음 */ { }
}
```

---

## 사용자 정의 예외 — Checked vs Unchecked

```java
// Checked: 호출자가 복구/대응해야 하는 시나리오
public class InsufficientFundsException extends Exception {
    public InsufficientFundsException(String msg) { super(msg); }
}

// Unchecked: 잘못된 사용/상태 위반
public class DomainInvariantViolationException extends RuntimeException {
    public DomainInvariantViolationException(String msg) { super(msg); }
}
```

> 규칙: **복구 가능성**이 있다면 Checked. **코드 버그/계약 위반**이면 Unchecked.

---

## 예외 전환(번역)과 체이닝 — 원인 보존이 생명

### Checked → Unchecked (API 부담 줄이기)

```java
String readSilently(java.nio.file.Path path) {
    try {
        return java.nio.file.Files.readString(path);
    } catch (java.io.IOException e) {
        throw new RuntimeException("읽기 실패: " + path, e); // cause로 포장
    }
}
```

### Unchecked → 도메인 예외(명시적 의미화)

```java
try {
    repo.save(order);
} catch (RuntimeException e) {
    throw new OrderPersistenceException("주문 저장 실패", e); // 도메인 의미 부여
}
```

> **체이닝 필수**: 항상 `new XxxException(message, cause)`로 **원인 예외를 보존**하세요.

---

## 람다/스트림에서 Checked 예외 다루기

스트림 연산의 함수형 인터페이스는 **checked 예외를 선언하지 않음**. 처리 방법:

### 래핑 유틸리티로 변환

```java
@FunctionalInterface interface ThrowingFunction<T, R, E extends Exception> {
    R apply(T t) throws E;
}

static <T, R> java.util.function.Function<T, R>
wrap(ThrowingFunction<T, R, ?> f) {
    return t -> {
        try { return f.apply(t); }
        catch (Exception e) { throw new RuntimeException(e); } // 전환
    };
}

// 사용
var lines = java.util.stream.Stream.of("a.txt", "b.txt")
    .map(wrap(java.nio.file.Files::readString))
    .toList();
```

### 개별 try-catch로 매핑

```java
var list = files.stream().map(p -> {
    try {
        return java.nio.file.Files.readString(p);
    } catch (java.io.IOException e) {
        return "ERROR:" + p; // 대체값 매핑
    }
}).toList();
```

> 팀 규약에 따라 7.1(전환) 또는 7.2(대체값) 전략을 정해 일관성 유지.

---

## `CompletableFuture`와 예외

- 비동기 작업의 예외는 **CompletionException**(Unchecked)으로 래핑되어 전파
- `join()`은 언체크 예외로, `get()`은 체크드(`ExecutionException`)로 노출

```java
var f = java.util.concurrent.CompletableFuture.supplyAsync(() -> {
    if (true) throw new IllegalArgumentException("boom");
    return 1;
});
try {
    f.join(); // CompletionException(IllegalArgumentException) throw
} catch (java.util.concurrent.CompletionException e) {
    System.out.println("cause: " + e.getCause()); // 원인 접근
}
```

핸들링 메서드:

```java
f.exceptionally(ex -> { log(ex); return -1; });
f.handle((val, ex) -> ex == null ? val : fallback());
```

---

## 레이어드 아키텍처 — 예외 매핑 가이드

| 레이어 | 권장 예외 | 비고 |
|---|---|---|
| **Infra**(DB, 파일, 외부 API) | **Checked**(I/O/SQL) 내부 처리 후 **도메인/Unchecked로 전환** | 인프라 상세를 상위에 누출하지 않기 |
| **Repository/DAO** | 도메인 특정 `DataAccess`(Unchecked) | 상위 서비스는 인프라 종속 제거 |
| **Service** | 도메인 `BusinessException`(Unchecked/Checked) | 정책상 복구 가능 여부 판단 |
| **Controller/API** | 상위로 전파 → 글로벌 핸들러에서 **HTTP 상태 매핑** | 예: 400/404/409/500 |

Spring 예시(컨트롤러 어드바이스):

```java
@org.springframework.web.bind.annotation.ControllerAdvice
class GlobalHandler {
    @org.springframework.web.bind.annotation.ExceptionHandler(NotFoundException.class)
    org.springframework.http.ResponseEntity<?> handle(NotFoundException e) {
        return org.springframework.http.ResponseEntity.status(404).body(e.getMessage());
    }
}
```

---

## 베스트 프랙티스 체크리스트

- [ ] **무의미한 `catch (Exception e) {}` 금지** — 로그/대응/전달 중 하나는 반드시
- [ ] **도메인 의미가 드러나는 예외 타입** 설계(메시지에 원인/키값 포함)
- [ ] **예외로 흐름 제어 금지**(빈번한 정상 로직에 예외 사용 X)
- [ ] **try-with-resources**로 자원 누수 방지 + 억제 예외 보존
- [ ] **체이닝 필수**: 예외 전환 시 `cause` 연결
- [ ] **문서화**: Checked는 `@throws`, Unchecked도 Javadoc에 기재
- [ ] **오버라이딩 throws 확대 금지**(컴파일 규칙 숙지)
- [ ] **`Throwable`/`Error` 잡지 않기** (필요시 매우 제한적)

---

## 실전 예제 모음

### 체크드 처리 + 복구/대체 플로우

```java
String readWithFallback(java.nio.file.Path p, String fallback) {
    try {
        return java.nio.file.Files.readString(p);
    } catch (java.io.IOException e) {
        // 복구/대체 경로
        return fallback;
    }
}
```

### 체크드 → 도메인 언체크 전환(체이닝)

```java
class DataAccessException extends RuntimeException {
    public DataAccessException(String message, Throwable cause) { super(message, cause); }
}
String loadUser(long id) {
    try {
        return db.fetchUser(id); // throws SQLException
    } catch (java.sql.SQLException e) {
        throw new DataAccessException("사용자 로드 실패 id=" + id, e);
    }
}
```

### 억제 예외 관찰

```java
try (var in = java.nio.file.Files.newInputStream(java.nio.file.Path.of("x"))) {
    throw new IllegalStateException("main");
} catch (Exception e) {
    for (Throwable s : e.getSuppressed()) System.out.println(s);
}
```

### 정확한 재-throw(precise rethrow, Java 7+)

```java
static void rethrow() throws java.io.IOException, java.sql.SQLException {
    try {
        mayThrow();
    } catch (Exception e) {
        throw e; // 컴파일러가 정적 분석으로 실제 선언된 예외만 허용
    }
}
```

---

## 안티패턴 예시와 개선

### 예외 삼키기

```java
try { risky(); } catch (Exception e) { /* 침묵 */ } // ❌
```

**개선**: 최소한 로깅/전달/대체

```java
try { risky(); } catch (SpecificException e) {
    log.warn("실패: {}", e.getMessage(), e); // 혹은 전환
    throw new DomainException("작업 실패", e);
}
```

### finally에서 주 예외 유실

```java
try {
    throw new IllegalStateException();
} finally {
    throw new RuntimeException(); // ❌ 주 예외가 덮여버림
}
```

**개선**: try-with-resources 사용 또는 finally에서 새 예외 던지지 않기.

---

## 성능과 테스트 관점

- 예외 생성/스택트레이스 채우기는 **비용이 큼** → 빈번한 정상 흐름에 사용 금지
- 단위 테스트에서 **예외 발생 조건/메시지/원인 체이닝/억제 예외**까지 검증

```java
org.junit.jupiter.api.Test
void testCause() {
    var ex = org.junit.jupiter.api.Assertions.assertThrows(
        DataAccessException.class, () -> loadUser(1)
    );
    org.assertj.core.api.Assertions.assertThat(ex).hasCauseInstanceOf(java.sql.SQLException.class);
}
```

---

## 최종 정리(선택 가이드)

1) **복구 가능성**이 있고 호출자가 결정/대응해야 한다 → **Checked**
2) **계약 위반/버그/잘못된 사용** → **Unchecked**
3) 내부(인프라) 예외는 **의미 있는 도메인 예외로 전환**하고 **체이닝**
4) 자원은 **try-with-resources**로 관리, 억제 예외를 보존
5) API 문서에 **던질 수 있는 예외**(특히 Unchecked도) 명확히 기술

---

## 부록: 미니 예제 프로젝트(메인 포함)

```java
import java.io.*;
import java.nio.file.*;
import java.util.concurrent.*;

class App {

    // Checked 처리 + 전환
    static String readFileOrThrow(Path p) {
        try { return Files.readString(p); }
        catch (IOException e) { throw new RuntimeException("IO 실패: " + p, e); }
    }

    // 스트림 래핑
    @FunctionalInterface interface TF<T,R,E extends Exception> { R apply(T t) throws E; }
    static <T,R> java.util.function.Function<T,R> wrap(TF<T,R,?> f) {
        return t -> { try { return f.apply(t); } catch (Exception e) { throw new RuntimeException(e); } };
    }

    static void suppressedDemo() {
        class C implements AutoCloseable {
            @Override public void close() { throw new RuntimeException("close"); }
        }
        try (var c = new C()) {
            throw new IllegalStateException("main");
        } catch (Exception e) {
            System.out.println("main: " + e);
            for (var s : e.getSuppressed()) System.out.println("suppressed: " + s);
        }
    }

    static void completableDemo() {
        var f = CompletableFuture.supplyAsync(() -> { throw new IllegalArgumentException("boom"); });
        try { f.join(); }
        catch (CompletionException e) { System.out.println("cause: " + e.getCause()); }
    }

    public static void main(String[] args) {
        // 1) Checked → Unchecked 전환
        try { System.out.println(readFileOrThrow(Path.of("nope"))); }
        catch (RuntimeException e) { System.out.println("wrapped: " + e.getCause()); }

        // 2) 람다에서 checked 다루기
        var out = java.util.stream.Stream.of("a", "b")
            .map(wrap(Files::createTempFile)) // IOException → RuntimeException
            .toList();
        System.out.println(out);

        // 3) 억제 예외 관찰
        suppressedDemo();

        // 4) CompletableFuture 예외
        completableDemo();
    }
}
```

---

### 마무리

Checked/Unchecked의 선택은 **컴파일러 강제를 통한 사용성**과 **실제 복구 가능성**의 균형입니다. **체이닝으로 원인을 보존**하고, **try-with-resources로 억제 예외까지 안전하게 관리**하며, **레벨별 예외 매핑**을 통해 의미 있는 에러 경로를 유지하세요. 이 가이드를 팀 규약으로 삼으면, 예외는 더 이상 골칫거리가 아니라 **정확한 실패 소통 도구**가 됩니다.
