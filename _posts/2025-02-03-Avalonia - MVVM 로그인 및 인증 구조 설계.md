---
layout: post
title: Avalonia - MVVM 로그인 및 인증 구조 설계
date: 2025-02-03 20:20:23 +0900
category: Avalonia
---
# Avalonia MVVM 로그인·인증 구조 설계

## 로그인 기능의 핵심 요구사항

로그인은 대부분의 애플리케이션에서 필수적인 기능이다. MVVM 패턴을 적용하면 UI와 인증 로직을 깔끔하게 분리할 수 있다. 설계 시 고려할 주요 사항은 다음과 같다.

- **화면 분리**: 로그인 화면과 메인 화면을 분리하여 각각 ViewModel을 가진다.
- **인증 처리**: API 서버와 통신하여 사용자 인증을 수행하고, 성공 시 토큰(예: JWT)을 받아 보관한다.
- **상태 관리**: 로그인 상태를 전역적으로 공유하여 애플리케이션의 여러 부분에서 접근 가능하게 한다.
- **자동 로그인**: 사용자가 선택한 경우, 암호화된 토큰을 로컬에 저장해 다음 실행 시 자동으로 로그인한다.
- **보안**: 비밀번호는 네트워크 전송 시 TLS로 보호하고, 토큰은 암호화하여 저장한다.

## 전체 흐름

```
사용자 입력 → LoginViewModel.LoginCommand → IAuthService.LoginAsync
                 ↓ (성공)
           UserSession 생성 → AppState에 저장 → ITokenStore.SaveAsync
                 ↓
            메인 화면으로 전환
                 ↓
           (실패 시) 오류 메시지 표시
```

## 프로젝트 구조 (간략)

```
MyApp/
├── Models/
│   ├── UserSession.cs        // 로그인 세션 정보
│   └── AuthResult.cs         // 서버 응답 DTO
├── Services/
│   ├── IAuthService.cs       // 인증 관련 추상화
│   ├── AuthService.cs        // HTTP 통신 구현
│   ├── ITokenStore.cs        // 토큰 저장 추상화
│   └── EncryptedJsonTokenStore.cs // 암호화 저장 구현
├── State/
│   └── AppState.cs           // 전역 로그인 상태
├── ViewModels/
│   ├── LoginViewModel.cs
│   └── MainViewModel.cs
├── Views/
│   ├── LoginView.axaml
│   └── MainView.axaml
└── App.axaml.cs
```

## 모델 정의

### UserSession (사용자 세션)

인증 성공 후 서버에서 받은 정보를 담는 객체다. 토큰의 만료 시간을 저장해 유효성을 확인할 수 있다.

```csharp
public class UserSession
{
    public string Username { get; set; } = "";
    public string AccessToken { get; set; } = "";
    public string? RefreshToken { get; set; }
    public DateTimeOffset IssuedAt { get; set; }
    public DateTimeOffset ExpiresAt { get; set; }

    public bool IsExpired(DateTimeOffset now) => now >= ExpiresAt;
}
```

### AuthResult (서버 응답)

API가 반환하는 로그인 성공 응답의 형태를 정의한다.

```csharp
public class AuthResult
{
    public string AccessToken { get; set; } = "";
    public string? RefreshToken { get; set; }
    public int ExpiresInSeconds { get; set; }
    public string Username { get; set; } = "";
}
```

## 전역 상태 관리 (AppState)

`AppState`는 현재 로그인된 사용자 정보를 담고, 변경 시 UI에 알린다. `ReactiveObject`를 상속받아 `INotifyPropertyChanged`를 구현한다.

```csharp
public class AppState : ReactiveObject
{
    private UserSession? _currentUser;

    public UserSession? CurrentUser
    {
        get => _currentUser;
        set => this.RaiseAndSetIfChanged(ref _currentUser, value);
    }

    public bool IsLoggedIn => CurrentUser != null && !CurrentUser.IsExpired(DateTimeOffset.UtcNow);
}
```

## 토큰 저장소 (자동 로그인)

자동 로그인을 위해 토큰을 로컬 파일에 저장해야 한다. 평문 저장은 위험하므로 AES-256-GCM으로 암호화한다. 아래는 인터페이스와 구현의 핵심이다.

```csharp
public interface ITokenStore
{
    Task SaveAsync(UserSession session, bool rememberMe, CancellationToken ct = default);
    Task<UserSession?> LoadAsync(CancellationToken ct = default);
    Task ClearAsync(CancellationToken ct = default);
}
```

`EncryptedJsonTokenStore`는 실제 파일 I/O와 암호화를 처리한다. 키 관리는 안전한 방법(예: 운영체제 보안 저장소)을 사용해야 한다.

## 인증 서비스 (IAuthService)

`IAuthService`는 로그인, 로그아웃, 토큰 갱신 등의 추상화를 제공한다. `HttpClient`를 통해 API와 통신한다.

```csharp
public interface IAuthService
{
    Task<UserSession?> LoginAsync(string username, string password, CancellationToken ct = default);
    Task<UserSession?> RefreshAsync(string refreshToken, CancellationToken ct = default);
    Task LogoutAsync(CancellationToken ct = default);
}
```

