---
layout: post
title: Avalonia - MVVM 로그인 및 인증 구조 설계
date: 2025-02-03 20:20:23 +0900
category: Avalonia
---
# Avalonia MVVM 로그인·인증 구조 설계

## 핵심 목표와 전체 플로우

- 로그인 화면 분리: `LoginView` / `LoginViewModel`
- 인증 처리: API 연동(아이디/패스워드 → JWT/세션 토큰), `IAuthService`
- 상태 관리: 성공 시 `AppState`(전역) 업데이트 → 메인 화면 전환
- 보안: 토큰 암호화 저장, 자동로그인, 만료/갱신, 실패 횟수 제한, 예외 처리
- 테스트: ViewModel 단위 테스트·Mock 서비스

**흐름 요약**

```
사용자 입력 → LoginCommand → IAuthService.LoginAsync
  → 성공: UserSession 생성/토큰 저장(암호화) → AppState 반영 → 화면 전환
  → 실패: 에러 메시지/재시도 제어
```

---

## 프로젝트 구조(확장)

```
MyApp/
├── App.axaml / App.axaml.cs
├── Models/
│   ├── UserSession.cs
│   └── AuthResult.cs               // 서버 응답 DTO (JWT/Refresh 포함)
├── Services/
│   ├── IAuthService.cs
│   ├── AuthService.cs              // API 연동
│   ├── ITokenStore.cs
│   ├── EncryptedJsonTokenStore.cs  // 토큰 로컬 암호화 저장(자동로그인)
│   ├── AuthHttpMessageHandler.cs   // HTTP Authorization 주입/갱신
│   └── AesCryptoService.cs         // AES-256 GCM 암호화
├── State/
│   └── AppState.cs
├── ViewModels/
│   ├── LoginViewModel.cs
│   └── MainViewModel.cs
├── Views/
│   ├── LoginView.axaml
│   └── MainView.axaml
└── Tests/
    └── LoginViewModelTests.cs
```

---

## 모델

### 사용자 세션

```csharp
// Models/UserSession.cs
public class UserSession
{
    public string Username { get; set; } = "";
    public string AccessToken { get; set; } = ""; // JWT 등
    public string? RefreshToken { get; set; }
    public DateTimeOffset IssuedAt { get; set; }
    public DateTimeOffset ExpiresAt { get; set; }

    public bool IsExpired(DateTimeOffset now) => now >= ExpiresAt;

    public TimeSpan TimeToExpire(DateTimeOffset now) => ExpiresAt - now;
}
```

### 서버 응답 DTO(예시)

```csharp
// Models/AuthResult.cs
public class AuthResult
{
    public string AccessToken { get; set; } = "";
    public string? RefreshToken { get; set; }
    public int ExpiresInSeconds { get; set; }
    public string Username { get; set; } = "";
}
```

---

## 전역 상태(AppState)

```csharp
// State/AppState.cs
using ReactiveUI;

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

- ViewModel/서비스에서 DI로 주입받아 **현재 로그인 상태**를 공유한다.

---

## 토큰 저장소(자동로그인/암호화)

자동로그인을 위해 토큰을 로컬에 저장할 수 있다. **평문 저장 금지** → AES-256-GCM으로 암호화.

```csharp
// Services/ITokenStore.cs
public interface ITokenStore
{
    Task SaveAsync(UserSession session, bool rememberMe, CancellationToken ct = default);
    Task<UserSession?> LoadAsync(CancellationToken ct = default);
    Task ClearAsync(CancellationToken ct = default);
}
```

```csharp
// Services/EncryptedJsonTokenStore.cs
using System.Text.Json;

public sealed class EncryptedJsonTokenStore : ITokenStore
{
    private readonly AesCryptoService _crypto;
    private readonly string _path;

