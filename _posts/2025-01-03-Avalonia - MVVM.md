---
layout: post
title: Avalonia - MVVM
date: 2025-01-03 20:20:23 +0900
category: Avalonia
---
# Avalonia와 MVVM 패턴

## MVVM이 필요한 이유

전통적인 UI 개발 방식에서는 코드 비하인드(Code-Behind) 파일에 UI 이벤트 처리와 비즈니스 로직이 함께 위치했다. 이 방식은 작은 규모에서는 편리하지만 애플리케이션이 커지면서 다음과 같은 문제가 발생한다.

- UI를 수정하면 로직에 영향을 주고, 로직을 수정하면 UI가 깨지는 **강한 결합**
- 자동화된 테스트가 어려움 (UI 컨트롤에 의존하는 코드는 테스트하기 까다로움)
- 같은 로직을 다른 UI에서 재사용하기 어려움

**MVVM(Model-View-ViewModel)** 패턴은 이러한 문제를 해결하기 위해 등장했다. UI와 비즈니스 로직 사이에 **ViewModel**이라는 계층을 두어 관심사를 분리한다.

## 세 구성 요소의 역할

| 구성 요소 | 역할 | 구현 시 주의점 |
|-----------|------|----------------|
| Model | 데이터 구조, 비즈니스 규칙, 데이터 접근 로직 | UI 프레임워크에 대한 의존성을 가지지 않아야 함 |
| View | 사용자 인터페이스 (XAML) | 로직을 최소화하고 바인딩으로 데이터를 표시 |
| ViewModel | View의 상태와 명령을 관리, Model 데이터를 가공 | INotifyPropertyChanged와 ICommand 구현 |

### 데이터 흐름

```
사용자 입력 → View → ViewModel → Model
                ↑          ↑
            바인딩     데이터 갱신
```

View는 ViewModel의 속성과 명령에 바인딩한다. 사용자가 버튼을 클릭하면 ViewModel의 명령이 실행되고, ViewModel은 필요에 따라 Model을 갱신한다. Model이 변경되면 ViewModel이 이를 감지해 View에 반영한다.

## Avalonia 프로젝트 구조

MVVM 패턴을 적용한 일반적인 프로젝트 구조는 다음과 같다.

```
MyApp/
├── Models/           # 데이터 클래스
├── ViewModels/       # ViewModel 클래스
├── Views/            # XAML 파일
├── Services/         # 외부 의존성 (API, 파일 등)
├── App.axaml         # 애플리케이션 리소스
└── Program.cs
```

## ViewModel 구현하기

ViewModel의 핵심은 **INotifyPropertyChanged** 인터페이스다. 이 인터페이스를 구현하면 속성값이 변경될 때 View에 자동으로 알릴 수 있다.

Avalonia에서는 ReactiveUI 또는 CommunityToolkit.Mvvm을 주로 사용한다. 두 접근 방식 모두 살펴보자.

### ReactiveUI 방식

```csharp
// ViewModelBase.cs
using ReactiveUI;

public class ViewModelBase : ReactiveObject
{
}
```

```csharp
// MainWindowViewModel.cs
using ReactiveUI;
using System.Reactive;

public class MainWindowViewModel : ViewModelBase
{
    private string _userName = "";
    public string UserName
    {
        get => _userName;
        set => this.RaiseAndSetIfChanged(ref _userName, value);
    }

    public ReactiveCommand<Unit, Unit> GreetCommand { get; }

    public MainWindowViewModel()
    {
        GreetCommand = ReactiveCommand.Create(() =>
        {
            UserName = $"안녕하세요, {UserName}님!";
        });
    }
}
```

### CommunityToolkit.Mvvm 방식

```csharp
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;

public partial class MainWindowViewModel : ObservableObject
{
    [ObservableProperty]
    private string _userName = "";

    [RelayCommand]
    private void Greet()
    {
        UserName = $"안녕하세요, {UserName}님!";
    }
}
```

소스 생성기를 활용하는 CommunityToolkit 방식이 코드가 더 간결하다. `[ObservableProperty]`가 INotifyPropertyChanged 구현을 자동으로 생성해주고, `[RelayCommand]`가 ICommand 구현을 생성해준다.

