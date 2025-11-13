---
layout: post
title: AspNet - NET Core의 글로벌화
date: 2025-05-03 21:20:23 +0900
category: AspNet
---
# ASP.NET Core의 글로벌화(Localization)

## 0. 로드맵

1) **리소스 준비**: `.resx` or DB/JSON
2) **미들웨어**: `AddLocalization` + `UseRequestLocalization`
3) **문화권 발견(Providers)**: Query → Cookie → Header → (옵션) Route/Subdomain
4) **사용**: `IStringLocalizer<T>`, `IViewLocalizer`, `IHtmlLocalizer`
5) **데이터주석/유효성**: `AddDataAnnotationsLocalization()`
6) **형식화**: `CultureInfo.CurrentCulture`로 날짜/숫자/통화
7) **고급**: 라우트/링크, 플러럴/성별, 동적 번역, 캐싱/성능, 테스트/도구

---

## 1. 필수 구성: 서비스와 미들웨어

### 1.1 Program.cs (기본형 + 다중 Provider + 리소스 경로)
```csharp
using System.Globalization;
using Microsoft.AspNetCore.Localization;
using Microsoft.Extensions.Options;

var builder = WebApplication.CreateBuilder(args);

// 1) 리소스 경로 지정
builder.Services.AddLocalization(opts => opts.ResourcesPath = "Resources");

// 2) 지원 문화권 정의
var supported = new[] { "en-US", "ko-KR", "fr-FR" }
    .Select(c => new CultureInfo(c)).ToList();

builder.Services.Configure<RequestLocalizationOptions>(options =>
{
    options.DefaultRequestCulture = new RequestCulture("en-US");
    options.SupportedCultures = supported;
    options.SupportedUICultures = supported;

    // 3) 문화권 발견 순서 (쿼리→쿠키→헤더)
    options.RequestCultureProviders = new IRequestCultureProvider[]
    {
        new QueryStringRequestCultureProvider(),      // ?culture=ko-KR
        new CookieRequestCultureProvider(),           // .AspNetCore.Culture 쿠키
        new AcceptLanguageHeaderRequestCultureProvider()
    };
});

// MVC / Razor Pages + Localization
builder.Services
    .AddRazorPages()
    .AddViewLocalization()                // _ViewImports, View에서 Localizer 사용
    .AddDataAnnotationsLocalization();    // 데이터 주석 메시지 번역

var app = builder.Build();

// 4) Localization 미들웨어는 Routing 전에 두는 것을 권장
var loc = app.Services.GetRequiredService<IOptions<RequestLocalizationOptions>>();
app.UseRequestLocalization(loc.Value);

app.UseStaticFiles();
app.UseRouting();

app.MapRazorPages();

app.Run();
```

> **순서**: `UseRequestLocalization`은 **Routing 전** 초기화가 안전하다(모델바인딩/검증/뷰 렌더링의 문화 맥락을 맞춤).

---

## 2. 리소스(.resx) 구성 — 규칙/패턴

### 2.1 폴더 구조(권장)
```
Resources/
├── Pages/
│   ├── Index.en.resx
│   ├── Index.ko.resx
│   └── Index.fr.resx
├── SharedResource.resx        // 기본(중립) 문자열
├── SharedResource.ko.resx
└── Validation.resx            // 검증 공용 키 모음
```

- **페이지/컨트롤러별** 리소스: `Pages/Index.ko.resx` (클래스/뷰명에 매칭)
- **공용 리소스**: `SharedResource.resx` 패턴(“마커 클래스” 방식)

### 2.2 마커 클래스(SharedResource)
```csharp
public class SharedResource { } // 빈 클래스 (타입 기준 Localizer 생성)
```

사용:
```csharp
public class LayoutModel : PageModel
{
    private readonly IStringLocalizer<SharedResource> _S;
    public LayoutModel(IStringLocalizer<SharedResource> S) => _S = S;

    public string Title => _S["AppTitle"]; // SharedResource.resx의 AppTitle
}
```

> 파일명은 클래스명과 일치(네임스페이스 경로 포함)해야 `ResourceManagerStringLocalizer`가 찾기 쉽다.

---

## 3. Razor Pages/MVC에서 문자열 로컬라이징

### 3.1 PageModel에서 사용
```csharp
public class IndexModel : PageModel
{
    private readonly IStringLocalizer<IndexModel> _L;
    public IndexModel(IStringLocalizer<IndexModel> L) => _L = L;
    public string Message { get; private set; } = "";

    public void OnGet() => Message = _L["Welcome"]; // Index.*.resx: Welcome=...
}
```

