---
layout: post
title: HTML - Web Worker
date: 2025-04-18 21:20:23 +0900
category: HTML
---
# Web Worker (HTML5 비동기 처리의 핵심)

## 0) 한눈에 개요

- **정의**: 메인(UI) 스레드와 분리된 **백그라운드 스레드**에서 JS를 실행.  
- **목표**: **무거운 연산/블로킹 I/O**를 워커로 넘겨 **UI 멈춤 방지**.  
- **핵심 API**: `new Worker()`, `postMessage()`, `onmessage`, `terminate()`  
- **불가능**: DOM 접근, `alert/confirm`, 동기 XHR 등 UI-스레드 전용 기능.  
- **가능**: `fetch`, `async/await`, `IndexedDB`, `OffscreenCanvas`, `WebAssembly` 등.

---

## 1) 기본 구조와 왕초보 예제

### 폴더 구조
```
/index.html
/main.js
/worker.js
```

### index.html
```html
<!doctype html>
<html lang="ko">
  <head><meta charset="utf-8"><title>Worker 기본</title></head>
  <body>
    <button id="run">무거운 합계 계산</button>
    <pre id="log"></pre>
    <script type="module" src="./main.js"></script>
  </body>
</html>
```

### main.js
```js
const log = (...a) => (document.querySelector('#log').textContent += a.join(' ') + '\n');
const btn = document.querySelector('#run');
const worker = new Worker('./worker.js'); // 기본(Dedicated) 워커

worker.onmessage = (e) => {
  log('결과:', e.data);
};
worker.onerror = (e) => {
  log('워커 오류:', e.message || e);
};

btn.addEventListener('click', () => {
  // 0.8초 정도 걸리는 합계(환경차)
  worker.postMessage({ cmd: 'sum', n: 80_000_000 });
});
```

### worker.js
```js
self.onmessage = (e) => {
  const { cmd, n } = e.data || {};
  if (cmd === 'sum') {
    let s = 0;
    for (let i = 0; i < n; i++) s += i;
    postMessage(s);
  }
};
```

**포인트**
- 워커는 `self`(또는 `onmessage`) 컨텍스트에서 수신하고 `postMessage`로 응답.
- 메인 스레드는 `onmessage`로 결과를 받고 UI를 갱신.

---

## 2) 데이터 전달: Structured Clone vs Transferable

메시지는 **Structured Clone** 규칙으로 복사됩니다. 대용량 데이터는 **복사 비용**이 큼 → **Transferable**로 **소유권 이전** 권장.

### 2.1 Transferable(대표)
- `ArrayBuffer`, `MessagePort`, `ImageBitmap`, `OffscreenCanvas` 등.
- `postMessage(payload, [transferList])` 형태로 전달.

```js
// 메인: 대용량 버퍼 생성 후 워커로 "이전" (복사 X, 소유권 이동)
const buf = new ArrayBuffer(8_000_000); // 8MB
worker.postMessage({ cmd: 'process', buf }, [buf]); // buf는 메인에서 곧 비활성화됨
```

```js
// 워커: buf를 바로 활용
self.onmessage = (e) => {
  const { cmd, buf } = e.data;
  if (cmd === 'process') {
    const view = new Uint8Array(buf);
    // 처리...
    postMessage({ ok: true });
  }
};
```

**주의**: 이전 후 **원본은 비활성화**(byteLength=0처럼 보임). 필요 시 역방향으로도 다시 이전 가능.

---

## 3) 프로그레스, 취소, 에러 설계

### 3.1 진행률 전송
```js
// worker.js
onmessage = (e) => {
  const { cmd, n } = e.data;
  if (cmd === 'sumProgress') {
    let s = 0;
    const step = Math.max(1, Math.floor(n / 100));
    for (let i = 0; i < n; i++) {
      s += i;
      if (i % step === 0) postMessage({ type: 'progress', pct: Math.round(i / n * 100) });
    }
    postMessage({ type: 'done', value: s });
  }
};
```

