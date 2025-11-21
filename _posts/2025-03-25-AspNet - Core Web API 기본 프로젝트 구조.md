---
layout: post
title: AspNet - Core Web API 기본 프로젝트 구조
date: 2025-03-25 19:20:23 +0900
category: AspNet
---
# ASP.NET Core Web API 기본 프로젝트 구조

## Web API 한 장 요약

| 구분 | 핵심 |
|---|---|
| 목적 | View 없이 JSON(또는 XML) 응답을 제공하는 백엔드 API |
| 구성 | `Program.cs`(서비스/미들웨어) + `Controllers/` + 선택적 `Models/DTOs/Data/Services` |
| 컨트롤러 베이스 | `ControllerBase` + `[ApiController]` + 특성 라우팅 |
| 표준 응답 | `IActionResult` (`Ok/Created/NoContent/NotFound/Problem/...`) |
| 문서/테스트 | Swagger (OpenAPI) |
| 확장 | DI, EF Core, 인증/인가, CORS, 버전관리, 로깅, 필터/미들웨어 |

---

## 기본 프로젝트 생성과 구조

```bash
dotnet new webapi -n MyApiApp
```

```text
MyApiApp/
├─ Controllers/
│  └─ WeatherForecastController.cs
├─ Models/
│  └─ WeatherForecast.cs
├─ Program.cs
├─ appsettings.json
└─ Properties/
   └─ launchSettings.json
```

템플릿에는 개발 편의용 Swagger, 샘플 컨트롤러, 개발 환경 프로필이 포함된다.

---

## Program.cs — 서비스 등록과 미들웨어 파이프라인

```csharp
var builder = WebApplication.CreateBuilder(args);

// 3.1 서비스 등록
builder.Services.AddControllers()
    .ConfigureApiBehaviorOptions(opt =>
    {
        // 자동 400(BadRequest) 제공을 유지하되 세부 제어 가능
        // opt.SuppressModelStateInvalidFilter = true;
    });

builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

// CORS(예: 프론트엔드 https://localhost:5173 허용)
builder.Services.AddCors(opt =>
{
    opt.AddPolicy("FrontPolicy", p =>
        p.WithOrigins("https://localhost:5173")
         .AllowAnyHeader()
         .AllowAnyMethod());
});

// 예: EF Core/Repository/Service는 아래에서 추가
// builder.Services.AddDbContext<AppDbContext>(...);
// builder.Services.AddScoped<IProductRepository, EfProductRepository>();
// builder.Services.AddScoped<IProductService, ProductService>();

var app = builder.Build();

// 3.2 미들웨어 파이프라인
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

// 예외 → 표준 ProblemDetails로 변환하는 커스텀 미들웨어/필터를 둘 수 있음
// app.UseMiddleware<ExceptionHandlingMiddleware>();

app.UseHttpsRedirection();
app.UseCors("FrontPolicy");

app.UseAuthorization();

app.MapControllers();

app.Run();
```

포인트:
- **서비스 등록**: Controller, Swagger, CORS, EF Core, DI 구성
- **미들웨어 순서**: 예외 처리 → HTTPS → CORS → 인증/인가 → 라우팅/컨트롤러

---

## 컨트롤러 — `[ApiController]`와 특성 라우팅

### 기본 패턴

