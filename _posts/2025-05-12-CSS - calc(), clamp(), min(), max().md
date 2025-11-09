---
layout: post
title: CSS - calc(), clamp(), min(), max()
date: 2025-05-12 20:20:23 +0900
category: CSS
---
# 📐 CSS 함수 정리: `calc()`, `clamp()`, `min()`, `max()`

## 0) 핵심 한 장 요약

- `calc()` : **단위 혼합/산술 연산**으로 “정확히 이만큼 빼기/더하기/비율” 같은 미세 조정.
- `clamp(min, ideal, max)` : **유동값(예: `vw`)**을 안전 범위 안에 가둬, **미디어쿼리 없는 반응형**.
- `min(a, b, …)` : **가장 작은 값** 선택 → “최대 너비 제한” 같은 상한선.
- `max(a, b, …)` : **가장 큰 값** 선택 → “최소 패딩/최소 글자크기” 같은 하한선.

> 공통 규칙: **연산 우선순위는 표준 수학과 동일**(곱·나눗셈 → 덧·뺄셈), `calc()`의 연산자 주변 **공백 필수**, `min/max/clamp` 인자는 **유효 길이/수치**여야 함.

---

## 1) `calc()` — 단위 혼합/산술의 만능 렌치

### 1.1 기본 문법/규칙

```css
.selector {
  width: calc(100% - 200px);
  margin-left: calc(1rem + 2vw);
  font-size: calc(1rem + 0.3vw);
}
```

- **연산자 좌우 공백** 필수: `calc(100%-2rem)` ❌ → `calc(100% - 2rem)` ✅
- **단위 혼합 가능**: `% + px`, `rem + vw`, `vh - px` 등.
- **우선순위**: `*` `/` → `+` `-` (괄호로 명시적 제어 권장).

### 1.2 퍼센트의 기준(percentage basis)

- `width`의 `%` → **근접 블록 컨테이너의 content-box** 기준.
- `height`의 `%` → 부모의 **계산된 height** 필요(미설정시 0처럼 행동) → `min-height`/`aspect-ratio`와 조합하거나 `dvh`/`svh` 등 사용.
- `padding/margin`의 `%` → **수평/수직 모두 컨테이너의 inline-size(가로)** 기준(전통 규칙).

### 1.3 실전 패턴

#### (A) 고정 사이드바 + 유동 본문

```html
<div class="wrap">
  <aside class="sidebar">Sidebar</aside>
  <main class="main">Main</main>
</div>
```

```css
.wrap { display: flex; }
.sidebar { inline-size: 280px; }
.main { inline-size: calc(100% - 280px); }
```

#### (B) 뷰포트 기반 배너 높이(주소창 변동 대응 포함)

```css
.banner {
  /* 폴백 */
  min-block-size: calc(100vh - 72px);
  /* 동적 뷰포트: 모바일 주소창 변동 반영 */
  min-block-size: calc(100dvh - 72px);
}
```

#### (C) 그리드 칼럼 간격을 고려해 균등 폭 만들기

```css
.grid {
  --cols: 3;
  --gap: 1rem;
  display: grid;
  gap: var(--gap);
  grid-template-columns: repeat(var(--cols), minmax(0, 1fr));
}
.card {
  /* gap을 고려한 내부 컨텐츠 폭 */
  inline-size: calc(100% - var(--gap));
}
```

> Tip: gap과 내부 여백이 중복되면 **overflow** 가능성이 있으니 `box-sizing: border-box`를 전역 적용 권장.

---

## 2) `clamp()` — 유동값을 안전한 범위로 “클램프”

### 2.1 문법과 의미

```css
/* 최소, 이상적(유동), 최대 */
font-size: clamp(1rem, 2.5vw, 2rem);
```

- **min ≤ ideal ≤ max** 가 되도록 브라우저가 자동 보정.
- 유동값이 작은 화면에서 너무 작아지거나 큰 화면에서 과하게 커지는 문제를 해결.