```js
// main.js
worker.onmessage = (e) => {
  if (e.data.type === 'progress') {
    console.log('진행률', e.data.pct + '%');
  } else if (e.data.type === 'done') {
    console.log('합계', e.data.value);
  }
};
worker.postMessage({ cmd: 'sumProgress', n: 50_000_000 });
```

### 3.2 취소
- 간단한 방법: `worker.terminate()`
- 정교한 방법: **사용자 정의 Abort** 플래그 전달

```js
// main.js
let running = false;
let abortId = 0;
function start() {
  running = true;
  abortId++; // 새로운 작업 아이디
  worker.postMessage({ cmd: 'start', jobId: abortId, n: 80_000_000 });
}
function cancel() {
  running = false;
  worker.postMessage({ cmd: 'abort', jobId: abortId });
}
```

```js
// worker.js
let aborted = new Set();
onmessage = (e) => {
  const { cmd, jobId, n } = e.data;
  if (cmd === 'abort') aborted.add(jobId);
  if (cmd === 'start') {
    let s = 0;
    for (let i = 0; i < n; i++) {
      if (aborted.has(jobId)) { postMessage({ type: 'aborted', jobId }); return; }
      s += i;
    }
    postMessage({ type: 'done', jobId, value: s });
  }
};
```

### 3.3 에러
```js
// worker.js
self.onerror = (ev) => { /* 로그/리포트 */ };
try {
  // 코드
} catch (err) {
  postMessage({ type: 'error', message: String(err) });
}
```

메인에서 `worker.onerror`로 **치명적 에러** 수신, 일반 로직 에러는 메시지로 통일.

---

## 4) Module Worker, MIME, CORS

### 4.1 Module Worker
현대 환경에서는 ESM 권장.

```js
// main.js
const worker = new Worker(new URL('./module-worker.js', import.meta.url), { type: 'module' });
```

```js
// module-worker.js (type: module)
self.addEventListener('message', (e) => {
  // import도 사용 가능
  // import { foo } from './lib.js';
  postMessage({ got: e.data });
});
```

- 서버가 `Content-Type: text/javascript` 등 ESM을 올바르게 서빙해야 함.
- 로컬 파일(`file://`)이 아닌 **HTTP(S)** 환경 권장.

### 4.2 CORS
다른 출처의 워커 스크립트를 로드할 때는 CORS 헤더 필요. 또는 **Blob URL/데이터 URL**로 인라인 주입.

```js
// 인라인 워커(번들 환경에서 가끔 유용)
const code = `onmessage = e => postMessage('ok ' + e.data)`;
const blob = new Blob([code], { type: 'text/javascript' });
const url = URL.createObjectURL(blob);
const worker = new Worker(url);
worker.postMessage('hello');
```

---

## 5) SharedWorker, Service Worker와의 차이

| 구분 | Dedicated Worker | SharedWorker | Service Worker |
|---|---|---|---|
| 목적 | 탭 내부 전용 백그라운드 | **여러 탭** 간 공유 | 네트워크 프록시(PWA), 오프라인/푸시 |
| 수명 | 생성된 문맥에 종속 | 브라우저 프로세스 수준 | 사이트 범위(등록된 스코프) |
| 통신 | `postMessage` | `MessagePort`(연결 다수) | `fetch` 가로채기, 캐시, 푸시 |
| DOM | 접근 불가 | 접근 불가 | 접근 불가 |

SharedWorker는 동일 출처의 여러 탭이 **같은 워커 인스턴스**를 공유.

```js
// main.js (SharedWorker)
const sw = new SharedWorker('./shared.js');
sw.port.onmessage = (e) => console.log('from shared:', e.data);
sw.port.start();
sw.port.postMessage('hello');
```

