---
layout: post
title: Java - HashMap, TreeMap, LinkedHashMap
date: 2025-07-24 19:20:23 +0900
category: Java
---
# Java의 Map 구현 클래스 총정리 — HashMap · TreeMap · LinkedHashMap

## 한눈 요약 (Cheat Sheet)

| 선택 기준 | 권장 Map | 이유/메모 |
|---|---|---|
| **가장 빠른 일반 용도**, 순서 불필요 | **HashMap** | 평균 O(1) 접근, 캐시 친화적 |
| **키 정렬(범위/탐색) 필요** | **TreeMap** | Red-Black Tree, O(log n), `subMap/ceiling/floor` 등 풍부 |
| **삽입 순서 유지** | **LinkedHashMap** | 해시 + 이중연결리스트로 순회 순서 보존 |
| **LRU 캐시** | **LinkedHashMap(accessOrder=true)** | `removeEldestEntry` 훅으로 간단 구현 |
| **Enum 키** | **EnumMap**(참고) | 배열 기반이라 매우 빠르고 메모리 효율적 |
| **동시성** | **ConcurrentHashMap**(참고) | 본 문서 3종은 동기화 미제공 |

> **Tip**: 값 기준 정렬이 필요하면 `HashMap` → 스트림 정렬(값 기준) → `LinkedHashMap`로 수집.

---

## Map 인터페이스 공통기초

```java
Map<K,V> m;
m.put(k, v); m.get(k); m.remove(k);
m.containsKey(k); m.containsValue(v);
m.keySet(); m.values(); m.entrySet(); // 반복/뷰
```

- **키 중복 X**, 값 중복 O.
- 반복은 **`entrySet()`**가 가장 효율적(키/값 동시접근).
- **fail-fast** 반복자: 순회 중 구조 변경 시 `ConcurrentModificationException`.
- **스레드 안전 아님**: 병렬 환경은 외부 동기화 또는 동시성 컬렉션.

### 해시·동등성 계약(중요)

- 키 타입은 `equals()`/`hashCode()` 일관 필요.
- **키를 Map에 넣은 뒤 필드를 바꿔 hashCode가 변하면 탐색 불가**(버그 원인 1순위).

```java
final class Point {
  final int x, y; // 불변 키 권장
  Point(int x,int y){this.x=x;this.y=y;}
  @Override public boolean equals(Object o){ /* x,y 비교 */ }
  @Override public int hashCode(){ return 31*x + y; }
}
```

---

## HashMap — 내부 동작과 실전

### 구조 개요

```
table (배열, 길이=2^k)
 ├─ bucket[0]  : Node -> next -> ...  (충돌 체이닝)
 ├─ bucket[1]  : Node (혹은 TreeNode: Red-Black Tree)
 └─ ...
```

- **배열 + 체이닝(LinkedList)**, 충돌 심하면 **트리화(TreeNode, Red-Black Tree)**.
- **Java 8+**: 버킷 길이 ≥ **8**이면 트리화, ≤ **6**이면 리스트로 되돌림.
- `null` 키 **1개 허용**(해시 0 특별취급).

### 핵심 파라미터

- **초기 용량**(power-of-two), **로드 팩터**(기본 0.75)
- 리사이즈 임계값: `threshold = capacity * loadFactor`

수식:
$$
\lambda = \frac{n}{m}
$$
- \(n\): 엔트리 수, \(m\): 버킷 수(=capacity), \(\lambda\): **평균 체인 길이**
- **평균 시간복잡도**: 삽입/탐색/삭제 **O(1)** (충돌 적정), **최악 O(n)**

> **권장 크기 산정**: 예상 엔트리 `N`이면
> `initialCapacity = ceil(N / loadFactor)` 로 설정 → **리사이즈 최소화**.

### 해시 분산(요지)

- 인덱스 = `(table.length - 1) & spread(hash)`
- 상위/하위 비트를 섞어 버킷 균형화(실제 구현은 JDK 버전별 상이할 수 있으나 개념 동일).

