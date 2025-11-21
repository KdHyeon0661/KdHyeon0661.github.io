---
layout: post
title: AspNet - Razor 문법 고급 요약
date: 2025-05-10 22:20:23 +0900
category: AspNet
---
# Razor 문법 **고급편**

## 전략 요약 — 언제 **무엇**을 쓰나?

| 상황 | 권장 도구 | 이유 |
|---|---|---|
| 단순한 반복 UI 조각 | **Partial View** | 빠르고 간단 |
| UI + 데이터 조회/DI + 단위 테스트 | **ViewComponent** | 캡슐화·DI 가능·테스트 쉬움 |
| 속성/마크업 수준의 재사용(클래스/속성 주입) | **Custom Tag Helper** | HTML 친화적 DSL |
| 뷰 내부 소규모 계산·서식 | **`@functions` / `@code`** | 한 파일 한정의 경량 헬퍼 |
| 전역 포맷/로직 | **HtmlHelper 확장 메서드** | 전역 재사용·강한 컴파일 안정성 |

---

## View 내부 **Custom Helper** (경량/한 파일 한정)

### `@functions`로 간단 헬퍼

```razor
@functions {
    public string FormatCurrency(decimal amount)
        => string.Format("{0:N0}원", amount);

    public IHtmlContent Badge(string text, string type = "secondary")
        => new HtmlString($"<span class=\"badge bg-{type}\">{text}</span>");
}

<p>가격: @FormatCurrency(Model.Price)</p>
@Badge("NEW", "primary")
```

- **장점**: 가장 빠르고 간단.
- **제한**: 현재 `.cshtml` 파일에서만 재사용. 로직이 커지면 분리 권장.

### Razor Pages의 `@functions` vs MVC `@section`

- `@functions`는 **C# 멤버**를 정의(메서드/필드).
- *크게 복잡하면* **ViewModel/Service**로 승격 후 주입(테스트성↑).

---

## 재사용을 **전역화**: HtmlHelper 확장 메서드

### `IHtmlHelper` 확장 (전역 재사용)

```csharp
// /Infrastructure/HtmlHelperExtensions.cs
using Microsoft.AspNetCore.Html;
using Microsoft.AspNetCore.Mvc.Rendering;

namespace MyApp.Infrastructure;

public static class HtmlHelperExtensions
{
    public static IHtmlContent HighlightIf(this IHtmlHelper html, bool condition, string text)
        => new HtmlString($"<span class=\"{(condition ? "highlight" : "")}\">{text}</span>");

    public static string Currency(this IHtmlHelper html, decimal amount, string symbol = "₩")
        => $"{symbol}{amount:N0}";
}
```

```razor
@* _ViewImports.cshtml *@
@using MyApp.Infrastructure
```

```razor
<p>@Html.Currency(Model.Price)</p>
@Html.HighlightIf(Model.IsHot, "인기 상품")
```

- 반환 타입은 **`IHtmlContent`**가 안전(자동 인코딩/Raw 제어).
- **장점**: 강타입, 전역 재사용, 뷰 로직 단순화.
- **팁**: 문자열 반환은 Razor 기본 인코딩. Raw HTML은 `IHtmlContent`로.

---

## **Partial View** — 뷰 조각 재사용

### 기본

```razor
@* Views/Shared/_ProductCard.cshtml *@
@model Product

<div class="card">
  <h3 class="h5">@Model.Name</h3>
  <p class="text-muted">가격: @Model.Price.ToString("N0") 원</p>
  <a class="btn btn-sm btn-primary"
     asp-controller="Products" asp-action="Detail" asp-route-id="@Model.Id">자세히</a>
</div>
```

```razor
@* 호출 측 *@
<partial name="_ProductCard" model="product" />
@* 혹은 *@
@await Html.PartialAsync("_ProductCard", product)
```

- **장점**: 빠르고 쉬움.
- **주의**: 데이터/DI/로깅 등 로직이 커지면 **ViewComponent**로 이동.

### Partial vs Section

- **Partial**: UI 조각 자체.
- **Section**: 레이아웃이 `@RenderSection`으로 **슬롯** 제공 → 각 뷰가 채움.

---

## **ViewComponent** — DI·로직·테스트 가능한 컴포넌트

### 예제: 장바구니 요약

