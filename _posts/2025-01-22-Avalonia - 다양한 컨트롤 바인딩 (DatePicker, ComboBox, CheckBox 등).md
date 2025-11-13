---
layout: post
title: Avalonia - 다양한 컨트롤 바인딩 (DatePicker, ComboBox, CheckBox 등)
date: 2025-01-22 19:20:23 +0900
category: Avalonia
---
# Avalonia MVVM: 다양한 컨트롤 바인딩

## 0. 예제 프로젝트 스캐폴드

```
MyAvaloniaApp/
├── App.axaml
├── Themes/
│   ├── Colors.axaml
│   └── Controls.axaml
├── Views/
│   └── ControlsDemoView.axaml
├── ViewModels/
│   └── ControlsDemoViewModel.cs
├── Models/
│   └── User.cs
├── Converters/
│   ├── BoolToTextConverter.cs
│   └── DateTimeOffsetFormatConverter.cs
├── Services/
│   └── JsonStorageService.cs
└── MyAvaloniaApp.csproj
```

> 본문은 **초안의 ViewModel/뷰 구조를 그대로 사용**하면서, 필요한 클래스를 덧붙이는 식으로 확장한다.

---

## 1. DatePicker — 날짜 선택 바인딩 (기본형 → 실전형)

### 1.1 기본형

```xml
<!-- Views/ControlsDemoView.axaml (발췌) -->
<StackPanel Spacing="8">
  <DatePicker SelectedDate="{Binding SelectedDate}" />
  <TextBlock
    Text="{Binding SelectedDate,
                   StringFormat='선택한 날짜: {0:yyyy-MM-dd}'}" />
</StackPanel>
```

```csharp
// ViewModels/ControlsDemoViewModel.cs (발췌)
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

핵심 포인트

- `DatePicker.SelectedDate` 타입은 `DateTimeOffset?`.
- `null` 허용 → 초기 미선택 처리 가능.
- 문자열 포맷은 `StringFormat` 혹은 `IValueConverter`로 수행.

### 1.2 포맷/빈값 처리 — Converter 활용

```csharp
// Converters/DateTimeOffsetFormatConverter.cs
using System;
using Avalonia.Data.Converters;
using System.Globalization;

public sealed class DateTimeOffsetFormatConverter : IValueConverter
{
    public string Format { get; set; } = "yyyy-MM-dd";

    public object? Convert(object? value, Type targetType, object? parameter, CultureInfo culture)
    {
        if (value is DateTimeOffset dto)
            return dto.ToString(Format, culture);
        return string.Empty; // null 또는 잘못된 형식 처리
    }

    public object? ConvertBack(object? value, Type targetType, object? parameter, CultureInfo culture)
    {
        if (value is string s && DateTimeOffset.TryParse(s, culture, DateTimeStyles.None, out var dto))
            return dto;
        return null;
    }
}
```

```xml
<!-- App.axaml (리소스 등록 예) -->
<Application ... xmlns:conv="clr-namespace:MyAvaloniaApp.Converters">
  <Application.Styles>
    <FluentTheme Mode="Light"/>
  </Application.Styles>
  <Application.Resources>
    <conv:DateTimeOffsetFormatConverter x:Key="DateFmt" Format="yyyy-MM-dd"/>
  </Application.Resources>
</Application>
```

```xml
<!-- Views/ControlsDemoView.axaml (발췌) -->
<TextBlock
  Text="{Binding SelectedDate, Converter={StaticResource DateFmt}}"/>
```

### 1.3 최소/최대 범위(가드), 적용 버튼 활성화

```csharp
// ViewModels/ControlsDemoViewModel.cs (발췌)
using System.Reactive;
using System.Reactive.Linq;

public class ControlsDemoViewModel : ReactiveObject
{
    private DateTimeOffset? _selectedDate = DateTimeOffset.Now;
    public DateTimeOffset? SelectedDate
    {
        get => _selectedDate;
        set => this.RaiseAndSetIfChanged(ref _selectedDate, value);
    }

    public DateTimeOffset MinDate { get; } = new DateTimeOffset(2020,1,1,0,0,0,TimeSpan.Zero);
    public DateTimeOffset MaxDate { get; } = new DateTimeOffset(2030,12,31,0,0,0,TimeSpan.Zero);

    public ReactiveCommand<Unit, Unit> ApplyDateCommand { get; }

