---
layout: post
title: CSS - Animate.css
date: 2025-05-02 21:20:23 +0900
category: CSS
---
# 애니메이션 라이브러리 소개: **Animate.css** 완전 가이드

초안에 적어 둔 핵심(설치·기본 사용·옵션 클래스·스크롤 연동·성능 주의)을 살리면서,
**v4 기준 클래스 체계**, **CSS 커스텀 변수로 미세 제어**, **재실행 패턴**, **접근성/성능 체크리스트**,
**WOW/AOS 없이 IntersectionObserver로 트리거**, **프레임워크 연동 스니펫(React/Vue)**까지 확장했습니다.
필요한 코드는 바로 붙여 넣어 쓸 수 있도록 예제 중심으로 정리합니다.

---

## Animate.css 한 줄 정의

- **오픈소스** CSS 애니메이션 컬렉션 (by Daniel Eden, v4+)
- 요소에 `animate__animated` + **효과 이름**(예: `animate__fadeInUp`) 클래스를 붙이면 **즉시 실행**
- **유틸 클래스**(속도/지연/반복) + **CSS 변수**(`--animate-duration`, `--animate-delay`, `--animate-repeat`)로 미세 조정

---

## 설치

### CDN (가장 빠름)

```html
<link rel="stylesheet"
      href="https://cdnjs.cloudflare.com/ajax/libs/animate.css/4.1.1/animate.min.css" />
```

> `<head>`에 포함 후 바로 사용

### npm

```bash
npm install animate.css --save
```

```css
/* 전역 스타일 진입점에 */
@import "animate.css";
```

---

## 기본 사용법

```html
<div class="animate__animated animate__bounce">Hello!</div>
```

- `animate__animated`: 필수 베이스 클래스
- `animate__bounce`: 실행할 애니메이션 이름

### 가장 자주 쓰는 효과군(샘플)

| 카테고리 | 대표 클래스 |
|---|---|
| **Fade** | `animate__fadeIn`, `animate__fadeInUp/Down/Left/Right`, `animate__fadeOut*` |
| **Slide** | `animate__slideInUp/Down/Left/Right`, `animate__slideOut*` |
| **Zoom/Flip/Rotate** | `animate__zoomIn/Out`, `animate__flip`, `animate__flipInX/Y`, `animate__rotateIn/Out` |
| **Attention** | `animate__bounce`, `animate__pulse`, `animate__rubberBand`, `animate__shakeX/Y`, `animate__tada`, `animate__heartBeat` |

> 전체 목록은 공식 사이트에서 검색 가능.

---

## 클래스 & CSS 변수로 미세 제어

### 속도/지연/반복 (미리 정의된 유틸)

```html
<div class="animate__animated animate__fadeInUp animate__slow animate__delay-1s"></div>
```

- 속도: `animate__slow`(2s), `animate__slower`(3s), `animate__fast`(800ms), `animate__faster`(500ms)
- 지연: `animate__delay-1s` ~ `animate__delay-5s`
- 반복: `animate__infinite`, 또는 `animate__repeat-1`/`-2`/`-3`

### CSS 변수(권장, 더 유연)

```css
/* 전역 기본값 수정 */
:root {
  --animate-duration: 0.8s;
  --animate-delay: 0s;
  --animate-repeat: 1;
}

/* 특정 컴포넌트만 빠르게 */
.hero-in {
  --animate-duration: .5s;
  --animate-delay: .15s;
}
```

```html
<h1 class="animate__animated animate__fadeInUp hero-in">Welcome</h1>
```

> 유틸 클래스로 충분하면 그대로 사용, **세밀 조정이 필요하면 CSS 변수**를 덮어씁니다.

---

## 즉시 써먹는 실전 스니펫

### “아래에서 위로 페이드 인” 히어로 카피

```html
<h1 class="animate__animated animate__fadeInUp animate__faster">
  Build fast. Ship faster.
</h1>
```

### 버튼 Hover 한 번만 “Pulse” (재실행 패턴 포함)

```html
<button class="btn">Hover Me</button>
```

