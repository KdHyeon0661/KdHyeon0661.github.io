---
layout: post
title: Java - REST API 클라이언트
date: 2025-08-18 14:25:23 +0900
category: Java
---
# REST API 클라이언트

## 빠른 개요 — 언제 무엇을 쓸까?

| 상황 | 추천 라이브러리 | 이유/특징 |
|---|---|---|
| **표준, 의존 최소화** | **JDK `HttpClient`** | JDK 11+ 표준, HTTP/2, 동기/비동기, 의존성 0 |
| **세밀한 설정/풀/프록시/전통** | **Apache HttpClient** | 풍부한 구성/인터셉터/인증/프록시/풀링 |
| **간결한 API + 인터셉터** | **OkHttp** | 직관적, 커넥션 풀/인터셉터/웹소켓/(옵션) 핀닝 |
| **리액티브/대규모 동시성** | **Spring WebClient** | 논블로킹(Reactor), 필터/코덱/보안 통합 |

> 신규 프로젝트에서 **표준만으로 충분**하면 `HttpClient`. **스프링 기반**이고 비동기/리액티브가 필요하면 `WebClient`.
> 세밀 제어가 필요하면 Apache, 간결함/인터셉터 생태계면 OkHttp.

---

## REST 클라이언트 기본기

- **HTTP 메서드/의미**: GET(조회), POST(생성), PUT/PATCH(부분/전체 수정), DELETE(삭제)
- **상태 코드**: 2xx 성공, 4xx 클라이언트 오류, 5xx 서버 오류
- **헤더**: `Accept`, `Content-Type`, `Authorization`, `User-Agent`, `If-None-Match`(캐시/Etag), `Retry-After`(재시도 힌트)
- **본문**: JSON(권장), XML/Proto 등
- **무상태성**: 서버 상태를 세션에 의존하지 않게, 토큰/키로 인증

---

## 의존성 (예시)

### Maven

```xml
<dependencies>
  <!-- JSON 직렬화/역직렬화 -->
  <dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
    <version>2.17.2</version>
  </dependency>

  <!-- OkHttp (선택) -->
  <dependency>
    <groupId>com.squareup.okhttp3</groupId>
    <artifactId>okhttp</artifactId>
    <version>4.12.0</version>
  </dependency>

  <!-- Apache HttpClient (선택) -->
  <dependency>
    <groupId>org.apache.httpcomponents.client5</groupId>
    <artifactId>httpclient5</artifactId>
    <version>5.3.1</version>
  </dependency>

  <!-- Spring WebClient (선택, Spring Boot 미사용 순수 WebFlux) -->
  <dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-webflux</artifactId>
    <version>6.1.12</version>
  </dependency>

  <!-- WireMock (테스트용, 선택) -->
  <dependency>
    <groupId>com.github.tomakehurst</groupId>
    <artifactId>wiremock-jre8</artifactId>
    <version>2.35.2</version>
    <scope>test</scope>
  </dependency>
</dependencies>
```

### Gradle (Kotlin DSL)

```kotlin
dependencies {
    implementation("com.fasterxml.jackson.core:jackson-databind:2.17.2")

    // 선택
    implementation("com.squareup.okhttp3:okhttp:4.12.0")
    implementation("org.apache.httpcomponents.client5:httpclient5:5.3.1")
    implementation("org.springframework:spring-webflux:6.1.12")

    testImplementation("com.github.tomakehurst:wiremock-jre8:2.35.2")
}
```

> 버전은 예시다. 실프로젝트에서는 **BOM(플랫폼)** 또는 **버전 카탈로그**로 중앙 관리하라.

---

## 데이터 모델 & 공통 유틸

### DTO는 `record`로 간결하게

```java
public record Post(long id, long userId, String title, String body) {}
```

### Jackson `ObjectMapper` 공용 인스턴스

```java
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.SerializationFeature;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;

public final class Jsons {
  public static final ObjectMapper MAPPER = new ObjectMapper()
      .registerModule(new JavaTimeModule())
      .disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);
}
```

---

## JDK 표준 `HttpClient` — 기본/비동기/타임아웃/백오프

### 설정과 간단 GET

