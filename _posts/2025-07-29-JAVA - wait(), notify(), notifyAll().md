---
layout: post
title: Java - wait(), notify(), notifyAll()
date: 2025-07-29 21:20:23 +0900
category: Java
---
# `wait()`, `notify()`, `notifyAll()`

## 0. 한눈에 핵심 요약

| 메서드 | 하는 일 | 반드시 필요한 전제 | 깨어난 뒤 |
|---|---|---|---|
| `wait()` | **현재 스레드를 대기**시키며 **해당 객체 모니터 락을 해제** | **`synchronized(obj)` 내부**에서만 호출 | **다시 같은 모니터 락을 획득**해야 다음 줄부터 진행 |
| `notify()` | 같은 모니터의 **대기(wait set) 중 임의의 1개** 스레드를 깨움 | **`synchronized(obj)` 내부** | 깨워진 스레드는 **호출자가 락을 풀고 나서** 락 경쟁 후 진행 |
| `notifyAll()` | 같은 모니터의 **모든 대기 스레드**를 깨움 | **`synchronized(obj)` 내부** | 모두 락 경쟁 → **조건 재검사** 필요(`while`) |

**규칙**: 조건은 항상 `while`로 감싸 재검사  
```java
synchronized (lock) {
    while (!condition()) {
        lock.wait(); // 또는 wait(timeout)
    }
    // 조건 성립 이후의 작업
}
```

---

## 1. 동작 원리 (정밀)

1. `wait()`  
   - 호출 스레드는 **해당 객체의 모니터를 반드시 보유**해야 함(= `synchronized(obj)` 안).  
   - 호출 즉시 **모니터를 반납**하고 **wait set**으로 이동하여 대기.  
   - 다음 중 하나로 깨어남:  
     - 다른 스레드의 `notify()/notifyAll()`  
     - `wait(long[, int])` 타임아웃 만료  
     - `interrupt()` → `InterruptedException` 발생  
   - 깨어난 직후 곧바로 실행되는 것이 아니라, **다시 같은 모니터 락을 획득해야** `wait()` 다음 줄부터 이어감.

2. `notify()` / `notifyAll()`  
   - 역시 **모니터 보유 중** 호출해야 하며, 호출 시점에는 **단지 신호만 설정**.  
   - 실제로 깨워진 스레드가 실행 재개하려면 **호출 스레드가 모니터를 해제**해야 함(블록/메서드 종료).

3. **Lost notification**  
   - 대기자가 없을 때 `notify()`가 호출되면 신호는 **사라짐**(버퍼링되지 않음).  
   - 그러므로 **항상 상태(조건)를 먼저 기록 → 그 다음 신호**가 안전한 패턴.

---

## 2. 필수 규칙 · 예외

- `wait/notify/notifyAll`은 **반드시** 해당 객체의 모니터를 가진 상태에서 호출(아니면 `IllegalMonitorStateException`).  
- `wait()`는 `InterruptedException`을 던짐 → **catch 후 인터럽트 상태 복원** 또는 **상위로 전파**.  
- 타임아웃 버전: `wait(long millis)`, `wait(long millis, int nanos)`  
- `while` 가드 필수: **spurious wakeup**(이유 없는 깨움)과 **경쟁 조건** 방지.

---

## 3. 안전 패턴 ① — Guarded Suspension (while + wait)

```java
final class Mailbox<T> {
    private final Object lock = new Object();
    private T message; // null = empty

    public void put(T m) throws InterruptedException {
        synchronized (lock) {
            while (message != null) {     // 가드 조건(가득 차면 대기)
                lock.wait();
            }
            message = m;                   // 상태 변경
            lock.notifyAll();              // 상태 변경 후 신호
        }
    }

    public T take() throws InterruptedException {
        synchronized (lock) {
            while (message == null) {      // 비어 있으면 대기
                lock.wait();
            }
            T m = message;
            message = null;                // 상태 변경
            lock.notifyAll();              // 생산자 깨우기
            return m;
        }
    }
}
```

