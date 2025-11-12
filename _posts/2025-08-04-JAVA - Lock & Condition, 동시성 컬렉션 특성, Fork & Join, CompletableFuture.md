---
layout: post
title: Java - Lock & Condition, 동시성 컬렉션 특성, Fork & Join, CompletableFuture
date: 2025-08-04 14:20:23 +0900
category: Java
---
# Lock & Condition 내부 동작 원리, 동시성 컬렉션 특성, Fork/Join 심화, CompletableFuture 사용법

## 0. 한눈에 구조도

```
[앱 코드]
   ├─ Lock/Condition (ReentrantLock, ReadWriteLock, StampedLock)
   │    └─ AQS(AbstractQueuedSynchronizer) ─ CAS/park/unpark, Sync Queue, Condition Queue
   ├─ 동시성 컬렉션 (CHM, LBQ, COW, SkipList, CLQ)
   │    ├─ 락-프리/락-경량 CAS, 분리 락, 트리화, 이중 락, 스냅샷 읽기
   ├─ Fork/Join (RecursiveTask/Action, Work-Stealing, ManagedBlocker)
   └─ CompletableFuture (CompletionStage 조합, 예외, 타임아웃, 취소, 커스텀 Executor)
```

---

## 1. `Lock`과 `Condition` 내부 동작 원리

### 1.1 왜 `Lock`인가? (`synchronized` vs `Lock`)
| 항목 | `synchronized` | `ReentrantLock` |
|---|---|---|
| 획득/해제 | 진입/블록 종료 시 자동 | `lock()/unlock()` 수동 (try/finally 필수) |
| 대기/신호 | `wait/notify/notifyAll`(Object) | `Condition.await/signal/signalAll` (다중 조건 큐) |
| 타임아웃 | 불가 | `tryLock(timeout)` |
| 인터럽트 | 제한적 | `lockInterruptibly()` |
| 공정성 | 보장 없음 | `new ReentrantLock(true)` (공정), 기본 비공정(성능↑) |
| 구현 | JVM 모니터 | AQS 기반 사용자 수준 동기화 |

> 기본은 `synchronized`로 시작하고, **타임아웃/인터럽트/다중조건/진단**이 필요할 때 `Lock`로 승격.

---

### 1.2 AQS 핵심: 상태/큐/차단
- **state (int)**: 소유/재진입 카운트 등 (exclusive), 공유 모드도 지원.
- **Sync Queue(FIFO)**: 실패한 획득 시 노드를 enq → `LockSupport.park()`로 잠듦 → 해제 시 `unpark()`.
- **CAS**: 락 소유/대기 노드 링크 연산을 원자적으로 수행.

```
[lock()]                [unlock()]
  CAS state→acq 실패      state→0
     ↓                     ↓
  Sync Queue enq       head.next unpark
     ↓                     ↓
  park() (수면)         깬 스레드는 다시 CAS로 lock 경쟁
```

수식(간단 직감):
$$
T_{\text{acquire}} \approx T_{\text{CAS\_success}} + P(\text{fail}) \cdot (T_{\text{enq}} + T_{\text{park\_wake}})
$$

---

### 1.3 Condition의 두 큐: Condition Queue → Sync Queue
- 하나의 `Lock`에 **여러 Condition** 가능(서로 다른 wait-set).
- `await()` 흐름:
  1) 락 소유 검사(아니면 `IllegalMonitorStateException`)  
  2) 현재 스레드를 **Condition Queue**에 넣고 **락 해제**  
  3) `park()`로 대기  
  4) `signal()`로 깨우면, Condition 노드를 **Sync Queue**로 옮김  
  5) Sync Queue에서 **락을 다시 획득**한 뒤 `await()` 다음 줄부터 재개

ASCII 흐름도:
```
       +--------------------+                +-----------------+
       |  Condition Queue   | --signal()-->  |   Sync Queue    |
await  +--------------------+                +-----------------+
  │           (대기)                             (락 재경쟁)
  └─ unlock + park()
```

