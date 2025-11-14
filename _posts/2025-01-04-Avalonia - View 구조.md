---
layout: post
title: Avalonia - View 구조
date: 2025-01-04 19:20:23 +0900
category: Avalonia
---
# Avalonia의 View 구조와 확장 방법

## View란 무엇인가

**View**는 사용자가 직접 보는 UI를 의미하며 Avalonia에서는 **`.axaml` 파일(XAML)** 로 정의된다.
MVVM 구조에서 View는 **표현과 레이아웃, 스타일**에 집중하고, **상태·동작은 ViewModel**이 담당한다. View는 일반적으로 **DataContext**로 ViewModel을 참조하며, 바인딩(속성/명령)으로 상호작용한다.

---

## 기본 구조와 파일 구성

### 예시: MainWindow.axaml

```xml
<Window xmlns="https://github.com/avaloniaui"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:d="https://github.com/avaloniaui"
        xmlns:vm="clr-namespace:MyApp.ViewModels"
        xmlns:local="clr-namespace:MyApp.Views"
        x:Class="MyApp.Views.MainWindow"
        Title="Main Window"
        Width="400" Height="300">

  <Window.DataContext>
    <vm:MainWindowViewModel/>
  </Window.DataContext>

  <StackPanel Margin="20" Spacing="10">
    <TextBlock Text="Hello Avalonia!" FontSize="24" />
    <Button Content="클릭" Command="{Binding ClickCommand}"/>
  </StackPanel>
</Window>
```

### 코드 비하인드: MainWindow.axaml.cs

```csharp
using Avalonia.Controls;

namespace MyApp.Views;

public partial class MainWindow : Window
{
    public MainWindow()
    {
        InitializeComponent();
    }
}
```

핵심 포인트
- View는 **XAML**에서 선언, **코드 비하인드**는 UI 초기화나 최소 이벤트 연결 정도로 제한한다.
- **DataContext**는 XAML(디자인 타임) 또는 **조립 루트(부트스트랩, DI)** 에서 설정한다.

---

## 새 View 만들기

1) `Views` 폴더에 **`.axaml` + `.axaml.cs`** 쌍 생성
2) `ViewModels` 폴더에 대응되는 ViewModel 생성
3) View의 **DataContext**를 ViewModel로 연결

### 예: SettingsView.axaml

```xml
<UserControl xmlns="https://github.com/avaloniaui"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:vm="clr-namespace:MyApp.ViewModels"
             x:Class="MyApp.Views.SettingsView">

  <UserControl.DataContext>
    <vm:SettingsViewModel/>
  </UserControl.DataContext>

  <StackPanel Margin="16" Spacing="8">
    <TextBlock Text="{Binding Title}" FontSize="20"/>
    <Button Content="저장" Command="{Binding SaveCommand}"/>
  </StackPanel>
</UserControl>
```

### SettingsView.axaml.cs

```csharp
using Avalonia.Controls;

namespace MyApp.Views;

public partial class SettingsView : UserControl
{
    public SettingsView()
    {
        InitializeComponent();
    }
}
```

설계 팁
- View 내부에서 DataContext를 직접 생성하지 말고, **App 초기화 시 DI로 주입**하거나 **DataTemplate 매핑**을 사용하는 방식으로 점진적 확장성을 확보하는 것이 바람직하다.
  단, 간단 예제나 디자인 타임 힌트에서는 XAML 내부에서 생성해도 된다.

---

## View 종류와 사용 시점

| View 타입     | 용도                                   | 예시                         |
|---------------|----------------------------------------|------------------------------|
| `Window`      | 독립된 최상위 창                       | MainWindow, 독립 툴 윈도우  |
| `UserControl` | 다른 컨테이너에 조립되는 재사용 뷰    | SettingsView, ListItemView  |

- **Window**는 애플리케이션의 메인 셸이거나, 다이얼로그/보조 창으로 사용된다.
- **UserControl**은 페이지, 카드, 폼 조각, 항목 템플릿 등 **재사용 단위**로 만든다.

재사용 예

```xml
<StackPanel>
  <local:SettingsView/>
</StackPanel>
```

