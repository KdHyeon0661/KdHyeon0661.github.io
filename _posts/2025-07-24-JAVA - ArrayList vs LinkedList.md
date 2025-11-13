---
layout: post
title: Java - ArrayList vs LinkedList
date: 2025-07-24 18:20:23 +0900
category: Java
---
# Java 컬렉션 상세 : ArrayList vs LinkedList

## 0. 한눈 요약 (Cheat Sheet)

| 질문 | 권장 선택 | 이유/메모 |
|---|---|---|
| **랜덤 인덱스 접근**이 많다 | **ArrayList** | O(1) 인덱스 접근, 캐시 지역성 매우 우수 |
| **중간 삽입/삭제**가 많다 | **LinkedList (＋ListIterator)** 또는 **재설계** | 인덱스로는 O(n), **Iterator 위치를 잡아둔 경우** O(1) / 다만 실제로는 ArrayList가 더 빠른 경우 많음 |
| **큐/덱**(앞·뒤에서 삽입/삭제) | **ArrayDeque** 우선, 그다음 LinkedList | JDK 권장: 덱/스택은 ArrayDeque가 대체로 더 빠름 |
| **메모리 효율** | ArrayList | Node 오브젝트/포인터 오버헤드 없음 |
| **빈번한 contains/remove(Object)** | 비슷(모두 O(n)) | 단, ArrayList가 캐시 친화적이라 체감상 빠른 편 |
| **정렬(sort)** | ArrayList | 내부 배열 기반 정렬이 유리(추가 메모리 적고 빠름) |
| **동시성 필요** | 별도 고려 | `Collections.synchronizedList`, `CopyOnWriteArrayList`, `ConcurrentLinkedDeque` 등 |

---

## 1. 내부 구조와 핵심 개념

### 1.1 ArrayList — “동적 배열(Dynamic Array)”
- 내부에 **연속된 `Object[]` 배열**을 보유.
- 공간이 부족해지면 **확장(reallocate)**: 보통 **1.5배** 수준으로 증가(JDK 버전별 세부 구현은 약간 상이할 수 있으나 개념적으로 1.5x).
- 인덱스 접근은 `array[index]` → **O(1)**.
- **장점**: 캐시 지역성(연속 메모리)으로 실제 런타임 성능이 매우 좋음.
- **비용**: 중간 삽입/삭제 시 **요소 시프트(copy)** 필요 → O(n).

핵심 API:
```java
ArrayList<E> list = new ArrayList<>();
list.ensureCapacity(100_000); // 대량 추가 전 확장 비용 줄이기
list.trimToSize();            // 여유 capacity를 size에 맞춰 줄이기
```

### 1.2 LinkedList — “이중 연결 리스트(Doubly Linked List)”
- 각 요소는 `Node { E item; Node prev; Node next; }`로 **개별 객체**.
- **앞/뒤 노드 참조**를 통한 연결.
- **장점**: **현재 위치(ListIterator)가 있다면** 그 지점에서의 삽입/삭제가 **O(1)**.
- **비용**: 임의 인덱스로 접근하려면 앞 또는 뒤에서부터 **선형 탐색** → O(n).
  Node 객체/포인터로 인한 **메모리·GC 오버헤드**도 큼.

핵심 API(Deque 인터페이스도 구현):
```java
LinkedList<E> list = new LinkedList<>();
list.addFirst(x); list.addLast(y);
list.removeFirst(); list.removeLast();
list.peekFirst(); list.peekLast();
list.push(x); list.pop(); // 스택처럼
```

> **중요 정정**: “LinkedList의 중간 삽입/삭제가 O(1)”은 **정확히 말하면 ‘해당 노드 참조를 이미 보유한 경우’**입니다.
> 보통은 **해당 지점까지 O(n) 탐색** 후 O(1) 링크 갱신이 일어나므로 총 O(n)이 됩니다.

---

## 2. 시간복잡도(정밀 버전)

| 연산 | ArrayList | LinkedList | 비고 |
|---|---|---|---|
| `get(i)` | **O(1)** | **O(n)** | LinkedList는 `min(i, n-i)`만큼 전진/후진 탐색 |
| `set(i, e)` | **O(1)** | **O(n)** | 접근 비용 동일 |
| `add(e)` (끝) | **암화 O(1)**, 가끔 확장 O(n) | **O(1)** | LinkedList는 tail 참조로 상수 시간 |
| `add(i, e)` (중간) | **O(n)** (시프트) | **탐색 O(n) + 링크 O(1)** ≈ O(n) | Iterator로 지점 고정 시 O(1) 추가 |
| `remove(i)` | **O(n)** (시프트) | **탐색 O(n) + 링크 O(1)** ≈ O(n) | Iterator 위치에서 제거는 O(1) |
| `contains(x)` | O(n) | O(n) | ArrayList가 체감상 빠른 경향(캐시/분기 예측) |
| `iterator.remove()` | O(n) 가능 | **O(1)** | LinkedList 강점 포인트 |
| `sort` | 매우 유리 | 상대적 불리 | LinkedList는 보통 배열로 복사 후 정렬 |

