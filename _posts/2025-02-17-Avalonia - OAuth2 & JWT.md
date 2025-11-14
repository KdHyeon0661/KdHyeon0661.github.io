---
layout: post
title: Avalonia - OAuth2 & JWT 인증
date: 2025-02-17 20:20:23 +0900
category: Avalonia
---
# Avalonia MVVM + OAuth2 / JWT 인증 구조화

- OAuth2 Password/Resource Owner Password(레거시)·Authorization Code(PKCE)·Refresh Token 로테이션 전략
- `DelegatingHandler` 체인으로 **자동 토큰 첨부/만료 감지/Refresh 후 재시도**까지
- **안전한 토큰 저장소 추상화**(`ITokenStore`)와 Windows DPAPI/Linux SecretService(macOS Keychain) 예시
- **동시성**: 다중 API 호출 시 Refresh를 **단 1회**로 보장하는 Single-Flight Gate
- **시계 드리프트**(clock skew) 고려한 선제적 갱신 T−Δ
- **권한(Scopes/Roles)** 기반 UI 제어, 401/403/429/5xx 재시도/로그아웃 플로우
- **테스트 전략**: `HttpMessageHandler` 목킹, 만료/401/Refresh 실패 시나리오
- **DI/구성**: `HttpClientFactory`(대안), `IAuthService`·`IAuthState`·`ITokenStore`의 계약
- **보안 주의점**: JWT 클레임 신뢰경계, XSS/Clipboard·로그 출력, Refresh Rotation/Replay 대응
- **확장**: OIDC Discovery, JWKs 키 회전 대응(클라이언트 검증이 필요한 경우의 참고), SSO 로그아웃 등

---

## 디렉터리 구조(확장판)

```
MyApp/
├── App.axaml / App.axaml.cs
├── State/
│   └── AppAuthState.cs                // 런타임 인증 상태(메모리)
├── Services/
│   ├── IAuthService.cs                // 로그인/갱신/로그아웃 계약
│   ├── AuthService.cs                 // API 연동 구현
│   ├── ITokenStore.cs                 // 안전한 토큰 저장소 계약
│   ├── InMemoryTokenStore.cs          // 메모리 저장(기본)
│   ├── DpapiTokenStore.cs             // Windows DPAPI 예시(선택)
│   ├── SecretServiceTokenStore.cs     // Linux SecretService 예시(선택)
│   ├── AuthenticatedHttpHandler.cs    // Authorization 주입 + 만료/401 자동 처리
│   ├── RefreshGate.cs                 // Single-flight 동시 갱신 게이트
│   └── BackoffPolicy.cs               // 429/5xx 재시도 지수백오프
├── ViewModels/
│   ├── LoginViewModel.cs
│   ├── ShellViewModel.cs              // 앱 루트(VM) — 인증 전/후 화면 전환
│   └── ClaimsAwareViewModel.cs        // 예: Role/Scope에 따라 UI 제어
├── Views/
│   ├── LoginView.axaml
│   └── ShellView.axaml
└── Models/
    └── AuthModels.cs                  // DTO: AuthResponse, Introspection 등
```

---

## 런타임 인증 상태 — AppAuthState

- ViewModel/UI와 서비스가 공유하는 **현재 인증 맥락**.
- **엑세스 토큰/리프레시 토큰/만료시각/스코프/사용자 식별자** 등.

```csharp
// State/AppAuthState.cs
public sealed class AppAuthState : ReactiveUI.ReactiveObject
{
    private string? _accessToken;
    private string? _refreshToken;
    private DateTimeOffset _accessTokenExpiresAtUtc;
    private string[] _scopes = Array.Empty<string>();
    private string? _subject; // sub / userId
    private string[] _roles = Array.Empty<string>();

    public string? AccessToken
    {
        get => _accessToken;
        set => this.RaiseAndSetIfChanged(ref _accessToken, value);
    }

    public string? RefreshToken
    {
        get => _refreshToken;
        set => this.RaiseAndSetIfChanged(ref _refreshToken, value);
    }

    public DateTimeOffset AccessTokenExpiresAtUtc
    {
        get => _accessTokenExpiresAtUtc;
        set => this.RaiseAndSetIfChanged(ref _accessTokenExpiresAtUtc, value);
    }

    public string[] Scopes
    {
        get => _scopes;
        set => this.RaiseAndSetIfChanged(ref _scopes, value);
    }

    public string[] Roles
    {
        get => _roles;
        set => this.RaiseAndSetIfChanged(ref _roles, value);
    }

    public string? Subject
    {
        get => _subject;
        set => this.RaiseAndSetIfChanged(ref _subject, value);
    }

    // 만료 판정 시 clock skew(예: 30초)를 둬서 선제 갱신
    public bool IsAccessTokenNearExpiry(TimeSpan skew)
        => AccessToken is not null && DateTimeOffset.UtcNow >= AccessTokenExpiresAtUtc - skew;

    public bool IsAuthenticated
        => !string.IsNullOrEmpty(AccessToken) && DateTimeOffset.UtcNow < AccessTokenExpiresAtUtc;

    public void Clear()
    {
        AccessToken = null;
        RefreshToken = null;
        AccessTokenExpiresAtUtc = DateTimeOffset.MinValue;
        Scopes = Array.Empty<string>();
        Roles = Array.Empty<string>();
        Subject = null;
    }
}
```