### 2.2 수학적 직관

유동 폰트 크기 \( f(w) \)를 뷰포트 너비 \( w \)에 대해 선형 근사로 두되, **하한 \( f_{\min} \)** 과 **상한 \( f_{\max} \)**을 둔다:

$$
f(w) = \mathrm{clamp}\big( f_{\min},\ a \cdot w + b,\ f_{\max} \big)
$$

- CSS에서는 `ideal` 자리에 예: `4vw` 같이 **뷰포트 비율값**을 넣어 선형 스케일을 구성.
- 브라우저가 자동으로 **하한/상한으로 잘라냄**.

### 2.3 실전 패턴

#### (A) 유동 타이포 스케일

```css
:root {
  --fs-body: clamp(1rem, 0.9rem + 0.5vw, 1.25rem);
  --fs-h1: clamp(1.75rem, 1.2rem + 3vw, 3rem);
  --fs-h2: clamp(1.375rem, 1.1rem + 1.8vw, 2.25rem);
}
body { font-size: var(--fs-body); }
h1 { font-size: var(--fs-h1); line-height: 1.15; }
h2 { font-size: var(--fs-h2); line-height: 1.2; }
```

#### (B) 컴포넌트 패딩/코너 반경 유동화

```css
.button {
  padding-inline: clamp(0.75rem, 0.5rem + 1vw, 1.5rem);
  padding-block: clamp(0.5rem, 0.4rem + 0.5vw, 0.9rem);
  border-radius: clamp(6px, 1vw, 14px);
}
```

#### (C) 카드 폭: 미디어쿼리 없이도 “줄어들되, 너무 작지 않게”

```css
.card {
  inline-size: clamp(280px, 45vw, 520px);
}
```

---

## 3) `min()` / `max()` — 상한/하한을 즉시 표현

### 3.1 문법

```css
width: min(100%, 1280px);   /* 1280px를 넘지 않는다(상한). */
padding: max(1rem, 3vw);    /* 1rem 보다 작아지지 않는다(하한). */
```

- 다수 인자도 가능: `min(a, b, c)`, `max(a, b, c)`

### 3.2 실전 패턴

#### (A) 래핑 컨테이너 최대 너비 제한

```css
.container {
  inline-size: min(100%, 72rem); /* 약 1152px 상한 */
  margin-inline: auto;
  padding-inline: clamp(1rem, 2vw, 2rem);
}
```

#### (B) 가독성 보장 패딩

```css
.section {
  padding-inline: max(1rem, 4vw); /* 작은 화면에서도 최소 1rem */
}
```

#### (C) `min()` + `max()` 중첩(복합 조건)

```css
.content {
  /* 화면이 아주 작으면 100%, 그렇지 않으면 60% 또는 640px 중 큰 값,
     그리고 전체 상한 1200px */
  inline-size: min(1200px, max(60%, 640px));
}
```

---

## 4) 함수 조합 레시피 — 흔한 요구를 간결하게

### 4.1 반응형 카드 그리드 (미디어쿼리 없음)

```css
.grid {
  --min: 16rem;          /* 카드 최소폭 */
  --gap: 1rem;

  display: grid;
  gap: var(--gap);
  grid-template-columns: repeat(auto-fit, minmax(min(100%, var(--min)), 1fr));
}
.card { padding: clamp(1rem, 1rem + 0.5vw, 2rem); }
```

- `auto-fit + minmax`로 그리드를 유동 구성.
- `min(100%, var(--min))`로 **작은 화면에서 칼럼 폭이 100%를 초과하지 않게** 제한.

### 4.2 헤더 높이 + 컨텐츠 뷰포트 맞춤

```css
:root { --header: clamp(56px, 6vw, 88px); }

header { block-size: var(--header); }
main {
  min-block-size: calc(100dvh - var(--header)); /* 동적 뷰포트 */
}
```

### 4.3 안전 영역(safe area)까지 고려한 패딩

