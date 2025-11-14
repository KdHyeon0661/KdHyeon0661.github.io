---
layout: post
title: 웹해킹 - WebSocket CSRF
date: 2025-10-05 16:25:23 +0900
category: 웹해킹
---
# WebSocket CSRF (Cross-Site WebSocket Hijacking)

## 한눈에 보기 (Executive Summary)

- **문제**
  - WebSocket은 **HTTP Upgrade**로 시작하지만 **CORS에 의해 보호되지 않습니다**.
  - 브라우저는 교차 출처로 `new WebSocket("wss://victim.example/ws")`를 시도할 수 있으며,
    서버가 **핸드셰이크 시 `Origin`을 검증하지 않으면** 사용자의 **세션 쿠키**(조건 충족 시)가 포함된 요청으로 **연결**됩니다.
  - 연결이 성립되면, 공격자 페이지에서 **피해자 세션 권한으로** 메시지를 보내고 응답을 **읽을 수 있습니다.**

- **왜 쿠키가 붙나요?**
  - 핸드셰이크는 평범한 **HTTP GET**이므로 브라우저의 **쿠키/`SameSite` 정책**에 따릅니다.
  - 세션 쿠키가 `SameSite=None; Secure`(또는 특정 브라우저 정책에서 허용)로 설정되어 있으면 **교차 출처 WebSocket 핸드셰이크에도 쿠키가 전송**될 수 있습니다.

- **방어 핵심**
  1) **핸드셰이크에서 `Origin` 화이트리스트**: 허용 도메인만 연결. `null` Origin(파일/샌드박스) **거절**.
  2) **토큰 요구**: 임의 페이지가 열 수 없도록 **CSRF 토큰·1회성 토큰·JWT**를 **쿼리/경로/서브프로토콜**로 받아 **서버에서 검증**.
  3) **권한/세션 재검증 + 레이트**: 연결 직후 **인증 확인**(세션/토큰 매칭) & **오퍼레이션 단위 권한**.
  4) **프록시/LB에서 선차단**: Nginx/HAProxy 등에서 **`Origin` ACL**로 **Upgrade 거절**.
  5) **로그/모니터링**: 실패한 `Origin`, `null` Origin, 토큰 누락/오류 카운트로 탐지.

---

## 동작 원리와 위험

### WebSocket 핸드셰이크 요약

브라우저:
```
GET /ws HTTP/1.1
Host: victim.example
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Version: 13
Sec-WebSocket-Key: <랜덤>
Origin: https://attacker.tld
Cookie: session=...   ← (정책 충족 시)
```
서버(정상 허용 시):
```
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: <...>
```
> **주의**: HTTP CORS는 적용되지 않습니다. **서버가 `Origin`을 직접 검사**해야 합니다.

### 공격 시나리오(개념)

1) 사용자가 `victim.example`에 로그인(세션 쿠키 보유).
2) 공격자 페이지(`attacker.tld`)에서:
   ```js
   const ws = new WebSocket("wss://victim.example/ws");
   ws.onopen = () => ws.send(JSON.stringify({ op:"transfer", to:"attacker", amount:999 }));
   ws.onmessage = e => console.log("stolen:", e.data);
   ```
3) 서버가 **`Origin`을 검사하지 않으면** 연결 성립 → **세션 권한으로 API 실행**.

### SameSite가 방패가 될까?

- 기본 `SameSite=Lax`는 **대부분의 교차 사이트 요청에서 쿠키를 차단**하지만, 브라우저/버전/정책 예외와
  **의도적으로 `None`**을 쓴 서비스(SSO/임베드 필요)도 많습니다.
- **결론**: **쿠키 전송에 의존하여 안전을 기대하지 말고**, **서버단 Origin+토큰 검증**을 필수로 하십시오.

---

## “안전 재현” 점검(스테이징 전용) — **막혀야 정상**

> 아래 스모크 테스트는 **차단 확인** 목적입니다. 운영·외부 시스템 금지.

### 허용되지 않은 Origin