---

## 토큰 저장소 추상화 — ITokenStore

- **보안 수준**에 따라 저장소를 교체할 수 있게 추상화.
- 앱 메모리(기본), Windows DPAPI, Linux SecretService, macOS Keychain 등을 선택적으로 구현.

```csharp
// Services/ITokenStore.cs
public interface ITokenStore
{
    Task SaveAsync(string accessToken, DateTimeOffset expiresAtUtc, string? refreshToken);
    Task<(string? accessToken, DateTimeOffset expiresAtUtc, string? refreshToken)> LoadAsync();
    Task ClearAsync();
}
```

### InMemoryTokenStore (기본)

```csharp
// Services/InMemoryTokenStore.cs
public sealed class InMemoryTokenStore : ITokenStore
{
    private string? _access;
    private string? _refresh;
    private DateTimeOffset _exp;

    public Task SaveAsync(string accessToken, DateTimeOffset expiresAtUtc, string? refreshToken)
    {
        _access = accessToken;
        _refresh = refreshToken;
        _exp    = expiresAtUtc;
        return Task.CompletedTask;
    }

    public Task<(string? accessToken, DateTimeOffset expiresAtUtc, string? refreshToken)> LoadAsync()
        => Task.FromResult((_access, _exp, _refresh));

    public Task ClearAsync()
    {
        _access = null;
        _refresh = null;
        _exp = DateTimeOffset.MinValue;
        return Task.CompletedTask;
    }
}
```

### DPAPI(Windows) 예시(요점만)

```csharp
// Services/DpapiTokenStore.cs (Windows 전용)
public sealed class DpapiTokenStore : ITokenStore
{
    private readonly string _path;

    public DpapiTokenStore(string filePath)
    {
        _path = filePath;
        Directory.CreateDirectory(Path.GetDirectoryName(_path)!);
    }

    public async Task SaveAsync(string accessToken, DateTimeOffset expiresAtUtc, string? refreshToken)
    {
        var obj = new
        {
            access = accessToken,
            expUtc = expiresAtUtc,
            refresh = refreshToken
        };
        var json = System.Text.Json.JsonSerializer.Serialize(obj);
        var plain = System.Text.Encoding.UTF8.GetBytes(json);
        var cipher = ProtectedData.Protect(plain, optionalEntropy: null, DataProtectionScope.CurrentUser);
        await File.WriteAllBytesAsync(_path, cipher);
    }

    public async Task<(string? accessToken, DateTimeOffset expiresAtUtc, string? refreshToken)> LoadAsync()
    {
        if (!File.Exists(_path)) return (null, DateTimeOffset.MinValue, null);
        var cipher = await File.ReadAllBytesAsync(_path);
        var plain = ProtectedData.Unprotect(cipher, null, DataProtectionScope.CurrentUser);
        var json = System.Text.Encoding.UTF8.GetString(plain);
        var doc = System.Text.Json.JsonDocument.Parse(json).RootElement;
        var access = doc.GetProperty("access").GetString();
        var expUtc = doc.GetProperty("expUtc").GetDateTimeOffset();
        var refresh = doc.GetProperty("refresh").GetString();
        return (access, expUtc, refresh);
    }

    public Task ClearAsync()
    {
        if (File.Exists(_path)) File.Delete(_path);
        return Task.CompletedTask;
    }
}
```

> Linux·macOS의 안전 저장은 플랫폼 비의존 라이브러리 또는 SecretService/Keychain 바인딩을 사용하는 방식을 권장.

---

## Auth DTO와 계약 — Models/AuthModels.cs, IAuthService.cs

