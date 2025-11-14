---
layout: post
title: AspNet - IConfiguration
date: 2025-03-19 19:20:23 +0900
category: AspNet
---
# ASP.NET Core에서 구성 객체 `IConfiguration`

## `IConfiguration` 개요와 키 원리

- **계층형 키**: `:`(JSON/명령줄) 또는 `__`(환경 변수) 구분자 사용
- **소스 병합**: 여러 소스에서 읽은 값을 **우선순위**에 따라 병합(뒤에서 추가된 공급자가 우선)
- **DI 주입**: 어디서든 생성자에 `IConfiguration`을 주입하여 읽기

```csharp
public class IndexModel : PageModel
{
    private readonly IConfiguration _config;
    public IndexModel(IConfiguration config) => _config = config;

    public string SiteName { get; private set; } = "";
    public int MaxItems { get; private set; }

    public void OnGet()
    {
        SiteName = _config["AppSettings:SiteName"];
        MaxItems = _config.GetValue<int>("AppSettings:MaxItems", defaultValue: 10);
    }
}
```

**팁**
- 기본형은 `GetValue<T>(key, default?)`로 안전하게 파싱
- 배열/리스트 바인딩 시 `Section.Get<T>()`를 선호

---

## 기본 JSON 구성과 환경별 파일

### `appsettings.json`

```json
{
  "AppSettings": {
    "SiteName": "My ASP.NET App",
    "MaxItems": 10,
    "Features": {
      "EnableNewUI": true
    }
  },
  "ConnectionStrings": {
    "DefaultConnection": "Server=.;Database=MyDb;Trusted_Connection=True;"
  }
}
```

### 환경별 파일

`appsettings.Development.json`, `appsettings.Production.json` 등을 사용하며 **환경 변수 `ASPNETCORE_ENVIRONMENT`** 에 의해 선택된다.

```json
// appsettings.Development.json
{
  "AppSettings": {
    "SiteName": "My App (Dev)",
    "MaxItems": 3
  }
}
```

**Program.cs 기본 구성(템플릿 기본 제공)**
```csharp
var builder = WebApplication.CreateBuilder(args);
// builder.Configuration은 이미 다음 소스들로 구성됨:
// - appsettings.json
// - appsettings.{Environment}.json
// - User Secrets(Development)
// - Environment variables
// - Command-line
```

---

## 문자열 키 접근 vs 바인딩(타입 안전)

### Key 접근

```csharp
var name = config["AppSettings:SiteName"];
var max = config.GetValue<int>("AppSettings:MaxItems");
```

### POCO 바인딩(권장)

```csharp
public class AppSettings
{
    public string SiteName { get; set; } = "";
    public int MaxItems { get; set; }
    public FeaturesOptions Features { get; set; } = new();
}

public class FeaturesOptions
{
    public bool EnableNewUI { get; set; }
}
```

등록(Options 패턴):
```csharp
builder.Services.Configure<AppSettings>(
    builder.Configuration.GetSection("AppSettings"));
```

사용:
```csharp
using Microsoft.Extensions.Options;

public class HomeController : Controller
{
    private readonly AppSettings _settings;
    public HomeController(IOptions<AppSettings> options)
        => _settings = options.Value;

    public IActionResult Index()
    {
        ViewData["Title"] = _settings.SiteName;
        return View();
    }
}
```

**장점**
- 컴파일타임 검증
- 중첩/배열 구조를 정확히 매핑
- Options 생태계(IOptionsSnapshot/Monitor/Validation) 활용

---

## Options 고급: Snapshot/Monitor/Validation/Named

### `IOptions<T>` / `IOptionsSnapshot<T>` / `IOptionsMonitor<T>`

| 인터페이스 | 라이프사이클 | 용도 |
|---|---|---|
| `IOptions<T>` | 싱글톤 캐시(앱 시작 시 고정) | 간단, 변동 없음 |
| `IOptionsSnapshot<T>` | **Scoped**(요청마다 재구성) | 웹 요청마다 최신 구성 반영 |
| `IOptionsMonitor<T>` | **변경 알림 + 싱글톤** | 파일 변경 등 실시간 반영 + 콜백 |

