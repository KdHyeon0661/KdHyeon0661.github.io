---
layout: post
title: AspNet - Partial View와 Layout
date: 2025-03-25 20:20:23 +0900
category: AspNet
---
# ASP.NET Core MVC — Partial View와 Layout

## View 계층 개관 (복습 + 확장)

```
_Layout.cshtml        ← 사이트 공통 마스터(헤더/푸터/네비/푸터/공통 스크립트)
 └ View (.cshtml)     ← 페이지 본문(컨트롤러 액션이 반환)
    ├ Partial View     ← 재사용 가능한 조각(카드/목록/폼 섹션/네비 조각 등)
    └ ViewComponent    ← “로직+뷰” 캡슐화(부분 데이터 조회+렌더링)
```

> 핵심 구분
> - **Layout**: “한 페이지의 뼈대”
> - **Partial**: “뷰 조각(템플릿화된 HTML)”
> - **ViewComponent**: “부분 컨트롤러+뷰” (부분 데이터 로딩/로직 포함)

---

## Layout — _Layout.cshtml 구조와 모범 사례

### 기본 위치/지정

- 기본 위치: `Views/Shared/_Layout.cshtml`
- 모든 뷰에 기본 레이아웃을 적용하려면 `Views/_ViewStart.cshtml`에서 지정

```cshtml
@* Views/_ViewStart.cshtml *@
@{
    Layout = "_Layout";
}
```

개별 뷰에서 레이아웃을 바꾸려면 해당 뷰 파일 상단에서 덮어쓰기:

```cshtml
@{
    Layout = "_AdminLayout"; // 페이지 단위로 다른 레이아웃 적용
}
```

### _Layout 기본 골격(Section/Environment/Partial 포함)

```cshtml
@* Views/Shared/_Layout.cshtml *@
@using Microsoft.AspNetCore.Mvc.Localization
@inject IViewLocalizer L

<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="utf-8" />
    <title>@ViewData["Title"] - MyApp</title>

    <environment include="Development">
        <link rel="stylesheet" href="~/css/site.css" />
    </environment>
    <environment exclude="Development">
        <link rel="stylesheet" href="~/css/site.min.css" asp-append-version="true" />
    </environment>

    @RenderSection("Head", required: false) @* 페이지별 추가 head 콘텐츠 *@
</head>
<body>
    <header>
        @await Html.PartialAsync("_Navbar")  @* 공통 네비게이션 조각 *@
    </header>

    <main role="main" class="container">
        @RenderBody()
    </main>

    <footer>
        @await Html.PartialAsync("_Footer")
        <small>&copy; @DateTime.UtcNow.Year MyApp</small>
    </footer>

    <script src="~/lib/jquery/jquery.min.js"></script>
    <script src="~/js/site.js" asp-append-version="true"></script>

    @RenderSection("Scripts", required: false) @* 페이지 전용 스크립트 자리 *@
</body>
</html>
```

**포인트**
- `@RenderBody()` : 페이지 본문 삽입 위치
- `@RenderSection()` : 페이지에서 필요한 “추가 구역(Head/Scripts 등)” 제공
- `<environment>` Tag Helper : 개발/운영별 리소스 분기
- `@inject IViewLocalizer` : 뷰/레이아웃에서 다국어 문자열 사용

---

## Partial View — 4가지 호출 방식과 데이터 전달

### 위치/명명

- 위치: `Views/Shared/` 또는 `Views/{Controller}/`
- 컨벤션: 파일명 앞에 `_` 접두(예: `_ProductCard.cshtml`)

### 호출 방식 비교

| 방식 | 문법 | 반환/동작 | 비고 |
|---|---|---|---|
| HTML Helper (Sync) | `@Html.Partial("_ProductCard", model)` | HTML 문자열 반환 | 동기 |
| HTML Helper (Async) | `@await Html.PartialAsync("_ProductCard", model)` | Task\<IHtmlContent> | 권장 |
| **Partial Tag Helper** | `<partial name="_ProductCard" model="product" />` | 태그 헬퍼 | **가독성↑** |
| RenderPartial | `@{await Html.RenderPartialAsync("_ProductCard", model);}` | 직접 출력(스트림) | 반환값 없음 |

