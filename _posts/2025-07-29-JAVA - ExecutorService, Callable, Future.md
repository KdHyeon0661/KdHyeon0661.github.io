---
layout: post
title: Java - ExecutorService, Callable, Future
date: 2025-07-29 22:20:23 +0900
category: Java
---
# ExecutorService, Callable, Future

## 0. 한눈에 개념 맵

```
+--------------------+      submit(...)      +--------------------+      get()/cancel()
|  ExecutorService   |  ------------------>  |       Future       |  ------------------>
| (스레드 풀, 큐,정책) |                      | (비동기 결과 핸들)  |   (결과, 예외, 취소) |
+--------------------+                       +--------------------+
         ^
         | execute/submit
         |
   +---------------+
   | Runnable/     |
   | Callable<V>   |  ← 작업 단위
   +---------------+
```

- **ExecutorService**: 스레드/큐/거부 정책을 관리하는 **실행기(스레드 풀)**.
- **Callable\<V>**: 값을 **반환**하고 **체크 예외**를 던질 수 있는 작업.
- **Future\<V>**: 비동기 작업의 **결과/상태**를 다루는 핸들(완료 대기, 취소, 예외 확인).

---

## 1. 왜 ExecutorService를 쓰는가?

- 직접 `new Thread(...)` 남발 → **스레드 폭주, 자원 고갈, 종료 누락, 예외 유실**.
- ExecutorService는 **스레드 재사용 + 큐잉 + 백프레셔(거부 정책) + 표준 종료 절차**를 제공.
- **자체 메모리 장벽**: 작업 제출 시점과 실행 시점 사이에 happens-before 관계가 형성되어, 제출한 쓰기가 작업에서 보임(가시성 보장).

---

## 2. 생성: Executors vs ThreadPoolExecutor

### 2.1 빠른 시작 (팩토리)
```java
ExecutorService fixed    = Executors.newFixedThreadPool(8);
ExecutorService cached   = Executors.newCachedThreadPool();
ExecutorService single   = Executors.newSingleThreadExecutor();
ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(2);
```

> 주의: 일부 팩토리는 **무제한 큐/스레드**로 설정되어 운영 환경에서 메모리 폭증 위험.  
> 실무에서는 가급적 **`ThreadPoolExecutor`를 직접 구성**해 **bounded queue** + **거부 정책**을 명시하세요.

### 2.2 안전한 표준 풀 (권장 템플릿)
```java
import java.util.UUID;
import java.util.concurrent.*;

public final class Pools {
  public static ExecutorService ioBound(int core, int queueCapacity, String namePrefix) {
    BlockingQueue<Runnable> q = new ArrayBlockingQueue<>(queueCapacity);
    ThreadFactory tf = r -> {
      Thread t = new Thread(r);
      t.setName(namePrefix + "-" + UUID.randomUUID());
      t.setDaemon(false);
      return t;
    };
    return new ThreadPoolExecutor(
        core, core, 0L, TimeUnit.MILLISECONDS,
        q, tf,
        new ThreadPoolExecutor.CallerRunsPolicy() // 자연스러운 백프레셔
    );
  }
}
```

- **유한 큐**: 메모리 상한.  
- **CallerRunsPolicy**: 과부하 시 **제출자 스레드가 직접 실행** → 자동 속도 제한.

---

## 3. 작업 단위: Runnable vs Callable

| 항목 | Runnable | Callable\<V> |
|---|---|---|
| 시그니처 | `void run()` | `V call() throws Exception` |
| 반환 | 없음 | **있음** |
| 체크 예외 | 던질 수 없음 | **던질 수 있음** |
| 제출 | `execute/submit` | **`submit`** |

```java
Callable<Integer> task = () -> {
  // 비지니스 로직, 예외 가능
  return 42;
};
Future<Integer> f = executor.submit(task);
```

---

## 4. Future — 결과, 예외, 취소

### 4.1 핵심 메서드
| 메서드 | 의미 |
|---|---|
| `get()` | 완료까지 블록하고 결과 반환(성공/예외/취소 포함) |
| `get(timeout, unit)` | 타임아웃 시 `TimeoutException` |
| `isDone()/isCancelled()` | 상태 조회 |
| `cancel(mayInterruptIfRunning)` | **협력적 취소** 요청(인터럽트 시도) |

