---
layout: post
title: Java - Object 클래스
date: 2025-07-19 20:20:23 +0900
category: Java
---
# Java의 Object 클래스

## 0. 왜 `Object`인가 — “모두의 부모”

- **Java의 모든 클래스는 암묵적으로 `java.lang.Object`를 상속**합니다. (예외: 기본형 `int`, `double` 등은 객체가 아님)
- 따라서 모든 객체는 최소한 다음 메서드 군을 **보유/상속**합니다.

### 0.1 주요 메서드 시그니처(요약)

| 메서드 | 요약 |
|---|---|
| `public final Class<?> getClass()` | 런타임 타입 정보 반환(리플렉션의 시작점) |
| `public native int hashCode()` | 해시값(기본은 식별자 기반) |
| `public boolean equals(Object obj)` | 논리적 동등성(기본은 `==`) |
| `protected native Object clone()` | 얕은 복제(`Cloneable` 필요) |
| `public String toString()` | 문자열 표현(기본: `클래스@16진수해시`) |
| `public final void wait(...)`, `notify()`, `notifyAll()` | 객체 모니터 기반 동기화 원시 |
| `@Deprecated protected void finalize()` | GC 직전 훅(비권장, 향후 제거 예정) |

> `getClass()`/`wait()`/`notify()`/`notifyAll()`은 **`final`** 이라 오버라이드 불가.

---

## 1. `equals(Object)` — “논리적 동등성” 계약

### 1.1 기본 동작과 오버라이드 필요성
- 기본 구현은 **동일성**(same reference) 비교: `this == obj`.
- **값 객체(Value Object)** 를 정의한다면 반드시 **오버라이드**하여 **내용 기반**으로 동등성을 정의.

### 1.2 `equals`가 지켜야 할 5가지 규약
1. **반사성(Reflexive)**: `x.equals(x)`는 `true`.
2. **대칭성(Symmetric)**: `x.equals(y)`와 `y.equals(x)`는 동일 결과.
3. **추이성(Transitive)**: `x=y`, `y=z`이면 `x=z`.
4. **일관성(Consistent)**: 비교 대상이 불변이면 결과도 변하지 않음.
5. **null-비동등**: `x.equals(null)`은 `false`.

### 1.3 구현 패턴 (instanceof 패턴 매칭, null-안전)
```java
public final class Email {
    private final String local;
    private final String domain;

    public Email(String local, String domain) {
        this.local  = local;
        this.domain = domain;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;                     // 빠른 경로
        if (!(o instanceof Email e)) return false;      // 타입 체크 + 캐스트
        return java.util.Objects.equals(local, e.local)
            && java.util.Objects.equals(domain, e.domain);
    }

    @Override
    public int hashCode() {
        return java.util.Objects.hash(local, domain);
    }
}
```

> **getClass vs instanceof**:  
> - **`instanceof` 방식**: 하위 타입도 **같은 값 의미**를 공유할 때 적합.  
> - **`getClass()` 비교**: **정확히 같은 런타임 타입**만 동등하게 취급할 때 적합.  
> 값 의미가 **상속으로 확장될 여지**가 있다면 `instanceof`가 보통 더 안전합니다.

### 1.4 컬렉션과의 일관성
- `HashSet`, `HashMap`, `LinkedHashMap` 등 **해시 기반 컬렉션**은 `equals`와 `hashCode`의 **일관**을 전제.
- 정렬 컬렉션(`TreeSet`/`TreeMap`)을 쓸 때는 `equals`와 `compareTo`도 **가능하면 일치**하도록 설계.

---

## 2. `hashCode()` — 해시 기반 컬렉션의 친구

### 2.1 규약
- `x.equals(y) == true` 라면 **반드시** `x.hashCode() == y.hashCode()`.
- 반대는 필수 아님(충돌 허용)이나, **충돌은 적을수록 성능↑**.

### 2.2 구현 팁
- `java.util.Objects.hash(...)` 또는 **수동 조합**: `31 * result + fieldHash`.
- **가변 필드**로 `equals`/`hashCode`를 구성하면, 컬렉션에 넣은 뒤 값이 바뀌었을 때 **검색 실패/불변식 파괴** 위험 → 가급적 **불변** 또는 **키 필드 고정**.

---

## 3. `toString()` — 로그/디버깅/도메인 출력

### 3.1 기본 형태
- 기본은 `클래스이름@16진수hashCode`, 예: `Email@5a39699c`.

### 3.2 오버라이드 가이드
- **간결·의미있는 요약**만 담기(민감 정보 노출 금지).
- 디버깅/로그 가치를 높이되, **무거운 계산 금지**.
- 예:
```java
@Override public String toString() {
    return "Email{" + local + "@" + domain + "}";
}
```

