---
layout: post
title: Java - Spring, POJO·DI, 애너테이션 기반 개발
date: 2025-08-18 16:25:23 +0900
category: Java
---
# Java에서 Spring으로 넘어가기 / POJO·DI 개념 / 애너테이션 기반 개발

## 전환 개요 — 왜 Spring인가, 무엇이 달라지는가

### 왜 Spring인가

- **IoC/DI 컨테이너**가 객체 생명주기·의존성 조립을 맡는다 → **결합도↓, 테스트 용이성↑**
- **횡단 관심사**(트랜잭션, 보안, 로깅, 캐시)를 **AOP**로 선언·정책화
- **생태계**: MVC/WebFlux, Data(JPA/Redis), Security, Batch, Integration, Actuator 등

### 전환 후 달라지는 점

| 항목 | 전(코어 Java) | 후(Spring) |
|---|---|---|
| 객체 생성/조립 | `new`/팩토리 직접 호출 | **컨테이너가 주입(DI)** |
| 설정 | 하드코딩/상수 | **YAML/Env** 외부화 + 프로파일 |
| 모듈 경계 | 라이브러리 호출 | **포트/어댑터** + 인터페이스 주입 |
| 에러 처리 | 예외 전파/로깅 제각각 | **@ControllerAdvice**, AOP 로깅 표준화 |
| 트랜잭션 | 수동 관리 | **@Transactional** 선언적 관리 |
| 테스트 | 통합 중심/느림 | **슬라이스 테스트**로 빠르고 독립적 |

---

## POJO와 DI — 용어, 이점, 주입 방식 비교

### POJO(Plain Old Java Object)

특정 프레임워크에 의존하지 않는 **순수 자바 객체**. 테스트·재사용·리팩터링이 수월하다.

```java
public interface PricingPolicy {
    long price(long base, int qty);
}

public class FlatPricingPolicy implements PricingPolicy {
    @Override public long price(long base, int qty) { return base * qty; }
}
```

### IoC/DI 개념

- **IoC**: 객체 생명주기와 조립의 **제어권을 컨테이너가 가짐**
- **DI**: 필요한 의존을 **외부에서 주입** (생성자/세터/필드)

#### 주입 방식 비교

| 방식 | 장점 | 단점 | 권장 |
|---|---|---|---|
| **생성자 주입** | 불변성, 테스트 용이, 순환 의존 조기 검출 | 생성자 파라미터 증가 | **기본 선택** |
| 세터 주입 | 선택적 의존, 가독성 | 불변성 약함, NPE 리스크 | 일부 옵셔널 의존성 |
| 필드 주입 | 코드 짧음 | 테스트 어려움, 프록시/리플렉션 의존 | **지양** |

---

## 애너테이션 기반 개발 — 컴포넌트 스캔, 설정, 주입, 스코프

### 컴포넌트와 스캔

- **빈 등록**: `@Component`(범용), `@Service`, `@Repository`, `@Controller`/`@RestController`
- **스캔 루트**: `@SpringBootApplication`이 위치한 **루트 패키지 하위**가 기본 스캔 대상

```java
import org.springframework.stereotype.Service;

@Service
public class OrderService {
    private final PricingPolicy policy;
    public OrderService(PricingPolicy policy) { this.policy = policy; } // 생성자 주입
    public long order(long unit, int qty) { return policy.price(unit, qty); }
}
```

### 자바 설정과 팩토리 메서드

```java
import org.springframework.context.annotation.*;

@Configuration
class PricingConfig {
    @Bean
    PricingPolicy pricingPolicy() { return new FlatPricingPolicy(); }
}
```

### 구현 다형성과 선호도 지정

```java
public class TieredPricingPolicy implements PricingPolicy { /* ... */ }

@Configuration
class PricingConfig2 {
    @Bean @Primary
    PricingPolicy primaryPolicy() { return new FlatPricingPolicy(); }

    @Bean
    PricingPolicy tieredPolicy() { return new TieredPricingPolicy(); }
}

// 혹은 @Qualifier 사용
@Service
class AnotherService {
    private final PricingPolicy policy;
    public AnotherService(@Qualifier("tieredPolicy") PricingPolicy policy) { this.policy = policy; }
}
```

### 스코프와 지연 로딩

```java
import org.springframework.context.annotation.Scope;
import org.springframework.web.context.WebApplicationContext;

@Scope(WebApplicationContext.SCOPE_REQUEST)
@Component
class RequestScopedContext { /* 요청 단위 상태 */ }

// 싱글톤 빈이 요청 스코프 빈을 주입받을 때
@Component
class UsesRequest {
    private final ObjectProvider<RequestScopedContext> provider; // Provider/Proxy 사용
    UsesRequest(ObjectProvider<RequestScopedContext> provider) { this.provider = provider; }
    void handle() { RequestScopedContext ctx = provider.getObject(); /* ... */ }
}
```