```java
import java.net.http.*;
import java.net.*;
import java.time.Duration;

HttpClient client = HttpClient.newBuilder()
    .connectTimeout(Duration.ofSeconds(5))
    .followRedirects(HttpClient.Redirect.NORMAL)
    .version(HttpClient.Version.HTTP_2) // 서버 지원 시 HTTP/2
    .build();

HttpRequest req = HttpRequest.newBuilder()
    .uri(URI.create("https://jsonplaceholder.typicode.com/posts/1"))
    .header("Accept", "application/json")
    .timeout(Duration.ofSeconds(10))    // 요청별 타임아웃
    .GET()
    .build();

HttpResponse<String> res = client.send(req, HttpResponse.BodyHandlers.ofString());
if (res.statusCode() / 100 == 2) {
    Post p = Jsons.MAPPER.readValue(res.body(), Post.class);
    System.out.println("title=" + p.title());
} else {
    throw new RuntimeException("HTTP " + res.statusCode());
}
```

### POST(JSON) — idempotency-key/추적 헤더

```java
String body = Jsons.MAPPER.writeValueAsString(new Post(0, 1, "Hello", "World"));

HttpRequest req = HttpRequest.newBuilder()
    .uri(URI.create("https://api.example.com/posts"))
    .header("Content-Type", "application/json")
    .header("Idempotency-Key", java.util.UUID.randomUUID().toString())
    .header("X-Request-Id", "client-abc-123")
    .POST(HttpRequest.BodyPublishers.ofString(body))
    .build();
HttpResponse<String> res = client.send(req, HttpResponse.BodyHandlers.ofString());
```

### 비동기 — `CompletableFuture`

```java
client.sendAsync(req, HttpResponse.BodyHandlers.ofString())
      .thenApply(HttpResponse::body)
      .thenApply(b -> Jsons.MAPPER.readValue(b, Post.class))
      .thenAccept(p -> System.out.println(p.title()))
      .exceptionally(ex -> { ex.printStackTrace(); return null; })
      .join();
```

### 지수 백오프 재시도(간단 구현)

```java
static <T> T withRetry(java.util.concurrent.Callable<T> op, int max, long baseMillis) throws Exception {
    int attempt = 0;
    while (true) {
        try {
            return op.call();
        } catch (Exception e) {
            if (++attempt > max) throw e;
            long delay = (long) (baseMillis * Math.pow(2, attempt - 1));
            Thread.sleep(Math.min(delay, 5_000)); // 상한 5s
        }
    }
}

// 사용
Post p = withRetry(() -> {
    HttpResponse<String> res = client.send(req, HttpResponse.BodyHandlers.ofString());
    if (res.statusCode() >= 500 || res.statusCode() == 429) {
        // Retry-After 헤더가 있으면 우선 고려
        String ra = res.headers().firstValue("Retry-After").orElse(null);
        if (ra != null) Thread.sleep(1000L * Long.parseLong(ra));
        throw new RuntimeException("retryable: " + res.statusCode());
    }
    if (res.statusCode() / 100 != 2) throw new IllegalStateException("HTTP " + res.statusCode());
    return Jsons.MAPPER.readValue(res.body(), Post.class);
}, 3, 200);
```

### 멀티파트 업로드(간단 빌더)

```java
public static HttpRequest multipartRequest(URI uri, Map<String,String> fields, String fileField, String filename, byte[] file) {
    String boundary = "----JavaClientBoundary" + System.currentTimeMillis();
    var byteArrays = new java.util.ArrayList<byte[]>();

    fields.forEach((k,v) -> {
        String part = "--" + boundary + "\r\n"
            + "Content-Disposition: form-data; name=\"" + k + "\"\r\n\r\n"
            + v + "\r\n";
        byteArrays.add(part.getBytes(java.nio.charset.StandardCharsets.UTF_8));
    });

    String fileHead = "--" + boundary + "\r\n"
        + "Content-Disposition: form-data; name=\"" + fileField + "\"; filename=\"" + filename + "\"\r\n"
        + "Content-Type: application/octet-stream\r\n\r\n";
    byteArrays.add(fileHead.getBytes(java.nio.charset.StandardCharsets.UTF_8));
    byteArrays.add(file);
    byteArrays.add("\r\n--".concat(boundary).concat("--\r\n").getBytes(java.nio.charset.StandardCharsets.UTF_8));

    var body = HttpRequest.BodyPublishers.ofByteArrays(byteArrays);

    return HttpRequest.newBuilder(uri)
        .header("Content-Type", "multipart/form-data; boundary=" + boundary)
        .POST(body).build();
}
```

