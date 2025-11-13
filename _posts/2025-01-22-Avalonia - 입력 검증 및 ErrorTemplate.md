---
layout: post
title: Avalonia - 입력 검증 및 ErrorTemplate
date: 2025-01-22 20:20:23 +0900
category: Avalonia
---
# Avalonia MVVM: 입력 검증(Validation)과 시각화(Error UI)

여기서는 세 가지 축으로 정리한다.

1) **검증 소스**
- `ReactiveUI.Validation` (권장: 선언적, 테스트 용이)
- `INotifyDataErrorInfo` 직접 구현 (프레임워크 의존↓, 세밀 제어)
- `DataAnnotations` (간단 명료, 모델 재사용)

2) **오류 시각화**
- `:invalid` 의사 클래스 기반 스타일(테두리/배경/아이콘)
- `DataValidationErrors`(첨부 속성)로 **필드 단위 메시지 바인딩**
- 폼 헤더/푸터에 **ValidationSummary** 패턴

3) **고급 기능**
- 교차 필드/복합 규칙, 비동기(서버) 검증, 디바운스
- 커맨드 활성화(CanExecute)와 검증 연동
- JSON 저장/복구, 단위 테스트

---

## 0. 예제 구조

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

> 초안에서 제시한 뷰/뷰모델을 그대로 확장하는 느낌으로, **대체 가능한 3가지 VM**을 나란히 제시한다.

---

## 1. ReactiveUI.Validation 기반 — 선언적 규칙/교차 필드/비동기

### 1.1 ViewModel (권장)

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

        // 교차 필드 규칙(비밀번호 확인)
        this.ValidationRule(
            vm => vm.ConfirmPassword,
            _ => Password == ConfirmPassword,
            "비밀번호가 일치하지 않습니다.");

        // 비동기 규칙(사용자명 중복 검사) — 디바운스 + 서버 호출 시뮬
        // ReactiveUI.Validation은 Sync/Async 규칙 모두 지원
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

        // Submit 가능 여부: 전체 오류 없음
        var canSubmit = this.IsValid(); // IObservable<bool>
        SubmitCommand = ReactiveCommand.Create(OnSubmit, canSubmit);
    }

    private void OnSubmit()
    {
        // 저장/전송 등의 로직. 폼이 유효할 때만 호출됨.
    }
}
```

> `ReactiveValidationObject`는 `INotifyDataErrorInfo`를 구현한다. Avalonia는 이 인터페이스를 인식하여 컨트롤을 `:invalid` 상태로 만든다.

### 1.2 비동기 검증용 서비스 (시뮬레이션)

```csharp
// Services/FakeUserService.cs
using System.Threading.Tasks;

public interface IFakeUserService
{
    Task<bool> ExistsAsync(string userName);
}

public sealed class FakeUserService : IFakeUserService
{
    // 아주 단순한 샘플: 특정 아이디만 "중복"이라고 가정
    public Task<bool> ExistsAsync(string userName)
        => Task.FromResult(userName.Trim().ToLower() is "admin" or "root" or "guest");
}
```

---

## 2. INotifyDataErrorInfo 수동 구현 — 프레임워크 의존↓/세밀 제어

ReactiveUI.Validation 없이도 가능하다.
장점: 의존성↓, 런타임 제어↑. 단점: 코드량↑.

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

---

## 3. DataAnnotations — 모델/DTO 재사용에 유리

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

---

## 4. View — 오류 시각화(스타일) + 메시지 바인딩

### 4.1 공통 리소스 (컨버터)

```csharp
// Converters/InverseBooleanConverter.cs
using Avalonia.Data.Converters;
using System;
using System.Globalization;

public sealed class InverseBooleanConverter : IValueConverter
{
    public object? Convert(object? value, Type targetType, object? parameter, CultureInfo culture)
        => value is bool b ? !b : Avalonia.Data.BindingOperations.DoNothing;

    public object? ConvertBack(object? value, Type targetType, object? parameter, CultureInfo culture)
        => throw new NotSupportedException();
}
```

```csharp
// Converters/FirstErrorConverter.cs
using Avalonia.Data.Converters;
using System;
using System.Collections;
using System.Globalization;
using System.Linq;

