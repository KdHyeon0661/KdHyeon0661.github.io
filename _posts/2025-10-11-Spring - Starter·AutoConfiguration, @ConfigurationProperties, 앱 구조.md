---
layout: post
title: Spring - Starter·AutoConfiguration, @ConfigurationProperties, 앱 구조
date: 2025-10-11 20:25:23 +0900
category: Spring
---
# 스프링 부트 빠른 시작 — Starter·AutoConfiguration, `@ConfigurationProperties`, 앱 구조(패키지/계층/멀티모듈)

> 부트는 “**의존성 + 기본 설정 = 바로 실행 가능한 앱**”을 목표로 한다.
> 핵심은 ① Starter(의존성 세트) ② AutoConfiguration(조건부 자동 설정) ③ 타입-세이프 설정 바인딩(`@ConfigurationProperties`) ④ 일관성 있는 패키지/계층/멀티모듈 설계다.

---

## A. Starter & AutoConfiguration 이해

### A-1. Starter란?

- **Starter** = 관련 라이브러리들을 한 번에 묶은 **의존성 묶음**.
- 예) `spring-boot-starter-web` = Spring MVC + Jackson + 내장 톰캣 + 로깅(등등).
- 장점: **버전 충돌 최소화**(부트 BOM), **의존성 선택 스트레스↓**.

**Gradle 의존성 예**
```kotlin
dependencies {
  implementation("org.springframework.boot:spring-boot-starter-web")
  implementation("org.springframework.boot:spring-boot-starter-validation")
  implementation("org.springframework.boot:spring-boot-starter-actuator")
  testImplementation("org.springframework.boot:spring-boot-starter-test")
}
```

### A-2. AutoConfiguration이란?

- “**클래스패스 상태 + 환경 + 빈 존재 여부**를 조건으로 **자동으로 빈을 등록**”하는 구성.
- 예) `spring-boot-starter-web` 추가 + `spring-webmvc` 존재 → `DispatcherServlet`, `RequestMappingHandlerMapping` 등 자동 등록.
- **조건부 애노테이션**으로 제어:
  - `@ConditionalOnClass`, `@ConditionalOnMissingBean`, `@ConditionalOnProperty`, `@ConditionalOnMissingClass`, `@ConditionalOnWebApplication` …

**자동 구성 조건의 실제 느낌**
```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(name = "org.springframework.web.servlet.DispatcherServlet")
@ConditionalOnProperty(prefix = "app.web", name = "enabled", havingValue = "true", matchIfMissing = true)
class WebAutoConfiguration {

  @Bean
  @ConditionalOnMissingBean
  public HandlerMethodArgumentResolver userIdResolver() {
    return new UserIdArgumentResolver();
  }
}
```

> **핵심 규칙**
> - 개발자가 **직접 동일 타입 빈을 등록하면**(예: 커스텀 `ObjectMapper`) 자동 구성은 **물러난다**(`@ConditionalOnMissingBean`).
> - **클래스패스 + 프로퍼티 + 런타임 환경**을 조합해 “필요할 때만” 적용된다.

---

### A-3. 직접 Starter + AutoConfiguration 만들어보기

> “우리 팀 공통 로깅/ID 마스킹/응답 표준화”를 Starter로 만든다는 가정.

#### A-3-1. 멀티모듈(요약 구조)

```
root/
├─ build.gradle.kts
├─ settings.gradle.kts
├─ app/                 # 실제 서비스 앱
└─ company-boot-starter/  # 커스텀 스타터(자동 구성 포함)
```

#### A-3-2. 커스텀 Starter `build.gradle.kts`

```kotlin
plugins {
  `java-library`
  `maven-publish`
}

group = "com.company"
version = "0.1.0"

java { toolchain { languageVersion.set(JavaLanguageVersion.of(21)) } }

dependencies {
  api(platform("org.springframework.boot:spring-boot-dependencies:3.3.4"))
  api("org.springframework.boot:spring-boot-autoconfigure")
  api("org.springframework.boot:spring-boot-starter-logging")
  compileOnly("org.springframework.boot:spring-boot-configuration-processor")
  testImplementation("org.springframework.boot:spring-boot-starter-test")
}
```

