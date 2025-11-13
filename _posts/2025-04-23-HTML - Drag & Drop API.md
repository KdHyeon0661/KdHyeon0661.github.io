---
layout: post
title: HTML - Drag & Drop API
date: 2025-04-23 22:20:23 +0900
category: HTML
---
# HTML5 Drag and Drop

## 1. 개요와 정신 모델

HTML5 DnD는 크게 세 축으로 이해합니다.

1) **드래그 소스(Drag Source)**
- `draggable="true"` 또는 기본 드래그 가능한 요소(이미지, 링크, 선택된 텍스트 등)
- `dragstart`에서 `event.dataTransfer`에 데이터를 싣는다.

2) **드롭 타깃(Drop Target)**
- `dragover`에서 **반드시** `event.preventDefault()` 호출(기본 거부 → 허용 전환)
- `drop`에서 데이터를 꺼내 처리한다.

3) **데이터 전송(DataTransfer)**
- MIME 타입 기반(`text/plain`, `text/html`, 커스텀 타입 등)으로 여러 페이로드를 담고, `dropEffect/effectAllowed`로 동작 힌트를 제공.

---

## 2. 기본 예제 — 하나의 박스를 다른 영역으로 이동

```html
<style>
  .box { width:140px; height:140px; background:#cfe8ff; display:flex; align-items:center; justify-content:center; border-radius:8px; user-select:none; }
  .zone { width:220px; height:220px; border:2px dashed #999; display:flex; align-items:center; justify-content:center; transition: background .2s, border-color .2s; }
  .zone--over { background:#f6fbff; border-color:#4c9aff; }
</style>

<div id="dragBox" class="box" draggable="true" aria-grabbed="false" role="button" tabindex="0">드래그</div>
<div id="dropZone" class="zone" aria-label="드롭 영역">여기에 드롭</div>

<script>
const box = document.getElementById('dragBox');
const zone = document.getElementById('dropZone');

// dragstart: 전송 데이터와 효과 허용
box.addEventListener('dragstart', (e) => {
  e.dataTransfer.setData('text/plain', 'drag-box-id:dragBox'); // 일부 브라우저는 setData 없으면 드래그 비활성
  e.dataTransfer.effectAllowed = 'move'; // move | copy | link | copyMove | all ...
  box.setAttribute('aria-grabbed', 'true');

  // 사용자 정의 고스트 이미지(옵션)
  const ghost = box.cloneNode(true);
  ghost.style.cssText = 'position:absolute;top:-99999px;left:-99999px;opacity:.8;';
  document.body.appendChild(ghost);
  e.dataTransfer.setDragImage(ghost, 20, 20);
  setTimeout(() => document.body.removeChild(ghost), 0);
});

// dragover: 드롭 허용 전환
zone.addEventListener('dragover', (e) => {
  e.preventDefault();
  e.dataTransfer.dropEffect = 'move'; // UI 힌트
});

// 시각 피드백
zone.addEventListener('dragenter', () => zone.classList.add('zone--over'));
zone.addEventListener('dragleave', () => zone.classList.remove('zone--over'));

// drop: 데이터 수신
zone.addEventListener('drop', (e) => {
  e.preventDefault();
  zone.classList.remove('zone--over');
  const payload = e.dataTransfer.getData('text/plain');
  if (payload.startsWith('drag-box-id:')) zone.appendChild(box);
});

// dragend: 상태 정리
box.addEventListener('dragend', () => box.setAttribute('aria-grabbed', 'false'));
</script>
```

핵심 포인트
- `dragover`에서 **반드시** `preventDefault()`를 호출해야 `drop` 이벤트가 발생합니다.
- Firefox 등은 `dragstart`에 `setData`가 없으면 기본 드래그를 차단하는 경우가 있으므로 관용적으로 넣습니다.
- `effectAllowed`와 `dropEffect` 조합으로 커서/아이콘 힌트를 제어합니다.

