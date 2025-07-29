---
layout: post
title: HTML - Web Worker
date: 2025-04-18 21:20:23 +0900
category: HTML
---
# 🧵 Web Worker 완전 정리 (HTML5 비동기 처리의 핵심)

## ✅ Web Worker란?

**Web Worker**는 HTML5에서 도입된 API로,  
**JavaScript를 백그라운드에서 병렬로 실행**할 수 있게 해주는 기술입니다.

> JavaScript는 기본적으로 싱글 스레드(Single-thread)입니다.  
> 무거운 연산을 실행하면 **UI가 멈추거나 브라우저가 응답 없음 상태**가 되는데,  
> 이를 해결하기 위해 **Worker 스레드를 따로 생성**해서 처리합니다.

---

## 🔧 언제 쓰나?

| 사용 예 | 설명 |
|---------|------|
| 대용량 데이터 처리 | 예: JSON 파싱, 암호화, 압축 |
| 이미지 필터링 및 렌더링 | 예: 캔버스 픽셀 처리 |
| AI 추론/머신러닝 계산 | 예: TensorFlow.js |
| 실시간 계산 | 예: 마우스 위치 추적, 게임 물리 연산 |
| 메인 UI 보호 | 무거운 연산으로 UI 멈춤 방지

---

## 🧱 기본 구조

Web Worker는 **main script**와 **worker script**로 나뉘어 작동합니다.

```text
Main Thread  <── message ──>  Worker Thread
```

---

## 🧪 기본 사용법

### 📁 파일 구조

```
/index.html
/worker.js
```

### 📄 main.js

```javascript
const worker = new Worker("worker.js");

// 워커로 메시지 보내기
worker.postMessage("안녕 Worker!");

// 워커에서 메시지 받기
worker.onmessage = (event) => {
  console.log("Worker로부터 응답:", event.data);
};
```

### 📄 worker.js

```javascript
onmessage = (event) => {
  console.log("Main으로부터 메시지:", event.data);
  // 계산 수행
  const result = event.data + " 👋";
  // 결과 반환
  postMessage(result);
};
```

---

## 🧠 주요 메서드 및 속성

| 메서드 / 속성 | 설명 |
|----------------|------|
| `new Worker(url)` | 워커 인스턴스 생성 (JS 파일 경로 필요) |
| `worker.postMessage(data)` | 워커에 메시지 전송 |
| `worker.onmessage` | 워커로부터 메시지 수신 |
| `worker.terminate()` | 워커 종료 |
| `onerror` | 워커 내부 오류 발생 시 처리 |

---

## 🧩 예제: 무거운 계산 비동기 처리

### `main.js`

```javascript
const worker = new Worker("sum-worker.js");

worker.postMessage(1_000_000_000); // 10억까지 합산 요청

worker.onmessage = (e) => {
  console.log("합계 결과:", e.data);
};
```

### `sum-worker.js`

```javascript
onmessage = (e) => {
  let sum = 0;
  for (let i = 0; i < e.data; i++) {
    sum += i;
  }
  postMessage(sum);
};
```

> 💡 이 작업을 메인 스레드에서 하면 페이지가 몇 초간 멈춥니다. Worker를 쓰면 백그라운드에서 처리됩니다.

---

## 🔄 양방향 메시지 처리

```js
// Main
worker.postMessage({ name: "홍길동", age: 30 });

// Worker
onmessage = function (e) {
  const { name, age } = e.data;
  postMessage(`안녕하세요, ${name}님. 당신은 ${age}세입니다.`);
};
```

---

## 🚧 제한 사항 및 주의점

| 제한 | 설명 |
|------|------|
| ❌ DOM 접근 불가 | Worker는 document, window 객체에 접근할 수 없습니다. |
| ❌ alert(), confirm() 사용 불가 | 사용자 인터페이스 관련 함수 사용 불가 |
| ✅ XMLHttpRequest, fetch 가능 | 네트워크 요청은 사용 가능 |
| ✅ importScripts() 사용 가능 | 여러 스크립트를 Worker에서 불러올 수 있음 |
| ❗ CORS 제약 있음 | 다른 출처의 스크립트 로딩 시 `CORS` 허용 필요 |

### 예: 워커 내부에서 스크립트 추가

```javascript
importScripts("lib1.js", "lib2.js");
```

---

## 🔐 보안 및 성능

- 모든 데이터는 **복사된 후 전달**됨 (structured clone)
- 큰 데이터를 주고받으면 성능 저하
- WebSocket, IndexedDB, Cache API 등 일부 기능도 사용 가능

> 📌 `SharedWorker`나 `Atomics`, `WebAssembly`와 함께 사용하면 다중 탭 간 공유도 가능

---

## 🧠 요약 정리

| 항목 | 설명 |
|------|------|
| 기술명 | Web Worker API |
| 용도 | 백그라운드에서 JS 실행 (병렬 처리) |
| 주요 메서드 | postMessage, onmessage, terminate |
| 사용 불가 | DOM 조작, alert 등 |
| 브라우저 지원 | 대부분의 최신 브라우저에서 지원 ✅ |

---

## 📚 참고 링크

- [MDN - Web Workers](https://developer.mozilla.org/ko/docs/Web/API/Web_Workers_API)
- [HTML Living Standard - Workers](https://html.spec.whatwg.org/multipage/workers.html)
- [Google Web Fundamentals - Web Workers](https://developers.google.com/web/fundamentals/performance/web-workers)

---

## 🛠️ 확장 주제 추천

- `SharedWorker`와 `Service Worker` 차이점
- `Web Worker` + `Canvas` 활용 예제
- Web Worker를 사용하는 웹 앱 구조 설계
- React/Vue에서 Web Worker 연동 (예: `worker-loader`)
