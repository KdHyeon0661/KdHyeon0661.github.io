---
layout: post
title: AspNet - 로깅 시스템 구성 (ILogger, Serilog, NLog)
date: 2025-04-10 21:20:23 +0900
category: AspNet
---
# ASP.NET Core 로깅 시스템 완전 가이드

## ASP.NET Core 로깅 추상화 개요

- 모든 로그는 `Microsoft.Extensions.Logging`의 **`ILogger<T>` 추상화**를 통해 기록.
- 실제 출력(콘솔/파일/DB/Elastic/Seq/Cloud)은 **프로바이더**가 담당(플러그 가능).
- 장점: 코드 변경 없이 구성만 바꿔 **Serilog/NLog** 등으로 **갈아끼우기**가 용이.

```csharp
public sealed class HomeController : Controller
{
    private readonly ILogger<HomeController> _logger;
    public HomeController(ILogger<HomeController> logger) => _logger = logger;

    public IActionResult Index()
    {
        _logger.LogInformation("홈페이지 접근됨");
        _logger.LogWarning("주의가 필요한 이벤트");
        try
        {
            // ...
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Index 처리 중 오류");
        }
        return View();
    }
}
```

---

## 로그 레벨, 카테고리, 필터링

### 로그 레벨

| 레벨 | 용도 |
|---|---|
| Trace | 최세부(핫패스 디버깅) |
| Debug | 개발·진단 |
| Information | 정상 플로우 이벤트 |
| Warning | 경고·한계치 도달 |
| Error | 실패 이벤트(복구 필요) |
| Critical | 시스템 중단 급 |

### `appsettings.json` 필터

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft": "Warning",
      "Microsoft.AspNetCore.Hosting": "Information",
      "MyApp.Infrastructure": "Debug"
    }
  }
}
```

- **카테고리** = 일반적으로 클래스 **네임스페이스**.
- 특정 영역(예: `MyApp.Infrastructure`)만 수준 상향/하향 가능.

---

## `ILogger` 핵심 패턴 — 구조화, 스코프, 이벤트 ID

### 구조화 메시지(파라미터를 필드로)

```csharp
_logger.LogInformation("사용자 {UserId}가 {Action}을 수행", userId, action);
```

- Serilog/NLog 등 **구조화 가능한 프로바이더**에서 `{UserId}`, `{Action}`이 **필드**로 저장되어 조회·쿼리 가능.

### 스코프(`BeginScope`) — 요청/트랜잭션 상관관계

```csharp
using (_logger.BeginScope(new Dictionary<string, object?>
{
    ["CorrelationId"] = HttpContext.TraceIdentifier,
    ["User"] = User.Identity?.Name ?? "anonymous"
}))
{
    _logger.LogInformation("주문 조회 시작: {OrderId}", orderId);
    // ...
}
```

- 스코프 내 모든 로그에 **공통 메타데이터**가 자동 첨부.

### 이벤트 ID

```csharp
public static class LogEvents
{
    public static readonly EventId OrderFetched = new(1001, nameof(OrderFetched));
}

_logger.LogInformation(LogEvents.OrderFetched, "주문 {OrderId} 조회", orderId);
```

- SIEM/대시보드에서 **이벤트 유형 기반** 분석에 유용.

---

## 콘솔/디버그/이벤트소스 기본 프로바이더

Program.cs(또는 `builder.Logging`)에서 활성화/제거 가능:

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Logging
    .ClearProviders()       // 선택: 기본 제공 모두 제거
    .AddConsole()
    .AddDebug();            // VS Output 창
```

- **개발**: 콘솔/디버그 중심.
- **운영**: 파일/원격 싱크(Serilog/NLog/Otel 등) 권장.

---

## Serilog — 구조화 로깅의 표준

### 패키지

```bash
dotnet add package Serilog.AspNetCore
dotnet add package Serilog.Sinks.Console
dotnet add package Serilog.Sinks.File
# 선택: Elastic/Seq/ApplicationInsights/ConsoleJson 등

```

