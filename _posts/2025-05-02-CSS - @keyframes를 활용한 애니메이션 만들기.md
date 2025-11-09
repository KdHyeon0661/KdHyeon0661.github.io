---
layout: post
title: CSS - @keyframes를 활용한 애니메이션 만들기
date: 2025-05-02 19:20:23 +0900
category: CSS
---
# `@keyframes`로 만드는 CSS 애니메이션

## 0) 애니메이션의 철학: 언제, 왜 쓰나

- **의미 있는 변화**만 애니메이션: 상태 전환, 피드백, 주의 환기, 정보 흐름 유도
- **레이아웃 영향 최소화**: 가능하면 `transform`, `opacity` 중심(합성 단계)
- **접근성(현기증/주의력)**: 과도한 모션 지양, `prefers-reduced-motion` 존중
- **성능**: `transition: all` 지양, 변경 속성만 명시, `will-change` 남발 금지

---

## 1) 기본 개념 & 문법

```css
@keyframes fade-up {
  0%   { opacity: 0; transform: translateY(12px); }
  100% { opacity: 1; transform: translateY(0); }
}

.card {
  animation-name: fade-up;
  animation-duration: .4s;
  animation-timing-function: cubic-bezier(.2,.6,.2,1);
  animation-fill-mode: both; /* 시작·끝 상태 유지 */
}
```

### 1.1 애니메이션 속성 요약

| 속성 | 설명 | 팁 |
|---|---|---|
| `animation-name` | 사용할 `@keyframes` 이름 | 여러 개 콤마로 결합 가능 |
| `animation-duration` | 재생 시간 | `200ms ~ 500ms` 범위가 흔함 |
| `animation-timing-function` | 속도 곡선 | `ease`, `linear`, `cubic-bezier()`, `steps()` |
| `animation-delay` | 시작 지연 | 스태깅, 순차 효과 |
| `animation-iteration-count` | 반복 횟수 | `1`, `3`, `infinite` |
| `animation-direction` | 반복 방향 | `normal`, `reverse`, `alternate`, `alternate-reverse` |
| `animation-fill-mode` | 전·후 상태 | `none`, `forwards`, `backwards`, `both` |
| `animation-play-state` | 재생/일시정지 | `running`, `paused` |
| `animation` | 축약형 | `name duration timing delay iter dir fill` |

**축약형 예시**

```css
.box { animation: fade-up .4s cubic-bezier(.2,.6,.2,1) 0s 1 normal both; }
```

---

## 2) 키프레임 정의 패턴

### 2.1 `%` vs `from`/`to`

```css
@keyframes pulse {
  from { transform: scale(1); }
  to   { transform: scale(1.06); }
}
```

```css
@keyframes rainbow {
  0%   { background: #ef4444; }
  25%  { background: #f59e0b; }
  50%  { background: #10b981; }
  75%  { background: #3b82f6; }
  100% { background: #8b5cf6; }
}
```

### 2.2 여러 속성을 동시에

```css
@keyframes flip-in {
  0%   { opacity: 0; transform: rotateX(-90deg) translateZ(0); transform-origin: top; }
  100% { opacity: 1; transform: rotateX(0deg) translateZ(0); }
}
```

> **주의**: 키프레임 내에서 같은 속성을 여러 번 정의하면 **나중 선언**이 우선합니다.

---

## 3) 타이밍 함수 깊게 보기

### 3.1 `cubic-bezier` 사용자 곡선

```css
/* 빠르게 시작-살짝 튕김 느낌 */
.button { animation-timing-function: cubic-bezier(.17,.67,.3,1.3); }
```

### 3.2 `steps(n, jump-*)` — 단계적 애니메이션

```css
@keyframes tick {
  from { background-position:   0 0; }
  to   { background-position: -100% 0; }
}
.counter {
  animation: tick 1s steps(10, end) infinite;
}
```

- 스프라이트, 카운터, 글자 하나씩 등장 같은 **계단형 효과**에 최적

---

## 4) 반복/방향/채움/재생 상태

```css
.loader {
  animation: spin 1s linear infinite;        /* 반복 */
}
@keyframes spin { to { transform: rotate(360deg); } }

.bounce {
  animation: jump .5s ease-in-out infinite alternate;  /* 왕복 */
}
@keyframes jump { from { transform: translateY(0) } to { transform: translateY(-16px) } }

.toast {
  animation: fade-in .24s ease-out 0s 1 normal both;   /* 끝 상태 유지 */
}
```

