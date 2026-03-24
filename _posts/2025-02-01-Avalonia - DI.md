---
layout: post
title: Avalonia - DI(Dependency Injection)
date: 2025-02-01 20:20:23 +0900
category: Avalonia
---
# Avalonia MVVM에서의 의존성 주입(DI) 구조와 베스트 프랙티스

의존성 주입(Dependency Injection, DI)은 객체가 직접 의존 객체를 생성하지 않고 외부에서 받아 사용하는 설계 패턴입니다. Avalonia 애플리케이션에서 DI를 도입하면 ViewModel과 서비스 간 결합도를 낮추고, 테스트 용이성과 유지보수성을 크게 향상시킬 수 있습니다. 이 글에서는 초중급 개발자를 위해 DI의 기본 개념부터 실전 적용, 그리고 프로젝트 규모에 따른 구성 전략까지 단계별로 설명합니다.

---

## 프로젝트 구조

```
MyAvaloniaApp/
├─ App.axaml
├─ App.axaml.cs
├─ Program.cs
├─ Infrastructure/               # DI·Host·Options·Logging 등 인프라
│  ├─ Bootstrapper.cs
│  ├─ ViewLocator.cs
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
│  └─ DashboardViewModel.cs
├─ Views/
│  ├─ MainView.axaml
│  ├─ LoginView.axaml
│  └─ DashboardView.axaml
└─ Tests/
   └─ LoginViewModelTests.cs
```

위 구조는 관심사를 명확히 분리합니다. `Services/Abstractions`에는 인터페이스, `Services/Implementations`에는 구체 클래스, `ViewModels`에는 프레젠테이션 로직, `Views`에는 XAML UI를 배치합니다. `Infrastructure`에는 DI 구성, 뷰 로케이터, 옵션 설정 등 인프라 관련 코드를 모읍니다.

---

## 서비스 인터페이스와 구현 예시

DI를 적용하려면 먼저 서비스의 인터페이스를 정의하고, 이를 구현한 클래스를 만듭니다. 여기서는 몇 가지 필수적인 서비스를 예시로 보여줍니다.

### 인증 서비스 (IAuthService)

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
        // 실제 환경에서는 API 호출, 토큰 저장 등을 수행
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

### 파일 다이얼로그 서비스 (IFileDialogService)

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
using System.Linq;
using System.Threading.Tasks;
using Avalonia.Controls;
using MyAvaloniaApp.Services.Abstractions;

namespace MyAvaloniaApp.Services.Implementations;

public sealed class FileDialogService : IFileDialogService
{
    private readonly Window? _owner;

    public FileDialogService()
    {
        // MainWindow가 나타난 후에 Owner를 얻을 수 있음
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

### 네비게이션 서비스 (INavigationService)

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

### 메시지 버스 (IMessageBus)

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

### 테마 서비스 (IThemeService)

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

        // 기존 테마 리소스 제거
        var existing = app.Styles.FirstOrDefault(s =>
            s is ResourceInclude ri && ri.Source?.ToString()?.Contains("Theme") == true);
        if (existing is not null) app.Styles.Remove(existing);

        // 새 테마 추가
        var include = new ResourceInclude(new Uri(uri)) { Source = new Uri(uri) };
        app.Styles.Add(include);
        IsDark = isDark;
    }
}
```

### JSON 저장소 (IJsonStore)

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

## ViewModel에 서비스 주입

DI의 핵심은 필요한 서비스를 생성자 매개변수로 받는 것입니다. ViewModel은 자신이 의존하는 서비스의 인터페이스만 알면 됩니다.

### LoginViewModel

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

### DashboardViewModel

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
        Title = $"환영합니다, {_auth.CurrentUser ?? "Guest"}";
    }

    private string _title;
    public string Title
    {
        get => _title;
        set => this.RaiseAndSetIfChanged(ref _title, value);
    }
}
```

### MainViewModel

`MainViewModel`은 네비게이션 서비스를 통해 현재 화면을 바꿉니다.

```csharp
// ViewModels/MainViewModel.cs
using System.Reactive;
using ReactiveUI;
using MyAvaloniaApp.Services.Abstractions;

namespace MyAvaloniaApp.ViewModels;

public sealed class MainViewModel : ReactiveObject
{
    private readonly INavigationService _nav;
    private object? _current;

    public object? Current
    {
        get => _current;
        set => this.RaiseAndSetIfChanged(ref _current, value);
    }

    public MainViewModel(INavigationService nav, LoginViewModel loginVm)
    {
        _nav = nav;
        Current = loginVm;
        _nav.Navigated += vm => Current = vm;
    }
}
```

