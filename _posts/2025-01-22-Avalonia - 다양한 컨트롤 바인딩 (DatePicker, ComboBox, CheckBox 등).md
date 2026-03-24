---
layout: post
title: Avalonia - 다양한 컨트롤 바인딩 (DatePicker, ComboBox, CheckBox 등)
date: 2025-01-22 19:20:23 +0900
category: Avalonia
---
# Avalonia MVVM: 다양한 컨트롤 바인딩

MVVM 패턴에서는 View가 ViewModel의 속성과 명령에 바인딩된다. TextBox 외에도 날짜 선택, 콤보박스, 체크박스 등 다양한 컨트롤을 ViewModel과 연결하는 방법을 익히면 대부분의 폼 기반 UI를 구현할 수 있다.

## 프로젝트 구조

```
MyAvaloniaApp/
├── ViewModels/
│   └── ControlsDemoViewModel.cs
├── Views/
│   └── ControlsDemoView.axaml
├── Models/
│   └── User.cs
├── Converters/
│   ├── BoolToTextConverter.cs
│   └── DateTimeOffsetFormatConverter.cs
└── Services/
    └── JsonStorageService.cs
```

## DatePicker: 날짜 선택

DatePicker의 `SelectedDate` 속성은 `DateTimeOffset?` 타입이다. ViewModel에서도 동일한 타입으로 속성을 만들고 양방향 바인딩한다.

```csharp
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

XAML에서는 다음과 같이 바인딩한다.

```xml
<DatePicker SelectedDate="{Binding SelectedDate}" />
<TextBlock Text="{Binding SelectedDate, StringFormat='선택: {0:yyyy-MM-dd}'}" />
```

### 날짜 범위 제한

날짜 선택의 범위를 제한해야 할 때가 있다. ViewModel에서 `MinDate`와 `MaxDate` 속성을 제공하고, 저장 버튼의 활성화 여부를 날짜 유효성에 따라 결정할 수 있다.

```csharp
public DateTimeOffset MinDate { get; } = new DateTimeOffset(2020, 1, 1, 0, 0, 0, TimeSpan.Zero);
public DateTimeOffset MaxDate { get; } = new DateTimeOffset(2030, 12, 31, 0, 0, 0, TimeSpan.Zero);

public ReactiveCommand<Unit, Unit> ApplyDateCommand { get; }

public ControlsDemoViewModel()
{
    var canApply = this.WhenAnyValue(x => x.SelectedDate)
        .Select(d => d.HasValue && d.Value >= MinDate && d.Value <= MaxDate);
    ApplyDateCommand = ReactiveCommand.Create(ApplyDate, canApply);
}

private void ApplyDate() { /* 저장 로직 */ }
```

버튼은 Command에 바인딩하면 자동으로 활성/비활성 상태가 관리된다.

```xml
<Button Content="날짜 적용" Command="{Binding ApplyDateCommand}" />
```

## ComboBox: 선택형 컨트롤

ComboBox는 여러 항목 중 하나를 선택할 때 사용한다. 항목은 단순 문자열일 수도 있고, 복잡한 객체일 수도 있다.

### 문자열 목록

```csharp
public ObservableCollection<string> Fruits { get; } = new() { "사과", "바나나", "포도" };
private string? _selectedFruit;
public string? SelectedFruit
{
    get => _selectedFruit;
    set => this.RaiseAndSetIfChanged(ref _selectedFruit, value);
}
```

```xml
<ComboBox Items="{Binding Fruits}" SelectedItem="{Binding SelectedFruit}" />
```

### 객체 목록

객체를 표시할 때는 `ToString()`을 오버라이드하거나 ItemTemplate을 사용해 원하는 형식으로 보여줄 수 있다.

```csharp
public class User
{
    public int Id { get; init; }
    public string Name { get; init; } = "";
    public override string ToString() => Name;
}

public ObservableCollection<User> Users { get; } = new()
{
    new User { Id = 1, Name = "홍길동" },
    new User { Id = 2, Name = "이순신" }
};

private User? _selectedUser;
public User? SelectedUser
{
    get => _selectedUser;
    set => this.RaiseAndSetIfChanged(ref _selectedUser, value);
}