---

## OkHttp — 간결한 API, 인터셉터, 핀닝(옵션)

### 기본 GET/POST/타임아웃

```java
import okhttp3.*;

OkHttpClient ok = new OkHttpClient.Builder()
    .connectTimeout(java.time.Duration.ofSeconds(5))
    .readTimeout(java.time.Duration.ofSeconds(10))
    .addInterceptor(chain -> {
        Request r = chain.request().newBuilder()
            .header("User-Agent", "demo-client/1.0")
            .build();
        long t0 = System.nanoTime();
        Response res = chain.proceed(r);
        long dt = (System.nanoTime() - t0)/1_000_000;
        System.out.println(res.code() + " " + r.url() + " (" + dt + "ms)");
        return res;
    })
    .build();

Request get = new Request.Builder().url("https://jsonplaceholder.typicode.com/posts/1").build();
try (Response res = ok.newCall(get).execute()) {
    Post p = Jsons.MAPPER.readValue(res.body().string(), Post.class);
}

Request post = new Request.Builder()
    .url("https://api.example.com/posts")
    .post(RequestBody.create(
        Jsons.MAPPER.writeValueAsBytes(new Post(0,1,"t","b")),
        MediaType.get("application/json")))
    .build();
```

### 인증 토큰 주입 인터셉터

```java
class BearerAuth implements Interceptor {
  private volatile String token;
  BearerAuth(String token){ this.token = token; }
  public void update(String t){ this.token = t; }

  @Override public Response intercept(Chain chain) throws java.io.IOException {
    Request req = chain.request().newBuilder()
        .header("Authorization", "Bearer " + token)
        .build();
    Response res = chain.proceed(req);
    if (res.code() == 401) {
      // 여기서 토큰 갱신 로직(동기/락 주의)
    }
    return res;
  }
}
```

### 인증서 핀닝(선택)

```java
CertificatePinner pinner = new CertificatePinner.Builder()
    .add("api.example.com", "sha256/AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=")
    .build();
OkHttpClient secure = ok.newBuilder().certificatePinner(pinner).build();
```

---

## Apache HttpClient — 풀/프록시/정교한 구성

```java
import org.apache.hc.client5.http.classic.methods.HttpGet;
import org.apache.hc.client5.http.impl.classic.*;
import org.apache.hc.client5.http.config.RequestConfig;
import org.apache.hc.core5.util.Timeout;

PoolingHttpClientConnectionManager cm = new PoolingHttpClientConnectionManager();
cm.setMaxTotal(200);
cm.setDefaultMaxPerRoute(50);

RequestConfig rc = RequestConfig.custom()
    .setConnectTimeout(Timeout.ofSeconds(5))
    .setResponseTimeout(Timeout.ofSeconds(10))
    .build();

try (CloseableHttpClient cli = HttpClients.custom()
    .setConnectionManager(cm)
    .setDefaultRequestConfig(rc)
    .addRequestInterceptorLast((req, ctx) -> req.addHeader("Accept", "application/json"))
    .build()) {

  HttpGet get = new HttpGet("https://jsonplaceholder.typicode.com/posts/1");

  try (CloseableHttpResponse res = cli.execute(get)) {
      String json = new String(res.getEntity().getContent().readAllBytes(), java.nio.charset.StandardCharsets.UTF_8);
      Post p = Jsons.MAPPER.readValue(json, Post.class);
  }
}
```

---

## Spring **WebClient** — 리액티브/필터/에러 매핑

### 기본 구성

```java
import org.springframework.web.reactive.function.client.*;

WebClient client = WebClient.builder()
    .baseUrl("https://api.example.com")
    .defaultHeader("Accept", "application/json")
    .clientConnector(new ReactorClientHttpConnector()) // 기본
    .build();

// GET
Post p = client.get().uri("/posts/{id}", 1)
    .retrieve()
    .onStatus(s -> s.is4xxClientError(), resp -> resp.createException().flatMap(Mono::error))
    .onStatus(s -> s.is5xxServerError(), resp -> resp.createException().flatMap(Mono::error))
    .bodyToMono(Post.class)
    .block(); // 데모용. 실제로는 논블로킹 체인으로 사용하는 것이 이상적.
```

