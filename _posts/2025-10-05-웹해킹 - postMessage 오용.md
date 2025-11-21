---
layout: post
title: 웹해킹 - postMessage 오용
date: 2025-10-05 17:25:23 +0900
category: 웹해킹
---
# `postMessage` 오용 — 원리, 위험 시나리오, “안전 재현” 점검, 실전 방어 레시피(Origin/Source/스키마/토큰)

> ⚠️ **법·윤리 고지**
> 본 문서는 **자신/조직 소유**의 웹 애플리케이션에서 **방어·점검**을 위한 자료입니다.
> 목적은 **오구성의 원인을 파악하고 안전하게 수정하는 것**이며, 실제 공격 절차를 조장하지 않습니다.

---

## 한눈에 보기 (Executive Summary)

- **문제**
  - `window.postMessage` 수신 시 **`event.origin` 검증**이 없거나 느슨(`*` 등)하면,
    임의 도메인의 문서가 **민감 데이터 요청/행위 유발** 메시지를 보내고 **응답을 탈취**할 수 있습니다.
- **왜 발생하나?**
  - 브라우저는 **출처가 다른 창/iframe 간에도** `postMessage` 자체는 전송하게 해줍니다.
  - **수신측**이 `event.origin`(보낸 쪽 출처)과 `event.source`(보낸 창 핸들) 및 **메시지 스키마**를 확인해야 안전합니다.
- **핵심 방어**
  1) **Origin 화이트리스트**(정확히 `scheme://host[:port]` 매칭, `null` 금지)
  2) **Source 고정**(예상된 iframe/window 객체만 수락)
  3) **메시지 스키마 검증**(타입·버전·필수 필드, 값 범위)
  4) **토큰/논스/요청-ID**(채널 바인딩·재생 방지)
  5) **보안 헤더·프레임 정책**(CSP `frame-ancestors`, `X-Frame-Options`, sandbox)
  6) **로그/모니터링/테스트**(허용되지 않은 origin 거절 여부 자동 점검)

---

## `postMessage` 보안 모델 요약

- **발신**: `targetWindow.postMessage(message, targetOrigin, [transfer])`
  - `targetOrigin`에 **정확한 오리진**을 넣으면 브라우저가 **그 오리진이 아닐 경우 배송 자체를 차단**합니다.
  - `*`은 “어디든 보내라”는 뜻 → 수신측 검증 실패 시 위험 증가.
- **수신**: `window.addEventListener('message', handler)`
  - `handler(event)`에서 반드시:
    - `event.origin`이 **허용된 오리진인지**
    - `event.source`가 **예상한 창/iframe인지**
    - `event.data`가 **정상 스키마인지**
  - 를 검증 후 처리해야 합니다.

---

## 위험 시나리오

### Origin 검증 누락

```js
// ❌ 나쁜 예: 아무 오리진 메시지도 처리
window.addEventListener("message", (event) => {
  if (event.data?.type === "GET_PROFILE") {
    // 민감 데이터 응답
    event.source.postMessage({ type: "PROFILE", data: currentUserProfile }, "*");
  }
});
```
- 누구든 iframe/팝업으로 이 페이지를 불러 메시지를 보내면 **사용자 데이터가 외부로 전송**될 수 있습니다.

### 느슨한 검증(하위 문자열/와일드카드 오류)

```js
// ❌ 나쁜 예: includes로 도메인 검증 → evil.com.example.com 우회 가능
const ALLOW = "https://example.com";
if (event.origin.includes("example.com")) { /* … */ }
```
- **정확 매칭**(프로토콜·호스트·포트)로 해야 합니다.

### `event.source` 미검증

- 동일 오리진 A/B 두 창 중 **어느 창**이 보냈는지 모르면 **혼동(Confused Deputy)** 위험.
- 특정 iframe과만 통신해야 하는데 다른 창에서 메시지 주입 가능.

### `Origin: null` 허용

- `file://` 페이지, sandbox iframe, 일부 브라우저 확장 등에서 **`event.origin === "null"`**이 됩니다.
- 일반적으로 **거절**하세요(특정 내부 도구에서만 예외적으로 허용).

### 스키마/액션 검증 부족

- 문자열 커맨드 실행, `eval`/DOM 삽입 등은 **RCE/클릭재킹**으로 번질 수 있음.
- **액션 enum + 페이로드 스키마**로 제한하세요.

---

## — **막혀야 정상 ✅**

> 목적: 허용되지 않은 오리진/소스에서 온 메시지가 **무시**되는지, 스키마가 잘 **검증**되는지 확인

