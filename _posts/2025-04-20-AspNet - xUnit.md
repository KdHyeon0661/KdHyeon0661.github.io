---
layout: post
title: AspNet - xUnit
date: 2025-04-10 20:20:23 +0900
category: AspNet
---
# xUnit 사용법

## 0) 왜 xUnit인가? (요약 복습)

- .NET Core/ASP.NET Core 팀이 실무에서 사용하는 **경량·확장성·비동기 친화** 프레임워크  
- MSTest/NUnit 대비 장점:  
  - 생성자 기반 **Setup**, `IDisposable`/Fixture 기반 **Teardown**  
  - **async Task** 직접 지원, 어댑터 없이 **VS/CLI** 통합

---

## 1) 프로젝트 스캐폴딩 & 솔루션 연결

```bash
# 앱 & 테스트 프로젝트 생성
dotnet new webapi -n MyApp
dotnet new xunit  -n MyApp.Tests

# 솔루션
dotnet new sln -n MyApp
dotnet sln add ./MyApp/MyApp.csproj
dotnet sln add ./MyApp.Tests/MyApp.Tests.csproj

# 테스트 프로젝트가 앱 프로젝트 참조
dotnet add ./MyApp.Tests/MyApp.Tests.csproj reference ./MyApp/MyApp.csproj
```

### 추천 폴더 구조

```
MyApp/
├── Controllers/
├── Services/
├── Data/
└── Program.cs

MyApp.Tests/
├── Unit/
│   ├── Services/
│   └── Controllers/
├── Integration/
│   ├── MinimalApi/
│   └── EfCore/
└── MyApp.Tests.csproj
```

---

## 2) 첫 테스트 — `[Fact]` / `[Theory]`

### 시스템 언더 테스트(SUT)

```csharp
public class Calculator
{
    public int Add(int a, int b) => a + b;
    public int Div(int a, int b) => a / b;
}
```

### 단일 사례: `[Fact]`

```csharp
public class CalculatorTests
{
    [Fact]
    public void Add_Should_Return_Correct_Sum()
    {
        var sut = new Calculator();
        var result = sut.Add(2, 3);
        Assert.Equal(5, result);
    }
}
```

### 데이터 기반: `[Theory]` + `InlineData`

```csharp
public class CalculatorTheoryTests
{
    [Theory]
    [InlineData(1, 2, 3)]
    [InlineData(5, 5, 10)]
    [InlineData(-2, 7, 5)]
    public void Add_Should_Work_For_Multiple_Cases(int a, int b, int expected)
    {
        var sut = new Calculator();
        Assert.Equal(expected, sut.Add(a, b));
    }
}
```

### 대용량·복잡 데이터: `MemberData` / `ClassData`

```csharp
public static class AddCases
{
    public static IEnumerable<object[]> Data =>
        new[]
        {
            new object[] { int.MaxValue, 0, int.MaxValue },
            new object[] { -100, 100, 0 }
        };
}

public class CalculatorMemberDataTests
{
    [Theory]
    [MemberData(nameof(AddCases.Data), MemberType = typeof(AddCases))]
    public void Add_MemberData(int a, int b, int expected)
    {
        new Calculator().Add(a, b).Equals(expected);
    }
}
```

`ClassData`는 `IEnumerable<object[]>` 구현 클래스를 제공하면 된다.

---

## 3) 비동기/예외/시간 제어

### 비동기: `async Task` 반환

```csharp
[Fact]
public async Task Async_Test_Sample()
{
    await Task.Delay(10);
    Assert.True(true);
}
```

### 예외 검증

```csharp
[Fact]
public void Div_ByZero_Should_Throw()
{
    var sut = new Calculator();
    Assert.Throws<DivideByZeroException>(() => sut.Div(1, 0));
}
```

### 비동기 예외

```csharp
[Fact]
public async Task Async_Exception_Should_Throw()
{
    Task Thrower() => Task.FromException(new InvalidOperationException());
    await Assert.ThrowsAsync<InvalidOperationException>(() => Thrower());
}
```

### 시간 의존 로직(안정화)

```csharp
public interface IClock { DateTimeOffset UtcNow { get; } }
public sealed class SystemClock : IClock { public DateTimeOffset UtcNow => DateTimeOffset.UtcNow; }

public sealed class TokenService(IClock clock)
{
    public string Issue() => $"{clock.UtcNow:yyyyMMddHHmmss}-{Guid.NewGuid()}";
}

[Fact]
public void Deterministic_Time_By_Abstraction()
{
    var clock = new Mock<IClock>();
    clock.SetupGet(c => c.UtcNow).Returns(new DateTimeOffset(2025,01,01,0,0,0,TimeSpan.Zero));
    var svc = new TokenService(clock.Object);

    var token = svc.Issue();
    Assert.StartsWith("20250101000000-", token);
}
```

