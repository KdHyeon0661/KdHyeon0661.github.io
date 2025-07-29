---
layout: post
title: AspNet - Razor 문법 고급 요약
date: 2025-05-10 22:20:23 +0900
category: AspNet
---
# 🚀 Razor 문법 고급편 (Custom Helper, Component 구조, 조건부 HTML 생성 등)

---

## 1️⃣ Custom Helper 메서드 정의 및 재사용

Razor에서 공통적인 HTML 또는 텍스트 조각을 **함수처럼 재사용**하고 싶을 때 사용합니다.

---

### ✅ View에서 간단한 헬퍼 정의

```razor
@functions {
    public string FormatCurrency(decimal amount)
    {
        return String.Format("{0:N0}원", amount);
    }
}

<p>가격: @FormatCurrency(Model.Price)</p>
```

> `@functions` 블록은 해당 `.cshtml` 파일 내부에서 C# 메서드를 정의할 수 있도록 해줍니다.

---

### ✅ Layout 또는 Partial에서 재사용 가능

- 공통 Helper는 `_ViewImports.cshtml`에 공통 네임스페이스 추가
- 또는 `HtmlHelperExtensions` 클래스 형태로 별도 작성

```csharp
public static class HtmlHelperExtensions
{
    public static string HighlightIf(this IHtmlHelper html, bool condition)
    {
        return condition ? "highlight" : "";
    }
}
```

```razor
<p class="@Html.HighlightIf(Model.IsHot)">인기 상품</p>
```

> 이 경우엔 `@using YourApp.Helpers`를 `_ViewImports.cshtml`에 등록해줘야 합니다.

---

## 2️⃣ Razor Component 구조 (Partial View 대체)

MVC와 Razor Pages에서도 재사용 가능한 **UI 컴포넌트** 구조를 만들 수 있습니다.

---

### ✅ Partial View 사용

```razor
<!-- ProductCard.cshtml -->
@model Product

<div class="card">
  <h3>@Model.Name</h3>
  <p>가격: @Model.Price</p>
</div>
```

```razor
<!-- 부모 View -->
<partial name="ProductCard" model="product" />
```

---

### ✅ ViewComponent 방식 (로직 포함 컴포넌트)

```csharp
public class CartSummaryViewComponent : ViewComponent
{
    public IViewComponentResult Invoke()
    {
        var cart = HttpContext.Session.GetString("cart");
        return View("Default", cart);
    }
}
```

사용 시:

```razor
@await Component.InvokeAsync("CartSummary")
```

뷰 파일: `/Views/Shared/Components/CartSummary/Default.cshtml`

> ViewComponent는 MVC에서 컴포넌트 로직 + 뷰를 분리하여 재사용할 수 있는 강력한 기능입니다.

---

## 3️⃣ 조건부 HTML 생성 (고급 Razor 활용)

복잡한 조건을 기반으로 동적으로 HTML을 렌더링할 수 있어요.

---

### ✅ `class`에 조건 삽입

```razor
<p class="@(Model.IsActive ? "text-success" : "text-danger")">상태</p>
```

또는 C# 변수 활용:

```razor
@{
    var css = Model.Count > 5 ? "highlight" : "normal";
}
<div class="@css">아이템 수: @Model.Count</div>
```

---

### ✅ 태그 자체 조건 렌더링

```razor
@if (Model.IsAdmin)
{
    <button>관리자 설정</button>
}
```

```razor
@foreach (var item in Model.Items)
{
    <div class="@(item.IsSelected ? "selected" : "")">
        @item.Name
    </div>
}
```

---

### ✅ Custom 조건식 Helper 활용

```razor
@functions {
    public string BadgeClass(int value)
    {
        return value > 10 ? "badge-danger" : "badge-secondary";
    }
}

<span class="badge @BadgeClass(Model.Count)">
  @Model.Count
</span>
```

---

## 🧩 Razor Utility 팁

| 기술 | 예시 | 설명 |
|------|------|------|
| `@Html.Raw()` | `@Html.Raw(Model.Content)` | HTML 직접 출력 |
| `@string.Format()` | `@($"{Model.Name}님")` | C# 문자열 포매팅 |
| `@await` | `@await Component.InvokeAsync("NavBar")` | 비동기 컴포넌트 실행 |
| `@{}` 블록 | `@{ var now = DateTime.Now; }` | 코드 실행 블록 |
| `@* *@` | `@* Razor 주석 *@` | 주석 처리 |

---

## 🧠 고급 Razor 컴포넌트 설계 예시

```razor
<!-- _AlertBox.cshtml -->
@model string
<div class="alert alert-warning">
    ⚠️ @Model
</div>
```

사용:

```razor
<partial name="_AlertBox" model="로그인이 필요합니다." />
```

---

## 🔍 View에 Helpers, 컴포넌트, 조건부 렌더링을 조합한 예

```razor
@functions {
    public string RatingStar(int score) => new string('★', score);
}

@foreach (var product in Model.Products)
{
    <div class="product-card">
        <h4>@product.Name</h4>
        <p>가격: @product.Price 원</p>
        <p>평점: @RatingStar(product.Rating)</p>

        @if (product.IsNew)
        {
            <span class="badge bg-primary">NEW</span>
        }

        <partial name="_BuyButton" model="product" />
    </div>
}
```

---

## ✅ 요약

| 기능 | 설명 |
|------|------|
| `@functions` | View 내부 헬퍼 정의 |
| `Partial View` | UI 조각 분리 재사용 |
| `ViewComponent` | 로직 포함한 재사용 뷰 |
| 조건부 class | `class="@(조건 ? 값 : 값)"` |
| Custom 조건 Helper | `@HelperMethod(value)` |
| `Html.Raw` | HTML 문자열 렌더링 |