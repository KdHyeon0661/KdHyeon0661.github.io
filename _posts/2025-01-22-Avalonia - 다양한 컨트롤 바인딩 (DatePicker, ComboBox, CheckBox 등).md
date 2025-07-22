---
layout: post
title: Avalonia - 다양한 컨트롤 바인딩 (DatePicker, ComboBox, CheckBox 등)
date: 2025-01-22 19:20:23 +0900
category: Avalonia
---
# 🧰 Avalonia MVVM: 다양한 컨트롤 바인딩 (DatePicker, ComboBox, CheckBox 등)

---

## ✅ 목표

- Avalonia에서 기본 컨트롤을 MVVM 아키텍처에 맞게 사용하는 방법 학습
- 양방향 바인딩, 선택 항목 처리, 선택 값 추적 등 컨트롤 별 특징 이해
- 실무 예제 기반 View + ViewModel 구조 제공

---

## 📁 예제 구조

```
MyAvaloniaApp/
├── Views/
│   └── ControlsDemoView.axaml
├── ViewModels/
│   └── ControlsDemoViewModel.cs
```

---

# 📅 1. DatePicker: 날짜 선택 바인딩

## 📄 ControlsDemoView.axaml

```xml
<StackPanel>
    <DatePicker SelectedDate="{Binding SelectedDate}" />
    <TextBlock Text="{Binding SelectedDate, StringFormat='선택한 날짜: {0:yyyy-MM-dd}'}" />
</StackPanel>
```

## 📄 ControlsDemoViewModel.cs

```csharp
using ReactiveUI;
using System;

public class ControlsDemoViewModel : ReactiveObject
{
    private DateTimeOffset? _selectedDate = DateTimeOffset.Now;

    public DateTimeOffset? SelectedDate
    {
        get => _selectedDate;
        set => this.RaiseAndSetIfChanged(ref _selectedDate, value);
    }
}
```

> ✅ `DatePicker.SelectedDate`는 `DateTimeOffset?` 타입입니다. 날짜를 초기화하거나 null도 처리할 수 있습니다.

---

# 🔽 2. ComboBox: 목록에서 선택 바인딩

## 📄 ControlsDemoView.axaml

```xml
<StackPanel>
    <ComboBox Items="{Binding Fruits}"
              SelectedItem="{Binding SelectedFruit}" />
    <TextBlock Text="{Binding SelectedFruit, StringFormat='선택한 과일: {0}'}" />
</StackPanel>
```

## 📄 ControlsDemoViewModel.cs

```csharp
using ReactiveUI;
using System.Collections.ObjectModel;

public class ControlsDemoViewModel : ReactiveObject
{
    public ObservableCollection<string> Fruits { get; } = new()
    {
        "🍎 사과", "🍌 바나나", "🍇 포도", "🍊 오렌지"
    };

    private string? _selectedFruit;

    public string? SelectedFruit
    {
        get => _selectedFruit;
        set => this.RaiseAndSetIfChanged(ref _selectedFruit, value);
    }
}
```

> ✅ `Items`는 바인딩 가능한 목록(ObservableCollection)이고, `SelectedItem`은 사용자가 선택한 값입니다.

---

## 📌 ComboBox에서 객체 선택 바인딩

### 예: 사용자 리스트

```csharp
public class User
{
    public string Name { get; set; }
    public int Id { get; set; }

    public override string ToString() => $"{Name} (ID: {Id})";
}
```

### ViewModel

```csharp
public ObservableCollection<User> Users { get; } = new()
{
    new User { Id = 1, Name = "홍길동" },
    new User { Id = 2, Name = "이순신" }
};

private User? _selectedUser;

public User? SelectedUser
{
    get => _selectedUser;
    set => this.RaiseAndSetIfChanged(ref _selectedUser, value);
}
```

### View

```xml
<ComboBox Items="{Binding Users}" SelectedItem="{Binding SelectedUser}" />
<TextBlock Text="{Binding SelectedUser.Name}" />
```

> ✅ ComboBox는 객체 자체를 바인딩하고, `ToString()` 결과로 표시합니다.

---

# ✅ 3. CheckBox: 논리 값 바인딩

## 📄 ControlsDemoView.axaml

```xml
<StackPanel>
    <CheckBox IsChecked="{Binding IsAccepted}" Content="약관에 동의합니다" />
    <TextBlock Text="{Binding IsAccepted, StringFormat='동의 여부: {0}'}" />
</StackPanel>
```

## 📄 ControlsDemoViewModel.cs

```csharp
private bool _isAccepted;

public bool IsAccepted
{
    get => _isAccepted;
    set => this.RaiseAndSetIfChanged(ref _isAccepted, value);
}
```

> ✅ `IsChecked`는 `bool` 또는 `bool?` (삼상 체크박스) 타입을 바인딩할 수 있습니다.

---

## 🔄 체크박스 여러 개 → 리스트로 바인딩

```xml
<StackPanel>
    <CheckBox Content="메일 수신" IsChecked="{Binding ReceiveEmail}" />
    <CheckBox Content="SMS 수신" IsChecked="{Binding ReceiveSMS}" />
</StackPanel>
```

```csharp
public bool ReceiveEmail { get; set; }
public bool ReceiveSMS { get; set; }
```

> 체크 항목이 많아질 경우 `Dictionary<string, bool>` 또는 별도 모델로 관리하는 것이 좋습니다.

---

## 🧠 팁: Validation 활용

예: 체크박스 미동의 시 버튼 비활성화

```xml
<Button Content="다음 단계"
        IsEnabled="{Binding IsAccepted}" />
```

---

# 🧪 추가 컨트롤 간단 소개

| 컨트롤 | 주요 바인딩 속성 | 설명 |
|--------|------------------|------|
| `TextBox` | `Text` | 문자열 입력 (양방향) |
| `Slider` | `Value` | 숫자 범위 바인딩 |
| `ProgressBar` | `Value`, `Maximum` | 진행 상태 표시 |
| `ToggleSwitch` | `IsChecked` | On/Off 설정 |
| `RadioButton` | `IsChecked`, `GroupName` | 선택 그룹 바인딩 |

---

# ✅ 종합 예제

```xml
<StackPanel Spacing="10">
    <DatePicker SelectedDate="{Binding SelectedDate}" />
    <ComboBox Items="{Binding Fruits}" SelectedItem="{Binding SelectedFruit}" />
    <CheckBox IsChecked="{Binding IsAccepted}" Content="동의" />
    <TextBlock Text="{Binding Summary}" FontWeight="Bold" />
</StackPanel>
```

```csharp
public string Summary =>
    $"날짜: {SelectedDate?.ToString("yyyy-MM-dd")}, " +
    $"과일: {SelectedFruit}, 동의: {(IsAccepted ? "예" : "아니오")}";
```

> ✅ `Summary`는 `WhenAnyValue` 또는 `RaisePropertyChanged()`로 변경 시 동기화할 수 있습니다.
