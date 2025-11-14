---
layout: post
title: Avalonia - 역할 기반 UI 다르게 구성
date: 2025-02-22 19:20:23 +0900
category: Avalonia
---
# Avalonia MVVM — 로그인 후 역할(Role) 기반 UI 구성 완전 가이드

핵심은 다음과 같다.

1) **역할 정보의 안정적 취득**: 로그인 응답 혹은 JWT 클레임에서 안전하게 파싱
2) **전역 상태·정책(Policy) 계층**: 단순 Role 문자열을 넘어서 권한 플래그/정책 평가 서비스로 일반화
3) **UI 제어 패턴**: 바인딩, `DataTrigger`/스타일, 컨버터, 첨부 동작(Attached Behavior)로 반복 제거
4) **네비게이션 가드**: 화면 전환 시 권한 점검 및 우회 차단
5) **토큰 갱신/역할 변경 동기화**: Refresh 시 역할 재로딩·UI 리바인딩
6) **테스트 전략**: ViewModel 단위, Policy 단위, 네비게이션 가드 단위 테스트

> 중요: **클라이언트의 역할/권한 제어는 UX 개선용일 뿐** 보안적으로는 불충분하다. **서버에서 API 권한을 반드시 재검증**해야 한다.

---

## 서버 응답/토큰 클레임 설계

### 로그인 응답에 역할 포함

```json
{
  "access_token": "eyJhbGciOiJIUzI1...",
  "refresh_token": "eyJhbGciOiJIUzI1...",
  "expires_in": 3600,
  "username": "user123",
  "role": "Admin"
}
```

### JWT 클레임에 역할 포함 (대안)

```json
{
  "sub": "user123",
  "role": "User",
  "exp": 1721142305,
  "permissions": ["orders.read", "orders.write"]
}
```

- 서버는 `role`(단일/다중) 혹은 `permissions`(세밀 권한) 중 하나 또는 둘 다를 제공할 수 있다.
- 클라이언트는 **두 경로 모두를 지원**해 상호 운용성을 확보한다.

---

## 모델·전역 상태 설계

### 역할/권한 모델

```csharp
public enum AppRole
{
    Unknown = 0,
    User    = 1,
    Manager = 2,
    Admin   = 3
}

[Flags]
public enum AppPermission
{
    None          = 0,
    OrdersRead    = 1 << 0,
    OrdersWrite   = 1 << 1,
    UsersRead     = 1 << 2,
    UsersWrite    = 1 << 3,
    SystemConfig  = 1 << 4,
}
```

### 사용자 정보 모델

```csharp
public class AuthUserInfo
{
    public string UserName { get; set; } = "";
    public AppRole Role { get; set; } = AppRole.Unknown;

    // 세밀 권한(선택)
    public AppPermission Permissions { get; set; } = AppPermission.None;

    public bool IsAdmin   => Role == AppRole.Admin;
    public bool IsManager => Role == AppRole.Manager || Role == AppRole.Admin;
    public bool IsUser    => Role == AppRole.User || IsManager || IsAdmin;

    public bool Has(AppPermission p) => (Permissions & p) == p;
}
```

### 전역 인증 상태

```csharp
public class AppAuthState : ReactiveUI.ReactiveObject
{
    private string? _accessToken;
    private string? _refreshToken;
    private DateTime _expiresAtUtc;

    private AuthUserInfo? _user;

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

    public DateTime TokenExpiresAtUtc
    {
        get => _expiresAtUtc;
        set => this.RaiseAndSetIfChanged(ref _expiresAtUtc, value);
    }

    public AuthUserInfo? User
    {
        get => _user;
        set => this.RaiseAndSetIfChanged(ref _user, value);
    }

    public bool IsAuthenticated =>
        !string.IsNullOrEmpty(AccessToken) && DateTime.UtcNow < TokenExpiresAtUtc;

    // 역할·권한 변경 브로드캐스트
    public void NotifyIdentityChanged()
    {
        this.RaisePropertyChanged(nameof(User));
        this.RaisePropertyChanged(nameof(IsAuthenticated));
    }
}
```

---

## AuthService — 로그인/Refresh + 역할 파싱

### 로그인 시 역할/권한 설정