    public EncryptedJsonTokenStore(AesCryptoService crypto)
    {
        _crypto = crypto;
        var dir = Path.Combine(
            Environment.GetFolderPath(Environment.SpecialFolder.ApplicationData),
            "MyApp");
        Directory.CreateDirectory(dir);
        _path = Path.Combine(dir, "session.json");
    }

    public async Task SaveAsync(UserSession session, bool rememberMe, CancellationToken ct = default)
    {
        if (!rememberMe)
        {
            // 메모리 내 유지. 파일은 삭제
            if (File.Exists(_path)) File.Delete(_path);
            return;
        }

        var plain = JsonSerializer.Serialize(session);
        var cipher = _crypto.Encrypt(plain);
        await File.WriteAllTextAsync(_path, cipher, ct);
    }

    public async Task<UserSession?> LoadAsync(CancellationToken ct = default)
    {
        if (!File.Exists(_path)) return null;
        var cipher = await File.ReadAllTextAsync(_path, ct);
        var plain = _crypto.Decrypt(cipher);
        return JsonSerializer.Deserialize<UserSession>(plain);
    }

    public Task ClearAsync(CancellationToken ct = default)
    {
        if (File.Exists(_path)) File.Delete(_path);
        return Task.CompletedTask;
    }
}
```

> **키 관리 주의**: 예제는 단순 키 주입. 실제 서비스는 OS 보호(Windows DPAPI, macOS Keychain, Linux Secret Service 등) 또는 보안 모듈 사용을 검토.

---

## 인증 서비스(IAuthService)

### 인터페이스

```csharp
// Services/IAuthService.cs
public interface IAuthService
{
    Task<UserSession?> LoginAsync(string username, string password, CancellationToken ct = default);
    Task<UserSession?> RefreshAsync(string refreshToken, CancellationToken ct = default);
    Task LogoutAsync(CancellationToken ct = default);
}
```

### 구현(HTTP API 연동 예시)

```csharp
// Services/AuthService.cs
using System.Net.Http.Json;

public sealed class AuthService : IAuthService
{
    private readonly HttpClient _http;
    private readonly AppState _state;
    private readonly ITokenStore _store;

    public AuthService(HttpClient http, AppState state, ITokenStore store)
    {
        _http = http; _state = state; _store = store;
    }

    public async Task<UserSession?> LoginAsync(string username, string password, CancellationToken ct = default)
    {
        var payload = new { username, password };
        using var res = await _http.PostAsJsonAsync("/api/auth/login", payload, ct);
        if (!res.IsSuccessStatusCode) return null;

        var dto = await res.Content.ReadFromJsonAsync<AuthResult>(cancellationToken: ct);
        if (dto == null) return null;

        var now = DateTimeOffset.UtcNow;
        var session = new UserSession
        {
            Username   = dto.Username,
            AccessToken = dto.AccessToken,
            RefreshToken = dto.RefreshToken,
            IssuedAt   = now,
            ExpiresAt  = now.AddSeconds(dto.ExpiresInSeconds)
        };
        _state.CurrentUser = session;
        return session;
    }

    public async Task<UserSession?> RefreshAsync(string refreshToken, CancellationToken ct = default)
    {
        var payload = new { refreshToken };
        using var res = await _http.PostAsJsonAsync("/api/auth/refresh", payload, ct);
        if (!res.IsSuccessStatusCode) return null;

        var dto = await res.Content.ReadFromJsonAsync<AuthResult>(cancellationToken: ct);
        if (dto == null) return null;

        var now = DateTimeOffset.UtcNow;
        var session = new UserSession
        {
            Username   = dto.Username,
            AccessToken = dto.AccessToken,
            RefreshToken = dto.RefreshToken ?? refreshToken,
            IssuedAt   = now,
            ExpiresAt  = now.AddSeconds(dto.ExpiresInSeconds)
        };
        _state.CurrentUser = session;
        return session;
    }

