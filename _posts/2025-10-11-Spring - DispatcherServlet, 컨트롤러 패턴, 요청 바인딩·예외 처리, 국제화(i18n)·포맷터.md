---
layout: post
title: Spring - DispatcherServlet, 컨트롤러 패턴, 요청 바인딩·예외 처리, 국제화(i18n)·포맷터
date: 2025-10-11 21:25:23 +0900
category: Spring
---
# 웹 MVC 기본기 — DispatcherServlet, 컨트롤러 패턴, 요청 바인딩·예외 처리, 국제화(i18n)·포맷터

> 목표: **“요청 → 컨트롤러 → 응답”** 흐름을 정확히 이해하고, **검증·예외·국제화**를 표준화하여 **예측 가능**한 API를 만든다.
> 환경 가정: Spring Boot 3.3+, Java 21, Gradle, JSON API.

---

## A. DispatcherServlet 동작 흐름(요청 수명주기)

### A-1. 큰 그림: 필터 → 서블릿 → 핸들러 매핑 → 핸들러 어댑터 → 컨트롤러 → 뷰/메시지컨버터

1. **톰캣/서블릿 컨테이너**가 요청을 받음 → **Filter** 체인 통과(`OncePerRequestFilter`, `SecurityFilterChain` 등)
2. **`DispatcherServlet`** 진입
3. **HandlerMapping**이 요청 경로/메서드로 **핸들러(컨트롤러 메서드)** 탐색
   - `RequestMappingHandlerMapping`(애노테이션 기반), `SimpleUrlHandlerMapping` 등
4. **HandlerAdapter**가 해당 핸들러 호출 방법 결정
   - `RequestMappingHandlerAdapter`가 **인자 바인딩**(변환·검증), **리턴값 처리**
5. **Controller** 호출 → 반환
6. 반환이 **객체**이면 **HttpMessageConverter**가 JSON/타입 변환, **문자열/뷰**면 ViewResolver 경유
7. 응답 헤더/본문 전송

> JSON API는 보통 **뷰 리졸버 없이**, `MappingJackson2HttpMessageConverter`가 POJO ↔ JSON 변환을 담당.

---

### 핵심

- 기본은 `Accept` 헤더로 결정(JSON이 일반적).
- 확장자 기반 협상은 비권장(`.json`, `.xml` URI).
- `produces`, `consumes`로 메서드별 명시 가능.

```java
@GetMapping(value="/items/{id}", produces="application/json")
public ItemDto get(@PathVariable long id) { /* ... */ }
```

---

### A-3. HandlerMethodArgumentResolver(핵심 훅)

- 컨트롤러 **메서드 인자**에 값을 주입하는 확장 포인트.
- 스프링이 제공: `@RequestParam`, `@PathVariable`, `@RequestHeader`, `@CookieValue`, `Principal`, `Pageable`, `@RequestBody` 등.
- 커스텀 사용자 정보 주입기(예: JWT에서 userId 꺼내기)도 구현 가능.

---

### A-4. Interceptor vs Filter (어디서 무엇을?)

- **Filter**: 서블릿 스펙, 시큐리티·XSS 방어·로깅 헤더 등 **아주 앞단**.
- **HandlerInterceptor**: 핸들러(컨트롤러) 앞/뒤, `preHandle/postHandle/afterCompletion`. **로깅/트레이싱/Locale** 제어에 적합.

```java
@Configuration
class WebMvcConfig implements WebMvcConfigurer {
  @Override public void addInterceptors(InterceptorRegistry r) {
    r.addInterceptor(new MdcInterceptor()).addPathPatterns("/**");
  }
}
```

---

## B. 컨트롤러 작성 패턴 — `@RestController`, Validation, 설계 지침

### B-1. `@RestController` 기본 패턴

```java
@RestController
@RequestMapping("/api/users")
class UserController {

  private final UserService service;
  UserController(UserService service){ this.service = service; }

  @PostMapping
  @ResponseStatus(HttpStatus.CREATED)
  public UserResponse create(@Valid @RequestBody CreateUserRequest req) {
    var u = service.create(req.email(), req.name());
    return UserResponse.from(u);
  }

  @GetMapping("/{id}")
  public UserResponse get(@PathVariable long id) {
    return UserResponse.from(service.get(id));
  }

  @GetMapping
  public PageResponse<UserResponse> list(@RequestParam(defaultValue="0") int page,
                                         @RequestParam(defaultValue="20") int size) {
    return PageResponse.of(service.list(page, size).map(UserResponse::from));
  }
}
```