1) **비허용 오리진**에서 iframe으로 페이지를 삽입하고 `postMessage` 송신 → **응답 없어야 정상**
2) **`null` 오리진**(sandbox iframe, `file://`)에서 송신 → **무시**되어야 정상
3) **스키마 위반 메시지**(필수 필드 누락, 타입 불일치) → **무시/에러 로그**
4) **요청-ID 불일치** 응답 수신 → **드롭**

아래 §7의 테스트 하니스 예제를 활용하세요.

---

## 안전한 설계 패턴 (정리)

- **발신 측**
  - `postMessage(payload, exactOrigin)` — 절대 `*` 사용 금지(디버깅 외).
  - 최초 연결 시 **핸드셰이크**: 상호 오리진/버전/nonce 교환.
- **수신 측**
  - `origin` **정확 매칭**(예: `new URL(origin).origin`과 대조, 포트 포함).
  - `source === expectedWindow` 확인(초기 핸드셰이크 때 고정).
  - `data`는 **JSON 스키마**로 검증(타입/범위/필수 필드).
  - `nonce`/`token`/`requestId`로 **재생·혼동 방지**.
  - **권한**: 민감 행위는 **명시적 사용자 제스처** 또는 **추가 토큰** 요구.
- **플랫폼 보강**
  - CSP: `frame-ancestors`로 **누가 이 페이지를 iframe 할 수 있는지 제한**.
  - `X-Frame-Options: DENY|SAMEORIGIN` (레거시)
  - `sandbox` 속성 사용 시 `allow-scripts allow-same-origin` 등 **최소 권한**.

---

## 견고한 수신 코드 (Vanilla JS + Zod 스키마 예)

```html
<!-- host.html (수신자, 예: https://app.example.com/host.html) -->
<!doctype html>
<meta charset="utf-8" />
<title>Secure postMessage Host</title>
<iframe id="partner" src="https://widget.partner.com/embed.html" referrerpolicy="origin"></iframe>
<script type="module">
// npm 없이 간단 스키마 체크(예: zod CDN) — 실제 운영에선 번들로 포함
// <script src="https://cdn.skypack.dev/zod"></script> 같은 방식 가능
import { z } from "https://cdn.skypack.dev/zod@3.23.8";

// 1) 화이트리스트 오리진(정확 문자열)
const ALLOW_ORIGINS = new Set([
  "https://widget.partner.com",
  "https://widget.partner-cdn.com:8443",
]);

// 2) 스키마 정의(버전·액션·페이로드)
const BaseMsg = z.object({
  v: z.literal(1),                 // 프로토콜 버전
  id: z.string().min(8).max(64),   // 요청/응답 상호연결용
  type: z.enum(["PING","GET_PROFILE","SET_THEME"]),
  nonce: z.string().length(16),    // 재생 방지 논스
  data: z.any(),                   // 각 type에 따라 별도 체크
});

// 2-1) type별 상세 스키마
const MsgGetProfile = BaseMsg.extend({
  type: z.literal("GET_PROFILE"),
  data: z.object({ fields: z.array(z.enum(["name","email","avatar"])).max(3) })
});
const MsgPing = BaseMsg.extend({ type: z.literal("PING"), data: z.object({}) });
const MsgSetTheme = BaseMsg.extend({
  type: z.literal("SET_THEME"),
  data: z.object({ theme: z.enum(["light","dark"]) })
});

// 3) partner iframe 핸들 고정(핸드셰이크 후 업데이트 가능)
const iframe = document.getElementById("partner");
let partnerWin = iframe.contentWindow;
let partnerOrigin = new URL(iframe.src).origin; // 예상 오리진

// 4) 안전한 수신 핸들러
window.addEventListener("message", (event) => {
  // 4-1) Origin 검증
  if (!ALLOW_ORIGINS.has(event.origin)) return;
  // 4-2) Source 검증
  if (event.source !== partnerWin) return;

  // 4-3) 데이터 검증(최소 구조 확인)
  const parsed = BaseMsg.safeParse(event.data);
  if (!parsed.success) return;

  // 4-4) type별 처리
  const msg = parsed.data;
  switch (msg.type) {
    case "PING": {
      event.source.postMessage({ v:1, id:msg.id, type:"PONG", nonce:msg.nonce, data:{} }, event.origin);
      break;
    }
    case "GET_PROFILE": {
      const p2 = MsgGetProfile.safeParse(msg);
      if (!p2.success) return;
      // 민감 데이터 — 최소화/마스킹
      const out = { name: user.name, email: mask(user.email), avatar: user.avatarUrl };
      event.source.postMessage({ v:1, id:msg.id, type:"PROFILE", nonce:msg.nonce, data:out }, event.origin);
      break;
    }
    case "SET_THEME": {
      const p3 = MsgSetTheme.safeParse(msg);
      if (!p3.success) return;
      if (!document.hasFocus()) return; // 예: 사용자 제스처 조건 등
      setTheme(p3.data.theme);
      event.source.postMessage({ v:1, id:msg.id, type:"OK", nonce:msg.nonce, data:{} }, event.origin);
      break;
    }
  }
});

function mask(e){ const [u, d] = e.split("@"); return u[0]+"***@"+d; }
function setTheme(t){ document.documentElement.dataset.theme = t; }
</script>
```