    public async Task LogoutAsync(CancellationToken ct = default)
    {
        // 서버에 세션 종료를 알릴 수 있음(선택)
        _state.CurrentUser = null;
        await _store.ClearAsync(ct);
    }
}
```

---

## HTTP Authorization 자동 주입/갱신

401 수신 시 Refresh → 재시도 패턴(단순화 예시).

```csharp
// Services/AuthHttpMessageHandler.cs
using System.Net;

public sealed class AuthHttpMessageHandler : DelegatingHandler
{
    private readonly AppState _state;
    private readonly IAuthService _auth;

    public AuthHttpMessageHandler(AppState state, IAuthService auth)
    {
        _state = state; _auth = auth;
    }

    protected override async Task<HttpResponseMessage> SendAsync(HttpRequestMessage request, CancellationToken ct)
    {
        var session = _state.CurrentUser;
        if (session is not null && !session.IsExpired(DateTimeOffset.UtcNow))
        {
            request.Headers.Authorization = new("Bearer", session.AccessToken);
        }

        var res = await base.SendAsync(request, ct);
        if (res.StatusCode != HttpStatusCode.Unauthorized) return res;

        // 401인 경우 Refresh 시도
        if (session?.RefreshToken is null) return res;

        var refreshed = await _auth.RefreshAsync(session.RefreshToken, ct);
        if (refreshed is null) return res;

        // 재시도
        request = Clone(request);
        request.Headers.Authorization = new("Bearer", refreshed.AccessToken);
        res.Dispose();
        return await base.SendAsync(request, ct);
    }

    private static HttpRequestMessage Clone(HttpRequestMessage req)
    {
        var clone = new HttpRequestMessage(req.Method, req.RequestUri)
        {
            Content = req.Content,
            Version = req.Version
        };
        foreach (var h in req.Headers)
            clone.Headers.TryAddWithoutValidation(h.Key, h.Value);
        foreach (var p in req.Properties)
            clone.Properties[p.Key] = p.Value;
        return clone;
    }
}
```

> **주의**: 멱등성·재시도 정책·동시 갱신 경합 등은 실제 서비스에 맞게 보완.

---

## 로그인 ViewModel (UI/상태/자동로그인/검증)

```csharp
// ViewModels/LoginViewModel.cs
using ReactiveUI;
using System.Reactive;
using System.Reactive.Linq;

public sealed class LoginViewModel : ReactiveObject
{
    private readonly IAuthService _auth;
    private readonly ITokenStore _store;
    private readonly AppState _state;
    private readonly Action _onSuccess;

    private string _username = "";
    private string _password = "";
    private bool _rememberMe = false;
    private string _error = "";
    private bool _isBusy = false;
    private int _failedCount = 0;

    public LoginViewModel(IAuthService auth, ITokenStore store, AppState state, Action onSuccess)
    {
        _auth = auth; _store = store; _state = state; _onSuccess = onSuccess;

        var canLogin = this.WhenAnyValue(
            x => x.Username, x => x.Password, x => x.IsBusy,
            (u, p, b) => !b && !string.IsNullOrWhiteSpace(u) && !string.IsNullOrWhiteSpace(p));

        LoginCommand = ReactiveCommand.CreateFromTask(LoginAsync, canLogin);
        LoadSavedSessionCommand = ReactiveCommand.CreateFromTask(LoadSavedSessionAsync);
    }

    public string Username
    {
        get => _username;
        set => this.RaiseAndSetIfChanged(ref _username, value);
    }

    public string Password
    {
        get => _password;
        set => this.RaiseAndSetIfChanged(ref _password, value);
    }

    public bool RememberMe
    {
        get => _rememberMe;
        set => this.RaiseAndSetIfChanged(ref _rememberMe, value);
    }

    public string ErrorMessage
    {
        get => _error;
        private set => this.RaiseAndSetIfChanged(ref _error, value);
    }

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
            if (session is null)
            {
                _failedCount++;
                ErrorMessage = "아이디 또는 비밀번호가 잘못되었습니다.";
                if (_failedCount >= 5) ErrorMessage += " 잠시 후 다시 시도하세요.";
                return;
            }

