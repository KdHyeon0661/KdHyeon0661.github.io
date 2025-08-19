---
layout: post
title: Java - Spring, POJO·DI, 애너테이션 기반 개발
date: 2025-08-18 16:25:23 +0900
category: Java
---
# Java에서 Spring으로 넘어가기 / POJO·DI 개념 / 애너테이션 기반 개발

Spring은 “**평범한 자바 객체(POJO)** 중심 설계”와 “**의존성 주입(DI)**, **제어의 역전(IoC)**”을 통해 애플리케이션 구조를 단순·유연하게 만듭니다.  
여기서는 (1) **코어 Java에서 Spring으로 전환하는 흐름**, (2) **POJO·DI 핵심 개념**, (3) **애너테이션 기반 개발 방법**을 실전 관점으로 정리합니다.

---

## 1) Java에서 Spring으로 넘어가기

### 1.1 왜 Spring인가
- **객체 생명주기/의존성 관리**를 프레임워크(IoC 컨테이너)가 담당 → 결합도↓, 테스트 용이성↑
- **관심사 분리**: 트랜잭션, 보안, 로깅, 캐시 등 횡단 관심사를 AOP로 통합
- **생태계**: Spring MVC/WebFlux, Data JPA, Security, Batch… 광범위한 통합

### 1.2 전환 전략(실무형)
1. **순수 Java/POJO로 도메인 로직 정리**  
   - 비즈니스 규칙을 프레임워크 독립적으로 캡슐화 (의존 최소화)
2. **생성자 `new` 제거 → DI로 전환**  
   - 의존 객체 생성/조립을 컨테이너에 위임
3. **설정 분리**  
   - Bean 정의(`@Configuration`, `@Bean`)와 애플리케이션 로직 분리
4. **레이어 정리**  
   - `controller`(웹) / `service`(도메인) / `repository`(영속화) 계층화
5. **Spring Boot 도입**  
   - 의존성 관리·자동설정으로 생산성↑, 외부 설정(properties/yaml)로 환경 분리

### 1.3 최소 구동(순수 Spring → Spring Boot)
- (A) **순수 Spring 컨테이너**
```java
import org.springframework.context.annotation.*;

@Configuration
@ComponentScan(basePackages = "com.example")
class AppConfig {}

public class App {
    public static void main(String[] args) {
        try (var ctx = new AnnotationConfigApplicationContext(AppConfig.class)) {
            var svc = ctx.getBean(GreetingService.class);
            System.out.println(svc.greet("Spring"));
        }
    }
}
```

- (B) **Spring Boot**
```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication // @Configuration + @ComponentScan + @EnableAutoConfiguration
public class DemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}
```
> `@SpringBootApplication`을 기준 패키지의 **루트**에 두면 하위 패키지가 컴포넌트 스캔 대상이 됩니다.

---

## 2) POJO와 DI의 개념

### 2.1 POJO (Plain Old Java Object)
- **특정 프레임워크에 종속되지 않은** 순수 자바 객체
- 장점: **테스트 용이**, 재사용성↑, 유지보수성↑  
- 예: 인터페이스 + 구현체, 도메인 엔티티, 서비스 로직 등

```java
public interface GreetingService {
    String greet(String name);
}

public class SimpleGreetingService implements GreetingService {
    @Override public String greet(String name) {
        return "Hello, " + name;
    }
}
```

### 2.2 IoC(제어의 역전) & DI(의존성 주입)
- **IoC**: 객체 생성/조립/생명주기를 **컨테이너가 제어**
- **DI**: 객체가 필요한 의존성을 **외부에서 주입**
  - **생성자 주입(권장)**, 세터 주입, 필드 주입(지양)
- DI 이점
  - **결합도↓**: 구현 교체 쉬움 (`@Primary`, `@Qualifier`)
  - **테스트 용이**: Mock/Fake 주입으로 단위 테스트 쉬움

```java
import org.springframework.stereotype.Service;

@Service
public class GreetingFacade {
    private final GreetingService delegate;
    public GreetingFacade(GreetingService delegate) { // 생성자 주입
        this.delegate = delegate;
    }
    public String greetUser(String user) { return delegate.greet(user); }
}
```

---

## 3) 애너테이션 기반 개발

### 3.1 구성 요소 애너테이션
- **빈 등록(컴포넌트 스캔 대상)**
  - `@Component`(범용), `@Service`(서비스), `@Repository`(예외 변환), `@Controller`/`@RestController`(웹)
- **설정/팩토리 메서드**
  - `@Configuration` + `@Bean`
- **주입**
  - `@Autowired`(생성자/세터/필드), **생성자 주입 시 생략 가능**(스프링 4.3+), `@Qualifier`, `@Primary`
- **스코프**
  - `@Scope("singleton"|"prototype"|"request"|"session")`, `@Lazy`
- **설정 바인딩**
  - `@Value("${key}")`, `@ConfigurationProperties(prefix = "app")`

```java
import org.springframework.context.annotation.*;

@Configuration
class BeanConfig {
    @Bean
    GreetingService greetingService() {
        return new SimpleGreetingService();
    }
}
```

### 3.2 웹 계층(예: REST 컨트롤러)
```java
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/greet")
class GreetingController {
    private final GreetingService service;
    public GreetingController(GreetingService service) { this.service = service; }

    @GetMapping
    public String greet(@RequestParam String name) {
        return service.greet(name);
    }
}
```

