---
layout: post
title: Avalonia - Avalonia 구조
date: 2025-01-03 19:20:23 +0900
category: Avalonia
---
# Avalonia 기본 템플릿 구조 분석 및 확장 방법

- 템플릿별 차이(reactive/mvvm 옵션)와 `ViewModelBase`의 정체
- `Program.cs`의 부트스트랩 옵션, Lifetime 구분, 환경 분기
- `App.axaml`의 전역 리소스/스타일/머지 전략과 DataTemplate 네임스페이스 설정
- `MainWindow`를 루트 셸로 삼아 **ContentControl + DataTemplate**로 화면 전환
- **ReactiveUI** 기준 예제(`RaiseAndSetIfChanged`, `ReactiveCommand`)와 **CommunityToolkit.Mvvm** 대안
- DI(의존성 주입), 네비게이션 서비스, 다이얼로그 서비스 설계
- 단위 테스트 구도(ViewModel 중심), 폴더 구조 리팩터링, 배포/성능 팁
- 자주 겪는 오류와 해결(네임스페이스, DataTemplate, 바인딩 실패 등)

코드는 모두 ```로 감싸고, 수학이 있으면 반드시 $$...$$로 감싼다(본 글은 수식을 사용하지 않는다).

---

## 1. 템플릿 생성

```bash
dotnet new avalonia.app -o MyAvaloniaApp
cd MyAvaloniaApp
```

- `avalonia.app` 템플릿은 **데스크톱(Windows/Linux/macOS)** 대상의 최소 앱 골격을 만든다.
- 템플릿에는 옵션이 존재할 수 있다. 예: `--mvvm`(CommunityToolkit 기반) 또는 `--reactive`(ReactiveUI 기반) 등.  
  설치된 템플릿 버전에 따라 스캐폴딩 결과가 다를 수 있으므로 `dotnet new --list`로 확인한다.

---

## 2. 기본 프로젝트 구조

초안에서 제시된 구조는 다음과 유사하다(템플릿/옵션에 따라 달라질 수 있음):

```
MyAvaloniaApp/
├── App.axaml
├── App.axaml.cs
├── MainWindow.axaml
├── MainWindow.axaml.cs
├── Program.cs
├── ViewModels/
│   └── MainWindowViewModel.cs
├── Views/
│   └── MainWindow.axaml          (템플릿에 따라 View 분리될 수 있음)
├── MyAvaloniaApp.csproj
├── obj/
└── bin/
```

핵심 포인트(확장 설명 포함):
- **Program.cs**: 앱 진입점. `AppBuilder`를 구성하고 플랫폼별 초기화를 수행한다.
- **App.axaml / App.axaml.cs**: 전역 리소스/스타일/테마/시작 윈도우 설정. DataTemplate, 리소스 병합 포인트.
- **MainWindow**: 시작 화면이자 셸(루트 컨테이너). `ContentControl`로 내부 화면 교체 추천.
- **ViewModels**: MVVM의 중심. 상태/명령/네비게이션/검증/서비스 주입의 핵심 레이어.
- **Views**: XAML 기반 화면. 로직은 최소화하고 바인딩에 집중.

---

## 3. Program.cs — 부트스트랩, Lifetime, 환경 분기

초안의 코드:

```csharp
public static class Program
{
    public static void Main(string[] args) =>
        BuildAvaloniaApp().StartWithClassicDesktopLifetime(args);

    public static AppBuilder BuildAvaloniaApp()
        => AppBuilder.Configure<App>()
                     .UsePlatformDetect()
                     .LogToTrace();
}
```

**확장 포인트**:

- `UsePlatformDetect()`  
  OS/디스플레이/입력 백엔드를 자동 세팅한다. 데스크톱 타깃에서 일반적으로 사용.
- `StartWithClassicDesktopLifetime(args)`  
  창을 닫으면 프로세스를 종료하는 **클래식 데스크톱** 수명 모델.  
  모바일/싱글뷰 앱일 때는 `SingleViewApplicationLifetime`을 사용한다.
- 환경별 분기(예: 개발/운영 빌드에 따른 로깅 레벨, 다크 테마 기본값 등):

```csharp
using Avalonia;

public static class Program
{
    [STAThread]
    public static void Main(string[] args)
        => BuildAvaloniaApp().StartWithClassicDesktopLifetime(args);

