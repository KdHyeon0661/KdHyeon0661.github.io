---
layout: post
title: Avalonia - Core Web API 특성
date: 2025-03-28 19:20:23 +0900
category: Avalonia
---
# 🏷️ ASP.NET Core Web API 특성(Attribute) 완전 정리

---

## ✅ 1. [ApiController]

### ✅ 역할

- 이 특성을 `Controller` 클래스에 붙이면 **Web API 전용 기능**이 활성화됩니다.
- 주로 `ControllerBase`와 함께 사용.

```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
```

### ✅ 주요 효과

| 기능 | 설명 |
|------|------|
| 자동 모델 유효성 검사 | `ModelState.IsValid`가 자동 처리됨 |
| [FromBody] 생략 가능 | 복잡한 객체는 기본적으로 Body에서 추출 |
| 400 BadRequest 자동 응답 | 유효성 검사 실패 시 자동 반환 |
| 바인딩 명시 오류 감지 | 누락된 필드에 대해 400 반환 |

---

## ✅ 2. [Route]

### ✅ 역할

- API의 **URL 경로를 정의**함
- 컨트롤러 레벨과 액션 레벨에서 모두 사용 가능

```csharp
[Route("api/[controller]")]
public class ProductsController : ControllerBase

[HttpGet("{id}")]
public IActionResult GetById(int id)
```

| 표현식 | 설명 |
|--------|------|
| `[controller]` | 컨트롤러 이름 (`ProductsController` → `products`) |
| `[action]` | 액션 메서드 이름 |
| `api/products/{id}` | URL 파라미터 포함 |

---

## ✅ 3. [HttpGet], [HttpPost], [HttpPut], [HttpDelete]

### ✅ 역할

- HTTP 메서드에 따라 **API 메서드를 구분**하는 데 사용
- RESTful API 구현에 필수

```csharp
[HttpGet]            // GET /api/products
public IActionResult GetAll() { ... }

[HttpPost]           // POST /api/products
public IActionResult Create(Product product) { ... }

[HttpPut("{id}")]    // PUT /api/products/5
public IActionResult Update(int id, Product product) { ... }

[HttpDelete("{id}")] // DELETE /api/products/5
public IActionResult Delete(int id) { ... }
```

---

## ✅ 4. [FromBody], [FromQuery], [FromRoute], [FromForm]

### ✅ 역할

| 특성 | 설명 | 데이터 위치 |
|------|------|-------------|
| `[FromBody]` | JSON에서 파싱 | Request Body |
| `[FromQuery]` | URL 쿼리스트링 | `?key=value` |
| `[FromRoute]` | URL 경로 파라미터 | `/api/products/5` |
| `[FromForm]` | HTML 폼에서 전달된 값 | multipart/form-data |

### ✅ 예시

```csharp
[HttpPost]
public IActionResult Create([FromBody] Product product)

[HttpGet]
public IActionResult Search([FromQuery] string keyword)

[HttpGet("{id}")]
public IActionResult Get([FromRoute] int id)

[HttpPost("upload")]
public IActionResult Upload([FromForm] IFormFile file)
```

---

## 📌 5. 자동 Model Validation ([ApiController] 사용 시)

```csharp
public class Product
{
    [Required]
    public string Name { get; set; }

    [Range(0, 100000)]
    public decimal Price { get; set; }
}
```

```csharp
[HttpPost]
public IActionResult Create(Product product)
{
    // ModelState 자동 검사됨 (유효하지 않으면 400 BadRequest 반환)
    return Ok(product);
}
```

---

## 🧪 6. 라우팅 고급 예제

```csharp
[Route("api/products")]
public class ProductsController : ControllerBase
{
    [HttpGet] // GET /api/products
    public IActionResult GetAll() => ...

    [HttpGet("{id:int}")] // GET /api/products/3
    public IActionResult GetById(int id) => ...

    [HttpGet("category/{name}")] // GET /api/products/category/electronics
    public IActionResult GetByCategory(string name) => ...
}
```

---

## ⚠️ 7. 주의사항

| 항목 | 설명 |
|------|------|
| `[ApiController]`는 Web API 전용 기능만 활성화됨 (View 사용 안 함) |
| `[FromBody]`는 하나의 파라미터에만 사용 가능 |
| `[FromForm]`은 파일 업로드 시 자주 사용됨 |
| 모델 유효성 검사를 직접 다루려면 `ModelState.IsValid` 활용 |
| 기본 Route 설정을 잘못하면 API 경로가 꼬일 수 있음 |

---

## 🧩 8. 전체 예제

```csharp
[ApiController]
[Route("api/[controller]")]
public class UsersController : ControllerBase
{
    [HttpGet]
    public IActionResult GetUsers([FromQuery] int page = 1) => Ok(...);

    [HttpGet("{id}")]
    public IActionResult GetById([FromRoute] int id) => Ok(...);

    [HttpPost]
    public IActionResult Register([FromBody] UserDto dto)
    {
        if (!ModelState.IsValid) return BadRequest(ModelState);
        return Ok(dto);
    }

    [HttpPost("upload-avatar")]
    public IActionResult Upload([FromForm] IFormFile avatar) => Ok();
}
```

---

## ✅ 요약

| 특성 | 설명 |
|------|------|
| `[ApiController]` | Web API 편의 기능 제공 (자동 유효성 검사, [FromBody] 생략 등) |
| `[Route]` | URL 경로 지정 |
| `[HttpGet]`, `[HttpPost]`, ... | HTTP 메서드와 라우트 연결 |
| `[FromBody]` | JSON → 객체 바인딩 |
| `[FromQuery]` | URL 파라미터 |
| `[FromForm]` | 폼 데이터 바인딩 |
| `[FromRoute]` | 경로 변수 추출 |

---

## 🔜 추천 다음 주제

- ✅ `Model Validation` 심화
- ✅ `Swagger`에서 특성 자동 인식
- ✅ `Route Constraint` 사용법 (`int`, `guid`, `minlength`)
- ✅ `Custom ActionFilter` 만들기
- ✅ `Versioning`, `HATEOAS`, `Content Negotiation`