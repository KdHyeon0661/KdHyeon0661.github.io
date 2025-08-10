---
layout: post
title: Java - Object 클래스
date: 2025-07-19 20:20:23 +0900
category: Java
---
# Java의 Object 클래스

Java에서 **모든 클래스는 Object 클래스의 자식**입니다. 즉, **Object는 모든 클래스의 최상위 부모 클래스(Superclass)**입니다.

```java
public class MyClass {
    // 자동으로 다음과 같음
    // extends Object
}
```

이로 인해 Java의 모든 클래스는 `Object` 클래스의 메서드를 **상속**하게 되며, 필요에 따라 **오버라이딩**해서 사용할 수 있습니다.

---

## 1. Object 클래스의 주요 메서드

다음은 `Object` 클래스가 제공하는 대표적인 메서드들입니다:

| 메서드 | 설명 |
|--------|------|
| `equals(Object obj)` | 두 객체가 같은지 비교 |
| `hashCode()` | 객체의 해시 코드 반환 |
| `toString()` | 객체를 문자열로 표현 |
| `getClass()` | 객체의 클래스 정보를 반환 |
| `clone()` | 객체를 복제 |
| `finalize()` | 가비지 컬렉션 직전 호출됨 |
| `wait()`, `notify()`, `notifyAll()` | 스레드 동기화 제어 메서드 |

---

## 2. equals()

두 객체가 **논리적으로 같은지** 비교하는 메서드입니다.

기본 구현은 `==`와 동일하게 **참조값(주소)**을 비교합니다.

```java
Object obj1 = new Object();
Object obj2 = new Object();

System.out.println(obj1.equals(obj2)); // false
```

그러나 우리가 작성하는 클래스에서 **논리적 동등성**을 비교하고 싶다면 `equals()`를 오버라이드 해야 합니다.

```java
public class Person {
    String name;

    Person(String name) {
        this.name = name;
    }

    @Override
    public boolean equals(Object obj) {
        if (obj instanceof Person) {
            Person p = (Person) obj;
            return this.name.equals(p.name);
        }
        return false;
    }
}
```

---

## 3. hashCode()

`hashCode()`는 객체의 **해시 코드(int)**를 반환합니다. 이는 **HashMap, HashSet** 등 해시 기반 컬렉션에서 중요하게 사용됩니다.

- `equals()`를 오버라이딩 했다면 `hashCode()`도 반드시 오버라이딩 해야 합니다.
- 두 객체가 `equals()`로 같으면 `hashCode()`도 같아야 합니다.

```java
@Override
public int hashCode() {
    return name.hashCode();
}
```

---

## 4. toString()

객체를 **문자열로 표현**하는 메서드입니다.

기본 구현:

```java
ClassName@16진수해시코드
```

예시:

```java
Person p = new Person("Alice");
System.out.println(p); // Person@5a39699c
```

보통은 오버라이딩하여 의미 있는 정보를 출력하도록 합니다.

```java
@Override
public String toString() {
    return "Person{name=" + name + "}";
}
```

---

## 5. getClass()

객체의 실제 **클래스 정보를 담은 Class 객체**를 반환합니다.

```java
Person p = new Person("Alice");
System.out.println(p.getClass().getName()); // Person
```

---

## 6. clone()

객체를 복제할 때 사용하는 메서드입니다. `Cloneable` 인터페이스를 구현한 클래스만 사용 가능하며, 기본은 **얕은 복사**를 수행합니다.

```java
public class Person implements Cloneable {
    String name;

    @Override
    public Object clone() throws CloneNotSupportedException {
        return super.clone();
    }
}
```

사용 예:

```java
Person p1 = new Person("Alice");
Person p2 = (Person) p1.clone();
```

---

## 7. finalize()

- 객체가 **가비지 컬렉션**되기 전에 호출됩니다.
- 보통은 사용하지 않으며, 자원 해제를 위해 `try-with-resources` 또는 `close()`를 사용하는 것이 좋습니다.
- Java 9부터는 `finalize()`는 deprecated 되었습니다.

---

## 8. 동기화 관련 메서드

- `wait()`, `notify()`, `notifyAll()`은 **멀티스레딩 환경에서 객체 잠금(lock)**을 제어할 때 사용됩니다.
- 이 메서드들은 모두 **synchronized 블록 내에서**만 사용할 수 있습니다.

```java
synchronized(obj) {
    obj.wait();     // 현재 스레드는 대기
    obj.notify();   // 하나의 대기 스레드를 깨움
    obj.notifyAll(); // 모든 대기 스레드를 깨움
}
```

---

## 9. Object를 이용한 다형성 활용

Java의 다형성은 모든 클래스가 Object를 상속한다는 점을 이용해 **다양한 객체를 Object 타입으로 처리**할 수 있게 해줍니다.

```java
Object obj = "Hello";
System.out.println(obj.toString()); // "Hello"
```

---

## 10. Object를 상속하지 않는 경우?

- Java에서 작성하는 **모든 클래스는 Object를 암묵적으로 상속**합니다.
- 단, **인터페이스(interface)**는 `Object`를 직접 상속하지 않지만, 인터페이스를 구현한 클래스는 `Object`의 메서드를 사용할 수 있습니다.

---

## 정리

| 메서드 | 설명 | 자주 오버라이딩? |
|--------|------|------------------|
| `equals()` | 객체 논리적 비교 | ✅ |
| `hashCode()` | 해시 기반 자료구조 지원 | ✅ |
| `toString()` | 문자열 표현 | ✅ |
| `clone()` | 객체 복사 | ⚠️ 제한적 사용 |
| `finalize()` | GC 직전 자원 해제 | ❌ 비권장 |
| `getClass()` | 클래스 정보 반환 | ⭕️ |
| `wait()/notify()` | 스레드 동기화 | ⭕️ 스레드 프로그래밍 시 |

---

## 결론

- `Object` 클래스는 Java에서 모든 클래스의 공통 부모입니다.
- `equals()`, `hashCode()`, `toString()`은 사용자 정의 클래스에서 자주 오버라이딩 해야 합니다.
- Java의 **다형성, 상속, 컬렉션 API, 스레드** 등 핵심 기능은 `Object`를 기반으로 설계되어 있습니다.