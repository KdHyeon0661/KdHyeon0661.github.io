---
layout: post
title: Spring - Spring Security
date: 2025-10-16 15:25:23 +0900
category: Spring
---
# 7) 보안 — Spring Security 기초
> 목표: **요구사항 모델링(인증/인가)**부터 시작해 **필터 체인/AuthenticationProvider**의 흐름을 이해하고, **폼 로그인·세션/쿠키·Remember-me**를 손에 익히며, **패스워드 저장·CSRF/XSS/CORS**의 필수 보안 기초를 실전 예제로 정리한다.  
> 환경 가정: Spring Boot 3.3+, Spring Security 6.x, Java 21, MVC(서블릿).

---

## A. 요구사항 모델링 — 인증(Authentication) vs 인가(Authorization)

### A-1. 핵심 개념 요약
- **인증**: “누구인지”를 확인. (아이디/비밀번호, OTP, SSO, 토큰 등)  
- **인가**: “무엇을 할 수 있는지”를 결정. (권한/역할/정책에 따른 접근 통제)  
- **주체(Principal)**: 인증된 사용자(또는 클라이언트 애플리케이션).  
- **권한(Authorities)**: `ROLE_ADMIN`, `order:read` 같은 레이블.  
- **컨텍스트(SecurityContext)**: 현재 요청의 인증 정보(`Authentication`)가 담긴 공간(스레드 로컬 by default).

### A-2. 요구사항 정리 체크리스트
1) **누가** 접속하나? (내부직원/파트너/최종사용자/머신계정)  
2) **어떻게** 인증하나? (폼 로그인/SSO/SAML/OIDC/JWT/MTLS)  
3) **무엇을** 보호하나? (경로·메서드·리소스)  
4) **승인 정책**은? (역할 기반 RBAC, 권한 기반 ABAC, 규칙 기반 SpEL)  
5) **세션 전략**은? (STATEFUL 세션/쿠키 vs STATELESS 토큰)  
6) **브라우저 보안**: CSRF/CORS/XSS/쿠키 속성(httponly, secure, sameSite)

---

## B. Spring Security 러닝맵 — “요청 → 필터 → 인증 → 인가” 흐름

### B-1. 큰 흐름(서블릿 스택)
```
Client → (Servlet Filter Chain)
      → Security Filters … → UsernamePasswordAuthenticationFilter (폼 로그인)
      → AuthenticationManager → AuthenticationProvider (비밀번호 검증)
      → SecurityContextRepository (세션/쿠키 저장)
      → Handler (Controller)
```

### B-2. 핵심 구성요소
- **SecurityFilterChain**: 보안 필터들의 순서/설정을 담는 빈(여러 개 구성 가능).  
- **AuthenticationManager**: 인증 시도 총괄(Provider 위임).  
- **AuthenticationProvider**: 실제 인증(비밀번호 매칭, 사용자 조회).  
- **UserDetailsService**: 사용자 조회 계약(아이디→사용자).  
- **PasswordEncoder**: 해시/검증.  
- **SecurityContextRepository**: 인증을 저장/복원(세션, 헤더 등).

---

## C. 최소 실습: 폼 로그인 + 인가 규칙

### C-1. 의존성
```kotlin
dependencies {
  implementation("org.springframework.boot:spring-boot-starter-security")
  implementation("org.springframework.boot:spring-boot-starter-web")
  implementation("org.springframework.boot:spring-boot-starter-thymeleaf") // (선택) 로그인 폼
}
```

### C-2. 유저/비밀번호/권한 정의
> 실무에서는 DB/LDAP/OIDC를 쓰지만, 먼저 **인메모리**로 흐름부터 체득한다.

```java
@Configuration
public class SecurityUsersConfig {

  @Bean
  UserDetailsService userDetailsService(PasswordEncoder encoder) {
    UserDetails user = User.withUsername("user")
        .password(encoder.encode("password"))
        .roles("USER") // ROLE_USER
        .build();

    UserDetails admin = User.withUsername("admin")
        .password(encoder.encode("admin123"))
        .roles("ADMIN") // ROLE_ADMIN
        .build();

    return new InMemoryUserDetailsManager(user, admin);
  }

  @Bean
  PasswordEncoder passwordEncoder() {
    // 실무: BCrypt(기본). 옵션 조정은 강도(cost)로.
    return new BCryptPasswordEncoder();
  }
}
```

