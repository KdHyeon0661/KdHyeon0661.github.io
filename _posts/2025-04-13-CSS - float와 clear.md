---
layout: post
title: CSS - float와 clear
date: 2025-04-06 23:20:23 +0900
category: CSS
---
# float와 clear 속성 — 개념, 함정, 해결책(clearfix/flow-root), 그리고 현대 대체안까지

## 1) `float`란 무엇인가 — “흐름에서 떠서, 텍스트를 감싼다”

```css
img.thumb { float: left; margin: 0 12px 8px 0; }
```

```html
<p>
  <img class="thumb" src="image.jpg" alt="..." />
  텍스트가 이미지의 오른쪽과 아래로 흘러가며 감쌉니다. 이미지는 왼쪽에 ‘띄워진’ 상태로 배치됩니다.
</p>
```

- **핵심 효과**: 요소를 정상 문서 흐름(normal flow)의 블록 레벨 배치에서 빼내어 좌/우로 “띄우고”, **인접 인라인 콘텐츠(텍스트/인라인 요소)가 그 주변을 흐르게** 합니다.
- `float` 값:
  - `left` / `right`: 좌/우로 띄움
  - `none`(기본): 띄우지 않음
  - `inline-start` / `inline-end`(일부 환경): 글방향 고려

---

## 2) float의 동작 원리(한 번에 이해)

### 2.1 라인 박스와의 상호작용
- 인라인 포맷팅 컨텍스트(IFC)에서 텍스트는 **라인 박스(line box)**를 형성합니다.
- 떠 있는 상자(float box)를 피하기 위해 라인 박스가 **수평 폭을 줄이며** 텍스트를 재흐름(reflow)시킵니다.  
  → 결과: 텍스트가 마치 이미지를 둘러싸듯 흐름.

### 2.2 박스 포맷팅 컨텍스트(BFC)와 float
- **float 요소 자체는 새로운 BFC를 생성**합니다.  
  - BFC 내부의 마진은 바깥과 **마진 겹침이 일어나지 않음**(레이아웃 안정성↑).
  - float의 내부는 스스로 레이아웃을 “자급자족”합니다(외부 float의 지배 감소).

### 2.3 부모 높이 상실(고전 함정)
- float는 **정상 흐름(out-of-flow)** 요소로 취급 → 부모는 자식 float의 높이를 **컨텐츠 높이에 포함시키지 않음**.  
  → 배경/테두리가 “꺼져 보이는” 현상이 발생(초안에서 지적한 바로 그 문제).

---

## 3) 예전 레이아웃에서의 float — 왜 더 이상 권장되지 않나

```html
<div class="container">
  <aside class="sidebar">사이드바</aside>
  <main class="content">본문</main>
</div>
```

```css
.sidebar { float: left; width: 30%; }
.content { float: right; width: 70%; }
.container { background: #f6f6f6; } /* 높이 붕괴 위험 */
```

- **단점**:
  - 부모 높이 붕괴 → clearfix/overflow 해킹 필요
  - 정렬/간격/역방향 배치가 불편
  - 반응형 대응 시 예외처리 다수  
- **대안**: Flex/Grid는 이 모든 문제를 본질적으로 해결(§10 참고).

---

## 4) `clear` — float 아래에서 새 줄 시작

```css
.clear-both { clear: both; } /* left/right 어느 쪽 float도 회피 */
```

```html
<div class="left" style="float:left;width:120px;height:60px;background:#cce;">L</div>
<p>이 문단은 float을 피해 오른쪽으로 감쌉니다.</p>
<div class="clear-both"></div>
<p>여기부터는 float 아래 새 줄에서 시작.</p>
```

- 값: `left | right | both | none`
- **주의**: `clear`는 해당 요소를 **아래로 밀어** float의 수평 간섭을 회피시킬 뿐, 부모 높이 붕괴 자체를 해결하지는 않습니다(부모가 여전히 float만 자식으로 갖고 있으면 높이는 0).

---

## 5) clearfix — “부모가 float 자식을 감싸도록” 만드는 고전 처방

### 5.1 가장 표준적인 clearfix(의사요소)
```css
/* 재사용 가능한 클래스 */
.clearfix::after {
  content: "";
  display: block;
  clear: both;
}
```

```html
<div class="wrap clearfix">
  <img style="float:left;width:80px;height:80px;background:#def" alt="">
  <p>텍스트가 옆으로 감싸이고, 부모 .wrap은 의사요소로 인해 높이가 생깁니다.</p>
</div>
```

- 원리: 보이지 않는 블록을 **마지막에 삽입**하여 float 아래로 내려가게 하고, 그 블록이 부모 높이에 기여 → 부모가 내용물을 **정상적으로 감쌈**.

### 5.2 overflow 해법(간단하지만 부작용 주의)
```css
.wrapper { overflow: hidden; } /* 또는 overflow:auto */
```
- 장점: 간단, float를 포함하는 **새로운 BFC**를 만들어 부모 높이 복구.
- 단점: 원치 않는 스크롤/클리핑 발생 가능(특히 도형/그림자/툴팁이 넘칠 때).

