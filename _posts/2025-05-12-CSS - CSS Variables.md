---
layout: post
title: CSS - CSS Variables
date: 2025-05-12 19:20:23 +0900
category: CSS
---
# CSS 커스텀 속성(CSS Variables)

## 핵심 요약 (한 장)

- **선언**: `--token-name: value;` (접두사 `--` 필수)
- **사용**: `var(--token-name, fallback)` (두 번째 인자는 **폴백**)
- **상속**: 커스텀 속성은 **기본적으로 상속**됨(일반 속성과 다르게 대부분 상속).
- **스코프**: `:root`(전역) + **로컬 스코프**(컴포넌트/컨테이너 내부 재정의).
- **장점**: **런타임** 제어(JS), **반응형/미디어쿼리**와 결합, **테마(라이트/다크/브랜드)**, **설계 토큰화**.
- **주의**: 해석 실패 시 **computed 단계에서 무효**(폴백 권장), IE 미지원.

---

## 기본 문법 & 스코프/상속

```css
/* 전역(문서 루트) 설계 토큰 */
:root {
  --brand: #3498db;
  --radius: 12px;
  --gap: 1rem;
  --fs-body: 1rem;
}

/* 컴포넌트 로컬 재정의(스코프 우선) */
.card {
  --radius: 16px;                /* :root보다 우선 */
  border-radius: var(--radius);
  padding: var(--gap);
  color: var(--text-color, #333); /* 미정의 시 폴백 #333 */
}
```

- **로컬 변수가 전역보다 우선**(카스케이딩 규칙).
- **상속**: 하위 요소는 상위에서 정의된 `--*`를 **상속**받음.
- **폴백**은 “**변수가 전혀 없을 때만**” 사용. 값은 있는데 **빈 문자열**이면 그 값이 그대로 적용됨(주의).

---

## 계산 규칙 & 폴백 & 에러(Invalid at computed value time)

### `var()`는 **computed 단계**에서 치환

- `var(--a)`가 해석 불가하면 해당 **선언 전체가 무효**(shorthand 포함).
- **폴백 제공**: `var(--a, 10px)` → `--a` 미정의 시 `10px` 사용.

```css
.box {
  /* 잘못된 예: --pad가 미정의면 전체 shorthand 무효가 된다 */
  padding: 8px var(--pad) 8px;

  /* 안전한 예: 각 자리에 폴백 부여(또는 사전 정의) */
  padding: 8px var(--pad, 16px) 8px;
}
```

### 수학 함수와 혼용

모든 **길이/수치형** 컨텍스트에서 사용 가능. `calc()`/`clamp()`/`min()`/`max()`와 조합 권장.

```css
:root {
  --space: 1rem;
  --w-card-min: 280px;
  --w-card-max: 520px;
}
.card {
  width: clamp(var(--w-card-min), 40vw, var(--w-card-max));
  margin-inline: var(--space);
  padding: calc(var(--space) * 1.5);
}
```

---

## 설계 토큰(Design Tokens)로 체계화

### → 의미 토큰(semantic) 구분

```css
/* Primitive tokens */
:root {
  --c-blue-600: hsl(210 90% 45%);
  --c-red-600:  hsl(0 80% 50%);
  --space-1: .25rem; --space-2: .5rem; --space-4: 1rem; --space-6: 1.5rem;
  --fs-1: .875rem; --fs-2: 1rem; --fs-3: 1.125rem; --fs-4: 1.375rem;
}

/* Semantic tokens (역할 기반) */
:root {
  --brand: var(--c-blue-600);
  --danger: var(--c-red-600);
  --gap: var(--space-4);
  --font-body: var(--fs-2);
  --font-h1: var(--fs-4);
}
```

> **원천 토큰**은 팔레트/스케일, **의미 토큰**은 역할/컴포넌트 맵핑. 교체가 쉬움.

### 반응형 토큰(미디어쿼리에서 변수만 바꾸기)

```css
:root {
  --gap: 1rem;
  --fs-body: 1rem;
}
@media (min-width: 768px) {
  :root {
    --gap: 1.5rem;
    --fs-body: 1.125rem;
  }
}
.prose {
  font-size: var(--fs-body);
  gap: var(--gap);
}
```

---

## & 사용자 프리퍼런스

### Attribute/클래스로 테마 스왑

```css
:root[data-theme="light"] {
  --bg: #fff; --fg: #111; --surface: #f6f7f8;
  --brand: hsl(221 83% 53%);
}
:root[data-theme="dark"] {
  --bg: #0b0b0c; --fg: #eaeaea; --surface: #141518;
  --brand: hsl(221 83% 70%);
}
body { background: var(--bg); color: var(--fg); }
.card { background: var(--surface); border-radius: 12px; }
.btn-primary { background: var(--brand); color: #fff; }
```

