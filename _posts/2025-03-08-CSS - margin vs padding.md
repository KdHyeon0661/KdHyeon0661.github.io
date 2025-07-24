---
layout: post
title: CSS - margin vs padding
date: 2025-03-08 21:20:23 +0900
category: CSS
---
# 🔍 margin vs padding 차이 — 완벽 정리

CSS를 배우다 보면 가장 헷갈리는 개념 중 하나가 바로 **`margin`과 `padding`의 차이**입니다.  
두 속성 모두 **공간을 띄우는 역할**을 하지만, 적용되는 위치와 목적이 다릅니다.

이 글에서는 `margin`과 `padding`의 차이를 시각적으로, 개념적으로, 코드 예제로 완벽하게 정리합니다.

---

## 📦 다시 보는 박스 모델 요약

먼저 박스 모델을 간단히 떠올려 봅시다:

```
[ margin ]
   [ border ]
     [ padding ]
       [ content ]
```

- `padding`: 콘텐츠와 테두리 사이의 **내부 여백**
- `margin`: 박스와 다른 요소 사이의 **외부 여백**

---

## 📌 margin 이란?

- **박스 바깥쪽 여백**
- **다른 요소와의 간격 조정**에 사용
- 투명하며 배경색은 적용되지 않음

### ✅ 예시

```css
.box {
  margin: 20px;
}
```

```html
<div class="box">박스 1</div>
<div class="box">박스 2</div>
```

- 위 두 박스 사이의 간격은 20px (또는 margin collapse로 인해 더 적을 수 있음)

---

## 📌 padding 이란?

- **박스 안쪽 여백**
- 콘텐츠와 테두리 사이의 공간 확보
- **배경색이 포함됨**

### ✅ 예시

```css
.box {
  padding: 20px;
  background-color: lightblue;
}
```

```html
<div class="box">텍스트가 테두리에서 20px 떨어져 있음</div>
```

- 콘텐츠와 테두리 사이에 여백이 생김

---

## 🆚 비교: margin vs padding

| 항목 | margin | padding |
|------|--------|---------|
| 위치 | 요소 **바깥쪽** | 요소 **안쪽** |
| 목적 | **다른 요소와 거리** 확보 | **콘텐츠와 테두리** 간 거리 확보 |
| 배경색 포함 | ❌ 포함되지 않음 | ✅ 포함됨 |
| 중첩 여부 | 마진 겹침(margin collapse) 발생 가능 | 없음 |
| 애니메이션 | 가능 | 가능 |
| box-sizing 영향 | 영향을 받지 않음 | `border-box`이면 포함됨 |

---

## 🎯 시각 예시 코드

```html
<style>
  .box1 {
    background: #ddd;
    margin: 20px;
  }
  .box2 {
    background: #add8e6;
    padding: 20px;
  }
</style>

<div class="box1">Margin 사용</div>
<div class="box2">Padding 사용</div>
```

- `.box1`: 다른 요소들과의 **외부 거리**
- `.box2`: 콘텐츠가 **내부에서 여유 공간**을 가짐

---

## 🚨 margin collapse 주의

```css
p {
  margin: 20px;
}
```

```html
<p>문단 1</p>
<p>문단 2</p>
```

두 문단의 상하 `margin`은 겹쳐서 총 40px이 아닌 **20px만 적용**됩니다.

> 🎯 해결 방법: `overflow: hidden;` 또는 `padding`으로 대체 사용

---

## 🧠 실무 적용 예시

| 목적 | 사용 속성 |
|------|-----------|
| 박스 사이 간격 띄우기 | `margin` |
| 버튼 안쪽 여백 확보 | `padding` |
| 카드 안의 텍스트와 테두리 간 거리 | `padding` |
| 여러 박스 사이 균등 간격 유지 | `margin` |
| 내부 요소 위치 맞춤 | `padding` + `text-align`, `line-height` 등과 함께 사용

---

## ✅ 마무리 요약

| 항목 | margin | padding |
|------|--------|---------|
| 공간 위치 | 요소 바깥쪽 | 요소 안쪽 |
| 배경색 적용 | ❌ 안 됨 | ✅ 됨 |
| 마진 겹침 | 발생 가능 | 없음 |
| 다른 요소와의 간격 | ✅ | ❌ |
| 콘텐츠 여백 조정 | ❌ | ✅ |

---

> **결론:**  
> - **margin**은 바깥과의 관계  
> - **padding**은 안쪽과의 거리  
> - 둘 다 중요하고 자주 쓰이므로 **의도에 맞게 선택**하는 것이 중요합니다.