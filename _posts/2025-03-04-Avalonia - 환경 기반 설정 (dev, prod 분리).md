---
layout: post
title: Avalonia - 환경 기반 설정 (dev, prod 분리)
date: 2025-03-04 20:20:23 +0900
category: Avalonia
---
# 6. 환경 기반 설정(dev/prod 분리): Avalonia에서 `appsettings.*.json` 로딩·DI·옵션 패턴·재로딩까지

## 1) 디렉터리·파일 구성

```
📁 Project Root
├── appsettings.json                 # 공통 기본값(필수)
├── appsettings.dev.json             # 개발 환경
├── appsettings.prod.json            # 운영 환경
├── appsettings.local.json           # 개인 로컬 오버라이드(VC 제외 권장)
├── src/
│   └── MyApp/
│       ├── Program.cs               # 설정/DI 초기화
│       ├── App.xaml.cs
│       ├── Options/
│       │   ├── ApiOptions.cs
│       │   ├── FeatureFlags.cs
│       │   └── UiOptions.cs
│       ├── Services/
│       │   ├── IApiClient.cs
│       │   └── ApiClient.cs
│       └── ViewModels/
│           └── HomeViewModel.cs
└── MyApp.csproj
```

> 팁  
> - `appsettings.local.json`은 개인 개발자별 override 용. **소스관리 제외**(.gitignore).  
> - 실제 배포 아티팩트에 설정 파일을 포함하려면 `.csproj`의 `CopyToOutputDirectory`를 활용(아래 9장 참조).

---

## 2) 환경 변수와 우선순위

`ConfigurationBuilder`는 **등록 순서가 중요**하다. 뒤에 오는 소스가 앞선 값을 **덮어쓴다**.

우선순위(권장 등록 순서):
1. `appsettings.json` (공통)
2. `appsettings.{env}.json` (환경별)
3. `appsettings.local.json` (개인 로컬)
4. `EnvironmentVariables` (CI/비밀 주입)
5. `CommandLine` (실행 시 오버라이드)

환경 선택:
- `DOTNET_ENVIRONMENT=dev` → `appsettings.dev.json` 로딩
- 미설정 시 기본값을 `prod`로 가정하면 보수적이며 안전

---

## 3) Program.cs — 구성 로딩의 표준 패턴

```csharp
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Logging;
using System.Reflection;

var env = Environment.GetEnvironmentVariable("DOTNET_ENVIRONMENT") ?? "prod";

var configuration = new ConfigurationBuilder()
    .SetBasePath(AppContext.BaseDirectory)
    .AddJsonFile("appsettings.json", optional: false, reloadOnChange: true)
    .AddJsonFile($"appsettings.{env}.json", optional: true, reloadOnChange: true)
    .AddJsonFile("appsettings.local.json", optional: true, reloadOnChange: true)
    .AddEnvironmentVariables() // e.g. Api__BaseUrl, Logging__LogLevel__Default
    .AddCommandLine(args)      // e.g. --Api:BaseUrl=https://override
    .Build();

// DI
var services = new ServiceCollection();

// Options pattern 바인딩
services.Configure<ApiOptions>(configuration.GetSection("Api"));
services.Configure<FeatureFlags>(configuration.GetSection("Features"));
services.Configure<UiOptions>(configuration.GetSection("Ui"));

// 로깅(기본 콘솔)
services.AddLogging(b =>
{
    b.AddConfiguration(configuration.GetSection("Logging"));
    b.AddSimpleConsole();
});

// HttpClient + 베이스 주소를 Options에서 주입
services.AddHttpClient<IApiClient, ApiClient>((sp, http) =>
{
    var opts = sp.GetRequiredService<Microsoft.Extensions.Options.IOptionsMonitor<ApiOptions>>().CurrentValue;
    http.BaseAddress = new Uri(opts.BaseUrl);
    // 타임아웃, 기본헤더 등도 Options 반영 가능
});

// ViewModel 등
services.AddTransient<ViewModels.HomeViewModel>();

var provider = services.BuildServiceProvider();

// 이후 Avalonia AppBuilder 초기화 시 provider를 전달하거나 정적 보관
// (예: App.Services = provider;)
```

