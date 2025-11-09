---
layout: post
title: CSS - Sass와 CSS
date: 2025-05-12 21:20:23 +0900
category: CSS
---
# Sass와 CSS의 차이점

## 0) 한눈에 보는 결론

- **CSS**: 런타임(브라우저)에서 동작. 표준/호환성/간결함. 최신 사양(`:has()`, `@layer`, 컨테이너쿼리, 부분적 네스팅 등)도 빠르게 채택 중.  
- **Sass(SCSS)**: 컴파일 타임에 **구조화/재사용/자동화**를 돕는 도구. 큰 코드베이스/디자인시스템에서 **토큰·유틸 자동생성**, **모듈화**, **반복 제거**에 탁월.  
- **정답은 “둘 다”**: **설계 토큰·유틸리티·반복 생성**은 Sass, **런타임 테마/환경 선호도**는 CSS 변수와 현대 CSS로 처리하는 **하이브리드 전략**이 실무 최적.

---

## 1) 기본 개념 비교

| 구분 | CSS | Sass(SCSS) |
|---|---|---|
| 정의 | 웹 표준 스타일 언어 | CSS를 확장한 **전처리기** (SCSS 문법이 CSS와 거의 동일) |
| 실행 | **브라우저가 직접** 해석 | **컴파일**로 CSS 변환 후 브라우저가 해석 |
| 문법 | 표준 CSS | **변수/중첩/믹스인/함수/제어문/모듈(@use/@forward)** |
| 목적 | 문서 스타일링 | **대규모/복잡** 스타일의 **구조화·재사용·자동화** |

---

## 2) 기능 차이 핵심 요약

| 기능 | CSS | Sass(SCSS) |
|---|---|---|
| 변수 | `var(--x)` (런타임 변수) | `$x` (컴파일 변수) |
| 중첩 | 일부 최신 브라우저에서 **CSS Nesting** 지원, 범위/문법 제약 | 자유로운 **선택자 중첩** |
| 믹스인 | 없음 | `@mixin` / `@include` |
| 함수 | 내장(예: `calc()`, `min()`) 중심 | **사용자 함수** `@function` 지원 |
| 연산 | `calc()` 중심(런타임) | 산술 `+ - * /` (컴파일) |
| 조건/반복 | 없음 | `@if`, `@each`, `@for`, `@while` |
| 모듈 | `@import`(CSS 파일) / `@layer` | **`@use` / `@forward` (권장)** |
| 출력 | 고정 | `expanded` / `compressed` 등 **빌드 출력 제어** |

> 포인트: **CSS 변수는 런타임 동적 변경**, **Sass 변수는 컴파일 정적 치환**. 두 세계의 장점을 **결합**하는 설계가 관건.

---

## 3) 예제로 보는 차이

### 3.1 변수

**CSS(런타임·테마 친화):**
```css
:root { --brand: #2563eb; }
[data-theme="dark"] { --brand: #8b5cf6; }

.button { background: var(--brand); }
```

**Sass(설계 토큰·반복 제거):**
```scss
$brand: #2563eb;

.button {
  background: $brand;
}
```

### 3.2 중첩

**CSS(네스팅 신문법의 한 예):**
```css
.card {
  color: #111;
  & > h3 { font-weight: 700; }
  &:hover { filter: brightness(1.05); }
}
```

**Sass(안정된 중첩 + BEM 보일러플레이트 단축):**
```scss
.card {
  color: #111;

  &__title { font-weight: 700; }
  &:hover { filter: brightness(1.05); }
}
```

### 3.3 믹스인/함수/제어문 (Sass만)

```scss
@mixin flex-center { display:flex; align-items:center; justify-content:center; }

@function rem($px, $base: 16px) { @return ($px / $base) * 1rem; }

$spacings: (sm: .5rem, md: 1rem, lg: 1.5rem);

.button {
  @include flex-center;
  padding: rem(16) rem(20);
}

@each $k, $v in $spacings {
  .m-#{$k} { margin: $v; }
}
```

---

## 4) Sass 모듈 시스템: `@use` / `@forward` (실무 필수)

