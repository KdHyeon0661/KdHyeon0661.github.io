---
layout: post
title: Avalonia - Avalonia 구조
date: 2025-01-03 19:20:23 +0900
category: Avalonia
---
# Avalonia 구조와 확장 전략

Avalonia는 .NET용 크로스 플랫폼 UI 프레임워크로, 데스크톱(Windows, Linux, macOS)부터 모바일, 웹 어셈블리까지 지원합니다. 이 글은 공식 템플릿(`dotnet new avalonia.app`)으로 생성한 프로젝트를 시작점으로 삼아, 초중급 수준에서 Avalonia 앱의 구조를 이해하고 실제 프로젝트에서 활용할 수 있는 확장 방법을 설명합니다.

---

## 템플릿 생성

```bash
dotnet new avalonia.app -o MyAvaloniaApp
cd MyAvaloniaApp
```

생성된 프로젝트의 기본 구조는 다음과 같습니다.

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
│   └── MainWindow.axaml
├── MyAvaloniaApp.csproj
└── ...
```

이 구조는 MVVM 패턴을 따르며, 간단한 바인딩 예제를 포함하고 있습니다. 앞으로 이 구조를 점진적으로 확장해 나갑니다.

---

## Program.cs – 부트스트랩과 라이프타임

`Program.cs`는 앱의 진입점입니다. 기본 코드는 다음과 같습니다.

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

`UsePlatformDetect()`는 현재 실행 중인 운영체제에 맞는 백엔드(윈도우, X11, macOS 등)를 자동으로 설정합니다. `StartWithClassicDesktopLifetime`은 마지막 창이 닫힐 때 애플리케이션을 종료하는 전형적인 데스크톱 수명 주기를 제공합니다.

### 환경 분기

개발 환경과 운영 환경에서 로깅 레벨을 다르게 설정하거나, 디버그용 코드를 포함시킬 수 있습니다.

```csharp
public static AppBuilder BuildAvaloniaApp()
{
    var builder = AppBuilder.Configure<App>().UsePlatformDetect();

#if DEBUG
    builder.LogToTrace();  // 개발 중에는 콘솔에 로그 출력
#endif

    return builder;
}
```

> **팁**: `StartWithClassicDesktopLifetime` 대신 `SingleViewApplicationLifetime`을 사용하면 모바일이나 싱글뷰 앱에 적합한 수명 주기를 적용할 수 있습니다.

---

## App.axaml – 전역 리소스와 데이터 템플릿

`App.axaml`은 애플리케이션 전체에 적용될 리소스(색상, 브러시, 스타일)와 데이터 템플릿을 정의합니다.

### 기본 구조

```xml
<Application xmlns="https://github.com/avaloniaui"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             x:Class="MyAvaloniaApp.App">
    <Application.Styles>
        <FluentTheme Mode="Light"/>
    </Application.Styles>
</Application>
```

### 리소스 병합

규모가 커지면 리소스를 별도 파일로 분리하는 것이 좋습니다.

```xml
<Application.Resources>
    <ResourceDictionary>
        <ResourceDictionary.MergedDictionaries>
            <ResourceInclude Source="avares://MyAvaloniaApp/Resources/Colors.axaml" />
            <ResourceInclude Source="avares://MyAvaloniaApp/Resources/Styles.axaml" />
        </ResourceDictionary.MergedDictionaries>
        <SolidColorBrush x:Key="PrimaryBrush" Color="#3B82F6"/>
    </ResourceDictionary>
</Application.Resources>
```

### 데이터 템플릿과 네임스페이스

`Application.DataTemplates`는 ViewModel 타입과 View를 연결하는 데 사용됩니다. 네임스페이스 선언이 정확해야 합니다.

```xml
<Application xmlns="https://github.com/avaloniaui"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:vm="using:MyAvaloniaApp.ViewModels"
             xmlns:views="using:MyAvaloniaApp.Views"
             x:Class="MyAvaloniaApp.App">
    <Application.DataTemplates>
        <DataTemplate DataType="{x:Type vm:MainWindowViewModel}">
            <views:MainWindowView/>
        </DataTemplate>
    </Application.DataTemplates>
