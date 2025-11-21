---
layout: post
title: Avalonia - MVVM
date: 2025-01-03 20:20:23 +0900
category: Avalonia
---
# MVVM 패턴이란? (Model–View–ViewModel)

## MVVM의 정의

**MVVM(Model–View–ViewModel)**은 XAML 계열 UI 프레임워크(WPF, Avalonia, Xamarin, MAUI 등)에서 광범위하게 쓰이는 **관심사 분리(Separation of Concerns)** 아키텍처 패턴이다.
핵심 목표는 **UI(View)**와 **비즈니스 로직(Model)** 사이의 결합을 **ViewModel**을 통해 느슨하게 만드는 것이다. ViewModel은 **상태(state)**와 **명령(command)**를 노출하고, View는 **바인딩(binding)**으로 이를 소비한다. View와 ViewModel은 서로 **직접 참조하지 않거나(권장)**, 최소한으로만 참조하도록 설계한다.

---

## 구성 요소 (요약 + 확장)

| 구성 요소 | 핵심 역할 | 구현 포인트 |
|-----------|-----------|-------------|
| Model | 데이터/도메인 로직(엔티티, 리포지토리, API 클라이언트 등) | UI 프레임워크에 의존하지 않도록 순수 .NET 코드 유지 |
| View | XAML UI(레이아웃, 스타일, 템플릿, 트리거) | 로직 최소화. DataContext로 ViewModel을 연결 |
| ViewModel | 상태/명령/검증/네비게이션 허브 | `INotifyPropertyChanged`(INPC), `ICommand`, 비동기, DI, 테스트 용이성 |

---

## 흐름 구조

```
[사용자] → View ↔ ViewModel ↔ Model
                         ↑
                  (Command, Binding)
```

- **View ↔ ViewModel**: 바인딩(OneWay/TwoWay)과 명령(ICommand)으로 연결.
- **ViewModel ↔ Model**: 데이터 쿼리/갱신, 서비스 호출, 상태 변환.
- **단방향 의존 권장**: View는 ViewModel에 의존, ViewModel은 View에 의존하지 않음(다이얼로그/네비게이션은 서비스 추상화로 해결).

---

## 왜 MVVM인가? (장점 재정리)

| 이유 | 설명 |
|------|------|
| UI–로직 분리 | UI 수정이 로직에, 로직 변경이 UI에 최소 영향 |
| 테스트 용이 | ViewModel/서비스를 View 없이 단위 테스트 가능 |
| 재사용성 | 동일 ViewModel을 다른 View(XAML)에서 재사용 |
| 유지보수 | 의존성 역전/DI로 변형/교체/확장이 쉬움 |
| 확장성 | DataTemplate/Style/Resource로 손쉬운 UI 확장 |

---

## Avalonia에서 MVVM 구현: 시작하기

### 솔루션 구조(권장 템플릿)

```
MyApp/
├── MyApp.csproj
├── Program.cs
├── App.axaml
├── App.axaml.cs
├── Resources/
│   ├── Colors.axaml
│   └── Styles.axaml
├── Models/
│   └── Person.cs
├── Services/
│   ├── IPeopleService.cs
│   └── PeopleService.cs
├── ViewModels/
│   ├── ViewModelBase.cs
│   ├── MainWindowViewModel.cs
│   ├── HomeViewModel.cs
│   └── SettingsViewModel.cs
└── Views/
    ├── MainWindow.axaml
    ├── HomeView.axaml
    └── SettingsView.axaml
```

### 패키지 선택

- **ReactiveUI 기반**: `ReactiveObject`, `ReactiveCommand`로 간결한 INPC/커맨드 구현
- **CommunityToolkit.Mvvm 기반**: `[ObservableProperty]`, `[RelayCommand]` 소스 생성기로 보일러플레이트 제거

본 가이드는 두 방식을 모두 소개한다. 둘 중 조직 표준/선호에 맞춰 선택하자.

---

## 기본 예제 (확장)

### Model