### 3.2 View에서 사용 (`IViewLocalizer`)
```razor
@using System.Globalization
@inject Microsoft.AspNetCore.Mvc.Localization.IViewLocalizer L

<h2>@L["Welcome"]</h2>
<p>@DateTime.Now.ToString("D", CultureInfo.CurrentCulture)</p>
```

### 3.3 HTML 인코딩 제어 (`IHtmlLocalizer`)
```razor
@inject Microsoft.AspNetCore.Mvc.Localization.IHtmlLocalizer<SharedResource> H

<div>@H["RichText_Welcome"]</div> <!-- 리소스에 HTML 포함 시 안전히 인코딩 제어 -->
```

---

## 4. 문화권 전환 UI — 쿠키/쿼리/링크

### 4.1 Razor Pages 핸들러(쿠키 기록)
```csharp
public IActionResult OnPostSetLanguage(string culture, string? returnUrl = null)
{
    Response.Cookies.Append(
        CookieRequestCultureProvider.DefaultCookieName,
        CookieRequestCultureProvider.MakeCookieValue(new RequestCulture(culture)),
        new CookieOptions { Expires = DateTimeOffset.UtcNow.AddYears(1), IsEssential = true });

    return LocalRedirect(returnUrl ?? Url.Page("/Index")!);
}
```

### 4.2 선택 UI
```razor
<form method="post" asp-page-handler="SetLanguage">
    <select name="culture" onchange="this.form.submit()">
        <option value="en-US">English</option>
        <option value="ko-KR">한국어</option>
        <option value="fr-FR">Français</option>
    </select>
    <input type="hidden" name="returnUrl" value="@Context.Request.Path" />
</form>
```

### 4.3 쿼리스트링 방식 (간편 테스트)
- `https://site.com/?culture=ko-KR`
- Provider 우선순위를 “Query → Cookie → Header”로 구성했으므로 즉시 반영

---

## 5. 라우트 기반 Localization (`/{culture}/...`)

> SEO/북마크/시맨틱 URL에 유리. 예) `/ko-KR/products/42`

### 5.1 라우트 템플릿/제약
```csharp
app.UseRequestLocalization(loc.Value);

app.MapControllerRoute(
    name: "localizedDefault",
    pattern: "{culture=en-US}/{controller=Home}/{action=Index}/{id?}",
    constraints: new { culture = "en-US|ko-KR|fr-FR" } // 화이트리스트
);
```

### 5.2 라우트 Provider 추가
```csharp
public class RouteDataRequestCultureProviderEx : RequestCultureProvider
{
    public override Task<ProviderCultureResult?> DetermineProviderCultureResult(HttpContext context)
    {
        var culture = context.Request.RouteValues["culture"] as string;
        if (string.IsNullOrWhiteSpace(culture)) return Task.FromResult<ProviderCultureResult?>(null);
        return Task.FromResult<ProviderCultureResult?>(new ProviderCultureResult(culture, culture));
    }
}
```

등록 시 Provider 체인에 **맨 앞** 또는 **원하는 위치**에 삽입:
```csharp
builder.Services.PostConfigure<RequestLocalizationOptions>(opt =>
{
    var routeProvider = new RouteDataRequestCultureProviderEx();
    opt.RequestCultureProviders.Insert(0, routeProvider);
});
```

### 5.3 문화권 포함 링크 생성
```csharp
@{
    var ci = System.Globalization.CultureInfo.CurrentUICulture.Name;
}
<a asp-controller="Home" asp-action="Privacy" asp-route-culture="@ci">Privacy</a>
```

---

## 6. 데이터 주석/검증 메시지 다국어

### 6.1 서비스 구성(이미 등록됨)
```csharp
builder.Services
    .AddRazorPages()
    .AddViewLocalization()
    .AddDataAnnotationsLocalization(opts =>
    {
        // 옵션: 공유 리소스에서 데이터주석 메시지 찾기
        opts.DataAnnotationLocalizerProvider = (type, factory)
            => factory.Create(typeof(Validation)); // Validation.resx 키 사용
    });
```

### 6.2 모델
```csharp
public class ContactModel
{
    [Required(ErrorMessage = "NameRequired")]
    [StringLength(30, MinimumLength = 2, ErrorMessage = "NameLength")]
    public string Name { get; set; } = "";
}
```