주의: 네임스페이스 선언 필요
`xmlns:local="clr-namespace:MyApp.Views"`

---

## 레이아웃 패널과 배치 전략

대표 패널
- `StackPanel`: 수직/수평 스택. 간단 폼/툴바에 적합
- `Grid`: 정교한 행/열 배치. 대다수 화면의 기본
- `DockPanel`: 상하좌우 고정 + 중앙 채우기
- `UniformGrid`: 균일한 격자 배치

### Grid 예시(대화면 기본 레이아웃)

```xml
<Grid>
  <Grid.RowDefinitions>
    <RowDefinition Height="Auto"/>
    <RowDefinition Height="*"/>
    <RowDefinition Height="Auto"/>
  </Grid.RowDefinitions>

  <Grid.ColumnDefinitions>
    <ColumnDefinition Width="220"/>
    <ColumnDefinition Width="*"/>
  </Grid.ColumnDefinitions>

  <!-- 헤더 -->
  <Border Grid.Row="0" Grid.ColumnSpan="2" Padding="12">
    <TextBlock Text="헤더"/>
  </Border>

  <!-- 좌측 내비 -->
  <StackPanel Grid.Row="1" Grid.Column="0" Spacing="6" Padding="12">
    <Button Content="홈" Command="{Binding NavigateHomeCommand}"/>
    <Button Content="설정" Command="{Binding NavigateSettingsCommand}"/>
  </StackPanel>

  <!-- 본문 -->
  <ContentControl Grid.Row="1" Grid.Column="1" Content="{Binding CurrentViewModel}" Margin="12"/>

  <!-- 푸터 -->
  <Border Grid.Row="2" Grid.ColumnSpan="2" Padding="12">
    <TextBlock Text="푸터"/>
  </Border>
</Grid>
```

---

## 스타일과 리소스 분리

View가 커질수록 **스타일/브러시/두께/여백** 등은 전역 리소스로 분리한다.

### App.axaml에서 리소스 사전 병합

```xml
<Application xmlns="https://github.com/avaloniaui"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             x:Class="MyApp.App">
  <Application.Resources>
    <ResourceDictionary>
      <ResourceDictionary.MergedDictionaries>
        <ResourceInclude Source="avares://MyApp/Resources/Colors.axaml"/>
        <ResourceInclude Source="avares://MyApp/Resources/Styles.axaml"/>
      </ResourceDictionary.MergedDictionaries>

      <!-- 공용 리소스 -->
      <SolidColorBrush x:Key="BrandBrush" Color="#3B82F6"/>
      <Thickness x:Key="PagePadding">16</Thickness>
    </ResourceDictionary>
  </Application.Resources>

  <Application.Styles>
    <FluentTheme Mode="Light"/>
  </Application.Styles>
</Application>
```

### Styles.axaml 예시

```xml
<ResourceDictionary xmlns="https://github.com/avaloniaui"
                    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">

  <Style Selector="Button.primary">
    <Setter Property="Background" Value="{StaticResource BrandBrush}"/>
    <Setter Property="Foreground" Value="White"/>
    <Setter Property="Padding" Value="8,6"/>
  </Style>

  <!-- 상태 기반 셀렉터 -->
  <Style Selector="Button:pointerover">
    <Setter Property="Opacity" Value="0.92"/>
  </Style>

  <Style Selector="Button:pressed">
    <Setter Property="RenderTransform">
      <Setter.Value>
        <ScaleTransform ScaleX="0.98" ScaleY="0.98"/>
      </Setter.Value>
    </Setter>
  </Style>
</ResourceDictionary>
```

사용

```xml
<Button Classes="primary" Content="저장"/>
```

---

## DataTemplate를 활용한 View 자동 연결

**핵심**: ViewModel 타입에 따라 자동으로 View를 선택하는 **DataTemplate 매핑**을 전역으로 등록하면, View에서는 `ContentControl.Content`에 ViewModel만 바인딩해도 뷰가 바뀐다.

### App.axaml에 템플릿 등록

