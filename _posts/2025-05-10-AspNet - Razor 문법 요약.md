---
layout: post
title: AspNet - Razor 문법 요약
date: 2025-05-10 21:20:23 +0900
category: AspNet
---
# Razor 문법 완전 정리 (ASP.NET Core)
## Razor란?

- HTML 안에 **C#**을 자연스럽게 섞을 수 있게 하는 **뷰 템플릿 엔진**.
- 파일 확장자: `.cshtml`
- **기본 HTML 인코딩**으로 XSS에 강함(필요 시 `Html.Raw`로 예외 처리).

---

## Razor의 핵심: `@` 전환과 출력

### 인라인 전환

```razor
<p>Hello, @Model.UserName!</p>
<p>합계: @(Model.Price * Model.Quantity)</p>  <!-- 복잡 표현은 괄호 권장 -->
```

### 코드 블록

```razor
@{
    var now = DateTime.Now;
    var title = "Razor 예제";
    int Sum(int a, int b) => a + b;
}
<h1>@title</h1>
<p>현재 시간: @now</p>
<p>2 + 3 = @Sum(2, 3)</p>
```

### 문자열/HTML 출력과 인코딩

```razor
@("<b>Bold</b>")            <!-- &lt;b&gt;Bold&lt;/b&gt; 로 인코딩되어 출력 -->
@Html.Raw("<b>Bold</b>")    <!-- 실제 <b>Bold</b> 로 렌더링 -->
```

### `@` 자체를 문자로 출력

```razor
@@  <!-- 결과: @ -->
```

---

## 흐름 제어(제어문/표현식)

### 조건/반복

```razor
@if (Model.IsAdmin) { <p>관리자입니다.</p> } else { <p>일반 사용자입니다.</p> }

@for (var i = 0; i < 3; i++) { <p>Index: @i</p> }

<ul>
@foreach (var p in Model.Products)
{
    <li>@p.Name - @p.Price.ToString("N0") 원</li>
}
</ul>
```

### switch

```razor
@switch (Model.Status)
{
    case Status.Pending: <span class="text-warning">대기</span>; break;
    case Status.Active:  <span class="text-success">활성</span>; break;
    default:             <span>기타</span>; break;
}
```

### null 전파/조건부

```razor
<p>@Model?.Profile?.DisplayName ?? "이름 없음"</p>
```

---

## 모델 바인딩 & 형식 지정

### 뷰의 모델 타입 지정

```razor
@model MyApp.Models.User
<h1>@Model.UserName 님 환영합니다!</h1>
```

### 날짜/숫자 포맷(문화권 반영)

```razor
@DateTime.Now.ToString("D", System.Globalization.CultureInfo.CurrentCulture)
@1234567.ToString("N", System.Globalization.CultureInfo.CurrentCulture)
```

---

## 레이아웃, 섹션, 시작 파일

### 레이아웃 지정

```razor
@{
    Layout = "_Layout"; // /Views/Shared/_Layout.cshtml 또는 /Pages/Shared/_Layout.cshtml
}
```

### 섹션 정의/렌더링

_레이아웃 내:_
```razor
<body>
    @RenderBody()
    @RenderSection("Scripts", required: false)
</body>
```
_각 View에서:_
```razor
@section Scripts {
    <script src="..."></script>
}
```

### `_ViewStart.cshtml` / `_ViewImports.cshtml`

- `_ViewStart.cshtml`: 모든 뷰의 공통 설정(예: `Layout`) 지정.
- `_ViewImports.cshtml`: 공통 `@using`, Tag Helper, 네임스페이스 등.
```razor
@* _ViewImports.cshtml *@
@using MyApp.Models
@addTagHelper *, Microsoft.AspNetCore.Mvc.TagHelpers
```

---

## 주석과 텍스트

- Razor 주석(클라이언트에 출력 안 됨):
```razor
@* 이건 Razor 주석입니다 *@
```
- HTML 주석(클라이언트로 전달됨):
```html
<!-- 클라이언트에서 보이는 주석 -->
```

---

## 폼/바인딩/검증(태그 헬퍼)

### 폼 + Tag Helper

