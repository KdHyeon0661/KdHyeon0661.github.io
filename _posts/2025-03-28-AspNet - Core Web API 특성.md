---
layout: post
title: AspNet - Core Web API 특성
date: 2025-03-28 19:20:23 +0900
category: AspNet
---
# ASP.NET Core Web API 특성(Attribute)

## 1. [ApiController] — 자동 바인딩/검증/오류 응답

### 1.1 핵심 효과

- **자동 모델 유효성 검사**: 액션 진입 전에 `ModelState.IsValid`가 false면 **자동 400 BadRequest**로 종료. (문제세부 ProblemDetails 응답)
- **바인딩 원천 추론**(Infer Binding Sources): 복합 타입은 Body, 원시/간단 타입은 Query/Route 에서 추론
- **필수 파라미터 누락 400**: 바인딩 실패 시 자동 400
- **문제 응답 형식**: RFC 7807 `application/problem+json` (오류 페이로드 일관화)

```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    [HttpPost]
    public IActionResult Create(ProductCreateDto dto)
    {
        // ModelState 검사 생략 가능(자동 수행)
        // 유효하지 않으면 400 + ProblemDetails 자동 반환
        var id = _service.Create(dto);
        return CreatedAtAction(nameof(GetById), new { id }, new { id });
    }

    [HttpGet("{id:int}")]
    public ActionResult<ProductReadDto> GetById(int id) =>
        _service.Find(id) is { } p ? Ok(p) : NotFound();
}
```

> `ApiBehaviorOptions`로 일부 동작(자동 400 등)을 세밀하게 끌 수 있다.

```csharp
builder.Services.AddControllers()
    .ConfigureApiBehaviorOptions(o =>
    {
        // 자동 400 응답 비활성화 (수동 제어용)
        o.SuppressModelStateInvalidFilter = true;
        // 바인딩 소스 추론 비활성화
        // o.SuppressInferBindingSourcesForParameters = true;
    });
```

---

## 2. HTTP 메서드 특성 — 액션 선택

```csharp
[HttpGet]                     // GET /api/products
[HttpGet("{id:int}")]         // GET /api/products/5
[HttpPost]                    // POST /api/products
[HttpPut("{id:int}")]         // PUT /api/products/5
[HttpPatch("{id:int}")]       // PATCH /api/products/5 (부분 수정: JSON Patch 등)
[HttpDelete("{id:int}")]      // DELETE /api/products/5
[AcceptVerbs("HEAD","OPTIONS","TRACE")] // 비표준/다중
```

### 2.1 PATCH 주의
- JSON Patch(`application/json-patch+json`) 사용 시 `Microsoft.AspNetCore.Mvc.NewtonsoftJson` 또는 System.Text.Json 기반 구현 필요.
- 동시성/검증 규칙 별도 고려.

---

## 3. [Route] / [Http…("template")] — 특성 라우팅

### 3.1 컨트롤러·액션 템플릿

```csharp
[Route("api/[controller]")]
public class OrdersController : ControllerBase
{
    [HttpGet]                         // GET /api/orders
    public IActionResult GetAll() => ...

    [HttpGet("{id:guid}")]            // GET /api/orders/{guid}
    public IActionResult Get(Guid id) => ...

    [HttpGet("by-customer/{customerId:int:min(1)}")]
    public IActionResult GetByCustomer(int customerId) => ...
}
```

- 토큰: `[controller]`, `[action]`
- 라우트 제약조건: `int`, `guid`, `min`, `max`, `length`, `regex(…)` 등
- 선택 매개변수: `{slug?}`

### 3.2 컨트롤러 상위 경로 + 액션 상대 경로

```csharp
[Route("api/inventory")]
public class InventoryController : ControllerBase
{
    [HttpGet("items")]                // GET /api/inventory/items
    public IActionResult Items() => ...

    [HttpGet("items/{sku}")]          // GET /api/inventory/items/ABC-001
    public IActionResult Item(string sku) => ...
}
```

### 3.3 이름 있는 라우트

```csharp
[HttpGet("{id:int}", Name = "orders.get")]
public IActionResult Get(int id) => ...

[HttpPost]
public IActionResult Create(OrderCreateDto dto)
{
    var id = _svc.Create(dto);
    return CreatedAtRoute("orders.get", new { id }, new { id });
}
```