    public ControlsDemoViewModel()
    {
        var canApply =
            this.WhenAnyValue(vm => vm.SelectedDate)
                .Select(d => d.HasValue && d.Value >= MinDate && d.Value <= MaxDate);

        ApplyDateCommand = ReactiveCommand.Create(
            () => { /* 저장/적용 로직 */ },
            canApply);
    }
}
```

```xml
<!-- Views/ControlsDemoView.axaml (발췌) -->
<StackPanel Spacing="8">
  <TextBlock Text="날짜 범위: 2020-01-01 ~ 2030-12-31"/>
  <DatePicker SelectedDate="{Binding SelectedDate}"/>
  <Button Content="날짜 적용" Command="{Binding ApplyDateCommand}" />
</StackPanel>
```

핵심 포인트

- UI에서 직접 `MinDate/MaxDate` 속성이 노출되지 않더라도, **버튼 활성화 조건**으로 간접 제약을 건다.
- 날짜 범위가 필요하면 커스텀 Validation과 `DataValidationErrors`(고급)로도 가능.

---

## 2. ComboBox — 문자열/객체/Enum/Id 바인딩

### 2.1 문자열 컬렉션 선택

```xml
<StackPanel Spacing="8">
  <ComboBox Items="{Binding Fruits}" SelectedItem="{Binding SelectedFruit}" />
  <TextBlock Text="{Binding SelectedFruit, StringFormat='선택: {0}'}" />
</StackPanel>
```

```csharp
public partial class ControlsDemoViewModel : ReactiveObject
{
    public ObservableCollection<string> Fruits { get; } = new()
    {
        "사과", "바나나", "포도", "오렌지"
    };

    private string? _selectedFruit;
    public string? SelectedFruit
    {
        get => _selectedFruit;
        set => this.RaiseAndSetIfChanged(ref _selectedFruit, value);
    }
}
```

### 2.2 객체 컬렉션(표시/값 분리)

```csharp
// Models/User.cs
public sealed class User
{
    public int Id { get; init; }
    public string Name { get; init; } = "";
    public override string ToString() => Name;
}
```

```csharp
// ViewModels/ControlsDemoViewModel.cs (발췌)
public ObservableCollection<User> Users { get; } = new()
{
    new User { Id = 1, Name = "홍길동" },
    new User { Id = 2, Name = "이순신" },
    new User { Id = 3, Name = "신사임당" },
};

private User? _selectedUser;
public User? SelectedUser
{
    get => _selectedUser;
    set => this.RaiseAndSetIfChanged(ref _selectedUser, value);
}

// Id만 따로 보관/활용하고 싶을 때
public int? SelectedUserId
    => SelectedUser?.Id;
```

```xml
<!-- ToString() 표시 사용 -->
<ComboBox Items="{Binding Users}"
          SelectedItem="{Binding SelectedUser}" />
<TextBlock Text="{Binding SelectedUser.Name, StringFormat='사용자: {0}'}" />
<TextBlock Text="{Binding SelectedUserId, StringFormat='Id: {0}'}" />
```

> Avalonia는 WPF의 `SelectedValuePath`와 완전히 동일하진 않다. 실전에서는 **`SelectedItem`로 객체를 바인딩**하고, ViewModel에서 **파생 속성**(예: `SelectedUserId`)을 노출하는 패턴이 가장 예측 가능하고 테스트하기 쉽다.

### 2.3 DataTemplate로 표시 커스터마이징

```xml
<ComboBox Items="{Binding Users}"
          SelectedItem="{Binding SelectedUser}">
  <ComboBox.ItemTemplate>
    <DataTemplate>
      <StackPanel Orientation="Horizontal" Spacing="6">
        <TextBlock Text="{Binding Name}" FontWeight="Bold"/>
        <TextBlock Text="{Binding Id, StringFormat='(ID: {0})'}"
                   Foreground="Gray"/>
      </StackPanel>
    </DataTemplate>
  </ComboBox.ItemTemplate>
</ComboBox>
```

### 2.4 Enum 바인딩

```csharp
public enum Priority { Low, Normal, High }

public Priority[] Priorities { get; } = (Priority[])Enum.GetValues(typeof(Priority));

private Priority _selectedPriority = Priority.Normal;
public Priority SelectedPriority
{
    get => _selectedPriority;
    set => this.RaiseAndSetIfChanged(ref _selectedPriority, value);
}
```

```xml
<ComboBox Items="{Binding Priorities}"
          SelectedItem="{Binding SelectedPriority}"/>
