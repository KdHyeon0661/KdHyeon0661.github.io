---
layout: post
title: CSS - @keyframes를 활용한 애니메이션 만들기
date: 2025-05-02 19:20:23 +0900
category: CSS
---
# 🎞️ @keyframes를 활용한 애니메이션 만들기

CSS의 `@keyframes`를 이용하면 **시간에 따라 단계적으로 변하는 애니메이션 효과**를 직접 정의할 수 있습니다.  
`transition`이 상태 변화에 반응하는 애니메이션이라면, `@keyframes`는 **자체적으로 움직이는 독립적인 애니메이션**을 만들 수 있습니다.

---

## ✅ 1. 기본 개념

```css
@keyframes animation-name {
  0% {
    /* 시작 상태 */
  }
  100% {
    /* 끝 상태 */
  }
}
```

이렇게 만든 애니메이션은 CSS에서 아래처럼 적용합니다:

```css
.element {
  animation-name: animation-name;
  animation-duration: 2s;
}
```

---

## ✅ 2. animation 속성 목록

| 속성                  | 설명                                             |
|------------------------|--------------------------------------------------|
| `animation-name`       | 사용할 `@keyframes` 이름                         |
| `animation-duration`   | 애니메이션 재생 시간 (`2s`, `500ms` 등)         |
| `animation-timing-function` | 속도 곡선 (`ease`, `linear`, 등)            |
| `animation-delay`      | 애니메이션 시작 전 대기 시간                    |
| `animation-iteration-count` | 반복 횟수 (`infinite` 무한 반복)          |
| `animation-direction`  | 반복 시 방향 (`normal`, `reverse`, `alternate`) |
| `animation-fill-mode`  | 애니메이션 전후 상태 유지 (`forwards`, `backwards`) |
| `animation-play-state` | 재생/일시정지 (`running`, `paused`)            |

---

## ✅ 3. 간단한 예제: 박스가 왼쪽 → 오른쪽으로 이동

```css
@keyframes slide-right {
  0% {
    transform: translateX(0);
  }
  100% {
    transform: translateX(300px);
  }
}

.box {
  width: 100px;
  height: 100px;
  background: coral;
  animation-name: slide-right;
  animation-duration: 2s;
}
```

---

## ✅ 4. 여러 단계 지정

```css
@keyframes rainbow {
  0%   { background: red; }
  25%  { background: yellow; }
  50%  { background: green; }
  75%  { background: blue; }
  100% { background: purple; }
}
```

- 이렇게 하면 애니메이션이 시간에 따라 여러 색으로 바뀜

---

## ✅ 5. 축약형 문법

```css
animation: name duration timing-function delay iteration-count direction fill-mode;
```

### 예시

```css
animation: rainbow 3s ease-in-out 0s infinite alternate;
```

---

## ✅ 6. 애니메이션 반복

```css
animation-iteration-count: infinite;
```

- 무한 반복
- 숫자로 반복 횟수 지정 가능 (`2`, `5` 등)

---

## ✅ 7. 애니메이션 방향 제어

| 값          | 설명                                         |
|-------------|----------------------------------------------|
| `normal`    | 0% → 100%                                     |
| `reverse`   | 100% → 0%                                     |
| `alternate` | 0% → 100% → 0% 순으로 반복                    |
| `alternate-reverse` | 100% → 0% → 100% 순으로 반복         |

---

## ✅ 8. 시작 전/후 상태 유지: `animation-fill-mode`

```css
animation-fill-mode: forwards;
```

- 애니메이션 끝난 후 마지막 상태 유지

```css
animation-fill-mode: backwards;
```

- delay 동안 시작 상태 유지

---

## ✅ 9. 실전 예제: 점프하는 버튼

```html
<button class="bounce">Jump!</button>
```

```css
.bounce {
  animation: jump 0.5s ease-in-out infinite alternate;
}

@keyframes jump {
  from {
    transform: translateY(0);
  }
  to {
    transform: translateY(-20px);
  }
}
```

- 버튼이 위로 튀는 효과
- `alternate`를 사용해 위아래 반복

---

## ✅ 10. 애니메이션 일시 정지 / 재생

```css
.element:hover {
  animation-play-state: paused;
}
```

- 마우스를 올리면 애니메이션 정지
- 기본값은 `running`

---

## ✅ 11. animation과 transition의 차이

| 항목          | transition                       | animation (`@keyframes`)                   |
|---------------|----------------------------------|--------------------------------------------|
| 트리거        | 사용자 상호작용 (hover 등)       | 자동 시작 or 제어 가능                     |
| 단순성        | 간단한 변화에 적합                | 복잡한 단계적 변화 가능                    |
| 반복          | 반복 불가                         | 반복, 방향 제어 등 가능                    |
| 제어          | 상태 간 변화만                   | 시간 기반 동작 가능                        |

---

## ✅ 12. 실전: 로딩 스피너

```html
<div class="loader"></div>
```

```css
.loader {
  width: 40px;
  height: 40px;
  border: 5px solid lightgray;
  border-top-color: royalblue;
  border-radius: 50%;
  animation: spin 1s linear infinite;
}

@keyframes spin {
  100% {
    transform: rotate(360deg);
  }
}
```

- 로딩 애니메이션처럼 계속 회전

---

## 📌 요약 정리

| 속성                     | 설명                                     |
|--------------------------|------------------------------------------|
| `@keyframes`             | 애니메이션 단계 정의                    |
| `animation-name`         | 적용할 애니메이션 이름                  |
| `animation-duration`     | 재생 시간                                |
| `animation-iteration-count` | 반복 횟수                           |
| `animation-direction`    | 반복 방향 설정 (`alternate` 등)        |
| `animation-fill-mode`    | 애니메이션 전후 상태 유지               |

---

## 💡 마무리 팁

- `@keyframes`는 **복잡한 애니메이션 구현의 핵심**입니다.
- `transition`은 상태 변화용, `animation`은 **자체 타이머로 실행되는 효과**에 적합합니다.
- 성능을 고려해 애니메이션에는 **GPU 가속** 가능한 `transform`, `opacity` 속성을 주로 사용하는 것이 좋습니다.

---

🔗 참고 자료
- [MDN: @keyframes](https://developer.mozilla.org/ko/docs/Web/CSS/@keyframes)
- [MDN: animation](https://developer.mozilla.org/ko/docs/Web/CSS/animation)
- [CSS Tricks: Animations Guide](https://css-tricks.com/snippets/css/keyframe-animation-syntax/)