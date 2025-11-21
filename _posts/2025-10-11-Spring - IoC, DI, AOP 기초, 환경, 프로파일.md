---
layout: post
title: Spring - IoC/DI, AOP 기초, 환경/프로파일
date: 2025-10-11 19:25:23 +0900
category: Spring
---
# 스프링 핵심 개념 — IoC/DI, AOP 기초, 환경/프로파일

> “**객체의 생명주기와 의존성은 컨테이너가 맡고**, 횡단 관심사는 **프록시(Proxy)**로 분리하며, 실행 환경별 설정은 **프로파일(Profile)**로 관리한다.”

---

## A. IoC/DI: Bean, Container, Scope, Lifecycle

### A-1. 용어와 큰 그림

- **IoC (Inversion of Control)**: 객체 생성·연결·수명 관리를 개발자가 아니라 **컨테이너(ApplicationContext)**가 담당.
- **DI (Dependency Injection)**: 필요한 의존성을 **외부에서 주입**. (생성자/세터/필드)
- **Bean**: 컨테이너가 관리하는 객체.
- **Container**: `BeanFactory`(최소 기능) ⊂ `ApplicationContext`(국제화/이벤트/리소스/환경 등 포함).
- **Scope**: 빈의 **수명 범위**. `singleton`(기본), `prototype`, `request`, `session`, `application`, `websocket`.
- **Lifecycle**: **정의 → 생성 → 의존성 주입 → 초기화 → 사용 → 소멸**.

> 핵심 규칙
> - **생성자 주입**을 기본으로. (불변성/테스트 용이성/순환 참조 조기 탐지)
> - 필요한 경우에만 `@Lazy`, `ObjectProvider`, `Provider<T>`로 지연 주입.
> - **빈의 “목적(역할)”**별로 인터페이스/구현을 분리하고 @Qualifier/@Primary로 선택성을 부여.

---

### A-2. 가장 권장되는 DI 형태: **생성자 주입**

```java
// A-2-1) 서비스 인터페이스/구현
public interface Notifier { void send(String to, String message); }

@Component
class EmailNotifier implements Notifier {
  @Override public void send(String to, String message) { /* SMTP 전송 */ }
}

@Component
class SmsNotifier implements Notifier {
  @Override public void send(String to, String message) { /* SMS Gateway */ }
}

// A-2-2) @Primary / @Qualifier로 선택
@Configuration
class NotifierConfig {
  @Bean @Primary Notifier emailNotifier() { return new EmailNotifier(); }
  @Bean @Qualifier("sms") Notifier smsNotifier() { return new SmsNotifier(); }
}

// A-2-3) 생성자 주입
@Service
class SignUpService {
  private final Notifier notifier;        // @Primary가 기본 선택됨
  public SignUpService(Notifier notifier) { this.notifier = notifier; }
  public void signUp(String email) {
    // 가입 로직...
    notifier.send(email, "Welcome!");
  }
}

// A-2-4) 특정 구현을 명시하고 싶으면
@Service
class AlertService {
  private final Notifier sms;
  public AlertService(@Qualifier("sms") Notifier sms) { this.sms = sms; }
}
```

**장점**
- 의존성 누락/순환 참조를 **컴파일/실행 초기**에 바로 드러냄.
- 테스트에서 **대체 구현**(스텁/목)을 생성자로 간단히 주입.

---

### A-3. 스코프(Scopes)와 상태 설계

| 스코프        | 설명 | 주 사용처 |
|---|---|---|
| `singleton`   | 컨테이너당 1개 (기본) | 대부분의 서비스/리포지토리 |
| `prototype`   | 요청할 때마다 새로 생성 | 상태가 매번 달라야 할 때, 짧은 객체 |
| `request`     | HTTP 요청마다 1개 | 요청 컨텍스트 데이터 홀더 |
| `session`     | 세션마다 1개 | 세션별 사용자 상태 |
| `application` | 서블릿 컨텍스트마다 1개 | 앱 전역 공유 객체 |
| `websocket`   | 웹소켓 세션마다 1개 | 실시간 연결 상태 |

