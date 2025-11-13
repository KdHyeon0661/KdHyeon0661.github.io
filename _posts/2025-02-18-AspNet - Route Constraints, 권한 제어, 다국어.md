---
layout: post
title: AspNet -Route Constraints, 권한 제어, 다국어
date: 2025-02-18 19:20:23 +0900
category: AspNet
---
# ASP.NET Core 고급 라우팅: Route Constraints, 권한 제어, 다국어(Localization)

## 0. 라우팅 고급 개요 — “경로가 규칙을 말한다”

- **Route Constraints**로 **형식·범위**를 라우팅 단계에서 강제 → 잘못된 요청을 **초기에 거부**하고 핸들러 오버헤드를 줄입니다.
- **권한(Authorization)**은 라우트 구조와 결합하여 **영역/리소스 단위의 접근 정책**을 명확히 합니다.
- **Localization**을 라우트 단계에 녹이면 **언어/문화권**에 따라 **동일한 리소스의 다양한 표현**을 제공하고, **SEO·북마크**에 유리합니다.

핵심 원칙:
1) **식별자(Route)**는 “무엇”을 가리키고,
2) **상태(Query)**는 “어떻게” 볼지를 결정,
3) **권한**은 “누가” 접근 가능한지,
4) **Localization**은 “어떤 언어/문화권”으로 보일지를 결정.

---

## 1. Route Constraints(라우트 제약 조건) — 카탈로그와 조합 규칙

### 1.1 기본 사용

Razor Pages:
```cshtml
@page "{id:int}"
```
MVC:
```csharp
[HttpGet("product/{id:int}")]
public IActionResult Details(int id) => View();
```
- 정수가 아니면 404(매칭 실패). 컨트롤러/핸들러 실행 이전에 **차단**됩니다.

### 1.2 내장 제약 카탈로그(실무 예 포함)

| 제약 | 설명 | Razor Pages 예 | MVC/Minimal 예 |
|---|---|---|---|
| `int`, `long`, `float`, `double`, `decimal` | 숫자 | `@page "{id:int}"` | `[HttpGet("{id:int}")]` |
| `bool` | 불리언 | `@page "{flag:bool}"` | `[HttpGet("{flag:bool}")]` |
| `guid` | GUID | `@page "{key:guid}"` | `[HttpGet("{key:guid}")]` |
| `datetime`, `date`, `time` | 날짜/시간 | `@page "{dt:datetime}"` | `[HttpGet("{dt:datetime}")]` |
| `alpha` | 알파벳만 | `@page "{name:alpha}"` | `[HttpGet("{name:alpha}")]` |
| `min(n)` / `max(n)` / `range(a,b)` | 범위 | `@page "{age:int:range(1,120)}"` | `[HttpGet("{age:int:min(1)}")]` |
| `length(n)` / `minlength(n)` / `maxlength(n)` | 문자열 길이 | `@page "{code:length(6)}"` | `[HttpGet("{code:minlength(3)}")]` |
| `regex(pat)` | 정규식 | `@page "{slug:regex(^[a-z0-9-]+$)}"` | `[HttpGet("{slug:regex(^[a-z0-9-]+$)}")]` |
| 선택·캐치올 | `?` 선택, `*` 캐치올 | `@page "{id?}"`, `{*path}` | `[HttpGet("{*path}")]` |

실무 팁:
- **조합 가능**: `{id:int:min(1)}`
- **엄격한 제약**은 곧 **보안**(불필요한 컨트롤러/DB 접근 차단)과 **성능**(미스매치 조기 종료)입니다.

### 1.3 고급: 커스텀 제약 구현(IRouteConstraint)

예) 특정 화이트리스트 코드만 허용
```csharp
public sealed class CodeWhitelistConstraint : IRouteConstraint
{
    private static readonly HashSet<string> Allowed =
        new(StringComparer.OrdinalIgnoreCase) { "A1", "B2", "C3" };

    public bool Match(HttpContext? httpContext, IRouter? route, string routeKey,
                      RouteValueDictionary values, RouteDirection routeDirection)
    {
        if (!values.TryGetValue(routeKey, out var raw) || raw is null) return false;
        return Allowed.Contains(raw.ToString()!);
    }
}
```
등록/사용:
```csharp
builder.Services.Configure<RouteOptions>(o =>
    o.ConstraintMap["codewl"] = typeof(CodeWhitelistConstraint));

app.MapControllerRoute("code", "promo/{code:codewl}",
    new { controller = "Promo", action = "Show" });
```

