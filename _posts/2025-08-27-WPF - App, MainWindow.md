---
layout: post
title: WPF - App, MainWindow
date: 2025-08-27 20:25:23 +0900
category: WPF
---
# App.xaml / MainWindow.xaml: WPF 애플리케이션의 뼈대 이해하기

## WPF 애플리케이션의 구조적 기반

WPF 애플리케이션에서 `App.xaml`과 `MainWindow.xaml`은 전체 애플리케이션의 구조적 기반을 형성합니다. 이 두 파일은 각각 애플리케이션 수명 주기와 사용자 인터페이스의 시작점으로서 중요한 역할을 합니다. 이들이 어떻게 상호작용하며 전체 시스템을 구성하는지 이해하는 것은 효과적인 WPF 애플리케이션 개발의 첫걸음입니다.

---

## App.xaml: 애플리케이션의 심장

### App.xaml의 포괄적인 역할

`App.xaml`은 WPF 애플리케이션의 진입점으로서 단순히 시작 창을 지정하는 것 이상의 다양한 기능을 수행합니다:

```xml
<!-- App.xaml의 전형적인 구조 -->
<Application x:Class="InventorySystem.App"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             Startup="OnApplicationStartup"
             Exit="OnApplicationExit"
             SessionEnding="OnSessionEnding"
             DispatcherUnhandledException="OnUnhandledException"
             ShutdownMode="OnMainWindowClose">
    
    <!-- 애플리케이션 수준의 리소스 정의 -->
    <Application.Resources>
        <ResourceDictionary>
            <!-- 병합된 리소스 사전들 -->
            <ResourceDictionary.MergedDictionaries>
                <!-- 색상 테마 -->
                <ResourceDictionary Source="Resources/Themes/ColorPalette.xaml"/>
                <!-- 공통 스타일 -->
                <ResourceDictionary Source="Resources/Themes/CommonStyles.xaml"/>
                <!-- 아이콘 리소스 -->
                <ResourceDictionary Source="Resources/Icons/IconResources.xaml"/>
            </ResourceDictionary.MergedDictionaries>
            
            <!-- 전역적으로 사용되는 값 변환기 -->
            <converters:BooleanToVisibilityConverter x:Key="BooleanToVisibility"/>
            <converters:InverseBooleanConverter x:Key="InverseBoolean"/>
            
            <!-- 애플리케이션 설정 -->
            <system:Double x:Key="DefaultFontSize">14</system:Double>
            <system:Double x:Key="LargeFontSize">16</system:Double>
        </ResourceDictionary>
    </Application.Resources>
</Application>
```

### App.xaml.cs: 애플리케이션 로직의 구현

`App.xaml`의 코드 비하인드 파일은 애플리케이션의 시작부터 종료까지의 전체 수명 주기를 관리합니다:

```csharp
namespace InventorySystem
{
    public partial class App : Application
    {
        // 의존성 주입 컨테이너 (실제 프로젝트에서는 DI 프레임워크 사용)
        private IServiceProvider _serviceProvider;
        
        protected override async void OnStartup(StartupEventArgs e)
        {
            base.OnStartup(e);
            
            // 설정 로드
            var settings = await LoadApplicationSettingsAsync();
            
            // 데이터베이스 연결 초기화
            await InitializeDatabaseAsync();
            
            // 의존성 주입 컨테이너 구성
            _serviceProvider = ConfigureServices();
            
            // 메인 윈도우 생성 및 표시
            var mainWindow = _serviceProvider.GetRequiredService<MainWindow>();
            mainWindow.Show();
            
            // 시작 화면 표시 (선택적)
            await ShowSplashScreenAsync();
            
            // 초기 데이터 로드
            await PreloadEssentialDataAsync();
        }
        
        private IServiceProvider ConfigureServices()
        {
            var services = new ServiceCollection();
            
            // 뷰모델 등록
            services.AddSingleton<MainViewModel>();
            services.AddTransient<ProductViewModel>();
            services.AddTransient<CustomerViewModel>();
            
            // 서비스 등록
            services.AddSingleton<IDataService, SqlDataService>();
            services.AddSingleton<ILoggingService, FileLoggingService>();
            services.AddSingleton<IThemeService, ThemeService>();
            
            // 윈도우 등록
            services.AddSingleton<MainWindow>();
            
            return services.BuildServiceProvider();
        }
        
        protected override void OnExit(ExitEventArgs e)
        {
            // 애플리케이션 종료 시 정리 작업
            SaveUserPreferences();
            CloseDatabaseConnections();
            CleanupTemporaryFiles();
            
            base.OnExit(e);
        }
        
        private void OnUnhandledException(object sender, DispatcherUnhandledExceptionEventArgs e)
        {
            // 처리되지 않은 예외 처리
            var exception = e.Exception;
            LogCriticalError(exception);
            
            // 사용자에게 친숙한 오류 메시지 표시
            MessageBox.Show(
                "죄송합니다. 예기치 않은 오류가 발생했습니다.\n" +
                "자세한 내용은 로그 파일을 확인해 주세요.",
                "시스템 오류",
                MessageBoxButton.OK,
                MessageBoxImage.Error);
            
            // 예외 처리 완료 (애플리케이션 종료 방지)
            e.Handled = true;
        }
    }
}
```