```csharp
// Models/AuthModels.cs
public sealed class AuthResponse
{
    public string access_token { get; init; } = string.Empty;
    public string? refresh_token { get; init; }
    public int expires_in { get; init; }
    public string? scope { get; init; }             // "read write"
    public string? token_type { get; init; }        // "Bearer"
    public string? id_token { get; init; }          // (OIDC) 선택
}

public sealed class RefreshRequest { public string refresh_token { get; init; } = string.Empty; }
public sealed class LoginRequest   { public string username { get; init; } = string.Empty; public string password { get; init; } = string.Empty; }
```

```csharp
// Services/IAuthService.cs
public interface IAuthService
{
    Task<bool> LoginAsync(string username, string password, CancellationToken ct = default);
    Task<bool> RefreshAsync(CancellationToken ct = default);
    Task LogoutAsync(CancellationToken ct = default);
}
```

---

## AuthService — 로그인/갱신/로그아웃, 상태 반영, 토큰 저장

- 로그인 성공 시: `AppAuthState`와 `ITokenStore` 모두 업데이트
- Refresh 성공 시: **Refresh Token Rotation**(서버가 새 refresh 제공하면 교체)
- 로그아웃 시: 상태/저장소 정리
- JWT 파싱(선택)으로 `sub/roles/scopes/exp`를 UI에 반영(단, **검증 없이 신뢰 금지**: 서버 응답 UI 표시 정도만)

```csharp
// Services/AuthService.cs
using System.Net.Http.Headers;
using System.Text;
using System.Text.Json;

public sealed class AuthService : IAuthService
{
    private readonly HttpClient _http;
    private readonly AppAuthState _state;
    private readonly ITokenStore _store;

    // 선제 갱신 스큐(예: 30초)
    private static readonly TimeSpan Skew = TimeSpan.FromSeconds(30);

    public AuthService(HttpClient http, AppAuthState state, ITokenStore store)
    {
        _http = http;
        _state = state;
        _store = store;
    }

    public async Task<bool> LoginAsync(string username, string password, CancellationToken ct = default)
    {
        var payload = new LoginRequest { username = username, password = password };
        var reqJson = JsonSerializer.Serialize(payload);
        var res = await _http.PostAsync("/api/auth/login",
            new StringContent(reqJson, Encoding.UTF8, "application/json"), ct);

        if (!res.IsSuccessStatusCode) return false;

        var body = await res.Content.ReadAsStringAsync(ct);
        var auth = JsonSerializer.Deserialize<AuthResponse>(body);
        if (auth is null || string.IsNullOrEmpty(auth.access_token)) return false;

        var exp = DateTimeOffset.UtcNow.AddSeconds(auth.expires_in);
        await ApplyTokensAsync(auth.access_token, exp, auth.refresh_token);

        // 스코프/클레임 반영(비신뢰, UI 힌트용)
        PopulateClaimsFromAccessToken(auth.access_token);

        return true;
    }

    public async Task<bool> RefreshAsync(CancellationToken ct = default)
    {
        if (string.IsNullOrWhiteSpace(_state.RefreshToken)) return false;

        var req = new RefreshRequest { refresh_token = _state.RefreshToken! };
        var reqJson = JsonSerializer.Serialize(req);
        var res = await _http.PostAsync("/api/auth/refresh",
            new StringContent(reqJson, Encoding.UTF8, "application/json"), ct);

        if (!res.IsSuccessStatusCode) return false;

        var body = await res.Content.ReadAsStringAsync(ct);
        var auth = JsonSerializer.Deserialize<AuthResponse>(body);
        if (auth is null || string.IsNullOrEmpty(auth.access_token)) return false;

        var exp = DateTimeOffset.UtcNow.AddSeconds(auth.expires_in);
        await ApplyTokensAsync(auth.access_token, exp, auth.refresh_token);

        PopulateClaimsFromAccessToken(auth.access_token);

        return true;
    }

    public async Task LogoutAsync(CancellationToken ct = default)
    {
        try
        {
            // 서버에 refresh revoke를 제공한다면 호출(선택)
            // await _http.PostAsync("/api/auth/logout", null, ct);
        }
        catch { /* 네트워크 실패는 무시 가능(로컬 로그아웃 강제) */ }

        _state.Clear();
        await _store.ClearAsync();
    }

    private async Task ApplyTokensAsync(string accessToken, DateTimeOffset exp, string? refreshToken)
    {
        _state.AccessToken = accessToken;
        _state.AccessTokenExpiresAtUtc = exp;
        if (!string.IsNullOrWhiteSpace(refreshToken))
            _state.RefreshToken = refreshToken; // rotation 반영

        await _store.SaveAsync(_state.AccessToken, _state.AccessTokenExpiresAtUtc, _state.RefreshToken);
    }

    // 순수 디코딩(UI표시용): 신뢰 경계 밖. (서명 검증 없음)
    private void PopulateClaimsFromAccessToken(string jwt)
    {
        try
        {
            var parts = jwt.Split('.');
            if (parts.Length != 3) return;
            var payload = parts[1];
            var json = Encoding.UTF8.GetString(Base64UrlDecode(payload));
            using var doc = JsonDocument.Parse(json);
            var root = doc.RootElement;

            _state.Subject = root.TryGetProperty("sub", out var sub) ? sub.GetString() : null;

            if (root.TryGetProperty("scope", out var scopeProp))
                _state.Scopes = scopeProp.GetString()?.Split(' ', StringSplitOptions.RemoveEmptyEntries) ?? Array.Empty<string>();

            if (root.TryGetProperty("roles", out var rolesProp) && rolesProp.ValueKind == JsonValueKind.Array)
                _state.Roles = rolesProp.EnumerateArray().Select(x => x.GetString()!).Where(x => x is not null).ToArray();
        }
        catch { /* 무시 - UI 힌트 실패 */ }
    }

    private static byte[] Base64UrlDecode(string input)
    {
        string s = input.Replace('-', '+').Replace('_', '/');
        switch (s.Length % 4) { case 2: s += "=="; break; case 3: s += "="; break; }
        return Convert.FromBase64String(s);
    }
}
```

