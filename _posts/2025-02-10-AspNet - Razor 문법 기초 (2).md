---
layout: post
title: AspNet - Razor 문법 기초 (2)
date: 2025-02-10 20:20:23 +0900
category: AspNet
---
# 📝 Razor 폼 처리와 Tag Helper, 유효성 검사 완전 정리

Razor Pages에서는 사용자 입력을 받기 위해 **폼 처리**, **모델 바인딩**, **유효성 검사**를 지원합니다. 이와 함께 **Tag Helper**를 사용하면 뷰 코드가 훨씬 간결하고 유지보수가 쉬워집니다.

---

## 📌 1. Razor에서 Form 기본 구조

```html
<form method="post">
    <label>이름</label>
    <input type="text" name="Name" />
    <button type="submit">제출</button>
</form>
```

- `method="post"`를 지정하면 PageModel의 `OnPost()` 메서드가 호출됩니다.
- `input`의 `name` 속성이 PageModel의 속성과 매칭되어 자동 바인딩됩니다.

---

## 🧩 2. 모델 클래스(Model) 정의

```csharp
public class ContactForm
{
    [Required]
    public string Name { get; set; }

    [EmailAddress]
    public string Email { get; set; }
}
```

- `DataAnnotations` 특성을 사용하면 유효성 검사를 자동화할 수 있습니다.

---

## 📄 3. PageModel (백엔드 로직)

```csharp
public class ContactModel : PageModel
{
    [BindProperty]
    public ContactForm Form { get; set; }

    public string Result { get; set; }

    public void OnGet() { }

    public IActionResult OnPost()
    {
        if (!ModelState.IsValid)
        {
            return Page(); // 유효성 검사 실패 → 폼 다시 표시
        }

        Result = $"이름: {Form.Name}, 이메일: {Form.Email}";
        return Page(); // 또는 RedirectToPage("Success")
    }
}
```

- `[BindProperty]`는 Razor Page에서 Form 데이터를 자동으로 바인딩하는 핵심
- `ModelState.IsValid`는 유효성 검사 통과 여부를 확인

---

## 💡 4. Tag Helper 사용 예시

Tag Helper는 HTML 태그에 `asp-` 접두사를 붙여 뷰와 모델을 연결해주는 Razor 기능입니다.

```html
<form method="post">
    <div>
        <label asp-for="Form.Name"></label>
        <input asp-for="Form.Name" class="form-control" />
        <span asp-validation-for="Form.Name" class="text-danger"></span>
    </div>

    <div>
        <label asp-for="Form.Email"></label>
        <input asp-for="Form.Email" class="form-control" />
        <span asp-validation-for="Form.Email" class="text-danger"></span>
    </div>

    <button type="submit">제출</button>
</form>
```

| Tag Helper | 역할 |
|------------|------|
| `asp-for` | 모델 속성 바인딩 |
| `asp-validation-for` | 유효성 검사 메시지 출력 |
| `asp-page-handler` | 핸들러 메서드 지정 (예: `OnPostSubmit`) |

---

## ✅ 5. 유효성 검사 활성화

### ✅ 클라이언트 측 유효성 검사 활성화

폼 하단에 다음 코드를 삽입합니다 (Layout이나 `_ValidationScriptsPartial.cshtml` 포함 가능):

```html
@section Scripts {
    <partial name="_ValidationScriptsPartial" />
}
```

이 부분이 jQuery Validate와 unobtrusive validation을 활성화해줍니다.

---

## 🔍 6. Handler 메서드 응용 (`asp-page-handler`)

```csharp
public IActionResult OnPostSave() { ... }
public IActionResult OnPostDelete() { ... }
```

```html
<form method="post" asp-page-handler="Save">
    <button type="submit">저장</button>
</form>

<form method="post" asp-page-handler="Delete">
    <button type="submit">삭제</button>
</form>
```

- `asp-page-handler="Save"`는 `OnPostSave()`를 호출
- 여러 개의 버튼을 하나의 페이지에서 처리할 수 있어 유용함

---

## 📑 7. 전체 흐름 요약

| 단계 | 설명 |
|------|------|
| ① 모델 클래스 정의 | 유효성 검사 포함 |
| ② PageModel에 `[BindProperty]` 선언 | 자동 바인딩 |
| ③ Razor 페이지에 `asp-for`, `asp-validation-for` 사용 | UI 자동 바인딩 |
| ④ `ModelState.IsValid` 확인 | 서버 측 검증 |
| ⑤ `partial _ValidationScriptsPartial` 포함 | 클라이언트 검증 지원 |

---

## 🧪 예제 결과

입력 값이 비어 있거나 잘못된 경우:

```html
<span class="text-danger">Name 필드는 필수입니다.</span>
```

입력 값이 올바르면 `OnPost()` 메서드에서 처리됨

---

# 📝 마무리

Razor Pages에서는 `@model`, `asp-for`, `asp-validation-for` 같은 기능과  
`BindProperty`, `ModelState.IsValid`의 조합으로 **폼 처리와 유효성 검사**를 쉽게 구현할 수 있습니다.