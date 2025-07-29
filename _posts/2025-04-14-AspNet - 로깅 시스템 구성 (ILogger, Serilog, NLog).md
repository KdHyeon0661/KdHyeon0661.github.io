---
layout: post
title: AspNet - 로깅 시스템 구성 (ILogger, Serilog, NLog)
date: 2025-04-10 21:20:23 +0900
category: AspNet
---
# 📋 ASP.NET Core 로깅 시스템 구성 (`ILogger`, `Serilog`, `NLog`)

---

## ✅ 1. ASP.NET Core 로깅 개요

ASP.NET Core는 강력한 **로깅 추상화 시스템**을 제공하여  
`ILogger` 인터페이스를 기반으로 다양한 로깅 제공자(콘솔, 파일, DB 등)를 쉽게 설정할 수 있음.

---

## 🧩 2. 기본 로깅 구성 (`ILogger` 사용)

ASP.NET Core는 `ILogger<T>`를 기본 DI로 제공함.

### 🔹 사용 예

```csharp
public class HomeController : Controller
{
    private readonly ILogger<HomeController> _logger;
    
    public HomeController(ILogger<HomeController> logger)
    {
        _logger = logger;
    }

    public IActionResult Index()
    {
        _logger.LogInformation("홈페이지 접근됨");
        _logger.LogWarning("주의가 필요한 이벤트!");
        _logger.LogError("오류 발생!");

        return View();
    }
}
```

---

## 🪵 3. 로그 레벨

| 로그 레벨 | 설명 |
|-----------|------|
| `Trace`   | 가장 상세한 로그 (디버깅용) |
| `Debug`   | 디버깅 정보 |
| `Information` | 일반 흐름 정보 |
| `Warning` | 예상된 문제 |
| `Error`   | 예외 발생 등 심각한 오류 |
| `Critical` | 시스템 다운 등의 치명적 오류 |

### 🔹 appsettings.json 설정

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  }
}
```

---

## 🧪 4. 파일/콘솔 로깅 (기본 제공)

ASP.NET Core는 기본적으로 `Console`, `Debug`, `EventSource`를 지원함.

- 콘솔 출력: 개발 시
- 디버그 출력: Visual Studio Output 창 등

---

## 💎 5. Serilog로 로깅 고급화하기

### 🔹 Serilog란?

- 구조화 로깅(structured logging) 지원
- 다양한 Sink (파일, 콘솔, DB, ElasticSearch 등) 제공
- 강력한 템플릿, 필터링, 출력 포맷 가능

---

### 📦 설치

```bash
dotnet add package Serilog.AspNetCore
dotnet add package Serilog.Sinks.File
```

---

### 🛠️ 구성 예 (Program.cs)

```csharp
using Serilog;

Log.Logger = new LoggerConfiguration()
    .MinimumLevel.Debug()
    .WriteTo.Console()
    .WriteTo.File("Logs/log-.txt", rollingInterval: RollingInterval.Day)
    .CreateLogger();

var builder = WebApplication.CreateBuilder(args);

// 기존 로깅 제거 + Serilog 사용
builder.Host.UseSerilog();

var app = builder.Build();

app.MapGet("/", (ILogger<Program> logger) =>
{
    logger.LogInformation("요청 처리됨!");
    return "Hello, world!";
});

app.Run();
```

---

### 📄 구조화 로깅 예

```csharp
logger.LogInformation("유저 {UserId}가 {Action}을 수행했습니다", userId, action);
```

결과:
```json
{
  "UserId": 42,
  "Action": "로그인",
  "Message": "유저 42가 로그인 을 수행했습니다"
}
```

---

## 🧰 6. NLog 설정하기 (대안 로거)

- 성능이 좋고 설정 파일을 XML로 관리 가능
- 기업 환경에서도 많이 사용됨

---

### 📦 설치

```bash
dotnet add package NLog.Web.AspNetCore
```

---

### 📁 nlog.config 파일 생성

```xml
<nlog>
  <targets>
    <target name="file" xsi:type="File" fileName="Logs/nlog.txt" />
  </targets>
  <rules>
    <logger name="*" minlevel="Info" writeTo="file" />
  </rules>
</nlog>
```

---

### 🛠️ Program.cs에 설정

```csharp
builder.Logging.ClearProviders();
builder.Host.UseNLog();
```

---

## 🔐 7. 민감 정보 필터링

로그에 개인정보, 비밀번호 등이 포함되지 않도록 주의!

```csharp
logger.LogInformation("비밀번호 입력: {Password}", "[FILTERED]");
```

또는 Serilog/NLog에서 자체적으로 `Filter` 기능을 통해 필터링 가능

---

## ✅ 8. 요약 비교

| 항목 | 기본 `ILogger` | Serilog | NLog |
|------|----------------|---------|------|
| 내장 제공 | ✅ | ❌ | ❌ |
| 파일 로깅 | 간단 | 고급 가능 | 고급 가능 |
| 구조화 로깅 | 제한적 | ✅ 매우 우수 | 제한적 |
| 설정 파일 분리 | JSON | JSON or 코드 | XML 기반 |
| 확장성 | 보통 | 매우 우수 | 우수 |

---

## 📊 9. 로깅 팁

- `ILogger`는 **모든 클래스에 DI로 주입 가능**
- 실시간 분석이 필요하면 **Serilog + Elastic Stack** 추천
- `Try-Catch` 내에서 `LogError(ex, "...")` 활용 필수
- `LogCritical`은 시스템 종료나 다운 시점에 사용

---

## 🔜 추천 다음 주제

- ✅ Serilog의 Enrichers로 사용자 정보 포함시키기
- ✅ 로그 레벨 동적 변경 (`appsettings.json` → 실시간 변경)
- ✅ Cloud logging: AWS CloudWatch / Azure Monitor 연동
- ✅ `ILoggerFactory`를 통한 커스텀 로거 구현