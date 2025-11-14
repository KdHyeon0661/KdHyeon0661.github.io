---
layout: post
title: Java - Math, Arrays, Collections
date: 2025-07-24 16:20:23 +0900
category: Java
---
# Math, Arrays, Collections

## `java.lang.Math` — 수학 함수·정밀도·성능

### 핵심 상수와 구조

- **정적 클래스**: 인스턴스화 불가, 모든 메서드 `static`
- 상수: `Math.PI`, `Math.E`

### 기본 연산 메서드(확장)

| 범주 | 메서드 | 설명/특징 |
|---|---|---|
| 절댓값/최대/최소 | `abs`, `max`, `min` | 원시/랩퍼 오버로드 제공 |
| 거듭제곱/루트 | `pow(a,b)`, `sqrt(x)`, `cbrt(x)`, `hypot(x,y)` | `hypot`은 언더/오버플로 완화 |
| 지수/로그 | `exp(x)`, `expm1(x)`, `log(x)`, `log10(x)`, `log1p(x)` | `expm1`,`log1p`는 **x≈0** 근방 정밀 |
| 삼각함수 | `sin`, `cos`, `tan`, `asin`, `acos`, `atan`, `atan2(y,x)` | `atan2`는 사분면 고려 |
| 반올림 | `round`, `floor`, `ceil`, `rint` | `rint`는 **짝수 쪽** 근사(은행가 반올림) |
| 다음 표현가능 수 | `nextUp`, `nextDown`, `ulp(x)` | **ULP**(Unit in Last Place) 확인 |
| 부호·나눗셈 | `copySign(mag, sign)`, `floorDiv`, `floorMod` | 수학적 바닥 나눗셈/나머지 |
| 난수 | `random()` | 내부적으로 `Random` 공유(멀티스레드 contention 가능) |
| 정확 산술 | `addExact`, `subtractExact`, `multiplyExact`, `incrementExact`, `decrementExact`, `toIntExact` | **오버플로 시 ArithmeticException** |

```java
public class MathBasics {
  public static void main(String[] args) {
    System.out.println(Math.PI);                 // 3.141592653589793
    System.out.println(Math.hypot(3, 4));        // 5.0
    System.out.println(Math.expm1(1e-10));       // 1e-10에 정밀
    System.out.println(Math.rint(2.5));          // 2.0 (짝수 쪽)
    System.out.println(Math.floorDiv(-3, 2));    // -2  (수학적 바닥)
    System.out.println(Math.floorMod(-3, 2));    // 1   (항상 0<=r<|m|)
    System.out.println(Math.nextUp(1.0));        // 1.0000000000000002
  }
}
```

#### 수학적 정의(반올림 비교)

- $$\text{round}(x)\ \text{≈ 가장 가까운 정수, .5는 0에서 멀리}$$
- $$\text{rint}(x)\ \text{≈ 가장 가까운 정수, .5는 짝수쪽}$$
- $$\text{floor}(x)=\max\{n\in\mathbb{Z}\mid n\le x\},\quad \text{ceil}(x)=\min\{n\in\mathbb{Z}\mid n\ge x\}$$

### `StrictMath` vs `Math`

- **`StrictMath`**: 플랫폼 독립적 결정적 결과(fdlibm 기반)
- **`Math`**: JIT/하드웨어 가속 허용(보통 더 빠름). **정확성 vs 일관성** 요구에 따라 선택.

### 난수: 무엇을 쓸까?

| 케이스 | 권장 |
|---|---|
| 단일 스레드/일반 용도 | `new java.util.Random(seed)` 또는 `Math.random()` |
| 다중 스레드 고성능 | `java.util.concurrent.ThreadLocalRandom.current()` |
| 통계/분할·복제 | `java.util.SplittableRandom` |
| 보안(토큰/키) | `java.security.SecureRandom` |

```java
// 멀티스레드에서 contention 줄이기
int r = java.util.concurrent.ThreadLocalRandom.current().nextInt(100);
```

