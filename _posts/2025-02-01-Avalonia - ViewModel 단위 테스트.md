---
layout: post
title: Avalonia - 다양한 컨트롤 바인딩 (DatePicker, ComboBox, CheckBox 등)
date: 2025-02-01 19:20:23 +0900
category: Avalonia
---
# Avalonia MVVM: ViewModel 단위 테스트

## 0. 테스트 철학 요약

- **ViewModel = 순수 클래스**: UI 의존 제거(파일/다이얼로그/네비게이션/테마 등은 서비스 인터페이스로 추상화).
- **시간·스레드 결정성**: Rx/비동기 코드는 **스케줄러/클럭 주입**으로 테스트 가능하게 만든다.
- **상태 관찰 방식 통일**: `INotifyPropertyChanged`, `ReactiveCommand.CanExecute/IsExecuting`, `ThrownExceptions`, `ValidationContext`, `HasErrors`.
- **DI/Mocking 규범화**: `ServiceCollection` 혹은 TestHost로 **한 번** 올리고, 테스트별로 격리.

---

## 1. 프로젝트/솔루션 구조(샘플)

```
MyApp/
├─ MyApp.csproj
├─ ViewModels/
│  ├─ LoginViewModel.cs
│  ├─ SignUpViewModel.cs
│  ├─ ProfileViewModel.cs
│  └─ DashboardViewModel.cs
├─ Services/
│  ├─ Abstractions/
│  │  ├─ IAuthService.cs
│  │  ├─ IUserService.cs
│  │  ├─ IClock.cs
│  │  └─ ILoggerFacade.cs
│  └─ Implementations/
│     ├─ AuthService.cs
│     ├─ UserService.cs
│     ├─ SystemClock.cs
│     └─ LoggerFacade.cs
└─ Tests/
   ├─ MyApp.Tests.csproj
   ├─ ViewModels/
   │  ├─ LoginViewModelTests.cs
   │  ├─ SignUpViewModelTests.cs
   │  ├─ ProfileViewModelTests.cs
   │  └─ DashboardViewModelTests.cs
   └─ TestInfra/
      ├─ TestClock.cs
      ├─ RxTestContext.cs
      ├─ DI.cs
      └─ TestHelpers.cs
```

---

## 2. 기본 ViewModel(테스트 대상) — 개선 버전

초안의 `LoginViewModel`을 **테스트 친화적**으로 정돈한다.

```csharp
// MyApp/ViewModels/LoginViewModel.cs
using ReactiveUI;
using System;
using System.Reactive;
using System.Reactive.Linq;
using System.Threading.Tasks;
using MyApp.Services.Abstractions;

namespace MyApp.ViewModels;

public class LoginViewModel : ReactiveObject
{
    private readonly IAuthService _auth;
    private readonly IClock _clock; // 시간 의존성 분리(테스트에서 제어 가능)

    public LoginViewModel(IAuthService auth, IClock clock)
    {
        _auth = auth;
        _clock = clock;

        var canLogin = this.WhenAnyValue(
            x => x.Username, x => x.Password,
            (u, p) => !string.IsNullOrWhiteSpace(u) && !string.IsNullOrWhiteSpace(p));

        LoginCommand = ReactiveCommand.CreateFromTask(ExecuteLoginAsync, canLogin);

        // 커맨드 수행 중/결과/예외 관찰 가능
        LoginCommand.IsExecuting
            .Subscribe(exec => IsBusy = exec);

        LoginCommand.ThrownExceptions
            .Subscribe(ex => LastError = ex.Message);
    }

    private string _username = string.Empty;
    public string Username
    {
        get => _username;
        set => this.RaiseAndSetIfChanged(ref _username, value);
    }

    private string _password = string.Empty;
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
        // 인위적 지연이 필요하면 Task.Delay 대신 clock/scheduler를 사용하거나 주입한다.
        var ok = await _auth.LoginAsync(Username, Password);
        if (ok) LastLoginAt = _clock.Now;
        return ok;
    }
}
```

```csharp
// MyApp/Services/Abstractions/IClock.cs
using System;

namespace MyApp.Services.Abstractions;

public interface IClock
{
    DateTimeOffset Now { get; }
}
```