// 선택된 사용자의 ID만 따로 관리하고 싶다면 파생 속성 활용
public int? SelectedUserId => SelectedUser?.Id;
```

```xml
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
<TextBlock Text="{Binding SelectedUser.Name, StringFormat='선택: {0}'}" />
```

### Enum 바인딩

Enum 타입은 `Enum.GetValues`로 배열을 만들어 ItemsSource로 제공한다.

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
<ComboBox Items="{Binding Priorities}" SelectedItem="{Binding SelectedPriority}" />
```

## CheckBox: 불리언 상태

CheckBox는 `IsChecked` 속성을 `bool` 또는 `bool?`에 바인딩한다. 일반 체크박스는 두 가지 상태(true/false)를, `IsThreeState="True"`를 설정하면 세 가지 상태(true/false/null)를 사용할 수 있다.

```csharp
private bool _isAccepted;
public bool IsAccepted
{
    get => _isAccepted;
    set => this.RaiseAndSetIfChanged(ref _isAccepted, value);
}
```

```xml
<CheckBox Content="약관 동의" IsChecked="{Binding IsAccepted}" />
<Button Content="계속" IsEnabled="{Binding IsAccepted}" />
```

### 마스터 체크박스

전체 선택/해제를 위한 마스터 체크박스는 세 가지 상태를 활용한다. 모든 하위 항목이 체크되면 true, 모두 해제되면 false, 일부만 체크되면 null(Indeterminate)로 표시한다.

```csharp
public class OptionItem : ReactiveObject
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
    new OptionItem { Name = "SMS 알림" }
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
            foreach (var opt in Options)
                opt.Checked = value.Value;
        }
    }
}

// 하위 항목의 변경을 감지해 마스터 상태 갱신
public ControlsDemoViewModel()
{
    Options
        .ToObservableChangeSet()
        .AutoRefresh(x => x.Checked)
        .Throttle(TimeSpan.FromMilliseconds(50))
        .Subscribe(_ => UpdateMasterCheck());
}

private void UpdateMasterCheck()
{
    var all = Options.All(x => x.Checked);
    var any = Options.Any(x => x.Checked);
    CheckAll = all ? true : any ? null : false;
}
```

XAML에서는 마스터 체크박스에 `IsThreeState="True"`를 설정한다.

```xml
<CheckBox Content="전체 선택" IsThreeState="True" IsChecked="{Binding CheckAll}" />
<ItemsControl Items="{Binding Options}">
    <ItemsControl.ItemTemplate>
        <DataTemplate>
            <CheckBox Content="{Binding Name}" IsChecked="{Binding Checked, Mode=TwoWay}" />
        </DataTemplate>
    </ItemsControl.ItemTemplate>
</ItemsControl>
```

## 라디오 버튼 그룹

라디오 버튼은 그룹 내에서 단일 선택을 구현한다. ViewModel에서는 선택된 값을 문자열이나 Enum으로 관리하고, 변환기로 체크 상태를 바인딩한다.

```csharp
private string _payment = "카드";
public string Payment
{
    get => _payment;
    set => this.RaiseAndSetIfChanged(ref _payment, value);
}
```

변환기: 현재 값과 파라미터가 같으면 true, 다르면 false를 반환한다.

```csharp
public class StringEqualsConverter : IValueConverter
{
    public static StringEqualsConverter Instance { get; } = new();
    public object? Convert(object? value, Type targetType, object? parameter, CultureInfo culture)
        => string.Equals(value?.ToString(), parameter?.ToString(), StringComparison.Ordinal);
    public object? ConvertBack(object? value, Type targetType, object? parameter, CultureInfo culture)
        => (value is true) ? parameter?.ToString() : BindingOperations.DoNothing;
}
```

```xml
<StackPanel Orientation="Horizontal" Spacing="12">
    <RadioButton Content="카드" GroupName="Pay"
                 IsChecked="{Binding Payment, Converter={x:Static conv:StringEqualsConverter.Instance}, ConverterParameter=카드}" />
    <RadioButton Content="계좌이체" GroupName="Pay"
                 IsChecked="{Binding Payment, Converter={x:Static conv:StringEqualsConverter.Instance}, ConverterParameter=계좌이체}" />
</StackPanel>
```

