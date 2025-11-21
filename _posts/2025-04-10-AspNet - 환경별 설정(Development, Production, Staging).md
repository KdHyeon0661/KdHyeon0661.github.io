---
layout: post
title: AspNet - 환경별 설정(Development, Production, Staging)
date: 2025-04-10 20:20:23 +0900
category: AspNet
---
# ASP.NET Core 환경별 설정 적용 완전 정리 (`Development`, `Staging`, `Production`)

## 개념과 표준 이름

ASP.NET Core는 실행 환경을 문자열로 구분한다. 대표 값은 다음과 같다.

| 환경 이름 | 용도 | 비고 |
|---|---|---|
| `Development` | 로컬 개발 | 디버깅, 상세 오류 노출, Swagger 기본 on |
| `Staging` | 사전검증 | 운영과 유사한 설정/스펙, 외부 공개 제한 |
| `Production` | 운영 | 보안/성능 옵션 최적화, 상세 오류 숨김 |

> 표준 3개를 권장하지만, `MyCompanyQa`처럼 **커스텀 이름**도 가능하다. 다만 파일명(`appsettings.{Name}.json`)과 코드 분기에서 같은 문자열을 사용해야 한다.

---

## 환경 설정 방법

### OS 환경 변수로 지정

| 플랫폼 | 명령 |
|---|---|
| Windows CMD | `set ASPNETCORE_ENVIRONMENT=Development` |
| PowerShell | `$env:ASPNETCORE_ENVIRONMENT="Production"` |
| Linux/macOS | `export ASPNETCORE_ENVIRONMENT=Staging` |

### `launchSettings.json` (개발 편의)

```json
{
  "profiles": {
    "MyApp": {
      "commandName": "Project",
      "dotnetRunMessages": true,
      "applicationUrl": "https://localhost:5001;http://localhost:5000",
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development"
      }
    }
  }
}
```

- VS/`dotnet run` 로컬 실행 시에만 적용된다.
- 실제 배포(컨테이너, 서비스)는 **OS 환경 변수/배포 플랫폼 설정**을 사용한다.

### 명령줄 인자로 지정

```bash
dotnet run --environment "Staging"
```

- 일회성으로 환경을 바꾸고 싶을 때 유용하다.

---

## 환경별 설정 파일 구성과 병합 규칙

ASP.NET Core는 **여러 구성 소스**를 순서대로 병합한다. 일반 템플릿의 대표 순서는 다음과 같다(나중에 로드되는 값이 앞의 값을 덮어쓴다).

1. `appsettings.json`
2. `appsettings.{Environment}.json`
3. User Secrets(Development일 때)
4. 환경 변수
5. 명령줄 인자

### JSON 병합의 핵심

- **객체**는 키 단위로 **오버라이드**된다.
- **배열**은 **병합이 아닌 교체**다(환경 파일의 배열이 전체를 덮는다).
  배열을 환경마다 일부만 바꾸고 싶다면 배열 대신 객체/키 방식을 고려하라.

### 코드로 명시 구성(선택)

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Configuration
    .SetBasePath(Directory.GetCurrentDirectory())
    .AddJsonFile("appsettings.json", optional: false, reloadOnChange: true)
    .AddJsonFile($"appsettings.{builder.Environment.EnvironmentName}.json", optional: true, reloadOnChange: true)
    .AddUserSecrets<Program>(optional: true)  // Dev에서만 자동 포함되지만 명시해도 됨
    .AddEnvironmentVariables()
    .AddCommandLine(args);
```

> `reloadOnChange: true`를 사용하면 파일 변경 시 설정이 다시 로드된다. 단, 바인딩 소비자는 `IOptionsSnapshot`/`IOptionsMonitor`로 받아야 실시간 반영이 가능하다(아래 참조).

---

## 코드에서 환경 판별하기

### `app`/`builder`/`IWebHostEnvironment`

```csharp
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    // 개발 전용
}
else if (app.Environment.IsStaging())
{
    // 스테이징 전용
}
else if (app.Environment.IsProduction())
{
    // 운영 전용
}
```

### 컨트롤러/서비스에서 판별

```csharp
public sealed class HomeController : Controller
{
    private readonly IWebHostEnvironment _env;
    public HomeController(IWebHostEnvironment env) => _env = env;

