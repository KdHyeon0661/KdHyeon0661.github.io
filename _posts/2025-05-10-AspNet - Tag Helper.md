---
layout: post
title: AspNet - Tag Helper
date: 2025-05-10 19:20:23 +0900
category: AspNet
---
# 자주 쓰는 Tag Helper 완전 정리 (ASP.NET Core)

## 준비: Tag Helper가 “안 먹힐” 때 체크리스트

`_ViewImports.cshtml`(또는 Razor Pages의 `_ViewImports.cshtml`, MVC의 `/Views/_ViewImports.cshtml`)에 다음 등록이 있어야 한다.

```razor
@addTagHelper *, Microsoft.AspNetCore.Mvc.TagHelpers   <!-- 기본 Tag Helpers -->
@using Microsoft.AspNetCore.Mvc.TagHelpers             <!-- (선택) 네임스페이스 노출 -->
```

- 특정 커스텀 Tag Helper 어셈블리 사용 시:
  `@addTagHelper *, YourAppAssemblyName`
- 반대로 특정 Tag Helper를 제외하고 싶다면:
  `@removeTagHelper *, Microsoft.AspNetCore.Mvc.TagHelpers`

---

## `asp-for` 계열 — **모델 바인딩 + 클라이언트 검증**의 핵심

### 기본 예제

```razor
@model RegisterViewModel

<form asp-action="Register" method="post">
    <label asp-for="UserName" class="form-label"></label>
    <input asp-for="UserName" class="form-control" />
    <span asp-validation-for="UserName" class="text-danger"></span>

    <label asp-for="Email" class="form-label"></label>
    <input asp-for="Email" class="form-control" />
    <span asp-validation-for="Email" class="text-danger"></span>

    <button type="submit" class="btn btn-primary mt-3">가입</button>
</form>

<partial name="_ValidationScriptsPartial" />
```

- `asp-for`는 `id/name/value`를 **모델명 기반으로 일관**되게 생성.
- `asp-validation-for`는 **DataAnnotations**(예: `[Required]`, `[EmailAddress]`)와 연동.
- `_ValidationScriptsPartial`에는 `jquery.validate` + `jquery.validate.unobtrusive` 로딩이 포함되어야 **클라이언트 검증**이 동작.

### 복합 속성/컬렉션의 이름 생성

```razor
@for (int i = 0; i < Model.Addresses.Count; i++)
{
    <input asp-for="Addresses[i].ZipCode" class="form-control" />
    <span asp-validation-for="Addresses[i].ZipCode"></span>
}
```

- 바인딩 이름이 자동으로 `Addresses[0].ZipCode` 형태로 생성되어 서버에서 컬렉션으로 안전하게 매핑된다.

---

## 링크/폼 경로: `asp-action`, `asp-controller`, `asp-page`, `asp-page-handler`

### MVC 라우팅(Controller/Action)

```razor
<a asp-controller="Home" asp-action="About">About</a>

<form asp-controller="Account" asp-action="Login" method="post">
  ...
</form>
```

### Razor Pages 라우팅(Page/Handler)

```razor
<a asp-page="/Contact">Contact</a>

<form asp-page="/Account/Login" asp-page-handler="TwoFactor" method="post">
  <!-- OnPostTwoFactor(...) 호출 -->
</form>
```

### `asp-area`

```razor
<a asp-area="Admin" asp-controller="Dashboard" asp-action="Index">관리자</a>
```

- 영역(Area)을 쓰면 URL 생성 시 자동으로 Area 토큰이 포함된다.

---

## 라우트 데이터: `asp-route` 및 `asp-route-{param}` — **우선순위/충돌 규칙**

### 쿼리스트링/경로 파라미터 전달

```razor
<!-- /Product/Detail/5 또는 /Product/Detail?id=5 (라우팅 규칙에 따름) -->
<a asp-controller="Product" asp-action="Detail" asp-route-id="5">자세히</a>

<!-- 쿼리 여러 개 -->
<a asp-action="Search" asp-route-q="phone" asp-route-page="2">검색</a>
```

### `asp-all-route-data`(딕셔너리 한 번에 바인딩)

```razor
@{
    var routes = new Dictionary<string, string?>
    {
        ["q"] = "tablet",
        ["page"] = "3",
        ["sort"] = "price"
    };
}
<a asp-action="Search" asp-all-route-data="routes">검색(딕셔너리)</a>
```

### 우선순위

- **강한 바인딩 > 약한 바인딩**
  1) `asp-route-{name}`
  2) `asp-all-route-data`
  3) 기존 라우트 데이터
