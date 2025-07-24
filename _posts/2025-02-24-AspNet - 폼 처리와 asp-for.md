---
layout: post
title: AspNet - 폼 처리와 asp-for
date: 2025-02-24 19:20:23 +0900
category: AspNet
---
# 📝 ASP.NET Core Razor Pages의 폼 처리와 `asp-for` Tag Helper 완전 정복

웹 개발에서 사용자 입력을 받는 **폼(Form)**은 핵심 기능입니다.  
ASP.NET Core Razor Pages는 `Tag Helpers`와 모델 바인딩 시스템 덕분에  
타입 안정성과 HTML 생성의 편의성을 동시에 제공합니다.

이 글에서는 다음 내용을 다룹니다:

- `asp-for`와 Tag Helper의 역할
- 폼 데이터 바인딩과 `[BindProperty]`
- 입력 검증 (Validation) 흐름
- 실전 예제까지!

---

## 📌 1. 폼 데이터 처리 개요

ASP.NET Core Razor Pages에서는 보통 아래와 같은 흐름으로 폼 데이터를 처리합니다.

1. Razor 페이지에 `<form>` 구성
2. 입력 필드에 `asp-for` 사용
3. PageModel에서 `[BindProperty]`로 데이터 수신
4. `OnPost()`에서 검증 및 처리

---

## 🧩 2. `asp-for`와 Tag Helper란?

### ✅ `asp-for`

`asp-for`는 **Tag Helper**의 일종으로, **모델 속성과 자동으로 연결**되는 입력 필드를 생성합니다.

```html
<input asp-for="Name" class="form-control" />
```

자동으로 아래와 같은 HTML로 변환됩니다:

```html
<input type="text" id="Name" name="Name" class="form-control" value="홍길동" />
```

→ `Name` 속성의 값과 자동으로 바인딩됨.

---

### ✅ 주요 Tag Helpers

| 태그 | 설명 |
|------|------|
| `asp-for` | 모델 속성과 바인딩된 필드 |
| `asp-validation-for` | 특정 필드의 유효성 오류 메시지 |
| `asp-validation-summary` | 모든 유효성 메시지 요약 출력 |
| `asp-page` | 페이지 라우팅 링크 생성 |
| `asp-route-*` | URL 파라미터 바인딩 |

---

## 🧪 3. 실전 예제: 회원 가입 폼

### 📄 Pages/Register.cshtml

```razor
@page
@model RegisterModel

<h2>회원 가입</h2>

<form method="post">
    <div>
        <label asp-for="User.Name"></label>
        <input asp-for="User.Name" />
        <span asp-validation-for="User.Name"></span>
    </div>
    <div>
        <label asp-for="User.Email"></label>
        <input asp-for="User.Email" />
        <span asp-validation-for="User.Email"></span>
    </div>
    <div>
        <label asp-for="User.Age"></label>
        <input asp-for="User.Age" />
        <span asp-validation-for="User.Age"></span>
    </div>
    <button type="submit">가입</button>
</form>

<partial name="_ValidationScriptsPartial" />
```

---

### 📄 Pages/Register.cshtml.cs

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.RazorPages;
using System.ComponentModel.DataAnnotations;

public class RegisterModel : PageModel
{
    [BindProperty]
    public UserInputModel User { get; set; } = new();

    public void OnGet() { }

    public IActionResult OnPost()
    {
        if (!ModelState.IsValid)
        {
            return Page(); // 유효성 오류 시 다시 폼 표시
        }

        // TODO: 저장 처리
        return RedirectToPage("Success");
    }
}

public class UserInputModel
{
    [Required]
    [Display(Name = "이름")]
    public string Name { get; set; }

    [Required, EmailAddress]
    [Display(Name = "이메일")]
    public string Email { get; set; }

    [Range(0, 120)]
    [Display(Name = "나이")]
    public int Age { get; set; }
}
```

---

## 🧷 4. `[BindProperty]` 사용

`[BindProperty]`는 Razor Page가 HTTP 요청 값을 해당 속성에 자동 바인딩하도록 도와줍니다.

- `GET`, `POST` 둘 다 가능 (`SupportsGet = true` 지정 시)
- 보통 `POST` 처리에 사용

```csharp
[BindProperty]
public string Name { get; set; }
```

---

## 📌 5. 유효성 검사 흐름

ASP.NET Core는 **Data Annotation 기반 유효성 검사**를 지원합니다.

| 특성 | 설명 |
|------|------|
| `[Required]` | 필수 입력 |
| `[EmailAddress]` | 이메일 형식 |
| `[Range(min, max)]` | 범위 제한 |
| `[StringLength(max)]` | 길이 제한 |
| `[RegularExpression("...")]` | 정규식 검사 |

→ Razor Tag Helper가 자동으로 오류 메시지를 출력

---

## 🧪 유효성 검사 메시지 출력

- `asp-validation-for="User.Name"`: 해당 필드 오류 출력
- `asp-validation-summary="All"`: 모든 오류 출력

```html
<span asp-validation-for="User.Name" class="text-danger"></span>
```

---

## ✅ 마무리 요약

| 요소 | 설명 |
|------|------|
| `asp-for` | 모델 속성과 자동 연결된 입력 필드 |
| `[BindProperty]` | 폼 데이터 수신용 속성 바인딩 |
| `OnPost()` | 폼 제출 후 처리 메서드 |
| `ModelState.IsValid` | 유효성 검사 결과 확인 |
| `asp-validation-for` | 개별 오류 메시지 표시 |
| `_ValidationScriptsPartial` | JS 기반 클라이언트 검사 포함 |
