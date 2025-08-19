---
layout: post
title: Java - Stream API
date: 2025-08-04 17:20:23 +0900
category: Java
---
# Stream API

Java 8부터 도입된 **Stream API**는 컬렉션(Collection)이나 배열(Array) 등 데이터 소스를 **선언적(Declarative)**이고 **함수형 스타일**로 처리할 수 있도록 도와주는 강력한 도구입니다.  
Stream을 이용하면 반복문과 조건문을 통한 **명령형 프로그래밍** 대신, 데이터를 **파이프라인 방식**으로 가공할 수 있습니다.

---

## 1. Stream의 개념
- **데이터의 흐름(Flow)**을 추상화한 객체.
- 데이터 원본(컬렉션, 배열, I/O 채널 등)으로부터 생성되어, **중간 연산**과 **최종 연산**을 거쳐 결과를 생성.
- 데이터 원본을 **변경하지 않음** (Immutable).
- **내부 반복(Internal Iteration)**을 사용하여 멀티코어 환경에서 병렬 처리 가능.

---

## 2. Stream의 주요 특징
1. **데이터 저장 X**  
   Stream은 데이터를 저장하지 않고, 데이터 소스를 기반으로 연산을 수행.
   
2. **한 번만 사용 가능**  
   한 Stream 객체는 한 번의 최종 연산(Terminal Operation) 후 재사용 불가.
   
3. **지연 연산(Lazy Evaluation)**  
   중간 연산은 최종 연산이 호출될 때까지 실제 연산을 수행하지 않음.
   
4. **병렬 처리 지원**  
   `.parallelStream()` 또는 `.parallel()`로 손쉽게 멀티스레드 처리 가능.

---

## 3. Stream 생성 방법

### 3.1 컬렉션에서 생성
```java
import java.util.*;
import java.util.stream.*;

public class StreamCreateExample {
    public static void main(String[] args) {
        List<String> list = Arrays.asList("a", "b", "c");
        
        // 순차 스트림
        Stream<String> stream = list.stream();
        
        // 병렬 스트림
        Stream<String> parallelStream = list.parallelStream();
        
        stream.forEach(System.out::println);
        parallelStream.forEach(System.out::println);
    }
}
```

### 3.2 배열에서 생성
```java
String[] arr = {"a", "b", "c"};
Stream<String> stream = Arrays.stream(arr);
```

### 3.3 값 직접 전달
```java
Stream<Integer> stream = Stream.of(1, 2, 3, 4, 5);
```

### 3.4 무한 스트림
```java
Stream<Integer> evenNumbers = Stream.iterate(0, n -> n + 2);
Stream<Double> randoms = Stream.generate(Math::random);
```

---

## 4. 중간 연산 (Intermediate Operations)
중간 연산은 **새로운 Stream을 반환**하며, 연속적으로 연결하여 파이프라인을 구성할 수 있습니다.

| 연산        | 설명 |
|-------------|------|
| `filter(Predicate)` | 조건에 맞는 요소만 필터링 |
| `map(Function)` | 각 요소를 변환 |
| `flatMap(Function)` | 중첩 구조를 평면화 |
| `distinct()` | 중복 제거 |
| `sorted()` | 정렬 |
| `limit(n)` | 앞에서부터 n개만 가져오기 |
| `skip(n)` | 앞에서 n개 건너뛰기 |
| `peek(Consumer)` | 중간에 요소를 확인(디버깅용) |

**예시**
```java
List<String> names = Arrays.asList("Kim", "Lee", "Park", "Lee");
names.stream()
     .filter(name -> name.startsWith("L"))
     .distinct()
     .sorted()
     .forEach(System.out::println);
```

---

## 5. 최종 연산 (Terminal Operations)
최종 연산은 **Stream을 소모하여 결과를 생성**합니다.

| 연산        | 설명 |
|-------------|------|
| `forEach(Consumer)` | 각 요소 처리 |
| `toArray()` | 배열로 변환 |
| `collect(Collector)` | 컬렉션으로 변환 |
| `reduce()` | 누적 연산 수행 |
| `count()` | 요소 개수 |
| `anyMatch()`, `allMatch()`, `noneMatch()` | 조건 검사 |
| `findFirst()`, `findAny()` | 요소 검색 |

**예시**
```java
List<String> names = Arrays.asList("Kim", "Lee", "Park");
long count = names.stream()
                  .filter(name -> name.length() > 3)
                  .count();
System.out.println(count);
```

---

## 6. 매핑과 평면화 (map vs flatMap)

- `map` : 요소를 1:1로 변환
- `flatMap` : 요소를 여러 개로 변환한 뒤 평면화

```java
List<List<String>> list = Arrays.asList(
    Arrays.asList("a", "b"),
    Arrays.asList("c", "d")
);

list.stream()
    .flatMap(Collection::stream)
    .forEach(System.out::println); // a b c d
```

---

## 7. 병렬 스트림 (Parallel Stream)
멀티코어 CPU에서 병렬 처리 가능
```java
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5);

int sum = numbers.parallelStream()
                 .mapToInt(Integer::intValue)
                 .sum();
System.out.println(sum);
```
> 병렬 스트림은 데이터 크기가 크고 CPU 바운드 연산일 때 유리하지만,  
> 스레드 컨텍스트 스위칭 비용으로 인해 항상 빠르지는 않음.

---

## 8. collect()와 Collectors
Stream의 결과를 다양한 형태로 변환하는데 사용
```java
List<String> names = Arrays.asList("Kim", "Lee", "Park");

String joined = names.stream()
                     .collect(Collectors.joining(", ")); // "Kim, Lee, Park"

Map<Integer, List<String>> grouped =
    names.stream()
         .collect(Collectors.groupingBy(String::length));
```

---

## 9. Stream과 Optional
`findFirst`, `findAny`, `max`, `min` 등의 메서드는 결과가 없을 수 있으므로 **Optional**로 반환
```java
Optional<String> result = names.stream()
                               .filter(n -> n.startsWith("P"))
                               .findFirst();

result.ifPresent(System.out::println);
```

---

## 10. 주의사항
- Stream은 재사용 불가.
- 병렬 스트림은 성능 테스트 후 사용.
- I/O 기반 Stream은 반드시 닫아야 함(try-with-resources 권장).

---

## 11. 요약
| 항목        | 설명 |
|-------------|------|
| 생성        | 컬렉션, 배열, 값, 무한 스트림 |
| 중간 연산   | filter, map, distinct, sorted, limit, skip, flatMap |
| 최종 연산   | forEach, collect, reduce, count, match, findFirst |
| 특징        | 내부 반복, 불변성, 지연 연산, 병렬 처리 지원 |