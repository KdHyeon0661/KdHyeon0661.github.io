---
layout: post
title: 웹해킹 - Service Worker 남용, 캐시 포이즈닝
date: 2025-10-05 21:25:23 +0900
category: 웹해킹
---
# Service Worker 남용 / 캐시 포이즈닝

## 한눈에 보기 (Executive Summary)

- **문제(요약)**
  - 악성 SW가 등록되면 **해당 스코프의 모든 요청**을 가로채서 **변조/유출/캐시 포이즈닝**을 할 수 있음.
  - 취약한 **등록 경로**(XSS, 반사형 JS, MIME 스니핑, 오리진 혼합)·너무 넓은 **스코프**(`/`)·무방비 **캐시 로직**이 주요 원인.
- **핵심 방어**
  1) **스코프 최소화**: `/app/sw.js` → `scope: '/app/'` (기본: 스크립트 경로 하위만)
  2) **등록 경로 보호**: SW 파일은 **정적 빌드 산출물**만 서빙, `Content-Type: application/javascript` + **`X-Content-Type-Options: nosniff`**, **HTTPS 전용**
  3) **CSP/worker-src**로 SW **출처 화이트리스트**, **`Service-Worker-Allowed`** 남발 금지
  4) **캐시 무결성**: **화이트리스트 경로/타입만 캐시**, **해시 검증**, **opaque 응답 미캐시**, **키 정규화**
  5) **킬스위치/청소**: `Clear-Site-Data`, 버전 롤링, `registration.unregister()` 경로

---

# 배경 — SW가 “오프라인 프록시”인 이유

- **핵심 API**: `self.addEventListener('fetch', ...)` 에서 **모든 네트워크 요청**을 가로채 **임의 응답** 가능.
- **지속성**: 등록되면 브라우저 저장소(서비스워커·CacheStorage)에 남음 → **로그아웃/쿠키 삭제와 무관**.
- **스코프**: 기본적으로 **SW 스크립트가 위치한 디렉터리 이하**만 제어. `Service-Worker-Allowed` 헤더로 확장 가능(❌ 남발 금지).

---

# 대표 위험 시나리오

## “루트 스코프” 과도 제어

```js
// ❌ 나쁜 예: /sw.js + scope: '/'  → 오리진 전체 프록시화
navigator.serviceWorker.register('/sw.js', { scope: '/' });
```
- 원치 않는 경로(로그인/결제/관리자 페이지)까지 **전면 중간자**가 됨.

## 등록 경로가 오염 가능

- **XSS** 지점이 `/sw.js`와 **같은 오리진·경로 하위**에 존재하고,
- 반사 응답이 `Content-Type: application/javascript`로 스니핑되면,
- 공격자가 **임의 JS**를 `/sw.js`로 주입해 등록 → **지속 감염**.

## 캐시 포이즈닝(Worker 내부)

- SW가 `fetch`로 받은 응답을 **검증 없이 put**하거나,
- **opaque** 응답(교차 오리진, no-cors)을 **그대로 캐시**하면 **임의 페이로드**가 **오프라인 시 제공**됨.

## 메시지/업데이트 경로 오용

- `postMessage`로 **캐시 갱신 명령**을 받아 **임의 URL**을 캐시하는 로직 → 외부 조작에 취약.
- SW 스크립트 자체가 **강하게 캐시**되어 업데이트가 지연 → 악성 상태 지속.

---

# — **막혀야 정상 ✅**

1) **루트 스코프 등록 거절**
   ```js
   navigator.serviceWorker.register('/sw.js', { scope: '/' })
   // 기대: 서버에서 sw.js 응답 헤더/정책으로 루트 제어가 불가하거나, 앱 코드에서 자체적으로 scope 제한
   ```

2) **MIME 스니핑 불가**
   - `/echo?q=...` 같은 엔드포인트로 `</script>...` 등 주입 →
   - `Content-Type: text/plain` + **`nosniff`**로 **JS로 인식/실행되지 않아야** 함.
   - `/sw.js` 경로는 **빌드 산출물**만 매핑되고 **동적 라우트·리다이렉트 금지**.

3) **캐시 화이트리스트 적용 확인**
   - 교차 오리진 리소스(opaque) 요청 → **캐시되지 않아야** 함.
   - 허용되지 않은 경로/타입 → **캐시 miss**.