```java
try {
  Integer v = f.get(3, TimeUnit.SECONDS);
} catch (InterruptedException e) {
  Thread.currentThread().interrupt(); // 정책: 인터럽트 재설정
} catch (TimeoutException e) {
  f.cancel(true); // 타임아웃 → 취소 시도
} catch (ExecutionException e) {
  Throwable cause = e.getCause(); // 실제 예외
  // 로깅/보상 처리
}
```

### 4.2 협력적 취소(중요)
```java
Callable<Void> cancellable = () -> {
  while (!Thread.currentThread().isInterrupted()) {
    // 작업...
    // 차단 호출 사용 시 InterruptedException 처리
    try {
      Thread.sleep(50);
    } catch (InterruptedException ie) {
      Thread.currentThread().interrupt(); // 재설정 후 정리
      break;
    }
  }
  return null;
};
```

---

## 5. 제출 방식: execute vs submit

- `execute(Runnable)`: 반환 없음, 예외는 **스레드의 UncaughtExceptionHandler**로.
- `submit(...)`: **Future 반환**, 예외는 `ExecutionException`에 래핑되어 `get()` 시점에 관찰.

> 결과/예외/취소 제어가 필요하면 **항상 `submit`**.

---

## 6. 여러 작업 한꺼번에: invokeAll / invokeAny

```java
List<Callable<Integer>> tasks = List.of(
  () -> slowCompute(1), () -> slowCompute(2), () -> slowCompute(3)
);

// 1) 모두 완료될 때까지 대기
List<Future<Integer>> fs = executor.invokeAll(tasks); // 개별 get()

// 2) 가장 먼저 "성공"한 결과 하나만
Integer first = executor.invokeAny(tasks); // 나머지는 취소 시도
```

- `invokeAll`은 **전체 완료**(실패 포함).  
- `invokeAny`는 **최초 성공** 결과를 반환(실패만 있다면 예외).

---

## 7. 스케줄링: ScheduledExecutorService

```java
ScheduledExecutorService sch = Executors.newScheduledThreadPool(2);

// 지연 실행
ScheduledFuture<String> sf = sch.schedule(() -> "done", 2, TimeUnit.SECONDS);

// 고정 비율(시각 격자 유지; 드리프트 발생 가능)
sch.scheduleAtFixedRate(() -> heartbeat(), 0, 5, TimeUnit.SECONDS);

// 고정 지연(이전 종료→지연→다음 시작)
sch.scheduleWithFixedDelay(() -> poll(), 0, 1, TimeUnit.MINUTES);
```

> **주기 작업 내부에서 예외를 반드시 잡으세요.** 던지면 태스크가 중단되어 더 이상 실행되지 않을 수 있습니다.

---

## 8. 안전한 종료(필수)

```java
executor.shutdown(); // 신규 제출 거부
try {
  if (!executor.awaitTermination(60, TimeUnit.SECONDS)) {
    executor.shutdownNow(); // 대기 중 작업 취소 + 인터럽트
    if (!executor.awaitTermination(30, TimeUnit.SECONDS)) {
      System.err.println("풀 종료 실패");
    }
  }
} catch (InterruptedException ie) {
  executor.shutdownNow();
  Thread.currentThread().interrupt();
}
```

- **항상 종료**: try/finally 또는 애플리케이션 종료 훅에서.

---

## 9. 예외 처리 — 누수 없이 보기 좋게

### 9.1 작업 내부
```java
Callable<Integer> safe = () -> {
  try {
    return dangerous();
  } catch (SpecificException se) {
    // 로깅/보상
    throw se; // 체크 예외면 그대로; 언체크면 래핑 고려
  }
};
```

### 9.2 Future에서
```java
try {
  f.get();
} catch (ExecutionException ee) {
  log.error("task failed", ee.getCause());
}
```

### 9.3 주기 작업
```java
sch.scheduleAtFixedRate(() -> {
  try {
    doWork();
  } catch (Throwable t) {
    log.error("periodic task failed", t); // 삼켜지지 않게!
  }
}, 0, 10, TimeUnit.SECONDS);
```

---

## 10. CompletionService — “먼저 끝난 것부터” 수확

```java
ExecutorService pool = Executors.newFixedThreadPool(8);
CompletionService<String> ecs = new ExecutorCompletionService<>(pool);

for (String url : urls) {
  ecs.submit(() -> fetch(url));
}

for (int i = 0; i < urls.size(); i++) {
  Future<String> f = ecs.take();     // 완료 순서대로
  try {
    System.out.println(f.get());
  } catch (ExecutionException e) {
    log.warn("fetch failed", e.getCause());
  }
}
pool.shutdown();
```