### 5.3 최신 해법: `display: flow-root`(권장)
```css
.flow-root { display: flow-root; }
```

```html
<div class="flow-root" style="background:#f5f5f5;padding:8px;">
  <img style="float:right;width:80px;height:80px;background:#feb" alt="">
  <p>부모가 스스로 BFC(정확히는 새로운 블록 포맷팅 컨텍스트의 루트)를 형성하여 float를 자연스럽게 포함합니다.</p>
</div>
```

- **의도 표현력이 가장 높음**: “이 컨테이너는 자체 흐름의 루트”
- 부작용 적고 명시적이라 **최신 CSS에서 가장 추천**되는 방법

---

## 6) float가 레이어/페인팅에 미치는 영향(요약)

- 일반적으로 **float는 비포지셔닝(non-positioned) 콘텐츠보다 먼저 페인트**되며, 인라인 콘텐츠는 그 주변을 회피하며 흐릅니다.
- `z-index`는 포지셔닝 컨텍스트가 필요하지만, float 자체의 페인팅 순서가 겹침 결과에 영향을 줄 수 있습니다(복잡한 중첩에서는 포지셔닝 사용 권장).

---

## 7) 실전 패턴

### 7.1 본문 이미지 감싸기(표준)
```html
<article class="prose">
  <img class="float-img" src="..." alt="..." width="320" height="200">
  <p>기사 본문이 이미지 주변을 감쌉니다. 캡션/크레딧이 있다면 별도 블록으로 처리하는 편이 좋습니다.</p>
</article>
```

```css
.prose { display: flow-root; } /* 최신 권장 */
.float-img { float: left; margin: 0 12px 8px 0; shape-margin: 8px; }
```

### 7.2 카드/요약박스의 좌측 썸네일
```html
<div class="media flow-root">
  <img class="thumb" src="..." alt="">
  <h3>제목</h3>
  <p>요약 텍스트가 썸네일을 감싸며 흐릅니다.</p>
</div>
```

```css
.media { padding: 12px; border: 1px solid #e5e7eb; border-radius: 8px; }
.media .thumb { float: left; width: 96px; height: 96px; margin: 0 12px 4px 0; object-fit: cover; }
```

### 7.3 구형 코드 유지: clearfix 유틸
```css
.clearfix::after { content:""; display:block; clear:both; }
.float-left { float:left; } .float-right { float:right; }
```

---

## 8) 문제 해결 시나리오별 처방전

| 증상 | 원인 | 처방 |
|---|---|---|
| 부모 배경·테두리 사라짐 | 부모가 float 자식을 **높이로 인식 안 함** | `.clearfix`, `overflow:auto/hidden`, `display: flow-root` |
| 예상치 못한 스크롤 | overflow로 BFC 생성했으나 오버플로우도 클리핑 | `flow-root` 또는 컨텐츠 구조 재검토 |
| 텍스트가 이미지에 너무 달라붙음 | 마진/shape-margin 부재 | `margin-right`/`shape-margin`으로 여백 확보 |
| 레이아웃에 사용하다가 반응형에서 붕괴 | float는 레이아웃 제어가 취약 | Flex/Grid로 마이그레이션 |
| 아래 요소가 float 옆으로 올라옴 | float 영향 범위 회피 실패 | `clear: both`(또는 적절한 방향) 적용 |

---

## 9) `clear`의 미세 규칙(정확히 이해)

```css
.break { clear: both; }
```

- `clear`는 해당 요소의 **마진 경계**를 float의 **가장자리 아래쪽**까지 밀어내 **새 라인 박스 시작**을 강제합니다.
- `clear: left|right`를 선택적으로 써서 **한쪽 float만 회피**할 수도 있습니다.

---

## 10) 현대 대안: Flex/Grid로 대체

### 10.1 사이드바/본문(구 float → 신 Flex)
```css
/* 예전: .sidebar{float:left;width:30%} .content{float:right;width:70%} */
.layout { display: flex; gap: 1rem; }
.sidebar { flex: 0 0 300px; }
.content { flex: 1 1 auto; min-width: 0; }
```

### 10.2 카드 갤러리(구 float-wrap → 신 Flex wrap)
```css
.gallery { display:flex; flex-wrap:wrap; gap:1rem; }
.card { flex: 1 1 220px; }
```

### 10.3 2차원 격자(Grid)
```css
.grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(220px, 1fr));
  gap: 1rem;
}
```

- **장점**: 정렬/간격/반응형/역순/정렬라인이 **표현력↑ & 유지보수성↑**  
- **권장**: 새 프로젝트/리팩토링 시 **float 제거**를 목표로.

---

## 11) float로 “모양대로 감싸기”: `shape-outside`(텍스트 래핑 고급)

```html
<div class="shape"></div>
<p>텍스트가 원형 이미지의 바깥을 부드럽게 따라 흐르도록 만들 수 있습니다.</p>
```