```csharp
// MyApp/Services/Implementations/SystemClock.cs
using System;
using MyApp.Services.Abstractions;

namespace MyApp.Services.Implementations;

public sealed class SystemClock : IClock
{
    public DateTimeOffset Now => DateTimeOffset.Now;
}
```

---

## 3. 테스트 인프라: Rx/TestScheduler/Clock/DI 유틸

**결정성** 확보를 위해 ReactiveUI/Rx 스케줄러와 시간을 제어한다.

```csharp
// Tests/TestInfra/TestClock.cs
using System;
using MyApp.Services.Abstractions;

namespace MyApp.Tests.TestInfra;

public sealed class TestClock : IClock
{
    public DateTimeOffset Now { get; set; } = new DateTimeOffset(2020,1,1,0,0,0,TimeSpan.Zero);
}
```

```csharp
// Tests/TestInfra/RxTestContext.cs
using System;
using ReactiveUI;
using ReactiveUI.Testing;
using Microsoft.Reactive.Testing;

namespace MyApp.Tests.TestInfra;

public sealed class RxTestContext : IDisposable
{
    public TestScheduler Ui { get; }
    public TestScheduler TaskPool { get; }

    private readonly IDisposable _disp1;
    private readonly IDisposable _disp2;

    public RxTestContext()
    {
        Ui = new TestScheduler();
        TaskPool = new TestScheduler();

        _disp1 = RxApp.MainThreadScheduler.With(Ui);
        _disp2 = RxApp.TaskpoolScheduler.With(TaskPool);
    }

    public void AdvanceUiBy(TimeSpan ts) => Ui.AdvanceBy(ts.Ticks);
    public void AdvanceTaskBy(TimeSpan ts) => TaskPool.AdvanceBy(ts.Ticks);

    public void Dispose()
    {
        _disp1.Dispose();
        _disp2.Dispose();
    }
}
```

> `ReactiveUI.Testing`의 `With` 확장은 스케줄러 대체를 쉽게 해준다. 이 컨텍스트를 각 테스트에서 `using`으로 감싸면, Rx 연산이 결정적으로 재현된다.

```csharp
// Tests/TestInfra/DI.cs
using Microsoft.Extensions.DependencyInjection;
using MyApp.Services.Abstractions;
using MyApp.Services.Implementations;

namespace MyApp.Tests.TestInfra;

public static class DI
{
    public static ServiceProvider BuildProviderForTests()
    {
        var sc = new ServiceCollection();

        // 프로덕션 대비: 테스트에서만 가짜/테스트 구현을 주입 가능
        sc.AddSingleton<IAuthService, AuthService>(); // 필요 시 FakeAuth로 교체
        sc.AddSingleton<IClock, TestClock>();

        return sc.BuildServiceProvider();
    }
}
```

---

## 4. xUnit + FluentAssertions + Rx 스케줄러 제어 테스트

```csharp
// Tests/ViewModels/LoginViewModelTests.cs
using System.Threading.Tasks;
using FluentAssertions;
using Xunit;
using MyApp.ViewModels;
using MyApp.Tests.TestInfra;
using MyApp.Services.Abstractions;
using Microsoft.Extensions.DependencyInjection;

public class LoginViewModelTests
{
    [Fact]
    public void LoginCommand_Should_Disable_When_Fields_Empty()
    {
        using var rx = new RxTestContext();
        using var sp = DI.BuildProviderForTests();

        var auth = sp.GetRequiredService<IAuthService>();
        var clock = sp.GetRequiredService<IClock>();

        var vm = new LoginViewModel(auth, clock);

        // CanExecute는 IObservable<bool>이다.
        vm.LoginCommand.CanExecute.FirstAsync().Wait().Should().BeFalse();

        vm.Username = "admin";
        vm.Password = "";
        vm.LoginCommand.CanExecute.FirstAsync().Wait().Should().BeFalse();

        vm.Password = "1234";
        vm.LoginCommand.CanExecute.FirstAsync().Wait().Should().BeTrue();
    }

    [Fact]
    public async Task LoginCommand_Should_Succeed_And_Set_Timestamp()
    {
        using var rx = new RxTestContext();
        using var sp = DI.BuildProviderForTests();

        var clock = (TestClock)sp.GetRequiredService<IClock>();
        clock.Now = new System.DateTimeOffset(2030, 12, 31, 23, 59, 59, System.TimeSpan.Zero);

        var auth = sp.GetRequiredService<IAuthService>();
        var vm = new LoginViewModel(auth, clock)
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

    [Fact]
    public async Task LoginCommand_Should_Fail_On_Invalid_Creds()
    {
        using var rx = new RxTestContext();
        using var sp = DI.BuildProviderForTests();

        var auth = sp.GetRequiredService<IAuthService>();
        var clock = sp.GetRequiredService<IClock>();

        var vm = new LoginViewModel(auth, clock)
        {
            Username = "admin",
            Password = "wrong"
        };

        var result = await vm.LoginCommand.Execute();
        result.Should().BeFalse();
        vm.IsBusy.Should().BeFalse();
        vm.LastLoginAt.Should().BeNull();
    }
}
```

