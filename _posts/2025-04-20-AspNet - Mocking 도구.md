---
layout: post
title: AspNet - Mocking 도구
date: 2025-04-10 22:20:23 +0900
category: AspNet
---
# Mocking 도구 완전 정리

## 0. 테스트 더블 용어 정리(빠른 리마인드)

| 용어 | 핵심 | Moq에서 보통 |
|---|---|---|
| Dummy | 의미 없는 채움값 | `Guid.Empty`, 기본값 |
| **Stub** | 반환만 정의 | `Setup(...).Returns(...)` |
| **Mock** | Stub + 호출 검증 | `Verify(..., Times.Once)` |
| Fake | 간이 구현체(메모리 DB 등) | 직접 클래스 작성 |
| Spy | 호출 기록 저장 | `Callback(...)`로 일부 대체 |

> 실무에서는 Stub + Mock 조합이 가장 흔하다.

---

## 1. 프로젝트 준비

```bash
dotnet new xunit -n Demo.Tests
cd Demo.Tests
dotnet add package Moq
dotnet add package FluentAssertions
```

테스트 대상 도메인(샘플):

```csharp
public interface IUserService
{
    Task<User?> GetAsync(int id, CancellationToken ct = default);
    Task<bool> CreateAsync(UserCreate dto, CancellationToken ct = default);
    IAsyncEnumerable<User> StreamAllAsync(CancellationToken ct = default);
}

public record User(int Id, string Name, string Email);
public record UserCreate(string Name, string Email);

public sealed class UserController
{
    private readonly IUserService _svc;
    public UserController(IUserService svc) => _svc = svc;

    public async Task<IResult> Get(int id, CancellationToken ct = default)
    {
        var u = await _svc.GetAsync(id, ct);
        return u is null ? Results.NotFound() : Results.Ok(u);
    }

    public async Task<IResult> Post(UserCreate dto, CancellationToken ct = default)
    {
        var ok = await _svc.CreateAsync(dto, ct);
        return ok ? Results.Created($"/users/{dto.Email}", dto) : Results.BadRequest();
    }

    public IAsyncEnumerable<User> Stream(CancellationToken ct = default)
        => _svc.StreamAllAsync(ct);
}
```

---

## 2. Moq 기본 — `Setup`, `Returns`, `Verify`

```csharp
public class UserControllerTests
{
    [Fact]
    public async Task Get_Should_Return_Ok_When_User_Exists()
    {
        var mock = new Mock<IUserService>();
        mock.Setup(s => s.GetAsync(1, It.IsAny<CancellationToken>()))
            .ReturnsAsync(new User(1, "Alice", "a@x.io"));

        var sut = new UserController(mock.Object);

        var result = await sut.Get(1);

        result.Should().BeOfType<Ok<User>>()
              .Which.Value.Name.Should().Be("Alice");

        mock.Verify(s => s.GetAsync(1, It.IsAny<CancellationToken>()), Times.Once);
        mock.VerifyNoOtherCalls();
    }
}
```

핵심 포인트
- `It.IsAny<T>()` 남용은 테스트 신뢰도 저하 → **가능하면 조건 명시**.
- 마지막에 `VerifyNoOtherCalls()`로 **의도치 않은 상호작용 차단**.

---

## 3. Strict vs Loose, DefaultValue

```csharp
// 기본: Loose(정의되지 않은 호출은 default 반환)
var loose = new Mock<IUserService>(MockBehavior.Loose);

// Strict: 정의 안 한 호출 → 예외
var strict = new Mock<IUserService>(MockBehavior.Strict);

strict.Setup(s => s.GetAsync(It.IsAny<int>(), It.IsAny<CancellationToken>()))
      .ReturnsAsync((int id, CancellationToken _) => new User(id, "X", "x@y.z"));
```

- **Strict**는 회귀 방지에 강력하지만 유지보수 비용↑. 레거시·핵심 서비스에 국소 적용 추천.
- `DefaultValueProvider`/`DefaultValue.Mock`로 **깊은 Stub**도 가능(복잡 계층 테스트를 빠르게 시동).