- 스테이징 도메인과 다른 페이지에서 콘솔 실행:
  ```js
  // https://not-allowed.tld 콘솔에서
  const ws = new WebSocket("wss://staging.example.com/ws");
  ws.onerror = e => console.log("blocked (expected)", e);
  ```
  **기대**: 서버가 403/핸드셰이크 거절 → 연결 실패.

### `Origin: null` 거절

- 로컬 파일(`file://`) 페이지에서:
  ```js
  const ws = new WebSocket("wss://staging.example.com/ws");
  ```
  **기대**: 거절(`null` Origin은 기본 금지).

### 토큰 누락/위조 거절

- 허용 Origin에서라도 토큰 없이:
  ```js
  const ws = new WebSocket("wss://staging.example.com/ws?csrf=missing");
  ```
  **기대**: 서버가 즉시 종료(440/close).

---

## 방어 전략(설계 → 구현 → 운영)

1) **인증/권한 모델**
   - 연결 직후 **세션/토큰을 재검증**하고, 각 메시지에 대해 **권한 확인**(특히 상태 변경).
2) **Origin 화이트리스트**
   - 허용 도메인(정확 호스트·프로토콜·포트)만 승인. `null`은 **거절**.
3) **토큰**
   - **한 번 발급한 1회성/짧은 TTL** 토큰을 **쿼리/경로/서브프로토콜**로 요구.
   - **더블 서브밋 쿠키** 패턴: `csrfCookie=...` + `?csrf=...` 일치 확인.
   - 또는 **JWT**(aud/iss 검증, 만료 짧게).
4) **프록시**
   - Nginx/HAProxy에서 `Origin` ACL로 **Upgrade 전**에 차단.
5) **레이트/동시연결 제한·IP 평판**
   - 사용자/토큰/테넌트별 연결 수·초당 메시지 수 제한.
6) **로그/모니터링**
   - `origin`, `allowed`, `token_ok`, `user_id`, `remote_ip` 캡처, **실패율 경보**.

---

## 프레임워크별 구현 레시피

### Node.js — `ws` 라이브러리

#### `Origin` 검사 + 토큰 검증

```js
// ws-server.js
import http from "node:http";
import { WebSocketServer } from "ws";
import { parse } from "node:url";
import crypto from "node:crypto";

const ALLOW_ORIGINS = new Set([
  "https://app.example.com",
  "https://admin.example.com",
]);

// 서버 발급(예: HTML 렌더링 시 메타/JS 변수로 내려줌)
function issueOneTimeToken(userId) {
  // 60초 TTL, userId와 묶어서 서버 메모리/캐시(또는 서명 JWT)로 관리
  const t = crypto.randomBytes(16).toString("hex");
  tokenStore.set(t, { userId, exp: Date.now()+60_000 });
  return t;
}
const tokenStore = new Map();

const server = http.createServer();
const wss = new WebSocketServer({ noServer: true });

server.on("upgrade", (req, socket, head) => {
  const origin = req.headers.origin;
  if (!origin || !ALLOW_ORIGINS.has(origin)) {
    socket.write("HTTP/1.1 403 Forbidden\r\n\r\n"); return socket.destroy();
  }
  // URL에서 토큰 추출
  const { query, pathname } = parse(req.url, true);
  if (pathname !== "/ws") { socket.write("HTTP/1.1 404 Not Found\r\n\r\n"); return socket.destroy(); }

  const token = query.csrf;
  const rec = token && tokenStore.get(String(token));
  if (!rec || rec.exp < Date.now()) {
    socket.write("HTTP/1.1 440 Invalid Token\r\n\r\n"); return socket.destroy();
  }
  tokenStore.delete(token); // 1회성

  // (선택) 쿠키 기반 세션도 동시 확인
  const userId = rec.userId; // 세션과 매칭 확인 절차 추가 권장

  wss.handleUpgrade(req, socket, head, (ws) => {
    ws.userId = userId;
    wss.emit("connection", ws, req);
  });
});

wss.on("connection", (ws) => {
  ws.on("message", (raw) => {
    let msg; try { msg = JSON.parse(raw); } catch { return; }
    // 권한/유효성 재검증
    if (msg.op === "transfer") {
      // ex) 금액/상대 체크, 권한 확인
      if (!Number.isFinite(msg.amount) || msg.amount <= 0) return ws.send(JSON.stringify({ err:"bad" }));
      // do transfer for ws.userId ...
      return ws.send(JSON.stringify({ ok:true }));
    }
  });
});

server.listen(8080);
```