> 실무에선 **Partial Tag Helper** 또는 **`Html.PartialAsync`**를 주로 사용.

### Partial 예시(카드 구성)

```cshtml
@* Views/Shared/_ProductCard.cshtml *@
@model ProductCardVm

<div class="card">
  <div class="card-header">
    <strong>@Model.Name</strong>
  </div>
  <div class="card-body">
    <p class="text-muted">@Model.Description</p>
    <div>가격: @Model.Price.ToString("C")</div>
  </div>
</div>
```

호출(목록에서 반복 렌더링):

```cshtml
@* Views/Products/Index.cshtml *@
@model IEnumerable<ProductCardVm>
@{
    ViewData["Title"] = "상품 목록";
}

<h2>@ViewData["Title"]</h2>

@* Tag Helper 방식 *@
@foreach (var p in Model)
{
    <partial name="_ProductCard" model="p" />
}

@* 또는 Helper 방식 *@
@* @foreach (var p in Model) { @await Html.PartialAsync("_ProductCard", p) } *@
```

### Partial에 추가 데이터(ViewData) 전달

```cshtml
@{
    var vd = new ViewDataDictionary(ViewData)
    {
        ["ShowPrice"] = true
    };
}
<partial name="_ProductCard" model="p" view-data="vd" />
```

Partial 내에서:

```cshtml
@{ var show = (bool?)ViewData["ShowPrice"] ?? false; }
@if (show) { <span>@Model.Price.ToString("C")</span> }
```

---

## Partial vs Layout vs ViewComponent — 언제 무엇을?

| 대상 | 사용 상황 | 장점 | 주의 |
|---|---|---|---|
| Layout | 페이지 공통 뼈대 | 일관된 구조, 중복 제거 | Section/Body 위치 설계 |
| Partial View | **단순한 뷰 조각** 재사용(데이터는 **이미** 준비됨) | 단순/빠름 | 데이터 로딩 로직을 포함하지 않음 |
| **ViewComponent** | 부분 데이터 로딩 + 렌더링이 함께 필요(“미니 컨트롤러”) | 테스트 용이, 독립성↑ | 파일 2개(클래스/뷰), 학습 요소 |

### ViewComponent 간단 예시

```csharp
// Components/CartSummaryViewComponent.cs
public class CartSummaryViewComponent : ViewComponent
{
    private readonly ICartService _cart;
    public CartSummaryViewComponent(ICartService cart) => _cart = cart;

    public async Task<IViewComponentResult> InvokeAsync()
    {
        var count = await _cart.GetItemCountAsync(HttpContext.RequestAborted);
        return View("Default", count); // Views/Shared/Components/CartSummary/Default.cshtml
    }
}
```

```cshtml
@* Views/Shared/Components/CartSummary/Default.cshtml *@
@model int
<a href="/cart">장바구니 (@Model)</a>
```

레이아웃/뷰에서 호출:

```cshtml
@await Component.InvokeAsync("CartSummary")
@* 또는 Tag Helper *@
<vc:cart-summary />
```

> **로직이 필요한 재사용 블록**은 Partial 대신 ViewComponent를 선호.

---

## Section — 페이지별 스크립트/스타일 주입

### Layout에 정의

```cshtml
@RenderSection("Head", required: false)
@RenderSection("Scripts", required: false)
```

### View에서 채우기

```cshtml
@section Head {
  <link rel="stylesheet" href="~/css/details.css" />
}

@section Scripts {
  <script src="~/js/details.js" asp-append-version="true"></script>
}
```

> `required: true`이면 해당 섹션을 뷰에서 반드시 제공해야 하며, 미제공 시 예외 발생.