## View에서 바인딩

XAML에서 ViewModel의 속성과 명령을 바인딩하는 방식은 다음과 같다.

```xml
<Window xmlns="https://github.com/avaloniaui"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:vm="using:MyApp.ViewModels"
        x:Class="MyApp.Views.MainWindow"
        Title="MVVM 예제"
        Width="400" Height="200">
    
    <Window.DataContext>
        <vm:MainWindowViewModel/>
    </Window.DataContext>

    <StackPanel HorizontalAlignment="Center" VerticalAlignment="Center" Spacing="8">
        <TextBox Text="{Binding UserName, Mode=TwoWay}" 
                 Watermark="이름을 입력하세요" Width="200"/>
        <Button Content="인사하기" Command="{Binding GreetCommand}" />
    </StackPanel>
</Window>
```

`Mode=TwoWay`는 텍스트 상자의 입력이 ViewModel로 전달되고, ViewModel의 변경이 텍스트 상자에 반영됨을 의미한다. 버튼의 `Command`는 클릭 시 ViewModel의 명령을 실행한다.

## 바인딩 모드와 변환기

### 바인딩 모드 종류

| 모드 | 설명 | 사용 예 |
|------|------|---------|
| OneWay | ViewModel → View 단방향 | 텍스트 블록, 이미지 |
| TwoWay | 양방향 동기화 | 텍스트 상자, 체크박스 |
| OneTime | 초기값만 전달 | 정적인 레이블 |
| OneWayToSource | View → ViewModel 단방향 | 비밀번호 입력 등 |

### 값 변환기 (IValueConverter)

ViewModel의 데이터를 View에 표시할 때 변환이 필요한 경우가 있다. 예를 들어 불리언 값을 텍스트로 변환하는 경우다.

```csharp
using System;
using Avalonia.Data.Converters;
using System.Globalization;

public class BoolToTextConverter : IValueConverter
{
    public object? Convert(object? value, Type targetType, object? parameter, CultureInfo culture)
    {
        return (value is bool b && b) ? "활성화됨" : "비활성화됨";
    }

    public object? ConvertBack(object? value, Type targetType, object? parameter, CultureInfo culture)
    {
        return (value as string) == "활성화됨";
    }
}
```

XAML에서 사용할 때는 리소스로 등록한 후 바인딩에 지정한다.

```xml
<Window.Resources>
    <local:BoolToTextConverter x:Key="BoolToText"/>
</Window.Resources>

<TextBlock Text="{Binding IsActive, Converter={StaticResource BoolToText}}"/>
```

## ViewModel 간 탐색

단일 창에서 여러 화면을 전환하는 경우, 셸 ViewModel이 현재 표시할 ViewModel을 관리하는 방식이 일반적이다.

```csharp
public class MainWindowViewModel : ViewModelBase
{
    private ViewModelBase _currentViewModel;
    public ViewModelBase CurrentViewModel
    {
        get => _currentViewModel;
        set => this.RaiseAndSetIfChanged(ref _currentViewModel, value);
    }

    public ReactiveCommand<Unit, Unit> GoToHomeCommand { get; }
    public ReactiveCommand<Unit, Unit> GoToSettingsCommand { get; }

    public MainWindowViewModel()
    {
        CurrentViewModel = new HomeViewModel();
        
        GoToHomeCommand = ReactiveCommand.Create(() => 
            CurrentViewModel = new HomeViewModel());
        GoToSettingsCommand = ReactiveCommand.Create(() => 
            CurrentViewModel = new SettingsViewModel());
    }
}
```

XAML에서는 ContentControl을 사용해 CurrentViewModel이 바뀌면 해당 ViewModel에 맞는 View가 자동으로 표시되도록 한다.

```xml
<DockPanel>
    <StackPanel DockPanel.Dock="Top" Orientation="Horizontal" Spacing="8">
        <Button Command="{Binding GoToHomeCommand}" Content="홈"/>
        <Button Command="{Binding GoToSettingsCommand}" Content="설정"/>
    </StackPanel>
    <ContentControl Content="{Binding CurrentViewModel}"/>
</DockPanel>
```

