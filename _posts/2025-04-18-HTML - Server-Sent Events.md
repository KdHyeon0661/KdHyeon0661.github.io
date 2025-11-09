---
layout: post
title: HTML - Server-Sent Events
date: 2025-04-18 22:20:23 +0900
category: HTML
---
# Server-Sent Events (SSE)

## 0) SSE 한눈에 보기

- **정의**: 브라우저에서 `EventSource`로 서버에 **하나의 HTTP 연결**을 열고, 서버가 `text/event-stream`으로 **지속적 이벤트를 푸시**하는 **단방향** 실시간 통신.
- **특징**
  - 단방향(서버→클라이언트), 자동 재연결, 가벼운 텍스트 포맷
  - HTTP(1.1/2) 위에서 동작, 프록시/방화벽 친화적
  - 전송은 서버가 push, 클라이언트는 JS로 수신(양방향이 필요하면 WebSocket)
- **주요 용도**: 알림/토스트, 진행률, 로그/대시보드 피드, 주가/센서 업데이트, 스트리밍 LLM 토큰 출력 등

---

## 1) 기본 동작 구조

```
Client --------> Server   : GET /events  (Accept: text/event-stream)
Client <======== Server   : HTTP 200, Content-Type: text/event-stream
                              data: ...
                              
                              data: ...
                              
                              : heartbeat
```

- 연결은 **하나의 장기 지속 응답**. 각 메시지는 **빈 줄(\n\n)** 로 구분.
- 브라우저는 네트워크 끊김/에러 시 **자동 재연결**(기본 수 초) 한다.

---

## 2) 클라이언트: EventSource 기본

```html
<button id="stop">연결 해제</button>
<script>
  // 1) 연결 열기
  const es = new EventSource('/events'); // 같은 오리진

  // 2) 기본 메시지(data: ... 만 있는 메시지)
  es.onmessage = (e) => {
    console.log('MSG:', e.data);
  };

  // 3) 에러/재연결 로깅
  es.onerror = (e) => {
    console.warn('SSE error or reconnecting...', e);
    // 브라우저가 자동으로 재연결을 시도한다.
  };

  // 4) 명명된 커스텀 이벤트
  es.addEventListener('update', (e) => {
    const payload = JSON.parse(e.data);
    console.log('UPDATE:', payload);
  });

  // 5) 수동 종료
  document.getElementById('stop').onclick = () => es.close();
</script>
```

- 다른 오리진이라면 `new EventSource(url, { withCredentials: true })` 로 **쿠키 전송** 가능(CORS 설정 필요).
- HTTP 헤더를 직접 넣을 수는 없으므로 **토큰은 쿼리스트링** 또는 **쿠키**를 쓰는 것이 보편적이다.

---

## 3) 서버: 최소 Express 예제(기본기 + 즉시 flush)

```js
// server.js
const express = require('express');
const app = express();

app.get('/events', (req, res) => {
  // 1) SSE 헤더
  res.setHeader('Content-Type', 'text/event-stream; charset=utf-8');
  res.setHeader('Cache-Control', 'no-cache, no-transform');
  res.setHeader('Connection', 'keep-alive');

  // 2) 프록시 버퍼링 방지(가능하다면)
  res.setHeader('X-Accel-Buffering', 'no'); // Nginx
  // Cloudflare는 "no-transform" 또는 페이지 규칙/대시보드에서 비활성 필요

  // 3) 즉시 첫 줄을 보내 연결을 '사용 중' 상태로 만든다
  res.write(': connected\n\n');

  // 4) 주기적 이벤트
  const timer = setInterval(() => {
    const now = new Date().toISOString();
    res.write(`event: tick\n`);
    res.write(`id: ${Date.now()}\n`);
    res.write(`data: ${JSON.stringify({ now })}\n\n`);
  }, 3000);

  // 5) 연결 종료 시 정리
  req.on('close', () => {
    clearInterval(timer);
  });
});

app.listen(3000, () => console.log('SSE on :3000'));
```