**포인트**  
- **상태 변경 → notifyAll()** 순서로 신호를 보냄(가시성/순서 보장).  
- **조건은 while**(스퍼리어스·경쟁 모두 방어).  
- 신호는 **버퍼링되지 않는다** → 반드시 **상태 변수**로 의도를 기록.

---

## 4. 안전 패턴 ② — 타임아웃 대기 (`wait(timeout)`)

**벽시계(clock) 오류 방지**를 위해 `System.nanoTime()` 기반 남은시간 재계산:

```java
void awaitUntil(BooleanSupplier condition, long timeoutMillis) throws InterruptedException {
    final long deadline = System.nanoTime() + timeoutMillis * 1_000_000L;
    synchronized (lock) {
        long remainingNanos;
        while (!condition.getAsBoolean() &&
               (remainingNanos = deadline - System.nanoTime()) > 0) {
            long ms = remainingNanos / 1_000_000L;
            int ns = (int)(remainingNanos % 1_000_000L);
            lock.wait(ms, ns); // 남은 시간만큼 대기
        }
        // 여기서 condition 재검사 결과로 성공/타임아웃 판단
    }
}
```

---

## 5. 안전 패턴 ③ — 인터럽트 처리

`wait()`가 던진 `InterruptedException`은 **인터럽트 상태를 지움**. 보존하려면 복원:

```java
try {
    synchronized (lock) {
        while (!ready) lock.wait();
    }
} catch (InterruptedException e) {
    Thread.currentThread().interrupt(); // 상태 복원
    // 정리/로그/전파 중 하나 선택
}
```

---

## 6. 흔한 실수와 교정

### 6.1 `synchronized` 없이 `wait()` 호출
```java
lock.wait(); // IllegalMonitorStateException
```
**교정**: 반드시 `synchronized (lock) { lock.wait(); }`

### 6.2 `if` 가드 사용
```java
synchronized (lock) {
    if (queue.isEmpty()) lock.wait();  // ❌
    item = queue.remove();             // 경쟁/스퍼리어스 시 null 가능
}
```
**교정**: `while (queue.isEmpty()) lock.wait();`

### 6.3 `notify()` 오남용
- 생산자/소비자 **여러 종류의 대기자**가 섞인 경우 `notify()`는 **틀린 스레드를 깨울 수 있음** → 다시 대기 → **교착/활성정지** 위험.  
**권장**: 기본은 `notifyAll()`을 쓰고, 정확히 한 종류의 대기자·단일 버퍼 등 **증명 가능한** 단순 조건에서만 `notify()` 최적화를 고려.

---

## 7. `notify()` vs `notifyAll()` — 선택 기준

| 상황 | 권장 |
|---|---|
| 대기자 **종류가 하나**, 조건 **단순**(예: 단일 생산자-단일 소비자, 단일 슬롯 버퍼) | 성능상 `notify()` 고려 가능 |
| 대기자 **종류가 여러 개**(예: notFull vs notEmpty), 조건 **복잡**, 다수 스레드 | **`notifyAll()`** 권장(안전성) |
| **틀린 스레드를 깨워도** `while` 재검사로 안전·진행 보장 | 둘 다 가능하나 기본은 `notifyAll()` |

> `notifyAll()`은 **“가용성 우선”**. 비용이 염려되면 구조를 `java.util.concurrent`로 올리는 것이 일반적으로 더 낫습니다.

---

## 8. 메모리 모델 · 가시성 (happens-before)

- 같은 객체 모니터에 대해  
  - **`synchronized` 블록 종료(모니터 해제)** 는 그 이전 쓰기들을 **배출(release)**  
  - **다음에 그 모니터를 획득**하는 스레드는 그 쓰기들을 **관측(acquire)**  
- 올바른 패턴: **상태 변경**과 **대기/신호**가 **같은 락**에서 이뤄져야 함.  
  - 그래야 `notify/notifyAll` 이후 **깨운 스레드가 락을 재획득**했을 때, **상태 변경이 보임**.

---

## 9. 실전 예제 — 유한 버퍼(배열 기반) 생산자/소비자

