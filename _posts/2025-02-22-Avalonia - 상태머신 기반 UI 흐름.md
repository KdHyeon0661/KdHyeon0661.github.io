---
layout: post
title: Avalonia - 상태머신 기반 UI 흐름
date: 2025-02-17 21:20:23 +0900
category: Avalonia
---
# Avalonia MVVM에서 상태머신 기반 UI 흐름 구현하기

복잡한 사용자 흐름(회원가입 위저드, 주문 단계, 인증 프로세스 등)을 관리할 때 **상태머신(State Machine)** 은 매우 효과적인 패턴입니다. 상태머신을 도입하면 UI의 흐름 제어, 입력 검증, 서버 연동, 오류 처리를 체계적으로 분리할 수 있습니다. 이 글에서는 Avalonia와 MVVM 환경에서 상태머신을 어떻게 설계하고 구현하는지, 초중급 개발자 관점에서 단계별로 설명합니다.

---

## 상태머신의 핵심 요소

상태머신은 다음 다섯 가지 요소로 구성됩니다.

- **상태(State)**: UI가 위치한 단계 (예: 이메일 입력, 비밀번호 입력, 완료)
- **이벤트(Event)**: 사용자 또는 시스템의 입력 (예: 다음, 이전, 제출)
- **전이(Transition)**: 현재 상태와 이벤트, 조건(가드)에 따라 다음 상태로 이동
- **가드(Guard)**: 전이가 가능한지 판단하는 조건 (예: 이메일 형식 확인, 서버 중복 검사)
- **효과(Effect)**: 전이 성공 시 실행되는 부수 작업 (예: 서버 가입 요청, 로깅)

상태 전이 함수를 수식으로 표현하면 다음과 같습니다.

$$
\delta : (S \times E) \times G \to S
$$

여기서 \( S \)는 상태 집합, \( E \)는 이벤트 집합, \( G \)는 가드(참/거짓)입니다.

---

## 예제: 회원가입 위저드

간단한 회원가입 위저드를 예로 들어보겠습니다.

- **Step1**: 이메일 입력 (서버 중복 검사)
- **Step2**: 비밀번호 및 확인 입력
- **Step3**: 완료 화면

사용자는 `다음`, `이전`, `제출` 버튼으로 이동하며, 각 단계에서 유효성 검증이 이루어집니다.

### 프로젝트 구조

```
MyApp/
├── Models/
│   └── SignupData.cs
├── StateMachine/
│   ├── SignupStep.cs
│   ├── SignupEvent.cs
│   ├── Transition.cs
│   ├── StateMachineCore.cs
│   └── SignupFlowStateMachine.cs
├── Services/
│   └── IAccountService.cs
├── ViewModels/
│   ├── Step1ViewModel.cs
│   ├── Step2ViewModel.cs
│   ├── Step3ViewModel.cs
│   └── SignupFlowViewModel.cs
├── Views/
│   ├── Step1View.axaml
│   ├── Step2View.axaml
│   ├── Step3View.axaml
│   └── SignupFlowView.axaml
└── App.axaml.cs
```

---

## 상태와 이벤트 정의

```csharp
// StateMachine/SignupStep.cs
public enum SignupStep
{
    Step1_Email,
    Step2_Password,
    Step3_Complete
}

// StateMachine/SignupEvent.cs
public enum SignupEvent
{
    Next,
    Back,
    Submit
}
```

---

## 모델과 서비스

### 사용자 입력 데이터

```csharp
// Models/SignupData.cs
public class SignupData
{
    public string Email { get; set; } = "";
    public string Password { get; set; } = "";
    public string ConfirmPassword { get; set; } = "";
}
```

### 서버 연동 서비스 (이메일 중복 검사, 가입)

```csharp
// Services/IAccountService.cs
public interface IAccountService
{
    Task<bool> CheckEmailAvailableAsync(string email, CancellationToken ct = default);
    Task<bool> SignupAsync(string email, string password, CancellationToken ct = default);
}
```

---

## 상태머신 코어 (범용)

