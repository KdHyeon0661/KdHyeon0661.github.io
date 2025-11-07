---
layout: post
title: Avalonia - DI(Dependency Injection)
date: 2025-02-01 20:20:23 +0900
category: Avalonia
---
# Avalonia MVVM에서의 Dependency Injection 구조, 베스트 프랙티스 총정리

초안의 핵심(서비스/뷰모델 DI, 수명 주기, 생성자 주입)을 그대로 유지하면서, **조금 더 실전적이고 확장 가능한 DI 설계**로 넓힌다.  
본 문서는 다음을 모두 다룬다.

- DI 구성(두 가지 방식)
  - [A] 단순 `ServiceCollection` (App에서 직접 구성)
  - [B] .NET Generic Host(`Host.CreateDefaultBuilder`) 연동 — 구성/로깅/옵션을 한 번에
- 수명 주기 전략(Singleton/Transient + “유사 Scoped(윈도우/페이지 단위)”)
- ViewModel Factory / `Func<T>` / `IServiceScopeFactory` 활용
- View 연결: ViewLocator/`DataTemplates`와 DI
- HTTP/파일/설정 서비스, MessageBus, NavigationService, DialogService, ThemeService 등 **서비스 레이어 표준화**
- 테스트/목 객체 주입 전략, 디자인 타임 데이터
- 다중 윈도우/모듈/플러그인 아키텍처로 확장

---

## 0. 예시 솔루션 구조

```
MyAvaloniaApp/
├─ App.axaml
├─ App.axaml.cs
├─ Program.cs
├─ Infrastructure/               # DI·Host·Options·Logging 등 인프라
│  ├─ Bootstrapper.cs
│  ├─ ViewLocator.cs
│  ├─ Extensions/
│  │  ├─ ServiceCollectionExtensions.cs
│  │  └─ HostExtensions.cs
│  └─ Options/
│     └─ AppOptions.cs
├─ Services/
│  ├─ Abstractions/
│  │  ├─ IAuthService.cs
│  │  ├─ IFileDialogService.cs
│  │  ├─ INavigationService.cs
│  │  ├─ IMessageBus.cs
│  │  ├─ IThemeService.cs
│  │  └─ IJsonStore.cs
│  ├─ Implementations/
│  │  ├─ AuthService.cs
│  │  ├─ FileDialogService.cs
│  │  ├─ NavigationService.cs
│  │  ├─ MessageBus.cs
│  │  ├─ ThemeService.cs
│  │  └─ JsonStore.cs
│  └─ Http/
│     ├─ ApiClient.cs
│     └─ ApiClientOptions.cs
├─ ViewModels/
│  ├─ MainViewModel.cs
│  ├─ LoginViewModel.cs
│  ├─ DashboardViewModel.cs
│  └─ SettingsViewModel.cs
├─ Views/
│  ├─ MainView.axaml
│  ├─ MainView.axaml.cs
│  ├─ LoginView.axaml
│  ├─ LoginView.axaml.cs
│  ├─ DashboardView.axaml
│  ├─ DashboardView.axaml.cs
│  ├─ SettingsView.axaml
│  └─ SettingsView.axaml.cs
├─ Themes/
│  ├─ LightTheme.axaml
│  └─ DarkTheme.axaml
└─ Tests/
   ├─ MyAvaloniaApp.Tests.csproj
   └─ LoginViewModelTests.cs
```

---

## 1. 서비스 인터페이스와 구현

### 1.1 인증 서비스

```csharp
// Services/Abstractions/IAuthService.cs
using System.Threading.Tasks;

namespace MyAvaloniaApp.Services.Abstractions;

public interface IAuthService
{
    Task<bool> LoginAsync(string username, string password);
    Task LogoutAsync();
    bool IsAuthenticated { get; }
    string? CurrentUser { get; }
}
```

```csharp
// Services/Implementations/AuthService.cs
using System.Threading.Tasks;
using MyAvaloniaApp.Services.Abstractions;

namespace MyAvaloniaApp.Services.Implementations;

public sealed class AuthService : IAuthService
{
    public bool IsAuthenticated { get; private set; }
    public string? CurrentUser { get; private set; }

    public Task<bool> LoginAsync(string username, string password)
    {
        // 실제 환경에서는 API 호출/토큰 저장 등을 수행
        IsAuthenticated = (username == "admin" && password == "1234");
        CurrentUser = IsAuthenticated ? username : null;
        return Task.FromResult(IsAuthenticated);
    }

    public Task LogoutAsync()
    {
        IsAuthenticated = false;
        CurrentUser = null;
        return Task.CompletedTask;
    }
}
```

### 1.2 파일 다이얼로그 서비스

```csharp
// Services/Abstractions/IFileDialogService.cs
using System.Threading.Tasks;

namespace MyAvaloniaApp.Services.Abstractions;

public interface IFileDialogService
{
    Task<string?> ShowOpenFileAsync(string title, string[]? filterExtensions = null);
    Task<string?> ShowSaveFileAsync(string title, string defaultName = "data.json");
}
```

