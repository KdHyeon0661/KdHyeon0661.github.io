---
layout: post
title: CSS - 아이콘과 Font Awesome
date: 2025-04-06 19:20:23 +0900
category: CSS
---
# CSS 아이콘과 Font Awesome 활용법

- 순수 CSS 아이콘을 위한 **의미론/레이어링/애니메이션** 패턴
- **currentColor**, `mask-image`, 인라인 **SVG**와의 비교 및 선택 가이드
- Font Awesome v6 기준 **스타일 프리픽스/사이징/정렬/애니메이션/레이어링**
- **접근성(A11y)**과 **성능 최적화**(서브셋, 지연 로딩, FOUT 방지) 체크리스트
- 프레임워크(React) 사용 예, 다크 모드 대응, 저동작 설정(`prefers-reduced-motion`) 등

---

## 1. 순수 CSS로 만드는 아이콘

### 1.1 햄버거 메뉴 — `::before`/`::after` 3줄 레이어링
```html
<button class="hamburger" aria-label="메뉴 열기"></button>
```

```css
.hamburger {
  --w: 28px; --h: 2px; --gap: 7px; --color: #333;
  position: relative;
  width: var(--w);
  height: calc(var(--h) * 3 + var(--gap) * 2);
  border: 0; background: none; padding: 0; cursor: pointer;
}
.hamburger::before,
.hamburger::after,
.hamburger:focus-visible {
  outline: 2px solid #1d4ed8; outline-offset: 4px; /* 접근성: 키보드 포커스 */
}
.hamburger::before,
.hamburger::after {
  content: "";
  position: absolute; left: 0; right: 0; height: var(--h);
  background: var(--color);
}
.hamburger::before { top: 0; }
.hamburger::after  { bottom: 0; }
.hamburger::marker, .hamburger::selection { /* 안전 무시 */ }
.hamburger {
  background:
    linear-gradient(var(--color), var(--color)) 0 50% / 100% var(--h) no-repeat;
}
```
- 가운데 줄은 `background`로, 위/아래 줄은 `::before/::after`로 표현(3레이어).
- 키보드 사용자에게 **가시 포커스** 제공.

#### 아이콘 → “닫기(X)” 로 전환 애니메이션
```css
.hamburger {
  transition: transform .25s ease;
}
.hamburger.is-active { transform: rotate(90deg); }

.hamburger::before,
.hamburger::after {
  transition: transform .25s ease, top .25s ease, bottom .25s ease, opacity .2s ease;
}
.hamburger.is-active::before {
  top: 50%; transform: translateY(-50%) rotate(45deg);
}
.hamburger.is-active::after {
  bottom: 50%; transform: translateY(50%) rotate(-45deg);
}
.hamburger.is-active {
  background-size: 0 var(--h); /* 가운데 줄 숨김 */
}
```

### 1.2 단색 아이콘: `currentColor`로 테마 연동
```html
<span class="icon icon-circle" aria-hidden="true"></span> 저장
```

```css
.icon {
  display: inline-block; width: 1em; height: 1em; /* 글꼴 크기 비례 */
  vertical-align: -0.15em; /* 텍스트 라인에 자연스럽게 */
  background: currentColor; /* 부모 글자색을 그대로 */
  border-radius: 50%;
}
button.primary { color: #2563eb; } /* icon도 함께 파란색으로 */
```
- 아이콘 색상을 **글자색과 동기화** → 테마/상태 변화에 자동 추종.

### 1.3 외곽선(스트로크) 아이콘: `mask-image`(SVG/PNG) + `background-color`
```html
<span class="mask-icon" aria-hidden="true"></span> 북마크
```

```css
.mask-icon {
  --size: 1.125em;
  width: var(--size); height: var(--size);
  display: inline-block; vertical-align: -0.2em;
  background-color: currentColor; /* 색상 제어는 배경색으로 */
  -webkit-mask: url('bookmark.svg') no-repeat center / contain;
          mask: url('bookmark.svg') no-repeat center / contain;
}
```
- 마스크로 형태만 따오고 **색은 CSS로** → 다크 모드/상태에 유연.
- 단색/외곽선 계열에 적합. (복잡한 다색은 SVG가 더 낫다)

---

## 2. 아이콘 전략 — CSS vs SVG vs 폰트(FA) 비교