상태머신의 핵심 로직을 재사용 가능한 클래스로 분리합니다.

### 가드 결과

```csharp
// StateMachine/GuardResult.cs
public readonly struct GuardResult
{
    public bool Allow { get; }
    public string? Reason { get; }

    private GuardResult(bool allow, string? reason)
    {
        Allow = allow;
        Reason = reason;
    }

    public static GuardResult Ok() => new(true, null);
    public static GuardResult Deny(string reason) => new(false, reason);
}
```

### 전이 정의

```csharp
// StateMachine/Transition.cs
public sealed class Transition<TState, TEvent>
{
    public TState From { get; init; }
    public TEvent When { get; init; }
    public TState To { get; init; }
    public Func<Task<GuardResult>>? GuardAsync { get; init; }
    public Func<Task>? EffectAsync { get; init; }
    public Func<string, Task>? OnDeniedAsync { get; init; }
}
```

### 상태머신 코어

```csharp
// StateMachine/StateMachineCore.cs
public sealed class StateMachineCore<TState, TEvent>
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

        foreach (var tr in candidates)
        {
            ct.ThrowIfCancellationRequested();

            var guard = tr.GuardAsync == null
                ? GuardResult.Ok()
                : await tr.GuardAsync();

            if (!guard.Allow)
            {
                if (tr.OnDeniedAsync != null)
                    await tr.OnDeniedAsync(guard.Reason ?? "Guard denied");
                continue;
            }

            // 상태 변경
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

## 도메인 상태머신 작성

실제 회원가입 흐름에 맞춰 상태머신을 구성합니다.

```csharp
// StateMachine/SignupFlowStateMachine.cs
public sealed class SignupFlowStateMachine
{
    private readonly SignupData _data;
    private readonly IAccountService _account;
    private string? _lastError;

    public IStateManager<SignupStep, SignupEvent> Machine { get; }

    public SignupFlowStateMachine(SignupData data, IAccountService account)
    {
        _data = data;
        _account = account;

        var transitions = new List<Transition<SignupStep, SignupEvent>>
        {
            // Step1 → Step2 (Next)
            new()
            {
                From = SignupStep.Step1_Email,
                When = SignupEvent.Next,
                To = SignupStep.Step2_Password,
                GuardAsync = async () =>
                {
                    if (string.IsNullOrWhiteSpace(_data.Email))
                        return GuardResult.Deny("이메일을 입력하세요.");
                    if (!_data.Email.Contains("@"))
                        return GuardResult.Deny("이메일 형식이 올바르지 않습니다.");
                    bool available = await _account.CheckEmailAvailableAsync(_data.Email);
                    return available ? GuardResult.Ok() : GuardResult.Deny("이미 사용 중인 이메일입니다.");
                },
                OnDeniedAsync = reason => { _lastError = reason; return Task.CompletedTask; }
            },

            // Step2 → Step3 (Submit)
            new()
            {
                From = SignupStep.Step2_Password,
                When = SignupEvent.Submit,
                To = SignupStep.Step3_Complete,
                GuardAsync = async () =>
                {
                    if (string.IsNullOrWhiteSpace(_data.Password))
                        return GuardResult.Deny("비밀번호를 입력하세요.");
                    if (_data.Password != _data.ConfirmPassword)
                        return GuardResult.Deny("비밀번호가 일치하지 않습니다.");
                    if (_data.Password.Length < 8)
                        return GuardResult.Deny("비밀번호는 8자 이상이어야 합니다.");

                    bool ok = await _account.SignupAsync(_data.Email, _data.Password);
                    return ok ? GuardResult.Ok() : GuardResult.Deny("서버 가입에 실패했습니다.");
                },
                OnDeniedAsync = reason => { _lastError = reason; return Task.CompletedTask; }
            },

            // Back 전이 (Step2 → Step1, Step3 → Step2)
            new()
            {
                From = SignupStep.Step2_Password,
                When = SignupEvent.Back,
                To = SignupStep.Step1_Email
            },
            new()
            {
                From = SignupStep.Step3_Complete,
                When = SignupEvent.Back,
                To = SignupStep.Step2_Password
            }
        };

        Machine = new StateMachineCore<SignupStep, SignupEvent>(
            initial: SignupStep.Step1_Email,
            transitions: transitions);
    }

