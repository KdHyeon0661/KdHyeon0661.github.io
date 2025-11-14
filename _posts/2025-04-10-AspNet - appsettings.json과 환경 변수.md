---
layout: post
title: AspNet - appsettings.json과 환경 변수
date: 2025-04-10 19:20:23 +0900
category: AspNet
---
# ASP.NET Core: `appsettings.json`과 환경 변수

## 설정 시스템 한눈에 보기 (우선순위와 병합)

ASP.NET Core는 **여러 소스의 키-값**을 **계층 키**(`:`)로 병합한다. **나중에 추가된 소스가 먼저 것들을 덮어쓴다**.

기본(일반적인 템플릿) 추가 순서(아래로 갈수록 우선순위 ↑):

1. `appsettings.json`
2. `appsettings.{Environment}.json` (예: Development, Staging, Production)
3. **User Secrets** (개발 전용; 개발 환경일 때 자동)
4. **환경 변수** (Environment Variables)
5. **명령줄 인수** (`--Key:SubKey=value`)
6. (선택) 사용자 지정 Provider (예: DB/원격 설정)

**중요 포인트**

- **키 구분자**: 코드/JSON에서는 `:`. 환경 변수에서는 `__`(언더스코어 2개).
- **병합(override)**: 동일 키면 **나중에 추가된 소스**가 **앞의 값을 덮어씀**.
- **대/소문자**: 키 비교는 기본적으로 **대/소문자 구분 안 함**.

---

## `appsettings.json` + 환경별 파일

### 기본/환경별 파일 구조

```json
// appsettings.json
{
  "Logging": { "LogLevel": { "Default": "Information", "Microsoft.AspNetCore": "Warning" } },
  "ConnectionStrings": {
    "DefaultConnection": "Server=.;Database=AppDb;Trusted_Connection=True;"
  },
  "MySettings": {
    "FeatureEnabled": true,
    "MaxItems": 100,
    "Nested": {
      "Endpoint": "https://api.local",
      "TimeoutSeconds": 5
    }
  }
}
```

```json
// appsettings.Development.json
{
  "MySettings": {
    "FeatureEnabled": false,
    "MaxItems": 10,
    "Nested": {
      "Endpoint": "https://dev.api.local"
    }
  }
}
```

```json
// appsettings.Production.json
{
  "MySettings": {
    "Nested": {
      "TimeoutSeconds": 2
    }
  }
}
```

- **병합 예시**: Development 환경이면 `MySettings.FeatureEnabled=false`, `MaxItems=10`, `Nested.Endpoint=https://dev.api.local`, `TimeoutSeconds=5(기본)`가 적용된다.
- Production에서는 `TimeoutSeconds=2`로 덮어쓴다.

### 환경 선택

- Windows CMD
  ```cmd
  set ASPNETCORE_ENVIRONMENT=Development
  ```
- PowerShell
  ```powershell
  $env:ASPNETCORE_ENVIRONMENT = "Production"
  ```
- Linux/macOS bash
  ```bash
  export ASPNETCORE_ENVIRONMENT=Staging
  ```

---

## `Program.cs` — 설정 파이프라인/서비스 구성

템플릿은 이미 환경별 파일/유저 시크릿/환경변수/명령줄까지 자동 추가한다.
필요 시 **명시적 구성**으로 제어할 수 있다.

