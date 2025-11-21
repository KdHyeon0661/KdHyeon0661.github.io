---
layout: post
title: Java - Java 11+ HttpClient
date: 2025-08-13 20:20:23 +0900
category: Java
---
# Java 11+ `HttpClient` — 현대적 HTTP 클라이언트 API

## 한눈에 개요 — 왜 HttpClient인가

| 항목 | 내용 |
|---|---|
| 표준성 | JDK 내장 표준 API (의존성 無), 장기 지원 LTS JDK(11/17/21+)에서 안정적 |
| 프로토콜 | **HTTP/2** 자동 협상(서버가 미지원이면 1.1로 폴백) |
| 실행모델 | 동기 `send()` / 비동기 `sendAsync()` (CompletableFuture) |
| 타임아웃 | **연결 타임아웃**(클라이언트 레벨) + **요청별 타임아웃** |
| 본문 | BodyPublisher(요청) / BodyHandler(응답) — 문자열/바이트/스트림/파일 |
| 보안 | SSL/TLS, 사용자 지정 `SSLContext`, **mTLS**, 인증(Authenticator), 프록시 |
| 기타 | 리다이렉트 정책, 쿠키 핸들러, 사용자 지정 Executor, WebSocket API 포함 |

---

## 클라이언트 만들기 — 빌더 패턴과 권장 기본값

```java
import java.net.http.HttpClient;
import java.time.Duration;

HttpClient client = HttpClient.newBuilder()
        .version(HttpClient.Version.HTTP_2)           // HTTP/2 우선 (서버와 ALPN 협상)
        .connectTimeout(Duration.ofSeconds(5))        // TCP 연결 타임아웃 (필수)
        .followRedirects(HttpClient.Redirect.NORMAL)  // GET/HEAD 자동 리다이렉트
        .build();

// 실무 팁: HttpClient는 스레드-세이프. 앱 전체에서 싱글턴 재사용(연결 재사용/HTTP2 세션 이점).
```

### 옵션 요약

| 메서드 | 의미 | 비고 |
|---|---|---|
| `version(HttpClient.Version)` | HTTP_1_1 / HTTP_2 | 서버와 자동 협상 |
| `connectTimeout(Duration)` | TCP Connect 타임아웃 | 네트워크 슬로우 시 필수 |
| `followRedirects(Redirect)` | NEVER / NORMAL / ALWAYS | NORMAL=GET/HEAD만 전환 |
| `proxy(ProxySelector)` | HTTP 프록시 지정 | 기업망 필수 |
| `authenticator(Authenticator)` | 기본/프록시 인증 | Bearer는 헤더로 직접 |
| `cookieHandler(CookieHandler)` | 쿠키 저장/전달 | `CookieManager` 권장 |
| `sslContext(SSLContext)` | TLS 설정 | 자체 CA/mTLS |
| `executor(Executor)` | 내부 풀 커스터마이즈 | 고부하/관측성 제어 |
| `priority(int)` | HTTP/2 우선순위 힌트 | 서버가 무시할 수 있음 |
| `expectContinue(boolean)` | `Expect: 100-continue` | 대용량 POST에 유용 |

---

## 구성 — GET/POST/헤더/타임아웃/쿼리

### GET (쿼리 인코딩·헤더·요청별 타임아웃)

```java
import java.net.URI;
import java.net.URLEncoder;
import static java.nio.charset.StandardCharsets.UTF_8;
import java.net.http.*;

String q = URLEncoder.encode("한글 검색", UTF_8); // 쿼리 값에만 사용
URI uri = URI.create("https://httpbin.org/get?q=" + q);

HttpRequest req = HttpRequest.newBuilder(uri)
        .GET()
        .header("Accept", "application/json")
        .timeout(java.time.Duration.ofSeconds(3)) // 읽기 타임아웃(요청별)
        .build();

HttpResponse<String> res = client.send(req, HttpResponse.BodyHandlers.ofString(UTF_8));
if (res.statusCode() / 100 != 2) {
    throw new IllegalStateException("HTTP " + res.statusCode() + " : " + res.body());
}
System.out.println(res.body());
```

