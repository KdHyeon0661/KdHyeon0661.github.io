---
layout: post
title: Java - HashSet vs TreeSet
date: 2025-07-24 20:20:23 +0900
category: Java
---
# Java Set 구현 클래스 총정리 — HashSet · TreeSet

## 0. 한눈 요약 (Cheat Sheet)

| 선택 기준 | 권장 Set | 이유/메모 |
|---|---|---|
| **가장 빠른 일반 용도**, 순서 불필요 | **HashSet** | 평균 O(1) 삽입/탐색/삭제 |
| **정렬·범위 질의**(사전식, 구간 쿼리) | **TreeSet** | Red-Black Tree, O(log n), `subSet/ceiling/floor` |
| **삽입 순서 보존하며 중복 제거** | **LinkedHashSet** | 해시 + 이중연결리스트 |
| **정렬 + 동시성** | **ConcurrentSkipListSet** | lock-free-ish 스킵리스트, O(log n) |
| **Enum 전용, 초고속/저메모리** | **EnumSet** | 비트셋 기반 |
| **동시성(정렬 불필요)** | `ConcurrentHashMap.newKeySet()` | 해시 기반 병렬성 |

---

## 1. Set 인터페이스 핵심

- **중복 불가**, `null` 허용 여부는 구현체마다 다름.
- 대표 메서드: `add`, `remove`, `contains`, `size`, `isEmpty`, `iterator`, `forEach`.
- **fail-fast** 반복자: 순회 중 구조 변경 시 `ConcurrentModificationException` 발생(동시성 컬렉션 제외).

---

## 2. HashSet — 해시 기반 집합

### 2.1 내부 구조 & 동작 원리
- 실질적으로 **`HashMap<E, Object>`** 를 래핑: 요소는 **키**에 저장, 값은 고정 더미(`PRESENT`) 객체.
- **배열 + 체이닝(LinkedList/TreeNode)** 구조(충돌 심하면 **트리화**, JDK 8+).
- `null` **요소 1개 허용**.

```
table[0] -> Node(e1) -> Node(e2) -> ...
table[1] -> TreeBin(...)
...
```

### 2.2 시간복잡도 & 로드 팩터
- 평균 **O(1)** 삽입/탐색/삭제, 최악 **O(n)** (해시 품질/충돌에 좌우).
- 임계값(리사이즈 트리거):  
  $$ \text{threshold} = \text{capacity} \times \text{loadFactor} $$
  기본 용량 16, 로드 팩터 0.75.

> **초기 용량 산정**: 예상 엔트리 수 \(N\)라면  
> `initialCapacity = ceil(N / loadFactor)` 로 생성 → 리사이즈 최소화.

### 2.3 `equals`/`hashCode` 계약 (중요)
- **키(요소)의 동등성은 `equals`로, 버킷 선택은 `hashCode`로**.  
- **집합에 넣은 뒤 요소 상태가 바뀌어 hashCode/equal 결과가 변하면 탐색 실패** → **불변(immutable) 요소** 권장.

```java
final class Point {
  final int x, y;
  Point(int x, int y){ this.x=x; this.y=y; }
  @Override public boolean equals(Object o){
    if(this==o) return true; if(!(o instanceof Point p)) return false;
    return x==p.x && y==p.y;
  }
  @Override public int hashCode(){ return 31*x + y; }
}
```

### 2.4 실전 예제
```java
import java.util.*;

public class HashSetExample {
  public static void main(String[] args) {
    Set<String> s = new HashSet<>();
    s.add("apple"); s.add("banana"); s.add("cherry"); s.add("apple"); // 중복 무시
    System.out.println(s);                  // 순서 비보장
    System.out.println(s.contains("banana"));// true
  }
}
```

### 2.5 흔한 함정
- **가변 객체를 요소로 사용** 후 필드 변경 → `contains`/`remove` 실패.
- `contains`는 `equals` 기준, **값의 대소 관계는 무의미**.
- **값 기준 정렬 출력**은 `HashSet` 자체로 불가 → 스트림 정렬 후 목록화.

---

## 3. LinkedHashSet — 순서가 있는 HashSet (보너스)

- 내부적으로 **HashMap + 이중연결리스트**, **삽입 순서**(또는 접근 순서) 유지.
- 빠른 중복 제거 + **원본 순서 보전**에 최적.

