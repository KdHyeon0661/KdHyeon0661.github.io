---
layout: post
title: Avalonia - 입력 검증 및 ErrorTemplate
date: 2025-01-22 20:20:23 +0900
category: Avalonia
---
# Avalonia MVVM: 입력 검증과 오류 시각화

사용자 입력을 받는 애플리케이션에서 검증(validation)은 필수적입니다. Avalonia는 `INotifyDataErrorInfo` 인터페이스를 기본으로 지원하며, `:invalid` 의사 클래스와 `DataValidationErrors` 첨부 속성을 통해 일관된 오류 시각화를 제공합니다. 이 글에서는 **검증 소스(ReactiveUI.Validation, 수동 구현, DataAnnotations)**, **오류 시각화 스타일**, **교차 필드/비동기 검증**, **커맨드 연동**까지 초중급 개발자 관점에서 정리합니다.

---

## 예제 구조

```
ValidationSamples/
├─ App.axaml
├─ Views/
│  ├─ ValidationView.axaml
│  └─ ValidationView.axaml.cs
├─ ViewModels/
│  ├─ ValidationViewModel.cs             // ReactiveUI.Validation 버전
│  ├─ ValidationViewModel_INDEI.cs       // INotifyDataErrorInfo 수동 버전
│  └─ ValidationViewModel_DataAnno.cs    // DataAnnotations 버전
├─ Converters/
│  ├─ InverseBooleanConverter.cs
│  └─ FirstErrorConverter.cs
└─ Services/
   └─ FakeUserService.cs                 // 비동기 중복 검사 시뮬레이션
```

세 가지 뷰모델은 모두 같은 뷰와 함께 사용할 수 있도록 설계되었습니다. 실제 프로젝트에서는 하나의 방식을 선택해 사용합니다.

---

## 검증 소스

Avalonia는 `INotifyDataErrorInfo`를 구현한 객체를 자동으로 인식합니다. 검증을 구현하는 방법은 크게 세 가지입니다.

| 방식 | 특징 |
|------|------|
| **ReactiveUI.Validation** | 선언적 규칙, 교차/비동기 지원, 코드 간결. MVVM 패턴에 최적. |
| **INotifyDataErrorInfo 수동** | 프레임워크 의존성 최소, 세밀한 제어 가능. 코드량 증가. |
| **DataAnnotations** | 모델 재사용성 높음, 특성 기반 선언. 뷰모델에서 래핑 필요. |

이 글에서는 세 가지 방식을 모두 소개하지만, **ReactiveUI.Validation**이 가장 생산성이 높으므로 중점적으로 다룹니다.

---

### ReactiveUI.Validation 기반 뷰모델

`ReactiveValidationObject`를 상속받아 규칙을 선언적으로 정의합니다.

