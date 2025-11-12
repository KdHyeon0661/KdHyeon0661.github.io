---
layout: post
title: Avalonia - TextBox 양방향 바인딩
date: 2025-01-09 19:20:23 +0900
category: Avalonia
---
# Avalonia에서 TextBox 양방향 바인딩 구현하기

## 1. 목표

- `TextBox.Text` ↔ ViewModel 속성을 **양방향**으로 동기화한다.
- 입력이 바뀌면 **즉시 혹은 포커스 아웃 시** ViewModel이 업데이트되도록 제어한다.
- 입력값 유효성 검사와 표시(붉은 테두리/메시지), 포맷/변환, 디바운스 등 실전 기능을 결합한다.

---

## 2. 예제 구조 (확장판)

```
MyAvaloniaApp/
├── ViewModels/
│   ├── BindingViewModel.cs            # 기본 예제
│   ├── ValidationViewModel.cs         # IDataErrorInfo/INotifyDataErrorInfo
│   ├── ThrottledSearchViewModel.cs    # 디바운스 검색
│   └── Converters.cs                  # IValueConverter 구현
├── Views/
│   ├── BindingView.axaml
│   ├── ValidationView.axaml
│   └── ThrottledSearchView.axaml
└── Styles/
    └── Validation.axaml               # 오류 스타일
```

---

## 3. 가장 기본: ReactiveObject + TextBox TwoWay

초안의 핵심 그대로, Avalonia는 `TextBox.Text` 바인딩이 **기본 TwoWay**이다. 명시적으로 `Mode=TwoWay`를 적어도 무방하다.

### 3.1 ViewModel — `BindingViewModel.cs`

```csharp
using ReactiveUI;

public class BindingViewModel : ReactiveObject
{
    private string _userInput = "초기값입니다.";

    public string UserInput
    {
        get => _userInput;
        set => this.RaiseAndSetIfChanged(ref _userInput, value);
    }
}
```

### 3.2 View — `BindingView.axaml`

```xml
<UserControl xmlns="https://github.com/avaloniaui"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:vm="using:MyAvaloniaApp.ViewModels"
             x:Class="MyAvaloniaApp.Views.BindingView"
             Width="400" Height="200">

  <UserControl.DataContext>
    <vm:BindingViewModel/>
  </UserControl.DataContext>

  <StackPanel Margin="20" Spacing="10">
    <!-- 기본 양방향 바인딩 -->
    <TextBox Watermark="입력하세요" Text="{Binding UserInput}" />

    <!-- 확인 -->
    <TextBlock Text="{Binding UserInput}" FontWeight="Bold"/>
  </StackPanel>
</UserControl>
```

동작
- TextBox에 입력 → `UserInput` 즉시 변경(기본은 `PropertyChanged` 트리거).
- ViewModel에서 값 변경 → TextBox/다른 바인딩도 자동 갱신.

---

## 4. 바인딩 모드/갱신 시점(중요)

Avalonia 바인딩은 **Mode**와 **UpdateSourceTrigger**로 동작 타이밍을 제어할 수 있다.

| 옵션 | 의미 |
|-----|-----|
| `Mode=TwoWay` | View ⇄ ViewModel 동기화 |
| `Mode=OneWay` | ViewModel → View 단방향 |
| `Mode=OneTime` | 최초 한 번만 적용 |
| `UpdateSourceTrigger=PropertyChanged` | 타이핑할 때마다 ViewModel 업데이트(기본) |
| `UpdateSourceTrigger=LostFocus` | 포커스가 빠질 때 한 번 업데이트 |
| `UpdateSourceTrigger=Explicit` | 코드에서 명시적 갱신 시에만 업데이트 |

예시:

```xml
<TextBox Text="{Binding UserInput, Mode=TwoWay, UpdateSourceTrigger=LostFocus}"
         Watermark="포커스 벗어날 때 반영"/>
```

> 폼 검증/대량 연산/네트워크 호출이 입력마다 트리거되면 부담이 크다. 이때는 `LostFocus`나 디바운스를 고려한다.

---

## 5. 값 변환/포맷: 숫자·소수점·문화권 처리

문자열 ↔ 숫자 변환, 소수점 자리 포맷 등은 변환기(`IValueConverter`)를 사용한다.

### 5.1 변환기 — `Converters.cs`