- **명시적 값**이 있으면 라우트 템플릿 디폴트를 덮는다.

### 추가 속성: `asp-protocol`, `asp-host`, `asp-fragment`

```razor
<a asp-action="Detail" asp-controller="Blog"
   asp-route-id="10" asp-protocol="https" asp-host="my.example.com" asp-fragment="comments">
   글보기(외부 호스트 + 앵커)
</a>
```

---

## `asp-items` — `<select>` 목록 바인딩

```razor
<select asp-for="CategoryId" asp-items="Model.Categories" class="form-select"></select>
```

```csharp
// Controller/PageModel
Model.Categories = new List<SelectListItem> {
    new("Phone", "1"),
    new("Tablet", "2"),
    new SelectListItem("Laptop", "3") { Selected = true }
};
```

- `SelectListItem`의 `Selected`가 true이면 해당 항목 선택.
- `asp-for`의 현재 값과 `Value`가 일치하면 자동 선택.

---

## 유효성 검증 Tag Helpers — `asp-validation-for`, `asp-validation-summary`

```razor
<span asp-validation-for="Email" class="text-danger"></span>
<div asp-validation-summary="All" class="alert alert-danger"></div>
```

- `asp-validation-summary="ModelOnly" | "All" | "None"`
- **서버측 검증**은 항상 동작. **클라이언트 검증**은 스크립트 로딩 필요.

---

## 폼 + CSRF: `asp-antiforgery` & Method Spoofing

```razor
<form asp-action="Delete" asp-route-id="@Model.Id" method="post" asp-antiforgery="true">
  <button type="submit" class="btn btn-danger">삭제</button>
</form>
```

- 기본값이 `true`이므로 생략 가능.
- **HTTP PUT/DELETE**가 필요하면 **메서드 스푸핑**(폼 안에 숨겨진 `X-HTTP-Method-Override` 필드나 라우팅으로 처리)을 사용하거나 Ajax에 맞게 API 설계.

---

## 파일 업로드: `<input type="file" asp-for="...">`

```razor
<form asp-action="Upload" method="post" enctype="multipart/form-data">
  <input type="file" asp-for="ProfileImage" class="form-control" />
  <button type="submit" class="btn btn-primary mt-2">업로드</button>
</form>
```

```csharp
public class UploadModel
{
    [Required] public IFormFile ProfileImage { get; set; } = default!;
}
```

- `IFormFile`/`List<IFormFile>`로 바인딩.
- **보안 팁**: 확장자/MIME/크기/경로 검증 필수. 업로드 폴더 화이트리스트.

---

## 정적 리소스: `asp-append-version`(이미지/스크립트/스타일 **캐시 무효화**)

> 정적 파일이 바뀌면 **쿼리 파라미터에 파일 해시**를 붙여 브라우저 캐시를 무효화.

### Image Tag Helper

```razor
<img src="~/images/logo.png" asp-append-version="true" alt="로고" />
```

- 결과: `/images/logo.png?v={해시}`
- `wwwroot` 기준 경로를 `~`로 시작.

### Script/Link Tag Helper

```razor
<link rel="stylesheet" href="~/css/site.css" asp-append-version="true" />
<script src="~/js/site.js" asp-append-version="true"></script>
```

---

## 환경 분기: `<environment>` — 빌드/배포 모드에 맞추기

```razor
<environment include="Development">
  <script src="~/lib/vue.js"></script>
</environment>
<environment exclude="Development">
  <script src="~/lib/vue.min.js"></script>
</environment>
```

- `include`/`exclude`에 쉼표로 여러 값 지정 가능: `"Development,Staging"`

---

## 부분 캐싱: `<cache>` Tag Helper — **UI 단위 성능 개선**

```razor
<cache expires-after="00:01:00"
       vary-by-user="true"
       vary-by-route="@ViewContext.RouteData.Values["id"]">
    <partial name="_ExpensiveFragment" />
</cache>
```

### 주요 속성 요약

| 속성 | 의미 |
|---|---|
| `expires-after="hh:mm:ss"` | 상대 만료 |
| `expires-on="yyyy-MM-ddTHH:mm:ssZ"` | 절대 만료 |
| `expires-sliding="hh:mm:ss"` | 미접근 시점 기준 연장 |
| `vary-by` | 임의 문자열 키 |
| `vary-by-user="true|false"` | 사용자별 분기 |
| `vary-by-cookie` / `vary-by-header` / `vary-by-query` / `vary-by-route` | 해당 요소 기준 분기 |
| `priority="Low|Normal|High|NeverRemove"` | 제거 우선순위 |
| `enabled="true|false"` | 캐시 사용 스위치 |

