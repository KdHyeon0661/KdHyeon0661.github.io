---
layout: post
title: Java - throw vs throws
date: 2025-07-29 14:20:23 +0900
category: Java
---
# `throw` vs `throws`

## 0. 한눈에 보는 차이

| 키워드 | 핵심 의미 | 쓰는 곳 | 역할 |
|---|---|---|---|
| `throw` | **예외 객체를 즉시 발생** | 메서드/블록 **내부** | 런타임 시점에 예외를 던짐 |
| `throws` | **이 메서드가 예외를 전파 가능** | **메서드/생성자 선언부** | 컴파일 시점에 호출자에게 “처리/선언” 요구(Checked에 대해) |

---

## 1. `throw` — 예외를 “발생”시키는 문(statement)

### 문법
```java
throw new ExceptionType("메시지");  // new로 생성한 Throwable 하위 타입만 허용
```

### 특징 요약
- 실행되는 그 지점에서 **제어 흐름을 끊고** 예외를 상위 스택으로 전파.
- 한 번에 **단 하나의** 예외 객체만 던질 수 있음.
- `throw null;` → 즉시 `NullPointerException` (컴파일 OK, 런타임 NPE).
- `throw` 뒤 코드는 **도달 불가**로 간주(컴파일 오류 가능).

### 기본 예제
```java
public class ThrowExample {
    static int normalizeAge(int age) {
        if (age < 0) {
            throw new IllegalArgumentException("나이는 음수가 될 수 없습니다: " + age);
        }
        return age;
    }
    public static void main(String[] args) {
        normalizeAge(-1); // IllegalArgumentException 발생
    }
}
```

---

## 2. `throws` — 예외를 “선언”하는 시그니처

### 문법
```java
public void readFile(Path p) throws IOException { ... }
public MyService() throws ConfigException { ... } // 생성자에도 가능
```

### 특징 요약
- **Checked** 예외가 전파될 수 있음을 **호출자에게** 알리는 계약(contract).
- 쉼표로 **여러 타입**을 선언 가능: `throws IOException, ParseException`.
- **Unchecked(RuntimeException 하위)** 는 선언 의무 없음(선언해도 무방).

### 기본 예제
```java
public class ThrowsExample {
    public static void main(String[] args) {
        try {
            readFirstLine(Path.of("nope.txt"));
        } catch (IOException e) {
            System.out.println("파일 읽기 실패: " + e.getMessage());
        }
    }
    static String readFirstLine(Path p) throws IOException {
        try (var br = java.nio.file.Files.newBufferedReader(p)) {
            return br.readLine();
        }
    }
}
```

---

## 3. Checked vs Unchecked와의 관계

| 구분 | 대표 타입 | `throws` 필요? | `throw` 사용 |
|---|---|---|---|
| **Checked** | `IOException`, `SQLException`, `ReflectiveOperationException` 등 | **필요**(catch 또는 throws) | 가능 |
| **Unchecked** | `RuntimeException` 하위(`IllegalArgumentException`, `NullPointerException` 등) | 선택(생략 가능) | 가능 |
| **Error** | `OutOfMemoryError`, `StackOverflowError` 등 | 선언 불필요(잡지 말 것) | 가능(권장X) |

> **원칙**: 호출자가 **현실적으로 복구** 가능하면 Checked, 호출자의 **버그**나 계약 위반이면 Unchecked.

---

## 4. `throw`와 `throws`를 함께 쓰는 전형 패턴

```java
// 선언: 이 메서드는 AgeException을 전파할 수 있음
public static void validate(int age) throws AgeException {
    if (age < 0) {                  // 발생: 조건 위반 → 예외 던짐
        throw new AgeException("음수 나이: " + age);
    }
}
```

사용자 정의 예외:
```java
public class AgeException extends Exception {
    public AgeException(String msg) { super(msg); }
}
```

---

## 5. 재던지기(rethrow)·전환(translation)·체이닝(chaining)

