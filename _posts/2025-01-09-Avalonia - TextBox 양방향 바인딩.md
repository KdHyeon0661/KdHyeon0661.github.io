---
layout: post
title: Avalonia - TextBox 양방향 바인딩
date: 2025-01-09 19:20:23 +0900
category: Avalonia
---
# Avalonia에서 TextBox 양방향 바인딩 구현하기

## 왜 양방향 바인딩이 필요한가

사용자 입력을 받는 대부분의 UI는 입력값을 ViewModel의 속성과 동기화해야 한다. TextBox에 텍스트를 입력하면 ViewModel의 속성이 갱신되고, 반대로 ViewModel에서 속성값이 변경되면 TextBox에 표시되는 내용도 바뀌어야 한다.

이러한 **양방향 동기화**를 수동으로 구현하려면 TextBox의 이벤트를 구독하고, 속성 변경 이벤트를 다시 TextBox에 반영하는 복잡한 코드가 필요하다. Avalonia의 데이터 바인딩 시스템은 이러한 과정을 선언적으로 처리해준다.

```
ViewModel 속성 ←→ TextBox.Text
```

## 가장 간단한 양방향 바인딩

### ViewModel 준비

ViewModel은 속성 변경을 View에 알릴 수 있어야 한다. ReactiveUI의 `ReactiveObject`를 상속받아 구현한다.

```csharp
using ReactiveUI;

public class BindingViewModel : ReactiveObject
{
    private string _userInput = "초기값";
    
    public string UserInput
    {
        get => _userInput;
        set => this.RaiseAndSetIfChanged(ref _userInput, value);
    }
}
```

CommunityToolkit.Mvvm을 사용한다면 `ObservableObject`와 `[ObservableProperty]`로 더 간결하게 작성할 수 있다.

```csharp
using CommunityToolkit.Mvvm.ComponentModel;

public partial class BindingViewModel : ObservableObject
{
    [ObservableProperty]
    private string _userInput = "초기값";
}
```

### XAML 바인딩

TextBox의 `Text` 속성을 ViewModel의 속성에 바인딩한다. Avalonia에서 `Mode=TwoWay`는 기본값이므로 생략해도 된다.

```xml
<UserControl xmlns="https://github.com/avaloniaui"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:vm="using:MyApp.ViewModels"
             x:Class="MyApp.Views.BindingView">
    
    <UserControl.DataContext>
        <vm:BindingViewModel/>
    </UserControl.DataContext>

    <StackPanel Margin="20" Spacing="10">
        <TextBox Text="{Binding UserInput}" Watermark="입력하세요"/>
        <TextBlock Text="{Binding UserInput}" FontWeight="Bold"/>
    </StackPanel>
</UserControl>
```

이제 TextBox에 입력할 때마다 ViewModel의 `UserInput`이 즉시 갱신되고, TextBlock에도 동일한 값이 표시된다.

## 바인딩 모드와 갱신 시점 제어

### Mode 옵션

| Mode | 설명 |
|------|------|
| TwoWay | View와 ViewModel이 서로 변경을 반영 (기본값) |
| OneWay | ViewModel → View 단방향 |
| OneTime | 초기값만 전달, 이후 변경 무시 |
| OneWayToSource | View → ViewModel 단방향 (드물게 사용) |

### UpdateSourceTrigger

어떤 시점에 View의 변경을 ViewModel에 반영할지 결정한다.

| Trigger | 설명 |
|---------|------|
| PropertyChanged | 텍스트가 변경될 때마다 즉시 반영 (기본값) |
| LostFocus | 포커스가 벗어날 때 한 번 반영 |
| Explicit | 코드에서 명시적으로 갱신할 때만 반영 |

```xml
<!-- 포커스가 벗어날 때만 ViewModel 갱신 -->
<TextBox Text="{Binding UserInput, Mode=TwoWay, UpdateSourceTrigger=LostFocus}"/>
```

`LostFocus`는 실시간 검증이나 네트워크 호출이 불필요하게 자주 발생하는 것을 방지할 때 유용하다.

## 값 변환과 포맷팅

ViewModel의 속성 타입이 문자열이 아닌 경우, TextBox와의 바인딩을 위해 변환이 필요하다.

### 숫자형 변환기

