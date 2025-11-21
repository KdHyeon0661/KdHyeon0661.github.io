---
layout: post
title: Java - try-catch-finally
date: 2025-07-24 23:20:23 +0900
category: Java
---
# 자바 `try-catch-finally`/`try-with-resources`

---

## 예외 모델과 `try-catch-finally`의 기본

### 예외 계층과 분류

| 분류 | 대표 타입 | 의미/권장 사용 |
|---|---|---|
| **Error** | `OutOfMemoryError`, `StackOverflowError` | JVM 레벨 치명 오류. **catch하지 말 것**(복구 불가). |
| **Exception (checked)** | `IOException`, `SQLException` | *컴파일러가 처리 강제*. 외부 세계(I/O, DB)에 대한 **합리적 복구/대안 경로**가 있을 때 사용. |
| **RuntimeException (unchecked)** | `NullPointerException`, `IllegalArgumentException`, `IllegalStateException`, `IndexOutOfBoundsException`, `UnsupportedOperationException` | **프로그래밍 오류/계약 위반**. 호출자 코드 수정이 해법. `throws` 강제 없음. |

> 규칙: **복구 가능성**이 있는 경우 → *checked*, **버그/계약 위반** → *unchecked*.

### 기본 문법(복습)

```java
try {
    // 예외가 발생할 수 있는 코드
} catch (ExceptionType e) {
    // 예외 처리
} finally {
    // 자원 정리(예외 유무와 무관)
}
```

### 제어 흐름 요약

- `try`에서 예외 발생 → **가장 가까운 호환 `catch`**로 이동 → 처리 후 `finally` 실행 → 다음 문장.
- `try`에서 예외 없음 → `catch` 건너뜀 → **`finally` 실행** → 다음 문장.
- `try`/`catch`에서 `return`/`throw`가 있어도 → **`finally`는 실행**(일부 예외적 상황은 §7.4).

---

## `catch` 설계 — 멀티-캐치, 순서, 재던지기

### 캐치 순서(상위 타입은 아래)

```java
try {
    // ...
} catch (NullPointerException e) { /* ... */ }
  catch (RuntimeException e) { /* ... */ }  // 상위는 뒤쪽
```
- 상속 관계에서 **하위 → 상위** 순으로. 그렇지 않으면 도달 불가(컴파일 에러).

### 멀티-캐치(Java 7+)

```java
try {
    // ...
} catch (IOException | SQLException e) {     // '|' 로 묶음
    // e는 암묵적 final → 재할당 불가
}
```

### 정확 재던지기(Precise Rethrow, Java 7+)

- `catch (Exception e)`로 받아도, **조건부로 더 구체적 타입을 다시 던지는** 것이 가능.
```java
static void rethrowing() throws IOException, SQLException {
    try {
        maybeIOorSQL();
    } catch (Exception e) {
        throw e;  // 컴파일러가 e의 정적 정보 추론 → 선언 목록 유지
    }
}
```

### & 래핑(wrapping)

- **레이어 경계**에서 내부 예외를 **도메인 고수준 예외로 변환**.
```java
class UserRepository {
  User find(long id) {
    try {
      return query(id);
    } catch (SQLException e) {
      throw new DataAccessException("쿼리 실패 id=" + id, e); // 원인(cause) 보존
    }
  }
}
class DataAccessException extends RuntimeException {
  DataAccessException(String msg, Throwable cause) { super(msg, cause); }
}
```

> 원칙: **원인 체인**을 유지(`cause` 전파), **이중 로깅 금지**(아래 §8.2).

---

## `finally` — 자원 해제와 함정

### 자원 해제의 정석

```java
BufferedReader br = null;
try {
  br = new BufferedReader(new FileReader("a.txt"));
  System.out.println(br.readLine());
} catch (IOException e) {
  // 처리/번역
} finally {
  try { if (br != null) br.close(); } catch (IOException ignored) {}
}
```

### `finally`에서의 **금기**

1) **`return`/`throw`/`break` 사용 금지**: 원래 예외/반환값을 **가려버림**.
```java
static int bad() {
  try { throw new RuntimeException("원인"); }
  finally { return 42; } // 원래 예외 소실!
}
```
2) **상태 변경/비즈니스 로직 수행** 금지: 해제 외 로직은 다른 곳으로.

