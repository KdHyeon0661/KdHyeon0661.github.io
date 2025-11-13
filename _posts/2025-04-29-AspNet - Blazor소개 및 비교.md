---
layout: post
title: AspNet - Blazor소개 및 비교
date: 2025-04-29 21:20:23 +0900
category: AspNet
---
# Blazor 소개 및 기존 기술들과의 비교

## 1. Blazor란? (요약 복습 + 배경)

**Blazor**는 C#과 Razor(Component)로 **SPA(단일 페이지 애플리케이션)**를 작성하는 웹 UI 프레임워크다. JavaScript를 완전히 대체하진 않지만, JS 없이도 대부분의 UI를 만들 수 있고, 필요할 땐 **JS Interop**으로 상호연동한다.

- 언어: C# (공유 모델/검증/DTO를 서버와 재사용)
- 뷰 엔진: Razor + 컴포넌트 기반
- 런타임: .NET WebAssembly(클라이언트) 또는 .NET 서버(Blazor Server)
- 핵심 장점: **C# 풀스택**, **재사용 가능한 컴포넌트**, **ASP.NET Core 생태계와 밀접 통합**

---

## 2. Blazor의 주요 특징 (확장)

| 항목 | 설명 | 운영 팁 |
|---|---|---|
| C# 기반 | UI/상태/도메인 로직을 C#로 통일 | 서버/클라이언트 **공유 프로젝트**로 DTO·검증 특성 재활용 |
| Razor 컴포넌트 | .razor 파일로 캡슐화 (매개변수/이벤트/렌더링) | **Partial 클래스**로 UI/코드 분리, 테스트성↑ |
| SPA | 라우팅/상태/클라이언트 내 네비게이션 | **프리렌더링**과 **Streaming Rendering**으로 FCP 개선 |
| 호스팅 모델 | Server/WASM/Hybrid/United(SSR+인터랙티브) | 요구사항별로 **지연/대역폭/보안/오프라인** 축에서 선택 |
| JS Interop | 필요한 부분만 JS 호출/호출받기 | 복잡한 UI(차트/지도)는 JS 라이브러리와 혼합 전략 |
| 상태 관리 | Cascading, 상태 컨테이너, Fluxor 등 | **회로(Blazor Server)**·**브라우저 저장소(WASM)** 고려 |

---

## 3. 호스팅 모델 비교 (Server / WASM / Hybrid / “United”)

### 3.1 개념 요약

| 모델 | 실행 위치 | UI 업데이트 | 장점 | 단점 | 적합 사례 |
|---|---|---|---|---|---|
| **Blazor Server** | 서버(.NET) | SignalR 회로(Circuit) | 초기 로딩 빠름, 보안/비밀 키 서버측 보관 | 서버/네트워크 의존, 지연 민감 | 기업 내망, 즉시 반응형 백오피스 |
| **Blazor WebAssembly** | 브라우저(WebAssembly) | 클라이언트 렌더 | 오프라인/PWA, 서버 부하↓ | 초기 DLL 다운로드, 일부 .NET API 제한 | 공개 사이트, 모바일 친화 SPA |
| **Blazor Hybrid** | 데스크탑/모바일(.NET MAUI) | WebView | 네이티브 API 접근, 배포 일관성 | 앱 배포/스토어 승인 필요 | 사내 도구, 장치 통합 앱 |
| **“United”(.NET 8+)** | SSR + 인터랙티브 | 필요 구간만 클라 상호작용 | SEO/초기 페인트↑, 하이브리드 아키텍처 | 구성 복합도↑ | 마케팅+앱 대시보드 복합 사이트 |

> “United”는 서버 렌더(SSR)를 기본으로 하고, 특정 섹션만 클라이언트 상호작용을 붙이는 **온디맨드 상호작용** 방식(서버/자동/WASM 선택)을 지향한다.

### 3.2 간단한 기준
- **지연 시간/네트워크 품질이 낮거나 보안 민감(내부망)** → **Server**
- **오프라인/PWA/모바일 네트워크 환경** → **WASM**
- **장치 API·오프라인·배포 통합** → **Hybrid**
- **SEO 최우선 + 필요한 곳만 상호작용** → **United(SSR+Interactive)**

---

## 4. 시작 예제 (Server/WASM)