---

## 11. FutureTask — 직접 제어가 필요할 때

```java
FutureTask<Integer> ft = new FutureTask<>(() -> 123);
new Thread(ft, "single-worker").start();
System.out.println(ft.get());
```

- `Runnable`이면서 `Future`이기도 한 **양면 객체**: 캐시-스탬피드 방지, 재사용, 래핑에 쓸모.

---

## 12. 타임아웃·취소 유틸(레시피)

### 12.1 타임아웃 래퍼
```java
static <T> T getOrCancel(Future<T> f, long timeout, TimeUnit unit, T fallback) {
  try { return f.get(timeout, unit); }
  catch (TimeoutException te) { f.cancel(true); return fallback; }
  catch (InterruptedException ie) { f.cancel(true); Thread.currentThread().interrupt(); return fallback; }
  catch (ExecutionException ee) { return fallback; }
}
```

### 12.2 첫 성공 or 타임아웃
```java
static <T> T firstSuccessOrTimeout(ExecutorService ex, List<Callable<T>> tasks, long t, TimeUnit u) throws Exception {
  List<Future<T>> fs = new java.util.ArrayList<>(tasks.size());
  try {
    for (Callable<T> c : tasks) fs.add(ex.submit(c));
    long deadline = System.nanoTime() + u.toNanos(t);
    while (!fs.isEmpty()) {
      for (var it = fs.iterator(); it.hasNext();) {
        Future<T> f = it.next();
        if (f.isDone()) {
          try { return f.get(); }
          catch (ExecutionException ee) { it.remove(); } // 실패면 다른 것 대기
        }
      }
      if (System.nanoTime() > deadline) throw new TimeoutException();
      Thread.sleep(5);
    }
    throw new Exception("모든 작업 실패");
  } finally {
    for (Future<T> f : fs) f.cancel(true);
  }
}
```

---

## 13. ThreadFactory & 모니터링

```java
import java.util.concurrent.atomic.AtomicInteger;

ThreadFactory tf = new ThreadFactory() {
  private final AtomicInteger seq = new AtomicInteger(1);
  public Thread newThread(Runnable r) {
    Thread t = new Thread(r);
    t.setName("api-" + seq.getAndIncrement());
    t.setDaemon(false);
    // t.setUncaughtExceptionHandler((th, ex) -> log.error("uncaught", ex));
    return t;
  }
};
ExecutorService pool = Executors.newFixedThreadPool(8, tf);
```

- 이름 규칙은 **스레드 덤프/로그 상관관계**에 매우 유용.
- (선택) **UncaughtExceptionHandler** 등록.

---

## 14. 스케줄링 선택: fixed-rate vs fixed-delay

| 메서드 | 기준 | 적합 사례 |
|---|---|---|
| `scheduleAtFixedRate` | **이상적 시작 시각 간격** 고정 | 메트릭 출力처럼 주기 **격자**가 중요한 경우 |
| `scheduleWithFixedDelay` | **이전 종료→지연** 후 시작 | 폴링/청소처럼 작업 시간 변동이 큰 경우 |

---

## 15. 풀 크기 가이드 (근사)

- **CPU 바운드**: `≈ CPU 코어 수`  
- **I/O 바운드**: 대기 비율에 따라 더 크게  
  $$ N_{\text{threads}} \approx N_{\text{cores}} \times \left(1 + \frac{T_{\text{wait}}}{T_{\text{compute}}}\right) $$
- 정답은 **부하/성능 테스트**.

---

## 16. 실전 시나리오 — “I/O 바운드 호출 + 캐시 스탬피드 방지 + 백프레셔”

