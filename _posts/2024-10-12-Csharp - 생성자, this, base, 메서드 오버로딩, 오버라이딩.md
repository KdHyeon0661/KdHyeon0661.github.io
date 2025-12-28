---
layout: post
title: C# - 생성자, this, base, 메서드 오버로딩, 오버라이딩
date: 2024-10-12 19:20:23 +0900
category: Csharp
---
# 생성자, this/base 키워드, 메서드 오버로딩, 메서드 오버라이딩

## 개요: 객체의 생성과 행동 확장

객체 지향 프로그래밍에서 객체가 올바르게 생성되고, 상속 계층에서 적절하게 동작하도록 하는 것은 핵심적인 개념입니다. 생성자(Constructor)는 객체를 초기화하고, `this`와 `base` 키워드는 현재 객체와 부모 객체를 참조하며, 메서드 오버로딩(Overloading)과 오버라이딩(Overriding)은 메서드의 다양한 형태와 동적 동작을 가능하게 합니다.

---

## 1. 생성자(Constructor): 객체의 초기 상태 보장

생성자는 클래스의 새 인스턴스를 만들 때 호출되는 특별한 메서드로, 객체가 유효한 상태로 시작하도록 보장하는 역할을 합니다.

### 기본 생성자

```csharp
public class Person
{
    public string Name { get; set; }
    
    // 기본 생성자 (매개변수 없음)
    public Person()
    {
        Name = "Unknown";
    }
}

var person = new Person();
Console.WriteLine(person.Name); // "Unknown"
```

### 매개변수가 있는 생성자와 유효성 검사

```csharp
public class BankAccount
{
    public string AccountNumber { get; }
    public decimal Balance { get; private set; }
    
    // 매개변수가 있는 생성자
    public BankAccount(string accountNumber, decimal initialBalance)
    {
        // 유효성 검사: 객체가 올바른 상태로 시작하도록 보장
        if (string.IsNullOrWhiteSpace(accountNumber))
            throw new ArgumentException("계좌번호는 필수입니다.", nameof(accountNumber));
        
        if (initialBalance < 0)
            throw new ArgumentOutOfRangeException(nameof(initialBalance), "초기 잔액은 0 이상이어야 합니다.");
        
        AccountNumber = accountNumber;
        Balance = initialBalance;
    }
}

// 유효한 객체 생성
var account = new BankAccount("123-456-789", 1000m);
// var invalidAccount = new BankAccount("", -100); // 예외 발생
```

생성자의 가장 중요한 역할은 **객체가 항상 유효한 상태로만 존재할 수 있도록 보장**하는 것입니다. 이를 위해 필요한 모든 검증을 생성자에서 수행해야 합니다.

### 생성자 오버로딩(Constructor Overloading)

하나의 클래스에 여러 생성자를 정의하여 다양한 방식으로 객체를 초기화할 수 있습니다.

```csharp
public class Rectangle
{
    public int Width { get; }
    public int Height { get; }
    
    // 기본 생성자: 1x1 사각형
    public Rectangle() : this(1, 1) { }
    
    // 정사각형 생성자
    public Rectangle(int size) : this(size, size) { }
    
    // 주요 생성자: 모든 검증 로직을 여기에 집중
    public Rectangle(int width, int height)
    {
        if (width <= 0 || height <= 0)
            throw new ArgumentException("너비와 높이는 양수여야 합니다.");
        
        Width = width;
        Height = height;
    }
}

// 다양한 방식으로 객체 생성
var defaultRect = new Rectangle();      // 1x1
var square = new Rectangle(5);          // 5x5
var rectangle = new Rectangle(3, 4);    // 3x4
```

### 생성자 체이닝: `this()` 사용

위 예제에서 `: this(...)` 구문은 **생성자 체이닝**을 나타냅니다. 한 생성자가 다른 생성자를 호출하여 코드 중복을 피하고, 검증 로직을 한 곳에 집중시킬 수 있습니다.

```csharp
public Rectangle() : this(1, 1) { }
// 위 코드는 아래와 동일합니다:
// public Rectangle()
// {
//     Width = 1;
//     Height = 1;
// }
```

