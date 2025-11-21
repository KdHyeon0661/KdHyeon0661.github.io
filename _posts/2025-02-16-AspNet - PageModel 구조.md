---
layout: post
title: AspNet - PageModel 구조
date: 2025-02-16 16:20:23 +0900
category: AspNet
---
# Razor Pages의 PageModel 구조

## Razor Pages의 구조 핵심: View와 PageModel의 1:1 쌍

```
Pages/
├── Index.cshtml         # 뷰(HTML + Razor)
└── Index.cshtml.cs      # PageModel(백엔드 로직)
```

- `.cshtml`은 화면/마크업과 **경량 표기**를 담당
- `.cshtml.cs`의 **PageModel**은 입력 바인딩, 검증, 핸들러(액션), 결과 반환을 담당
- MVC의 `Controller` + `Action`에 해당하는 부분이 **PageModel의 핸들러(OnGet/OnPost/OnPostX 등)**

---

## PageModel 클래스의 기본 형태와 생애주기

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.RazorPages;

public class IndexModel : PageModel
{
    // 1) 페이지에서 사용할 데이터(뷰 모델)
    public string Message { get; private set; } = "";

    // 2) GET 요청 핸들러
    public void OnGet()
    {
        Message = "페이지 처음 진입 시 호출됨 (GET)";
    }

    // 3) POST 요청 핸들러(폼 제출)
    public void OnPost()
    {
        Message = "폼 제출 시 호출됨 (POST)";
    }
}
```

핸들러 표준 시그니처:
- **반환형**: `void`, `IActionResult`, `Task`, `Task<IActionResult>`
- **이름 규칙**: `On{HttpVerb}` 또는 `On{HttpVerb}{Handler}`
  예) `OnGet`, `OnGetAsync`, `OnPostSave`, `OnPostDeleteAsync`

핸들러 실행 흐름(요약):
1. **모델 바인딩** → 2. **검증(ModelState)** → 3. **핸들러 실행** → 4. **결과 반환**
필터가 존재하면 1~4 사이에 **필터 인터셉트**가 발생

---

## Razor Page와 PageModel 연결

`Index.cshtml`
```cshtml
@page
@model IndexModel

<h2>@Model.Message</h2>

<form method="post">
    <button type="submit">Submit</button>
</form>
```

연결 요점:
- `@page`가 **라우팅 엔드포인트**가 됨
- `@model IndexModel`로 **해당 .cshtml.cs 클래스**를 지정
- 뷰에서 `Model.속성`으로 PageModel 데이터 접근

---

## 폼 제출과 데이터 바인딩: `[BindProperty]`와 SupportsGet

PageModel
```csharp
using System.ComponentModel.DataAnnotations;

public class ContactModel : PageModel
{
    // POST 뿐 아니라 쿼리스트링 GET 바인딩까지 허용하려면 SupportsGet = true
    [BindProperty(SupportsGet = true)]
    [Required, StringLength(50)]
    public string Name { get; set; } = "";

    public void OnGet()
    {
        // /Contact?Name=홍길동 → Name 자동 바인딩
    }

    public IActionResult OnPost()
    {
        if (!ModelState.IsValid) return Page();
        // Name 값 사용
        TempData["flash"] = $"Hello, {Name}";
        return RedirectToPage("Result");
    }
}
```

뷰
```cshtml
@page
@model ContactModel

<form method="post">
    <input asp-for="Name" />
    <span asp-validation-for="Name" class="text-danger"></span>
    <button type="submit">제출</button>
</form>

@section Scripts {
    <partial name="_ValidationScriptsPartial" />
}
```

핵심:
- `[BindProperty]`로 **Form → 속성** 자동 매핑
- `SupportsGet = true`로 **GET 쿼리 바인딩** 허용
- DataAnnotations로 **서버/클라이언트 검증** 병행(2편 내용)

---

## 여러 핸들러: `asp-page-handler`와 `OnPostX` 패턴

PageModel
```csharp
public class EditModel : PageModel
{
    [BindProperty] public int Id { get; set; }
    [BindProperty] public string Title { get; set; } = "";