---

## _ViewStart / _ViewImports — 전역 Razor 설정

### _ViewStart.cshtml

- 모든 뷰에 **기본 Layout** 지정
- 뷰 단에서 필요 시 덮어쓰기 가능

```cshtml
@{
    Layout = "_Layout";
}
```

### _ViewImports.cshtml

- 네임스페이스, Tag Helper, 공통 using 지정

```cshtml
@using MyApp.Models
@using MyApp.ViewModels
@addTagHelper *, Microsoft.AspNetCore.Mvc.TagHelpers
```

> Partial/뷰 어디서나 Tag Helper를 사용할 수 있게 함.

---

## Partial로 폼 조립하기 — 중첩 폼 주의 & 검증 스크립트

### 중첩 폼 금지

HTML 표준상 **폼 안에 폼**은 허용되지 않는다.
“부품 폼”을 Partial로 분리할 수는 있지만, **최상위 폼은 하나**여야 한다.

```cshtml
@model ProductEditVm

<form asp-action="Edit" method="post">
  <partial name="_ProductEditFields" model="Model" />
  <button type="submit" class="btn btn-primary">저장</button>
</form>

@section Scripts {
  <partial name="_ValidationScriptsPartial" />
}
```

```cshtml
@* Views/Shared/_ProductEditFields.cshtml *@
@model ProductEditVm
<div class="mb-3">
  <label asp-for="Name" class="form-label"></label>
  <input asp-for="Name" class="form-control" />
  <span asp-validation-for="Name" class="text-danger"></span>
</div>
<div class="mb-3">
  <label asp-for="Price" class="form-label"></label>
  <input asp-for="Price" class="form-control" />
  <span asp-validation-for="Price" class="text-danger"></span>
</div>
```

### 안티포저리 토큰

- Razor `form` Tag Helper는 자동으로 토큰을 넣지만, 순수 `<form>`을 직접 쓸 땐 `@Html.AntiForgeryToken()` 추가.

```cshtml
<form method="post">
  @Html.AntiForgeryToken()
  ...
</form>
```

---

## Partial Tag Helper 고급 옵션

```cshtml
<partial name="_ProductCard"
         model="p"
         for="Model.Products"
         view-data="ViewData" />
```

- `for` : `EditorTemplates`/`DisplayTemplates`와 함께 사용되는 시나리오에서 유용
- `view-data` : 뷰 데이터 전달

---

## EditorTemplates/DisplayTemplates — Partial과 다른 “형식 템플릿”

- 위치: `Views/Shared/EditorTemplates/` 또는 `Views/{Controller}/EditorTemplates/`
- 파일명 = 타입명(또는 별칭)

예) `Views/Shared/DisplayTemplates/ProductCardVm.cshtml`:

```cshtml
@model ProductCardVm
<div class="card shadow-sm">
  <div class="card-body">
    <strong>@Model.Name</strong>
    <div>@Model.Price.ToString("C")</div>
  </div>
</div>
```

사용:

```cshtml
@Html.DisplayFor(m => m.Product)            @* 단일 객체 *@
@Html.DisplayFor(m => m.Products)           @* 컬렉션: 반복 렌더링 *@
@Html.EditorFor(m => m.ProductEdit)         @* 편집 템플릿 *@
```

> **템플릿 기반**이라 호출부가 간결해지고 컬렉션 자동 반복이 장점.

---

## 캐시/환경 분기/조건부 렌더링

### Cache Tag Helper (부분 캐싱)

```cshtml
<cache expires-after="00:05:00"
       vary-by-route="id"
       vary-by-query="q">
  <partial name="_ProductCard" model="Model" />
</cache>
```

- 핫한 목록/요약 카드 조각에 적용하면 성능↑

### Environment Tag Helper

- 개발/운영에 따라 리소스 분리(이미 레이아웃에서 사용)

### 권한/상태에 따른 조건부 노출