---

## 4. 바인딩 소스 특성 — 데이터가 ‘어디서 오는가’

### 4.1 기본 규칙([ApiController] 활성 시)

- **복합 타입(클래스/레코드/DTO)** → Body에서 추론
- **단순 타입(int/string/Guid/DateTime 등)** → Route/Query에서 추론
- **경로 템플릿에 이름이 있으면 우선 Route**

> 불분명할 땐 **명시 특성**으로 혼동 제거.

### 4.2 대표 특성

| 특성 | 바인딩 위치 | 비고 |
|---|---|---|
| `[FromBody]` | Request Body(JSON) | **하나의 파라미터만** Body 바인딩 가능(혼동 방지) |
| `[FromQuery]` | `?key=value` | 페이징·검색 필터 |
| `[FromRoute]` | URL 경로 | `"{id}"`와 이름 일치 필요 |
| `[FromHeader]` | HTTP 헤더 | 상호작용 토큰, 상관ID |
| `[FromForm]` | multipart/form-data | 파일 업로드, 폼 필드 |
| `[FromServices]` | DI 컨테이너 | 액션 매개변수로 서비스 직접 주입 |
| `[Bind]` | 화이트리스트/블랙리스트 | 오버포스팅 방지(모델 수준) |

#### 4.2.1 혼합 바인딩 예시

```csharp
public record SearchFilter(int Page, int PageSize);

[HttpPost("search/{category}")]
public IActionResult Search(
    [FromRoute] string category,
    [FromQuery] SearchFilter filter,  // ?page=1&pageSize=20
    [FromBody] SearchBody body)       // { "keyword": "...", "ranges": [...] }
{
    ...
}
```

#### 4.2.2 헤더·서비스 바인딩

```csharp
[HttpGet]
public IActionResult GetAll(
    [FromHeader(Name = "X-Correlation-Id")] string correlationId,
    [FromServices] IClock clock)
{
    _log.WithCorrelation(correlationId).Info("List at {time}", clock.UtcNow());
    return Ok(_svc.List());
}
```

#### 4.2.3 오버포스팅 방지 — [Bind]

```csharp
public class UserUpdateDto
{
    public string DisplayName { get; set; } = "";
    public string? About { get; set; }
    public bool IsAdmin { get; set; } // <- 클라이언트가 바꾸면 안 되는 필드라고 가정
}

[HttpPut("{id:int}")]
public IActionResult Update(
    int id,
    [Bind(nameof(UserUpdateDto.DisplayName), nameof(UserUpdateDto.About))]
    UserUpdateDto dto)
{
    // IsAdmin은 무시됨(바인딩 제외)
    ...
    return NoContent();
}
```

> 보다 강력하게는 **입력 DTO에서 아예 제거**하거나 서버에서 매핑 시 화이트리스트만 매핑.

---

## 5. 모델 검증 — 자동/수동 제어

### 5.1 Data Annotations

```csharp
public class ProductCreateDto
{
    [Required, StringLength(100)]
    public string Name { get; set; } = "";

    [Range(0, 100000)]
    public decimal Price { get; set; }

    [RegularExpression(@"^[A-Z0-9\-]+$")]
    public string Sku { get; set; } = "";
}
```

### 5.2 자동 400 (ApiController)

```csharp
[HttpPost]
public IActionResult Create(ProductCreateDto dto)
{
    // 유효하지 않으면 자동 400 + ProblemDetails
    return Ok();
}
```

### 5.3 수동 제어(자동 400 해제 시)

```csharp
[HttpPost]
public IActionResult Create(ProductCreateDto dto)
{
    if (!ModelState.IsValid)
        return ValidationProblem(ModelState); // ProblemDetails(422 유사) or BadRequest(ModelState)
    return Ok();
}
```

---

## 6. 컨텐츠 협상/미디어 타입 — [Produces]/[Consumes]/[ProducesResponseType]

### 6.1 응답 포맷 명시

```csharp
[Produces("application/json")]       // 또는 컨트롤러 상단에 적용
[HttpGet("{id:int}")]
[ProducesResponseType(typeof(ProductReadDto), StatusCodes.Status200OK)]
[ProducesResponseType(StatusCodes.Status404NotFound)]
public IActionResult Get(int id) => ...
```

### 6.2 요청 포맷 제한

