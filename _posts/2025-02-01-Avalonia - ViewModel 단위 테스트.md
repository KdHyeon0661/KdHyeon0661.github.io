---
layout: post
title: Avalonia - 다양한 컨트롤 바인딩 (DatePicker, ComboBox, CheckBox 등)
date: 2025-02-01 19:20:23 +0900
category: Avalonia
---
# Avalonia MVVM: ViewModel 단위 테스트

## 테스트가 필요한 이유

MVVM 패턴의 핵심은 ViewModel이 UI와 비즈니스 로직을 분리하는 계층이라는 점이다. 이 구조 덕분에 ViewModel은 순수 C# 클래스로 존재하며, View에 의존하지 않는다. 따라서 **ViewModel만 따로 떼어 자동화된 테스트**를 작성할 수 있다.

테스트를 통해 다음과 같은 이점을 얻는다.

- 로직 변경 시 기존 동작이 망가졌는지 빠르게 확인
- 복잡한 비즈니스 규칙을 문서화하는 효과
- 리팩토링에 대한 안전망 확보

## 테스트를 위한 ViewModel 설계 원칙

ViewModel을 테스트하기 쉽게 만들려면 몇 가지 원칙을 지켜야 한다.

| 원칙 | 설명 |
|------|------|
| UI 의존성 제거 | 파일 대화상자, 네비게이션, 메시지 박스 등은 인터페이스로 추상화 |
| 시간 의존성 제어 | `DateTime.Now` 대신 `IClock` 인터페이스를 주입받아 테스트에서 고정 시간 사용 |
| 스케줄러 주입 | `Task.Delay`나 Rx 타이머 대신 `IScheduler`를 주입받아 테스트 스케줄러로 대체 |
| 명령 기반 동작 | 버튼 클릭 이벤트 대신 `ReactiveCommand`를 사용해 실행 흐름을 테스트 |

## 프로젝트 구조 예시

```
MyApp/
├─ ViewModels/
│  ├─ LoginViewModel.cs
│  └─ ProfileViewModel.cs
├─ Services/
│  ├─ Abstractions/
│  │  ├─ IAuthService.cs
│  │  └─ IClock.cs
│  └─ Implementations/
│     ├─ AuthService.cs
│     └─ SystemClock.cs
└─ Tests/
   ├─ ViewModels/
   │  ├─ LoginViewModelTests.cs
   │  └─ ProfileViewModelTests.cs
   └─ TestInfra/
      ├─ TestClock.cs
      └─ RxTestContext.cs
```

## 의존성 추상화 예시

### 시간 추상화

```csharp
public interface IClock
{
    DateTimeOffset Now { get; }
}

public class SystemClock : IClock
{
    public DateTimeOffset Now => DateTimeOffset.Now;
}
```

### 로그인 서비스 추상화

```csharp
public interface IAuthService
{
    Task<bool> LoginAsync(string username, string password);
}
```

## 테스트 가능한 ViewModel 작성

```csharp
using ReactiveUI;
using System;
using System.Reactive;
using System.Reactive.Linq;
using System.Threading.Tasks;

public class LoginViewModel : ReactiveObject
{
    private readonly IAuthService _auth;
    private readonly IClock _clock;

    public LoginViewModel(IAuthService auth, IClock clock)
    {
        _auth = auth;
        _clock = clock;

        var canLogin = this.WhenAnyValue(
            x => x.Username, x => x.Password,
            (u, p) => !string.IsNullOrWhiteSpace(u) && !string.IsNullOrWhiteSpace(p));

        LoginCommand = ReactiveCommand.CreateFromTask(ExecuteLoginAsync, canLogin);

        LoginCommand.IsExecuting
            .Subscribe(isBusy => IsBusy = isBusy);

        LoginCommand.ThrownExceptions
            .Subscribe(ex => LastError = ex.Message);
    }

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

    private bool _isBusy;
    public bool IsBusy
    {
        get => _isBusy;
        private set => this.RaiseAndSetIfChanged(ref _isBusy, value);
    }

    private string? _lastError;
    public string? LastError
    {
        get => _lastError;
        private set => this.RaiseAndSetIfChanged(ref _lastError, value);
    }

    private DateTimeOffset? _lastLoginAt;
    public DateTimeOffset? LastLoginAt
    {
        get => _lastLoginAt;
        private set => this.RaiseAndSetIfChanged(ref _lastLoginAt, value);
    }

    public ReactiveCommand<Unit, bool> LoginCommand { get; }

    private async Task<bool> ExecuteLoginAsync()
    {
        var ok = await _auth.LoginAsync(Username, Password);
        if (ok) LastLoginAt = _clock.Now;
        return ok;
    }
}
```