View와 ViewModel을 연결하려면 App.axaml에 DataTemplate을 정의해야 한다.

```xml
<Application.DataTemplates>
    <DataTemplate DataType="{x:Type vm:HomeViewModel}">
        <views:HomeView/>
    </DataTemplate>
    <DataTemplate DataType="{x:Type vm:SettingsViewModel}">
        <views:SettingsView/>
    </DataTemplate>
</Application.DataTemplates>
```

## 의존성 주입으로 ViewModel 구성

실제 애플리케이션에서는 ViewModel이 서비스(데이터 접근, 파일, API 등)에 의존하는 경우가 많다. 의존성 주입(DI)을 사용하면 ViewModel 생성과 의존성 관리를 체계적으로 할 수 있다.

```csharp
// App.axaml.cs
using Avalonia;
using Avalonia.Controls.ApplicationLifetimes;
using Microsoft.Extensions.DependencyInjection;

public partial class App : Application
{
    public static ServiceProvider Services { get; private set; } = null!;

    public override void OnFrameworkInitializationCompleted()
    {
        var services = new ServiceCollection();
        
        // 서비스 등록
        services.AddSingleton<IPeopleService, PeopleService>();
        
        // ViewModel 등록
        services.AddSingleton<MainWindowViewModel>();
        services.AddTransient<HomeViewModel>();
        services.AddTransient<SettingsViewModel>();
        
        Services = services.BuildServiceProvider();

        if (ApplicationLifetime is IClassicDesktopStyleApplicationLifetime desktop)
        {
            var mainViewModel = Services.GetRequiredService<MainWindowViewModel>();
            desktop.MainWindow = new Views.MainWindow 
            { 
                DataContext = mainViewModel 
            };
        }

        base.OnFrameworkInitializationCompleted();
    }
}
```

## 비동기 작업 처리

네트워크 호출이나 파일 I/O 같은 비동기 작업은 UI 스레드를 차단하지 않도록 처리해야 한다.

### ReactiveUI에서 비동기 명령

```csharp
public class DataViewModel : ViewModelBase
{
    private string _status = "대기 중";
    public string Status
    {
        get => _status;
        set => this.RaiseAndSetIfChanged(ref _status, value);
    }

    public ReactiveCommand<Unit, Unit> LoadDataCommand { get; }

    public DataViewModel()
    {
        LoadDataCommand = ReactiveCommand.CreateFromTask(async () =>
        {
            Status = "로딩 중...";
            await Task.Delay(2000); // 실제 API 호출로 대체
            Status = "로딩 완료!";
        });
    }
}
```

### CommunityToolkit에서 비동기 명령

```csharp
public partial class DataViewModel : ObservableObject
{
    [ObservableProperty]
    private string _status = "대기 중";

    [RelayCommand]
    private async Task LoadDataAsync()
    {
        Status = "로딩 중...";
        await Task.Delay(2000);
        Status = "로딩 완료!";
    }
}
```

XAML에서는 바인딩이 동일하다.

```xml
<Button Content="데이터 로드" Command="{Binding LoadDataCommand}"/>
<TextBlock Text="{Binding Status}"/>
```

## 컬렉션 바인딩

리스트나 그리드에 데이터를 표시할 때는 ObservableCollection을 사용한다.

```csharp
using System.Collections.ObjectModel;

public class ListViewModel : ViewModelBase
{
    public ObservableCollection<string> Items { get; } = new();

    public ReactiveCommand<Unit, Unit> AddCommand { get; }

    public ListViewModel()
    {
        AddCommand = ReactiveCommand.Create(() =>
        {
            Items.Add($"항목 {Items.Count + 1}");
        });
    }
}
```

XAML에서 ItemsControl이나 ListBox를 사용해 바인딩한다.

```xml
<ListBox Items="{Binding Items}"/>
<Button Content="추가" Command="{Binding AddCommand}"/>
```

## 입력 검증

사용자 입력을 검증할 때는 INotifyDataErrorInfo 인터페이스를 구현한다. CommunityToolkit.Mvvm은 ObservableValidator 클래스를 제공해 이 작업을 간소화한다.