### 5.1 있는 그대로 재던지기
```java
try {
    doIo();
} catch (IOException e) {
    // 로깅 후 그대로
    System.err.println("I/O 실패: " + e.getMessage());
    throw e; // 스택트레이스 보존
}
```

### 5.2 예외 **전환**(Checked → Unchecked 또는 도메인 예외로 감싸기)
```java
try {
    doIo();
} catch (IOException e) {
    throw new RuntimeException("설정 로드 실패", e); // cause로 체인 보존
}
```

### 5.3 다중 캐치 + 정밀 재던지기(precise rethrow, Java 7+)
```java
try {
    mayThrow();
} catch (IOException | SQLException e) {
    // 여기서 그냥 throw e; 하면 컴파일러가 정확한 타입 추론 가능
    throw e;
}
```

**지침**
- **원인 보존**: 새로 감쌀 땐 **반드시 cause** 전달(`new X(msg, e)` 또는 `initCause(e)`).
- **스택 손실 금지**: `throw new ...`로 갈아치우면 원래 스택이 사라짐(원인 체인으로 대체).

---

## 6. 오버라이딩 시 `throws` 규칙 (중요!)

- **하위(오버라이딩) 메서드**는 상위 메서드보다 **넓은 Checked 예외**를 **새로 추가할 수 없음**.
- 하위는 **더 적거나, 더 좁은(하위 타입)** Checked 예외만 선언 가능.
- Unchecked는 자유롭게 던져도 선언 의무 없음.

```java
class A {
    void run() throws IOException {}
}
class B extends A {
    @Override
    void run() throws FileNotFoundException { } // ✓ 좁힘(하위 타입)
    // void run() throws Exception { } // ✗ 더 넓음 → 컴파일 오류
}
```

**생성자**
- 상위 생성자가 던지는 Checked 예외는 **하위 생성자에서도 처리(try-catch)하거나 선언(throws)** 해야 함.

---

## 7. `try-with-resources`와 suppressed 예외

- 본문이 `throw`로 실패하고, 리소스 `close()`에서도 예외가 나면, **close 예외는 suppressed**로 붙음.
- 반대 상황(본문은 정상, `close()`에서 실패)이면 **close 예외가 던져지고** 본문은 성공으로 간주.

```java
try (var br = java.nio.file.Files.newBufferedReader(Path.of("x"))) {
    throw new IllegalStateException("본문 실패");
} catch (Exception e) {
    for (Throwable s : e.getSuppressed()) {
        System.err.println("suppressed: " + s);
    }
    throw e;
}
```

**수동 추가**
```java
var primary = new IOException("메인 실패");
primary.addSuppressed(new IOException("정리 중 실패"));
throw primary;
```

---

## 8. `finally`와 `throw`의 미묘한 상호작용 (주의!)

- `finally`에서 **`return`을 하면** 앞서 던진 예외가 **소거**될 수 있음(절대 금지).
- `finally`에서 **새 예외를 던지면** 원래 예외가 사라지고 새 예외만 보임(필요 시 suppressed로 연결).

```java
try {
    throw new IOException("A");
} finally {
    // return 0;           // ✗ A가 사라짐
    // throw new RuntimeException("B"); // A 사라지고 B만 남음
}
```

---

## 9. 인터럽트/동시성과 예외 (실무 필수)

- 차단 메서드(`sleep`, `wait`, `join`)는 **`InterruptedException`(Checked)** 을 던짐 →
  **메서드 시그니처에 `throws` 하거나, catch 후 `Thread.currentThread().interrupt()`로 복원**.

```java
void doWork() throws InterruptedException {
    Thread.sleep(200); // throws 선언로 전파
}

void loop() {
    try {
        while (!Thread.currentThread().isInterrupted()) {
            // ...
            Thread.sleep(100);
        }
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt(); // 인터럽트 상태 복원 후 종료
    }
}
```

> 인터럽트를 **무시하거나 삼키지 말 것** — 종료 불가/응답 불가 상태 초래.

---

## 10. 람다·메서드 참조와 Checked 예외

