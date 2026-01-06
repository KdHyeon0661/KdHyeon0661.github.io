---
layout: post
title: TCPIP - HTTP 헤더
date: 2025-12-08 18:25:23 +0900
category: TCPIP
---
# HTTP 메시지 헤더

HTTP 메시지 헤더는 클라이언트와 서버 간의 통신을 제어하고 메타데이터를 전달하는 핵심 메커니즘입니다. 헤더들은 단순한 키-값 쌍을 넘어, 웹의 성능, 보안, 캐싱, 콘텐츠 협상 등 다양한 기능을 구현하는 구성 요소입니다.

## HTTP 일반 헤더

일반 헤더는 요청과 응답 모두에서 사용될 수 있으며, 메시지 전체의 동작을 제어합니다.

### 캐싱 제어 헤더

**Cache-Control:**
가장 중요한 캐싱 헤더로, 다양한 디렉티브를 통해 정교한 캐싱 정책을 구현합니다.

```
Cache-Control: max-age=3600, public, must-revalidate

주요 디렉티브:
- max-age=<초>: 캐시 유효 시간 (상대적)
- s-maxage=<초>: 공유 캐시(프록시)용 유효 시간
- public: 공개 캐시 가능
- private: 개인 캐시만 가능 (브라우저)
- no-cache: 캐시 사용 전 서버 재검증 필요
- no-store: 어떤 형태로도 저장 금지
- must-revalidate: 만료 후 반드시 재검증
- immutable: 콘텐츠 변경되지 않음을 보장
```

**Pragma:**
HTTP/1.0 호환성을 위한 레거시 헤더 (현재는 Cache-Control 사용 권장)

```
Pragma: no-cache  # HTTP/1.0 스펙의 캐시 제어
```

### 연결 관리 헤더

**Connection:**
```
Connection: keep-alive  # 지속 연결 요청
Connection: close       # 트랜잭션 후 연결 종료
Connection: Upgrade     # 프로토콜 업그레이드 준비
```

**Keep-Alive:**
지속 연결의 파라미터를 지정합니다 (HTTP/1.1에서는 기본값).

```
Keep-Alive: timeout=5, max=100
- timeout: 유휴 상태 유지 시간(초)
- max: 최대 요청 수
```

### 메시지 포워딩 헤더

**Via:**
메시지가 통과한 중간 노드(프록시, 게이트웨이) 정보를 기록합니다.

```
Via: 1.1 proxy1.example.com, 1.0 proxy2.example.com
형식: HTTP-버전 호스트 [설명]
```

**Forwarded:**
RFC 7239에 정의된 현대적 프록시 헤더로, 클라이언트 원본 정보를 보존합니다.

```
Forwarded: for=192.0.2.60;proto=http;by=203.0.113.43
파라미터:
- for: 클라이언트 IP
- proto: 사용 프로토콜
- host: 원본 Host 헤더 값
- by: 요청을 수신한 노드 식별자
```

### 기타 일반 헤더

**Date:**
메시지 생성 일시를 RFC 1123 형식으로 표시합니다.

```
Date: Tue, 15 Nov 1994 08:12:31 GMT
```

**Warning:**
메시지 상태에 대한 추가 정보나 경고를 제공합니다.

```
Warning: 199 - "Miscellaneous warning"
경고 코드:
- 1xx: 캐시에 대한 경고
- 2xx: 데이터 표현 변환 관련
- 199: 기타 경고
```

## HTTP 요청 헤더

요청 헤더는 클라이언트가 서버에 추가 정보를 전달하고, 원하는 응답의 특성을 지정합니다.

### 클라이언트 식별 헤더

**User-Agent:**
클라이언트 애플리케이션(브라우저) 정보를 서버에 알립니다.

```
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 
           (KHTML, like Gecko) Chrome/91.0.4472.124 Safari/537.36

구성 요소:
- 제품: Mozilla (역사적 호환성)
- 플랫폼: Windows NT 10.0
- 엔진: AppleWebKit/537.36
- 브라우저: Chrome/91.0.4472.124
```

**From:**
사용자 이메일 주소를 제공합니다 (일반적으로 로봇에서 사용).

```
From: webmaster@example.com
```

### 콘텐츠 협상 헤더

클라이언트가 선호하는 콘텐츠 형식을 서버에 알려, 서버가 최적의 표현을 선택할 수 있게 합니다.

