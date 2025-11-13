---
layout: post
title: AspNet - RESTful API
date: 2025-03-28 20:20:23 +0900
category: AspNet
---
# RESTful API 설계

## 목차

1. REST 핵심 복습 — 리소스·표현·상태 전이
2. 리소스 모델링 — URI 규칙, 계층·연관, 다형성, 수집(Collection) 설계
3. 요청 설계 — 페이징/정렬/필터/검색, 필드 마스킹, 확장 포함(Include)
4. 응답 규격 — JSON 스키마, 메타/링크/HATEOAS 최소형, Enveloping 선택 기준
5. 상태 코드·표준 헤더 — 일관성 테이블, 대표 응답 패턴
6. 오류 처리 — ProblemDetails 규격, Validation 오류 형식
7. 부분 업데이트 — PUT vs PATCH, JSON Merge Patch vs JSON Patch
8. 멱등성·동시성 — Idempotency-Key, ETag/If-Match, 409·412 처리
9. 캐싱 전략 — Cache-Control, ETag, Last-Modified, 304
10. 장기 작업·비동기 — 202 Accepted + Location/Retry-After, 작업 조회 모델
11. 파일 업로드·큰 요청 — multipart, 제한·보안, 스트리밍
12. 버전 관리 — URI/헤더/협상, 지원 종료(Deprecation) 시그널링
13. 보안 — 인증·인가, CORS, 입력 검증, Rate Limit 헤더, 감사 로깅
14. 문서화 — OpenAPI(Swagger), 예제, 상태코드, 스키마 제너레이터 팁
15. 테스트·운영 체크리스트 — 계약 테스트, 모니터링, 호환성 가드
16. 통합 예제 — “Users/Posts” 도메인 API 설계/구현 스니펫

---

## 1. REST 핵심 복습 — 리소스·표현·상태 전이

- **리소스(Resource)**: 식별 가능한 대상(사용자, 주문, 결제 등) — **URI**로 식별.
- **표현(Representation)**: 리소스를 나타내는 구체 포맷(JSON/XML 등).
- **상태 전이(State Transfer)**: 하이퍼미디어/링크 또는 규약된 URI로 **리소스 상태 변경**(POST/PUT/PATCH/DELETE).

**무상태성**: 각 HTTP 요청은 **완전한 자기 서술적 정보**를 가져야 하며, 서버 세션 상태에 의존하지 않는다(액세스 토큰/컨텍스트는 헤더나 본문에 포함).

---

## 2. 리소스 모델링 — URI 규칙, 계층·연관, 다형성, 수집 설계

### 2.1 URI 규칙(권장)

- **복수형 명사** 사용: `/users`, `/orders`
- **소문자-케밥케이스**: `/purchase-orders`, `/access-logs`
- **동사는 금지**: `POST /users`가 “생성”의 의미를 담는다. `POST /createUser`는 비권장.
- **리소스 식별자**: `/{resource}/{id}` — 숫자, GUID, 슬러그(고유문자열) 허용
- **연관/계층**:
  - `/users/{id}/posts` (소유 관계, 강한 종속)
  - `/posts/{id}/comments`
  - 다만, **깊이를 2~3단계 이내**로 유지. 너무 깊으면 별도 루트 리소스로 승격 고려.

### 2.2 관계 설계

- **포린키 기반 조회**: `/posts?userId=42`
- **중첩 리소스**: `GET /users/42/posts` — 특정 사용자 관점의 컬렉션
- **링크/HATEOAS 최소형**: 응답에 `links.self`, `links.related`(선택)를 추가해 탐색성↑

### 2.3 다형성·서브타입

- 단일 컬렉션에서 여러 타입을 제공해야 한다면 `type` 필드 추가:
  ```json
  { "id": 1, "type": "photo", "url": "...", "width": 1024, "height": 768 }
  { "id": 2, "type": "article", "title": "...", "body": "..." }
  ```
- 또는 `/media/photos`, `/media/articles`로 분리.

---

## 3. 요청 설계 — 페이징/정렬/필터/검색, 필드 마스킹, Include

### 3.1 페이징

- **Offset 모델**:
  - `GET /users?page=1&pageSize=20`
  - 응답에 `meta.total`, `meta.page`, `meta.pageSize`, `links.next/prev` 권장
