---
layout: post
title: Java - Wrapper 클래스
date: 2025-07-19 21:20:23 +0900
category: Java
---
# Java의 Wrapper 클래스

## 한눈에 보는 핵심

| 항목 | 기본형(primitive) | Wrapper(참조형) |
|---|---|---|
| 값 보관 | 스택/레지스터에 값 자체 | 힙의 객체(참조를 통해 접근) |
| null 가능 | 불가 | 가능 |
| 제네릭/컬렉션 | 사용 불가 | 사용 가능 |
| 성능 | 빠름, 할당 없음 | 느릴 수 있음(할당/GC/오토박싱 비용) |
| 동치성 비교 | `==` 값 비교 | `equals()` 내용 비교 권장(단, `Boolean.TRUE/FALSE`는 싱글턴) |

Wrapper 클래스 매핑:

| 기본형 | Wrapper |
|---|---|
| `byte` | `Byte` |
| `short` | `Short` |
| `int` | `Integer` |
| `long` | `Long` |
| `float` | `Float` |
| `double` | `Double` |
| `char` | `Character` |
| `boolean` | `Boolean` |

> 대부분의 숫자 Wrapper는 `java.lang.Number`를 **상속**합니다. `Character`, `Boolean`은 예외로 Number를 상속하지 않습니다.

---

## 왜 Wrapper가 필요한가?

### 제네릭/컬렉션에서 객체만 허용

```java
// List<int>  // 불가
List<Integer> nums = new ArrayList<>(); // 가능
```

### null 표현

```java
Integer maybeCount = null; // "없음" 상태 표현 가능
```

### 문자열 변환 및 유틸 메서드

```java
int v = Integer.parseInt("123");
String s = Integer.toString(123);   // "123"
```

### 리플렉션/varargs/빈 컨테이너와의 상호 운용

- `Object...` 가변인자, 리플렉션 API 등은 **객체**를 기대하므로 Wrapper가 필요합니다.

---

## 오토박싱/언박싱(Autoboxing/Unboxing)

### 개념

- **박싱(Boxing)**: 기본형 → Wrapper 객체
- **언박싱(Unboxing)**: Wrapper → 기본형
  Java는 문맥상 필요한 경우 **자동으로 변환**합니다.

```java
Integer a = 5;     // 오토박싱: Integer.valueOf(5)
int b = a;         // 오토언박싱: a.intValue()
```

### 산술/비교 시 암묵적 언박싱

```java
Integer x = 10, y = 20;
int sum = x + y; // 둘 다 언박싱되어 int 연산
```

### 주의: null 언박싱은 NPE

```java
Integer n = null;
int k = n; // NullPointerException
```

회피 패턴:
```java
int safe = (n != null) ? n : 0;
// 또는
int safe2 = java.util.Optional.ofNullable(n).orElse(0);
```

---

## 캐싱과 `==` 비교의 함정

### 정수/문자/불리언 캐싱

- `Integer`, `Short`, `Byte`, `Long` : **[-128, 127]** 범위 **캐싱**
- `Character` : **[0, 127]** 캐싱
- `Boolean` : `Boolean.TRUE` / `Boolean.FALSE` **싱글턴**
- `Float`, `Double` : 일반적으로 캐시 **안 함**

> 일부 JVM에서는 **`Integer` 캐시 상한(-XX:AutoBoxCacheMax)** 를 조정할 수 있습니다. 이식성 관점에서는 **`==` 비교에 의존하지 말 것**.

### `==` vs `equals`

```java
Integer a = 127, b = 127;   // 캐시 범위
System.out.println(a == b);      // true (같은 인스턴스일 가능성 높음)
Integer c = 128, d = 128;   // 캐시 밖
System.out.println(c == d);      // false (서로 다른 객체)
System.out.println(c.equals(d)); // true  (값 비교)
```

**규칙**: Wrapper 동등성은 항상 `equals()`를 사용하라.
예외적으로 `Boolean`은 싱글턴이지만 **일관성을 위해 equals 권장**.

---

## 클래스별 특징과 자주 쓰는 API

### `Integer` / `Long`

- 파싱/출력:
```java
int  n  = Integer.parseInt("101", 2);     // 2진수 → 5
long m  = Long.parseLong("ff", 16);       // 16진수 → 255
String s = Integer.toString(255, 16);     // "ff"
int v1   = Integer.decode("0x10");        // 16
int v2   = Integer.decode("010");         // 8(8진수)
```
- 비교/부호 없는 연산(정수 전용):
```java
int cmp = Integer.compare(3, 5);          // -1
long u  = Integer.toUnsignedLong(-1);     // 4294967295
int cu  = Integer.compareUnsigned(-1, 1); // >0
String us = Integer.toUnsignedString(-1); // "4294967295"
```
- 비트 유틸리티:
```java
int bc = Integer.bitCount(0b1011);        // 3
int hi = Integer.highestOneBit(20);       // 16
int lo = Integer.lowestOneBit(20);        // 4
int rl = Integer.rotateLeft(0b0011, 2);   // 0b1100
```

