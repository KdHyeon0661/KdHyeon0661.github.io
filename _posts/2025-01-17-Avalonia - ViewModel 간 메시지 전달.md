---
layout: post
title: Avalonia - ViewModel 간 메시지 전달
date: 2025-01-17 19:20:23 +0900
category: Avalonia
---
# Avalonia MVVM: ViewModel 간 메시지 전달

## 0. 왜 ViewModel 간 통신이 필요한가?

MVVM에서 ViewModel끼리는 **직접 참조를 피해야** 한다. 그러나 다음 상황에서는 간접 통신이 필요하다.

- 화면 A에서 갱신한 설정/입력값을 화면 B에 반영
- 공통 상태(로그인, 테마, 권한, 네트워크 상태)를 다수 VM이 구독
- 포그라운드/백그라운드 작업의 진행률, 알림 브로드캐스트
- 느슨한 결합의 **플러그인성/확장성** 확보

이를 위해 ReactiveUI의 **MessageBus** 또는 경량 **EventAggregator** 패턴을 사용한다.

---

## 1. 선택지 비교: Event vs MessageBus

| 항목 | Event / EventAggregator | ReactiveUI MessageBus |
|------|-------------------------|------------------------|
| 결합도 | 비교적 높음(서브스크립션 수동 연결) | 낮음(타입·계약 기반 라우팅) |
| 규모 확장 | 이벤트 수 증가 시 관리 어려움 | 타입/계약 체계화로 확장 용이 |
| 스레딩 | 직접 처리 필요 | ObserveOn/SubscribeOn으로 명시적 제어 |
| 메모리 안전 | 이벤트 해제 누락 시 누수 위험 | `CompositeDisposable` 기반 해제 용이 |
| 테스트 | 목 객체 필요 | 인터페이스/계약 기반으로 테스트 용이 |

규모와 복잡도가 올라갈수록 MessageBus가 유리하다.

---

## 2. ReactiveUI MessageBus 빠른 시작

### 2.1 패키지

```bash
dotnet add package ReactiveUI
```

### 2.2 메시지 타입 정의

```csharp
public sealed class UserNameChangedMessage
{
    public string NewUserName { get; }
    public UserNameChangedMessage(string newUserName) => NewUserName = newUserName;
}
```

> 권장: 메시지는 **불변(immutable)** 으로 정의한다. record도 적합하다.

```csharp
public sealed record ThemeChangedMessage(bool IsDark);
```

### 2.3 발송자(SettingsViewModel)

```csharp
using ReactiveUI;
using System.Reactive;

public sealed class SettingsViewModel : ReactiveObject
{
    private string _userName = "";
    public string UserName
    {
        get => _userName;
        set => this.RaiseAndSetIfChanged(ref _userName, value);
    }

    public ReactiveCommand<Unit, Unit> ApplyCommand { get; }

    public SettingsViewModel()
    {
        ApplyCommand = ReactiveCommand.Create(() =>
        {
            MessageBus.Current.SendMessage(new UserNameChangedMessage(UserName));
        });
    }
}
```

### 2.4 수신자(DashboardViewModel)

```csharp
using ReactiveUI;
using System.Reactive.Disposables;
using System.Reactive.Linq;

public sealed class DashboardViewModel : ReactiveObject, IActivatableViewModel
{
    private string _displayUserName = "기본 사용자";
    public string DisplayUserName
    {
        get => _displayUserName;
        set => this.RaiseAndSetIfChanged(ref _displayUserName, value);
    }

    public ViewModelActivator Activator { get; } = new();

    public DashboardViewModel()
    {
        this.WhenActivated(disposables =>
        {
            MessageBus.Current
                .Listen<UserNameChangedMessage>()
                .ObserveOn(RxApp.MainThreadScheduler)   // UI 쓰레드 보장
                .Subscribe(msg => DisplayUserName = $"사용자: {msg.NewUserName}")
                .DisposeWith(disposables);
        });
    }
}
```

> 핵심: `WhenActivated` + `DisposeWith`로 **수명 관리**와 **누수 방지**, `ObserveOn(MainThread)`로 **UI 스레드 안전**을 동시에 달성한다.

---

## 3. 계약(Contract)으로 채널 분리하기

MessageBus는 **타입 + 계약 문자열**(선택)로 구분한다. 동일 타입을 **기능별로 분리**할 때 유용하다.

