---
layout: post
title: AspNet - Razor 문법 기초 (2)
date: 2025-02-10 20:20:23 +0900
category: AspNet
---
# Razor 폼 처리와 Tag Helper, 유효성 검사 완전 정리

## 무엇을 만들 것인가 — 시나리오

- `ContactForm` 입력 페이지
- 필수/형식 검증, 서버/클라이언트 메시지 출력
- 다중 핸들러(`Save`, `Delete`), PRG(Post/Redirect/Get) 패턴
- 파일 업로드, 라디오/체크/셀렉트, 컬렉션 바인딩
- 사용자 지정 검증과 원격 검증, ModelState 수동 오류 주입
- 과바인딩 방지, CSRF/안티포저리, 크기 제한과 에러 처리

---

## 폼의 최소 골격과 바인딩 흐름

### 기본 HTML 폼과 OnPost

```cshtml
@page
@model ContactModel

<form method="post">
    <label>이름</label>
    <input type="text" name="Name" />
    <button type="submit">제출</button>
</form>
```

```csharp
public class ContactModel : PageModel
{
    [BindProperty]              // 폼 필드를 PageModel 속성으로 자동 바인딩
    public string? Name { get; set; }

    public void OnGet() { }

    public IActionResult OnPost()
    {
        if (!ModelState.IsValid) return Page();
        // 처리 로직
        return RedirectToPage("Success");
    }
}
```

- `method="post"` → 같은 페이지의 `OnPost()` 실행
- 입력의 `name` 속성과 **동일한 이름의 속성**에 바인딩
- 1편에서 배운 `@{ }`/`@Model`로 값 출력 가능

---

## 강한 형식 바인딩과 DataAnnotations 검증

### 모델 클래스 정의

```csharp
using System.ComponentModel.DataAnnotations;

public class ContactForm
{
    [Required(ErrorMessage = "이름은 필수입니다.")]
    [StringLength(50, MinimumLength = 2)]
    public string Name { get; set; } = "";

    [EmailAddress(ErrorMessage = "이메일 형식이 올바르지 않습니다.")]
    public string? Email { get; set; }

    [Phone]
    public string? Phone { get; set; }

    [Range(0, 150)]
    public int? Age { get; set; }

    [DataType(DataType.Date)]
    [DisplayFormat(ApplyFormatInEditMode = true, DataFormatString = "{0:yyyy-MM-dd}")]
    public DateTime? BirthDate { get; set; }
}
```

### PageModel과 바인딩

```csharp
public class ContactModel : PageModel
{
    [BindProperty]             // 폼의 모든 하위 필드를 ContactForm에 바인딩
    public ContactForm Form { get; set; } = new();

    public string? Result { get; set; }

    public void OnGet() { }

    public IActionResult OnPost()
    {
        if (!ModelState.IsValid)
        {
            // 검증 실패 → 폼 재표시, 오류 메시지는 뷰에서 Tag Helper로 노출
            return Page();
        }

        Result = $"이름: {Form.Name}, 이메일: {Form.Email}";
        TempData["flash"] = "저장되었습니다.";
        return RedirectToPage("Success");
    }
}
```

---

## Tag Helper로 강한 형식 폼 만들기

- `_ViewImports.cshtml`에 다음이 필요합니다.

```cshtml
@addTagHelper *, Microsoft.AspNetCore.Mvc.TagHelpers
```

### `asp-for`와 검증 메시지 출력

```cshtml
@page
@model ContactModel
@{
    ViewData["Title"] = "연락처";
}

<h1>@ViewData["Title"]</h1>

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

    <div>
        <label asp-for="Form.BirthDate"></label>
        <input asp-for="Form.BirthDate" class="form-control" />
        <span asp-validation-for="Form.BirthDate" class="text-danger"></span>
    </div>

    <button type="submit">제출</button>
</form>

@section Scripts {
    <partial name="_ValidationScriptsPartial" />
}
```

- `asp-for`는 **네임/아이디/값/유효성 어트리뷰트**를 자동 생성
- `asp-validation-for`는 해당 필드의 **검증 메시지**를 표시
- `_ValidationScriptsPartial`은 jQuery Validate + unobtrusive 스크립트를 포함

### 요약 메시지(`asp-validation-summary`)

```cshtml
<div asp-validation-summary="ModelOnly" class="text-danger"></div>
```

옵션
- `None`, `ModelOnly`, `All`

---

## 다중 핸들러(버튼별로 다른 OnPost)