### C-3. 필터 체인 설정(기본 폼 로그인)
```java
@Configuration
@EnableMethodSecurity // @PreAuthorize 등 메서드 보안 사용 가능
public class SecurityConfig {

  @Bean
  SecurityFilterChain http(HttpSecurity http) throws Exception {
    http
      .csrf(csrf -> csrf.disable()) // (폼 제출이면 활성 권장, API-only면 보통 disable)
      .authorizeHttpRequests(auth -> auth
        .requestMatchers("/login", "/css/**", "/js/**", "/public/**").permitAll()
        .requestMatchers("/admin/**").hasRole("ADMIN")
        .requestMatchers("/api/**").hasAnyRole("ADMIN","USER")
        .anyRequest().authenticated()
      )
      .formLogin(form -> form
        .loginPage("/login")               // 커스텀 로그인 페이지
        .loginProcessingUrl("/doLogin")    // UsernamePasswordAuthenticationFilter 엔드포인트
        .defaultSuccessUrl("/", true)      // 성공 후
        .failureUrl("/login?error=true")   // 실패 후
        .permitAll()
      )
      .logout(logout -> logout
        .logoutUrl("/logout")
        .logoutSuccessUrl("/login?logout")
        .invalidateHttpSession(true)
        .deleteCookies("JSESSIONID")
      )
      .rememberMe(me -> me
        .key("strong-remember-me-key") // 키는 안전하게 관리
        .tokenValiditySeconds(1209600) // 14일
      );

    return http.build();
  }
}
```

> **요점**  
> - `/login` GET: 로그인 폼 제공(Thymeleaf 등)  
> - `/doLogin` POST: 아이디/비밀번호 전송 → **UsernamePasswordAuthenticationFilter**가 처리  
> - 성공 시 `SecurityContext`에 인증 생성 후 `JSESSIONID` 쿠키 발급(STATEFUL)  
> - Remember-me 활성 시 **장기 쿠키**로 인증 복원

### C-4. 로그인 폼(예시)
```html
<!-- templates/login.html (간단 예제) -->
<form method="post" action="/doLogin">
  <input type="text"     name="username" placeholder="username" />
  <input type="password" name="password" placeholder="password" />
  <label><input type="checkbox" name="remember-me" /> Remember me</label>
  <button type="submit">Sign in</button>
  <div th:if="${param.error}">Invalid credentials</div>
  <div th:if="${param.logout}">Logged out</div>
</form>
```

---

## D. 인증 흐름 내부 — 필터/Provider/세션

### D-1. 폼 로그인 인증 시퀀스
1) `UsernamePasswordAuthenticationFilter`가 `/doLogin` POST를 가로챔  
2) `UsernamePasswordAuthenticationToken(unauthenticated)` 생성 → `AuthenticationManager.authenticate()` 호출  
3) `DaoAuthenticationProvider`가 `UserDetailsService`로 사용자 조회 → `PasswordEncoder.matches()`로 비밀번호 검증  
4) 성공 시 `UsernamePasswordAuthenticationToken(authenticated)` 반환  
5) `SecurityContext`에 저장, `SecurityContextRepository`가 세션에 보존 → `JSESSIONID` 쿠키 전달  
6) 이후 요청은 **세션 복원**으로 인증 유지

### D-2. 커스텀 AuthenticationProvider (예: 사내 LDAP/외부 API 인증)
```java
@Component
public class ExternalAuthProvider implements AuthenticationProvider {

  private final ExternalAuthClient client;

  public ExternalAuthProvider(ExternalAuthClient client) { this.client = client; }

  @Override
  public Authentication authenticate(Authentication auth) throws AuthenticationException {
    String username = auth.getName();
    String raw = auth.getCredentials().toString();

    ExternalUser u = client.verify(username, raw); // 외부 인증 API 호출
    if (u == null) throw new BadCredentialsException("Invalid");

    var authorities = u.roles().stream().map(SimpleGrantedAuthority::new).toList();
    var principal = User.withUsername(username).password("N/A").authorities(authorities).build();
    return new UsernamePasswordAuthenticationToken(principal, null, authorities);
  }

  @Override
  public boolean supports(Class<?> authentication) {
    return UsernamePasswordAuthenticationToken.class.isAssignableFrom(authentication);
  }
}
```

```java
@Configuration
public class ProviderOrderConfig {
  @Bean
  AuthenticationManager authenticationManager(List<AuthenticationProvider> providers) {
    // 여러 Provider를 순서대로 위임
    return new ProviderManager(providers);
  }
}
```

> **노하우**: 외부 인증 실패/지연 시 **시간 상수화**(timing attack 완화), **재시도/서킷브레이커**를 넣는다.

---

## E. 인가 — 경로/메서드 보안

