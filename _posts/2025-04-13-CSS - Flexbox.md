---
layout: post
title: CSS - Flexbox
date: 2025-04-06 21:20:23 +0900
category: CSS
---
# Flexbox 완전 가이드 (Flexbox Complete Guide)

## 0. 큰 그림 한 장

- **Flex Container**: `display: flex | inline-flex` (축 설정, 줄바꿈, 정렬, 간격)
- **Flex Item**: `flex: grow shrink basis`(크기 정책), `align-self`, `order`
- **1차원 레이아웃**(주축 중심)에서 **정렬/간격/늘이고 줄이기**를 가장 간단히 해결

---

## 1. 핵심 개념 복습

```html
<div class="flex">
  <div>Item 1</div>
  <div>Item 2</div>
  <div>Item 3</div>
</div>
```

```css
.flex { display: flex; }
```

- **주축(main axis)**: `flex-direction`이 정하는 방향 (`row`=가로, `column`=세로)
- **교차축(cross axis)**: 주축과 직교하는 방향
- **여유 공간 분배**: `flex-grow`/`flex-shrink`/`flex-basis` 조합으로 결정

> 그리드는 **2차원(행+열)** 전체 영역 설계에 강함, Flex는 **콘텐츠 흐름 중심 1차원 정렬**에 강함.

---

## 2. 컨테이너 속성 총정리

| 속성 | 값(대표) | 설명 |
|---|---|---|
| `display` | `flex` \| `inline-flex` | flex 컨텍스트 생성 |
| `flex-direction` | `row` \| `row-reverse` \| `column` \| `column-reverse` | 주축 방향 |
| `flex-wrap` | `nowrap` \| `wrap` \| `wrap-reverse` | 줄바꿈 정책 |
| `flex-flow` | `<direction> <wrap>` | 방향+줄바꿈 단축 |
| `justify-content` | `flex-start` `center` `flex-end` `space-between` `space-around` `space-evenly` | **주축 정렬** |
| `align-items` | `stretch` `flex-start` `center` `flex-end` `baseline` | **교차축 정렬(단일 줄)** |
| `align-content` | `flex-start` `center` `flex-end` `space-between` `space-around` `space-evenly` `stretch` | **여러 줄 간 정렬** |
| `gap` | `row-gap`/`column-gap` | 아이템 간 간격(마진 대신 권장) |

### 2.1 자주 쓰는 컨테이너 프리셋

```css
/* 수평 중앙 정렬(가운데-가운데) */
.center-center {
  display: flex;
  justify-content: center;
  align-items: center;
}

/* 양끝 정렬(헤더/푸터) */
.space-between {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

/* 세로 스택 + 중앙 */
.col-center {
  display: flex;
  flex-direction: column;
  align-items: center;
}

/* 감싸며 카드 그리드 느낌 */
.wrap-gap {
  display: flex;
  flex-wrap: wrap;
  gap: 1rem;
}
```

---

## 3. 아이템 속성 총정리

| 속성 | 설명 |
|---|---|
| `flex-grow` | 여유 공간을 **비율대로 확장** |
| `flex-shrink` | 공간 부족 시 **비율대로 축소** |
| `flex-basis` | 아이템의 **초기 크기**(폭/높이) |
| `flex` | `grow shrink basis` 단축 |
| `align-self` | 개별 아이템 교차축 정렬(컨테이너의 `align-items` 덮어씀) |
| `order` | **시각적 순서**(DOM 순서와 별개 → 접근성 유의) |

### 3.1 `flex` 단축 사용 패턴

```css
/* grow=1, shrink=1, basis=0% → 남은 공간을 균등 분배(그리드 흉내) */
.flex-1 { flex: 1 1 0; }

/* grow=1, shrink=0, basis=auto → 줄어들지 않고 남는 공간을 가져감 */
.flex-auto { flex: 1 0 auto; }

/* grow=0, shrink=0, basis=auto → 고정(늘/줄지 않음) */
.flex-none { flex: 0 0 auto; }

/* grow=0, shrink=1, basis=200px → 기본 200px, 부족하면 줄어듦 */
.flex-b200 { flex: 0 1 200px; }
```

> **주의**: `flex-basis`가 `auto`일 땐 **content size 또는 `width/height`**가 초기 크기에 관여.  
> `flex-basis: 0`은 “기본 폭을 0으로 보고 **남는 공간만** 비율로 분배”하겠다는 뜻.

