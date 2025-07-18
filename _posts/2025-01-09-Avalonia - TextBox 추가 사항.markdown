---
layout: post
title: Avalonia - TextBox 심화
date: 2025-01-09 20:20:23 +0900
category: Avalonia
---
# 🧠 Avalonia TextBox 심화: 포커스 이벤트, 입력 포맷, 커맨드 바인딩

---

## 🧩 기본 구조

```
MyAvaloniaApp/
├── ViewModels/
│   └── AdvancedBindingViewModel.cs
├── Views/
│   └── AdvancedBindingView.axaml
```

---

# 1️⃣ 포커스 이벤트 처리 (GotFocus / LostFocus)

## 📄 ViewModel: `AdvancedBindingViewModel.cs`

```csharp
using ReactiveUI;
using System.Reactive;

public class AdvancedBindingViewModel : ReactiveObject
{
    private string _input;
    private string _log;

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
        OnFocusCommand = ReactiveCommand.Create(() => Log = "✏️ 입력창에 포커스 진입");
        OnBlurCommand = ReactiveCommand.Create(() => Log = $"✅ 최종 입력: {Input}");
    }
}
```

---

## 📄 View: `AdvancedBindingView.axaml`

```xml
<UserControl xmlns="https://github.com/avaloniaui"
             x:Class="MyAvaloniaApp.Views.AdvancedBindingView"
             xmlns:vm="clr-namespace:MyAvaloniaApp.ViewModels"
             Width="400" Height="200">

    <UserControl.DataContext>
        <vm:AdvancedBindingViewModel/>
    </UserControl.DataContext>

    <StackPanel Margin="20" Spacing="10">
        <!-- Focus 관련 -->
        <TextBox Text="{Binding Input}"
                 GotFocus="{Binding OnFocusCommand}"
                 LostFocus="{Binding OnBlurCommand}"
                 Watermark="포커스 시 로그" />

        <TextBlock Text="{Binding Log}" Foreground="DarkGreen" />
    </StackPanel>
</UserControl>
```

---

# 2️⃣ 입력 포맷 제어 (숫자만 허용 + 대문자 자동화)

Avalonia의 `TextInputEvent`를 통해 사용자 입력을 직접 제어할 수 있습니다.

## 📄 ViewModel 추가 속성

```csharp
public void FormatInput(ref string text)
{
    // 대문자 변환 & 숫자만 유지
    text = new string(text.ToUpper().Where(char.IsLetterOrDigit).ToArray());
}
```

## 📄 View 수정

```xml
<TextBox Text="{Binding Input}"
         Watermark="숫자 + 영문 대문자만"
         TextInput="TextBox_TextInput"
         LostFocus="{Binding OnBlurCommand}" />
```

## 📄 Code-behind (AdvancedBindingView.axaml.cs)

```csharp
public partial class AdvancedBindingView : UserControl
{
    public AdvancedBindingView()
    {
        InitializeComponent();
    }

    private void TextBox_TextInput(object? sender, TextInputEventArgs e)
    {
        if (DataContext is AdvancedBindingViewModel vm && sender is TextBox tb)
        {
            var text = tb.Text ?? "";
            vm.FormatInput(ref text);
            tb.Text = text;
            tb.CaretIndex = text.Length; // 커서 이동
        }
    }
}
```

---

# 3️⃣ 커맨드와 입력 결합 (Enter 입력 시 실행)

사용자가 `Enter` 키를 입력했을 때 커맨드 실행하기

## 📄 ViewModel에 커맨드 추가

```csharp
public ReactiveCommand<Unit, Unit> SubmitCommand { get; }

public AdvancedBindingViewModel()
{
    SubmitCommand = ReactiveCommand.Create(() => Log = $"🚀 제출됨: {Input}");
}
```

## 📄 View에서 `KeyDown` 이벤트 사용

```xml
<TextBox Text="{Binding Input}"
         Watermark="Enter로 제출"
         KeyDown="TextBox_KeyDown" />
<Button Content="제출" Command="{Binding SubmitCommand}" />
```

## 📄 Code-behind 추가

```csharp
private void TextBox_KeyDown(object? sender, KeyEventArgs e)
{
    if (e.Key == Key.Enter && DataContext is AdvancedBindingViewModel vm)
    {
        vm.SubmitCommand.Execute().Subscribe();
    }
}
```

# ⚙️ Avalonia MVVM 실전 기능: 유효성 검증, 스피너, 포맷 마스킹, 재사용 컨트롤

---

## 🧩 프로젝트 구조 예시

```
MyAvaloniaApp/
├── Models/
│   └── UserForm.cs
├── ViewModels/
│   └── FormViewModel.cs
├── Views/
│   └── FormView.axaml
├── Controls/
│   └── LabeledTextBox.axaml (+ cs)
```

---

# 1️⃣ 폼 전체 유효성 검증 (IValidatableObject 활용)

## 📄 Models/UserForm.cs

```csharp
using System.ComponentModel.DataAnnotations;

public class UserForm
{
    [Required(ErrorMessage = "이름은 필수입니다.")]
    public string Name { get; set; }

    [Range(1, 150, ErrorMessage = "나이는 1 ~ 150 사이여야 합니다.")]
    public int Age { get; set; }

    [RegularExpression(@"^\d{3}-\d{4}-\d{4}$", ErrorMessage = "전화번호 형식이 올바르지 않습니다.")]
    public string Phone { get; set; }
}
```

## 📄 ViewModel: FormViewModel.cs