> 포인트  
> - `RxTestContext`로 메인/백그라운드 스케줄러 **완전 통제** → UI 없는 환경에서도 동작이 결정적.  
> - `IClock`을 TestClock으로 대체하여 **시간 의존성 제거**.

---

## 5. ReactiveUI.Validation 테스트(폼 검증)

```csharp
// MyApp/ViewModels/SignUpViewModel.cs
using ReactiveUI.Validation.Abstractions;
using ReactiveUI.Validation.Contexts;
using ReactiveUI.Validation.Extensions;
using ReactiveUI.Validation.Helpers;
using ReactiveUI;

using System.Text.RegularExpressions;

namespace MyApp.ViewModels;

public class SignUpViewModel : ReactiveValidationObject
{
    public SignUpViewModel()
    {
        this.ValidationRule(
            vm => vm.Email,
            email => Regex.IsMatch(email ?? "", @"^[\w\.-]+@[\w\.-]+\.\w+$"),
            "이메일 형식이 잘못되었습니다");

        this.ValidationRule(
            vm => vm.Password,
            p => !string.IsNullOrWhiteSpace(p) && p!.Length >= 8,
            "비밀번호는 8자 이상이어야 합니다");

        this.ValidationRule(
            vm => vm.Confirm,
            _ => Password == Confirm,
            "비밀번호가 일치하지 않습니다");
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

    private string? _confirm;
    public string? Confirm
    {
        get => _confirm;
        set => this.RaiseAndSetIfChanged(ref _confirm, value);
    }

    public bool CanSubmit => !this.ValidationContext?.GetIsValid() == false ? false : true;
}
```

```csharp
// Tests/ViewModels/SignUpViewModelTests.cs
using FluentAssertions;
using Xunit;
using MyApp.ViewModels;

public class SignUpViewModelTests
{
    [Fact]
    public void Email_Should_Fail_When_Invalid()
    {
        var vm = new SignUpViewModel { Email = "invalid@@@" };

        vm.ValidationContext.GetIsValid().Should().BeFalse();
        var msg = string.Join("; ", vm.ValidationContext.Text);
        msg.Should().Contain("이메일 형식이 잘못되었습니다");
    }

    [Fact]
    public void Confirm_Should_Fail_When_Not_Match()
    {
        var vm = new SignUpViewModel { Password = "abcd1234", Confirm = "abcd123X" };

        vm.ValidationContext.GetIsValid().Should().BeFalse();
        string.Join("; ", vm.ValidationContext.Text).Should().Contain("비밀번호가 일치하지 않습니다");
    }

    [Fact]
    public void All_Valid_Should_Allow_Submit()
    {
        var vm = new SignUpViewModel
        {
            Email = "user@test.com",
            Password = "abcd1234",
            Confirm = "abcd1234"
        };

        vm.ValidationContext.GetIsValid().Should().BeTrue();
        vm.CanSubmit.Should().BeTrue();
    }
}
```

---

## 6. DI 의존 ViewModel 테스트 — Moq/NSubstitute 예시

서비스로 외부 의존을 숨기고, Mock으로 주입하여 **상호작용**을 검증한다.

```csharp
// MyApp/ViewModels/ProfileViewModel.cs
using System.Threading.Tasks;
using ReactiveUI;
using MyApp.Services.Abstractions;

namespace MyApp.ViewModels;

public class ProfileViewModel : ReactiveObject
{
    private readonly IUserService _users;

    public ProfileViewModel(IUserService users) => _users = users;

    private User? _user;
    public User? User
    {
        get => _user;
        private set => this.RaiseAndSetIfChanged(ref _user, value);
    }

    public async Task LoadAsync() => User = await _users.GetCurrentUserAsync();
}

public record User(string Name);
```