            await _store.SaveAsync(session, RememberMe);
            _onSuccess();
        }
        catch (Exception ex)
        {
            ErrorMessage = $"로그인 중 오류가 발생했습니다: {ex.Message}";
        }
        finally
        {
            IsBusy = false;
            Password = ""; // 보안: 메모리 잔류 최소화
        }
    }

    private async Task LoadSavedSessionAsync()
    {
        try
        {
            IsBusy = true;
            var saved = await _store.LoadAsync();
            if (saved is not null && !saved.IsExpired(DateTimeOffset.UtcNow))
            {
                _state.CurrentUser = saved;
                _onSuccess();
            }
        }
        finally { IsBusy = false; }
    }
}
```

---

## 로그인 View (Password 마스킹, 진행 UI)

```xml
<!-- Views/LoginView.axaml -->
<UserControl xmlns="https://github.com/avaloniaui"
             x:Class="MyApp.Views.LoginView">
  <StackPanel Margin="30" Spacing="10">
    <TextBlock Text="로그인" FontSize="24" Margin="0,0,0,12"/>

    <TextBox Watermark="아이디" Text="{Binding Username}" />

    <!-- Avalonia는 PasswordBox 또는 TextBox+PasswordChar 모두 가능(버전에 따라) -->
    <TextBox Watermark="비밀번호" Text="{Binding Password}" PasswordChar="*" />

    <CheckBox Content="자동 로그인" IsChecked="{Binding RememberMe}" />

    <Button Content="로그인"
            Command="{Binding LoginCommand}"
            IsEnabled="{Binding LoginCommand.CanExecute}"
            />

    <ProgressBar IsIndeterminate="True"
                 IsVisible="{Binding IsBusy}" Height="6"/>

    <TextBlock Text="{Binding ErrorMessage}" Foreground="Red" TextWrapping="Wrap"/>
  </StackPanel>
</UserControl>
```

---

## App 초기화와 화면 전환

```csharp
// App.axaml.cs (핵심 부분)
using Microsoft.Extensions.DependencyInjection;

public class App : Application
{
    public static IServiceProvider Services { get; private set; } = default!;

    public override void OnFrameworkInitializationCompleted()
    {
        var sc = new ServiceCollection();
        ConfigureServices(sc);
        Services = sc.BuildServiceProvider();

        ShowLoginWindow(); // 자동로그인은 LoginViewModel.LoadSavedSessionCommand에서 처리
        base.OnFrameworkInitializationCompleted();
    }

    private void ConfigureServices(IServiceCollection s)
    {
        // 간단한 키 예시(32바이트). 운영 환경은 안전한 키 관리 필수.
        var key = Enumerable.Repeat((byte)0x11, 32).ToArray();
        s.AddSingleton(new AesCryptoService(key));

        s.AddSingleton<AppState>();
        s.AddSingleton<ITokenStore, EncryptedJsonTokenStore>();

        // HttpClient + DelegatingHandler
        s.AddTransient<AuthHttpMessageHandler>();
        s.AddHttpClient<IAuthService, AuthService>(client =>
        {
            client.BaseAddress = new Uri("https://api.example.com");
            client.Timeout = TimeSpan.FromSeconds(15);
        }).AddHttpMessageHandler<AuthHttpMessageHandler>();

        s.AddSingleton<MainViewModel>();
    }

    private void ShowLoginWindow()
    {
        var state = Services.GetRequiredService<AppState>();
        var auth = Services.GetRequiredService<IAuthService>();
        var store = Services.GetRequiredService<ITokenStore>();

        var vm = new LoginViewModel(auth, store, state, onSuccess: ShowMainWindow);
        var win = new Window { Content = new Views.LoginView(), DataContext = vm };
        win.Show();

        // 자동로그인 시도
        _ = vm.LoadSavedSessionCommand.Execute();
    }

