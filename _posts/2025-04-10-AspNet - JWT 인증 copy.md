---
layout: post
title: AspNet - appsettings.json과 환경 변수
date: 2025-04-10 19:20:23 +0900
category: AspNet
---
# ⚙️ ASP.NET Core: `appsettings.json`과 환경 변수 완전 정리

---

## ✅ 1. ASP.NET Core의 설정 시스템 개요

ASP.NET Core는 **다중 설정 소스**를 조합하여 설정을 구성합니다:

> 기본 순서 (우선순위 ↑)
1. `appsettings.json`
2. `appsettings.{Environment}.json`
3. **환경 변수 (Environment Variables)**
4. 커맨드라인 인자
5. Secret Manager (개발용)
6. 사용자 지정 Provider

---

## 📁 2. `appsettings.json` 기본 구조

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "ConnectionStrings": {
    "DefaultConnection": "Server=.;Database=AppDb;Trusted_Connection=True;"
  },
  "MySettings": {
    "FeatureEnabled": true,
    "MaxItems": 100
  }
}
```

> ⛔ 민감 정보(비밀번호, API Key)는 직접 넣지 말 것!

---

## 🧪 3. 환경별 설정 파일: `appsettings.Development.json`

- 환경에 따라 다른 설정을 적용할 수 있습니다
- 파일 이름: `appsettings.{환경이름}.json`

```json
// appsettings.Development.json
{
  "MySettings": {
    "FeatureEnabled": false,
    "MaxItems": 10
  }
}
```

---

## 🌎 4. 현재 환경 설정 방법 (`ASPNETCORE_ENVIRONMENT`)

| 환경 | 용도 |
|------|------|
| `Development` | 개발 |
| `Staging`     | 테스트 |
| `Production`  | 운영 배포 |

### 설정 방법 (예시):

- Windows (CMD):
  ```cmd
  set ASPNETCORE_ENVIRONMENT=Development
  ```

- PowerShell:
  ```powershell
  $env:ASPNETCORE_ENVIRONMENT="Development"
  ```

- Linux/macOS (bash):
  ```bash
  export ASPNETCORE_ENVIRONMENT=Production
  ```

---

## 🧑‍💻 5. 설정 값 읽기 (`IConfiguration` 사용)

```csharp
public class MyService
{
    private readonly IConfiguration _config;

    public MyService(IConfiguration config)
    {
        _config = config;
    }

    public void Print()
    {
        var feature = _config.GetValue<bool>("MySettings:FeatureEnabled");
        Console.WriteLine($"Feature Enabled: {feature}");
    }
}
```

---

## 🧬 6. 바인딩: 설정을 객체로 매핑하기

```csharp
public class MySettings
{
    public bool FeatureEnabled { get; set; }
    public int MaxItems { get; set; }
}
```

### 등록

```csharp
builder.Services.Configure<MySettings>(
    builder.Configuration.GetSection("MySettings"));
```

### 사용

```csharp
public class HomeController : Controller
{
    private readonly MySettings _settings;

    public HomeController(IOptions<MySettings> options)
    {
        _settings = options.Value;
    }
}
```

---

## 🔐 7. 환경 변수 사용 (보안, CI/CD 등에서 중요)

### 등록 방법 (예시):

```bash
export MySettings__FeatureEnabled=true
export ConnectionStrings__DefaultConnection="Server=prod-db;..."
```

> `:` 대신 `__` (언더바 2개) 사용!

---

## 💥 8. 환경 변수 vs appsettings 비교

| 항목 | `appsettings.json` | 환경 변수 |
|------|---------------------|------------|
| 위치 | 파일 시스템 | OS 환경 |
| 용도 | 일반 설정 | 비밀 키, 비밀번호, 배포별 차이 |
| 보안 | 소스 코드에 노출될 수 있음 | 안전하게 관리 가능 |
| 우선순위 | 낮음 | 높음 (덮어씀) |

---

## ⛓️ 9. 설정 로드 순서 및 병합

ASP.NET Core는 설정을 병합(override)하는 방식으로 구성합니다:

```text
appsettings.json
↓ 병합
appsettings.Development.json
↓ 병합
환경 변수
↓ 병합
명령줄 인자
```

- 나중에 추가된 소스가 **앞의 설정을 덮어씀**

---

## 📦 10. 실전 예시

```bash
export Logging__LogLevel__Default=Debug
export MySettings__MaxItems=500
```

`Logging:LogLevel:Default` → `Debug`로 설정됨  
`MySettings:MaxItems` → `500`으로 덮어씀

---

## ✅ 요약

| 항목 | 설명 |
|------|------|
| `appsettings.json` | 기본 설정 저장용 |
| `appsettings.{env}.json` | 환경별 설정 덮어쓰기 |
| 환경 변수 | 민감 정보, 배포 자동화 |
| `IConfiguration` | 설정 읽기 인터페이스 |
| 우선순위 | 환경 변수 > 설정 파일 |
| 실전 팁 | API Key는 반드시 환경변수나 Secret Manager에 저장할 것 |

---

## 🔜 추천 다음 주제

- ✅ `User Secrets` (개발용 민감 정보 관리)
- ✅ `IOptionsSnapshot` / `IOptionsMonitor` 비교
- ✅ 구성 변경에 실시간 반응하는 방법
- ✅ Azure App Configuration 사용