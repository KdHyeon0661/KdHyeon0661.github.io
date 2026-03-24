---
layout: post
title: Avalonia - TextBox 심화
date: 2025-01-09 20:20:23 +0900
category: Avalonia
---
# Avalonia TextBox 심화: 포커스, 입력 포맷, 커맨드 바인딩

사용자 입력을 받는 `TextBox`는 단순한 텍스트 입력 이상의 기능을 필요로 합니다. 포커스 진입/이탈 시 로그를 남기거나, 전화번호 형식을 자동으로 맞추거나, Enter 키로 제출하는 등의 동작은 MVVM 패턴과 자연스럽게 결합되어야 합니다. 이 글에서는 Avalonia에서 `TextBox`를 확장하는 다양한 기법을 초중급 개발자 관점에서 정리합니다.

---

## 준비물: NuGet 패키지

이 글의 예제는 다음 패키지를 사용합니다.

```bash
dotnet add package Avalonia
dotnet add package Avalonia.ReactiveUI
dotnet add package Avalonia.Xaml.Interactions
```

`Avalonia.Xaml.Interactions`는 XAML에서 이벤트를 커맨드로 연결할 때 필요합니다. 또한 `App.axaml.cs`에서 `UseReactiveUI()`를 호출해야 ReactiveUI 기능이 활성화됩니다.

---

## 포커스 이벤트를 MVVM으로 연결하기

`GotFocus`와 `LostFocus` 같은 이벤트를 코드 비하인드에서 처리하면 MVVM 원칙을 위반하게 됩니다. 대신 **Avalonia.Xaml.Interactions**의 `EventTriggerBehavior`를 사용해 이벤트를 뷰모델의 커맨드에 바인딩합니다.

**ViewModel**

```csharp
using ReactiveUI;
using System.Reactive;

public class AdvancedBindingViewModel : ReactiveObject
{
    private string _input = "";
    public string Input
    {
        get => _input;
        set => this.RaiseAndSetIfChanged(ref _input, value);
    }

    private string _log = "대기";
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

**View (XAML)**

```xml
<UserControl xmlns="https://github.com/avaloniaui"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:i="using:Avalonia.Xaml.Interactivity"
             xmlns:ia="using:Avalonia.Xaml.Interactions.Core"
             x:Class="MyApp.Views.AdvancedBindingView">
  <StackPanel Margin="20" Spacing="10">
    <TextBox Text="{Binding Input, Mode=TwoWay}"
             Watermark="포커스 시 로그가 업데이트됩니다.">
      <i:Interaction.Behaviors>
        <ia:EventTriggerBehavior EventName="GotFocus">
          <ia:InvokeCommandAction Command="{Binding OnFocusCommand}"/>
        </ia:EventTriggerBehavior>
        <ia:EventTriggerBehavior EventName="LostFocus">
          <ia:InvokeCommandAction Command="{Binding OnBlurCommand}"/>
        </ia:EventTriggerBehavior>
      </i:Interaction.Behaviors>
    </TextBox>
    <TextBlock Text="{Binding Log}" />
  </StackPanel>
</UserControl>
```

이제 포커스가 이동할 때마다 뷰모델의 `Log`가 업데이트됩니다.

---

## 입력 포맷 제어: 세 가지 전략

입력값을 원하는 형식으로 변환하거나 제한하는 방법은 크게 세 가지입니다.

| 전략                  | 설명                                                                 |
|-----------------------|----------------------------------------------------------------------|
| **Converter**         | 바인딩 시 값을 변환. 주로 대소문자 변환이나 특수문자 제거 등에 사용. |
| **Behavior**          | 실시간 입력 이벤트를 가로채 즉시 포맷 적용. 전화번호 마스킹 등에 적합. |
| **Validation**        | 입력 완료 후 오류 메시지 표시. 사용자에게 명확한 피드백을 줌.        |

### Converter로 자동 변환

예를 들어 영문/숫자만 남기고 대문자로 변환하는 컨버터를 만들어 봅니다.

```csharp
public class UpperAlphaNumConverter : IValueConverter
{
    public object? Convert(object? value, Type targetType, object? parameter, CultureInfo culture) => value;

