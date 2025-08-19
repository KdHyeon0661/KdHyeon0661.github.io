---
layout: post
title: Java - wait(), notify(), notifyAll()
date: 2025-07-29 21:20:23 +0900
category: Java
---
# `wait()`, `notify()`, `notifyAll()` — 자세한 정리

Java의 `wait()` / `notify()` / `notifyAll()`은 **저수준 스레드 협력(통신)** API입니다. 이들 메서드는 `Object`에 정의되어 있고 **모니터(monitor)**, 즉 `synchronized`와 함께 사용되어야 합니다. 올바르게 쓰지 않으면 `IllegalMonitorStateException`, 교착(Deadlock), 논리적 버그(예: missed notification, spurious wakeup) 등이 발생하므로 주의가 필요합니다.

---

## 1. 핵심 요약 (한눈에)
- `wait()` : 현재 스레드를 **대기 상태**로 만들고, **모니터 락을 해제**한다. 다른 스레드가 `notify()`/`notifyAll()`하거나 타임아웃이 걸리면(또는 스레드가 인터럽트되면) 깨어난다. 반드시 모니터(즉 `synchronized` 블록/메서드) 안에서 호출해야 한다.
- `notify()` : 같은 모니터를 기다리고 있는 스레드 중 **임의의 하나**를 깨운다(스케줄러/VM에 따라 어떤 스레드인지 불명). 깨워진 스레드는 **모니터 락을 다시 얻어야** 실행을 계속한다.
- `notifyAll()` : 같은 모니터를 기다리고 있는 **모든 스레드**를 깨운다. 이후 각 스레드는 락 획득 순서에 따라 실행 재개 여부가 결정된다.
- 항상 **조건을 while 루프로 체크**(not if) — `while(condition) lock.wait();` — *spurious wakeup* 및 재검사를 위해 필수.

---

## 2. 동작 원리(정밀)
1. `wait()` 호출 시:
   - 반드시 현재 스레드가 해당 객체의 모니터(락)를 소유하고 있어야 함(즉 synchronized 내부).
   - 스레드는 모니터를 **반납(release)** 하고 **wait set**(대기 큐)에 들어감.
   - 나중에 `notify()`/`notifyAll()`에 의해 깨이거나 `wait(long)` 타임아웃으로 깨어나거나 `interrupt()`로 `InterruptedException`을 받음.
   - 깨어나면 곧바로 실행되는 것이 아니라 **모니터 락을 다시 획득한 후** `wait()` 다음 명령부터 실행.

2. `notify()` / `notifyAll()` 호출 시:
   - 역시 `synchronized` 내부에서 호출해야 함.
   - `notify()`는 wait set에서 하나의 스레드를 선택해 '깨움' 표시(선택된 스레드는 runnable 대기 상태가 됨).
   - `notifyAll()`은 wait set의 모든 스레드에 대해 깨움 표시.
   - 깨워진 스레드들은 **모니터 락이 해제될 때**(즉 notify를 호출한 스레드가 synchronized 블록을 빠져나갈 때) 락을 획득하려 경쟁함.

3. 중요한 포인트:
   - `notify()` 호출 직후에도 호출한 스레드가 synchronized 블록을 벗어나기 전까지 깨운 스레드는 **실제로 진행하지 못함**. 즉 notify는 *신호(signal)*만 보내고 락 해제까지는 기다림.
   - 만약 `notify()`를 호출했을 때 **대기 중인 스레드가 없으면** 그 신호는 사라짐(나중에 기다리는 스레드에게 전달되지 않음). — **lost notification** 문제

---

## 3. 필수 규칙 및 예외
- `wait()` / `notify()` / `notifyAll()`은 **반드시** 모니터 소유 상태(즉 `synchronized(obj) { ... }` 또는 `synchronized` 메서드)에서 호출해야 함. 그렇지 않으면 `IllegalMonitorStateException` 발생.
- `wait()`는 `InterruptedException`을 던질 수 있으므로 적절히 처리(try/catch)해야 함.
- `wait(long millis)`와 `wait(long millis, int nanos)`로 타임아웃 지정 가능.

---

