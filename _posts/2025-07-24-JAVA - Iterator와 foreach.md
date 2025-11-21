---
layout: post
title: Java - Iterator와 foreach
date: 2025-07-24 21:20:23 +0900
category: Java
---
# Java의 Iterator와 foreach

## Iterator 기본기

### 개념

`Iterator<E>`는 컬렉션 요소를 **순차 접근**하기 위한 표준 인터페이스입니다. 모든 `Collection` 구현체에서 `iterator()`로 획득합니다.

### 핵심 메서드

| 메서드 | 설명 | 예외/비고 |
|---|---|---|
| `boolean hasNext()` | 다음 요소 존재 여부 |  |
| `E next()` | 다음 요소 반환 및 커서 전진 | 더 이상 요소가 없으면 `NoSuchElementException` |
| `void remove()` | **직전 `next()`로 반환한 요소** 삭제 | 선택적 연산(미지원 시 `UnsupportedOperationException`), **`next()` 호출 전엔 `IllegalStateException`** |
| `default void forEachRemaining(Consumer<? super E> action)` | 남은 요소에 일괄 동작 | Java 8+ |

### 기본 사용

```java
import java.util.*;

public class IteratorExample {
  public static void main(String[] args) {
    List<String> names = Arrays.asList("Java", "Python", "C++");
    Iterator<String> it = names.iterator();
    while (it.hasNext()) {
      String s = it.next();
      System.out.println(s);
    }
  }
}
```

---

## — 문법은 간결, 내부는 Iterator

### 문법과 대상

```java
for (타입 변수 : 배열_or_Iterable) {
  // 처리
}
```
- **배열**: 컴파일러가 **인덱스 기반 루프**로 변환.
- **`Iterable`**: 내부적으로 `iterator()`를 호출해 Iterator 기반 루프로 변환.

### 간단 예

```java
import java.util.*;

public class ForeachExample {
  public static void main(String[] args) {
    List<String> fruits = Arrays.asList("Apple", "Banana", "Cherry");
    for (String f : fruits) {
      System.out.println(f);
    }
  }
}
```

### 특징

- **가독성** 우수, **인덱스** 접근 불가.
- 루프 변수에 값을 할당해도 **원본을 수정하지 않음**.
- 루프 중 컬렉션 요소를 **직접 제거하면 위험**(CME 가능) → §4 참고.

---

## Iterator vs foreach — 선택 기준

| 항목 | Iterator | foreach(향상된 for) |
|---|---|---|
| 삭제 | **가능**(`iterator.remove()`) | **불가**(직접 삭제 시 CME 위험) |
| 가독성 | 상대적으로 장황 | **간결** |
| 인덱스 활용 | 불가(커서만) | 불가 |
| 대상 | 모든 `Collection` | **배열 + Iterable** |
| 내부 동작 | 명시적 Iterator | 배열은 인덱스, 컬렉션은 Iterator |

> **읽기 전용 순회** → foreach 권장.
> **순회하며 안전하게 삭제/치환/삽입** → Iterator 또는 `ListIterator`.

---

## 실패-안전(fail-fast)과 ConcurrentModificationException

### 왜 발생하나?

대부분의 비동시성 컬렉션은 **구조적 변경(추가/삭제)** 시 내부 수정 횟수(`modCount`)를 증가시킵니다. Iterator는 생성 당시의 `modCount`를 캐싱하고, 순회 중 값이 달라지면 **최대한 빨리** `ConcurrentModificationException`(CME)을 던집니다(보장 아님, **best-effort**).

### 잘못된 삭제(예: foreach)

```java
List<String> list = new ArrayList<>(List.of("a","b","c"));
for (String s : list) {
  if (s.equals("b")) {
    list.remove(s);          // ❌ CME 발생 가능
  }
}
```

### 안전한 삭제 (정석)

```java
List<String> list = new ArrayList<>(List.of("a","b","c"));
for (Iterator<String> it = list.iterator(); it.hasNext();) {
  if (it.next().equals("b")) {
    it.remove();             // ✅ 안전
  }
}
```