> **포인트**
> - `server.on('upgrade')` 단계에서 **`Origin`**과 **토큰**을 확인해야 **핸드셰이크 차단**이 확실합니다.
> - 토큰은 **짧은 TTL** + **1회성**이 이상적.

#### `Sec-WebSocket-Protocol`(서브프로토콜)에 토큰 싣기

브라우저는 임의 헤더를 못 붙입니다. 대신 **서브프로토콜 문자열**을 사용할 수 있습니다.
```js
// 클라이언트
const token = window.__WS_TOKEN__;
const ws = new WebSocket("wss://app.example.com/ws", ["v1", `csrf.${token}`]);

// 서버 (ws)
server.on("upgrade", (req, socket, head) => {
  const protocols = (req.headers["sec-websocket-protocol"] || "").split(",").map(s=>s.trim());
  const csrfProto = protocols.find(p => p.startsWith("csrf."));
  const token = csrfProto?.slice(5);
  // token 검증 로직 ... (없으면 440)
});
```

---

### Socket.IO (엔진: WebSocket + 폴백)

```ts
import { createServer } from "http";
import { Server } from "socket.io";

const httpServer = createServer();
const ALLOW = new Set(["https://app.example.com"]);

const io = new Server(httpServer, {
  cors: {
    origin: (origin, cb) => {
      if (!origin) return cb(null, false);
      cb(null, ALLOW.has(origin));
    },
    credentials: true
  },
  allowRequest: (req, cb) => {
    // 핸드셰이크 헤더 직접 확인 가능
    const origin = req.headers.origin;
    if (!origin || !ALLOW.has(origin)) return cb("forbidden origin", false);

    const url = new URL(req.url!, "http://x");
    const token = url.searchParams.get("csrf");
    if (!token || !validateToken(token)) return cb("bad token", false);
    cb(null, true);
  }
});

io.on("connection", socket => {
  // 권한·레이트 제한 등
});
```
> Socket.IO의 `cors` 옵션만 믿지 말고, **`allowRequest`로 토큰·Origin을 이중 확인**하세요.

---

### Go — Gorilla WebSocket

```go
var allowed = map[string]bool{
  "https://app.example.com":   true,
  "https://admin.example.com": true,
}

var upgrader = websocket.Upgrader{
  CheckOrigin: func(r *http.Request) bool {
    o := r.Header.Get("Origin")
    return allowed[o]
  },
  // ReadBufferSize, WriteBufferSize 등…
}

func wsHandler(w http.ResponseWriter, r *http.Request) {
  // 토큰 검증
  token := r.URL.Query().Get("csrf")
  if !ValidateToken(token, r) {
    http.Error(w, "bad token", http.StatusUnauthorized)
    return
  }
  c, err := upgrader.Upgrade(w, r, nil)
  if err != nil { return }
  defer c.Close()

  for {
    var msg Message
    if err := c.ReadJSON(&msg); err != nil { break }
    // 권한 체크 후 처리 …
  }
}
```

---

### Python — Starlette / FastAPI

```python
from fastapi import FastAPI, WebSocket, WebSocketDisconnect
from starlette.requests import Request

app = FastAPI()
ALLOW = {"https://app.example.com","https://admin.example.com"}

@app.websocket("/ws")
async def ws_endpoint(ws: WebSocket):
    origin = ws.headers.get("origin")
    if origin not in ALLOW:
        await ws.close(code=4401)  # policy violation
        return

    token = ws.query_params.get("csrf")
    if not validate_token(token):  # TTL/서명 검증
        await ws.close(code=4401)
        return

    await ws.accept(subprotocol="v1")
    try:
        while True:
            data = await ws.receive_json()
            # 권한·검증 …
            await ws.send_json({"ok": True})
    except WebSocketDisconnect:
        pass
```

