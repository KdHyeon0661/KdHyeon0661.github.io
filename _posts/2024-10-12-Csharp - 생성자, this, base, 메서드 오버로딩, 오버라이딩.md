---
layout: post
title: C# - 생성자, this, base, 메서드 오버로딩, 오버라이딩
date: 2024-10-12 19:20:23 +0900
category: Csharp
---
# C# 객체의 탄생과 다형성: 생성자부터 오버라이딩까지

객체 지향 프로그래밍(OOP)에서 객체는 스스로 올바른 상태를 유지하며 태어나야 하고, 상속이라는 거대한 가계도 속에서 자신의 역할을 정확히 수행해야 합니다. C#은 이를 위해 객체의 탄생을 책임지는 생성자(Constructor), 자신과 부모를 가리키는 `this` 및 `base` 키워드, 그리고 메서드의 형태와 동작을 확장하는 오버로딩(Overloading)과 오버라이딩(Overriding)이라는 강력한 도구들을 제공합니다.

이 글에서는 단순히 문법을 나열하는 것을 넘어, 객체가 메모리에 생성되는 순서, 가상 메서드 테이블(vtable)의 내부 동작 원리, 그리고 실무 면접에서 단골로 등장하는 치명적인 설계 함정들까지 깊이 있게 파헤쳐 보겠습니다.

## 생성자: 객체의 무결성을 보장하는 첫 관문

생성자는 클래스의 새로운 인스턴스(객체)가 메모리에 할당될 때 가장 먼저 호출되는 특수한 메서드입니다. 생성자의 가장 핵심적인 존재 이유는 **객체가 논리적으로 완벽하게 유효한 상태로만 세상에 존재할 수 있도록 보장**하는 것입니다.

### 기본 생성자와 유효성 검사

아무런 매개변수를 받지 않는 기본 생성자부터, 필수 데이터를 강제하는 매개변수 생성자까지 다양하게 정의할 수 있습니다. 객체가 불안정한 상태(예: 계좌번호가 비어있거나, 잔액이 음수인 상태)로 생성되는 것을 막으려면 생성자 내부에서 철저한 유효성 검사를 수행해야 합니다.

```csharp
public class BankAccount
{
    public string AccountNumber { get; }
    public decimal Balance { get; private set; }
    
    // 매개변수가 있는 생성자를 통해 필수 값을 강제합니다.
    public BankAccount(string accountNumber, decimal initialBalance)
    {
        // 객체가 잘못된 상태로 생성되는 것을 원천 차단합니다.
        if (string.IsNullOrWhiteSpace(accountNumber))
            throw new ArgumentException("계좌번호는 필수입니다.", nameof(accountNumber));
        
        if (initialBalance < 0)
            throw new ArgumentOutOfRangeException(nameof(initialBalance), "초기 잔액은 0 이상이어야 합니다.");
        
        AccountNumber = accountNumber;
        Balance = initialBalance;
    }
}

```

이렇게 설계된 클래스는 시스템 어디에서 사용되더라도 객체가 존재하기만 한다면 그 안의 데이터는 무조건 신뢰할 수 있다는 강력한 보증을 제공합니다.

### 생성자 오버로딩과 this 체이닝

객체를 생성하는 방법은 상황에 따라 여러 가지가 필요할 수 있습니다. 매개변수의 개수나 타입을 다르게 하여 여러 개의 생성자를 제공하는 것을 생성자 오버로딩이라고 합니다.

이때 발생하는 코드의 중복을 막기 위해 `this` 키워드를 사용한 **생성자 체이닝(Constructor Chaining)** 기법을 활용합니다.

```csharp
public class Rectangle
{
    public int Width { get; }
    public int Height { get; }
    
    // 기본 생성자는 매개변수 2개짜리 생성자에게 초기화를 위임합니다.
    public Rectangle() : this(1, 1) { }
    
    // 정사각형을 위한 생성자 역시 위임을 활용합니다.
    public Rectangle(int size) : this(size, size) { }
    
    // 모든 검증 로직과 데이터 할당은 오직 이 '마스터 생성자' 한 곳에서만 이루어집니다.
    public Rectangle(int width, int height)
    {
        if (width <= 0 || height <= 0)
            throw new ArgumentException("너비와 높이는 양수여야 합니다.");
        
        Width = width;
        Height = height;
    }
}

```

