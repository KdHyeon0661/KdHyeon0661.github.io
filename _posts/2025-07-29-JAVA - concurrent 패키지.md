---
layout: post
title: Java - concurrent 패키지
date: 2025-07-29 23:20:23 +0900
category: Java
---
# java.util.concurrent 패키지

## 0. 왜 `java.util.concurrent`(JUC) 인가?

- `synchronized` + `wait/notify`만으로는 **락 경합·스레드 폭주·자원 누수·깨지기 쉬운 코드**가 되기 쉽다.
- JUC는 **표준화된 스레드 풀/락/동시성 컬렉션/원자 연산/비동기 조합**을 제공:
  - 안정성(일관된 인터럽트 정책/시간 단위/정지 프로토콜)
  - 성능(CAS, work-stealing, 세분화 락, 저경합 알고리즘)
  - 가독성/유지보수성(관용 패턴과 고수준 추상화)

---

## 1. Executor Framework — 스레드 풀의 표준

### 1.1 핵심 개념

| 구성요소 | 역할 | 비고 |
|---|---|---|
| `Executor` | `execute(Runnable)` 단일 메서드 | 기초 실행기 |
| `ExecutorService` | 제출/종료/`Future` | `submit(Callable)` |
| `ScheduledExecutorService` | 지연·주기 작업 | `scheduleAtFixedRate/WithFixedDelay` |
| `Future` | 결과/취소 | `get/cancel/isDone` |
| `Callable<V>` | 값 반환·예외 전파 | `Runnable`의 상위 버전 |
| `ThreadPoolExecutor` | 스레드 풀 구현 | 모든 튜닝 포인트 보유 |

#### `ThreadPoolExecutor` 생성자(핵심 파라미터)
```java
ThreadPoolExecutor(
  int corePoolSize,
  int maximumPoolSize,
  long keepAliveTime, TimeUnit unit,
  BlockingQueue<Runnable> workQueue,
  ThreadFactory threadFactory,
  RejectedExecutionHandler handler
)
```
작업 흐름(요약):
1) `core` 미만이면 새 스레드 생성 → 큐가 비면 재사용  
2) 큐가 가득이면 `maximum`까지 스레드 증가  
3) 그래도 못 담으면 **거부 정책** 수행

### 1.2 큐/거부 정책/스레드 팩토리 — 선택 가이드

| 항목 | 옵션 | 사용 시점 |
|---|---|---|
| `workQueue` | `SynchronousQueue` | **짧고 빠른 작업**(핸드오프). 급상승 시 스레드 급증 위험 주의 |
|  | `LinkedBlockingQueue`(기본 무제한) | 무한 대기 가능·메모리 증가 위험. 보통 **용량 제한 권장** |
|  | `ArrayBlockingQueue`(고정 용량) | **백프레셔** 구현(거부/대기). 예측 가능한 지연 |
|  | `PriorityBlockingQueue` | 우선순위 작업 |
| 거부 정책 | `AbortPolicy` | 즉시 `RejectedExecutionException` |
|  | `CallerRunsPolicy` | **제출자 스레드가 실행** → 자연스런 속도 제한(백프레셔) |
|  | `DiscardPolicy`/`DiscardOldestPolicy` | 지양(정보 손실) |
| ThreadFactory | 이름/우선순위/데몬 | 로깅·모니터링/디버깅 용이 |

### 1.3 생명주기 관리

```java
executor.shutdown();                      // 신규 제출 금지
executor.awaitTermination(30, SECONDS);   // 남은 작업 대기
executor.shutdownNow();                   // 인터럽트로 즉시 종료 시도
```

> 규칙: **항상 종료**(try-finally 또는 shutdown hook), **인터럽트 정책 준수**(차단 메서드 호출 시 `InterruptedException` 처리 후 `Thread.currentThread().interrupt()` 재설정).

### 1.4 실전 레시피 — “안전한 고정 스레드 풀 + 백프레셔”

