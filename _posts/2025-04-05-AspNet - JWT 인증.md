---
layout: post
title: AspNet - JWT 인증
date: 2025-04-05 19:20:23 +0900
category: AspNet
---
# 🔐 ASP.NET Core JWT 인증 기본 구성

---

## ✅ 1. JWT란?

**JWT (JSON Web Token)**는  
사용자 인증 정보를 **토큰 형식으로 안전하게 전송**하는 방식.

- 클라이언트가 로그인 후 서버로부터 **JWT 토큰을 발급**받아 보관하고,
- 이후 요청 시 HTTP 헤더에 토큰을 실어 전송하여 **인증을 유지**함.

---

## 📦 2. JWT의 구조

JWT는 **3개의 점(.)으로 구분된 문자열**:

```txt
xxxxx.yyyyy.zzzzz
```

| 구분 | 설명 | 예시 |
|------|------|------|
| Header | 알고리즘 정보 | `{ "alg": "HS256", "typ": "JWT" }` |
| Payload | 사용자 정보(Claims) | `{ "sub": "kimdohyun", "role": "Admin" }` |
| Signature | 검증용 서명 | HMACSHA256(Header + Payload, Secret) |

> ✨ Payload는 **Base64 인코딩된 JSON 객체**로 디코딩 가능하지만, **위변조 방지는 Signature가 맡음**

---

## 🔄 3. JWT 인증 흐름

```text
[1] 로그인 요청 (ID/PW)
    ↓
[2] 서버에서 사용자 검증
    ↓
[3] JWT 토큰 발급 → 클라이언트에 전달
    ↓
[4] 이후 모든 요청 헤더에 토큰 포함
    ↓
[5] 서버는 토큰 유효성 검증 → 인증 허용
```

---

## 🛠️ 4. NuGet 패키지 설치

```bash
dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer
```

---

## ⚙️ 5. Program.cs에 JWT 인증 구성

```csharp
builder.Services.AddAuthentication("Bearer")
    .AddJwtBearer("Bearer", options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,

            ValidIssuer = "myapi",
            ValidAudience = "myclient",
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes("super_secret_jwt_key!123"))
        };
    });

builder.Services.AddAuthorization();
```

```csharp
var app = builder.Build();

app.UseAuthentication(); // JWT 인증 활성화
app.UseAuthorization();  // 권한 확인
```

---

## 🧾 6. JWT 토큰 발급 예제 (Login API)

```csharp
[HttpPost("login")]
public IActionResult Login([FromBody] LoginModel model)
{
    if (model.Username == "admin" && model.Password == "1234")
    {
        var claims = new[]
        {
            new Claim(ClaimTypes.Name, model.Username),
            new Claim(ClaimTypes.Role, "Admin")
        };

        var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes("super_secret_jwt_key!123"));
        var creds = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);

        var token = new JwtSecurityToken(
            issuer: "myapi",
            audience: "myclient",
            claims: claims,
            expires: DateTime.UtcNow.AddHours(1),
            signingCredentials: creds
        );

        return Ok(new
        {
            token = new JwtSecurityTokenHandler().WriteToken(token)
        });
    }

    return Unauthorized();
}
```

---

## 🔐 7. 클라이언트 요청 시 JWT 사용 방법

```http
GET /api/protected
Authorization: Bearer {JWT_TOKEN}
```

- 클라이언트는 **매 요청 시 Authorization 헤더**에 `Bearer {토큰}`을 포함
- 서버는 이 헤더를 기반으로 인증 수행

---

## 📌 8. 인증 및 권한 속성 사용

```csharp
[Authorize]               // 인증된 사용자만 접근
public IActionResult Get() => ...

[Authorize(Roles = "Admin")]   // Admin 역할만 허용
public IActionResult AdminOnly() => ...
```

---

## 🧠 9. 토큰 발급 정보(Claims)

Claim은 토큰의 "Payload"에 저장되는 사용자 정보입니다.

| Claim | 설명 |
|-------|------|
| `sub` | 사용자 고유 ID |
| `name` | 사용자 이름 |
| `role` | 역할 |
| `exp` | 만료 시간 |

→ 직접 Claim 추가 가능

```csharp
new Claim("Department", "HR")
```

---

## 🧰 10. 실전 팁

| 항목 | 설명 |
|------|------|
| 시크릿 키 보안 | 환경 변수 또는 `appsettings.json`에 보관 |
| 만료 시간 | 보통 1~2시간 설정, Refresh Token 별도 구현 필요 |
| CORS 설정 | 클라이언트 도메인에 대한 허용 필요 |
| Swagger 연동 | `AddSecurityDefinition`, `AddSecurityRequirement` 설정 필요 |
| 무상태 인증 | 세션 없이 인증 상태 유지, 서버 확장성 우수 |

---

## ✅ 요약 정리

| 항목 | 설명 |
|------|------|
| 목적 | 로그인 인증 후 토큰으로 상태 유지 |
| 구조 | Header.Payload.Signature |
| 사용 위치 | Web API, SPA, 모바일 앱 |
| 주요 API | `AddJwtBearer`, `TokenValidationParameters` |
| 특징 | 무상태(Stateless), 확장성 우수 |
| 단점 | 토큰 탈취 시 위험 → HTTPS 필수 |