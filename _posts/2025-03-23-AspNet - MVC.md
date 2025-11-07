---
layout: post
title: AspNet - MVC
date: 2025-03-23 21:20:23 +0900
category: AspNet
---
# ASP.NET Core MVC의 Controller / Action / View 구조

## 0) MVC 한 장 요약

| 구성요소 | 핵심 책임 | 입력/출력 |
|---|---|---|
| Controller | 요청 조율, 도메인 호출, 결과 선택 | HttpContext → IActionResult |
| Action | 단일 유스케이스 처리 | Route/Query/Form/Body → View/Redirect/JSON/File |
| View | UI 렌더링(Razor) | Model → HTML |

요청 플로우:

```
브라우저 → [Routing] → Controller 선택 → Action 매개변수 바인딩 → 모델 검증 → 비즈니스 처리 → IActionResult 반환 → View 엔진 렌더링
```

---

## 1) 프로젝트 토폴로지

```
/Controllers
  └── ProductsController.cs
/Models
  └── Product.cs
/Views
  └── Shared/
      ├── _Layout.cshtml
      └── _ValidationScriptsPartial.cshtml
  └── Products/
      ├── Index.cshtml
      ├── Details.cshtml
      ├── Create.cshtml
      ├── Edit.cshtml
      └── Delete.cshtml
/Views/_ViewImports.cshtml
/Views/_ViewStart.cshtml
Program.cs
```

- `_ViewImports.cshtml`: 태그 헬퍼, 네임스페이스 임포트
- `_ViewStart.cshtml`: 모든 뷰의 Layout 지정

```razor
@* Views/_ViewImports.cshtml *@
@using MyApp
@using MyApp.Models
@addTagHelper *, Microsoft.AspNetCore.Mvc.TagHelpers
```

```razor
@* Views/_ViewStart.cshtml *@
@{
    Layout = "_Layout";
}
```

---

## 2) 모델, 리포지토리, DbContext (예시)

```csharp
// Models/Product.cs
using System.ComponentModel.DataAnnotations;

public class Product
{
    public int Id { get; set; }

    [Required, StringLength(80)]
    public string Name { get; set; } = string.Empty;

    [Range(0, 999999)]
    public decimal Price { get; set; }

    [StringLength(2000)]
    public string? Description { get; set; }
}
```

```csharp
// Data/AppDbContext.cs (EF Core)
using Microsoft.EntityFrameworkCore;

public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> opt) : base(opt) { }
    public DbSet<Product> Products => Set<Product>();
}
```

```csharp
// Data/IProductRepository.cs
public interface IProductRepository
{
    Task<List<Product>> GetAllAsync(CancellationToken ct);
    Task<Product?> GetByIdAsync(int id, CancellationToken ct);
    Task AddAsync(Product product, CancellationToken ct);
    Task UpdateAsync(Product product, CancellationToken ct);
    Task DeleteAsync(int id, CancellationToken ct);
}
```

```csharp
// Data/EfProductRepository.cs
using Microsoft.EntityFrameworkCore;

public class EfProductRepository : IProductRepository
{
    private readonly AppDbContext _db;
    public EfProductRepository(AppDbContext db) => _db = db;

    public Task<List<Product>> GetAllAsync(CancellationToken ct) =>
        _db.Products.AsNoTracking().OrderBy(p => p.Id).ToListAsync(ct);

    public Task<Product?> GetByIdAsync(int id, CancellationToken ct) =>
        _db.Products.AsNoTracking().FirstOrDefaultAsync(p => p.Id == id, ct);

    public async Task AddAsync(Product product, CancellationToken ct)
    {
        _db.Products.Add(product);
        await _db.SaveChangesAsync(ct);
    }

    public async Task UpdateAsync(Product product, CancellationToken ct)
    {
        _db.Attach(product).State = EntityState.Modified;
        await _db.SaveChangesAsync(ct);
    }

    public async Task DeleteAsync(int id, CancellationToken ct)
    {
        var entity = await _db.Products.FindAsync([id], ct);
        if (entity is null) return;
        _db.Products.Remove(entity);
        await _db.SaveChangesAsync(ct);
    }
}
```

