---
layout: post
title: AspNet - Razor 문법 기초 (1)
date: 2025-02-10 19:20:23 +0900
category: AspNet
---
# Razor 문법 기초 완전 정복

## Razor란 무엇인가 — HTML + C# 혼합 템플릿

- **Razor**는 **C# 기반**의 서버 사이드 템플릿 엔진입니다.
- `.cshtml` 파일에서 **HTML**과 **C#**을 자연스럽게 혼합하며, 기본적으로 **HTML 인코딩이 자동 적용**되어 XSS를 예방합니다.
- **Razor Pages**(페이지 지향)와 **MVC View**(컨트롤러-뷰 구조) 모두에서 동일 문법을 사용합니다.

---

## Razor 핵심 기호 `@` — 출력과 코드 전환

### 기본 출력

```cshtml
@DateTime.Now
@("안녕하세요! 오늘은 " + DateTime.Today.ToString("yyyy-MM-dd") + "입니다.")
```

### 식과 코드의 경계

- 단순 식은 `@식`으로 바로 출력합니다.
- **HTML과 혼합**될 때 **모호성**이 생기면 `@( … )`로 감싸 명확히 합니다.

```cshtml
<p>총합: @(100 + 200)원</p>
<p>결과: @(Model.Score >= 60 ? "합격" : "불합격")</p>
```

### 코드 블록 `@{ ... }`

```cshtml
@{
    var name = "홍길동";
    var age = 30;
}
<p>@name (@age세)</p>
```

> 팁: 코드 블록 내부에서 지역 변수/조건/반복을 구성하고, **표현식 출력은 `@…`로**.

---

## 모델 지시문 `@model`과 `@Model`

### 모델 지정

```cshtml
@model MyApp.Models.Product
<h2>@Model.Name</h2>
<p>가격: @Model.Price 원</p>
```

- 뷰/페이지 상단에서 **단일 모델 타입**을 지정합니다.
- 페이지 모델(Razor Pages)에서는 `@page`와 함께 사용됩니다.

### ViewData/ViewBag/TempData(간단 비교)

```cshtml
@{
    ViewData["Title"] = "상품";
    // ViewBag.Title = "상품"; // 동등 사용 가능
}
<title>@ViewData["Title"]</title>
```
- **ViewData**: `Dictionary<string, object?>`
- **ViewBag**: `dynamic` 래퍼
- **TempData**: **리다이렉트 이후 1회성** 전달에 사용

> 실전 권장: **강한 형식의 `@model` 우선**, 보조로 `ViewData/TempData`.

---

## 흐름 제어: 반복문/조건문/스위치/for/while

### foreach

```cshtml
@model List<string>
<ul>
@foreach (var item in Model)
{
    <li>@item</li>
}
</ul>
```

### for/while

```cshtml
@{
    for (var i = 0; i < 3; i++)
    {
        <span>@i</span>
    }
}
```

### switch

```cshtml
@{
    var grade = "A";
}
@switch (grade)
{
    case "A":
        <strong>우수</strong>
        break;
    case "B":
        <span>보통</span>
        break;
    default:
        <span>미달</span>
        break;
}
```

---

## 출력과 인코딩 — Html.Raw, IHtmlContent, 보안

### 기본 인코딩

- Razor는 **기본적으로 HTML 인코딩**을 수행합니다.

```cshtml
@("<script>alert('XSS')</script>")  <!-- 안전하게 문자열로 렌더링됨 -->
```

### Html.Raw(주의)

```cshtml
@Html.Raw("<strong>굵게</strong>")
```
- **신뢰 가능한 콘텐츠만** Raw로 렌더링하십시오. 사용자 입력 그대로 Raw 출력은 XSS 위험.

### IHtmlContent

- Tag Helper/뷰 컴포넌트가 반환하는 HTML은 **IHtmlContent**로 안전하게 처리됩니다.

---

## 주석, 문자 그대로, 이스케이프

### Razor 주석(클라이언트에 미노출)

```cshtml
@* 이건 Razor 주석입니다. HTML에는 표시되지 않음 *@
```

### HTML 주석(클라이언트에 노출)

```html
<!-- 이건 HTML 주석입니다 -->
```

### `@` 이스케이프

```cshtml
@@user  <!-- 결과: @user -->
```

---

## 디렉티브 모음 — 실제로 자주 쓰는 것들

### `@page` (Razor Pages에서 라우팅)

```cshtml
@page "{id:int:min(1)}"
@model DetailsModel
<h1>상품 #@Model.Id</h1>
```

### `@using`, `@inject`, `@addTagHelper`

```cshtml
@using MyApp.Services
@inject IClock Clock   <!-- DI로 서비스 주입 -->
현재 시간: @Clock.UtcNow

@addTagHelper *, Microsoft.AspNetCore.Mvc.TagHelpers
```