```csharp
using Microsoft.AspNetCore.Mvc;

[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    // 데모용 인-메모리 저장소
    private static readonly List<Product> _store = new();

    // GET /api/products
    [HttpGet]
    public ActionResult<IEnumerable<Product>> GetAll() => Ok(_store);

    // GET /api/products/5
    [HttpGet("{id:int}")]
    public ActionResult<Product> GetById(int id)
    {
        var p = _store.FirstOrDefault(x => x.Id == id);
        return p is null ? NotFound() : Ok(p);
    }

    // POST /api/products
    [HttpPost]
    public ActionResult<Product> Create(Product model)
    {
        model.Id = _store.Count == 0 ? 1 : _store.Max(x => x.Id) + 1;
        _store.Add(model);
        return CreatedAtAction(nameof(GetById), new { id = model.Id }, model);
    }

    // PUT /api/products/5
    [HttpPut("{id:int}")]
    public IActionResult Update(int id, Product model)
    {
        if (id != model.Id) return BadRequest();

        var existing = _store.FirstOrDefault(x => x.Id == id);
        if (existing is null) return NotFound();

        existing.Name = model.Name;
        existing.Price = model.Price;
        return NoContent();
    }

    // DELETE /api/products/5
    [HttpDelete("{id:int}")]
    public IActionResult Delete(int id)
    {
        var existing = _store.FirstOrDefault(x => x.Id == id);
        if (existing is null) return NotFound();
        _store.Remove(existing);
        return NoContent();
    }
}

public record Product(int Id, string Name, decimal Price);
```

특징:
- `[ApiController]`는 모델 검증 실패 시 자동 400 응답 등 **Web API 친화 기본값**을 제공
- `ActionResult<T>`를 사용하면 `Ok(T)`/`NotFound()` 혼용이 자연스러움
- `CreatedAtAction`은 **Location 헤더** 포함(REST 관례)

### 고급 라우팅: 다중 경로, 제약조건

```csharp
[HttpGet("by-code/{code:length(6)}")]
public ActionResult<Product> GetByCode(string code) { ... }

[HttpGet("search")]
public ActionResult<IEnumerable<Product>> Search([FromQuery] string keyword, [FromQuery] int page = 1) { ... }
```

---

## DTO와 검증(Data Annotations / FluentValidation)

엔터티를 직접 노출하지 않고 **DTO 분리**로 경계를 명확히 한다.

```csharp
public class ProductCreateRequest
{
    [Required, StringLength(80)]
    public string Name { get; set; } = string.Empty;

    [Range(0, 999999)]
    public decimal Price { get; set; }
}

public class ProductResponse
{
    public int Id { get; init; }
    public string Name { get; init; } = string.Empty;
    public decimal Price { get; init; }
}
```

컨트롤러에서 DTO를 수신/반환:

```csharp
[HttpPost]
public async Task<ActionResult<ProductResponse>> Create(ProductCreateRequest req)
{
    // ModelState 자동검증: [ApiController] 활성화 시 유효성 실패 → 400
    var newId = await _service.CreateAsync(req);
    var dto = await _service.GetByIdAsync(newId);
    return CreatedAtAction(nameof(GetById), new { id = dto.Id }, dto);
}
```

> FluentValidation 사용 시 `AddFluentValidationAutoValidation()`을 등록 후 독립적인 검증 규칙을 구성할 수 있다.

---

## Service/Repository/EF Core — 레이어드 구조

```
/Models
  Product.cs
/DTOs
  ProductCreateRequest.cs
  ProductResponse.cs
/Data
  AppDbContext.cs
  IProductRepository.cs
  EfProductRepository.cs
/Services
  IProductService.cs
  ProductService.cs
/Controllers
  ProductsController.cs
```

### EF Core 컨텍스트

```csharp
using Microsoft.EntityFrameworkCore;

public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> opt) : base(opt) { }
    public DbSet<Product> Products => Set<Product>();

    protected override void OnModelCreating(ModelBuilder mb)
    {
        mb.Entity<Product>(e =>
        {
            e.Property(p => p.Name).HasMaxLength(80).IsRequired();
            e.Property(p => p.Price).HasColumnType("decimal(18,2)");
        });
    }
}

public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public decimal Price { get; set; }
}
```

### Repository

```csharp
public interface IProductRepository
{
    Task<int> CreateAsync(Product p, CancellationToken ct);
    Task<Product?> GetByIdAsync(int id, CancellationToken ct);
    Task<List<Product>> GetPagedAsync(int page, int size, CancellationToken ct);
    Task UpdateAsync(Product p, CancellationToken ct);
    Task<bool> DeleteAsync(int id, CancellationToken ct);
}
```

