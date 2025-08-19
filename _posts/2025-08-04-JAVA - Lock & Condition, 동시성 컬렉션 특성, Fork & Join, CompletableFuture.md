---
layout: post
title: Java - Lock & Condition, 동시성 컬렉션 특성, Fork & Join, CompletableFuture
date: 2025-08-04 14:20:23 +0900
category: Java
---
# Lock & Condition 내부 동작 원리, 동시성 컬렉션 특성, Fork/Join 심화, CompletableFuture 사용법 — 자세 정리

아래 내용은 Java 동시성의 고급 주제들을 한 번에 정리한 문서입니다. 각 주제마다 **원리 → 코드 예제 → 실무 팁/권장사항** 순으로 설명합니다. 코드 블록은 ```로 표기했습니다.

---

## 1. `Lock`과 `Condition`의 내부 동작 원리

### 1.1 큰 그림
- `java.util.concurrent.locks.Lock` 인터페이스는 `synchronized`보다 더 유연한 락(예: 타임아웃 획득, 공정성, interruptible 획득 등)을 제공합니다.
- 대부분 구현체(`ReentrantLock`)는 **AbstractQueuedSynchronizer(AQS)** 를 내부 기반으로 합니다.
- `Condition`(예: `lock.newCondition()`)은 `wait/notify`의 대체물로, AQS의 `ConditionObject`가 조건 대기/신호를 관리합니다.

---

### 1.2 AbstractQueuedSynchronizer(AQS) 핵심 원리 (요약)
- **상태(state)**: 정수(`int state`)로 락 점유 상태(예: 재진입 횟수)를 관리. CAS로 변경.
- **동기화 큐(sync queue)**: 락을 획득하지 못한 스레드는 AQS의 FIFO 대기 큐(노드들)에 enq 된다.
- **모드**: Exclusive(배타적 락) vs Shared(공유 락) 모드 지원.
- **park/unpark**: 대기 스레드를 `LockSupport.park()`로 차단하고, 다른 스레드가 `unpark()`로 깨움.
- 즉, 락 획득 시 CAS로 `state`를 변경 → 실패하면 큐에 노드 추가 → blocking → 락이 해제되면 큐에서 다음 노드 `unpark`.

> 참고: `ReentrantLock`은 `AQS`의 `state`를 재진입 카운터로 사용. 소유자 스레드가 다시 lock()하면 state++.

---

### 1.3 Condition(ConditionObject)의 동작
- `Condition`은 **독립된 wait set(조건 대기자 리스트)** 를 유지한다(각 Condition마다 별도 큐).
- `await()` 호출:
  1. 현재 스레드는 반드시 락을 소유해야 함(아니면 `IllegalMonitorStateException`).
  2. 현재 스레드의 노드를 Condition 큐에 추가.
  3. 락을 해제하고(`release`), 스레드를 차단(park).
- `signal()` 호출:
  1. Condition 큐에서 하나의 노드를 꺼내 **동기화 큐(sync queue)** 로 옮긴다.
  2. 옮겨진 노드는 락을 다시 얻기 위해 sync queue에서 경쟁한다.
- 중요한 포인트: `signal()` 시점에 바로 실행되지 않음 — signal을 호출한 스레드가 synchronized / lock 영역을 벗어나고 락을 해제한 뒤에야 깨어난 스레드가 락을 획득해 `await()` 다음 라인부터 실행한다.

---

### 1.4 예제: ReentrantLock + Condition (생산자/소비자)

```java
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

public class BoundedBuffer<T> {
    private final T[] items;
    private int putPtr, takePtr, count;
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition notEmpty = lock.newCondition();
    private final Condition notFull = lock.newCondition();

    @SuppressWarnings("unchecked")
    public BoundedBuffer(int capacity) {
        items = (T[]) new Object[capacity];
    }

    public void put(T x) throws InterruptedException {
        lock.lock();
        try {
            while (count == items.length) {
                notFull.await();          // 대기: 락 해제
            }
            items[putPtr] = x;
            if (++putPtr == items.length) putPtr = 0;
            count++;
            notEmpty.signal();            // 소비자 깨우기
        } finally {
            lock.unlock();
        }
    }