> 실제 성능은 **캐시 지역성**(ArrayList 유리), **분기 예측**, **JIT 최적화** 등 JVM 요인에 크게 좌우됩니다.

---

## 3. 실무에서 자주 쓰는 패턴과 팁

### 3.1 대량 추가 성능 올리기 (ArrayList)
```java
import java.util.*;

public class BulkAdd {
    public static void main(String[] args) {
        List<Integer> src = new ArrayList<>();
        for (int i = 0; i < 1_000_000; i++) src.add(i);

        ArrayList<Integer> dst = new ArrayList<>(src.size()); // 초기 용량 지정
        dst.addAll(src); // 내부 System.arraycopy 활용 → 매우 빠름
    }
}
```
- **초기 용량 지정** 또는 `ensureCapacity`로 **재할당 횟수** 최소화.

### 3.2 중간 삽입/삭제가 잦은 경우의 안전한 선택
- **반드시** `ListIterator`로 **위치 고정** 후 `add/remove`(O(1))를 활용.
- 예시: 편집기 버퍼, 슬라이드 이동 같은 **인접 위치 변형이 잦은 시나리오**.

```java
import java.util.*;

public class InsertManyWithIterator {
    public static void main(String[] args) {
        LinkedList<Integer> list = new LinkedList<>();
        for (int i = 0; i < 10; i++) list.add(i);

        ListIterator<Integer> it = list.listIterator(5); // 중간 지점 "참조" 확보
        it.add(100); // O(1)
        it.add(200); // O(1)
        it.previous(); it.remove(); // 현재 노드 제거도 O(1)
        System.out.println(list);
    }
}
```

> **대부분의 비즈니스 로직**에서는 **ArrayList 재설계**(배치 삽입, 끝 중심 처리)만으로도 성능/단순성을 담보할 수 있습니다.

### 3.3 큐/덱/스택
- **ArrayDeque**가 일반적으로 **LinkedList보다 빠름** (박싱 없이 원형 배열).
- 단, **자연스러운 노드 교체/이동**이 핵심이면 `LinkedList` + `ListIterator`.

```java
import java.util.ArrayDeque;
import java.util.Deque;

public class UseArrayDeque {
    public static void main(String[] args) {
        Deque<String> dq = new ArrayDeque<>();
        dq.addLast("A"); dq.addLast("B");
        dq.addFirst("Z");
        System.out.println(dq.removeFirst()); // Z
        System.out.println(dq.removeLast());  // B
    }
}
```

---

## 4. 실패-안전(Fail-Fast) 반복자 & 동시수정

- 두 구현 모두 **modCount** 기반 **fail-fast iterator**: 반복 중 구조가 바뀌면 `ConcurrentModificationException`.
- 해결:
  - 반복 중 구조 변경 대신 **Iterator의 `remove()`** 사용
  - 병렬 환경이면 **동시성 컬렉션** 사용
    (`CopyOnWriteArrayList`, `ConcurrentLinkedDeque` 등)
  - 또는 `Collections.synchronizedList(list)`로 감싸되, **반복 전체를 동기화 블록**으로 감싸기.

```java
import java.util.*;

public class FailFastDemo {
    public static void main(String[] args) {
        List<Integer> list = new ArrayList<>(List.of(1,2,3,4,5));
        for (Integer x : list) {
            if (x == 3) list.remove(Integer.valueOf(3)); // ConcurrentModificationException
        }
    }
}
```

정상 패턴:
```java
List<Integer> list = new ArrayList<>(List.of(1,2,3,4,5));
Iterator<Integer> it = list.iterator();
while (it.hasNext()) {
    if (it.next() == 3) it.remove(); // OK
}
```

---

## 5. RandomAccess 마커와 알고리즘 분기

- `ArrayList`는 `RandomAccess`를 구현(빠른 인덱스 접근 신호).
- `LinkedList`는 **미구현** → 인덱스 반복 대신 **Iterator** 기반 루프가 유리.