```csharp
// Services/Implementations/FileDialogService.cs
using System.Threading.Tasks;
using Avalonia.Controls;
using MyAvaloniaApp.Services.Abstractions;

namespace MyAvaloniaApp.Services.Implementations;

public sealed class FileDialogService : IFileDialogService
{
    private readonly Window? _owner;

    public FileDialogService()
    {
        // MainWindow가 뜬 후에는 NavigationService 등에서 Owner를 주입해 줄 수도 있다
        _owner = (Avalonia.Application.Current?.ApplicationLifetime as IClassicDesktopStyleApplicationLifetime)?.MainWindow;
    }

    public async Task<string?> ShowOpenFileAsync(string title, string[]? filterExtensions = null)
    {
        var dialog = new OpenFileDialog { Title = title, AllowMultiple = false };
        if (filterExtensions is { Length: > 0 })
        {
            dialog.Filters?.Add(new FileDialogFilter { Name = "Files", Extensions = filterExtensions.ToList() });
        }
        var result = await dialog.ShowAsync(_owner);
        return result?.FirstOrDefault();
    }

    public async Task<string?> ShowSaveFileAsync(string title, string defaultName = "data.json")
    {
        var dialog = new SaveFileDialog { Title = title, InitialFileName = defaultName };
        return await dialog.ShowAsync(_owner);
    }
}
```

### 1.3 NavigationService

```csharp
// Services/Abstractions/INavigationService.cs
using System;

namespace MyAvaloniaApp.Services.Abstractions;

public interface INavigationService
{
    event Action<object>? Navigated;
    void NavigateTo(object viewModel);
}
```

```csharp
// Services/Implementations/NavigationService.cs
using System;
using MyAvaloniaApp.Services.Abstractions;

namespace MyAvaloniaApp.Services.Implementations;

public sealed class NavigationService : INavigationService
{
    public event Action<object>? Navigated;
    public void NavigateTo(object viewModel) => Navigated?.Invoke(viewModel);
}
```

### 1.4 MessageBus (간단 구현)

```csharp
// Services/Abstractions/IMessageBus.cs
using System;

namespace MyAvaloniaApp.Services.Abstractions;

public interface IMessageBus
{
    void Publish<T>(T message);
    IDisposable Subscribe<T>(Action<T> handler);
}
```

```csharp
// Services/Implementations/MessageBus.cs
using System;
using System.Collections.Generic;
using MyAvaloniaApp.Services.Abstractions;

namespace MyAvaloniaApp.Services.Implementations;

public sealed class MessageBus : IMessageBus
{
    private readonly Dictionary<Type, List<Delegate>> _routes = new();

    public void Publish<T>(T message)
    {
        if (_routes.TryGetValue(typeof(T), out var handlers))
        {
            foreach (var h in handlers.ToArray())
                (h as Action<T>)?.Invoke(message);
        }
    }

    public IDisposable Subscribe<T>(Action<T> handler)
    {
        if (!_routes.TryGetValue(typeof(T), out var handlers))
            _routes[typeof(T)] = handlers = new List<Delegate>();
        handlers.Add(handler);
        return new Unsubscriber(() => handlers.Remove(handler));
    }

    private sealed class Unsubscriber : IDisposable
    {
        private readonly Action _onDispose;
        public Unsubscriber(Action onDispose) => _onDispose = onDispose;
        public void Dispose() => _onDispose();
    }
}
```

### 1.5 ThemeService

```csharp
// Services/Abstractions/IThemeService.cs
namespace MyAvaloniaApp.Services.Abstractions;

public interface IThemeService
{
    bool IsDark { get; }
    void ApplyDark();
    void ApplyLight();
}
```

```csharp
// Services/Implementations/ThemeService.cs
using System;
using Avalonia.Markup.Xaml.Styling;
using MyAvaloniaApp.Services.Abstractions;

namespace MyAvaloniaApp.Services.Implementations;

public sealed class ThemeService : IThemeService
{
    public bool IsDark { get; private set; }

    public void ApplyDark()  => Apply("avares://MyAvaloniaApp/Themes/DarkTheme.axaml", true);
    public void ApplyLight() => Apply("avares://MyAvaloniaApp/Themes/LightTheme.axaml", false);

    private void Apply(string uri, bool isDark)
    {
        var app = Avalonia.Application.Current;
        if (app is null) return;

        var remove = app.Styles.FirstOrDefault(s =>
            s is ResourceInclude ri && ri.Source?.ToString()?.Contains("Theme") == true);
        if (remove is not null) app.Styles.Remove(remove);

        var include = new ResourceInclude(new Uri(uri)) { Source = new Uri(uri) };
        app.Styles.Add(include);
        IsDark = isDark;
    }
}
```

### 1.6 간단한 JSON 저장소

