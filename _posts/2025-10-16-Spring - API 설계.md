---
layout: post
title: Spring - API 설계
date: 2025-10-16 14:25:23 +0900
category: Spring
---
# API 설계 실전 — REST 설계 가이드, HATEOAS/문서화, 파일 업·다운로드/대용량 스트리밍

> 목표: 실무에서 **예측 가능하고 확장 가능한 HTTP API**를 설계/구현/문서화하고, **대용량 파일 전송**까지 안전하게 다루는 방법을 예제로 정리한다.
> 환경 가정: Spring Boot 3.3+, Java 21, Spring Web MVC, springdoc-openapi 또는 Spring REST Docs, JPA(Optional), S3/로컬 스토리지.

---

## A. REST 설계 가이드 — 리소스, 경로, 상태코드, 에러 모델

### A-1. 리소스와 경로 규칙

- **명사형 복수 리소스**: `/users`, `/orders`, `/files`
- **리소스 식별자**: `/users/{userId}`
- **하위 리소스**: `/orders/{id}/items`, `/users/{id}/addresses`
- **검색/필터**: `/orders?status=PAID&from=2025-01-01&to=2025-01-31`
- **컬렉션 조작**: `POST /orders`, `GET /orders`, `DELETE /orders/{id}`
- **액션성 동사**는 **하위 리소스**나 **상태 전이**로 모델링
  - 결제: `POST /orders/{id}/payment`
  - 상태변경: `PATCH /orders/{id}` with body `{ "status": "CANCELED" }`

**버전 전략(선택)**
- 경로 버전: `/v1/orders` (가장 단순)
- 미디어 타입 버전: `Accept: application/vnd.example.v1+json` (HATEOAS 친화)
- 헤더 버전: `X-API-Version: 1` (권장도 낮음)

> **일관성**이 최우선. 팀 룰을 문서화하고 자동화(스캐폴드/검사)하자.

---

### A-2. 상태코드 매핑 표 (요약)

| 동작 | 성공 코드 | 실패 예시 |
|---|---|---|
| 생성(POST) | **201 Created** + `Location` 헤더 | 400(검증), 409(충돌), 415(미디어 타입) |
| 단건 조회(GET) | **200 OK** | 404(없음), 406(협상 실패) |
| 목록 조회(GET) | **200 OK** | 400(필터 오류) |
| 전체 교체(PUT) | **200 OK** or 204 | 400/404/409 |
| 부분 수정(PATCH) | **200 OK** or 204 | 400/404/409 |
| 삭제(DELETE) | **204 No Content** | 404/409 |
| 비동기 수락 | **202 Accepted** | 409 등 |

응답 바디는 **문제 상세**가 필요할 때 `application/problem+json`(RFC 7807)을 권장.

---

### 표준화

**모델**
```java
public record ApiProblem(
  String type,     // URI 혹은 about:blank
  String title,    // 짧은 요약
  int status,      // HTTP 상태 코드
  String detail,   // 사람 친화적 설명(로깅용은 금지)
  String instance, // 요청 식별(예: 요청 경로 혹은 요청 ID)
  Map<String, Object> errors // 필드 검증 등 추가 정보
) {
  public static ApiProblem of(HttpStatus s, String title, String detail, Map<String, Object> errors, String instance) {
    return new ApiProblem("about:blank", title, s.value(), detail, instance, errors == null ? Map.of() : errors);
  }
}
```

