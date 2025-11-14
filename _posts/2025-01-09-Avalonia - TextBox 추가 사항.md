---
layout: post
title: Avalonia - TextBox 심화
date: 2025-01-09 20:20:23 +0900
category: Avalonia
---
# Avalonia TextBox 심화: 포커스 이벤트, 입력 포맷, 커맨드 바인딩

## 준비물: NuGet 패키지

- Avalonia 11.x (Core)
- ReactiveUI (MVVM)
- **Avalonia.Xaml.Interactions** ← 이벤트를 커맨드로 연결할 때 사용

```bash
dotnet add package Avalonia
dotnet add package Avalonia.ReactiveUI
dotnet add package Avalonia.Xaml.Interactions
```

프로젝트 시작부(`Program.cs` 또는 `App.axaml.cs`)에서 `UseReactiveUI()`가 필요할 수 있다.

---

## 포커스 이벤트를 MVVM으로: GotFocus / LostFocus

### ViewModel — `AdvancedBindingViewModel.cs`

```csharp
using ReactiveUI;
using System.Reactive;

public class AdvancedBindingViewModel : ReactiveObject
{
    private string _input = "";
    private string _log = "대기";

    public string Input
    {
        get => _input;
        set => this.RaiseAndSetIfChanged(ref _input, value);
    }

    public string Log
    {
        get => _log;
        set => this.RaiseAndSetIfChanged(ref _log, value);
    }

    public ReactiveCommand<Unit, Unit> OnFocusCommand { get; }
    public ReactiveCommand<Unit, Unit> OnBlurCommand { get; }

    public AdvancedBindingViewModel()
    {
        OnFocusCommand = ReactiveCommand.Create(() => Log = "입력창 포커스 진입");
        OnBlurCommand  = ReactiveCommand.Create(() => Log = $"최종 입력: {Input}");
    }
}
```

### View — `AdvancedBindingView.axaml` (Behaviors로 이벤트→커맨드)

```xml
<UserControl xmlns="https://github.com/avaloniaui"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:vm="using:MyAvaloniaApp.ViewModels"
             xmlns:i="using:Avalonia.Xaml.Interactivity"
             xmlns:ia="using:Avalonia.Xaml.Interactions.Core"
             x:Class="MyAvaloniaApp.Views.AdvancedBindingView"
             Width="420" Height="220">

  <UserControl.DataContext>
    <vm:AdvancedBindingViewModel/>
  </UserControl.DataContext>

  <StackPanel Margin="20" Spacing="10">
    <TextBox Text="{Binding Input, Mode=TwoWay}"
             Watermark="포커스 시 로그가 업데이트됩니다.">
      <i:Interaction.Behaviors>
        <!-- GotFocus → OnFocusCommand -->
        <ia:EventTriggerBehavior EventName="GotFocus">
          <ia:InvokeCommandAction Command="{Binding OnFocusCommand}"/>
        </ia:EventTriggerBehavior>

        <!-- LostFocus → OnBlurCommand -->
        <ia:EventTriggerBehavior EventName="LostFocus">
          <ia:InvokeCommandAction Command="{Binding OnBlurCommand}"/>
        </ia:EventTriggerBehavior>
      </i:Interaction.Behaviors>
    </TextBox>

    <TextBlock Text="{Binding Log}" />
  </StackPanel>
</UserControl>
```

> 이 방식은 **코드비하인드 없이** 순수 MVVM으로 포커스 이벤트를 커맨드에 연결한다.

---

## 입력 포맷 제어: 숫자·대문자·마스킹(전화번호) — 3가지 전략

입력 제어는 보통 ① 변환기(Converter), ② 검증(Validation), ③ 동작(Behavior/이벤트) 중 **목적**에 맞게 고른다.

### 변환기(Converter)로 ViewModel 타입을 안전 변환

- 장점: **순수 MVVM**, 바인딩 오류를 Validation 스타일로 표시 가능
- 단점: 마스킹(중간 하이픈 삽입 등)은 입력 UX가 약간 어색할 수 있음