- `Runnable.run()`은 `throws`가 없으므로 **람다 내부에서 Checked 예외를 직접 던질 수 없음**.
  → **전환(wrap)** 또는 **사용자 정의 함수형 인터페이스**로 `throws`를 선언.

```java
// 1) 전환
ExecutorService ex = java.util.concurrent.Executors.newSingleThreadExecutor();
ex.submit(() -> {
    try { doIo(); }
    catch (IOException e) { throw new RuntimeException(e); }
});

// 2) throws 보유 인터페이스 정의
@FunctionalInterface interface IoTask { void run() throws IOException; }
static Runnable uncheck(IoTask t) {
    return () -> { try { t.run(); } catch (IOException e) { throw new RuntimeException(e); } };
}
ex.submit(uncheck(() -> doIo()));
```

---

## 11. 실무 예외 설계 지침

### 11.1 언제 Checked / Unchecked?
- **Checked**: 호출자가 **복구/대체경로**가 명확(재시도, 다른 리소스 선택 등).
- **Unchecked**: 호출자 **버그** 또는 **불변식 위반**(잘못된 인수, 부적절한 상태).

### 11.2 예외 전환 & 도메인 예외
- 외부 계층(드라이버/네트워크) → **도메인 특정 예외**로 감싸서 상위 계층에 의미 전달.
- 항상 `cause` 체인 보존.

### 11.3 메시지·타입
- **구체적 타입** 사용(일반 `Exception`/`RuntimeException` 남발 금지).
- 메시지는 **행동 지침**을 포함(무엇을 할지).

### 11.4 로그·모니터링
- 잡은 예외는 **의도적으로 무시**하지 말고, 로그 레벨/메트릭을 정책화.

---

## 12. 성능 팁과 안티패턴

- 예외 생성은 **스택트레이스 채움**(비용 큼). **정상 흐름 제어**에 예외를 쓰지 말 것.
- 고빈도 경로에서 예외가 예상되면 **사전 검증**(예: `Files.exists`, preconditions).
- 대량 루프에서 `throw` 반복은 비용 큼. **계약 위반을 조용히 무시하지 말되** 경로 설계를 재검토.

고급: 스택트레이스 작성 억제(특수 상황)
```java
// Throwable(String message, Throwable cause, boolean enableSuppression, boolean writableStackTrace)
throw new MyFastException("msg", null, true, false); // 스택 비활성화(매우 제한적으로)
```

---

## 13. 자주 틀리는 포인트(FAQ)

- **Q. `throws`는 클래스에 쓸 수 있나요?** → **아니요.** 메서드/생성자에만.
- **Q. `main`에 `throws Exception` 써도 되나요?** → 가능. JVM까지 전파되어 스택 출력 후 종료.
- **Q. `throw`는 if 없이 쓸 수 있나요?** → 가능. 조건부일 필요 없음.
- **Q. `catch`에서 새 예외를 만들면 원래 스택은요?** → cause로 넘기지 않으면 **유실**.
- **Q. Lombok `@SneakyThrows` 써도 되나요?** → “스니키 스로우”는 Checked를 숨겨 API 계약을 흐림. **권장하지 않음**(제한적으로만).

---

## 14. 예제 모음

### 14.1 검증(Validation) 가드
```java
public static void requireNonEmpty(String s) {
    if (s == null || s.isEmpty())
        throw new IllegalArgumentException("빈 문자열은 허용되지 않습니다");
}
```

### 14.2 I/O 래핑(전환)
```java
String load(Path p) {
    try { return java.nio.file.Files.readString(p); }
    catch (IOException e) {
        throw new ConfigLoadException("설정 읽기 실패: " + p, e); // Unchecked 도메인 예외
    }
}
class ConfigLoadException extends RuntimeException {
    public ConfigLoadException(String m, Throwable c) { super(m, c); }
}
```

