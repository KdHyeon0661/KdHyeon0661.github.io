---
layout: post
title: HTML - Application Cache
date: 2025-04-18 20:20:23 +0900
category: HTML
---
# Application Cache (AppCache)

## 0. 왜 이 글인가? — 지금도 알아야 하는 실무적 이유

- **레거시 시스템** 유지보수: 오래된 사내 포털/키오스크/내장 브라우저에 AppCache 흔적이 남아 있을 수 있음  
- **마이그레이션**: AppCache에서 **Service Worker**로 옮겨야 할 때 위험 포인트를 미리 파악  
- **오프라인 설계 감각**: AppCache의 실패 사례는 오늘날 PWA 전략 설계에 귀중한 반면교사

---

## 1. AppCache란 무엇이었나?

HTML5 초기에 제안된 오프라인 기술로, **매니페스트(manifest) 파일**에 적힌 리소스를 브라우저가 **일괄 캐시**하여  
네트워크가 없어도 페이지를 열 수 있게 했습니다.

### 동작 개념 요약

1. HTML 문서에 `manifest` 속성을 연결  
2. `.appcache`(또는 `.manifest`) 파일에 **캐시 전략**을 선언  
3. 브라우저가 해당 자산을 **전원-offline 캐시**  
4. 이후 오프라인에서도 캐시된 자원으로 페이지가 열린다

---

## 2. 최소 사용 예제 (역사적 참고용)

> ⚠️ 대부분의 최신 브라우저에서 **더 이상 동작하지 않습니다.**  
> 아래 예제는 “구조 이해”가 목적입니다.

### 2.1 HTML에 manifest 연결

```html
<!DOCTYPE html>
<html lang="ko" manifest="offline.appcache">
<head>
  <meta charset="UTF-8" />
  <title>AppCache Demo (Legacy)</title>
</head>
<body>
  <h1>오프라인에서도 이 페이지는 열립니다!</h1>
  <img src="images/logo.png" alt="로고" />
  <script src="main.js"></script>
</body>
</html>
```

### 2.2 `offline.appcache` (매니페스트)

```text
CACHE MANIFEST
# 버전 토큰 (업데이트 트리거용)
# 2025-11-09T19:00Z

CACHE:
index.html
main.js
styles.css
images/logo.png

NETWORK:
*  # 명시된 자원 외에는 네트워크 시도

FALLBACK:
/ /offline.html
```

> **MIME 타입**: 서버에서 `.appcache`(또는 `.manifest`)를 `text/cache-manifest`로 서빙해야 했습니다.  
> (Apache: `AddType text/cache-manifest .appcache .manifest`)

---

## 3. AppCache 매니페스트 구조와 규칙

| 섹션 | 의미 |
|---|---|
| `CACHE:` | “반드시 캐싱할 파일 목록” |
| `NETWORK:` | “항상 네트워크에서만 불러올 경로(패턴)” |
| `FALLBACK:` | “요청 실패 시 대체할 경로 매핑” (`<원래경로> <대체경로>`) |
| 최상단 라인 | **반드시** `CACHE MANIFEST` 로 시작 |
| 주석 | `#` 로 시작하는 라인, **버전 토큰**으로 활용 (내용 변경 시 업데이트) |

### 자주 겪던 함정
- **암묵적 자동 캐싱**: 문서가 AppCache에 연결되면 의도치 않은 경로가 **암묵적으로 캐시**되는 것처럼 보이는 사례가 다수 보고됨  
- **업데이트 지옥**: 어떤 파일 하나만 바꿔도 **매니페스트 파일의 내용이 변화**해야 전체 캐시가 갱신됨(주석 날짜라도 바꿔야 함)  
- **비직관적인 FALLBACK** 규칙: 경로 매칭과 우선순위가 일관되게 느껴지지 않아 디버깅 난이도↑  
- **동적 앱 부적합**: AJAX·SPA 라우팅·인증 흐름과 **충돌** 빈번

---

## 4. AppCache는 왜 실패했나? (설계적 결함 요약)

