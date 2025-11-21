---
layout: post
title: AspNet - NET Core 성능 튜닝 및 Caching
date: 2025-05-03 19:20:23 +0900
category: AspNet
---
# ASP.NET Core 성능 튜닝 & Caching 전략

## 성능 최적화의 전체 지도

- **우선순위**: I/O(데이터베이스/네트워크) → 직렬화 → 렌더링 → 메모리/GC → 동시성/락
- **원칙**: *측정 없는 튜닝은 위험하다.* 항상 **가설 → 측정 → 변경 → 재측정**.
- **핵심 레버**: (1) **캐싱**(가장 큰 지렛대) (2) **비동기화** (3) **데이터 접근 축소/정규화** (4) **전송량 절감** (5) **할당/GC 감소**

---

## 성능 측정/관찰 — “보이는 만큼 빠르게”

### 프로파일/진단 툴 셋

| 목적 | 도구 | 핵심 지표 |
|---|---|---|
| 런타임 카운터 | `dotnet-counters` | GC(Gen0/1/2), LOH, 스레드풀 QLen |
| 트레이싱 | `dotnet-trace`, `PerfView`, `dotTrace` | Hot Path, 할당, 예외 |
| 코드 단위 벤치 | `BenchmarkDotNet` | 평균/분산, 할당/GC |
| 요청 시간 | `MiniProfiler` | 쿼리 시간, View 렌더링 |

### 최소 관찰 예시 (개발 중)

```bash
dotnet-counters monitor --process-id <PID> System.Runtime Microsoft.AspNetCore.Hosting
```
- **Budget 규칙**: P95 요청 < 200ms, Gen2 빈도 낮음, % Time in GC < 5~10%
- **첫 조치**: P95가 높은 엔드포인트부터 쿼리/직렬화/외부호출을 분해 측정.

---

## 데이터베이스 레이어 튜닝(가장 큰 지렛대)

### EF Core 실전 규칙

- **쿼리 투명화**: `AsNoTracking()` 기본(읽기성), 필요한 곳만 Tracking.
- **투사**: `Select`로 필요한 필드만 직렬화 대상에 포함.
- **N+1 제거**: `Include` 혹은 **명시적** 다중 쿼리 + 조인/키 셀렉션.
- **Compiled Query**: 반복되는 핫 쿼리에 사용.
- **DbContext Pool**: 생성 비용 절감.
- **Batching**: 저장 시 `SaveChanges` 빈도 조절(단, 트랜잭션·락 주의).

```csharp
// 등록: DbContext Pool
builder.Services.AddDbContextPool<AppDbContext>(opt =>
    opt.UseNpgsql(connStr)
       .UseQueryTrackingBehavior(QueryTrackingBehavior.NoTracking));

// Projection + Compiled query
public static class Compiled
{
    public static readonly Func<AppDbContext,int,Task<ProductDto?>> ProductById =
        EF.CompileAsyncQuery((AppDbContext db, int id) =>
            db.Products.Where(p => p.Id == id)
              .Select(p => new ProductDto(p.Id, p.Name, p.Price))
              .FirstOrDefault());
}

// 사용
var dto = await Compiled.ProductById(db, id);
```

### 인덱스 & 실행계획

- **커버링 인덱스**(필요 컬럼 포함), **선행 컬럼 선택성**(Cardinality) 확인.
- 파라미터 스니핑 이슈 → **옵션 리컴파일**(DB 별) 또는 **균질 파라미터**.

---

## 비동기와 동시성 — I/O 지연을 숨긴다

- **모든 I/O에 `async/await` 철저**: DB, 파일, HTTP, Redis.
- **스레드풀 고갈 방지**: 동기 블로킹 금지(`.Result`, `.Wait()` 절대 금지).
- ASP.NET Core엔 `SynchronizationContext`가 없어도, **async 전파**는 필수.

```csharp
// 잘못된 예: 블로킹
var data = httpClient.GetAsync(url).Result; // 교착/스레드풀 고갈 위험

// 올바른 예
var data = await httpClient.GetStringAsync(url);
```

---

## 전송량 최적화 — 직렬화·압축·프로토콜

### System.Text.Json 최적화

- **Source Generator**로 리플렉션 비용 절감
- 네이밍, NullIgnore, 숫자·날짜 포맷 일관