### 부트스트랩(가장 먼저 구성)

```csharp
using Serilog;

Log.Logger = new LoggerConfiguration()
    .MinimumLevel.Information()
    .Enrich.FromLogContext()                        // 스코프/HTTP 컨텍스트
    .Enrich.WithMachineName().Enrich.WithThreadId()
    .WriteTo.Console()
    .WriteTo.File("logs/app-.log", rollingInterval: RollingInterval.Day, retainedFileCountLimit: 7)
    .CreateLogger();

var builder = WebApplication.CreateBuilder(args);
builder.Host.UseSerilog(); // ASP.NET Core 로깅 파이프에 연결

var app = builder.Build();
// ...
app.Run();
```

### JSON 기반 구성(appsettings) + 자동 재로드

```json
{
  "Serilog": {
    "Using": [ "Serilog.Sinks.Console", "Serilog.Sinks.File" ],
    "MinimumLevel": {
      "Default": "Information",
      "Override": {
        "Microsoft": "Warning",
        "MyApp": "Debug"
      }
    },
    "Enrich": [ "FromLogContext", "WithMachineName", "WithThreadId" ],
    "WriteTo": [
      { "Name": "Console" },
      {
        "Name": "File",
        "Args": {
          "path": "logs/app-.log",
          "rollingInterval": "Day",
          "retainedFileCountLimit": 7
        }
      }
    ]
  }
}
```

Program.cs에서 읽기:

```csharp
Log.Logger = new LoggerConfiguration()
    .ReadFrom.Configuration(builder.Configuration, sectionName: "Serilog")
    .CreateLogger();

builder.Host.UseSerilog();
```

> `builder.Configuration`에 `reloadOnChange: true`가 기본 활성화되어 있으면 파일 변경 시 **동적 반영** 가능(일부 항목).

### Serilog Request Logging(자동 요청/응답 로그)

```csharp
app.UseSerilogRequestLogging(options =>
{
    options.MessageTemplate = "HTTP {RequestMethod} {RequestPath} responded {StatusCode} in {Elapsed:0.0000} ms";
    options.EnrichDiagnosticContext = (ctx, http) =>
    {
        ctx.Set("User", http.User.Identity?.Name);
        ctx.Set("TraceId", http.TraceIdentifier);
    };
});
```

- 각 요청 단위의 지표(지연/상태 코드/사용자)를 표준화.

### 민감 정보 마스킹 — 메시지 템플릿/필터

```csharp
// 예: 비밀번호 마스킹
_logger.LogInformation("로그인 시도: user={User}, password={Password}", user, "[FILTERED]");
```

Serilog 필터(고급):

```csharp
dotnet add package Serilog.Expressions
```

```csharp
Log.Logger = new LoggerConfiguration()
    .Filter.ByExcluding("@m like '%password=%'") // 메시지에 패턴 포함 시 제외
    .CreateLogger();
```

### 동적 레벨 스위치

```csharp
var levelSwitch = new Serilog.Core.LoggingLevelSwitch(Serilog.Events.LogEventLevel.Information);

Log.Logger = new LoggerConfiguration()
    .MinimumLevel.ControlledBy(levelSwitch)
    .WriteTo.Console()
    .CreateLogger();

// 런타임에서: levelSwitch.MinimumLevel = LogEventLevel.Debug;
```

운영 중 **진단 레벨 상향**에 유용.

---

## NLog — 고성능·XML 구성 선호 환경

### 패키지 & 기본 설정

```bash
dotnet add package NLog.Web.AspNetCore
```

`nlog.config`:

```xml
<nlog xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
  <targets>
    <target name="console" xsi:type="Console" layout="${longdate}|${uppercase:${level}}|${logger}|${message} ${exception:format=ToString}" />
    <target name="file" xsi:type="File" fileName="logs/nlog-${shortdate}.log" archiveNumbering="Rolling" maxArchiveFiles="7" />
  </targets>
  <rules>
    <logger name="*" minlevel="Info" writeTo="console,file" />
  </rules>
</nlog>
```

