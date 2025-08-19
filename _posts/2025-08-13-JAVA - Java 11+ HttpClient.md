---
layout: post
title: Java - Java 11+ HttpClient
date: 2025-08-13 20:20:23 +0900
category: Java
---
# Java 11+ `HttpClient` — 현대적 HTTP 클라이언트 API 완전 정리

Java 11에서 도입된 `java.net.http.HttpClient`는 기존 `HttpURLConnection`의 불편한 점을 개선하고,  
**HTTP/2, 비동기 요청, 타임아웃, 리다이렉트, 쿠키, 프록시** 등 현대적인 기능을 지원하는 표준 API입니다.

---

## 1. 특징
- **HTTP/2 지원** — 서버와 협상 후 HTTP/2 또는 HTTP/1.1 사용
- **동기/비동기 요청** — `send()`(동기), `sendAsync()`(비동기)
- **타임아웃 설정** — 연결/요청별로 별도 지정 가능
- **빌더 패턴** — 가독성이 높고 명시적인 구성
- **응답 처리 유연성** — 문자열, 바이트 배열, InputStream 등 다양한 `BodyHandler`
- **리다이렉트 정책**, **쿠키 핸들링**, **프록시 설정** 가능

---

## 2. 기본 구조

```java
HttpClient client = HttpClient.newBuilder()
    .version(HttpClient.Version.HTTP_2)
    .connectTimeout(Duration.ofSeconds(5))
    .followRedirects(HttpClient.Redirect.NORMAL)
    .build();
```
- `.version()` : HTTP 버전 설정 (HTTP_1_1, HTTP_2)
- `.connectTimeout()` : 연결 타임아웃
- `.followRedirects()` : 리다이렉트 정책 (`NEVER`, `NORMAL`, `ALWAYS`)

---

## 3. 요청(Request) 생성

### 3.1 GET 요청
```java
HttpRequest request = HttpRequest.newBuilder()
    .uri(URI.create("https://httpbin.org/get"))
    .GET()
    .header("Accept", "application/json")
    .timeout(Duration.ofSeconds(3))
    .build();
```

### 3.2 POST 요청 (JSON)
```java
String json = "{\"name\":\"Alice\",\"age\":30}";

HttpRequest request = HttpRequest.newBuilder()
    .uri(URI.create("https://httpbin.org/post"))
    .header("Content-Type", "application/json")
    .POST(HttpRequest.BodyPublishers.ofString(json))
    .build();
```

---

## 4. 동기 요청
```java
HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());
System.out.println("Status: " + response.statusCode());
System.out.println("Body: " + response.body());
```

- `BodyHandlers.ofString()` : 응답을 문자열로 반환
- `BodyHandlers.ofByteArray()` : 바이트 배열로 반환
- `BodyHandlers.ofInputStream()` : 스트림으로 반환 (대용량 처리 적합)

---

## 5. 비동기 요청
```java
client.sendAsync(request, HttpResponse.BodyHandlers.ofString())
    .thenApply(HttpResponse::body)
    .thenAccept(System.out::println)
    .exceptionally(e -> { e.printStackTrace(); return null; });
```
- `CompletableFuture` 기반으로 콜백 체이닝 가능
- 논블로킹 방식으로 대량 병렬 요청 처리에 적합

---

## 6. 타임아웃
- **클라이언트 레벨**: `.connectTimeout(Duration.ofSeconds(5))`
- **요청 레벨**: `.timeout(Duration.ofSeconds(3))`
- 읽기/응답 타임아웃은 `HttpRequest`에서 설정

---

## 7. 리다이렉트 정책
```java
HttpClient client = HttpClient.newBuilder()
    .followRedirects(HttpClient.Redirect.ALWAYS)
    .build();
```
- `NEVER` : 리다이렉트 안 함
- `NORMAL` : GET/HEAD만 리다이렉트
- `ALWAYS` : 모든 메서드 리다이렉트

---

## 8. 쿠키 & 프록시
```java
CookieManager cookieManager = new CookieManager();
cookieManager.setCookiePolicy(CookiePolicy.ACCEPT_ALL);

HttpClient client = HttpClient.newBuilder()
    .cookieHandler(cookieManager)
    .proxy(ProxySelector.of(new InetSocketAddress("proxy.example.com", 8080)))
    .build();
```

---

## 9. 파일 업로드 (멀티파트)
JDK 표준엔 멀티파트 전송 메서드가 없으므로, 바운더리와 파트를 직접 구성해야 함.
대규모 업로드 시 `BodyPublishers.ofInputStream()` 사용을 권장.

---

## 10. 베스트 프랙티스
- **타임아웃 필수 지정** — 무한 대기 방지
- **응답 상태 코드 검사** — 2xx 외 상태 처리
- **대용량 응답은 스트리밍 처리**
- **비동기 처리 시 예외 핸들링 필수**
- **멀티스레드 환경에서 HttpClient 재사용** (스레드 세이프)
- **테스트 시 MockWebServer/WireMock 활용**

---

## 11. 간단한 헬퍼 클래스 예시
```java
public class SimpleHttp {
    private final HttpClient client;

    public SimpleHttp() {
        this.client = HttpClient.newBuilder()
            .connectTimeout(Duration.ofSeconds(5))
            .build();
    }

    public String get(String url) throws Exception {
        HttpRequest req = HttpRequest.newBuilder()
            .uri(URI.create(url))
            .GET()
            .build();
        HttpResponse<String> res = client.send(req, HttpResponse.BodyHandlers.ofString());
        if (res.statusCode() / 100 != 2) {
            throw new RuntimeException("HTTP Error: " + res.statusCode());
        }
        return res.body();
    }
}
```

---

### 요약
Java 11+ `HttpClient`는 HTTP/2, 비동기 요청, 타임아웃, 쿠키, 프록시 등 최신 기능을 표준 API로 제공합니다.  
**레거시 `HttpURLConnection`보다 훨씬 간결하고 직관적**이므로, 신규 개발 시 `HttpClient` 사용을 권장합니다.