---

## 계층 구조와 샘플 애플리케이션 — MVC + Service + Repository

### 도메인/DTO/Repository

```java
// 도메인
public record Product(Long id, String name, long unitPrice) {}

// Repository 포트(인터페이스)
public interface ProductRepository {
    Optional<Product> findById(Long id);
    Product save(Product p);
}

// 인메모리 구현(Fake) — JDBC/JPA로 교체 가능
@Component
class InMemoryProductRepository implements ProductRepository {
    private final Map<Long, Product> db = new ConcurrentHashMap<>();
    @Override public Optional<Product> findById(Long id) { return Optional.ofNullable(db.get(id)); }
    @Override public Product save(Product p) { db.put(p.id(), p); return p; }
}
```

### 서비스 + 트랜잭션 경계(읽기/쓰기 분리)

```java
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
class ProductService {
    private final ProductRepository repo;
    private final PricingPolicy pricing;
    ProductService(ProductRepository repo, PricingPolicy pricing) {
        this.repo = repo; this.pricing = pricing;
    }

    @Transactional(readOnly = true)
    public long quote(Long id, int qty) {
        Product p = repo.findById(id).orElseThrow();
        return pricing.price(p.unitPrice(), qty);
    }

    @Transactional
    public Product register(String name, long unitPrice) {
        return repo.save(new Product(ThreadLocalRandom.current().nextLong(), name, unitPrice));
    }
}
```

### 컨트롤러 + 검증

```java
import jakarta.validation.constraints.*;
import org.springframework.validation.annotation.Validated;
import org.springframework.web.bind.annotation.*;

record RegisterReq(@NotBlank String name, @Positive long unitPrice) {}
record QuoteRes(long total) {}

@RestController
@RequestMapping("/api/products")
@Validated
class ProductController {
    private final ProductService service;
    public ProductController(ProductService service) { this.service = service; }

    @PostMapping
    public Product register(@RequestBody @Validated RegisterReq req) {
        return service.register(req.name(), req.unitPrice());
    }

    @GetMapping("{id}/quote")
    public QuoteRes quote(@PathVariable Long id, @RequestParam @Positive int qty) {
        return new QuoteRes(service.quote(id, qty));
    }
}
```

---

## 트랜잭션, AOP, 검증, 예외 처리 — 실전 패턴

### 트랜잭션 규칙

- **읽기 전용**은 `@Transactional(readOnly = true)`로 플러시 최소화
- **체크 예외**는 기본 롤백 대상 아님 → `rollbackFor = Exception.class` 지정 가능
- **메서드 내부 `this` 호출**은 프록시를 거치지 않음 → **자기호출 주의**

```java
@Transactional(rollbackFor = Exception.class)
public void placeOrder(...) { /* ... */ }
```

### AOP로 횡단 관심사(실행 시간 로깅)

```java
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.*;
import org.springframework.stereotype.Component;

@Aspect
@Component
class ProfilingAspect {
    @Around("within(com.example..service..*)")
    public Object profile(ProceedingJoinPoint pjp) throws Throwable {
        long t0 = System.nanoTime();
        try { return pjp.proceed(); }
        finally { System.out.println(pjp.getSignature()+" took "+(System.nanoTime()-t0)/1_000_000+" ms"); }
    }
}
```

### 컨트롤러 예외 처리

```java
import org.springframework.http.*;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.*;

record ErrorRes(String code, String message) {}

@RestControllerAdvice
class GlobalExceptionHandler {
    @ExceptionHandler(NoSuchElementException.class)
    ResponseEntity<ErrorRes> notFound(NoSuchElementException e) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(new ErrorRes("NOT_FOUND", e.getMessage()));
    }
    @ExceptionHandler(MethodArgumentNotValidException.class)
    ResponseEntity<ErrorRes> badReq(MethodArgumentNotValidException e) {
        return ResponseEntity.badRequest().body(new ErrorRes("VALIDATION", e.getMessage()));
    }
}
```

---

## 설정 외부화와 프로파일 — 타입 안전 바인딩

### `@ConfigurationProperties` 바인딩

```yaml
# application.yml

app:
  name: demo
  pricing:
    default-policy: FLAT
```