| 문제 | 실제로 겪는 증상 |
|---|---|
| 과도한 이항적 전략 | “캐시 or 네트워크”로 단순화되어 **부분 갱신**/정교한 정책이 불가 |
| 예측 불가 갱신 | 빌드 후 일부 파일만 바꿔도 **매니페스트 변경**을 잊으면 사용자에게 영원히 구버전 |
| FALLBACK 모호성 | 라우팅/오류 상황에서 **원치 않는 리소스**로 대체되는 현상 |
| 디버깅 난감 | 이벤트/상태 전이가 복잡하고 브라우저마다 상이 |
| 보안/정책 | 인증 흐름, 권한 제어, 민감 경로 처리에 **일괄 캐시**가 부적합 |

> 결과적으로 표준은 **HTML5.1에서 폐기**, 주요 브라우저는 **제거**했습니다. (IE11만 일부 잔존)

---

## 5. 실무: “망가진 AppCache”에서 **빠져나오기**

레거시 서비스가 아직 **AppCache 잔재** 때문에 캐시가 고착(갱신 불가)된 경우:

### 5.1 AppCache 분리/무력화 전략

1) **HTML에서 `manifest` 속성 제거**  
2) **매니페스트 파일 자체를 404/410**으로 응답 (또는 무해한 빈 파일로 교체)  
3) **서버 캐시 헤더**로 `Cache-Control: no-store`를 일시 부여해 클라이언트 강제 리프레시 유도  
4) 사용자의 강제 새로고침 안내 (DevTools ‘Disable cache’, hard reload)

### 5.2 임시 “해제”용 매니페스트(과거 테크닉)

```text
CACHE MANIFEST
# breaker 2025-11-09

NETWORK:
*
```

- 캐시 목록을 모두 제거하고 `NETWORK:*`만 남겨, 브라우저가 네트워크를 보게 유도  
- 이후 HTML에서 `manifest` 속성을 **완전히 삭제**하고 롤아웃

---

## 6. 정식 대안: **Service Worker + Cache API** (현 표준)

> 목적: **요청 단위로** 캐시·네트워크 정책을 정밀 제어. 오프라인/백그라운드/푸시/동기 등 PWA 기능 제공.  
> 필수: **HTTPS**(또는 `localhost`), 명시적 스코프, 명확한 생명주기(`install`→`activate`→`fetch`).

### 6.1 최소 오프라인 SW 예제

`/sw.js`:
```js
const CACHE_NAME = 'app-v1';
const PRECACHE = [
  '/',            // 앱 셸
  '/index.html',
  '/styles.css',
  '/main.js',
  '/images/logo.png',
  '/offline.html'
];

// 설치: 필수 자원 선캐시
self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open(CACHE_NAME).then(cache => cache.addAll(PRECACHE))
  );
});

// 활성화: 오래된 캐시 제거
self.addEventListener('activate', (event) => {
  event.waitUntil(
    caches.keys().then(keys =>
      Promise.all(keys.map(k => (k === CACHE_NAME ? null : caches.delete(k))))
    )
  );
});

// 요청 가로채기: 네트워크 실패 시 오프라인 폴백
self.addEventListener('fetch', (event) => {
  const { request } = event;
  event.respondWith(
    fetch(request).catch(async () => {
      const cache = await caches.open(CACHE_NAME);
      // 문서 요청이면 offline.html 대체
      if (request.mode === 'navigate') return cache.match('/offline.html');
      // 그 외는 캐시 히트 반환
      const hit = await cache.match(request);
      return hit || Response.error();
    })
  );
});
```

앱에서 등록:

```html
<script>
if ('serviceWorker' in navigator) {
  navigator.serviceWorker.register('/sw.js', { scope: '/' })
    .then(reg => console.log('SW registered', reg))
    .catch(err => console.error('SW failed', err));
}
</script>
```

### 6.2 캐싱 전략 패턴(요청별)

- **Cache First**: 아이콘/폰트/정적 이미지
- **Network First**: HTML 문서/JSON API(오프라인 폴백 포함)
- **Stale-While-Revalidate**: 뉴스피드/목록(빠른 응답 + 배경 갱신)
- **Network Only**: 인증/결제/민감 요청