### 3.3 프로퍼티/프로파일에 의한 외부화 설정
- `application.yml`로 환경별 값을 외부화, `@Profile`로 빈 활성화 제어
```yaml
# application.yml
app:
  greeting-prefix: "Hello"
spring:
  profiles:
    active: local
```

```java
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;

@Component
@ConfigurationProperties(prefix = "app")
class AppProps {
    private String greetingPrefix;
    public String getGreetingPrefix() { return greetingPrefix; }
    public void setGreetingPrefix(String s) { this.greetingPrefix = s; }
}
```

```java
import org.springframework.context.annotation.Profile;
import org.springframework.stereotype.Component;

@Profile("local")
@Component
class LocalOnlyBean { /* ... */ }
```

### 3.4 트랜잭션/AOP(횡단 관심사)
- **`@Transactional`**: 트랜잭션 경계 선언
- **AOP**: `@Aspect` + 포인트컷으로 로깅/검증/권한 등 공통 로직 주입

```java
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
class OrderService {
    @Transactional
    public void placeOrder(Long userId) {
        // 여러 저장소 작업 → 실패 시 롤백
    }
}
```

```java
import org.aspectj.lang.annotation.*;
import org.springframework.stereotype.Component;

@Aspect
@Component
class LoggingAspect {
    @Before("execution(* com.example..*Service.*(..))")
    public void beforeService() {
        System.out.println("[AOP] service call");
    }
}
```

### 3.5 빈 생명주기 콜백
- **초기화/소멸**: `@PostConstruct`, `@PreDestroy`
- 또는 `InitializingBean`, `DisposableBean`, `@Bean(initMethod, destroyMethod)`

```java
import jakarta.annotation.PostConstruct;
import jakarta.annotation.PreDestroy;
import org.springframework.stereotype.Component;

@Component
class Startup {
    @PostConstruct void init() { System.out.println("init"); }
    @PreDestroy  void bye() { System.out.println("bye"); }
}
```

### 3.6 유용한 메타-애너테이션
- `@Retention`, `@Target` 등을 이용해 **커스텀 합성 애너테이션** 정의 가능  
  (예: 다수의 애너테이션을 하나로 묶어 표준화)

---

## 4) 스프링 부트 자동 구성 개념(요약)
- `@EnableAutoConfiguration`이 클래스패스와 환경설정을 보고 **필요한 빈 자동 등록**
- 예: `spring-boot-starter-web`을 추가하면 **Tomcat + MVC 구성**, JSON 직렬화(Jackson) 자동 설정
- 필요 시 **조건부 빈**(`@ConditionalOnMissingBean`)으로 오버라이드 가능

---

## 5) 테스트와 DI
- **슬라이스 테스트**: `@WebMvcTest`(컨트롤러), `@DataJpaTest`(JPA) 등
- **통합 테스트**: `@SpringBootTest`
- **Mock 주입**: `@MockBean`으로 컨테이너 레벨에 테스트 더블 삽입

```java
import org.junit.jupiter.api.Test;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.test.web.servlet.MockMvc;

import static org.mockito.Mockito.*;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@WebMvcTest(GreetingController.class)
class GreetingControllerTest {

    @MockBean GreetingService service;
    private final MockMvc mvc;

    GreetingControllerTest(MockMvc mvc) { this.mvc = mvc; }

    @Test
    void greet_ok() throws Exception {
        when(service.greet("Kim")).thenReturn("Hello, Kim");
        mvc.perform(get("/api/greet").param("name", "Kim"))
           .andExpect(status().isOk())
           .andExpect(content().string("Hello, Kim"));
    }
}
```

---

## 6) 베스트 프랙티스(핵심 요약)
- **생성자 주입** 기본, **필드 주입 지양**
- 컴포넌트 스캔은 **의미 있는 패키지 경계**로 제한(루트 너무 넓게 잡지 않기)
- 설정 값은 **`@ConfigurationProperties`**로 묶어 타입 안전하게 주입
- 빈의 **단일 책임** 유지, 레이어 간 **인터페이스**로 의존(테스트/교체 용이)
- **순환 의존** 피하기(설계 개선), 필요 시 `@Lazy`로 임시 회피
- 횡단 관심사는 **AOP**로 처리, 비즈니스 로직을 오염시키지 않기
- Boot 자동설정은 **필요할 때만 오버라이드** (불필요한 수동 설정 최소화)

---

## 7) 한눈에 보는 “전환 체크리스트”
- [ ] 도메인 로직을 POJO로 추출 (프레임워크 의존 제거)  
- [ ] `new` 제거 → 생성자 DI로 대체  
- [ ] `@SpringBootApplication` 루트·패키지 구조 정리  
- [ ] `@Service/@Repository/@Controller`로 역할 주석화  
- [ ] 설정은 `@Configuration`/`@Bean` 또는 Boot 자동설정 활용  
- [ ] 환경 값은 `application.yml` + `@ConfigurationProperties`  
- [ ] 트랜잭션 경계 `@Transactional` 명확화  
- [ ] 슬라이스/통합 테스트 체계화 (`@WebMvcTest`, `@SpringBootTest`)  

---

### 마무리
Spring으로의 전환은 **POJO 중심 설계**와 **DI**를 통해 **결합도↓·테스트 용이성↑**를 달성하는 과정입니다.  
애너테이션 기반 개발로 **구성은 간결하게**, **비즈니스 로직은 프레임워크로부터 독립**적으로 유지하세요.  
그 위에 Spring Boot의 자동설정과 생태계를 더하면, **빠르게도 견고하게** 애플리케이션을 확장할 수 있습니다.