---
layout: post
title: Avalonia - 상태머신 기반 UI 흐름
date: 2025-02-17 21:20:23 +0900
category: Avalonia
---
# Avalonia MVVM에서 상태머신 기반 UI 흐름 구현하기

핵심은 다음 다섯 가지다.

1) **명확한 상태 모델**: 상태·이벤트·전이·가드(전이 조건)·효과(사이드이펙트)를 분리
2) **UI-상태 분리**: ViewModel은 상태머신을 구독하고, View는 바인딩만 담당
3) **유효성 검증**: Step별 검증 실패 시 전이 거부, 오류 메시지/에러 템플릿 연계
4) **비동기/서버 연동**: 이메일 중복 검사 등 전이 효과(Effect)에서 수행
5) **운영성**: 상태 기록, 재진입/복구, 직렬화/복원, 단위 테스트/시뮬레이션

---

## 0. 용어와 표기

- **State(상태)**: UI의 단계(예: `Email` → `Password` → `Complete`)
- **Event(이벤트)**: 사용자/시스템 입력(예: `NEXT`, `BACK`, `SUBMIT`)
- **Transition(전이)**: `(현재 상태, 이벤트, 가드 충족) → 다음 상태`
- **Guard(가드)**: 전이 조건(예: 이메일 형식·중복 검사 통과)
- **Effect(효과)**: 전이 시 부수 효과(예: 서버 호출, 로깅, 토스트)

상태머신을 간단히 수식으로 적으면 다음과 같다.

$$
\delta : (S \times E) \times G \to S
$$

여기서 \( S \)는 상태 집합, \( E \)는 이벤트 집합, \( G \)는 가드(불리언)이며, \(\delta\)는 전이 함수이다.

---

## 1. 예제 도메인 시나리오(회원가입 Wizard)

- Step1: 이메일 입력 및 **서버 중복 검사**
- Step2: 비밀번호/확인 입력 및 **규칙 검증**
- Step3: 완료 화면
- 제약: 유효하지 않으면 `Next` 불가, `Back`은 항상 가능(옵션)

디렉터리 구조(확장판):

```
MyApp/
├── Models/
│   ├── SignupData.cs
│   └── ValidationResult.cs
├── StateMachine/
│   ├── SignupStep.cs
│   ├── SignupEvent.cs
│   ├── IStateManager.cs
│   ├── GuardResult.cs
│   ├── Transition.cs
│   ├── StateMachineCore.cs            // 범용 상태머신 코어
│   ├── SignupFlowStateMachine.cs      // 도메인 상태머신 정의
│   └── StateLogger.cs
├── Services/
│   ├── IAccountService.cs             // 서버 연동(중복 검사, 가입 등)
│   └── AccountService.cs
├── ViewModels/
│   ├── Step1ViewModel.cs
│   ├── Step2ViewModel.cs
│   ├── Step3ViewModel.cs
│   └── SignupFlowViewModel.cs         // Shell/Orchestrator
├── Views/
│   ├── Step1View.axaml
│   ├── Step2View.axaml
│   ├── Step3View.axaml
│   └── SignupFlowView.axaml
└── App.axaml(.cs)
```

---

## 2. 모델과 상태/이벤트 정의

### Models/SignupData.cs

```csharp
public class SignupData
{
    public string Email { get; set; } = "";
    public string Password { get; set; } = "";
    public string ConfirmPassword { get; set; } = "";
}
```

### Models/ValidationResult.cs

```csharp
public sealed class ValidationResult
{
    public bool IsValid { get; }
    public string? ErrorMessage { get; }

    private ValidationResult(bool ok, string? msg)
    {
        IsValid = ok;
        ErrorMessage = msg;
    }

    public static ValidationResult Ok() => new(true, null);
    public static ValidationResult Fail(string message) => new(false, message);
}
```

### StateMachine/SignupStep.cs

```csharp
public enum SignupStep
{
    Step1_Email,
    Step2_Password,
    Step3_Complete
}
```

### StateMachine/SignupEvent.cs

```csharp
public enum SignupEvent
{
    Next,
    Back,
    Submit // Step2 -> Step3 전용 이벤트로 사용 가능
}
```

---

## 3. 상태머신 코어(범용)와 가드/전이

### StateMachine/IStateManager.cs (범용 인터페이스)