> `reloadOnChange: true`를 사용하면 설정 파일 변경 시 `IOptionsMonitor<T>` 구독자를 통해 **런타임 갱신**이 가능하다(아래 6장).

---

## 4) Options 클래스(강타입)와 스키마

### 4.1 API 옵션

```csharp
namespace MyApp.Options;

public sealed class ApiOptions
{
    public string BaseUrl { get; set; } = "https://api.example.com";
    public int TimeoutSeconds { get; set; } = 30;
    public bool UseCompression { get; set; } = true;
}
```

### 4.2 기능 플래그

```csharp
namespace MyApp.Options;

public sealed class FeatureFlags
{
    public bool EnableNewDashboard { get; set; } = false;
    public bool DevToolsVisible { get; set; } = false;
    public bool UseMockData { get; set; } = false;
}
```

### 4.3 UI 옵션(테마/로캘 등)

```csharp
namespace MyApp.Options;

public sealed class UiOptions
{
    public string Theme { get; set; } = "Light";     // Light/Dark/System
    public string Language { get; set; } = "ko";     // ko/en/...
    public double DefaultFontSize { get; set; } = 13;
}
```

### 4.4 appsettings 예시

```json
// appsettings.json
{
  "Api": {
    "BaseUrl": "https://api.example.com",
    "TimeoutSeconds": 30,
    "UseCompression": true
  },
  "Features": {
    "EnableNewDashboard": false,
    "DevToolsVisible": false,
    "UseMockData": false
  },
  "Ui": {
    "Theme": "Light",
    "Language": "ko",
    "DefaultFontSize": 13
  },
  "Logging": {
    "LogLevel": { "Default": "Information" }
  }
}
```

```json
// appsettings.dev.json
{
  "Api": {
    "BaseUrl": "https://api-dev.example.com",
    "TimeoutSeconds": 10,
    "UseCompression": false
  },
  "Features": {
    "EnableNewDashboard": true,
    "DevToolsVisible": true,
    "UseMockData": true
  },
  "Logging": {
    "LogLevel": { "Default": "Debug" }
  }
}
```

```json
// appsettings.prod.json
{
  "Api": {
    "BaseUrl": "https://api.example.com",
    "TimeoutSeconds": 30,
    "UseCompression": true
  },
  "Features": {
    "EnableNewDashboard": false,
    "DevToolsVisible": false,
    "UseMockData": false
  },
  "Logging": {
    "LogLevel": { "Default": "Warning" }
  }
}
```

---

## 5) HttpClient + Options 연동

```csharp
public interface IApiClient
{
    Task<string> GetStatusAsync(CancellationToken ct = default);
}

public sealed class ApiClient : IApiClient
{
    private readonly HttpClient _http;
    private readonly Microsoft.Extensions.Options.IOptionsMonitor<ApiOptions> _api;

    public ApiClient(HttpClient http, Microsoft.Extensions.Options.IOptionsMonitor<ApiOptions> api)
    {
        _http = http;
        _api = api;
        ApplyOptions(_api.CurrentValue);
        _api.OnChange(ApplyOptions); // 설정 파일 변경 시 즉시 반영
    }

    private void ApplyOptions(ApiOptions o)
    {
        if (_http.BaseAddress is null || _http.BaseAddress.ToString() != o.BaseUrl)
            _http.BaseAddress = new Uri(o.BaseUrl);

        _http.Timeout = TimeSpan.FromSeconds(o.TimeoutSeconds);
        _http.DefaultRequestHeaders.AcceptEncoding.Clear();
        if (o.UseCompression)
            _http.DefaultRequestHeaders.AcceptEncoding.ParseAdd("gzip, deflate, br");
    }

    public async Task<string> GetStatusAsync(CancellationToken ct = default)
    {
        using var res = await _http.GetAsync("/status", ct);
        res.EnsureSuccessStatusCode();
        return await res.Content.ReadAsStringAsync(ct);
    }
}
```

> 핵심: `IOptionsMonitor<T>.OnChange`로 **재시작 없이** 설정값 반영.

---

## 6) ViewModel에서 `IOptionsMonitor`로 실시간 반영

