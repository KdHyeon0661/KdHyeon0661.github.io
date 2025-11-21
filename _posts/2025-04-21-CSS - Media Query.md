---
layout: post
title: CSS - Media Query
date: 2025-04-21 20:20:23 +0900
category: CSS
---
# 반응형 디자인 기초: Media Query 사용법

## 미디어 쿼리란?

CSS에서 **환경(화면 크기, 방향, 해상도, 입력 방식 등)**을 조건으로 삼아 **조건부 스타일**을 적용하는 규칙입니다.

```css
@media (조건) {
  /* 조건을 만족할 때 적용될 CSS */
}
```

---

## 미디어 타입과 기본 문법

### 미디어 타입

- `screen` (화면), `print` (인쇄), `speech`(음성) 등

```css
@media screen and (max-width: 768px) {
  body { background: #e8f3ff; }
}
@media print {
  /* 인쇄 전용 스타일 */
  a::after { content: " (" attr(href) ")"; }
}
```

### 연결자

- `and`: AND 조건
- `,`(쉼표): OR 조건
- `not`: 부정

```css
/* 600~1024px 사이의 screen */
@media screen and (min-width: 600px) and (max-width: 1024px) {}

/* (max-width:600px) OR 세로방향 */
@media (max-width: 600px), (orientation: portrait) {}
```

---

## — 치트시트

| 분류 | 특성 | 예시 | 설명 |
|---|---|---|---|
| 크기 | `width/height` | `(min-width:768px)` | 뷰포트 크기 (레이아웃 핵심) |
| 방향 | `orientation` | `(orientation: landscape)` | 가로/세로 |
| 비율 | `aspect-ratio` | `(aspect-ratio > 3/2)` | 뷰포트 가로:세로 비 |
| 해상도 | `resolution` | `(min-resolution: 2dppx)` | 고DPI(Retina) 대응 |
| 색역 | `color-gamut` | `(color-gamut: p3)` | sRGB보다 넓은 색역 |
| 입력 | `hover`/`pointer` | `(hover: none)`, `(pointer: coarse)` | 터치/마우스 특성 |
| 선호 | `prefers-color-scheme` | `(prefers-color-scheme: dark)` | 다크/라이트 |
| 선호(모션) | `prefers-reduced-motion` | `(prefers-reduced-motion: reduce)` | 모션 최소화 |
| 대비 | `prefers-contrast`* | `(prefers-contrast: more)` | 고대비 선호(브라우저 지원 주의) |

\* 일부 브라우저 한정. 강건하게 설계(폴백 필수).

---

## Level 4 범위 문법(가독성↑)

기존: `min-/max-` 조합 → **범위 표현**으로 더 자연스럽게:

```css
/* 600px 이상 */
@media (width >= 600px) {}

/* 600~1024px 구간 */
@media (600px <= width <= 1024px) {}

/* 768px 이상 1200px 미만 */
@media (768px <= width < 1200px) {}
```

> **가독성/유지보수성** 향상. (지원 범위: 최신 브라우저 중심)

---

## 모바일 퍼스트 전략(권장)

1) **기본 스타일** = 모바일(좁은 화면)
2) **확장** = `min-width`로 큰 화면에만 오버라이드

```css
/* 1) 기본: 모바일 */
.card-grid {
  display: grid;
  grid-template-columns: 1fr;
  gap: 1rem;
}

/* 2) 태블릿 이상 */
@media (min-width: 600px) {
  .card-grid { grid-template-columns: repeat(2, 1fr); }
}

/* 3) 데스크탑 이상 */
@media (min-width: 900px) {
  .card-grid { grid-template-columns: repeat(3, 1fr); }
}
```

> **데스크탑 퍼스트**는 줄여나가는 방식(`max-width`)이지만, 실무에선 모바일 퍼스트가 **코드량/성능/접근성** 측면에서 유리.

---

## 브레이크포인트 설계(토큰화)

### 권장 구간

- `sm: 600px`, `md: 768px`, `lg: 1024px`, `xl: 1280px`, `2xl: 1536px` 등
- 프로젝트/디자인 시스템에 맞춰 **토큰 변수**로 통일

```css
:root{
  --bp-sm: 600px;
  --bp-md: 768px;
  --bp-lg: 1024px;
  --bp-xl: 1280px;
}
@media (min-width:  var(--bp-md)) { /* ... */ }
@media (min-width:  var(--bp-lg)) { /* ... */ }
```

### 구간 정의(수학적)

한 요소 스타일의 활성 구간을
$$\text{Active}(w) = [w_\text{min},\; w_\text{max})$$
로 두고, **상충/틈(겹침/빈 구간)** 없게 설계합니다.

---

## 주요 패턴 — 실전 스니펫