```csharp
// ViewModels/Converters.cs
using Avalonia.Data;
using Avalonia.Data.Converters;
using System;
using System.Globalization;
using System.Linq;

public sealed class UpperAlphaNumConverter : IValueConverter
{
    public object? Convert(object? value, Type targetType, object? parameter, CultureInfo culture) => value;

    public object? ConvertBack(object? value, Type targetType, object? parameter, CultureInfo culture)
    {
        var s = value?.ToString() ?? "";
        var formatted = new string(s.ToUpper().Where(char.IsLetterOrDigit).ToArray());
        return formatted;
    }
}
```

XAML:

```xml
<UserControl ...
             xmlns:cvt="using:MyAvaloniaApp.ViewModels">
  <UserControl.Resources>
    <cvt:UpperAlphaNumConverter x:Key="UpperAlphaNumConverter"/>
  </UserControl.Resources>

  <TextBox Text="{Binding Input, Mode=TwoWay, Converter={StaticResource UpperAlphaNumConverter}}"
           Watermark="영문/숫자만, 자동 대문자"/>
</UserControl>
```

### Behavior/이벤트로 **마스킹**(전화번호 등) — UX 친화적

- 장점: 사용자가 입력할 때 즉시 포맷이 **자연스럽게 적용**
- 단점: IME(조합 입력)·Caret 이동 처리 등 세심한 구현 필요

```csharp
// Views/Behaviors/PhoneMaskBehavior.cs
using Avalonia.Controls;
using Avalonia.Input;
using Avalonia.Xaml.Interactivity;
using System.Linq;

public sealed class PhoneMaskBehavior : Behavior<TextBox>
{
    protected override void OnAttached()
    {
        base.OnAttached();
        if (AssociatedObject is { } tb)
            tb.AddHandler(InputElement.TextInputEvent, OnTextInput, Avalonia.Interactivity.RoutingStrategies.Tunnel);
    }

    protected override void OnDetaching()
    {
        if (AssociatedObject is { } tb)
            tb.RemoveHandler(InputElement.TextInputEvent, OnTextInput);
        base.OnDetaching();
    }

    private void OnTextInput(object? sender, TextInputEventArgs e)
    {
        if (sender is not TextBox tb) return;

        var digits = new string((tb.Text ?? "").Where(char.IsDigit).ToArray());
        if (digits.Length > 11) digits = digits[..11];

        string formatted = digits.Length switch
        {
            <= 3 => digits,
            <= 7 => $"{digits[..3]}-{digits[3..]}",
            _    => $"{digits[..3]}-{digits[3..7]}-{digits[7..]}"
        };

        var oldLen = tb.Text?.Length ?? 0;
        tb.Text = formatted;
        // 가능한 Caret 보정: 단순 끝으로
        if (formatted.Length >= oldLen)
            tb.CaretIndex = formatted.Length;
    }
}
```

XAML:

```xml
<UserControl ...
             xmlns:b="using:MyAvaloniaApp.Views.Behaviors">
  <TextBox Text="{Binding Phone, Mode=TwoWay}" Watermark="000-0000-0000">
    <i:Interaction.Behaviors>
      <b:PhoneMaskBehavior/>
    </i:Interaction.Behaviors>
  </TextBox>
</UserControl>
```

### 숫자 전용(하드 제한) — 간단 Behavior

```csharp
// Views/Behaviors/NumericOnlyBehavior.cs
using Avalonia.Controls;
using Avalonia.Input;
using Avalonia.Xaml.Interactivity;

public sealed class NumericOnlyBehavior : Behavior<TextBox>
{
    protected override void OnAttached()
    {
        base.OnAttached();
        if (AssociatedObject is { } tb)
            tb.AddHandler(InputElement.TextInputEvent, OnTextInput, Avalonia.Interactivity.RoutingStrategies.Tunnel);
    }

    protected override void OnDetaching()
    {
        if (AssociatedObject is { } tb)
            tb.RemoveHandler(InputElement.TextInputEvent, OnTextInput);
        base.OnDetaching();
    }

    private void OnTextInput(object? sender, TextInputEventArgs e)
    {
        if (string.IsNullOrEmpty(e.Text)) return;
        foreach (var ch in e.Text)
        {
            if (!char.IsDigit(ch))
            {
                e.Handled = true;
                return;
            }
        }
    }
}
```

XAML:

