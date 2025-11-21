---
layout: post
title: 웹해킹 - HTTP Request Smuggling, Desync
date: 2025-09-30 17:25:23 +0900
category: 웹해킹
---
# 🔀 HTTP Request Smuggling / Desync (H1/H2)

## 한눈에 보는 요약

- **문제(Desync)**: 클라이언트–프록시–백엔드가 **요청 경계(boundary)**를 서로 다르게 해석하면(예: `Content-Length` vs `Transfer-Encoding`), 프록시가 다음 사용자의 요청 일부를 **앞선 요청의 바디로 붙여 보내는** 등 **요청이 섞임**.
  - 결과: **세션 하이재킹**, **캐시 오염(Cache Poisoning)**, **ACL 우회**, **로그 교란**.
- **징후**:
  - 프록시 **경유**와 **직접(백엔드)** 접근 시 **응답 내용이 다름**
  - 간헐적 **400/408/411/413/502/504**
  - **“upstream prematurely closed connection”** 류의 로그
- **대표 원인**:
  1) **H1/H1**: `CL` vs `TE` 충돌(이중 지정, 표준 미준수)
  2) **H2/H1 변환**: HTTP/2 → HTTP/1.1 다운그레이드 시 **프레이밍 ↔ 헤더 기반** 경계 불일치
  3) **프록시 체인 이질성**: 서로 다른 파서/정책(예: 프론트는 `TE` 우선, 백엔드는 `CL` 우선)
- **핵심 방어**:
  - 프록시에서 **요청 완충(buffering) + 정상화(normalization) + 엄격 검증(strict)**
  - **`TE`(Transfer-Encoding) 차단/정규화**, **CL/TE 동시 존재 시 거절(400)**
  - **H2C(HTTP/2 cleartext) 비활성**, **H2→H1 변환 규칙 고정**
  - **모든 홉**에서 **동일한 파서·정책** 사용(가능하면 **한 제품군**으로 통일)
  - **WAF/IDS**에 **이중 헤더/모순 헤더** 탐지 룰 적용

---

# 동작 원리와 분류

## 왜 “경계”가 어긋나나?

HTTP/1.1은 요청 본문 경계를 **`Content-Length`**(고정 길이) 또는 **`Transfer-Encoding: chunked`**(청크 길이들의 합)로 구분합니다.
규격상 **두 개를 동시에 둘 수 없으며**, 있어도 **해석 일관**이 필요합니다.
그러나 현실에선 프록시 A·백엔드 B가 **다른 우선순위/버그**를 가지면 **서로 다른 경계**로 해석되어 **요청이 한쪽에서 남거나 넘치는 현상**이 생깁니다.

## 유형(콘셉트)

> 아래는 **개념적 설명**입니다. 실제 공격 페이로드는 **생략/단순화**합니다.

- **CL.TE**: 프론트 프록시는 `Content-Length`를 따르고, 백엔드는 `Transfer-Encoding: chunked`를 따름 → **앞쪽 파서가 남긴 바디 조각**이 **뒷쪽 파서에게 다음 요청의 일부**로 해석.
- **TE.CL**: 반대 상황.
- **CL.CL(이중 CL)**: 같은 헤더가 여러 번 있을 때 합치기/첫 값/마지막 값 등 정책 차이로 경계 어긋남.
- **H2/H1 Desync**: HTTP/2는 **바디 길이를 헤더로 표기하지 않고 프레임으로 구분**. 프락시가 H2→H1로 변환하면서 길이/전송인코딩을 **재구성**하는데, 그 로직이 백엔드의 해석과 달라지면 Desync.

---

# **징후**와 **관측 포인트**

- **프록시 경유 vs 백엔드 직결** 응답 차이(콘텐츠/헤더/캐시 키)
- 간헐적인 **4xx/5xx**(특히 400/408/411/413/502/504)
- **캐시 적중률 이상치**(특정 경로가 비정상 오염)
- **로그 상관 불일치**: 프록시 접근로그의 `Content-Length`와 백엔드가 읽은 바디 길이가 다름
- **보안 제품 경보**: 이중 헤더/모순 헤더/금지 헤더 탐지

**Nginx 에러로그 예(관념적)**
```
client prematurely closed connection while reading client request body
upstream sent too big header while reading response header from upstream
upstream prematurely closed connection
```

---

# 테스트

> 다음은 **자신의 연구/스테이징** 환경에서, **차단이 올바르게 작동**하는지 확인하기 위한 **방어 검증** 테스트입니다.
> 실제 시스템에 대한 공격으로 사용하지 마십시오.

## 구성 예시(도커-스케치)

- **edge**: 리버스 프록시(Nginx/Envoy/HAProxy)
- **app**: 간단한 애플리케이션(Node/Flask)

