---
layout: post
title: CSS - Viewport와 Media Query
date: 2025-05-06 19:20:23 +0900
category: CSS
---
# Viewport와 Media Query 완전 정리

## 1. Viewport란? (Layout vs Visual)

- **Layout Viewport**: CSS 레이아웃 계산에 쓰이는 **논리적 뷰포트**. `vw/vh` 단위, `@media (width)` 등에서 사용.
- **Visual Viewport**: 사용자가 실제로 보는 **가시 영역**. 모바일에서 주소창 등장/스크롤에 따라 크기가 변합니다.

모바일 브라우저는 메타 태그가 없으면 **데스크탑 폭 가정(≈980px)** 으로 축소 렌더링합니다.

### 1-1) HTML 메타 뷰포트(필수)

```html
<meta name="viewport" content="width=device-width, initial-scale=1.0">
```

| 속성                | 의미                                   |
|---------------------|----------------------------------------|
| `width=device-width`| 레이아웃 폭을 **디바이스 CSS 픽셀 폭**으로 설정 |
| `initial-scale=1.0` | 초기 확대/축소 배율                         |

> **주의**: 접근성을 위해 `user-scalable=no`나 `maximum-scale=1`로 **줌을 막지 마세요**.

### 1-2) iOS 안전영역(노치) & 전체화면

노치/라운드 코너가 있는 기기에서 **상하단 안전 여백**을 고려합니다.

```html
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
```

```css
/* 안전영역 CSS 변수 */
:root {
  --safe-top: env(safe-area-inset-top);
  --safe-right: env(safe-area-inset-right);
  --safe-bottom: env(safe-area-inset-bottom);
  --safe-left: env(safe-area-inset-left);
}

/* 고정 헤더/푸터가 안전영역과 겹치면 패딩 추가 */
.header { padding-top: calc(1rem + var(--safe-top)); }
.footer { padding-bottom: calc(1rem + var(--safe-bottom)); }
```

---

## 2. 최신 Viewport 단위 — `dvh`, `svh`, `lvh`

모바일 주소창 노출/숨김에 따라 `100vh`가 **튀는 문제**를 해결하기 위한 신단위:

| 단위 | 의미 |
|------|------|
| `dvh` | **Dynamic** VH(주소창 등 UI 변동 반영) |
| `svh` | **Small** VH(가장 작은 가시 높이) — 주소창 펼침 기준 |
| `lvh` | **Large** VH(가장 큰 가시 높이) — 주소창 접힘 기준 |

```css
/* 주소창 높이 변동에도 꽉 채우는 섹션 */
.hero {
  min-height: 100dvh; /* 100vh 대신 권장 */
}

/* 고정 헤더가 있을 때 안전하게 레이아웃 계산 */
.main {
  min-height: calc(100svh - var(--header-height));
}
```

폴백을 위해 **순서대로 선언**하면 지원 브라우저에서만 덮어씁니다.

```css
.section { min-height: 100vh; }
.section { min-height: 100dvh; } /* 지원 브라우저에서 우선 */
```

---

## 3. Media Query란? (문법·논리·타입)

### 3-1) 기본 문법

```css
@media (조건) {
  /* 조건 만족 시 적용 */
}
```

연결/그룹화:

```css
@media screen and (min-width: 768px) and (hover: hover) { ... }
@media (max-width: 600px), (orientation: portrait) { ... } /* OR */
@media not (prefers-reduced-motion: reduce) { ... }
```

미디어 타입:

- `screen`(기본), `print`(인쇄), `speech`(보이스)

---

## 4. 자주 쓰는 미디어 특성(Features) — 기본부터 고급까지

### 4-1) 크기 기반

```css
@media (min-width: 768px) { ... }     /* 모바일 퍼스트 */
@media (max-width: 1024px) { ... }    /* 데스크탑 퍼스트 */
@media (min-height: 700px) { ... }    /* 가로모드 태블릿 최적화 등 */
@media (aspect-ratio: 16/9) { ... }   /* 화면 비율 정합 */
```

> **비율 공식(참고)**
> $$ \text{aspectRatio} = \frac{\text{width}}{\text{height}} $$
> `aspect-ratio: 16/9`는 위 비를 만족하는 뷰포트에만 적용됩니다.

### 4-2) 입력/상호작용

```css
/* 마우스 포인터 존재 & 정밀한 호버 가능 */
@media (hover: hover) and (pointer: fine) {
  .button:hover { transform: translateY(-1px); }
}

/* 터치 우선: 호버 대체 피드백 제공 */
@media (hover: none) and (pointer: coarse) {
  .button { min-height: 44px; } /* 터치 목표 크기 */
}
```