```xml
<TextBox Text="{Binding Age, Mode=TwoWay}" Watermark="숫자만">
  <i:Interaction.Behaviors>
    <b:NumericOnlyBehavior/>
  </i:Interaction.Behaviors>
</TextBox>
```

> 권장: 숫자만 강제하기보다, **검증 + 친절한 오류 메시지** UX가 더 자연스러운 경우가 많다. 필요할 때만 하드 제한을 쓰자.

---

## 커맨드와 입력 결합: Enter로 제출하기

### 가장 깔끔한 방법 — KeyBinding

```xml
<UserControl ...>
  <UserControl.KeyBindings>
    <KeyBinding Gesture="Enter" Command="{Binding SubmitCommand}"/>
  </UserControl.KeyBindings>

  <StackPanel>
    <TextBox Text="{Binding Input, Mode=TwoWay}" Watermark="Enter로 제출"/>
    <Button Content="제출" Command="{Binding SubmitCommand}"/>
  </StackPanel>
</UserControl>
```

ViewModel:

```csharp
using ReactiveUI;
using System.Reactive;

public partial class AdvancedBindingViewModel : ReactiveObject
{
    public ReactiveCommand<Unit, Unit> SubmitCommand { get; }

    public AdvancedBindingViewModel()
    {
        SubmitCommand = ReactiveCommand.Create(() => Log = $"제출됨: {Input}");
        // 기존 OnFocusCommand/OnBlurCommand 등과 함께 공존 가능
    }
}
```

> KeyBinding은 **코드비하인드 없이** Enter 제스처를 커맨드에 연결하므로 MVVM 친화적이다.

---

## 폼 유효성 검증 + 스피너 + 제출

### 모델 + ViewModel

```csharp
// Models/UserForm.cs
using System.ComponentModel.DataAnnotations;

public class UserForm
{
    [Required(ErrorMessage = "이름은 필수입니다.")]
    public string Name { get; set; } = "";

    [Range(1, 150, ErrorMessage = "나이는 1~150 사이여야 합니다.")]
    public int Age { get; set; }

    [RegularExpression(@"^\d{3}-\d{4}-\d{4}$", ErrorMessage = "전화번호 형식이 올바르지 않습니다.")]
    public string Phone { get; set; } = "";
}
```

```csharp
// ViewModels/FormViewModel.cs
using ReactiveUI;
using System.Collections.Generic;
using System.ComponentModel.DataAnnotations;
using System.Linq;
using System.Reactive;
using System.Threading.Tasks;

public class FormViewModel : ReactiveObject
{
    public UserForm Form { get; } = new();

    private string _errors = "대기";
    public string Errors
    {
        get => _errors;
        set => this.RaiseAndSetIfChanged(ref _errors, value);
    }

    private bool _isBusy;
    public bool IsBusy
    {
        get => _isBusy;
        set => this.RaiseAndSetIfChanged(ref _isBusy, value);
    }

    public ReactiveCommand<Unit, Unit> SubmitCommand { get; }

    public FormViewModel()
    {
        SubmitCommand = ReactiveCommand.CreateFromTask(SubmitAsync);
    }

    private async Task SubmitAsync()
    {
        IsBusy = true;
        try
        {
            // 서버 호출/파일 I/O 등 비동기 동작을 가정
            await Task.Delay(600);

            var results = new List<ValidationResult>();
            var ctx = new ValidationContext(Form);
            bool valid = Validator.TryValidateObject(Form, ctx, results, true);

            Errors = valid
                ? "제출 완료"
                : string.Join("\n", results.Select(r => r.ErrorMessage));
        }
        finally
        {
            IsBusy = false;
        }
    }
}
```

### View — `FormView.axaml`

