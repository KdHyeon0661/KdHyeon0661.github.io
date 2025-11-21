---
layout: post
title: JavaScript - Fetch API
date: 2025-05-13 22:20:23 +0900
category: JavaScript
---
# Fetch API 사용법

**Fetch API**는 브라우저와 Node.js(18+)에서 공통으로 쓸 수 있는 표준 **Promise 기반 네트워킹 API**입니다. 이 글은 기초 GET부터 업로드/다운로드, 스트리밍, CORS, 캐시, 취소·타임아웃, 재시도, 보안까지 **실무용 패턴**을 한 번에 정리합니다.

---

## 빠른 시작 — 가장 작은 GET

```js
fetch("https://jsonplaceholder.typicode.com/posts/1")
  .then((res) => {
    if (!res.ok) throw new Error(`HTTP ${res.status}`);
    return res.json();
  })
  .then((data) => console.log(data))
  .catch((err) => console.error("Fetch 실패:", err));
```

핵심 요약
- `fetch()`는 **HTTP 에러(404/500)**에서도 **reject하지 않음** → `res.ok`/`res.status`를 **반드시 확인**.
- 본문은 **1회만 소비 가능**(body stream) → 재사용 시 `res.clone()`.

---

## 기본 패턴(Async/Await)

```js
async function fetchUser() {
  try {
    const res = await fetch("/api/user");
    if (!res.ok) throw new Error(`HTTP ${res.status}`);
    const data = await res.json();
    return data;
  } catch (e) {
    // 네트워크 오류, CORS 차단, 중단(Abort) 등
    console.error(e);
    return null;
  }
}
```

---

## Response 다루기 — 상태/헤더/본문 1회 규칙

```js
const res = await fetch("/api/data");

console.log(res.ok, res.status, res.statusText);
console.log(res.headers.get("content-type")); // e.g. application/json

// 본문 소비(한 번만 가능)
const text = await res.text();
// 다시 필요하면 clone
// const copy = res.clone(); await copy.json();
```

- `res.ok`: 200–299
- **본문 1회 규칙**: `res.text()`, `res.json()`, `res.blob()` 등 중 **한 번만** 호출 가능 → **로그 + 파싱** 필요 시 `clone()`.

---

## 다양한 본문 파싱

```js
await res.text();        // 문자열
await res.json();        // JSON → 객체
await res.blob();        // 파일/바이너리(다운로드/미리보기)
await res.arrayBuffer(); // 저수준 바이너리
await res.formData();    // multipart/form-data 파싱
```

응답 `Content-Type`과 무관하게 메서드 선택은 **개발자 책임**입니다(서버가 올바른 타입을 주지 않아도 호출 가능).

---

## 요청 옵션 총정리

```js
fetch(url, {
  method: "GET",                 // "POST" | "PUT" | "PATCH" | "DELETE" ...
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ a: 1 }),// GET/HEAD 제외
  mode: "cors",                  // "cors" | "no-cors" | "same-origin"
  credentials: "same-origin",    // "omit" | "same-origin" | "include"
  cache: "no-cache",             // "default" | "no-store" | "reload" | "no-cache" | "force-cache" | "only-if-cached"
  redirect: "follow",            // "follow" | "error" | "manual"
  referrer: "about:client",      // 또는 URL
  referrerPolicy: "strict-origin-when-cross-origin",
  integrity: "",                 // SRI
  keepalive: false,              // 페이지 언로드 시에도 전송 계속할지(작은 요청)
  signal: abortController.signal,// 취소/타임아웃
  // priority: "high" | "low" | "auto" (일부 브라우저)
});
```

팁
- `mode: "no-cors"`는 **대부분의 커스텀 헤더/메소드 불가**, 응답이 **opaque**로 돌아와 **읽을 수 없음** → 문제 해결책이 아님(서버 CORS 설정이 정답).
- `credentials`는 **쿠키·HTTP 인증 헤더** 포함 동작을 제어(아래 §12).

---

## POST 패턴 모음

### JSON POST

```js
await fetch("/api/login", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ username: "alice", password: "1234" }),
});
```

### `application/x-www-form-urlencoded`

```js
const params = new URLSearchParams({ q: "hello", page: 1 });
await fetch("/search", {
  method: "POST",
  headers: { "Content-Type": "application/x-www-form-urlencoded;charset=UTF-8" },
  body: params.toString(),
});
```

