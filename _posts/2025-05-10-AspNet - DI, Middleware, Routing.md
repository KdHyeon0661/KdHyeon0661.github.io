---
layout: post
title: AspNet - DI, Middleware, Routing
date: 2025-05-10 23:20:23 +0900
category: AspNet
---
# ASP.NET Core 핵심 개념 확장 요약: DI, Middleware, Routing

## 0. 큰 그림: 요청이 앱을 통과하는 방법

1. 앱 시작 시 **DI 컨테이너**에 서비스 등록(싱글턴/스코프/트랜지언트, 옵션, HttpClient 등)
2. **미들웨어 파이프라인** 구성(정적 파일 → 라우팅 → 인증/권한 → 엔드포인트)
3. **라우팅**이 URL을 **엔드포인트**(컨트롤러/레이저/Minimal API)로 매핑
4. 대상 핸들러는 **DI**로 의존성을 주입받아 실행
5. 응답이 미들웨어를 역방향으로 지나며 공통 후처리(로깅, 헤더, 압축 등)

---

## 1. DI(Dependency Injection) — 기본에서 고급까지

### 1.1 기본 등록 & 수명(Lifetime) 복습

```csharp
builder.Services.AddTransient<IMyService, MyService>();   // 매 호출마다 새 인스턴스
builder.Services.AddScoped<IUserService, UserService>();  // 요청(Request)당 1개
builder.Services.AddSingleton<ILogger, ConsoleLogger>();  // 앱 전체 1개
```

- **Transient**: 가볍고 상태 없는 서비스(파서, 매퍼, 규칙 검사 등)
- **Scoped**: 요청 흐름에 걸쳐 유지되어야 하는 서비스(Unit of Work, 리포지토리)
- **Singleton**: 구성/캐시/클라이언트(스레드 안전) 등 **공유 자원**

#### Anti-pattern 주의
- **Scoped → Singleton 주입 금지**: 싱글턴에서 스코프 서비스 참조 시 수명 역전
- **Heavy Transient 남발**: 고빈도 호출에서 GC 압박; 풀링/캐시 고려

---

### 1.2 생성자 주입(권장) & 기타 주입

```csharp
public class HomeController : Controller
{
    private readonly IMyService _service;
    public HomeController(IMyService service) => _service = service;

    public IActionResult Index() => View(_service.GetData());
}
```

- 생성자 주입이 기본. 필요한 경우 **메서드 주입**(Minimal API), **옵션 주입**(IOptions) 병행.

---

### 1.3 Options 패턴 — 설정을 타입으로

```csharp
public sealed class MyOptions
{
    public string Endpoint { get; init; } = "";
    public int CacheSeconds { get; init; } = 60;
}

builder.Services
    .AddOptions<MyOptions>()
    .Bind(builder.Configuration.GetSection("MyOptions"))
    .Validate(o => Uri.IsWellFormedUriString(o.Endpoint, UriKind.Absolute), "Endpoint invalid")
    .ValidateOnStart(); // 앱 시작 시 유효성 점검
```

사용:

```csharp
public class DataService(IOptionsSnapshot<MyOptions> options, HttpClient http)
{
    public async Task<string> FetchAsync()
        => await http.GetStringAsync(options.Value.Endpoint);
}
```

> **IOptions**(싱글턴), **IOptionsSnapshot**(스코프/요청별 스냅샷), **IOptionsMonitor**(런타임 변경 감지) 차이를 목적에 맞게 선택.

---

### 1.4 HttpClient 팩토리 — 타임아웃/폴리시/네이밍

```csharp
builder.Services.AddHttpClient("Weather", c =>
{
    c.BaseAddress = new Uri("https://api.example.com/");
    c.Timeout = TimeSpan.FromSeconds(5);
});
```

Typed client:

```csharp
public class WeatherClient(HttpClient http)
{
    public Task<string> GetAsync() => http.GetStringAsync("weather/today");
}
builder.Services.AddHttpClient<WeatherClient>(c =>
{
    c.BaseAddress = new Uri("https://api.example.com/");
});
```

> Polly(Jitter/Retry/CircuitBreaker) 연계 권장.

---

### 1.5 Open Generics / Decorator / Scrutor / Keyed Services(.NET 8)

#### Open Generics
```csharp
builder.Services.AddScoped(typeof(IRepository<>), typeof(EfRepository<>));
```

#### Decorator (Scrutor)
```csharp
// dotnet add package Scrutor
builder.Services.AddScoped<IOrderService, OrderService>();
builder.Services.Decorate<IOrderService, CachingOrderService>();
```

#### Keyed Services(.NET 8)
```csharp
builder.Services.AddKeyedScoped<IStorage>("s3", sp => new S3Storage(...));
builder.Services.AddKeyedScoped<IStorage>("blob", sp => new BlobStorage(...));

public class Uploader([FromKeyedServices("s3")] IStorage storage) { ... }
```

