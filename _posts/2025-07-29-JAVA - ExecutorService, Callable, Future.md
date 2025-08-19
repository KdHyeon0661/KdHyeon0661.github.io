---
layout: post
title: Java - ExecutorService, Callable, Future
date: 2025-07-29 22:20:23 +0900
category: Java
---
# ExecutorService, Callable, Future — 자세하게 정리

Java에서 병렬/비동기 작업 관리를 할 때 `ExecutorService`, `Callable`, `Future`는 가장 핵심이 되는 API입니다.  
아래에 개념, 주요 동작, 사용 예제, 모범 사례와 주의사항까지 **실무 관점에서 자세하게** 정리했습니다.

---

## 개요 (한줄 요약)
- **ExecutorService**: 스레드 풀을 관리하고 `Runnable`/`Callable` 작업을 제출해 실행하도록 하는 고수준 추상화 (스레드 생성/관리 책임을 대신 담당).  
- **Callable<V>**: 결과를 반환하거나 체크 예외를 던질 수 있는 작업 단위(제네릭 반환 타입).  
- **Future<V>**: 비동기 작업의 결과를 나타내는 핸들(결과 가져오기, 취소, 완료 여부 확인 등).

---

## 1. ExecutorService

### 1.1 무엇인가?
- `ExecutorService`는 `Executor`의 확장으로, 작업 제출, 종료(shutdown) 등을 지원합니다.
- 스레드 생성·재사용, 큐잉, 정책(거부 처리) 등을 담당하므로 **직접 `new Thread(...)`를 하는 것보다 안전**하고 효율적입니다.

### 1.2 생성(대표 팩토리)
(간단한 팩토리 메서드 — 주의: unbounded/특성 이해 필요)

```java
ExecutorService fixed = Executors.newFixedThreadPool(10);
ExecutorService cached = Executors.newCachedThreadPool();
ExecutorService single = Executors.newSingleThreadExecutor();
ScheduledExecutorService scheduled = Executors.newScheduledThreadPool(4);
```

> **주의**: `Executors`의 일부 팩토리(예: `newCachedThreadPool`, `newFixedThreadPool`)는 내부적으로 **무제한 작업 큐** 또는 **무제한 스레드 생성** 정책을 가질 수 있어, 실무에서는 `ThreadPoolExecutor`를 직접 구성해 **bounded queue + 적절한 RejectedExecutionHandler**를 설정하는 것이 안전합니다.

### 1.3 ThreadPoolExecutor (주요 파라미터)
- `corePoolSize`, `maximumPoolSize`, `keepAliveTime`, `workQueue`, `ThreadFactory`, `RejectedExecutionHandler`

```java
ThreadPoolExecutor pool = new ThreadPoolExecutor(
    4,                       // core
    20,                      // max
    60L, TimeUnit.SECONDS,   // keepAlive
    new ArrayBlockingQueue<>(100), // bounded queue
    Executors.defaultThreadFactory(),
    new ThreadPoolExecutor.AbortPolicy() // 거부 정책
);
```

---

## 2. Callable vs Runnable

### 2.1 Runnable
- `void run()` — 반환값 없음, 체크 예외 던질 수 없음(던지면 런타임 예외로 포장됨).
- `executor.execute(runnable)` 또는 `executor.submit(runnable)` 사용.

### 2.2 Callable<V>
- `V call()` — 제네릭 타입 `V` 반환, 체크 예외 던질 수 있음.
- `executor.submit(callable)` 사용하면 `Future<V>` 반환.

```java
Callable<Integer> task = () -> {
    // 작업 수행
    return 42;
};
Future<Integer> f = executor.submit(task);
```

---

## 3. Future — 주요 메서드와 동작

| 메서드 | 설명 |
|--------|------|
| `V get()` | 작업 완료 시 결과 반환(블록). |
| `V get(long timeout, TimeUnit unit)` | 타임아웃 후 `TimeoutException` 발생 가능. |
| `boolean cancel(boolean mayInterruptIfRunning)` | 작업 취소 요청. 이미 실행 중이면 인터럽트 시도(매개변수에 따라). |
| `boolean isCancelled()` | 취소되었는지. |
| `boolean isDone()` | 완료(성공/예외/취소 포함)되었는지. |