### 항상 실행되지 않는 경우(드묾)

- `System.exit`, `Runtime.halt`, JVM 강제 종료/크래시, 무한 루프/전원 차단, **`Error`**로 인한 비정상 중단.

---

## `try-with-resources` — 자동 자원 해제(Java 7+)

### 기본 사용과 close 순서

```java
try (BufferedReader br = new BufferedReader(new FileReader("a.txt"))) {
  System.out.println(br.readLine());
} catch (IOException e) { /*...*/ }
```
- `br`는 `AutoCloseable`이므로 블록 종료 시 **자동으로 `close()`**.
- 다중 자원은 **역순**으로 닫힘.
```java
try (InputStream in = new FileInputStream("in");
     OutputStream out = new FileOutputStream("out")) {
  in.transferTo(out);
}
```

### Java 9: 사전 선언 자원의 TWR

```java
BufferedReader br = new BufferedReader(new FileReader("a.txt"));
try (br) {                    // br이 effectively final 이면 가능
  System.out.println(br.readLine());
}
```

### **Suppressed 예외**와 진짜 원인 보존

- 본문에서 예외 A 발생, `close()`에서 예외 B 발생 → **A가 주 예외**, B는 **suppressed**로 부착.
```java
try (MyRes r = new MyRes()) {
  r.fail();
} catch (Exception e) {
  System.out.println("main: " + e);
  for (Throwable s : e.getSuppressed()) System.out.println("suppressed: " + s);
}

static class MyRes implements AutoCloseable {
  void fail() { throw new RuntimeException("body"); }
  public void close() { throw new RuntimeException("close"); }
}
```
- 출력: `body`가 main 예외, `close`가 suppressed.
- 수동 추가: `e.addSuppressed(suppressedThrowable)`.

### `AutoCloseable` vs `Closeable`

| 항목 | `AutoCloseable` | `Closeable` |
|---|---|---|
| 선언 | `void close() throws Exception` | `void close() throws IOException` |
| 범용성 | 일반 예외 | I/O 특화 |
| 권장 | 새 타입은 `AutoCloseable` | I/O는 여전히 유용 |

---

## 동시성과 예외 — 락 해제, 인터럽트

### `ReentrantLock`은 **반드시 finally에서 unlock**

```java
Lock lock = new ReentrantLock();
lock.lock();
try {
  // 임계영역
} finally {
  lock.unlock();
}
```

### 인터럽트 보존 패턴

```java
try {
  Thread.sleep(1000);
} catch (InterruptedException e) {
  Thread.currentThread().interrupt();  // 인터럽트 상태 복구
  // 필요하다면 상위로 예외 번역/전파
}
```

---

## 체크 vs 언체크 — API 설계 원칙

### 선택 기준

| 질문 | 체크 | 언체크 |
|---|---|---|
| 호출자가 **합리적으로 복구** 가능한가? | ✔ |  |
| **프로그래밍 오류** 또는 **사전조건 위반**인가? |  | ✔ |
| 외부 시스템/환경 의존(I/O, 네트워크, DB)? | ✔ |  |

### 권장 관례

- **입력 검증 실패** → `IllegalArgumentException`
- **상태 위반** → `IllegalStateException`
- **옵션 부재**(API 호출 결과 없음) → `Optional<T>` 반환을 고려
- **존재하지 않는 자원** → 체크 예외(`FileNotFoundException`) 또는 도메인 예외로 번역

---

## 고급 패턴과 함정

### “예외로 제어 흐름” 사용 금지

- 핫패스에서 예외를 **정상 흐름 대체**로 쓰면 성능 저하(스택트레이스 비용 큼).
- 조건/옵셔널/결과 타입으로 처리.

### 로깅 가이드(중복 로그 금지)

- **한 번만** 로깅. 보통 **경계 레이어**(컨트롤러/진입점)에서 로깅하고 내부는 **번역/전파만**.
```java
try {
  service.process();
} catch (AppException e) {
  log.error("요청 실패: {}", reqId, e); // 여기서 1회 로깅
  // → 상위로 또 로깅하지 않기
}
```

