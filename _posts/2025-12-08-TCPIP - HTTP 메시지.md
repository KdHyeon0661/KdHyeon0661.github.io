---
layout: post
title: TCPIP - HTTP 메시지
date: 2025-12-08 17:25:23 +0900
category: TCPIP
---
# HTTP 메시지, 메시지 형식, 메서드 및 상태 코드

## HTTP 일반 메시지 형식

HTTP(Hypertext Transfer Protocol)는 웹의 근간을 이루는 애플리케이션 계층 프로토콜로, 클라이언트와 서버 간의 통신을 위한 구조화된 메시지 형식을 정의합니다. 모든 HTTP 메시지는 텍스트 기반으로, ASCII 문자 집합을 사용하여 사람이 읽을 수 있는 형태로 구성됩니다.

### 기본 메시지 구조

HTTP 메시지는 시작 줄(start-line), 헤더(header), 빈 줄(blank line), 메시지 본문(message body)으로 구성됩니다. 이 구조는 요청과 응답 모두에 공통적으로 적용됩니다.

```
HTTP 메시지 일반 형식:
┌─────────────────────────────────────────┐
│ 시작 줄 (Start-line)                     │
├─────────────────────────────────────────┤
│ 헤더 (Headers)                           │
│ • 일반 헤더                              │
│ • 요청/응답 헤더                         │
│ • 엔터티 헤더                            │
├─────────────────────────────────────────┤
│ 빈 줄 (CRLF만 있는 줄)                   │
├─────────────────────────────────────────┤
│ 본문 (Body) - 선택적                    │
└─────────────────────────────────────────┘
```

**문자 인코딩과 줄 종결자:**
- **인코딩**: ASCII (HTTP/1.1), UTF-8 (일부 헤더 값)
- **줄 종결자**: CRLF (Carriage Return + Line Feed, `\r\n`)
- **텍스트 기반**: 사람과 기계 모두 읽을 수 있음

### 메시지 구문 분석 규칙

HTTP 메시지 파싱은 엄격한 규칙을 따릅니다:

1. **시작 줄 구분**: 첫 번째 공백이나 탭까지가 메서드/상태 코드
2. **헤더 필드**: `필드명: 값` 형식 (대소문자 구분 없음)
3. **헤더 종료**: 빈 줄(CRLF만 있는 줄)로 헤더 영역 끝 표시
4. **본문 존재**: Content-Length 또는 Transfer-Encoding 헤더로 판단
5. **청크 전송**: Transfer-Encoding: chunked일 경우 특수 형식

## HTTP 요청 메시지 형식

HTTP 요청은 클라이언트가 서버에 자원을 요청할 때 사용하는 메시지 형식입니다. 웹 브라우저가 웹 서버에 페이지를 요청하는 것이 가장 일반적인 예시입니다.

### 요청 메시지 구조

```
HTTP 요청 형식:
┌─────────────────────────────────────────┐
│ 요청 줄 (Request Line)                   │
│ • 메서드 SP URI SP HTTP-버전 CRLF       │
├─────────────────────────────────────────┤
│ 요청 헤더 (Request Headers)              │
│ • 일반 헤더                              │
│ • 요청 헤더                              │
│ • 엔터티 헤더                            │
├─────────────────────────────────────────┤
│ 빈 줄 (CRLF)                             │
├─────────────────────────────────────────┤
│ 메시지 본문 (Message Body) - 선택적     │
└─────────────────────────────────────────┘
```

### 요청 줄 상세 분석

요청 줄은 세 부분으로 구성되며, 각 부분은 SP(공백, ASCII 32)로 구분됩니다:

**형식:** `메서드 SP 요청-URI SP HTTP-버전 CRLF`

**예시:** `GET /index.html HTTP/1.1\r\n`

#### 1. 메서드 (HTTP Method)
- 자원에 수행할 작업 지정
- 대소문자 구분 (일반적으로 대문자 사용)
- 예: GET, POST, PUT, DELETE 등

#### 2. 요청 URI (Request-URI)
- 요청 대상 자원 식별
- **절대 경로**: `/images/logo.png`
- **절대 URI**: `http://example.com/path` (프록시에서 사용)
- **권한 형식**: `example.com:80` (CONNECT 메서드용)
- **별표(\*)**: 서버 전체 (OPTIONS 메서드용)