```js
// shared.js
const clients = [];
onconnect = (e) => {
  const port = e.ports[0];
  clients.push(port);
  port.onmessage = (ev) => {
    // 브로드캐스트
    clients.forEach(p => p.postMessage(ev.data.toUpperCase()));
  };
};
```

---

## 6) OffscreenCanvas + Worker로 캔버스 렌더링

메인 UI 스레드 대신 **워커**에서 캔버스 렌더링을 수행 → 애니메이션/그림 처리 시 **프레임 드랍 감소**.

```html
<canvas id="c" width="600" height="400"></canvas>
<script type="module">
  const canvas = document.getElementById('c');
  const off = canvas.transferControlToOffscreen();
  const w = new Worker('./render-worker.js', { type: 'module' });
  w.postMessage({ canvas: off }, [off]); // TransferControl
</script>
```

```js
// render-worker.js
let ctx;
self.onmessage = (e) => {
  const { canvas } = e.data;
  if (canvas) {
    ctx = canvas.getContext('2d');
    draw();
  }
};
function draw(t = 0) {
  ctx.clearRect(0, 0, 600, 400);
  ctx.fillStyle = 'steelblue';
  ctx.beginPath();
  ctx.arc(300 + 100 * Math.cos(t/500), 200 + 100 * Math.sin(t/500), 40, 0, Math.PI*2);
  ctx.fill();
  requestAnimationFrame(draw);
}
```

---

## 7) Atomics + SharedArrayBuffer 로 락·신호·프로그레스

고급 패턴: **복사/이전 없이** 워커·메인 간 **동일 버퍼 공유**.  
단, **Cross-Origin Isolation** 필요(헤더: COOP/COEP). 배포 시 서버 설정 필수.

```js
// main.js
const sab = new SharedArrayBuffer(Int32Array.BYTES_PER_ELEMENT * 2);
const view = new Int32Array(sab); // [progress, flag]
const worker = new Worker('./sabar-worker.js');
worker.postMessage({ sab });

function waitAndRead() {
  // 진행률 폴링 예시(간단)
  console.log('progress', Atomics.load(view, 0));
  if (Atomics.load(view, 1) === 1) {
    console.log('done');
  } else {
    requestAnimationFrame(waitAndRead);
  }
}
waitAndRead();
```

```js
// sabar-worker.js
onmessage = (e) => {
  const { sab } = e.data;
  const view = new Int32Array(sab);
  for (let i = 0; i <= 100_000_000; i++) {
    if (i % 1_000_000 === 0) {
      Atomics.store(view, 0, Math.min(100, Math.floor(i/1_000_000)));
      // 필요하면 Atomics.notify(view, index, count)
    }
  }
  Atomics.store(view, 1, 1); // done flag
};
```

**주의**: 보안 헤더 설정이 안 되어 있으면 SharedArrayBuffer 사용 불가.

---

## 8) 워커 풀(Worker Pool)로 병렬 처리

단일 워커도 UI를 살리지만, **작업을 여러 워커로 분할**하면 **병렬 처리량**이 늘어남.

```js
// pool.js
export class WorkerPool {
  constructor(url, size = navigator.hardwareConcurrency || 4) {
    this.url = url;
    this.size = size;
    this.idle = [];
    this.busy = new Map();
    for (let i = 0; i < size; i++) {
      const w = new Worker(url, { type: 'module' });
      w.onmessage = (e) => {
        const { resolve } = this.busy.get(w) || {};
        this.busy.delete(w);
        this.idle.push(w);
        resolve?.(e.data);
      };
      w.onerror = (e) => {
        const { reject } = this.busy.get(w) || {};
        this.busy.delete(w);
        this.idle.push(w);
        reject?.(e.message || e);
      };
      this.idle.push(w);
    }
  }
  run(payload) {
    return new Promise((resolve, reject) => {
      const w = this.idle.pop();
      if (!w) return reject('모든 워커가 바쁨');
      this.busy.set(w, { resolve, reject });
      w.postMessage(payload);
    });
  }
  destroy() {
    [...this.idle, ...this.busy.keys()].forEach(w => w.terminate());
    this.idle = []; this.busy.clear();
  }
}
```