### 4-3) 접근성/환경 선호

```css
/* 모션 최소화 선호: 애니메이션 끔/완화 */
@media (prefers-reduced-motion: reduce) {
  * { animation: none !important; transition: none !important; }
}

/* 다크 모드 */
@media (prefers-color-scheme: dark) {
  :root { color-scheme: dark; }
  body { background: #0b1020; color: #e6e9ef; }
}

/* 대비 선호(일부 브라우저) */
@media (prefers-contrast: more) {
  :root { --border-strong: 2px; }
  .card { border-width: var(--border-strong); }
}

/* 강제 색상(Windows High Contrast Mode) */
@media (forced-colors: active) {
  .icon { forced-color-adjust: none; }
}
```

### 4-4) 색역/해상도

```css
/* 광색역 디스플레이 */
@media (color-gamut: p3) {
  :root { --brand: color(display-p3 0.2 0.6 0.9); }
}

/* 레티나/고해상도 */
@media (min-resolution: 2dppx) {
  .logo { background-image: url(logo@2x.png); }
}
```

---

## 5. 반응형 브레이크포인트 전략

**모바일 퍼스트**가 유지보수에 유리합니다.

```css
/* Mobile default */
.card-grid { grid-template-columns: 1fr; }

/* ≥ 600px: 2열 */
@media (min-width: 600px) {
  .card-grid { grid-template-columns: repeat(2, 1fr); }
}

/* ≥ 900px: 3열 */
@media (min-width: 900px) {
  .card-grid { grid-template-columns: repeat(3, 1fr); }
}
```

팀 규칙으로 토큰화:

```css
:root {
  --bp-sm: 600px;
  --bp-md: 900px;
  --bp-lg: 1200px;
}
@media (min-width:  var(--bp-sm)) { /* ... */ }
@media (min-width:  var(--bp-md)) { /* ... */ }
@media (min-width:  var(--bp-lg)) { /* ... */ }
```

---

## 6. 유동 타이포/간격 — `clamp()` 패턴

```css
:root {
  /* 글자 크기: 최소 1rem, 이상적 2vw+1rem, 최대 1.25rem */
  --fs-body: clamp(1rem, 2vw + 1rem, 1.25rem);

  /* 헤딩 스케일 */
  --fs-h2: clamp(1.25rem, 1.2vw + 1rem, 1.75rem);

  /* 유동 여백 */
  --space-4: clamp(.75rem, 1vw + .5rem, 1.25rem);
}

body { font-size: var(--fs-body); }
h2   { font-size: var(--fs-h2); margin: var(--space-4) 0; }
```

**장점**: 뷰포트에 따라 자연히 스케일, 미디어쿼리 수를 줄일 수 있음.

---

## 7. 실전 예제 모음

### 7-1) 반응형 헤더/내비 + 햄버거

```html
<header class="site-header">
  <a class="brand" href="/">Brand</a>
  <button class="hamburger" aria-expanded="false" aria-controls="nav">☰</button>
  <nav id="nav" class="nav-menu">
    <a href="#">Docs</a><a href="#">Blog</a><a href="#">About</a>
  </nav>
</header>
```

```css
.site-header {
  display: grid;
  grid-template-columns: auto 1fr auto;
  align-items: center;
  gap: 1rem;
  padding: .75rem clamp(1rem, 3vw, 2rem);
}

.nav-menu { display: flex; gap: 1rem; }
.hamburger { display: none; }

/* ≤ 768px: 메뉴 숨기고 햄버거 노출 */
@media (max-width: 768px) {
  .nav-menu { display: none; }
  .hamburger { display: inline-block; }
}

/* 주소창 변동에도 고정 높이로 보이는 히어로 섹션 */
.hero { min-height: 100dvh; display: grid; place-items: center; }
```

> 햄버거 클릭 토글은 JS로 `aria-expanded`와 `.nav-menu` 표시를 토글하세요.

### 7-2) 고해상도(레티나) 이미지/아이콘

```css
.logo { background: url(logo.png) no-repeat center / contain; width: 180px; height: 40px; }

@media (min-resolution: 2dppx) {
  .logo { background-image: url(logo@2x.png); }
}
```

대안: **벡터** 사용(SVG/Icon font) 또는 `<img srcset>`:

```html
<img
  src="card-600.jpg"
  srcset="card-600.jpg 600w, card-1200.jpg 1200w"
  sizes="(max-width: 600px) 100vw, 600px"
  alt="썸네일">
```

