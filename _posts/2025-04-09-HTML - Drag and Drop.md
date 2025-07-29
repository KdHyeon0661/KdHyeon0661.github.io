---
layout: post
title: HTML - Drag and Drop
date: 2025-04-09 22:20:23 +0900
category: HTML
---
# 🧲 HTML5 Drag and Drop 완벽 가이드

HTML5에서 제공하는 **Drag and Drop API**는 사용자가 웹 요소를 마우스로 **끌어다 놓을 수 있도록** 만들어주는 기능입니다.  
간단한 파일 업로드 인터페이스부터 복잡한 UI 구성까지 다양한 곳에 활용됩니다.

---

## ✅ Drag and Drop이란?

> 사용자가 마우스로 HTML 요소를 드래그(drag)한 후, 다른 요소로 드롭(drop)할 수 있게 해주는 API

기본적으로 `draggable` 속성을 가진 요소는 마우스로 드래그할 수 있고, 특정 영역에 드롭하도록 제어할 수 있습니다.

---

## 🧱 기본 사용 구조

Drag & Drop 구현에는 **다음 3가지 단계**가 필요합니다:

1. **드래그 가능한 요소** 설정
2. **드롭 대상 영역** 설정
3. **이벤트 핸들링** (dragstart, dragover, drop 등)

---

## 🔧 주요 속성

- `draggable="true"`  
  드래그 가능하게 만들기 위해 필요한 HTML 속성입니다.

```html
<img src="logo.png" draggable="true" />
```

---

## 📦 주요 이벤트 흐름

| 이벤트 | 발생 위치 | 설명 |
|--------|-----------|------|
| `dragstart` | 드래그 시작 요소 | 드래그가 시작될 때 발생 |
| `drag` | 드래그 중 요소 | 드래그 중 지속적으로 발생 |
| `dragenter` | 드롭 대상 | 드래그 대상이 진입할 때 |
| `dragover` | 드롭 대상 | 드래그 대상 위에 머무를 때 (기본 동작 막아야 drop 허용됨) |
| `dragleave` | 드롭 대상 | 대상에서 벗어날 때 |
| `drop` | 드롭 대상 | 드롭이 완료될 때 |
| `dragend` | 드래그 원본 | 드래그 종료 시 발생 |

---

## 🖼 실전 예제

```html
<style>
#box {
  width: 150px;
  height: 150px;
  background: lightblue;
  margin: 20px;
  display: inline-block;
}
#dropzone {
  width: 200px;
  height: 200px;
  background: lightgray;
  border: 2px dashed #888;
}
</style>

<div id="box" draggable="true">📦 드래그!</div>
<div id="dropzone">🎯 여기에 드롭</div>

<script>
const box = document.getElementById('box');
const dropzone = document.getElementById('dropzone');

// 드래그 시작 시 데이터 설정
box.addEventListener('dragstart', (e) => {
  e.dataTransfer.setData("text/plain", "dragged-box");
});

// 드롭 대상에서 기본 동작 막기 (drop 허용)
dropzone.addEventListener('dragover', (e) => {
  e.preventDefault();
});

// 드롭 처리
dropzone.addEventListener('drop', (e) => {
  e.preventDefault();
  const data = e.dataTransfer.getData("text/plain");
  if (data === "dragged-box") {
    dropzone.appendChild(box);
  }
});
</script>
```

---

## 🧠 주요 개념 정리

### 📌 `dataTransfer` 객체

- `setData(type, data)`: 드래그할 데이터 설정 (예: `text/plain`, `text/html`)
- `getData(type)`: 드롭 시 데이터 가져오기
- `effectAllowed`, `dropEffect`: 허용/표현되는 드래그 유형 설정

### 예:
```js
e.dataTransfer.effectAllowed = "move";
e.dataTransfer.dropEffect = "copy";
```

---

## 🛠 파일 업로드 예제 (Drag & Drop)

```html
<div id="drop-area" style="width:300px;height:150px;border:2px dashed #999;">
  여기에 파일을 드롭하세요
</div>

<script>
const area = document.getElementById('drop-area');

area.addEventListener('dragover', (e) => {
  e.preventDefault();
});

area.addEventListener('drop', (e) => {
  e.preventDefault();
  const files = e.dataTransfer.files;
  alert(`업로드할 파일 수: ${files.length}`);
});
</script>
```

---

## 🚧 주의사항

| 항목 | 설명 |
|------|------|
| ❗ 반드시 `dragover` 이벤트에서 `e.preventDefault()` 해야 drop 이벤트가 발생합니다 |
| 📋 `setData()`를 호출하지 않으면 일부 브라우저는 drag-and-drop을 무시할 수 있습니다 |
| 🔐 보안상 브라우저는 파일 드래그 시 파일 내용에 접근을 제한합니다 |
| 📱 모바일은 기본적으로 Drag & Drop이 제한적이므로 별도 구현 필요 (예: touch API 사용) |

---

## 💡 활용 예시

| 분야 | 예시 |
|------|------|
| 📁 파일 업로드 | 사용자가 파일을 드래그해서 업로드 창에 드롭 |
| 📦 요소 이동 | 카드형 UI에서 아이템 순서 변경 (ex. Trello) |
| 🧩 위젯 재배치 | 사용자 정의 대시보드 구성 |
| 📊 구성요소 설정 | 컴포넌트 구성, 드래그로 필터 이동 등 |

---

## ✅ 마무리

HTML5의 Drag and Drop API는 기본적인 인터페이스만 제공하며,  
**직접 DOM과 이벤트를 제어하여 사용자 경험을 완성**해야 합니다.

복잡한 UI에서는 **외부 라이브러리**(예: `SortableJS`, `React DnD`, `Dragula`, `interact.js`)를 사용하는 것도 좋은 방법입니다.

---

## 📚 참고 자료

- [MDN - HTML Drag and Drop API](https://developer.mozilla.org/ko/docs/Web/API/HTML_Drag_and_Drop_API)
- [HTML Drag and Drop Tutorial (W3Schools)](https://www.w3schools.com/html/html5_draganddrop.asp)
- [Can I use - Drag and Drop](https://caniuse.com/dragndrop)