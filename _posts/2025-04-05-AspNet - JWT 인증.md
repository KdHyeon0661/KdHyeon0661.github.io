---
layout: post
title: AspNet - JWT 인증
date: 2025-04-05 19:20:23 +0900
category: AspNet
---
# ASP.NET Core JWT 인증

## 핵심 요약(1페이지)

- **JWT**는 로그인 성공 후 발급되는 **서명된 토큰**(Header.Payload.Signature)으로, 이후 요청의 `Authorization: Bearer` 헤더로 인증.
- 서버는 무상태(Stateless)로 확장성 우수. 단, **탈취 방지(HTTPS/CORS/보관)**와 **수명 관리(만료/갱신)**가 중요.
- **권장 기본값**: `ValidateIssuer/Audience/Lifetime/IssuerSigningKey = true`, `ClockSkew = 0~2분`, 만료 15~60분.
- **Refresh Token**으로 장기 세션 유지, **로테이션/재사용 탐지** 필수.
- **Swagger**에 보안 스키마를 등록해 콘솔에서 Bearer 토큰 테스트.
- **CORS**는 허용 도메인/헤더/메서드/자격증명 세밀 설정.
- 운영 시 **키 롤오버**, **로그/감사**, **속도 제한(Rate Limit)**, **IP/UA 기반 이상 탐지**를 병행.

---

## 프로젝트 준비 및 패키지

```bash
dotnet new webapi -n JwtPlayground
cd JwtPlayground
dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer
dotnet add package Swashbuckle.AspNetCore
```

구성 파일(`appsettings.json`)에 다음을 준비(예: 개발용 비밀):

```json
{
  "Jwt": {
    "Issuer": "myapi",
    "Audience": "myclient",
    "SigningKey": "change-this-dev-key-to-very-long-random-secret",
    "AccessTokenMinutes": 60,
    "RefreshTokenDays": 7
  },
  "AllowedHosts": "*"
}
```

> 운영에서는 키/민감정보를 **환경 변수**나 **Secret Manager/KeyVault**로 분리.

---

## `Program.cs` — JWT 인증/인가 최소 구성

```csharp
using System.Text;
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.IdentityModel.Tokens;

var builder = WebApplication.CreateBuilder(args);

// 1) Jwt 설정 바인딩
var jwtSection = builder.Configuration.GetSection("Jwt");
var issuer      = jwtSection["Issuer"]!;
var audience    = jwtSection["Audience"]!;
var signingKey  = jwtSection["SigningKey"]!; // HS256 개발용
var accessMin   = int.Parse(jwtSection["AccessTokenMinutes"] ?? "60");

// 2) 인증/인가
builder.Services
    .AddAuthentication("Bearer")
    .AddJwtBearer("Bearer", options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidIssuer = issuer,
            ValidAudience = audience,
            IssuerSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(signingKey)),

            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateIssuerSigningKey = true,
            ValidateLifetime = true,

            // 실무 권장: 시계 편차 최소화(동기화됨을 가정)
            ClockSkew = TimeSpan.FromSeconds(30)
        };

        // 401/403 응답 JSON 표준화 등 이벤트 훅
        options.Events = new JwtBearerEvents
        {
            OnChallenge = async ctx =>
            {
                // 기본 처리 막기
                ctx.HandleResponse();
                ctx.Response.StatusCode = StatusCodes.Status401Unauthorized;
                ctx.Response.ContentType = "application/problem+json";
                await ctx.Response.WriteAsJsonAsync(new
                {
                    type = "about:blank",
                    title = "Unauthorized",
                    status = 401,
                    detail = "유효한 인증 자격이 필요합니다."
                });
            },
            OnForbidden = async ctx =>
            {
                ctx.Response.StatusCode = StatusCodes.Status403Forbidden;
                ctx.Response.ContentType = "application/problem+json";
                await ctx.Response.WriteAsJsonAsync(new
                {
                    type = "about:blank",
                    title = "Forbidden",
                    status = 403,
                    detail = "요청한 리소스에 접근 권한이 없습니다."
                });
            }
        };
    });

builder.Services.AddAuthorization(options =>
{
    // 예: Scope 기반 정책
    options.AddPolicy("read:orders", p => p.RequireClaim("scope", "read:orders"));
    options.AddPolicy("write:orders", p => p.RequireClaim("scope", "write:orders"));
});

builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

app.UseHttpsRedirection();
app.UseAuthentication();
app.UseAuthorization();

app.UseSwagger();
app.UseSwaggerUI();

// 보호 엔드포인트 예시
app.MapGet("/api/orders", () => new[] { new { id = 1, item = "Book" } })
   .RequireAuthorization("read:orders");

app.Run();
```