```xml
<UserControl xmlns="https://github.com/avaloniaui"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:vm="using:MyAvaloniaApp.ViewModels"
             xmlns:i="using:Avalonia.Xaml.Interactivity"
             xmlns:b="using:MyAvaloniaApp.Views.Behaviors"
             x:Class="MyAvaloniaApp.Views.FormView">

  <UserControl.DataContext>
    <vm:FormViewModel/>
  </UserControl.DataContext>

  <StackPanel Margin="20" Spacing="8">
    <TextBox Text="{Binding Form.Name, Mode=TwoWay}" Watermark="이름"/>

    <TextBox Text="{Binding Form.Age, Mode=TwoWay}" Watermark="나이">
      <i:Interaction.Behaviors>
        <b:NumericOnlyBehavior/>
      </i:Interaction.Behaviors>
    </TextBox>

    <TextBox Text="{Binding Form.Phone, Mode=TwoWay}" Watermark="전화번호(000-0000-0000)">
      <i:Interaction.Behaviors>
        <b:PhoneMaskBehavior/>
      </i:Interaction.Behaviors>
    </TextBox>

    <Button Content="제출" Command="{Binding SubmitCommand}"/>

    <ProgressBar IsIndeterminate="True"
                 IsVisible="{Binding IsBusy}"
                 Height="6"/>

    <TextBlock Text="{Binding Errors}" TextWrapping="Wrap"/>
  </StackPanel>
</UserControl>
```

---

## 재사용 가능한 LabeledTextBox 컨트롤

간단 UserControl로도 충분하지만, 템플릿/스타일 재정의까지 고려하면 **TemplatedControl**이 더 유연하다. 여기서는 간단 버전으로 제시한다.

### 컨트롤

```csharp
// Controls/LabeledTextBox.axaml.cs
using Avalonia;
using Avalonia.Controls;

namespace MyAvaloniaApp.Controls;

public partial class LabeledTextBox : UserControl
{
    public static readonly StyledProperty<string?> LabelProperty =
        AvaloniaProperty.Register<LabeledTextBox, string?>(nameof(Label));

    public static readonly StyledProperty<string?> TextProperty =
        AvaloniaProperty.Register<LabeledTextBox, string?>(nameof(Text), defaultBindingMode: Avalonia.Data.BindingMode.TwoWay);

    public string? Label
    {
        get => GetValue(LabelProperty);
        set => SetValue(LabelProperty, value);
    }

    public string? Text
    {
        get => GetValue(TextProperty);
        set => SetValue(TextProperty, value);
    }

    public LabeledTextBox() => InitializeComponent();
}
```

```xml
<!-- Controls/LabeledTextBox.axaml -->
<UserControl x:Class="MyAvaloniaApp.Controls.LabeledTextBox"
             xmlns="https://github.com/avaloniaui"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">

  <StackPanel>
    <TextBlock Text="{Binding Label, RelativeSource={RelativeSource AncestorType=UserControl}}"
               Margin="0,0,0,4"/>
    <TextBox Text="{Binding Text, Mode=TwoWay, RelativeSource={RelativeSource AncestorType=UserControl}}"/>
  </StackPanel>
</UserControl>
```

### 사용

```xml
<local:LabeledTextBox Label="이름" Text="{Binding Form.Name}"/>
<local:LabeledTextBox Label="전화번호" Text="{Binding Form.Phone}"/>
```

---

## 디바운스 검색(보너스): TextBox 입력 폭주 억제

```csharp
// ViewModels/ThrottledSearchViewModel.cs
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
        this.WhenAnyValue(vm => vm.Query)
            .Throttle(System.TimeSpan.FromMilliseconds(300))
            .DistinctUntilChanged()
            .SelectMany(async q =>
            {
                if (string.IsNullOrWhiteSpace(q)) return "대기";
                await Task.Delay(200); // 실제 검색 대체
                return $"검색 결과: '{q}' 3건";
            })
            .ObserveOn(RxApp.MainThreadScheduler)
            .Subscribe(r => Result = r);
    }
}
```

XAML:

```xml
<StackPanel Margin="16" Spacing="8">
  <TextBox Text="{Binding Query, Mode=TwoWay, UpdateSourceTrigger=PropertyChanged}"
           Watermark="검색어"/>
  <TextBlock Text="{Binding Result}"/>
</StackPanel>
```

---

## 문제 해결 체크리스트

- **이벤트→커맨드가 안 된다**: `Avalonia.Xaml.Interactions` 설치/네임스페이스 확인, `EventTriggerBehavior`/`InvokeCommandAction` 사용 여부 확인.
- **IME 조합 입력과 마스킹 충돌**: Behavior에서 `TextChanging`/`TextInput` 모두 시험. 조합 중인 텍스트(플랫폼별 차이) 변경 억제는 피하고, 최종 커밋 순간에만 포맷을 적용해 UX를 높인다.
- **바인딩이 반영되지 않음**: `ReactiveObject` 상속, `RaiseAndSetIfChanged` 호출 여부, 프로퍼티 이름 오타, 바인딩 오류 로그 확인.
- **성능**: `UpdateSourceTrigger=LostFocus` 또는 디바운스(Throttle) 적용. 마스킹 로직은 O(n)로 단순하게.
- **테스트**: `ReactiveCommand.CanExecute`, 디바운스 후 결과 `FirstAsync()` 등으로 View 없이도 검증 가능.

