---
layout: post
title: Java - Thread 생성 방법
date: 2025-07-29 19:20:23 +0900
category: Java
---
# Thread 생성 방법 (Thread, Runnable)

## 한눈에 요약

| 항목 | 핵심 포인트 |
|---|---|
| 생성 방식 | **A)** `class X extends Thread` → `new X().start()` / **B)** `class X implements Runnable` → `new Thread(new X(), "name").start()` |
| 권장 기본 | **Runnable 구현 + `new Thread(r, name)` (또는 스레드 풀)** — 단일 상속 제약 회피, 재사용/테스트 용이 |
| `start()` vs `run()` | `start()`가 **새 스레드 생성** 후 내부에서 `run()` 호출. `run()` 직접 호출은 **새 스레드 아님** |
| 종료 | `interrupt()`로 **협력적 종료**(루프 내 `isInterrupted()`/`InterruptedException` 처리) |
| 동기화 | `Atomic*`, `synchronized`, `Lock`, `volatile`로 **가시성/원자성** 보장. 공유 상태 최소화 |
| 예외 | 스레드 내부 미처리 예외는 **스레드만 종료**. `UncaughtExceptionHandler`로 로깅/알림 |
| 데몬 | 백그라운드용. **모든 user 스레드 종료 시 JVM 즉시 종료**(정리 코드 기대 금지) |
| 금지 API | `stop/suspend/resume` **사용 금지**. `run()` 직호출, `available()` 오용, 인터럽트 무시 금지 |
| 현대적 대안 | 스레드 풀(`ExecutorService`), **가벼운 스레드(Java 21+ Virtual Thread)** |

---

## Thread vs Runnable — 개념과 선택

| 비교 항목 | `extends Thread` | `implements Runnable` (권장) |
|---|---|---|
| 상속 제약 | 단일 상속으로 유연성 낮음 | 다른 클래스를 상속하면서 **작업만** 캡슐화 |
| 역할 분리 | **작업**과 **실행체**가 결합 | 작업(Runnable)과 실행체(Thread) **분리** |
| 재사용성/테스트 | 비교적 낮음 | 높음(동일 Runnable을 다양한 Thread/풀에서 실행) |
| 스레드 이름 지정 | `new MyThread("name")` 등 | `new Thread(r, "name")`로 손쉬운 네이밍 |

> **권장**: 작업 로직은 `Runnable`/`Callable`에 담고, 실행은 `Thread` 또는 **스레드 풀**에 맡기기.

---

## 방법 A — `Thread` 상속 (예제와 주의)

```java
public class MyThread extends Thread {
    public MyThread(String name) { super(name); }

    @Override public void run() {
        try {
            for (int i = 0; i < 5; i++) {
                System.out.printf("[%s] i=%d%n", getName(), i);
                Thread.sleep(500);
            }
        } catch (InterruptedException ie) {
            // 인터럽트 복원 후 종료
            Thread.currentThread().interrupt();
            System.out.printf("[%s] interrupted%n", getName());
        }
    }

    public static void main(String[] args) {
        Thread t = new MyThread("T-Worker");
        t.start();           // ✔ 새 스레드
        // t.run();          // ✘ 같은(메인) 스레드에서 실행
    }
}
```

- **주의**: **같은 Thread 인스턴스에 `start()` 2회 호출** → `IllegalThreadStateException`.

---

## 방법 B — `Runnable` 구현 (권장 기본)

```java
public class MyRunnable implements Runnable {
    private final String name;
    public MyRunnable(String name) { this.name = name; }

    @Override public void run() {
        try {
            for (int i = 1; i <= 3; i++) {
                System.out.printf("[%s] task step %d%n", name, i);
                Thread.sleep(300);
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            System.out.printf("[%s] interrupted%n", name);
        }
    }

    public static void main(String[] args) {
        Thread t = new Thread(new MyRunnable("R-Worker"), "R-Worker");
        t.start();
    }
}
```

- 작업 로직을 **Thread와 분리**하므로 **스레드 풀**, **테스트**에 그대로 재사용 가능.

---

## 람다/익명 클래스 — 보일러플레이트 최소화

