---
layout: post
title: Java - Stream API
date: 2025-08-04 17:20:23 +0900
category: Java
---
# Java Stream API

## Stream의 개념/철학

- **선언적 파이프라인**: “무엇을” 변환/필터링/집계할지 선언하고, “어떻게”(루프/인덱스)는 라이브러리에 위임.
- **내부 반복(Internal Iteration)**: 런타임이 최적의 순회/병렬화를 결정.
- **불변 처리**: 원본을 변경하지 않고 변환 결과를 생산(함수형 스타일).
- **지연 평가(Lazy)**: **중간 연산은 실행 계획**이며, **최종 연산**이 호출될 때 실제 연산이 수행.

---

## 스트림 생성 총망라

### 컬렉션

```java
List<String> list = Arrays.asList("a","b","c");
Stream<String> s1 = list.stream();          // 순차
Stream<String> s2 = list.parallelStream();  // 병렬
```

### 배열/가변 인자

```java
String[] arr = {"a","b","c"};
Stream<String> s = Arrays.stream(arr);
Stream<Integer> s2 = Stream.of(1,2,3,4);
```

### 무한 스트림

```java
Stream<Integer> evens = Stream.iterate(0, n -> n + 2);           // 무한
Stream<Double> rnd   = Stream.generate(Math::random);            // 무한

// JDK 9+: 조건 포함 iterate
Stream<Integer> to100 = Stream.iterate(0, n -> n <= 100, n -> n + 10);
```

### 파일/디렉터리 (I/O)

```java
try (Stream<String> lines = java.nio.file.Files.lines(path)) {
    long nonEmpty = lines.filter(l -> !l.isEmpty()).count();
}

try (Stream<java.nio.file.Path> walk = java.nio.file.Files.walk(root)) {
    List<java.nio.file.Path> md = walk
        .filter(p -> p.toString().endsWith(".md"))
        .collect(Collectors.toList());
}
```
> **리소스는 반드시 닫기**: `Files.lines/walk`는 `Stream`이 `AutoCloseable`.

### 빌더

```java
Stream<String> built = Stream.<String>builder().add("x").add("y").build();
```

### `Stream.ofNullable` (JDK 9+)

```java
Stream<String> s = Stream.ofNullable(maybeNull); // null→빈 스트림, 값→하나짜리 스트림
```

### Spliterator 기반 (고급)

```java
Spliterator<Integer> spl = new Spliterators.AbstractSpliterator<Integer>(Long.MAX_VALUE,
        Spliterator.ORDERED | Spliterator.NONNULL) {
    int cur = 0;
    public boolean tryAdvance(Consumer<? super Integer> act) {
        if (cur >= 10) return false;
        act.accept(cur++);
        return true;
    }
};
Stream<Integer> custom = StreamSupport.stream(spl, false);
```

---

## 중간 연산(지연)

| 연산 | 설명 | 주의 |
|---|---|---|
| `filter(Predicate)` | 조건 필터 | 순서 유지 |
| `map(Function)` | 1:1 변환 | 박싱 비용 주의 |
| `flatMap(Function)` | 1:N → 평탄화 | 과도한 스트림 생성 주의 |
| `mapMulti(BiConsumer)` (JDK 16+) | flatMap 대체(컬렉션 생성 없이 다중 emit) | 고성능 평탄화 |
| `distinct()` | 중복 제거(상태ful) | 메모리 사용 증가 |
| `sorted(Comparator)` | 정렬(상태ful) | O(n log n), 병렬 시 비용 큼 |
| `limit/skip` | 슬라이싱(단락) | 병렬은 비효율 가능 |
| `peek(Consumer)` | 관찰/디버깅 | 부작용 로직은 금지 |
| `unordered()` | 순서 무시 힌트 | 병렬 최적화에 유익 |

**예시**
```java
List<String> names = Arrays.asList("Kim","Lee","Park","Lee");
names.stream()
     .filter(n -> n.startsWith("L"))
     .distinct()
     .sorted()
     .forEach(System.out::println);
```

---

## 최종 연산(소모)

| 연산 | 설명 |
|---|---|
| `forEach/forEachOrdered` | 처리(병렬 시 순서 보장 여부 차이) |
| `toArray` / `collect` | 배열/컬렉션/맵 등으로 수집 |
| `reduce` | 누적(식별자·결합자 필요) |
| `count/sum/average`(원시형) | 집계 |
| `anyMatch/allMatch/noneMatch` | 단락(짧게 끝남) |
| `findFirst/findAny` | 요소 검색(단락) |
| `max/min` | 최댓/최솟 |

