---
layout: post
title: Java - List, Set, Map
date: 2025-07-24 17:20:23 +0900
category: Java
---
# Java 컬렉션 프레임워크 개요: List, Set, Map

## 0. 컬렉션 프레임워크 한눈에 보기

- **List**: 순서 유지, **중복 허용**, 인덱스 접근
  구현체: `ArrayList`, `LinkedList`, (`Vector`, `Stack` 레거시)
- **Set**: **중복 불가**, 수학적 집합
  구현체: `HashSet`, `LinkedHashSet`(입력 순서), `TreeSet`(정렬)
- **Map**: 키-값 쌍, **키 중복 불가**
  구현체: `HashMap`, `LinkedHashMap`(입력·접근 순서), `TreeMap`(키 정렬), (`Hashtable` 레거시)

> 공통: **제네릭(Generics)**, **반복자(Iterator)**, **fail-fast** 탐지, **알고리즘/메모리 튜닝 파라미터**(초기 용량, 로드 팩터 등)를 이해하면 품질과 성능이 눈에 띄게 좋아집니다.

---

## 1. List — 순서·중복·인덱스

### 1.1 주요 구현체와 특성

| 구현체 | 내부 구조 | 인덱스 접근 | 중간 삽입/삭제 | 메모리 | 비고 |
|---|---|---:|---:|---:|---|
| `ArrayList` | 동적 배열 | **O(1)** | O(n) | 낮음 | `ensureCapacity`, `trimToSize` 지원 |
| `LinkedList` | 이중 연결 리스트 | O(n) | **O(1)**(노드 참조 시) | 높음 | `Deque`/큐 메서드 풍부 |
| `Vector` | 동기화 배열 | O(1) | O(n) | 낮음 | 레거시(불필요한 동기화) |
| `Stack` | `Vector` 상속 | O(1) | O(1)/O(n) | 낮음 | 레거시, 대신 `ArrayDeque` 권장 |

### 1.2 예제 — 기본 조작
```java
import java.util.*;

public class ListExample {
  public static void main(String[] args) {
    List<String> list = new ArrayList<>();

    list.add("apple");
    list.add("banana");
    list.add("apple");       // 중복 허용

    System.out.println(list.get(0)); // "apple"
    list.remove("banana");           // 값 삭제
    list.add(1, "cherry");           // 중간 삽입

    System.out.println(list);        // [apple, cherry, apple]
  }
}
```

### 1.3 시간복잡도 요약(List)

| 연산 | ArrayList | LinkedList |
|---|---:|---:|
| `get(i)` | **O(1)** | O(n) |
| `add(e)`(끝) | 평균 **O(1)** | O(1) |
| `add(i,e)` | O(n) | O(1) (해당 노드 참조를 이미 가진 경우) |
| `remove(i/e)` | O(n) | O(1) (노드 참조 시) |

> **대부분의 일반 케이스**에서 `ArrayList`가 빠르고 메모리 효율이 좋습니다. 정말로 **중간 삽입/삭제가 지배적**이고 **반드시 리스트 구조**가 필요할 때만 `LinkedList`를 선택하세요(대안: `ArrayDeque`).

---

## 2. Set — 중복 금지·동치 규약

### 2.1 주요 구현체와 특성

| 구현체 | 내부 구조 | 순서 | null | 요구 조건 |
|---|---|---|---|---|
| `HashSet` | 해시테이블 | 순서 없음 | 1개 허용 | `hashCode`/`equals` 일관 |
| `LinkedHashSet` | 해시 + 이중 연결리스트 | **입력 순서 유지** | 1개 허용 | 동치 동일 |
| `TreeSet` | Red-Black Tree | **정렬**(자연 or 비교자) | 불가 | `compareTo/Comparator` 일관 |

### 2.2 해시 기반 Set의 핵심 — 동치 규약
사용자 타입을 Set에 넣을 때는 반드시 **동치 규약**을 지켜야 합니다.

- **규약**: `a.equals(b) == true` → 반드시 `a.hashCode() == b.hashCode()`
- `equals`/`hashCode` 둘 다 재정의해야 충돌/누락 없고 성능 유지