### 6.3 리소스(Validation.ko.resx)
```
NameRequired = 이름은 필수입니다.
NameLength = 이름은 2자 이상 30자 이하여야 합니다.
```

> **클라이언트 측 검증**(jQuery Validation) 메시지도 현재 `Culture`에 맞춰 렌더링된다. 필요 시 jQuery Validation 글로벌 메시지 번역 스크립트를 추가.

---

## 7. 날짜/숫자/통화/모델바인딩

- **출력**: `ToString("C", CultureInfo.CurrentCulture)` → 통화
- **입력/모델바인딩**: 폼의 숫자/날짜 패턴은 **현재 Culture**를 따른다
  (예: `fr-FR` 는 쉼표가 소수점, `MM/dd/yyyy` vs `dd/MM/yyyy` 이슈)

### 7.1 Razor 사용 예
```razor
@using System.Globalization
<p>@DateTime.UtcNow.ToLocalTime().ToString("f", CultureInfo.CurrentCulture)</p>
<p>@(1234567.89.ToString("N", CultureInfo.CurrentCulture))</p>
<p>@(1234.5m.ToString("C", CultureInfo.CurrentCulture))</p>
```

### 7.2 커스텀 모델바인더(문화 고정 필드) — 선택
- 특정 필드를 “항상 en-US 서식”으로 파싱하고 싶다면 전용 ModelBinder를 붙인다(금융/내부API 수치 등).

---

## 8. Minimal API/Blazor에서의 Localization

### 8.1 Minimal API 예
```csharp
app.MapGet("/hello", (IStringLocalizer<SharedResource> L) => new { message = L["Hello"] });
```

### 8.2 Blazor (Server/WASM 공통)
- `@inject IStringLocalizer<App> L`
- WASM은 **리소스 번들 포함** 필요(위성 어셈블리), `blazor.boot.json`의 `resources` 섹션에 포함됨
- `CultureInfo.CurrentUICulture` 전환 시 `CultureChanged` 트리거 또는 네비게이션 리로드 전략 사용

```razor
@page "/"
@inject IStringLocalizer<SharedResource> L

<h3>@L["Welcome"]</h3>
<button @onclick="() => SetCulture("ko-KR")">한국어</button>

@code {
    private async Task SetCulture(string culture)
    {
        var ci = new System.Globalization.CultureInfo(culture);
        System.Globalization.CultureInfo.DefaultThreadCurrentCulture = ci;
        System.Globalization.CultureInfo.DefaultThreadCurrentUICulture = ci;
        // Blazor WASM은 JS interop으로 문서 문화 갱신/리로드가 필요할 수 있음
        await Task.Yield();
    }
}
```

---

## 9. 플러럴(복수)·성별·포매팅 — 실무 패턴

`.resx`는 기본적으로 **단순 키-값**이다. 다음 패턴 중 하나를 고려:

1) **키 분리**: `ItemsCount_Zero`, `ItemsCount_One`, `ItemsCount_Other`
2) **SmartFormat/Humanizer** 라이브러리 활용 (ICU 메시지 형식 유사)
3) **PO 파일(OrchardCore.Localization)** 사용(고급 언어 규칙/플러럴 규칙 포함)

### 9.1 키 분리 예
```csharp
public static string ItemsCount(IStringLocalizer<SharedResource> L, int n)
{
    return n switch
    {
        0 => L["ItemsCount_Zero"],
        1 => L["ItemsCount_One"],
        _ => string.Format(L["ItemsCount_Other"], n)
    };
}
```

리소스:
```
ItemsCount_Zero=항목이 없습니다.
ItemsCount_One=항목이 1개 있습니다.
ItemsCount_Other=항목이 {0}개 있습니다.
```

---

## 10. 동적 번역(데이터베이스/JSON) — 커스텀 Localizer

> 운영 중 자주 바뀌는 문구를 **DB**에서 관리하고 싶을 때

### 10.1 IStringLocalizer 구현 개략
```csharp
public class DbStringLocalizer : IStringLocalizer
{
    private readonly ITranslationStore _store; // DB/캐시 조회
    private readonly string _baseName;

    public DbStringLocalizer(ITranslationStore store, string baseName)
        => (_store, _baseName) = (store, baseName);

    public LocalizedString this[string name]
        => new(name, _store.Get(_baseName, name) ?? name, resourceNotFound: _store.Get(_baseName, name) is null);

    public LocalizedString this[string name, params object[] arguments]
        => new(name, string.Format(_store.Get(_baseName, name) ?? name, arguments));

    public IEnumerable<LocalizedString> GetAllStrings(bool includeParentCultures)
        => _store.GetAll(_baseName).Select(kv => new LocalizedString(kv.Key, kv.Value, false));
}
```

