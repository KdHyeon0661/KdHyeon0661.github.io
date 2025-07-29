---
layout: post
title: CSS - float와 clear
date: 2025-04-06 23:20:23 +0900
category: CSS
---
# 🌊 float와 clear 속성 이해하기

CSS의 `float`는 원래 텍스트 흐름에 이미지 등을 배치하기 위해 만들어졌지만,  
레이아웃 기법으로도 오랫동안 활용되어 왔습니다.  
`float` 사용 시 발생하는 문제를 해결하는 데 `clear` 속성이나 clearfix 기법이 자주 사용됩니다.

---

## ✅ 1. float란?

요소를 **왼쪽 또는 오른쪽으로 띄워서** 주변 텍스트가 흐르게 만듭니다.

```css
img {
  float: left;
}
```

```html
<p>
  <img src="image.jpg" alt="..." />
  텍스트는 이미지의 오른쪽으로 흐르게 됩니다.
</p>
```

### float 속성 값

| 값          | 설명                           |
|-------------|--------------------------------|
| `left`      | 왼쪽으로 띄우기                |
| `right`     | 오른쪽으로 띄우기              |
| `none`      | float 해제 (기본값)            |
| `inherit`   | 부모 요소로부터 상속           |

---

## ✅ 2. float의 주요 사용 예

### 🔹 텍스트 주변 이미지 배치

```html
<img src="..." style="float: left; margin-right: 10px;" />
<p>이미지가 왼쪽에 있고, 텍스트가 오른쪽으로 감싸집니다.</p>
```

### 🔹 레이아웃 분할 (과거 방식)

```html
<div class="container">
  <div class="sidebar">사이드바</div>
  <div class="content">본문</div>
</div>
```

```css
.sidebar {
  float: left;
  width: 30%;
}
.content {
  float: right;
  width: 70%;
}
```

---

## ⚠️ 3. float로 생기는 문제점

`float`된 요소는 **부모의 높이를 무시**하게 됩니다.

```html
<div class="box">
  <div class="float-box">내용</div>
</div>
```

```css
.float-box {
  float: left;
  width: 100px;
  height: 100px;
  background: lightblue;
}

.box {
  background: lightgray;
  /* 높이가 collapse 되어 안 보일 수 있음 */
}
```

- `.box`는 자식의 높이를 인식하지 못하고 0px처럼 처리됨

---

## ✅ 4. clear 속성

`float`의 영향을 **무시하고 새 줄에서 시작**하게 만드는 속성입니다.

```css
.clearfix {
  clear: both;
}
```

### clear 값

| 값        | 설명                         |
|-----------|------------------------------|
| `left`    | 왼쪽 float 무시               |
| `right`   | 오른쪽 float 무시             |
| `both`    | 양쪽 float 무시               |
| `none`    | 기본값, float 영향을 받음     |

### 예시

```html
<div style="float: left; width: 100px;">왼쪽</div>
<div style="clear: both;">float 요소 아래에서 시작</div>
```

---

## ✅ 5. float 문제 해결법: clearfix 기법

부모 요소가 float된 자식을 감싸지 못하는 문제를 해결하기 위해 **clearfix**라는 방법을 사용합니다.

### 방법 1: `::after`를 이용한 clearfix

```css
.clearfix::after {
  content: "";
  display: block;
  clear: both;
}
```

```html
<div class="container clearfix">
  <div style="float: left;">왼쪽</div>
  <div style="float: right;">오른쪽</div>
</div>
```

> CSS만으로 float 문제를 해결하는 가장 일반적인 방법

---

## ✅ 6. float vs modern layout (flex/grid)

| 항목       | float                  | Flexbox / Grid              |
|------------|------------------------|-----------------------------|
| 목적       | 이미지 주변 텍스트 배치 | 전체 레이아웃, 정렬        |
| 정렬       | 직접 제어 어려움        | 정렬, 간격 조정 유연       |
| 부모 높이 문제 | 있음                 | 없음                        |
| 반응형     | 제어 어려움             | 용이                        |
| 현재 추천 | ❌ (구식)               | ✅ Flex/Grid 사용 권장       |

---

## ✅ 7. 실전 예제: float + clearfix

```html
<div class="card clearfix">
  <img src="..." alt="..." class="float-img" />
  <p>이미지 옆에 있는 텍스트입니다. float는 이미지가 왼쪽으로 뜨게 합니다.</p>
</div>
```

```css
.float-img {
  float: left;
  width: 100px;
  margin-right: 15px;
}

.card::after {
  content: "";
  display: block;
  clear: both;
}
```

---

## 📌 요약 정리

| 항목       | 설명                                |
|------------|-------------------------------------|
| `float`    | 요소를 왼쪽 또는 오른쪽으로 띄움     |
| `clear`    | float 영향을 제거하고 새 줄에서 시작 |
| `clearfix` | 부모가 float 자식을 감싸도록 처리    |

---

## 💡 마무리

- `float`는 이제 레이아웃 용도로는 잘 **사용하지 않지만**,  
  **텍스트 감싸기**, **이미지 정렬** 등에서는 여전히 쓰이는 기능입니다.
- 복잡한 레이아웃에는 `flex`나 `grid` 사용을 권장합니다.
- float를 써야 한다면 **clearfix** 패턴을 꼭 기억해두세요!

---

🔗 참고 링크
- [MDN: float](https://developer.mozilla.org/ko/docs/Web/CSS/float)
- [MDN: clear](https://developer.mozilla.org/ko/docs/Web/CSS/clear)
- [clearfix 패턴 소개 (CSS-Tricks)](https://css-tricks.com/snippets/css/clear-fix/)