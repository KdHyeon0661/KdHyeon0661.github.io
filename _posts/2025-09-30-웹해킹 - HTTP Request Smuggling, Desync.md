---
layout: post
title: 웹해킹 - HTTP Request Smuggling, Desync
date: 2025-09-30 17:25:23 +0900
category: 웹해킹
---
# HTTP Request Smuggling / Desync (H1/H2)

## 한눈에 보는 요약

### 1. 문제(Desync)의 본질

- HTTP Request Smuggling / Desync는  
  **클라이언트 → 프록시(또는 CDN/LB) → 백엔드** 사이에서  
  **“HTTP 요청 경계(boundary)”를 다르게 해석**하는 틈을 이용하는 공격이다.
- 대표적으로:
  - 한쪽은 `Content-Length`(CL)를 기준으로,
  - 다른 쪽은 `Transfer-Encoding: chunked`(TE)를 기준으로,
  - 또는 **헤더 중복/버그/HTTP/2→1 변환** 때문에  
    **“이 지점이 요청 1의 끝이다”**에 대한 합의가 깨질 때,
  - 프록시·백엔드 중 **어느 한 쪽이 다음 사용자의 요청 일부를  
    “앞선 요청의 바디”로 붙여서 보내거나, 반대로 남긴 부분을 다음 요청으로 해석**하게 된다.
- 결과적으로:
  - **세션 하이재킹**(다른 사용자의 요청/쿠키를 가로채거나 섞음)
  - **캐시 오염(Cache Poisoning)** 및 **ACL 우회**
  - **로그 교란**, **WAF 우회**, 심지어 **0-day 형태의 인증 우회**까지 이어진다.

최근 몇 년 사이에도:

- 2019년 PortSwigger의 “HTTP Desync Attacks” 연구 이후,  
  **대형 리버스 프록시·CDN·웹 서버에서 다수의 Request Smuggling 취약점이 발견**되었다. :contentReference[oaicite:0]{index=0}  
- 2020년 Black Hat USA에서는 **주요 서버/프록시 스택 전반(IIS, Apache, nginx, Varnish, HAProxy, Tomcat 등)**에서  
  **새로운 변종(Request Smuggling in 2020)**이 보고되며, 이 취약점이 “옛날 얘기”가 아님이 확인되었다. :contentReference[oaicite:1]{index=1}  
- 2025년에는:
  - **ASP.NET Core Kestrel**에서 9.9/10 점의 **치명적 Request Smuggling(CVE-2025-55315)** 이 보고되었고, :contentReference[oaicite:2]{index=2}  
  - 글로벌 CDN 업체의 플랫폼에서도 **Request Smuggling 및 캐시 오염(CVE-2025-32094, CVE-2025-4366)** 이 연달아 보고되었다. :contentReference[oaicite:3]{index=3}  
- MITRE CWE-444는 이를 **“HTTP 요청의 불일치 해석(Inconsistent Interpretation)”** 문제로 정식 분류한다. :contentReference[oaicite:4]{index=4}  

즉, **2025년 현재에도** HTTP Request Smuggling / Desync는 **실제 대형 벤더에서 계속 CVE가 나오는 “살아 있는” 취약점**이다.

### 2. 전형적인 징후

- 프록시/캐시를 **경유한 요청과, 백엔드에 직접 붙은 요청의 응답이 다르다.**
- 간헐적으로 다음과 같은 **에러 코드가 터진다.**
  - `400 Bad Request`, `408 Request Timeout`, `411 Length Required`,  
    `413 Payload Too Large`, `502 Bad Gateway`, `504 Gateway Timeout`
- 엣지/프록시 에러 로그에:
  - `upstream prematurely closed connection`
  - `client prematurely closed connection while reading client request body`
  - `upstream sent too big header while reading response header from upstream`
  - 같은 메시지가 **간헐적으로 스파이크** 형태로 쌓인다.
- 특정 경로/호스트에서 **캐시 적중률(HIT)이 이상하게 튄다**거나,  
  버그 리포트에 “가끔 **다른 사람 페이지가 보인다**”는 식의 현상이 보고된다.

### 3. 대표 원인(요약)

1. **H1/H1 조합**
   - **CL vs TE 충돌**
     - `Content-Length` vs `Transfer-Encoding: chunked`  
       어느 쪽을 신뢰할지에 대한 우선순위가 다르거나,
       둘 다 존재할 때 벤더마다 다른 정책을 씀.
   - **중복 Content-Length(CL.CL)**
     - CL 헤더가 두 번 이상 있을 때
       - 첫 값만 사용
       - 마지막 값 사용
       - 병합/에러 처리 등 **서버마다 차이**.
2. **H2/H1 변환**
   - HTTP/2는 바디 길이를 헤더로 쓰지 않고, **프레임 길이 필드로만 구분**한다.
   - 프록시가 H2→H1로 다운그레이드하면서  
     `Content-Length`를 “추가로 만들어 넣는”데,  
     이 로직이 백엔드의 H1 파서 해석과 어긋나면 Desync.
   - 특히 HTTP/2의 “스트림 재사용·우선순위·플로우 제어”까지 섞이면  
     **H2/H1 Desync**가 상당히 복잡해진다. :contentReference[oaicite:5]{index=5}  