```csharp
// ViewModels/ValidationViewModel.cs
using ReactiveUI;
using ReactiveUI.Validation.Extensions;
using ReactiveUI.Validation.Helpers;
using System;
using System.Reactive;
using System.Reactive.Linq;
using System.Text.RegularExpressions;
using System.Threading.Tasks;

public sealed class ValidationViewModel : ReactiveValidationObject
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

    private string _email = "";
    public string Email
    {
        get => _email;
        set => this.RaiseAndSetIfChanged(ref _email, value);
    }

    private string _password = "";
    public string Password
    {
        get => _password;
        set => this.RaiseAndSetIfChanged(ref _password, value);
    }

    private string _confirmPassword = "";
    public string ConfirmPassword
    {
        get => _confirmPassword;
        set => this.RaiseAndSetIfChanged(ref _confirmPassword, value);
    }

    private string _userName = "";
    public string UserName
    {
        get => _userName;
        set => this.RaiseAndSetIfChanged(ref _userName, value);
    }

    public ReactiveCommand<Unit, Unit> SubmitCommand { get; }

    public ValidationViewModel(IFakeUserService userService)
    {
        // 단일 필드 규칙
        this.ValidationRule(vm => vm.Name,
            name => !string.IsNullOrWhiteSpace(name),
            "이름을 입력하세요.");

        this.ValidationRule(vm => vm.Age,
            age => age >= 0 && age <= 120,
            "나이는 0~120 사이여야 합니다.");

        this.ValidationRule(vm => vm.Email,
            email => Regex.IsMatch(email ?? "", @"^[^@\s]+@[^@\s]+\.[^@\s]+$"),
            "이메일 형식이 올바르지 않습니다.");

        // 교차 필드 규칙 (비밀번호 확인)
        this.ValidationRule(
            vm => vm.ConfirmPassword,
            _ => Password == ConfirmPassword,
            "비밀번호가 일치하지 않습니다.");

        // 비동기 규칙 (사용자명 중복 검사) – 디바운스 + 서버 호출 시뮬레이션
        this.ValidationRule(
            vm => vm.UserName,
            vm => vm.WhenAnyValue(x => x.UserName)
                    .Throttle(TimeSpan.FromMilliseconds(300))
                    .SelectMany(async user =>
                        string.IsNullOrWhiteSpace(user)
                            ? Observable.Return(false)
                            : Observable.Return(!await userService.ExistsAsync(user)))
                    .Switch(),
            _ => "이미 사용 중인 사용자명입니다.");

        // 제출 가능 여부: 전체 오류 없음
        var canSubmit = this.IsValid(); // IObservable<bool>
        SubmitCommand = ReactiveCommand.Create(OnSubmit, canSubmit);
    }

    private void OnSubmit()
    {
        // 저장/전송 로직
    }
}
```

**주요 포인트**
- `ValidationRule`로 각 속성의 규칙을 선언합니다.
- 교차 필드 규칙은 람다 내에서 다른 속성을 참조할 수 있습니다.
- 비동기 규칙은 `WhenAnyValue`와 `SelectMany`를 결합해 구현합니다.
- `IsValid()`는 전체 오류 상태를 관찰 가능한 스트림으로 반환하므로, 커맨드의 `canExecute`로 바로 사용할 수 있습니다.

---

### INotifyDataErrorInfo 수동 구현

의존성을 최소화하고 싶다면 직접 구현할 수 있습니다.

```csharp
// ViewModels/ValidationViewModel_INDEI.cs
using System.Collections;
using System.Collections.Generic;
using System.ComponentModel;
using System.Linq;
using System.Text.RegularExpressions;

public sealed class ValidationViewModel_INDEI : INotifyPropertyChanged, INotifyDataErrorInfo
{
    public event PropertyChangedEventHandler? PropertyChanged;
    private readonly Dictionary<string, List<string>> _errors = new();

    private string _name = "";
    public string Name
    {
        get => _name;
        set { _name = value; OnPropertyChanged(nameof(Name)); ValidateName(); }
    }

    private int _age;
    public int Age
    {
        get => _age;
        set { _age = value; OnPropertyChanged(nameof(Age)); ValidateAge(); }
    }

    private string _email = "";
    public string Email
    {
        get => _email;
        set { _email = value; OnPropertyChanged(nameof(Email)); ValidateEmail(); }
    }

    public bool HasErrors => _errors.Count > 0;
    public event EventHandler<DataErrorsChangedEventArgs>? ErrorsChanged;

    public IEnumerable GetErrors(string? propertyName)
        => propertyName != null && _errors.TryGetValue(propertyName, out var list)
           ? list : Enumerable.Empty<string>();

    private void OnPropertyChanged(string p) => PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(p));
    private void RaiseErrorsChanged(string p) => ErrorsChanged?.Invoke(this, new DataErrorsChangedEventArgs(p));

    private void SetErrors(string p, IEnumerable<string> errs)
    {
        var list = errs.ToList();
        if (list.Count == 0)
        {
            if (_errors.Remove(p)) RaiseErrorsChanged(p);
        }
        else
        {
            _errors[p] = list;
            RaiseErrorsChanged(p);
        }
    }

    private void ValidateName()
    {
        var errs = new List<string>();
        if (string.IsNullOrWhiteSpace(Name)) errs.Add("이름은 필수입니다.");
        SetErrors(nameof(Name), errs);
    }

    private void ValidateAge()
    {
        var errs = new List<string>();
        if (Age < 0 || Age > 120) errs.Add("나이는 0~120 사이여야 합니다.");
        SetErrors(nameof(Age), errs);
    }

    private void ValidateEmail()
    {
        var errs = new List<string>();
        if (!Regex.IsMatch(Email ?? "", @"^[^@\s]+@[^@\s]+\.[^@\s]+$"))
            errs.Add("이메일 형식이 올바르지 않습니다.");
        SetErrors(nameof(Email), errs);
    }
}
```