<TextBlock Text="{Binding SelectedPriority}"/>
```

---

## 3. CheckBox — bool/nullable/마스터-디테일 패턴

### 3.1 단일 체크

```xml
<StackPanel Spacing="8">
  <CheckBox Content="약관에 동의합니다"
            IsChecked="{Binding IsAccepted}" />
  <Button Content="계속"
          IsEnabled="{Binding IsAccepted}" />
</StackPanel>
```

```csharp
private bool _isAccepted;
public bool IsAccepted
{
    get => _isAccepted;
    set => this.RaiseAndSetIfChanged(ref _isAccepted, value);
}
```

### 3.2 삼상 체크(bool?)와 마스터 체크

- 모든 항목이 체크 → `true`
- 아무 항목도 체크 아님 → `false`
- 혼합(일부만 체크) → `null` (Indeterminate)

```csharp
public sealed class OptionItem : ReactiveObject
{
    private bool _checked;
    public string Name { get; init; } = "";
    public bool Checked
    {
        get => _checked;
        set => this.RaiseAndSetIfChanged(ref _checked, value);
    }
}

public ObservableCollection<OptionItem> Options { get; } = new()
{
    new OptionItem { Name = "메일 알림" },
    new OptionItem { Name = "SMS 알림" },
    new OptionItem { Name = "푸시 알림" },
};

private bool? _checkAll = false;
public bool? CheckAll
{
    get => _checkAll;
    set
    {
        this.RaiseAndSetIfChanged(ref _checkAll, value);
        if (value.HasValue)
        {
            foreach (var o in Options) o.Checked = value.Value;
        }
    }
}

public ControlsDemoViewModel()
{
    // 항목의 개별 변경 → 마스터 상태 갱신
    Options
        .ToObservableChangeSet() // DynamicData 사용 시
        .AutoRefresh(x => x.Checked)
        .Throttle(TimeSpan.FromMilliseconds(50))
        .Subscribe(_ => UpdateMasterCheck());
}

private void UpdateMasterCheck()
{
    var cnt = Options.Count;
    var checkedCnt = Options.Count(o => o.Checked);

    if (checkedCnt == 0) CheckAll = false;
    else if (checkedCnt == cnt) CheckAll = true;
    else CheckAll = null;
}
```

```xml
<StackPanel Spacing="6">
  <CheckBox Content="전체 선택"
            IsThreeState="True"
            IsChecked="{Binding CheckAll}"/>

  <ItemsControl Items="{Binding Options}">
    <ItemsControl.ItemTemplate>
      <DataTemplate>
        <CheckBox Content="{Binding Name}" IsChecked="{Binding Checked, Mode=TwoWay}"/>
      </DataTemplate>
    </ItemsControl.ItemTemplate>
  </ItemsControl>
</StackPanel>
```

> DynamicData 없이도 `Options.CollectionChanged` + 각 항목 `PropertyChanged` 구독으로 동일 구현 가능.

---

## 4. 라디오 버튼 · 토글 스위치 · 슬라이더/프로그레스

### 4.1 RadioButton — 단일 선택(그룹)

```csharp
public string[] PaymentMethods { get; } = { "카드", "계좌이체", "포인트" };

private string _payment = "카드";
public string Payment
{
    get => _payment;
    set => this.RaiseAndSetIfChanged(ref _payment, value);
}
```

```xml
<StackPanel>
  <TextBlock Text="결제 수단"/>
  <StackPanel Orientation="Horizontal" Spacing="12">
    <RadioButton Content="카드"       GroupName="Pay" IsChecked="{Binding Payment, Converter={x:Static converters:StringEqualsConverter.Instance}, ConverterParameter=카드}"/>
    <RadioButton Content="계좌이체"   GroupName="Pay" IsChecked="{Binding Payment, Converter={x:Static converters:StringEqualsConverter.Instance}, ConverterParameter=계좌이체}"/>
    <RadioButton Content="포인트"     GroupName="Pay" IsChecked="{Binding Payment, Converter={x:Static converters:StringEqualsConverter.Instance}, ConverterParameter=포인트}"/>
  </StackPanel>
</StackPanel>
```

간단히 하려면, 각 라디오의 `Checked` 이벤트에서 ViewModel 속성 변경(코드비하인드)도 가능하지만 **Converter**로 MVVM 유지가 깔끔하다. 아래와 같은 `StringEqualsConverter`를 하나 만들어두면 여러 곳에 재사용 가능하다.

```csharp
// Converters/StringEqualsConverter.cs
using Avalonia.Data.Converters;
using System;
using System.Globalization;

