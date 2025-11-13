---
layout: post
title: CSS - transition과 transform
date: 2025-04-21 21:20:23 +0900
category: CSS
---
# CSS `transition`과 `transform`

## 0. 개요

- **`transition`**: 어떤 *상태 변화*가 일어날 때, 그 *변화 과정*을 부드럽게 보이게 함.
- **`transform`**: 요소의 기하학적 변형(이동/회전/확대/기울임)을 시각적으로 적용. 레이아웃엔 영향 없음(= reflow 유발 X).

둘을 함께 쓰면, 클릭/호버/토글/스크롤 인터랙션을 **짧은 코드로** 유연하게 구현할 수 있습니다.

---

## 1. `transition` 핵심

```css
.box {
  transition: transform 300ms ease, opacity 200ms linear 50ms;
}
/* 축약형: [property] [duration] [timing] [delay] */
```

### 1.1 속성 분해
| 속성 | 의미 | 예시 |
|---|---|---|
| `transition-property` | 애니메이션 대상 속성 | `transform, opacity` |
| `transition-duration` | 지속시간 | `300ms`, `0.2s` |
| `transition-timing-function` | 속도 곡선 | `ease`, `linear`, `ease-in-out`, `cubic-bezier(...)` |
| `transition-delay` | 지연 | `120ms` |

> 권장: `transition: all ...`은 피하고 **필요한 속성만** 지정(성능/예측성↑).

### 1.2 애니메이션 가능한 속성(요지)
- **권장**: `opacity`, `transform` 계열(translate/scale/rotate) → 합성(compositing) 단계에서 처리되어 부드럽고 비용 낮음.
- **주의**: `width/height/left/top/margin` 등 레이아웃에 영향 주는 속성은 **reflow**를 유발, 프레임 드랍 위험.

> 가능한 한 **opacity/transform**로 대체하는 패턴을 우선 검토.

---

## 2. `transform` 핵심

```css
.target {
  transform: translateX(24px) rotate(6deg) scale(1.05);
}
```

### 2.1 주요 함수(2D)
| 함수 | 설명 | 예 |
|---|---|---|
| `translate(x, y)` | 평행 이동 | `translate(20px, -10px)` |
| `translateX/Y` | 축 별 이동 | `translateY(50%)` |
| `scale(x, y)` | 확대/축소 | `scale(1.2, .9)` |
| `scaleX/Y` | 축 별 스케일 | `scaleX(1.1)` |
| `rotate(a)` | 회전(시계) | `rotate(15deg)` |
| `skew(xa, ya)` | 기울이기 | `skew(10deg, 0)` |
| `matrix(a,b,c,d,e,f)` | 행렬 직접 지정 | 고급용 |

### 2.2 트랜스폼 순서 중요
```css
/* 회전 후 확대 */
transform: rotate(45deg) scale(1.2);
/* 확대 후 회전 → 최종 좌표/모양이 달라짐 */
transform: scale(1.2) rotate(45deg);
```
복수 함수는 **왼쪽부터** 차례로 적용됩니다.

### 2.3 기준점: `transform-origin`
```css
.box {
  transform-origin: bottom right; /* 기본: center center */
  transform: rotate(12deg);
}
```

### 2.4 3D 확장
| 함수 | 설명 | 예 |
|---|---|---|
| `translateZ(z)` | Z축 이동 | `translateZ(40px)` |
| `scaleZ(s)` | Z축 스케일 | `scaleZ(1.2)` |
| `rotateX/Y/Z(a)` | 축 회전 | `rotateY(25deg)` |
| `perspective(d)` | 원근감(요소별) | `perspective(800px) rotateY(25deg)` |

관련 속성:
```css
.scene {
  perspective: 800px;           /* 컨테이너에서 원근 설정 */
  perspective-origin: 50% 50%;  /* 시점 원점 */
}
.card {
  transform-style: preserve-3d; /* 자식의 3D 유지 */
  backface-visibility: hidden;  /* 뒤집힐 때 뒷면 숨김 */
}
```