```csharp
// Models/Person.cs
namespace MyApp.Models
{
    public sealed class Person
    {
        public string Name { get; }
        public int Age { get; }

        public Person(string name, int age)
        {
            Name = name;
            Age = age;
        }

        public Person Birthday() => new(Name, Age + 1);
    }
}
```

### ViewModel (ReactiveUI 버전)

```csharp
// ViewModels/ViewModelBase.cs
using ReactiveUI;

namespace MyApp.ViewModels
{
    public class ViewModelBase : ReactiveObject
    {
    }
}
```

```csharp
// ViewModels/MainWindowViewModel.cs
using ReactiveUI;
using System.Reactive;
using MyApp.Models;

namespace MyApp.ViewModels
{
    public class MainWindowViewModel : ViewModelBase
    {
        private string _name = "홍길동";
        public string Name
        {
            get => _name;
            set => this.RaiseAndSetIfChanged(ref _name, value);
        }

        public ReactiveCommand<Unit, Unit> GreetCommand { get; }

        public MainWindowViewModel()
        {
            GreetCommand = ReactiveCommand.Create(() =>
            {
                Name = $"안녕하세요, {Name}!";
            });
        }
    }
}
```

### View (Avalonia XAML)

> Avalonia XAML 네임스페이스 매핑은 일반적으로 `using:` 구문을 사용한다. 다음 예시는 `using:MyApp.ViewModels`를 사용한다.

```xml
<!-- Views/MainWindow.axaml -->
<Window xmlns="https://github.com/avaloniaui"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:vm="using:MyApp.ViewModels"
        x:Class="MyApp.Views.MainWindow"
        Title="MVVM Demo"
        Width="400" Height="200">
  <Window.DataContext>
    <vm:MainWindowViewModel/>
  </Window.DataContext>

  <StackPanel HorizontalAlignment="Center" VerticalAlignment="Center" Spacing="8">
    <TextBox Text="{Binding Name, Mode=TwoWay}" Width="200"/>
    <Button Content="인사하기" Command="{Binding GreetCommand}" />
  </StackPanel>
</Window>
```

### 코드 비하인드

```csharp
// Views/MainWindow.axaml.cs
using Avalonia.Controls;

namespace MyApp.Views
{
    public partial class MainWindow : Window
    {
        public MainWindow()
        {
            InitializeComponent();
            // DataContext는 XAML에서 설정했으므로 생략 가능
        }
    }
}
```

---

## 설계 — ReactiveUI vs CommunityToolkit

### ReactiveUI `ReactiveCommand`

- 장점: 비동기/예외 스트림/CanExecute 스트림 연계가 자연스러움
- 예: API 호출 후 상태 갱신

```csharp
using ReactiveUI;
using System.Reactive;
using System.Reactive.Linq;
using System.Threading.Tasks;

public class HomeViewModel : ViewModelBase
{
    private string _status = "대기";
    public string Status
    {
        get => _status;
        set => this.RaiseAndSetIfChanged(ref _status, value);
    }

    public ReactiveCommand<Unit, string> LoadCommand { get; }

    public HomeViewModel()
    {
        // 실행 가능 여부를 외부 상태로 제어하고 싶다면 IObservable<bool>을 전달
        LoadCommand = ReactiveCommand.CreateFromTask(async () =>
        {
            Status = "로딩 중...";
            await Task.Delay(500); // 예시: API call
            return "완료";
        });

        // 결과 스트림 구독
        LoadCommand.Subscribe(result => Status = result);

        // 에러 스트림 처리
        LoadCommand.ThrownExceptions
            .Subscribe(ex => Status = $"오류: {ex.Message}");
    }
}
```

XAML:

```xml
<Button Content="불러오기" Command="{Binding LoadCommand}"/>
<TextBlock Text="{Binding Status}"/>
```

### CommunityToolkit `[RelayCommand]`

```csharp
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using System.Threading.Tasks;

public partial class HomeViewModel : ObservableObject
{
    [ObservableProperty]
    private string status = "대기";

    [RelayCommand]
    private async Task LoadAsync()
    {
        Status = "로딩 중...";
        await Task.Delay(500); // 예시: API call
        Status = "완료";
    }
}
```

