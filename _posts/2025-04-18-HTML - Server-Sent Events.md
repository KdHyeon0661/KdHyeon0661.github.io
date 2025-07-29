---
layout: post
title: HTML - Server-Sent Events
date: 2025-04-18 22:20:23 +0900
category: HTML
---
# 🔁 Server-Sent Events (SSE) 완전 가이드

## ✅ Server-Sent Events란?

**Server-Sent Events (SSE)**는 HTML5에서 새롭게 도입된 **단방향 실시간 통신 기술**입니다.  
브라우저(클라이언트)가 서버에 연결하면, **서버가 클라이언트에 지속적으로 데이터를 푸시(push)**할 수 있습니다.

> 주로 실시간 뉴스, 주식 시세, 알림, 대시보드 등에 사용됩니다.

---

## 🔁 단방향 통신 구조

SSE는 **HTTP 기반의 단방향 통신 프로토콜**로, 다음과 같은 흐름으로 작동합니다:

```
Client --------> Server   : HTTP 요청
Client <======== Server   : 지속적인 이벤트 스트림 (text/event-stream)
```

- **브라우저 → 서버** : 단 한 번의 HTTP 요청
- **서버 → 브라우저** : 실시간으로 여러 메시지 전송 가능

---

## 🔧 SSE의 특징

| 항목 | 내용 |
|------|------|
| ✅ 단방향 푸시 | 서버 → 클라이언트 방향만 가능 |
| ✅ HTTP 기반 | 별도 프로토콜 X (기존 HTTP 사용) |
| ✅ 자동 재연결 | 네트워크 끊김 시 자동 복구 |
| ✅ 헤더 설정 간단 | Content-Type: `text/event-stream` |
| ✅ 가볍고 설정이 쉬움 | WebSocket보다 구현 단순 |
| ❌ 브라우저 단일 연결 | 클라이언트당 1개 연결 권장 |
| ❌ 보안 제한 | HTTP/2 미지원, 프록시 통제 어려움

---

## 📦 기본 사용법

### 1️⃣ 클라이언트 (JavaScript)

```javascript
const evtSource = new EventSource("/events");

evtSource.onmessage = function(event) {
  console.log("서버로부터 메시지:", event.data);
};

evtSource.onerror = function(err) {
  console.error("SSE 오류 발생:", err);
};
```

### 2️⃣ 서버 (Node.js 예시)

```js
const express = require("express");
const app = express();

app.get("/events", (req, res) => {
  res.setHeader("Content-Type", "text/event-stream");
  res.setHeader("Cache-Control", "no-cache");
  res.setHeader("Connection", "keep-alive");

  // 3초마다 서버가 메시지 보냄
  setInterval(() => {
    res.write(`data: 현재 시간: ${new Date().toLocaleTimeString()}\n\n`);
  }, 3000);
});

app.listen(3000, () => console.log("SSE 서버 실행 중"));
```

---

## 📋 전송 포맷

SSE는 텍스트 기반 포맷으로 다음과 같은 규칙을 따릅니다:

```text
data: 메시지 내용
data: 여러 줄도 가능
event: customEvent
id: 고유 ID
retry: 재연결 대기 시간(ms)

\n\n ← 메시지 종료
```

예시:

```text
data: Hello World!
id: 123
retry: 5000

data: 두 번째 메시지입니다.
```

---

## 🎯 커스텀 이벤트 예시

### 서버

```text
event: update
data: {"status": "ok"}
```

### 클라이언트

```javascript
evtSource.addEventListener("update", function(e) {
  const json = JSON.parse(e.data);
  console.log("업데이트 이벤트:", json.status);
});
```

---

## 🔄 WebSocket vs SSE

| 항목 | Server-Sent Events | WebSocket |
|------|--------------------|-----------|
| 통신 방식 | 단방향 (서버 → 클라이언트) | 양방향 |
| 기반 프로토콜 | HTTP (text/event-stream) | 새로운 프로토콜 (ws://) |
| 구현 난이도 | 매우 쉬움 | 다소 복잡 |
| 사용 목적 | 실시간 알림, 피드 등 | 채팅, 게임 등 양방향 통신 |
| 재연결 | 자동 지원 | 직접 구현 필요 |
| 브라우저 지원 | 일부 IE 제외 전부 지원 | 거의 모든 최신 브라우저 지원 |

---

## 🌐 브라우저 지원

| 브라우저 | 지원 여부 |
|----------|------------|
| Chrome | ✅ |
| Firefox | ✅ |
| Safari | ✅ |
| Edge | ✅ |
| IE 10 이하 | ❌ 미지원 |
| Android/iOS WebView | ✅ 대부분 지원 (주의 필요) |

> 👉 [Can I Use - EventSource](https://caniuse.com/eventsource)

---

## 🚧 주의 사항

- **CORS 설정** 필요: 다른 도메인에서 SSE 수신 시 서버에서 CORS 허용
- **보안 연결**은 반드시 `HTTPS` 사용
- **서버 리소스 사용 주의**: 클라이언트 수에 따라 서버 부하 증가
- **헤더 설정 필수**: `Content-Type: text/event-stream`, `Connection: keep-alive`

---

## 💡 실전 활용 예

| 활용 사례 | 설명 |
|-----------|------|
| 실시간 알림 | SNS, 메시지 알림 표시 |
| 주식/코인 시세 | 1초마다 실시간 갱신 |
| 대시보드 | 서버 상태, 센서값 실시간 표시 |
| 게임 상태 표시 | 점수판, 시간 정보 등 일방향 푸시
| 뉴스 티커 | 웹사이트 상단 실시간 뉴스 스트림

---

## 🧠 요약

| 항목 | 내용 |
|------|------|
| API | `EventSource` 객체 사용 |
| 방식 | 서버 → 클라이언트 단방향 푸시 |
| 포맷 | text/event-stream (헤더 필수) |
| 특징 | 자동 재연결, 브라우저 내장, 간편 |
| 대안 | WebSocket, Long Polling, MQTT 등 |

---

## 📚 참고 링크

- [MDN - Server-Sent Events](https://developer.mozilla.org/ko/docs/Web/API/Server-sent_events)
- [HTML Living Standard - EventSource](https://html.spec.whatwg.org/multipage/server-sent-events.html)
- [W3Schools - Server-Sent Events](https://www.w3schools.com/html/html5_serversentevents.asp)