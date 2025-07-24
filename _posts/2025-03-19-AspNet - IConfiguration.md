---
layout: post
title: AspNet - IConfiguration
date: 2025-03-19 19:20:23 +0900
category: AspNet
---
# ⚙️ ASP.NET Core에서 구성 객체 `IConfiguration` 완전 정복

ASP.NET Core에서는 앱의 설정 정보를 `appsettings.json`, 환경 변수, 명령줄 인수 등 다양한 소스에서 가져옵니다.  
이 정보를 통합해서 읽을 수 있도록 해주는 것이 바로 **`IConfiguration`** 인터페이스입니다.

---

## 🧠 IConfiguration이란?

- 설정 정보를 추상화한 인터페이스
- 계층형 키 기반의 설정 구조 (`:` 또는 `__` 구분자)
- **appsettings.json**, 환경 변수, 커맨드라인 인수 등에서 읽기 가능
- `DI`로 어디서든 주입받아 사용할 수 있음

---

## 📄 기본 사용법

### 1. `appsettings.json`에 설정 추가

```json
// appsettings.json
{
  "AppSettings": {
    "SiteName": "My ASP.NET App",
    "MaxItems": 10
  }
}
```

---

### 2. `IConfiguration`으로 값 읽기

```csharp
public class IndexModel : PageModel
{
    private readonly IConfiguration _config;

    public IndexModel(IConfiguration config)
    {
        _config = config;
    }

    public void OnGet()
    {
        string siteName = _config["AppSettings:SiteName"];
        int maxItems = int.Parse(_config["AppSettings:MaxItems"]);
        // 또는 int.TryParse
    }
}
```

- `:`로 계층적 키 접근
- JSON의 구조를 그대로 반영

---

## 📦 강력한 방법: 설정 바인딩 (`GetSection().Bind()`)

클래스와 매핑하여 한번에 구성 값을 주입받을 수 있음

### 1. POCO 클래스 정의

```csharp
public class AppSettings
{
    public string SiteName { get; set; }
    public int MaxItems { get; set; }
}
```

---

### 2. 구성 바인딩 등록

```csharp
// Program.cs
builder.Services.Configure<AppSettings>(
    builder.Configuration.GetSection("AppSettings"));
```

---

### 3. 사용 시 주입 (IOptions 사용)

```csharp
using Microsoft.Extensions.Options;

public class IndexModel : PageModel
{
    private readonly AppSettings _settings;

    public IndexModel(IOptions<AppSettings> options)
    {
        _settings = options.Value;
    }

    public void OnGet()
    {
        var name = _settings.SiteName;
        var max = _settings.MaxItems;
    }
}
```

> ✅ `IOptions<T>` 패턴은 재시작하지 않아도 실시간 변경 감지 가능 (`IOptionsSnapshot`, `IOptionsMonitor`로 확장 가능)

---

## 🌍 다른 설정 소스 사용하기

ASP.NET Core는 다양한 구성 소스를 자동 통합합니다:

| 소스 | 예시 |
|------|------|
| `appsettings.json` | 앱 설정 기본 파일 |
| `appsettings.{Environment}.json` | 환경별 설정 (예: `appsettings.Production.json`) |
| 환경 변수 | `AppSettings__SiteName=NewName` |
| 명령줄 인수 | `--AppSettings:SiteName=CLIName` |
| 사용자 비밀 | 개발용 민감 정보 저장 (`dotnet user-secrets`) |

모든 설정은 우선순위 순서로 병합되며, **나중 값이 우선 적용**됩니다.

---

## ⚠️ 환경 변수 예시

환경 변수에서는 `:` 대신 `__` 사용:

```bash
export AppSettings__SiteName="EnvApp"
```

→ 코드에서는 여전히 `_config["AppSettings:SiteName"]` 으로 접근

---

## 🧪 실전 디버깅 팁

전체 설정을 순회하며 확인 가능:

```csharp
foreach (var kv in _config.AsEnumerable())
{
    Console.WriteLine($"{kv.Key} = {kv.Value}");
}
```

---

## 📌 IConfiguration vs IOptions

| 항목 | IConfiguration | IOptions<T> |
|------|----------------|-------------|
| 접근 방식 | 문자열 키 | 바인딩된 객체 |
| 타입 안정성 | 낮음 (수동 변환 필요) | 높음 (자동 매핑) |
| 실시간 변경 감지 | 불가 | `IOptionsSnapshot`, `IOptionsMonitor`로 가능 |
| 단순 조회 | 적합 | 부적합 |
| 구조화된 설정 | 불편 | 매우 편리 |

---

## 💡 구성 파일 분리 전략

- `appsettings.json`: 공통 설정
- `appsettings.Development.json`: 개발 전용
- `appsettings.Production.json`: 배포용
- `appsettings.Staging.json`: 중간 배포 테스트용
- `.AddJsonFile("custom.json")`으로 직접 커스텀 구성 가능

---

## 🔐 민감 정보 관리 - User Secrets

개발 환경에서 비밀번호, API 키 등 민감한 정보는  
`user-secrets` 기능으로 관리

```bash
dotnet user-secrets init
dotnet user-secrets set "ApiKey" "123456"
```

→ 코드에서 `_config["ApiKey"]`로 접근 가능  
→ `appsettings.json`에 노출되지 않음

---

## ✅ 마무리 요약

| 항목 | 설명 |
|------|------|
| `IConfiguration` | 키 기반 설정 접근 |
| `GetSection()` | 하위 설정 섹션 가져오기 |
| `Bind()` | 클래스와 바인딩 |
| `IOptions<T>` | 강력한 타입 지원 방식 |
| `환경 변수`, `명령줄`, `비밀` | 다양한 설정 소스 지원 |
| 우선순위 | CLI > 환경변수 > appsettings.Production.json > appsettings.json |

---

## 🔜 다음 추천 주제

- ✅ `IOptionsSnapshot`, `IOptionsMonitor`를 이용한 실시간 설정 감지
- ✅ 환경별 실행 분기 (`IWebHostEnvironment`)
- ✅ `ILogger<T>`와 구성 연동 (로깅 수준 조정 등)
- ✅ 사용자 지정 구성 공급자 만들기