XAML:

```xml
<Button Content="불러오기" Command="{Binding LoadAsyncCommand}"/>
<TextBlock Text="{Binding Status}"/>
```

---

## 바인딩 심화 — 모드, 경로, 업데이트, 변환기

### 모드

- `OneWay`(기본 TextBlock 등), `TwoWay`(TextBox 등), `OneTime`(초기값만), `OneWayToSource`(드묾)
- 예:

```xml
<TextBlock Text="{Binding Title, Mode=OneWay}"/>
<TextBox Text="{Binding UserInput, Mode=TwoWay, UpdateSourceTrigger=PropertyChanged}"/>
```

### 변환기(IValueConverter)

```csharp
// Converters/BoolToTextConverter.cs
using System;
using Avalonia.Data.Converters;
using System.Globalization;

public sealed class BoolToTextConverter : IValueConverter
{
    public object? Convert(object? value, Type targetType, object? parameter, CultureInfo culture)
        => value is bool b && b ? "참" : "거짓";

    public object? ConvertBack(object? value, Type targetType, object? parameter, CultureInfo culture)
        => (value as string) == "참";
}
```

XAML 등록/사용:

```xml
<Window xmlns:conv="using:MyApp.Converters">
  <Window.Resources>
    <conv:BoolToTextConverter x:Key="BoolToText"/>
  </Window.Resources>
  <TextBlock Text="{Binding Flag, Converter={StaticResource BoolToText}}"/>
</Window>
```

### DataTemplate — ViewModel→View 자동 매핑

```xml
<!-- App.axaml -->
<Application
  xmlns="https://github.com/avaloniaui"
  xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
  xmlns:vm="using:MyApp.ViewModels"
  xmlns:views="using:MyApp.Views"
  x:Class="MyApp.App">

  <Application.DataTemplates>
    <DataTemplate DataType="{x:Type vm:HomeViewModel}">
      <views:HomeView/>
    </DataTemplate>
    <DataTemplate DataType="{x:Type vm:SettingsViewModel}">
      <views:SettingsView/>
    </DataTemplate>
  </Application.DataTemplates>
</Application>
```

```xml
<!-- Views/MainWindow.axaml (셸) -->
<ContentControl Content="{Binding CurrentViewModel}"/>
```

ViewModel만 바꾸면 View가 자동으로 교체된다.

---

## — DataAnnotations/커스텀

### CommunityToolkit `ObservableValidator`

```csharp
using CommunityToolkit.Mvvm.ComponentModel;
using System.ComponentModel.DataAnnotations;

public partial class AccountViewModel : ObservableValidator
{
    [ObservableProperty]
    [Required(ErrorMessage = "사용자 이름은 필수입니다.")]
    private string? userName;

    [ObservableProperty]
    [EmailAddress(ErrorMessage = "이메일 형식이 아닙니다.")]
    private string? email;

    partial void OnUserNameChanged(string? value) => ValidateAllProperties();
    partial void OnEmailChanged(string? value) => ValidateAllProperties();
}
```

XAML(간단한 에러 표시 예):

```xml
<TextBox Text="{Binding UserName, Mode=TwoWay}"/>
<!-- 컨트롤 템플릿/Adorner로 에러 스타일링 가능 -->
```

### ReactiveUI ValidatableObject(직접 구현)

- Reactive Validation 패턴 혹은 `IDataErrorInfo`/`INotifyDataErrorInfo`를 구현해도 된다.

---

## — 셸 + 서비스

### + CurrentViewModel 패턴

```csharp
// ViewModels/MainWindowViewModel.cs
using ReactiveUI;
using System.Reactive;

public class MainWindowViewModel : ViewModelBase
{
    private ViewModelBase _currentViewModel = new HomeViewModel();
    public ViewModelBase CurrentViewModel
    {
        get => _currentViewModel;
        set => this.RaiseAndSetIfChanged(ref _currentViewModel, value);
    }

    public ReactiveCommand<Unit, Unit> NavigateHomeCommand { get; }
    public ReactiveCommand<Unit, Unit> NavigateSettingsCommand { get; }

    public MainWindowViewModel()
    {
        NavigateHomeCommand = ReactiveCommand.Create(() => CurrentViewModel = new HomeViewModel());
        NavigateSettingsCommand = ReactiveCommand.Create(() => CurrentViewModel = new SettingsViewModel());
    }
}
```