</Application>
```

### 시작 윈도우 설정 (`App.axaml.cs`)

`OnFrameworkInitializationCompleted`에서 시작 윈도우를 지정합니다.

```csharp
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
```

---

## MainWindow – 셸(Shell)로서의 역할

초기 템플릿의 `MainWindow`는 단순히 `TextBlock` 하나를 포함하고 있습니다. 하지만 실제 애플리케이션에서는 `MainWindow`를 전체 화면을 관리하는 **셸**로 사용합니다.

### 셸 구조

```xml
<Window xmlns="https://github.com/avaloniaui"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        x:Class="MyAvaloniaApp.MainWindow"
        Width="960" Height="640" Title="My App">
    <DockPanel>
        <StackPanel DockPanel.Dock="Top" Orientation="Horizontal" Margin="8">
            <Button Command="{Binding NavigateHomeCommand}" Content="홈"/>
            <Button Command="{Binding NavigateSettingsCommand}" Content="설정"/>
        </StackPanel>
        <ContentControl Content="{Binding CurrentViewModel}" Margin="12"/>
    </DockPanel>
</Window>
```

`ContentControl`의 `Content`는 현재 활성화된 ViewModel에 바인딩됩니다. `DataTemplate`이 ViewModel 타입을 적절한 View로 변환해 줍니다.

---

## ViewModelBase와 바인딩 기초

템플릿에 따라 `ViewModelBase`는 `ReactiveObject`(ReactiveUI) 또는 `ObservableObject`(CommunityToolkit.Mvvm)를 상속합니다. 두 방식 모두 `INotifyPropertyChanged`를 구현하며, 바인딩 가능한 속성을 쉽게 만들 수 있습니다.

### ReactiveUI 예제

```csharp
using ReactiveUI;

public class ViewModelBase : ReactiveObject
{
}

public class MainViewModel : ViewModelBase
{
    private string _greeting = "Welcome!";
    public string Greeting
    {
        get => _greeting;
        set => this.RaiseAndSetIfChanged(ref _greeting, value);
    }

    private bool _isLoading;
    public bool IsLoading
    {
        get => _isLoading;
        set => this.RaiseAndSetIfChanged(ref _isLoading, value);
    }
}
```

### CommunityToolkit.Mvvm 예제

```csharp
using CommunityToolkit.Mvvm.ComponentModel;

public partial class ViewModelBase : ObservableObject
{
}

public partial class MainViewModel : ViewModelBase
{
    [ObservableProperty]
    private string greeting = "Welcome!";

    [ObservableProperty]
    private bool isLoading;
}
```

`[ObservableProperty]` 소스 생성기가 `Greeting` 속성을 자동으로 생성하고 `OnGreetingChanged` 부분 메서드도 만들어 줍니다.

---

## 화면 전환 – ContentControl + DataTemplate

여러 화면을 전환하려면 각 화면에 대한 View/ViewModel 쌍을 만들고, `App.axaml`에 `DataTemplate`을 등록한 후, 셸 ViewModel에서 현재 ViewModel을 교체합니다.

### 1. View와 ViewModel 생성

`Views/HomeView.axaml`

```xml
<UserControl xmlns="https://github.com/avaloniaui"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             x:Class="MyAvaloniaApp.Views.HomeView">
    <StackPanel>
        <TextBlock Text="{Binding WelcomeText}" FontSize="20"/>
    </StackPanel>
</UserControl>
```

`ViewModels/HomeViewModel.cs`

```csharp
public class HomeViewModel : ViewModelBase
{
    private string _welcomeText = "홈 화면입니다.";
    public string WelcomeText
    {
        get => _welcomeText;
        set => this.RaiseAndSetIfChanged(ref _welcomeText, value);
    }
}
```

같은 방식으로 `SettingsView` / `SettingsViewModel`을 만듭니다.

### 2. DataTemplate 등록

`App.axaml`에 네임스페이스와 데이터 템플릿을 추가합니다.

```xml
<Application xmlns="https://github.com/avaloniaui"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:vm="using:MyAvaloniaApp.ViewModels"
             xmlns:views="using:MyAvaloniaApp.Views"
             x:Class="MyAvaloniaApp.App">
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

### 3. 셸 ViewModel에서 전환

`MainWindowViewModel`이 현재 화면의 ViewModel을 보유하고, 전환 명령을 제공합니다.

```csharp
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
        _currentViewModel = new HomeViewModel();
        NavigateHomeCommand = ReactiveCommand.Create(() => CurrentViewModel = new HomeViewModel());
        NavigateSettingsCommand = ReactiveCommand.Create(() => CurrentViewModel = new SettingsViewModel());
    }
}
```

`MainWindow` XAML은 이미 `ContentControl`을 `CurrentViewModel`에 바인딩했으므로, 버튼을 누르면 화면이 전환됩니다.

---

## DI(의존성 주입)와 서비스 계층