```java
import java.util.*;

public class StableDedup {
  public static void main(String[] args) {
    List<String> raw = List.of("b","a","b","c","a");
    Set<String> unique = new LinkedHashSet<>(raw);
    System.out.println(unique); // [b, a, c] ← 최초 등장 순서 유지
  }
}
```

---

## 4. TreeSet — 정렬·범위 질의가 강점

### 4.1 내부 구조 & 정렬
- **Red-Black Tree(균형 BST)** 기반, 요소는 **항상 정렬 상태**.
- 기본은 **자연 순서(Comparable)**, 또는 **생성자에 Comparator** 전달.
- 시간복잡도: 삽입/탐색/삭제 **O(log n)**.

> **null 요소**: 자연 순서 사용 시 **불가**.  
> 다만 **명시적 Comparator가 `null`을 처리하면 허용 가능**(예: `Comparator.nullsFirst(...)`).

### 4.2 NavigableSet API (핵심)
- **경계/이웃 탐색**: `lower`, `floor`, `ceiling`, `higher`
- **범위 뷰**: `subSet(from, to)`, `subSet(fromIncl, from, to, toIncl)`, `headSet`, `tailSet`
- **추출/역순**: `pollFirst/Last`, `descendingSet`

```java
import java.util.*;

public class TreeSetExample {
  public static void main(String[] args) {
    NavigableSet<String> set = new TreeSet<>();
    set.add("banana"); set.add("apple"); set.add("cherry");
    System.out.println(set);           // [apple, banana, cherry]
    System.out.println(set.first());   // apple
    System.out.println(set.ceiling("b"));// banana
    System.out.println(set.subSet("banana", true, "cherry", false)); // [banana]
  }
}
```

### 4.3 접두사 범위 질의(사전식)
```java
TreeSet<String> dict = new TreeSet<>(List.of("app","apple","apricot","banana"));
SortedSet<String> ap = dict.subSet("ap", "aq"); // "ap"로 시작하는 범위
System.out.println(ap); // [app, apple, apricot]
```

### 4.4 Comparator 일관성 주의
- `Comparator`가 **전이적/반사적/대칭적** 토털 오더여야 함.
- **`equals`와의 일관성**: TreeSet은 `compare(a,b)==0`이면 같은 원소로 취급(하나만 저장).  
  `equals`와 다르면 **예상치 못한 중복 제거**가 발생할 수 있음.

---

## 5. HashSet vs TreeSet — 종합 비교

| 항목 | HashSet | TreeSet |
|---|---|---|
| 내부 구조 | HashMap 기반(해시) | Red-Black Tree |
| 순서/정렬 | 없음 | **자동 정렬** |
| 평균 시간 | **O(1)** | **O(log n)** |
| null 허용 | **허용(1개)** | 자연순서 **불가**, Comparator가 허용시만 가능 |
| 요구사항 | `hashCode`/`equals` | `Comparable` 또는 `Comparator` |
| 강점 | 최대 성능, 단순 멤버십 테스트 | 범위 질의/순차 탐색/이웃 검색 |

---

## 6. 값 동등성 vs 정렬 동치성 — BigDecimal 사례

- `HashSet`은 **`equals`**로 동등성 결정.
- `TreeSet`은 **`compareTo/Comparator`** 결과 0을 **동일 원소**로 간주.

```java
import java.math.BigDecimal;
import java.util.*;

public class BigDecimalTrap {
  public static void main(String[] args) {
    Set<BigDecimal> hs = new HashSet<>();
    hs.add(new BigDecimal("2.0"));
    hs.add(new BigDecimal("2"));
    System.out.println("HashSet size = " + hs.size()); // 2 (equals가 스케일까지 비교)

    Set<BigDecimal> ts = new TreeSet<>();
    ts.add(new BigDecimal("2.0"));
    ts.add(new BigDecimal("2"));
    System.out.println("TreeSet size = " + ts.size()); // 1 (compareTo는 수치 동등시 0)
  }
}
```

> 설계 시, **어떤 “동등”을 원하나?** 를 먼저 결정하고 Set을 택하세요.

---

## 7. 실전 레시피 모음

### 7.1 문자열 중복 제거 + 원본 순서 유지
```java
List<String> in = List.of("a","b","a","c","b");
List<String> out = new ArrayList<>(new LinkedHashSet<>(in));
System.out.println(out); // [a, b, c]
```