```cshtml
@if (User.IsInRole("Admin")) {
  <partial name="_AdminNav" />
}
```

---

## 동적 레이아웃 전환 패턴

### _ViewStart에서 조건 분기

```cshtml
@inject Microsoft.AspNetCore.Http.IHttpContextAccessor HttpAcc

@{
    var path = HttpAcc.HttpContext?.Request?.Path.Value ?? string.Empty;
    Layout = path.StartsWith("/Admin", StringComparison.OrdinalIgnoreCase)
        ? "_AdminLayout"
        : "_Layout";
}
```

> 경로/도메인/쿠키/테넌트에 따라 다른 레이아웃을 적용 가능.

### 액션 단에서 ViewBag/RouteData로 전달하여 뷰에서 선택

- 컨트롤러에서 `ViewBag.UseAdmin = true;` 후 뷰에서 `Layout = ViewBag.UseAdmin ? "_AdminLayout" : "_Layout";`

---

## 메뉴 활성화(Active) — 현재 라우트 반영

```cshtml
@* Views/Shared/_Navbar.cshtml *@
@inject Microsoft.AspNetCore.Mvc.Rendering.IHtmlHelper<dynamic> H
@{
    string ctl = (string)ViewContext.RouteData.Values["controller"]!;
    string act = (string)ViewContext.RouteData.Values["action"]!;
    string Active(string c, string a = "Index") => (ctl == c && act == a) ? "active" : "";
}
<nav class="nav">
  <a class="nav-link @Active("Home")" asp-controller="Home" asp-action="Index">홈</a>
  <a class="nav-link @Active("Products","Index")" asp-controller="Products" asp-action="Index">상품</a>
</nav>
```

---

## Areas와 레이아웃/Partial 배치

구조:

```
/Areas
  /Admin
    /Views
      /Shared/_Layout.cshtml
      /Home/Index.cshtml
/Views
  /Shared/_Layout.cshtml   ← 기본 사이트 레이아웃
```

- Area 내부 전용 레이아웃/Partial을 `Areas/{Area}/Views/Shared/`에 둔다.
- Area 뷰에서 `_ViewStart.cshtml`로 Area 전용 레이아웃을 지정하면 깔끔.

---

## 다국어(Localization) — 뷰/레이아웃에서 문자열

```cshtml
@* _Layout.cshtml *@
@inject Microsoft.AspNetCore.Mvc.Localization.IViewLocalizer L
<title>@L["SiteTitle"] - @ViewData["Title"]</title>
<nav>
  <a>@L["Menu_Home"]</a>
</nav>
```

- 리소스 파일: `Resources/Views/Shared/_Layout.ko.resx`, `_Layout.en.resx` 등
- 다국어 URL/컬처 라우팅은 라우팅/미들웨어와 함께 구성(별도 글 참고)

---

## 실전 예시 — “목록 + 카드 + 상세 + 레이아웃 섹션 스크립트” 종합

### ViewModel

```csharp
public record ProductCardVm(int Id, string Name, string Description, decimal Price);
public record ProductListVm(IReadOnlyList<ProductCardVm> Items, int Page, int Size, int Total);
```

### 목록 뷰(부분 카드 사용 + 섹션 스크립트)

```cshtml
@* Views/Products/Index.cshtml *@
@model ProductListVm
@{
    ViewData["Title"] = "상품 목록";
}

<h2>@ViewData["Title"]</h2>

<div class="grid">
@foreach (var p in Model.Items)
{
    <partial name="_ProductCard" model="p" />
}
</div>

<nav class="mt-3">
  @* 간단 페이지네이션 *@
  <a asp-action="Index" asp-route-page="@(Model.Page-1)" class="btn btn-sm btn-outline-secondary" disabled="@(Model.Page<=1)">이전</a>
  <a asp-action="Index" asp-route-page="@(Model.Page+1)" class="btn btn-sm btn-outline-secondary" disabled="@(Model.Page*Model.Size>=Model.Total)">다음</a>
</nav>

@section Scripts {
  <script>
    // 카드 클릭 시 상세로 이동(델리게이션 예시)
    document.querySelector('.grid')?.addEventListener('click', e => {
      const card = e.target.closest('.card');
      if (!card) return;
      const id = card.getAttribute('data-id');
      if (id) location.href = `/Products/Details/${id}`;
    });
  </script>
}
```