- **Cursor 모델**(대규모/변경 잦은 목록에 안정적):
  - `GET /events?cursor=eyJpZCI6MTIzfQ==&limit=50`
  - 다음 페이지 커서만 제공하여 **불변성** 유지

### 3.2 정렬/필터/검색

- 정렬: `GET /users?sort=-createdAt,name` (`-` 내림차순)
- 필터: `GET /orders?status=paid&from=2025-01-01&to=2025-01-31`
- 검색(풀텍스트/간단검색): `GET /posts?q=aspnet core`
- 인덱스/쿼리 비용 고려하여 **필터 허용 목록(white-list)** 명세화

### 3.3 부분 필드 선택(마스킹)

- `GET /users?fields=id,name,email`
- 대전제: 과도한 N+1 필드 노출 방지, 응답 트래픽 절감

### 3.4 연관 포함(Include)

- `GET /orders?include=items,customer`
- 포함 가능한 연관을 **명세/문서화**하고, 깊이를 제한.

---

## 4. 응답 규격 — JSON 스키마, 메타/링크/HATEOAS 최소형, Enveloping

### 4.1 단일/목록 응답 스켈레톤

```json
// 단일
{
  "data": {
    "id": 101,
    "name": "Alice",
    "email": "alice@example.com"
  },
  "links": { "self": "/api/v1/users/101" }
}

// 목록(페이징)
{
  "data": [{ "id": 101, "name": "Alice" }, { "id": 102, "name": "Bob" }],
  "meta": { "page": 1, "pageSize": 20, "total": 132 },
  "links": {
    "self": "/api/v1/users?page=1&pageSize=20",
    "next": "/api/v1/users?page=2&pageSize=20"
  }
}
```

> **Enveloping(랩핑)**은 메타/링크/일관성을 위해 유용. 단순 API는 평평한 구조도 가능. 프로젝트 전체에서 **일관성**이 가장 중요.

---

## 5. 상태 코드·표준 헤더 — 일관성 표

| 상황 | 상태 | 헤더/설명 |
|---|---|---|
| 조회 성공 | `200 OK` | 본문 포함 |
| 생성 성공 | `201 Created` | **Location**: 새 리소스 URL, 본문(생성된 리소스) |
| 수정 성공(본문 無) | `204 No Content` | PUT/PATCH 후 내용 없음 |
| 잘못된 요청 | `400 Bad Request` | ProblemDetails(유효성/형식 오류) |
| 인증 필요 | `401 Unauthorized` | WWW-Authenticate(optional) |
| 권한 없음 | `403 Forbidden` |  |
| 없음 | `404 Not Found` |  |
| 충돌 | `409 Conflict` | 중복, 상태 충돌 |
| 사전조건 실패 | `412 Precondition Failed` | If-Match/ETag 불일치 |
| 과도한 요청 | `429 Too Many Requests` | **Retry-After**, RateLimit-Remaining 등 |
| 서버 오류 | `500 Internal Server Error` |  |
| 비동기 수락 | `202 Accepted` | **Location**(작업 조회), **Retry-After** |

---

## 6. 오류 처리 — ProblemDetails 규격(추천)

### 6.1 표준 오류 포맷

```json
{
  "type": "https://api.example.com/problems/validation-error",
  "title": "One or more validation errors occurred.",
  "status": 400,
  "traceId": "00-8c3c...-01",
  "errors": {
    "email": ["Invalid email format"],
    "name": ["Required"]
  }
}
```

- `type`(문서화된 오류 유형 URL), `title`, `status`, `traceId`, `errors`(필드별)
- ASP.NET Core: `ProblemDetails`, `ValidationProblemDetails` 자동/수동 사용

### 6.2 ASP.NET Core 샘플

```csharp
if (!ModelState.IsValid)
    return ValidationProblem(ModelState); // 400 + ValidationProblemDetails
```

---

## 7. 부분 업데이트 — PUT vs PATCH

- **PUT**: **전체 교체**(클라이언트가 리소스 전체 표현을 제공)
- **PATCH**: **부분 업데이트**(일부 필드만 변경)
  - **JSON Merge Patch** (`application/merge-patch+json`): 단순·직관
  - **JSON Patch** (`application/json-patch+json`): 연산 목록(op/add/replace/remove) — 강력하지만 복잡

