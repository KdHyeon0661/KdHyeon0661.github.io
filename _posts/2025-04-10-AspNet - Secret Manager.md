---
layout: post
title: AspNet - Secret Manager
date: 2025-04-10 21:20:23 +0900
category: AspNet
---
# ASP.NET Core 시크릿 매니저(Secret Manager) 사용법

## 1. Secret Manager란? — “개발 시 로컬에서 비밀을 소스코드 밖에 두는 도구”

- **목적**: 개발 환경에서 **민감 정보(API 키, DB 비밀번호 등)**를 `appsettings.json`에 넣지 않고, **개발자 로컬 프로필**에 따로 저장해 **버전관리(Git)에서 격리**.
- **저장 방식**: 사용자 프로필 경로의 **JSON 파일**에 저장된다.  
  **중요**: 이 파일은 **암호화되어 저장되지 않는다**(로컬 사용자 프로필 권한에 의존). “보안 금고”가 아니라 **개발 시 편의/격리** 도구다.
- **로드 조건**: 일반 템플릿은 **Development 환경**일 때 자동 포함.  
  운영(Production)에서는 **환경 변수/Key Vault** 등으로 전환 권장.

### 실제 저장 경로
| OS | 경로 |
|----|------|
| Windows | `%APPDATA%\Microsoft\UserSecrets\{UserSecretsId}\secrets.json` |
| Linux/macOS | `~/.microsoft/usersecrets/{UserSecretsId}/secrets.json` |

> 동일 PC의 **해당 사용자**만 접근(권한 전제). **운영 비밀 저장소**로 사용하면 안 된다.

---

## 2. 사용 전 준비

- .NET SDK 설치
- 프로젝트(`.csproj`)에 `UserSecretsId` 존재(없으면 `init`로 생성)
- 현재 환경 `ASPNETCORE_ENVIRONMENT=Development`(일반 템플릿은 개발일 때 자동 로드)

---

## 3. 프로젝트에 Secret Manager 활성화

```bash
dotnet user-secrets init
```

실행 후 `.csproj`에 다음이 추가된다:

```xml
<PropertyGroup>
  <UserSecretsId>your-project-guid-or-id</UserSecretsId>
</PropertyGroup>
```

> 여러 프로젝트가 있을 때는 **각 프로젝트별로** `UserSecretsId`가 다르게 생긴다.  
> 특정 프로젝트를 명시하려면 `--project <csproj 경로>` 옵션 사용 가능.

---

## 4. 비밀 값 저장/조회/삭제 — CLI 실습

### 4.1 저장(Set)

```bash
dotnet user-secrets set "JwtSettings:SecretKey" "MySuperSecretKey123"
dotnet user-secrets set "ConnectionStrings:DefaultConnection" "Server=.;Database=AppDb;User Id=app;Password=P@ssw0rd;"
dotnet user-secrets set "ThirdParty:Stripe:ApiKey" "sk_test_xxx"
dotnet user-secrets set "Smtp:Host" "smtp.example.com"
dotnet user-secrets set "Smtp:Port" "587"
dotnet user-secrets set "Smtp:Username" "user@example.com"
dotnet user-secrets set "Smtp:Password" "app-specific-password"
```

- `:`로 **계층 키** 표현
- 문자열 이외 타입(정수/불린/TimeSpan 등)도 문자열로 저장되며, **바인딩 시 변환**된다.

### 4.2 조회(List)

```bash
dotnet user-secrets list
```

출력 예:
```text
JwtSettings:SecretKey = MySuperSecretKey123
ConnectionStrings:DefaultConnection = Server=.;Database=AppDb;User Id=app;Password=P@ssw0rd;
ThirdParty:Stripe:ApiKey = sk_test_xxx
Smtp:Host = smtp.example.com
Smtp:Port = 587
Smtp:Username = user@example.com
Smtp:Password = app-specific-password
```

### 4.3 삭제(Remove/Clear)

