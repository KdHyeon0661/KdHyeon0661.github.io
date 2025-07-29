---
layout: post
title: CSS - Media Query
date: 2025-04-21 20:20:23 +0900
category: CSS
---
# 📱 반응형 디자인 기초: Media Query 사용법

**반응형 웹 디자인 (Responsive Web Design)**이란 다양한 디바이스(모바일, 태블릿, 데스크탑)에 따라 **레이아웃과 스타일을 자동으로 조정**하는 기술입니다.  
이를 가능하게 해주는 핵심 기능이 바로 **Media Query**입니다.

---

## ✅ 1. Media Query란?

CSS에서 **디바이스의 화면 크기, 방향, 해상도** 등을 조건으로 삼아  
특정 스타일을 **조건부로 적용**할 수 있는 기능입니다.

```css
@media (조건) {
  /* 조건을 만족할 때 적용될 CSS */
}
```

---

## ✅ 2. 기본 문법

```css
@media screen and (max-width: 768px) {
  body {
    background-color: lightblue;
  }
}
```

### 구성 요소

| 구성 요소 | 설명 |
|-----------|------|
| `@media`  | 미디어 쿼리 시작 |
| `screen`  | 미디어 타입 (화면 출력용) |
| `and`     | 조건 연결 |
| `(max-width: 768px)` | 조건 - 최대 너비가 768px일 때만 적용 |

---

## ✅ 3. 주요 조건 (Media Features)

| 조건                | 설명                                  |
|---------------------|----------------------------------------|
| `max-width`         | 최대 너비 (이 값 이하일 때 적용)       |
| `min-width`         | 최소 너비 (이 값 이상일 때 적용)       |
| `max-height`        | 최대 높이                             |
| `orientation`       | `portrait` (세로), `landscape` (가로) |
| `hover`, `pointer`  | 입력 방식 (터치, 마우스 등)            |
| `resolution`        | 해상도 (dpi, dppx 등)                  |

---

## ✅ 4. 실전 예제

### 📱 모바일 레이아웃 적용

```css
@media (max-width: 600px) {
  .nav {
    flex-direction: column;
  }

  .sidebar {
    display: none;
  }
}
```

- 화면 너비가 600px 이하인 경우:
  - 내비게이션을 세로로 배치
  - 사이드바를 숨김

---

### 💻 데스크탑 전용 스타일

```css
@media (min-width: 1024px) {
  .container {
    max-width: 960px;
    margin: 0 auto;
  }
}
```

- 1024px 이상 화면에만 중앙 정렬과 너비 제한 적용

---

## ✅ 5. 여러 조건 결합하기

### `and`로 여러 조건 연결

```css
@media screen and (min-width: 600px) and (max-width: 1024px) {
  body {
    font-size: 18px;
  }
}
```

### `,` (쉼표)로 조건 그룹화 (OR 조건)

```css
@media (max-width: 600px), (orientation: portrait) {
  .header {
    font-size: 1.2rem;
  }
}
```

> 여러 조건 중 하나라도 만족하면 적용됨

---

## ✅ 6. 반응형 디자인 구현 전략

### 모바일 퍼스트 (권장 방식)

- 기본 스타일을 **모바일 기준**으로 작성
- 이후 `min-width` 조건을 통해 **큰 화면에서만 오버라이드**

```css
/* 모바일 기본 */
.button {
  font-size: 14px;
}

/* 태블릿 이상 */
@media (min-width: 768px) {
  .button {
    font-size: 16px;
  }
}

/* 데스크탑 이상 */
@media (min-width: 1024px) {
  .button {
    font-size: 18px;
  }
}
```

### 데스크탑 퍼스트

- 반대로 먼저 데스크탑 기준으로 작성 후, `max-width`로 줄여나가는 방식

---

## ✅ 7. 뷰포트 설정 (`<meta>` 태그)

반응형 디자인은 HTML에서 **뷰포트 설정**도 필수입니다.

```html
<meta name="viewport" content="width=device-width, initial-scale=1.0" />
```

- `width=device-width`: 디바이스 화면 너비에 맞춤
- `initial-scale=1.0`: 초기 줌 배율

> 이 설정이 없으면 반응형 CSS가 제대로 작동하지 않습니다!

---

## ✅ 8. 실전 예제: 카드 그리드 반응형

```css
.card-grid {
  display: grid;
  grid-template-columns: 1fr;
  gap: 1rem;
}

@media (min-width: 600px) {
  .card-grid {
    grid-template-columns: repeat(2, 1fr);
  }
}

@media (min-width: 900px) {
  .card-grid {
    grid-template-columns: repeat(3, 1fr);
  }
}
```

- 모바일: 1열
- 태블릿: 2열
- 데스크탑: 3열

---

## ✅ 9. 참고 단위 정리

| 단위    | 설명                          |
|---------|-------------------------------|
| `px`    | 고정 픽셀 (절대 단위)          |
| `em` / `rem` | 폰트 크기 기준 상대 단위     |
| `%`     | 부모 요소 기준 상대 단위       |
| `vw` / `vh` | 뷰포트 기준 단위 (너비/높이) |

---

## 📌 요약 정리

| 항목              | 설명                                      |
|-------------------|-------------------------------------------|
| `@media`          | 미디어 쿼리 시작                          |
| `min-width`       | 해당 너비 이상일 때만 적용                 |
| `max-width`       | 해당 너비 이하일 때만 적용                 |
| `and`             | 여러 조건 AND 연결                         |
| `,` (쉼표)        | 여러 조건 중 하나라도 만족하면 적용 (OR)   |
| 모바일 퍼스트     | 기본 스타일은 모바일, 큰 화면에만 오버라이드 |
| `<meta viewport>` | 모바일 디바이스 대응을 위한 필수 설정       |

---

## 💡 마무리

- `media query`는 반응형 디자인의 **핵심 도구**입니다.
- 다양한 해상도에 대응하려면 `min-width` 기반의 **모바일 퍼스트** 전략을 추천합니다.
- Flexbox, Grid와 함께 사용하면 **강력하고 유연한 반응형 UI**를 만들 수 있습니다.

---

🔗 참고 링크
- [MDN: Media Queries](https://developer.mozilla.org/ko/docs/Web/CSS/Media_Queries)
- [CSS Tricks: Media Query Examples](https://css-tricks.com/css-media-queries/)
- [Responsive Web Design Basics (Google)](https://web.dev/responsive-web-design-basics/)