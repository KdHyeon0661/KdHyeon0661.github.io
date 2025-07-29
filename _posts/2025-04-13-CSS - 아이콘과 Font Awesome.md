---
layout: post
title: CSS - 아이콘과 Font Awesome
date: 2025-04-06 19:20:23 +0900
category: CSS
---
# CSS 아이콘과 Font Awesome 활용법

웹에서 아이콘은 정보를 직관적으로 전달하고 인터페이스를 풍부하게 만드는 중요한 시각 요소입니다.  
CSS만으로 간단한 아이콘을 구현할 수 있고, Font Awesome 같은 **아이콘 라이브러리**를 활용하면 수천 개의 아이콘을 손쉽게 사용할 수 있습니다.

---

## 🔹 1. CSS만으로 만드는 간단한 아이콘

CSS의 `::before` 또는 `::after`를 활용해 간단한 아이콘을 만들 수 있습니다.

### ✅ 예: 햄버거 메뉴 아이콘

```html
<div class="hamburger"></div>
```

```css
.hamburger {
  width: 30px;
  height: 2px;
  background: #333;
  position: relative;
  margin: 20px;
}

.hamburger::before,
.hamburger::after {
  content: "";
  position: absolute;
  width: 30px;
  height: 2px;
  background: #333;
  left: 0;
}

.hamburger::before {
  top: -8px;
}

.hamburger::after {
  top: 8px;
}
```

이렇게 하면 순수 CSS만으로 햄버거 아이콘이 생성됩니다.

> CSS 아이콘은 가볍고 유연하지만 복잡한 아이콘에는 한계가 있습니다.

---

## 🔹 2. Font Awesome이란?

[Font Awesome](https://fontawesome.com/)은 웹에서 사용할 수 있는 가장 인기 있는 아이콘 라이브러리 중 하나입니다.  
**벡터 기반의 폰트 아이콘**으로 제공되어 크기 변경이나 색상 변경에 매우 유리합니다.

---

## ✅ 3. Font Awesome 설치 방법

### 🔸 방법 1: CDN 사용 (가장 간단)

```html
<!-- 최신 Font Awesome (v6 기준) -->
<link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.5.0/css/all.min.css" />
```

CDN을 `<head>`에 추가한 후 바로 사용할 수 있습니다.

### 🔸 방법 2: NPM 설치 (프로젝트에 통합 시)

```bash
npm install @fortawesome/fontawesome-free
```

```scss
// SCSS 또는 CSS 파일에 import
@import '~@fortawesome/fontawesome-free/css/all.min.css';
```

---

## ✅ 4. 아이콘 사용법

### 🔹 HTML에서 사용

```html
<i class="fa-solid fa-user"></i>
<i class="fa-regular fa-heart"></i>
<i class="fa-brands fa-github"></i>
```

| 클래스           | 설명                |
|------------------|---------------------|
| `fa-solid`       | 실선 아이콘         |
| `fa-regular`     | 얇은 외곽선 스타일 |
| `fa-brands`      | 브랜드 아이콘       |
| `fa-xxx`         | 아이콘 이름         |

> 아이콘 이름은 Font Awesome 공식 사이트에서 검색 가능

---

## ✅ 5. 스타일 커스터마이징

### 아이콘 크기

```html
<i class="fa-solid fa-camera fa-2x"></i> <!-- 2배 크기 -->
<i class="fa-solid fa-camera fa-lg"></i> <!-- large 크기 -->
```

| 클래스 | 크기 비율 |
|--------|-----------|
| `fa-sm` | 0.875x   |
| `fa-lg` | 1.333x   |
| `fa-2x` ~ `fa-10x` | 2~10배 |

### 색상 변경

```css
i {
  color: #3498db;
}
```

### 회전, 반전, 애니메이션

```html
<i class="fa-solid fa-sync fa-spin"></i> <!-- 회전 애니메이션 -->
<i class="fa-solid fa-arrow-right fa-flip-horizontal"></i> <!-- 반전 -->
```

| 클래스        | 설명                    |
|---------------|-------------------------|
| `fa-spin`     | 회전 애니메이션         |
| `fa-pulse`    | 빠른 회전 애니메이션    |
| `fa-rotate-90`| 90도 회전               |
| `fa-flip-horizontal` | 수평 반전         |
| `fa-flip-vertical`   | 수직 반전         |

---

## ✅ 6. 버튼/링크에 아이콘 넣기

```html
<button>
  <i class="fa-solid fa-paper-plane"></i> 전송
</button>

<a href="https://github.com">
  <i class="fa-brands fa-github"></i> GitHub
</a>
```

- 아이콘은 텍스트와 조합되어도 잘 어울림
- 폰트 기반이라 색상, 크기, hover 스타일 변경이 자유로움

---

## ✅ 7. 접근성(A11y) 주의사항

아이콘만 있을 경우 **`aria-hidden="true"`**를 추가하거나, **`aria-label`**로 의미 전달을 해야 합니다.

```html
<i class="fa-solid fa-trash" aria-hidden="true"></i>

<!-- 또는 -->

<i class="fa-solid fa-trash" aria-label="삭제"></i>
```

---

## ✅ 8. Font Awesome 외의 아이콘 라이브러리

| 라이브러리 이름      | 특징                        |
|----------------------|-----------------------------|
| [Material Icons](https://fonts.google.com/icons) | 구글의 공식 아이콘 |
| [Bootstrap Icons](https://icons.getbootstrap.com/) | Bootstrap과 통합  |
| [Heroicons](https://heroicons.com/) | Tailwind 기반 프로젝트에 적합 |
| [Remix Icon](https://remixicon.com/) | 가볍고 깔끔한 스타일 |

---

## 📌 요약

| 방법 | 설명 |
|------|------|
| CSS 아이콘 | 순수 CSS로 단순한 UI 요소 구현 가능 |
| Font Awesome | 수천 개 아이콘을 클래스 한 줄로 사용 가능 |
| 커스터마이징 | 색상, 크기, 회전, 애니메이션 모두 가능 |
| 접근성 | 의미 있는 아이콘엔 `aria-label` 추가 권장 |

---

## 💡 마무리 팁

- **아이콘은 적절히 사용해야 의미가 있다**: 남용하면 혼란만 줌
- 폰트 기반 아이콘은 **벡터 형식**이라 고해상도에서도 깨지지 않음
- SCSS와 함께 사용 시 테마별 아이콘 스타일도 쉽게 구현 가능