```bash
dotnet user-secrets remove "JwtSettings:SecretKey"
dotnet user-secrets clear
```

> `clear`는 해당 프로젝트의 모든 시크릿 삭제.

---

## 5. 코드에서 사용하기 — `IConfiguration`/Options 패턴

### 5.1 기본: `IConfiguration`로 직접 읽기

```csharp
var builder = WebApplication.CreateBuilder(args);
// 템플릿 기본 동작: Development이면 user-secrets 자동 포함

var secretKey = builder.Configuration["JwtSettings:SecretKey"];
var conn = builder.Configuration.GetConnectionString("DefaultConnection");
Console.WriteLine($"SecretKey? {secretKey is not null}, Conn? {conn is not null}");
```

### 5.2 Options 바인딩(권장)

```csharp
public sealed class JwtSettings
{
    public string SecretKey { get; set; } = "";
    public string Issuer { get; set; } = "myapi";
    public string Audience { get; set; } = "myclient";
    public int ExpiresMinutes { get; set; } = 60;
}

builder.Services.AddOptions<JwtSettings>()
    .Bind(builder.Configuration.GetSection("JwtSettings"))
    .Validate(s => !string.IsNullOrWhiteSpace(s.SecretKey), "SecretKey required")
    .ValidateOnStart();

var app = builder.Build();
app.MapGet("/jwt-check", (IOptions<JwtSettings> opt) => Results.Json(opt.Value));
app.Run();
```

> 시크릿 값과 `appsettings.*.json` 값이 **병합**되며, **나중에 추가된 소스가 덮어쓴다**.  
> 보통 템플릿 순서는: 기본 파일 → 환경별 파일 → **User Secrets(개발)** → 환경 변수 → 명령줄.

---

## 6. EF Core/DB 연결 문자열을 Secret로 관리

### 6.1 CLI로 저장

```bash
dotnet user-secrets set "ConnectionStrings:DefaultConnection" "Server=localhost;Database=AppDev;User Id=dev;Password=devpass;"
```

### 6.2 `Program.cs` DbContext 구성

```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));
```

### 6.3 마이그레이션/실행 테스트

```bash
dotnet ef migrations add Init
dotnet ef database update
dotnet run
```

> 개발자별로 **다른 DB 접속 정보**를 시크릿에 보관하면, `appsettings.json`은 공통값만 유지해 깔끔하다.

---

## 7. JWT/서드파티 키 — Secret + Swagger 연동까지

### 7.1 시크릿 저장

```bash
dotnet user-secrets set "JwtSettings:SecretKey" "dev-only-ultra-secret"
```

### 7.2 JWT 인증 구성

```csharp
builder.Services.AddAuthentication("Bearer")
    .AddJwtBearer("Bearer", o =>
    {
        var jwt = builder.Configuration.GetSection("JwtSettings");
        var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(jwt["SecretKey"]!));
        o.TokenValidationParameters = new()
        {
            ValidateIssuer = true, ValidateAudience = true, ValidateLifetime = true, ValidateIssuerSigningKey = true,
            ValidIssuer = jwt["Issuer"], ValidAudience = jwt["Audience"], IssuerSigningKey = key
        };
    });

builder.Services.AddAuthorization();
```

### 7.3 Swagger에서 “Authorize” 입력 허용(선택)

```csharp
builder.Services.AddSwaggerGen(c =>
{
    c.AddSecurityDefinition("Bearer", new OpenApiSecurityScheme
    {
        Name = "Authorization", Type = SecuritySchemeType.Http, Scheme = "bearer",
        BearerFormat = "JWT", In = ParameterLocation.Header
    });
    c.AddSecurityRequirement(new OpenApiSecurityRequirement {
        { new OpenApiSecurityScheme{ Reference = new OpenApiReference{ Type=ReferenceType.SecurityScheme, Id="Bearer"}}, new string[]{} }
    });
});
```

---

## 8. 우선순위/덮어쓰기 규칙 — Secret은 어디에 끼어드나?