```csharp
public sealed class HomeViewModel : ReactiveUI.ReactiveObject
{
    private readonly Microsoft.Extensions.Options.IOptionsMonitor<UiOptions> _ui;
    private string _theme;
    private string _language;
    private double _fontSize;

    public string Theme { get => _theme; private set => this.RaiseAndSetIfChanged(ref _theme, value); }
    public string Language { get => _language; private set => this.RaiseAndSetIfChanged(ref _language, value); }
    public double FontSize { get => _fontSize; private set => this.RaiseAndSetIfChanged(ref _fontSize, value); }

    public HomeViewModel(Microsoft.Extensions.Options.IOptionsMonitor<UiOptions> ui)
    {
        _ui = ui;
        ApplyUi(ui.CurrentValue);
        _ui.OnChange(o => Avalonia.Threading.Dispatcher.UIThread.Post(() => ApplyUi(o)));
    }

    private void ApplyUi(UiOptions o)
    {
        Theme = o.Theme;
        Language = o.Language;
        FontSize = o.DefaultFontSize;

        // 여기서 Avalonia Theme/RequestedTheme 교체, 리소스 딕셔너리 스왑 등 적용 가능
        // LocalizationService.SetCulture(Language) 등
    }
}
```

> `reloadOnChange: true`일 때, `appsettings.*.json` 파일 저장 → UI가 **즉시 반응**.

---

## 7) 설정 유효성 검증(스타트업 Fail Fast)

실전에서는 잘못된 설정(예: 빈 URL, 음수 타임아웃)을 **초기 구동에서 차단**해야 한다.

```csharp
using Microsoft.Extensions.Options;

public sealed class ApiOptionsValidator : IValidateOptions<ApiOptions>
{
    public ValidateOptionsResult Validate(string name, ApiOptions options)
    {
        if (string.IsNullOrWhiteSpace(options.BaseUrl)) 
            return ValidateOptionsResult.Fail("Api:BaseUrl is required.");

        if (!Uri.IsWellFormedUriString(options.BaseUrl, UriKind.Absolute))
            return ValidateOptionsResult.Fail("Api:BaseUrl must be an absolute URI.");

        if (options.TimeoutSeconds <= 0 || options.TimeoutSeconds > 600)
            return ValidateOptionsResult.Fail("Api:TimeoutSeconds must be 1..600.");

        return ValidateOptionsResult.Success;
    }
}

// DI 등록
services.AddOptions<ApiOptions>()
        .Bind(configuration.GetSection("Api"))
        .ValidateOnStart()                     // 앱 시작 시 검증
        .Services.AddSingleton<IValidateOptions<ApiOptions>, ApiOptionsValidator>();
```

---

## 8) 비밀/민감정보 처리(토큰·키)

**금지**: `appsettings*.json`에 평문 비밀 저장.  
권장 대안:

| OS | 권장 저장소 |
|---|---|
| Windows | DPAPI(ProtectedData), Credential Manager |
| macOS | Keychain |
| Linux | Secret Service(KWallet/GNOME Keyring), `keyring` 라이브러리 |

### 예: DPAPI로 민감정보 암복호(Windows)

```csharp
using System.Security.Cryptography;
using System.Text;

public static class SecretStore
{
    public static string Protect(string plain)
    {
        var bytes = Encoding.UTF8.GetBytes(plain);
        var prot = ProtectedData.Protect(bytes, null, DataProtectionScope.CurrentUser);
        return Convert.ToBase64String(prot);
    }

    public static string Unprotect(string cipher)
    {
        var data = Convert.FromBase64String(cipher);
        var unprot = ProtectedData.Unprotect(data, null, DataProtectionScope.CurrentUser);
        return Encoding.UTF8.GetString(unprot);
    }
}
```

> 전략: **구성(옵션)**에는 민감정보의 **Key 이름/슬롯**만 두고, 실제 값은 OS 보안 저장소에서 읽어 DI를 통해 주입.

---

## 9) 배포 시 설정 파일 포함/변형

### 9.1 .csproj에서 출력에 포함

```xml
<ItemGroup>
  <None Include="appsettings.json" CopyToOutputDirectory="Always" />
  <None Include="appsettings.dev.json" CopyToOutputDirectory="PreserveNewest" />
  <None Include="appsettings.prod.json" CopyToOutputDirectory="PreserveNewest" />
  <None Include="appsettings.local.json" CopyToOutputDirectory="Never" />
</ItemGroup>
```