    public void OnGet(int id)
    {
        // 초기 로드
        Id = id;
        Title = $"Item #{id}";
    }

    public IActionResult OnPostSave()
    {
        if (!ModelState.IsValid) return Page();
        // 저장 로직
        TempData["flash"] = "저장 완료";
        return RedirectToPage("Index");
    }

    public IActionResult OnPostDelete()
    {
        // 삭제 로직
        TempData["flash"] = "삭제 완료";
        return RedirectToPage("Index");
    }
}
```

뷰
```cshtml
@page "{id:int}"
@model EditModel

<form method="post" asp-page-handler="Save">
    <input asp-for="Id" type="hidden" />
    <input asp-for="Title" />
    <button type="submit">저장</button>
</form>

<form method="post" asp-page-handler="Delete">
    <input asp-for="Id" type="hidden" />
    <button type="submit">삭제</button>
</form>
```

핵심:
- `asp-page-handler="Save"` → `OnPostSave()`
- `asp-page-handler="Delete"` → `OnPostDelete()`
- 한 페이지에서 **여러 동작**을 명확히 분기

---

## 라우팅 템플릿과 매개변수 바인딩

`@page` 템플릿
```cshtml
@page "{id:int:min(1)}"
@model DetailsModel
<h1>상세 @Model.Id</h1>
```

PageModel
```csharp
public class DetailsModel : PageModel
{
    public int Id { get; private set; }

    public IActionResult OnGet(int id)
    {
        if (id < 1) return NotFound();
        Id = id;
        return Page();
    }
}
```

추가 팁:
- 라우트 제약: `int`, `guid`, `min`, `max`, `alpha` 등
- 복합 라우트도 지원: `@page "{category}/{slug}"`

---

## PageModel의 환경과 기본 제공 멤버

PageModel에서 자주 쓰는 멤버:
- `HttpContext`, `Request`, `Response`
- `RouteData`, `PageContext`, `ModelState`
- `Url`, `LinkGenerator`
- `TempData`(Provider: Cookie/Session)
- `User`(인증/권한)

예시:
```csharp
public IActionResult OnGet()
{
    if (!User.Identity?.IsAuthenticated ?? false) return Challenge();
    Response.Headers["X-Page"] = "Index";
    return Page();
}
```

---

## 등

핸들러에서 자주 쓰는 결과:
```csharp
return Page();                         // 현재 페이지 렌더링
return RedirectToPage("Index");        // Razor Page로 리다이렉트
return Redirect("/some/url");          // 절대/상대 URL
return LocalRedirect("~/local");       // 로컬 URL만 허용
return NotFound();                     // 404
return Forbid();                       // 403
return Challenge();                    // 인증 흐름 트리거
return File(bytes, "application/pdf", "x.pdf"); // 파일 반환
return Content("text");                // 단순 텍스트
return Json(new { ok = true });        // JSON 응답(컨트롤러 스타일)
```

PRG(Post-Redirect-Get) 예:
```csharp
public IActionResult OnPost()
{
    if (!ModelState.IsValid) return Page();
    TempData["flash"] = "완료";
    return RedirectToPage("Result");
}
```

---

## 모델 바인딩 심화: 컬렉션/중첩/화이트리스트/TryUpdateModelAsync

DTO와 화이트리스트 바인딩:
```csharp
public class ProductEditDto
{
    [Required] public string Name { get; set; } = "";
    [Range(0, int.MaxValue)] public int Price { get; set; }
}

public async Task<IActionResult> OnPostAsync(int id)
{
    var entity = await _db.Products.FindAsync(id);
    if (entity is null) return NotFound();

    // "Form" 접두사의 필드 중에서 지정한 속성만 바인딩
    var ok = await TryUpdateModelAsync(entity, "Form",
        e => e.Name, e => e.Price);
    if (!ok) return Page();

    await _db.SaveChangesAsync();
    return RedirectToPage("Index");
}
```

핵심:
- **과바인딩 방지**: 수정 불가 필드를 DTO 분리 또는 화이트리스트로 제한
- 컬렉션 인덱싱: `Form.Addresses[i].Line1` (2편 참고)

---

## 유효성 검증 통합: DataAnnotations, IValidatableObject, 커스텀 Attribute

DataAnnotations:
```csharp
public class RegisterForm
{
    [Required, EmailAddress]
    public string Email { get; set; } = "";