| 방식 | 장점 | 단점 | 추천 용도 |
|---|---|---|---|
| **순수 CSS** | 의존성 0, 가볍고 테마 연동 쉬움 | 복잡한 형태/다색 어려움 | 단순 UI 표시(햄버거, 점 3개, 배지) |
| **SVG(Inline)** | 정확한 벡터, 다색/그라디언트/ARIA 우수 | DOM 부하·인라인 길어짐 | 브랜딩/다색/복잡한 상징 |
| **폰트 아이콘(FA)** | 사용·일관성 쉬움, 클래스 한 줄 | 폰트 로딩 의존, 세밀한 정렬 이슈 | 광범위한 일반 아이콘 세트 |

> 다크 모드, 상태 색상, 접근성 커스텀을 많이 한다면 **SVG/`currentColor`** 조합이 유리.
> 빠르게 **방대한 카탈로그**가 필요하면 **Font Awesome**이 생산적.

---

## 3. Font Awesome 빠른 시작

### 3.1 CDN (간단/프로토타입)
```html
<link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.5.0/css/all.min.css">
```

### 3.2 NPM (프로덕션/번들링)
```bash
npm i @fortawesome/fontawesome-free
```
```css
/* main.css or entry */
@import "~@fortawesome/fontawesome-free/css/all.min.css";
```

> **프리픽스**(v6):
> - 실선: `fa-solid`
> - 레귤러: `fa-regular`
> - 브랜드: `fa-brands`
> (Pro에는 `fa-light`, `fa-thin`, `fa-duotone` 등 추가)

---

## 4. Font Awesome 사용법

### 4.1 기본 HTML
```html
<i class="fa-solid fa-user"></i>
<i class="fa-regular fa-heart"></i>
<i class="fa-brands fa-github"></i>
```

### 4.2 크기/정렬/색상
```html
<i class="fa-solid fa-camera fa-lg"></i>
<i class="fa-solid fa-camera fa-2x"></i>
<i class="fa-solid fa-camera" style="color:#2563eb"></i>
```
```css
/* 텍스트 라인에 자연스러운 정렬 */
i[class^="fa-"],
i[class*=" fa-"] { vertical-align: -0.125em; }
```

- 사이즈: `fa-sm`, `fa-lg`, `fa-2x`~`fa-10x`
- 색상: CSS `color`로 제어(폰트이므로)

### 4.3 회전/플립/애니메이션
```html
<i class="fa-solid fa-rotate-right fa-spin" aria-hidden="true"></i>
<i class="fa-solid fa-arrow-right fa-flip-horizontal"></i>
```

#### 사용자 환경 배려(저동작)
```css
@media (prefers-reduced-motion: reduce) {
  .fa-spin, .fa-pulse { animation-duration: 2s !important; }
}
```

### 4.4 버튼/링크와 결합
```html
<button class="btn">
  <i class="fa-solid fa-paper-plane" aria-hidden="true"></i>
  <span class="sr-only">전송</span>
  전송
</button>

<a href="https://github.com" rel="noopener">
  <i class="fa-brands fa-github" aria-hidden="true"></i>
  GitHub
</a>
```

```css
.btn i { margin-right: .5em; }
.sr-only {
  position:absolute; width:1px; height:1px; padding:0; margin:-1px;
  overflow:hidden; clip:rect(0,0,0,0); white-space:nowrap; border:0;
}
```

---

## 5. Font Awesome 접근성(A11y)

- **장식용** 아이콘 → `aria-hidden="true"`
- **의미 전달** 아이콘 → 텍스트 제공(`sr-only`) 또는 `aria-label`

```html
<!-- 장식 -->
<i class="fa-solid fa-check" aria-hidden="true"></i>

<!-- 의미 제공 -->
<i class="fa-solid fa-trash" aria-label="삭제"></i>
<!-- or -->
<button aria-label="삭제">
  <i class="fa-solid fa-trash" aria-hidden="true"></i>
</button>
```

- 아이콘만으로 색/형태만 전달하지 않도록 **텍스트 보강**(색각 이상 고려).
- 반복 회전 등 애니메이션은 `prefers-reduced-motion` 존중.

---

## 6. Font Awesome 정렬/줄바꿈/간격 디테일

### 6.1 텍스트 정렬
```css
/* 폰트 메트릭에 따라 약간 들뜨는 현상 보정 */
.fa-fw { width: 1.25em; text-align: center; } /* 고정폭 아이콘 */
```
```html
<li><i class="fa-solid fa-user fa-fw"></i> 프로필</li>
<li><i class="fa-solid fa-gear fa-fw"></i> 설정</li>
```