```csharp
public interface IStateManager<TState, TEvent>
{
    TState Current { get; }
    Task<bool> SendAsync(TEvent @event, CancellationToken ct = default);
    bool Can(TEvent @event);
}
```

### StateMachine/GuardResult.cs

```csharp
public readonly struct GuardResult
{
    public bool Allow { get; }
    public string? Reason { get; }

    public GuardResult(bool allow, string? reason)
    {
        Allow = allow;
        Reason = reason;
    }

    public static GuardResult Ok() => new(true, null);
    public static GuardResult Deny(string reason) => new(false, reason);
}
```

### StateMachine/Transition.cs

```csharp
public sealed class Transition<TState, TEvent>
{
    public TState From { get; init; }
    public TEvent When { get; init; }
    public TState To { get; init; }

    // 가드: 비동기 허용 (서버검증)
    public Func<Task<GuardResult>>? GuardAsync { get; init; }

    // 효과: 전이 성공 시 실행
    public Func<Task>? EffectAsync { get; init; }

    // 전이 불가 시(가드 실패) 실행되는 후크(옵션)
    public Func<string, Task>? OnDeniedAsync { get; init; }
}
```

### StateMachine/StateMachineCore.cs

```csharp
public sealed class StateMachineCore<TState, TEvent> : IStateManager<TState, TEvent>
{
    private readonly List<Transition<TState, TEvent>> _transitions;
    private readonly Action<TState>? _onStateChanged;

    public TState Current { get; private set; }

    public StateMachineCore(
        TState initial,
        IEnumerable<Transition<TState, TEvent>> transitions,
        Action<TState>? onStateChanged = null)
    {
        Current = initial;
        _transitions = transitions.ToList();
        _onStateChanged = onStateChanged;
    }

    public bool Can(TEvent @event)
        => _transitions.Any(t => Equals(t.From, Current) && Equals(t.When, @event));

    public async Task<bool> SendAsync(TEvent @event, CancellationToken ct = default)
    {
        var candidates = _transitions
            .Where(t => Equals(t.From, Current) && Equals(t.When, @event))
            .ToList();

        if (candidates.Count == 0)
            return false;

        foreach (var tr in candidates)
        {
            ct.ThrowIfCancellationRequested();

            GuardResult guard = GuardResult.Ok();
            if (tr.GuardAsync != null)
            {
                guard = await tr.GuardAsync();
            }

            if (!guard.Allow)
            {
                if (tr.OnDeniedAsync != null)
                    await tr.OnDeniedAsync(guard.Reason ?? "Guard denied");
                continue; // 다른 후보 전이 시도(있다면)
            }

            // Effect 먼저 실행할지, 상태 변경 후 실행할지는 정책에 따라 선택
            // 여기서는 상태 변경 후 Effect 실행
            Current = tr.To;
            _onStateChanged?.Invoke(Current);

            if (tr.EffectAsync != null)
                await tr.EffectAsync();

            return true;
        }

        return false;
    }
}
```

---

## 4. 도메인 상태머신 작성: SignupFlowStateMachine

서버 연동(이메일 중복 검사, 가입 API 호출)은 Service에 위임한다.

### Services/IAccountService.cs

```csharp
public interface IAccountService
{
    Task<bool> CheckEmailAvailableAsync(string email, CancellationToken ct = default);
    Task<bool> SignupAsync(string email, string password, CancellationToken ct = default);
}
```

### Services/AccountService.cs (예시: 샘플 구현)

```csharp
public sealed class AccountService : IAccountService
{
    public async Task<bool> CheckEmailAvailableAsync(string email, CancellationToken ct = default)
    {
        await Task.Delay(200, ct); // API 호출 시뮬
        return !email.Contains("used@example.com", StringComparison.OrdinalIgnoreCase);
    }

    public async Task<bool> SignupAsync(string email, string password, CancellationToken ct = default)
    {
        await Task.Delay(300, ct); // API 호출 시뮬
        return password.Length >= 8;
    }
}
```

### StateMachine/StateLogger.cs

```csharp
public sealed class StateLogger<TState>
{
    private readonly List<TState> _history = new();
    public IReadOnlyList<TState> History => _history;

    public void OnChanged(TState s) => _history.Add(s);
}
```

### StateMachine/SignupFlowStateMachine.cs