### `finally`의 return/throw가 **원인 덮어쓰기**

```java
static void badFinally() {
  try {
    throw new IllegalStateException("원래 예외");
  } finally {
    throw new RuntimeException("finally 덮음"); // 원래 예외 소실
  }
}
```
- **절대 금지**. 해제만 수행.

### `System.exit`/`Error`/프로세스 중단 시 finally 미실행 가능

- 종료 훅/자원 정리는 OS/JVM 종료경로를 별도 설계.

---

## 테스트·디버깅·성능

### 테스트: JUnit의 `assertThrows`

```java
import static org.junit.jupiter.api.Assertions.*;

@Test
void throwsOnBadInput() {
  Exception e = assertThrows(IllegalArgumentException.class,
      () -> parseUser(null));
  assertTrue(e.getMessage().contains("null"));
}
```

### 스택 트레이스 단서

```java
try {
  foo();
} catch (Exception e) {
  e.printStackTrace(); // 개발 단계
  // 실무: 로거 사용 + 샘플링/마스킹
}
```

### 비용 상식

- 예외 **생성/스택 캡처** 비용 큼. 드문 에러 경로엔 OK, **정상 경로엔 금지**.

---

## 종합 예제 모음

### 파일 복사 — `try-with-resources` + 번역 + suppressed 관찰

```java
import java.io.*;
import java.nio.file.*;
import java.util.*;

public class CopyExample {
  public static void main(String[] args) {
    try {
      copy(Path.of("a.txt"), Path.of("b.txt"));
    } catch (CopyFailureException e) {
      System.err.println("복사 실패: " + e.getMessage());
      for (Throwable s : e.getSuppressed())
        System.err.println("suppressed: " + s);
    }
  }

  static void copy(Path src, Path dst) {
    try (InputStream in = Files.newInputStream(src);
         OutputStream out = Files.newOutputStream(dst)) {
      in.transferTo(out);
    } catch (IOException e) {
      throw new CopyFailureException("파일 복사 중 오류: " + src + " -> " + dst, e);
    }
  }

  static class CopyFailureException extends RuntimeException {
    CopyFailureException(String m, Throwable c) { super(m, c); }
  }
}
```

### 리소스 3개 — 닫힘 순서 역순 & suppressed

```java
try (R r1 = new R("r1"); R r2 = new R("r2"); R r3 = new R("r3")) {
  throw new RuntimeException("body");
} catch (Exception e) {
  System.out.println("main: " + e);
  for (var s : e.getSuppressed()) System.out.println("suppressed: " + s);
}

static class R implements AutoCloseable {
  final String name;
  R(String n) { name = n; }
  public void close() { throw new RuntimeException("close " + name); }
}
```
- 출력: main 예외 `body`, suppressed: `close r3`, `close r2`, `close r1` (역순).

### 동시성 — 락과 인터럽트 보존

```java
import java.util.concurrent.locks.*;
class Counter {
  private final Lock lock = new ReentrantLock();
  private int value;

  int inc() {
    lock.lock();
    try {
      return ++value;
    } finally {
      lock.unlock();
    }
  }

  void sleepWork() {
    try {
      Thread.sleep(100);
    } catch (InterruptedException e) {
      Thread.currentThread().interrupt();
      throw new IllegalStateException("인터럽트됨", e);
    }
  }
}
```

### 입력 검증 — 언체크 예외

```java
static int parseAge(String s) {
  if (s == null) throw new IllegalArgumentException("age null");
  int age = Integer.parseInt(s);
  if (age < 0 || age > 150) throw new IllegalArgumentException("age 범위");
  return age;
}
```

---

## FAQ — 자주 묻는 함정/세부 규칙

1) **멀티-캐치에서 `e`는 effectively final** → `e = new ...` 불가.
2) **`try-with-resources`의 close 순서**: 선언 **역순**.
3) **`catch` 없이 `try-finally`만** 가능? → 가능. 단, 예외는 상위로 전파.
4) **`finally`의 예외와 본문 예외가 동시** → 본문 예외가 주 예외, close 예외는 **suppressed**.
5) **체크 예외를 전부 언체크로 감싸도 되나?** → 경계에서 **의미 있는 도메인 예외**로 번역하되, 복구 가능성이 있으면 체크 유지.
6) **자원 자동 해제가 안 되는 경우** → 반드시 `AutoCloseable` 구현을 고려(예: 커넥션/세션 핸들).

