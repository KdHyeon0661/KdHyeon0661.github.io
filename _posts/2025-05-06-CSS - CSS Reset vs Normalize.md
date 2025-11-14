---
layout: post
title: CSS - CSS Reset vs Normalize
date: 2025-05-06 22:20:23 +0900
category: CSS
---
# CSS Reset vs Normalize.css

## 공통 목표와 철학

| 항목        | 설명 |
|-------------|------|
| 목적        | 브라우저별 기본 스타일 차이로 인한 **예상치 못한 UI 불일치** 최소화 |
| 적용 시점   | **프로젝트 최상단 CSS**(모든 컴포넌트보다 먼저) |
| 접근법      | Reset은 **기본값을 제거**해 “백지 상태”에서 시작, Normalize는 **표준/가치 있는 기본값은 보존**하고 “일관화” |

**핵심 질문**: “나는 브라우저 기본값까지 모두 제거하고 처음부터 쌓고 싶은가?
아니면 브라우저 기본값 중 ‘괜찮은’ 것들은 살리면서 균일하게 맞추고 싶은가?”

---

## CSS Reset — “백지 상태로 초기화”

Reset은 브라우저 기본 스타일을 **전방위로 제거**합니다.
예: 기본 margin/padding, 리스트 마커, 링크 밑줄, 폼 컨트롤의 네이티브 스타일 등.

### 2-1. 간단한 Reset 예시 (학습/개념용)

```css
/* Mini Reset (학습용: 꼭 이대로 쓰라는 의미는 아님) */
*,
*::before,
*::after {
  box-sizing: border-box;
}

html, body,
h1, h2, h3, h4, h5, h6,
p, figure, blockquote,
dl, dd {
  margin: 0;
  padding: 0;
}

ul[role='list'],
ol[role='list'],
menu, ul, ol {
  list-style: none;
  margin: 0;
  padding: 0;
}

/* 기본 링크 장식 제거 → 프로젝트 정책으로 재정의 */
a {
  text-decoration: none;
  color: inherit;
}

/* 이미지/미디어의 컨테이너 초과 방지 */
img, picture, video, canvas, svg {
  display: block;
  max-width: 100%;
}

/* 폼 요소도 글꼴 상속 */
input, button, textarea, select {
  font: inherit;
  color: inherit;
}

/* 포커스 표시를 아예 없애지 말 것! (접근성) */
/* outline: 0; 같은 파괴적 리셋은 지양 */
```

### 2-2. Reset의 장단점

- 장점
  - **완전한 제어권**: 모든 UI를 **디자인 시스템** 기준으로 통일
  - 디자인이 강하게 정의된 서비스(브랜드/콘텐츠가 일관)에서 유리
- 단점
  - 브라우저가 제공하는 **유용한 기본값(특히 폼/타이포/접근성)**도 사라짐
  - 포커스 링 제거 등은 **A11y 리스크** → 반드시 대체 포커스를 재정의해야 함
  - 초반에 **손이 많이 감**

> 결론: **“내가 모든 걸 직접 통제하겠다”**면 Reset. 단, 접근성/사용성 기본을 **재구축**해야 합니다.

---

## Normalize.css — “표준/유용 기본값은 보존 + 차이만 보정”

