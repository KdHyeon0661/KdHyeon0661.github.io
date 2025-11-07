---
layout: post
title: AspNet - 폼 처리와 asp-for
date: 2025-02-24 19:20:23 +0900
category: AspNet
---
# ASP.NET Core Razor Pages의 폼 처리와 `asp-for` Tag Helper 완전 정복

## 0. 폼 처리 핵심 요약

1) Razor 페이지에 `<form>` 작성 → 2) 입력 필드에 `asp-for` 사용 → 3) PageModel의 `[BindProperty]`로 모델 수신 → 4) `OnPost()`에서 `ModelState.IsValid` 검사 → 5) 성공 시 처리/리다이렉트, 실패 시 다시 렌더링.

**핵심 원칙**
- `asp-for`는 **모델의 속성 이름**으로 `id`/`name`/`value` 등을 자동 생성한다(타입·표시 형식 포함).
- `[BindProperty]`는 POST(기본)에서 **자동 바인딩**한다. GET 질의 수신은 `SupportsGet = true`.
- 서버 검증이 **최종 권위**다. 클라이언트 검증은 편의 기능일 뿐이며 언제든 우회 가능.

---

## 1. Tag Helper와 `asp-for`의 역할

### 1.1 `asp-for`가 하는 일

```cshtml
<input asp-for="Name" class="form-control" />
```

다음처럼 렌더링된다(예시):

```html
<input type="text" id="Name" name="Name" value="홍길동" class="form-control" />
```

- `name`과 `id`는 모델 경로를 반영한다. 예) `User.Name` → `id="User_Name"`, `name="User.Name"`.
- 타입 추론: 숫자는 `type="number"`, 이메일은 `type="email"` 등으로 선택된다(일부는 `DataType`/주석에 따라 힌트).

### 1.2 Tag Helper 전체 지도(자주 쓰는 것)

| Tag Helper | 용도 | 예시 |
|---|---|---|
| `asp-for` | 모델 속성과 바인딩된 입력 필드/라벨 생성 | `<input asp-for="User.Email" />` |
| `asp-validation-for` | 해당 필드의 유효성 메시지 | `<span asp-validation-for="User.Email"></span>` |
| `asp-validation-summary` | 전체 또는 모델 수준 메시지 | `<div asp-validation-summary="All"></div>` |
| `asp-page` | Razor Page로 링크 생성 | `<a asp-page="/Account/Login">로그인</a>` |
| `asp-route-*` | 라우트/쿼리 값 추가 | `<a asp-page="/Search" asp-route-q="tv">검색</a>` |
| `asp-items` | `<select>` 항목 묶음 | `<select asp-for="CategoryId" asp-items="Model.Categories"></select>` |
| `asp-page-handler` | 폼을 특정 핸들러로 라우팅 | `<form method="post" asp-page-handler="Save">` |

> Tag Helper가 동작하려면 일반적으로 `Pages/_ViewImports.cshtml`에 `@addTagHelper *, Microsoft.AspNetCore.Mvc.TagHelpers`가 있어야 한다(템플릿 기본 포함).

---

## 2. 실전 예제 1 — 회원 가입 폼(확장판)

기존 예제에 **라벨/요약/반복 가능한 에러**, `DisplayName`, 클라이언트 검증 스크립트 포함, 간단한 PRG 패턴을 더했다.

### 2.1 Pages/Register.cshtml

```cshtml
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

    <div asp-validation-summary="ModelOnly"></div>

    <button type="submit">가입</button>
</form>

@section Scripts {
    <partial name="_ValidationScriptsPartial" />
}
```

### 2.2 Pages/Register.cshtml.cs

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
        if (!ModelState.IsValid) return Page();

        // TODO: 생성/저장 로직
        TempData["Notice"] = "가입이 완료되었습니다.";
        return RedirectToPage("Success");
    }
}

public class UserInputModel
{
    [Required, Display(Name = "이름")]
    [StringLength(50)]
    public string Name { get; set; } = "";

    [Required, EmailAddress, Display(Name = "이메일")]
    public string Email { get; set; } = "";