```csharp
// /ViewComponents/CartSummaryViewComponent.cs
using Microsoft.AspNetCore.Mvc;

public class CartSummaryViewComponent : ViewComponent
{
    private readonly ICartService _cart;
    public CartSummaryViewComponent(ICartService cart) => _cart = cart;

    public async Task<IViewComponentResult> InvokeAsync()
    {
        var model = await _cart.GetSummaryAsync(HttpContext.User);
        return View(model); // Views/Shared/Components/CartSummary/Default.cshtml
    }
}
```

```razor
@* Views/Shared/Components/CartSummary/Default.cshtml *@
@model CartSummary
<div class="cart-summary">
  <span>수량: @Model.Count</span> / <span>합계: @Model.Total:N0 원</span>
  <a asp-page="/Cart">장바구니</a>
</div>
```

```razor
@* 호출 *@
@await Component.InvokeAsync("CartSummary")
```

- **장점**: DI/데이터 조회/조건 분기/캐싱/로그가 **컴포넌트 단위**로 캡슐화.
- **테스트**: 생성자에 **서비스 Mock** 주입 → **단위 테스트** 용이.

---

## **Custom Tag Helper** — HTML 친화 DSL

> 속성/마크업 레벨에서 재사용/규칙을 강제하고 싶을 때.

### 조건 클래스 자동 부여 Tag Helper

```csharp
// /TagHelpers/WhenClassTagHelper.cs
using Microsoft.AspNetCore.Razor.TagHelpers;

[HtmlTargetElement(Attributes = "class-when, class-name")]
public class WhenClassTagHelper : TagHelper
{
    [HtmlAttributeName("class-when")]
    public bool Condition { get; set; }

    [HtmlAttributeName("class-name")]
    public string ClassName { get; set; } = "";

    public override void Process(TagHelperContext context, TagHelperOutput output)
    {
        if (!Condition) return;
        var cls = output.Attributes["class"]?.Value?.ToString();
        var merged = string.IsNullOrWhiteSpace(cls) ? ClassName : $"{cls} {ClassName}";
        output.Attributes.SetAttribute("class", merged);
    }
}
```

```razor
@* _ViewImports.cshtml *@
@addTagHelper *, Microsoft.AspNetCore.Mvc.TagHelpers
@addTagHelper *, MyApp   @* 어셈블리명 *@
```

```razor
<button class="btn" class-when="@(Model.IsHot)" class-name="btn-danger">구매</button>
```

- **효과**: 조건 로직을 **HTML 속성**으로 승격 → 뷰 가독성↑, 중복↓.

### 출력 자체를 억제 (`SuppressOutput`)

```csharp
[HtmlTargetElement("if-claims")]
public class IfClaimsTagHelper : TagHelper
{
    public string? Claim { get; set; }
    public string? Value { get; set; }

    public override void Process(TagHelperContext ctx, TagHelperOutput output)
    {
        // (실전: HttpContext.User.Claims 검사)
        var show = /* 사용자 권한 검사 로직 */ false;
        if (!show) output.SuppressOutput();
    }
}
```

```razor
<if-claims claim="Role" value="Admin">
  <a asp-page="/Admin">관리자</a>
</if-claims>
```

- **장점**: 권한/상태 조건부 렌더링을 마크업에서 선언적으로.

### 자주 쓰는 패턴

- **데이터 포맷**(예: 통화/날짜) → `<span money-for="Model.Price">…`
- **국제화** → `<loc key="Welcome" />` (IStringLocalizer 래핑)
- **Feature Flag** → `<feature name="NewHeader">…`

> Tag Helper는 **디자인 시스템**에 유리(팀 내 HTML 규칙 강제, 일관성↑).

---

## 조건부 HTML 생성 — 깨끗한 패턴

### 클래스/속성 조건

```razor
<button class="btn @(Model.Enabled ? "btn-primary" : "btn-outline-secondary")"
        disabled="@(Model.Enabled ? null : "disabled")">저장</button>
```

### 빌더 패턴으로 속성 합성(뷰 내부)

```razor
@{
    var css = new List<string> {"card"};
    if (Model.IsNew) css.Add("border-primary");
    var cls = string.Join(" ", css);
}
<div class="@cls">내용</div>
```

### HTML 청크를 조건 조립

```razor
@{
    IHtmlContentBuilder b = new HtmlContentBuilder();
    b.AppendHtml("<ul>");
    foreach (var item in Model.Items)
    {
        b.AppendHtml($"<li class=\"{(item.IsSelected ? "selected" : "")}\">{HtmlEncoder.Default.Encode(item.Name)}</li>");
    }
    b.AppendHtml("</ul>");
}
@b
```
- 복잡한 문자열 합성은 **`HtmlContentBuilder`**를 쓰면 안전/효율 좋음.