### Moq

```csharp
// Tests/ViewModels/ProfileViewModelTests.cs
using System.Threading.Tasks;
using FluentAssertions;
using Xunit;
using Moq;
using MyApp.ViewModels;
using MyApp.Services.Abstractions;

public class ProfileViewModelTests
{
    [Fact]
    public async Task LoadAsync_Should_Set_User()
    {
        var mock = new Mock<IUserService>();
        mock.Setup(s => s.GetCurrentUserAsync())
            .ReturnsAsync(new User("홍길동"));

        var vm = new ProfileViewModel(mock.Object);
        await vm.LoadAsync();

        vm.User.Should().NotBeNull();
        vm.User!.Name.Should().Be("홍길동");
        mock.Verify(s => s.GetCurrentUserAsync(), Times.Once);
    }
}
```

### NSubstitute

```csharp
// Tests/ViewModels/ProfileViewModelNSubstituteTests.cs
using System.Threading.Tasks;
using FluentAssertions;
using Xunit;
using NSubstitute;
using MyApp.ViewModels;
using MyApp.Services.Abstractions;

public class ProfileViewModelNSubstituteTests
{
    [Fact]
    public async Task LoadAsync_Should_Set_User()
    {
        var svc = Substitute.For<IUserService>();
        svc.GetCurrentUserAsync().Returns(new User("이순신"));

        var vm = new ProfileViewModel(svc);
        await vm.LoadAsync();

        vm.User!.Name.Should().Be("이순신");
        await svc.Received(1).GetCurrentUserAsync();
    }
}
```

---

## 7. 커맨드/예외/중복 실행 테스트 팁

```csharp
// MyApp/ViewModels/DashboardViewModel.cs
using ReactiveUI;
using System;
using System.Reactive;
using System.Threading.Tasks;

namespace MyApp.ViewModels;

public class DashboardViewModel : ReactiveObject
{
    public ReactiveCommand<Unit, Unit> DangerousCommand { get; }

    private bool _flag;
    public bool Flag
    {
        get => _flag;
        set => this.RaiseAndSetIfChanged(ref _flag, value);
    }

    public DashboardViewModel()
    {
        var canRun = this.WhenAnyValue(x => x.Flag);
        DangerousCommand = ReactiveCommand.CreateFromTask(DoAsync, canRun);
        DangerousCommand.ThrownExceptions.Subscribe(ex => LastError = ex.GetType().Name);
    }

    private string? _lastError;
    public string? LastError
    {
        get => _lastError;
        private set => this.RaiseAndSetIfChanged(ref _lastError, value);
    }

    private async Task DoAsync()
    {
        await Task.Yield();
        throw new InvalidOperationException("Boom");
    }
}
```

```csharp
// Tests/ViewModels/DashboardViewModelTests.cs
using System.Threading.Tasks;
using FluentAssertions;
using Xunit;

public class DashboardViewModelTests
{
    [Fact]
    public async Task DangerousCommand_Should_Emit_Exception_Into_ThrownExceptions()
    {
        var vm = new MyApp.ViewModels.DashboardViewModel
        {
            Flag = true
        };

        // 예외는 Throw되지 않고 ThrownExceptions 스트림으로 전달됨.
        await vm.DangerousCommand.Execute();
        vm.LastError.Should().Be("InvalidOperationException");
    }

    [Fact]
    public void Command_Should_Be_Disabled_If_Flag_False()
    {
        var vm = new MyApp.ViewModels.DashboardViewModel
        {
            Flag = false
        };

        vm.DangerousCommand.CanExecute.FirstAsync().Wait().Should().BeFalse();
    }
}
```

> **반패턴**: `await Assert.ThrowsAsync<...>(...)`를 ReactiveCommand에 바로 쓰면 종종 실패한다(예외가 스트림으로 흘러가기 때문).

---

## 8. 속성 변경/알림 테스트(Notify)

