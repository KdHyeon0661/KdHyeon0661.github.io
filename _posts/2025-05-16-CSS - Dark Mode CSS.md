---
layout: post
title: CSS - Dark Mode CSS
date: 2025-05-16 21:20:23 +0900
category: CSS
---
# Dark Mode CSS 구현 방법 완전 정리

다크 모드는 **눈의 피로 저감**, **OLED 배터리 절감**, **야간/저조도 환경 최적화**라는 분명한 목적을 갖습니다.

- 구현 3패턴: **`prefers-color-scheme`**, **클래스 토글**, **CSS 변수 + JS(권장)**
- **FOUC(잘못된 테마가 번쩍 보이는 현상) 방지**, 초기 페인트 전략
- **접근성(명도 대비·모션·강제 색상)** 대응
- **이미지/SVG/아이콘** 다크 자산 처리
- 네이티브 컨트롤/스크롤바/브라우저 UI 색 동기화
- **디자인 토큰 구조**, 컴포넌트 스케일 전략

---

## 빠른 요약 — 상황별 최적 선택

| 상황 | 추천 방식 | 이유 |
|---|---|---|
| 시스템 설정만 따르고 버튼은 불필요 | `@media (prefers-color-scheme)` | JS 없이 가장 간단·견고 |
| 사용자 토글 필요하지만 규모 작음 | `body.dark/.light` 클래스 토글 | 코드 이해가 쉽고 도입 빠름 |
| 규모가 크고 테마 확장/브랜딩 중요 | **CSS 변수 + `[data-theme]`** | 토큰 기반 확장성·성능·유지보수 최적 |

---

## 시스템 다크 모드 자동 감지 — `prefers-color-scheme`

### 기본 사용

```css
/* 라이트 기본 */
:root {
  color-scheme: light dark; /* 폼/스크롤 등 네이티브도 라이트/다크 지원 */
}
body { background:#ffffff; color:#000000; }

/* OS가 다크를 선호할 때 */
@media (prefers-color-scheme: dark) {
  body { background:#121212; color:#ffffff; }
}
```

**포인트**
- `color-scheme: light dark;`를 루트에 선언하면 **폼 컨트롤/스크롤바** 등 네이티브 UI도 자동 테마링.
- JS 불필요, 단 **사용자 오버라이드(토글)**는 제공되지 않음.

### 브라우저 UI 동기화

```html
<!-- 브라우저 툴바 색 동기화 -->
<meta name="theme-color" media="(prefers-color-scheme: light)" content="#ffffff">
<meta name="theme-color" media="(prefers-color-scheme: dark)"  content="#121212">
```

---

## 클래스 기반 토글 — 구현은 단순, 확장은 제한

### 최소 구현 (학습/프로토타입에 적합)

```html
<body class="light">
  <button id="toggle">테마 전환</button>
</body>
```

```css
body.light { background:#fff; color:#111; }
body.dark  { background:#121212; color:#fff; }
```

```js
const $b=document.body,$t=document.getElementById('toggle');
$t.addEventListener('click',()=>{
  $b.classList.toggle('dark');
  $b.classList.toggle('light');
});
```

### 로컬 저장 + 초기 적용 (FOUC 방지 X)

```js
// 저장
localStorage.setItem('theme', document.body.classList.contains('dark')?'dark':'light');

// 초기 적용(간단 버전)
document.body.classList.add(localStorage.getItem('theme') || 'light');
```
> 클래스 방식은 **색·간격·그림자 등 속성 중복**이 늘어 확장성이 떨어질 수 있습니다. 대규모 시스템에는 **CSS 변수 방식**이 더 적합.

---

## CSS 변수 + JS — 추천 아키텍처 (확장성/성능/유지보수 우수)

### 디자인 토큰(변수) 정의

```css
/* 공통 토큰 */
:root {
  --bg: #ffffff;
  --text: #111111;
  --muted: #6b7280;
  --surface: #f7f7f8;
  --brand: #2563eb;
  --border: #e5e7eb;
  --focus: #3b82f6;
  color-scheme: light dark; /* 네이티브 제어 */
}

/* 다크 토큰 */
[data-theme="dark"] {
  --bg: #121212;
  --text: #eaeaea;
  --muted: #a3a3a3;
  --surface: #1b1b1b;
  --brand: #8b5cf6;
  --border: #2b2b2b;
  --focus: #a78bfa;
}

html, body {
  background: var(--bg);
  color: var(--text);
}
```