```csharp
// Services/Abstractions/IJsonStore.cs
using System.Threading.Tasks;

namespace MyAvaloniaApp.Services.Abstractions;

public interface IJsonStore
{
    Task SaveAsync<T>(string path, T data);
    Task<T?> LoadAsync<T>(string path);
}
```

```csharp
// Services/Implementations/JsonStore.cs
using System.IO;
using System.Text.Json;
using System.Threading.Tasks;
using MyAvaloniaApp.Services.Abstractions;

namespace MyAvaloniaApp.Services.Implementations;

public sealed class JsonStore : IJsonStore
{
    private static readonly JsonSerializerOptions Opt = new() { WriteIndented = true };

    public async Task SaveAsync<T>(string path, T data)
    {
        Directory.CreateDirectory(Path.GetDirectoryName(path)!);
        var json = JsonSerializer.Serialize(data, Opt);
        await File.WriteAllTextAsync(path, json);
    }

    public async Task<T?> LoadAsync<T>(string path)
    {
        if (!File.Exists(path)) return default;
        var json = await File.ReadAllTextAsync(path);
        return JsonSerializer.Deserialize<T>(json);
    }
}
```

---

## 2. ViewModel — 생성자 주입

### 2.1 LoginViewModel

```csharp
// ViewModels/LoginViewModel.cs
using System.Reactive;
using System.Threading.Tasks;
using ReactiveUI;
using MyAvaloniaApp.Services.Abstractions;

namespace MyAvaloniaApp.ViewModels;

public sealed class LoginViewModel : ReactiveObject
{
    private readonly IAuthService _auth;
    private readonly INavigationService _nav;

    public LoginViewModel(IAuthService auth, INavigationService nav)
    {
        _auth = auth;
        _nav = nav;

        LoginCommand = ReactiveCommand.CreateFromTask(ExecuteLogin, this.WhenAnyValue(
            x => x.Username, x => x.Password, (u, p) => !string.IsNullOrWhiteSpace(u) && !string.IsNullOrWhiteSpace(p)));
    }

    private string _username = "";
    public string Username
    {
        get => _username;
        set => this.RaiseAndSetIfChanged(ref _username, value);
    }

    private string _password = "";
    public string Password
    {
        get => _password;
        set => this.RaiseAndSetIfChanged(ref _password, value);
    }

    public ReactiveCommand<Unit, bool> LoginCommand { get; }

    private async Task<bool> ExecuteLogin()
    {
        var ok = await _auth.LoginAsync(Username, Password);
        if (ok)
        {
            _nav.NavigateTo(new DashboardViewModel(_auth, _nav));
        }
        return ok;
    }
}
```

### 2.2 DashboardViewModel

```csharp
// ViewModels/DashboardViewModel.cs
using ReactiveUI;
using MyAvaloniaApp.Services.Abstractions;

namespace MyAvaloniaApp.ViewModels;

public sealed class DashboardViewModel : ReactiveObject
{
    private readonly IAuthService _auth;
    private readonly INavigationService _nav;

    public DashboardViewModel(IAuthService auth, INavigationService nav)
    {
        _auth = auth;
        _nav = nav;

        _title = $"환영합니다, {_auth.CurrentUser ?? "Guest"}";
    }

    private string _title;
    public string Title
    {
        get => _title;
        set => this.RaiseAndSetIfChanged(ref _title, value);
    }
}
```

### 2.3 MainViewModel — Navigation 바인딩

```csharp
// ViewModels/MainViewModel.cs
using ReactiveUI;
using MyAvaloniaApp.Services.Abstractions;

namespace MyAvaloniaApp.ViewModels;

public sealed class MainViewModel : ReactiveObject
{
    private object? _current;
    public object? Current
    {
        get => _current;
        set => this.RaiseAndSetIfChanged(ref _current, value);
    }

    public MainViewModel(INavigationService nav, LoginViewModel loginVm)
    {
        // 초기 페이지
        Current = loginVm;

        // 페이지 전환 이벤트 수신
        nav.Navigated += vm => Current = vm;
    }
}
```

> 초안에서의 “View 내부에서 ViewModel 생성 금지” 원칙을 지키기 위해, **항상 생성자 주입**으로 ViewModel을 제공한다.

---

## 3. DI 구성 — 두 가지 방식

### 3.1 [A] App에서 간단 구성 (`ServiceCollection` 직접)