> **주의**: `URLEncoder`는 **쿼리 파라미터** 전용이다. **경로 세그먼트** 인코딩은 별도 유틸(예: RFC 3986 안전 문자만 허용)로 처리하라.

### — Jackson 직렬화

```java
// Gradle (Kotlin)
// implementation("com.fasterxml.jackson.core:jackson-databind:2.18.0")

import com.fasterxml.jackson.databind.ObjectMapper;
import java.util.Map;
import java.net.http.*;
import java.net.URI;
import java.nio.charset.StandardCharsets;

ObjectMapper om = new ObjectMapper();
String json = om.writeValueAsString(Map.of("name","Alice","age",30));

HttpRequest req = HttpRequest.newBuilder(URI.create("https://httpbin.org/post"))
        .header("Content-Type","application/json; charset=UTF-8")
        .POST(HttpRequest.BodyPublishers.ofString(json, StandardCharsets.UTF_8))
        .build();

HttpResponse<String> res = client.send(req, HttpResponse.BodyHandlers.ofString());
System.out.println(res.statusCode());
System.out.println(res.body());
```

### x-www-form-urlencoded

```java
String form = "id=" + URLEncoder.encode("kim", UTF_8)
            + "&pw=" + URLEncoder.encode("s3cr3t", UTF_8);

HttpRequest req = HttpRequest.newBuilder(URI.create("https://example.com/login"))
        .header("Content-Type","application/x-www-form-urlencoded; charset=UTF-8")
        .POST(HttpRequest.BodyPublishers.ofString(form, UTF_8))
        .build();
```

### Expect: 100-continue (대용량 업로드 최적화)

```java
HttpRequest req = HttpRequest.newBuilder(URI.create("https://upload.example.com"))
        .expectContinue(true) // 서버가 100 Continue를 주면 본문 전송 시작
        .POST(HttpRequest.BodyPublishers.ofInputStream(() -> new java.io.FileInputStream("big.bin")))
        .build();
```

---

## — 문자열/바이트/스트림/파일

| 핸들러 | 설명 | 용도 |
|---|---|---|
| `ofString([charset])` | 전체 응답을 문자열로 | 소형 JSON/텍스트 |
| `ofByteArray()` | 전체 응답을 바이트 배열로 | 바이너리 소형 |
| `ofInputStream()` | 응답을 스트림으로 | **대용량/스트리밍** |
| `ofFile(Path)` | 파일로 바로 저장 | 대용량 다운로드 |
| `ofLines()` | 라인 스트림 | NDJSON/로그 |

예: 대용량 다운로드 직접 스트리밍
```java
HttpResponse<java.io.InputStream> r =
    client.send(req, HttpResponse.BodyHandlers.ofInputStream());
try (java.io.InputStream is = r.body();
     java.io.OutputStream os = new java.io.FileOutputStream("out.bin")) {
    is.transferTo(os); // JDK 9+ 편의
}
```

---

## — 팬아웃·합류·예외

```java
HttpRequest req = HttpRequest.newBuilder(URI.create("https://httpbin.org/delay/1"))
        .GET().build();

client.sendAsync(req, HttpResponse.BodyHandlers.ofString())
      .thenApply(HttpResponse::body)
      .thenAccept(System.out::println)
      .exceptionally(ex -> { ex.printStackTrace(); return null; });
```

여러 요청 병렬 처리:
```java
var urls = java.util.List.of(
    "https://httpbin.org/get?i=1",
    "https://httpbin.org/get?i=2",
    "https://httpbin.org/get?i=3");

var futs = urls.stream()
    .map(u -> client.sendAsync(
            HttpRequest.newBuilder(URI.create(u)).GET().build(),
            HttpResponse.BodyHandlers.ofString()))
    .toList();

java.util.concurrent.CompletableFuture.allOf(futs.toArray(java.util.concurrent.CompletableFuture[]::new)).join();
futs.forEach(f -> System.out.println(f.join().statusCode()));
```

