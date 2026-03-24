---
layout: post
title: Avalonia - View 구조
date: 2025-01-04 19:20:23 +0900
category: Avalonia
---
# Avalonia의 View 구조와 확장 방법

View는 사용자가 직접 보는 UI를 의미하며, Avalonia에서는 `.axaml` 파일(XAML)로 정의합니다. MVVM 패턴에서 View는 표현과 레이아웃, 스타일에 집중하고, 상태와 동작은 ViewModel이 담당합니다. View는 일반적으로 `DataContext`로 ViewModel을 참조하며, 바인딩을 통해 상호작용합니다.

---

## View의 기본 구성

Avalonia의 View는 크게 두 파일로 구성됩니다.

- **.axaml 파일**: UI의 구조와 스타일을 선언하는 XAML 마크업
- **.axaml.cs 파일**: 코드 비하인드로, 주로 생성자에서 `InitializeComponent()`를 호출하고 가끔 이벤트 핸들러를 연결합니다.

**MainWindow.axaml 예시**

```xml
<Window xmlns="https://github.com/avaloniaui"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:vm="clr-namespace:MyApp.ViewModels"
        x:Class="MyApp.Views.MainWindow"
        Title="Main Window" Width="400" Height="300">

  <Window.DataContext>
    <vm:MainWindowViewModel/>
  </Window.DataContext>

  <StackPanel Margin="20" Spacing="10">
    <TextBlock Text="Hello Avalonia!" FontSize="24" />
    <Button Content="클릭" Command="{Binding ClickCommand}"/>
  </StackPanel>
</Window>
```

**MainWindow.axaml.cs**

```csharp
using Avalonia.Controls;

namespace MyApp.Views;

public partial class MainWindow : Window
{
    public MainWindow() => InitializeComponent();
}
```

> **DataContext**는 View가 참조할 ViewModel을 지정합니다. 위 예제에서는 XAML 내에서 직접 인스턴스를 생성했지만, 실제 애플리케이션에서는 DI 컨테이너나 뷰 로케이터를 통해 주입하는 경우가 많습니다.

---

## View의 종류와 사용처

| View 타입     | 설명                                                                 |
|---------------|----------------------------------------------------------------------|
| `Window`      | 독립된 최상위 창. 메인 윈도우, 다이얼로그, 팝업 등에 사용합니다.     |
| `UserControl` | 재사용 가능한 UI 조각. 페이지, 패널, 리스트 아이템 템플릿 등에 사용합니다. |

`UserControl`은 다른 View 안에 포함할 수 있습니다.

```xml
<StackPanel>
  <local:SettingsView/>
</StackPanel>
```

네임스페이스 선언이 필요합니다. `xmlns:local="clr-namespace:MyApp.Views"`

---

## 새 View 만들기

1. **Views** 폴더에 `.axaml` 파일과 `.axaml.cs` 파일을 추가합니다.
2. **ViewModels** 폴더에 대응하는 ViewModel을 만듭니다.
3. View의 `DataContext`를 ViewModel로 연결합니다.

**SettingsView.axaml**

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

**SettingsView.axaml.cs**

```csharp
using Avalonia.Controls;

namespace MyApp.Views;

public partial class SettingsView : UserControl
{
    public SettingsView() => InitializeComponent();
}
```

**SettingsViewModel.cs** (간단한 예)

```csharp
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;

namespace MyApp.ViewModels;

public partial class SettingsViewModel : ObservableObject
{
    [ObservableProperty]
    private string _title = "설정";

    [RelayCommand]
    private void Save()
    {
        // 저장 로직
    }
}
```

---

## 레이아웃 패널 선택하기

Avalonia는 다양한 패널을 제공합니다. 각 패널은 배치 방식이 다르므로 용도에 맞게 선택해야 합니다.