3. **프록시 체인 이질성**
   - **프론트 CDN**, **중간 API 게이트웨이**, **내부 리버스 프록시**, **앱 서버**가  
     각각 **다른 HTTP 파서/버전/패치 수준**을 가지고 있을 때
   - 예:  
     - 프론트: TE 우선, 백엔드: CL 우선  
     - 프론트: 이중 CL 허용, 백엔드: 첫 CL만 참조  
   - MITRE CWE-444가 말하는 “여러 컴포넌트의 해석 일관성 붕괴”가 바로 이 케이스다. :contentReference[oaicite:6]{index=6}  

### 4. 핵심 방어 전략(요약)

- **엣지(프록시/CDN)에서 모든 요청을 “정상화 + 완충” 후, 백엔드에는 단일 표준 형태로 전달**
- 구체적으로:
  1. **요청 버퍼링(Buffering) 활성화**
     - 프록시가 클라이언트 바디 전체를 읽고 검증한 뒤,  
       백엔드에는 **단일 `Content-Length` 기반 요청**으로 재전달.
  2. **헤더 엄격 검증**
     - `Content-Length` **중복** → 무조건 400
     - `Content-Length`와 `Transfer-Encoding`이 **동시에 존재** → 400
     - 필요하지 않은 `Transfer-Encoding`은 **완전히 차단** 또는 **제거**
  3. **H2C 비활성 + H2/H1 변환 규칙 고정**
     - HTTP/2 cleartext(H2C)는 비활성.
     - 엣지가 H2를 수신해도, 백엔드로는  
       **단일 CL을 가진 H1 요청**만 보내도록 일관되게 구성.
  4. **스택 통일 및 최신 패치**
     - 엣지·중간 프록시·백엔드가 가능한 한 **동일 벤더/버전**의 HTTP 스택 사용.
     - 2025년에도 주요 스택(Varnish, HAProxy, Apache, ASP.NET Core 등)에서  
       새로운 Request Smuggling CVE가 계속 보고되고 있으므로  
       **패치 추적**이 필수. :contentReference[oaicite:7]{index=7}  
  5. **WAF/IDS 룰**
     - 이중 `Content-Length`, CL+TE 동시 존재, TE에 비정상 값,  
       비표준 헤더 순서 등을 **정책적으로 차단/경보**.

---

## 동작 원리와 분류

### 왜 “경계(boundary)”가 어긋나는가?

HTTP/1.1에서 **요청 본문이 어디서 끝나는지**를 정하는 방법은 크게 두 가지다.

1. **Content-Length (CL)**
   - 바디 길이를 **바이트 단위 정수**로 명시.
   - 서버는 헤더를 읽은 후, CL만큼 정확히 읽으면 요청 1개 끝.
2. **Transfer-Encoding: chunked (TE)**
   - 바디를 여러 개의 “청크”로 나눠 보내고,
   - 각 청크 앞에 그 길이(16진수)를 쓰고, 마지막에 `0\r\n\r\n` 청크로 종료.

규격(RFC 9112 기준) 상으로는: :contentReference[oaicite:8]{index=8}  

- **CL과 TE를 동시에 둘 수 없고**,  
- TE가 있으면 TE를 기준으로 해석해야 하며,  
- 이상한 조합은 400으로 거절하는 것이 맞다.

하지만 현실의 구현은:

- 오래된/커스텀 프록시, 일부 라이브러리, 프레임워크가
  - CL과 TE가 동시에 있을 때 **CL 우선** 또는 **TE 우선**  
    또는 에러 없이 **비일관 처리**를 하고,
  - 중복 CL에 대해 첫 값/마지막 값/합산 등  
    제각각 정책을 사용한다.
- HTTP/2, HTTP/3, gRPC, 웹소켓 등 **다양한 상위 프로토콜**이  
  각자의 프레이밍 규칙을 가지고 있고,  
  이를 HTTP/1.1으로 변환하면서 버그가 나온다. :contentReference[oaicite:9]{index=9}  

결국, **클라이언트–프록시–백엔드가 “요청 1개가 어디까지인지”에 대해 서로 다른 그림을 갖게 되고**,  
이 틈을 이용해 **다른 사람의 요청 일부를 “내가 보낸 요청의 바디”로 흡수하거나, 반대로 남긴 조각으로 다음 요청을 구성**하게 만드는 것이 **Request Smuggling / Desync**이다. :contentReference[oaicite:10]{index=10}  

---

### 유형(콘셉트) — CL.TE, TE.CL, CL.CL, H2/H1

> 아래는 **개념적인 흐름**만 설명한다.  
> 실제 취약 조합 페이로드는 생략하고, **방어·이해용 수준**으로 단순화한다.

#### 공통 그림: keep-alive 파이프라인

```
[클라이언트] ──(연결 1개, 여러 요청)──> [프록시] ──(연결 1개, 여러 요청)──> [백엔드]
```

- Request Smuggling은 이 **한 연결 안에 “요청 N개”를 나란히 태우는 구조**를 이용한다.
- 공격자는 첫 번째 요청을 **미묘하게 손상된 형태**로 만들어:
  - 프록시는 “여기까지가 요청 1”이라고 생각하지만,
  - 백엔드는 “아직 바디가 덜 왔다”고 생각하게 만들고,