---

## 토큰 발급 API — HS256/RS256, 사용자 클레임

### DTO & 간단 사용자 스토어(예시)

```csharp
public record LoginRequest(string Username, string Password);
public record TokenResponse(string access_token, int expires_in, string token_type, string? refresh_token);

public static class FakeUserStore
{
    private static readonly Dictionary<string, (string Pw, string[] Roles, string[] Scopes)> Users = new()
    {
        ["admin"] = ("1234", new[] { "Admin" }, new[] { "read:orders", "write:orders" }),
        ["user"]  = ("1234", Array.Empty<string>(), new[] { "read:orders" })
    };

    public static (bool ok, string[] roles, string[] scopes) Validate(string u, string p)
        => Users.TryGetValue(u, out var meta) && meta.Pw == p
           ? (true, meta.Roles, meta.Scopes)
           : (false, Array.Empty<string>(), Array.Empty<string>());
}
```

### 발급 서비스 — HS256(대칭키)

```csharp
using System.IdentityModel.Tokens.Jwt;
using System.Security.Claims;
using Microsoft.IdentityModel.Tokens;
using System.Text;

public sealed class TokenService
{
    private readonly IConfiguration _cfg;
    public TokenService(IConfiguration cfg) => _cfg = cfg;

    public (string token, DateTime expiresUtc) CreateAccessToken(string userId, string[] roles, string[] scopes)
    {
        var issuer = _cfg["Jwt:Issuer"]!;
        var audience = _cfg["Jwt:Audience"]!;
        var signingKey = _cfg["Jwt:SigningKey"]!;
        var accessMin = int.Parse(_cfg["Jwt:AccessTokenMinutes"] ?? "60");

        var claims = new List<Claim>
        {
            new(JwtRegisteredClaimNames.Sub, userId),
            new(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString()),
            new(ClaimTypes.Name, userId),
        };

        foreach (var r in roles)
            claims.Add(new Claim(ClaimTypes.Role, r));
        foreach (var s in scopes)
            claims.Add(new Claim("scope", s));

        var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(signingKey));
        var creds = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);

        var now = DateTime.UtcNow;
        var expires = now.AddMinutes(accessMin);

        var token = new JwtSecurityToken(
            issuer: issuer,
            audience: audience,
            claims: claims,
            notBefore: now,
            expires: expires,
            signingCredentials: creds);

        var encoded = new JwtSecurityTokenHandler().WriteToken(token);
        return (encoded, expires);
    }
}
```

> **RS256(RSA)로 전환** 시 `RsaSecurityKey`를 사용하고 **공개키(검증)/개인키(서명) 분리**로 보안/배포 유연성↑. JWKS/OIDC로 공개키 배포 가능(13장).

### 로그인 엔드포인트(Access + Refresh 발급)

```csharp
// Program.cs 혹은 Controller
builder.Services.AddSingleton<TokenService>();
builder.Services.AddSingleton<RefreshTokenStore>(); // 아래 정의

// RefreshToken 저장/검증(메모리 예시 → 운영은 DB/분산캐시)
public sealed class RefreshTokenStore
{
    // key: token string, value: (userId, expires, revoked, replacement)
    private readonly Dictionary<string, (string userId, DateTime exp, bool revoked, string? replacedBy)> _db = new();

    public string Issue(string userId, TimeSpan lifetime)
    {
        var token = Convert.ToBase64String(Guid.NewGuid().ToByteArray());
        _db[token] = (userId, DateTime.UtcNow.Add(lifetime), false, null);
        return token;
    }
    public (bool ok, string userId) ValidateAndRotate(string token, out string newToken)
    {
        newToken = "";
        if (!_db.TryGetValue(token, out var row)) return (false, "");
        if (row.revoked || DateTime.UtcNow > row.exp) return (false, "");
        // 재사용 방지: 기존 토큰 즉시 revoke + 새 토큰 발급
        var nt = Convert.ToBase64String(Guid.NewGuid().ToByteArray());
        _db[token] = (row.userId, row.exp, true, nt);
        _db[nt] = (row.userId, DateTime.UtcNow.AddDays(7), false, null);
        newToken = nt;
        return (true, row.userId);
    }
}

app.MapPost("/api/auth/login", (LoginRequest req, TokenService tokens, IConfiguration cfg, RefreshTokenStore rts) =>
{
    var (ok, roles, scopes) = FakeUserStore.Validate(req.Username, req.Password);
    if (!ok) return Results.Unauthorized();

    var (access, exp) = tokens.CreateAccessToken(req.Username, roles, scopes);
    var refresh = rts.Issue(req.Username, TimeSpan.FromDays(int.Parse(cfg["Jwt:RefreshTokenDays"] ?? "7")));

    return Results.Ok(new TokenResponse(
        access, (int)(exp - DateTime.UtcNow).TotalSeconds, "Bearer", refresh));
});
```