> `@import`는 비권장. **네임스페이스 / API 관리**를 위해 `@use`/`@forward` 사용.

**토큰과 유틸 모듈화 예시**

```scss
/* scss/abstracts/_tokens.scss */
$colors: ("brand": #2563eb, "danger": #ef4444);
$bp: ("md": 768px, "lg": 1024px);

@function color($name) { @return map-get($colors, $name); }
@function bp($key) { @return map-get($bp, $key); }

/* scss/abstracts/_mixins.scss */
@mixin up($k) { @media (min-width: bp($k)) { @content; } }

/* scss/abstracts/_index.scss */
@forward "tokens";
@forward "mixins";

/* scss/main.scss */
@use "abstracts" as a;

.button {
  background: a.color("brand");
  @include a.up(md) { padding: 1rem 1.25rem; }
}
```

---

## 5) CSS 변수와 Sass의 **하이브리드 전략**

- **Sass**로 **설계 토큰·유틸리티/컴포넌트 CSS 생성**,  
- **CSS 변수**로 **런타임 테마/환경 선호도** 대응.

**하이브리드 예시**

```scss
/* SCSS에서 전역 기본값 선언 */
$radius: 12px;
$brand: #2563eb;

/* CSS 변수로 노출: 런타임 오버라이드 허용 */
:root {
  --radius: #{$radius};
  --brand: #{$brand};
}

/* 사용부 */
.card { border-radius: var(--radius); }
.button { background: var(--brand); }

/* 다크 테마 런타임 전환(자바스크립트 or HTML 속성) */
:root[data-theme="dark"] {
  --brand: #8b5cf6;
}
```

> 장점: 디자인시스템은 SCSS로 **일관성/자동화**, 사용자/환경 변화는 CSS 변수로 **즉시 적용**.

---

## 6) 반응형·타이포·레이아웃 토큰 패턴 (Sass 베스트)

### 6.1 브레이크포인트 맵 + 믹스인

```scss
$bp: ("sm": 640px, "md": 768px, "lg": 1024px);

@mixin up($k) {
  $v: map-get($bp, $k);
  @if $v { @media (min-width: $v) { @content; } }
}

.grid {
  display: grid; grid-template-columns: 1fr; gap: 1rem;
  @include up(md) { grid-template-columns: repeat(2, 1fr); }
  @include up(lg) { grid-template-columns: repeat(3, 1fr); }
}
```

### 6.2 Fluid 타이포(미디어쿼리 없음, `clamp`)

```scss
@function fluid($min, $max, $vw-min: 360, $vw-max: 1280) {
  $slope: ($max - $min) / ($vw-max - $vw-min) * 100;
  $intercept: $min - ($slope * $vw-min / 100);
  @return clamp(#{$min}, #{$intercept}rem + #{$slope}vw, #{$max});
}

h1 { font-size: fluid(1.5rem, 3rem); } /* 화면폭에 비례해 부드럽게 */
```

> 수식 개념:  
> $$ f(w)=\mathrm{clamp}\big(f_{\min}, a\cdot w + b, f_{\max}\big),\quad a=\frac{f_{\max}-f_{\min}}{vw_{\max}-vw_{\min}} $$

### 6.3 컨테이너 쿼리와 SCSS

```scss
.card-list { container-type: inline-size; }

@container (inline-size > 42rem) {
  .card { display: grid; grid-template-columns: 1fr auto; }
}
```

---

## 7) 실전 미니 프로젝트(구조·예제)

```
scss/
├─ abstracts/
│  ├─ _tokens.scss
│  ├─ _mixins.scss
│  └─ _functions.scss
├─ utilities/
│  ├─ _spacing.scss
│  └─ _colors.scss
├─ components/
│  ├─ _button.scss
│  └─ _card.scss
└─ main.scss
```

**토큰/믹스인**

```scss
/* abstracts/_tokens.scss */
$colors: ("brand": #2563eb, "muted": #6b7280, "danger": #ef4444);
$spaces: (0: 0, 1: .25rem, 2: .5rem, 3: .75rem, 4: 1rem);

/* abstracts/_mixins.scss */
@mixin focus-ring($c: #2563eb) {
  outline: 2px solid $c; outline-offset: 2px;
}
```