### 4.1 Blazor Server Program.cs (기본 골자)

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddRazorComponents().AddInteractiveServerComponents(); // .NET 8 스타일
builder.Services.AddSignalR(); // 회로에 필요 (내부적으로 포함되지만 옵션 튜닝용)
var app = builder.Build();

if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler("/Error");
    app.UseHsts();
}

app.UseStaticFiles();
app.MapRazorComponents<App>()
   .AddInteractiveServerRenderMode(); // 서버 인터랙티브
app.Run();
```

### 4.2 Blazor WASM Program.cs (클라이언트)

```csharp
using Microsoft.AspNetCore.Components.WebAssembly.Hosting;

var builder = WebAssemblyHostBuilder.CreateDefault(args);
builder.RootComponents.Add<App>("#app");

// WASM 클라이언트 서비스 등록
builder.Services.AddScoped(sp => new HttpClient { BaseAddress = new Uri(builder.HostEnvironment.BaseAddress) });

await builder.Build().RunAsync();
```

---

## 5. 기본 컴포넌트 예제 확장

### 5.1 Counter.razor (양방향 바인딩/이벤트)

```razor
<h3>Counter</h3>

<p>Current count: @count</p>

<input type="number" @bind="step" min="1" />
<button class="btn btn-primary" @onclick="() => count += step">Increment</button>

@code {
    private int count = 0;
    private int step = 1;
}
```

### 5.2 매개변수/콜백/상태 상승(Lifting State)

**Child.razor**
```razor
<div class="card">
  <div class="card-body">
    <p>Value: @Value</p>
    <button class="btn btn-secondary" @onclick="Increment">+1</button>
  </div>
</div>

@code {
    [Parameter] public int Value { get; set; }
    [Parameter] public EventCallback<int> ValueChanged { get; set; }
    private async Task Increment() => await ValueChanged.InvokeAsync(Value + 1);
}
```

**Parent.razor**
```razor
<h4>Parent</h4>
<Child Value="@num" ValueChanged="@OnChanged" />

@code {
    private int num = 0;
    private void OnChanged(int v) => num = v;
}
```

---

## 6. 라우팅과 네비게이션

```razor
@page "/products/{id:int}"
@inject NavigationManager Nav

<h3>Product @id</h3>
<button @onclick="GoHome">Home</button>

@code {
    [Parameter] public int id { get; set; }
    void GoHome() => Nav.NavigateTo("/");
}
```

- 고급: **라우트 제약**(정수/정규식), **QueryString** 파싱, **라우트 데이터 바인딩**

---

## 7. 상태 관리 전략

### 7.1 CascadingParameter
```razor
<!-- App.razor -->
<CascadingValue Value="_theme">
    <Router AppAssembly="@typeof(App).Assembly" />
</CascadingValue>

@code {
    private string _theme = "dark";
}
```

```razor
@* Child *@
@code {
    [CascadingParameter] public string Theme { get; set; } = "light";
}
```

### 7.2 상태 컨테이너(서비스) 패턴
```csharp
public class AppState
{
    public int Counter { get; private set; }
    public event Action? OnChange;
    public void Inc() { Counter++; OnChange?.Invoke(); }
}
```

```csharp
// Program.cs
builder.Services.AddSingleton<AppState>();
```

```razor
@inject AppState State

<p>@State.Counter</p>
<button @onclick="State.Inc">+</button>

@code {
    protected override void OnInitialized() => State.OnChange += StateHasChanged;
    public void Dispose() => State.OnChange -= StateHasChanged;
}
```

### 7.3 Fluxor(Flux/Redux 스타일) 스니펫
```bash
dotnet add package Fluxor.Blazor.Web
```

```csharp
builder.Services.AddFluxor(o => o.ScanAssemblies(typeof(Program).Assembly)
                                 .UseReduxDevTools());
