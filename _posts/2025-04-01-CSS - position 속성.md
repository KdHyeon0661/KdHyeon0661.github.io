---
layout: post
title: CSS - position 속성
date: 2025-04-01 20:20:23 +0900
category: CSS
---
# position 속성 (static, relative, absolute, fixed, sticky)

## 0. 큰 그림 요약

- `position`은 **박스를 문서 흐름에 어떻게 참여시킬지**와 **오프셋(top/left/…)을 어떻게 해석할지**를 정의.
- 다섯 값:
  - `static`: 기본 흐름. 오프셋 무시.
  - `relative`: 자리(흐름)는 그대로 두고, **시각적**으로만 이동.
  - `absolute`: 흐름에서 제거, **가장 가까운 positioned 조상**(또는 초기 컨테이닝 블록)을 기준.
  - `fixed`: **뷰포트**(또는 특정 조상; 아래 설명)를 기준. 스크롤에도 고정.
  - `sticky`: 특정 임계점까지는 `relative`처럼, 넘으면 **컨테이너 범위 내에서** 고정.

---

## 1. 용어 정리 — Containing Block(컨테이닝 블록)과 Offset

오프셋(`top/right/bottom/left`)은 **컨테이닝 블록**의 패딩 박스 기준으로 계산됩니다.

- `static`: 오프셋 무시 → 컨테이닝 블록 개념을 사실상 쓰지 않음.
- `relative`: 자신의 **원래 자리**(static 위치)의 컨테이닝 블록을 기준.
- `absolute`: 가장 가까운 **positioned 조상**(`relative/absolute/fixed/sticky`)의 **패딩 박스**. 없으면 **뷰포트 초기 컨테이닝 블록**.
- `fixed`: 기본은 **뷰포트** 기준. 단, 조상 중 `transform/perspective/filter/contain/transform-style: preserve-3d/will-change` 등 **containing block을 생성하는 요소**가 있으면 **그 요소** 기준이 됨.
- `sticky`: **자신이 속한 스크롤 컨테이너의 패딩 박스** 기준 + 조상(스크롤 클리핑) 경계 제약.

오프셋 논리 속성(국제화/세로쓰기 대응):
```css
/* 물리 속성 */
top: 0; right: 0; bottom: 0; left: 0;

/* 논리 속성 (권장) */
inset-block-start: 0;  /* top에 해당 */
inset-inline-end: 0;   /* right에 해당 */
inset-block-end: 0;    /* bottom */
inset-inline-start: 0; /* left */

/* 일괄 지정 */
inset: 0;              /* top/right/bottom/left 모두 0 */
```

퍼센트 오프셋 기준:
- `top/bottom`의 `%`는 **컨테이닝 블록의 블록 축 높이** 기준.
- `left/right`의 `%`는 **컨테이닝 블록의 인라인 축 너비** 기준.

---

## 2. `static` — 기본 흐름

```html
<div class="box static">static</div>
```
```css
.box.static {
  position: static;   /* 기본값 */
  top: 20px;          /* 무시 */
  background: #a3a3a3;
}
```

특징
- 문서 흐름에 따라 배치. `top/left/right/bottom`은 **무시**.
- `z-index` 지정해도 **stacking context 생성 X**(페인팅 순서에서 특별 대우 없음).

사용
- 의미 없는 오프셋을 남발하지 않고, 기본 흐름·레이아웃 시스템(Grid/Flex)으로 충분할 때.

---

## 3. `relative` — 자리 유지 + 시각 이동

```html
<div class="card">
  <span class="badge">NEW</span>
  설명 텍스트…
</div>
```
```css
.card {
  position: relative; /* 자식 absolute의 기준점 역할도 겸함 */
  padding: 1rem;
  border: 1px solid #e5e7eb;
}
.badge {
  position: relative;
  top: -4px;          /* 원래 자리에서 살짝 위로 */
  left: .25rem;
  background: #111; color:#fff; padding: .125rem .375rem; border-radius: .25rem;
}
```

특징
- 원래 **배치 공간은 그대로**(다른 요소의 레이아웃에 영향 X) + 시각만 이동.
- **자식 absolute의 기준**으로 자주 활용(부모를 `relative`, 자식을 `absolute`).
- `z-index`를 주면 그 요소는 **stacking context** 생성(조건: `position`이 `relative/absolute/fixed/sticky` 중 하나이고 `z-index`가 `auto`가 아님).

