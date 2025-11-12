---
layout: post
title: C# - 생성자, this, base, 메서드 오버로딩, 오버라이딩
date: 2024-10-12 19:20:23 +0900
category: Csharp
---
# 생성자, this/base 키워드, 메서드 오버로딩, 메서드 오버라이딩

## 0. 빠른 개요 (TL;DR)

- **생성자**: 유효 상태 보장(검증/불변조건), **체이닝**은 `this(...)`·**상속 호출**은 `base(...)`로, 둘 다 **첫 줄**에서만 호출.  
- **this**: 현재 인스턴스 참조, 멤버 숨김 해소, **생성자 체이닝**. 확장 메서드의 `this T`는 **수신자 매개변수**.  
- **base**: 부모 생성자/메서드 호출, **재정의 메서드 내부에서 부모 구현 호출** 가능.  
- **오버로딩**: **메서드 시그니처(이름+매개변수 타입/개수/`ref/out/in` 구분)**가 달라야 한다. 반환형만 다른 것은 **불가**.  
- **오버라이딩**: `virtual`/`override`/`sealed`로 **런타임 다형성**. `new`는 **숨김(정적 바인딩)**.  
- **권장 설계**: 기본은 **컴포지션**, 상속이 필요한 경우에만 **확장 지점에 한해 `virtual`** 허용.

---

## 1. 생성자(Constructor)

### 1.1 기본 생성자와 초기화

```csharp
class Person
{
    public string Name;

    public Person() // 기본 생성자
    {
        Name = "이름 없음";
    }
}

var p = new Person();
Console.WriteLine(p.Name); // "이름 없음"
```

### 1.2 매개변수 생성자와 검증

```csharp
class Person2
{
    public string Name { get; }

    public Person2(string name)
    {
        if (string.IsNullOrWhiteSpace(name))
            throw new ArgumentException("name");
        Name = name;
    }
}
```

- **역할**: 객체를 **유효한 상태**로만 생성 가능하게 한다(불변조건 확보).

### 1.3 생성자 체이닝 — `this(...)`

```csharp
class Person3
{
    public string Name { get; }
    public int Age { get; }

    public Person3(string name) : this(name, 0) { }  // 첫 줄에서만 호출 가능
    public Person3(string name, int age)
    {
        Name = name;
        Age  = age;
    }
}
```

> **규칙**: `this(...)` 또는 `base(...)` 호출은 **반드시 첫 줄**에 있어야 하며, 둘 중 **하나만** 가능.

### 1.4 상속과 부모 생성자 호출 — `base(...)`

```csharp
class Animal
{
    public string Name { get; }
    public Animal(string name) { Name = name; }
}

class Dog : Animal
{
    public string Breed { get; }
    public Dog(string name, string breed) : base(name) // 부모 생성자 먼저
    {
        Breed = breed;
    }
}
```

- **실행 순서**: **부모 생성자 → 자식 생성자**.

### 1.5 정적 생성자(Static Constructor)

```csharp
class EnvConfig
{
    public static readonly string Root;
    static EnvConfig() // 타입 최초 사용 시 1회, 스레드 안전
    {
        Root = "/var/app";
    }
}
```

- 인스턴스 생성과 무관, **타입 단위 초기화**에 사용.

### 1.6 구조체/레코드/프라이머리 생성자

- **구조체(struct)**: C# 10+에서 **매개변수 없는 생성자 허용**. 모든 필드를 설정해야 함.
- **레코드(record)**: **프라이머리 생성자**로 간결히 정의.

```csharp
public record User(string Id, string Name); // 프라이머리 생성자

public class Point(int x, int y) // C# 12: 클래스 프라이머리 생성자
{
    public int X { get; } = x;
    public int Y { get; } = y;
}
```

---

## 2. `this` 키워드

### 2.1 현재 인스턴스 참조/숨김 해소

```csharp
class Person
{
    private string name;
    public Person(string name) => this.name = name; // 매개변수와 필드 구분
}
```

### 2.2 생성자 체이닝

```csharp
public Person(string name) : this(name, 0) { /*...*/ }
```

### 2.3 확장 메서드의 `this` (수신자)

```csharp
static class StringExt
{
    public static bool IsNullOrWhite(this string? s) =>
        string.IsNullOrWhiteSpace(s);
}

Console.WriteLine("  ".IsNullOrWhite()); // true
```

---

## 3. `base` 키워드

### 3.1 부모 생성자 호출

```csharp
class Cat : Animal
{
    public Cat(string name) : base(name) { }
}
```

### 3.2 부모 메서드 호출

```csharp
class Animal
{
    public virtual void Speak() => Console.WriteLine("...");
}

class Dog : Animal
{
    public override void Speak()
    {
        base.Speak(); // 부모 동작 재사용
        Console.WriteLine("멍멍");
    }
}
```

---

## 4. 생성자 오버로딩(Overloading)과 패턴

```csharp
class Rectangle
{
    public int Width  { get; }
    public int Height { get; }

    public Rectangle() : this(1, 1) { }      // 기본값
    public Rectangle(int size) : this(size, size) { }
    public Rectangle(int width, int height)
    {
        if (width <= 0 || height <= 0) throw new ArgumentOutOfRangeException();
        Width = width; Height = height;
    }
}
```