### 내비게이션: mobile → desktop

```css
.nav {
  display:flex; align-items:center; gap:.75rem;
  overflow-x:auto; /* 모바일 스크롤 */
}
@media (min-width: 1024px) {
  .nav { justify-content: space-between; overflow:visible; }
}
```

### 사이드바: off-canvas → 고정

```css
.sidebar {
  position: fixed; inset: 0 0 0 20%; transform: translateX(-100%);
  transition: transform .25s ease; background:#fff; box-shadow: 0 10px 30px rgba(0,0,0,.15);
}
.sidebar[aria-expanded="true"] { transform: none; }

@media (min-width: 1024px) {
  .sidebar { position: sticky; inset: auto; top: 0; transform: none; box-shadow: none; }
}
```

### 카드 그리드(반응형)

```css
.cards {
  display:grid; grid-template-columns: 1fr; gap: 1rem;
}
@media (min-width: 600px)  { .cards { grid-template-columns: repeat(2, 1fr); } }
@media (min-width: 900px)  { .cards { grid-template-columns: repeat(3, 1fr); } }
@media (min-width: 1200px) { .cards { grid-template-columns: repeat(4, 1fr); } }
```

### 유동 타이포그래피: `clamp()`

```css
:root{
  --fs-body: clamp(0.95rem, 0.5vw + 0.8rem, 1.1rem);
  --fs-h1:   clamp(1.6rem, 2.5vw + 1rem, 2.6rem);
}
body { font-size: var(--fs-body); }
h1   { font-size: var(--fs-h1); line-height: 1.2; }
```

---

## 입력/상호작용 환경 고려

터치 스크린에서 `:hover`는 기대대로 동작하지 않을 수 있습니다. `hover/pointer`를 활용해 UI를 조정합니다.

```css
/* 포인터가 정확하고 hover 가능(마우스) */
@media (hover: hover) and (pointer: fine) {
  .menu .item:hover { background: rgba(0,0,0,.05); }
}

/* 터치 환경(hover 없음, 포인터 거침) → 클릭영역 확대/간격 넓힘 */
@media (hover: none) and (pointer: coarse) {
  .menu .item { padding: 14px 16px; }
  .tooltip { display: none; } /* 터치에선 툴팁 제거 또는 버튼화 */
}
```

`any-pointer/any-hover`는 복합 환경(노트북 터치+마우스)에 대응할 때 활용.

---

## 방향/비율 기반 레이아웃

### 방향

```css
/* 가로 모드일 때 사이드바 + 콘텐츠 배치 */
@media (orientation: landscape) {
  .layout { grid-template-columns: 280px 1fr; }
}
```

### 비율 기반(영화/슬라이드)

```css
/* 16:9보다 넓은 와이드 레이아웃 */
@media (aspect-ratio > 16/9) {
  .hero { max-height: 70vh; }
}
```

---

## 해상도/색역(고품질 이미지/아이콘)

### 고DPI 그래픽

```css
/* 2배 밀도 이상 디스플레이 */
@media (min-resolution: 2dppx) {
  .logo { background-image: url(logo@2x.png); background-size: 160px 40px; }
}
```

### 넓은 색역(P3)

```css
/* P3 색역에서만 더 선명한 색상 사용 */
@media (color-gamut: p3) {
  :root { --brand: color(display-p3 0.2 0.6 0.9); }
}
```

---

## 사용자 선호도: 다크모드/모션/대비

### 다크 모드

```css
/* 기본(라이트) */
:root{ --bg: #ffffff; --fg: #111; --card: #fff; }
/* 다크 */
@media (prefers-color-scheme: dark) {
  :root{ --bg: #0b1220; --fg: #e6e8ef; --card: #0f172a; }
  .card{ box-shadow: 0 2px 8px rgba(0,0,0,.5); }
}
body { background: var(--bg); color: var(--fg); }
```

### 모션 감소

```css
@media (prefers-reduced-motion: reduce) {
  * { animation: none !important; transition: none !important; }
  .carousel { scroll-behavior: auto; }
}
```

### 높은 대비(지원 브라우저 한정)

```css
@media (prefers-contrast: more) {
  :root { --border: #000; --focus: #1d4ed8; }
  .btn { border-color: var(--border); }
  .btn:focus-visible { outline: 3px solid var(--focus); outline-offset: 2px; }
}
```

---

## 뷰포트 메타와 동적 뷰포트 단위

### 필수 `<meta>`

```html
<meta name="viewport" content="width=device-width, initial-scale=1.0">
```

### 동적 뷰포트 단위 (모바일 브라우저 UI 반영)

- `svh`, `lvh`, `dvh` 등: 안전/큰/동적 viewport height
```css
.hero { height: 100dvh; } /* 주소창 유무에 동적으로 반응 */
```

