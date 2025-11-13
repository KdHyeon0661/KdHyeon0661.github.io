---
layout: post
title: CSS - display 속성
date: 2025-04-01 19:20:23 +0900
category: CSS
---
# display 속성: block, inline, inline-block

## 0. 한눈 개요

- **`display`는 요소가 생성하는 “박스 트리”와 그 박스들이 **어떤 문맥에서** 배치되는지**를 정의**합니다.
- 핵심 축:
  - **Outside**(바깥 참여 방식): `block` | `inline` | `run-in(폐기됨)` …
  - **Inside**(안쪽 포맷팅): `flow` | `flow-root` | `flex` | `grid` | `ruby` | `table` …
  - 단일 키워드로 축약 가능: `display: block;` ≈ `display: block flow;`
- **Formatting Context**:
  - **BFC**(Block Formatting Context): 블록 레벨 레이아웃; 마진 겹침 규칙 존재
  - **IFC**(Inline Formatting Context): 인라인 레이아웃; 라인 박스·베이스라인·화이트스페이스 규칙
- `inline-block`은 **Outside = inline**, **Inside = flow-root**에 상응(독립 박스 + 인라인처럼 줄에 놓임).

---

## 1. `display: block` — 블록 포맷팅 컨텍스트의 기본

### 1.1 요약
- **새 줄에서 시작**, 가능한 가로 공간을 **가득 차지**(width가 auto면 컨테이너의 content box만큼).
- 블록 정류: `<div>`, `<p>`, `<h1>…<h6>`, `<section>` …

```html
<div style="display:block;background:#cfe8ff">Block 요소</div>
<span style="display:block;background:#c8f7c8">Span도 block이 되면 줄바꿈</span>
```

### 1.2 레이아웃 규칙(핵심)
- Block Formatting Context(IFC가 아닌 **BFC**)에 참여.
- 수직 방향으로 **상하 margin이 겹침**(margin collapse): 인접 형제·부모/첫·마지막 자식 간.
- `width/height/margin/padding/border` 모두 효과적.
- `vertical-align`은 **인라인 컨텍스트**에서만 의미 → block에는 영향 없음.

### 1.3 상하 마진 겹침(요약 수식)
- 두 블록의 상하 마진 \(m_1, m_2\)가 만날 때 실제 간격:
  $$
  \text{gap} = \max(m_1, m_2)
  $$
  (음수 마진 포함 시 합성 규칙이 더 복잡해짐)

### 1.4 실전 팁
- 문단 간격은 `.prose p{ margin-block:1em; }`처럼 **block 마진**으로.
- 겹침을 피하고 싶으면 **BFC 생성**: `overflow:auto|hidden` 또는 `display: flow-root;` 적용.

```css
.article { display: flow-root; } /* BFC → 자식 p의 상단 마진이 부모와 겹치지 않음 */
```

---

## 2. `display: inline` — 인라인 포맷팅 컨텍스트(IFC)

### 2.1 요약
- **줄 안(line box)** 에서 다른 인라인들과 **흐름대로 배치**.
- 대표: `<span>`, `<a>`, `<strong>`, `<em>`, `<code>`, 텍스트 노드 등.

```html
<span style="display:inline;background:#ffe680">Inline 요소</span>
<span style="display:inline;background:#ffc299">옆에 붙습니다</span>
```

### 2.2 레이아웃 규칙(핵심)
- **IFC**에서 `line-height`, 글꼴 메트릭, 공백/줄바꿈 규칙에 따라 줄 상자(line box)가 생성.
- `width/height`는 **직접 지정해도 무시**(컨텐츠 크기에 의해 결정).
- `padding/border`는 적용되나, **수직 마진(top/bottom)**은 라인 박스 배치에 **영향이 거의 없음**(겹치거나 무시됨처럼 보임).
- `vertical-align`이 **중요**: baseline, middle, top/bottom, `-2px` 등으로 라인 내 정렬을 제어.