**팁**

- **유효성 검증 로직을 한 생성자**에 모아 `this(...)`로 재사용.
- **팩토리 메서드**로 의미 있는 이름 제공:

```csharp
public static Rectangle Square(int size) => new(size, size);
```

---

## 5. 메서드 오버로딩 (Method Overloading)

### 5.1 기본 규칙

- **시그니처** = 메서드 이름 + **매개변수 목록**(개수/순서/타입/`ref/out/in`).  
- **반환형만 달라서는** 오버로딩 **불가**.

```csharp
class Printer
{
    public void Print(string msg) => Console.WriteLine(msg);
    public void Print(int number) => Console.WriteLine($"숫자: {number}");
    public void Print(string msg, int repeat)
    {
        for (int i = 0; i < repeat; i++) Console.WriteLine(msg);
    }
}
```

### 5.2 `params`/옵셔널/이름있는 인자와 오버로딩

```csharp
void Log(string msg, string level = "INFO") { /*...*/ }
void Log(params string[] msgs) { /*...*/ }

Log("hi");            // 둘 다 후보 → 오버로드 해석: 더 구체적인 쪽(단일 매개변수)이 우선
Log("a","b","c");     // params 버전 선택
Log(level: "WARN", msg: "oops"); // 이름있는 인자 가능
```

**규칙 요약**

- **정확한 시그니처 일치** > **정수 승격/암시 변환** > **params**.
- 모호성(ambiguous)이 생기면 **컴파일 오류** → **명시적 캐스팅**이나 **이름 바꾸기**로 해결.

```csharp
void F(long x) { }
void F(double x) { }
// F(5); // 모호할 수 있음 → F(5L) 또는 F(5.0)로 명시
```

### 5.3 `ref/out/in`은 서로 다른 시그니처

```csharp
void Inc(ref int x) => x++;
bool TryParse(string s, out int x) { /*...*/ x=0; return false; }
int Sum(in ReadOnlySpan<int> s) { /*...*/ return 0; }
```

---

## 6. 메서드 오버라이딩 (Method Overriding)

### 6.1 기본

```csharp
class Animal
{
    public virtual void Speak() => Console.WriteLine("동물이 소리를 냅니다");
}

class Cat : Animal
{
    public override void Speak() => Console.WriteLine("야옹");
}

Animal a = new Cat();
a.Speak(); // "야옹" (런타임 다형성)
```

- **부모에 `virtual`**, **자식에 `override`** 필요.

### 6.2 `sealed`/`abstract`/`virtual` 조합

```csharp
abstract class Shape
{
    public abstract double Area(); // 파생형이 반드시 구현
    public virtual string Name => "Shape"; // 재정의 가능
}

class Circle : Shape
{
    public double R { get; }
    public Circle(double r) => R = r;

    public sealed override double Area() => Math.PI * R * R; // 더 이상 재정의 금지
    public override string Name => "Circle";
}
```

### 6.3 `new` 숨김 vs `override` 재정의

```csharp
class Foo { public void Print() => Console.WriteLine("Foo"); }
class Bar : Foo { public new void Print() => Console.WriteLine("Bar"); } // 숨김

Foo f = new Bar();
f.Print(); // Foo (정적 바인딩)

class Zoo { public virtual void Print() => Console.WriteLine("Zoo"); }
class Zoo2 : Zoo { public override void Print() => Console.WriteLine("Zoo2"); }

Zoo z = new Zoo2();
z.Print(); // Zoo2 (동적 바인딩)
```

- **숨김(new)**: 참조 타입에 따라 **정적 바인딩**.  
- **오버라이드**: **동적 바인딩**으로 다형성.

### 6.4 공변 반환(Covariant Return)

```csharp
abstract class Repo
{
    public abstract object Create();
}
class User { }
class UserRepo : Repo
{
    public override User Create() => new(); // object → User (공변 반환 허용)
}
```

---

## 7. `base`와 오버라이딩 활용 패턴

```csharp
class Logger
{
    public virtual void Write(string message) => Console.WriteLine(message);
}

class TimeLogger : Logger
{
    public override void Write(string message)
    {
        base.Write($"[{DateTime.UtcNow:O}] {message}");
    }
}
```

- **부모 기본동작 + 확장**을 자연스럽게 결합.

---

## 8. 실전 종합 예제 — 생성자 체이닝·검증·오버로딩·오버라이딩