### 테마 관리와 리소스 동적 교체

실제 애플리케이션에서는 런타임에 테마를 변경할 수 있는 기능이 중요합니다:

```csharp
public class ThemeManager
{
    public static void SwitchTheme(Theme theme)
    {
        var app = Application.Current;
        var mergedDictionaries = app.Resources.MergedDictionaries;
        
        // 기존 테마 리소스 제거
        var currentTheme = mergedDictionaries
            .FirstOrDefault(d => d.Source?.OriginalString.Contains("Themes/") == true);
        
        if (currentTheme != null)
        {
            mergedDictionaries.Remove(currentTheme);
        }
        
        // 새 테마 리소스 추가
        string themePath = theme switch
        {
            Theme.Light => "Resources/Themes/LightTheme.xaml",
            Theme.Dark => "Resources/Themes/DarkTheme.xaml",
            Theme.HighContrast => "Resources/Themes/HighContrastTheme.xaml",
            _ => "Resources/Themes/LightTheme.xaml"
        };
        
        var newTheme = new ResourceDictionary
        {
            Source = new Uri(themePath, UriKind.Relative)
        };
        
        mergedDictionaries.Add(newTheme);
        
        // 테마 변경 이벤트 발생
        OnThemeChanged?.Invoke(null, theme);
    }
    
    public static event EventHandler<Theme> OnThemeChanged;
}

public enum Theme
{
    Light,
    Dark,
    HighContrast
}
```

### ShutdownMode의 전략적 선택

`ShutdownMode`는 애플리케이션의 종료 동작을 결정하는 중요한 설정입니다:

```xml
<!-- 다양한 ShutdownMode 옵션 -->
<Application ShutdownMode="OnMainWindowClose">  <!-- 기본 메인 창이 닫힐 때 종료 -->
<Application ShutdownMode="OnLastWindowClose">  <!-- 마지막 창이 닫힐 때 종료 -->
<Application ShutdownMode="OnExplicitShutdown"> <!-- 명시적 Shutdown 호출 시에만 종료 -->
```

각 모드의 사용 사례:
- **OnMainWindowClose**: 대부분의 단일 창 애플리케이션에 적합
- **OnLastWindowClose**: 다중 문서 인터페이스(MDI) 애플리케이션에 적합
- **OnExplicitShutdown**: 종료 조건이 복잡한 애플리케이션에 적합

---

## MainWindow.xaml: 사용자 경험의 시작점

### MainWindow의 구조적 설계

`MainWindow`는 사용자가 애플리케이션을 사용할 때 가장 먼저 접하게 되는 인터페이스입니다. 잘 설계된 MainWindow는 직관적인 내비게이션과 효율적인 작업 흐름을 제공해야 합니다:

```xml
<!-- MainWindow.xaml의 완전한 예시 -->
<Window x:Class="InventorySystem.Views.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:local="clr-namespace:InventorySystem"
        xmlns:controls="clr-namespace:InventorySystem.Controls"
        xmlns:iconPacks="http://metro.mahapps.com/winfx/xaml/iconpacks"
        
        Title="재고 관리 시스템"
        WindowStartupLocation="CenterScreen"
        Height="800" Width="1200"
        MinHeight="600" MinWidth="900"
        Icon="../Assets/AppIcon.ico"
        WindowState="Maximized"
        
        mc:Ignorable="d"
        d:DesignHeight="800" d:DesignWidth="1200"
        d:DataContext="{d:DesignInstance Type=viewModels:MainViewModel, IsDesignTimeCreatable=True}">
    
    <!-- 창 수준의 리소스 정의 -->
    <Window.Resources>
        <!-- 창 전용 스타일 -->
        <Style x:Key="NavigationButtonStyle" TargetType="Button" BasedOn="{StaticResource {x:Type Button}}">
            <Setter Property="Height" Value="40"/>
            <Setter Property="Margin" Value="0,2"/>
            <Setter Property="HorizontalContentAlignment" Value="Left"/>
            <Setter Property="Template">
                <Setter.Value>
                    <ControlTemplate TargetType="Button">
                        <Border Background="Transparent">
                            <ContentPresenter Content="{TemplateBinding Content}"
                                              HorizontalAlignment="Stretch"
                                              VerticalAlignment="Center"
                                              Margin="10,0"/>
                        </Border>
                        <ControlTemplate.Triggers>
                            <Trigger Property="IsMouseOver" Value="True">
                                <Setter Property="Background" Value="{DynamicResource PrimaryLightBrush}"/>
                            </Trigger>
                            <Trigger Property="IsPressed" Value="True">
                                <Setter Property="Background" Value="{DynamicResource PrimaryBrush}"/>
                                <Setter Property="Foreground" Value="White"/>
                            </Trigger>
                        </ControlTemplate.Triggers>
                    </ControlTemplate>
                </Setter.Value>
            </Setter>
        </Style>
    </Window.Resources>
    
    <!-- 창 수준의 명령 바인딩 -->
    <Window.CommandBindings>
        <CommandBinding Command="ApplicationCommands.Save"
                        CanExecute="SaveCommand_CanExecute"
                        Executed="SaveCommand_Executed"/>
        <CommandBinding Command="ApplicationCommands.Print"
                        CanExecute="PrintCommand_CanExecute"
                        Executed="PrintCommand_Executed"/>
        <CommandBinding Command="NavigationCommands.GoToPage"
                        CanExecute="NavigationCommand_CanExecute"
                        Executed="NavigationCommand_Executed"/>
    </Window.CommandBindings>
    
    <!-- 키보드 단축키 설정 -->
    <Window.InputBindings>
        <KeyBinding Command="ApplicationCommands.Save" Key="S" Modifiers="Control"/>
        <KeyBinding Command="ApplicationCommands.Print" Key="P" Modifiers="Control"/>
        <KeyBinding Command="ApplicationCommands.Find" Key="F" Modifiers="Control"/>
        <KeyBinding Command="ApplicationCommands.Undo" Key="Z" Modifiers="Control"/>
        <KeyBinding Command="ApplicationCommands.Redo" Key="Y" Modifiers="Control"/>
    </Window.InputBindings>
    
    <!-- 메인 레이아웃 -->
    <Grid>
        <Grid.ColumnDefinitions>
            <ColumnDefinition Width="Auto"/>    <!-- 내비게이션 패널 -->
            <ColumnDefinition Width="*"/>       <!-- 콘텐츠 영역 -->
            <ColumnDefinition Width="Auto"/>    <!-- 사이드 패널 (옵션) -->
        </Grid.ColumnDefinitions>
        
        <!-- 내비게이션 패널 -->
        <Border Grid.Column="0" 
                Background="{DynamicResource NavigationBackground}"
                BorderBrush="{DynamicResource BorderBrush}"
                BorderThickness="0,0,1,0">
            
            <StackPanel Margin="10">
                <!-- 애플리케이션 로고 -->
                <StackPanel Orientation="Horizontal" Margin="0,0,0,20">
                    <Image Source="../Assets/Logo.png" Width="32" Height="32"/>
                    <TextBlock Text="재고관리" 
                               FontSize="18" 
                               FontWeight="Bold"
                               VerticalAlignment="Center"
                               Margin="10,0,0,0"/>
                </StackPanel>
                
                <!-- 내비게이션 메뉴 -->
                <Button Content="대시보드" 
                        Style="{StaticResource NavigationButtonStyle}"
                        Command="{Binding NavigateToDashboardCommand}">
                    <Button.ContentTemplate>
                        <DataTemplate>
                            <StackPanel Orientation="Horizontal">
                                <iconPacks:PackIconMaterial Kind="ViewDashboard" Width="20"/>
                                <TextBlock Text="대시보드" Margin="10,0,0,0"/>
                            </StackPanel>
                        </DataTemplate>
                    </Button.ContentTemplate>
                </Button>
                
                <Button Content="제품 관리"
                        Style="{StaticResource NavigationButtonStyle}"
                        Command="{Binding NavigateToProductsCommand}">
                    <!-- 아이콘과 텍스트 결합 -->
                </Button>
                
                <Separator Margin="0,10"/>
                
                <!-- 시스템 메뉴 -->
                <Button Content="설정"
                        Style="{StaticResource NavigationButtonStyle}"
                        Command="{Binding NavigateToSettingsCommand}"/>
            </StackPanel>
        </Border>
        
        <!-- 메인 콘텐츠 영역 -->
        <Grid Grid.Column="1">
            <Grid.RowDefinitions>
                <RowDefinition Height="Auto"/>  <!-- 툴바 -->
                <RowDefinition Height="*"/>     <!-- 동적 콘텐츠 -->
                <RowDefinition Height="Auto"/>  <!-- 상태 표시줄 -->
            </Grid.RowDefinitions>
            
            <!-- 툴바 -->
            <ToolBar Grid.Row="0" Background="Transparent">
                <Button Command="ApplicationCommands.Save" ToolTip="저장 (Ctrl+S)">
                    <StackPanel Orientation="Horizontal">
                        <iconPacks:PackIconMaterial Kind="ContentSave"/>
                        <TextBlock Text=" 저장" Margin="5,0,0,0"/>
                    </StackPanel>
                </Button>
                <Separator/>
                <Button Command="ApplicationCommands.Print" ToolTip="인쇄 (Ctrl+P)">
                    <!-- 아이콘 -->
                </Button>
            </ToolBar>
            
            <!-- 동적 콘텐츠 영역 -->
            <ContentControl Grid.Row="1"
                            Content="{Binding CurrentViewModel}"
                            Margin="10">
                <ContentControl.ContentTemplate>
                    <DataTemplate>
                        <!-- DataTemplate은 App.xaml의 리소스에 정의됨 -->
                        <ContentPresenter Content="{Binding}"/>
                    </DataTemplate>
                </ContentControl.ContentTemplate>
            </ContentControl>
            
            <!-- 상태 표시줄 -->
            <StatusBar Grid.Row="2" Background="{DynamicResource StatusBarBackground}">
                <StatusBarItem>
                    <TextBlock Text="{Binding StatusMessage}"/>
                </StatusBarItem>
                <Separator/>
                <StatusBarItem HorizontalAlignment="Right">
                    <StackPanel Orientation="Horizontal">
                        <TextBlock Text="사용자: "/>
                        <TextBlock Text="{Binding CurrentUser.Name}" FontWeight="Bold"/>
                    </StackPanel>
                </StatusBarItem>
                <Separator/>
                <StatusBarItem HorizontalAlignment="Right">
                    <TextBlock Text="{Binding CurrentTime, StringFormat='{}{0:yyyy-MM-dd HH:mm}'}"/>
                </StatusBarItem>
            </StatusBar>
        </Grid>
        
        <!-- 사이드 패널 (선택적) -->
        <controls:SidePanel Grid.Column="2" 
                            Visibility="{Binding IsSidePanelVisible, Converter={StaticResource BooleanToVisibility}}"
                            Width="300"/>
    </Grid>
</Window>
```

