---
layout: post
title: AspNet - Swagger UI
date: 2025-04-02 19:20:23 +0900
category: AspNet
---
# ASP.NET Core에서 Swagger UI 적용

## 목차

1. 핵심 요약(빠른 스타트)
2. 기본 설정(패키지, Program.cs, Development/Production 분기)
3. 문서 메타데이터 & XML 주석 & Annotations
4. 보안 연동(JWT, OAuth2)과 테스트 흐름
5. API 버전 관리와 Swagger 다중 문서
6. 예제 페이로드/응답 스니펫(Example) 넣기
7. 스키마 커스터마이징(Enums, Nullable, Polymorphism)
8. Operation 커스터마이징(헤더/필터/상태코드 표준화)
9. Minimal API와 Swagger(WithOpenApi)
10. 파일 업로드/다운로드 문서화
11. ProblemDetails 및 에러 응답 문서화
12. 캐싱/ETag/조건부 요청 문서화
13. Swagger UI 커스터마이징, ReDoc 병행
14. 배포/운영 Tips(보안, 성능, 문서 품질 체크리스트)
15. 종합 예제(Program.cs, Controller, Filters)

---

## 1. 핵심 요약(빠른 스타트)

```bash
dotnet add package Swashbuckle.AspNetCore
```

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();                // /swagger/v1/swagger.json
    app.UseSwaggerUI();              // /swagger
}

app.MapControllers();
app.Run();
```

- XML 주석 문서화: csproj에 `<GenerateDocumentationFile>true</GenerateDocumentationFile>` 추가 → `IncludeXmlComments` 등록
- JWT 버튼: `AddSecurityDefinition` + `AddSecurityRequirement`

---

## 2. 기본 설정(패키지, Program.cs, Development/Production 분기)

### 2.1 필수 패키지

```bash
dotnet add package Swashbuckle.AspNetCore
# 선택(예제/필터 등 확장)
dotnet add package Swashbuckle.AspNetCore.Annotations
dotnet add package Swashbuckle.AspNetCore.Filters
```

### 2.2 Program.cs 기본 골격

```csharp
using System.Reflection;
using Microsoft.OpenApi.Models;

var builder = WebApplication.CreateBuilder(args);

// MVC/Minimal API 동시 사용 가능
builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();

builder.Services.AddSwaggerGen(c =>
{
    c.SwaggerDoc("v1", new OpenApiInfo
    {
        Title = "MyApp API",
        Version = "v1",
        Description = "MyApp 백엔드 API 문서",
        Contact = new OpenApiContact { Name = "Dev Team", Email = "dev@example.com" },
        License = new OpenApiLicense { Name = "MIT" }
    });

    // XML 주석
    var xml = $"{Assembly.GetExecutingAssembly().GetName().Name}.xml";
    var xmlPath = Path.Combine(AppContext.BaseDirectory, xml);
    if (File.Exists(xmlPath))
        c.IncludeXmlComments(xmlPath, includeControllerXmlComments: true);

    // 널 참조 타입 반영(참고: Swashbuckle 6+)
    c.SupportNonNullableReferenceTypes();

    // 중복 타입명 충돌 방지(네임스페이스 포함)
    c.CustomSchemaIds(t => t.FullName);

    // Annotations 사용(예: [SwaggerOperation], [SwaggerResponse])
    c.EnableAnnotations();
});

var app = builder.Build();

// Prod에서도 Swagger를 쓰고 싶다면 Feature-flag/환경변수로 제어
if (app.Environment.IsDevelopment() || builder.Configuration.GetValue<bool>("Swagger:EnableInProd"))
{
    app.UseSwagger();
    app.UseSwaggerUI(ui =>
    {
        ui.SwaggerEndpoint("/swagger/v1/swagger.json", "MyApp API v1");
        ui.RoutePrefix = "docs"; // /docs 로 접근
        ui.DisplayRequestDuration();
        ui.EnableFilter();       // UI 상단 필터
    });
}

app.UseHttpsRedirection();
app.UseAuthorization();
app.MapControllers();
app.Run();
```

---

## 3. 문서 메타데이터 & XML 주석 & Annotations

### 3.1 csproj 설정

```xml
<PropertyGroup>
  <TargetFramework>net8.0</TargetFramework>
  <Nullable>enable</Nullable>
  <GenerateDocumentationFile>true</GenerateDocumentationFile>
  <NoWarn>1591</NoWarn> <!-- 공개 멤버 주석 경고 무시(선택) -->
</PropertyGroup>
```

### 3.2 컨트롤러/액션 XML 주석 & Annotations

```csharp
using Swashbuckle.AspNetCore.Annotations;