- 그 사이에 **다음 사용자의 요청 또는 공격자의 두 번째 요청이 이어서 들어오도록** 유도한다.

이를 유형별로 나눠보면:

#### 1) CL.TE

- **프론트(프록시)**: `Content-Length` 기반으로 경계 해석
- **백엔드**: `Transfer-Encoding: chunked` 기반으로 해석

| 유형 | 프론트(프록시) | 백엔드 | 결과 |
|------|----------------|--------|------|
| CL.TE | CL 우선 | TE 우선 | 프론트가 남긴 바디/데이터가 백엔드에서 **다음 요청 일부**로 합쳐짐 |

- 공격자는 첫 요청에
  - `Content-Length`를 짧게 잡고,
  - `Transfer-Encoding: chunked`를 길게 잡는 식으로 **모순**을 만들 수 있다.
- 프록시는 CL 기준으로 **짧게 끊고**, 나머지 바디를 **다음 요청으로 넘긴다.**
- 백엔드는 TE 기준으로 **더 길게 읽으려고 하며**,  
  이때 “다음 요청 헤더”가 그대로 바디로 들어오게 된다.

#### 2) TE.CL

- **프론트**: TE(`chunked`) 기준
- **백엔드**: CL 기준

| 유형 | 프론트 | 백엔드 | 결과 |
|------|--------|--------|------|
| TE.CL | TE 우선 | CL 우선 | 프론트 입장에선 끝난 요청인데, 백엔드는 추가 바디를 기다리거나 반대로 더 읽음 |

- 이 경우엔 백엔드가 **남은 데이터**를  
  “다음 요청” 또는 “더 긴 바디”로 착각한다.
- 특정 조합에서는 **백엔드가 다음 사용자의 요청 헤더를 바디로 읽어버려**  
  **ACL 우회 / 캐시 오염**이 가능하다.

#### 3) CL.CL(이중 Content-Length)

- `Content-Length` 헤더가 여러 개 있을 때:

| 정책 | 예시 동작 |
|------|-----------|
| 첫 값 사용 | `CL: 10, CL: 100` → 10만 신뢰 |
| 마지막 값 사용 | 같은 예에서 100만 신뢰 |
| 에러 처리 | 중복이면 400 |

- 프론트/백엔드가 서로 다른 정책을 쓰면:
  - 프론트는 10, 백엔드는 100 또는 그 반대가 되어  
    다시 한 번 **경계가 어긋난다.**

#### 4) H2/H1 Desync

- HTTP/2는 요청 바디 길이를 **헤더에 적지 않고, DATA 프레임의 길이 필드**로만 나타낸다.
- 프록시가 H2를 받아 H1로 변환할 때:
  - **어떤 시점에 “요청 1개가 끝났다”**고 판단해서
  - `Content-Length`를 계산해 H1 요청을 만든다.
- 2022년 USENIX 연구(FrameshiFter)는 여러 구현에서  
  **H2→H1 변환 과정의 모호성/버그로 인해 Request Smuggling이 가능**함을 보여줬다. :contentReference[oaicite:11]{index=11}  

또한 PortSwigger의 최신 연구에서는 **0.CL, Chunk Extension 변조, H2/H3 기반 Desync** 등  
더 복잡한 변종들이 보고되고 있으며,  
그 중 일부는 **타임아웃·커넥션 풀 고갈 형태의 DoS**로도 악용 가능하다고 한다. :contentReference[oaicite:12]{index=12}  

---

## 징후와 관측 포인트

### 1. 응답 관측

- 같은 URL에 대해:
  - **프록시를 경유**했을 때와
  - **백엔드에 직접 붙었을 때**의 응답이 자주 다르다.
- 특히:
  - 프록시 경유: 가끔 400/408/502/504 또는 엉뚱한 HTML
  - 백엔드 직접: 항상 정상 페이지
- 또는, 같은 프록시 경유 요청인데도:
  - 어떤 사용자는 정상 페이지,  
  - 어떤 사용자는 **이상한 리다이렉트, 다른 사람의 응답, 엉뚱한 캐시된 HTML**을 받는다.

### 2. 로그·메트릭 징후

- **프록시 에러 로그**에 다음과 같은 메시지가 간헐적으로 몰려 들어온다.
  - `client prematurely closed connection while reading client request body`
  - `upstream prematurely closed connection`
  - `upstream sent too big header while reading response header from upstream`
- **에러율 히스토그램**:
  - 특정 경로나 시간대에 400/408/502/504가 **짧은 시간 스파이크**로 튄다.
- **프록시 vs 애플리케이션 로그 불일치**
  - 프록시 접근 로그에 기록된 `Content-Length`와
  - 애플리케이션에서 실제로 읽은 바디 길이(로그/메트릭으로 추적)가 **다르게 나타나는 경우**.

### 3. 보안 장비 경보

- WAF/IDS에서:
  - 이중 `Content-Length`
  - `Transfer-Encoding`에 비정상 값
  - `Content-Length`와 `Transfer-Encoding` 동시 존재
- 또는
  - HTTP/2→1 변환 구간에서 **비정상 헤더·프레이밍** 관련 경보.