---

## 3. 이벤트 흐름과 DataTransfer 모델

### 3.1 이벤트 타임라인
- dragstart → drag → (dragenter → dragover → dragleave)* → drop → dragend
- drop은 타깃에서, dragend는 **원본**에서 발생합니다.

### 3.2 DataTransfer 핵심 속성/메서드
- `setData(type, data)`, `getData(type)`
- `types`(전송된 타입 목록), `clearData([type])`
- `effectAllowed`(소스 허용: `none|copy|link|move|copyLink|copyMove|linkMove|all|uninitialized`)
- `dropEffect`(타깃 표기: `none|copy|link|move`)
- 파일 드롭 시: `dataTransfer.files`, 항목 기반 접근: `dataTransfer.items`

예시(복수 타입 싣기):

```js
e.dataTransfer.setData('text/plain', '카드:42');
e.dataTransfer.setData('text/html', '<div data-id="42">카드</div>');
// 커스텀 타입(동일 출처 컨텍스트 간 전송에서만 현실적으로 의미 있음)
e.dataTransfer.setData('application/x.myapp.card', JSON.stringify({ id: 42, title: 'Task' }));
```

---

## 4. 파일 드래그 앤드 드롭 업로드

### 4.1 미리보기 + 확장자 필터 + 업로드

```html
<style>
  #dropArea { width:420px; min-height:140px; border:2px dashed #aaa; border-radius:8px; padding:12px; }
  #dropArea.over { border-color:#4c9aff; background:#f7fbff; }
  #thumbs { display:flex; gap:8px; flex-wrap:wrap; margin-top:10px; }
  .thumb { width:96px; height:96px; object-fit:cover; border:1px solid #ddd; border-radius:6px; }
  .row { font:12px/1.3 system-ui; }
</style>

<div id="dropArea" aria-label="이미지 파일을 드롭하세요" tabindex="0">
  이미지 파일을 드래그하여 업로드(최대 5개, jpg/png).
  <div id="thumbs"></div>
  <div id="status" class="row"></div>
</div>

<script>
const area = document.getElementById('dropArea');
const thumbs = document.getElementById('thumbs');
const status = document.getElementById('status');

const accept = ['image/jpeg', 'image/png'];
const maxFiles = 5;

['dragenter','dragover'].forEach(evt => area.addEventListener(evt, (e) => {
  e.preventDefault(); e.stopPropagation();
  area.classList.add('over');
}));
['dragleave','drop'].forEach(evt => area.addEventListener(evt, (e) => {
  e.preventDefault(); e.stopPropagation();
  if (evt === 'dragleave') area.classList.remove('over');
}));

area.addEventListener('drop', async (e) => {
  area.classList.remove('over');
  const items = [...(e.dataTransfer.items || [])];
  let files = [...(e.dataTransfer.files || [])];

  // macOS Finder 등에서 폴더 드롭 시 무시(간단 처리)
  if (items.length && items.some(it => it.kind === 'file')) {
    // 크롬 계열: items 활용 시 타입 필터링 유리
    files = items.filter(it => it.kind === 'file').map(it => it.getAsFile());
  }

  // 필터링
  const accepted = files.filter(f => accept.includes(f.type)).slice(0, maxFiles);
  status.textContent = `선택 ${files.length}개 중 허용 ${accepted.length}개`;

  // 미리보기
  thumbs.innerHTML = '';
  accepted.forEach(f => {
    const url = URL.createObjectURL(f);
    const img = document.createElement('img');
    img.className = 'thumb'; img.src = url; img.alt = f.name;
    img.onload = () => URL.revokeObjectURL(url);
    thumbs.appendChild(img);
  });

  // 업로드(예시: /upload로 POST)
  for (const f of accepted) {
    const body = new FormData();
    body.append('file', f, f.name);
    try {
      const res = await fetch('/upload', { method:'POST', body });
      if (!res.ok) throw new Error(`HTTP ${res.status}`);
    } catch (err) {
      console.error('업로드 실패', f.name, err);
    }
  }
});
</script>
```

