---
layout: post
title: CSS - DevTools
date: 2025-05-20 19:20:23 +0900
category: CSS
---
# 🧪 디버깅을 위한 DevTools(개발자 도구) 활용 가이드

웹 개발에서 **디버깅(Debugging)**은 필수 과정이며, 이를 도와주는 강력한 도구가 바로 브라우저의 **DevTools (개발자 도구)**입니다.  
Chrome, Edge, Firefox 등 주요 브라우저에서 기본으로 제공되며, **HTML/CSS/JavaScript의 실시간 점검 및 수정**이 가능합니다.

이 글에서는 **DevTools의 주요 패널을 설명하고**, 디버깅과 성능 분석에 실질적으로 어떻게 활용할 수 있는지를 구체적으로 안내합니다.

---

## 🧭 DevTools 열기

- **Chrome/Edge 기준 단축키**
  - `F12`
  - `Ctrl + Shift + I` (Windows)
  - `Cmd + Option + I` (Mac)
- 우클릭 → “검사(Inspect)” 선택

---

## 🧱 1. **Elements 패널**

> DOM 구조 & CSS 확인 및 수정

### 주요 기능

- **DOM 트리 구조 실시간 확인 / 편집**
- **CSS 스타일 실시간 수정**
- **Box Model 시각화**
- **클래스 토글 / 강제 상태(:hover, :focus 등) 적용**

### 예제

```html
<div class="card">Hello</div>
```

- `.card`에 적용된 스타일을 확인하고, 즉석에서 `background-color` 수정 가능
- 클래스 이름을 더블 클릭하여 동적으로 변경 가능

---

## 🎨 2. **Styles / Computed / Layout**

- **Styles**: 요소에 적용된 CSS 규칙 확인 및 수정
- **Computed**: 최종 계산된 값 확인 (e.g., margin, color)
- **Layout** (Chrome): Grid / Flexbox 레이아웃 시각화 지원

> 💡 CSS 충돌이나 상속 이슈를 분석할 때 매우 유용

---

## 🐞 3. **Console 패널**

> 자바스크립트 디버깅과 로깅에 사용

### 기능

- `console.log()` 출력 확인
- 에러/경고 메시지 확인
- JS 표현식 직접 실행
- DOM 객체 접근 (`$0`, `$1`, `document.querySelector` 등)

### 팁

```js
console.log('디버그 중');
console.dir(myObject);           // 객체 구조 트리 보기
console.table(myArray);          // 테이블 형태로 배열 출력
console.time('test'); // ... code ...
console.timeEnd('test');
```

---

## 🧠 4. **Sources 패널**

> JS 파일 구조 확인, **중단점(Breakpoint)** 설정, **실행 흐름 추적**

### 주요 기능

- 파일 트리 탐색
- **중단점(Breakpoint)** 설정 → 코드 실행 중단 후 상태 확인
- Watch Expressions, Call Stack, Scope 보기

### 예제

```js
function calc(a, b) {
  let result = a + b;
  debugger; // 실행 중단
  return result;
}
```

### 브레이크포인트 예시

- 특정 줄에서 중단
- 이벤트 브레이크포인트 (`click`, `keydown` 등)
- DOM 변화 감지 중단점 (Attributes, Node 변경 등)

---

## 📡 5. **Network 패널**

> API 요청/응답, 리소스 로딩 상황 분석

### 주요 기능

- HTTP 요청 (XHR, fetch 등) 확인
- 응답 코드 및 Body, Header, Params 보기
- 로딩 시간 및 용량 분석
- 요청 필터링 (`JS`, `XHR`, `Font` 등)

### 실전 사용

- API 호출 실패 확인
- 요청이 몇 초 걸리는지 분석
- 이미지 또는 폰트 로딩 실패 디버그

---

## 🧮 6. **Performance 패널**

> 렌더링, 레이아웃, 페인팅 비용 시각화 → **성능 병목 식별**

### 사용법

1. Record 클릭
2. 페이지 조작 or 스크롤
3. Stop → 분석 결과 보기

- Recalculate Style, Layout, Paint, Composite 등 항목 확인
- Main Thread 블로킹 분석

---

## 📊 7. **Lighthouse / Core Web Vitals**

> 접근성, SEO, 성능 점수 분석 도구

- `Lighthouse` 탭 또는 `Web Vitals` 확장으로 사용
- 실제 사용자 환경에 가까운 지표 제공

### 주요 지표

- FCP: First Contentful Paint
- LCP: Largest Contentful Paint
- CLS: Cumulative Layout Shift
- TTI: Time To Interactive

---

## 📦 8. **Application 패널**

> 웹 스토리지, 쿠키, 캐시, PWA 관리

### 기능

- **LocalStorage, SessionStorage, Cookie** 데이터 확인/삭제
- IndexedDB, CacheStorage 등 서비스워커 리소스 점검
- Service Worker 등록 여부 확인

```js
localStorage.setItem('token', 'abc123');
```

---

## 🧑‍🔧 실전 디버깅 시나리오 예시

| 문제 | 활용 도구 | 해결 방법 |
|------|------------|-----------|
| CSS 적용 안 됨 | Elements / Styles | 우선순위, 상속, !important 여부 확인 |
| 버튼 클릭 시 동작 안함 | Console / Sources | 이벤트 연결 확인, 브레이크포인트 설정 |
| API 응답 오류 | Network | 응답 코드, 헤더, 응답 Body 확인 |
| 레이아웃 깨짐 | Layout / Computed | Flex/Grid 시각화, BoxModel 확인 |
| 느린 로딩 | Performance / Network | 렌더링 단계 분석, 리소스 지연 원인 파악 |

---

## 📌 유용한 DevTools 팁

| 팁 | 설명 |
|-----|------|
| `$0` | Elements에서 선택한 요소를 JS에서 참조 |
| `Ctrl + Shift + C` | 화면 요소 선택 모드 |
| `Preserve Log` | 페이지 리로드해도 로그 유지 |
| `Disable Cache` | 네트워크 캐시 끄고 테스트 |
| `Local Overrides` | DevTools에서 파일 변경 후 저장/적용 가능 (실험적 기능) |

---

## 🔗 참고 링크

- [Chrome DevTools 공식 문서](https://developer.chrome.com/docs/devtools/)
- [Google 웹 개발자 도구 소개](https://developers.google.com/web/tools/chrome-devtools)
- [FireFox 개발자 도구 소개](https://firefox-source-docs.mozilla.org/devtools-user/index.html)

---

## ✅ 요약

| 패널        | 주요 기능                      |
|-------------|-------------------------------|
| Elements    | DOM 구조 확인, CSS 수정        |
| Console     | JS 오류 및 로그 확인           |
| Sources     | 중단점 디버깅, 코드 흐름 추적  |
| Network     | API 요청/응답, 속도 분석       |
| Performance | 렌더링/레이아웃 병목 분석      |
| Application | 웹 저장소, 쿠키, 캐시 등 관리 |