### 컴포넌트는 **항상 변수만** 참조

```css
.card {
  background: var(--surface);
  border: 1px solid var(--border);
  color: var(--text);
  border-radius: 1rem;
  padding: 1rem;
}
.button {
  background: var(--brand);
  color: #fff;
  border: 1px solid transparent;
  border-radius: .75rem;
  padding: .6rem .9rem;
}
```

### 토글 + 로컬 저장 + 초기 페인트 최적화(FOUC 방지 ✔)

```html
<!-- <head> 최상단 **인라인**: CSS 로딩 전 data-theme 확정 -->
<script>
(function(){
  try{
    var saved = localStorage.getItem('theme');
    if(!saved){
      // 시스템 선호 반영 (초기 기본값)
      var isDark = window.matchMedia && window.matchMedia('(prefers-color-scheme: dark)').matches;
      saved = isDark ? 'dark' : 'light';
    }
    document.documentElement.setAttribute('data-theme', saved);
  }catch(e){}
})();
</script>
```

```html
<button id="toggle-theme">테마 전환</button>
<script>
const html = document.documentElement;
document.getElementById('toggle-theme').addEventListener('click', ()=>{
  const next = html.getAttribute('data-theme') === 'dark' ? 'light' : 'dark';
  html.setAttribute('data-theme', next);
  try { localStorage.setItem('theme', next); } catch(e){}
});
</script>
```

**왜 인라인 스니펫이 head 상단에 필요한가?**
- CSS가 로드되기 전에 `data-theme`를 설정하면 **라이트→다크 순간 깜빡임(FOUC)**을 제거.

### 시스템 변경 실시간 반영(옵션)

```js
// 사용자 저장값이 없을 때만 OS 변화를 따라감
const mq = window.matchMedia('(prefers-color-scheme: dark)');
mq.addEventListener?.('change', e=>{
  if(!localStorage.getItem('theme')) {
    document.documentElement.setAttribute('data-theme', e.matches?'dark':'light');
  }
});
```

---

## 접근성(A11y)·명도 대비·모션·강제 색상

### 명도 대비(AA/AAA) 체크

웹 콘텐츠 접근성 가이드라인(WS 2.1) 대비비는 다음과 같습니다:

\[
\text{Contrast Ratio} = \frac{L_1 + 0.05}{L_2 + 0.05} \quad (L_1 \ge L_2)
\]
\[
L = 0.2126\,R_s + 0.7152\,G_s + 0.0722\,B_s
\]
여기서 \(R_s,G_s,B_s\)는 sRGB를 선형 공간으로 변환한 값. **본문 4.5:1 이상(AA)**, **대문자 큰 텍스트 3:1 이상** 권장.