```xml
<Application
  xmlns="https://github.com/avaloniaui"
  xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
  xmlns:vm="using:MyApp.ViewModels"
  xmlns:views="using:MyApp.Views"
  x:Class="MyApp.App">

  <Application.DataTemplates>
    <DataTemplate DataType="{x:Type vm:MainViewModel}">
      <views:MainView/>
    </DataTemplate>
    <DataTemplate DataType="{x:Type vm:SettingsViewModel}">
      <views:SettingsView/>
    </DataTemplate>
  </Application.DataTemplates>
</Application>
```

### 셸에서 화면 전환

```xml
<!-- MainWindow.axaml -->
<DockPanel>
  <StackPanel DockPanel.Dock="Top" Orientation="Horizontal" Spacing="8" Margin="8">
    <Button Content="홈" Command="{Binding NavigateHomeCommand}"/>
    <Button Content="설정" Command="{Binding NavigateSettingsCommand}"/>
  </StackPanel>

  <ContentControl Content="{Binding CurrentViewModel}" Margin="12"/>
</DockPanel>
```

ViewModel(예, ReactiveUI)

```csharp
using ReactiveUI;
using System.Reactive;

namespace MyApp.ViewModels;

public class MainWindowViewModel : ViewModelBase
{
    private ViewModelBase _current = new MainViewModel();
    public ViewModelBase CurrentViewModel
    {
        get => _current;
        set => this.RaiseAndSetIfChanged(ref _current, value);
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

---

## ViewLocator 패턴(이름 규칙 기반 자동 매핑)

DataTemplate 대신 **ViewLocator**를 사용하면 `FooViewModel` → `FooView`로 **이름 규칙**에 따라 자동 매핑할 수 있다.

```csharp
// Infrastructure/ViewLocator.cs
using System;
using Avalonia.Controls;
using Avalonia.Controls.Templates;
using MyApp.ViewModels;

namespace MyApp.Infrastructure;

public class ViewLocator : IDataTemplate
{
    public IControl Build(object? data)
    {
        if (data is null) return new TextBlock { Text = "Null ViewModel" };

        var name = data.GetType().FullName!.Replace("ViewModel", "View");
        var type = Type.GetType(name);

        if (type != null)
            return (Control)Activator.CreateInstance(type)!;

        return new TextBlock { Text = $"Not Found: {name}" };
    }

    public bool Match(object? data) => data is ViewModelBase;
}
```

App.axaml에 등록

```xml
<Application xmlns="https://github.com/avaloniaui"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:infra="using:MyApp.Infrastructure"
             x:Class="MyApp.App">
  <Application.DataTemplates>
    <infra:ViewLocator/>
  </Application.DataTemplates>
</Application>
```

이제 `ContentControl Content="{Binding CurrentViewModel}"`만으로 이름 규칙에 따라 View를 탐색한다.

---

## ControlTemplate과 템플릿 커스터마이징

`UserControl`은 내부를 XAML로 합성하는 방식이다. 더 고급화하려면 **TemplatedControl + ControlTemplate**로 **컨트롤의 외형 전체를 스타일/템플릿**으로 교체 가능하게 만든다.

간단 개념 예(버튼 템플릿 재정의)

```xml
<Style Selector="Button.special">
  <Setter Property="Template">
    <Setter.Value>
      <ControlTemplate>
        <Border CornerRadius="6" Background="{TemplateBinding Background}" Padding="8">
          <ContentPresenter/>
        </Border>
      </ControlTemplate>
    </Setter.Value>
  </Setter>
</Style>
```

사용

```xml
<Button Classes="special" Content="템플릿 버튼" Background="Silver"/>
```

포인트
- 템플릿은 **재사용성/테마 교체**에 유리하다.
- 사용자 정의 컨트롤을 라이브러리화할 때 TemplatedControl 기반이 적합하다.

---

## 명령/바인딩/컨버터 — View에서의 소비

View는 **상태 바인딩**과 **명령 바인딩**을 통해 ViewModel을 소비한다.

```xml
<StackPanel>
  <TextBox Text="{Binding Query, Mode=TwoWay, UpdateSourceTrigger=PropertyChanged}"/>
  <Button Content="검색" Command="{Binding SearchCommand}"/>
  <TextBlock Text="{Binding Status}"/>