## 슬라이더와 프로그레스 바

슬라이더와 프로그레스 바는 `double` 타입의 값에 바인딩한다.

```csharp
private double _progress;
public double Progress
{
    get => _progress;
    set => this.RaiseAndSetIfChanged(ref _progress, value);
}
```

```xml
<Slider Minimum="0" Maximum="100" Value="{Binding Progress}" />
<ProgressBar Minimum="0" Maximum="100" Value="{Binding Progress}" />
```

## ToggleSwitch

토글 스위치는 `bool` 값에 바인딩한다.

```csharp
private bool _darkMode;
public bool DarkMode
{
    get => _darkMode;
    set => this.RaiseAndSetIfChanged(ref _darkMode, value);
}
```

```xml
<ToggleSwitch IsChecked="{Binding DarkMode}" Content="다크 모드" />
```

## 변환기(Converter) 모음

자주 사용하는 변환기를 미리 만들어 두면 XAML에서 재사용할 수 있다.

### 불리언을 텍스트로 변환

```csharp
public class BoolToTextConverter : IValueConverter
{
    public string TrueText { get; set; } = "예";
    public string FalseText { get; set; } = "아니오";
    public object? Convert(object? value, Type targetType, object? parameter, CultureInfo culture)
        => (value is true) ? TrueText : FalseText;
    public object? ConvertBack(object? value, Type targetType, object? parameter, CultureInfo culture)
        => throw new NotSupportedException();
}
```

### DateTimeOffset을 문자열로 포맷

```csharp
public class DateTimeOffsetFormatConverter : IValueConverter
{
    public string Format { get; set; } = "yyyy-MM-dd";
    public object? Convert(object? value, Type targetType, object? parameter, CultureInfo culture)
        => value is DateTimeOffset dto ? dto.ToString(Format, culture) : string.Empty;
    public object? ConvertBack(object? value, Type targetType, object? parameter, CultureInfo culture)
        => DateTimeOffset.TryParse(value?.ToString(), culture, DateTimeStyles.None, out var dto) ? dto : null;
}
```

## 파생 상태와 명령 활성화

ViewModel에서는 여러 속성의 조합으로 버튼 활성화 여부나 요약 텍스트를 자동으로 계산할 수 있다.

```csharp
public string Summary =>
    $"날짜: {SelectedDate:yyyy-MM-dd}, 사용자: {SelectedUser?.Name ?? "-"}, 동의: {(IsAccepted ? "예" : "아니오")}";

public ReactiveCommand<Unit, Unit> SaveCommand { get; }

public ControlsDemoViewModel()
{
    // Summary 갱신
    this.WhenAnyValue(x => x.SelectedDate, x => x.SelectedUser, x => x.IsAccepted)
        .Subscribe(_ => this.RaisePropertyChanged(nameof(Summary)));

    var canSave = this.WhenAnyValue(
        x => x.SelectedDate,
        x => x.SelectedUser,
        x => x.IsAccepted,
        (d, u, a) => d.HasValue && u != null && a);
    SaveCommand = ReactiveCommand.Create(Save, canSave);
}
```

## 저장과 복원 (JSON)

간단한 JSON 파일로 ViewModel의 상태를 저장하고 복원할 수 있다.

```csharp
public class JsonStorageService
{
    private readonly string _path;
    public JsonStorageService(string path = "demo.json") => _path = path;
    public async Task SaveAsync(object data) =>
        await File.WriteAllTextAsync(_path, JsonSerializer.Serialize(data));
    public async Task<T?> LoadAsync<T>() where T : class =>
        File.Exists(_path) ? JsonSerializer.Deserialize<T>(await File.ReadAllTextAsync(_path)) : null;
}

// 스냅샷 레코드 정의
private sealed record ControlsSnapshot(
    DateTimeOffset? SelectedDate,
    int? SelectedUserId,
    bool IsAccepted,
    bool DarkMode,
    double Progress);

private ControlsSnapshot Snapshot() =>
    new(SelectedDate, SelectedUser?.Id, IsAccepted, DarkMode, Progress);

private void Restore(ControlsSnapshot s)
{
    SelectedDate = s.SelectedDate;
    SelectedUser = Users.FirstOrDefault(u => u.Id == s.SelectedUserId);
    IsAccepted = s.IsAccepted;
    DarkMode = s.DarkMode;
    Progress = s.Progress;
}

private async void Save() => await _storage.SaveAsync(Snapshot());
private async void Load()
{
    var s = await _storage.LoadAsync<ControlsSnapshot>();
    if (s != null) Restore(s);
}
```