파일 변경 감지(`reloadOnChange: true`) 예:
```csharp
builder.Configuration.AddJsonFile("appsettings.json", optional: false, reloadOnChange: true);

builder.Services.AddOptions<AppSettings>()
    .Bind(builder.Configuration.GetSection("AppSettings"))
    .ValidateDataAnnotations(); // 유효성 검사(아래 참고)

// IOptionsMonitor 사용
public class BannerService
{
    private AppSettings _current;
    public BannerService(IOptionsMonitor<AppSettings> monitor)
    {
        _current = monitor.CurrentValue;
        monitor.OnChange(newValue => _current = newValue);
    }
    public string CurrentSiteName() => _current.SiteName;
}
```

### 데이터 주석 기반 유효성 검사(Validation)

```csharp
public class AppSettings
{
    [Required, MinLength(3)]
    public string SiteName { get; set; } = "";
    [Range(1, 1000)]
    public int MaxItems { get; set; }
}
```

등록 시:
```csharp
builder.Services.AddOptions<AppSettings>()
    .Bind(builder.Configuration.GetSection("AppSettings"))
    .ValidateDataAnnotations()
    .Validate(o => o.MaxItems % 2 == 0, "MaxItems must be even.");
```

### Named Options(복수 설정 변형)

```csharp
builder.Services.AddOptions<CacheOptions>("Memory")
    .Bind(builder.Configuration.GetSection("Caches:Memory"));
builder.Services.AddOptions<CacheOptions>("Redis")
    .Bind(builder.Configuration.GetSection("Caches:Redis"));

public class CacheFactory
{
    private readonly IOptionsMonitor<CacheOptions> _monitor;
    public CacheFactory(IOptionsMonitor<CacheOptions> monitor) => _monitor = monitor;

    public CacheOptions Get(string name) => _monitor.Get(name);
}
```

### PostConfigure / ConfigureOptions 클래스로 모듈화

```csharp
builder.Services.PostConfigure<AppSettings>(opt =>
{
    if (string.IsNullOrWhiteSpace(opt.SiteName))
        opt.SiteName = "Fallback";
});

public class ConfigureMyFeature : IConfigureOptions<AppSettings>
{
    public void Configure(AppSettings options)
    {
        if (options.MaxItems < 1) options.MaxItems = 1;
    }
}
builder.Services.AddSingleton<IConfigureOptions<AppSettings>, ConfigureMyFeature>();
```

---

## 환경 변수/명령줄/시크릿(비밀) — 우선순위·키 규칙

### 환경 변수

- `__`(더블 언더스코어)로 **계층 표현**
```bash
# Linux/macOS

export AppSettings__SiteName="EnvApp"
export ConnectionStrings__DefaultConnection="Server=.;Database=EnvDb;Trusted_Connection=True;"

# Windows PowerShell

$env:AppSettings__SiteName="EnvApp"
```

코드 접근은 동일:
```csharp
var name = config["AppSettings:SiteName"]; // "EnvApp"
```

### 명령줄

```bash
dotnet run --AppSettings:SiteName="CLIName" --AppSettings:MaxItems=42
```
- 템플릿은 기본적으로 `AddCommandLine(args)` 포함 → **가장 높은 우선순위**

### User Secrets(개발용 비밀)

```bash
dotnet user-secrets init
dotnet user-secrets set "ApiKeys:Stripe" "sk_test_..."
```

```csharp
var stripe = config["ApiKeys:Stripe"];
```

**우선순위(일반적)**
명령줄 > 환경 변수 > `appsettings.{Environment}.json` > `appsettings.json`
(나중에 추가된 공급자가 앞선 값을 덮어쓴다)

---

## 배열/컬렉션 바인딩

JSON:
```json
{
  "AppSettings": {
    "Admins": [ "kim", "lee" ],
    "Endpoints": [
      { "Name": "Main", "Url": "https://example.com" },
      { "Name": "Backup", "Url": "https://backup.example.com" }
    ]
  }
}
```