```csharp
var deep = new Mock<IDepA> { DefaultValue = DefaultValue.Mock };
// deep.Object.B.B2.C까지 자동 Mock 생성
```

---

## 4. 파라미터 매칭 — `It.Is`, `It.IsIn`, `It.IsRegex`

```csharp
mock.Setup(s => s.CreateAsync(
        It.Is<UserCreate>(x => x.Email.EndsWith("@company.com")),
        It.IsAny<CancellationToken>()))
    .ReturnsAsync(true);

mock.Setup(s => s.GetAsync(It.IsIn(1,2,3), It.IsAny<CancellationToken>()))
    .ReturnsAsync((int id, CancellationToken _) => new User(id, "P", "p@x.io));
```

문자열 패턴:

```csharp
mock.Setup(s => s.CreateAsync(
        It.Is<UserCreate>(x => System.Text.RegularExpressions.Regex.IsMatch(x.Email, @"^\S+@\S+$")),
        It.IsAny<CancellationToken>()))
    .ReturnsAsync(true);
```

---

## 5. 예외/콜백/상태 캡처 — `Throws`, `Callback`, `Returns`

```csharp
[Fact]
public async Task Post_Should_Return_BadRequest_On_Service_Exception()
{
    var mock = new Mock<IUserService>();
    mock.Setup(s => s.CreateAsync(It.IsAny<UserCreate>(), It.IsAny<CancellationToken>()))
        .ThrowsAsync(new InvalidOperationException("dup"));

    var sut = new UserController(mock.Object);

    var res = await sut.Post(new("A", "a@x.io"));
    res.Should().BeOfType<BadRequest>();
}
```

콜백으로 기록/검증:

```csharp
[Fact]
public async Task Post_Should_Pass_Exact_Payload()
{
    var captured = new List<UserCreate>();
    var mock = new Mock<IUserService>();
    mock.Setup(s => s.CreateAsync(It.IsAny<UserCreate>(), It.IsAny<CancellationToken>()))
        .Callback<UserCreate, CancellationToken>((dto, _) => captured.Add(dto))
        .ReturnsAsync(true);

    var sut = new UserController(mock.Object);
    await sut.Post(new("B", "b@x.io"));

    captured.Should().ContainSingle(x => x.Name == "B" && x.Email == "b@x.io");
}
```

`SetupSequence`로 순차 동작:

```csharp
mock.SetupSequence(s => s.GetAsync(10, It.IsAny<CancellationToken>()))
    .ReturnsAsync((User?)null)
    .ReturnsAsync(new User(10, "Retried", "r@x.io"))
    .ThrowsAsync(new TimeoutException());
```

---

## 6. 속성/인덱서/이벤트/`ref/out`/비가상 멤버

- 속성:

```csharp
var m = new Mock<IConfig>();
m.SetupGet(c => c.RetryCount).Returns(3);
m.SetupProperty(c => c.Enabled, true); // set/get 추적
```

- 이벤트:

```csharp
var m = new Mock<IWatcher>();
EventHandler? captured;
m.SetupAdd(w => w.Changed += It.IsAny<EventHandler>())
 .Callback<EventHandler>(h => captured = h);
```

- `out`/`ref`:

```csharp
public interface IParser { bool TryParse(string s, out int value); }

var mp = new Mock<IParser>();
mp.Setup(p => p.TryParse("42", out It.Ref<int>.IsAny))
  .Callback(new TryParseCallback((string s, out int v) => v = 42))
  .Returns(true);

delegate void TryParseCallback(string s, out int value);
```

- 주의: **Moq는 비가상/정적/비인터페이스 멤버**를 직접 Mock 못한다.
  → 설계 개선(인터페이스/virtual), 혹은 JustMock/TypeMock/Mocklis/Proxy 기반 대안 고려.

---

## 7. 비동기 & 스트리밍 — `Task`, `IAsyncEnumerable<T>`