- 운영 빌드: `appsettings.prod.json`만 포함하거나, CI에서 **환경에 맞는 파일만** 복사.
- `local`은 절대 포함 금지.

### 9.2 CI에서 환경 주입(우선순위 상단인 환경 변수 활용)

GitHub Actions 예:

```yaml
- name: Publish
  run: dotnet publish -c Release -r win-x64 --self-contained true -o out

- name: Set env override
  env:
    Api__BaseUrl: https://api-prod.example.com
  run: |
    # 실행 시 환경이 이 값을 덮어쓰게 설계되어 있으면 별도 파일 교체 불필요
```

---

## 10) 플랫폼별 설정 경로(사용자 오버라이드 파일)

사용자별 오버라이드 파일을 OS 표준 경로에 저장/로딩하면, 배포 파일은 불변으로 유지하고 **러런타임 사용자 설정만 덮어쓰기** 가능.

```csharp
static string GetUserConfigPath()
{
    if (OperatingSystem.IsWindows())
        return Path.Combine(Environment.GetFolderPath(Environment.SpecialFolder.ApplicationData), "MyApp", "usersettings.json");
    if (OperatingSystem.IsMacOS())
        return Path.Combine(Environment.GetFolderPath(Environment.SpecialFolder.Personal), "Library", "Application Support", "MyApp", "usersettings.json");
    // Linux
    var dir = Environment.GetEnvironmentVariable("XDG_CONFIG_HOME")
              ?? Path.Combine(Environment.GetFolderPath(Environment.SpecialFolder.Personal), ".config");
    return Path.Combine(dir, "MyApp", "usersettings.json");
}

// Program.cs 빌더에 추가
var userConfigPath = GetUserConfigPath();
var cfgBuilder = new ConfigurationBuilder()
    .SetBasePath(AppContext.BaseDirectory)
    .AddJsonFile("appsettings.json", false, true)
    .AddJsonFile($"appsettings.{env}.json", true, true);

if (File.Exists(userConfigPath))
    cfgBuilder.AddJsonFile(userConfigPath, optional: true, reloadOnChange: true);

cfgBuilder.AddEnvironmentVariables().AddCommandLine(args);

var configuration = cfgBuilder.Build();
```

---

## 11) 기능 플래그(Feature Flags)로 UX 제어(A/B 실험)

```csharp
public sealed class FeatureGate
{
    private readonly Microsoft.Extensions.Options.IOptionsMonitor<FeatureFlags> _flags;
    public FeatureGate(Microsoft.Extensions.Options.IOptionsMonitor<FeatureFlags> flags) => _flags = flags;

    public bool IsNewDashboardEnabled => _flags.CurrentValue.EnableNewDashboard;

    // 뷰모델/뷰에서는 이 게이트만 의존 → 설정 변경 시 즉시 반영
}
```

View XAML:

```xml
<!-- DataTriggers/IsVisible 바인딩으로 조건부 렌더링 -->
<StackPanel>
  <views:NewDashboardView IsVisible="{Binding FeatureGate.IsNewDashboardEnabled}" />
  <views:LegacyDashboardView IsVisible="{Binding FeatureGate.IsNewDashboardEnabled, Converter={StaticResource InverseBool}}" />
</StackPanel>
```

---

## 12) 로깅 체계: 환경별 레벨·싱크

```json
// dev
{
  "Logging": {
    "LogLevel": {
      "Default": "Debug",
      "Microsoft": "Warning"
    }
  }
}
```

```json
// prod
{
  "Logging": {
    "LogLevel": {
      "Default": "Warning"
    }
  }
}
```

Serilog를 쓰는 경우:

```csharp
// Program.cs
using Serilog;
Log.Logger = new LoggerConfiguration()
   .ReadFrom.Configuration(configuration)
   .Enrich.FromLogContext()
   .CreateLogger();

services.AddLogging(lb => lb.ClearProviders().AddSerilog());
```

`serilog` 섹션을 `appsettings.*.json`에 분리 관리.

---

## 13) 커맨드라인/환경변수 오버라이드 실전

- 환경변수: `Api__BaseUrl=https://stg.example.com`  
- CLI: `--Api:TimeoutSeconds=5 --Features:UseMockData=true`