```csharp
public sealed class SignupFlowStateMachine
{
    private readonly SignupData _data;
    private readonly IAccountService _account;
    private readonly StateLogger<SignupStep> _logger;

    public IStateManager<SignupStep, SignupEvent> Machine { get; }

    public SignupFlowStateMachine(SignupData data, IAccountService account)
    {
        _data = data;
        _account = account;
        _logger = new StateLogger<SignupStep>();

        var transitions = new List<Transition<SignupStep, SignupEvent>>
        {
            // Step1 -> Step2 (NEXT)
            new()
            {
                From = SignupStep.Step1_Email,
                When = SignupEvent.Next,
                To   = SignupStep.Step2_Password,
                GuardAsync = async () =>
                {
                    if (string.IsNullOrWhiteSpace(_data.Email))
                        return GuardResult.Deny("이메일을 입력하세요.");
                    if (!_data.Email.Contains("@"))
                        return GuardResult.Deny("이메일 형식이 올바르지 않습니다.");
                    bool available = await _account.CheckEmailAvailableAsync(_data.Email);
                    return available ? GuardResult.Ok() : GuardResult.Deny("이미 사용 중인 이메일입니다.");
                }
            },

            // Step2 -> Step3 (SUBMIT or NEXT)
            new()
            {
                From = SignupStep.Step2_Password,
                When = SignupEvent.Submit,
                To   = SignupStep.Step3_Complete,
                GuardAsync = async () =>
                {
                    if (string.IsNullOrWhiteSpace(_data.Password) || string.IsNullOrWhiteSpace(_data.ConfirmPassword))
                        return GuardResult.Deny("비밀번호를 입력하세요.");
                    if (_data.Password != _data.ConfirmPassword)
                        return GuardResult.Deny("비밀번호가 일치하지 않습니다.");
                    if (_data.Password.Length < 8)
                        return GuardResult.Deny("비밀번호는 8자 이상이어야 합니다.");

                    bool ok = await _account.SignupAsync(_data.Email, _data.Password);
                    return ok ? GuardResult.Ok() : GuardResult.Deny("서버 가입에 실패했습니다.");
                }
            },

            // Back
            new()
            {
                From = SignupStep.Step2_Password, When = SignupEvent.Back, To = SignupStep.Step1_Email
            },
            new()
            {
                From = SignupStep.Step3_Complete, When = SignupEvent.Back, To = SignupStep.Step2_Password
            }
        };

        Machine = new StateMachineCore<SignupStep, SignupEvent>(
            initial: SignupStep.Step1_Email,
            transitions: transitions,
            onStateChanged: _logger.OnChanged);
    }

    public IReadOnlyList<SignupStep> History => (_logger.History);
}
```

---

## 5. 각 Step ViewModel + Shell(SignupFlowViewModel)

### ViewModels/Step1ViewModel.cs

```csharp
using ReactiveUI;

public sealed class Step1ViewModel : ReactiveObject
{
    private readonly SignupData _data;

    public Step1ViewModel(SignupData data)
    {
        _data = data;
    }

    public string Email
    {
        get => _data.Email;
        set
        {
            if (_data.Email != value)
            {
                _data.Email = value;
                this.RaisePropertyChanged();
            }
        }
    }
}
```

### ViewModels/Step2ViewModel.cs

```csharp
using ReactiveUI;

public sealed class Step2ViewModel : ReactiveObject
{
    private readonly SignupData _data;

    public Step2ViewModel(SignupData data)
    {
        _data = data;
    }

    public string Password
    {
        get => _data.Password;
        set { if (_data.Password != value) { _data.Password = value; this.RaisePropertyChanged(); } }
    }

    public string ConfirmPassword
    {
        get => _data.ConfirmPassword;
        set { if (_data.ConfirmPassword != value) { _data.ConfirmPassword = value; this.RaisePropertyChanged(); } }
    }
}
```

### ViewModels/Step3ViewModel.cs

```csharp
public sealed class Step3ViewModel
{
    public string Message => "가입이 완료되었습니다.";
}
```

### ViewModels/SignupFlowViewModel.cs