- `@RestController` = `@Controller + @ResponseBody`(메서드 반환을 그대로 바디로).
- **상태코드**는 `@ResponseStatus`나 `ResponseEntity`로 명시.

---

### B-2. 요청 DTO, 응답 DTO, 도메인 분리

- 컨트롤러는 **DTO(입출력)**만. 도메인 엔티티를 직접 노출하지 않는다.
- 응답 표준화: `data`, `error`, `meta` 등 일관 구조.

```java
public record CreateUserRequest(
  @Email @NotBlank String email,
  @NotBlank @Size(min=2, max=30) String name
) {}
public record UserResponse(long id, String email, String name) {
  public static UserResponse from(User u){ return new UserResponse(u.getId(), u.getEmail(), u.getName()); }
}
```

---

### B-3. 파라미터 바인딩(요약 표)

| 애노테이션 | 용도 | 예 |
|---|---|---|
| `@PathVariable` | 경로 변수 | `/users/{id}` |
| `@RequestParam` | 쿼리/폼 | `?page=0&size=20` |
| `@RequestHeader` | 요청 헤더 | `@RequestHeader("X-Req-Id") String id` |
| `@CookieValue` | 쿠키 | `@CookieValue("sid") String sid` |
| `@RequestBody` | JSON/XML → 객체 | `@RequestBody @Valid CreateUserRequest` |
| `@ModelAttribute` | 폼/쿼리 바인딩(객체) | 검색 폼 DTO |
| `@Validated` | 타입 레벨 검증 그룹 | `@Validated(Create.class)` |

**주의**: `@RequestBody`는 **JSON 파싱 실패** 시 `HttpMessageNotReadableException`이 발생 → 예외 처리 필요.

---

### B-4. `ResponseEntity`와 헤더·상태 제어

```java
@PostMapping
public ResponseEntity<UserResponse> create(@Valid @RequestBody CreateUserRequest req, UriComponentsBuilder uri) {
  var id = service.create(req.email(), req.name()).getId();
  var location = uri.path("/api/users/{id}").buildAndExpand(id).toUri();
  return ResponseEntity.created(location).body(new UserResponse(id, req.email(), req.name()));
}
```

---

### B-5. Validation(Bean Validation + 그룹)

```java
public interface OnCreate {}
public interface OnUpdate {}

public record UpdateUserRequest(
  @NotNull(groups = OnUpdate.class) Long id,
  @NotBlank(groups = {OnCreate.class, OnUpdate.class}) @Size(max=30) String name
) {}
```

```java
@PatchMapping("/{id}")
public UserResponse update(@PathVariable long id,
                           @Validated(OnUpdate.class) @RequestBody UpdateUserRequest req) {
  return UserResponse.from(service.update(id, req.name()));
}
```

- 검증 실패 시 `MethodArgumentNotValidException` → 글로벌 예외 처리로 **일관 에러 포맷** 리턴.

---

### B-6. 파일 업로드/다운로드(스트리밍)

```java
@PostMapping(path="/upload", consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
public Map<String,Object> upload(@RequestPart("file") MultipartFile file) throws IOException {
  // 용량/확장자/바이러스 스캔 등 검증
  var size = file.getSize();
  return Map.of("name", file.getOriginalFilename(), "size", size);
}

@GetMapping(value="/files/{id}", produces=MediaType.APPLICATION_OCTET_STREAM_VALUE)
public ResponseEntity<Resource> download(@PathVariable long id) {
  var resource = storage.getAsResource(id); // FileSystemResource/S3Resource…
  return ResponseEntity.ok()
      .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=\""+resource.getFilename()+"\"")
      .body(resource);
}
```

- **대용량**: `resources.setUseFileChannels(true)`(NIO), `Stream`/`InputStreamResource` 활용.

---

## C. 요청 바인딩 & 예외 처리 — `@ControllerAdvice`, Error Handling 표준

### C-1. 에러 응답 표준 정의(예: RFC 7807 스타일)

- **권장**: `ProblemDetails`(Spring 6 도입) 또는 **유사 구조**.

```java
public record ApiError(String type, String title, int status, String detail, String instance, Map<String, Object> errors) {
  public static ApiError of(HttpStatus status, String title, String detail, Map<String,Object> errors) {
    return new ApiError("about:blank", title, status.value(), detail, null, errors);
  }
}
```

---