## 4. 꼭 지켜야 할 패턴: **조건 검사 → while → wait**
항상 조건을 `while`로 검사해야 합니다. 이유는:
- `notifyAll()`로 여러 스레드가 깨여나 모두 동일한 조건을 재검사해야 할 수도 있음.
- *Spurious wakeups* (드물게, 이유 없이 깨어남)가 있기 때문.
```java
synchronized (lock) {
    while (!condition) {
        lock.wait();
    }
    // 조건 성립 시 작업 수행
}
```
`if`로 검사하면 스퍼리어스 웨이크업이나 경쟁으로 인해 잘못된 실행을 할 수 있음.

---

## 5. 잘못된 사용 예 (실수 사례)

### 5.1 `wait()`를 synchronized 없이 호출 → 예외
```java
Object lock = new Object();
lock.wait(); // IllegalMonitorStateException
```

### 5.2 `if`로 검사하면 race / spurious wakeup 문제
```java
synchronized(lock) {
    if (queue.isEmpty()) {
        lock.wait(); // 잘못된 패턴: wakeup 후 바로 poll하면 null 가능
    }
    Object x = queue.poll();
}
```

---

## 6. 실전 예제 — 생산자/소비자 (Producer-Consumer) (올바른 패턴)
아래는 고전적인 버퍼(한 칸) 구현. `notifyAll()`을 사용해 안전하게 여러 소비자/생산자 동작을 지원.

```java
public class BoundedBuffer<T> {
    private T item = null;
    private final Object lock = new Object();

    public void put(T value) throws InterruptedException {
        synchronized (lock) {
            while (item != null) {      // 버퍼가 가득하면 대기
                lock.wait();
            }
            item = value;
            lock.notifyAll();           // 소비자 깨우기
        }
    }

    public T take() throws InterruptedException {
        synchronized (lock) {
            while (item == null) {      // 버퍼가 비어 있으면 대기
                lock.wait();
            }
            T value = item;
            item = null;
            lock.notifyAll();           // 생산자 깨우기
            return value;
        }
    }
}
```

사용 예:
```java
// 생산자
new Thread(() -> {
    try {
        buffer.put("data");
    } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
}).start();

// 소비자
new Thread(() -> {
    try {
        String s = buffer.take();
    } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
}).start();
```

> **권장**: `notifyAll()`을 기본으로 사용하고, 성능 이슈가 명확히 증명될 때만 `notify()` 고려.

---

## 7. wait(long timeout) — 타임아웃 대기
- `wait(long millis)`는 타임아웃 후 자동으로 깨어날 수 있음. 깨어났다고 해서 조건이 반드시 만족되는 것은 아니므로 역시 `while` 루프 사용.
- 타임아웃은 **대체 전략(fallback)** 으로 유용. 예: 주기적으로 상태 점검, 데드락 감지, 종료 신호 확인 등.

```java
synchronized(lock) {
    long end = System.currentTimeMillis() + timeout;
    while (!condition && System.currentTimeMillis() < end) {
        long remaining = end - System.currentTimeMillis();
        if (remaining > 0) lock.wait(remaining);
    }
}
```

---

## 8. InterruptedException 처리 관례
- `wait()`가 `InterruptedException`을 던지면 스레드의 인터럽트 상태는 **클리어**됩니다.
- 일반 관례:
  - 리소스 정리 후 `return` 하거나,
  - 호출자에게 던지거나,
  - 인터럽트 상태를 보존하려면 `Thread.currentThread().interrupt()`로 재설정.

```java
try {
    lock.wait();
} catch (InterruptedException e) {
    Thread.currentThread().interrupt(); // 상태 복원
    // 필요한 정리 후 종료 또는 rethrow
}
```

---

## 9. notify() vs notifyAll() — 언제 어떤 걸 쓸까?
- `notify()` : wait set의 임의 스레드 하나만 깨움 — 빠르고 비용 적음. 하지만 **조건이 복잡하거나 여러 종류의 대기자가 섞여 있는 경우** 잘못된 스레드가 깨워져 다시 대기할 수 있어 deadlock/효율 저하 위험.
- `notifyAll()` : 모든 대기자 전부 깨움 — 각 스레드는 조건을 재검사하고, 조건 성립인 스레드만 진행. 안전하지만 깨우고 다시 경쟁하는 비용이 듦.

