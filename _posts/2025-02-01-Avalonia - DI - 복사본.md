---
layout: post
title: Avalonia - 다양한 컨트롤 바인딩 (DatePicker, ComboBox, CheckBox 등)
date: 2025-01-22 19:20:23 +0900
category: Avalonia
---
# ✅ Avalonia MVVM: ViewModel 단위 테스트 작성 가이드

---

## 🧪 왜 ViewModel을 테스트해야 하나요?

| 이유 | 설명 |
|------|------|
| UI 없는 테스트 가능 | Avalonia 없이 순수 C# 단위 테스트 가능 |
| 로직 검증 | 사용자 입력 → 상태 변화 → 검증까지 체크 |
| 회귀 방지 | 검증 로직, 커맨드, 조건부 처리 오류 방지 |
| CI/CD 자동화 | GUI 없이 빠르게 자동 검증 가능 |

---

## 🧱 테스트 구성 요소 요약

| 항목 | 예시 |
|------|------|
| 테스트 프레임워크 | xUnit / NUnit / MSTest |
| Assertion | FluentAssertions, Shouldly, 기본 Assert |
| Mocking | Moq / NSubstitute (의존성 주입 테스트용) |
| Reactive 지원 | ReactiveUI.Testing, TestScheduler (Rx 전용) |

---

## 📁 프로젝트 구조 예시

```
MyApp/
├── ViewModels/
│   └── LoginViewModel.cs
MyApp.Tests/
├── ViewModels/
│   └── LoginViewModelTests.cs
```

---

# 1️⃣ 기본 ViewModel 예시 (테스트 대상)

```csharp
public class LoginViewModel : ReactiveObject
{
    private string _username = "";
    public string Username
    {
        get => _username;
        set => this.RaiseAndSetIfChanged(ref _username, value);
    }

    private string _password = "";
    public string Password
    {
        get => _password;
        set => this.RaiseAndSetIfChanged(ref _password, value);
    }

    public ReactiveCommand<Unit, bool> LoginCommand { get; }

    public LoginViewModel()
    {
        var canLogin = this.WhenAnyValue(
            x => x.Username, x => x.Password,
            (u, p) => !string.IsNullOrWhiteSpace(u) && !string.IsNullOrWhiteSpace(p));

        LoginCommand = ReactiveCommand.CreateFromTask(async () =>
        {
            await Task.Delay(100); // 서버 호출 시뮬레이션
            return Username == "admin" && Password == "1234";
        }, canLogin);
    }
}
```

---

# 2️⃣ 단위 테스트 코드 작성 (xUnit + FluentAssertions)

## 📄 LoginViewModelTests.cs

```csharp
using Xunit;
using FluentAssertions;
using MyApp.ViewModels;
using System.Threading.Tasks;

public class LoginViewModelTests
{
    [Fact]
    public void LoginCommand_ShouldBeDisabled_WhenFieldsAreEmpty()
    {
        var vm = new LoginViewModel();

        vm.LoginCommand.CanExecute.FirstAsync().Result.Should().BeFalse();

        vm.Username = "admin";
        vm.Password = "";

        vm.LoginCommand.CanExecute.FirstAsync().Result.Should().BeFalse();
    }

    [Fact]
    public async Task LoginCommand_ShouldReturnTrue_WhenCorrectCredentials()
    {
        var vm = new LoginViewModel
        {
            Username = "admin",
            Password = "1234"
        };

        var result = await vm.LoginCommand.Execute();
        result.Should().BeTrue();
    }

    [Fact]
    public async Task LoginCommand_ShouldReturnFalse_WhenWrongPassword()
    {
        var vm = new LoginViewModel
        {
            Username = "admin",
            Password = "wrong"
        };

        var result = await vm.LoginCommand.Execute();
        result.Should().BeFalse();
    }
}
```

> ✅ `FirstAsync().Result`로 `IObservable<bool>`을 검사할 수 있습니다  
> ✅ `ReactiveCommand.Execute()`는 async이므로 `await` 테스트 필요

---

# 3️⃣ 검증 포함 ViewModel 테스트 (ReactiveUI.Validation)

### 📄 SignUpViewModel.cs (일부 발췌)

```csharp
this.ValidationRule(
    vm => vm.Email,
    email => Regex.IsMatch(email ?? "", @"^[\w\.-]+@[\w\.-]+\.\w+$"),
    "이메일 형식이 잘못되었습니다");

public bool CanSubmit => !HasErrors;
```

### 📄 SignUpViewModelTests.cs

```csharp
[Fact]
public void EmailValidation_ShouldFail_WithInvalidEmail()
{
    var vm = new SignUpViewModel
    {
        Email = "invalid@@@"
    };

    vm.HasErrors.Should().BeTrue();
    vm.ValidationContext.Text.Should().Contain("이메일 형식이 잘못되었습니다");
}
```

---

# 4️⃣ Command 테스트 팁

| 시나리오 | 예시 |
|----------|------|
| 커맨드 조건 만족 여부 | `LoginCommand.CanExecute.First()` |
| 커맨드 실행 결과 | `await LoginCommand.Execute()` |
| 커맨드 중복 실행 방지 | `IsExecuting` 체크 가능 |
| 커맨드 예외 처리 | `ThrownExceptions.Subscribe(...)` 테스트 가능 |

---

# 5️⃣ 의존성 주입된 ViewModel 테스트

### ViewModel에 의존성 주입

```csharp
public class ProfileViewModel
{
    private readonly IUserService _userService;

    public ProfileViewModel(IUserService userService)
    {
        _userService = userService;
    }

    public async Task LoadUser()
    {
        User = await _userService.GetCurrentUser();
    }

    public User? User { get; private set; }
}
```

### 테스트 코드 (Moq 사용)

```csharp
[Fact]
public async Task LoadUser_ShouldSetUser_WhenServiceReturnsUser()
{
    var mock = new Mock<IUserService>();
    mock.Setup(s => s.GetCurrentUser()).ReturnsAsync(new User { Name = "홍길동" });

    var vm = new ProfileViewModel(mock.Object);
    await vm.LoadUser();

    vm.User.Should().NotBeNull();
    vm.User!.Name.Should().Be("홍길동");
}
```

---

## 📦 팁: 테스트 프레임워크 선택 비교

| 프레임워크 | 특징 |
|------------|------|
| **xUnit** | 널리 사용, 간결함, async 테스트 기본 지원 |
| **NUnit** | 풍부한 기능, `SetUp`, `TearDown` |
| **MSTest** | Microsoft 기본, Visual Studio 친화적 |
| **FluentAssertions** | 가독성 높은 문법 (`x.Should().Be(...)`) |
| **Moq / NSubstitute** | Mocking용 (DI 테스트 필수) |

---

## ✅ 결론: ViewModel 테스트 전략

| 항목 | 테스트 내용 |
|------|-------------|
| 속성 변경 | RaisePropertyChanged 발생 여부 |
| 커맨드 | 조건부 실행, 결과, 예외 |
| 유효성 검사 | ValidationContext, HasErrors 등 |
| 외부 서비스 호출 | Mocking으로 주입 후 확인 |
| 전체 플로우 | 입력 → 상태 → 실행 → 결과 확인

---

## 📘 다음 확장 주제 제안

- 🧪 ViewModel 테스트에서 Rx TestScheduler 사용
- 🧭 NavigationViewModel 테스트 (메시지, 상태 전환)
- 📦 Form 유효성 + 서버 API Mock 테스트
- 🧬 상태 스냅샷/롤백 가능한 ViewModel 아키텍처
