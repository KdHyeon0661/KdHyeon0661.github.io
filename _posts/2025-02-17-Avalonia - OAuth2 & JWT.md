---
layout: post
title: Avalonia - OAuth2 & JWT 인증
date: 2025-02-17 20:20:23 +0900
category: Avalonia
---
# Avalonia MVVM + OAuth2 / JWT 인증 구조화

Avalonia 데스크톱 애플리케이션에서 OAuth2 기반 인증을 구현하려면 액세스 토큰을 안전하게 저장하고, 만료 시 자동으로 갱신하며, API 요청마다 토큰을 첨부하는 일관된 파이프라인이 필요합니다. 이 글에서는 초중급 개발자를 기준으로, ReactiveUI를 활용한 MVVM 구조에서 JWT 토큰을 관리하는 전체적인 설계와 구현 방법을 설명합니다. 핵심은 **책임 분리**(상태/저장소/서비스/핸들러), **선제적 갱신**, **동시성 제어**, **실패 대응**, **보안 수칙**입니다.

---

## 전체 구조 개요

```
MyApp/
├── State/
│   └── AppAuthState.cs               // 런타임 인증 상태 (Access/Refresh/만료/Scope/Role)
├── Services/
│   ├── IAuthService.cs               // 로그인/갱신/로그아웃 계약
│   ├── AuthService.cs                // API 호출 + 상태/저장소 업데이트
│   ├── ITokenStore.cs                // 토큰 저장소 추상화 (메모리/DPAPI/SecretService)
│   ├── InMemoryTokenStore.cs         // 메모리 저장 (기본)
│   ├── DpapiTokenStore.cs            // Windows DPAPI 예시
│   ├── AuthenticatedHttpHandler.cs   // Authorization 주입 + 만료/401 자동 처리
│   ├── RefreshGate.cs                // 동시 갱신 단일화 게이트
│   └── BackoffPolicy.cs              // 429/5xx 재시도 지수백오프
├── ViewModels/
│   ├── LoginViewModel.cs
│   ├── ShellViewModel.cs
│   └── ClaimsAwareViewModel.cs       // 역할/스코프 기반 UI 제어 예시
├── Views/
│   ├── LoginView.axaml
│   └── ShellView.axaml
└── Models/
    └── AuthModels.cs                 // DTO: AuthResponse, LoginRequest 등
```

이 구조는 인증 관련 로직을 독립적으로 분리하여 테스트와 유지보수를 쉽게 합니다.

---

## 런타임 인증 상태: AppAuthState

인증 상태는 앱 전반에서 공유됩니다. ReactiveUI의 `ReactiveObject`를 상속받아 바인딩 가능한 속성으로 제공합니다.

```csharp
// State/AppAuthState.cs
public sealed class AppAuthState : ReactiveUI.ReactiveObject
{
    private string? _accessToken;
    private string? _refreshToken;
    private DateTimeOffset _accessTokenExpiresAtUtc;
    private string[] _scopes = Array.Empty<string>();
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
    }
}
```

**선제 갱신 조건**:  
$$ t \ge T_{\text{exp}} - \Delta $$  
여기서 \(t\)는 현재 시각, \(T_{\text{exp}}\)는 만료 시각, \(\Delta\)는 보통 15~60초 사이의 여유 시간입니다. 이 조건을 만족하면 API 요청 전에 토큰을 갱신합니다.

---

## 토큰 저장소 추상화: ITokenStore

액세스 토큰과 리프레시 토큰을 **안전한 위치**에 저장해야 합니다. 인터페이스로 추상화하여 구현을 교체할 수 있도록 합니다.

```csharp
// Services/ITokenStore.cs
public interface ITokenStore
{
    Task SaveAsync(string accessToken, DateTimeOffset expiresAtUtc, string? refreshToken);
    Task<(string? accessToken, DateTimeOffset expiresAtUtc, string? refreshToken)> LoadAsync();
    Task ClearAsync();
}
```

### 메모리 저장소 (기본)

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
        _exp = expiresAtUtc;
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

### Windows DPAPI 저장소 예시