### 7-3) 프린트 스타일

```css
@media print {
  /* 여백/폰트 재조정 */
  body { color: #000; background: #fff; }
  a[href]::after { content: " (" attr(href) ")"; font-size: .85em; }
  .no-print { display: none !important; }
}
```

### 7-4) 입력기기 차별화(터치 vs 마우스)

```css
/* 터치: 탭 타겟 키우기 */
@media (hover: none) and (pointer: coarse) {
  .btn { padding: .9rem 1.2rem; }
}

/* 마우스: 호버 인터랙션 강화 */
@media (hover: hover) and (pointer: fine) {
  .btn:hover { transform: translateY(-1px); box-shadow: 0 8px 24px rgba(0,0,0,.12); }
}
```

### 7-5) 안전영역 고려 고정 푸터

```css
.footer-fixed {
  position: sticky;
  bottom: 0;
  padding: .75rem calc(1rem + env(safe-area-inset-right)) calc(.75rem + env(safe-area-inset-bottom)) calc(1rem + env(safe-area-inset-left));
  backdrop-filter: saturate(1.2) blur(6px);
  background: color-mix(in oklab, white 85%, transparent);
}
```

---

## 8. 흔한 함정과 대처

1. **`100vh` 점프** (모바일 주소창 표시/숨김)
   → `100dvh/svh/lvh` 활용 + 레이아웃에 `min-height` 사용.
2. **줌 비활성화**(접근성 저하)
   → `user-scalable=no` 지양. 포커스/탭 이동이 가능한 UI 설계.
3. **DOM 순서와 시각 순서 불일치**
   → 미디어쿼리로 시각 재배치해도 **DOM 탐색 순서**는 접근성에 영향. 논리적 순서를 유지.
4. **호버 의존 UX**
   → `(hover:none)` 환경에서 **동등한 피드백**(active/focus/aria-expanded) 제공.
5. **색만으로 차이 전달**
   → 대비, 밑줄, 아이콘, 텍스트 등 **다중 신호** 제공. `(prefers-contrast)` 대응.

---

## 9. 성능·유지보수 팁

- 브레이크포인트는 **토큰화**(`--bp-md: 900px`)하여 재사용.
- **컴포넌트화**(BEM/유틸리티/프레임워크)로 미디어쿼리 분산 관리.
- 가능한 곳에서 **유동 스케일(clamp)** 로 미디어쿼리 개수 줄이기.
- 이미지에는 **`srcset` + `sizes`** 로 네트워크 최적화.
- **컨테이너 쿼리**(`@container`)도 검토: 부모 폭 기준 반응형(별도 주제).

---

## 10. 최종 체크리스트

- [ ] `<meta name="viewport" content="width=device-width, initial-scale=1">`
- [ ] `100vh` 대신 `100dvh/svh/lvh` 검토
- [ ] 터치/마우스 입력 차이(`hover/pointer`) 반영
- [ ] 다크/모션/대비 선호(`prefers-*`) 대응
- [ ] 고해상도(`min-resolution: 2dppx`)와 색역(`color-gamut`) 고려
- [ ] 프린트 스타일 제공(`@media print`)
- [ ] 링크/버튼 포커스 표시 확실(`:focus-visible`)
- [ ] DOM 순서가 논리적이고 접근 가능
- [ ] 이미지 `srcset/sizes` 및 레이아웃 셰이프 설정(누적 레이아웃 이동 방지)

---

## 부록) Sass 믹스인 예시 (선택)

```scss
$bp-sm: 600px;
$bp-md: 900px;
$bp-lg: 1200px;

@mixin up($w) { @media (min-width: $w) { @content; } }
@mixin down($w){ @media (max-width: $w) { @content; } }

.grid { display: grid; grid-template-columns: 1fr; gap: 1rem; }
@include up($bp-sm) { .grid { grid-template-columns: repeat(2, 1fr); } }
@include up($bp-md) { .grid { grid-template-columns: repeat(3, 1fr); } }
@include up($bp-lg) { .grid { grid-template-columns: repeat(4, 1fr); } }
```

---

## 참고 링크

- MDN — Using media queries: https://developer.mozilla.org/en-US/docs/Web/CSS/Media_Queries/Using_media_queries
- MDN — Viewport meta tag: https://developer.mozilla.org/en-US/docs/Web/HTML/Viewport_meta_tag
- CSS-Tricks — Media Queries Guide: https://css-tricks.com/a-complete-guide-to-css-media-queries/
- Viewport units `dvh/svh/lvh`: 각 브라우저 릴리스 노트/MDN 참고