### 1.4 파라미터 변환기(출력 변환)로 SEO 개선

PascalCase 컨트롤러/액션을 kebab-case로 노출:
```csharp
public class SlugifyTransformer : IOutboundParameterTransformer
{
    public string? TransformOutbound(object? value) =>
        value is null ? null :
        Regex.Replace(value.ToString()!, "([a-z])([A-Z])", "$1-$2").ToLowerInvariant();
}

builder.Services.AddRouting(o => o.ConstraintMap["slugify"] = typeof(SlugifyTransformer));

app.MapControllerRoute(
    name: "default",
    pattern: "{controller:slugify}/{action:slugify}/{id?}");
```

---

## 2. 라우트 기반 권한 제어 — 정책과 구조의 결합

권한은 “**라우트가 가리키는 리소스**”의 **접근 통제**입니다. 라우팅 구조가 명확할수록, 권한 정책도 **예측 가능**합니다.

### 2.1 Razor Pages — 페이지/폴더별 정책

PageModel 속성:
```csharp
[Authorize]
public class EditModel : PageModel
{
    public void OnGet() { }
}
```
폴더/페이지 규칙(Program.cs):
```csharp
builder.Services.AddRazorPages(options =>
{
    options.Conventions.AuthorizeFolder("/Admin", "AdminPolicy");
    options.Conventions.AllowAnonymousToPage("/Account/Login");
});
```

정책 정의:
```csharp
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("AdminPolicy", p => p.RequireRole("Admin"));
});
```

### 2.2 MVC/Minimal — 속성·필터 기반

컨트롤러/액션:
```csharp
[Authorize(Roles = "Admin")]
public class UsersController : Controller
{
    [AllowAnonymous]
    public IActionResult PublicProfile() => View();
}
```

특정 **핸들러만 보호(Razor Pages)**:
```csharp
public class OrderModel : PageModel
{
    public void OnGet() { }

    [Authorize] // POST /Order?handler=Pay 보호
    public IActionResult OnPostPay() { /* 결제 */ return RedirectToPage("Done"); }
}
```

### 2.3 정책 기반(Claims·조건·시간대)

예) 만 18세 이상 정책(간단화 예):
```csharp
builder.Services.AddAuthorization(o =>
{
    o.AddPolicy("Over18", p => p.RequireAssertion(ctx =>
    {
        var claim = ctx.User.FindFirst("Age");
        return int.TryParse(claim?.Value, out var age) && age >= 18;
    }));
});
```
사용:
```csharp
[Authorize(Policy = "Over18")]
public class AdultOnlyPage : PageModel { }
```

### 2.4 엔드포인트 라우팅과 정책의 만남

엔드포인트에 메타데이터로 정책 포함(필터 없이도 가능):
```csharp
app.MapControllerRoute(
    name: "admin",
    pattern: "admin/{controller=Home}/{action=Index}/{id?}")
   .RequireAuthorization("AdminPolicy");
```
- 경로 자체에 **RequireAuthorization**를 부여 → 영역 전체를 보호.

### 2.5 실무 설계 지침

- **네임스페이스화**: `/admin`, `/account`, `/api` 같은 프리픽스로 **권한 경계**를 분명히.
- **정책 이름 = 역할/목적**: “AdminPolicy”, “PaidCustomer”, “OwnerOrAdmin”.
- **라우트가 간결할수록** 권한도 간결. 라우트 난립은 권한 예외의 **폭발**로 이어짐.

---

## 3. 라우트 기반 Localization(다국어 URL) — 문화권을 경로에 담기

URL에 문화권을 포함:
```
/en/products/42
/ko/products/42
/fr/products/42
```
장점: **SEO**, **공유/북마크**, **언어 전환 링크**가 명확.

### 3.1 RequestLocalizationOptions 설정