[Normalize.css](https://github.com/necolas/normalize.css)는 브라우저가 제공하는 **합리적인 기본 스타일은 유지**하면서,
브라우저마다 다른 **미묘한 차이**를 **보정**해 **일관된 기본 상태**를 만듭니다.

### 3-1. 사용 방법

```html
<!-- 상단에 포함 -->
<link rel="stylesheet" href="https://necolas.github.io/normalize.css/8.0.1/normalize.css">
```

### 3-2. Normalize의 장단점

- 장점
  - 헤딩/본문/폼 등 **표준적 기본값 존중** → 접근성, 가독성 유지
  - “필요한 차이만” 줄이므로 초기 개발이 **가볍고 빠름**
- 단점
  - **완전한 백지 상태가 아님** → 완전 커스텀이면 수정/오버라이드 작업 증가
  - 팀/디자인 시스템에 따라 “남아있는 기본값”이 귀찮게 느껴질 수 있음

> 결론: **“합리적 기본 + 최소 보정”**이 목적이면 Normalize.css가 적합.

---

## Reset vs Normalize.css — 한눈에 비교

| 항목            | CSS Reset                                  | Normalize.css                            |
|-----------------|---------------------------------------------|-------------------------------------------|
| 철학            | **모두 제거** 후 프로젝트 규칙으로 재구축   | **보존 가능한 기본은 유지** + **차이는 보정** |
| 접근성          | 포커스/폼 기본 제거 → **다시 설계 필요**    | 기본 접근성 유지에 유리                   |
| 커스터마이즈    | 최고 (백지)                                  | 중간 (보존값 위에 오버라이드)            |
| 러닝커브/초기비용| 높음                                        | 낮음                                       |
| 권장 사례       | 강한 브랜딩/디자인 시스템, 앱형 대시보드    | 콘텐츠/문서형, 합리적 기본을 활용하는 서비스 |

---

## 실무에서 자주 쓰는 “혼합 전략” (Reset + Normalize)

Tailwind(Preflight), Bootstrap(Reboot)처럼 **둘을 적절히 혼합**하는 접근이 가장 널리 쓰입니다.
핵심은 **파괴적 리셋을 피하면서** (예: outline 없애지 않기)
**타이포/폼/포커스/레거시 보정** 등을 **선택적으로 정리**하는 것입니다.

### 5-1. 실전용 “모던 베이스 스타일” (복붙 템플릿)

> Normalize 성격 + 최소 Reset + 접근성/현대 CSS 반영

```css
/* 1) 박스 모델 일관화 */
*, *::before, *::after { box-sizing: border-box; }

/* 2) 기본 마진 제거 + 타이포 리듬은 전역에서 재정의 */
:where(h1,h2,h3,h4,h5,h6,p,figure,blockquote,dl,dd) { margin: 0; }

/* 3) 바디 베이스: 시스템 폰트 + 적절한 라인하이트 */
html:focus-within { scroll-behavior: smooth; }
html, body { height: 100%; }
body {
  margin: 0;
  font-family: system-ui, -apple-system, Segoe UI, Roboto, Noto Sans KR, Apple SD Gothic Neo, sans-serif;
  font-size: 16px;
  line-height: 1.6;
  -webkit-text-size-adjust: 100%;
  text-rendering: optimizeLegibility;
}

/* 4) 이미지/미디어는 컨테이너 초과 금지 */
img, picture, video, canvas, svg { display: block; max-width: 100%; }

/* 5) 폼 요소는 폰트 상속, 버튼 커서 일관화 */
input, button, textarea, select { font: inherit; color: inherit; }
button { cursor: pointer; }
button:disabled { cursor: not-allowed; }

/* 6) 링크: 밑줄/색상 정책은 프로젝트에서 재정의 */
a { color: inherit; text-decoration: none; }
a:focus-visible {
  outline: 2px solid #2563eb; /* 접근성 포커스 */
  outline-offset: 3px;
}

/* 7) 리스트: 기본 마커 제거(필요 시 컴포넌트에서 지정) */
:where(ul,ol)[role='list'],
:where(ul,ol) {
  list-style: none;
  margin: 0; padding: 0;
}

/* 8) 텍스트 오버플로우 유틸(프로젝트 전역에서 자주 필요) */
.text-ellipsis {
  overflow: hidden;
  white-space: nowrap;
  text-overflow: ellipsis;
}

/* 9) 모션 최소화 선호 존중 */
@media (prefers-reduced-motion: reduce) {
  * { animation: none !important; transition: none !important; scroll-behavior: auto !important; }
}

/* 10) 다크 모드 기본 토큰 (선택) */
@media (prefers-color-scheme: dark) {
  :root {
    color-scheme: dark;
  }
}

/* 11) 폼 요소 표준화(부드럽게) — 너무 과도한 appearance 리셋은 지양 */
input[type="search"]::-webkit-search-cancel-button {
  /* 필요 시 스타일 조정; 보통은 기본 유지가 접근성과 사용성에 좋음 */
}
input[type=number]::-webkit-outer-spin-button,
input[type=number]::-webkit-inner-spin-button { /* iOS/Chrome 스피너 */
  height: auto;
}
```

- 포커스 스타일을 **살리는 대신 보기 좋게 재정의**(outline 유지)
- `appearance: none`을 전역으로 깔아 폼 네이티브 UI를 모두 지우면
  **플랫폼 일관성/접근성 손실** → 꼭 필요한 컴포넌트 내부에서만 사용

---

## 타이포그래피/기본 간격 설정(리듬)

Reset 후에는 본문/헤딩의 **기본 간격**을 프로젝트 문체에 맞게 부여합니다.

```css
:root {
  --space-1: 0.25rem;
  --space-2: 0.5rem;
  --space-3: 0.75rem;
  --space-4: 1rem;
  --space-6: 1.5rem;
  --space-8: 2rem;

  /* 유동 타이포(선택): clamp로 상/하한을 둔 가변 폰트 크기 */
  --fs-body: clamp(0.95rem, 0.3vw + 0.85rem, 1.0625rem);
  --fs-h2: clamp(1.25rem, 1.2vw + 1rem, 1.75rem);
}

body { font-size: var(--fs-body); }

h1, h2, h3 { line-height: 1.25; font-weight: 700; }
p + p { margin-top: var(--space-4); }

h2 { font-size: var(--fs-h2); margin: var(--space-6) 0 var(--space-2); }
ul, ol { margin: var(--space-4) 0 var(--space-4) var(--space-6); list-style: disc; }
```

---

## 폼 컨트롤 — “과도한 리셋”을 경계

폼 컨트롤은 플랫폼/브라우저 네이티브 상호작용(키보드/보조기기 포함)이 **접근성의 핵심**입니다.

- `appearance: none`을 전역으로 적용하면 **유용한 네이티브 affordance**를 잃음
  → **컴포넌트 내부에서만 타겟팅**해 최소한으로.
- 플레이스홀더/라벨/에러 메시지/헬프 텍스트/포커스 레이블 이동 등은 **디자인 시스템화**.
- 포커스 상태(`:focus-visible`)를 반드시 **명확히**.

```css
/* 예: 프로젝트 공통 인풋 스타일(네이티브 유지 + 최소 커스텀) */
.input {
  width: 100%;
  padding: .6rem .8rem;
  border: 1px solid #cbd5e1;
  border-radius: .5rem;
  background: #fff;
  color: #0f172a;
}
.input:focus-visible {
  outline: 2px solid #2563eb;
  outline-offset: 2px;
  border-color: #2563eb;
}

/* 커스텀 셀렉트 — appearance 최소화, 키보드/스크린리더 동작 유지 */
.select {
  width: 100%;
  padding: .6rem 2rem .6rem .8rem;
  border: 1px solid #cbd5e1;
  border-radius: .5rem;
  background: #fff url('data:image/svg+xml;utf8,<svg xmlns="http://www.w3.org/2000/svg" ...>') no-repeat right .6rem center / 1rem;
}
.select:focus-visible {
  outline: 2px solid #2563eb;
  outline-offset: 2px;
  border-color: #2563eb;
}
```

---

## 링크/버튼/포커스 — “보이되 과하지 않게”

Reset로 **밑줄/색상 제거**만 해놓고 포커스 링도 지우면 **키보드 사용자가 길을 잃습니다.**

```css
/* 링크: 기본 정책 — 본문 링크엔 밑줄 유지가 정보전달에 유리 */
a[href] {
  color: #2563eb;
  text-decoration: underline;
  text-underline-offset: .2em;
}
a[href]:hover { color: #1d4ed8; }

/* 버튼: 포커스 링 명확화, hover/active 피드백 */
.button {
  display: inline-flex; align-items: center; justify-content: center;
  padding: .6rem 1rem;
  border-radius: .6rem;
  background: #111827; color: #fff;
  border: 1px solid transparent;
  transition: background .2s ease, transform .08s ease;
}
.button:hover { background: #0b1220; }
.button:active { transform: translateY(1px); }
.button:focus-visible {
  outline: 3px solid #22d3ee;
  outline-offset: 2px;
}
```

---

## 프레임워크가 하는 일

| 프레임워크   | 초기화 방식(요지) |
|--------------|-------------------|
| Tailwind CSS | **Preflight**: Normalize 기반 + 박스모델/타이포/폼 상속 등 커스텀 |
| Bootstrap    | **Reboot**: Normalize에서 출발, 타이포/폼 등 현대화 |
| Foundation   | Normalize.css 사용(버전에 따라 커스텀) |

⇒ **이미 “혼합 전략”을 구현**해 둔 상태. 프레임워크 사용 시 **추가 리셋을 또 넣지 않도록** 주의.

---

## “Reset만 쓸 때”와 “Normalize만 쓸 때”의 보완 가이드

### 10-1. Reset만 쓰는 경우(백지 → 전부 구축)

- **필수**: 포커스 링, 스킵 링크, 헤딩/본문 리듬, 리스트 스타일, 링크 기본 정책
- 폼 컨트롤: 라벨/에러/헬프/필수 표시/aria-속성 설계, 모바일 가상 키보드 타입(`input[type=email/tel]`)
- 네이티브 컨트롤 제거는 **국지적으로만**

### 10-2. Normalize만 쓰는 경우(기본 유지 → 커스텀 최소)

- 브랜드/디자인 톤에 맞게 **색/간격/라운드**만 가볍게 오버라이드
- 폼 요소는 **주요 상태(hover/focus/disabled/error)** 만 정책화
- 본문 링크 정책(밑줄/색상) 정립 → 콘텐츠 가독성 확보

---

## 프로덕션 체크리스트

- [ ] **포커스 표시**가 모든 인터랙티브 요소에서 명확한가? (`:focus-visible`)
- [ ] **키보드 탭 순서**가 논리적/예측 가능한가? (CSS 시각 재배치 vs DOM 순서)
- [ ] 폼 컨트롤의 **라벨/aria** 연결, 에러 메시지 읽기 가능?
- [ ] 본문 링크는 **밑줄/컬러 대비**로 구분 가능한가? (색만으로 구분 X)
- [ ] 이미지/비디오가 컨테이너를 넘치지 않는가? (`max-width: 100%`)
- [ ] 다크 모드/모션 축소/대비 강화 등 **환경 선호**를 존중하는가?
- [ ] 프레임워크와 **중복 Reset/Normalize**가 충돌하지 않는가?
- [ ] LCP/CLS에 악영향을 주는 리셋은 없는가? (폰트 FOUT/FOIT, 레이아웃 점프)
- [ ] 인쇄/프린트 보기(선택) 고려는? (`@media print` 기본 최소화)

---

## 실전: “커스텀 Reboot” 템플릿 (Reset+Normalize 하이브리드)

> 프레임워크 미사용 프로젝트에서 권장할 만한 베이스.
> 접근성 친화 + 현대 CSS + 오버라이드 여지 충분.

```css
/* ========== Base ========== */
*, *::before, *::after { box-sizing: border-box; }

html, body { height: 100%; }
html:focus-within { scroll-behavior: smooth; }
body {
  margin: 0;
  -webkit-text-size-adjust: 100%;
  font-family: system-ui, -apple-system, Segoe UI, Roboto, Noto Sans KR, Apple SD Gothic Neo, sans-serif;
  font-size: 16px; line-height: 1.65; color: #0f172a; background: #ffffff;
  text-rendering: optimizeLegibility;
}

/* ========== Typography ========== */
:where(h1,h2,h3,h4,h5,h6,p,figure,blockquote,dl,dd) { margin: 0; }
h1,h2,h3 { line-height: 1.25; font-weight: 700; }
p + p { margin-top: 1rem; }

a[href] {
  color: #2563eb;
  text-decoration: underline;
  text-underline-offset: .2em;
}
a[href]:hover { color: #1d4ed8; }
a:focus-visible { outline: 3px solid #22d3ee; outline-offset: 3px; }

/* List: 문서형 콘텐츠에서는 마커 유지가 더 자연스러움 */
ul, ol { margin: .75rem 0 .75rem 1.25rem; }

/* Media */
img, picture, video, canvas, svg { display: block; max-width: 100%; }

/* ========== Forms ========== */
input, button, textarea, select { font: inherit; color: inherit; }
button { cursor: pointer; }
button:disabled { cursor: not-allowed; }

label { display: inline-block; margin-bottom: .25rem; font-weight: 600; }
.input, .select, .textarea {
  width: 100%;
  padding: .6rem .8rem;
  border: 1px solid #cbd5e1; border-radius: .6rem; background: #fff;
}
.input:focus-visible, .select:focus-visible, .textarea:focus-visible {
  outline: 2px solid #2563eb; outline-offset: 2px; border-color: #2563eb;
}
.textarea { min-height: 6rem; resize: vertical; }

/* ========== Utilities ========== */
.container {
  width: min(100% - 2rem, 1200px);
  margin-inline: auto;
}
.visually-hidden {
  position: absolute !important; height: 1px; width: 1px; overflow: hidden;
  clip: rect(1px, 1px, 1px, 1px); white-space: nowrap; clip-path: inset(50%); border: 0; padding: 0; margin: -1px;
}

/* Motion/Color Preferences */
@media (prefers-reduced-motion: reduce) {
  * { animation: none !important; transition: none !important; scroll-behavior: auto !important; }
}
@media (prefers-color-scheme: dark) {
  :root { color-scheme: dark; }
  body { background: #0b1020; color: #e6e9ef; }
  a[href] { color: #7dd3fc; }
  .input, .select, .textarea { background: #0f172a; border-color: #334155; color: #e6e9ef; }
}

/* Print (선택) */
@media print {
  a[href]::after { content: " (" attr(href) ")"; font-size: .875em; }
}
```

- **문서형 사이트**: 리스트 마커를 살려 의미와 리듬을 유지
- **앱형/Dashboard**: `ul, ol` 전역 마커 제거 후 컴포넌트에서 명시적으로 스타일링

---

## 현업에서 자주 겪는 함정과 해법

1. **outline 전역 제거**
   - 해법: 절대로 지우지 말고 **보기에 좋은 포커스 스타일**로 **대체**.
2. **폼 appearance 전역 제거**
   - 해법: **특정 컴포넌트 범위에서만** 최소한으로. 키보드/스크린리더 흐름 깨지지 않게.
3. **Reset → 모든 margin 0** 후 타이포 리듬 없음
   - 해법: 프로젝트 문체에 맞는 **전역 리듬 스케일**(헤딩/문단 간격)을 재정의.
4. **프레임워크 + 추가 Reset 중복**
   - 해법: 프레임워크 **Preflight/Reboot**를 존중. 중복 초기화는 제거.
5. **링크 정책 불명확**
   - 해법: 본문 링크는 밑줄 유지, 내비/버튼형 링크는 **버튼 룩**으로 구분.

---

## 선택 가이드 — 어떤 프로젝트에 무엇을?

| 상황/요구 | 권장 |
|---|---|
| 강한 브랜드/컴포넌트 완전 커스텀(앱형) | **Reset 기반 + 접근성 보강 + 폼/포커스 재정의** |
| 문서/블로그/콘텐츠 중심 | **Normalize 기반 + 최소 오버라이드** |
| 혼합(대부분의 실무) | **Reboot/Preflight 스타일의 하이브리드**(본 문서 템플릿 참고) |

---

## 예제: Reset vs Normalize의 체감 비교

### 15-1. Reset 기반(포커스/타이포/폼 재구축 필요)

```css
/* 핵심만: 모든 기본 제거 후 프로젝트 규칙으로 재정의 */
*,
*::before,
*::after { box-sizing: border-box; }

:where(h1,h2,h3,h4,h5,h6,p,ul,ol,li,figure,blockquote,dl,dd) { margin: 0; padding: 0; }
ul, ol { list-style: none; }
a { text-decoration: none; color: inherit; }
button, input, select, textarea { font: inherit; color: inherit; }
/* 이후: 포커스/리듬/폼/링크 정책을 “반드시” 별도로 설계 */
```

### 15-2. Normalize 기반(유지 + 보정)

```html
<link rel="stylesheet" href="https://necolas.github.io/normalize.css/8.0.1/normalize.css">
```

```css
/* 위에 소량의 오버라이드만 추가 */
a[href] { color: #2563eb; text-decoration: underline; }
a:hover { color: #1d4ed8; }
```

**체감 포인트**: Reset은 “아무것도 안 정하면 아무것도 안 보인다”. Normalize는 “기본이 깔려 있어 숨만 쉬어도 쓸 만하다”.

---

## 결론

- **Reset**은 **백지**에서 **모든 걸 직접 구축**하고 싶은 팀에 적합(접근성/폼/포커스 재설계 필수).
- **Normalize**는 **합리적 기본값을 보존**하며 **브라우저 차이를 줄이는** 실용적 대안.
- 실무에선 두 접근을 **혼합**한 “Reboot/Preflight” 스타일이 가장 흔하고 **안전**합니다.
  이 문서의 **커스텀 Reboot 템플릿**을 복붙해 **팀 규칙**으로 확장하세요.

---

## 참고 자료

- Normalize.css: https://necolas.github.io/normalize.css/
- Eric Meyer’s Reset: https://meyerweb.com/eric/tools/css/reset/
- MDN — 기본 스타일/크로스 브라우저: https://developer.mozilla.org/ko/
- Tailwind Preflight 소스(참고): https://github.com/tailwindlabs/tailwindcss
- Bootstrap Reboot 소스(참고): https://github.com/twbs/bootstrap
