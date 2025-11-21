---
layout: post
title: WPF - App, MainWindow
date: 2025-08-27 20:25:23 +0900
category: WPF
---
# App.xaml / MainWindow.xaml

WPF 애플리케이션의 **진입점(App.xaml)**과 **첫 화면(MainWindow.xaml)**은 프로젝트의 뼈대를 이룹니다.
이 글에서는 두 파일의 **역할**, **구성 포인트**, **권장 패턴(MVVM/리소스/스타트업)**을 실제 코드와 함께 상세히 설명합니다.

---

## App.xaml: 애플리케이션 진입점 & 전역 리소스

### App.xaml의 역할

- **애플리케이션 수명주기**: 시작(Startup), 종료(Exit), 세션 종료(SessionEnding) 이벤트 연결
- **전역 리소스 루트**: `ResourceDictionary`를 통해 **스타일/템플릿/브러시/컨버터** 등 앱 전체 공용 리소스 제공
- **시작 화면 지정**: `StartupUri`로 최초 윈도우 지정, 혹은 `Startup` 이벤트에서 동적으로 결정
- **종료 조건 관리**: `ShutdownMode`(`OnLastWindowClose`, `OnMainWindowClose`, `OnExplicitShutdown`)
- **테마 병합/스위칭**: `MergedDictionaries`로 사전 구조화 후 런타임 교체

### App.xaml 기본 예시

```xml
<Application x:Class="MyWpfApp.App"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             StartupUri="Views/MainWindow.xaml"
             ShutdownMode="OnMainWindowClose">
  <Application.Resources>
    <ResourceDictionary>
      <ResourceDictionary.MergedDictionaries>
        <!-- 색/브러시 팔레트 -->
        <ResourceDictionary Source="pack://application:,,,/Themes/Colors.xaml"/>
        <!-- 공통 스타일(버튼, 텍스트 등) -->
        <ResourceDictionary Source="pack://application:,,,/Themes/Styles.xaml"/>
        <!-- 컨트롤 템플릿 -->
        <ResourceDictionary Source="pack://application:,,,/Themes/ControlTemplates.xaml"/>
        <!-- 데이터 템플릿(뷰-뷰모델 매핑용) -->
        <ResourceDictionary Source="pack://application:,,,/Themes/DataTemplates.xaml"/>
      </ResourceDictionary.MergedDictionaries>
      <!-- 앱 전역에서 쓰는 값/컨버터 -->
      <SolidColorBrush x:Key="PrimaryBrush" Color="#4F46E5"/>
      <local:StringNotEmptyToBoolConverter x:Key="NotEmpty"/>
    </ResourceDictionary>
  </Application.Resources>
</Application>
```

> **Pack URI**
> 같은 어셈블리: `pack://application:,,,/경로/파일.xaml`
> 다른 어셈블리 `MyLib`: `pack://application:,,,/MyLib;component/경로/파일.xaml`

### App.xaml.cs에서 커스텀 시작 흐름

`StartupUri` 대신 **Startup 이벤트**로 초기 로직(로그인/권한/설정)을 처리하고 창을 선택합니다.

```csharp
public partial class App : Application
{
    protected override void OnStartup(StartupEventArgs e)
    {
        base.OnStartup(e);

        // 예: 관리자 모드면 AdminWindow, 아니면 MainWindow
        Window shell = (e.Args.Contains("--admin"))
            ? new AdminWindow()
            : new MainWindow();

        // MainWindow 속성 지정(ShutdownMode=OnMainWindowClose 시 중요)
        this.MainWindow = shell;
        shell.Show();
    }

    protected override void OnExit(ExitEventArgs e)
    {
        base.OnExit(e);
        // 로그 flush, 리소스 정리 등
    }
}
```

### 전역 예외 처리(권장)