**전역 예외 처리기**
```java
@RestControllerAdvice
class GlobalErrors {

  @ExceptionHandler(MethodArgumentNotValidException.class)
  @ResponseStatus(HttpStatus.BAD_REQUEST)
  ApiProblem validation(MethodArgumentNotValidException ex, HttpServletRequest req) {
    var fields = ex.getBindingResult().getFieldErrors().stream()
      .collect(Collectors.groupingBy(FieldError::getField,
        Collectors.mapping(DefaultMessageSourceResolvable::getDefaultMessage, Collectors.toList())));
    return ApiProblem.of(HttpStatus.BAD_REQUEST, "Validation failed", "Request body validation failed",
                         Map.of("fields", fields), req.getRequestURI());
  }

  @ExceptionHandler(HttpMessageNotReadableException.class)
  @ResponseStatus(HttpStatus.BAD_REQUEST)
  ApiProblem badJson(HttpMessageNotReadableException ex, HttpServletRequest req) {
    return ApiProblem.of(HttpStatus.BAD_REQUEST, "Malformed JSON",
                         Optional.ofNullable(ex.getMostSpecificCause()).map(Throwable::getMessage).orElse("Invalid JSON"),
                         Map.of(), req.getRequestURI());
  }

  @ExceptionHandler(EntityNotFoundException.class)
  @ResponseStatus(HttpStatus.NOT_FOUND)
  ApiProblem notFound(EntityNotFoundException ex, HttpServletRequest req) {
    return ApiProblem.of(HttpStatus.NOT_FOUND, "Not Found", ex.getMessage(), Map.of(), req.getRequestURI());
  }

  @ExceptionHandler(AccessDeniedException.class)
  @ResponseStatus(HttpStatus.FORBIDDEN)
  ApiProblem forbidden(AccessDeniedException ex, HttpServletRequest req) {
    return ApiProblem.of(HttpStatus.FORBIDDEN, "Forbidden", ex.getMessage(), Map.of(), req.getRequestURI());
  }

  @ExceptionHandler(Exception.class)
  @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
  ApiProblem any(Exception ex, HttpServletRequest req) {
    return ApiProblem.of(HttpStatus.INTERNAL_SERVER_ERROR, "Internal Server Error",
                         "Unexpected error", Map.of("requestId", req.getHeader("X-Request-Id")), req.getRequestURI());
  }
}
```

---

### A-4. 페이징/정렬/필터의 표준 형태

요청:
```
GET /orders?page=0&size=20&sort=createdAt,desc&status=PAID&from=2025-01-01&to=2025-01-31
```

응답(예):
```json
{
  "data": [ { "id": 1001, "status": "PAID", "total": 9900 } ],
  "page": {
    "size": 20,
    "number": 0,
    "totalElements": 341,
    "totalPages": 18
  },
  "links": {
    "self": "/orders?page=0&size=20&sort=createdAt,desc",
    "next": "/orders?page=1&size=20&sort=createdAt,desc",
    "last": "/orders?page=17&size=20&sort=createdAt,desc"
  }
}
```

---

### A-5. 캐시/동시성: ETag, Last-Modified, If-* 헤더

- **조건부 GET**: `ETag`/`Last-Modified` 제공 → `If-None-Match`/`If-Modified-Since` 수용 → **304 Not Modified**
- **낙관적 락**: `If-Match` + ETag(혹은 엔티티 버전)로 **중복 수정 방지**
- **캐시 지시**: `Cache-Control`, `Vary`, `Expires` 헤더로 CDN/브라우저 캐시 제어

**예: 조건부 GET**
```java
@GetMapping("/products/{id}")
public ResponseEntity<ProductView> get(@PathVariable long id) {
  ProductView view = service.view(id);
  var eTag = "\"%s\"".formatted(view.version()); // 버전 기반
  return ResponseEntity.ok()
    .eTag(eTag)
    .cacheControl(CacheControl.maxAge(Duration.ofMinutes(5)).cachePublic())
    .body(view);
}
```

---

### A-6. 멱등성/안전성

- **GET/HEAD**: 안전(idempotent + 부작용 없음)
- **PUT/DELETE**: 멱등(idempotent)
- **POST**: 보통 비멱등 → **Idempotency-Key 헤더**로 재시도 안전 설계(결제 등)

---

## B. 컨트롤러 작성 패턴 — DTO, 상태코드, Location 헤더

**DTO**
```java
public record CreateOrderRequest(
  @NotNull Long memberId,
  @NotEmpty List<Line> lines
) {
  public record Line(@NotNull Long productId, @Positive int qty) {}
}

public record OrderResponse(Long id, String status, long total) { }
```

