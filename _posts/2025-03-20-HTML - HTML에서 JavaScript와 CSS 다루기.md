---
layout: post
title: HTML - HTML에서 JavaScript와 CSS 다루기
date: 2025-03-20 19:20:23 +0900
category: HTML
---
# HTML에서 JavaScript와 CSS 다루기

## CSS — HTML에 스타일을 더하는 3가지 기본 방법

### 인라인 스타일(Inline)

```html
<p style="color: red; font-size: 18px;">빨간 텍스트</p>
```
- 장점: 아주 빠른 적용/테스트.
- 단점: **재사용/유지보수/검색성** 최악, **특이성(specificity)** 높아 후속 재정의 어렵다.
- 권장도: **테스트/임시** 외 비권장. 추후 외부 CSS로 이관.

---

### 내부 스타일(Internal)

```html
<head>
  <style>
    h1 { color: navy; background-color: #eee; }
    p  { color: #555; line-height: 1.7; }
  </style>
</head>
```
- 장점: 문서 단위로 간단한 스타일을 즉시 포함.
- 단점: 여러 페이지에서 **중복**될 수 있음.
- 쓰임새: 단일 페이지 샘플, 문서 전용 스타일.

---

### 외부 스타일(External) — 권장

```html
<!-- head 내부 -->
<link rel="stylesheet" href="/assets/style.css">
```
```css
/* /assets/style.css */
:root {
  --brand: #0e7afe; /* CSS 변수 */
}
h1 { color: var(--brand); }
```
- 장점: **캐시**, **재사용**, **빌드/검증 파이프라인**(Lint/Minify/Prefix) 적용 가능.
- 대규모/실무 권장 기본형.

---

## CSS를 “어디에” 둘 것인가 — 위치와 성능

- `<link rel="stylesheet">`는 **반드시 `<head>`**에.
  브라우저는 CSS를 **렌더 차단** 리소스로 다루므로, head 초반에 로드해야 **FOUC(깜빡임)** 최소화.
- `@import`는 **비권장**: 추가 네트워크 홉으로 느려질 수 있음.

### 핵심 성능 패턴(초기 페인트 최적화)

```html
<!-- 핵심 above-the-fold 스타일 일부를 인라인(크리티컬 CSS) -->
<style>/* …필요 최소한의 핵심 스타일… */</style>

<!-- 나머지 큰 CSS는 preload + onload 비차단 로드 -->
<link rel="preload" as="style" href="/assets/app.css">
<link rel="stylesheet" href="/assets/app.css" media="print" onload="this.media='all'">
<noscript><link rel="stylesheet" href="/assets/app.css"></noscript>
```
- 작은 핵심 스타일만 인라인 → 초기 렌더 가속.
- 대용량 CSS는 **preload** 후 **비차단 로드**(onload) → 사용자 체감 속도 개선.

---

## JavaScript — HTML에 “동작”을 더하는 3가지 방법

### 인라인 스크립트(이벤트 속성) — 비권장

```html
<button onclick="alert('클릭!')">클릭</button>
```
- 장점: 빠른 데모.
- 단점: 유지보수/보안(CSP) 측면에서 불리. **분리 원칙** 위배.

---

### 내부 스크립트(Internal)

```html
<script>
  document.addEventListener('DOMContentLoaded', () => {
    console.log('DOM 준비 완료');
  });
</script>
```
- 소규모 페이지나 실험 코드에 유용.

---

### 외부 스크립트(External) — 권장

```html
<script src="/assets/main.js" defer></script>
```
```js
// /assets/main.js
document.addEventListener('DOMContentLoaded', () => {
  const btn = document.querySelector('#btn');
  btn?.addEventListener('click', () => alert('버튼 클릭!'));
});
```
- 장점: **캐시·빌드·테스트** 파이프라인 적용, 코드 분리로 유지보수 용이.

---

## 스크립트 “위치”와 로드 전략 — defer vs async vs module