**중요 규칙**: 생성자 체이닝(`this(...)`) 또는 부모 생성자 호출(`base(...)`)은 반드시 생성자의 첫 번째 문장으로만 사용할 수 있으며, 둘 중 하나만 선택할 수 있습니다.

### 상속과 부모 생성자 호출: `base()`

파생 클래스(자식 클래스)를 생성할 때는 부모 클래스의 생성자를 호출해야 합니다.

```csharp
public class Animal
{
    public string Name { get; }
    
    public Animal(string name)
    {
        if (string.IsNullOrWhiteSpace(name))
            throw new ArgumentException("이름은 필수입니다.", nameof(name));
        
        Name = name;
    }
}

public class Dog : Animal
{
    public string Breed { get; }
    
    // base()를 사용하여 부모 생성자 호출
    public Dog(string name, string breed) : base(name)  // Animal(name) 호출
    {
        Breed = breed ?? "Unknown";
    }
}

var dog = new Dog("바둑이", "진돗개");
Console.WriteLine($"{dog.Name} ({dog.Breed})"); // "바둑이 (진돗개)"
```

**실행 순서**: 부모 생성자 → 자식 생성자의 순서로 실행됩니다. 이는 부모 클래스의 초기화가 완료된 후에 자식 클래스의 초기화가 이루어져야 하기 때문입니다.

### 정적 생성자(Static Constructor)

정적 생성자는 클래스가 처음 사용되기 전에 한 번만 실행되어 정적 필드를 초기화합니다.

```csharp
public class Configuration
{
    public static readonly string AppName;
    public static readonly DateTime StartupTime;
    
    // 정적 생성자
    static Configuration()
    {
        AppName = "MyApplication";
        StartupTime = DateTime.UtcNow;
        Console.WriteLine("정적 생성자가 실행되었습니다.");
    }
}

// Configuration 클래스에 처음 접근할 때 정적 생성자가 실행됨
Console.WriteLine(Configuration.AppName);
```

---

## 2. `this` 키워드: 현재 인스턴스 참조

`this` 키워드는 현재 인스턴스를 참조하며, 여러 상황에서 유용하게 사용됩니다.

### 필드와 매개변수 이름 구분

```csharp
public class Person
{
    private string name;
    
    public Person(string name)
    {
        // this.name은 인스턴스의 필드를,
        // name은 매개변수를 가리킵니다.
        this.name = name;
    }
}
```

### 생성자 체이닝

```csharp
public class Product
{
    public string Name { get; }
    public decimal Price { get; }
    public int Stock { get; }
    
    public Product(string name) : this(name, 0, 0) { }
    
    public Product(string name, decimal price) : this(name, price, 0) { }
    
    public Product(string name, decimal price, int stock)
    {
        Name = name;
        Price = price;
        Stock = stock;
    }
}
```

### 현재 객체 반환 (메서드 체이닝 지원)

```csharp
public class StringBuilder
{
    private string _value = "";
    
    public StringBuilder Append(string text)
    {
        _value += text;
        return this; // 현재 인스턴스 반환
    }
    
    public override string ToString() => _value;
}

// 메서드 체이닝 가능
var result = new StringBuilder()
    .Append("Hello, ")
    .Append("World!")
    .ToString();
```

---

## 3. `base` 키워드: 부모 클래스 멤버 접근

`base` 키워드는 부모 클래스의 멤버에 접근할 때 사용됩니다.

### 부모 생성자 호출

```csharp
public class Vehicle
{
    public string Make { get; }
    
    public Vehicle(string make)
    {
        Make = make;
    }
}

public class Car : Vehicle
{
    public string Model { get; }
    
    public Car(string make, string model) : base(make) // Vehicle 생성자 호출
    {
        Model = model;
    }
}
```

### 부모 메서드 호출 (오버라이딩 시)

```csharp
public class Logger
{
    public virtual void Log(string message)
    {
        Console.WriteLine($"LOG: {message}");
    }
}

public class TimestampLogger : Logger
{
    public override void Log(string message)
    {
        // 부모 클래스의 Log 메서드 호출
        base.Log($"[{DateTime.Now:yyyy-MM-dd HH:mm:ss}] {message}");
    }
}

var logger = new TimestampLogger();
logger.Log("애플리케이션 시작됨");
// 출력: LOG: [2024-01-15 10:30:00] 애플리케이션 시작됨
```