**유틸 자동 생성**

```scss
/* utilities/_spacing.scss */
@use "../abstracts/tokens" as t;

@each $k, $v in t.$spaces {
  .m-#{$k} { margin: $v; }
  .p-#{$k} { padding: $v; }
}
```

**컴포넌트**

```scss
/* components/_button.scss */
@use "../abstracts/tokens" as t;
@use "../abstracts/mixins" as m;

.button {
  display:inline-flex; align-items:center; justify-content:center;
  padding:.6rem 1rem; border:1px solid transparent; border-radius:.75rem;
  background: map-get(t.$colors, "brand"); color:#fff; font-weight:600;
  transition: filter .2s ease, transform .15s ease;

  &:hover { filter: brightness(1.05); }
  &:active { transform: translateY(1px); }
  &:focus-visible { @include m.focus-ring(); }
}
```

**엔트리**

```scss
/* main.scss */
@use "abstracts/tokens";
@use "utilities/spacing";
@use "components/button";
@use "components/card";
```

---

## 8) 빌드/도구 세팅(요점)

### 8.1 Dart Sass (CLI)

```bash
npm i -D sass
sass scss/main.scss public/assets/main.css --source-map
sass scss/main.scss public/assets/main.css --style=compressed
sass --watch scss:public/assets
```

### 8.2 PostCSS + Autoprefixer

`postcss.config.cjs`
```js
module.exports = { plugins: { autoprefixer: {} } };
```

`package.json` Browserslist 예:
```json
"browserslist": ["defaults", "not IE 11"]
```

### 8.3 Vite/Webpack/Gulp
- **Vite**: `preprocessorOptions.scss.additionalData`로 전역 토큰 주입 가능.  
- **Webpack**: `sass-loader` + `postcss-loader` + `css-loader` + `MiniCssExtractPlugin`.  
- **Gulp**: `gulp-sass` + `autoprefixer` + `clean-css` + sourcemaps.

---

## 9) 품질: Stylelint/Prettier/레이어링

- **Stylelint(standard-scss)**로 규칙/일관성 확보, **Prettier**로 포맷.  
- CSS의 **`@layer`**로 우선순위 층 나누기(초기화/base → 컴포넌트 → 유틸 → 페이지 오버라이드).  
- **소스맵**은 개발만, **압축/해시**로 배포 최적화.

---

## 10) Sass vs CSS 변수: 역할 분담 정리

| 축 | CSS 변수(`--x`) | Sass 변수(`$x`) |
|---|---|---|
| 처리 시점 | **런타임** | **컴파일** |
| 목적 | 테마/환경/사용자 선호도 | 토큰/유틸 생성/반복 제거 |
| 변경 | JS/미디어쿼리/상태에 따라 동적 | 빌드 시 고정 |
| 상속 | O (Cascading) | X(스코프는 소스단) |
| 베스트 | **다크모드**, 고대비, 리듀스 모션, 컨테이너별 조정 | 스케일·맵 기반 **자동 생성**, 모듈화 |

---

## 11) 언제 무엇을 쓰나? (의사결정 가이드)

- **소규모/순수 마크업**: 최신 CSS만으로도 충분(`:has()`, `@layer`, 컨테이너쿼리, 제한적 네스팅).  
- **디자인 시스템/대규모**: **Sass + CSS 변수 하이브리드** 권장.  
  - Sass: 토큰·유틸·반복 생성, 모듈, 품질 파이프라인  
  - CSS: 런타임 테마/환경 선호도(다크모드, `prefers-reduced-motion`, 컨테이너 기반 조정)
- **SSR/프레임워크**: CSS Modules/Scoped CSS와 함께 쓰되, 전역 토큰/유틸은 Sass로 관리.

---

## 12) 마이그레이션: `@import` → `@use/@forward`

**Before**
```scss
@import "variables";
@import "mixins";
.button { @include flex-center; color: $brand; }
```

