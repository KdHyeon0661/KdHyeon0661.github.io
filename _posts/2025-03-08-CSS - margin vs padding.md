---
layout: post
title: CSS - margin vs padding
date: 2025-03-08 21:20:23 +0900
category: CSS
---
# margin vs padding

## 0. 한눈 요약

- **padding**: 콘텐츠 **안쪽 여백**. 배경색·그라디언트·이미지가 **포함**됨.
- **margin**: 박스 **바깥 여백**. 배경에 **포함되지 않음**. **마진 겹침**이 발생할 수 있음.
- 레이아웃 간격은 **gap(Flex/Grid)**가 더 안전한 선택인 경우가 많음.
- `box-sizing: border-box`일 때 **padding/border는 요소 너비에 포함**, margin은 **항상 제외**.

---

## 1. 박스 모델 재정리

```
[ margin ]
   [ border ]
     [ padding ]
       [ content ]
```

- `padding`: 콘텐츠와 `border` 사이
- `margin`: 요소 외부, 이웃 요소와의 간격

**수식(수평방향, 스크롤바/기타 제외)**

- **content-box**(기본):
  $$
  \text{rendered width} = \text{width} + \text{padding-left} + \text{padding-right} + \text{border-left} + \text{border-right}
  $$
- **border-box**:
  $$
  \text{content width} = \text{width} - (\text{padding-left} + \text{padding-right} + \text{border-left} + \text{border-right})
  $$

전역 권장:
```css
/* 예측 가능한 너비 계산 */
*,
*::before,
*::after { box-sizing: border-box; }
```

---

## 2. margin — 바깥 여백

특징
- 배경에 포함되지 않음(완전히 투명).
- **이웃 블록 요소와 간격** 확보.
- **마진 겹침(collapse)** 가능(아래 4장).
- Flex/Grid 컨텍스트에서는 **아이템 간 간격에 margin보다 gap이 우선 고려**.

기본 예시
```css
.box { margin: 20px; }
```
```html
<div class="box">박스 1</div>
<div class="box">박스 2</div>
```

---

## 3. padding — 안쪽 여백

특징
- 배경색/배경이미지/그라디언트가 **패딩 영역까지** 칠해짐.
- 콘텐츠 가독성 확보(버튼, 카드, 알림).

기본 예시
```css
.box {
  padding: 20px;
  background: lightblue;
  border: 1px solid #9cc;
}
```
```html
<div class="box">텍스트가 테두리에서 20px 떨어져 있음</div>
```

---

## 4. margin collapse(마진 겹침) — 반드시 알아야 할 규칙

**상하 방향 블록 마진**끼리는 종종 **겹쳐서 하나로** 계산됩니다. 대표 규칙:

1) **형제 블록 간 상하 마진**: `20px` + `20px` → **최댓값 20px**  
2) **부모와 첫/마지막 자식의 상하 마진**: 부모의 상/하 마진과 자식의 상/하 마진이 **겹칠 수 있음**  
3) **빈 블록 요소의 상하 마진**: 안에 `border/padding/inline 콘텐츠`가 없으면 상·하 마진이 겹침  
4) **Flex/Grid 아이템, absolutely positioned, floating**에서는 **겹치지 않음**

예시
```css
p { margin: 20px 0; }
```
```html
<p>문단 1</p>
<p>문단 2</p>
```
→ 두 문단 사이 간격은 **40px가 아니라 20px**.

### 겹침 방지(택 1)
- 부모에 **padding**이나 **border** 추가
- 부모에 **`overflow: auto|hidden`** (BFC 생성)
- **`display: flow-root`**(새 BFC)
- 상하 간격을 **gap(Flex/Grid)**으로 대체

```css
.article { 
  /* 겹침 방지 */
  padding-top: 1px; /* 시각 영향 없게 0→1px 미세 패딩 */
  padding-top: 0;   /* 실제론 0이지만 개념 예시 */
  overflow: auto;   /* 또는 display: flow-root; */
}
```

---

## 5. 논리 속성으로 쓰기(다국어/세로쓰기 대응)

글 방향 혹은 쓰기 모드가 바뀌어도 유지보수 쉬운 **논리 속성**:

- `margin-top/bottom/left/right` → **`margin-block-start/end`, `margin-inline-start/end`**
- `padding-...` → **`padding-block-*`, `padding-inline-*`**