### 필터(인터셉터 유사)로 공통 헤더/로깅/재시도

```java
ExchangeFilterFunction auth = (req, next) -> {
    ClientRequest r = ClientRequest.from(req)
        .header("Authorization", "Bearer " + getToken())
        .build();
    long t0 = System.nanoTime();
    return next.exchange(r).doOnSuccess(res ->
        System.out.println(res.statusCode() + " " + req.url() + " (" + (System.nanoTime()-t0)/1_000_000 + "ms)"));
};

WebClient wc = WebClient.builder()
    .baseUrl("https://api.example.com")
    .filter(auth)
    .build();
```

### 대용량/스트리밍(JSON Lines, SSE)

```java
Flux<String> lines = wc.get().uri("/stream")
    .accept(org.springframework.http.MediaType.APPLICATION_NDJSON)
    .retrieve()
    .bodyToFlux(String.class);
// lines.subscribe(...)
```

> Spring Security를 쓴다면 `oauth2Client` 설정으로 **자동 토큰 주입** 가능.

---

## 에러 매핑(예외 계층) — 실무 패턴

```java
sealed interface HttpProblem extends RuntimeException permits BadRequest, Unauthorized, TooManyRequests, ServerError { }
final class BadRequest extends RuntimeException { BadRequest(String m){ super(m);} }
final class Unauthorized extends RuntimeException { Unauthorized(){ super("unauthorized"); } }
final class TooManyRequests extends RuntimeException { TooManyRequests(String ra){ super("rate limited, retry-after="+ra);} }
final class ServerError extends RuntimeException { ServerError(int sc){ super("server error: "+sc);} }

static void ensure2xx(HttpResponse<?> res) {
    int sc = res.statusCode();
    if (sc/100 == 2) return;
    if (sc == 400) throw new BadRequest("bad request");
    if (sc == 401) throw new Unauthorized();
    if (sc == 429) throw new TooManyRequests(res.headers().firstValue("Retry-After").orElse("unknown"));
    if (sc/100 == 5) throw new ServerError(sc);
    throw new RuntimeException("unexpected http status " + sc);
}
```

---

## 인증 — API Key / OAuth 2.0(Client Credentials)

### API Key (단순 헤더)

```java
HttpRequest req = HttpRequest.newBuilder()
    .uri(URI.create("https://api.example.com/data"))
    .header("X-API-Key", System.getenv("API_KEY"))
    .GET().build();
```

### OAuth2 Client Credentials (표준 플로우)

```java
record TokenResponse(String access_token, String token_type, long expires_in) {}

static String fetchToken(HttpClient cli, URI tokenEndpoint, String clientId, String clientSecret) throws Exception {
    String body = "grant_type=client_credentials&client_id=" + URLEncoder.encode(clientId, "UTF-8")
                + "&client_secret=" + URLEncoder.encode(clientSecret, "UTF-8");
    HttpRequest req = HttpRequest.newBuilder(tokenEndpoint)
        .header("Content-Type", "application/x-www-form-urlencoded")
        .POST(HttpRequest.BodyPublishers.ofString(body))
        .build();
    HttpResponse<String> res = cli.send(req, HttpResponse.BodyHandlers.ofString());
    ensure2xx(res);
    return Jsons.MAPPER.readValue(res.body(), TokenResponse.class).access_token();
}

// 사용: Authorization 헤더에 Bearer {token}
```

> 토큰 캐시/만료 갱신은 **동시성 제어**(원자적 갱신)와 **선제 갱신 임계(예: 만료 60초 전)**를 적용하라.

---

## **Typed Client** 만들기 — 재사용 가능한 API 모듈

