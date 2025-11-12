---
layout: post
title: Java - static과 final
date: 2025-07-19 19:20:23 +0900
category: Java
---
# static과 final

## 0. 큰 그림 요약

| 키워드 | 핵심 의미 | 적용 대상 | 실전 포인트 |
|---|---|---|---|
| `static` | **클래스 수준**(인스턴스와 무관) | 필드, 메서드, 블록, 중첩 클래스 | 공유 상태·유틸 함수·Factory·싱글톤 Holder |
| `final` | **재할당/재정의/상속 금지** | 변수(필드/지역/매개), 메서드, 클래스 | 상수, 불변 설계, 동등성/스레드 안전성 강화 |

> 조합 **`public static final`** = “클래스 상수” (컴파일타임 상수인 경우 인라이닝/상수 폴딩됨)

---

## 1. `static` — 인스턴스와 분리된 **클래스 소속** 멤버

### 1.1 `static` 필드(클래스 변수)
- **클래스 로더당 1개**(클래스 로딩/초기화 시 생성). 모든 인스턴스가 공유.
- 접근은 `Counter.count`처럼 **클래스명.이름**이 권장.

```java
public class Counter {
    public static int count = 0;
    public Counter() { count++; }
}

new Counter(); new Counter();
System.out.println(Counter.count); // 2
```

> 동일 JVM이라도 **클래스 로더가 다르면** `static` 필드는 **각각** 유지(서블릿/플러그인 환경 유의).

### 1.2 `static` 메서드
- **객체 없이 호출**, `this` 접근 불가.
- 상속 시 **오버라이드가 아닌 “메서드 숨김(hiding)”** 발생.

```java
class A { static void who() { System.out.println("A"); } }
class B extends A { static void who() { System.out.println("B"); } }

A a = new B();
a.who(); // A  ← 정적 바인딩(참조 타입 A 기준)
B.who(); // B
```

### 1.3 `static` 초기화 블록
- 클래스 **초기화(최초 사용 시)** 에 한 번 실행.
- 체크 예외를 던질 수 없고(시그니처에 `throws` 불가), 예외 발생 시 **ExceptionInInitializerError** 포장.

```java
public class AppConfig {
    public static final int PORT;
    static {
        int p = Integer.getInteger("app.port", 8080);
        if (p < 0) throw new IllegalStateException("invalid port");
        PORT = p;
        System.out.println("static init: " + PORT);
    }
}
```

### 1.4 `static` 중첩 클래스 vs 내부 클래스
- **정적 중첩 클래스**: 외부 인스턴스에 대한 암묵 참조 없음 → **가벼움/누수 위험↓**
- **비정적 내부 클래스**: 외부 인스턴스를 캡처(암묵적 `Outer.this`) → UI/상태 결합 시

```java
public class Nutrition {
    private final int a,b,c;
    private Nutrition(Builder b) { a=b.a; this.b=b.b; c=b.c; }

    public static class Builder {      // 정적 중첩
        private final int a,b; private int c;
        public Builder(int a, int b){ this.a=a; this.b=b; }
        public Builder c(int x){ c=x; return this; }
        public Nutrition build(){ return new Nutrition(this); }
    }
}
```

### 1.5 `static import` (가독성 주의)
```java
import static java.lang.Math.*;  // sqrt, pow 등 바로 사용
double h = hypot(3, 4);          // √(3²+4²) with Java 20+ hypot
```
- 과도 사용 시 **가독성 저하**(정적 멤버 출처 불명). **수학 유틸** 정도만 절제 사용.

---

## 2. `final` — **한 번만** 대입·정의·상속

### 2.1 `final` 변수(필드/지역/매개)
- **재할당 금지**. “참조가 final”이지, **참조 대상의 가변성**과는 별개.

```java
final int[] a = {1,2};
a[0] = 9;         // OK: 배열 내용 변경
// a = new int[0]; // 컴파일 오류: 참조 재할당 불가
```

#### (1) **blank final** 필드
- 선언 시 미초기화 가능하나, **생성자/인스턴스 초기화 블록에서 1회** 반드시 대입.

```java
class Node {
    private final int id;
    Node(int id){ this.id = id; } // 모든 경로에서 반드시 대입
}
```

#### (2) **effectively final** (람다/내부 클래스 캡처 규칙)
- 명시적 `final`이 없어도 **실제로 재할당이 없다면** 캡처 허용.

```java
void run() {
    int base = 10;                 // effectively final
    Runnable r = () -> System.out.println(base);
}
```

#### (3) `final`의 메모리 모델 이점
- **생성자 종료 후** `final` 필드 값은 **안전하게 공개(safe publication)** 됨.  
  다중 스레드에서 초기화 재주입/찢김 관측 위험이 **현저히 낮음**.

### 2.2 `final` 메서드
- **오버라이드 금지**. 보안/불변식 유지가 필요한 핵심 동작 봉인.
- `static` 메서드는 원래 오버라이드 개념이 아니므로 `final` 표시는 **의미상 중복**(허용되나 실효성 적음).

