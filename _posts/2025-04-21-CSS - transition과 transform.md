---
layout: post
title: CSS - transition과 transform
date: 2025-04-21 21:20:23 +0900
category: CSS
---
# ✨ CSS `transition`과 `transform` 기초 정리

웹 페이지를 더욱 **동적이고 인터랙티브**하게 만들어주는 CSS 속성이 바로  
`transition`과 `transform`입니다.  
이 두 속성을 조합하면 **부드러운 애니메이션 효과**를 쉽게 구현할 수 있습니다.

---

## ✅ 1. `transition`이란?

특정 CSS 속성 값이 **변화할 때 그 과정을 애니메이션처럼 부드럽게 처리**합니다.

```css
.box {
  transition: all 0.3s ease;
}
```

### 주요 속성

| 속성             | 설명                                  |
|------------------|-----------------------------------------|
| `transition-property` | 애니메이션 대상 속성 (`width`, `opacity`, 등) |
| `transition-duration` | 변화에 걸리는 시간 (`0.5s`, `200ms` 등) |
| `transition-timing-function` | 변화의 속도 곡선 (`ease`, `linear`, 등) |
| `transition-delay`    | 시작까지의 지연 시간                    |

### 축약형 문법

```css
transition: [property] [duration] [timing-function] [delay];
```

```css
transition: background-color 0.3s ease-in-out 0s;
```

---

### 예제: hover 시 배경색 전환

```html
<div class="btn">Hover Me</div>
```

```css
.btn {
  background: lightblue;
  transition: background-color 0.3s;
}
.btn:hover {
  background: steelblue;
}
```

---

## ✅ 2. `transform`이란?

요소의 **형태나 위치를 시각적으로 변경**하는 속성입니다.  
이 속성은 CSS 레이아웃에는 영향을 주지 않고 **렌더링 상의 위치만** 바꿉니다.

### 주요 함수

| 함수              | 설명                              |
|-------------------|-----------------------------------|
| `translate(x, y)` | 위치 이동                          |
| `scale(x, y)`     | 크기 조정                          |
| `rotate(deg)`     | 회전 (시계 방향)                  |
| `skew(x, y)`      | 기울이기                           |
| `matrix()`        | 모든 변형을 행렬로 설정 (고급)     |

---

### 예제: 요소 이동 & 회전

```css
.box {
  transform: translateX(100px) rotate(45deg);
}
```

- 요소가 오른쪽으로 100px 이동하고, 45도 회전됨

---

## ✅ 3. `transition` + `transform` 함께 사용하기

```html
<div class="card">🔥</div>
```

```css
.card {
  width: 100px;
  height: 100px;
  background: tomato;
  color: white;
  font-size: 2rem;
  text-align: center;
  line-height: 100px;
  transition: transform 0.3s ease;
}

.card:hover {
  transform: scale(1.2) rotate(5deg);
}
```

- hover 시 **확대 + 회전 효과**가 부드럽게 적용됨

---

## ✅ 4. `transform-origin`

변형 기준점을 설정하는 속성 (기본값: `center center`)

```css
transform-origin: left top;
```

### 예제

```css
.box {
  transform: rotate(45deg);
  transform-origin: bottom right;
}
```

- 회전 기준점이 오른쪽 아래가 됨

---

## ✅ 5. `transition-timing-function` 상세

| 값             | 설명                                                 |
|----------------|------------------------------------------------------|
| `ease`         | 기본값, 천천히 시작 → 빠름 → 천천히 끝남              |
| `linear`       | 일정한 속도                                           |
| `ease-in`      | 천천히 시작                                           |
| `ease-out`     | 천천히 끝남                                           |
| `ease-in-out`  | 천천히 시작 + 천천히 끝남                             |
| `cubic-bezier(x1, y1, x2, y2)` | 사용자 정의 곡선 (ex: `cubic-bezier(.17,.67,.83,.67)`) |

---

## ✅ 6. 여러 속성에 transition 적용

```css
.box {
  transition: transform 0.3s ease, background-color 0.2s linear;
}
```

- `transform`은 ease로 0.3초
- `background-color`는 linear로 0.2초

---

## ✅ 7. transform의 순서 중요성

```css
transform: rotate(45deg) scale(1.5);
```

- 회전 후 확대

```css
transform: scale(1.5) rotate(45deg);
```

- 확대 후 회전  
→ 결과가 다르게 보일 수 있음!

---

## ✅ 8. 실전 예제: 버튼 인터랙션

```html
<button class="btn">Click Me</button>
```

```css
.btn {
  background: dodgerblue;
  color: white;
  border: none;
  padding: 1rem 2rem;
  font-size: 1rem;
  transition: transform 0.2s ease, box-shadow 0.2s;
}

.btn:hover {
  transform: scale(1.05);
  box-shadow: 0 8px 16px rgba(0,0,0,0.2);
}
```

---

## ✅ 9. `will-change` 속성으로 성능 최적화

```css
.box {
  will-change: transform;
}
```

- 브라우저에 미리 알림 → 렌더링 최적화 가능  
단, **과도한 사용은 오히려 성능 저하** 유발

---

## 📌 요약 정리

| 속성         | 역할                                  |
|--------------|----------------------------------------|
| `transition` | 속성 변화에 애니메이션 효과 부여       |
| `transform`  | 요소의 위치, 회전, 크기 등을 시각적으로 조정 |
| `transform-origin` | 변형 기준점 조정                 |
| `timing-function` | 변화 속도 조절 (ease, linear 등) |

---

## 💡 마무리 팁

- `transition`은 상태 변화에 애니메이션을 주는 속성  
- `transform`은 위치/회전/크기 등 요소 자체의 형태를 변화  
- 이 둘을 함께 사용하면 클릭/호버/스크롤 인터랙션이 **훨씬 부드러워짐**
- `transition: all`은 성능에 영향을 줄 수 있으므로 **필요한 속성만 명시**하는 것이 좋음

---

🔗 참고 자료
- [MDN: transition](https://developer.mozilla.org/ko/docs/Web/CSS/transition)
- [MDN: transform](https://developer.mozilla.org/ko/docs/Web/CSS/transform)
- [CSS Tricks: Transition Guide](https://css-tricks.com/almanac/properties/t/transition/)