```java
var queue = new ArrayBlockingQueue<Runnable>(2000);
var factory = new ThreadFactory() {
  private final ThreadFactory df = Executors.defaultThreadFactory();
  private final java.util.concurrent.atomic.AtomicInteger id = new java.util.concurrent.atomic.AtomicInteger(1);
  @Override public Thread newThread(Runnable r) {
    Thread t = df.newThread(r);
    t.setName("api-worker-" + id.getAndIncrement());
    t.setDaemon(false);
    return t;
  }
};

var pool = new ThreadPoolExecutor(
  8, 8, 0L, TimeUnit.MILLISECONDS,
  queue, factory, new ThreadPoolExecutor.CallerRunsPolicy()
);
```

- **고정 크기**(CPU 바운드) + **유한 큐**(메모리 보호) + **CallerRunsPolicy**(자연스런 조절).

### 1.5 `CompletionService` — 먼저 끝난 것부터 받기

```java
var pool = Executors.newFixedThreadPool(8);
var ecs  = new ExecutorCompletionService<String>(pool);

for (var url : urls) ecs.submit(() -> fetch(url));
for (int i=0; i<urls.size(); i++) {
  Future<String> f = ecs.take();       // 완료되는 순서대로
  System.out.println(f.get());
}
pool.shutdown();
```

### 1.6 스케줄링 — `ScheduledExecutorService`

```java
var sch = Executors.newScheduledThreadPool(2);

// 고정 비율: 시작 시각을 격자로 유지(지연 누적 발생 가능)
sch.scheduleAtFixedRate(() -> heartbeat(), 0, 5, TimeUnit.SECONDS);

// 고정 지연: 이전 작업이 끝난 시점 기준으로 지연
sch.scheduleWithFixedDelay(() -> poll(), 0, 1, TimeUnit.MINUTES);
```

---

## 2. 동기화 도구 — Lock/Condition와 동기화 장치

### 2.1 `ReentrantLock` & `Condition`

- 장점: **공정성 옵션**, `tryLock(타임아웃)`, `lockInterruptibly()`, **여러 Condition** 지원
- `Condition.await()`는 **스퍼리어스 웨이크업** 가능 → 항상 **조건을 while로 재검사**
- 예: 고전적 **유한 버퍼**(생산자/소비자)

```java
class BoundedBuffer<T> {
  private final Object[] buf;
  private int head=0, tail=0, count=0;
  private final ReentrantLock lock = new ReentrantLock();
  private final Condition notEmpty = lock.newCondition();
  private final Condition notFull  = lock.newCondition();

  BoundedBuffer(int capacity) { this.buf = new Object[capacity]; }

  public void put(T x) throws InterruptedException {
    lock.lock();
    try {
      while (count == buf.length) notFull.await();
      buf[tail] = x; tail = (tail+1) % buf.length; count++;
      notEmpty.signal();
    } finally { lock.unlock(); }
  }

  @SuppressWarnings("unchecked")
  public T take() throws InterruptedException {
    lock.lock();
    try {
      while (count == 0) notEmpty.await();
      T x = (T) buf[head]; buf[head] = null; head = (head+1) % buf.length; count--;
      notFull.signal();
      return x;
    } finally { lock.unlock(); }
  }
}
```

### 2.2 읽기-쓰기 락 & `StampedLock`

| 도구 | 특징 | 사용 팁 |
|---|---|---|
| `ReentrantReadWriteLock` | 동시 읽기·단독 쓰기 | 읽기 비율이 매우 높고 쓰기 드묾 |
| `StampedLock` | **낙관적 읽기**(검증 실패 시 재시도) | 단순 캐시/좌표 읽기 등 단명 읽기 최적화 |