**단락 예시**
```java
boolean hasLarge = nums.stream().anyMatch(x -> x > 1_000_000);
Optional<String> firstLee = names.stream().filter(n -> n.startsWith("Lee")).findFirst();
```

---

## 매핑·평탄화·보조 API

### `map` vs `flatMap`

```java
// map: 1→1
List<Integer> lengths = words.stream().map(String::length).collect(Collectors.toList());

// flatMap: 1→N
List<String> tokens = lines.stream()
    .flatMap(l -> Arrays.stream(l.split("\\s+")))
    .collect(Collectors.toList());
```

### `mapMulti` (JDK 16+)

```java
List<String> tokens = lines.stream()
    .mapMulti((line, sink) -> {
        for (String t : line.split("\\s+")) sink.accept(t);
    })
    .toList(); // JDK 16+: 불변에 가까운 뷰
```
> `mapMulti`는 중간 컬렉션을 만들지 않아 `flatMap`보다 메모리/할당 면에서 유리.

### `takeWhile/dropWhile` (JDK 9+)

```java
List<Integer> prefix = nums.stream().takeWhile(n -> n < 1000).toList();
List<Integer> tail   = nums.stream().dropWhile(n -> n < 1000).toList();
```

---

## 원시형 스트림과 박싱 회피

- `IntStream`, `LongStream`, `DoubleStream` 제공
  (중간: `map`, `filter`, `distinct`, `sorted`, `limit` / 최종: `sum`, `average`, `summaryStatistics`, `toArray`)

```java
int sum = IntStream.rangeClosed(1, 1_000_000).sum();
double avg = IntStream.of(1,2,3).average().orElse(0);
List<Integer> list = IntStream.range(0,10).boxed().collect(Collectors.toList()); // boxed()로 객체화
```

> **핫패스** 수치 연산은 원시형 스트림으로 박싱/언박싱 비용을 줄이세요.

---

## Collectors 심화(Downstream/불변/요약/teeing)

### 기본 수집

```java
List<String> list = stream.collect(Collectors.toList()); // 가변 리스트 (구현 비공개)
List<String> u = stream.collect(Collectors.toUnmodifiableList()); // JDK 10+: 불변
List<String> u2 = stream.toList(); // JDK 16+: 변경 불가 뷰(사실상 불변)
Set<String> set = stream.collect(Collectors.toSet());
```

### 맵 수집과 중복 키 처리

```java
Map<Integer, String> idx = words.stream()
    .collect(Collectors.toMap(String::length, w -> w, (a,b) -> a, LinkedHashMap::new));
```
- 세 번째 인자: **병합 함수**(중복 키 충돌 해결). 지정하지 않으면 `IllegalStateException`.

### 그룹화/분할/다운스트림

```java
Map<Integer, List<String>> byLen =
    words.stream().collect(Collectors.groupingBy(String::length));

Map<Integer, Set<String>> byLenSet =
    words.stream().collect(Collectors.groupingBy(String::length, Collectors.toSet()));

Map<Boolean, List<String>> part =
    words.stream().collect(Collectors.partitioningBy(w -> w.length() >= 5));
```

#### filtering / mapping / flatMapping (JDK 9+)

```java
Map<Integer, List<String>> onlyL =
    words.stream().collect(Collectors.groupingBy(
        String::length,
        Collectors.filtering(w -> w.startsWith("L"), Collectors.toList())
    ));

Map<Integer, Set<Character>> initials =
    words.stream().collect(Collectors.groupingBy(
        String::length,
        Collectors.mapping(w -> w.charAt(0), Collectors.toSet())
    ));
```

### 요약 통계

```java
IntSummaryStatistics st =
    words.stream().collect(Collectors.summarizingInt(String::length));
long count = st.getCount(); int max = st.getMax(); double avg = st.getAverage();
```

### joining

```java
String csv = words.stream().collect(Collectors.joining(", ", "[", "]"));
```

### collectingAndThen (후처리)

```java
List<String> unmodifiable =
    words.stream().collect(Collectors.collectingAndThen(
        Collectors.toList(), Collections::unmodifiableList));
```