이러한 징후가 **캐시 오염, 세션 탈취, ACL 우회 관련 리포트와 함께 나타난다면**,  
**Desync/Request Smuggling 의심도가 상당히 높다.**

---

## 테스트

> 목적: **실제 공격이 아니라**,  
> 프록시/게이트웨이 설정이 **“모순 헤더를 확실히 거절하고 있는지”**를  
> 사내 스테이징 환경에서 **안전하게 검증**하기 위함이다.  
> 실제 외부 서비스/3rd-party 시스템에 이런 테스트를 수행하는 것은 **법적/윤리적으로 위법**일 수 있다.

### 구성 예시(도커-스케치)

```text
[Client(Test Script)] → [edge(proxy)] → [app(server)]
```

- `edge`:
  - Nginx / HAProxy / Envoy / API Gateway 등
- `app`:
  - 간단한 Node.js/Flask 앱.
  - `/echo`, `/health` 등의 엔드포인트를 만들고  
    받은 헤더·바디 길이를 그대로 로그에 남기면 분석에 유리하다.

### 방어 검증 테스트 아이디어

- **케이스 A: 이중 Content-Length**
  - `Content-Length`가 두 번 이상 존재하는 요청 → **항상 400**이어야 한다.
- **케이스 B: CL + TE 동시 존재**
  - `Content-Length`와 `Transfer-Encoding: chunked`가 함께 존재 → **항상 400**이어야 한다.
- **케이스 C: HTTP/2 클라이언트 경로**
  - `curl --http2` 등으로 H2 요청을 보냈을 때도  
    엣지가 **내부에서 단일 `Content-Length`만 사용**하도록 변환하고 있는지.
- **케이스 D: 캐시/프록시 일관성**
  - 같은 URL에 대해 여러 번 요청했을 때:
    - 응답 바디·헤더가 항상 완전히 일치하는지,
    - 간헐적인 Desync 징후(헤더/바디 섞임)가 없는지.

---

### Node.js Raw TCP 테스트 — “이중 CL → 400 기대”

```js
// test/desync-guard.test.js (승인된 내부 스테이징용)
// 목적: "이중 Content-Length"를 엣지가 반드시 400으로 거절하는지 확인
import net from "node:net";

function sendRaw(raw) {
  return new Promise((resolve) => {
    const socket = net.connect(443, "edge.example.internal", () => {
      socket.write(raw);
    });
    let data = "";
    socket.on("data", (chunk) => (data += chunk.toString("utf8")));
    socket.on("end", () => resolve(data));
  });
}

(async () => {
  const req =
`POST /echo HTTP/1.1\r
Host: edge.example.internal\r
Content-Length: 5\r
Content-Length: 15\r
Connection: close\r
\r
HELLO`;

  const res = await sendRaw(req);
  console.log(res);
  // 기대: "HTTP/1.1 400 Bad Request" 또는 벤더 특유의 4xx 에러 응답
})();
```

- 이 테스트는 **Desync를 일으키지 않고**,  
  단지 **엣지가 “모순 헤더를 거절한다”는 방어 정책이 적용되었는지 확인**하는 용도이다.
- CI에서 주기적으로 돌리면,  
  **새로운 프록시 설정 변경/업데이트** 시 정책이 깨졌는지 빠르게 발견할 수 있다.

---

## 프록시·게이트웨이 방어 설정 (벤더 중립 원칙 + 대표 예시)

> 현실적인 전략은 **“엣지에서 모든 복잡성을 흡수”**하는 것이다.  
> 즉, 엣지가:
> - 클라이언트 요청을 **버퍼링 + 검증**하고,
> - 백엔드로는 **단 하나의 `Content-Length`만 포함된 H1 요청**을 보내도록 강제하는 것.

### 공통 원칙(요약)

1. **요청 버퍼링(Buffering) ON**
   - 프록시가 클라이언트 바디를 충분히 읽어들인 뒤,  
     내부에서 자체 버퍼를 기준으로 요청을 재구성해 백엔드로 보낸다.
2. **헤더 엄격 검증**
   - `Content-Length`가 두 개 이상이면 400.
   - `Content-Length`와 `Transfer-Encoding`이 동시에 존재하면 400.
   - 필요하지 않은 TE는 제거하거나 아예 허용하지 않는다.
3. **H2/H1 변환 정책**
   - 클라이언트에서는 `h2`를 받아도 상관없지만,
   - 백엔드로는 가급적 **H1 + 단일 `Content-Length`**로만 보내도록 제한.
4. **H2C 비활성**
   - 평문 HTTP/2(H2C)는 중간 프록시·로드밸런서와의 호환성 문제로  
     Request Smuggling/Desync 공격 표면을 넓힐 수 있으므로  
     특별한 이유가 없다면 꺼두는 것이 안전하다.
5. **Stack 통일**
   - 엣지·내부 프록시·백엔드가 가능한 한 **동일 제품/버전** 사용.
   - HTTP 파서 버그(CVE) 패치 여부를 **한 번에 관리**할 수 있다.

---

### Nginx 예시