### `multipart/form-data`(파일 업로드)

```js
const fd = new FormData();
fd.append("avatar", fileInput.files[0]);
fd.append("displayName", "Alice");
await fetch("/profile", { method: "POST", body: fd }); // 헤더 자동
```

---

## 파일 업로드/다운로드

### 업로드 진행률(대안)

- 표준 Fetch는 **업로드 진행률 이벤트 미지원**.
  옵션: `XMLHttpRequest`로 전환 또는 **Service Worker + Streams** 커스텀(고급).

### 다운로드 → Blob 저장

```js
const res = await fetch("/report.pdf");
if (!res.ok) throw new Error();
const blob = await res.blob();
const url = URL.createObjectURL(blob);
const a = document.createElement("a");
a.href = url; a.download = "report.pdf"; a.click();
URL.revokeObjectURL(url);
```

---

## 에러 처리 전략

### HTTP 에러는 `ok`로 판정 X

```js
const res = await fetch("/api");
if (!res.ok) throw new Error(`HTTP ${res.status}`);
```

### 네트워크/중단/CORS 차단

- DNS 실패/네트워크 끊김/요청 취소/브라우저 보안 차단 → **Promise reject**.
- CORS 위반 시 **opaque**/reject/에러 메시지 제약.

### — **멱등 메서드 우선**

```js
async function fetchWithRetry(url, init = {}, { retries = 3, backoff = 300 } = {}) {
  let lastErr;
  for (let i = 0; i < retries; i++) {
    try {
      const res = await fetch(url, init);
      if (!res.ok) throw new Error(`HTTP ${res.status}`);
      return res;
    } catch (e) {
      lastErr = e;
      await new Promise(r => setTimeout(r, backoff * 2 ** i));
    }
  }
  throw lastErr;
}
```
- **POST** 재시도는 **중복 생성** 위험 → 서버의 **Idempotency-Key** 전략 권장.

---

## 취소와 타임아웃

### `AbortController`

```js
const ac = new AbortController();
const p = fetch("/slow", { signal: ac.signal })
  .catch(e => { if (e.name === "AbortError") console.log("취소됨"); });
setTimeout(() => ac.abort(), 1000);
await p;
```

### 타임아웃 헬퍼

```js
async function fetchWithTimeout(url, init={}, ms=5000) {
  const ac = new AbortController();
  const id = setTimeout(() => ac.abort(), ms);
  try {
    return await fetch(url, { ...init, signal: ac.signal });
  } finally {
    clearTimeout(id);
  }
}
```

> 일부 환경은 `AbortSignal.timeout(ms)` 또는 `AbortSignal.any([...])` 지원.

---

## 병렬·병목 제어(동시성 제한)

```js
async function withLimit(limit, tasks) {
  const q = new Set();
  const out = [];
  for (const t of tasks) {
    const job = Promise.resolve().then(t).finally(() => q.delete(job));
    q.add(job); out.push(job);
    if (q.size >= limit) await Promise.race(q);
  }
  return Promise.allSettled(out);
}

const urls = [...Array(20)].map((_, i) => `/api/item/${i}`);
await withLimit(5, urls.map(u => () => fetch(u)));
```

---

## — 대용량/점진 처리

### 텍스트 스트림 읽기(진행률)

```js
const res = await fetch("/large.txt");
const reader = res.body.getReader();
const decoder = new TextDecoder();
let received = 0, chunks = "";

while (true) {
  const { value, done } = await reader.read();
  if (done) break;
  received += value.length;
  chunks += decoder.decode(value, { stream: true });
  // 진행률 업데이트: received / (parseInt(res.headers.get("content-length")) || NaN)
}
chunks += decoder.decode();
```

### NDJSON(줄 단위 JSON)

```js
async function* ndjson(stream) {
  const reader = stream.getReader();
  const dec = new TextDecoder();
  let buf = "";
  for (;;) {
    const { value, done } = await reader.read();
    if (done) break;
    buf += dec.decode(value, { stream: true });
    let idx;
    while ((idx = buf.indexOf("\n")) >= 0) {
      const line = buf.slice(0, idx).trim();
      buf = buf.slice(idx + 1);
      if (line) yield JSON.parse(line);
    }
  }
  if (buf.trim()) yield JSON.parse(buf.trim());
}

for await (const row of ndjson((await fetch("/stream")).body)) {
  console.log("row", row);
}
```