</StackPanel>
```

컨버터 등록/사용

```xml
<Window xmlns:conv="using:MyApp.Converters">
  <Window.Resources>
    <conv:BoolToTextConverter x:Key="BoolToText"/>
  </Window.Resources>

  <TextBlock Text="{Binding IsBusy, Converter={StaticResource BoolToText}}"/>
</Window>
```

---

## 다이얼로그와 모달 View

- 간단 팝업: `Window`를 생성하여 `ShowDialog<T>()`
- 권장: View에서 직접 창을 만들기보다 **IDialogService** 인터페이스로 추상화하고, DI로 구현체를 주입해 ViewModel에서 호출

간단 예(서비스 인터페이스)

```csharp
public interface IDialogService
{
    Task<string?> PromptAsync(string title, string message);
}
```

ViewModel은 `IDialogService`만 참조하므로 테스트가 용이하다.

---

## 국제화·접근성·테스트

- 국제화: 문자열을 리소스 사전으로 분리, 문화권에 따라 리소스 교체
- 접근성: 키보드 탐색 가능, 포커스 표시, 색 대비 등 고려
- 테스트: **ViewModel 단위 테스트**를 우선. View의 시각 테스트는 범위를 최소화하고, 레이아웃/바인딩은 스냅샷 또는 수동 검증

xUnit으로 ViewModel 테스트 예

```csharp
using Xunit;
using MyApp.ViewModels;

public class SettingsViewModelTests
{
    [Fact]
    public void Title_Default_NotEmpty()
    {
        var vm = new SettingsViewModel();
        Assert.False(string.IsNullOrWhiteSpace(vm.Title));
    }
}
```

---

## 성능과 구조 팁

- 복잡한 화면은 **UserControl**로 분해하여 유지보수성을 높인다.
- 스타일/리소스는 **전역 사전**으로 관리하여 중복/상충을 최소화한다.
- 리스트/그리드에 대량 데이터 표시 시 **가상화가 가능한 컨트롤** 사용을 고려하고, 데이터는 **지연 로딩**한다.
- 빈번한 바인딩 업데이트가 필요한 TextBox는 `UpdateSourceTrigger=PropertyChanged`를 사용하되, 비용이 높은 계산은 **디바운스**나 명시적 커맨드로 조절한다.

---

## 종합 예제: 셸 + 페이지 전환 + 스타일 + 리소스

폴더 구조

```
MyApp/
  App.axaml
  App.axaml.cs
  Resources/
    Colors.axaml
    Styles.axaml
  Views/
    MainWindow.axaml
    MainWindow.axaml.cs
    HomeView.axaml
    SettingsView.axaml
  ViewModels/
    ViewModelBase.cs
    MainWindowViewModel.cs
    HomeViewModel.cs
    SettingsViewModel.cs
```

리소스

```xml
<!-- Resources/Colors.axaml -->
<ResourceDictionary xmlns="https://github.com/avaloniaui"
                    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">
  <SolidColorBrush x:Key="BrandBrush" Color="#3B82F6"/>
</ResourceDictionary>
```

```xml
<!-- Resources/Styles.axaml -->
<ResourceDictionary xmlns="https://github.com/avaloniaui"
                    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">
  <Style Selector="Button.primary">
    <Setter Property="Background" Value="{StaticResource BrandBrush}"/>
    <Setter Property="Foreground" Value="White"/>
    <Setter Property="Padding" Value="10,6"/>
  </Style>
</ResourceDictionary>
```

App.axaml

```xml
<Application xmlns="https://github.com/avaloniaui"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:vm="using:MyApp.ViewModels"
             xmlns:views="using:MyApp.Views"
             x:Class="MyApp.App">

  <Application.Resources>
    <ResourceDictionary>
      <ResourceDictionary.MergedDictionaries>
        <ResourceInclude Source="avares://MyApp/Resources/Colors.axaml"/>
        <ResourceInclude Source="avares://MyApp/Resources/Styles.axaml"/>
      </ResourceDictionary.MergedDictionaries>
    </ResourceDictionary>
  </Application.Resources>

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

MainWindow.axaml