예시(오른쪽에서 왼쪽, 세로쓰기까지 대응):
```css
.card {
  padding-block: 1rem;     /* top/bottom */
  padding-inline: 1.25rem; /* left/right in LTR, but auto in RTL/vertical */
  margin-block-end: 1rem;
}
```

---

## 6. 퍼센트 padding의 기준

`padding`의 `%`는 **항상 컨테이너의 **가로 너비** 기준**입니다(수직 패딩도 width 기준).

```css
.ratio-16-9 {
  width: 100%;
  /* 9/16 = 56.25%  → 비율 박스 */
  padding-top: 56.25%;
  position: relative;
}
.ratio-16-9 > iframe {
  position: absolute; inset: 0; width: 100%; height: 100%;
}
```

---

## 7. Flex/Grid에서의 간격 — margin vs gap

**권장:** 아이템 간 간격은 **`gap`** 사용.  
장점: 마진 겹침 없음, 부모/첫·마지막 아이템 보정 불필요, 읽기 쉬움.

```css
.list {
  display: flex; /* grid도 동일 */
  gap: 12px;
}
```

마진을 사용할 때는 **양 끝 여백이 비대칭**될 수 있으므로 부모에 `padding`을 더해 맞추는 패턴이 필요합니다.

---

## 8. scroll-margin / scroll-padding — 스크롤 위치 제어

앵커 이동/프래그먼트 내비게이션 시 **고정 헤더에 가리는 현상** 해결:

```css
:root { scroll-padding-top: 72px; }     /* 전역 헤더 높이 */
#section-title { scroll-margin-top: 72px; } /* 특정 타깃만 보정 */
```

---

## 9. 안전 영역(iOS Notch) — env()와 padding

모바일 브라우저 상단 노치/하단 홈 인디케이터를 피하려면 **padding + safe area**:

```css
.header {
  padding-top: calc(16px + env(safe-area-inset-top));
}
.footer {
  padding-bottom: calc(16px + env(safe-area-inset-bottom));
}
```

---

## 10. 배경 페인팅 영역과 clipping

- **padding**은 배경에 포함 ⇒ `background-clip`으로 경계 제어 가능:
```css
.card {
  background: linear-gradient(180deg, #e8f2ff, #fff);
  border: 1px solid #cfe0ff;
  padding: 16px;
  background-clip: padding-box; /* border 바깥으로 번짐 방지 */
}
```
- **margin**은 배경과 무관(항상 투명).

---

## 11. `auto` margin으로 정렬/분배

블록에서 가운데 정렬:
```css
.container { width: min(90vw, 60ch); margin-inline: auto; }
```

Flex에서 남는 공간 밀어내기:
```css
.header { display: flex; align-items: center; }
.header .spacer { margin-inline-start: auto; } /* 오른쪽으로 붙이기 */
```

---

## 12. 실전 비교 & 리팩터링

### 12.1 버튼 내부 여백은 padding
```css
.btn { 
  padding: .5rem 1rem;
  border-radius: .5rem;
}
```

### 12.2 카드 묶음 간 간격은 gap(대안: margin)
```css
.cards {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(16rem, 1fr));
  gap: 1rem; /* 권장 */
}
```

### 12.3 문단 간격 — margin vs padding
문단 사이 간격은 **문단의 바깥 관계**이므로 margin:
```css
.prose p { margin-block: 1em; }
```
마진 겹침 이슈가 신경 쓰이면 부모에:
```css
.prose { display: flow-root; } /* 또는 overflow: auto; padding-top: ... */
```

---

## 13. 시각 예시(간단 데모)

```html
<style>
  .demo { display: grid; grid-template-columns: 1fr 1fr; gap: 16px; }
  .box1 { background: #ddd; margin: 20px; }
  .box2 { background: #add8e6; padding: 20px; }
  .wrap  { border: 1px dashed #aaa; padding: 8px; }
</style>

<div class="demo">
  <div class="wrap">
    <div class="box1">Margin 사용(박스 외부에 20px 공간)</div>
    <div>이웃 요소</div>
  </div>

  <div class="wrap">
    <div class="box2">Padding 사용(콘텐츠와 테두리 사이 20px)</div>
    <div>이웃 요소</div>
  </div>
</div>
```

---

## 14. 마진 겹침 — 실제 문제와 해결 패턴

문제 예시
```css
.article p { margin-block: 20px; }
```
```html
<article class="article">
  <p>문단 1</p>
  <p>문단 2</p>
</article>
```
→ 문단 사이 간격은 40px가 아니라 **20px**.