이 패턴을 사용하면 향후 초기화 로직이나 유효성 검사 기준이 변경되더라도 단 한 곳의 마스터 생성자만 수정하면 되므로 유지보수성이 극대화됩니다.

### 상속 계층에서의 생성자 호출 (base 키워드)

파생 클래스(자식 클래스)가 생성될 때는 반드시 기반 클래스(부모 클래스)의 생성자가 먼저 호출되어야 합니다. 부모가 가진 뼈대가 먼저 메모리에 완성되어야 자식이 살을 붙일 수 있기 때문입니다. 이를 위해 `base` 키워드를 사용하여 부모의 생성자에게 필요한 매개변수를 전달합니다.

```csharp
public class Animal
{
    public string Name { get; }
    
    public Animal(string name)
    {
        if (string.IsNullOrWhiteSpace(name))
            throw new ArgumentException("이름은 필수입니다.");
        Name = name;
    }
}

public class Dog : Animal
{
    public string Breed { get; }
    
    // base(name)을 통해 부모인 Animal의 생성자를 먼저 호출하여 Name을 초기화합니다.
    public Dog(string name, string breed) : base(name)
    {
        Breed = breed ?? "Unknown";
    }
}

```

만약 부모 클래스에 매개변수가 없는 기본 생성자가 존재한다면, 컴파일러가 알아서 자식 생성자 뒤에 `: base()`를 삽입해 줍니다. 하지만 부모 클래스에 매개변수가 있는 생성자만 존재한다면, 개발자가 반드시 명시적으로 `base(...)`를 호출해 주어야 컴파일 오류가 발생하지 않습니다.

### 생성자 내 가상 메서드 호출의 치명적 위험성

C# 아키텍처 설계에서 가장 빈번하게 발생하는 실수이자 기술 면접의 단골 질문입니다. **생성자 내부에서는 절대 `virtual`이나 `abstract` 등 오버라이딩이 가능한 가상 메서드를 호출해서는 안 됩니다.**

그 이유는 객체의 생성 순서와 깊은 관련이 있습니다. 부모 클래스의 생성자가 실행되는 시점에는 자식 클래스의 생성자 본문이 아직 단 한 줄도 실행되지 않은 상태입니다. 이때 부모 생성자에서 가상 메서드를 호출하면, 다형성에 의해 부모의 코드가 아닌 자식 클래스에서 재정의(Override)한 메서드가 뜬금없이 실행되어 버립니다.

```csharp
public class Base
{
    public Base()
    {
        // 🔴 매우 위험한 설계: 부모 생성자에서 가상 메서드 호출
        Initialize(); 
    }
    
    protected virtual void Initialize() 
    { 
        Console.WriteLine("부모의 기본 초기화"); 
    }
}

public class Derived : Base
{
    private string _importantData;
    
    public Derived(string data)
    {
        // 부모 생성자가 끝난 뒤에야 이 코드가 실행됩니다.
        _importantData = data; 
    }
    
    protected override void Initialize()
    {
        // 부모 생성자에 의해 강제로 이 곳이 먼저 실행됩니다.
        // 이때 _importantData는 값이 할당되지 않아 null인 상태입니다!
        Console.WriteLine($"자식의 데이터 길이: {_importantData.Length}"); // NullReferenceException 발생!
    }
}

```

이처럼 자식 객체의 내부 필드가 세팅되기도 전에 자식의 메서드가 호출되면서 심각한 런타임 에러를 유발하게 됩니다. 초기화 로직이 복잡하다면 초기화 전용 메서드를 따로 분리하는 것이 바람직합니다.

### 정적 생성자 (Static Constructor)

클래스 자체에 소속된 정적(static) 필드를 초기화할 때는 정적 생성자를 사용합니다.

정적 생성자는 개발자가 코드에서 직접 호출할 수 없으며, 매개변수도 가질 수 없습니다. CLR(Common Language Runtime)이 프로그램 실행 중 해당 클래스가 최초로 사용되는 순간을 감지하여, 그 직전에 단 한 번만 스레드 안전(Thread-safe)하게 자동으로 호출해 줍니다.