---

## 멀티파트 파일 업로드 — **메모리 복사 없이 스트리밍**

표준 API에는 멀티파트 퍼블리셔가 없으므로 **경계(boundary)**/파트를 직접 구성한다.
아래 유틸은 **파일을 스트리밍**하여 대용량 업로드 시 메모리 사용을 최소화한다.

```java
import java.net.http.HttpRequest.BodyPublisher;
import java.net.http.HttpRequest.BodyPublishers;
import java.nio.charset.StandardCharsets;
import java.nio.file.Files;
import java.nio.file.Path;

final class Multipart {
    private final String boundary = "----JavaBoundary" + System.currentTimeMillis();
    String contentType() { return "multipart/form-data; boundary=" + boundary; }
    private static final String CRLF = "\r\n";
    private BodyPublisher str(String s) { return BodyPublishers.ofString(s, StandardCharsets.UTF_8); }

    BodyPublisher field(String name, String value) {
        String part = "--" + boundary + CRLF +
                "Content-Disposition: form-data; name=\"" + name + "\"" + CRLF + CRLF +
                value + CRLF;
        return str(part);
    }
    BodyPublisher file(String field, Path file, String filename, String contentType) throws Exception {
        String pre = "--" + boundary + CRLF +
                "Content-Disposition: form-data; name=\"" + field + "\"; filename=\"" + filename + "\"" + CRLF +
                "Content-Type: " + contentType + CRLF + CRLF;
        String post = CRLF;
        return BodyPublishers.concat(
                str(pre),
                BodyPublishers.ofInputStream(() -> Files.newInputStream(file)),
                str(post));
    }
    BodyPublisher build(BodyPublisher... parts) {
        BodyPublisher end = str("--" + boundary + "--" + CRLF);
        BodyPublisher concat = parts.length == 0 ? end
                : java.util.Arrays.stream(parts).reduce(BodyPublishers::concat).orElseThrow();
        return BodyPublishers.concat(concat, end);
    }
}
```

사용:
```java
Multipart mp = new Multipart();
BodyPublisher body = mp.build(
        mp.field("meta", "hello"),
        mp.file("file", Path.of("hello.txt"), "hello.txt", "text/plain"));

HttpRequest req = HttpRequest.newBuilder(URI.create("https://httpbin.org/post"))
        .header("Content-Type", mp.contentType())
        .POST(body)
        .build();

HttpResponse<String> res = client.send(req, HttpResponse.BodyHandlers.ofString());
```

---

## 리다이렉트 정책 — 보안·호환성

```java
HttpClient c1 = HttpClient.newBuilder().followRedirects(HttpClient.Redirect.NEVER).build();   // 금지
HttpClient c2 = HttpClient.newBuilder().followRedirects(HttpClient.Redirect.NORMAL).build();  // GET/HEAD만
HttpClient c3 = HttpClient.newBuilder().followRedirects(HttpClient.Redirect.ALWAYS).build();  // 모두 허용
```

- `NORMAL`: **POST→GET 전환** 등 표준 관행을 따른다(멱등/안전성 고려).
- 민감 메서드(POST/DELETE 등)의 자동 리다이렉트는 **의도적**이어야 한다.

---

## 쿠키/프록시/인증 — 기업망·보안

```java
import java.net.*;
import java.net.http.*;

CookieManager cm = new CookieManager();
cm.setCookiePolicy(CookiePolicy.ACCEPT_ORIGINAL_SERVER);

HttpClient client = HttpClient.newBuilder()
        .cookieHandler(cm)
        .proxy(ProxySelector.of(new InetSocketAddress("proxy.local", 8080)))
        .authenticator(new Authenticator() {
            @Override protected PasswordAuthentication getPasswordAuthentication() {
                return new PasswordAuthentication("user", "pass".toCharArray()); // 프록시/기본 인증
            }
        })
        .build();

// Bearer 토큰은 헤더로:
HttpRequest req = HttpRequest.newBuilder(URI.create("https://api.example.com"))
        .header("Authorization", "Bearer " + token)
        .GET().build();
```