---

## 4. 정렬 속성 심화

### 4.1 주축 정렬 — `justify-content`
```css
.toolbar {
  display: flex;
  justify-content: space-between;
  gap: .75rem;
}
```

### 4.2 교차축 정렬 — `align-items`
```css
.row {
  display: flex;
  align-items: center; /* 다른 높이의 버튼/텍스트 baseline/center 정렬 */
  gap: .5rem;
}
```

### 4.3 여러 줄 정렬 — `align-content` (wrap 필요)
```css
.grid-ish {
  display: flex; flex-wrap: wrap;
  align-content: space-between; /* 줄과 줄 사이 간격 */
  height: 60vh;
}
```

### 4.4 개별 아이템 정렬 — `align-self`
```css
.item-end { align-self: flex-end; }
```

### 4.5 자동 마진으로 정렬
```css
.header {
  display: flex; gap: 1rem;
}
.header .spacer { margin-left: auto; } /* 나머지를 오른쪽으로 밀기 */
```

---

## 5. 수학으로 보는 공간 분배(직관)

여유 공간이 \(F\), 각 아이템의 `flex-grow`가 \(g_i\)일 때  
**증가분** \(\Delta w_i\)는

$$
\Delta w_i = 
\begin{cases}
\frac{g_i}{\sum_j g_j}\cdot F & \text{(공간이 남을 때)} \\
0 & \text{(공간이 없을 때)}
\end{cases}
$$

공간이 부족할 때는 각 아이템의 `flex-shrink` \(s_i\)와 **기준 크기**(보통 `flex-basis` 또는 content size)에 비례하여 줄어듭니다(브라우저별 구현 세부차 존재).

---

## 6. `gap` 적극 활용(마진보다 안전하고 직관적)

```css
.row-gap-1 {
  display: flex;
  gap: 1rem;          /* 가로/세로 간격 모두 */
}
.col-gap {
  display: flex;
  flex-direction: column;
  row-gap: .5rem;     /* 세로 간격만 */
}
```

- **장점**: 마진 중첩/첫·마지막 요소 여백 예외 처리 불필요, **RTL/쓰기방향**에도 안전.

---

## 7. 반응형·실전 패턴 모음

### 7.1 수직 가운데 정렬(뷰포트 가득)
```css
.full-center {
  min-height: 100vh;
  display: flex;
  justify-content: center;
  align-items: center;
}
```

### 7.2 네비게이션 바(양끝 정렬)
```css
.navbar {
  display: flex; align-items: center;
  justify-content: space-between; gap: 1rem;
  padding: 1rem;
}
```

### 7.3 카드 그리드(감싸기 + 최소 폭)
```css
.cards {
  display: flex; flex-wrap: wrap; gap: 1rem;
}
.card {
  flex: 1 1 220px;   /* 최소 220px, 남으면 늘어남 */
  border: 1px solid #d1d5db; border-radius: .75rem; padding: 1rem;
}
```

### 7.4 균등 폭 버튼 줄
```css
.btns {
  display: flex; gap: .5rem;
}
.btns > button { flex: 1 1 0; } /* 모두 같은 비율로 채움 */
```

### 7.5 Sticky Footer(콘텐츠가 짧아도 푸터 하단)
```css
.page {
  min-height: 100vh;
  display: flex; flex-direction: column;
}
.page main { flex: 1 0 auto; }
.page footer { margin-top: auto; }
```

### 7.6 사이드바 레이아웃(고정폭 + 유동 본문)
```css
.layout {
  display: flex; min-height: 100vh;
}
.sidebar { flex: 0 0 280px; border-right: 1px solid #e2e8f0; }
.content { flex: 1 1 auto; padding: 1.25rem; }
```

### 7.7 입력+버튼 결합(한 줄, 남는 공간 입력이 차지)
```css
.input-row {
  display: flex; gap: .5rem;
}
.input-row input { flex: 1 1 auto; }
.input-row button { flex: 0 0 auto; }
```

### 7.8 프로필 행(baseline·center 비교)
```css
.profile {
  display: flex; gap: .75rem; align-items: center;
}
.profile.baseline { align-items: baseline; } /* 텍스트 기준선 맞추기 */
```

---

## 8. Flex vs width/height — 충돌·우선순위 이해