```csharp
public class DatabaseConfig
{
    public static readonly string ConnectionString;
    
    // 접근 제한자(public 등)를 붙일 수 없습니다.
    static DatabaseConfig()
    {
        ConnectionString = "Server=myServer;Database=myDB;";
        Console.WriteLine("정적 생성자가 단 한 번 실행되었습니다.");
    }
}

```

정적 생성자 내부에서 예외가 터지면 `TypeInitializationException`이라는 치명적인 예외로 감싸져 프로그램에 던져지며, 이후 해당 애플리케이션 도메인이 종료될 때까지 그 클래스는 영원히 사용할 수 없는 먹통 상태가 되므로 예외 처리에 각별히 주의해야 합니다.

## this와 base 키워드의 활용

`this`와 `base`는 객체 지향 프로그래밍에서 컨텍스트(문맥)를 명확히 지정해 주는 내비게이션 역할을 합니다.

`this` 키워드는 메모리에 생성된 **자기 자신(현재 인스턴스)**을 가리킵니다. 주로 매개변수 이름과 클래스의 필드 이름이 같을 때 이를 구분하기 위해 사용하거나, 앞서 살펴본 생성자 체이닝, 혹은 메서드 체이닝(Builder 패턴 등)을 구현하여 자기 자신을 반환할 때 요긴하게 쓰입니다.

`base` 키워드는 **부모 클래스의 멤버**에 접근할 때 사용합니다. 자식 클래스에서 부모의 메서드를 오버라이딩하여 내용을 완전히 덮어쓰지 않고, 부모의 원래 로직을 실행한 뒤 그 앞뒤로 살을 붙이고 싶을 때 `base.MethodName()` 형태로 호출합니다.

## 메서드 오버로딩: 다채로운 입력의 수용 (컴파일 타임 다형성)

메서드 오버로딩(Overloading)은 동일한 이름을 가진 메서드를 매개변수의 개수, 타입, 순서를 다르게 하여 여러 개 정의하는 기법입니다. 호출하는 사용자 입장에서는 같은 개념의 작업을 동일한 이름의 메서드로 일관성 있게 처리할 수 있어 API의 사용성이 크게 향상됩니다.

```csharp
public class Calculator
{
    public int Add(int a, int b) => a + b;
    public double Add(double a, double b) => a + b;
    public int Add(params int[] numbers) => numbers.Sum();
}

```

오버로딩을 설계할 때 주의할 점은 **반환 타입만 다른 경우는 오버로딩으로 인정되지 않는다**는 것입니다. 컴파일러는 메서드의 이름과 매개변수 목록(시그니처)만을 기준으로 메서드를 식별합니다.

또한 암시적 형변환으로 인해 발생하는 모호성을 주의해야 합니다. 컴파일러는 전달된 인수와 가장 '구체적으로 일치하는(변환 과정이 적은)' 오버로드를 선택하지만, 여러 타입으로 똑같이 변환될 수 있는 상황에서는 컴파일 에러를 발생시킵니다.

## 메서드 오버라이딩: 객체의 진화 (런타임 다형성)

메서드 오버라이딩(Overriding)은 부모 클래스로부터 물려받은 메서드를 자식 클래스에서 자신만의 방식으로 새롭게 재정의하는 것입니다. 객체 지향의 꽃이라 불리는 **다형성(Polymorphism)**을 실현하는 핵심 메커니즘입니다.

부모 클래스에서 재정의를 허락한다는 의미로 `virtual`이나 `abstract` 키워드를 붙이면, 자식 클래스에서는 `override` 키워드를 사용하여 해당 로직을 덮어씁니다.

### 가상 메서드 테이블 (vtable)의 내부 동작

C#에서 오버라이딩이 다형성을 발휘할 수 있는 이유는 메모리 이면에 **가상 메서드 테이블(Virtual Method Table, vtable)**이라는 정교한 시스템이 돌아가고 있기 때문입니다.