XAML:

```xml
<DockPanel>
  <StackPanel DockPanel.Dock="Top" Orientation="Horizontal" Spacing="8" Margin="8">
    <Button Content="홈" Command="{Binding NavigateHomeCommand}"/>
    <Button Content="설정" Command="{Binding NavigateSettingsCommand}"/>
  </StackPanel>
  <ContentControl Content="{Binding CurrentViewModel}" Margin="12"/>
</DockPanel>
```

### DI 기반 네비게이션 서비스

```csharp
// Services/INavigationService.cs
using MyApp.ViewModels;

public interface INavigationService
{
    ViewModelBase Current { get; }
    void NavigateTo<T>() where T : ViewModelBase;
}
```

```csharp
// Services/NavigationService.cs
using System;
using Microsoft.Extensions.DependencyInjection;
using MyApp.ViewModels;

public sealed class NavigationService : INavigationService
{
    private readonly IServiceProvider _sp;
    public ViewModelBase Current { get; private set; }

    public NavigationService(IServiceProvider sp)
    {
        _sp = sp;
        Current = _sp.GetRequiredService<HomeViewModel>();
    }

    public void NavigateTo<T>() where T : ViewModelBase
        => Current = _sp.GetRequiredService<T>();
}
```

셸 VM에서 `CurrentViewModel = _nav.Current`를 바인딩하고, 버튼은 `_nav.NavigateTo<SettingsViewModel>()`로 이동.

---

## — 서비스 추상화

ViewModel이 `Window`를 직접 생성하지 않도록 **IDialogService**를 둔다.

```csharp
// Services/IDialogService.cs
using System.Threading.Tasks;

public interface IDialogService
{
    Task<string?> ShowInputAsync(string title, string prompt);
}
```

```csharp
// Services/DialogService.cs (간단 구현)
using Avalonia.Controls;
using System.Threading.Tasks;

public sealed class DialogService : IDialogService
{
    private readonly Window _owner;
    public DialogService(Window owner) => _owner = owner;

    public async Task<string?> ShowInputAsync(string title, string prompt)
    {
        var dlg = new Window { Title = title, Width = 360, Height = 160 };
        // 실제로는 별도의 XAML 다이얼로그를 구성하는 것을 권장
        return await dlg.ShowDialog<string?>(_owner);
    }
}
```

ViewModel은 인터페이스에만 의존하므로 테스트가 쉬워진다.

---

## — Microsoft.Extensions.DependencyInjection

```csharp
// App.axaml.cs
using Avalonia;
using Avalonia.Controls.ApplicationLifetimes;
using Microsoft.Extensions.DependencyInjection;
using MyApp.ViewModels;
using MyApp.Services;

public partial class App : Application
{
    public static ServiceProvider Services { get; private set; } = default!;

    public override void OnFrameworkInitializationCompleted()
    {
        var sc = new ServiceCollection();

        // 서비스/리포지토리
        sc.AddSingleton<INavigationService, NavigationService>();
        sc.AddSingleton<IPeopleService, PeopleService>();

        // ViewModel
        sc.AddSingleton<MainWindowViewModel>();
        sc.AddTransient<HomeViewModel>();
        sc.AddTransient<SettingsViewModel>();

        Services = sc.BuildServiceProvider();

        if (ApplicationLifetime is IClassicDesktopStyleApplicationLifetime desktop)
        {
            var shell = Services.GetRequiredService<MainWindowViewModel>();
            desktop.MainWindow = new Views.MainWindow { DataContext = shell };
        }

        base.OnFrameworkInitializationCompleted();
    }
}
```

---

## 비동기·취소·예외 — 견고한 ViewModel

### ReactiveUI: 취소 토큰/에러 스트림

