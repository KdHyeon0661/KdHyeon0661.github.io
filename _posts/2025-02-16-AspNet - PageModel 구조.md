---
layout: post
title: AspNet - PageModel 구조
date: 2025-02-16 16:20:23 +0900
category: AspNet
---
# 🧠 Razor Pages의 PageModel 구조 완전 정복

ASP.NET Core Razor Pages는 **MVC의 Controller 역할**을 `PageModel`이라는 클래스로 대체합니다.  
각 Razor 페이지(`.cshtml`)와 짝을 이루는 `.cshtml.cs` 파일이 PageModel입니다.

---

## 📁 기본 파일 구조

```
Pages/
├── Index.cshtml         ← HTML + Razor
└── Index.cshtml.cs      ← PageModel 클래스 (백엔드 로직)
```

---

## 📄 1. PageModel 클래스 구조

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.RazorPages;

public class IndexModel : PageModel
{
    public string Message { get; set; }

    public void OnGet()
    {
        Message = "페이지 처음 진입 시 호출됨 (GET)";
    }

    public void OnPost()
    {
        Message = "폼 제출 시 호출됨 (POST)";
    }
}
```

| 메서드 | 요청 방식 | 설명 |
|--------|-----------|------|
| `OnGet()` | GET | 페이지를 처음 열 때 |
| `OnPost()` | POST | `<form method="post">`로 제출 시 |
| `OnGetAsync()` | GET 비동기 | 비동기 초기화 |
| `OnPostDelete()`, `OnPostSave()` 등 | POST + 핸들러 | `asp-page-handler` 속성으로 지정된 메서드 |

---

## 🔁 2. Razor Page와 PageModel 연결

**Index.cshtml**

```razor
@page
@model IndexModel
<h2>@Model.Message</h2>

<form method="post">
    <button type="submit">Submit</button>
</form>
```

- `@model` 지시어를 통해 PageModel과 연결
- `Model.Message`로 PageModel의 속성에 접근

---

## ✏️ 3. 폼 제출과 데이터 바인딩

### ✅ PageModel에 속성 선언 + `[BindProperty]`

```csharp
public class ContactModel : PageModel
{
    [BindProperty]
    public string Name { get; set; }

    public void OnPost()
    {
        // Name 속성에 자동으로 바인딩됨
    }
}
```

**cshtml**

```html
<form method="post">
    <input type="text" asp-for="Name" />
    <button type="submit">제출</button>
</form>
```

- `[BindProperty]`를 사용하면 Razor 폼 입력값이 자동으로 속성에 바인딩됨
- `OnPost()`에서 해당 속성을 통해 값 사용 가능

---

## 🧩 4. 여러 핸들러: `asp-page-handler` + `OnPostX()`

### ✅ PageModel

```csharp
public IActionResult OnPostDelete()
{
    // 삭제 처리
    return RedirectToPage("Index");
}
```

### ✅ Razor 뷰

```html
<form method="post" asp-page-handler="Delete">
    <button type="submit">삭제</button>
</form>
```

- `asp-page-handler="Delete"` → `OnPostDelete()` 메서드 실행
- `asp-page-handler="Save"` → `OnPostSave()` 메서드 실행

이 방식은 **한 페이지에서 여러 동작(저장, 삭제 등)**을 처리할 때 매우 유용함.

---

## 🔄 5. OnGet vs OnPost 정리

| 구분 | 설명 | 실행 시점 |
|------|------|------------|
| `OnGet()` | 페이지 로드 시 초기화 | `GET` 요청 |
| `OnPost()` | 폼 제출 시 처리 | `POST` 요청 |
| `OnPostX()` | 핸들러 지정된 버튼 클릭 시 | `POST` 요청 + 핸들러 지정 |
| `OnGetAsync()` | 비동기 초기화 | `GET` 요청 (비동기) |
| `OnPostAsync()` | 비동기 폼 처리 | `POST` 요청 (비동기) |

---

## 🔄 6. Redirect, TempData 등 활용

```csharp
public IActionResult OnPost()
{
    TempData["Result"] = "성공적으로 처리되었습니다.";
    return RedirectToPage("Result");
}
```

---

## ✅ 마무리 요약

| 구성 요소 | 설명 |
|------------|------|
| `.cshtml.cs` | Razor 페이지의 백엔드 역할 |
| `OnGet()` | 페이지 처음 요청 처리 |
| `OnPost()` | 폼 제출 처리 |
| `[BindProperty]` | 입력 값 자동 바인딩 |
| `asp-page-handler` | 메서드 분기 제어 |
| `RedirectToPage()` | 페이지 이동 제어 |
| `TempData` | 페이지 간 데이터 전달 (일회성) |