### 7.2 대소문자 무시 중복 제거(주의: 자연 `equals`와 불일치)
```java
Set<String> caseInsensitive = new TreeSet<>(String.CASE_INSENSITIVE_ORDER);
caseInsensitive.addAll(List.of("a","A","b"));
System.out.println(caseInsensitive); // [a, b] ← "a"와 "A"를 동일시
```

### 7.3 길이 기준 상위 K개(동률은 사전식 보조정렬, 일관성 유지)
```java
import java.util.*;

public class TopKByLength {
  public static void main(String[] args) {
    Comparator<String> cmp = Comparator
        .comparingInt(String::length)
        .thenComparing(Comparator.naturalOrder());
    NavigableSet<String> top = new TreeSet<>(cmp);
    List<String> data = List.of("pear","apple","banana","fig","date","avocado");
    int K = 3;
    for (String s: data) {
      top.add(s);
      if (top.size() > K) top.pollFirst(); // 가장 짧은 것 제거
    }
    System.out.println(top); // 길이 상위 3개 정렬셋
  }
}
```

### 7.4 집합 연산(합집합/교집합/차집합)
```java
import java.util.*;

public class SetOps {
  public static void main(String[] args) {
    Set<Integer> a = new HashSet<>(Set.of(1,2,3,4));
    Set<Integer> b = new HashSet<>(Set.of(3,4,5));

    Set<Integer> union = new HashSet<>(a); union.addAll(b);
    Set<Integer> inter = new HashSet<>(a); inter.retainAll(b);
    Set<Integer> diff  = new HashSet<>(a); diff.removeAll(b);

    System.out.println(union); // [1,2,3,4,5]
    System.out.println(inter); // [3,4]
    System.out.println(diff);  // [1,2]
  }
}
```

### 7.5 TreeSet에서 `null` 허용 (명시 Comparator)
```java
import java.util.*;

public class NullFriendlyTreeSet {
  public static void main(String[] args) {
    Comparator<Integer> cmp = Comparator.nullsFirst(Comparator.naturalOrder());
    Set<Integer> set = new TreeSet<>(cmp);
    set.add(null); set.add(1); set.add(0);
    System.out.println(set); // [null, 0, 1]
  }
}
```

---

## 8. 성능·메모리 관점

| 항목 | HashSet | TreeSet |
|---|---|---|
| 캐시 지역성 | **좋음**(배열 중심) | 중간(노드 분산) |
| 메모리 오버헤드 | 낮음(버킷/노드) | 높음(RBTree 노드·색상·링크) |
| 재해싱 비용 | 리사이즈 시 큼 | 없음(하지만 삽입/삭제마다 회전/재색칠) |

> **대용량**·**멤버십 테스트 빈번** → HashSet 우선.  
> **정렬/범위 질의**가 핵심 → TreeSet 우선.

---

## 9. 초기 용량 계산(수식)

예상 원소 수 \(N\), 로드 팩터 \(\alpha\)(기본 0.75)일 때,
$$
\text{initialCapacity} = \left\lceil \frac{N}{\alpha} \right\rceil
$$

```java
int expected = 1_000_000;
float lf = 0.75f;
int cap = (int)Math.ceil(expected / lf);
Set<Integer> s = new HashSet<>(cap, lf);
```

---

## 10. 동시성 컬렉션 간단 가이드

| 요구 | 컬렉션 | 메모 |
|---|---|---|
| 고성능 동시성(정렬 불필요) | `ConcurrentHashMap.newKeySet()` | Java 8+, 평균 O(1) |
| 읽기 위주, 소형 | `CopyOnWriteArraySet` | 쓰기 비용 큼 |
| 정렬 + 동시성 | `ConcurrentSkipListSet` | O(log n), 범위 질의 가능 |
| 단순 동기화 래핑 | `Collections.synchronizedSet(new HashSet<>())` | 외부 락 기반 |

```java
import java.util.*;
import java.util.concurrent.*;

public class ConcurrentSets {
  public static void main(String[] args) {
    Set<Integer> hs = ConcurrentHashMap.newKeySet();
    Set<Integer> ts = new ConcurrentSkipListSet<>();
    hs.add(1); ts.add(1);
  }
}
```

---