4) **킬스위치 동작**
   - 운영 플래그로 **즉시 언레지스터/캐시 삭제**가 되는지 확인.

---

# — 안전한 등록/해제 패턴

## 등록 시 스코프 최소화 + 출처 고정

```js
// ✅ 좋은 예: 앱 섹션 전용
if ('serviceWorker' in navigator) {
  window.addEventListener('load', async () => {
    try {
      // /app/ 이하만 제어. 루트를 절대 쓰지 않음.
      const reg = await navigator.serviceWorker.register('/app/sw.js', { scope: '/app/' });
      console.log('SW registered', reg.scope);
    } catch (e) {
      console.warn('SW register failed', e);
    }
  });
}
```

### CSP로 등록 출처 제한

```html
<!-- 페이지(등록을 허용하는 곳)에서 -->
<meta http-equiv="Content-Security-Policy" content="
  default-src 'self';
  script-src 'self' 'strict-dynamic' 'nonce-__NONCE__';
  worker-src 'self';       /* SW/Workers는 반드시 same-origin만 */
  child-src 'self';        /* 구형 브라우저 호환 */
  object-src 'none';
">
```
> **포인트**: `worker-src 'self'`(필수), 등록을 허용하지 않는 페이지는 **`worker-src 'none'`**로 **전면 차단**.

## 제어 범위 확인/강제 언레지스터(사용자/운영 킬스위치)

```js
// 설정 화면 등에서 수동 해제 옵션 제공
async function unregisterAllSW() {
  if (!('serviceWorker' in navigator)) return;
  const regs = await navigator.serviceWorker.getRegistrations();
  await Promise.all(regs.map(r => r.unregister()));
  if ('caches' in window) {
    const names = await caches.keys();
    await Promise.all(names.map(n => caches.delete(n)));
  }
  // (선택) 페이지 리로드 권고
}

// 운영 긴급 킬스위치 (백엔드 플래그)
(async () => {
  const kill = await fetch('/app/sw-kill').then(r => r.ok ? r.text() : '0');
  if (kill.trim() === '1') unregisterAllSW();
})();
```

---

# 서버 — “등록 경로 보호” 헤더/서빙

## Nginx (정적 sw.js 전용, 안전 헤더)

```nginx
# SW 스크립트는 오직 빌드 산출물 폴더에서만

location = /app/sw.js {
  add_header Content-Type "application/javascript; charset=utf-8";
  add_header X-Content-Type-Options "nosniff" always;
  # 등록 스코프를 /app/ 밖으로 넓히지 않음 (헤더 미설정 시 기본: 자신 디렉터리 이하)
  # add_header Service-Worker-Allowed "/app/";  # 필요한 경우에만 최소 범위로

  # 업데이트 지연 방지 (필요 시 짧은 캐시 또는 항상 재검증)
  add_header Cache-Control "no-cache, must-revalidate";
  try_files /build/sw.js =404;
}

# 등록을 금지할 영역(예: 루트·관리자)

location = /sw.js { return 403; }       # 루트 sw.js는 차단
add_header Content-Security-Policy "worker-src 'none'" always;
```

## — MIME 고정 + nosniff

```js
app.get('/app/sw.js', (req,res)=>{
  res.set('Content-Type','application/javascript; charset=utf-8');
  res.set('X-Content-Type-Options','nosniff');
  res.set('Cache-Control','no-cache, must-revalidate');
  res.sendFile(path.join(__dirname,'build/sw.js'));
});
```

> **반드시**: `/app/sw.js` 경로에 **동적 라우트/리다이렉트/반사형 응답**을 두지 마세요.
> SW 스크립트는 **단일, 서명된 정적 파일**로만 제공해야 합니다.

---

# Service Worker 내부 — **안전한 fetch/캐시** 구현

## 기본 골격(스코프 한정·무분별 차단)

