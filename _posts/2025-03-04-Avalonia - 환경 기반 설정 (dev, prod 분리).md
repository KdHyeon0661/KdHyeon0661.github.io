---
layout: post
title: Avalonia - 환경 기반 설정 (dev, prod 분리)
date: 2025-03-04 20:20:23 +0900
category: Avalonia
---
# Avalonia - 환경 기반 설정 (dev, prod 분리)

Avalonia 애플리케이션을 개발할 때는 개발(dev), 운영(prod) 환경에 따라 API 주소, 로깅 수준, 기능 플래그 등을 다르게 설정해야 합니다. .NET의 `Microsoft.Extensions.Configuration`과 `Microsoft.Extensions.DependencyInjection`을 활용하면 **환경별 설정 파일 분리**, **강타입 옵션 바인딩**, **런타임 재로딩**, **유효성 검증**을 손쉽게 구현할 수 있습니다. 이 글에서는 초중급 개발자를 기준으로, Avalonia 프로젝트에서 환경 기반 설정을 구축하는 방법을 단계별로 설명합니다.

---

## 파일 구조와 설정 파일

프로젝트 루트에 다음과 같은 JSON 설정 파일을 둡니다.

```
Project Root/
├── appsettings.json            # 공통 기본값 (필수)
├── appsettings.dev.json        # 개발 환경 전용
├── appsettings.prod.json       # 운영 환경 전용
├── appsettings.local.json      # 개인 로컬 오버라이드 (소스 관리 제외)
├── MyApp.csproj
└── Program.cs
```

**설명**  
- `appsettings.json`은 모든 환경의 공통 값(예: 기본 API 주소)을 담습니다.  
- `appsettings.{env}.json`은 환경별 값을 담습니다. 예를 들어 개발용 API는 `https://dev-api.example.com`, 운영용은 `https://api.example.com`으로 설정합니다.  
- `appsettings.local.json`은 개발자 개인 PC에서만 필요한 값(예: 로컬 디버깅용 API)을 담으며, `.gitignore`에 추가하여 소스 관리에서 제외합니다.

---

## 환경 선택

.NET은 환경 변수 `DOTNET_ENVIRONMENT`로 현재 환경을 식별합니다. 값은 `Development`, `Staging`, `Production` 등이 일반적이나 간단히 `dev`, `prod`를 사용해도 됩니다.

```csharp
string env = Environment.GetEnvironmentVariable("DOTNET_ENVIRONMENT") ?? "prod";
```

- 환경 변수가 설정되지 않았다면 기본값을 `prod`로 두어 운영 환경에서 안전하게 동작하도록 합니다.  
- Visual Studio에서는 프로젝트 속성에서 환경 변수를 설정하거나, 디버그 시 명령줄 인자로 전달할 수 있습니다.

---

## 설정 로딩 (우선순위)

`ConfigurationBuilder`는 **등록 순서가 중요**합니다. 나중에 등록된 소스가 앞선 값을 덮어씁니다. 일반적인 우선순위는 다음과 같습니다.

1. `appsettings.json` (공통)
2. `appsettings.{env}.json` (환경별)
3. `appsettings.local.json` (로컬 오버라이드)
4. 환경 변수 (CI/CD나 Docker 등에서 주입)
5. 명령줄 인수

`Program.cs` 또는 `App.axaml.cs`에서 다음과 같이 구성합니다.

```csharp
using Microsoft.Extensions.Configuration;

string env = Environment.GetEnvironmentVariable("DOTNET_ENVIRONMENT") ?? "prod";

var config = new ConfigurationBuilder()
    .SetBasePath(AppContext.BaseDirectory)                // 실행 파일 위치 기준
    .AddJsonFile("appsettings.json", optional: false, reloadOnChange: true)
    .AddJsonFile($"appsettings.{env}.json", optional: true, reloadOnChange: true)
    .AddJsonFile("appsettings.local.json", optional: true, reloadOnChange: true)
    .AddEnvironmentVariables()                             // 예: Api__BaseUrl
    .AddCommandLine(args)                                  // 예: --Api:BaseUrl=https://override
    .Build();
```

`reloadOnChange: true`를 설정하면 파일 변경 시 구성이 다시 로드되어, 앱을 재시작하지 않고도 설정을 반영할 수 있습니다.

---

## 강타입 옵션 클래스

설정 값은 문자열이 아닌 강타입 클래스로 바인딩하면 코드 안정성이 높아집니다. 예를 들어 API, 기능 플래그, UI 설정을 위한 클래스를 만듭니다.