### teeing (JDK 12+): 두 결과를 동시에 결합

```java
record Stats(long count, int max) {}
Stats s = words.stream().collect(Collectors.teeing(
    Collectors.counting(),
    Collectors.mapping(String::length, Collectors.maxBy(Integer::compareTo)),
    (c, m) -> new Stats(c, m.orElse(0))
));
```

### 병렬 수집과 CONCURRENT

- `groupingByConcurrent`는 **무주문(unordered)** 스트림과 함께일 때 효과적.
- 순서가 중요하면 일반 `groupingBy` 사용.

---

## reduce 심화 — 항등원/결합법칙

### 정석 서명

- `T reduce(T identity, BinaryOperator<T> accumulator)`
- `<U> U reduce(U identity, BiFunction<U,? super T,U> accumulator, BinaryOperator<U> combiner)`

### 병렬 안전 조건

- 결합자(Combiner)와 누산자(Accumulator)는 **결합법칙(associative)**이어야 병렬에서 정확.

$$
(a \circ b) \circ c = a \circ (b \circ c)
$$

**나쁜 예** (감산은 결합법칙 X):
```java
int bad = IntStream.range(1, 10).parallel().reduce(0, (a,b) -> a - b); // 비결정적/오류
```

**올바른 예**:
```java
int sum = IntStream.range(1, 10).parallel().reduce(0, Integer::sum);
```

> 가능하면 복잡한 축약은 `collect`(가변 누산기)로 구현하고, `reduce`는 **교환·결합 가능 연산**에 한정.

---

## 병렬 스트림 — 가이드

### 이득이 되는 조건

- 데이터가 **크고**, 연산이 **CPU 바운드**이며, **무상태/독립**이고, **분할(splitting) 비용**이 낮을 때.

### 위험/주의

- 외부 상태 변경(사이드 이펙트) 금지: `parallelStream().forEach(list::add)` → 경쟁/오류
- 순서 필요한 경우 `forEachOrdered`. (병렬성 일부 희생)
- `findAny`는 병렬 친화, `findFirst`는 순서 제약으로 병렬 이점 감소.
- 작은 컬렉션/가벼운 연산에는 오히려 느릴 수 있음(스레드 오버헤드).

### 실무 팁

- 병렬 전/후 **측정**(JMH, Micrometer)로 검증.
- I/O-블로킹은 병렬 스트림보다 **별도 Executor**(CompletableFuture 등) 고려.
- `unordered()` 힌트로 최적화 여지 제공.

---

## Stream × Optional 결합

```java
Optional<String> maybe = words.stream().filter(w -> w.startsWith("K")).findFirst();
maybe.ifPresent(System.out::println);

// JDK 9+: Optional.stream()
List<String> nicknames = users.stream()
    .map(User::nicknameOptional)     // Stream<Optional<String>>
    .flatMap(Optional::stream)       // Stream<String>
    .toList();
```

---

## I/O 스트림과 자원 관리

```java
// 파일 행 스트림 처리 (자동 닫힘)
try (Stream<String> lines = Files.lines(path, StandardCharsets.UTF_8)) {
    long errors = lines.filter(l -> l.contains("ERROR")).count();
}
```

> `BufferedReader.lines()`도 `Stream<String>`을 반환. 항상 **try-with-resources**로 감싸세요.

---

## 예외 처리 패턴

### 체크 예외를 Optional/Result로 래핑

```java
static <T> Optional<T> wrap(Callable<T> c) {
    try { return Optional.ofNullable(c.call()); }
    catch (Exception e) { return Optional.empty(); }
}

List<Integer> ints = lines.stream()
    .flatMap(l -> wrap(() -> Integer.parseInt(l)).stream())
    .toList();
```

### 사용자 유틸로 감싸기

```java
@FunctionalInterface interface ThrowingFn<T,R> { R apply(T t) throws Exception; }
static <T,R> Function<T,R> sneaky(ThrowingFn<T,R> f) {
    return t -> {
        try { return f.apply(t); }
        catch (RuntimeException re) { throw re; }
        catch (Exception e) { throw new RuntimeException(e); }
    };
}

List<Integer> ints = lines.stream()
    .map(sneaky(Integer::parseInt))
    .toList();
```

---

## 커스텀 Collector & Spliterator

### Collector 직접 구현(마지막 N개만 유지)