**핵심 포인트**
- `Content-Type: text/event-stream`, `Cache-Control: no-cache` 필수.
- **빈 줄 두 개(\n\n)** 로 메시지 경계.
- **주석 라인(`: comment`)** 을 간헐적으로 보내 **유휴 연결 유지(heartbeat)**.
- 서버는 **`res.write` 후 버퍼 flush** 가 필요할 수 있다(일부 런타임/프록시 설정 참고).

---

## 4) 전송 포맷 상세

메시지는 **여러 필드**로 구성 가능하며, 각 필드는 `key: value` 형식이다.

- `data:`  실제 페이로드(여러 줄 가능, 각 줄에 `data:` 반복)
- `event:`  커스텀 이벤트명(클라이언트가 `addEventListener('name')` 로 수신)
- `id:`  이벤트 ID(재연결 복구에 사용)
- `retry:`  클라이언트 재연결 지연(ms) 제안

예시:

```
id: 1700000000001
event: update
data: {"status":"ok","count":1}

: heartbeat (서버 주석/keepalive)
```

**여러 줄 데이터**

```
data: line 1
data: line 2
data: {"json":"ok"}

```

---

## 5) 재연결/복구 설계 — Last-Event-ID

SSE의 진짜 힘은 **자동 재연결 + 마지막 ID 복구**다.

- 서버가 각 메시지에 `id:` 를 넣는다.
- 브라우저가 끊긴 뒤 재연결 시 **`Last-Event-ID` 헤더**를 자동으로 첨부.
- 서버는 이 ID 이후의 **미전달 메시지**를 **버퍼(큐)** 에서 재전송하여 **손실 없이 복구** 구현 가능.

서버 처리 흐름:

1) 각 연결의 마지막 ID를 파악: `req.headers['last-event-id']`  
2) **메시지 큐(메모리/Redis/DB)** 에서 ID 이후 항목을 찾아 재전송  
3) 실시간으로 이어서 스트리밍

Node(개념):

```js
// 매우 단순화된 큐(실무는 Redis stream/Kafka/DB 사용)
const queue = []; // { id, event, data }

function pushEvent(evt) {
  queue.push(evt);
  if (queue.length > 1000) queue.shift(); // 메모리 한도 관리
}

app.get('/events', (req, res) => {
  // ... 헤더 설정 생략
  const last = req.headers['last-event-id'];

  // 1) 미전달 복구
  if (last) {
    const idx = queue.findIndex(e => e.id === last);
    const pending = idx >= 0 ? queue.slice(idx + 1) : queue;
    pending.forEach(e => {
      res.write(`id: ${e.id}\n`);
      if (e.event) res.write(`event: ${e.event}\n`);
      res.write(`data: ${JSON.stringify(e.data)}\n\n`);
    });
  }

  // 2) 새로운 이벤트 실시간 푸시
  const onNew = (e) => {
    res.write(`id: ${e.id}\n`);
    if (e.event) res.write(`event: ${e.event}\n`);
    res.write(`data: ${JSON.stringify(e.data)}\n\n`);
  };

  emitter.on('event', onNew);
  req.on('close', () => emitter.off('event', onNew));
});
```

브라우저 측은 별도 코드 필요 없이 자동으로 처리된다.

---

## 6) 커스텀 이벤트 수신

서버:

```
event: price
data: {"symbol":"AAPL","price":198.21}

```

클라이언트:

```js
const es = new EventSource('/quotes');
es.addEventListener('price', (e) => {
  const { symbol, price } = JSON.parse(e.data);
  // UI 업데이트
});
```

---

## 7) CORS/인증/보안

### 7.1 CORS

- 단순 GET이지만 **장기 연결**이므로 정확한 헤더가 중요.
- **자격증명(쿠키) 사용 시**:
  - 클라이언트: `new EventSource(url, { withCredentials: true })`
  - 서버:
    - `Access-Control-Allow-Origin: https://your-site.example` (와일드카드 `*` 금지)
    - `Access-Control-Allow-Credentials: true`

Express 예:

```js
app.get('/events', (req, res) => {
  res.setHeader('Access-Control-Allow-Origin', 'https://your-site.example');
  res.setHeader('Access-Control-Allow-Credentials', 'true');
  // ... 나머지 SSE 헤더
});
```

### 7.2 토큰 전달

- EventSource는 커스텀 헤더 설정 불가 → **쿼리스트링**(예: `/events?token=...`) 이나 **쿠키**로 전달.
- 토큰은 **짧은 TTL**과 **서버측 검증** 필수. URL 로그/리퍼러에 노출될 수 있으니 주의.

### 7.3 HTTPS/CSP

- 사용자 데이터가 섞이는 경우 **HTTPS 필수**.
- CSP로 `connect-src` 에 SSE 엔드포인트 도메인을 허용.

---

## 8) 프록시/배포/성능

### 8.1 프록시 버퍼링

- Nginx 등은 기본적으로 응답을 버퍼링 → **SSE가 묶일 수 있음**.  
  Nginx 설정:
  ```
  location /events {
      proxy_pass http://app;
      proxy_set_header Connection '';
      proxy_http_version 1.1;
      chunked_transfer_encoding on;
      proxy_buffering off;           # 중요
      proxy_cache off;
      proxy_read_timeout 1h;         # 장기 연결
      add_header X-Accel-Buffering no;
  }
  ```
- Cloudflare/ELB/Ingress 등도 **buffering/idle timeout** 설정을 조정.

### 8.2 Keep-Alive/Idle Timeout

- PaaS(예: Heroku, 일부 LB)는 **빈 응답 장시간 유지**를 끊을 수 있음 → **주기적 heartbeat** 전송(`: ping\n\n`) 권장.
- HTTP/2 환경에서도 대부분 잘 동작하나, **중간 프록시 호환성**을 확인.

### 8.3 동시 연결/스케일

- 브라우저/오리진당 동시 연결 제한(브라우저 네트워크 정책)을 고려. 페이지당 **SSE 1개 연결**을 권장.
- 서버는 연결 수만큼 파일 디스크립터/메모리/타이머를 소비 → **수평 확장** 필요 시 **메시지 브로커(Redis/Kafka)** 로 분기.

### 8.4 압축

- `text/event-stream` 은 **Content-Length 없이 스트리밍**.  
  일부 프록시는 압축을 비권장/비활성화. 압축이 필요하면 **서버와 프록시 양쪽에서 안전성**을 검증.

---

## 9) 실전 패턴

### 9.1 대시보드(여러 위젯 동시 업데이트)

- 단일 SSE 연결에서 **서로 다른 `event:` 이름**으로 구분 → 위젯별 리스너에 라우팅.

```js
es.addEventListener('cpu', (e) => updateCPU(JSON.parse(e.data)));
es.addEventListener('ram', (e) => updateRAM(JSON.parse(e.data)));
es.addEventListener('jobs', (e) => updateJobs(JSON.parse(e.data)));
```

### 9.2 알림/토스트

- 브라우저 백그라운드 탭 고려: `document.visibilityState` 응용, **토스트 UI**로 비침투적 표시.

### 9.3 스트리밍 진행률/LLM 토큰

- 긴 작업의 중간 상태를 `data:` 로 chunking → 사용자 체감 개선.
- 작업 ID와 `Last-Event-ID` 로 **중단 후 복구** 템플릿.

---

## 10) 다양한 서버 구현

### 10.1 Node.js (Express) — CORS/Heartbeat/Retry

```js
import express from 'express';
const app = express();

app.get('/events', (req, res) => {
  res.set({
    'Content-Type': 'text/event-stream; charset=utf-8',
    'Cache-Control': 'no-cache, no-transform',
    'Connection': 'keep-alive',
    'Access-Control-Allow-Origin': 'https://your-site.example',
    'Access-Control-Allow-Credentials': 'true',
    'X-Accel-Buffering': 'no',
  });

  // 재연결 주기 제안(클라가 채택)
  res.write('retry: 3000\n\n');
  res.write(': hello\n\n'); // heartbeat

  const interval = setInterval(() => {
    const payload = { ts: Date.now(), value: Math.random() };
    res.write(`id: ${payload.ts}\n`);
    res.write('event: metric\n');
    res.write(`data: ${JSON.stringify(payload)}\n\n`);
  }, 2000);

  req.on('close', () => clearInterval(interval));
});

app.listen(3000);
```