### 대안 1 — `removeIf` (Java 8+)

```java
list.removeIf("b"::equals);  // 조건부 일괄 삭제, 내부적으로 안전 처리
```

### 대안 2 — 인덱스 루프 역순 삭제

```java
for (int i = list.size()-1; i >= 0; --i) {
  if (list.get(i).equals("b")) list.remove(i); // 역방향이면 인덱스 안정
}
```

---

## ListIterator — 양방향 이동, 중간 삽입/치환

`ListIterator<E>`는 `List` 전용 확장 반복자.

| 기능 | 메서드 |
|---|---|
| 뒤로 이동 | `hasPrevious()`, `previous()` |
| 현재 위치 인덱스 | `nextIndex()`, `previousIndex()` |
| 삭제/치환/삽입 | `remove()`, `set(E e)`, `add(E e)` |

### 예: 순회하며 조건부 치환·삽입

```java
import java.util.*;

public class ListIteratorDemo {
  public static void main(String[] args) {
    List<String> list = new ArrayList<>(List.of("a","b","c"));
    ListIterator<String> it = list.listIterator();
    while (it.hasNext()) {
      String s = it.next();
      if (s.equals("b")) {
        it.set("B");       // 치환
        it.add("b+");      // 현재 위치 뒤에 삽입
      }
    }
    System.out.println(list); // [a, B, b+, c]
  }
}
```

---

## Map 순회 — entrySet가 정석

| 방식 | 코드 | 특징 |
|---|---|---|
| 키만 순회 | `for (K k : map.keySet())` | 값이 필요하면 두 번 조회 |
| 값만 순회 | `for (V v : map.values())` | 값만 필요할 때 |
| **엔트리 순회** | `for (Map.Entry<K,V> e : map.entrySet())` | **가장 효율적**, 키/값 동시 접근 |

```java
Map<String,Integer> m = Map.of("apple",3,"banana",2);
for (Map.Entry<String,Integer> e : m.entrySet()) {
  System.out.println(e.getKey() + " -> " + e.getValue());
}
```

---

## 배열 vs 컬렉션 — foreach가 만드는 실제 코드

- **배열**: 컴파일러가 단순 인덱스 루프로 바꿈.
- **컬렉션**: `for (T x : coll)` → `for (Iterator<T> it=coll.iterator(); it.hasNext();) { T x = it.next(); ... }`

### 루프 변수 재할당은 원본에 영향 없음

```java
List<String> xs = new ArrayList<>(List.of("a","b"));
for (String s : xs) {
  s = s.toUpperCase(); // 루프 변수만 바뀜, 리스트 요소는 그대로
}
System.out.println(xs); // [a, b]
```

---

## Stream과 forEach/forEachOrdered

| 대상 | 메서드 | 순서 보장 | 용도 |
|---|---|---|---|
| `Iterable` | `default void forEach(Consumer)` | 컬렉션 특성에 따름 | 간단 순회 |
| `Stream` | `void forEach(Consumer)` | **병렬 스트림에서 순서 비보장** | 사이드이펙트 최소화 권장 |
| `Stream` | `void forEachOrdered(Consumer)` | **순서 보장**(비용↑) | 순서가 중요한 경우 |

### Stream은 변환/집계에 강함

```java
List<String> upper =
  list.stream()
      .filter(s -> s.length() >= 3)
      .map(String::toUpperCase)
      .toList();
```
> 단순 출력/조회만이면 **foreach**, 변환·필터링·집계면 **Stream**.

---

## 동시성 컬렉션의 반복자 — 약한 일관성(weakly consistent)

| 컬렉션 | 반복자 특성 |
|---|---|
| `ConcurrentHashMap` / `ConcurrentSkipListSet` 등 | **CME를 던지지 않음**, 순회 중 일부 최신 변경을 반영할 수도 있고 안 할 수도 있음(약한 일관성). |