```csharp
// Options/ApiOptions.cs
public sealed class ApiOptions
{
    public string BaseUrl { get; set; } = "https://api.example.com";
    public int TimeoutSeconds { get; set; } = 30;
    public bool UseCompression { get; set; } = true;
}

// Options/FeatureFlags.cs
public sealed class FeatureFlags
{
    public bool EnableNewDashboard { get; set; } = false;
    public bool DevToolsVisible { get; set; } = false;
    public bool UseMockData { get; set; } = false;
}

// Options/UiOptions.cs
public sealed class UiOptions
{
    public string Theme { get; set; } = "Light";
    public string Language { get; set; } = "ko";
    public double DefaultFontSize { get; set; } = 13;
}
```

---

## DI 등록과 IOptionsMonitor

Microsoft.Extensions.DependencyInjection을 사용하여 옵션을 DI 컨테이너에 등록합니다.

```csharp
using Microsoft.Extensions.DependencyInjection;

var services = new ServiceCollection();

services.Configure<ApiOptions>(config.GetSection("Api"));
services.Configure<FeatureFlags>(config.GetSection("Features"));
services.Configure<UiOptions>(config.GetSection("Ui"));

// 추가 서비스 등록...
```

**IOptionsMonitor<T>**를 주입하면 설정 파일이 변경될 때 이벤트를 수신할 수 있습니다. 예를 들어 `ApiClient`에서 API 주소 변경을 실시간 반영하려면:

```csharp
public class ApiClient : IApiClient
{
    private readonly HttpClient _http;
    private readonly IOptionsMonitor<ApiOptions> _apiOptions;

    public ApiClient(HttpClient http, IOptionsMonitor<ApiOptions> apiOptions)
    {
        _http = http;
        _apiOptions = apiOptions;
        ApplyOptions(_apiOptions.CurrentValue);
        _apiOptions.OnChange(ApplyOptions);   // 파일 변경 시 콜백
    }

    private void ApplyOptions(ApiOptions opts)
    {
        _http.BaseAddress = new Uri(opts.BaseUrl);
        _http.Timeout = TimeSpan.FromSeconds(opts.TimeoutSeconds);
    }
}
```

ViewModel에서 UI 설정을 실시간 반영하려면:

```csharp
public class HomeViewModel : ReactiveObject
{
    private readonly IOptionsMonitor<UiOptions> _uiOptions;
    public string Theme { get; private set; }

    public HomeViewModel(IOptionsMonitor<UiOptions> uiOptions)
    {
        _uiOptions = uiOptions;
        Theme = uiOptions.CurrentValue.Theme;
        uiOptions.OnChange(o => Avalonia.Threading.Dispatcher.UIThread.Post(() =>
        {
            Theme = o.Theme;
            this.RaisePropertyChanged(nameof(Theme));
        }));
    }
}
```

---

## 설정 파일 예시

### appsettings.json (공통)