    public T take() throws InterruptedException {
        lock.lock();
        try {
            while (count == 0) {
                notEmpty.await();
            }
            T x = items[takePtr];
            if (++takePtr == items.length) takePtr = 0;
            count--;
            notFull.signal();
            return x;
        } finally {
            lock.unlock();
        }
    }
}
```

---

### 1.5 실무 팁
- `Condition` 사용 시 **while 루프**로 상태를 재검사하라(스퍼리어스 웨이크업&다중 스레드 경쟁 대비).
- 여러 조건이 있으면 **별도 Condition**을 만들어 락 범위를 세밀하게 제어하라(불필요한 깨움 방지).
- `ReentrantLock(true)` 처럼 공정성(fair) 옵션을 지정하면 락 대기 큐에서 FIFO 정책을 따르지만 성능 저하 가능.
- 가능하면 `BlockingQueue`, `Semaphore` 등 고수준 동시성 도구를 사용해 `Condition` 사용을 줄이는 것이 안전.

---

## 2. 동시성 컬렉션 종류별 특성 및 내부 전략

아래는 대표적 컬렉션들의 **특성 · 내부 전략 · 사용 시나리오**입니다.

### 2.1 `ConcurrentHashMap`
- **특성**: 스레드-세이프한 Map, 고성능 동시 접근 지원.
- **Java 8+ 내부**: 기본적으로 **버킷에 대한 CAS 기반 업데이트** + 필요시 해당 버킷을 `synchronized`로 잠그는 전략. 해시 충돌이 많은 경우 bin을 `TreeNode`(Red-Black Tree)로 변환.
- **읽기**: 락 없이도 거의 안전하게 읽을 수 있음(volatile/읽기 가시성 보장).
- **이터레이션**: *weakly consistent* — 반복 중에도 수정 허용, 수정 반영 여부는 보장되지 않지만 `ConcurrentModificationException` 없음.
- **사용 경우**: 높은 동시 읽기/쓰기, 캐시, 통계 집계.

### 2.2 `CopyOnWriteArrayList` / `CopyOnWriteArraySet`
- **특성**: 쓰기 시 배열을 **복사**(불변 스냅샷 생성). 읽기는 락 없이 빠름.
- **장단점**: 읽기 빈도가 매우 높고 쓰기가 드문 경우에 적합. 쓰기 비용(메모리/GC) 큼.
- **이터레이션**: 안전한 불변 스냅샷을 리턴하므로 iterator에서 변경해도 안전.

### 2.3 `BlockingQueue` 계열
- `ArrayBlockingQueue` : 고정 크기, 내부에서 하나의 `ReentrantLock`과 두 `Condition`(notEmpty/notFull) 사용 (공정성 옵션 있음).
- `LinkedBlockingQueue` : 링크드 노드 기반, 두 락(takeLock/putLock)으로 높은 동시성(put/take 병렬) 제공.
- `PriorityBlockingQueue` : 우선순위 큐(비교 기준), 내부 동작에 따라 FIFO 보장 안 됨.
- `SynchronousQueue` : 버퍼가 없는 큐 — 요소를 넣는 쓰레드는 즉시 다른 쓰레드가 받지 않으면 블록(스레드 간 직접 핸드오프에 유용).
- `DelayQueue` : 지연 큐, 스케줄링용.

### 2.4 `ConcurrentSkipListMap` / `ConcurrentSkipListSet`
- **특성**: 정렬된 Map/Set을 스레드-세이프하게 제공. 내부는 Skip List(락 프리-ish 구조).
- **사용**: 정렬된 동시성 맵이 필요할 때.

### 2.5 `ConcurrentLinkedQueue` / `ConcurrentLinkedDeque`
- **특성**: 비차단(lock-free) 링크드 큐, CAS 기반으로 높은 확장성.
- **사용**: 낮은 레이턴시의 비차단 큐가 필요할 때.

---

### 2.6 선택 가이드 (요약)
- **생산자/소비자**: `LinkedBlockingQueue`, `ArrayBlockingQueue`, `SynchronousQueue` 등.
- **높은 읽기, 거의 쓰기 없음**: `CopyOnWriteArrayList`.
- **높은 동시성 해시맵**: `ConcurrentHashMap`.
- **정렬 필요**: `ConcurrentSkipListMap`.
- **낮은 지연, CAS 기반**: `ConcurrentLinkedQueue`.

---

## 3. Fork/Join 프레임워크 심화

### 3.1 핵심 아이디어
- **분할 정복(작업 분해)**: 큰 작업을 작은 작업으로 쪼갠 후 병렬로 실행하고 결과를 병합.
- **Work-stealing**: 각 스레드는 자신의 deque(작업 데크)를 사용. 작업이 비게 되면 다른 스레드의 데크 끝에서 작업을 훔쳐(steal) 수행 → 로드 밸런싱 향상.
- 제공 클래스: `ForkJoinPool`, `ForkJoinTask`, `RecursiveTask<V>`, `RecursiveAction`.

---

### 3.2 내부 동작 포인트
- 작업은 **LIFO**로 push/pop(같은 워커가 빠르게 연속 작업 수행) → 지역성↑.
- 다른 워커는 **FIFO**쪽(데크 앞)에서 steal → 공정성/부하 균형.
- steal은 부담이 있으므로 적당한 크기의 작업으로 쪼갤 것(과도한 분할은 오버헤드).
- `commonPool()` 기본 풀은 가용 프로세서 수에 기반하여 크기 결정(병렬스트림과 공유될 수 있음).

---

### 3.3 예제: RecursiveTask 합계 (심화)

```java
import java.util.concurrent.RecursiveTask;
import java.util.concurrent.ForkJoinPool;