일반 템플릿의 구성 추가 순서 예(아래로 갈수록 우선순위 ↑):

1. `appsettings.json`
2. `appsettings.{Environment}.json`
3. **User Secrets** (Development일 때)
4. 환경 변수
5. 명령줄 인자

> 즉, **환경 변수/명령줄**이 **Secret보다 더 강하게 덮어쓴다**.  
> 팀원별 로컬 개발 값(Secret)을 기본으로, 필요 시 **명령줄로 임시 오버라이드**가 가능하다.

---

## 9. 멀티 프로젝트/솔루션/테스트에서의 사용

### 9.1 여러 프로젝트가 같은 비밀을 공유해야 한다면?

- 가장 단순한 방법: **각 프로젝트에 동일 키를 각각 설정**(개발자 편의 스크립트 마련).
- 혹은 **한 프로젝트의 `UserSecretsId`를 다른 프로젝트로 복사**(의도적으로 동일 ID).  
  다만 프로젝트 분리 간 **의존/누수**를 초래할 수 있어 설계 상 신중히.

### 9.2 통합 테스트 프로젝트

- 테스트 프로젝트에도 `dotnet user-secrets init` 수행 후 테스트 전용 비밀 주입.
- `WebApplicationFactory`를 사용하는 경우 **테스트 프로젝트 환경**이 Development로 동작하도록 설정하거나, `ConfigureAppConfiguration`에서 `AddUserSecrets`를 명시 호출.

```csharp
public class TestFactory : WebApplicationFactory<Program>
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureAppConfiguration((ctx, cfg) =>
        {
            cfg.AddUserSecrets<Program>(optional: true); // 테스트에서도 로딩
        });
    }
}
```

---

## 10. Docker/Kubernetes/CI-CD와 Secret Manager

- Secret Manager는 **호스트(개발 PC) 사용자 프로필**에 저장되므로, **컨테이너 내부**에서는 접근 불가.
- 컨테이너/클러스터 배포 시:
  - **Docker**: `-e` 환경 변수를 통해 주입.
  - **Docker Compose**: `.env` 혹은 `secrets`(스웜) 사용.
  - **Kubernetes**: **ConfigMap/Secret**(환경 변수 또는 파일 마운트)로 주입.
- CI/CD(예: GitHub Actions, Azure DevOps): 파이프라인의 **보안 변수 스토어**를 사용하여 ENV로 주입.

> 개발 중에는 **Secret Manager**, 운영/배포에서는 **ENV/플랫폼 Secret**로 전환이 정석.

---

## 11. 설정 진단 — 안전하게 값 확인하기

실무 중 “왜 값이 안 들어오지?”를 빨리 확인하려면 **일부 키만 노출**하는 진단 엔드포인트를 만든다.

```csharp
app.MapGet("/__cfg-check", (IConfiguration cfg) =>
{
    // 민감 값은 절대 그대로 내보내지 말 것!
    var safe = new {
        Env = Environment.GetEnvironmentVariable("ASPNETCORE_ENVIRONMENT"),
        FeatureEnabled = cfg.GetValue<bool?>("MySettings:FeatureEnabled"),
        // 존재 여부만 True/False로 확인
        HasJwtSecret = !string.IsNullOrEmpty(cfg["JwtSettings:SecretKey"]),
        HasConnStr = !string.IsNullOrEmpty(cfg.GetConnectionString("DefaultConnection"))
    };
    return Results.Json(safe);
});
```

- 운영 배포 전에는 반드시 제거/보호.

---

## 12. 자주 하는 실수 & 트러블슈팅

