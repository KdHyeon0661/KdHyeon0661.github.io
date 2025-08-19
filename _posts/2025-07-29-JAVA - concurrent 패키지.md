---
layout: post
title: Java - concurrent 패키지
date: 2025-07-29 23:20:23 +0900
category: Java
---
# java.util.concurrent 패키지 — 자세한 정리

`java.util.concurrent` 패키지는 Java 5부터 도입된 **고성능 동시성 프로그래밍을 위한 API 모음**입니다.  
기본 스레드 제어와 `wait/notify`보다 훨씬 안전하고 편리하게 동시성 문제를 해결할 수 있도록 설계되었습니다.

---

## 1. 주요 목적
- 스레드 생성 및 관리 (스레드 풀, 스케줄링)
- 락(lock)과 동기화 도구 제공 (재진입 락, 읽기/쓰기 락, 조건 변수)
- 동기화 컬렉션 및 동시성 자료구조 제공
- 원자적 연산 지원 (Atomic 변수들)
- 작업 조합 및 병렬 처리 지원

---

## 2. 주요 컴포넌트

### 2.1 Executor Framework (실행기 프레임워크)
- **Executor**: 작업 실행 추상화 (단순 실행기)  
- **ExecutorService**: 작업 제출, 종료, Future 반환 지원 (스레드 풀 관리)  
- **ScheduledExecutorService**: 예약/주기 작업 지원  
- 대표 구현체: `ThreadPoolExecutor`, `ScheduledThreadPoolExecutor`

### 2.2 동기화 도구 (Synchronizers)
- **Lock 인터페이스**과 구현체:
  - `ReentrantLock` (재진입 락)
  - `ReadWriteLock` / `ReentrantReadWriteLock` (읽기-쓰기 락)
- **Condition 인터페이스**: `wait/notify` 대체용 조건 변수  
- **Semaphore**: 허용된 동시 접근 수 제한  
- **CountDownLatch**: 카운트다운 후 대기 해제  
- **CyclicBarrier**: 스레드 집단의 동기화  
- **Exchanger**: 두 스레드가 데이터 교환  
- **Phaser**: 유연한 단계별 스레드 동기화

### 2.3 동시성 컬렉션
- **ConcurrentHashMap**: 고성능 동시성 해시맵  
- **ConcurrentSkipListMap/Set**: 정렬된 동시성 맵/셋  
- **CopyOnWriteArrayList/Set**: 읽기 위주의 스레드 안전 컬렉션  
- **BlockingQueue 인터페이스** 및 구현체:
  - `ArrayBlockingQueue`, `LinkedBlockingQueue`, `PriorityBlockingQueue` 등  
- **BlockingDeque**, **TransferQueue** 등

### 2.4 원자 변수 (Atomic Variables)
- `AtomicInteger`, `AtomicLong`, `AtomicBoolean`  
- `AtomicReference<T>`, `AtomicStampedReference<T>` 등  
- CAS (Compare-And-Swap) 기반 원자적 업데이트 지원

### 2.5 Fork/Join Framework
- 병렬 작업 분할 및 병합 지원  
- `ForkJoinPool`, `RecursiveTask<V>`, `RecursiveAction`  
- Java 8 스트림 API의 병렬 처리 기반

### 2.6 기타
- `ThreadLocalRandom`: 다중 스레드 환경에 최적화된 난수 생성기  
- `Locks`, `Executors`, `TimeUnit`, `ThreadFactory` 등의 유틸 클래스  

---

## 3. 대표 클래스와 기능