`AuthService` 구현 예시 (핵심 로직만):

```csharp
public class AuthService : IAuthService
{
    private readonly HttpClient _http;
    private readonly AppState _state;
    private readonly ITokenStore _store;

    public AuthService(HttpClient http, AppState state, ITokenStore store) { ... }

    public async Task<UserSession?> LoginAsync(string username, string password, CancellationToken ct)
    {
        var payload = new { username, password };
        var response = await _http.PostAsJsonAsync("/api/auth/login", payload, ct);
        if (!response.IsSuccessStatusCode) return null;

        var dto = await response.Content.ReadFromJsonAsync<AuthResult>(ct);
        var session = new UserSession
        {
            Username = dto.Username,
            AccessToken = dto.AccessToken,
            RefreshToken = dto.RefreshToken,
            ExpiresAt = DateTimeOffset.UtcNow.AddSeconds(dto.ExpiresInSeconds)
        };
        _state.CurrentUser = session;
        return session;
    }
}
```

## 로그인 ViewModel

`LoginViewModel`은 사용자 입력을 바인딩하고, 로그인 명령을 제공하며, 로그인 과정의 상태(로딩, 오류)를 관리한다. 또한 자동 로그인 시도를 위한 명령도 포함한다.

```csharp
public class LoginViewModel : ReactiveObject
{
    private readonly IAuthService _auth;
    private readonly ITokenStore _store;
    private readonly AppState _state;
    private readonly Action _onSuccess;  // 로그인 성공 시 호출할 콜백

    public LoginViewModel(IAuthService auth, ITokenStore store, AppState state, Action onSuccess)
    {
        _auth = auth; _store = store; _state = state; _onSuccess = onSuccess;

        var canLogin = this.WhenAnyValue(
            x => x.Username, x => x.Password, x => x.IsBusy,
            (u, p, b) => !b && !string.IsNullOrWhiteSpace(u) && !string.IsNullOrWhiteSpace(p));

        LoginCommand = ReactiveCommand.CreateFromTask(LoginAsync, canLogin);
        LoadSavedSessionCommand = ReactiveCommand.CreateFromTask(LoadSavedSessionAsync);
    }

    // 바인딩 속성들
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

    private bool _rememberMe;
    public bool RememberMe
    {
        get => _rememberMe;
        set => this.RaiseAndSetIfChanged(ref _rememberMe, value);
    }

    private string _errorMessage = "";
    public string ErrorMessage
    {
        get => _errorMessage;
        private set => this.RaiseAndSetIfChanged(ref _errorMessage, value);
    }

    private bool _isBusy;
    public bool IsBusy
    {
        get => _isBusy;
        private set => this.RaiseAndSetIfChanged(ref _isBusy, value);
    }

    public ReactiveCommand<Unit, Unit> LoginCommand { get; }
    public ReactiveCommand<Unit, Unit> LoadSavedSessionCommand { get; }

    private async Task LoginAsync()
    {
        try
        {
            IsBusy = true;
            ErrorMessage = "";

            var session = await _auth.LoginAsync(Username, Password);
            if (session == null)
            {
                ErrorMessage = "아이디 또는 비밀번호가 잘못되었습니다.";
                return;
            }

            await _store.SaveAsync(session, RememberMe);
            _onSuccess(); // 메인 화면으로 전환
        }
        catch (Exception ex)
        {
            ErrorMessage = $"오류 발생: {ex.Message}";
        }
        finally
        {
            IsBusy = false;
            Password = ""; // 메모리에서 비밀번호 제거
        }
    }

    private async Task LoadSavedSessionAsync()
    {
        var session = await _store.LoadAsync();
        if (session != null && !session.IsExpired(DateTimeOffset.UtcNow))
        {
            _state.CurrentUser = session;
            _onSuccess();
        }
    }
}
```

## 로그인 View (XAML)

View는 ViewModel의 속성과 명령을 바인딩한다. 비밀번호 입력을 위해 `PasswordChar` 속성을 사용하거나, 별도 컨트롤을 사용한다.

```xml
<UserControl xmlns="https://github.com/avaloniaui"
             x:Class="MyApp.Views.LoginView">
    <StackPanel Margin="30" Spacing="10">
        <TextBox Watermark="아이디" Text="{Binding Username}" />
        <TextBox Watermark="비밀번호" Text="{Binding Password}" PasswordChar="*" />
        <CheckBox Content="자동 로그인" IsChecked="{Binding RememberMe}" />
        <Button Content="로그인" Command="{Binding LoginCommand}" />
        <ProgressBar IsIndeterminate="True" IsVisible="{Binding IsBusy}" Height="6" />
        <TextBlock Text="{Binding ErrorMessage}" Foreground="Red" TextWrapping="Wrap" />
    </StackPanel>
</UserControl>
```

## 앱 초기화와 화면 전환

`App.axaml.cs`에서 의존성 주입 컨테이너를 구성하고, 로그인 화면을 띄운다. 자동 로그인 시도는 ViewModel의 명령을 실행한다.