| 속성/위치 | 실행 타이밍 | 의존성/순서 | 사용처 |
|---|---|---|---|
| `<body>` 끝 | DOM 파싱 후 | 삽입 순서 | 레거시/간단 페이지 |
| `defer` | **DOM 파싱과 병렬 다운로드**, 파싱 완료 후 **순서대로** 실행 | 보장 | **일반 앱 권장** |
| `async` | 다운로드 끝나는 즉시 실행(파싱 중단) | **불가** | 광고/분석 스크립트 |
| `type="module"` | 모듈 의존성 자동 로드, **defer처럼** 실행 | 모듈 그래프 순서 | 현대적 앱(ESM) |

### 예시

```html
<!-- 권장: 의존성 있는 앱 코드 -->
<script src="/assets/vendor.js" defer></script>
<script src="/assets/app.js" defer></script>

<!-- ESM: 더 선호(브라우저 기본 모듈) -->
<script type="module">
  import { boot } from '/assets/app.mjs';
  boot();
</script>

<!-- analytics는 async -->
<script src="https://analytics.example.com/sdk.js" async></script>
```

---

## 통합 예제 — HTML + CSS + JS (접근성/성능/보안 포함)

```html
<!DOCTYPE html>
<html lang="ko">
<head>
  <meta charset="UTF-8">
  <title>HTML + CSS + JS 통합 예제</title>
  <meta name="viewport" content="width=device-width, initial-scale=1">

  <!-- 크리티컬 CSS(필수 최소 스타일) -->
  <style>
    :root {
      --brand: #0e7afe;
      --bg: #ffffff;
      --fg: #1b1b1b;
    }
    @media (prefers-color-scheme: dark) {
      :root { --bg: #0e0e0e; --fg: #e8e8e8; }
    }
    body { margin:0; background:var(--bg); color:var(--fg); font:16px/1.7 system-ui, sans-serif; }
    header { padding:1rem; border-bottom:1px solid #ddd; }
    .btn { display:inline-block; padding:.6rem 1rem; background:var(--brand); color:#fff; border-radius:8px; border:0; cursor:pointer; }
    .sr-only { position:absolute; left:-10000px; width:1px; height:1px; overflow:hidden; }
  </style>

  <!-- 대용량 CSS를 비차단 로드 -->
  <link rel="preload" as="style" href="/assets/app.css">
  <link rel="stylesheet" href="/assets/app.css" media="print" onload="this.media='all'">
  <noscript><link rel="stylesheet" href="/assets/app.css"></noscript>

  <!-- CSP를 사용하는 경우 인라인 스크립트 금지하고 nonce/해시 사용 권장(아래 보안 섹션 참고) -->
</head>
<body>
  <a href="#main" class="sr-only">본문 바로가기</a>
  <header>
    <h1>통합 데모</h1>
  </header>

  <main id="main" tabindex="-1">
    <h2>이벤트 버튼</h2>
    <button id="btn" class="btn" type="button" aria-describedby="btnHelp">눌러보세요</button>
    <p id="btnHelp">버튼을 누르면 안내 메시지가 나타납니다.</p>

    <section aria-labelledby="theme">
      <h3 id="theme">테마 전환</h3>
      <button id="toggle" class="btn" type="button">다크/라이트 토글</button>
    </section>
  </main>

  <!-- 권장: 외부 스크립트 + defer -->
  <script src="/assets/main.js" defer></script>
</body>
</html>
```

```js
// /assets/main.js
(function () {
  // DOM 준비 후 실행(스크립트가 defer이므로 DOMContentLoaded 직후 실행 보장)
  const btn = document.getElementById('btn');
  btn?.addEventListener('click', () => {
    alert('버튼을 눌렀습니다!');
  });

  // prefers-color-scheme 기반 토글(사용자 선택은 CSS 변수로 저장)
  const toggle = document.getElementById('toggle');
  toggle?.addEventListener('click', () => {
    const dark = document.documentElement.classList.toggle('dark');
    // CSS에서 :root.dark { --bg: ... } 등으로 제어
    localStorage.setItem('theme', dark ? 'dark' : 'light');
  });

  // 초기 테마 적용
  const saved = localStorage.getItem('theme');
  if (saved === 'dark') document.documentElement.classList.add('dark');
})();
```