```java
interface PostsClient {
    Post get(long id) throws Exception;
    java.util.List<Post> list(int page, int size) throws Exception;
    Post create(Post p) throws Exception;
}

final class HttpPostsClient implements PostsClient {
    private final HttpClient cli;
    private final URI base;

    HttpPostsClient(URI base, HttpClient cli){ this.base = base; this.cli = cli; }

    @Override public Post get(long id) throws Exception {
        HttpRequest req = HttpRequest.newBuilder(base.resolve("/posts/" + id))
            .header("Accept", "application/json")
            .GET().build();
        HttpResponse<String> res = cli.send(req, HttpResponse.BodyHandlers.ofString());
        ensure2xx(res);
        return Jsons.MAPPER.readValue(res.body(), Post.class);
    }

    @Override public java.util.List<Post> list(int page, int size) throws Exception {
        URI uri = URI.create(base + "/posts?page=" + page + "&size=" + size);
        HttpResponse<String> res = cli.send(HttpRequest.newBuilder(uri).GET().build(), HttpResponse.BodyHandlers.ofString());
        ensure2xx(res);
        return Jsons.MAPPER.readValue(res.body(),
            Jsons.MAPPER.getTypeFactory().constructCollectionType(java.util.List.class, Post.class));
    }

    @Override public Post create(Post p) throws Exception {
        String json = Jsons.MAPPER.writeValueAsString(p);
        HttpRequest req = HttpRequest.newBuilder(base.resolve("/posts"))
            .header("Content-Type", "application/json")
            .header("Idempotency-Key", java.util.UUID.randomUUID().toString())
            .POST(HttpRequest.BodyPublishers.ofString(json)).build();
        HttpResponse<String> res = cli.send(req, HttpResponse.BodyHandlers.ofString());
        ensure2xx(res);
        return Jsons.MAPPER.readValue(res.body(), Post.class);
    }
}
```

> 이 클라이언트를 **서비스 레이어에 주입**하면 테스트에서 **더블(Stub/Mock/WireMock)**로 쉽게 교체 가능하다.

---

## **Resilience** — 재시도/백오프/서킷브레이커

### 간단 재시도(상단 4.4 참고) + Retry-After 존중
### Resilience4j로 데코레이터(선택)

```java
// pseudo: Decorators.ofSupplier(() -> client.get(1)).withCircuitBreaker(cb).withRetry(retry).get();
```
> 재시도는 **멱등 연산(GET/PUT/DELETE)** 중심으로. POST는 **Idempotency-Key**가 있는 엔드포인트에서만.

---

## 로깅/추적 — 요청/응답/코릴레이션

- **코릴레이션 ID**: `X-Request-Id`를 요청/응답에 통일해 로그 상관.
- **민감정보 마스킹**: Authorization/쿠키/개인정보는 로깅 금지/마스킹.
- **OkHttp**: 인터셉터에서 로깅.
- **JDK HttpClient**: 직접 래핑/SLF4J 로그.

---

## 테스트 — WireMock / MockWebServer / 통합

### WireMock(JUnit 5)

```java
import static com.github.tomakehurst.wiremock.client.WireMock.*;
import com.github.tomakehurst.wiremock.WireMockServer;
import org.junit.jupiter.api.*;

class PostsClientTest {
  static WireMockServer wm;
  @BeforeAll static void up(){ wm = new WireMockServer(0); wm.start(); }
  @AfterAll static void down(){ wm.stop(); }

  @Test
  void get_ok() throws Exception {
    wm.stubFor(get(urlEqualTo("/posts/1"))
      .willReturn(aResponse()
        .withStatus(200)
        .withHeader("Content-Type","application/json")
        .withBody("{\"id\":1,\"userId\":1,\"title\":\"t\",\"body\":\"b\"}")));

    HttpClient cli = HttpClient.newHttpClient();
    PostsClient api = new HttpPostsClient(URI.create("http://localhost:"+wm.port()), cli);
    Post p = api.get(1);
    org.junit.jupiter.api.Assertions.assertEquals(1, p.id());
  }
}
```

### OkHttp MockWebServer

```java
MockWebServer server = new MockWebServer();
server.enqueue(new MockResponse().setBody("{\"id\":1,\"userId\":1,\"title\":\"t\",\"body\":\"b\"}").setHeader("Content-Type","application/json"));
server.start();
// ... 클라이언트의 baseUrl을 server.url("/").toString() 로 지정
server.shutdown();
```

> 단위 테스트는 **빠르고 결정적**이어야 한다. 통합 테스트는 **별 태그/소스셋**으로 분리.

