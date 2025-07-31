---
layout: post
title: CSS - Dark Mode CSS
date: 2025-05-16 21:20:23 +0900
category: CSS
---
# 🌙 Dark Mode CSS 구현 방법 완전 정리

**Dark Mode(다크 모드)**는 눈의 피로를 줄이고, 배터리 소모를 줄이며, 심미적으로도 사용자에게 선택권을 주는 디자인 모드입니다.  
CSS만으로도 사용자의 시스템 설정이나 버튼 클릭에 따라 다크 모드를 손쉽게 구현할 수 있습니다.

이 글에서는 **기본 개념부터 구현 방식 3가지**, 실무 팁까지 단계별로 설명합니다.

---

## 🧩 Dark Mode 구현 방법 요약

| 방법                            | 설명                                               | 지원 환경       |
|---------------------------------|----------------------------------------------------|------------------|
| 1. `prefers-color-scheme` 사용 | 사용자의 운영체제/브라우저 설정 자동 반영          | 최신 브라우저 ✅ |
| 2. 클래스 기반 토글 방식        | 버튼 클릭 시 class로 테마 전환                    | 제어 유연함 ✅   |
| 3. CSS 변수 + JS 조합          | 변수 기반 테마 관리 + localStorage 기억 기능 가능 | 확장성 우수 ✅   |

---

## ✅ 1. 시스템 다크모드 감지 (`prefers-color-scheme`)

### 📌 기본 예제

```css
/* 기본 (라이트) 테마 */
body {
  background: #ffffff;
  color: #000000;
}

/* 사용자가 다크모드를 선호할 경우 */
@media (prefers-color-scheme: dark) {
  body {
    background: #121212;
    color: #ffffff;
  }
}
```

### 🧪 지원 브라우저

| 브라우저     | 지원 여부 |
|--------------|-----------|
| Chrome       | ✅         |
| Firefox      | ✅         |
| Safari       | ✅         |
| Edge         | ✅         |
| IE11 이하     | ❌         |

> 자동으로 시스템 설정 감지되므로, **JS 없이도 적용 가능**  
> 단, **사용자 전환 버튼은 구현 불가**

---

## ✅ 2. 클래스 기반 토글 방식 (JS 사용)

### 📌 HTML

```html
<body class="light">
  <button id="toggle-theme">🌓 테마 전환</button>
</body>
```

### 📌 CSS

```css
body.light {
  background: #ffffff;
  color: #000000;
}

body.dark {
  background: #121212;
  color: #ffffff;
}
```

### 📌 JS

```js
const button = document.getElementById('toggle-theme');
const body = document.body;

button.addEventListener('click', () => {
  body.classList.toggle('dark');
  body.classList.toggle('light');
});
```

> 사용자 전환 버튼을 만들고 싶을 때 사용  
> `localStorage`로 사용자 선택 기억도 가능

---

## ✅ 3. CSS 변수 기반 테마 (추천)

### 📌 CSS 변수 정의

```css
:root {
  --bg-color: #ffffff;
  --text-color: #000000;
}

[data-theme="dark"] {
  --bg-color: #121212;
  --text-color: #ffffff;
}

body {
  background: var(--bg-color);
  color: var(--text-color);
}
```

### 📌 HTML

```html
<body data-theme="light">
  <button id="toggle-theme">🌗 Toggle Theme</button>
</body>
```

### 📌 JS로 테마 토글

```js
const toggle = document.getElementById("toggle-theme");
const html = document.documentElement;

toggle.addEventListener("click", () => {
  const current = html.getAttribute("data-theme");
  const next = current === "dark" ? "light" : "dark";
  html.setAttribute("data-theme", next);
  localStorage.setItem("theme", next);
});

// 페이지 로드 시 적용
window.addEventListener("DOMContentLoaded", () => {
  const saved = localStorage.getItem("theme");
  if (saved) html.setAttribute("data-theme", saved);
});
```

> 💡 CSS 변수 기반 방식은 **확장성과 유지보수에 매우 유리**
- 모든 색상, 그림자, 테두리를 변수로 관리 가능
- dark, light 외 다른 테마도 쉽게 추가 가능

---

## 💡 테마별 변수 구조 예시

```css
:root {
  --color-bg: #fff;
  --color-text: #111;
  --color-border: #ddd;
}

[data-theme="dark"] {
  --color-bg: #1e1e1e;
  --color-text: #f1f1f1;
  --color-border: #444;
}
```

```css
.card {
  background: var(--color-bg);
  color: var(--color-text);
  border: 1px solid var(--color-border);
}
```

---

## 🛡 다크 모드 관련 실전 팁

| 항목               | 팁 |
|--------------------|----|
| 배경색만 변경 ❌     | 텍스트/아이콘/링크/보더 등 전체 요소 고려 |
| 이미지 반전        | `filter: invert()` 혹은 별도 다크 이미지 사용 |
| svg나 icon 색상     | `currentColor` 활용하면 색상 변수에 연동 |
| localStorage 저장   | 사용자 선택 기억, 페이지 새로고침 시 적용 |
| transition 효과     | 부드러운 전환 추가 시 UX 향상 |

```css
body {
  transition: background 0.3s ease, color 0.3s ease;
}
```

---

## 🔍 Dark Mode 체크 사이트

- [Darkmode Design Gallery](https://darkmode.design/)
- [prefers-color-scheme 테스트](https://whatpwacando.today/themes/)
- [CSS Variables + Dark Mode 예제](https://css-tricks.com/a-complete-guide-to-dark-mode-on-the-web/)

---

## ✅ 요약

| 방법                     | 장점                                | 단점                          |
|--------------------------|-------------------------------------|-------------------------------|
| `prefers-color-scheme`   | 브라우저 설정 자동 인식            | 사용자 수동 전환 불가         |
| 클래스 기반              | 사용자 제어 가능, 구현 간단         | 변수 관리 어려움              |
| CSS 변수 기반            | 유지보수 쉬움, 다크/라이트 확장 쉬움 | JS와 CSS 연동 필요             |

---

## 🔗 참고 링크

- [MDN: prefers-color-scheme](https://developer.mozilla.org/en-US/docs/Web/CSS/@media/prefers-color-scheme)
- [CSS Tricks: Dark Mode](https://css-tricks.com/dark-modes-with-css/)
- [Google Dev Guide - Dark Theme](https://web.dev/prefers-color-scheme/)