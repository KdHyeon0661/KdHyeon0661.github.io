---
layout: post
title: 데이터 통신 - Standard Client-Server Protocols (1)
date: 2024-09-05 22:20:23 +0900
category: DataCommunication
---
# Chapter 26.1 World Wide Web and HTTP — 구조, 동작, 예제까지

이 장에서는 **표준 클라이언트–서버 프로토콜** 중 가장 핵심인  
**World Wide Web(WWW)** 과 **HTTP(Hypertext Transfer Protocol)** 을 다룬다.

앞에서 이미
- **전송 계층(TCP/UDP)**  
- **소켓 프로그래밍, 클라이언트–서버 모델**  

을 공부했으므로, 여기서는 그 위에 올라가는 **애플리케이션 계층 프로토콜** 관점에서  
**웹이 실제로 어떻게 동작하는지**를 “실제 트래픽” 수준까지 내려가서 보게 된다.

---

## 26.1.1 World Wide Web (WWW)

### 1) WWW의 개념 — 인터넷 vs 웹

먼저 용어부터 정리하자.

- **Internet(인터넷)**  
  - 전 세계의 라우터·링크·호스트로 구성된 **패킷 교환 네트워크** 그 자체  
  - IP, 라우팅, BGP, AS 등 네트워크 인프라 수준의 개념
- **World Wide Web(WWW, 웹)**   
  - 인터넷 위에서 동작하는 **하이퍼텍스트/hypermedia 시스템**  
  - “브라우저 + 웹 서버 + HTTP + URI + HTML 등”이 만드는 논리적 서비스 계층

즉:

> 인터넷은 “도로망”, 웹은 그 위를 달리는 “웹 서비스/콘텐츠”라고 보면 된다.

#### WWW의 핵심 구성 요소

1. **웹 클라이언트(브라우저)**  
   - Chrome, Firefox, Edge, Safari 등  
   - 사용자가 URL 입력 → HTTP 요청 생성 → 서버에서 온 응답을 해석/렌더링

2. **웹 서버**  
   - Apache, Nginx, IIS, Node.js 기반 서버 등  
   - HTTP 요청을 받아 HTML, CSS, JS, 이미지, JSON, 동영상 등 **리소스(resource)** 반환

3. **리소스(Resource) & URI/URL**  
   - 리소스:  
     - 정적: `.html`, `.css`, `.js`, `.png`, `.jpg`, `.mp4`, `.pdf` …  
     - 동적: `/api/user/42`, `/search?q=http` 등 서버 코드가 실행되어 생성되는 응답
   - URI(Uniform Resource Identifier), URL(Uniform Resource Locator):  
     - 웹에서 리소스를 **식별**하고 **위치**를 지정하기 위한 표준 문자열

4. **전송 프로토콜: HTTP/HTTPS**  
   - 애플리케이션 계층에서 브라우저와 서버가 통신하는 프로토콜  
   - 오늘날 HTTP/1.1, HTTP/2, HTTP/3가 공존

5. **DNS**  
   - `www.example.com` → IP 주소 변환

6. **캐시/프록시/CDN**  
   - 웹 캐시, 프록시 서버, CDN이 HTTP 요청/응답을 **중간에서 최적화**한다.   

---

### 2) WWW 아키텍처 — 클라이언트–서버 + 리소스 기반

#### (1) 기본 구조

텍스트로 그린 간단한 구조:

```text
[Web Browser] --HTTP--> [Web Server] --(파일/DB/애플리케이션)-->
             <--HTTP--            <--
```

조금 더 실제에 가깝게 확장하면:

```text
+------------+      +-----------+      +-----------------+      +----------+
|   Browser  | ---> |  Proxy /  | ---> |  Origin Server  | ---> | Database |
| (Client)   | <--- |   CDN     | <--- | (App + Static)  | <--- |  / Cache |
+------------+      +-----------+      +-----------------+      +----------+
         ^                 ^                     ^
         |                 |                     |
         +------DNS / TLS handshake / HTTP-------+
```