```csharp
// Program.cs (필수 등록)
builder.Services.AddControllersWithViews();
builder.Services.AddDbContext<AppDbContext>(opt =>
    opt.UseSqlite(builder.Configuration.GetConnectionString("DefaultConnection")));
builder.Services.AddScoped<IProductRepository, EfProductRepository>();

app.MapControllerRoute(
    name: "default",
    pattern: "{controller=Home}/{action=Index}/{id?}");
```

---

## 3) Controller / Action — 표준 CRUD와 반환 형식

```csharp
// Controllers/ProductsController.cs
using Microsoft.AspNetCore.Mvc;

public class ProductsController : Controller
{
    private readonly IProductRepository _repo;

    public ProductsController(IProductRepository repo) => _repo = repo;

    // GET /Products
    public async Task<IActionResult> Index(CancellationToken ct)
    {
        var products = await _repo.GetAllAsync(ct);
        return View(products); // Views/Products/Index.cshtml
    }

    // GET /Products/Details/5
    public async Task<IActionResult> Details(int id, CancellationToken ct)
    {
        var product = await _repo.GetByIdAsync(id, ct);
        if (product is null) return NotFound();
        return View(product); // Views/Products/Details.cshtml
    }

    // GET /Products/Create
    public IActionResult Create() => View();

    // POST /Products/Create
    [HttpPost]
    [ValidateAntiForgeryToken]
    public async Task<IActionResult> Create(Product model, CancellationToken ct)
    {
        if (!ModelState.IsValid) return View(model);
        await _repo.AddAsync(model, ct);
        TempData["Message"] = "등록되었습니다.";
        return RedirectToAction(nameof(Index)); // PRG
    }

    // GET /Products/Edit/5
    public async Task<IActionResult> Edit(int id, CancellationToken ct)
    {
        var p = await _repo.GetByIdAsync(id, ct);
        if (p is null) return NotFound();
        return View(p);
    }

    // POST /Products/Edit/5
    [HttpPost]
    [ValidateAntiForgeryToken]
    public async Task<IActionResult> Edit(int id, Product model, CancellationToken ct)
    {
        if (id != model.Id) return BadRequest();
        if (!ModelState.IsValid) return View(model);
        await _repo.UpdateAsync(model, ct);
        TempData["Message"] = "수정되었습니다.";
        return RedirectToAction(nameof(Index));
    }

    // GET /Products/Delete/5
    public async Task<IActionResult> Delete(int id, CancellationToken ct)
    {
        var p = await _repo.GetByIdAsync(id, ct);
        if (p is null) return NotFound();
        return View(p);
    }

    // POST /Products/Delete/5
    [HttpPost, ActionName("Delete")]
    [ValidateAntiForgeryToken]
    public async Task<IActionResult> DeleteConfirmed(int id, CancellationToken ct)
    {
        await _repo.DeleteAsync(id, ct);
        TempData["Message"] = "삭제되었습니다.";
        return RedirectToAction(nameof(Index));
    }

    // 보너스: 다양한 반환 예제
    public IActionResult Ping() => Content("Pong");           // text/plain
    public IActionResult AsJson() => Json(new { ok = true }); // JSON
    public IActionResult AsFile() => File(new byte[0], "application/octet-stream", "empty.bin");
}
```

반환 형식 요약:

| 반환 | 설명 |
|---|---|
| `View(model)` | Razor View 렌더링 |
| `RedirectToAction(...)` | PRG/다른 액션 이동 |
| `NotFound()`, `BadRequest()` | 표준 상태코드 응답 |
| `Json(obj)` | JSON 직렬화 |
| `File(...)`, `PhysicalFile(...)` | 파일 다운로드/스트리밍 |

> `ActionResult<T>`를 사용하면 `return model;` 또는 `return NotFound();` 같은 패턴을 혼합할 수 있다(주로 API에서 유용).

---

## 4) Views — Layout/Partial/Tag Helper

### 4.1 레이아웃

```razor
@* Views/Shared/_Layout.cshtml *@
<!DOCTYPE html>
<html>
<head>
    <title>@ViewData["Title"] - MyApp</title>
    <link rel="stylesheet" href="~/css/site.css" />
</head>
<body>
<header>
    <a asp-controller="Home" asp-action="Index">Home</a>
    <a asp-controller="Products" asp-action="Index">Products</a>
</header>
<main class="container">
    @RenderBody()
</main>
<footer>
    @RenderSection("Footer", required: false)
</footer>
<script src="~/lib/jquery/jquery.min.js"></script>
@RenderSection("Scripts", required: false)
</body>
</html>
```

