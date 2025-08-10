---
layout: post
title: Java - Math, Arrays, Collections
date: 2025-07-24 16:20:23 +0900
category: Java
---
# Math, Arrays, Collections에 대해 자세하게

## 1. `Math` 클래스

`java.lang.Math` 클래스는 수학적 연산을 위한 다양한 메서드를 제공합니다. 이 클래스는 인스턴스를 생성하지 않고, 모든 메서드는 `static`으로 선언되어 있어 바로 사용할 수 있습니다.

### 주요 메서드

| 메서드 | 설명 |
|--------|------|
| `Math.abs(x)` | 절댓값 반환 |
| `Math.max(a, b)` | 최대값 반환 |
| `Math.min(a, b)` | 최소값 반환 |
| `Math.pow(a, b)` | a의 b 제곱 |
| `Math.sqrt(x)` | 제곱근 |
| `Math.random()` | 0.0 이상 1.0 미만의 난수 생성 |
| `Math.round(x)` | 반올림 |
| `Math.floor(x)` | 내림 |
| `Math.ceil(x)` | 올림 |

### 예제

```java
public class MathExample {
    public static void main(String[] args) {
        System.out.println(Math.abs(-5)); // 5
        System.out.println(Math.max(10, 20)); // 20
        System.out.println(Math.pow(2, 3)); // 8.0
        System.out.println(Math.random()); // 0.0 <= x < 1.0
    }
}
```

---

## 2. `Arrays` 클래스

`java.util.Arrays` 클래스는 배열을 다루기 위한 다양한 유틸리티 메서드를 제공합니다.

### 주요 메서드

| 메서드 | 설명 |
|--------|------|
| `Arrays.sort(array)` | 배열 정렬 |
| `Arrays.toString(array)` | 배열을 문자열로 출력 |
| `Arrays.equals(a1, a2)` | 두 배열이 동일한지 비교 |
| `Arrays.copyOf(arr, length)` | 배열 복사 |
| `Arrays.fill(arr, val)` | 배열을 특정 값으로 채움 |
| `Arrays.binarySearch(arr, key)` | 정렬된 배열에서 이진탐색 수행 |

### 예제

```java
import java.util.Arrays;

public class ArraysExample {
    public static void main(String[] args) {
        int[] arr = {5, 2, 4, 1, 3};

        Arrays.sort(arr);
        System.out.println(Arrays.toString(arr)); // [1, 2, 3, 4, 5]

        int[] copy = Arrays.copyOf(arr, arr.length);
        System.out.println(Arrays.equals(arr, copy)); // true
    }
}
```

---

## 3. `Collections` 클래스

`java.util.Collections` 클래스는 **List, Set, Map** 같은 컬렉션 프레임워크를 위한 유틸리티 메서드를 제공합니다. 

컬렉션 프레임워크는 객체들을 효율적으로 저장하고, 검색, 정렬, 수정하는 기능을 제공합니다.

### 주요 메서드

| 메서드 | 설명 |
|--------|------|
| `Collections.sort(list)` | 리스트 정렬 |
| `Collections.reverse(list)` | 리스트 반전 |
| `Collections.shuffle(list)` | 요소 무작위 섞기 |
| `Collections.max(list)` | 최대값 반환 |
| `Collections.min(list)` | 최소값 반환 |
| `Collections.frequency(list, obj)` | 요소 출현 횟수 |
| `Collections.swap(list, i, j)` | 두 인덱스 값 교환 |
| `Collections.fill(list, obj)` | 리스트 전체를 obj로 채움 |

### 예제

```java
import java.util.*;

public class CollectionsExample {
    public static void main(String[] args) {
        List<String> list = new ArrayList<>(Arrays.asList("apple", "banana", "cherry"));

        Collections.sort(list);
        System.out.println(list); // [apple, banana, cherry]

        Collections.reverse(list);
        System.out.println(list); // [cherry, banana, apple]

        Collections.shuffle(list);
        System.out.println(list); // 무작위 순서
    }
}
```

---

## 정리

| 클래스 | 주로 다루는 것 | 특징 |
|--------|----------------|------|
| `Math` | 수학 함수 | static 메서드, 간단한 연산 |
| `Arrays` | 배열 유틸리티 | 정렬, 비교, 복사 등 |
| `Collections` | 컬렉션 유틸리티 | 리스트/셋/맵 등의 처리 보조 |

이 세 클래스는 Java에서 배열과 컬렉션을 처리하거나 수학적 계산을 할 때 매우 자주 사용되며, 표준 라이브러리의 강력함을 보여주는 핵심 도구들입니다.