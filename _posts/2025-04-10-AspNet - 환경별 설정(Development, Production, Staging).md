---
layout: post
title: AspNet - 환경별 설정(Development, Production, Staging)
date: 2025-04-10 20:20:23 +0900
category: AspNet
---
# 🌐 ASP.NET Core 환경별 설정 적용 (`Development`, `Production`, `Staging`)

---

## ✅ 1. 환경(Environment) 이란?

ASP.NET Core는 애플리케이션이 실행되는 **환경 종류**에 따라  
설정 파일이나 로직을 **자동으로 분기**할 수 있게 해줍니다.

### 기본 제공 환경 이름

| 이름 | 설명 |
|------|------|
| `Development` | 개발 환경 |
| `Staging`     | 중간 테스트 환경 |
| `Production`  | 운영 환경 |

> ✅ 환경 이름은 커스텀 이름도 가능하지만 위 3가지를 표준으로 권장함

---

## 🧭 2. 환경 설정 방법

### (1) 환경 변수로 지정

| 플랫폼 | 명령 |
|--------|------|
| Windows CMD | `set ASPNETCORE_ENVIRONMENT=Development` |
| PowerShell | `$env:ASPNETCORE_ENVIRONMENT="Production"` |
| Linux/macOS | `export ASPNETCORE_ENVIRONMENT=Staging` |

### (2) `launchSettings.json` (개발 전용)

```json
{
  "profiles": {
    "MyApp": {
      "commandName": "Project",
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development"
      }
    }
  }
}
```

> 이 설정은 Visual Studio, `dotnet run` 등에만 반영됨

---

## 🧾 3. 환경별 설정 파일 구성

ASP.NET Core는 다음과 같은 파일을 자동으로 병합(override)합니다:

- `appsettings.json` → 기본 설정
- `appsettings.{Environment}.json` → 환경별 덮어쓰기

예시:

```
appsettings.json
appsettings.Development.json
appsettings.Production.json
```

### 🔍 병합 예시

```jsonc
// appsettings.json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information"
    }
  },
  "FeatureX": true
}
```

```jsonc
// appsettings.Production.json
{
  "Logging": {
    "LogLevel": {
      "Default": "Warning"
    }
  },
  "FeatureX": false
}
```

> `Production` 환경에서는 `LogLevel`이 `Warning`, `FeatureX`는 `false`로 적용됨

---

## ⚙️ 4. 코드에서 환경 판별하기 (`IWebHostEnvironment`)

```csharp
public class HomeController : Controller
{
    private readonly IWebHostEnvironment _env;

    public HomeController(IWebHostEnvironment env)
    {
        _env = env;
    }

    public IActionResult Index()
    {
        if (_env.IsDevelopment())
        {
            ViewBag.Message = "개발 환경입니다.";
        }
        else if (_env.IsProduction())
        {
            ViewBag.Message = "운영 환경입니다.";
        }

        return View();
    }
}
```

---

## 🔐 5. 환경별로 실행 로직 분기

```csharp
if (app.Environment.IsDevelopment())
{
    app.UseDeveloperExceptionPage();  // 상세 오류
}
else
{
    app.UseExceptionHandler("/Error"); // 사용자 친화적인 오류 페이지
    app.UseHsts(); // HTTPS 보안 강화
}
```

> `app.Environment`는 `builder.Environment` 또는 `IWebHostEnvironment`로 접근 가능

---

## 🧬 6. 환경별 의존성 주입

```csharp
if (builder.Environment.IsDevelopment())
{
    builder.Services.AddSingleton<IMyService, DevMyService>();
}
else
{
    builder.Services.AddSingleton<IMyService, ProdMyService>();
}
```

---

## ⚠️ 7. 환경별 민감 정보 보호

| 구성 | 처리 방식 |
|------|------------|
| DB 연결 문자열 | `appsettings.Production.json`에만 넣거나 환경 변수로 설정 |
| API Key | Secret Manager (개발) 또는 환경 변수 (운영) 사용 |
| 디버깅 도구 | 개발 환경에서만 사용 (`DeveloperExceptionPage`) |

---

## 📊 8. 환경별 설정 우선순위

```text
appsettings.json
↓ 병합
appsettings.{ENV}.json
↓ 병합
환경 변수
↓ 병합
명령줄 인자
```

> 나중에 설정된 값이 앞의 설정을 덮어씌움

---

## ✅ 요약

| 항목 | 설명 |
|------|------|
| 환경 설정 | `ASPNETCORE_ENVIRONMENT` 환경 변수 |
| 기본 파일 | `appsettings.json` |
| 환경별 파일 | `appsettings.Development.json` 등 |
| 환경 판별 코드 | `env.IsDevelopment()`, `env.IsProduction()` |
| 민감 정보 | 환경 변수 또는 Secret Manager 사용 권장 |