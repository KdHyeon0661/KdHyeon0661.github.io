---
layout: post
title: HTML - Drag & Drop API
date: 2025-04-23 22:20:23 +0900
category: HTML
---
# 🎯 HTML Drag & Drop API 완전 정복

## ✅ Drag and Drop API란?

HTML5에서 제공하는 **기본 이벤트 기반 드래그 기능**으로,  
마우스로 요소를 **끌어다 놓을 수 있게** 해주는 기능입니다.

**웹 게임**, **관리자 UI 정렬**, **파일 업로드 영역**, **커스터마이징 도구** 등에 자주 사용됩니다.

---

## 📦 기본 개념 흐름

1. **드래그 대상 요소**에 `draggable="true"` 설정
2. `dragstart` 이벤트에서 **데이터 등록**
3. 드롭 대상에 `dragover`, `drop` 이벤트 리스너 부착
4. `drop` 이벤트에서 **데이터 처리 및 이동**

---

## 🔨 간단한 예제: 박스 이동하기

### ✅ HTML

```html
<style>
  #box {
    width: 100px;
    height: 100px;
    background: skyblue;
    margin-bottom: 20px;
  }
  #dropzone {
    width: 300px;
    height: 150px;
    border: 2px dashed #999;
    padding: 20px;
  }
</style>

<div id="box" draggable="true">📦 드래그</div>

<div id="dropzone">🎯 여기로 드롭</div>
```

---

### ✅ JavaScript

```html
<script>
  const box = document.getElementById('box');
  const dropzone = document.getElementById('dropzone');

  // 1. 드래그 시작 시
  box.addEventListener('dragstart', (e) => {
    e.dataTransfer.setData('text/plain', 'box'); // 전송할 데이터 설정
    e.target.style.opacity = 0.5;
  });

  // 2. 드래그가 드롭 영역 위에 있을 때
  dropzone.addEventListener('dragover', (e) => {
    e.preventDefault(); // 기본 동작(드롭 금지) 막기
    dropzone.style.background = '#eef';
  });

  // 3. 드롭 했을 때
  dropzone.addEventListener('drop', (e) => {
    e.preventDefault();
    const data = e.dataTransfer.getData('text/plain');
    if (data === 'box') {
      dropzone.appendChild(box); // 요소 이동
      box.style.opacity = 1;
    }
    dropzone.style.background = ''; // 원상복구
  });

  // 4. 드래그 종료
  box.addEventListener('dragend', (e) => {
    box.style.opacity = 1;
  });
</script>
```

---

## 📌 주요 이벤트 및 메서드 요약

| 이벤트 | 설명 |
|--------|------|
| `dragstart` | 드래그 시작할 때 발생 |
| `drag` | 드래그 중에 계속 발생 |
| `dragend` | 드래그 끝났을 때 발생 |
| `dragenter` | 드롭 대상 위로 처음 진입할 때 |
| `dragover` | 드롭 대상 위에 있을 때 (여기서 `preventDefault()` 필수) |
| `dragleave` | 드롭 대상에서 벗어날 때 |
| `drop` | 드롭 되었을 때 발생 |

| 메서드 | 설명 |
|--------|------|
| `e.dataTransfer.setData(type, data)` | 전송할 데이터를 설정 |
| `e.dataTransfer.getData(type)` | 드롭 시 데이터를 가져옴 |
| `e.preventDefault()` | 기본 동작 방지 (특히 dragover에서 필요) |

---

## 🖼️ 예제 2: 이미지 드래그로 옮기기

```html
<img src="cat.png" id="cat" width="100" draggable="true">
<div id="target" style="width:200px;height:150px;border:1px solid #ccc;">🐾 Drop Zone</div>

<script>
  document.getElementById('cat').addEventListener('dragstart', function (e) {
    e.dataTransfer.setData('text/plain', e.target.id);
  });

  const target = document.getElementById('target');
  target.addEventListener('dragover', function (e) {
    e.preventDefault();
  });

  target.addEventListener('drop', function (e) {
    e.preventDefault();
    const id = e.dataTransfer.getData('text');
    const draggedEl = document.getElementById(id);
    target.appendChild(draggedEl);
  });
</script>
```

---

## ⚠️ 주의 사항

- `draggable="true"`를 반드시 설정해야 함
- `dragover` 이벤트에서 반드시 `preventDefault()`를 호출해야 **drop 이벤트가 발생**
- 브라우저마다 기본 드래그 동작이 달라질 수 있으므로 테스트 필요
- `dataTransfer`는 문자열만 지원 → 객체는 JSON으로 직렬화해야 함

---

## 🧪 고급 활용 아이디어

- ✅ 파일 업로드 영역
- ✅ Trello / Kanban 스타일 카드 정렬
- ✅ 드래그 가능한 리스트 (Sortable.js 등과 결합)
- ✅ 커스터마이징 가능한 UI 컴포넌트
- ✅ 이미지 / 파일 미리보기 + 위치 이동

---

## 📚 참고 자료

- [MDN Drag and Drop API](https://developer.mozilla.org/en-US/docs/Web/API/HTML_Drag_and_Drop_API)
- [HTML5 Rocks Drag and Drop Guide](https://www.html5rocks.com/en/tutorials/dnd/basics/)
- [Sortable.js (드래그 정렬 라이브러리)](https://github.com/SortableJS/Sortable)