**실무 권장**: 단순한 경우(한 종류의 대기자, 단일-생산자-단일-소비자)라면 `notify()` 가능. 그러나 일반적으로는 `notifyAll()`을 사용해 안전성을 확보하라.

---

## 10. 스레드가 깨어난 뒤(재진입) 동작과 happens-before
- `notify()`/`notifyAll()`가 일어난 시점에서 notify를 호출한 스레드가 synchronized 블록을 벗어나 **모니터를 해제**할 때, 깨어난 스레드들이 모니터를 획득할 수 있음.
- 락 해제(모니터를 빠져나가는 시점)는 **happens-before** 관계를 제공하여 락을 건 스레드가 수행한 변경이 깨어난 스레드에서 보이게 됨(메모리 가시성 보장).

---

## 11. 위험/주의사항 정리
- `wait()`/`notify()`는 **저수준** API. 잘못 쓰기 쉬우며 버그 찾기 어려움.
- `notify()`로 잘못된 스레드를 깨우면 **livelock 혹은 deadlock** 발생 가능.
- **절대** `String` 상수, 공개된 객체(예: `this`를 외부에 공개) 같은 공유 가능한 락을 사용하면 안 됨(외부 코드가 같은 락을 사용해 교착을 유발할 수 있음). 대신 `private final Object lock = new Object();` 권장.
- `wait()`의 반환은 항상 조건 재검사가 필요함(while).
- `notify()` 호출 시점에 대기자가 없으면 신호가 사라짐 — 설계 상 이 점을 고려.

---

## 12. 고수준 대안(권장)
Java의 `java.util.concurrent` 패키지는 `wait/notify`보다 사용하기 쉬운 동기화 도구를 제공:
- `BlockingQueue` (예: `ArrayBlockingQueue`, `LinkedBlockingQueue`) — 생산자/소비자에 최적화. `put()`/`take()`가 내부적으로 적절히 블록.
- `Lock` + `Condition` (java.util.concurrent.locks) — `Condition.await()` / `signal()` / `signalAll()`은 `wait/notify`보다 명확한 조건 변수 사용 가능.
- `Semaphore`, `CountDownLatch`, `CyclicBarrier` 등.

**예: BlockingQueue 대체(더 간단, 안전)**

```java
BlockingQueue<String> q = new ArrayBlockingQueue<>(1);

// 생산자
q.put("data"); // 블록/대기 자동 처리

// 소비자
String s = q.take(); // 블록/대기 자동 처리
```

---

## 13. 예시 비교: wait/notify 구현 vs BlockingQueue

### wait/notify (직접)
```java
// 위 BoundedBuffer 클래스 예제 (put/take) 사용
```

### BlockingQueue (권장)
```java
BlockingQueue<String> buffer = new ArrayBlockingQueue<>(1);

Thread producer = new Thread(() -> {
    try { buffer.put("hello"); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
});

Thread consumer = new Thread(() -> {
    try { System.out.println(buffer.take()); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
});

producer.start(); consumer.start();
```

`BlockingQueue`는 내부적으로 락/조건/시그널을 잘 처리하므로 애플리케이션 코드가 단순해지고 안전성이 높아집니다.

---

## 14. 요약 체크리스트 (실전 팁)
- `wait()`/`notify()` 사용 규칙:
  - 항상 `synchronized` 블록/메서드 안에서 호출.
  - 조건은 `while`로 검사.
  - `wait()`는 락을 해제하고 대기, 깨어나면 락 재획득 후 실행.
  - `notifyAll()`을 우선 사용. 성능 검증 후 `notify()` 선택.
  - `wait(long)` 사용 시에도 `while`로 재검사.
  - `InterruptedException` 처리 시 인터럽트 상태 보존 고려.
  - 공유 락 객체는 `private final Object lock = ...`로 만들 것.
- 가능하면 **고수준 동시성 유틸(BlockingQueue, Condition 등)** 사용.

---

## 참고(짧게)
- `IllegalMonitorStateException` : `wait/notify`를 synchronized 없이 호출할 때 발생.
- `spurious wakeup` : 이유 없이 깨어나는 현상(while 필요).
- `lost notification` : notify를 기다리기 전 호출하면 신호가 사라져 대기자가 영원히 기다릴 수 있음(조건 기반 설계로 회피).
