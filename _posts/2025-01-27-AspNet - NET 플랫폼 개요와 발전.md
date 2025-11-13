---
layout: post
title: AspNet - NET 플랫폼 개요와 발전
date: 2025-01-27 19:20:23 +0900
category: AspNet
---
# .NET 플랫폼 개요와 발전

## 0. .NET 세대 구분 요약(확장)

| 항목 | .NET Framework | .NET Core | .NET (5/6/7/8) |
|---|---|---|---|
| 초판/통합 | 2002(.NET 1.0) | 2016(Core 1.0) | 2020(.NET 5부터 통합 브랜드) |
| 플랫폼 | Windows 전용 | Windows/macOS/Linux | 크로스 플랫폼(서버·데스크톱·클라우드·IoT) |
| 라이선스/개발 | 폐쇄+일부 오픈 | 완전 오픈소스 | 완전 오픈소스 |
| 성능/현대화 | 무겁고 Windows 친화 | 경량·고성능 | 지속 최적화(HTTP/2/3, AOT, R2R 등) |
| 유지보수 | 레거시(기능 추가 미미) | 현행 이전 세대 | **현행 표준(.NET 8 LTS)** |
| 대표 기술 | ASP.NET, WPF, WinForms | ASP.NET Core 초창기, CLI | ASP.NET Core, gRPC, Minimal API, Blazor, MAUI 등 |

### 0.1 릴리스 모델
- **LTS (Long Term Support)**: 3년 지원. 예: **.NET 8 LTS**. 서버/엔터프라이즈 권장.
- **STS (Standard Term Support)**: 18개월 전후 지원. 새 기능 시험·조기 도입.

> 실무 팁: 신규/중요 서비스는 **LTS**를 기본값으로 삼고, 프리뷰/STS는 **파일럿·비핵심 워크로드**에 한정하는 전략이 안전합니다.

### 0.2 .NET Core → .NET으로 “브랜드 통합”
- **.NET 5**부터 “Core” 접미사가 사라지고 **.NET**으로 통합.
- 러닝타임/SDK/라이브러리/도구 체계를 **단일 생태계**로 일원화.

---

## 1. 마이그레이션 로드맵(Framework → Core/현행 .NET)

현실적으로 다음 네 단계를 권장합니다.

1) **분리**: UI/호스팅 의존(예: System.Web)을 비즈니스 로직에서 분리 → **클래스 라이브러리(.NET Standard/.NET)**로 핵심 로직 이동
2) **등가치 대체**:
   - `Global.asax`/HTTP 핸들러/모듈 → **미들웨어 파이프라인**
   - `Web.config` → **appsettings.json + Program.cs** 구성
   - 인증/권한 OWIN → **ASP.NET Core 인증/정책 인가**
3) **라이브러리 치환**: 오래된 패키지(이미지 처리, 캐시, ORM 등)를 **현대 .NET 호환** 대안으로 교체
4) **운영 자동화**: 컨테이너(Docker), CI/CD, Health Check, 관측성(OpenTelemetry) 도입

간단 체크리스트:
- System.Web 의존 제거 여부
- WebForms → Razor Pages/MVC 전환 계획
- 인증 스택(쿠키/JWT/OIDC) 매핑표
- 구성/비밀(User Secrets/Key Vault) 전략 수립
- 배포 모델(단일 파일, Self-contained) 선택

---

## 2. ASP.NET Core의 특징(확장)

1) **크로스 플랫폼**: Windows/IIS, Linux/Nginx, macOS 개발 환경
2) **고성능/현대화**: Kestrel, HTTP/2·3, 압축/캐싱, AOT/ReadyToRun, 스레드 풀 최적화
3) **통합 프레임워크**: Razor Pages/MVC, Web API/Minimal API, SignalR, gRPC, Blazor
4) **DI 기본 탑재**: `AddTransient`/`AddScoped`/`AddSingleton` 수명 주기
5) **모듈형 미들웨어**: 파이프라인 구성으로 보안/성능/로깅 일원화
6) **클라우드 친화**: 구성 계층(AppSettings/ENV/KeyVault), Health Checks, 컨테이너
7) **오픈소스**: 투명한 로드맵/이슈 관리, 예측 가능한 릴리스

---

## 3. 프로젝트 뼈대(Program.cs)와 공통 인프라