```csharp
public class SearchViewModel : ViewModelBase
{
    private readonly CancellationTokenSource _cts = new();

    public ReactiveCommand<string, string[]> SearchCommand { get; }

    public SearchViewModel()
    {
        SearchCommand = ReactiveCommand.CreateFromTask<string, string[]>(async query =>
        {
            // API 호출 예시
            await Task.Delay(300, _cts.Token);
            return new[] { "결과1", "결과2" };
        });

        SearchCommand.ThrownExceptions.Subscribe(ex => /* 상태/로그 갱신 */);
    }

    public void Cancel() => _cts.Cancel();
}
```

### Toolkit: `AsyncRelayCommand` + 취소

```csharp
public partial class SearchViewModel : ObservableObject
{
    private CancellationTokenSource? _cts;

    [ObservableProperty] private string[] results = Array.Empty<string>();

    public IAsyncRelayCommand<string> SearchCommand { get; }

    public SearchViewModel()
    {
        SearchCommand = new AsyncRelayCommand<string>(SearchAsync);
    }

    private async Task SearchAsync(string query)
    {
        _cts?.Cancel();
        _cts = new CancellationTokenSource();

        await Task.Delay(300, _cts.Token);
        Results = new[] { "결과1", "결과2" };
    }
}
```

---

## 리스트/그리드/가상화

### ObservableCollection + DataGrid

```xml
<DataGrid Items="{Binding People}" AutoGenerateColumns="False">
  <DataGrid.Columns>
    <DataGridTextColumn Header="이름" Binding="{Binding Name}"/>
    <DataGridTextColumn Header="나이" Binding="{Binding Age}"/>
  </DataGrid.Columns>
</DataGrid>
```

ViewModel:

```csharp
public ObservableCollection<Person> People { get; } = new();
```

대량 데이터에서는 **가상화**되는 컨트롤(DataGrid 등)과 **배치 추가**를 고려하자.

---

## 디자인 타임 데이터(Design.DataContext)

디자인 상태에서 XAML 힌트를 보려면:

```xml
<!-- Views/HomeView.axaml -->
<UserControl
  xmlns="https://github.com/avaloniaui"
  xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
  xmlns:d="https://github.com/avaloniaui/2016/xaml"
  xmlns:vm="using:MyApp.ViewModels"
  x:Class="MyApp.Views.HomeView">

  <Design.DataContext>
    <vm:HomeViewModel/>
  </Design.DataContext>

  <StackPanel Margin="12">
    <TextBlock Text="{Binding Status}"/>
  </StackPanel>
</UserControl>
```

---

## — ViewModel 중심

### xUnit 테스트 프로젝트 생성

```bash
dotnet new xunit -o MyApp.Tests
cd MyApp.Tests
dotnet add reference ../MyApp/MyApp.csproj
```

### ViewModel 테스트 예시

```csharp
using Xunit;
using MyApp.ViewModels;

public class MainWindowViewModelTests
{
    [Fact]
    public void Greeting_Default()
    {
        var vm = new MainWindowViewModel();
        Assert.False(string.IsNullOrWhiteSpace(vm.Name));
    }

    [Fact]
    public void GreetCommand_ChangesName()
    {
        var vm = new MainWindowViewModel { Name = "테스터" };
        vm.GreetCommand.Execute().Subscribe(); // ReactiveUI 실행
        Assert.Contains("안녕하세요", vm.Name);
    }
}
```

Toolkit 사용 시 `RelayCommand`/`AsyncRelayCommand` 호출 방식에 맞게 수정한다.

---

## 리소스/스타일/테마 — 전역 관리

`App.axaml`에서 리소스 병합:

```xml
<Application.Resources>
  <ResourceDictionary>
    <ResourceDictionary.MergedDictionaries>
      <ResourceInclude Source="avares://MyApp/Resources/Colors.axaml"/>
      <ResourceInclude Source="avares://MyApp/Resources/Styles.axaml"/>
    </ResourceDictionary.MergedDictionaries>
    <SolidColorBrush x:Key="BrandBrush" Color="#3B82F6"/>
  </ResourceDictionary>
</Application.Resources>
```