- 사용자는 브라우저에서 URL 입력
- 브라우저는 DNS, TLS(HTTPS일 때), HTTP 요청을 거쳐 서버와 통신
- 중간에 프록시·캐시·CDN이 개입할 수 있다.

#### (2) URL 구조

일반적인 URL 형식:

```text
scheme://user:pass@host:port/path?query#fragment
예) https://user:pw@www.example.com:8443/articles/42?print=1#comments
```

- `scheme`: `http`, `https`, `ws`, `wss`, `ftp` …
- `host`: 도메인 또는 IP (`www.example.com`, `203.0.113.10`)
- `port`: 생략 시 기본값 (HTTP 80, HTTPS 443)
- `path`: 서버 안에서의 리소스 경로 (`/index.html`, `/api/v1/users`)
- `query`: `?key=value&key2=value2` 형태의 추가 파라미터
- `fragment`: 문서 내부 위치 (`#section3`)

HTTP/1.1 스펙은 `http`/`https` URI 스킴과 URL의 세부 구문을 정의한다.   

---

### 3) WWW 동작 과정 예제 — “브라우저 주소창에 URL을 입력했을 때”

**시나리오:** 사용자가 브라우저 주소창에 `https://www.example.com/` 을 입력하고 Enter를 눌렀다.

1. **URL 파싱**
   - 브라우저는 `scheme=https`, `host=www.example.com`, `port=443`, `path=/` 로 나눈다.

2. **DNS 조회**
   - OS의 DNS 리졸버를 통해 `www.example.com` 의 IP를 얻는다 (예: `93.184.216.34`).

3. **TCP/QUIC 연결 수립**
   - HTTP/1.1 또는 HTTP/2라면:  
     - 브라우저는 IP `93.184.216.34`, 포트 443으로 **TCP 3-way handshake** 수행 후  
       TLS 핸드셰이크(HTTPS) 진행.
   - HTTP/3라면:
     - TCP 대신 **QUIC(UDP 기반)** 연결을 수립하고 그 위에 HTTP/3를 올린다.   

4. **HTTP 요청 전송 (예: GET)**

   ```http
   GET / HTTP/1.1
   Host: www.example.com
   User-Agent: Mozilla/5.0 ...
   Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
   Accept-Language: en-US,en;q=0.5
   Accept-Encoding: gzip, deflate, br
   Connection: keep-alive

   ```

5. **서버 처리**
   - 웹 서버(또는 애플리케이션 서버)가 요청을 분석:
     - 가상 호스트 선택(Host 헤더)
     - URL 라우팅(`/` → 컨트롤러 또는 정적 파일)
     - 필요한 경우 DB, 캐시, 다른 서비스 호출

6. **HTTP 응답 전송**

   ```http
   HTTP/1.1 200 OK
   Date: Tue, 18 Nov 2025 10:00:00 GMT
   Server: nginx/1.24.0
   Content-Type: text/html; charset=utf-8
   Content-Length: 1024
   Connection: keep-alive

   <!doctype html>
   <html>
   <head> ... </head>
   <body> ... </body>
   </html>
   ```

7. **브라우저 렌더링**
   - HTML 파싱 → DOM 트리 생성
   - `<link>`, `<script>`, `<img>` 등 추가 리소스 발견 시,  
     각 리소스에 대해 추가 HTTP 요청 전송 (동일 서버 또는 다른 서버)
   - CSSOM, JS 실행, 레이아웃, 페인팅 단계를 거쳐 화면에 렌더링

8. **추가 최적화**
   - 브라우저 캐시, HTTP 캐시 헤더, 쿠키, LocalStorage 등을 사용해  
     이후 방문 시 성능을 개선한다.   

이 전체 과정이 대부분 **수백 ms ~ 수초** 안에 일어난다.

---

## 26.1.2 HTTP (Hypertext Transfer Protocol)

### 1) HTTP 개요 — 의미와 위치

#### (1) HTTP의 정의

HTTP는 IETF/W3C에 의해 정의된

> “분산·협업·하이퍼텍스트 정보 시스템을 위한 **애플리케이션 계층**의, **stateless**한 프로토콜”

이다.   