---

## 성능 & 보안 **핵심 체크리스트**

### 성능

- **부분 뷰 남발** 주의: 큰 루프 내부의 Partial 렌더링은 비용↑ → ViewComponent/Batch 처리/캐시 고려.
- **정적 리소스**: `asp-append-version`로 캐시 제어, CDN·HTTP/2, 압축(브로틀리/Gzip).
- **Razor Runtime Compilation**은 개발용. 운영에서는 **미리 컴파일**(기본) 사용.

### 보안

- 기본은 **HTML 인코딩**. 임의 HTML은 `Html.Raw`나 `IHtmlContent`로 **의도적** 사용.
- **Anti-forgery**: form Tag Helper 사용 시 자동 삽입. Ajax면 헤더에 토큰 포함.
- **출처 데이터 신뢰 금지**: 사용자 입력 포함 조립 시 반드시 인코딩.
- **Custom Tag Helper**에서 외부 입력을 그대로 속성/HTML에 삽입 금지 → 검증/인코딩.

---

## 테스트 전략 (단위/통합)

### ViewComponent 단위 테스트

```csharp
public class CartSummaryViewComponentTests
{
    [Fact]
    public async Task Render_Count_And_Total()
    {
        var mock = new Mock<ICartService>();
        mock.Setup(s => s.GetSummaryAsync(It.IsAny<ClaimsPrincipal>()))
            .ReturnsAsync(new CartSummary { Count = 3, Total = 12000 });

        var vc = new CartSummaryViewComponent(mock.Object);
        var result = await vc.InvokeAsync() as ViewViewComponentResult;

        Assert.NotNull(result);
        var model = Assert.IsType<CartSummary>(result!.ViewData.Model);
        Assert.Equal(3, model.Count);
        Assert.Equal(12000, model.Total);
    }
}
```

### Razor 출력 검증 팁

- **View 없음** 로직은 **ViewModel/Service**로 빼서 테스트.
- 통합 테스트에서 `WebApplicationFactory`로 **실제 HTML 응답** 검증.

---

## 국제화(Localization)와 조합

### IStringLocalizer 헬퍼화

```csharp
public static class HtmlL10nExtensions
{
    public static IHtmlContent L(this IHtmlHelper html, IViewLocalizer loc, string key, object? args = null)
        => new HtmlString(loc[key, args].Value);
}
```

```razor
@inject IViewLocalizer Lc
<h2>@Html.L(Lc, "WelcomeTitle")</h2>
```

- UI 문자열은 **Tag Helper**로 만드는 것도 좋다: `<loc key="WelcomeTitle" />`.

---

## 예제 — “조건부 뱃지 · 카드 · 구매 버튼” 컴포넌트 풀셋

### HtmlHelper 확장(통화/뱃지)

```csharp
public static class UiHelpers
{
    public static string Won(this IHtmlHelper _, decimal money) => $"{money:N0}원";

    public static IHtmlContent Badge(this IHtmlHelper _, string text, string type="secondary")
        => new HtmlString($"<span class=\"badge bg-{type}\">{HtmlEncoder.Default.Encode(text)}</span>");
}
```

### Tag Helper(조건 클래스)

```csharp
[HtmlTargetElement(Attributes = "if, add-class")]
public class IfClassTagHelper : TagHelper
{
    [HtmlAttributeName("if")] public bool Condition { get; set; }
    [HtmlAttributeName("add-class")] public string AddClass { get; set; } = "";

    public override void Process(TagHelperContext ctx, TagHelperOutput output)
    {
        if (!Condition) return;
        var cls = output.Attributes["class"]?.Value?.ToString();
        var merged = string.IsNullOrWhiteSpace(cls) ? AddClass : $"{cls} {AddClass}";
        output.Attributes.SetAttribute("class", merged);
    }
}
```

### ViewComponent(재고·혜택 로직)

```csharp
public class ProductCardViewComponent : ViewComponent
{
    private readonly IStockService _stock;
    public ProductCardViewComponent(IStockService stock) => _stock = stock;

    public async Task<IViewComponentResult> InvokeAsync(Product p)
    {
        var stock = await _stock.GetAsync(p.Id);
        var vm = new ProductCardVM {
            Product = p, InStock = stock > 0, Benefit = p.Price > 100000 ? "무료 배송" : null
        };
        return View(vm);
    }
}

public class ProductCardVM
{
    public Product Product { get; set; } = default!;
    public bool InStock { get; set; }
    public string? Benefit { get; set; }
}
```