---

## 4) xUnit 수명주기 — 생성자/`IDisposable`/Fixture

### 클래스 단위 Setup/Teardown

```csharp
public class UserServiceTests : IDisposable
{
    private readonly UserService _svc;

    public UserServiceTests()
    {
        _svc = new UserService(); // Setup (매 테스트 인스턴스마다)
    }

    public void Dispose()
    {
        // Teardown
    }

    [Fact] public void Basic() => Assert.True(_svc != null);
}
```

### **`IClassFixture<T>`** — 클래스 전체 공유 리소스

```csharp
public class SharedFixture : IDisposable
{
    public readonly HttpClient Client = new() { BaseAddress = new Uri("https://example.org") };
    public void Dispose() => Client.Dispose();
}

public class UsesFixture : IClassFixture<SharedFixture>
{
    private readonly SharedFixture _fx;
    public UsesFixture(SharedFixture fx) => _fx = fx;

    [Fact] public void Can_Use_HttpClient() => Assert.NotNull(_fx.Client);
}
```

### **Collection Fixture** — 여러 테스트 클래스 간 공유

```csharp
[CollectionDefinition("db")]
public class DbCollection : ICollectionFixture<DbFixture> { }

[Collection("db")]
public class RepoTestsA { /* ... */ }

[Collection("db")]
public class RepoTestsB { /* ... */ }
```

---

## 5) 로깅·출력 — `ITestOutputHelper`

```csharp
public class OutputTests
{
    private readonly ITestOutputHelper _out;
    public OutputTests(ITestOutputHelper output) => _out = output;

    [Fact]
    public void Log_To_Test_Console()
    {
        _out.WriteLine("message visible in test runner");
        Assert.True(true);
    }
}
```

---

## 6) ASP.NET Core — 컨트롤러/Minimal API/미들웨어 단위 테스트

### 컨트롤러(서비스 모킹)

```csharp
public interface IUserService { Task<User?> GetAsync(int id); }
public record User(int Id, string Name);

public class UsersController : ControllerBase
{
    private readonly IUserService _svc;
    public UsersController(IUserService svc) => _svc = svc;

    [HttpGet("api/users/{id}")]
    public async Task<ActionResult<User>> Get(int id)
    {
        var u = await _svc.GetAsync(id);
        return u is null ? NotFound() : Ok(u);
    }
}

public class UsersControllerTests
{
    [Fact]
    public async Task Get_Should_Return_Ok_When_Exists()
    {
        var m = new Mock<IUserService>();
        m.Setup(s => s.GetAsync(7)).ReturnsAsync(new User(7,"A"));

        var sut = new UsersController(m.Object);
        var res = await sut.Get(7);

        var ok = Assert.IsType<OkObjectResult>(res.Result);
        var user = Assert.IsType<User>(ok.Value);
        Assert.Equal(7, user.Id);
    }
}
```

### Minimal API 핸들러 함수 테스트(순수 함수)

```csharp
public static class Handlers
{
    public static IResult GetPing() => Results.Ok(new { pong = true });
}

[Fact]
public void Minimal_Handler_Pure_Test()
{
    var result = Handlers.GetPing();
    var ok = Assert.IsType<Ok<object>>(result);
    Assert.NotNull(ok.Value);
}
```

### 미들웨어 단위 테스트

```csharp
public sealed class HeaderMiddleware(RequestDelegate next)
{
    public async Task InvokeAsync(HttpContext ctx)
    {
        ctx.Response.OnStarting(() =>
        {
            ctx.Response.Headers["X-App"] = "Test";
            return Task.CompletedTask;
        });
        await next(ctx);
    }
}

[Fact]
public async Task Middleware_Should_Add_Header()
{
    var ctx = new DefaultHttpContext();
    var mw = new HeaderMiddleware(_ => Task.CompletedTask);

    await mw.InvokeAsync(ctx);

    Assert.True(ctx.Response.Headers.ContainsKey("X-App"));
}
```

---

## 7) 통합 테스트 — `TestServer` / `WebApplicationFactory<T>`

### Minimal API를 통합 테스트

**Program.cs** (예시)

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddSingleton<IUserService, InMemoryUserService>();
var app = builder.Build();
app.MapGet("/api/users/{id:int}", async (int id, IUserService svc)
    => await svc.GetAsync(id) is { } u ? Results.Ok(u) : Results.NotFound());
app.Run();

public partial class Program { } // WebApplicationFactory용
```

**통합 테스트**

```csharp
dotnet add MyApp.Tests package Microsoft.AspNetCore.Mvc.Testing
```

```csharp
public class ApiFactory : WebApplicationFactory<Program>
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        // 테스트용 DI 재정의도 가능
        builder.ConfigureServices(services =>
        {
            // services.Remove(...) + services.AddSingleton(...)
        });
    }
}