```csharp
// App.axaml.cs
using Avalonia;
using Avalonia.Controls.ApplicationLifetimes;
using Avalonia.Markup.Xaml;
using Microsoft.Extensions.DependencyInjection;
using MyAvaloniaApp.ViewModels;
using MyAvaloniaApp.Views;
using MyAvaloniaApp.Services.Abstractions;
using MyAvaloniaApp.Services.Implementations;

namespace MyAvaloniaApp;

public partial class App : Application
{
    public static ServiceProvider Services = default!;

    public override void Initialize()
        => AvaloniaXamlLoader.Load(this);

    public override void OnFrameworkInitializationCompleted()
    {
        var sc = new ServiceCollection();

        // Services
        sc.AddSingleton<IAuthService, AuthService>();
        sc.AddSingleton<INavigationService, NavigationService>();
        sc.AddSingleton<IMessageBus, MessageBus>();
        sc.AddSingleton<IFileDialogService, FileDialogService>();
        sc.AddSingleton<IThemeService, ThemeService>();
        sc.AddSingleton<IJsonStore, JsonStore>();

        // ViewModels
        sc.AddSingleton<MainViewModel>();
        sc.AddTransient<LoginViewModel>();
        sc.AddTransient<DashboardViewModel>();

        // ViewLocator(선택) DI가 필요하면 등록
        sc.AddSingleton<Infrastructure.ViewLocator>();

        Services = sc.BuildServiceProvider();

        if (ApplicationLifetime is IClassicDesktopStyleApplicationLifetime desktop)
        {
            var main = new MainWindow
            {
                DataContext = Services.GetRequiredService<MainViewModel>()
            };
            desktop.MainWindow = main;
        }

        base.OnFrameworkInitializationCompleted();
    }
}
```

**장점**: 단순/직관.  
**단점**: 구성/로깅/환경변수/설정(AppSettings) 같은 “호스트 기능”은 직접 마련해야 한다.

---

### 3.2 [B] Generic Host 연동 — 구성/로깅/옵션/HttpClient까지 표준대로

```csharp
// Program.cs
using Avalonia;
using System;

namespace MyAvaloniaApp;

internal static class Program
{
    [STAThread]
    public static void Main(string[] args)
        => BuildAvaloniaApp().StartWithClassicDesktopLifetime(args);

    public static AppBuilder BuildAvaloniaApp()
        => AppBuilder.Configure<App>()
                     .UsePlatformDetect()
                     .LogToTrace();
}
```

```csharp
// Infrastructure/Bootstrapper.cs
using System;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;
using MyAvaloniaApp.Services.Abstractions;
using MyAvaloniaApp.Services.Implementations;
using MyAvaloniaApp.ViewModels;

namespace MyAvaloniaApp.Infrastructure;

public static class Bootstrapper
{
    public static IHost BuildHost()
    {
        // HostBuilder의 기본 구성을 활용(환경변수, appsettings.json, 로깅 등 활성화)
        var host = Host.CreateDefaultBuilder()
            .ConfigureAppConfiguration(cfg =>
            {
                // 필요 시 추가 구성 소스 등록
                // cfg.AddJsonFile("appsettings.local.json", optional: true);
            })
            .ConfigureServices((ctx, services) =>
            {
                // Options 바인딩 예시
                services.Configure<Options.AppOptions>(ctx.Configuration.GetSection("App"));

                // HttpClient/Typed Client(선택)
                services.AddHttpClient<Services.Http.ApiClient>(client =>
                {
                    // appsettings.json에서 ApiBaseUrl 불러와 설정할 수 있음
                    var apiBase = ctx.Configuration["Api:BaseUrl"];
                    if (!string.IsNullOrWhiteSpace(apiBase))
                        client.BaseAddress = new Uri(apiBase);
                });

                // Services
                services.AddSingleton<IAuthService, AuthService>();
                services.AddSingleton<INavigationService, NavigationService>();
                services.AddSingleton<IMessageBus, MessageBus>();
                services.AddSingleton<IFileDialogService, FileDialogService>();
                services.AddSingleton<IThemeService, ThemeService>();
                services.AddSingleton<IJsonStore, JsonStore>();

                // ViewModels
                services.AddSingleton<MainViewModel>();
                services.AddTransient<LoginViewModel>();
                services.AddTransient<DashboardViewModel>();

                // ViewLocator (선택)
                services.AddSingleton<ViewLocator>();
            })
            .ConfigureLogging(b =>
            {
                b.ClearProviders();
                b.AddDebug();
                b.AddConsole();
            })
            .Build();

        return host;
    }
}
```

```csharp
// App.axaml.cs (Host 통합 버전)
using Avalonia;
using Avalonia.Controls.ApplicationLifetimes;
using Avalonia.Markup.Xaml;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using MyAvaloniaApp.Infrastructure;
using MyAvaloniaApp.ViewModels;
using MyAvaloniaApp.Views;

namespace MyAvaloniaApp;

public partial class App : Application
{
    public static IHost Host { get; private set; } = default!;

    public override void Initialize()
        => AvaloniaXamlLoader.Load(this);

    public override void OnFrameworkInitializationCompleted()
    {
        Host = Bootstrapper.BuildHost();

        if (ApplicationLifetime is IClassicDesktopStyleApplicationLifetime desktop)
        {
            var mainVm = Host.Services.GetRequiredService<MainViewModel>();
            desktop.MainWindow = new MainWindow { DataContext = mainVm };
        }

        base.OnFrameworkInitializationCompleted();
    }
}
```

**장점**
- `appsettings.json` / 환경 변수 / 사용자 비밀 등 “구성”을 그대로 사용
- `IOptions<T>` / `IOptionsMonitor<T>` / `IOptionsSnapshot<T>` 패턴 적용
- `HttpClientFactory`, 로깅, 백그라운드 서비스 등 .NET 표준 기능 활용