```csharp
using ReactiveUI;
using System.ComponentModel.DataAnnotations;
using System.Collections.Generic;
using System.Linq;
using System.Reactive;

public class FormViewModel : ReactiveObject
{
    public UserForm Form { get; set; } = new();

    private string _errors;
    public string Errors
    {
        get => _errors;
        set => this.RaiseAndSetIfChanged(ref _errors, value);
    }

    public ReactiveCommand<Unit, Unit> SubmitCommand { get; }

    public FormViewModel()
    {
        SubmitCommand = ReactiveCommand.Create(OnSubmit);
    }

    private void OnSubmit()
    {
        var results = new List<ValidationResult>();
        var context = new ValidationContext(Form);
        bool valid = Validator.TryValidateObject(Form, context, results, true);

        if (valid)
            Errors = "✅ 제출 완료!";
        else
            Errors = string.Join("\n", results.Select(e => $"❌ {e.ErrorMessage}"));
    }
}
```

## 📄 View: FormView.axaml

```xml
<StackPanel Margin="20" Spacing="8">
    <TextBox Text="{Binding Form.Name}" Watermark="이름" />
    <TextBox Text="{Binding Form.Age}" Watermark="나이" />
    <TextBox Text="{Binding Form.Phone}" Watermark="전화번호 (000-0000-0000)" />

    <Button Content="제출" Command="{Binding SubmitCommand}" />
    <TextBlock Text="{Binding Errors}" Foreground="Red" TextWrapping="Wrap"/>
</StackPanel>
```

---

# 2️⃣ 스피너 처리 (IsBusy, ReactiveCommand + IsExecuting)

## 📄 ViewModel 수정

```csharp
private bool _isBusy;
public bool IsBusy
{
    get => _isBusy;
    set => this.RaiseAndSetIfChanged(ref _isBusy, value);
}

public ReactiveCommand<Unit, Unit> SubmitCommand { get; }

public FormViewModel()
{
    SubmitCommand = ReactiveCommand.CreateFromTask(async () =>
    {
        IsBusy = true;
        await Task.Delay(2000); // 작업 시뮬레이션
        IsBusy = false;

        OnSubmit(); // 유효성 검사
    });
}
```

## 📄 View 추가

```xml
<ProgressBar IsIndeterminate="True" IsVisible="{Binding IsBusy}" Height="8" />
```

---

# 3️⃣ 자동 포맷 마스킹 (전화번호)

## 📄 View에서 이벤트 연결

```xml
<TextBox Text="{Binding Form.Phone}" TextInput="OnPhoneInput" />
```

## 📄 Code-behind: FormView.axaml.cs

```csharp
private void OnPhoneInput(object? sender, TextInputEventArgs e)
{
    if (sender is TextBox tb)
    {
        string digits = new string(tb.Text.Where(char.IsDigit).ToArray());

        if (digits.Length >= 11)
            digits = digits[..11];

        string formatted = digits.Length switch
        {
            <= 3 => digits,
            <= 7 => $"{digits[..3]}-{digits[3..]}",
            _ => $"{digits[..3]}-{digits[3..7]}-{digits[7..]}"
        };

        tb.Text = formatted;
        tb.CaretIndex = formatted.Length;
    }
}
```

> ✅ 마스킹은 전화번호 외에도 주민번호, 카드번호 등에 활용 가능!

---

# 4️⃣ 커스텀 컨트롤 재사용 (LabeledTextBox)

## 📄 Controls/LabeledTextBox.axaml

```xml
<UserControl x:Class="MyAvaloniaApp.Controls.LabeledTextBox"
             xmlns="https://github.com/avaloniaui"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">

    <StackPanel>
        <TextBlock Text="{Binding Label}" Margin="0,0,0,4"/>
        <TextBox Text="{Binding Text, Mode=TwoWay}" />
    </StackPanel>
</UserControl>
```

## 📄 Code-behind: LabeledTextBox.axaml.cs

```csharp
public partial class LabeledTextBox : UserControl
{
    public static readonly StyledProperty<string> LabelProperty =
        AvaloniaProperty.Register<LabeledTextBox, string>(nameof(Label));

    public static readonly StyledProperty<string> TextProperty =
        AvaloniaProperty.Register<LabeledTextBox, string>(nameof(Text));

    public string Label
    {
        get => GetValue(LabelProperty);
        set => SetValue(LabelProperty, value);
    }

    public string Text
    {
        get => GetValue(TextProperty);
        set => SetValue(TextProperty, value);
    }

    public LabeledTextBox()
    {
        InitializeComponent();
        DataContext = this;
    }
}
```

## 📄 사용 예시

```xml
<local:LabeledTextBox Label="이름" Text="{Binding Form.Name}" />
<local:LabeledTextBox Label="전화번호" Text="{Binding Form.Phone}" />
```

---

## ✅ 정리

| 기능 | 주요 기술 |
|------|-----------|
| 포커스 이벤트 | `GotFocus` / `LostFocus` + 커맨드 |
| 입력 포맷 제한 | `TextInputEvent` + ViewModel 로직 |
| 커맨드 실행 | `KeyDown`으로 엔터 입력 감지 후 실행 |
| 전체 유효성 검사 | `Validator`, `ValidationResult`, `DataAnnotations` |
| 스피너 표시 | `ProgressBar` + `IsBusy` 상태 |
| 입력 마스킹 | `TextInputEvent` + `TextBox.Text` 포맷팅 |
| 재사용 컴포넌트 | `UserControl` + `AvaloniaProperty` 바인딩 |