```css
.btn { transition: transform .12s ease; }
.btn:hover { transform: translateY(-1px); }
```

```js
const btn = document.querySelector('.btn');
btn.addEventListener('mouseenter', () => {
  btn.classList.add('animate__animated','animate__pulse');
});
btn.addEventListener('animationend', () => {
  // 끝나면 제거해서 다음 hover 때 재실행 가능
  btn.classList.remove('animate__animated','animate__pulse');
});
```

### 카드 그리드 스태거(순차 지연)

```html
<div class="grid">
  <article class="card animate__animated animate__fadeInUp" style="--animate-delay:0ms"></article>
  <article class="card animate__animated animate__fadeInUp" style="--animate-delay:80ms"></article>
  <article class="card animate__animated animate__fadeInUp" style="--animate-delay:160ms"></article>
</div>
```

> 숫자를 서버/JS에서 루프 돌며 넣으면 유지보수 용이.

### 알림 Toast (등장/퇴장 두 효과)

```html
<div id="toast" class="toast animate__animated" hidden>Saved!</div>
```

```css
.toast {
  position: fixed; right: 16px; bottom: 16px;
  background: #111827; color: #fff; padding: .75rem 1rem;
  border-radius: 12px;
}
```

```js
const toast = document.getElementById('toast');
function showToast() {
  toast.hidden = false;
  toast.classList.remove('animate__fadeOutDown');
  toast.classList.add('animate__fadeInUp');
  toast.addEventListener('animationend', onInDone, { once: true });
}
function onInDone() {
  setTimeout(() => {
    toast.classList.remove('animate__fadeInUp');
    toast.classList.add('animate__fadeOutDown');
    toast.addEventListener('animationend', () => (toast.hidden = true), { once: true });
  }, 1400);
}
```

---

## 스크롤 진입 시 애니메이션(Intersection Observer만으로)

WOW/AOS 없이도 **표준 API**로 충분히 구현 가능합니다.

```html
<section>
  <h2 class="reveal animate__animated" data-anim="animate__fadeInUp">Title</h2>
  <p class="reveal animate__animated" data-anim="animate__fadeIn" style="--animate-delay:.08s">
    Content...
  </p>
</section>
```

```js
const io = new IntersectionObserver((entries) => {
  entries.forEach(e => {
    if (e.isIntersecting) {
      const el = e.target;
      el.classList.add(el.dataset.anim || 'animate__fadeInUp');
      io.unobserve(el); // 1회만
    }
  });
}, { threshold: 0.15 });

document.querySelectorAll('.reveal').forEach(el => io.observe(el));
```

> 여러 번 재생하고 싶다면 `unobserve`를 제거하고 `rootMargin`/`threshold`를 조정하세요.

---

## WOW.js / AOS와의 연동(선택)

### WOW.js

```html
<div class="wow animate__animated animate__fadeInUp">Hello</div>

<script src="wow.min.js"></script>
<script> new WOW().init(); </script>
```

### AOS (별도 클래스 체계)

AOS는 자체 `data-aos` 속성을 사용합니다. Animate.css와 **함께 쓸 수는 있지만**,
같은 요소에 **중복 애니메이션**을 얹지 않도록 주의하세요.

---

## 재실행/한정 실행/상태 제어 패턴

### `animationend`로 클래스 제거

```js
function playAnim(el, name){
  el.classList.remove('animate__animated', name);
  // 리플로우 강제(브라우저가 변화 인지하도록)
  void el.offsetWidth;
  el.classList.add('animate__animated', name);
  el.addEventListener('animationend', () => {
    el.classList.remove('animate__animated', name);
  }, { once: true });
}
```

```js
playAnim(document.querySelector('.badge'), 'animate__tada');
```

### 최초 1회만

```js
const once = document.querySelectorAll('.once');
once.forEach(el => {
  el.classList.add('animate__animated','animate__fadeInUp');
  el.addEventListener('animationend', () => el.classList.remove('animate__animated'), { once: true });
});
```

### “열기/닫기”에 서로 다른 애니메이션