### 2.3 `final` 클래스
- **상속 금지**. 대표: `String`, `Integer`, `LocalDate`, `Math`, `System`.
- 상속 확장을 차단해 **불변/보안 계약** 보장, 동적 디스패치에 의한 변경 차단.

---

## 3. **compile-time constant**와 상수 인라이닝

### 3.1 컴파일타임 상수 요건
- `public static final` **원시 타입** 또는 `String`
- **상수식**으로 초기화(메서드 호출/런타임 계산 없음)

```java
public class Consts {
    public static final int    MAX = 1024;    // 인라인됨
    public static final String NAME = "APP";  // 인라인됨
    public static final Integer BOX = 7;      // 박싱은 상수식 아님(인라인 X)
}
```

- **다른 클래스에서 참조 시 값이 바이트코드에 복사**(인라인).  
  **버전 불일치 위험**: 상수 제공 측만 재배포하고, 소비 측을 재컴파일 안 하면 **옛 값** 참조 가능.

> 변경 가능성이 있는 상수는 **상수식으로 만들지 말고**(혹은 `enum`/설정 로딩), 런타임에 읽히도록 설계.

### 3.2 `switch`와 상수
- `case` 라벨에는 **컴파일타임 상수**가 필요(또는 `enum`, `String`).

```java
switch (code) {
    case Consts.MAX: ...
}
```

---

## 4. **클래스 초기화 순서**와 트리거

### 4.1 순서(상속 고려)
1. 상위 클래스의 `static` 필드/정적 블록 →  
2. 하위 클래스의 `static` 필드/정적 블록 →  
3. `new` 시 **인스턴스** 초기화(부모 → 자식)

### 4.2 초기화 트리거
- 첫 인스턴스 생성, **정적 필드/메서드 접근**, 리플렉션 등.
- 단, **컴파일타임 상수**만 사용하면 **클래스 초기화가 생략**될 수 있음(인라인 때문).

```java
System.out.println(Consts.MAX); // Consts 초기화 생략될 수 있음
System.out.println(Consts.BOX); // 메서드/생성자 경유 → 초기화 보장
```

---

## 5. 실전 패턴

### 5.1 **클래스 상수** 모듈화
- (권장) “상수 인터페이스” **안티패턴** 지양. 전용 **final 클래스 + private 생성자** 사용.

```java
public final class Config {
    private Config() {}
    public static final String VERSION = "1.2.3";
    public static final int    DEFAULT_PORT = 8080;
}
```

### 5.2 **Initialization-on-demand holder** — 싱글톤 Lazy & 안전
```java
public final class AppContext {
    private AppContext() {}
    private static class Holder {  // 클래스 로딩 시점 지연
        static final AppContext INSTANCE = new AppContext();
    }
    public static AppContext getInstance() { return Holder.INSTANCE; }
}
```
- 클래스 초기화의 **원자성/스레드안전성**에 의존 → **동기화 불필요**.

### 5.3 `static` 유틸 + `final` 불변 객체
```java
public final class Money {
    private final long amount;
    public Money(long amount){ if(amount<0) throw new IllegalArgumentException(); this.amount=amount; }
    public long value(){ return amount; }
    public Money add(long delta){ return new Money(amount + delta); }
}

public final class MoneyOps { // 유틸
    private MoneyOps() {}
    public static Money max(Money a, Money b){ return a.value() >= b.value() ? a : b; }
}
```

### 5.4 `enum`으로 “의미 있는 상수”
- `public static final int` 대신 **타입 안전한 `enum`** 권장.

```java
public enum Level { LOW, MEDIUM, HIGH }
```

---

## 6. 제네릭과 `static`

### 6.1 클래스 타입 매개변수와 `static`
- **클래스의 타입 매개변수(T)** 는 **인스턴스 수준**.  
  따라서 **정적 컨텍스트에서 직접 사용할 수 없음**.

```java
class Box<T> {
    // static T cache;           // 컴파일 오류
    static <U> U id(U x){ return x; }  // 정적 메서드 자체를 제네릭으로
}
```

### 6.2 정적 팩토리의 제네릭
```java
public final class Pair<L,R> {
    public final L left; public final R right;
    private Pair(L l, R r){ left=l; right=r; }
    public static <L,R> Pair<L,R> of(L l, R r){ return new Pair<>(l,r); }
}
```

---

## 7. 동시성과 `static`/`final`

### 7.1 공유 정적 상태
- 다중 스레드 접근 시 **가시성/경쟁 조건** 주의. `volatile`/락/병렬 친화 컬렉션 사용.
- **초기화-한정한 정적 상수(`public static final`)**는 안전(불변 + 초기화-안전 공개).

### 7.2 `final` 필드의 안전 공개
- JMM에서 `final` 필드는 생성자 완료 후 **불변처럼 관측**(레이스 없는 가시성) → **DTO/값 타입에 유리**.

---

## 8. 예제 모음