#### A-3-3. 자동 구성 클래스

```java
// company-boot-starter/src/main/java/com/company/autoconfig/MaskingAutoConfiguration.java
package com.company.autoconfig;

import com.company.core.MaskingService;
import org.springframework.boot.autoconfigure.AutoConfiguration;
import org.springframework.boot.autoconfigure.condition.*;
import org.springframework.context.annotation.Bean;

@AutoConfiguration
@ConditionalOnClass(MaskingService.class)
@ConditionalOnProperty(prefix = "company.mask", name = "enabled", havingValue = "true", matchIfMissing = true)
public class MaskingAutoConfiguration {

  @Bean
  @ConditionalOnMissingBean
  public MaskingService maskingService() {
    return new MaskingService("****", 2); // 기본 마스킹 규칙
  }
}
```

#### A-3-4. 구성 속성 바인딩(Starter 내부)

```java
// company-boot-starter/src/main/java/com/company/autoconfig/MaskingProperties.java
package com.company.autoconfig;

import org.springframework.boot.context.properties.ConfigurationProperties;

@ConfigurationProperties(prefix = "company.mask")
public record MaskingProperties(String pattern, int visiblePrefix) {}
```

```java
// company-boot-starter/src/main/java/com/company/autoconfig/MaskingPropsAutoConfig.java
package com.company.autoconfig;

import com.company.core.MaskingService;
import org.springframework.boot.autoconfigure.AutoConfiguration;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.context.annotation.Bean;

@AutoConfiguration
@EnableConfigurationProperties(MaskingProperties.class)
public class MaskingPropsAutoConfig {
  @Bean
  MaskingService maskingService(MaskingProperties p) {
    return new MaskingService(p.pattern(), p.visiblePrefix());
  }
}
```

#### A-3-5. AutoConfiguration 등록 (Boot 3.x 방식)

- **Boot 3.x**부터 `spring.factories`가 아니라 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 사용.

```
# company-boot-starter/src/main/resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports

com.company.autoconfig.MaskingAutoConfiguration
com.company.autoconfig.MaskingPropsAutoConfig
```

> 이렇게 배포한 **Starter**를 서비스 앱에서 `implementation("com.company:company-boot-starter:0.1.0")`만 추가하면,
> `company.mask.enabled=true`일 때 **자동으로** `MaskingService` 빈이 들어온다.

---

## B. `@ConfigurationProperties`로 타입-세이프 설정 바인딩

### B-1. 왜 필요한가?

- `@Value`는 **흩어진 단일 값**에 적합하지만, 설정이 많아지면 유지보수가 어렵다.
- `@ConfigurationProperties`는 **계층 구조**를 **타입**에 **일괄 바인딩**하고, 빌더/레코드로 **불변성**을 지키며 검증까지 가능.

### B-2. 기본 예제 (레코드 + 검증)

```java
// src/main/java/com/example/config/AppProperties.java
package com.example.config;

import jakarta.validation.constraints.*;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.validation.annotation.Validated;

import java.time.Duration;
import java.util.List;
import java.util.Map;

@Validated
@ConfigurationProperties(prefix = "app")
public record AppProperties(
  Server server,
  Auth auth,
  List<String> corsAllowedOrigins,
  Map<String, Integer> featureWeights
) {
  public record Server(
    @NotBlank String host,
    @Min(1) @Max(65535) int port
  ) {}

  public record Auth(
    @NotBlank String issuer,
    @NotNull Duration timeout,
    @Pattern(regexp="^https?://.*$") String jwkSetUri
  ) {}
}
```

```java
// src/main/java/com/example/config/PropsConfig.java
package com.example.config;

import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.context.annotation.Configuration;

@Configuration
@EnableConfigurationProperties(AppProperties.class)
class PropsConfig { }
```