### 10.2 팩토리/서비스 등록
```csharp
public class DbStringLocalizerFactory : IStringLocalizerFactory
{
    private readonly ITranslationStore _store;
    public DbStringLocalizerFactory(ITranslationStore store) => _store = store;

    public IStringLocalizer Create(Type resourceSource) => new DbStringLocalizer(_store, resourceSource.FullName!);
    public IStringLocalizer Create(string baseName, string location) => new DbStringLocalizer(_store, baseName);
}

// Program.cs
builder.Services.AddSingleton<ITranslationStore, MySqlTranslationStore>();
builder.Services.AddSingleton<IStringLocalizerFactory, DbStringLocalizerFactory>();
```

> 실무에서는 **메모리 캐시 + 변경 알림(예: Redis Pub/Sub)** 로 핫스왑을 구현한다.

---

## 11. 성능/캐싱/보안

### 11.1 성능
- `.resx → 위성 어셈블리`는 **ResourceManager** 레벨 캐싱이 기본 제공
- Localizer를 빈번히 생성하기보다 **DI 주입**으로 재사용
- **키 정규화**(일관 키), **공용 리소스** 적극 활용
- 동적 번역(DB/JSON) 시 **메모리 캐시 + TTL + 변경 알림** 필수

### 11.2 보안
- **사용자 입력을 키로 사용하지 말 것**(키 주입 공격 방지)
- **HTML 포함 문자열**은 `IHtmlLocalizer` 사용(중복 인코딩/미인코딩 방지)
- **Right-To-Left(RTL)** 언어 지원 시 레이아웃/아이콘 방향성 점검

### 11.3 운영성(로깅/진단)
- 누락 키 탐지 로그: `resourceNotFound == true` 일 때 Warning
- `Accept-Language` 오용/무한 다양 문화권 → 지원 목록 강제 매핑

---

## 12. 국제화와 데이터 계층/정렬

- **DB 정렬/Collation**(예: SQL Server `Korean_100_CI_AI` vs `Latin1_General_*`)은 **검색/정렬**에 큰 영향
- UI 문화권과 DB Collation이 다르면 정렬 결과가 사용자 기대와 다를 수 있음 → **문서화/정책화**
- EF Core 쿼리 시 **서버측 정렬**이 DB Collation을 따른다(문화권별 정렬이 필요하면 별도 컬럼/별도 인덱스 고려)

---

## 13. 테스트(단위/통합) 시 문화 고정

### 13.1 단위 테스트에서 Culture 스코프
```csharp
public class CultureScope : IDisposable
{
    private readonly CultureInfo _origC, _origUi;
    public CultureScope(string culture)
    {
        _origC = CultureInfo.CurrentCulture;
        _origUi = CultureInfo.CurrentUICulture;
        var ci = new CultureInfo(culture);
        CultureInfo.CurrentCulture = ci;
        CultureInfo.CurrentUICulture = ci;
    }
    public void Dispose()
    {
        CultureInfo.CurrentCulture = _origC;
        CultureInfo.CurrentUICulture = _origUi;
    }
}
```

사용:
```csharp
[Fact]
public void Number_Format_frFR()
{
    using var _ = new CultureScope("fr-FR");
    var s = (1234.56m).ToString("N", CultureInfo.CurrentCulture);
    Assert.Contains(",", s); // 소수점이 쉼표
}
```

### 13.2 통합 테스트에서 Accept-Language 시뮬레이션
```csharp
var req = new HttpRequestMessage(HttpMethod.Get, "/");
req.Headers.AcceptLanguage.ParseAdd("ko-KR, en;q=0.8");
var res = await _client.SendAsync(req);
```

---

## 14. 진짜로 자주 겪는 실무 이슈 & 해결

| 이슈 | 원인 | 해결 |
|---|---|---|
| `@Html.ValidationMessageFor`가 영어 | DataAnnotationsLocalization 미구성 | `AddDataAnnotationsLocalization()` 추가 + Validation.resx |
| 숫자 입력이 지역마다 다르게 파싱 | 모델바인딩은 CurrentCulture에 의존 | 표준화 필드에는 커스텀 바인더 또는 입력 마스크/통일 포맷 |
| 누락 키가 화면에 그대로 | 리소스 미존재 | 공통 키/디자인 시스템으로 키 표준화, 로깅, 빌드 시 리소스 검사 |
| SEO에 언어별 URL 원함 | 라우트에 culture 삽입 | `{culture}` 라우트 + Provider, sitemap 언어별 링크 |
| Blazor WASM에 번역 반영 안됨 | 리소스 위성 어셈블리 누락 | Publish 설정/`blazor.boot.json` 확인, 위성 포함 |