```java
import java.util.Arrays;

public final class BoundedQueue<T> {
    private final Object lock = new Object();
    private final Object[] buf;
    private int head = 0, tail = 0, size = 0;

    public BoundedQueue(int capacity) {
        if (capacity <= 0) throw new IllegalArgumentException();
        this.buf = new Object[capacity];
    }

    public void put(T x) throws InterruptedException {
        synchronized (lock) {
            while (size == buf.length) {
                lock.wait();
            }
            buf[tail] = x;
            tail = (tail + 1) % buf.length;
            size++;
            lock.notifyAll(); // notEmpty 신호
        }
    }

    @SuppressWarnings("unchecked")
    public T take() throws InterruptedException {
        synchronized (lock) {
            while (size == 0) {
                lock.wait();
            }
            T x = (T) buf[head];
            buf[head] = null;
            head = (head + 1) % buf.length;
            size--;
            lock.notifyAll(); // notFull 신호
            return x;
        }
    }

    @Override public String toString() {
        synchronized (lock) { return "BQ" + Arrays.toString(buf); }
    }
}
```

- 두 개의 **조건(notEmpty / notFull)** 을 **하나의 wait set**으로 처리 → **`notifyAll()`이 안전**.  
- `notify()` 최적화는 **엄격한 증명** 없이는 지양.

---

## 10. Sleep vs Wait (자주 혼동)

| 항목 | `Thread.sleep` | `Object.wait` |
|---|---|---|
| 락 해제 여부 | **해제하지 않음** | **해제함(해당 모니터만)** |
| 선언 위치 | `Thread` 정적 메서드 | `Object` 인스턴스 메서드 |
| 깨우는 법 | 시간 경과 또는 인터럽트 | `notify/notifyAll`, 타임아웃, 인터럽트 |
| 사용 목적 | 단순 지연 | **조건 대기(협력)** |

---

## 11. 락 선택, 재진입, 중첩락 주의

- 락 객체는 **`private final Object lock = new Object();`** 처럼 **비공개 전용**으로.  
  - `this`, 상수 `String`(intern 공유) 등은 외부 코드와 **락 간섭** 위험.
- `synchronized`는 **재진입 가능**(같은 스레드가 같은 모니터를 여러 번 진입 가능).  
  - 그러나 `wait()`는 **호출한 모니터만 해제**. **다른 락**은 계속 보유 → **교착 위험**(또 다른 스레드가 그 락을 필요로 하면서 이 스레드를 깨우지 못하는 상황).

**안티패턴(중첩락 보유 후 wait)**
```java
synchronized (lockA) {
  synchronized (lockB) {
    while (!cond) lockA.wait(); // lockB는 붙잡은 채로 대기 → 잠재적 교착
  }
}
```
**지침**: `wait()`는 **가능하면 단일 락만 보유**한 위치에서 호출.

---

## 12. 고수준 대안 (권장)

### 12.1 `java.util.concurrent.locks.Condition`
- `await()` / `signal()` / `signalAll()` — **복수 조건을 명시적 분리** 가능.
```java
import java.util.concurrent.locks.*;

final class BQ2<T> {
    private final Lock lock = new ReentrantLock();
    private final Condition notEmpty = lock.newCondition();
    private final Condition notFull  = lock.newCondition();
    private final Object[] buf; int head, tail, size;

    BQ2(int cap) { this.buf = new Object[cap]; }

    void put(T x) throws InterruptedException {
        lock.lock();
        try {
            while (size == buf.length) notFull.await();
            buf[tail] = x; tail = (tail+1)%buf.length; size++;
            notEmpty.signal(); // 필요한 쪽만 깨움
        } finally { lock.unlock(); }
    }

    @SuppressWarnings("unchecked")
    T take() throws InterruptedException {
        lock.lock();
        try {
            while (size == 0) notEmpty.await();
            T x = (T) buf[head]; buf[head]=null; head=(head+1)%buf.length; size--;
            notFull.signal(); // 필요한 쪽만 깨움
            return x;
        } finally { lock.unlock(); }
    }
}
```