```css
:root{
  --safe-t: env(safe-area-inset-top, 0px);
  --safe-r: env(safe-area-inset-right, 0px);
  --safe-b: env(safe-area-inset-bottom, 0px);
  --safe-l: env(safe-area-inset-left, 0px);
}
.footer {
  padding: clamp(1rem, 1rem + 1vw, 2rem);
  padding-bottom: calc(clamp(1rem, 1rem + 1vw, 2rem) + var(--safe-b));
}
```

### 4.4 이미지/비디오 박스: 종횡비 + 최대/최소 조합

```css
.media {
  aspect-ratio: 16 / 9;
  inline-size: min(100%, 960px);
  block-size: auto;
  border-radius: clamp(8px, 1vw, 16px);
}
```

---

## 5) 타이포그래피: 수학으로 보는 유동 디자인

### 5.1 선형 스케일의 `clamp()` 구현

목표: `w = 360px`에서 `f = 16px`, `w = 1440px`에서 `f = 22px`로 선형 증가.

선형식 \( f(w) = a w + b \) 에서
- \( a = \dfrac{22-16}{1440-360} = \dfrac{6}{1080} = \dfrac{1}{180} \approx 0.00556 \,\text{px/px} \)
- \( b = f(360) - a \cdot 360 = 16 - 2 = 14 \)

이를 `vw`로 변환(뷰포트 100vw = 화면 너비 \( w \)):
- \( a w = 0.00556 \cdot w \) → `0.556vw` (왜냐하면 `1vw = w/100`)
- 최종: `clamp(16px, 14px + 0.556vw, 22px)`

```css
:root { --fs-body: clamp(16px, 14px + 0.556vw, 22px); }
body { font-size: var(--fs-body); }
```

> 실제로는 `rem` 기준으로 재조정하거나, `vw` 비율을 소수점 3자리 내외로 반올림해 실무 가독성 확보.

---

## 6) 레이아웃과의 상호작용 — Flex/Grid/Aspect-ratio

### 6.1 Flex 컨테이너에서의 `%`/`calc()`

- Flex 아이템의 `flex-basis` 에 `calc()` 사용 가능.
- `flex-basis: calc(50% - 1rem);` 로 두 칸 배치 시 gap 대체 가능(요즘은 그냥 `gap` 쓰는 게 더 명확).

```css
.flex {
  display: flex;
  flex-wrap: wrap;
  gap: 1rem;
}
.flex > .item { flex: 1 1 calc(50% - 1rem); }
@media (max-width: 640px){
  .flex > .item { flex-basis: 100%; }
}
```

### 6.2 Grid 트랙과 `minmax()` + 함수 혼합

```css
.grid {
  display: grid;
  gap: 1rem;
  grid-template-columns: repeat(auto-fit, minmax(min(22rem, 100%), 1fr));
}
```

- 작은 화면에서는 단일 컬럼(100%),
- 충분히 넓으면 22rem을 하한으로 유지하며 자동 래핑.

### 6.3 `aspect-ratio`와 min/max

```css
.thumb {
  aspect-ratio: 1;
  inline-size: clamp(120px, 20vw, 240px);
  block-size: auto;
}
```

---

## 7) 상태/상호작용과의 조합

### 7.1 :hover/:focus-visible와 애니메이션

```css
.card {
  transition: transform .25s ease, box-shadow .25s ease;
  transform: translateY(0);
}
@media (hover: hover){
  .card:hover {
    transform: translateY(clamp(-2px, -0.2vw, -6px));
    box-shadow: 0 clamp(6px, 0.6vw, 16px) clamp(24px, 2vw, 40px) rgba(0,0,0,.12);
  }
}
.card:focus-visible { outline: max(2px, 0.2vw) solid #2563eb; }
```

### 7.2 스크롤 잠금/안전 높이(모달)

```css
body.modal-open { overflow: hidden; }
.modal {
  inline-size: min(640px, 92vw);
  max-block-size: calc(100dvh - max(2rem, 4vh));
  overflow: auto;
  padding: clamp(1rem, 2vw, 2rem);
}
```