```csharp
// Program.cs (.NET 8)
using Microsoft.EntityFrameworkCore;

var builder = WebApplication.CreateBuilder(args);

// 서비스 등록
builder.Services.AddRazorPages();          // Razor Pages
builder.Services.AddControllers();         // MVC/Web API
builder.Services.AddSignalR();             // SignalR
builder.Services.AddDbContext<AppDb>(opt => opt.UseSqlite("Data Source=app.db"));

builder.Services.AddHealthChecks().AddDbContextCheck<AppDb>();
builder.Services.AddAuthorization(o => o.AddPolicy("AdminOnly", p => p.RequireRole("Admin")));

var app = builder.Build();

// 파이프라인
if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler("/Error");
    app.UseHsts();
}

app.UseHttpsRedirection();
app.UseStaticFiles();

app.UseRouting();
app.UseAuthentication();
app.UseAuthorization();

// 엔드포인트
app.MapRazorPages();
app.MapControllers();
app.MapHub<NotifyHub>("/hubs/notify");
app.MapHealthChecks("/health");

app.Run();

public sealed class AppDb : DbContext
{
    public AppDb(DbContextOptions<AppDb> o) : base(o) {}
    public DbSet<Product> Products => Set<Product>();
}
public sealed class Product { public int Id { get; set; } public string Name { get; set; } = ""; public int Price { get; set; } }
public sealed class NotifyHub : Microsoft.AspNetCore.SignalR.Hub { }
```

---

## 4. Razor Pages — 서버 렌더링 UI(간단 CRUD에 최적)

### 4.1 페이지/모델 쌍

`Pages/Products/Index.cshtml.cs`
```csharp
using Microsoft.AspNetCore.Mvc.RazorPages;
using Microsoft.EntityFrameworkCore;

public class IndexModel : PageModel
{
    private readonly AppDb _db;
    public IndexModel(AppDb db) => _db = db;

    public List<Product> Items { get; private set; } = [];

    public async Task OnGetAsync()
        => Items = await _db.Products.AsNoTracking().OrderBy(p => p.Id).ToListAsync();
}
```

`Pages/Products/Index.cshtml`
```cshtml
@page
@model IndexModel
@{
    ViewData["Title"] = "Products";
}
<h1>Products</h1>
<table>
  <thead><tr><th>Id</th><th>Name</th><th>Price</th></tr></thead>
  <tbody>
  @foreach (var p in Model.Items)
  {
    <tr><td>@p.Id</td><td>@p.Name</td><td>@p.Price</td></tr>
  }
  </tbody>
</table>
```

### 4.2 생성/수정/삭제 핸들러(요지)

`Pages/Products/Create.cshtml.cs`
```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.RazorPages;

public class CreateModel : PageModel
{
    private readonly AppDb _db;
    public CreateModel(AppDb db) => _db = db;

    [BindProperty] public Product Item { get; set; } = new();

    public void OnGet() { }

    public async Task<IActionResult> OnPostAsync()
    {
        if (!ModelState.IsValid) return Page();
        _db.Add(Item);
        await _db.SaveChangesAsync();
        return RedirectToPage("./Index");
    }
}
```

---

## 5. MVC — 구조화/복잡 UI/SEO/필터가 중요한 경우

`Controllers/ProductsController.cs`
```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;

[Route("products")]
public class ProductsController : Controller
{
    private readonly AppDb _db;
    public ProductsController(AppDb db) => _db = db;

    [HttpGet("")]
    public async Task<IActionResult> Index()
    {
        var items = await _db.Products.AsNoTracking().ToListAsync();
        return View(items); // Views/Products/Index.cshtml
    }
}
```

**언제 MVC를 선택하나?**
- 대규모 사이트, 복잡한 필터/모델 바인딩, 정교한 SEO 제어, 기존 MVC 자산 재사용

---

## 6. Web API — REST/내부 서비스/프론트엔드 분리