    public static AppBuilder BuildAvaloniaApp()
    {
#if DEBUG
        var builder = AppBuilder.Configure<App>()
            .UsePlatformDetect()
            .LogToTrace(); // 디버그 환경 로깅 강화
#else
        var builder = AppBuilder.Configure<App>()
            .UsePlatformDetect();
#endif
        return builder;
    }
}
```

---

## 4. App.axaml / App.axaml.cs — 전역 리소스, 스타일, 시작 윈도우

초안의 XAML:

```xml
<Application xmlns="https://github.com/avaloniaui"
             ...
             x:Class="MyAvaloniaApp.App">
    <Application.Styles>
        <FluentTheme Mode="Light"/>
    </Application.Styles>
</Application>
```

**확장 포인트**:

1) 전역 리소스 병합(브러시/두께/문자열/스타일 분리)을 권장한다.

```xml
<Application
    xmlns="https://github.com/avaloniaui"
    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
    x:Class="MyAvaloniaApp.App">

  <Application.Resources>
    <ResourceDictionary>
      <ResourceDictionary.MergedDictionaries>
        <ResourceInclude Source="avares://MyAvaloniaApp/Resources/Colors.axaml" />
        <ResourceInclude Source="avares://MyAvaloniaApp/Resources/Spacing.axaml" />
        <ResourceInclude Source="avares://MyAvaloniaApp/Resources/Styles.axaml" />
      </ResourceDictionary.MergedDictionaries>
      <!-- 전역 문자열/브러시 등 -->
      <SolidColorBrush x:Key="BrandBrush" Color="#3B82F6"/>
    </ResourceDictionary>
  </Application.Resources>

  <Application.Styles>
    <FluentTheme Mode="Light"/>
  </Application.Styles>
</Application>
```

2) DataTemplate 네임스페이스 설정과 등록(아래 §7에서 상세).

3) 시작 윈도우 설정(`App.axaml.cs`):

```csharp
using Avalonia;
using Avalonia.Controls.ApplicationLifetimes;

namespace MyAvaloniaApp;

public partial class App : Application
{
    public override void OnFrameworkInitializationCompleted()
    {
        if (ApplicationLifetime is IClassicDesktopStyleApplicationLifetime desktop)
        {
            desktop.MainWindow = new MainWindow
            {
                DataContext = new ViewModels.MainWindowViewModel()
            };
        }
        base.OnFrameworkInitializationCompleted();
    }
}
```

- 대규모 앱에서는 여기서 DI 컨테이너를 구성해 ViewModel/서비스를 주입한다(§8 참조).

---

## 5. MainWindow — 셸로서의 역할과 레이아웃

초안의 단순 바인딩:

```xml
<Window ...>
    <StackPanel>
        <TextBlock Text="{Binding Greeting}" />
    </StackPanel>
</Window>
```

**확장 설계**: `MainWindow`를 **루트 셸**로 삼아 내부 콘텐츠를 `ContentControl`로 교체한다.

```xml
<!-- Views/MainWindow.axaml (또는 루트 Window) -->
<Window xmlns="https://github.com/avaloniaui"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:vm="using:MyAvaloniaApp.ViewModels"
        xmlns:views="using:MyAvaloniaApp.Views"
        x:Class="MyAvaloniaApp.MainWindow"
        Width="960" Height="640"
        Title="My Avalonia App">

  <DockPanel>
    <!-- 상단 바 -->
    <StackPanel DockPanel.Dock="Top" Orientation="Horizontal" Margin="8" Spacing="8">
      <Button Content="홈" Command="{Binding NavigateHomeCommand}"/>
      <Button Content="설정" Command="{Binding NavigateSettingsCommand}"/>
    </StackPanel>

    <!-- 본문: ViewModel에 따라 View가 DataTemplate로 연결되어 표시 -->
    <ContentControl Content="{Binding CurrentViewModel}" Margin="12"/>
  </DockPanel>