이 방식은 완전히 수동이므로 규칙 추가 시마다 검증 메서드를 작성해야 합니다. 하지만 외부 라이브러리 없이도 작동합니다.

---

### DataAnnotations 기반 뷰모델

모델 클래스에 검증 특성을 부여하고, 뷰모델에서 `INotifyDataErrorInfo`를 구현해 모델을 감쌉니다.

```csharp
// ViewModels/ValidationViewModel_DataAnno.cs
using System.Collections.Generic;
using System.ComponentModel;
using System.ComponentModel.DataAnnotations;
using System.Linq;

public sealed class UserForm : INotifyPropertyChanged, INotifyDataErrorInfo
{
    public event PropertyChangedEventHandler? PropertyChanged;

    private string? _name;
    [Required(ErrorMessage = "이름은 필수입니다.")]
    [MinLength(2, ErrorMessage = "이름은 2자 이상이어야 합니다.")]
    public string? Name
    {
        get => _name;
        set { _name = value; OnPropertyChanged(nameof(Name)); Validate(nameof(Name)); }
    }

    private int _age;
    [Range(0, 120, ErrorMessage = "나이는 0~120 사이여야 합니다.")]
    public int Age
    {
        get => _age;
        set { _age = value; OnPropertyChanged(nameof(Age)); Validate(nameof(Age)); }
    }

    private string? _email;
    [EmailAddress(ErrorMessage = "이메일 형식이 올바르지 않습니다.")]
    public string? Email
    {
        get => _email;
        set { _email = value; OnPropertyChanged(nameof(Email)); Validate(nameof(Email)); }
    }

    private readonly Dictionary<string, List<string>> _errors = new();
    public bool HasErrors => _errors.Any();
    public event EventHandler<DataErrorsChangedEventArgs>? ErrorsChanged;
    public IEnumerable<object> GetErrors(string? propertyName)
        => propertyName != null && _errors.TryGetValue(propertyName, out var list) ? list : Enumerable.Empty<object>();

    private void OnPropertyChanged(string p) => PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(p));
    private void RaiseErrorsChanged(string p) => ErrorsChanged?.Invoke(this, new DataErrorsChangedEventArgs(p));

    public void Validate(string propertyName)
    {
        var ctx = new ValidationContext(this) { MemberName = propertyName };
        var results = new List<ValidationResult>();
        Validator.TryValidateProperty(
            GetType().GetProperty(propertyName)?.GetValue(this),
            ctx, results);

        if (results.Count == 0) _errors.Remove(propertyName);
        else _errors[propertyName] = results.Select(r => r.ErrorMessage ?? "").ToList();
        RaiseErrorsChanged(propertyName);
    }

    public void ValidateAll()
    {
        var ctx = new ValidationContext(this);
        var results = new List<ValidationResult>();
        Validator.TryValidateObject(this, ctx, results, true);
        _errors.Clear();
        foreach (var g in results.GroupBy(r => r.MemberNames.FirstOrDefault() ?? ""))
            _errors[g.Key] = g.Select(r => r.ErrorMessage ?? "").ToList();
        foreach (var k in _errors.Keys.ToArray()) RaiseErrorsChanged(k);
    }
}
```