```java
import java.util.Set;
import java.util.concurrent.*;

public class WeaklyConsistentDemo {
  public static void main(String[] args) {
    Set<Integer> set = ConcurrentHashMap.newKeySet();
    for (int i=0;i<5;i++) set.add(i);

    new Thread(() -> { for (int i=5;i<10;i++) set.add(i); }).start();

    for (Integer v : set) { // 약한 일관성: CME 없음, 보이는 범위는 비결정적
      System.out.print(v+" ");
    }
  }
}
```

> **멀티스레드에서 안전한 순회**가 필요하면 **동시성 컬렉션**을 사용하거나 **스냅샷 복사 후 순회**.

---

## Spliterator — 병렬 분할 순회(Streams의 하부)

`Spliterator`는 **분할 가능한 반복자**로, 스트림의 병렬 처리 기반입니다.

| 주요 메서드 | 설명 |
|---|---|
| `trySplit()` | 일부 구간을 잘라 또 다른 Spliterator 반환(병렬 분할) |
| `tryAdvance(Consumer)` | 다음 요소 처리(있으면 true) |
| `forEachRemaining(Consumer)` | 남은 요소 일괄 처리 |
| `characteristics()` | `SIZED`, `ORDERED`, `SORTED`, `CONCURRENT`, `IMMUTABLE`, `NONNULL`, `SUBSIZED` 등 |

### 커스텀 Spliterator 간단 예

```java
import java.util.Spliterator;
import java.util.function.Consumer;

class RangeSpliterator implements Spliterator<Integer> {
  private int current, endExclusive, step;
  RangeSpliterator(int start, int endExclusive, int step){
    this.current=start; this.endExclusive=endExclusive; this.step=step;
  }
  @Override public boolean tryAdvance(Consumer<? super Integer> action) {
    if (current < endExclusive) { action.accept(current); current += step; return true; }
    return false;
  }
  @Override public Spliterator<Integer> trySplit() {
    int mid = current + ((endExclusive - current) / (2*step))*step;
    if (mid <= current) return null;
    int oldCurrent = current; current = mid;
    return new RangeSpliterator(oldCurrent, mid, step);
  }
  @Override public long estimateSize() { return Math.max(0, (endExclusive - current + step -1)/step); }
  @Override public int characteristics() { return ORDERED | SIZED | SUBSIZED | IMMUTABLE | NONNULL; }
}
```
> 보통 직접 구현할 일은 드물고, **스트림 API**가 자동으로 활용합니다.

---

## Iterable 구현 — 사용자 정의 컨테이너에 foreach 지원

```java
import java.util.*;

class IntBox implements Iterable<Integer> {
  private final int[] data;
  IntBox(int... xs){ this.data=xs; }
  @Override public Iterator<Integer> iterator() {
    return new Iterator<>() {
      int i=0;
      @Override public boolean hasNext() { return i < data.length; }
      @Override public Integer next() {
        if (!hasNext()) throw new NoSuchElementException();
        return data[i++];
      }
      @Override public void remove() { throw new UnsupportedOperationException(); }
    };
  }
}

public class CustomIterableDemo {
  public static void main(String[] args) {
    for (int v : new IntBox(1,2,3)) {
      System.out.print(v+" ");
    }
  }
}
```

---

## 실전 패턴 모음

### 안전한 조건부 삭제 3종 비교

```java
// 1) Iterator.remove
for (Iterator<String> it = list.iterator(); it.hasNext();) if (bad(it.next())) it.remove();

// 2) removeIf
list.removeIf(MyClass::isBad);

// 3) 역방향 인덱스
for (int i=list.size()-1;i>=0;--i) if (bad(list.get(i))) list.remove(i);
```

### 변환(치환)이 목표면: 새 컨테이너로 수집

```java
List<String> out = new ArrayList<>(list.size());
for (String s : list) out.add(transform(s));
list = out;
```
또는
```java
list = list.stream().map(this::transform).toList();
```