일시 정지:

```css
.anim:hover { animation-play-state: paused; }
```

JS로 제어:

```js
const el = document.querySelector('.anim');
el.style.animationPlayState = 'paused';
el.addEventListener('animationend', () => console.log('done'));
```

---

## 5) 여러 애니메이션 결합(병렬·직렬)

### 5.1 병렬(동시에)

```css
.badge {
  animation:
    fade-in .25s ease-out both,
    slight-up .25s ease-out both;
}
```

### 5.2 직렬(순차) — `animation-delay` 활용

```css
.item:nth-child(1) { animation-delay: 0ms; }
.item:nth-child(2) { animation-delay: 60ms; }
.item:nth-child(3) { animation-delay: 120ms; }
```

**스태거 유틸**

```css
.stagger > * { animation: fade-up .4s ease both; }
.stagger > *:nth-child(n) { animation-delay: calc((var(--i, 0)) * 70ms); }
```

```html
<ul class="stagger">
  <li style="--i:0">A</li>
  <li style="--i:1">B</li>
  <li style="--i:2">C</li>
</ul>
```

---

## 6) 실전 패턴 라이브러리

### 6.1 로딩 스피너(경계선 회전)

```html
<div class="spinner" aria-label="로딩 중" role="status"></div>
```

```css
.spinner {
  width: 40px; height: 40px; border-radius: 50%;
  border: 4px solid rgba(0,0,0,.1);
  border-top-color: #3b82f6;
  animation: spin 1s linear infinite;
}
@keyframes spin { to { transform: rotate(360deg); } }
```

### 6.2 Skeleton Shimmer(콘텐츠 자리 표시자)

```html
<div class="skeleton" style="width:100%;height:16px"></div>
```

```css
.skeleton{
  position:relative; overflow:hidden; background:#e5e7eb; border-radius:8px;
}
.skeleton::after{
  content:""; position:absolute; inset:0 -150%;
  background: linear-gradient(90deg, transparent, rgba(255,255,255,.6), transparent);
  animation: shimmer 1.2s ease-in-out infinite;
}
@keyframes shimmer { from { transform: translateX(-40%); } to { transform: translateX(40%); } }
```

### 6.3 Toast 등장/퇴장

```html
<div class="toast show">Saved</div>
```

```css
.toast{
  position:fixed; right:16px; bottom:16px; padding:.75rem 1rem;
  background:#111827; color:#fff; border-radius:12px; opacity:0; transform: translateY(8px);
}
.toast.show { animation: toast-in .28s cubic-bezier(.2,.6,.2,1) both; }
.toast.hide { animation: toast-out .24s ease-in both; }

@keyframes toast-in  { to { opacity:1; transform: translateY(0); } }
@keyframes toast-out { to { opacity:0; transform: translateY(8px); } }
```

### 6.4 드로어(사이드 패널)

```css
.drawer{
  position:fixed; inset:0 auto 0 0; width:min(80%, 360px);
  background:#fff; box-shadow:0 16px 40px rgba(0,0,0,.24);
  transform:translateX(-100%); opacity:0; pointer-events:none;
}
.drawer.open { animation: drawer-in .22s ease-out both; }
.drawer.closing { animation: drawer-out .2s ease-in both; }

@keyframes drawer-in  { to { transform:translateX(0); opacity:1; pointer-events:auto; } }
@keyframes drawer-out { to { transform:translateX(-100%); opacity:0; pointer-events:none; } }
```

### 6.5 버튼 피드백(프레스/릴리스)

```css
.btn {
  transition: transform .06s ease;
  will-change: transform;
}
.btn:active { transform: scale(.98); }
```

> 빠른 피드백은 `transition`이 낫습니다. 복잡한 단계는 `@keyframes`.

### 6.6 카운트업(단계형)

```html
<div class="counter" aria-live="polite">0</div>
```

```css
@keyframes count-10 {
  0% { content: "0"; }
  10% { content: "1"; }
  20% { content: "2"; }
  /* ... */
  100% { content: "10"; }
}
.counter::after{
  content:"0";
  animation: count-10 1s steps(10, end) forwards;
}
```

> 복잡한 숫자/포맷은 JS로 바꾸고 **애니메이션은 시각 효과만** 맡기는 편이 낫습니다.

### 6.7 카드 등장 스태거(Grid + 변수)