### C-2. 글로벌 예외 처리기 — `@RestControllerAdvice`

```java
@RestControllerAdvice
class GlobalExceptionHandler {

  @ExceptionHandler(MethodArgumentNotValidException.class)
  @ResponseStatus(HttpStatus.BAD_REQUEST)
  public ApiError handleValidation(MethodArgumentNotValidException ex, HttpServletRequest req) {
    Map<String,Object> fieldErrors = ex.getBindingResult().getFieldErrors().stream()
        .collect(Collectors.groupingBy(FieldError::getField,
            Collectors.mapping(DefaultMessageSourceResolvable::getDefaultMessage, Collectors.toList())));
    return ApiError.of(HttpStatus.BAD_REQUEST, "Validation failed", "Invalid request body", Map.of("fields", fieldErrors));
  }

  @ExceptionHandler(HttpMessageNotReadableException.class)
  @ResponseStatus(HttpStatus.BAD_REQUEST)
  public ApiError handleParse(HttpMessageNotReadableException ex) {
    return ApiError.of(HttpStatus.BAD_REQUEST, "Malformed JSON", ex.getMostSpecificCause().getMessage(), Map.of());
  }

  @ExceptionHandler(EntityNotFoundException.class)
  @ResponseStatus(HttpStatus.NOT_FOUND)
  public ApiError notFound(EntityNotFoundException ex) {
    return ApiError.of(HttpStatus.NOT_FOUND, "Not Found", ex.getMessage(), Map.of());
  }

  @ExceptionHandler(AccessDeniedException.class)
  @ResponseStatus(HttpStatus.FORBIDDEN)
  public ApiError forbidden(AccessDeniedException ex) {
    return ApiError.of(HttpStatus.FORBIDDEN, "Forbidden", ex.getMessage(), Map.of());
  }

  @ExceptionHandler(Exception.class)
  @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
  public ApiError handleAny(Exception ex, HttpServletRequest req) {
    return ApiError.of(HttpStatus.INTERNAL_SERVER_ERROR, "Internal error", "Unexpected error", Map.of("requestId", req.getHeader("X-Request-Id")));
  }
}
```

- **장점**: 컨트롤러 전체에서 **통일된 에러 포맷** 유지.
- **Tip**: 운영 환경에서는 스택트레이스/내부 메시지를 숨기고, **요청 ID**로 추적 가능하게.

---

### C-3. BindingResult로 **부분 실패** 처리(선택)

```java
@PostMapping("/bulk")
public Map<String,Object> bulk(@RequestBody List<@Valid CreateUserRequest> reqs, BindingResult binding) {
  if (binding.hasErrors()) {
    // 부분 실패 세부 응답 설계(배치 API에 유용)
  }
  // …
  return Map.of("count", reqs.size());
}
```

---

### C-4. 도메인/서비스 예외 → API 에러 매핑

- 도메인 계층에서 `DuplicateEmailException`, `InsufficientBalance` 등 **의미 있는 예외**를 던지고,
- `@ControllerAdvice`에서 **HTTP 상태**로 매핑.

```java
@ExceptionHandler(DuplicateEmailException.class)
@ResponseStatus(HttpStatus.CONFLICT)
ApiError duplicate(DuplicateEmailException ex) {
  return ApiError.of(HttpStatus.CONFLICT, "Email exists", ex.getMessage(), Map.of("email", ex.getEmail()));
}
```

---

### C-5. 예외 발생 시 트랜잭션

- `@Transactional` 메서드 안에서 **런타임 예외** 발생 시 **롤백**.
- API 레벨에서 적절한 상태코드와 메시지로 변환.

---

## & 포맷터/메시지 — `MessageSource`, Locale, Formatter

### D-1. MessageSource 설정(부트 기본)

- 부트는 `messages.properties`(기본), `messages_ko.properties`, `messages_en.properties` 자동 로드.

```yaml
# application.yml

spring:
  messages:
    basename: messages,errors          # 여러 파일 접두어 가능
    fallback-to-system-locale: false
```

`src/main/resources/messages.properties`
```
greeting=Hello, {0}!
user.created=User {0} is created.
```
`messages_ko.properties`
```
greeting=안녕하세요, {0}님!
user.created=사용자 {0}이(가) 생성되었습니다.
```

---

### D-2. Locale 결정 — `LocaleResolver` + `LocaleChangeInterceptor`

- 기본은 `Accept-Language` 헤더.
- **쿼리 파라미터**로 언어 바꾸기: `?lang=ko`