#### 3. HTTP 버전
- 사용 중인 HTTP 프로토콜 버전
- 형식: `HTTP/주버전.부버전`
- 예: `HTTP/1.1`, `HTTP/2`, `HTTP/3`

### 요청 헤더 카테고리

#### 일반 헤더 (General Headers)
```
요청과 응답 모두에 사용:
Date: 요청 생성 시간
Connection: 연결 관리
Cache-Control: 캐시 지시자
Upgrade: 프로토콜 업그레이드
Via: 중개 서버 정보
```

#### 요청 헤더 (Request Headers)
```
클라이언트의 추가 정보:
Host: 요청 대상 호스트 (HTTP/1.1 필수)
User-Agent: 클라이언트 소프트웨어
Accept: 수용 가능한 미디어 타입
Accept-Language: 선호 언어
Accept-Encoding: 수용 가능한 압축 방식
Authorization: 인증 정보
Cookie: 서버가 설정한 쿠키
Referer: 이전 페이지 URL
```

#### 엔터티 헤더 (Entity Headers)
```
본문에 대한 메타데이터:
Content-Type: 본문 미디어 타입
Content-Length: 본문 크기(바이트)
Content-Encoding: 본문 압축 방식
Content-Language: 본문 언어
```

### 실제 요청 메시지 예시

**간단한 GET 요청:**
```
GET /index.html HTTP/1.1
Host: www.example.com
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64)
Accept: text/html,application/xhtml+xml
Accept-Language: ko-KR,ko;q=0.9,en;q=0.8
Accept-Encoding: gzip, deflate
Connection: keep-alive
```

**POST 요청 (폼 데이터 전송):**
```
POST /submit-form HTTP/1.1
Host: www.example.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 27
User-Agent: Mozilla/5.0

username=johndoe&password=secret123
```

**멀티파트 폼 데이터:**
```
POST /upload HTTP/1.1
Host: www.example.com
Content-Type: multipart/form-data; boundary=----WebKitFormBoundary7MA4YWxkTrZu0gW
Content-Length: 348

------WebKitFormBoundary7MA4YWxkTrZu0gW
Content-Disposition: form-data; name="text"

제목입니다
------WebKitFormBoundary7MA4YWxkTrZu0gW
Content-Disposition: form-data; name="file"; filename="example.txt"
Content-Type: text/plain

파일 내용입니다.
------WebKitFormBoundary7MA4YWxkTrZu0gW--
```

### HTTP/2와 HTTP/3의 변화

**HTTP/2:**
- 텍스트 기반에서 바이너리 프레임으로 변경
- 헤더 압축(HPACK) 도입
- 멀티플렉싱 지원
- 하지만 논리적 메시지 구조는 유사

**HTTP/3:**
- QUIC 프로토콜 기반
- TCP 대신 UDP 사용
- 내장 보안(TLS 1.3)
- 연결 설정 지연 감소

## HTTP 응답 메시지 형식

HTTP 응답은 서버가 클라이언트의 요청에 대해 반환하는 메시지입니다. 요청 처리 결과와 요청된 자원(있는 경우)을 포함합니다.

### 응답 메시지 구조

```
HTTP 응답 형식:
┌─────────────────────────────────────────┐
│ 상태 줄 (Status Line)                    │
│ • HTTP-버전 SP 상태코드 SP 이유구문 CRLF│
├─────────────────────────────────────────┤
│ 응답 헤더 (Response Headers)             │
│ • 일반 헤더                              │
│ • 응답 헤더                              │
│ • 엔터티 헤더                            │
├─────────────────────────────────────────┤
│ 빈 줄 (CRLF)                             │
├─────────────────────────────────────────┤
│ 메시지 본문 (Message Body) - 선택적     │
└─────────────────────────────────────────┘
```

### 상태 줄 상세 분석

상태 줄은 서버의 응답 상태를 요약합니다:

**형식:** `HTTP-버전 SP 상태코드 SP 이유구문 CRLF`

