---
layout: post
title: CSS - calc(), clamp(), min(), max()
date: 2025-05-12 20:20:23 +0900
category: CSS
---
# 📐 CSS 함수 정리: `calc()`, `clamp()`, `min()`, `max()` 활용법

이 함수들은 **유연한 레이아웃, 반응형 디자인, 유동적인 단위 조합**에 매우 강력하게 사용됩니다.  
복잡한 미디어 쿼리 없이도 다양한 디바이스에 대응할 수 있는 **현대 CSS의 핵심 도구**입니다.

---

## ✅ 1. `calc()`: CSS 값 계산

### 📌 문법

```css
width: calc(100% - 200px);
```

- 다양한 단위를 **조합**할 수 있음
- 연산자: `+`, `-`, `*`, `/`
- 단위 혼합 가능: `% + px`, `em - rem` 등

### ✅ 예제

```css
.container {
  width: calc(100% - 2rem);
  height: calc(50vh + 100px);
  font-size: calc(1rem + 0.5vw);
}
```

### ❗ 주의사항

- 연산 기호 양쪽에 **공백 필수**
  ```css
  /* 잘못된 예 */
  width: calc(100%-2rem); /* ❌ */
  ```

---

## ✅ 2. `clamp()`: 반응형 제한 범위 설정

### 📌 문법

```css
font-size: clamp(1rem, 2.5vw, 2rem);
```

| 순서 | 의미                        |
|------|-----------------------------|
| min  | 최소값 (작아지지 않음)      |
| ideal | 유동값 (브라우저 너비 기반) |
| max  | 최대값 (커지지 않음)        |

### ✅ 예제: 반응형 타이틀

```css
h1 {
  font-size: clamp(1.5rem, 4vw, 3rem);
}
```

- 화면이 작을 땐 `1.5rem`, 클 땐 `3rem`, 그 사이에선 `vw` 기반으로 유동 적용
- **미디어 쿼리 없이 반응형 구현 가능**

---

## ✅ 3. `min()`: 최소값을 선택

### 📌 문법

```css
width: min(100%, 800px);
```

- 둘 중 **작은 값**을 사용
- 유용한 경우: 최대 너비 제한

### ✅ 예제

```css
.container {
  width: min(100%, 1280px);
}
```

→ 1280px 이상으로 커지지 않도록 제한

---

## ✅ 4. `max()`: 최대값을 선택

### 📌 문법

```css
padding: max(1rem, 5%);
```

- 둘 중 **큰 값**을 사용
- 유용한 경우: 최소 패딩 확보

### ✅ 예제

```css
.section {
  padding-left: max(1rem, 3vw);
}
```

→ 화면이 너무 작아도 **최소한의 공간 확보**

---

## ✅ 실전 예제 1: 반응형 카드 레이아웃

```css
.card {
  width: clamp(300px, 50%, 500px);
  padding: clamp(1rem, 2vw, 2rem);
}
```

- 너비는 300~500px 사이에서 화면 크기에 따라 유동적
- 패딩도 자동으로 크기 조절

---

## ✅ 실전 예제 2: `min()` + `max()` 조합

```css
.container {
  width: min(100%, max(640px, 60%));
}
```

- 화면이 매우 작을 때는 `100%`
- 중간 이상에서는 `640px` 또는 `60%` 중 더 큰 값 사용

---

## ✅ 실전 예제 3: `calc()`으로 요소 정렬

```css
.sidebar {
  width: 250px;
}
.main {
  width: calc(100% - 250px);
}
```

- 고정 사이드바를 제외한 **가변 콘텐츠 영역 확보**

---

## 📊 함수별 비교 정리

| 함수       | 기능                     | 활용 예                          |
|------------|--------------------------|----------------------------------|
| `calc()`   | 값 계산 (단위 혼합 가능) | `width: calc(100% - 2rem)`       |
| `clamp()`  | 최소~최대 사이 유동 적용 | `font-size: clamp(1rem, 3vw, 2rem)` |
| `min()`    | 둘 중 작은 값 선택        | `width: min(100%, 1200px)`       |
| `max()`    | 둘 중 큰 값 선택         | `padding: max(1rem, 5%)`         |

---

## ✅ 브라우저 지원

| 브라우저     | calc | clamp | min/max |
|--------------|------|-------|---------|
| Chrome       | ✅   | ✅    | ✅      |
| Firefox      | ✅   | ✅    | ✅      |
| Edge         | ✅   | ✅    | ✅      |
| Safari       | ✅   | ✅    | ✅      |
| IE11 이하     | ✅   | ❌    | ❌      |

> ❗ `clamp()`, `min()`, `max()`는 **IE11에서 미지원** → Polyfill 불가 → 적절한 fallback 고려 필요

---

## 📌 팁: `clamp()`를 미디어 쿼리 대체용으로 활용

```css
:root {
  --fluid-font-size: clamp(1rem, 2vw, 2rem);
}

body {
  font-size: var(--fluid-font-size);
}
```

- 미디어 쿼리 없이도 가독성 유지 가능한 유동 폰트

---

## 🔗 참고 링크

- [MDN - calc()](https://developer.mozilla.org/en-US/docs/Web/CSS/calc)
- [MDN - clamp()](https://developer.mozilla.org/en-US/docs/Web/CSS/clamp)
- [CSS Tricks - Fluid Typography](https://css-tricks.com/using-calc-to-build-a-responsive-grid/)