### 예외 처리
- `get()`은 `InterruptedException`과 `ExecutionException`을 던짐.  
- `ExecutionException`의 `getCause()`에서 실제 예외 원인을 확인.

```java
try {
    Integer result = future.get(5, TimeUnit.SECONDS);
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
} catch (ExecutionException e) {
    Throwable cause = e.getCause();
    // 실제 예외 처리
} catch (TimeoutException e) {
    // 타임아웃 처리
}
```

### 취소와 인터럽트
- `cancel(true)` → 가능하면 스레드를 인터럽트하여 중단을 요청.  
- 취소는 협력적: 작업 내부에서 `Thread.currentThread().isInterrupted()` 또는 `InterruptedException`을 체크/처리해야 안전히 종료 가능.

---

## 4. 제출 방식: execute vs submit
- `execute(Runnable)` — 반환값 없음; 예외는 `UncaughtExceptionHandler`로 전달됨.  
- `submit(...)` — `Future`를 반환해 결과 확인/예외 캡처/취소 가능. (Runnable 제출 시 Future.get()은 `null` 반환)

권장: 결과를 받아야 하면 `submit(Callable)` 사용.

---

## 5. 모아 제출: invokeAll / invokeAny

### 5.1 `invokeAll(Collection<? extends Callable<T>>)`

- 모든 태스크를 제출하고 **모두 완료**될 때까지 대기 후 `List<Future<T>>` 반환.
- 각 Future는 `get()`으로 결과 또는 예외를 확인.

```java
List<Callable<Integer>> tasks = Arrays.asList(() -> 1, () -> 2);
List<Future<Integer>> futures = executor.invokeAll(tasks);
for (Future<Integer> f : futures) {
    System.out.println(f.get()); // 블록 없이 바로 결과 가능(이미 완료)
}
```

### 5.2 `invokeAny(Collection<? extends Callable<T>>)`

- 작업들 중 **하나가 성공적으로 완료된 결과**(가장 먼저 성공한 것)를 반환하고, 나머지 작업을 취소하려 시도함.
- 성공한 결과 반환, 실패 시 예외 던짐.

---

## 6. ScheduledExecutorService (예약/주기 작업)
- 지연 실행 및 주기적 실행을 위한 스케줄러.

```java
ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(2);

// 1) 지연 실행
ScheduledFuture<String> sf = scheduler.schedule(() -> "done", 5, TimeUnit.SECONDS);

// 2) 주기 실행 (fixed rate)
scheduler.scheduleAtFixedRate(() -> {
    // 작업
}, 0, 10, TimeUnit.SECONDS);

// 3) 주기 실행 (fixed delay)
scheduler.scheduleWithFixedDelay(() -> {
    // 작업
}, 0, 10, TimeUnit.SECONDS);
```

> 주의: `scheduleAtFixedRate`는 작업이 오래 걸리면 다음 실행이 바로 시작될 수 있으므로 작업 시간이 불확실하면 `scheduleWithFixedDelay` 권장.

---

## 7. graceful shutdown (중요!)
항상 `ExecutorService`는 종료해야 합니다. 안전한 패턴:

```java
executor.shutdown(); // 더 이상 새로운 작업 받지 않음
try {
    if (!executor.awaitTermination(60, TimeUnit.SECONDS)) {
        executor.shutdownNow(); // 즉시 취소 시도
        if (!executor.awaitTermination(60, TimeUnit.SECONDS)) {
            System.err.println("풀 종료 실패");
        }
    }
} catch (InterruptedException ie) {
    executor.shutdownNow();
    Thread.currentThread().interrupt();
}
```

- `shutdown()` → 기존 작업은 완료, 신규 작업 거부.
- `shutdownNow()` → 대기 중인 작업 목록 반환, 실행 중인 스레드에 인터럽트 보냄.