```csharp
public sealed class AuthService : IAuthService
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
        var payload = JsonSerializer.Serialize(new { username, password });
        var resp = await _http.PostAsync("/api/auth/login",
            new StringContent(payload, Encoding.UTF8, "application/json"));

        if (!resp.IsSuccessStatusCode) return false;

        var body = await resp.Content.ReadAsStringAsync();
        var dto  = JsonSerializer.Deserialize<LoginResponse>(body);
        if (dto is null) return false;

        _state.AccessToken       = dto.access_token;
        _state.RefreshToken      = dto.refresh_token;
        _state.TokenExpiresAtUtc = DateTime.UtcNow.AddSeconds(dto.expires_in);

        var user = new AuthUserInfo
        {
            UserName = dto.username ?? username,
            Role     = MapRole(dto.role),
            Permissions = MapPermissions(dto.permissions)
        };

        // JWT가 오면 Claims에서 보강
        if (!string.IsNullOrWhiteSpace(dto.access_token))
        {
            var claims = JwtReader.TryReadClaims(dto.access_token);
            if (claims is not null)
            {
                if (claims.TryGetValue("role", out var roleVal))
                    user.Role = MapRole(roleVal);

                if (claims.TryGetValue("permissions", out var perValCsv))
                    user.Permissions = ParsePermissionsCsv(perValCsv);
            }
        }

        _state.User = user;
        _state.NotifyIdentityChanged();
        return true;
    }

    public async Task<bool> RefreshAsync()
    {
        if (string.IsNullOrEmpty(_state.RefreshToken)) return false;

        var payload = JsonSerializer.Serialize(new { refresh_token = _state.RefreshToken });
        var resp = await _http.PostAsync("/api/auth/refresh",
            new StringContent(payload, Encoding.UTF8, "application/json"));

        if (!resp.IsSuccessStatusCode) return false;

        var body = await resp.Content.ReadAsStringAsync();
        var dto  = JsonSerializer.Deserialize<RefreshResponse>(body);
        if (dto is null) return false;

        _state.AccessToken       = dto.access_token;
        _state.TokenExpiresAtUtc = DateTime.UtcNow.AddSeconds(dto.expires_in);

        // 토큰에 역할이 들어있다면 갱신
        var claims = JwtReader.TryReadClaims(dto.access_token);
        if (claims is not null && _state.User is not null)
        {
            if (claims.TryGetValue("role", out var roleVal))
                _state.User.Role = MapRole(roleVal);
            if (claims.TryGetValue("permissions", out var perValCsv))
                _state.User.Permissions = ParsePermissionsCsv(perValCsv);

            _state.NotifyIdentityChanged();
        }

        return true;
    }

    private static AppRole MapRole(string? s) => s?.ToLowerInvariant() switch
    {
        "admin"   => AppRole.Admin,
        "manager" => AppRole.Manager,
        "user"    => AppRole.User,
        _         => AppRole.Unknown
    };

    private static AppPermission MapPermissions(string[]? arr)
        => arr is null ? AppPermission.None
                       : arr.Select(MapPermission).Aggregate(AppPermission.None, (a, b) => a | b);

    private static AppPermission ParsePermissionsCsv(string csv)
        => MapPermissions(csv.Split(',', StringSplitOptions.RemoveEmptyEntries | StringSplitOptions.TrimEntries));

    private static AppPermission MapPermission(string s) => s.ToLowerInvariant() switch
    {
        "orders.read"   => AppPermission.OrdersRead,
        "orders.write"  => AppPermission.OrdersWrite,
        "users.read"    => AppPermission.UsersRead,
        "users.write"   => AppPermission.UsersWrite,
        "system.config" => AppPermission.SystemConfig,
        _               => AppPermission.None
    };

    private sealed class LoginResponse
    {
        public string access_token { get; set; } = "";
        public string refresh_token { get; set; } = "";
        public int    expires_in    { get; set; }
        public string? username     { get; set; }
        public string? role         { get; set; }
        public string[]? permissions{ get; set; }
    }

    private sealed class RefreshResponse
    {
        public string access_token { get; set; } = "";
        public int    expires_in    { get; set; }
    }
}
```

### JWT 파서 (의존성 최소화)

```csharp
public static class JwtReader
{
    // 외부 라이브러리 없이 페이로드만 디코딩(검증 없음, UX 목적)
    public static Dictionary<string, string>? TryReadClaims(string jwt)
    {
        try
        {
            var parts = jwt.Split('.');
            if (parts.Length < 2) return null;

            string payload = parts[1];
            payload = payload.Replace('-', '+').Replace('_', '/');
            switch (payload.Length % 4)
            {
                case 2: payload += "=="; break;
                case 3: payload += "="; break;
            }
            var bytes = Convert.FromBase64String(payload);
            var json  = Encoding.UTF8.GetString(bytes);

            using var doc = JsonDocument.Parse(json);
            var dict = new Dictionary<string, string>(StringComparer.OrdinalIgnoreCase);
            foreach (var p in doc.RootElement.EnumerateObject())
            {
                dict[p.Name] = p.Value.ValueKind switch
                {
                    JsonValueKind.String => p.Value.GetString() ?? "",
                    JsonValueKind.Number => p.Value.GetRawText(),
                    JsonValueKind.True   => "true",
                    JsonValueKind.False  => "false",
                    JsonValueKind.Array  => string.Join(',', p.Value.EnumerateArray().Select(e => e.GetString())),
                    _ => p.Value.GetRawText()
                };
            }
            return dict;
        }
        catch { return null; }
    }
}
```