```razor
@model MyApp.Models.ContactInput

<form asp-action="Submit" asp-controller="Contact" method="post">
    <div class="mb-2">
        <label asp-for="Email"></label>
        <input asp-for="Email" class="form-control" />
        <span asp-validation-for="Email" class="text-danger"></span>
    </div>
    <div class="mb-2">
        <label asp-for="Message"></label>
        <textarea asp-for="Message" class="form-control"></textarea>
        <span asp-validation-for="Message" class="text-danger"></span>
    </div>
    <button type="submit" class="btn btn-primary">보내기</button>
</form>

@section Scripts {
    <partial name="_ValidationScriptsPartial" />
}
```

> `asp-for`는 모델 속성 이름을 기반으로 name/id/value, validation 속성 등을 자동 구성.

### Razor Pages의 POST 핸들러

```csharp
public class IndexModel : PageModel
{
    [BindProperty] public string Email { get; set; } = "";
    [BindProperty] public string Message { get; set; } = "";

    public IActionResult OnPost()
    {
        if (!ModelState.IsValid) return Page();
        // 처리 로직
        return RedirectToPage("Success");
    }
}
```

### 검증 메시지와 요약

```razor
<div asp-validation-summary="All" class="text-danger"></div>
<span asp-validation-for="Email" class="text-danger"></span>
```

### Anti-forgery(기본)

- `form` Tag Helper가 자동으로 `__RequestVerificationToken` 생성(컨트롤러/페이지에서 `[ValidateAntiForgeryToken]` 기본 활성).
- Ajax 시 토큰 헤더 포함 필요.

---

## 링크/라우팅 Tag Helper

### MVC 링크

```razor
<a asp-controller="Home" asp-action="About">소개</a>
<a asp-controller="Users" asp-action="Detail" asp-route-id="5">유저</a>
```

### Razor Pages 링크

```razor
<a asp-page="/Contact">문의하기</a>
<a asp-page="/Products/Detail" asp-route-id="10">제품 상세</a>
```

### 환경별 자원 로딩

```razor
<environment include="Development">
    <script src="~/lib/jquery/jquery.js"></script>
</environment>
<environment exclude="Development">
    <script src="~/lib/jquery/jquery.min.js"></script>
</environment>
```

---

## Partial View, View Component, Editor/Display Templates

### Partial View

```razor
<partial name="_LoginPartial" />
@await Html.PartialAsync("_ProductCard", product)
```
- **장점**: 재사용 가능, 간단.
- **주의**: 데이터 전달은 모델/뷰데이터/뷰백 등.

### View Component(로직+뷰 결합 재사용)

**클래스:**
```csharp
public class CartSummaryViewComponent : ViewComponent
{
    private readonly ICartService _cart;
    public CartSummaryViewComponent(ICartService cart) => _cart = cart;

    public IViewComponentResult Invoke()
    {
        var model = _cart.GetSummary();
        return View(model); // Views/Shared/Components/CartSummary/Default.cshtml
    }
}
```
**호출:**
```razor
@await Component.InvokeAsync("CartSummary")
```
- **장점**: DI/비즈니스 로직 포함한 캡슐화된 UI 블록.

### Editor/Display Templates(이름 규칙 기반)

- 위치: `Views/Shared/EditorTemplates/TypeName.cshtml`, `Views/Shared/DisplayTemplates/TypeName.cshtml`
```razor
@Html.EditorFor(m => m.Address)   // Address 타입 템플릿 자동 찾음
@Html.DisplayFor(m => m.Address)
```
- **장점**: 타입별 UI 표준화, 반복 제거.

---

## 조건부/동적 특성 렌더링

### 조건부 속성

```razor
<input class="@(Model.IsError ? "form-control is-invalid" : "form-control")" />

<button disabled="@(Model.IsBusy ? "disabled" : null)">저장</button>
```

### 반복으로 속성 합성

```razor
@{
    var attrs = new Dictionary<string, object> {
        ["class"] = "btn btn-primary",
        ["data-id"] = Model.Id
    };
}
<button @attributes="attrs">확인</button>  @* Blazor에서 주로 사용(참고) *@
```
> MVC Razor에서는 위와 같은 `@attributes` 패턴은 적용되지 않는다(Blazor 문법). MVC에서는 문자열로 조립하자.

---

## 비동기/await & HtmlHelper

### 비동기 호출

```razor
@await Html.PartialAsync("_User", Model.User)
@await Component.InvokeAsync("CartSummary")
```

### Url/Html/Json 헬퍼 예시