**예시:** `HTTP/1.1 200 OK\r\n`

#### 1. HTTP 버전
- 서버가 사용하는 HTTP 프로토콜 버전
- 클라이언트의 버전과 같거나 낮을 수 있음
- 예: `HTTP/1.1`, `HTTP/2`, `HTTP/3`

#### 2. 상태 코드 (Status Code)
- 요청 처리 결과를 나타내는 3자리 숫자
- 첫 자리로 응답 카테고리 구분
- 예: `200`, `404`, `500`

#### 3. 이유 구문 (Reason Phrase)
- 상태 코드의 텍스트 설명
- 사람이 읽을 수 있는 형태
- 표준 구문이 있지만 서버마다 다를 수 있음
- 예: `OK`, `Not Found`, `Internal Server Error`

### 응답 헤더 카테고리

#### 일반 헤더 (General Headers)
```
Date: 응답 생성 시간
Connection: 연결 관리 옵션
Cache-Control: 캐싱 정책
Upgrade: 프로토콜 업그레이드 제안
Via: 중개 서버 경로
```

#### 응답 헤더 (Response Headers)
```
서버 정보와 추가 지시:
Server: 서버 소프트웨어 정보
Location: 리다이렉트 대상 (3xx 응답)
Retry-After: 재시도 권장 시간
ETag: 엔터티 태그 (캐시 검증)
WWW-Authenticate: 인증 요구 (401 응답)
Set-Cookie: 클라이언트에 쿠키 설정
```

#### 엔터티 헤더 (Entity Headers)
```
본문에 대한 메타데이터:
Content-Type: 본문 미디어 타입 (MIME)
Content-Length: 본문 크기(바이트)
Content-Encoding: 압축 방식
Content-Language: 언어
Last-Modified: 최종 수정 시간
Expires: 만료 시간
```

### 실제 응답 메시지 예시

**성공적인 HTML 페이지 응답:**
```
HTTP/1.1 200 OK
Date: Mon, 15 Jan 2024 10:30:00 GMT
Server: Apache/2.4.41
Content-Type: text/html; charset=utf-8
Content-Length: 1234
Connection: keep-alive
Cache-Control: max-age=3600
ETag: "abc123"

<!DOCTYPE html>
<html>
<head><title>Welcome</title></head>
<body><h1>Hello World</h1></body>
</html>
```

**리다이렉트 응답:**
```
HTTP/1.1 301 Moved Permanently
Date: Mon, 15 Jan 2024 10:31:00 GMT
Server: nginx/1.18.0
Location: https://www.example.com/
Content-Type: text/html
Content-Length: 178
Connection: close

<html>
<head><title>301 Moved Permanently</title></head>
<body><h1>Moved Permanently</h1></body>
</html>
```

**클라이언트 오류 응답:**
```
HTTP/1.1 404 Not Found
Date: Mon, 15 Jan 2024 10:32:00 GMT
Server: Apache/2.4.41
Content-Type: text/html; charset=utf-8
Content-Length: 456
Connection: keep-alive

<html>
<head><title>404 Not Found</title></head>
<body><h1>The requested URL was not found</h1></body>
</html>
```

**서버 오류 응답:**
```
HTTP/1.1 500 Internal Server Error
Date: Mon, 15 Jan 2024 10:33:00 GMT
Server: Apache/2.4.41
Content-Type: text/html; charset=utf-8
Content-Length: 567
Connection: close
Retry-After: 60

<html>
<head><title>500 Internal Server Error</title></head>
<body><h1>Something went wrong</h1></body>
</html>
```

### 청크 전송 인코딩

대용량 응답이나 스트리밍 데이터에 사용되는 전송 방식:

```
HTTP/1.1 200 OK
Content-Type: text/plain
Transfer-Encoding: chunked

7\r\n
Mozilla\r\n
9\r\n
Developer\r\n
7\r\n
Network\r\n
0\r\n
\r\n
```

**청크 형식 해석:**
- `크기(16진수)\r\n` + `데이터\r\n` 반복
- 마지막 청크: `0\r\n\r\n`
- 트레일러 헤더: 마지막 청크 후 추가 헤더 가능

## HTTP 메서드