### 7.1 JSON Merge Patch 예

요청:
```http
PATCH /api/v1/users/101
Content-Type: application/merge-patch+json

{ "name": "Alice Kim" }
```
서버:
- 기존 리소스를 불러온 후 변경 필드만 병합 → 저장
- 유효성 검사·권한 체크 필요

---

## 8. 멱등성·동시성 — Idempotency-Key, ETag/If-Match

### 8.1 멱등성(Idempotency)

- **PUT**, **DELETE**는 멱등적(같은 요청 반복해도 결과 동일)
- **POST**는 기본 비멱등 → 중복 생성 방지:
  - 클라이언트가 **Idempotency-Key** 헤더를 제공
  - 서버는 키별 결과를 캐시/락하여 **중복 생성 방지** 및 **동일 응답 반환**

```http
POST /api/v1/payments
Idempotency-Key: 6b6c9d7f-...

{ "amount": 1000, "currency": "KRW" }
```

### 8.2 동시성 제어(Optimistic Concurrency)

- 응답에 **ETag** 제공 → 클라이언트는 갱신 시 `If-Match: <etag>`:
  - 불일치 시 **412 Precondition Failed** (낙관적 잠금)

```http
GET /api/v1/users/101
ETag: "v3-7b2c..."

PUT /api/v1/users/101
If-Match: "v3-7b2c..."
```

---

## 9. 캐싱 전략 — Cache-Control, ETag, Last-Modified, 304

- **ETag** 또는 **Last-Modified**를 제공
- 재검사 요청:
  - `If-None-Match: "v3-7b2c..."` → 변경 없으면 **304 Not Modified**
  - `If-Modified-Since: <date>`
- **Cache-Control**
  - 정적/변경 적은 리소스: `Cache-Control: public, max-age=60`
  - 민감/개인화: `Cache-Control: private, no-store`

---

## 10. 장기 작업·비동기 — 202 + Location/Retry-After

대량 처리/외부 연동 등 **즉시 완료 불가**:
- **202 Accepted** 반환 + **Location: /operations/{id}**
- 클라이언트는 폴링 또는 콜백(Webhook)으로 완료 확인

```http
HTTP/1.1 202 Accepted
Location: /api/v1/operations/8b1f...
Retry-After: 5
```

작업 조회:
```json
{
  "id": "8b1f...",
  "status": "running", // queued | running | succeeded | failed
  "links": { "self": "/api/v1/operations/8b1f..." }
}
```

---

## 11. 파일 업로드·큰 요청

- **multipart/form-data** + `IFormFile`(ASP.NET Core)
- 서버·프록시 **요청 크기 한도** 설정, 확장자/시그니처 검증, 바이러스 스캔
- 대용량은 **직접 업로드(Pre-signed URL)** 권장: 클라이언트 → 스토리지, 서버에는 메타데이터만 전달

---

## 12. 버전 관리 — URI/헤더/협상, Deprecation 시그널

- **URI 방식(권장)**: `/api/v1/users`
- **헤더/미디어 타입**: `Accept: application/vnd.myapp.v2+json` (운영 복잡도↑)
- **Deprecation 알림**:
  - `Deprecation: true`, `Sunset: <date>`, 문서 URL 제공
  - 응답 본문/문서에 마이그레이션 안내

---

## 13. 보안 — 인증·인가, CORS, 입력 검증, Rate Limit

- **HTTPS 필수**, HSTS 적용
- **JWT/OAuth2/OIDC**: 표준 기반 인증, 최소 권한(Scopes/Claims)
- **입력 검증**: 크기·형식·화이트리스트, SQL/NoSQL 인젝션 대비
- **CORS**: 명시 허용 도메인/메서드/헤더
- **Rate Limiting 헤더**:
  - `RateLimit-Limit`, `RateLimit-Remaining`, `RateLimit-Reset`(표준·벤더 헤더 중 택1)
- **감사 로깅**: 요청 ID, 주체, 리소스, 결과, 실패 이유

---

## 14. 문서화 — OpenAPI(Swagger)

- **정확한 스키마**(요청/응답), **상태코드**, **예제** 포함
- 가능한 한 **DTO를 명확히 분리**(엔티티 직접 노출 금지)
- 오류(Response) 스키마(ProblemDetails)도 문서화
- 버전별 문서 그룹화(v1/v2)