```js
// main.js
import { WorkerPool } from './pool.js';
const pool = new WorkerPool(new URL('./chunk-worker.js', import.meta.url), 4);

async function parallelSum(N = 80_000_000, parts = 4) {
  const step = Math.floor(N / parts);
  const tasks = [];
  for (let i = 0; i < parts; i++) {
    const a = i * step;
    const b = i === parts - 1 ? N : (i + 1) * step;
    tasks.push(pool.run({ a, b }));
  }
  const partial = await Promise.all(tasks);
  return partial.reduce((s, x) => s + x, 0);
}

parallelSum().then(v => console.log('합계', v));
```

```js
// chunk-worker.js
onmessage = (e) => {
  const { a, b } = e.data;
  let s = 0;
  for (let i = a; i < b; i++) s += i;
  postMessage(s);
};
```

---

## 9) 네트워크/스트리밍/파일 처리 in Worker

워커에서도 `fetch`/`Response.body` 스트림, `FileReader`(또는 `Blob.arrayBuffer`) 사용 가능.

```js
// worker.js
onmessage = async (e) => {
  const { url } = e.data;
  const res = await fetch(url);
  const reader = res.body.getReader(); // 바이트 스트림
  let received = 0;
  for (;;) {
    const { done, value } = await reader.read();
    if (done) break;
    received += value.byteLength;
    postMessage({ type: 'progress', received });
  }
  postMessage({ type: 'done' });
};
```

---

## 10) 번들러/프레임워크 연동

### 10.1 Vite/webpack + ESM
```js
// Vite 권장
const worker = new Worker(new URL('./w.js', import.meta.url), { type: 'module' });
```

### 10.2 React 커스텀 훅
```jsx
// useWorker.js
import { useEffect, useRef } from 'react';
export function useWorker(url, { module = true } = {}) {
  const ref = useRef(null);
  useEffect(() => {
    const w = new Worker(new URL(url, import.meta.url), { type: module ? 'module' : undefined });
    ref.current = w;
    return () => w.terminate();
  }, [url, module]);
  return ref;
}

// 사용
// const workerRef = useWorker('./worker.js');
// workerRef.current?.postMessage(...)
```

### 10.3 Comlink(메시지 RPC 추상화) 아이디어
메시지 구조를 직접 관리하기 번거롭다면 RPC 스타일 라이브러리를 활용(개념만 언급).

---

## 11) 성능 최적화 체크리스트

- **전달 비용 최소화**: 대용량은 **Transferable**(예: `ArrayBuffer`)로 전달.
- **작고 잦은 메시지 지양**: 배치/버퍼링 후 전송.
- **워커 수 조절**: `navigator.hardwareConcurrency` 근거로 상한 설정.
- **JSON 직렬화 부담 고려**: 숫자/타입드 배열 위주, 필요한 필드만.
- **연산 쪼개기**: 긴 루프는 **프로그레스 + 휴식**(yield)도 고려.
- **Warm-up**: WASM/ML 등 초기화 시간을 앱 시작 시 비동기로 미리 수행.
- **오류/리소스 해제**: 작업 종료 시 `terminate()` 또는 내부 루프 정리.

---

## 12) 보안/배포 체크리스트

- **CSP(Content-Security-Policy)**: 워커 스크립트 출처 허용(`worker-src`/`script-src`).
- **COOP/COEP**(Cross-Origin Isolation): `SharedArrayBuffer`/`Atomics`/고급 API 필요 시 서버 헤더 설정.
- **HTTPS**: 최신 API/성능 기능 대부분은 보안 컨텍스트 요구.
- **CORS**: 다른 출처 워커 로딩, 내부 import 시 헤더 설정.
- **무한 워커 생성 방지**: 사용자 액션마다 새 워커 생성하지 말고 **재사용**.