### 실전 레시피

#### 빈도수 계산(가독성/성능 우수)

```java
import java.util.*;

public class Freq {
  public static void main(String[] args) {
    String[] words = {"a","b","a","c","b","a"};
    Map<String,Integer> freq = new HashMap<>();
    for (String w: words) freq.merge(w, 1, Integer::sum); // 핵심!
    System.out.println(freq); // {a=3, b=2, c=1}
  }
}
```

#### 다중값 맵(리스트 누적)

```java
Map<String, List<Integer>> idx = new HashMap<>();
idx.computeIfAbsent("k", k -> new ArrayList<>()).add(1);
idx.computeIfAbsent("k", k -> new ArrayList<>()).add(2);
System.out.println(idx.get("k")); // [1,2]
```

#### 초기 용량 최적화(대량 put)

```java
int expected = 1_000_000;
int initialCapacity = (int)Math.ceil(expected / 0.75);
Map<Integer,Integer> m = new HashMap<>(initialCapacity, 0.75f);
```

### 흔한 함정

- **키 가변성**(필드 변경) ⇒ `equals/hashCode` 변동 ⇒ 탐색 실패.
- **값 기준 정렬 필요**: `HashMap` 자체로 불가 → 스트림 정렬 후 `LinkedHashMap`으로 수집.
- `containsValue`는 O(n) (피해야 함). 인덱스용 보조 구조를 고려.

---

## LinkedHashMap — 순서 유지 & LRU

### 구조

- **HashMap + 이중연결리스트**(head ↔ … ↔ tail)
- **삽입 순서** 또는 **접근 순서(accessOrder=true)** 유지
- **null 키 1개 허용**

### LRU 캐시(정석)

```java
import java.util.*;

class LRUCache<K,V> extends LinkedHashMap<K,V> {
  private final int cap;
  LRUCache(int cap){ super(cap, 0.75f, true); this.cap = cap; }
  @Override protected boolean removeEldestEntry(Map.Entry<K,V> e){ return size()>cap; }
}

public class LruDemo {
  public static void main(String[] args) {
    LRUCache<Integer,String> cache = new LRUCache<>(2);
    cache.put(1,"A"); cache.put(2,"B"); cache.get(1); // 1이 최신
    cache.put(3,"C"); // 2가 제거
    System.out.println(cache.keySet()); // [1, 3]
  }
}
```

> `removeEldestEntry`는 **put 후 호출**. `get()`이 **접근 순서**를 갱신.

---

## TreeMap — 정렬 & 범위 질의

### 개요

- **Red-Black Tree**(균형 이진탐색트리) 기반, **키 정렬 유지**.
- **null 키 불가**, 값 `null`은 가능.
- 시간복잡도: 삽입/탐색/삭제 **O(log n)**.

### 정렬 기준

- 키의 **자연 순서**(`Comparable`) 또는 **생성자에 `Comparator` 전달**.

```java
import java.util.*;

public class TreeMapDemo {
  public static void main(String[] args) {
    TreeMap<String,Integer> m = new TreeMap<>(Comparator.reverseOrder());
    m.put("apple",3); m.put("banana",2); m.put("cherry",5);
    System.out.println(m);             // {cherry=5, banana=2, apple=3}
    System.out.println(m.ceilingKey("b")); // banana
    System.out.println(m.subMap("banana", true, "cherry", false)); // {banana=2}
  }
}
```

### Navigable API(강력)

- `firstKey/lastKey`, `higher/lower`, `ceiling/floor`,
  `subMap/headMap/tailMap`(경계 포함/제외 선택).

### 주의

- `Comparator`가 `equals`와 **일관되지 않으면** `Map` 규약(키 유일성) 혼란.
  (예: 대소문자 무시 비교기는 서로 다른 “a”/“A”를 동일 키로 간주)

---

## 성능·메모리·복잡도 비교 (정밀)