> 기능별 구현 다형성을 **키 기반**으로 주입 — A/B, 멀티 테넌시에서 유용.

---

### 1.6 Factory/Scope/Async Disposal

- **IServiceScopeFactory**로 **수동 스코프** 생성(백그라운드 작업 등)
- **IAsyncDisposable** 구현 서비스는 컨테이너가 안전하게 `DisposeAsync` 호출
- **BackgroundService** 내부에서 **스코프 서비스 사용 시** `CreateScope()` 필수

```csharp
public class Worker(IServiceScopeFactory scopes) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            using var scope = scopes.CreateScope();
            var repo = scope.ServiceProvider.GetRequiredService<IRepo>();
            await repo.DoAsync(ct);
        }
    }
}
```

---

## 2. 미들웨어 — 파이프라인, 순서, 분기, 단위테스트

### 2.1 파이프라인 핵심

```csharp
var app = builder.Build();

app.UseStaticFiles();          // 1) 정적 파일
app.UseRouting();              // 2) 라우팅 매핑
app.UseAuthentication();       // 3) 인증
app.UseAuthorization();        // 4) 권한
app.MapControllers();          // 5) 엔드포인트 실행
app.Run();
```

- **UseX**: 다음으로 넘김(`await _next(ctx)`), **Map/MapWhen**: 하위 브랜치, **Run**: 종단(더 이상 진행 X)
- **순서가 기능**. 잘못된 순서 → 인증 무효, 404/401 혼선 등

---

### 2.2 커스텀 미들웨어

```csharp
public class LoggingMiddleware(RequestDelegate next, ILogger<LoggingMiddleware> log)
{
    public async Task InvokeAsync(HttpContext ctx)
    {
        var path = ctx.Request.Path;
        log.LogInformation("REQ {Path}", path);
        await next(ctx); // 다음
        log.LogInformation("RES {Status}", ctx.Response.StatusCode);
    }
}

app.UseMiddleware<LoggingMiddleware>();
```

> **예외 처리**는 파이프라인 상단에서 `UseExceptionHandler`/글로벌 핸들러로 일괄 처리.

---

### 2.3 분기(특정 경로/조건만)

```csharp
// 경로 분기
app.Map("/healthz", sub => sub.Run(async ctx => await ctx.Response.WriteAsync("OK")));

// 조건 분기
app.UseWhen(ctx => ctx.Request.Path.StartsWithSegments("/api"), branch =>
{
    branch.Use(async (ctx, next) =>
    {
        ctx.Response.Headers.TryAdd("X-Api", "v1");
        await next();
    });
});
```

---

### 2.4 Minimal API Filters(.NET 7+) & Endpoint Filters(.NET 8)

```csharp
app.MapPost("/orders", (OrderDto dto) => Results.Ok())
   .AddEndpointFilter(async (ctx, next) =>
   {
       if (dto.Amount <= 0) return Results.BadRequest("Invalid");
       return await next(ctx); // 필터 체인
   });
```

> 컨트롤러 필터(ActionFilter)와 유사하지만 **Minimal API/엔드포인트 중심**.

---

### 2.5 Rate Limiting, CORS, Compression(간단 예)

```csharp
builder.Services.AddResponseCompression();
app.UseResponseCompression();

app.UseCors(p => p.WithOrigins("https://app.example.com").AllowAnyHeader().AllowAnyMethod());

builder.Services.AddRateLimiter(_ => _.AddFixedWindowLimiter("fixed", o =>
{
    o.Window = TimeSpan.FromSeconds(10);
    o.PermitLimit = 100;
}));
app.UseRateLimiter();
```

---

### 2.6 미들웨어 테스트

- **WebApplicationFactory**(Microsoft.AspNetCore.Mvc.Testing)로 통합 검증
- 헤더/리다이렉트/상태코드/쿠키 확인

```csharp
public class PipelineTests(CustomFactory factory)
{
    [Fact]
    public async Task Adds_Custom_Header()
    {
        var client = factory.CreateClient();
        var resp = await client.GetAsync("/");
        Assert.True(resp.Headers.Contains("X-Api"));
    }
}
```

---

## 3. 라우팅 — 속성 라우팅, 제약, 링크 생성, 그룹, 버저닝

### 3.1 Razor Pages

```csharp
app.MapRazorPages(); // Pages/Index.cshtml → "/"
```

파일 구조 기반 매핑 + `@page "{id:int}"`로 페이지 자체 라우팅 가능.

---

### 3.2 MVC 속성 라우팅