---

## 토큰 검증 파라미터 해설(실무 포인트)

- `ValidateIssuer/Audience`: **발급자/대상**을 엄격히 검증해 타 서비스 토큰 거부.
- `ValidateLifetime`: 만료/`nbf` 검증. `ClockSkew`를 최소화(0~2분)하려면 서버 시간 동기화 필수.
- `ValidateIssuerSigningKey`: 서명키 필수. HS256은 **키 길이**를 충분히 길게(32+바이트).
- **권장 수명**: Access 15~60분. 너무 길면 탈취 피해 커짐.
- **클레임 표준화**: `sub`(ID), `jti`(Unique), `iat`(발급시각), `scope`(권한), `role`(역할).
- 토큰 크기 제한: 프록시/게이트웨이의 헤더 제한 고려.

---

## 권한 — Authorize/Role/Policy/Scope

### 엔드포인트 예시(역할/정책 혼합)

```csharp
[Authorize] // 인증 필요
[ApiController]
[Route("api/[controller]")]
public class OrdersController : ControllerBase
{
    [HttpGet]
    [Authorize(Policy = "read:orders")]
    public IActionResult GetAll() => Ok(new[] { new { id = 1, item = "Book" } });

    [HttpPost]
    [Authorize(Roles = "Admin")]
    [Authorize(Policy = "write:orders")]
    public IActionResult Create(object dto) => Ok(new { ok = true });
}
```

> 역할은 coarse-grained, scope/claim은 fine-grained 권한. **정책 기반**으로 통일하면 확장성↑.

---

## Refresh Token 설계(로테이션/재사용 탐지/블랙리스트)

### 엔드포인트

```csharp
public record RefreshRequest(string refresh_token);

app.MapPost("/api/auth/refresh", (RefreshRequest req, TokenService tokens, RefreshTokenStore rts) =>
{
    if (!rts.ValidateAndRotate(req.refresh_token, out var newRefresh).ok)
        return Results.Unauthorized();

    var userId = rts.ValidateAndRotate(req.refresh_token, out _).userId; // 이미 위에서 처리했으므로 리팩터 필요
    // 간단화 위해 다시 validate: 실제 코드는 한번만 호출하고 결과 유지
    var (ok, uid) = rts.ValidateAndRotate(req.refresh_token, out newRefresh);
    if (!ok) return Results.Unauthorized();

    // 사용자 권한 재구성(변경 반영)
    var roles = FakeUserStore.Validate(uid, "1234").roles; // 실제론 DB에서
    var scopes = FakeUserStore.Validate(uid, "1234").scopes;

    var (access, exp) = tokens.CreateAccessToken(uid, roles, scopes);
    return Results.Ok(new TokenResponse(access, (int)(exp - DateTime.UtcNow).TotalSeconds, "Bearer", newRefresh));
});
```

> 단순화 예제. 실제 구현은 **한 번의 조회/검증/회전**으로 처리, **재사용 탐지 시 세션 강제 종료** 및 **경보**가 필요.

### 베스트 프랙티스

- **로테이션**: Refresh가 사용될 때마다 즉시 폐기 후 새 Refresh 생성.
- **재사용 탐지**: 이미 사용/폐기된 Refresh가 재사용되면 **계정 탈취 의심**으로 **모든 Refresh 무효화**(세션 전부 종료).
- 저장소: DB/분산 캐시로 유지(토큰 문자열, 소유자, 만료, 폐기 여부, 교체 토큰, 발급/IP/UA 메타).
- API는 **Idempotent**하게 설계(중복 요청 내성).

---

## Swagger(OpenAPI)와 JWT 연동