### MainWindow.xaml.cs: 뷰와 뷰모델의 연결

MainWindow의 코드 비하인드는 뷰와 뷰모델을 연결하고 창 수준의 이벤트를 처리합니다:

```csharp
namespace InventorySystem.Views
{
    public partial class MainWindow : Window
    {
        private readonly MainViewModel _viewModel;
        private readonly IThemeService _themeService;
        
        // 의존성 주입을 통한 생성자
        public MainWindow(MainViewModel viewModel, IThemeService themeService)
        {
            InitializeComponent();
            
            _viewModel = viewModel;
            _themeService = themeService;
            
            // 데이터 컨텍스트 설정
            DataContext = _viewModel;
            
            // 창 이벤트 구독
            Loaded += OnWindowLoaded;
            Closing += OnWindowClosing;
            StateChanged += OnWindowStateChanged;
            
            // 테마 변경 이벤트 구독
            ThemeManager.OnThemeChanged += OnThemeChanged;
            
            // 디자인 모드 확인
            if (DesignerProperties.GetIsInDesignMode(this))
            {
                // 디자인 타임 데이터 설정
                SetupDesignTimeData();
            }
        }
        
        private void OnWindowLoaded(object sender, RoutedEventArgs e)
        {
            // 창 로드 시 초기화 작업
            RestoreWindowState();
            InitializeAsync();
        }
        
        private async void InitializeAsync()
        {
            try
            {
                // 비동기 초기화 작업
                await _viewModel.InitializeAsync();
                
                // 초기 내비게이션
                await _viewModel.NavigateToDashboardAsync();
            }
            catch (Exception ex)
            {
                MessageBox.Show($"초기화 중 오류가 발생했습니다: {ex.Message}", 
                                "오류", 
                                MessageBoxButton.OK, 
                                MessageBoxImage.Error);
            }
        }
        
        private void OnWindowClosing(object sender, CancelEventArgs e)
        {
            // 변경 사항 저장 확인
            if (_viewModel.HasUnsavedChanges)
            {
                var result = MessageBox.Show(
                    "저장되지 않은 변경 사항이 있습니다. 정말 종료하시겠습니까?",
                    "확인",
                    MessageBoxButton.YesNo,
                    MessageBoxImage.Question);
                
                if (result == MessageBoxResult.No)
                {
                    e.Cancel = true;
                    return;
                }
            }
            
            // 창 상태 저장
            SaveWindowState();
            
            // 리소스 정리
            _themeService?.Dispose();
        }
        
        private void SaveCommand_CanExecute(object sender, CanExecuteRoutedEventArgs e)
        {
            e.CanExecute = _viewModel?.CanSave ?? false;
        }
        
        private void SaveCommand_Executed(object sender, ExecutedRoutedEventArgs e)
        {
            _viewModel?.SaveCommand.Execute(null);
        }
        
        private void OnThemeChanged(object sender, Theme theme)
        {
            // 테마 변경에 따른 UI 업데이트
            UpdateWindowChromeForTheme(theme);
        }
        
        private void UpdateWindowChromeForTheme(Theme theme)
        {
            // 테마에 맞는 창 테두리 색상 업데이트
            switch (theme)
            {
                case Theme.Dark:
                    BorderBrush = Brushes.DarkGray;
                    break;
                case Theme.Light:
                    BorderBrush = Brushes.LightGray;
                    break;
                case Theme.HighContrast:
                    BorderBrush = Brushes.Black;
                    break;
            }
        }
        
        private void SaveWindowState()
        {
            // 창 위치, 크기, 상태 저장
            var settings = Properties.Settings.Default;
            settings.WindowLeft = Left;
            settings.WindowTop = Top;
            settings.WindowWidth = Width;
            settings.WindowHeight = Height;
            settings.WindowState = WindowState.ToString();
            settings.Save();
        }
        
        private void RestoreWindowState()
        {
            // 저장된 창 상태 복원
            var settings = Properties.Settings.Default;
            
            if (settings.WindowLeft >= 0 && settings.WindowTop >= 0)
            {
                Left = settings.WindowLeft;
                Top = settings.WindowTop;
            }
            
            if (settings.WindowWidth > 0 && settings.WindowHeight > 0)
            {
                Width = settings.WindowWidth;
                Height = settings.WindowHeight;
            }
            
            if (Enum.TryParse(settings.WindowState, out WindowState state))
            {
                WindowState = state;
            }
        }
    }
}
```