> 포인트
> - **Origin/Source/Schema** 삼중 검증이 핵심.
> - 응답 `postMessage`에서 **`event.origin`을 그대로 targetOrigin으로 사용**(정확 매칭).
> - 민감 데이터는 **최소화/마스킹** 후 전달.

---

## 안전한 발신 코드 (위젯/임베디드 측)

```html
<!-- https://widget.partner.com/embed.html (발신자) -->
<!doctype html>
<script type="module">
const parentOrigin = "https://app.example.com"; // 계약된 정확 오리진
const id = () => Math.random().toString(36).slice(2, 10);
const nonce = () => crypto.getRandomValues(new Uint8Array(8)).reduce((s,b)=>s+("0"+b.toString(16)).slice(-2), "");

function requestProfile() {
  const msg = { v:1, id:id(), type:"GET_PROFILE", nonce:nonce(), data:{ fields:["name","email"] } };
  window.parent.postMessage(msg, parentOrigin); // 절대 '*'
}

// 응답 수신 — 부모에서만 오는지 확인
window.addEventListener("message", (event)=>{
  if (event.origin !== parentOrigin) return;
  if (event.source !== window.parent) return;
  const res = event.data;
  if (res?.v !== 1 || res?.id == null) return;
  // id·nonce 매칭 확인 등…
  console.log("PROFILE:", res.data);
});

requestProfile();
</script>
```

---

## — Express 2개 포트

> **목적**: 서로 다른 오리진(예: `http://localhost:3000` vs `http://localhost:4000`)에서
> 비허용 오리진 메시지가 **무시**되는지, 허용 오리진만 **응답**하는지 눈으로 확인

```js
// server-3000.js (Host)
import express from "express";
const app = express();
app.use(express.static("./host")); // host.html 등
app.listen(3000, ()=>console.log("Host http://localhost:3000"));
```

```js
// server-4000.js (Widget)
import express from "express";
const app = express();
app.use(express.static("./widget")); // embed.html 등
app.listen(4000, ()=>console.log("Widget http://localhost:4000"));
```

- `host/host.html`의 화이트리스트에 **`http://localhost:4000`만** 넣습니다.
- 다른 임의 페이지(예: `http://localhost:5000`)에서 보내는 메시지는 **무시**되어야 합니다.

---

## React/Hooks로 안전 구현(수신자 관점)

```tsx
// useSecurePostMessage.tsx
import { useEffect, useRef } from "react";

type Allowed = `https://${string}` | `http://${string}:${number}`;
const ALLOW = new Set<Allowed>(["http://localhost:4000" as Allowed]);

export function useSecurePostMessage(iframeRef: React.RefObject<HTMLIFrameElement>) {
  const partner = useRef<Window|null>(null);

  useEffect(() => {
    partner.current = iframeRef.current?.contentWindow ?? null;
    const onMsg = (event: MessageEvent) => {
      if (!ALLOW.has(event.origin as Allowed)) return;
      if (event.source !== partner.current) return;
      // 스키마·액션 검증 후 처리…
    };
    window.addEventListener("message", onMsg);
    return () => window.removeEventListener("message", onMsg);
  }, [iframeRef]);
}
```

---

## 고급 토픽

### 핸드셰이크 & 채널 바인딩

- 최초 통신 시:
  1) **부모 → iframe**: `HELLO {nonceA}`
  2) **iframe → 부모**: `HELLO_ACK {nonceA, nonceB}`
  3) **부모 → iframe**: `BIND {nonceB, token}`
- 이후 메시지는 `token` 포함(또는 HMAC) → **다른 창 오염 방지**.

### `MessageChannel` / `BroadcastChannel`

- **MessageChannel**: `port1/port2`로 전용 채널 사용 → **수신자 고정**에 유리.
- **BroadcastChannel**: 같은 출처 내 브로드캐스트. **교차 출처 간 사용 금지**(채널 명 추측 리스크, 분리돼도 설계상 불필요).

### 팝업/`window.opener`와 탭내비깅

- 새 창을 열 때 **`rel="noopener noreferrer"`**로 `window.opener` 제거.
- 팝업과의 통신에도 **origin/source 검증** 동일하게 적용.

### Electron·하이브리드 앱

- `contextIsolation: true`, `nodeIntegration: false`
- `postMessage`는 렌더러 간 안전 채널로만.
- 외부 콘텐츠 임베드 시 **엄격한 오리진** + **preload에서 스키마 검사**.

---

## 보안 헤더/프레임 정책과의 연계

- **CSP**
  - `frame-ancestors 'self' https://trusted-embedder.com`
    → **누가 이 페이지를 iframe 할 수 있는지**를 제한(수신 측 보호).
  - `frame-src https://widget.partner.com`
    → **우리가 임베드할 수 있는 출처 제한**(발신 측 보호).
