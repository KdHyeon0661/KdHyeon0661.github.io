---
layout: post
title: Java - Thread 생성 방법
date: 2025-07-29 19:20:23 +0900
category: Java
---
# Thread 생성 방법 (Thread, Runnable) — 자세 정리

Java에서 **스레드(Thread)**는 병렬(또는 동시) 작업을 수행하기 위한 실행 흐름이다.  
스레드를 생성하는 전통적인 방법은 크게 두 가지이다:

1. `Thread` 클래스를 **상속**하는 방법  
2. `Runnable` 인터페이스를 **구현**하는 방법

아래에서 두 방법의 문법, 차이, 예제, 관련 API(시작/종료/인터럽트/동기화)와 실무 권장사항까지 자세히 설명한다.

---

## 목차
1. Thread vs Runnable 개념 차이  
2. 방법 A — `Thread` 상속 (예제)  
3. 방법 B — `Runnable` 구현 (예제)  
4. 람다/익명 클래스 방식  
5. 스레드 시작과 `run()`의 차이 (`start()` vs `run()`)  
6. 스레드 생명주기 주요 메서드 (`join`, `interrupt`, `isAlive`, `sleep`, `yield`)  
7. 데몬 스레드(daemon)와 사용자 스레드(user thread)  
8. 동기화 & 스레드 안전 (예: race condition, `synchronized`, `volatile`, `AtomicInteger`)  
9. 예외 처리 및 UncaughtExceptionHandler  
10. 성능/실무 팁 — 스레드 풀(Executors) 및 Callable/Future  
11. 요약 / 권장 관행

---

## 1. Thread vs Runnable 개념 차이
- **Thread 상속**: `Thread`를 서브클래스로 만들어 `run()`을 오버라이드. 간단하지만 Java는 단일 상속만 허용하므로 유연성이 떨어짐.
- **Runnable 구현**: 스레드에서 실행될 작업(테스크)을 `Runnable`에 넣고 `new Thread(runnable).start()`로 실행. 재사용성과 확장성(다중 상속 회피)에서 유리.

---

## 2. 방법 A — `Thread` 상속 예제

```java
public class MyThread extends Thread {
    private String name;

    public MyThread(String name) {
        this.name = name;
    }

    @Override
    public void run() {
        // 스레드가 실행할 코드
        for (int i = 0; i < 5; i++) {
            System.out.println(name + " - count: " + i);
            try {
                Thread.sleep(500); // 0.5초 대기
            } catch (InterruptedException e) {
                System.out.println(name + " interrupted");
                return; // 종료
            }
        }
    }

    public static void main(String[] args) {
        Thread t1 = new MyThread("T1");
        t1.start(); // run()을 별도 스레드에서 실행
    }
}
```

---

## 3. 방법 B — `Runnable` 구현 예제 (권장 방식)

```java
public class MyRunnable implements Runnable {
    private final String name;

    public MyRunnable(String name) {
        this.name = name;
    }

    @Override
    public void run() {
        for (int i = 0; i < 5; i++) {
            System.out.println(name + " - count: " + i);
            try {
                Thread.sleep(500);
            } catch (InterruptedException e) {
                System.out.println(name + " interrupted");
                return;
            }
        }
    }

    public static void main(String[] args) {
        Thread t = new Thread(new MyRunnable("Worker-1"), "Worker-1");
        t.start();
    }
}
```

**장점**: 작업(Runnable)과 스레드(Thread)를 분리해 재사용성↑, 테스트 용이성↑.

---

## 4. 람다/익명 클래스 방식 (간결)

```java
// 익명 클래스
new Thread(new Runnable() {
    @Override
    public void run() {
        System.out.println("익명 Runnable 실행");
    }
}).start();

// 람다 (Java 8+)
new Thread(() -> {
    System.out.println("람다 Runnable 실행");
}, "LambdaThread").start();
```

---

## 5. `start()` vs `run()`의 차이
- `start()` : 새 스레드를 생성하고 JVM이 새 스레드에서 `run()`을 호출하도록 한다. **항상** `start()`를 사용해서 새로운 스레드에서 실행해야 한다.
- `run()` : 단순한 메서드 호출일 뿐, 현재 스레드(예: main)에서 실행된다. (새 스레드가 생성되지 않음)