```java
class LastN<T> implements Collector<T, Deque<T>, List<T>> {
    private final int n;
    LastN(int n) { this.n = n; }

    public Supplier<Deque<T>> supplier() { return ArrayDeque::new; }

    public BiConsumer<Deque<T>, T> accumulator() {
        return (dq, t) -> { dq.addLast(t); if (dq.size() > n) dq.removeFirst(); };
    }

    public BinaryOperator<Deque<T>> combiner() {
        return (a, b) -> { while (b.size() > 0) { a.addLast(b.removeFirst()); if (a.size() > n) a.removeFirst(); } return a; };
    }

    public Function<Deque<T>, List<T>> finisher() { return ArrayList::new; }

    public Set<Characteristics> characteristics() { return Collections.emptySet(); }
}
```
사용:
```java
List<Integer> tail100 = IntStream.range(0, 10_000).boxed().collect(new LastN<>(100));
```

### Spliterator로 균등 청크 분할 (병렬 친화)

```java
class ChunkSpliterator implements Spliterator<List<Integer>> {
    final int[] arr; final int chunk; int index = 0;
    ChunkSpliterator(int[] arr, int chunk) { this.arr = arr; this.chunk = chunk; }

    public boolean tryAdvance(Consumer<? super List<Integer>> action) {
        if (index >= arr.length) return false;
        int end = Math.min(arr.length, index + chunk);
        List<Integer> ret = new ArrayList<>(end - index);
        for (int i = index; i < end; i++) ret.add(arr[i]);
        index = end; action.accept(ret); return true;
    }
    public Spliterator<List<Integer>> trySplit() {
        int remaining = arr.length - index;
        if (remaining <= chunk * 2) return null;
        int mid = index + remaining/2;
        int[] right = Arrays.copyOfRange(arr, mid, arr.length);
        return new ChunkSpliterator(right, chunk);
    }
    public long estimateSize() { return (arr.length - index + chunk - 1) / chunk; }
    public int characteristics() { return ORDERED | SIZED | SUBSIZED | NONNULL | IMMUTABLE; }
}
```
사용:
```java
int[] big = IntStream.range(0, 1_000_000).toArray();
Stream<List<Integer>> chunks = StreamSupport.stream(new ChunkSpliterator(big, 10_000), true);
long sum = chunks.parallel().flatMapToInt(li -> li.stream().mapToInt(Integer::intValue)).asLongStream().sum();
```

---

## 성능/안티패턴 체크리스트

- [ ] **외부 상태 변경 금지**: `forEach(list::add)` 대신 `collect(toList())`.
- [ ] **중간 컬렉션 방지**: `flatMap`/`mapMulti`로 바로 평탄화.
- [ ] **박싱 회피**: 수치는 `IntStream/LongStream/DoubleStream`.
- [ ] **과한 병렬화 금지**: 소규모 데이터/가벼운 연산은 순차가 낫다.
- [ ] **정렬/중복제거 남용 자제**: `sorted/distinct`는 상태 보관·비용 큼.
- [ ] **`peek`는 디버깅 전용**: 로직 의존 금지.
- [ ] **스트림 재사용 금지**: 최종 연산 후 다시 쓰면 `IllegalStateException`.
- [ ] **Collectors 중복 키**: `toMap` 병합 함수 지정.
- [ ] **I/O 스트림 닫기**: `try (Stream<?> s = …)`.
- [ ] **reduce 결합법칙**: 병렬에 안전한 연산만.
- [ ] **`forEachOrdered` 필요성 판단**: 순서 보장이 성능을 깎을 수 있음.
- [ ] **`unordered()` 힌트**: 순서 불필요시 최적화 여지 제공.

---

## 실무 레시피 15선

### 상위 K개(우선순위 큐)

```java
int K = 10;
List<Item> topK = items.stream().collect(
    Collector.of(
        () -> new PriorityQueue<Item>(Comparator.comparingInt(Item::score)), // min-heap
        (pq, it) -> { pq.offer(it); if (pq.size() > K) pq.poll(); },
        (a, b) -> { b.forEach(x -> { a.offer(x); if (a.size() > K) a.poll();}); return a; },
        pq -> { ArrayList<Item> r = new ArrayList<>(pq); r.sort(Comparator.comparingInt(Item::score).reversed()); return r; }
    )
);
```