**Controller**
```java
@RestController
@RequestMapping("/api/orders")
class OrderController {

  private final OrderService service;

  OrderController(OrderService service) { this.service = service; }

  @PostMapping
  public ResponseEntity<OrderResponse> create(@Valid @RequestBody CreateOrderRequest req,
                                              UriComponentsBuilder uri) {
    var id = service.create(req); // returns new orderId
    var location = uri.path("/api/orders/{id}").buildAndExpand(id).toUri();
    return ResponseEntity.created(location)
      .body(service.get(id));
  }

  @GetMapping("/{id}")
  public OrderResponse get(@PathVariable long id) {
    return service.get(id);
  }

  @GetMapping
  public PageResponse<OrderResponse> list(@ParameterObject Pageable pageable,
                                          @RequestParam(required = false) String status) {
    return service.list(status, pageable);
  }

  @PatchMapping("/{id}")
  public OrderResponse changeStatus(@PathVariable long id, @RequestBody Map<String,String> body) {
    return service.changeStatus(id, body.get("status"));
  }

  @DeleteMapping("/{id}")
  @ResponseStatus(HttpStatus.NO_CONTENT)
  public void delete(@PathVariable long id) { service.delete(id); }
}
```

**PageResponse**
```java
public record PageResponse<T>(List<T> data, Meta page, Map<String,String> links) {
  public record Meta(int size, int number, long totalElements, int totalPages) {}
  public static <T> PageResponse<T> of(Page<T> p, String base) {
    var meta = new Meta(p.getSize(), p.getNumber(), p.getTotalElements(), p.getTotalPages());
    var links = new LinkedHashMap<String,String>();
    links.put("self", base + "?page=%d&size=%d".formatted(p.getNumber(), p.getSize()));
    if (p.hasNext()) links.put("next", base + "?page=%d&size=%d".formatted(p.getNumber()+1, p.getSize()));
    if (!p.isEmpty()) links.put("first", base + "?page=0&size=%d".formatted(p.getSize()));
    if (p.getTotalPages()>0) links.put("last", base + "?page=%d&size=%d".formatted(p.getTotalPages()-1, p.getSize()));
    return new PageResponse<>(p.getContent(), meta, links);
  }
}
```

---

## C. HATEOAS — 링크/상태 전이 힌트

Spring HATEOAS 사용 시:
```kotlin
dependencies {
  implementation("org.springframework.boot:spring-boot-starter-hateoas")
}
```

**리소스 모델**
```java
import static org.springframework.hateoas.server.mvc.WebMvcLinkBuilder.*;

public record OrderModel(Long id, String status, long total, List<Link> links) {

  public static OrderModel of(OrderResponse r) {
    var self = linkTo(methodOn(OrderController.class).get(r.id())).withSelfRel();
    var cancel = r.status().equals("CREATED")
      ? linkTo(methodOn(OrderController.class).changeStatus(r.id(), Map.of("status","CANCELED"))).withRel("cancel")
      : null;
    var links = new ArrayList<Link>();
    links.add(self);
    if (cancel!=null) links.add(cancel);
    return new OrderModel(r.id(), r.status(), r.total(), links);
  }
}
```

**응답 예**
```json
{
  "id": 1001,
  "status": "CREATED",
  "total": 9900,
  "links": [
    {"rel": "self", "href": "/api/orders/1001"},
    {"rel": "cancel", "href": "/api/orders/1001"} // PATCH with status=CANCELED
  ]
}
```

> HATEOAS는 **클라이언트가 상태 전이를 링크로 발견**할 수 있게 한다. 완전 채택이 부담스럽다면 **부분적 링크**(self/next)만 제공해도 유용하다.

---

## & Spring REST Docs

### D-1. OpenAPI (springdoc-openapi)

**의존성**
```kotlin
dependencies {
  implementation("org.springdoc:springdoc-openapi-starter-webmvc-ui:2.6.0")
}
```

기본 제공: `/v3/api-docs`, UI: `/swagger-ui.html`