```js
function openDrawer(el){
  el.classList.remove('animate__fadeOutLeft');
  el.classList.add('animate__animated','animate__fadeInLeft');
}
function closeDrawer(el){
  el.classList.remove('animate__fadeInLeft');
  el.classList.add('animate__animated','animate__fadeOutLeft');
}
```

---

## 성능·접근성 체크리스트

### 성능

- 가능한 한 **`transform`, `opacity` 기반** 효과 사용(합성 단계, 부드러움 ↑)
- 동일 시점에 **과도한 요소**에 애니메이션 금지(특히 모바일)
- 긴 리스트는 **스태거**(지연 분산)로 **동시 레이어 폭발** 방지
- `transition/animation: all` 지양, 필요한 속성만 지정
- DevTools Performance 탭으로 Layout/Paint/Composite 확인

### 접근성 — `prefers-reduced-motion`

```css
@media (prefers-reduced-motion: reduce) {
  .animate__animated {
    animation: none !important;
    transition: none !important;
  }
}
```

- 핵심 상태는 **정적 스타일**로도 전달되도록 설계(색상·모양 변화 등)

---

## 커스터마이징(선택)

- Animate.css는 **Sass** 기반 — 필요 효과만 빌드하여 **크기 최적화** 가능
- 또는 애니메이션 이름은 유지하고 **CSS 변수**로 팀 가이드에 맞게 전역 Duration/Delay를 조정

```css
/* 전역 톤앤매너 */
:root { --animate-duration: .45s; }
```

---

## 실전 컴포넌트 레시피

### 배너 “ZoomIn + 페이드 업” 병렬

```html
<div class="banner animate__animated animate__zoomIn animate__fadeInUp"
     style="--animate-duration:.6s"></div>
```

> Animate.css는 **여러 이름**을 공존시켜 병렬 재생 가능(브라우저별 순서 이슈가 있으면 하나로 충분한지 검토).

### Skeleton → 실제 카드 전환

```html
<article class="card">
  <div class="skeleton"></div>
  <div class="real animate__animated" hidden>Loaded</div>
</article>
```

```css
.skeleton{ height:120px; border-radius:12px; background:#e5e7eb; position:relative; overflow:hidden; }
.skeleton::after{
  content:""; position:absolute; inset:0 -150%;
  background: linear-gradient(90deg, transparent, rgba(255,255,255,.6), transparent);
  animation: shimmer 1.1s ease-in-out infinite;
}
@keyframes shimmer { from{ transform:translateX(-40%);} to{ transform:translateX(40%);} }
```

```js
// 로드 완료 시
const real = document.querySelector('.real');
const skel = document.querySelector('.skeleton');
skel.hidden = true;
real.hidden = false;
real.classList.add('animate__fadeInUp');
```

### Nav 드롭다운: “slideInDown/slideOutUp”

```html
<nav>
  <button id="menuBtn">Menu</button>
  <div id="menu" class="animate__animated" hidden>
    ...
  </div>
</nav>
```

```js
const menu = document.getElementById('menu');
document.getElementById('menuBtn').addEventListener('click', () => {
  if (menu.hidden) {
    menu.hidden = false;
    menu.classList.remove('animate__slideOutUp');
    menu.classList.add('animate__slideInDown');
  } else {
    menu.classList.remove('animate__slideInDown');
    menu.classList.add('animate__slideOutUp');
    menu.addEventListener('animationend', () => (menu.hidden = true), { once: true });
  }
});
```

---

## 프레임워크 연동 스니펫

### React (컴포넌트 마운트 때 1회 등장)

```jsx
import "animate.css";

export default function FadeInSection({ children }) {
  const ref = React.useRef(null);
  React.useEffect(() => {
    const el = ref.current;
    el.classList.add("animate__animated","animate__fadeInUp");
    const off = () => el.classList.remove("animate__animated","animate__fadeInUp");
    el.addEventListener("animationend", off, { once: true });
    return () => el.removeEventListener("animationend", off);
  }, []);
  return <div ref={ref}>{children}</div>;
}
```