### 뷰 전환 패턴의 구현

MainWindow에서 다양한 뷰 사이를 전환하는 패턴은 중요한 설계 결정입니다:

```csharp
// MainViewModel.cs에서의 뷰 전환 구현
public class MainViewModel : ViewModelBase
{
    private ViewModelBase _currentViewModel;
    private readonly INavigationService _navigationService;
    
    public ViewModelBase CurrentViewModel
    {
        get => _currentViewModel;
        set => SetProperty(ref _currentViewModel, value);
    }
    
    public ICommand NavigateToDashboardCommand { get; }
    public ICommand NavigateToProductsCommand { get; }
    public ICommand NavigateToSettingsCommand { get; }
    
    public MainViewModel(INavigationService navigationService)
    {
        _navigationService = navigationService;
        
        // 명령 초기화
        NavigateToDashboardCommand = new RelayCommand(NavigateToDashboard);
        NavigateToProductsCommand = new RelayCommand(NavigateToProducts);
        NavigateToSettingsCommand = new RelayCommand(NavigateToSettings);
        
        // 내비게이션 이벤트 구독
        _navigationService.NavigationRequested += OnNavigationRequested;
    }
    
    private void NavigateToDashboard()
    {
        _navigationService.NavigateTo<DashboardViewModel>();
    }
    
    private void NavigateToProducts()
    {
        _navigationService.NavigateTo<ProductListViewModel>();
    }
    
    private void NavigateToSettings()
    {
        _navigationService.NavigateTo<SettingsViewModel>();
    }
    
    private void OnNavigationRequested(object sender, ViewModelBase viewModel)
    {
        CurrentViewModel = viewModel;
        
        // 내비게이션 히스토리 관리
        AddToNavigationHistory(viewModel);
        
        // 뷰 전환 애니메이션 트리거
        OnViewChanged?.Invoke(this, EventArgs.Empty);
    }
    
    public event EventHandler OnViewChanged;
}
```