</Window>
```

---

## 6. ViewModelBase와 바인딩 기초

템플릿에 따라 `ViewModelBase`는 다음 중 하나일 수 있다:

- **ReactiveUI 기반**: `ViewModelBase : ReactiveObject`  
  - `RaiseAndSetIfChanged(ref field, value)` 제공  
  - `ReactiveCommand.Create(...)`
- **CommunityToolkit.Mvvm 기반**: `ViewModelBase : ObservableObject`  
  - `[ObservableProperty]` 소스생성기, `RelayCommand` 제공

초안에 등장한 `RaiseAndSetIfChanged`는 **ReactiveUI** 문법이다. 이하 기본 예제는 ReactiveUI를 기준으로 하고, 바로 뒤에 Toolkit 대안을 함께 제시한다.

### 6.1 ReactiveUI 기반 ViewModelBase

```csharp
// ViewModels/ViewModelBase.cs
using ReactiveUI;

namespace MyAvaloniaApp.ViewModels;

public class ViewModelBase : ReactiveObject
{
}
```

### 6.2 간단 ViewModel 예시

```csharp
// ViewModels/MainWindowViewModel.cs
using ReactiveUI;

namespace MyAvaloniaApp.ViewModels;

public class MainWindowViewModel : ViewModelBase
{
    private string _greeting = "Welcome to Avalonia!";
    public string Greeting
    {
        get => _greeting;
        set => this.RaiseAndSetIfChanged(ref _greeting, value);
    }
}
```

**CommunityToolkit.Mvvm 대안**:

```csharp
// ViewModels/ViewModelBase.cs
using CommunityToolkit.Mvvm.ComponentModel;

namespace MyAvaloniaApp.ViewModels;

public class ViewModelBase : ObservableObject
{
}

// ViewModels/MainWindowViewModel.cs
using CommunityToolkit.Mvvm.ComponentModel;

public partial class MainWindowViewModel : ViewModelBase
{
    [ObservableProperty]
    private string greeting = "Welcome to Avalonia!";
}
```

---

## 7. 화면 추가와 화면 전환 — DataTemplate + ContentControl

초안의 방향(새로운 View/VM 추가 → DataTemplate → `ContentControl`)을 **완성형 패턴**으로 제시한다.

### 7.1 폴더 정리

```
Views/
  MainView.axaml
  SettingsView.axaml
ViewModels/
  MainViewModel.cs
  SettingsViewModel.cs
```

### 7.2 View 정의

```xml
<!-- Views/MainView.axaml -->
<UserControl xmlns="https://github.com/avaloniaui"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             x:Class="MyAvaloniaApp.Views.MainView">
  <StackPanel Margin="12" Spacing="8">
    <TextBlock Text="메인 화면" FontSize="18"/>
    <TextBlock Text="{Binding Greeting}" />
  </StackPanel>
</UserControl>
```

```xml
<!-- Views/SettingsView.axaml -->
<UserControl xmlns="https://github.com/avaloniaui"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             x:Class="MyAvaloniaApp.Views.SettingsView">
  <StackPanel Margin="12" Spacing="8">
    <TextBlock Text="설정 화면" FontSize="18"/>
    <TextBox Text="{Binding Title}" Watermark="설정 제목" Width="240"/>
  </StackPanel>
</UserControl>
```

### 7.3 ViewModel 정의(reactive)

```csharp
// ViewModels/MainViewModel.cs
using ReactiveUI;

namespace MyAvaloniaApp.ViewModels;

public class MainViewModel : ViewModelBase
{
    private string _greeting = "메인 화면에 오신 것을 환영합니다.";
    public string Greeting
    {
        get => _greeting;
        set => this.RaiseAndSetIfChanged(ref _greeting, value);
    }
}
```

```csharp
// ViewModels/SettingsViewModel.cs
using ReactiveUI;

namespace MyAvaloniaApp.ViewModels;

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

### 7.4 App.axaml에 DataTemplate 등록

**중요**: 네임스페이스 매핑이 정확해야 한다.

```xml
<!-- App.axaml -->
<Application
    xmlns="https://github.com/avaloniaui"
    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
    xmlns:vm="using:MyAvaloniaApp.ViewModels"
    xmlns:views="using:MyAvaloniaApp.Views"
    x:Class="MyAvaloniaApp.App">

  <Application.DataTemplates>
    <!-- ViewModel 타입 → View로 변환 -->
    <DataTemplate DataType="{x:Type vm:MainViewModel}">
      <views:MainView/>
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

### 7.5 MainWindowViewModel에서 전환 로직

```csharp
// ViewModels/MainWindowViewModel.cs  (네비게이션 허브)
using ReactiveUI;
using System;
using System.Reactive;