```yaml
# src/main/resources/application.yml

app:
  server:
    host: localhost
    port: 8080
  auth:
    issuer: my-service
    timeout: 30s
    jwkSetUri: "https://idp.local/.well-known/jwks.json"
  corsAllowedOrigins:
    - "https://admin.local"
    - "https://app.local"
  featureWeights:
    recommend: 7
    hot: 3
```

**사용**
```java
@RestController
class InfoController {
  private final AppProperties props;
  InfoController(AppProperties props){ this.props = props; }

  @GetMapping("/_info")
  Map<String,Object> info() {
    return Map.of(
      "host", props.server().host(),
      "port", props.server().port(),
      "issuer", props.auth().issuer(),
      "timeoutSec", props.auth().timeout().toSeconds()
    );
  }
}
```

> 팁
> - **IDE 힌트**: `spring-boot-configuration-processor`를 의존성에 추가하면 YML 자동완성/검증 힌트가 좋아진다.
> - `constructor binding`은 **레코드** 사용 시 자연스럽다(불변 + null-safety).

**Gradle 의존성**
```kotlin
dependencies {
  annotationProcessor("org.springframework.boot:spring-boot-configuration-processor")
}
```

---

### B-3. 프로파일 별 바인딩 + 중첩 문서 사용

```yaml
# application.yml

app:
  server: { host: localhost, port: 8080 }
  auth: { issuer: local, timeout: 10s, jwkSetUri: "http://localhost:9000/jwks" }
---
spring:
  config:
    activate:
      on-profile: prod
app:
  server: { host: 0.0.0.0, port: 8080 }
  auth: { issuer: prod, timeout: 30s, jwkSetUri: "https://idp.prod/jwks" }
```

테스트에서:
```java
@SpringBootTest
@ActiveProfiles("prod")
class AppPropertiesTest {
  @Autowired AppProperties props;

  @Test void prodBinding() {
    assertThat(props.auth().issuer()).isEqualTo("prod");
  }
}
```

---

### B-4. 리팩토링 팁

- 설정이 방대해질 때 **도메인/기능별 클래스**로 나누고 prefix도 분리(`app.payment`, `app.security`).
- 외부 비밀 값은 `${ENV_VAR}`로 **환경변수**에서 바인딩하고, 기본값은 **개발 프로파일**에만 둔다.
- **@ConfigurationProperties** 클래스는 **순수 데이터**만(비즈니스 로직 금지). 로직은 서비스/컴포넌트에.

---

## C. 부트 앱 구조 — 패키지 전략·계층 구조·멀티 모듈

### C-1. 패키지 기본 전략

- `@SpringBootApplication`이 있는 패키지를 **루트**로 하여 **하위 패키지**를 컴포넌트 스캔.
- **권장 구조**(단일 모듈 기준):

```
com.example.myapp
├─ App.java                      # @SpringBootApplication (루트)
├─ config/                       # 설정, 보안, WebMvcConfigurer 등
├─ api/                          # Controller (DTO, request/response)
├─ application/                  # UseCase / Facade (트랜잭션 경계)
├─ domain/                       # 엔티티, 도메인 서비스, 리포지토리 인터페이스
├─ infrastructure/               # 구현체(Repository impl, 외부 API, 메시징)
└─ common/                       # 공통 유틸, 예외, 애노테이션
```

> **스캔 범위 최소화 팁**: 루트 패키지 하위로만 배치하면 `@ComponentScan`의 basePackages를 따로 줄 필요가 없다.

---

### C-2. 계층(헥사고널/클린) 관점의 책임 분리

- **API/Controller**: HTTP 입출력, DTO 변환, 인증/인가 관점.
- **Application/UseCase**: **트랜잭션 경계**, 시나리오 실행 순서, 도메인 서비스 호출, 외부 포트 호출.
- **Domain**: **비즈니스 규칙/불변식/엔티티/값 객체/도메인 이벤트**, Repository **인터페이스(포트)**.
- **Infrastructure**: JPA 구현, 메시징/Kafka, 외부 API **어댑터**.