```csharp
[ApiController]
[Route("api/[controller]")]
public class UsersController : ControllerBase
{
    [HttpGet("{id:int:min(1)}")]
    public IActionResult Get(int id) => Ok(id);
}
```

- **제약**: `int`, `guid`, `min`, `max`, `length`, `alpha`, `regex(...)` 등
- **기본값**: `"{lang=en}"`와 결합 가능

---

### 3.3 전역 라우트 템플릿 / 기본 라우트

```csharp
app.MapControllerRoute(
    name: "default",
    pattern: "{controller=Home}/{action=Index}/{id?}");
```

---

### 3.4 Parameter Transformer(슬러그 변환)

```csharp
// dotnet add package Microsoft.AspNetCore.Mvc.Razor.RuntimeCompilation (개발 시)
public class SlugifyParameterTransformer : IOutboundParameterTransformer
{
    public string? TransformOutbound(object? value)
        => value?.ToString()?.Replace('_','-').ToLowerInvariant();
}

builder.Services.AddRouting(opt => opt.ConstraintMap["slugify"] = typeof(SlugifyParameterTransformer));
```

라우트 사용 예:
```csharp
app.MapControllerRoute(
    name: "slug",
    pattern: "blog/{title:slugify}",
    defaults: new { controller = "Blog", action = "Show" });
```

---

### 3.5 LinkGenerator로 안전한 링크 생성

```csharp
public class MenuService(LinkGenerator linker, IHttpContextAccessor accessor)
{
    public string? UserLink(int id)
        => linker.GetPathByAction(
            httpContext: accessor.HttpContext!,
            action: "Get",
            controller: "Users",
            values: new { id });
}
```

> 라우트 변경에도 **컴파일 의존 없는** 안전한 링크.

---

### 3.6 Minimal API + 그룹 라우팅(.NET 8)

```csharp
var api = app.MapGroup("/api").RequireRateLimiting("fixed");
api.MapGet("/ping", () => "pong");
api.MapGet("/users/{id:int}", (int id) => Results.Ok(new { id }));
```

- 그룹에 **공통 미들웨어/필터/정책** 적용
- 버전 그룹: `/api/v1`, `/api/v2` 등 계층화

---

### 3.7 API Versioning(간단 개념)

- 패키지: `Asp.Versioning.Http` (전문 버저닝)
- 경로/헤더/쿼리 기반 버전 협상 + `ApiVersion` 특성 사용

---

## 4. 엔드투엔드 예제(Program.cs + 컨트롤러 + 테스트)

### 4.1 Program.cs

```csharp
var builder = WebApplication.CreateBuilder(args);

// DI
builder.Services.AddScoped<IUserService, UserService>();
builder.Services.AddHttpClient<WeatherClient>(c =>
    c.BaseAddress = new Uri("https://api.example.com/"));

// Options
builder.Services.AddOptions<MyOptions>()
       .Bind(builder.Configuration.GetSection("MyOptions"))
       .Validate(o => o.CacheSeconds > 0)
       .ValidateOnStart();

// MVC
builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

// Pipeline
var app = builder.Build();

app.UseSwagger();
app.UseSwaggerUI();

app.UseResponseCompression();
app.UseRouting();

app.UseAuthentication();
app.UseAuthorization();

app.MapControllers();

app.Run();

// --- types ---
public sealed class MyOptions { public int CacheSeconds { get; init; } = 10; }
public interface IUserService { Task<UserDto?> GetAsync(int id); }
public sealed class UserService : IUserService
{
    public Task<UserDto?> GetAsync(int id)
        => Task.FromResult<UserDto?>(id > 0 ? new(id, $"User_{id}") : null);
}
public sealed record UserDto(int Id, string Name);
public sealed class WeatherClient(HttpClient http)
{
    public Task<string> GetAsync() => http.GetStringAsync("weather/today");
}
```

### 4.2 컨트롤러(속성 라우팅 + 제약 + DI)

```csharp
[ApiController]
[Route("api/users")]
public class UsersController(IUserService users, IOptionsSnapshot<MyOptions> opts) : ControllerBase
{
    [HttpGet("{id:int:min(1)}")]
    public async Task<IActionResult> Get(int id)
    {
        var u = await users.GetAsync(id);
        if (u is null) return NotFound();
        Response.Headers["X-Cache-Sec"] = opts.Value.CacheSeconds.ToString();
        return Ok(u);
    }
}
```

### 4.3 통합 테스트(WebApplicationFactory)

```csharp
public class ApiTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;
    public ApiTests(WebApplicationFactory<Program> factory) => _client = factory.CreateClient();

    [Fact]
    public async Task Get_User_Returns200_And_Header()
    {
        var res = await _client.GetAsync("/api/users/5");
        var body = await res.Content.ReadAsStringAsync();
        Assert.True(res.IsSuccessStatusCode);
        Assert.Contains("X-Cache-Sec", res.Headers.ToString());
        Assert.Contains("User_5", body);
    }
}
```