```js
/// <reference lib="webworker" />
const VERSION = 'v2025.10.28';
const CACHE_STATIC = `static-${VERSION}`;
const ALLOW_PATHS = ['/app/static/', '/app/assets/']; // 화이트리스트 경로

self.addEventListener('install', (e) => {
  // 즉시 활성화가 필요하면:
  self.skipWaiting();
  e.waitUntil(caches.open(CACHE_STATIC).then(c => c.addAll([
    // 초기 프리캐시(정적만)
    '/app/assets/app.css',
    '/app/assets/app.js',
  ])));
});

self.addEventListener('activate', (e) => {
  e.waitUntil((async () => {
    const names = await caches.keys();
    await Promise.all(names.filter(n => !n.endsWith(VERSION)).map(n => caches.delete(n)));
    await self.clients.claim();
  })());
});

// 동일 오리진, GET, 화이트리스트 경로만 취급
self.addEventListener('fetch', (event) => {
  const req = event.request;
  const url = new URL(req.url);

  // 1) 동일 오리진만
  if (url.origin !== self.location.origin) return;
  // 2) 안전 메서드만
  if (req.method !== 'GET') return;
  // 3) 허용 경로만
  if (!ALLOW_PATHS.some(p => url.pathname.startsWith(p))) return;

  event.respondWith(cachedOrNetwork(req));
});

async function cachedOrNetwork(req) {
  const cache = await caches.open(CACHE_STATIC);
  const cached = await cache.match(req, { ignoreSearch: false });
  if (cached) return cached;

  const res = await fetch(req, { credentials: 'same-origin' });

  // 4) 캐시 무결성: 상태/타입/크기/opaque 거부
  if (!res.ok) return res;
  if (res.type === 'opaque') return res;                    // ❌ 외부/무검증 응답 미캐시
  const ctype = res.headers.get('content-type') || '';
  if (!/^text\/css|application\/javascript|image\//.test(ctype))
    return res;                                             // ❌ HTML 등 동적 문서 미캐시

  // 5) (선택) 해시 검증: 매니페스트와 일치할 때만 캐시
  const isValid = await verifyDigest(req, res.clone());
  if (!isValid) return res;

  await cache.put(req, res.clone());
  return res;
}
```

## 콘텐츠 해시 검증(캐시 포이즈닝 방지)

```js
// 빌드 시 생성하는 매니페스트(경로 → SHA-256 hex)
const DIGESTS = {
  '/app/assets/app.js': 'ab12cd34...ef',
  '/app/assets/app.css': '0011aa22...ff',
};

async function verifyDigest(req, res) {
  const url = new URL(req.url);
  const expected = DIGESTS[url.pathname];
  if (!expected) return false;
  const buf = await res.clone().arrayBuffer();
  const hash = await crypto.subtle.digest('SHA-256', buf);
  const hex = [...new Uint8Array(hash)].map(b=>b.toString(16).padStart(2,'0')).join('');
  return hex === expected;
}
```
> **포인트**
> - **화이트리스트 경로/타입만 캐시**.
> - **opaque 응답 미캐시**(교차 오리진/모호).
> - **해시 매니페스트**로 **변조 감지**(SRI와 상보).

## 메시지 핸들링(안전)

```js
// 외부에서 캐시 조작 명령을 받지 않거나, 최소한 엄격 검증
self.addEventListener('message', (event) => {
  // event.source.origin 없음 — 메시지는 same-origin 페이지만 허용 가정
  const { type } = event.data || {};
  if (type === 'PURGE') {
    // (선택) 관리자 페이지에서만 보내도록 앱 쪽에서 토큰/컨텍스트 검증
    caches.keys().then(names => names.forEach(n => caches.delete(n)));
  }
});
```

---

# SRI와의 결합 — **강한 무결성**

- **SRI(Subresource Integrity)**는 `<script src="...">`, `<link rel="stylesheet">`에 대해 **해시 검증**을 수행합니다.
- SW가 제공하는 정적 자산에도 SRI를 걸어두면, **SW가 변조된 파일을 서빙해도 브라우저 로드가 실패** → **알림/롤백**.

```html
<link rel="stylesheet" href="/app/assets/app.css"
      integrity="sha256-0011aa22..." crossorigin="anonymous">
<script src="/app/assets/app.js" defer
        integrity="sha256-ab12cd34..." crossorigin="anonymous"></script>
```

> **중요**: SRI는 **리소스**에 적용됩니다. **SW 스크립트 자체**에는 SRI 속성을 붙일 수 없으므로,
> **SW는 정적 빌드 산출물로만** 배포하고 **등록 경로 보호**(§5)를 철저히 하세요.

---