```java
// 익명 클래스
new Thread(new Runnable() {
    @Override public void run() { System.out.println("익명 Runnable"); }
}, "Anon").start();

// 람다 (Java 8+)
new Thread(() -> System.out.println("람다 Runnable"), "Lambda").start();
```

---

## `start()` vs `run()` — 헷갈리는 포인트 정리

```java
Thread t = new Thread(() -> System.out.println(Thread.currentThread().getName()));
t.run();   // 출력: main (새 스레드 X)
t.start(); // 출력: Thread-0 (새 스레드 O)
```

- **항상 `start()`**로 **새 스레드**를 시작.
- `run()` 직접 호출은 **일반 메서드 호출**과 동일.

---

## 스레드 생명주기와 주요 메서드

### 상태(State)

`NEW → RUNNABLE ↔ BLOCKED/WAITING/TIMED_WAITING → TERMINATED`

| 메서드/상황 | 효과 |
|---|---|
| `start()` | `NEW → RUNNABLE` |
| `sleep(ms)` | `TIMED_WAITING`(시간 경과/인터럽트 시 깨어남) |
| `wait()/notify` | 모니터 조건 대기/깨움 (`synchronized` 블록 내에서만) |
| `join()` | **다른 스레드 종료 대기** → 호출자 `WAITING/TIMED_WAITING` |
| **종료** | `run()` 종료 또는 미처리 예외 → `TERMINATED` |

### `join()` 패턴

```java
Thread t1 = new Thread(task1, "t1");
Thread t2 = new Thread(task2, "t2");
t1.start(); t2.start();
t1.join();  // t1 끝까지
t2.join(1000); // 최대 1초만 대기
```

---

## 인터럽트(Interrupt) — 안전한 종료의 표준

### 핵심 규칙

- `interrupt()`는 **요청**일 뿐 **강제 종료 아님**.
- 차단 메서드(`sleep`, `wait`, `join`, 일부 I/O 등)는 **`InterruptedException`** 발생.
- 루프에서는 `Thread.currentThread().isInterrupted()` **폴링** 또는 예외 캐치로 **협력 종료**.

### 패턴: 루프 + 슬립

```java
public void run() {
    while (!Thread.currentThread().isInterrupted()) {
        try {
            // 작업...
            Thread.sleep(200);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt(); // 상태 복원
            break; // 정리 후 종료
        }
    }
}
```

- **인터럽트 삼키지 말 것**: 예외를 잡은 뒤 `interrupt()`로 **상태 복원**하거나 종료.

---

## 데몬(daemon) vs 사용자(user) 스레드

```java
Thread daemon = new Thread(() -> { /* 백그라운드 */ }, "daemon");
daemon.setDaemon(true); // 반드시 start() 전에
daemon.start();
```

- **모든 user 스레드가 종료되면** JVM은 **데몬을 강제 종료**.
- 데몬에서 파일/네트워크 **정리 작업 기대 금지**(언제든 중단될 수 있음).

---

## 동기화 & 메모리 모델 — 원자성/가시성/순서

### 왜 문제가 생기는가?

- `count++`는 **원자적 아님**(읽기→증가→쓰기). 다중 스레드에서 **레이스 컨디션**.

### 해결 옵션

#### A) `synchronized` (내장 락)

```java
class SafeCounter {
    private int n;
    public synchronized void inc() { n++; }
    public synchronized int get() { return n; }
}
```
- **진입/이탈**이 **happens-before**를 형성 → **가시성** 보장.

#### B) `AtomicInteger` (무잠금 CAS)

```java
AtomicInteger n = new AtomicInteger();
n.incrementAndGet();
```

#### C) `volatile` (가시성 전용)

```java
volatile boolean running = true; // 플래그 변경이 즉시 보임
```
- **원자성 보장 X**. 카운터 등엔 부적합.

#### D) `Lock` (`ReentrantLock`) — 고급 기능

```java
Lock lock = new ReentrantLock();
lock.lock();
try { /* 임계 구역 */ }
finally { lock.unlock(); }
```

### 안전한 게시(Safe Publication)