### Vue (v-if로 등장/퇴장)

```html
<template>
  <transition
    enter-active-class="animate__animated animate__fadeInUp"
    leave-active-class="animate__animated animate__fadeOutDown">
    <div v-if="open"><slot/></div>
  </transition>
</template>

<script setup>
import "animate.css";
defineProps({ open: Boolean });
</script>
```

---

## 디버깅 FAQ

| 증상 | 원인 | 해결 |
|---|---|---|
| “아무 일도 안 일어남” | `animate__animated` 누락, 오타 | 두 클래스 모두 확인 |
| 한 번만 되고 다시 안 됨 | 클래스가 유지되어 **재트리거 불가** | `animationend`에서 클래스 제거 후 재첨부 |
| 너무 무거움 | 동시 실행 요소 많음, 복잡한 효과(blur/box-shadow) | 핵심 요소만, transform/opacity 위주, 스태거 사용 |
| 페이지 멀미 호소 | 모션 과다, 장시간 반복 | `prefers-reduced-motion` 존중, 반복 최소화 |
| 다른 CSS와 충돌 | 동일 속성 덮어쓰기, 우선순위 문제 | 스코프 분리, 필요 시 `!important` (최소화) |

---

## 종합 예제 — 랜딩 히어로 + 스크롤 리빌 + 버튼 피드백

```html
<header class="hero">
  <h1 class="animate__animated animate__fadeInUp" style="--animate-duration:.6s">
    Ship delightful apps
  </h1>
  <p  class="animate__animated animate__fadeInUp" style="--animate-delay:.12s">Modern tooling for teams.</p>
  <button id="cta" class="cta">Get Started</button>
</header>

<section>
  <article class="reveal animate__animated" data-anim="animate__fadeInUp" style="--animate-delay:.04s">A</article>
  <article class="reveal animate__animated" data-anim="animate__fadeInUp" style="--animate-delay:.08s">B</article>
  <article class="reveal animate__animated" data-anim="animate__fadeInUp" style="--animate-delay:.12s">C</article>
</section>
```

```css
.hero { padding: 8rem 1.5rem; text-align: center; }
.cta  { padding: .9rem 1.4rem; border-radius: 12px; background:#111827; color:#fff; border:0; }
.cta:active { transform: scale(.98); }
```

```js
// 스크롤 리빌
const io = new IntersectionObserver((ents)=>{
  ents.forEach(e=>{
    if(e.isIntersecting){
      e.target.classList.add(e.target.dataset.anim);
      io.unobserve(e.target);
    }
  });
},{threshold:.15});
document.querySelectorAll('.reveal').forEach(el=>io.observe(el));

// CTA 누를 때 피드백 애니메이션 재실행
const cta = document.getElementById('cta');
cta.addEventListener('click', ()=>{
  cta.classList.remove('animate__animated','animate__heartBeat');
  void cta.offsetWidth; // reflow
  cta.classList.add('animate__animated','animate__heartBeat');
  cta.addEventListener('animationend', ()=>cta.classList.remove('animate__animated','animate__heartBeat'), { once:true });
});
```

---

## 참고/요약

- **설치**: CDN 또는 npm, 바로 사용
- **핵심 클래스**: `animate__animated` + `animate__<Effect>`
- **조절**: 유틸(속도/지연/반복) 또는 **CSS 변수**로 미세 제어
- **트리거**: hover/클릭/스크롤(IntersectionObserver), 혹은 WOW/AOS
- **품질**: transform/opacity 위주, 스태거로 자연스러움, `prefers-reduced-motion` 고려

**Animate.css**는 “빠르게 생동감”을 주는 데 훌륭합니다.
브랜드 고유 모션이나 섬세한 단계가 필요하면 Animate.css로 **프로토타입** → 필요 지점만 `@keyframes` 커스텀으로 확장하세요.

---

### 🔗 링크 모음

- 공식: https://animate.style/
- GitHub: https://github.com/animate-css/animate.css
- WOW.js: https://wowjs.uk/
- AOS: https://michalsnik.github.io/aos/