- `flex-basis`와 `width/height`가 **둘 다** 존재할 때:
  - 기본적으로 **`flex-basis`가 우선**(단, `flex-basis:auto`이면 width/height 또는 content size가 반영)
- 고정폭 아이템은 `flex: 0 0 200px`처럼 명시해 **늘/줄지 않게** 만드는 것이 안전

```css
.fixed { flex: 0 0 200px; } /* width보다 의도가 명확 */
```

---

## 9. order(시각 순서)와 접근성

```css
/* 시각상 순서만 바뀜, DOM 순서는 그대로 → 키보드/스크린리더 순서 문제 가능 */
.item-a { order: 2; }
.item-b { order: 1; }
```

- **폼/네비게이션** 등 포커스 순서가 중요한 영역에서 무분별한 `order` 사용은 지양.  
  시맨틱/DOM 순서와 시각 순서를 **가능한 일치**시키는 것이 접근성에 유리.

---

## 10. 스크롤·넘침과의 상호작용

- Flex 아이템이 넘칠 때: `min-width: 0`(수평) or `min-height: 0`(수직) 지정 필요할 때가 많음.  
  일부 브라우저는 기본 min-size가 auto여서 **스크롤이 안 생기는** 오해를 유발.

```css
.scrollable {
  display: flex;
}
.scrollable .col {
  flex: 1 1 0;
  min-width: 0;            /* 중요: 내용이 길어도 축소 허용 */
  overflow: auto;
}
```

---

## 11. 디버깅 체크리스트

1. **축 방향 확인**: `flex-direction`이 의도와 일치하는가?  
2. **여유/부족 공간**: `flex-grow`/`flex-shrink` 기본값(1) 때문에 의도치 않게 늘/줄고 있지 않은가?  
3. **기본 크기 출처**: `flex-basis:auto` → content/width/height가 영향.  
4. **줄바꿈**: `flex-wrap`이 필요한가? 여러 줄 정렬은 `align-content`.  
5. **간격**: 마진 대신 `gap` 사용으로 예외/첫마진 문제 제거.  
6. **접근성**: `order`로 포커스 순서 꼬이지 않는가?  
7. **넘침**: `min-width:0`/`min-height:0`으로 스크롤 정상 동작?  
8. **기본 정렬**: `align-items: stretch`가 원치 않는 높이 확장을 만들고 있지 않은가?

---

## 12. 실전 샘플 모음(독립 실행)

### 12.1 히어로 센터링 + CTA
```html
<section class="hero">
  <div class="hero__inner">
    <h1>Build UI Faster</h1>
    <p>Flexbox로 간단히 정렬하고, gap으로 간격을 관리하세요.</p>
    <div class="actions">
      <button>Get Started</button>
      <button class="ghost">Learn More</button>
    </div>
  </div>
</section>
<style>
.hero {
  min-height: 60vh;
  display: flex; justify-content: center; align-items: center;
  padding: 2rem;
  background: linear-gradient(135deg, #eef2ff, #e0f2fe);
}
.hero__inner { max-width: 60ch; text-align: center; display:flex; flex-direction:column; gap: 1rem; }
.actions { display:flex; justify-content:center; gap:.75rem; }
button { padding:.6rem 1rem; border:1px solid #334155; border-radius:.5rem; background:#0ea5e9; color:#fff; }
button.ghost { background:#fff; color:#0ea5e9; }
</style>
```

### 12.2 상품 카드 그리드(감싸기 + 최소폭)
```html
<ul class="products">
  <li class="card">Card A</li>
  <li class="card">Card B</li>
  <li class="card">Card C</li>
  <li class="card">Card D</li>
</ul>
<style>
.products { list-style:none; margin:0; padding:0; display:flex; flex-wrap:wrap; gap:1rem; }
.card { flex: 1 1 220px; border:1px solid #cbd5e1; border-radius:.75rem; padding:1rem; background:#fff; }
</style>
```

### 12.3 사이드바 + 본문 + 보조패널
```html
<div class="shell">
  <aside class="side">Sidebar</aside>
  <main class="main">Main Content</main>
  <aside class="aux">Aux Panel</aside>
</div>
<style>
.shell { display:flex; min-height:60vh; gap:1rem; }
.side { flex:0 0 240px; border:1px solid #e2e8f0; }
.main { flex:1 1 auto; min-width:0; border:1px solid #e2e8f0; }
.aux  { flex:0 0 320px; border:1px solid #e2e8f0; }
</style>
```