```

- 장점: 예측 가능한 상태/타임트래블 디버깅
- 단점: 러닝 커브, 보일러플레이트

### 7.4 Blazor Server 특수 고려
- **회로 복원/끊김 처리**(네트워크 장애)
- **유저별 상태**는 **Scoped** 또는 **회로 바인딩** 서비스로 관리
- **서버 메모리 사용량**/동접 사용량에 따라 **스케일아웃(백플레인)** 필요

---

## 8. 폼/검증(EditForm + DataAnnotations)

### 8.1 모델과 검증 속성
```csharp
public class SignUpModel
{
    [Required, EmailAddress]
    public string Email { get; set; } = "";
    [Required, MinLength(8)]
    public string Password { get; set; } = "";
    [Compare(nameof(Password))]
    public string Confirm { get; set; } = "";
}
```

### 8.2 Razor 폼
```razor
@page "/signup"
@using System.ComponentModel.DataAnnotations

<EditForm Model="@model" OnValidSubmit="Submit">
    <DataAnnotationsValidator />
    <ValidationSummary />

    <InputText @bind-Value="model.Email" />
    <InputText @bind-Value="model.Password" type="password" />
    <InputText @bind-Value="model.Confirm" type="password" />

    <button class="btn btn-primary">Sign Up</button>
</EditForm>

@code {
    private SignUpModel model = new();
    private Task Submit() { /* 서버 전송 */ return Task.CompletedTask; }
}
```

- 복잡 검증: **IValidatableObject**, **FluentValidation** 통합

---

## 9. JS Interop (필수 작법)

### 9.1 .NET→JS 호출
```razor
@inject IJSRuntime JS

<button @onclick="Show">Alert</button>

@code {
    private async Task Show() =>
        await JS.InvokeVoidAsync("alert", "Hello from .NET");
}
```

### 9.2 JS→.NET 호출
```csharp
public class JsCallback
{
    [JSInvokable]
    public static string Echo(string msg) => $"[.NET] {msg}";
}
```

```js
// wwwroot/js/site.js
window.callDotNet = async () => {
  const result = await DotNet.invokeMethodAsync('YourAssemblyName', 'Echo', 'ping');
  console.log(result);
};
```

- 복잡 UI(차트/지도/에디터)는 **Interop 래퍼 컴포넌트** 작성 권장
- 성능: JSObjectReference 재사용, 빈번 호출 최소화

---

## 10. Blazor Server 운영 이슈: Circuit/SignalR/스케일링

- **SignalR 백플레인**: Redis/Azure SignalR Service로 다중 서버 확장
- **상태 보존**: 각 사용자는 서버 메모리에 세션 유사 상태가 존재 → **세션 핀닝** 또는 **스티키 세션** 필요
- **지연 예산 계산**
  네트워크 왕복(RTT) \( r \), 인터랙션당 업데이트 \( n \) 회라면 체감 지연 \( L \)은
  $$
  L \approx n \cdot r + t_{server} + t_{render}
  $$
  → 모바일/원거리 사용자는 **WASM/United**가 유리할 수 있다.

---

## 11. Blazor WASM 성능/번들 최적화

- **트리밍(Trim), AOT(선택), PWA 캐시**, **Lazy Loading(Assembly)**
- **압축/HTTP/2/3**, CDN 활용
- 대형 라이브러리 JS Interop → **동적 import**/지연 로딩

**WASM AOT 예시(.NET 8)**
```xml
<PropertyGroup>
  <RunAOTCompilation>true</RunAOTCompilation>
</PropertyGroup>
```
- AOT는 **용량↑/빌드시간↑** vs **런타임 성능↑** 트레이드오프

---

## 12. 인증/권한(요약)

- **ASP.NET Core Identity**(Server) 또는 **JWT/OAuth/OIDC**(WASM + API)
- Server: 쿠키 기반이 자연스러움
- WASM: **토큰 기반** 권장(Access/Refresh), 보관소(XSS 위험 대비)
- “United”: SSR 경로는 쿠키, 인터랙티브 섹션은 **기존 인증 컨텍스트 재사용**

---

## 13. 에러 처리/로깅/진단

- UI: **ErrorBoundary** 컴포넌트로 컴포넌트 단위 격리
```razor
<ErrorBoundary>
  <ChildContent>
    <ProblematicComponent />
  </ChildContent>
  <ErrorContent>
    <p>문제가 발생했습니다.</p>
  </ErrorContent>