```java
@Component @Scope("prototype")
class TraceContext { /* 호출체인별 상관ID 저장 등 */ }

@RestController
class DemoController {
  private final ObjectProvider<TraceContext> traces;
  DemoController(ObjectProvider<TraceContext> traces) { this.traces = traces; }

  @GetMapping("/do")
  public String doWork() {
    TraceContext t = traces.getObject(); // 요청마다 “새 인스턴스”
    // ...
    return "ok";
  }
}
```

> 주의: **싱글톤 빈**이 **프로토타입 빈**을 주입받으면 **주입 시점에 1번만 생성**된다.
> → 매 호출마다 새 인스턴스가 필요하면 `ObjectProvider`/`Provider<T>`/`ApplicationContext.getBean()` 사용.

---

### A-4. 라이프사이클: 초기화·소멸 훅

- **초기화**: `@PostConstruct`, `InitializingBean#afterPropertiesSet()`, `@Bean(initMethod=...)`
- **소멸**: `@PreDestroy`, `DisposableBean#destroy()`, `@Bean(destroyMethod=...)`
- **실행 시점**: `CommandLineRunner`, `ApplicationRunner`
- **자동 시작/중지**: `SmartLifecycle`

```java
@Component
class ChannelManager implements InitializingBean, DisposableBean {
  @Override public void afterPropertiesSet() { /* 커넥션 풀 준비 */ }
  @Override public void destroy() { /* 커넥션 정리 */ }
}

@Component
class WarmUpRunner implements ApplicationRunner {
  @Override public void run(ApplicationArguments args) { /* 캐시 예열 */ }
}
```

---

### A-5. 빈 등록 방법: 컴포넌트 스캔 vs @Bean

```java
// A-5-1) 컴포넌트 스캔
@SpringBootApplication
public class App { public static void main(String[] args) { SpringApplication.run(App.class, args); }}

// @Component / @Service / @Repository / @Controller / @RestController
@Service class PaymentService { /* ... */ }

// A-5-2) @Bean: 외부 라이브러리/팩토리 패턴에 적합
@Configuration
class ExternalConfig {
  @Bean ObjectMapper objectMapper() {
    return new ObjectMapper().findAndRegisterModules();
  }
}
```

> 규칙: **도메인 로직/앱 코드**는 컴포넌트 스캔, **서드파티/설정성 객체**는 `@Bean`.

---

### A-6. 충돌과 선택: @Primary, @Qualifier, @Order

- 같은 타입의 빈이 다수 → `@Primary`가 기본 선택.
- 이름 지정 필요 → `@Qualifier("name")`.
- AOP/후처리기 실행 순서 → `@Order(n)` 또는 `Ordered` 구현.

---

### A-7. 순환 참조, 지연 주입, 팩토리

- **순환 참조**(A→B, B→A)는 **설계 냄새**. 분리/이벤트 발행/전략 객체로 끊기.
- 꼭 필요하면 `@Lazy` 또는 `ObjectProvider`.
- 팩토리/빌더는 **도메인 객체 생성 규칙**을 캡슐화.

```java
@Component class A {
  private final B b;
  A(@Lazy B b) { this.b = b; } // 지연 해결(권장X, 임시방편)
}
@Component class B {
  private final A a;
  B(A a) { this.a = a; }
}
```

---

### A-8. 컨테이너 확장: PostProcessor

```java
@Component
class TimingBeanPostProcessor implements BeanPostProcessor {
  @Override public Object postProcessBeforeInitialization(Object bean, String name) {
    // 초기화 전 로깅/프록시 래핑 등
    return bean;
  }
  @Override public Object postProcessAfterInitialization(Object bean, String name) {
    return bean;
  }
}
```

- **BeanFactoryPostProcessor**: 빈 정의(메타데이터) 단계 수정.
- **BeanPostProcessor**: 실제 빈 인스턴스 가로채어 래핑/프록시.

---

## B. AOP 기초 — Pointcut, Advice, Proxy

