---
layout: post
title: Java - HashSet vs TreeSet
date: 2025-07-24 20:20:23 +0900
category: Java
---
# Java Set 구현 클래스: `HashSet` vs `TreeSet` 상세 정리

Java에서 `Set`은 **중복을 허용하지 않는 자료구조**입니다. 대표적인 구현체는 `HashSet`과 `TreeSet`이며, 각각 **해시 기반**, **이진 탐색 트리 기반**으로 동작합니다. 이 문서에서는 이 두 클래스를 구조, 동작 원리, 시간 복잡도, 예제 중심으로 자세히 비교합니다.

---

## 1. `Set` 인터페이스 개요

- **중복을 허용하지 않음**
- **순서 없음 또는 정렬된 순서 제공**
- 주요 메서드:
  - `add(E e)`
  - `remove(Object o)`
  - `contains(Object o)`
  - `size()`
  - `isEmpty()`

---

## 2. `HashSet`

### ✅ 개요

- `HashMap`을 내부적으로 사용하여 구현된 Set
- **순서를 보장하지 않음**
- **null 값 허용** (단, 한 개만)

### ✅ 내부 구조

- `HashMap`의 key에 요소를 저장
- 해시 함수를 이용해 데이터를 빠르게 검색

### ✅ 시간 복잡도

| 연산 | 시간 복잡도 |
|------|--------------|
| 삽입 | 평균 O(1) |
| 삭제 | 평균 O(1) |
| 검색 | 평균 O(1) |

> ⚠ 해시 충돌이 많거나 해시 함수가 좋지 않으면 성능이 O(n)까지 저하될 수 있음.

### ✅ 예제

```java
import java.util.HashSet;

public class HashSetExample {
    public static void main(String[] args) {
        HashSet<String> set = new HashSet<>();

        set.add("apple");
        set.add("banana");
        set.add("cherry");
        set.add("apple"); // 중복 추가, 무시됨

        System.out.println(set); // [banana, cherry, apple] (순서 보장 X)
        System.out.println(set.contains("banana")); // true
    }
}
```

---

## 3. `TreeSet`

### ✅ 개요

- `NavigableSet`을 구현한 클래스
- 내부적으로 **Red-Black Tree** 기반의 정렬 이진 트리로 구성
- **자동 정렬된 순서로 저장**
- **null 요소 허용하지 않음**

### ✅ 내부 구조

- 이진 탐색 트리 (Red-Black Tree)
- 삽입될 때마다 자동으로 정렬
- 요소가 `Comparable` 인터페이스를 구현해야 함 (또는 `Comparator` 필요)

### ✅ 시간 복잡도

| 연산 | 시간 복잡도 |
|------|--------------|
| 삽입 | O(log n) |
| 삭제 | O(log n) |
| 검색 | O(log n) |

### ✅ 예제

```java
import java.util.TreeSet;

public class TreeSetExample {
    public static void main(String[] args) {
        TreeSet<String> set = new TreeSet<>();

        set.add("banana");
        set.add("apple");
        set.add("cherry");

        System.out.println(set); // [apple, banana, cherry] (자동 정렬)

        System.out.println(set.first()); // apple
        System.out.println(set.last()); // cherry
    }
}
```

---

## 4. 주요 차이점 비교

| 항목 | HashSet | TreeSet |
|------|---------|---------|
| 내부 구조 | HashMap 기반 | Red-Black Tree 기반 |
| 정렬 여부 | 없음 (순서 불확실) | 있음 (자동 정렬) |
| 성능 | O(1) (평균) | O(log n) |
| null 허용 | 가능 (1개) | 불가 |
| 사용 조건 | 해시 코드와 equals 필요 | Comparable 또는 Comparator 필요 |

---

## 5. 사용 시기

| 상황 | 추천 클래스 |
|------|-------------|
| **빠른 검색/삽입/삭제**가 필요하고 순서가 중요하지 않음 | `HashSet` |
| **자동 정렬된 순서**가 필요하거나 범위 검색이 중요함 | `TreeSet` |

---

## 6. 정렬 기준 커스터마이징 (TreeSet)

```java
import java.util.TreeSet;
import java.util.Comparator;

public class CustomSortExample {
    public static void main(String[] args) {
        TreeSet<String> set = new TreeSet<>(Comparator.reverseOrder());

        set.add("apple");
        set.add("banana");
        set.add("cherry");

        System.out.println(set); // [cherry, banana, apple]
    }
}
```

---

## 7. 요약

| 구분 | HashSet | TreeSet |
|------|---------|---------|
| 정렬 여부 | 정렬되지 않음 | 자동 정렬 |
| 속도 | 빠름 (O(1)) | 느림 (O(log n)) |
| null 허용 | 가능 | 불가능 |
| 내부 구조 | 해시 기반 | 트리 기반 |
| 요구사항 | `hashCode()`, `equals()` | `compareTo()` 또는 `Comparator` |

---

`HashSet`은 속도 중심의 비정렬된 집합을 원할 때,  
`TreeSet`은 정렬된 데이터와 탐색 중심의 작업에 유리합니다.