---

## 4. `absolute` — 흐름에서 분리, 기준 조상 기준

```html
<div class="wrap">
  <button class="fab">+</button>
</div>
```
```css
.wrap {
  position: relative;  /* 기준 조상 */
  min-height: 240px; background: #f8fafc; border: 1px dashed #94a3b8;
}
.fab {
  position: absolute;
  right: 1rem; bottom: 1rem;
  width: 3rem; height: 3rem; border-radius: 50%;
  background: #111; color:#fff; border:0;
}
```

특징
- 문서 흐름에서 **공간 차지 X**.
- 가장 가까운 **positioned 조상** 기준. 없으면 초기 컨테이닝 블록(보통 뷰포트).
- 오버레이/툴팁/버튼 배치에 유용.

주의
- 스크롤 영역에서 **조상에 `overflow: hidden/auto`**가 있으면 **클리핑**될 수 있음(툴팁 잘림).
- 테이블 레이아웃(`display: table-cell/table`) 내부 기준은 브라우저별 특수 규칙이 있으므로 가능하면 래퍼로 `relative`를 명시.

---

## 5. `fixed` — 뷰포트(또는 특수 조상) 기준 고정

```html
<div class="toast">Saved</div>
```
```css
.toast {
  position: fixed;
  right: 1rem; bottom: 1rem;
  background: #111; color:#fff; padding:.75rem 1rem; border-radius:.5rem;
}
```

특징
- 스크롤에도 **항상 같은 곳**.
- **원칙**: 뷰포트 기준이지만, 조상 중 `transform: translateZ(0)` 등 **containing block 생성자**가 있으면 **그 조상 기준**이 될 수 있음.
  - 고정 헤더/모달이 특정 섹션에 “붙어” 버릴 때, 상위의 `transform`/`filter`/`perspective` 여부 확인.
- 모바일 브라우저에서 주소창/툴바의 등장·퇴장에 따라 상대적 **보이는 영역**이 달라짐 → 배치가 출렁일 수 있음.  
  `env(safe-area-inset-*)`와 함께 여백 보정 가능.

```css
.header {
  position: fixed; top: 0; left: 0; right: 0;
  padding-top: calc(8px + env(safe-area-inset-top));
}
```

---

## 6. `sticky` — 스크롤 임계점에서 고정

```html
<div class="scroll">
  <div class="sticky-head">목록</div>
  <!-- 긴 콘텐츠 나열 -->
</div>
```
```css
.scroll {
  height: 260px; overflow: auto; border: 1px solid #94a3b8;
  padding: .5rem;
}
.sticky-head {
  position: sticky;
  top: 0;                     /* 임계점 기준 */
  background: #fff; z-index: 1;  /* 겹침 보정 */
  padding: .5rem .75rem; border-bottom: 1px solid #e5e7eb;
}
```

작동 조건
- **스크롤 가능한 조상**(자기 자신 포함)의 **패딩 박스**가 기준.
- `top`(또는 `inset-*`) **필수**.
- **부모의 경계를 벗어나지 않음**(부모 영역 내에서만 고정).
- **조상에 `overflow: hidden/auto/scroll`**가 있으면 **그 컨테이너 안**에서만 sticky 적용.
- 스택 순서가 겹칠 수 있으므로 `z-index`로 보정.

흔한 오해/실수
- sticky가 “안 먹는다”: 스크롤 컨테이너가 어디인지, `top`을 줬는지, 조상 `overflow`가 임의로 끊고 있지 않은지 점검.

---

## 7. stacking context(페인팅·레이어 순서)와 z-index

**다음 중 하나에 해당하면 stacking context를 생성**합니다(발췌):
- `position`이 `relative/absolute/fixed/sticky` 이고 `z-index`가 `auto`가 아닌 경우
- `opacity < 1`, `filter`, `transform`, `will-change`, `mix-blend-mode`, `isolation: isolate`, `perspective` …
- `position: fixed` 자체는 브라우저에 따라 상위 컨텍스트와의 상호작용이 달라질 수 있으나 실전에서는 **z-index와 transform 조상을 함께 고려**하는 습관이 중요.