---

## TLS·mTLS — 자체 CA/클라이언트 인증서

```java
import javax.net.ssl.*;
import java.security.KeyStore;
import java.io.FileInputStream;

// TrustStore (사내 CA)
KeyStore ts = KeyStore.getInstance("JKS");
try (var in = new FileInputStream("truststore.jks")) { ts.load(in, "tsPass".toCharArray()); }
TrustManagerFactory tmf = TrustManagerFactory.getInstance(TrustManagerFactory.getDefaultAlgorithm());
tmf.init(ts);

// KeyStore (클라이언트 인증서, mTLS)
KeyStore ks = KeyStore.getInstance("PKCS12");
try (var in = new FileInputStream("client.p12")) { ks.load(in, "keyPass".toCharArray()); }
KeyManagerFactory kmf = KeyManagerFactory.getInstance(KeyManagerFactory.getDefaultAlgorithm());
kmf.init(ks, "keyPass".toCharArray());

SSLContext ssl = SSLContext.getInstance("TLS");
ssl.init(kmf.getKeyManagers(), tmf.getTrustManagers(), new java.security.SecureRandom());

HttpClient secure = HttpClient.newBuilder().sslContext(ssl).build();
```

> **금지**: 테스트 편의를 위한 “모든 인증서 신뢰” TrustManager는 운영에서 사용하지 말 것.

---

## 재시도·백오프·레이트 리미트 — 멱등 우선

```java
import java.net.http.HttpResponse;
import java.time.Duration;

int max = 3;
Duration backoff = Duration.ofMillis(200);
HttpResponse<String> res = null;

for (int i=1; i<=max; i++) {
    res = client.send(req, HttpResponse.BodyHandlers.ofString());
    int s = res.statusCode();

    if (s == 429 || s/100 == 5) {
        var ra = res.headers().firstValue("Retry-After");
        long waitMs = ra.map(v -> {
            try { return Long.parseLong(v) * 1000L; } catch (NumberFormatException e) { return backoff.toMillis(); }
        }).orElse(backoff.toMillis());
        Thread.sleep(waitMs);
        backoff = backoff.multipliedBy(2);
        continue;
    }
    break;
}
if (res == null || res.statusCode()/100 != 2) throw new IllegalStateException("HTTP 실패");
```

- **멱등 메서드(GET/PUT/DELETE/HEAD)** 위주로 재시도.
- **POST**는 멱등 키(Idempotency-Key) 또는 서버 보장 없으면 신중.

---

## 캐싱·조건부 요청 — ETag/If-None-Match

```java
HttpResponse<String> first = client.send(
    HttpRequest.newBuilder(URI.create("https://example.com/data")).GET().build(),
    HttpResponse.BodyHandlers.ofString());

first.headers().firstValue("ETag").ifPresent(etag -> {
    try {
        HttpResponse<String> second = client.send(
            HttpRequest.newBuilder(URI.create("https://example.com/data"))
                .header("If-None-Match", etag).GET().build(),
            HttpResponse.BodyHandlers.ofString());
        if (second.statusCode() == 304) {
            // 캐시 적중 (변경 없음)
        }
    } catch (Exception e) { throw new RuntimeException(e); }
});
```

---

## 압축·스트리밍 — GZip 수동 처리 예

```java
HttpResponse<java.io.InputStream> r =
    client.send(req, HttpResponse.BodyHandlers.ofInputStream());

try (java.io.InputStream raw = r.body();
     java.io.InputStream is = "gzip".equalsIgnoreCase(r.headers().firstValue("Content-Encoding").orElse(""))
            ? new java.util.zip.GZIPInputStream(raw) : raw;
     java.io.OutputStream os = new java.io.FileOutputStream("out.bin")) {
    is.transferTo(os);
}
```

