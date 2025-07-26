---
layout: post
title: CSS - z-index와 레이어
date: 2025-04-01 21:20:23 +0900
category: CSS
---
# z-index와 레이어 개념

CSS에서 요소는 2차원(x, y) 평면에 배치되는 것처럼 보이지만, 실제로는 **z축(깊이)**도 존재합니다. 이 z축은 요소들이 **겹칠 때 어떤 것이 위에 올라갈지**를 결정하는데, 이를 제어하는 속성이 바로 `z-index`입니다.

`z-index`는 **position이 지정된 요소들**에만 작동하며, **정수값**을 사용하여 우선순위를 지정합니다.

---

## 1. 기본 레이어 개념

HTML 요소들은 기본적으로 **작성 순서(문서 흐름 순서)**에 따라 아래에서 위로 쌓입니다. 나중에 작성된 요소일수록 더 위에 배치됩니다. 하지만 `z-index`를 사용하면 **이 쌓임 순서(stacking order)**를 직접 제어할 수 있습니다.

---

## 2. position과 z-index의 관계

> `z-index`는 `position`이 `relative`, `absolute`, `fixed`, `sticky`일 때에만 효과가 있습니다.

```html
<div style="position: static; z-index: 100;">
  이건 z-index가 있어도 static이라 무시됨
</div>

<div style="position: relative; z-index: 2;">위에 보이는 요소</div>
<div style="position: relative; z-index: 1;">아래에 가려지는 요소</div>
```

---

## 3. z-index 기본 사용법

```css
.box1 {
  position: absolute;
  z-index: 1;
}

.box2 {
  position: absolute;
  z-index: 2;
}
```

위 코드에서 `.box2`는 `.box1`보다 **더 위에 렌더링**됩니다.

---

## 4. 음수, 0, 양수 모두 가능

`z-index`는 음수도 가능하며, 기본값은 `auto`입니다.

```css
.low {
  z-index: -1;
}

.normal {
  z-index: 0;
}

.high {
  z-index: 100;
}
```

`z-index: -1`을 주면 해당 요소는 **배경처럼 뒤로 밀리며**, 다른 요소에 가려질 수 있습니다.

---

## 5. stacking context (쌓임 맥락)

`z-index`는 전역적으로 작동하는 것이 아니라 **stacking context(쌓임 맥락)**이라는 독립적인 공간 내에서 작동합니다.

### stacking context를 생성하는 조건

- `position` + `z-index`가 지정된 요소
- `opacity < 1`
- `transform`, `filter`, `perspective`, `will-change`, `mix-blend-mode` 등 효과 사용
- `flex`나 `grid`의 자식 요소 중 `z-index` 지정된 항목

이러한 요소는 자신만의 z축 공간을 만들어 내부 요소끼리만 z-index를 비교합니다.

#### 예시:

```html
<div style="position: relative; z-index: 10;">
  부모 (z-index 10)
  <div style="position: absolute; z-index: 1;">
    자식 A
  </div>
</div>

<div style="position: relative; z-index: 5;">
  다른 부모 (z-index 5)
</div>
```

- 자식 A는 `z-index: 1`이지만 **부모가 만든 stacking context 내에서만** 유효합니다.
- 그래서 `z-index: 5`인 다른 부모보다 무조건 위에 배치됩니다.  
  (자식의 `z-index: 1`보다 부모의 `z-index: 10`이 더 우선)

---

## 6. z-index가 무시되는 경우

- `position: static`인 요소
- 부모 stacking context 내에서 자식의 z-index가 부모보다 커도 전체 레이어는 부모 기준
- 일부 브라우저 버그 (예: `overflow: hidden` + `z-index` 문제)

---

## ✅ 실전 예시: 모달 창 위에 배치

```css
.modal {
  position: fixed;
  z-index: 1000;
  background: white;
  width: 300px;
  height: 200px;
}

.overlay {
  position: fixed;
  z-index: 900;
  background: rgba(0, 0, 0, 0.5);
  width: 100%;
  height: 100%;
}
```

- `modal`이 `overlay` 위에 오도록 `z-index`를 더 크게 지정
- `overlay`는 배경을 어둡게 하는 반투명 레이어

---

## 💡 디버깅 팁

- **브라우저 DevTools**에서 `z-index`를 시각적으로 확인 가능
- stacking context가 예상과 다르게 동작할 경우, 부모 요소들의 `z-index`, `opacity`, `transform` 등을 의심해볼 것

---

## 정리

| 개념              | 설명                                                                 |
|-------------------|----------------------------------------------------------------------|
| `z-index`         | 요소의 쌓이는 순서를 지정 (숫자가 높을수록 위에 표시됨)            |
| `position`        | 반드시 `relative`, `absolute`, `fixed`, `sticky` 중 하나여야 함     |
| stacking context  | 요소의 z축 계산 범위를 구분하는 독립된 공간                         |
| 음수 z-index      | 요소를 뒤로 밀어 배경처럼 만들 수 있음                              |

---

레이어의 개념은 단순히 시각적 우선순위를 넘어서, 모달, 팝업, 툴팁, 드롭다운 등 **인터랙션 UI 구성에서 필수적**입니다.  
z-index와 stacking context를 제대로 이해하면 훨씬 더 유연한 CSS 구성이 가능합니다.