```css
/* /assets/app.css — 비핵심(나중 로드) 스타일 */
@layer reset, base, components, utilities;

/* 1) reset */
@layer reset {
  *,*::before,*::after { box-sizing: border-box; }
}

/* 2) base */
@layer base {
  :root.dark { --bg: #0e0e0e; --fg: #e8e8e8; }
  a { color: var(--brand); text-decoration: none; }
  a:hover { text-decoration: underline; }
}

/* 3) components (BEM 예시) */
@layer components {
  .card { background:#f6f6f6; padding:1rem; border-radius:12px; }
  .dark .card { background:#181818; }
  .btn--ghost { background:transparent; color:var(--brand); border:1px solid var(--brand); }
}

/* 4) utilities */
@layer utilities {
  .mt-2 { margin-top:.5rem; }
  .mt-4 { margin-top:1rem; }
}
```

포인트
- **접근성**: 스킵 링크, `aria-describedby`, 포커스 이동(`tabindex="-1"`).
- **성능**: 크리티컬 CSS 인라인 + 비차단 CSS 로드.
- **유지보수**: CSS **레이어(@layer)**, **BEM 네이밍**으로 충돌 최소화.

---

## CSS 실전 토픽 — 선택자·특이성·레이어·변수

### 특이성(Specificity) 요약

- 인라인 > ID > 클래스/속성/가상 > 태그/가상요소.
- 무분별한 `!important`는 피하고, **레이어(@layer)**와 **구조적 네이밍(BEM)** 으로 관리.

### CSS 변수(Custom Properties)

```css
:root { --brand: #0e7afe; --radius: 12px; }
.button { background: var(--brand); border-radius: var(--radius); }
```
- 런타임 토글(다크 모드, 테마 스위치)에 강력.

### 반응형/접근성 미디어 쿼리

```css
@media (max-width: 768px) { /* 모바일 레이아웃 */ }
@media (prefers-color-scheme: dark) { /* 다크 */ }
@media (prefers-reduced-motion: reduce) { * { animation: none !important; } }
```

### @layer로 우선순위 제어

```css
@layer reset, base, components, utilities;
```
- 선언 순서에 따라 충돌을 예측 가능하게 관리.

---

## JavaScript 실전 토픽 — DOM, 이벤트, 모듈, 대기열

### DOMContentLoaded vs load

- **DOMContentLoaded**: DOM 트리가 준비되면 실행(이미지/스타일 로드 기다리지 않음).
- **load**: 모든 리소스 로드 후 실행.

### 이벤트 위임(Event Delegation)

```js
document.body.addEventListener('click', (e) => {
  const a = e.target.closest('a[data-track]');
  if (!a) return;
  console.log('트래킹:', a.href);
});
```
- 동적 요소에 유용, 리스너 개수 감소 → 성능/메모리 이점.

### ES 모듈(권장)

```html
<script type="module">
  import { init } from '/assets/boot.mjs';
  init();
</script>
```
- 네이티브 모듈 그래프, 지연 실행(사실상 defer), **스코프 격리**.

---

## 보안 — CSP, SRI, 교차 출처

### CSP(Content-Security-Policy) 기본

```http
Content-Security-Policy:
  default-src 'self';
  script-src 'self' https://cdn.example 'strict-dynamic';
  style-src  'self' 'unsafe-inline'; /* 권장: nonce/hash 도입으로 unsafe-inline 제거 */
  object-src 'none';
```
- 인라인 스크립트는 **차단**이 기본. 필요한 경우 **nonce** 또는 **해시**를 사용.
- 프레임 삽입 방지: `frame-ancestors 'self';`

### 스크립트 무결성(SRI) + CORS

```html
<script src="https://cdn.example/lib.min.js"
        integrity="sha384-...hash..."
        crossorigin="anonymous"
        defer></script>
<link rel="stylesheet" href="https://cdn.example/lib.min.css"
      integrity="sha384-...hash..." crossorigin="anonymous">
```
- 외부 리소스 변조 방지.
- **SRI 사용 시** `crossorigin` 속성도 맞춰준다(대개 `anonymous`).

---

## 성능 — 로딩/페인트/인터랙션 최적화 체크리스트