```java
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;

@Component
@ConfigurationProperties(prefix = "app.pricing")
class PricingProps {
    private String defaultPolicy = "FLAT";
    public String getDefaultPolicy() { return defaultPolicy; }
    public void setDefaultPolicy(String v) { this.defaultPolicy = v; }
}
```

### 프로파일 분기

```yaml
# application-dev.yml

logging:
  level:
    com.example: debug

# application-prod.yml

server:
  tomcat:
    threads:
      max: 200
```

```java
import org.springframework.context.annotation.Profile;
import org.springframework.stereotype.Component;

@Profile("dev")
@Component
class DevOnlyBean { /* 개발 환경에서만 활성 */ }
```

---

## 테스트 전략 — 단위/슬라이스/통합

### 단위 테스트(순수 자바)

```java
class OrderServiceUnitTest {
    @Test
    void quote_usesPolicy() {
        ProductRepository repo = id -> Optional.of(new Product(1L, "A", 100));
        PricingPolicy policy = (base, qty) -> base * qty + 10; // Fake
        ProductService svc = new ProductService(repo, policy);
        assertEquals(310, svc.quote(1L, 3));
    }
}
```

### 슬라이스 테스트 — MVC

```java
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.test.web.servlet.MockMvc;

@WebMvcTest(ProductController.class)
class ProductControllerTest {
    @MockBean ProductService service;
    private final MockMvc mvc;
    ProductControllerTest(MockMvc mvc) { this.mvc = mvc; }

    @Test
    void quote_ok() throws Exception {
        when(service.quote(1L, 2)).thenReturn(200);
        mvc.perform(get("/api/products/1/quote").param("qty","2"))
           .andExpect(status().isOk())
           .andExpect(jsonPath("$.total").value(200));
    }
}
```

### 통합 테스트 — `@SpringBootTest`

- H2/PostgreSQL(Testcontainers)로 실제 영속 계층 검증
- `@AutoConfigureMockMvc`로 MockMvc 주입

```java
@SpringBootTest
@AutoConfigureMockMvc
class IntegrationTest {
    @Autowired MockMvc mvc;
    @Test void register_and_quote() throws Exception {
        mvc.perform(post("/api/products")
             .contentType("application/json")
             .content("""{"name":"X","unitPrice":150}"""))
           .andExpect(status().isOk());

        mvc.perform(get("/api/products/1/quote").param("qty","3"))
           .andExpect(status().isOk());
    }
}
```

---

## 마이그레이션 가이드 — “new → DI → Boot” 단계별 변환

### 0단계: 코어 Java (직접 조립)

```java
public class App0 {
    public static void main(String[] args) {
        PricingPolicy policy = new FlatPricingPolicy();
        ProductRepository repo = new InMemoryProductRepository();
        ProductService service = new ProductService(repo, policy);
        System.out.println(service.quote(1L, 2));
    }
}
```

### 1단계: Spring 컨테이너 도입(애너테이션 설정)

```java
@Configuration
@ComponentScan(basePackages = "com.example")
class AppConfig {}

public class App1 {
    public static void main(String[] args) {
        try (var ctx = new AnnotationConfigApplicationContext(AppConfig.class)) {
            var service = ctx.getBean(ProductService.class);
            System.out.println(service.quote(1L, 2));
        }
    }
}
```

### 2단계: Spring Boot로 전환

```java
@SpringBootApplication
public class DemoApplication {
    public static void main(String[] args) { SpringApplication.run(DemoApplication.class, args); }
}
```

> 점진 전환 팁: **도메인/규칙은 POJO 유지**, 경계 어댑터만 Spring으로 감싸며 점진적으로 의존 제거.

---

## 확장 토픽 — 캐시, 스케줄링, 이벤트, 모듈 경계

### 캐싱

```java
@EnableCaching
@SpringBootApplication
class App { }

@Service
class PricingCacheService {
    @Cacheable(cacheNames = "quote", key = "#id + '-' + #qty")
    public long quote(Long id, int qty) { /* 비싼 계산 */ return 0; }
}
```

### 스케줄링

```java
@EnableScheduling
@SpringBootApplication
class App2 { }

@Component
class HousekeepingJob {
    @Scheduled(cron = "0 0 * * * *")
    void cleanup() { /* ... */ }
}
```

### 이벤트

```java
@Component
class Publisher {
    private final ApplicationEventPublisher pub;
    Publisher(ApplicationEventPublisher pub) { this.pub = pub; }
    void notifyProductCreated(Product p) { pub.publishEvent(p); }
}

@Component
class Listener {
    @EventListener
    void onProduct(Product p) { /* 알림/지표 */ }
}
```

### 모듈 경계(헥사고날/포트-어댑터)

