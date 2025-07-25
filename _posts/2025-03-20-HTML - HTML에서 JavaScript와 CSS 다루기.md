---
layout: post
title: HTML - HTML에서 JavaScript와 CSS 다루기
date: 2025-03-20 19:20:23 +0900
category: HTML
---
# 🌐 HTML에서 JavaScript와 CSS 다루기 – 웹 개발 입문자를 위한 완전 가이드

HTML은 구조를 만드는 언어입니다. 하지만 **동작(행동)**은 `JavaScript`, **디자인(스타일)**은 `CSS`를 통해 표현됩니다. 이 글에서는 HTML 문서 안에서 **JavaScript(JS)**와 **CSS**를 어떻게 포함하고 연결하는지 자세히 알아보겠습니다.

---

## 🎨 CSS – HTML에 스타일을 더하는 방법

### ✅ CSS를 적용하는 3가지 방법

#### 1. 인라인 스타일 (inline style)

HTML 요소의 `style` 속성으로 직접 지정합니다.

```html
<p style="color: red; font-size: 18px;">빨간 텍스트</p>
```

- **장점**: 빠르게 적용 가능
- **단점**: 코드 분리 불가, 유지보수 어려움

---

#### 2. 내부 스타일 (internal style)

HTML 문서의 `<head>` 안에 `<style>` 태그로 작성합니다.

```html
<head>
  <style>
    h1 {
      color: navy;
      background-color: lightgray;
    }
  </style>
</head>
```

- 작은 프로젝트나 단일 문서에 적합

---

#### 3. 외부 스타일 (external stylesheet)

CSS 파일을 따로 만들고 `<link>` 태그로 HTML에 연결합니다.

```html
<!-- HTML 문서 head 안에 작성 -->
<link rel="stylesheet" href="style.css">
```

```css
/* style.css */
p {
  color: green;
  font-size: 16px;
}
```

- **가장 권장되는 방식**
- **유지보수 및 재사용**에 유리

---

### ✅ CSS 위치는 어디?

- `<style>` 태그는 **반드시 `<head>` 안**에 위치해야 합니다.
- `<link>` 태그도 **`<head>` 내**에 위치시켜야 CSS가 올바르게 적용됩니다.

---

### 🎨 CSS를 활용한 스타일 예시

```html
<!DOCTYPE html>
<html>
<head>
  <style>
    body {
      background-color: #f5f5f5;
    }
    h1 {
      color: darkblue;
      text-align: center;
    }
    p {
      color: gray;
      line-height: 1.6;
    }
  </style>
</head>
<body>
  <h1>CSS 예제</h1>
  <p>이 문장은 CSS를 사용해서 스타일링되었습니다.</p>
</body>
</html>
```

---

## ⚙️ JavaScript – HTML에 동작을 추가하는 방법

### ✅ JavaScript 삽입 방식 3가지

#### 1. 인라인 스크립트 (이벤트 속성)

HTML 요소의 속성에서 직접 JavaScript 실행

```html
<button onclick="alert('클릭!')">클릭하세요</button>
```

- 빠르게 테스트 가능하지만 비추천 (유지보수 어려움)

---

#### 2. 내부 스크립트 (Internal Script)

HTML 문서 안에 `<script>` 태그로 직접 작성

```html
<script>
  console.log("페이지가 로드되었습니다.");
</script>
```

보통 `<head>`나 **`<body>`의 끝부분에 위치**시킵니다.

---

#### 3. 외부 스크립트 (External Script)

`.js` 파일을 생성한 후 `<script src="...">`로 불러옵니다.

```html
<!-- HTML -->
<script src="main.js"></script>
```

```js
// main.js
console.log("외부 스크립트가 실행되었습니다.");
```

- **가장 권장되는 방식**
- **HTML과 JS 코드 분리**, 재사용 용이

---

### ✅ JavaScript 위치는 어디?

| 위치 | 설명 |
|------|------|
| `<head>` | HTML 로딩 전에 스크립트가 실행됨. `defer` 또는 `async` 사용 권장 |
| `<body>` 끝 | HTML 요소가 모두 로드된 이후 실행. **실무에서 자주 사용** |

```html
<!-- 가장 일반적인 위치 -->
</body>
<script src="main.js"></script>
</html>
```

---

### 📌 `defer`와 `async` 속성

```html
<script src="script.js" defer></script>
```

- `defer`: HTML 파싱 완료 후 실행 (순서 보장)
- `async`: 다운로드 즉시 실행 (순서 보장 안 됨)

> **SPA, 프론트엔드 프레임워크에서는 defer 사용 권장**

---

## 🧪 HTML + JS + CSS 통합 예제

```html
<!DOCTYPE html>
<html lang="ko">
<head>
  <meta charset="UTF-8">
  <title>HTML + CSS + JS</title>
  <style>
    body {
      font-family: sans-serif;
      text-align: center;
    }
    #btn {
      padding: 10px 20px;
      background-color: teal;
      color: white;
      border: none;
      cursor: pointer;
    }
  </style>
</head>
<body>
  <h1>이벤트 버튼</h1>
  <button id="btn">눌러보세요</button>

  <script>
    document.getElementById("btn").addEventListener("click", function () {
      alert("버튼을 눌렀습니다!");
    });
  </script>
</body>
</html>
```

---

## ✅ 정리

| 구분 | 방식 | 특징 | 위치 |
|------|------|------|------|
| CSS | 인라인 | 요소에 직접 스타일 | `style=""` |
|      | 내부   | `<style>` 태그 | `<head>` 안 |
|      | 외부   | 별도 CSS 파일 | `<link href="...">` |
| JS  | 인라인 | 이벤트 속성 직접 실행 | `onclick="..."` |
|      | 내부   | `<script>` 태그 | `<head>` 또는 `<body>` 끝 |
|      | 외부   | 별도 JS 파일 | `<script src="...">` |

---

HTML에 CSS와 JavaScript를 연동하는 것은 **정적 페이지 → 동적 웹사이트**로 나아가는 첫 단계입니다.  
구조(HTML) + 스타일(CSS) + 동작(JS)의 삼박자가 맞아야 현대적인 웹이 완성됩니다.

> 다음 단계로는 CSS 선택자/레이아웃, JavaScript DOM 조작, 이벤트 핸들링 등을 학습해 보세요!