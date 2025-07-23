---
layout: post
title: Avalonia - OAuth2 & JWT 인증
date: 2025-02-17 20:20:23 +0900
category: Avalonia
---
# 🔒 Avalonia MVVM + OAuth2 / JWT 인증 구조화

---

## 🎯 목표

| 항목 | 설명 |
|------|------|
| 로그인 API 연동 | 사용자 ID/PW로 AccessToken, RefreshToken 발급 |
| 토큰 저장 | 앱 메모리 또는 디스크 (보안 고려) |
| API 호출 시 토큰 포함 | Authorization 헤더에 `Bearer <AccessToken>` |
| 만료 처리 | 자동 재로그인 또는 Refresh 처리 |
| 인증 상태 관리 | AppState 등에서 상태 공유

---

## 🧱 구성 예시

```
MyApp/
├── Views/
│   └── LoginView.axaml
├── ViewModels/
│   └── LoginViewModel.cs
├── Services/
│   ├── IAuthService.cs
│   ├── AuthService.cs
│   └── AuthenticatedHttpHandler.cs
├── State/
│   └── AppAuthState.cs
```

---

## 1️⃣ 로그인 API 예시 (서버)

```http
POST /api/auth/login
{
  "username": "test",
  "password": "1234"
}

응답:
{
  "access_token": "eyJhbGciOiJIUzI1...",
  "refresh_token": "eyJhbGciOiJIUzI1...",
  "expires_in": 3600
}
```

---

## 2️⃣ 인증 상태를 전역으로 관리

### 📄 State/AppAuthState.cs

```csharp
public class AppAuthState
{
    public string? AccessToken { get; set; }
    public string? RefreshToken { get; set; }
    public DateTime TokenExpiresAt { get; set; }

    public bool IsAuthenticated => !string.IsNullOrEmpty(AccessToken) &&
                                   TokenExpiresAt > DateTime.UtcNow;
}
```

---

## 3️⃣ AuthService (로그인 및 Refresh)

### 📄 Services/IAuthService.cs

```csharp
public interface IAuthService
{
    Task<bool> LoginAsync(string username, string password);
    Task<bool> RefreshAsync();
}
```

### 📄 Services/AuthService.cs

```csharp
public class AuthService : IAuthService
{
    private readonly HttpClient _http;
    private readonly AppAuthState _state;

    public AuthService(HttpClient http, AppAuthState state)
    {
        _http = http;
        _state = state;
    }

    public async Task<bool> LoginAsync(string username, string password)
    {
        var json = JsonSerializer.Serialize(new { username, password });
        var content = new StringContent(json, Encoding.UTF8, "application/json");

        var response = await _http.PostAsync("/api/auth/login", content);
        if (!response.IsSuccessStatusCode) return false;

        var body = await response.Content.ReadAsStringAsync();
        var result = JsonSerializer.Deserialize<AuthResponse>(body);

        if (result == null) return false;

        _state.AccessToken = result.access_token;
        _state.RefreshToken = result.refresh_token;
        _state.TokenExpiresAt = DateTime.UtcNow.AddSeconds(result.expires_in);

        return true;
    }

    public async Task<bool> RefreshAsync()
    {
        if (string.IsNullOrEmpty(_state.RefreshToken))
            return false;

        var json = JsonSerializer.Serialize(new { refresh_token = _state.RefreshToken });
        var content = new StringContent(json, Encoding.UTF8, "application/json");

        var response = await _http.PostAsync("/api/auth/refresh", content);
        if (!response.IsSuccessStatusCode) return false;

        var body = await response.Content.ReadAsStringAsync();
        var result = JsonSerializer.Deserialize<AuthResponse>(body);

        if (result == null) return false;

        _state.AccessToken = result.access_token;
        _state.TokenExpiresAt = DateTime.UtcNow.AddSeconds(result.expires_in);
        return true;
    }

    private class AuthResponse
    {
        public string access_token { get; set; } = "";
        public string refresh_token { get; set; } = "";
        public int expires_in { get; set; }
    }
}
```