> 주의: 클라이언트에서 JWT 서명을 **검증하지 않는다**(일반 SPA/Native 클라이언트는 서버 신뢰에 의존). 위 파싱은 **UI 힌트용**이며 보안 결정을 내리지 말 것. 권한은 API 서버가 판단한다(401/403).

---

## RefreshGate — 동시 갱신 단 1회 보장

- 다수의 API 요청이 동시에 만료를 감지해서 **여러 번** Refresh를 호출하는 **폭주를 방지**.
- 단 한 요청만 Refresh를 수행하고 나머지는 그 결과를 기다린다.

```csharp
// Services/RefreshGate.cs
public sealed class RefreshGate
{
    private readonly SemaphoreSlim _sem = new(1, 1);
    private Task<bool>? _inFlight;

    public async Task<bool> EnterAsync(Func<Task<bool>> doRefresh)
    {
        await _sem.WaitAsync().ConfigureAwait(false);
        try
        {
            // 이미 누군가 진행 중이면 그 태스크 반환
            if (_inFlight is not null) return await _inFlight.ConfigureAwait(false);

            _inFlight = doRefresh();
        }
        finally
        {
            _sem.Release();
        }

        try
        {
            return await _inFlight.ConfigureAwait(false);
        }
        finally
        {
            await _sem.WaitAsync().ConfigureAwait(false);
            try { _inFlight = null; } finally { _sem.Release(); }
        }
    }
}
```

---

## AuthenticatedHttpHandler — Authorization 주입 + 만료/401 처리 + 재시도/백오프

- 요청 전: `IsAccessTokenNearExpiry(Δ)`면 RefreshGate를 통해 선제 갱신
- 헤더 주입: `Authorization: Bearer <access>`
- 응답이 401/403일 경우: 1회에 한해 Refresh 후 **원 요청 재시도**
- 429/5xx: 백오프 정책으로 재시도(상황에 맞게 조정)