### 6.1 컨트롤러 기반 API
```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;

[ApiController]
[Route("api/v1/products")]
public class ProductsApiController : ControllerBase
{
    private readonly AppDb _db;
    public ProductsApiController(AppDb db) => _db = db;

    [HttpGet]
    public async Task<IEnumerable<Product>> Get()
        => await _db.Products.AsNoTracking().ToListAsync();

    [HttpPost]
    public async Task<IActionResult> Create(Product p)
    {
        _db.Add(p); await _db.SaveChangesAsync();
        return CreatedAtAction(nameof(GetById), new { id = p.Id }, p);
    }

    [HttpGet("{id:int}")]
    public async Task<ActionResult<Product>> GetById(int id)
        => await _db.Products.FindAsync(id) is { } e ? e : NotFound();
}
```

### 6.2 Minimal API(경량)
```csharp
var group = app.MapGroup("/api/v2/products").WithTags("Products");
group.MapGet("/", async (AppDb db) => Results.Ok(await db.Products.ToListAsync()));
group.MapPost("/", async (AppDb db, Product p) => { db.Add(p); await db.SaveChangesAsync(); return Results.Created($"/api/v2/products/{p.Id}", p); });
```

**컨트롤러 vs Minimal API 선택**
- **컨트롤러**: 필터/모델 검증/버전/표준화가 필요한 **공용 API**
- **Minimal**: 내부 API/게이트웨이/경량 라우팅/빠른 프로토타이핑

---

## 7. Blazor — C#으로 프론트엔드까지

### 7.1 Blazor Server(서버 연결형)
- 서버에서 UI 상태를 보유, **SignalR**을 통해 DOM 동기화
- 빠른 초기 로딩, 보안 경계 서버

`Pages/Counter.razor`
```razor
@page "/counter"
<h3>Counter</h3>
<p>Current count: @_count</p>
<button @onclick="Inc">Click me</button>

@code {
    private int _count;
    void Inc() => _count++;
}
```

### 7.2 Blazor WebAssembly(클라이언트 실행형)
- 브라우저에서 **WASM 런타임**으로 C# 실행(오프라인 가능)
- 초기 다운로드 부담, 이후 상호작용 매우 빠름

> 선택 기준: 보안/SEO/초기 로딩/오프라인 요구사항을 비교해 Server/WASM을 결정하거나 **하이브리드**를 고려합니다.

---

## 8. SignalR — 실시간 양방향(채팅/알림/대시보드)

### 8.1 허브
```csharp
using Microsoft.AspNetCore.SignalR;

public class ChatHub : Hub
{
    public Task Send(string user, string message)
        => Clients.All.SendAsync("message", user, message);
}
```

### 8.2 서버 등록
```csharp
builder.Services.AddSignalR();
app.MapHub<ChatHub>("/hubs/chat");
```

### 8.3 클라이언트(JS)
```html
<script src="/lib/signalr/signalr.min.js"></script>
<script>
  const conn = new signalR.HubConnectionBuilder().withUrl("/hubs/chat").build();
  conn.on("message", (user, msg) => console.log(`${user}: ${msg}`));
  conn.start().then(() => conn.invoke("Send", "alice", "hello"));
</script>
```

**사용처**: 실시간 협업, 모니터링 보드, 주문 현황, IoT 이벤트 스트림

---

## 9. 보안·구성·성능(핵심 팁)

### 9.1 인증/권한(요지)
```csharp
builder.Services.AddAuthentication("Cookies").AddCookie("Cookies", o => o.LoginPath = "/Account/Login");
builder.Services.AddAuthorization(o => o.AddPolicy("AdminOnly", p => p.RequireRole("Admin")));
app.UseAuthentication(); app.UseAuthorization();
```

### 9.2 CORS·보안 헤더
```csharp
app.UseCors(p => p.WithOrigins("https://app.example.com").AllowAnyHeader().AllowAnyMethod().AllowCredentials());
app.Use(async (ctx, next) => {
  ctx.Response.Headers.TryAdd("X-Content-Type-Options", "nosniff");
  ctx.Response.Headers.TryAdd("X-Frame-Options", "DENY");
  await next();
});
```

### 9.3 응답 압축·캐싱·레이트 리미트(요지)
```csharp
builder.Services.AddResponseCompression();
builder.Services.AddMemoryCache();
builder.Services.AddRateLimiter(o => o.AddFixedWindowLimiter("api", opt => { opt.PermitLimit = 100; opt.Window = TimeSpan.FromMinutes(1); }));
app.UseResponseCompression();
app.UseRateLimiter();
```

---