**샘플**
```java
// domain/OrderRepository.java
package com.example.myapp.domain;
import java.util.Optional;
public interface OrderRepository {
  Order save(Order o);
  Optional<Order> findById(Long id);
}

// application/PlaceOrder.java (유스케이스)
package com.example.myapp.application;
import com.example.myapp.domain.*;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class PlaceOrder {
  private final OrderRepository repo;
  private final PaymentPort payment;
  public PlaceOrder(OrderRepository repo, PaymentPort payment) {
    this.repo = repo; this.payment = payment;
  }

  @Transactional
  public Long handle(PlaceOrderCommand cmd) {
    Order o = Order.create(cmd.userId(), cmd.items());
    repo.save(o);
    payment.charge(o.totalPrice());
    o.markPaid();
    return o.id();
  }
}
```

```java
// infrastructure/JpaOrderRepository.java
package com.example.myapp.infrastructure;
import com.example.myapp.domain.*;
import org.springframework.data.jpa.repository.JpaRepository;

interface JpaOrderJpaRepository extends JpaRepository<OrderEntity, Long> {}

@org.springframework.stereotype.Repository
class JpaOrderRepository implements OrderRepository {
  private final JpaOrderJpaRepository jpa;
  JpaOrderRepository(JpaOrderJpaRepository jpa){ this.jpa = jpa; }

  @Override public Order save(Order o){ /* 매핑 → 저장 → 역매핑 */ return o; }
  @Override public Optional<Order> findById(Long id){ /* ... */ return Optional.empty(); }
}
```

```java
// api/OrderController.java
@RestController
@RequestMapping("/api/orders")
class OrderController {
  private final PlaceOrder place;
  OrderController(PlaceOrder place){ this.place = place; }

  @PostMapping
  public Map<String, Object> place(@RequestBody PlaceOrderRequest req){
    Long id = place.handle(new PlaceOrderCommand(req.userId(), req.items()));
    return Map.of("orderId", id);
  }
}
```

> **장점**: 프레임워크 의존을 **Infrastructure**로 몰아넣고, **Domain**은 순수 자바로 유지 → 테스트 용이/교체 용이.

---

### C-3. 멀티모듈 구성(추천)

> 목표: **도메인/애플리케이션 로직을 프레임워크로부터 분리**하고, 재사용 가능한 **공통 모듈**을 만든다.

#### C-3-1. 구조 예시

```
myapp/
├─ settings.gradle.kts
├─ build.gradle.kts
├─ common/                # 공통 유틸/예외/annotation
├─ domain/                # 엔티티, 값객체, 도메인 서비스, 포트 인터페이스
├─ application/           # 유스케이스(트랜잭션), 도메인 조합
├─ infrastructure/        # JPA, 외부 API, 메시징, config
└─ boot-api/              # Spring Boot 웹/API(컨트롤러, 시큐리티, starter 의존)
```

#### C-3-2. `settings.gradle.kts`

```kotlin
rootProject.name = "myapp"
include("common", "domain", "application", "infrastructure", "boot-api")
```

#### C-3-3. 루트 `build.gradle.kts`

```kotlin
plugins {
  id("org.springframework.boot") version "3.3.4" apply false
  id("io.spring.dependency-management") version "1.1.6" apply false
  java
}

subprojects {
  repositories { mavenCentral() }
  java { toolchain { languageVersion.set(JavaLanguageVersion.of(21)) } }
  tasks.withType<Test> { useJUnitPlatform() }
}
```

#### C-3-4. 각 모듈 의존성