HTTP 메서드는 자원에 대해 수행할 작업의 종류를 정의합니다. 각 메서드는 특정 의미론적(semantic)을 가지며, 안전성(safety)과 멱등성(idempotence) 특성을 가집니다.

### 주요 HTTP 메서드

#### 1. GET
**의미:** 자원의 표현 조회
**안전성:** 안전함 (서버 상태 변경 없음)
**멱등성:** 멱등함 (여러 번 호출해도 같은 결과)
**사용처:**
- 웹 페이지 로드
- 이미지, 스타일시트, 스크립트 다운로드
- API 데이터 조회
**예시:**
```
GET /users/123 HTTP/1.1
Host: api.example.com
```

#### 2. HEAD
**의미:** GET과 동일하지만 본문 없이 헤더만 반환
**안전성:** 안전함
**멱등성:** 멱등함
**사용처:**
- 자원 존재 여부 확인
- 자원 메타데이터 확인
- 조건부 GET 전 확인
**예시:**
```
HEAD /document.pdf HTTP/1.1
Host: www.example.com
```

#### 3. POST
**의미:** 새 자원 생성 또는 데이터 처리
**안전성:** 안전하지 않음 (상태 변경 가능)
**멱등성:** 멱등하지 않음 (여러 번 호출 시 다른 결과)
**사용처:**
- 폼 제출
- 파일 업로드
- 새 게시물 작성
- 트랜잭션 시작
**예시:**
```
POST /users HTTP/1.1
Host: api.example.com
Content-Type: application/json

{"name": "John", "email": "john@example.com"}
```

#### 4. PUT
**의미:** 자원 전체 대체 또는 생성
**안전성:** 안전하지 않음
**멱등성:** 멱등함 (여러 번 호출해도 같은 결과)
**사용처:**
- 자원 전체 업데이트
- 특정 URI에 자원 생성
- 파일 업로드 (덮어쓰기)
**예시:**
```
PUT /users/123 HTTP/1.1
Host: api.example.com
Content-Type: application/json

{"name": "John Updated", "email": "john@example.com"}
```

#### 5. DELETE
**의미:** 자원 삭제
**안전성:** 안전하지 않음
**멱등성:** 멱등함
**사용처:**
- 사용자 계정 삭제
- 게시물 삭제
- 파일 삭제
**예시:**
```
DELETE /users/123 HTTP/1.1
Host: api.example.com
```

#### 6. PATCH
**의미:** 자원 부분적 수정
**안전성:** 안전하지 않음
**멱등성:** 멱등하지 않을 수 있음 (구현에 따라)
**사용처:**
- 자원의 특정 필드만 업데이트
- 증분 변경
**예시:**
```
PATCH /users/123 HTTP/1.1
Host: api.example.com
Content-Type: application/json-patch+json

[{"op": "replace", "path": "/email", "value": "new@example.com"}]
```

#### 7. OPTIONS
**의미:** 자원에 대해 지원되는 메서드 조회
**안전성:** 안전함
**멱등성:** 멱등함
**사용처:**
- CORS(Cross-Origin Resource Sharing) 사전 요청
- 서버 기능 탐지
**예시:**
```
OPTIONS /users HTTP/1.1
Host: api.example.com
```

#### 8. TRACE
**의미:** 요청 메시지 루프백 (디버깅용)
**안전성:** 안전함
**멱등성:** 멱등함
**보안 문제:** Cross-Site Tracing 공격 가능성으로 제한적 사용
**예시:**
```
TRACE / HTTP/1.1
Host: www.example.com
```

#### 9. CONNECT
**의미:** 터널 연결 설정 (프록시용)
**안전성:** 안전하지 않음
**멱등성:** 멱등하지 않음
**사용처:**
- SSL 터널링
- 프록시를 통한 HTTPS 연결
**예시:**
```
CONNECT www.example.com:443 HTTP/1.1
Host: www.example.com:443
```

### 메서드 특성 비교