```csharp
// Tests/TestInfra/TestHelpers.cs
using System;
using System.Collections.Generic;
using System.ComponentModel;

namespace MyApp.Tests.TestInfra;

public static class TestHelpers
{
    public static List<string> CapturePropertyChanges(INotifyPropertyChanged npc, Action act)
    {
        var list = new List<string>();
        PropertyChangedEventHandler handler = (_, e) => list.Add(e.PropertyName!);
        npc.PropertyChanged += handler;
        try { act(); }
        finally { npc.PropertyChanged -= handler; }
        return list;
    }
}
```

```csharp
// Tests/ViewModels/LoginViewModel_PropertyChanged_Tests.cs
using FluentAssertions;
using Xunit;
using MyApp.Tests.TestInfra;
using MyApp.ViewModels;
using MyApp.Services.Abstractions;
using Microsoft.Extensions.DependencyInjection;

public class LoginViewModel_PropertyChanged_Tests
{
    [Fact]
    public void Username_Raises_PropertyChanged()
    {
        using var sp = DI.BuildProviderForTests();
        var vm = new LoginViewModel(
            sp.GetRequiredService<IAuthService>(),
            sp.GetRequiredService<IClock>());

        var changes = TestHelpers.CapturePropertyChanges(vm, () => vm.Username = "abc");
        changes.Should().Contain("Username");
    }
}
```

---

## 9. 시간·지연·폴링 테스트 — 가짜 클럭/스케줄러

- `Task.Delay` 대신 **스케줄러 기반 지연**을 VM에서 사용하거나, 시간 기준 로직은 `IClock`을 주입.
- Rx 타이머/Interval은 `TestScheduler`로 제어(AdvanceBy/AdvanceTo).

예: 3초 후 상태 플래그 ON

```csharp
// MyApp/ViewModels/TimerViewModel.cs
using ReactiveUI;
using System;
using System.Reactive.Linq;

namespace MyApp.ViewModels;

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
// Tests/ViewModels/TimerViewModelTests.cs
using FluentAssertions;
using Xunit;
using Microsoft.Reactive.Testing;

public class TimerViewModelTests
{
    [Fact]
    public void Should_Fire_After_3_Seconds()
    {
        var sched = new TestScheduler();
        var vm = new MyApp.ViewModels.TimerViewModel(sched);

        vm.Fired.Should().BeFalse();
        sched.AdvanceBy(TimeSpan.FromSeconds(2).Ticks);
        vm.Fired.Should().BeFalse();

        sched.AdvanceBy(TimeSpan.FromSeconds(1).Ticks);
        vm.Fired.Should().BeTrue();
    }
}
```

> 스케줄러를 **생성자 주입**하면 Rx 시간 연산이 100% 결정적이다.

---

## 10. NUnit/MSTest 차이 간단 정리

| 항목 | xUnit | NUnit | MSTest |
|---|---|---|---|
| 테스트 속성 | `[Fact]`/`[Theory]` | `[Test]`/`[TestCase]` | `[TestMethod]`/`[DataTestMethod]` |
| SetUp/TearDown | `ctor`/`IDisposable.Dispose()` | `[SetUp]`/`[TearDown]` | `[TestInitialize]`/`[TestCleanup]` |
| async 지원 | 기본적 | 기본적 | 기본적 |
| 어서션 | `Assert` + FluentAssertions | `Assert` + FluentAssertions | `Assert` + FluentAssertions |

> 프레임워크와 무관하게 **핵심은 동일**: DI/Mocking/Rx 스케줄러 제어.

---

## 11. CI에서의 실용 팁

- **Flaky 방지**: Task.Delay/Thread.Sleep 제거, 스케줄러/클럭으로 결정성 확보.
- **병렬 테스트**: 상태 공유하는 싱글턴을 테스트 범위에서 사용하면 **격리** 필요(테스트용 DI 구축).
- **커버리지**: ViewModel과 서비스 로직 위주로 측정. UI(XAML)는 스냅샷/Golden Test를 별도 도구로.

---

## 12. 종합 예시: “입력 → 상태 → 실행 → 결과/검증”