## 테스트 인프라 구축

### 가짜 클럭

```csharp
public class TestClock : IClock
{
    public DateTimeOffset Now { get; set; } = new DateTimeOffset(2020, 1, 1, 0, 0, 0, TimeSpan.Zero);
}
```

### Rx 스케줄러 제어를 위한 컨텍스트

```csharp
using System;
using ReactiveUI;
using ReactiveUI.Testing;
using Microsoft.Reactive.Testing;

public sealed class RxTestContext : IDisposable
{
    public TestScheduler Main { get; }
    public TestScheduler TaskPool { get; }

    private readonly IDisposable _resetMain;
    private readonly IDisposable _resetTask;

    public RxTestContext()
    {
        Main = new TestScheduler();
        TaskPool = new TestScheduler();

        _resetMain = RxApp.MainThreadScheduler.With(Main);
        _resetTask = RxApp.TaskpoolScheduler.With(TaskPool);
    }

    public void AdvanceMainBy(TimeSpan time) => Main.AdvanceBy(time.Ticks);
    public void AdvanceTaskBy(TimeSpan time) => TaskPool.AdvanceBy(time.Ticks);

    public void Dispose()
    {
        _resetMain.Dispose();
        _resetTask.Dispose();
    }
}
```

### 의존성 주입 컨테이너 (테스트용)

```csharp
using Microsoft.Extensions.DependencyInjection;

public static class TestServiceProvider
{
    public static ServiceProvider Build()
    {
        return new ServiceCollection()
            .AddSingleton<IAuthService, FakeAuthService>()
            .AddSingleton<IClock, TestClock>()
            .BuildServiceProvider();
    }
}
```

## xUnit + FluentAssertions를 사용한 기본 테스트

### 명령 활성화 조건 테스트

```csharp
using FluentAssertions;
using Xunit;

public class LoginViewModelTests
{
    [Fact]
    public void LoginCommand_ShouldBeDisabled_WhenFieldsEmpty()
    {
        using var ctx = new RxTestContext();
        var sp = TestServiceProvider.Build();

        var vm = new LoginViewModel(
            sp.GetRequiredService<IAuthService>(),
            sp.GetRequiredService<IClock>());

        vm.LoginCommand.CanExecute.FirstAsync().Wait().Should().BeFalse();

        vm.Username = "admin";
        vm.LoginCommand.CanExecute.FirstAsync().Wait().Should().BeFalse();

        vm.Password = "1234";
        vm.LoginCommand.CanExecute.FirstAsync().Wait().Should().BeTrue();
    }
}
```

### 로그인 성공 테스트

```csharp
[Fact]
public async Task LoginCommand_Success_SetsLastLoginAt()
{
    using var ctx = new RxTestContext();
    var sp = TestServiceProvider.Build();

    var clock = (TestClock)sp.GetRequiredService<IClock>();
    clock.Now = new DateTimeOffset(2030, 12, 31, 23, 59, 59, TimeSpan.Zero);

    var vm = new LoginViewModel(
        sp.GetRequiredService<IAuthService>(),
        clock)
    {
        Username = "admin",
        Password = "1234"
    };

    var result = await vm.LoginCommand.Execute();

    result.Should().BeTrue();
    vm.IsBusy.Should().BeFalse();
    vm.LastError.Should().BeNull();
    vm.LastLoginAt.Should().Be(clock.Now);
}
```

## Mocking 프레임워크 사용 (Moq 예시)

```csharp
using Moq;
using Xunit;

[Fact]
public async Task LoginCommand_CallsAuthServiceOnce()
{
    var mockAuth = new Mock<IAuthService>();
    mockAuth.Setup(a => a.LoginAsync("user", "pass")).ReturnsAsync(true);

    var vm = new LoginViewModel(mockAuth.Object, new TestClock())
    {
        Username = "user",
        Password = "pass"
    };

    await vm.LoginCommand.Execute();

    mockAuth.Verify(a => a.LoginAsync("user", "pass"), Times.Once);
}
```

## ReactiveUI.Validation을 사용한 폼 검증 테스트

