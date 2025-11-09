---
layout: post
title: CSS - z-index와 레이어
date: 2025-04-01 21:20:23 +0900
category: CSS
---
# z-index와 레이어(스택)

## 0) 한눈에 보기

- `z-index`는 **같은 스택킹 컨텍스트(stacking context)** 안에서만 우열을 가립니다.
- **새 스택킹 컨텍스트**를 만들면 그 내부의 모든 레이어는 **한 덩어리**처럼 외부와 경쟁합니다.
- **positioned 요소**(`relative/absolute/fixed/sticky`) 또는 **특정 속성**(예: `opacity<1`, `transform` 등)이 스택킹 컨텍스트를 만듭니다.
- **Flex/Grid 아이템**은 `position: static`이어도 `z-index`를 주면 스택킹 컨텍스트를 만들 수 있습니다(규격 변경 이후).

---

## 1) 기본: 문서 흐름과 레이어

기본적으로 요소는 **문서 흐름(소스 순서)**에 따라 위로 쌓입니다. 뒤에 오는 형제가 **위**에 그려지죠.  
`z-index`는 이 기본 규칙을 **같은 컨텍스트** 안에서 재정의합니다.

- 기본 스택: “먼저 그려진 것(뒤에 있는 것)” → “나중에 그려진 것(앞에 보이는 것)”
- `z-index` 숫자가 클수록 **앞**(사용자 쪽)으로 옵니다.

---

## 2) position과 z-index

- `z-index`가 의미 있으려면 보통 **positioned**여야 합니다:
  - `position: relative | absolute | fixed | sticky` → `z-index` 유효
  - `position: static` → 전통적으로 `z-index` 무효

> **예외(현대 스펙)**  
> **Flex 아이템과 Grid 아이템**은 `position: static`이어도 `z-index`가 **자동이 아닌 값**이면 스택킹 컨텍스트를 생성합니다.  
> 즉, 컨테이너가 `display: flex | grid`이고, 자식이 그 “아이템”이라면 `z-index`가 작동합니다.

```html
<!-- static 이지만 flex 아이템이므로 z-index 동작 -->
<ul style="display:flex">
  <li style="z-index:2">위</li>
  <li style="z-index:1">아래</li>
</ul>
```

---

## 3) z-index 값과 의미

```css
.low    { z-index: -1; } /* 음수 → 같은 컨텍스트 안에서 뒤쪽 */
.normal { z-index:  0; } /* 기준 레벨 */
.high   { z-index: 100; }/* 양수 → 같은 컨텍스트 내부에서 앞쪽 */
```

- 기본값은 `auto`(부모/문맥에 따라 결정, 별도 스택킹 컨텍스트 미생성).
- **음수**는 같은 컨텍스트 내부에서 **배경에 깔리는** 느낌으로 뒤로 밀립니다.
- **주의**: 음수라고 해서 **부모의 배경 뒤**로 완전히 빠지는 것은 **아닙니다**. 같은 컨텍스트 내부 **순서만** 낮아질 뿐입니다.

---

## 4) 스택킹 컨텍스트(stacking context)

### 4.1 왜 중요한가?
스택킹 컨텍스트는 **독립된 z축 무대**입니다. **컨텍스트 밖의 요소**와는 **자식의 z-index 크고 작음이 직접 비교되지 않습니다**.  
**컨텍스트(부모)끼리** 먼저 비교가 되고, 내부(자식) 비교는 **그 다음**이죠.

### 4.2 스택킹 컨텍스트를 만드는 대표 조건

- **positioned** 요소(`relative/absolute/fixed/sticky`) + `z-index`가 `auto`가 아닌 경우
- **`opacity < 1`**
- **`transform`**(2D/3D), **`perspective`**, **`filter`**, **`backdrop-filter`**
- **`mix-blend-mode`**가 `normal`이 아닌 경우
- **`will-change`**에 `transform`/`opacity`/`filter` 등이 선언된 경우
- **`contain: paint | layout paint | strict | content`** (paint 포함)
- **`isolation: isolate`**
- **`clip-path`**(대부분의 브라우저)
- **Flex/Grid 아이템**이면서 `z-index`가 `auto`가 아닌 경우(앞서 언급)

```html
<div class="parent">
  부모
  <div class="child">자식</div>
</div>
```
```css
.parent {
  position: relative;
  z-index: 10;       /* ❗ 새 스택킹 컨텍스트 생성 */
}
.child {
  position: absolute;
  z-index: 9999;     /* 부모 컨텍스트 안에서만 의미 있음 */
}
```