```csharp
builder.Services.AddSwaggerGen(c =>
{
    c.SwaggerDoc("v1", new() { Title = "My API", Version = "v1" });

    c.AddSecurityDefinition("Bearer", new Microsoft.OpenApi.Models.OpenApiSecurityScheme
    {
        Name = "Authorization",
        Type = Microsoft.OpenApi.Models.SecuritySchemeType.Http,
        Scheme = "bearer",
        BearerFormat = "JWT",
        In = Microsoft.OpenApi.Models.ParameterLocation.Header,
        Description = "Bearer {your JWT}"
    });

    c.AddSecurityRequirement(new Microsoft.OpenApi.Models.OpenApiSecurityRequirement
    {
        {
            new Microsoft.OpenApi.Models.OpenApiSecurityScheme
            {
                Reference = new Microsoft.OpenApi.Models.OpenApiReference
                {
                    Type = Microsoft.OpenApi.Models.ReferenceType.SecurityScheme,
                    Id = "Bearer"
                }
            }, Array.Empty<string>()
        }
    });
});
```

Swagger UI에서 **Authorize 버튼** 클릭 → `Bearer <토큰>` 입력 후 API 호출.

---

## CORS 구성 — SPA/모바일 연동 체크리스트

```csharp
builder.Services.AddCors(options =>
{
    options.AddPolicy("Spa", p =>
    {
        p.WithOrigins("https://app.example.com")
         .AllowAnyHeader()
         .AllowAnyMethod()
         .AllowCredentials(); // 필요 시만(쿠키/SignalR). Bearer만 사용하면 보통 필요없음
    });
});

app.UseCors("Spa");
```

- 프론트엔드 도메인을 명시(`WithOrigins`).
- 프리플라이트(OPTIONS) 응답을 허용.
- 인증 헤더(`Authorization`) 허용.
- **개발 편의의 `AllowAnyOrigin()`+`AllowCredentials()` 조합**은 금지(브라우저 정책상 예외/보안 위험).

---

## 실무 보안 체크리스트

- [ ] **HTTPS 강제**, HSTS, 프록시/로드밸런서에서 헤더 보존.
- [ ] **SigningKey 보호**: 환경 변수/KeyVault, 파일 권한 제한, 주기적 **Key Rollover**.
- [ ] Access 만료 짧게(15~60분), Refresh 로테이션+재사용 탐지.
- [ ] **로그/감사**: `sub`, `jti`, 요청 IP/UA(개인정보 최소화), 실패/금지 이벤트.
- [ ] **속도 제한/봇 방어**: 로그인/리프레시 엔드포인트 Rate Limit.
- [ ] **오류 응답 표준화**(ProblemDetails), 민감 정보 노출 금지.
- [ ] **CORS 엄격화**: Origins/Headers/Methods 제한.
- [ ] **권한 정책** 중앙집중, 최소권한 원칙.

---

## — 에러/로그/헤더

```csharp
.AddJwtBearer("Bearer", options =>
{
    // ...
    options.Events = new JwtBearerEvents
    {
        OnMessageReceived = ctx =>
        {
            // 예: WebSocket/SignalR 특정 쿼리스트링에서 토큰 추출
            if (ctx.Request.Path.StartsWithSegments("/ws") &&
                ctx.Request.Query.TryGetValue("access_token", out var token))
            {
                ctx.Token = token;
            }
            return Task.CompletedTask;
        },
        OnTokenValidated = ctx =>
        {
            // 여기가 인증 성공 지점: 추가 검사/로그
            var sub = ctx.Principal!.FindFirst("sub")?.Value;
            ctx.HttpContext.Items["user_id"] = sub;
            return Task.CompletedTask;
        },
        OnAuthenticationFailed = ctx =>
        {
            ctx.NoResult();
            ctx.Response.StatusCode = 401;
            ctx.Response.ContentType = "application/problem+json";
            return ctx.Response.WriteAsJsonAsync(new
            {
                title = "Invalid token",
                status = 401,
                detail = ctx.Exception.Message
            });
        }
    };
});
```

---

## + Bearer(API), Minimal API

```csharp
builder.Services
    .AddAuthentication()
    .AddCookie("Cookies", o => { /* MVC 로그인용 */ })
    .AddJwtBearer("Bearer", o => { /* 위 구성 */ });

builder.Services.AddAuthorization();

app.MapGet("/api/me", (HttpContext http) =>
{
    var name = http.User.Identity?.Name ?? "guest";
    return Results.Ok(new { name });
}).RequireAuthorization(new AuthorizeAttribute { AuthenticationSchemes = "Bearer" });
```

MVC는 Cookies, API는 Bearer를 적용. 컨트롤러/엔드포인트별로 `AuthenticationSchemes`를 지정.

---

## OIDC/JWKS(공개키), RS256 & Key Rollover