> 주의: 위 파서는 **서명 검증을 수행하지 않는다.** 서버 API 접근 시는 **서버 측 권한 검증**을 반드시 통과해야 한다.

---

## 정책 계층: Role/Permission을 평가하는 AuthorizationService

UI 단마다 직접 `IsAdmin`을 붙이면 중복·누락이 발생한다. **정책 이름**으로 평가하도록 추상화한다.

```csharp
public interface IAuthorizationService
{
    bool IsInRole(AppRole role);
    bool Has(params AppPermission[] permissions);

    bool Evaluate(string policyName);
}

public sealed class AuthorizationService : IAuthorizationService
{
    private readonly AppAuthState _state;

    public AuthorizationService(AppAuthState state) => _state = state;

    public bool IsInRole(AppRole role) => _state.User?.Role == role;

    public bool Has(params AppPermission[] permissions)
    {
        var u = _state.User;
        if (u is null) return false;
        return permissions.All(u.Has);
    }

    public bool Evaluate(string policyName) => policyName switch
    {
        "AdminOnly"                 => IsInRole(AppRole.Admin),
        "ManageUsers"               => Has(AppPermission.UsersRead, AppPermission.UsersWrite),
        "OrdersReadOrWrite"         => Has(AppPermission.OrdersRead) || Has(AppPermission.OrdersWrite),
        "SystemConfig"              => Has(AppPermission.SystemConfig),
        _                           => false
    };
}
```

DI:

```csharp
services.AddSingleton<AppAuthState>();
services.AddSingleton<IAuthorizationService, AuthorizationService>();
```

---

## ViewModel — 역할/정책 쓰기

```csharp
public sealed class MainViewModel : ReactiveUI.ReactiveObject
{
    private readonly AppAuthState _auth;
    private readonly IAuthorizationService _authz;

    public bool CanSeeAdminMenu   => _authz.Evaluate("AdminOnly");
    public bool CanManageUsers    => _authz.Evaluate("ManageUsers");
    public bool CanConfigureSystem=> _authz.Evaluate("SystemConfig");

    public MainViewModel(AppAuthState auth, IAuthorizationService authz)
    {
        _auth  = auth;
        _authz = authz;

        _auth.Changed.Subscribe(_ =>
        {
            this.RaisePropertyChanged(nameof(CanSeeAdminMenu));
            this.RaisePropertyChanged(nameof(CanManageUsers));
            this.RaisePropertyChanged(nameof(CanConfigureSystem));
        });
    }
}
```

---

## View — 역할 기반 렌더링 패턴 모음

### 단순 바인딩(IsVisible/IsEnabled)

```xml
<Menu>
  <MenuItem Header="파일" />
  <MenuItem Header="관리자" IsVisible="{Binding CanSeeAdminMenu}">
    <MenuItem Header="사용자 관리" IsEnabled="{Binding CanManageUsers}" />
    <MenuItem Header="시스템 설정" IsEnabled="{Binding CanConfigureSystem}" />
  </MenuItem>
</Menu>
```

### DataTrigger/Style로 반복 제거

```xml
<UserControl.Styles>
  <Style Selector="Button#AdminOnly">
    <Style.Triggers>
      <DataTrigger Binding="{Binding CanSeeAdminMenu}" Value="False">
        <Setter Property="IsVisible" Value="False"/>
      </DataTrigger>
    </Style.Triggers>
  </Style>
</UserControl.Styles>

<StackPanel>
  <Button x:Name="AdminOnly" Content="관리자 패널"/>
</StackPanel>
```

### 컨버터 사용 (Policy 이름 → bool)

```csharp
public sealed class PolicyToBoolConverter : IValueConverter
{
    public object? Convert(object? value, Type targetType, object? parameter, CultureInfo culture)
    {
        if (parameter is not string policy) return false;
        var authz = App.Services.GetRequiredService<IAuthorizationService>();
        return authz.Evaluate(policy);
    }

    public object? ConvertBack(object? value, Type targetType, object? parameter, CultureInfo culture)
        => throw new NotSupportedException();
}
```