```csharp
public IActionResult OnPostSave()
{
    if (!ModelState.IsValid) return Page();
    // 저장 로직
    TempData["flash"] = "저장 완료";
    return RedirectToPage("Index");
}

public IActionResult OnPostDelete()
{
    // 삭제 로직
    TempData["flash"] = "삭제 완료";
    return RedirectToPage("Index");
}
```

```cshtml
<form method="post" asp-page-handler="Save">
    <button type="submit">저장</button>
</form>

<form method="post" asp-page-handler="Delete">
    <button type="submit">삭제</button>
</form>
```

- `asp-page-handler="Save"` → `OnPostSave()` 실행
- **PRG 패턴** 권장: POST 이후 Redirect로 새로고침 중복 방지

---

## 라우팅/전송 대상 제어: `asp-page`/`asp-route-*`/`asp-antiforgery`

```cshtml
<form method="post" asp-page="/Contacts/Edit" asp-route-id="@Model.FormId">
    <button type="submit">업데이트</button>
</form>
```

- `asp-page`로 **다른 페이지**에 POST 가능
- `asp-route-키`로 **라우트 값** 제공
- Razor Pages의 폼엔 자동으로 **안티포저리 토큰**이 삽입(기본값). API나 특별 케이스에선 다음처럼 제어:

```cshtml
<form method="post" asp-antiforgery="false">
```

> 보안상 권장하지 않음. API는 일반적으로 JSON POST + CSRF 전략 다르게 설계.

---

## 선택/체크/라디오/셀렉트 박스

### 라디오/체크

```csharp
public class Prefs
{
    public bool Agree { get; set; }
    public string ContactMethod { get; set; } = "Email"; // Email, Phone
}
```

```cshtml
<label>
    <input asp-for="Prefs.Agree" /> 약관 동의
</label>

<div>
    <input type="radio" asp-for="Prefs.ContactMethod" value="Email" /> 이메일
    <input type="radio" asp-for="Prefs.ContactMethod" value="Phone" /> 전화
</div>
<span asp-validation-for="Prefs.ContactMethod" class="text-danger"></span>
```

### 드롭다운(`asp-items`)

```csharp
public IEnumerable<SelectListItem> Countries => new[]
{
    new SelectListItem("Korea", "KR"),
    new SelectListItem("USA", "US"),
    new SelectListItem("Japan", "JP")
};

[BindProperty]
public string CountryCode { get; set; } = "KR";
```

```cshtml
<select asp-for="CountryCode" asp-items="Model.Countries" class="form-select"></select>
<span asp-validation-for="CountryCode" class="text-danger"></span>
```

---

## 파일 업로드(IFormFile, 다중 파일, 크기 제한)

### 모델/페이지 모델

```csharp
public class UploadForm
{
    [Required]
    public IFormFile? Avatar { get; set; }

    public List<IFormFile> Attachments { get; set; } = new();
}

public class UploadModel : PageModel
{
    [BindProperty]
    public UploadForm Form { get; set; } = new();

    public async Task<IActionResult> OnPostAsync()
    {
        if (!ModelState.IsValid) return Page();

        if (Form.Avatar is { Length: > 0 })
        {
            var path = Path.Combine("wwwroot/uploads", Path.GetFileName(Form.Avatar.FileName));
            await using var fs = System.IO.File.Create(path);
            await Form.Avatar.CopyToAsync(fs);
        }

        foreach (var file in Form.Attachments.Where(f => f.Length > 0))
        {
            var path = Path.Combine("wwwroot/uploads", Path.GetFileName(file.FileName));
            await using var fs = System.IO.File.Create(path);
            await file.CopyToAsync(fs);
        }

        TempData["flash"] = "업로드 완료";
        return RedirectToPage();
    }
}
```

### 뷰

```cshtml
<form method="post" enctype="multipart/form-data">
    <div>
        <label asp-for="Form.Avatar"></label>
        <input asp-for="Form.Avatar" type="file" />
        <span asp-validation-for="Form.Avatar" class="text-danger"></span>
    </div>

    <div>
        <label>첨부 파일</label>
        <input asp-for="Form.Attachments" type="file" multiple />
    </div>

    <button type="submit">업로드</button>
</form>
```

### 제한/보안 체크리스트

- `RequestSizeLimit`, `FormOptions.MultipartBodyLengthLimit`으로 **최대 크기 제한**
- 파일명 검증/허용 확장자 화이트리스트/AV 스캔/랜덤 파일명/별도 스토리지
- Public 폴더 업로드 시 **XSS/실행 위험** 고려(정적 호스팅 정책/헤더/권한)