### `Float` / `Double` (부동소수점)

- 파싱/판별:
```java
double d = Double.parseDouble("3.14");
boolean inf = Double.isInfinite(d); // false
boolean nan = Double.isNaN(d);      // false
```
- **NaN과 동치성**:
```java
double p = Double.NaN;
System.out.println(p == Double.NaN);                  // false (primitive 비교)
System.out.println(Double.valueOf(p).equals(Double.NaN)); // true (Wrapper equals는 비트 비교)
System.out.println(Double.compare(p, Double.NaN));    // 0 (동등)
```
- **+0.0과 -0.0**:
```java
System.out.println(+0.0 == -0.0);                  // true
System.out.println(Double.compare(+0.0, -0.0));    // 1 (부호 구분)
```
- **`MIN_VALUE` 오해 주의**:
  `Double.MIN_VALUE`/`Float.MIN_VALUE`는 **가장 작은 양의(>0) 서브노멀 값**.
  가장 작은(가장 음수) 값은 `-Double.MAX_VALUE`.
- 유용 메서드: `toHexString`, `sum`, `max`, `min`, `Math.nextUp/nextDown`.

> 화폐/금융 계산은 부동소수점 대신 **`BigDecimal`** 사용 권장.

### `Boolean`

- 파싱 규칙:
```java
Boolean b1 = Boolean.valueOf("true");   // 대소문자 무관
Boolean b2 = Boolean.valueOf("anything"); // false
boolean p  = Boolean.parseBoolean("TRUE"); // true
```
- 싱글턴:
```java
System.out.println(Boolean.TRUE == Boolean.valueOf(true)); // true
```

### `Character`

- 분류/변환:
```java
char ch = '한';
boolean letter   = Character.isLetter(ch);
boolean digit    = Character.isDigit('3');
boolean white    = Character.isWhitespace(' ');
char upper       = Character.toUpperCase('a'); // 'A'
int  codePoint  = Character.codePointAt("A😊", 1); // 이모지 코드포인트
```

---

## `Number` 추상 클래스와 다형성

`Integer`, `Long`, `Float`, `Double`, `Byte`, `Short`는 `Number`를 상속.
공통 변환 메서드 제공:

```java
Number n = 42;  // Integer로 오토박싱
int    i = n.intValue();
double d = n.doubleValue();
long   l = n.longValue();
```

제네릭으로 숫자 처리:

```java
static double sumAll(List<? extends Number> xs) {
    double s = 0;
    for (Number n : xs) s += n.doubleValue(); // 공통 인터페이스
    return s;
}
```

---

## 문자열 ↔ 숫자 변환 모음

| 변환 | 메서드 |
|---|---|
| `"123"` → `int` | `Integer.parseInt("123")` |
| `"ff"`(16) → `int` | `Integer.parseInt("ff", 16)` |
| `"0x10"`/`"010"` | `Integer.decode(..)` |
| `int` → `"123"` | `Integer.toString(123)` / `String.valueOf(123)` |
| `"3.14"` → `double` | `Double.parseDouble("3.14")` |
| `double` → `"3.14"` | `Double.toString(3.14)` / `String.valueOf(3.14)` |
| `"true"` → `boolean` | `Boolean.parseBoolean("true")` |
| `boolean` → `"true"` | `Boolean.toString(true)` |

주의:
- 잘못된 포맷 → 숫자 파싱은 `NumberFormatException`.
- 공백/구분자 제거는 직접 처리 필요(또는 `trim()`).

---

## 스트림/컬렉션과 성능: 박싱 회피

### 박싱이 많은 코드(비권장)

```java
List<Integer> xs = IntStream.range(0, 1_000_000) // 박싱 발생
    .boxed()
    .collect(java.util.stream.Collectors.toList());
long sum = xs.stream().mapToLong(Integer::longValue).sum();
```

### 원시 스트림으로 처리(권장)

```java
long sum = java.util.stream.IntStream.range(0, 1_000_000).asLongStream().sum();
```

### Optional vs OptionalInt

```java
Optional<Integer> oi = Optional.of(10);     // 박싱 존재
OptionalInt     oi2 = OptionalInt.of(10);   // 박싱 없음
```

---

## null 안전 패턴

### 안전 언박싱