```csharp
using System;

abstract class Account
{
    public string Id { get; }
    public decimal Balance { get; protected set; }

    protected Account(string id, decimal initial = 0m)
    {
        if (string.IsNullOrWhiteSpace(id)) throw new ArgumentException(nameof(id));
        if (initial < 0) throw new ArgumentOutOfRangeException(nameof(initial));
        Id = id; Balance = initial;
    }

    public virtual void Deposit(decimal amount)
    {
        if (amount <= 0) throw new ArgumentOutOfRangeException(nameof(amount));
        Balance += amount;
    }

    public abstract bool Withdraw(decimal amount); // 파생형 전략으로 위임
}

class SavingAccount : Account
{
    public decimal MinBalance { get; }
    public SavingAccount(string id) : this(id, 0m, 0m) { }
    public SavingAccount(string id, decimal initial) : this(id, initial, 0m) { }
    public SavingAccount(string id, decimal initial, decimal min)
        : base(id, initial)
    {
        if (min < 0) throw new ArgumentOutOfRangeException(nameof(min));
        MinBalance = min;
    }

    public override bool Withdraw(decimal amount)
    {
        if (amount <= 0) return false;
        if (Balance - amount < MinBalance) return false;
        Balance -= amount;
        return true;
    }

    public override void Deposit(decimal amount) // 확장: 보너스 지급
    {
        base.Deposit(amount);
        if (amount >= 1000m) Balance += 1m; // 간단 보너스
    }
}

class Program
{
    static void Main()
    {
        Account a = new SavingAccount("A-100", initial: 500m, min: 100m);
        a.Deposit(1500m);
        Console.WriteLine(a.Balance); // 2001
        Console.WriteLine(a.Withdraw(1900m)); // false (최소 잔액 미달)
        Console.WriteLine(a.Withdraw(1800m)); // true
        Console.WriteLine(a.Balance); // 201
    }
}
```

---

## 9. 오버로드 해석(Overload Resolution) 미세 규칙 요약

1. **정확 일치**가 최우선.  
2. **암시적 수치 변환**(int→long, float→double 등)은 가능하나, 모호하면 오류.  
3. `ref/out/in` 일치 필요(다르면 **다른 시그니처**).  
4. `params`는 **마지막 후보**.  
5. **nullable**과 **대리자/람다**는 예상 타입에 따라 추론되며, **더 구체적인** 오버로드가 선택됨.  
6. 이름있는 인자와 기본값이 섞이면 **한쪽으로 일관되게** 사용(가독성).

```csharp
void F(int x) {}
void F(long x) {}
// F(5); // 컴파일러는 int버전 선택
```

---

## 10. 설계 팁과 체크리스트

- **생성자**
  - 유효성 검증을 **항상** 포함.  
  - **한 생성자에 검증을 모으고** 나머지는 `this(...)`.  
  - 외부에서 직접 생성시키지 않으려면 **private/protected 생성자 + 정적 팩토리**.

- **this/base**
  - `this(...)`·`base(...)`는 **첫 줄**.  
  - 재정의 메서드에서 **`base` 호출을 강제할지** 설계(문서화).

- **오버로딩**
  - API가 모호해지면 오버로드를 **줄이고** 다른 이름으로 명확화.  
  - `params`와 기본값 혼용 시 **호출 예측성** 점검.

- **오버라이딩**
  - 기본은 **sealed** 클래스, 필요한 지점만 `virtual`.  
  - `new` 숨김은 **예외적**으로만 사용(다형성 기대를 깨뜨림).  
  - 공변 반환 활용으로 반환 타입을 **구체화**.

---

## 11. 수학적 직관 — 오버로드/오버라이드 선택

오버로드 해석은 **정적(컴파일 타임)** 최적 일치 문제이며, 타입 변환 비용 함수 \(c\)가 있을 때

$$
\text{Pick} = \arg\min_{f \in \mathcal{F}} \sum_{i} c(\text{arg}_i \Rightarrow \text{param}_i^f)
$$

오버라이딩 선택은 **동적(런타임)** 디스패치로, 실제 인스턴스 타입 \(T\)의 **가상 테이블**에서 일치 항목을 고릅니다.

---

## 12. 연습 문제

1) `Order`  
   - 생성자 오버로딩: `(id)`, `(id, items)`, `(id, items, coupon)` → 모든 검증은 **하나의 생성자**에 집중 후 `this(...)`.  
2) `Shape`  
   - `Area()`는 `abstract`, `Name`은 `virtual`, `Circle.Area()`는 **sealed override**로 고정.  
3) 오버로드 해석  
   - `void G(int)`, `void G(long)`, `void G(double)`, `void G(params int[])` 조합에서 호출식 `G(5)`, `G(5L)`, `G(5.0)`, `G()`의 선택 결과 설명.  
4) `Logger`  
   - `Write(string)`, `Write(ReadOnlySpan<char>)`를 둘 다 제공하고 **큰 문자열에서 할당을 줄이는 호출 전략** 설계.  
5) 확장 메서드  
   - `this IEnumerable<int>`에 `SumEven()` 구현 후, 오버로드로 `SumEven(this ReadOnlySpan<int>)`도 제공하여 **할당 없는 경로** 제공.

---

## 13. 요약

- **생성자**: 유효 상태 보장, `this(...)`/`base(...)` 규칙 숙지.  
- **this/base**: 현재/부모 맥락 명시, 체이닝·부모호출에 필수.  
- **오버로딩**: 시그니처 차이로 다형적 편의 제공, 모호성 피하기.  
- **오버라이딩**: 런타임 다형성, `virtual/override/sealed`로 확장 제어.  
- **안정적 API**는 모호성을 최소화하고, 불변조건을 생성자/메서드 경로에서 강제한다.