```csharp
// Services/DpapiTokenStore.cs
using System.Security.Cryptography;

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
        return (doc.GetProperty("access").GetString(),
                doc.GetProperty("expUtc").GetDateTimeOffset(),
                doc.GetProperty("refresh").GetString());
    }

    public Task ClearAsync()
    {
        if (File.Exists(_path)) File.Delete(_path);
        return Task.CompletedTask;
    }
}
```

> **보안 주의**: Linux/macOS에서는 `SecretService`나 Keychain을 사용하는 별도 구현이 필요합니다. 프로덕션에서는 반드시 OS가 제공하는 안전한 저장소를 활용하세요.

---

## 인증 서비스: IAuthService

인증 서비스는 로그인, 토큰 갱신, 로그아웃을 담당하며, `AppAuthState`와 `ITokenStore`를 업데이트합니다.

```csharp
// Models/AuthModels.cs
public sealed class LoginRequest { public string username { get; init; } = ""; public string password { get; init; } = ""; }
public sealed class RefreshRequest { public string refresh_token { get; init; } = ""; }
public sealed class AuthResponse
{
    public string access_token { get; init; } = "";
    public string? refresh_token { get; init; }
    public int expires_in { get; init; }
    public string? scope { get; init; }
}
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

```csharp
// Services/AuthService.cs
using System.Text;
using System.Text.Json;

public sealed class AuthService : IAuthService
{
    private readonly HttpClient _http;
    private readonly AppAuthState _state;
    private readonly ITokenStore _store;

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
            // 서버에 리프레시 토큰 폐기 요청 (옵션)
            // await _http.PostAsync("/api/auth/logout", null, ct);
        }
        catch { /* 네트워크 실패 시 로컬 로그아웃 진행 */ }

        _state.Clear();
        await _store.ClearAsync();
    }

    private async Task ApplyTokensAsync(string accessToken, DateTimeOffset exp, string? refreshToken)
    {
        _state.AccessToken = accessToken;
        _state.AccessTokenExpiresAtUtc = exp;
        if (!string.IsNullOrWhiteSpace(refreshToken))
            _state.RefreshToken = refreshToken;  // 리프레시 토큰 로테이션 반영

        await _store.SaveAsync(_state.AccessToken, _state.AccessTokenExpiresAtUtc, _state.RefreshToken);
    }

    // JWT 페이로드 파싱 (UI 힌트용, 서명 검증 없음)
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

            if (root.TryGetProperty("scope", out var scopeProp))
                _state.Scopes = scopeProp.GetString()?.Split(' ', StringSplitOptions.RemoveEmptyEntries) ?? Array.Empty<string>();

            if (root.TryGetProperty("roles", out var rolesProp) && rolesProp.ValueKind == JsonValueKind.Array)
                _state.Roles = rolesProp.EnumerateArray().Select(x => x.GetString()!).Where(x => x is not null).ToArray();
        }
        catch { /* 파싱 실패 시 무시 */ }
    }

    private static byte[] Base64UrlDecode(string input)
    {
        string s = input.Replace('-', '+').Replace('_', '/');
        switch (s.Length % 4)
        {
            case 2: s += "=="; break;
            case 3: s += "="; break;
        }
        return Convert.FromBase64String(s);
    }
}
```

---

## 갱신 게이트: RefreshGate

동시에 여러 API 호출이 만료를 감지하면 여러 번의 갱신 요청이 발생할 수 있습니다. 이를 방지하기 위해 **한 번의 갱신만 수행**하고 나머지는 그 결과를 대기하도록 하는 게이트를 만듭니다.

```csharp
// Services/RefreshGate.cs
public sealed class RefreshGate
{
    private readonly SemaphoreSlim _sem = new(1, 1);
    private Task<bool>? _inFlight;