| 항목 | HashMap | LinkedHashMap | TreeMap |
|---|---|---|---|
| 내부 | 해시테이블(체이닝/트리화) | 해시 + 이중연결리스트 | Red-Black Tree |
| 정렬/순서 | 없음 | **삽입/접근 순서 유지** | **키 정렬** |
| 키 `null` | **허용(1개)** | **허용(1개)** | **불가** |
| 평균 접근 | **O(1)** | **O(1)** | **O(log n)** |
| 최악 | O(n) | O(n) | O(log n) |
| 메모리 | △ | ▲(링크 오버헤드) | △(노드/색상/링크) |
| 범위 질의 | ✗ | ✗ | **✓ (subMap 등)** |

---

## 값 기준 정렬, 상위 K, 그룹핑 등 실전 레시피

### 값으로 정렬해 순서 유지

```java
import java.util.*;
import java.util.stream.*;

public class SortByValue {
  public static void main(String[] args) {
    Map<String,Integer> m = Map.of("a",3,"b",1,"c",2);
    LinkedHashMap<String,Integer> sorted = m.entrySet().stream()
      .sorted(Map.Entry.<String,Integer>comparingByValue().reversed())
      .collect(Collectors.toMap(
        Map.Entry::getKey, Map.Entry::getValue,
        (x,y)->x, LinkedHashMap::new));
    System.out.println(sorted); // {a=3, c=2, b=1}
  }
}
```

### 상위 K 빈도 (HashMap + 힙)

```java
import java.util.*;

public class TopK {
  public static void main(String[] args) {
    String[] arr = {"a","a","b","c","b","a","d","c","c"};
    Map<String,Integer> freq = new HashMap<>();
    for (String s: arr) freq.merge(s,1,Integer::sum);

    int K = 2;
    PriorityQueue<Map.Entry<String,Integer>> pq =
      new PriorityQueue<>(Map.Entry.comparingByValue()); // min-heap
    for (var e: freq.entrySet()) {
      pq.offer(e);
      if (pq.size() > K) pq.poll();
    }
    List<String> top = new ArrayList<>();
    while(!pq.isEmpty()) top.add(pq.poll().getKey());
    Collections.reverse(top);
    System.out.println(top); // [a, c]
  }
}
```

### 접두사/범위 검색(사전식)

```java
TreeMap<String,Integer> dict = new TreeMap<>();
dict.put("app",1); dict.put("apple",2); dict.put("apricot",3); dict.put("banana",4);

// "ap"로 시작하는 범위: [ "ap", "aq" )
SortedMap<String,Integer> apOnly =
  dict.subMap("ap", "aq");
System.out.println(apOnly); // {app=1, apple=2, apricot=3}
```

### 삽입 순서 보존 출력(로그/리플레이)

```java
Map<Long, String> log = new LinkedHashMap<>();
log.put(1001L,"login");
log.put(1002L,"view");
log.put(1003L,"logout");
log.forEach((t,ev)->System.out.println(t+" : "+ev));
```

---

## 현대 자바 Map API 꿀팁

- **안전 삽입**: `putIfAbsent(k, v)`
- **조건부 제거/치환**: `remove(k, v)`, `replace(k, oldV, newV)`
- **계산형 갱신**: `compute`, `computeIfAbsent`, `computeIfPresent`
- **누적**: `merge(k, x, (oldV, x) -> ...)`

```java
Map<String,Integer> m = new HashMap<>();
m.putIfAbsent("a", 0);
m.computeIfPresent("a", (k,v)->v+1); // 1
m.merge("a", 1, Integer::sum);       // 2
```

- **불변 Map**: `Map.of(...)`, `Map.copyOf(...)`(수정 불가 뷰/복사)

---

## 동시성/반복/불변성 주의

- 세 구현체 모두 **동기화 없음** → 멀티스레드 쓰기에는 **ConcurrentHashMap**(분절/노드 레벨 동시성) 사용.
- 반복 중 구조 변경은 fail-fast. 필요한 경우:
  - `Iterator.remove()` 사용
  - 전체를 복사한 뒤 수정
  - 동시성 컬렉션 사용