> **항상 while-재검사**: 스퍼리어스 웨이크업/경쟁으로 깨어난 뒤에도 조건 재확인 필요.

---

### 1.4 실전: `ReentrantLock + Condition` 단일-버퍼
```java
import java.util.concurrent.locks.*;

public class OneCellBuffer<T> {
    private final Lock lock = new ReentrantLock();    // 공정성 필요 시 new ReentrantLock(true)
    private final Condition notEmpty = lock.newCondition();
    private final Condition notFull  = lock.newCondition();
    private T cell;

    public void put(T v) throws InterruptedException {
        lock.lock();
        try {
            while (cell != null) {           // while 재검사
                notFull.await();             // 락 해제 + 대기
            }
            cell = v;
            notEmpty.signal();               // 대기 소비자 1개 신호
        } finally {
            lock.unlock();
        }
    }
    public T take() throws InterruptedException {
        lock.lock();
        try {
            while (cell == null) {
                notEmpty.await();
            }
            T v = cell; cell = null;
            notFull.signal();
            return v;
        } finally {
            lock.unlock();
        }
    }
}
```

---

### 1.5 추가 API와 패턴
- `tryLock(long, TimeUnit)`: 데드락 회피/순환 대기 타임아웃에 유용
- `lockInterruptibly()`: 인터럽트 가능한 획득 (취소/종료 설계에 필수)
- `awaitNanos`, `awaitUntil(Date)`, `awaitUninterruptibly`
- **다중 조건** 설계: 상태별 Condition 분리 → 불필요한 깨움 감소

---

### 1.6 `ReentrantReadWriteLock` & `StampedLock`
#### ReadWriteLock
- **여러 reader 동시** + **writer 단독**
- 업/다운그레이드: **쓰기 → 읽기** 다운그레이드만 권장(쓰기 락 보유 중 read lock 획득 후 write 해제)
```java
var rw = new ReentrantReadWriteLock();
var r = rw.readLock(); var w = rw.writeLock();

w.lock();
try {
    // write...
    r.lock();          // 다운그레이드
} finally {
    w.unlock();
}
try {
    // long read...
} finally {
    r.unlock();
}
```

#### StampedLock
- **낙관적 읽기**(optimistic read): 충돌 없으면 잠금 없이 읽기 성공
- 재검사 실패 시 read/write 락으로 격상
```java
import java.util.concurrent.locks.StampedLock;
class Point {
    private double x,y;
    private final StampedLock sl = new StampedLock();
    double distance() {
        long stamp = sl.tryOptimisticRead();
        double cx = x, cy = y;
        if (!sl.validate(stamp)) {      // 변경 감지되면
            stamp = sl.readLock();
            try { cx = x; cy = y; }
            finally { sl.unlockRead(stamp); }
        }
        return Math.hypot(cx, cy);
    }
}
```
> 장점: 읽기 경합에서 효율. 단, **인터럽트 불가/재진입 불가** 등 제약과 **Starvation** 주의.

---

### 1.7 실무 체크리스트
- [ ] **while-await** 필수(조건 재검사)
- [ ] `try/finally`로 `unlock()` 보장
- [ ] `tryLock(timeout)`으로 **교착 방지**
- [ ] 상태가 여러 개면 **Condition 분리**
- [ ] 공정성은 **정말 필요할 때만**(성능 손실)
- [ ] `StampedLock`은 **낙관 읽기**에만 제한적으로(복잡성/디버깅 난이도)

---

## 2. 동시성 컬렉션 내부 전략과 선택 가이드