**단점**: 진입 장벽이 약간 높음. 하지만 중대형 앱에서는 권장.

---

## 4. View 연결 — ViewLocator + DataTemplates

### 4.1 ViewLocator

```csharp
// Infrastructure/ViewLocator.cs
using Avalonia.Controls;
using Avalonia.Controls.Templates;
using MyAvaloniaApp.ViewModels;
using MyAvaloniaApp.Views;
using Microsoft.Extensions.DependencyInjection;

namespace MyAvaloniaApp.Infrastructure;

public sealed class ViewLocator : IDataTemplate
{
    public Control Build(object? data)
    {
        return data switch
        {
            LoginViewModel      => new LoginView(),
            DashboardViewModel  => new DashboardView(),
            SettingsViewModel   => new SettingsView(),
            MainViewModel       => new MainView(),
            _                   => new TextBlock { Text = "View Not Found" }
        };
    }

    public bool Match(object? data) => data is ViewModelBase || data is ReactiveUI.ReactiveObject;
}
```

> DI를 통해 **View까지** 만들고 싶다면 `Build` 내부에서 `App.Host.Services.GetRequiredService<SomeView>()`로 해결할 수 있다.  
> 단, Avalonia의 XAML 로더와의 균형을 맞추기 위해 View는 보통 XAML 인스턴스화를 그대로 두고, **DataContext만 DI**로 공급하는 패턴이 흔하다.

### 4.2 App.axaml — DataTemplates 등록

```xml
<!-- App.axaml -->
<Application xmlns="https://github.com/avaloniaui"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:infra="clr-namespace:MyAvaloniaApp.Infrastructure;assembly=MyAvaloniaApp"
             x:Class="MyAvaloniaApp.App">
  <Application.Styles>
    <FluentTheme Mode="Light"/>
    <ResourceInclude Source="avares://MyAvaloniaApp/Themes/LightTheme.axaml"/>
  </Application.Styles>

  <Application.DataTemplates>
    <infra:ViewLocator/>
  </Application.DataTemplates>
</Application>
```

### 4.3 MainView — ContentControl로 페이지 교체

```xml
<!-- Views/MainView.axaml -->
<UserControl xmlns="https://github.com/avaloniaui"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             x:Class="MyAvaloniaApp.Views.MainView">
  <DockPanel>
    <StackPanel Orientation="Horizontal" DockPanel.Dock="Top" Spacing="8" Margin="8">
      <Button Content="Login" Command="{Binding NavigateLogin}"/>
      <Button Content="Dashboard" Command="{Binding NavigateDashboard}"/>
      <Button Content="Settings" Command="{Binding NavigateSettings}"/>
    </StackPanel>

    <ContentControl Content="{Binding Current}"/>
  </DockPanel>
</UserControl>
```

```csharp
// Views/MainView.axaml.cs
using Avalonia.Controls;

namespace MyAvaloniaApp.Views;

public partial class MainView : UserControl
{
    public MainView() => InitializeComponent();
}
```

```csharp
// ViewModels/MainViewModel.cs (네비게이션 커맨드 추가)
using System.Reactive;
using ReactiveUI;
using MyAvaloniaApp.Services.Abstractions;

namespace MyAvaloniaApp.ViewModels;

public sealed class MainViewModel : ReactiveObject
{
    private readonly INavigationService _nav;
    private readonly LoginViewModel _loginVm;
    private readonly DashboardViewModel _dashVm;
    private readonly SettingsViewModel _settingsVm;

    public ReactiveCommand<Unit, Unit> NavigateLogin { get; }
    public ReactiveCommand<Unit, Unit> NavigateDashboard { get; }
    public ReactiveCommand<Unit, Unit> NavigateSettings { get; }

    private object? _current;
    public object? Current
    {
        get => _current;
        set => this.RaiseAndSetIfChanged(ref _current, value);
    }

    public MainViewModel(INavigationService nav, LoginViewModel loginVm,
                         DashboardViewModel dashVm, SettingsViewModel settingsVm)
    {
        _nav = nav;
        _loginVm = loginVm;
        _dashVm = dashVm;
        _settingsVm = settingsVm;

        _nav.Navigated += vm => Current = vm;

        NavigateLogin      = ReactiveCommand.Create(() => _nav.NavigateTo(_loginVm));
        NavigateDashboard  = ReactiveCommand.Create(() => _nav.NavigateTo(_dashVm));
        NavigateSettings   = ReactiveCommand.Create(() => _nav.NavigateTo(_settingsVm));

        Current = _loginVm;
    }
}
```

> **대안**: `Func<LoginViewModel>`을 DI 받아 **새 인스턴스 생성**(Transient) 구조로 바꿀 수 있다. 페이지 전환 시마다 “깨끗한 VM”이 필요하면 `Func<T>`/Factory 패턴이 유용하다.

---

