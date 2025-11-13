---
layout: post
title: Java - Optional
date: 2025-08-04 18:20:23 +0900
category: Java
---
# Optional

## 1. Optional이 왜 필요한가

### 1.1 Null의 문제
- `NullPointerException`(NPE)은 런타임 폭탄이며, **의도**(값이 없을 수 있음)를 타입만으로 드러내지 못합니다.
- 호출자는 문서/주석에 의존해 `null` 가능성을 추측해야 합니다.

### 1.2 Optional의 철학
- **“값이 없을 수 있다”**를 **타입 레벨**로 명시합니다.
- 호출자는 **컴파일 타임에 강제된 처리**(분기/폴백/예외)로 의도를 드러냅니다.
- 목표는 “`null` 제거”가 아니라 **의도를 명시하고 안전한 흐름을 강제**하는 것입니다.

---

## 2. 생성 — `of`, `ofNullable`, `empty`

```java
Optional<String> a = Optional.of("hi");            // 절대 null 아님 (null이면 NPE 즉시)
Optional<String> b = Optional.ofNullable(maybe()); // null 허용 → empty
Optional<String> c = Optional.empty();             // 명시적 비어있음
```

> 규칙: **입력 값이 null 가능**이면 `ofNullable`만 사용합니다. `of`는 사전 조건 보증이 있을 때만.

---

## 3. 꺼내기 — 안전한 접근 API

### 3.1 `isPresent` / `isEmpty` (JDK 11+)
```java
if (opt.isPresent()) {
    System.out.println(opt.get());
}
if (opt.isEmpty()) {
    System.out.println("없음");
}
```
- `get()`은 empty면 `NoSuchElementException`. **테스트용/이행기에서만** 제한적으로.

### 3.2 `orElse`, `orElseGet`, `orElseThrow`
```java
String v1 = opt.orElse("기본값");                // 인자 즉시 평가
String v2 = opt.orElseGet(() -> heavyCompute()); // 필요할 때만 Supplier 실행(지연)
String v3 = opt.orElseThrow(() -> new IllegalArgumentException("없음"));
```
- **비싼 계산/IO**는 `orElseGet`으로. 아래는 **나쁜 예**:
```java
String v = opt.orElse(expensive()); // opt가 present여도 expensive()가 실행됨 (낭비)
```

### 3.3 `ifPresent` / `ifPresentOrElse`(JDK 9+)
```java
opt.ifPresent(v -> log(v));
opt.ifPresentOrElse(
  v -> log(v),
  () -> warn("없음"));
```

---

## 4. 변환 — 함수형 체이닝의 핵심

### 4.1 `map` (T→U)
```java
Optional<Integer> len = opt.map(String::length);
```

### 4.2 `flatMap` (T→Optional<U>) — 중첩 제거
```java
Optional<User> u = repo.findById(id);          // Optional<User>
Optional<Address> a = u.flatMap(User::mainAddress); // Optional<Address>
```
- `map(User::mainAddress)`는 `Optional<Optional<Address>>`가 되므로 **flatMap**을 사용.

### 4.3 `filter` — 조건 불만족 시 empty
```java
Optional<String> onlyH = opt.filter(s -> s.startsWith("H"));
```

### 4.4 `or(Supplier<Optional<T>>)` (JDK 9+)
```java
Optional<String> v = primary()
    .or(() -> fallback())         // primary empty면 fallback Optional 사용
    .orElse("최종 기본값");
```

---

## 5. 예외와 Optional

### 5.1 예외로 전환
```java
User user = repo.findById(id)
    .orElseThrow(() -> new NotFoundException("user: " + id));
```

### 5.2 예외를 Optional로 전환 (파싱/검증)
```java
Optional<Integer> parseInt(String s) {
    try { return Optional.of(Integer.parseInt(s)); }
    catch (NumberFormatException e) { return Optional.empty(); }
}
```

### 5.3 결과 or 예외 매핑
```java
String name = repo.findById(id)
    .map(User::getName)
    .filter(n -> !n.isBlank())
    .orElseThrow(BadRequest::new);
```

---

## 6. 스트림과 Optional