> **레코드(record)** 는 컴포넌트 기반 `toString/equals/hashCode`를 자동 제공.

---

## 4. `getClass()` — 런타임 타입과 리플렉션의 관문

- **`final` 메서드**: 오버라이드 불가.
- 반환 타입은 `Class<?>`; 리플렉션으로 필드/메서드 탐색 가능.
```java
Object o = new java.util.ArrayList<>();
System.out.println(o.getClass().getName()); // java.util.ArrayList
```
- `equals` 설계에서 **정확 타입 비교가 필요**하면 `getClass()` 사용.

---

## 5. `clone()` — 얕은 복제, 주의 깊게만 쓰기

### 5.1 전제
- `Object.clone()`은 **`protected`** + **얕은 복제**.  
- 사용하려면 **`Cloneable` 인터페이스**를 구현하고, 보통 **가시성을 `public`으로 올려 재정의**.

### 5.2 기본 패턴
```java
public class Person implements Cloneable {
    private String name;             // String은 불변이라 얕은 복제 OK
    private java.util.Date birth;    // 가변 → 깊은 복제 고려

    @Override
    public Person clone() {          // 가시성 public, 예외 제거(권장)
        try {
            Person copy = (Person) super.clone(); // 얕은 복제
            copy.birth = (java.util.Date) birth.clone(); // 가변 필드는 깊은 복제
            return copy;
        } catch (CloneNotSupportedException e) {
            throw new AssertionError(e);          // 발생하지 않도록 설계
        }
    }
}
```

### 5.3 대안(권장 순)
1) **복사 생성자**: `new Person(other)`  
2) **정적 팩토리**: `Person.from(other)`  
3) (복잡한 그래프) 전용 매퍼/직렬화

> `clone/Cloneable`은 설계가 난해하고 함정이 많습니다. **복사 생성자/팩토리**가 일반적으로 더 안전합니다.

---

## 6. `finalize()` — 쓰지 마세요

- GC 직전 호출되는 훅이었으나, **비결정성/성능/보안 문제**로 **비권장** 및 **폐지 예정/중단 추세**.
- 외부 자원 해제는 **`AutoCloseable` + try-with-resources** 사용이 표준.
```java
try (var in = java.nio.file.Files.newInputStream(path)) {
    // 사용
} // 자동 close
```

---

## 7. `wait()/notify()/notifyAll()` — 객체 모니터 기반 동기화

### 7.1 핵심 규칙
- 세 메서드는 **반드시 같은 객체 모니터를 보유한 `synchronized` 블록 안**에서 호출.
- **스퍼리어스 웨이크업**(허위 기상)을 고려해 **조건 대기 시 `while` 루프**가 표준.

### 7.2 생산자-소비자(미니 버전)
```java
class BoundedQueue<T> {
    private final java.util.Queue<T> q = new java.util.ArrayDeque<>();
    private final int cap;

    BoundedQueue(int cap){ this.cap = cap; }

    public void put(T x) throws InterruptedException {
        synchronized (q) {
            while (q.size() == cap) q.wait(); // 조건 충족까지 대기
            q.add(x);
            q.notifyAll(); // 소비자 깨우기
        }
    }
    public T take() throws InterruptedException {
        synchronized (q) {
            while (q.isEmpty()) q.wait();
            T v = q.remove();
            q.notifyAll(); // 생산자 깨우기
            return v;
        }
    }
}
```

> 실무에서는 **`java.util.concurrent`**(예: `BlockingQueue`, `ReentrantLock/Condition`) 사용을 우선 고려하세요.  
> `wait/notify`는 **저수준 원시**로서 정확한 사용이 까다롭습니다.

### 7.3 타임아웃 오버로드
- `wait(long timeout)`/`wait(long timeout, int nanos)` 제공.
- `notify()`는 임의의 하나, `notifyAll()`은 **모든 대기 스레드** 깨움.  
  **여러 조건이 섞이면 `notifyAll()`이 안전한 경우가 많음**(또는 Condition으로 분리).

---

## 8. 배열/래퍼와 `equals`의 함정

- **배열의 `equals`는 참조 동일성**. 내용 비교는 `java.util.Arrays.equals(...)` / `deepEquals(...)`.
```java
int[] a = {1,2}, b = {1,2};
System.out.println(a.equals(b));                    // false
System.out.println(java.util.Arrays.equals(a, b));  // true
```

- 부동소수점 비교 시 `Double.compare(a,b)`/`Float.compare(a,b)` 사용을 권장(NaN/±0 처리 일관).