```csharp
using ReactiveUI;
using System.Reactive;
using System.Threading;
using System.Threading.Tasks;

public sealed class SignupFlowViewModel : ReactiveObject
{
    private readonly SignupFlowStateMachine _flow;
    private readonly SignupData _data;

    public object? CurrentViewModel { get; private set; }
    public string? Error { get; private set; }
    public bool IsBusy { get; private set; }

    public ReactiveCommand<Unit, Unit> NextCommand { get; }
    public ReactiveCommand<Unit, Unit> BackCommand { get; }
    public ReactiveCommand<Unit, Unit> SubmitCommand { get; }

    public SignupFlowViewModel(IAccountService account)
    {
        _data = new SignupData();
        _flow = new SignupFlowStateMachine(_data, account);

        NextCommand   = ReactiveCommand.CreateFromTask(NextAsync);
        BackCommand   = ReactiveCommand.CreateFromTask(BackAsync);
        SubmitCommand = ReactiveCommand.CreateFromTask(SubmitAsync);

        UpdateCurrentView();
    }

    private async Task NextAsync()
    {
        await TransitAsync(SignupEvent.Next);
    }

    private async Task BackAsync()
    {
        await TransitAsync(SignupEvent.Back);
    }

    private async Task SubmitAsync()
    {
        await TransitAsync(SignupEvent.Submit);
    }

    private async Task TransitAsync(SignupEvent e, CancellationToken ct = default)
    {
        Error = null;
        IsBusy = true;
        this.RaisePropertyChanged(nameof(Error));
        this.RaisePropertyChanged(nameof(IsBusy));

        // 가드 실패 사유를 UI로 올리려면 OnDenied 훅이 필요하다.
        // 간단히: 전이 시도 후 상태 변화를 보고 Step 유지 시 에러로 간주하는 전략도 가능.
        var before = _flow.Machine.Current;
        bool ok = await _flow.Machine.SendAsync(e, ct);
        IsBusy = false;

        if (!ok)
        {
            // 전이 후보 없음 또는 모든 가드 실패
            Error = "이동할 수 없습니다. 입력을 확인하세요.";
        }
        else
        {
            if (_flow.Machine.Current == before)
            {
                // 동일 상태면 거부된 것. GuardResult.Deny 사유를 꺼내려면 OnDenied 훅에서 저장해둔다.
                Error = Error ?? "조건을 충족하지 못했습니다.";
            }
        }

        this.RaisePropertyChanged(nameof(Error));
        this.RaisePropertyChanged(nameof(IsBusy));
        UpdateCurrentView();
    }

    private void UpdateCurrentView()
    {
        CurrentViewModel = _flow.Machine.Current switch
        {
            SignupStep.Step1_Email    => new Step1ViewModel(_data),
            SignupStep.Step2_Password => new Step2ViewModel(_data),
            SignupStep.Step3_Complete => new Step3ViewModel(),
            _ => null
        };
        this.RaisePropertyChanged(nameof(CurrentViewModel));
    }
}
```

> 가드 실패 사유를 뷰모델에 전달하려면 `Transition.OnDeniedAsync = reason => { _lastError = reason; }`와 같은 저장소를 두고, `TransitAsync` 끝에서 `Error = _lastError`로 반영하는 패턴이 더 명확하다.

---

## 6. 뷰 구성과 DataTemplate 바인딩

### Views/Step1View.axaml

```xml
<UserControl xmlns="https://github.com/avaloniaui"
             x:Class="MyApp.Views.Step1View">
  <StackPanel Spacing="8" Margin="16">
    <TextBlock Text="이메일"/>
    <TextBox Text="{Binding Email}" Watermark="example@domain.com"/>
  </StackPanel>
</UserControl>
```

### Views/Step2View.axaml

```xml
<UserControl xmlns="https://github.com/avaloniaui"
             x:Class="MyApp.Views.Step2View">
  <StackPanel Spacing="8" Margin="16">
    <TextBlock Text="비밀번호"/>
    <TextBox Text="{Binding Password}" PasswordChar="*"/>
    <TextBlock Text="비밀번호 확인"/>
    <TextBox Text="{Binding ConfirmPassword}" PasswordChar="*"/>
  </StackPanel>
</UserControl>
```

### Views/Step3View.axaml

```xml
<UserControl xmlns="https://github.com/avaloniaui"
             x:Class="MyApp.Views.Step3View">
  <StackPanel Margin="16">
    <TextBlock Text="{Binding Message}" FontSize="18" />
  </StackPanel>
</UserControl>
```

### Views/SignupFlowView.axaml