</ErrorBoundary>
```

- 로깅: `ILogger<T>`, Blazor WASM은 **콘솔 전송** 또는 **서버 수집 API** 필요
- 성능 측정: 브라우저 Performance 탭 + 서버 측 **분석/분포 트레이싱**

---

## 14. 스타일/컴포넌트 라이브러리

- **Bootstrap** 기본 템플릿, **MudBlazor**, **Radzen**, **Syncfusion** 등
- 서버/클라 공통 UI 키트 선택 시 **라이선스/번들 크기** 고려
- **CSS 격리**(`Component.razor.css`)로 컴포넌트 단위 스타일 캡슐화

---

## 15. 배포 패턴

- **Server**: 일반 ASP.NET Core처럼 배포(IIS, Nginx+Kestrel, Docker, Azure App Service) + SignalR 백플레인
- **WASM**: 정적 파일(호스팅된 API와 분리 가능). **Azure Static Web Apps**, Nginx 정적 호스팅
- **United**: ASP.NET Core 앱으로 배포, SSR + 인터랙티브 섹션 협업

---

## 16. 기존 프론트엔드(React/Vue/Angular)와 비교(확장)

| 축 | React/Vue/Angular | Blazor 관점 |
|---|---|---|
| 언어 | TypeScript/JavaScript | C# |
| 렌더링 | CSR/SSR/하이브리드(Vite/Next/Nuxt) | Server/WASM/United(SSR+Interactive) |
| 생태계 | 방대, NPM 중심 | 성숙 중, NuGet+Interop |
| 인력 수급 | JS 인력 많음 | .NET 인력에게 유리 |
| 번들/성능 | 최적화 도구 다양 | WASM 다운로드/트리밍/AOT 전략 필요 |
| 오프라인 | PWA 용이 | WASM PWA 강점 |
| SEO | SSR/ISR 풍부 | United/SSR로 보완 |
| 마이그레이션 | 풍부한 가이드/도구 | Interop 또는 Mix & Match(부분 도입) |

**전략**: 조직 내 **.NET 역량이 강하고** 프론트 코드/검증/모델을 **C#로 통일**하고 싶다면 Blazor가 큰 생산성 이득.

---

## 17. 실전 예제 모음

### 17.1 서버 렌더 + 인터랙티브 섹션(United풍)

**App.razor**
```razor
<Router AppAssembly="@typeof(App).Assembly" />
```

**Index.razor**
```razor
@page "/"

<h1>SSR로 처음 보여주고, 아래 위젯은 인터랙티브</h1>
<StaticMarketingSection />

<ClientInteractiveSection />

@code {
    // StaticMarketingSection: 순수 SSR 컴포넌트
    // ClientInteractiveSection: AddInteractiveServerRenderMode() 또는 WASM 모드 지정 가능
}
```

### 17.2 브라우저 저장소(로컬/세션) 사용 예 (WASM)

```bash
dotnet add package Blazored.LocalStorage
```

```csharp
builder.Services.AddBlazoredLocalStorage();
```

```razor
@inject Blazored.LocalStorage.ILocalStorageService LocalStorage

<button @onclick="Save">Save</button>
<button @onclick="Load">Load</button>
<p>@value</p>

@code {
    string value = "";
    async Task Save() => await LocalStorage.SetItemAsync("key","hello");
    async Task Load() => value = await LocalStorage.GetItemAsync<string>("key");
}
```

---

## 18. 보안 팁

- **XSS/CSRF**: WASM은 토큰 보관 위치 주의(가능하면 메모리+리프레시 전략)
- **Blazor Server**: 회로 고정/권한 확인은 **서버 측**에서 강제, 중요 로직은 **서버만** 접근 가능
- **콘텐츠 보안 정책(CSP)**, **HTTPS/HSTS**, **쿠키 보안 플래그** 적용
- **무결성/난독화**: WASM DLL은 디컴파일 가능 → **서버 신뢰 경계 유지**

---

## 19. 테스트 전략

- **컴포넌트 단위 테스트**: [bUnit] 사용하여 렌더링/이벤트/검증
- **E2E**: Playwright/Selenium으로 라우팅/폼/인증 플로우 검증
- **상태/서비스 단위 테스트**: xUnit + Moq/FluentAssertions

간단 bUnit 스니펫:
```csharp
using Bunit;
using Xunit;