namespace MyAvaloniaApp.ViewModels;

public class MainWindowViewModel : ViewModelBase
{
    private ViewModelBase _currentViewModel;
    public ViewModelBase CurrentViewModel
    {
        get => _currentViewModel;
        set => this.RaiseAndSetIfChanged(ref _currentViewModel, value);
    }

    public ReactiveCommand<Unit, Unit> NavigateHomeCommand { get; }
    public ReactiveCommand<Unit, Unit> NavigateSettingsCommand { get; }

    public MainWindowViewModel()
    {
        _currentViewModel = new MainViewModel();
        NavigateHomeCommand = ReactiveCommand.Create(() => CurrentViewModel = new MainViewModel());
        NavigateSettingsCommand = ReactiveCommand.Create(() => CurrentViewModel = new SettingsViewModel());
    }
}
```

**CommunityToolkit.Mvvm 대안**:

```csharp
// ViewModels/MainWindowViewModel.cs (Toolkit)
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;

public partial class MainWindowViewModel : ViewModelBase
{
    [ObservableProperty]
    private ViewModelBase currentViewModel = new MainViewModel();

    [RelayCommand]
    private void NavigateHome() => CurrentViewModel = new MainViewModel();

    [RelayCommand]
    private void NavigateSettings() => CurrentViewModel = new SettingsViewModel();
}
```

### 7.6 MainWindow에 버튼 + ContentControl

```xml
<!-- Views/MainWindow.axaml -->
<Window xmlns="https://github.com/avaloniaui"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        x:Class="MyAvaloniaApp.MainWindow"
        Width="960" Height="640" Title="My Avalonia App">

  <DockPanel>
    <StackPanel DockPanel.Dock="Top" Orientation="Horizontal" Margin="8" Spacing="8">
      <Button Content="홈" Command="{Binding NavigateHomeCommand}"/>
      <Button Content="설정" Command="{Binding NavigateSettingsCommand}"/>
    </StackPanel>

    <ContentControl Content="{Binding CurrentViewModel}" Margin="12"/>
  </DockPanel>
</Window>
```

---

## 8. DI(의존성 주입)와 서비스 계층

규모가 커지면 ViewModel 생성/수명 관리를 DI 컨테이너로 맡기는 것이 좋다.

### 8.1 Microsoft.Extensions.DependencyInjection 사용

```csharp
// App.axaml.cs
using Avalonia;
using Avalonia.Controls.ApplicationLifetimes;
using Microsoft.Extensions.DependencyInjection;
using MyAvaloniaApp.ViewModels;

namespace MyAvaloniaApp;

public partial class App : Application
{
    public static ServiceProvider Services { get; private set; } = default!;

    public override void OnFrameworkInitializationCompleted()
    {
        var sc = new ServiceCollection();

        // 서비스/리포지토리 등록
        sc.AddSingleton<INavigationService, NavigationService>(); // 예시
        sc.AddSingleton<MainWindowViewModel>();
        sc.AddTransient<MainViewModel>();
        sc.AddTransient<SettingsViewModel>();

        Services = sc.BuildServiceProvider();

        if (ApplicationLifetime is IClassicDesktopStyleApplicationLifetime desktop)
        {
            var shell = Services.GetRequiredService<MainWindowViewModel>();
            desktop.MainWindow = new MainWindow { DataContext = shell };
        }
        base.OnFrameworkInitializationCompleted();
    }
}
```

### 8.2 간단 네비게이션 서비스

```csharp
// Services/INavigationService.cs
using MyAvaloniaApp.ViewModels;

public interface INavigationService
{
    ViewModelBase Current { get; }
    void NavigateTo<TViewModel>() where TViewModel : ViewModelBase;
}
```

```csharp
// Services/NavigationService.cs
using System;
using Microsoft.Extensions.DependencyInjection;
using MyAvaloniaApp.ViewModels;

public class NavigationService : INavigationService
{
    private readonly IServiceProvider _provider;
    public ViewModelBase Current { get; private set; }

    public NavigationService(IServiceProvider provider)
    {
        _provider = provider;
        Current = _provider.GetRequiredService<MainViewModel>();
    }