    public async Task<bool> EnterAsync(Func<Task<bool>> doRefresh)
    {
        await _sem.WaitAsync();
        try
        {
            if (_inFlight is not null) return await _inFlight;
            _inFlight = doRefresh();
        }
        finally
        {
            _sem.Release();
        }

        try
        {
            return await _inFlight;
        }
        finally
        {
            await _sem.WaitAsync();
            try { _inFlight = null; } finally { _sem.Release(); }
        }
    }
}
```

---

## 인증 핸들러: AuthenticatedHttpHandler

이 핸들러는 `HttpClient`의 요청 파이프라인에 끼어들어:

- 선제 갱신 조건 만족 시 RefreshGate를 통해 갱신 시도
- 요청에 `Authorization: Bearer` 헤더 추가
- 응답이 401/403이면 한 번만 갱신 후 원 요청 재시도
- 429 또는 5xx 오류는 백오프 정책으로 재시도

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
        // 1. 선제 갱신
        if (_state.IsAccessTokenNearExpiry(_skew))
        {
            await _gate.EnterAsync(() => _auth.RefreshAsync(ct));
        }

        // 2. Authorization 헤더 추가
        if (!string.IsNullOrWhiteSpace(_state.AccessToken))
        {
            request.Headers.Authorization = new AuthenticationHeaderValue("Bearer", _state.AccessToken);
        }

        // 3. 1차 전송
        var response = await base.SendAsync(request, ct);

        // 4. 401/403 → 갱신 후 재시도
        if (response.StatusCode is HttpStatusCode.Unauthorized or HttpStatusCode.Forbidden)
        {
            response.Dispose();

            var refreshed = await _gate.EnterAsync(() => _auth.RefreshAsync(ct));
            if (!refreshed)
                throw new UnauthorizedAccessException("세션 만료 (Refresh 실패)");

            // 새 토큰으로 재시도
            var retry = CloneRequest(request);
            retry.Headers.Authorization = new AuthenticationHeaderValue("Bearer", _state.AccessToken);
            return await BackoffPolicy.RunAsync(async () => await base.SendAsync(retry, ct));
        }

        // 5. 429/5xx → 백오프 재시도
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
        // 헤더 복사
        foreach (var h in req.Headers)
            clone.Headers.TryAddWithoutValidation(h.Key, h.Value);
        // 콘텐츠 복사 (멀티 사용 가능하도록 버퍼링)
        if (req.Content is not null)
        {
            var ms = new MemoryStream();
            req.Content.CopyToAsync(ms).GetAwaiter().GetResult();
            ms.Position = 0;
            var newContent = new StreamContent(ms);
            foreach (var h in req.Content.Headers)
                newContent.Headers.TryAddWithoutValidation(h.Key, h.Value);
            clone.Content = newContent;
        }
        clone.Version = req.Version;
        clone.Options = req.Options;
        return clone;
    }
}
```

---

## 백오프 정책: BackoffPolicy

429(Too Many Requests)나 5xx 오류 발생 시 지수 백오프로 재시도합니다.

```csharp
// Services/BackoffPolicy.cs
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
            int code = (int)r.StatusCode;
            return code != 429 && code < 500;
        }
    }
}
```

---

## DI 구성 및 앱 초기화

`App.axaml.cs`에서 DI 컨테이너를 구성하고, 저장된 토큰을 복원합니다.