### 부모의 숨겨진 멤버 접근

```csharp
public class BaseClass
{
    public void Display()
    {
        Console.WriteLine("BaseClass.Display");
    }
}

public class DerivedClass : BaseClass
{
    public new void Display()  // new 키워드로 숨김
    {
        Console.WriteLine("DerivedClass.Display");
        base.Display();  // 부모 클래스의 Display 메서드 호출
    }
}
```

---

## 4. 메서드 오버로딩(Method Overloading): 다양한 입력 처리

메서드 오버로딩은 같은 이름의 메서드를 매개변수의 타입, 개수, 순서를 다르게 하여 여러 개 정의하는 것을 말합니다. 이는 사용자에게 직관적이고 일관된 API를 제공하는 데 유용합니다.

### 기본 오버로딩 예제

```csharp
public class Calculator
{
    // 정수 덧셈
    public int Add(int a, int b)
    {
        return a + b;
    }
    
    // 실수 덧셈 (매개변수 타입이 다름)
    public double Add(double a, double b)
    {
        return a + b;
    }
    
    // 세 개의 정수 덧셈 (매개변수 개수가 다름)
    public int Add(int a, int b, int c)
    {
        return a + b + c;
    }
    
    // 가변 길이 매개변수
    public int Add(params int[] numbers)
    {
        int sum = 0;
        foreach (var num in numbers)
        {
            sum += num;
        }
        return sum;
    }
}

var calc = new Calculator();
Console.WriteLine(calc.Add(5, 3));        // 8 (int 버전)
Console.WriteLine(calc.Add(2.5, 3.7));    // 6.2 (double 버전)
Console.WriteLine(calc.Add(1, 2, 3));     // 6 (3개 매개변수 버전)
Console.WriteLine(calc.Add(1, 2, 3, 4, 5)); // 15 (params 버전)
```

### 오버로딩 규칙과 주의사항

1. **오버로딩은 메서드 시그니처(이름 + 매개변수 목록)가 달라야 합니다.**
2. **반환 타입만 다른 경우는 오버로딩으로 간주되지 않습니다.**

```csharp
// 컴파일 오류: 반환 타입만 다른 메서드는 오버로딩 불가
// public int GetValue() { return 1; }
// public string GetValue() { return "1"; }
```

3. **`ref`, `out`, `in` 한정자는 시그니처의 일부로 간주됩니다.**

```csharp
public void Process(ref int x) { }
public void Process(out int x) { x = 0; } // 가능: ref와 out은 다른 시그니처
```

4. **컴파일러는 가장 구체적인(적합한) 오버로드를 선택합니다.**

```csharp
public class Example
{
    public void Print(int x) => Console.WriteLine($"int: {x}");
    public void Print(long x) => Console.WriteLine($"long: {x}");
    public void Print(double x) => Console.WriteLine($"double: {x}");
}

var ex = new Example();
ex.Print(5);    // "int: 5" (가장 구체적인 int 버전 선택)
ex.Print(5L);   // "long: 5" (long 리터럴)
ex.Print(5.0);  // "double: 5" (double 리터럴)
```

### 선택적 매개변수와 오버로딩

```csharp
public class MessageService
{
    // 기본값을 가진 매개변수
    public void Send(string message, string priority = "Normal")
    {
        Console.WriteLine($"[{priority}] {message}");
    }
    
    // 오버로드된 버전
    public void Send(string message, int retryCount)
    {
        for (int i = 0; i < retryCount; i++)
        {
            Console.WriteLine($"시도 {i + 1}: {message}");
        }
    }
}

var service = new MessageService();
service.Send("Hello");               // [Normal] Hello
service.Send("Hello", "High");       // [High] Hello
service.Send("Hello", 3);            // 시도 1: Hello \n 시도 2: Hello \n 시도 3: Hello
```

---

## 5. 메서드 오버라이딩(Method Overriding): 런타임 다형성

메서드 오버라이딩은 상속 관계에서 부모 클래스의 메서드를 자식 클래스에서 재정의하는 것을 말합니다. 이는 객체 지향 프로그래밍의 핵심 개념인 **다형성(Polymorphism)** 을 구현하는 방법입니다.

### 기본 오버라이딩