**예시 애노테이션**
```java
@Tag(name="Orders", description="Order management APIs")
@RestController
@RequestMapping("/api/orders")
class OrderController {

  @Operation(summary="Create order", description="Create an order with lines")
  @ApiResponses({
    @ApiResponse(responseCode="201", description="Created",
      headers = @Header(name="Location", description="New resource URI")),
    @ApiResponse(responseCode="400", description="Bad Request")
  })
  @PostMapping
  public ResponseEntity<OrderResponse> create(@Valid @RequestBody CreateOrderRequest req,
                                              UriComponentsBuilder uri) { ... }

  @Operation(summary="Get order")
  @ApiResponse(responseCode="200", description="OK",
    content=@Content(schema=@Schema(implementation=OrderResponse.class)))
  @GetMapping("/{id}")
  public OrderResponse get(@PathVariable long id) { ... }
}
```

**스키마/예제 커스터마이즈**
```java
@Schema(name="OrderResponse", description="Order view")
public record OrderResponse(
  @Schema(description="Order ID", example="1001") Long id,
  @Schema(description="Order status", example="PAID") String status,
  @Schema(description="Total amount", example="9900") long total
) {}
```

**보안 스키마**
```java
@SecurityScheme(name="bearerAuth", type=SecuritySchemeType.HTTP, scheme="bearer", bearerFormat="JWT")
@SpringBootApplication
class App { }
```

Controller 메서드에 `@SecurityRequirement(name="bearerAuth")` 추가.

---

### D-2. Spring REST Docs (테스트 기반 스니펫 생성)

**의존성**
```kotlin
dependencies {
  testImplementation("org.springframework.restdocs:spring-restdocs-mockmvc")
}
```

**테스트**
```java
@AutoConfigureRestDocs(outputDir = "build/generated-snippets")
@AutoConfigureMockMvc
@SpringBootTest
class OrderDocsTest {

  @Autowired MockMvc mvc;
  @Autowired ObjectMapper om;

  @Test
  void create_order_docs() throws Exception {
    var req = Map.of("memberId",1, "lines", List.of(Map.of("productId",99,"qty",2)));
    mvc.perform(post("/api/orders").contentType(APPLICATION_JSON).content(om.writeValueAsString(req)))
      .andExpect(status().isCreated())
      .andDo(document("orders-create",
        requestFields(
          fieldWithPath("memberId").description("Member ID"),
          fieldWithPath("lines[].productId").description("Product ID"),
          fieldWithPath("lines[].qty").description("Quantity")
        ),
        responseFields(
          fieldWithPath("id").description("Order ID"),
          fieldWithPath("status").description("Order status"),
          fieldWithPath("total").description("Total amount")
        )));
  }
}
```

템플릿(Asciidoctor)로 HTML 문서 생성 → **빌드 결과물**로 게시.

> OpenAPI는 **실시간 탐색과 계약 공유**에, REST Docs는 **테스트 증거 기반 문서**에 강점. 함께 쓰기도 많다.

---

## E. 파일 업로드/다운로드, 대용량 스트리밍

### E-1. 멀티파트 업로드 — 단건/다건, 검증, 제한

**설정**
```yaml
spring:
  servlet:
    multipart:
      max-file-size: 50MB
      max-request-size: 60MB
```

**컨트롤러**
```java
@PostMapping(path="/files", consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
public ResponseEntity<Map<String,Object>> upload(
  @RequestPart("file") MultipartFile file,
  @RequestParam(required=false) String folder,
  UriComponentsBuilder uri) throws IOException {

  if (file.isEmpty()) throw new IllegalArgumentException("Empty file");
  if (!List.of("image/png","image/jpeg","application/pdf").contains(file.getContentType()))
    throw new IllegalArgumentException("Unsupported content type");

  String id = storage.save(folder, file.getOriginalFilename(), file.getContentType(), file.getBytes());
  var location = uri.path("/files/{id}").buildAndExpand(id).toUri();

  return ResponseEntity.created(location).body(Map.of(
    "id", id, "name", file.getOriginalFilename(), "size", file.getSize(), "contentType", file.getContentType()));
}
```