```csharp
[HttpPost]
[Consumes("application/json")]       // 이 액션은 JSON만 허용
public IActionResult Create([FromBody] ProductCreateDto dto) => ...
```

### 6.3 다양한 미디어 타입 반환

```csharp
[HttpGet("{id:int}")]
[Produces("application/json", "application/xml")] // XML 포맷터 추가 필요
public IActionResult Get(int id) => ...
```

### 6.4 포맷 필터([FormatFilter])

```csharp
[HttpGet("{id}.{format}")]  // /api/products/5.json 또는 5.xml
[FormatFilter]
public IActionResult Get(int id) => Ok(...);
```

> 실제 XML 응답을 원하면 XML 포맷팅 서비스 등록이 필요(`AddXmlSerializerFormatters()` 등).

---

## 7. 결과 형식·상태코드 — CreatedAt*, ProblemDetails

### 7.1 리소스 생성 패턴

```csharp
[HttpPost]
public IActionResult Create(ProductCreateDto dto)
{
    var id = _svc.Create(dto);
    return CreatedAtAction(nameof(GetById), new { id }, new { id });
}
```

### 7.2 ProblemDetails 직접 사용

```csharp
if (conflict)
{
    return Problem(
        title: "SKU already exists",
        detail: $"SKU '{dto.Sku}' is duplicated.",
        statusCode: StatusCodes.Status409Conflict);
}
```

> 자동 검증 실패 시에도 ProblemDetails 형식으로 400 응답.

---

## 8. 파일 업로드·요청 한도

### 8.1 단일/다중 파일 업로드

```csharp
[HttpPost("upload")]
public async Task<IActionResult> Upload(
    [FromForm] IFormFile file,
    [FromForm] string? description)
{
    if (file is null || file.Length == 0) return BadRequest("Empty file");

    using var stream = System.IO.File.Create(Path.Combine(_root, file.FileName));
    await file.CopyToAsync(stream);
    return Ok(new { file.FileName, file.Length, description });
}

[HttpPost("upload-multiple")]
public async Task<IActionResult> UploadMany([FromForm] List<IFormFile> files)
{
    foreach (var f in files)
    {
        // 저장…
    }
    return Ok(new { count = files.Count });
}
```

### 8.2 요청 크기 제한 조정

```csharp
[DisableRequestSizeLimit]                      // 무제한(주의)
[RequestSizeLimit(100_000_000)]               // 100MB
[RequestFormLimits(MultipartBodyLengthLimit = 200_000_000)]
[HttpPost("upload-large")]
public IActionResult UploadLarge([FromForm] IFormFile file) => Ok();
```

> 대용량은 **스트리밍**(PipeReader/Stream) + **안티바이러스/확장자 화이트리스트/임시 저장소 관리** 고려.

---

## 9. 보안·권한·캐싱

### 9.1 인증/인가

```csharp
[Authorize]                   // 컨트롤러/액션 단위
public class SecureController : ControllerBase
{
    [HttpGet("me")]
    public IActionResult Me() => Ok(User.Identity?.Name);

    [AllowAnonymous]
    [HttpGet("public")]
    public IActionResult Public() => Ok("Hello");
}
```

### 9.2 응답 캐싱 힌트

```csharp
[ResponseCache(Duration = 60, Location = ResponseCacheLocation.Any, NoStore = false)]
[HttpGet]
public IActionResult List() => Ok(_svc.List());
```

> 실제 캐시를 위해선 **ResponseCaching 미들웨어**(프록시/클라이언트 기준) 또는 **서버 캐시** 전략(메모리/분산) 병행.

---

## 10. API 탐색/Swagger — [ApiExplorerSettings]

```csharp
[ApiExplorerSettings(IgnoreApi = true)] // Swagger/ApiExplorer에서 숨김
[HttpGet("internal")]
public IActionResult InternalUse() => Ok();
```

```csharp
[ApiExplorerSettings(GroupName = "v1")]
[Route("api/v1/[controller]")]
public class V1ProductsController : ControllerBase { ... }
```

---

## 11. 버전 관리 — [ApiVersion]/[MapToApiVersion]

> `Microsoft.AspNetCore.Mvc.Versioning` 패키지 필요.