> 네임스페이스 구분은 `:`이며, 환경변수에서는 `__`(더블 언더스코어) 사용.

---

## 14) 설정 스키마 버전 관리·마이그레이션

배포 후 시간이 지나면 설정 키가 바뀐다. 앱 시작 시 버전을 점검해 **자동 마이그레이션**을 수행하면 사용자가 손대지 않아도 안정적으로 전환된다.

```csharp
public sealed class SettingsMigrator
{
    public void MigrateIfNeeded(IConfiguration cfg, string userConfigPath)
    {
        var version = cfg["SchemaVersion"] ?? "1";
        if (version == "1")
        {
            // 예: Ui:ThemeName -> Ui:Theme 로 키 이동
            // userConfigPath JSON을 로드→변환→백업 후 저장
        }
    }
}
```

---

## 15) 테스트 전략

- **Options 바인딩 단위 테스트**: `ConfigurationBuilder().AddInMemoryCollection()`으로 가짜 설정 주입 → `services.Configure<T>()` 바인딩 검증  
- **IOptionsMonitor 변경 이벤트**: `reloadOnChange` 대신, 테스트에선 `IOptionsMonitorCache<T>`를 써서 `TryAdd/Reset`으로 변경 시뮬레이션  
- **ApiClient**: `HttpMessageHandler`를 mock으로 대체 → `HttpClient` 주입 테스트

```csharp
[Fact]
public void ApiOptions_Binds_From_Config()
{
    var dict = new Dictionary<string, string?>
    {
        ["Api:BaseUrl"] = "https://dev.example.com",
        ["Api:TimeoutSeconds"] = "5",
        ["Api:UseCompression"] = "false"
    };
    var cfg = new ConfigurationBuilder().AddInMemoryCollection(dict).Build();

    var services = new ServiceCollection();
    services.Configure<ApiOptions>(cfg.GetSection("Api"));
    var sp = services.BuildServiceProvider();
    var opts = sp.GetRequiredService<Microsoft.Extensions.Options.IOptions<ApiOptions>>().Value;

    Assert.Equal("https://dev.example.com", opts.BaseUrl);
    Assert.Equal(5, opts.TimeoutSeconds);
    Assert.False(opts.UseCompression);
}
```

---

## 16) 예시: dev/prod 분기에 따른 UI·API 자동 전환

- dev:  
  - API = `https://api-dev.example.com`  
  - 로깅 = Debug  
  - FeatureFlags.UseMockData = true → 샘플 카드 표시  
- prod:  
  - API = `https://api.example.com`  
  - 로깅 = Warning  
  - FeatureFlags.UseMockData = false → 실제 API만 사용

결과: 빌드/실행 환경만 바꿔도 앱은 **자연스럽게 다른 동작**을 한다.

---

## 17) 빌드/배포 파이프라인에서 환경 주입

### GitHub Actions 매트릭스 예시(발췌)

```yaml
- name: Set environment
  run: echo "DOTNET_ENVIRONMENT=prod" >> $GITHUB_ENV

- name: Publish
  run: dotnet publish -c Release -r win-x64 --self-contained true -o out
```

> 필요 시 **환경변수**로 민감정보 주입(토큰/키) → 런타임에 OS 보안 저장소와 결합해 사용.

---

## 18) 성능·안정성 고려사항

- `reloadOnChange`는 파일 시스템 감시를 사용. 빈번한 저장(에디터 자동 저장) 시 변경 이벤트가 잦을 수 있으므로, **구독처에서 속도 완충(Throttle)** 또는 **핵심만 반영**.  
- XAML Theme/리소스 스왑은 **UI 스레드**에서 수행.  
- `PublishTrimmed=true` 사용 시 리플렉션/리소스 키가 트리밍 대상인지 확인(링커 지시파일 사용).

---

## 19) 요약 표