```csharp
using CommunityToolkit.Mvvm.ComponentModel;
using System.ComponentModel.DataAnnotations;

public partial class LoginViewModel : ObservableValidator
{
    [ObservableProperty]
    [Required(ErrorMessage = "이메일은 필수입니다.")]
    [EmailAddress(ErrorMessage = "올바른 이메일 형식이 아닙니다.")]
    private string? _email;

    [ObservableProperty]
    [Required(ErrorMessage = "비밀번호는 필수입니다.")]
    [MinLength(6, ErrorMessage = "비밀번호는 6자 이상이어야 합니다.")]
    private string? _password;

    partial void OnEmailChanged(string? value)
    {
        ValidateProperty(value, nameof(Email));
    }

    partial void OnPasswordChanged(string? value)
    {
        ValidateProperty(value, nameof(Password));
    }

    public bool HasErrors => GetErrors().Any();
}
```

XAML에서 검증 오류를 표시하려면 TextBox에 Validation.ErrorTemplate을 적용할 수 있다.

## 단위 테스트

MVVM의 큰 장점은 ViewModel이 View에 의존하지 않아 단위 테스트가 가능하다는 점이다.

```csharp
using Xunit;

public class MainWindowViewModelTests
{
    [Fact]
    public void GreetCommand_UpdatesUserName()
    {
        // Arrange
        var viewModel = new MainWindowViewModel();
        viewModel.UserName = "홍길동";

        // Act
        viewModel.GreetCommand.Execute();

        // Assert
        Assert.Contains("홍길동", viewModel.UserName);
    }
}
```

## 디자인 타임 데이터

XAML 디자이너에서 미리보기용 데이터를 표시하려면 Design.DataContext를 사용한다.

```xml
<UserControl xmlns="https://github.com/avaloniaui"
             xmlns:d="https://github.com/avaloniaui/2016/xaml"
             xmlns:vm="using:MyApp.ViewModels"
             d:DataContext="{x:Static vm:DesignTimeViewModel.Instance}">
    
    <TextBlock Text="{Binding Title}"/>
</UserControl>
```

## 성능 고려사항

- **속성 변경 알림 최소화**: 불필요한 PropertyChanged 이벤트 발생을 줄인다
- **컬렉션 갱신**: 대량의 항목을 추가할 때는 AddRange 같은 배치 작업을 사용한다
- **가상화**: 대량의 데이터를 표시할 때는 VirtualizingStackPanel을 활용한다
- **비동기 처리**: UI 스레드를 블로킹하지 않도록 async/await를 사용한다

## 자주 발생하는 문제와 해결

| 문제 | 해결 방법 |
|------|-----------|
| 바인딩이 동작하지 않음 | 출력 창에서 바인딩 오류 확인, INotifyPropertyChanged 구현 확인 |
| 명령 버튼이 비활성화됨 | CanExecute 조건과 RaiseCanExecuteChanged 호출 확인 |
| DataTemplate이 적용되지 않음 | DataType 네임스페이스와 타입명 정확히 지정 |
| ViewModel 변경 시 View 갱신 안됨 | CurrentViewModel 속성에서 RaisePropertyChanged 호출 확인 |

## 정리하며

MVVM 패턴은 Avalonia 애플리케이션 개발에서 사실상 표준으로 자리잡고 있다. 이 패턴을 적용함으로써 얻을 수 있는 이점을 정리하면 다음과 같다.

- UI와 비즈니스 로직이 분리되어 **유지보수성**이 향상된다
- ViewModel만 독립적으로 테스트할 수 있어 **안정성**이 높아진다
- 같은 ViewModel을 다른 View에서 재사용할 수 있어 **생산성**이 올라간다
- 의존성 주입과 결합하면 **확장성** 있는 구조를 만들 수 있다

초기 학습 곡선이 다소 가파를 수 있지만, 일단 구조에 익숙해지면 규모 있는 애플리케이션을 체계적으로 개발할 수 있다. 작은 예제부터 시작해 점차 복잡도를 높여가며 MVVM 패턴에 익숙해지길 권장한다.