```csharp
builder.Services.AddApiVersioning(o =>
{
    o.DefaultApiVersion = new ApiVersion(1, 0);
    o.AssumeDefaultVersionWhenUnspecified = true;
    o.ReportApiVersions = true;
});
builder.Services.AddVersionedApiExplorer(o =>
{
    o.GroupNameFormat = "'v'VVV";
});
```

컨트롤러:

```csharp
[ApiController]
[ApiVersion("1.0")]
[ApiVersion("2.0")]
[Route("api/v{version:apiVersion}/products")]
public class ProductsController : ControllerBase
{
    [HttpGet]
    [MapToApiVersion("1.0")]
    public IActionResult GetV1() => Ok(new { v = "1.0" });

    [HttpGet]
    [MapToApiVersion("2.0")]
    public IActionResult GetV2() => Ok(new { v = "2.0" });
}
```

---

## 12. 액션/컨트롤러 제어 — [NonAction], [ActionName]

```csharp
[NonAction]
public void Helper() { } // 라우팅 대상 제외(내부 헬퍼)

[HttpPost("submit")]
[ActionName("create")]   // 라우팅/링크에서 'create'로 노출
public IActionResult CreateAlias([FromBody] OrderCreateDto dto) => ...
```

---

## 13. 고급 라우팅 — 링크 생성/영역/네임드 경로

### 13.1 링크 생성

```csharp
var url = Url.Link("orders.get", new { id = 123 }); // 이름 있는 라우트의 절대 URL
```

### 13.2 Area 패턴(관리 콘솔 등 분리)

```csharp
[Area("Admin")]
[Route("api/admin/[controller]")]
public class UsersController : ControllerBase { ... }
```

---

## 14. 커스텀 모델 바인더/필터

### 14.1 모델 바인더

```csharp
public class CsvIntArrayBinder : IModelBinder
{
    public Task BindModelAsync(ModelBindingContext ctx)
    {
        var raw = ctx.ValueProvider.GetValue(ctx.ModelName).FirstValue;
        if (string.IsNullOrWhiteSpace(raw)) { ctx.Result = ModelBindingResult.Success(Array.Empty<int>()); return Task.CompletedTask; }
        var parts = raw.Split(',');
        if (parts.All(p => int.TryParse(p, out _)))
            ctx.Result = ModelBindingResult.Success(parts.Select(int.Parse).ToArray());
        else
            ctx.ModelState.AddModelError(ctx.ModelName, "Invalid CSV");
        return Task.CompletedTask;
    }
}

public class FromCsvAttribute : ModelBinderAttribute
{
    public FromCsvAttribute() : base(typeof(CsvIntArrayBinder)) {}
}
```

사용:

```csharp
[HttpGet("bulk")]
public IActionResult BulkGet([FromCsv] int[] ids) => Ok(_svc.GetByIds(ids));
```

### 14.2 액션 필터로 공통 로직

```csharp
public class CorrelationIdFilter : IActionFilter
{
    public void OnActionExecuting(ActionExecutingContext context)
    {
        var cid = context.HttpContext.Request.Headers["X-Correlation-Id"].FirstOrDefault()
                  ?? Guid.NewGuid().ToString("N");
        context.HttpContext.Items["cid"] = cid;
    }
    public void OnActionExecuted(ActionExecutedContext context) { }
}

// 등록
builder.Services.AddControllers(o => o.Filters.Add<CorrelationIdFilter>());
```

---

## 15. 자주 겪는 함정과 체크리스트

| 증상 | 원인 | 해결 |
|---|---|---|
| Body 파라미터가 null | [FromBody] 여러 개 사용, 또는 Content-Type 미스매치 | **Body 파라미터는 1개**, `Content-Type: application/json` 확인 |
| 경로 변수 바인딩 실패 | `{id}` 이름 미일치 | `"{id}"`와 매개변수명 일치 또는 `[FromRoute(Name="...")]` |
| 자동 400이 싫다 | [ApiController] 기본 동작 | `SuppressModelStateInvalidFilter = true` 후 직접 `ValidationProblem` |
| 큰 파일 업로드 413 | 요청 크기 제한 | `[DisableRequestSizeLimit]`/`[RequestFormLimits]` + 리버스 프록시 설정 |
| Swagger에 내부 API 노출 | 탐색 활성화 | `[ApiExplorerSettings(IgnoreApi = true)]` |
| JSON Patch 적용 실패 | 포맷터 없음 | 패키지 및 입력 포맷터 구성 |
| 오버포스팅 | 엔티티 직접 바인딩 | **입력 DTO 분리**, [Bind], 서버 매핑 화이트리스트 |