```nginx
# ① 요청 버퍼링: 클라이언트로부터 전체 바디를 읽고 난 뒤 백엔드로 전달
proxy_request_buffering on;
proxy_http_version 1.1;

# ② Transfer-Encoding 검증/차단
#   - 여기서는 TE가 "chunked" 외의 값을 가지면 거절하는 패턴을 예시로 든다.
map $http_transfer_encoding $te_bad {
  default         1;        # 기본은 나쁨
  "~*^chunked$"   0;        # 정확히 chunked인 경우만 예외
}

server {
  listen 443 ssl http2;
  server_name edge.example.internal;

  # HTTP/2 튜닝 (필요시)
  http2_max_field_size 16k;
  http2_max_header_size 32k;

  location / {
    # 케이스: TE에 이상한 값이 오면 400
    if ($te_bad) { return 400; }

    # 백엔드에는 TE를 전달하지 않음
    proxy_set_header Transfer-Encoding "";
    # Nginx가 계산한 Content-Length만 사용
    proxy_set_header Content-Length $content_length;

    # 요청 크기/시간 제한
    client_max_body_size 10m;
    client_body_timeout 30s;
    proxy_read_timeout 30s;

    proxy_pass http://app_backend;
  }
}
```

- `proxy_request_buffering on`:
  - 클라이언트 요청을 **완전히 읽고** 백엔드로 보낸다.
  - H1/H1, H2/H1 구조에서도 백엔드 입장에선 “정상적인 H1 + CL 한 개”로 보이게 된다.
- `Transfer-Encoding` 제거:
  - 백엔드에서 TE/CL 동시 해석 버그가 터질 여지를 줄인다.

실제 운영에서는 여기에 **추가적으로**:

- `large_client_header_buffers`,  
- `client_header_timeout`,  
- `lingering_close` 등  
을 엄격하게 설정해 **비정상 요청에 대한 타임아웃·자원 통제**를 강화하는 것이 권장된다.

---

### HAProxy 예시 (개념적)

```haproxy
global
  log stdout format raw daemon
  tune.ssl.default-dh-param 2048

defaults
  mode http
  option httplog
  option http-keep-alive
  option http-buffer-request      # 요청 버퍼링
  timeout client  30s
  timeout server  30s
  timeout connect 5s

frontend fe_https
  bind :443 ssl crt /etc/haproxy/cert.pem alpn h2,http/1.1

  # 1) 이중 Content-Length 거절
  http-request deny status 400 if { req.hdr_cnt(content-length) gt 1 }

  # 2) Transfer-Encoding 자체를 금지(정책에 따라 허용 가능)
  http-request deny status 400 if { req.hdr(transfer-encoding) -m found }

  default_backend be_app

backend be_app
  option http-keep-alive
  server s1 app:8080 check
```

- 최신 HAProxy는 **HTX 엔진**을 통해 요청을 내부 표현으로 정규화하는 기능을 제공하고,  
  이를 적절히 활용하면 **복잡한 CL/TE 조합을 표준 형태로 강제**할 수 있다.
- 위 예시는 단순화된 패턴이지만,
  - **요청 버퍼링 + TE 차단 + 이중 CL 거절**이라는  
    **세 가지 핵심 축**은 대부분의 환경에서 그대로 적용 가능하다.

---

### Envoy 예시 (개념적 YAML)

```yaml
static_resources:
  listeners:
    - name: https
      address:
        socket_address: { address: 0.0.0.0, port_value: 443 }
      filter_chains:
        - filters:
            - name: envoy.filters.network.http_connection_manager
              typed_config:
                "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
                codec_type: AUTO   # H1/H2 자동
                stat_prefix: ingress_http
                stream_error_on_invalid_http_message: true
                request_headers_timeout: 0.5s
                http2_protocol_options:
                  allow_connect: false
                http_protocol_options:
                  # Envoy 최신 버전 기준, 헤더 검증 옵션 사용
                  allow_absolute_url: false
                route_config:
                  name: local_route
                  virtual_hosts:
                    - name: app
                      domains: ["*"]
                      routes:
                        - match: { prefix: "/" }
                          route:
                            cluster: app
                            timeout: 30s

  clusters:
    - name: app
      type: logical_dns
      connect_timeout: 2s
      lb_policy: round_robin
      load_assignment:
        cluster_name: app
        endpoints:
          - lb_endpoints:
              - endpoint:
                  address:
                    socket_address: { address: app, port_value: 8080 }
```

- Envoy는 기본적으로 HTTP 파서를 엄격하게 구현하고 있으며,
  - `stream_error_on_invalid_http_message: true` 와 같은 옵션을 통해  
    잘못된 메시지(이중 CL/CL+TE 등)를 **스트림 에러로 즉시 끊게** 할 수 있다.
- 실제 운영에서는 **Envoy 버전별 옵션 지원 여부**를 확인해  
  CL/TE 조합, 헤더 중복에 대한 정책을 명시적으로 설정하는 것이 좋다.

---

### Apache HTTPD / 기타 레거시 서버 설정 포인트

```apache
# HTTP/1.1 프로토콜 엄격 옵션
HttpProtocolOptions Strict

# 프록시 사용 시 요청 읽기 제한
ProxyRequests Off
RequestReadTimeout header=20-40,MinRate=500 body=20,MinRate=500

# 헤더 크기/개수 제한
LimitRequestFieldSize 16384
LimitRequestFields 100

# Transfer-Encoding 제거(프론트 프록시 뒤에 있는 앱 서버라면 특히)
RequestHeader unset Transfer-Encoding early
```