```text
[vtable의 논리적 동작 구조]

Base 타입의 변수에 Derived 객체를 담고 가상 메서드를 호출할 때:
Base obj = new Derived();
obj.Draw();

1. 컴파일러는 obj가 런타임에 어떤 객체일지 확신하지 못합니다.
2. 실행 시점(Runtime)에 obj가 가리키는 힙 메모리 객체의 vtable을 조회합니다.
3. 해당 vtable에는 Base의 Draw가 아닌, Derived가 덮어씌운(Override) Draw 메서드의 실제 메모리 주소가 들어있습니다.
4. 따라서 컴파일 타임 변수 타입과 무관하게, 실제 객체의 재정의된 메서드가 완벽하게 실행됩니다.

```

### new 키워드: 메서드 숨기기와 설계의 실패

파생 클래스에서 부모와 똑같은 이름의 메서드를 만들면서 `override`가 아닌 `new` 키워드를 사용하면, 이는 메서드를 재정의하는 것이 아니라 부모의 메서드를 **숨겨버리는(Hiding)** 결과를 낳습니다.

이는 vtable의 슬롯을 교체하는 것이 아니라, 아예 별개의 새로운 슬롯을 하나 더 파버리는 행동입니다. 결과적으로 다형성이 완벽하게 깨지게 됩니다.

```csharp
public class BaseClass
{
    public virtual void Show() => Console.WriteLine("부모의 Show");
}

public class DerivedClass : BaseClass
{
    // new를 사용하여 부모의 Show를 무시하고 새로운 Show를 만듭니다.
    public new void Show() => Console.WriteLine("자식의 Show");
}

BaseClass obj1 = new DerivedClass();
obj1.Show(); // 출력: "부모의 Show" (다형성이 깨져서 변수 타입인 BaseClass의 메서드가 호출됨!)

DerivedClass obj2 = new DerivedClass();
obj2.Show(); // 출력: "자식의 Show"

```

실무에서 `new` 키워드를 통한 메서드 숨기기는 코드를 읽는 동료 개발자들에게 엄청난 혼란을 야기합니다. 피치 못하게 레거시 라이브러리의 메서드 이름과 충돌하는 상황이 아니라면 철저히 지양해야 하는 안티 패턴입니다.

## 종합 실전 예제: 객체 지향 은행 계좌 시스템

지금까지 배운 생성자, `this`/`base` 키워드, 그리고 오버라이딩을 통한 다형성을 완벽하게 융합한 실전 예제입니다.