> `_ViewImports.cshtml`에서 공통으로 선언해 반복을 줄입니다.

### `@namespace`, `@inherits` (고급)

```cshtml
@namespace MyApp.Pages
@inherits Microsoft.AspNetCore.Mvc.Razor.RazorPage<dynamic>
```

### `@section` / `@RenderSection`

- 레이아웃과 함께 사용(아래 7장 참조).

---

## 레이아웃, ViewStart, ViewImports

### 레이아웃 적용

`Pages/Shared/_Layout.cshtml`
```cshtml
<!DOCTYPE html>
<html>
<head>
    <title>@ViewData["Title"] - MyApp</title>
</head>
<body>
<header>
    <a asp-page="/Index">홈</a>
</header>
<main role="main" class="container">
    @RenderBody()
</main>
@RenderSection("Scripts", required: false)
</body>
</html>
```

페이지에서:
```cshtml
@{
    Layout = "_Layout";
    ViewData["Title"] = "Products";
}
<h1>Products</h1>
```

### `_ViewStart.cshtml`

```cshtml
@{
    Layout = "_Layout";
}
```
- 모든 뷰/페이지에 기본 레이아웃 적용.

### `_ViewImports.cshtml`

```cshtml
@using MyApp.Models
@addTagHelper *, Microsoft.AspNetCore.Mvc.TagHelpers
@namespace MyApp.Pages
```
- 공통 `using`, Tag Helper, 네임스페이스를 **한 곳에서 선언**.

---

## 부분 뷰(Partial)와 뷰 컴포넌트(ViewComponent)

### Partial

`Pages/Shared/_ProductRow.cshtml`
```cshtml
@model MyApp.Models.Product
<tr>
  <td>@Model.Id</td>
  <td>@Model.Name</td>
  <td>@Model.Price</td>
</tr>
```

사용:
```cshtml
<table>
  <tbody>
    @foreach (var p in Model.Products)
    {
        <partial name="_ProductRow" model="p" />
    }
  </tbody>
</table>
```

### ViewComponent

`ViewComponents/SummaryViewComponent.cs`
```csharp
using Microsoft.AspNetCore.Mvc;

public class SummaryViewComponent : ViewComponent
{
    public IViewComponentResult Invoke(int total) => View(total);
}
```

`Views/Shared/Components/Summary/Default.cshtml`
```cshtml
@model int
<div>총 상품 수: @Model</div>
```

호출:
```cshtml
@await Component.InvokeAsync("Summary", new { total = Model.Products.Count })
```

> Partial은 **간단 재사용**, ViewComponent는 **데이터/로직 캡슐화**가 필요할 때.

---

## Tag Helper 기초 — Razor 문법과의 만남

> Tag Helper는 2편에서 폼/검증과 함께 자세히 다루지만, **Razor 문법과 동시에 이해**하면 좋습니다.

```cshtml
<label asp-for="Product.Name"></label>
<input asp-for="Product.Name" class="form-control" />
<span asp-validation-for="Product.Name"></span>

<a asp-page="/Products/Edit" asp-route-id="@p.Id">수정</a>
```
- `asp-for`는 **강한 형식 연결**을 제공하여 리팩터링 친화적입니다.
- `asp-page`, `asp-controller`, `asp-action` 등으로 **라우팅 안전성**을 확보합니다.

---

## 비동기, `await`, 비동기 부분 렌더링

### View에서 async 호출

```cshtml
@await Html.PartialAsync("_ProductRow", p)
@await Component.InvokeAsync("Summary", new { total = 10 })
```

### 페이지 모델/컨트롤러에서 async

```csharp
public async Task<IActionResult> OnGetAsync()
{
    Products = await _db.Products.AsNoTracking().ToListAsync();
    return Page();
}
```

> 지연/병렬 처리를 적시에 사용하면 **응답 지연을 줄이면서** 서버 스레드를 절약할 수 있습니다.

---

## 로컬라이제이션(뷰 로컬라이저)와 Razor

`_ViewImports.cshtml` 등에서:
```cshtml
@using Microsoft.AspNetCore.Mvc.Localization
@inject IViewLocalizer L
```

사용:
```cshtml
<h2>@L["Products"]</h2>
<p>@L["Price: {0}", Model.Price]</p>
```

> 강한 형식 리소스(소스 제너레이터)와 함께 사용하면 **다국어 지원**이 체계적입니다.

---

## HTML 특성 렌더링, 조건부 특성, null 병합/조건

### 조건부 특성

```cshtml
@{
    bool disabled = true;
}
<input type="text" @(disabled ? "disabled" : "") />
```

### null 병합/조건 연산자

```cshtml
<p>@(Model?.Name ?? "이름 없음")</p>
```

### 문자열 보간

```cshtml
<p>가격: @($"{Model.Price:n0} 원")</p>
```

---