```csharp
// Source-gen 설정
[JsonSerializable(typeof(ProductDto[]))]
[JsonSourceGenerationOptions(PropertyNamingPolicy = JsonKnownNamingPolicy.CamelCase)]
public partial class JsonCtx : JsonSerializerContext { }

// 사용
return Results.Json(products, JsonCtx.Default.ProductDtoArray);
```

### 압축/HTTP2/3

```csharp
builder.Services.AddResponseCompression(o =>
{
    o.EnableForHttps = true;
    o.Providers.Add<BrotliCompressionProvider>();
    o.Providers.Add<GzipCompressionProvider>();
});

app.UseResponseCompression();
// Kestrel HTTP/2/3는 기본 제공(환경/프록시와 TLS 세팅 확인)
```

---

## 메모리/GC — “할당을 줄이고, 재사용한다”

### 풀링(객체/버퍼)

```csharp
// StringBuilderPool 예시
var provider = new DefaultObjectPoolProvider();
var pool = provider.CreateStringBuilderPool(1024);

var sb = pool.Get();
try
{
    sb.Append("heavy concat ...");
    return sb.ToString();
}
finally { pool.Return(sb); }
```

- `ArrayPool<T>.Shared`로 큰 버퍼 재사용 → LOH(85KB+) 진입 최소화.
- **Server GC** 사용(기본) + 컨테이너 환경이라면 `COMPlus_GCHeapCount` 조정 고려.

### 할당 줄이기

- `Span<T>/Memory<T>` 사용(파싱/인코딩)
- LINQ 체이닝에서 박싱/할당 많은 지점은 **for/foreach**로 전환

---

## Razor/View 렌더링

- **Tag Helper**/Partial의 반복 사용 시 **ViewComponent + 캐시** 고려.
- ViewData/TempData 남용 금지(박싱/박탈).
- **프리컴파일**(Razor SDK 기본) + 단순화된 모델.

---

## 캐싱 총론 — 전략/유효시간/무효화

### 캐시 적합성 점검

- **Deterministic?** 입력 동일→출력 동일
- **변경 빈도/비용** vs **조회 빈도/가치**
- **사용자 종속성**(개인화) → 키를 분리(사용자/언어/권한)

### 캐시 적중률/지연 절감 (기본 수식)

$$
\text{평균 지연} \approx H \cdot L_c + (1-H) \cdot L_o
$$
- \(H\): 캐시 히트율, \(L_c\): 캐시 비용, \(L_o\): 원본 비용
- 목표: **H↑**, **L_c↓**, **Invalidation(일관성)** 관리

---

## In-Memory Cache — 빠르고, 인스턴스 한정

### 기본 패턴

```csharp
builder.Services.AddMemoryCache();

public class CodeTableService
{
    private readonly IMemoryCache _cache;
    private readonly AppDbContext _db;

    public CodeTableService(IMemoryCache cache, AppDbContext db)
    { _cache = cache; _db = db; }

    public async Task<Dictionary<string,string>> GetAsync(CancellationToken ct)
    {
        return await _cache.GetOrCreateAsync("codes:v1", async entry =>
        {
            entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10);
            entry.SlidingExpiration = TimeSpan.FromMinutes(3);
            entry.SetPriority(CacheItemPriority.High);
            entry.SetSize(1); // MemoryCacheOptions.SizeLimit 사용 시 필수

            var rows = await _db.Codes.AsNoTracking().ToListAsync(ct);
            return rows.ToDictionary(x => x.Key, x => x.Value);
        });
    }
}
```

### — Single Flight

```csharp
// 키별 단일 진입 보호
private static readonly ConcurrentDictionary<string, SemaphoreSlim> _locks = new();

public async Task<T> GetOrLoadAsync<T>(string key, Func<Task<T>> loader, TimeSpan ttl)
{
    if (_cache.TryGetValue<T>(key, out var cached)) return cached;

    var gate = _locks.GetOrAdd(key, _ => new SemaphoreSlim(1,1));
    await gate.WaitAsync();
    try
    {
        if (_cache.TryGetValue<T>(key, out cached)) return cached; // 더블체크
        var value = await loader();
        _cache.Set(key, value, ttl);
        return value;
    }
    finally
    {
        gate.Release();
        _locks.TryRemove(key, out _);
    }
}
```

### 변경 알림 기반 무효화(IChangeToken)