## 5. 수명 주기(Lifetime) 선택과 “유사 Scoped”

| 등록 | 권장 사용 |
|---|---|
| `AddSingleton<T>` | 전역 서비스(설정, 네비, 메시지버스, 테마 등) |
| `AddTransient<T>` | ViewModel, 값 변경을 가져가는 가벼운 서비스 |
| `AddScoped<T>` | 웹/DI 범주 개념. Avalonia에서는 기본적으로 사용하지 않음 |

**유사 Scoped**: 윈도우/다이얼로그 단위로 별도의 DI 범위를 두고 싶다면:

```csharp
// 윈도우를 열 때 스코프를 만들어 해당 윈도우 수명에 맞춰 해제
using var scope = App.Host.Services.CreateScope();
var childVm = scope.ServiceProvider.GetRequiredService<SomeWindowViewModel>();
var window = new SomeWindow { DataContext = childVm };
await window.ShowDialog(owner);
```

이렇게 하면 윈도우의 생애 동안만 필요한 객체(Transient 포함)를 묶어 관리할 수 있다.

---

## 6. Settings/Options 패턴과 저장

### 6.1 Options 정의/바인딩

```csharp
// Infrastructure/Options/AppOptions.cs
namespace MyAvaloniaApp.Infrastructure.Options;

public sealed class AppOptions
{
    public string DataFolder { get; set; } = "AppData";
    public string Theme { get; set; } = "Light";
}
```

`appsettings.json`:

```json
{
  "App": {
    "DataFolder": "AppData",
    "Theme": "Light"
  },
  "Api": {
    "BaseUrl": "https://api.example.com/"
  }
}
```

DI 바인딩은 위의 Host 구성에서 이미 보였다(`services.Configure<AppOptions>(...)`).  
VM에서 읽으려면 `IOptionsMonitor<AppOptions>` 주입:

```csharp
using Microsoft.Extensions.Options;
using MyAvaloniaApp.Infrastructure.Options;
using ReactiveUI;

public sealed class SettingsViewModel : ReactiveObject
{
    private readonly IOptionsMonitor<AppOptions> _opts;
    private readonly IThemeService _theme;

    public SettingsViewModel(IOptionsMonitor<AppOptions> opts, IThemeService theme)
    {
        _opts = opts;
        _theme = theme;

        if (_opts.CurrentValue.Theme == "Dark") _theme.ApplyDark();
        else _theme.ApplyLight();
    }
}
```

### 6.2 사용자 설정(런타임) 저장

서비스(`IJsonStore`)로 AppData 경로에 저장/불러오기:

```csharp
public sealed class SettingsViewModel : ReactiveObject
{
    private readonly IJsonStore _store;
    private readonly string _path;

    private string _theme = "Light";
    public string Theme
    {
        get => _theme;
        set => this.RaiseAndSetIfChanged(ref _theme, value);
    }

    public SettingsViewModel(IJsonStore store, IOptionsMonitor<AppOptions> opts)
    {
        _store = store;
        _path = Path.Combine(opts.CurrentValue.DataFolder, "settings.json");
    }

    public async Task SaveAsync()  => await _store.SaveAsync(_path, new { Theme });
    public async Task LoadAsync()
    {
        var data = await _store.LoadAsync<dynamic>(_path);
        if (data is not null && data.Theme is string th) Theme = th;
    }
}
```

---

## 7. HTTP 클라이언트(typed client)와 DI

```csharp
// Services/Http/ApiClientOptions.cs
namespace MyAvaloniaApp.Services.Http;

public sealed class ApiClientOptions
{
    public string? BaseUrl { get; set; }
}
```

```csharp
// Services/Http/ApiClient.cs
using System;
using System.Net.Http;
using System.Threading.Tasks;

namespace MyAvaloniaApp.Services.Http;

public sealed class ApiClient
{
    private readonly HttpClient _http;

    public ApiClient(HttpClient http) => _http = http;

    public async Task<string> GetHelloAsync()
    {
        var res = await _http.GetStringAsync("hello");
        return res;
    }
}
```

등록은 Generic Host 예시에서 `AddHttpClient<ApiClient>`로 수행했다.  
VM에서 주입받아 사용:

```csharp
public sealed class DashboardViewModel : ReactiveObject
{
    private readonly Services.Http.ApiClient _api;
    public DashboardViewModel(IAuthService auth, INavigationService nav, Services.Http.ApiClient api) { _api = api; }

    public async Task<string> LoadFromServerAsync() => await _api.GetHelloAsync();
}
```

---

## 8. Dialog/Navigation/Theme/MessageBus DI 결합

- **Dialog**: `IFileDialogService`를 주입받아 모든 열기/저장 상호작용을 **VM에서** 트리거
- **Navigation**: `INavigationService.NavigateTo(vm)`로 페이지 교체 — **ViewModel끼리 종속 제거**
- **Theme**: `IThemeService`로 라이트/다크 전환을 **서비스 한 곳**에서
- **MessageBus**: `IMessageBus`로 브로드캐스트/구독 — **헐겁게 결합**