리소스 등록 후:

```xml
<UserControl.Resources>
  <local:PolicyToBoolConverter x:Key="Policy"/>
</UserControl.Resources>

<Button Content="사용자 관리"
        IsVisible="{Binding ., Converter={StaticResource Policy}, ConverterParameter=ManageUsers}"/>
```

### Attached Behavior — 선언형 권한 태그

```csharp
public static class AuthBehaviors
{
    public static readonly AttachedProperty<string?> PolicyProperty =
        AvaloniaProperty.RegisterAttached<Interactive, string?>("Policy", typeof(AuthBehaviors));

    static AuthBehaviors()
    {
        PolicyProperty.Changed.Subscribe(args =>
        {
            if (args.Sender is Interactive target)
            {
                var policy = args.NewValue.GetValueOrDefault<string?>();
                target.AttachedToVisualTree += (_, __) =>
                {
                    var authz = App.Services.GetRequiredService<IAuthorizationService>();
                    var ok = !string.IsNullOrEmpty(policy) && authz.Evaluate(policy!);
                    if (target is Control c) c.IsVisible = ok;
                    // 필요시 IsEnabled도 함께 제어
                };
            }
        });
    }

    public static void SetPolicy(AvaloniaObject o, string? value) => o.SetValue(PolicyProperty, value);
    public static string? GetPolicy(AvaloniaObject o) => o.GetValue(PolicyProperty);
}
```

XAML:

```xml
<Button Content="시스템 설정"
        local:AuthBehaviors.Policy="SystemConfig"/>
```

> 이 패턴은 **중복 로직을 XAML 속성 한 줄로 치환**해 대규모 화면에서 생산성을 높인다.

---

## 네비게이션 가드(라우팅 제어)

### 서비스 정의

```csharp
public interface INavigationGuard
{
    bool CanNavigateTo(Type viewModelType);
}

public sealed class RoleNavigationGuard : INavigationGuard
{
    private readonly IAuthorizationService _authz;

    public RoleNavigationGuard(IAuthorizationService authz) => _authz = authz;

    public bool CanNavigateTo(Type vmType) => vmType.Name switch
    {
        "AdminDashboardViewModel"   => _authz.Evaluate("AdminOnly"),
        "UserManagementViewModel"   => _authz.Evaluate("ManageUsers"),
        _                            => true
    };
}
```

DI:

```csharp
services.AddSingleton<INavigationGuard, RoleNavigationGuard>();
```

### Shell에서 가드 적용

```csharp
public sealed class ShellViewModel : ReactiveUI.ReactiveObject
{
    private readonly INavigationGuard _guard;
    public object? CurrentView { get; private set; }

    public ShellViewModel(INavigationGuard guard)
    {
        _guard = guard;
    }

    public void NavigateTo<TVm>(Func<TVm> factory)
    {
        var t = typeof(TVm);
        if (!_guard.CanNavigateTo(t))
        {
            // UX 알림
            // MessageBox.Show("권한이 없습니다.");
            return;
        }
        CurrentView = factory();
        this.RaisePropertyChanged(nameof(CurrentView));
    }
}
```

---

## API 호출 — 인증 핸들러와 403 처리

권한 부족 시 서버는 **403 Forbidden**을 반환해야 한다. 클라이언트는 UX로 안내하고, 필요한 경우 **역할 재동기화**를 시도한다.

```csharp
public sealed class AuthenticatedHttpHandler : DelegatingHandler
{
    private readonly AppAuthState _state;
    private readonly IAuthService _auth;

    public AuthenticatedHttpHandler(AppAuthState s, IAuthService a)
    { _state = s; _auth = a; }

    protected override async Task<HttpResponseMessage> SendAsync(HttpRequestMessage request, CancellationToken ct)
    {
        if (!_state.IsAuthenticated)
        {
            if (!await _auth.RefreshAsync())
                throw new UnauthorizedAccessException("세션 만료");
        }

        request.Headers.Authorization =
            new AuthenticationHeaderValue("Bearer", _state.AccessToken);

        var resp = await base.SendAsync(request, ct);

        if (resp.StatusCode == HttpStatusCode.Forbidden)
        {
            // 역할이 변경되었을 수 있으므로 최신 토큰/역할 재확인(정책)
            // 필요시 알림/로그
        }
        return resp;
    }
}
```

---

## 역할 변경 실시간 반영

- `Refresh` 또는 별도 `GET /me` 호출로 역할·권한 갱신
- `_state.NotifyIdentityChanged()` 호출 → **모든 바인딩 재평가**
- `MainViewModel` 등에서 `Changed` 구독으로 파생 속성 갱신