---

## 8) 디버깅/퍼포먼스 체크포인트

- `calc()` 중첩 과다/복잡한 네스팅은 **읽기성↓** → 디자인 토큰(변수)로 **분해**.
- `%`의 기준이 모호하면 DevTools로 **computed** 확인. 부모의 `position/height/overflow`가 영향.
- 함수는 **레이아웃 계산 시점**에 평가 → 값이 빈번히 변하는 속성(vw/vh) 남용 시 리플로우/리페인트 비용 고려.
- 폰트 스케일 유동(`clamp(vw)`)은 **CLS(레이아웃 이동)**를 유발하지 않게 line-height·컨테이너 높이 전략을 함께.

---

## 9) IE/레거시 폴백(정책적으로 필요한 경우)

- `calc()` : IE9+ 지원(가능)  
- `clamp()/min()/max()` : **IE 미지원**(폴리필 불가) → **미디어쿼리**/고정 값 **대체**.

예) `clamp(1rem, 2vw, 2rem)` 폴백:

```css
.title { font-size: 1rem; }            /* 기본(작은 화면) */
@media (min-width: 800px){
  .title { font-size: 2rem; }          /* 큰 화면 */
}
/* 중간 구간은 선택적으로 여러 브레이크포인트 배치 */
```

> 기업 인트라/키오스크 등 특정 브라우저 타겟이면 **Browserslist**로 빌드 타겟 명시.

---

## 10) 컴포넌트 예제 — “절대 안 깨지는” 카드는 이렇게

```html
<section class="wrap">
  <article class="card">
    <h3>Card Title</h3>
    <p>본문은 가독성 우선으로 유동 타이포와 여백이 적용됩니다.</p>
    <button class="btn">Action</button>
  </article>
  <article class="card">
    <h3>Another Card</h3>
    <p>상/하한을 명시해 과도한 확대/축소를 방지합니다.</p>
    <button class="btn">Action</button>
  </article>
</section>
```

```css
:root{
  --radius: clamp(8px, 1vw, 16px);
  --pad: clamp(1rem, 2vw, 2rem);
  --fs-body: clamp(1rem, 0.95rem + 0.35vw, 1.125rem);
  --fs-h3: clamp(1.125rem, 0.9rem + 1.2vw, 1.5rem);
}

*{ box-sizing: border-box }
body{ margin: 0; font-family: system-ui, -apple-system, Segoe UI, Roboto, 'Noto Sans KR', sans-serif; }

.wrap{
  display: grid;
  gap: clamp(1rem, 2vw, 2rem);
  grid-template-columns: repeat(auto-fit, minmax(min(22rem, 100%), 1fr));
  padding: var(--pad);
  inline-size: min(100%, 72rem);
  margin-inline: auto;
}

.card{
  background: #fff;
  border-radius: var(--radius);
  padding: var(--pad);
  box-shadow: 0 clamp(6px, 1vw, 16px) clamp(24px, 2vw, 40px) rgba(0,0,0,.08);
}
.card h3{ font-size: var(--fs-h3); margin: 0 0 .5em; }
.card p{ font-size: var(--fs-body); margin: 0 0 1em; line-height: 1.7; }

.btn{
  display: inline-flex; align-items: center; justify-content: center;
  padding: clamp(0.5rem, 0.4rem + 0.6vw, 0.9rem) clamp(1rem, 0.8rem + 1vw, 1.5rem);
  border-radius: calc(var(--radius) - 2px);
  border: 0;
  background: hsl(221 83% 53%);
  color: #fff; font-weight: 600;
  transition: transform .2s ease, box-shadow .2s ease;
}
@media (hover:hover){
  .btn:hover{
    transform: translateY(clamp(-1px, -0.2vw, -3px));
    box-shadow: 0 10px 24px rgba(0,0,0,.16);
  }
}
```