뷰모델에서는 이 `UserForm` 인스턴스를 속성으로 노출하고, 바인딩은 `Form.Name` 등으로 합니다. 검증은 `ValidateAll` 또는 각 속성 변경 시 `Validate(propertyName)`을 호출합니다.

---

## 오류 시각화

Avalonia는 `INotifyDataErrorInfo`를 구현한 객체에 대해 자동으로 컨트롤의 `:invalid` 의사 클래스를 활성화합니다. 또한 `DataValidationErrors.Errors` 첨부 속성을 통해 현재 오류 컬렉션에 접근할 수 있습니다.

### 공통 컨버터

첫 번째 오류 메시지를 추출하는 컨버터와 불리언 반전 컨버터를 준비합니다.

```csharp
// Converters/FirstErrorConverter.cs
public sealed class FirstErrorConverter : IValueConverter
{
    public static FirstErrorConverter Instance { get; } = new();
    public object? Convert(object? value, Type targetType, object? parameter, CultureInfo culture)
        => value is IEnumerable e ? e.Cast<object>().FirstOrDefault() : null;
    public object? ConvertBack(object? value, Type targetType, object? parameter, CultureInfo culture)
        => throw new NotSupportedException();
}
```

```csharp
// Converters/InverseBooleanConverter.cs
public sealed class InverseBooleanConverter : IValueConverter
{
    public object? Convert(object? value, Type targetType, object? parameter, CultureInfo culture)
        => value is bool b ? !b : Avalonia.Data.BindingOperations.DoNothing;
    public object? ConvertBack(object? value, Type targetType, object? parameter, CultureInfo culture)
        => throw new NotSupportedException();
}
```

### 뷰에서 검증 UI 구현

