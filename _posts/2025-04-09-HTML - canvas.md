---
layout: post
title: HTML - canvas
date: 2025-04-09 19:20:23 +0900
category: HTML
---
# 🎨 HTML5 `<canvas>` 태그 완전 가이드

HTML5의 `<canvas>` 태그는 웹 페이지에 **그래픽을 직접 그릴 수 있는 도구**입니다.  
JavaScript와 함께 사용하여 **도형, 그래프, 게임, 시뮬레이션, 데이터 시각화**까지 다양한 작업을 수행할 수 있습니다.

---

## ✅ `<canvas>`란?

`<canvas>`는 웹 페이지 상에 **픽셀 기반의 그림을 그릴 수 있는 빈 캔버스**를 제공합니다.  
이 캔버스 위에 **JavaScript를 이용해 그림을 그리는 방식**으로 작동합니다.

> 즉, `<canvas>` 자체는 단순한 요소이고, **실제 그리기는 JavaScript로** 처리합니다.

---

## 🧱 기본 문법

```html
<canvas id="myCanvas" width="400" height="300">
  이 브라우저는 canvas를 지원하지 않습니다.
</canvas>
```

- `width`와 `height`는 캔버스의 실제 해상도를 설정합니다.
- 내용은 `<canvas>`를 지원하지 않는 브라우저를 위한 **대체 텍스트**입니다.

---

## 🖌 JavaScript로 그리기

### 1️⃣ 캔버스 컨텍스트 가져오기

```js
const canvas = document.getElementById('myCanvas');
const ctx = canvas.getContext('2d');  // 2D 컨텍스트 가져오기
```

> `getContext('2d')`를 통해 2차원 그리기 API를 사용할 수 있습니다.

---

## 🎯 주요 기능 및 메서드

### 🔷 도형 그리기

#### 사각형
```js
ctx.fillStyle = "skyblue";
ctx.fillRect(50, 50, 100, 80); // x, y, width, height
```

#### 선(Line)
```js
ctx.beginPath();
ctx.moveTo(20, 20);
ctx.lineTo(200, 100);
ctx.strokeStyle = "red";
ctx.stroke();
```

#### 원(Circle)
```js
ctx.beginPath();
ctx.arc(150, 150, 50, 0, Math.PI * 2);
ctx.fillStyle = "orange";
ctx.fill();
```

---

### 🖋 텍스트

```js
ctx.font = "24px Arial";
ctx.fillStyle = "black";
ctx.fillText("Hello Canvas", 50, 200);
```

---

### 🖼 이미지 삽입

```js
const img = new Image();
img.src = "image.jpg";
img.onload = function () {
  ctx.drawImage(img, 0, 0, 200, 150);
};
```

---

### 🧼 초기화 / 지우기

```js
ctx.clearRect(0, 0, canvas.width, canvas.height);
```

---

## ⚙️ 전체 예제

```html
<canvas id="myCanvas" width="400" height="300" style="border:1px solid #ccc;"></canvas>

<script>
  const canvas = document.getElementById('myCanvas');
  const ctx = canvas.getContext('2d');

  // 배경
  ctx.fillStyle = "#f0f0f0";
  ctx.fillRect(0, 0, 400, 300);

  // 텍스트
  ctx.font = "20px sans-serif";
  ctx.fillStyle = "blue";
  ctx.fillText("Canvas Demo", 120, 40);

  // 원
  ctx.beginPath();
  ctx.arc(200, 150, 60, 0, Math.PI * 2);
  ctx.fillStyle = "green";
  ctx.fill();

  // 선
  ctx.beginPath();
  ctx.moveTo(50, 250);
  ctx.lineTo(350, 250);
  ctx.strokeStyle = "black";
  ctx.stroke();
</script>
```

---

## 📌 장단점 정리

### ✅ 장점
- 빠른 렌더링 성능 (특히 애니메이션)
- JavaScript와 직접 연동 가능
- 자유로운 픽셀 조작
- WebGL과 연계하면 3D 렌더링도 가능

### ❌ 단점
- DOM 요소가 없기 때문에 **이벤트나 접근성에 불리**
- 자동 크기 조정이나 반응형 적용이 까다로움
- 상태 유지가 어려움 (지속적으로 다시 그려야 함)

---

## 💡 실전 활용 사례

| 활용 분야 | 설명 |
|-----------|------|
| 🕹 HTML5 게임 | 캐릭터, 배경, 충돌 처리 등 모두 canvas 기반 |
| 📊 데이터 시각화 | 차트, 그래프 (Chart.js 등) |
| 🖍 그림판 앱 | 사용자 드로잉 구현 |
| 🧮 물리 시뮬레이션 | 입자, 충돌, 중력 등 표현 |
| 📷 영상 처리 | 실시간 필터, 픽셀 조작 |

---

## 🔮 캔버스 고급 주제

- 🎮 **애니메이션 루프 구현** (`requestAnimationFrame`)
- 🎨 **픽셀 데이터 조작** (`getImageData`, `putImageData`)
- 🌌 **WebGL**: 3D 그래픽을 위한 고성능 렌더링
- 📦 라이브러리: Fabric.js, Konva.js, Pixi.js 등

---

## ✅ 마무리

HTML5의 `<canvas>`는 **픽셀 단위로 직접 그리기를 가능하게 해주는 강력한 도구**입니다.  
단순한 그림부터 고급 애니메이션까지, JavaScript와 결합하여 웹의 표현력을 대폭 확장시켜 줍니다.

> 캔버스는 "공간"일 뿐이며, "그림을 그리는 도구"는 자바스크립트입니다.

---

## 📚 참고 자료

- [MDN - Canvas API](https://developer.mozilla.org/ko/docs/Web/API/Canvas_API)
- [HTML Living Standard - Canvas](https://html.spec.whatwg.org/multipage/canvas.html)
- [Chart.js로 캔버스를 이용한 차트 만들기](https://www.chartjs.org/)
- [Pixi.js - 고성능 2D 렌더링 엔진](https://pixijs.com/)

---

> 다음으로 추천하는 주제:
> - 🎮 `requestAnimationFrame`으로 애니메이션 만들기
> - 🎯 Canvas로 충돌 감지 구현하기
> - 📦 Canvas 기반 라이브러리 비교 (Fabric.js, Konva.js 등)