실전 규칙
- 같은 stacking context 안에서는 **z-index 숫자 큰 것**이 앞.
- 다른 stacking context 간에는 **컨텍스트 전체가 하나의 레이어처럼** 정렬.
- “z-index 안 먹는다”의 대부분은 **다른 컨텍스트에 갇힘** 문제.

---

## 8. 논리 오프셋과 4방 오프셋 우선 순서

- `inset`(축약)은 `top/right/bottom/left` 또는 논리 쌍(`inset-block`, `inset-inline`)보다 **낮은 우선순위**.
- 예:
```css
.box { inset: 0; left: 20px; } /* left가 우선 */
```

---

## 9. 퍼센트 오프셋/크기와 컨테이닝 블록

- `absolute/fixed`의 `%` 오프셋은 **컨테이닝 블록의 해당 축 크기** 기준.
- `width/height:auto` + 오프셋 조합:
  - `left`와 `right`를 동시에 주면 **너비가 결정**됨.
  - `top`과 `bottom`을 동시에 주면 **높이가 결정**됨.
```css
.abs {
  position: absolute;
  left: 1rem; right: 1rem; /* → 박스 width = CB 너비 - (left+right) */
  top: 0; bottom: 0;       /* → 박스 height = CB 높이 - (top+bottom) */
}
```

---

## 10. 테이블/리플레이스드 요소와 position

- `img`, `video`, `canvas`(replaced)는 **intrinsic size**를 가지므로 인라인이라도 크기 제어가 직접 가능.
- `absolute`가 테이블 셀 내부에서 기준을 잃을 수 있으므로, **셀/행/랩퍼에 `position: relative`**를 명시해 기준을 고정.

---

## 11. 실전 패턴 레시피

### 11.1 툴팁/드롭다운(스크롤·클리핑 안전형)
```html
<button class="btn">
  액션
  <span class="pop" role="tooltip">툴팁 내용</span>
</button>
```
```css
.btn {
  position: relative; /* 기준 */
  padding: .5rem .75rem; border:1px solid #e5e7eb; background:#fff;
}
.pop {
  position: absolute; inset-inline-start: 0; inset-block-start: 100%;
  margin-block-start: .5rem;
  transform: translateX(-10%);
  background:#111; color:#fff; padding:.5rem .75rem; border-radius:.5rem;
  white-space: nowrap;
  z-index: 10;
}
.btn:focus .pop, .btn:hover .pop { display:block; }
```
팁
- 스크롤 컨테이너 내부라면 **클리핑**(툴팁 잘림) 가능 → 포털(루트로 이동) 또는 `position: fixed` + 좌표 계산으로 대체.

### 11.2 고정 헤더 + 앵커 스크롤 보정
```css
.header {
  position: fixed; inset-inline: 0; inset-block-start: 0;
  height: 64px; background:#fff; border-bottom:1px solid #e5e7eb; z-index:100;
  padding-block-start: env(safe-area-inset-top); /* 노치 보정 */
}
:root { scroll-padding-top: 64px; }        /* 전역 보정 */
h2[id] { scroll-margin-top: 64px; }        /* 개별 보정 */
```

### 11.3 섹션 내 고정 사이드 목차(sticky)
```css
.layout { display:grid; grid-template-columns: 1fr 280px; gap: 2rem; }
.toc {
  position: sticky; top: 1rem; /* 섹션 내에서만 고정 */
  align-self: start;
  max-height: calc(100dvh - 2rem); overflow:auto; /* 긴 목록 스크롤 */
}
```

### 11.4 모달 오버레이(fixed + 스택킹 컨텍스트)
```html
<div class="modal" hidden>
  <div class="dialog">내용</div>
</div>
```
```css
.modal {
  position: fixed; inset: 0; 
  display: grid; place-items: center;
  background: rgba(0,0,0,.5);
  z-index: 1000;         /* 상위로 올리기 */
}
.dialog {
  position: relative;     /* 내부 absolute 배치용 */
  width: min(92vw, 560px); background:#fff; border-radius:.75rem; padding: 1.25rem;
}
```