실무 팁:
- 다크 배경은 **완전 블랙(#000)** 대신 **오프 블랙(#121212 등)** + 대비 컬러 조합이 가독성 우수.
- 링크/강조는 **채도·명도 차**를 충분히 두고, `:focus-visible`로 포커스 링 제공.

```css
:root { --focus:#3b82f6; }
a:focus-visible, button:focus-visible {
  outline: 2px solid var(--focus);
  outline-offset: 2px;
}
```

### 모션 민감 사용자

```css
@media (prefers-reduced-motion: reduce) {
  * { animation: none !important; transition: none !important; }
}
```

### 강제 색상(Windows High Contrast)

```css
@media (forced-colors: active) {
  .button {
    forced-color-adjust: auto; /* 기본값: 사용자의 고대비 테마 존중 */
    border: 1px solid ButtonText;
    background: Canvas;
    color: ButtonText;
  }
}
```

---

## 이미지/SVG/아이콘 — 다크 모드 최적 처리

### 사진·썸네일은 **색 반전 지양**, 별도 자산 권장

```html
<picture>
  <source srcset="thumb-dark.avif" media="(prefers-color-scheme: dark)" type="image/avif">
  <img src="thumb-light.avif" alt="미리보기">
</picture>
```

### 단색 아이콘은 **currentColor**

```svg
<!-- inline SVG -->
<svg width="20" height="20" viewBox="0 0 24 24" aria-hidden="true">
  <path fill="currentColor" d="..."/>
</svg>
```
```css
.icon { color: var(--text); } /* 테마 색 자동 동기화 */
```

### 배경 이미지(일러스트) 전환

```css
.hero {
  background: url(/img/hero-light.svg) center/cover no-repeat;
}
@media (prefers-color-scheme: dark) {
  .hero {
    background-image: url(/img/hero-dark.svg);
  }
}
```

---

## 네이티브 컨트롤·스크롤바·양식

### `color-scheme`로 네이티브 동기화

```css
:root { color-scheme: light dark; } /* 이미 위에서 사용 */
```

### 스크롤바

```css
/* Firefox */
* { scrollbar-width: thin; scrollbar-color: var(--muted) transparent; }

/* WebKit */
*::-webkit-scrollbar        { width: 10px; height:10px; }
*::-webkit-scrollbar-thumb  { background: var(--muted); border-radius: 8px; }
*::-webkit-scrollbar-track  { background: transparent; }
```

---

## 트랜지션(부드러운 전환)과 성능

```css
html, body, .card, .button {
  transition: background-color .25s ease, color .25s ease, border-color .25s ease;
}
```

**주의**
- 테마 토글 시 **layout-affecting 속성**(예: `box-shadow` 과도 변경, 이미지 필터)은 프레임 드랍 유발 가능.
- **GPU 친화 속성**(color/background/border-color/opacity/transform) 위주로 전환.

---

## FOUC(테마 깜빡임) 완전 방지 전략

1) **head 최상단 인라인 스니펫**으로 `data-theme` 즉시 설정(§3.3).
2) CSS 번들 상단에 토큰 정의.
3) SSR 환경에선 서버에서 쿠키/설정으로 초깃값 렌더.

**SSR 예시(개념):**

{% raw %}
```html
<html data-theme="{{cookie.theme || systemPref}}">
```
{% endraw %}

---

## 컴포넌트/페이지 스케일 — BEM/레이어와 결합

```css
@layer tokens, base, components, utilities;

/* tokens */
@layer tokens {
  :root { /* --bg, --text, --brand ... */ }
  [data-theme="dark"] { /* 다크 토큰 */ }
}

/* components */
@layer components {
  .card { background:var(--surface); border:1px solid var(--border); }
  .button--primary { background:var(--brand); color:#fff; }
}
```

**장점**: `@layer`로 **특이도 충돌 최소화**, 테마 교체 시 **토큰 레이어만** 바꾸면 전체가 바뀜.

---

## 실제 페이지 뼈대 예제 — 토글 + 시스템 추적 + 이미지 전환

```html
<meta name="color-scheme" content="light dark">
<meta name="theme-color" media="(prefers-color-scheme: light)" content="#ffffff">
<meta name="theme-color" media="(prefers-color-scheme: dark)"  content="#121212">

<!-- 초기 FOUC 방지 -->
<script>
(function(){
  try{
    var s=localStorage.getItem('theme');
    if(!s){
      s = (window.matchMedia && matchMedia('(prefers-color-scheme: dark)').matches) ? 'dark' : 'light';
    }
    document.documentElement.setAttribute('data-theme', s);
  }catch(e){}
})();
</script>

<header class="site-header">
  <button id="toggle-theme" class="button">테마 전환</button>
</header>

<main class="content">
  <picture>
    <source srcset="hero-dark.svg" media="(prefers-color-scheme: dark)">
    <img src="hero-light.svg" alt="히어로 일러스트">
  </picture>

  <article class="card">
    <h2 class="card__title">다크 모드 카드</h2>
    <p class="card__text">테마 토큰으로 색이 바뀝니다.</p>
    <button class="button button--primary">액션</button>
  </article>
</main>

<script>
const html=document.documentElement, btn=document.getElementById('toggle-theme');
btn.addEventListener('click', ()=>{
  const next=html.getAttribute('data-theme')==='dark'?'light':'dark';
  html.setAttribute('data-theme', next);
  try{ localStorage.setItem('theme', next);}catch(e){}
});
if(!localStorage.getItem('theme') && window.matchMedia){
  const mq=matchMedia('(prefers-color-scheme: dark)');
  mq.addEventListener?.('change', e=>{
    html.setAttribute('data-theme', e.matches?'dark':'light');
  });
}
</script>
```