> 서버 Sent Events(SSE)는 `EventSource` 사용이 간단, 웹소켓은 양방향.

---

## 인증/쿠키·세션과 `credentials`

```js
await fetch("/me", { credentials: "include" });
```

- `credentials: "include"`: **CORS 요청에도 쿠키 포함**(서버는 `Access-Control-Allow-Credentials: true`와 **정확한** `Access-Control-Allow-Origin` 필요, `*` 불가).
- SameSite 기본 `Lax`인 쿠키는 크로스 사이트 서브요청에 제한 → CSRF 완화. API 설계 시 **CSRF 토큰** 또는 **Authorization Bearer 토큰** 고려.

---

## CORS 심화

- 사전 요청(Preflight): `OPTIONS` + `Access-Control-Request-*`
  서버는 `Access-Control-Allow-Origin`, `-Methods`, `-Headers` 적절 응답 필요.
- `mode: "no-cors"`: 응답이 **opaque**(읽을 수 없음), 대부분의 커스텀 헤더/메소드 차단 → **해결책 아님**.
- **정답은 서버 CORS 설정**: 필요한 오리진/헤더/메서드만 허용.

---

## 캐싱 전략

### HTTP 캐시 + `cache` 옵션

- `cache: "no-cache"`: **검증 후** 사용(If-None-Match/If-Modified-Since)
- `cache: "reload"`: 네트워크 강제
- `cache: "force-cache"`: 캐시 우선
- ETag/Last-Modified를 서버가 제공하면 효율↑.

### Cache Storage API(프로그래매틱 캐시)

```js
const res = await fetch("/data");
const cache = await caches.open("v1");
await cache.put("/data", res.clone());
const cached = await cache.match("/data");
```

> 서비스 워커와 결합하면 **오프라인/프리페치/폴백** 구현.

---

## 서비스 워커 연계(요약)

- `fetch` 이벤트 가로채서 캐싱/프록시/폴백.
- 전략: **Cache First**, **Network First**, **Stale-While-Revalidate** 등.

---

## 리다이렉트

- 기본 `redirect: "follow"`(최대 횟수 제한).
- `redirect: "manual"`은 브라우저에서 `opaqueredirect`로 제한적(헤더 확인 어려움) — 보통 서버/라우터에서 처리.

---

## 보안 모범사례

- **XSS**: 서버가 반환한 HTML을 `innerHTML`로 주입하지 말고, 가능하면 텍스트로. JSON은 렌더 전 **escape**.
- **CSRF**: 쿠키 기반 세션이면 **SameSite=Strict/Lax + CSRF 토큰**. API는 Bearer 토큰 추천.
- **SRI**(`integrity`) + **CSP**로 외부 스크립트 방어.
- **민감정보 로그 금지**(Authorization, 쿠키).

---

## 브라우저 vs Node.js

- Node.js **18+**: `globalThis.fetch` 기본 제공(undici 기반).
- 차이:
  - 일부 브라우저 전용 옵션/기능(예: `keepalive` 제한) vs Node 환경 차이.
  - 파일 업로드 시 `FormData`/`Blob` 지원(최근 Node 가능).
  - 쿠키/세션은 브라우저가 자동 관리하지만, Node는 **쿠키 헤더 수동 처리** 필요(또는 라이브러리).

---

## 실전 유틸 함수 모음

### 안전 JSON 헬퍼

```js
export async function fetchJson(url, init) {
  const res = await fetch(url, init);
  if (!res.ok) throw new Error(`HTTP ${res.status}`);
  const ct = res.headers.get("content-type") || "";
  if (!ct.includes("application/json")) {
    const text = await res.text();
    throw new Error(`예상치 못한 응답 타입: ${ct}\n${text.slice(0,200)}`);
  }
  return res.json();
}
```

### 타임아웃 + 재시도 합본

```js
export async function robustFetch(url, init={}, { timeout=8000, retries=2 }={}) {
  for (let i=0;; i++) {
    try {
      const ac = new AbortController();
      const id = setTimeout(() => ac.abort(), timeout);
      const res = await fetch(url, { ...init, signal: ac.signal });
      clearTimeout(id);
      if (!res.ok) throw new Error(`HTTP ${res.status}`);
      return res;
    } catch (e) {
      if (i >= retries) throw e;
      await new Promise(r => setTimeout(r, 300 * 2 ** i));
    }
  }
}
```