public class CounterTests : TestContext
{
    [Fact]
    public void Increment_ShouldIncreaseCount()
    {
        var cut = RenderComponent<Counter>();
        cut.Find("button").Click();
        cut.MarkupMatches(@"<h3>Counter</h3><p>Current count: 1</p><button class=""btn btn-primary"">Click me</button>");
    }
}
```

---

## 20. 성능 체크리스트

- Server
  - 대기 시간 높은 구간 → **인터랙션 최소화**, **서버 CPU/메모리/회로 수 모니터링**
  - **Redis/Azure SignalR 백플레인** 구성
- WASM
  - **트리밍/AOT/Assembly Lazy Load**
  - 대형 라이브러리 지연 로드, 이미지/폰트 최적화
  - PWA/캐시 전략 튜닝
- 공통
  - **ErrorBoundary**로 장애 격리
  - 프로파일링/로그 표준화(Serilog 등)
  - **빌드 파이프라인에서 Lighthouse/Playwright**로 품질 게이트

---

## 21. 배포 시나리오 요약

- **Blazor Server**: ASP.NET Core 표준 배포(IIS, Nginx+Kestrel, Docker, Azure App Service) + SignalR 스케일 아웃
- **Blazor WASM**: 정적 자산/CDN + API 서버 분리(or Host in ASP.NET Core)
  - Azure Static Web Apps, CloudFront/S3, Nginx 정적 호스팅
- **Hybrid**: .NET MAUI 배포 채널(스토어/엔터프라이즈 서명)
- **United**: ASP.NET Core 앱 배포 + 인터랙티브 섹션 모드 설정

---

## 22. 요약 정리

| 항목 | 핵심 |
|---|---|
| Blazor 핵심 | C# 기반 SPA, Razor 컴포넌트, ASP.NET Core와 자연스러운 통합 |
| 호스팅 모델 | Server(회로/저지연), WASM(PWA/오프라인), Hybrid(네이티브), United(SSR+인터랙티브) |
| 상태/폼 | Cascading/상태 컨테이너/Fluxor, EditForm+DataAnnotations/FluentValidation |
| JS Interop | 필요 부분만 JS를 감싸서 사용(차트/지도/에디터) |
| 성능 전략 | Server: 백플레인/회로 관리, WASM: 트리밍/AOT/지연로딩 |
| 보안/배포 | 쿠키/토큰 전략, CSP/HSTS, Server 표준 배포 / WASM 정적 호스팅 |
| 테스트 | bUnit(컴포넌트), Playwright(E2E), xUnit+Moq(서비스) |

---

## 23. 다음 단계 제안

1. Blazor WASM + ASP.NET Core API + JWT 인증 실습
2. United(SSR+Interactive) 패턴으로 마케팅 페이지 + 대시보드 통합
3. 상태 관리 비교 실험: 상태 컨테이너 vs Fluxor
4. 성능 튜닝 과제: WASM AOT/트리밍/지연로딩의 체감 효과 측정

---

## 부록 A) 간단 United 스타일 구성 예시(.NET 8)

**Program.cs**
```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddRazorComponents()
    .AddInteractiveServerComponents()    // 서버 인터랙티브
    .AddInteractiveWebAssemblyComponents(); // WASM 인터랙티브 선택 가능

var app = builder.Build();
app.UseStaticFiles();

app.MapRazorComponents<App>()
   .AddInteractiveServerRenderMode()
   .AddInteractiveWebAssemblyRenderMode(); // 페이지/섹션별 모드선택

app.Run();
```

**SomePage.razor**
```razor
@page "/some"

<h2>서버 SSR + 필요한 섹션은 클라이언트 모드</h2>

<ServerSideWidget />  @* 기본 SSR + 서버 인터랙티브 *@

<ClientOnlyWidget @rendermode="RenderMode.WebAssemblyPrerendered" />
```

---

## 부록 B) 간단 수식: WASM 다운로드 비용 추정

초기 다운로드 크기 \( S \) (압축 후), 네트워크 대역폭 \( B \), 초기 파싱/로드 시간 \( T_p \)가 있을 때,
초기 지연 \( T \)는 근사적으로

$$
T \approx \frac{S}{B} + T_p
$$

- **Trim/AOT/지연 로딩**은 \( S \)를 줄이고, 브라우저/장치 성능은 \( T_p \)에 영향을 미친다.
- 대역폭이 제한적인 모바일 환경에서는 \( S \)의 감소가 사용자 체감에 큰 이득을 준다.