```java
Thread t = new Thread(() -> System.out.println("hi"));
t.run();   // main 스레드에서 실행
t.start(); // 새 스레드에서 실행
```

---

## 6. 스레드 제어 주요 메서드

- `join()` : 다른 스레드가 끝날 때까지 현재 스레드를 대기시킴.
- `interrupt()` : 스레드에 인터럽트 신호를 보냄; `sleep()`/`wait()` 중이면 `InterruptedException` 발생.
- `isAlive()` : 스레드가 아직 실행 중인지 여부.
- `sleep(long millis)`(static) : 현재 스레드를 잠깐 멈춤(예외 처리 필요).
- `yield()`(static) : 같은 우선순위 다른 스레드에게 실행 기회 양보.

```java
Thread t = new Thread(() -> {
    try {
        Thread.sleep(2000);
    } catch (InterruptedException e) {
        System.out.println("interrupted");
    }
});

t.start();
t.interrupt(); // sleep 중이면 InterruptedException 발생
try {
    t.join(); // t가 종료될 때까지 대기
} catch (InterruptedException e) { /* handle */ }
```

---

## 7. 데몬 스레드(daemon)와 사용자 스레드
- 기본 스레드는 **user thread**: JVM은 모든 user thread가 종료될 때까지 실행을 유지.
- **daemon thread**: 백그라운드 서비스용, JVM은 모든 user thread 종료 시 daemon 스레드 강제 종료.
- 설정: `thread.setDaemon(true)` — 반드시 `start()` 전에 설정해야 함.

```java
Thread daemon = new Thread(() -> {
    while (true) {
        System.out.println("데몬 실행");
        try { Thread.sleep(1000); } catch (InterruptedException e) { break; }
    }
});
daemon.setDaemon(true);
daemon.start();
```

---

## 8. 동기화 & 스레드 안전 (race condition 예시)

### 문제: 동시 접근에 의한 race condition

```java
// RaceConditionExample.java
public class RaceConditionExample {
    static class Counter implements Runnable {
        private int count = 0;
        public void run() {
            for (int i = 0; i < 10000; i++) {
                count++; // 비원자적 연산: read-modify-write
            }
        }
        public int getCount() { return count; }
    }

    public static void main(String[] args) throws InterruptedException {
        Counter c = new Counter();
        Thread t1 = new Thread(c);
        Thread t2 = new Thread(c);
        t1.start(); t2.start();
        t1.join(); t2.join();
        System.out.println("count = " + c.getCount()); // 예상 20000이 아닐 수 있음
    }
}
```

### 해결 1: `synchronized` 사용

```java
class SafeCounter implements Runnable {
    private int count = 0;
    public synchronized void increment() { count++; }
    public void run() {
        for (int i = 0; i < 10000; i++) increment();
    }
    public int getCount() { return count; }
}
```

### 해결 2: `AtomicInteger` 사용 (권장)

```java
import java.util.concurrent.atomic.AtomicInteger;

class AtomicCounter implements Runnable {
    private AtomicInteger count = new AtomicInteger(0);
    public void run() {
        for (int i = 0; i < 10000; i++) count.incrementAndGet();
    }
    public int getCount() { return count.get(); }
}
```

**권장사항**: 가능한 경우 `synchronized`보다 `java.util.concurrent`의 원자 타입/락/컨커런시 유틸을 사용.

---

## 9. 인터럽트(Interrupt) 설계 패턴

- 스레드를 종료시키려면 `interrupt()`를 호출하고 스레드는 `InterruptedException`을 확인하거나 `Thread.currentThread().isInterrupted()`로 플래그를 점검해 안전히 종료한다.

```java
public void run() {
    while (!Thread.currentThread().isInterrupted()) {
        // 작업
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            // 인터럽트 시 루프 종료(또는 정리)
            Thread.currentThread().interrupt(); // 상태 복원 후 종료
            break;
        }
    }
}
```

---

## 10. 예외 처리 및 UncaughtExceptionHandler