- **RS256**로 서명하면 공개키로 검증 가능 → 여러 서비스가 **중앙 IdP**의 키를 참조(JWKS URL).
- ASP.NET Core에서 `Authority`/`MetadataAddress`로 OIDC 메타데이터를 읽고 `AddJwtBearer`가 자동 키 로드.

```csharp
.AddJwtBearer("Bearer", o =>
{
    o.Authority = "https://login.example.com"; // OIDC
    o.TokenValidationParameters = new TokenValidationParameters
    {
        ValidateAudience = true,
        ValidAudience = "myapi"
    };
    o.RequireHttpsMetadata = true;
});
```

- 키 회전 시 IdP의 JWKS가 업데이트되면 API는 자동 반영.
- 자체 발급일 경우 JWKS 엔드포인트를 운영하거나, **키 버전(kid) 포함** 후 롤오버 전략 수립.

---

## 통합 테스트(xUnit)에서 JWT 다루기

- 테스트에서 **직접 토큰 생성** 후 `Authorization` 헤더에 부착.

```csharp
[Fact]
public async Task Protected_Endpoint_Requires_Valid_Token()
{
    using var app = new WebApplicationFactory<Program>();
    var client = app.CreateClient();

    // 유효하지 않은 토큰
    client.DefaultRequestHeaders.Authorization =
        new System.Net.Http.Headers.AuthenticationHeaderValue("Bearer", "invalid");

    var res = await client.GetAsync("/api/orders");
    Assert.Equal(System.Net.HttpStatusCode.Unauthorized, res.StatusCode);
}
```

- 더 정교하게는 **테스트용 서명키**로 토큰 생성해 통과케이스 구성.

---

## 운영 팁 & 트러블슈팅

### 운영 팁

- **게이트웨이/프록시**에서 JWT 검증 오프로딩 시, 내부 마이크로서비스는 `X-User-*` 헤더를 신뢰하기 전에 **서명/원천 검증**을 재확인할 것.
- **토큰 크기**가 큰 경우(클레임 많음) 헤더 한도 초과 위험. 필요한 최소 클레임만 담고 나머지는 API 조회.

### 트러블슈팅 표

| 증상 | 원인 | 해결 |
|---|---|---|
| 401 Unauthorized(유효 토큰인데) | `aud/iss` 불일치, `ClockSkew`/시간 문제 | `ValidIssuer/Audience` 재확인, 서버 시간 동기화 |
| 403 Forbidden | 인증 성공, 정책 실패 | 역할/스코프 누락, 정책명 오타 |
| Swagger에서 401/HTML 반환 | 인증 실패시 리다이렉트/HTML | JwtBearerEvents로 JSON 401/403 표준화 |
| SPA에서 요청 실패(CORS) | Origin/헤더 미허용 | CORS 정책 수정(`Authorization` 허용) |
| Refresh 재사용 공격 의심 | 로테이션 미구현/검출 미흡 | 로테이션 + 재사용 탐지 → 모든 세션 무효화 |

---

## Minimal API — 전체 샘플(요지)

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);
// ... Jwt 구성(3장), TokenService/RefreshTokenStore 등록(4장)
var app = builder.Build();

app.MapPost("/api/auth/login", /* 4장 코드 */);
app.MapPost("/api/auth/refresh", /* 7장 코드 */);

app.MapGet("/api/orders", () => new[] { new { id = 1 } })
   .RequireAuthorization("read:orders");

app.UseSwagger();
app.UseSwaggerUI();

app.UseAuthentication();
app.UseAuthorization();

app.Run();
```

---

## RS256 발급(요지)

```csharp
var rsa = RSA.Create();
// 운영: PEM/KeyVault/KMS에서 로드
var key = new RsaSecurityKey(rsa) { KeyId = "kid-2025-01" };
var creds = new SigningCredentials(key, SecurityAlgorithms.RsaSha256);
var token = new JwtSecurityToken(issuer, audience, claims, expires: exp, signingCredentials: creds);
```

검증 측은 **Authority/JWKS**로 `kid`에 맞는 공개키 조회.

---

## 결론

- **JWT**는 무상태 인증의 표준. 핵심은 **짧은 Access + 안전한 Refresh**의 수명 전략과 **정책 기반 권한**.
- 운영 품질은 **키 관리/롤오버**, **CORS/HTTPS/오류 표준화**, **로깅/감사/속도 제한**에서 갈린다.
- 본 가이드의 **발급·검증·이벤트·로테이션·Swagger·CORS** 샘플을 조합하면,
  **API·모바일·SPA** 어디든 안정적인 인증 체인을 빠르게 구현할 수 있다.
