---
layout: post
title: Avalonia - ViewModel 간 메시지 전달
date: 2025-01-17 19:20:23 +0900
category: Avalonia
---
# Avalonia MVVM: ViewModel 간 메시지 전달

## 왜 ViewModel 간 통신이 필요한가

MVVM 패턴에서는 ViewModel 간 **직접 참조**를 피하는 것이 원칙이다. 하지만 여러 화면이 서로 영향을 주어야 하는 상황은 자주 발생한다.

- 설정 화면에서 변경한 사용자 이름을 대시보드 화면에 반영
- 로그인 상태 변경을 여러 ViewModel이 알아야 함
- 백그라운드 작업의 진행률을 알림 영역에 표시

이러한 요구를 해결하려면 ViewModel들이 **느슨하게 통신**할 수 있는 메커니즘이 필요하다. ReactiveUI가 제공하는 **MessageBus**는 이런 용도에 적합한 도구다.

## MessageBus 기본 사용법

### 메시지 타입 정의

먼저 주고받을 메시지의 형태를 클래스로 정의한다. 데이터를 담는 용도이므로 불변(immutable) 객체로 만드는 것이 좋다.

```csharp
// 사용자 이름 변경 알림 메시지
public sealed class UserNameChangedMessage
{
    public string NewUserName { get; }
    public UserNameChangedMessage(string newUserName) => NewUserName = newUserName;
}
```

### 메시지 발송 (Sender)

메시지를 보내는 ViewModel은 `MessageBus.Current.SendMessage`를 호출한다.

```csharp
using ReactiveUI;
using System.Reactive;

public class SettingsViewModel : ReactiveObject
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
            // 변경된 사용자 이름을 메시지로 발송
            MessageBus.Current.SendMessage(new UserNameChangedMessage(UserName));
        });
    }
}
```

### 메시지 수신 (Receiver)

메시지를 받는 ViewModel은 `MessageBus.Current.Listen<T>`로 구독한다. 구독은 ViewModel이 활성화된 동안만 유지하는 것이 안전하다. ReactiveUI의 `WhenActivated`를 사용하면 수명 관리를 쉽게 할 수 있다.

```csharp
using ReactiveUI;
using System.Reactive.Disposables;
using System.Reactive.Linq;

public class DashboardViewModel : ReactiveObject, IActivatableViewModel
{
    private string _displayUserName = "게스트";
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
                .ObserveOn(RxApp.MainThreadScheduler)   // UI 스레드에서 처리
                .Subscribe(msg =>
                {
                    DisplayUserName = msg.NewUserName;
                })
                .DisposeWith(disposables);              // ViewModel 해제 시 구독 해제
        });
    }
}
```

## 구독 해제의 중요성

구독을 해제하지 않으면 ViewModel이 메모리에서 사라져도 계속 메시지를 받아 **메모리 누수**가 발생할 수 있다. 위 예제처럼 `DisposeWith(disposables)`를 사용하면 `WhenActivated` 블록이 종료될 때 자동으로 구독이 해제된다.

## 채널 분리 (Contract)

같은 타입의 메시지를 서로 다른 목적으로 사용해야 할 때가 있다. 예를 들어 `UserNameChangedMessage`를 프로필 설정용과 보안 설정용으로 분리하고 싶다면 계약(contract) 문자열을 추가할 수 있다.

```csharp
// 계약 상수 정의
public static class BusContracts
{
    public const string Profile = "profile";
    public const string Security = "security";
}

// 발송
MessageBus.Current.SendMessage(new UserNameChangedMessage(userName), BusContracts.Profile);

// 수신
MessageBus.Current.Listen<UserNameChangedMessage>(BusContracts.Profile)
    .Subscribe(...);
```

## 실전 예제: 테마 변경 동기화

여러 화면에서 동시에 테마를 변경해야 하는 상황을 생각해보자.

**메시지 정의**
```csharp
public sealed record ThemeChangedMessage(bool IsDark);
```

**테마 설정 ViewModel**
```csharp
public class ThemeSettingsViewModel : ReactiveObject
{
    private bool _isDark;
    public bool IsDark
    {
        get => _isDark;
        set => this.RaiseAndSetIfChanged(ref _isDark, value);
    }

    public ReactiveCommand<Unit, Unit> ApplyThemeCommand { get; }

    public ThemeSettingsViewModel()
    {
        ApplyThemeCommand = ReactiveCommand.Create(() =>
        {
            MessageBus.Current.SendMessage(new ThemeChangedMessage(IsDark));
        });
    }
}
```

**여러 ViewModel에서 수신**
```csharp
// HeaderViewModel, DashboardViewModel 등
this.WhenActivated(disposables =>
{
    MessageBus.Current.Listen<ThemeChangedMessage>()
        .ObserveOn(RxApp.MainThreadScheduler)
        .Subscribe(msg =>
        {
            // 테마 변경에 따른 UI 상태 업데이트
            IsDarkMode = msg.IsDark;
        })
        .DisposeWith(disposables);
});
```

## DI와 테스트

MessageBus는 전역 싱글톤(`MessageBus.Current`)으로 사용해도 되지만, 의존성 주입(DI)을 통해 인터페이스로 감싸면 테스트가 훨씬 쉬워진다.

```csharp
public interface IMessageBus
{
    void Send<T>(T message, string? contract = null);
    IObservable<T> Listen<T>(string? contract = null);
}

public class ReactiveUIMessageBus : IMessageBus
{
    public void Send<T>(T message, string? contract = null) =>
        MessageBus.Current.SendMessage(message, contract);

    public IObservable<T> Listen<T>(string? contract = null) =>
        MessageBus.Current.Listen<T>(contract);
}
```

테스트용으로 가짜 버스를 만들어 사용할 수 있다.

```csharp
public class TestMessageBus : IMessageBus
{
    private readonly Subject<object> _subject = new();

    public void Send<T>(T message, string? contract = null) =>
        _subject.OnNext(message!);

    public IObservable<T> Listen<T>(string? contract = null) =>
        _subject.OfType<T>();
}
```

## 주의사항 및 팁

| 문제 | 해결 방법 |
|------|-----------|
| UI 업데이트가 안 됨 | `ObserveOn(RxApp.MainThreadScheduler)`로 UI 스레드 지정 |
| 메모리 누수 | `WhenActivated` + `DisposeWith`로 구독 해제 |
| 같은 타입 메시지 충돌 | 계약(contract) 문자열로 채널 분리 |
| 메시지 폭주 | `Throttle` / `DistinctUntilChanged` 등으로 빈도 제어 |

## 간단한 요약

- **MessageBus**는 ViewModel 간 느슨한 결합을 가능하게 한다.
- 메시지 클래스를 정의하고 `SendMessage`로 발송, `Listen`으로 수신한다.
- 구독은 반드시 ViewModel 수명과 함께 해제되어야 한다.
- `ObserveOn(RxApp.MainThreadScheduler)`로 UI 스레드 안전을 보장한다.
- 복잡한 앱에서는 계약 문자열이나 DI를 활용해 체계적으로 관리한다.

MVVM에서 메시지 버스를 적절히 활용하면 화면 간 의존성을 낮추고 유지보수성을 높일 수 있다. 처음에는 간단한 알림부터 시작해 점차 필요한 곳에 적용해 보자.