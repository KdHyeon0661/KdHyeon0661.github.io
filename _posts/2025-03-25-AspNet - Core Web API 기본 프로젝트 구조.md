---
layout: post
title: AspNet - Core Web API 기본 프로젝트 구조
date: 2025-03-25 19:20:23 +0900
category: AspNet
---
# 🌐 ASP.NET Core Web API 기본 프로젝트 구조 완전 정리

---

## ✅ 1. ASP.NET Core Web API란?

- ASP.NET Core Web API는 **RESTful API**를 구축하기 위한 프레임워크.
- **View 없이 JSON/XML 데이터를 반환**하는 백엔드 중심 애플리케이션.
- 프론트엔드와 분리된 SPA, 모바일 앱, 외부 서비스 연동 등에 주로 사용됨.

---

## 🗂️ 2. 기본 프로젝트 구조

```bash
dotnet new webapi -n MyApiApp
```

### 기본 생성 구조 예시

```
MyApiApp/
├── Controllers/
│   └── WeatherForecastController.cs
├── Models/
│   └── WeatherForecast.cs
├── Program.cs
├── appsettings.json
├── Properties/
│   └── launchSettings.json
```

---

## 📁 3. 주요 폴더와 파일 설명

### ✅ `Controllers/`

- API 요청을 처리하는 핵심 컨트롤러
- 일반적으로 `SomethingController.cs` 형식
- `[ApiController]` 및 `[Route("api/[controller]")]` 특성 사용

```csharp
[ApiController]
[Route("api/[controller]")]
public class WeatherForecastController : ControllerBase
{
    [HttpGet]
    public IEnumerable<WeatherForecast> Get() => ...;
}
```

---

### ✅ `Models/`

- API에서 사용하는 데이터 모델 클래스 정의
- DTO, 엔티티, 요청/응답 전용 모델 포함

```csharp
public class WeatherForecast
{
    public DateTime Date { get; set; }
    public int TemperatureC { get; set; }
    public string? Summary { get; set; }
}
```

---

### ✅ `Program.cs`

- 앱의 진입점
- 서비스 등록 (`builder.Services.Add...`) 및 미들웨어 설정 (`app.Use...`)

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers(); // 컨트롤러 등록
builder.Services.AddEndpointsApiExplorer(); // Swagger
builder.Services.AddSwaggerGen();

var app = builder.Build();

app.UseSwagger();
app.UseSwaggerUI();

app.MapControllers(); // 라우팅

app.Run();
```

---

### ✅ `appsettings.json`

- 구성 파일
- DB 연결 문자열, 로깅, 사용자 정의 설정 등 포함

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information"
    }
  },
  "AllowedHosts": "*"
}
```

---

### ✅ `launchSettings.json`

- 실행 환경 구성
- 개발 시 포트, 환경 변수 등 설정

```json
"profiles": {
  "MyApiApp": {
    "commandName": "Project",
    "launchBrowser": true,
    "applicationUrl": "https://localhost:5001;http://localhost:5000",
    "environmentVariables": {
      "ASPNETCORE_ENVIRONMENT": "Development"
    }
  }
}
```

---

## 🔄 4. 컨트롤러 예시

```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private static readonly List<Product> Products = new();

    [HttpGet]
    public ActionResult<IEnumerable<Product>> GetAll() => Products;

    [HttpPost]
    public IActionResult Create(Product product)
    {
        Products.Add(product);
        return CreatedAtAction(nameof(GetById), new { id = product.Id }, product);
    }

    [HttpGet("{id}")]
    public ActionResult<Product> GetById(int id)
    {
        var product = Products.FirstOrDefault(p => p.Id == id);
        if (product == null) return NotFound();
        return product;
    }
}
```

---

## 🧩 5. MVC vs Web API 차이

| 항목 | MVC (View 기반) | Web API |
|------|------------------|---------|
| 리턴 | View (.cshtml) | JSON, XML |
| 목적 | UI 렌더링 | 데이터 제공 |
| Controller | `Controller` | `ControllerBase` |
| 특성 | `@ViewBag`, `View()` | `return Ok()`, `return NotFound()` |
| 용도 | 웹 페이지 | 모바일/SPA 백엔드 |

---

## 🛠 6. Web API 개발에 필요한 추가 구성

| 기능 | 구성 방법 |
|------|-----------|
| Swagger (API 문서) | `AddSwaggerGen()` + `UseSwagger()` |
| CORS 허용 | `builder.Services.AddCors()` |
| 인증/인가 | `AddAuthentication()`, `AddAuthorization()` |
| Entity Framework Core | `AddDbContext()` |
| Validation | `DataAnnotations` 또는 `FluentValidation` |
| 로깅 | Serilog, NLog 등 |
| 버전 관리 | `[ApiVersion]` + `MapToApiVersion` |

---

## 🚀 7. Swagger 기본 설정

Swagger는 API 문서를 자동 생성하고, 테스트 UI 제공

```csharp
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();
```

```csharp
app.UseSwagger();
app.UseSwaggerUI();
```

접속 경로:  
`https://localhost:5001/swagger`

---

## ✅ 8. 추천 구조 확장 예시

```
/Controllers
    ProductsController.cs
/Models
    Product.cs
    ProductDto.cs
/Data
    AppDbContext.cs
    IProductRepository.cs
/Services
    ProductService.cs
/DTOs
    ProductCreateRequest.cs
    ProductResponse.cs
/Middleware
    ExceptionHandlingMiddleware.cs
```

---

## 🧪 9. 주요 라우팅 형식

| 요청 | 의미 |
|------|------|
| `GET /api/products` | 전체 목록 |
| `GET /api/products/3` | ID=3 항목 조회 |
| `POST /api/products` | 새 항목 추가 |
| `PUT /api/products/3` | ID=3 항목 수정 |
| `DELETE /api/products/3` | ID=3 항목 삭제 |

---

## ✅ 요약

| 구성 요소 | 역할 |
|-----------|------|
| `Program.cs` | 앱 진입점, 서비스 및 미들웨어 설정 |
| `Controllers/` | API 라우팅 처리 |
| `Models/` | 데이터 구조 정의 |
| `appsettings.json` | 구성 설정 (DB, 로깅 등) |
| `Swagger` | API 문서 제공 |
| `Services/Repository` | 비즈니스 로직 및 데이터 접근 계층 구성 |

---

## 🔜 추천 다음 주제

- ✅ RESTful API 설계 원칙
- ✅ DTO vs Entity 분리 전략
- ✅ Global Exception Filter
- ✅ API 버전 관리 (v1, v2)
- ✅ 테스트 (xUnit, Postman, Swagger UI)