```csharp
var cultures = new[] { "en", "ko", "fr" };

builder.Services.Configure<RequestLocalizationOptions>(options =>
{
    options.SetDefaultCulture("en");
    options.AddSupportedCultures(cultures);
    options.AddSupportedUICultures(cultures);

    // URL에서 문화권을 먼저 해석
    options.RequestCultureProviders.Insert(0, new RouteDataRequestCultureProvider
    {
        RouteDataStringKey = "culture",
        UIRouteDataStringKey = "culture"
    });
});
```

미들웨어:
```csharp
var app = builder.Build();
app.UseRequestLocalization(); // 반드시 UseRouting 이전/이후 상관없이 일찍 배치 권장
app.UseRouting();
```

### 3.2 라우트 템플릿에 `{culture}` 추가

MVC 컨벤션:
```csharp
app.MapControllerRoute(
    name: "localizedDefault",
    pattern: "{culture=en}/{controller=Home}/{action=Index}/{id?}");
```

Razor Pages 전역 규칙(모든 페이지에 culture 프리픽스 추가):
```csharp
builder.Services.AddRazorPages(options =>
{
    options.Conventions.AddFolderRouteModelConvention("/", model =>
    {
        foreach (var sel in model.Selectors)
        {
            var t = sel.AttributeRouteModel?.Template ?? string.Empty;
            sel.AttributeRouteModel = new AttributeRouteModel
            {
                Template = "{culture=en}/" + t
            };
        }
    });
});
```

### 3.3 문화권 제약(정규식) — 잘못된 코드 차단

두 글자 언어 코드만 허용(간략화):
```csharp
app.MapControllerRoute(
    "localizedDefault",
    "{culture:regex(^[a-z]{2}$)}/{controller=Home}/{action=Index}/{id?}");
```

실무에서는 `en-US` 같은 **언어-지역** 조합을 더 자주 사용:
```csharp
"{culture:regex(^[a-z]{2}-[A-Z]{2}$)}"
```

### 3.4 뷰/페이지에서 문화권 유지 링크

Razor Pages:
```cshtml
<a asp-page="/Index" asp-route-culture="ko">한국어</a>
<a asp-page="/Index" asp-route-culture="en">English</a>
```

MVC:
```cshtml
<a asp-controller="Home" asp-action="Index" asp-route-culture="ko">한국어</a>
```

현재 URL의 **쿼리 파라미터 유지**:
```cshtml
@{
    var route = ViewContext.HttpContext.Request.Query
        .ToDictionary(kv => kv.Key, kv => (string)kv.Value);
    route["culture"] = "ko-KR";
}
<a asp-page="/Index" asp-all-route-data="route">한국어(쿼리 유지)</a>
```

### 3.5 리소스 파일 `.resx`와 IStringLocalizer

리소스 배치:
```
Resources/
  Pages/
    Index.ko.resx
    Index.en.resx
```

페이지에서:
```cshtml
@inject Microsoft.Extensions.Localization.IStringLocalizer<IndexModel> L
<h1>@L["WelcomeMessage"]</h1>
```

- 현재 `Thread.CurrentUICulture`에 따라 적절한 리소스를 로드.
- URL로 문화권이 바뀌어도 **일관된 키**로 메시지를 조회.

### 3.6 Culture 유지 PRG 패턴

POST 이후 Redirect 시 문화권 유지:
```csharp
public IActionResult OnPost(string culture)
{
    // 처리...
    return RedirectToPage("/Index", new { culture });
}
```

---

## 4. 링크 생성/리다이렉트 — 문화권·권한·제약과 함께

### 4.1 LinkGenerator/Url.* API

서비스에서 안전하게 경로 생성:
```csharp
public class Links
{
    private readonly LinkGenerator _link;
    public Links(LinkGenerator link) => _link = link;

    public string? Product(string culture, int id, HttpContext ctx) =>
        _link.GetPathByAction(ctx, action: "Details", controller: "Products",
            values: new { culture, id });
}
```

### 4.2 `asp-route-*`와 `asp-all-route-data`

검색 상태(쿼리) 보존:
```cshtml
@{
    var merged = ViewContext.HttpContext.Request.Query
        .ToDictionary(kv => kv.Key, kv => (string)kv.Value);
    merged["page"] = "2";
}
<a asp-page="./Index" asp-all-route-data="merged">2페이지</a>
```