    public string? GetLastError() => _lastError;
    public void ClearError() => _lastError = null;
}
```

> 가드 실패 시 `OnDeniedAsync`를 통해 에러 메시지를 저장합니다. 이 메시지는 ViewModel이 가져가 사용자에게 표시할 수 있습니다.

---

## ViewModel 구성

### 각 Step의 ViewModel (입력만 바인딩)

```csharp
// ViewModels/Step1ViewModel.cs
public class Step1ViewModel : ReactiveObject
{
    private readonly SignupData _data;
    public Step1ViewModel(SignupData data) => _data = data;

    public string Email
    {
        get => _data.Email;
        set => this.RaiseAndSetIfChanged(ref _data.Email, value);
    }
}

// Step2ViewModel, Step3ViewModel도 유사하게 구현
```

### 셸 ViewModel (오케스트레이터)

```csharp
// ViewModels/SignupFlowViewModel.cs
public class SignupFlowViewModel : ReactiveObject
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

        NextCommand = ReactiveCommand.CreateFromTask(NextAsync);
        BackCommand = ReactiveCommand.CreateFromTask(BackAsync);
        SubmitCommand = ReactiveCommand.CreateFromTask(SubmitAsync);

        UpdateCurrentView();
    }

    private async Task NextAsync() => await TransitAsync(SignupEvent.Next);
    private async Task BackAsync() => await TransitAsync(SignupEvent.Back);
    private async Task SubmitAsync() => await TransitAsync(SignupEvent.Submit);

    private async Task TransitAsync(SignupEvent ev)
    {
        _flow.ClearError();
        Error = null;
        IsBusy = true;
        this.RaisePropertyChanged(nameof(Error));
        this.RaisePropertyChanged(nameof(IsBusy));

        var before = _flow.Machine.Current;
        bool success = await _flow.Machine.SendAsync(ev);

        IsBusy = false;

        if (!success || _flow.Machine.Current == before)
        {
            Error = _flow.GetLastError() ?? "이동할 수 없습니다.";
        }

        this.RaisePropertyChanged(nameof(Error));
        this.RaisePropertyChanged(nameof(IsBusy));
        UpdateCurrentView();
    }

    private void UpdateCurrentView()
    {
        CurrentViewModel = _flow.Machine.Current switch
        {
            SignupStep.Step1_Email => new Step1ViewModel(_data),
            SignupStep.Step2_Password => new Step2ViewModel(_data),
            SignupStep.Step3_Complete => new Step3ViewModel(),
            _ => null
        };
        this.RaisePropertyChanged(nameof(CurrentViewModel));
    }
}
```

---

## View 구성

### 각 Step의 View

```xml
<!-- Views/Step1View.axaml -->
<UserControl ...>
  <StackPanel Margin="16" Spacing="8">
    <TextBlock Text="이메일"/>
    <TextBox Text="{Binding Email}" Watermark="example@domain.com"/>
  </StackPanel>
</UserControl>
```

### 셸 View (데이터 템플릿으로 Step View 자동 선택)

```xml
<!-- Views/SignupFlowView.axaml -->
<UserControl xmlns="https://github.com/avaloniaui"
             xmlns:views="clr-namespace:MyApp.Views"
             x:Class="MyApp.Views.SignupFlowView">

  <UserControl.DataTemplates>
    <DataTemplate DataType="{x:Type vm:Step1ViewModel}">
      <views:Step1View/>
    </DataTemplate>
    <DataTemplate DataType="{x:Type vm:Step2ViewModel}">
      <views:Step2View/>
    </DataTemplate>
    <DataTemplate DataType="{x:Type vm:Step3ViewModel}">
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

---

## 상태 다이어그램

```
[Step1_Email] --Next(이메일 형식/중복 OK)--> [Step2_Password]
[Step2_Password] --Submit(비밀번호 규칙 OK, 서버 가입 OK)--> [Step3_Complete]
[Step2_Password] --Back--> [Step1_Email]
[Step3_Complete] --Back--> [Step2_Password]
```