```css
.shape {
  float: left;
  width: 160px; height: 160px;
  background: url(avatar.jpg) center/cover no-repeat;
  shape-outside: circle(50% at 50% 50%);
  shape-margin: 10px; /* 텍스트와 모양 사이 여백 */
}
```

- `shape-outside`: **텍스트가 회피해야 하는 윤곽** 정의(원/타원/경로/이미지 알파 등)
- 디자인 기사/매거진 풍의 **모던 타이포그래피**를 float로 구현할 때 유용
- **주의**: 브라우저 지원/성능 고려

---

## 12) 종합 예제(복붙 가능)

```html
<!doctype html>
<meta charset="utf-8">
<title>float / clear / clearfix / flow-root 데모</title>
<style>
  .section { margin: 24px auto; max-width: 720px; padding: 16px; border:1px solid #e5e7eb; border-radius: 12px; }
  .title { font: 700 1.25rem/1.3 system-ui, -apple-system, "Segoe UI", Roboto, "Noto Sans KR", sans-serif; margin: 0 0 12px; }

  /* 1) 기본: float + 텍스트 감싸기 */
  .prose { display: flow-root; } /* 최신 권장 */
  .prose img { float: left; width: 160px; height: 120px; object-fit: cover; margin: 0 12px 8px 0; border-radius: 8px; }

  /* 2) clearfix 유틸 */
  .clearfix::after { content:""; display:block; clear:both; }

  /* 3) shape-outside로 둥근 감싸기 */
  .avatar { float: right; width: 160px; height: 160px; background: url(https://picsum.photos/240) center/cover no-repeat; shape-outside: circle(50%); shape-margin: 10px; border-radius: 50%; margin: 0 0 12px 12px; }

  /* 4) 레이아웃에 float 쓰지 말고(참고용) flex/grid 권장 */
  .flex-bar { display:flex; justify-content:space-between; align-items:center; padding:8px 12px; background:#f8fafc; border-radius: 8px; }
</style>

<section class="section prose">
  <h2 class="title">기본 float 감싸기</h2>
  <img src="https://picsum.photos/320/240" alt="">
  <p>이 문단은 좌측의 이미지를 피해 흐릅니다. 부모에 <code>display: flow-root;</code>가 있어 별도 clearfix 없이 부모가 높이를 정상 인식합니다.</p>
  <p>여백은 이미지의 마진으로 확보했습니다.</p>
</section>

<section class="section clearfix">
  <h2 class="title">clearfix로 부모 높이 복구</h2>
  <div style="float:left;width:120px;height:80px;background:#e0f2fe;border-radius:8px;margin:0 12px 8px 0;"></div>
  <p>부모에 <code>.clearfix</code>를 붙여 float 자식을 감싸도록 했습니다. 고전 코드 유지/수정 시 유용합니다.</p>
</section>

<section class="section prose">
  <h2 class="title">shape-outside로 생동감 있는 래핑</h2>
  <div class="avatar" aria-hidden="true"></div>
  <p>float에 <code>shape-outside</code>를 사용하면 텍스트가 원형/경로를 따라 흐릅니다. 잡지 레이아웃 같은 고급 타이포그래피를 구현할 수 있습니다.</p>
</section>

<section class="section">
  <h2 class="title">대체: Flex를 쓰자</h2>
  <div class="flex-bar">
    <div>로고</div>
    <nav>메뉴</nav>
    <div>프로필</div>
  </div>
  <p>정렬/간격/반응형을 자주 다루는 헤더/바는 <strong>float 대신 Flex</strong>가 정석입니다.</p>
</section>
```

---

## 13) 디버깅/체크리스트

1. **부모 높이 붕괴?** → `flow-root` 또는 `.clearfix` 적용.  
2. **원치 않는 스크롤/클리핑?** → `overflow`로 해결했다면 `flow-root`로 교체.  
3. **텍스트가 너무 붙음?** → float 요소에 `margin` 또는 `shape-margin`.  
4. **요소가 옆으로 기어올라감?** → 적절한 요소에 `clear: both|left|right`.  
5. **레이아웃 전체에 float 사용?** → Flex/Grid로 리팩토링 계획 수립.  

---

## 14) 요약 표

| 주제 | 핵심 |
|---|---|
| float의 본질 | 텍스트/인라인 흐름이 **회피**하며 주변을 감쌈 |
| 고전 문제 | 부모 높이 붕괴, 정렬/반응형 어려움 |
| 해결 | `.clearfix`(의사요소), `overflow:auto/hidden`(부작용 가능), **`display: flow-root`(권장)** |
| 현대 대체 | 레이아웃은 **Flex/Grid**, 텍스트 감싸기는 필요 시 **float + shape-outside** |
| 보조 | `clear`로 float 아래에서 새 줄 시작 |

---

## 15) 결론

- float는 원래 **텍스트 래핑**용. 레이아웃에는 **이제 쓰지 않는다**가 표준 실무 감각입니다.  
- **부모 높이 문제**는 `flow-root`로 명시적으로 해결하는 시대.  
- 복잡한 배치/정렬/반응형은 **Flex/Grid**를 기본값으로 선택하세요.