| 메서드 | 안전함 | 멱등함 | 캐시 가능 | 본문 허용 |
|--------|--------|--------|-----------|-----------|
| GET | ✅ | ✅ | ✅ | ❌ |
| HEAD | ✅ | ✅ | ✅ | ❌ |
| POST | ❌ | ❌ | △ | ✅ |
| PUT | ❌ | ✅ | ❌ | ✅ |
| DELETE | ❌ | ✅ | ❌ | △ |
| PATCH | ❌ | △ | ❌ | ✅ |
| OPTIONS | ✅ | ✅ | ❌ | ❌ |
| TRACE | ✅ | ✅ | ❌ | ❌ |
| CONNECT | ❌ | ❌ | ❌ | ✅ |

### RESTful API 설계 원칙

REST 아키텍처 스타일에서 HTTP 메서드는 CRUD 작업에 매핑됩니다:

```
CRUD        HTTP 메서드    URI 패턴          의미
Create      POST          /resources        새 자원 생성
Read        GET           /resources/:id    자원 조회
Update      PUT           /resources/:id    자원 전체 대체
Update      PATCH         /resources/:id    자원 부분 수정
Delete      DELETE        /resources/:id    자원 삭제
List        GET           /resources        자원 목록 조회
```

### 확장 메서드와 사용자 정의 메서드

일부 애플리케이션은 표준 메서드 외에 확장 메서드를 정의할 수 있습니다:

**예시 확장 메서드:**
- **LOCK / UNLOCK**: WebDAV에서 자원 잠금/해제
- **MOVE / COPY**: WebDAV에서 자원 이동/복사
- **MKCOL**: WebDAV에서 컬렉션 생성
- **PROPFIND / PROPPATCH**: WebDAV에서 속성 조회/수정

**사용자 정의 메서드:**
- 앞에 `X-` 접두사 사용 (관례)
- 예: `X-EXPIRE`, `X-MERGE`
- 표준화되지 않아 상호운용성 문제 가능

## HTTP 상태 코드 형식, 상태 코드 및 이유 구문

HTTP 상태 코드는 서버가 클라이언트 요청을 어떻게 처리했는지를 나타내는 3자리 숫자입니다. 각 코드는 특정 카테고리에 속하며, 사람이 읽을 수 있는 이유 구문과 함께 제공됩니다.

### 상태 코드 카테고리

상태 코드의 첫 번째 숫자는 응답 클래스를 정의합니다:

#### 1xx: 정보 응답 (Informational)
요청을 받았으며 프로세스를 계속 진행함

#### 2xx: 성공 응답 (Successful)
요청이 성공적으로 수신, 이해, 수락됨

#### 3xx: 리다이렉션 응답 (Redirection)
요청 완료를 위해 추가 조치가 필요함

#### 4xx: 클라이언트 오류 응답 (Client Error)
클라이언트가 잘못된 요청을 보냄

#### 5xx: 서버 오류 응답 (Server Error)
서버가 유효한 요청을 이행하지 못함

### 주요 상태 코드 상세

#### 1xx - 정보 응답

**100 Continue:**
- 클라이언트가 본문을 계속 전송해도 됨
- Expect: 100-continue 헤더와 함께 사용
- 사용처: 대용량 데이터 전송 전 확인

**101 Switching Protocols:**
- 클라이언트가 Upgrade 헤더 요청 수락
- 프로토콜 전환 (예: HTTP → WebSocket)
- 예: 웹소켓 연결 업그레이드

**102 Processing (WebDAV):**
- 요청 처리 중, 응답 아직 준비되지 않음
- 장시간 처리 작업에서 타임아웃 방지

**103 Early Hints:**
- 최종 응답 전에 일부 리소스 힌트 제공
- 성능 최적화용 (preconnect, preload)

#### 2xx - 성공 응답

**200 OK:**
- 요청 성공, 응답 본문에 요청 결과 포함
- 가장 일반적인 성공 응답
- GET, POST, PUT, DELETE 등에 사용

**201 Created:**
- 새 자원 성공적으로 생성됨
- Location 헤더로 새 자원 URI 제공
- 주로 POST, PUT에 사용

**202 Accepted:**
- 요청 수락됨 but 아직 처리되지 않음
- 비동기 처리, 배치 작업에 사용
- 결과는 별도로 확인 필요