### 업로드 헬퍼(FormData)

```js
export async function upload(url, files, extra = {}) {
  const fd = new FormData();
  for (const [name, file] of Object.entries(files)) fd.append(name, file);
  for (const [k, v] of Object.entries(extra)) fd.append(k, v);
  const res = await fetch(url, { method: "POST", body: fd });
  if (!res.ok) throw new Error(`HTTP ${res.status}`);
  return res.json().catch(() => ({}));
}
```

---

## 디버깅 팁

- **DevTools Network 탭**: 요청/응답/헤더/CORS/캐시 상태 확인.
- **콘솔에서 재전송**: Network 항목 우클릭 → Copy as fetch.
- **Service Worker 영향**: Application 패널에서 SW 무력화/언레지스터 확인.

---

## 체크리스트 & 미니 퀴즈

체크리스트
- [ ] `res.ok` 확인(HTTP 에러 수동 처리)
- [ ] 본문 **1회 소비** 규칙 기억(`clone` 필요 시 사용)
- [ ] CORS는 **서버 설정**으로 해결 — `no-cors`는 해법 아님
- [ ] 긴 요청은 **Abort + 타임아웃** 적용
- [ ] 재시도는 **멱등 요청에 한해** 백오프 전략
- [ ] 대용량은 **스트리밍/NDJSON**으로 점진 처리
- [ ] 쿠키 필요 시 `credentials` / SameSite/CSRF 고려
- [ ] 캐시 정책(HTTP/Cache Storage) 명확화

퀴즈
```js
// Q1) 왜 이 코드는 오류일까?
const res = await fetch("/api");
const a = await res.json();
const b = await res.text(); // ?

// Q2) 3초 내 응답이 없으면 중단하는 fetch를 작성하라.

// Q3) CORS 에러를 클라이언트에서 해결하려면 mode:"no-cors"로 바꾸면 된다 (O/X)?

// Q4) 다음 중 쿠키를 반드시 포함하려면?
fetch("/me", { /* ? */ });

// Q5) 다음 옵션 중 캐시 검증 후 네트워크 사용할 수 있는 것은?
// "reload" | "no-cache" | "force-cache"
```

**정답 힌트**
- Q1: 본문은 한 번만 소비 가능 → `clone()` 필요
- Q2: `AbortController` + `setTimeout`
- Q3: X(서버 CORS 설정이 정답)
- Q4: `credentials: "include"`(CORS 시), `same-origin`(동일 출처)
- Q5: `no-cache`(재검증), `reload`는 강제 네트워크, `force-cache`는 캐시 우선

---

## 부록: 실전 예제 — 검색 API(디바운스 + 취소 + 동시성 제한)

```js
const input = document.querySelector("#q");
let lastCtrl = null;

input.addEventListener("input", debounce(async (e) => {
  const q = e.target.value.trim();
  if (!q) return render([]);

  // 이전 요청 취소
  lastCtrl?.abort();
  lastCtrl = new AbortController();

  try {
    const res = await fetch(`/api/search?q=${encodeURIComponent(q)}`, {
      signal: lastCtrl.signal,
      headers: { "Accept": "application/json" },
      cache: "no-cache"
    });
    if (!res.ok) throw new Error(`HTTP ${res.status}`);
    const data = await res.json();
    render(data.items);
  } catch (e) {
    if (e.name !== "AbortError") console.error(e);
  }
}, 250));

function debounce(fn, ms=200) {
  let t; return (...args) => { clearTimeout(t); t = setTimeout(() => fn(...args), ms); };
}

function render(items) {
  const ul = document.querySelector("#results");
  ul.innerHTML = items.map(x => `<li>${escapeHtml(x.title)}</li>`).join("");
}

function escapeHtml(s) {
  return s.replace(/[&<>"']/g, (m) => ({ "&":"&amp;", "<":"&lt;", ">":"&gt;", '"':"&quot;", "'":"&#39;" }[m]));
}
```

이 예제는 **타이핑 시 디바운스**, **이전 요청 취소**, **XSS 방지**까지 포함한 **현업형 Fetch 패턴**입니다.