## 10. 배포(컨테이너/리버스 프록시/단일 파일)

### 10.1 Dockerfile
```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY . .
RUN dotnet publish -c Release -o /out

FROM mcr.microsoft.com/dotnet/aspnet:8.0
WORKDIR /app
COPY --from=build /out .
ENV ASPNETCORE_URLS=http://+:8080
EXPOSE 8080
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

### 10.2 Nginx 리버스 프록시(요지)
```nginx
server {
  listen 80;
  server_name example.com;
  location / {
    proxy_pass http://app:8080;
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
  }
}
```

### 10.3 Self-contained/Single-File/ReadyToRun
```bash
dotnet publish -c Release -r linux-x64 --self-contained true -o out/linux
dotnet publish -c Release -r win-x64 -p:PublishSingleFile=true -o out/win
dotnet publish -c Release -p:PublishReadyToRun=true -o out/r2r
```

---

## 11. 관측성/운영: Health Checks·OpenTelemetry(개요)

```csharp
builder.Services.AddHealthChecks().AddDbContextCheck<AppDb>();
app.MapHealthChecks("/health");
```

- OpenTelemetry(추적/지표/로그) → OTLP Exporter → Tempo/Jaeger/Prometheus/Loki 등
- 컨테이너 오케스트레이션(Kubernetes)과 연계해 **liveness/readiness**에 사용

---

## 12. 선택 가이드(무엇을 언제 쓸까?)

| 요구 | 권장 스택 |
|---|---|
| 서버 렌더링 CRUD, 단순 페이지 | **Razor Pages** |
| 복잡 UI/SEO/필터·모델·정책 | **MVC** |
| 프런트 분리(React/Vue/Svelte) 백엔드 | **Web API**(컨트롤러) 또는 **Minimal API**(경량) |
| C#로 SPA/복잡 프런트 | **Blazor**(Server/WASM) |
| 실시간 알림/대시보드/콜라보 | **SignalR** |
| 내부 고성능 RPC | **gRPC**(여기서는 개요만 언급) |

---

## 13. 실전 미니 샘플 통합(한 프로젝트에 동시 탑재)

- `/`           → Razor Pages 홈
- `/products`   → MVC 뷰
- `/api/v1/...` → Web API
- `/hubs/chat`  → SignalR 허브
- `/health`     → 헬스 체크

위에서 제시한 **Program.cs**를 베이스로, **페이지/컨트롤러/허브** 각각의 예제 파일을 추가하면 **단일 호스트에서 혼합 구동**이 가능합니다. 파일/폴더 구조는 다음처럼 정리합니다.

```
MyApp/
├── Pages/               # Razor Pages
├── Controllers/         # MVC/API
├── Hubs/                # SignalR 허브
├── Data/Models/         # EF Core 엔터티/DbContext
├── wwwroot/             # 정적 파일
├── Program.cs
└── appsettings.json
```

---

## 14. 흔한 함정과 해결책

1) `UseAuthentication`/`UseAuthorization` 순서 혼동 → **Authentication이 먼저**
2) CORS에서 `AllowAnyOrigin` + `AllowCredentials` 동시 사용 금지
3) EF Core N+1 → Projection/Include 조합·캐시로 완화
4) Swagger에서 민감 엔드포인트 노출 → 환경별 제한/보안 스키마 설정
5) Trimming/AOT로 리플렉션 대상 제거 → `DynamicallyAccessedMembers` 주석/소스 제너레이터 고려

---

## 15. 요약

- **.NET Framework → .NET Core → .NET(5~8)**로 오며 **크로스 플랫폼·고성능·오픈소스**로 수렴했습니다.
- 현재 실무 표준은 **.NET 8 LTS**. 신규 프로젝트는 이를 기본값으로 삼는 것이 안전합니다.
- **ASP.NET Core**는 **Razor Pages/MVC/Web API/Blazor/SignalR**을 하나의 호스트에서 유연하게 결합할 수 있어, **모놀리식에서 마이크로서비스까지** 폭넓은 구조를 커버합니다.
- 본 문서의 **샘플 코드**를 바탕으로, **필요 최소 스택부터** 시작한 뒤 요구사항에 따라 **SignalR(실시간)·Blazor(UI)·gRPC(RPC)·관측성**을 점진적으로 도입하십시오.
