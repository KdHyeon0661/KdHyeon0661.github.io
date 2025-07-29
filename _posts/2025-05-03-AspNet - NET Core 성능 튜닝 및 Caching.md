---
layout: post
title: AspNet - NET Core 성능 튜닝 및 Caching
date: 2025-05-03 19:20:23 +0900
category: AspNet
---
# ⚡ ASP.NET Core 성능 튜닝 및 Caching 전략

---

## ✅ 1. 성능 튜닝의 핵심 포인트

ASP.NET Core에서 성능 최적화를 위해 고려할 수 있는 영역은 다음과 같아요:

| 영역 | 최적화 전략 |
|------|--------------|
| ❗ 데이터베이스 | Index 최적화, Lazy/Eager Loading 관리, N+1 제거 |
| ❗ I/O 지연 | 비동기 처리 (`async/await`) 철저히 적용 |
| ❗ 반복 로직 | 캐싱, 미리 계산된 데이터 활용 |
| ❗ View 렌더링 | Razor Partial 미리 컴파일, ViewData 최소화 |
| ❗ 네트워크 전송 | JSON 최소화, Gzip 압축, HTTP/2 |
| ❗ GC 부담 | 객체 풀링, 구조체 활용, `IOptionsSnapshot` 지양 |

---

## 🧠 2. 캐싱(Caching)의 필요성

> "계산하지 않아도 되는 데이터를 다시 계산하지 않도록 저장하는 것"

캐싱은 **성능 최적화의 첫 걸음**이자, **반복 작업에 대한 비용 절감**입니다.

### 캐싱 대상 예시:

- DB에서 자주 조회되는 코드값, 설정값
- 결과가 변하지 않는 API 호출 결과
- 파일/이미지 응답
- 로그인 사용자 프로필 정보 등

---

## 🧱 3. ASP.NET Core의 캐싱 종류

| 유형 | 설명 | 스코프 |
|------|------|--------|
| ✅ In-Memory Cache | 서버 메모리에 저장 | 프로세스 내 |
| ✅ Distributed Cache | Redis, SQL Server 등에 저장 | 여러 서버 공유 |
| ✅ Response Cache | HTTP 응답 자체를 캐싱 | 클라이언트/프록시 |
| ✅ Output Cache (.NET 8+) | Razor 페이지나 Controller 응답 전체 캐싱 | 프레임워크 통합 |
| ✅ Static Files Cache | wwwroot 파일 캐싱 (`Cache-Control`) | CDN 활용 가능 |

---

## 💾 4. In-Memory Cache

### 💡 사용 예시

```csharp
// 등록
services.AddMemoryCache();
```

```csharp
public class MyService
{
    private readonly IMemoryCache _cache;

    public MyService(IMemoryCache cache)
    {
        _cache = cache;
    }

    public string GetData()
    {
        return _cache.GetOrCreate("my-key", entry =>
        {
            entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5);
            return "Hello from DB!";
        });
    }
}
```

> 동일 요청 시 메모리에서 즉시 반환되어 DB 접근이 생략됨

---

## 🌐 5. Distributed Cache (Redis)

### 💡 설치 및 구성

```bash
dotnet add package Microsoft.Extensions.Caching.StackExchangeRedis
```

```csharp
services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = "localhost:6379";
});
```

```csharp
public class MyService
{
    private readonly IDistributedCache _cache;

    public MyService(IDistributedCache cache)
    {
        _cache = cache;
    }

    public async Task<string> GetCachedValueAsync()
    {
        var value = await _cache.GetStringAsync("key");
        if (value == null)
        {
            value = "Loaded from DB";
            await _cache.SetStringAsync("key", value, new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10)
            });
        }
        return value;
    }
}
```

> 서버 간 공유 가능한 캐시를 구성할 수 있어, **클러스터 구성 필수 시 사용**

---

## 📦 6. Response Caching

### 💡 응답 캐시 (클라이언트/중간 프록시용)

```csharp
services.AddResponseCaching();

app.UseResponseCaching();
```

```csharp
[ResponseCache(Duration = 60, Location = ResponseCacheLocation.Any)]
public IActionResult Get()
{
    return Ok("This is cached for 60 seconds");
}
```

> 결과 HTML/JSON 등 응답 자체가 그대로 캐시됨  
> 헤더 기반이므로 프록시 서버(CDN, 브라우저)도 활용 가능

---

## 🚀 7. Output Caching (.NET 8 이상)

### 💡 Razor 페이지 전체 결과를 캐싱

```csharp
app.MapGet("/", () => "Hello world!")
    .CacheOutput(p => p.Expire(TimeSpan.FromMinutes(1)));
```

- 미들웨어 기반 Output Caching 도입
- Header 캐싱보다 더 유연하고 세밀한 구성 가능
- 인증 상태, QueryString 등에 따라 분기 처리 가능

---

## 🧠 8. 캐싱 전략 설계 시 고려사항

| 항목 | 설명 |
|------|------|
| ❓ 캐싱 유효시간 | 짧게 유지할지, 절대 만료/슬라이딩 만료 |
| 🔐 사용자별 캐싱 여부 | 로그인/인증 상태 고려 필요 |
| ⛔ 무효화 정책 | 데이터 변경 시 캐시 강제 무효화 |
| 💥 동시성 문제 | 캐시 초기화 시 중복 처리 방지 (`Lazy<T>` 활용 등) |
| 📏 크기 관리 | 대용량 캐시 시 TTL 조정/메모리 모니터링 필요 |

---

## 🔧 9. 실무 팁: 캐시 키 관리

```csharp
var key = $"ProductDetail:{productId}:{culture}";
```

- Prefix + 식별자 조합으로 키 관리
- 사용자, 페이지, 언어 등으로 분리
- 해시 키 사용 시에는 충돌 방지 필요

---

## 🔍 10. 캐싱과 성능 측정 도구

| 도구 | 설명 |
|------|------|
| `dotnet-counters` | 실시간 CPU/GC/메모리 |
| `MiniProfiler` | 쿼리 및 렌더링 시간 측정 |
| `Application Insights` | Azure 기반 실시간 분석 |
| `BenchmarkDotNet` | 코드 단위 성능 비교 |
| `PerfView`, `dotTrace` | GC, HotPath 분석용 |

---

## ✅ 요약

| 주제 | 핵심 요약 |
|------|-----------|
| 성능 튜닝 대상 | DB, 비동기 처리, View, 캐싱 |
| 캐시 종류 | In-Memory, Redis, Response, Output |
| 실무 기준 | TTL 설정, 무효화 전략, 사용자별 분리 |
| 측정 도구 | MiniProfiler, BenchmarkDotNet, AppInsights |