- **도메인**: 규칙/값/서비스(포트 인터페이스) — **Spring 비의존**
- **어댑터**: Web/JPA/외부 API(스프링 컴포넌트) — 구현 교체 용이
- **구성**: `domain`, `application`, `adapter-web`, `adapter-persistence` 멀티모듈 추천

---

## 안티패턴과 베스트 프랙티스

### 안티패턴

- **필드 주입**: 테스트와 불변성에 취약
- **무분별한 컴포넌트 스캔**: 광범위 스캔으로 **의도치 않은 빈 등록**
- **순환 의존**: 설계 개선 없이 `@Lazy` 남발
- **서비스에 비즈니스/인프라 로직 뒤섞음**: SRP 위반
- **자기호출 트랜잭션**: 프록시 미적용
- **설정 하드코딩**: `@Value` 남발 → `@ConfigurationProperties`로 타입 안전 전환

### 베스트 프랙티스

- **생성자 주입 + 불변 필드**
- **인터페이스(포트) → 구현(어댑터)** 분리
- **설정 외부화 + 프로파일**
- **AOP로 횡단 관심사 분리**
- **슬라이스/통합 테스트 분리, 빠른 단위 테스트 우선**
- **패키지 경계 명시**: `api` vs `internal`
- **DTO/엔티티/도메인 모델 분리**(웹/영속/도메인 혼재 금지)

---

## 체크리스트 — 전환 전/후 최종 점검

- [ ] 도메인 규칙이 **POJO**로 분리되어 있는가
- [ ] `new`를 **생성자 DI**로 대체했는가
- [ ] **컴포넌트 스캔 범위**가 의도대로 설정되었는가
- [ ] 설정 값이 **YAML + @ConfigurationProperties**로 바인딩되는가
- [ ] **트랜잭션 경계**와 격리가 명확한가 (`readOnly`, 전파)
- [ ] **예외 처리**가 `@ControllerAdvice`로 통일되어 있는가
- [ ] **AOP/로깅/메트릭**이 표준화되어 있는가(Actuator 권장)
- [ ] **테스트**: 단위/슬라이스/통합이 분리되어 있고 빠르게 실행되는가
- [ ] **모듈 경계**(포트-어댑터)가 명확하며 교체 가능하게 설계되었는가

---

## JPA로 Repository 교체 예 (선택)

```java
import jakarta.persistence.*;
@Entity
@Table(name = "product")
public class ProductEntity {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    Long id;
    String name;
    long unitPrice;
    // getters/setters
}

public interface JpaProductRepository extends org.springframework.data.jpa.repository.JpaRepository<ProductEntity, Long> { }

@Component
class JpaProductAdapter implements ProductRepository {
    private final JpaProductRepository jpa;
    JpaProductAdapter(JpaProductRepository jpa){ this.jpa = jpa; }

    @Override public Optional<Product> findById(Long id) {
        return jpa.findById(id).map(e -> new Product(e.getId(), e.getName(), e.getUnitPrice()));
    }
    @Override public Product save(Product p) {
        ProductEntity e = new ProductEntity();
        e.setName(p.name()); e.setUnitPrice(p.unitPrice());
        ProductEntity saved = jpa.save(e);
        return new Product(saved.getId(), saved.getName(), saved.getUnitPrice());
    }
}
```

---

## 커스텀 합성 애너테이션(메타 애너테이션)

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Service
@Transactional(readOnly = true)
public @interface ReadonlyService {}

// 사용
@ReadonlyService
class ReportService { /* 모든 public 메서드는 기본 readOnly */ }
```

---

## Boot 애플리케이션 최소 뼈대

```java
@SpringBootApplication
public class App {
    public static void main(String[] args) { SpringApplication.run(App.class, args); }
}

// 실행 시점 로직
@Component
class AppRunner implements org.springframework.boot.ApplicationRunner {
    private final ProductService service;
    AppRunner(ProductService service) { this.service = service; }
    @Override public void run(org.springframework.boot.ApplicationArguments args) {
        service.register("Item", 120);
    }
}
```

---

### 마무리

**POJO 중심 설계**와 **DI**는 프레임워크에서 도메인을 분리하여 **변경에 강한 구조**를 만든다.
**애너테이션 기반 구성**은 개발 경험을 단순화하되, **컴포넌트 경계/설정/트랜잭션/테스트 규율**을 갖출 때 비로소 팀 생산성과 안정성이 극대화된다.
본 가이드의 예제와 체크리스트를 그대로 적용해 **“코어 Java → Spring”** 전환을 단계적으로 완성하라.