    private void ShowMainWindow()
    {
        var main = new Window
        {
            DataContext = Services.GetRequiredService<MainViewModel>(),
            Content = new Views.MainView()
        };

        // 이전 로그인 창 닫기
        foreach (var w in Application.Current.Windows.ToArray())
            if (w.DataContext is LoginViewModel) w.Close();

        main.Show();
        if (ApplicationLifetime is IClassicDesktopStyleApplicationLifetime life)
            life.MainWindow = main;
    }
}
```

---

## 메인 화면에서 로그아웃 처리

```csharp
// ViewModels/MainViewModel.cs
using ReactiveUI;
using System.Reactive;

public sealed class MainViewModel : ReactiveObject
{
    private readonly IAuthService _auth;
    private readonly AppState _state;

    public MainViewModel(IAuthService auth, AppState state)
    {
        _auth = auth; _state = state;
        LogoutCommand = ReactiveCommand.CreateFromTask(async () =>
        {
            await _auth.LogoutAsync();
            // 앱 정책에 따라 로그인 화면으로 전환(여기서는 App에서 처리)
        });
    }

    public string Welcome => _state.CurrentUser is null
        ? "게스트"
        : $"{_state.CurrentUser.Username} 님 환영합니다";

    public ReactiveCommand<Unit, Unit> LogoutCommand { get; }
}
```

```xml
<!-- Views/MainView.axaml -->
<UserControl xmlns="https://github.com/avaloniaui"
             x:Class="MyApp.Views.MainView">
  <StackPanel Margin="20" Spacing="12">
    <TextBlock Text="{Binding Welcome}" FontSize="18"/>
    <Button Content="로그아웃" Command="{Binding LogoutCommand}"/>
  </StackPanel>
</UserControl>
```

> 로그아웃 후 로그인 화면으로의 전환은 `App`에서 `ShowLoginWindow()`를 재호출하도록 설계할 수 있다. (MessageBus/NavigationService와 연동해도 된다.)

---

## 보안·신뢰성 체크리스트

- 비밀번호는 **네트워크 전송 시 TLS** 필수, 평문 로그에 남기지 말 것
- 토큰 저장은 **암호화(AES-256-GCM)** + 파일 권한 최소화
- 자동로그인(remember me)은 사용자 선택 옵션 + 토큰 만료 고려
- 로그인 실패 **횟수 제한**, 지연(백오프) 도입
- 토큰 만료 전 **사전 갱신**(유휴시점, 포그라운드 전환 등)
- 401 수신 시 **Refresh → 재시도** 단, 루프 방지(최대 1회)
- 예외/네트워크 장애 로그(Serilog, NLog 등) + 사용자 안내

---

## 단위 테스트(예시)

```csharp
// Tests/LoginViewModelTests.cs
using Xunit;
using FluentAssertions;
using Moq;

public class LoginViewModelTests
{
    [Fact]
    public async Task Login_Succeeds_UpdatesAppState_AndCallsOnSuccess()
    {
        var state = new AppState();
        var store = new Mock<ITokenStore>();
        bool navigated = false;

        var auth = new Mock<IAuthService>();
        auth.Setup(a => a.LoginAsync("admin", "1234", default))
            .ReturnsAsync(new UserSession
            {
                Username = "admin",
                AccessToken = "jwt",
                ExpiresAt = DateTimeOffset.UtcNow.AddMinutes(30)
            });

        var vm = new LoginViewModel(auth.Object, store.Object, state, () => navigated = true)
        {
            Username = "admin",
            Password = "1234",
            RememberMe = true
        };

        await vm.LoginCommand.Execute();

        state.IsLoggedIn.Should().BeTrue();
        navigated.Should().BeTrue();
        store.Verify(s => s.SaveAsync(It.IsAny<UserSession>(), true, default), Times.Once);
    }

