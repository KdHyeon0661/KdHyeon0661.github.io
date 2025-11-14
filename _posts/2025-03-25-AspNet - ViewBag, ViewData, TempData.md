---
layout: post
title: AspNet - ViewBag, ViewData, TempData
date: 2025-03-25 19:20:23 +0900
category: AspNet
---
# ASP.NET Core — ViewBag · ViewData · TempData

## 한눈 비교(복습)

| 이름 | 타입/형태 | 수명 | 주 사용처 | 리다이렉트 유지 | 타입 안정성 |
|---|---|---|---|---|---|
| `ViewData` | `ViewDataDictionary` (키/값) | **현재 요청** | Controller → View 간 소량 전달 | ✗ | 낮음(캐스팅 필요) |
| `ViewBag` | `dynamic` (ViewData 래퍼) | **현재 요청** | 간단한 UI 표시 값 | ✗ | 낮음(런타임 바인딩) |
| `TempData` | `ITempDataDictionary` | **다음 요청까지** (1회성) | PRG/Flash 메시지 | **✓** | 중간(직렬화로 객체 가능) |

> 핵심: **ViewModel이 1순위**. 다만 **소량의 보조 정보**(페이지 타이틀, 토스트 메시지, 단발성 경고/성공 알림)에는 ViewData/ViewBag/TempData가 유용하다.

---

## ViewData — 딕셔너리로 전달(캐스팅 필요)

### Controller → View

```csharp
public IActionResult Index()
{
    ViewData["Title"] = "제품 목록";
    ViewData["Count"] = 10;               // boxing
    ViewData["ShowPrice"] = true;         // 옵션 스위치
    return View(products);
}
```

```cshtml
@* Views/Products/Index.cshtml *@
@model IEnumerable<ProductVm>

<h2>@(ViewData["Title"] as string)</h2>
<p>총 개수: @(ViewData["Count"] is int c ? c : 0)</p>

@foreach (var p in Model)
{
  <div class="item">@p.Name</div>
}

@section Scripts {
  <script>
    const showPrice = @((bool?)ViewData["ShowPrice"] == true ? "true" : "false");
  </script>
}
```

### Partial로 전달(전달 명시)

```cshtml
@* View에서 Partial 호출 시 ViewData 확장 *@
@{
    var vd = new ViewDataDictionary(ViewData) { ["Highlight"] = true };
}
<partial name="_ProductCard" model="p" view-data="vd" />
```

```cshtml
@* Views/Shared/_ProductCard.cshtml *@
@model ProductVm
@{ var hi = (bool?)ViewData["Highlight"] ?? false; }
<div class="card @(hi ? "highlight" : "")">
  <strong>@Model.Name</strong>
</div>
```

> **팁**: Partial에 **ViewData를 따로 주입**하면, 상위 ViewData를 오염시키지 않고 옵션을 전달할 수 있다.

---

## ViewBag — ViewData의 dynamic 래퍼(가독성↑, 컴파일 타임 체크×)

```csharp
public IActionResult Index()
{
    ViewBag.Title = "제품 목록";
    ViewBag.Count = 10;
    return View();
}
```

```cshtml
<h2>@ViewBag.Title</h2>
<p>총 개수: @ViewBag.Count</p>
```

**장점**: 짧고 읽기 쉬움
**단점**: 오타·형 변환은 **런타임 오류**로 터짐 → 린트/리뷰로 통제, **복잡 데이터엔 ViewModel 우선**

---

## TempData — 다음 요청까지 1회성 유지(PRG·Flash)

### 핵심 개념(정확한 동작 원리)

- ASP.NET Core의 기본 TempData Provider는 **CookieTempDataProvider**(쿠키 기반 직렬화)다.
  - 별도 설치 시 **`SessionStateTempDataProvider`** 로 전환 가능(세션 기반).
- **한 번 읽으면 소멸**(읽자마자 제거).
  - **`Peek()`**: 읽고 **유지**
  - **`Keep()`**: 특정 키(혹은 전체)를 **다음 요청까지 연장**

### PRG(Post-Redirect-Get) 패턴 예제

```csharp
[HttpPost]
public IActionResult Create(ProductCreateVm vm)
{
    if (!ModelState.IsValid) return View(vm);

    _service.Create(vm);

    TempData["Flash.Type"] = "success";
    TempData["Flash.Message"] = "상품이 등록되었습니다.";
    return RedirectToAction(nameof(Index)); // 새로고침 중복 방지
}
```

```csharp
public IActionResult Index()
{
    var flash = new FlashVm
    {
        Type = TempData.Peek("Flash.Type") as string,     // 유지
        Message = TempData["Flash.Message"] as string     // 1회성
    };
    return View(flash);
}
```