---

## 테스트·벤치마크 메모

- 마이크로 벤치마크는 **JMH** 권장(워밍업/포크/GC 제어).
- `System.nanoTime()` 단발 측정은 왜곡이 큼(옵티마이저·캐시·분기 예측 등).

---

## 확장 토픽(짚고 넘어가기)

- **WeakHashMap**: 키가 약참조(캐시·메모리 릭 방지 용)
- **IdentityHashMap**: `==` 으로 키 동일성 판정(실험/특수 목적)
- **EnumMap**: 열거형 키에 최적(배열 인덱싱, 매우 빠름)
- **TreeMap null 키**: 자연 순서 사용 시 **NPE/불가**(Comparator가 명시적으로 처리하지 않으면)

---

## 종합 선택 가이드 (표)

| 요구사항 | 추천 | 근거 |
|---|---|---|
| 최대 성능, 순서 무관 | **HashMap** | O(1) 평균, 캐시 친화 |
| 키 범위 탐색/구간 쿼리 | **TreeMap** | 정렬/Navigable API |
| 삽입/접근 순서 유지 | **LinkedHashMap** | 이중연결리스트 |
| LRU | **LinkedHashMap(accessOrder=true)** | `removeEldestEntry` |
| Enum 키 | **EnumMap** | 배열 기반 |
| 병렬 업데이트 다수 | **ConcurrentHashMap** | 동시성 보장 |

---

## 전체 데모: 세 Map의 차이 한 번에 보기

```java
import java.util.*;

public class MapLandscape {
  public static void main(String[] args) {
    // 1) HashMap: 순서 X, 빠름
    Map<String,Integer> h = new HashMap<>();
    h.put("banana",2); h.put("apple",3); h.put("cherry",5);
    System.out.println("HashMap     : " + h);

    // 2) LinkedHashMap: 삽입 순서
    Map<String,Integer> lh = new LinkedHashMap<>();
    lh.put("banana",2); lh.put("apple",3); lh.put("cherry",5);
    System.out.println("LinkedHashMap: " + lh);

    // 3) TreeMap: 정렬 순서
    Map<String,Integer> t = new TreeMap<>();
    t.put("banana",2); t.put("apple",3); t.put("cherry",5);
    System.out.println("TreeMap     : " + t);

    // 4) 값 기준 정렬 -> LinkedHashMap
    LinkedHashMap<String,Integer> byValDesc =
      h.entrySet().stream()
        .sorted(Map.Entry.<String,Integer>comparingByValue().reversed())
        .collect(LinkedHashMap::new,
                 (m,e)->m.put(e.getKey(), e.getValue()),
                 LinkedHashMap::putAll);
    System.out.println("Value-Desc  : " + byValDesc);
  }
}
```

---

## ASCII 도식

### HashMap (버킷/체이닝/트리화)

```
index: 0    1        2         3 ...
       |    |        |         |
       v    v        v         v
      [*]  [ ]  ->  [*]  ->   [T]
       |           (Node)     (TreeBin)
      next
```

### LinkedHashMap (순회 순서)

```
head <-> e1 <-> e2 <-> e3 <-> tail
   ^             ^
  put 순서, get(accessOrder=true)이면 최근 사용이 뒤로 이동
```

### TreeMap (Red-Black Tree 개념)

```
        (root,B)
        /     \
     (R)       (R)
     / \       / \
   ... ...   ... ...
균형 유지로 O(log n)
```

---

## 마무리

- **HashMap**: “기본값” — 대부분의 요구에 충분히 빠르고 간단
- **LinkedHashMap**: “순서를 보장해야 할 때”, 특히 **LRU**
- **TreeMap**: “정렬·범위 탐색이 핵심일 때”
- 키의 **불변성**, `equals/hashCode`/`Comparator` **일관성**을 지키면 **성능과 정확성**을 동시에 확보할 수 있습니다.
