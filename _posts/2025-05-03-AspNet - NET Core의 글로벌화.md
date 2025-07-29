---
layout: post
title: AspNet - NET Core의 글로벌화
date: 2025-05-03 21:20:23 +0900
category: AspNet
---
# 🌍 ASP.NET Core의 글로벌화 (Localization) 처리

---

## ✅ 1. 글로벌화란?

**Localization**은 애플리케이션을 **다국어/다문화에 맞게 조정**하는 작업입니다.  
ASP.NET Core는 강력한 `IStringLocalizer`와 `Resource (.resx)` 시스템을 내장해  
UI, 메시지, 날짜/숫자 등을 **문화권에 맞게 동적으로 전환**할 수 있습니다.

---

## 🧠 2. 국제화 관련 용어

| 용어 | 설명 |
|------|------|
| **Globalization** | 다국어 지원 가능한 구조 설계 |
| **Localization** | 특정 언어/문화에 맞게 번역 및 형식 조정 |
| **Culture** | `"en-US"`, `"ko-KR"`, `"fr-FR"` 등 |
| **UICulture** | UI 텍스트의 언어 (ex: 리소스 문자열) |
| **Culture** | 숫자, 날짜, 통화 형식 등 문화적 포맷

---

## 🛠 3. Localization 설정 흐름

1. `services.AddLocalization()` 등록
2. 리소스 파일 (.resx) 생성
3. `RequestCultureProvider`로 문화권 설정
4. Razor/MVC에서 `IStringLocalizer` 사용

---

## ⚙️ 4. 서비스 등록 및 미들웨어 설정

### 📄 Program.cs (.NET 6+)

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddLocalization(options => options.ResourcesPath = "Resources");

builder.Services.Configure<RequestLocalizationOptions>(options =>
{
    var supportedCultures = new[] { "en-US", "ko-KR", "fr-FR" }
        .Select(c => new CultureInfo(c)).ToList();

    options.DefaultRequestCulture = new RequestCulture("en-US");
    options.SupportedCultures = supportedCultures;
    options.SupportedUICultures = supportedCultures;

    // 쿼리스트링, 쿠키, 헤더 순서대로 감지
    options.RequestCultureProviders = new[]
    {
        new QueryStringRequestCultureProvider(),
        new CookieRequestCultureProvider(),
        new AcceptLanguageHeaderRequestCultureProvider()
    };
});

builder.Services.AddRazorPages();

var app = builder.Build();

// Localization 미들웨어 추가
var locOptions = app.Services.GetRequiredService<IOptions<RequestLocalizationOptions>>();
app.UseRequestLocalization(locOptions.Value);

app.MapRazorPages();
app.Run();
```

---

## 📁 5. 리소스 파일 (.resx) 구성

### 🔹 예시 구조 (Resources 폴더)

```
Resources/
├── Pages/
│   ├── Index.ko.resx       👉 한국어
│   ├── Index.en.resx       👉 영어
├── SharedResource.resx     👉 공용 리소스
```

### 🔹 파일명 규칙

- 페이지 이름 또는 클래스 이름 + `.문화코드.resx`
- 예: `Index.ko.resx`, `SharedResource.fr.resx`

---

## 🧾 6. Razor Page에서 사용

### 📄 Index.cshtml.cs

```csharp
public class IndexModel : PageModel
{
    private readonly IStringLocalizer<IndexModel> _localizer;

    public IndexModel(IStringLocalizer<IndexModel> localizer)
    {
        _localizer = localizer;
    }

    public string Message { get; private set; }

    public void OnGet()
    {
        Message = _localizer["Welcome"];
    }
}
```

### 📄 Index.cshtml

```razor
<h2>@Model.Message</h2>
```

---

## 🌐 7. 문화권 전환 방법 (사용자 선택)

### 📄 Culture 설정용 핸들러

```csharp
public IActionResult OnPostSetLanguage(string culture)
{
    Response.Cookies.Append(
        CookieRequestCultureProvider.DefaultCookieName,
        CookieRequestCultureProvider.MakeCookieValue(new RequestCulture(culture)),
        new CookieOptions { Expires = DateTimeOffset.UtcNow.AddYears(1) }
    );

    return RedirectToPage();
}
```

### 📄 언어 선택 UI

```razor
<form method="post" asp-page-handler="SetLanguage">
    <select name="culture" onchange="this.form.submit()">
        <option value="en-US">English</option>
        <option value="ko-KR">한국어</option>
        <option value="fr-FR">Français</option>
    </select>
</form>
```

---

## ✨ 8. View에서 직접 `IViewLocalizer` 사용

```razor
@inject IViewLocalizer Localizer

<h2>@Localizer["Welcome"]</h2>
```

> Razor View 내에서 직접 다국어 처리 가능

---

## 📦 9. 데이터 주석 (유효성 메시지) 다국어 처리

### 📄 Startup 설정 추가

```csharp
services.AddMvc()
    .AddViewLocalization()
    .AddDataAnnotationsLocalization();
```

### 📄 모델에서 메시지 지정

```csharp
public class ContactModel
{
    [Required(ErrorMessage = "NameRequired")]
    public string Name { get; set; }
}
```

→ `ContactModel.ko.resx` 파일에 `"NameRequired"` 키를 추가하여 다국어 메시지 구성

---

## 💡 10. 날짜/숫자/통화 형식 자동 처리

```razor
@DateTime.Now.ToString("D", CultureInfo.CurrentCulture)
@1234567.ToString("N", CultureInfo.CurrentCulture)
```

> 사용자의 현재 `CultureInfo`에 따라 자동으로 포맷이 다르게 출력됨

---

## 🧩 11. 실무 고려사항

| 항목 | 권장 방법 |
|------|------------|
| 문자열 키 관리 | 상수 또는 CodeGenerator로 관리 |
| 리소스 중복 제거 | SharedResource 사용 |
| 자주 변경되는 번역 | DB 또는 JSON 기반 동적 번역 고려 |
| 퍼포먼스 | 리소스 캐싱 및 정적 컴파일 고려 |
| 번역 주기 | Excel → .resx 변환 자동화 도구 활용

---

## ✅ 요약

| 주제 | 핵심 요약 |
|------|-----------|
| 기본 구성 | `AddLocalization`, `.resx`, `IStringLocalizer` |
| 라우트 문화권 적용 | 쿼리/쿠키/헤더 방식 감지 |
| Razor 사용법 | `IViewLocalizer`, `Model Localizer` |
| 문화권 포맷 | `CultureInfo.CurrentCulture` 기준으로 날짜/숫자 처리 |
| 실무 전략 | 공통 리소스, 상수 키, 리소스 자동화 관리