---

## 고급 주제 — 캐시/압축/페이징/HATEOAS/프록시

- **HTTP 캐시**: `ETag`/`If-None-Match`, `Last-Modified`/`If-Modified-Since`로 304 핸들링.
- **압축**: `Accept-Encoding: gzip` + 응답 자동 압축 해제(대부분 라이브러리 기본).
- **페이징**: `Link` 헤더(`rel="next"`) 파싱, 커서 기반 토큰 유지.
- **HATEOAS**: 응답 링크로 후속 액션을 노출, 클라이언트는 링크 따라가기.
- **프록시/기업망**: Apache/OkHttp/JDK 모두 HTTP/SOCKS 프록시 설정 가능.

---

## 보안 체크리스트

- [ ] **TLS 필수**, 호스트 검증/루트 CA 신뢰 고정.
- [ ] (필요 시) **인증서 핀닝**으로 중간자 공격 억제.
- [ ] **토큰/키 보호**: 환경변수/시크릿 스토어, 로그/예외에 노출 금지.
- [ ] **재시도 시 본문 재전송 크기** 제한(대용량 업로드는 재시도 방식 분리).
- [ ] **Idempotency-Key** + 서버 지원 확인 후 POST 재시도.

---

## 운영/성능 팁

- **타임아웃 3종** 구분: 연결/응답(소켓 읽기)/요청(전체) — **기본값 금지**.
- **커넥션 풀**: OkHttp/Apache는 풀 사이즈/유휴 시간 조정.
- **HTTP/2**: 멀티플렉싱으로 성능 향상(서버 지원 가정).
- **대량 호출**: 배치/압축/필드 선택(서버 지원 시 `fields=`)으로 응답 크기 감소.
- **가용성**: DNS 다중 IP, 리트라이 시 **서킷 브레이커**/**지수 백오프 + Jitter** 적용.

---

## 미니 레퍼런스 — 코드 조각 모음

### 쿼리 파라미터 안전 조립

```java
static URI uri(String base, Map<String,String> q) {
  StringBuilder sb = new StringBuilder(base);
  if (!q.isEmpty()) sb.append("?");
  q.forEach((k,v) -> {
    if (sb.charAt(sb.length()-1)!='?') sb.append("&");
    try {
      sb.append(URLEncoder.encode(k,"UTF-8")).append("=").append(URLEncoder.encode(v,"UTF-8"));
    } catch (Exception e) { throw new RuntimeException(e); }
  });
  return URI.create(sb.toString());
}
```

### 파일 다운로드(스트리밍)

```java
HttpResponse<java.io.InputStream> res = client.send(req, HttpResponse.BodyHandlers.ofInputStream());
ensure2xx(res);
try (var in = res.body(); var out = java.nio.file.Files.newOutputStream(java.nio.file.Path.of("out.bin"))) {
  in.transferTo(out);
}
```

---

## 최종 체크리스트

- [ ] **라이브러리 선택**이 요구(동기/비동기, 스프링 여부, 운영 제약)에 부합하는가?
- [ ] **타임아웃/재시도/백오프**와 **에러 매핑**이 일관적으로 구현됐는가?
- [ ] **인증**(API Key/OAuth2)과 **토큰 갱신**이 안전한가(락/캐시/만료 여유)?
- [ ] **JSON 매핑**(nullable/필드명/시간대) 규칙이 통일되어 있는가?
- [ ] **로깅/마스킹/추적**이 준비되어 있는가?
- [ ] **테스트**: 단위(WireMock/MockWebServer) + 통합(별 소스셋)으로 분리되어 있는가?
- [ ] **운영/성능**: 풀/HTTP/2/압축/캐시 활용 점검했는가?

---

## 요약

- 표준 `HttpClient`만으로도 **의존성 없이** 상당수 요구사항을 충족한다.
- 스프링 환경/리액티브 요구는 **WebClient**가 자연스러운 선택.
- 세밀한 네트워크 제어/전통 자산이 있다면 **Apache/OkHttp**도 유효하다.
- **타임아웃·에러 매핑·재시도·인증·테스트**를 표준화하면, 클라이언트 모듈은 재사용 가능한 **플랫폼 구성요소**가 된다.