---

## 4️⃣ 인증 포함된 HttpClient 구성

### 📄 Services/AuthenticatedHttpHandler.cs

```csharp
public class AuthenticatedHttpHandler : DelegatingHandler
{
    private readonly AppAuthState _authState;
    private readonly IAuthService _authService;

    public AuthenticatedHttpHandler(AppAuthState authState, IAuthService authService)
    {
        _authState = authState;
        _authService = authService;
    }

    protected override async Task<HttpResponseMessage> SendAsync(
        HttpRequestMessage request, CancellationToken cancellationToken)
    {
        if (!_authState.IsAuthenticated)
        {
            bool refreshed = await _authService.RefreshAsync();
            if (!refreshed)
                throw new UnauthorizedAccessException("토큰이 만료되었고 재인증에 실패했습니다.");
        }

        request.Headers.Authorization = 
            new AuthenticationHeaderValue("Bearer", _authState.AccessToken);
        
        return await base.SendAsync(request, cancellationToken);
    }
}
```

---

## 5️⃣ DI 등록

```csharp
services.AddSingleton<AppAuthState>();
services.AddSingleton<IAuthService, AuthService>();
services.AddTransient<AuthenticatedHttpHandler>();

services.AddSingleton(provider =>
{
    var authHandler = provider.GetRequiredService<AuthenticatedHttpHandler>();
    return new HttpClient(authHandler)
    {
        BaseAddress = new Uri("https://api.example.com")
    };
});
```

---

## 6️⃣ 로그인 ViewModel 예시

### 📄 ViewModels/LoginViewModel.cs

```csharp
public class LoginViewModel : ReactiveObject
{
    private readonly IAuthService _authService;

    public string Username { get; set; } = "";
    public string Password { get; set; } = "";
    public string Status { get; private set; } = "";

    public ReactiveCommand<Unit, Unit> LoginCommand { get; }

    public LoginViewModel(IAuthService authService)
    {
        _authService = authService;
        LoginCommand = ReactiveCommand.CreateFromTask(LoginAsync);
    }

    private async Task LoginAsync()
    {
        var success = await _authService.LoginAsync(Username, Password);
        Status = success ? "✅ 로그인 성공" : "❌ 로그인 실패";
        this.RaisePropertyChanged(nameof(Status));
    }
}
```

---

## ✅ 전체 흐름 요약

```
[1] 사용자가 로그인 정보 입력
 → AuthService.LoginAsync()
 → access_token + refresh_token 저장

[2] API 호출 시 AuthenticatedHttpHandler 사용
 → access_token 삽입
 → 만료 시 자동 Refresh 처리

[3] 인증 상태는 AppAuthState에서 관리
 → ViewModel/Service 간 공유됨
```

---

## 🔐 토큰 저장 방식 (선택지)

| 저장 방식 | 보안성 | 지속성 |
|-----------|--------|--------|
| 메모리 (AppAuthState) | 👍 안전함 | ❌ 앱 종료 시 삭제 |
| 파일 (암호화 저장) | 중간 | ✅ 지속됨 |
| OS 인증 저장소 (Windows DPAPI 등) | 최고 | ✅ 지속됨 |

---

## 💡 확장 아이디어

- ✅ 토큰 만료 시 사용자 로그아웃 처리
- 🔄 자동 재시도 메커니즘 구현
- 🔒 역할(Role) 기반 UI 제어 (Admin만 보이는 메뉴 등)
- 🔑 OAuth2 Authorization Code Flow (Google, Kakao 로그인 등)

---

## 📚 참고 링크

- [RFC 6749: OAuth2.0 공식 스펙](https://tools.ietf.org/html/rfc6749)
- [ASP.NET Core JWT Auth 서버 구현 (API 쪽)](https://learn.microsoft.com/en-us/aspnet/core/security/authentication/jwt)