---

## 9. 아이덴티티/유틸

- **식별 해시**가 필요하면 `System.identityHashCode(obj)` (논리 `hashCode`와 무관).
- `java.util.Objects` 유틸: `equals(a,b)`, `hash(...)`, `requireNonNull(...)`, `toString(obj, default)`.
- `String.valueOf(obj)`는 `null`도 `"null"`로 안전 변환.

---

## 10. 실전 예제 모음

### 10.1 `equals/hashCode` 누락 시 문제
```java
import java.util.*;

class Point { int x,y; Point(int x,int y){this.x=x;this.y=y;} }

public class Demo {
    public static void main(String[] args) {
        Set<Point> set = new HashSet<>();
        set.add(new Point(1,2));
        System.out.println(set.contains(new Point(1,2))); // false (기본 equals/hashCode 미구현)
    }
}
```

### 10.2 올바른 값 타입
```java
final class Point {
    final int x,y;
    Point(int x,int y){this.x=x;this.y=y;}
    @Override public boolean equals(Object o){
        if(this==o) return true;
        if(!(o instanceof Point p)) return false;
        return x==p.x && y==p.y;
    }
    @Override public int hashCode(){ return 31*x + y; }
    @Override public String toString(){ return "(" + x + "," + y + ")"; }
}
```

### 10.3 레코드(record)로 간단히
```java
public record Person(String name, int age) {} // equals/hashCode/toString 자동 생성
```

---

## 11. 표 — 메서드별 설계 체크리스트

| 메서드 | 꼭 지킬 것 | 피해야 할 것 |
|---|---|---|
| `equals` | 5대 규약, null-안전, 타입 검사 | 대칭성/추이성 위반, 가변 필드 의존 |
| `hashCode` | `equals`와 일관, 충돌 최소화 | 컬렉션 삽입 후 키 필드 변경 |
| `toString` | 간결/의미, 민감정보 제외 | 무거운 연산, 순환 참조로 인한 StackOverflow |
| `clone` | `Cloneable` + 가시성 public, 가변 필드 깊은 복사 | 그대로 얕은 복사로 유출, 예외 누수 |
| `wait/notify` | 같은 모니터, `while` 대기, `notifyAll` 고려 | 모니터 미보유 호출(예외), 조건 누락 |
| `finalize` | 사용 지양(대신 `AutoCloseable`) | 자원 해제를 finalize에 의존 |

---

## 12. 종합 미니 프로젝트 — `equals/hashCode/toString` 일관성 검증

```java
import java.util.*;

final class Money {
    private final long won;
    public Money(long won) {
        if (won < 0) throw new IllegalArgumentException();
        this.won = won;
    }
    public long value(){ return won; }

    @Override public boolean equals(Object o){
        if (this == o) return true;
        if (!(o instanceof Money m)) return false;
        return won == m.won;
    }
    @Override public int hashCode(){ return Long.hashCode(won); }
    @Override public String toString(){ return won + "원"; }
}

final class Item {
    private final String name;
    private final Money price;
    Item(String name, Money price){ this.name = name; this.price = price; }
    public Money price(){ return price; }
    @Override public String toString(){ return name + ":" + price; }
}

public class App {
    public static void main(String[] args) {
        Set<Money> set = new HashSet<>();
        set.add(new Money(1000));
        System.out.println(set.contains(new Money(1000))); // true (일관)

        List<Item> items = List.of(
            new Item("Pen", new Money(500)),
            new Item("Pad", new Money(1500))
        );
        System.out.println(items); // [Pen:500원, Pad:1500원]

        Object any = "hello";
        System.out.println(any.getClass()); // class java.lang.String
        System.out.println(System.identityHashCode(any)); // 아이덴티티 해시
    }
}
```

---

## 13. 요약

- `Object`는 **모든 클래스의 공통 조상**이며, 그 메서드는 **언어적 계약의 뼈대**입니다.
- `equals`/`hashCode`는 **함께 재정의**하고 **계약을 지키는 것**이 핵심.
- `toString`은 **간결하고 유의미**하게, 민감 정보는 금지.
- `clone`은 제한적·복잡 — **복사 생성자/팩토리**를 우선 고려.
- 동기화는 `wait/notify` 대신 **고수준 동시성 도구**를 1순위로.
- `finalize`는 **사용 지양**, 자원은 **`AutoCloseable`**로 관리.

> 결론: `Object`의 계약을 올바르게 지키는 순간, **컬렉션/동시성/디버깅/테스트** 전반에서 코드가 **예측 가능**하고 **튼튼**해집니다.