## 11. fail-fast 반복과 안전한 삭제

```java
Set<String> s = new HashSet<>(Set.of("a","b","c"));
// 잘못된 방법: for-each 중 직접 remove → 예외 가능
// for (String x: s) if (x.equals("b")) s.remove(x);

// 올바른 1: 반복자 remove
for (Iterator<String> it = s.iterator(); it.hasNext();) {
  if (it.next().equals("b")) it.remove();
}

// 올바른 2: 조건부 일괄 삭제
s.removeIf("a"::equals);
```

---

## 12. ASCII 구조 도식

### HashSet (버킷/체이닝/트리화)
```
index: 0    1         2           3 ...
       |    |         |           |
       v    v         v           v
      [ ]  [*]  ->  [*]  ->     [T]
           (Node)    (Node)     (TreeBin)
```

### TreeSet (Red-Black Tree 개념)
```
        (root,B)
        /     \
     (R)       (R)
     / \       / \
   ... ...   ... ...    // 균형 유지 → O(log n)
```

---

## 13. 선택 가이드 (최종)

| 상황 | 추천 | 근거 |
|---|---|---|
| 멤버십 테스트, 집합 연산 빈번, 순서 무관 | **HashSet** | 평균 O(1), 단순/저오버헤드 |
| 정렬 순회/이웃·범위 질의가 핵심 | **TreeSet** | NavigableSet 기능 |
| 중복 제거 + 입력 순서 유지 | **LinkedHashSet** | 이중연결리스트 |
| 정렬 + 멀티스레드 | **ConcurrentSkipListSet** | 정렬/범위 + 동시성 |
| Enum 키 | **EnumSet** | 비트셋 기반 |

---

## 14. 전체 데모 — 두 Set의 성질 비교

```java
import java.util.*;
import java.util.stream.*;

public class SetLandscape {
  public static void main(String[] args) {
    // 1) HashSet: 순서 비보장, 빠른 멤버십
    Set<String> hs = new HashSet<>(List.of("banana","apple","cherry","apple"));
    System.out.println("HashSet      : " + hs);

    // 2) TreeSet: 정렬 + 범위 질의
    NavigableSet<String> ts = new TreeSet<>(hs);
    System.out.println("TreeSet      : " + ts);
    System.out.println("ceiling(b)   : " + ts.ceiling("b"));
    System.out.println("subSet[ba,ch): " + ts.subSet("ba", true, "ch", false));

    // 3) LinkedHashSet: 순서 보존 중복 제거
    List<String> raw = List.of("b","a","b","c","a");
    Set<String> lhs = new LinkedHashSet<>(raw);
    System.out.println("LinkedHashSet: " + lhs);

    // 4) 값 길이 기준 상위 K with TreeSet
    Comparator<String> lenThenLex = Comparator
      .comparingInt(String::length).thenComparing(Comparator.naturalOrder());
    NavigableSet<String> topK = new TreeSet<>(lenThenLex);
    for (String s: List.of("pear","apple","banana","fig","date","avocado")) {
      topK.add(s);
      if (topK.size() > 3) topK.pollFirst();
    }
    System.out.println("TopK by len  : " + topK);
  }
}
```

---

## 15. 체크리스트 (실무 베스트 프랙티스)

- [ ] **요소 불변성**: Set에 넣은 뒤 `equals/hashCode`/`compareTo` 결과가 변하지 않게.
- [ ] **초기 용량**(HashSet) 미리 산정 → 리사이즈 최소화.
- [ ] **Comparator**는 **전순서**(total order), `equals`와 가능하면 일관.
- [ ] 범위 질의 필요 시 **TreeSet**/`NavigableSet` 우선 고려.
- [ ] 대용량/동시성 필요 시 **Concurrent** 계열로 전환.
- [ ] 순서 보존 요구 시 **LinkedHashSet**.

---

### 마무리
- **HashSet**: “기본값” — 최적의 평균 성능으로 대다수 멤버십/집합 연산을 해결합니다.  
- **TreeSet**: “정렬·범위 탐색기” — 사전식 탐색/이웃/구간 처리가 핵심일 때 선택하세요.  
- **LinkedHashSet/ConcurrentSkipListSet/EnumSet**까지 상황에 맞게 조합하면, **성능·표현력·안정성**을 모두 잡을 수 있습니다.