    [Range(0, 120), Display(Name = "나이")]
    public int Age { get; set; }
}
```

> `asp-validation-for`, `asp-validation-summary`, `_ValidationScriptsPartial`까지 포함하면 클라이언트 검증(편의)과 서버 검증(최종)이 조합되어 UX가 좋아진다.

---

## 3. `[BindProperty]` 정확히 이해하기

```csharp
[BindProperty]
public MyInput Input { get; set; } = new();
```

- 기본은 **POST만** 바인딩한다. 쿼리로 GET 파라미터를 바인딩하려면:

```csharp
[BindProperty(SupportsGet = true)]
public string? Keyword { get; set; }
```

**보안 주의(Overposting)**
- 모델에 서버 전용 속성(예: `IsAdmin`)을 노출하지 말 것.
- 필요 시 DTO/입력 전용 모델을 분리하고, 엔터티에는 서버에서 필요한 값만 매핑한다.
- 또 다른 방지: `[BindNever]`를 서버가 지정하는 속성에 사용하여 입력 바인딩 차단.

---

## 4. 유효성 검사 흐름과 메시지 표시

### 4.1 Data Annotations 요약

| 특성 | 설명 |
|---|---|
| `[Required]` | 필수값 |
| `[EmailAddress]` | 이메일 형식 |
| `[Range(min, max)]` | 범위 제한 |
| `[StringLength]` / `[MinLength]` / `[MaxLength]` | 길이 제한 |
| `[RegularExpression("…")]` | 정규식 |
| `[Compare("OtherProperty")]` | 일치 비교(예: 비밀번호 확인) |

### 4.2 메시지 출력 패턴

- 개별 필드: `<span asp-validation-for="User.Name"></span>`
- 요약: `<div asp-validation-summary="All"></div>` 또는 `ModelOnly`

**서버측 커스텀 에러 추가**
```csharp
ModelState.AddModelError("User.Email", "이 이메일은 사용할 수 없습니다.");
```

---

## 5. 실전 예제 2 — 중첩/컬렉션 바인딩(주소 목록)

**목표**: `User.Addresses[i]` 형태로 여러 주소 입력을 받고 유효성 검사까지 수행.

### 5.1 모델

```csharp
public class AddressInput
{
    [Required, StringLength(80)]
    public string Line1 { get; set; } = "";

    [StringLength(80)]
    public string? Line2 { get; set; }

    [Required, StringLength(40)]
    public string City { get; set; } = "";

    [Required, StringLength(10)]
    public string Zip { get; set; } = "";
}

public class ProfileInput
{
    [Required]
    public string FullName { get; set; } = "";

    [MinLength(1, ErrorMessage = "주소는 최소 1개 필요합니다.")]
    public List<AddressInput> Addresses { get; set; } = new();
}
```

### 5.2 PageModel

```csharp
public class ProfileModel : PageModel
{
    [BindProperty] public ProfileInput Input { get; set; } = new();

    public void OnGet()
    {
        if (Input.Addresses.Count == 0)
            Input.Addresses.Add(new AddressInput());
    }

    public IActionResult OnPost()
    {
        if (!ModelState.IsValid) return Page();

        // 저장 로직
        return RedirectToPage("Success");
    }
}
```

### 5.3 Razor

```cshtml
@page
@model ProfileModel

<h2>프로필</h2>

<form method="post">
  <div>
    <label asp-for="Input.FullName"></label>
    <input asp-for="Input.FullName" />
    <span asp-validation-for="Input.FullName"></span>
  </div>

  <fieldset>
    <legend>주소</legend>
    @for (int i = 0; i < Model.Input.Addresses.Count; i++)
    {
      <div>
        <input asp-for="Input.Addresses[i].Line1" placeholder="주소1" />
        <span asp-validation-for="Input.Addresses[i].Line1"></span>

        <input asp-for="Input.Addresses[i].Line2" placeholder="주소2" />
        <span asp-validation-for="Input.Addresses[i].Line2"></span>

        <input asp-for="Input.Addresses[i].City" placeholder="도시" />
        <span asp-validation-for="Input.Addresses[i].City"></span>

        <input asp-for="Input.Addresses[i].Zip" placeholder="우편번호" />
        <span asp-validation-for="Input.Addresses[i].Zip"></span>
      </div>
    }
  </fieldset>

  <div asp-validation-summary="ModelOnly"></div>

  <button type="submit">저장</button>
</form>

@section Scripts {
    <partial name="_ValidationScriptsPartial" />
}
```

**핵심**: 인덱싱된 `asp-for`를 사용하면 `Input.Addresses[0].City` 같은 `name`을 자동 생성하여 서버 바인딩이 정확히 맞물린다.

---

## 6. 실전 예제 3 — 셀렉트/라디오/체크박스와 `asp-items`

### 6.1 PageModel

```csharp
public class ProductCreateModel : PageModel
{
    [BindProperty] public ProductInput Input { get; set; } = new();

    public SelectList Categories { get; private set; } = default!;
    public List<SelectListItem> Tags { get; private set; } = default!;