```java
class Point {
  private double x,y;
  private final java.util.concurrent.locks.StampedLock sl = new java.util.concurrent.locks.StampedLock();

  double distanceFromOrigin() {
    long stamp = sl.tryOptimisticRead();
    double cx = x, cy = y;
    if (!sl.validate(stamp)) {                 // 실패 시 비관적 읽기
      stamp = sl.readLock();
      try { cx = x; cy = y; }
      finally { sl.unlockRead(stamp); }
    }
    return Math.hypot(cx, cy);
  }

  void move(double dx, double dy) {
    long stamp = sl.writeLock();
    try { x += dx; y += dy; }
    finally { sl.unlockWrite(stamp); }
  }
}
```

### 2.3 기타 동기화 장치

| 클래스 | 용도 | 예제 아이디어 |
|---|---|---|
| `Semaphore(n)` | 동시 허용 수 제한 | 커넥션/슬롯 레이트 리미팅 |
| `CountDownLatch(n)` | n개 이벤트 후 진행 | 여러 서비스 준비 대기 |
| `CyclicBarrier(n)` | n개 스레드 **동시 지점** 동기화 | 단계별 병렬 알고리즘 |
| `Phaser` | Barrier 일반화(동적 등록/해제) | 가변 참여자 단계 동기화 |
| `Exchanger<T>` | 두 스레드의 버퍼 교환 | 더블 버퍼링 파이프라인 |

`Semaphore` 레이트 리미터(간단형):
```java
var sem = new Semaphore(20); // 동시 20개
void call() throws InterruptedException {
  sem.acquire();
  try { invokeRemote(); }
  finally { sem.release(); }
}
```

---

## 3. 동시성 컬렉션 — 안전한 컨테이너

### 3.1 `ConcurrentHashMap`(CHM)

- **null 키/값 불가**, **약한 일관성 반복자**(ConcurrentModificationException 미발생)
- Java 8+: **bin-tree(레드블랙트리)로 충돌 성능 개선**
- 동시 업데이트 APIs: `compute[IfAbsent]`, `merge`, `putIfAbsent`, `replace`
- 집계: `mappingCount()`, `forEach/reduce/search` (병렬 임계치 단위)

```java
var map = new java.util.concurrent.ConcurrentHashMap<String, Long>();
map.merge("key", 1L, Long::sum);
map.computeIfAbsent("user:42", k -> loadUserFromDB(k));
map.forEach(1, (k,v) -> System.out.println(k+"="+v)); // 1=병렬 임계치
```

> 빈번 카운터는 `LongAdder` 조합이 유리:
```java
var hits = new java.util.concurrent.ConcurrentHashMap<String, java.util.concurrent.atomic.LongAdder>();
hits.computeIfAbsent(path, k -> new java.util.concurrent.atomic.LongAdder()).increment();
```

### 3.2 `CopyOnWriteArrayList/Set`

- **읽기 매우 많고, 쓰기 적음** → 반복자는 **스냅샷** 기반, 락 경쟁 거의 없음
- 단점: 쓰기 시 **배열 복사 비용** 큼

### 3.3 `BlockingQueue` 계열

| 큐 | 특성 | 사용 |
|---|---|---|
| `ArrayBlockingQueue` | 고정 크기, 예측 가능한 대기 | 백프레셔 |
| `LinkedBlockingQueue` | 링크드, 보통 무제한(가급적 용량 지정) | 일반 생산자-소비자 |
| `SynchronousQueue` | 용량 0, 직접 건네주기 | 스레드 핸드오프, 짧은 작업 |
| `PriorityBlockingQueue` | 우선순위 | 스케줄링/정렬 필요 |
| `DelayQueue` | 지연 만료 후 추출 | 타임아웃·캐시 만료 |
| `LinkedTransferQueue` | `transfer`(소비자 대기 시 즉시 전달) | 높은 처리량 파이프 |

생산자-소비자(표준형):
```java
var q = new java.util.concurrent.LinkedBlockingQueue<String>(1000);

// 생산자
pool.submit(() -> {
  try { q.put(fetch()); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
});

// 소비자
pool.submit(() -> {
  try { handle(q.take()); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
});
```

---

## 4. 원자 변수 & Adder/Accumulator

