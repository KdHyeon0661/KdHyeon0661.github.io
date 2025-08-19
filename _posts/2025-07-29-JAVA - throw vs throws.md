---
layout: post
title: Java - throw vs throws
date: 2025-07-29 14:20:23 +0900
category: Java
---
# `throw` vs `throws` in Java: 자세한 설명과 차이점

Java에서 예외 처리를 할 때 `throw`와 `throws`는 매우 유사하게 보이지만 **용도와 의미가 완전히 다릅니다.** 이 문서에서는 두 키워드의 차이와 사용법, 예제를 통해 그 개념을 명확히 설명합니다.

---

## 1. 개요

| 키워드 | 의미 | 위치 | 용도 |
|--------|------|------|------|
| `throw` | **예외 객체를 발생시킴** | 메서드 내부 | 실제 예외를 발생시킬 때 |
| `throws` | **예외를 메서드 시그니처에 선언** | 메서드 선언부 | 예외를 호출자에게 위임할 때 |

---

## 2. `throw` – 예외를 "발생"시키는 키워드

### 사용 목적
- 예외 객체를 명시적으로 **던지는** 역할
- `throw new 예외객체` 형식으로 사용

### 문법
```java
throw new ExceptionType("예외 메시지");
```

### 예제
```java
public class ThrowExample {
    public static void main(String[] args) {
        int age = -1;
        if (age < 0) {
            throw new IllegalArgumentException("나이는 음수가 될 수 없습니다.");
        }
    }
}
```

- 이 코드는 `IllegalArgumentException`을 직접 발생시킵니다.
- `throw`는 **예외 객체 단 하나만** 던질 수 있습니다.

---

## 3. `throws` – 예외를 "선언"하는 키워드

### 사용 목적
- 메서드가 특정 예외를 **처리하지 않고 호출자에게 전달**할 경우 사용
- 체크 예외(checked exception)를 **컴파일러에 알리는 역할**

### 문법
```java
public void readFile() throws IOException {
    // 예외 발생 가능 코드
}
```

### 예제
```java
public class ThrowsExample {
    public static void main(String[] args) {
        try {
            readFile();
        } catch (IOException e) {
            System.out.println("파일 읽기 실패: " + e.getMessage());
        }
    }

    public static void readFile() throws IOException {
        BufferedReader reader = new BufferedReader(new FileReader("test.txt"));
        reader.readLine();
        reader.close();
    }
}
```

- `readFile()` 메서드는 `IOException`이 발생할 수 있으므로 `throws`로 선언
- 이 예외는 호출한 쪽(`main`)에서 처리합니다

---

## 4. `throw`와 `throws` 비교

| 항목 | `throw` | `throws` |
|------|---------|----------|
| 목적 | 예외를 실제로 **발생시킴** | 예외가 발생할 수 있음을 **선언** |
| 위치 | 메서드 **내부** | 메서드 **선언부** |
| 동작 | 실행 시점에 예외를 던짐 | 컴파일 시점에 예외 전파를 알림 |
| 예외 개수 | 단 **하나의 예외 객체만** 가능 | **여러 예외 타입** 선언 가능 (쉼표로 구분) |
| 예외 객체 | 반드시 `new`로 생성해야 함 | 예외 객체 없이 타입만 명시 |

---

## 5. 두 키워드 함께 사용하기

두 키워드는 보통 함께 사용됩니다. 예외를 **던질 메서드는 `throws`로 선언**, 그리고 내부에서는 `throw`로 실제 예외를 던집니다.

```java
public static void validate(int age) throws IllegalArgumentException {
    if (age < 0) {
        throw new IllegalArgumentException("나이는 음수일 수 없습니다.");
    }
}
```

- `throws`: 이 메서드는 예외를 던질 수 있음을 명시
- `throw`: 실제로 예외를 던짐

---

## 6. 체크 예외 vs 언체크 예외와의 관계

### 체크 예외 (Checked Exception)
- 컴파일러가 강제하는 예외 (예: `IOException`, `SQLException`)
- 반드시 `throws`로 선언하거나 `try-catch`로 처리해야 함

```java
public void connect() throws IOException {
    // 파일 또는 네트워크 접근
}
```

### 언체크 예외 (Unchecked Exception)
- `RuntimeException` 하위 클래스
- `throws` 선언 없이 `throw`만 사용해도 됨

```java
public void divide(int a, int b) {
    if (b == 0) {
        throw new ArithmeticException("0으로 나눌 수 없습니다.");
    }
}
```

- `throws ArithmeticException` 선언은 **선택 사항**

---

## 7. 다중 예외 선언 (throws)

`throws`는 쉼표로 여러 예외를 선언할 수 있습니다.

```java
public void process() throws IOException, SQLException {
    // 예외 발생 가능 코드
}
```

---

## 8. 사용자 정의 예외와 `throw`, `throws`

### 사용자 정의 예외 클래스

```java
public class AgeException extends Exception {
    public AgeException(String message) {
        super(message);
    }
}
```

### 사용하는 예제

```java
public void checkAge(int age) throws AgeException {
    if (age < 0) {
        throw new AgeException("음수 나이는 허용되지 않습니다.");
    }
}
```

---

## 9. 요약

| 구분 | throw | throws |
|------|-------|--------|
| 정의 | 예외를 발생시킴 | 예외를 선언함 |
| 위치 | 메서드 내부 | 메서드 시그니처 |
| 목적 | 예외 객체를 명시적으로 던짐 | 예외가 발생할 수 있음을 호출자에게 알림 |
| 사용 대상 | 예외 객체 | 예외 타입 |
| 예외 개수 | 1개만 | 여러 개 가능 |
| new 키워드 | 필수 | 없음 |

---

## 10. 결론

- `throw`는 **예외 객체를 던지는 명령어**이고, `throws`는 **예외가 발생할 수 있음을 알리는 선언**입니다.
- 예외를 다룰 때는 항상 어떤 예외가 발생할 수 있는지 예측하고, `throw`와 `throws`를 적절히 조합하여 사용해야 합니다.
- 견고한 자바 프로그램을 작성하기 위해서는 이 두 키워드의 차이를 명확히 이해하고 있어야 합니다.