```csharp
// App.axaml.cs (일부)
public override async void OnFrameworkInitializationCompleted()
{
    var services = new ServiceCollection();

    // 1. 상태, 저장소, 게이트 등록
    services.AddSingleton<AppAuthState>();
    services.AddSingleton<ITokenStore>(sp =>
        new DpapiTokenStore(Path.Combine(
            Environment.GetFolderPath(Environment.SpecialFolder.ApplicationData),
            "MyApp", "auth.bin")));
    services.AddSingleton<RefreshGate>();

    // 2. 베이스 핸들러 (실제 네트워크 통신)
    services.AddSingleton<HttpMessageHandler>(_ => new HttpClientHandler
    {
        AutomaticDecompression = System.Net.DecompressionMethods.All
    });

    // 3. AuthService (베이스 HttpClient 사용)
    services.AddSingleton<IAuthService>(sp =>
    {
        var baseHandler = sp.GetRequiredService<HttpMessageHandler>();
        var http = new HttpClient(baseHandler) { BaseAddress = new Uri("https://api.example.com") };
        var state = sp.GetRequiredService<AppAuthState>();
        var store = sp.GetRequiredService<ITokenStore>();
        return new AuthService(http, state, store);
    });

    // 4. 인증 핸들러를 포함한 HttpClient 등록
    services.AddSingleton(sp =>
    {
        var state = sp.GetRequiredService<AppAuthState>();
        var auth = sp.GetRequiredService<IAuthService>();
        var gate = sp.GetRequiredService<RefreshGate>();
        var authHandler = new AuthenticatedHttpHandler(state, auth, gate)
        {
            InnerHandler = sp.GetRequiredService<HttpMessageHandler>()
        };
        return new HttpClient(authHandler) { BaseAddress = new Uri("https://api.example.com") };
    });

    // 5. ViewModel 등록
    services.AddTransient<LoginViewModel>();
    services.AddSingleton<ShellViewModel>();

    var provider = services.BuildServiceProvider();

    // 6. 저장된 토큰 복원
    var store = provider.GetRequiredService<ITokenStore>();
    var state = provider.GetRequiredService<AppAuthState>();
    var (access, exp, refresh) = await store.LoadAsync();
    if (access is not null && exp > DateTimeOffset.UtcNow)
    {
        state.AccessToken = access;
        state.AccessTokenExpiresAtUtc = exp;
        state.RefreshToken = refresh;
        // (선택) 클레임 파싱은 AuthService가 수행하므로 여기서는 간단히만 복원
    }

    // 7. 셸 표시
    var shell = new Views.ShellView { DataContext = provider.GetRequiredService<ShellViewModel>() };
    (ApplicationLifetime as IClassicDesktopStyleApplicationLifetime)!.MainWindow = shell;
    shell.Show();

    base.OnFrameworkInitializationCompleted();
}
```

---

## ViewModel: LoginViewModel

로그인 화면의 ViewModel입니다. 로그인 성공 시 `AuthService`를 호출하고, 상태가 자동으로 업데이트됩니다.

```csharp
// ViewModels/LoginViewModel.cs
using ReactiveUI;

public sealed class LoginViewModel : ReactiveObject
{
    private readonly IAuthService _auth;
    private readonly AppAuthState _state;

    public LoginViewModel(IAuthService auth, AppAuthState state)
    {
        _auth = auth;
        _state = state;

        LoginCommand = ReactiveCommand.CreateFromTask(LoginAsync,
            this.WhenAnyValue(x => x.Username, x => x.Password,
                (u, p) => !string.IsNullOrWhiteSpace(u) && !string.IsNullOrWhiteSpace(p)));
        LogoutCommand = ReactiveCommand.CreateFromTask(LogoutAsync);
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

    private string _status = "";
    public string Status
    {
        get => _status;
        set => this.RaiseAndSetIfChanged(ref _status, value);
    }

    public ReactiveCommand<Unit, Unit> LoginCommand { get; }
    public ReactiveCommand<Unit, Unit> LogoutCommand { get; }

    private async Task LoginAsync()
    {
        Status = "로그인 중...";
        var ok = await _auth.LoginAsync(Username, Password);
        Status = ok ? "로그인 성공" : "로그인 실패";
    }

    private async Task LogoutAsync()
    {
        await _auth.LogoutAsync();
        Status = "로그아웃됨";
    }

    // UI에서 인증 상태 표시용 (옵션)
    public bool IsLoggedIn => _state.IsAuthenticated;
}
```

---

## ViewModel: ShellViewModel

인증 상태에 따라 현재 화면을 전환합니다. 인증되면 메인 화면(예: HomeViewModel)을, 그렇지 않으면 로그인 화면을 보여줍니다.