```csharp
[Fact]
public async Task Stream_Should_Yield_Items()
{
    var mock = new Mock<IUserService>();
    mock.Setup(s => s.StreamAllAsync(It.IsAny<CancellationToken>()))
        .Returns(async (CancellationToken ct) =>
        {
            async IAsyncEnumerable<User> Impl([System.Runtime.CompilerServices.EnumeratorCancellation] CancellationToken token)
            {
                yield return new User(1, "A", "a@x.io");
                await Task.Delay(1, token);
                yield return new User(2, "B", "b@x.io");
            }
            return Impl(ct);
        });

    var sut = new UserController(mock.Object);
    var list = new List<User>();
    await foreach (var u in sut.Stream())
        list.Add(u);

    list.Should().HaveCount(2);
}
```

- `EnumeratorCancellation` 어트리뷰트로 CT 전달.
- 실무에선 대량 스트림 대신 **소형 샘플**로 로직 검증.

---

## 8. 상호작용 검증 — `Verify`, `Times`, `InSequence`, 호출 순서

```csharp
[Fact]
public async Task Get_Should_Call_Service_Once_Then_NoMore()
{
    var mock = new Mock<IUserService>();
    mock.Setup(s => s.GetAsync(5, It.IsAny<CancellationToken>()))
        .ReturnsAsync(new User(5, "E", "e@x"));

    var sut = new UserController(mock.Object);
    await sut.Get(5);

    mock.Verify(s => s.GetAsync(5, It.IsAny<CancellationToken>()), Times.Once);
    mock.VerifyNoOtherCalls();
}
```

순서 제약은 Moq 내장 지원이 제한적 → **Moq.Sequences**(외부) 또는 캡처/플래그로 우회:

```csharp
var order = new List<string>();
mock.Setup(s => s.GetAsync(It.IsAny<int>(), It.IsAny<CancellationToken>()))
    .Callback(() => order.Add("Get"))
    .ReturnsAsync(new User(1,"X","x@x"));
mock.Setup(s => s.CreateAsync(It.IsAny<UserCreate>(), It.IsAny<CancellationToken>()))
    .Callback(() => order.Add("Create"))
    .ReturnsAsync(true);

// 실행 후:
order.Should().ContainInOrder("Get", "Create");
```

---

## 9. DI/조합 — `As<T>`, 다중 인터페이스, AutoFixture

여러 인터페이스를 한 Mock로:

```csharp
var m = new Mock<IPrimary>();
m.As<ISecondary>().Setup(x => x.Ping()).Returns(true);
var composite = m.Object; // IPrimary & ISecondary 동시
```

AutoFixture로 생성 편의:

```csharp
dotnet add package AutoFixture
dotnet add package AutoFixture.AutoMoq
```

```csharp
var fixture = new Fixture().Customize(new AutoMoqCustomization { ConfigureMembers = true });

var mock = fixture.Freeze<Mock<IUserService>>();
mock.Setup(s => s.GetAsync(It.IsAny<int>(), It.IsAny<CancellationToken>()))
    .ReturnsAsync(new User(9,"Auto","auto@x"));

var sut = fixture.Create<UserController>();
var res = await sut.Get(9);
res.Should().BeOfType<Ok<User>>();
```

---

## 10. 시간/랜덤/환경 추상화 — 테스트 안정성

**안정적 테스트**를 위해 `DateTime.UtcNow`, `Guid.NewGuid`, `Random` 등을 직접 호출하지 않고 **추상화**:

```csharp
public interface IClock { DateTimeOffset UtcNow { get; } }
public sealed class SysClock : IClock { public DateTimeOffset UtcNow => DateTimeOffset.UtcNow; }

public sealed class TokenService(IClock clock)
{
    public string Issue() => $"{clock.UtcNow:yyyyMMddHHmmss}-{Guid.NewGuid()}";
}
```

테스트:

```csharp
var clock = new Mock<IClock>();
clock.SetupGet(c => c.UtcNow).Returns(new DateTimeOffset(2025,1,1,0,0,0,TimeSpan.Zero));

var svc = new TokenService(clock.Object);
var token = svc.Issue();
token.Should().StartWith("20250101000000-");
```

---

## 11. 웹 계층 실전 — 컨트롤러/미들웨어/필터