public sealed class StringEqualsConverter : IValueConverter
{
    public static StringEqualsConverter Instance { get; } = new();

    public object? Convert(object? value, Type targetType, object? parameter, CultureInfo culture)
        => string.Equals(value?.ToString(), parameter?.ToString(), StringComparison.Ordinal);

    public object? ConvertBack(object? value, Type targetType, object? parameter, CultureInfo culture)
        => (value is bool b && b) ? parameter?.ToString() : BindingOperations.DoNothing;
}
```

### 4.2 ToggleSwitch — On/Off 설정

```csharp
private bool _darkMode;
public bool DarkMode
{
    get => _darkMode;
    set => this.RaiseAndSetIfChanged(ref _darkMode, value);
}
```

```xml
<ToggleSwitch IsChecked="{Binding DarkMode}" Content="다크 모드"/>
```

### 4.3 Slider/ProgressBar — 숫자 바인딩

```csharp
private double _progress;
public double Progress
{
    get => _progress;
    set => this.RaiseAndSetIfChanged(ref _progress, value);
}
```

```xml
<Slider Minimum="0" Maximum="100" Value="{Binding Progress}"/>
<ProgressBar Minimum="0" Maximum="100" Value="{Binding Progress}"/>
```

---

## 5. 종합 ViewModel — 파생 상태 · 명령 활성화

```csharp
// ViewModels/ControlsDemoViewModel.cs (전체형 예시)
using ReactiveUI;
using System;
using System.Collections.ObjectModel;
using System.Linq;
using System.Reactive;
using System.Reactive.Linq;

public partial class ControlsDemoViewModel : ReactiveObject
{
    // DatePicker
    private DateTimeOffset? _selectedDate = DateTimeOffset.Now;
    public DateTimeOffset? SelectedDate
    {
        get => _selectedDate;
        set => this.RaiseAndSetIfChanged(ref _selectedDate, value);
    }

    // ComboBox
    public ObservableCollection<User> Users { get; } = new()
    {
        new User { Id = 1, Name = "홍길동" },
        new User { Id = 2, Name = "이순신" },
        new User { Id = 3, Name = "신사임당" },
    };

    private User? _selectedUser;
    public User? SelectedUser
    {
        get => _selectedUser;
        set => this.RaiseAndSetIfChanged(ref _selectedUser, value);
    }

    // CheckBox
    private bool _isAccepted;
    public bool IsAccepted
    {
        get => _isAccepted;
        set => this.RaiseAndSetIfChanged(ref _isAccepted, value);
    }

    // Toggle
    private bool _darkMode;
    public bool DarkMode
    {
        get => _darkMode;
        set => this.RaiseAndSetIfChanged(ref _darkMode, value);
    }

    // Slider/Progress
    private double _progress;
    public double Progress
    {
        get => _progress;
        set => this.RaiseAndSetIfChanged(ref _progress, value);
    }

    // Summary (파생 상태)
    public string Summary =>
        $"날짜: {SelectedDate:yyyy-MM-dd}, 사용자: {SelectedUser?.Name ?? "-"}, 동의: {(IsAccepted ? "예" : "아니오")}";

    // 버튼 커맨드
    public ReactiveCommand<Unit, Unit> SaveCommand { get; }
    public ReactiveCommand<Unit, Unit> LoadCommand { get; }

    public ControlsDemoViewModel()
    {
        // 파생 상태 변경 알림
        this.WhenAnyValue(vm => vm.SelectedDate, vm => vm.SelectedUser, vm => vm.IsAccepted)
            .Subscribe(_ => this.RaisePropertyChanged(nameof(Summary)));

        // 저장 가능 조건: 날짜 선택 + 사용자 선택 + 동의
        var canSave = this.WhenAnyValue(
            vm => vm.SelectedDate,
            vm => vm.SelectedUser,
            vm => vm.IsAccepted,
            (d, u, a) => d.HasValue && u != null && a);

        SaveCommand = ReactiveCommand.Create(Save, canSave);
        LoadCommand = ReactiveCommand.Create(Load);
    }

    private void Save()
    {
        // 저장: 파일/서비스/메시지 등
    }