---

## 핵심 요약

| 주제 | 권장 패턴 |
|------|-----------|
| 포커스 이벤트 | `Avalonia.Xaml.Interactions`의 `EventTriggerBehavior` + `InvokeCommandAction` |
| 입력 포맷 | 간단 변환은 `IValueConverter`, 즉시 마스킹은 `Behavior` |
| Enter 제출 | `KeyBinding Gesture="Enter" → Command` |
| 검증/저장 | `DataAnnotations` + `ReactiveCommand`(IsExecuting, ThrownExceptions) |
| 재사용 | `UserControl`/`TemplatedControl`로 공통 TextBox 래핑 |

---

## 전체 예시 뷰: 핵심 요소 한 화면 통합

```xml
<UserControl xmlns="https://github.com/avaloniaui"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:vm="using:MyAvaloniaApp.ViewModels"
             xmlns:i="using:Avalonia.Xaml.Interactivity"
             xmlns:ia="using:Avalonia.Xaml.Interactions.Core"
             xmlns:b="using:MyAvaloniaApp.Views.Behaviors"
             x:Class="MyAvaloniaApp.Views.AdvancedBindingView"
             Width="560" Height="420">

  <UserControl.DataContext>
    <vm:AdvancedBindingViewModel/>
  </UserControl.DataContext>

  <UserControl.KeyBindings>
    <!-- Enter로 제출 -->
    <KeyBinding Gesture="Enter" Command="{Binding SubmitCommand}"/>
  </UserControl.KeyBindings>

  <StackPanel Margin="16" Spacing="10">

    <!-- 포커스 이벤트 + 기본 입력 -->
    <TextBox Text="{Binding Input, Mode=TwoWay}" Watermark="포커스/블러 로그 확인">
      <i:Interaction.Behaviors>
        <ia:EventTriggerBehavior EventName="GotFocus">
          <ia:InvokeCommandAction Command="{Binding OnFocusCommand}"/>
        </ia:EventTriggerBehavior>
        <ia:EventTriggerBehavior EventName="LostFocus">
          <ia:InvokeCommandAction Command="{Binding OnBlurCommand}"/>
        </ia:EventTriggerBehavior>
      </i:Interaction.Behaviors>
    </TextBox>

    <!-- 숫자만 -->
    <TextBox Text="{Binding Age, Mode=TwoWay}" Watermark="숫자만">
      <i:Interaction.Behaviors>
        <b:NumericOnlyBehavior/>
      </i:Interaction.Behaviors>
    </TextBox>

    <!-- 전화번호 마스킹 -->
    <TextBox Text="{Binding Phone, Mode=TwoWay}" Watermark="000-0000-0000">
      <i:Interaction.Behaviors>
        <b:PhoneMaskBehavior/>
      </i:Interaction.Behaviors>
    </TextBox>

    <!-- 상태/제출 -->
    <Button Content="제출" Command="{Binding SubmitCommand}"/>
    <TextBlock Text="{Binding Log}" />
  </StackPanel>
</UserControl>
```

---

## 결론

- **포커스 이벤트**는 `Avalonia.Xaml.Interactions`를 통해 **MVVM스럽게** 커맨드에 연결하자.
- **입력 포맷**은 목적에 맞춰 **Converter(타입 변환)**, **Behavior(마스킹)**, **Validation(오류 메시지)**를 조합하자.
- **Enter 제출**은 `KeyBinding`이 간결하고 테스트 가능하다.
- **ReactiveCommand.IsExecuting/ThrownExceptions**를 활용하면 스피너·오류 처리가 일관된다.
- 이 패턴들을 템플릿화/컨트롤화하면 프로젝트 전반에 **재사용 가능한 UX·코드 품질**을 구축할 수 있다.
