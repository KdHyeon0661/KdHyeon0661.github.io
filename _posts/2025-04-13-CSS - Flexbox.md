---
layout: post
title: CSS - Flexbox
date: 2025-04-06 21:20:23 +0900
category: CSS
---
# 💪 Flexbox 완전 가이드 (Flexbox Complete Guide)

Flexbox(Flexible Box)는 1차원(가로 or 세로) 레이아웃을 손쉽게 다룰 수 있게 해주는 CSS 레이아웃 시스템입니다.  
정렬, 간격, 축 방향 배치 등 복잡한 레이아웃도 **적은 코드로 직관적으로** 처리할 수 있어 현대 웹 UI의 핵심 기술입니다.

---

## ✅ Flexbox의 핵심 개념

Flexbox는 두 가지 개념으로 구성됩니다:

- **Flex Container (부모 요소)**: `display: flex;` 또는 `inline-flex;`로 지정
- **Flex Items (자식 요소들)**: 컨테이너 안의 직접적인 자식 요소들

```html
<div class="flex-container">
  <div>Item 1</div>
  <div>Item 2</div>
  <div>Item 3</div>
</div>
```

```css
.flex-container {
  display: flex;
}
```

---

## 🧭 주요 속성 정리

### 🔹 컨테이너(Container)에 적용하는 속성

| 속성               | 설명                                      |
|--------------------|-------------------------------------------|
| `display`          | `flex` 또는 `inline-flex`로 설정           |
| `flex-direction`   | 주축 방향 설정 (`row`, `column`, ...)     |
| `flex-wrap`        | 줄바꿈 허용 여부 (`nowrap`, `wrap`)       |
| `justify-content`  | 주축 방향 정렬 (`center`, `space-between`, ...) |
| `align-items`      | 교차축 정렬 방식 (`stretch`, `center`, ...)  |
| `align-content`    | 여러 줄 정렬 방식 (`space-between`, ...)   |

---

### 🔹 아이템(Item)에 적용하는 속성

| 속성          | 설명                                         |
|---------------|----------------------------------------------|
| `flex-grow`   | 남은 공간 비율로 늘리기                      |
| `flex-shrink` | 공간 부족 시 줄어드는 비율                   |
| `flex-basis`  | 초기 크기 설정 (px, %, auto 등)              |
| `flex`        | 위 세 가지를 축약한 속성                     |
| `align-self`  | 개별 아이템 정렬 (override `align-items`)     |
| `order`       | 시각적 순서 지정                             |

---

## 🎯 flex-direction: 주축 방향 설정

```css
.flex-container {
  flex-direction: row;       /* 기본값: 가로 정렬 (좌→우) */
  flex-direction: row-reverse; /* 가로 정렬 (우→좌) */
  flex-direction: column;    /* 세로 정렬 (위→아래) */
  flex-direction: column-reverse; /* 세로 정렬 (아래→위) */
}
```

---

## ⤵ flex-wrap: 줄바꿈 설정

```css
.flex-container {
  flex-wrap: nowrap;    /* 기본값: 한 줄만 */
  flex-wrap: wrap;      /* 넘치면 줄바꿈 */
  flex-wrap: wrap-reverse; /* 줄바꿈, 반대 방향 */
}
```

---

## 🎯 justify-content: 주축 방향 정렬

```css
.flex-container {
  justify-content: flex-start;     /* 왼쪽 정렬 (기본) */
  justify-content: center;         /* 가운데 정렬 */
  justify-content: flex-end;       /* 오른쪽 정렬 */
  justify-content: space-between;  /* 양 끝 정렬 + 사이 균등 */
  justify-content: space-around;   /* 요소 주위에 동일한 간격 */
  justify-content: space-evenly;   /* 요소 사이 균등한 간격 */
}
```

---

## 🎯 align-items: 교차축 정렬

```css
.flex-container {
  align-items: stretch;   /* 교차축 늘림 (기본) */
  align-items: center;    /* 교차축 가운데 정렬 */
  align-items: flex-start;/* 교차축 시작점 정렬 */
  align-items: flex-end;  /* 교차축 끝점 정렬 */
  align-items: baseline;  /* 텍스트 baseline 기준 정렬 */
}
```