```csharp
public class FeatureFlagProvider
{
    private readonly CancellationTokenSource _cts = new();
    public IChangeToken Token => new CancellationChangeToken(_cts.Token);
    public void Reload() => _cts.Cancel(); // 모든 캐시 엔트리가 무효화됨
}

// 캐시에 토큰 결합
entry.AddExpirationToken(_featureFlags.Token);
```

---

## Distributed Cache — Redis/SQL, 다중 인스턴스용

### Redis 구성 & 사용

```csharp
builder.Services.AddStackExchangeRedisCache(opt =>
    opt.Configuration = "localhost:6379");

public class DistributedProfileCache
{
    private readonly IDistributedCache _cache;
    private readonly AppDbContext _db;

    public DistributedProfileCache(IDistributedCache cache, AppDbContext db)
    { _cache = cache; _db = db; }

    public async Task<UserProfileDto?> GetAsync(int userId, CancellationToken ct)
    {
        var key = $"profile:v2:{userId}";
        var json = await _cache.GetStringAsync(key, ct);
        if (json is not null) return JsonSerializer.Deserialize<UserProfileDto>(json);

        var entity = await _db.Users.AsNoTracking()
            .Where(u => u.Id == userId)
            .Select(u => new UserProfileDto(u.Id, u.Name, u.Email))
            .FirstOrDefaultAsync(ct);

        if (entity is null) return null;

        json = JsonSerializer.Serialize(entity);
        await _cache.SetStringAsync(key, json, new DistributedCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(30)
        }, ct);

        return entity;
    }
}
```

### 캐시 무효화 전략

- **Cache-Aside**(권장): 읽기 시 캐시 → 미스면 원본 조회 후 캐시, **쓰기 시 원본 변경 후 캐시 제거**.
- **Write-Through**: 쓰기 시 캐시/원본 동시.
- **Write-Behind**: 캐시에 먼저 쓰고 비동기로 원본 반영(일관성 주의).

```csharp
public async Task UpdateProfileAsync(UserProfileDto dto, CancellationToken ct)
{
    // 1) 원본 먼저
    var user = await _db.Users.FindAsync(new object?[]{dto.Id}, ct);
    if (user is null) return;
    user.Name = dto.Name; user.Email = dto.Email;
    await _db.SaveChangesAsync(ct);

    // 2) 캐시 무효화
    await _cache.RemoveAsync($"profile:v2:{dto.Id}", ct);
}
```

---

## Response Caching — 헤더 기반(프록시/CDN 포함)

```csharp
builder.Services.AddResponseCaching();
app.UseResponseCaching();

[ResponseCache(Duration = 60, Location = ResponseCacheLocation.Any, VaryByQueryKeys = new[] { "lang" })]
public IActionResult GetArticle([FromQuery]string lang = "ko")
{
    // 60초 동안 브라우저/프록시/CDN에서 캐시 가능
    return Ok(GetArticleContent(lang));
}
```

- **Vary**: 쿼리/헤더(쿠키는 주의) 별로 분리.
- CDN 앞단에 **ETag/Last-Modified**(조건부 요청)도 병행하면 트래픽 절감.

---

## — 라우트/정책 기반 서버 캐시

### 기본 사용

```csharp
builder.Services.AddOutputCache(options =>
{
    options.AddPolicy("any-60", p => p.Expire(TimeSpan.FromSeconds(60)));
});
app.UseOutputCache();

app.MapGet("/news", async () => await LoadNewsAsync())
   .CacheOutput("any-60");
```

### 사용자/권한 분기

```csharp
options.AddPolicy("by-user", p => p
    .SetVaryByRouteValue("id")
    .SetVaryByHeader("Authorization")
    .Expire(TimeSpan.FromMinutes(2))
);
```

- **OutputCache**는 서버 내부 메모리 캐시이며, 분산 환경에선 **분산 저장소 연결** 또는 **인스턴스간 균일 정책** 필요.

---

## Static Files/ETag/브라우저 캐시

```csharp
app.UseStaticFiles(new StaticFileOptions
{
    OnPrepareResponse = ctx =>
    {
        var headers = ctx.Context.Response.GetTypedHeaders();
        headers.CacheControl = new Microsoft.Net.Http.Headers.CacheControlHeaderValue
        {
            Public = true, MaxAge = TimeSpan.FromDays(30)
        };
        // ETag/Last-Modified는 ASP.NET Core가 자동 처리(파일 해시 기반)
    }
});
```