```xml
<UserControl xmlns="https://github.com/avaloniaui"
             xmlns:views="clr-namespace:MyApp.Views"
             x:Class="MyApp.Views.SignupFlowView">

  <UserControl.DataTemplates>
    <DataTemplate DataType="vm:Step1ViewModel">
      <views:Step1View/>
    </DataTemplate>
    <DataTemplate DataType="vm:Step2ViewModel">
      <views:Step2View/>
    </DataTemplate>
    <DataTemplate DataType="vm:Step3ViewModel">
      <views:Step3View/>
    </DataTemplate>
  </UserControl.DataTemplates>

  <DockPanel>
    <Border DockPanel.Dock="Bottom" Padding="12">
      <StackPanel Orientation="Horizontal" Spacing="8" HorizontalAlignment="Center">
        <Button Content="뒤로" Command="{Binding BackCommand}"/>
        <Button Content="다음" Command="{Binding NextCommand}"/>
        <Button Content="제출" Command="{Binding SubmitCommand}"/>
        <TextBlock Text="{Binding Error}" Foreground="Red" Margin="12,0,0,0"/>
        <TextBlock Text="처리 중..." IsVisible="{Binding IsBusy}" Margin="8,0,0,0"/>
      </StackPanel>
    </Border>

    <ContentControl Content="{Binding CurrentViewModel}" Margin="16"/>
  </DockPanel>
</UserControl>
```

> 버튼 활성화/비활성 논리는 `Can(...)`로도 가능하다. 예: `IsEnabled="{Binding Path=CanSubmit}"` 같은 파생 속성을 Shell VM에서 노출.

---

## 7. 가드 실패 사유를 UI로 전달하는 방법

위 VM에서 `Error`를 단순 메시지 슬롯으로 사용했다. 더 정교하게 하려면:

- `Transition.OnDeniedAsync = reason => _errors.Enqueue(reason);`
- `TransitAsync` 완료 후 `_errors.TryDequeue(out var msg)` → `Error = msg;`

또는 상태별 에러 컬렉션을 두고 컨트롤 옆에 툴팁/에러 템플릿을 표시한다(Validation 글과 결합).

---

## 8. 고급 전이: 조건부 분기, 스킵, 루프, 타임아웃

복잡한 흐름에서 특정 조건에 따라 **스텝 스킵**이 필요할 수 있다.
예: 이메일이 사내 도메인이면 `Step2_Password`로, 외부 도메인이면 추가 Step `Step2B_ExtraVerification`을 거치도록.

```csharp
new Transition<SignupStep, SignupEvent>
{
    From = SignupStep.Step1_Email,
    When = SignupEvent.Next,
    To   = ShouldGoExtra(_data.Email)
          ? SignupStep.Step2_Password  // false이면 다른 상태로 전이하도록 스위칭
          : SignupStep.Step2_Password, // 데모상 동일하게 두었지만, 실제로는 다른 상태
    GuardAsync = async () => { /* ... */ return GuardResult.Ok(); }
}
```

**타임아웃 가드**: 서버 검증이 지연되면 거부 처리.

```csharp
GuardAsync = async () =>
{
    using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(3));
    try
    {
        bool ok = await _account.CheckEmailAvailableAsync(_data.Email, cts.Token);
        return ok ? GuardResult.Ok() : GuardResult.Deny("이미 사용 중인 이메일입니다.");
    }
    catch (OperationCanceledException)
    {
        return GuardResult.Deny("검증이 시간 초과되었습니다.");
    }
}
```

---

## 9. 상태 유지/복원(직렬화)

위저드가 길거나 앱 재시작 후 이어하기가 필요하면 `SignupData` + 현재 상태를 저장한다.

```csharp
public sealed class FlowSnapshot
{
    public SignupStep Current { get; set; }
    public SignupData Data { get; set; } = new();
}

public static class FlowPersistence
{
    public static string Serialize(FlowSnapshot snap)
        => System.Text.Json.JsonSerializer.Serialize(snap);

    public static FlowSnapshot Deserialize(string json)
        => System.Text.Json.JsonSerializer.Deserialize<FlowSnapshot>(json) ?? new();
}
```

앱 종료 시 스냅샷 저장, 시작 시 복원해 `SignupFlowStateMachine` 초기 상태를 바꿔준다.

---

## 10. DI 구성과 테스트 전략

DI 등록:

```csharp
services.AddSingleton<IAccountService, AccountService>();
services.AddTransient<SignupFlowViewModel>();
```

테스트(중복 검사/가입 성공/실패):