---

## 📚 align-content: 여러 줄 정렬

여러 줄로 wrap될 때 **전체 줄들 간의 정렬 방식**을 지정

```css
.flex-container {
  flex-wrap: wrap;
  align-content: flex-start;
  align-content: center;
  align-content: space-between;
}
```

---

## 📦 flex-grow, shrink, basis

### 🔸 flex-grow: 남은 공간을 비율대로 분배

```css
.item1 { flex-grow: 1; }
.item2 { flex-grow: 2; }
/* item2가 item1보다 2배 더 큼 */
```

### 🔸 flex-shrink: 공간이 부족할 때 줄어드는 비율

```css
.item1 { flex-shrink: 1; }
.item2 { flex-shrink: 0; }
/* item2는 절대 줄어들지 않음 */
```

### 🔸 flex-basis: 초기 크기 지정

```css
.item1 { flex-basis: 200px; }
/* 초기 넓이를 200px로 설정 */
```

---

## ✂️ flex: 단축 속성

```css
.item {
  flex: 1 0 150px; /* grow, shrink, basis 순서 */
}
```

### 자주 쓰는 단축 형태

```css
flex: 1;      /* flex: 1 1 0; */
flex: auto;   /* flex: 1 1 auto; */
flex: none;   /* flex: 0 0 auto; */
```

---

## 🔀 order: 아이템 순서 변경

```css
.item1 { order: 1; }
.item2 { order: 0; }
/* item2가 앞에 표시됨 */
```

---

## 🧍 align-self: 개별 아이템 정렬

```css
.item {
  align-self: center;
  align-self: flex-end;
}
```

- 컨테이너의 `align-items`를 덮어쓰는 **개별 설정**

---

## ✅ 실전 예제: 수직 가운데 정렬

```css
.container {
  display: flex;
  justify-content: center;
  align-items: center;
  height: 100vh;
}
```

---

## ✅ 실전 예제: 네비게이션 바

```css
.navbar {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 1rem;
}
```

---

## 🧠 Flexbox vs Grid

| 항목         | Flexbox                          | Grid                              |
|--------------|----------------------------------|-----------------------------------|
| 방향         | 1차원(가로 또는 세로)            | 2차원(행과 열)                    |
| 정렬         | 콘텐츠 중심                      | 레이아웃 중심                     |
| 유연성       | 내용에 따라 자동 확장             | 명시적으로 영역 분할 가능         |
| 사용 용도    | 버튼 그룹, 네비게이션, 카드 목록 등 | 페이지 전체 레이아웃 구성 등      |

---

## 📌 요약 정리

| 속성             | 대상        | 설명 |
|------------------|-------------|------|
| `display: flex`  | 컨테이너    | Flex 컨텍스트 설정 |
| `flex-direction` | 컨테이너    | 주축 방향 지정 |
| `justify-content`| 컨테이너    | 주축 정렬 |
| `align-items`    | 컨테이너    | 교차축 정렬 |
| `flex-wrap`      | 컨테이너    | 줄바꿈 여부 |
| `flex`           | 아이템      | 크기 유연성 통제 |
| `align-self`     | 아이템      | 개별 정렬 |
| `order`          | 아이템      | 시각 순서 변경 |

---

## 💡 마무리 팁

- Flexbox는 `수직/수평 정렬`, `공간 분배`, `반응형 레이아웃` 구현에 최적
- 요소 간 **간격 조절**엔 `gap` 속성도 함께 사용하면 편리함

```css
display: flex;
gap: 1rem;
```

- Flexbox는 간단한 UI 정렬부터 카드 그리드까지 다양하게 쓰이는 **필수 기술**입니다!

---

🔗 참고 사이트:
- [Flexbox Froggy 게임](https://flexboxfroggy.com/)
- [CSS Tricks - Flexbox Guide](https://css-tricks.com/snippets/css/a-guide-to-flexbox/)