위에서 `.child`가 `9999`라도, **다른 컨텍스트**의 요소가 `z-index: 11`인 부모에 속해 있으면  
그 **다른 부모(11)**가 `.parent(10)`보다 앞에 오며, `.child`는 부모와 **함께** 뒤에 남습니다.

---

## 5) 페인팅(그려지는) 순서 — 한 컨텍스트 내부의 계층

하나의 스택킹 컨텍스트 내부에서의 **단순화된** 페인팅 순서는 아래와 같습니다(상세 스펙은 더 복잡).

1) 컨텍스트 **자기 자신의 배경과 테두리**  
2) **음수 `z-index`**를 가진 자손(낮은 → 높은 순)  
3) `z-index: auto/0`의 **일반 흐름/플로트/포지션드** 자손(대략 문서/소스 순)  
4) **양수 `z-index`**를 가진 자손(낮은 → 높은 순)

> **핵심**: 음수는 항상 배경 뒤쪽 그룹, 양수는 항상 맨 앞 그룹입니다.  
> `auto/0`(또는 미지정)은 중간층에서 **소스 순** 등으로 정렬됩니다.

수식으로 요약하면(컨텍스트 C에서 레이어 L의 정렬 키 \(k\)):
$$
k(L) =
\begin{cases}
(-2,\, \text{z}) & \text{if } z < 0 \\
(-1,\, \text{flowOrder}) & \text{if } z = 0 \text{ or } z=\text{auto}\\
(0,\, \text{z}) & \text{if } z > 0
\end{cases}
$$
(스펙 단순화 표현. 실제는 더 많은 세부 단계가 있습니다.)

---

## 6) transform/opacity/filter/isolation/contain과의 상호작용

```css
.box {
  transform: translateZ(0); /* ✅ 새 스택킹 컨텍스트 생성 */
  /* opacity: .9999;       /* 이것만으로도 새 컨텍스트(불투명도 < 1) */
  /* filter: blur(0);      /* 필터 지정 시 컨텍스트 */
  /* isolation: isolate;   /* 자신의 독립 페인팅 콘텍스트 분리 */
  /* contain: paint;       /* 페인팅 격리 → 컨텍스트 */
}
```

- **의도치 않은 컨텍스트 생성**은 모달/툴팁 같은 레이어가 **“올라오지 않거나 내려오지 않는”** 문제를 유발합니다.
- `transform`이 걸린 조상을 경계로 **`position: fixed`의 기준**이 바뀌기도 합니다(포지션 글에서 다룸).
- `isolation: isolate`는 **컴포넌트 스코프 단위**로 혼합/블렌딩 영향을 차단할 때 유용합니다.

---

## 7) Flex/Grid 아이템 특례

- 과거: `z-index`는 **positioned** 요소에서만 의미.
- 현재: **Flex/Grid 아이템**은 `position: static`이라도 `z-index`가 **auto가 아닌 값**이면 스택킹 컨텍스트를 생성하고, **겹침 순서에도 참여**합니다.

```html
<ul class="row">
  <li class="card a">A</li>
  <li class="card b">B</li>
</ul>
```
```css
.row { display: flex; gap: 1rem; }
.card { /* position: static */ }
.a { z-index: 10; } /* flex 아이템: 동작 */
.b { z-index:  1; }
```

---

## 8) 음수 z-index의 정확한 작동

```html
<div class="ctx">
  <div class="neg">neg</div>
  <div class="mid">mid</div>
  <div class="pos">pos</div>
</div>
```
```css
.ctx { position: relative; z-index: 0; background: #eef2ff; }
.neg { position: absolute; z-index: -1; background: #93c5fd; }
.mid { position: relative;             background: #60a5fa; }
.pos { position: absolute; z-index:  1; background: #3b82f6; }
```

- `.neg`는 `.ctx`(같은 컨텍스트)의 **배경 위**에서만 뒤로 깔립니다.  
- `.ctx` **외부**의 요소보다 더 뒤로 가는 것이 아닙니다(컨텍스트 밖과는 묶음 단위 비교).

---

## 9) overflow/clip과 z-index

- `overflow: hidden/auto/scroll`은 **스택킹 컨텍스트를 만들지 않지만**, **클리핑 영역**을 만듭니다(보이는 영역 제한).
- `clip-path`는 대개 **새 스택킹 컨텍스트**를 만듭니다. 모양 클리핑과 레이어 분리를 함께 고려해야 합니다.

---

## 10) 모달/오버레이/드롭다운 — 레이어 설계 패턴

### 10.1 “루트 포털” 전략(권장)
- 모든 오버레이/모달/툴팁을 **문서 루트(body) 직계**에 모읍니다.
- 앱 어디든 추가되더라도 같은 큰 컨텍스트에서 `z-index` 우선순위를 단순화.