```razor
@Url.Action("Detail", "Products", new { id = 5 })     <!-- /Products/Detail/5 -->
@Html.DisplayNameFor(m => m.UserName)
@Html.TextBoxFor(m => m.Email, new { @class = "form-control" })
@Html.Raw(Json.Serialize(Model))  @* 필요 시 JSON 출력(직접 유틸 작성/주입) *@
```

---

## 핵심 모음

| 지시문 | 설명 | 예시 |
|---|---|---|
| `@model` | 뷰 모델 지정 | `@model MyApp.Models.User` |
| `@using` | using 선언 | `@using MyApp.Common` |
| `@inject` | DI 주입 | `@inject ILogger<IndexModel> Log` |
| `@inherits` | 커스텀 베이스 뷰 | `@inherits MyBaseView<TModel>` |
| `@section` | 레이아웃 섹션 정의 | `@section Scripts { ... }` |
| `@addTagHelper` / `@removeTagHelper` | Tag Helper 범위 설정 | `_ViewImports.cshtml`에서 관리 |
| `@page` | Razor Pages에서 페이지로 지정 | `@page "{id:int}"`(Pages 전용) |
| `@functions` | (MVC Razor) 뷰 내부 C# 멤버 | `@functions { string Hi() => "hi"; }` |

### `@inject` 예

```razor
@inject Microsoft.Extensions.Logging.ILogger<IndexModel> Logger
@{ Logger.LogInformation("뷰 렌더링"); }
```

### `@inherits` 베이스 뷰

```csharp
public abstract class MyBaseView<T> : RazorPage<T>
{
    public string AppName => "MyApp";
}
```
뷰에서:
```razor
@inherits MyApp.Infrastructure.MyBaseView<MyApp.Models.User>
<h1>@AppName - @Model.UserName</h1>
```

---

## — `@page`

```razor
@page "{id:int?}"
@model Pages.Products.DetailModel
<h2>제품 상세: @Model.Id</h2>
```
- `@page`가 있으면 해당 `.cshtml`이 **엔드포인트**.
- 경로 패턴, 제약(`int`, `min`, `max`) 지정 가능.

---

## 환경/정적 파일/버전 태그

### 정적 파일 버전 태그

```razor
<link rel="stylesheet" href="~/css/site.css" asp-append-version="true" />
<script src="~/js/site.js" asp-append-version="true"></script>
```
- 파일 해시를 쿼리스트링에 붙여 캐시 무효화.

### 환경 태그

```razor
<environment include="Development">
    <script src="~/js/dev-only.js"></script>
</environment>
<environment exclude="Development">
    <script src="~/js/prod-only.min.js"></script>
</environment>
```

---

## 보안/권한과 Razor

### 인증 상태에 따른 출력

```razor
@using Microsoft.AspNetCore.Authorization
@inject Microsoft.AspNetCore.Identity.SignInManager<MyUser> SignInManager

@if (SignInManager.IsSignedIn(User))
{
    <a asp-controller="Account" asp-action="Logout">로그아웃</a>
}
else
{
    <a asp-controller="Account" asp-action="Login">로그인</a>
}
```

### 권한 체크

```razor
@inject IAuthorizationService Authz
@{
    var allowed = (await Authz.AuthorizeAsync(User, "AdminPolicy")).Succeeded;
}
@if (allowed) { <a asp-page="/Admin">관리</a> }
```

---

## 성능/운영 팁

- **인코딩 규칙**: 기본 HTML 인코딩. `HtmlString`/`IHtmlContent`는 신중 사용.
- **부분 뷰 남발 주의**: 렌더링 비용 + 모델 준비 비용 고려. 캐싱/뷰 컴포넌트와 균형.
- **정적 리소스**: `asp-append-version`, CDN/압축/HTTP/2 활용.
- **러닝 타임 변경**: 개발 중에만 Runtime Compilation(개발환경 패키지) 고려.
- **반복 내 비싼 계산**: 뷰에서 계산 최소화하고 컨트롤러/페이지 모델에서 준비.

---

## 실전 예제 — 제품 목록 + 상세 카드 + 검증 + 섹션

### `_ViewImports.cshtml`

```razor
@using MyApp.Models
@addTagHelper *, Microsoft.AspNetCore.Mvc.TagHelpers
```

### `_Layout.cshtml`

