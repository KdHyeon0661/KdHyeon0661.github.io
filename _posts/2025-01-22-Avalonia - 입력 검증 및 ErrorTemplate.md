---
layout: post
title: Avalonia - 다양한 컨트롤 바인딩 (DatePicker, ComboBox, CheckBox 등)
date: 2025-01-22 19:20:23 +0900
category: Avalonia
---
# ✅ Avalonia MVVM: 입력 검증(Validation)과 ErrorTemplate 구현

---

## 🎯 목표

- ViewModel에서 사용자 입력 값 검증
- View에서 오류 상태 시각화 (`ErrorTemplate`)
- 필드 단위 오류 메시지 바인딩
- 전반적인 유효성 검사(폼 검증) 구현

---

## 🔧 사용하는 기술

| 기술 | 설명 |
|------|------|
| `INotifyDataErrorInfo` | Avalonia의 기본 오류 알림 인터페이스 |
| `IDataValidationPlugin` | Avalonia UI 바인딩 오류 표시용 |
| `ReactiveUI.Validation` | ReactiveUI 기반 Validation 확장 도구 |

---

# 1️⃣ 기본 예제: 이름과 나이를 검증

## 📁 예제 구조

```
Views/
└── ValidationView.axaml
ViewModels/
└── ValidationViewModel.cs
```

---

## 📄 ValidationViewModel.cs

```csharp
using ReactiveUI;
using ReactiveUI.Validation.Extensions;
using ReactiveUI.Validation.Helpers;

public class ValidationViewModel : ReactiveValidationObject
{
    private string _name = "";
    public string Name
    {
        get => _name;
        set => this.RaiseAndSetIfChanged(ref _name, value);
    }

    private int _age;
    public int Age
    {
        get => _age;
        set => this.RaiseAndSetIfChanged(ref _age, value);
    }

    public ValidationViewModel()
    {
        this.ValidationRule(
            vm => vm.Name,
            name => !string.IsNullOrWhiteSpace(name),
            "이름을 입력하세요");

        this.ValidationRule(
            vm => vm.Age,
            age => age >= 0 && age <= 120,
            "나이는 0~120 사이여야 합니다");
    }
}
```

> ✅ `ReactiveValidationObject`는 `INotifyDataErrorInfo`를 구현하여 오류를 자동 처리합니다.

---

## 📄 ValidationView.axaml

```xml
<UserControl xmlns="https://github.com/avaloniaui"
             xmlns:d="https://github.com/avaloniaui"
             xmlns:vm="clr-namespace:MyApp.ViewModels"
             x:Class="MyApp.Views.ValidationView">

  <UserControl.DataContext>
    <vm:ValidationViewModel />
  </UserControl.DataContext>

  <StackPanel Spacing="8">
    <TextBox Text="{Binding Name, Mode=TwoWay, ValidatesOnExceptions=True, NotifyOnValidationError=True}"
             Watermark="이름"
             ToolTip.Tip="{Binding (Validation.Errors)[0].ErrorContent, RelativeSource={RelativeSource Self}}" />

    <TextBox Text="{Binding Age, Mode=TwoWay}"
             Watermark="나이"
             ToolTip.Tip="{Binding (Validation.Errors)[0].ErrorContent, RelativeSource={RelativeSource Self}}" />

    <Button Content="확인" IsEnabled="{Binding HasErrors, Converter={StaticResource InverseBooleanConverter}}" />
  </StackPanel>
</UserControl>
```

> 🧠 `ToolTip` 또는 `ErrorTemplate`을 통해 오류 내용을 사용자에게 보여줄 수 있습니다.  
> ✅ `HasErrors`는 전체 유효성 상태입니다.

---

## 📄 Converter 등록 (InverseBooleanConverter)

```csharp
public class InverseBooleanConverter : IValueConverter
{
    public object Convert(object value, Type targetType, object parameter, CultureInfo culture) =>
        value is bool b ? !b : BindingOperations.DoNothing;

    public object ConvertBack(object value, Type targetType, object parameter, CultureInfo culture) =>
        throw new NotSupportedException();
}
```

```xml
<UserControl.Resources>
  <local:InverseBooleanConverter x:Key="InverseBooleanConverter" />
</UserControl.Resources>
```

---

# 2️⃣ ErrorTemplate 사용 예시 (스타일로 오류 표시)

Avalonia는 직접 `ErrorTemplate`을 제공하지 않지만, **`Validation.ErrorTemplate` 비슷한 시각적 처리는 커스텀 스타일로 구현**합니다.

```xml
<Styles xmlns="https://github.com/avaloniaui">
  <Style Selector="TextBox:invalid">
    <Setter Property="BorderBrush" Value="Red" />
    <Setter Property="BorderThickness" Value="2" />
    <Setter Property="ToolTip.Tip" Value="{Binding (Validation.Errors)[0].ErrorContent, RelativeSource={RelativeSource Self}}" />
  </Style>
</Styles>
```

> ✅ 이 스타일은 `Validation` 오류가 발생한 `TextBox`에 빨간 테두리를 자동 적용합니다.

---

# 3️⃣ 전체 유효성 검사 (폼 단위)

ViewModel의 모든 필드가 유효할 때만 버튼 활성화:

```xml
<Button Content="제출"
        IsEnabled="{Binding !HasErrors}" />
```

또는 ViewModel에 속성 추가:

```csharp
public bool CanSubmit => !HasErrors;
```

---

# 4️⃣ 커스텀 복합 규칙 예제

```csharp
this.ValidationRule(
    vm => vm.Name,
    name => name.Length >= 2 && name.Length <= 10,
    "이름은 2자 이상 10자 이하입니다.");

this.ValidationRule(
    vm => vm.Name,
    name => Regex.IsMatch(name, @"^[가-힣]+$"),
    "한글만 입력 가능합니다");
```

---

# 🧪 고급: 두 필드 비교 (비밀번호 확인 등)

```csharp
this.ValidationRule(
    vm => vm.ConfirmPassword,
    _ => Password == ConfirmPassword,
    "비밀번호가 일치하지 않습니다");
```

---

# ✅ 정리

| 기능 | 구현 방법 |
|------|-----------|
| 필드 단일 검증 | `ValidationRule()` 사용 |
| 오류 메시지 표시 | `ToolTip`, 스타일, 커스텀 오류 템플릿 |
| 전체 폼 유효성 | `HasErrors`, `CanSubmit` 속성 |
| 복합 검증 | 여러 필드 조합 검증 가능 |
| 시각적 표시 | `:invalid` 스타일 활용 |