| 증상 | 원인 | 해결 |
|------|------|------|
| 값이 전혀 로드되지 않음 | 환경이 Development가 아님 | `ASPNETCORE_ENVIRONMENT=Development` 설정 또는 `AddUserSecrets<Program>()` 명시 추가 |
| `dotnet user-secrets list` 값은 있는데 코드에서 null | **프로젝트 디렉터리**가 달라 CLI가 다른 프로젝트에 저장 | `--project` 옵션으로 명시, 올바른 `.csproj`에서 명령 실행 |
| 동일 키가 다른 값으로 보임 | **ENV/명령줄**이 Secret을 덮어씀 | `env`/명령줄 인자 확인, 우선순위 이해 |
| 팀원 A/B 비밀이 서로 다름 | Secret은 **사용자 프로필**별/프로젝트별 | 팀 온보딩 스크립트 작성(필수 키 일괄 `set`) |
| 컨테이너에서 Secret이 비어있음 | Secret Manager는 **호스트 사용자 프로필**에 존재 | 컨테이너에는 ENV/마운트로 주입 |
| 운영에서 Secret Manager 쓰려 함 | 설계 오용 | **Key Vault/Parameter Store/ENV**로 전환 |

---

## 13. 시크릿 스키마를 Options로 강타입/검증

```csharp
public sealed class SmtpOptions
{
    [Required] public string Host { get; set; } = "";
    [Range(1,65535)] public int Port { get; set; } = 587;
    [Required, EmailAddress] public string Username { get; set; } = "";
    [Required] public string Password { get; set; } = "";
    public bool EnableSsl { get; set; } = true;
}

builder.Services.AddOptions<SmtpOptions>()
    .Bind(builder.Configuration.GetSection("Smtp"))
    .ValidateDataAnnotations()
    .Validate(o => !string.Equals(o.Password, "password", StringComparison.OrdinalIgnoreCase), "Weak password.")
    .ValidateOnStart();
```

- 시크릿/설정 스키마를 **명확히** 하고 **Fail Fast**를 적용.

---

## 14. 예제: Razor Pages 로그인용 쿠키 키/Anti-CSRF/외부 API 키

```bash
dotnet user-secrets set "Auth:CookieName" ".MyApp.Auth"
dotnet user-secrets set "AntiForgery:CookieName" "__Host-AF"
dotnet user-secrets set "ThirdParty:OpenAI:ApiKey" "sk-xxxx"
```

```csharp
builder.Services.AddAuthentication("Cookie")
    .AddCookie("Cookie", o =>
    {
        o.Cookie.Name = builder.Configuration["Auth:CookieName"] ?? ".App.Auth";
        o.LoginPath = "/account/login";
        o.SlidingExpiration = true;
        o.Cookie.HttpOnly = true;
        o.Cookie.SecurePolicy = CookieSecurePolicy.Always;
        o.Cookie.SameSite = SameSiteMode.Strict;
    });

// 외부 API 클라이언트
builder.Services.AddHttpClient("openai", (sp, http) =>
{
    var key = builder.Configuration["ThirdParty:OpenAI:ApiKey"];
    http.BaseAddress = new Uri("https://api.openai.com/");
    http.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Bearer", key);
});
```

---

## 15. 실무 운영 전환(Production) 권장 패턴

- **개발**: Secret Manager  
- **스테이징/운영**:  
  - **환경 변수**(Docker/K8s/PAAS)  
  - **클라우드 시크릿 서비스**(Azure Key Vault, AWS Secrets Manager, GCP Secret Manager)
- **키 회전**: 플랫폼 기능 사용(버전/만료/자동 교체), 앱은 **`IOptionsMonitor`**로 즉시 반영.

예: Azure Key Vault 간단 등록(개요)

```csharp
builder.Configuration.AddAzureKeyVault(
    new Uri("https://<your-vault>.vault.azure.net/"),
    new DefaultAzureCredential());
```

> Key Vault의 Secret 이름이 설정 키가 되어 바인딩 가능. 운영에서는 **Secret Manager를 사용하지 않는다**.

---

## 16. 전체 샘플 — EF/JWT/SMTP/외부키를 Secret으로 관리

### 16.1 시크릿 넣기