```csharp
[Fact]
public async Task Next_From_Step1_To_Step2_When_Email_Valid_And_Available()
{
    var fake = new Mock<IAccountService>();
    fake.Setup(s => s.CheckEmailAvailableAsync("ok@dom.com", It.IsAny<CancellationToken>()))
        .ReturnsAsync(true);

    var data = new SignupData { Email = "ok@dom.com" };
    var flow = new SignupFlowStateMachine(data, fake.Object);

    (await flow.Machine.SendAsync(SignupEvent.Next)).Should().BeTrue();
    flow.Machine.Current.Should().Be(SignupStep.Step2_Password);
}

[Fact]
public async Task Submit_To_Step3_Fails_When_Server_Signup_Fails()
{
    var fake = new Mock<IAccountService>();
    fake.Setup(s => s.SignupAsync("u@d.com", "weak", It.IsAny<CancellationToken>()))
        .ReturnsAsync(false);

    var data = new SignupData { Email = "u@d.com", Password = "weak", ConfirmPassword = "weak" };
    var flow = new SignupFlowStateMachine(data, fake.Object);

    // Step1 -> Step2는 통과했다고 가정
    // (테스트 짧게 하려면 초기 상태를 Step2로 구성하는 생성자 오버로드도 가능)
    await flow.Machine.SendAsync(SignupEvent.Next);
    var ok = await flow.Machine.SendAsync(SignupEvent.Submit);

    ok.Should().BeFalse();
    flow.Machine.Current.Should().Be(SignupStep.Step2_Password);
}
```

---

## 11. UI/UX 향상: 버튼 상태, 진행 표시, 단축키

- 버튼 활성화: `IsBusy` 동안 `Next`/`Submit` 비활성화
- 단축키: `Enter` → Next/Submit, `Esc` → Back 바인딩
- 진행 바(ProgressBar): 현재 Step/총 Step 바인딩

예: Shell VM에 파생 속성 추가

```csharp
public int StepIndex => _flow.Machine.Current switch
{
    SignupStep.Step1_Email => 1,
    SignupStep.Step2_Password => 2,
    SignupStep.Step3_Complete => 3,
    _ => 0
};

public int StepCount => 3;
```

XAML:

```xml
<StackPanel Orientation="Horizontal" Spacing="6">
  <TextBlock Text="{Binding StepIndex}"/>
  <TextBlock Text="/"/>
  <TextBlock Text="{Binding StepCount}"/>
</StackPanel>
```

---

## 12. 상태 전이 로깅/관측/테레메트리

`StateLogger`로 이력 저장 외에, 전이 시 마다 이벤트를 발행하여 로그/분석에 보낸다.

```csharp
public sealed class TelemetryService
{
    public void TrackTransition(SignupStep from, SignupEvent ev, SignupStep to) { /* ... */ }
}
```

`StateMachineCore.SendAsync`에서 전이 성공 후 호출하도록 확장 가능.

---

## 13. 상태 패턴(객체지향)과의 비교

본 글은 테이블 기반(Transition 리스트) 상태머신이다.
대안: 각 상태를 클래스로 만들고 `Handle(event)`에서 다음 상태를 반환(GoF **State 패턴**).
장점: 상태별 규칙을 클래스에 캡슐화, 다형 확장 용이.
단점: 전이 테이블 가독성이 떨어질 수 있음.
규모/팀 선호에 따라 선택.

간단 스켈레톤:

```csharp
public abstract class SignupState
{
    protected readonly SignupData Data;
    protected readonly IAccountService Account;

    protected SignupState(SignupData d, IAccountService a) { Data = d; Account = a; }

    public abstract Task<SignupState> HandleAsync(SignupEvent ev, CancellationToken ct);
}

public sealed class EmailState : SignupState
{
    public EmailState(SignupData d, IAccountService a) : base(d, a) { }

    public override async Task<SignupState> HandleAsync(SignupEvent ev, CancellationToken ct)
    {
        if (ev == SignupEvent.Next)
        {
            // 가드/검증...
            return new PasswordState(Data, Account);
        }
        return this; // 변화 없음
    }
}
```

---

## 14. 계층형/병렬 상태(HFSM), 타이머 상태

복잡한 플로우에서 **서브머신**(예: 인증 과정 내에서 또 다른 wizard)을 상태 하나로 포함시키거나,
타이머 이벤트(예: 일정 시간 후 자동 Next)를 도입할 수 있다.

