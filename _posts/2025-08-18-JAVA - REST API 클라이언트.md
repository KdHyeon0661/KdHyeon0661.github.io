---
layout: post
title: Java - REST API 클라이언트
date: 2025-08-18 14:25:23 +0900
category: Java
---
# REST API 클라이언트 — 개념부터 구현까지

Java에서 **REST API 클라이언트**를 작성하는 방법은 매우 다양하며, 표준 라이브러리(`HttpURLConnection`, `HttpClient`)부터 Apache HttpClient, OkHttp, Spring의 `RestTemplate` / `WebClient` 등 고급 라이브러리까지 선택지가 많습니다.  
아래에서는 개념, 기본 동작, 주요 라이브러리, 구현 예제를 순서대로 설명합니다.

---

## 1. REST API 클라이언트 개념

### 1.1 REST API란?
- **REST(Representational State Transfer)**: HTTP를 기반으로 한 아키텍처 스타일
- **클라이언트-서버 구조** + **무상태(stateless)** + **자원 기반(URI)** + **표준 HTTP 메서드(GET, POST, PUT, DELETE)** 활용
- 응답은 보통 **JSON** 또는 **XML** 형식

### 1.2 REST API 클라이언트란?
- 서버의 REST API 엔드포인트에 요청을 보내고 응답을 받아 **데이터를 처리**하는 프로그램/코드
- 주요 기능:
  - HTTP 요청 생성
  - 요청 헤더, 쿼리 파라미터, 본문 설정
  - 응답 상태 코드/헤더/본문 처리
  - 예외/에러 처리

---

## 2. HTTP 요청과 응답 구조

### 2.1 요청 구성 요소
- **HTTP 메서드**: GET(조회), POST(생성), PUT(수정), DELETE(삭제)
- **URI**: 예) `https://api.example.com/users`
- **헤더**: `Content-Type`, `Accept`, `Authorization` 등
- **본문(body)**: JSON, XML, 폼 데이터 등 (GET은 보통 없음)

### 2.2 응답 구성 요소
- **상태 코드**: 200(성공), 201(생성됨), 400(잘못된 요청), 404(없음), 500(서버 오류)
- **헤더**: 응답 정보(`Content-Type`, `Cache-Control` 등)
- **본문**: 요청한 데이터(JSON/XML 등)

---

## 3. Java에서 REST API 클라이언트 구현 방식

### 3.1 표준 라이브러리 — `HttpURLConnection`
```java
URL url = new URL("https://jsonplaceholder.typicode.com/posts/1");
HttpURLConnection conn = (HttpURLConnection) url.openConnection();
conn.setRequestMethod("GET");
conn.setRequestProperty("Accept", "application/json");

if (conn.getResponseCode() == 200) {
    try (BufferedReader br = new BufferedReader(new InputStreamReader(conn.getInputStream()))) {
        String line;
        while ((line = br.readLine()) != null) {
            System.out.println(line);
        }
    }
}
conn.disconnect();
```
- 장점: 표준 JDK 제공
- 단점: 코드가 장황하고 에러 처리 복잡

---

### 3.2 Java 11+ `HttpClient`
```java
HttpClient client = HttpClient.newHttpClient();
HttpRequest request = HttpRequest.newBuilder()
    .uri(URI.create("https://jsonplaceholder.typicode.com/posts"))
    .header("Content-Type", "application/json")
    .POST(HttpRequest.BodyPublishers.ofString("{\"title\":\"Hello\",\"body\":\"World\"}"))
    .build();

HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());
System.out.println(response.statusCode());
System.out.println(response.body());
```
- 장점: 비동기 지원(`sendAsync`), 체이닝 API
- 단점: Java 11 이상 필요

---

### 3.3 Apache HttpClient
```java
CloseableHttpClient httpClient = HttpClients.createDefault();
HttpGet request = new HttpGet("https://jsonplaceholder.typicode.com/posts/1");
request.addHeader("Accept", "application/json");

try (CloseableHttpResponse response = httpClient.execute(request)) {
    String result = EntityUtils.toString(response.getEntity());
    System.out.println(result);
}
```
- 장점: 풍부한 기능, 세부 제어 가능
- 단점: 외부 라이브러리 의존성 필요

---

### 3.4 OkHttp (Square)
```java
OkHttpClient client = new OkHttpClient();

Request request = new Request.Builder()
    .url("https://jsonplaceholder.typicode.com/posts/1")
    .build();

try (Response response = client.newCall(request).execute()) {
    System.out.println(response.body().string());
}
```
- 장점: 간결, 비동기·동기 모두 지원
- 단점: 외부 라이브러리 필요

---

### 3.5 Spring `RestTemplate` / `WebClient`
```java
// RestTemplate 예제
RestTemplate restTemplate = new RestTemplate();
String url = "https://jsonplaceholder.typicode.com/posts/{id}";
Map<String, String> params = Map.of("id", "1");

String result = restTemplate.getForObject(url, String.class, params);
System.out.println(result);
```
- `RestTemplate`: 동기식 API (Spring 5 이후는 `WebClient` 권장)
- `WebClient`: 비동기·리액티브 지원

---

## 4. REST API 호출 시 주의 사항

- **타임아웃 설정**: 무한 대기 방지
- **에러 처리**: 상태 코드별 로직 구현
- **인증 처리**: OAuth 2.0, Basic Auth, API Key 등
- **JSON 직렬화/역직렬화**: Jackson, Gson 사용
- **로깅**: 요청·응답 로그로 디버깅 가능하게

---

## 5. JSON 처리 예제 (Jackson)
```java
ObjectMapper mapper = new ObjectMapper();
String json = "{\"id\":1,\"title\":\"Hello\"}";

Post post = mapper.readValue(json, Post.class);
System.out.println(post.getTitle());
```
```java
public class Post {
    private int id;
    private String title;
    // getter, setter
}
```

---

## 6. 요약
- REST API 클라이언트는 **HTTP 요청/응답** 처리의 핵심 요소
- 표준 `HttpClient`는 최신 JDK에서 권장
- 고급 기능은 **Apache HttpClient**, **OkHttp**, **Spring WebClient** 고려
- JSON 변환, 인증, 타임아웃 등 **실무 환경에 맞게** 설계 필요