```csharp
using Avalonia.Data.Converters;
using System;
using System.Globalization;

public sealed class IntConverter : IValueConverter
{
    public object? Convert(object? value, Type targetType, object? parameter, CultureInfo culture)
        => value;

    public object? ConvertBack(object? value, Type targetType, object? parameter, CultureInfo culture)
    {
        var s = value?.ToString();
        if (int.TryParse(s, NumberStyles.Integer, culture, out var n))
            return n;
        return Avalonia.Data.BindingNotification.FromError(new FormatException("정수가 아닙니다."), value);
    }
}

public sealed class DoubleFormatConverter : IValueConverter
{
    public string? Format { get; set; } = "F2";

    public object? Convert(object? value, Type targetType, object? parameter, CultureInfo culture)
    {
        if (value is double d)
            return d.ToString(Format, culture);
        return value;
    }

    public object? ConvertBack(object? value, Type targetType, object? parameter, CultureInfo culture)
    {
        var s = value?.ToString();
        if (double.TryParse(s, NumberStyles.Float | NumberStyles.AllowThousands, culture, out var d))
            return d;
        return Avalonia.Data.BindingNotification.FromError(new FormatException("실수가 아닙니다."), value);
    }
}
```

> `BindingNotification.FromError`를 반환하면 바인딩 오류가 컨트롤에 전파되고, Validation 스타일로 표시할 수 있다.

### 5.2 XAML에서 변환기 리소스 등록

```xml
<UserControl xmlns="https://github.com/avaloniaui"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:cvt="using:MyAvaloniaApp.ViewModels"
             ...>
  <UserControl.Resources>
    <cvt:IntConverter x:Key="IntConverter"/>
    <cvt:DoubleFormatConverter x:Key="Double2Converter" Format="F2"/>
  </UserControl.Resources>

  <StackPanel Margin="16" Spacing="8">
    <!-- 정수 -->
    <TextBox Text="{Binding Age, Mode=TwoWay, Converter={StaticResource IntConverter}}"
             Watermark="정수만 입력"/>
    <!-- 소수점 둘째자리까지 표기 -->
    <TextBox Text="{Binding Price, Mode=TwoWay, Converter={StaticResource Double2Converter}}"
             Watermark="예: 12.30"/>
  </StackPanel>
</UserControl>
```

> 간단한 출력 포맷만 필요하면 `StringFormat='F2'` 도 가능하지만, **입력값을 ViewModel 타입으로 변환**하려면 변환기가 안전하다.

---

## 6. 검증(Validation): `IDataErrorInfo` / `INotifyDataErrorInfo`

### 6.1 간단: `IDataErrorInfo`

```csharp
using ReactiveUI;
using System;
using System.ComponentModel;

public class ValidationViewModel : ReactiveObject, IDataErrorInfo
{
    private string _name = "";
    private int _age;

    public string Name
    {
        get => _name;
        set => this.RaiseAndSetIfChanged(ref _name, value);
    }

    public int Age
    {
        get => _age;
        set => this.RaiseAndSetIfChanged(ref _age, value);
    }

    // 전체 오류 요약(여기서는 미사용)
    public string Error => null;

    // 프로퍼티별 오류
    public string this[string columnName]
    {
        get
        {
            return columnName switch
            {
                nameof(Name) => string.IsNullOrWhiteSpace(Name) ? "이름은 필수입니다." : null,
                nameof(Age)  => Age <= 0 ? "나이는 0보다 커야 합니다." : null,
                _ => null
            };
        }
    }
}
```

### 6.2 고급: `INotifyDataErrorInfo`(비동기 검증/여러 오류 메시지)

```csharp
using ReactiveUI;
using System.Collections;
using System.Collections.Generic;
using System.ComponentModel;

public class NotifyErrorsViewModel : ReactiveObject, INotifyDataErrorInfo
{
    private readonly Dictionary<string, List<string>> _errors = new();
    public event EventHandler<DataErrorsChangedEventArgs> ErrorsChanged;

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

    public IEnumerable GetErrors(string propertyName)
        => propertyName != null && _errors.TryGetValue(propertyName, out var list) ? list : null;

    private void ValidateEmail()
    {
        var list = new List<string>();
        if (string.IsNullOrWhiteSpace(Email)) list.Add("이메일은 필수입니다.");
        else if (!Email.Contains("@")) list.Add("이메일 형식이 아닙니다.");

        SetErrors(nameof(Email), list);
    }

    private void SetErrors(string prop, List<string> list)
    {
        if (list?.Count > 0) _errors[prop] = list;
        else _errors.Remove(prop);

        ErrorsChanged?.Invoke(this, new DataErrorsChangedEventArgs(prop));
    }
}
```

### 6.3 오류 표시 스타일 — `Styles/Validation.axaml`