---

## 베스트 프랙티스 체크리스트

- [ ] **가장 좁은 범위**의 `try`로 감싸라(오염 방지).
- [ ] `catch`는 **정확한 타입**과 **정확한 대응**만(빈 `catch` 금지).
- [ ] **예외 메시지**는 **맥락**과 **키 데이터**를 담아라(PII/비밀은 마스킹).
- [ ] **예외 번역** 시 `cause`를 보존하라(`new X(msg, cause)`).
- [ ] **finally에서 반환/throw 금지**.
- [ ] 자원은 **`try-with-resources` 우선**.
- [ ] **동시성 자원(락)**은 `try-finally`로 해제.
- [ ] **정상 흐름에 예외 사용 금지**(성능/가독성 저하).
- [ ] **한 번만 로깅**(중복 로그/이중 스택트레이스 금지).
- [ ] 단위 테스트는 `assertThrows`로 명세화.

---

## 미니 요약표

| 주제 | 핵심 |
|---|---|
| 예외 분류 | 복구 가능(checked) vs 버그(unchecked) |
| 캐치 순서 | 하위 → 상위 |
| 멀티-캐치 | `IOException | SQLException` |
| precise rethrow | `catch (Exception e) { throw e; }` (선언 유지) |
| finally | 자원 해제만, return/throw 금지 |
| TWR | 자동 close, 역순, suppressed 보존 |
| 동시성 | 락/인터럽트는 finally/interrupt 보존 |
| 설계 | 번역(translation) + cause 유지, 한 번만 로깅 |

---

## 연습 문제(간단)

1) 파일 두 개를 읽어 **한 줄씩 병합 출력**하되, **모든 스트림은 TWR**로 열고, **본문/close 동시 예외**를 로그로 구분해 보세요.
2) DB 호출 코드에서 `SQLException`을 `RepositoryException`(언체크)으로 **번역**하되, 상위 서비스 계층에서 **한 번만 로깅**하도록 구성해 보세요.
3) `ReentrantLock`으로 보호되는 카운터에서, `InterruptedException`을 **보존 + 번역**하는 메서드를 작성해 보세요.

---

## 부록 — 최소 예제(통합)

```java
import java.io.*;
import java.nio.file.*;
import java.util.concurrent.locks.*;

public class TryCatchFinallyGuide {
  public static void main(String[] args) {
    // 1) TWR + 번역
    try {
      String first = firstLine(Path.of("build.gradle"));
      System.out.println(first);
    } catch (AppIoException e) {
      e.printStackTrace();
    }

    // 2) finally에서 unlock
    SafeCounter c = new SafeCounter();
    System.out.println(c.inc());

    // 3) 잘못된 finally 예시(주석 해제 금지)
    // badFinally();
  }

  static String firstLine(Path p) {
    try (BufferedReader br = Files.newBufferedReader(p)) {
      return br.readLine();
    } catch (IOException e) {
      throw new AppIoException("read fail: " + p, e);
    }
  }

  static class AppIoException extends RuntimeException {
    AppIoException(String m, Throwable c) { super(m, c); }
  }

  static class SafeCounter {
    private final Lock lock = new ReentrantLock();
    private int value;
    int inc() {
      lock.lock();
      try { return ++value; }
      finally { lock.unlock(); }
    }
  }

  static void badFinally() {
    try { throw new IllegalStateException("원래 예외"); }
    finally { throw new RuntimeException("finally 덮음"); }
  }
}
```

---

### 마무리

- `try-catch-finally`는 **안전한 종료/자원 회수**의 기본 골격, `try-with-resources`는 **현대적 기본값**입니다.
- 예외는 **계약**이자 **문서화된 제어 흐름**입니다. **정확한 타입**, **명확한 메시지**, **원인 보존**, **적절한 레이어에서의 번역/로깅**이 견고한 코드의 핵심입니다.