```csharp
public class Shape
{
    // virtual 키워드: 이 메서드는 파생 클래스에서 재정의될 수 있음
    public virtual double CalculateArea()
    {
        return 0;
    }
    
    public virtual string GetDescription()
    {
        return "일반 도형";
    }
}

public class Circle : Shape
{
    public double Radius { get; }
    
    public Circle(double radius)
    {
        Radius = radius;
    }
    
    // override 키워드: 부모 클래스의 메서드를 재정의
    public override double CalculateArea()
    {
        return Math.PI * Radius * Radius;
    }
    
    public override string GetDescription()
    {
        return $"반지름 {Radius}인 원";
    }
}

public class Rectangle : Shape
{
    public double Width { get; }
    public double Height { get; }
    
    public Rectangle(double width, double height)
    {
        Width = width;
        Height = height;
    }
    
    public override double CalculateArea()
    {
        return Width * Height;
    }
    
    public override string GetDescription()
    {
        return $"{Width}x{Height} 크기의 사각형";
    }
}

// 다형성의 힘: 동일한 Shape 타입으로 다양한 도형 처리
Shape[] shapes = new Shape[]
{
    new Circle(5),
    new Rectangle(3, 4),
    new Circle(2.5)
};

foreach (var shape in shapes)
{
    Console.WriteLine($"{shape.GetDescription()} - 면적: {shape.CalculateArea():F2}");
}
// 출력:
// 반지름 5인 원 - 면적: 78.54
// 3x4 크기의 사각형 - 면적: 12.00
// 반지름 2.5인 원 - 면적: 19.63
```

### `abstract` 메서드와 `sealed` 오버라이드

```csharp
public abstract class Animal
{
    // 추상 메서드: 구현이 없으며, 파생 클래스에서 반드시 구현해야 함
    public abstract string MakeSound();
    
    // virtual 메서드: 기본 구현을 제공하며, 필요시 재정의 가능
    public virtual string GetHabitat()
    {
        return "알 수 없는 서식지";
    }
}

public class Dog : Animal
{
    // 추상 메서드 구현
    public override string MakeSound()
    {
        return "멍멍!";
    }
    
    // virtual 메서드 재정의
    public override string GetHabitat()
    {
        return "주택, 농장, 야외";
    }
}

public class Cat : Animal
{
    public override string MakeSound()
    {
        return "야옹~";
    }
    
    // sealed override: 이 메서드는 더 이상 재정의될 수 없음
    public sealed override string GetHabitat()
    {
        return "실내, 야외";
    }
}

// Cat을 상속받는 클래스에서 GetHabitat()는 재정의할 수 없음
public class PersianCat : Cat
{
    public override string MakeSound()
    {
        return "미야옹~";
    }
    
    // 다음 코드는 컴파일 오류: GetHabitat()는 sealed로 봉인됨
    // public override string GetHabitat() { return "실내"; }
}
```

### `new` 키워드: 메서드 숨기기

`new` 키워드는 메서드를 재정의하는 것이 아니라, 부모 클래스의 메서드를 **숨기는** 역할을 합니다. 이는 다형성과는 다른 동작을 합니다.

```csharp
public class BaseClass
{
    public void Show()
    {
        Console.WriteLine("BaseClass.Show()");
    }
}

public class DerivedClass : BaseClass
{
    // new 키워드로 부모 메서드 숨기기
    public new void Show()
    {
        Console.WriteLine("DerivedClass.Show()");
    }
}

BaseClass obj1 = new DerivedClass();
obj1.Show();  // "BaseClass.Show()" - 컴파일 타임 타입에 따라 호출

DerivedClass obj2 = new DerivedClass();
obj2.Show();  // "DerivedClass.Show()"
```

**`override` vs `new`**:
- `override`: 동적 바인딩(런타임에 실제 객체 타입에 따라 메서드 호출)
- `new`: 정적 바인딩(컴파일 타임에 변수 타입에 따라 메서드 호출)

---

## 6. 종합 예제: 은행 계좌 시스템