```java
import java.util.Objects;

final class User {
  final String id; final String name;
  User(String id, String name){ this.id=id; this.name=name; }
  @Override public boolean equals(Object o){
    if (this==o) return true; if(!(o instanceof User u)) return false;
    return Objects.equals(id, u.id);
  }
  @Override public int hashCode(){ return Objects.hash(id); }
}
```

### 2.3 정렬 기반 Set의 핵심 — 비교 일관성
- `TreeSet`은 `compareTo`/`Comparator`로 **정렬 기준**을 잡습니다.
- **주의**: `compareTo(a,b)==0`인 두 객체는 **같은 원소로 간주**되어 **하나만** 저장됩니다.
  `equals`와의 **일관성**이 깨지지 않도록 설계하세요.

```java
import java.util.*;

record Product(String sku, String name) {}
// 이름 기준으로만 정렬하면, 이름이 같은 서로 다른 sku가 중복 제거될 수 있음
Set<Product> set = new TreeSet<>(Comparator.comparing(Product::name));
```

---

## 3. Map — 키/값·탐색·정렬

### 3.1 주요 구현체와 특성

| 구현체 | 내부 구조 | 순서 | 키 null | 값 null | 평균 복잡도 |
|---|---|---|---|---|---|
| `HashMap` | 해시테이블(+트리화) | 없음 | **1개 허용** | 허용 | put/get/remove: **O(1)** |
| `LinkedHashMap` | 해시 + 이중 연결리스트 | **입력 순서** or **접근 순서** | 1개 허용 | 허용 | O(1) |
| `TreeMap` | Red-Black Tree | **키 정렬** | 불가 | 허용 | O(log n) |
| `Hashtable` | 동기화 해시 | 버킷 순 | 불가 | 불가 | 레거시(불필요한 동기화) |

### 3.2 기본 조작과 엔트리 순회
```java
import java.util.*;

public class MapExample {
  public static void main(String[] args) {
    Map<String,Integer> map = new HashMap<>();
    map.put("apple", 1);
    map.put("banana", 2);
    map.put("apple", 3); // 키 중복 → 값 덮어쓰기

    System.out.println(map.get("apple")); // 3

    for (Map.Entry<String,Integer> e : map.entrySet()) {
      System.out.println(e.getKey() + " -> " + e.getValue());
    }
  }
}
```

### 3.3 LRU 캐시 — `LinkedHashMap(accessOrder=true)`
```java
import java.util.*;

class LRUCache<K,V> extends LinkedHashMap<K,V> {
  private final int capacity;
  LRUCache(int capacity){ super(capacity, 0.75f, true); this.capacity=capacity; }
  @Override protected boolean removeEldestEntry(Map.Entry<K,V> eldest){
    return size() > capacity;
  }
}
```

---

## 4. 반복·삭제·정렬 — 실무 핵심 패턴

### 4.1 안전한 삭제 (CME 방지)
```java
// 1) Iterator.remove
for (var it = list.iterator(); it.hasNext();) {
  if (shouldDelete(it.next())) it.remove();
}

// 2) removeIf (Java 8+)
list.removeIf(MyClass::shouldDelete);

// 3) 역방향 인덱스 삭제(ArrayList)
for (int i = list.size()-1; i >= 0; --i) if (bad(list.get(i))) list.remove(i);
```

### 4.2 정렬 출력 (Map → 키 기준)
```java
map.entrySet().stream()
   .sorted(Map.Entry.comparingByKey())
   .forEach(e -> System.out.println(e.getKey() + "=" + e.getValue()));
```

### 4.3 foreach vs Iterator 요약
- **읽기 전용**: foreach 간결
- **순회하며 삭제/치환/삽입**: `Iterator.remove` / `ListIterator` 사용

---

## 5. 시간복잡도 총정리