```razor
@* Views/Shared/Components/ProductCard/Default.cshtml *@
@model ProductCardVM
@inject IViewLocalizer Lc

<div class="card" if="@(Model.InStock == false)" add-class="opacity-50">
  <div class="card-body">
    <h5 class="card-title">@Model.Product.Name</h5>
    <p>@Html.Won(Model.Product.Price)</p>

    @if (!string.IsNullOrEmpty(Model.Benefit))
    {
        @Html.Badge(Model.Benefit!, "success")
    }

    @if (Model.InStock)
    {
        <form method="post" asp-page="/Cart/Add">
          <input type="hidden" name="id" value="@Model.Product.Id" />
          <button class="btn btn-primary">구매</button>
        </form>
    }
    else
    {
        @Html.Badge(Lc["SoldOut"].Value, "danger")
    }
  </div>
</div>
```

```razor
@* 호출 측 *@
@await Component.InvokeAsync("ProductCard", new { p = product })
```

- **요지**: 가격 포맷/뱃지 = `HtmlHelper` 확장, 상태별 클래스 = `Tag Helper`, 재고·혜택 로직 = **ViewComponent** 캡슐화.

---

## 폼/검증 고급 — 동적 필드 표시, 부분 유효성

### 조건부 입력 필드

```razor
@if (Model.IsCompany)
{
    <div class="mb-2">
      <label asp-for="CompanyName"></label>
      <input asp-for="CompanyName" class="form-control" />
      <span asp-validation-for="CompanyName" class="text-danger"></span>
    </div>
}
```

### 클라이언트 검증(부분 무효화)

- 「서버 검증은 항상」, 클라이언트 검증은 **조건부** 스크립트로 제어.
- 동적 표시/숨김일 때 **ModelState** 정합성 확인.

---

## 응답 조각 **캐싱**(성능)

- Output Caching(.NET 8+) 또는 Response Caching으로 **ViewComponent/Partial** 결과를 캐시.
- **키 전략**(언어, 인증, 쿼리)에 유의.

---

## 디자이너/퍼블리셔 협업 팁

- **Tag Helper**로 “디자인 규칙”을 코드화 → 마크업을 선언적으로.
- `_ViewImports.cshtml`에 디자이너가 쓸 네임스페이스·TagHelper만 노출.
- 린트 규칙(HTMLHint/Stylelint)과 함께 운용.

---

## 안티패턴 🚫

- **`Html.Raw` 남용**: 사용자 입력 포함시 XSS 위험.
- **루프 안에서 DB 호출**: 데이터는 컨트롤러/VC에서 **미리 수집**.
- **거대한 @functions**: 파일 커짐 → 확장 메서드/서비스로 승격.
- **Tag Helper에서 비즈니스 로직**: Tag Helper는 **표현/속성 규칙**에 집중.

---

## 요약

| 주제 | 한 줄 가이드 |
|---|---|
| View helper(경량) | `@functions`로 소규모 서식/계산 |
| 전역 재사용 | `HtmlHelper` 확장(문자/IHtmlContent) |
| 재사용 UI 조각 | Partial(단순) → **ViewComponent**(로직/DI/테스트) |
| 선언적 조건/규칙 | **Custom Tag Helper** (속성 머지/억제) |
| 조건부 HTML | 클래스/속성/태그 억제로 깨끗하게 |
| 성능/보안 | 캐시·인코딩·안티포저리·Partial 남용 주의 |
| 테스트 | ViewComponent 단위 테스트, 통합테스트로 렌더링 검증 |

---

### 수식 표기(블로그용)

```
$$
\text{ScoreBadge}(x) =
\begin{cases}
\text{"badge-danger"} & x \ge 90 \\
\text{"badge-warning"} & 70 \le x < 90 \\
\text{"badge-secondary"} & x < 70
\end{cases}
$$
```
> Razor 자체는 수식을 렌더링하지 않으므로 MathJax를 레이아웃에 추가해 사용.

---

이제 **Razor 고급 도구 상자**를 갖췄다.
작은 것은 `@functions`, 전역은 `HtmlHelper`, 조각은 **Partial**, 로직/DI는 **ViewComponent**, 선언적 규칙은 **Custom Tag Helper**로 설계하면,
뷰는 **간결**, 팀 규칙은 **일관**, 성능과 테스트는 **현실적**이 된다.