- Apache 2.4 계열에서도 mod_proxy_ajp 등에서 Request Smuggling 관련 CVE가 보고된 바 있다. :contentReference[oaicite:13]{index=13}  
- 가능하면 Request Smuggling 방어가 개선된 최신 버전으로 업그레이드하고,
  - **프론트 프록시에서 TE를 제거해 들어오게** 하거나,
  - Apache 레벨에서 TE 헤더를 조기에 제거/검증하는 것이 안전하다.

---

## 애플리케이션 레이어 완화책(보조 안전망)

> HTTP Request Smuggling의 **근본 원인**은 **HTTP 레이어(프록시/게이트웨이)**에서 해결해야 한다.  
> 그럼에도 불구하고 앱 자체에서도 **“모순 헤더를 가진 요청은 거부한다”**는 보조 안전망을 두면  
> 방어 레이어가 하나 더 생긴다.

### Node/Express — 모순 헤더 선 차단

```js
// 앱 진입점 상단에 위치시키는 미들웨어 예시
app.use((req, res, next) => {
  const te = req.headers["transfer-encoding"];

  // rawHeaders를 사용해 Content-Length 헤더 개수 계산
  let clCount = 0;
  for (let i = 0; i < req.rawHeaders.length; i += 2) {
    if (req.rawHeaders[i].toLowerCase() === "content-length") {
      clCount += 1;
    }
  }

  if (te) {
    return res.status(400).send("Bad Request");
  }

  if (clCount > 1) {
    return res.status(400).send("Bad Request");
  }

  next();
});
```

- 엣지에서 이미 TE/이중 CL을 차단하고 있다면  
  이 미들웨어는 **실제로 작동할 일이 거의 없겠지만**,  
  설정 실수나 미들 프록시 우회를 탔을 때 마지막 방어선을 제공한다.

### Flask — 무효 요청 거절

```python
from flask import Flask, request, abort

app = Flask(__name__)

@app.before_request
def reject_conflicts():
    cl_values = [
        v for (k, v) in request.headers.items()
        if k.lower() == "content-length"
    ]
    has_te = any(k.lower() == "transfer-encoding" for (k, _) in request.headers.items())

    if has_te:
        abort(400)
    if len(cl_values) > 1:
        abort(400)
```

- Python WSGI 스택 / 리버스 프록시 구성에 따라  
  이미 TE/CL 정규화가 끝난 상태로 들어올 수 있으나,
- 프락시 설정이 미흡할 때 “이중 CL을 가진 요청”이 앱까지 내려오는 것을  
  추가로 방지할 수 있다.

### 프레임워크·런타임 자체의 취약점

- 2025년 보고된 **ASP.NET Core Kestrel의 고위험 Request Smuggling 취약점(CVE-2025-55315)** 은,  
  **프레임워크 자체의 HTTP 파서 구현**에서 경계 해석 버그가 발생한 사례다. :contentReference[oaicite:14]{index=14}  
- 이런 취약점은 **앱 레벨 코드로는 방어가 불가능**하고,
  - **패치 적용**
  - **엣지 프록시에서 선제적 정규화**  
  를 통해 줄이는 수밖에 없다.

따라서, 애플리케이션 개발자는:

1. **런타임/프레임워크의 보안 공지와 CVE**를 주기적으로 확인하고,
2. Request Smuggling 관련 패치가 나오면 **신속히 업그레이드** 해야 한다.

---

## HTTP/2 관련 주의점

HTTP/2는:

- 텍스트가 아닌 **바이너리 프레이밍 프로토콜**이고,
- 각 요청/응답이 **스트림**으로 구분되며,
- 바디는 **DATA 프레임들의 길이 합**으로 표현된다.

이 구조 자체는 **H1의 CL/TE 모호성**을 상당 부분 없애지만,  
현실의 문제는 **H2 ↔ H1 변환**에서 생긴다. :contentReference[oaicite:15]{index=15}  

### H2/H1 Desync 패턴

- 엣지가 H2를 받고, 내부 서비스는 H1만 지원하는 경우:
  - 엣지는 H2 요청을 받아 **한 번에 H1 요청으로 변환**해야 한다.
  - 이때:
    - 어느 시점까지를 **“요청 1개로 묶을지”**,
    - `Content-Length`를 얼마나로 설정할지,
    - 여러 스트림을 하나의 H1 연결로 어떻게 매핑할지  
      구현 세부가 다르다.
- 여러 연구에 따르면:
  - 특정 구현에서는 “H2 프레임 조합 + 헤더 순서”를 교묘하게 조작해  
    H1 변환 결과가 **백엔드의 기대와 달라지도록** 만들 수 있다. :contentReference[oaicite:16]{index=16}  

### 실무 가이드

1. **H2C 비활성**
   - TLS 없는 HTTP/2는 중간 구간에서 **다양한 프록시/로드밸런서와의 미묘한 호환성 문제**를 낳는다.
2. **엣지에서 H2 → 내부는 H1 고정**
   - 엣지가 H2 요청을 충분히 버퍼링·검증해  
     내부 서비스에는 **항상 H1 + 단일 Content-Length**로 보내도록 한다.