### 6.2 인라인 높이/줄 간격
```css
.icon-inline { line-height: 1; vertical-align: -0.15em; }
```

---

## 7. Font Awesome 레이어링/뱃지(스택)

**둘러싼 레이어**를 만들어 뱃지/원형 배경을 겹치기

```html
<span class="fa-stack" aria-label="알림 있음">
  <i class="fa-regular fa-bell fa-stack-1x"></i>
  <i class="fa-solid fa-circle fa-stack-2x" style="color:#ef4444"></i>
  <span class="fa-stack-1x" style="color:#fff; font-size:.6em; font-weight:700">3</span>
</span>
```

```css
.fa-stack { display:inline-grid; place-items:center; position:relative; width:2em; height:2em; }
.fa-stack-1x, .fa-stack-2x { grid-area:1/1; }
.fa-stack-2x { transform: scale(1.2); } /* 배경 원을 약간 키움 */
```

> Pro의 **Duotone**(두 색상) 스타일이 있으면 더 풍부한 레이어 표현 가능.

---

## 8. 성능 최적화 (필수 체크리스트)

1) **최소 자산만 로드**: `all.min.css`는 편하지만 큼.
   - 가능하면 **서브셋**(필요 아이콘만)으로 번들(빌드 단계에서 PostCSS/수동 선택).
2) **HTTP/2/3 + 캐싱**: CDN 사용 시 캐시 길게, 해시 버전 사용.
3) **FOIT/FOUT 방지**:
   - `<link rel="preload" as="font" crossorigin>`로 폰트 미리 불러오기.
   - 초기 로딩에서 아이콘 자리 비는 현상을 줄임.
4) **지연 로딩**: 접히는 섹션/모달에서 **필요할 때만** 로드.
5) **대체 텍스트**: 아이콘 폰트가 로딩 실패해도 의미 손실 최소화(텍스트/`sr-only`).

예: 폰트 프리로드
```html
<link rel="preload" href="/fonts/fa-solid-900.woff2" as="font" type="font/woff2" crossorigin>
```

---

## 9. 다크 모드/상태 색상 전략

- **폰트 아이콘**: 단일 `color`면 끝. 테마 스위치 시 부모 색만 바꾸면 됨.
- **CSS 마스크**: `background-color: currentColor;` + `prefers-color-scheme`로 전환.

```css
:root { color-scheme: light dark; }
html { color: #111; background:#fff; }
@media (prefers-color-scheme: dark) {
  html { color:#e5e7eb; background:#0b1220; }
}
```

---

## 10. 실전: “설정” 토글(태양/달) 아이콘 — Font Awesome + 상태 싱크

```html
<button class="theme-toggle" aria-pressed="false" aria-label="다크 모드 전환">
  <i class="fa-solid fa-sun" aria-hidden="true"></i>
</button>
```

```css
.theme-toggle { font-size: 1.125rem; color: #f59e0b; }
.theme-toggle[aria-pressed="true"] { color:#60a5fa; }
.theme-toggle[aria-pressed="true"] i { content: "\f186"; } /* 해 → 달 (단, 폰트 코드 직접 사용은 권장X) */
```

> 실제로는 **클래스 교체**가 안전:
```js
const btn = document.querySelector('.theme-toggle');
btn.addEventListener('click', () => {
  const pressed = btn.getAttribute('aria-pressed') === 'true';
  btn.setAttribute('aria-pressed', String(!pressed));
  btn.innerHTML = pressed
    ? '<i class="fa-solid fa-sun" aria-hidden="true"></i>'
    : '<i class="fa-solid fa-moon" aria-hidden="true"></i>';
});
```

---

## 11. React에서 Font Awesome 쓰기(간단 버전)

> 공식 React 래퍼도 있지만, CDN/CSS 클래스로도 충분한 경우가 많다.

```jsx
export function Icon({ name, style="solid", className="", label }) {
  const styleClass = style === "brands" ? "fa-brands" :
                     style === "regular"? "fa-regular" : "fa-solid";
  const aria = label ? { role: "img", "aria-label": label } : { "aria-hidden": true };
  return <i className={`${styleClass} fa-${name} ${className}`} {...aria} />;
}

// 사용
<Icon name="user" />
<Icon name="github" style="brands" />
<Icon name="trash" label="삭제" className="text-red-600" />
```

---

## 12. 대체 라이브러리 빠른 비교