---

## 인쇄/접근성 미디어

### 인쇄

```css
@media print {
  /* 배경 제거, 잉크 절약 */
  * { background: transparent !important; box-shadow: none !important; }
  body { color: #000; }
  nav, .no-print { display: none !important; }
  a::after { content: " (" attr(href) ")"; font-size: .9em; }
}
```

### 대화형(키오스크 등)

```css
@media speech {
  /* 음성 합성 환경에 특화된 스타일(필요시) */
}
```

---

## 종합 예제 — 반응형 마케팅 섹션

```html
<!doctype html>
<html lang="ko">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Responsive Demo</title>
<style>
  :root{
    --bp-sm: 600px; --bp-md: 768px; --bp-lg: 1024px; --bp-xl: 1280px;
    --bg: #ffffff; --fg: #111827; --muted: #6b7280; --card: #fff;
    --brand: #2563eb;
    --fs-body: clamp(0.98rem, 0.4vw + 0.9rem, 1.1rem);
    --fs-h1: clamp(1.6rem, 2.5vw + 1rem, 2.6rem);
  }
  @media (prefers-color-scheme: dark){
    :root{ --bg:#0b1220; --fg:#e5e7eb; --muted:#9ca3af; --card:#0f172a; }
  }

  body{
    margin:0; font-family: system-ui, "Noto Sans KR", sans-serif;
    background: var(--bg); color: var(--fg); font-size: var(--fs-body);
  }
  .nav{
    display:flex; align-items:center; gap:.75rem; padding: .8rem 1rem;
    border-bottom: 1px solid rgba(0,0,0,.05);
  }
  .brand{ font-weight:700; color: var(--brand); font-size: 1.1rem; }
  .spacer{ flex:1; }
  .hero{
    display:grid; align-content:center; justify-items:center; text-align:center;
    padding: clamp(2rem, 6vw, 6rem) 1rem;
  }
  .hero h1{ font-size: var(--fs-h1); margin:.15em 0 .3em; }
  .hero p { color: var(--muted); }

  .cta{
    display:inline-flex; align-items:center; gap:.5rem; margin-top:1rem;
    padding:.75rem 1rem; border-radius: 12px; background: var(--brand); color:#fff;
    text-decoration:none; box-shadow: 0 6px 16px rgba(37,99,235,.25);
  }
  .cards{
    display:grid; grid-template-columns:1fr; gap:1rem; padding:1rem; max-width:1200px; margin:0 auto;
  }
  .card{
    background: var(--card);
    border:1px solid rgba(0,0,0,.06);
    border-radius: 14px; padding: 1rem;
    box-shadow: 0 1px 2px rgba(0,0,0,.06), 0 6px 12px rgba(0,0,0,.06);
  }
  .card h3{ margin:.2em 0 .4em; }
  .card p { color: var(--muted); }

  /* Tablet+ */
  @media (min-width:  var(--bp-sm))  { .cards { grid-template-columns: repeat(2, 1fr); } }
  /* Laptop+ */
  @media (min-width:  var(--bp-lg))  {
    .nav{ padding: 1rem 2rem; }
    .cards { grid-template-columns: repeat(3, 1fr); gap:1.25rem; padding:2rem; }
    .hero{ text-align:left; justify-items:start; padding-inline: clamp(2rem, 6vw, 8rem); }
  }

  /* 좁은 하이와이드 화면에서 히어로 높이 제한 (가로비 화면) */
  @media (aspect-ratio > 16/9) {
    .hero{ max-height: 70vh; }
  }

  /* 터치 환경: 톡톡한 버튼, 도구팁 제거 */
  @media (hover: none) and (pointer: coarse) {
    .cta{ padding: .9rem 1.1rem; border-radius: 16px; }
  }

  /* 고DPI: 더 선명한 로고 이미지 */
  .logo{ width: 120px; height: 30px; background: url(logo.png) no-repeat / 120px 30px; }
  @media (min-resolution: 2dppx) {
    .logo{ background-image: url(logo@2x.png); }
  }

  @media print {
    .nav, .cta { display:none !important; }
    .card { box-shadow: none; border-color: #000; }
    a::after{ content:" (" attr(href) ")"; }
  }
</style>
</head>
<body>
  <header class="nav">
    <div class="logo" aria-label="Brand"></div>
    <strong class="brand">MyBrand</strong>
    <div class="spacer"></div>
    <nav class="nav-links">
      <a href="#">Features</a>
      <a href="#">Pricing</a>
      <a href="#">Docs</a>
    </nav>
  </header>

  <section class="hero">
    <h1>한 번의 코드로 모든 화면에 딱 맞게</h1>
    <p>Media Query와 Grid/Flex를 조합해 모바일부터 데스크탑까지 자연스러운 레이아웃을 구현합니다.</p>
    <a class="cta" href="#learn">시작하기</a>
  </section>

  <section class="cards">
    <article class="card">
      <h3>모바일 퍼스트</h3>
      <p>기본 스타일은 모바일, 큰 화면만 오버라이드합니다. 초기 페인트가 가볍고 예측 가능합니다.</p>
    </article>
    <article class="card">
      <h3>유동 타이포</h3>
      <p><code>clamp()</code>로 자연스러운 폰트 스케일과 가독성을 보장하세요.</p>
    </article>
    <article class="card">
      <h3>상호작용 적응</h3>
      <p><code>hover/pointer</code>로 입력 장치에 맞춰 간격/상태를 조정합니다.</p>
    </article>
  </section>
</body>
</html>
```