---

## 컬렉션/중첩 모델 바인딩

### 중첩 모델

```csharp
public class Address
{
    [Required] public string Line1 { get; set; } = "";
    public string? Line2 { get; set; }
    [Required] public string City { get; set; } = "";
}

public class CustomerForm
{
    [Required] public string Name { get; set; } = "";
    public List<Address> Addresses { get; set; } = new();
}
```

### 뷰(인덱스 기반 이름)

```cshtml
@for (int i = 0; i < Model.Form.Addresses.Count; i++)
{
    <div>
        <input asp-for="Form.Addresses[i].Line1" />
        <span asp-validation-for="Form.Addresses[i].Line1"></span>
    </div>
    <div>
        <input asp-for="Form.Addresses[i].City" />
        <span asp-validation-for="Form.Addresses[i].City"></span>
    </div>
}
```

- **동적 추가/삭제** 시 인덱스 일관성 유지
- 부분 뷰/템플릿으로 분리하면 유지보수 용이

---

## 서버 측 검증 고급: 수동 오류, IValidatableObject, 커스텀 Attribute

### ModelState 수동 오류 추가

```csharp
if (Form.Age is < 14)
{
    ModelState.AddModelError(nameof(Form.Age), "만 14세 이상만 가입할 수 있습니다.");
    return Page();
}
```

### 객체 단위 검증(IValidatableObject)

```csharp
public class RegisterForm : IValidatableObject
{
    [Required] public string Password { get; set; } = "";
    [Required] public string Confirm { get; set; } = "";

    public IEnumerable<ValidationResult> Validate(ValidationContext context)
    {
        if (Password != Confirm)
            yield return new ValidationResult("비밀번호가 일치하지 않습니다.", new[] { nameof(Confirm) });
    }
}
```

### 커스텀 검증 특성

```csharp
public class NotDisposableEmailAttribute : ValidationAttribute
{
    public override bool IsValid(object? value)
    {
        if (value is not string email) return true;
        return !email.EndsWith("@examplemail.xyz", StringComparison.OrdinalIgnoreCase);
    }
}
```

```csharp
public class SignUpForm
{
    [Required, EmailAddress, NotDisposableEmail(ErrorMessage = "일회용 메일은 허용되지 않습니다.")]
    public string Email { get; set; } = "";
}
```

---

## 원격 검증(Remote)과 클라이언트 검증

### Remote(컨트롤러 또는 페이지 핸들러에서 제공)

컨트롤러 예시:

```csharp
[AcceptVerbs("Get", "Post")]
public IActionResult CheckName(string name)
{
    var exists = _db.Users.Any(u => u.Name == name);
    return exists ? Json($"이미 사용 중인 이름입니다.") : Json(true);
}
```

모델:

```csharp
using Microsoft.AspNetCore.Mvc;

public class NameForm
{
    [Required]
    [Remote(action: "CheckName", controller: "Validation")]
    public string Name { get; set; } = "";
}
```

주의
- Razor Pages에서도 `page: "/Validation", handler: "CheckName"` 패턴 사용 가능
- 클라이언트에서 jQuery Validate가 Ajax 호출

---

## TryUpdateModelAsync와 과바인딩 방지

### 과바인딩 문제

- 폼에서 **수정하면 안 되는 필드**(예: Role, IsAdmin)가 무단으로 바인딩될 수 있음

### 화이트리스트 바인딩

```csharp
public async Task<IActionResult> OnPostAsync(int id)
{
    var entity = await _db.Users.FindAsync(id);
    if (entity is null) return NotFound();

    // 허용 필드만 바인딩
    var ok = await TryUpdateModelAsync(entity, prefix: "User",
        u => u.DisplayName, u => u.Phone, u => u.Address);

    if (!ok) return Page();
    await _db.SaveChangesAsync();
    return RedirectToPage("Index");
}
```

### [Bind]/[BindNever]/[ValidateNever]

```csharp
public class UserEditDto
{
    [BindRequired] public string DisplayName { get; set; } = "";
    [ValidateNever] public string? ServerOnlyNote { get; set; }
}

public class AdminOnly
{
    [BindNever] public bool IsAdmin { get; set; }   // 바인딩 금지
}
```

- DTO를 별도로 두어 **입력 모델과 도메인 모델을 분리**하는 것이 가장 안전

---

## 날짜/숫자/문화권 이슈