Program.cs:

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Logging.ClearProviders();
builder.Host.UseNLog();

var app = builder.Build();
app.Run();
```

### 레이아웃 렌더러로 상관관계/사용자 출력

```xml
<target name="file" xsi:type="File" fileName="logs/app-${shortdate}.log"
        layout="${longdate}|${level}|${logger}|trace=${aspnet-traceidentifier}|user=${aspnet-user-identity}|${message} ${exception}" />
```

---

## 보안/프라이버시 — PII 마스킹·GDPR

- 로그에 **암호/토큰/주민번호/카드번호** 등 **PII** 저장 금지.
- 수집 최소화(원칙), 보존 주기(예: 7~30일)와 파기 정책 명시.
- 샘플 코드(마스킹 헬퍼):

```csharp
public static class PiiMask
{
    public static string MaskEmail(string email)
    {
        if (string.IsNullOrWhiteSpace(email) || !email.Contains('@')) return "[FILTERED]";
        var parts = email.Split('@');
        return parts[0][..1] + "***@" + parts[1];
    }
}
// 사용
_logger.LogInformation("가입: {Email}", PiiMask.MaskEmail(email));
```

---

## 상관관계/분산 추적 — `Activity`/OpenTelemetry 연동

### `Activity`로 Trace/Span 추적

```csharp
using System.Diagnostics;

app.Use(async (ctx, next) =>
{
    var traceId = Activity.Current?.TraceId.ToString() ?? ctx.TraceIdentifier;
    using (_ = _logger.BeginScope(new Dictionary<string, object?> { ["TraceId"] = traceId }))
    {
        await next();
    }
});
```

### OpenTelemetry(OTel) 로깅/트레이싱

```bash
dotnet add package OpenTelemetry.Extensions.Hosting
dotnet add package OpenTelemetry.Exporter.Otlp
dotnet add package OpenTelemetry.Instrumentation.AspNetCore
dotnet add package OpenTelemetry.Instrumentation.Http
```

```csharp
builder.Services.AddOpenTelemetry()
    .WithTracing(b => b
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddOtlpExporter())  // OTLP → Grafana Tempo/Jaeger/Collector
    .WithMetrics(b => b
        .AddAspNetCoreInstrumentation()
        .AddRuntimeInstrumentation()
        .AddOtlpExporter());
```

- Serilog와 함께 사용해도 무방(로그↔트레이스 상호 참조는 `TraceId`로).

---

## 전역 예외 처리와 로깅 통합

- 글로벌 예외 미들웨어(혹은 `UseExceptionHandler`)에서 **`LogError(ex, ...)`** + **`ProblemDetails`** 반환.

```csharp
app.UseExceptionHandler(err =>
{
    err.Run(async ctx =>
    {
        var feature = ctx.Features.Get<IExceptionHandlerFeature>();
        var ex = feature?.Error;
        var status = ex is NotFoundException ? 404 : 500;

        // 로깅
        var traceId = Activity.Current?.Id ?? ctx.TraceIdentifier;
        ctx.RequestServices.GetRequiredService<ILogger<Program>>()
           .LogError(ex, "Unhandled exception. traceId={TraceId}", traceId);

        ctx.Response.StatusCode = status;
        ctx.Response.ContentType = "application/problem+json";
        await ctx.Response.WriteAsJsonAsync(new ProblemDetails
        {
            Status = status,
            Title = status == 404 ? "Not Found" : "Internal Server Error",
            Detail = app.Environment.IsDevelopment() ? ex?.ToString() : null,
            Instance = ctx.Request.Path,
            Extensions = { ["traceId"] = traceId }
        });
    });
});
```

---

## 성능: 비동기·배칭·롤링·필터

- **비동기/배칭**: 파일/네트워크 싱크에서 I/O 병목 최소화(Serilog/NLog 기본 제공).
- **롤링 파일**: 일별/용량별 분할 + 보존 개수 제한.
- **필터**: 불필요한 대량 로그(Trace/Debug, 특정 카테고리) 차단으로 비용 절감.
- **샘플링**: 트래픽 폭증 구간에서 일부만 기록(고급 구성).

Serilog File Sink 예:

```csharp
.WriteTo.File(
   path: "logs/app-.log",
   rollingInterval: RollingInterval.Day,
   retainedFileCountLimit: 7,
   buffered: true,               // 버퍼링(기본)
   shared: false)