---

## 5. 보안·성능·SEO — 라우팅 단계에서의 품질 설계

### 5.1 보안 수칙

- **화이트리스트 기반 제약**: 정렬/필드명/슬러그/코드 등은 라우팅 또는 바인딩 레벨에서 허용값 제한.
- **민감정보 GET 금지**: 토큰/비밀은 URL에 두지 말 것. POST/헤더/쿠키/서버 상태 사용.
- **경로 순회 방지**: 파일 경로를 파라미터로 받을 때 `Path.GetFileName()` 등으로 정규화.

### 5.2 성능

- 제약으로 미매칭 요청을 **빠르게 배제** → 컨트롤러/페이지 진입 전 종료.
- **정확-특정 → 일반** 순서로 라우트 등록(특히 MVC 컨벤션).
- 정규식 제약은 강력하지만 **비싸다**: 가능한 구체적 타입/길이 제약 우선.

### 5.3 SEO·국제화

- 문화권을 경로 프리픽스로 노출(예: `/ko/…`), 슬러그는 소문자/kebab-case.
- 중복 URL 방지: **Canoncial Link** 또는 리다이렉트 규칙을 정의.
- 301/308 리다이렉트로 **영구 이동** 명확화(예: 언어 없을 때 기본 문화권으로).

---

## 6. 실전 시나리오

### 6.1 지역화 + 제약 + 권한(관리 영역)

라우트:
```csharp
app.MapControllerRoute(
    name: "admin",
    pattern: "{culture:regex(^[a-z]{2}-[A-Z]{2}$)}/admin/{controller=Home}/{action=Index}/{id?}")
   .RequireAuthorization("AdminPolicy");
```
- 관리자 UI는 **언어별 URL** 제공, 동시에 **정책 보호**.

컨트롤러:
```csharp
[Authorize(Policy = "AdminPolicy")]
public class HomeController : Controller
{
    [HttpGet("")]
    public IActionResult Index() => View();
}
```

### 6.2 Razor Pages — 제품 상세(숫자 ID) + 문화권 + SEO 별칭

Program.cs(별칭 라우트):
```csharp
builder.Services.AddRazorPages(o =>
{
    // /ko/p/42, /en/p/42 와 같은 짧은 경로 제공
    o.Conventions.AddPageRoute("/Products/Details", "{culture}/p/{id:int}");
});
```

Page:
```cshtml
@page "{culture:regex(^[a-z]{2}-[A-Z]{2}$)}/products/{id:int}"
@model ProductDetailsModel
<h1>@Model.Title</h1>
```

---

## 7. 테스트/디버깅 전략

- **라우팅 통합 테스트**: 특정 URL → 원하는 핸들러/상태 코드/리다이렉트 확인.
- **엔드포인트 확인 미들웨어**:
```csharp
app.Use(async (ctx, next) =>
{
    await next();
    var ep = ctx.GetEndpoint();
    if (ep != null)
        Console.WriteLine($"Matched: {ep.DisplayName}");
});
```
- 개발 환경에서 **UseDeveloperExceptionPage** 및 라우트 로깅 활성화.

---

## 8. 에지 케이스와 함정

1) **제약 충돌**: 같은 경로 패턴에 서로 다른 제약이 섞여 등록되면 **예기치 못한 미스매치**.
2) **문화권 코드 느슨함**: `en`, `en-US`, `EN-us` 혼용 → 제약으로 **형식 고정**.
3) **긴 쿼리**: 필터가 많은 목록 화면은 URL 길이 제한에 부딪힐 수 있음 → POST/서버 상태 토큰.
4) **다중 문화권 리다이렉트 루프**: 쿠키/브라우저 선호 언어/경로 문화권이 **충돌**하지 않도록 우선순위 명확히.
5) **권한 예외 누락**: 로그인/헬스체크/에러 페이지 등은 반드시 **AllowAnonymous** 또는 특정 정책에서 제외.

---

## 9. 요약 표