public sealed class FirstErrorConverter : IValueConverter
{
    public static FirstErrorConverter Instance { get; } = new();
    public object? Convert(object? value, Type targetType, object? parameter, CultureInfo culture)
        => value is IEnumerable e ? e.Cast<object>().FirstOrDefault() : null;
    public object? ConvertBack(object? value, Type targetType, object? parameter, CultureInfo culture)
        => throw new NotSupportedException();
}
```

### 4.2 뷰(XAML)

> 핵심: Avalonia는 `:invalid` 의사 클래스와 `DataValidationErrors`(첨부 속성)를 제공한다.

- `:invalid` → 컨트롤이 오류일 때 스타일 적용
- `ac:DataValidationErrors.Errors` → 필드의 에러 컬렉션

```xml
<!-- Views/ValidationView.axaml -->
<UserControl xmlns="https://github.com/avaloniaui"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:vm="clr-namespace:ValidationSamples.ViewModels"
             xmlns:conv="clr-namespace:ValidationSamples.Converters"
             xmlns:ac="clr-namespace:Avalonia.Controls;assembly=Avalonia.Controls"
             x:Class="ValidationSamples.Views.ValidationView">
  <UserControl.DataContext>
    <!-- 세 가지 중 하나를 주입해서 실험 가능 -->
    <!--<vm:ValidationViewModel />-->
    <!--<vm:ValidationViewModel_INDEI />-->
    <vm:ValidationViewModel>
      <vm:ValidationViewModel.UserService>
        <services:FakeUserService/>
      </vm:ValidationViewModel.UserService>
    </vm:ValidationViewModel>
  </UserControl.DataContext>

  <UserControl.Styles>
    <!-- 오류 시각화: 빨간 테두리 + 배경 살짝 -->
    <Style Selector="TextBox:invalid">
      <Setter Property="BorderBrush" Value="#E53935"/>
      <Setter Property="BorderThickness" Value="2"/>
      <Setter Property="Background" Value="#FFF3F3"/>
    </Style>

    <!-- Tooltip에 첫 에러 노출 -->
    <Style Selector="TextBox:invalid">
      <Setter Property="ToolTip.Tip"
              Value="{Binding #This.(ac:DataValidationErrors.Errors), ElementName=This, Converter={x:Static conv:FirstErrorConverter.Instance}}"/>
    </Style>
  </UserControl.Styles>

  <ScrollViewer>
    <StackPanel Margin="20" Spacing="10">
      <TextBlock Text="회원 가입 폼" FontSize="18" FontWeight="Bold"/>

      <!-- 이름 -->
      <StackPanel x:Name="This" Spacing="4">
        <TextBlock Text="이름"/>
        <TextBox Text="{Binding Name, Mode=TwoWay}" />
        <!-- 인라인 에러 -->
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

      <!-- 사용자명 (비동기 중복 검사: ReactiveUI.Validation 버전일 때 동작) -->
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

      <!-- 비밀번호 확인 (교차 규칙) -->
      <StackPanel Spacing="4">
        <TextBlock Text="비밀번호 확인"/>
        <TextBox PasswordChar="●" Text="{Binding ConfirmPassword, Mode=TwoWay}"/>
        <TextBlock Foreground="#E53935" FontSize="12"
                   Text="{Binding (ac:DataValidationErrors.Errors), RelativeSource={RelativeSource Previous}, Converter={x:Static conv:FirstErrorConverter.Instance}}"/>
      </StackPanel>

      <Separator/>

      <!-- 폼 단위: HasErrors/IsValid를 버튼 활성화와 연결 -->
      <StackPanel Orientation="Horizontal" Spacing="8">
        <Button Content="제출" Command="{Binding SubmitCommand}" />
        <!-- ReactiveUI.Validation: SubmitCommand CanExecute==IsValid -->
        <!-- INotifyDataErrorInfo/Annotations 버전을 쓸 때는 다음처럼: -->
        <!-- <Button Content="제출" IsEnabled="{Binding HasErrors, Converter={StaticResource InverseBooleanConverter}}"/> -->
      </StackPanel>
    </StackPanel>
  </ScrollViewer>
</UserControl>
```

> **포인트**
> - `TextBox:invalid` selector로 오류 테두리/배경을 통일 스타일로 적용
> - `ac:DataValidationErrors.Errors`에 바인딩해 **첫 번째 에러**를 인라인 표기
> - 버튼 활성화는 `SubmitCommand`의 CanExecute 또는 `HasErrors`와 `InverseBooleanConverter`로 처리

---

## 5. 고급 규칙 모음

### 5.1 다중 규칙(이름 길이 + 허용 문자)

```csharp
this.ValidationRule(vm => vm.Name,
    name => !string.IsNullOrWhiteSpace(name),
    "이름은 필수입니다.");

this.ValidationRule(vm => vm.Name,
    name => name.Length is >= 2 and <= 16,
    "이름은 2~16자여야 합니다.");

this.ValidationRule(vm => vm.Name,
    name => Regex.IsMatch(name ?? "", @"^[가-힣a-zA-Z\s]+$"),
    "이름에는 한글/영문/공백만 허용됩니다.");
```

> 한 속성에 **여러 개의 룰**을 선언하면, 실패한 모든 규칙의 메시지가 Errors 컬렉션에 누적된다.

### 5.2 날짜 범위/구간 검증

```csharp
private DateTimeOffset? _from, _to;
public DateTimeOffset? From { get => _from; set => this.RaiseAndSetIfChanged(ref _from, value); }
public DateTimeOffset? To   { get => _to;   set => this.RaiseAndSetIfChanged(ref _to,   value); }