```csharp
// ViewModels/ShellViewModel.cs
public sealed class ShellViewModel : ReactiveObject
{
    private readonly AppAuthState _state;
    private object? _current;

    public ShellViewModel(AppAuthState state, LoginViewModel loginVm, HomeViewModel homeVm)
    {
        _state = state;
        _current = _state.IsAuthenticated ? homeVm : loginVm;

        this.WhenAnyValue(_ => _state.AccessToken, _ => _state.AccessTokenExpiresAtUtc)
            .Subscribe(_ =>
            {
                Current = _state.IsAuthenticated ? homeVm : loginVm;
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

## View: LoginView 및 ShellView

간단한 로그인 UI와 콘텐츠 전환을 위한 `ContentControl`입니다.

```xml
<!-- Views/LoginView.axaml -->
<UserControl xmlns="https://github.com/avaloniaui"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             x:Class="MyApp.Views.LoginView">
    <StackPanel Margin="20" Spacing="8">
        <TextBlock Text="로그인" FontSize="20"/>
        <TextBox Watermark="아이디" Text="{Binding Username, Mode=TwoWay}"/>
        <TextBox Watermark="비밀번호" PasswordChar="*" Text="{Binding Password, Mode=TwoWay}"/>
        <Button Content="로그인" Command="{Binding LoginCommand}"/>
        <Button Content="로그아웃" Command="{Binding LogoutCommand}"/>
        <TextBlock Text="{Binding Status}" Foreground="Gray"/>
    </StackPanel>
</UserControl>
```

```xml
<!-- Views/ShellView.axaml -->
<Window xmlns="https://github.com/avaloniaui"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        x:Class="MyApp.Views.ShellView"
        Title="My App" Width="800" Height="600">
    <ContentControl Content="{Binding Current}"/>
</Window>
```

---

## 권한 기반 UI 제어 (힌트)

JWT의 `scope` 또는 `roles` 클레임을 UI에서 참조하여 버튼/메뉴를 표시하거나 숨길 수 있습니다. **실제 접근 제어는 반드시 서버에서 수행**되어야 합니다.

```csharp
// ViewModels/ClaimsAwareViewModel.cs
public sealed class ClaimsAwareViewModel : ReactiveObject
{
    private readonly AppAuthState _state;
    public ClaimsAwareViewModel(AppAuthState state) => _state = state;

    public bool CanSeeAdminPanel => _state.Roles.Contains("admin") || _state.Scopes.Contains("admin:read");
}
```

XAML 예:

```xml
<Button Content="관리자 패널" IsVisible="{Binding CanSeeAdminPanel}"/>
```

---

## OAuth2 플로우 선택 가이드

| 플로우 | 설명 | 데스크톱 앱 권장 |
|-------|------|-----------------|
| Resource Owner Password (ROPC) | 사용자명/비밀번호 직접 전송 | **권장하지 않음** (보안 취약) |
| Authorization Code + PKCE | 시스템 브라우저로 로그인 → 리디렉션 → 코드 교환 | **가장 권장** |
| Device Code | 브라우저 없는 디바이스에서 코드 입력 | 특수 환경에 적합 |

위 코드 예제는 이해를 돕기 위해 **ROPC**를 사용했습니다. 실제 프로젝트에서는 반드시 **Authorization Code + PKCE**를 적용하세요. 구현 시에는 `Avalonia.WebView`나 시스템 브라우저를 열고 `http://localhost:port/callback` 또는 커스텀 URI 스킴을 통해 코드를 수신하는 패턴을 사용합니다.

---

## 보안 주의점

- **토큰 로깅 금지**: 예외 메시지, 콘솔 출력, 파일 로그에 액세스 토큰 전체를 기록하지 마세요. 앞 몇 글자만 표시하는 식으로 마스킹 처리합니다.
- **안전한 저장소**: 프로덕션에서는 반드시 OS가 제공하는 안전한 저장소(DPAPI, Keychain, SecretService)를 사용하세요.
- **리프레시 토큰 로테이션**: 서버가 리프레시 토큰을 교체해 주면 클라이언트는 항상 새 토큰으로 갱신해야 합니다.
- **재시도 멱등성**: POST 요청 등 멱등하지 않은 요청은 재시도 시 중복 처리를 방지하기 위해 서버 측에서 멱등성 키 등을 지원해야 할 수 있습니다.
- **클레임 신뢰 경계**: 클라이언트는 JWT의 클레임을 **UI 힌트로만** 사용하고, 실제 권한 검증은 API 서버에서 수행하세요.