- CSS: **크리티컬 인라인 + 비차단 로드**, `@import` 지양.
- JS: **defer** 기본, 광고/분석은 **async**.
- 이미지: `loading="lazy"`, `decoding="async"`, `srcset/sizes`, WebP/AVIF.
- 폰트: `preload` + `font-display: swap`; 서브셋 생성.
- 리소스 힌트: `dns-prefetch`, `preconnect`, `preload`, `modulepreload`.
- CLS 방지: 이미지/비디오에 `width/height` 또는 `aspect-ratio`.
- DevTools Coverage/Lighthouse로 **미사용 코드** 감축.

---

## 유지보수 — 아키텍처와 네이밍

- CSS: **BEM**(`.card__title--large`), ITCSS/레이어링(@layer)로 예측 가능한 우선순위.
- JS: 기능 단위 모듈(`user.mjs`, `cart.mjs`) + index에서 조립.
- 폴더: `/assets/css`, `/assets/js`, `/assets/img`, `/assets/fonts` 등 목적별 분리.

---

## 반응형·다크모드·설정 저장까지 한 번에

### 반응형 컨테이너/그리드

```css
.container { width: min(100% - 2rem, 72rem); margin-inline: auto; }
.grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(16rem, 1fr)); gap: 1rem; }
```

### 사용자 설정 보존(LocalStorage)

```js
const key = 'theme';
const saved = localStorage.getItem(key);
if (saved) document.documentElement.dataset.theme = saved;
```

### CSS 변수로 테마 분기

```css
:root[data-theme="dark"] { --bg:#0e0e0e; --fg:#e8e8e8; }
```

---

## 디버깅 팁

- **DevTools**: Elements(Computed)로 충돌/특이성 확인.
- **Coverage**: 미사용 CSS/JS 파악.
- **Network**: 우선순위/캐시/HTTP2 multiplexing 확인.
- **Performance**: Long Task, Layout/Style 리플로우 원인 분석.

---

## 자주 겪는 문제와 해결

| 문제 | 원인 | 해결 |
|---|---|---|
| FOUC | CSS 늦게 로드 | 핵심 스타일 인라인 + 비차단 로드/Preload |
| 이벤트 안 먹음 | DOM 전에 JS 실행 | `defer` 또는 DOMContentLoaded 사용 |
| 스타일 충돌 | 특이성 전쟁 | @layer/BEM/모듈화, `!important` 지양 |
| CLS 발생 | 이미지 크기 없음 | `width/height` 또는 `aspect-ratio` 지정 |
| CSP 차단 | 인라인 스크립트 | nonce/해시 도입, 외부 파일로 분리 |

---

## 요약 표

| 구분 | 방식 | 장점 | 단점 | 권장도 |
|---|---|---|---|---|
| CSS 인라인 | `style=""` | 즉시·국소 | 유지보수/특이성 문제 | 낮음 |
| CSS 내부 | `<style>` | 단일 문서 편리 | 재사용 어려움 | 중간 |
| CSS 외부 | `<link>` | 캐시·빌드·분리 | 초기 요청 추가 | **높음** |
| JS 인라인 | `onclick="..."` | 초간단 | 유지보수/보안 취약 | 낮음 |
| JS 내부 | `<script>` | 단문서 편리 | 재사용 어려움 | 중간 |
| JS 외부 | `<script src>` | 캐시·빌드·분리 | 요청 추가 | **높음** |
| 로드 전략 | `defer` | 순서/안정 | 없음 | **기본** |
| 로드 전략 | `async` | 빠름 | 순서 불가 | 부가 스크립트 |
| 모듈 | `type="module"` | 의존성/스코프 | 구형 브라우저 폴리필 | **현대 권장** |

---

## 다음 단계 제안

- CSS: **Flexbox/Grid** 레이아웃, `@container`(지원 브라우저에서)
- JS: **모듈·동적 import**, 이벤트 위임 패턴, Fetch/Streams
- 빌드 없이도 쓰는 **ESM + HTTP/2** 워크플로
- 보안: **CSP + SRI + frame-ancestors**, 폼 보안(XSS/CSRF)

이 문서의 패턴(외부 CSS/JS + defer/module + 크리티컬 CSS + 접근성 + CSP/SRI)을 기본 골격으로 삼으면, **정적 페이지 → 현대적 웹앱**으로의 전환이 자연스럽고 견고해진다.