버튼 클래스 기반 스타일:

```xml
<!-- Resources/Styles.axaml -->
<ResourceDictionary xmlns="https://github.com/avaloniaui"
                    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">
  <Style Selector="Button.primary">
    <Setter Property="Background" Value="{StaticResource BrandBrush}"/>
    <Setter Property="Foreground" Value="White"/>
  </Style>
</ResourceDictionary>
```

사용:

```xml
<Button Classes="primary" Content="저장"/>
```

---

## 로깅/설정/구성

- **로깅**: `AppBuilder.LogToTrace()`로 기본 로그. 필요하면 Serilog/NLog 사용.
- **설정**: JSON 기반 설정 파일을 서비스로 로딩해 ViewModel에 주입.
- **환경 분리**: 개발/운영에 따라 로그 레벨/엔드포인트 분기.

---

## 성능/스레딩/반응성 팁

- 과도한 중첩 레이아웃/효과는 렌더링 비용을 증가시킨다.
- 빈번한 `PropertyChanged` 폭주를 막기 위해 디바운스/배치 갱신 고려.
- 비동기 I/O는 `async/await`로 UI 스레드 블로킹 방지.
- 이미지/대용량 리소스는 지연 로딩.
- DataGrid/리스트는 가능한 **가상화** 옵션을 활용.

---

## 자주 겪는 문제와 해결

- **DataTemplate 미적용**: `xmlns:vm="using:..."`, `DataType="{x:Type vm:MyVm}"` 네임스페이스/타입명 오타 점검.
- **바인딩 실패**: 출력 로그 확인, 프로퍼티명/INPC 구현/경로 점검.
- **명령 비활성**: CanExecute 조건·초기 값·구독 순서 확인.
- **ViewModel에서 View 참조**: 다이얼로그/네비게이션은 서비스로 추상화해 테스트 가능하게 유지.

---

## 완성 예제: 홈/설정 전환 + 서비스/DI(요약)

### ViewModels

```csharp
// ViewModels/HomeViewModel.cs (ReactiveUI)
using ReactiveUI;

public class HomeViewModel : ViewModelBase
{
    private string _message = "홈 화면입니다.";
    public string Message
    {
        get => _message;
        set => this.RaiseAndSetIfChanged(ref _message, value);
    }
}
```

```csharp
// ViewModels/SettingsViewModel.cs (ReactiveUI)
using ReactiveUI;

public class SettingsViewModel : ViewModelBase
{
    private string _title = "설정";
    public string Title
    {
        get => _title;
        set => this.RaiseAndSetIfChanged(ref _title, value);
    }
}
```

```csharp
// ViewModels/MainWindowViewModel.cs (ReactiveUI)
using ReactiveUI;
using System.Reactive;

public class MainWindowViewModel : ViewModelBase
{
    private ViewModelBase _currentViewModel = new HomeViewModel();
    public ViewModelBase CurrentViewModel
    {
        get => _currentViewModel;
        set => this.RaiseAndSetIfChanged(ref _currentViewModel, value);
    }

    public ReactiveCommand<Unit, Unit> NavigateHomeCommand { get; }
    public ReactiveCommand<Unit, Unit> NavigateSettingsCommand { get; }

    public MainWindowViewModel()
    {
        NavigateHomeCommand = ReactiveCommand.Create(() => CurrentViewModel = new HomeViewModel());
        NavigateSettingsCommand = ReactiveCommand.Create(() => CurrentViewModel = new SettingsViewModel());
    }
}
```

### Views

```xml
<!-- Views/HomeView.axaml -->
<UserControl xmlns="https://github.com/avaloniaui"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             x:Class="MyApp.Views.HomeView">
  <StackPanel Margin="12">
    <TextBlock Text="{Binding Message}" FontSize="18"/>
  </StackPanel>
</UserControl>
```

```xml
<!-- Views/SettingsView.axaml -->
<UserControl xmlns="https://github.com/avaloniaui"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             x:Class="MyApp.Views.SettingsView">
  <StackPanel Margin="12">
    <TextBlock Text="{Binding Title}" FontSize="18"/>
  </StackPanel>
</UserControl>
```