- 불변/`final` 필드, 정적 초기화, 락/volatile 경유, 스레드 안전 컨테이너를 통해 **초기화가 완료된 객체**를 공개.

### 금지 패턴: 잘못된 DCL

```java
// 잘못된 예시
class Lazy {
  private static Helper h;
  static Helper get() {
    if (h == null) {         // 레이스
      synchronized(Lazy.class) {
        if (h == null) h = new Helper(); // h에 volatile 필요
      }
    }
    return h;
  }
}
```
- **해결**: `h`를 `volatile`로.

---

## UncaughtExceptionHandler — 보이지 않는 예외 잡기

```java
Thread t = new Thread(() -> { throw new RuntimeException("Boom"); }, "Worker");
t.setUncaughtExceptionHandler((th, ex) -> {
    System.err.printf("Thread %s crashed: %s%n", th.getName(), ex);
});
t.start();
```

- 전역 설정: `Thread.setDefaultUncaughtExceptionHandler(...)`
- 스레드 풀을 쓰면 **작업 예외는 `Future.get()` or 핸들러에서 처리**(별도 주제).

---

## 스레드 이름/우선순위/팩토리

```java
Thread t = new Thread(r, "ETL-Worker-1");
t.setPriority(Thread.NORM_PRIORITY); // 우선순위 의존 로직은 지양
```

`ThreadFactory`로 **일관된 네이밍/데몬/우선순위/핸들러** 지정:
```java
import java.util.concurrent.ThreadFactory;
import java.util.concurrent.atomic.AtomicInteger;

class NamedFactory implements ThreadFactory {
    private final AtomicInteger seq = new AtomicInteger();
    public Thread newThread(Runnable r) {
        Thread t = new Thread(r, "app-worker-" + seq.incrementAndGet());
        t.setUncaughtExceptionHandler((th, ex) -> ex.printStackTrace());
        return t;
    }
}
```

---

## 반드시 피할 것(안티패턴)

- `stop() / suspend() / resume()` **금지**(데드락/일관성 파괴).
- `run()` 직접 호출(새 스레드 미생성).
- 인터럽트 **무시**(무한 대기/종료 불가).
- 공유 변경 상태 방치(레이스/가시성 버그).
- 데몬 스레드에서 **파일/DB 정리** 기대.
- `Thread.sleep()`으로 **락 대기/폴링** 해결 시도(깨짐). 조건/락/동시성 유틸 사용.

---

## 스레드 풀/고수준 API와의 접점 (간단 스니펫)

> 대량/반복 작업은 **직접 Thread 생성 대신 스레드 풀** 권장

```java
import java.util.concurrent.*;

ExecutorService pool = Executors.newFixedThreadPool(4, new NamedFactory());
Future<Integer> f = pool.submit(() -> 42); // Callable
int v = f.get(); // 예외/취소 처리 생략
pool.shutdown();
```

---

## (선택) Virtual Thread — Java 21+ 가벼운 스레드

> **개념**: OS 스레드보다 **훨씬 가벼운** JVM 스케줄 스레드. **블로킹 동작을 가독성 그대로** 유지하며 대규모 동시성 구현.

```java
// 1) 직접 생성
Thread.ofVirtual().start(() -> System.out.println(Thread.currentThread()));

// 2) 작업당 가상 스레드 실행기
try (var exec = java.util.concurrent.Executors.newVirtualThreadPerTaskExecutor()) {
    exec.submit(() -> { /* 블로킹 호출도 OK */ });
} // 자동 종료
```

- 여전히 **`Runnable`** 실행 모델. 기존 패턴과 호환.
- CPU 바운드 대량 작업에는 스레드 수 조절/풀 전략을 함께 고려.

---

## 실전 체크리스트

- [ ] 작업은 `Runnable/Callable`로, 실행은 `Thread/Executor`.
- [ ] **항상 `start()`**, 동일 스레드 2중 시작 금지.
- [ ] 종료는 **`interrupt()` 기반 협력적** 설계.
- [ ] 공유 상태 최소화. 필요 시 `Atomic*`/`synchronized`/`Lock`/`volatile`.
- [ ] **try-with-resources**로 외부 자원 정리.
- [ ] `UncaughtExceptionHandler`로 **예외 로깅/알림**.
- [ ] 데몬 스레드는 **정리 로직 금지**.
- [ ] 금지 API 사용하지 않기.