| 주제 | 핵심 |
|---|---|
| 제약(Constraints) | 형식/범위/패턴을 라우팅에서 강제. 조합·커스텀 제약으로 보안·성능 개선 |
| 권한(Authorization) | 폴더/페이지/경로별 정책. RequireAuthorization로 엔드포인트 레벨 보호 |
| Localization | `{culture}` 세그먼트 + RequestLocalizationProvider + `.resx` 자원 |
| 링크/리다이렉트 | `asp-route-*` / `asp-all-route-data` / `LinkGenerator`로 강타입 생성 |
| SEO/국제화 | 언어 프리픽스, 소문자 슬러그, 캐논니컬, 301/308 리다이렉트 |
| 테스트/디버깅 | 통합 테스트, `GetEndpoint()` 로깅, 개발자 예외 페이지 |

---

# 부록 A. 코드 스니펫 모음

### A.1 전체 Program.cs 골격(.NET 8, MVC+Pages+Localization)

```csharp
var builder = WebApplication.CreateBuilder(args);

// MVC + Pages
builder.Services.AddControllersWithViews();
builder.Services.AddRazorPages();

// Localization
var cultures = new[] { "en-US", "ko-KR", "fr-FR" };
builder.Services.Configure<RequestLocalizationOptions>(opt =>
{
    opt.SetDefaultCulture("en-US");
    opt.AddSupportedCultures(cultures);
    opt.AddSupportedUICultures(cultures);
    opt.RequestCultureProviders.Insert(0, new RouteDataRequestCultureProvider
    {
        RouteDataStringKey = "culture",
        UIRouteDataStringKey = "culture"
    });
});

// Authorization
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("AdminPolicy", p => p.RequireRole("Admin"));
});

var app = builder.Build();

app.UseRequestLocalization();
app.UseHttpsRedirection();
app.UseStaticFiles();

app.UseRouting();

app.UseAuthentication();
app.UseAuthorization();

// Localized MVC route
app.MapControllerRoute(
    name: "localized-default",
    pattern: "{culture:regex(^[a-z]{2}-[A-Z]{2}$)}/{controller=Home}/{action=Index}/{id?}");

// Admin area protected
app.MapControllerRoute(
    name: "admin",
    pattern: "{culture:regex(^[a-z]{2}-[A-Z]{2}$)}/admin/{controller=Home}/{action=Index}/{id?}")
   .RequireAuthorization("AdminPolicy");

// Razor Pages (optionally add conventions)
app.MapRazorPages();

app.Run();
```

### A.2 Razor Pages 전역 라우팅 규칙(문화권 프리픽스 붙이기)

```csharp
builder.Services.AddRazorPages(options =>
{
    options.Conventions.AddFolderRouteModelConvention("/", model =>
    {
        foreach (var s in model.Selectors)
        {
            var template = s.AttributeRouteModel?.Template ?? string.Empty;
            s.AttributeRouteModel = new AttributeRouteModel
            {
                Template = "{culture:regex(^[a-z]{2}-[A-Z]{2}$)}/" + template
            };
        }
    });
});
```

### A.3 Razor 페이지에서 리소스 사용 예

```cshtml
@page "{culture:regex(^[a-z]{2}-[A-Z]{2}$)}/home"
@model IndexModel
@inject Microsoft.Extensions.Localization.IStringLocalizer<IndexModel> L
<h1>@L["WelcomeMessage"]</h1>
```

리소스:
```
Resources/Pages/IndexModel.en-US.resx
Resources/Pages/IndexModel.ko-KR.resx
```

---

# 결론

- **Route Constraints**는 첫 관문에서 형식/범위를 강제하여 **보안과 성능**을 동시에 잡습니다.
- **권한 제어**는 라우트 구조와 정책을 결합하여 **명확한 경계**를 제공합니다(폴더/페이지/엔드포인트 수준).
- **Localization**을 라우팅에 녹이면 **언어별 URL**, **리소스 로딩**, **SEO**가 자연스럽게 결합됩니다.
- 본문 패턴(제약·정책·문화권·링크·PRG·테스트)을 템플릿화하면, 팀 전체의 **URL 일관성**, **유지보수성**, **국제화 품질**을 크게 끌어올릴 수 있습니다.