Avalonia는 바인딩 오류/검증 오류가 있을 때 컨트롤에 스타일을 적용할 수 있다. 간단한 예:

```xml
<!-- Styles/Validation.axaml -->
<Styles xmlns="https://github.com/avaloniaui">
  <!-- 바인딩/검증 오류 시 붉은 테두리 -->
  <Style Selector="TextBox:focus:errors, TextBox:errors">
    <Setter Property="BorderBrush" Value="Crimson"/>
    <Setter Property="BorderThickness" Value="2"/>
    <Setter Property="ToolTip.Tip" Value="입력 오류가 있습니다."/>
  </Style>
</Styles>
```

`App.axaml` 또는 `View`에 병합:

```xml
<Application.Styles>
  <FluentTheme/>
  <StyleInclude Source="avares://MyAvaloniaApp/Styles/Validation.axaml"/>
</Application.Styles>
```

### 6.4 검증 화면 — `ValidationView.axaml`

```xml
<UserControl xmlns="https://github.com/avaloniaui"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:vm="using:MyAvaloniaApp.ViewModels"
             x:Class="MyAvaloniaApp.Views.ValidationView">

  <UserControl.DataContext>
    <vm:ValidationViewModel/>
  </UserControl.DataContext>

  <StackPanel Margin="16" Spacing="8">
    <TextBox Watermark="이름" Text="{Binding Name, Mode=TwoWay}"/>
    <TextBox Watermark="나이"  Text="{Binding Age, Mode=TwoWay}"/>
    <TextBlock>
      <Run Text="상태: "/>
      <Run Text="{Binding (Validation.HasErrors), RelativeSource={RelativeSource AncestorType=UserControl}}"/>
    </TextBlock>
  </StackPanel>
</UserControl>
```

> 실무에선 버튼 `CanExecute`를 `HasErrors == false`와 결합해 "검증 성공 시에만 저장"을 구현한다.

---

## 7. 입력 제어: 숫자 전용/마스킹/정규식

WPF의 `PreviewTextInput`와 유사하게 Avalonia도 입력 이벤트를 가로채어 제어할 수 있지만, **MVVM 순수성을 유지**하려면 “검증 + 시각적 피드백” 권장. 그래도 하드 제약이 필요하면 다음과 같이 **부작용 최소화**로 구현한다.

```csharp
// 숫자 외 입력 차단(간단 예시) — 코드비하인드/AttachedBehavior 로도 분리 가능
using Avalonia.Controls;
using Avalonia.Input;

public static class TextBoxBehaviors
{
    public static void AttachNumericOnly(TextBox tb)
    {
        tb.AddHandler(TextInputEvent, (sender, e) =>
        {
            if (e.Text != null)
            {
                foreach (var ch in e.Text)
                {
                    if (!char.IsDigit(ch))
                    {
                        e.Handled = true;
                        break;
                    }
                }
            }
        }, Avalonia.Interactivity.RoutingStrategies.Tunnel);
    }
}
```

View에서 사용(코드비하인드):

```csharp
public partial class BindingView : UserControl
{
    public BindingView()
    {
        InitializeComponent();
        var numBox = this.FindControl<TextBox>("AgeBox");
        if (numBox != null)
            TextBoxBehaviors.AttachNumericOnly(numBox);
    }
}
```

XAML:

```xml
<TextBox x:Name="AgeBox" Watermark="숫자만" Text="{Binding Age, Mode=TwoWay}"/>
```

> 제한적 필요에만 사용하고, 가능한 한 **검증 + 오류 스타일**로 사용자에게 친절하게 알리는 흐름이 UX상 바람직하다.

---

## 8. 디바운스(Throttle)로 실시간 입력 반응을 제어

검색창처럼 입력이 자주 바뀌는 UI는 **디바운스**로 과도한 처리(네트워크/쿼리)를 줄인다.

### 8.1 ViewModel — `ThrottledSearchViewModel.cs`

```csharp
using ReactiveUI;
using System.Reactive.Linq;
using System.Threading.Tasks;

public sealed class ThrottledSearchViewModel : ReactiveObject
{
    private string _query = "";
    public string Query
    {
        get => _query;
        set => this.RaiseAndSetIfChanged(ref _query, value);
    }

    private string _result = "대기";
    public string Result
    {
        get => _result;
        set => this.RaiseAndSetIfChanged(ref _result, value);
    }

    public ThrottledSearchViewModel()
    {
        // 300ms 동안 추가 타이핑이 없을 때만 검색
        this.WhenAnyValue(vm => vm.Query)
            .Throttle(System.TimeSpan.FromMilliseconds(300))
            .DistinctUntilChanged()
            .SelectMany(async q =>
            {
                if (string.IsNullOrWhiteSpace(q)) return "대기";
                // 실제 비동기 검색 호출로 대체
                await Task.Delay(200); 
                return $"결과: '{q}' 관련 3건";
            })
            .ObserveOn(RxApp.MainThreadScheduler)
            .Subscribe(result => Result = result);
    }
}
```