```csharp
var builder = WebApplication.CreateBuilder(args);

// (선택) 구성 파이프라인을 직접 제어하고 싶다면:
// builder.Host.ConfigureAppConfiguration((ctx, config) =>
// {
//     config.Sources.Clear(); // 초기 소스 제거 (주의)
//     var env = ctx.HostingEnvironment;
//
//     config
//         .AddJsonFile("appsettings.json", optional: false, reloadOnChange: true)
//         .AddJsonFile($"appsettings.{env.EnvironmentName}.json", optional: true, reloadOnChange: true);
//
//     if (env.IsDevelopment())
//         config.AddUserSecrets<Program>(optional: true);
//
//     config
//         .AddEnvironmentVariables()       // ENV VAR
//         .AddCommandLine(args);           // CLI args
// });

builder.Services.AddOptions(); // Options 패턴 사용 시 권장

// Strongly-typed options 바인딩 + 유효성 검증
builder.Services.AddOptions<MySettings>()
    .Bind(builder.Configuration.GetSection("MySettings"))
    .ValidateDataAnnotations()
    .Validate(s => s.MaxItems >= 0, "MaxItems must be non-negative")
    .ValidateOnStart(); // 앱 시작 시 검증 실패 -> 바로 throw

var app = builder.Build();
app.MapGet("/", (IOptions<MySettings> opt) => Results.Json(opt.Value));
app.Run();

// 옵션 객체
public sealed class MySettings
{
    public bool FeatureEnabled { get; set; }
    [System.ComponentModel.DataAnnotations.Range(0, 10000)]
    public int MaxItems { get; set; }
    public NestedSettings Nested { get; set; } = new();
}
public sealed class NestedSettings
{
    [System.ComponentModel.DataAnnotations.Url]
    public string Endpoint { get; set; } = "";
    [System.ComponentModel.DataAnnotations.Range(1, 60)]
    public int TimeoutSeconds { get; set; }
}
```

> `reloadOnChange:true` → JSON 파일 변경 시 **자동 재로딩**(IOptionsSnapshot/IOptionsMonitor에서 반영).
> 환경 변수/명령줄은 기본적으로 **런타임 중 변경 감지 없음**.

---

## `IConfiguration`으로 직접 읽기 + 기본값

```csharp
public class MyService
{
    private readonly IConfiguration _cfg;
    public MyService(IConfiguration cfg) => _cfg = cfg;

    public (bool FeatureEnabled, int MaxItems) Snapshot()
    {
        var feature = _cfg.GetValue("MySettings:FeatureEnabled", defaultValue: false);
        var max = _cfg.GetValue<int?>("MySettings:MaxItems") ?? 50;
        return (feature, max);
    }
}
```

> `GetValue<T>(key, defaultValue)`를 활용해 **누락 시 기본값** 지정.

---

## Options 패턴 심화 — IOptions / IOptionsSnapshot / IOptionsMonitor

| 인터페이스 | 특징 | 용도 |
|------------|------|------|
| `IOptions<T>` | 앱 생애주기 동안 **고정된 스냅샷** | 싱글톤 서비스 등 불변 설정 |
| `IOptionsSnapshot<T>` | **요청마다** 새 바인딩(Scoped), JSON 변경 반영 | Web 앱에서 요청 단위 최신값 |
| `IOptionsMonitor<T>` | **구독/콜백** 가능, 변경 즉시 반영(Singleton도 OK) | 백그라운드 서비스/싱글톤 변동 반영 |

### Snapshot/Monitor 사용 예

```csharp
// Snapshot: 요청 스코프에서 최신값
app.MapGet("/snapshot", (IOptionsSnapshot<MySettings> opt) => Results.Json(opt.Value));

// Monitor: 변경 시 콜백
var monitor = app.Services.GetRequiredService<IOptionsMonitor<MySettings>>();
monitor.OnChange(newVal =>
{
    app.Logger.LogInformation("MySettings changed: MaxItems={Max}", newVal.MaxItems);
});
```

---

## 환경 변수를 통한 덮어쓰기 — 규칙/예제

- **키 구분자 `:` 대신 `__`** 사용.
- 부울/정수/배열 등 **문자열로 전달**되며 Binder가 적절히 변환함.

```bash
# Linux/macOS

export MySettings__FeatureEnabled=true
export MySettings__MaxItems=500
export MySettings__Nested__Endpoint="https://prod.api"
export ConnectionStrings__DefaultConnection="Server=prod-db;Database=App;User Id=app;Password=***"
```

```powershell
# PowerShell

$env:MySettings__FeatureEnabled = "false"
$env:MySettings__Nested__TimeoutSeconds = "3"
```

**배열 바인딩 예시** (`appsettings.json`)

