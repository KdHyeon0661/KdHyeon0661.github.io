---
layout: post
title: Java - Iterator와 foreach
date: 2025-07-24 21:20:23 +0900
category: Java
---
# Iterator와 foreach에 대해 자세하게 정리

자바에서 컬렉션 프레임워크는 데이터를 저장하고 처리하는 데 매우 중요한 역할을 합니다. 이 데이터를 순회(iterate)하기 위한 대표적인 방법 두 가지는 **Iterator**와 **foreach**입니다. 이 둘은 내부 동작과 사용 목적, 코드 스타일 면에서 차이가 있으며 상황에 따라 적절히 선택해야 합니다.

---

## 1. Iterator란?

**Iterator**는 컬렉션 객체(예: `List`, `Set`)의 요소를 **하나씩 순차적으로 접근**할 수 있는 객체입니다. `java.util.Iterator` 인터페이스로 제공되며, `Collection` 인터페이스를 구현한 클래스에서 `iterator()` 메서드를 통해 얻을 수 있습니다.

### 주요 메서드

| 메서드 | 설명 |
|--------|------|
| `boolean hasNext()` | 다음 요소가 존재하는지 확인 |
| `E next()` | 다음 요소를 반환 |
| `void remove()` | 현재 요소를 삭제 (선택적 연산) |

### 사용 예제

```java
import java.util.*;

public class IteratorExample {
    public static void main(String[] args) {
        List<String> names = Arrays.asList("Java", "Python", "C++");
        Iterator<String> iterator = names.iterator();

        while (iterator.hasNext()) {
            String name = iterator.next();
            System.out.println(name);
        }
    }
}
```

### 특징
- 요소를 제거할 수 있음 (`remove()` 지원)
- 모든 `Collection` 구현체에서 사용 가능
- 반복 도중 `ConcurrentModificationException` 주의 (특히 `ArrayList`에서 `remove()` 사용 시)

---

## 2. foreach 문 (향상된 for문)

**foreach**는 Java 5부터 도입된 **향상된 for문**으로, 컬렉션이나 배열 요소를 간단하게 순회하는 문법입니다. 내부적으로는 `Iterator`를 사용하지만 코드가 훨씬 간결합니다.

### 문법

```java
for (자료형 변수 : 컬렉션/배열) {
    // 반복 실행할 코드
}
```

### 사용 예제

```java
import java.util.*;

public class ForeachExample {
    public static void main(String[] args) {
        List<String> fruits = Arrays.asList("Apple", "Banana", "Cherry");

        for (String fruit : fruits) {
            System.out.println(fruit);
        }
    }
}
```

### 특징
- 코드가 간결하고 가독성이 좋음
- 요소 제거 불가능 (`remove()` 제공 안 됨)
- 인덱스 접근 불가능 (`List.get(i)` 같은 연산 불가)
- 내부적으로 `Iterator` 사용함

---

## 3. Iterator vs foreach 비교

| 항목 | Iterator | foreach |
|------|----------|---------|
| 제거 기능 | 가능 (`remove()`) | 불가능 |
| 코드 가독성 | 다소 복잡 | 간결 |
| 사용 대상 | 모든 컬렉션 | 배열 및 `Iterable` |
| 인덱스 접근 | 불가능 | 불가능 |
| 내부 구조 | 명시적 반복자 | 내부적으로 `Iterator` 사용 |

---

## 4. 주의 사항

### 1. remove()는 꼭 next() 호출 이후에만 사용 가능
```java
Iterator<String> iterator = list.iterator();
while (iterator.hasNext()) {
    String item = iterator.next();
    if (item.equals("삭제할값")) {
        iterator.remove(); // OK
    }
}
```

`next()` 없이 `remove()`를 호출하면 `IllegalStateException` 발생

### 2. foreach에서는 remove() 사용 불가
```java
for (String item : list) {
    if (item.equals("삭제할값")) {
        list.remove(item); // ConcurrentModificationException 발생 가능
    }
}
```

- 이 경우는 `Iterator`로 바꾸는 것이 안전함

---

## 5. Stream API와의 관계

Java 8 이후에는 `Stream`을 사용한 반복도 많이 쓰입니다.

```java
list.stream().forEach(item -> System.out.println(item));
```

`Stream`은 병렬 처리, 필터링, 매핑 등 고급 처리가 가능하므로 단순 순회보다는 더 많은 기능을 제공합니다. 하지만 단순 순회에는 `foreach`가 여전히 유용합니다.

---

## 결론

- **Iterator**는 컬렉션 요소를 안전하게 제거하며 순회할 수 있음
- **foreach**는 단순 순회에 적합하고 코드가 깔끔함
- 컬렉션을 **변경하지 않고 읽기만 할 때는 foreach**, 요소 제거가 필요하면 **Iterator**를 사용