ASP.NET Core 예:
```csharp
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(c =>
{
    c.SwaggerDoc("v1", new() { Title = "My API", Version = "v1" });
    // JWT 보안 정의, 예제 스키마 등 추가
});
```

---

## 15. 테스트·운영 체크리스트

- **계약 테스트(Consumer-Driven)**: 프런트/클라이언트 기대와 서버 API 계약 검증
- **회귀 테스트**: 버전·엔드포인트·상태코드·스키마 변경 감지
- **관측성**: 로깅(구조화), 메트릭(레이트/지연/오류율), 분산 트레이싱(TraceId)
- **릴리즈 정책**: 점진적 롤아웃(카나리), 롤백, 피처 플래그
- **호환성 가드**: 필드 추가는 호환적, **필드 제거/의미 변경은 메이저 버전으로**

---

## 16. 통합 예제 — Users/Posts 도메인

### 16.1 URI/요청/응답 예시

#### 생성(POST)
```http
POST /api/v1/users
Content-Type: application/json

{
  "name": "Alice",
  "email": "alice@example.com"
}
```

응답:
```http
HTTP/1.1 201 Created
Location: /api/v1/users/101
ETag: "v1-67fd3a"
```
```json
{
  "data": {
    "id": 101,
    "name": "Alice",
    "email": "alice@example.com",
    "createdAt": "2025-11-07T11:20:45Z"
  },
  "links": { "self": "/api/v1/users/101" }
}
```

#### 목록 조회(페이징/정렬/필드)
```http
GET /api/v1/users?page=1&pageSize=20&sort=-createdAt&fields=id,name,email
```
```json
{
  "data": [
    { "id": 101, "name": "Alice", "email": "alice@example.com" }
  ],
  "meta": { "page": 1, "pageSize": 20, "total": 132 },
  "links": {
    "self": "/api/v1/users?page=1&pageSize=20&sort=-createdAt&fields=id,name,email",
    "next": "/api/v1/users?page=2&pageSize=20&sort=-createdAt&fields=id,name,email"
  }
}
```

#### 부분 업데이트(PATCH, Merge Patch)
```http
PATCH /api/v1/users/101
Content-Type: application/merge-patch+json
If-Match: "v1-67fd3a"

{ "name": "Alice Kim" }
```
응답:
```http
HTTP/1.1 204 No Content
ETag: "v2-1f9c0b"
```

#### 삭제(멱등)
```http
DELETE /api/v1/users/101
```
```http
HTTP/1.1 204 No Content
```

#### 오류(유효성)
```http
POST /api/v1/users
Content-Type: application/json

{ "name": "", "email": "bad-email" }
```
```json
{
  "type": "https://api.example.com/problems/validation-error",
  "title": "One or more validation errors occurred.",
  "status": 400,
  "errors": {
    "name": ["Required"],
    "email": ["Invalid email format"]
  },
  "traceId": "00-8c3c..."
}
```

### 16.2 ASP.NET Core 컨트롤러 스니펫

```csharp
[ApiController]
[Route("api/v1/users")]
[Produces("application/json")]
public class UsersController : ControllerBase
{
    private readonly IUserService _svc;
    public UsersController(IUserService svc) => _svc = svc;

    [HttpGet]
    [ProducesResponseType(typeof(Paged<UserReadDto>), StatusCodes.Status200OK)]
    public IActionResult GetAll([FromQuery] int page = 1, [FromQuery] int pageSize = 20,
                                [FromQuery] string? sort = null, [FromQuery] string? fields = null)
        => Ok(_svc.GetUsers(page, pageSize, sort, fields));

    [HttpGet("{id:int}", Name = "users.get")]
    [ProducesResponseType(typeof(UserReadDto), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public IActionResult GetById(int id)
        => _svc.Find(id) is { } u ? Ok(u) : NotFound();

    [HttpPost]
    [Consumes("application/json")]
    [ProducesResponseType(typeof(object), StatusCodes.Status201Created)]
    [ProducesResponseType(StatusCodes.Status400BadRequest)]
    public IActionResult Create([FromBody] UserCreateDto dto)
    {
        // [ApiController] 자동 유효성 검사
        var (id, etag) = _svc.Create(dto);
        Response.Headers.ETag = etag;
        return CreatedAtRoute("users.get", new { id }, new { id });
    }

    [HttpPatch("{id:int}")]
    [Consumes("application/merge-patch+json", "application/json")]
    [ProducesResponseType(StatusCodes.Status204NoContent)]
    [ProducesResponseType(StatusCodes.Status412PreconditionFailed)]
    public IActionResult Patch(int id, [FromBody] JsonElement patchDoc, [FromHeader(Name = "If-Match")] string? ifMatch)
    {
        // 낙관적 동시성
        if (!_svc.TryApplyMergePatch(id, patchDoc, ifMatch, out var newEtag, out var preconditionFailed))
            return preconditionFailed ? StatusCode(StatusCodes.Status412PreconditionFailed) : NotFound();

        Response.Headers.ETag = newEtag!;
        return NoContent();
    }

    [HttpDelete("{id:int}")]
    [ProducesResponseType(StatusCodes.Status204NoContent)]
    public IActionResult Delete(int id)
    {
        _svc.Delete(id); // 멱등: 이미 없어도 204
        return NoContent();
    }
}
```