포인트
- 파일 내용은 사용자가 드롭한 **실제 파일들만 접근 가능**(보안상 임의 경로 접근 불가).
- 폴더 드롭을 정교하게 지원하려면 크롬의 `webkitGetAsEntry()` 또는 File System Access API(권한 기반)를 활용합니다.

---

## 5. 카드 정렬(재배치) — Sortable UI를 순수 DnD로 구현

```html
<style>
  .list { display:grid; grid-template-columns:repeat(3, 1fr); gap:8px; width:520px; }
  .item { background:#fff; border:1px solid #ddd; border-radius:8px; padding:10px; cursor:move; user-select:none; }
  .item.dragging { opacity:.5; }
  .placeholder { border:2px dashed #4c9aff; height:64px; border-radius:8px; }
</style>

<div id="list" class="list" aria-label="카드 정렬 목록">
  <div class="item" draggable="true">A</div>
  <div class="item" draggable="true">B</div>
  <div class="item" draggable="true">C</div>
  <div class="item" draggable="true">D</div>
  <div class="item" draggable="true">E</div>
</div>

<script>
const list = document.getElementById('list');
let draggingEl = null;
let placeholder = document.createElement('div'); placeholder.className = 'placeholder';

list.addEventListener('dragstart', (e) => {
  const target = e.target.closest('.item');
  if (!target) return;
  draggingEl = target;
  e.dataTransfer.effectAllowed = 'move';
  e.dataTransfer.setData('text/plain', 'reorder'); // 일부 브라우저 요구
  requestAnimationFrame(() => draggingEl.classList.add('dragging'));
});

list.addEventListener('dragend', () => {
  if (draggingEl) draggingEl.classList.remove('dragging');
  draggingEl = null;
  placeholder.remove();
});

list.addEventListener('dragover', (e) => {
  e.preventDefault();
  if (!draggingEl) return;
  const after = getDragAfterElement(list, e.clientY, e.clientX);
  if (after == null) {
    if (list.lastElementChild !== placeholder) list.appendChild(placeholder);
  } else {
    list.insertBefore(placeholder, after);
  }
  e.dataTransfer.dropEffect = 'move';
});

list.addEventListener('drop', (e) => {
  e.preventDefault();
  if (placeholder.parentNode && draggingEl) {
    list.insertBefore(draggingEl, placeholder);
  }
});

function getDragAfterElement(container, y, x) {
  const els = [...container.querySelectorAll('.item:not(.dragging)')];
  // 격자 기준: 가장 가까운 요소 앞에 배치
  let closest = null, closestDist = Infinity;
  els.forEach(el => {
    const r = el.getBoundingClientRect();
    const cx = r.left + r.width/2, cy = r.top + r.height/2;
    const d = (cx-x)**2 + (cy-y)**2;
    if (d < closestDist) { closestDist = d; closest = el; }
  });
  return closest;
}
</script>
```

포인트
- 순수 DnD로도 간단 정렬 구현 가능.
- 고급 기능(다중 선택, 가상 스크롤, 모바일 터치)은 **SortableJS**, **interact.js**, **React DnD** 등 라이브러리 사용이 효율적입니다.

---

## 6. 고급 주제

### 6.1 effectAllowed vs dropEffect

```js
// 소스
dragstart -> e.dataTransfer.effectAllowed = 'copyMove';

// 타깃
dragover  -> e.dataTransfer.dropEffect = 'move'; // move, copy, link 중 effectAllowed와 교집합이어야 UI가 일치
```

주의:
- 브라우저마다 키 조합(Alt/Option=copy, Cmd/Ctrl=link 등) 처리 차이가 있습니다.
- `dropEffect`는 **타깃 힌트**일 뿐, 최종 동작은 **애플리케이션 로직**으로 결정하세요.