```css
.grid { display:grid; grid-template-columns:repeat(auto-fit, minmax(220px,1fr)); gap:16px; }
.grid > * {
  opacity:0; transform: translateY(12px);
  animation: fade-up .4s cubic-bezier(.2,.6,.2,1) both;
  animation-delay: calc(var(--row, 0) * 60ms + var(--col, 0) * 40ms);
}
```

```html
<!-- 행/열 인덱스는 서버·JS에서 계산해서 style에 세팅 -->
<div class="grid">
  <article style="--row:0;--col:0">A</article>
  <article style="--row:0;--col:1">B</article>
  <article style="--row:1;--col:0">C</article>
  <article style="--row:1;--col:1">D</article>
</div>
```

### 6.8 마퀴(무한 가로 스크롤)

```html
<div class="marquee">
  <div class="track">
    <span>NEWS · SALE · UPDATE · </span><span>NEWS · SALE · UPDATE · </span>
  </div>
</div>
```

```css
.marquee{ overflow:hidden; white-space:nowrap; }
.track{
  display:inline-block;
  animation: slide-left 12s linear infinite;
}
@keyframes slide-left {
  from { transform: translateX(0); }
  to   { transform: translateX(-50%); } /* 반복 문자열 길이에 맞춰 조정 */
}
```

---

## 7) 상태 기계처럼 쓰기 — 클래스 전환 & 이벤트 훅

```html
<button id="save">저장</button>
<div class="toast" id="t">Saved</div>
```

```css
/* 위 Toast 예제의 .show/.hide 애니메이션 재사용 */
```

```js
const t = document.getElementById('t');
document.getElementById('save').addEventListener('click', () => {
  t.classList.remove('hide');
  t.classList.add('show');
  const end = () => { t.classList.remove('show'); t.removeEventListener('animationend', end); };
  t.addEventListener('animationend', end);
  // N초 후 사라짐
  setTimeout(() => {
    t.classList.add('hide');
  }, 1500);
});
```

- `animationend`/`animationstart`/`animationiteration` 이벤트로 **상태 전환**을 정확히 동기화

---

## 8) 성능·접근성·호환성

### 8.1 성능

- **합성 단계**: `transform`, `opacity` 위주. `top/left/width/height`는 레이아웃/페인트 비용 ↑
- `will-change: transform;`은 **짧게** 사용(오픈 직전 ~ 종료 직후). 상시 사용은 메모리 낭비
- `transition/animation: all` 지양. 필요한 속성만
- DevTools → Performance → **Layout/paint/composite** 타임라인 확인

### 8.2 접근성

```css
@media (prefers-reduced-motion: reduce) {
  * { animation: none !important; transition: none !important; }
  /* 대체: 즉시 상태 적용 또는 더 느린 duration */
}
```

- 의미 전달이 핵심이면 **정적 대체**(아이콘 변화, 색상 변경 등) 제공

### 8.3 z-index/레이어

- 오버레이/모달/드로어는 **스태킹 컨텍스트** 영향(부모의 `transform`, `opacity`, `filter`)에 주의
- 애니메이션 도중 클릭 차단이 필요하면 `pointer-events` 제어

---

## 9) 고급 주제

### 9.1 여러 이름/여러 값 매칭 규칙

```css
.box {
  animation-name: fade-up, pulse;
  animation-duration: .4s, 1.2s;
  animation-iteration-count: 1, infinite; /* 순서 매칭 */
}
```

### 9.2 `animation-delay`·`iteration` 동작 상식

- 지연 동안 `backwards` 또는 `both`이면 **첫 프레임**이 적용
- `alternate` 반복에서는 **짝/홀수 루프**에 따라 진행 방향이 바뀜

### 9.3 `steps()` 파생 — 타이핑 효과

```css
.type {
  width: 24ch; overflow: hidden; border-right: .08em solid currentColor;
  animation:
    typing 2.5s steps(24, end) 0s 1 both,
    caret  .8s steps(1, end) infinite;
}
@keyframes typing { from { width: 0; } }
@keyframes caret  { 50% { border-color: transparent; } }
```

### 9.4 스크롤 진입 시 재생(간단, JS 한 줄)

```js
const io = new IntersectionObserver(ents=>{
  ents.forEach(e=>{
    if(e.isIntersecting){ e.target.classList.add('play'); io.unobserve(e.target); }
  });
});
document.querySelectorAll('.reveal').forEach(el=>io.observe(el));
```

```css
.reveal { opacity:0; transform: translateY(12px); }
.reveal.play { animation: fade-up .4s ease both; }
```