### 카드 Partial(데이터 속성으로 식별자 부여)

```cshtml
@* Views/Shared/_ProductCard.cshtml *@
@model ProductCardVm
<div class="card mb-3" data-id="@Model.Id">
  <div class="card-header"><strong>@Model.Name</strong></div>
  <div class="card-body">
    <p class="text-muted">@Model.Description</p>
    <div class="fw-bold">가격: @Model.Price.ToString("C")</div>
  </div>
</div>
```

---

## 모듈화 팁 & 성능/안전 체크리스트

- **모듈화**
  - 작은 Partial로 쪼개고 조립(카드/리스트/필터/폼 섹션)
  - 데이터 로딩이 필요한 경우 **ViewComponent**로 승격
  - **Editor/Display 템플릿**으로 폼/표시를 타입 기반으로 단순화
- **성능**
  - 리스트/요약 Partial에 **Cache Tag Helper** 적용 검토
  - 큰 페이지의 경우 **Section 분리 + 지연 로딩**(필요 시)
- **안전/품질**
  - 폼은 반드시 **안티포저리 토큰**
  - **중첩 폼 금지**, 부분 폼은 “필드 집합(Partial)”로만 구성
  - 레이아웃 스크립트 로딩 순서/중복 확인(검증 스크립트는 페이지 말미)
  - Partial/뷰에서 널 참조 방지(`@Model?....`)
- **유지보수**
  - `_ViewImports.cshtml`에 Tag Helper 설정 통일
  - `_ViewStart.cshtml`로 기본 레이아웃 일관 유지
  - Areas로 **영역별 테마/레이아웃** 분리

---

## 자주 겪는 실수와 해결책

| 증상 | 원인 | 해결 |
|---|---|---|
| “The following sections have been defined but have not been rendered” | Layout에 없는 섹션을 뷰에서 정의 | 레이아웃에 `@RenderSection` 추가 또는 뷰에서 섹션 제거 |
| “Section not defined” 예외 | `required:true` 섹션을 뷰에서 누락 | 뷰에 해당 섹션 작성 또는 `required:false` |
| Partial에서 모델 널 예외 | 호출부에서 모델 전달 누락 | `<partial name="..." model="...">` 확인 |
| 중첩 폼 이슈 | Partial에 `<form>` 포함 후 상위에도 `<form>` 존재 | 부분 폼은 필드만 포함, 최상위 뷰에서 폼 한 번만 |
| 검증 메시지 안 나옴 | 검증 스크립트 누락 | 뷰의 `@section Scripts { <partial name="_ValidationScriptsPartial" /> }` 확인 |
| 환경별 CSS/JS 잘못 로드 | `<environment>` 설정 오류 | `include/exclude` 조건과 경로 점검 |

---

## 마무리 요약

| 개념 | 핵심 |
|---|---|
| Layout | 페이지 공통 뼈대(`@RenderBody`, `@RenderSection`) |
| Partial View | 재사용 가능한 HTML 조각, **데이터는 이미 전달된 상태** |
| Partial Tag Helper | `<partial name="..." model="...">` 간결/권장 |
| ViewComponent | 부분 데이터 로딩+뷰 렌더링(“미니 컨트롤러”) |
| _ViewStart / _ViewImports | 기본 레이아웃/Tag Helper/using 일괄 설정 |
| Editor/Display 템플릿 | 타입 기반 템플릿화로 폼/표시 단순화 |
| Cache/Environment/Sections | 성능/환경/페이지별 스크립트 관리 |