### 2.1 ConcurrentHashMap (JDK 8+)
- **버킷 단위 CAS + 필요시 bin-level 동기화**  
- 빈이 길어지면 **트리화(Red-Black Tree)**  
- Resize는 **점진적**으로 이루어져 STW 방지
- **atomic 연산**: `compute`, `computeIfAbsent`, `merge`, `putIfAbsent` 등은 키 단위로 원자적
```java
var map = new java.util.concurrent.ConcurrentHashMap<String, Integer>();
map.merge("k", 1, Integer::sum);               // 1 upsert
map.compute("k", (k, v) -> v == null ? 1 : v+1); // 원자적 누적
```
> **LongAdder**와 궁합: 다중 스레드 카운팅은 `map.computeIfAbsent(k, kk -> new LongAdder()).increment()` 권장.

### 2.2 CopyOnWriteArrayList/Set
- 쓰기 때 **전체 배열 복사 → 불변 스냅샷**  
- **읽기 많고 쓰기 드문** 구성/설정/리스너 테이블에 적합
```java
var listeners = new java.util.concurrent.CopyOnWriteArrayList<Runnable>();
listeners.add(() -> { /* ... */ }); // 쓰기: 비싸지만 드뭄
for (var l : listeners) l.run();    // 읽기: 락 없이 안전
```

### 2.3 BlockingQueue
| 구현 | 내부 | 특징 |
|---|---|---|
| ArrayBlockingQueue | **단일 락** + 두 Condition | 고정 크기/메모리 예측 |
| LinkedBlockingQueue | **이중 락**(put/take 분리) | 생산/소비 병렬성, 선택적 capacity |
| SynchronousQueue | **버퍼 0** | 핸드오프(스레드-투-스레드) |
| PriorityBlockingQueue | 힙 | 우선순위, FIFO 보장 아님 |
| DelayQueue | 타임+우선순위 | 스케줄링/재시도 큐 |

### 2.4 SkipList/CLQ
- `ConcurrentSkipListMap/Set`: 정렬 + 동시성(락-프리 성질)  
- `ConcurrentLinkedQueue/Deque`: **락-프리 CAS 큐**, 대기 없는 빠른 인큐/디큐

### 2.5 선택 요약
- **캐시/카운팅**: CHM(+LongAdder)  
- **생산자-소비자**: Linked/Array/SynchronousQueue  
- **읽기≫쓰기**: COW 계열  
- **정렬 필요**: SkipList  
- **지연 최소**: CLQ

---

## 3. Fork/Join 프레임워크 심화

### 3.1 Work-Stealing 핵심
- 워커는 자신의 **데크**에 푸시한 작업을 LIFO로 처리(지역성↑)  
- 놀고 있는 워커는 타 워커의 **앞쪽(FIFO)** 에서 **steal**  
- **과분할**은 steal/스케줄링 오버헤드↑, **과소분할**은 병렬성↓ → **임계값(THRESHOLD)** 튜닝 관건

### 3.2 예제: 병렬 머지소트
```java
import java.util.concurrent.*;

class MergeSortTask extends RecursiveAction {
    private static final int THRESHOLD = 1 << 13; // 실험으로 조정
    private final int[] a; private final int lo, hi; private final int[] tmp;

    MergeSortTask(int[] a, int lo, int hi, int[] tmp) {
        this.a=a; this.lo=lo; this.hi=hi; this.tmp=tmp;
    }
    @Override protected void compute() {
        int n = hi - lo;
        if (n <= THRESHOLD) {
            java.util.Arrays.sort(a, lo, hi);
            return;
        }
        int mid = lo + (n >> 1);
        var left = new MergeSortTask(a, lo, mid, tmp);
        var right= new MergeSortTask(a, mid, hi, tmp);
        invokeAll(left, right);
        merge(lo, mid, hi);
    }
    private void merge(int lo, int mid, int hi) {
        int i=lo, j=mid, k=lo;
        while (i<mid && j<hi) tmp[k++] = (a[i]<=a[j])? a[i++] : a[j++];
        while (i<mid) tmp[k++]=a[i++];
        while (j<hi) tmp[k++]=a[j++];
        System.arraycopy(tmp, lo, a, lo, hi-lo);
    }
    static void sort(int[] a) {
        int[] tmp = new int[a.length];
        ForkJoinPool.commonPool().invoke(new MergeSortTask(a, 0, a.length, tmp));
    }
}
```

