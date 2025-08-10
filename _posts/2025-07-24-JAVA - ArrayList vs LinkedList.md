---
layout: post
title: Java - ArrayList vs LinkedList
date: 2025-07-24 18:20:23 +0900
category: Java
---
# Java 컬렉션 상세: ArrayList vs LinkedList

Java에서 `List` 인터페이스를 구현한 대표적인 두 클래스는 `ArrayList`와 `LinkedList`입니다. 두 클래스는 모두 **순서를 유지하고**, **중복된 요소를 허용**하지만, 내부 구현 방식과 성능에서 중요한 차이가 있습니다. 이 문서에서는 그 차이점을 구조, 주요 메서드, 시간 복잡도, 예제 중심으로 자세히 살펴봅니다.

---

## 1. ArrayList

### ✅ 개요

- **배열 기반**의 List 구현체
- 동적 배열(Dynamic Array) 구조
- 요소들은 **연속된 인덱스**에 저장됨
- 검색 속도가 빠르고, 요소 삽입/삭제는 느릴 수 있음

### ✅ 특징

| 특징 | 설명 |
|------|------|
| 내부 구조 | Object 배열 |
| 검색 속도 | 빠름 (`O(1)` 인덱스 접근) |
| 삽입/삭제 속도 | 느림 (중간 요소를 이동해야 함) |
| 메모리 | 공간이 더 필요하면 내부 배열 확장 |

### ✅ 주요 메서드

- `add(E e)` : 리스트 끝에 요소 추가
- `add(int index, E e)` : 특정 위치에 요소 삽입
- `get(int index)` : 요소 반환
- `remove(int index)` : 요소 삭제
- `set(int index, E e)` : 요소 교체
- `size()` : 리스트 크기

### ✅ 예제

```java
import java.util.ArrayList;

public class ArrayListExample {
    public static void main(String[] args) {
        ArrayList<String> list = new ArrayList<>();

        list.add("apple");
        list.add("banana");
        list.add("cherry");

        System.out.println(list.get(1)); // banana

        list.remove(0); // "apple" 삭제
        list.add(1, "date");

        System.out.println(list); // [banana, date, cherry]
    }
}
```

---

## 2. LinkedList

### ✅ 개요

- **이중 연결 리스트(Doubly Linked List)** 기반의 List 구현체
- 각 노드는 데이터와 **앞/뒤 노드에 대한 참조**를 가짐
- 중간 삽입/삭제가 빠르고, 인덱스 기반 접근은 느림

### ✅ 특징

| 특징 | 설명 |
|------|------|
| 내부 구조 | 노드(Node) 연결 |
| 검색 속도 | 느림 (`O(n)` 인덱스 접근) |
| 삽입/삭제 속도 | 빠름 (참조만 변경하면 됨) |
| 메모리 | 노드당 참조 2개 필요 (앞, 뒤) |

### ✅ 주요 메서드

- `add(E e)` : 리스트 끝에 추가
- `addFirst(E e)`, `addLast(E e)` : 양쪽 끝에 추가
- `removeFirst()`, `removeLast()` : 양쪽 끝 삭제
- `get(int index)` : 특정 위치 요소 반환
- `poll()`, `peek()` : 큐처럼 사용 가능
- `size()` : 리스트 크기

### ✅ 예제

```java
import java.util.LinkedList;

public class LinkedListExample {
    public static void main(String[] args) {
        LinkedList<String> list = new LinkedList<>();

        list.add("apple");
        list.add("banana");
        list.addFirst("cherry");

        System.out.println(list); // [cherry, apple, banana]

        list.removeLast(); // "banana" 삭제
        System.out.println(list.get(1)); // apple
    }
}
```

---

## 3. 성능 비교

| 연산 | ArrayList | LinkedList |
|------|-----------|------------|
| 인덱스로 접근 | **O(1)** | O(n) |
| 요소 삽입 (끝에) | 평균 O(1), 최악 O(n) | O(1) |
| 요소 삽입 (중간) | O(n) | **O(1)** (노드 참조만 바꾸면 됨) |
| 요소 삭제 | O(n) | O(1) (노드 참조만 해제) |
| 메모리 사용량 | 낮음 | 높음 (노드 포인터 2개 추가) |

---

## 4. 선택 기준

| 상황 | 추천 클래스 | 이유 |
|------|--------------|------|
| 인덱스를 자주 사용하여 요소 접근 | `ArrayList` | 인덱스 기반 접근이 빠름 |
| 삽입/삭제가 빈번한 경우 | `LinkedList` | 중간 삽입/삭제가 빠름 |
| 큐나 덱 구조로 사용할 때 | `LinkedList` | `addFirst`, `removeLast` 등 풍부한 메서드 |
| 메모리 효율이 중요한 경우 | `ArrayList` | 참조가 없고 간단한 배열 기반 |

---

## 5. 요약

| 구분 | ArrayList | LinkedList |
|------|-----------|------------|
| 자료구조 | 배열 | 이중 연결 리스트 |
| 속도 | 접근은 빠름 | 삽입/삭제는 빠름 |
| 메모리 효율 | 상대적으로 좋음 | 상대적으로 나쁨 |
| 용도 | 읽기 중심 | 삽입/삭제 중심 |

`ArrayList`와 `LinkedList`는 각각의 장단점이 있으므로, 상황에 따라 적절히 선택하여 사용하는 것이 중요합니다.