---

## 테스트

상태머신의 가장 큰 장점은 테스트 용이성입니다. ViewModel 없이 상태머신만 단독으로 테스트할 수 있습니다.

```csharp
[Fact]
public async Task Next_From_Step1_To_Step2_When_Email_Valid()
{
    var mockService = new Mock<IAccountService>();
    mockService.Setup(s => s.CheckEmailAvailableAsync("test@example.com", It.IsAny<CancellationToken>()))
               .ReturnsAsync(true);

    var data = new SignupData { Email = "test@example.com" };
    var flow = new SignupFlowStateMachine(data, mockService.Object);

    bool result = await flow.Machine.SendAsync(SignupEvent.Next);

    Assert.True(result);
    Assert.Equal(SignupStep.Step2_Password, flow.Machine.Current);
}

[Fact]
public async Task Submit_Fails_When_Password_TooShort()
{
    var mockService = new Mock<IAccountService>();
    var data = new SignupData
    {
        Email = "test@example.com",
        Password = "short",
        ConfirmPassword = "short"
    };
    var flow = new SignupFlowStateMachine(data, mockService.Object);

    // Step1 통과
    await flow.Machine.SendAsync(SignupEvent.Next);
    // Step2 시도
    bool result = await flow.Machine.SendAsync(SignupEvent.Submit);

    Assert.False(result);
    Assert.Equal(SignupStep.Step2_Password, flow.Machine.Current);
    Assert.Contains("8자 이상", flow.GetLastError());
}
```

---

## 고급 주제

### 가드 실패 메시지 다국어 지원

`GuardResult.Deny`에 리소스 키를 전달하고, ViewModel에서 `ILocalizer`를 통해 실제 메시지로 변환할 수 있습니다.

### 타임아웃 처리

서버 검증에 타임아웃을 걸고, 시간 초과 시 거부 처리합니다.

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
        return GuardResult.Deny("서버 응답이 지연됩니다. 잠시 후 다시 시도해주세요.");
    }
};
```

### 상태 복원 (앱 재시작 후 이어하기)

상태머신의 현재 상태와 입력 데이터를 직렬화해 저장하고, 앱 시작 시 복원합니다.

```csharp
public sealed class FlowSnapshot
{
    public SignupStep Current { get; set; }
    public SignupData Data { get; set; } = new();
}

public static string Serialize(FlowSnapshot snap) => JsonSerializer.Serialize(snap);
public static FlowSnapshot Deserialize(string json) => JsonSerializer.Deserialize<FlowSnapshot>(json) ?? new();
```

### 로깅 및 분석

상태 변경 시마다 텔레메트리를 전송해 사용자 이탈 지점을 분석할 수 있습니다.

---

## 설계 체크리스트

| 항목 | 설명 |
|------|------|
| 상태 정의 | 명확한 enum 또는 클래스로 정의 |
| 전이 테이블 | 가독성 높게 구성, 가드와 효과 분리 |
| 비동기 처리 | GuardAsync/EffectAsync에서 CancellationToken 지원 |
| 에러 전달 | 가드 실패 시 사용자 메시지 체계적으로 반환 |
| 테스트 | 상태머신 단위 테스트로 흐름 검증 |
| 복구 | 직렬화로 상태 저장/복원 가능하게 설계 |

---

## 결론

상태머신은 복잡한 UI 흐름을 체계적으로 관리하는 강력한 도구입니다. Avalonia MVVM 환경에서 위와 같이 상태머신을 도입하면 **흐름 제어, 입력 검증, 서버 연동, 오류 처리**를 깔끔하게 분리할 수 있습니다. 이 패턴은 회원가입 위저드뿐 아니라 주문 프로세스, 설정 마법사, 인증 흐름 등 다양한 도메인에 적용할 수 있습니다. 초기 설계 비용은 있지만, 유지보수성과 테스트 용이성 측면에서 큰 이점을 얻을 수 있습니다.