**204 No Content:**
- 요청 성공 but 응답 본문 없음
- 자원 삭제 후, 업데이트 후 피드백 없을 때
- 주로 DELETE, PUT에 사용

**205 Reset Content:**
- 요청 성공, 클라이언트 문서 뷰 리셋 요청
- 폼 제출 후 입력 필드 초기화에 사용

**206 Partial Content:**
- 범위 요청에 대한 부분적 내용 응답
- Content-Range 헤더로 전송 범위 지정
- 대용량 파일 다운로드 재개, 스트리밍에 사용

**207 Multi-Status (WebDAV):**
- 여러 자원에 대한 상태 정보 포함
- XML 본문으로 각 자원별 상태 코드 제공

**208 Already Reported (WebDAV):**
- 동일 컬렉션의 멤버 상태 이미 보고됨

**226 IM Used:**
- 서버가 인스턴스 조작(Instance Manipulation) 수행
- Delta 인코딩에서 변경사항만 반환

#### 3xx - 리다이렉션 응답

**300 Multiple Choices:**
- 요청에 대해 여러 응답 가능
- 클라이언트가 적절한 선택 필요
- 드물게 사용됨

**301 Moved Permanently:**
- 요청된 자원이 새 URI로 영구 이동
- 북마크, 링크 업데이트 필요
- GET 메서드는 자동으로 리다이렉트

**302 Found:**
- 자원이 일시적으로 다른 URI에 있음
- 클라이언트는 이후 요청도 원본 URI 사용
- 역사적 이유로 혼란스러움 (302 vs 303)

**303 See Other:**
- 요청에 대한 응답이 다른 URI에 있음
- POST 후 결과 페이지 표시에 사용
- GET으로 리다이렉트해야 함

**304 Not Modified:**
- 조건부 요청 시 자원 수정되지 않음
- 캐시된 버전 사용 지시
- 본문 없이 헤더만 반환

**305 Use Proxy:**
- 요청한 자원은 지정된 프록시를 통해야 함
- 보안 문제로 현대 브라우저에서 지원 안 함

**307 Temporary Redirect:**
- 자원이 일시적으로 다른 URI에 있음
- 원본 요청 메서드 유지 (302와 차이)

**308 Permanent Redirect:**
- 자원이 영구적으로 다른 URI로 이동
- 요청 메서드와 본문 유지 (301과 차이)

#### 4xx - 클라이언트 오류

**400 Bad Request:**
- 잘못된 구문으로 서버가 요청 이해 못함
- 누락된 필드, 잘못된 형식 등

**401 Unauthorized:**
- 인증 필요, 인증 실패
- WWW-Authenticate 헤더로 인증 방식 제시

**402 Payment Required:**
- 예약됨, 미래 사용을 위해
- 디지털 결제 시스템에서 사용 가능

**403 Forbidden:**
- 서버가 요청 이해했지만 승인 거부
- 권한 부족, IP 차단 등

**404 Not Found:**
- 서버가 요청된 자원을 찾지 못함
- 가장 일반적인 오류

**405 Method Not Allowed:**
- 요청 메서드가 자원에서 지원되지 않음
- Allow 헤더로 지원 메서드 목록 제공

**406 Not Acceptable:**
- 요청의 Accept 헤더에 맞는 자원 없음
- 컨텐츠 협상 실패

**407 Proxy Authentication Required:**
- 프록시 인증 필요
- Proxy-Authenticate 헤더 포함

**408 Request Timeout:**
- 서버가 요청 대기 중 타임아웃
- 클라이언트는 같은 요청 재시도 가능

**409 Conflict:**
- 요청이 서버 현재 상태와 충돌
- 동시 수정, 제약 조건 위반 등

**410 Gone:**
- 자원이 영구적으로 제거됨 (404와 차이)
- 클라이언트는 링크/리소스 제거해야 함

**411 Length Required:**
- Content-Length 헤더 필요 but 없음
- POST, PUT 요청에서 발생

**412 Precondition Failed:**
- 조건부 요청의 전제조건 실패
- If-Match, If-Unmodified-Since 등

**413 Payload Too Large:**
- 요청 본문이 서버 제한 초과
- 서버는 Retry-After 헤더로 재시도 시간 제안