```java
@Configuration
class I18nConfig implements WebMvcConfigurer {
  @Bean LocaleResolver localeResolver() { return new AcceptHeaderLocaleResolver(); }

  @Override public void addInterceptors(InterceptorRegistry registry) {
    var i = new LocaleChangeInterceptor();
    i.setParamName("lang");
    registry.addInterceptor(i);
  }
}
```

**사용**
```java
@RestController
class GreetingController {
  private final MessageSource ms;
  GreetingController(MessageSource ms){ this.ms = ms; }

  @GetMapping("/greet")
  public Map<String,String> greet(@RequestParam(defaultValue="world") String name, Locale locale) {
    var msg = ms.getMessage("greeting", new Object[]{name}, locale);
    return Map.of("message", msg);
  }
}
```

- `GET /greet?name=Kim&lang=ko` → `안녕하세요, Kim님!`
- `GET /greet?name=Kim` with `Accept-Language: en` → `Hello, Kim!`

---

### D-3. 포맷터/컨버터 — 도메인 타입 바인딩

- **ConversionService**가 문자열 ↔ 타입 변환.
- 날짜/숫자/자체 타입에 **Formatter/Converter** 등록.

```java
// 도메인 ID 타입
public record UserId(long value){}

// "123" → UserId(123)
@Component
class UserIdConverter implements Converter<String, UserId> {
  @Override public UserId convert(String s) { return new UserId(Long.parseLong(s)); }
}

@RestController
@RequestMapping("/api/u")
class UserQueryController {
  @GetMapping("/{id}")
  public Map<String,Object> get(@PathVariable UserId id){
    return Map.of("id", id.value());
  }
}
```

**날짜/숫자 포맷**
```java
public record ReportQuery(
  @DateTimeFormat(iso = DateTimeFormat.ISO.DATE) LocalDate from,
  @DateTimeFormat(iso = DateTimeFormat.ISO.DATE) LocalDate to,
  @NumberFormat(style = NumberFormat.Style.CURRENCY) BigDecimal minAmount
) {}

@GetMapping("/report")
public ReportQuery q(@Valid @ModelAttribute ReportQuery q){ return q; }
```

---

### D-4. 검증 메시지 국제화

- Bean Validation 메시지를 `messages_*.properties`에서 제공.

`messages.properties`
```
javax.validation.constraints.NotBlank.message=must not be blank
size.name=length must be between {min} and {max}
```
`messages_ko.properties`
```
javax.validation.constraints.NotBlank.message=값을 비워둘 수 없습니다
size.name=길이는 {min}~{max}여야 합니다
```

```java
public record CreateUserRequest(
  @NotBlank(message="{javax.validation.constraints.NotBlank.message}") String email,
  @Size(min=2, max=30, message="{size.name}") String name
) {}
```

---

## E. 통합 예제 — “사용자 관리” 미니 API (검증·예외·i18n·포맷터 적용)

### E-1. 도메인/서비스(요약)

```java
@Entity @Table(name="users")
class User {
  @Id @GeneratedValue Long id;
  @Column(nullable=false, unique=true) String email;
  @Column(nullable=false) String name;
  // getter/setter/생성 메서드…
}

interface UserRepository extends JpaRepository<User, Long> {
  boolean existsByEmail(String email);
}

@Service
class UserService {
  private final UserRepository repo;
  UserService(UserRepository repo){ this.repo = repo; }

  @Transactional
  public User create(String email, String name) {
    if (repo.existsByEmail(email)) throw new DuplicateEmailException(email);
    var u = new User(); u.email = email; u.name = name;
    return repo.save(u);
  }
  @Transactional(readOnly = true)
  public User get(long id) { return repo.findById(id).orElseThrow(() -> new EntityNotFoundException("user")); }
}
class DuplicateEmailException extends RuntimeException {
  private final String email; public DuplicateEmailException(String e){ this.email=e; }
  public String getEmail(){ return email; }
}
```

### E-2. API + i18n 메시지

```java
@RestController
@RequestMapping("/api/users")
class UserApi {
  private final UserService service;
  private final MessageSource ms;
  UserApi(UserService s, MessageSource ms){ this.service=s; this.ms=ms; }

  @PostMapping
  @ResponseStatus(HttpStatus.CREATED)
  public UserResponse create(@Valid @RequestBody CreateUserRequest req, Locale locale) {
    var u = service.create(req.email(), req.name());
    var msg = ms.getMessage("user.created", new Object[]{u.name}, locale);
    return new UserResponse(u.id, u.email, u.name, msg);
  }

  @GetMapping("/{id}")
  public UserResponse get(@PathVariable long id) {
    var u = service.get(id);
    return new UserResponse(u.id, u.email, u.name, null);
  }
}

public record UserResponse(Long id, String email, String name, String message) {}
```