[ApiController]
[Route("api/v1/users")]
public class UsersController : ControllerBase
{
    /// <summary>사용자 목록을 페이징 조회합니다.</summary>
    /// <param name="page">페이지(1부터)</param>
    /// <param name="pageSize">페이지 크기(기본 20)</param>
    [HttpGet]
    [SwaggerOperation(Summary = "사용자 목록 조회", Description = "페이징/정렬/필터 지원")]
    [ProducesResponseType(typeof(Paged<UserDto>), StatusCodes.Status200OK)]
    public IActionResult Get([FromQuery] int page = 1, [FromQuery] int pageSize = 20)
        => Ok(new Paged<UserDto>(/*...*/));
}
```

---

## 4. 보안 연동(JWT, OAuth2)과 테스트 흐름

### 4.1 JWT Bearer 버튼

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
        Description = "Bearer {token} 형태로 입력"
    });

    c.AddSecurityRequirement(new OpenApiSecurityRequirement
    {
        {
            new OpenApiSecurityScheme { Reference = new OpenApiReference
              { Type = ReferenceType.SecurityScheme, Id = "Bearer" } },
            Array.Empty<string>()
        }
    });
});
```

컨트롤러 레벨에서 `[Authorize]`를 적용하면 UI에서 **Authorize** 버튼으로 토큰 입력 후 테스트 가능.

### 4.2 OAuth2(Authorization Code 예)

```csharp
c.AddSecurityDefinition("OAuth2", new OpenApiSecurityScheme
{
    Type = SecuritySchemeType.OAuth2,
    Flows = new OpenApiOAuthFlows
    {
        AuthorizationCode = new OpenApiOAuthFlow
        {
            AuthorizationUrl = new Uri("https://idp.example.com/oauth2/authorize"),
            TokenUrl = new Uri("https://idp.example.com/oauth2/token"),
            Scopes = new Dictionary<string, string> {
                ["api.read"] = "Read access", ["api.write"] = "Write access"
            }
        }
    }
});

c.AddSecurityRequirement(new OpenApiSecurityRequirement
{
    {
        new OpenApiSecurityScheme { Reference =
            new OpenApiReference { Type = ReferenceType.SecurityScheme, Id = "OAuth2" } },
        new [] { "api.read", "api.write" }
    }
});
```

UI 구동 시:

```csharp
app.UseSwaggerUI(ui =>
{
    ui.OAuthClientId("swagger-ui-client");
    ui.OAuthUsePkce();
    ui.OAuthScopes("api.read", "api.write");
});
```

---

## 5. API 버전 관리와 Swagger 다중 문서

### 5.1 Microsoft.AspNetCore.Mvc.Versioning 연동(선택)

```bash
dotnet add package Microsoft.AspNetCore.Mvc.Versioning
dotnet add package Microsoft.AspNetCore.Mvc.Versioning.ApiExplorer
```

```csharp
builder.Services.AddApiVersioning(o =>
{
    o.AssumeDefaultVersionWhenUnspecified = true;
    o.DefaultApiVersion = new Microsoft.AspNetCore.Mvc.ApiVersion(1,0);
    o.ReportApiVersions = true;
});

builder.Services.AddVersionedApiExplorer(o =>
{
    o.GroupNameFormat = "'v'VVV"; // v1, v1.1
    o.SubstituteApiVersionInUrl = true;
});
```

```csharp
// SwaggerGen에 버전 문서 등록
builder.Services.AddSwaggerGen();
builder.Services.ConfigureOptions<ConfigureSwaggerOptions>(); // 커스텀 옵션 등록
```

```csharp
// 예시: 각 API 버전에 SwaggerDoc 생성
public sealed class ConfigureSwaggerOptions : IConfigureOptions<SwaggerGenOptions>
{
    private readonly IApiVersionDescriptionProvider _provider;
    public ConfigureSwaggerOptions(IApiVersionDescriptionProvider provider) => _provider = provider;

    public void Configure(SwaggerGenOptions options)
    {
        foreach (var d in _provider.ApiVersionDescriptions)
        {
            options.SwaggerDoc(d.GroupName, new OpenApiInfo
            {
                Title = "MyApp API",
                Version = d.ApiVersion.ToString(),
                Description = d.IsDeprecated ? "This version is deprecated" : null
            });
        }
    }
}
```

```csharp
// UI에 버전별 엔드포인트 노출
app.UseSwaggerUI(ui =>
{
    var provider = app.Services.GetRequiredService<IApiVersionDescriptionProvider>();
    foreach (var d in provider.ApiVersionDescriptions)
        ui.SwaggerEndpoint($"/swagger/{d.GroupName}/swagger.json", $"MyApp API {d.GroupName}");
});
```