이들 모두 **생성자 주입**으로 전달하므로, 테스트 시 **Fake/Mock**으로 치환하기 쉽다.

---

## 9. 테스트 — 목 주입으로 VM 단위 테스트

```csharp
// Tests/LoginViewModelTests.cs
using System.Threading.Tasks;
using Microsoft.Extensions.DependencyInjection;
using MyAvaloniaApp.Services.Abstractions;
using MyAvaloniaApp.ViewModels;
using Xunit;

public sealed class FakeAuth : IAuthService
{
    public bool IsAuthenticated { get; private set; }
    public string? CurrentUser { get; private set; }
    public Task<bool> LoginAsync(string u, string p)
    {
        IsAuthenticated = true; CurrentUser = u;
        return Task.FromResult(true);
    }
    public Task LogoutAsync() { IsAuthenticated = false; CurrentUser = null; return Task.CompletedTask; }
}

public sealed class FakeNav : INavigationService
{
    public object? LastVm;
    public event System.Action<object>? Navigated;
    public void NavigateTo(object vm) { LastVm = vm; Navigated?.Invoke(vm); }
}

public class LoginViewModelTests
{
    [Fact]
    public async Task Login_Navigates_To_Dashboard()
    {
        var sc = new ServiceCollection();
        sc.AddSingleton<IAuthService, FakeAuth>();
        sc.AddSingleton<INavigationService, FakeNav>();
        sc.AddTransient<LoginViewModel>();
        var sp = sc.BuildServiceProvider();

        var vm = sp.GetRequiredService<LoginViewModel>();
        vm.Username = "tester";
        vm.Password = "pw";

        var ok = await vm.LoginCommand.Execute();
        Assert.True(ok);

        var nav = (FakeNav) sp.GetRequiredService<INavigationService>();
        Assert.NotNull(nav.LastVm);
        Assert.IsType<DashboardViewModel>(nav.LastVm);
    }
}
```

> “View가 없는 순수 ViewModel 테스트”가 **DI의 가장 큰 이점**이다.

---

## 10. 디자인-타임 데이터 (XAML 프리뷰)

```xml
<!-- Views/LoginView.axaml -->
<UserControl xmlns="https://github.com/avaloniaui"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:d="https://github.com/avaloniaui"
             xmlns:vm="clr-namespace:MyAvaloniaApp.ViewModels"
             x:Class="MyAvaloniaApp.Views.LoginView">
  <UserControl.DataContext>
    <vm:LoginViewModel d:DesignInstance="True" />
  </UserControl.DataContext>

  <!-- 디자인 시에는 DI가 없으므로, d:DesignInstance로 임시 VM 제공.
       런타임에서는 Host에서 DataContext 주입(또는 MainView에서 전개) -->
  <StackPanel Margin="20" Spacing="8">
    <TextBox Watermark="ID" Text="{Binding Username}"/>
    <TextBox Watermark="PW" Text="{Binding Password}"/>
    <Button Content="Login" Command="{Binding LoginCommand}"/>
  </StackPanel>
</UserControl>
```

> 디자인 타임에서는 DI 컨테이너가 없으므로, `d:` 네임스페이스를 활용해 임시 인스턴스를 붙인다. 런타임 연결은 MainView/MainWindow에서 수행.

---

## 11. 모듈/플러그인 아키텍처(확장)

크게 나누어 **기본 모듈**과 **기능 모듈**로 나누고, 각 모듈에서 `IServiceCollection` 확장 메서드로 자기 등록을 수행:

```csharp
// Infrastructure/Extensions/ServiceCollectionExtensions.cs
using Microsoft.Extensions.DependencyInjection;
using MyAvaloniaApp.Services.Abstractions;
using MyAvaloniaApp.Services.Implementations;

namespace MyAvaloniaApp.Infrastructure.Extensions;

public static class ServiceCollectionExtensions
{
    public static IServiceCollection AddCoreServices(this IServiceCollection s)
        => s.AddSingleton<IAuthService, AuthService>()
            .AddSingleton<INavigationService, NavigationService>()
            .AddSingleton<IMessageBus, MessageBus>()
            .AddSingleton<IFileDialogService, FileDialogService>()
            .AddSingleton<IThemeService, ThemeService>()
            .AddSingleton<IJsonStore, JsonStore>();
}
```

Host 구성:

```csharp
services.AddCoreServices()
        .AddTransient<LoginViewModel>()
        .AddTransient<DashboardViewModel>()
        .AddSingleton<MainViewModel>();
```

플러그인 라이브러리가 있을 경우, **IServiceCollection 확장**을 외부 어셈블리가 노출하면 주 애플리케이션에서 `services.AddPluginXyz()`로 모듈을 조립할 수 있다.

---

## 12. 다중 윈도우와 스코프

여러 윈도우를 동시에 띄우면서 **각 윈도우마다 다른 DI 스코프**를 주고 싶다면:

```csharp
var scope = App.Host.Services.CreateScope();
var vm = scope.ServiceProvider.GetRequiredService<SomeWindowViewModel>();
var win = new SomeWindow { DataContext = vm };
win.Closing += (_, __) => scope.Dispose(); // 윈도우 닫히면 스코프 해제
win.Show();
```

윈도우-스코프 간 결합을 **NavigationService**에서 지원하도록 만들어도 된다(윈도우 생성 책임을 서비스로 승격).

---

## 13. 백그라운드 작업/타이머와 DI

`.NET`의 `PeriodicTimer`를 통한 폴링/백그라운드 작업:

```csharp
public sealed class BackgroundRefresher
{
    private readonly IMessageBus _bus;
    private readonly PeriodicTimer _timer = new(TimeSpan.FromSeconds(10));
    private readonly CancellationTokenSource _cts = new();

    public BackgroundRefresher(IMessageBus bus) => _bus = bus;

    public async Task RunAsync()
    {
        while (await _timer.WaitForNextTickAsync(_cts.Token))
        {
            _bus.Publish(new RefreshTick());
        }
    }

    public void Stop() => _cts.Cancel();
}

public sealed record RefreshTick();
```

**DI로 싱글턴** 등록 후 `App.OnFrameworkInitializationCompleted`에서 시작.  
종료 시 `Stop`. 메시지는 `DashboardViewModel` 등에서 구독하여 UI 갱신.

---

## 14. 요약/가이드라인

- **생성자 주입** 고정: ViewModel/Service는 항상 DI에서 제공
- **Singleton**: 전역 상태/라우팅/테마/메시지버스/저장소
- **Transient**: ViewModel(페이지마다 새 인스턴스가 유리한 경우)
- “Scoped”가 필요하면 **윈도우 단위 스코프**를 직접 만든다
- **Navigation**/MessageBus/Theme/FileDialog 등 **UI 인프라**는 서비스화
- 중대형 앱: **Generic Host**로 가서 구성/로깅/HttpClient/Options를 한 번에
- 테스트에서는 **Fake/Mock** 서비스 주입으로 **View 없이** 검증
- 디자인 타임은 `d:`로 해결, 런타임은 DI

---

## 부록 A. 전체 실행 흐름(Host 버전)

1) `App.OnFrameworkInitializationCompleted()` → `Host = Bootstrapper.BuildHost()`  
2) `MainViewModel`을 `Host.Services.GetRequiredService<MainViewModel>()`로 호출 → DI가 내부적으로 `LoginViewModel` 등 생성  
3) `MainWindow`의 `DataContext = MainViewModel`  
4) `MainView`에는 `<ContentControl Content="{Binding Current}" />` + `ViewLocator`  
5) 페이지 전환은 `INavigationService.NavigateTo(vm)` 호출로 UI 반영

---

## 부록 B. 간단 View 코드 (Login/Dashboard/Settings)

```xml
<!-- Views/LoginView.axaml -->
<UserControl xmlns="https://github.com/avaloniaui"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             x:Class="MyAvaloniaApp.Views.LoginView">
  <StackPanel Margin="20" Spacing="8">
    <TextBox Watermark="Username" Text="{Binding Username}"/>
    <TextBox Watermark="Password" Text="{Binding Password}"/>
    <Button Content="Login" Command="{Binding LoginCommand}"/>
  </StackPanel>
</UserControl>
```

```xml
<!-- Views/DashboardView.axaml -->
<UserControl xmlns="https://github.com/avaloniaui"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             x:Class="MyAvaloniaApp.Views.DashboardView">
  <StackPanel Margin="20" Spacing="8">
    <TextBlock Text="{Binding Title}" FontSize="18" FontWeight="Bold"/>
  </StackPanel>
</UserControl>
```

```xml
<!-- Views/SettingsView.axaml -->
<UserControl xmlns="https://github.com/avaloniaui"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             x:Class="MyAvaloniaApp.Views.SettingsView">
  <StackPanel Margin="20" Spacing="8">
    <TextBlock Text="Settings"/>
  </StackPanel>
</UserControl>
```

---

## 결론

- 본 문서는 초안의 DI 구성을 **확장/일반화**했다.  
- 작은 프로젝트는 **App에서 간단 DI**로 시작해도 충분하다.  
- 규모가 커질수록 **Generic Host**로 전환해 구성/로깅/옵션/HTTP/백그라운드까지 **표준화**하라.  
- ViewModel/Service는 항상 **생성자 주입**, View는 **XAML + DataContext만 DI**를 유지하면 MVVM과 DI가 자연스럽게 결합된다.

필요 시 다음 단계로:
- 네임드/키드(Named/Keyed) 서비스 등록(멀티 구현)
- 플러그인(어셈블리 스캔) 자동 등록
- 다국어(리소스) + Options 변경 시 실시간 반영(IOptionsMonitor)
- 고급 네비게이션(스택, 매개변수, 히스토리)과 상태 보존(StateContainer) 설계