```
[Client] → [edge(proxy)] → [app(server)]
```

## 방어 검증 테스트 아이디어

- **이중 헤더**(예: `Content-Length`가 두 번) → **400**이어야 정상
- **`CL` + `TE` 동시 존재** → **400**이어야 정상
- **H2 → H1 변환 경로**에서도 동일하게 **거절/정규화**되는지 확인
- **버퍼링** ON 상태에서 **edge가 백엔드로 단일 `Content-Length`만 전달**하는지 확인

### Node.js로 **부적합 요청을 보냈을 때 400 기대** 테스트(안전)

```js
// test/desync-guard.test.js  (승인된 사내 스테이징용)
// 목적: 엣지가 모순 헤더를 가진 요청을 반드시 400으로 거절하는지 확인
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
  // 기대: "HTTP/1.1 400 Bad Request" or 프록시 벤더의 에러 응답
})();
```

> 포인트: **명백히 잘못된(모순) 요청**을 보냈을 때 **프록시가 차단**하는지만 검증합니다.
> 실제 취약 조합 페이로드는 제공하지 않습니다.

---

# 프록시·게이트웨이 **방어 설정** (벤더 중립 원칙 + 대표 예시)

> 현실적으로 “**프론트에서 단일 표준 형태**로 **정규화**해 백엔드에 전달”하는 것이 최선입니다.

## 공통 원칙(요약)

1. **요청 완충(Buffering)**: 프록시가 **요청 전체를 읽고 유효성 검증** 후 백엔드로 **단일 규격**(`Content-Length` 1개)으로 전달.
2. **헤더 정규화/차단**:
   - `Transfer-Encoding`은 **허용하지 않거나**(`chunked` 필요 없음) **강제 제거**.
   - `CL`과 `TE`가 **동시에 오면 거절(400)**.
   - 중복 `Content-Length` → **거절**.
3. **H2/H1 변환 정책 고정**: H2 수신 시 프록시가 **CL을 스스로 계산**하여 **하나만** 백엔드로 전달.
4. **H2C 비활성**: TLS 없는 HTTP/2(`h2c`)는 비표준 게이트웨이에서 문제를 키움.
5. **한 제품/한 파서 통일**: 프론트·미들·백 프록시가 동일한 HTTP 스택/버전을 사용하도록 통일.
6. **요청 크기/타임아웃 제한**: `client_max_body_size`, `request_timeout` 등.

---

## 예시

```nginx
# ① 요청 버퍼링으로 백엔드에 단일 Content-Length만 전달

proxy_request_buffering on;     # 기본 on (명시)
proxy_http_version 1.1;         # 업스트림과 H1 고정(환경에 맞게)

# — map/if 조합으로 방어

map $http_transfer_encoding $te_bad {
  default         1;
  "~*^chunked$"   0;   # 필요 시 허용하지만, 보통 proxy_request_buffering on이면 필요 없음
}
server {
  listen 443 ssl http2;     # 클라이언트는 H2 수용하되…
  http2_max_field_size 16k; # 엄격히
  http2_max_header_size 32k;

  # H2C(클리어텍스트) 비활성: 별도 80 h2c 업그레이드 경로 두지 말 것

  location / {
    if ($te_bad) { return 400; }     # TE가 chunked 외 값이 오면 거절
    proxy_set_header Transfer-Encoding "";  # 업스트림으로 TE 제거
    proxy_set_header Content-Length $content_length;  # nginx가 계산한 값만 전달

    proxy_read_timeout 30s;
    client_body_timeout 30s;
    client_max_body_size 10m;
    # 캐시/변조 방지 추가 헤더 등은 별도로
    proxy_pass http://app_backend;
  }
}
```

> **해설**
> - `proxy_request_buffering on`: Nginx가 클라이언트 바디를 **전부 읽고** 백엔드로 보냄 → **TE 사용 불필요**.
> - 업스트림엔 **하나의 `Content-Length`만** 전달.
> - `Transfer-Encoding`을 제거해 **CL/TE 다중 해석 여지**를 차단.

---

## HAProxy 예시(개념적)

```haproxy
global
  # 최신 릴리스 사용(HTX 엔진 기본) → 요청을 단일 표준 형식으로 내부 표현
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
  # H2C 비활성(명시적 업그레이드 경로 만들지 마세요)
  http-response set-header Strict-Transport-Security max-age=31536000

  # 모순 헤더 거절
  http-request deny if { req.hdr_cnt(content-length) gt 1 }
  http-request deny if { req.hdr(transfer-encoding) -m found }
  # 또는 allowlist: TE가 존재하면 즉시 거절

  default_backend be_app

backend be_app
  server s1 app:8080 check
```

> **포인트**: **요청 버퍼링 + TE 거절 + 이중 CL 거절**이라는 세 가지 축.