예: `Network First` with fallback

```js
async function networkFirst(request) {
  const cache = await caches.open('app-v1');
  try {
    const fresh = await fetch(request);
    cache.put(request, fresh.clone());
    return fresh;
  } catch {
    const cached = await cache.match(request);
    return cached || caches.match('/offline.html');
  }
}
self.addEventListener('fetch', (e) => {
  if (e.request.mode === 'navigate') {
    e.respondWith(networkFirst(e.request));
  }
});
```

### 6.3 추천: Workbox (구글 유지 OSS)

복잡한 전략을 **선언형**으로:

```js
// workbox-sw를 번들에 포함했다고 가정
workbox.routing.registerRoute(
  ({request}) => request.destination === 'document',
  new workbox.strategies.NetworkFirst({
    cacheName: 'pages',
    plugins: [new workbox.expiration.ExpirationPlugin({ maxEntries: 50 })]
  })
);

workbox.routing.registerRoute(
  ({request}) => ['style','script','worker'].includes(request.destination),
  new workbox.strategies.StaleWhileRevalidate({ cacheName: 'assets' })
);

workbox.routing.registerRoute(
  ({request}) => request.destination === 'image',
  new workbox.strategies.CacheFirst({
    cacheName: 'images',
    plugins: [new workbox.expiration.ExpirationPlugin({ maxEntries: 100, maxAgeSeconds: 60*60*24*30 })]
  })
);
```

---

## 7. AppCache → Service Worker **마이그레이션 체크리스트**

1. **HTML에서 manifest 제거**  
2. **오프라인 폴백 페이지** 준비: `/offline.html`
3. **사전 캐시 목록** 정의: 앱셸·핵심 자산
4. **요청별 전략** 설계: 문서/정적/이미지/API 구분
5. **버전 네임스페이스**: `CACHE_NAME`에 빌드 해시 반영
6. **런타임 캐시 크기 관리**: 만료/개수 제한
7. **HTTPS 배포 및 SRI/CSP 검토**
8. **탭별 업데이트 흐름**: `waiting` SW를 UI로 교체 안내
9. **오류/네트워크 테스트**: DevTools → Offline/Slow 3G
10. **백/롤백 플랜**: 신규 SW 문제시 즉시 비활성화 로직

---

## 8. 디버깅 & 운영 팁

### 8.1 DevTools (Chrome 기준)
- **Application** 패널 → “Service Workers”:  
  - Update/Skip waiting/Unregister  
- **Cache Storage**: 프리캐시 항목 확인/삭제  
- **Network** 탭: Offline/Throttling으로 시나리오 재현

### 8.2 공용 헤더 권장
- 정적 파일에 `Cache-Control: immutable, max-age=31536000` (파일명 해시 필수)
- HTML 문서에는 `no-cache`(재검증)로 최신 SW 등록 보장

### 8.3 CORS/opaque 대응
제3자 원본을 캐시하면 **opaque 응답**이 될 수 있어 크기/만료 제어가 어렵습니다.  
가능하면 **동일 출처** 또는 **CORS 허용**으로 조정.

---

## 9. 보안 고려사항

- **HTTPS 필수** (Service Worker 등록 요구)  
- **XSS 방어**: SW/캐시에 악성 스크립트가 고착되지 않도록 CSP·SRI 적용  
- **인증/권한**: 민감 API는 **Network Only** + 서버 검증, 오프라인 폴백은 “읽기 전용” 범위로 제한  
- **버전 롤백**: 문제 SW가 배포되면 즉시 `unregister`/이전 버전 재배포

---

## 10. 흔한 Q&A

**Q1. IE11만 사용하는 사내망 키오스크에서 AppCache가 여전히 필요합니까?**  
A1. 기술적으로 동작할 수 있으나, **운영 리스크**(갱신 지연·디버깅 난이도)가 큽니다. 가능하다면 **내장 브라우저 업그레이드 + SW 전환**을 권장.