### 10.2 Python — FastAPI(ASGI) / StreamingResponse

```python
# pip install fastapi uvicorn
from fastapi import FastAPI, Request
from fastapi.responses import StreamingResponse
import asyncio, json, time

app = FastAPI()

async def event_generator():
    yield ": connected\n\n"
    while True:
        await asyncio.sleep(2)
        payload = {"ts": int(time.time()*1000), "value": 42}
        yield f"id: {payload['ts']}\n"
        yield "event: metric\n"
        yield f"data: {json.dumps(payload)}\n\n"

@app.get("/events")
async def events(request: Request):
    async def stream():
        async for chunk in event_generator():
            # 클라이언트가 끊으면 중단
            if await request.is_disconnected():
                break
            yield chunk
    return StreamingResponse(stream(), media_type="text/event-stream")
```

### 10.3 Java — Spring Boot (SseEmitter)

```java
// build.gradle: implementation 'org.springframework.boot:spring-boot-starter-web'
@RestController
public class SseController {
  @GetMapping(value="/events", produces=MediaType.TEXT_EVENT_STREAM_VALUE)
  public SseEmitter events() {
    SseEmitter emitter = new SseEmitter(0L); // 무제한
    ScheduledExecutorService exec = Executors.newSingleThreadScheduledExecutor();
    exec.scheduleAtFixedRate(() -> {
      try {
        Map<String,Object> payload = Map.of("ts", System.currentTimeMillis());
        emitter.send(SseEmitter.event().id(String.valueOf(payload.get("ts")))
                               .name("tick")
                               .data(payload));
      } catch (Exception ex) {
        emitter.complete();
      }
    }, 0, 2, TimeUnit.SECONDS);
    emitter.onCompletion(exec::shutdown);
    emitter.onTimeout(() -> { emitter.complete(); exec.shutdown(); });
    return emitter;
  }
}
```

### 10.4 Go — net/http + Flusher

```go
package main

import (
  "encoding/json"
  "fmt"
  "log"
  "net/http"
  "time"
)

func events(w http.ResponseWriter, r *http.Request) {
  w.Header().Set("Content-Type", "text/event-stream")
  w.Header().Set("Cache-Control", "no-cache")
  w.Header().Set("Connection", "keep-alive")
  w.Header().Set("X-Accel-Buffering", "no")

  flusher, ok := w.(http.Flusher)
  if !ok { http.Error(w, "Streaming unsupported", http.StatusInternalServerError); return }

  fmt.Fprintf(w, ": connected\n\n")
  flusher.Flush()

  ticker := time.NewTicker(2 * time.Second)
  defer ticker.Stop()

  for {
    select {
    case t := <-ticker.C:
      payload := map[string]any{"ts": t.UnixMilli()}
      b, _ := json.Marshal(payload)
      fmt.Fprintf(w, "id: %d\n", t.UnixMilli())
      fmt.Fprintf(w, "event: tick\n")
      fmt.Fprintf(w, "data: %s\n\n", b)
      flusher.Flush()
    case <-r.Context().Done():
      return
    }
  }
}

func main() {
  http.HandleFunc("/events", events)
  log.Fatal(http.ListenAndServe(":3000", nil))
}
```

---

## 11) React 클라이언트 미니 패턴(자동 정리 포함)

```jsx
import { useEffect, useRef, useState } from 'react';

export default function UseSSE() {
  const [rows, setRows] = useState([]);
  const esRef = useRef(null);

  useEffect(() => {
    const es = new EventSource('/events');
    es.addEventListener('metric', (e) => {
      setRows((prev) => [JSON.parse(e.data), ...prev].slice(0, 50));
    });
    es.onerror = () => { /* 브라우저가 자동 재연결 */ };
    esRef.current = es;
    return () => es.close();
  }, []);

  return (
    <ul>
      {rows.map((r, i) => <li key={i}>{r.ts} - {r.value}</li>)}
    </ul>
  );
}
```