---

## 6. 예제 페이로드/응답 스니펫(Example) 넣기

### 6.1 Swashbuckle.Filters 사용(간결)

```csharp
// Program.cs
builder.Services.AddSwaggerGen(c =>
{
    c.ExampleFilters();
});
builder.Services.AddSwaggerExamplesFromAssemblyOf<Program>();
```

```csharp
// Example 클래스
public class UserCreateRequestExample : IExamplesProvider<UserCreateRequest>
{
    public UserCreateRequest GetExamples() => new()
    {
        Name = "Alice",
        Email = "alice@example.com"
    };
}
```

```csharp
[HttpPost]
[SwaggerRequestExample(typeof(UserCreateRequest), typeof(UserCreateRequestExample))]
[SwaggerResponseExample(201, typeof(UserCreateResponseExample))]
public IActionResult Create(UserCreateRequest request) => Created(...);
```

### 6.2 Attributes 없이 수동 Example

```csharp
c.MapType<DateOnly>(() => new OpenApiSchema
{
    Type = "string",
    Format = "date",
    Example = new Microsoft.OpenApi.Any.OpenApiString("2025-11-07")
});
```

---

## 7. 스키마 커스터마이징(Enums, Nullable, Polymorphism)

### 7.1 Enums 문자열로 노출 + 설명

```csharp
builder.Services.AddSwaggerGen(c =>
{
    c.SchemaGeneratorOptions = new()
    {
        // net8+에서는 아래와 같은 방식 대신 DescribeAllEnumsAsStrings는 제거됨. MapType/SchemaFilter 활용.
    };
});

// SchemaFilter 예시
public class EnumSchemaFilter : ISchemaFilter
{
    public void Apply(OpenApiSchema schema, SchemaFilterContext ctx)
    {
        if (ctx.Type.IsEnum)
        {
            schema.Enum = Enum.GetNames(ctx.Type)
                .Select(n => (Microsoft.OpenApi.Any.IOpenApiAny)new Microsoft.OpenApi.Any.OpenApiString(n))
                .ToList();
            schema.Type = "string";
            schema.Format = null;
            schema.Description = $"One of [{string.Join(", ", Enum.GetNames(ctx.Type))}]";
        }
    }
}
```

```csharp
c.SchemaFilter<EnumSchemaFilter>();
```

### 7.2 Nullable 참조 타입 반영

```csharp
c.SupportNonNullableReferenceTypes(); // 참조형 nullability 반영
```

### 7.3 Polymorphism(oneOf/allOf) 표현

```csharp
// 예: Base -> Photo/Article
public abstract record MediaBase { public int Id { get; init; } public string Type { get; init; } = default!; }
public record Photo : MediaBase { public int Width { get; init; } public int Height { get; init; } }
public record Article : MediaBase { public string Title { get; init; } = default!; }

// OperationFilter로 oneOf 수동 지정
public class PolymorphismOperationFilter : IOperationFilter
{
    public void Apply(OpenApiOperation op, OperationFilterContext ctx)
    {
        foreach (var resp in op.Responses.Values)
        foreach (var content in resp.Content.Values)
        {
            if (content.Schema.Reference?.Id == nameof(MediaBase))
            {
                content.Schema.OneOf = new()
                {
                    ctx.SchemaGenerator.GenerateSchema(typeof(Photo), ctx.SchemaRepository),
                    ctx.SchemaGenerator.GenerateSchema(typeof(Article), ctx.SchemaRepository),
                };
                content.Schema.Discriminator = new OpenApiDiscriminator { PropertyName = "type" };
            }
        }
    }
}
```

```csharp
c.OperationFilter<PolymorphismOperationFilter>();
```

---

## 8. Operation 커스터마이징(헤더/필터/상태코드 표준화)

### 8.1 공통 헤더(예: X-Request-Id) 추가

```csharp
public class CorrelationIdOperationFilter : IOperationFilter
{
    public void Apply(OpenApiOperation operation, OperationFilterContext context)
    {
        operation.Parameters ??= new List<OpenApiParameter>();
        operation.Parameters.Add(new OpenApiParameter
        {
            Name = "X-Request-Id",
            In = ParameterLocation.Header,
            Required = false,
            Schema = new OpenApiSchema { Type = "string" },
            Description = "요청 상관관계 ID"
        });
    }
}
```

```csharp
c.OperationFilter<CorrelationIdOperationFilter>();
```

### 8.2 표준 상태코드/응답 헤더 부착

- `ProducesResponseType`로 최소 명세
- 공통 에러는 OperationFilter로 ProblemDetails 추가