**다건 업로드**
```java
@PostMapping(path="/files/bulk", consumes=MediaType.MULTIPART_FORM_DATA_VALUE)
public List<Map<String,Object>> bulk(@RequestPart("files") List<MultipartFile> files) { ... }
```

**보안/안티바이러스**
- 파일명 정규화(경로 탈출 방지), 금지 확장자 차단
- MIME 탐지(Tika 등)로 **컨텐츠 기반 판별**
- 바이러스 스캔(ClamAV/외부 DLP) 후 저장

---

### E-2. 다운로드 — `Resource` + Content-Disposition

```java
@GetMapping(value="/files/{id}")
public ResponseEntity<Resource> download(@PathVariable String id) {
  var f = storage.load(id); // 커스텀 메타 + Resource
  var filename = URLEncoder.encode(f.originalName(), StandardCharsets.UTF_8).replace("+","%20");
  return ResponseEntity.ok()
    .contentType(MediaType.parseMediaType(f.contentType()))
    .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename*=UTF-8''" + filename)
    .cacheControl(CacheControl.noCache())
    .body(f.resource()); // FileSystemResource/InputStreamResource/S3Resource
}
```

> `FileSystemResource`는 대용량도 OS 커널 버퍼와 스트리밍된다. S3는 `S3ObjectInputStream` → `InputStreamResource`.

---

### — HTTP Range

**구현 개요**
- 요청 헤더: `Range: bytes=START-END`
- 응답: **206 Partial Content**, 헤더 `Content-Range`, `Accept-Ranges: bytes`
- Spring MVC: `ResourceRegion` + `HttpRange` 지원

**예시**
```java
@GetMapping(value="/files/{id}/stream")
public ResponseEntity<?> stream(@PathVariable String id, @RequestHeader HttpHeaders headers) throws IOException {
  var f = storage.load(id);
  var resource = f.resource(); // Resource with length
  List<HttpRange> ranges = headers.getRange();
  if (ranges == null || ranges.isEmpty()) {
    return ResponseEntity.ok()
      .contentType(MediaType.parseMediaType(f.contentType()))
      .contentLength(resource.contentLength())
      .header("Accept-Ranges","bytes")
      .body(resource);
  }
  // 단일 범위만 처리(복수 범위는 multipart/byteranges 필요)
  HttpRange range = ranges.get(0);
  long length = resource.contentLength();
  ResourceRegion region = range.toResourceRegion(resource, length);
  return ResponseEntity.status(HttpStatus.PARTIAL_CONTENT)
    .contentType(MediaType.parseMediaType(f.contentType()))
    .header("Accept-Ranges","bytes")
    .header(HttpHeaders.CONTENT_RANGE,
      "bytes %d-%d/%d".formatted(region.getPosition(), region.getPosition()+region.getCount()-1, length))
    .body(region);
}
```

---

### E-4. 대용량 스트리밍 — 메모리 압박 피하기

- **InputStreamResource** 또는 **FileSystemResource** 사용(메모리 로드 금지)
- **`ResponseBodyEmitter`/`SseEmitter`**로 서버 푸시/실시간 스트림
- **Zero-copy**(톰캣/네티 등 전송 최적화)는 컨테이너/서버 설정에 따름

**주기적 청크 쓰기 예**
```java
@GetMapping(value="/reports/live", produces=MediaType.TEXT_EVENT_STREAM_VALUE)
public SseEmitter live() {
  SseEmitter emitter = new SseEmitter(0L); // timeout infinite
  Executors.newSingleThreadExecutor().submit(() -> {
    try {
      for (int i=1;i<=100;i++) {
        emitter.send(SseEmitter.event().id(String.valueOf(i)).data(Map.of("progress", i)));
        Thread.sleep(50);
      }
      emitter.complete();
    } catch (Exception e) { emitter.completeWithError(e); }
  });
  return emitter;
}
```

---

### E-5. 무결성/재개/병렬 다운로드

- 업로드 재개: **chunked upload API**(offset + chunkSize + checksum) 설계
- 다운로드 병렬: 클라이언트가 **여러 Range**로 병렬 GET → 빠른 합치기
- 체크섬 헤더: `ETag`에 MD5(혹은 더 안전한 해시) 제공하면 검증/캐시 효율↑