애플리케이션이 커지면 ViewModel 생성, 네비게이션, 다이얼로그 등에 DI 컨테이너를 도입하는 것이 좋습니다. `Microsoft.Extensions.DependencyInjection`을 사용해 봅시다.

### 1. 패키지 추가

```bash
dotnet add package Microsoft.Extensions.DependencyInjection
```

### 2. App.axaml.cs에서 컨테이너 구성

```csharp
using Microsoft.Extensions.DependencyInjection;

public partial class App : Application
{
    public static IServiceProvider Services { get; private set; } = default!;

    public override void OnFrameworkInitializationCompleted()
    {
        var services = new ServiceCollection();

        // 서비스 등록
        services.AddSingleton<INavigationService, NavigationService>();
        services.AddSingleton<IDialogService, DialogService>();

        // ViewModel 등록 (Transient 또는 Scoped)
        services.AddTransient<HomeViewModel>();
        services.AddTransient<SettingsViewModel>();
        services.AddSingleton<MainWindowViewModel>();

        Services = services.BuildServiceProvider();

        if (ApplicationLifetime is IClassicDesktopStyleApplicationLifetime desktop)
        {
            var mainViewModel = Services.GetRequiredService<MainWindowViewModel>();
            desktop.MainWindow = new MainWindow { DataContext = mainViewModel };
        }
        base.OnFrameworkInitializationCompleted();
    }
}
```

### 3. 간단한 네비게이션 서비스

```csharp
public interface INavigationService
{
    ViewModelBase Current { get; }
    void NavigateTo<T>() where T : ViewModelBase;
}

public class NavigationService : INavigationService
{
    private readonly IServiceProvider _serviceProvider;
    public ViewModelBase Current { get; private set; }

    public NavigationService(IServiceProvider serviceProvider)
    {
        _serviceProvider = serviceProvider;
        Current = _serviceProvider.GetRequiredService<HomeViewModel>();
    }

    public void NavigateTo<T>() where T : ViewModelBase
    {
        Current = _serviceProvider.GetRequiredService<T>();
    }
}
```

`MainWindowViewModel`은 `INavigationService`를 주입받아 `Current`를 관찰하거나 직접 교체할 수 있습니다.

### 4. 다이얼로그 서비스

ViewModel에서 직접 윈도우를 생성하지 않도록 추상화합니다.

```csharp
public interface IDialogService
{
    Task<string?> ShowInputDialogAsync(string title, string prompt);
}

public class DialogService : IDialogService
{
    private readonly Window _owner;

    public DialogService(Window owner) => _owner = owner;

    public async Task<string?> ShowInputDialogAsync(string title, string prompt)
    {
        var dialog = new Window
        {
            Title = title,
            Width = 400,
            Height = 200,
            Content = new TextBox { Watermark = prompt }
        };
        return await dialog.ShowDialog<string?>(_owner);
    }
}
```

---

## 스타일과 리소스 분리

### 리소스 파일 예제

`Resources/Colors.axaml`

```xml
<ResourceDictionary xmlns="https://github.com/avaloniaui"
                    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">
    <SolidColorBrush x:Key="PrimaryColor" Color="#3498db"/>
    <SolidColorBrush x:Key="SecondaryColor" Color="#2ecc71"/>
</ResourceDictionary>
```

`Resources/Styles.axaml`

```xml
<ResourceDictionary xmlns="https://github.com/avaloniaui"
                    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">
    <Style Selector="Button.primary">
        <Setter Property="Background" Value="{StaticResource PrimaryColor}"/>
        <Setter Property="Foreground" Value="White"/>
    </Style>
</ResourceDictionary>
```

이 파일들을 `App.axaml`에서 병합합니다.

### 상태 스타일 (호버, 클릭 등)

```xml
<Style Selector="Button:pointerover">
    <Setter Property="Opacity" Value="0.8"/>
</Style>
<Style Selector="Button:pressed">
    <Setter Property="RenderTransform">
        <Setter.Value>
            <ScaleTransform ScaleX="0.97" ScaleY="0.97"/>
        </Setter.Value>
    </Setter>
</Style>
```

---

## 단위 테스트

ViewModel은 순수 C# 클래스이므로 xUnit 등으로 쉽게 테스트할 수 있습니다.

```bash
dotnet new xunit -o MyAvaloniaApp.Tests
cd MyAvaloniaApp.Tests
dotnet add reference ../MyAvaloniaApp/MyAvaloniaApp.csproj
```