- 빌드 파이프라인에서 **파일명 Fingerprint**(e.g., `site.abc123.css`) 사용 시 캐시 무효화가 쉬워짐.

---

## 캐시 키/버전/세분화 설계

```csharp
string Key(string prefix, params (string name,string value)[] dims)
{
    // 예: product:v3:id=123|culture=ko
    var sb = new StringBuilder(prefix);
    sb.Append(":v3:");
    for (int i=0; i<dims.Length; i++)
    {
        if (i>0) sb.Append('|');
        sb.Append(dims[i].name).Append('=').Append(dims[i].value);
    }
    return sb.ToString();
}
```

- **Version suffix**(vN)로 스키마/포맷 변경 시 손쉬운 대체.
- **사용자/권한/언어** 차원 분리(키 폭증 주의 → 필요 시 상위 레벨만 분리).

---

## “따끈한 캐시” 유지(백그라운드 리프레시)

- 만료 직전 **백그라운드에서 미리 갱신**해 첫 요청 지연 제거(soft TTL + hard TTL).
- `IHostedService`/Quartz로 주기 리프레시, 실패 시 기존 값 유지(Fallback).

```csharp
public class WarmUpService : BackgroundService
{
    private readonly CodeTableService _svc;
    public WarmUpService(CodeTableService svc) => _svc = svc;

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while(!stoppingToken.IsCancellationRequested)
        {
            try { _ = await _svc.GetAsync(stoppingToken); }
            catch { /* log */ }
            await Task.Delay(TimeSpan.FromMinutes(2), stoppingToken);
        }
    }
}
```

---

## API 게이트/조건부 요청: ETag/If-None-Match

```csharp
app.MapGet("/api/products/{id:int}", async (int id, HttpContext ctx, AppDbContext db) =>
{
    var dto = await db.Products.AsNoTracking()
        .Where(p => p.Id == id)
        .Select(p => new ProductDto(p.Id, p.Name, p.UpdatedAt))
        .FirstOrDefaultAsync();

    if (dto is null) return Results.NotFound();

    var etag = $"\"p-{dto.Id}-{dto.UpdatedAt.Ticks}\"";
    ctx.Response.Headers.ETag = etag;

    if (ctx.Request.Headers.IfNoneMatch == etag)
        return Results.StatusCode(StatusCodes.Status304NotModified);

    return Results.Json(dto);
});
```

- **전송량/렌더링 시간** 절감, CDN과 궁합 좋음.

---

## — 남용 방지로 안정성 향상

```csharp
builder.Services.AddRateLimiter(_ => _.AddFixedWindowLimiter("api", opt =>
{
    opt.PermitLimit = 100;         // 윈도우 당 허용 수
    opt.Window = TimeSpan.FromMinutes(1);
    opt.QueueLimit = 0;
}));

app.UseRateLimiter();

app.MapGet("/api/hot", GetHotData)
   .RequireRateLimiting("api");
```

- 캐시와 함께 **핫 엔드포인트** 보호.

---

## JSON/HTML 응답 캐시 + “부분 캐시”(뷰 조각)

### Razor Cache Tag Helper

```razor
<cache vary-by-route="id" expires-after="@TimeSpan.FromMinutes(2)">
    @await Component.InvokeAsync("ProductSummary", new { id = RouteData.Values["id"] })
</cache>
```
- **부분 뷰** 캐시: 상세 페이지 내 **변하지 않는 블록**을 캐시해 전체 렌더링 비용 감소.

---

## HTTP 클라이언트 성능 — HttpClientFactory 권장

```csharp
builder.Services.AddHttpClient("remote", c =>
{
    c.Timeout = TimeSpan.FromSeconds(3);
    c.DefaultRequestHeaders.Accept.Add(new MediaTypeWithQualityHeaderValue("application/json"));
})
.ConfigurePrimaryHttpMessageHandler(() => new SocketsHttpHandler
{
    PooledConnectionLifetime = TimeSpan.FromMinutes(5),
    EnableMultipleHttp2Connections = true
});
```

- **연결 풀**/DNS 갱신/HTTP2 최적화. 요청은 항상 팩토리에서 생성된 클라이언트 사용.

---

## Kestrel/프록시/배포 튜닝