### 6.1 `Optional.stream()` (JDK 9+)
- Optional → 0/1개 스트림. **평탄화의 정석**:
```java
List<String> names = users.stream()
    .map(User::getNicknameOptional)    // Stream<Optional<String>>
    .flatMap(Optional::stream)         // Stream<String>
    .toList();
```

### 6.2 `findFirst/findAny`는 Optional 반환
```java
Optional<Order> first = orders.stream()
    .filter(o -> o.isPending())
    .findFirst();
```

### 6.3 `ofNullable` + `flatMap`
```java
Stream.of("a", null, "b")
    .map(Optional::ofNullable)
    .flatMap(Optional::stream)  // null 제거
    .forEach(System.out::println);
```

---

## 7. 원시형 전용 Optional — 박싱 비용 절감

| 타입 | 생성 | 꺼내기 | 변환 |
|---|---|---|---|
| `OptionalInt` | `OptionalInt.of(7)` | `getAsInt()` | `orElse(int)`, `orElseGet(IntSupplier)` |
| `OptionalLong` | `OptionalLong.of(…)` | `getAsLong()` | … |
| `OptionalDouble` | `OptionalDouble.of(…)` | `getAsDouble()` | … |

> 컬렉션·도메인 반환에는 `Optional<T>`가 편하지만, **핫패스 수치 연산**에는 원시형 전용이 GC·박싱 비용을 줄입니다.

---

## 8. 컬렉션/도메인 설계 시 주의점

### 8.1 “Optional을 필드로” — 피해야 함
- Optional은 **반환 타입**이 주용도입니다.
- 필드에 두면 **직렬화/프레임워크 바인딩/ORM 매핑**에서 잡음이 큽니다.
- 대안: **필드 자체는 nullable**로 두고, **getter는 Optional**로 감쌉니다.
```java
class Person {
    private String nickname; // null 허용
    public Optional<String> getNickname() { return Optional.ofNullable(nickname); }
}
```

### 8.2 메서드 **파라미터** 타입으로 Optional — 가급적 피함
- 오버로드/빌더 패턴/별도 메서드로 의도를 드러내세요.
```java
// 나쁨: void send(Optional<Header> h)
// 대안:
void send();              // 기본
void send(Header h);      // 헤더 제공 버전
```

### 8.3 컬렉션<Optional<T>> — 피함
- 의미가 겹칩니다. **“빈 Optional” 대신 “요소 부재(없음)”**를 사용하세요.
```java
// 나쁨: List<Optional<User>>
// 대안: List<User> (필요 시 필터링 단계에서 제외)
```

---

## 9. 성능/스타일 가이드

### 9.1 비용 감각
- `Optional`은 **작은 객체**이나 객체는 객체입니다.
  **핫루프/대량 구조**에서는 박싱/할당 비용이 누적될 수 있습니다.
- “안전성 vs 비용”에서 **핫패스만** 원시형 Optional/직접 분기로 최적화.

### 9.2 스타일
- `if (opt.isPresent()) { use(opt.get()); }` → 가급적 `ifPresent`/함수형으로
- **heavy 기본값**은 `orElseGet`으로 지연
- **중첩**을 만들지 말 것: `Optional<Optional<T>>`는 설계 냄새
- **값-기반 클래스** 취급: 레퍼런스 동등성(==) 비교/동기화 대상으로 사용하지 않기

---

## 10. 실무 시나리오 6선

### 10.1 DB 조회 (Spring Data JPA 예)
```java
Optional<User> ou = userRepo.findById(id);
User user = ou.orElseThrow(() -> new NotFound("user " + id));
```

### 10.2 구성(설정) 조회 + 캐스팅
```java
String host = props.get("service.host")              // Map<String,Object>
    .map(Object::toString)
    .filter(s -> !s.isBlank())
    .orElse("localhost");
```

### 10.3 HTTP 파라미터 파싱
```java
OptionalInt page = Optional.ofNullable(req.getParameter("page"))
    .flatMap(s -> {
        try { return Optional.of(Integer.parseInt(s)); }
        catch (NumberFormatException e) { return Optional.empty(); }
    })
    .filter(p -> p > 0 && p <= 100)
    .map(OptionalInt::of) // Optional<Integer> → Optional<OptionalInt>
    .orElse(OptionalInt.empty());
```