```

---

## 환경별 구성/핫 리로드

`appsettings.Development.json`과 `Production.json`에서 로그 레벨/싱크 분리:

```jsonc
// appsettings.Production.json
{
  "Serilog": {
    "MinimumLevel": { "Default": "Information" },
    "WriteTo": [
      { "Name": "File", "Args": { "path": "logs/app-.log", "rollingInterval": "Day" } }
      // 선택: Seq/Elastic/Azure Monitor 등
    ]
  }
}
```

- 운영에선 **Console만으로는 부족** → 파일/원격 수집 꼭 추가.
- `reloadOnChange`: 일부 공급자에서 레벨/필터 동적 반영.

---

## 각 계층에서의 로깅 패턴

### 미들웨어(요청 전후/지표)

```csharp
app.Use(async (ctx, next) =>
{
    var sw = Stopwatch.StartNew();
    await next();
    sw.Stop();

    var log = ctx.RequestServices.GetRequiredService<ILogger<Program>>();
    log.LogInformation("HTTP {Method} {Path} => {Status} in {Elapsed} ms",
        ctx.Request.Method, ctx.Request.Path, ctx.Response.StatusCode, sw.ElapsedMilliseconds);
});
```

### EF Core 로깅(쿼리·성능)

```json
{
  "Logging": {
    "LogLevel": {
      "Microsoft.EntityFrameworkCore.Database.Command": "Information",
      "Microsoft.EntityFrameworkCore.Infrastructure": "Warning"
    }
  }
}
```

- `Information` 이상으로 설정 시 SQL/시간/파라미터 출력(PII 주의).

### HttpClient(외부 호출)

```json
{
  "Logging": {
    "LogLevel": {
      "System.Net.Http.HttpClient": "Information"
    }
  }
}
```

- 요청/응답 요약 로깅(헤더/본문 PII 주의).

### 백그라운드 작업(HostedService)

```csharp
public sealed class Worker : BackgroundService
{
    private readonly ILogger<Worker> _log;
    public Worker(ILogger<Worker> log) => _log = log;

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _log.LogInformation("Worker 시작");
        while (!stoppingToken.IsCancellationRequested)
        {
            using (_log.BeginScope("{Tick}", DateTimeOffset.UtcNow))
                _log.LogDebug("주기 작업 실행");
            await Task.Delay(TimeSpan.FromSeconds(5), stoppingToken);
        }
    }
}
```

---

## 클라우드/집중 수집: Elastic/Seq/CloudWatch/Application Insights

- **Serilog**: `Serilog.Sinks.Elasticsearch`, `Serilog.Sinks.Seq`, `Serilog.Sinks.ApplicationInsights`.
- **NLog**: Elastic/DB 타깃 등 다양한 타깃.
- 쿼리 예(Seq): `@Level = 'Error' and User = 'kimdohyun' and RequestPath = '/api/orders'`.

---

## 운영 가이드 — 알람/보존/용량/비용

- **알람**: `Error/Critical` 폭증, 5xx 비율 증가, 특정 이벤트 ID 임계값 초과.
- **보존**: 규정/보안 기준에 따라 7~90일. 일일 롤링 + 자동 삭제.
- **샘플링**: 피크 타임에는 일부만 저장하거나 카테고리 필터.
- **비용**: 원격 로그 스토리지(Elastic/Seq/Cloud) 과금 주의.

---

## 종합 예시 — Serilog + 글로벌 예외 + 요청 로깅 + EF/HTTP 튜닝

```csharp
using Serilog;

