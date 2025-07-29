---
layout: post
title: HTML - Application Cache
date: 2025-04-18 20:20:23 +0900
category: HTML
---
# 🗂️ Application Cache (AppCache) 완전 정리

## ✅ AppCache란?

**Application Cache**는 HTML5에서 처음 도입된 기술로,  
웹 애플리케이션을 **오프라인에서도 사용할 수 있도록** 해주는 기능입니다.

> 한 번 방문한 웹 페이지를 캐시에 저장하여, **인터넷 연결이 없어도** 웹 애플리케이션이 작동하도록 만들어줍니다.

---

## 📦 기본 개념

AppCache는 **manifest 파일**을 기반으로 작동하며,  
브라우저는 이 파일을 통해 어떤 자원을 캐시할지 결정합니다.

### 흐름 요약:

1. HTML 문서에 `manifest` 속성 추가  
2. `.appcache` 파일 작성  
3. 브라우저가 해당 리소스를 캐시  
4. 이후에는 오프라인에서도 접속 가능

---

## 🧱 사용 방법

### 1️⃣ HTML 문서에 manifest 속성 추가

```html
<!DOCTYPE html>
<html manifest="example.appcache">
```

### 2️⃣ `.appcache` 파일 작성

```text
CACHE MANIFEST

# 주석

CACHE:
index.html
style.css
script.js
images/logo.png

NETWORK:
*

FALLBACK:
/offline.html
```

---

## 🧾 AppCache 매니페스트 구조

| 섹션 | 설명 |
|------|------|
| `CACHE:` | 오프라인에서도 사용할 리소스 명시 |
| `NETWORK:` | 온라인에서만 접근해야 하는 리소스 (`*` = 모든 요청은 네트워크 시도) |
| `FALLBACK:` | 요청 실패 시 대체할 리소스 지정 |

---

## 🧪 간단 예제

### `index.html`

```html
<!DOCTYPE html>
<html manifest="offline.appcache">
<head>
  <meta charset="UTF-8">
  <title>Offline Test</title>
</head>
<body>
  <h1>오프라인에서도 이 페이지는 열립니다!</h1>
  <img src="logo.png" alt="로고" />
</body>
</html>
```

### `offline.appcache`

```text
CACHE MANIFEST

CACHE:
index.html
logo.png

NETWORK:
*
```

---

## ✅ AppCache의 장점

- ✅ 오프라인 환경에서도 웹 사용 가능
- ✅ 캐시 자동 처리
- ✅ 초기 HTML만 수정하면 적용 가능

---

## ❌ AppCache의 심각한 문제점

| 문제 | 설명 |
|------|------|
| ⚠️ 과도한 자동 캐싱 | 모든 자원이 자동으로 캐싱됨 → 원하지 않는 리소스까지 저장됨 |
| ⚠️ 캐시 갱신이 어려움 | 파일 하나만 변경해도 전체 캐시 무효화 필요 |
| ⚠️ 에러 처리가 어렵고 불안정함 | 예외 처리 및 디버깅이 매우 까다로움 |
| ⚠️ 복잡한 동적 애플리케이션과 부적합 | SPA, AJAX 기반 앱과 충돌 잦음 |

---

## 📉 AppCache는 폐기(deprecated)되었습니다

**AppCache는 공식적으로 HTML 표준에서 제거되었으며, 더 이상 사용을 권장하지 않습니다.**

| 브라우저 | 지원 현황 |
|----------|------------|
| Chrome | ❌ 제거됨 |
| Firefox | ❌ 제거됨 |
| Edge | ❌ 제거됨 |
| Safari | ❌ 제거 예정 |
| IE 11 | ✅ (하지만 비추천) |

> 📢 HTML5.1부터 AppCache는 폐기(deprecated)되었고,  
> 최신 브라우저에서는 **더 이상 작동하지 않습니다.**

---

## 🔁 대체 기술: Service Worker

AppCache의 대체 기술은 바로 **Service Worker + Cache API** 조합입니다.

| 비교 항목 | AppCache | Service Worker |
|-----------|----------|----------------|
| 유연성 | 낮음 | 매우 높음 |
| 제어 가능성 | 자동 캐싱 | 프로그래밍으로 정밀 제어 |
| 지원 상태 | 폐기됨 ❌ | 표준 ✅ |
| 실무 활용 | 거의 없음 | 대부분의 PWA에서 사용됨 |

> Service Worker는 백그라운드에서 동작하며, 요청 가로채기, 캐시 관리, 푸시 알림 등 다양한 기능을 수행합니다.

---

## 🧠 요약 정리

| 항목 | 설명 |
|------|------|
| 기술명 | Application Cache |
| 목적 | 웹 앱 오프라인 지원 |
| 사용 방식 | manifest 파일로 캐시 리소스 정의 |
| 장점 | 구현이 단순 |
| 단점 | 캐시 관리 미비, 예외 처리 어려움 |
| 상태 | ❌ HTML5.1부터 폐기됨 |
| 대안 | ✅ Service Worker + Cache API

---

## 📚 참고 자료

- [MDN - AppCache](https://developer.mozilla.org/en-US/docs/Web/HTML/Using_the_application_cache)
- [AppCache is a Douchebag](https://alistapart.com/article/application-cache-is-a-douchebag/)
- [Service Worker 공식 문서 (MDN)](https://developer.mozilla.org/ko/docs/Web/API/Service_Worker_API)