```json
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

### appsettings.dev.json (개발 환경)

```json
{
  "Api": {
    "BaseUrl": "https://dev-api.example.com",
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

### appsettings.prod.json (운영 환경)

```json
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

## 유효성 검증 (Fail Fast)

설정 값이 올바르지 않으면 앱 시작 시 즉시 예외를 발생시켜 문제를 조기에 발견할 수 있습니다. `IValidateOptions<T>` 인터페이스를 구현합니다.

```csharp
public class ApiOptionsValidator : IValidateOptions<ApiOptions>
{
    public ValidateOptionsResult Validate(string name, ApiOptions options)
    {
        if (string.IsNullOrWhiteSpace(options.BaseUrl))
            return ValidateOptionsResult.Fail("Api:BaseUrl is required.");
        if (!Uri.IsWellFormedUriString(options.BaseUrl, UriKind.Absolute))
            return ValidateOptionsResult.Fail("Api:BaseUrl must be an absolute URI.");
        if (options.TimeoutSeconds <= 0 || options.TimeoutSeconds > 600)
            return ValidateOptionsResult.Fail("Api:TimeoutSeconds must be between 1 and 600.");
        return ValidateOptionsResult.Success;
    }
}
```

DI 등록 시 검증을 추가합니다.

```csharp
services.AddOptions<ApiOptions>()
        .Bind(config.GetSection("Api"))
        .ValidateOnStart()   // 앱 시작 시 검증
        .Services.AddSingleton<IValidateOptions<ApiOptions>, ApiOptionsValidator>();
```

---

## 민감 정보 (비밀) 관리

**절대** 설정 파일에 API 키나 비밀번호를 평문으로 저장하지 마세요. 대신 운영체제가 제공하는 안전한 저장소를 사용합니다.

| 운영체제 | 권장 저장소 |
|----------|------------|
| Windows | DPAPI (ProtectedData), Windows Credential Manager |
| macOS | Keychain |
| Linux | Secret Service (GNOME Keyring, KWallet) |

예를 들어 Windows DPAPI를 사용해 문자열을 암호화하고 복호화하는 유틸리티를 만들 수 있습니다.

```csharp
public static class SecretStore
{
    public static string Protect(string plain)
    {
        var bytes = System.Text.Encoding.UTF8.GetBytes(plain);
        var cipher = System.Security.Cryptography.ProtectedData.Protect(bytes, null, System.Security.Cryptography.DataProtectionScope.CurrentUser);
        return Convert.ToBase64String(cipher);
    }

    public static string Unprotect(string cipher)
    {
        var data = Convert.FromBase64String(cipher);
        var plain = System.Security.Cryptography.ProtectedData.Unprotect(data, null, System.Security.Cryptography.DataProtectionScope.CurrentUser);
        return System.Text.Encoding.UTF8.GetString(plain);
    }
}
```

실제 키는 이 저장소에서 읽어와 DI로 주입합니다.

---

## 배포 시 설정 파일 포함

.csproj 파일에서 설정 파일을 출력 디렉터리에 복사하도록 지정합니다.

```xml
<ItemGroup>
  <None Update="appsettings.json" CopyToOutputDirectory="PreserveNewest" />
  <None Update="appsettings.dev.json" CopyToOutputDirectory="PreserveNewest" />
  <None Update="appsettings.prod.json" CopyToOutputDirectory="PreserveNewest" />
  <None Update="appsettings.local.json" CopyToOutputDirectory="Never" />
</ItemGroup>
```

CI 파이프라인에서 환경 변수 `DOTNET_ENVIRONMENT`를 설정하고, 필요한 파일만 포함하도록 합니다. GitHub Actions 예시:

```yaml
- name: Publish
  run: dotnet publish -c Release -r win-x64 --self-contained true -o out
- name: Set environment
  run: echo "DOTNET_ENVIRONMENT=prod" >> $GITHUB_ENV
```

---

## 테스트

구성 바인딩을 단위 테스트로 검증할 수 있습니다.

```csharp
[Fact]
public void ApiOptions_Binds_Correctly()
{
    var dict = new Dictionary<string, string?>
    {
        ["Api:BaseUrl"] = "https://test.com",
        ["Api:TimeoutSeconds"] = "15"
    };
    var config = new ConfigurationBuilder().AddInMemoryCollection(dict).Build();

    var services = new ServiceCollection();
    services.Configure<ApiOptions>(config.GetSection("Api"));
    var sp = services.BuildServiceProvider();
    var opts = sp.GetRequiredService<IOptions<ApiOptions>>().Value;

    Assert.Equal("https://test.com", opts.BaseUrl);
    Assert.Equal(15, opts.TimeoutSeconds);
}
```

---

## 요약

| 주제 | 핵심 내용 |
|------|----------|
| 환경 선택 | `DOTNET_ENVIRONMENT` 환경 변수 |
| 로딩 순서 | 공통 → 환경 → 로컬 → 환경변수 → 명령줄 |
| 옵션 패턴 | `services.Configure<T>()` + `IOptionsMonitor<T>` |
| 재로딩 | `reloadOnChange: true` + `OnChange` 이벤트 |
| 유효성 검증 | `IValidateOptions<T>` + `ValidateOnStart()` |
| 민감 정보 | OS 보안 저장소, 환경 변수 |
| 배포 | `.csproj` 복사 설정, CI 환경 변수 |
| 테스트 | `InMemoryCollection`으로 구성 모의 |

---

## 결론

Avalonia 앱에서도 .NET의 표준 구성 시스템을 그대로 활용하면 환경별 설정 분리, 강타입 옵션, 런타임 재로딩, 유효성 검증을 쉽게 구현할 수 있습니다. 이렇게 구축된 설정 체계는 개발/운영 환경을 명확히 구분하고, 민감 정보를 안전하게 관리하며, 배포 자동화를 원활하게 만듭니다. 위 패턴을 자신의 프로젝트에 적용하여 안정적이고 유지보수하기 쉬운 Avalonia 애플리케이션을 만들어 보세요.