    [Fact]
    public async Task Login_Fails_ShowsError_AndDoesNotNavigate()
    {
        var state = new AppState();
        var store = new Mock<ITokenStore>();
        bool navigated = false;

        var auth = new Mock<IAuthService>();
        auth.Setup(a => a.LoginAsync(It.IsAny<string>(), It.IsAny<string>(), default))
            .ReturnsAsync((UserSession?)null);

        var vm = new LoginViewModel(auth.Object, store.Object, state, () => navigated = true)
        {
            Username = "x",
            Password = "y"
        };

        await vm.LoginCommand.Execute();

        state.IsLoggedIn.Should().BeFalse();
        navigated.Should().BeFalse();
        vm.ErrorMessage.Should().NotBeNullOrWhiteSpace();
    }
}
```

---

## 고급 확장

- **역할/권한(Role/Policy)**: AccessToken의 클레임 파싱 → AppState에 `IsAdmin` 등 노출
- **2FA/MFA**: 1차 비밀번호 성공 후 OTP 화면으로 분기
- **SAML/OIDC**: 외부 브라우저/임베디드 WebView 로그인(리디렉션/딥링크)
- **오프라인 모드**: 만료 전 캐시된 세션으로 제한 기능만 허용
- **세션 타임아웃/유휴감지**: 입력 이벤트 없을 때 경고 후 자동 로그아웃

---

## 운영 팁

- **서버 시간과의 오차**(Clock Skew) 보정 → 만료 60초 전 미리 갱신
- **HttpClient 재사용**: DI 팩토리로 생성, 소켓 핸들 고갈 방지
- **로깅**: 성공/실패/갱신/로그아웃 이벤트 구분 로그
- **프롬프트 보호**: Password는 ViewModel에 오래 보관하지 말고 즉시 폐기

---

## 요약

| 항목 | 핵심 포인트 |
|------|-------------|
| MVVM/DI | `IAuthService`/`ITokenStore`로 관심사 분리 |
| 전역 상태 | `AppState`로 로그인 상태·세션 공유 |
| 자동 로그인 | 토큰 암호화 저장 + 시작 시 로드 |
| 토큰 갱신 | 401 처리/만료 전 갱신, 재시도 1회 |
| UI | Busy/에러/검증/기본 보호(PasswordChar) |
| 테스트 | ViewModel 단위 테스트로 회귀 방지 |

---

## AES-256 GCM 서비스(요지)

```csharp
// Services/AesCryptoService.cs
using System.Security.Cryptography;
using System.Text;

public sealed class AesCryptoService
{
    private readonly byte[] _key;
    public AesCryptoService(byte[] key) => _key = key;

    public string Encrypt(string plain)
    {
        if (string.IsNullOrEmpty(plain)) return plain;

        using var aes = new AesGcm(_key);
        var nonce = RandomNumberGenerator.GetBytes(12);
        var bytes = Encoding.UTF8.GetBytes(plain);
        var cipher = new byte[bytes.Length];
        var tag = new byte[16];
        aes.Encrypt(nonce, bytes, cipher, tag);
        return Convert.ToBase64String(nonce.Concat(cipher).Concat(tag).ToArray());
    }

    public string Decrypt(string cipherText)
    {
        if (string.IsNullOrEmpty(cipherText)) return cipherText;

        var raw = Convert.FromBase64String(cipherText);
        var nonce = raw[..12];
        var tag = raw[^16..];
        var cipher = raw[12..^16];

        using var aes = new AesGcm(_key);
        var plain = new byte[cipher.Length];
        aes.Decrypt(nonce, cipher, tag, plain);
        return Encoding.UTF8.GetString(plain);
    }
}
```

---

## 간단 수식(만료 전 갱신 정책)

만료시각을 \( T_\text{exp} \), 현재시각을 \( t \), 스큐 여유를 \( \delta \)라 하면,
**갱신 조건**은 다음과 같이 둘 수 있다:

$$
t \ge T_\text{exp} - \delta
$$

예: \( \delta = 60 \)초일 때, 만료 60초 전에 Refresh를 시도한다.