```csharp
using Microsoft.EntityFrameworkCore;

public class EfProductRepository : IProductRepository
{
    private readonly AppDbContext _db;
    public EfProductRepository(AppDbContext db) => _db = db;

    public async Task<int> CreateAsync(Product p, CancellationToken ct)
    {
        _db.Products.Add(p);
        await _db.SaveChangesAsync(ct);
        return p.Id;
    }

    public Task<Product?> GetByIdAsync(int id, CancellationToken ct) =>
        _db.Products.AsNoTracking().FirstOrDefaultAsync(x => x.Id == id, ct);

    public Task<List<Product>> GetPagedAsync(int page, int size, CancellationToken ct) =>
        _db.Products.AsNoTracking()
           .OrderBy(x => x.Id).Skip((page - 1) * size).Take(size)
           .ToListAsync(ct);

    public async Task UpdateAsync(Product p, CancellationToken ct)
    {
        _db.Attach(p).State = EntityState.Modified;
        await _db.SaveChangesAsync(ct);
    }

    public async Task<bool> DeleteAsync(int id, CancellationToken ct)
    {
        var entity = await _db.Products.FindAsync([id], ct);
        if (entity is null) return false;
        _db.Products.Remove(entity);
        await _db.SaveChangesAsync(ct);
        return true;
    }
}
```

### Service 레이어

```csharp
public interface IProductService
{
    Task<int> CreateAsync(ProductCreateRequest req, CancellationToken ct = default);
    Task<ProductResponse?> GetByIdAsync(int id, CancellationToken ct = default);
    Task<IReadOnlyList<ProductResponse>> GetPagedAsync(int page, int size, CancellationToken ct = default);
    Task<bool> UpdateAsync(int id, ProductCreateRequest req, CancellationToken ct = default);
    Task<bool> DeleteAsync(int id, CancellationToken ct = default);
}
```

```csharp
public class ProductService : IProductService
{
    private readonly IProductRepository _repo;
    public ProductService(IProductRepository repo) => _repo = repo;

    public async Task<int> CreateAsync(ProductCreateRequest req, CancellationToken ct = default)
    {
        var entity = new Product { Name = req.Name, Price = req.Price };
        return await _repo.CreateAsync(entity, ct);
    }

    public async Task<ProductResponse?> GetByIdAsync(int id, CancellationToken ct = default)
    {
        var e = await _repo.GetByIdAsync(id, ct);
        return e is null ? null : new ProductResponse { Id = e.Id, Name = e.Name, Price = e.Price };
    }

    public async Task<IReadOnlyList<ProductResponse>> GetPagedAsync(int page, int size, CancellationToken ct = default)
    {
        var list = await _repo.GetPagedAsync(page, size, ct);
        return list.Select(e => new ProductResponse { Id = e.Id, Name = e.Name, Price = e.Price }).ToList();
    }

    public async Task<bool> UpdateAsync(int id, ProductCreateRequest req, CancellationToken ct = default)
    {
        var current = await _repo.GetByIdAsync(id, ct);
        if (current is null) return false;

        current.Name = req.Name;
        current.Price = req.Price;
        await _repo.UpdateAsync(current, ct);
        return true;
    }

    public Task<bool> DeleteAsync(int id, CancellationToken ct = default) => _repo.DeleteAsync(id, ct);
}
```

### Program.cs 등록

```csharp
builder.Services.AddDbContext<AppDbContext>(opt =>
    opt.UseSqlite(builder.Configuration.GetConnectionString("DefaultConnection")));

builder.Services.AddScoped<IProductRepository, EfProductRepository>();
builder.Services.AddScoped<IProductService, ProductService>();
```

---

## Controller ↔ Service ↔ DTO 연결 예제

