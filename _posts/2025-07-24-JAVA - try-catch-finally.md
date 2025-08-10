---
layout: post
title: Java - try-catch-finally
date: 2025-07-24 23:20:23 +0900
category: Java
---
# 자바의 `try-catch-finally` 구조에 대해 자세하게 정리

자바(Java)에서는 프로그램 실행 중 발생할 수 있는 예외(Exception)를 처리하기 위해 `try-catch-finally` 구조를 사용합니다. 이 구조는 예외를 안전하게 처리하고, 프로그램이 비정상 종료되지 않도록 도와줍니다.

---

## 1. 기본 개념

### try
- **예외가 발생할 수 있는 코드 블록**을 정의합니다.
- 예외가 발생하면 곧바로 `catch` 블록으로 제어가 이동합니다.

### catch
- try 블록에서 **발생한 예외를 처리**하는 블록입니다.
- 특정 예외 타입을 지정하여 해당 예외를 처리합니다.

### finally
- **예외 발생 여부와 관계없이 항상 실행되는 블록**입니다.
- 주로 **자원 해제**(파일 닫기, DB 연결 해제 등)에 사용됩니다.

---

## 2. 기본 문법

```java
try {
    // 예외가 발생할 수 있는 코드
} catch (ExceptionType e) {
    // 예외 처리 코드
} finally {
    // 항상 실행되는 코드 (자원 정리 등)
}
```

---

## 3. 예제: 0으로 나누기

```java
public class TryCatchExample {
    public static void main(String[] args) {
        try {
            int result = 10 / 0;  // 예외 발생
            System.out.println("Result: " + result);
        } catch (ArithmeticException e) {
            System.out.println("예외 발생: " + e.getMessage());
        } finally {
            System.out.println("finally 블록 실행됨");
        }
    }
}
```

### 출력
```
예외 발생: / by zero
finally 블록 실행됨
```

- `ArithmeticException` 예외가 발생하여 `catch` 블록으로 이동합니다.
- `finally` 블록은 예외 발생과 상관없이 **항상 실행**됩니다.

---

## 4. 다중 catch 블록

여러 종류의 예외를 따로 처리하고 싶은 경우, **다중 catch 블록**을 사용할 수 있습니다.

```java
try {
    String s = null;
    System.out.println(s.length());  // NullPointerException 발생
} catch (ArithmeticException e) {
    System.out.println("수학적 오류 발생");
} catch (NullPointerException e) {
    System.out.println("널 참조 오류 발생");
}
```

### 주의
- 예외 클래스는 **상속 관계**가 있으므로 **하위 예외부터 위에 작성**해야 합니다.

```java
// 잘못된 예시
catch (Exception e) { ... }
catch (NullPointerException e) { ... } // 도달 불가 컴파일 오류
```

---

## 5. 다중 예외 한 번에 처리 (Java 7+)

Java 7부터는 **하나의 catch 블록에서 여러 예외를 처리**할 수 있습니다.

```java
try {
    // 코드
} catch (IOException | SQLException e) {
    System.out.println("입출력 또는 SQL 오류 발생");
}
```

- `|` (파이프 기호)로 여러 예외를 묶어 처리 가능
- `e`는 **final로 암묵적으로 선언**되므로, 내부에서 수정 불가

---

## 6. finally 블록의 용도

`finally`는 주로 **리소스 해제** 목적으로 사용됩니다.

```java
BufferedReader reader = null;
try {
    reader = new BufferedReader(new FileReader("file.txt"));
    String line = reader.readLine();
    System.out.println(line);
} catch (IOException e) {
    e.printStackTrace();
} finally {
    try {
        if (reader != null) {
            reader.close();
        }
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```

- 파일을 열었다면 반드시 닫아야 하므로 `finally`에서 `close()`를 수행합니다.

---

## 7. return과 finally

`finally` 블록은 **return이 호출되어도 반드시 실행**됩니다.

```java
public static int test() {
    try {
        return 1;
    } finally {
        System.out.println("finally 실행됨");
    }
}
```

### 출력
```
finally 실행됨
```

> `finally`는 예외가 발생하거나 `return`이 호출되어도 반드시 실행됨

---

## 8. 예외가 없는 경우의 finally

예외가 발생하지 않아도 `finally`는 실행됩니다.

```java
try {
    System.out.println("정상 실행");
} catch (Exception e) {
    System.out.println("예외 발생");
} finally {
    System.out.println("무조건 실행");
}
```

### 출력
```
정상 실행  
무조건 실행
```

---

## 9. try-finally 구조 (catch 생략 가능)

예외를 처리하지 않고 무조건 자원을 해제하고 싶을 때는 `catch` 없이 `try-finally`만 사용 가능합니다.

```java
try {
    // 예외 발생 가능 코드
} finally {
    // 자원 정리
}
```

---

## 10. try-with-resources (Java 7+)

`AutoCloseable`을 구현한 객체는 `try-with-resources`를 사용해 자동으로 자원을 해제할 수 있습니다.

```java
try (BufferedReader br = new BufferedReader(new FileReader("file.txt"))) {
    String line = br.readLine();
    System.out.println(line);
} catch (IOException e) {
    e.printStackTrace();
}
```

- `try` 안에 선언한 객체는 `try` 블록이 끝나면 자동으로 `close()` 호출됨
- `finally` 블록 없이도 안전한 자원 해제 가능

---

## 11. 예외 전파 (`throws`)

메서드에서 예외를 처리하지 않고 **호출한 쪽으로 전달**하려면 `throws` 키워드를 사용합니다.

```java
public void readFile() throws IOException {
    BufferedReader br = new BufferedReader(new FileReader("file.txt"));
    br.readLine();
}
```

- `IOException`이 발생하면 호출자가 처리하도록 위임합니다.

---

## 요약 정리

| 구문 | 설명 |
|------|------|
| `try` | 예외 발생 가능 코드 |
| `catch` | 특정 예외 처리 |
| `finally` | 예외 유무 상관없이 항상 실행 |
| 다중 catch | 여러 예외 유형을 분리해서 처리 |
| 멀티-catch (Java 7+) | `|`로 여러 예외를 한 번에 처리 |
| try-finally | catch 없이 자원 해제 전용 |
| try-with-resources | 자동 자원 해제 (`AutoCloseable` 필요) |
| throws | 예외를 호출자에게 위임 |

---

## 결론

- `try-catch-finally`는 예외 발생에 안전하게 대처할 수 있도록 하는 필수적인 구조입니다.
- `finally`는 리소스 누수를 방지하고, `try-with-resources`는 이를 간결하게 만들어줍니다.
- 명확한 예외 처리 정책은 견고한 프로그램의 핵심입니다.