POCO:
```csharp
public class AppSettings
{
    public List<string> Admins { get; set; } = new();
    public List<Endpoint> Endpoints { get; set; } = new();
}

public class Endpoint { public string Name { get; set; } = ""; public string Url { get; set; } = ""; }
```

등록/사용은 기존과 동일(`Configure<AppSettings>`).
**주의**: 환경 변수로 배열을 덮을 때는 인덱스로 지정
`AppSettings__Admins__0=kim`, `AppSettings__Admins__1=lee`

---

## 최소호스트/Minimal API에서의 구성 사용

```csharp
var builder = WebApplication.CreateBuilder(args);

// 읽기
var siteName = builder.Configuration["AppSettings:SiteName"];

// Options 바인딩
builder.Services.Configure<AppSettings>(builder.Configuration.GetSection("AppSettings"));

var app = builder.Build();

// 엔드포인트에서 바로 읽기
app.MapGet("/info", (IConfiguration cfg) =>
{
    var name = cfg["AppSettings:SiteName"];
    return Results.Ok(new { name });
});

app.Run();
```

---

## 실시간 재로딩(reloadOnChange)와 파일 감시

```csharp
builder.Configuration
    .AddJsonFile("appsettings.json", optional: false, reloadOnChange: true)
    .AddJsonFile($"appsettings.{builder.Environment.EnvironmentName}.json", optional: true, reloadOnChange: true);
```

- `IOptionsSnapshot<T>`: 요청 시점에 최신 스냅샷
- `IOptionsMonitor<T>`: 변경 이벤트 콜백 등록

**주의**
- 컨테이너/클라우드에서 **실제 파일 변경 감지가 어려울 수 있음**(ConfigMap 마운트 방식 따라 상이) → `IOptionsMonitor`에 적합한 공급자 활용(예: Azure App Configuration)

---

## 로깅과 구성의 연동

`appsettings.json`에서 로깅 레벨 제어:
```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.EntityFrameworkCore.Database.Command": "Warning"
    }
  }
}
```

사용:
```csharp
var logger = app.Services.GetRequiredService<ILogger<Program>>();
logger.LogInformation("SiteName={Site}", builder.Configuration["AppSettings:SiteName"]);
```

- **환경별 JSON**으로 개발/운영 로그 레벨 분리
- 런타임에 파일 수정 + `reloadOnChange` → 레벨 실시간 반영

---

## 사용자 지정 구성 공급자(파일/DB/외부 API)

### 간단한 INI/CSV 등 커스텀 Provider

```csharp
public sealed class SimpleTextConfigurationSource : IConfigurationSource
{
    public string Path { get; set; } = "";
    public bool Optional { get; set; }

    public IConfigurationProvider Build(IConfigurationBuilder builder)
        => new SimpleTextConfigurationProvider(this);
}

public sealed class SimpleTextConfigurationProvider : ConfigurationProvider
{
    private readonly SimpleTextConfigurationSource _source;
    public SimpleTextConfigurationProvider(SimpleTextConfigurationSource source) => _source = source;

    public override void Load()
    {
        if (!File.Exists(_source.Path))
        {
            if (_source.Optional) return;
            throw new FileNotFoundException(_source.Path);
        }

        var dict = new Dictionary<string, string?>();
        foreach (var line in File.ReadAllLines(_source.Path))
        {
            var parts = line.Split('=', 2);
            if (parts.Length == 2)
            {
                var key = parts[0].Trim();
                var val = parts[1].Trim();
                dict[key] = val;
            }
        }
        Data = dict;
    }
}

public static class SimpleTextConfigurationExtensions
{
    public static IConfigurationBuilder AddSimpleText(this IConfigurationBuilder b, string path, bool optional = false)
        => b.Add(new SimpleTextConfigurationSource { Path = path, Optional = optional });
}
```

등록:
```csharp
builder.Configuration.AddSimpleText("custom.config", optional: true);
```

### 클라우드 예시

- **Azure App Configuration** / **Azure Key Vault**
- AWS AppConfig / Parameter Store, GCP Secret Manager 등
→ 공식/커뮤니티 제공 패키지로 `IConfiguration`에 통합