- `DateTime`, `decimal` 파싱은 **현재 `CultureInfo`**에 의존
- 서버/클라이언트 문화권이 다르면 형식 불일치 → **일관된 포맷**과 `DisplayFormat`/`DataType` 사용
- 서버 진입 시 `CultureInfo.CurrentCulture`를 로깅하고 이슈 추적

---

## 유효성 스크립트, Unobtrusive, 부분 렌더링 주의

- `_ValidationScriptsPartial`에는 jQuery와 unobtrusive가 포함
- 부분 뷰/동적 DOM 삽입 시 **re-parse** 필요(ajax 폼 새로고침 시)
- 클라이언트 검증은 **보조 수단**일 뿐, **서버 검증**은 필수

---

## 안전/보안 체크리스트

1) **CSRF**: Razor Pages 폼은 기본 **안티포저리 토큰** 자동 삽입. API는 **JWT/Origin/CSRF 전략** 별도
2) **XSS**: Razor는 기본 인코딩. `Html.Raw`는 신뢰 콘텐츠만
3) **과바인딩**: DTO/화이트리스트 바인딩/`[BindNever]`
4) **파일 업로드**: 허용 확장자/크기 제한/랜덤 이름/외부 저장/스캔
5) **PRG 패턴**: 중복 제출 방지, 플래시 메시지는 `TempData` 활용
6) **로깅/감사**: 실패한 검증 사유, 클라이언트 정보, Culture 로깅

---

## 검증 UI 컴포넌트 패턴

### 필드 컴포넌트화(부분 뷰)

`Pages/Shared/_FormField.cshtml`:
```cshtml
@model (string Label, string For, string? Placeholder)

<div class="mb-3">
    <label asp-for="@Model.For">@Model.Label</label>
    <input asp-for="@Model.For" class="form-control" placeholder="@Model.Placeholder" />
    <span asp-validation-for="@Model.For" class="text-danger"></span>
</div>
```

사용:
```cshtml
<partial name="_FormField" model='("이름", "Form.Name", "이름을 입력")' />
```

> 팀 표준 UI를 부분 뷰/Tag Helper로 캡슐화하면 **일관성**과 **생산성**이 크게 향상됩니다.

---

## 에러/예외 처리와 사용자 경험

### 유효성 실패 시

```csharp
if (!ModelState.IsValid)
{
    // 필요 시 커스텀 메시지 추가
    ModelState.AddModelError(string.Empty, "입력을 다시 확인하세요.");
    return Page();
}
```

```cshtml
<div asp-validation-summary="All" class="text-danger"></div>
```

### 서버 예외

- `try/catch`로 예외 포착 → `ModelState.AddModelError`로 사용자 친화적 메시지
- 시스템 로그에는 실제 예외 스택 기록(PII 노출 금지)

---

## End-to-End 실전 조각

### 모델

```csharp
public class TicketForm : IValidatableObject
{
    [Required, StringLength(100)]
    public string Title { get; set; } = "";

    [Required, StringLength(1000)]
    public string Content { get; set; } = "";

    [Required]
    public string Category { get; set; } = "General";

    public bool NotifyByEmail { get; set; }
    public IFormFile? Attachment { get; set; }

    public IEnumerable<ValidationResult> Validate(ValidationContext context)
    {
        if (Category == "Billing" && !NotifyByEmail)
            yield return new ValidationResult("요금 관련 문의는 이메일 알림 동의가 필요합니다.", new[] { nameof(NotifyByEmail) });
    }
}
```

### PageModel

```csharp
public class CreateTicketModel : PageModel
{
    [BindProperty]
    public TicketForm Form { get; set; } = new();

    public IEnumerable<SelectListItem> Categories => new[]
    {
        new SelectListItem("일반", "General"),
        new SelectListItem("기술", "Tech"),
        new SelectListItem("요금", "Billing")
    };

    public void OnGet() { }

    public async Task<IActionResult> OnPostAsync()
    {
        if (!ModelState.IsValid) return Page();

        if (Form.Attachment is { Length: > 0 })
        {
            var path = Path.Combine("wwwroot/attachments", Path.GetFileName(Form.Attachment.FileName));
            await using var fs = System.IO.File.Create(path);
            await Form.Attachment.CopyToAsync(fs);
        }

        TempData["flash"] = "티켓이 등록되었습니다.";
        return RedirectToPage("Index");
    }
}
```

### 뷰