3. **또는 전 구간 H2**
   - 엣지–내부 모두 H2를 사용하고,  
     **중간에 H1 변환 지점**을 두지 않는 것도 하나의 방법이다.
4. **변환 규칙 문서화/테스트**
   - 어떤 요청이 들어왔을 때 내부로 어떤 H1 요청이 나가는지  
     공식 도큐먼트/테스트로 남겨두면,  
     나중에 Desync 취약점 리포트가 들어왔을 때 **원인 분석이 훨씬 수월**하다.

---

## 캐시 오염(Cache Poisoning)과 결합 방지

HTTP Request Smuggling은 **단독으로도 위험**하지만,  
**캐시 시스템과 결합되면 “한 번의 오염 → 다수 사용자에게 전파”**라는 특성이 생긴다. :contentReference[oaicite:17]{index=17}  

### Desync → 캐시 오염 시나리오(개념)

1. 공격자가 **Smuggling 페이로드**를 포함한 요청을 보내,  
   프록시와 백엔드 사이에서 Desync를 일으킨다.
2. 그 결과:
   - 백엔드는 **공격자가 의도한 두 번째 요청**을 처리한 뒤,
   - 그 응답을 **첫 번째 요청에 대한 응답**으로 보내게 된다.
3. 이 응답이 **캐시에 저장**되면:
   - 동일 URL을 요청하는 다른 사용자들은  
     모두 **“스무글된(second) 요청에 대한 응답”**을 받게 된다.
   - 이는 곧:
     - **Access-Control 우회**
     - **세션/토큰 노출**
     - **XSS/JS 삽입 페이지** 장기간 노출
     - 등으로 이어질 수 있다.

### 방어 포인트 요약

- 캐시 키에서 **불필요한 헤더/파라미터를 제거/정규화**.
- `Transfer-Encoding`, 이중 `Content-Length` 등  
  **금지/모순 헤더가 포함된 요청/응답은 캐시하지 않도록 정책화**.
- 민감 응답:
  - 인증 필요, 세션 포함, `Set-Cookie` 동반, 개인화 등은  
    `Cache-Control: private, no-store`를 사용해 **공유 캐시에서 제거**.
- Request Smuggling 관련 CVE 보고 사례(예: 대형 CDN의 Pingora 기반 프록시 취약점)는,  
  **캐시 HIT 시 공격자가 임의 요청을 주입할 수 있었음**을 보여준다. :contentReference[oaicite:18]{index=18}  

---

## 모니터링/탐지 룰 예시

### 지표

- **Transfer-Encoding 사용률**
  - 클라이언트→엣지 구간에서, 정상 트래픽이라면 TE가 등장할 일이 거의 없다.
  - TE가 들어오면 **0에 가깝게** 유지되도록 정책을 만드는 것이 안전하다.
- **Content-Length 중복 건수**
  - 하루 기준 0건이어야 한다.
- **경로별 4xx/5xx 분포**
  - 특정 경로에서 400/408/502/504가 **짧은 시간 동안 튄다**면  
    Desync 시도 혹은 방어 정책이 작동하는 것일 수 있다.

### Loki(LogQL) 예시

```logql
# Transfer-Encoding이 등장하는 요청 빈도 체크
{service="edge", msg="http_access"} |= "Transfer-Encoding"
| count_over_time(5m)
```

### Splunk(SPL) 예시

```spl
index=edge msg=http_access ("Transfer-Encoding"=* OR content_length_count>1)
| stats count by src_ip, uri, user_agent
```

- TE 사용/중복 CL 등 **정책 위반 패턴**이 특정 IP·UA·경로에 몰려 있다면,  
  **침해 시도 또는 스캐닝**일 가능성이 높다.

---

## 운영 체크리스트 (배포 전/후)

1. [ ] **프록시**:
   - 요청 **버퍼링 ON**
   - 백엔드에는 **단일 `Content-Length`**만 전달
2. [ ] **TE 정책**:
   - 불필요한 `Transfer-Encoding` 헤더 → 차단/제거
   - CL+TE 동시 존재 시 → 400
3. [ ] **Content-Length 정책**:
   - 이중 CL → 400
4. [ ] **HTTP/2**:
   - H2C 비활성
   - H2/H1 변환 규칙 문서화
   - 최신 패치 적용(특히 H2→H1 변환 관련)
5. [ ] **스택 통일**:
   - 엣지–중간–백엔드 HTTP 파서/제품군/버전 일치 여부 확인
6. [ ] **요청 제한**:
   - 헤더 크기/개수 제한
   - 바디 크기 제한
   - 타임아웃(헤더/바디 읽기, 백엔드 응답)
7. [ ] **캐시 정책**:
   - 금지/모순 헤더 포함 요청/응답 → 캐시 금지
   - 민감 경로/응답 → `no-store` 또는 최소 private
8. [ ] **테스트**:
   - (스테이징) 이중 CL·CL+TE 요청에 400이 나오는지 자동화 테스트
   - H2 클라이언트에서 동일 결과인지 확인
9. [ ] **모니터링**:
   - TE 사용률, CL 중복률, 경로별 4xx/5xx 이상치 알림