해결 패턴
```css
.article { 
  /* 겹침 방지: 택1 */
  padding-top: 1px; padding-top: 0; /* 개념 예시 */
  border-top: 1px solid transparent;
  overflow: auto;          /* BFC */
  /* 또는 */
  display: flow-root;      /* BFC */
}
```

---

## 15. 흔한 함정과 체크리스트

| 주제 | 실수 | 권장 |
|---|---|---|
| 버튼 내부 여백 | `margin`으로 안쪽 공간 확보 | **`padding`** 사용 |
| 카드 간격 | 각 카드에 `margin-bottom` | 부모 컨테이너 **`gap`**(flex/grid) |
| 문단 위아래 | margin 겹침 몰라서 불규칙 | `flow-root`/`overflow:auto`/부모 `padding` |
| 반응형 여백 | 고정 px만 사용 | `clamp()`/상대 단위/컨테이너 쿼리 |
| 다국어 대응 | left/right만 사용 | **논리 속성** `inline/block` 축 사용 |
| 정렬 | 임의 padding으로 “가짜” 가운데 | **`margin-inline: auto`**, flex 정렬 |

---

## 16. 반응형·유동 여백 설계

`clamp(min, preferred, max)`로 적응형 패딩:
```css
.section {
  padding-inline: clamp(1rem, 5vw, 3rem);
  padding-block: clamp(1.25rem, 6vw, 4rem);
}
```

컨테이너 크기에 따라 여백 조정(컨테이너 쿼리):
```css
.article { container-type: inline-size; container-name: prose; }
@container prose (width < 600px) {
  .article { padding-inline: 1rem; }
}
```

---

## 17. 접근성/UX와 여백

- 텍스트 블록은 **line-height** + **문단 간 margin**으로 가독성 유지.
- 터치 대상은 **padding**으로 히트영역 확대(44px 안팎 권장).
- 포커스 outline이 보이도록 **padding/outline-offset** 조정.

```css
.button {
  padding: .625rem 1rem;
  outline-offset: 2px;
}
:focus-visible { outline: 3px solid #0ea5e9; }
```

---

## 18. 실전 레시피 모음

### 18.1 카드 레이아웃(권장안: gap)
```css
.cards { display: grid; grid-template-columns: repeat(auto-fit, minmax(18rem, 1fr)); gap: 1.25rem; }
.card  { padding: 1rem; border: 1px solid #e5e7eb; border-radius: .75rem; background: #fff; }
```

### 18.2 네비게이션 바
```css
.nav   { display: flex; align-items: center; gap: .75rem; padding: .5rem 1rem; }
.nav a { padding: .375rem .5rem; border-radius: .375rem; }
```

### 18.3 기사 본문
```css
.prose { 
  max-inline-size: 70ch; margin-inline: auto;
  padding-inline: 1rem; padding-block: 2rem;
}
.prose p { margin-block: 1em; }
```

---

## 19. 종합 비교 표(확장판)

| 항목 | margin | padding |
|---|---|---|
| 위치 | 요소 바깥쪽 | 요소 안쪽 |
| 배경 포함 | 포함 안 됨 | 포함됨 |
| 마진 겹침 | **가능**(블록 상하) | 불가능 |
| box-sizing 영향 | **X** | **O**(border-box면 너비에 포함) |
| 간격(레이아웃) | 형제 간격, 외부 관계 | 콘텐츠-테두리 관계 |
| 권장 대안 | Flex/Grid에선 **gap** | – |
| 정렬/분배 | `auto` margin으로 공간 밀어내기 | 히트영역/가독성 확보 |
| 퍼센트 기준 | 의미 없음 | **컨테이너 폭 기준** |
| 논리 속성 | `margin-block/inline-*` | `padding-block/inline-*` |

---

## 20. 최종 결론

- **의도**가 “요소 **밖**과의 거리”면 **margin**, “요소 **안**의 여유”면 **padding**.  
- **리스트/그리드/행 간격**은 **gap**이 안전·간결.  
- 마진 겹침이 의심되면 **BFC(flow-root/overflow)**, 부모 padding/border로 **즉시 차단**.  
- 다국어/세로쓰기/RTL을 고려해 **논리 속성**을 습관화.  
- 반응형에선 `clamp()`·컨테이너 쿼리로 **스케일하는 여백**을 설계.