    public void NavigateTo<TViewModel>() where TViewModel : ViewModelBase
        => Current = _provider.GetRequiredService<TViewModel>();
}
```

`MainWindowViewModel`에서 이 서비스를 사용해 `CurrentViewModel`에 반영하면, 화면 전환 로직이 단일 책임으로 정리된다.

---

## 9. 다이얼로그 서비스 패턴(권장)

ViewModel이 직접 `Window`를 생성하지 않도록 **IDialogService**를 둔다.

```csharp
// Services/IDialogService.cs
using System.Threading.Tasks;

public interface IDialogService
{
    Task<string?> ShowInputAsync(string title, string prompt);
}
```

```csharp
// Services/DialogService.cs (간단 예)
using Avalonia.Controls;
using System.Threading.Tasks;

public class DialogService : IDialogService
{
    private readonly Window _owner;

    public DialogService(Window owner) => _owner = owner;

    public async Task<string?> ShowInputAsync(string title, string prompt)
    {
        var dlg = new Window { Title = title, Width = 360, Height = 160 };
        // XAML 다이얼로그를 만들어 붙여도 되고, 간단히 구성해도 된다.
        return await dlg.ShowDialog<string?>(_owner);
    }
}
```

실전에서는 XAML 다이얼로그를 만들고, DI로 ViewModel에서 호출하도록 구성한다.

---

## 10. 스타일/리소스/테마 — 분리와 상태 스타일

### 10.1 리소스 딕셔너리 분리

```
Resources/
  Colors.axaml
  Spacing.axaml
  Styles.axaml
```

`Colors.axaml`:

```xml
<ResourceDictionary xmlns="https://github.com/avaloniaui"
                    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">
  <SolidColorBrush x:Key="BrandBrush" Color="#3B82F6"/>
</ResourceDictionary>
```

`Styles.axaml`:

```xml
<ResourceDictionary xmlns="https://github.com/avaloniaui"
                    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">
  <Style Selector="Button.confirm">
    <Setter Property="Background" Value="{StaticResource BrandBrush}"/>
    <Setter Property="Foreground" Value="White"/>
  </Style>
</ResourceDictionary>
```

사용:

```xml
<Button Classes="confirm" Content="저장"/>
```

### 10.2 상태 스타일

```xml
<Style Selector="Button:pointerover">
  <Setter Property="Opacity" Value="0.9"/>
</Style>

<Style Selector="Button:pressed">
  <Setter Property="RenderTransform">
    <Setter.Value>
      <ScaleTransform ScaleX="0.98" ScaleY="0.98"/>
    </Setter.Value>
  </Setter>
</Style>
```

---

## 11. 검증/컨버터/템플릿

### 11.1 IValueConverter

```csharp
// Converters/BoolToTextConverter.cs
using System;
using Avalonia.Data.Converters;
using System.Globalization;

public class BoolToTextConverter : IValueConverter
{
    public object? Convert(object? value, Type targetType, object? parameter, CultureInfo culture)
        => value is bool b && b ? "완료" : "진행 중";

    public object? ConvertBack(object? value, Type targetType, object? parameter, CultureInfo culture)
        => (value as string) == "완료";
}
```

XAML 등록/사용:

```xml
<Window xmlns:conv="using:MyAvaloniaApp.Converters">
  <Window.Resources>
    <conv:BoolToTextConverter x:Key="BoolToText"/>
  </Window.Resources>
  <TextBlock Text="{Binding IsDone, Converter={StaticResource BoolToText}}"/>
</Window>
```

### 11.2 검증(CommunityToolkit 예)

```csharp
using CommunityToolkit.Mvvm.ComponentModel;
using System.ComponentModel.DataAnnotations;

public partial class AccountViewModel : ObservableValidator
{
    [ObservableProperty]
    [Required(ErrorMessage = "사용자 이름은 필수입니다.")]
    private string? userName;

    partial void OnUserNameChanged(string? value) => ValidateAllProperties();
}
```

---

## 12. 리스트/그리드 — ItemsControl, ListBox, DataGrid

### 12.1 ListBox 선택 바인딩

```xml
<ListBox Items="{Binding Items}" SelectedItem="{Binding SelectedItem, Mode=TwoWay}">
  <ListBox.ItemTemplate>
    <DataTemplate>
      <StackPanel Orientation="Horizontal" Spacing="8">
        <CheckBox IsChecked="{Binding IsDone}"/>
        <TextBlock Text="{Binding Title}"/>
      </StackPanel>
    </DataTemplate>
  </ListBox.ItemTemplate>