**After**
```scss
@use "variables" as v;
@use "mixins" as m;

.button { @include m.flex-center; color: v.$brand; }
```

> 장점: 네임스페이스로 **충돌 방지**, 의존성 명확, 중복 평가 방지.

---

## 13) 트러블슈팅/주의점

- **슬래시 나눗셈 경고**: `10px/2` → `@use "sass:math"; math.div(10px, 2)` 사용.  
- **과도한 중첩**: 특이도 폭발/유지보수 악화 → 3~4단계 이내.  
- **`@extend` 남용 금지**: 선택자 병합 부작용 → 믹스인 우선.  
- **경로 문제**: includePaths/별칭 정리.  
- **CSS 변수 폴백**: 런타임 변수 사용 시 합리적 기본값 준비.  
- **브라우저 지원**: 최신 CSS 기능(네스팅/컨테이너쿼리)은 대상 브라우저 범위 확인 후 단계적 적용.

---

## 14) 종합 예제: 테마 + 반응형 + 유틸 자동생성(요약)

```scss
/* abstracts/_tokens.scss */
$colors: ("brand": #2563eb, "accent": #8b5cf6, "danger": #ef4444, "muted": #6b7280);
$spaces: (0: 0, 1: .25rem, 2: .5rem, 3: .75rem, 4: 1rem);
$bp: ("md": 768px, "lg": 1024px);

/* abstracts/_mixins.scss */
@use "sass:math";
@mixin up($k) { @media (min-width: map-get($bp, $k)) { @content; } }
@function fluid($min, $max, $vw-min: 360, $vw-max: 1280) {
  $slope: ($max - $min) / ($vw-max - $vw-min) * 100;
  $intercept: $min - ($slope * $vw-min / 100);
  @return clamp(#{$min}, #{$intercept}rem + #{$slope}vw, #{$max});
}

/* utilities/_spacing.scss */
@use "../abstracts/tokens" as t;
@each $k, $v in t.$spaces {
  .m-#{$k} { margin: $v; }
  .p-#{$k} { padding: $v; }
}

/* components/_button.scss */
@use "../abstracts/tokens" as t;
@use "../abstracts/mixins" as m;
.button {
  display:inline-flex; align-items:center; justify-content:center;
  padding:.6rem 1rem; font-size: m.fluid(1rem, 1.125rem);
  border-radius:.75rem; border:1px solid transparent;
  background: map-get(t.$colors, "brand"); color:#fff; font-weight:600;
  transition: filter .2s ease, transform .15s ease;
  &:hover { filter: brightness(1.05); }
  &:active { transform: translateY(1px); }
}

/* themes/_expose-vars.scss (하이브리드 포인트) */
:root {
  --radius: .75rem;
  --surface: #fff;
  --text: #111;
}
:root[data-theme="dark"] {
  --surface: #14161a;
  --text: #eaeaea;
}
.card { border-radius: var(--radius); background: var(--surface); color: var(--text); }

/* main.scss */
@use "abstracts/tokens";
@use "abstracts/mixins";
@use "utilities/spacing";
@use "components/button";
@use "themes/expose-vars";
```

---

## 15) 참고 링크

- Sass 공식 문서: https://sass-lang.com/documentation  
- Sass 설치: https://sass-lang.com/install  
- Autoprefixer: https://github.com/postcss/autoprefixer  
- Stylelint SCSS: https://github.com/stylelint-scss/stylelint-scss  
- CSS Custom Properties 가이드: https://css-tricks.com/a-complete-guide-to-custom-properties/

---

## 마무리

- **Sass**는 “대규모 스타일의 설계·유지보수·자동화”에 강하고, **CSS**는 “런타임 적응성과 표준성”이 강점입니다.  
- **하이브리드**로 설계하면(토큰/유틸은 Sass, 테마/환경은 CSS 변수) **개발 효율 + UX 적응성**을 동시에 얻을 수 있습니다.  
- 새로운 프로젝트라면 `@use/@forward` 기반 모듈화, 맵·믹스인·함수로 **반복 제거**, CSS 변수로 **테마/접근성**을 완성하세요.