### 3.3 블로킹 작업과 ManagedBlocker
ForkJoin 워커가 블록되면 풀 전체가 굶을 수 있음 → **ManagedBlocker**로 보정
```java
class IOBoundBlocker implements ForkJoinPool.ManagedBlocker {
    private final java.nio.channels.AsynchronousFileChannel ch;
    private volatile boolean done;
    IOBoundBlocker(java.nio.channels.AsynchronousFileChannel ch){ this.ch=ch; }
    public boolean block() throws InterruptedException { /* 실제 블록 I/O */ done=true; return true; }
    public boolean isReleasable(){ return done; }
}
// 사용:
ForkJoinPool pool = new ForkJoinPool();
ForkJoinPool.managedBlock(new IOBoundBlocker(channel));
```

### 3.4 실무 팁
- [ ] **THRESHOLD**를 작업/머신에 맞게 실험  
- [ ] **공용 풀(commonPool)** 공유 주의(병렬 스트림/다른 컴포넌트와 경합) → **전용 풀** 고려  
- [ ] I/O/블로킹은 **ManagedBlocker** 또는 별도 Executor

---

## 4. `CompletableFuture` 심화 — 조합/예외/타임아웃/취소

### 4.1 기본기
- 시작: `supplyAsync`, `runAsync` (+executor)  
- 변환: `thenApply(Async)`, `thenAccept(Async)`, `thenRun(Async)`  
- 연속(FlatMap): `thenCompose`  
- 결합: `thenCombine`, `allOf`, `anyOf`, `applyToEither`, `acceptEither`  
- 예외: `exceptionally`, `handle`, `whenComplete`  
- 제어: `orTimeout`, `completeOnTimeout`, `cancel`, `obtrudeValue/Exception`

### 4.2 조합 예시: 팬아웃/팬인 + 타임아웃 + 폴백
```java
import java.util.concurrent.*;
import java.util.stream.*;

class CfPatterns {
    static final Executor IO = Executors.newFixedThreadPool(32);

    static CompletableFuture<String> fetch(String url) {
        return CompletableFuture.supplyAsync(() -> /* HTTP 호출 */ url, IO);
    }
    static CompletableFuture<String> fastest(String... urls) {
        var futures = Stream.of(urls).map(CfPatterns::fetch).toList();
        return futures.stream()
                .reduce((a,b) -> a.applyToEither(b, s -> s))
                .orElse(CompletableFuture.failedFuture(new IllegalArgumentException("empty")));
    }
    static CompletableFuture<String> withTimeout(CompletableFuture<String> cf, long ms) {
        return cf.completeOnTimeout("fallback", ms, TimeUnit.MILLISECONDS);
    }
    static CompletableFuture<Integer> robustPipeline(String id) {
        return CompletableFuture.supplyAsync(() -> id, IO)
          .thenCompose(uid -> fetch("user/" + uid))     // async 1
          .thenCompose(resp -> fetch("profile/" + resp))// async 2
          .orTimeout(2, TimeUnit.SECONDS)               // 전체 마감
          .exceptionally(ex -> "default-profile")       // 폴백
          .thenApply(String::length);
    }
}
```

### 4.3 `thenCompose` vs `thenApply`
```java
// thenApply: T -> U
cf.thenApply(x -> transform(x));

// thenCompose: T -> CF<U> (비동기 연결)
cf.thenCompose(x -> asyncTransform(x));
```
- **여러 원격 호출을 순차적으로 연결**할 때 `thenCompose`가 자연스럽고 낭비 없음.