**Q2. AppCache 파일이 남아서 갱신이 안 됩니다.**  
A2. `manifest` 제거 → 매니페스트 404/410 응답 → 강제 새로고침 유도 → SW로 전환. (중간에 “해제용 매니페스트”를 써서 네트워크로 돌리는 방식도 있음)

**Q3. Service Worker로 바꾸면 성능이 느려지지 않나요?**  
A3. 적절한 전략(문서는 Network First, 정적은 Cache First 등)과 캐시 만료/최적화로 **더 빠른 UX**를 제공할 수 있습니다.

---

## 11. “레거시 AppCache”와 “현대 SW”를 나란히 비교

| 항목 | AppCache | Service Worker + Cache API |
|---|---|---|
| 상태 | **폐기**, 브라우저 제거 | **표준**, 광범위 지원 |
| 제어 | 선언적/경직 | 프로그래밍/정밀 제어 |
| 갱신 | 매니페스트 수정 필수 | 버전명/해시/롤링 업데이트 |
| 전략 | 제한적 | 요청별 맞춤(문서/정적/이미지/API) |
| 디버깅 | 난이도 높음 | DevTools로 명확 |
| 보안 | 설계 취약 | HTTPS/CSP/SRI/스코프 제어 |

---

## 12. 실전 템플릿 — “기본 PWA 오프라인 셸”

### 12.1 파일 구조

```
/public
  index.html
  offline.html
  styles.css
  main.js
  images/logo.png
  sw.js
```

### 12.2 `sw.js` (조금 더 다듬은 버전)

```js
const VERSION = '2025.11.09';
const SHELL_CACHE = `shell-${VERSION}`;
const RUNTIME_CACHE = `runtime-${VERSION}`;

const SHELL_ASSETS = [
  '/',
  '/index.html',
  '/offline.html',
  '/styles.css',
  '/main.js',
  '/images/logo.png'
];

self.addEventListener('install', (e) => {
  e.waitUntil(caches.open(SHELL_CACHE).then(c => c.addAll(SHELL_ASSETS)));
  self.skipWaiting();
});

self.addEventListener('activate', (e) => {
  e.waitUntil(
    caches.keys().then(keys =>
      Promise.all(keys
        .filter(k => ![SHELL_CACHE, RUNTIME_CACHE].includes(k))
        .map(k => caches.delete(k))
      )
    )
  );
  self.clients.claim();
});

self.addEventListener('fetch', (e) => {
  const { request } = e;

  // 문서: Network First → 실패시 offline.html
  if (request.mode === 'navigate') {
    e.respondWith((async () => {
      try {
        const fresh = await fetch(request);
        const cache = await caches.open(SHELL_CACHE);
        cache.put(request, fresh.clone());
        return fresh;
      } catch {
        const cache = await caches.open(SHELL_CACHE);
        return (await cache.match(request)) || cache.match('/offline.html');
      }
    })());
    return;
  }

  // 정적 자산(CSS/JS/이미지): Stale-While-Revalidate
  if (['style','script','image','font'].includes(request.destination)) {
    e.respondWith((async () => {
      const cache = await caches.open(RUNTIME_CACHE);
      const cached = await cache.match(request);
      const fetchPromise = fetch(request).then(resp => {
        cache.put(request, resp.clone());
        return resp;
      }).catch(() => null);
      return cached || fetchPromise || Response.error();
    })());
    return;
  }

  // 그 외: 네트워크 우선, 실패시 캐시 히트
  e.respondWith((async () => {
    try {
      return await fetch(request);
    } catch {
      const cache = await caches.open(RUNTIME_CACHE);
      return (await cache.match(request)) || Response.error();
    }
  })());
});
```

---

## 결론

- AppCache는 **역사의 뒤안길**. 잔재가 보이면 **즉시 해제/정리**하고,  
- **Service Worker + Cache API**로 오프라인·성능·UX를 **요청 단위**로 설계하세요.  
- 빌드 해시/버전·요청별 전략·만료 정책·DevTools/모니터링을 갖춘 **운영 가능한 PWA 캐시 체계**가 현대 표준입니다.