| 클래스 | 용도 | 비고 |
|---|---|---|
| `AtomicInteger/Long/Boolean/Reference` | 단일 원자 갱신 | `compareAndSet`, `getAndUpdate` |
| `AtomicStampedReference` | **ABA 문제** 방지(스탬프) | 스택/큐 설계 시 |
| `LongAdder/DoubleAdder` | **고경합 카운터** | 셀 분산 → 합산 |
| `LongAccumulator` | 사용자 정의 결합 연산 | reduce 연산자 지정 |
| 필드 업데이터 | `AtomicIntegerFieldUpdater` 등 | `volatile` 필드 대상 |

```java
java.util.concurrent.atomic.LongAdder adder = new java.util.concurrent.atomic.LongAdder();
adder.increment(); adder.add(5);
long sum = adder.sum(); // 스냅샷 합계
```

---

## 5. Fork/Join — 분할정복과 work-stealing

- `ForkJoinPool`(공용 풀 포함): 각 워커가 자기 덱에서 팝, 부족하면 **다른 워커의 뒤에서 훔침**
- `RecursiveTask<V>`(값 반환) / `RecursiveAction`(void)
- 임계치(작업 크기) 튜닝이 핵심, **블로킹 호출 금지**(필요시 `ManagedBlocker`)

```java
class SumTask extends java.util.concurrent.RecursiveTask<Long> {
  static final int THRESHOLD = 1_000;
  final long[] a; final int s, e;
  SumTask(long[] a, int s, int e) { this.a=a; this{s=s;e=e;} }
  @Override protected Long compute() {
    int len = e - s;
    if (len <= THRESHOLD) {
      long sum=0; for (int i=s;i<e;i++) sum+=a[i]; return sum;
    }
    int m = s + len/2;
    var left  = new SumTask(a, s, m);
    var right = new SumTask(a, m, e);
    left.fork();
    long r = right.compute();
    long l = left.join();
    return l + r;
  }
}
```

> 블로킹 I/O가 섞인다면 별도 `ExecutorService`를 쓰거나 `ForkJoinPool.ManagedBlocker`로 워커 보전.

---

## 6. `CompletableFuture` — 비동기 조합

### 6.1 생성과 실행

```java
var pool = Executors.newFixedThreadPool(8);

CompletableFuture<Integer> f =
  CompletableFuture.supplyAsync(() -> compute(), pool); // 커스텀 풀

CompletableFuture<Void> g =
  CompletableFuture.runAsync(() -> sideEffect());       // 공용 풀
```

### 6.2 조합/변환

| 메서드 | 의미 | 비고 |
|---|---|---|
| `thenApply(fn)` | 값 → 값 | 동기/비동기 버전(`thenApplyAsync`) |
| `thenCompose(fn)` | 값 → `CompletionStage` 평탄화 | 비동기 의존 체인 |
| `thenCombine(f2, combiner)` | 두 스테이지 결합 | 병렬 조합 |
| `allOf/anyOf` | 다수 결합 | `allOf` 결과 수집은 개별 `join` 필요 |
| `handle/exceptionally/whenComplete` | 예외 다루기 | 예외 래핑은 `CompletionException` |

예:
```java
var price = CompletableFuture.supplyAsync(() -> fetchPrice(id), pool);
var stock = CompletableFuture.supplyAsync(() -> fetchStock(id), pool);

var dto = price.thenCombine(stock, (p, s) -> new ProductDto(p, s))
               .orTimeout(2, TimeUnit.SECONDS)        // JDK 9+
               .exceptionally(ex -> fallbackDto());
```

취소/타임아웃:
```java
f.orTimeout(1, TimeUnit.SECONDS);         // 9+
f.completeOnTimeout(defaultValue, 500, MILLISECONDS);
f.cancel(true);
```

> `thenApply`는 기본 **같은 스레드**에서 실행될 수 있음. **풀 격리/순환 의존**에 주의, 필요한 곳엔 `Async` + 전용 풀.

