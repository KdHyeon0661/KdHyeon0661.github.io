---
layout: post
title: AspNet - Mocking 도구
date: 2025-04-10 22:20:23 +0900
category: AspNet
---
# 🧪 Mocking 도구 소개 (Moq 중심 + 대안 도구 비교)

---

## ✅ 1. Mock란?

**Mock 객체**는 테스트 중 실제 구현 대신 사용되는 **가짜 객체**로,  
다음과 같은 상황에서 사용됨:

- DB, API 같은 **외부 의존성 제거**
- **단위 테스트 격리** 및 빠른 실행
- **예외 상황/특정 조건** 시뮬레이션

---

## 🔧 2. 대표 Mocking 도구 비교

| 도구 | 특징 | 사용 언어 | 인기 |
|------|------|-----------|------|
| ✅ Moq | 가장 널리 쓰이는 .NET용 Mock 라이브러리 | C# | 매우 높음 |
| NSubstitute | 간단하고 문법이 직관적 | C# | 중간 |
| FakeItEasy | 깔끔한 API, BDD 스타일 | C# | 중간 |
| Rhino Mocks | 오래된 레거시용 | C# | 낮음 |

> 이 문서에서는 가장 많이 쓰이는 **Moq**을 중심으로 설명하고,  
> 다른 도구들과의 차이점은 마지막에 비교해줘.

---

## 📦 3. Moq 설치

```bash
dotnet add package Moq
```

또는 프로젝트 `.csproj` 파일에 수동 추가 가능:

```xml
<PackageReference Include="Moq" Version="4.18.4" />
```

---

## 🧪 4. 기본 사용법

### 👇 인터페이스 예시

```csharp
public interface IUserService
{
    string GetUserName(int id);
}
```

### 👇 Mocking & 테스트

```csharp
public class UserControllerTests
{
    [Fact]
    public void GetUserName_ShouldReturnMockedName()
    {
        // 1. Mock 객체 생성
        var mock = new Mock<IUserService>();

        // 2. 동작 지정
        mock.Setup(s => s.GetUserName(1)).Returns("TestUser");

        // 3. 객체 주입
        var controller = new UserController(mock.Object);

        // 4. 테스트 실행
        var result = controller.GetUserName(1) as OkObjectResult;

        Assert.Equal("TestUser", result?.Value);
    }
}
```

---

## 🧩 5. Moq 주요 메서드 정리

| 메서드 | 설명 |
|--------|------|
| `Setup()` | 특정 메서드 호출 시 반환값 지정 |
| `Returns()` | 반환값 지정 |
| `Throws()` | 예외 발생 설정 |
| `Verify()` | 특정 메서드가 호출되었는지 검증 |
| `It.IsAny<T>()` | 어떤 값이든 매칭 |
| `It.Is<T>(...)` | 특정 조건에 맞는 인자만 매칭 |

---

### 📌 `Throws()` 예제

```csharp
mock.Setup(s => s.GetUserName(0)).Throws<ArgumentException>();
```

---

### 📌 `Verify()` 예제

```csharp
mock.Verify(s => s.GetUserName(1), Times.Once());
```

---

### 📌 매개변수 조건 매칭

```csharp
mock.Setup(s => s.GetUserName(It.Is<int>(id => id > 0)))
    .Returns("ValidUser");
```

---

## 🔄 6. 콜백 & 상태 기반 테스트

### 👉 콜백으로 값 캡처

```csharp
string captured = "";
mock.Setup(s => s.GetUserName(It.IsAny<int>()))
    .Callback<int>(id => captured = $"ID: {id}")
    .Returns("Done");
```

---

## 🧠 7. Stub vs Mock vs Fake 비교

| 용어 | 의미 | 예시 |
|------|------|------|
| **Stub** | 반환값만 설정, 동작은 없음 | `.Returns(...)` |
| **Mock** | Stub + 호출 검증 (`Verify`) | `.Verify(...)` |
| **Fake** | 실제 동작하는 가짜 객체 | `InMemoryDbContext` 등 |

> 실무에서는 대부분 Stub + Mock 조합을 사용함.

---

## 🔀 8. 다른 Mock 프레임워크 간 비교

| 기능 | Moq | NSubstitute | FakeItEasy |
|------|-----|-------------|-------------|
| 기본 문법 | `mock.Setup(...)` | `sub.SomeMethod().Returns(...)` | `A.CallTo(...).Returns(...)` |
| BDD 스타일 | ❌ | ✅ | ✅ |
| 익명 객체 사용 | 가능 | 매우 쉬움 | 쉬움 |
| 진입 장벽 | 낮음 | 매우 낮음 | 낮음 |

### 🔹 NSubstitute 예

```csharp
var sub = Substitute.For<IUserService>();
sub.GetUserName(1).Returns("User123");
```

---

## 🚀 9. 실전 활용 팁

- Mock은 **서비스, DB, API** 등에 주로 사용 (예: `IUserRepository`)
- `Verify`를 통해 **서비스가 호출됐는지 확인**
- `It.IsAny<T>()`를 남용하면 **불명확한 테스트**가 될 수 있음
- 외부 리소스(Mock DB, 파일, API 등)는 최대한 **인터페이스화**하고 테스트 분리

---

## ✅ 요약

| 항목 | 내용 |
|------|------|
| Mocking 목적 | 의존성 격리, 빠른 테스트, 예외 시뮬레이션 |
| 가장 많이 쓰는 도구 | Moq |
| 핵심 메서드 | `Setup()`, `Returns()`, `Throws()`, `Verify()` |
| 대체 도구 | NSubstitute, FakeItEasy |
| 실전 팁 | 호출 횟수 검증, 조건 지정, 콜백 활용 가능 |