---

### Django Channels

```python
# asgi.py / routing.py 구성은 생략

from channels.generic.websocket import AsyncJsonWebsocketConsumer
from urllib.parse import urlparse, parse_qs

ALLOW = {"https://app.example.com","https://admin.example.com"}

class SecureConsumer(AsyncJsonWebsocketConsumer):
    async def connect(self):
        origin = self.scope["headers_dict"].get(b"origin", b"").decode()
        if origin not in ALLOW:
            return await self.close(code=4401)

        qs = parse_qs(self.scope["query_string"].decode())
        token = (qs.get("csrf") or [None])[0]
        if not validate_token(token):
            return await self.close(code=4401)

        await self.accept(subprotocol="v1")
```
> Channels에는 `AllowedHostsOriginValidator`/`CorsOriginValidator`가 있으나, **명시 화이트리스트 + 토큰 검증**을 **추가로** 구현하는 것이 안전합니다.

---

### Java Spring (STOMP/SockJS 포함)

```java
@Configuration
@EnableWebSocketMessageBroker
public class WsConfig implements WebSocketMessageBrokerConfigurer {
  @Override
  public void registerStompEndpoints(StompEndpointRegistry registry) {
    registry.addEndpoint("/ws")
      .setAllowedOrigins("https://app.example.com","https://admin.example.com") // Origin 화이트리스트
      .withSockJS();
  }
}
```

**토큰 검증(HandshakeInterceptor)**:
```java
@Component
public class CsrfHandshakeInterceptor implements HandshakeInterceptor {
  @Override
  public boolean beforeHandshake(ServerHttpRequest req, ServerHttpResponse res,
                                 WebSocketHandler wsHandler, Map<String, Object> attrs) {
    var headers = req.getHeaders();
    var origin = headers.getOrigin();
    if (origin == null || !(origin.equals("https://app.example.com") || origin.equals("https://admin.example.com")))
      return false;

    var uri = req.getURI();
    var params = UriComponentsBuilder.fromUri(uri).build().getQueryParams();
    var token = params.getFirst("csrf");
    return validateToken(token); // 1회성/TTL 검증
  }
  @Override public void afterHandshake(...) { }
}
```

---

## 리버스 프록시/LB에서 선차단

### Nginx — `Origin` ACL로 Upgrade 거절

```nginx
# 허용 오리진 매핑

map $http_origin $ws_ok {
  default 0;
  "~^https://app\.example\.com$"   1;
  "~^https://admin\.example\.com$" 1;
}

server {
  listen 443 ssl http2;
  server_name app.example.com;

  location /ws {
    # 비허용 Origin → 403 (핸드셰이크 전에 컷)
    if ($ws_ok = 0) { return 403; }

    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "Upgrade";
    proxy_http_version 1.1;
    proxy_pass http://app_ws;
  }
}
```
> **팁**: `Origin`이 비어 있거나 `null`이면 **거절**. 필요 시 `map`에 별도 분기.

### HAProxy — ACL

```haproxy
frontend fe
  bind :443 ssl crt /etc/haproxy/cert.pem
  acl is_ws hdr(Upgrade) -i WebSocket
  acl good_origin hdr(Origin) -i https://app.example.com https://admin.example.com
  http-request deny if is_ws !good_origin
  use_backend be_ws if is_ws
```

### Cloudflare Workers / CDN 엣지

- 핸드셰이크의 **`Origin` 헤더 검사** 후 `fetch(request)` 전에 **거절**.
- 엣지에서 차단하면 **앱까지 트래픽이 가지 않음**(코스트 절감/연결 폭주 완화).

---

## 토큰 설계 패턴(실무)

1) **더블 서브밋 쿠키**
   - `csrfCookie`(쿠키, HttpOnly 아님)와 `?csrf`(쿼리)를 비교.
   - WebSocket은 커스텀 헤더를 못쓰므로 **쿼리/서브프로토콜**로 전달.