---

## 3. 타이밍 함수 깊게 보기

타이밍 함수는 시간 \(t \in [0,1]\)에 대한 *진행도* \(p = f(t)\)를 정의합니다.
대표적인 **Cubic Bezier**는 네 점 \((0,0), (x_1, y_1), (x_2, y_2), (1,1)\)로 정의:

\[
\mathbf{B}(t) = (1-t)^3 \mathbf{P}_0 + 3(1-t)^2 t \mathbf{P}_1 + 3(1-t)t^2 \mathbf{P}_2 + t^3 \mathbf{P}_3
\]

CSS에서는 `cubic-bezier(x1, y1, x2, y2)`로 지정합니다.

```css
/* 느리게 시작, 빠르게 끝 (커스텀) */
transition-timing-function: cubic-bezier(.2, .7, .2, 1);
```

자주 쓰는 프리셋:
- `ease`(기본), `ease-in`, `ease-out`, `ease-in-out`, `linear`
- 자연스러운 인터랙션을 위해 **ease-out** 계열이 버튼/카드에 자주 쓰임.

---

## 4. 상태 변화 패턴 모음

### 4.1 카드 호버 리프트(권장 패턴: transform+shadow)
```html
<div class="card">콘텐츠</div>
```

```css
.card {
  background: #fff;
  border-radius: 12px;
  box-shadow: 0 1px 2px rgba(0,0,0,.06);
  transform: translateY(0);
  transition: transform .2s ease, box-shadow .2s ease;
}
.card:hover {
  transform: translateY(-6px);
  box-shadow: 0 10px 24px rgba(0,0,0,.14);
}
```

### 4.2 버튼 프레스(누르는 느낌)
```css
.button {
  transform: translateY(0);
  transition: transform 120ms ease, box-shadow 120ms ease;
}
.button:active {
  transform: translateY(1px) scale(.98);
  box-shadow: 0 2px 8px rgba(0,0,0,.12) inset;
}
```

### 4.3 토글 아이콘(햄버거→X)
```html
<button class="burger" aria-expanded="false"><span></span></button>
```

```css
.burger {
  width: 40px; height: 40px; position: relative; border:0; background:transparent;
}
.burger span,
.burger::before,
.burger::after {
  content:""; position:absolute; left:8px; right:8px; height:2px; background:#111;
  transition: transform .2s ease, opacity .2s ease;
}
.burger span { top: 19px; }
.burger::before { top: 12px; }
.burger::after  { top: 26px; }

.burger[aria-expanded="true"]::before { transform: translateY(7px) rotate(45deg); }
.burger[aria-expanded="true"] span    { opacity:0; }
.burger[aria-expanded="true"]::after  { transform: translateY(-7px) rotate(-45deg); }
```

### 4.4 아코디언(높이 애니메이션의 한계와 우회)
`height: auto`는 **직접 애니메이션 불가**. `max-height` 트릭 또는 `grid`/`scaleY`/`content-visibility` 응용.

```css
.accordion__panel {
  max-height: 0;
  overflow: hidden;
  transition: max-height .3s ease;
}
.accordion__panel[aria-hidden="false"] {
  max-height: 600px; /* 충분히 큰 값(콘텐츠 예상치 이상) */
}
```

또는 `scaleY`:
```css
.panel {
  transform: scaleY(0);
  transform-origin: top;
  transition: transform .25s ease;
}
.panel[aria-hidden="false"] { transform: scaleY(1); }
```
> 시각은 부드럽지만 내부 콘텐츠 자동 레이아웃과의 정밀도는 `max-height`가 더 나을 때도 있음.

---

## 5. 이벤트 훅: `transitionend`

상태 전환 완료 후 DOM 조작이 필요할 때:

```js
const panel = document.querySelector('.panel');
panel.addEventListener('transitionend', (e) => {
  if (e.propertyName === 'transform') {
    // 전환 완료 후 포커스 이동, aria 속성 갱신 등
  }
});
```