### 8.1 **상수 인라이닝**과 버전 스큐
```java
// lib-a.jar
public final class A {
    public static final int MAGIC = 7; // 상수식 → 인라인됨
}

// app(컴파일 시점에 7로 인라인)
// 이후 A.MAGIC=9로 변경해 lib-a만 교체하면,
// 재컴파일 전까지 app은 여전히 7을 사용. (버전 불일치)
```
> 변화 가능성이 있는 값은 **상수식 금지** 또는 **구성 파일/메서드 호출**로 치환.

### 8.2 `static` 숨김 vs 오버라이딩 비교
```java
class P { static void hi(){ System.out.println("P"); } void ins(){ System.out.println("P.ins"); } }
class C extends P { static void hi(){ System.out.println("C"); } @Override void ins(){ System.out.println("C.ins"); } }

P p = new C();
p.hi();  // P (정적 숨김: 참조 타입 기준)
p.ins(); // C.ins (동적 디스패치: 실제 객체 기준)
```

### 8.3 `final`과 불변 오해 방지
```java
final java.util.List<String> list = new java.util.ArrayList<>();
list.add("A"); // OK: 리스트 내용은 가변
// list = List.of("X"); // 불가: 참조 재할당 금지
```

### 8.4 `static` 초기화 실패
```java
public class Broken {
    static final int v;
    static {
        if (true) throw new RuntimeException("boom");
    }
}
// 첫 접근 시 ExceptionInInitializerError(원인: RuntimeException)
```

### 8.5 `switch`와 상수/enum
```java
public static final int TYPE_A = 1, TYPE_B = 2; // 컴파일타임 상수
int t = TYPE_A;
switch (t) { case TYPE_A -> ...; case TYPE_B -> ...; }

// 더 안전한 enum:
enum Type { A, B }
Type tt = Type.A;
switch (tt) { case A -> ...; case B -> ...; }
```

---

## 9. 표 — 실전 체크리스트

| 주제 | 체크 |
|---|---|
| 상수 정의 | 변경 가능성 있으면 **상수식 금지**(인라인 회피) / `enum`·설정 사용 |
| 초기화 순서 | 상위 `static` → 하위 `static` → 인스턴스(부모→자식) 숙지 |
| 동시성 | 공유 `static` 상태는 `volatile`/락/불변으로 제어 |
| `final` 필드 | **모든 경로에서 1회 대입**(blank final), 가변 객체는 방어적 복사 고려 |
| 정적/동적 디스패치 | `static`은 숨김(hiding), 인스턴스 메서드는 오버라이드(런타임 바인딩) |
| 제네릭 | `static` 컨텍스트에서는 타입 매개변수 직접 사용 불가 → 메서드 제네릭으로 |
| 내부/중첩 클래스 | 외부 캡처가 필요 없으면 **정적 중첩** 선호(메모리/직렬화 유리) |
| 상수 인터페이스 | 지양. **전용 final 클래스 + private 생성자** 사용 |
| 로그/디버깅 | 상수/정적 상태 남용 지양, `toString` 민감정보 금지 |

---

## 10. 미니 프로젝트 — `static` 유틸/상수 + `final` 값 타입 + Holder 싱글톤

```java
// 1) 상수 모듈
public final class AppConst {
    private AppConst() {}
    public static final String APP_NAME = "NotePad";
    public static final int    DEFAULT_PAGE = 20;
}

// 2) 값 타입(불변) — final 필드 + 메서드로 새 인스턴스 생성
public final class Page {
    private final int index;
    private final String content;
    public Page(int index, String content) {
        if (index < 0) throw new IllegalArgumentException();
        this.index = index;
        this.content = content == null ? "" : content;
    }
    public int index(){ return index; }
    public String content(){ return content; }
    public Page append(String s){ return new Page(index, content + s); }
    @Override public String toString(){ return "#" + index + " " + content; }
}

// 3) 싱글톤 컨텍스트 — Initialization-on-demand holder
public final class AppContext {
    private AppContext() {}
    private static class Holder { static final AppContext I = new AppContext(); }
    public static AppContext get(){ return Holder.I; }
    public Page firstPage(){ return new Page(0, "[" + AppConst.APP_NAME + "]"); }
}

// 4) 사용
class Main {
    public static void main(String[] args) {
        Page p = AppContext.get().firstPage().append(" Hello");
        System.out.println(p); // #0 [NotePad] Hello
        System.out.println(AppConst.DEFAULT_PAGE); // 20
    }
}
```

---

## 결론

- **`static`**: “인스턴스와 분리된 클래스 차원의 소속/생애주기”. **공유 상태 최소화**, 유틸/팩토리/Holder 패턴에 적합.  
- **`final`**: “한 번만”. **참조 재할당/오버라이드/상속 금지**를 통해 **불변식과 스레드 안전성**을 강화.  
- 두 키워드를 **상수·불변 객체·정적 팩토리·싱글톤**에 올바르게 조합하면, **성능/안정성/가독성**을 동시에 잡을 수 있습니다.