- **그리드**: `auto-fit` + `minmax(min(22rem, 100%), 1fr)` → 한 칼럼 폭이 22rem 이하로 줄면 자동 래핑.
- **모든 스케일**: `clamp()`로 상/하한 지정 → 과도한 수축/확대 방지.
- **여백/코너**도 유동화 → 기기별 일관적 시각 리듬.

---

## 11) 디버깅/검증 체크리스트

- [ ] `calc()` 연산자 **공백** 확인.
- [ ] `%` 기준이 모호하면 DevTools **Computed/Box Model**로 부모 계산 확인.
- [ ] `clamp()`의 `min ≤ max` 위배 여부.
- [ ] `min()/max()` 인자에 **단위 미스매치** 없는지(길이/길이, 수/수).
- [ ] `vw/vh` 사용 시 모바일에서 **주소창 변화** 대응(`100dvh`) 여부.
- [ ] 폰트 유동화로 **줄바꿈/오버플로우**가 생기지 않는지(최대값 하향 조정).
- [ ] 레거시 타겟이면 **미디어쿼리 폴백** 추가.

---

## 부록 A) 자주 쓰는 스니펫 모음

### A-1. 중앙 래핑 컨테이너(패딩·상한 포함)

```css
.container {
  inline-size: min(100%, 72rem);
  margin-inline: auto;
  padding-inline: clamp(1rem, 2vw, 2rem);
}
```

### A-2. 읽기 폭 제어(문장 길이 제한)

```css
.prose {
  max-inline-size: min(65ch, 95vw);
  font-size: clamp(1rem, 0.9rem + 0.4vw, 1.125rem);
  line-height: 1.7;
}
```

### A-3. 카드 폭과 갭의 균형

```css
.cards {
  --w: clamp(18rem, 40vw, 24rem);
  --gap: clamp(0.75rem, 2vw, 1.25rem);
  display: grid;
  gap: var(--gap);
  grid-template-columns: repeat(auto-fill, minmax(min(var(--w), 100%), 1fr));
}
```

### A-4. 헤더/푸터 제외 영역 자동 채우기

```css
:root { --header: 64px; --footer: 56px; }
main { min-block-size: calc(100dvh - (var(--header) + var(--footer))); }
```

---

## 부록 B) 수학 메모 — `clamp()`로 구간 선형 보간 만들기

두 점 \((w_1, f_1)\), \((w_2, f_2)\)에서 선형식을 만들면:

\[
a = \frac{f_2 - f_1}{w_2 - w_1}, \quad b = f_1 - a w_1,\quad f(w) = a w + b
\]

이를 CSS로:

```css
/* 360px→16px, 1440px→22px 예시 */
:root {
  --f-min: 16px;
  --f-max: 22px;
  --ideal: calc(14px + 0.556vw); /* b + a·w (vw로 변환) */
}
.title { font-size: clamp(var(--f-min), var(--ideal), var(--f-max)); }
```

---

## 레퍼런스/추가 읽을거리

- MDN: [`calc()`](https://developer.mozilla.org/en-US/docs/Web/CSS/calc), [`clamp()`](https://developer.mozilla.org/en-US/docs/Web/CSS/clamp), [`min()`/`max()`](https://developer.mozilla.org/en-US/docs/Web/CSS/min)
- CSS-Tricks: Fluid/Responsive Typography, Modern CSS Functions
- “Intrinsic & Extrinsic Sizing” (MDN): 퍼센트/콘텐츠 기반 크기 산정

---

## 마무리

- 네 함수는 **레이아웃·타이포·여백**의 “연속적 설계(continuity)”를 가능케 해, **미디어쿼리 개수**와 **분기 복잡도**를 크게 줄입니다.
- 실무에서는 **디자인 토큰화(변수)**와 함께 사용해 **읽기성/재사용성**을 개선하고, 모바일 뷰포트·안전영역·접근성(가독 폭/대비/포커스 링)을 함께 고려하면 **깨지지 않는 반응형**을 일관되게 달성할 수 있습니다.