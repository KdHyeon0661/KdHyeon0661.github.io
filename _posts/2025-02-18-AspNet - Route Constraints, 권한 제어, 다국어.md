---
layout: post
title: AspNet -Route Constraints, 권한 제어, 다국어
date: 2025-02-18 19:20:23 +0900
category: AspNet
---
# 🧭 ASP.NET Core 고급 라우팅: Route Constraints, 권한 제어, 다국어(Localization)

ASP.NET Core에서는 단순한 URL 매핑을 넘어서, 라우팅을 세밀하게 제어하고 보안을 강화하며  
국제화를 지원하는 다양한 기능을 제공합니다.

이번 글에서는 세 가지 고급 라우팅 기능을 다룹니다:

1. 🔒 Route Constraints (라우트 제약 조건)  
2. 🔐 라우트 기반 권한 제어  
3. 🌐 라우트 기반 Localization (다국어 URL 지원)

---

## 1️⃣ Route Constraints (라우트 제약 조건)

**라우트 제약 조건**은 URL 파라미터에 허용되는 형식이나 값을 제한하는 기능입니다.

### ✅ 기본 사용법

```razor
@page "{id:int}"
```

→ `id`는 정수(`int`)일 경우에만 라우트에 매핑됩니다.  
`/Page/abc` 요청은 404 Not Found.

---

### ✅ 사용 가능한 제약 조건

| 제약 조건 | 설명 | 예시 |
|-----------|------|------|
| `int` | 정수 | `{id:int}` |
| `bool` | 불리언 | `{flag:bool}` |
| `datetime` | 날짜/시간 | `{dt:datetime}` |
| `decimal` | 소수 | `{price:decimal}` |
| `guid` | GUID 형식 | `{id:guid}` |
| `alpha` | 알파벳만 | `{name:alpha}` |
| `minlength(n)` | 최소 길이 | `{title:minlength(3)}` |
| `maxlength(n)` | 최대 길이 | `{title:maxlength(10)}` |
| `range(min,max)` | 숫자 범위 제한 | `{age:range(1,120)}` |
| `length(n)` | 정확한 길이 | `{code:length(6)}` |

---

### ✅ 복수 조건 지정

```razor
@page "{id:int:min(1)}"
```

→ 1 이상의 정수만 허용

---

### ✅ MVC에서도 동일 사용 가능

```csharp
[Route("product/{id:int}")]
public IActionResult Details(int id) => View();
```

---

## 2️⃣ 라우트 기반 권한 제어

라우트는 보안 정책과도 연결할 수 있습니다.  
`Authorize` 속성을 이용해 특정 URL이나 핸들러에 인증/권한을 적용합니다.

---

### ✅ Razor Page 전체 보호

**Product/Edit.cshtml.cs**

```csharp
[Authorize]
public class EditModel : PageModel
{
    public void OnGet() { }
}
```

→ 로그인하지 않은 사용자는 `/Product/Edit` 접근 불가

---

### ✅ 역할 기반 제어

```csharp
[Authorize(Roles = "Admin")]
public class AdminModel : PageModel
{
    public void OnGet() { }
}
```

→ `Admin` 역할만 해당 페이지 접근 가능

---

### ✅ 특정 핸들러만 보호

```csharp
public class OrderModel : PageModel
{
    public void OnGet() { }

    [Authorize]
    public IActionResult OnPostPay() { ... }
}
```

→ `GET /Order`는 모두 접근 가능하지만, `POST /Order?handler=Pay`는 로그인 필요

---

### ✅ 정책 기반 제어

Startup.cs 또는 Program.cs에서 정책 정의 후 적용

```csharp
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("Over18", policy =>
        policy.RequireClaim("Age", "18", "19", "20"));
});
```

```csharp
[Authorize(Policy = "Over18")]
public class AdultOnlyPage : PageModel { }
```

---

## 3️⃣ 라우트 기반 Localization (다국어 URL)

ASP.NET Core는 URL에 언어 코드를 포함시켜 **문화권/언어에 따라 콘텐츠를 전환**할 수 있습니다.

---

### ✅ URL 패턴 예

```
/en/home
/ko/home
/fr/products/details/5
```

→ 언어 코드에 따라 다국어 콘텐츠 표시

---

### ✅ Startup.cs 또는 Program.cs 설정

```csharp
var supportedCultures = new[] { "en", "ko", "fr" };

builder.Services.Configure<RequestLocalizationOptions>(options =>
{
    options.SetDefaultCulture("en");
    options.AddSupportedCultures(supportedCultures);
    options.AddSupportedUICultures(supportedCultures);
    options.RequestCultureProviders.Insert(0, new RouteDataRequestCultureProvider());
});
```

→ URL에서 `/{culture}` 경로를 우선적으로 해석

---

### ✅ 라우트 템플릿에 문화권 추가

**MVC**

```csharp
app.MapControllerRoute(
    name: "default",
    pattern: "{culture=en}/{controller=Home}/{action=Index}/{id?}");
```

**Razor Pages**

```csharp
app.MapRazorPages().AddRazorPagesOptions(options =>
{
    options.Conventions.AddFolderRouteModelConvention("/", model =>
    {
        foreach (var selector in model.Selectors)
        {
            selector.AttributeRouteModel = new AttributeRouteModel()
            {
                Template = "{culture=en}/" + selector.AttributeRouteModel?.Template
            };
        }
    });
});
```

---

### ✅ Razor 뷰에서 문화 코드 기반 링크 생성

```razor
<a asp-page="/Index" asp-route-culture="ko">한국어</a>
<a asp-page="/Index" asp-route-culture="en">English</a>
```

---

### ✅ 자원 파일 사용 (`.resx`)

- `Resources/Pages/Index.ko.resx`
- `Resources/Pages/Index.en.resx`

```razor
@inject IStringLocalizer<IndexModel> L

<h1>@L["WelcomeMessage"]</h1>
```

---

## ✅ 마무리 요약

| 기능 | 설명 | 예시 |
|------|------|------|
| Route Constraints | 라우트 파라미터 형식 제한 | `{id:int}`, `{slug:guid}` |
| 라우트 기반 권한 제어 | 특정 URL에 인증/인가 설정 | `[Authorize(Roles = "Admin")]` |
| 라우트 기반 Localization | URL로 언어 감지 및 전환 | `/ko/index`, `/en/index` |

---

## 🔜 다음 추천 주제

- ✅ `IStringLocalizer` 다국어 리소스 활용법
- ✅ `Custom Middleware`로 라우트 기반 동작 확장
- ✅ API 라우팅과 버저닝 (`/api/v1/products`)

---

위 세 가지 주제는 **실제 서비스 품질 및 글로벌 지원에 매우 중요한 포인트**야.  
다음으로 `resx` 기반 다국어 처리나 `RouteDataRequestCultureProvider` 커스터마이징을 원한다면 이어서 도와줄게!