    [Required, StringLength(100, MinimumLength = 8)]
    public string Password { get; set; } = "";
}
```

객체 단위 검증:
```csharp
public class RegisterForm : IValidatableObject
{
    public string Password { get; set; } = "";
    public string Confirm  { get; set; } = "";

    public IEnumerable<ValidationResult> Validate(ValidationContext context)
    {
        if (Password != Confirm)
            yield return new ValidationResult("비밀번호가 일치하지 않습니다.",
                new[] { nameof(Confirm) });
    }
}
```

커스텀 Attribute:
```csharp
public sealed class NotDisposableEmailAttribute : ValidationAttribute
{
    public override bool IsValid(object? value)
        => value is not string s || !s.EndsWith("@examplemail.xyz", StringComparison.OrdinalIgnoreCase);
}
```

---

## 보안: CSRF, XSS, 인증/인가, Anti-forgery, 파일 업로드

- **CSRF**: Razor Pages의 `<form method="post">`는 기본적으로 **안티포저리 토큰** 포함
- **XSS**: Razor는 **기본 HTML 인코딩**. `Html.Raw`는 신뢰 콘텐츠만
- **인증/인가**: PageModel에 `[Authorize]` 또는 페이지 폴더별 정책(Startup/Program에서 규칙)
- **파일 업로드**: 확장자 화이트리스트, 크기 제한, 랜덤 파일명, 외부 스토리지 권장

페이지별 권한 부여:
```csharp
using Microsoft.AspNetCore.Authorization;

[Authorize(Roles = "Admin")]
public class AdminOnlyModel : PageModel
{
    public void OnGet() { }
}
```

폴더 전체 정책(Program.cs):
```csharp
builder.Services.AddRazorPages(options =>
{
    options.Conventions.AuthorizeFolder("/Admin", "AdminPolicy");
    options.Conventions.AllowAnonymousToPage("/Account/Login");
});
```

---

## 페이지 필터: IPageFilter / IAsyncPageFilter로 횡단 관심사 처리

필터 구현:
```csharp
using Microsoft.AspNetCore.Mvc.Filters;
using Microsoft.Extensions.Logging;

public class TimingFilter : IAsyncPageFilter
{
    private readonly ILogger<TimingFilter> _log;
    public TimingFilter(ILogger<TimingFilter> log) => _log = log;

    public Task OnPageHandlerSelectionAsync(PageHandlerSelectedContext ctx) => Task.CompletedTask;

    public async Task OnPageHandlerExecutionAsync(PageHandlerExecutingContext ctx, PageHandlerExecutionDelegate next)
    {
        var sw = System.Diagnostics.Stopwatch.StartNew();
        var result = await next();
        sw.Stop();
        _log.LogInformation("Page {Page} took {Elapsed} ms", ctx.ActionDescriptor.ViewEnginePath, sw.ElapsedMilliseconds);
    }
}
```

적용(의존성 주입):
```csharp
builder.Services.AddRazorPages()
    .AddMvcOptions(o => o.Filters.Add<TimingFilter>());
```

필터 용도:
- 공통 로깅/메트릭/권한/헤더/모델 보정 등

---

## TempData, ViewData, [TempData] 바인딩

TempData 사용:
```csharp
public IActionResult OnPost()
{
    TempData["Msg"] = "작업이 완료되었습니다.";
    return RedirectToPage("Result");
}
```

PageModel 속성에 바인딩:
```csharp
using Microsoft.AspNetCore.Mvc.ViewFeatures;

public class ResultModel : PageModel
{
    [TempData] public string? Msg { get; set; }

