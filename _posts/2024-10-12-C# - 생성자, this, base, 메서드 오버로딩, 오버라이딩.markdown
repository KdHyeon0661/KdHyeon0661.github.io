---
layout: post
title: C# - 생성자, this, base, 메서드 오버로딩, 오버라이딩
date: 2024-10-12 19:20:23 +0900
category: C#
---
# C# 생성자, this/base 키워드, 메서드 오버로딩, 메서드 오버라이드

이번 글에서는 클래스 설계의 핵심 요소인 **생성자**, 그리고 **this / base 키워드**, **메서드 오버로딩과 오버라이딩**에 대해 살펴봅니다.

---

## 🔷 생성자(Constructor)

**생성자**는 객체가 생성될 때 자동으로 호출되는 메서드입니다. 주로 **초기화 작업**에 사용됩니다.

```csharp
class Person
{
    public string Name;

    public Person()  // 기본 생성자
    {
        Name = "이름 없음";
    }
}
```

### ✅ 객체 생성 시 자동 호출

```csharp
Person p = new Person(); // 생성자가 자동 실행됨
Console.WriteLine(p.Name); // "이름 없음"
```

---

## 🔷 매개변수가 있는 생성자

```csharp
class Person
{
    public string Name;

    public Person(string name)
    {
        Name = name;
    }
}
```

```csharp
Person p = new Person("홍길동");
Console.WriteLine(p.Name); // "홍길동"
```

---

## 🔷 this 키워드

`this`는 **현재 객체 자신**을 가리킬 때 사용합니다.

```csharp
class Person
{
    private string name;

    public Person(string name)
    {
        this.name = name; // 매개변수와 필드 이름이 같을 때 구분
    }
}
```

### ✅ this() 생성자 호출

```csharp
class Person
{
    public string Name;
    public int Age;

    public Person(string name) : this(name, 0) { }

    public Person(string name, int age)
    {
        Name = name;
        Age = age;
    }
}
```

- 하나의 생성자가 다른 생성자를 호출할 때 사용
- 반드시 생성자의 **첫 줄**에서 호출해야 함

---

## 🔷 base 키워드

`base`는 **부모 클래스의 멤버 또는 생성자**를 호출할 때 사용합니다.

```csharp
class Animal
{
    public Animal(string name)
    {
        Console.WriteLine("Animal 생성자: " + name);
    }
}

class Dog : Animal
{
    public Dog(string name) : base(name)
    {
        Console.WriteLine("Dog 생성자: " + name);
    }
}
```

---

## 🔷 생성자 오버로딩

**여러 개의 생성자**를 정의하여 다양한 방식으로 객체를 초기화할 수 있습니다.

```csharp
class Rectangle
{
    public int Width, Height;

    public Rectangle()
    {
        Width = 1;
        Height = 1;
    }

    public Rectangle(int size)
    {
        Width = Height = size;
    }

    public Rectangle(int width, int height)
    {
        Width = width;
        Height = height;
    }
}
```

---

## 🔷 메서드 오버라이딩 (Method Overriding)

**상속 관계에서 부모 클래스의 메서드를 자식 클래스에서 재정의**하는 기능입니다.

```csharp
class Animal
{
    public virtual void Speak()
    {
        Console.WriteLine("동물이 소리를 냅니다");
    }
}

class Cat : Animal
{
    public override void Speak()
    {
        Console.WriteLine("야옹");
    }
}
```

```csharp
Animal a = new Cat();
a.Speak(); // "야옹" 출력
```

- 부모 메서드에 `virtual` 필요
- 자식 클래스에서 `override`로 재정의

---

## 🔷 메서드 오버로딩 (Method Overloading)

**같은 이름의 메서드를 매개변수만 다르게** 정의하는 기법입니다.

```csharp
class Printer
{
    public void Print(string msg)
    {
        Console.WriteLine(msg);
    }

    public void Print(int number)
    {
        Console.WriteLine("숫자: " + number);
    }

    public void Print(string msg, int repeat)
    {
        for (int i = 0; i < repeat; i++)
            Console.WriteLine(msg);
    }
}
```

- 반환형이 달라져도 **매개변수가 같으면 오버로딩 불가**
- 메서드 이름은 같고 **매개변수 개수 또는 타입**이 달라야 함

---

## ✅ 정리

| 개념 | 설명 |
|------|------|
| 생성자 | 객체 생성 시 자동 실행되는 메서드 |
| this | 현재 객체 참조, 다른 생성자 호출에도 사용 |
| base | 부모 클래스의 생성자/멤버 호출 |
| 생성자 오버로딩 | 다양한 방식의 초기화를 위한 생성자 다중 정의 |
| 메서드 오버로딩 | 같은 이름의 메서드를 매개변수 다르게 정의 |
| 메서드 오버라이딩 | 상속받은 메서드를 자식 클래스에서 재정의 (`virtual` / `override`) |