### E-1. 경로 기반(위에서 사용)
```java
.authorizeHttpRequests(auth -> auth
  .requestMatchers("/admin/**").hasRole("ADMIN")
  .anyRequest().authenticated()
)
```

### E-2. 메서드 보안(`@EnableMethodSecurity`)
```java
@Service
public class OrderService {
  @PreAuthorize("hasRole('ADMIN') or (#userId == authentication.name)")
  public OrderView getForUser(String userId, long id) { /* ... */ return null; }

  @PostAuthorize("returnObject.owner == authentication.name")
  public OrderView loadWithCheck(long id) { /* ... */ return null; }
}
```

- `@PreAuthorize`, `@PostAuthorize`, `@PreFilter`, `@PostFilter`  
- SpEL에서 `authentication`, `principal`, `#param` 사용 가능  
- **복잡한 정책**은 **PermissionEvaluator**/ABAC로 분리 구현 권장.

---

## F. 세션/쿠키/Remember-me 전략

### F-1. 세션 관리
```java
http
  .sessionManagement(sm -> sm
    .sessionCreationPolicy(SessionCreationPolicy.IF_REQUIRED) // STATEFUL
    .maximumSessions(1)                                      // 동시 세션 제한
    .expiredUrl("/login?expired")
  );
```

- **동시 세션 제한**: 계정 공유/탈취 대응  
- **고객센터/어드민**은 STATEFUL이 일반적, **순수 API**는 보통 **STATELESS**(JWT)로 구성

### F-2. Remember-me
- 디폴트 쿠키 이름: `remember-me`  
- 동작: 쿠키가 유효하면 세션 없어도 인증 컨텍스트 복원  
- 보안: **강한 키/짧은 수명/도메인 제한/SameSite=strict** 권장, 민감 기능은 재인증

```java
.rememberMe(me -> me
  .key("super-strong-unique-key-from-secret-store")
  .tokenValiditySeconds(7 * 24 * 60 * 60)
  .userDetailsService(userDetailsService)
)
```

---

## G. 패스워드 저장 — 해시·업그레이드·정책

### G-1. 해시(BCrypt 권장)
- 적응형 해시(비용 인자)로 **무차별 대입 공격** 지연  
- 스프링 기본 `DelegatingPasswordEncoder`는 `{id}encoded` 형식을 이해(BCrypt 기본)

```java
@Bean PasswordEncoder passwordEncoder() {
  return PasswordEncoderFactories.createDelegatingPasswordEncoder(); // {bcrypt}
}
```

### G-2. 마이그레이션/업그레이드
- 기존 해시(MD5/SHA-1 등) → 로그인 성공 시 **BCrypt 재해시**로 업그레이드  
- 비밀번호 정책: 최소 길이/복잡도/누출 목록(hibp API) 체크, **락아웃/백오프**(실패 횟수)

### G-3. 비밀번호 재설정
- 토큰 기반(단기/일회용), **시도 제한** 및 **IP/디바이스 지표** 로깅

---

## H. CSRF / XSS / CORS — 브라우저 보안 기초

### H-1. CSRF (Cross-Site Request Forgery)
- **문제**: 브라우저가 자동으로 **쿠키를 동봉** → 공격자가 다른 사이트에서 희생자의 쿠키로 POST/PUT/DELETE 실행  
- **해결**: **CSRF 토큰**(양식/헤더에 담아 서버가 검증)  
- Spring Security: **폼/세션 기반 앱에서 기본 활성화**(Boot 3는 기본 on). API만 쓰는 **stateless** 백엔드는 보통 **disable**.

```java
// 폼 앱: enable 유지 (기본)
http.csrf(Customizer.withDefaults());

// JS 요청 시 헤더에 전송(예: X-CSRF-TOKEN)
<meta name="_csrf" th:content="${_csrf.token}">
<meta name="_csrf_header" th:content="${_csrf.headerName}">
```

```javascript
// fetch 예시
const token = document.querySelector('meta[name="_csrf"]').content;
const header = document.querySelector('meta[name="_csrf_header"]').content;
fetch('/do/post', { method: 'POST', headers: { [header]: token } });
```

> **API 서버**에서 세션/쿠키 인증을 사용하지 않고 **Bearer 토큰**만 쓰면 보통 `csrf.disable()`.

### H-2. XSS (Cross-Site Scripting)
- **출력 인코딩**이 최우선(템플릿 엔진 기본 인코딩 유지, React/Vue는 v-html/dangerouslySetInnerHTML 주의)  
- 입력에서 HTML 허용 시 **화이트리스트 기반 sanitize**(OWASP Java HTML Sanitizer 등)  
- 응답 헤더: `Content-Security-Policy`, `X-XSS-Protection`(구형 브라우저), `X-Content-Type-Options: nosniff`