```csharp
using System;

// 기본 계좌 추상 클래스
public abstract class Account
{
    public string AccountNumber { get; }
    public string OwnerName { get; }
    public decimal Balance { get; protected set; }
    
    // 부모 생성자: 기본 계좌 정보 초기화 및 검증
    protected Account(string accountNumber, string ownerName, decimal initialBalance = 0)
    {
        if (string.IsNullOrWhiteSpace(accountNumber))
            throw new ArgumentException("계좌번호는 필수입니다.");
        
        if (string.IsNullOrWhiteSpace(ownerName))
            throw new ArgumentException("예금주명은 필수입니다.");
        
        if (initialBalance < 0)
            throw new ArgumentException("초기 잔액은 0 이상이어야 합니다.");
        
        AccountNumber = accountNumber;
        OwnerName = ownerName;
        Balance = initialBalance;
    }
    
    // 입금 가상 메서드 (자식 클래스에서 오버라이드 가능하도록 열어둠)
    public virtual void Deposit(decimal amount)
    {
        if (amount <= 0)
            throw new ArgumentException("입금액은 0보다 커야 합니다.");
        
        Balance += amount;
        Console.WriteLine($"{amount:C} 입금 완료. 현재 잔액: {Balance:C}");
    }
    
    // 출금 추상 메서드 (계좌 종류마다 출금 규칙이 다르므로 자식에게 구현을 강제함)
    public abstract bool Withdraw(decimal amount);
    
    public virtual void DisplayInfo()
    {
        Console.WriteLine($"계좌번호: {AccountNumber} | 예금주: {OwnerName} | 잔액: {Balance:C}");
    }
}

// 저축 계좌 (Account 상속)
public class SavingsAccount : Account
{
    public decimal InterestRate { get; }
    
    // 생성자 체이닝 (this)
    public SavingsAccount(string accountNumber, string ownerName) 
        : this(accountNumber, ownerName, 0, 0.02m) { }
    
    // 부모 생성자 호출 (base)
    public SavingsAccount(string accountNumber, string ownerName, decimal initialBalance, decimal interestRate)
        : base(accountNumber, ownerName, initialBalance)
    {
        InterestRate = interestRate;
    }
    
    // 입금 메서드 오버라이딩 (부모 로직 재사용 + 보너스 로직 추가)
    public override void Deposit(decimal amount)
    {
        base.Deposit(amount); 
        
        // 저축 계좌만의 특별 혜택: 대량 입금 시 보너스
        if (amount >= 1000000)
        {
            decimal bonus = amount * 0.001m;
            Balance += bonus;
            Console.WriteLine($"[이벤트] 대량 입금 보너스 {bonus:C} 추가 적립!");
        }
    }
    
    // 추상 메서드 구현 (최소 잔액 유지 규칙 적용)
    public override bool Withdraw(decimal amount)
    {
        decimal minimumBalance = 10000;
        
        if (Balance - amount < minimumBalance)
        {
            Console.WriteLine($"[거절] 저축 계좌는 최소 잔액 {minimumBalance:C}을 유지해야 합니다.");
            return false;
        }
        
        Balance -= amount;
        Console.WriteLine($"{amount:C} 출금 완료. 현재 잔액: {Balance:C}");
        return true;
    }
}

// 당좌 계좌 (마이너스 통장 기능)
public class CheckingAccount : Account
{
    public decimal OverdraftLimit { get; }
    
    public CheckingAccount(string accountNumber, string ownerName, decimal overdraftLimit = 500000)
        : base(accountNumber, ownerName)
    {
        OverdraftLimit = overdraftLimit;
    }
    
    public override bool Withdraw(decimal amount)
    {
        // 마이너스 통장 한도 규칙 적용
        if (Balance - amount < -OverdraftLimit)
        {
            Console.WriteLine($"[거절] 마이너스 한도 초과 (최대 {-OverdraftLimit:C}까지 가능)");
            return false;
        }
        
        Balance -= amount;
        Console.WriteLine($"{amount:C} 출금 완료. 현재 잔액: {Balance:C}");
        return true;
    }
    
    public override void DisplayInfo()
    {
        base.DisplayInfo();
        Console.WriteLine($"[옵션] 마이너스 통장 한도: {OverdraftLimit:C}");
    }
}

class Program
{
    static void Main()
    {
        Console.WriteLine("=== 은행 시스템 시뮬레이션 ===\n");
        
        // 1. 저축 계좌 테스트
        SavingsAccount savings = new SavingsAccount("SAV-001", "홍길동", 50000, 0.03m);
        savings.DisplayInfo();
        savings.Deposit(1500000); // 100만 원 이상 입금 보너스 발동
        savings.Withdraw(1530000); // 잔액이 10,000원 미만으로 떨어지려 하여 거절됨
        
        Console.WriteLine("\n---------------------------\n");
        
        // 2. 당좌 계좌 테스트
        CheckingAccount checking = new CheckingAccount("CHK-001", "김철수");
        checking.DisplayInfo();
        checking.Withdraw(300000); // 마이너스 30만 원으로 출금 성공
    }
}

```

## 마무리

생성자는 객체의 생명을 안전하게 잉태시키고, `this`와 `base`는 상속 관계 속에서 자식과 부모의 컨텍스트를 명확히 조율합니다. 나아가 오버로딩은 API의 편의성을, 오버라이딩은 런타임 다형성이라는 객체 지향의 꽃을 피워냅니다.

상속 구조에서 생성자 내 가상 메서드 호출의 위험성을 인지하고, vtable 기반의 동적 디스패치 원리를 이해하여 `new` 키워드의 오남용을 막는 것은 주니어 개발자가 시니어로 성장하기 위해 반드시 거쳐야 할 필수 관문입니다. 이러한 메커니즘을 정확히 통제할 때 비로소 예측 가능하고 유지보수가 뛰어난 애플리케이션을 설계할 수 있을 것입니다.