---

## 컨테이너·쿠버네티스에서의 구성

### Docker 환경 변수 주입

```dockerfile
ENV AppSettings__SiteName="ContainerApp"
```

### Kubernetes ConfigMap/Secret

- ConfigMap을 파일 또는 환경 변수로 마운트
- Secret은 민감값(연결 문자열, API 키)에 사용

**주의**
- 파일 마운트 방식일 때 `reloadOnChange` 지원은 볼륨 드라이버/마운트 방식에 좌우
- 재로딩이 필수면 App Configuration/Consul 등 동적 공급자 고려

---

## 연결 문자열과 `IConfiguration`

관례적으로 `ConnectionStrings` 섹션:
```json
{
  "ConnectionStrings": {
    "Default": "Server=.;Database=MyDb;Trusted_Connection=True;"
  }
}
```

사용:
```csharp
var cs = builder.Configuration.GetConnectionString("Default");
builder.Services.AddDbContext<AppDbContext>(opt => opt.UseSqlServer(cs));
```

---

## 국제화/문화권과 바인딩 주의

- 숫자/날짜 파싱은 현재 문화권의 포맷 영향을 받는다.
- 배포 환경에서 문화권이 달라질 수 있으므로 숫자에는 `GetValue<int>`처럼 타입 지정 바인딩을 권장.
- 사용자 정의 타입 변환은 `TypeConverter` 또는 바인딩 확장을 고려.

---

## 구성 유효성 검증과 실패 전략

**부팅 시 치명 설정 검증**:
```csharp
builder.Services.AddOptions<AppSettings>()
    .Bind(builder.Configuration.GetSection("AppSettings"))
    .ValidateDataAnnotations()
    .Validate(o => !string.IsNullOrWhiteSpace(o.SiteName), "SiteName is required.")
    .ValidateOnStart(); // 앱 시작 시 유효성 실패 → 예외
```

운영 환경에서 **Fail Fast**로 잘못된 설정을 조기에 감지한다.

---

## 테스트에서의 구성 주입

### 단위 테스트(가짜 구성)

```csharp
var dict = new Dictionary<string, string?>
{
    ["AppSettings:SiteName"] = "TestApp",
    ["AppSettings:MaxItems"] = "5"
};

IConfiguration config = new ConfigurationBuilder()
    .AddInMemoryCollection(dict)
    .Build();
```

### 통합 테스트(WebApplicationFactory)

```csharp
public class MyFactory : WebApplicationFactory<Program>
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureAppConfiguration((ctx, cfg) =>
        {
            cfg.AddInMemoryCollection(new Dictionary<string, string?>
            {
                ["AppSettings:SiteName"] = "IT",
            });
        });
    }
}
```

---

## 보안 모범 사례

- 비밀은 `user-secrets`(개발) 또는 **보안 비밀 관리 서비스**(운영)에 저장
- 로그에 구성 값(특히 비밀/토큰) 출력 금지
- `EnableSensitiveDataLogging`은 개발에만
- 구성으로부터 가져온 **허용 목록/정책**은 유효성 검증으로 방어

---

## 실전 예제: Razor Pages + OptionsSnapshot + TempData 메시지

### `appsettings.json`

```json
{
  "AppSettings": {
    "SiteName": "Docs",
    "MaxItems": 20,
    "Admins": [ "kim", "lee" ]
  }
}
```

### 등록

```csharp
builder.Services.AddOptions<AppSettings>()
    .Bind(builder.Configuration.GetSection("AppSettings"))
    .ValidateDataAnnotations();
```

### PageModel

```csharp
using Microsoft.Extensions.Options;

public class SettingsModel : PageModel
{
    private readonly IOptionsSnapshot<AppSettings> _snap;
    public SettingsModel(IOptionsSnapshot<AppSettings> snap) => _snap = snap;

    [TempData] public string? Info { get; set; }
    public AppSettings Current => _snap.Value;

    public void OnGet() { }

    public IActionResult OnPostRefresh()
    {
        Info = $"Reloaded. MaxItems={_snap.Value.MaxItems}";
        return RedirectToPage();
    }
}
```