```js
// JS로 테마 전환
const root = document.documentElement;
const next = root.getAttribute('data-theme') === 'dark' ? 'light' : 'dark';
root.setAttribute('data-theme', next);
```

### OS 설정 연동 (prefers-color-scheme)

```css
/* 기본 라이트 */
:root { --bg:#fff; --fg:#111; }
/* OS 다크 선호 시 자동 적용 */
@media (prefers-color-scheme: dark) {
  :root { --bg:#0b0b0c; --fg:#eaeaea; }
}
```

### 하이 콘트라스트/감소된 모션

```css
@media (prefers-contrast: more) {
  :root { --outline: 3px; }
}
@media (prefers-reduced-motion: reduce) {
  :root { --transition: 0ms; }
}
.button { transition: background var(--transition, 160ms) ease; }
```

---

## 타이포그래피/레이아웃/컴포넌트 실전 패턴

### 유동 타이포(미디어쿼리 없이)

```css
:root { --fs-fluid: clamp(1rem, 0.9rem + 0.6vw, 1.25rem); }
body  { font-size: var(--fs-fluid); line-height: 1.6; }
h1    { font-size: clamp(1.5rem, 1.1rem + 3vw, 3rem); }
```

**수학 직관**: 뷰포트 너비 \(w\)에 대해 선형 스케일 \(a\cdot w + b\)를 **clamp(min, ideal, max)**로 감싼다.

\[
f(w) = \mathrm{clamp}\big(f_{\min}, a \cdot w + b, f_{\max}\big)
\]

### & 모바일 뷰포트

```css
:root {
  --safe-top:    env(safe-area-inset-top, 0px);
  --safe-right:  env(safe-area-inset-right, 0px);
  --safe-bottom: env(safe-area-inset-bottom, 0px);
  --safe-left:   env(safe-area-inset-left, 0px);
}
.header {
  padding-block-start: calc(1rem + var(--safe-top));
}
.footer {
  padding-block-end: calc(1rem + var(--safe-bottom));
}
```

### Grid 레이아웃 + 토큰

```css
:root { --gap: clamp(.75rem, 1vw, 1.25rem); --w-min: 18rem; }
.grid {
  display: grid;
  gap: var(--gap);
  grid-template-columns: repeat(auto-fit, minmax(min(100%, var(--w-min)), 1fr));
}
.card { padding: var(--gap); border-radius: 12px; background: var(--surface, #fff); }
```

### 버튼/카드의 상태/테마 대응

```css
:root { --elev: 0 8px 24px rgba(0,0,0,.12); }
.card { box-shadow: var(--elev); transition: box-shadow .2s ease, transform .2s ease; }
@media (hover:hover){
  .card:hover { transform: translateY(-2px); box-shadow: 0 14px 40px rgba(0,0,0,.18); }
}

.btn {
  --btn-bg: var(--brand, #2563eb);
  --btn-fg: #fff;
  padding: .75rem 1.25rem;
  background: var(--btn-bg);
  color: var(--btn-fg);
  border: 0; border-radius: 10px;
}
.btn.ghost {
  --btn-bg: color-mix(in oklab, var(--brand) 12%, transparent);
  --btn-fg: var(--brand);
}
```

> `color-mix()`는 최신 기능(대부분 모던 브라우저 지원). 구버전 폴백 필요 시 단색 대체.

---

## JavaScript 연동 (읽기/쓰기/테마 저장)

```js
const root = document.documentElement;

/* 읽기 */
const brand = getComputedStyle(root).getPropertyValue('--brand').trim();

/* 쓰기 */
root.style.setProperty('--brand', 'hsl(10 80% 50%)');

/* 토글 + LocalStorage */
const next = (root.dataset.theme === 'dark') ? 'light' : 'dark';
root.dataset.theme = next;
localStorage.setItem('theme', next);

/* 초기 부팅 시 복원(FOUC 최소화 위해 inline 스크립트 권장) */
const saved = localStorage.getItem('theme');
if (saved) root.dataset.theme = saved;
```

---

## 애니메이션/전환과 커스텀 속성

- **변수가 바뀌면** 이를 **참조**하는 속성(`background`, `transform`, `opacity` 등)이 **변경**되어 **그 속성에 `transition`이 걸려 있으면** 부드럽게 전환됨.
- 커스텀 속성 **자체를 애니메이션**하려면 **CSS Properties and Values API**의 `@property`로 **타입 등록** 필요.