### B-1. 왜 AOP인가?

- **횡단 관심사**(트랜잭션, 로깅, 보안, 측정, 리트라이 등)를 **핵심 로직과 분리**.
- 스프링은 **프록시 기반 AOP**(JDK Dynamic Proxy/CGLIB).
- 실제 메서드 호출 전후에 **어드바이스(Advice)**가 삽입.

> 주의: **자기 자신(this)**의 내부 메서드 호출(**self-invocation**)은 프록시를 통과하지 않아 어드바이스가 적용되지 않는다.

---

### 표현식 핵심

- `execution(modifiers-pattern? ret-type-pattern declaring-type-pattern? name-pattern(param-pattern) throws-pattern?)`
- 자주 쓰는 예:
  - `execution(* com.example..*Service.*(..))`
  - `within(com.example..controller..*)`
  - `@annotation(com.example.Loggable)`
  - `bean(*Controller)` (빈 이름 기준)

---

### B-3. Advice 종류와 예제

```java
@Aspect
@Component
@Order(1) // 우선순위
class LoggingAspect {

  // B-3-1) Before
  @Before("execution(* com.example..*Service.*(..))")
  public void before(JoinPoint jp) {
    System.out.println("[BEFORE] " + jp.getSignature());
  }

  // B-3-2) AfterReturning
  @AfterReturning(
    pointcut = "execution(* com.example..*Service.*(..))",
    returning = "ret")
  public void afterReturning(JoinPoint jp, Object ret) {
    System.out.println("[RETURN] " + jp.getSignature() + " => " + ret);
  }

  // B-3-3) AfterThrowing
  @AfterThrowing(
    pointcut = "execution(* com.example..*Service.*(..))",
    throwing = "ex")
  public void afterThrowing(JoinPoint jp, Throwable ex) {
    System.out.println("[THROW] " + jp.getSignature() + " => " + ex.getMessage());
  }

  // B-3-4) Around: 가장 강력(전/후/예외/대체)
  @Around("execution(* com.example..*Service.*(..))")
  public Object around(ProceedingJoinPoint pjp) throws Throwable {
    long t0 = System.nanoTime();
    try {
      Object res = pjp.proceed();
      return res;
    } finally {
      long tookMs = (System.nanoTime() - t0) / 1_000_000;
      System.out.println("[TIME] " + pjp.getSignature() + " " + tookMs + "ms");
    }
  }
}
```

---

### B-4. 프록시 타입과 제약

- **JDK 동적 프록시**: 인터페이스 기반.
- **CGLIB**: 클래스 상속 기반(최종 클래스/메서드 `final`이면 불가).
- 스프링 부트는 기본적으로 **두 방식 자동 선택**(인터페이스 있으면 JDK, 아니면 CGLIB).
- 설정: `@EnableAspectJAutoProxy(proxyTargetClass = true, exposeProxy = true)`

```java
@Configuration
@EnableAspectJAutoProxy(exposeProxy = true) // AopContext 사용 허용
class AopConfig { }
```

**Self-invocation 해결**
```java
@Service
class ReportService {
  public void entry() {
    // 내부 호출은 프록시 미적용 → 아래 라인처럼 프록시를 얻어 호출
    ((ReportService) AopContext.currentProxy()).internal();
  }
  public void internal() { /* 트랜잭션/로깅 어드바이스 대상 */ }
}
```

---

### B-5. 트랜잭션과 AOP (@Transactional)

```java
@Service
class OrderService {

  private final OrderRepository repo;

  OrderService(OrderRepository repo) { this.repo = repo; }

  @Transactional // 트랜잭션 경계(프록시로 구현)
  public Long place(Order o) {
    repo.save(o);
    // 다른 트랜잭션 전파/격리도 설정 가능
    return o.getId();
  }

  public void outer() {
    // self-invocation 문제: 같은 클래스 내에서 place() 호출 시 @Transactional 미적용
    // 해결: 같은 빈의 프록시를 통해 호출
    ((OrderService) AopContext.currentProxy()).place(new Order());
  }
}
```

