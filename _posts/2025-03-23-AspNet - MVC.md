---
layout: post
title: Avalonia - MVC
date: 2025-03-22 21:20:23 +0900
category: Avalonia
---
# 🧩 ASP.NET Core MVC의 Controller / Action / View 구조 완전 정복

---

## 📌 1. MVC란?

**MVC (Model-View-Controller)**는 웹 애플리케이션의 구조를 다음 세 부분으로 나눠 관리하는 디자인 패턴입니다:

| 구성요소 | 역할 |
|----------|------|
| **Model** | 데이터와 비즈니스 로직 담당 |
| **View** | 사용자에게 보여지는 UI 담당 |
| **Controller** | 사용자의 요청 처리, 모델/뷰 연결 |

---

## 🔗 기본 요청 흐름

```
브라우저 요청 (/Products/Details/3)
   ↓
[Routing]
   ↓
Controller: ProductsController
   ↓
Action: Details(int id)
   ↓
Model 가져오기 → View 반환
   ↓
View: Views/Products/Details.cshtml
```

---

## 🏗 2. 프로젝트 구조 예시

```
/Controllers
 └── ProductsController.cs

/Models
 └── Product.cs

/Views
 └── Products/
     ├── Index.cshtml
     ├── Details.cshtml
     ├── Create.cshtml
     └── Edit.cshtml
```

---

## 📁 3. Controller

### ✅ 기본 컨트롤러 예시

```csharp
using Microsoft.AspNetCore.Mvc;
using MyApp.Models;

public class ProductsController : Controller
{
    public IActionResult Index()
    {
        var products = ProductRepository.GetAll();
        return View(products);
    }

    public IActionResult Details(int id)
    {
        var product = ProductRepository.GetById(id);
        if (product == null) return NotFound();

        return View(product);
    }
}
```

### 🔹 특징

- `Controller`는 **클래스 이름 뒤에 `Controller`를 붙임**
- 각 public 메서드는 **Action** 역할
- `return View(...)`로 **View를 호출하고, 모델 전달**

---

## 🔸 4. Action

> 사용자의 요청에 대해 **로직 처리 + 뷰 반환** or **Redirect, JSON 반환 등**

### ✅ 다양한 반환 예시

```csharp
return View();                        // View 렌더링
return RedirectToAction("Index");     // 다른 액션으로 이동
return NotFound();                    // 404 반환
return Json(model);                   // JSON 반환
return Content("Hello World");        // 단순 문자열
```

### ✅ 라우팅 예시

```csharp
// GET /Products/Edit/3
public IActionResult Edit(int id)

// POST /Products/Edit/3
[HttpPost]
public IActionResult Edit(int id, Product product)
```

---

## 📄 5. View

> 실제 **HTML 페이지를 담당하는 Razor 템플릿**  
> 위치는 항상 `Views/컨트롤러명/액션명.cshtml`

### ✅ 예시: `Views/Products/Details.cshtml`

```html
@model MyApp.Models.Product

<h2>@Model.Name</h2>
<p>가격: @Model.Price.ToString("C")</p>
<p>@Model.Description</p>

<a asp-action="Index">← 목록으로</a>
```

### ✅ View 구성요소

| 요소 | 설명 |
|------|------|
| `@model` | 해당 View에서 사용될 데이터 타입 |
| `@Model` | 전달된 실제 데이터 인스턴스 |
| `asp-action` | 링크 클릭 시 이동할 Action 지정 |

---

## 🔧 6. Startup 또는 Program.cs에서 MVC 설정

```csharp
builder.Services.AddControllersWithViews();
```

```csharp
app.MapControllerRoute(
    name: "default",
    pattern: "{controller=Home}/{action=Index}/{id?}");
```

- 기본 경로: `/Controller/Action/id`

---

## ✅ URL → Controller 매핑 예시

| URL 요청 | 매핑 결과 |
|----------|------------|
| `/` | HomeController → Index() |
| `/Products` | ProductsController → Index() |
| `/Products/Details/5` | ProductsController → Details(5) |
| `/Admin/User/10` | AdminController → User(10) (커스텀 라우트 필요) |

---

## 🧪 7. 간단한 CRUD 예시 (요약)

### 📁 ProductsController.cs

```csharp
public IActionResult Create() => View();

[HttpPost]
public IActionResult Create(Product model)
{
    if (!ModelState.IsValid) return View(model);

    ProductRepository.Add(model);
    return RedirectToAction("Index");
}
```

### 📄 Create.cshtml

```html
@model Product

<form asp-action="Create" method="post">
    <label asp-for="Name"></label>
    <input asp-for="Name" />
    <button type="submit">저장</button>
</form>
```

---

## ✨ 8. ViewModel 사용 패턴

> View 전용 데이터를 만들고 싶을 때 ViewModel을 만들어 사용:

```csharp
public class ProductDetailsViewModel
{
    public Product Product { get; set; }
    public bool IsAdmin { get; set; }
}
```

컨트롤러에서:

```csharp
return View(new ProductDetailsViewModel {
    Product = product,
    IsAdmin = User.IsInRole("Admin")
});
```

---

## ✅ 요약

| 구성 | 역할 | 위치 |
|------|------|------|
| Controller | 요청 처리 및 View 연결 | `/Controllers` |
| Action | 요청별 메서드 | Controller 내부 |
| View | 사용자에게 보여질 Razor 템플릿 | `/Views/{Controller}/{Action}.cshtml` |

---

## 🔜 추천 다음 주제

- ✅ 라우팅 커스터마이징 (Attribute Routing)
- ✅ ViewComponent, Partial View
- ✅ Validation / ModelState
- ✅ Layout / Section 구조 이해
- ✅ Areas로 대규모 앱 구조화