```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private readonly IProductService _svc;
    public ProductsController(IProductService svc) => _svc = svc;

    [HttpGet]
    public async Task<ActionResult<IEnumerable<ProductResponse>>> GetPaged([FromQuery] int page = 1, [FromQuery] int size = 20)
    {
        var items = await _svc.GetPagedAsync(page, size, HttpContext.RequestAborted);
        return Ok(items);
    }

    [HttpGet("{id:int}")]
    public async Task<ActionResult<ProductResponse>> GetById(int id)
    {
        var dto = await _svc.GetByIdAsync(id, HttpContext.RequestAborted);
        return dto is null ? NotFound() : Ok(dto);
    }

    [HttpPost]
    public async Task<ActionResult<ProductResponse>> Create(ProductCreateRequest req)
    {
        var id = await _svc.CreateAsync(req, HttpContext.RequestAborted);
        var dto = await _svc.GetByIdAsync(id, HttpContext.RequestAborted);
        return CreatedAtAction(nameof(GetById), new { id }, dto);
    }

    [HttpPut("{id:int}")]
    public async Task<IActionResult> Update(int id, ProductCreateRequest req)
    {
        var ok = await _svc.UpdateAsync(id, req, HttpContext.RequestAborted);
        return ok ? NoContent() : NotFound();
    }

    [HttpDelete("{id:int}")]
    public async Task<IActionResult> Delete(int id)
    {
        var ok = await _svc.DeleteAsync(id, HttpContext.RequestAborted);
        return ok ? NoContent() : NotFound();
    }
}
```

---

## 표준 오류 응답 — ProblemDetails + 전역 예외 처리

### 전역 예외 미들웨어

```csharp
public class ExceptionHandlingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<ExceptionHandlingMiddleware> _logger;

    public ExceptionHandlingMiddleware(RequestDelegate next, ILogger<ExceptionHandlingMiddleware> logger)
    {
        _next = next; _logger = logger;
    }

    public async Task Invoke(HttpContext ctx)
    {
        try
        {
            await _next(ctx);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Unhandled exception");
            var problem = new ProblemDetails
            {
                Title = "An unexpected error occurred",
                Status = StatusCodes.Status500InternalServerError,
                Detail = appDetailed(ctx) ? ex.Message : "Internal Server Error",
                Instance = ctx.TraceIdentifier
            };
            ctx.Response.ContentType = "application/problem+json";
            ctx.Response.StatusCode = StatusCodes.Status500InternalServerError;
            await ctx.Response.WriteAsJsonAsync(problem);
        }
    }

    private static bool appDetailed(HttpContext ctx) =>
        ctx.RequestServices.GetRequiredService<IHostEnvironment>().IsDevelopment();
}
```

`Program.cs`에 등록:

```csharp
app.UseMiddleware<ExceptionHandlingMiddleware>();
```

### 자동 응답

`[ApiController]`가 활성화되면 ModelState 실패 시 자동으로 RFC 7807 형식의 `ValidationProblemDetails`가 반환된다.

---

## — 문서화/테스트

```csharp
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(opt =>
{
    opt.SwaggerDoc("v1", new() { Title = "My API", Version = "v1" });
    // JWT Bearer 보안정의, XML 주석 포함 등 확장 가능
});
```

```csharp
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI(c =>
    {
        c.SwaggerEndpoint("/swagger/v1/swagger.json", "My API v1");
        c.DocExpansion(Swashbuckle.AspNetCore.SwaggerUI.DocExpansion.List);
    });
}
```

접속: `/swagger`

---

## CORS — 프론트엔드 연동

```csharp
builder.Services.AddCors(opt =>
{
    opt.AddPolicy("FrontPolicy", p => p
        .WithOrigins("https://localhost:5173", "https://myapp.example")
        .AllowAnyMethod()
        .AllowAnyHeader()
        .AllowCredentials()); // 필요 시
});

app.UseCors("FrontPolicy");
```

SPA/모바일/외부 클라이언트에서 API 호출 시 **필수 설정**.

---

## 버전 관리 — URL/쿼리/헤더 기반

패키지: `Microsoft.AspNetCore.Mvc.Versioning`

```csharp
builder.Services.AddApiVersioning(opt =>
{
    opt.AssumeDefaultVersionWhenUnspecified = true;
    opt.DefaultApiVersion = new ApiVersion(1, 0);
    opt.ReportApiVersions = true;
});
```

컨트롤러:

```csharp
[ApiController]
[ApiVersion("1.0")]
[Route("api/v{version:apiVersion}/products")]
public class ProductsV1Controller : ControllerBase { ... }

[ApiController]
[ApiVersion("2.0")]
[Route("api/v{version:apiVersion}/products")]
public class ProductsV2Controller : ControllerBase { ... }
```

---

## 성능/확장 포인트

- **페이징/키셋 페이징**: `Skip/Take` vs `WHERE Id > @lastId ORDER BY Id LIMIT k`
- **AsNoTracking**: 읽기 전용 쿼리
- **Response Caching**: 변경 드문 GET에 캐시 헤더
- **CancellationToken**: 긴 쿼리/외부 호출 취소 가능
- **Rate Limiting**: .NET 8 `AddRateLimiter` 사용
- **압축**: `ResponseCompression` 패키지

예: 간단 RateLimiter

```csharp
builder.Services.AddRateLimiter(opt =>
{
    opt.GlobalLimiter = PartitionedRateLimiter.Create<HttpContext, string>(ctx =>
        RateLimitPartition.GetTokenBucketLimiter("global",
            _ => new TokenBucketRateLimiterOptions
            {
                TokenLimit = 100,
                QueueProcessingOrder = QueueProcessingOrder.OldestFirst,
                QueueLimit = 0,
                ReplenishmentPeriod = TimeSpan.FromSeconds(1),
                TokensPerPeriod = 100,
                AutoReplenishment = true
            }));
});

app.UseRateLimiter();
```

---

## 설정/환경 — appsettings.*와 연결 문자열

`appsettings.json` / `appsettings.Development.json` / `Environment Variables`의 **우선순위 병합**.
EF Core의 연결 문자열:

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Data Source=app.db"
  }
}
```

```csharp
builder.Services.AddDbContext<AppDbContext>(opt =>
    opt.UseSqlite(builder.Configuration.GetConnectionString("DefaultConnection")));
```

---

## 테스트 — 통합 테스트(WebApplicationFactory)

패키지: `Microsoft.AspNetCore.Mvc.Testing`

```csharp
public class ProductsApiTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;

    public ProductsApiTests(WebApplicationFactory<Program> factory)
        => _client = factory.CreateClient();

    [Fact]
    public async Task GetAll_ReturnsOk()
    {
        var res = await _client.GetAsync("/api/products");
        res.EnsureSuccessStatusCode();
        var json = await res.Content.ReadAsStringAsync();
        Assert.NotNull(json);
    }
}
```

> 단위 테스트는 Service/Repository를 모킹하여 로직을 분리 검증, 통합 테스트는 실제 파이프라인을 검증.

---

## 보안 — 인증/인가 준비

- **인증(Authentication)**: JWT Bearer(OpenIddict/IdentityServer/Azure AD B2C 등)
- **인가(Authorization)**: `[Authorize]`, 정책 기반
- Swagger 보안 스키마 추가 후 **Try it out**에서도 토큰 입력 가능

예시(요약):

```csharp
builder.Services.AddAuthentication("Bearer")
    .AddJwtBearer("Bearer", opt =>
    {
        opt.Authority = "https://issuer.example";
        opt.Audience = "myapi";
        opt.RequireHttpsMetadata = true;
    });

builder.Services.AddAuthorization(opt =>
{
    opt.AddPolicy("Products.Read", p => p.RequireClaim("scope", "products.read"));
});

app.UseAuthentication();
app.UseAuthorization();
```

컨트롤러:

```csharp
[Authorize(Policy = "Products.Read")]
[HttpGet]
public async Task<ActionResult<IEnumerable<ProductResponse>>> Get() { ... }
```

---

## 로깅 — 기본/서드파티(Serilog)

간단 구성:

```csharp
builder.Logging.ClearProviders();
builder.Logging.AddConsole();
// Serilog 사용 시: UseSerilog(), appsettings.json에 Sink/Enricher 구성
```

컨트롤러 주입:

```csharp
private readonly ILogger<ProductsController> _log;
public ProductsController(IProductService svc, ILogger<ProductsController> log)
{
    _svc = svc; _log = log;
}