### 12.4 입력 줄(남는 공간 입력이 차지)
```html
<div class="inputrow">
  <input placeholder="Search..." />
  <button>Go</button>
</div>
<style>
.inputrow { display:flex; gap:.5rem; max-width:32rem; }
.inputrow input { flex:1 1 auto; min-width:0; padding:.6rem .8rem; border:1px solid #94a3b8; border-radius:.5rem; }
.inputrow button { flex:0 0 auto; padding:.6rem 1rem; }
</style>
```

### 12.5 균등 버튼 열
```html
<div class="btnline">
  <button>Yes</button><button>No</button><button>Maybe</button>
</div>
<style>
.btnline { display:flex; gap:.5rem; }
.btnline button { flex: 1 1 0; padding:.6rem 0; }
</style>
```

### 12.6 여러 줄 정렬 + align-content
```html
<div class="tags">
  <span>Design</span><span>CSS</span><span>Flexbox</span>
  <span>Accessibility</span><span>Performance</span><span>Responsive</span>
</div>
<style>
.tags {
  display:flex; flex-wrap: wrap; gap:.5rem;
  align-content: space-between; /* 컨테이너 높이가 충분할 때 줄 간 간격 분배 */
  min-height: 8rem; border:1px dashed #94a3b8; padding:.5rem;
}
.tags span { border:1px solid #94a3b8; padding:.25rem .5rem; border-radius:.5rem; }
</style>
```

---

## 13. 고급 주제

### 13.1 `aspect-ratio`와 Flex
이미지/카드를 **비율 고정**하면서 Flex로 정렬:

```css
.thumb {
  aspect-ratio: 16 / 9;
  display: flex; align-items: center; justify-content: center;
  background: #e2e8f0; border-radius: .5rem;
}
```

### 13.2 Nested Flex(중첩)
자식 행마다 수평 정렬, 부모는 세로 스택:

```css
.list { display:flex; flex-direction:column; gap:.75rem; }
.list .row { display:flex; align-items:center; justify-content:space-between; }
```

### 13.3 Safe Area와 모바일 툴바(고정 헤더/푸터)
`position: sticky` + Flex로 안전 여백 고려:

```css
.header {
  position: sticky; top: 0;
  padding-top: env(safe-area-inset-top);
  display: flex; align-items: center; justify-content: space-between;
}
```

---

## 14. 성능·유지보수·팀 규칙 제안

- **간격은 `gap`** 우선 → 마진 규칙을 최소화(일관성↑, 예외↓).
- **유틸리티 클래스화**: `.flex`, `.items-center`, `.justify-between`, `.flex-1`, `.flex-none` 등.  
- **레이아웃 의도 명시**: 고정폭은 `flex:0 0 Xpx`로 표현(폭 고정과 확장 불가를 동시에 명시).
- **접근성**: `order` 남용 금지, 시각/DOM 순서 일치 유지.  
- **디버그 테두리**: 레이아웃 깨질 때 `outline:1px solid`로 박스 시각화.

---

## 15. 요약 표(핵심만 재정리)

| 분류 | 키 속성 | 기억 포인트 |
|---|---|---|
| 컨테이너 | `display:flex`, `flex-direction`, `flex-wrap`, `justify-content`, `align-items`, `align-content`, `gap` | 축·줄바꿈·정렬·간격 |
| 아이템 | `flex-grow`, `flex-shrink`, `flex-basis`, `flex`, `align-self`, `order` | 크기 정책·개별 정렬·순서 |
| 설계 팁 | `gap`, `min-width:0`, `flex:0 0 Xpx`, auto margin | 간격 일관, 넘침 제어, 고정폭 명시 |
| 접근성 | `order` 자제 | DOM·포커스·리더 순서 유지 |

---

## 16. 마무리

- Flexbox는 **1차원 정렬·간격·크기 분배**의 최강 도구입니다.  
- `flex-basis` vs `width`, `gap` vs margin, `grow/shrink` 기본값 등 **작은 디테일**이 결과를 좌우합니다.  
- 본 가이드의 **패턴 레시피**를 팀 공통 유틸로 정리하면 **개발 속도와 일관성**이 크게 향상됩니다.