```kotlin
// common/build.gradle.kts
plugins { `java-library` }
dependencies { /* 로깅/유틸 등 필요시 */ }

// domain/build.gradle.kts
plugins { `java-library` }
dependencies {
  api(project(":common"))
  // 도메인은 스프링에 의존하지 않도록 권장
}

// application/build.gradle.kts
plugins { `java-library` }
dependencies {
  api(project(":domain"))
  implementation("org.springframework:spring-tx") // 트랜잭션 경계만
}

// infrastructure/build.gradle.kts
plugins { `java-library` }
dependencies {
  api(project(":application"))
  implementation("org.springframework.data:spring-data-jpa")
  implementation("org.hibernate.orm:hibernate-core")
  runtimeOnly("org.postgresql:postgresql")
}

// boot-api/build.gradle.kts
plugins {
  id("org.springframework.boot")
  id("io.spring.dependency-management")
}
dependencies {
  implementation(project(":infrastructure"))
  implementation("org.springframework.boot:spring-boot-starter-web")
  implementation("org.springframework.boot:spring-boot-starter-validation")
  implementation("org.springframework.boot:spring-boot-starter-actuator")
  testImplementation("org.springframework.boot:spring-boot-starter-test")
}
```

> **핵심**: `boot-api`만 스프링 부트 스타터에 의존.
> `domain/application`은 **프레임워크 독립**을 유지해 순수 테스트/재사용 용이.

---

### C-4. DTO/엔티티 매핑과 계층 간 변환

- Controller는 **Request/Response DTO**만 다룬다.
- Application 계층은 **Command/Query 모델**을 사용.
- Infrastructure는 **JPA 엔티티 ↔ 도메인 모델 매핑** 책임.

**간단 예**
```java
// api/dto
record PlaceOrderRequest(String userId, List<ItemDto> items) {}
record ItemDto(Long productId, int qty) {}

// application command
public record PlaceOrderCommand(String userId, List<Line> lines) {
  public record Line(long productId, int qty) {}
}

// 변환 헬퍼
class OrderMapper {
  static PlaceOrderCommand toCommand(PlaceOrderRequest r){
    var lines = r.items().stream()
      .map(i -> new PlaceOrderCommand.Line(i.productId(), i.qty()))
      .toList();
    return new PlaceOrderCommand(r.userId(), lines);
  }
}
```

---

### C-5. 설정/보안/관측 모듈화

- `infrastructure`에 **JPA 설정/레포 구현**, 메시지 브로커(Kafka/Rabbit) 클라이언트, 외부 API 어댑터.
- `boot-api`에 **Spring Security** config, WebMvc config, Actuator 노출 범위 등 **배포 앱 전용 설정**.

---

### C-6. 테스트 전략(슬라이스 & 통합)

- **슬라이스 테스트**: `@DataJpaTest`(JPA만), `@WebMvcTest`(Controller만), `@JsonTest`(직렬화 검증) …
- **통합 테스트**: `@SpringBootTest` + Testcontainers(DB/브로커)로 **실제 환경 복제**.
- **도메인 단위 테스트**: 순수 자바로 **빠르게**.

**예) JPA 슬라이스**
```java
@DataJpaTest
class OrderJpaTest {
  @Autowired TestEntityManager em;

  @Test void saveLoad(){
    var e = new OrderEntity(/*...*/);
    em.persistAndFlush(e);
    assertThat(em.find(OrderEntity.class, e.getId())).isNotNull();
  }
}
```

---

## D. 빠른 실습: 10분 안에 “부트 API + 설정 바인딩 + 자동 구성 오버라이드” 맛보기

### D-1. 프로젝트 생성(요약)

```bash
curl -s https://start.spring.io/starter.tgz \
  -d bootVersion=3.3.4 \
  -d type=gradle-project \
  -d language=java \
  -d groupId=com.example \
  -d artifactId=quick-boot \
  -d name=quick-boot \
  -d packageName=com.example.quick \
  -d javaVersion=21 \
  -d dependencies=web,validation,actuator,configuration-processor \
| tar -xz --strip-components=1
```

### D-2. `@ConfigurationProperties` 추가

```java
// com/example/quick/config/ApiProps.java
package com.example.quick.config;

import org.springframework.boot.context.properties.ConfigurationProperties;

@ConfigurationProperties(prefix = "api")
public record ApiProps(String title, String baseUrl, int timeoutMs) { }
```