```java
import java.util.*;

static <E> void process(List<E> list) {
    if (list instanceof RandomAccess) {
        for (int i = 0; i < list.size(); i++) {
            E e = list.get(i); // 인덱스 루프
            // ...
        }
    } else {
        for (E e : list) { // Iterator 기반
            // ...
        }
    }
}
```

---

## 6. 정렬/검색/부분보기(subList)

### 6.1 정렬
- `ArrayList`는 내부 배열 기반 → **TimSort** 최적화가 잘 먹힘.
- `LinkedList`는 보통 **배열로 복사 후 정렬**하거나, 리스트 자체 정렬 시 **추가 비용**(메모리/탐색)이 따름.

```java
import java.util.*;

List<Integer> a = new ArrayList<>(List.of(3,1,4,1,5));
a.sort(Integer::compareTo); // TimSort
System.out.println(a);
```

### 6.2 이진검색
- `Collections.binarySearch(list, key)`는 **정렬된 리스트** + **RandomAccess**일 때 유리.
- `LinkedList`에 이진검색은 이점 거의 없음(랜덤 접근 비용 O(n)).

### 6.3 subList 뷰
- `list.subList(from, to)`는 **원본의 뷰**. 구조 변경 시 **동기화 필요 및 예외** 주의.
- **원본/뷰 중 한 쪽의 구조적 변경**이 다른 쪽 반복자에 영향을 줌.

---

## 7. 메모리/GC 관점

| 항목 | ArrayList | LinkedList |
|---|---|---|
| 요소 저장 | 연속 배열의 슬롯 | **Node 객체**(item + 2 refs) |
| 오버헤드 | 낮음 | 높음(객체 헤더 + prev/next 참조) |
| GC 압력 | 상대적으로 낮음 | 상대적으로 높음 |

> 정확한 바이트 수는 JVM/압축 OOPS 등 환경에 의존. **경향**만 기억:
> LinkedList는 **요소 수만큼 추가 객체**가 생성되어 **메모리/GC 비용이 증가**.

---

## 8. 실전 시나리오 의사결정

### 8.1 로그 버퍼(읽기 많고, 끝에 추가)
- **ArrayList**
- 주기적으로 용량 증가가 걱정되면 **초기 용량** 지정

### 8.2 문서 편집 버퍼(커서 인근 삽입/삭제 잦음)
- `LinkedList` + **커서용 ListIterator 유지**
- 또는 **ArrayList + 배치 편집 모델**(작업을 모아서 한 번에 재구성)로 재설계

### 8.3 메시지 큐/덱
- **ArrayDeque** 우선
- 이미 `List` API가 꼭 필요하다면 `LinkedList`의 `Deque` 메서드 사용

### 8.4 정렬/검색 중심(대량)
- **ArrayList** + `sort` + `binarySearch`
- 필요 시 `Collections.unmodifiableList`로 불변 뷰 제공

---

## 9. 자주 하는 실수와 예방

| 실수 | 문제 | 예방책 |
|---|---|---|
| “**중간 삽입 많으니 LinkedList**” | 실제론 인덱스 접근 O(n) 때문에 **더 느려짐** | **Iterator 위치 고정** 또는 **ArrayList 재설계** |
| 반복 중 `list.remove(i)` | `ConcurrentModificationException`, O(n) 시프트 | **Iterator.remove** 사용 |
| 큐/스택으로 LinkedList 습관적 사용 | 대부분 **ArrayDeque가 더 빠름** | 덱/스택은 **ArrayDeque** 우선 |
| 무분별한 `contains` 반복 | O(n) 반복으로 전체 O(n^2) | `HashSet` 병행 사용 / 인덱스 맵 유지 |
| 정렬·검색 전에 정렬 보장 X | `binarySearch` 등 무의미 | 정렬 상태 관리 / 입력 순간 정렬 |

---

## 10. 간단 퍼포먼스 데모(주의: 진짜 벤치는 JMH 권장)

> **주의**: `System.nanoTime()` 미세 측정은 JIT/GC/워밍업 영향으로 왜곡될 수 있습니다. 신뢰도 높은 측정엔 **JMH**를 사용하세요.