```bash
dotnet user-secrets set "ConnectionStrings:DefaultConnection" "Server=.;Database=DevDb;User Id=dev;Password=dev;"
dotnet user-secrets set "JwtSettings:SecretKey" "dev-ultra-secret"
dotnet user-secrets set "Smtp:Host" "smtp.local"
dotnet user-secrets set "Smtp:Port" "587"
dotnet user-secrets set "Smtp:Username" "dev@example.com"
dotnet user-secrets set "Smtp:Password" "dev-app-pass"
dotnet user-secrets set "ThirdParty:Stripe:ApiKey" "sk_test_123"
```

### 16.2 `Program.cs`

```csharp
var builder = WebApplication.CreateBuilder(args);

// EF
builder.Services.AddDbContext<AppDbContext>(opt =>
    opt.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));

// JWT
builder.Services.AddAuthentication("Bearer")
    .AddJwtBearer("Bearer", o =>
    {
        var jwt = builder.Configuration.GetSection("JwtSettings");
        var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(jwt["SecretKey"]!));
        o.TokenValidationParameters = new()
        {
            ValidateIssuer = true, ValidateAudience = true, ValidateLifetime = true, ValidateIssuerSigningKey = true,
            ValidIssuer = jwt["Issuer"] ?? "myapi",
            ValidAudience = jwt["Audience"] ?? "myclient",
            IssuerSigningKey = key
        };
    });
builder.Services.AddAuthorization();

// SMTP
builder.Services.AddOptions<SmtpOptions>()
    .Bind(builder.Configuration.GetSection("Smtp"))
    .ValidateDataAnnotations()
    .ValidateOnStart();

// 외부 API
builder.Services.AddHttpClient("stripe", (sp, http) =>
{
    var key = builder.Configuration["ThirdParty:Stripe:ApiKey"];
    http.BaseAddress = new Uri("https://api.stripe.com/");
    http.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Bearer", key);
});

var app = builder.Build();

app.MapGet("/health", () => "OK");
app.Run();

public sealed class SmtpOptions
{
    [Required] public string Host { get; set; } = "";
    [Range(1,65535)] public int Port { get; set; } = 587;
    [Required, EmailAddress] public string Username { get; set; } = "";
    [Required] public string Password { get; set; } = "";
}
```

---

## 17. 보안 주의사항 — 핵심 요약

- Secret Manager는 **개발 전용**. 운영에서는 쓰지 않는다.
- Secret 파일은 **암호화되어 있지 않다**. 사용자 프로필 권한 보호에 의존.
- 민감 값을 **로그/응답으로 노출 금지**.
- 저장 키는 **네임스페이스 구조**로 체계화(`Feature:Module:Key`).
- 팀 온보딩 시 **필수 키 자동 세팅 스크립트** 제공(예: `scripts/dev-secrets.ps1`).

---

## 18. 체크리스트

- [ ] `.csproj`에 `UserSecretsId`가 있는가?
- [ ] `ASPNETCORE_ENVIRONMENT=Development`인가(혹은 `AddUserSecrets` 수동 추가)?
- [ ] `dotnet user-secrets list` 결과와 코드상의 키 스펠링이 정확히 일치하는가?
- [ ] ENV/명령줄이 Secret을 예상치 못하게 덮어쓰고 있지 않은가?
- [ ] 컨테이너/CI-CD/운영은 ENV/Key Vault로 전환했는가?

---

## 19. 요약

| 항목 | 내용 |
|------|------|
| 목적 | 개발 환경에서 비밀을 **소스 밖**(로컬 프로필)으로 분리 |
| 활성화 | `dotnet user-secrets init` → `.csproj`에 `UserSecretsId` |
| 명령어 | `set / list / remove / clear` (+ `--project`로 대상 지정) |
| 코드 사용 | `IConfiguration["Section:Key"]`/`IOptions<T>` 바인딩 |
| 우선순위 | 파일 → 환경별 파일 → **User Secrets**(Dev) → **ENV** → **명령줄** |
| 운영 전환 | **ENV / Key Vault / Secrets Manager** 등 사용 |
| 경고 | Secret 파일은 **암호화되지 않음**(개발 전용) |