- **애플리케이션 계층**: TCP/UDP 위에서 동작한다 (HTTP/1.1, HTTP/2는 TCP / HTTP/3는 QUIC).
- **stateless**:
  - 각 요청은 **독립적**이며 서버는 기본적으로 이전 요청의 상태를 기억하지 않는다.
  - 필요한 상태(로그인 정보 등)는 쿠키, 세션 ID, 토큰 등으로 구현.

#### (2) HTTP의 역할

- 리소스(HTML, JSON, 이미지, 동영상 등) **전송** (요청/응답)
- 메시지 **메타데이터** 전달:
  - 콘텐츠 타입(`Content-Type`), 길이(`Content-Length`), 인코딩(`Content-Encoding`)
  - 캐시 정책, 인증, 쿠키, 언어/지역 정보 등
- 웹 애플리케이션의 **REST API** 기반 통신
- 오늘날 웹 브라우징, 모바일 앱 백엔드, RESTful API, 마이크로서비스 대부분이 HTTP 기반

---

### 2) HTTP의 발전 — HTTP/0.9 → 1.0 → 1.1 → 2 → 3

HTTP는 1990년대 초 이후 크게 발전해 왔다.   

| 버전 | 시대 | 특징 요약 |
|------|------|----------|
| HTTP/0.9 | 1991 경 | 매우 단순, 오직 `GET`만, 헤더 없음, HTML만 전송 |
| HTTP/1.0 | 1996 | 헤더 도입, 여러 메서드(GET/HEAD/POST), 상태 코드, MIME 타입 |
| HTTP/1.1 | 1997, RFC 2616 → 7230~7235 | **Persistent Connection(keep-alive)**, Host 헤더 필수, 캐싱/Range/프락시 등 정교화 |
| HTTP/2 | 2015 | **바이너리 프레이밍**, 스트림 멀티플렉싱, 헤더 압축(HPACK), 서버 푸시 |
| HTTP/3 | 2022 | 전송 계층을 **QUIC(UDP 기반)** 으로 변경, HOL blocking 완화, 1-RTT 핸드셰이크 등 |

- 2024년 기준, 주요 브라우저의 HTTP/3 지원율은 95% 이상이며,  
  상위 1천만 웹사이트 중 약 30% 이상이 HTTP/3를 사용 중이라는 분석도 있다.   

HTTP/3는 **HTTP의 “의미적 계층”(method, status code, header의 의미)** 는 유지하면서  
“전송 계층 매핑”만 QUIC으로 바꾼 것이라고 이해하면 된다.   

---

### 3) HTTP 메시지 구조 — Request / Response

HTTP는 **텍스트 기반**(HTTP/1.x 기준) 요청/응답 메시지로 동작한다.   

#### (1) 요청(Request) 메시지 형식

```http
<메서드> <요청-대상> <버전>
<헤더-이름>: <값>
<헤더-이름>: <값>
...

<빈 줄>
<메시지 바디(선택)>
```

예: 간단한 GET 요청

```http
GET /index.html HTTP/1.1
Host: www.example.com
User-Agent: curl/8.5.0
Accept: text/html

```

#### (2) 응답(Response) 메시지 형식

```http
<버전> <상태 코드> <사유 구문>
<헤더-이름>: <값>
...

<빈 줄>
<메시지 바디>
```

예: 200 OK 응답

```http
HTTP/1.1 200 OK
Date: Tue, 18 Nov 2025 10:00:00 GMT
Server: Apache/2.4.62 (Unix)
Content-Type: text/html; charset=UTF-8
Content-Length: 512

<!doctype html>
<html>...</html>
```

---

### 4) HTTP 메서드와 상태 코드

#### (1) 주요 메서드

| 메서드 | 용도 |
|--------|------|
| `GET` | 리소스 조회(부작용 없어야 함) |
| `HEAD` | 바디 없이 헤더만 요청 (메타데이터 확인) |
| `POST` | 서버에 데이터 전송 (폼 제출, 리소스 생성 등) |
| `PUT` | 리소스 전체 교체/저장 (idempotent) |
| `PATCH` | 리소스 부분 수정 |
| `DELETE` | 리소스 삭제 |
| `OPTIONS` | 서버/리소스가 지원하는 메서드/옵션 질의 |