`messages.properties`
```
user.created=User {0} is created.
```
`messages_ko.properties`
```
user.created=사용자 {0}이(가) 생성되었습니다.
```

### E-3. 글로벌 예외 처리(도메인 매핑 포함)

```java
@RestControllerAdvice
class ApiErrors {
  @ExceptionHandler(DuplicateEmailException.class)
  @ResponseStatus(HttpStatus.CONFLICT)
  ApiError dup(DuplicateEmailException ex, Locale locale, MessageSource ms){
    var title = ms.getMessage("error.email.duplicate", null, "Duplicate email", locale);
    return ApiError.of(HttpStatus.CONFLICT, title, ex.getEmail(), Map.of("email", ex.getEmail()));
  }
}
```

`messages.properties`
```
error.email.duplicate=Duplicate email
```
`messages_ko.properties`
```
error.email.duplicate=이미 사용 중인 이메일입니다
```

---

## F. 테스트 — `MockMvc`로 바인딩/검증/에러 포맷 검증

```java
@SpringBootTest
@AutoConfigureMockMvc
class UserApiTest {

  @Autowired MockMvc mvc;
  @Autowired ObjectMapper om;

  @Test
  void create_ok() throws Exception {
    var req = Map.of("email","a@b.com","name","Alice");
    mvc.perform(post("/api/users")
        .contentType(MediaType.APPLICATION_JSON)
        .content(om.writeValueAsString(req))
        .header("Accept-Language","ko"))
      .andExpect(status().isCreated())
      .andExpect(jsonPath("$.message").value(startsWith("사용자")));
  }

  @Test
  void validation_error() throws Exception {
    var req = Map.of("email","","name","A");
    mvc.perform(post("/api/users")
        .contentType(MediaType.APPLICATION_JSON)
        .content(om.writeValueAsString(req)))
      .andExpect(status().isBadRequest())
      .andExpect(jsonPath("$.title").value("Validation failed"))
      .andExpect(jsonPath("$.errors.fields.name").exists());
  }
}
```

---

## G. 운영 실전 팁 & 체크리스트

### G-1. 설계 원칙

- **DTO/도메인 분리**: 컨트롤러는 DTO 변환만 담당.
- **검증은 가장자리에서**: 요청 DTO에 Bean Validation → 서비스로 들어오기 전에 실패시킴.
- **예외 매핑 통일**: `@ControllerAdvice`로 하나의 에러 포맷.
- **국제화 메시지 일원화**: 모든 사용자 메시지는 MessageSource에서 관리.
- **로깅 상관ID**: Interceptor에서 `MDC.put("traceId", …)` → 에러 응답에도 포함.

### G-2. 보안/성능

- **CORS/CSRF** 정책 명확히 (`Spring Security`).
- **페이징/정렬** 정확한 기본값과 상한 설정.
- **Jackson 보안**: `FAIL_ON_UNKNOWN_PROPERTIES`(요청 바디 엄격 모드), 민감 필드 직렬화 제외.
- **대용량 응답**: 스트리밍(`ResponseBodyEmitter`, `SseEmitter`)·gzip·캐시 헤더 활용.

### G-3. 흔한 함정

- 컨트롤러에서 엔티티 직접 반환 → 무한루프/LAZY 로딩 폭발 → **DTO로 바꾸기**.
- `@Valid` 누락 → 검증 안 됨.
- 내부 예외 메시지를 그대로 노출 → **국제화/일반화된 메시지**로 변환.
- 날짜/숫자 포맷 지역화 누락 → `@DateTimeFormat`, `@NumberFormat`, 메시지 템플릿 이용.

---

## H. 요약(한 페이지)

- **DispatcherServlet 파이프라인**을 이해하면 “왜 이 지점에서 실패하는지”가 보인다.
- **컨트롤러 패턴**: DTO 경계·검증·상태코드 명확화 → API 일관성.
- **예외 처리 표준화**: `@RestControllerAdvice`로 전역 에러 포맷, 도메인 예외 매핑.
- **i18n/포맷터**: MessageSource/Locale/Formatter로 언어·형식 일관성 유지.