---

## 8. FutureTask & 직접 사용
- `FutureTask<V>`는 `RunnableFuture<V>` 구현체로, 직접 생성해 스레드나 executor로 실행 가능.  
- 디버깅/조합용으로 유용.

```java
FutureTask<Integer> task = new FutureTask<>(() -> 123);
new Thread(task).start();
Integer v = task.get();
```

---

## 9. 예제: Callable 제출과 결과 처리

```java
ExecutorService ex = Executors.newFixedThreadPool(4);
try {
    Callable<Integer> job = () -> {
        // 예외 던질 수 있음
        return 1 + 1;
    };
    Future<Integer> f = ex.submit(job);

    // 비차단 작업 예: 다른 일 수행

    // 결과 가져오기 (블록)
    Integer result = f.get(); // InterruptedException, ExecutionException 처리 필요
    System.out.println("Result: " + result);
} finally {
    ex.shutdown();
}
```

---

## 10. 예외 & 실패 처리 패턴
- `ExecutionException` → `getCause()`로 실제 원인 확인.
- 작업 내부에서 체크 예외가 발생하면 `ExecutionException`에 래핑됨.
- 주기 작업에서 예외가 발생하면 해당 태스크는 종료될 수 있으므로 **주기 작업 내부에서 예외를 잡고 처리**하거나 스케줄러에서 재등록 패턴 필요.

---

## 11. ThreadFactory, 이름 지정, 모니터링
- `ThreadFactory`를 제공하면 스레드 이름, 데몬 여부, 우선순위 설정 및 로깅이 쉬워짐.

```java
ThreadFactory tf = r -> {
    Thread t = new Thread(r);
    t.setName("worker-" + counter.incrementAndGet());
    t.setDaemon(false);
    return t;
};
ExecutorService pool = Executors.newFixedThreadPool(4, tf);
```

---

## 12. 풀 사이즈 결정(간단 가이드)
- **CPU-bound** 작업: `Nthreads ≈ numCores` (일반적으로 `Runtime.getRuntime().availableProcessors()` 사용)
- **I/O-bound** 작업: `Nthreads > numCores`, 대기 시간 고려(예: `numCores * (1 + waitTime/computeTime)`)
- 실무: 부하/성능 테스트로 최적 설정 권장.

---

## 13. 성능/안정성 권장사항 (요약)
- `Executors`의 기본 팩토리를 쓸 수 있지만, **운영 환경에선 ThreadPoolExecutor를 직접 구성**(bounded queue, 적절한 RejectedExecutionHandler) 권장.
- 작업은 가능한 **무상태(stateless)**로 만들고 공유 자원 동기화 주의.
- `Future.get()`을 오래 블록시키지 말고 타임아웃 사용 고려.
- **예외 발생 시 로깅**을 빠짐없이 수행하라 (누락된 예외는 문제 추적 어렵다).
- 비동기 파이프라인(복합 작업)엔 `CompletableFuture`를 검토하라 — 더 강력한 조합/에러 처리 API 제공.

---

## 14. CompletableFuture 한 줄 (참고)
- Java 8부터 `CompletableFuture`는 `Executor`와 결합해 비동기 파이프라인(thenApply, thenCompose 등)을 쉽게 구성할 수 있다.
```java
CompletableFuture.supplyAsync(() -> compute(), executor)
                 .thenApply(result -> process(result))
                 .exceptionally(ex -> handle(ex));
```

---

## 결론
- `ExecutorService` + `Callable` + `Future`는 자바에서 **비동기 작업의 표준 패턴**입니다.  
- 올바른 스레드 풀 구성, 예외/취소/종료 전략, 작업 설계(인터럽트 가능, 상태 최소화)를 지키면 안정적이고 확장 가능한 동시성 시스템을 만들 수 있습니다.  
- 복잡한 비동기 조합이나 파이프라인이 필요하면 `CompletableFuture`를 고려하세요.