```json
"Tags": [ "a", "b" ]
```

환경 변수로 추가/덮어쓰기:

```bash
export Tags__0=a
export Tags__1=b
export Tags__2=c
```

사전(Dictionary):

```json
"Map": { "kr": "Korea", "us": "United States" }
```

환경 변수:

```bash
export Map__kr="Korea"
export Map__us="United States"
```

---

## 명령줄 인자(가장 강력한 우선순위 중 하나)

```bash
dotnet run --MySettings:MaxItems=777 --MySettings:Nested:Endpoint=https://override
```

- CI/CD나 `launchSettings.json`에서 자주 활용.

---

## User Secrets (개발 전용 민감정보)

개발 환경에서만 로딩되며, 파일은 사용자 프로필에 저장(프로젝트와 분리).

```bash
dotnet user-secrets init
dotnet user-secrets set "MySettings:Nested:Endpoint" "https://dev.secret"
dotnet user-secrets set "ApiKeys:Stripe" "sk_test_123"
```

`Program.cs`에서 별도 호출 없이 **템플릿이 자동 추가**(Development일 때).
직접 추가하려면:

```csharp
builder.Configuration.AddUserSecrets<Program>();
```

---

## 로깅/레벨을 설정으로 제어

```json
// appsettings.json
"Logging": {
  "LogLevel": {
    "Default": "Information",
    "Microsoft.AspNetCore": "Warning",
    "MyApp.Namespace": "Debug"
  }
}
```

런타임 오버라이드(환경 변수):

```bash
export Logging__LogLevel__Default=Debug
export Logging__LogLevel__MyApp.Namespace=Trace
```

---

## 실시간 변경: `reloadOnChange`와 한계

- `AddJsonFile(..., reloadOnChange:true)` → 파일 변경 시 **파일 SystemWatcher**로 자동 반영.
- **환경 변수/명령줄**은 **재로딩 불가**(일반적으로 프로세스 재시작 필요).
- 바인딩 소비 측:
  - `IOptionsSnapshot<T>`: 다음 요청부터 반영.
  - `IOptionsMonitor<T>`: 즉시 콜백/값 반영.

---

## Azure/클라우드 연계

### Azure Key Vault (비밀 저장)

> 비밀번호/키는 **소스/파일에 두지 말고** 전용 비밀 저장소로.

(패키지/인증 설정 필요; 간단 개요)

```csharp
builder.Configuration
    .AddAzureKeyVault(new Uri("https://<your-vault>.vault.azure.net/"),
                      new DefaultAzureCredential());
```

- Key Vault의 **Secret 이름**을 설정 키로 매핑해 바인딩 가능.

### Azure App Configuration (원격 구성/피처 플래그)

```csharp
builder.Configuration.AddAzureAppConfiguration(options =>
{
    options.Connect("<connection-string>")
           .Select(KeyFilter.Any, LabelFilter.Null)
           .Select(KeyFilter.Any, "Production")
           .ConfigureRefresh(refresh =>
           {
               refresh.Register("Sentinel", refreshAll: true)
                      .SetCacheExpiration(TimeSpan.FromSeconds(30));
           });
});
```

---

## Docker/Kubernetes 배포 전략

### Docker

- `appsettings.json`은 이미지에 포함.
- **환경 변수**로 **민감 정보/환경별 차이**를 덮어씀.

```dockerfile
# Dockerfile 일부

ENV ASPNETCORE_URLS=http://+:8080
ENV MySettings__MaxItems=200
ENV ConnectionStrings__DefaultConnection="Server=db;Database=App;User Id=app;Password=***"
```

`docker run` 시:

```bash
docker run -e MySettings__MaxItems=300 -p 8080:8080 myapp:latest
```

### Kubernetes

- ConfigMap/Secret을 **환경 변수** 또는 **파일 마운트**로 주입.

ConfigMap → 환경 변수:

```yaml
env:
- name: MySettings__Nested__Endpoint
  valueFrom:
    configMapKeyRef:
      name: app-config
      key: endpoint
```

