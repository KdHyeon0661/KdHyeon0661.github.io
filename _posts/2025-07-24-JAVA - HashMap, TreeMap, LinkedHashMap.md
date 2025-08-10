---
layout: post
title: Java - HashMap, TreeMap, LinkedHashMap
date: 2025-07-24 19:20:23 +0900
category: Java
---
# Java의 Map 구현 클래스: `HashMap`, `TreeMap`, `LinkedHashMap` 자세히 정리

Java의 `Map` 인터페이스는 **키-값 쌍(key-value pair)**으로 데이터를 저장하는 자료구조입니다. `Set`이 키의 집합이라면, `Map`은 키와 값의 쌍을 저장하는 형태입니다.

자바에서 `Map`을 구현한 대표적인 클래스는 다음과 같습니다:

- `HashMap`
- `TreeMap`
- `LinkedHashMap`

각 클래스는 내부 구조, 정렬 여부, 성능, 특징이 다르며 사용 목적에 따라 선택됩니다.

---

## 1. Map 인터페이스 개요

```java
Map<K, V>
```

- **K**: Key의 타입
- **V**: Value의 타입
- **중복된 키 허용 X**, **중복된 값은 허용 O**
- 주요 메서드:
  - `put(K key, V value)`
  - `get(Object key)`
  - `remove(Object key)`
  - `containsKey(Object key)`
  - `containsValue(Object value)`
  - `keySet()`, `values()`, `entrySet()`

---

## 2. `HashMap`

### ✅ 개요

- 가장 일반적으로 사용하는 Map 구현체
- **정렬되지 않음 (순서 보장 X)**
- 내부적으로 **배열 + 연결 리스트 (또는 트리)** 로 구성
- **null 키 1개 허용, null 값 여러 개 허용**

### ✅ 내부 구조

- 내부적으로 `배열 + 해시 함수 + 체이닝(LinkedList)` 구조
- Java 8 이후, **충돌이 많은 버킷은 Tree 구조(Red-Black Tree)** 사용

### ✅ 시간 복잡도

| 연산 | 평균 시간 복잡도 |
|------|-----------------|
| 삽입 | O(1) |
| 삭제 | O(1) |
| 검색 | O(1) |

### ✅ 예제

```java
import java.util.HashMap;

public class HashMapExample {
    public static void main(String[] args) {
        HashMap<String, Integer> map = new HashMap<>();

        map.put("apple", 3);
        map.put("banana", 2);
        map.put("orange", 5);

        System.out.println(map.get("apple")); // 3
        System.out.println(map.containsKey("banana")); // true
    }
}
```

---

## 3. `TreeMap`

### ✅ 개요

- **정렬된 Map** (자동으로 키를 오름차순 정렬)
- 내부적으로 **Red-Black Tree (이진 탐색 트리)** 사용
- **null 키 허용 X**, null 값 허용 O

### ✅ 정렬 방식

- 기본적으로 키의 **자연 순서 (Comparable)** 기준 정렬
- 또는 생성자에서 `Comparator` 전달 가능

### ✅ 시간 복잡도

| 연산 | 시간 복잡도 |
|------|-------------|
| 삽입 | O(log n) |
| 삭제 | O(log n) |
| 검색 | O(log n) |

### ✅ 예제

```java
import java.util.TreeMap;

public class TreeMapExample {
    public static void main(String[] args) {
        TreeMap<String, Integer> map = new TreeMap<>();

        map.put("banana", 2);
        map.put("apple", 3);
        map.put("cherry", 5);

        System.out.println(map); // {apple=3, banana=2, cherry=5}
        System.out.println(map.firstKey()); // apple
        System.out.println(map.lastEntry()); // cherry=5
    }
}
```

---

## 4. `LinkedHashMap`

### ✅ 개요

- `HashMap` 기반이지만, **입력 순서를 유지**
- 내부적으로 **해시 테이블 + 이중 연결 리스트** 구성
- **null 키 1개 허용, null 값 허용**

### ✅ 주요 특징

- **삽입 순서 유지**
- Java 1.4부터 제공
- **LRU 캐시** 구현 시 사용 가능 (`accessOrder` 플래그 사용)

### ✅ 시간 복잡도

| 연산 | 평균 시간 복잡도 |
|------|----------------|
| 삽입 | O(1) |
| 삭제 | O(1) |
| 검색 | O(1) |

### ✅ 예제

```java
import java.util.LinkedHashMap;

public class LinkedHashMapExample {
    public static void main(String[] args) {
        LinkedHashMap<String, Integer> map = new LinkedHashMap<>();

        map.put("apple", 3);
        map.put("banana", 2);
        map.put("cherry", 5);

        System.out.println(map); // {apple=3, banana=2, cherry=5}
    }
}
```

---

## 5. 세 클래스 비교표

| 항목 | HashMap | TreeMap | LinkedHashMap |
|------|---------|---------|----------------|
| 내부 구조 | 해시 테이블 | Red-Black Tree | 해시 테이블 + 연결 리스트 |
| 정렬 여부 | 없음 | 키의 정렬된 순서 | 입력 순서 유지 |
| 키 null 허용 | O (1개) | X | O (1개) |
| 값 null 허용 | O | O | O |
| 성능 (삽입/검색/삭제) | O(1) | O(log n) | O(1) |
| 순서 유지 | X | 정렬 순 | 입력 순서 |

---

## 6. 정렬 순서 커스터마이징 (`TreeMap`)

```java
import java.util.*;

public class CustomComparatorTreeMap {
    public static void main(String[] args) {
        TreeMap<String, Integer> map = new TreeMap<>(Comparator.reverseOrder());

        map.put("apple", 3);
        map.put("banana", 2);
        map.put("cherry", 5);

        System.out.println(map); // {cherry=5, banana=2, apple=3}
    }
}
```

---

## 7. 요약

| 사용 목적 | 추천 클래스 |
|-----------|--------------|
| 빠른 성능 (순서 불필요) | `HashMap` |
| 키 정렬된 Map 필요 | `TreeMap` |
| 삽입 순서 유지 필요 | `LinkedHashMap` |
| LRU 캐시 구현 | `LinkedHashMap (accessOrder = true)` |

---

## 8. LRU 캐시 예시 (LinkedHashMap 활용)

```java
import java.util.*;

class LRUCache<K, V> extends LinkedHashMap<K, V> {
    private final int capacity;

    public LRUCache(int capacity) {
        super(capacity, 0.75f, true); // accessOrder = true
        this.capacity = capacity;
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > capacity;
    }
}
```

---

## 🔚 결론

- `HashMap`: 일반적인 용도에 가장 빠른 선택
- `TreeMap`: 정렬된 데이터가 필요한 경우
- `LinkedHashMap`: 순서가 중요한 경우 or 캐시 구조

각 Map 클래스는 쓰임새가 분명하므로 요구 사항에 맞게 선택하는 것이 핵심입니다.