---

## 7. 랜덤/시간/유틸

- `ThreadLocalRandom` — 다중 스레드에서 `Random`보다 경합이 낮음
```java
int n = java.util.concurrent.ThreadLocalRandom.current().nextInt(100);
```
- `TimeUnit` — 가독성 높은 시간 변환, `sleep/await`에 일관 사용
- `ThreadFactory` — 스레드 이름/데몬/우선순위 정책 일원화

---

## 8. 자바 메모리 모델(JMM) — 가시성과 순서 보장 핵심

**happens-before** 주요 규칙(요점):
- **락 언락 → 같은 락의 이후 락 획득** 사이의 쓰기/읽기 가시성
- **`volatile`** 쓰기 → 이후 `volatile` 읽기
- **스레드 시작 이전(main thread의 준비)** → 스레드 시작 후
- **스레드 종료 이전(쓰레드 내부)** → 다른 스레드의 `join()` 이후
- **`ExecutorService` 제출 전 준비** → 워커에서 실행 시점(실행 경계로 인해 통상 안전)

가시성 보장이 필요하면: **락/`volatile`/원자 변수/스레드 경계** 중 하나를 이용.

---

## 9. 디버깅/테스트/운영 체크리스트

- 스레드 덤프(`jstack`/JFR)로 **데드락** 탐지: 락 소유/대기 관계 확인
- **인터럽트 정책** 준수: 차단 메서드에서 인터럽트 시 `interrupt()` 재설정
- **풀 메트릭** 모니터링: 작업 큐 길이, 액티브/풀 크기, 거부 횟수
- **백프레셔 설계**: 유한 큐 + CallerRunsPolicy/Rate Limit
- 마이크로벤치마크는 **JMH** 권장(자바 JIT 워밍업/교란 방지)
- 병렬 스트림/`ForkJoinPool` **공용 풀 오염** 주의(블로킹 섞지 않기)

---

## 10. 선택 가이드 — 무엇을 언제 쓰나

| 문제 | 권장 도구 | 이유 |
|---|---|---|
| 웹/서비스 처리 스레드 풀 | `ThreadPoolExecutor` + `ArrayBlockingQueue` + `CallerRunsPolicy` | 백프레셔+안정성 |
| 주기적 작업 | `ScheduledExecutorService` | 고정 비율 vs 고정 지연 |
| 읽기 위주 캐시 | `ReentrantReadWriteLock` 또는 `StampedLock` | 동시 읽기 최적화 |
| 파이프라인 큐 | `LinkedBlockingQueue`/`SynchronousQueue` | 생산자-소비자 |
| 고경합 카운터 | `LongAdder` | AtomicLong보다 확장성 |
| 빠른 키-값 동시 접근 | `ConcurrentHashMap` + `compute/merge` | 락 경합 저감 |
| 분할정복 계산 | `ForkJoinPool` + `RecursiveTask` | work-stealing |
| 비동기 조합/타임아웃 | `CompletableFuture` | 조합·예외·시간 제어 |

---

## 11. 종합 예제 — “I/O 바운드 호출 + 캐시 + 비동기 조합 + 백프레셔”