```html
<style>
  .tag, .pill, .badge { display:inline; padding: .25em .5em; border-radius: .5rem; }
  .tag   { background:#eef2ff; vertical-align:baseline; }
  .pill  { background:#dcfce7; vertical-align:middle; }
  .badge { background:#fee2e2; vertical-align:-2px; }
</style>
<p>
  기본텍스트 <span class="tag">baseline</span>
  <span class="pill">middle</span>
  <span class="badge">-2px</span> 기준 차이
</p>
```

### 2.3 replaced 요소 예외
- `img`, `video`, `canvas` 등 **replaced elements**는 기본 `display:inline`이지만,
  **width/height 지정이 유효**(컨텐츠 자체가 치수 가짐).
  즉, 인라인이라도 사이즈를 직접 제어 가능.

```html
<img src="..." alt="thumb" style="width:120px;height:80px;vertical-align:middle">
```

---

## 3. `display: inline-block` — 줄에 놓이는 **미니 독립 박스**

### 3.1 요약
- 줄 안에 놓이되(**Outside=inline**), 내부는 작은 블록처럼 **width/height/margin/padding** 모두 적용.
- 내부는 사실상 **flow-root**(자체 BFC) → 자식 블록 마진이 밖으로 겹치지 않음.

```html
<span style="display:inline-block;width:120px;height:48px;background:#ffc9de">inline-block</span>
<span style="display:inline-block;width:140px;height:48px;background:#ffb3c1">또 다른 박스</span>
```

### 3.2 베이스라인/문자 간 간극
- `inline-block`은 기본적으로 **박스의 베이스라인**을 라인 박스 정렬 기준으로 삼음.
  다른 인라인들과 높이 차이가 나면 **baseline이 어긋난 듯** 보일 수 있음 → `vertical-align: middle|top|bottom` 등으로 조정.
- **인접 inline-block 사이 공백 문자**(HTML 소스의 줄바꿈/스페이스)가 **시각적 간격**을 만듦.

```html
<style>
  .ib { display:inline-block; width:120px; height:40px; background:#e0f2fe; vertical-align:top; }
</style>

<div>
  <span class="ib">A</span><!-- 공백 제거 트릭: 주석 -->
  <span class="ib">B</span>
</div>
```

공백 제거 대안:
- 부모에 `font-size:0;` 후 자식에 다시 폰트 지정
- 태그를 붙여쓰기
- CSS Grid/Flex로 전환(권장)

---

## 4. 세 값 비교 — 개념/속성/문맥

| 속성 | 줄바꿈 | 가로 점유 | width/height | margin/padding | 포맷팅 컨텍스트 | 베이스라인 정렬 |
|---|---|---|---|---|---|---|
| `block` | O | 기본 전체 | O | O | **BFC** | N/A |
| `inline` | X | 내용만 | (무시) | padding/border만 체감, 수직 margin 영향 미미 | **IFC** | `vertical-align`으로 제어 |
| `inline-block` | X | 내용만(줄 안) | O | O | **자체 BFC(flow-root)** | `vertical-align` 필요시 조정 |

---

## 5. 두-값 디스플레이 구문(현대 문법)

- 문법: `display: <outside> <inside>;`
  - `<outside>`: `block` | `inline`
  - `<inside>` : `flow` | `flow-root` | `flex` | `grid` | `table` | `ruby`

자주 쓰는 조합:
```css
display: block flow;      /* == display:block */
display: inline flow;     /* == display:inline */
display: inline flow-root;/* inline formatting + 독립 내부 BFC → inline-block과 상응 */
display: block flow-root; /* 새로운 BFC(마진 겹침 차단) */
display: block flex;      /* == display:flex */
display: block grid;      /* == display:grid */
```

> 참고: `inline-block`은 개념적으로 `inline flow-root`와 매우 가깝습니다.

---

## 6. 실무에서 자주 묻는 질문(FAQ) — 핵심 이슈

### 6.1 왜 인라인에 `width/height`가 안 먹나요?
- IFC에서는 **컨텐츠의 글꼴 메트릭**이 줄 높이를 결정. 크기 지정은 무시됨(패딩/보더는 시각적 영역만 확장).
  **해결**: `display:inline-block`/`inline-flex`로 바꿔 박스화.

### 6.2 인라인의 상하 마진이 안 먹는 것처럼 보여요
- 수직 마진은 라인 박스 배치에 큰 영향이 없음.
  **해결**: 상하 간격은 `line-height`, `padding`, 또는 래퍼를 블록으로 분리.

