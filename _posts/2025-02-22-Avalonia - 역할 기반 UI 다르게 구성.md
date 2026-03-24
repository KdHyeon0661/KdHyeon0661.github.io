---
layout: post
title: Avalonia - 역할 기반 UI 다르게 구성
date: 2025-02-22 19:20:23 +0900
category: Avalonia
---
# Avalonia - 역할 기반 UI 다르게 구성

사용자 역할(role)에 따라 UI를 다르게 보여주는 것은 관리자 화면, 기능 접근 제한, 사용자별 메뉴 구성 등에서 필수적입니다. Avalonia 애플리케이션에서 역할 기반 UI를 구현할 때는 **전역 상태 관리**, **정책 평가 서비스**, **선언적 XAML 바인딩**, **네비게이션 가드**를 함께 구성해야 합니다. 이 글에서는 초중급 개발자를 기준으로, 역할과 권한 정보를 안전하게 취득하고, UI를 일관되게 제어하는 방법을 단계별로 설명합니다.

> **중요**: 클라이언트에서 역할을 확인하여 UI를 숨기는 것은 사용자 경험을 위한 것입니다. 실제 보안은 **서버에서 반드시 재검증**해야 합니다.

---

## 역할 정보의 취득

역할 정보는 로그인 응답에 포함되거나 JWT(JSON Web Token)의 클레임(claim)에 포함될 수 있습니다. 두 경우 모두 처리할 수 있도록 준비합니다.

### 서버 응답 예시 (로그인 성공)

```json
{
  "access_token": "eyJhbGciOiJIUzI1...",
  "refresh_token": "eyJhbGciOiJIUzI1...",
  "expires_in": 3600,
  "username": "user123",
  "role": "Admin",
  "permissions": ["orders.read", "orders.write"]
}
```

### JWT 클레임 예시

```json
{
  "sub": "user123",
  "role": "User",
  "exp": 1721142305,
  "permissions": ["orders.read"]
}
```

---

## 모델 정의

### 역할과 권한 열거형

역할은 계층적이거나 단순 문자열로 구분할 수 있습니다. 권한은 비트 플래그(Flags)로 정의하면 조합이 쉽습니다.

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
    public AppPermission Permissions { get; set; } = AppPermission.None;

    public bool IsAdmin   => Role == AppRole.Admin;
    public bool IsManager => Role == AppRole.Manager || Role == AppRole.Admin;
    public bool IsUser    => Role == AppRole.User || IsManager || IsAdmin;

    public bool Has(AppPermission p) => (Permissions & p) == p;
}
```

---

## 전역 인증 상태

`AppAuthState`는 액세스 토큰, 리프레시 토큰, 만료 시각, 사용자 정보를 보관합니다. ReactiveUI의 `ReactiveObject`를 상속받아 변경 시 UI에 알립니다.

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

    // 역할/권한 변경 시 바인딩 갱신용
    public void NotifyIdentityChanged()
    {
        this.RaisePropertyChanged(nameof(User));
        this.RaisePropertyChanged(nameof(IsAuthenticated));
    }
}
```

---

## AuthService에서 역할 파싱

로그인 응답과 JWT에서 역할과 권한을 추출하여 `AppAuthState.User`에 설정합니다.

### 간단한 JWT 페이로드 파서 (서명 미검증, UI 목적)

```csharp
public static class JwtReader
{
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
            var json = Encoding.UTF8.GetString(bytes);
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

### AuthService에서 로그인 처리

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
        // ... (API 호출) ...
        var dto = await ParseLoginResponse(resp);
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

        // JWT에서 추가 클레임 보강
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

    private static AppPermission MapPermission(string s) => s.ToLowerInvariant() switch
    {
        "orders.read"   => AppPermission.OrdersRead,
        "orders.write"  => AppPermission.OrdersWrite,
        "users.read"    => AppPermission.UsersRead,
        "users.write"   => AppPermission.UsersWrite,
        "system.config" => AppPermission.SystemConfig,
        _               => AppPermission.None
    };
}
```

---

## 정책 평가 서비스 (AuthorizationService)