**Accept:**
선호하는 미디어 타입을 품질 값(q)과 함께 지정합니다.

```
Accept: text/html, application/xhtml+xml, application/xml;q=0.9, */*;q=0.8

해석:
1. text/html (기본 q=1.0)
2. application/xhtml+xml (q=1.0)
3. application/xml (q=0.9)
4. */* (q=0.8) - 모든 타입
```

**Accept-Charset, Accept-Encoding, Accept-Language:**
```
Accept-Charset: utf-8, iso-8859-1;q=0.5
Accept-Encoding: gzip, deflate, br
Accept-Language: ko-KR, ko;q=0.9, en-US;q=0.8, en;q=0.7
```

### 조건부 요청 헤더

특정 조건이 만족될 때만 응답을 받기 위한 헤더들입니다.

**If-Modified-Since:**
지정 시간 이후 변경된 경우만 응답 요청.

```
If-Modified-Since: Sat, 29 Oct 1994 19:43:31 GMT
```

**If-None-Match:**
ETag가 일치하지 않는 경우만 응답 요청.

```
If-None-Match: "737060cd8c284d8af7ad3082f209582d"
```

**If-Match, If-Unmodified-Since:**
낙관적 동시성 제어(optimistic concurrency control)에 사용.

```
If-Match: "737060cd8c284d8af7ad3082f209582d"
# ETag가 일치할 때만 요청 처리
```

### 인증 헤더

**Authorization:**
인증 정보(자격 증명)를 서버에 전달합니다.

```
Authorization: Basic QWxhZGRpbjpvcGVuIHNlc2FtZQ==
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...

인증 스킴:
- Basic: Base64 인코딩 사용자:비밀번호
- Bearer: OAuth 2.0 액세스 토큰
- Digest: 챌린지-응답 인증
```

**Proxy-Authorization:**
프록시 서버 인증용.

### 요청 컨텍스트 헤더

**Referer:**
현재 요청을 유발한 이전 리소스의 URI.

```
Referer: https://example.com/page1.html
주의: 철자가 Referrer가 아닌 Referer (역사적 오타가 표준이 됨)
```

**Origin:**
CORS 요청에서 요청 출처를 명시.

```
Origin: https://example.com
```

## HTTP 응답 헤더

응답 헤더는 서버가 클라이언트에 추가 정보를 제공하고, 응답의 처리를 제어합니다.

### 서버 정보 헤더

**Server:**
서버 소프트웨어 정보.

```
Server: Apache/2.4.1 (Unix) OpenSSL/1.0.2g
보안 권고: 민감한 정보 노출 최소화
```

### 콘텐츠 협상 결과 헤더

**Vary:**
서버가 응답을 선택할 때 고려한 요청 헤더를 지정.

```
Vary: Accept-Encoding, User-Agent
의미: Accept-Encoding과 User-Agent가 다른 캐시 키 생성
```

### 리디렉션 헤더

**Location:**
리디렉션할 URI를 지정 (3xx 상태 코드와 함께 사용).

```
HTTP/1.1 302 Found
Location: https://example.com/new-location
```

### 보안 관련 헤더

**Strict-Transport-Security (HSTS):**
브라우저가 HTTPS로만 접속하도록 강제.

```
Strict-Transport-Security: max-age=31536000; includeSubDomains
- max-age: 정책 유효 시간(초)
- includeSubDomains: 모든 서브도메인 포함
- preload: HSTS 프리로드 리스트 포함
```

**Content-Security-Policy (CSP):**
크로스사이트 스크립팅(XSS) 공격 방지.

```
Content-Security-Policy: default-src 'self'; script-src 'self' https://apis.google.com
```

### 캐싱 관련 응답 헤더

**Age:**
응답이 캐시에 저장된 시간(초).

```
Age: 3600  # 1시간 전에 캐시됨
```

**ETag:**
리소스의 특정 버전을 식별하는 고유 값.

```
ETag: "33a64df551425fcc55e4d42a148795d9f25f89d4"
ETag: W/"0815"  # 약한 ETag (내용은 같지만 표현이 다를 수 있음)
```

### 요청 제한 헤더

**Retry-After:**
서비스 이용 가능 시점이나 재시도 간격.

```
Retry-After: 120  # 120초 후 재시도
Retry-After: Fri, 31 Dec 1999 23:59:59 GMT  # 특정 시간까지 대기
```

## HTTP 엔티티 헤더