### 6.2 사용자 지정 드래그 이미지

```js
const ghost = document.createElement('canvas');
ghost.width = 120; ghost.height = 40;
const g = ghost.getContext('2d');
g.fillStyle = '#000'; g.globalAlpha=.1; g.fillRect(0,0,120,40);
g.globalAlpha=1; g.fillStyle='#fff'; g.font='14px system-ui'; g.fillText('이동 중', 10, 24);
e.dataTransfer.setDragImage(ghost, 10, 10);
```

### 6.3 콘텐츠 선택/링크 충돌 회피

- 링크/이미지 기본 드래그 방지: `draggable="false"`, `event.preventDefault()`
- 텍스트가 드래그 중 선택되지 않게 `user-select: none` 지정.

### 6.4 성능 최적화

- `dragover`는 매우 자주 발생. 내부 연산은 **가볍게**(레이아웃 계산 최소, 스로틀).
- `getBoundingClientRect()` 호출을 줄이고, 필요 시 캐시/버치 처리.
- 고스트/플레이스홀더는 재사용하여 GC 줄이기.

---

## 7. 접근성(Accessibility)과 키보드 대안

DnD는 기본적으로 마우스 중심이므로 **키보드/스크린리더 사용자**를 위한 대체가 필요합니다.

권장 전략
- **키보드 정렬 모드**: 포커스된 항목에서 Space로 “들기”, 방향키로 이동, Space/Enter로 “놓기”.
- **ARIA**: `aria-grabbed`, `aria-dropeffect`(비권장이나 레거시 보조기기 호환), 상태 안내 라이브 리전.
- **시각 피드백**: 포커스 링, 플레이스홀더, 라이브 텍스트.

간단 키보드 핸들 예:

```js
box.addEventListener('keydown', (e) => {
  if (e.code === 'Space') {
    const grabbed = box.getAttribute('aria-grabbed') === 'true';
    box.setAttribute('aria-grabbed', String(!grabbed));
    e.preventDefault();
  }
});
```

실무에서는 DnD와 무관하게 **키보드 전용 재배치 UI**(위/아래로 이동 버튼, 이동 대화상자 등)를 함께 제공하는 것이 좋습니다.

---

## 8. 모바일/터치 환경 대응 — 포인터/터치 기반 커스텀 DnD

브라우저의 HTML5 DnD는 모바일에서 제한적입니다. 모바일 대응은 **Pointer/Touch 이벤트로 커스텀 DnD**를 구현하는 것이 일반적입니다.

간단 스케치:

```js
const el = document.getElementById('dragBox');
let startX=0, startY=0, ox=0, oy=0, dragging=false;

el.addEventListener('pointerdown', (e) => {
  dragging = true;
  el.setPointerCapture(e.pointerId);
  startX = e.clientX; startY = e.clientY;
  const r = el.getBoundingClientRect(); ox = r.left; oy = r.top;
});

el.addEventListener('pointermove', (e) => {
  if (!dragging) return;
  const dx = e.clientX - startX, dy = e.clientY - startY;
  el.style.transform = `translate(${dx}px, ${dy}px)`;
});

el.addEventListener('pointerup', () => {
  dragging = false;
  el.releasePointerCapture?.();
  // 스냅/영역 판정 후 원위치 또는 고정 등
});
```

권장 라이브러리
- `interact.js`, `draggable`(GSAP), `SortableJS`(모바일 지원), 프레임워크별 DnD(React DnD, Vue Draggable 등)

---

## 9. 보안·프라이버시 고려사항

- 파일 드롭: 사용자가 실제로 드롭한 파일 핸들에만 접근 가능. 임의 경로/다른 파일 접근 불가.
- 교차 출처 데이터: 브라우저는 임의의 민감 데이터가 드래그 중 유출되지 않도록 차단.
- 텍스트/HTML 전송 시, **신뢰 경계**를 분명히: 외부에서 온 HTML을 그대로 삽입하지 말고 반드시 **sanitize**.