</ListBox>
```

### 12.2 DataGrid

```bash
dotnet add package Avalonia.Controls.DataGrid
```

```xml
<DataGrid Items="{Binding Items}" AutoGenerateColumns="False">
  <DataGrid.Columns>
    <DataGridTextColumn Header="제목" Binding="{Binding Title}" />
    <DataGridCheckBoxColumn Header="완료" Binding="{Binding IsDone}" />
  </DataGrid.Columns>
</DataGrid>
```

---

## 13. 단위 테스트 — ViewModel 중심

```bash
dotnet new xunit -o MyAvaloniaApp.Tests
cd MyAvaloniaApp.Tests
dotnet add reference ../MyAvaloniaApp/MyAvaloniaApp.csproj
```

```csharp
using Xunit;
using MyAvaloniaApp.ViewModels;

public class MainViewModelTests
{
    [Fact]
    public void Greeting_DefaultValue()
    {
        var vm = new MainViewModel();
        Assert.False(string.IsNullOrWhiteSpace(vm.Greeting));
    }
}
```

---

## 14. 빌드/실행/핫 리로드/배포

### 14.1 개발

```bash
dotnet run
dotnet watch   # 핫 리로드/자동 빌드
```

### 14.2 배포(self-contained, 단일 파일 선택)

```bash
# Windows
dotnet publish -c Release -r win-x64 --self-contained true

# Linux
dotnet publish -c Release -r linux-x64 --self-contained true

# macOS
dotnet publish -c Release -r osx-x64 --self-contained true

# 단일 파일 (선택)
dotnet publish -c Release -r win-x64 --self-contained true -p:PublishSingleFile=true
```

운영체제별 서명/권한 문제가 있을 수 있으므로 배포 타깃 환경에서 실행 검증 필수.

---

## 15. 성능/구조/운영 팁

- `ContentControl + DataTemplate`는 화면 교체의 표준 패턴. View 이름 규칙과 `ViewLocator`를 도입하면 매핑 자동화 가능.
- 바인딩 에러는 반드시 로그에서 확인. 속성명/INPC 여부/네임스페이스 철자 점검.
- 대량 데이터 UI는 가상화되는 컨트롤 사용(DataGrid 등)과 배치 갱신 고려.
- 이미지/대용량 리소스는 지연 로딩, 비동기 I/O, CancellationToken 처리.
- 설정/환경 분리: 개발/스테이징/운영에 따라 로깅/서버 엔드포인트 분기.
- 접근성: 키보드 탐색, 포커스 스타일, 콘트라스트, 스크린 리더 고려.

---

## 16. 예제 묶음: Settings 화면 추가 및 전환(완결)

### 16.1 View/VM 생성

```bash
mkdir -p Views ViewModels
```

```csharp
// ViewModels/SettingsViewModel.cs (Reactive)
using ReactiveUI;

namespace MyAvaloniaApp.ViewModels;

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

```xml
<!-- Views/SettingsView.axaml -->
<UserControl xmlns="https://github.com/avaloniaui"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             x:Class="MyAvaloniaApp.Views.SettingsView">
  <StackPanel Margin="12" Spacing="8">
    <TextBlock Text="설정 화면" FontSize="18"/>
    <TextBox Text="{Binding Title}" Watermark="설정 제목" Width="240"/>
  </StackPanel>
</UserControl>
```

### 16.2 App.axaml DataTemplate

```xml
<Application.DataTemplates xmlns:vm="using:MyAvaloniaApp.ViewModels"
                           xmlns:views="using:MyAvaloniaApp.Views">
  <DataTemplate DataType="{x:Type vm:MainViewModel}">
    <views:MainView/>
  </DataTemplate>
  <DataTemplate DataType="{x:Type vm:SettingsViewModel}">
    <views:SettingsView/>
  </DataTemplate>
</Application.DataTemplates>
```

### 16.3 Shell ViewModel(전환 커맨드)