### 4.4 에러 전파/복구 패턴
```java
CompletableFuture<String> f = fetch("api")
  .thenApply(this::parse)
  .exceptionally(ex -> cacheFallback());     // 로컬 캐시 폴백

// 결과/예외를 모두 다루려면 handle
f.handle((res, ex) -> ex != null ? "bad" : res);
```

### 4.5 allOf/anyOf 안전하게 결과 모으기
```java
CompletableFuture<String> a = fetch("a");
CompletableFuture<String> b = fetch("b");
CompletableFuture<String> c = fetch("c");

CompletableFuture<Void> all = CompletableFuture.allOf(a,b,c);

CompletableFuture<java.util.List<String>> joined = all.thenApply(v ->
    java.util.stream.Stream.of(a,b,c)
      .map(CompletableFuture::join) // 여기서 예외는 CompletionException으로 래핑됨
      .toList());
```
> 실패 원인 추적 시 `handle`로 각 future를 감싸 개별 예외/결과를 수집.

### 4.6 취소/인터럽트/리소스 정리
```java
var cf = CompletableFuture.supplyAsync(this::slow);
cf.cancel(true); // true면 인터럽트 신호 (작업이 인터럽트-친화적이어야 효과)
```
- 비동기 작업 내부에서 **인터럽트 체크**(`Thread.currentThread().isInterrupted()`) 및 **타임아웃**/**정리** 로직 필요.

### 4.7 커스텀 Executor 설계
- **CPU 바운드**: 코어 수 ~= 스레드 수 (`newFixedThreadPool(Runtime.getRuntime().availableProcessors())`)
- **I/O 바운드**: 동시 I/O 수에 맞춰 더 큼 + 큐 제한/백프레셔
- **공용 풀(commonPool)** 오염 금지: 외부 라이브러리/프레임워크와 경합 → 성능/지연 변동

### 4.8 재시도(지수 백오프) 유틸 패턴
```java
static <T> CompletableFuture<T> retry(
        Supplier<CompletableFuture<T>> op, int max, long backoffMs) {
    return op.get().handle((v, ex) -> {
        if (ex == null) return CompletableFuture.completedFuture(v);
        if (max <= 0) return CompletableFuture.failedFuture(ex);
        try { Thread.sleep(backoffMs); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
        return retry(op, max - 1, backoffMs * 2);
    }).thenCompose(x -> x);
}
```

### 4.9 디버깅 팁
- 스테이지 이름 붙이기(로깅 람다)  
- `whenComplete`로 타임라인 로깅  
- `JFR` 이벤트(스레드/락/정지/할당/네트워크)와 함께 분석

---

## 5. 성능/안정성 수식 & 체크리스트

### 5.1 Amdahl 법칙(최대 가속도 직감)
$$
S(N)=\frac{1}{(1-P)+\frac{P}{N}} \quad (P:\ \text{병렬화 가능한 비율})
$$
- 임계 영역(락 구간)이 크면 `P`가 작아져 **스레드를 더 늘려도 한계**.

### 5.2 Fork/Join 임계값 추정(감각식)
$$
T \approx \frac{W}{B\cdot N} + O_{\text{split}} + O_{\text{steal}}
$$
- `W`: 총 작업량, `B`: 단일 스레드 처리량, `N`: 워커 수  
- **`THRESHOLD`는 `O_split`와 `O_steal`을 최소화**하도록 조정

### 5.3 체크리스트(현장)
- [ ] 락 구간 최소화(필요한 필드만 보호, 불변·분할)
- [ ] **`while`-await** / **signalAll 기본** / 필요시 `signal` 최적화
- [ ] CHM `compute/merge`로 **원자적 갱신**
- [ ] LBQ/CLQ 등 목적 맞는 큐 선택(핸드오프/버퍼/우선순위)
- [ ] Fork/Join: **전용 풀** + `ManagedBlocker` for I/O
- [ ] CF: **커스텀 Executor** + **thenCompose** + **타임아웃/취소**
- [ ] JFR/GC/스레드풀 메트릭 모니터링(큐 길이, 활성 스레드, 거부율)

---

## 6. 안티패턴 모음

- `if (cond) await()` — ❌ → **`while` 사용**  
- `unlock()` 누락 — ❌ → **try/finally**  
- `CompletableFuture.get()`를 비동기 체인 중간에서 호출 — ❌ (데드락/병렬 상실)  
- 공용 `ForkJoinPool.commonPool()`에 **블로킹 작업** 제출 — ❌  
- `CopyOnWriteArrayList`에 잦은 쓰기 — ❌ (복사 폭증)  
- `ConcurrentHashMap` 값 객체에 **비원자 복합 연산** — ❌ → `compute/merge` or `LongAdder`  
- `StampedLock` 무분별 사용 — ❌ (인터럽트/재진입/디버깅 난이도)

---

## 7. 확장 예제 모음

### 7.1 `tryLock(timeout)`으로 데드락 회피
```java
boolean acquireBoth(Lock a, Lock b, long timeout, TimeUnit u) throws InterruptedException {
    long end = System.nanoTime() + u.toNanos(timeout);
    while (true) {
        if (a.tryLock(10, TimeUnit.MILLISECONDS)) {
            try {
                if (b.tryLock(10, TimeUnit.MILLISECONDS)) {
                    return true; // 둘 다 획득
                }
            } finally {
                a.unlock();
            }
        }
        if (System.nanoTime() > end) return false;
        Thread.yield();
    }
}
```

### 7.2 `LongAdder`를 이용한 고컨텐션 카운팅
```java
var counts = new java.util.concurrent.ConcurrentHashMap<String, java.util.concurrent.atomic.LongAdder>();
void incr(String key) {
    counts.computeIfAbsent(key, k -> new java.util.concurrent.atomic.LongAdder()).increment();
}
```

### 7.3 `ConcurrentLinkedQueue`로 무대기 파이프라인
```java
var q = new java.util.concurrent.ConcurrentLinkedQueue<Runnable>();
void submit(Runnable r){ q.offer(r); } // 생산
void drain() { Runnable r; while ((r=q.poll()) != null) r.run(); } // 소비(폴링)
```

### 7.4 CF: 레이트 리미터와 결합(토큰 버킷 흉내)
```java
class RateLimiter {
    private final java.util.concurrent.Semaphore sem;
    RateLimiter(int permitsPerWindow) { sem = new java.util.concurrent.Semaphore(permitsPerWindow); }
    CompletableFuture<Void> acquireAsync() {
        return CompletableFuture.runAsync(() -> {
            try { sem.acquire(); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
        });
    }
    void replenish(int permits) { sem.release(permits); }
}
```

---

## 8. 마무리 요약

- **Lock/Condition**: AQS 큐/상태/park-unpark를 이해하면 공정성/타임아웃/다중 조건을 정확히 설계할 수 있습니다.  
- **동시성 컬렉션**: 내부 전략(CAS/트리화/이중 락/스냅샷)을 이해하면 **올바른 선택과 원자적 API 사용**으로 락 직접 사용을 줄일 수 있습니다.  
- **Fork/Join**: 워크-스틸링의 **임계값**이 성능을 좌우. 공용 풀 오염/블로킹에 특히 유의하세요.  
- **CompletableFuture**: `thenCompose`로 비동기 플로우를 평탄화, **타임아웃/취소/예외**를 체계적으로 설계하고, **커스텀 Executor**로 풀 경합을 피하세요.

> 실무에서는 **불변 데이터/함수형 조합**과 **고수준 동시성 도구**를 우선 채택하고, 필요한 최소한만 Lock 레벨로 내려가십시오.  
> 모든 변경은 **부하 테스트 + JFR/메트릭**으로 검증하는 습관이 최선의 성능/안정성을 보장합니다.