```csharp
// Services/AuthenticatedHttpHandler.cs
using System.Net;
using System.Net.Http.Headers;

public sealed class AuthenticatedHttpHandler : DelegatingHandler
{
    private readonly AppAuthState _state;
    private readonly IAuthService _auth;
    private readonly RefreshGate _gate;
    private readonly TimeSpan _skew;

    public AuthenticatedHttpHandler(
        AppAuthState state,
        IAuthService auth,
        RefreshGate gate,
        TimeSpan? proactiveSkew = null)
    {
        _state = state;
        _auth = auth;
        _gate = gate;
        _skew = proactiveSkew ?? TimeSpan.FromSeconds(30);
    }

    protected override async Task<HttpResponseMessage> SendAsync(HttpRequestMessage request, CancellationToken ct)
    {
        // 선제 갱신(만료 임박)
        if (_state.IsAccessTokenNearExpiry(_skew))
        {
            await _gate.EnterAsync(() => _auth.RefreshAsync(ct));
        }

        // Authorization 주입
        if (!string.IsNullOrWhiteSpace(_state.AccessToken))
        {
            request.Headers.Authorization = new AuthenticationHeaderValue("Bearer", _state.AccessToken);
        }

        // 1차 전송
        var response = await base.SendAsync(request, ct);

        // 401/403 → 1회 Refresh 후 재시도
        if (response.StatusCode is HttpStatusCode.Unauthorized or HttpStatusCode.Forbidden)
        {
            response.Dispose();

            var refreshed = await _gate.EnterAsync(() => _auth.RefreshAsync(ct));

            if (!refreshed)
            {
                // 강제 로그아웃을 상위에서 처리하거나 예외
                throw new UnauthorizedAccessException("세션 만료(Refresh 실패)");
            }

            // 새 토큰으로 다시 시도
            var retry = CloneRequest(request);
            retry.Headers.Authorization = new AuthenticationHeaderValue("Bearer", _state.AccessToken);

            // 429/5xx는 백오프 정책 사용
            return await BackoffPolicy.RunAsync(async () => await base.SendAsync(retry, ct));
        }

        // 429/5xx → 백오프 재시도
        if ((int)response.StatusCode == 429 || (int)response.StatusCode >= 500)
        {
            response.Dispose();
            return await BackoffPolicy.RunAsync(async () =>
            {
                var retry = CloneRequest(request);
                return await base.SendAsync(retry, ct);
            });
        }

        return response;
    }

    private static HttpRequestMessage CloneRequest(HttpRequestMessage req)
    {
        var clone = new HttpRequestMessage(req.Method, req.RequestUri);
        // 컨텐츠 복제(가능한 경우)
        if (req.Content is not null)
        {
            var ms = new MemoryStream();
            req.Content.CopyToAsync(ms).GetAwaiter().GetResult();
            ms.Position = 0;
            var newContent = new StreamContent(ms);
            foreach (var h in req.Content.Headers) newContent.Headers.TryAddWithoutValidation(h.Key, h.Value);
            clone.Content = newContent;
        }
        foreach (var h in req.Headers) clone.Headers.TryAddWithoutValidation(h.Key, h.Value);
        clone.Version = req.Version;
        clone.Options = req.Options;
        return clone;
    }
}
```

> 주의: 컨텐츠 스트림을 **한 번만 읽을 수 있는 타입**(네트워크 스트림 등)으로 보내는 경우 재시도 전에 **버퍼링**해야 한다. 위 예시는 메모리 버퍼로 복제. 대용량 업로드 재시도는 전략적으로 재시도를 제한하거나 서버 측 멱등성/리줌 업로드를 활용.

---

## BackoffPolicy — 429/5xx 재시도 지수백오프

```csharp
// Services/BackoffPolicy.cs
using System.Net;

public static class BackoffPolicy
{
    public static async Task<HttpResponseMessage> RunAsync(
        Func<Task<HttpResponseMessage>> action,
        int maxRetries = 3,
        TimeSpan? initialDelay = null)
    {
        initialDelay ??= TimeSpan.FromMilliseconds(300);
        var delay = initialDelay.Value;

        for (int attempt = 0; ; attempt++)
        {
            var res = await action();
            if (IsOk(res) || attempt >= maxRetries)
                return res;

            res.Dispose();
            await Task.Delay(delay);
            delay = TimeSpan.FromMilliseconds(Math.Min(delay.TotalMilliseconds * 2, 5000));
        }

        static bool IsOk(HttpResponseMessage r)
        {
            var code = (int)r.StatusCode;
            if (code == 429) return false;
            if (code >= 500) return false;
            return true;
        }
    }
}
```

---

## DI 구성(App.xaml.cs)

- 앱 시작 시 `ITokenStore.LoadAsync()`로 저장된 토큰을 복원 → `AppAuthState` 반영.
- `HttpClient`에 `AuthenticatedHttpHandler` 체인 연결.