    public void OnGet()
    {
        Categories = new SelectList(new[]
        {
            new { Id = 1, Name = "전자" }, new { Id = 2, Name = "도서" }
        }, "Id", "Name");

        Tags = new()
        {
            new SelectListItem("신상품", "new"),
            new SelectListItem("인기", "hot"),
            new SelectListItem("할인", "sale")
        };
    }

    public IActionResult OnPost()
    {
        // OnPost에서 Categories/Tags 재준비(재렌더링 필요)
        OnGet();

        if (!ModelState.IsValid) return Page();
        return RedirectToPage("Success");
    }
}

public class ProductInput
{
    [Required] public string Name { get; set; } = "";
    [Range(1, int.MaxValue)] public int CategoryId { get; set; }
    public List<string> SelectedTags { get; set; } = new();
}
```

### 6.2 Razor

```cshtml
@page
@model ProductCreateModel

<form method="post">
  <div>
    <label asp-for="Input.Name"></label>
    <input asp-for="Input.Name" />
    <span asp-validation-for="Input.Name"></span>
  </div>

  <div>
    <label asp-for="Input.CategoryId">카테고리</label>
    <select asp-for="Input.CategoryId" asp-items="Model.Categories"></select>
    <span asp-validation-for="Input.CategoryId"></span>
  </div>

  <fieldset>
    <legend>태그</legend>
    @for (int i = 0; i < Model.Tags.Count; i++)
    {
        var t = Model.Tags[i];
        <label>
          <input type="checkbox" name="Input.SelectedTags" value="@t.Value"
                 checked="@(Model.Input.SelectedTags.Contains(t.Value))" />
          @t.Text
        </label>
    }
  </fieldset>

  <button type="submit">등록</button>
</form>

@section Scripts {
  <partial name="_ValidationScriptsPartial" />
}
```

> `<select asp-for ... asp-items=...>`가 가장 간결하다. 체크박스/라디오 그룹은 `name`을 동일하게 맞춰 컬렉션/단일 선택을 모아준다.

---

## 7. 날짜/통화/숫자 포맷과 문화권(Culture)

- HTML5 `<input type="date">`는 브라우저가 로케일 포맷을 처리한다. 서버 바인딩은 현재 **Culture**에 영향받으므로 로케일이 다른 환경에서는 ISO 8601(예: `yyyy-MM-dd`) 사용을 권장.
- 서버에서 표준 포맷 적용:
  ```csharp
  [DisplayFormat(ApplyFormatInEditMode = true, DataFormatString = "{0:yyyy-MM-dd}")]
  public DateTime BirthDate { get; set; }
  ```
- 앱 전반의 문화권은 `UseRequestLocalization()`과 지원 문화권 설정으로 일관화하라(다국어/다지역 서비스 참조).

---

## 8. 파일 업로드와 `asp-for`

모델:

```csharp
public class UploadInput
{
    [Required] public IFormFile File { get; set; } = default!;
}
```

뷰:

```cshtml
<form method="post" enctype="multipart/form-data">
  <input asp-for="Input.File" type="file" />
  <span asp-validation-for="Input.File"></span>
  <button>업로드</button>
</form>
```

PageModel:

```csharp
public class UploadModel : PageModel
{
    [BindProperty] public UploadInput Input { get; set; } = new();

    public async Task<IActionResult> OnPostAsync()
    {
        if (!ModelState.IsValid) return Page();

        var f = Input.File;
        if (f.Length == 0) { ModelState.AddModelError("Input.File", "빈 파일입니다."); return Page(); }

        // 크기/ContentType/시그니처 검사 등 보안 필수
        using var s = f.OpenReadStream();
        // 저장 처리...
        return RedirectToPage("Success");
    }
}
```

> `enctype="multipart/form-data"`를 반드시 설정한다. ContentType은 신뢰하지 말고 **파일 시그니처 검사**를 권장.

---

## 9. 다중 핸들러와 버튼 라우팅

같은 페이지에서 저장/삭제 등 여러 동작을 분기하고 싶을 때:

```csharp
public IActionResult OnPostSave() { /* 저장 */ return RedirectToPage("List"); }
public IActionResult OnPostDelete() { /* 삭제 */ return RedirectToPage("List"); }
```

```cshtml
<form method="post" asp-page-handler="Save">
  <button type="submit">저장</button>
</form>

<form method="post" asp-page-handler="Delete">
  <button type="submit">삭제</button>