this.ValidationRule(vm => vm.To,
    _ => (From, To) is (not null, not null) && From <= To,
    "종료일은 시작일 이후여야 합니다.");
```

### 5.3 서버 사이드 검증(비동기) + 디바운스

앞서 `UserName`에서 구현한 패턴처럼,
`WhenAnyValue(...).Throttle(...).SelectMany(async ...)` 형태로 비동기 호출을 연결한다.
오래 걸리는 호출에는 타임아웃/취소 토큰을 더한다.

---

## 6. 검증과 Command 활성화 연동

- **ReactiveUI.Validation**: `this.IsValid()`가 `IObservable<bool>`을 내어주므로 Command 생성 시 그대로 적용
- **수동(INotifyDataErrorInfo)**: `HasErrors` 변화에 대응해 `RaiseCanExecuteChanged()`(표준 ICommand) 또는 ReactiveCommand의 `canExecute`를 구성

```csharp
// ReactiveCommand 예
var canSubmit = this.IsValid(); // ReactiveUI.Validation
SubmitCommand = ReactiveCommand.Create(OnSubmit, canSubmit);
```

```csharp
// 수동: INDEI 버전에서
var canSubmit = this.WhenAnyValue(vm => vm.HasErrors).Select(h => !h);
SubmitCommand = ReactiveCommand.Create(OnSubmit, canSubmit);
```

---

## 7. ValidationSummary 패턴 (폼 상단/하단에 전역 메시지)

`INotifyDataErrorInfo`의 모든 에러를 한데 모아 사용자에게 요약을 보여준다.
간단한 구현은 VM에서 `AllErrors` 파생 속성을 제공하면 된다.

```csharp
public string AllErrors =>
    string.Join(Environment.NewLine,
        from prop in new[] { nameof(Name), nameof(Age), nameof(Email), nameof(UserName), nameof(Password), nameof(ConfirmPassword) }
        from string err in (GetErrors(prop) ?? Array.Empty<object>())
        select $"{prop}: {err}");
```

```xml
<TextBlock Text="{Binding AllErrors}" Foreground="#E53935" TextWrapping="Wrap" />
```

---

## 8. 저장/복원(옵션)

검증과 무관하지만 폼 UX에서는 유용하다.

```csharp
// 폼 스냅샷
private record FormSnapshot(string Name, int Age, string Email, string UserName);

private FormSnapshot ToSnapshot() => new(Name, Age, Email, UserName);
private void Restore(FormSnapshot s)
{
    Name = s.Name; Age = s.Age; Email = s.Email; UserName = s.UserName;
}
```

---

## 9. 단위 테스트(요지)

검증 로직은 **View 없이** 테스트 가능해야 한다.

```csharp
// xUnit 샘플
[Fact]
public void Name_Required()
{
    var vm = new ValidationViewModel(new FakeUserService());
    vm.Name = "";
    // ReactiveUI.Validation: HasErrors는 Aggregated
    Assert.True(vm.ValidationContext?.IsValid == false || vm.HasErrors);
}

[Fact]
public async Task UserName_Duplicate_Fails()
{
    var vm = new ValidationViewModel(new FakeUserService());
    vm.UserName = "admin";
    await Task.Delay(400); // debounce + async
    Assert.True(vm.HasErrors);
}
```

---

## 10. 성능/설계 팁

- **규칙은 VM**에, **스타일은 XAML**에: 관심사 분리
- **데이터 흐름은 단방향**으로(입력 → 검증 → 요약/상태)
- **비동기 검증은 디바운스**로 과도한 호출 억제
- **ErrorTemplate-like UI**는 `:invalid` + `DataValidationErrors.Errors`로 일관되게
- 다국어/로캘: **리소스 키** 기반 메시지로 국제화 준비
- 공용 규칙은 **확장 메서드**나 **Validator 유틸**로 모듈화

---

## 결론

- Avalonia는 `INotifyDataErrorInfo` + `:invalid` + `DataValidationErrors`로 **검증/시각화 경로**를 명확히 제공한다.
- **ReactiveUI.Validation**을 쓰면 규칙 선언, 교차/비동기, Command 연동이 대폭 간단해진다.
- 팀/프로젝트 성격에 따라
  - 빠른 생산성: **ReactiveUI.Validation**,
  - 의존성 최소화: **INotifyDataErrorInfo 수동**,
  - 모델 재사용: **DataAnnotations**
  를 선택하되, **시각화 스타일과 바인딩 패턴은 동일**하게 유지하면 된다.

위 설계를 토대로 기존 문서의 핵심(필드 검증/오류 시각화/폼 유효성)을 그대로 살리면서, **교차/비동기/요약/테스트**까지 한 번에 끌어올릴 수 있다.