```xml
<!-- Views/ValidationView.axaml -->
<UserControl xmlns="https://github.com/avaloniaui"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:vm="using:ValidationSamples.ViewModels"
             xmlns:conv="using:ValidationSamples.Converters"
             xmlns:ac="using:Avalonia.Controls"
             x:Class="ValidationSamples.Views.ValidationView">

  <UserControl.DataContext>
    <!-- 세 가지 중 하나를 선택 -->
    <vm:ValidationViewModel>
      <vm:ValidationViewModel.UserService>
        <services:FakeUserService/>
      </vm:ValidationViewModel.UserService>
    </vm:ValidationViewModel>
  </UserControl.DataContext>

  <UserControl.Styles>
    <!-- 오류 상태의 TextBox 스타일 -->
    <Style Selector="TextBox:invalid">
      <Setter Property="BorderBrush" Value="#E53935"/>
      <Setter Property="BorderThickness" Value="2"/>
      <Setter Property="Background" Value="#FFF3F3"/>
      <Setter Property="ToolTip.Tip"
              Value="{Binding (ac:DataValidationErrors.Errors), RelativeSource={RelativeSource Self}, Converter={x:Static conv:FirstErrorConverter.Instance}}"/>
    </Style>
  </UserControl.Styles>

  <ScrollViewer>
    <StackPanel Margin="20" Spacing="10">
      <TextBlock Text="회원 가입 폼" FontSize="18" FontWeight="Bold"/>

      <!-- 이름 -->
      <StackPanel Spacing="4">
        <TextBlock Text="이름"/>
        <TextBox Text="{Binding Name, Mode=TwoWay}" />
        <TextBlock Foreground="#E53935" FontSize="12"
                   Text="{Binding (ac:DataValidationErrors.Errors), RelativeSource={RelativeSource Previous}, Converter={x:Static conv:FirstErrorConverter.Instance}}"/>
      </StackPanel>

      <!-- 나이 -->
      <StackPanel Spacing="4">
        <TextBlock Text="나이"/>
        <TextBox Text="{Binding Age, Mode=TwoWay}" />
        <TextBlock Foreground="#E53935" FontSize="12"
                   Text="{Binding (ac:DataValidationErrors.Errors), RelativeSource={RelativeSource Previous}, Converter={x:Static conv:FirstErrorConverter.Instance}}"/>
      </StackPanel>

      <!-- 이메일 -->
      <StackPanel Spacing="4">
        <TextBlock Text="이메일"/>
        <TextBox Text="{Binding Email, Mode=TwoWay}" />
        <TextBlock Foreground="#E53935" FontSize="12"
                   Text="{Binding (ac:DataValidationErrors.Errors), RelativeSource={RelativeSource Previous}, Converter={x:Static conv:FirstErrorConverter.Instance}}"/>
      </StackPanel>

      <!-- 사용자명 (비동기 중복 검사) -->
      <StackPanel Spacing="4">
        <TextBlock Text="사용자명"/>
        <TextBox Text="{Binding UserName, Mode=TwoWay}" />
        <TextBlock Foreground="#E53935" FontSize="12"
                   Text="{Binding (ac:DataValidationErrors.Errors), RelativeSource={RelativeSource Previous}, Converter={x:Static conv:FirstErrorConverter.Instance}}"/>
      </StackPanel>

      <!-- 비밀번호 -->
      <StackPanel Spacing="4">
        <TextBlock Text="비밀번호"/>
        <TextBox PasswordChar="●" Text="{Binding Password, Mode=TwoWay}"/>
        <TextBlock Foreground="#E53935" FontSize="12"
                   Text="{Binding (ac:DataValidationErrors.Errors), RelativeSource={RelativeSource Previous}, Converter={x:Static conv:FirstErrorConverter.Instance}}"/>
      </StackPanel>

      <!-- 비밀번호 확인 -->
      <StackPanel Spacing="4">
        <TextBlock Text="비밀번호 확인"/>
        <TextBox PasswordChar="●" Text="{Binding ConfirmPassword, Mode=TwoWay}"/>
        <TextBlock Foreground="#E53935" FontSize="12"
                   Text="{Binding (ac:DataValidationErrors.Errors), RelativeSource={RelativeSource Previous}, Converter={x:Static conv:FirstErrorConverter.Instance}}"/>
      </StackPanel>

      <Separator/>

      <StackPanel Orientation="Horizontal" Spacing="8">
        <Button Content="제출" Command="{Binding SubmitCommand}" />
        <!-- ReactiveUI.Validation이 아닌 경우 아래와 같이 IsEnabled 직접 바인딩 -->
        <!-- <Button Content="제출" IsEnabled="{Binding HasErrors, Converter={StaticResource InverseBooleanConverter}}"/> -->
      </StackPanel>
    </StackPanel>
  </ScrollViewer>
</UserControl>
```

**시각화 포인트**
- `TextBox:invalid` 스타일로 오류 시 테두리와 배경을 변경합니다.
- `ToolTip.Tip`에 첫 번째 오류 메시지를 표시합니다.
- 각 필드 아래에 인라인 오류 메시지를 배치합니다.
- 버튼의 `IsEnabled`는 `HasErrors`의 반전으로 처리하거나, ReactiveUI.Validation의 `SubmitCommand`가 자동 처리합니다.

---

## 교차 필드 검증과 비동기 검증

### 교차 필드 (비밀번호 확인)

ReactiveUI.Validation에서 교차 필드 규칙은 다음과 같이 작성합니다.

```csharp
this.ValidationRule(
    vm => vm.ConfirmPassword,
    _ => Password == ConfirmPassword,
    "비밀번호가 일치하지 않습니다.");
```

이 규칙은 `Password` 또는 `ConfirmPassword`가 변경될 때마다 재평가됩니다.

### 비동기 검증 (사용자명 중복)

`WhenAnyValue`와 `Throttle`, `SelectMany`를 결합해 서버 호출을 디바운스합니다.

```csharp
this.ValidationRule(
    vm => vm.UserName,
    vm => vm.WhenAnyValue(x => x.UserName)
            .Throttle(TimeSpan.FromMilliseconds(300))
            .SelectMany(async user =>
                string.IsNullOrWhiteSpace(user)
                    ? Observable.Return(false)
                    : Observable.Return(!await userService.ExistsAsync(user)))
            .Switch(),
    _ => "이미 사용 중인 사용자명입니다.");
```