```csharp
public static class BusContracts
{
    public const string Profile = "profile";
    public const string Security = "security";
}

// 발송
MessageBus.Current.SendMessage(new UserNameChangedMessage(user), BusContracts.Profile);

// 수신
MessageBus.Current.Listen<UserNameChangedMessage>(BusContracts.Profile)
    .Subscribe(...);
```

> 장점: 타입 충돌 없이 **모듈 단위 채널화** 가능.
> 규칙을 문서화해 팀 내 일관성 유지.

---

## 4. 요청/응답 패턴 구현

단방향 알림만이 아니라 **질의/응답**이 필요할 때가 있다. 두 가지 접근을 제공한다.

### 4.1 코리드 메시지로 응답 스트림 전달

```csharp
public sealed record QueryUserDetail(string UserId, IObserver<UserDetail> ReplyTo);
public sealed record UserDetail(string Id, string Name, string Email);

// 요청자
var reply = new ReplaySubject<UserDetail>(1);
MessageBus.Current.SendMessage(new QueryUserDetail("U-001", reply));
reply.Subscribe(detail => ...);

// 응답자
MessageBus.Current.Listen<QueryUserDetail>()
    .SelectMany(async q =>
    {
        var detail = await _repo.FetchAsync(q.UserId);
        q.ReplyTo.OnNext(detail);
        q.ReplyTo.OnCompleted();
        return Unit.Default;
    })
    .Subscribe();
```

장점: 간단, 스트림 조합 용이.
주의: `ReplyTo`를 반드시 `OnCompleted`로 닫아 수명 누수 방지.

### 4.2 CorrelationId(상관관계 ID) + 단일 응답 버스

```csharp
public sealed record Request<TResponse>(Guid CorrelationId, object Payload);
public sealed record Response<TResponse>(Guid CorrelationId, TResponse Data);

// 요청
var id = Guid.NewGuid();
MessageBus.Current.SendMessage(new Request<UserDetail>(id, "U-001"));
MessageBus.Current.Listen<Response<UserDetail>>()
    .Where(r => r.CorrelationId == id)
    .Take(1)
    .Subscribe(r => ...);

// 응답
MessageBus.Current.Listen<Request<UserDetail>>()
    .SelectMany(async req =>
    {
        var userId = (string)req.Payload;
        var data = await _repo.FetchAsync(userId);
        MessageBus.Current.SendMessage(new Response<UserDetail>(req.CorrelationId, data));
        return Unit.Default;
    })
    .Subscribe();
```

장점: 메시지 타입의 **일관적 패턴화**, 멀티 요청 동시 처리 용이.

---

## 5. 스레딩·디스패처 정석

- **Listen**은 기본적으로 구독 스레드에서 콜백 실행
- UI 업데이트는 반드시 `ObserveOn(RxApp.MainThreadScheduler)`
- 비동기 IO/CPU 바운드 작업 시작은 `SubscribeOn(TaskPoolScheduler.Default)` 권장

예:

```csharp
MessageBus.Current.Listen<ThemeChangedMessage>()
    .SubscribeOn(RxApp.TaskpoolScheduler)
    .SelectMany(async msg =>
    {
        await _theme.ApplyAsync(msg.IsDark);
        return msg;
    })
    .ObserveOn(RxApp.MainThreadScheduler)
    .Subscribe(_ => CurrentThemeName = msg.IsDark ? "Dark" : "Light");
```

---

## 6. 실전 패턴 모음

### 6.1 테마 브로드캐스트

```csharp
public sealed record ThemeChangedMessage(bool IsDark);

// 발송: 토글 뷰모델
MessageBus.Current.SendMessage(new ThemeChangedMessage(IsDark));

// 수신: 여러 뷰모델
MessageBus.Current.Listen<ThemeChangedMessage>()
    .ObserveOn(RxApp.MainThreadScheduler)
    .Subscribe(m => IsDarkUi = m.IsDark);
```

### 6.2 로그인 상태 공유

```csharp
public sealed record AuthStateChanged(bool IsAuthenticated, string? UserId);

// 로그인 성공 시
MessageBus.Current.SendMessage(new AuthStateChanged(true, userId));
// 로그아웃 시
MessageBus.Current.SendMessage(new AuthStateChanged(false, null));
```

### 6.3 진행률(Progress) 브로드캐스트

```csharp
public sealed record TaskProgress(string TaskId, double Percent, string Stage);

MessageBus.Current.SendMessage(new TaskProgress(taskId, 0.3, "Downloading"));
```

UI:

```csharp
MessageBus.Current.Listen<TaskProgress>()
    .Where(p => p.TaskId == _boundTask)
    .ObserveOn(RxApp.MainThreadScheduler)
    .Subscribe(p => { Progress = p.Percent; Stage = p.Stage; });
```