```csharp
public class ProblemDetailsOperationFilter : IOperationFilter
{
    public void Apply(OpenApiOperation op, OperationFilterContext ctx)
    {
        foreach (var code in new[] { "400", "401", "403", "404", "409", "500" })
        {
            op.Responses.TryAdd(code, new OpenApiResponse
            {
                Description = $"HTTP {code}",
                Content = new Dictionary<string, OpenApiMediaType>
                {
                    ["application/problem+json"] = new OpenApiMediaType
                    {
                        Schema = ctx.SchemaGenerator.GenerateSchema(typeof(ProblemDetails), ctx.SchemaRepository)
                    }
                }
            });
        }
    }
}
```

```csharp
c.OperationFilter<ProblemDetailsOperationFilter>();
```

---

## 9. Minimal API와 Swagger(WithOpenApi)

```csharp
var app = builder.Build();

app.MapGet("/api/v1/ping", () => Results.Ok(new { message = "pong" }))
   .WithName("Ping")
   .WithOpenApi(op =>
   {
       op.Summary = "헬스 체크";
       op.Description = "서버 상태 확인용 엔드포인트";
       return op;
   });

app.Run();
```

- Minimal API 라우트의 메타를 `WithOpenApi`로 채우면 Swagger에 예쁘게 반영된다.

---

## 10. 파일 업로드/다운로드 문서화

### 10.1 업로드

```csharp
[HttpPost("upload")]
[Consumes("multipart/form-data")]
[ProducesResponseType(StatusCodes.Status201Created)]
public IActionResult Upload([FromForm] IFormFile file)
{
    // 검증(확장자/시그니처/크기), 저장...
    return Created("/api/v1/files/123", null);
}
```

Swagger는 `multipart/form-data`를 감지해 파일 선택 UI 제공.

### 10.2 다운로드(바이너리)

```csharp
[HttpGet("files/{id}")]
[Produces("application/octet-stream")]
public IActionResult Download(int id)
{
    var bytes = System.IO.File.ReadAllBytes("path");
    return File(bytes, "application/octet-stream", "report.pdf");
}
```

---

## 11. ProblemDetails 및 에러 응답 문서화

- ASP.NET Core의 **ProblemDetails** 사용 권장(`application/problem+json`)
- 전역 예외 처리 미들웨어에서 변환 → Swagger에 4xx/5xx 응답으로 기술(앞선 OperationFilter 참고)

---

## 12. 캐싱/ETag/조건부 요청 문서화

```csharp
[HttpGet("{id}")]
[ProducesResponseType(typeof(UserDto), 200)]
[ResponseCache(Duration = 60, Location = ResponseCacheLocation.Any)]
public IActionResult GetById(int id)
{
    // ETag/Last-Modified 헤더 설정은 미들웨어/필터에서 수행 가능
    Response.Headers.ETag = "\"v3-1f9c0b\"";
    return Ok(new UserDto { /*...*/ });
}
```

Swagger 설명에 `ETag`, `If-None-Match`, `If-Match`의 사용법을 주석/OperationFilter로 추가.

---

## 13. Swagger UI 커스터마이징, ReDoc 병행

### 13.1 Swagger UI 옵션

```csharp
app.UseSwaggerUI(ui =>
{
    ui.RoutePrefix = "docs";
    ui.SwaggerEndpoint("/swagger/v1/swagger.json", "MyApp API v1");
    ui.DocumentTitle = "MyApp API Docs";
    ui.DefaultModelsExpandDepth(-1); // 좌측 Models 접기
    ui.InjectStylesheet("/swagger-ui/custom.css"); // 커스텀 CSS
    ui.InjectJavascript("/swagger-ui/custom.js", "text/javascript");
});
```

정적 파일 제공 설정 후 `/wwwroot/swagger-ui/custom.css` 배치.

### 13.2 ReDoc 병행(선택)

```csharp
dotnet add package Redoc.AspNetCore
```

```csharp
app.UseReDoc(o =>
{
    o.RoutePrefix = "redoc";
    o.SpecUrl("/swagger/v1/swagger.json");
    o.DocumentTitle = "MyApp API ReDoc";
});
```

---

## 14. 배포/운영 Tips(보안, 성능, 문서 품질 체크리스트)

- **보안**: 운영에서 Swagger 노출 시 **IP 제한/Auth** 적용, 관리자만 접근
- **성능**: 대형 스키마 생성 비용 → 캐시/사전 생성 사용 고려
- **문서 품질**:
  - 모든 엔드포인트에 **Summary/Description/Status codes** 기입
  - 요청/응답 DTO 분리(엔티티 직접 노출 금지)
  - 빈 200 응답 대신 204 사용 구분
  - 에러는 ProblemDetails 규격 통일
  - 예제(Example) 제공으로 클라이언트 학습 비용 절감