- 스레드 안에서 처리되지 않은 예외가 발생하면 스레드는 종료된다. 전역 또는 스레드별로 언캐치 예외 핸들러를 설정할 수 있다.

```java
Thread t = new Thread(() -> { throw new RuntimeException("oops"); });
t.setUncaughtExceptionHandler((thread, ex) -> {
    System.err.println(thread.getName() + " threw " + ex);
});
t.start();
```

또는 전역:
```java
Thread.setDefaultUncaughtExceptionHandler((thread, ex) -> {
    // 로깅 등
});
```

---

## 11. 성능/실무 팁 — 스레드 풀(Executors) 및 Callable/Future

**직접 Thread를 생성하는 것보다 스레드 풀을 통한 작업 제출이 권장**된다(오버헤드/자원 관리).

### ExecutorService 예제 (권장)

```java
import java.util.concurrent.*;

public class ExecutorExample {
    public static void main(String[] args) throws Exception {
        ExecutorService pool = Executors.newFixedThreadPool(4);

        Runnable task = () -> {
            System.out.println(Thread.currentThread().getName() + " 실행");
        };
        pool.submit(task);

        // Callable (반환값 지원)
        Callable<Integer> callable = () -> {
            return 42;
        };
        Future<Integer> future = pool.submit(callable);
        System.out.println("result = " + future.get()); // blocking

        pool.shutdown();
    }
}
```

**장점**:
- 스레드 생성 비용 절감
- 작업 큐잉, 구성 가능한 정책, graceful shutdown 등 제공

---

## 12. Callable & Future (스레드의 결과 받기)

```java
ExecutorService ex = Executors.newSingleThreadExecutor();
Future<String> f = ex.submit(() -> {
    Thread.sleep(500);
    return "완료";
});
System.out.println(f.get()); // "완료"
ex.shutdown();
```

---

## 13. 스레드 이름, 우선순위, 그룹

- 이름: `new Thread(r, "Worker-1")` 또는 `thread.setName("...")`
- 우선순위: `thread.setPriority(Thread.MAX_PRIORITY)` (권장 비의존적 사용)
- 그룹: `ThreadGroup` (현대 코드에서는 잘 안 씀)

---

## 14. 권장 관행(요약)
- 작업을 **Runnable/Callable**로 정의하고 **스레드 풀(Executors)**을 이용해 실행하라.  
- 가능한 경우 **고수준 동시성 API**(`ExecutorService`, `CompletableFuture`, `Concurrent` 컬렉션 등)를 사용하라.  
- **절대** `run()`이 아니라 `start()`를 호출하라.  
- 공유 상태는 최소화 하고, 동기화가 필요하면 `Atomic*` 또는 `synchronized`/락으로 보호하라.  
- 스레드 종료는 `interrupt()`를 통해 신호를 보내고 스레드가 자율적으로 종료하도록 설계하라.  
- 리소스 관리(파일/소켓)는 반드시 try-with-resources 혹은 finally에서 닫아라.  
- 스레드 수는 CPU 코어 수와 작업 유형(IO-bound vs CPU-bound)을 고려해 설정하라.

---

### 마무리 예제: Runnable + ExecutorService + AtomicInteger
```java
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;

public class PoolCounterExample {
    public static void main(String[] args) throws InterruptedException {
        ExecutorService pool = Executors.newFixedThreadPool(4);
        AtomicInteger counter = new AtomicInteger(0);

        Runnable job = () -> {
            for (int i = 0; i < 10000; i++) {
                counter.incrementAndGet();
            }
        };

        for (int i = 0; i < 4; i++) pool.submit(job);

        pool.shutdown();
        pool.awaitTermination(1, TimeUnit.MINUTES);
        System.out.println("counter = " + counter.get()); // 40000
    }
}
```

---

## 결론
- 스레드를 직접 생성하는 법(extends `Thread`, implements `Runnable`)을 이해하는 것은 중요하지만, **실무에서는 스레드 풀과 고수준 동시성 유틸을 사용**하는 것이 안전하고 효율적이다.  
- 동시성은 오류가 발생하기 쉬우므로, 공유 자원 관리와 예외·중단 처리에 특히 신경 써야 한다.