    public IActionResult Index()
    {
        ViewBag.Message = _env.IsDevelopment() ? "개발 환경" : _env.EnvironmentName;
        return View();
    }
}
```

---

## 환경별 미들웨어 파이프라인 분기(보안/성능)

```csharp
if (app.Environment.IsDevelopment())
{
    app.UseDeveloperExceptionPage();
    app.UseSwagger();
    app.UseSwaggerUI(c => c.SwaggerEndpoint("/swagger/v1/swagger.json", "My API v1"));
}
else
{
    app.UseExceptionHandler("/Error"); // 사용자 친화 오류 페이지
    app.UseHsts();                     // 엄격 전송 보안(HTTPS 강제)
    // 캐시/압축/보안 헤더 등 운영 최적화
}

app.UseHttpsRedirection();
app.UseStaticFiles();
app.UseRouting();
app.UseAuthentication();
app.UseAuthorization();

app.MapControllers();
app.MapRazorPages();
```

> Swagger를 운영에서 사용하려면 **보호**(인증/네트워크 제어) 없이 공개하지 말 것.

---

## — 서비스 구현 바꾸기

```csharp
if (builder.Environment.IsDevelopment())
{
    builder.Services.AddSingleton<IMailSender, DevConsoleMailSender>();
}
else
{
    builder.Services.AddSingleton<IMailSender, SmtpMailSender>();
}
```

또는 **조건부 팩토리**로 한 번에 처리:

```csharp
builder.Services.AddSingleton<IMailSender>(sp =>
{
    var env = sp.GetRequiredService<IWebHostEnvironment>();
    return env.IsDevelopment()
        ? new DevConsoleMailSender()
        : new SmtpMailSender(sp.GetRequiredService<IOptions<SmtpOptions>>().Value);
});
```

---

## 환경별 Options 패턴 — 설정 바인딩/검증

### 강타입 바인딩과 검증

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
    .ValidateOnStart();
```

### 실행 중 변경 반영

- **`IOptionsSnapshot<T>`**: 요청 범위마다 최신값(웹앱에서 주로 사용).
- **`IOptionsMonitor<T>`**: 구독 기반 콜백, 백그라운드에서도 즉시 반영.

```csharp
app.MapGet("/smtp", (IOptionsSnapshot<SmtpOptions> opt) => Results.Json(opt.Value));
```

---

## 환경별 로깅 레벨/프로바이더

`appsettings.{ENV}.json`로 레벨을 조정한다.

```json
// appsettings.Development.json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Information"
    }
  }
}
```

```json
// appsettings.Production.json
{
  "Logging": {
    "LogLevel": {
      "Default": "Warning",
      "Microsoft.AspNetCore": "Warning"
    }
  }
}
```

Serilog 등 외부 로거도 동일하게 환경별 분리 가능하다(싱크/레벨/필터 변경).

---

## 환경별 CORS/보안 헤더/Static 파일 캐시

### CORS

```csharp
if (builder.Environment.IsDevelopment())
{
    builder.Services.AddCors(o => o.AddPolicy("DevCors",
        p => p.WithOrigins("https://localhost:5173").AllowAnyHeader().AllowAnyMethod().AllowCredentials()));
}
else
{
    builder.Services.AddCors(o => o.AddPolicy("ProdCors",
        p => p.WithOrigins("https://app.example.com").AllowAnyHeader().AllowAnyMethod()));
}

var app = builder.Build();
app.UseCors(app.Environment.IsDevelopment() ? "DevCors" : "ProdCors");
```

### 정적 파일 캐시

```csharp
app.UseStaticFiles(new StaticFileOptions
{
    OnPrepareResponse = ctx =>
    {
        var headers = ctx.Context.Response.GetTypedHeaders();
        headers.CacheControl = app.Environment.IsDevelopment()
            ? new CacheControlHeaderValue { NoCache = true, NoStore = true, MustRevalidate = true }
            : new CacheControlHeaderValue { Public = true, MaxAge = TimeSpan.FromDays(30) };
    }
});
```

---

## 환경별 DB 연결/마이그레이션 안전장치

### 연결 문자열

- `appsettings.Development.json`: 로컬/도커 DB
- `appsettings.Staging.json`: 스테이징 DB
- `appsettings.Production.json`: 운영 DB는 파일 대신 **환경 변수/비밀 저장소**로