```html
<!-- 앱 어디선가 -->
<button class="btn" data-open="#dialog">열기</button>

<!-- 문서 하단의 포털 루트 -->
<div id="__portal"></div>
```

오버레이 DOM을 `#__portal`에 삽입(프레임워크 포털/텔레포트 기능 활용).

### 10.2 z-index 스케일(디자인 토큰)
```css
:root {
  --z-base:      0;
  --z-dropdown:  1000;
  --z-sticky:    2000;
  --z-fixed:     3000;
  --z-popover:   4000;
  --z-modal:     5000;
  --z-toast:     6000;
  --z-critical:  9999;
}
```

이처럼 **의미 기반** 레벨을 정의해 두면 팀 협업/유지보수가 쉬워집니다.

### 10.3 드롭다운이 안 보이는 이유
- 조상에 **스택킹 컨텍스트**(transform/opacity/…)가 있거나 **클리핑(overflow)**이 있어 **잘려서** 그렇습니다.  
  → 포털로 올리거나, 컨텍스트/클리핑 경계를 재설계합니다.

---

## 11) 종합 예제 — 겹침 이슈가 있는/없는 버전

```html
<!doctype html>
<meta charset="utf-8" />
<title>z-index layer lab</title>
<style>
  body{font:16px/1.6 system-ui, -apple-system, Segoe UI, Roboto, Malgun Gothic, sans-serif; margin:0; padding:2rem; background:#f8fafc}

  .stage { display:grid; grid-template-columns: 1fr 1fr; gap:2rem; }

  /* 좌측: 의도치 않은 컨텍스트/클리핑으로 드롭다운이 가려짐 */
  .left .card {
    position: relative; padding:1rem; border:1px solid #cbd5e1; border-radius:.75rem; background:#fff;
    transform: translateZ(0); /* ❗ 새 스택킹 컨텍스트 생성 */
    overflow: hidden;         /* ❗ 클리핑 발생 */
  }
  .left .btn { position: relative; z-index: 1; }
  .left .menu {
    position: absolute; top:2.5rem; left:0; background:#111; color:#fff; padding:.5rem .75rem; border-radius:.5rem;
    z-index: 9999; /* 숫자 커도, card 컨텍스트/클리핑에 갇힘 */
  }

  /* 우측: 포털로 루트에 띄우는 버전(여기서는 단순히 fixed로 데모) */
  .right .card {
    position: relative; padding:1rem; border:1px solid #cbd5e1; border-radius:.75rem; background:#fff;
  }
  .right .menu {
    position: fixed; /* 포털로 body에 붙였다고 가정 */
    top: 5rem; left: 50%; transform: translateX(-50%);
    background:#111; color:#fff; padding:.5rem .75rem; border-radius:.5rem;
    z-index: var(--z-popover, 4000);
  }

  /* 시각 보조 */
  .ghost { height: 36vh; background:linear-gradient(90deg,#e2e8f0 0 50%, #f1f5f9 0 100%); border-radius:.75rem }
</style>

<div class="stage">

  <section class="left">
    <h2>문제 사례: transform + overflow로 드롭다운이 잘림</h2>
    <div class="card">
      <button class="btn">열기</button>
      <div class="menu">드롭다운(가림/잘림)</div>
      <div class="ghost"></div>
    </div>
  </section>

  <section class="right">
    <h2>해결: 루트 포털(또는 fixed) + 토큰 스케일</h2>
    <div class="card">
      <button class="btn">열기</button>
      <div class="menu">드롭다운(항상 위)</div>
      <div class="ghost"></div>
    </div>
  </section>

</div>
```

---

## 12) DevTools로 디버깅하는 절차(체크리스트)

1. **요소 선택** → **Computed 패널**에서 다음 속성들을 확인:
   - `position`, `z-index`, `opacity`, `transform`, `filter`, `mix-blend-mode`, `isolation`, `contain`, `clip-path`
2. **부모 체인**을 거슬러 올라가며 **스택킹 컨텍스트 생성자**가 있는지 찾기:
   - 있으면 그 **경계 밖**의 요소와는 자식 z-index가 직접 경쟁하지 않음
3. **Layers(Chrome)** / **3D View(Firefox)**:
   - 레이어 분리/컴포지팅 여부를 시각화
4. **overflow/clip** 확인:
   - 조상에 `overflow:hidden/auto/scroll`이 있으면 **가려질 수 있음**
5. **대안 설계**:
   - 오버레이는 **루트 포털**로 이동
   - 또는 컨텍스트 생성 속성 제거/이동
6. **안전 스케일** 적용:
   - 디자인 토큰으로 `--z-modal`, `--z-popover` 등 **의미 기반** 상수 사용