```csharp
using System;

// 기본 계좌 클래스
public abstract class Account
{
    public string AccountNumber { get; }
    public string OwnerName { get; }
    public decimal Balance { get; protected set; }
    
    // 생성자: 기본 계좌 정보 초기화
    protected Account(string accountNumber, string ownerName, decimal initialBalance = 0)
    {
        if (string.IsNullOrWhiteSpace(accountNumber))
            throw new ArgumentException("계좌번호는 필수입니다.", nameof(accountNumber));
        
        if (string.IsNullOrWhiteSpace(ownerName))
            throw new ArgumentException("예금주명은 필수입니다.", nameof(ownerName));
        
        if (initialBalance < 0)
            throw new ArgumentException("초기 잔액은 0 이상이어야 합니다.", nameof(initialBalance));
        
        AccountNumber = accountNumber;
        OwnerName = ownerName;
        Balance = initialBalance;
    }
    
    // 입금 메서드 (오버로딩 예정)
    public virtual void Deposit(decimal amount)
    {
        if (amount <= 0)
            throw new ArgumentException("입금액은 0보다 커야 합니다.", nameof(amount));
        
        Balance += amount;
        Console.WriteLine($"{amount:C} 입금 완료. 현재 잔액: {Balance:C}");
    }
    
    // 추상 메서드: 출금 로직은 계좌 종류에 따라 다름
    public abstract bool Withdraw(decimal amount);
    
    // 계좌 정보 출력 (가상 메서드)
    public virtual void DisplayInfo()
    {
        Console.WriteLine($"계좌번호: {AccountNumber}");
        Console.WriteLine($"예금주: {OwnerName}");
        Console.WriteLine($"잔액: {Balance:C}");
    }
}

// 저축 계좌
public class SavingsAccount : Account
{
    public decimal InterestRate { get; } // 연 이율
    
    // 생성자 체이닝
    public SavingsAccount(string accountNumber, string ownerName) 
        : this(accountNumber, ownerName, 0, 0.02m) { }
    
    public SavingsAccount(string accountNumber, string ownerName, decimal initialBalance)
        : this(accountNumber, ownerName, initialBalance, 0.02m) { }
    
    public SavingsAccount(string accountNumber, string ownerName, decimal initialBalance, decimal interestRate)
        : base(accountNumber, ownerName, initialBalance)
    {
        if (interestRate < 0 || interestRate > 1)
            throw new ArgumentException("이율은 0과 1 사이여야 합니다.", nameof(interestRate));
        
        InterestRate = interestRate;
    }
    
    // 입금 메서드 오버라이딩 (이자 추가)
    public override void Deposit(decimal amount)
    {
        base.Deposit(amount); // 부모 클래스의 Deposit 호출
        
        // 저축 계좌 특별 혜택: 큰 금액 입금 시 보너스
        if (amount >= 1000000)
        {
            decimal bonus = amount * 0.001m; // 0.1% 보너스
            Balance += bonus;
            Console.WriteLine($"대량 입금 보너스 {bonus:C} 적립!");
        }
    }
    
    // 출금 메서드 구현 (최소 잔액 유지)
    public override bool Withdraw(decimal amount)
    {
        if (amount <= 0)
            return false;
        
        // 저축 계좌는 최소 10,000원 유지
        decimal minimumBalance = 10000;
        
        if (Balance - amount < minimumBalance)
        {
            Console.WriteLine($"출금 실패: 최소 잔액 {minimumBalance:C}을 유지해야 합니다.");
            return false;
        }
        
        Balance -= amount;
        Console.WriteLine($"{amount:C} 출금 완료. 현재 잔액: {Balance:C}");
        return true;
    }
    
    // 계좌 정보 출력 오버라이딩
    public override void DisplayInfo()
    {
        base.DisplayInfo(); // 부모 클래스의 DisplayInfo 호출
        Console.WriteLine($"이율: {InterestRate:P2}");
        Console.WriteLine($"최소 유지 잔액: 10,000원");
    }
    
    // 이자 계산 메서드 (오버로딩)
    public decimal CalculateInterest() => Balance * InterestRate;
    
    public decimal CalculateInterest(int years) => Balance * (decimal)Math.Pow(1 + (double)InterestRate, years) - Balance;
}

// 당좌 계좌
public class CheckingAccount : Account
{
    public decimal OverdraftLimit { get; } // 마이너스 통장 한도
    
    public CheckingAccount(string accountNumber, string ownerName, decimal overdraftLimit = 500000)
        : base(accountNumber, ownerName)
    {
        OverdraftLimit = overdraftLimit;
    }
    
    // 출금 메서드 구현 (마이너스 통장 지원)
    public override bool Withdraw(decimal amount)
    {
        if (amount <= 0)
            return false;
        
        if (Balance - amount < -OverdraftLimit)
        {
            Console.WriteLine($"출금 실패: 한도 초과 (최대 {OverdraftLimit:C}까지 마이너스 가능)");
            return false;
        }
        
        Balance -= amount;
        Console.WriteLine($"{amount:C} 출금 완료. 현재 잔액: {Balance:C}");
        
        if (Balance < 0)
        {
            Console.WriteLine("경고: 마이너스 잔액 상태입니다.");
        }
        
        return true;
    }
    
    // 계좌 정보 출력 오버라이딩
    public override void DisplayInfo()
    {
        base.DisplayInfo();
        Console.WriteLine($"마이너스 통장 한도: {OverdraftLimit:C}");
    }
}

class Program
{
    static void Main()
    {
        Console.WriteLine("=== 은행 계좌 시스템 ===\n");
        
        // 저축 계좌 테스트
        Console.WriteLine("1. 저축 계좌 생성 및 테스트");
        SavingsAccount savings = new SavingsAccount("SAV-001", "홍길동", 50000, 0.03m);
        savings.DisplayInfo();
        Console.WriteLine();
        
        savings.Deposit(20000);
        savings.Deposit(1500000); // 대량 입금 보너스 테스트
        savings.Withdraw(10000);
        savings.Withdraw(60000); // 최소 잔액 제한 테스트
        
        Console.WriteLine($"\n1년 후 예상 이자: {savings.CalculateInterest():C}");
        Console.WriteLine($"5년 후 예상 이자: {savings.CalculateInterest(5):C}");
        
        Console.WriteLine("\n" + new string('-', 40) + "\n");
        
        // 당좌 계좌 테스트
        Console.WriteLine("2. 당좌 계좌 생성 및 테스트");
        CheckingAccount checking = new CheckingAccount("CHK-001", "김철수", 1000000);
        checking.DisplayInfo();
        Console.WriteLine();
        
        checking.Deposit(300000);
        checking.Withdraw(400000); // 마이너스 통장 사용
        checking.Withdraw(1000000); // 한도 초과 테스트
    }
}
```