**414 URI Too Long:**
- 요청 URI가 서버 처리 가능 길이 초과
- 일반적으로 8KB 제한

**415 Unsupported Media Type:**
- 서버가 본문의 미디어 타입 지원 안 함
- Content-Type 문제

**416 Range Not Satisfiable:**
- 요청된 범위가 자원의 범위를 벗어남
- Content-Range 헤더로 유효 범위 알림

**417 Expectation Failed:**
- Expect 헤더의 기대사항 충족 불가
- 주로 Expect: 100-continue 실패

**418 I'm a teapot:**
- RFC 2324 (HTCPCP)의 만우절 농담
- 실제 사용 안 함

**421 Misdirected Request:**
- 서버가 요청을 처리할 수 있는 서버 아님
- HTTP/2 연결 재사용 문제에서 발생

**422 Unprocessable Entity (WebDAV):**
- 요청 형식은 올바르나 의미론적 오류
- XML 구문 오류, 검증 실패 등

**423 Locked (WebDAV):**
- 접근하려는 자원이 잠겨 있음

**424 Failed Dependency (WebDAV):**
- 이전 요청 실패로 현재 요청 실패

**425 Too Early:**
- 요청이 재생(replay) 공격 가능성 있음
- TLS 1.3 0-RTT와 관련

**426 Upgrade Required:**
- 클라이언트가 프로토콜 업그레이드 필요
- Upgrade 헤더로 제안

**428 Precondition Required:**
- 조건부 요청 필요 (If-Match 등)
- 동시 수정 방지

**429 Too Many Requests:**
- 속도 제한 초과
- Retry-After 헤더로 재시도 시간 알림

**431 Request Header Fields Too Large:**
- 헤더 필드가 너무 커서 서버 처리 불가
- 개별 필드 또는 전체 헤더 크기 초과

**451 Unavailable For Legal Reasons:**
- 법적 이유로 자원 접근 불가
- 로보트스 배제 표준에서 정의

#### 5xx - 서버 오류

**500 Internal Server Error:**
- 일반적 서버 오류, 구체적 정보 없음
- 예상치 못한 조건 발생

**501 Not Implemented:**
- 서버가 요청 메서드 지원 안 함
- 서버 기능 부족

**502 Bad Gateway:**
- 게이트웨이나 프록시가 업스트림 서버로부터 잘못된 응답
- 주로 로드 밸런서, CDN, 프록시에서 발생

**503 Service Unavailable:**
- 서버가 요청 처리할 준비 안 됨
- 과부하, 유지보수 중
- Retry-After 헤더로 가용 시간 알림 가능

**504 Gateway Timeout:**
- 게이트웨이나 프록시가 업스트림 서버 응답 대기 중 타임아웃
- 502와 유사 but 타임아웃 관련

**505 HTTP Version Not Supported:**
- 요청 HTTP 버전을 서버가 지원 안 함
- Upgrade 헤더로 지원 버전 알림 가능

**506 Variant Also Negotiates:**
- 투명한 컨텐츠 협상에서 순환 참조 발생

**507 Insufficient Storage (WebDAV):**
- 자원 저장 공간 부족

**508 Loop Detected (WebDAV):**
- 무한 루프 감지 (예: 디렉토리 구조)

**510 Not Extended:**
- 요청에 추가 확장 필요
- 서버가 확장을 처리할 정책 없음

**511 Network Authentication Required:**
- 네트워크 접근 전 인증 필요
- 공용 와이파이 포털 등

### 상태 코드 사용 모범 사례

**RESTful API 설계:**
```
GET    /users/123      → 200 (성공) 또는 404 (없음)
POST   /users         → 201 (생성됨) + Location 헤더
PUT    /users/123      → 200 (성공) 또는 204 (본문 없음)
DELETE /users/123      → 204 (본문 없음)
PATCH  /users/123      → 200 (성공)
```

**오류 처리:**
```
400: 클라이언트 입력 오류
401: 인증 필요
403: 권한 부족
404: 자원 없음
409: 충돌 (동시 수정 등)
422: 유효성 검사 실패
429: 속도 제한 초과
500: 서버 내부 오류
503: 서비스 일시 중단
```

