---
layout: post
title: Java - static과 final
date: 2025-07-19 19:20:23 +0900
category: Java
---
# static과 final 키워드

Java에서 `static`과 `final`은 매우 중요한 키워드로, 클래스 설계와 메모리 관리, 상수 정의, 상속 제어 등 여러 측면에서 사용됩니다.

---

## 1. static 키워드

`static` 키워드는 **클래스 수준에 속하는 멤버(변수, 메서드, 블록 등)**를 정의할 때 사용됩니다.

### 1.1 static 변수 (클래스 변수)

- 인스턴스가 아니라 클래스에 **공유되는 변수**입니다.
- 모든 인스턴스가 **하나의 static 변수**를 공유합니다.

```java
public class Counter {
    static int count = 0;

    public Counter() {
        count++;
    }
}

public class Main {
    public static void main(String[] args) {
        new Counter();
        new Counter();
        System.out.println(Counter.count); // 2
    }
}
```

- `Counter.count`처럼 **클래스명.변수명**으로 접근 가능

---

### 1.2 static 메서드

- 객체를 생성하지 않아도 호출 가능한 메서드입니다.
- `this` 키워드를 사용할 수 없습니다. (인스턴스 정보 없음)
- 주로 **유틸리티 메서드**나 **공통 로직**에 사용

```java
public class MathUtil {
    public static int square(int x) {
        return x * x;
    }
}

public class Main {
    public static void main(String[] args) {
        int result = MathUtil.square(5); // 25
    }
}
```

---

### 1.3 static 블록

- 클래스가 **처음 로딩될 때 한 번만 실행되는 초기화 블록**입니다.
- 복잡한 정적 초기화가 필요할 때 사용

```java
public class AppConfig {
    static int port;

    static {
        port = 8080;
        System.out.println("Static block executed");
    }
}
```

---

## 2. final 키워드

`final` 키워드는 **변경 불가(immutable)**를 의미하며, **변수, 메서드, 클래스**에 모두 사용할 수 있습니다.

---

### 2.1 final 변수

- **한 번 값이 할당되면 변경 불가능**한 상수입니다.
- **명시적 초기화** 또는 **생성자 초기화**가 반드시 필요

```java
public class Circle {
    final double PI = 3.14159;

    public double area(double r) {
        return PI * r * r;
    }
}
```

- `final` 변수는 **대문자+스네이크 표기**를 사용하는 것이 관례입니다 (`MAX_SIZE`, `DEFAULT_TIMEOUT` 등).

---

### 2.2 final 메서드

- **하위 클래스에서 오버라이드(재정의)할 수 없는 메서드**입니다.
- 동작을 변경하면 안 되는 로직에 사용

```java
class Parent {
    public final void show() {
        System.out.println("Can't override");
    }
}

class Child extends Parent {
    // public void show() {} // 컴파일 오류!
}
```

---

### 2.3 final 클래스

- **상속할 수 없는 클래스**입니다.
- 보안이나 안정성이 중요한 클래스에 사용

```java
public final class Constants {
    public static final int MAX = 100;
}

// class Extended extends Constants {} // 오류!
```

대표적인 `final` 클래스: `java.lang.String`, `Integer`, `Math`, `System` 등

---

## static vs final 요약 비교

| 키워드 | 의미 | 사용 대상 | 특징 |
|--------|------|------------|--------|
| `static` | 클래스 소속 | 변수, 메서드, 블록 | 클래스 단위 공유, 객체 없이 접근 가능 |
| `final` | 변경 불가 | 변수, 메서드, 클래스 | 상수, 재정의 금지, 상속 금지 등 |

---

## static final 함께 사용하기

두 키워드는 함께 자주 사용되며, **"변경 불가능한 클래스 상수"**를 정의할 때 매우 유용합니다.

```java
public class Config {
    public static final String VERSION = "1.0.0";
}
```

- `static`: 객체 없이 사용 가능
- `final`: 변경 불가
- `Config.VERSION`으로 접근 가능

---

## 정리

- `static`: 클래스 기반 멤버 정의, 모든 인스턴스에 공통 적용
- `final`: 값이나 동작 변경을 방지, 안전성과 불변성을 보장
- **조합**하여 클래스 상수나 안전한 공통 로직 구현에 활용

이 두 키워드를 잘 활용하면 코드의 **명확성, 효율성, 안정성**을 높일 수 있습니다.