```csharp
// App.xaml.cs 일부
public override async void OnFrameworkInitializationCompleted()
{
    var services = new ServiceCollection();

    services.AddSingleton<AppAuthState>();
    services.AddSingleton<ITokenStore>(sp =>
        new DpapiTokenStore(Path.Combine(Environment.GetFolderPath(Environment.SpecialFolder.ApplicationData), "MyApp", "auth.bin")));
    services.AddSingleton<RefreshGate>();

    // Base HttpClient (Handler 체인 끝에 HttpClientHandler)
    services.AddSingleton<HttpMessageHandler>(_ => new HttpClientHandler
    {
        AutomaticDecompression = System.Net.DecompressionMethods.All
    });

    services.AddSingleton<IAuthService>(sp =>
    {
        var baseHandler = sp.GetRequiredService<HttpMessageHandler>();
        var http = new HttpClient(baseHandler) { BaseAddress = new Uri("https://api.example.com") };
        var state = sp.GetRequiredService<AppAuthState>();
        var store = sp.GetRequiredService<ITokenStore>();
        return new AuthService(http, state, store);
    });

    services.AddSingleton(sp =>
    {
        var state = sp.GetRequiredService<AppAuthState>();
        var auth = sp.GetRequiredService<IAuthService>();
        var gate = sp.GetRequiredService<RefreshGate>();
        var authHandler = new AuthenticatedHttpHandler(state, auth, gate);
        authHandler.InnerHandler = sp.GetRequiredService<HttpMessageHandler>();
        return new HttpClient(authHandler) { BaseAddress = new Uri("https://api.example.com"), Timeout = TimeSpan.FromSeconds(100) };
    });

    services.AddTransient<LoginViewModel>();
    services.AddSingleton<ShellViewModel>();

    var provider = services.BuildServiceProvider();

    // 저장소에서 복원
    var store = provider.GetRequiredService<ITokenStore>();
    var state = provider.GetRequiredService<AppAuthState>();
    var (a, exp, r) = await store.LoadAsync();
    if (a is not null && exp > DateTimeOffset.UtcNow)
    {
        state.AccessToken = a;
        state.AccessTokenExpiresAtUtc = exp;
        state.RefreshToken = r;
    }

    // Shell 로드
    var shell = new Views.ShellView { DataContext = provider.GetRequiredService<ShellViewModel>() };
    (ApplicationLifetime as IClassicDesktopStyleApplicationLifetime)!.MainWindow = shell;
    shell.Show();

    base.OnFrameworkInitializationCompleted();
}
```

> 대안: `IHttpClientFactory`(Microsoft.Extensions.Http)를 사용해 명명된 클라이언트를 구성하면 핸들러 수명/소켓 고갈 문제를 잘 관리할 수 있다.

---

## ViewModel — 로그인(상태 반영)·셸 전환

```csharp
// ViewModels/LoginViewModel.cs
public sealed class LoginViewModel : ReactiveUI.ReactiveObject
{
    private readonly IAuthService _auth;
    private readonly AppAuthState _state;

    public LoginViewModel(IAuthService auth, AppAuthState state)
    {
        _auth = auth;
        _state = state;
        LoginCommand = ReactiveUI.ReactiveCommand.CreateFromTask(LoginAsync, canExecute: this.WhenAnyValue(
            x => x.Username, x => x.Password, (u, p) => !string.IsNullOrWhiteSpace(u) && !string.IsNullOrWhiteSpace(p)));
        LogoutCommand = ReactiveUI.ReactiveCommand.CreateFromTask(LogoutAsync);
    }

    private string _username = "";
    public string Username { get => _username; set => this.RaiseAndSetIfChanged(ref _username, value); }

    private string _password = "";
    public string Password { get => _password; set => this.RaiseAndSetIfChanged(ref _password, value); }

    private string _status = "";
    public string Status { get => _status; set => this.RaiseAndSetIfChanged(ref _status, value); }

    public ReactiveUI.ReactiveCommand<Unit, Unit> LoginCommand { get; }
    public ReactiveUI.ReactiveCommand<Unit, Unit> LogoutCommand { get; }

    private async Task LoginAsync()
    {
        Status = "로그인 중...";
        var ok = await _auth.LoginAsync(Username, Password);
        Status = ok ? "로그인 성공" : "로그인 실패";
    }

    private async Task LogoutAsync()
    {
        await _auth.LogoutAsync();
        Status = "로그아웃";
    }

    public bool IsLoggedIn => _state.IsAuthenticated;
}
```

```csharp
// ViewModels/ShellViewModel.cs
public sealed class ShellViewModel : ReactiveUI.ReactiveObject
{
    private readonly AppAuthState _state;
    private object? _current;

    public ShellViewModel(AppAuthState state, LoginViewModel loginVm /* 다른 페이지 VM들 DI 가능 */)
    {
        _state = state;
        _current = _state.IsAuthenticated ? new HomeViewModel() : loginVm;

        // 인증 상태가 바뀌면 화면 전환(간단 예)
        this.WhenAnyValue(_ => _state.AccessToken, _ => _state.AccessTokenExpiresAtUtc)
            .Subscribe(_ =>
            {
                Current = _state.IsAuthenticated ? new HomeViewModel() : loginVm;
            });
    }

    public object? Current
    {
        get => _current;
        private set => this.RaiseAndSetIfChanged(ref _current, value);
    }
}
```

---

## XAML — Login/Shell (요점만)