```java
import java.util.concurrent.*;
import java.util.concurrent.atomic.*;
import java.util.*;

class Service {
  private final ConcurrentHashMap<String, CompletableFuture<String>> cache = new ConcurrentHashMap<>();
  private final ExecutorService ioPool;

  Service() {
    var q = new ArrayBlockingQueue<Runnable>(500);
    ioPool = new ThreadPoolExecutor(16, 16, 0, TimeUnit.MILLISECONDS, q,
      r -> { Thread t = new Thread(r, "io-" + UUID.randomUUID()); t.setDaemon(true); return t; },
      new ThreadPoolExecutor.CallerRunsPolicy());
  }

  // 캐시-스탬피드 방지: 키별 단일 비동기 계산 공유
  public CompletableFuture<String> getData(String key) {
    return cache.computeIfAbsent(key, k ->
      CompletableFuture.supplyAsync(() -> fetchSlow(k), ioPool)
        .orTimeout(2, TimeUnit.SECONDS)
        .whenComplete((v, ex) -> { if (ex != null) cache.remove(k); }) // 실패 시 캐시 제거
    );
  }

  private String fetchSlow(String key) {
    try { TimeUnit.MILLISECONDS.sleep(200); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
    return "value:" + key;
  }

  void shutdown() { ioPool.shutdown(); }
}

public class App {
  public static void main(String[] args) {
    var svc = new Service();
    var all = new ArrayList<CompletableFuture<String>>();
    for (int i=0;i<1000;i++) {
      String key = "k" + (i % 100); // 중복 키 다수(스탬피드 방지 확인)
      all.add(svc.getData(key).exceptionally(ex -> "fallback"));
    }
    var result = CompletableFuture.allOf(all.toArray(new CompletableFuture[0]))
                  .thenApply(v -> all.stream().map(CompletableFuture::join).toList())
                  .join();
    System.out.println("size=" + result.size());
    svc.shutdown();
  }
}
```

포인트:
- **키별 1회 계산 공유**(캐시-스탬피드 방지)  
- **타임아웃·예외 처리**(실패 시 캐시 비우기)  
- **유한 큐 + CallerRunsPolicy**로 백프레셔

---

## 12. 흔한 함정과 방어 전략

- 무제한 큐 + 큰 `maximumPoolSize` → **메모리 폭증**: 항상 **유한 큐** 또는 `SynchronousQueue`로 의도 명시
- `Future.get()` 남용 → **스레드 블로킹**: 가능하면 **조합 API**(`thenCompose/thenCombine`) 사용
- `CopyOnWriteArrayList`에 빈번한 쓰기 → **큰 복사 비용**: 읽기 위주에만
- `StampedLock` 낙관 읽기 남발 → **검증 실패 루프**: 충돌 많은 구간은 `readLock`으로 전환
- `ForkJoinPool`에 블로킹 I/O → **스레드 고갈**: 별도 풀 또는 `ManagedBlocker`
- 인터럽트 무시 → **정지 불가능 시스템**: 인터럽트 시 **즉시 정리/플래그 설정/재설정**

---

## 13. 빠른 레퍼런스 표

### 13.1 스케줄러 차이
| 메서드 | 특징 | 사용 |
|---|---|---|
| `scheduleAtFixedRate` | 시작 간격 고정(드리프트 가능) | 모니터링 비트 |
| `scheduleWithFixedDelay` | 종료→다음 시작 지연 고정 | 폴링/청소 작업 |

### 13.2 Latch/Barrier/Phaser
| 도구 | 일회성 | 재사용 | 동적 파티 | 비고 |
|---|---|---|---|---|
| `CountDownLatch` | 예 | 아니오 | 아니오 | 0되면 영원히 열린다 |
| `CyclicBarrier` | 아니오 | 예 | 아니오 | 모두 모이면 동시 출발 |
| `Phaser` | 아니오 | 예 | **예** | 가장 유연 |

---

## 14. 결론

- **Executor(풀)로 스레드를 직접 관리하지 말라** — 큐·거부·종료를 표준으로.  
- 락 선택은 **읽기/쓰기 패턴**과 **경합 정도**로 결정, 조건 대기는 **while 재검사**.  
- 동시성 컬렉션/원자 변수로 **락을 줄이고** 가시성/원자성 보장.  
- 계산은 `ForkJoin`/조합은 `CompletableFuture`로 **구조화**하라.  
- **백프레셔/인터럽트/종료**는 시스템 안정성의 핵심 정책이다.

이 문서의 예제들을 기반으로 **라이브러리화(스레드 팩토리/풀 빌더/예외 헬퍼/비동기 래퍼)** 하면, 대부분의 동시성 요구를 안정적으로 표준화할 수 있다.