| 클래스/인터페이스 | 설명 |
|-----------------|-----|
| ExecutorService | 작업 제출 및 관리, 종료 지원하는 스레드 풀 추상화 |
| ThreadPoolExecutor | 스레드 풀의 대표 구현체 |
| ScheduledExecutorService | 지연/주기적 작업 스케줄러 |
| ReentrantLock | 재진입 가능한 락 |
| ReentrantReadWriteLock | 읽기/쓰기 락으로 동시성 최적화 |
| Condition | 락과 연계된 조건 변수 (wait/notify 대체) |
| Semaphore | 지정된 허용 수만큼 접근 허용 |
| CountDownLatch | 특정 카운트 완료까지 대기 |
| CyclicBarrier | 지정된 수만큼 스레드 도달 후 진행 |
| ConcurrentHashMap | 동시성 지원 해시맵 |
| BlockingQueue | 생산자-소비자 패턴에 최적화된 큐 |
| AtomicInteger 등 | 원자적 변수 업데이트 지원 |
| ForkJoinPool | 분할-정복 병렬 작업 처리 |

---

## 4. 예제: ThreadPoolExecutor 생성 및 사용

```java
ExecutorService executor = Executors.newFixedThreadPool(4);

Callable<Integer> task = () -> {
    Thread.sleep(1000);
    return 123;
};

Future<Integer> future = executor.submit(task);

try {
    System.out.println("Result: " + future.get());
} catch (InterruptedException | ExecutionException e) {
    e.printStackTrace();
}

executor.shutdown();
```

---

## 5. 예제: ReentrantLock과 Condition 사용

```java
ReentrantLock lock = new ReentrantLock();
Condition condition = lock.newCondition();
boolean ready = false;

lock.lock();
try {
    while (!ready) {
        condition.await();  // wait() 대신 사용
    }
    // 작업 수행
} finally {
    lock.unlock();
}

// 다른 스레드에서
lock.lock();
try {
    ready = true;
    condition.signalAll(); // notifyAll() 대신 사용
} finally {
    lock.unlock();
}
```

---

## 6. 예제: BlockingQueue를 이용한 생산자-소비자

```java
BlockingQueue<String> queue = new LinkedBlockingQueue<>();

// 생산자
new Thread(() -> {
    try {
        queue.put("item");
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    }
}).start();

// 소비자
new Thread(() -> {
    try {
        String item = queue.take();
        System.out.println(item);
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    }
}).start();
```

---

## 7. Atomic 변수 예제

```java
AtomicInteger count = new AtomicInteger(0);

count.incrementAndGet(); // 원자적 증가

int prev = count.getAndSet(10); // 이전 값 반환 후 10으로 설정

boolean updated = count.compareAndSet(10, 20); // 현재 값이 10이면 20으로 변경
```

---

## 8. ForkJoin 예제 (병렬 합계)

```java
class SumTask extends RecursiveTask<Long> {
    private static final int THRESHOLD = 1000;
    private long[] array;
    private int start, end;

    SumTask(long[] array, int start, int end) {
        this.array = array;
        this.start = start;
        this.end = end;
    }

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
            left.fork();
            long rightResult = right.compute();
            long leftResult = left.join();
            return leftResult + rightResult;
        }
    }
}

// 사용
ForkJoinPool pool = new ForkJoinPool();
long[] numbers = new long[10000];
// ... 초기화
long total = pool.invoke(new SumTask(numbers, 0, numbers.length));
```

---

## 9. 주요 장점
- **안정성**: 직접 락 관리 및 `wait/notify`를 직접 하는 것보다 안전하고 버그 위험 적음  
- **성능**: 고성능, 낮은 락 경합 설계, CAS 원자 연산 활용  
- **편리성**: 다양한 고수준 동시성 도구 제공  
- **확장성**: 병렬 처리, 스케줄링, 동시성 자료구조 등 다양한 시나리오 지원

---

## 10. 참고
- `java.util.concurrent`는 매우 방대한 API이므로 필요에 따라 하위 패키지별로(`locks`, `atomic`, `executors`, `collections` 등) 공부하는 것을 추천합니다.
- 공식 문서와 고급 서적(“Java Concurrency in Practice”)이 큰 도움이 됩니다.