> **Pitfall**
> - `private` 메서드/`final` 클래스는 프록시에서 어드바이스 불가.
> - 트랜잭션은 **런타임 예외(언체크)** 기본 롤백. 체크 예외는 지정 필요.
> - `@Transactional(readOnly = true)`는 읽기 최적화(드라이버/프레임워크 의존) 및 쓰기 금지 관례.

---

### B-6. 어노테이션 기반 커스텀 AOP

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Loggable { String value() default ""; }

@Aspect @Component
class AnnotationLoggingAspect {
  @Around("@annotation(loggable)")
  public Object around(ProceedingJoinPoint pjp, Loggable loggable) throws Throwable {
    System.out.println("[" + loggable.value() + "] start");
    try { return pjp.proceed(); }
    finally { System.out.println("[" + loggable.value() + "] end"); }
  }
}
```

---

## C. 환경/프로파일: `application.yml`, Profile, Property 우선순위

### C-1. Config Data & 파일 구조(스프링 부트 2.4+)

- 기본 파일: `application.yml`
- 프로파일별: `application-dev.yml`, `application-prod.yml`
- 하나의 파일 내 다중 문서: `---` 구분 + `spring.config.activate.on-profile`

```yaml
# application.yml

spring:
  application:
    name: shop-api
  datasource:
    url: jdbc:postgresql://localhost:5432/shop
    username: app
    password: app

management:
  endpoints:
    web.exposure.include: health,info,metrics

---
spring:
  config:
    activate:
      on-profile: prod
  datasource:
    url: jdbc:postgresql://prod-db:5432/shop
    username: produser
    password: ${DB_PASSWORD}      # 환경변수/외부 시크릿 연동
logging:
  level:
    root: INFO
```

실행 시 프로파일 지정:
```bash
# 환경변수

export SPRING_PROFILES_ACTIVE=dev
# JVM 파라미터

java -jar app.jar --spring.profiles.active=prod
# 테스트에서

@ActiveProfiles("test")
```

---

### C-2. Property 우선순위(핵심 요약)

> **우선순위 높음 → 낮음** (일반적인 부트 기본값 기준)

1. **커맨드라인 인자**: `--server.port=9090`
2. **OS 환경변수**: `SERVER_PORT=9090` (Relaxed Binding)
3. **`application-<profile>.yml`** (활성 프로파일)
4. **`application.yml`**
5. **스프링 코드 내 `@PropertySource`/기타**
6. **자동 구성/기본값**

> “나중에 로드된(더 특화된) 설정”이 “먼저 로드된(덜 특화된) 설정”을 덮는다.

---

### C-3. @Value vs @ConfigurationProperties

- `@Value("${path.to.prop:default}")`: **개별 값** 주입, SpEL 지원, 간단하지만 산재하기 쉬움.
- `@ConfigurationProperties(prefix="app")`: **묶음 바인딩**(계층 전체), 타입 세이프, 리팩토링 친화.

```java
// C-3-1) 타입 세이프 바인딩
@ConfigurationProperties(prefix = "app")
public record AppProps(Server server, Auth auth) {
  public record Server(String host, int port) { }
  public record Auth(String issuer, Duration timeout) { }
}

@Configuration
@EnableConfigurationProperties(AppProps.class)
class PropsConfig { }

// C-3-2) 사용
@RestController
class InfoController {
  private final AppProps props;
  InfoController(AppProps props) { this.props = props; }

  @GetMapping("/info")
  Map<String,Object> info() {
    return Map.of("host", props.server().host(), "port", props.server().port());
  }
}
```

```yaml
# application.yml

app:
  server:
    host: localhost
    port: 8080
  auth:
    issuer: demo
    timeout: 30s
```

---

### C-4. Profile로 **환경별 Bean** 분기

```java
public interface PaymentGateway { void pay(long amount); }

@Component @Profile("dev")
class FakeGateway implements PaymentGateway {
  @Override public void pay(long amount) { System.out.println("[DEV] paid " + amount); }
}