> 서버/클라이언트의 자동 압축 협상 동작은 환경에 따라 달라질 수 있다. 명시적으로 처리하면 안전하다.

---

## WebSocket — 실시간 양방향

```java
import java.net.http.WebSocket;
import java.net.URI;
import java.util.concurrent.CompletionStage;

WebSocket ws = HttpClient.newHttpClient().newWebSocketBuilder()
        .buildAsync(URI.create("wss://echo.websocket.events"), new WebSocket.Listener() {
            @Override public void onOpen(WebSocket webSocket) {
                System.out.println("OPEN");
                webSocket.request(1);
            }
            @Override public CompletionStage<?> onText(WebSocket webSocket, CharSequence data, boolean last) {
                System.out.println("RECV: " + data);
                webSocket.request(1);
                return null;
            }
            @Override public void onError(WebSocket webSocket, Throwable error) { error.printStackTrace(); }
        }).join();

ws.sendText("hello", true);
Thread.sleep(200);
ws.sendClose(WebSocket.NORMAL_CLOSURE, "bye");
```

---

## 오류·예외 — 분류와 대응

| 유형 | 원인 | 대응 |
|---|---|---|
| `HttpTimeoutException` | 요청 타임아웃 | 타임아웃 조정/재시도 |
| `ConnectException` | 연결 실패(네트워크/프록시) | 네트워크/프록시 점검·재시도 |
| `SSLHandshakeException` | 인증서/프로토콜 문제 | 신뢰/키 저장소·TLS 버전 점검 |
| 3xx | 리다이렉트 | 정책 확인(NORMAL/ALWAYS) |
| 4xx | 클라이언트 오류 | 파라미터/인증/권한 수정 |
| 5xx | 서버 오류 | 백오프 재시도·알림 |

---

## 실행기(Executor)·관측성 — 고부하 대응

```java
var exec = java.util.concurrent.Executors.newFixedThreadPool(
        Math.max(4, Runtime.getRuntime().availableProcessors() * 2),
        r -> { Thread t = new Thread(r, "http-worker"); t.setDaemon(true); return t; });

HttpClient tuned = HttpClient.newBuilder().executor(exec).build();
```

- **전용 스레드풀**로 지연관측/메트릭 수집/제한을 명시적으로 제어.
- 개발 모드: `-Djdk.httpclient.HttpClient.log=all` 로 내부 로그 확인(운영 비권장).

---

## 간단 헬퍼 — 안전한 GET/POST 래퍼

```java
import java.net.http.*;
import java.net.URI;
import java.nio.charset.StandardCharsets;
import java.io.IOException;

public final class SimpleHttp {
    private final HttpClient client;
    public SimpleHttp(HttpClient client) { this.client = client; }

    public String getJson(String url) throws IOException, InterruptedException {
        HttpRequest req = HttpRequest.newBuilder(URI.create(url))
                .header("Accept","application/json")
                .GET().build();
        HttpResponse<String> res = client.send(req, HttpResponse.BodyHandlers.ofString(StandardCharsets.UTF_8));
        if (res.statusCode()/100 != 2) throw new IOException("HTTP " + res.statusCode() + ": " + res.body());
        return res.body();
    }

    public String postJson(String url, String json) throws IOException, InterruptedException {
        HttpRequest req = HttpRequest.newBuilder(URI.create(url))
                .header("Content-Type","application/json; charset=UTF-8")
                .POST(HttpRequest.BodyPublishers.ofString(json, StandardCharsets.UTF_8))
                .build();
        HttpResponse<String> res = client.send(req, HttpResponse.BodyHandlers.ofString(StandardCharsets.UTF_8));
        if (res.statusCode()/100 != 2) throw new IOException("HTTP " + res.statusCode() + ": " + res.body());
        return res.body();
    }
}
```