### 6.3 inline-block 사이의 간격(화이트스페이스)이 싫어요
- 소스의 공백이 **텍스트 노드**로 간주되어 간격이 생김.
  **해결**: 주석/붙여쓰기/`font-size:0` 트릭, 또는 Grid/Flex 사용.

### 6.4 블록 자식의 상단 마진이 부모 밖으로 튀어나옵니다
- BFC의 **마진 겹침** 때문.
  **해결**: 부모에 `overflow:auto|hidden` 또는 `display: flow-root;` 로 BFC 경계 생성.

---

## 7. `display: contents`/`flow-root`/`list-item` — 유용한 보너스

### 7.1 `display: contents`
- **자기 박스를 없애고** 자식만 **부모의 레이아웃**에 참여시킴(“래퍼를 없애는” 효과).
- 스타일 상속·선택자 스코프는 유지하지만, **접근성 트리/스크린리더 영향** 있을 수 있음(주의).

```html
<ul>
  <li>그룹 1</li>
  <div class="contents" style="display:contents">
    <li>그룹 2-1</li>
    <li>그룹 2-2</li>
  </div>
</ul>
<!-- 렌더링상 div 래퍼가 사라져 li들이 같은 계층처럼 작동 -->
```

### 7.2 `display: flow-root`
- **새 BFC 생성**. 자식의 상단 마진 겹침 방지, float 컨테이너 라핑 등 레이아웃 안정에 유용.

```css
.card { display: flow-root; }
```

### 7.3 `display: list-item`
- 블록처럼 배치 + **리스트 마커**(`::marker`)를 가짐.
  커스텀 컴포넌트에 목록 스타일을 입힐 때 유용.

```css
.feature { display:list-item; margin-inline-start:1ch; }
.feature::marker { content:"✔ "; color:#16a34a; }
```

---

## 8. 실전 패턴 모음

### 8.1 버튼/배지: 인라인 → inline-block으로 히트영역 확보
```html
<style>
  .btn {
    display:inline-block; /* width/height/padding 유효 */
    padding:.5em 1em;
    border-radius:.5rem; border:1px solid #e5e7eb;
    background:#111; color:#fff; text-decoration:none;
    vertical-align:middle;
  }
  .btn + .btn { margin-inline-start:.5rem; }
</style>

<a class="btn" href="#">기본</a>
<a class="btn" href="#">다음</a>
```

### 8.2 인라인 아이콘 정렬: baseline vs middle
```html
<style>
  .icon { width:1em; height:1em; vertical-align:-0.125em; } /* 살짝 내려 시각적 중앙 */
</style>
텍스트와 <img class="icon" src="check.svg" alt=""> 아이콘 정렬
```

### 8.3 카드 그리드: inline-block 대안으로 Flex/Grid
```html
<style>
  .grid { display:grid; grid-template-columns:repeat(auto-fill,minmax(16rem,1fr)); gap:1rem; }
  .card { padding:1rem; border:1px solid #e5e7eb; border-radius:.75rem; background:#fff; }
</style>
<section class="grid">
  <article class="card">A</article>
  <article class="card">B</article>
  <article class="card">C</article>
</section>
```
- **권장**: inline-block 대신 Grid/Flex를 사용하면 **공백 간극 문제**/정렬/분배가 한 번에 해결.

### 8.4 문단 상단 마진 “겹침” 제거: flow-root
```html
<style>
  .prose { display:flow-root; padding:1rem; background:#f8fafc; }
  .prose p { margin-block:1em; }
</style>
<div class="prose">
  <p>문단 1</p>
  <p>문단 2</p>
</div>
```

### 8.5 inline-block 공백 제거 트릭
```html
<style>.ib{display:inline-block;width:6ch;background:#eef2ff}</style>
<div>
  <span class="ib">A</span><!--
  --><span class="ib">B</span><!--
  --><span class="ib">C</span>
</div>
```

---

## 9. 데모: 핵심 차이 한눈에