@Component @Profile("prod")
class RealGateway implements PaymentGateway {
  @Override public void pay(long amount) { /* 외부 결제사 호출 */ }
}
```

- 다형성 + 프로파일로 운영/개발 분리를 명확히.
- 더 섬세한 조건은 `@ConditionalOnProperty`, `@ConditionalOnClass`, `@Conditional(...)` 사용.

```java
@Configuration
class MetricsConfig {
  @Bean
  @ConditionalOnProperty(name = "metrics.enabled", havingValue = "true", matchIfMissing = true)
  MeterRegistryCustomizer<?> commonTags() { /* ... */ return registry -> {}; }
}
```

---

### C-5. 외부 설정 위치/시크릿, 덮어쓰기 전략

- **외부 디렉토리/URL에서 추가 로드**
  `--spring.config.additional-location=./ext/`
- **환경 변수로 시크릿 전달**
  `DB_PASSWORD=...` → `${DB_PASSWORD}`로 참조
- **Kubernetes**: ConfigMap/Secret을 **환경변수 또는 마운트 파일**로 주입.

```yaml
# application-prod.yml

spring:
  datasource:
    password: ${DB_PASSWORD}    # 런타임에 주입
```

---

### C-6. 테스트 전용 속성 주입

```java
@SpringBootTest
@TestPropertySource(properties = {
  "app.auth.timeout=5s",
  "spring.jpa.hibernate.ddl-auto=create-drop"
})
@ActiveProfiles("test")
class DemoTests { /* ... */ }
```

또는 테스트 리소스에 `application-test.yml` 두고 `@ActiveProfiles("test")`.

---

## D. 실전 예제: “주문/결제” 미니 케이스로 묶어 보기

### D-1. 도메인/리포지토리/서비스

```java
@Entity
class Order {
  @Id @GeneratedValue Long id;
  String userId;
  long amount;
  @Enumerated(EnumType.STRING) Status status = Status.CREATED;
  enum Status { CREATED, PAID, FAILED }
}

public interface OrderRepository extends JpaRepository<Order, Long> { }

public interface PaymentGateway { void pay(long amount); }

@Service
class OrderService {
  private final OrderRepository repo;
  private final PaymentGateway gateway;
  OrderService(OrderRepository repo, PaymentGateway gateway) {
    this.repo = repo; this.gateway = gateway;
  }

  @Transactional
  public Long place(String userId, long amount) {
    Order o = new Order();
    o.userId = userId; o.amount = amount;
    repo.save(o);
    // 결제 시도
    try {
      gateway.pay(amount);
      o.status = Order.Status.PAID;
    } catch (RuntimeException e) {
      o.status = Order.Status.FAILED;
      throw e; // 트랜잭션 롤백
    }
    return o.id;
  }
}
```

```java
// 프로파일 분기: 결제 게이트웨이
@Component @Profile("dev")
class FakeGateway implements PaymentGateway {
  @Override public void pay(long amount) { /* no-op: 성공 가정 */ }
}

@Component @Profile("prod")
class RealGateway implements PaymentGateway {
  @Override public void pay(long amount) { /* 외부 결제사 API 호출/예외 처리 */ }
}
```

### D-2. AOP로 로깅/시간 측정/마스킹

```java
@Target(ElementType.METHOD) @Retention(RetentionPolicy.RUNTIME)
public @interface MaskLog { }

@Aspect @Component @Order(1)
class MaskingAspect {
  @Around("@annotation(MaskLog)")
  public Object mask(ProceedingJoinPoint pjp) throws Throwable {
    Object[] args = pjp.getArgs();
    Object[] safe = Arrays.stream(args).map(a -> maskIfSensitive(a)).toArray();
    System.out.println("[ARGS] " + Arrays.toString(safe));
    return pjp.proceed();
  }
  private Object maskIfSensitive(Object a) {
    if (a instanceof String s && s.contains("@")) return "***@***"; // 이메일 마스킹 예
    return a;
  }
}