---

## 10. 브라우저별 팁과 함정

- Firefox: `dragstart`에 `setData`가 없으면 일반 요소 드래그 불가한 경우가 있음(관용적으로 `text/plain` 설정 권장).
- Safari: `dropEffect` 반영이 다르게 보일 수 있음. 고스트 이미지가 다소 흐릿하게 보일 수 있음.
- 링크/이미지의 기본 드래그가 원치 않을 경우 `draggable="false"` 명시.
- 복잡한 레이아웃에서 `dragleave`는 버블 경로상 자주 발생하므로 **하이라이트 토글** 로직을 신중히(컨테이너 안에서의 자식 간 이동 시 클래스가 깜빡이는 현상 방지).

---

## 11. 테스트 시나리오 체크리스트

- 마우스만으로 드래그/드롭 가능한가?
- 키보드/스크린리더 사용자 대체 수단이 있는가?
- 드래그 중 시각 피드백(고스트, 타깃 하이라이트)이 일관적인가?
- `dragover` 스로틀로 스크롤/리플로우가 부드러운가?
- 파일 드롭 시 타입/용량 제한, 에러 메시지 제공하는가?
- 모바일 터치에서 합리적인 폴백(포인터 기반 DnD 또는 버튼 기반 재배치)이 있는가?

---

## 12. 미니 프로젝트 종합 예제 — 카드 정렬 + 파일 드롭 업로더가 섞인 보드

```html
<style>
  .board { display:grid; grid-template-columns:repeat(3, 1fr); gap:16px; max-width:960px; margin:24px auto; }
  .col { background:#f7f9fb; border:1px solid #e5e9ef; border-radius:10px; padding:12px; min-height:220px; }
  .col.dragover { outline:2px dashed #4c9aff; }
  .card { background:#fff; border:1px solid #dde3ea; border-radius:8px; padding:10px; margin:8px 0; cursor:move; user-select:none; }
  .uploader { border:2px dashed #b6c2cf; border-radius:8px; padding:10px; text-align:center; color:#6b778c; }
  .uploader.over { border-color:#4c9aff; background:#f6fbff; color:#1d4ed8; }
</style>

<div class="board" id="board">
  <section class="col" id="todo" aria-label="To Do">
    <div class="uploader" tabindex="0">여기에 파일 드롭 → 카드 생성</div>
    <div class="card" draggable="true">일감 A</div>
    <div class="card" draggable="true">일감 B</div>
  </section>
  <section class="col" id="doing" aria-label="Doing">
    <div class="uploader" tabindex="0">드롭하여 카드 만들기</div>
    <div class="card" draggable="true">일감 C</div>
  </section>
  <section class="col" id="done" aria-label="Done">
    <div class="uploader" tabindex="0">드롭하여 카드 만들기</div>
  </section>
</div>

<script>
const board = document.getElementById('board');
let dragging = null;

board.addEventListener('dragstart', (e) => {
  const card = e.target.closest('.card');
  if (!card) return;
  dragging = card;
  e.dataTransfer.effectAllowed = 'move';
  e.dataTransfer.setData('text/plain', JSON.stringify({ type:'card', text: card.textContent }));
  requestAnimationFrame(() => card.style.opacity = .5);
});
board.addEventListener('dragend', () => {
  if (dragging) dragging.style.opacity = '';
  dragging = null;
});

board.querySelectorAll('.col').forEach(col => {
  col.addEventListener('dragover', (e) => { e.preventDefault(); e.dataTransfer.dropEffect = 'move'; });
  col.addEventListener('dragenter', () => col.classList.add('dragover'));
  col.addEventListener('dragleave', () => col.classList.remove('dragover'));
  col.addEventListener('drop', (e) => {
    e.preventDefault(); col.classList.remove('dragover');
    // 카드로서의 드롭
    try {
      const data = JSON.parse(e.dataTransfer.getData('text/plain') || '{}');
      if (data.type === 'card' && dragging) { col.appendChild(dragging); return; }
    } catch {}
  });

  // 파일 드롭으로 카드 생성
  const up = col.querySelector('.uploader');
  ['dragenter','dragover'].forEach(ev => up.addEventListener(ev, (e)=>{ e.preventDefault(); up.classList.add('over'); }));
  ;['dragleave','drop'].forEach(ev => up.addEventListener(ev, (e)=>{ e.preventDefault(); if (ev==='dragleave') up.classList.remove('over'); }));
  up.addEventListener('drop', (e) => {
    up.classList.remove('over');
    const files = [...e.dataTransfer.files||[]].slice(0,5);
    files.forEach(f => {
      const card = document.createElement('div');
      card.className = 'card'; card.draggable = true;
      card.textContent = `파일: ${f.name}`;
      col.appendChild(card);
    });
  });
});
</script>
```