---

## 그리드/플렉스와의 조합

- **Breakpoints**는 “언제 레이아웃을 바꿀지” 결정 → **Grid/Flex**는 “어떻게 배치할지” 처리
- 예: 모바일 1열 → `(min-width: 768px)`에서 2열, `(min-width: 1024px)`에서 3열

```css
.list { display:flex; flex-direction: column; gap: .75rem; }
@media (min-width: 768px) {
  .list { flex-direction: row; flex-wrap: wrap; }
  .list > li { flex: 1 1 calc(50% - .75rem); }
}
@media (min-width: 1024px) {
  .list > li { flex-basis: calc(33.333% - .75rem); }
}
```

---

## 디버깅/성능/안정성 체크리스트

- [ ] **중첩/중복 규칙**으로 인한 특이성 폭증을 방지(컴포넌트 범위화, BEM/SCSS 모듈)
- [ ] 불필요한 `max-width`/`min-width` 조합 제거, **범위 문법**으로 단순화
- [ ] **레이아웃 전환 지점**(breakpoint)에서 **재흐름/점프** 최소화(높이/간격의 부드러운 변화)
- [ ] 애니메이션은 `(prefers-reduced-motion: reduce)` 고려
- [ ] 터치 환경에서 **대상 크기**(48px 이상)와 간격 확보
- [ ] 고DPI/넓은 색역에서는 자산/색상도 고품질 버전 제공
- [ ] 인쇄 스타일 제공(브랜드 신뢰/문서 가독성 향상)

---

## 자주 쓰는 미디어 쿼리 스니펫 모음

```css
/* 1) 3단계 브레이크포인트 */
@media (min-width: 600px)  { /* tablet */ }
@media (min-width: 900px)  { /* small desktop */ }
@media (min-width: 1200px) { /* large desktop */ }

/* 2) 범위 문법으로 섬세하게 */
@media (600px <= width < 900px) { /* mid range only */ }

/* 3) 다크모드 */
@media (prefers-color-scheme: dark) {}

/* 4) 모션 감소 */
@media (prefers-reduced-motion: reduce) {}

/* 5) 터치 우선 */
@media (hover: none) and (pointer: coarse) {}

/* 6) 와이드 스크린 */
@media (aspect-ratio > 16/9) {}

/* 7) 고해상도 */
@media (min-resolution: 2dppx) {}

/* 8) 프린트 */
@media print {}
```

---

## 실수/함정 피하기

- **브레이크포인트 남발** → 유지보수 지옥. **컴포넌트 목적** 중심으로 3~5개 이내.
- 터치환경에서 **hover 의존 UI** → 요약 뱃지/툴팁은 클릭/포커스로도 접근 가능하게.
- `vh` 고정 높이로 모바일 주소창 가림 → **`dvh`/`svh`** 사용 고려.
- 데스크탑 퍼스트에서 **작은 화면 폴백 미흡** → 초기 페인트 엉킴/스크롤 가림.

---

## 결론 요약

| 항목 | 핵심 |
|---|---|
| 전략 | **모바일 퍼스트**, 브레이크포인트 **토큰화** |
| 문법 | Level 4 **범위 문법**으로 가독성/정확성↑ |
| 적응 | `hover/pointer`, `orientation/aspect-ratio`로 상호작용/배치 보완 |
| 품질 | `resolution`, `color-gamut`로 고DPI/넓은 색역 대응 |
| 접근성 | `prefers-color-scheme`/`prefers-reduced-motion` 반영 |
| 성능 | 브레이크포인트 최소화, 전환 지점 튜닝, 인쇄/다크모드 폴백 |

반응형의 본질은 **환경 가정이 아니라, 증거(특성 탐지) 기반 적응**입니다.
Media Query를 “브라우저의 감각 기관”이라 생각하고, **명확한 구간 설계**와 **컴포넌트 중심 구성**으로 견고한 UI를 완성하세요.