`Throttle`로 타이핑 중에 빈번한 호출을 막고, `Switch`로 최신 요청만 처리합니다.

---

## 폼 수준 요약 (ValidationSummary)

전체 오류를 한 곳에 모아 보여주는 `ValidationSummary`는 뷰모델에서 파생 속성으로 쉽게 구현할 수 있습니다.

```csharp
public string AllErrors =>
    string.Join(Environment.NewLine,
        from prop in new[] { nameof(Name), nameof(Age), nameof(Email), nameof(UserName), nameof(Password), nameof(ConfirmPassword) }
        from string err in (GetErrors(prop) ?? Enumerable.Empty<object>())
        select $"{prop}: {err}");
```

XAML에서 바인딩:

```xml
<TextBlock Text="{Binding AllErrors}" Foreground="#E53935" TextWrapping="Wrap"/>
```

---

## 검증과 커맨드 연동

### ReactiveUI.Validation

`this.IsValid()`은 전체 유효성 상태를 `IObservable<bool>`로 반환하므로, `ReactiveCommand`의 `canExecute` 인자로 바로 사용합니다.

```csharp
SubmitCommand = ReactiveCommand.Create(OnSubmit, this.IsValid());
```

### 수동 구현 시

`HasErrors` 속성이 변경될 때마다 `ICommand`의 `CanExecuteChanged`를 발생시킵니다.

```csharp
// INotifyPropertyChanged를 상속한 경우
private void OnPropertyChanged(string p)
{
    PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(p));
    if (p == nameof(HasErrors))
        SubmitCommand.RaiseCanExecuteChanged();
}
```

또는 `ReactiveCommand`의 `canExecute`를 직접 구성합니다.

```csharp
var canSubmit = this.WhenAnyValue(vm => vm.HasErrors).Select(h => !h);
SubmitCommand = ReactiveCommand.Create(OnSubmit, canSubmit);
```

---

## 단위 테스트

검증 로직은 뷰 없이도 테스트할 수 있어야 합니다.

```csharp
[Fact]
public void Name_Required_ValidationFails()
{
    var vm = new ValidationViewModel(new FakeUserService());
    vm.Name = "";
    Assert.True(vm.HasErrors); // 또는 vm.ValidationContext.IsValid == false
}

[Fact]
public async Task UserName_Duplicate_Fails()
{
    var vm = new ValidationViewModel(new FakeUserService());
    vm.UserName = "admin";
    await Task.Delay(400); // 디바운스 대기
    Assert.True(vm.HasErrors);
}
```

---

## 성능과 설계 팁

- **검증 규칙은 뷰모델에**, 스타일은 XAML에 배치해 관심사를 분리합니다.
- 비동기 검증에는 반드시 **디바운스**와 **취소 토큰**을 적용해 불필요한 호출을 막습니다.
- `DataValidationErrors.Errors`에 바인딩할 때는 `FirstErrorConverter` 같은 컨버터로 첫 번째 메시지만 표시하는 것이 일반적입니다.
- 다국어를 고려한다면 메시지를 리소스 키로 분리합니다.
- 복잡한 규칙은 확장 메서드나 헬퍼 클래스로 모듈화합니다.

---

## 결론

Avalonia에서 입력 검증은 `INotifyDataErrorInfo`를 기반으로 하며, `:invalid`와 `DataValidationErrors`를 통해 일관된 시각화를 제공합니다. 세 가지 구현 방식 중 **ReactiveUI.Validation**이 선언적이고 생산성이 높아 가장 권장됩니다. 교차 필드, 비동기 검증, 커맨드 연동까지 자연스럽게 처리할 수 있으며, 단위 테스트도 용이합니다. 이 글의 패턴을 활용하면 사용자 친화적이고 유지보수하기 쉬운 입력 폼을 구축할 수 있습니다.