```csharp
using Avalonia.Data.Converters;
using System;
using System.Globalization;

public class IntConverter : IValueConverter
{
    public object? Convert(object? value, Type targetType, object? parameter, CultureInfo culture)
    {
        return value?.ToString();
    }

    public object? ConvertBack(object? value, Type targetType, object? parameter, CultureInfo culture)
    {
        if (int.TryParse(value?.ToString(), NumberStyles.Integer, culture, out var result))
            return result;
        return Avalonia.Data.BindingNotification.FromError(
            new FormatException("정수를 입력하세요."), value);
    }
}
```

변환기에서 오류가 발생하면 `BindingNotification.FromError`를 반환해 검증 스타일을 적용할 수 있다.

### XAML에서 사용

```xml
<UserControl.Resources>
    <local:IntConverter x:Key="IntConverter"/>
</UserControl.Resources>

<TextBox Text="{Binding Age, Converter={StaticResource IntConverter}}"/>
```

### 출력 포맷만 필요한 경우

단순히 출력 형식만 지정하려면 `StringFormat`을 사용한다.

```xml
<TextBlock Text="{Binding Price, StringFormat='{0:F2} 원'}"/>
```

## 입력 검증

### IDataErrorInfo 방식

간단한 오류 메시지를 제공하려면 `IDataErrorInfo` 인터페이스를 구현한다.

```csharp
using ReactiveUI;
using System.ComponentModel;

public class ValidationViewModel : ReactiveObject, IDataErrorInfo
{
    private string _name = "";
    public string Name
    {
        get => _name;
        set => this.RaiseAndSetIfChanged(ref _name, value);
    }

    public string Error => null;

    public string this[string columnName]
    {
        get
        {
            if (columnName == nameof(Name) && string.IsNullOrWhiteSpace(Name))
                return "이름을 입력하세요.";
            return null;
        }
    }
}
```

### INotifyDataErrorInfo 방식

복수의 오류 메시지나 비동기 검증이 필요하면 `INotifyDataErrorInfo`를 사용한다.

```csharp
using ReactiveUI;
using System.Collections;
using System.Collections.Generic;
using System.ComponentModel;

public class EmailViewModel : ReactiveObject, INotifyDataErrorInfo
{
    private readonly Dictionary<string, List<string>> _errors = new();
    public event EventHandler<DataErrorsChangedEventArgs>? ErrorsChanged;

    private string _email = "";
    public string Email
    {
        get => _email;
        set
        {
            this.RaiseAndSetIfChanged(ref _email, value);
            ValidateEmail();
        }
    }

    public bool HasErrors => _errors.Count > 0;

    public IEnumerable GetErrors(string? propertyName)
    {
        if (propertyName != null && _errors.TryGetValue(propertyName, out var list))
            return list;
        return null;
    }

    private void ValidateEmail()
    {
        var errors = new List<string>();
        if (string.IsNullOrWhiteSpace(Email))
            errors.Add("이메일은 필수입니다.");
        else if (!Email.Contains("@"))
            errors.Add("이메일 형식이 아닙니다.");

        if (errors.Count > 0)
            _errors[nameof(Email)] = errors;
        else
            _errors.Remove(nameof(Email));

        ErrorsChanged?.Invoke(this, new DataErrorsChangedEventArgs(nameof(Email)));
    }
}
```

### 오류 시각화

검증 오류가 있을 때 TextBox 테두리를 빨간색으로 표시하는 스타일을 정의한다.

```xml
<Styles xmlns="https://github.com/avaloniaui">
    <Style Selector="TextBox:error">
        <Setter Property="BorderBrush" Value="Crimson"/>
        <Setter Property="BorderThickness" Value="2"/>
        <Setter Property="ToolTip.Tip" Value="입력 오류가 있습니다."/>
    </Style>
</Styles>
```

이 스타일을 App.axaml에 병합하면 모든 TextBox에 적용된다.

## 입력 제한 (숫자 전용)

MVVM 원칙상 ViewModel은 순수한 상태 관리에 집중하고, 입력 제한은 View 계층에서 처리하는 것이 바람직하다. Avalonia에서는 Attached Behavior를 사용해 숫자만 입력받도록 제한할 수 있다.

```csharp
public static class TextBoxExtensions
{
    public static void AttachNumericOnly(TextBox textBox)
    {
        textBox.AddHandler(InputElement.TextInputEvent, (sender, e) =>
        {
            if (e.Text != null && !int.TryParse(e.Text, out _))
                e.Handled = true;
        }, RoutingStrategies.Tunnel);
    }
}
```

