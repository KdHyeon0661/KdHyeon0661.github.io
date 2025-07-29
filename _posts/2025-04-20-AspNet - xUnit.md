---
layout: post
title: AspNet - xUnit
date: 2025-04-10 20:20:23 +0900
category: AspNet
---
# 🧪 xUnit 사용법 완전 가이드 (ASP.NET Core 기준)

---

## ✅ 1. xUnit이란?

> .NET에서 가장 많이 사용되는 **단위 테스트 프레임워크**  
(MSTest, NUnit도 있지만 xUnit이 .NET Core/ASP.NET Core에 가장 적합)

- **경량**, **확장성**, **비동기 지원**이 뛰어남
- ASP.NET 팀이 공식 채택한 테스트 프레임워크

---

## ⚙️ 2. xUnit 프로젝트 생성 및 설치

### 📦 CLI로 테스트 프로젝트 만들기

```bash
dotnet new xunit -n MyApp.Tests
cd MyApp.Tests
```

### 📦 ASP.NET Core 프로젝트와 연결

솔루션에 추가:

```bash
dotnet new sln -n MyApp
dotnet sln add ./MyApp/MyApp.csproj
dotnet sln add ./MyApp.Tests/MyApp.Tests.csproj
dotnet add ./MyApp.Tests/MyApp.Tests.csproj reference ./MyApp/MyApp.csproj
```

### 📁 폴더 구조 예시

```
MyApp/
├── Controllers/
├── Services/
└── MyApp.csproj

MyApp.Tests/
├── Services/
│   └── UserServiceTests.cs
└── MyApp.Tests.csproj
```

---

## 🧪 3. 기본 테스트 작성법

```csharp
public class Calculator
{
    public int Add(int a, int b) => a + b;
}
```

### 🔹 테스트 클래스

```csharp
public class CalculatorTests
{
    [Fact] // 단일 테스트
    public void Add_ShouldReturnCorrectSum()
    {
        var calc = new Calculator();
        var result = calc.Add(2, 3);
        Assert.Equal(5, result);
    }
}
```

---

## 🧩 4. 주요 어노테이션 정리

| 어노테이션 | 설명 |
|------------|------|
| `[Fact]` | 매개변수 없는 테스트 |
| `[Theory]` | 매개변수 테스트 (데이터 기반) |
| `[InlineData(...)]` | Theory에 데이터 주입 |

### 🔹 `[Theory]` 예제

```csharp
[Theory]
[InlineData(1, 2, 3)]
[InlineData(5, 5, 10)]
public void Add_MultipleCases(int a, int b, int expected)
{
    var calc = new Calculator();
    var result = calc.Add(a, b);
    Assert.Equal(expected, result);
}
```

---

## 🧵 5. 비동기 테스트 지원

xUnit은 `async Task` 반환을 기본 지원함

```csharp
[Fact]
public async Task GetUserAsync_ShouldReturnUser()
{
    var userService = new UserService();
    var user = await userService.GetUserAsync(1);
    Assert.NotNull(user);
}
```

---

## 🧪 6. 예외 테스트

```csharp
[Fact]
public void Divide_ByZero_ShouldThrow()
{
    var calc = new Calculator();
    Assert.Throws<DivideByZeroException>(() => calc.Divide(10, 0));
}
```

---

## 🧰 7. 테스트 초기화 & 해제 (Setup / Teardown)

### 클래스 단위의 테스트 준비

```csharp
public class UserServiceTests : IDisposable
{
    private readonly UserService _service;

    public UserServiceTests()
    {
        _service = new UserService(); // Setup
    }

    public void Dispose()
    {
        // Cleanup
    }

    [Fact]
    public void ShouldWork() { }
}
```

---

## 🔧 8. 의존성 주입 환경에서 테스트하기

서비스에 DI가 필요한 경우 → **Mock 객체** 사용 또는 **테스트용 DI 구성**

### 예: `ILogger` 모킹

```csharp
public class MyServiceTests
{
    private readonly MyService _service;
    private readonly ILogger<MyService> _logger;

    public MyServiceTests()
    {
        _logger = new Mock<ILogger<MyService>>().Object;
        _service = new MyService(_logger);
    }

    [Fact]
    public void Run_ShouldLogMessage()
    {
        var result = _service.Run();
        Assert.True(result);
    }
}
```

---

## 📊 9. 테스트 실행 방법

### 📦 CLI에서 실행

```bash
dotnet test
```

### 📦 Visual Studio에서

- 테스트 탐색기(Test Explorer) 사용
- 테스트 클래스/메서드 오른쪽 클릭 → "테스트 실행"

---

## 🛠️ 10. 유용한 도구

| 도구 | 설명 |
|------|------|
| `Moq` | 인터페이스 Mock 객체 생성 |
| `FluentAssertions` | 더 읽기 쉬운 Assertion 문법 제공 |
| `Coverlet` | 코드 커버리지 측정 도구 |
| `xunit.runner.visualstudio` | VS 테스트 실행 지원 |

---

## 🧠 11. 실전 팁

- 한 테스트에는 **하나의 검증만** 포함하자 (단일 책임)
- **테스트 이름은 명확하게** (`메서드명_조건_결과`)
- **순서 의존 테스트 금지**
- 데이터베이스 I/O 테스트는 `InMemory`, `SQLite` 등을 활용

---

## ✅ 요약

| 항목 | 설명 |
|------|------|
| 프로젝트 생성 | `dotnet new xunit` |
| 단위 테스트 | `[Fact]`, `Assert.Equal()` |
| 매개변수 테스트 | `[Theory]`, `[InlineData(...)]` |
| 비동기 테스트 | `async Task` 지원 |
| 예외 테스트 | `Assert.Throws<T>()` |
| 의존성 주입 테스트 | Moq 등 활용 |
| 실행 | `dotnet test`, Visual Studio |