```xml
<!-- Views/LoginView.axaml -->
<UserControl xmlns="https://github.com/avaloniaui" xmlns:d="https://github.com/avaloniaui">
  <StackPanel Margin="20" Spacing="8">
    <TextBlock Text="로그인" FontSize="20"/>
    <TextBox Watermark="아이디" Text="{Binding Username, Mode=TwoWay}"/>
    <TextBox Watermark="비밀번호" PasswordChar="*" Text="{Binding Password, Mode=TwoWay}"/>
    <StackPanel Orientation="Horizontal" Spacing="8">
      <Button Content="로그인" Command="{Binding LoginCommand}"/>
      <Button Content="로그아웃" Command="{Binding LogoutCommand}"/>
    </StackPanel>
    <TextBlock Text="{Binding Status}"/>
  </StackPanel>
</UserControl>
```

```xml
<!-- Views/ShellView.axaml -->
<Window xmlns="https://github.com/avaloniaui" x:Class="MyApp.Views.ShellView">
  <ContentControl Content="{Binding Current}"/>
</Window>
```

---

## 권한 기반(UI) 제어 — ClaimsAwareViewModel

- 서버가 발급한 토큰의 `scope`·`roles`를 UI 힌트로 활용.
- 중요한 접근 제어는 **반드시 서버**가 한다(403).

```csharp
// ViewModels/ClaimsAwareViewModel.cs
public sealed class ClaimsAwareViewModel : ReactiveUI.ReactiveObject
{
    private readonly AppAuthState _state;

    public ClaimsAwareViewModel(AppAuthState state)
    {
        _state = state;
    }

    public bool CanSeeAdminPanel => _state.Roles.Contains("admin") || _state.Scopes.Contains("admin:read");
}
```

XAML에서:
```xml
<Button Content="관리자" IsVisible="{Binding CanSeeAdminPanel}"/>
```

---

## OAuth2 플로우 선택 가이드

| 플로우 | 설명 | 데스크톱 네이티브 앱 권장 |
|---|---|---|
| Password(ROPC) | ID/PW를 직접 API에 제출(레거시, 권장X) | 지양 |
| Authorization Code + PKCE | 시스템 브라우저로 로그인, 리디렉션 URI로 코드 수신 후 토큰 교환 | 권장 |
| Device Code | 브라우저 없는 환경에서 디바이스 코드 입력 | 상황별 사용 |
| Client Credentials | 사용자 없는 서버 간 통신 | 비해당(백엔드) |

> 본 문서의 코드는 **단순화한 예시**이며, 실제 배포 시에는 **PKCE**를 강력 권장. Avalonia에서는 시스템 브라우저를 열고 **loopback http listener** 또는 **custom URI scheme**으로 리디렉션을 받는 패턴을 사용한다.

---

## 401/403/429/5xx 처리 요약

- 401/403: 한 번만 Refresh 후 **원 요청 재시도**, 그래도 실패면 **세션 만료 처리(로그아웃/로그인 화면)**
- 429/5xx: `BackoffPolicy`로 **지수 백오프 재시도**, 그래도 실패면 사용자에게 명확히 피드백
- 대용량 업로드/다운로드: **재시도 시 컨텐츠 재사용 전략**(버퍼/임시파일/리줌) 사전 설계

---

## 안전한 로깅/디버깅

- 절대 **토큰 전체를 로그 출력**하지 말 것. 마스킹 처리:
  - `Bearer abcdef...` 앞 8자만
- 예외 메시지에 민감정보(토큰/비번) 포함 금지
- 클립보드 복사/오버레이 UI에 토큰 표시 금지

---

## 테스트 전략

### HttpMessageHandler 목킹으로 만료/401/Refresh 성공/실패 검증

```csharp
public sealed class FakeHandler : HttpMessageHandler
{
    private readonly Func<HttpRequestMessage, HttpResponseMessage> _responder;
    public FakeHandler(Func<HttpRequestMessage, HttpResponseMessage> responder) => _responder = responder;
    protected override Task<HttpResponseMessage> SendAsync(HttpRequestMessage request, CancellationToken cancellationToken)
        => Task.FromResult(_responder(request));
}
```

- 시나리오 예:
  1) 첫 요청 401 → Refresh 엔드포인트 200(new token) → 원 요청 재시도 200
  2) 첫 요청 401 → Refresh 400 → 예외/로그아웃
  3) 만료 임박 상태 → 요청 전 선제 Refresh → 200