### 단순 전환(참조 속성에 transition)

```css
.card { background: var(--surface, #fff); transition: background 160ms ease; }
:root[data-theme="dark"] { --surface: #141518; }
```

### `@property`로 숫자형 변수 등록 → 직접 transition

```css
/* 커스텀 속성 타입 선언(길이로 지정) */
@property --r {
  syntax: '<length>';
  inherits: true;
  initial-value: 8px;
}

.box {
  border-radius: var(--r);
  transition: --r 200ms ease; /* 변수 자체를 트랜지션 */
}
.box:hover {
  --r: 24px;
}
```

> 지원: Chrome/Edge/Safari 등 모던 브라우저. (Firefox는 플래그/진행상황 확인)
> 타입 등록을 안 하면 변수는 **타이핑 불명** → 직접 전환 불가(참조 속성 전환으로 우회).

---

## Cascade Layers, 컨테이너/미디어/지원쿼리와의 결합

### Cascade Layers로 우선순위 체계화

```css
@layer reset, tokens, base, components, utilities;

@layer tokens {
  :root { --gap: 1rem; --brand: #2563eb; }
}
@layer components {
  .button { padding: var(--gap); background: var(--brand); }
}
@layer utilities {
  .p-lg { --gap: 2rem; }     /* 유틸에서 토큰 오버라이드 */
}
```

### @media / @supports / @container 안에서 변수만 바꾸기

```css
@supports (backdrop-filter: blur(8px)) {
  :root { --glass: blur(8px); }
}
.glass { backdrop-filter: var(--glass, none); }

@container (inline-size > 720px) {
  .panel { --cols: 3; }
}
.panel { display: grid; grid-template-columns: repeat(var(--cols, 1), 1fr); }
```

---

## Shadow DOM(웹 컴포넌트)와 커스텀 속성

- **커스텀 속성은 Shadow 경계도 통과**(상속) → 호스트에서 테마/토큰 주입 가능.

```css
/* 호스트 페이지 */
my-badge { --badge-bg: #111; --badge-fg: #fff; }
/* 컴포넌트 내부(Shadow) */
:host { display: inline-block; }
.badge {
  background: var(--badge-bg, #333);
  color: var(--badge-fg, #eee);
}
```

---

## 성능/안정성/팀 협업 규칙

- **네이밍**: `--scope-role-state`(예: `--btn-bg-hover`) / **프리픽스**로 도메인 구분(`--app-*`).
- **중첩 계산**을 줄이고, **토큰 분해**(원천/의미/컴포넌트 단위).
- **DevTools**에서 **Computed** 패널로 실제 값 추적(유효성 문제는 *invalid at computed value time* 여부 체크).
- **빌드 체인**: `postcss-custom-properties`(단, 런타임 동적 변경 기능은 사라짐 → 목적 따라 병행 전략).

---

## 디버깅 체크리스트

- [ ] `var(--x, fallback)`의 **폴백**이 실제로 유효한 타입/단위인가?
- [ ] shorthand 속성에서 `var()` 실패 시 **선언 전체 무효** → longhand로 분리?
- [ ] 상속/스코프: 예상치 못한 **부모의 변수값**이 오버라이드?
- [ ] DevTools에서 **Computed**/“Force state”/“Emulate CSS prefers-*”로 시뮬레이션 테스트.
- [ ] `@property` 미등록 상태에서 **커스텀 속성 직접 transition** 기대하지 않았는가?
- [ ] 브라우저 지원: **IE 미지원**(필요 시 전처리기 변수/폴백 CSS 병행).

---

## 완성 예제 — “테마 가능한 대시보드 카드”

```html
<main class="wrap">
  <article class="card">
    <h3>Sessions</h3>
    <p>Last 24h</p>
    <strong>12,843</strong>
    <button class="btn">Details</button>
  </article>
  <article class="card alt">
    <h3>Revenue</h3>
    <p>Last 24h</p>
    <strong>$8,230</strong>
    <button class="btn">Details</button>
  </article>
</main>

<script>
  // 테마 토글
  const root = document.documentElement;
  document.addEventListener('keydown', (e) => {
    if (e.key.toLowerCase() === 't') {
      root.dataset.theme = (root.dataset.theme === 'dark') ? 'light' : 'dark';
    }
  });
</script>
```

