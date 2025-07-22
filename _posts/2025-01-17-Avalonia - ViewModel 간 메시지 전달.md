---
layout: post
title: Avalonia - 여러 페이지 구조
date: 2024-12-26 19:20:23 +0900
category: Avalonia
---
# 📡 Avalonia MVVM: ViewModel 간 메시지 전달 (이벤트 / MessageBus)

---

## ✅ 왜 ViewModel 간 통신이 필요한가?

MVVM 구조에서 ViewModel은 ViewModel끼리 **직접 참조하지 않아야** 합니다.  
하지만 다음과 같은 상황에서는 **간접적인 통신**이 필요합니다.

- ✅ 화면 A(ViewModel1)에서 데이터를 입력하면  
     화면 B(ViewModel2)에 반영되어야 할 때
- ✅ 공통 상태(예: 테마, 로그인 정보)를 여러 ViewModel이 구독해야 할 때
- ✅ 이벤트 중심(예: Notification) 구조가 필요할 때

---

## 🧩 대표적인 통신 방식 2가지

| 방식 | 설명 |
|------|------|
| 1️⃣ Event / EventAggregator | .NET의 `event`, ReactiveUI의 `IObservable` 등 사용 |
| 2️⃣ MessageBus (ReactiveUI) | ViewModel 간 메시지를 타입 기반으로 발송/수신 |

---

# 1️⃣ ReactiveUI의 MessageBus

### 📌 MessageBus는 ReactiveUI에서 제공하는 전역 메시지 라우터입니다.

- ViewModel 간 **느슨한 결합(loose coupling)** 가능
- 타입 기반으로 발송하고, 타입 기반으로 구독
- 모든 ViewModel이 특정 형식의 메시지를 수신 가능

---

## 📦 NuGet 패키지 설치

```bash
dotnet add package ReactiveUI
```

---

## 📄 예제 시나리오

- `SettingsViewModel`에서 사용자 이름을 변경
- `DashboardViewModel`이 그 변경을 감지해서 표시

---

## 🧪 1. 메시지 타입 정의

```csharp
public class UserNameChangedMessage
{
    public string NewUserName { get; }

    public UserNameChangedMessage(string newUserName)
    {
        NewUserName = newUserName;
    }
}
```

---

## 📄 2. 메시지 발송: SettingsViewModel.cs

```csharp
using ReactiveUI;
using System.Reactive;
using System.Reactive.Subjects;

public class SettingsViewModel : ReactiveObject
{
    private string _userName;
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
            // MessageBus로 사용자 이름 변경 메시지 전송
            MessageBus.Current.SendMessage(new UserNameChangedMessage(UserName));
        });
    }
}
```

---

## 📄 3. 메시지 수신: DashboardViewModel.cs

```csharp
using ReactiveUI;

public class DashboardViewModel : ReactiveObject
{
    private string _displayUserName = "기본 사용자";

    public string DisplayUserName
    {
        get => _displayUserName;
        set => this.RaiseAndSetIfChanged(ref _displayUserName, value);
    }

    public DashboardViewModel()
    {
        // MessageBus를 통해 메시지 수신
        MessageBus.Current.Listen<UserNameChangedMessage>()
            .Subscribe(msg =>
            {
                DisplayUserName = $"👤 사용자: {msg.NewUserName}";
            });
    }
}
```

> ✅ 구독은 생성자에서 한 번만 설정되며, 이벤트가 발생할 때마다 처리됩니다.

---

## 📄 4. View 바인딩 예시

### DashboardView.axaml

```xml
<TextBlock Text="{Binding DisplayUserName}" FontSize="20" />
```

### SettingsView.axaml

```xml
<StackPanel>
    <TextBox Text="{Binding UserName}" Watermark="사용자 이름 입력" />
    <Button Content="적용" Command="{Binding ApplyCommand}" />
</StackPanel>
```

---

# 2️⃣ Custom Event 방식 (경량, ReactiveUI 사용 안 함)

> 간단한 앱에서는 .NET 이벤트를 사용하는 것도 가능

## 📄 EventAggregator.cs

```csharp
public class EventAggregator
{
    public event Action<string>? UserNameChanged;

    public void PublishUserNameChanged(string name)
    {
        UserNameChanged?.Invoke(name);
    }
}
```

## 📄 SettingsViewModel.cs

```csharp
public class SettingsViewModel
{
    private readonly EventAggregator _events;

    public string UserName { get; set; } = "";

    public SettingsViewModel(EventAggregator events)
    {
        _events = events;
    }

    public void Apply()
    {
        _events.PublishUserNameChanged(UserName);
    }
}
```

## 📄 DashboardViewModel.cs

```csharp
public class DashboardViewModel
{
    public string DisplayUserName { get; set; }

    public DashboardViewModel(EventAggregator events)
    {
        events.UserNameChanged += name =>
        {
            DisplayUserName = $"👤 사용자: {name}";
        };
    }
}
```

> ⚠️ 단점: 수동으로 이벤트를 연결해야 하며, 메시지 타입에 대한 타입 안정성이 떨어집니다.

---

# ✅ 정리

| 방식 | 특징 | 장점 | 단점 |
|------|------|------|------|
| MessageBus (ReactiveUI) | 타입 기반 메시지 발송/수신 | 느슨한 결합, 확장성 좋음 | 학습 필요, 패키지 의존성 |
| EventAggregator (수동) | 단일 서비스 통한 수신 | 구현 간단 | 규모 커지면 유지 어려움 |