---

## 예시 화면 구성

```xml
<DockPanel>
  <Menu DockPanel.Dock="Top">
    <MenuItem Header="파일"/>
    <MenuItem Header="관리자" IsVisible="{Binding CanSeeAdminMenu}">
      <MenuItem Header="사용자 관리" IsEnabled="{Binding CanManageUsers}"/>
      <MenuItem Header="시스템 설정" IsEnabled="{Binding CanConfigureSystem}"/>
    </MenuItem>
  </Menu>

  <ContentControl Content="{Binding CurrentView}" />
</DockPanel>
```

---

## 테스트 전략

### AuthorizationService 테스트

```csharp
[Fact]
public void Policy_AdminOnly_True_For_Admin()
{
    var state = new AppAuthState
    {
        User = new AuthUserInfo { Role = AppRole.Admin }
    };
    var svc = new AuthorizationService(state);

    svc.Evaluate("AdminOnly").Should().BeTrue();
}
```

### NavigationGuard 테스트

```csharp
[Fact]
public void Guard_Blocks_AdminDashboard_For_User()
{
    var state = new AppAuthState { User = new AuthUserInfo { Role = AppRole.User } };
    var authz = new AuthorizationService(state);
    var guard = new RoleNavigationGuard(authz);

    guard.CanNavigateTo(typeof(AdminDashboardViewModel)).Should().BeFalse();
}
```

### ViewModel 바인딩 테스트

```csharp
[Fact]
public void MainVm_Reacts_To_Role_Change()
{
    var state = new AppAuthState { User = new AuthUserInfo { Role = AppRole.User } };
    var vm = new MainViewModel(state, new AuthorizationService(state));
    vm.CanSeeAdminMenu.Should().BeFalse();

    state.User!.Role = AppRole.Admin;
    state.NotifyIdentityChanged();
    vm.CanSeeAdminMenu.Should().BeTrue();
}
```

---

## 운영 체크리스트

- 로그인/Refresh 시 **역할과 권한을 최신화**, UI 재평가
- **권한 실패(403)** 처리: 안내·재로그인/역할 재동기화
- **로깅/감사**: 중요한 관리 메뉴 접근 시도 기록
- **다국어(i18n)**: 정책명 → 리소스 메시지 매핑
- **구성 파일**로 정책-뷰매핑 관리(핫스왑) 가능

---

## 보안 유의점

- **UI 숨김은 방어가 아니다**: 공격자는 네트워크 요청을 직접 보낼 수 있다. 서버는 **항상 재검증**
- 토큰 저장 시 **OS 보호 저장소**(Windows DPAPI 등) 고려
- JWT 디코딩은 UX 보조 용도이며, **서명 검증은 서버 책임**
- 역할 변경 시 사용자를 재인증하거나 Refresh로 **권한 즉시 반영**

---

## 전체 예제 묶음(간단 조립)

```csharp
// DI
services.AddSingleton<AppAuthState>();
services.AddSingleton<IAuthorizationService, AuthorizationService>();
services.AddSingleton<IAuthService, AuthService>();
services.AddTransient<INavigationGuard, RoleNavigationGuard>();
services.AddTransient<MainViewModel>();
services.AddTransient<ShellViewModel>();

// ViewModel 사용
var shell = Services.GetRequiredService<ShellViewModel>();
// 로그인 후:
await Services.GetRequiredService<IAuthService>().LoginAsync("admin", "1234");
shell.NavigateTo(() => new AdminDashboardViewModel());
```

---

## 결론

- **전역 상태(AppAuthState)** + **정책 서비스(AuthorizationService)**로 역할/권한 로직을 한 곳에 모으고,
- ViewModel은 **정책 평가 결과**만 바인딩, View는 **Style/DataTrigger/Attached Behavior**로 선언형 렌더링,
- **네비게이션 가드**로 뷰 전환을 보호하며,
- **Refresh/역할 변경 시 재동기화**하여 UI를 일관되게 유지한다.

실전에서는 위 패턴을 기반으로 **정책 이름·권한 플래그를 구성 파일/서버에서 내려받아** 핫스왑하는 등 운영 자동화를 더하면, 대규모 화면에서도 일관된 권한 모델을 유지할 수 있다.

---

## 다음 확장 주제 제안

- JWT 서명 검증·만료 처리와 **백그라운드 토큰 갱신 전략**
- 모놀리식 정책을 **모듈별 Policy Provider**로 분리하는 아키텍처
- **A/B 권한 실험**과 기능 토글(Feature Flag) 연동
- **오프라인 모드**에서의 권한 캐싱·제한적 UI 동작