### 11.5 좌표식 레이아웃(absolute로 카드 매핑)
```css
.canvas { position: relative; width: 800px; height: 480px; background:#f1f5f9; }
.node {
  position: absolute; width: 160px; min-height: 80px; padding: .75rem;
  border:1px solid #cbd5e1; border-radius:.5rem; background:#fff;
}
.node.a { left: 24px;  top: 24px; }
.node.b { left: 320px; top: 72px; }
.node.c { left: 520px; top: 300px; }
```
- 반응형이 어렵다 → 가능하면 Grid/Flex + auto-placement를 우선 고려.

---

## 12. 디버깅 체크리스트

- **sticky**가 안 된다 → 스크롤 컨테이너/`top` 지정/조상 `overflow` 확인.
- **fixed**가 스크롤 따라 움직인다 → 조상 `transform/filter/contain` 여부 확인.
- **absolute**가 기준에서 어긋난다 → 기대한 조상에 `position: relative` 명시.
- **z-index**가 안 먹는다 → stacking context를 어디서 생성했는지 추적.
- **툴팁 잘림** → 조상 `overflow`에 클리핑. 루트로 포털/`fixed`로 재설계.

---

## 13. 비교 요약표(확장판)

| 값 | 흐름 참여 | 기준(Containing Block) | 스크롤 영향 | z-index로 SC 생성 | 대표 용도 |
|---|---|---|---|---|---|
| static | O | 기본 흐름 | O | X | 기본 문서 |
| relative | O(자리 유지) | 자기 static 위치 | O | **O**(z-index!=auto) | 미세 정렬, absolute 기준 제공 |
| absolute | X | 가장 가까운 **positioned 조상**(없으면 초기) | X | **O**(z-index!=auto) | 오버레이/툴팁/버튼 |
| fixed | X | **뷰포트**(또는 containing-block 생성 조상) | X | **O**(z-index!=auto) | 헤더/토스트/모달 |
| sticky | O | 스크롤 컨테이너 패딩 박스 | 임계 전 O, 임계 후 고정 | **O**(z-index!=auto) | 섹션 헤더/목차 |

---

## 14. 수식 메모 — 오프셋이 치수에 미치는 영향

상하 오프셋을 모두 지정하면 **높이가 결정**됩니다:
$$
\text{height} = \text{CB\_height} - (\text{top} + \text{bottom})
$$
좌우 오프셋을 모두 지정하면 **너비가 결정**됩니다:
$$
\text{width} = \text{CB\_width} - (\text{left} + \text{right})
$$
여기서 \( \text{CB\_height}, \text{CB\_width} \)는 **컨테이닝 블록의 패딩 박스** 치수.

---

## 15. 통합 예제(학습용 페이지)