```cshtml
@page
@model CreateTicketModel
@{
    ViewData["Title"] = "티켓 생성";
}

<h1>@ViewData["Title"]</h1>

<div asp-validation-summary="ModelOnly" class="text-danger"></div>

<form method="post" enctype="multipart/form-data">
    <div class="mb-3">
        <label asp-for="Form.Title"></label>
        <input asp-for="Form.Title" class="form-control" />
        <span asp-validation-for="Form.Title" class="text-danger"></span>
    </div>

    <div class="mb-3">
        <label asp-for="Form.Content"></label>
        <textarea asp-for="Form.Content" class="form-control"></textarea>
        <span asp-validation-for="Form.Content" class="text-danger"></span>
    </div>

    <div class="mb-3">
        <label asp-for="Form.Category"></label>
        <select asp-for="Form.Category" asp-items="Model.Categories" class="form-select"></select>
        <span asp-validation-for="Form.Category" class="text-danger"></span>
    </div>

    <div class="form-check mb-3">
        <input asp-for="Form.NotifyByEmail" class="form-check-input" />
        <label asp-for="Form.NotifyByEmail" class="form-check-label"></label>
    </div>

    <div class="mb-3">
        <label asp-for="Form.Attachment"></label>
        <input asp-for="Form.Attachment" type="file" class="form-control" />
        <span asp-validation-for="Form.Attachment" class="text-danger"></span>
    </div>

    <button type="submit" class="btn btn-primary">등록</button>
</form>

@section Scripts {
    <partial name="_ValidationScriptsPartial" />
}
```

---

## MVC와의 차이(간단 비교)

- Razor Pages: `PageModel` 내부에 `OnPost*` 핸들러 메서드
- MVC: `Controller`의 `Action` 메서드, 파라미터/모델로 바인딩

컨트롤러 예:

```csharp
[HttpPost]
public IActionResult Create([FromForm] TicketForm form)
{
    if (!ModelState.IsValid) return View(form);
    // ...
    return RedirectToAction("Index");
}
```

> API 컨트롤러는 `FromBody`/JSON 입력, Razor Pages/MVC 폼은 `FromForm`가 기본.

---

## 성능/UX 팁

- 읽기 전용 조회는 `AsNoTracking()`
- 긴 폼은 **탭/스텝 폼**으로 분할, 섹션별 유효성 확인
- 클라이언트 검증 실패 시 **첫 오류로 스크롤 이동**
- 서버 왕복 최소화가 중요하면 Ajax/Partial 업데이트(검증 메시지 영역만 갱신)

---

## 체크리스트(요약)

1) `_ViewImports.cshtml`에 Tag Helper 활성
2) `_ValidationScriptsPartial` 포함으로 클라이언트 검증
3) DTO 분리/화이트리스트(`TryUpdateModelAsync`)로 과바인딩 방지
4) PRG 패턴 + `TempData`로 중복 제출 방지
5) 파일 업로드 보안(확장자/크기/경로/권한)
6) 날짜/숫자 문화권 일관화
7) 필요 시 `IValidatableObject`/커스텀 Attribute/Remote 검증

---

## 부록 A. 모델 상태 디버깅 팁

```csharp
foreach (var kv in ModelState)
{
    var key = kv.Key;
    var errors = kv.Value.Errors.Select(e => e.ErrorMessage);
    // 로그로 남겨 원인 파악
}
```

---

## 부록 B. 요청 크기 제한과 메시지

```csharp
[RequestSizeLimit(10_000_000)] // 10MB
public class UploadModel : PageModel { ... }
```

미들웨어 수준:

```csharp
app.Use(async (ctx, next) =>
{
    try { await next(); }
    catch (BadHttpRequestException ex) when (ex.Message.Contains("Request body too large"))
    {
        ctx.Response.StatusCode = StatusCodes.Status413PayloadTooLarge;
        await ctx.Response.WriteAsync("파일이 너무 큽니다.");
    }
});
```

---

## 부록 C. 단위 테스트 아이디어

- `PageModel`을 **서비스/리포지토리**와 분리, **DI**로 주입
- 검증 실패 시 `ModelState` 내용과 반환 결과가 `PageResult`인지 확인
- 성공 시 리다이렉트 대상 검증, 파일 저장 경로/명 규칙 테스트

---

# 결론

- Razor Pages의 폼 처리는 **Tag Helper + `[BindProperty]` + DataAnnotations + ModelState** 조합만 알아도 **안전하고 생산적**입니다.
- 실전에서는 **PRG**, **과바인딩 방지**, **파일 업로드 보안**, **문화권 포맷**, **원격/커스텀 검증**을 더해 **단단한 사용자 입력 파이프라인**을 완성하십시오.
