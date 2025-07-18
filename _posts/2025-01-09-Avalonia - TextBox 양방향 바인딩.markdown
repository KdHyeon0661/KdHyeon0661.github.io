---
layout: post
title: Avalonia - TextBox 양방향 바인딩
date: 2025-01-09 19:20:23 +0900
category: Avalonia
---
# 🔄 Avalonia에서 TextBox 양방향 바인딩 구현하기

## ✅ 목표

- `TextBox` 입력을 ViewModel의 속성과 **양방향으로 동기화**
- MVVM 구조에 맞는 데이터 흐름 이해
- 입력값 변경 시 실시간 반응 구현

---

## 📁 구조

```
MyAvaloniaApp/
├── ViewModels/
│   └── BindingViewModel.cs
├── Views/
│   └── BindingView.axaml
```

---

## 🎛️ ViewModel: `BindingViewModel.cs`

```csharp
using ReactiveUI;

public class BindingViewModel : ReactiveObject
{
    private string _userInput;

    public string UserInput
    {
        get => _userInput;
        set => this.RaiseAndSetIfChanged(ref _userInput, value);
    }

    public BindingViewModel()
    {
        UserInput = "초기값입니다.";
    }
}
```

> 🔎 `ReactiveObject`는 Avalonia에서 `INotifyPropertyChanged`를 대체하여 효율적인 상태 업데이트를 도와줍니다.

---

## 🖼️ View: `BindingView.axaml`

```xml
<UserControl xmlns="https://github.com/avaloniaui"
             x:Class="MyAvaloniaApp.Views.BindingView"
             xmlns:vm="clr-namespace:MyAvaloniaApp.ViewModels"
             Width="400" Height="200">
    
    <UserControl.DataContext>
        <vm:BindingViewModel/>
    </UserControl.DataContext>

    <StackPanel Margin="20" Spacing="10">
        <!-- 🔄 양방향 바인딩 -->
        <TextBox Watermark="입력하세요" Text="{Binding UserInput}" />

        <!-- ViewModel 값 확인 -->
        <TextBlock Text="{Binding UserInput}" FontWeight="Bold"/>
    </StackPanel>
</UserControl>
```

---

## 🧪 동작 확인

| 동작 | 결과 |
|------|------|
| TextBox에 입력 | ViewModel의 `UserInput`이 변경됨 |
| ViewModel에서 `UserInput` 변경 | TextBox에 반영됨 |
| `TextBlock`도 자동 반영 | `Text="{Binding UserInput}"` 덕분 |

---

## 💬 커스텀 바인딩 옵션

| 구문 | 설명 |
|------|------|
| `Text="{Binding UserInput}"` | 기본 양방향 (TwoWay) 바인딩 |
| `Text="{Binding UserInput, Mode=TwoWay}"` | 명시적 양방향 바인딩 |
| `Mode=OneWay` | ViewModel → View (단방향) |
| `Mode=OneTime` | 최초만 ViewModel 값 적용 |

---

## ✅ 정리

- Avalonia에서는 TextBox와 ViewModel 속성을 `{Binding}`만으로도 **양방향 바인딩**이 기본
- `ReactiveObject`의 `RaiseAndSetIfChanged()`를 사용하면 자동으로 변경 이벤트가 반영됨
- 바인딩된 속성은 다른 UI 요소와 연동하여 실시간 상태 출력 가능