---

### E-6. S3 연동 팁(요약)

- presigned URL을 발급해 **클라이언트 → S3 직접 업/다운** (서버는 인증/권한만)
- 서버 스트리밍은 비용↑, 네트워크 병목 고려
- CDN 앞단(CloudFront) + `Cache-Control`/`ETag`로 효율

---

## F. 보안/성능/운영 실무 체크리스트

### F-1. 보안

- 인증/인가: Spring Security + JWT/OAuth2
- 입력 검증: Bean Validation + JSON strict 모드(`FAIL_ON_UNKNOWN_PROPERTIES=true`)
- CORS: Origin 화이트리스트 + 자격증명 쿠키 전략
- HTTPS/TLS 강제, HSTS, Content Security Policy(프런트 API 통합 시)
- 헤더 강화: `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Referrer-Policy` 등
- 민감 데이터 마스킹(로그/에러)

### F-2. 성능

- 압축: `server.compression.enabled=true` (큰 JSON/CSV)
- 페이징 상한: `maxSize` 정책(예: 100)
- DTO 프로젝션: 필요한 필드만
- 캐싱: ETag/Cache-Control, 레디스 캐시
- DB N+1 차단: fetch join/배치 페치

### F-3. 관측성

- **요청 ID(MDC)** 주입 + 로깅 포맷 JSON
- Actuator: `/actuator/health`, `/actuator/metrics`, `/actuator/httptrace`(대체는 Micrometer)
- 분산 트레이싱: OpenTelemetry → traceId를 응답 헤더로 노출(예: `X-Trace-Id`)

---

## G. 끝에서 끝까지 예시 — “파일 + 주문” API 스니펫 모음

### G-1. Idempotency-Key로 멱등 POST

```java
@PostMapping("/payments")
public ResponseEntity<?> pay(@RequestHeader("Idempotency-Key") String key,
                             @Valid @RequestBody PaymentRequest req) {
  var result = paymentService.processOnce(key, req);
  return switch (result.status()) {
    case CREATED -> ResponseEntity.status(201).body(result);
    case EXISTS  -> ResponseEntity.status(200).body(result);
  };
}
```

### G-2. 조건부 업데이트(If-Match)

```java
@PutMapping("/products/{id}")
public ResponseEntity<ProductView> replace(@PathVariable long id,
    @RequestHeader(value="If-Match", required=false) String eTag,
    @Valid @RequestBody ProductUpsert req) {
  var pv = service.replace(id, req, eTag); // 내부에서 ETag/버전 검증
  return ResponseEntity.ok().eTag("\"%s\"".formatted(pv.version())).body(pv);
}
```

### 샘플

```json
{
  "type": "about:blank",
  "title": "Validation failed",
  "status": 400,
  "detail": "Request body validation failed",
  "instance": "/api/orders",
  "errors": { "fields": { "memberId": ["must not be null"] } }
}
```

---

## H. 테스트(계약/문서/부하)

- **단위/슬라이스**: `@WebMvcTest`로 컨트롤러/에러 처리 검증
- **통합**: `@SpringBootTest` + Testcontainers(DB/MinIO)
- **계약 테스트**: OpenAPI 스키마 검증(예: swagger-request-validator)
- **부하/대용량**: Gatling/k6/JMeter — Range/Chunk/대용량 파일 시나리오 포함

---

## I. 요약(한 페이지)

- **REST 규칙**: 명사형 경로, 일관된 상태코드, 표준 에러(Problem+JSON), 페이징/정렬 규약.
- **HATEOAS/문서화**: 링크로 상태 전이 힌트, OpenAPI/REST Docs로 문서 자동화.
- **파일 처리**: 멀티파트 업로드, 안전한 다운로드, **Range 206** 재개, **스트리밍**으로 메모리 절약.
- **운영**: 캐시/ETag, 멱등성 키, 보안 헤더, 관측성을 갖춘 **예측 가능**한 API.