| 패널          | 특징                                                                 |
|---------------|----------------------------------------------------------------------|
| `StackPanel`  | 자식 요소를 수직 또는 수평으로 순서대로 쌓습니다. 폼, 툴바에 적합합니다. |
| `Grid`        | 행과 열을 정의하여 정교한 배치를 합니다. 대부분의 화면에 사용합니다.     |
| `DockPanel`   | 자식 요소를 상, 하, 좌, 우에 도킹하고 남은 공간을 채웁니다.           |
| `UniformGrid` | 모든 셀의 크기를 균일하게 만드는 격자입니다.                          |

**Grid 예시 – 헤더, 사이드바, 본문, 푸터 구성**

```xml
<Grid>
  <Grid.RowDefinitions>
    <RowDefinition Height="Auto"/>
    <RowDefinition Height="*"/>
    <RowDefinition Height="Auto"/>
  </Grid.RowDefinitions>
  <Grid.ColumnDefinitions>
    <ColumnDefinition Width="200"/>
    <ColumnDefinition Width="*"/>
  </Grid.ColumnDefinitions>

  <Border Grid.Row="0" Grid.ColumnSpan="2" Background="LightGray" Padding="12">
    <TextBlock Text="헤더 영역"/>
  </Border>

  <StackPanel Grid.Row="1" Grid.Column="0" Background="LightBlue" Padding="8">
    <Button Content="메뉴1"/>
    <Button Content="메뉴2"/>
  </StackPanel>

  <Border Grid.Row="1" Grid.Column="1" Background="White" Padding="12">
    <TextBlock Text="본문 내용"/>
  </Border>

  <Border Grid.Row="2" Grid.ColumnSpan="2" Background="LightGray" Padding="8">
    <TextBlock Text="푸터"/>
  </Border>
</Grid>
```

`Height="Auto"`는 내용물 크기에 맞추고, `Height="*"`는 남은 공간을 모두 차지합니다.

---

## 스타일과 리소스 분리

반복적으로 사용되는 브러시, 두께, 스타일은 리소스로 정의해 중앙 관리합니다.

**App.axaml – 리소스 병합**

```xml
<Application ...>
  <Application.Resources>
    <ResourceDictionary>
      <ResourceDictionary.MergedDictionaries>
        <ResourceInclude Source="avares://MyApp/Resources/Colors.axaml"/>
        <ResourceInclude Source="avares://MyApp/Resources/Styles.axaml"/>
      </ResourceDictionary.MergedDictionaries>

      <SolidColorBrush x:Key="BrandBrush" Color="#3B82F6"/>
      <Thickness x:Key="PagePadding">16</Thickness>
    </ResourceDictionary>
  </Application.Resources>

  <Application.Styles>
    <FluentTheme Mode="Light"/>
  </Application.Styles>
</Application>
```

**Styles.axaml – 컨트롤 스타일 정의**

```xml
<ResourceDictionary>
  <Style Selector="Button.primary">
    <Setter Property="Background" Value="{StaticResource BrandBrush}"/>
    <Setter Property="Foreground" Value="White"/>
    <Setter Property="Padding" Value="8,6"/>
  </Style>

  <Style Selector="Button:pointerover">
    <Setter Property="Opacity" Value="0.92"/>
  </Style>
</ResourceDictionary>
```

사용 예:

```xml
<Button Classes="primary" Content="저장" Padding="{StaticResource PagePadding}"/>
```

---

## DataTemplate으로 View와 ViewModel 자동 연결

`ContentControl`의 `Content`에 ViewModel을 바인딩하면, 등록된 `DataTemplate`에 따라 적절한 View가 자동으로 선택됩니다.

**App.axaml에 DataTemplate 등록**

```xml
<Application xmlns="https://github.com/avaloniaui"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:vm="using:MyApp.ViewModels"
             xmlns:views="using:MyApp.Views">
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

**메인 뷰에서 콘텐츠 영역 바인딩**

```xml
<ContentControl Content="{Binding CurrentViewModel}"/>
```

`CurrentViewModel`이 바뀌면 자동으로 해당 View가 나타납니다.

---

## ViewLocator 패턴: 이름 규칙 기반 자동 매핑

`DataTemplate`을 일일이 등록하는 대신, **ViewLocator**를 사용하면 `FooViewModel` → `FooView` 규칙으로 자동 매핑할 수 있습니다.

```csharp
// Infrastructure/ViewLocator.cs
using System;
using Avalonia.Controls;
using Avalonia.Controls.Templates;