설명
- 같은 보드 안에서 카드 이동(드래그)과 파일 드롭(카드 생성)이 공존.
- `dataTransfer`를 카드/파일 분기 처리.
- 단일 이벤트 바인딩으로 컬럼마다 동일 동작을 재사용.

---

## 13. 요약

- 핵심 3요소: **소스(draggable)**, **타깃(dragover에서 preventDefault)**, **DataTransfer 모델**.
- 파일 드롭은 `files/items`로 접근, 폴더는 별도 API 필요.
- 시각 피드백: 고스트 이미지, 타깃 하이라이트, 플레이스홀더.
- 정렬은 순수 DnD로 가능하나, 복잡해지면 라이브러리 사용.
- 접근성: 키보드 대체, ARIA 상태, 라이브 리전 제공.
- 모바일: HTML5 DnD 대신 포인터/터치 기반 커스텀 DnD 권장.
- 성능: `dragover` 최소화, 레이아웃/리플로우 억제, GC 줄이기.
- 보안: 외부 입력은 검증/정화, 파일은 사용자 드롭 범위 내에서만 접근.

---

## 14. 참고 구현 체크포인트

- [ ] `dragover`에서 `preventDefault()` 호출
- [ ] `dragstart`에서 `setData('text/plain', ...)` 호출(호환성)
- [ ] `effectAllowed`/`dropEffect` 일관성
- [ ] 사용자 지정 고스트 이미지(`setDragImage`)
- [ ] 파일 타입/용량 검증, 오류 피드백
- [ ] 키보드 재배치 대안 제공
- [ ] 모바일 폴백(포인터/터치 DnD 또는 버튼 기반 이동)
- [ ] 텍스트 선택 방지(`user-select: none`)
- [ ] 성능(스로틀, 최소 계산, 오브젝트 재사용)

---

## 15. 자주 묻는 질문

1) 왜 `drop`이 발생하지 않나요?
→ `dragover`에서 `event.preventDefault()`를 호출하지 않으면 기본 거부 상태라 `drop`이 발생하지 않습니다.

2) Firefox에서 드래그가 시작되지 않아요.
→ `dragstart`에서 `dataTransfer.setData`를 호출하세요.

3) 모바일에서는 작동이 불안정합니다.
→ 표준 DnD 대신 포인터/터치 이벤트로 커스텀 DnD를 구현하거나 전용 라이브러리를 사용하세요.

4) 드래그 중 텍스트가 선택돼 UI가 어색합니다.
→ 드래그 가능한 카드/영역에 `user-select: none`을 적용하세요.

5) 외부에서 드롭된 HTML을 그대로 넣어도 되나요?
→ XSS 위험이 있으니 반드시 sanitize 후 사용하세요.

---

본 가이드를 토대로, 단순 이동부터 파일 업로드, 보드 정렬, 모바일/접근성 대응까지 **안전하고 일관된 Drag and Drop UX**를 설계할 수 있습니다.