---

## 15. 샘플: 종합 예제(페이지 + 라우트 + 공용/검증)

### 15.1 Program.cs 요약
```csharp
builder.Services.AddLocalization(o => o.ResourcesPath = "Resources");
builder.Services.AddRazorPages().AddViewLocalization().AddDataAnnotationsLocalization();

builder.Services.Configure<RequestLocalizationOptions>(opt =>
{
    var list = new[] { "en-US", "ko-KR", "fr-FR" }.Select(c => new CultureInfo(c)).ToList();
    opt.DefaultRequestCulture = new RequestCulture("en-US");
    opt.SupportedCultures = list; opt.SupportedUICultures = list;

    opt.RequestCultureProviders = new IRequestCultureProvider[]
    {
        new RouteDataRequestCultureProviderEx(), // 1) Route
        new QueryStringRequestCultureProvider(), // 2) Query
        new CookieRequestCultureProvider(),      // 3) Cookie
        new AcceptLanguageHeaderRequestCultureProvider()
    };
});

var app = builder.Build();
app.UseRequestLocalization(app.Services.GetRequiredService<IOptions<RequestLocalizationOptions>>().Value);
app.UseRouting();
app.MapControllerRoute("localized", "{culture=en-US}/{controller=Home}/{action=Index}/{id?}");
app.MapRazorPages();
app.Run();
```

### 15.2 모델(검증 메시지 키)
```csharp
public class SignupModel
{
    [Required(ErrorMessage = "UsernameRequired")]
    [StringLength(20, MinimumLength = 3, ErrorMessage = "UsernameLength")]
    public string Username { get; set; } = "";

    [Required(ErrorMessage = "EmailRequired")]
    [EmailAddress(ErrorMessage = "EmailInvalid")]
    public string Email { get; set; } = "";
}
```

### 15.3 Razor (View/Pages) — 공용 + 페이지 로컬라이저
```razor
@page
@model IndexModel
@inject Microsoft.AspNetCore.Mvc.Localization.IViewLocalizer L
@inject Microsoft.Extensions.Localization.IStringLocalizer<SharedResource> S

<h1>@L["Welcome"]</h1>
<p>@S["AppDescription"]</p>

<form method="post">
    <input asp-for="Form.Username" />
    <span asp-validation-for="Form.Username"></span>

    <input asp-for="Form.Email" />
    <span asp-validation-for="Form.Email"></span>

    <button type="submit">@S["Submit"]</button>
</form>
```

### 15.4 리소스 예 (Validation.ko.resx)
```
UsernameRequired=사용자 이름은 필수입니다.
UsernameLength=사용자 이름은 3~20자여야 합니다.
EmailRequired=이메일은 필수입니다.
EmailInvalid=올바른 이메일 형식이 아닙니다.
```

---

## 16. 운영 체크리스트 (요약)

- [ ] `UseRequestLocalization`을 Routing 전에 구성
- [ ] Provider 순서: Route/Query/Cookie/Header 중 정책화
- [ ] `.resx` 네이밍/경로 표준(Shared/Validation/Feature별)
- [ ] DataAnnotationsLocalization 활성 + 공용 Validation 리소스
- [ ] 날짜/숫자/통화 포맷은 `CultureInfo.CurrentCulture`로 출력
- [ ] 플러럴/성별: 키 분리 또는 SmartFormat/Humanizer
- [ ] 동적 번역(DB/JSON) 시 캐시/무효화 설계
- [ ] 누락 키 로깅/테스트, Accept-Language 품질 필터
- [ ] SEO/UX: 라우트 문화권, 언어 스위처, RTL 레이아웃

---

## 결론

ASP.NET Core의 Localization은 **리소스 기반의 단순 문자열 치환**을 넘어,
**라우팅/검증/모델바인딩/형식화/SEO/성능/운영**까지 **전 시스템적인 문화권 인지 설계**가 핵심이다.
작게 시작(Shared/Validation 리소스 + Provider 3종)해서,
요구에 맞춰 **라우트 기반, 플러럴 처리, 동적 번역, 캐시/테스트 자동화**로 확장하라.