```java
// com/example/quick/config/Config.java
@Configuration
@EnableConfigurationProperties(ApiProps.class)
class Config { }
```

```yaml
# application.yml

api:
  title: "Quick Boot"
  baseUrl: "https://api.local"
  timeoutMs: 1500
```

### D-3. Controller에서 사용

```java
@RestController
@RequestMapping("/hello")
class HelloController {
  private final ApiProps props;
  HelloController(ApiProps props){ this.props = props; }

  @GetMapping
  Map<String,Object> hello(@RequestParam(defaultValue="world") String name){
    return Map.of(
      "message", "Hello, " + name,
      "title", props.title(),
      "baseUrl", props.baseUrl()
    );
  }
}
```

### D-4. 자동 구성 **오버라이드** 체험

- Jackson ObjectMapper를 개발자가 직접 등록하면 **부트 기본 구성**이 물러난다.

```java
@Configuration
class JacksonConfig {
  @Bean
  ObjectMapper objectMapper() {
    return JsonMapper.builder()
      .configure(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS, false)
      .findAndRegisterModules()
      .build();
  }
}
```

> **검증**: `/hello` JSON 포맷/날짜 직렬화가 커스텀 규칙을 따른다.

---

## E. 실전 체크리스트 & 베스트 프랙티스

### E-1. Starter/AutoConfiguration

- [ ] “팀 Starter”를 만들어 **로깅/응답 표준/에러 핸들러**를 중앙화.
- [ ] `@ConditionalOnMissingBean`으로 **개발자 오버라이드** 허용.
- [ ] `AutoConfiguration.imports`에 **자동 구성 클래스**를 등록(Boot 3.x).

### E-2. `@ConfigurationProperties`

- [ ] 기능별로 prefix 분리(`app.auth`, `app.payment`, `app.cache`).
- [ ] 레코드/불변 타입 + **검증 애노테이션**으로 초기부터 **유효성 보장**.
- [ ] `configuration-processor` 추가로 IDE **힌트** 활용.

### E-3. 구조/계층/멀티모듈

- [ ] **부트 의존은 boot-api에만**. 도메인/애플리케이션은 **순수 자바**.
- [ ] Controller는 DTO만, 비즈니스는 UseCase/Domain에 배치.
- [ ] Infra에서만 JPA/외부 API/메시징 구현(포트/어댑터).

### E-4. 운영/성능

- [ ] Actuator 노출은 최소화(프로파일/네트워크 경계 뒤).
- [ ] `spring.main.lazy-initialization=true`는 **개발 빌드 속도**에만(운영은 권장X).
- [ ] 이미지 빌드: Jib/Buildpacks/멀티스테이지 중 팀 표준화, 계층 캐시 효율을 극대화.
- [ ] 트래픽/DTO 큰 경우 **Jackson Afterburner** 등 고려.

### E-5. 흔한 함정

- **자동 구성 무시**: 같은 타입의 커스텀 빈 존재로 오토구성이 꺼지는 경우 → 의도 확인.
- **패키지 스캔 누락**: `@SpringBootApplication`의 루트 위치를 잘못 잡아 하위가 스캔되지 않음.
- **프로퍼티 키 오타**: IDE 힌트/테스트로 조기 탐지.
- **순환 의존**: Starter에서 무심코 강한 의존을 만들지 말 것(조건부 + 인터페이스 사용).

---

## F. 마무리 요약

- **Starter**는 의존성 세트, **AutoConfiguration**은 “상황에 맞춰” 빈을 깔아준다.
- **`@ConfigurationProperties`**로 설정을 타입-세이프하게 묶어 **검증/리팩토링**이 쉬워진다.
- **앱 구조**는 부트 의존을 가장자리에 두고, **도메인/애플리케이션을 순수화**해 테스트성과 유지보수를 높인다.
- **멀티모듈**은 재사용/경계 명확화/빌드 시간 최적화에 유리하며, 팀 Starter로 **조직 표준**을 캡슐화하자.