    public object? ConvertBack(object? value, Type targetType, object? parameter, CultureInfo culture)
    {
        var s = value?.ToString() ?? "";
        return new string(s.ToUpper().Where(char.IsLetterOrDigit).ToArray());
    }
}
```

XAML에서 사용:

```xml
<TextBox Text="{Binding Input, Mode=TwoWay, Converter={StaticResource UpperAlphaNumConverter}}"
         Watermark="영문/숫자만, 자동 대문자"/>
```

### Behavior로 실시간 마스킹 (전화번호)

사용자가 입력하는 동안 자동으로 `000-0000-0000` 형식을 만들어 주는 `Behavior`를 구현합니다.

```csharp
public class PhoneMaskBehavior : Behavior<TextBox>
{
    protected override void OnAttached()
    {
        base.OnAttached();
        if (AssociatedObject != null)
            AssociatedObject.AddHandler(InputElement.TextInputEvent, OnTextInput, RoutingStrategies.Tunnel);
    }

    protected override void OnDetaching()
    {
        if (AssociatedObject != null)
            AssociatedObject.RemoveHandler(InputElement.TextInputEvent, OnTextInput);
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

        tb.Text = formatted;
        tb.CaretIndex = formatted.Length;
    }
}
```

XAML에서 적용:

```xml
<TextBox Text="{Binding Phone, Mode=TwoWay}" Watermark="000-0000-0000">
  <i:Interaction.Behaviors>
    <b:PhoneMaskBehavior/>
  </i:Interaction.Behaviors>
</TextBox>
```

> **주의**: IME(한글 등) 조합 중에는 마스킹이 방해될 수 있으므로, 실시간 마스킹은 영문/숫자 입력에 한정하는 것이 좋습니다.

### 간단한 숫자만 입력 Behavior

숫자만 허용하는 `Behavior`도 쉽게 만들 수 있습니다.

```csharp
public class NumericOnlyBehavior : Behavior<TextBox>
{
    protected override void OnAttached()
    {
        base.OnAttached();
        if (AssociatedObject != null)
            AssociatedObject.AddHandler(InputElement.TextInputEvent, OnTextInput, RoutingStrategies.Tunnel);
    }

    protected override void OnDetaching()
    {
        if (AssociatedObject != null)
            AssociatedObject.RemoveHandler(InputElement.TextInputEvent, OnTextInput);
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

---

## Enter 키로 제출하기

`KeyBinding`을 사용하면 코드 비하인드 없이 Enter 키를 커맨드에 연결할 수 있습니다.

**XAML**

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

**ViewModel**

```csharp
public class AdvancedBindingViewModel : ReactiveObject
{
    public ReactiveCommand<Unit, Unit> SubmitCommand { get; }

    public AdvancedBindingViewModel()
    {
        SubmitCommand = ReactiveCommand.Create(() => Log = $"제출됨: {Input}");
    }
}
```

---

## 폼 검증과 제출

`DataAnnotations`를 활용해 사용자 입력을 검증하고, 비동기 제출 시 스피너를 표시하는 예제입니다.

**모델**

```csharp
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

**ViewModel**

```csharp
public class FormViewModel : ReactiveObject
{
    public UserForm Form { get; } = new();

    private string _errors = "";
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
            await Task.Delay(600); // 서버 호출 시뮬레이션

            var results = new List<ValidationResult>();
            var ctx = new ValidationContext(Form);
            bool valid = Validator.TryValidateObject(Form, ctx, results, true);

            Errors = valid ? "제출 완료" : string.Join("\n", results.Select(r => r.ErrorMessage));
        }
        finally
        {
            IsBusy = false;
        }
    }
}
```

**XAML (위의 Behavior와 결합)**

```xml
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
  <ProgressBar IsIndeterminate="True" IsVisible="{Binding IsBusy}" Height="6"/>
  <TextBlock Text="{Binding Errors}" TextWrapping="Wrap"/>
</StackPanel>
```

---

## 디바운스(Throttle)로 검색 입력 최적화

사용자가 타이핑할 때마다 검색 요청을 보내면 서버에 부하가 큽니다. `ReactiveUI`의 `Throttle`을 사용해 일정 시간 동안 입력이 멈추면 검색을 수행합니다.

**ViewModel**

```csharp
public class ThrottledSearchViewModel : ReactiveObject
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
            .Throttle(TimeSpan.FromMilliseconds(300))
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