@Aspect @Component @Order(2)
class TimingAspect {
  @Around("execution(* com.example..*Service.*(..))")
  public Object around(ProceedingJoinPoint pjp) throws Throwable {
    long t0 = System.nanoTime();
    try { return pjp.proceed(); }
    finally {
      long ms = (System.nanoTime() - t0)/1_000_000;
      System.out.println("[TIME] " + pjp.getSignature() + " " + ms + "ms");
    }
  }
}

@RestController
class OrderController {
  private final OrderService service;
  OrderController(OrderService service) { this.service = service; }

  @PostMapping("/orders")
  @MaskLog
  public Map<String,Object> place(@RequestParam String userId, @RequestParam long amount) {
    long id = service.place(userId, amount);
    return Map.of("orderId", id);
  }
}
```

### D-3. 설정과 프로파일 YML

```yaml
# application.yml

spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/shop
    username: app
    password: app
  jpa:
    hibernate.ddl-auto: update
    properties:
      hibernate.format_sql: true
logging:
  level:
    org.hibernate.SQL: debug

---
spring:
  config.activate.on-profile: prod
  datasource:
    url: jdbc:postgresql://prod-db:5432/shop
    username: prod
    password: ${DB_PASSWORD}
logging:
  level:
    root: INFO
```

---

## E. 트러블슈팅 & 베스트 프랙티스

1) **어드바이스가 안 먹는다?**
   - 내부 호출(self-invocation)인지 확인 → `AopContext.currentProxy()` 또는 구조 분리.
   - 메서드/클래스 `final`인지 확인. 인터페이스 유무/프록시 타입.

2) **순환 참조 예외**
   - 설계를 끊어라(이벤트 발행/포트-어댑터/도메인 서비스 분해).
   - 임시로 `@Lazy`는 가능하지만, 원인 제거가 정답.

3) **프로파일 적용이 이상**
   - 활성 프로파일 확인: `/actuator/env`(dev에서만 노출) 또는 로그.
   - `spring.config.activate.on-profile` vs `spring.profiles.active` 구분.

4) **설정 우선순위 충돌**
   - 커맨드라인/환경변수가 최상위.
   - 로깅 레벨/데이터소스가 엎어쓰였는지 확인.

5) **@Transactional 롤백 안 됨**
   - 체크 예외/커스텀 예외의 롤백 규칙 확인(`rollbackFor`).
   - 메서드 `public`인지, 프록시 경유 호출인지 확인.

6) **성능/부팅**
   - AOP는 최소한의 포인트컷 범위로.
   - JPA N+1은 페치 전략/쿼리 튜닝/캐시로 해결.

---

## F. 과제 & 체크리스트 (이 장 완료 기준)

- [ ] 생성자 주입으로 서비스 2~3개 구성, `@Primary/@Qualifier` 사용
- [ ] `prototype` 빈을 `ObjectProvider`로 런타임 의존 획득
- [ ] `@PostConstruct` 초기화와 `@PreDestroy` 정리 로직 구현
- [ ] `@Aspect`로 **시간 측정** `Around` 어드바이스 적용, 범위는 `*Service`로 제한
- [ ] `@Transactional`을 단 서비스 메서드에서 **self-invocation** 재현 & 해결
- [ ] `application.yml` + `application-prod.yml`로 DB/로깅 차등 구성
- [ ] `@ConfigurationProperties`로 타입 세이프 설정 바인딩
- [ ] `@Profile("dev"/"prod")`로 결제 게이트웨이 구현 분기
- [ ] 테스트에서 `@ActiveProfiles("test")` + `@TestPropertySource` 사용

---

## G. 한 페이지 요약

- **IoC/DI**: “**컨테이너가 객체를 만들고 연결**한다.” → 생성자 주입 기본, 스코프·라이프사이클 이해.
- **AOP**: “**프록시가 공통 관심사(트랜잭션/로깅)를 주입**한다.” → 포인트컷/어드바이스, self-invocation 주의.
- **환경/프로파일**: “**설정은 외부화·계층화**하고 프로파일로 분기한다.” → 우선순위·타입 세이프 바인딩.