컨트롤러에서 서비스 Mock 주입은 위 예시 참고.
미들웨어 단위 테스트는 **`DefaultHttpContext`**와 파이프라인 델리게이트를 조립:

```csharp
[Fact]
public async Task Middleware_Should_Append_Header()
{
    var ctx = new DefaultHttpContext();
    var next = new RequestDelegate(_ => Task.CompletedTask);

    var mw = new CustomHeaderMiddleware(next);
    await mw.InvokeAsync(ctx);

    ctx.Response.Headers.Should().ContainKey("X-Powered-By");
}
```

필터/바인더는 **Moq로 `ActionContext`/`HttpContext`/`ModelState`**를 스텁.

---

## 12. EF Core, HttpClient, 외부 API

- EF Core는 **Mock보다 InMemory/Sqlite**가 더 적합(쿼리 변환 차이 최소화 위해 **Sqlite InMemory** 권장).
- `HttpClient`는 `HttpMessageHandler`를 Mock:

```csharp
public class StubHandler(HttpResponseMessage res) : HttpMessageHandler
{
    protected override Task<HttpResponseMessage> SendAsync(HttpRequestMessage request, CancellationToken ct)
        => Task.FromResult(res);
}

[Fact]
public async Task Client_Should_Parse_Json()
{
    var res = new HttpResponseMessage(System.Net.HttpStatusCode.OK)
    {
        Content = new StringContent("""{"id":1,"name":"A"}""", System.Text.Encoding.UTF8, "application/json")
    };
    var http = new HttpClient(new StubHandler(res)) { BaseAddress = new Uri("https://api.local") };

    // IExternalApi(client) 등 주입받아 테스트
}
```

---

## 13. 레거시/난이도 높은 영역

- **정적 메서드/비가상/Sealed**: Moq 한계. 리팩터(래퍼 인터페이스) 또는 **JustMock(상용)/TypeMock** 검토.
- **Thread/Timer/Background**: `IHostedService`/`PeriodicTimer`를 추상화하고 CT 활용.
- **파일/프로세스/네트워크**: `IFileSystem`(System.IO.Abstractions) 같은 추상화 계층 사용.

---

## 14. 안티패턴 & 베스트 프랙티스

안티패턴
- `It.IsAny<T>()` 범람 → 테스트 약화.
- `Verify` 남발(구현 상세에 결박) → **관찰 가능한 결과** 위주 검증.
- 과도한 Strict → 유지비용 폭증.

베스트
- **작은 단위**로 Mock(1~2 의존성), Given-When-Then 구조 명확화.
- Side effect는 `Callback`으로 제한적 검증, Return 값 기반 결과 위주.
- `VerifyNoOtherCalls()`로 **의도치 않은 상호작용** 방지.
- Fixture/Builder 패턴으로 **테스트 데이터 중복 제거**.

---

## 15. NSubstitute/FakeItEasy 비교(요약)

| 항목 | **Moq** | **NSubstitute** | **FakeItEasy** |
|---|---|---|---|
| 기본 문법 | `mock.Setup(...).Returns(...)` | `sub.Some().Returns(...)` | `A.CallTo(...).Returns(...)` |
| 진입 장벽 | 낮음 | 매우 낮음 | 낮음 |
| BDD 친화 | 보통 | 높음 | 높음 |
| 호출 검증 | `Verify(..., Times.Once)` | `Received(1)` | `MustHaveHappenedOnceExactly()` |
| 학습 자료/커뮤니티 | 매우 풍부 | 풍부 | 보통 |
| 정적/비가상 | 불가(같음) | 불가 | 불가 |

예시(동일 시나리오):

```csharp
// NSubstitute
var s = Substitute.For<IUserService>();
s.GetAsync(1, default).Returns(new User(1,"A","a@x"));
await s.Received(1).GetAsync(1, default);

// FakeItEasy
var f = A.Fake<IUserService>();
A.CallTo(() => f.GetAsync(1, A<CancellationToken>._))
 .Returns(new User(1,"A","a@x"));
A.CallTo(() => f.GetAsync(1, A<CancellationToken>._))
 .MustHaveHappenedOnceExactly();
```