Log.Logger = new LoggerConfiguration()
    .ReadFrom.Configuration(new ConfigurationBuilder()
        .AddJsonFile("appsettings.json", optional: false, reloadOnChange: true)
        .AddJsonFile($"appsettings.{Environment.GetEnvironmentVariable("ASPNETCORE_ENVIRONMENT")}.json", optional: true, reloadOnChange: true)
        .AddEnvironmentVariables()
        .Build(), sectionName: "Serilog")
    .CreateLogger();

var builder = WebApplication.CreateBuilder(args);
builder.Host.UseSerilog();
builder.Services.AddControllers();

// 로깅 필터(기본)
builder.Logging.AddFilter("Microsoft", LogLevel.Warning);
builder.Logging.AddFilter("System.Net.Http.HttpClient", LogLevel.Information);
builder.Logging.AddFilter("Microsoft.EntityFrameworkCore.Database.Command", LogLevel.Information);

var app = builder.Build();

// 요청 로깅
app.UseSerilogRequestLogging(o =>
{
    o.MessageTemplate = "HTTP {RequestMethod} {RequestPath} => {StatusCode} in {Elapsed:0.0000} ms";
});

// 예외 처리 + ProblemDetails
app.UseExceptionHandler(err =>
{
    err.Run(async ctx =>
    {
        var ex = ctx.Features.Get<IExceptionHandlerFeature>()?.Error;
        var status = ex is NotFoundException ? 404 : 500;
        var traceId = Activity.Current?.Id ?? ctx.TraceIdentifier;

        Log.ForContext("TraceId", traceId)
           .Error(ex, "Unhandled exception");

        ctx.Response.StatusCode = status;
        ctx.Response.ContentType = "application/problem+json";
        await ctx.Response.WriteAsJsonAsync(new ProblemDetails
        {
            Status = status,
            Title = status == 404 ? "Not Found" : "Internal Server Error",
            Detail = app.Environment.IsDevelopment() ? ex?.ToString() : null,
            Instance = ctx.Request.Path,
            Extensions = { ["traceId"] = traceId }
        });
    });
});

app.MapControllers();
app.Run();
```

---

## 체크리스트 — 실제 프로젝트에 적용하기

- [ ] 카테고리별 로그 레벨 정의(개발/운영 분리)
- [ ] 구조화 로깅(메시지 템플릿 파라미터 적극 사용)
- [ ] 스코프/TraceId/사용자/요청 ID 부여
- [ ] 민감 정보 마스킹/필터 정책
- [ ] 전역 예외 처리 + `ProblemDetails` 표준
- [ ] 성능: 비동기/배칭/롤링/필터
- [ ] 원격 싱크(Elastic/Seq/Cloud) + 알람
- [ ] 보존 정책/용량 관리/비용 추적
- [ ] 환경별 구성/동적 레벨 스위치
- [ ] EF/HttpClient/BackgroundService 로그 조정

---

## 요약

| 항목 | 핵심 |
|---|---|
| 추상화 | `ILogger<T>`로 코드 독립성 확보 |
| 구조화 | 메시지 템플릿 `{Field}`로 필드화 |
| Serilog/NLog | 강력한 싱크·필터·성능·구성 |
| 보안 | PII 마스킹, 최소 수집, 보존·파기 |
| 관찰성 | TraceId/스코프/OTel로 상관관계 |
| 안정성 | 전역 예외 + 표준 오류(JSON) |
| 운영 | 알람/보존/비용 + 동적 레벨 제어 |

잘 설계된 로깅은 **디버깅 시간 절감**, **운영 가시성 향상**, **보안/컴플라이언스 준수**를 동시에 달성한다.
위 패턴들을 조합해, 개발부터 운영까지 **한 번에 통과하는 로깅 체계**를 구축하자.