2) **1회성·짧은 TTL 토큰**
   - 렌더/REST 호출로 잠깐 발급 → **Upgrade 직전에만 유효**.
   - **연결 성공 시 즉시 무효화**.
3) **JWT(Subprotocol)**
   - `Sec-WebSocket-Protocol: jwt.<base64>` 로 전달 → 서버에서 서명 검증.
   - `aud`(ws 엔드포인트), `exp`(매우 짧게), `sub`(사용자) 확인.
4) **권한/스코프**
   - 토큰에 **허용 오퍼레이션**을 명시해 서버에서 **메시지 레벨 권한**을 빠르게 결정.

---

## 메시지 레벨 방어(연결 후)

- **메시지 스키마 검증**: JSON Schema/유효성 체크(타입/범위/길이).
- **리플레이 차단**: 상태 변경 오퍼레이션은 **idempotency key** 필요.
- **레이트 제한**: 사용자·테넌트·연결당 **초당 N회**.
- **리소스 스코프**: `tenantId`/`userId` **컨텍스트에서 강제**(클라이언트 입력 신뢰 금지).
- **감사 로그**: `who did what`(userId, op, args hash).

---

## 로깅/탐지/알림

- **로그 필드**: `ts`, `remote_ip`, `origin`, `allowed`, `token_ok`, `user_id`, `path`, `closed_reason`
- **탐지 룰**
  - `origin not allowed` 급증 → 스캔 가능성
  - `null origin` 시도 다수
  - 토큰 검증 실패율 상승
  - IP/오리진별 비정상 연결 시도 > 임계

**예 — Loki(LogQL)**
```logql
{service="ws", event="handshake"} | json | allowed="false"
| count_over_time(5m) > 10
```

**예 — Splunk(SPL)**
```spl
index=app service=ws event=handshake allowed=false
| stats count by origin, remote_ip
| where count > 20
```

---

## 테스트/CI 자동화

- **유닛/통합**:
  - 비허용 `Origin` → 403/close
  - `null` Origin → 거절
  - 토큰 누락/만료 → 거절
  - 허용 Origin + 유효 토큰 → 성공
- **E2E(스테이징)**:
  - 다른 도메인 정적 페이지에서 연결 시도 → 실패해야 성공
  - 허용 도메인에서 연결 + 상태 변경 오퍼레이션 → 성공
- **정적 스캔**:
  - 앱 코드에 `Origin` 검증 누락 여부(핸드셰이크 경로 검색)
  - 프록시 설정에 `hdr(Origin)`/`$http_origin` ACL 존재 확인

---

## 체크리스트 (현장용)

- [ ] **핸드셰이크에서 `Origin` 화이트리스트**(정확 호스트·프로토콜·포트)
- [ ] `Origin: null` **거절**(필요 시 예외 경로만)
- [ ] **토큰 요구**(1회성·TTL 짧게) — 쿼리/경로/서브프로토콜로 전달
- [ ] 연결 직후 **세션/권한 재검증**, 메시지 단위 **권한 체크**
- [ ] **레이트 제한/동시연결 상한**(사용자·IP·테넌트)
- [ ] Nginx/HAProxy/CDN에서 **Upgrade 전 선차단**
- [ ] **로그/알림**: origin·token_ok·allowed·close_reason
- [ ] **테스트/CI**: 비허용 Origin/토큰 누락 **거절** 케이스 포함
- [ ] 세션 쿠키 **SameSite** 적절성 검토(가능하면 교차 전송 최소화)
- [ ] 문서화: 프런트는 **허용 도메인에서만** WebSocket을 열고 **토큰을 필수 첨부**

---

## 맺음말

**WebSocket은 CORS가 지켜주지 않습니다.**
핵심은 **핸드셰이크 단계에서 차단(Origin 화이트리스트 + 토큰)** 하고,
연결 후에도 **권한/레이트/스키마 검증**으로 2차 방어선을 구축하는 것입니다.