namespace MyApp.Infrastructure;

public class ViewLocator : IDataTemplate
{
    public IControl Build(object? data)
    {
        if (data is null) return new TextBlock { Text = "Null ViewModel" };

        var viewModelType = data.GetType();
        var viewTypeName = viewModelType.FullName!.Replace("ViewModel", "View");
        var viewType = Type.GetType(viewTypeName);

        if (viewType != null)
            return (Control)Activator.CreateInstance(viewType)!;

        return new TextBlock { Text = $"View not found: {viewTypeName}" };
    }

    public bool Match(object? data) => data is ViewModelBase;
}
```

App.axaml에 등록:

```xml
<Application ...>
  <Application.DataTemplates>
    <infra:ViewLocator/>
  </Application.DataTemplates>
</Application>
```

이제 `ContentControl`에 ViewModel을 넣기만 하면 `ViewModel` 접미사가 `View`로 바뀐 타입을 찾아 인스턴스화합니다.

---

## 명령과 바인딩

View는 ViewModel의 속성과 명령을 바인딩합니다.

```xml
<StackPanel>
  <TextBox Text="{Binding UserName, Mode=TwoWay, UpdateSourceTrigger=PropertyChanged}"/>
  <Button Content="저장" Command="{Binding SaveCommand}"/>
  <TextBlock Text="{Binding StatusMessage}"/>
</StackPanel>
```

**컨버터 예제** – 불리언 값을 문자열로 변환

```csharp
public class BoolToTextConverter : IValueConverter
{
    public object? Convert(object? value, Type targetType, object? parameter, CultureInfo culture)
        => (value is true) ? "활성" : "비활성";

    public object? ConvertBack(object? value, Type targetType, object? parameter, CultureInfo culture)
        => (value as string) == "활성";
}
```

XAML에서 사용:

```xml
<Window.Resources>
  <conv:BoolToTextConverter x:Key="BoolToText"/>
</Window.Resources>

<TextBlock Text="{Binding IsActive, Converter={StaticResource BoolToText}}"/>
```

---

## 다이얼로그 처리

간단한 다이얼로그는 `Window`를 새로 만들어 `ShowDialog`로 띄웁니다.

```csharp
var dialog = new MyDialogWindow();
var result = await dialog.ShowDialog<bool?>(this);
```

테스트와 유지보수를 고려한다면 **IDialogService** 인터페이스를 만들어 ViewModel에서 추상화하는 것이 좋습니다.

```csharp
public interface IDialogService
{
    Task<string?> ShowInputDialogAsync(string title, string message);
}
```

ViewModel은 이 인터페이스에 의존하고, 실제 구현체는 View 계층에서 Window를 생성합니다.

---

## 국제화와 접근성

- **국제화**: 문자열 리소스를 `.resx` 또는 리소스 사전으로 분리하고, `CultureInfo` 변경 시 리소스를 다시 로드합니다.
- **접근성**: 키보드 탐색(`TabIndex`), 포커스 시각화, 스크린 리더 대응을 고려합니다.

---

## 테스트 전략

ViewModel은 순수 C# 클래스이므로 xUnit 같은 도구로 단위 테스트가 가능합니다.

```csharp
[Fact]
public void SaveCommand_WhenCalled_UpdatesStatus()
{
    var vm = new SettingsViewModel();
    vm.SaveCommand.Execute(null);
    Assert.Equal("저장됨", vm.StatusMessage);
}
```

View 자체의 테스트는 최소화하고, UI 로직은 최대한 ViewModel로 옮기는 것이 핵심입니다.

---

## 성능 팁

- 대량의 데이터를 표시할 때는 `ListBox`나 `DataGrid`의 가상화를 활성화합니다.
- `ObservableCollection`에 많은 항목을 추가할 때는 배치 작업을 고려합니다.
- 이미지 등 큰 리소스는 비동기 로딩하고, 필요할 때만 메모리에 올립니다.
- 복잡한 레이아웃은 `UserControl`으로 분할해 렌더링 부담을 줄입니다.

---

## 종합 예제: 간단한 셸 애플리케이션

**폴더 구조**

```
MyApp/
  App.axaml
  Views/
    MainWindow.axaml
    HomeView.axaml
    SettingsView.axaml
  ViewModels/
    ViewModelBase.cs
    MainWindowViewModel.cs
    HomeViewModel.cs
    SettingsViewModel.cs
  Resources/
    Styles.axaml