---

## 10) 데모 모음(복습)

### 10.1 “왼쪽 → 오른쪽” 이동 (초안 확장)

```css
@keyframes slide-right {
  0%   { transform: translateX(0); }
  100% { transform: translateX(300px); }
}
.box { width:100px; height:100px; background: coral; animation: slide-right 2s ease-in-out both; }
```

### 10.2 색상 단계 전환

```css
@keyframes rainbow {
  0%   { background: #ef4444; }
  25%  { background: #f59e0b; }
  50%  { background: #10b981; }
  75%  { background: #3b82f6; }
  100% { background: #8b5cf6; }
}
.tag { padding:.5rem 1rem; color:#fff; animation: rainbow 3s ease-in-out infinite alternate; }
```

### 10.3 점프 버튼(초안의 bounce 확장: 압축/이완)

```css
@keyframes jump {
  0%   { transform: translateY(0) scale(1); }
  50%  { transform: translateY(-18px) scale(1.02); }
  100% { transform: translateY(0) scale(.98); }
}
.button { animation: jump .5s cubic-bezier(.28,.84,.42,1) infinite; }
```

---

## 11) 흔한 이슈와 해결

| 이슈 | 설명 | 해결 |
|---|---|---|
| 애니메이션이 “먹지 않음” | `animation-name` 오타, 키프레임 이름 미일치 | 철자/스코프 확인 |
| 버벅임 | 레이아웃 속성 애니메이션, 그림자 과다 | `transform/opacity` 사용, 레이어 수 최소화 |
| 종료 후 상태가 초기화 | `fill-mode` 기본 `none` | `forwards` 또는 `both` |
| 모바일에서 깜빡임 | 저성능 GPU, 과도한 blur/filter | 효과 단순화, duration↑ |
| 스크롤 막힘 | 전체 화면 오버레이 중 배경 스크롤 | `body { overflow:hidden; }` 제어 |

---

## 12) 통합 예제 — 카드 리스트 등장 + Skeleton → 실제 콘텐츠 치환

```html
<section class="cards loading">
  <article class="card">
    <div class="skeleton" style="height:140px"></div>
    <div class="skeleton" style="height:16px;margin-top:12px;width:70%"></div>
  </article>
  <article class="card">...</article>
  <article class="card">...</article>
</section>
```

```css
.cards { display:grid; grid-template-columns:repeat(auto-fit, minmax(220px,1fr)); gap:16px; }
.card  { background:#fff; border-radius:12px; padding:12px; box-shadow:0 1px 2px rgba(0,0,0,.06); }

.skeleton{ position:relative; overflow:hidden; background:#e5e7eb; border-radius:8px; }
.skeleton::after{
  content:""; position:absolute; inset:0 -150%;
  background: linear-gradient(90deg, transparent, rgba(255,255,255,.6), transparent);
  animation: shimmer 1.1s ease-in-out infinite;
}
@keyframes shimmer { from{ transform:translateX(-40%);} to{ transform:translateX(40%);} }

.cards:not(.loading) .card{
  opacity:0; transform: translateY(10px);
  animation: fade-up .35s cubic-bezier(.2,.6,.2,1) both;
}
.cards:not(.loading) .card:nth-child(1){ animation-delay:  0ms; }
.cards:not(.loading) .card:nth-child(2){ animation-delay: 60ms; }
.cards:not(.loading) .card:nth-child(3){ animation-delay:120ms; }

@keyframes fade-up { to { opacity:1; transform: translateY(0); } }

@media (prefers-reduced-motion: reduce) {
  * { animation: none !important; transition: none !important; }
}
```

```js
// 데이터 로드 시 skeleton 제거 + 등장 애니메이션
setTimeout(()=>{
  document.querySelector('.cards')?.classList.remove('loading');
}, 1200);
```

---

## 요약

- `@keyframes`는 **시간 기반 단계**를 정의하고, `animation-*` 속성으로 **재생을 설계**한다.
- 간단한 상태 변화는 `transition`, **자체 구동/복잡 단계/반복**은 `animation`.
- **성능**은 `transform/opacity` 중심, **접근성**은 `prefers-reduced-motion` 존중.
- **스태거/병렬/직렬/상태 이벤트**를 조합하면 실전 UI/UX 대부분을 커버한다.

필요한 스니펫을 그대로 가져다 쓰면서, 프로젝트 컨텍스트에 맞게 곡선/시간/지연만 조정하면 됩니다.