### 10.4 파서(검증) 체인
```java
record Price(int cents){}
Optional<Price> parsePrice(String s) {
    return Optional.ofNullable(s)
        .map(String::trim)
        .filter(x -> x.matches("\\d+"))
        .map(Integer::parseInt)
        .filter(v -> v >= 0)
        .map(Price::new);
}
```

### 10.5 Spring `ResponseEntity`
```java
return repo.findById(id)
    .map(ResponseEntity::ok)
    .orElseGet(() -> ResponseEntity.notFound().build());
```

### 10.6 캐시 + 재계산
```java
Optional<Value> cached = cache.get(key); // Optional<Value>
Value v = cached.or(() -> compute(key))  // JDK 9+
               .orElseThrow(IllegalStateException::new);
```

---

## 11. 안티패턴 모음 (현상 → 대안)

| 안티패턴 | 왜 문제인가 | 대안 |
|---|---|---|
| 필드 타입을 Optional로 | 직렬화/프레임워크 바인딩 잡음, 의도 중복 | 필드는 nullable, Getter는 Optional |
| 파라미터 타입 Optional | 호출 측 가독성/오버헤드 | 오버로드/빌더/플래그 객체 |
| `get()` 남용 | empty 시 예외(런타임 폭탄) | `orElse*`, `ifPresent*` |
| `orElse(expensive())` | 값 있어도 계산 수행 | `orElseGet(() -> expensive())` |
| 컬렉션에 Optional | 의미 중복, 복잡도 ↑ | 요소 부재로 표현(필터 단계에서 제거) |
| Optional 중첩 | 가독성 저하 | `flatMap`으로 평탄화 |
| `isPresent` + `get` 패턴 | 명령형·장황 | `map/flatMap/ifPresentOrElse` |

---

## 12. 테스트 팁 & 요약 체크리스트

### 12.1 JUnit/assertJ
```java
Optional<String> opt = Optional.of("x");
assertTrue(opt.isPresent());
assertEquals("x", opt.orElseThrow());

import static org.assertj.core.api.Assertions.assertThat;
assertThat(opt).isPresent().contains("x");
assertThat(Optional.empty()).isEmpty();
```

### 12.2 체크리스트
- [ ] 반환 타입에서만 Optional 우선 적용
- [ ] heavy 기본값은 `orElseGet`
- [ ] 변환은 `map/flatMap/filter`로 구성
- [ ] 예외 전환은 `orElseThrow`로 명시
- [ ] 스트림 평탄화는 `Optional.stream()`
- [ ] 핫패스 수치엔 `OptionalInt/Long/Double` 고려
- [ ] 컬렉션/필드/파라미터에 Optional 남용 금지

---

## 부록 A. API 요약 표

| 분류 | 메서드 | 설명 |
|---|---|---|
| 생성 | `of`, `ofNullable`, `empty` | 값 감싸기 |
| 검사 | `isPresent`, `isEmpty`(11+) | 비어있음/있음 |
| 꺼내기 | `get` | empty면 예외(지양) |
| 기본값 | `orElse`, `orElseGet`, `orElseThrow` | 즉시/지연/예외 |
| 소비 | `ifPresent`, `ifPresentOrElse`(9+) | 소비/폴백 |
| 변환 | `map`, `flatMap`, `filter`, `or`(9+) | 변환·중첩 제거·조건·대체 |
| 스트림 | `stream`(9+) | Optional→Stream(0/1) |
| 원시 | `OptionalInt/Long/Double` | 박싱 제거 |

---

## 결론

- `Optional`은 “없을 수 있음”을 **타입**으로 드러내 **API의 의도를 강제**하고 NPE를 구조적으로 줄입니다.
- 반환 타입에서 적극 사용하되, **필드/파라미터/컬렉션에의 남용은 피하고**, 변환/폴백/예외 전환은 `map/flatMap/orElse*`로 명료하게 표현하세요.
- 스트림과 결합한 **평탄화 패턴**(`Optional.stream`)과 **원시형 Optional**을 적재적소에 활용하면 안전성과 성능을 함께 잡을 수 있습니다.