| 분류 | 연산 | `ArrayList` | `LinkedList` | `HashSet`/`HashMap` | `LinkedHash*` | `TreeSet`/`TreeMap` |
|---|---|---:|---:|---:|---:|---:|
| 조회 | `get(i)` / `get(key)` | **O(1)** | O(n) | **O(1)** | **O(1)** | O(log n) |
| 삽입 | 끝/키없음 | **O(1)** amort. | O(1) | **O(1)** | **O(1)** | O(log n) |
| 삽입 | 중간/충돌심함 | O(n) | **O(1)**(노드) | 최악 O(n) | 최악 O(n) | O(log n) |
| 삭제 | 값/키 | O(n) | **O(1)**(노드) | **O(1)** | **O(1)** | O(log n) |
| 순회 | iterator | O(n) | O(n) | O(n) | O(n) | O(n) |

> 해시 구조는 **좋은 해시 함수**와 적절한 **초기 용량/로드 팩터(기본 0.75)**가 성능을 좌우합니다.

---

## 6. 불변/수정불가 컬렉션 — `List.of`, `Set.of`, `Map.of`

### 6.1 불변 팩토리(Java 9+)
```java
var ilist = List.of("a","b","c");          // 불변
var iset  = Set.of("a","b","c");           // 중복 전달 시 예외
var imap  = Map.of("a",1,"b",2,"c",3);     // 불변
```
- 크기/내용 변경 메서드 호출 시 `UnsupportedOperationException`.
- **스레드 안전성** 측면에서 유리, 공유에 적합.

### 6.2 `Collections.unmodifiable*` vs 진짜 불변
```java
var base = new ArrayList<>(List.of(1,2,3));
var view = Collections.unmodifiableList(base); // 수정불가 "뷰"
base.add(4); // view에도 4가 보임(기저 컬렉션 변경 반영)
```
> **진짜 불변**(방어적 복사)과 **수정불가 뷰**를 구분하세요.

---

## 7. 동시성 컬렉션 — CME 없는 약한 일관성

| 컬렉션 | 특징 |
|---|---|
| `ConcurrentHashMap` | 세그먼트/버킷 레벨 동시성, 반복자는 **CME 없이** 약한 일관성 |
| `CopyOnWriteArrayList` | 쓰기 시 전체 복사, 읽기 많은 워크로드에 유리 |
| `ConcurrentSkipListMap/Set` | 정렬 + 동시성(스킵 리스트), O(log n) |
| `BlockingQueue` 계열 | 생산자/소비자 패턴 |

```java
import java.util.Set;
import java.util.concurrent.ConcurrentHashMap;

Set<Integer> set = ConcurrentHashMap.newKeySet();
set.add(1); set.add(2);
for (int v : set) { /* CME 없음, 약한 일관성 */ }
```

---

## 8. 정렬·비교 — `Comparator` 활용

```java
record Person(String name, int age) {}

List<Person> people = new ArrayList<>(List.of(
  new Person("Ann", 30),
  new Person("Bob", 25),
  new Person("Ann", 20)
));

people.sort(Comparator
  .comparing(Person::name)
  .thenComparingInt(Person::age));

System.out.println(people);
```

- `TreeSet`/`TreeMap`에 동일 기준을 전달하면 **정렬 일관성**을 유지할 수 있습니다.

---

## 9. 튜닝 포인트 — 초기 용량·로드 팩터·배열 크기

```java
var list = new ArrayList<String>(10_000); // 재할당 최소화
var map  = new HashMap<String,Integer>(16, 0.75f);
list.ensureCapacity(20_000);
```
- **ArrayList**: 예상 크기를 알면 `initialCapacity`로 **재복사 비용** 절감.
- **HashMap**: 초기 용량과 로드 팩터로 **리해시 비용** 조정.

---

## 10. 실전 레시피

### 10.1 빈도수 세기(토큰 카운팅)
```java
import java.util.*;

Map<String,Integer> freq = new HashMap<>();
for (String t : List.of("a","b","a")) {
  freq.merge(t, 1, Integer::sum); // 없으면 1, 있으면 누적
}
System.out.println(freq); // {a=2, b=1}
```