---

## 결론: 적절한 도구 선택의 중요성

생성자, `this`/`base` 키워드, 메서드 오버로딩과 오버라이딩은 C#의 객체 지향 기능을 완전히 활용하기 위한 필수 도구입니다. 각 개념은 특정한 목적과 상황에 맞게 사용되어야 합니다:

1. **생성자**는 객체의 **유효한 초기 상태를 보장**하는 책임을 가집니다. 생성자 오버로딩과 체이닝을 활용하여 다양한 초기화 옵션을 제공하면서도 검증 로직을 중앙화하세요.

2. **`this` 키워드**는 현재 객체를 명시적으로 참조할 때, 특히 이름 충돌을 해결하거나 생성자 체이닝을 구현할 때 사용됩니다.

3. **`base` 키워드**는 상속 계층에서 부모 클래스의 기능을 재사용하거나 확장할 때 필수적입니다. 부모의 구현을 호출하여 중복을 피하고 일관성을 유지하세요.

4. **메서드 오버로딩**은 **편의성과 API 일관성**을 제공합니다. 같은 개념의 작업에 대해 다양한 입력 형식을 지원할 때 사용하되, 사용자가 혼란스러워하지 않도록 주의하세요.

5. **메서드 오버라이딩**은 **다형성과 확장성**의 핵심입니다. 부모 클래스에서 `virtual`이나 `abstract`로 확장 지점을 명확히 표시하고, 자식 클래스에서 `override`로 구체적인 동작을 구현하세요. `new` 키워드는 일반적으로 피해야 하며, 명시적인 설계 결정이 필요할 때만 사용하세요.

이러한 개념들을 적절히 조합하면 유연하면서도 견고한 클래스 계층 구조를 설계할 수 있습니다. 객체가 올바르게 생성되고, 명확하게 행동하며, 필요에 따라 확장될 수 있는 시스템을 구축하는 데 이 지식이 핵심이 될 것입니다.