```csharp
[Fact]
public async Task Should_Refresh_And_Retry_On_401()
{
    var state = new AppAuthState { AccessToken = "old", AccessTokenExpiresAtUtc = DateTimeOffset.UtcNow.AddMinutes(-1), RefreshToken = "r1" };
    var store = new InMemoryTokenStore();

    var handler = new FakeHandler(req =>
    {
        if (req.RequestUri!.AbsolutePath == "/api/auth/refresh")
            return new HttpResponseMessage(HttpStatusCode.OK)
            { Content = new StringContent("{\"access_token\":\"new\",\"expires_in\":3600}", Encoding.UTF8, "application/json") };

        // 원 요청: 최초 401, 재시도 200
        if (req.Headers.Authorization?.Parameter == "old")
            return new HttpResponseMessage(HttpStatusCode.Unauthorized);
        if (req.Headers.Authorization?.Parameter == "new")
            return new HttpResponseMessage(HttpStatusCode.OK) { Content = new StringContent("ok") };

        return new HttpResponseMessage(HttpStatusCode.BadRequest);
    });

    var baseHttp = new HttpClient(handler) { BaseAddress = new Uri("https://fake") };
    var auth = new AuthService(baseHttp, state, store);
    var gate = new RefreshGate();

    var authHandler = new AuthenticatedHttpHandler(state, auth, gate) { InnerHandler = handler };
    var http = new HttpClient(authHandler) { BaseAddress = new Uri("https://fake") };

    var res = await http.GetAsync("/data");
    var s = await res.Content.ReadAsStringAsync();
    Assert.Equal("ok", s);
    Assert.Equal("new", state.AccessToken);
}
```

---

## 로그아웃/토큰 폐기/Refresh Rotation

- 서버가 **Refresh Token Rotation**을 사용한다면, Refresh 성공 시 항상 **새 refresh**가 온다 → 클라이언트는 **항상 교체**.
- 오래된 refresh로 재사용을 시도하면 서버는 거부 → 클라이언트는 로그아웃 처리.
- 로그아웃 시 로컬 저장소/상태 **모두 제거**.

---

## UI/UX 팁

- 로그인 중/갱신 중 **스피너** 표시(`IsBusy`, `IsRefreshing`)
- 세션 만료 시 **친절한 안내** + 로그인 버튼
- 관리자 메뉴 등 **권한 기반 노출**(UI 힌트) — 실제 권한 판단은 API에서.

---

## 수학적 모델(토큰 잔여시간/선제갱신) — 간단 식

만료시각을 \( T_{\text{exp}} \), 현재시각을 \( t \), 선제갱신 여유를 \( \Delta \)라 하자.
선제 갱신 조건은 다음과 같다.

$$
t \ge T_{\text{exp}} - \Delta
$$

즉, 잔여 시간이 \( \Delta \) 이하가 되면 갱신을 수행한다. 일반적으로 \( \Delta \in [15s, 60s] \)가 실무에서 안정적으로 쓰인다.

---

## 확장 주제

- **OIDC Discovery**: `/.well-known/openid-configuration`에서 토큰/인증 엔드포인트 자동 탐색
- **JWKs 키 회전**: 리소스 서버를 직접 호출하기 전에 클라이언트가 ID Token의 서명을 검증해야 하는 사용례(고급)라면 `jwks_uri`에서 키를 갱신
- **SSO/Single Logout**: OIDC의 RP-Init Logout Flow
- **Device Code Flow**: 키보드/브라우저 없는 단말

---

## 요약 표

| 항목 | 구현 포인트 |
|---|---|
| 상태 관리 | `AppAuthState` (Access/Refresh/만료/Scope/Role) |
| 토큰 저장 | `ITokenStore` 추상화: 메모리/DPAPI/SecretService |
| 로그인/갱신 | `IAuthService.Login/Refresh/Logout` |
| 자동 주입/갱신 | `AuthenticatedHttpHandler` + `RefreshGate`(single-flight) |
| 선제 갱신 | 만료 임박 \( t \ge T_{\text{exp}} - \Delta \) 시 Refresh |
| 에러 처리 | 401/403 1회 재시도, 429/5xx 백오프 |
| 권한 제어 | Roles/Scopes UI 힌트(서버가 최종 권한 판단) |
| 보안 수칙 | 토큰 로그금지, 안전 저장소, Rotation 준수, 재시도 멱등성 |
| 테스트 | `HttpMessageHandler` 목킹, 만료/401/Refresh 실패 시나리오 |

---

## 마무리

이 설계는 Avalonia MVVM 환경에서 **안전하고 견고한 인증 파이프라인**을 제공한다.
핵심은 **책임 분리**(상태/저장소/서비스/핸들러), **선제적 갱신/동시성 제어**, **실패 시 재시도와 명확한 UX**, **보안 수칙 준수**다.
실 운영에서는 **Authorization Code + PKCE** 흐름을 사용하는 것을 권장하며, 여기의 구조를 그대로 적용해 토큰 교환/갱신/저장/자동 주입을 일관되게 유지하면 된다.