### 4.2 Index/Details

```razor
@* Views/Products/Index.cshtml *@
@model List<Product>
@{
    ViewData["Title"] = "Products";
}
<h2>Products</h2>

@if (TempData["Message"] is string msg)
{
    <div class="alert alert-success">@msg</div>
}

<p><a asp-action="Create" class="btn btn-primary">Create</a></p>

<table class="table">
    <thead><tr><th>Id</th><th>Name</th><th>Price</th><th></th></tr></thead>
    <tbody>
    @foreach (var p in Model)
    {
        <tr>
            <td>@p.Id</td>
            <td>@p.Name</td>
            <td>@p.Price.ToString("C")</td>
            <td>
                <a asp-action="Details" asp-route-id="@p.Id">Details</a> |
                <a asp-action="Edit" asp-route-id="@p.Id">Edit</a> |
                <a asp-action="Delete" asp-route-id="@p.Id">Delete</a>
            </td>
        </tr>
    }
    </tbody>
</table>
```

```razor
@* Views/Products/Details.cshtml *@
@model Product
@{ ViewData["Title"] = "Details"; }
<h2>@Model.Name</h2>
<dl>
  <dt>Price</dt><dd>@Model.Price.ToString("C")</dd>
  <dt>Description</dt><dd>@Model.Description</dd>
</dl>
<a asp-action="Index">Back</a>
```

### 4.3 Create/Edit/Delete

```razor
@* Views/Products/Create.cshtml *@
@model Product
@{ ViewData["Title"] = "Create"; }
<h2>Create</h2>

<form asp-action="Create" method="post">
    <div>
        <label asp-for="Name"></label>
        <input asp-for="Name" />
        <span asp-validation-for="Name"></span>
    </div>
    <div>
        <label asp-for="Price"></label>
        <input asp-for="Price" />
        <span asp-validation-for="Price"></span>
    </div>
    <div>
        <label asp-for="Description"></label>
        <textarea asp-for="Description"></textarea>
        <span asp-validation-for="Description"></span>
    </div>
    <button type="submit">Save</button>
</form>

@section Scripts {
    <partial name="_ValidationScriptsPartial" />
}
```

```razor
@* Views/Products/Edit.cshtml *@
@model Product
@{ ViewData["Title"] = "Edit"; }
<h2>Edit</h2>

<form asp-action="Edit" method="post">
    <input type="hidden" asp-for="Id" />
    <div>
        <label asp-for="Name"></label>
        <input asp-for="Name" />
        <span asp-validation-for="Name"></span>
    </div>
    <div>
        <label asp-for="Price"></label>
        <input asp-for="Price" />
        <span asp-validation-for="Price"></span>
    </div>
    <div>
        <label asp-for="Description"></label>
        <textarea asp-for="Description"></textarea>
        <span asp-validation-for="Description"></span>
    </div>
    <button type="submit">Update</button>
</form>

@section Scripts {
    <partial name="_ValidationScriptsPartial" />
}
```

```razor
@* Views/Products/Delete.cshtml *@
@model Product
@{ ViewData["Title"] = "Delete"; }
<h2>Delete</h2>
<p>정말로 삭제하시겠습니까?</p>
<dl>
  <dt>@Model.Name</dt><dd>@Model.Price.ToString("C")</dd>
</dl>
<form asp-action="Delete" method="post">
    <input type="hidden" asp-for="Id" />
    <button type="submit">Confirm</button>
    <a asp-action="Index">Cancel</a>
</form>
```

> `asp-` 접두사의 Tag Helper는 라우팅 안전성, name/id 자동 매핑, 검증 메시지 등 생산성을 크게 높인다.

---

## 5) 라우팅: 전통/특성(Attribute) 라우팅

### 5.1 전통 라우팅(Program.cs)

```csharp
app.MapControllerRoute(
    name: "default",
    pattern: "{controller=Home}/{action=Index}/{id?}");
```

### 5.2 특성 라우팅

```csharp
[Route("products")]
public class ProductsController : Controller
{
    [HttpGet("")]
    public IActionResult Index() => View();

    [HttpGet("details/{id:int}")]
    public IActionResult Details(int id) => View();

    [HttpPost("create")]
    [ValidateAntiForgeryToken]
    public IActionResult Create(Product model) { ... }
}
```