```cshtml
@* Views/Shared/_Flash.cshtml *@
@model FlashVm
@if (!string.IsNullOrWhiteSpace(Model?.Message))
{
  <div class="alert alert-@Model.Type">@Model.Message</div>
}
```

```cshtml
@* Views/Products/Index.cshtml *@
@model FlashVm
<partial name="_Flash" model="Model" />
```

### Keep/Peek 활용

```csharp
// 1) 한 번 더 보여주고 싶다면
var m = TempData["Flash.Message"]; // 읽으면 제거됨
TempData.Keep("Flash.Message");    // 해당 키 유지

// 2) 처음부터 유지하며 읽기
var type = TempData.Peek("Flash.Type") as string; // 제거되지 않음
```

### 복합 객체 저장(직렬화)

```csharp
var notice = new NoticeVm { Title = "공지", Lines = new[] { "A", "B" } };
TempData["Notice"] = JsonSerializer.Serialize(notice);   // 명시 직렬화(권장)
return RedirectToAction(nameof(Index));
```

```csharp
public IActionResult Index()
{
    var json = TempData["Notice"] as string;
    NoticeVm? vm = null;
    if (!string.IsNullOrEmpty(json))
        vm = JsonSerializer.Deserialize<NoticeVm>(json);
    return View(vm);
}
```

> TempData는 내부적으로도 직렬화를 수행하지만, **명시 JSON 직렬화**는 버전/호환/디버깅에 더 예측 가능하다.

---

## TempData Provider/Session 구성

### 기본(CookieTempDataProvider) — 별도 구성 없음

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddControllersWithViews();
var app = builder.Build();
app.MapDefaultControllerRoute();
app.Run();
```

- 기본은 **쿠키 기반**이며, 암호화된 쿠키에 JSON 직렬화 결과를 보관한다.

### 세션 기반 TempData로 전환(선택)

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddControllersWithViews();

// 세션 사용
builder.Services.AddSession();

// TempData Provider를 세션 기반으로 교체
builder.Services.AddSingleton<ITempDataProvider, SessionStateTempDataProvider>();

var app = builder.Build();
app.UseSession();               // 반드시 추가
app.MapDefaultControllerRoute();
app.Run();
```

> 세션 기반의 장점: 쿠키 크기 제약에서 자유로움. 단점: 서버 메모리/스토리지 부담, 스케일아웃 고려 필요.

---

## Layout/Partial/ViewComponent와 데이터 전달

### 레이아웃에서 ViewData/TempData 사용

```cshtml
@* _Layout.cshtml *@
@if (TempData["Global.Alert"] is string alert)
{
    <div class="alert alert-info">@alert</div>
}
<title>@(ViewData["PageTitle"] ?? "MyApp")</title>
```

```csharp
public IActionResult Privacy()
{
    ViewData["PageTitle"] = "개인정보 처리방침";
    TempData["Global.Alert"] = "신규 약관이 적용되었습니다.";
    return View();
}
```

### Partial로 옵션 전달(앞서 본 `view-data` 패턴)

### ViewComponent에서 ViewData 사용

```csharp
public class BannerViewComponent : ViewComponent
{
    public IViewComponentResult Invoke(string scope)
    {
        ViewData["Scope"] = scope; // VC 내부에서 ViewData 사용 가능
        return View("Default", new BannerVm { Text = "이벤트 진행 중" });
    }
}
```

```cshtml
@* Views/Shared/Components/Banner/Default.cshtml *@
@model BannerVm
<div class="banner">
  @Model.Text <small>(@ViewData["Scope"])</small>
</div>
```

호출:

```cshtml
<vc:banner scope="Home" />
```

---

## Razor Pages에서의 ViewData/ViewBag/TempData

### ViewData/ViewBag

```csharp
public class IndexModel : PageModel
{
    public void OnGet()
    {
        ViewData["Title"] = "Razor Pages 목록";
        ViewBag.Count = 42; // 가능
    }
}
```

```cshtml
@page
@model IndexModel
<h2>@ViewData["Title"]</h2>
<p>@ViewBag.Count</p>
```

### TempData — 속성 바인딩으로 간결하게

```csharp
public class CreateModel : PageModel
{
    [TempData]
    public string? FlashMessage { get; set; }

    public IActionResult OnPost()
    {
        // 저장…
        FlashMessage = "등록되었습니다.";
        return RedirectToPage("Index");
    }
}
```

```csharp
public class IndexModel : PageModel
{
    [TempData]
    public string? FlashMessage { get; set; }

    public void OnGet() { } // FlashMessage가 자동 바인딩
}
```

```cshtml
@page
@model IndexModel
@if (!string.IsNullOrEmpty(Model.FlashMessage))
{
  <div class="alert alert-success">@Model.FlashMessage</div>
}
```