### H-3. CORS (Cross-Origin Resource Sharing)
- 브라우저의 **타 출처 요청 제한**을 제어  
- 서버에서 명시적으로 허용 도메인/메서드/헤더 지정

```java
@Configuration
public class CorsConfig {
  @Bean
  CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration c = new CorsConfiguration();
    c.setAllowedOrigins(List.of("https://app.example.com"));
    c.setAllowedMethods(List.of("GET","POST","PUT","PATCH","DELETE"));
    c.setAllowedHeaders(List.of("Authorization","Content-Type","X-Requested-With"));
    c.setAllowCredentials(true);
    c.setMaxAge(3600L);
    UrlBasedCorsConfigurationSource s = new UrlBasedCorsConfigurationSource();
    s.registerCorsConfiguration("/**", c);
    return s;
  }
}
```

```java
// SecurityFilterChain과 연결
http.cors(cors -> cors.configurationSource(corsConfigurationSource()));
```

> **주의**: `AllowedOrigins="*"` + `AllowCredentials=true`는 **스펙 위반**(브라우저가 거부). 자격증명 필요하면 **정확한 Origin 화이트리스트**를 써야 한다.

---

## I. 헤더/쿠키 안전 설정

- **쿠키**: `Secure`(HTTPS만), `HttpOnly`(JS 접근 금지), `SameSite=Lax/Strict`  
- **보안 헤더**(Spring Security 기본):  
  - `X-Content-Type-Options: nosniff`  
  - `X-Frame-Options: DENY`(클릭재킹)  
  - `Cache-Control: no-cache, no-store, must-revalidate`(민감 응답)  
  - `Strict-Transport-Security`(HSTS; HTTPS 강제)

```java
http.headers(h -> h
  .httpStrictTransportSecurity(hsts -> hsts
    .includeSubDomains(true).maxAgeInSeconds(31536000))
  .contentSecurityPolicy(csp -> csp
    .policyDirectives("default-src 'self'; frame-ancestors 'none'"))
);
```

---

## J. 로그인/로그아웃 UX와 감사(로그)

### J-1. 성공/실패 핸들러
```java
.formLogin(form -> form
  .successHandler((req,res,auth) -> {
    // 감사 로그, MFA여부 체크, 리다이렉션 정책
    res.sendRedirect("/");
  })
  .failureHandler((req,res,ex) -> {
    // 실패 횟수 증가/백오프/캡차 트리거
    res.sendRedirect("/login?error");
  })
)
```

### J-2. 감사 로깅(MDC)
- 요청 ID/사용자/원격 IP/UA 기록  
- 로그인 시도 성공/실패/로그아웃 시점 이벤트 리스너 등록(`AbstractAuthenticationEvent`)

```java
@Component
public class AuthEventsListener {
  @EventListener
  public void onSuccess(AuthenticationSuccessEvent e) { /* audit */ }
  @EventListener
  public void onFailure(AbstractAuthenticationFailureEvent e) { /* audit */ }
}
```

---

## K. 흔한 요구: “일부 API만 세션, 나머지는 토큰(JWT)”

> 운영에서 자주 만나는 **혼합 전략**: 관리자 콘솔은 **세션+폼 로그인**, 퍼블릭 API는 **JWT**.

- **두 개의 SecurityFilterChain**을 등록, **요청 매칭**으로 분리
```java
@Configuration
public class MultiChainSecurityConfig {

  @Bean
  @Order(1)
  SecurityFilterChain api(HttpSecurity http) throws Exception {
    http.securityMatcher("/api/**")
      .csrf(csrf -> csrf.disable())
      .sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
      .authorizeHttpRequests(a -> a.anyRequest().authenticated())
      .oauth2ResourceServer(oauth2 -> oauth2.jwt()); // 혹은 커스텀 JWT 필터
    return http.build();
  }

  @Bean
  @Order(2)
  SecurityFilterChain web(HttpSecurity http) throws Exception {
    http.securityMatcher("/**")
      .authorizeHttpRequests(a -> a
        .requestMatchers("/login","/css/**","/js/**").permitAll()
        .anyRequest().authenticated())
      .formLogin(Customizer.withDefaults());
    return http.build();
  }
}
```

---

## L. 테스트 — MockMvc + Security 테스트 지원