10. [ ] **런북**:
    - Desync 의심 시 **임시 완화 조치(TE 전면 차단, 특정 경로 read-only)** 절차
    - 보안팀/운영팀 커뮤니케이션 플랜

---

## 사고 대응 런북(요약) — “Desync 의심” 시

1. **식별**
   - 프록시 경유/직결 응답 차이 확인
   - 에러율(400/408/502/504) 스파이크
   - 로그에서 이중 CL/CL+TE/비정상 TE 패턴 발견
2. **격리**
   - 엣지에서 `Transfer-Encoding` 전면 차단 또는 강제 제거
   - **요청 버퍼링 강제 ON**
   - 문제 경로/호스트에 대해 **캐시 무효화 또는 BYPASS**
3. **근절**
   - 프록시/게이트웨이/앱 레이어 설정 재점검
   - 관련 CVE(프록시/서버/프레임워크) 패치 적용
   - H2C 비활성, H2/H1 변환 로직 명시
4. **복구**
   - 부하/지연 모니터링
   - 캐시 재가온
   - 스모크 테스트(모순 헤더 거절, 응답 일관성)를 상시화
5. **교훈화**
   - CI 파이프라인에 **프로토콜 유효성 테스트** 추가
   - Request Smuggling/Desync를 **보안 아키텍처 리뷰 체크리스트 항목**에 포함
   - 주요 벤더의 Request Smuggling 관련 연구/블로그(US/EU 기반) 추적 :contentReference[oaicite:19]{index=19}  

---

## 부록 — “프로토콜 유효성” 자동화 테스트 아이디어

> 목표: 배포 전, **프록시/게이트웨이가 최소한의 프로토콜 유효성 검사를 제대로 하고 있는지**를  
> 자동 테스트로 확인하는 것.

### 테스트 케이스 설계

- **케이스 A: 이중 Content-Length**
  - 기대: **400 Bad Request**
- **케이스 B: CL + TE 동시 존재**
  - 기대: **400 Bad Request**
- **케이스 C: HTTP/2 클라이언트**
  - 기대: 헤더/바디가 H1과 동일하게 처리되고,  
    모순 헤더가 있으면 마찬가지로 400
- **케이스 D: 반복 요청 일관성**
  - 기대: 같은 입력에 대해 응답 바디·헤더가 항상 완전히 동일

### Node + TLS 예제 (케이스 A/B)

```js
// node validate-proxy.js (사내 CI 전용)
import tls from "node:tls";

function raw(host, port, payload) {
  return new Promise((resolve, reject) => {
    const s = tls.connect({ host, port, rejectUnauthorized: false }, () => {
      s.write(payload);
    });
    let buf = "";
    s.on("data", (d) => (buf += d.toString()));
    s.on("error", reject);
    s.on("end", () => resolve(buf));
  });
}

const reqA =
`POST /health HTTP/1.1\r
Host: edge.example.internal\r
Content-Length: 5\r
Content-Length: 10\r
Connection: close\r
\r
HELLO`;

const reqB =
`POST /health HTTP/1.1\r
Host: edge.example.internal\r
Content-Length: 5\r
Transfer-Encoding: chunked\r
Connection: close\r
\r
HELLO`;

(async () => {
  for (const [name, req] of [
    ["A-dupCL", reqA],
    ["B-CL+TE", reqB],
  ]) {
    const res = await raw("edge.example.internal", 443, req);
    if (!/^HTTP\/1\.[01] 400/.test(res)) {
      throw new Error(`❌ ${name}: expected 400, got\n${res.slice(0, 200)}`);
    }
    console.log(`✅ ${name}: 400`);
  }
})();
```

- 이 스크립트를 CI 파이프라인에 넣으면:
  - 프록시/게이트웨이 설정 변경, 버전 업그레이드, 새 WAF 도입 시  
    **프로토콜 유효성 방어가 깨졌는지**를 즉시 감지할 수 있다.

---

## 정리

- HTTP Request Smuggling / Desync는 **“HTTP 요청 경계 해석의 불일치”**에서 비롯되는 공격이다.
- 2005년 처음 학계에 보고된 이후,  
  2019년 PortSwigger 연구를 계기로 다시 조명되었고,  
  2020~2025년 사이에도 **주요 서버·CDN·프레임워크에서 반복적으로 새로운 변종과 CVE가 보고**되고 있다. :contentReference[oaicite:20]{index=20}  
- **가장 효과적인 방어**는:
  1. 엣지 프록시에서 **요청을 완충/정상화**해,
  2. 백엔드에는 **단 하나의 `Content-Length`만 가진 표준 H1 요청**만 전달하고,
  3. **CL/TE 모순·이중 CL은 즉시 400**,  
  4. **H2C 비활성 및 H2/H1 변환 정책 고정**,  
  5. **전 홉 동일 파서/스택 + 최신 패치 적용**이다.
- 여기에 더해:
  - **모니터링(TE 사용률, CL 중복률, 에러 스파이크)**
  - **스테이징에서의 스모크 테스트(모순 헤더 거절 검증)**
  - **캐시·세션·ACL과의 결합 방어**  
  를 일상적으로 운영에 녹여 넣으면,
- 세션 하이재킹·캐시 오염·ACL 우회처럼  
  **고위험 결과로 이어지는 Desync 리스크를 구조적으로 낮출 수 있다.**