### 16.3 서비스 구현 포인트(의사코드)

```csharp
public (int id, string etag) Create(UserCreateDto dto)
{
    var entity = _mapper.Map<User>(dto);
    _db.Users.Add(entity);
    _db.SaveChanges();
    var etag = ETag.Of(entity.Version); // RowVersion → ETag
    return (entity.Id, etag);
}

public bool TryApplyMergePatch(int id, JsonElement patch, string? ifMatch, out string? newEtag, out bool preconditionFailed)
{
    newEtag = null; preconditionFailed = false;
    var entity = _db.Users.Find(id);
    if (entity is null) return false;

    var currentEtag = ETag.Of(entity.Version);
    if (ifMatch != null && ifMatch != currentEtag)
    {
        preconditionFailed = true;
        return false;
    }

    // patch(JSON Merge Patch) → DTO 병합 → 검증 → 매핑
    var dto = _mapper.Map<UserUpdateDto>(entity);
    dto = JsonMergePatch.Apply(dto, patch);
    Validate(dto);

    _mapper.Map(dto, entity);
    _db.SaveChanges();

    newEtag = ETag.Of(entity.Version);
    return true;
}
```

---

## 부록 A) 표준 헤더 예시 모음

- **Location**: 새로 생성된 리소스 경로, 작업 조회 경로
- **ETag**: 엔터티 태그(버전), 강/약 식별자(`W/`)
- **If-Match / If-None-Match**: 사전조건(갱신/캐시 재검사)
- **Cache-Control**: `public|private, max-age=60, no-store`
- **Retry-After**: 202/429에서 재시도 힌트
- **RateLimit-Limit / RateLimit-Remaining / RateLimit-Reset**: 요청 제한

---

## 부록 B) 설계 원칙 요약 체크리스트

- [ ] **URI**: 명사·복수형·계층 간결성 유지, 동사 금지
- [ ] **메서드语义**: GET/POST/PUT/PATCH/DELETE의 의미를 지키기
- [ ] **상태코드**: 시나리오와 일관성 있는 코드 사용
- [ ] **오류 포맷**: ProblemDetails 일관화
- [ ] **페이징/정렬/필터**: 쿼리 파라미터 계약 고정
- [ ] **멱등성**: Idempotency-Key(POST), PUT/DELETE 준수
- [ ] **동시성**: ETag + If-Match, 412 처리
- [ ] **캐싱**: ETag/Last-Modified/Cache-Control 설계
- [ ] **보안**: HTTPS, JWT/OAuth2, CORS, 입력 검증
- [ ] **버전**: `/v1`부터, Deprecation/Sunset 안내
- [ ] **문서화**: OpenAPI, 예제/스키마/상태코드 완비
- [ ] **테스트**: 계약/회귀/부하, 관측성(로그/메트릭/트레이스)

---

## 마무리

RESTful은 **리소스 중심 설계**와 **HTTP 의미의 정확한 사용**이 핵심이다.
본 문서의 원칙(URI·상태코드·오류·부분 업데이트·멱등성·캐싱·보안·버전·문서화)을 **일관되게 적용**하면,
**확장성·호환성·운영성**을 모두 갖춘 API를 구축할 수 있다.