직접 `IsAdmin` 등의 속성을 ViewModel에 여러 번 구현하면 중복이 발생합니다. **정책(policy)** 이름으로 평가하는 서비스를 만들어 모든 권한 판단을 중앙화합니다.

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
        "AdminOnly"      => IsInRole(AppRole.Admin),
        "ManageUsers"    => Has(AppPermission.UsersRead, AppPermission.UsersWrite),
        "OrdersReadOrWrite" => Has(AppPermission.OrdersRead) || Has(AppPermission.OrdersWrite),
        "SystemConfig"   => Has(AppPermission.SystemConfig),
        _ => false
    };
}
```

DI 컨테이너에 등록:

```csharp
services.AddSingleton<AppAuthState>();
services.AddSingleton<IAuthorizationService, AuthorizationService>();
```

---

## ViewModel에서 권한 사용

ViewModel에서는 `IAuthorizationService`를 주입받아 정책 평가 결과를 속성으로 노출합니다. 권한이 바뀔 때마다 속성 변경을 알리기 위해 `AppAuthState`의 변경을 구독합니다.

```csharp
public sealed class MainViewModel : ReactiveObject
{
    private readonly IAuthorizationService _authz;
    private readonly AppAuthState _authState;

    public MainViewModel(AppAuthState authState, IAuthorizationService authz)
    {
        _authState = authState;
        _authz = authz;

        // 사용자 정보 변경 시 모든 권한 관련 속성 갱신
        _authState.Changed.Subscribe(_ =>
        {
            this.RaisePropertyChanged(nameof(CanSeeAdminMenu));
            this.RaisePropertyChanged(nameof(CanManageUsers));
            this.RaisePropertyChanged(nameof(CanConfigureSystem));
        });
    }

    public bool CanSeeAdminMenu   => _authz.Evaluate("AdminOnly");
    public bool CanManageUsers    => _authz.Evaluate("ManageUsers");
    public bool CanConfigureSystem=> _authz.Evaluate("SystemConfig");
}
```

---

## XAML에서 역할 기반 UI 구성

### 1. 단순 바인딩 (IsVisible, IsEnabled)

```xml
<Menu>
  <MenuItem Header="파일" />
  <MenuItem Header="관리자" IsVisible="{Binding CanSeeAdminMenu}">
    <MenuItem Header="사용자 관리" IsEnabled="{Binding CanManageUsers}" />
    <MenuItem Header="시스템 설정" IsEnabled="{Binding CanConfigureSystem}" />
  </MenuItem>
</Menu>
```

### 2. DataTrigger를 이용한 스타일

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

### 3. 컨버터로 정책명을 직접 전달

정책 이름을 ConverterParameter로 전달하여 XAML에서 바로 권한을 평가할 수 있습니다.

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

리소스 등록:

```xml
<UserControl.Resources>
  <local:PolicyToBoolConverter x:Key="Policy"/>
</UserControl.Resources>
```

사용:

```xml
<Button Content="사용자 관리"
        IsVisible="{Binding ., Converter={StaticResource Policy}, ConverterParameter=ManageUsers}"/>
```

### 4. Attached Behavior (선언적 속성)

컨버터보다 더 깔끔하게, 특정 정책이 거짓일 때 컨트롤을 숨기는 첨부 동작을 정의할 수 있습니다.

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

이 방식은 XAML 속성 한 줄로 권한 제어를 할 수 있어 대규모 화면에서 유지보수성이 높아집니다.

---

## 네비게이션 가드 (Route Guard)

권한이 없는 화면으로의 전환을 막으려면 네비게이션 가드를 도입합니다.

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
        _                           => true
    };
}
```

Shell ViewModel에서 가드를 적용:

```csharp
public sealed class ShellViewModel : ReactiveObject
{
    private readonly INavigationGuard _guard;
    private object? _currentView;

    public object? CurrentView
    {
        get => _currentView;
        set => this.RaiseAndSetIfChanged(ref _currentView, value);
    }

    public ShellViewModel(INavigationGuard guard)
    {
        _guard = guard;
    }

    public void NavigateTo<TVm>(Func<TVm> factory)
    {
        if (!_guard.CanNavigateTo(typeof(TVm)))
        {
            // 사용자에게 권한 부족 알림 (예: 다이얼로그)
            // MessageBox.Show("접근 권한이 없습니다.");
            return;
        }
        CurrentView = factory();
    }
}
```

---

## API 응답 403 처리

서버에서 권한 부족으로 403 Forbidden을 반환하면 사용자에게 알리고, 역할이 변경되었을 가능성을 고려해 최신 정보를 다시 가져올 수 있습니다.

