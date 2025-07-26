---
layout: post
title: AspNet - Swagger UI
date: 2025-04-02 19:20:23 +0900
category: AspNet
---
# 📘 ASP.NET Core에서 Swagger UI 적용 완전 가이드

---

## ✅ 1. Swagger란?

- **Swagger**는 OpenAPI 사양에 따라 API 문서를 생성하고 시각화하는 도구입니다.
- Swagger UI는 API 명세를 웹 기반 UI로 보여주며, **테스트와 문서화를 동시에** 제공합니다.
- 개발자와 외부 소비자(프론트엔드, 모바일 등) 모두에게 유용한 **Self-Documenting API 툴**입니다.

---

## 📦 2. NuGet 패키지 설치

ASP.NET Core Web API 프로젝트에 다음 패키지를 설치합니다:

```
dotnet add package Swashbuckle.AspNetCore
```

또는 Visual Studio의 NuGet 패키지 관리자에서  
`Swashbuckle.AspNetCore` 검색 후 설치

---

## 🛠️ 3. Program.cs 설정 (ASP.NET Core 6 이상)

```csharp
var builder = WebApplication.CreateBuilder(args);

// Swagger 서비스 등록
builder.Services.AddEndpointsApiExplorer(); // API 탐색기
builder.Services.AddSwaggerGen();           // Swagger 문서 생성기

var app = builder.Build();

// 개발 환경에서만 Swagger 사용
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();             // swagger.json 생성
    app.UseSwaggerUI();           // Swagger UI 제공
}

app.UseAuthorization();
app.MapControllers();
app.Run();
```

---

## 📄 4. Swagger UI 접속 방법

앱 실행 후 다음 주소로 접속:

```
https://localhost:5001/swagger
```

자동으로 `swagger/v1/swagger.json`을 읽어 UI를 구성합니다.

---

## 🧾 5. Swagger 문서 메타데이터 설정

`Program.cs`에서 다음과 같이 문서 정보를 커스터마이징할 수 있습니다:

```csharp
builder.Services.AddSwaggerGen(options =>
{
    options.SwaggerDoc("v1", new OpenApiInfo
    {
        Title = "MyApp API",
        Version = "v1",
        Description = "MyApp의 백엔드 API 문서입니다.",
        Contact = new OpenApiContact
        {
            Name = "Do Hyun Kim",
            Email = "example@example.com"
        }
    });
});
```

---

## 📌 6. 주석(XML Comments)으로 API 문서화

### ✅ 1단계: `csproj` 파일 수정

```xml
<PropertyGroup>
  <GenerateDocumentationFile>true</GenerateDocumentationFile>
  <NoWarn>1591</NoWarn>
</PropertyGroup>
```

### ✅ 2단계: Program.cs에서 XML 경로 추가

```csharp
var xmlFilename = $"{Assembly.GetExecutingAssembly().GetName().Name}.xml";
options.IncludeXmlComments(Path.Combine(AppContext.BaseDirectory, xmlFilename));
```

### ✅ 3단계: 주석 달기

```csharp
/// <summary>
/// 모든 사용자 목록을 조회합니다.
/// </summary>
[HttpGet]
public IEnumerable<User> GetAllUsers() { ... }
```

---

## 🔒 7. JWT 인증과 Swagger 연동

JWT 인증이 활성화된 API의 경우 Swagger에서 토큰을 입력할 수 있도록 설정해야 합니다.

```csharp
builder.Services.AddSwaggerGen(c =>
{
    c.AddSecurityDefinition("Bearer", new OpenApiSecurityScheme
    {
        Name = "Authorization",
        Type = SecuritySchemeType.Http,
        Scheme = "bearer",
        BearerFormat = "JWT",
        In = ParameterLocation.Header,
        Description = "JWT 토큰을 'Bearer {token}' 형식으로 입력하세요."
    });

    c.AddSecurityRequirement(new OpenApiSecurityRequirement
    {
        {
            new OpenApiSecurityScheme
            {
                Reference = new OpenApiReference {
                    Type = ReferenceType.SecurityScheme,
                    Id = "Bearer"
                }
            },
            new string[] {}
        }
    });
});
```

→ Swagger UI에서 “Authorize” 버튼을 통해 JWT 토큰 입력 가능

---

## 🎨 8. Swagger 커스터마이징 팁

| 기능 | 방법 |
|------|------|
| Swagger UI 기본 경로 변경 | `app.UseSwaggerUI(c => c.SwaggerEndpoint("/swagger/v1/swagger.json", "My API V1"));` |
| 다중 문서 버전 관리 | `SwaggerDoc("v1", ...)`, `SwaggerDoc("v2", ...)` |
| Request/Response 샘플 명시 | `ProducesResponseType`, `Consumes` 어노테이션 |
| Enum 설명 출력 | `options.DescribeAllEnumsAsStrings()` (ASP.NET Core 5 이하) |
| 컨벤션 이름 필터링 | `options.DocInclusionPredicate(...)` 사용 |

---

## 📌 9. 실전 예시 전체 코드

```csharp
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(options =>
{
    options.SwaggerDoc("v1", new OpenApiInfo
    {
        Title = "Product API",
        Version = "v1",
        Description = "제품 정보 관리 API"
    });

    var xmlFilename = $"{Assembly.GetExecutingAssembly().GetName().Name}.xml";
    options.IncludeXmlComments(Path.Combine(AppContext.BaseDirectory, xmlFilename));
});

...

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI(c =>
    {
        c.SwaggerEndpoint("/swagger/v1/swagger.json", "Product API V1");
        c.RoutePrefix = string.Empty; // 루트 경로로 UI 띄우기
    });
}
```

---

## ✅ 요약

| 항목 | 설명 |
|------|------|
| 패키지 설치 | `Swashbuckle.AspNetCore` |
| 서비스 등록 | `AddSwaggerGen`, `AddEndpointsApiExplorer` |
| 개발 환경 조건부 실행 | `UseSwagger()`, `UseSwaggerUI()` |
| 문서 주석 | XML 파일 생성 후 `IncludeXmlComments` |
| 인증 연동 | `AddSecurityDefinition`, `AddSecurityRequirement` |
| 다국어, 버전 관리 등 | SwaggerGen 옵션으로 커스터마이징 |