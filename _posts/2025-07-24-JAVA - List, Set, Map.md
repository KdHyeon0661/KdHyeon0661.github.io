---
layout: post
title: Java - List, Set, Map
date: 2025-07-24 17:20:23 +0900
category: Java
---
# Java 컬렉션 프레임워크 개요: List, Set, Map

Java의 **컬렉션 프레임워크**는 데이터를 효율적으로 저장하고 조작하기 위한 표준화된 자료구조 집합입니다. 그 중에서도 **List**, **Set**, **Map**은 가장 널리 사용되는 핵심 인터페이스입니다. 각각의 특징과 구현 클래스들을 중심으로 자세히 살펴보겠습니다.

---

## 1. List

### ✅ 개요

- **순서가 있는 데이터 집합**
- **중복 허용**
- 인덱스를 이용해 요소에 접근 가능

### ✅ 주요 구현 클래스

| 클래스명 | 특징 |
|----------|------|
| `ArrayList` | 배열 기반, 검색이 빠름, 삽입/삭제는 느림 |
| `LinkedList` | 연결 리스트 기반, 삽입/삭제가 빠름 |
| `Vector` | 동기화 지원, 레거시 클래스 |
| `Stack` | 후입선출(LIFO) 구조를 따름 (`Vector` 상속)

### ✅ 예제

```java
import java.util.*;

public class ListExample {
    public static void main(String[] args) {
        List<String> list = new ArrayList<>();

        list.add("apple");
        list.add("banana");
        list.add("apple"); // 중복 허용

        System.out.println(list.get(0)); // "apple"
        System.out.println(list.size()); // 3

        list.remove("banana");
        System.out.println(list); // [apple, apple]
    }
}
```

---

## 2. Set

### ✅ 개요

- **순서를 보장하지 않는** 데이터 집합
- **중복 불허**
- 주로 **수학적 집합**의 개념에 기반

### ✅ 주요 구현 클래스

| 클래스명 | 특징 |
|----------|------|
| `HashSet` | 해시 기반, 순서 없음, 가장 빠름 |
| `LinkedHashSet` | 입력 순서 유지 |
| `TreeSet` | 이진 트리 기반, 정렬된 순서로 유지 (오름차순) |

### ✅ 예제

```java
import java.util.*;

public class SetExample {
    public static void main(String[] args) {
        Set<String> set = new HashSet<>();

        set.add("apple");
        set.add("banana");
        set.add("apple"); // 중복 무시

        System.out.println(set); // [banana, apple] (순서는 보장되지 않음)
        System.out.println(set.contains("banana")); // true
    }
}
```

---

## 3. Map

### ✅ 개요

- **키(key)와 값(value)**의 쌍으로 구성된 구조
- **키는 중복 불가**, **값은 중복 가능**
- 키를 이용한 빠른 데이터 조회

### ✅ 주요 구현 클래스

| 클래스명 | 특징 |
|----------|------|
| `HashMap` | 해시 기반, 가장 일반적인 Map |
| `LinkedHashMap` | 입력 순서 유지 |
| `TreeMap` | 키 기준 정렬 |
| `Hashtable` | 동기화 지원, 레거시 클래스 |

### ✅ 예제

```java
import java.util.*;

public class MapExample {
    public static void main(String[] args) {
        Map<String, Integer> map = new HashMap<>();

        map.put("apple", 1);
        map.put("banana", 2);
        map.put("apple", 3); // 키 중복 → 값 덮어쓰기

        System.out.println(map.get("apple")); // 3
        System.out.println(map.containsKey("banana")); // true
        System.out.println(map.keySet()); // [banana, apple]
    }
}
```

---

## 4. 정리: List vs Set vs Map

| 특징 | List | Set | Map |
|------|------|-----|-----|
| 순서 | 유지함 | 유지하지 않음 (`HashSet`), 유지함(`LinkedHashSet`) | 키 순서를 따름(`TreeMap`) |
| 중복 | 허용 | 허용하지 않음 | 키는 허용하지 않음, 값은 허용 |
| 접근 방식 | 인덱스 | 반복자 | 키 |
| 사용 목적 | 순차적 데이터 저장 | 유일한 데이터 저장 | 키-값 쌍 저장 |

---

## 5. 언제 무엇을 써야 할까?

- ✅ **순서가 중요하고 중복도 허용**해야 할 경우 → `List`
- ✅ **중복이 없어야 하고 빠른 검색**이 중요할 경우 → `HashSet`
- ✅ **키를 기준으로 값을 관리**하고 싶을 경우 → `HashMap`

이처럼 상황에 따라 `List`, `Set`, `Map`은 각각의 목적과 장점이 다르며, Java 컬렉션 프레임워크의 기본기를 형성하는 핵심 구성 요소입니다.