### 10.2 중복 제거 + 입력 순서 보존
```java
var input = List.of("b","a","b","c","a");
var dedup = new LinkedHashSet<>(input); // [b, a, c]
```

### 10.3 맵 정렬된 뷰
```java
var sorted = new TreeMap<>(Map.of("b",2,"a",1,"c",3));
System.out.println(sorted); // {a=1, b=2, c=3}
```

---

## 11. 선택 가이드 — 무엇을 언제 쓰나?

| 요구 | 추천 |
|---|---|
| **순서 + 중복 + 빠른 조회** | `ArrayList` |
| **중간 삽입/삭제 빈번** | `LinkedList` (또는 구조 재검토/`ArrayDeque`) |
| **중복 제거 + 빠른 포함 검사** | `HashSet` |
| **입력/접근 순서 유지** | `LinkedHashSet` / `LinkedHashMap` |
| **정렬된 집합/맵 필요** | `TreeSet` / `TreeMap` |
| **키-값 캐시 + LRU** | `LinkedHashMap(accessOrder=true)` |
| **멀티스레드 읽기 많음** | `CopyOnWriteArrayList` |
| **동시성 + 키-값** | `ConcurrentHashMap` |
| **변경 불필요/공유** | `List.of`/`Set.of`/`Map.of`(불변) |

---

## 12. 흔한 함정과 베스트 프랙티스

1) **사용자 타입을 Set/Map 키로 사용** → `equals`/`hashCode`(해시) 또는 `compareTo`/`Comparator`(정렬) **반드시 재정의/제공**.
2) **foreach 중 삭제** → `Iterator.remove`/`removeIf` 사용(§4).
3) **정렬 기준과 equals 불일치** → `TreeSet/TreeMap`에서 **중복 소거** 발생.
4) **`Hashtable`/`Vector` 사용** → 레거시. `ConcurrentHashMap`/`ArrayList` 사용.
5) **불변과 수정불가 뷰 혼동** → 기저 변경이 보일 수 있음(§6.2).

---

## 13. 종합 예제 — List/Set/Map 한 번에

```java
import java.util.*;

public class CollectionsAllInOne {
  public static void main(String[] args) {
    // 1) List
    List<String> list = new ArrayList<>(List.of("apple","banana","apple"));
    list.removeIf("banana"::equals);
    System.out.println(list); // [apple, apple]

    // 2) Set
    Set<String> set = new LinkedHashSet<>(list); // 입력 순서 유지 중복 제거
    set.add("cherry");
    System.out.println(set); // [apple, cherry]

    // 3) Map
    Map<String,Integer> map = new LinkedHashMap<>();
    for (String s : set) map.put(s, s.length());
    System.out.println(map); // {apple=5, cherry=6}

    // 4) 정렬된 뷰
    Map<String,Integer> sorted = new TreeMap<>(map);
    System.out.println(sorted); // {apple=5, cherry=6}
  }
}
```

---

## 부록 A. 인터페이스/구현체 정리표

| 인터페이스 | 대표 구현체 | 특기 |
|---|---|---|
| `List` | `ArrayList`, `LinkedList` | 순서, 중복 |
| `Set` | `HashSet`, `LinkedHashSet`, `TreeSet` | 중복 불가 |
| `Map` | `HashMap`, `LinkedHashMap`, `TreeMap` | 키-값 |
| `Queue`/`Deque` | `ArrayDeque`, `LinkedList` | 큐/덱 |
| 동시성 | `ConcurrentHashMap`, `CopyOnWriteArrayList`, `ConcurrentSkipListMap/Set` | 스레드 안전 |

---

## 결론

- **List/Set/Map**은 서로 다른 **목적/불변성/순서/성능** 트레이드오프를 가집니다.
- 실무에서는 **ArrayList + HashSet/HashMap**이 기본 축, **LinkedHash***로 순서 제어, **Tree***로 정렬, **Concurrent***로 동시성을 보완합니다.
- **동치/정렬 규약**, **fail-fast 규칙**, **불변 컬렉션**을 일관되게 적용하면 **안전·가독성·성능**이 크게 개선됩니다.