> 분산 캐시(예: Redis)를 쓰고 싶으면 **Response/Output Caching** 또는 **별도 캐시 전략**을 고려.

---

## Partial & ViewComponent & VC Tag Helper

### Partial

```razor
<partial name="_LoginPartial" />
```

### ViewComponent 호출(두 가지)

```razor
@await Component.InvokeAsync("CartSummary", new { userId = Model.UserId })
```

**또는 VC Tag Helper**

```razor
<vc:cart-summary user-id="@(Model.UserId)"></vc:cart-summary>
```

- VC 클래스가 `CartSummaryViewComponent`라면 태그는 `<vc:cart-summary>`.

---

## 앵커 고급: **라우팅 + 프래그먼트 + 프로토콜/호스트** 종합

```razor
<a asp-controller="Docs" asp-action="Read"
   asp-route-slug="intro"
   asp-protocol="https" asp-host="docs.example.com"
   asp-fragment="section-2">문서</a>
```

- 외부 호스트/프로토콜과 결합한 **절대 URL** 생성.
- 동일 앱 내 라우트 토큰과 안전하게 결합.

---

## & SEO에 유용한 태그 상호작용

- `<label asp-for="...">`는 해당 `input`의 `id`와 자동 연결 → 스크린리더 친화.
- `Validation` Tag Helpers는 오류 요약을 마크업으로 출력 → 보조공학 접근성 향상.
- `<environment>`로 **개발/운영 스크립트** 구분 → 불필요한 Dev 스크립트 유출 방지.

---

## **실전 폼 예제** — 파일 업로드 + 검증 + 리다이렉트 보존

```razor
@model RegisterViewModel
@{
    var returnUrl = Context.Request.Query["returnUrl"].ToString();
}

<form asp-action="Register" asp-route-returnUrl="@returnUrl" method="post" enctype="multipart/form-data">
    <div class="mb-2">
        <label asp-for="UserName" class="form-label"></label>
        <input asp-for="UserName" class="form-control" />
        <span asp-validation-for="UserName" class="text-danger"></span>
    </div>

    <div class="mb-2">
        <label asp-for="Email" class="form-label"></label>
        <input asp-for="Email" class="form-control" />
        <span asp-validation-for="Email" class="text-danger"></span>
    </div>

    <div class="mb-2">
        <label asp-for="Avatar" class="form-label"></label>
        <input type="file" asp-for="Avatar" class="form-control" />
        <span asp-validation-for="Avatar" class="text-danger"></span>
    </div>

    <button class="btn btn-primary mt-2">가입</button>
</form>

<partial name="_ValidationScriptsPartial" />
```

- `asp-route-returnUrl`로 로그인/가입 후 **원래 페이지 복귀** UX.

---

## **보안 모범 사례**

- **Anti-forgery 토큰**(기본 on) 유지, Ajax 시 헤더에 토큰 전달.
- **업로드 파일 검증**: 확장자·MIME·크기·저장 경로 화이트리스트.
- `Html.Raw`는 **꼭 필요할 때만**, 사용자 입력과 결합 금지.
- `asp-append-version`으로 정적 파일 **정확한 버전 캐싱** 유지.

---

## **트러블슈팅**

### Tag Helper가 동작하지 않음

- `_ViewImports.cshtml`에 `@addTagHelper`가 누락되었는지 확인.
- 뷰가 **Razor Class Library**에 있을 경우 해당 어셈블리 기준으로 추가.
- 커스텀 Tag Helper라면 **네임스페이스/어셈블리명 정확히**.

### 라우트 값 충돌

- `asp-route-*` vs 기존 라우트 값 충돌 시 **명시적 값이 우선**.
- `asp-all-route-data`와 병행 시 **개별 `asp-route-*`가 우선**.

### 클라이언트 검증이 안 됨

- `_ValidationScriptsPartial` 미포함/순서 불량.
- `jquery`, `jquery.validate`, `jquery.validate.unobtrusive` 로딩 순서 확인.
- `name/id` 불일치(직접 작성) → **반드시 `asp-for` 사용**.

---

## **확장: 커스텀 Tag Helper 미니 패턴**

### 조건 클래스 머지