## 검증(Validation)

간단한 검증은 DataAnnotations를 활용할 수 있다. ViewModel에서 `Validator.TryValidateObject`로 검증 결과를 문자열로 제공한다.

```csharp
public class ProfileForm
{
    [Required(ErrorMessage = "이름은 필수입니다.")]
    public string? Name { get; set; }
    [Range(1, 120, ErrorMessage = "나이는 1~120 사이여야 합니다.")]
    public int Age { get; set; }
}

private string? _validationErrors;
public string? ValidationErrors
{
    get => _validationErrors;
    private set => this.RaiseAndSetIfChanged(ref _validationErrors, value);
}

private bool Validate()
{
    var form = new ProfileForm { Name = SelectedUser?.Name, Age = 30 };
    var context = new ValidationContext(form);
    var results = new List<ValidationResult>();
    var isValid = Validator.TryValidateObject(form, context, results, true);
    ValidationErrors = isValid ? null : string.Join(Environment.NewLine, results.Select(r => r.ErrorMessage));
    return isValid;
}
```

XAML에서는 오류를 표시할 TextBlock을 두고 바인딩한다.

```xml
<TextBlock Text="{Binding ValidationErrors}" Foreground="Red" TextWrapping="Wrap" />
```

## 성능과 유지보수 팁

| 상황 | 권장 방법 |
|------|-----------|
| 자주 바뀌는 파생 속성 | WhenAnyValue로 자동 갱신, 수동 RaisePropertyChanged 최소화 |
| 복잡한 ItemTemplate | 정적 리소스로 정의해 재사용 |
| 많은 항목의 ComboBox | 가상화가 적용되는 ItemsControl 사용 (기본적으로 지원됨) |
| 저장/복원 | ViewModel 스냅샷을 직렬화, 불필요한 속성 제외 |
| 테스트 | ViewModel 단위 테스트로 로직 검증, Converter는 별도 테스트 |

## 종합 예제 ViewModel

위의 모든 요소를 포함한 ViewModel과 View를 조합하면 다양한 컨트롤을 MVVM 패턴으로 관리할 수 있다. XAML에서는 각 컨트롤을 적절히 배치하고, ViewModel의 속성과 명령에 바인딩한다.

```xml
<StackPanel Margin="20" Spacing="16">
    <!-- 날짜 선택 -->
    <DatePicker SelectedDate="{Binding SelectedDate}" />
    <!-- 사용자 선택 -->
    <ComboBox Items="{Binding Users}" SelectedItem="{Binding SelectedUser}" />
    <!-- 약관 동의 -->
    <CheckBox Content="약관 동의" IsChecked="{Binding IsAccepted}" />
    <!-- 저장 버튼 -->
    <Button Content="저장" Command="{Binding SaveCommand}" />
    <!-- 요약 표시 -->
    <TextBlock Text="{Binding Summary}" />
</StackPanel>
```

## 결론

Avalonia에서 DatePicker, ComboBox, CheckBox 등 다양한 컨트롤을 MVVM 패턴으로 바인딩하는 방법은 기본적인 원칙을 따르면 복잡하지 않다. 각 컨트롤의 핵심 속성(SelectedDate, SelectedItem, IsChecked, Value 등)을 ViewModel의 속성과 양방향 바인딩하고, 필요에 따라 변환기나 검증, 파생 상태 계산을 추가하면 된다. 이렇게 구성하면 UI와 로직이 분리되어 유지보수와 테스트가 훨씬 쉬워진다.