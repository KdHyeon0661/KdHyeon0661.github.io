---
layout: post
title: Java - Optional
date: 2025-08-04 18:20:23 +0900
category: Java
---
# Optional

`Optional<T>`는 Java 8에서 도입된 **null 안전성**을 강화하기 위한 컨테이너 객체이다.  
`null` 값을 직접 다루는 대신, 값이 있을 수도 있고 없을 수도 있는 상황을 안전하게 표현할 수 있게 한다.

---

## 1. Optional의 필요성

Java에서 `null`은 값이 없음을 나타내지만, 이를 잘못 처리하면 `NullPointerException`(NPE)이 발생한다.

예:
```java
String name = getName();
System.out.println(name.length()); // name이 null이면 NPE 발생
```

`Optional`은 다음과 같은 장점을 제공한다.
- **명시적**: 값이 없을 수 있다는 것을 타입 레벨에서 알린다.
- **안전성**: 메서드 체인에서 null 체크 로직을 줄일 수 있다.
- **의도 표현**: API 설계 시 null 반환보다 명확한 의미 전달.

---

## 2. Optional 생성 방법

### 2.1 값이 있는 경우
```java
Optional<String> opt = Optional.of("Hello");
```
- `of(T value)`는 **null이 아닌 값**을 감싼다.
- `null`을 넣으면 `NullPointerException` 발생.

### 2.2 값이 없을 수 있는 경우
```java
Optional<String> opt = Optional.ofNullable(getName());
```
- `ofNullable(T value)`는 값이 null이면 `Optional.empty()`를 반환.

### 2.3 비어 있는 Optional
```java
Optional<String> opt = Optional.empty();
```
- 값이 전혀 없는 Optional 객체를 생성.

---

## 3. 값 접근 방법

### 3.1 `isPresent()` / `isEmpty()` (Java 11+)
```java
if (opt.isPresent()) {
    System.out.println(opt.get());
}

if (opt.isEmpty()) {
    System.out.println("값이 없음");
}
```
- `get()`은 값이 없으면 `NoSuchElementException`을 던지므로 **주의**.

### 3.2 `orElse()` / `orElseGet()` / `orElseThrow()`
```java
String name = opt.orElse("기본값"); // 값이 없으면 기본값
String nameLazy = opt.orElseGet(() -> "지연 계산 값");
String nameThrow = opt.orElseThrow(() -> new IllegalArgumentException("값 없음"));
```
- `orElseGet()`은 필요할 때만 Supplier를 실행하므로 성능상 유리.
- `orElse()`는 무조건 인자를 평가.

---

## 4. 메서드 체이닝과 함수형 활용

### 4.1 `ifPresent()`
```java
opt.ifPresent(value -> System.out.println("값: " + value));
```

### 4.2 `map()` 변환
```java
Optional<Integer> lengthOpt = opt.map(String::length);
```
- 값이 있으면 함수를 적용, 없으면 그대로 `Optional.empty()` 반환.

### 4.3 `flatMap()` (Optional 중첩 제거)
```java
Optional<String> upperOpt = opt
    .flatMap(value -> Optional.of(value.toUpperCase()));
```

### 4.4 `filter()`
```java
Optional<String> filteredOpt = opt.filter(v -> v.startsWith("H"));
```
- 조건에 맞지 않으면 `Optional.empty()` 반환.

---

## 5. 잘못된 사용 예 (Anti-Pattern)

1. **Optional을 필드에 사용**
   - Optional은 메서드 반환 타입에 쓰는 것이 일반적.
   - 직렬화 문제와 불필요한 오버헤드 발생.

2. **Optional.get() 남용**
   - Optional의 목적은 null 체크를 없애는 것인데, get() 사용 시 다시 예외 위험 존재.

3. **컬렉션에 Optional 넣기**
   - 컬렉션에 null 대신 Optional을 넣는 것은 혼란을 초래.

---

## 6. 활용 예시

### 6.1 안전한 메서드 체이닝
```java
String name = person
    .map(Person::getAddress)
    .map(Address::getStreet)
    .orElse("주소 없음");
```

### 6.2 DB 조회 결과 처리
```java
Optional<User> userOpt = userRepository.findById(1L);

userOpt.ifPresent(user -> System.out.println(user.getName()));
```

---

## 7. 정리

- `Optional`은 null을 대체하는 안전한 컨테이너.
- 메서드 반환 타입에서만 주로 사용.
- `isPresent()`보다 `ifPresent()`, `orElse()` 계열 메서드 사용 권장.
- 스트림과 함께 사용하면 더 강력한 null-safe 처리 가능.