---

## App.xaml과 MainWindow.xaml의 통합 아키텍처

### 종합적인 애플리케이션 시작 흐름

실제 프로젝트에서 App.xaml과 MainWindow.xaml은 다음과 같은 흐름으로 통합됩니다:

```csharp
// App.xaml.cs의 최종 구현
public partial class App : Application
{
    private IHost _host;
    
    protected override async void OnStartup(StartupEventArgs e)
    {
        base.OnStartup(e);
        
        try
        {
            // 호스트 빌더 생성
            _host = Host.CreateDefaultBuilder()
                .ConfigureAppConfiguration((context, config) =>
                {
                    // 구성 설정
                    config.AddJsonFile("appsettings.json", optional: true);
                    config.AddEnvironmentVariables();
                    
                    if (e.Args.Contains("--development"))
                    {
                        config.AddUserSecrets<App>();
                    }
                })
                .ConfigureServices((context, services) =>
                {
                    // 서비스 등록
                    ConfigureServices(context.Configuration, services);
                })
                .Build();
            
            // 호스트 시작
            await _host.StartAsync();
            
            // 메인 윈도우 표시
            var mainWindow = _host.Services.GetRequiredService<MainWindow>();
            mainWindow.Show();
            
            // 시작 작업 실행
            await RunStartupTasksAsync();
        }
        catch (Exception ex)
        {
            // 시작 실패 처리
            HandleStartupFailure(ex);
            Shutdown(1); // 오류 코드와 함께 종료
        }
    }
    
    protected override async void OnExit(ExitEventArgs e)
    {
        // 호스트 정지
        if (_host != null)
        {
            using (_host)
            {
                await _host.StopAsync(TimeSpan.FromSeconds(5));
            }
        }
        
        base.OnExit(e);
    }
    
    private void ConfigureServices(IConfiguration configuration, IServiceCollection services)
    {
        // 뷰모델 등록
        services.AddSingleton<MainViewModel>();
        services.AddTransient<DashboardViewModel>();
        services.AddTransient<ProductViewModel>();
        services.AddTransient<SettingsViewModel>();
        
        // 서비스 등록
        services.AddSingleton<IDataService, EntityFrameworkDataService>();
        services.AddSingleton<IAuthenticationService, WindowsAuthenticationService>();
        services.AddSingleton<IThemeService, ThemeService>();
        services.AddSingleton<INavigationService, NavigationService>();
        
        // 윈도우 등록
        services.AddSingleton<MainWindow>();
        
        // 옵션 패턴
        services.Configure<AppSettings>(configuration.GetSection("AppSettings"));
        
        // HTTP 클라이언트
        services.AddHttpClient("ApiClient", client =>
        {
            client.BaseAddress = new Uri(configuration["Api:BaseUrl"]);
            client.DefaultRequestHeaders.Add("Accept", "application/json");
        });
    }
}
```

### 리소스 관리의 계층적 구조

효과적인 리소스 관리를 위해 계층적 구조를 사용합니다:

```
Resources/
├── Themes/
│   ├── ColorPalette.xaml          # 색상 정의
│   ├── CommonStyles.xaml          # 기본 스타일
│   ├── ControlTemplates.xaml      # 컨트롤 템플릿
│   ├── DataTemplates.xaml         # 데이터 템플릿
│   ├── LightTheme.xaml           # 밝은 테마
│   └── DarkTheme.xaml            # 어두운 테마
├── Icons/
│   ├── IconResources.xaml        # 아이콘 리소스
│   └── CustomIcons.xaml          # 사용자 정의 아이콘
├── Brushes/
│   └── GradientBrushes.xaml      # 그라데이션 브러시
└── Converters/
    └── ValueConverters.xaml      # 값 변환기
```