```html
<!doctype html>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width, initial-scale=1" />
<title>position 통합 데모</title>
<style>
  body{margin:0;font:16px/1.6 system-ui,-apple-system,Segoe UI,Roboto,Apple SD Gothic Neo,Malgun Gothic,sans-serif;background:#f8fafc;color:#0f172a}
  h2{margin:1rem 0 .5rem}
  .panel{border:1px solid #cbd5e1;border-radius:.75rem;background:#fff;padding:1rem;margin:1rem}
  .demo{display:grid;gap:1rem}

  .box{width:140px;height:80px;color:#fff;display:grid;place-items:center;border-radius:.5rem}
  .static{position:static;background:#737373}
  .relative{position:relative; top:10px; left:20px; background:#1e40af}
  .wrap{position:relative; border:1px dashed #94a3b8; padding:1rem; height:180px}
  .abs{position:absolute; right:1rem; top:1rem; background:#16a34a}
  .fix{position:fixed; right:1rem; bottom:1rem; background:#dc2626; z-index:999}
  .scroll{height:220px; overflow:auto; border:1px solid #94a3b8; padding:.5rem}
  .sticky{position:sticky; top:0; background:#fff; color:#111; border-bottom:1px solid #e5e7eb; padding:.5rem .75rem; z-index:1}
  .space{height:600px}

  /* absolute 크기 결정(양쪽 오프셋) */
  .stretch-wrap{position:relative; border:1px dashed #94a3b8; padding:1rem; height:200px}
  .stretch{
    position:absolute; left:1rem; right:1rem; top:1rem; bottom:1rem;
    border-radius:.5rem; display:grid; place-items:center;
    background:repeating-linear-gradient(45deg,#e0e7ff 0 10px,#fff 10px 20px);
  }

  /* z-index stacking 테스트 */
  .stack-wrap{position:relative; height:160px; background:#eef2ff; border-radius:.5rem}
  .s1,.s2{position:absolute; width:120px; height:80px; display:grid; place-items:center; border-radius:.5rem}
  .s1{left:24px; top:24px; background:#38bdf8; z-index: 1;}
  .s2{left:64px; top:48px; background:#fb7185; z-index: 2;} /* 더 앞 */

  /* fixed 기준 전환(부모 transform이 있으면 그 부모가 기준이 될 수 있음) */
  .tf{transform: translateZ(0);} /* containing block 생성자 */
  .tf .fix2{position:fixed; left:1rem; top:1rem; background:#0ea5e9; padding:.5rem .75rem; border-radius:.5rem}
</style>

<h2>1) static / relative</h2>
<div class="panel demo">
  <div class="box static">static</div>
  <div class="box relative">relative</div>
</div>

<h2>2) absolute (기준: 가장 가까운 positioned 조상)</h2>
<div class="panel wrap">
  래퍼는 <code>position:relative</code>
  <div class="box abs">absolute</div>
</div>

<h2>3) fixed (뷰포트 기준 고정)</h2>
<div class="panel">스크롤을 내려도 우하단 빨간 박스는 고정됩니다.</div>
<div class="box fix">fixed</div>
<div class="space"></div>

<h2>4) sticky (컨테이너 내 고정)</h2>
<div class="panel scroll">
  <div class="sticky">Sticky 헤더</div>
  <p>긴 내용 1...</p><p>긴 내용 2...</p><p>긴 내용 3...</p>
  <p>긴 내용 4...</p><p>긴 내용 5...</p><p>긴 내용 6...</p>
</div>

<h2>5) 절대 배치로 상하좌우 오프셋으로 크기 결정</h2>
<div class="panel stretch-wrap">
  <div class="stretch">left+right & top+bottom → width/height 결정</div>
</div>

<h2>6) z-index stacking</h2>
<div class="panel stack-wrap">
  <div class="s1">z=1</div>
  <div class="s2">z=2</div>
</div>

<h2>7) fixed 기준 전환(transform 조상)</h2>
<div class="panel tf">
  <div class="fix2">부모 transform이 있으면 이 부모가 기준이 될 수 있음</div>
  부모에 <code>transform: translateZ(0)</code> 적용
</div>
```

---

## 16. 자주 묻는 질문(FAQ)

**Q1. sticky가 동작하지 않아요.**  
A. 스크롤 컨테이너가 어디인지 확인하세요. `top`(또는 `inset-*`)을 지정했는지, 조상 `overflow`가 sticky를 **클리핑**하지 않는지 점검하세요. 필요 시 `z-index` 보정.

**Q2. fixed가 스크롤에 따라 움직입니다.**  
A. 조상에 `transform/filter/contain` 등이 있으면 **그 조상 기준**으로 동작할 수 있습니다. 해당 속성을 제거하거나 모달/토스트를 루트로 이동(포털)하세요.

**Q3. absolute가 이상한 곳을 기준으로 잡습니다.**  
A. 기대한 조상에 `position: relative`를 **명시**하세요. 테이블/셀 내부라면 래퍼를 만드는 것이 안전합니다.

**Q4. z-index가 안 먹습니다.**  
A. stacking context가 갈라져 있을 확률이 높습니다. 어떤 요소가 컨텍스트를 만드는지(불투명도, transform 등) 확인 후, 같은 컨텍스트로 묶거나 z-index를 부여할 요소를 재배치하세요.

---

## 17. 실무 체크리스트

- 기준점을 만들 부모에는 `position: relative`.
- 툴팁/드롭다운은 **클리핑**(overflow)과 **포털** 전략을 함께 고려.
- 고정 헤더/푸터는 `position: fixed` + `scroll-padding`/`scroll-margin` 보정.
- sticky는 스크롤 컨테이너·`top` 지정·`z-index` 충돌까지 점검.
- 국제화 대응은 물리 오프셋 대신 `inset-*`(논리 속성) 사용.
- 레이아웃은 Grid/Flex가 우선, `position`은 **예외적 좌표 배치**에 사용.