**캐싱 전략:**
```
200: 캐시 가능 (캐시 제어 헤더 따름)
301: 영구적 리다이렉트 캐시
302/307: 일시적 리다이렉트 캐시 안 함
304: 캐시된 버전 유효
```

### 사용자 정의 상태 코드

IETF에 등록되지 않은 상태 코드는 사용을 피해야 하지만, 필요한 경우 확장 가능:

**확장 코드 규칙:**
- x00-x99 범위는 IANA에 등록 가능
- 등록되지 않은 코드는 예측 불가능한 동작 발생 가능
- 클라이언트가 인식하지 못하면 같은 클래스의 x00 코드로 간주

**실제 확장 코드 예시:**
- **520 Unknown Error**: Cloudflare의 일반적 서버 오류
- **521 Web Server Is Down**: Cloudflare의 원본 서버 다운
- **522 Connection Timed Out**: Cloudflare의 연결 타임아웃
- **523 Origin Is Unreachable**: Cloudflare의 원본 서버 도달 불가
- **524 A Timeout Occurred**: Cloudflare의 핸드셰이크 타임아웃

## 결론

HTTP 메시지 형식, 메서드, 상태 코드는 웹 아키텍처의 근간을 이루는 세 가지 핵심 구성 요소입니다. 이들의 조화로운 상호작용은 단순한 데이터 전송을 넘어, 분산 시스템 간의 의미 있는 통신을 가능하게 합니다. HTTP의 텍스트 기반 설계는 의도적인 선택이었으며, 이로 인해 개발자 친화성, 디버깅 용이성, 진화적 유연성을 얻을 수 있었습니다.

HTTP 메시지 형식의 구조적 우아함은 그 단순함에서 비롯됩니다. 시작 줄, 헤더, 본문의 명확한 분리는 파싱을 쉽게 만들면서도 확장성을 보장합니다. 특히 헤더 메커니즘은 프로토콜 발전의 핵심 동인으로 작용해 왔으며, 캐싱, 인증, 협상, 보안 등 다양한 기능을 표준화된 방식으로 추가할 수 있게 했습니다.

HTTP 메서드의 의미론적 설계는 REST 아키텍처 스타일의 기반을 제공합니다. 각 메서드의 안전성과 멱등성 특성은 분산 시스템의 예측 가능성과 신뢰성을 보장합니다. 이는 단순한 기술적 명세를 넘어, 웹 자원에 대한 통일된 인터페이스를 정의함으로써 상호운용성을 극대화합니다. 특히 GET의 안전성과 멱등성은 웹의 캐싱 가능성과 검색 가능성의 기초가 되었습니다.

상태 코드 체계는 HTTP의 또 다른 혁신입니다. 5개의 클래스로 구분된 이 체계는 클라이언트가 서버 응답을 프로그램적으로 처리할 수 있게 합니다. 1xx의 정보성, 2xx의 성공, 3xx의 리다이렉션, 4xx의 클라이언트 오류, 5xx의 서버 오류라는 명확한 분류는 오류 처리와 복구 전략을 표준화합니다. 특히 200, 404, 500과 같은 코드는 인터넷 문화의 일부가 되었을 정도로 보편화되었습니다.

현대 웹 환경에서 이러한 기본 개념들은 계속해서 진화하고 있습니다. HTTP/2와 HTTP/3은 성능과 보안을 개선했지만, 논리적 메시지 모델은 이전 버전과 호환성을 유지합니다. 마찬가지로 새로운 메서드(PATCH 등)와 상태 코드(425, 451 등)가 추가되면서 프로토콜은 현실 세계의 요구에 부응하고 있습니다.

네트워크 및 웹 개발 전문가에게 HTTP 메시지 형식, 메서드, 상태 코드에 대한 깊은 이해는 필수적입니다. 이는 단순한 API 사용을 넘어, 효율적이고 안전하며 확장 가능한 웹 시스템을 설계하고 구현하는 데 필요한 기초 지식입니다. 웹이 계속해서 진화하는 한, 이러한 기본 원칙들은 새로운 기술과 패러다임의 토대 역할을 계속할 것입니다.