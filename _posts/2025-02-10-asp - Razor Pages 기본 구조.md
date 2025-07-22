---
layout: post
title: Asp - Razor Pages 기본 구조
date: 2025-01-27 19:20:23 +0900
category: Asp
---
# 📘 Razor Pages 기본 구조 완벽 분석

**Razor Pages**는 ASP.NET Core에서 제공하는 **페이지 기반 웹앱 개발 방식**입니다. 전통적인 MVC 패턴보다 **간단하고 직관적**하여 CRUD 기반 웹앱에 적합합니다.

---

## 🧱 1. Razor Pages의 기본 구조

`dotnet new webapp` 명령으로 생성한 기본 프로젝트는 다음과 같은 구조를 가집니다:

```
MyApp/
├── Pages/
│   ├── Index.cshtml
│   ├── Index.cshtml.cs
│   ├── Privacy.cshtml
│   ├── Shared/
│   │   ├── _Layout.cshtml
│   │   └── _ValidationScriptsPartial.cshtml
│   └── _ViewImports.cshtml
│
├── wwwroot/
├── Program.cs
├── appsettings.json
└── MyApp.csproj
```

---

## 📄 2. Razor Page 파일 구성 (예: `Index.cshtml`, `Index.cshtml.cs`)

Razor Page는 **두 개의 파일**로 구성됩니다:

| 파일 | 역할 |
|------|------|
| `Index.cshtml` | UI(뷰)를 구성하는 Razor 템플릿 |
| `Index.cshtml.cs` | C# 로직을 담은 PageModel 클래스 (백엔드) |

### ✅ `Index.cshtml` 예시

```html
@page
@model IndexModel
@{
    ViewData["Title"] = "홈 페이지";
}
<h1>@ViewData["Title"]</h1>
<p>Razor Pages 앱에 오신 것을 환영합니다!</p>
```

- `@page`: 이 파일이 라우팅 가능한 Razor Page임을 나타냄
- `@model IndexModel`: 해당 페이지가 사용할 PageModel 클래스 지정

### ✅ `Index.cshtml.cs` 예시

```csharp
public class IndexModel : PageModel
{
    public string Message { get; set; }

    public void OnGet()
    {
        Message = "페이지가 로드되었습니다.";
    }
}
```

- `PageModel`을 상속받아 Razor Page의 로직을 구현
- `OnGet()`, `OnPost()` 등 HTTP 메서드에 따라 실행됨

---

## 🚦 3. Razor Pages 라우팅

- 기본적으로 `Pages/Index.cshtml` → `/` (홈)
- `Pages/Products/List.cshtml` → `/Products/List`
- `@page "{id}"` → URL 매개변수 처리 가능 (`/Product/42`)

```csharp
@page "{id:int}"
public class ProductModel : PageModel
{
    public int Id { get; set; }

    public void OnGet(int id)
    {
        Id = id;
    }
}
```

---

## ⚙️ 4. 주요 파일 설명

| 파일/폴더 | 설명 |
|-----------|------|
| `Pages/` | 모든 Razor Page가 위치 |
| `Pages/Shared/` | 공통 레이아웃 및 파셜 뷰 |
| `_Layout.cshtml` | 전체 페이지의 기본 레이아웃 (헤더/푸터 포함) |
| `_ViewImports.cshtml` | 모든 Razor Page에 적용할 공통 using/import 설정 |
| `Program.cs` | 애플리케이션 시작 및 설정 |
| `wwwroot/` | CSS, JS, 이미지 등 정적 파일 보관 위치 |

---

## 🔁 5. Razor Page의 수명 주기 메서드

| 메서드 | 설명 |
|--------|------|
| `OnGet()` | GET 요청 처리 |
| `OnPost()` | POST 요청 처리 |
| `OnGetAsync()` | 비동기 GET 처리 |
| `OnPostAsync()` | 비동기 POST 처리 |
| `OnGet[핸들러]()` | 다중 핸들러 지원 |
| `OnPost[핸들러]()` | POST 다중 핸들러 |

```csharp
public IActionResult OnPostSubmit()
{
    // OnPostSubmit 핸들러 처리
    return RedirectToPage("Success");
}
```

```html
<form method="post" asp-page-handler="Submit">
    <button type="submit">제출</button>
</form>
```

---

## 📥 6. 폼 처리 및 데이터 바인딩

```csharp
public class ContactModel : PageModel
{
    [BindProperty]
    public ContactForm Form { get; set; }

    public IActionResult OnPost()
    {
        if (!ModelState.IsValid)
            return Page();

        // Form 처리 로직
        return RedirectToPage("Thanks");
    }
}

public class ContactForm
{
    [Required]
    public string Name { get; set; }

    [EmailAddress]
    public string Email { get; set; }
}
```

```html
<form method="post">
    <input asp-for="Form.Name" />
    <span asp-validation-for="Form.Name"></span>

    <input asp-for="Form.Email" />
    <span asp-validation-for="Form.Email"></span>

    <button type="submit">제출</button>
</form>
```

---

## ✅ 7. Razor Pages의 장점

- ✅ 간단한 CRUD 작업에 최적화
- ✅ UI 파일과 로직 파일이 한 쌍으로 유지되므로 가독성 ↑
- ✅ ASP.NET MVC보다 파일 구조가 단순
- ✅ 기본 제공되는 보안 기능 (Model Validation, Anti-forgery)

---

## ❓ 언제 Razor Pages를 사용해야 할까?

| 선택 기준 | Razor Pages | MVC |
|-----------|--------------|-----|
| 간단한 CRUD 웹앱 | 👍 적합 | 가능 |
| 대규모 도메인 분리 앱 | ❌ 다소 불편 | 👍 적합 |
| 웹 API와 혼합 | 가능 | 👍 더 자연스러움 |
| 초보자 친화도 | 👍 매우 높음 | 중간 |

---

# 🔚 마무리

Razor Pages는 ASP.NET Core에서 생산성과 구조적 유연성을 동시에 제공하는 강력한 웹 개발 방식입니다. 단순한 페이지 중심의 앱에는 MVC보다 훨씬 효율적입니다.

다음 글에서는 `폼 처리 및 유효성 검사`, `Razor 문법`, 또는 `Partial View와 Layout 사용법`을 이어서 다뤄보겠습니다.