**XAML**

```xml
<StackPanel Margin="16" Spacing="8">
  <TextBox Text="{Binding Query, Mode=TwoWay, UpdateSourceTrigger=PropertyChanged}"
           Watermark="검색어"/>
  <TextBlock Text="{Binding Result}"/>
</StackPanel>
```

---

## 재사용 가능한 LabeledTextBox 만들기

`UserControl`을 이용해 라벨과 텍스트 박스를 하나로 묶은 컨트롤을 만들 수 있습니다.

**Controls/LabeledTextBox.axaml.cs**

```csharp
using Avalonia;
using Avalonia.Controls;

public partial class LabeledTextBox : UserControl
{
    public static readonly StyledProperty<string?> LabelProperty =
        AvaloniaProperty.Register<LabeledTextBox, string?>(nameof(Label));

    public static readonly StyledProperty<string?> TextProperty =
        AvaloniaProperty.Register<LabeledTextBox, string?>(nameof(Text), defaultBindingMode: BindingMode.TwoWay);

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

**Controls/LabeledTextBox.axaml**

```xml
<UserControl x:Class="MyApp.Controls.LabeledTextBox">
  <StackPanel>
    <TextBlock Text="{Binding Label, RelativeSource={RelativeSource AncestorType=UserControl}}"
               Margin="0,0,0,4"/>
    <TextBox Text="{Binding Text, Mode=TwoWay, RelativeSource={RelativeSource AncestorType=UserControl}}"/>
  </StackPanel>
</UserControl>
```

**사용**

```xml
<local:LabeledTextBox Label="이름" Text="{Binding Form.Name}"/>
<local:LabeledTextBox Label="전화번호" Text="{Binding Form.Phone}"/>
```

---

## 문제 해결 체크리스트

- **이벤트→커맨드가 동작하지 않음**: `Avalonia.Xaml.Interactions` 패키지 설치 여부와 네임스페이스(`xmlns:i`, `xmlns:ia`)가 올바른지 확인합니다.
- **IME 조합 중 마스킹 충돌**: Behavior에서 `TextInput` 대신 `TextChanged` 이벤트를 사용하거나, 조합 완료 후에만 포맷을 적용하는 방식으로 개선할 수 있습니다.
- **바인딩 오류**: `ReactiveObject` 상속과 `RaiseAndSetIfChanged` 호출, 출력 창의 바인딩 로그를 확인합니다.
- **성능**: `UpdateSourceTrigger=PropertyChanged`를 남용하면 입력 시마다 ViewModel이 업데이트됩니다. 검색 등에는 `Throttle`을 적용하고, 일반 필드는 `LostFocus`를 기본으로 사용하는 것이 좋습니다.

---

## 핵심 요약

| 기능                | 구현 방법                                                                 |
|---------------------|--------------------------------------------------------------------------|
| 포커스 이벤트 처리  | `EventTriggerBehavior` + `InvokeCommandAction`                           |
| 입력 포맷           | 간단 변환은 `IValueConverter`, 실시간 마스킹은 `Behavior`                |
| Enter 제출          | `KeyBinding Gesture="Enter" Command="{Binding ...}"`                     |
| 검증                | `DataAnnotations` + `Validator.TryValidateObject`                        |
| 비동기 제출         | `ReactiveCommand.CreateFromTask` + `IsExecuting`/`ThrownExceptions`      |
| 입력 디바운스       | `WhenAnyValue(...).Throttle(...).Subscribe(...)`                         |
| 재사용 가능한 컨트롤 | `UserControl`에 의존성 속성 추가                                         |

---

## 결론

Avalonia의 `TextBox`는 MVVM과 함께 사용할 때 더욱 강력해집니다. 포커스 이벤트, 입력 포맷, 키보드 제스처, 검증, 비동기 작업까지 모두 XAML과 뷰모델만으로 깔끔하게 처리할 수 있습니다. 이 글에서 소개한 패턴들을 프로젝트에 적용하면 코드 재사용성과 테스트 용이성을 크게 향상시킬 수 있습니다.