```css
@layer tokens, base, components;

@layer tokens {
  :root[data-theme="light"] {
    --bg: #f7f7f8;
    --surface: #ffffff;
    --fg: #171717;
    --muted: #6b7280;
    --brand:  hsl(221 83% 53%);
    --gap: clamp(1rem, 1rem + 1vw, 2rem);
    --radius: clamp(10px, 1vw, 16px);
    --shadow: 0 10px 24px rgba(0,0,0,.08);
  }
  :root[data-theme="dark"] {
    --bg: #0c0d10;
    --surface: #14161a;
    --fg: #eaeaea;
    --muted: #9aa0a6;
    --brand:  hsl(221 83% 70%);
    --gap: clamp(1rem, 1rem + 1vw, 2rem);
    --radius: clamp(10px, 1vw, 16px);
    --shadow: 0 14px 40px rgba(0,0,0,.4);
  }
}

@layer base {
  * { box-sizing: border-box }
  body {
    margin: 0; background: var(--bg); color: var(--fg);
    font: 400 clamp(1rem, 0.95rem + 0.35vw, 1.125rem)/1.6 system-ui, -apple-system, Segoe UI, Roboto, 'Noto Sans KR', sans-serif;
  }
  .wrap {
    display: grid; gap: var(--gap);
    grid-template-columns: repeat(auto-fit, minmax(min(22rem, 100%), 1fr));
    inline-size: min(100%, 72rem);
    margin-inline: auto; padding: var(--gap);
  }
}

@layer components {
  @property --r { syntax: '<length>'; inherits: true; initial-value: 12px; }

  .card {
    background: var(--surface);
    border-radius: var(--r);
    padding: var(--gap);
    box-shadow: var(--shadow);
    transition: box-shadow .2s ease, transform .2s ease, --r .2s ease;
  }
  @media (hover:hover){
    .card:hover { transform: translateY(-2px); --r: calc(var(--radius) + 6px); }
  }
  .card h3 { margin: 0 0 .25em; font-weight: 700; }
  .card p  { margin: 0 0 1rem; color: var(--muted); }
  .card strong { font-size: clamp(1.25rem, 1rem + 2vw, 2rem); display: inline-block; margin-bottom: 1rem; }

  .btn {
    --btn-bg: var(--brand); --btn-fg: #fff;
    padding: .6rem 1.1rem; border: 0; border-radius: calc(var(--radius) - 2px);
    background: var(--btn-bg); color: var(--btn-fg); font-weight: 600;
    transition: filter .2s ease;
  }
  @media (hover:hover){
    .btn:hover { filter: brightness(1.1); }
  }

  .card.alt {
    --btn-bg: color-mix(in oklab, var(--brand) 12%, transparent);
    border: 1px solid color-mix(in oklab, var(--brand) 35%, transparent);
  }
}
```

사용법:
- **`t` 키**로 라이트/다크 전환.
- 모든 색/간격/코너/그림자는 **토큰 조합**이며, 컴포넌트는 **참조만** 함 → 전체적인 스킨 교체가 쉬움.

---

## 브라우저 지원 & 폴백

| 기능                         | Chrome/Edge | Firefox | Safari | IE11 |
|------------------------------|-------------|---------|--------|------|
| CSS Custom Properties         | ✅          | ✅      | ✅     | ❌   |
| `@property` (CSS Typed OM)    | ✅          | △(진행) | ✅     | ❌   |
| `color-mix()`                 | ✅          | ✅      | ✅     | ❌   |

- IE 대상이면 **SCSS 변수/전처리** + **빌드 타임 변환** 병행, 또는 **단색/고정값 폴백** 제공.
- 런타임 테마가 필수라면 **지원 브라우저 범위를 명시**(Browserslist)하고 폴백 UX를 설계.

---

## 참고/도구

- MDN: *Using CSS Custom Properties*, *@property*, *color-mix()*
- CSS-Tricks: *A Complete Guide to Custom Properties*
- DevTools: **Computed** 패널에서 `var(--*)` 해석 값 추적, **Emulate CSS media feature**(다크모드, 모션)
- PostCSS: `postcss-custom-properties`(정적 치환) — 런타임 변경 필요 시 **병행 전략** 고려.

---

## 마무리

CSS 커스텀 속성은 **설계 토큰 → 컴포넌트 스타일**의 연결고리입니다.
- **반응형/프리퍼런스/테마**를 “변수만 바꾸는” 방식으로 단순화하고,
- **@property**로 타입을 부여하면 **변수 자체 전환**도 가능해집니다.
팀에서는 **토큰 계층(원천/의미/컴포넌트) + 레이어 + 네이밍 규칙**을 정해 두면 **대규모 코드베이스**에서도 일관성과 변경 용이성을 동시에 확보할 수 있습니다.