### 금고급(정밀) 주의: 금융/화폐는?

- 부동소수점 **이진 표현** 오차 → **`BigDecimal`** 사용 권장
```java
import java.math.*;
BigDecimal price = new BigDecimal("19.99");
BigDecimal vat = price.multiply(new BigDecimal("0.10"));
System.out.println(vat.setScale(2, RoundingMode.HALF_UP)); // 2.00
```

---

## `java.util.Arrays` — 배열 유틸(정렬·비교·복사·병렬·스트림)

### 대표 메서드(확장 표)

| 그룹 | 메서드 | 설명/주의 |
|---|---|---|
| 출력/비교 | `toString`, `deepToString`, `equals`, `deepEquals`, `compare`, `mismatch` | **다차원 배열**은 `deep*` 사용 |
| 정렬 | `sort`, `parallelSort` | `parallelSort`는 큰 배열에서 유리 |
| 복사 | `copyOf`, `copyOfRange` | 내부적으로 `System.arraycopy` |
| 채우기 | `fill(a,val)`, `fill(a,from,to,val)` | 범위 채우기 지원 |
| 탐색 | `binarySearch(a,key)` | **정렬된 배열** 전제 |
| 생성 | `setAll`, `parallelSetAll` | 인덱스 기반 생성 |
| 스트림 | `Arrays.stream(...)` | 원시 스트림/박싱 변환 |

{% raw %}
```java
import java.util.*;
public class ArraysBasics {
  public static void main(String[] args) {
    int[] a = {5,2,4,1,3};
    Arrays.sort(a);
    System.out.println(Arrays.toString(a));     // [1, 2, 3, 4, 5]

    int[] b = Arrays.copyOf(a, a.length);
    System.out.println(Arrays.equals(a, b));    // true

    int at = Arrays.binarySearch(a, 4);         // 인덱스 3
    System.out.println(at);

    // 다차원 비교
    int[][] m1 = {{1,2},{3,4}}, m2 = {{1,2},{3,4}};
    System.out.println(Arrays.deepEquals(m1, m2)); // true

    // 스트림/박싱
    List<Integer> list = Arrays.stream(a).boxed().toList();
    System.out.println(list);
  }
}
```
{% endraw %}

### 성능/함정 체크리스트

1) **`Arrays.asList(arr)` 함정**
   - **고정 크기 리스트 뷰**: `add/remove` 불가, `set`만 가능
   - **원시 배열**: `int[]`를 넘기면 **단일 요소 리스트**가 됨
   → 해결: `Arrays.stream(intArr).boxed().toList()` 또는 `IntStream.of(...)`
```java
int[] p = {1,2,3};
System.out.println(Arrays.asList(p)); // [[I@xxxx] 단일 요소!
List<Integer> l = Arrays.stream(p).boxed().toList();
```

2) **`System.arraycopy` vs `Arrays.copyOf`**
   - `System.arraycopy`가 미세하게 빠를 수 있으나, **가독성/안전성**은 `copyOf` 선호.

3) **`parallelSort`**
   - 수만~수십만 이상 배열에서 효과. 작은 배열은 오히려 오버헤드.

4) **`mismatch(a,b)`**
   - 달라지는 **최초 인덱스** 반환(없으면 -1). 빠른 비교에 유용.

5) **`compareUnsigned(byte[] ...)`** 없음.
   - 바이트/정수 **부호 없는 비교**는 `Integer.compareUnsigned`, `Byte.toUnsignedInt` 조합.

---

## `java.util.Collections` — 컬렉션 유틸(정렬·뷰·동기화·레시피)

### 대표 메서드(확장 표)

| 그룹 | 메서드 | 설명/주의 |
|---|---|---|
| 정렬/순서 | `sort(list)`, `reverse`, `shuffle`, `rotate`, `swap` | 정렬은 **안정적**(TimSort) |
| 검색/통계 | `binarySearch(list,key)`, `min/max`, `frequency`, `disjoint` | `binarySearch`는 정렬 전제 |
| 구성 | `fill`, `nCopies`, `singleton`, `empty*` | 상수/빈 컬렉션 |
| 불변/뷰 | `unmodifiable*`, `synchronized*`, `checked*` | 수정불가/동기화/타입검사 뷰 |
| 서브리스트 | `indexOfSubList`, `lastIndexOfSubList` | 패턴 검색 |