- 타이머: `System.Threading.Timer` 또는 Rx `Observable.Timer`로 이벤트를 발행 → `Machine.SendAsync(TimedOut)`
- 병렬: 두 서브 상태를 독립적으로 진행시키고, 특정 동기화 지점에서 합류(복잡 → 상태차트 도구 권장)

---

## 15. 라우팅/딥링크와 상태 진입 제한

URL 쿼리로 직접 `Step2`로 진입하려는 시도가 있을 수 있다(웹/하이브리드).
상태머신 입구에서 “이전 단계 완료 여부”를 가드로 체크하여 **진입 제한**한다.

```csharp
new Transition<SignupStep, SignupEvent>
{
    From = SignupStep.Step1_Email,
    When = SignupEvent.Next,
    To   = SignupStep.Step2_Password,
    GuardAsync = async () =>
    {
        if (string.IsNullOrWhiteSpace(_data.Email)) return GuardResult.Deny("먼저 이메일을 입력하십시오.");
        // ...
        return GuardResult.Ok();
    }
}
```

---

## 16. 검증(ReactiveUI.Validation 등)과 결합

- 각 Step VM에 필드 검증(동기)
- 서버 연동은 Guard(비동기)
- 에러 표현: `:invalid` 스타일 또는 `ToolTip`/`ErrorText` 표시

예: Step2 비밀번호 길이 동기 검증(간단형)

```csharp
public bool CanSubmitLocal =>
    !string.IsNullOrEmpty(Password) &&
    Password == ConfirmPassword &&
    Password.Length >= 8;
```

Shell에서는 `Submit` 클릭 시 먼저 `CanSubmitLocal` 확인 → 실패면 즉시 메시지, 성공이면 Guard로 서버 호출.

---

## 17. 실전 체크리스트

- 가드 실패 사유를 사용자에게 **명확히** 전달
- 재시도 UX: 실패 후 다시 시도 버튼/자동 재시도
- 취소/중단: 비동기 가드/효과에 `CancellationToken` 전달
- 텔레메트리: 상태 체류 시간, 실패율, Drop-off 지점 수집
- 복구: 앱 재시작 시 스냅샷으로 상태·입력 복원

---

## 18. 전체 흐름 요약(개념)

1) Shell VM이 **상태머신을 소유**하고, 상태가 바뀔 때 현재 Step VM을 새로 생성
2) 각 Step VM은 **입력 모델**에 바인딩
3) 버튼(Next/Back/Submit)은 **이벤트**를 상태머신에 보냄
4) 상태머신은 전이 목록에서 일치 항목을 찾아 **가드**를 검사
5) 가드 통과 시 상태 변경 및 **효과** 실행, 실패 시 **사유 반환**
6) Shell VM은 상태/에러/바쁜 상태를 UI에 반영

---

## 19. 결론

- 상태머신을 도입하면 **흐름 제어, 검증, 서버 연동**을 체계화할 수 있다.
- Guard/Effect로 비즈니스 조건과 사이드이펙트를 분리하면 테스트 용이성이 높아진다.
- Shell VM은 오케스트레이터로서 상태머신을 구동하고, View는 **DataTemplate**로 화면을 렌더링한다.
- 운영 단계에서는 **로깅·복구·분석**을 접목해 사용자 여정을 개선하자.

---

## 부록 A) 최소 동작 샘플(조립)

DI 예시:

```csharp
services.AddSingleton<IAccountService, AccountService>();
services.AddTransient<SignupFlowViewModel>();
```

뷰 인스턴스:

```csharp
var vm = Services.GetRequiredService<SignupFlowViewModel>();
var view = new SignupFlowView { DataContext = vm };
```

---

## 부록 B) 상태 차트(간단)

```
[Step1_Email] --Next(이메일 형식/중복 OK)--> [Step2_Password]
[Step2_Password] --Submit(비번 규칙 OK, 서버 가입 OK)--> [Step3_Complete]
[Step2_Password] --Back--> [Step1_Email]
[Step3_Complete] --Back--> [Step2_Password]
```

---

## 부록 C) 확장 아이디어

- 모듈형 플러그인: 외부 모듈이 **자신의 상태/전이**를 등록해 Wizard 확장
- 다국어(i18n): 가드 실패 메시지를 리소스 키로 정의 → `LocalizationService`로 조회
- 접근성(A11y): 상태 변경 시 포커스 이동, 스크린 리더 공지
- E2E 테스트: Playwright/WinAppDriver로 Step 이동 흐름 검증