    private void Load()
    {
        // 불러오기: 파일/서비스/메시지 등
    }
}
```

---

## 6. 종합 View — 컨트롤 배치/템플릿/포맷

```xml
<!-- Views/ControlsDemoView.axaml -->
<UserControl xmlns="https://github.com/avaloniaui"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:vm="clr-namespace:MyAvaloniaApp.ViewModels"
             xmlns:conv="clr-namespace:MyAvaloniaApp.Converters"
             x:Class="MyAvaloniaApp.Views.ControlsDemoView">

  <UserControl.DataContext>
    <vm:ControlsDemoViewModel/>
  </UserControl.DataContext>

  <ScrollViewer>
    <StackPanel Margin="20" Spacing="16">

      <!-- DatePicker -->
      <StackPanel Spacing="6">
        <TextBlock Text="날짜 선택"/>
        <DatePicker SelectedDate="{Binding SelectedDate}"/>
        <TextBlock Text="{Binding SelectedDate,
                                  StringFormat='선택한 날짜: {0:yyyy-MM-dd}'}"/>
      </StackPanel>

      <!-- ComboBox (User) -->
      <StackPanel Spacing="6">
        <TextBlock Text="사용자 선택"/>
        <ComboBox Items="{Binding Users}" SelectedItem="{Binding SelectedUser}">
          <ComboBox.ItemTemplate>
            <DataTemplate>
              <StackPanel Orientation="Horizontal" Spacing="6">
                <TextBlock Text="{Binding Name}" FontWeight="Bold"/>
                <TextBlock Text="{Binding Id, StringFormat='(ID: {0})'}" Foreground="Gray"/>
              </StackPanel>
            </DataTemplate>
          </ComboBox.ItemTemplate>
        </ComboBox>
        <TextBlock Text="{Binding SelectedUser.Name, StringFormat='선택: {0}'}"/>
      </StackPanel>

      <!-- CheckBox -->
      <StackPanel Spacing="6">
        <CheckBox Content="약관 동의" IsChecked="{Binding IsAccepted}"/>
        <Button Content="저장" Command="{Binding SaveCommand}"/>
      </StackPanel>

      <!-- Toggle & Slider/Progress -->
      <StackPanel Spacing="6">
        <ToggleSwitch IsChecked="{Binding DarkMode}" Content="다크 모드"/>
        <Slider Minimum="0" Maximum="100" Value="{Binding Progress}"/>
        <ProgressBar Minimum="0" Maximum="100" Value="{Binding Progress}"/>
      </StackPanel>

      <Separator/>

      <!-- Summary -->
      <TextBlock Text="{Binding Summary}" FontWeight="Bold" FontSize="16"/>

    </StackPanel>
  </ScrollViewer>
</UserControl>
```

---

## 7. 검증(Validation)과 커맨드 활성화

### 7.1 DataAnnotations (간단)

```csharp
using System.ComponentModel.DataAnnotations;

public sealed class ProfileForm : ReactiveObject
{
    private string? _name;

    [Required(ErrorMessage = "이름은 필수입니다.")]
    public string? Name
    {
        get => _name;
        set => this.RaiseAndSetIfChanged(ref _name, value);
    }

    private int _age;

    [Range(1, 120, ErrorMessage = "나이는 1~120 사이여야 합니다.")]
    public int Age
    {
        get => _age;
        set => this.RaiseAndSetIfChanged(ref _age, value);
    }
}
```

```csharp
// VM에서 폼 검증 → 에러 문자열 바인딩
public string? Errors { get; private set; }

public ReactiveCommand<Unit, Unit> SubmitCommand { get; }

public ControlsDemoViewModel()
{
    SubmitCommand = ReactiveCommand.Create(Submit);
}

private void Submit()
{
    var form = new ProfileForm { Name = SelectedUser?.Name, Age = 30 };
    var ctx = new ValidationContext(form);
    var results = new List<ValidationResult>();
    var ok = Validator.TryValidateObject(form, ctx, results, true);

    Errors = ok ? "검증 통과" : string.Join(Environment.NewLine, results.Select(r => r.ErrorMessage));
    this.RaisePropertyChanged(nameof(Errors));
}
```

```xml
<TextBlock Text="{Binding Errors}" Foreground="Tomato" TextWrapping="Wrap"/>
```

> Avalonia의 `DataValidationErrors`(Attached)와 `INotifyDataErrorInfo`를 사용하면 컨트롤 옆에 에러 템플릿을 표시하는 고급 UX도 가능하다.

---

## 8. Converter 모음(실무 유용)

```csharp
// Converters/BoolToTextConverter.cs
using Avalonia.Data.Converters;
using System;
using System.Globalization;

public sealed class BoolToTextConverter : IValueConverter
{
    public string TrueText { get; set; } = "예";
    public string FalseText { get; set; } = "아니오";