> **주의**: `DashboardViewModel`이 `MainViewModel` 대신 `LoginViewModel`에서 직접 생성되는 구조는 `DashboardViewModel`의 생성자 인자를 위해 `IAuthService`와 `INavigationService`를 다시 전달해야 합니다. 더 나은 설계를 위해 `IServiceProvider`나 팩토리를 활용할 수도 있지만, 이 예제에서는 간단히 보여주기 위해 생성자 인자를 전달했습니다.

---

## DI 구성 방식

Avalonia 앱에서 DI 컨테이너를 구성하는 방법은 크게 두 가지로 나눌 수 있습니다. 작은 프로젝트에는 간단한 `ServiceCollection` 방식이, 중대형 프로젝트에는 .NET Generic Host 방식이 적합합니다.

### 방식 A: App.axaml.cs에서 직접 구성

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

    public override void Initialize() => AvaloniaXamlLoader.Load(this);

    public override void OnFrameworkInitializationCompleted()
    {
        var sc = new ServiceCollection();

        // 서비스 등록
        sc.AddSingleton<IAuthService, AuthService>();
        sc.AddSingleton<INavigationService, NavigationService>();
        sc.AddSingleton<IMessageBus, MessageBus>();
        sc.AddSingleton<IFileDialogService, FileDialogService>();
        sc.AddSingleton<IThemeService, ThemeService>();
        sc.AddSingleton<IJsonStore, JsonStore>();

        // ViewModel 등록
        sc.AddSingleton<MainViewModel>();
        sc.AddTransient<LoginViewModel>();
        sc.AddTransient<DashboardViewModel>();

        Services = sc.BuildServiceProvider();

        if (ApplicationLifetime is IClassicDesktopStyleApplicationLifetime desktop)
        {
            var mainVm = Services.GetRequiredService<MainViewModel>();
            desktop.MainWindow = new MainWindow { DataContext = mainVm };
        }

        base.OnFrameworkInitializationCompleted();
    }
}
```

**장점**: 단순하고 이해하기 쉽습니다.  
**단점**: 구성, 로깅, 옵션 바인딩 등이 내장되어 있지 않아 직접 구현해야 합니다.

### 방식 B: .NET Generic Host 연동

Generic Host는 `appsettings.json`, 환경 변수, 로깅, `IOptions<T>` 등을 표준 방식으로 제공합니다.

```csharp
// Infrastructure/Bootstrapper.cs
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
        var host = Host.CreateDefaultBuilder()
            .ConfigureAppConfiguration(cfg =>
            {
                // 필요 시 추가 구성 소스 등록
                // cfg.AddJsonFile("appsettings.local.json", optional: true);
            })
            .ConfigureServices((ctx, services) =>
            {
                // 옵션 바인딩
                services.Configure<Options.AppOptions>(ctx.Configuration.GetSection("App"));

                // HTTP 클라이언트 등록
                services.AddHttpClient<Services.Http.ApiClient>(client =>
                {
                    var apiBase = ctx.Configuration["Api:BaseUrl"];
                    if (!string.IsNullOrWhiteSpace(apiBase))
                        client.BaseAddress = new Uri(apiBase);
                });

                // 서비스 등록
                services.AddSingleton<IAuthService, AuthService>();
                services.AddSingleton<INavigationService, NavigationService>();
                services.AddSingleton<IMessageBus, MessageBus>();
                services.AddSingleton<IFileDialogService, FileDialogService>();
                services.AddSingleton<IThemeService, ThemeService>();
                services.AddSingleton<IJsonStore, JsonStore>();

                // ViewModel 등록
                services.AddSingleton<MainViewModel>();
                services.AddTransient<LoginViewModel>();
                services.AddTransient<DashboardViewModel>();
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
// App.axaml.cs (Host 버전)
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

    public override void Initialize() => AvaloniaXamlLoader.Load(this);

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

**장점**: 구성, 로깅, 옵션, HTTP 클라이언트 팩토리 등 .NET 표준 기능을 그대로 활용할 수 있습니다.  
**단점**: 초기 설정이 다소 복잡할 수 있습니다.

---

## View와 ViewModel 연결

DI로 생성된 ViewModel을 View에 주입하는 방법은 다양합니다. 가장 깔끔한 방법은 **ViewLocator**를 사용하는 것입니다.

### ViewLocator 구현

```csharp
// Infrastructure/ViewLocator.cs
using Avalonia.Controls;
using Avalonia.Controls.Templates;
using MyAvaloniaApp.ViewModels;
using MyAvaloniaApp.Views;

namespace MyAvaloniaApp.Infrastructure;

public sealed class ViewLocator : IDataTemplate
{
    public Control Build(object? data)
    {
        return data switch
        {
            LoginViewModel      => new LoginView(),
            DashboardViewModel  => new DashboardView(),
            MainViewModel       => new MainView(),
            _                   => new TextBlock { Text = "View Not Found" }
        };
    }

    public bool Match(object? data) => data is ViewModelBase;
}
```

### App.axaml에서 DataTemplate 등록

```xml
<!-- App.axaml -->
<Application xmlns="https://github.com/avaloniaui"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:infra="clr-namespace:MyAvaloniaApp.Infrastructure"
             x:Class="MyAvaloniaApp.App">
    <Application.Styles>
        <FluentTheme Mode="Light"/>
    </Application.Styles>

    <Application.DataTemplates>
        <infra:ViewLocator/>
    </Application.DataTemplates>
</Application>
```

### MainView에서 ContentControl로 현재 ViewModel 표시

```xml
<!-- Views/MainView.axaml -->
<UserControl xmlns="https://github.com/avaloniaui"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             x:Class="MyAvaloniaApp.Views.MainView">
    <ContentControl Content="{Binding Current}" />
</UserControl>
```

이제 `Current` 속성이 `LoginViewModel`이나 `DashboardViewModel`로 바뀌면 ViewLocator가 적절한 View를 자동으로 렌더링합니다.

---

## 수명 주기 전략

DI 컨테이너에서 객체의 수명을 결정하는 것은 중요합니다. Avalonia 앱에서는 주로 두 가지 수명을 사용합니다.

| 등록 방식 | 설명 | 적합한 대상 |
|-----------|------|-------------|
| `AddSingleton<T>` | 앱 전체에서 하나의 인스턴스만 생성 | 전역 서비스 (네비게이션, 테마, 메시지 버스, 설정) |
| `AddTransient<T>` | 요청할 때마다 새 인스턴스 생성 | ViewModel (각 화면마다 독립적인 상태가 필요) |

`AddScoped<T>`는 웹 애플리케이션에서 요청 단위로 수명을 관리하는 데 사용되지만, 데스크톱 앱에서는 기본적으로 지원되지 않습니다. 대신 **윈도우 단위 스코프**를 직접 구현할 수 있습니다.

### 윈도우 단위 스코프 (유사 Scoped)

새 윈도우를 열 때마다 별도의 DI 스코프를 생성하여 해당 윈도우의 ViewModel과 서비스를 독립적으로 관리할 수 있습니다.

```csharp
var scope = App.Host.Services.CreateScope();
var vm = scope.ServiceProvider.GetRequiredService<SomeWindowViewModel>();
var window = new SomeWindow { DataContext = vm };
window.Closing += (_, _) => scope.Dispose(); // 윈도우 닫히면 스코프 해제
window.Show();
```

이렇게 하면 윈도우가 닫힐 때 해당 윈도우에 연결된 모든 Transient 객체가 정리됩니다.

---

## 설정(Options) 패턴과 저장

Generic Host를 사용하면 `appsettings.json` 파일을 통해 설정을 관리할 수 있습니다.

### Options 클래스 정의

```csharp
// Infrastructure/Options/AppOptions.cs
namespace MyAvaloniaApp.Infrastructure.Options;

public sealed class AppOptions
{
    public string DataFolder { get; set; } = "AppData";
    public string Theme { get; set; } = "Light";
}
```

### appsettings.json 예시

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

### ViewModel에서 옵션 사용

```csharp
using Microsoft.Extensions.Options;
using MyAvaloniaApp.Infrastructure.Options;
using MyAvaloniaApp.Services.Abstractions;

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

옵션 변경 시 실시간 반영이 필요하면 `IOptionsMonitor<T>`의 `OnChange` 이벤트를 구독할 수 있습니다.

---

## HTTP 클라이언트와 DI

Typed HTTP 클라이언트를 사용하면 API 호출을 깔끔하게 캡슐화할 수 있습니다.

```csharp
// Services/Http/ApiClient.cs
using System.Net.Http;
using System.Threading.Tasks;

namespace MyAvaloniaApp.Services.Http;

public sealed class ApiClient
{
    private readonly HttpClient _http;
    public ApiClient(HttpClient http) => _http = http;

    public async Task<string> GetHelloAsync()
    {
        return await _http.GetStringAsync("hello");
    }
}
```

등록은 Generic Host 구성에서 `AddHttpClient<ApiClient>`로 처리했습니다. ViewModel에서 주입받아 사용합니다.

```csharp
public sealed class DashboardViewModel : ReactiveObject
{
    private readonly ApiClient _api;

    public DashboardViewModel(ApiClient api)
    {
        _api = api;
    }

    public async Task<string> LoadDataAsync() => await _api.GetHelloAsync();
}
```

---

## 단위 테스트: 목 주입으로 ViewModel 검증

DI의 큰 장점은 ViewModel을 실제 서비스 없이 독립적으로 테스트할 수 있다는 점입니다.

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
        IsAuthenticated = true;
        CurrentUser = u;
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

        var nav = (FakeNav)sp.GetRequiredService<INavigationService>();
        Assert.NotNull(nav.LastVm);
        Assert.IsType<DashboardViewModel>(nav.LastVm);
    }
}
```

---

## 디자인 타임 데이터 (XAML 프리뷰)

XAML 디자이너에서 ViewModel의 데이터를 미리 보려면 `d:DesignInstance`를 사용합니다.

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

  <StackPanel Margin="20" Spacing="8">
    <TextBox Watermark="ID" Text="{Binding Username}"/>
    <TextBox Watermark="PW" Text="{Binding Password}"/>
    <Button Content="Login" Command="{Binding LoginCommand}"/>
  </StackPanel>
</UserControl>
```

디자인 타임에는 DI 컨테이너가 없으므로, `d:DesignInstance`로 임시 ViewModel을 생성해 UI를 미리 볼 수 있습니다. 런타임에는 실제 DI로 생성된 ViewModel이 주입됩니다.

---

## 모듈/플러그인 아키텍처 확장

대규모 앱에서는 기능을 모듈로 분리하고, 각 모듈이 자신의 서비스를 DI 컨테이너에 등록하게 할 수 있습니다.

```csharp
// Infrastructure/Extensions/ServiceCollectionExtensions.cs
using Microsoft.Extensions.DependencyInjection;
using MyAvaloniaApp.Services.Abstractions;
using MyAvaloniaApp.Services.Implementations;

namespace MyAvaloniaApp.Infrastructure.Extensions;

public static class ServiceCollectionExtensions
{
    public static IServiceCollection AddCoreServices(this IServiceCollection s)
    {
        return s.AddSingleton<IAuthService, AuthService>()
                .AddSingleton<INavigationService, NavigationService>()
                .AddSingleton<IMessageBus, MessageBus>()
                .AddSingleton<IFileDialogService, FileDialogService>()
                .AddSingleton<IThemeService, ThemeService>()
                .AddSingleton<IJsonStore, JsonStore>();
    }
}
```

Host 구성에서 `services.AddCoreServices()`를 호출하면 됩니다. 플러그인 어셈블리가 있다면 해당 어셈블리의 확장 메서드를 호출하여 등록할 수 있습니다.

---

## 백그라운드 작업과 DI

`PeriodicTimer`를 이용해 주기적으로 실행되는 백그라운드 작업을 DI로 관리할 수 있습니다.

```csharp
// Services/Implementations/BackgroundRefresher.cs
using System;
using System.Threading;
using System.Threading.Tasks;
using MyAvaloniaApp.Services.Abstractions;

public sealed record RefreshTick();

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
```

이 서비스를 싱글턴으로 등록하고, `App.OnFrameworkInitializationCompleted`에서 시작합니다.

```csharp
var refresher = Host.Services.GetRequiredService<BackgroundRefresher>();
_ = refresher.RunAsync(); // 백그라운드에서 실행
```

종료 시 `refresher.Stop()`을 호출합니다.

---

## 요약 및 가이드라인

| 항목 | 권장 사항 |
|------|-----------|
| 서비스 인터페이스 | 항상 인터페이스로 정의하고, 구현은 별도 클래스로 분리 |
| 등록 수명 | 전역 서비스는 `Singleton`, ViewModel은 `Transient` |
| ViewModel 생성 | 생성자로 필요한 서비스 주입 (절대 `new`로 직접 생성 금지) |
| View 연결 | ViewLocator 또는 DataTemplate을 사용해 VM→View 자동 매핑 |
| 구성 | 작은 앱은 App.axaml.cs, 중대형 앱은 Generic Host 활용 |
| 설정 | `appsettings.json` + `IOptions<T>` 사용 |
| HTTP 통신 | `AddHttpClient<T>`로 Typed Client 등록 |
| 테스트 | Fake/Mock 서비스로 ViewModel 단위 테스트 작성 |
| 디자인 타임 | `d:DesignInstance`로 임시 ViewModel 제공 |
| 모듈화 | 확장 메서드로 서비스 등록을 캡슐화 |
| 백그라운드 | DI로 서비스 등록 후 앱 수명에 맞게 시작/종료 |

---

## 결론

Avalonia 애플리케이션에 DI를 도입하면 ViewModel과 서비스 간 결합도가 낮아지고, 테스트 용이성과 유지보수성이 크게 향상됩니다. 작은 프로젝트는 `ServiceCollection`으로 간단히 시작하고, 프로젝트가 성장함에 따라 Generic Host로 전환하여 구성, 로깅, 옵션 등을 표준화하는 것이 좋습니다. 이 글에서 소개한 패턴을 기반으로 자신의 앱에 맞는 DI 구조를 설계해 보세요.