```csharp
public partial class App : Application
{
    public static IServiceProvider Services { get; private set; } = null!;

    public override void OnFrameworkInitializationCompleted()
    {
        var services = new ServiceCollection();
        // 서비스 등록
        services.AddSingleton<AppState>();
        services.AddSingleton<ITokenStore, EncryptedJsonTokenStore>();
        services.AddHttpClient<IAuthService, AuthService>(client =>
        {
            client.BaseAddress = new Uri("https://api.example.com");
        });
        Services = services.BuildServiceProvider();

        ShowLoginWindow();
        base.OnFrameworkInitializationCompleted();
    }

    private void ShowLoginWindow()
    {
        var state = Services.GetRequiredService<AppState>();
        var auth = Services.GetRequiredService<IAuthService>();
        var store = Services.GetRequiredService<ITokenStore>();

        var vm = new LoginViewModel(auth, store, state, ShowMainWindow);
        var loginWindow = new Window { Content = new LoginView(), DataContext = vm };
        loginWindow.Show();

        // 자동 로그인 시도
        _ = vm.LoadSavedSessionCommand.Execute();
    }

    private void ShowMainWindow()
    {
        var mainWindow = new Window
        {
            DataContext = Services.GetRequiredService<MainViewModel>(),
            Content = new MainView()
        };
        // 기존 로그인 창 닫기
        var windows = Application.Current?.Windows.ToArray() ?? Array.Empty<Window>();
        foreach (var w in windows)
            if (w.DataContext is LoginViewModel) w.Close();

        mainWindow.Show();
        if (ApplicationLifetime is IClassicDesktopStyleApplicationLifetime desktop)
            desktop.MainWindow = mainWindow;
    }
}
```

## 로그아웃 처리

메인 ViewModel에서 로그아웃 명령을 제공한다. `IAuthService.LogoutAsync`는 로컬 상태를 초기화하고 저장된 토큰을 삭제한다. 로그아웃 후에는 다시 로그인 화면으로 전환한다.

```csharp
public class MainViewModel : ReactiveObject
{
    private readonly IAuthService _auth;
    private readonly AppState _state;

    public MainViewModel(IAuthService auth, AppState state)
    {
        _auth = auth;
        _state = state;
        LogoutCommand = ReactiveCommand.CreateFromTask(LogoutAsync);
    }

    public string Welcome => _state.CurrentUser?.Username ?? "게스트";

    public ReactiveCommand<Unit, Unit> LogoutCommand { get; }

    private async Task LogoutAsync()
    {
        await _auth.LogoutAsync();
        // App에서 로그인 화면으로 전환하도록 처리
    }
}
```

`AuthService.LogoutAsync`:

```csharp
public async Task LogoutAsync(CancellationToken ct = default)
{
    _state.CurrentUser = null;
    await _store.ClearAsync(ct);
}
```

## 보안 체크리스트

| 항목 | 설명 |
|------|------|
| TLS 사용 | 모든 네트워크 통신은 HTTPS로 암호화 |
| 비밀번호 메모리 | 로그인 후 비밀번호 문자열을 즉시 지움 |
| 토큰 저장 | AES-256-GCM으로 암호화하여 저장 |
| 실패 횟수 제한 | 일정 횟수 이상 실패 시 지연 또는 계정 잠금 고려 |
| 토큰 만료 처리 | 만료 시간을 기준으로 갱신 또는 재로그인 유도 |

## 단위 테스트 (간단 예시)

ViewModel은 의존성을 Mock으로 대체하여 테스트할 수 있다. 아래는 Moq를 사용한 예시다.

```csharp
[Fact]
public async Task Login_Success_UpdatesAppState_AndCallsOnSuccess()
{
    // Arrange
    var state = new AppState();
    var store = new Mock<ITokenStore>();
    bool navigated = false;

    var auth = new Mock<IAuthService>();
    auth.Setup(a => a.LoginAsync("user", "pass", default))
        .ReturnsAsync(new UserSession { Username = "user", AccessToken = "token" });

    var vm = new LoginViewModel(auth.Object, store.Object, state, () => navigated = true)
    {
        Username = "user",
        Password = "pass"
    };

    // Act
    await vm.LoginCommand.Execute();

    // Assert
    Assert.True(state.IsLoggedIn);
    Assert.True(navigated);
    store.Verify(s => s.SaveAsync(It.IsAny<UserSession>(), false, default), Times.Once);
}
```

## 정리 및 확장

이 설계는 기본적인 로그인/인증 구조를 MVVM 원칙에 맞게 구현한 것이다. 추가로 고려할 사항은 다음과 같다.

- **토큰 갱신**: 만료 전에 자동으로 갱신하거나 401 응답 시 재시도하는 로직을 추가할 수 있다.
- **역할 기반 권한**: 사용자 권한을 `AppState`에 포함하여 UI에 반영한다.
- **2단계 인증**: 로그인 후 추가 인증 화면을 도입할 수 있다.
- **보안 강화**: 비밀번호 해싱, 토큰 서명 검증 등은 서버 측에서 처리한다.

MVVM 패턴과 의존성 주입을 활용하면 인증 관련 로직을 테스트 가능하고 유지보수하기 쉬운 구조로 유지할 수 있다.