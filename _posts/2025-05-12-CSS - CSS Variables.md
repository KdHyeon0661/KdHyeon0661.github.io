---
layout: post
title: CSS - CSS Variables
date: 2025-05-12 19:20:23 +0900
category: CSS
---
# 🎨 CSS 커스텀 속성 (CSS Variables) 완전 정리

CSS 커스텀 속성(CSS Variables)은 **재사용 가능한 값(예: 색상, 크기, 폰트 등)**을 변수처럼 지정해  
**유지보수와 테마 관리에 매우 유리한 기능**입니다.

---

## ✅ 기본 문법

```css
:root {
  --main-color: #3498db;
  --font-size: 1.2rem;
}
```

```css
button {
  color: var(--main-color);
  font-size: var(--font-size);
}
```

| 구문         | 의미 |
|--------------|------|
| `--변수명`    | 변수 선언 (접두사 `--` 필수) |
| `var()`      | 변수 값 호출 |

---

## ✅ 어디에 선언하나?

- `:root` → HTML 문서 전체에 적용되는 글로벌 범위
- 일반 선택자(`.box`, `section` 등) 안에서도 선언 가능 (로컬 범위)

```css
/* 전역 변수 */
:root {
  --text-color: #333;
}

/* 로컬 변수 (해당 요소 내부에서만 유효) */
.card {
  --text-color: red;
  color: var(--text-color); /* red */
}
```

> ⚠ 로컬 변수가 우선순위에서 전역 변수보다 높다.

---

## ✅ 기본값 제공 (fallback)

```css
color: var(--undefined-var, black);
```

- 변수가 정의되어 있지 않은 경우, **두 번째 인자**가 기본값으로 사용됨

---

## ✅ JavaScript와 연동

```js
// 변수 값 읽기
const root = document.documentElement;
const color = getComputedStyle(root).getPropertyValue('--main-color');

// 변수 값 변경
root.style.setProperty('--main-color', '#e74c3c');
```

- **동적 테마 변경** 등에 유용
- ex) 다크모드 전환

---

## ✅ 실제 예제 1: 색상 테마 관리

```css
:root {
  --primary: #1abc9c;
  --secondary: #2c3e50;
  --danger: #e74c3c;
}

.btn {
  background-color: var(--primary);
}

.btn.danger {
  background-color: var(--danger);
}
```

→ 색상만 바꾸면 전체 UI 색상 일괄 변경 가능

---

## ✅ 실제 예제 2: 반응형 크기 관리

```css
:root {
  --gap: 1rem;
  --font-size: 1rem;
}

@media (min-width: 768px) {
  :root {
    --gap: 2rem;
    --font-size: 1.25rem;
  }
}
```

→ 반응형 값도 변수로 제어하면 유지보수가 편리해짐

---

## ✅ 실제 예제 3: 다크 모드 구현

```css
:root {
  --bg-color: #ffffff;
  --text-color: #000000;
}

[data-theme="dark"] {
  --bg-color: #1e1e1e;
  --text-color: #f1f1f1;
}

body {
  background-color: var(--bg-color);
  color: var(--text-color);
}
```

```js
// JS에서 테마 전환
document.documentElement.setAttribute('data-theme', 'dark');
```

---

## ✅ 커스텀 속성 vs SASS 변수

| 항목              | CSS 커스텀 속성      | SCSS/SASS 변수          |
|-------------------|----------------------|--------------------------|
| 브라우저 지원     | 최신 브라우저만 지원 | 컴파일 시 사용됨 (IE 포함) |
| 런타임 변경 가능   | 가능 (JS로 제어 가능) | 불가능                    |
| 계층적 범위        | 가능 (상속됨)         | 전처리기에서만 적용       |
| 조건부 설정        | 가능 (`@media`, `:root`) | 제한적 (`@if` 등 필요)   |

> 🔧 **CSS 변수는 런타임에서 동적으로 적용되며 JS 제어도 가능**, 반면 SCSS는 컴파일 타임에서만 유효

---

## ✅ 주의할 점

- CSS 변수는 **Cascading(상속) 원칙을 따름**
- 사용 전 반드시 선언되어 있어야 함
- 구형 브라우저 (IE11 이하)에서는 **지원되지 않음**

---

## ✅ 지원 브라우저

| 브라우저     | 지원 여부 |
|--------------|-----------|
| Chrome       | O         |
| Firefox      | O         |
| Edge         | O         |
| Safari       | O         |
| IE11 이하     | ❌ 미지원 |

→ 구형 브라우저 대응이 필요하다면 **SASS 변수로 병행하거나 폴백 스타일 추가 필요**

---

## 📌 요약 정리

| 개념          | 설명 |
|---------------|------|
| 선언 방법      | `--변수명: 값;` 형태 |
| 사용 방법      | `var(--변수명)` |
| 적용 위치      | `:root` or 특정 요소 (전역/지역 범위) |
| 사용 장점      | 재사용성, 테마 구성, 반응형 대응 용이 |
| JS 연동 가능 여부 | ✅ (동적 테마 변경 등 가능) |
| 브라우저 지원   | 최신 브라우저 (IE 제외) |

---

## 🔗 참고 링크

- [MDN - CSS Custom Properties](https://developer.mozilla.org/en-US/docs/Web/CSS/Using_CSS_custom_properties)
- [CSS-Tricks - A Complete Guide to Custom Properties](https://css-tricks.com/a-complete-guide-to-custom-properties/)