- **Nginx/Apache** 앞단: 압축/캐시/HTTP2/3/Keep-Alive 최적화, WebSocket 업그레이드.
- Kestrel: `MaxRequestBodySize`, `Limits.MaxConcurrentConnections` 등 조정(실측 기반).
- 컨테이너: **CPU/메모리 리밋**과 **스레드풀/GC** 상호작용 관찰.

---

## 안전 장치 — 예외/타임아웃/재시도/서킷브레이커

- 캐시 원본 호출은 **타임아웃** 필수.
- 폴리(Polly)로 **재시도/백오프/서킷브레이커** → 실패 격리로 시스템 보호.
- 실패 시 **기존 캐시 값 반환**(Stale While Revalidate) 설계 검토.

```csharp
var policy = Policy.TimeoutAsync(TimeSpan.FromSeconds(2))
    .WrapAsync(Policy.Handle<Exception>()
        .WaitAndRetryAsync(3, i => TimeSpan.FromMilliseconds(100 * i)));

var data = await policy.ExecuteAsync(() => LoadFromOriginAsync());
```

---

## 종합 시나리오: 제품 상세 페이지 성능 설계

1) **키 설계**: `product:v3:id={id}:lang={lang}`
2) **Cache-Aside**: 미스 → DB 조회(Projection/Compiled Query) → 캐시 10분
3) **OutputCache**: 로그인 안 한 사용자에 한해 60초
4) **부분 캐시**: 리뷰 요약 블록 2분, 가격 블록 30초
5) **ETag**: 상세 API에 조건부 요청(전송량 절감)
6) **백그라운드 리프레시**: Top-N 상품 warm-up
7) **무효화**: 상품 업데이트 이벤트 → 해당 키 삭제
8) **관찰**: P95 응답·캐시 히트율·Redis 대기·직렬화 시간 모니터

---

## 체크리스트(요약)

- **DB**: AsNoTracking / Projection / Compiled Query / 인덱스 검토 / N+1 제거
- **Async**: I/O 전부 async, 블로킹 금지
- **직렬화**: STJ Source-Gen / 필요한 필드만 / 압축(Brotli)
- **GC/메모리**: 풀링(ArrayPool/ObjectPool), 큰 할당 회피
- **캐싱**: In-memory(싱글) / Redis(다중) / OutputCache / ResponseCache / TagHelper
- **무효화**: Cache-Aside 원칙, ChangeToken, 버전 키
- **보호**: RateLimit, 타임아웃/재시도/서킷, 실패 시 Stale 허용
- **배포**: 프록시/HTTP2/3/압축/정적 파일 캐시 헤더
- **측정**: P95/P99, 할당, GC, Hot Path, 히트율

---

## 간단 성능 실험 템플릿(BenchmarkDotNet)

```csharp
[MemoryDiagnoser]
public class JsonBench
{
    private readonly ProductDto[] _data = Enumerable.Range(1,1000)
        .Select(i => new ProductDto(i, "Name"+i, i)).ToArray();

    [Benchmark]
    public string Default() => JsonSerializer.Serialize(_data);

    [Benchmark]
    public string SourceGen() => JsonSerializer.Serialize(_data, JsonCtx.Default.ProductDtoArray);
}
```

```bash
dotnet run -c Release
```

---

## 미세 팁

- `IOptionsSnapshot`는 요청마다 바인딩 → **고빈도 경로**에선 **IOptionsMonitor** 사용.
- 로그는 **Information 이하**를 프로덕션에서 줄이고, **구조화 로깅**만 남기기.
- 예외 던지기 비용 큼 → **핫패스에서 예외 대신 코드 경로 분기**.

---

## 최종 요약

| 주제 | 결론 |
|---|---|
| 가장 큰 레버 | **캐싱**(Output/Response/IMemory/Redis) + **DB 접근 축소** |
| 안정성 | RateLimit + 타임아웃/재시도 + 실패 시 Stale |
| 성능 | STJ Source-Gen, Brotli, HTTP/2/3, 풀링, 할당 최소화 |
| 운영 | **측정 기반**(P95/GC/할당/히트율)으로 가설 검증 |
| 설계 | 키/버전/무효화/백그라운드 리프레시/조건부 요청(ETag) |

이 문서의 패턴과 예제들을 조합하면, 별도 프레임워크 없이도 **대부분의 고트래픽 API·웹**에서 **응답시간·부하·비용**을 크게 낮출 수 있다. “항상 측정하고, 간단한 것부터” 적용하자.