```csharp
public partial class App : Application
{
    public App()
    {
        this.DispatcherUnhandledException += (s, e) =>
        {
            // UI 스레드 예외
            MessageBox.Show(e.Exception.Message, "오류");
            e.Handled = true; // 앱을 종료하지 않도록
        };
        AppDomain.CurrentDomain.UnhandledException += (s, e) =>
        {
            // 비-UI 스레드 예외 로깅
        };
        TaskScheduler.UnobservedTaskException += (s, e) =>
        {
            // 관찰되지 않은 Task 예외
        };
    }
}
```

### 테마 스위칭(런타임에 MergedDictionaries 교체)

```csharp
public static class ThemeManager
{
    public static void Apply(Uri themeUri)
    {
        var app = Application.Current;
        var dictionaries = app.Resources.MergedDictionaries;
        // 예: 첫 번째 딕셔너리를 테마로 가정하고 교체
        dictionaries[0] = new ResourceDictionary { Source = themeUri };
    }
}
// 사용: ThemeManager.Apply(new Uri("pack://application:,,,/Themes/Colors.Dark.xaml"));
```

### ShutdownMode 이해

- `OnLastWindowClose`(기본): 마지막 창이 닫히면 종료
- `OnMainWindowClose`: **MainWindow**가 닫힐 때만 종료(다른 창은 무관)
- `OnExplicitShutdown`: `Application.Shutdown()` 호출 시에만 종료

---

## MainWindow.xaml: 루트 창 & 화면 구성

### MainWindow.xaml의 역할

- **최초 표시되는 화면**(혹은 Shell): 메뉴/내비게이션/컨텐츠 영역/상태바 등 **앱 골격**
- **Window 수준 리소스**: 페이지 전용 스타일/템플릿/브러시(앱 전역 리소스와 분리)
- **데이터 바인딩의 중심**: `DataContext`(ViewModel) 연결, `CommandBindings`/`InputBindings`로 단축키/명령 처리
- **레이아웃 루트**: `Grid`(권장)로 **반응형 배치**, `Row/Column` 설계

### MainWindow.xaml 기본 구조(실전형 예시)

```xml
<Window x:Class="MyWpfApp.Views.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:d="http://schemas.microsoft.com/expression/blend/2008"
        xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"
        xmlns:local="clr-namespace:MyWpfApp"
        mc:Ignorable="d"
        Title="MyWpfApp"
        WindowStartupLocation="CenterScreen"
        Height="720" Width="1200"
        MinHeight="500" MinWidth="900"
        Icon="/Resources/Icons/app.ico"
        Background="{DynamicResource AppBackgroundBrush}">

  <!-- 창 전용 리소스(전역 리소스와 분리) -->
  <Window.Resources>
    <Style TargetType="Button" x:Key="PrimaryButton">
      <Setter Property="Padding" Value="12,6"/>
      <Setter Property="Background" Value="{StaticResource PrimaryBrush}"/>
      <Setter Property="Foreground" Value="White"/>
    </Style>
  </Window.Resources>

  <!-- Window-level Command & InputBindings (Ctrl+S 저장 등) -->
  <Window.CommandBindings>
    <CommandBinding Command="ApplicationCommands.Save"
                    CanExecute="Save_CanExecute"
                    Executed="Save_Executed"/>
  </Window.CommandBindings>
  <Window.InputBindings>
    <KeyBinding Command="ApplicationCommands.Save" Key="S" Modifiers="Control"/>
  </Window.InputBindings>

  <!-- 레이아웃 루트 -->
  <Grid>
    <Grid.RowDefinitions>
      <RowDefinition Height="Auto"/>   <!-- 상단 메뉴/툴바 -->
      <RowDefinition/>                 <!-- 컨텐츠 영역 -->
      <RowDefinition Height="Auto"/>   <!-- 상태바 -->
    </Grid.RowDefinitions>

    <!-- 상단 바(메뉴/툴바) -->
    <DockPanel Grid.Row="0" LastChildFill="False" Margin="0,0,0,8">
      <Menu DockPanel.Dock="Top">
        <MenuItem Header="_File">
          <MenuItem Header="_New" Command="ApplicationCommands.New"/>
          <MenuItem Header="_Open" Command="ApplicationCommands.Open"/>
          <Separator/>
          <MenuItem Header="_Save" Command="ApplicationCommands.Save"/>
          <MenuItem Header="E_xit" Click="Exit_Click"/>
        </MenuItem>
      </Menu>
      <ToolBar>
        <Button Content="Save" Command="ApplicationCommands.Save" Style="{StaticResource PrimaryButton}"/>
      </ToolBar>
    </DockPanel>

    <!-- 컨텐츠 영역(내비게이션/탭/프레임/ContentControl 중 택1) -->
    <ContentControl Grid.Row="1"
                    Content="{Binding CurrentView}"
                    HorizontalContentAlignment="Stretch"
                    VerticalContentAlignment="Stretch"/>

    <!-- 상태바 -->
    <StatusBar Grid.Row="2">
      <StatusBarItem Content="{Binding StatusText}" />
      <StatusBarItem HorizontalAlignment="Right" Content="{Binding Now, StringFormat='{}{0:HH:mm:ss}'}"/>
    </StatusBar>
  </Grid>
</Window>
```