- 제약조건: `{id:int}`, `{slug:alpha}`, `{code:length(6)}`
- 여러 라우트: `[HttpGet("notice")]`, `[HttpGet("공지사항")]`

---

## 6) 모델 바인딩과 검증

- **소스**: Route, QueryString, Form, Headers, Files
- **검증**: `DataAnnotations` + `ModelState.IsValid`

```csharp
public IActionResult Search([FromQuery] string keyword, [FromQuery] int page = 1)
{
    if (string.IsNullOrWhiteSpace(keyword))
        ModelState.AddModelError(nameof(keyword), "검색어는 필수입니다.");

    if (!ModelState.IsValid) return View(); // 오류 출력

    // 처리
    return View();
}
```

고급:

- `TryUpdateModelAsync(model, prefix)`로 선택적 업데이트
- 커스텀 `ValidationAttribute` 또는 `IModelBinder` 확장
- 숫자/통화/날짜 로캘은 `RequestLocalization`과 함께 고려

---

## 7) 보안과 PRG 패턴

- 폼 POST에는 **반드시** `[ValidateAntiForgeryToken]`
- 민감 데이터 로깅 금지, 서버 검증이 최종 방어선
- PRG(Post-Redirect-Get)로 새로고침 중복 제출 방지

```csharp
[HttpPost]
[ValidateAntiForgeryToken]
public IActionResult Create(Product model)
{
    if (!ModelState.IsValid) return View(model);
    // 저장
    TempData["Message"] = "등록 완료";
    return RedirectToAction(nameof(Index)); // PRG
}
```

---

## 8) View 구성 고급: Partial View · ViewComponent

### 8.1 Partial View

```razor
@* Views/Shared/_ProductRow.cshtml *@
@model Product
<tr>
  <td>@Model.Id</td><td>@Model.Name</td><td>@Model.Price.ToString("C")</td>
</tr>
```

```razor
@* 사용 *@
<table>
  <tbody>
    @foreach (var p in Model)
    {
        <partial name="_ProductRow" model="p" />
    }
  </tbody>
</table>
```

### 8.2 View Component (데이터 포함 위젯)

```csharp
// Components/TopProductsViewComponent.cs
using Microsoft.AspNetCore.Mvc;

public class TopProductsViewComponent : ViewComponent
{
    private readonly IProductRepository _repo;
    public TopProductsViewComponent(IProductRepository repo) => _repo = repo;

    public async Task<IViewComponentResult> InvokeAsync(int take = 5, CancellationToken ct = default)
    {
        var list = (await _repo.GetAllAsync(ct))
            .OrderByDescending(p => p.Price).Take(take).ToList();
        return View(list); // Views/Shared/Components/TopProducts/Default.cshtml
    }
}
```

```razor
@* Views/Shared/Components/TopProducts/Default.cshtml *@
@model List<Product>
<ul>
@foreach (var p in Model) { <li>@p.Name - @p.Price.ToString("C")</li> }
</ul>
```

```razor
@* 어떤 View에서도 *@
@await Component.InvokeAsync("TopProducts", new { take = 3 })
```

---

## 9) Filters — 횡단 관심사

- `IActionFilter`, `IAsyncActionFilter`, `IResultFilter` 등
- 로깅, 권한 체크, 트랜잭션 경계 설정에 유용

```csharp
public class LogActionFilter : IActionFilter
{
    public void OnActionExecuting(ActionExecutingContext context)
        => Console.WriteLine($"→ {context.ActionDescriptor.DisplayName}");

    public void OnActionExecuted(ActionExecutedContext context)
        => Console.WriteLine($"← {context.ActionDescriptor.DisplayName}");
}

// 등록(글로벌)
builder.Services.AddControllersWithViews(opt =>
    opt.Filters.Add<LogActionFilter>());
```

또는 컨트롤러/액션에 `[ServiceFilter(typeof(LogActionFilter))]`.

---

## 10) Areas — 대규모 모듈 분리

```
/Areas/Admin
  /Controllers
    └── UsersController.cs
  /Views
    └── Users/Index.cshtml
  /Views/_ViewImports.cshtml
```

```csharp
// Program.cs
app.MapControllerRoute(
    name: "areas",
    pattern: "{area:exists}/{controller=Home}/{action=Index}/{id?}");
```