# “스코프 제한”과 `Service-Worker-Allowed` 주의

- 기본적으로 **`/app/sw.js` → `/app/` 이하만** 제어.
- `Service-Worker-Allowed: /`를 설정하면 **루트까지 확장**됩니다(❌ 가급적 금지).
- **정말 필요할 때만**, **최소 경로**로 설정하세요.

```nginx
# 예: /app/ 하위까지만

add_header Service-Worker-Allowed "/app/" always;
```

---

# 구역 분리 — 사용자 콘텐츠/관리자/정적

- **사용자 생성 콘텐츠(UGC)**는 **별도 서브도메인**(예: `cdn-user.example.com`)에 격리하고,
  그 도메인에는 **SW 등록 자체 금지**(페이지 CSP: `worker-src 'none'`).
- **관리자/결제** 구역은 **SW 미사용**(페이지 CSP로 차단) 또는 별도 오리진.

```html
<!-- 관리자 페이지: SW 금지 -->
<meta http-equiv="Content-Security-Policy" content="worker-src 'none'">
```

---

# Clear-Site-Data — **강제 정리(캐시/스토리지/SW)**

보안사고/계약해지/테넌트 삭제 등에서 **원격으로 모든 로컬 데이터**를 정리합니다.

```http
HTTP/1.1 200 OK
Clear-Site-Data: "cache", "cookies", "storage"
```

> `"storage"`에는 **IndexedDB/LocalStorage/ServiceWorker 등록** 등이 포함됩니다(브라우저별 차이는 문서 확인).
> 사용자는 **로그아웃** 또는 **보안 이벤트** 시 이 헤더가 적용된 페이지로 유도하세요.

---

# 운영 체크리스트

- [ ] SW는 **루트가 아닌** `/app/` 등 한정 스코프에만
- [ ] `/app/sw.js`는 **정적 빌드 산출물**로 제공(동적 라우트/반사 금지)
- [ ] **`Content-Type: application/javascript` + `X-Content-Type-Options: nosniff`**
- [ ] **CSP**: `worker-src 'self'`(등록 허용 페이지), 그 외 페이지는 `worker-src 'none'`
- [ ] **`Service-Worker-Allowed`**는 **최소 경로** 또는 **미설정**
- [ ] SW 코드는 **동일 오리진·GET·화이트리스트 경로/타입**만 캐시
- [ ] **opaque 응답 미캐시**, **해시 매니페스트 검증**
- [ ] **버전 롤링 + 오래된 캐시 삭제**(activate 단계)
- [ ] **SRI**로 정적 자산 무결성 강화
- [ ] **Clear-Site-Data**로 킬스위치 경로 마련
- [ ] E2E/스테이징에서 **루트 스코프 등록 시도·캐시 포이즈닝 시도**가 **항상 실패**하는지 자동화

---

## “막혀야 정상” 테스트

```ts
test('루트 스코프 등록 차단', async ({ page }) => {
  await page.goto('https://staging.example.com/');
  const err = await page.evaluate(async () => {
    try {
      await navigator.serviceWorker.register('/sw.js', { scope: '/' });
      return 'registered';
    } catch (e) { return 'blocked'; }
  });
  expect(err).toBe('blocked');
});

test('opaque 응답 캐시 금지', async ({ page }) => {
  await page.goto('https://staging.example.com/app/');
  // 테스트용 메시지로 임의 URL 캐시 유도 시도 → SW가 무시해야 함
  const result = await page.evaluate(() => new Promise(res => {
    navigator.serviceWorker.controller.postMessage({type:'CACHE', url:'https://x.example.net/a.png'});
    setTimeout(()=>res('done'), 300);
  }));
  expect(result).toBe('done');
  // (네트워크 로그로 캐시 put 없음을 검증하거나, SW가 보고하도록 설계)
});
```

---

## 맺음말

Service Worker는 **성능·오프라인의 핵**이지만, 동시에 **오리진 전역 프록시**가 될 수 있는 **양날의 검**입니다.
**스코프 최소화 · 등록 경로 보호 · 강한 무결성(CSP/SRI) · 신중한 캐시 정책 · 킬스위치**를 표준으로 삼으면,
SW 남용/캐시 포이즈닝 리스크를 **구조적으로 줄일 수** 있습니다.