예: JSON API에 대한 POST 요청

```http
POST /api/v1/users HTTP/1.1
Host: api.example.com
Content-Type: application/json
Content-Length: 47

{"name": "Kim", "email": "kim@example.org"}
```

응답 예:

```http
HTTP/1.1 201 Created
Location: /api/v1/users/42
Content-Type: application/json

{"id": 42, "name": "Kim", "email": "kim@example.org"}
```

#### (2) 상태 코드

상태 코드는 **세 자리 숫자**이며, 첫 자리로 클래스를 나타낸다.   

- 1xx: 정보 (예: 100 Continue)
- 2xx: 성공 (200 OK, 201 Created, 204 No Content)
- 3xx: 리다이렉션 (301 Moved Permanently, 302 Found, 307/308 등)
- 4xx: 클라이언트 오류 (400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found)
- 5xx: 서버 오류 (500 Internal Server Error, 502 Bad Gateway, 503 Service Unavailable)

예: 존재하지 않는 페이지 요청

```http
GET /no-such-page HTTP/1.1
Host: www.example.com

HTTP/1.1 404 Not Found
Content-Type: text/html; charset=UTF-8

<html><body><h1>404 Not Found</h1></body></html>
```

---

### 5) HTTP 헤더 — 메타데이터와 제어 정보

HTTP 헤더는 요청·응답 모두에서 사용되며, **키:값** 쌍이다.   

#### (1) 대표적인 헤더 예시

| 범주 | 헤더 | 예 | 의미 |
|------|------|----|------|
| 일반 | `Date` | `Date: Tue, 18 Nov 2025 10:00:00 GMT` | 메시지 생성 시간 |
| 요청 | `Host` | `Host: www.example.com` | 가상호스트 구분 |
| 요청 | `User-Agent` | `User-Agent: Mozilla/5.0 ...` | 클라이언트 정보 |
| 요청 | `Accept` | `Accept: text/html, application/json` | 선호 콘텐츠 타입 |
| 요청 | `Accept-Language` | `Accept-Language: en-US,en;q=0.5` | 선호 언어 |
| 응답 | `Server` | `Server: nginx/1.24.0` | 서버 소프트웨어 |
| 응답 | `Content-Type` | `Content-Type: text/html; charset=UTF-8` | 바디 타입 |
| 응답 | `Content-Length` | `Content-Length: 1024` | 바디 길이 (바이트) |
| 응답 | `Location` | `Location: /login` | 리다이렉션 위치 (3xx) |
| 둘 다 | `Cache-Control` | `Cache-Control: max-age=3600` | 캐시 정책 |
| 둘 다 | `Cookie` / `Set-Cookie` | 세션/상태 관리용 쿠키 | 상태 유지 |

---

### 6) HTTP의 상태 없음(stateless)과 상태 유지 — 쿠키, 세션

HTTP 자체는 **stateless** 하므로, 기본적으로 서버는 요청 간의 상태를 유지하지 않는다.   

하지만 웹 애플리케이션은 로그인 상태, 장바구니, 사용자 설정 등 **상태**를 필요로 하기 때문에  
보통 다음을 조합한다:   

1. **쿠키(Cookie)**  
   - 서버가 `Set-Cookie` 헤더로 브라우저에 작은 텍스트 블록을 저장시키고,  
     이후 해당 사이트에 대한 요청마다 브라우저가 `Cookie` 헤더로 다시 보낸다.
   - 예:

     ```http
     HTTP/1.1 200 OK
     Set-Cookie: session_id=abc123; Path=/; HttpOnly; Secure
     ```

     이후 요청:

     ```http
     GET /dashboard HTTP/1.1
     Host: www.example.com
     Cookie: session_id=abc123
     ```

2. **서버 세션 저장소**  
   - `session_id` → 실제 상태(로그인 사용자 id, 장바구니 내용 등)를 서버 메모리/Redis/DB에 저장
   - HTTP 요청은 여전히 stateless 이지만, **쿠키+세션** 조합으로 상태처럼 보이게 만든다.