```java
Integer in = null;
// 방법 1
int v1 = (in != null) ? in : 0;
// 방법 2
int v2 = java.util.Optional.ofNullable(in).orElse(0);
// 방법 3 (JDK 9+)
Integer v3 = java.util.Objects.requireNonNullElse(in, 0);
```

### Comparator에서 null 처리

```java
Comparator<Integer> cmp = Comparator.nullsFirst(Integer::compare);
```

### switch와 null

```java
Integer k = null;
// switch (k) { ... } // NPE 위험(언박싱)
```

---

## 실전 예제

### CSV 숫자 컬럼 파싱(검증 포함)

```java
import java.util.*;
import java.util.stream.*;

public class CsvParse {
    static OptionalInt parseSafeInt(String s) {
        try {
            return OptionalInt.of(Integer.parseInt(s.trim()));
        } catch (NumberFormatException e) {
            return OptionalInt.empty();
        }
    }

    public static void main(String[] args) {
        String line = " 10, 20 , xxx , 40 ";
        int[] values = Arrays.stream(line.split(","))
            .map(CsvParse::parseSafeInt)
            .filter(OptionalInt::isPresent)
            .mapToInt(OptionalInt::getAsInt)
            .toArray();

        System.out.println(Arrays.toString(values)); // [10, 20, 40]
    }
}
```

### Double 특수값 처리

```java
static double parsePrice(String s) {
    double v = Double.parseDouble(s);
    if (Double.isNaN(v) || Double.isInfinite(v)) {
        throw new IllegalArgumentException("Invalid price: " + s);
    }
    return v;
}
```

### 정수 비트 통계/표현

```java
int x = -1;
System.out.println(Integer.toUnsignedString(x)); // 4294967295
System.out.println(Integer.bitCount(0b101010));   // 3
```

---

## 성능/메모리 관점 팁

- **루프 누적/연산은 기본형**을 사용하라. 불가피하면 원시 스트림 사용.
- **박싱 남발 금지**: `List<Integer>` 수백만 개는 **객체 오버헤드/GC 압박**.
- **캐시 범위에 기대어 `==` 비교하지 말 것**(이식성 X).
- 빈번한 파싱은 **사전 검증**(`Character.isDigit`, 범위체크)으로 예외 비용 최소화.
- 금융/정밀 계산은 **`BigDecimal`** 사용.
- 매우 큰 원시 컬렉션이 필요하면 **전용 라이브러리(primitive collections)** 고려(예: 특화된 해시맵/리스트). *표준 JDK에는 미포함.*

---

## 자주 하는 실수와 예방 체크리스트

| 실수 | 증상 | 해결 |
|---|---|---|
| Wrapper `==` 비교 | 예측불가(true/false 섞임) | **`equals` 사용** |
| null 언박싱 | NPE | `OptionalInt`/null 체크/`orElse` |
| 부동소수 `==` | 경계/반올림 오판 | `Double.compare`/허용 오차 비교 |
| `Double.MIN_VALUE` 의미 오해 | 음수 최솟값으로 오인 | “가장 작은 **양수** 값”임을 기억 |
| 스트림 박싱 남발 | 느림/GC↑ | `IntStream/LongStream/DoubleStream` |
| switch에 Wrapper | NPE | 언박싱 전 null 방어 또는 기본형 변환 |

---

## 요약

- **Wrapper는 기본형을 객체로 감싸** 제네릭/컬렉션/리플렉션 등에서 사용 가능하게 한다.
- **오토박싱/언박싱** 은 편리하지만 **null·성능 위험**이 있다.
- **캐싱과 `==` 함정**: 값 비교는 **항상 `equals`** 로.
- **스트림/Optional의 원시 특화 타입**으로 박싱 오버헤드를 줄여라.
- **부동소수 특수값/부호**(+0.0/-0.0, NaN, Infinity) 처리 규칙을 이해하라.

---

## 부록 A) 비교/캐싱 데모

```java
public class CacheDemo {
    public static void main(String[] args) {
        Integer a = 127,  b = 127;
        Integer c = 128,  d = 128;
        System.out.println(a == b);      // true (캐시)
        System.out.println(c == d);      // false (비캐시)
        System.out.println(c.equals(d)); // true  (값 비교)
    }
}
```

---

## 부록 B) 안전 합계(박싱 회피)

```java
long sum1 = java.util.stream.IntStream.rangeClosed(1, 1_000_000)
    .asLongStream()
    .sum(); // 박싱 없음

long sum2 = java.util.stream.Stream
    .iterate(1, x -> x + 1)
    .limit(1_000_000)
    .mapToLong(Integer::longValue) // 박싱 발생
    .sum();
```
첫 번째가 메모리/속도 모두 유리합니다.