```csharp
builder.Services.AddDbContext<AppDbContext>(o =>
    o.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));
```

### 위험 작업 가드(운영에서 마이그레이션 금지 등)

```csharp
if (app.Environment.IsDevelopment())
{
    using var scope = app.Services.CreateScope();
    var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
    db.Database.Migrate();
}
else
{
    // 운영에서는 자동 마이그레이션 비활성화
}
```

---

## 환경별 Swagger/UI

```csharp
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

if (app.Environment.IsDevelopment() || app.Environment.EnvironmentName == "Staging")
{
    app.UseSwagger();
    app.UseSwaggerUI(c =>
    {
        c.SwaggerEndpoint("/swagger/v1/swagger.json", "My API v1");
        // 필요 시 Basic Auth/네트워크 제한 등 추가
    });
}
```

> 운영에서 Swagger를 열어야 하는 특별한 사유가 없다면 닫아두는 것이 안전하다(혹은 인증/사설망 보호).

---

## Secret Manager/환경 변수와의 협업

- Development: **Secret Manager**로 민감 값 분리
- Staging/Production: **환경 변수** 또는 **클라우드 비밀 저장소**(예: Azure Key Vault) 사용
- 우선순위 상 환경 변수/명령줄이 Secret을 덮어쓸 수 있으니 의도 확인

```bash
# 운영 서버 예시(리눅스)

export ConnectionStrings__DefaultConnection="Server=prod;..."
export JwtSettings__SecretKey="prod-ultra-secret"
```

---

## Docker/Kubernetes/CI-CD에서의 환경 지정

### Dockerfile/Compose

```dockerfile
ENV ASPNETCORE_URLS=http://+:8080
ENV ASPNETCORE_ENVIRONMENT=Production
```

```yaml
# docker-compose.yml

services:
  api:
    environment:
      ASPNETCORE_ENVIRONMENT: Staging
      ConnectionStrings__DefaultConnection: ${DB_CONN}
```

### Kubernetes

- `Deployment`의 `env`로 주입 또는 `Secret/ConfigMap`을 마운트
- 각 환경(네임스페이스)별로 값 분리

---

## 통합/단위 테스트에서의 환경 제어

### `WebApplicationFactory`로 환경 지정

```csharp
public class TestFactory : WebApplicationFactory<Program>
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.UseEnvironment("Testing");
        builder.ConfigureAppConfiguration((ctx, cfg) =>
        {
            cfg.AddJsonFile("appsettings.Testing.json", optional: true);
            cfg.AddEnvironmentVariables();
        });
    }
}
```

- 테스트 전용 파일(`appsettings.Testing.json`)과 인메모리 DB/스토어 사용을 권장한다.

---

## 고급: 환경별 라우트/엔드포인트 노출

```csharp
if (app.Environment.IsDevelopment())
{
    app.MapGet("/__env", () => Results.Json(new
    {
        app.Environment.EnvironmentName,
        IsDev = true
    })).AllowAnonymous();
}
// 운영 전에는 반드시 제거/보호
```

> 내부 점검용 엔드포인트는 절대 외부에 노출하지 말 것.

---

## 실전 템플릿 — 환경별 파일과 코드 스켈레톤

```
MyApp/
  appsettings.json
  appsettings.Development.json
  appsettings.Staging.json
  appsettings.Production.json
  Program.cs
```

`appsettings.json`(공통):

```json
{
  "Logging": { "LogLevel": { "Default": "Information" } },
  "AllowedHosts": "*",
  "Smtp": { "Host": "", "Port": 587, "Username": "", "Password": "", "EnableSsl": true }
}
```

`appsettings.Development.json`:

```json
{
  "Logging": { "LogLevel": { "Default": "Information", "Microsoft.AspNetCore": "Information" } },
  "Smtp": { "Host": "smtp.dev.local", "Username": "dev@example.com" }
}
```

`appsettings.Staging.json`:

```json
{
  "Logging": { "LogLevel": { "Default": "Information", "Microsoft.AspNetCore": "Warning" } },
  "Smtp": { "Host": "smtp.stg.example.com", "Username": "stg@example.com" }
}
```