---

## 끝판왕 종합 예제 — 안전 종료·예외 로깅·조인·동기화

```java
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.atomic.AtomicInteger;

public class ThreadPlayground {
    static class Worker implements Runnable {
        private final String name;
        private final AtomicInteger counter;
        private final CountDownLatch started;
        Worker(String name, AtomicInteger counter, CountDownLatch started) {
            this.name = name; this.counter = counter; this.started = started;
        }
        @Override public void run() {
            try {
                started.countDown(); // 시작 신호
                while (!Thread.currentThread().isInterrupted()) {
                    counter.incrementAndGet(); // 원자적 증가
                    // 업무 로직...
                    Thread.sleep(100); // 차단 메서드 → 인터럽트 반응
                    if (counter.get() > 20) throw new IllegalStateException("테스트 예외");
                }
            } catch (InterruptedException ie) {
                Thread.currentThread().interrupt();        // 상태 복원
                System.out.printf("[%s] interrupted%n", name);
            }
        }
    }

    public static void main(String[] args) throws Exception {
        AtomicInteger shared = new AtomicInteger();
        CountDownLatch started = new CountDownLatch(2);

        Thread t1 = new Thread(new Worker("W1", shared, started), "W1");
        Thread t2 = new Thread(new Worker("W2", shared, started), "W2");

        // 스레드별 UncaughtExceptionHandler
        Thread.UncaughtExceptionHandler ueh = (th, ex) ->
            System.err.printf("Crash@%s: %s%n", th.getName(), ex);
        t1.setUncaughtExceptionHandler(ueh);
        t2.setUncaughtExceptionHandler(ueh);

        t1.start(); t2.start();

        // 두 스레드가 모두 시작될 때까지 대기
        started.await();

        // 1초 뒤, 정상 종료 시도
        Thread.sleep(1000);
        t1.interrupt();
        t2.interrupt();

        // 종료 동기화
        t1.join(2000);
        t2.join(2000);

        System.out.printf("shared=%d, alive: t1=%s, t2=%s%n",
                shared.get(), t1.isAlive(), t2.isAlive());
    }
}
```

**포인트**
- `CountDownLatch`로 **시작 동기화**.
- `AtomicInteger`로 **경량 원자 카운터**.
- `InterruptedException` 처리 시 **re-interrupt** + 종료.
- 예외는 `UncaughtExceptionHandler`로 **로깅**.
- `join(timeout)`으로 **그레이스풀 종료** 기다림.

---

## 자주 묻는 질문(FAQ)

- **Q. 왜 `run()`을 호출하면 안 되나요?**
  A. 별도 스레드가 생성되지 않고 **현재 스레드**에서 실행됩니다.

- **Q. 인터럽트는 강제 종료인가요?**
  A. 아니요. **협력적 신호**입니다. 루프/차단 지점에서 **반드시 처리**하도록 작성해야 합니다.

- **Q. `volatile`만 쓰면 충분한가요?**
  A. 가시성만 보장합니다. **증감/집계**에는 `Atomic*` 또는 락이 필요합니다.

- **Q. 데몬 스레드에서 파일 닫기/DB 커밋 가능?**
  A. 보장할 수 없습니다. **모든 user 스레드가 끝나면 즉시 종료**될 수 있습니다.

---

## 마무리

- **학습용**으로 `Thread`/`Runnable` 생성법을 익히되, **실무**에서는 스레드 풀/고수준 동시성 도구를 우선 사용하세요.
- 올바른 종료(인터럽트), 확실한 동기화, 예외 로깅/모니터링을 갖춘 스레드만이 **운영 환경에서 견고**합니다.
- Java 21+라면 **Virtual Thread**로 **읽기 쉬운 동시성**을 누리되, 자원 경합/동기화 원리는 동일하게 적용됩니다.
