---
layout: post
title: AspNet - DI, Middleware, Routing
date: 2025-05-10 23:20:23 +0900
category: AspNet
---
# 📦 ASP.NET Core 핵심 개념 요약: DI, Middleware, Routing

---

## 1️⃣ DI (Dependency Injection, 의존성 주입)

### ✅ 개념

- 객체 간 의존 관계를 **직접 생성하지 않고**, 외부에서 **주입**하는 방식.
- **관심사의 분리 (Separation of Concerns)**, **테스트 용이성**, **유지보수성 향상**을 위해 사용.
- ASP.NET Core는 **기본적으로 DI 컨테이너를 내장**하고 있음.

---

### ✅ 등록 방식

```csharp
builder.Services.AddTransient<IMyService, MyService>();
builder.Services.AddScoped<IUserService, UserService>();
builder.Services.AddSingleton<ILogger, ConsoleLogger>();
```

- **Transient**: 매번 새 인스턴스 생성 (stateless)
- **Scoped**: 요청(Request)당 하나
- **Singleton**: 앱 시작부터 종료까지 하나

---

### ✅ 사용 방식 (생성자 주입)

```csharp
public class HomeController : Controller
{
    private readonly IMyService _service;

    public HomeController(IMyService service)
    {
        _service = service;
    }

    public IActionResult Index()
    {
        var data = _service.GetData();
        return View(data);
    }
}
```

---

## 2️⃣ Middleware (미들웨어)

### ✅ 개념

- HTTP 요청(Request)와 응답(Response)을 처리하는 **중간 구성 요소**
- 요청 → 미들웨어 체인 → 라우팅 → 컨트롤러
- 응답 ← 미들웨어 체인 ← 응답 반환

> ASP.NET Core는 전통적인 ASP.NET의 HTTP Module/Handler를 대신해 **미들웨어 기반 파이프라인**을 사용함.

---

### ✅ 흐름

```plaintext
클라이언트 → [UseRouting] → [UseAuthentication] → [UseAuthorization] → [UseEndpoints] → 응답
```

### ✅ 등록 예

```csharp
var app = builder.Build();

app.UseStaticFiles();       // wwwroot에서 정적 파일 제공
app.UseRouting();           // 라우팅 활성화
app.UseAuthentication();    // 인증
app.UseAuthorization();     // 권한
app.UseEndpoints(endpoints =>
{
    endpoints.MapRazorPages();
});
```

---

### ✅ 커스텀 미들웨어

```csharp
public class LoggingMiddleware
{
    private readonly RequestDelegate _next;

    public LoggingMiddleware(RequestDelegate next) => _next = next;

    public async Task InvokeAsync(HttpContext context)
    {
        Console.WriteLine("요청 시작: " + context.Request.Path);
        await _next(context); // 다음 미들웨어로 진행
        Console.WriteLine("응답 완료: " + context.Response.StatusCode);
    }
}

// 등록
app.UseMiddleware<LoggingMiddleware>();
```

---

## 3️⃣ Routing (라우팅)

### ✅ 개념

- URL 경로를 기반으로 **컨트롤러, 액션, Razor Page 등을 호출**하는 매핑 시스템
- 미들웨어의 일부로 `UseRouting()`과 `UseEndpoints()`에 의해 처리됨

---

### ✅ Razor Pages 라우팅

```csharp
app.MapRazorPages(); // /Pages/Index.cshtml → "/"
```

경로는 파일 위치 기반:

```plaintext
Pages/
├── Index.cshtml        → "/"
├── Contact.cshtml      → "/Contact"
├── Products/
│   └── Detail.cshtml   → "/Products/Detail"
```

---

### ✅ MVC 라우팅

```csharp
app.MapControllerRoute(
    name: "default",
    pattern: "{controller=Home}/{action=Index}/{id?}");
```

URL `/Product/Detail/3` → `ProductController.Detail(int id = 3)`

---

### ✅ API 라우팅 예시

```csharp
[ApiController]
[Route("api/[controller]")]
public class UsersController : ControllerBase
{
    [HttpGet("{id}")]
    public IActionResult Get(int id) => Ok(id);
}
```

→ `/api/users/5` 요청 → `Get(5)` 실행

---

### ✅ 라우트 제약 조건 예시

```csharp
[HttpGet("user/{id:int:min(1)}")]
public IActionResult GetUser(int id) => Ok(id);
```

→ id는 정수이면서 최소값 1 이상만 허용

---

## 🔄 관계 요약

| 개념 | 주요 목적 | 코드 위치 | 순서 |
|------|----------|-----------|------|
| DI | 객체 간 의존성 관리 | `builder.Services` | 앱 시작 시 |
| Middleware | 요청/응답 흐름 제어 | `app.Use...()` | 요청 처리 체인 |
| Routing | URL → 처리기 매핑 | `app.UseRouting`, `app.Map...` | 미들웨어 내 포함 |

---

## ✅ 예제 흐름 정리

```plaintext
1. 클라이언트 요청
2. Middleware 흐름 시작
    → UseStaticFiles()
    → UseRouting()
    → UseAuthentication()
    → UseAuthorization()
3. Routing 결정
    → Controller or RazorPage 선택
4. DI 컨테이너에서 필요한 서비스 주입
5. 응답 반환
6. Middleware 체인 역방향 응답 처리
```

---

## 🧠 실무 팁

- DI는 **서비스 클래스 구조화와 테스트용 객체 주입**에 강력함
- Middleware는 **로깅, 인증, 요청 전처리**를 구현하는 곳
- Routing은 **정적 경로 + 동적 파라미터 + 제약조건**을 적절히 활용해야 유지보수에 유리함