</form>
```

또는 하나의 폼에서 여러 버튼을 구분:

```cshtml
<form method="post">
  <button type="submit" name="handler" value="Save">저장</button>
  <button type="submit" name="handler" value="Delete">삭제</button>
</form>
```

PageModel:

```csharp
public IActionResult OnPost(string handler)
{
    return handler switch
    {
        "Save" => OnPostSave(),
        "Delete" => OnPostDelete(),
        _ => Page()
    };
}
```

---

## 10. ModelState 고급: 값 우선순위와 정리

- 폼 POST 후, 서버에서 `Model.Property` 값을 바꿔도 **ModelState의 원본 값**이 다시 렌더링될 수 있다.
- 이런 경우:
  ```csharp
  ModelState.Remove("Input.Name");
  Input.Name = Normalize(Input.Name);
  TryValidateModel(Input, nameof(Input));
  ```
- 완전 초기화가 필요하면 `ModelState.Clear()` 후 다시 `TryValidateModel`.

---

## 11. PRG(Post-Redirect-Get) 패턴

폼 재제출 방지, 새로고침 안전, URL 공유 안전:

```csharp
public IActionResult OnPost()
{
    if (!ModelState.IsValid) return Page();

    TempData["Notice"] = "처리 완료";
    return RedirectToPage("Result");
}
```

---

## 12. 보안과 접근성(Accessibility)

### 12.1 보안
- **서버 검증 필수**: 클라이언트 검증은 우회 가능.
- **Overposting 방지**: 입력 전용 DTO 사용, `[BindNever]`/화이트리스트 적용.
- **입력 한도**: 본문/파일 크기 제한, 컬렉션 항목 수 제한.
- **HTML 인코딩**: 사용자 입력 출력 시 자동 인코딩을 신뢰하되, 필요시 `HtmlEncoder`/화이트리스트 마크다운 파서 사용.

### 12.2 접근성
- `label asp-for`는 `for`/`id`를 정확히 연결해 스크린리더 호환성 향상.
- 오류 영역에 `role="alert"` 또는 `aria-live="assertive"`를 둘 수 있다.
- 색만으로 오류를 전달하지 말고 텍스트를 제공하라. `asp-validation-for` 출력은 텍스트 메시지로 전달된다.

---

## 13. 최소 실행 골격(Program.cs, .NET 8 기준)

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddRazorPages(); // + AddMvcOptions 등 확장 가능

var app = builder.Build();

app.UseHttpsRedirection();
app.UseStaticFiles();
app.UseRouting();
app.MapRazorPages();

app.Run();
```

---

## 14. 테스트 팁

- **단위 테스트**: DataAnnotations 유효성은 `Validator.TryValidateObject`로 순수 검증 가능.
- **통합 테스트**: `WebApplicationFactory<T>`로 폼 POST → 리다이렉트/오류 렌더링 검증.
- **바인딩 이름 규칙**: 컬렉션/중첩 키(`Input.Addresses[0].Zip`)가 정확히 생성되었는지 확인.

---

## 15. 자주 겪는 문제와 해결

| 문제 | 원인 | 해결 |
|---|---|---|
| 값 수정했는데 화면에 반영 안 됨 | ModelState 원본 값 우선 | `ModelState.Remove()` 후 재검증 |
| 셀렉트 박스 항목 사라짐 | OnPost에서 리스트 재채움 누락 | OnPost 실패 시에도 데이터 소스 재준비 |
| 클라이언트 검증 미동작 | `_ValidationScriptsPartial` 누락 | 섹션 또는 레이아웃에 포함 |
| 컬렉션 바인딩 실패 | 잘못된 name 키 | `asp-for="Input.Items[i].Prop"` 패턴 사용 |
| GET 바인딩 안 됨 | SupportsGet 미설정 | `[BindProperty(SupportsGet = true)]` |

---

## 16. 요약

- `asp-for`는 모델에 정합된 `id`/`name`/`value`/`type`을 자동 생성해 **타입 안전한 폼**을 만든다.
- `[BindProperty]`로 데이터 수신, `ModelState.IsValid`로 서버 검증, 실패 시 **재렌더링**, 성공 시 **PRG**.
- 실전에서는 **중첩/컬렉션**, **셀렉트/체크박스/라디오**, **파일 업로드**, **문화권 포맷**을 정확히 처리해야 한다.
- **보안(Overposting 방지, 한도 설정, 인코딩)**과 **접근성(label/오류 텍스트)**을 기본 규약으로 삼아라.
- 문제의 90%는 **이름 규칙**(경로), **모델 상태 정리**, **데이터 소스 재준비**, **스크립트 포함**에서 발생한다.