여러 속성을 전환하면 이벤트가 **여러 번** 발생할 수 있으므로 `propertyName` 필터링 권장.

---

## 6. 성능 체크리스트

- **합성 단계**에서 끝나는 속성만 전환: `transform`, `opacity`
- `top/left/width/height` 등 레이아웃 변화는 피하기 → 필요하면 `transform`으로 **대체**
- `will-change`는 **짧게** 사용(전환 직전/호버 시): 과다 사용시 메모리/타일 증가
  ```css
  .card:hover { will-change: transform; } /* 지속적 고정 지정은 지양 */
  ```
- 초당 60프레임 목표. DevTools Performance/Rendering에서 FPS/합성 경로 확인.
- 모바일 GPU 과부하 주의(과도한 블러/박스 그림자/반투명 대면적).

---

## 7. 접근성 고려

- 모션 민감 사용자: `prefers-reduced-motion` 반영
```css
@media (prefers-reduced-motion: reduce) {
  * { transition: none !important; animation: none !important; }
}
```
- 초점 가시성 유지: 포커스 상태에서 `transform`으로 요소가 움직이며 **포커스 링**이 시각적으로 어긋나지 않도록 `outline`/`focus-visible` 스타일을 함께 조정.
- 의미 전달이 애니메이션에만 의존하지 않게 설계.

---

## 8. 스태킹/레이어링과 트랜스폼

- `transform`을 적용하면 요소는 **새 스태킹 컨텍스트**를 생성(일반적으로).
- `z-index` 계산이 달라질 수 있으니, 모달/툴팁/드롭다운 계층은 **레이어 토큰**으로 명시 관리:
```css
:root { --z-modal: 1000; --z-popover: 900; --z-overlay: 800; }
.modal { position: fixed; z-index: var(--z-modal); }
```

---

## 9. 3D 카드 플립 예제

```html
<div class="scene">
  <div class="card3d">
    <div class="face front">Front</div>
    <div class="face back">Back</div>
  </div>
</div>
```

```css
.scene {
  width: 280px; height: 180px; perspective: 1000px; margin: 2rem auto;
}
.card3d {
  position: relative; width: 100%; height: 100%;
  transform-style: preserve-3d;
  transition: transform .6s cubic-bezier(.2,.6,.2,1);
}
.card3d:hover { transform: rotateY(180deg); }

.face {
  position:absolute; inset:0; display:grid; place-items:center;
  border-radius: 12px; backface-visibility: hidden; font-weight:700;
  box-shadow: 0 10px 30px rgba(0,0,0,.15);
}
.front { background:#fff; }
.back  { background:#0f172a; color:#fff; transform: rotateY(180deg); }
```

---

## 10. 실전 컴포넌트: 토스트 알림

```html
<button id="show">토스트</button>
<div class="toast" role="status" aria-live="polite">저장되었습니다</div>
```

```css
.toast {
  position: fixed; right: 16px; bottom: 16px;
  transform: translateY(16px); opacity: 0;
  background: #111; color:#fff; padding: .75rem 1rem; border-radius: 10px;
  transition: transform .24s ease, opacity .24s ease;
}
.toast--visible {
  transform: translateY(0); opacity: 1;
}
```

```js
const btn = document.getElementById('show');
const toast = document.querySelector('.toast');

btn.addEventListener('click', () => {
  toast.classList.add('toast--visible');
  setTimeout(() => toast.classList.remove('toast--visible'), 1800);
});
```

> `opacity + translate` 조합은 시각적으로 자연스럽고 합성 단계에서 매끄럽게 동작.

---

## 11. 스크롤 구동 인터랙션(기본형)

스크롤에 따라 클래스만 토글하고, 실제 애니메이션은 `transition`이 처리.

```css
.header { transform: translateY(0); transition: transform .2s ease; }
.header--hide { transform: translateY(-100%); }
```