> **컨텐츠 바인딩 패턴**
> `ContentControl.Content`에 `CurrentView`(뷰모델 또는 뷰)를 바인딩하고
> **DataTemplate**로 형식→뷰를 매핑하면 **페이지 전환**을 깔끔하게 구현할 수 있습니다.
> (이 매핑은 보통 `Themes/DataTemplates.xaml`에 배치)

### MainWindow.xaml.cs (필요 최소한)

MVVM을 쓰더라도 **DataContext 연결**과 **윈도우 이벤트** 정도는 코드비하인드에서 담당할 수 있습니다.

```csharp
public partial class MainWindow : Window
{
    public MainWindow() // (DI 미사용 시)
    {
        InitializeComponent();
        DataContext = new MainViewModel(); // 간단 예시
    }

    // CommandBinding 예
    private void Save_CanExecute(object sender, CanExecuteRoutedEventArgs e)
    {
        e.CanExecute = (DataContext as MainViewModel)?.CanSave ?? false;
    }

    private void Save_Executed(object sender, ExecutedRoutedEventArgs e)
    {
        (DataContext as MainViewModel)?.Save();
    }

    private void Exit_Click(object sender, RoutedEventArgs e)
    {
        Application.Current.Shutdown();
    }
}
```

> **권장**: 실제 프로젝트에서는 **의존성 주입**으로 `MainWindow(MainViewModel vm)` 생성자 주입을 선호합니다.
> (App.xaml.cs에서 Host/ServiceProvider로 생성하여 `DataContext` 자동 세팅)

### 디자인 타임 데이터 주입(편리)

XAML 미리보기에서 더 나은 디자이너 경험을 위해 **d:DataContext**를 사용합니다.

```xml
<Window ... xmlns:d="http://schemas.microsoft.com/expression/blend/2008" mc:Ignorable="d">
  <Window.DataContext>
    <vm:MainViewModel /> <!-- 런타임 바인딩 -->
  </Window.DataContext>
  <!-- 디자인 전용 -->
  <d:Window.DataContext>
    <vm:Design.MainDesignViewModel />
  </d:Window.DataContext>
</Window>
```

### Window 속성 설계 팁

- **크기 정책**: `SizeToContent`, `MinWidth/MinHeight`, `MaxWidth/MaxHeight`
- **스타일/크롬**: `WindowStyle=None` + `AllowsTransparency=True`로 커스텀 타이틀바(드래그/리사이즈 코드 필요)
- **시작 위치**: `WindowStartupLocation=CenterScreen/CenterOwner`
- **아이콘/제목**: 배포 빌드에서 아이콘 누락 주의(리소스/경로 점검)