```java
import java.util.concurrent.*;
import java.util.concurrent.atomic.*;
import java.util.*;

class DataService {
  private final ConcurrentHashMap<String, Future<String>> cache = new ConcurrentHashMap<>();
  private final ExecutorService ioPool;

  DataService() {
    this.ioPool = new ThreadPoolExecutor(
      16, 16, 0, TimeUnit.MILLISECONDS,
      new ArrayBlockingQueue<>(1000),
      r -> { Thread t = new Thread(r); t.setName("io-"+UUID.randomUUID()); return t; },
      new ThreadPoolExecutor.CallerRunsPolicy()
    );
  }

  public String get(String key) throws Exception {
    while (true) {
      Future<String> f = cache.get(key);
      if (f == null) {
        Callable<String> eval = () -> fetchRemote(key); // 느린 I/O
        FutureTask<String> ft = new FutureTask<>(eval);
        f = cache.putIfAbsent(key, ft);
        if (f == null) {
          f = ft;
          ioPool.execute(ft); // 최초 1회만 실행
        }
      }
      try {
        return f.get(2, TimeUnit.SECONDS);
      } catch (CancellationException | ExecutionException e) {
        cache.remove(key, f); // 실패 → 캐시 제거
        throw e;
      } catch (TimeoutException te) {
        f.cancel(true);       // 오래 걸림 → 취소 시도 및 제거
        cache.remove(key, f);
        throw te;
      } catch (InterruptedException ie) {
        Thread.currentThread().interrupt();
        throw ie;
      }
    }
  }

  private String fetchRemote(String key) throws Exception {
    // 모의 I/O
    Thread.sleep(150);
    return "V:" + key;
  }

  public void shutdown() { ioPool.shutdown(); }
}
```

포인트:
- **키별 단일 계산 공유(FutureTask)** 로 **스탬피드 방지**.
- **타임아웃/실패 시 캐시 제거**로 일관성 유지.
- **유한 큐 + CallerRunsPolicy**로 **백프레셔** 확보.

---

## 17. 흔한 실수 & 대안

| 실수 | 문제 | 대안 |
|---|---|---|
| 무제한 큐(LinkedBlockingQueue 기본) | 메모리 폭증 | **ArrayBlockingQueue(용량 지정)** |
| `newCachedThreadPool` 남용 | 스레드 폭증 | 고정/유한 풀 + 큐 |
| `Future.get()` 무한 대기 | 장애 전파 지연 | **타임아웃 + 취소** |
| 주기 작업에서 예외 미처리 | 태스크 조기 종료 | **try/catch** 로깅/복구 |
| 인터럽트 무시 | 종료 불능 | 인터럽트 **정책 준수** |
| 스레드 이름 미설정 | 디버깅 지옥 | **ThreadFactory**로 이름 지정 |

---

## 18. 테스트/운영 체크리스트

- 풀 메트릭: **큐 길이, 활성 스레드, 완료 작업 수, 거부 횟수**를 모니터링.
- 종료 테스트: **`shutdown`+`awaitTermination` 시 정상 종료** 확인.
- 실패 주입: 일부 태스크에서 **의도적 예외/타임아웃**으로 회복 시나리오 검증.
- 로깅: **`ExecutionException.getCause()`** 반드시 기록.
- 마이크로벤치: JMH로 **워밍업/분산/노이즈** 통제.

---

## 19. 보너스: CompletableFuture로의 확장(간단 브리지)

```java
static <T> java.util.concurrent.CompletableFuture<T>
toCompletable(Future<T> f, Executor e) {
  return java.util.concurrent.CompletableFuture.supplyAsync(() -> {
    try { return f.get(); }
    catch (ExecutionException ee) { throw new RuntimeException(ee.getCause()); }
    catch (InterruptedException ie) { Thread.currentThread().interrupt(); throw new RuntimeException(ie); }
  }, e);
}
```

- 복합 비동기 파이프라인이 필요하면 **CompletableFuture**로 조합/에러/타임아웃을 더 우아하게 다룰 수 있음.

---

## 20. 요약 표

| 주제 | 핵심 포인트 |
|---|---|
| 생성 | 운영은 **ThreadPoolExecutor + bounded queue + 거부 정책** |
| 제출 | 결과·예외·취소 필요 → **submit(Callable)** |
| Future | `get(timeout)`, `cancel(true)`로 **타임아웃·취소** |
| 스케줄링 | **fixedRate**(격자), **fixedDelay**(변동시간) |
| 예외 | 작업/주기 작업에서 **반드시 처리**하고 로깅 |
| 종료 | `shutdown` → `awaitTermination` → `shutdownNow` |
| 백프레셔 | 유한 큐 + `CallerRunsPolicy` |
| 튜닝 | CPU vs I/O 바운드, 식:  $$N \approx C \times (1 + \frac{T_w}{T_c})$$ |

---

## 21. 결론

- **ExecutorService + Callable + Future**는 자바 비동기의 **표준 뼈대**입니다.  
- 올바른 풀 구성(유한 큐, 거부 정책), **타임아웃/취소**, **예외/로깅**, **안전한 종료**만 지켜도 안정성은 급상승합니다.  
- 복잡한 파이프라인은 **CompletableFuture**로 자연스럽게 확장하세요.