```html
<!doctype html>
<meta charset="utf-8">
<title>display: block vs inline vs inline-block</title>
<style>
  .demo h3{margin:.75rem 0}
  .wrap{border:1px dashed #94a3b8; padding:.75rem; margin:.75rem 0}
  .blk { display:block; background:#cfe8ff; margin:.25rem 0; }
  .inl { display:inline; background:#ffe680; padding:.25em .5em; }
  .ibl { display:inline-block; width:8ch; height:3em; background:#ffd6e7; vertical-align:middle; }
  .note{color:#334155; font-size:.925rem}
</style>

<section class="demo">
  <h3>1) block</h3>
  <div class="wrap">
    <span class="blk">block A</span>
    <span class="blk">block B</span>
    <div class="note">→ 새 줄, 가로 전체 차지</div>
  </div>

  <h3>2) inline</h3>
  <div class="wrap">
    <span class="inl">inline A</span>
    <span class="inl">inline B</span>
    <div class="note">→ 줄 안 배치, width/height 직접 지정 불가</div>
  </div>

  <h3>3) inline-block</h3>
  <div class="wrap">
    <span class="ibl">ibl A</span>
    <span class="ibl">ibl B</span>
    <span class="ibl">ibl C</span>
    <div class="note">→ 줄 안 + width/height 가능, 공백 간극 이슈 주의</div>
  </div>
</section>
```

---

## 10. “display만 바꿔” 해결하려다 겪는 함정과 해법

| 증상 | 원인 | 빠른 해법 |
|---|---|---|
| 인라인에 높이/너비 안 먹음 | IFC 규칙 | `inline-block`/`inline-flex`로 전환 |
| 아이콘이 위/아래로 어긋남 | baseline 정렬 | `vertical-align: middle|-Npx` 조정 |
| 문단 상단 마진이 부모 밖으로 | 마진 겹침 | 부모에 `flow-root`/`overflow:auto` |
| 카드 칼럼이 줄바꿈 위치 이상 | inline-block 공백 | Grid/Flex로 리팩터링(권장) |
| 래퍼에 display:contents 후 포커스 링 사라짐 | 접근성 트리 영향 | 가능하면 사용 자제, 대안 구조 고려 |

---

## 11. 수학(요약) — 라인 박스/베이스라인 개념

- **라인 박스 높이**는 해당 줄에 들어있는 인라인 박스들의 **`line-height`의 최대치**와 글꼴 메트릭으로 결정.
- 동일 줄의 인라인 박스 i에 대한 **정렬 오프셋**:
  $$
  \Delta y_i = f(\text{vertical-align}_i, \text{font metrics}, \text{line-height})
  $$
- `inline-block`의 기본 베이스라인은 내부 마지막 라인의 베이스라인(많은 경우 **박스 하단 padding 상단** 부근).
  일치가 필요하면 `vertical-align:top|middle|bottom`으로 **명시 정렬**.

---

## 12. 마이그레이션 체크리스트

- **줄에서 자연스럽게** 놓이되 **크기 제어**가 필요 → `inline-block` 또는 `inline-flex`.
- 박스 간 **분배/정렬**이 핵심 → `flex`/`grid`.
- 마진 겹침 버그/clearfix 대체 → `display: flow-root;`.
- 래퍼 제거가 필요한 복잡한 DOM → `display: contents;`(접근성 테스트 필수).
- 리스트 마커 커스텀 → `display: list-item` + `::marker`.

---

## 13. 보너스: 기타 display 값 간단 메모

- `none`: **박스 생성 안 함**(렌더·탭순서에서 제외). 애니메이션/전환 시나리오는 `visibility`/`opacity`가 더 매끄럽기도.
- `flex`: 1차원 레이아웃(축 정렬/분배).
- `grid`: 2차원 레이아웃(셀/트랙/오토플로우).
- `table`: 테이블 박스 모델(table/table-row/table-cell 등).
- `ruby`: 루비 주석(동아시아 발음 표기).
- (참고) `inline-flex`, `inline-grid`: 줄 안에서 놓이는 플렉스/그리드 컨테이너.

---

## 14. 예제: “초안 + 확장” 통합 실습 페이지