```js
let prev = window.scrollY;
const header = document.querySelector('.header');

addEventListener('scroll', () => {
  const y = window.scrollY;
  header.classList.toggle('header--hide', y > prev && y > 80);
  prev = y;
});
```

---

## 12. 개별 트랜스폼 속성(Transforms Level 2)

현대 브라우저는 `transform` 단일 속성 외에 **개별 속성**도 지원:

```css
.box {
  translate: 10px 0; /* x y */
  rotate: 6deg;
  scale: 1.1;        /* x=y */
  transition: translate .2s ease, rotate .2s ease;
}
```

장점: 다른 곳에서 `transform`을 재할당해도 **부분 충돌**이 감소.

---

## 13. 디버깅 팁

- DevTools → **Elements** 패널에서 `:hov` 상태 강제, `computed`에서 전환 중 속성 확인.
- **Performance**/**Layers**(또는 Rendering)로 합성 경로와 레이어 승격 여부 확인.
- 경계 조건: 전환 시작/끝 상태가 동일하면 `transitionend`가 발생하지 않을 수 있음(브라우저별). 로직에 방어 코드 필요.

---

## 14. 흔한 함정과 해결책

| 문제 | 원인 | 해결 |
|---|---|---|
| `height:auto` 전환 안 됨 | 애니메이션 불가 값 | `max-height` 트릭/`scaleY`/FLIP 기법 |
| `transform` 후 z-index 이상 | 새 스태킹 컨텍스트 | 의도된 레이어 설계, 레이어 토큰 사용 |
| 고사양에서만 부드럽고 모바일에서 끊김 | 레이아웃/그림자/필터 전환 | `transform/opacity` 중심, 그림자 강도 축소 |
| `transition: all` 오작동 | 원치 않는 속성도 전환 | 필요한 속성만 명시 |
| 포커스 링 사라짐 | outline 제거 | `:focus-visible`로 대안 제공, 키보드 접근성 유지 |

---

## 15. 종합 예제: 카드 리스트

```html
<ul class="cards">
  <li class="card">
    <h3>빠른 전환</h3>
    <p>transform/opacity 중심의 합성 전환</p>
  </li>
  <li class="card">
    <h3>부드러운 곡선</h3>
    <p>cubic-bezier로 상호작용 감성 튜닝</p>
  </li>
  <li class="card">
    <h3>모션 배려</h3>
    <p>prefers-reduced-motion 존중</p>
  </li>
</ul>
```

```css
.cards { display:grid; grid-template-columns: repeat(auto-fit, minmax(220px, 1fr)); gap: 16px; padding: 16px; }
.card {
  background:#fff; border-radius:14px; padding:16px;
  box-shadow: 0 1px 2px rgba(0,0,0,.06);
  transform: translateY(0) scale(1);
  transition: transform .18s cubic-bezier(.2,.6,.2,1), box-shadow .2s ease;
}
.card:hover {
  transform: translateY(-4px) scale(1.02);
  box-shadow: 0 10px 24px rgba(0,0,0,.14);
}
@media (prefers-reduced-motion: reduce) {
  .card { transition: none; }
}
```

---

## 16. 요약

| 항목 | 핵심 정리 |
|---|---|
| 전환 대상 | `opacity`/`transform` 우선(합성 처리) |
| 설계 | 상태(클래스)만 바꾸고, 전환은 CSS가 처리 |
| 타이밍 | `ease-out` 계열이 상호작용에 자연스러움 |
| 3D | `perspective`/`preserve-3d`/`backface-visibility` |
| 접근성 | `prefers-reduced-motion` 대응 필수 |
| 성능 | `will-change`는 국소적으로, `all` 지양 |
| 제약 | `auto` 값 전환 불가 → 우회(트릭/FLIP) |

---

## 참고
- MDN: transition / transform
- CSS Transforms Level 2(개별 변환 속성: `translate/scale/rotate`)
- Core Web Vitals를 고려한 합성 중심 애니메이션 전략