---

## 5. 성능·안정성 체크리스트

### 5.1 DI/옵션/HttpClient
- **HttpClientFactory** 사용(소켓 고갈 방지)
- 구성 검증: `ValidateOnStart`
- Options는 **IOptionsSnapshot**(웹) vs **IOptionsMonitor**(백그라운드) 구분

### 5.2 미들웨어/파이프라인
- **순서** 검증(라우팅 전/후, 인증/권한 순서)
- 비용 큰 로직은 **UseWhen/MapGroup**으로 필요한 경로에만
- 응답 스트리밍/압축/캐시 헤더 세팅 최적화

### 5.3 라우팅
- 제약/정규식은 최소화(성능 고려)
- LinkGenerator로 링크 안전성 유지
- Minimal API 그룹으로 정책 일괄 적용(권한/레이트 리밋)

### 5.4 메모리/할당
- `AsNoTracking()`(EF Core), `ArrayPool`/`ObjectPool`(빈번한 할당 방지)
- DTO/Record 구조체화는 신중하게(복사 비용/박싱 고려)

---

## 6. 보안·유지보수·운영 팁

- **헤더 표준화**: `X-Correlation-ID`, `X-Request-Time`
- **예외 처리**: 글로벌 핸들러 + ProblemDetails(표준 오류 응답)
- **로그 구조화**: `logger.LogInformation("User {UserId}", id)`
- **헬스체크/레디니스**: `/healthz/live`, `/healthz/ready`
- **Feature Flags**: `Microsoft.FeatureManagement`로 런타임 전환
- **버전 호환**: 라우트 버전/헤더 버전으로 점진적 이전

---

## 7. 흔한 오류 패턴과 해법

| 문제 | 원인 | 해결 |
|---|---|---|
| Singleton이 DbContext 참조 | 수명 역전 | DbContext는 Scoped, 필요한 곳에 Scope 생성 |
| 인증은 했는데 권한 실패 | 미들웨어 순서 | `UseAuthentication` → `UseAuthorization` 순서 준수 |
| 404가 간헐적으로 발생 | 라우팅/미들웨어 순서 충돌 | `UseRouting`/`MapControllers` 배치 점검 |
| 소켓/핸들 고갈 | HttpClient 직접 new | HttpClientFactory 사용 |
| 응답 헤더 누락 | 엔드포인트 이후에 헤더 설정 | 실행 전/필터에서 설정 또는 `OnStarting` 사용 |

---

## 8. 요약 표 — DI / Middleware / Routing

| 영역 | 핵심 | 고급 포인트 |
|---|---|---|
| DI | 수명 관리(Transient/Scoped/Singleton) | Options 패턴, Typed HttpClient, Decorator, Keyed Services |
| Middleware | 순서가 곧 기능 | Use/Map/UseWhen, Endpoint Filters, Rate Limiting |
| Routing | 속성/규칙/그룹 라우팅 | 제약/변환자, LinkGenerator, 버저닝, Minimal API 그룹 |

---

## 9. 실무용 스니펫 모음

### 9.1 ProblemDetails(표준 오류 응답)
```csharp
app.UseExceptionHandler(errApp =>
{
    errApp.Run(async ctx =>
    {
        var pd = Results.Problem(statusCode: 500, title: "Unhandled error");
        await pd.ExecuteAsync(ctx);
    });
});
```

### 9.2 응답 시작 직전 헤더 주입
```csharp
app.Use(async (ctx, next) =>
{
    ctx.Response.OnStarting(() =>
    {
        ctx.Response.Headers.TryAdd("X-App", "Core");
        return Task.CompletedTask;
    });
    await next();
});
```

### 9.3 Minimal API + 권한/필터/캐싱
```csharp
var v1 = app.MapGroup("/api/v1").RequireAuthorization("Admin");
v1.MapGet("/stats", (IMetrics m) => Results.Ok(m.Snapshot()))
  .CacheOutput(p => p.Expire(TimeSpan.FromSeconds(30)));
```

---

## 결론

- **DI**는 객체 수명/옵션/HttpClient/데코레이터/키드 서비스까지 **설계의 근간**이다.  
- **미들웨어**는 **순서가 기능**이다. 필요한 경로에만 적용하고, 예외/로그/보안/압축/캐시를 체계화하라.  
- **라우팅**은 단순 매핑을 넘어 **제약, 링크 생성, 그룹 정책, 버저닝**으로 서비스의 진화를 지지한다.  

위 확장 가이드를 골격으로 삼아, 코드 스니펫을 **그대로 복사/조합**하면 **운영급 파이프라인**을 구축할 수 있다.