### View

```razor
@page
@model SettingsModel

<h2>@Model.Current.SiteName (@Model.Current.MaxItems)</h2>
@if (!string.IsNullOrEmpty(Model.Info))
{
    <div class="alert alert-info">@Model.Info</div>
}
<form method="post" asp-page-handler="Refresh">
    <button type="submit">리로드</button>
</form>
```

---

## 실전 예제: Named HttpClient + Named Options

### JSON

```json
{
  "HttpClients": {
    "GitHub": { "BaseAddress": "https://api.github.com", "TimeoutSeconds": 5 },
    "Weather": { "BaseAddress": "https://api.weather.com", "TimeoutSeconds": 10 }
  }
}
```

### Options/등록

```csharp
public class HttpClientOptions
{
    public string BaseAddress { get; set; } = "";
    public int TimeoutSeconds { get; set; } = 30;
}

builder.Services.AddOptions<HttpClientOptions>("GitHub")
    .Bind(builder.Configuration.GetSection("HttpClients:GitHub"))
    .ValidateDataAnnotations();

builder.Services.AddOptions<HttpClientOptions>("Weather")
    .Bind(builder.Configuration.GetSection("HttpClients:Weather"));

builder.Services.AddHttpClient("GitHub", (sp, client) =>
{
    var opt = sp.GetRequiredService<IOptionsMonitor<HttpClientOptions>>().Get("GitHub");
    client.BaseAddress = new Uri(opt.BaseAddress);
    client.Timeout = TimeSpan.FromSeconds(opt.TimeoutSeconds);
    client.DefaultRequestHeaders.UserAgent.ParseAdd("MyApp/1.0");
});

builder.Services.AddHttpClient("Weather", (sp, client) =>
{
    var opt = sp.GetRequiredService<IOptionsMonitor<HttpClientOptions>>().Get("Weather");
    client.BaseAddress = new Uri(opt.BaseAddress);
    client.Timeout = TimeSpan.FromSeconds(opt.TimeoutSeconds);
});
```

사용:
```csharp
public class ApiService
{
    private readonly IHttpClientFactory _factory;
    public ApiService(IHttpClientFactory factory) => _factory = factory;

    public async Task<string> GetGitHubAsync()
    {
        var cli = _factory.CreateClient("GitHub");
        return await cli.GetStringAsync("/rate_limit");
    }
}
```

---

## 구성 탐색/디버깅 유틸

### 트리 전개

```csharp
void Dump(IConfiguration config, string path = "")
{
    foreach (var child in config.GetChildren())
    {
        var key = string.IsNullOrEmpty(path) ? child.Key : $"{path}:{child.Key}";
        var value = child.Value;
        Console.WriteLine($"{key} = {value}");
        Dump(child, key);
    }
}
```

### 전체 키-값 열람

```csharp
foreach (var kv in builder.Configuration.AsEnumerable(makePathsRelative: false))
{
    Console.WriteLine($"{kv.Key} = {kv.Value}");
}
```

---

## 수학적 관점의 우선순위 모델(직관)

각 공급자(provider)를 $$ P_1, P_2, \dots, P_n $$라 하고,
이들이 같은 키 $$ k $$에 대해 값을 제공하면 **뒤에서 추가된** 공급자 $$ P_n $$의 값이 최종값이다.
즉, 병합 결과 $$ V(k) $$는
$$
V(k) = \mathrm{last}\big(\{\, P_i(k) \neq \varnothing \mid i = 1,\dots,n \,\}\big)
$$
와 같은 “후첨자 우선” 규칙으로 결정된다.
이 직관은 **구성 빌더에 공급자를 추가하는 순서**가 중요하다는 점을 강조한다.

---

## 체크리스트와 모범 사례