```csharp
using Xunit;
using MyAvaloniaApp.ViewModels;

public class HomeViewModelTests
{
    [Fact]
    public void WelcomeText_Should_Be_NotNull()
    {
        var vm = new HomeViewModel();
        Assert.NotNull(vm.WelcomeText);
    }

    [Fact]
    public void RaiseAndSetIfChanged_Works()
    {
        var vm = new HomeViewModel();
        bool changed = false;
        vm.PropertyChanged += (s, e) => { if (e.PropertyName == nameof(HomeViewModel.WelcomeText)) changed = true; };
        vm.WelcomeText = "New text";
        Assert.True(changed);
        Assert.Equal("New text", vm.WelcomeText);
    }
}
```

---

## 배포 및 성능 팁

### 자체 포함(Self-Contained) 배포

```bash
# Windows
dotnet publish -c Release -r win-x64 --self-contained true -p:PublishSingleFile=true

# Linux
dotnet publish -c Release -r linux-x64 --self-contained true -p:PublishSingleFile=true

# macOS
dotnet publish -c Release -r osx-x64 --self-contained true -p:PublishSingleFile=true
```

### 성능 고려사항

- **가상화**: `ListBox`, `DataGrid` 등은 대량 데이터에서 가상화가 기본으로 활성화되어 있습니다.
- **이미지 로딩**: `Bitmap`은 비동기로 로드하고, 필요하면 캐시를 사용하세요.
- **바인딩**: `OneWay` 바인딩을 기본으로 하고, 양방향이 꼭 필요한 경우에만 `TwoWay`를 사용합니다.
- **컴파일 바인딩**: `x:DataType`을 사용하면 컴파일 타임에 바인딩을 검증하고 성능을 향상시킬 수 있습니다.

---

## 자주 겪는 오류와 해결

| 오류 현상 | 원인 | 해결 |
|-----------|------|------|
| DataTemplate이 적용되지 않음 | 네임스페이스(`xmlns:vm`)가 없거나 오타 | App.axaml에 올바른 네임스페이스 선언과 `DataType` 확인 |
| 바인딩 경로를 찾을 수 없음 | 프로퍼티 이름 오타, INPC 미구현 | 출력 창의 바인딩 로그 확인, 속성명과 `RaiseAndSetIfChanged` 점검 |
| 명령이 실행되지 않음 | `CanExecute` 조건이 false | `CanExecute` 관찰 가능 상태 확인, 또는 `ReactiveCommand`의 `CanExecute` 인자 확인 |
| `x:Class` 네임스페이스 불일치 | XAML의 `x:Class`와 코드 파일의 네임스페이스가 다름 | 두 파일의 네임스페이스와 클래스명을 일치시킴 |
| 리소스(`StaticResource`)를 찾을 수 없음 | 리소스 키 오타 또는 병합 순서 문제 | 리소스 사전 병합 순서 확인, 키 철자 점검 |

---

## 요약

| 구성 요소 | 주요 역할 | 확장/관리 방법 |
|-----------|-----------|----------------|
| Program.cs | 앱 부트스트랩, 플랫폼 설정 | 환경 분기, 로깅 수준 조정 |
| App.axaml | 전역 스타일, 리소스, 데이터 템플릿 | 리소스 분리, DataTemplate으로 View-ViewModel 연결 |
| MainWindow | 셸(루트 윈도우) | ContentControl + CurrentViewModel로 화면 전환 |
| ViewModelBase | 바인딩 가능 속성 기반 | ReactiveUI 또는 CommunityToolkit 선택 |
| MainWindowViewModel | 네비게이션 상태 관리 | DI로 서비스 주입, 화면 전환 명령 제공 |
| Services 계층 | 네비게이션, 다이얼로그 등 | DI 컨테이너로 관리, 인터페이스 분리 |
| Tests | ViewModel 단위 테스트 | xUnit, NUnit 등으로 ViewModel 로직 검증 |

---

## 다음 단계

이 글에서 다룬 내용을 바탕으로 다음과 같은 고급 주제로 확장할 수 있습니다.

- 사용자 정의 컨트롤 제작 및 템플릿 활용
- 복잡한 네비게이션(스택, 모달, 탭) 구조
- 국제화(i18n)와 접근성 지원
- 성능 프로파일링 및 최적화
- CI/CD 파이프라인에 테스트와 배포 포함

Avalonia는 WPF 개발자에게 익숙한 패턴을 제공하면서도 크로스 플랫폼을 지원합니다. 이 글에서 제시한 구조를 기본으로 프로젝트를 구성하면, 유지보수성과 확장성을 갖춘 애플리케이션을 개발할 수 있습니다.