### 빈도수 맵

```java
Map<String, Long> freq = words.stream()
    .collect(Collectors.groupingBy(w -> w, Collectors.counting()));
```

### 다중 키 그룹(복합 키)

```java
record Key(String city, int year) {}
Map<Key, List<Order>> byCityYear =
    orders.stream().collect(Collectors.groupingBy(o -> new Key(o.city(), o.year())));
```

### 인덱스화(값→인덱스)

```java
Map<String, Integer> index =
    IntStream.range(0, list.size()).boxed()
        .collect(Collectors.toMap(list::get, i -> i, (a,b) -> a, LinkedHashMap::new));
```

### 평균 = sum/count (teeing)

```java
double avgLen = words.stream().collect(Collectors.teeing(
    Collectors.summingInt(String::length),
    Collectors.counting(),
    (sum, cnt) -> cnt == 0 ? 0.0 : ((double) sum) / cnt
));
```

### 안정적 정렬 + 중복 제거 (Comparator + TreeSet)

```java
List<User> uniqSorted = users.stream()
    .collect(Collectors.collectingAndThen(
        Collectors.toCollection(() -> new TreeSet<>(Comparator.comparing(User::id))),
        ArrayList::new));
```

### 윈도우 유사(이웃 페어)

```java
List<Pair<Integer,Integer>> pairs =
    IntStream.range(0, nums.size()-1)
        .mapToObj(i -> Pair.of(nums.get(i), nums.get(i+1)))
        .toList();
```

### 안전한 trim/필터/기본값 체인

```java
String normalized = Stream.of(input)
    .filter(Objects::nonNull)
    .map(String::trim)
    .filter(s -> !s.isEmpty())
    .findFirst().orElse("N/A");
```

### 파일 트리에서 특정 크기 이상만 집계

```java
long bytes = Files.walk(root)
    .filter(Files::isRegularFile)
    .mapToLong(p -> {
        try { return Files.size(p); } catch (IOException e) { return 0L; }
    }).sum();
```

### 레코드 변환 + 검증

```java
List<User> valid = rows.stream()
    .map(Row::toUser)
    .filter(User::isValid)
    .toList();
```

### Nullable → 스트림 (JDK 9+)

```java
Stream.ofNullable(maybeNull).flatMap(Stream::of); // 0/1 요소
```

### 카테고리→상위 N per group

```java
Map<Category, List<Item>> topPerCat =
    items.stream().collect(Collectors.groupingBy(Item::category,
        Collectors.collectingAndThen(
            Collectors.toCollection(() -> new PriorityQueue<>(Comparator.comparingInt(Item::score).reversed())),
            pq -> pq.stream().limit(3).toList()
        )));
```

### 날짜 범위 스트림

```java
LocalDate start = LocalDate.now(), end = start.plusDays(30);
List<LocalDate> days = Stream.iterate(start, d -> !d.isAfter(end), d -> d.plusDays(1)).toList();
```

### 중복 키 안전 Map 만들기 (리스트 누적)

```java
Map<String, List<Item>> index =
    items.stream().collect(Collectors.groupingBy(Item::key, LinkedHashMap::new, Collectors.toList()));
```

### 컬렉션 플랫 평탄화 (성능형)

```java
List<String> flat = lists.stream().mapMulti((List<String> l, Consumer<String> sink) -> l.forEach(sink)).toList();
```

---

## 결론

- Stream은 **지연·선언적 파이프라인**으로 **가독성/안전성/최적화 여지**를 동시에 제공합니다.
- **중간 연산은 계획, 최종 연산이 실행**임을 기억하고, **불변·무상태**를 지키며 **박싱/정렬/중복제거/병렬** 비용을 감안하세요.
- 수집은 **Collectors**로, 복잡한 축약은 **downstream/teeing/collectingAndThen**으로 모델링하면 명료합니다.
- 병렬은 **측정 기반**으로만 채택하고, 외부 상태 변경을 피하세요.
- 필요 시 **커스텀 Collector/Spliterator**로 작업을 데이터에 맞게 특화하세요.

이 가이드를 템플릿 삼아 팀 코드 리뷰 체크리스트와 유닛 테스트(특히 경계/빈 컬렉션/병렬 케이스)를 함께 운영하면, 스트림 활용의 안정성과 성능을 동시에 끌어올릴 수 있습니다.