---

## 테스트 전략 — Mock 서버로 결정성 확보

- **단위 테스트**: HTTP 호출부를 인터페이스로 감싸 Mock/Fake 주입
- **통합 테스트**: **OkHttp MockWebServer** 또는 **WireMock** 사용
- 타임아웃/리다이렉트/에러(4xx/5xx)/재시도/압축 등 **경계 상황**을 케이스화

```java
// MockWebServer 예(요약)
mockWebServer.enqueue(new okhttp3.mockwebserver.MockResponse()
        .setResponseCode(200)
        .setBody("{\"ok\":true}")
        .addHeader("Content-Type","application/json"));

String baseUrl = mockWebServer.url("/").toString(); // 클라이언트 베이스 URL로 주입
```

---

## 레거시 비교 — `HttpURLConnection` (참고용)

```java
import java.net.*;
import java.io.*;
import java.nio.charset.StandardCharsets;

URL url = new URL("https://httpbin.org/get");
HttpURLConnection conn = (HttpURLConnection) url.openConnection();
conn.setRequestMethod("GET");
conn.setConnectTimeout(5000);
conn.setReadTimeout(5000);

int code = conn.getResponseCode();
try (InputStream is = (code >= 200 && code < 300) ? conn.getInputStream() : conn.getErrorStream()) {
    String body = new String(is.readAllBytes(), StandardCharsets.UTF_8);
    System.out.println(code + " " + body);
}
conn.disconnect();
```

> 신규 개발은 **HttpClient** 권장. 레거시 유지·간단 스크립팅 외에는 이점이 적다.

---

## 체크리스트 — 운영 품질을 높이기 위한 최소 기준

1. **HttpClient 싱글턴** 재사용(연결·세션 재활용)
2. **타임아웃**: `connectTimeout` + 요청별 `.timeout()` **반드시** 지정
3. **에러 처리**: 2xx 외 상태, 본문/코릴레이션ID 로깅(민감정보 제외)
4. **인코딩**: 쿼리=URLEncoder, 경로=세그먼트 인코딩, 바디=명시적 charset
5. **대용량**: `ofInputStream`/`ofFile`, 멀티파트 스트리밍(복사 최소화)
6. **재시도**: 429/5xx만, **백오프** + `Retry-After` 존중, 멱등성 원칙
7. **보안**: TLS 저장소·버전, mTLS 필요 시 `SSLContext`, 비밀 관리(환경/키체인/볼트)
8. **관측성**: 요청/응답 요약 로그(개발), 성공률·지연·재시도·타임아웃 메트릭
9. **프록시/쿠키**: 기업망 정책 반영, 도메인 스코프/보안 쿠키 옵션 점검
10. **테스트**: Mock 서버로 결정성, 카나리아/서킷브레이커(필요 시) 검토

---

## 고급 주제 메모

- **HTTP/2 서버 푸시**: 일부 서버/브라우저에서 비권장·중단 추세. JDK API는 핸들러가 있으나 실제 활용은 드묾.
- **인터셉터**: 표준 HttpClient는 전용 인터셉터 API 없음. 공통 래퍼/팩토리/데코레이터로 해결(로깅/서명/재시도).
- **HTTP/3**: 현재 표준 HttpClient는 HTTP/3(QUIC) 미지원. 필요 시 게이트웨이/역프록시 활용.

---

### 결론

**Java 11+ `HttpClient`**는 현대적 HTTP 클라이언트를 표준으로 제공한다.
이 문서의 예제와 체크리스트를 적용하면 **HTTP/2·비동기·타임아웃·보안·스트리밍** 요구사항을 무의존으로 충족할 수 있다.
멀티파트/관측/재시도 정책을 래퍼로 정리하고, 테스트는 **Mock 서버**로 결정성을 확보하라. 그러면 레거시를 넘어 **간결하면서도 견고한 네트워크 계층**을 즉시 구축할 수 있다.