---

## 플래시 메시지 시스템(토스트/Alert) 통합 — 실전 템플릿

### 필터로 TempData → ViewData 브리지(선택 패턴)

```csharp
public sealed class FlashToViewDataFilter : IResultFilter
{
    public void OnResultExecuting(ResultExecutingContext context)
    {
        if (context.Controller is Controller c)
        {
            if (c.TempData.TryGetValue("Flash.Type", out var type) &&
                c.TempData.TryGetValue("Flash.Message", out var msg))
            {
                c.ViewData["Flash.Type"] = type;
                c.ViewData["Flash.Message"] = msg;
                c.TempData.Keep("Flash.Type");
                c.TempData.Keep("Flash.Message");
            }
        }
    }
    public void OnResultExecuted(ResultExecutedContext context) { }
}
```

등록:

```csharp
builder.Services.AddControllersWithViews(o =>
{
    o.Filters.Add<FlashToViewDataFilter>();
});
```

레이아웃에서 한 번만 렌더링:

```cshtml
@* _Layout.cshtml *@
@if (ViewData["Flash.Message"] is string fmsg)
{
    <div class="toast @ViewData["Flash.Type"]">@fmsg</div>
}
```

> 장점: 개별 뷰에서 TempData를 직접 참조하지 않아도 됨.

---

## 테스트/디버깅 팁

### 단위 테스트에서 ViewData/TempData 검증

```csharp
[Fact]
public void Index_sets_title_and_count()
{
    // Arrange
    var c = new ProductsController(...);

    // Act
    var result = c.Index() as ViewResult;

    // Assert
    Assert.Equal("제품 목록", result?.ViewData["Title"]);
    Assert.Equal(10, result?.ViewData["Count"]);
}
```

### TempData 테스트

```csharp
var controller = new ProductsController(...);

// TempData 주입
controller.TempData = new TempDataDictionary(
    new DefaultHttpContext(),
    Mock.Of<ITempDataProvider>()
);

var result = controller.Create(...);

Assert.Equal("success", controller.TempData["Flash.Type"]);
```

### 디버깅

- **브라우저 쿠키**(CookieTempDataProvider): 이름 `TempData` 유사 키 존재 여부 확인
- TempData가 “사라진” 이슈 → **이미 읽혔는지**(다른 필터/뷰에서 접근) 확인

---

## 자주 겪는 함정과 해결

| 문제 | 원인 | 해결 |
|---|---|---|
| 리다이렉트 후 메시지 없음 | ViewBag/ViewData 사용 | **TempData** 사용, 또는 PRG 패턴 |
| TempData가 사라짐 | **읽자마자 제거**되는 특성 | `Peek()`/`Keep()`로 유지 |
| 큰 데이터 저장 실패/쿠키 초과 | CookieTempDataProvider 크기 한계 | 세션 기반 TempData로 전환 또는 DB/캐시 사용 |
| ViewBag NullReference | 오타/동적 바인딩 오류 | **ViewModel 우선**, 최소한 `ViewData`로 키 문자열 관리 |
| Partial에서 부모 ViewData 덮어씀 | 같은 키 사용 | Partial 호출 시 **`new ViewDataDictionary(ViewData)`** 로 범위 분리 |
| 레이아웃에서 TempData 읽고 뷰에서 또 쓰려함 | 한 번 읽은 뒤 제거됨 | 레이아웃에서 **`Peek()`** 사용 또는 필터로 브리지 |

---

## 권장 사용 시나리오 정리

- **ViewModel 우선**: 페이지 핵심 데이터는 항상 명시적 ViewModel
- **ViewData/ViewBag**: 레이아웃·헤더 타이틀, 작은 옵션 플래그, UI 전용 소량 데이터
- **TempData**: PRG의 1회성 안내, “성공/실패/경고” 알림, 리다이렉트 간 작은 상태 전달
- **객체 전달**: TempData에 **직렬화(명시 JSON)** 후 전송(크기 주의)

---

## 실전 종합 예제

### Controller

```csharp
public class ProductsController : Controller
{
    private readonly IProductService _svc;
    public ProductsController(IProductService svc) => _svc = svc;

    public IActionResult Index()
    {
        var items = _svc.GetList();
        ViewData["Title"] = "상품 목록";
        return View(items.Select(ProductVm.FromEntity));
    }

    public IActionResult Create() => View(new ProductEditVm());

    [HttpPost]
    public IActionResult Create(ProductEditVm vm)
    {
        if (!ModelState.IsValid) return View(vm);

        _svc.Create(vm.ToEntity());

        TempData["Flash.Type"] = "success";
        TempData["Flash.Message"] = "상품이 등록되었습니다.";
        return RedirectToAction(nameof(Index));
    }

    public IActionResult Edit(int id)
    {
        var entity = _svc.Get(id);
        if (entity is null) return NotFound();
        return View(ProductEditVm.FromEntity(entity));
    }

    [HttpPost]
    public IActionResult Edit(int id, ProductEditVm vm)
    {
        if (!ModelState.IsValid) return View(vm);
        _svc.Update(id, vm.ToEntity());
        TempData["Flash.Type"] = "success";
        TempData["Flash.Message"] = "수정되었습니다.";
        return RedirectToAction(nameof(Index));
    }
}
```