### 8.2 View — `ThrottledSearchView.axaml`

```xml
<UserControl xmlns="https://github.com/avaloniaui"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:vm="using:MyAvaloniaApp.ViewModels"
             x:Class="MyAvaloniaApp.Views.ThrottledSearchView">
  <UserControl.DataContext>
    <vm:ThrottledSearchViewModel/>
  </UserControl.DataContext>

  <StackPanel Margin="16" Spacing="8">
    <TextBox Watermark="검색어" Text="{Binding Query, Mode=TwoWay, UpdateSourceTrigger=PropertyChanged}"/>
    <TextBlock Text="{Binding Result}" />
  </StackPanel>
</UserControl>
```

> `Throttle` + `DistinctUntilChanged`로 **네트워크/DB 부담**을 크게 줄일 수 있다.

---

## 9. 단위 테스트(뷰 없이 ViewModel만)

뷰 없이도 바인딩 로직(상태 변화)을 검증할 수 있다.

```csharp
using Xunit;
using System.Threading.Tasks;

public class BindingTests
{
    [Fact]
    public void TwoWayBinding_LikeBehavior()
    {
        var vm = new BindingViewModel();
        Assert.Equal("초기값입니다.", vm.UserInput);

        vm.UserInput = "변경";
        Assert.Equal("변경", vm.UserInput);
    }

    [Fact]
    public async Task Throttle_Waits_For_User_To_Stop_Typing()
    {
        var vm = new ThrottledSearchViewModel();
        vm.Query = "A";
        vm.Query = "Ab";
        vm.Query = "Abc";

        await Task.Delay(500); // 300ms + 처리 여유
        Assert.Contains("Abc", vm.Result);
    }
}
```

---

## 10. 자주 겪는 문제와 해결책

- **바인딩이 안 먹는다**  
  네임스페이스(`using:`), 프로퍼티 이름 오타, `ReactiveObject` 상속/`RaiseAndSetIfChanged` 누락 여부 확인. 출력 창의 바인딩 오류 로그를 확인하자.
- **성능**  
  실시간 처리 비용이 크면 `UpdateSourceTrigger=LostFocus`나 `Throttle`을 고려한다.
- **숫자/포맷**  
  단순 표시만이면 `StringFormat`으로 충분. ViewModel 타입 변환까지 하려면 `IValueConverter` 사용.
- **검증 UX**  
  입력 차단보다는 “허용 + 오류 표시”가 UX상 자연스러울 때가 많다. 저장/전송 버튼의 `CanExecute`와 결합하라.
- **WPF에서 넘어올 때**  
  Avalonia의 바인딩 구문/개념은 유사하지만, `ListView+GridViewColumn` 같은 일부 패턴은 다르다. TextBox/Validation은 개념적으로 거의 동일하게 쓸 수 있다.

---

## 11. 핵심 요약 표

| 주제 | 권장 패턴 |
|------|-----------|
| 양방향 바인딩 | `Text="{Binding Prop}"` 기본 TwoWay |
| 갱신 시점 | `UpdateSourceTrigger`로 `PropertyChanged`/`LostFocus` 제어 |
| 변환/포맷 | `IValueConverter`로 안전 변환, `StringFormat`은 출력용 |
| 검증 | `IDataErrorInfo` 또는 `INotifyDataErrorInfo` + 오류 스타일 |
| 입력 제어 | 가능하면 검증/메시지, 필요한 경우 Behavior로 하드 제약 |
| 디바운스 | `Throttle`/`DistinctUntilChanged`로 과부하 방지 |
| 테스트 | ViewModel 단위 테스트로 로직 안정화 |

---

## 12. 결론

본 글은 기본적인 **TextBox ⇄ ViewModel 양방향 바인딩**을 넘어, 실무에서 요구되는 **검증, 변환, 포맷, 디바운스, 입력 제어**까지 통합하는 방법을 다뤘다.  
초안의 간결한 예제를 그대로 시작점으로 삼고, 여기서 소개한 확장 포인트를 상황에 맞게 조합하면 **견고하고 유연한 폼 입력 경험**을 만들 수 있다.