    public void OnGet() { /* Msg 자동 로드 */ }
}
```

---

## 비동기와 데이터 접근 패턴

```csharp
public class ProductsModel : PageModel
{
    private readonly AppDb _db;
    public ProductsModel(AppDb db) => _db = db;

    public IReadOnlyList<Product> Items { get; private set; } = Array.Empty<Product>();

    public async Task OnGetAsync()
    {
        Items = await _db.Products.AsNoTracking()
                                  .OrderBy(p => p.Id)
                                  .ToListAsync();
    }
}
```

팁:
- 읽기 전용 쿼리는 **AsNoTracking**
- 2개 이상 쿼리를 병렬 수행하지 말고 **적절히 합치거나 Cache** 고려

---

## DI(의존성 주입)와 서비스 계층

Program.cs
```csharp
builder.Services.AddScoped<IClock, SystemClock>();
builder.Services.AddScoped<IProductService, ProductService>();
```

PageModel
```csharp
public class DashboardModel : PageModel
{
    private readonly IClock _clock;
    private readonly IProductService _svc;

    public DashboardModel(IClock clock, IProductService svc)
    { _clock = clock; _svc = svc; }

    public DateTime UtcNow => _clock.UtcNow;

    public async Task OnGetAsync() => await _svc.WarmupAsync();
}
```

분리 이유:
- 테스트 용이성
- 재사용 가능 서비스 계층 정립
- PageModel은 **웹 바운더리** 로직 집중

---

## 고급 라우팅/링크: `asp-page`, `asp-route-*`, `LinkGenerator`

뷰
```cshtml
<a asp-page="/Products/Details" asp-route-id="42">상세</a>
```

PageModel에서 링크 생성:
```csharp
public class LinksModel : PageModel
{
    private readonly LinkGenerator _link;
    public LinksModel(LinkGenerator link) => _link = link;

    public string? DetailsUrl { get; private set; }

    public void OnGet()
    {
        DetailsUrl = _link.GetPathByPage(HttpContext, "/Products/Details", values: new { id = 42 });
    }
}
```

---

## 부분 페이지(Partial), ViewComponent와 PageModel 협업

- **Partial**: 단순 UI 조각 재사용
- **ViewComponent**: 데이터+뷰 캡슐화(미니 컴포넌트), PageModel에서 데이터 준비 부담 축소

PageModel:
```csharp
public class ListModel : PageModel
{
    private readonly IProductService _svc;
    public IReadOnlyList<Product> Items { get; private set; } = Array.Empty<Product>();

    public ListModel(IProductService svc) => _svc = svc;

    public async Task OnGetAsync() => Items = await _svc.GetTopAsync(10);
}
```

뷰:
```cshtml
@await Component.InvokeAsync("ProductSummary", new { items = Model.Items })
```

---

## 다국어/문화권과 PageModel

- 날짜·숫자 파싱은 **CurrentCulture**에 의존
- PageModel에서 문화권을 로깅/검사해 불일치를 조기 탐지
- 뷰 로컬라이저 `IViewLocalizer`, 데이터 로컬라이저 `IStringLocalizer` 사용

---

## 테스트 전략: PageModel 단위 테스트의 요령

- **PageModel**은 단순 C# 클래스이므로 **컨트롤러 테스트와 유사**
- DI로 주입되는 서비스·리포지토리를 **Mock**
- 반환형/ModelState/리다이렉트 대상/TempData를 검증

간단 예:
```csharp
[Fact]
public async Task Post_WhenInvalid_ReturnsPage()
{
    var svc = Substitute.For<IProductService>();
    var model = new EditModel(svc);
    model.ModelState.AddModelError("Title", "required");

    var result = await model.OnPostSaveAsync();

    result.Should().BeOfType<PageResult>();
}
```

---

## 에러 처리와 사용자 친화적 메시지

- 공통 예외 처리: `UseExceptionHandler("/Error")` + `Error.cshtml`
- PageModel에서 예외 포착 시 `ModelState.AddModelError("", "메시지")`로 사용자에 안전한 안내
- 민감 정보는 **로그**에만 남기고 UI로 노출 금지

---

## 체크리스트 요약

1) 핸들러 명명 규칙(HTTP + 선택적 Handler) 준수: `OnGet`, `OnPostSave`
2) `[BindProperty]`와 `SupportsGet` 사용 구분
3) DTO/화이트리스트 바인딩으로 **과바인딩 방지**
4) PRG 패턴으로 **새로고침 중복 제출 방지**
5) **안티포저리 토큰**(기본)과 검증 메시지 UI
6) 폴더/페이지 단위 **Authorize/AllowAnonymous** 및 정책
7) 공통 관심사는 **필터(IAsyncPageFilter)**로 추상화
8) 읽기 쿼리는 **AsNoTracking**, 링크는 **asp-page/LinkGenerator**
9) TempData는 **1회성 메시지**에만 사용
10) PageModel은 **웹 경계** 로직으로 한정, **도메인/인프라 로직은 서비스로 분리**

---

## 부록 A. 종합 예제: CRUD 한 페이지에 묶기

`Pages/Products/Edit.cshtml`
```cshtml
@page "{id:int}"
@model EditModel
@{
    ViewData["Title"] = "제품 편집";
}
<h1>@ViewData["Title"]</h1>