Secret → 환경 변수:

```yaml
env:
- name: ConnectionStrings__DefaultConnection
  valueFrom:
    secretKeyRef:
      name: db-secret
      key: conn
```

**권장**: 민감 정보는 **Secret** 사용, 일반 설정은 **ConfigMap**.

---

## 강타입 바인딩 팁/주의

- **Enum/TimeSpan**: Binder가 문자열을 변환하므로 `"Warning"`, `"00:00:05"` 형식 지원.
- **복합 타입/중첩 클래스/레코드** 지원.
- **필수 값 검증**: `ValidateDataAnnotations()`, `Validate(...)`, `ValidateOnStart()` 사용.
- **기본값**: 누락 시 C# 기본값/생성자 값 적용.

예: Enum/TimeSpan 포함 옵션

```csharp
public enum Mode { Basic, Pro }

public sealed class AdvancedSettings
{
    public Mode Mode { get; set; } = Mode.Basic;
    public TimeSpan CacheTtl { get; set; } = TimeSpan.FromSeconds(10);
}
```

`appsettings.json`

```json
"AdvancedSettings": {
  "Mode": "Pro",
  "CacheTtl": "00:00:05"
}
```

등록:

```csharp
builder.Services.AddOptions<AdvancedSettings>()
    .Bind(builder.Configuration.GetSection("AdvancedSettings"))
    .ValidateDataAnnotations()
    .ValidateOnStart();
```

---

## 컨트롤러/Minimal API에서의 사용

```csharp
app.MapGet("/cfg", (IConfiguration cfg) =>
{
    var enabled = cfg.GetValue<bool>("MySettings:FeatureEnabled");
    return Results.Json(new { FeatureEnabled = enabled });
});

app.MapGet("/opt", (IOptionsSnapshot<MySettings> opt) => Results.Json(opt.Value));
```

---

## 배포/운영 체크리스트

| 항목 | 포인트 |
|-----|--------|
| 민감 정보 | **절대** `appsettings.json`에 두지 말 것. Secret Manager(개발)/Key Vault(운영)/ENV |
| 우선순위 | 환경 변수/명령줄이 마지막에 덮어씀 — 의도치 않은 덮어쓰기 예방(접두사 사용 등) |
| 로그 레벨 | `Logging.LogLevel`를 ENV로 제어. 이슈 재현 시 일시적으로 `Debug/Trace` |
| reloadOnChange | 개발/테스트에 유용. 운영에서 파일 변경 주체/배포 전략을 명확히 |
| 유효성 검증 | `.ValidateOnStart()`로 잘못된 설정 **빠르게 Fail Fast** |
| 다중 환경 | `ASPNETCORE_ENVIRONMENT` 엄격히 관리(오타 방지: `Development`,`Staging`,`Production`) |
| Docker/K8s | ConfigMap/Secret로 분리, ENV 키는 `__` 구분 |

---

## 커스텀 설정 제공자 — DB에서 읽기 (샘플)

> 구성 저장소를 DB에 두고, 변경 신호(테이블 타임스탬프 등)로 **수동 리프레시**.

```csharp
public sealed class DbConfigurationSource : IConfigurationSource
{
    private readonly IServiceProvider _sp;
    public DbConfigurationSource(IServiceProvider sp) => _sp = sp;
    public IConfigurationProvider Build(IConfigurationBuilder builder) => new DbConfigurationProvider(_sp);
}

public sealed class DbConfigurationProvider : ConfigurationProvider
{
    private readonly IServiceProvider _sp;
    public DbConfigurationProvider(IServiceProvider sp) => _sp = sp;

    public override void Load()
    {
        using var scope = _sp.CreateScope();
        var repo = scope.ServiceProvider.GetRequiredService<IConfigRepository>();
        // 키-값 사전으로 로드
        Data = repo.LoadAll().ToDictionary(x => x.Key, x => x.Value, StringComparer.OrdinalIgnoreCase);
    }
}
```

등록:

```csharp
builder.Host.ConfigureAppConfiguration((ctx, config) =>
{
    var sp = builder.Services.BuildServiceProvider();
    config.Add(new DbConfigurationSource(sp)); // 우선순위: 호출 지점에 따라 달라짐(뒤에 추가될수록 우선)
});
```

> 프로덕션에서는 **리스너/폴링**으로 `Reload()` 호출 등 **갱신 전략** 필요.

---

## 통합 예시 — 한 번에 묶어서 보기

```csharp
var builder = WebApplication.CreateBuilder(args);

// 1) 기본 + 환경별 + reload
builder.Configuration
    .AddJsonFile("appsettings.json", optional: false, reloadOnChange: true)
    .AddJsonFile($"appsettings.{builder.Environment.EnvironmentName}.json", optional: true, reloadOnChange: true);

// 2) 개발에서만 User Secrets
if (builder.Environment.IsDevelopment())
    builder.Configuration.AddUserSecrets<Program>();

// 3) ENV + CLI
builder.Configuration
    .AddEnvironmentVariables()
    .AddCommandLine(args);

// 4) Options 바인딩/검증
builder.Services
    .AddOptions<MySettings>()
    .Bind(builder.Configuration.GetSection("MySettings"))
    .ValidateDataAnnotations()
    .Validate(s => s.MaxItems <= 10000, "MaxItems too large")
    .ValidateOnStart();

builder.Services.AddSingleton<MyService>();

var app = builder.Build();

app.MapGet("/info", (IOptionsMonitor<MySettings> opt) =>
{
    var v = opt.CurrentValue;
    return Results.Json(new { v.FeatureEnabled, v.MaxItems, v.Nested.Endpoint, v.Nested.TimeoutSeconds });
});

app.Run();
```

---

## 흔한 오류/함정 모음

- **환경 변수 구분자**: `:`가 아니라 **`__`**. (`MySettings__Nested__Endpoint`)
- **JSON 문법 오류**: **끝에 쉼표** 금지, 문자열 큰따옴표 필수.
- **불린/숫자/TimeSpan 파싱**: 환경 변수는 문자열이라서 값 타이핑을 정확히.
- **대소문자/오타**: `ASPNETCORE_ENVIRONMENT` 오타 → 환경별 파일 미로딩.
- **Reload 기대 오해**: ENV/CLI는 런타임 동적 갱신 안 됨(일반적으로 프로세스 재시작 필요).

---

## 보너스: 실전 시나리오 3종

### 기능 플래그(Feature Toggle)

```json
// appsettings.Production.json
"Features": { "NewCheckout": false }
```

```bash
# 긴급 오픈 (ENV)

export Features__NewCheckout=true
```

```csharp
public sealed class FeatureOptions { public bool NewCheckout { get; set; } }
builder.Services.Configure<FeatureOptions>(builder.Configuration.GetSection("Features"));
```

### API 키 안전 보관 (개발/운영 이원화)

- 개발: `dotnet user-secrets set "ApiKeys:Stripe" "sk_test_..."`
- 운영: Key Vault/ENV `ApiKeys__Stripe="sk_live_..."`

### 요청 단위 동적 반영(IOptionsSnapshot)

- `appsettings.json` 변경 → `IOptionsSnapshot<T>`가 **다음 요청부터** 최신값 제공.

---

## 요약

- **구성 우선순위**: 파일 → 환경별 파일 → User Secrets(개발) → **환경 변수** → **명령줄** (나중이 강함)
- **키 계층**: `:` / 환경 변수는 `__`
- **Options 패턴**: `IOptions`(고정), `Snapshot`(요청별 최신), `Monitor`(즉시 반영+콜백)
- **검증**: `ValidateDataAnnotations`, `Validate`, `ValidateOnStart`로 **Fail Fast**
- **보안**: 비밀은 파일에 두지 말고 **Secrets/Key Vault/ENV**로
- **배포**: Docker/K8s는 ENV/Secret/ConfigMap로 설정 주입
- **변경 감지**: JSON은 `reloadOnChange`, ENV/CLI는 일반적으로 재시작 필요