```java
import java.util.*;

public class Micro {
    public static void main(String[] args) {
        int N = 200_000;

        // ArrayList 끝 추가
        long t1 = System.nanoTime();
        ArrayList<Integer> a = new ArrayList<>();
        for (int i = 0; i < N; i++) a.add(i);
        long t2 = System.nanoTime();

        // LinkedList 끝 추가
        long t3 = System.nanoTime();
        LinkedList<Integer> l = new LinkedList<>();
        for (int i = 0; i < N; i++) l.add(i);
        long t4 = System.nanoTime();

        // 중간 삽입(인덱스로)
        long t5 = System.nanoTime();
        for (int i = 0; i < 2_000; i++) a.add(a.size()/2, -1);
        long t6 = System.nanoTime();

        long t7 = System.nanoTime();
        for (int i = 0; i < 2_000; i++) l.add(l.size()/2, -1); // 탐색 O(n) 때문에 비용 큼
        long t8 = System.nanoTime();

        System.out.printf("ArrayList add tail:  %.2f ms%n", (t2 - t1)/1e6);
        System.out.printf("LinkedList add tail: %.2f ms%n", (t4 - t3)/1e6);
        System.out.printf("ArrayList mid add:   %.2f ms%n", (t6 - t5)/1e6);
        System.out.printf("LinkedList mid add:  %.2f ms%n", (t8 - t7)/1e6);
    }
}
```

- 대개 **중간 삽입**은 `LinkedList`도 **인덱스 기반**이면 매우 느립니다.
  **반드시** `ListIterator`로 위치 고정 후 O(1) 삽입을 이용하세요.

---

## 11. 안전한 API 사용 예제 모음

### 11.1 안전한 제거(조건부)
```java
List<String> a = new ArrayList<>(List.of("a","b","c","d"));
a.removeIf(s -> s.equals("b")); // 내부적으로 Iterator.remove 사용
```

### 11.2 정렬 후 이진검색
```java
List<Integer> a = new ArrayList<>(List.of(5,1,4,2,3));
a.sort(Integer::compareTo);
int idx = Collections.binarySearch(a, 3); // >=0이면 존재
```

### 11.3 subList 안전 사용
```java
List<Integer> a = new ArrayList<>(List.of(1,2,3,4,5,6,7));
List<Integer> view = a.subList(2, 5); // [3,4,5]
view.set(0, 30); // 원본 반영
// 구조적 변경은 주의(원본/뷰 동시 수정 금지)
```

### 11.4 대규모 초기화
```java
int n = 1_000_000;
ArrayList<Integer> a = new ArrayList<>(n);
for (int i = 0; i < n; i++) a.add(i);
```

---

## 12. 최종 선택 가이드(간단 표)

| 상황 | ArrayList | LinkedList | 코멘트 |
|---|---|---|---|
| 랜덤 접근 다수 | ✅ | ❌ | 성능 차이 큼 |
| 끝에서만 추가/삭제 | ✅ | ✅ | 둘 다 좋음(덱이면 ArrayDeque 권장) |
| **중간** 삽입/삭제 | ⚠️ | ✅(단, Iterator 고정) | 인덱스로는 둘 다 O(n) |
| 메모리 효율 | ✅ | ❌ | Node/포인터 오버헤드 |
| 정렬/검색 | ✅ | ❌ | ArrayList 유리 |
| 큐/덱/스택 | **ArrayDeque** | (차선) | 덱은 ArrayDeque 우선 |

---

## 13. 결론

- **ArrayList**는 “연속 배열” 기반으로 **대부분의 일반적 시나리오에서 더 빠르고 간단**합니다.
- **LinkedList**는 **Iterator로 위치를 붙잡아 두고 주변을 국소적으로 조작**하는 특수 상황에서 의미가 있습니다.
- 큐/덱/스택은 **ArrayDeque**가 실전 표준입니다.
- 성능은 **이론 복잡도 + JVM/하드웨어 특성**의 합. **측정(JMH)**으로 검증하세요.

---

## 부록 A) 흔한 인터뷰 질문 정리

1. **LinkedList 중간 삽입 O(1)인가요?**
   → **그 지점의 노드 참조를 이미 갖고 있으면** O(1). 보통은 **탐색 O(n)+O(1)**로 **O(n)**.

2. **왜 ArrayList가 실무에서 더 빠르다고 하나요?**
   → **캐시 지역성**과 **분기/포인터 추적 비용** 차이. 현대 CPU에서 연속 메모리 접근이 압도적으로 유리.

3. **정렬/검색은 어떤 게 유리?**
   → **ArrayList**. LinkedList는 배열로 복사 후 정렬 등이 필요해 비용↑.

4. **동시성?**
   → 둘 다 비동기 안전 아님. 필요 시 **동시성 컬렉션/동기화 래퍼** 사용.

---

## 부록 B) 단위 테스트/벤치마크 권장

- 정밀 성능 측정: **JMH** 사용(워밍업, 포크, 스레딩, GC 제어).
- 기능 검증: **Iterator.remove**, **subList 동작**, **fail-fast** 예외까지 테스트.