### Map 정렬 출력(키 기준)

```java
map.entrySet().stream()
   .sorted(Map.Entry.comparingByKey())
   .forEach(e -> System.out.println(e.getKey()+"="+e.getValue()));
```

---

## 성능·메모리 관점

- **foreach vs 전통 for**: 배열에서는 인덱스 루프와 성능 유사. 컬렉션에서는 Iterator 오버헤드가 미세하게 있을 수 있으나 **대부분 미미**.
- **삭제 동반 작업**: `removeIf`가 가장 간결하고 빠른 편(내부 루프 최적화).
- **`CopyOnWriteArrayList`**: 읽기 많고 쓰기 적을 때 순회는 스냅샷 기반으로 매우 안전(단, 쓰기 비용 큼).

---

## 자주 묻는 함정(FAQ)

1) **`remove()`를 언제 부를 수 있나?**
→ **`next()` 직후의 요소**에만, 한 번만. 그 외엔 `IllegalStateException`.

2) **foreach에서 요소 바꾸려면?**
→ 루프 변수 재할당은 무의미. `ListIterator#set`을 쓰거나 새 컨테이너로 수집.

3) **CME는 항상 발생하나?**
→ 아니요. **best-effort** 감지. 발생하지 않았더라도 동작이 안전하단 뜻은 아닙니다.

4) **동시성 하에서 안전 순회는?**
→ `Concurrent*` 컬렉션 사용(약한 일관성) 또는 스냅샷(`new ArrayList<>(list)`) 후 순회.

---

## 종합 비교표

| 목적 | 가장 단순한 선택 | 수정(삭제/치환/삽입) 필요 | 동시성 |
|---|---|---|---|
| **읽기 전용 순회** | foreach | ListIterator(치환) / 새 리스트로 수집 | 동시성 컬렉션 or 스냅샷 |
| **조건부 삭제** | `removeIf` | `Iterator.remove` | 동시성 컬렉션(약한 일관성) |
| **Map 순회** | `entrySet()` foreach | `entrySet()` + Iterator | `ConcurrentHashMap`의 반복자 |

---

## 전체 데모 — Iterator/foreach/ListIterator/Stream 한 번에 보기

```java
import java.util.*;
import java.util.stream.*;

public class AllInOne {
  public static void main(String[] args) {
    List<String> list = new ArrayList<>(List.of("alpha","beta","pi","gamma","pi"));

    // 1) foreach: 출력만
    for (String s : list) System.out.print(s+" ");
    System.out.println();

    // 2) Iterator.remove: "pi" 제거
    for (Iterator<String> it = list.iterator(); it.hasNext();) {
      if (it.next().equals("pi")) it.remove();
    }
    System.out.println(list); // [alpha, beta, gamma]

    // 3) ListIterator.set/add: beta → BETA, 뒤에 + 붙이기
    ListIterator<String> li = list.listIterator();
    while (li.hasNext()) {
      String s = li.next();
      if (s.equals("beta")) {
        li.set("BETA");
        li.add("BETA+");
      }
    }
    System.out.println(list); // [alpha, BETA, BETA+, gamma]

    // 4) Stream 변환: 길이>=5만 upper로 수집
    List<String> upper = list.stream()
                             .filter(s -> s.length() >= 5)
                             .map(String::toUpperCase)
                             .toList();
    System.out.println(upper); // [ALPHA, GAMMA]
  }
}
```

---

## 결론 — 무엇을 언제 쓰나?

- **foreach**: 가장 읽기 좋은 **읽기 전용 순회**.
- **Iterator.remove / ListIterator**: 순회 중 **안전한 삭제/치환/삽입**.
- **removeIf**: 조건부 삭제의 **현대적 정답**.
- **Stream**: **변환/필터/집계**가 목적일 때.
- **동시성 컬렉션**: **CME 없이** 순회하고 싶을 때(결과는 약한 일관성).