```csharp
protected override async Task<HttpResponseMessage> SendAsync(HttpRequestMessage request, CancellationToken ct)
{
    // ... 토큰 추가 ...
    var resp = await base.SendAsync(request, ct);
    if (resp.StatusCode == HttpStatusCode.Forbidden)
    {
        // 필요 시 토큰 갱신 또는 역할 재조회
        // 사용자에게 권한 부족 안내
    }
    return resp;
}
```

---

## 역할 변경 시 UI 갱신

사용자 역할이 변경될 수 있는 경우(예: 관리자가 다른 사용자에게 권한을 부여), 앱은 주기적으로 또는 푸시 알림을 통해 새 역할을 받아와야 합니다. `AppAuthState`의 `User`를 업데이트하고 `NotifyIdentityChanged()`를 호출하면 모든 바인딩이 자동으로 재평가됩니다.

```csharp
// 예: 웹소켓으로 역할 변경 알림 수신
void OnRoleChanged(AppRole newRole)
{
    if (_state.User != null)
    {
        _state.User.Role = newRole;
        _state.NotifyIdentityChanged();
    }
}
```

---

## 테스트 전략

### AuthorizationService 단위 테스트

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

[Fact]
public void Policy_ManageUsers_False_For_User()
{
    var state = new AppAuthState
    {
        User = new AuthUserInfo { Role = AppRole.User }
    };
    var svc = new AuthorizationService(state);
    svc.Evaluate("ManageUsers").Should().BeFalse();
}
```

### ViewModel 권한 반응 테스트

```csharp
[Fact]
public void MainViewModel_Reacts_To_Role_Change()
{
    var state = new AppAuthState
    {
        User = new AuthUserInfo { Role = AppRole.User }
    };
    var authz = new AuthorizationService(state);
    var vm = new MainViewModel(state, authz);

    vm.CanSeeAdminMenu.Should().BeFalse();

    state.User!.Role = AppRole.Admin;
    state.NotifyIdentityChanged();

    vm.CanSeeAdminMenu.Should().BeTrue();
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

---

## 운영 체크리스트

- [ ] 로그인/Refresh 시 역할과 권한을 항상 최신으로 설정하고 UI 갱신
- [ ] API 호출에서 403 발생 시 적절한 사용자 안내와 역할 재동기화
- [ ] 중요 관리 메뉴 접근 로그 기록
- [ ] 다국어 지원 시 정책 이름을 리소스 키로 매핑
- [ ] 정책 정의를 구성 파일로 분리하여 핫스왑 가능하도록 설계 (선택)

---

## 보안 유의점

- **UI 숨김은 보안이 아니다**: 공격자는 네트워크 요청을 직접 보낼 수 있습니다. 서버는 모든 요청에 대해 권한을 재검증해야 합니다.
- **토큰 저장**: 민감한 토큰은 OS 보안 저장소(Windows DPAPI, macOS Keychain, Linux SecretService)에 저장하세요.
- **JWT 디코딩**: 클라이언트에서 JWT를 디코딩할 때 서명을 검증하지 않아도 됩니다. 단, UI 힌트로만 사용하고 실제 권한 결정은 서버에서 내려야 합니다.
- **역할 변경 시 강제 로그아웃**: 보안이 중요한 앱에서는 역할 변경 시 모든 토큰을 무효화하고 재로그인을 요구할 수 있습니다.

---

## 요약

| 구성 요소 | 역할 |
|-----------|------|
| `AppAuthState` | 인증 정보와 사용자 역할/권한 저장, 변경 알림 |
| `IAuthorizationService` | 정책 평가 중앙 서비스, ViewModel과 UI가 권한 질의 |
| ViewModel | `IAuthorizationService`로 정책 평가 결과 노출, `AppAuthState` 변경 구독 |
| XAML | 바인딩, DataTrigger, 컨버터, Attached Behavior로 선언적 제어 |
| `INavigationGuard` | 화면 전환 시 권한 확인, 차단 |
| API 핸들러 | 403 응답 처리 및 역할 동기화 |

이 구조를 통해 역할 기반 UI를 **일관되고 유지보수하기 쉽게** 구현할 수 있습니다. 클라이언트의 권한 제어는 사용자 경험을 위한 것이며, 실제 보안은 서버에서 책임진다는 점을 항상 기억하세요.