### 14.3 precise rethrow + 로깅
```java
void service() throws IOException, SQLException {
    try {
        repo.call(); // IOException or SQLException
    } catch (IOException | SQLException e) {
        logger.warn("저장소 호출 실패: {}", e.getMessage());
        throw e; // 선언된 그대로 재던지기
    }
}
```

### 14.4 인터럽트 전파 (Checked throws)
```java
void timedWait(CountDownLatch latch) throws InterruptedException {
    if (!latch.await(1, java.util.concurrent.TimeUnit.SECONDS)) {
        throw new IllegalStateException("타임아웃");
    }
}
```

### 14.5 finally에서 예외 보존
```java
try {
    doMain();
} catch (Exception primary) {
    try { cleanup(); }
    catch (Exception cleanup) { primary.addSuppressed(cleanup); }
    throw primary;
}
```

---

## 15. 요약 표

| 항목 | `throw` | `throws` |
|---|---|---|
| 역할 | 예외 **발생** | 예외 **전파 가능성 선언** |
| 위치 | 메서드/블록 내부 | 메서드/생성자 시그니처 |
| 시점 | **런타임** 제어 이전 | **컴파일** 계약 확인(특히 Checked) |
| 개수 | 한 번에 **하나** | 쉼표로 **여러 타입** |
| 대상 | 예외 **객체** 필요 (`new`) | 예외 **타입**만 표시 |
| 함께 쓰기 | 내부에서 `throw` | 시그니처에 `throws` |

---

## 16. 실전 체크리스트

- [ ] 복구 가능? → **Checked**로 `throws` 또는 `try-catch`.
- [ ] 계약 위반/버그? → **Unchecked**로 `throw`.
- [ ] 새 예외로 감쌀 때 **cause 체인 유지**.
- [ ] `finally`에서 `return/throw`로 **원래 예외 덮지 않기**(필요 시 `addSuppressed`).
- [ ] 람다에서 Checked 필요 시 **전환** 또는 **throws 포함 함수형 인터페이스**.
- [ ] 인터럽트는 **전파하거나 복원**.
- [ ] 오버라이드 시 Checked `throws`는 **좁히기만** 가능.

---

## 부록) 미세 팁

- `initCause`는 **한 번만** 가능(이미 cause가 있으면 `IllegalStateException`).
- 예외는 `Serializable` — 분산/원격 호출에서 전송됨(필드 주의).
- `Objects.requireNonNull(x, "msg")`는 **즉시 NPE**를 `throw`하는 유틸.

---
```java
// 마지막 한 방 예제: 선언(throws), 발생(throw), 전환, suppressed, 인터럽트 복원까지
class Endgame {
    static void doIo() throws IOException { throw new IOException("디스크 오류"); }

    static void runJob() {
        try (var r = new java.io.BufferedReader(new java.io.StringReader("hi"))) {
            doIo();                       // Checked 예외 발생 가능
        } catch (IOException e) {          // 선언 or 전환
            RuntimeException wrapped = new RuntimeException("작업 실패", e);
            // 정리 중 추가 오류를 suppressed로 보존(예시)
            wrapped.addSuppressed(new IllegalStateException("추가 정보"));
            throw wrapped;
        } catch (RuntimeException re) {    // Unchecked는 자유롭게
            throw re;
        }
    }

    static void waitOn(CountDownLatch latch) throws InterruptedException {
        try {
            latch.await();                 // Checked → throws로 전파
        } catch (InterruptedException ie) {
            Thread.currentThread().interrupt(); // 복원
            throw ie;                      // 상위에 알림
        }
    }
}
```

---

## 결론

- **`throw`는 발생**, **`throws`는 선언** — 용도도, 위치도, 시점도 다르다.
- Checked/Unchecked 구분과 오버라이딩 규칙, try-with-resources의 suppressed, 인터럽트 전파까지 이해하면 **예외 흐름을 설계**할 수 있다.
- “원인 보존(cause) + 의미 있는 타입/메시지 + 적절한 전파(throws)”를 지키면 **진단 가능하고 견고한** 코드를 만들 수 있다.