---

## 13) 디버깅/테스트 팁

- DevTools Sources → **Workers** 패널에서 코드/콘솔 확인.
- 워커 내부 `console.log`도 브라우저에 노출됨(브라우저별 위치 상이).
- E2E/단위 테스트 시 **메시지 프로토콜**을 별도 모듈로 분리해 순수 함수 테스트.
- 성능 측정: `performance.now()`, `console.time` 등으로 워커/메인 분리 비교.

---

## 14) 실전 미니 프로젝트 예시

### 14.1 이미지 그레이스케일 변환 (Transferable 활용)

```js
// main.js
const img = new Image();
img.src = 'image.jpg';
img.onload = async () => {
  const cvs = document.createElement('canvas');
  cvs.width = img.width; cvs.height = img.height;
  const ctx = cvs.getContext('2d');
  ctx.drawImage(img, 0, 0);
  const data = ctx.getImageData(0, 0, cvs.width, cvs.height);

  const worker = new Worker('./gray.js', { type: 'module' });
  worker.onmessage = (e) => {
    const out = new ImageData(new Uint8ClampedArray(e.data.buf), data.width, data.height);
    ctx.putImageData(out, 0, 0);
    document.body.appendChild(cvs);
  };
  // ArrayBuffer 이전
  worker.postMessage({ buf: data.data.buffer }, [data.data.buffer]);
};
```

```js
// gray.js
onmessage = (e) => {
  const u8 = new Uint8ClampedArray(e.data.buf); // RGBA
  for (let i = 0; i < u8.length; i += 4) {
    const g = (u8[i] + u8[i+1] + u8[i+2]) / 3 | 0;
    u8[i] = u8[i+1] = u8[i+2] = g;
  }
  postMessage({ buf: u8.buffer }, [u8.buffer]); // 다시 이전
};
```

### 14.2 대규모 JSON 파싱 + 전처리

```js
// worker.js
onmessage = (e) => {
  try {
    const obj = JSON.parse(e.data.text);  // 긴 문자열 파싱
    // 전처리
    const filtered = obj.items?.filter(x => x.score > 90) ?? [];
    postMessage({ ok: true, count: filtered.length });
  } catch (err) {
    postMessage({ ok: false, error: String(err) });
  }
};
```

메인에서는 파일/응답 본문을 문자열로 읽어서 워커에 전달.  
초대형이면 **스트리밍 파서**(NDJSON) 또는 IndexedDB와 조합 권장.

---

## 15) 자주 하는 실수와 방지법

- DOM 접근 시도 → 워커는 DOM 없음. **OffscreenCanvas** 또는 메시지로 UI 위임.
- 많은 작은 메시지 스팸 → **배치/디바운스**로 묶어서 전송.
- 모든 데이터를 JSON으로만 → **타입드 배열 + Transferable** 우선.
- 작업마다 워커 생성 → **풀/재사용**.
- `importScripts`를 모듈 환경에서 남용 → **Module Worker + ESM import**로 전환.
- SharedArrayBuffer를 아무 환경에서나 → **COOP/COEP** 필요 확인.

---

## 16) 마무리 요약

- 워커는 **UI 멈춤 없이** 무거운 연산을 수행하는 가장 쉬운 브라우저 병렬화 도구.
- 성능의 핵심은 **메시지 비용 최소화**(Transferable), **적절한 워커 수**, **올바른 분할 전략**.
- 고급 기능은 **OffscreenCanvas**, **SharedArrayBuffer + Atomics**, **워커 풀**로 확장.
- 보안·배포 요건(HTTPS, CORS, COOP/COEP, CSP)을 사전에 체크.

---

## 참고 링크(개념 확인용)
- MDN: Web Workers API  
- HTML Living Standard: Workers  
- OffscreenCanvas, Atomics/SharedArrayBuffer 관련 MDN 문서  
- 번들러별 Module Worker 가이드