| 라이브러리 | 장점 | 비고 |
|---|---|---|
| **Material Icons** | 구글 머티리얼 가이드와 일관 | 다양한 가중치/스타일 |
| **Bootstrap Icons** | 가볍고 범용, 부트스트랩 톤 | SVG 제공 중심 |
| **Heroicons** | Tailwind 톤, 깔끔한 선형 | SVG로 쓰기 좋음 |
| **Remix Icon** | 미니멀, 풍부한 카테고리 | 무료/가벼움 |

- **SVG 기반** 세트는 색/두께/접근성 제어가 세밀.
- **폰트 기반**은 심플하고 빠른 생산성이 강점.

---

## 13. 실전 컴포넌트 예제 — 알림 벨 + 뱃지 + 접근성

```html
<button class="notif" aria-label="알림 5개">
  <span class="fa-stack">
    <i class="fa-regular fa-bell fa-stack-1x" aria-hidden="true"></i>
    <span class="badge" aria-hidden="true">5</span>
  </span>
</button>
```

```css
.fa-stack { position: relative; display:inline-grid; place-items:center; width:1.75em; height:1.75em; }
.badge {
  position:absolute; top:-10%; right:-10%;
  min-width:1.2em; height:1.2em;
  background:#ef4444; color:#fff; font-size:.7em; font-weight:700;
  border-radius:999px; display:grid; place-items:center; padding:0 .25em;
}
.notif { line-height:1; color:#334155; }
.notif:hover { color:#111827; }
```

---

## 14. 점 3개(케밥/엘립시스) 메뉴 — CSS 아이콘

```html
<button class="dots" aria-label="메뉴 열기"></button>
```

```css
.dots {
  --size: .25em; --gap: .35em; color:#334155;
  display:inline-grid; grid-template-columns: repeat(3, var(--size)); gap: var(--gap);
  width: calc(var(--size)*3 + var(--gap)*2);
  height: var(--size); align-items:center; border:0; background:none; cursor:pointer;
}
.dots::before, .dots::after, .dots span { content:""; display:block; width:var(--size); height:var(--size); background: currentColor; border-radius:50%; }
.dots span { /* 중앙 점 */ }
```

```html
<!-- 중앙 점은 span 하나를 DOM으로 추가해도 되고, ::before/::after + background으로도 가능 -->
<button class="dots" aria-label="메뉴 열기"><span aria-hidden="true"></span></button>
```

---

## 15. 이미지 위 오버레이 아이콘(FA + 그라디언트)

```html
<figure class="card">
  <img src="photo.jpg" alt="풍경 사진">
  <figcaption>
    <i class="fa-solid fa-magnifying-glass" aria-hidden="true"></i>
    확대
  </figcaption>
</figure>
```

```css
.card { position:relative; overflow:hidden; border-radius:12px; }
.card figcaption {
  position:absolute; inset:auto 0 0 0;
  background: linear-gradient(to top, rgba(0,0,0,.55), transparent);
  color:#fff; padding:12px; display:flex; align-items:center; gap:8px;
}
.card img { display:block; width:100%; height:auto; }
```

---

## 16. 흔한 문제와 해결

1) **아이콘이 줄 기준에서 들뜬다**
   → `vertical-align` 미세 조정(예: `-0.125em`), `line-height:1` 확인.

2) **폰트 로딩 지연 시 아이콘 사라짐(FOUT/FOIT)**
   → 프리로드/캐싱, 중요한 곳은 **SVG 인라인**으로 대체.

3) **색상/상태별 변형이 많다**
   → `currentColor` 기반 설계(부모에 색만 바꾸기), 또는 **CSS 변수**로 테마.

4) **애니메이션 민감 사용자 불편**
   → `prefers-reduced-motion` 미디어 쿼리로 속도/회전 제한.

5) **의미 전달 실패**
   → `aria-label`/`sr-only` 텍스트 제공, 시각만 의존 금지.

---

## 17. 체크리스트 요약

- [ ] **용도 선택**: CSS(단순) / SVG(다색·정밀) / FA(대카탈로그)
- [ ] **접근성**: 장식 `aria-hidden="true"`, 의미 `aria-label`/텍스트
- [ ] **currentColor 활용**: 테마/상태 동기화
- [ ] **성능**: 서브셋/프리로드/지연 로딩/HTTP 캐시
- [ ] **정렬**: `vertical-align`, `fa-fw`, `line-height:1`
- [ ] **모션 배려**: `prefers-reduced-motion`
- [ ] **다크 모드**: 색 체계 변수화, 마스크/폰트 베이스면 손쉬움

---

## 18. 전체 샘플 — CSS 아이콘 + Font Awesome 혼합 UI