public class MinimalApiTests : IClassFixture<ApiFactory>
{
    private readonly HttpClient _client;
    public MinimalApiTests(ApiFactory f) => _client = f.CreateClient();

    [Fact]
    public async Task Get_User_Should_Return_200()
    {
        var res = await _client.GetAsync("/api/users/1");
        Assert.True(res.StatusCode is System.Net.HttpStatusCode.OK or System.Net.HttpStatusCode.NotFound);
    }
}
```

---

## 8) EF Core 테스트 — InMemory vs SQLite(InMemory)

- **InMemory Provider**: LINQ 동작이 실제 DB와 다를 수 있음(관계/제약 X)  
- **SQLite InMemory**: 실제 SQL 실행(제약/쿼리 변환 유사), **권장**

```csharp
dotnet add MyApp.Tests package Microsoft.Data.Sqlite
dotnet add MyApp.Tests package Microsoft.EntityFrameworkCore.Sqlite
```

```csharp
public static class SqliteInMemory
{
    public static (DbContextOptions<AppDb>, SqliteConnection) Create()
    {
        var conn = new SqliteConnection("DataSource=:memory:");
        conn.Open();

        var opts = new DbContextOptionsBuilder<AppDb>()
            .UseSqlite(conn)
            .Options;

        using var ctx = new AppDb(opts);
        ctx.Database.EnsureCreated();
        return (opts, conn);
    }
}

public class EfRepositoryTests : IDisposable
{
    private readonly SqliteConnection _conn;
    private readonly AppDb _db;

    public EfRepositoryTests()
    {
        var (opts, conn) = SqliteInMemory.Create();
        _conn = conn;
        _db   = new AppDb(opts);
    }

    [Fact]
    public async Task Add_And_Query()
    {
        _db.Users.Add(new UserEntity { Name = "A" });
        await _db.SaveChangesAsync();

        var n = await _db.Users.CountAsync();
        Assert.Equal(1, n);
    }

    public void Dispose()
    {
        _db.Dispose();
        _conn.Dispose();
    }
}
```

---

## 9) JSON 응답 검증 — `System.Text.Json` / `FluentAssertions`

```csharp
dotnet add MyApp.Tests package FluentAssertions
```

```csharp
[Fact]
public async Task Should_Return_ProblemDetails_On_Validation_Error()
{
    var content = JsonContent.Create(new { email = "" });
    var res = await _client.PostAsync("/api/register", content);

    res.StatusCode.Should().Be(System.Net.HttpStatusCode.BadRequest);

    var body = await res.Content.ReadFromJsonAsync<Dictionary<string, object>>();
    body.Should().ContainKey("errors");
}
```

---

## 10) Moq로 DI 의존성 제거 (요약)

```csharp
dotnet add MyApp.Tests package Moq
```

```csharp
[Fact]
public async Task Controller_Uses_Service_Exactly_Once()
{
    var m = new Mock<IUserService>(MockBehavior.Strict);
    m.Setup(s => s.GetAsync(3)).ReturnsAsync(new User(3,"X"));

    var sut = new UsersController(m.Object);
    var res = await sut.Get(3);

    Assert.IsType<OkObjectResult>(res.Result);
    m.Verify(s => s.GetAsync(3), Times.Once);
    m.VerifyNoOtherCalls();
}
```

> 상세 Moq 고급 패턴은 별도 “Moq 심화 가이드”에서 다룬 내용을 참조해 확장할 수 있다.

---

## 11) 병렬·순서·카테고리·조건부 실행

### 병렬 실행 전역 설정 (`MyApp.Tests.csproj`)

```xml
<PropertyGroup>
  <ParallelizeTestCollections>true</ParallelizeTestCollections>
  <MaxParallelThreads>0</MaxParallelThreads> <!-- 0 = logical processor count -->
</PropertyGroup>
```

### 컬렉션 단위 동기화 (DB/포트 공유)

```csharp
[CollectionDefinition("db", DisableParallelization = true)]
public class DbCollection : ICollectionFixture<DbFixture> { }
```

### 카테고리(트레이트) & 필터

```csharp
public class TraitAttribute : Attribute
{
    public string Name { get; }
    public string Value { get; }
    public TraitAttribute(string name, string value) => (Name, Value) = (name, value);
}

public class SlowTests
{
    [Fact, Trait("Category", "Slow")]
    public void Long_Running() => Assert.True(true);
}
```

CLI에서 특정 트레이트만:

```bash
dotnet test --filter "Category=Slow"
```

### 조건부 실행

```csharp
public class OnlyWindows
{
    [Fact(Skip = "Enable only on Windows")]
    public void Sample() { }
}