- [ ] 비밀은 절대 `appsettings.json`에 하드코딩하지 말 것(Secrets/Key Vault 등)
- [ ] `reloadOnChange` + `IOptionsMonitor`로 동적 구성을 설계
- [ ] 구성 유효성 검증(`ValidateDataAnnotations`, `ValidateOnStart`) 적용
- [ ] 구성 키는 일관된 네이밍(파스칼/케밥/스네이크 중 팀 규칙)
- [ ] 로깅 레벨/특정 기능 토글은 구성에서 제어
- [ ] 테스트에서 `InMemory` 구성으로 빠르게 주입/격리
- [ ] 컨테이너/쿠버네티스에서 환경 변수·ConfigMap/Secret로 외부화
- [ ] 사용자 지정 공급자로 레거시/외부 시스템과 안전하게 통합

---

## 요약

| 주제 | 핵심 포인트 |
|---|---|
| 키/계층 | `:` 또는 `__`로 중첩 키, `GetValue<T>`/`GetSection()` |
| 바인딩 | `Configure<T>(section)`, `IOptions/IOptionsSnapshot/IOptionsMonitor` |
| 실시간 | `reloadOnChange` + Monitor/OnChange 콜백 |
| 유효성 | DataAnnotations/커스텀 Validate/ValidateOnStart |
| 우선순위 | 명령줄 > 환경 변수 > 환경별 json > 기본 json |
| 보안 | user-secrets/Key Vault/Secret Manager, 로깅 주의 |
| 컨테이너 | 환경 변수·ConfigMap·Secret·동적 공급자 활용 |
| 테스트 | InMemory 구성/팩토리 패턴으로 격리 |

---

## 부록: 전체 샘플(Program.cs)

```csharp
var builder = WebApplication.CreateBuilder(args);

// 1) 구성 소스
builder.Configuration
    .AddJsonFile("appsettings.json", false, true)
    .AddJsonFile($"appsettings.{builder.Environment.EnvironmentName}.json", true, true)
    .AddEnvironmentVariables()
    .AddCommandLine(args);

// 2) Options
builder.Services.AddOptions<AppSettings>()
    .Bind(builder.Configuration.GetSection("AppSettings"))
    .ValidateDataAnnotations()
    .Validate(opts => opts.MaxItems > 0, "MaxItems must be positive.")
    .ValidateOnStart();

// 3) HTTP 클라이언트 + Named Options
builder.Services.AddOptions<HttpClientOptions>("GitHub")
    .Bind(builder.Configuration.GetSection("HttpClients:GitHub"));

builder.Services.AddHttpClient("GitHub", (sp, c) =>
{
    var o = sp.GetRequiredService<IOptionsMonitor<HttpClientOptions>>().Get("GitHub");
    c.BaseAddress = new Uri(o.BaseAddress);
    c.Timeout = TimeSpan.FromSeconds(o.TimeoutSeconds);
    c.DefaultRequestHeaders.UserAgent.ParseAdd("ConfigGuide/1.0");
});

var app = builder.Build();

app.MapGet("/config", (IConfiguration cfg) =>
{
    return Results.Ok(new
    {
        Site = cfg["AppSettings:SiteName"],
        Max = cfg.GetValue<int>("AppSettings:MaxItems")
    });
});

app.MapGet("/github", async (IHttpClientFactory f) =>
{
    var cli = f.CreateClient("GitHub");
    return Results.Text(await cli.GetStringAsync("/rate_limit"));
});

app.Run();

public class AppSettings
{
    [Required, MinLength(2)]
    public string SiteName { get; set; } = "";
    [Range(1, 1000)]
    public int MaxItems { get; set; }
}
public class HttpClientOptions
{
    [Required] public string BaseAddress { get; set; } = "";
    [Range(1, 60)] public int TimeoutSeconds { get; set; } = 10;
}
```

---

# 다음 추천 주제

- `IOptionsMonitor`로 다중 소스 핫리로드 패턴(Azure App Configuration/Consul)
- 사용자 지정 `IConfigurationProvider` 고급(폴더 감시, 암호화 값 자동 해독)
- 다국어/테넌트별 설정 분리(폴더/프리픽스/Named Options 전략)
- 연결 문자열 회전(Key Vault + Managed Identity)과 장애 대응