각 리소스 파일은 특정 목적에 맞게 구성됩니다:

```xml
<!-- ColorPalette.xaml -->
<ResourceDictionary xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation">
    <!-- 기본 색상 팔레트 -->
    <Color x:Key="PrimaryColor">#0078D4</Color>
    <Color x:Key="SecondaryColor">#605E5C</Color>
    <Color x:Key="SuccessColor">#107C10</Color>
    <Color x:Key="WarningColor">#F7630C</Color>
    <Color x:Key="ErrorColor">#D13438</Color>
    
    <!-- 브러시 정의 -->
    <SolidColorBrush x:Key="PrimaryBrush" Color="{StaticResource PrimaryColor}"/>
    <LinearGradientBrush x:Key="PrimaryGradientBrush" StartPoint="0,0" EndPoint="1,1">
        <GradientStop Color="{StaticResource PrimaryColor}" Offset="0"/>
        <GradientStop Color="#005A9E" Offset="1"/>
    </LinearGradientBrush>
</ResourceDictionary>
```

---

## 결론

WPF 애플리케이션에서 `App.xaml`과 `MainWindow.xaml`은 단순한 시작점을 넘어서 전체 애플리케이션 아키텍처의 기반을 형성합니다. 이들의 효과적인 설계와 구현은 애플리케이션의 유지보수성, 확장성, 사용자 경험에 직접적인 영향을 미칩니다.

### 핵심 통찰

1. **App.xaml은 애플리케이션의 생명주기를 관리하는 통제 센터입니다**. 전역 리소스 관리, 예외 처리, 테마 스위칭, 의존성 주입 구성 등의 책임을 집중적으로 담당합니다. 잘 설계된 App 클래스는 애플리케이션의 안정성과 일관성을 보장합니다.

2. **MainWindow.xaml은 사용자 경험의 중심 허브입니다**. 내비게이션 구조, 레이아웃, 명령 시스템, 상태 관리 등을 통합하여 사용자가 애플리케이션과 상호작용하는 주요 공간을 제공합니다.

3. **두 파일의 협력이 성공의 열쇠입니다**. App.xaml이 제공하는 전역 서비스와 리소스를 MainWindow.xaml이 효과적으로 활용할 때 비로소 일관되고 안정적인 사용자 경험이 만들어집니다.

### 실무적 권장사항

- **의존성 주입을 적극 활용하세요**: App.xaml.cs에서 서비스 컨테이너를 구성하고 MainWindow에 필요한 의존성을 주입함으로써 테스트 용이성과 유지보수성을 크게 향상시킬 수 있습니다.

- **리소스는 계층적으로 구성하세요**: App.xaml에는 애플리케이션 전반에 걸친 공통 리소스를, MainWindow.xaml에는 창 특화 리소스를 배치하여 관리의 복잡성을 줄이세요.

- **MVVM 패턴을 일관되게 적용하세요**: MainWindow의 DataContext를 ViewModel로 설정하고, 명령과 데이터 바인딩을 통해 뷰와 로직을 분리하세요.

- **예외 처리와 복원력을 강화하세요**: App.xaml에서 전역 예외 처리기를 구현하고, MainWindow에서는 사용자 친화적인 오류 메시지와 복구 메커니즘을 제공하세요.

- **사용자 경험을 고려한 설계를 하세요**: 창 상태 저장/복원, 테마 지원, 접근성 기능, 키보드 단축키 등을 MainWindow에 구현하여 전문적인 애플리케이션의 느낌을 제공하세요.

WPF의 강력함은 App.xaml과 MainWindow.xaml의 유기적인 협력에서 비롯됩니다. 이들을 단순한 템플릿 파일이 아니라 애플리케이션 아키텍처의 핵심 구성 요소로 이해하고 설계할 때, 견고하고 사용자 친화적인 데스크톱 애플리케이션을 구축할 수 있습니다. 이러한 기반 위에 MVVM 패턴, 모듈화된 리소스 관리, 현대적인 의존성 주입 등을 접목하면 엔터프라이즈 수준의 WPF 애플리케이션 개발이 가능해집니다.