### 6.4 탭/네비게이션 동기화

```csharp
public sealed record NavigateTo(string Route, object? Parameter = null);
MessageBus.Current.SendMessage(new NavigateTo("Settings", null));
```

여러 화면에서 동일 메시지를 수신하여 **라우팅 서비스**를 호출하도록 설계.

---

## 7. 메모리/수명 관리 — 누수 없이 운영하기

- 항상 `WhenActivated` + `DisposeWith` 사용(뷰모델 또는 뷰)
- 장시간 구독은 `IHostedService` 성격의 **앱 스코프 싱글톤**에서 운영
- 파일/네트워크/타이머 결합 시 `CancellationToken`을 메시지에 포함하여 **중단 가능성** 제공

예:

```csharp
public sealed record StartLongJob(Guid JobId, CancellationToken Token);
```

---

## 8. 장애·품질 — 재시도·시간제한·버퍼링

Reactive 스트림 연산자를 결합해 품질을 높인다.

```csharp
MessageBus.Current.Listen<UserNameChangedMessage>()
    .Throttle(TimeSpan.FromMilliseconds(200))      // 타자 입력 폭주 억제
    .DistinctUntilChanged(m => m.NewUserName)
    .SelectMany(m => Observable.FromAsync(() => _repo.SaveAsync(m.NewUserName))
                               .Timeout(TimeSpan.FromSeconds(3))
                               .Retry(2)
                               .Catch<Unit, Exception>(ex => { Log(ex); return Observable.Return(Unit.Default); }))
    .ObserveOn(RxApp.MainThreadScheduler)
    .Subscribe(_ => Saved = true);
```

---

## 9. EventAggregator(경량) 패턴의 안전한 구현

초안의 단순 이벤트는 누수 위험이 있다. **약한 참조(WeakReference)** 또는 **구독 해제 API**를 제공하자.

```csharp
public interface IEventAggregator
{
    IDisposable Subscribe<T>(Action<T> handler);
    void Publish<T>(T evt);
}

public sealed class EventAggregator : IEventAggregator
{
    private readonly Dictionary<Type, List<Delegate>> _handlers = new();

    public IDisposable Subscribe<T>(Action<T> handler)
    {
        var t = typeof(T);
        if (!_handlers.TryGetValue(t, out var list))
            _handlers[t] = list = new();
        list.Add(handler);
        return Disposable.Create(() => list.Remove(handler));
    }

    public void Publish<T>(T evt)
    {
        if (_handlers.TryGetValue(typeof(T), out var list))
            foreach (var d in list.ToArray())
                ((Action<T>)d).Invoke(evt);
    }
}
```

사용:

```csharp
var sub = aggregator.Subscribe<UserNameChangedMessage>(m => ...);
// 뷰/VM 해제 시
sub.Dispose();
```

---

## 10. DI(의존성 주입)와 테스트

MessageBus는 전역 `MessageBus.Current`를 써도 되지만, **인터페이스로 주입**하면 테스트가 쉬워진다.

```csharp
public interface IMessageBusFacade
{
    void Send<T>(T message, string? contract = null);
    IObservable<T> Listen<T>(string? contract = null);
}

public sealed class MessageBusFacade : IMessageBusFacade
{
    public void Send<T>(T message, string? contract = null)
        => MessageBus.Current.SendMessage(message, contract);

    public IObservable<T> Listen<T>(string? contract = null)
        => MessageBus.Current.Listen<T>(contract);
}
```

ViewModel에서:

```csharp
public sealed class SettingsViewModel : ReactiveObject
{
    private readonly IMessageBusFacade _bus;
    public SettingsViewModel(IMessageBusFacade bus) => _bus = bus;

    public void Apply(string name) => _bus.Send(new UserNameChangedMessage(name));
}
```

테스트:

```csharp
public sealed class TestBus : IMessageBusFacade
{
    private readonly Subject<object> _subject = new();
    public void Send<T>(T message, string? contract = null) => _subject.OnNext(message);
    public IObservable<T> Listen<T>(string? contract = null) => _subject.OfType<T>();
}

[Fact]
public void Settings_Apply_Publishes_UserName()
{
    var bus = new TestBus();
    var vm  = new SettingsViewModel(bus);
    string? received = null;

    bus.Listen<UserNameChangedMessage>().Subscribe(m => received = m.NewUserName);
    vm.Apply("Alice");

    Assert.Equal("Alice", received);
}
```

