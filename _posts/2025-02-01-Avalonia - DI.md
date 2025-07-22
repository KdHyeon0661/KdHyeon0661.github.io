---
layout: post
title: Avalonia - DI(Dependency Injection)
date: 2025-02-01 20:20:23 +0900
category: Avalonia
---
# 🧩 Avalonia MVVM에서의 Dependency Injection 구조 정리

---

## ✅ 왜 DI가 필요한가요?

| 항목 | 이유 |
|------|------|
| **결합도 감소** | ViewModel → Service 직접 참조 제거 |
| **테스트 가능성 향상** | Mock 객체로 대체 가능 |
| **확장성 확보** | 서비스 교체/버전 변경 시 유리 |
| **중앙 집중 관리** | 싱글턴, 범위, 임시 객체 수명 주기 관리

---

## 🔧 사용 도구

- DI 컨테이너: `Microsoft.Extensions.DependencyInjection`
- 라이프사이클 제어: `AddSingleton`, `AddTransient`, `AddScoped`
- ViewModel/Service 연결: 생성자 주입 방식
- App.xaml.cs → `ConfigureServices()`로 구성

---

## 📁 기본 구조 예시

```
MyAvaloniaApp/
├── App.xaml / App.xaml.cs
├── ViewModels/
│   ├── MainViewModel.cs
│   └── LoginViewModel.cs
├── Views/
│   └── LoginView.axaml
├── Services/
│   └── IAuthService.cs
│   └── AuthService.cs
└── Program.cs
```

---

# 1️⃣ 서비스 인터페이스 및 구현

## 📄 IAuthService.cs

```csharp
public interface IAuthService
{
    Task<bool> LoginAsync(string username, string password);
}
```

## 📄 AuthService.cs

```csharp
public class AuthService : IAuthService
{
    public Task<bool> LoginAsync(string username, string password)
    {
        // 실제 로그인 처리 로직 (예: API 호출)
        return Task.FromResult(username == "admin" && password == "1234");
    }
}
```

---

# 2️⃣ ViewModel에 서비스 주입

## 📄 LoginViewModel.cs

```csharp
public class LoginViewModel : ReactiveObject
{
    private readonly IAuthService _authService;

    public LoginViewModel(IAuthService authService)
    {
        _authService = authService;

        LoginCommand = ReactiveCommand.CreateFromTask(ExecuteLogin);
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
        return await _authService.LoginAsync(Username, Password);
    }
}
```

---

# 3️⃣ DI 등록 설정: App.xaml.cs 또는 Program.cs

### Avalonia 11+ 기준

## 📄 App.xaml.cs

```csharp
public class App : Application
{
    public static IServiceProvider Services { get; private set; } = default!;

    public override void Initialize()
    {
        AvaloniaXamlLoader.Load(this);
    }

    public override void OnFrameworkInitializationCompleted()
    {
        var serviceCollection = new ServiceCollection();

        ConfigureServices(serviceCollection);
        Services = serviceCollection.BuildServiceProvider();

        var mainWindow = new MainWindow
        {
            DataContext = Services.GetRequiredService<MainViewModel>()
        };

        ApplicationLifetime!.MainWindow = mainWindow;

        base.OnFrameworkInitializationCompleted();
    }

    private void ConfigureServices(IServiceCollection services)
    {
        // Service 등록
        services.AddSingleton<IAuthService, AuthService>();

        // ViewModel 등록
        services.AddSingleton<MainViewModel>();
        services.AddTransient<LoginViewModel>();
    }
}
```

---

# 4️⃣ View와 ViewModel 연결

## 📄 MainWindow.xaml.cs

```csharp
public partial class MainWindow : Window
{
    public MainWindow()
    {
        InitializeComponent();

        DataContext = App.Services.GetRequiredService<MainViewModel>();
    }
}
```

## 📄 View 내부에서 ViewModel 생성 안함 ❌

```csharp
// ❌ 이런 방식은 지양
<DataContext>
    <vm:LoginViewModel />
</DataContext>
```

> 대신 DI 컨테이너로부터 인스턴스를 주입받는 방식 사용

---

# 5️⃣ 네비게이션 시 ViewModel DI 사용

```csharp
public class MainViewModel : ReactiveObject
{
    private readonly Func<LoginViewModel> _loginVmFactory;

    public MainViewModel(Func<LoginViewModel> loginVmFactory)
    {
        _loginVmFactory = loginVmFactory;
    }

    public void NavigateToLogin()
    {
        var loginVm = _loginVmFactory();
        CurrentPage = loginVm;
    }

    private ReactiveObject? _currentPage;
    public ReactiveObject? CurrentPage
    {
        get => _currentPage;
        set => this.RaiseAndSetIfChanged(ref _currentPage, value);
    }
}
```

> ✅ `Func<T>`를 등록하면 매번 새로운 ViewModel을 DI를 통해 생성할 수 있음

---

## 🔁 라이프사이클 선택 가이드

| 등록 방식 | 사용 예 |
|-----------|----------|
| `AddSingleton<T>` | 앱 전체 공유 (예: 설정, 전역 상태) |
| `AddTransient<T>` | 매번 새 인스턴스 (ViewModel 등) |
| `AddScoped<T>` | 웹 전용, Avalonia에서는 사용 안 함 |

---

## 🧪 테스트에서 DI 활용

```csharp
var services = new ServiceCollection();
services.AddTransient<IAuthService, FakeAuthService>();
services.AddTransient<LoginViewModel>();

var provider = services.BuildServiceProvider();
var loginVm = provider.GetRequiredService<LoginViewModel>();
```

---

## 🧱 확장: NavigationService, MessageBus 도입 시

```csharp
services.AddSingleton<INavigationService, NavigationService>();
services.AddSingleton<IMessageBus, MessageBus>();
```

---

# ✅ 결론: Avalonia + DI 아키텍처 정리

| 역할 | 구현 방법 |
|------|-----------|
| 서비스 등록 | `ConfigureServices`에서 명시 |
| ViewModel 생성 | DI로 주입받기 (생성자 주입) |
| View 연결 | `App.Services.GetRequiredService<>()` |
| 테스트 유연성 | 모킹된 서비스 주입 가능 |
| 네비게이션 유연화 | ViewModel Factory 활용 |

---

## 📘 다음 주제 추천

- 🧭 NavigationService로 ViewModel 간 이동 구조화
- 🧪 ViewModel 단위 테스트에서 DI 활용
- 🧩 Scoped lifetime 없이 ViewModel 상태 공유 방법 (StateContainer)
- 🧬 MessageBus or EventAggregator와 DI 결합