---

## Envoy 예시(개념적 YAML)

```yaml
static_resources:
  listeners:
  - name: https
    address: { socket_address: { address: 0.0.0.0, port_value: 443 } }
    filter_chains:
    - filters:
      - name: envoy.filters.network.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          codec_type: AUTO
          stat_prefix: ingress_http
          stream_error_on_invalid_http_message: true   # 잘못된 메시지 스트림 에러
          request_headers_timeout: 0.5s
          http2_protocol_options:
            allow_connect: false
          http_protocol_options:
            # Envoy는 내부적으로 정규화/검증 수행. 아래는 개념적 스위치들:
            header_validation: true
            # allow_chunked_length: false  (베ンダ 버전에 따라 지원여부 상이)
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
    connect_timeout: 2s
    type: logical_dns
    lb_policy: round_robin
    load_assignment:
      cluster_name: app
      endpoints:
      - lb_endpoints:
        - endpoint: { address: { socket_address: { address: app, port_value: 8080 } } }
```

> **주의**: 실제 키 이름/지원 여부는 Envoy 버전에 따라 다릅니다. 목적은 **유효하지 않은 HTTP 메시지 거절** + **정규화**입니다.

---

## 포인트

```apache
# 엄격 모드

HttpProtocolOptions Strict

# Proxy 경유 시

ProxyRequests Off
RequestReadTimeout header=20-40,MinRate=500 body=20,MinRate=500

# 과도한/중복 헤더 거절(모듈·버전 의존)

LimitRequestFieldSize 16384
LimitRequestFields 100

# TE 헤더 제거

RequestHeader unset Transfer-Encoding early
```

---

# 애플리케이션 레이어 **완화책**(보조)

> 핵심 문제는 **HTTP 레이어**의 경계 불일치지만, 앱에서도 “이상 상황을 탐지/거절”하여 **2차 안전망**을 만들 수 있습니다.

## Node/Express — 모순 헤더 선거절

```js
app.use((req, res, next) => {
  const te = req.headers['transfer-encoding'];
  const cls = req.rawHeaders.filter((h, i) => i % 2 === 0 && h.toLowerCase() === 'content-length').length;

  // 1) TE가 오면(프록시 정책상 금지) 차단
  if (te) return res.status(400).send("Bad Request");
  // 2) Content-Length가 중복이면 차단
  if (cls > 1) return res.status(400).send("Bad Request");
  next();
});
```

## Flask — 무효 요청 거절

```python
@app.before_request
def reject_conflicts():
    cl_headers = [v for (k,v) in request.headers if k.lower() == 'content-length']
    if 'transfer-encoding' in (k.lower() for (k,_) in request.headers):
        abort(400)
    if len(cl_headers) > 1:
        abort(400)
```

> **주의**: 앱에서 막는 것은 **보조책**입니다. 근본 해결은 **엣지/프록시**에서 해야 합니다.

---

# **HTTP/2 관련** 주의점

- **HTTP/2는 바이너리 프레이밍**으로 경계 모호성이 작습니다.
  하지만 **게이트웨이가 H2→H1으로 변환**할 때 **길이/TE/CL 재구성**을 수행합니다.
- **안전 가이드**
  1) **H2C 비활성**: 평문 H2는 중간자/비일관 파서 위험 증가.
  2) **프록시가 H2 요청을 완충** 후, 백엔드에는 **H1 + 단일 `Content-Length`**로 고정 전달.
  3) **혼합 스택 금지**: 일부 L7 장비가 자체 파서 버그를 가질 수 있으므로 **일관 스택** 유지, 최신 패치.
  4) **H2 전용 업스트림**(프록시–백엔드 모두 H2) 구성이 아니라면, 변환 규칙을 **문서화/테스트**.

---

# **캐시 오염**(Cache Poisoning)과 결합 방지

- 프록시/캐시(LB, CDN, Varnish 등)는 **요청 라인·헤더**로 **캐시 키**를 만듭니다.
- Desync로 **다음 사용자의 요청 일부가 앞 요청에 붙으면**, **엣지 캐시가 오염**될 수 있습니다.

**방어 포인트**
- 캐시 키에 **불필요한 헤더 제외/정상화**(대소문자/중복/공백)
- **변이/금지 헤더**(예: `Transfer-Encoding`, 이중 `Content-Length`) 탐지 시 **캐싱 금지**
- **Vary 헤더 관리**: 사용자별/세션별 리소스는 **절대 캐시 금지** (`Cache-Control: private, no-store`)

---

# **모니터링/탐지** 룰 예시