엔티티 헤더는 메시지 본문(엔티티)에 대한 메타데이터를 설명합니다.

### 콘텐츠 설명 헤더

**Content-Type:**
엔티티의 미디어 타입과 문자 집합을 지정.

```
Content-Type: text/html; charset=utf-8
Content-Type: application/json
Content-Type: multipart/form-data; boundary=something

주요 MIME 타입:
- text/*: 텍스트 (html, plain, css)
- image/*: 이미지 (jpeg, png, gif)
- application/*: 이진 데이터 (json, xml, pdf)
- multipart/*: 여러 부분으로 구성
```

**Content-Length:**
엔티티 본문의 크기(바이트).

```
Content-Length: 3495
```

**Content-Encoding:**
적용된 압축 알고리즘.

```
Content-Encoding: gzip
Content-Encoding: br  # Brotli
```

### 콘텐츠 위치 헤더

**Content-Location:**
엔티티의 대체 위치.

```
Content-Location: /documents/example.html
```

### 콘텐츠 범위 헤더

**Content-Range:**
부분 콘텐츠 응답에서 범위 지정.

```
Content-Range: bytes 21010-47021/47022
형식: 단위 시작-끝/전체크기
```

### 다운로드 제어 헤더

**Content-Disposition:**
브라우저가 콘텐츠를 어떻게 처리할지 지시.

```
Content-Disposition: attachment; filename="example.pdf"
Content-Disposition: inline  # 브라우저 내 표시
```

### 언어 및 인코딩 헤더

**Content-Language:**
대상 청중의 자연어.

```
Content-Language: ko
Content-Language: en-US, ko
```

### 최종 수정 헤더

**Last-Modified:**
리소스가 마지막으로 수정된 시간.

```
Last-Modified: Tue, 15 Nov 1994 12:45:26 GMT
```

---

## 결론

HTTP 헤더 시스템은 웹 프로토콜의 유연성과 확장성의 핵심입니다. 단순한 키-값 쌍의 집합을 넘어, 헤더들은 웹의 복잡한 요구사항을 충족시키기 위한 정교한 메커니즘으로 발전해 왔습니다.

HTTP 헤더의 가장 큰 강점은 **계층적 추상화**에 있습니다. 일반 헤더, 요청 헤더, 응답 헤더, 엔티티 헤더로의 분류는 각 헤더의 역할과 사용 컨텍스트를 명확히 구분하며, 이는 프로토콜 구현과 이해를 크게 단순화합니다. 이러한 구조적 명확성은 수백 개의 헤더가 정의된 현대 HTTP에서도 일관된 이해를 가능하게 합니다.

콘텐츠 협상 헤더(Accept-* 계열)는 웹이 다양한 클라이언트와 사용자 요구에 어떻게 적응하는지를 보여주는 뛰어난 설계입니다. 품질 값(q) 매커니즘을 통한 선호도 표현은 단순하면서도 강력한 협상 수단을 제공하며, 이는 국제화, 접근성, 장치 최적화 등 다양한 요구를 충족시키는 기반이 됩니다.

캐싱 관련 헤더들은 웹 성능 최적화의 핵심 도구입니다. Cache-Control의 다양한 디렉티브들은 정적 콘텐츠의 효율적 배포에서 동적 콘텐츠의 신선도 유지에 이르기까지 광범위한 캐싱 전략을 구현할 수 있게 합니다. 특히 ETag와 조건부 요청의 조합은 대역폭 절약과 데이터 일관성 사이의 완벽한 균형을 제공합니다.

보안 헤더들의 진화는 웹이 어떻게 새로운 위협에 대응하는지를 보여줍니다. HSTS, CSP, CORS 관련 헤더들은 모두 특정 보안 취약점을 해결하기 위해 도입되었으며, 이는 프로토콜이 실전 요구사항에 지속적으로 적응하는 살아있는 시스템임을 입증합니다.

현대 웹 개발에서 HTTP 헤더의 중요성은 더욱 커지고 있습니다. 웹 성능 최적화(Preload, Prefetch 헤더), 보안 강화(보안 헤더들), API 설계(사용자 정의 헤더) 등 모든 영역에서 헤더들은 핵심적인 역할을 수행합니다. 헤더를 효과적으로 이해하고 활용하는 것은 현대 웹 개발자와 아키텍트에게 필수적인 역량이 되었으며, 이는 단순한 기술적 지식을 넘어 시스템적 사고와 최적화 능력을 요구합니다.