---

## 12) 파일 업로드/명령 전송 등 “역방향”이 필요할 때

- SSE는 **단방향**. 클라이언트→서버 요청은 **기존 HTTP(Fetch/XHR)** 를 병행한다.
- 예) “실행 버튼 클릭 → POST /run → 진행률은 SSE로 수신” 같은 **CQRS 패턴**이 깔끔하다.

---

## 13) 문제 해결 체크리스트

| 증상 | 점검 항목 |
|---|---|
| 메시지가 몰아서 도착 | 프록시/서버 **버퍼링 OFF**, `X-Accel-Buffering: no`, `proxy_buffering off` |
| 수 분 후 끊김 | LB/프록시 **idle timeout** 증가, **heartbeat** 주기 전송 |
| CORS 에러 | `Access-Control-Allow-Origin`/`Allow-Credentials` 설정, `withCredentials` 옵션 확인 |
| 인증 필요 | 쿠키 또는 쿼리 토큰, 토큰 TTL/회수 전략 |
| 재연결 시 유실 | `id:` 와 **Last-Event-ID 복구 루틴**, 메시지 큐(영속 저장) |
| 모바일 배터리 | 업데이트 주기/데이터 최소화, 필요 시 연결을 늦게 열기 |
| 탭 다중 연결 | 페이지당 **하나의 EventSource** 공유, BroadcastChannel/SharedWorker 고려 |

---

## 14) WebSocket vs SSE — 선택 가이드

- **SSE**: 서버→클라이언트 푸시만 필요, 구현 단순/저비용, HTTP 친화, 자동 재연결/복구 쉬움
- **WebSocket**: 채팅/게임/협업 편집처럼 **양방향/저지연** 필요할 때
- “대부분의 알림/진행률/대시보드”는 **SSE가 더 간단하고 충분**하다.

---

## 15) 미니 실전: “배치 작업 진행률 + 로그 스트림”

### 흐름
1) 사용자가 “배치 실행” 클릭 → `POST /jobs` → jobId 반환  
2) 클라가 `EventSource(/jobs/:id/stream)` 연결  
3) 서버는 상태/로그를 `event: progress`, `event: log` 로 전송  
4) 끊겼다가 재연결 시 `Last-Event-ID` 로 **로그 유실 없이 복구**

서버 전송 예:

```
id: 1001
event: progress
data: {"jobId":"abc","percent":10}

id: 1002
event: log
data: "step 1 done"

```

클라:

```js
const es = new EventSource(`/jobs/${jobId}/stream`);
es.addEventListener('progress', e => updateBar(JSON.parse(e.data)));
es.addEventListener('log', e => appendLog(e.data));
```

---

## 16) 정리

| 항목 | 내용 |
|---|---|
| API | `EventSource` (클라이언트), `text/event-stream` (서버) |
| 강점 | 단방향 스트리밍, 자동 재연결/복구(Last-Event-ID), HTTP 친화, 구현 간단 |
| 설계 핵심 | `id:`/큐 기반 복구, heartbeat, 프록시 버퍼링 해제, CORS/인증, 리소스 관리 |
| 대안 | WebSocket(양방향), Long-Polling(대체), Pusher/SSE 게이트웨이(매니지드) |

---

## 17) 추가 참고

- MDN: Server-sent events  
- HTML Living Standard: EventSource  
- Can I use: EventSource  
- Nginx proxy buffering docs, Cloudflare streaming guidance

이상으로, **SSE의 기초→실전 배포**까지 한 번에 정리했습니다. 실무에선 **하나의 연결, 명명 이벤트, ID/큐 복구, 프록시 설정** 네 가지를 먼저 체크하면 대부분의 문제를 예방할 수 있습니다.