```xml
<!-- Views/MainWindow.axaml -->
<Window xmlns="https://github.com/avaloniaui"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        x:Class="MyApp.Views.MainWindow"
        Width="960" Height="640" Title="MyApp">
  <DockPanel>
    <StackPanel DockPanel.Dock="Top" Orientation="Horizontal" Spacing="8" Margin="8">
      <Button Content="홈" Command="{Binding NavigateHomeCommand}"/>
      <Button Content="설정" Command="{Binding NavigateSettingsCommand}"/>
    </StackPanel>
    <ContentControl Content="{Binding CurrentViewModel}" Margin="12"/>
  </DockPanel>
</Window>
```

### App.axaml — DataTemplate 매핑

```xml
<Application
  xmlns="https://github.com/avaloniaui"
  xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
  xmlns:vm="using:MyApp.ViewModels"
  xmlns:views="using:MyApp.Views"
  x:Class="MyApp.App">

  <Application.DataTemplates>
    <DataTemplate DataType="{x:Type vm:HomeViewModel}">
      <views:HomeView/>
    </DataTemplate>
    <DataTemplate DataType="{x:Type vm:SettingsViewModel}">
      <views:SettingsView/>
    </DataTemplate>
  </Application.DataTemplates>

  <Application.Styles>
    <FluentTheme Mode="Light"/>
  </Application.Styles>
</Application>
```

### App.axaml.cs — DI/부트스트랩

```csharp
using Avalonia;
using Avalonia.Controls.ApplicationLifetimes;
using Microsoft.Extensions.DependencyInjection;
using MyApp.ViewModels;

public partial class App : Application
{
    public static ServiceProvider Services { get; private set; } = default!;

    public override void OnFrameworkInitializationCompleted()
    {
        var sc = new ServiceCollection();
        sc.AddSingleton<MainWindowViewModel>();
        sc.AddTransient<HomeViewModel>();
        sc.AddTransient<SettingsViewModel>();

        Services = sc.BuildServiceProvider();

        if (ApplicationLifetime is IClassicDesktopStyleApplicationLifetime desktop)
        {
            var shell = Services.GetRequiredService<MainWindowViewModel>();
            desktop.MainWindow = new Views.MainWindow { DataContext = shell };
        }

        base.OnFrameworkInitializationCompleted();
    }
}
```

---

## 빌드/디버그/핫 리로드/배포

- 개발 실행: `dotnet run`, 핫 리로드: `dotnet watch`
- 배포(self-contained):

```bash
dotnet publish -c Release -r win-x64 --self-contained true
dotnet publish -c Release -r linux-x64 --self-contained true
dotnet publish -c Release -r osx-x64 --self-contained true
```

단일 파일은 `-p:PublishSingleFile=true` 추가(플랫폼별 서명/권한 고려).

---

## 요약 체크리스트

- View는 **바인딩과 스타일**만, 로직은 ViewModel/서비스로 이동
- ViewModel은 **INPC**와 **ICommand**로 상태/동작 노출
- **DataTemplate**로 ViewModel↔View 자동 매핑
- **DI**로 서비스/네비게이션/다이얼로그 추상화
- 비동기/취소/예외를 **명시적**으로 관리
- **단위 테스트**로 ViewModel/서비스를 검증
- 리소스/스타일/테마는 **전역 사전**으로 관리
- 성능/가상화/반응성 고려(대량 데이터/이미지/동시성)

---

## 결론

초안의 핵심(정의, 구성 요소, 흐름, 장점, 간단 예제)에 **명령/바인딩/검증/네비게이션/다이얼로그/DI/비동기/테스트/성능/운영**을 총망라해 **Avalonia 실전 MVVM**을 완성했다.
이 패턴을 습관화하면 UI 변경과 로직 확장이 독립적으로 가능해지고, 테스트·배포·운영에서의 비용이 크게 줄어든다. 다음 단계로는 **모듈화된 기능(예: 검색/필터/페이지네이션)**을 같은 패턴으로 확장해보자.