---

## 11. 스코프 분리: 모듈/대화상자/문서별 버스

대규모 앱에서는 `MessageBus.Current` 단일 전역 대신 **스코프별 Bus**가 유용하다.

- 앱 전역: 인증/테마/환경
- 모듈 스코프: 특정 기능군(예: 리포트 편집기)
- 문서/탭 스코프: 동일 타입 메시지를 문서 인스턴스별로 분리

```csharp
public interface IMessageBusScope
{
    IMessageBus Bus { get; }
}

public sealed class DocumentScope : IMessageBusScope
{
    public IMessageBus Bus { get; } = new MessageBus();
}
```

문서 탭마다 `new DocumentScope()`를 생성하여 **독립 채널**을 제공하면 충돌이 없다.

---

## 12. 예제 통합: 설정 화면 → 대시보드·헤더·알림창 동시 갱신

### 12.1 메시지

```csharp
public sealed record SettingsApplied(string UserName, bool IsDark);
```

### 12.2 발송자

```csharp
MessageBus.Current.SendMessage(new SettingsApplied(UserName, IsDark));
```

### 12.3 수신자들

- 대시보드: 사용자명 표시
- 헤더: 사용자명과 테마 토글 반영
- 알림창: 토스트 알림

```csharp
// 공통 구독 헬퍼
IDisposable SubscribeSettings(Action<SettingsApplied> onNext) =>
    MessageBus.Current.Listen<SettingsApplied>()
        .ObserveOn(RxApp.MainThreadScheduler)
        .Subscribe(onNext);
```

각 ViewModel에서 `WhenActivated` 안에 `SubscribeSettings(...)`를 등록하고 `DisposeWith`.

---

## 13. 성능/신뢰성 팁

- 잦은 발송 이벤트는 **`Throttle`/`Sample`** 로 줄인다.
- 네트워크/디스크 작업은 **백그라운드 스레드**에서 실행하고 결과만 UI 스레드에 붙인다.
- 장시간 구독은 앱 종료 시점에 명시 해제되도록 **호스트 서비스**로 묶는다.
- 메시지 남발은 유지보수성을 떨어뜨린다. **DTO와 계약 네이밍 규칙**, **폴더 구조**로 설계도를 갖춘다.

---

## 14. 요약 체크리스트

- 타입·계약 기반 MessageBus로 **느슨한 결합** 확보
- `WhenActivated` + `DisposeWith`로 **누수 방지**
- `ObserveOn(RxApp.MainThreadScheduler)`로 **UI 스레드 안전**
- 계약 문자열로 **채널 분리**, CorrelationId로 **요청/응답 매칭**
- 디바운스/재시도/타임아웃으로 **품질 보강**
- DI로 **테스트 용이성** 확보, 필요 시 **스코프별 버스** 도입

---

## 15. 부록: 최소 동작 샘플 XAML

### DashboardView.axaml

```xml
<UserControl xmlns="https://github.com/avaloniaui"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:vm="using:MyAvaloniaApp.ViewModels"
             x:Class="MyAvaloniaApp.Views.DashboardView">
  <UserControl.DataContext>
    <vm:DashboardViewModel/>
  </UserControl.DataContext>
  <StackPanel Margin="16" Spacing="8">
    <TextBlock Text="{Binding DisplayUserName}" FontSize="20"/>
  </StackPanel>
</UserControl>
```

### SettingsView.axaml

```xml
<UserControl xmlns="https://github.com/avaloniaui"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:vm="using:MyAvaloniaApp.ViewModels"
             x:Class="MyAvaloniaApp.Views.SettingsView">
  <UserControl.DataContext>
    <vm:SettingsViewModel/>
  </UserControl.DataContext>
  <StackPanel Margin="16" Spacing="8">
    <TextBox Text="{Binding UserName}" Watermark="사용자 이름"/>
    <Button Content="적용" Command="{Binding ApplyCommand}"/>
  </StackPanel>
</UserControl>
```

---

## 결론

초안의 **MessageBus 개요**를 넘어, 본 글은 **계약 설계, 스레딩, 수명 관리, 스코프 분리, 요청/응답, 품질 연산자 적용, 테스트·DI**까지 포함한 **실전 운용 전략**을 제시했다.
이 가이드를 토대로 프로젝트 초기에 **메시지 타입/계약·스코프 규칙**을 명확히 합의하면, 화면이 늘어나도 결합도를 낮게 유지하면서 기능을 안정적으로 확장할 수 있다.