```xml
<Window xmlns="https://github.com/avaloniaui"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        x:Class="MyApp.Views.MainWindow"
        Width="960" Height="640" Title="MyApp">

  <Grid>
    <Grid.RowDefinitions>
      <RowDefinition Height="Auto"/>
      <RowDefinition Height="*"/>
    </Grid.RowDefinitions>

    <DockPanel Grid.Row="0" LastChildFill="False" Margin="8" >
      <Button Classes="primary" Content="홈" Margin="0,0,8,0"
              Command="{Binding NavigateHomeCommand}"/>
      <Button Classes="primary" Content="설정"
              Command="{Binding NavigateSettingsCommand}"/>
    </DockPanel>

    <Border Grid.Row="1" Margin="12" Padding="12">
      <ContentControl Content="{Binding CurrentViewModel}"/>
    </Border>
  </Grid>
</Window>
```

HomeView.axaml

```xml
<UserControl xmlns="https://github.com/avaloniaui"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             x:Class="MyApp.Views.HomeView">
  <StackPanel Spacing="8">
    <TextBlock Text="홈" FontSize="20"/>
    <TextBlock Text="{Binding Message}"/>
  </StackPanel>
</UserControl>
```

SettingsView.axaml

```xml
<UserControl xmlns="https://github.com/avaloniaui"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             x:Class="MyApp.Views.SettingsView">
  <StackPanel Spacing="8">
    <TextBlock Text="설정" FontSize="20"/>
    <TextBox Text="{Binding Title, Mode=TwoWay}" Width="260"/>
    <Button Content="저장" Command="{Binding SaveCommand}"/>
  </StackPanel>
</UserControl>
```

ViewModels 예시(간결 버전)

```csharp
// ViewModelBase.cs (ReactiveUI)
using ReactiveUI;

namespace MyApp.ViewModels;

public class ViewModelBase : ReactiveObject { }
```

```csharp
// HomeViewModel.cs
namespace MyApp.ViewModels;

public class HomeViewModel : ViewModelBase
{
    private string _message = "환영합니다.";
    public string Message
    {
        get => _message;
        set => this.RaiseAndSetIfChanged(ref _message, value);
    }
}
```

```csharp
// SettingsViewModel.cs
using System.Reactive;
using ReactiveUI;

namespace MyApp.ViewModels;

public class SettingsViewModel : ViewModelBase
{
    private string _title = "기본 설정";
    public string Title
    {
        get => _title;
        set => this.RaiseAndSetIfChanged(ref _title, value);
    }

    public ReactiveCommand<Unit, Unit> SaveCommand { get; }

    public SettingsViewModel()
    {
        SaveCommand = ReactiveCommand.Create(() =>
        {
            // 저장 로직
        });
    }
}
```

```csharp
// MainWindowViewModel.cs
using System.Reactive;
using ReactiveUI;

namespace MyApp.ViewModels;

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

---

## 설계 체크리스트

- View는 **표현과 레이아웃**만, 로직은 ViewModel로 이동한다.
- **DataContext 주입**은 조립 루트(App 초기화/DI)나 DataTemplate/Locator로 처리한다.
- 공통 스타일/브러시/두께는 **리소스 사전**으로 분리, 전역 병합한다.
- 화면 전환은 **ContentControl + DataTemplate** 패턴이 단순하고 강력하다.
- 복잡한 외형 치환이 필요하면 **ControlTemplate**을 고려한다.
- 다이얼로그/네비게이션은 **서비스 인터페이스**로 추상화하여 ViewModel 테스트 가능성을 확보한다.
- 대량 데이터/이미지에는 가상화/지연 로딩/비동기 I/O를 사용한다.

---

## 결론

초안의 핵심을 유지하면서, 실전 애플리케이션에서 View를 **조립 가능한 단위**로 설계하고, **DataTemplate/Locator**로 뷰 전환을 자동화하며, **스타일/리소스/템플릿**으로 테마를 체계화하는 방법을 확장했다.
이 원칙을 따르면 View는 **가볍고 재사용 가능**해지고, ViewModel/서비스는 **테스트 가능한 구조**를 유지한다. 애플리케이션이 커져도 유지보수가 쉬운 **견고한 MVVM 기반 UI**를 지속적으로 확장할 수 있다.