- **로그 파이프라인에서**
  - `Transfer-Encoding` 존재율(HTTP/1.1에서 클라이언트→프록시 경로 기준): **0%에 가깝게**
  - `Content-Length` **중복** 발생 건수: 0이어야 정상
  - **프록시 vs 앱**의 바디 길이 불일치(가능하면 지표화)
  - **에러율 히스토그램**: 경로별 400/502가 **가끔 급증**하는지

**Loki(LogQL) 예시**
```logql
{service="edge", msg="http_access"} |= "Transfer-Encoding"
|~ "chunked|deflate|gzip"  # 허용되지 않는 값 확인
| count_over_time(1m)
```

**Splunk(SPL) 예시**
```spl
index=edge msg=http_access ("Transfer-Encoding"=* OR content_length_count>1)
| stats count by src_ip, path, user_agent
```

---

# **운영 체크리스트** (배포 전/후)

- [ ] **프록시**: 요청 **버퍼링 on**, 백엔드로 **단일 `Content-Length`**만 전달
- [ ] **TE 거절/제거**, **CL 중복 거절**
- [ ] **HTTP/2**: H2C 비활성, 변환 정책 문서화, 최신 패치
- [ ] **통일된 파서**: 엣지–미들–백엔드 제품군/버전 일치
- [ ] **요청 제한**: 헤더 크기/개수, 바디 크기, 타임아웃
- [ ] **캐시**: 금지/변이 헤더 캐싱 금지, 민감 경로 `no-store`
- [ ] **테스트**: (승인된 랩) 이중 CL·CL+TE에 **400** 응답 확인, H2→H1 경로 동일성 확인
- [ ] **모니터링**: TE 사용률, CL 중복률, 경로별 4xx/5xx 이상치 알림
- [ ] **런북**: 의심 시 **임시 완화**(TE 전면 차단, 특정 경로 read-only), 커뮤니케이션 플랜

---

# **사고 대응 런북(요약)** — “Desync 의심” 시

1. **식별**: 프록시 경유/직결 응답 차이, 로그상 모순 헤더, 에러율 급증
2. **격리**:
   - 엣지에서 **`Transfer-Encoding` 즉시 차단**
   - **요청 버퍼링 강제**
   - 문제 경로 **캐시 무효화/비활성**
3. **근절**: 프록시/게이트웨이/앱 레이어 패치, **통일 파서** 적용
4. **복구**: 부하/지연 모니터링, 캐시 재가온, 테스트 케이스 상시화
5. **교훈화**: CI 파이프라인에 **프로토콜 유효성 테스트** 추가, 벤더 릴리스 노트 트래킹

---

# 부록 — “프로토콜 유효성” **자동화 테스트** 아이디어

> 목적: **배포 전** 프록시가 **모순 헤더를 거부**하는지, **H2/H1 변환**에서도 동일 동작인지 “스모크”로 확인.

- **케이스 A**: `Content-Length`가 **중복** → 400
- **케이스 B**: `CL` + `TE` **동시 존재** → 400
- **케이스 C**: **H2 클라이언트**(예: `curl --http2`)로 전송했을 때도 백엔드에는 단일 `Content-Length`만 도착
- **케이스 D**: 캐시 전면에서 동일 요청 반복 시 응답 바디/헤더 **완전 일치**

**간단 Node 스크립트(케이스 A/B)**
```js
// node validate-proxy.js (사내 CI에서만 사용)
import tls from "node:tls";

function raw(host, port, payload) {
  return new Promise((resolve, reject) => {
    const s = tls.connect({ host, port, rejectUnauthorized: false }, () => s.write(payload));
    let buf = "";
    s.on("data", d => buf += d.toString());
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

for (const [name, req] of [["A-dupCL", reqA], ["B-CL+TE", reqB]]) {
  const res = await raw("edge.example.internal", 443, req);
  if (!/^HTTP\/1\.[01] 400/.test(res)) {
    throw new Error(`❌ ${name}: expected 400, got\n${res.slice(0,200)}`);
  }
  console.log(`✅ ${name}: 400`);
}
```

> 주의: **유효성 실패가 곧 안전함**을 의미하지는 않습니다. 반대로, **이 테스트를 통과**해도 H2/H1 경계·캐시·특정 벤더 버그 등은 별도 점검이 필요합니다.

---

## 정리

- **Desync**는 “**요청 경계 해석의 불일치**”가 만든 사고입니다.
- **가장 효과적인 방어**는 엣지 프록시에서 **요청을 완충/정상화**하여 백엔드에 **단 하나의 표준 형태**로 전달하는 것입니다.
- **CL/TE 모순·중복 헤더는 즉시 400**으로 거절하고, **H2C 비활성** 및 **전 홉 동일 파서** 원칙을 지키세요.
- 배포 전/후 **스모크 테스트**와 **로그 기반 탐지**를 일상화하면, **세션 하이재킹·캐시 오염**으로 이어질 리스크를 크게 줄일 수 있습니다.