- **X-Frame-Options**
  - `SAMEORIGIN` 또는 `DENY` (레거시 브라우저 대응)
- **Referrer-Policy**
  - `strict-origin-when-cross-origin`(최소화) — 토큰/식별자 referrer 유출 방지
- **Cross-Origin-Opener-Policy / Embedder-Policy**
  - 필요 시 교차 출처 격리 강화(고급 시나리오).

---

## 로깅·모니터링

- **로그 필드**: `ts`, `page`, `allowed(boolean)`, `origin`, `sourceMatched`, `type`, `validSchema`, `nonceOk`, `userId`
- **탐지 룰 예시**
  - `allowed=false` 건수 급증 → 임베드 스캔 의심
  - `origin="null"` 다수 → sandbox/file 컨텍스트 남용
  - 스키마 실패율↑ → 클라이언트 SDK 버전 불일치/오용

---

## 테스트/CI 자동화

### 프론트 단위 테스트(예: Jest + JSDOM)

- **Origin/Source** 체크 로직을 모킹해 **허용/거절 케이스** 검증.

### E2E(Playwright/Cypress)

- **비허용 오리진 페이지**에서 iframe 로드 → 메시지 전송 → **응답 없음** 확인
- **허용 오리진**에서 올바른 스키마로 요청 → **정상 응답**
- **스키마 위반/nonce 불일치** → **드롭**

```ts
// Playwright 예시 개념
test("disallowed origin is ignored", async ({ page, context }) => {
  await page.goto("https://not-allowed.tld/hacker.html");
  // hacker.html은 /host.html을 iframe으로 로드하고 postMessage 보냄
  const res = await page.evaluate(() => window.__gotReply); // 테스트용 플래그
  expect(res).toBeFalsy(); // 응답이 없어야 함
});
```

---

## 안티패턴/주의사항 요약

- `postMessage(..., "*")`로 **남발 전송** 금지
- 수신 시 **`event.origin`을 `includes`/정규식 느슨 검증** 금지 → **정확 문자열 매칭**
- **`event.source` 미검증** 금지(특히 여러 iframe/팝업이 얽힌 앱)
- **스키마 없는 자유 문자열 커맨드** 금지 → **enum 기반 액션 + JSON 스키마**
- **민감 행위 자동 실행** 금지 → 사용자 제스처/추가 토큰 요구
- **`Origin: null` 허용** 금지(정말 필요한 내부 도구만 별도 경로에서)
- **응답에 민감 데이터 과다 포함** 금지(최소화·마스킹·TTL)

---

## 체크리스트 (현장용)

- [ ] 발신: `targetOrigin`에 **정확 오리진** 지정(`*` 금지)
- [ ] 수신: `event.origin` **화이트리스트 정확 매칭**, `null` 거절
- [ ] 수신: `event.source === expectedWindow` 확인
- [ ] 메시지: **버전/액션 enum/요청-ID/nonce** 포함, **JSON 스키마 검증**
- [ ] 민감 행위: **추가 토큰/사용자 제스처 요구**
- [ ] CSP: `frame-ancestors`/`frame-src`로 **임베드 제어**
- [ ] 로깅: origin/source/스키마/논스 검증 결과
- [ ] 테스트: 비허용 오리진·스키마 위반·nonce 불일치 **거절** 케이스 자동화
- [ ] 문서화: **파트너 오리진 목록/버전/액션 스펙** 관리(레지스트리)

---

## 맺음말

`postMessage`는 **출처 간 통신을 안전하게 하기 위한 도구**이지만,
**수신측 검증(Origin/Source/스키마/토큰)**을 빠뜨리면 즉시 **데이터 탈취와 행위 유발**의 통로가 됩니다.
위의 레시피를 적용하면 **구조적으로 안전한 채널**을 만들 수 있습니다.