**최소 체크리스트**
- [ ] 입력 DTO 분리(엔티티 직접 바인딩 금지)
- [ ] PRG/리소스 생성 시 `CreatedAtAction/Route` 반환
- [ ] [Produces]/[Consumes]/[ProducesResponseType] 문서화
- [ ] 파일 업로드 한도/보안/검증
- [ ] 버전 관리 정책 수립([ApiVersion])
- [ ] 자동 400 정책/오류 페이로드 일관화(ProblemDetails)
- [ ] Swagger 그룹/숨김/버전 노출 관리

---

## 전체 예제: 제품 API (종합)

```csharp
[ApiController]
[Route("api/v{version:apiVersion}/products")]
[ApiVersion("1.0")]
[Produces("application/json")]
public class ProductsController : ControllerBase
{
    private readonly IProductService _svc;
    public ProductsController(IProductService svc) => _svc = svc;

    // GET /api/v1/products?page=1&pageSize=20
    [HttpGet]
    [ProducesResponseType(typeof(Paged<ProductReadDto>), StatusCodes.Status200OK)]
    public IActionResult GetAll([FromQuery] int page = 1, [FromQuery] int pageSize = 20)
        => Ok(_svc.GetPaged(page, pageSize));

    // GET /api/v1/products/123
    [HttpGet("{id:int}", Name = "products.get")]
    [ProducesResponseType(typeof(ProductReadDto), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public IActionResult GetById(int id)
        => _svc.Find(id) is { } p ? Ok(p) : NotFound();

    // POST /api/v1/products
    [HttpPost]
    [Consumes("application/json")]
    [ProducesResponseType(typeof(object), StatusCodes.Status201Created)]
    [ProducesResponseType(StatusCodes.Status400BadRequest)]
    public IActionResult Create([FromBody] ProductCreateDto dto)
    {
        // 유효성 실패 시 자동 400
        var id = _svc.Create(dto);
        return CreatedAtRoute("products.get", new { id, version = "1.0" }, new { id });
    }

    // PUT /api/v1/products/123
    [HttpPut("{id:int}")]
    [ProducesResponseType(StatusCodes.Status204NoContent)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public IActionResult Update(int id, [FromBody] ProductUpdateDto dto)
    {
        if (!_svc.Exists(id)) return NotFound();
        _svc.Update(id, dto);
        return NoContent();
    }

    // PATCH /api/v1/products/123
    [HttpPatch("{id:int}")]
    [Consumes("application/json-patch+json", "application/json")]
    [ProducesResponseType(StatusCodes.Status204NoContent)]
    public IActionResult Patch(int id, [FromBody] JsonElement patchDoc)
    {
        // 예: DTO로 로드/머지/검증 수행 후 저장 처리
        _svc.ApplyPatch(id, patchDoc);
        return NoContent();
    }

    // DELETE /api/v1/products/123
    [HttpDelete("{id:int}")]
    [Authorize(Roles = "Admin")]
    [ProducesResponseType(StatusCodes.Status204NoContent)]
    public IActionResult Delete(int id)
    {
        _svc.Delete(id);
        return NoContent();
    }

    // POST /api/v1/products/import (CSV 업로드)
    [HttpPost("import")]
    [RequestSizeLimit(50_000_000)]
    [Consumes("multipart/form-data")]
    [ProducesResponseType(typeof(ImportResultDto), StatusCodes.Status200OK)]
    public async Task<IActionResult> Import([FromForm] IFormFile file)
    {
        var result = await _svc.ImportCsvAsync(file.OpenReadStream());
        return Ok(result);
    }
}
```

---

## 마무리

- **[ApiController]** 로 자동 바인딩·검증·문제응답을 기본기화하고,
- **바인딩 소스 특성**으로 데이터 출처를 명확히 하며,
- **라우팅·컨텐츠 협상·상태코드**를 특성으로 정확히 표현하면 **문서·테스트·유지보수**가 쉬워진다.
- 대규모 API에서는 **버전 관리·캐싱·권한·Swagger 노출 제어**를 특성으로 일관화하자.