---

## 13) 미세 규칙: inline/float/positioned의 기본 페인팅 흐름(요약)

한 컨텍스트 내부에서 **대략적인** 순서:  
1) 컨텍스트 자신의 배경/테두리  
2) 음수 z-index positioned  
3) 블록/플로트/인라인(일반 흐름)  
4) z-index `auto`인 positioned  
5) 양수 z-index positioned  

> 실제 스펙(CSS2.1)의 라인/플로트 상세 단계는 더 많지만, **문제 해결**에는 위 계층만 알아도 충분합니다.

---

## 14) 수식 두 가지 — 비교 키와 높이쌓기 직관화

### 14.1 같은 컨텍스트 내 z-index 비교 키(단순화)
$$
\text{order}(E) =
\begin{cases}
(-1, z) & z < 0 \\
(0, \text{flowIndex}) & z = 0 \text{ or } z=\text{auto} \\
(1, z) & z > 0
\end{cases}
$$

### 14.2 컨텍스트 간 비교
부모 컨텍스트 \(C_A, C_B\)가 있고 각각에 자식 \(a_i, b_j\)가 있을 때,  
**먼저** \(C_A\) vs \(C_B\)가 비교되고, **그 다음** 내부의 \(a_i\) vs \(a_k\)가 비교됩니다.  
즉, 자식이 **아무리 큰 z-index**라도 **부모의 순서**를 넘을 수 없습니다.

---

## 15) 실전 레시피 모음

### 15.1 모달/오버레이(최상위 계층)
```css
:root {
  --z-overlay:  9000;
  --z-modal:    9500;
  --z-toast:    9600;
}
.overlay { position: fixed; inset: 0; background: rgba(0,0,0,.5); z-index: var(--z-overlay); }
.modal   { position: fixed; inset: 0; display:grid; place-items:center; z-index: var(--z-modal); }
.toast   { position: fixed; right: 1rem; bottom: 1rem; z-index: var(--z-toast); }
```

### 15.2 헤더/드롭다운/팝오버(중간 계층)
```css
:root {
  --z-header:   2000;
  --z-dropdown: 3000;
  --z-popover:  4000;
}
.header   { position: sticky; top: 0; z-index: var(--z-header); }
.dropdown { position: absolute; z-index: var(--z-dropdown); }
.popover  { position: absolute; z-index: var(--z-popover); }
```

### 15.3 캔버스 위에 정보 오버레이
```css
.canvas { position: relative; }
.overlay-info {
  position: absolute; inset: 1rem; z-index: 10; 
  background: rgba(255,255,255,.8); backdrop-filter: blur(4px);
}
```

---

## 16) 흔한 함정과 해결

| 증상 | 원인 | 해결 |
|---|---|---|
| z-index를 크게 줘도 안 올라옴 | **다른 컨텍스트**에 갇힘 | 상위 컨텍스트 제거/이동, **포털**로 루트에 띄우기 |
| 드롭다운이 잘림 | 조상 `overflow` 클리핑 | 포털/배치 위치 변경, 클리핑 해소 |
| fixed가 스크롤 따라 움직임 | 조상 `transform/filter/contain` → 기준 전환 | 해당 속성 제거, 루트로 이동 |
| 음수 z-index가 부모 뒤로 사라짐 | 기대와 달리 **부모 배경 뒤로는 안 감** | 음수는 같은 컨텍스트 내 순서일 뿐임을 이해, 설계 재검토 |
| Flex/Grid에서 static인데 z-index가 동작 | 스펙 특례 | 의도된 것인지 확인. 그렇지 않다면 `auto`로 되돌리기 |

---

## 17) 최종 체크리스트

- **컨텍스트 먼저 본다**: transform/opacity/filter/isolation/contain/clip-path/position+z-index?
- **클리핑 확인**: overflow가 위/아래 잘라먹지 않는가?
- **루트 포털 전략**: 모달/툴팁/드롭다운은 가능한 상위(루트)에 모으기.
- **토큰 스케일**: 의미 기반 z-index 상수로 팀 규칙을 만든다.
- **DevTools 레이어 시각화**: Layers/3D View로 컨텍스트 단위 확인.

---

## 18) 마무리

`z-index`는 **숫자 싸움**이 아니라 **컨텍스트 싸움**입니다.  
“누가 더 큰 숫자냐”보다 **“같은 무대 위에 있느냐”**가 먼저죠.  
스택킹 컨텍스트와 페인팅 순서를 이해하고 **루트 포털 + 토큰 스케일**을 도입하면,  
모달/오버레이/툴팁/드롭다운 등 인터랙션 UI의 레이어 설계가 **단순**하고 **강력**해집니다.