```java
import java.util.*;

public class CollectionsBasics {
  public static void main(String[] args) {
    List<String> list = new ArrayList<>(Arrays.asList("c","a","b"));
    Collections.sort(list);
    System.out.println(list); // [a, b, c]

    Collections.reverse(list);
    Collections.rotate(list, 1); // 오른쪽 1칸 회전
    System.out.println(list); // [a, c, b]

    System.out.println(Collections.frequency(list, "a")); // 1
    System.out.println(Collections.disjoint(list, List.of("x","y"))); // true
  }
}
```

### 불변 vs 수정불가 vs JDK 9 팩토리

| 종류 | 생성 | 특징 |
|---|---|---|
| **수정불가 뷰** | `Collections.unmodifiableList(base)` | **기저 변경이 보임** |
| **불변 팩토리** | `List.of(...)`, `Set.of(...)`, `Map.of(...)` | 진짜 불변, 중복/`null` 제한 |

```java
var base = new ArrayList<>(List.of(1,2,3));
var view = Collections.unmodifiableList(base);
base.add(4);
System.out.println(view); // [1, 2, 3, 4] (뷰는 기저를 반영)
```

### 동기화/검사 뷰

```java
List<String> unsafe = new ArrayList<>();
List<String> sync = Collections.synchronizedList(unsafe);
List<String> checked = Collections.checkedList(unsafe, String.class);

// 동기화 뷰 반복 시 수동 동기화 권장
synchronized (sync) {
  for (String s : sync) { /* ... */ }
}
```

### 정렬 고급: Comparator 유틸

```java
record Person(String name, int age) {}

List<Person> people = new ArrayList<>(List.of(
  new Person("Ann", 30), new Person("Bob", 25), new Person("Ann", 20)));

people.sort(Comparator.comparing(Person::name)
       .thenComparingInt(Person::age)); // Ann(20), Ann(30), Bob(25)
```
- `naturalOrder()`, `reverseOrder()`, `nullsFirst/Last`, `comparingInt/Long/Double`

### 실전 레시피 모음

**1) 빈도수 테이블**
```java
Map<String,Integer> freq = new HashMap<>();
List.of("a","b","a").forEach(t -> freq.merge(t, 1, Integer::sum));
System.out.println(freq); // {a=2, b=1}
```

**2) 불변 읽기 전용 뷰 전달**
```java
List<String> mutable = new ArrayList<>(List.of("x","y"));
List<String> ro = Collections.unmodifiableList(mutable);
// API 인자로 ro만 노출 → 캡슐화 강화
```

**3) 섞기/씨드 고정(재현성)**
```java
var list = new ArrayList<>(List.of(1,2,3,4,5));
Collections.shuffle(list, new Random(42));
System.out.println(list); // 항상 동일한 순열
```

**4) 서브리스트 패턴 검색**
```java
var hay = List.of(1,2,3,4,2,3);
var needle = List.of(2,3);
System.out.println(Collections.indexOfSubList(hay, needle)); // 1
```

---

## 세 클래스 함께 쓰는 실전 시나리오

### 점수 통계(정렬·최솟값·평균·표준편차)

```java
import java.util.*;

public class StatsExample {
  public static void main(String[] args) {
    List<Integer> scores = new ArrayList<>(Arrays.asList(70, 90, 85, 90, 60, 75));
    Collections.sort(scores); // 오름차순

    int min = Collections.min(scores);
    int max = Collections.max(scores);
    double avg = scores.stream().mapToInt(Integer::intValue).average().orElse(0.0);

    double variance = scores.stream().mapToDouble(s -> (s - avg)*(s - avg)).average().orElse(0.0);
    double stddev = Math.sqrt(variance);

    System.out.printf("min=%d, max=%d, avg=%.2f, stddev=%.3f%n", min, max, avg, stddev);
  }
}
```