```csharp
[HtmlTargetElement(Attributes = "class-when, class-name")]
public class WhenClassTagHelper : TagHelper
{
    [HtmlAttributeName("class-when")] public bool Condition { get; set; }
    [HtmlAttributeName("class-name")] public string ClassName { get; set; } = "";

    public override void Process(TagHelperContext ctx, TagHelperOutput output)
    {
        if (!Condition) return;
        var cls = output.Attributes["class"]?.Value?.ToString();
        output.Attributes.SetAttribute("class",
            string.IsNullOrWhiteSpace(cls) ? ClassName : $"{cls} {ClassName}");
    }
}
```

```razor
<button class="btn" class-when="@(Model.IsDanger)" class-name="btn-danger">삭제</button>
```

### 출력 억제(권한/플래그)

```csharp
[HtmlTargetElement("if-claims")]
public class IfClaimsTagHelper : TagHelper
{
    public string? Claim { get; set; }
    public string? Value { get; set; }
    public override void Process(TagHelperContext c, TagHelperOutput o)
    {
        var ok = /* HttpContext.User.HasClaim(Claim, Value) */ false;
        if (!ok) o.SuppressOutput();
    }
}
```

```razor
<if-claims claim="Role" value="Admin">
  <a asp-area="Admin" asp-controller="Users" asp-action="Index">사용자 관리</a>
</if-claims>
```

---

## **요약 테이블** — 실무에서 가장 자주 쓰는 Tag Helpers

| Tag Helper | 핵심 역할 | 포인트 |
|---|---|---|
| `asp-for` | `id/name/value` 자동 생성 | DataAnnotations와 검증 연동 |
| `asp-validation-for`/`-summary` | 필드/전체 오류 표시 | `_ValidationScriptsPartial` 필수 |
| `asp-action/controller` | MVC 라우트 | `asp-area`와 조합 |
| `asp-page/page-handler` | Razor Pages 라우트 | 핸들러: `OnPostXxx` |
| `asp-route(-*)` | 쿼리/경로 파라미터 | 우선순위: route-* > all-route-data |
| `asp-all-route-data` | 딕셔너리 일괄 바인딩 | 대량 파라미터 편리 |
| `asp-protocol/host/fragment` | 절대 URL/앵커 | 외부 호스트 링크 생성 |
| `asp-items` | `<select>` 바인딩 | `List<SelectListItem>` |
| `<environment>` | 환경별 스크립트/스타일 | include/exclude |
| `asp-append-version` | 정적 파일 캐시 무효화 | 파일 해시 `?v=` 파라미터 |
| `<cache>` | 부분 UI 캐싱 | vary-* 로 분기 |
| `<partial>` / `<vc:* />` | UI 조각/VC 렌더링 | VC는 DI/테스트 우수 |

---

## 실전 스니펫: “검색 폼 + 결과 페이징” 한 방에

```razor
@model ProductSearchVM

<form asp-action="Search" method="get" class="row g-2 mb-3">
  <div class="col-auto">
    <input asp-for="Q" class="form-control" placeholder="검색어" />
  </div>
  <div class="col-auto">
    <select asp-for="CategoryId" asp-items="Model.Categories" class="form-select">
      <option value="">전체</option>
    </select>
  </div>
  <div class="col-auto">
    <button class="btn btn-primary">검색</button>
  </div>
</form>

<ul class="list-group">
@foreach (var p in Model.Items)
{
  <li class="list-group-item d-flex justify-content-between">
    <span>@p.Name</span>
    <a asp-action="Detail" asp-route-id="@p.Id" class="btn btn-sm btn-outline-secondary">보기</a>
  </li>
}
</ul>

<nav class="mt-3">
  <ul class="pagination">
    @for (var i = 1; i <= Model.TotalPages; i++)
    {
      <li class="page-item @(i==Model.Page ? "active" : "")">
        <a class="page-link"
           asp-action="Search"
           asp-route-q="@Model.Q"
           asp-route-categoryId="@Model.CategoryId"
           asp-route-page="@i">@i</a>
      </li>
    }
  </ul>
</nav>
```

- **포인트**: 검색 조건을 **모두 `asp-route-*`로 보존**하여 페이지 이동 시 상태 유지.

---

## 마무리

- “HTML처럼 보이지만 C#과 라우팅/검증/보안을 **컴파일 안전**하게” 연결하는 것이 Tag Helper의 본질이다.
- **폼/검증/라우팅/정적 리소스/캐시**를 Tag Helper로 일관 구성하면, 유지보수성과 접근성이 동시 개선된다.
- 컴포넌트화가 필요해지면 **Partial → ViewComponent → Custom Tag Helper** 순으로 승격을 검토하라.