---

## 조직 전략

### 전역 vs 창 전용

- **전역(App.xaml)**: 공통 스타일/색상/템플릿(앱 전체에 일관성)
- **창/뷰 전용(Window.Resources)**: 특정 화면에만 필요한 것(충돌 최소화, 빌드/로드 성능 개선)

### StaticResource vs DynamicResource

- **StaticResource**: 로드 시점 고정(성능 우수). **변경 감지 없음**
- **DynamicResource**: 런타임 변경 반영(테마 전환 등). **성능 비용 有**

```xml
<TextBlock Foreground="{StaticResource PrimaryBrush}"/>
<TextBlock Foreground="{DynamicResource PrimaryBrush}"/> <!-- 테마 전환 대응 -->
```

### DataTemplate로 뷰-뷰모델 연결(강력 추천)

```xml
<!-- Themes/DataTemplates.xaml -->
<ResourceDictionary
    xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
    xmlns:view="clr-namespace:MyWpfApp.Views"
    xmlns:vm="clr-namespace:MyWpfApp.ViewModels">
  <DataTemplate DataType="{x:Type vm:DashboardViewModel}">
    <view:DashboardView/>
  </DataTemplate>
  <DataTemplate DataType="{x:Type vm:SettingsViewModel}">
    <view:SettingsView/>
  </DataTemplate>
</ResourceDictionary>
```
- `ContentControl.Content`에 **뷰모델 인스턴스**만 바인딩하면 **알아서 대응하는 뷰**가 렌더링됩니다.

---

## App.xaml + MainWindow.xaml 통합 샘플(요약)

**App.xaml**
```xml
<Application x:Class="MyWpfApp.App"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             Startup="OnStartup"
             ShutdownMode="OnMainWindowClose">
  <Application.Resources>
    <ResourceDictionary>
      <ResourceDictionary.MergedDictionaries>
        <ResourceDictionary Source="pack://application:,,,/Themes/Colors.xaml"/>
        <ResourceDictionary Source="pack://application:,,,/Themes/Styles.xaml"/>
        <ResourceDictionary Source="pack://application:,,,/Themes/DataTemplates.xaml"/>
      </ResourceDictionary.MergedDictionaries>
    </ResourceDictionary>
  </Application.Resources>
</Application>
```

**App.xaml.cs**
```csharp
public partial class App : Application
{
    protected override void OnStartup(StartupEventArgs e)
    {
        base.OnStartup(e);
        var main = new Views.MainWindow
        {
            DataContext = new ViewModels.MainViewModel()
        };
        this.MainWindow = main;
        main.Show();
    }
}
```

**MainWindow.xaml**
```xml
<Window x:Class="MyWpfApp.Views.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="MyWpfApp" Height="720" Width="1200">
  <Grid>
    <Grid.RowDefinitions>
      <RowDefinition Height="Auto"/>
      <RowDefinition/>
    </Grid.RowDefinitions>

    <Menu Grid.Row="0">
      <MenuItem Header="_File">
        <MenuItem Header="_Save" Command="ApplicationCommands.Save"/>
      </MenuItem>
    </Menu>

    <ContentControl Grid.Row="1" Content="{Binding CurrentView}"/>
  </Grid>
</Window>
```

---

## 체크리스트(요약)

- **App.xaml**
  - `MergedDictionaries`로 리소스 모듈화(Colors/Styles/Templates/DataTemplates)
  - `StartupUri` 또는 `Startup` 이벤트로 시작 창 제어
  - `ShutdownMode` 적절히 선택
  - 전역 예외 처리 핸들링 구성

- **MainWindow.xaml**
  - `Window.Resources`에 창 전용 리소스
  - `CommandBindings/InputBindings`로 단축키/명령
  - `ContentControl + DataTemplate`로 뷰-뷰모델 매핑
  - `Grid` 기반 레이아웃, 메뉴/툴바/상태바 구성