### 배열 → 정렬 → 중복 제거(입력 순서 보존)

```java
int[] a = {3,1,2,3,2,1,4};
Arrays.sort(a);
var uniqueOrdered = new java.util.LinkedHashSet<Integer>();
for (int v : a) uniqueOrdered.add(v);
System.out.println(uniqueOrdered); // [1, 2, 3, 4]
```

### 큰 배열 병렬 초기화/연산

```java
int n = 1_000_000;
int[] big = new int[n];
Arrays.parallelSetAll(big, i -> i*i); // 인덱스 제곱
Arrays.parallelPrefix(big, Integer::sum); // 누적합
```

---

## 성능·정확성 체크리스트

1) **반올림 규칙**을 명확히: 재무는 `BigDecimal + RoundingMode`.
2) **멀티스레드 난수**: `ThreadLocalRandom`/`SplittableRandom`.
3) **큰 배열 정렬**: `parallelSort` 고려(프로파일로 확인).
4) **다차원 배열 비교/출력**: `deepEquals`/`deepToString`.
5) **asList 함정**(고정 크기/원시 배열 단일 요소) 피하기.
6) **불변 vs 수정불가 뷰** 구분: 외부에 **진짜 불변**을 노출.
7) **검사 뷰(checked*)**로 런타임 타입 오류 조기 발견.
8) **정렬 대비 동등성**: `Comparator` 기준과 `equals` 일관 유지(특히 Tree 기반 컬렉션 사용 시).
9) **정확 산술** 필요 시 `*Exact` 군 사용해 **오버플로 감지**.
10) **binarySearch** 전 **정렬 필수**(동일 비교 기준 유지).

---

## 요약 표

| 항목 | 포인트 | 한 줄 팁 |
|---|---|---|
| `Math` | 반올림/ULP/정확 산술/난수 | 재무: `BigDecimal` / 멀티스레드: `ThreadLocalRandom` |
| `Arrays` | 정렬/복사/비교/병렬/스트림 | 다차원은 `deep*`, asList/원시 주의 |
| `Collections` | 정렬/통계/불변·동기화·검사 뷰 | 외부 공개는 **불변**로, 내부는 가변 관리 |

---

## 부록) 실습 문제(간단)

1) 실수 배열에서 평균과 표준편차를 `Math`/`Arrays.stream`으로 계산하라.
2) 문자열 배열에서 **대소문자 무시** 정렬 후 **이진 탐색**으로 `"korea"` 위치를 찾되, **동일 Comparator**를 사용하라.
3) 주어진 리스트에서 **상위 K 빈도** 단어를 `Collections.frequency` 대신 **맵 + 정렬**로 구하라.

> 해답 팁: 2)는 `Comparator<String> ci = String.CASE_INSENSITIVE_ORDER; Arrays.sort(a, ci); Arrays.binarySearch(a, "korea", ci)`.

---

## 참고 수식(반올림 비교의 직관)

- $$\text{round}(x) = \left\lfloor x + \tfrac{1}{2} \right\rfloor \ \ (\text{for } x\ge 0),\quad \text{음수는 구현 정의에 유의}$$
- $$\text{rint}(x) = \operatorname*{argmin}_{n\in\mathbb{Z}} |x-n| \quad(\text{동일 거리면 짝수 } n)$$

---

## 마무리

- `Math`는 **정확성**(반올림/정밀), `Arrays`는 **배열 조작/병렬**, `Collections`는 **컬렉션 조립/뷰/레시피**의 기반입니다.
- 이 세 축을 정확히 이해하면, **성능·안정성·가독성**이 모두 올라갑니다.
- 실무에서는 **불변 노출 + 내부 가변 캡슐화**, **정확 산술/반올림 규칙 명시**, **프로파일 기반의 병렬화**가 베스트 프랙티스입니다.