### 12.2 `BlockingQueue`
- 생산자/소비자에 최적. 내부에서 모든 대기/신호/경쟁을 처리.
```java
BlockingQueue<String> q = new java.util.concurrent.ArrayBlockingQueue<>(1024);
q.put("data");          // 가득 차면 자동 대기
String s = q.take();    // 비면 자동 대기
```

> **실무 권장**: 저수준 `wait/notify` 대신 **`BlockingQueue` 또는 `Condition`** 사용.

---

## 13. 데모: 잘못된 코드 → 교정

### 13.1 Lost Notification 재현
```java
// ❌ 잘못된 순서: notify가 먼저, 그 후에 wait
final Object lock = new Object();
volatile boolean ready = false;

// T1
new Thread(() -> {
  synchronized (lock) {
    lock.notify();            // 대기자가 아직 없음 → 신호 소실
    ready = true;             // 상태 변경이 나중
  }
}).start();

// T2
new Thread(() -> {
  synchronized (lock) {
    while (!ready) {          // 여기서 영원히 대기 가능
      try { lock.wait(); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
    }
  }
}).start();
```

**교정(상태→신호)**
```java
synchronized (lock) {
  ready = true;               // 상태 먼저
  lock.notifyAll();           // 그 다음 신호
}
```

---

## 14. 테스트 하네스(간단)

```java
public class WaitNotifyHarness {
    public static void main(String[] args) throws Exception {
        var q = new BoundedQueue<Integer>(2);
        var prod = new Thread(() -> {
            try { for (int i=0;i<10;i++){ q.put(i); System.out.println("put " + i);} }
            catch (InterruptedException e){ Thread.currentThread().interrupt(); }
        }, "producer");

        var cons = new Thread(() -> {
            try { for (int i=0;i<10;i++){ int x=q.take(); System.out.println("take "+x);} }
            catch (InterruptedException e){ Thread.currentThread().interrupt(); }
        }, "consumer");

        prod.start(); cons.start();
        prod.join(); cons.join();
        System.out.println("done");
    }
}
```

---

## 15. FAQ

- **Q. `wait()`는 어떤 락을 풀나요?** → **해당 객체의 모니터**만 풉니다. 다른 락은 유지.  
- **Q. `wait()`는 반드시 `while`과?** → 예. **항상** `while`입니다.  
- **Q. `notify()`가 누구를 깨우는지 제어할 수 있나요?** → 없음(임의). **여러 조건이면 `notifyAll()`** 또는 `Condition`으로 분리.  
- **Q. `sleep()`과 차이?** → `sleep()`은 **락을 해제하지 않음**, 협력 신호에도 반응하지 않음.

---

## 16. 실전 체크리스트

- [ ] **상태 변수**(e.g., `size`, `ready`)로 조건을 표현하고 **동일 락**에서만 읽고 쓴다.  
- [ ] **`while` 가드 + `wait()`** 를 사용한다.  
- [ ] **상태 변경 후** `notifyAll()`(또는 적합한 `notify()`/`signal()`).  
- [ ] 인터럽트는 **복원**(`Thread.currentThread().interrupt()`) 또는 **전파**.  
- [ ] `wait(timeout)`은 **남은 시간 재계산**(nanoTime)으로 구현.  
- [ ] 락 객체는 **private final** 로 감춘다(외부 간섭 차단).  
- [ ] `wait()` 호출 시 **다른 락을 들고 있지 않도록** 설계.  
- [ ] 가능하면 **`BlockingQueue` / `Condition`** 등 고수준 도구 채택.

---

## 17. 요약

- `wait/notify/notifyAll`은 강력하지만 **취약한 저수준 도구**.  
- **같은 락**에서 **상태(조건)** 를 갱신·검사하고, **`while` 가드**로 재검사하며, **상태→신호 순서**를 지키면 올바르게 동작.  
- 복잡해질수록 **`Condition`** 혹은 **`BlockingQueue`** 로 추상화 계층을 올리는 것이 **안전·성능·가독성** 모두에서 유리하다.