---

## 15. 종합 예제

### 15.1 DTO

```csharp
public record UserDto(int Id, string Name, string Email);
public record Paged<T>(IReadOnlyList<T> Data, int Page, int PageSize, int Total);
public class UserCreateRequest { public string Name { get; set; } = ""; public string Email { get; set; } = ""; }
public class UserCreateResponse { public int Id { get; set; } public string Location { get; set; } = ""; }
```

### 15.2 Controller

```csharp
[ApiController]
[Route("api/v1/users")]
public class UsersController : ControllerBase
{
    /// <summary>사용자 목록을 페이징 조회합니다.</summary>
    [HttpGet]
    [ProducesResponseType(typeof(Paged<UserDto>), 200)]
    public IActionResult Get([FromQuery] int page = 1, [FromQuery] int pageSize = 20)
    {
        var list = new List<UserDto> { new(101, "Alice", "alice@example.com") };
        return Ok(new Paged<UserDto>(list, page, pageSize, total: 1));
    }

    /// <summary>새 사용자를 생성합니다.</summary>
    /// <remarks>멱등성 키(Idempotency-Key) 헤더로 중복 생성 방지 가능</remarks>
    [HttpPost]
    [Consumes("application/json")]
    [ProducesResponseType(typeof(UserCreateResponse), 201)]
    [ProducesResponseType(typeof(ProblemDetails), 400)]
    public IActionResult Create([FromBody] UserCreateRequest req)
    {
        if (string.IsNullOrWhiteSpace(req.Email)) return ValidationProblem();
        var id = 123;
        var location = $"/api/v1/users/{id}";
        return Created(location, new UserCreateResponse { Id = id, Location = location });
    }

    /// <summary>사용자 상세를 반환합니다.</summary>
    [HttpGet("{id:int}")]
    [ProducesResponseType(typeof(UserDto), 200)]
    [ProducesResponseType(404)]
    public IActionResult GetById(int id)
        => id == 123 ? Ok(new UserDto(123, "Alice", "alice@example.com")) : NotFound();
}
```

### 15.3 Program.cs(요약)

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();

builder.Services.AddSwaggerGen(c =>
{
    c.SwaggerDoc("v1", new() { Title = "MyApp API", Version = "v1" });
    var xml = $"{Assembly.GetExecutingAssembly().GetName().Name}.xml";
    var xmlPath = Path.Combine(AppContext.BaseDirectory, xml);
    if (File.Exists(xmlPath)) c.IncludeXmlComments(xmlPath, true);

    c.SupportNonNullableReferenceTypes();
    c.CustomSchemaIds(t => t.FullName);
    c.EnableAnnotations();

    // JWT
    c.AddSecurityDefinition("Bearer", new OpenApiSecurityScheme
    {
        Name = "Authorization", Type = SecuritySchemeType.Http, Scheme = "bearer",
        BearerFormat = "JWT", In = ParameterLocation.Header
    });
    c.AddSecurityRequirement(new OpenApiSecurityRequirement{
      { new OpenApiSecurityScheme { Reference = new OpenApiReference {
            Type = ReferenceType.SecurityScheme, Id = "Bearer"}}, Array.Empty<string>() }
    });

    // 공통 필터(예: ProblemDetails/헤더)
    c.OperationFilter<CorrelationIdOperationFilter>();
    c.OperationFilter<ProblemDetailsOperationFilter>();
});

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI(ui =>
    {
        ui.SwaggerEndpoint("/swagger/v1/swagger.json", "MyApp API v1");
        ui.RoutePrefix = string.Empty; // 루트(/)에서 바로 노출
    });
}

app.MapControllers();
app.Run();
```

---

## 결론

- **Swagger/OpenAPI**는 단순 UI가 아니라 **API 계약(Contract)** 을 자동화·표준화하는 **개발·운영 핵심 도구**다.
- 이 글의 확장 주제들(버전 관리, 보안, 예제, 스키마/오퍼레이션 필터, Minimal API, 에러/캐시 문서화, UI 커스터마이징)을 적용하면 **외부/내부 소비자 모두가 신뢰할 수 있는 API 문서**를 제공할 수 있다.
- 무엇보다 **일관성**이 중요하다. 프로젝트 초기에 **문서 규칙(상태코드/에러 포맷/페이징/보안/버전 전략)** 을 결정하고 모든 팀원이 동일 규칙을 따르도록 하자.