---

## 테스트 전략

`HttpMessageHandler`를 목킹하여 인증 핸들러의 동작을 테스트할 수 있습니다.

```csharp
// Tests/AuthenticatedHandlerTests.cs
public sealed class FakeHandler : HttpMessageHandler
{
    private readonly Func<HttpRequestMessage, HttpResponseMessage> _responder;
    public FakeHandler(Func<HttpRequestMessage, HttpResponseMessage> responder) => _responder = responder;
    protected override Task<HttpResponseMessage> SendAsync(HttpRequestMessage request, CancellationToken ct)
        => Task.FromResult(_responder(request));
}

[Fact]
public async Task Should_Refresh_And_Retry_On_401()
{
    // Arrange
    var state = new AppAuthState
    {
        AccessToken = "old",
        AccessTokenExpiresAtUtc = DateTimeOffset.UtcNow.AddMinutes(-1),
        RefreshToken = "r1"
    };
    var store = new InMemoryTokenStore();
    var baseHandler = new FakeHandler(req =>
    {
        if (req.RequestUri!.AbsolutePath == "/api/auth/refresh")
            return new HttpResponseMessage(HttpStatusCode.OK)
            {
                Content = new StringContent("{\"access_token\":\"new\",\"expires_in\":3600}", Encoding.UTF8, "application/json")
            };
        if (req.Headers.Authorization?.Parameter == "old")
            return new HttpResponseMessage(HttpStatusCode.Unauthorized);
        if (req.Headers.Authorization?.Parameter == "new")
            return new HttpResponseMessage(HttpStatusCode.OK) { Content = new StringContent("ok") };
        return new HttpResponseMessage(HttpStatusCode.BadRequest);
    });

    var baseHttp = new HttpClient(baseHandler) { BaseAddress = new Uri("https://fake") };
    var auth = new AuthService(baseHttp, state, store);
    var gate = new RefreshGate();
    var authHandler = new AuthenticatedHttpHandler(state, auth, gate) { InnerHandler = baseHandler };
    var http = new HttpClient(authHandler) { BaseAddress = new Uri("https://fake") };

    // Act
    var response = await http.GetAsync("/data");
    var content = await response.Content.ReadAsStringAsync();

    // Assert
    Assert.Equal("ok", content);
    Assert.Equal("new", state.AccessToken);
}
```

---

## 확장 주제

- **OIDC Discovery**: `/.well-known/openid-configuration`에서 엔드포인트 자동 탐색
- **JWKs 키 회전**: 클라이언트에서 ID Token의 서명을 검증해야 하는 경우 `jwks_uri`에서 키 주기적 갱신
- **SSO 로그아웃**: OIDC의 RP-Initiated Logout 구현
- **다중 인증 공급자**: Google, Microsoft 등 연동

---

## 요약

| 구성 요소 | 역할 |
|-----------|------|
| `AppAuthState` | 액세스 토큰, 리프레시 토큰, 만료 시각, 스코프, 역할을 메모리에 보관 |
| `ITokenStore` | 토큰을 안전하게 저장/복원하는 추상화 |
| `IAuthService` | 로그인/갱신/로그아웃 API 호출 및 상태/저장소 업데이트 |
| `AuthenticatedHttpHandler` | 요청에 토큰 첨부, 만료 시 선제 갱신, 401/403 시 갱신 후 재시도, 429/5xx 백오프 |
| `RefreshGate` | 동시 갱신을 하나로 통제 |
| `BackoffPolicy` | 429/5xx 재시도 지수 백오프 |

이 구조를 통해 Avalonia 앱에서 **안전하고 견고한 OAuth2/JWT 인증**을 구현할 수 있습니다. 초중급 개발자도 이해하기 쉽도록 핵심 개념과 코드를 분리하여 설명했습니다. 실제 프로젝트에서는 보안 수칙을 철저히 준수하고, Authorization Code + PKCE 플로우를 적용하여 최종 제품을 완성하세요.