public class Conditional
{
    [Fact]
    public void Run_Only_If_ENV_SET()
    {
        if (Environment.GetEnvironmentVariable("RUN_TESTS") != "1")
            return; // or Skip via Assert
        Assert.True(true);
    }
}
```

---

## 12) 커버리지 — Coverlet & 리포트

```bash
dotnet add MyApp.Tests package coverlet.collector
dotnet test /p:CollectCoverage=true /p:ExcludeByFile="**/Program.cs"
```

ReportGenerator(옵션):

```bash
dotnet tool install -g dotnet-reportgenerator-globaltool
dotnet test /p:CollectCoverage=true /p:CoverletOutputFormat=cobertura
reportgenerator -reports:**/coverage.cobertura.xml -targetdir:coveragereport
```

---

## 13) GitHub Actions로 CI 구성 (샘플)

```yaml
name: ci
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.0.x'
      - run: dotnet restore
      - run: dotnet build --no-restore -c Release
      - run: dotnet test --no-build -c Release /p:CollectCoverage=true
```

---

## 14) 실전 팁 (정확성·유지보수·속도)

- **AAA 패턴**(Arrange-Act-Assert)을 지켜 가독성 확보  
- 테스트 이름: `메서드_상태_기대결과` (예: `Get_Returns404_WhenNotFound`)  
- **한 테스트는 하나의 결과**에 집중(단, 복합 결과는 합리적 그룹화)  
- 외부 I/O(파일/HTTP/DB)는 **격리**하고 **속도** 유지(초 단위 내)  
- **Flaky 방지**: 시간/랜덤/환경 추상화, 네트워크 의존성 제거  
- **`VerifyNoOtherCalls`**로 의도치 않은 상호작용 제어  
- **ProblemDetails/Validation**도 명시 검증  
- 통합 테스트에서도 **테스트용 DI 재정의**(페이크/스텁)로 안정화

---

## 15) 확장 체크리스트

- [ ] `[Fact]`/`[Theory]`/`MemberData`/`ClassData` 사용
- [ ] 비동기/예외 모두 커버
- [ ] Fixture로 DB/서버 공유 + 컬렉션 병렬 제어
- [ ] Minimal API/미들웨어/컨트롤러 단위·통합 테스트 구분
- [ ] WebApplicationFactory로 통합 테스트 표준화
- [ ] EF는 **SQLite InMemory** 우선
- [ ] JSON 응답은 **구조** 중심 검증
- [ ] 커버리지 수집/리포트/CI 연동
- [ ] 느린 테스트는 **Trait**로 분리·필터링
- [ ] 시간/랜덤 추상화로 **결정적** 테스트 보장

---

## 부록 A) FluentAssertions 빠른 예시

```csharp
var list = new[] {1,2,3};
list.Should().ContainInOrder(1,2,3).And.HaveCount(3);

await FluentActions
    .Invoking(async () => await Thrower())
    .Should().ThrowAsync<InvalidOperationException>();
```

---

## 부록 B) ModelState/검증 테스트(컨트롤러)

모델:

```csharp
public record CreateUserRequest([Required] string Email, [MinLength(3)] string Name);
```

액션:

```csharp
[HttpPost("api/users")]
public IActionResult Create([FromBody] CreateUserRequest req)
{
    if (!ModelState.IsValid) return ValidationProblem(ModelState);
    return Created($"/api/users/{req.Email}", req);
}
```

테스트(통합):

```csharp
[Fact]
public async Task Create_Should_Return_Validation_Problem()
{
    var res = await _client.PostAsJsonAsync("/api/users", new { Email = "", Name = "AB" });
    res.StatusCode.Should().Be(System.Net.HttpStatusCode.BadRequest);

    var doc = await res.Content.ReadFromJsonAsync<ValidationProblemDetails>();
    doc!.Errors.Should().ContainKey("Email");
}
```

---

## 요약

| 항목 | 핵심 |
|---|---|
| 단위 테스트 | `[Fact]`, `[Theory]` (+ `MemberData`/`ClassData`) |
| 비동기/예외 | `async Task`, `Throws/ThrowsAsync` |
| 수명주기 | 생성자/`IDisposable` + `IClassFixture`/Collection Fixture |
| ASP.NET Core | 컨트롤러/Minimal/미들웨어 단·통합 테스트 패턴 |
| 통합 인프라 | `WebApplicationFactory` + TestServer |
| 데이터 계층 | **SQLite InMemory** 권장, InMemory Provider 주의 |
| 모킹 | Moq로 DI 의존성 격리, `VerifyNoOtherCalls` |
| JSON/검증 | ProblemDetails/Validation 명시 검증 |
| 병렬/필터 | 컬렉션 병렬 제어, Trait/Filter 운영 |
| 커버리지/CI | Coverlet + ReportGenerator, GitHub Actions 연동 |