3. **토큰 기반 인증(JWT 등)**  
   - 쿠키 대신 Authorization 헤더에 토큰을 넣는 방식도 널리 사용된다.

---

### 7) 연결 관리 — HTTP/1.1 vs HTTP/2 vs HTTP/3

HTTP는 **애플리케이션 계층**이고, 실제 바이트 전송은 전송 계층(TCP/QUIC)이 담당한다.  
버전에 따라 연결 관리 방식이 크게 달라졌다.   

#### (1) HTTP/1.0 — 비지속 연결(Non-persistent)

- 기본적으로 **요청 1개당 TCP 연결 1개**:
  - 리소스 10개를 받으려면 최소 10번의 TCP 핸드셰이크(3-way handshake)가 필요.
- 성능 문제가 심각해짐.

#### (2) HTTP/1.1 — Persistent Connection(keep-alive)

- 기본은 연결 **재사용**:
  - `Connection: keep-alive` (명시적 혹은 기본 정책)
  - 여러 요청/응답을 **하나의 TCP 연결**에서 순차적으로 처리 가능.
- 장점:
  - TCP 핸드셰이크 비용 감소 → 지연(latency) 감소.
  - TCP 혼잡 제어 슬로스타트가 진전된 상태에서 여러 요청 처리 가능.
- 단점:
  - 여러 요청을 직렬로 보내면, 앞 요청이 느리면 뒤 요청이 막히는 **HOL(Head-of-line) blocking** 발생.

#### (3) HTTP/2 — 멀티플렉싱과 헤더 압축

- 하나의 TCP 연결 위에 여러 **스트림(stream)** 을 논리적으로 나누어,  
  각 요청/응답을 독립적인 스트림으로 처리 (바이너리 프레이밍).
- 장점:
  - 멀티플렉싱으로 HOL-blocking 완화 (전송 계층 수준의 HOL은 여전히 존재).
  - 헤더 압축(HPACK)으로 중복 헤더 오버헤드 감소.

#### (4) HTTP/3 — QUIC 위의 HTTP

- 전송 계층을 TCP → QUIC으로 변경.
- QUIC은 **UDP 기반**의 새로운 전송 프로토콜로,  
  유저 공간 구현, 연결 ID, 스트림 단위의 독립적인 손실 복구 등을 제공한다.   
- 효과:
  - 손실된 패킷이 하나의 스트림에만 영향을 미치므로,  
    HTTP/2에서 TCP HOL-blocking 때문에 발생하던 지연을 크게 줄일 수 있음.
  - 0-RTT/1-RTT 핸드셰이크로 연결 수립 지연 감소.

#### (5) 단순 지연 시간 모델 예시

아주 단순화해서, 특정 리소스를 받기까지의 시간을

$$
T_{\text{total}} \approx N_{\text{handshake}} \cdot RTT + T_{\text{server}} + T_{\text{transfer}}
$$

라고 두면,

- HTTP/1.0: `N_handshake`가 리소스 수에 비례
- HTTP/1.1: 대부분 1 (혹은 소수의 연결), `T_transfer`가 개선
- HTTP/2: `T_transfer`가 멀티플렉싱 덕분에 여러 리소스를 병렬로 운반
- HTTP/3: `N_handshake`와 `RTT` 관점에서 추가 이득 (QUIC 핸드셰이크, 재전송 최적화)

---

### 8) HTTP 캐싱 — 브라우저와 중간 캐시의 동작

HTTP는 캐싱 메커니즘을 매우 정교하게 정의한다.   

#### (1) 캐시 계층

1. 브라우저 캐시
2. 프록시 캐시(사내 프록시, ISP 캐시)
3. CDN 캐시(Cloudflare, Akamai 등)

#### (2) 주요 헤더

- `Cache-Control`: `max-age=3600`, `no-cache`, `no-store`, `public`, `private` 등
- `ETag`: 리소스 버전 식별자
- `Last-Modified`: 최종 수정 시각
- `If-None-Match`, `If-Modified-Since`: 조건부 요청(Conditional Request)에 사용

#### (3) 예제: 이미지 캐싱