## 디버깅과 오류 메시지 이해

- **컴파일 오류**: `.cshtml`은 **빌드시 C#으로 변환**되어 컴파일됩니다. 라인/컬럼을 보고 해당 블록을 점검.
- **런타임 예외**: NullReferenceException 등은 **`@Model`/바인딩 경로**를 우선 확인.
- **미리보는 팁**: 복잡한 표현은 `@{ var x = ... }`로 분리 후 `@x`로 출력해 원인을 좁힙니다.

---

## 성능/유지보수 팁

1) **Partial/Component의 과다 중첩**은 렌더링 비용 증가 → 캐시/조합 최적화
2) **ViewModel** 사용으로 뷰 전용 데이터를 한 번에 전달 → 데이터 접근 명확화
3) **AsNoTracking**으로 읽기 전용 쿼리 가벼움 확보
4) **섹션/레이아웃 체계화**로 페이지 간 일관성 유지
5) **Tag Helper**로 라우트/네임 안전성 확보, 하드코딩 링크 지양

---

## 2편과의 연결: 폼/바인딩/검증으로 이어지는 문법 포인트

- `@model`과 `@Model`로 **폼의 강한 형식 연결**
- `asp-for`, `asp-validation-for`, `asp-page-handler`로 **검증/핸들러 라우팅**
- `[BindProperty]`, `ModelState.IsValid`, `_ValidationScriptsPartial`은 **2편에서 심화**
- **안전한 출력**과 **검증 메시지**는 **인코딩/로컬라이제이션**과 맞물림

---

## 요약 표(확장)

| 문법 | 설명 | 예시 |
|---|---|---|
| `@` | C# 표현식 삽입 | `@DateTime.Now` |
| `@( … )` | 모호성 제거/복합식 출력 | `@(Model.Score >= 60 ? "합격" : "불합격")` |
| `@{ … }` | 코드 블록 | `@{ var x = 1; }` |
| `@model` | 모델 타입 지정 | `@model Product` |
| `@Model` | 모델 인스턴스 접근 | `@Model.Name` |
| `@page` | Razor Pages 라우팅 | `@page "{id:int}"` |
| `@using`/`@inject` | 네임스페이스/DI 주입 | `@inject IClock Clock` |
| `@addTagHelper` | Tag Helper 활성 | `@addTagHelper *, Microsoft.AspNetCore.Mvc.TagHelpers` |
| Partial | 부분뷰 재사용 | `<partial name="_Row" model="p" />` |
| ViewComponent | 데이터+뷰 캡슐화 | `@await Component.InvokeAsync("Summary")` |
| 주석 | Razor/HTML 구분 | `@* … *@`, `<!-- … -->` |
| 인코딩 | 기본 자동 | `@Html.Raw()`는 신뢰 콘텐츠만 |

---

## 실전 미니 샘플: 목록+부분뷰+뷰컴포넌트 조합

### 페이지 모델

```csharp
public class ProductsModel : PageModel
{
    private readonly AppDb _db;
    public ProductsModel(AppDb db) => _db = db;

    public IReadOnlyList<Product> Items { get; private set; } = Array.Empty<Product>();

    public async Task OnGetAsync()
        => Items = await _db.Products.AsNoTracking().OrderBy(p => p.Id).ToListAsync();
}
```

### 페이지 뷰

```cshtml
@page
@model ProductsModel
@{
    ViewData["Title"] = "Products";
}
<h1>@ViewData["Title"]</h1>

<table>
  <thead><tr><th>Id</th><th>Name</th><th>Price</th></tr></thead>
  <tbody>
    @foreach (var p in Model.Items)
    {
        <partial name="_ProductRow" model="p" />
    }
  </tbody>
</table>

@await Component.InvokeAsync("Summary", new { total = Model.Items.Count })

@section Scripts {
    <!-- 2편에서 폼 검증 스크립트를 여기 넣게 됩니다 -->
}
```

### 부분뷰

```cshtml
@model MyApp.Models.Product
<tr>
  <td>@Model.Id</td>
  <td>@Model.Name</td>
  <td>@($"{Model.Price:n0}")</td>
</tr>
```

### 뷰 컴포넌트

```csharp
public class SummaryViewComponent : ViewComponent
{
    public IViewComponentResult Invoke(int total) => View(total);
}
```

```cshtml
@model int
<div>총 상품 수: @Model</div>
```

---

## 마무리

- Razor는 **C# 표현식 → 코드 블록 → 디렉티브 → 재사용(Partial/Component) → 레이아웃/섹션**으로 계층적 사고가 가능합니다.
- **기본 인코딩, 모호성 방지 `@( … )`, 주석/디렉티브, 비동기 렌더링, 로컬라이제이션**을 숙지하면 **안전하고 유지보수성 높은 서버 렌더링**을 구현할 수 있습니다.