    public object? Convert(object? value, Type targetType, object? parameter, CultureInfo culture)
        => (value is bool b && b) ? TrueText : FalseText;

    public object? ConvertBack(object? value, Type targetType, object? parameter, CultureInfo culture)
        => throw new NotSupportedException();
}
```

```xml
<!-- App.axaml -->
<Application ... xmlns:conv="clr-namespace:MyAvaloniaApp.Converters">
  <Application.Resources>
    <conv:BoolToTextConverter x:Key="BoolText" TrueText="예" FalseText="아니오"/>
  </Application.Resources>
</Application>
```

```xml
<TextBlock Text="{Binding IsAccepted, Converter={StaticResource BoolText}}"/>
```

---

## 9. 저장/복원(간단 JSON 스토리지)

```csharp
// Services/JsonStorageService.cs
using System.Text.Json;

public sealed class JsonStorageService
{
    private readonly string _path;
    public JsonStorageService(string path = "controls-demo.json") => _path = path;

    public async Task SaveAsync(object data)
    {
        var json = JsonSerializer.Serialize(data, new JsonSerializerOptions { WriteIndented = true });
        await File.WriteAllTextAsync(_path, json);
    }

    public async Task<T?> LoadAsync<T>() where T : class
    {
        if (!File.Exists(_path)) return null;
        var json = await File.ReadAllTextAsync(_path);
        return JsonSerializer.Deserialize<T>(json);
    }
}
```

```csharp
// ViewModels/ControlsDemoViewModel.cs (발췌)
private readonly JsonStorageService _storage = new();

private sealed record ControlsSnapshot(
    DateTimeOffset? SelectedDate,
    int? SelectedUserId,
    bool IsAccepted,
    bool DarkMode,
    double Progress);

private ControlsSnapshot Snapshot()
    => new(SelectedDate, SelectedUser?.Id, IsAccepted, DarkMode, Progress);

private void Restore(ControlsSnapshot s)
{
    SelectedDate = s.SelectedDate;
    SelectedUser = Users.FirstOrDefault(u => u.Id == s.SelectedUserId);
    IsAccepted = s.IsAccepted;
    DarkMode   = s.DarkMode;
    Progress   = s.Progress;
}

private async void Save()
{
    await _storage.SaveAsync(Snapshot());
}

private async void Load()
{
    var s = await _storage.LoadAsync<ControlsSnapshot>();
    if (s != null) Restore(s);
}
```

---

## 10. 성능/유지보수 팁

- **바인딩 경로 단순화**: `SelectedItem` → 파생 속성(Id/Name)을 VM에서 노출.
- **DataTemplate** 정적 선언: 런타임 탐색 줄이고 유지보수 가시성 향상.
- **ReactiveUI WhenAnyValue**: 파생 속성 재계산을 한 곳에서.
- **Converter**는 가볍게, 무거운 로직은 VM/서비스에서 처리.
- **검증/저장**은 UI 이벤트에 넣지 말고 **Command**로 일원화.
- **테스트**는 VM 중심으로(컨트롤은 스냅샷/시나리오 소수만).

---

## 11. 통합 미니 과제

요구

1) 날짜를 선택하고
2) 사용자 콤보에서 사용자 선택,
3) 약관 체크 후 저장 버튼 활성화,
4) 저장 시 JSON 스냅샷,
5) 다음 실행에서 불러오기.

구성

- 위의 `ControlsDemoViewModel` + `ControlsDemoView.axaml` + `JsonStorageService` 조합이면 그대로 충족한다.
- 커맨드 촉발(저장/불러오기)을 단축키(예: Ctrl+S/Ctrl+O)로 추가하는 것도 쉽다(키 바인딩은 `InputBindings` 또는 코드비하인드 이벤트 → Command 라우팅).

---

## 12. 결론

- **DatePicker/ComboBox/CheckBox**는 MVVM에서 **값·객체·상태**를 표현하는 기본 축이다.
- 단순 바인딩에서 출발하되, **DataTemplate/Converter/Validation/Command**를 조합하면 현업의 대부분 요구(표시/검증/저장/복원/조건 활성화)에 충분히 대응한다.
- 선택 모델이 복잡해질수록(멀티/삼상/의존) **파생 속성**과 **반응형 조합(WhenAnyValue)** 으로 VM 로직을 정돈하라. View는 최대한 얇게 유지하는 게 유지보수·테스트 모두에 유리하다.