1. 첫 요청

   ```http
   GET /static/logo.png HTTP/1.1
   Host: www.example.com

   HTTP/1.1 200 OK
   Cache-Control: max-age=3600
   ETag: "v1-logo"
   Content-Type: image/png
   Content-Length: 12345

   <이미지 바이트>
   ```

2. 30분 후 재방문

   ```http
   GET /static/logo.png HTTP/1.1
   Host: www.example.com
   If-None-Match: "v1-logo"

   HTTP/1.1 304 Not Modified
   ```

- 서버는 **304 Not Modified**로 응답, 바디를 다시 보내지 않는다.
- 브라우저는 캐시에 있던 이미지를 그대로 사용 → 네트워크 트래픽과 지연 감소.

---

### 9) HTTP 예제 — 간단한 서버/클라이언트 코드

여기서는 HTTP **프로토콜**에 집중하므로,  
코드는 “HTTP 메시지가 실제로 어떤 식으로 오가는지”를 보이기 위한 최소 예제만 다룬다.

#### (1) Python으로 만드는 초간단 HTTP 서버 (학습용)

```python
# simple_http_server.py
import socket

HOST = "0.0.0.0"
PORT = 8080

with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
    s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    s.bind((HOST, PORT))
    s.listen(5)
    print(f"Listening on {HOST}:{PORT}")

    while True:
        conn, addr = s.accept()
        with conn:
            print("Connected by", addr)
            data = conn.recv(4096)
            if not data:
                continue
            print("--- Request ---")
            print(data.decode("utf-8", errors="ignore"))
            body = "<html><body><h1>Hello HTTP</h1></body></html>"
            response = (
                "HTTP/1.1 200 OK\r\n"
                f"Content-Length: {len(body)}\r\n"
                "Content-Type: text/html; charset=utf-8\r\n"
                "Connection: close\r\n"
                "\r\n" +
                body
            )
            conn.sendall(response.encode("utf-8"))
```

실행 후 브라우저에서 `http://localhost:8080/` 으로 접속하면  
위에서 설명한 구조 그대로 HTTP 요청/응답이 오가는 것을 볼 수 있다.

#### (2) `curl`로 직접 HTTP 메시지 보기

터미널에서:

```bash
curl -v http://example.com/
```

- `-v` 옵션은 요청/응답 헤더를 포함한 디버그 정보를 출력해 준다.

출력 일부 예:

```text
> GET / HTTP/1.1
> Host: example.com
> User-Agent: curl/8.5.0
> Accept: */*
>
< HTTP/1.1 200 OK
< Content-Type: text/html; charset=UTF-8
< Content-Length: 1256
< Cache-Control: max-age=604800
< ...
```

이렇게 **실제 브라우저가 보내는 것과 거의 동일한** HTTP 메시지를 관찰할 수 있다.

---

## 마무리 정리

이 장에서 다룬 핵심 포인트를 다시 모아 보면:

1. **World Wide Web(WWW)** 는 인터넷 위에서 동작하는 **하이퍼텍스트/hypermedia 시스템**이며,
   - 브라우저(클라이언트), 웹 서버, 리소스, URI/URL, DNS, HTTP, 캐시/프록시 등으로 구성된다.

2. **HTTP** 는 WWW를 떠받치는 애플리케이션 계층 프로토콜로,
   - **요청/응답 메시지 구조**, 메서드, 상태 코드, 헤더, 캐싱, 쿠키, 보안(HTTPS)을 정의한다.
   - 설계상 **stateless** 이지만, 쿠키 + 세션/토큰으로 실제 애플리케이션 상태를 유지한다.

3. **HTTP 버전의 진화**
   - HTTP/1.1: Persistent connection, Host 헤더, 캐싱 메커니즘, 프락시 지원
   - HTTP/2: 스트림 멀티플렉싱, 헤더 압축
   - HTTP/3: QUIC 위에서 동작, HOL-blocking 완화, 낮은 지연

4. **실습 예제**를 통해:
   - C/Java/TCP/UDP 기반 클라이언트–서버 코드 위에  
     HTTP 메시지를 올려서 실제로 동작시키는 과정을 체험할 수 있다.