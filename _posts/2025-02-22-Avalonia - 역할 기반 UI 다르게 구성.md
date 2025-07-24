---
layout: post
title: Avalonia - 역할 기반 UI 다르게 구성
date: 2025-02-22 19:20:23 +0900
category: Avalonia
---
# 🔐 로그인 후 역할(Role) 기반 UI 다르게 구성하기

---

## 🎯 핵심 목표

| 항목 | 내용 |
|------|------|
| 서버에서 역할 정보 수신 | 로그인 시 서버에서 사용자 역할 포함 응답 |
| 전역 상태에 역할 저장 | AppAuthState 또는 AuthUserInfo 객체에 저장 |
| ViewModel/뷰에서 조건 분기 | `IsAdmin`, `IsEditor` 등의 속성으로 바인딩 |
| View 내 조건부 렌더링 | Avalonia의 `DataTrigger`, `Visible` 등으로 처리

---

## 1️⃣ 서버 응답 예시

서버에서 로그인 응답 시 역할 포함:

```json
{
  "access_token": "...",
  "refresh_token": "...",
  "expires_in": 3600,
  "role": "Admin"
}
```

> 또는 JWT 자체에 `role` 클레임 포함  
```json
{
  "sub": "user123",
  "role": "User",
  "exp": 1721142305
}
```

---

## 2️⃣ 역할 보관 모델 정의

### 📄 Models/AuthUserInfo.cs

```csharp
public class AuthUserInfo
{
    public string UserName { get; set; } = "";
    public string Role { get; set; } = "";

    public bool IsAdmin => Role == "Admin";
    public bool IsEditor => Role == "Editor" || Role == "Admin";
}
```

### 📄 AppAuthState에 포함

```csharp
public class AppAuthState
{
    public string? AccessToken { get; set; }
    public AuthUserInfo? User { get; set; }

    public bool IsAuthenticated => !string.IsNullOrEmpty(AccessToken);
}
```

---

## 3️⃣ AuthService에서 역할 설정

```csharp
public async Task<bool> LoginAsync(string username, string password)
{
    ...
    var result = JsonSerializer.Deserialize<AuthResponse>(body);

    _state.AccessToken = result.access_token;
    _state.RefreshToken = result.refresh_token;
    _state.TokenExpiresAt = DateTime.UtcNow.AddSeconds(result.expires_in);
    _state.User = new AuthUserInfo
    {
        UserName = result.username,
        Role = result.role
    };
}
```

---

## 4️⃣ ViewModel에서 역할 사용

```csharp
public class MainViewModel : ReactiveObject
{
    private readonly AppAuthState _auth;

    public bool IsAdmin => _auth.User?.IsAdmin ?? false;
    public bool IsEditor => _auth.User?.IsEditor ?? false;

    public MainViewModel(AppAuthState auth)
    {
        _auth = auth;
    }
}
```

---

## 5️⃣ View에서 역할 기반으로 렌더링

### ✅ 예: 관리자만 보이는 메뉴 항목

```xml
<Menu>
  <MenuItem Header="파일"/>
  <MenuItem Header="관리자" IsVisible="{Binding IsAdmin}">
    <MenuItem Header="사용자 관리"/>
    <MenuItem Header="시스템 로그"/>
  </MenuItem>
</Menu>
```

---

## 6️⃣ 역할별 네비게이션 제한

```csharp
if (!_auth.User?.IsAdmin ?? true)
{
    MessageBox.Show("권한이 없습니다.");
    return;
}
```

---

## 7️⃣ 전체 UI 상태 제어 (Shell 구조에서)

### 📄 ShellView.axaml

```xml
<Grid>
  <ContentControl Content="{Binding CurrentView}" />
</Grid>
```

### 📄 ShellViewModel.cs

```csharp
public object CurrentView { get; private set; }

public ShellViewModel(AppAuthState auth)
{
    if (auth.User?.IsAdmin == true)
        CurrentView = new AdminDashboardViewModel();
    else
        CurrentView = new UserDashboardViewModel();
}
```

---

## 8️⃣ 테스트를 위한 역할 지정 예시

```csharp
// 로그인 없이 테스트 시
authState.User = new AuthUserInfo
{
    UserName = "Test",
    Role = "Editor"
};
```

---

## ✅ 확장 아이디어

| 기능 | 설명 |
|------|------|
| ✅ 여러 역할 지원 (Admin, Manager, User) | User → 읽기, Manager → 작성, Admin → 전체 |
| ✅ Role에 따라 컨트롤 Disable 처리 | 버튼 Visible뿐만 아니라 `IsEnabled` |
| ✅ JWT에서 Role 클레임 직접 추출 | 서버가 JWT 클레임에 Role 포함 시 |
| ✅ 역할 변경에 따른 화면 리로드 | `RaisePropertyChanged`로 동기화 |

---

## 🧪 보안 팁

| 항목 | 설명 |
|------|------|
| 클라이언트 UI 제어는 UX용 | 보안은 서버에서 **역할 기반 API 제한**으로 필수 구현 |
| JWT의 `role` 클레임은 쉽게 변경 가능 | 민감 권한은 절대로 클라이언트에서만 검증 금지 |
| Refresh 흐름 시 역할도 함께 갱신 | 역할 변경된 사용자는 새 로그인 또는 Refresh 필요

---

## 🧭 실전 앱 예시

| 역할 | 가능 메뉴 |
|------|------------|
| Admin | 사용자관리, 로그보기, 설정 |
| Manager | 주문관리, 보고서 |
| User | 내 정보, 주문 조회 |

---

## 🧵 결론

- `AppAuthState` 또는 `UserContext`에 사용자 역할 저장
- ViewModel에서 `IsAdmin`, `IsEditor` 등을 통해 UI 제어
- View에서는 `IsVisible`, `IsEnabled`, `DataTrigger`로 제어
- 실제 권한 검사는 항상 **서버에서 이중 검증**

---

## 다음 주제로 추천

- 🔐 **JWT에서 역할 클레임 직접 파싱** (`System.IdentityModel.Tokens.Jwt`)
- ⚙️ **권한 기반 라우팅 제어**
- 🧬 **권한 기반 컴포넌트 재사용 (예: `<AuthOnly role="Admin">`)**