_log.LogInformation("GetPaged called: page={Page}, size={Size}", page, size);
```

---

## 실전 예제: 예외→ProblemDetails, DTO 검증, 페이징, Swagger까지

### DTO

```csharp
public class PagedRequest
{
    [Range(1, 1000)]
    public int Page { get; set; } = 1;

    [Range(1, 200)]
    public int Size { get; set; } = 20;
}

public class PagedResponse<T>
{
    public int Page { get; init; }
    public int Size { get; init; }
    public int Total { get; init; }
    public IReadOnlyList<T> Items { get; init; } = Array.Empty<T>();
}
```

### 컨트롤러 일부

```csharp
[HttpGet("paged")]
public async Task<ActionResult<PagedResponse<ProductResponse>>> GetPaged([FromQuery] PagedRequest req)
{
    // ModelState 실패 시 자동 400 (ApiController)
    var items = await _svc.GetPagedAsync(req.Page, req.Size, HttpContext.RequestAborted);
    // 총합 계산(예: DB Count) 생략:
    var total = 1000; // 데모
    return Ok(new PagedResponse<ProductResponse>
    {
        Page = req.Page,
        Size = req.Size,
        Total = total,
        Items = items.ToList()
    });
}
```

응답은 클라이언트에서 페이지네이션 UI 구현에 용이한 형태가 된다.

---

## Minimal APIs와 비교(선택)

- **Minimal APIs**: `app.MapGet("/api/products", ...);` 같은 라우팅 중심, 경량
- **Controller 기반 Web API**: 필터/모델 바인딩/특성 라우팅/테스트/도구 호환성 우수
대규모 프로젝트나 팀 협업에서는 컨트롤러 기반이 여전히 주류.

---

## 배포/Docker(요약)

`Dockerfile` 예시:

```dockerfile
# build

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY . .
RUN dotnet publish -c Release -o /app

# runtime

FROM mcr.microsoft.com/dotnet/aspnet:8.0
WORKDIR /app
COPY --from=build /app .
EXPOSE 8080
ENV ASPNETCORE_URLS=http://+:8080
ENTRYPOINT ["dotnet", "MyApiApp.dll"]
```

---

## 자주 겪는 이슈와 해결

| 증상 | 원인 | 해결 |
|---|---|---|
| 415 Unsupported Media Type | `Content-Type` 누락/불일치 | `application/json` 헤더 확인 |
| CORS 오류 | 클라이언트 Origin 미허용 | `AddCors` 정책에 Origin 추가, `UseCors` 순서 확인 |
| 400 BadRequest(검증실패) | DTO 어노테이션 불일치 | `ModelState` 메시지 확인, `[ApiController]` 자동 반환 이해 |
| 500 오류 | 처리되지 않은 예외 | 전역 예외 미들웨어/필터로 ProblemDetails 변환 |
| JSON 직렬화 문제 | 순환참조/Nullable | 필요 시 `ReferenceHandler.IgnoreCycles`, `Required` 제어 |

---

## 체크리스트

- [ ] 컨트롤러에 `[ApiController]` + 특성 라우팅 적용
- [ ] DTO/엔터티 분리, 검증 어노테이션/FluentValidation
- [ ] 서비스/리포지토리/EF Core 구성, DI 등록
- [ ] 전역 예외 처리(ProblemDetails)
- [ ] Swagger 문서/보안 설정
- [ ] CORS, 버전 관리
- [ ] 로깅, Rate Limiting, Response Caching(필요 시)
- [ ] 통합 테스트(WebApplicationFactory)

---

## 결론

기본 프로젝트 골격에서 시작해 **레이어드 설계, DTO·검증, 표준 오류 응답, Swagger, CORS, 버전 관리, 테스트, 배포**까지 이어지면, 규모가 커져도 유지보수성이 높고 안정적인 Web API를 제공할 수 있다.
본 가이드를 스캐폴딩 삼아 팀 표준 템플릿으로 구체화하면 실무 생산성이 크게 향상된다.