선택 가이드
- 빠른 온보딩·자료 풍부: **Moq**.
- 문법 간결/Bdd 스타일: **NSubstitute**.
- BDD/명료한 API: **FakeItEasy**.

---

## 16. 종합 실전 예제 — 실패 재시도·로그·순서·예외

```csharp
public interface IRetryingService
{
    Task<User?> GetWithRetryAsync(int id, int maxRetry, CancellationToken ct = default);
}

public sealed class RetryingService(IUserService inner, ILogger<RetryingService> log) : IRetryingService
{
    public async Task<User?> GetWithRetryAsync(int id, int maxRetry, CancellationToken ct = default)
    {
        for (var i = 0; i <= maxRetry; i++)
        {
            try
            {
                var u = await inner.GetAsync(id, ct);
                if (u is not null) return u;
            }
            catch (TimeoutException ex) when (i < maxRetry)
            {
                log.LogWarning(ex, "timeout, retry {Attempt}", i + 1);
                await Task.Delay(10, ct);
            }
        }
        return null;
    }
}

public class RetryingServiceTests
{
    [Fact]
    public async Task Should_Retry_On_Timeout_And_Succeed()
    {
        var svc = new Mock<IUserService>(MockBehavior.Strict);
        svc.SetupSequence(s => s.GetAsync(7, It.IsAny<CancellationToken>()))
           .ThrowsAsync(new TimeoutException())
           .ReturnsAsync(new User(7,"OK","ok@x"));

        var logger = new Mock<ILogger<RetryingService>>();
        var sut = new RetryingService(svc.Object, logger.Object);

        var u = await sut.GetWithRetryAsync(7, maxRetry: 2);

        u.Should().NotBeNull().And.Subject.As<User>().Id.Should().Be(7);

        svc.Verify(s => s.GetAsync(7, It.IsAny<CancellationToken>()), Times.Exactly(2));
        logger.Verify(l => l.Log(
            LogLevel.Warning,
            It.IsAny<EventId>(),
            It.Is<It.IsAnyType>((state, _) => state.ToString()!.Contains("timeout, retry")),
            It.IsAny<Exception>(),
            It.IsAny<Func<It.IsAnyType, Exception?, string>>()), Times.Once);
        svc.VerifyNoOtherCalls();
    }
}
```

포인트
- `SetupSequence`로 실패→성공 시나리오.
- 로거 검증은 템플릿 문자열 포함 여부로 최소화.
- `Strict`로 의도 외 호출 방지.

---

## 17. 체크리스트

- [ ] 인터페이스/virtual로 **테스트 가능 설계** 만들기
- [ ] 입력 매칭은 구체적(값/조건), `It.IsAny` 최소화
- [ ] `VerifyNoOtherCalls`로 부수 호출 차단
- [ ] 시간/랜덤/환경 추상화(`IClock` 등)
- [ ] EF는 Mock보다 **Sqlite InMemory** 권장
- [ ] 외부 HTTP는 `HttpMessageHandler` Stub
- [ ] 전역 상태/정적 의존성 피하기(불가 시 래퍼)
- [ ] 실패/예외/경계값/순서 테스트 포함
- [ ] 테스트는 **속도·독립성**이 생명(대규모 테스트셋의 90%는 단위 테스트로)

---

## 18. 요약

| 주제 | 핵심 |
|---|---|
| 목적 | 외부 의존성 격리, 빠르고 결정적 테스트 |
| Moq 핵심 | `Setup/Returns/Throws/Verify/Sequence/Callback/VerifyNoOtherCalls` |
| 고급 | `IAsyncEnumerable`, `ref/out`, `Strict`, `DefaultValue.Mock`, 다중 인터페이스 |
| 설계 | 시간/랜덤/환경 추상화, 테스트 가능 구조 |
| 대안 | NSubstitute(간결), FakeItEasy(BDD), 정적/비가상은 리팩터 or 상용 |
| 실무 팁 | 구체 매칭, 결과 중심 검증, 부수효과 최소화, 속도 유지 |