### L-1. 컨트롤러 테스트에서 사용자 주입
```java
@WebMvcTest(controllers = AdminController.class)
@Import(SecurityConfig.class)
class AdminControllerTest {

  @Autowired MockMvc mvc;

  @Test
  @WithMockUser(username="admin", roles={"ADMIN"})
  void admin_ok() throws Exception {
    mvc.perform(get("/admin/dashboard"))
      .andExpect(status().isOk());
  }

  @Test
  @WithAnonymousUser
  void admin_denied_for_anonymous() throws Exception {
    mvc.perform(get("/admin/dashboard"))
      .andExpect(status().is3xxRedirection()); // 로그인으로 리다이렉트
  }
}
```

### L-2. 메서드 보안 테스트
```java
@SpringBootTest
@AutoConfigureMockMvc
class MethodSecurityTest {

  @Autowired OrderService service;

  @Test
  @WithMockUser(username="alice", roles="USER")
  void preAuthorize_self_access() {
    service.getForUser("alice", 1L); // 통과
  }

  @Test
  @WithMockUser(username="bob", roles="USER")
  void preAuthorize_denied() {
    assertThatThrownBy(() -> service.getForUser("alice", 1L))
      .hasMessageContaining("Access is denied");
  }
}
```

---

## M. 운영 체크리스트

- [ ] **암호 해시**: BCrypt/Argon2, 비용 파라미터 점검, 구형 해시 업그레이드 경로  
- [ ] **락아웃/백오프**: 로그인 실패 카운트/지연/캡차  
- [ ] **세션 고정 공격 방어**: 로그인 성공 시 세션 새로 발급(스프링 기본)  
- [ ] **Remember-me**: 도메인/만료/경로 제한, 민감 기능 재인증  
- [ ] **CSRF**: 세션 웹앱은 활성 유지, API는 stateless면 disable  
- [ ] **CORS**: 정확한 Origin 화이트리스트 + 자격증명 여부 일치  
- [ ] **보안 헤더**: HSTS, CSP, nosniff, frame-ancestors  
- [ ] **감사 로깅**: login success/fail, 권한 거부, 관리자 기능  
- [ ] **비밀번호 재설정**: 단기 토큰, 시도 제한, 기기 로그  
- [ ] **서드파티 연동**: 외부 인증 API 타임아웃/재시도/서킷브레이커

---

## N. 자주 겪는 문제 & 트러블슈팅

1) **White-label 로그인 페이지만 보임**  
   - `formLogin().loginPage("/login")` 설정했는데 뷰 없음 → 템플릿 추가 또는 기본 폼 사용.

2) **정적 리소스까지 인증 요구**  
   - `/css/**`, `/js/**`, `/images/**`를 `permitAll()`로 허용.

3) **403 Forbidden(권한 부족)**  
   - 역할 접두사 확인: `hasRole("ADMIN")`는 내부적으로 `ROLE_ADMIN` 비교. 커스텀 권한은 `hasAuthority("order:read")`.

4) **CSRF 403**  
   - POST/PUT/DELETE인데 CSRF 토큰 헤더/폼 필드가 누락. API 서버(stateless)는 csrf.disable() 고려.

5) **CORS preflight 실패**  
   - `OPTIONS` 메서드 허용/헤더 화이트리스트 미스매치. 프런트 요청 헤더와 정확히 일치시키기.

6) **Remember-me 오작동**  
   - 키 불일치/도메인/경로 오설정. 만료되었거나 `userDetailsService` 미연결.

7) **메서드 보안이 안 먹는다**  
   - `@EnableMethodSecurity` 누락, 프록시 밖에서 자기 자신 호출(self-invocation) 이슈.

---

## O. 한 페이지 요약

- **인증**은 “누구인가”, **인가**는 “무엇을 할 수 있는가”. 스프링은 **필터→Provider→세션/컨텍스트**로 이를 구현.  
- **폼 로그인**은 `UsernamePasswordAuthenticationFilter`가 비밀번호를 검증하고 **세션**으로 인증 상태를 유지, 필요 시 **Remember-me**로 복원.  
- **비밀번호**는 반드시 **BCrypt/Argon2**로 해시, 실패 백오프/락아웃 정책을 갖춘다.  
- **웹 보안**의 3대 기초: **CSRF**(세션 웹앱은 on), **XSS**(출력 인코딩·CSP), **CORS**(정확한 화이트리스트).  
- 경로/메서드 보안을 조합해 **RBAC/권한 기반 정책**을 코드로 명확히 표현하고, **감사 로깅**과 **보안 헤더**로 운영 안전망을 강화한다.