```

**App.axaml** (DataTemplate 등록)

```xml
<Application xmlns="https://github.com/avaloniaui"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:vm="using:MyApp.ViewModels"
             xmlns:views="using:MyApp.Views">
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

**MainWindow.axaml**

```xml
<Window ...>
  <DockPanel>
    <StackPanel DockPanel.Dock="Top" Orientation="Horizontal" Margin="8">
      <Button Content="홈" Command="{Binding NavigateHomeCommand}"/>
      <Button Content="설정" Command="{Binding NavigateSettingsCommand}"/>
    </StackPanel>
    <ContentControl Content="{Binding CurrentViewModel}" Margin="12"/>
  </DockPanel>
</Window>
```

**MainWindowViewModel.cs** (ReactiveUI 예시)

```csharp
public class MainWindowViewModel : ViewModelBase
{
    private ViewModelBase _current = new HomeViewModel();
    public ViewModelBase CurrentViewModel
    {
        get => _current;
        set => this.RaiseAndSetIfChanged(ref _current, value);
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

**HomeView.axaml**

```xml
<UserControl ...>
  <StackPanel Margin="16">
    <TextBlock Text="홈 화면" FontSize="20"/>
    <TextBlock Text="{Binding WelcomeMessage}"/>
  </StackPanel>
</UserControl>
```

**HomeViewModel.cs**

```csharp
public class HomeViewModel : ViewModelBase
{
    public string WelcomeMessage => "Avalonia에 오신 것을 환영합니다!";
}
```

**SettingsView.axaml**

```xml
<UserControl ...>
  <StackPanel Margin="16" Spacing="8">
    <TextBlock Text="설정" FontSize="20"/>
    <TextBox Text="{Binding UserName, Mode=TwoWay}" Watermark="사용자 이름"/>
    <Button Content="저장" Command="{Binding SaveCommand}"/>
  </StackPanel>
</UserControl>
```

**SettingsViewModel.cs**

```csharp
public partial class SettingsViewModel : ObservableObject
{
    [ObservableProperty]
    private string _userName = string.Empty;

    [RelayCommand]
    private void Save()
    {
        // 저장 로직
    }
}
```

---

## 설계 시 주의할 점

- View는 **로직을 포함하지 않습니다**. 버튼 클릭 이벤트가 있다면 ViewModel의 명령으로 바인딩합니다.
- `DataContext`는 외부에서 주입하는 것이 좋습니다. XAML 내에서 직접 생성하면 테스트와 교체가 어렵습니다.
- 공통 스타일과 리소스는 전역으로 빼서 중복을 제거합니다.
- 화면 전환이 필요한 경우 `ContentControl`과 `DataTemplate` 조합을 기본으로 사용합니다.
- 다이얼로그, 파일 열기/저장 등 플랫폼 의존 기능은 서비스로 추상화합니다.

---

## 결론

Avalonia의 View는 XAML로 선언하고, MVVM 패턴에 따라 DataContext를 통해 ViewModel과 연결됩니다. 이 문서에서는 View의 기본 구성부터 레이아웃, 스타일, 데이터 템플릿, 뷰 로케이터, 명령 바인딩, 다이얼로그 처리, 테스트, 성능 팁까지 초중급 개발자가 실제로 활용할 수 있는 내용을 다루었습니다. 이 원칙을 따르면 유지보수성과 테스트 용이성을 갖춘 크로스 플랫폼 UI를 구축할 수 있습니다.