```csharp
// Areas/Admin/Controllers/UsersController.cs
[Area("Admin")]
public class UsersController : Controller
{
    public IActionResult Index() => View();
}
```

링크:

```razor
<a asp-area="Admin" asp-controller="Users" asp-action="Index">관리자</a>
```

---

## 11) 성능·안정성 팁

- 컨트롤러 액션은 **비동기** 기본 (`Task<IActionResult>`)
- 읽기 시 `AsNoTracking()`
- 페이징은 반드시 `Skip/Take` + **인덱스** 고려
- 긴 작업에는 `CancellationToken` 전달
- 에러 응답은 일관된 포맷(뷰/페이지/ProblemDetails) 유지

---

## 12) 테스트 — Controller 단위/통합

### 12.1 단위 테스트(비즈니스 로직 분리 가정)

```csharp
// xUnit 예시
[Fact]
public async Task Details_NotFound_WhenMissing()
{
    var repo = Substitute.For<IProductRepository>();
    repo.GetByIdAsync(Arg.Any<int>(), Arg.Any<CancellationToken>())
        .Returns((Product?)null);
    var sut = new ProductsController(repo);

    var result = await sut.Details(123, default);

    Assert.IsType<NotFoundResult>(result);
}
```

### 12.2 통합 테스트(WebApplicationFactory)

- `Microsoft.AspNetCore.Mvc.Testing` 패키지로 엔드투엔드 검증
- Razor 렌더링, 라우팅, 필터까지 포함해 테스트

---

## 13) 에러 처리/로그/사용자 메시지

- 사용자 메시지: `TempData` (PRG와 궁합)
- 서버 로그: `ILogger<T>`
- 에러 페이지: `UseExceptionHandler("/Home/Error")`, `UseStatusCodePagesWithReExecute("/Error/{0}")`

---

## 14) 자주 겪는 이슈와 해결

| 증상 | 원인 | 해결 |
|---|---|---|
| POST가 400/419 | Anti-forgery 토큰 누락 | `<form method="post">` + `@Html.AntiForgeryToken()` 또는 자동 Tag Helper, `[ValidateAntiForgeryToken]` |
| ModelState 항상 Invalid | 데이터 주석/형식 불일치 | `asp-for` 사용, 로캘/소수점 구분자 설정, ValidationSummary 확인 |
| 뷰 경로 못 찾음 | 폴더/파일 명명 규칙 위반 | `Views/{Controller}/{Action}.cshtml` 구조, `View("경로")`로 명시 |
| 링크가 틀린 곳으로 이동 | 라우트 값/영문 대소문자, 특성 라우팅 충돌 | `asp-controller/asp-action/asp-route-*` 사용, 특성 라우트 일관화 |
| 중복 제출 | F5 새로고침 | PRG 사용, 버튼 비활성화 UX |

---

## 15) 확장 주제 로드맵

- MVC + API 공존 시 라우팅 전략(엔드포인트 라우팅)
- 국제화(Localization)와 뷰 지역화
- Policy 기반 권한 `[Authorize(Policy="...")]`
- 파일 업로드/대용량 스트리밍
- ViewComponent 캐싱, ResponseCaching

---

## 부록) 간단 수식으로 보는 페이지네이션 비용 직관

페이지 크기 \( k \), 총 레코드 \( N \), 원하는 페이지 \( p \)에서  
`Skip((p-1) * k).Take(k)`의 **논리적 건너뛰기 비용**은 대략

$$
C_{\text{scan}} \approx \mathcal{O}((p-1) \cdot k)
$$

인덱스/키셋 페이지네이션(마지막 키 기준 `WHERE Id > @lastId ORDER BY Id LIMIT k`)을 쓰면

$$
C_{\text{scan}} \approx \mathcal{O}(k)
$$

으로 줄어드는 직관을 얻을 수 있다.

---

# 결론

초안의 MVC 개요를 토대로 **컨트롤러/액션/뷰의 표준 CRUD**, **라우팅(전통·특성)**,  
**모델 바인딩·검증**, **레이아웃/부분뷰/뷰 컴포넌트**, **필터/보안/PRG/Areas**,  
**테스트/운영 팁**까지 실무에 필요한 중심 축을 모두 연결했다.  
이 뼈대를 기준으로 규모가 커져도 **구조는 단순**, **관심사는 분리**, **UX는 안정**적으로 유지할 수 있다.