```html
<!doctype html>
<meta charset="utf-8">
<title>Icons Demo</title>
<link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.5.0/css/all.min.css">

<style>
  :root { color-scheme: light dark; }
  body { font-family: system-ui, -apple-system, "Noto Sans KR", sans-serif; margin: 2rem; line-height: 1.6; }
  .toolbar { display:flex; gap:12px; align-items:center; }

  /* CSS 햄버거 */
  .hamburger { --w: 24px; --h: 2px; --gap: 6px; --color:#111; position:relative; width:var(--w); height:calc(var(--h)*3 + var(--gap)*2); border:0; background:none; cursor:pointer; }
  .hamburger::before, .hamburger::after { content:""; position:absolute; left:0; right:0; height:var(--h); background:var(--color); transition:.25s ease; }
  .hamburger::before { top:0; }
  .hamburger::after  { bottom:0; }
  .hamburger { background: linear-gradient(var(--color), var(--color)) 0 50% / 100% var(--h) no-repeat; transition:.25s ease; }
  .hamburger.is-active { transform: rotate(90deg); background-size:0 var(--h); }
  .hamburger.is-active::before { top:50%; transform: translateY(-50%) rotate(45deg); }
  .hamburger.is-active::after  { bottom:50%; transform: translateY(50%) rotate(-45deg); }

  /* CSS 마스크 아이콘 (북마크) */
  .mask-icon {
    --s: 1.1em;
    width: var(--s); height: var(--s);
    display:inline-block; vertical-align:-0.2em;
    background-color: currentColor;
    -webkit-mask: url('bookmark.svg') no-repeat center / contain;
            mask: url('bookmark.svg') no-repeat center / contain;
  }

  /* Font Awesome 디테일 */
  .fa-fw { width:1.25em; text-align:center; }
  .btn { display:inline-flex; align-items:center; gap:.5em; padding:.6em .9em; border:1px solid #e5e7eb; border-radius:.6em; background:#fff; color:#111; }
  .btn:hover { background:#f9fafb; }

  /* 카드 */
  .cards { display:grid; gap:16px; grid-template-columns: repeat(auto-fit, minmax(220px, 1fr)); margin-top:1.5rem; }
  .card { border:1px solid #e5e7eb; border-radius:12px; padding:1rem; background:#fff; display:grid; gap:.5rem; }
  .card h3 { margin:0; font-size:1rem; display:flex; align-items:center; gap:.5em; }
  .card p { margin:0; color:#475569; }

  /* 모션 배려 */
  @media (prefers-reduced-motion: reduce) {
    .fa-spin, .fa-pulse { animation-duration: 2s !important; }
  }
</style>

<header class="toolbar">
  <button class="hamburger" aria-label="메뉴" id="btn-ham"></button>
  <button class="btn">
    <i class="fa-solid fa-paper-plane fa-fw" aria-hidden="true"></i>
    전송
  </button>
  <button class="btn">
    <span class="mask-icon" aria-hidden="true"></span>
    저장
  </button>
  <button class="btn" aria-label="새로고침">
    <i class="fa-solid fa-rotate-right fa-spin fa-fw" aria-hidden="true"></i>
  </button>
</header>

<section class="cards">
  <article class="card">
    <h3><i class="fa-solid fa-user fa-fw" aria-hidden="true"></i> 프로필</h3>
    <p>사용자 정보 관리</p>
  </article>
  <article class="card">
    <h3><i class="fa-regular fa-heart fa-fw" aria-hidden="true"></i> 찜</h3>
    <p>관심 목록 보기</p>
  </article>
  <article class="card">
    <h3><i class="fa-brands fa-github fa-fw" aria-hidden="true"></i> GitHub</h3>
    <p>저장소 연결</p>
  </article>
</section>

<script>
  document.getElementById('btn-ham').addEventListener('click', e => {
    e.currentTarget.classList.toggle('is-active');
  });
</script>
```

---

## 결론

- **빠른 생산성**이 필요하면 Font Awesome: **클래스 한 줄**로 방대한 아이콘.
- **테마/상태/색상** 유연성과 접근성·정밀함이 중요하면 **SVG/`currentColor`** 전략.
- 순수 CSS 아이콘은 **경량 UI 표현**에 최적. 복잡한 다색 로고/브랜드는 **SVG**.
- 어떤 방식을 선택하든 **접근성(레이블/저동작)**과 **성능(서브셋/프리로드)**을 반드시 고려할 것.