`appsettings.Production.json`:

```json
{
  "Logging": { "LogLevel": { "Default": "Warning", "Microsoft.AspNetCore": "Warning" } },
  "Smtp": { "Host": "smtp.example.com" }
}
```

`Program.cs`(축약):

```csharp
var builder = WebApplication.CreateBuilder(args);

// Options
builder.Services.AddOptions<SmtpOptions>()
    .Bind(builder.Configuration.GetSection("Smtp"))
    .ValidateDataAnnotations()
    .ValidateOnStart();

// MVC/API
builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

// CORS
if (builder.Environment.IsDevelopment())
{
    builder.Services.AddCors(o => o.AddPolicy("Dev", p => p.AllowAnyOrigin().AllowAnyHeader().AllowAnyMethod()));
}
else
{
    builder.Services.AddCors(o => o.AddPolicy("Default", p => p.WithOrigins("https://app.example.com").AllowAnyHeader().AllowAnyMethod()));
}

var app = builder.Build();

// Pipeline
if (app.Environment.IsDevelopment())
{
    app.UseDeveloperExceptionPage();
    app.UseSwagger();
    app.UseSwaggerUI();
}
else
{
    app.UseExceptionHandler("/Error");
    app.UseHsts();
}

app.UseHttpsRedirection();
app.UseStaticFiles();
app.UseRouting();
app.UseCors(app.Environment.IsDevelopment() ? "Dev" : "Default");
app.UseAuthorization();
app.MapControllers();

app.Run();

public sealed class SmtpOptions
{
    [Required] public string Host { get; set; } = "";
    [Range(1,65535)] public int Port { get; set; } = 587;
    [Required, EmailAddress] public string Username { get; set; } = "";
    [Required] public string Password { get; set; } = "";
    public bool EnableSsl { get; set; } = true;
}
```

---

## 환경별 설정 체크리스트

- [ ] 배포 플랫폼에서 `ASPNETCORE_ENVIRONMENT`가 올바른가?
- [ ] `appsettings.{ENV}.json` 파일명이 정확한가(대소문자/맞춤법)?
- [ ] 배열이 예상치 못하게 덮여쓰이지 않는가(배열=교체)?
- [ ] Development에서만 Swagger/상세 오류가 노출되는가?
- [ ] 운영의 민감 값은 파일이 아닌 환경 변수/비밀 저장소를 쓰는가?
- [ ] 로깅 레벨/보안 헤더/캐시 정책이 운영에 적절한가?
- [ ] 마이그레이션/시드 등 위험 작업이 운영에서 자동으로 실행되지 않는가?

---

## 자주 하는 실수와 진단 팁

| 증상 | 원인 | 해결 |
|---|---|---|
| 설정이 안 먹는다 | 환경명이 다르다 | `EnvironmentName` 확인, 대소문자/띄어쓰기 점검 |
| Dev에서 Secret 값이 null | Secret Manager가 다른 프로젝트에 저장 | `dotnet user-secrets list --project <csproj>`로 확인 |
| 운영에서 Swagger 노출 | 개발용 분기를 깜빡 | `IsDevelopment` 조건 재확인, 리버스프록시/방화벽으로 차단 |
| CORS 에러 | 환경별 오리진 오타/도메인 누락 | 실제 호출 도메인과 설정 비교, HTTPS/포트 포함 |
| 배열 설정이 사라짐 | JSON 병합 특성(배열=교체) | 객체/키 구조로 재설계 또는 환경별 전체 배열 재정의 |

---

## 요약

| 항목 | 내용 |
|---|---|
| 환경 지정 | `ASPNETCORE_ENVIRONMENT`(OS/배포 플랫폼/명령줄) |
| 병합 규칙 | 공통 → 환경파일 → Secret(Dev) → 환경 변수 → 명령줄 (나중이 이김) |
| 코드 분기 | `IsDevelopment/IsStaging/IsProduction` |
| 실무 포인트 | Swagger/오류 페이지/로깅/캐시/CORS/DB/DI를 환경별로 분리 |
| 보안 | 운영 민감 값은 환경 변수/비밀 저장소 사용, Swagger 공개 금지 |
| 테스트/배포 | 테스트 전용 환경/파일, Docker/K8s/CI는 ENV로 일관 관리 |