```css
:root{
  --bg:#fff; --text:#111; --surface:#f7f7f8; --muted:#6b7280; --brand:#2563eb; --border:#e5e7eb; --focus:#3b82f6;
  color-scheme: light dark;
}
[data-theme="dark"]{
  --bg:#121212; --text:#eaeaea; --surface:#1b1b1b; --muted:#a3a3a3; --brand:#8b5cf6; --border:#2b2b2b; --focus:#a78bfa;
}
html,body{ background:var(--bg); color:var(--text); margin:0; }
*{ box-sizing:border-box; }
.content{ padding:clamp(1rem, 3vw, 2rem); display:grid; gap:1rem; }
.card{
  background:var(--surface); border:1px solid var(--border); border-radius:1rem; padding:1rem;
}
.card__title{ margin:0 0 .25rem 0; }
.button{
  background:var(--brand); color:#fff; border:1px solid transparent; border-radius:.75rem; padding:.6rem .9rem;
  transition: background-color .2s, color .2s, border-color .2s;
}
.button:focus-visible{ outline:2px solid var(--focus); outline-offset:2px; }
```

---

## QA 체크리스트 (릴리즈 전 점검)

- [ ] 초기 로딩에서 **FOUC**가 없는가? (라이트→다크 깜빡임)
- [ ] `color-scheme`로 폼/스크롤바가 테마에 맞는가?
- [ ] 본문/링크/보더/아이콘 **모두** 테마 토큰을 사용했는가?
- [ ] 이미지/일러스트에 **다크 전용** 자산(또는 적절한 처리)이 있는가?
- [ ] 대비비(본문 4.5:1 이상) 충족? 포커스 표시 명확?
- [ ] `prefers-reduced-motion`, `forced-colors` 대응?
- [ ] `meta theme-color`가 테마별로 설정되었는가?
- [ ] 토글 후 선택이 **localStorage**에 저장/복원되는가?
- [ ] 시스템 테마 변경 시(저장값 없을 때) 실시간 동기화되는가?

---

## 문제 해결 모음

- **라이트 색이 잠깐 보임** → 인라인 스니펫로 `data-theme` 선반영(§3.3).
- **네이티브 컨트롤 라이트로 고정** → `:root{ color-scheme: light dark; }`.
- **사진/썸네일이 뜨는 느낌** → 다크 전용 자산 준비, 반전 필터는 피로감↑.
- **SVG 색 통일 안 됨** → `fill="currentColor"`로 통일, CSS `color`로 제어.
- **전환 시 버벅임** → 전환 속성 축소, 이미지 필터/박스섀도우 남용 금지.

---

## 확장: 멀티 테마(브랜드/하이콘트라스트)

```css
/* 브랜드 B 테마 */
[data-theme="brand-b"]{
  --brand:#10b981; --focus:#34d399;
  --bg:#0c0f0d; --surface:#121614; --text:#e8f5ee; --muted:#a3b2aa; --border:#1f2a24;
}
```
```js
// 사용 예: data-theme = 'light' | 'dark' | 'brand-b' …
```

---

## 결론 — 추천 기본 템플릿

1) **CSS 변수 토큰**으로 색·보더·표면·포커스 정의
2) `<html data-theme="...">` + **인라인 초기화 스니펫**으로 FOUC 제거
3) `color-scheme: light dark`로 네이티브 제어
4) 이미지/SVG는 **다크 자산** 또는 `currentColor` 활용
5) 대비/모션/강제색 등 접근성 미디어쿼리 대응
6) 컴포넌트는 **오직 변수만** 참조 → 테마 확장/브랜딩 시 유지보수 최소화

---

## 참고 링크

- MDN — `@media (prefers-color-scheme)`: https://developer.mozilla.org/en-US/docs/Web/CSS/@media/prefers-color-scheme
- CSS-Tricks — Dark Mode: https://css-tricks.com/dark-modes-with-css/
- web.dev — Dark Theme: https://web.dev/prefers-color-scheme/