### Views

```cshtml
@* Views/Products/Index.cshtml *@
@model IEnumerable<ProductVm>
@{
    ViewData["Title"] = ViewData["Title"] ?? "목록";
}
<h2>@ViewData["Title"]</h2>

<partial name="_Flash" model="new FlashVm {
    Type = TempData.Peek("Flash.Type") as string,
    Message = TempData["Flash.Message"] as string
}" />

<table class="table">
  <thead><tr><th>이름</th><th>가격</th><th></th></tr></thead>
  <tbody>
  @foreach (var p in Model)
  {
    <tr>
      <td>@p.Name</td>
      <td>@p.Price.ToString("C")</td>
      <td>
        <a asp-action="Edit" asp-route-id="@p.Id">수정</a>
      </td>
    </tr>
  }
  </tbody>
</table>

<a asp-action="Create" class="btn btn-primary">새로 만들기</a>
```

```cshtml
@* Views/Shared/_Flash.cshtml *@
@model FlashVm
@if (!string.IsNullOrWhiteSpace(Model?.Message))
{
  <div class="alert alert-@Model.Type">@Model.Message</div>
}
```

```cshtml
@* Views/Products/Create.cshtml *@
@model ProductEditVm
@{
    ViewData["Title"] = "상품 등록";
}
<h2>@ViewData["Title"]</h2>

<form asp-action="Create" method="post">
  <div class="mb-3">
    <label asp-for="Name" class="form-label"></label>
    <input asp-for="Name" class="form-control" />
    <span asp-validation-for="Name" class="text-danger"></span>
  </div>
  <div class="mb-3">
    <label asp-for="Price" class="form-label"></label>
    <input asp-for="Price" class="form-control" />
    <span asp-validation-for="Price" class="text-danger"></span>
  </div>
  <button class="btn btn-primary" type="submit">저장</button>
</form>

@section Scripts {
  <partial name="_ValidationScriptsPartial" />
}
```

---

## FAQ

**Q1. ViewData vs ViewBag 중 무엇을 써야 하나요?**
A. 기능상 동일. 팀 코드 스타일에 맞춰 **일관성** 있게 선택. 컴파일 타임 체크를 원하면 ViewData(캐스팅)보다 **ViewModel**을 권장.

**Q2. TempData는 어디에 저장되나요?**
A. 기본은 **쿠키 기반(CookieTempDataProvider)**. 세션 기반으로 바꾸려면 `SessionStateTempDataProvider` 구성 및 `UseSession()` 필요.

**Q3. TempData에 큰 객체를 넣어도 되나요?**
A. 지양. 쿠키 크기 제한/네트워크 오버헤드. 필요 시 세션 또는 DB/캐시(예: Redis)에 보관 후 키만 TempData로 전달.

**Q4. TempData가 갑자기 비어 있어요.**
A. 이미 읽혀서 제거되었을 가능성. **`Peek()`/`Keep()`** 사용 또는 레이아웃/필터 설계로 “읽기 위치”를 일관화.

---

## 체크리스트

- [ ] 페이지 핵심 데이터는 **ViewModel**로 전달했는가?
- [ ] 일회성 메시지는 **TempData + PRG**로 처리했는가?
- [ ] Partial에 옵션 전달 시 **`view-data`** 로 범위를 분리했는가?
- [ ] 레이아웃/뷰에서 TempData “중복 읽기”가 없는가? 필요 시 `Peek()`/`Keep()`
- [ ] 쿠키 크기/보안(민감 데이터 금지) 고려했는가? 필요 시 세션/서버 저장소 사용
- [ ] 테스트에서 ViewData/TempData 검증 케이스를 포함했는가?

---

## 결론

- **ViewModel 우선** 원칙을 지키되, **ViewData/ViewBag/TempData**는 **UI 보조 정보/플래시 메시지** 등에서 생산성을 크게 높여준다.
- TempData는 **PRG**와 찰떡궁합이며, **Cookie/Session Provider** 특성과 제약을 이해하고 사용하면 안정적이다.
- 레이아웃, Partial, ViewComponent, Razor Pages까지 **데이터 흐름의 위치와 시점**을 명확히 하면 유지보수성과 예측 가능성이 높아진다.
