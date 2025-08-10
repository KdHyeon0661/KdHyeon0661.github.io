---
layout: post
title: Java - Wrapper 클래스
date: 2025-07-19 21:20:23 +0900
category: Java
---
# Java의 Wrapper 클래스 (Integer, Double, etc.)

Java에는 `int`, `double`, `boolean`과 같은 **기본형(primitive type)**이 존재합니다. 그러나 Java는 **모든 것이 객체(Object)**인 언어 철학을 지향하기 때문에, **기본형을 객체로 다룰 수 있는 방법**이 필요합니다.

이때 사용하는 것이 바로 **Wrapper 클래스**입니다.

---

## 1. Wrapper 클래스란?

Wrapper 클래스는 **기본형 데이터를 객체로 감싸주는 클래스**입니다.

| 기본형 | Wrapper 클래스 |
|--------|----------------|
| `byte` | `Byte`         |
| `short`| `Short`        |
| `int`  | `Integer`      |
| `long` | `Long`         |
| `float`| `Float`        |
| `double`| `Double`      |
| `char` | `Character`    |
| `boolean` | `Boolean`    |

예시:

```java
int num = 10;
Integer obj = Integer.valueOf(num); // int → Integer
```

---

## 2. 사용 목적

### (1) 제네릭과 컬렉션에서 사용 가능
Java의 **제네릭(generic)**은 객체 타입만 지원합니다. 기본형은 사용할 수 없습니다.

```java
List<int> list = new ArrayList<>(); // 오류 ❌
List<Integer> list = new ArrayList<>(); // 가능 ✅
```

### (2) null 값 표현 가능
기본형은 `null`을 가질 수 없습니다. Wrapper는 가능해서 **값이 없는 상태(null)**를 표현할 수 있습니다.

```java
Integer x = null; // 유효
int y = null;     // 오류
```

### (3) 형 변환과 문자열 처리
Wrapper 클래스는 **문자열 ↔ 숫자 변환** 등의 유틸리티 메서드를 제공합니다.

```java
String str = "123";
int num = Integer.parseInt(str); // 문자열 → int
```

---

## 3. Boxing과 Unboxing

### ▶️ Boxing (기본형 → Wrapper)

```java
int a = 5;
Integer obj = Integer.valueOf(a); // 명시적 Boxing
Integer obj2 = a;                 // Auto Boxing (자동 박싱)
```

### ▶️ Unboxing (Wrapper → 기본형)

```java
Integer obj = 10;
int b = obj.intValue(); // 명시적 Unboxing
int c = obj;            // Auto Unboxing (자동 언박싱)
```

---

## 4. 주요 메서드

### ✅ `Integer` 클래스 예시

```java
Integer i = Integer.valueOf("42");  // 문자열 → Integer
int x = i.intValue();               // Integer → int

int y = Integer.parseInt("123");    // 문자열 → int
String s = Integer.toString(123);   // int → 문자열

Integer a = 100;
Integer b = 100;
System.out.println(a == b);         // true (캐싱 범위)
```

※ 주의: `Integer`는 `-128` ~ `127` 범위의 값은 **캐싱**되므로 `==` 비교가 일치할 수 있지만, 그 이상은 다릅니다.

```java
Integer x = 200;
Integer y = 200;
System.out.println(x == y); // false ❌
System.out.println(x.equals(y)); // true ✅
```

---

## 5. `Integer`, `Double`, `Boolean` 클래스의 특징 요약

### 📌 Integer

- 32비트 정수 표현
- `MAX_VALUE`, `MIN_VALUE` 등 상수 제공
- 유틸리티: `parseInt()`, `toString()`, `compare()`, `valueOf()`

```java
System.out.println(Integer.MAX_VALUE); // 2147483647
```

### 📌 Double

- 64비트 부동소수점
- 유사하게 `parseDouble()`, `valueOf()`, `isNaN()`, `isInfinite()` 등 존재

```java
double d = Double.parseDouble("3.14");
System.out.println(Double.isNaN(d)); // false
```

### 📌 Boolean

- true/false 값만 저장
- `Boolean.valueOf("true")` → true
- `Boolean.valueOf("false")` → false

```java
Boolean b = Boolean.valueOf("true");
System.out.println(b); // true
```

---

## 6. Wrapper 클래스의 불변성

Wrapper 클래스는 **Immutable(불변)**입니다. 즉, 객체의 값은 한 번 생성되면 변경할 수 없습니다.

```java
Integer a = 10;
a = 20; // 새로운 Integer 객체 생성
```

---

## 7. Wrapper 클래스의 캐싱

- `Integer`, `Byte`, `Short`, `Long`, `Character`의 일부 값은 JVM이 캐싱함
- `-128` ~ `127` 범위는 미리 생성되어 공유됨

```java
Integer x = 127;
Integer y = 127;
System.out.println(x == y); // true

Integer a = 128;
Integer b = 128;
System.out.println(a == b); // false
```

---

## 8. 성능 이슈

- Boxing/Unboxing은 **자동으로 변환되지만 비용이 존재**합니다.
- 루프나 연산에 많이 사용되는 경우, 기본형을 쓰는 것이 더 효율적입니다.

```java
long start = System.nanoTime();
int sum = 0;
for (int i = 0; i < 1_000_000; i++) {
    sum += i;
}
long end = System.nanoTime();
System.out.println("기본형 소요시간: " + (end - start));
```

---

## 9. null 처리 주의

Wrapper 클래스는 `null` 값을 가질 수 있으므로 **Unboxing 시 NullPointerException**이 발생할 수 있습니다.

```java
Integer x = null;
int y = x; // NullPointerException ❌
```

예외를 피하기 위해 `Optional`을 사용하거나 null 체크가 필요합니다.

---

## 10. 정리

| 항목 | 기본형 | Wrapper 클래스 |
|------|--------|----------------|
| 객체인가? | ❌ | ✅ |
| null 가능? | ❌ | ✅ |
| 제네릭 사용 | ❌ | ✅ |
| 성능 | 빠름 | 느림 (Boxing overhead) |
| 예외 발생 가능성 | 없음 | `NullPointerException` 가능 |

---

## 📌 결론

- Wrapper 클래스는 **기본형을 객체처럼 다룰 수 있게 해주는 클래스**입니다.
- 제네릭, 컬렉션, null 처리 등 객체 지향 프로그래밍에서 반드시 필요합니다.
- Boxing/Unboxing 자동화는 편리하지만 **성능 및 null 문제에 주의**해야 합니다.
- 실무에서는 `int`와 `Integer`, `double`과 `Double`을 상황에 맞게 **선택적으로 사용하는 능력**이 중요합니다.