<div asp-validation-summary="All" class="text-danger"></div>

<form method="post" asp-page-handler="Save">
    <input asp-for="Form.Id" type="hidden" />
    <div>
        <label asp-for="Form.Name"></label>
        <input asp-for="Form.Name" />
        <span asp-validation-for="Form.Name" class="text-danger"></span>
    </div>
    <div>
        <label asp-for="Form.Price"></label>
        <input asp-for="Form.Price" />
        <span asp-validation-for="Form.Price" class="text-danger"></span>
    </div>
    <button type="submit">저장</button>
</form>

<form method="post" asp-page-handler="Delete">
    <input asp-for="Form.Id" type="hidden" />
    <button type="submit">삭제</button>
</form>

@section Scripts { <partial name="_ValidationScriptsPartial" /> }
```

`Pages/Products/Edit.cshtml.cs`
```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.RazorPages;
using System.ComponentModel.DataAnnotations;

public class ProductEditDto
{
    public int Id { get; set; }
    [Required, StringLength(100)] public string Name { get; set; } = "";
    [Range(0, int.MaxValue)] public int Price { get; set; }
}

public class EditModel : PageModel
{
    private readonly IProductService _svc;

    [BindProperty(SupportsGet = true)]
    public int Id { get; set; }

    [BindProperty]
    public ProductEditDto Form { get; set; } = new();

    public EditModel(IProductService svc) => _svc = svc;

    public async Task<IActionResult> OnGetAsync()
    {
        var p = await _svc.FindAsync(Id);
        if (p is null) return NotFound();

        Form = new ProductEditDto { Id = p.Id, Name = p.Name, Price = p.Price };
        return Page();
    }

    public async Task<IActionResult> OnPostSaveAsync()
    {
        if (!ModelState.IsValid) return Page();
        await _svc.UpdateAsync(Form.Id, Form.Name, Form.Price);

        TempData["flash"] = "저장되었습니다.";
        return RedirectToPage("Index");
    }

    public async Task<IActionResult> OnPostDeleteAsync()
    {
        await _svc.DeleteAsync(Form.Id);
        TempData["flash"] = "삭제되었습니다.";
        return RedirectToPage("Index");
    }
}
```

---

## 부록 B. Program.cs 최소 구성 예시

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddRazorPages(options =>
{
    options.Conventions.AuthorizeFolder("/Admin", "AdminPolicy");
}).AddViewLocalization(); // 다국어 필요 시

builder.Services.AddAuthorization(o =>
{
    o.AddPolicy("AdminPolicy", p => p.RequireRole("Admin"));
});

builder.Services.AddScoped<IProductService, ProductService>();

var app = builder.Build();

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

app.MapRazorPages();
app.Run();
```