| 주제 | 핵심 요점 |
|---|---|
| 로딩 순서 | 공통 → 환경 → 로컬 → 환경변수 → CLI(뒤가 앞 덮음) |
| 환경 선택 | `DOTNET_ENVIRONMENT`(dev/prod) |
| DI 바인딩 | `services.Configure<T>(section)` + `IOptionsMonitor<T>` |
| 즉시 반영 | `reloadOnChange` + `OnChange` |
| 검증 | `IValidateOptions<T>` + `.ValidateOnStart()` |
| 비밀 | OS 보안 저장소/환경변수 활용, 설정 파일 평문 금지 |
| 배포 포함 | `.csproj` CopyToOutputDirectory 및 CI에서 선택 포함 |
| 테스트 | InMemoryConfiguration, HttpMessageHandler mock |
| UX | 기능 플래그/테마/언어 런타임 전환으로 QA/실험 가속 |

---

## 20) 부록: 전체 미니 샘플

### `appsettings.dev.json`

```json
{
  "Api": { "BaseUrl": "https://api-dev.example.com", "TimeoutSeconds": 10 },
  "Features": { "EnableNewDashboard": true, "UseMockData": true },
  "Ui": { "Theme": "Dark", "Language": "ko", "DefaultFontSize": 14 },
  "Logging": { "LogLevel": { "Default": "Debug" } }
}
```

### `Options/ApiOptions.cs`

```csharp
namespace MyApp.Options
{
    public sealed class ApiOptions
    {
        public string BaseUrl { get; set; } = "";
        public int TimeoutSeconds { get; set; } = 30;
        public bool UseCompression { get; set; } = true;
    }
}
```

### `Services/ApiClient.cs`(재로딩 반영)

```csharp
public sealed class ApiClient : IApiClient
{
    private readonly HttpClient _http;
    private readonly Microsoft.Extensions.Options.IOptionsMonitor<ApiOptions> _api;

    public ApiClient(HttpClient http, Microsoft.Extensions.Options.IOptionsMonitor<ApiOptions> api)
    {
        _http = http; _api = api;
        Apply(_api.CurrentValue);
        _api.OnChange(Apply);
    }

    private void Apply(ApiOptions o)
    {
        _http.BaseAddress = new Uri(o.BaseUrl);
        _http.Timeout = TimeSpan.FromSeconds(o.TimeoutSeconds);
    }

    public async Task<string> GetStatusAsync(CancellationToken ct = default)
    {
        using var res = await _http.GetAsync("/status", ct);
        res.EnsureSuccessStatusCode();
        return await res.Content.ReadAsStringAsync(ct);
    }
}
```

### `ViewModels/HomeViewModel.cs`

```csharp
public sealed class HomeViewModel : ReactiveUI.ReactiveObject
{
    private readonly IApiClient _api;
    private readonly Microsoft.Extensions.Options.IOptionsMonitor<UiOptions> _ui;

    public string Title { get; } = "환경 기반 설정 데모";
    public string CurrentTheme { get; private set; } = "";
    public string CurrentLanguage { get; private set; } = "";
    public string ApiStatus { get; private set; } = "";

    public ReactiveUI.ReactiveCommand<Unit, Unit> RefreshCommand { get; }

    public HomeViewModel(IApiClient api, Microsoft.Extensions.Options.IOptionsMonitor<UiOptions> ui)
    {
        _api = api; _ui = ui;

        ApplyUi(ui.CurrentValue);
        _ui.OnChange(o => Avalonia.Threading.Dispatcher.UIThread.Post(() => ApplyUi(o)));

        RefreshCommand = ReactiveUI.ReactiveCommand.CreateFromTask(async () =>
        {
            ApiStatus = await _api.GetStatusAsync();
            this.RaisePropertyChanged(nameof(ApiStatus));
        });
    }

    private void ApplyUi(UiOptions o)
    {
        CurrentTheme = o.Theme;
        CurrentLanguage = o.Language;
        this.RaisePropertyChanged(nameof(CurrentTheme));
        this.RaisePropertyChanged(nameof(CurrentLanguage));
    }
}
```

---

## 결론

- Avalonia에서도 .NET의 **구성 시스템**을 그대로 활용하면 **환경(dev/prod)별 설정 분리**가 쉽고 강력하다.  
- `Options 패턴 + IOptionsMonitor + reloadOnChange` 조합으로 **런타임 재로딩**까지 매끄럽게 지원한다.  
- 운영상 필수인 **유효성 검증, 비밀 관리, 경로 설계, CI/CD 주입, 테스트 가능성**을 함께 설계하면 **유지보수성과 배포 안정성, UX**를 모두 확보할 수 있다.