```csharp
// ViewModels/MainWindowViewModel.cs
using ReactiveUI;
using System.Reactive;

namespace MyAvaloniaApp.ViewModels;

public class MainWindowViewModel : ViewModelBase
{
    private ViewModelBase _currentViewModel = new MainViewModel();
    public ViewModelBase CurrentViewModel
    {
        get => _currentViewModel;
        set => this.RaiseAndSetIfChanged(ref _currentViewModel, value);
    }

    public ReactiveCommand<Unit, Unit> NavigateHomeCommand { get; }
    public ReactiveCommand<Unit, Unit> NavigateSettingsCommand { get; }

    public MainWindowViewModel()
    {
        NavigateHomeCommand = ReactiveCommand.Create(() => CurrentViewModel = new MainViewModel());
        NavigateSettingsCommand = ReactiveCommand.Create(() => CurrentViewModel = new SettingsViewModel());
    }
}
```

### 16.4 MainWindow에서 연결

```xml
<!-- Views/MainWindow.axaml -->
<Window xmlns="https://github.com/avaloniaui"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        x:Class="MyAvaloniaApp.MainWindow"
        Width="960" Height="640" Title="My Avalonia App">
  <DockPanel>
    <StackPanel DockPanel.Dock="Top" Orientation="Horizontal" Margin="8" Spacing="8">
      <Button Content="홈" Command="{Binding NavigateHomeCommand}"/>
      <Button Content="설정" Command="{Binding NavigateSettingsCommand}"/>
    </StackPanel>
    <ContentControl Content="{Binding CurrentViewModel}" Margin="12"/>
  </DockPanel>
</Window>
```

---

## 17. 자주 겪는 오류와 해결

- **DataTemplate가 적용되지 않음**  
  - `App.axaml`에서 `xmlns:vm="using:...ViewModels"` / `xmlns:views="using:...Views"` 정확히 지정  
  - `DataType="{x:Type vm:FooViewModel}"`에서 타입명, 네임스페이스 확인
- **바인딩 실패**  
  - 출력 로그에서 바인딩 에러 확인  
  - 프로퍼티 이름 오타, `INotifyPropertyChanged` 구현 여부 점검  
  - DataContext가 의도한 ViewModel인지 확인(디자인 타임/런타임)
- **명령이 비활성**  
  - ReactiveCommand/RelayCommand 생성 위치, `CanExecute` 조건 확인
- **네임스페이스 충돌/불일치**  
  - 프로젝트 루트 네임스페이스와 폴더 구조 간섭에 주의  
  - `x:Class`의 풀네임이 실제 cs 파일 네임스페이스와 일치해야 함

---

## 18. 요약 표

| 구성 요소 | 핵심 역할 | 확장 포인트 |
|-----------|-----------|-------------|
| `Program.cs` | 앱 부트스트랩, Lifetime | 환경 분기, 로깅, 플랫폼 설정 |
| `App.axaml` | 전역 스타일/리소스/DataTemplate | 리소스 병합, 테마, ViewModel→View 매핑 |
| `MainWindow` | 셸(루트 컨테이너) | `ContentControl`로 내부 화면 교체 |
| `ViewModelBase` | 공통 ViewModel 기능 | ReactiveUI 또는 Toolkit 선택 |
| `MainWindowViewModel` | 네비게이션 허브 | 서비스 주입, 명령, 상태 |
| `Views/*` | 화면(XAML) | 바인딩 중심, 로직 최소화 |
| Services | 네비게이션/다이얼로그/데이터 | DI로 수명/의존성 관리 |
| Tests | ViewModel 단위 테스트 | CI 포함, 회귀 방지 |

---

## 19. 다음 단계

- 사용자 정의 컨트롤/템플릿 확장(Attached Property, Behaviors)
- 복잡한 네비게이션(스택/백버튼, 모달/시트, 영역 분할)
- 설정/로깅/국제화/접근성 체계화
- 배포 파이프라인(서명, 자동 업데이트, 다중 OS)
- 성능 측정/프로파일링(대량 데이터, 이미지, 애니메이션)

---

## 결론

초안의 흐름(템플릿 생성→구조→각 파일 설명→확장)을 유지하면서, **DataTemplate 기반 화면 전환을 중심으로 한 셸 구조**, **ReactiveUI/Toolkit 양방향 ViewModel 패턴**, **DI/서비스화**, **리소스/스타일 분리**, **테스트/배포/운영 팁**까지 실전에 필요한 내용을 확장했다.  
이 구조를 기반으로 모듈을 추가하고 서비스를 주입해 나가면, 중형 이상 규모의 Avalonia 앱도 **체계적이고 장기적으로 유지보수 가능한 형태**로 성장시킬 수 있다.