```csharp
// Tests/ViewModels/EndToEnd_Login_Flow_Tests.cs
using System.Threading.Tasks;
using FluentAssertions;
using Xunit;
using MyApp.ViewModels;
using MyApp.Tests.TestInfra;
using Microsoft.Extensions.DependencyInjection;
using MyApp.Services.Abstractions;

public class EndToEnd_Login_Flow_Tests
{
    [Fact]
    public async Task Full_Login_Flow()
    {
        using var rx = new RxTestContext();
        using var sp = DI.BuildProviderForTests();

        var auth = sp.GetRequiredService<IAuthService>();
        var clock = sp.GetRequiredService<IClock>();

        var vm = new LoginViewModel(auth, clock);

        // 1) 입력
        vm.Username = "admin";
        vm.Password = "1234";

        // 2) 조건 확인
        (await vm.LoginCommand.CanExecute.FirstAsync()).Should().BeTrue();

        // 3) 실행
        var ok = await vm.LoginCommand.Execute();
        ok.Should().BeTrue();

        // 4) 결과 검증
        vm.IsBusy.Should().BeFalse();
        vm.LastError.Should().BeNull();
        vm.LastLoginAt.Should().NotBeNull();
    }
}
```

---

## 13. 보너스: Golden Master/스냅샷 테스트(간단 문자열 상태)

뷰모델이 내보내는 “요약 문자열/로그 라인”이 안정적으로 유지되는지 스냅샷 파일로 검증.

```csharp
// Tests/ViewModels/SnapshotTests.cs
using System.IO;
using System.Threading.Tasks;
using FluentAssertions;
using Xunit;

public class SnapshotTests
{
    [Fact]
    public async Task Summary_Snapshot_Should_Match()
    {
        var vm = new MyApp.ViewModels.SummaryViewModel
        {
            Title = "Report",
            Count = 3
        };

        var actual = vm.SummaryText; // "Report (3 items)" 등
        var snapshotPath = "Snapshots/Summary.txt";
        Directory.CreateDirectory("Snapshots");

        if (!File.Exists(snapshotPath))
        {
            await File.WriteAllTextAsync(snapshotPath, actual);
            // 최초 실행 시 스냅샷 생성. 이후 diff로 검증.
        }

        var expected = await File.ReadAllTextAsync(snapshotPath);
        actual.Should().Be(expected);
    }
}
```

> 도메인 로직 문자열/JSON 출력은 **스냅샷 기반**으로 회귀 테스트 가능.

---

## 14. 결론 — 테스트 전략 체크리스트

| 항목 | 실천 포인트 |
|---|---|
| View 없이 테스트 | ViewModel은 순수 C# 클래스. UI는 전부 서비스 인터페이스로 추상화. |
| 시간/스케줄러 | `IClock`/`IScheduler` 주입. `RxTestContext`로 RxApp 스케줄러 교체. |
| 커맨드 | `CanExecute`/`Execute`/`IsExecuting`/`ThrownExceptions` 전부 관찰. |
| Validation | `ReactiveUI.Validation` → `ValidationContext`/`HasErrors`/복합 규칙 테스트. |
| DI/Mocking | `ServiceCollection` or TestHost + Moq/NSubstitute. 테스트 격리. |
| 경쟁 상태 제거 | Task.Delay/Thread.Sleep 금지. 스케줄러/클럭으로 결정적 시뮬레이션. |
| 스냅샷 | 문자열/JSON 출력은 Golden Master로 회귀 방지. |

---

## 부록: 패키지 참조 예시(.csproj)

```xml
<ItemGroup>
  <PackageReference Include="ReactiveUI" Version="18.*" />
  <PackageReference Include="ReactiveUI.Validation" Version="3.*" />
  <PackageReference Include="ReactiveUI.Testing" Version="18.*" />
  <PackageReference Include="Microsoft.Reactive.Testing" Version="6.*" />
</ItemGroup>

<!-- Tests -->
<ItemGroup>
  <PackageReference Include="xunit" Version="2.*" />
  <PackageReference Include="xunit.runner.visualstudio" Version="2.*" />
  <PackageReference Include="FluentAssertions" Version="6.*" />
  <PackageReference Include="Moq" Version="4.*" />
  <PackageReference Include="NSubstitute" Version="5.*" />
  <PackageReference Include="Microsoft.Extensions.DependencyInjection" Version="8.*" />
</ItemGroup>
```

---

이로써 초안의 내용을 **테스트 설계/보일러플레이트/스케줄러와 시간 제어/DI·Mocking/Validation/예외·중복 실행/스냅샷**까지 확장했다.  
이 뼈대만 갖추면, **Avalonia 유무와 상관없이** ViewModel 로직은 빠르고 안정적으로 CI에서 검증된다.