```csharp
public class SignUpViewModel : ReactiveValidationObject
{
    public SignUpViewModel()
    {
        this.ValidationRule(
            vm => vm.Email,
            email => email?.Contains("@") == true,
            "이메일 형식이 올바르지 않습니다.");

        this.ValidationRule(
            vm => vm.Password,
            p => !string.IsNullOrWhiteSpace(p) && p.Length >= 6,
            "비밀번호는 6자 이상이어야 합니다.");
    }

    private string? _email;
    public string? Email
    {
        get => _email;
        set => this.RaiseAndSetIfChanged(ref _email, value);
    }

    private string? _password;
    public string? Password
    {
        get => _password;
        set => this.RaiseAndSetIfChanged(ref _password, value);
    }

    public bool CanSubmit => !this.ValidationContext.GetIsValid();
}
```

```csharp
[Fact]
public void Validation_ShouldFail_WhenEmailInvalid()
{
    var vm = new SignUpViewModel { Email = "invalid" };

    vm.ValidationContext.GetIsValid().Should().BeFalse();
    vm.ValidationContext.Text.Should().Contain("이메일 형식이 올바르지 않습니다.");
}
```

## 시간·지연·타이머 테스트 (스케줄러 주입)

```csharp
public class TimerViewModel : ReactiveObject
{
    private bool _fired;
    public bool Fired
    {
        get => _fired;
        private set => this.RaiseAndSetIfChanged(ref _fired, value);
    }

    public TimerViewModel(IScheduler scheduler)
    {
        Observable.Timer(TimeSpan.FromSeconds(3), scheduler)
            .Subscribe(_ => Fired = true);
    }
}
```

```csharp
[Fact]
public void Timer_FiresAfter3Seconds()
{
    var scheduler = new TestScheduler();
    var vm = new TimerViewModel(scheduler);

    vm.Fired.Should().BeFalse();
    scheduler.AdvanceBy(TimeSpan.FromSeconds(2).Ticks);
    vm.Fired.Should().BeFalse();

    scheduler.AdvanceBy(TimeSpan.FromSeconds(1).Ticks);
    vm.Fired.Should().BeTrue();
}
```

## 속성 변경 알림 테스트

```csharp
public static List<string> CapturePropertyChanges(INotifyPropertyChanged npc, Action action)
{
    var changes = new List<string>();
    PropertyChangedEventHandler handler = (_, e) => changes.Add(e.PropertyName!);
    npc.PropertyChanged += handler;
    try { action(); }
    finally { npc.PropertyChanged -= handler; }
    return changes;
}

[Fact]
public void Username_RaisesPropertyChanged()
{
    var vm = new LoginViewModel(Mock.Of<IAuthService>(), new TestClock());
    var changes = CapturePropertyChanges(vm, () => vm.Username = "new");
    changes.Should().Contain("Username");
}
```

## 예외 처리 테스트 (ReactiveCommand의 ThrownExceptions)

```csharp
public class FaultyViewModel : ReactiveObject
{
    public ReactiveCommand<Unit, Unit> FaultyCommand { get; }

    public FaultyViewModel()
    {
        FaultyCommand = ReactiveCommand.CreateFromTask(() => throw new InvalidOperationException("Fail"));
        FaultyCommand.ThrownExceptions.Subscribe(ex => LastError = ex.Message);
    }

    private string? _lastError;
    public string? LastError
    {
        get => _lastError;
        private set => this.RaiseAndSetIfChanged(ref _lastError, value);
    }
}

[Fact]
public async Task FaultyCommand_EmitsException()
{
    var vm = new FaultyViewModel();
    await vm.FaultyCommand.Execute();
    vm.LastError.Should().Be("Fail");
}
```

## CI에서의 테스트 실행 팁

- **결정성 확보**: `Task.Delay`나 `Thread.Sleep` 대신 스케줄러/클럭을 주입해 시간을 제어한다.
- **병렬 실행**: 각 테스트가 독립적인 DI 컨테이너를 사용해 상태를 격리한다.
- **커버리지 측정**: ViewModel과 서비스 계층 위주로 측정하고, UI(XAML)는 스냅샷 테스트 등 별도 도구로 검증한다.

## 결론

MVVM 패턴은 ViewModel을 순수 C# 클래스로 유지할 수 있게 해주며, 따라서 단위 테스트가 매우 용이하다. 테스트를 작성할 때는 의존성 주입을 통해 외부 의존을 추상화하고, 시간과 스케줄러를 제어할 수 있는 구조를 설계하는 것이 중요하다. ReactiveUI의 `ReactiveCommand`와 `WhenAnyValue`, `ReactiveUI.Validation` 등을 활용하면 테스트하기 쉬운 ViewModel을 만들 수 있다. 이러한 접근을 통해 애플리케이션의 핵심 로직을 안정적으로 보호하고, 리팩토링과 기능 확장에 자신감을 얻을 수 있다.