```html
<!doctype html>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>display 통합 실습</title>
<style>
  body{font:16px/1.6 system-ui,-apple-system,Segoe UI,Roboto,Apple SD Gothic Neo,Malgun Gothic,sans-serif;margin:0;padding:2rem;background:#f8fafc;color:#0f172a}
  h2{margin:.5rem 0}
  section{margin-block:1.5rem}
  .panel{border:1px solid #cbd5e1;border-radius:.75rem;background:#fff;padding:1rem}
  .row{display:flex;gap:1rem;flex-wrap:wrap}
  .box{padding:.5rem 1rem;border-radius:.5rem}

  /* block */
  .block-demo .a{display:block;background:#dbeafe}
  .block-demo .b{display:block;background:#bfdbfe}

  /* inline */
  .inline-demo .a{display:inline;background:#fde68a;padding:.25em .5em}
  .inline-demo .b{display:inline;background:#fcd34d;padding:.25em .5em}
  .inline-demo .note{font-size:.9375rem;color:#475569}

  /* inline-block */
  .ibl-demo .a,.ibl-demo .b{
    display:inline-block;width:10ch;height:3em;background:#fecdd3;vertical-align:middle;text-align:center;line-height:3em
  }

  /* flow-root to avoid margin-collapsing */
  .flow-root{display:flow-root;padding:1rem;background:#f1f5f9;border-radius:.5rem}
  .flow-root p{margin:1em 0}

  /* contents */
  .contents{display:contents}
  .contents > div{background:#ffe4e6;padding:.5rem;border:1px solid #f43f5e}

  /* inline-block gap fix (font-size:0) */
  .nogap{font-size:0}
  .nogap .ib{display:inline-block;width:6ch;height:2.5em;line-height:2.5em;background:#e0f2fe;margin:0 .125rem;font-size:1rem;text-align:center}
</style>

<h2>1) block</h2>
<div class="panel block-demo">
  <div class="a">Block A</div>
  <span class="b">Span도 block이면 새 줄</span>
</div>

<h2>2) inline</h2>
<div class="panel inline-demo">
  <span class="a">Inline A</span>
  <span class="b">Inline B</span>
  <div class="note">→ width/height는 직접 지정 불가, padding/border만 체감</div>
</div>

<h2>3) inline-block</h2>
<div class="panel ibl-demo">
  <span class="a">A</span>
  <span class="b">B</span>
</div>

<h2>4) flow-root로 문단 상단 마진 겹침 방지</h2>
<div class="panel flow-root">
  <p>문단 1(상단 마진)</p>
  <p>문단 2(상단 마진)</p>
</div>

<h2>5) contents(접근성 점검 필요)</h2>
<div class="panel">
  <div class="contents">
    <div>자식 1</div>
    <div>자식 2</div>
  </div>
  <p>부모 컨테이너 박스 없이 자식만 배치</p>
</div>

<h2>6) inline-block 간 공백 제거</h2>
<div class="panel nogap">
  <span class="ib">A</span><span class="ib">B</span><span class="ib">C</span>
</div>
```

---

## 15. 최종 체크리스트

- 줄 안에서 **히트 영역/크기 제어**가 필요 → `inline-block`/`inline-flex`.
- 간격과 분배가 핵심 → `flex`/`grid`(+ `gap`).
- 문단 마진 겹침/float 래핑 문제 → `flow-root`.
- 래퍼 제거가 필요하나 **접근성 영향** 검토 후 `contents` 선택.
- 베이스라인 어긋남 → `vertical-align`으로 정렬.
- inline-block 간 공백 → HTML 공백 제거/`font-size:0`/Flex/Grid로 리팩터링.

---

## 16. 결론

- **block/inline/inline-block**은 레이아웃의 **기초 문맥**을 고르는 스위치입니다.
- **무엇이 줄을 만들고**, **무엇이 라인 박스를 만들며**, **마진이 어떻게 상호작용하는지**를 이해하면
  Flex/Grid 같은 상위 레이아웃도 **안정적으로** 설계할 수 있습니다.
- 새 프로젝트라면: **콘텐츠(typography)는 block/inline 규칙**, **컴포넌트는 inline-block/flow-root**,
  **배치와 분배는 Flex/Grid**라는 층위를 의식해 구조를 설계하세요.