뷰의 코드 비하인드에서 호출하거나, Behavior 라이브러리를 활용해 XAML에서 선언적으로 적용할 수 있다.

## 입력 디바운스 (검색창 최적화)

검색어 입력처럼 사용자가 타이핑할 때마다 네트워크 호출을 하면 성능 문제가 생긴다. ReactiveUI의 `Throttle`을 사용해 일정 시간 동안 입력이 없을 때만 검색을 수행한다.

```csharp
public class SearchViewModel : ReactiveObject
{
    private string _query = "";
    public string Query
    {
        get => _query;
        set => this.RaiseAndSetIfChanged(ref _query, value);
    }

    private string _result = "";
    public string Result
    {
        get => _result;
        set => this.RaiseAndSetIfChanged(ref _result, value);
    }

    public SearchViewModel()
    {
        this.WhenAnyValue(x => x.Query)
            .Throttle(TimeSpan.FromMilliseconds(300))
            .DistinctUntilChanged()
            .SelectMany(async q =>
            {
                if (string.IsNullOrWhiteSpace(q)) return "결과 없음";
                await Task.Delay(200); // 실제 API 호출 대체
                return $"'{q}' 검색 결과 3건";
            })
            .ObserveOn(RxApp.MainThreadScheduler)
            .Subscribe(result => Result = result);
    }
}
```

XAML에서는 일반 바인딩과 동일하게 사용한다.

```xml
<TextBox Text="{Binding Query, Mode=TwoWay, UpdateSourceTrigger=PropertyChanged}"/>
<TextBlock Text="{Binding Result}"/>
```

## 단위 테스트

ViewModel은 View에 의존하지 않으므로 단위 테스트가 용이하다.

```csharp
[Fact]
public void TwoWayBinding_UpdatesProperty()
{
    var vm = new BindingViewModel();
    vm.UserInput = "새 값";
    Assert.Equal("새 값", vm.UserInput);
}

[Fact]
public async Task Throttle_ExecutesAfterDelay()
{
    var vm = new SearchViewModel();
    vm.Query = "A";
    vm.Query = "AB";
    vm.Query = "ABC";

    await Task.Delay(500);
    Assert.Contains("ABC", vm.Result);
}
```

## 자주 겪는 문제와 해결

| 문제 | 원인과 해결 |
|------|-------------|
| 바인딩이 동작하지 않음 | 속성명 오타, ViewModel이 INotifyPropertyChanged를 구현하지 않음, DataContext가 올바르게 설정되지 않음 |
| 텍스트 입력 시 느림 | UpdateSourceTrigger=PropertyChanged가 불필요한 연산을 유발. LostFocus로 변경하거나 디바운스 적용 |
| 숫자 변환 시 오류 표시 안됨 | IValueConverter에서 BindingNotification.FromError를 반환해야 스타일이 적용됨 |
| 검증 오류 스타일이 안 나옴 | 스타일 Selector가 :error 조건을 포함하는지 확인. INotifyDataErrorInfo가 올바르게 구현되었는지 검토 |

## 핵심 요약

| 주제 | 권장 방식 |
|------|-----------|
| 기본 양방향 바인딩 | `Text="{Binding Prop}"` (TwoWay 기본) |
| 갱신 시점 제어 | `UpdateSourceTrigger`로 PropertyChanged / LostFocus 선택 |
| 값 변환 | IValueConverter 구현, 오류 시 BindingNotification 반환 |
| 입력 검증 | IDataErrorInfo 또는 INotifyDataErrorInfo + 스타일링 |
| 실시간 검색 최적화 | ReactiveUI의 Throttle + DistinctUntilChanged |
| 단위 테스트 | ViewModel만 독립적으로 테스트 가능 |

## 마무리

Avalonia의 TextBox 양방향 바인딩은 WPF/Xamarin.Forms 등과 거의 동일한 패턴을 제공한다. 기본적인 사용법부터 실전에서 필요한 변환, 검증, 성능 최적화까지 익히면 다양한 입력 시나리오를 깔끔하게 구현할 수 있다.

MVVM 패턴을 일관되게 유지하면서 ViewModel은 비즈니스 로직과 상태 관리에 집중하고, View는 바인딩과 스타일로 사용자 경험을 담당하는 구조를 만들면 유지보수와 테스트가 쉬운 애플리케이션을 개발할 수 있다.