public class SumTask extends RecursiveTask<Long> {
    private static final int THRESHOLD = 1_000; // 임계값 조정 중요
    private final long[] array;
    private final int start, end;

    public SumTask(long[] array, int start, int end) {
        this.array = array; this.start = start; this.end = end;
    }

    @Override
    protected Long compute() {
        int length = end - start;
        if (length <= THRESHOLD) {
            long sum = 0;
            for (int i = start; i < end; i++) sum += array[i];
            return sum;
        } else {
            int mid = start + length / 2;
            SumTask left = new SumTask(array, start, mid);
            SumTask right = new SumTask(array, mid, end);
            left.fork();              // 왼쪽 하위 작업을 비동기 제출
            long rightResult = right.compute(); // 현재 스레드에서 오른쪽 처리
            long leftResult = left.join();     // 왼쪽 결과 합류
            return leftResult + rightResult;
        }
    }

    public static void main(String[] args) {
        ForkJoinPool pool = new ForkJoinPool();
        long[] arr = new long[10_000_000];
        // ... 초기화
        long total = pool.invoke(new SumTask(arr, 0, arr.length));
        System.out.println(total);
        pool.shutdown();
    }
}
```

---

### 3.4 성능/튜닝 팁
- **임계값(THRESHOLD)**: 너무 작으면 오버헤드, 너무 크면 병렬성 미활용. 실험으로 최적값 결정.
- **fork() vs invokeAll()**: `fork()`+`join()` 패턴이 유연. `invokeAll()`은 편리.
- **commonPool 주의**: 애플리케이션이 다른 병렬 도구(Streams)와 commonPool를 공유하면 풀 고갈 위험 → 별도 `ForkJoinPool` 사용 고려.
- **블로킹 작업 주의**: ForkJoinPool worker가 블록되면 스레드 증설(ManagedBlocker 사용) 또는 별도 executor 권장.

---

## 4. `CompletableFuture` 사용법 (심화)

### 4.1 핵심 개념
- `CompletableFuture<T>`는 비동기 파이프라인(조합과 에러 처리)을 선언형으로 작성할 수 있게 함.
- `supplyAsync`, `runAsync` 로 비동기 시작. 이후 `thenApply`, `thenCompose`, `thenCombine` 등으로 조합.
- `CompletableFuture`는 `CompletionStage` 인터페이스를 구현.

---

### 4.2 주요 연산자 요약
- `supplyAsync(Supplier<T>)` : 비동기 결과 공급.
- `thenApply(Function<T,R>)` : 완료 결과 동기 변환.
- `thenApplyAsync(...)` : 비동기 변환(별도 executor 사용 가능).
- `thenCompose(futureMaker)` : 비동기 연속(FlatMap) — `CompletableFuture<CF<U>>` → `CF<U>`.
- `thenCombine(other, BiFunction)` : 두 독립 결과 결합.
- `allOf(...)`, `anyOf(...)` : 다수의 Future 조합.
- `exceptionally`, `handle`, `whenComplete` : 예외 처리/후처리.

---

### 4.3 기본 예제

```java
import java.util.concurrent.*;

