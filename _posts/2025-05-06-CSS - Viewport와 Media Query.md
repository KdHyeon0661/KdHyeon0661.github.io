---
layout: post
title: CSS - Viewport와 Media Query
date: 2025-05-06 19:20:23 +0900
category: CSS
---
# 🖥️ Viewport와 Media Query 완전 정리

웹의 반응형 디자인을 구현하려면 **Viewport**와 **Media Query**의 개념을 정확히 이해해야 합니다.  
이 두 요소는 **디바이스 화면 크기 및 특성에 맞춰 콘텐츠를 동적으로 조절**하는 핵심 도구입니다.

---

## ✅ Viewport란?

`Viewport`는 사용자가 웹 페이지를 볼 수 있는 **화면의 가시 영역(브라우저 창)**을 말합니다.  
특히 모바일 환경에서는 뷰포트 설정이 필수입니다.

### 📌 HTML에서의 Viewport 설정

```html
<meta name="viewport" content="width=device-width, initial-scale=1.0">
```

| 속성              | 의미 |
|-------------------|------|
| `width=device-width` | 뷰포트의 너비를 디바이스의 실제 화면 너비로 설정 |
| `initial-scale=1.0`  | 초기 확대/축소 배율 (1은 100%) |

### ❗ 중요성

- 이 설정이 없으면 모바일 브라우저가 웹사이트를 **기본적으로 축소해서 보여주기 때문에** 디자인이 깨짐
- **반응형 웹을 만들 때 반드시 포함해야 하는 태그**

---

## ✅ Media Query란?

Media Query는 **CSS에서 화면의 크기나 특성에 따라 다른 스타일을 적용**할 수 있게 해주는 기능입니다.

### 📌 기본 문법

```css
@media (조건) {
  /* 조건이 참일 때 적용될 CSS */
}
```

### 예시: 768px 이하에서는 `.sidebar`를 숨기기

```css
@media (max-width: 768px) {
  .sidebar {
    display: none;
  }
}
```

---

## ✅ Media Query에서 자주 쓰는 조건

| 조건              | 설명 |
|-------------------|------|
| `min-width`       | 최소 너비 이상일 때 적용 |
| `max-width`       | 최대 너비 이하일 때 적용 |
| `min-height`      | 최소 높이 이상일 때 적용 |
| `max-height`      | 최대 높이 이하일 때 적용 |
| `orientation`     | `portrait`(세로) 또는 `landscape`(가로) |
| `hover`, `pointer`| 디바이스 입력 방식 (마우스/터치 여부) |

---

## ✅ 예제: 화면 크기별 레이아웃 변경

```css
/* 기본 데스크탑 레이아웃 */
.container {
  display: flex;
  flex-direction: row;
}

/* 태블릿 이하에서는 세로 정렬 */
@media (max-width: 1024px) {
  .container {
    flex-direction: column;
  }
}
```

---

## ✅ 예제: 모바일 퍼스트 전략 (권장)

```css
/* 모바일 기준 기본 스타일 */
.button {
  font-size: 16px;
}

/* 큰 화면에서 폰트 키움 */
@media (min-width: 768px) {
  .button {
    font-size: 20px;
  }
}
```

> Mobile First란?  
> - 가장 작은 화면(모바일)을 기준으로 스타일링하고  
> - `min-width` 조건으로 점차 확장

---

## ✅ Viewport + Media Query 조합 흐름

1. `viewport` 메타 태그로 디바이스 뷰포트 설정
2. `@media`를 통해 크기나 디바이스 특성에 따른 스타일 작성
3. 이를 통해 **하나의 HTML/CSS 코드로 모든 기기 대응 가능**

---

## ✅ 실전 활용 예: 반응형 네비게이션 메뉴

```css
.nav-menu {
  display: flex;
}

@media (max-width: 768px) {
  .nav-menu {
    display: none;
  }

  .hamburger {
    display: block;
  }
}
```

```html
<meta name="viewport" content="width=device-width, initial-scale=1.0">
```

> - 큰 화면에서는 `.nav-menu` 보이고,  
> - 작은 화면에서는 `.hamburger` 메뉴 아이콘만 보여짐

---

## ✅ 고급 예시: 방향(orientation) 반응

```css
@media (orientation: portrait) {
  body {
    background-color: lightblue;
  }
}

@media (orientation: landscape) {
  body {
    background-color: lightgreen;
  }
}
```

- 화면이 가로냐 세로냐에 따라 스타일 변경

---

## ✅ 다양한 Media Feature 조합

```css
@media (min-width: 600px) and (orientation: portrait) {
  /* 조건이 둘 다 참일 경우 */
}
```

---

## 📌 요약 정리

| 항목              | 설명 |
|-------------------|------|
| Viewport 메타태그 | 모바일에서 화면 너비, 확대 비율 설정 |
| Media Query       | CSS 조건 분기로 다양한 환경 대응 |
| Mobile First      | 작은 화면을 기준으로 점차 확장 |
| 조합 예시         | `@media (min-width: 768px)` 등으로 유연하게 레이아웃 제어 |

---

## 🔗 참고 자료

- [MDN: Using media queries](https://developer.mozilla.org/en-US/docs/Web/CSS/Media_Queries/Using_media_queries)
- [MDN: Viewport](https://developer.mozilla.org/en-US/docs/Web/HTML/Viewport_meta_tag)
- [CSS Tricks: Media Query Guide](https://css-tricks.com/a-complete-guide-to-css-media-queries/)