```razor
<!doctype html>
<html>
<head>
    <meta charset="utf-8" />
    <title>@ViewData["Title"] - MyApp</title>
    <link rel="stylesheet" href="~/css/site.css" asp-append-version="true" />
</head>
<body>
    <header><a asp-controller="Home" asp-action="Index">Home</a></header>
    <main class="container">
        @RenderBody()
    </main>
    @RenderSection("Scripts", required: false)
</body>
</html>
```

### `Views/Products/Index.cshtml`

```razor
@model IEnumerable<Product>
@{
    ViewData["Title"] = "제품 목록";
    Layout = "_Layout";
}
<h1>제품 목록</h1>
<div class="row">
@foreach (var p in Model)
{
    <div class="col-4 mb-3">
        @await Html.PartialAsync("_ProductCard", p)
    </div>
}
</div>
```

### `Views/Shared/_ProductCard.cshtml` (Partial)

```razor
@model Product
<div class="card">
  <div class="card-body">
    <h5 class="card-title">@Model.Name</h5>
    <p class="card-text">가격: @Model.Price.ToString("N0") 원</p>
    <a class="btn btn-sm btn-primary"
       asp-controller="Products"
       asp-action="Detail" asp-route-id="@Model.Id">자세히</a>
  </div>
</div>
```

### `Views/Products/Create.cshtml` (검증/폼)

```razor
@model ProductInput
@{
    ViewData["Title"] = "제품 등록";
    Layout = "_Layout";
}
<h1>제품 등록</h1>

<form asp-action="Create" method="post">
  <div asp-validation-summary="ModelOnly" class="text-danger"></div>
  <div class="mb-2">
    <label asp-for="Name" class="form-label"></label>
    <input asp-for="Name" class="form-control" />
    <span asp-validation-for="Name" class="text-danger"></span>
  </div>
  <div class="mb-2">
    <label asp-for="Price" class="form-label"></label>
    <input asp-for="Price" class="form-control" />
    <span asp-validation-for="Price" class="text-danger"></span>
  </div>
  <button type="submit" class="btn btn-primary">저장</button>
</form>

@section Scripts {
    <partial name="_ValidationScriptsPartial" />
}
```

---

## 디버깅/문제 해결 체크

| 증상 | 가능 원인 | 해결 힌트 |
|---|---|---|
| Tag Helper 작동 안 함 | `_ViewImports.cshtml`에 `@addTagHelper` 없음 | `@addTagHelper *, Microsoft.AspNetCore.Mvc.TagHelpers` 추가 |
| 링크 생성 경로 이상 | 라우트 값/영역/컨트롤러 액션명 오타 | `asp-area/asp-controller/asp-action/asp-page` 재확인 |
| 검증 메시지 미출력 | 스크립트 누락/모델 상태 미검증 | `_ValidationScriptsPartial`, 서버측 ModelState 점검 |
| XSS 우려 | `Html.Raw` 남용 | 반드시 신뢰된 HTML만 사용, 가능하면 피함 |
| 성능 저하 | 부분 뷰 과다/루프 내부 복잡 계산 | 컨트롤러에서 ViewModel로 계산/정리, 캐시 고려 |

---

## 요약 테이블

| 구분 | 핵심 |
|---|---|
| 출력/전환 | `@`, `@{ }`, 괄호로 복잡식 `@( )` |
| 모델 | `@model`, `asp-for`, 검증/요약 |
| 레이아웃/섹션 | `Layout`, `@section` + `@RenderSection` |
| 라우팅 링크 | `asp-controller/action`, `asp-page`, `asp-route-*` |
| 파셜/컴포넌트 | `partial`, `Html.PartialAsync`, `ViewComponent` |
| 지시문 | `@using`, `@inject`, `@inherits`, `@page`(Pages) |
| 보안/검증 | 기본 인코딩, Anti-forgery, DataAnnotations |
| 성능 팁 | 정적 리소스 버전, 부분 뷰·계산 최소화, 캐시 |

---

### 간단한 수식 포맷(블로그 표기용)

- Razor 자체는 수식을 렌더링하지 않지만, 블로그에 수학을 적을 경우 MathJax 등과 함께 다음처럼 마크업할 수 있다:
```
$$
S = \sum_{i=1}^{n} p_i \cdot q_i
$$
```
> 실제 웹에서는 MathJax 스크립트를 레이아웃에 추가해 렌더링한다.