public class CFExample {
    public static void main(String[] args) throws Exception {
        ExecutorService ex = Executors.newFixedThreadPool(4);

        CompletableFuture<Integer> f1 = CompletableFuture.supplyAsync(() -> {
            // CPU/IO 작업
            return 2;
        }, ex);

        CompletableFuture<Integer> f2 = CompletableFuture.supplyAsync(() -> 3, ex);

        // thenCombine: f1, f2 완료 후 합산
        CompletableFuture<Integer> sum = f1.thenCombine(f2, Integer::sum);

        System.out.println("Sum: " + sum.get()); // 블록(예시)

        ex.shutdown();
    }
}
```

---

### 4.4 thenCompose vs thenApply (차이)

- `thenApply` : `T -> U` 변환 (동기/비동기 가능)
- `thenCompose` : `T -> CompletableFuture<U>`를 반환하는 경우 결과를 **평탄화**(flatten)하여 `CompletableFuture<U>`를 얻음 — 연속 비동기 호출 연결에 사용

```java
CompletableFuture<String> userFuture = findUserAsync(id); // CF<User>
CompletableFuture<Account> accountFuture = userFuture.thenCompose(user -> findAccountAsync(user));
```

---

### 4.5 예외 처리

```java
CompletableFuture<Integer> f = CompletableFuture.supplyAsync(() -> {
    if (true) throw new RuntimeException("error");
    return 1;
});

CompletableFuture<Integer> handled = f.handle((res, ex) -> {
    if (ex != null) {
        // 예외 처리 로직
        return -1;
    } else return res;
});

// or
CompletableFuture<Integer> fallback = f.exceptionally(ex -> -1);
```

- `handle`은 결과/예외 둘 다 처리 가능, `exceptionally`는 예외 전용.

---

### 4.6 조합 예: allOf / anyOf

```java
CompletableFuture<Void> all = CompletableFuture.allOf(f1, f2, f3);
all.thenRun(() -> System.out.println("모두 완료"));

// 개별 결과 수집
CompletableFuture<List<Integer>> allResults = all.thenApply(v ->
    Stream.of(f1, f2, f3)
          .map(CompletableFuture::join) // 이미 완료되어야 함
          .collect(Collectors.toList()));
```

---

### 4.7 타임아웃, 취소, 완료 강제
- Java 9+에는 `orTimeout(long, TimeUnit)`와 `completeOnTimeout(value, time, unit)`가 있음.
- 직접 구현 시 `completeOnTimeout`과 `ScheduledExecutorService`로 타임아웃 처리 가능.

```java
CompletableFuture<String> cf = CompletableFuture.supplyAsync(() -> {
    // 작업
});

// Java 9+
cf = cf.completeOnTimeout("timeout-value", 2, TimeUnit.SECONDS);
```

취소:

```java
cf.cancel(true); // 취소 및 인터럽트 신호 전파(작업이 인터럽트 가능할 때)
```

---

### 4.8 실무 권장사항
- **블로킹 작업**(I/O 등)은 `CompletableFuture`의 기본 commonPool에서 수행하지 말고, 별도 **커스텀 Executor**를 전달하라. (commonPool 워커 고갈 방지)
- `get()`/`join()`은 블로킹이므로 비동기 파이프라인 내부에서 호출하면 deadlock 위험. 가능하면 비동기 API(thenApply/thenCompose 등)로 연결하라.
- 예외 처리를 파이프라인 상단에 설계해 전체 흐름의 실패를 적절히 관리하라.
- `thenCompose`로 비동기 연속 작업을 연결하면 가독성/성능 측면에서 자연스럽다.
- `allOf` 사용 시 개별 실패 원인(`ExecutionException`) 추적을 위해 `handle`로 래핑하거나 `join()`의 예외 캡처 방식을 고려하라.

---

## 5. 종합적인 권장 관행 (요약)
1. 가능한 고수준 API를 사용하라: `BlockingQueue`, `ConcurrentHashMap`, `ExecutorService`, `CompletableFuture`, `ForkJoinPool`.
2. 락이 필요한 경우 `ReentrantLock` + `Condition`을 이해하고 사용하되, 잘 설계된 동시성 컬렉션으로 대체하라.
3. **불변(immutable)** 객체와 무상태(stateless) 작업을 선호하라 — 동시성 버그 수를 줄여준다.
4. 블로킹 작업은 별도의 쓰레드 풀으로 분리하고, 공용 풀(commonPool) 자원 고갈을 피하라.
5. 병렬성/동시성 코드는 반드시 테스트(부하/경쟁 상황)로 검증하고, 데드락·race condition을 의심하면 코드를 재검토하라.
6. 로그와 모니터링을 통해 스레드 풀 상태(활성 스레드 수, 큐 길이 등)를 관찰하라.
