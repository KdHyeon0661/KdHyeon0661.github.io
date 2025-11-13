---
layout: post
title: Svelte - Svelte 기초 문법 (2)
date: 2025-09-28 17:25:23 +0900
category: Svelte
---
# 2. Svelte 기초 문법 (2)
**클래스/스타일 바인딩 · 슬롯(slot)/프래그먼트(fragment) · 라이프사이클(onMount/beforeUpdate/afterUpdate/onDestroy) · 액션(actions)**

> 이 장은 **컴포넌트 실사용에서 꼭 쓰는 4대 기능**을 예제로 정리한다.
> 1) **클래스/스타일 바인딩**으로 상태 → 뷰를 정확히 연결하고,
> 2) **슬롯/프래그먼트**로 재사용 가능한 레이아웃을 만들고,
> 3) **라이프사이클 훅**으로 DOM 시점에 맞게 부작용을 수행하며,
> 4) **액션(use:action)**으로 “DOM에 부착되는 동작”을 캡슐화한다.

---

## 2.1 클래스/스타일 바인딩

Svelte는 CSS 클래스/스타일을 **데이터에 반응**하도록 아주 간단하게 바인딩할 수 있다.
핵심 문법:

- **클래스**
  - `class={expr}`
  - `class:active={cond}` → 조건부 클래스 (여러 개 가능)
- **스타일**
  - `style="..."` + `{}` 표현식
  - `style:width={value}` 처럼 **단일 속성**에 바인딩

### 2.1.1 조건부 클래스 — `class:NAME={조건}`
```svelte
<script>
  let active = false;
  let error = '';
  function toggle() { active = !active; }
</script>

<button
  class="btn"
  class:active={active}
  class:error={!!error}
  on:click={toggle}
>
  {active ? 'Active' : 'Inactive'}
</button>

<style>
  .btn { padding: .6rem 1rem; border-radius: .4rem; border: 1px solid #ccc; }
  .btn.active { background: #0c7; color: white; }
  .btn.error { border-color: #e33; }
</style>
```

- `class:active={active}`는 `active === true`일 때만 `.active` 클래스를 추가한다.
- 여러 조건부 클래스를 **나란히** 쓸 수 있어 가독성이 좋다.

### 2.1.2 클래스 문자열 계산 — `class={expr}`
```svelte
<script>
  let size = 'md'; // 'sm' | 'md' | 'lg'
  let selected = true;

  $: cls = [
    'tag',
    size === 'sm' ? 'tag--sm' : size === 'lg' ? 'tag--lg' : 'tag--md',
    selected && 'tag--selected'
  ].filter(Boolean).join(' ');
</script>

<span class={cls}>Chip</span>

<style>
  .tag { padding: .2rem .6rem; border-radius: 999px; border: 1px solid #aaa; }
  .tag--sm { font-size: .8rem; }
  .tag--md { font-size: 1rem; }
  .tag--lg { font-size: 1.2rem; }
  .tag--selected { background: #222; color: #fff; }
</style>
```

- 복잡한 조합은 배열 → `filter(Boolean)` → `join(' ')` 패턴이 깔끔하다.
- **반응식 `$:`**로 `cls`를 계산하면 상태가 바뀔 때 자동으로 갱신된다.

### 2.1.3 스타일 바인딩 — 인라인/단일 속성
```svelte
<script>
  let progress = 30; // 0~100
  let color = '#0c7';
  let px = 120;
</script>

<div class="bar-wrap">
  <div class="bar" style:width={`${progress}%`} style:background={color}></div>
</div>

<div class="box" style={`width:${px}px; height:${px}px;`}></div>

<style>
  .bar-wrap { width: 100%; background: #eee; height: 10px; border-radius: 8px; overflow: hidden; }
  .bar { height: 100%; transition: width .2s ease; }
  .box { background: #f2f2f2; border: 1px dashed #bbb; margin-top: .6rem; }
</style>
```

- `style:width={...}` 처럼 **속성 단위**로 묶으면 직관적이다.
- 여러 속성을 한 번에 넣고 싶다면 `style={\`...\`}` 문자열로 처리한다.

### 2.1.4 상태에 따른 트랜지션/애니메이션과 궁합
클래스 변화를 **CSS 트랜지션**과 함께 쓰면 상태 전환이 부드럽다.
```svelte
<script>
  let open = false;
</script>

<button on:click={() => open = !open}>Toggle</button>
<div class:panel-open={open} class="panel">Content</div>

<style>
  .panel {
    height: 0;
    overflow: hidden;
    transition: height .25s ease;
  }
  .panel.panel-open {
    height: 100px;
  }
</style>
```

---

## 2.2 슬롯(slot)과 프래그먼트(fragment)

**슬롯**은 “부모가 자식 컴포넌트 내부의 특정 위치에 콘텐츠를 주입”하는 메커니즘이다.
**프래그먼트(svelte:fragment)**는 **여러 노드**를 하나의 슬롯 위치에 묶어서 넣고 싶을 때 사용한다.

### 2.2.1 기본 슬롯(기본 콘텐츠 + 대체 콘텐츠)
```svelte
<!-- Card.svelte -->
<script>
  export let title = 'Untitled';
</script>

<article class="card">
  <h3>{title}</h3>
  <slot>
    <!-- 부모가 아무것도 전달하지 않으면 이 내용이 보임(대체 콘텐츠) -->
    <p>Empty…</p>
  </slot>
</article>

<style>
  .card { border: 1px solid #ddd; border-radius: .6rem; padding: 1rem; }
  h3 { margin: 0 0 .6rem; }
</style>
```

```svelte
<!-- 사용 -->
<script>
  import Card from '$lib/components/Card.svelte';
</script>

<Card title="Hello">
  <p>이 내용은 부모가 주입한 기본 슬롯 콘텐츠입니다.</p>
</Card>

<Card title="No Content" />
```

### 2.2.2 **이름 있는 슬롯** — `<slot name="...">` + `slot="..."` 전달
```svelte
<!-- Dialog.svelte -->
<dialog class="dlg">
  <header><slot name="header">Header</slot></header>
  <section class="body"><slot /></section>
  <footer><slot name="footer"><button>Close</button></slot></footer>
</dialog>

<style>
  .dlg { border: none; border-radius: .6rem; box-shadow: 0 6px 22px #0002; padding: 0; }
  header, footer { padding: .8rem 1rem; background: #f7f7f7; }
  .body { padding: 1rem; }
</style>
```

```svelte
<script>
  import Dialog from '$lib/components/Dialog.svelte';
</script>

<Dialog>
  <svelte:fragment slot="header">
    <h3>제목</h3>
  </svelte:fragment>

  <!-- 기본 슬롯 -->
  <p>본문 컨텐츠입니다.</p>

  <svelte:fragment slot="footer">
    <button>확인</button>
    <button>취소</button>
  </svelte:fragment>
</Dialog>
```

- 부모는 특정 슬롯 위치에 콘텐츠를 넣기 위해 **`slot="name"`**을 지정한다.
- 전달할 요소가 **여러 개**일 때 `<svelte:fragment slot="name">…</svelte:fragment>`로 묶는다.

### 2.2.3 **슬롯 props** — 자식 → 부모로 데이터 넘기기
```svelte
<!-- List.svelte -->
<script>
  export let items = [];
</script>

<ul>
  {#each items as it, i}
    <!-- 자식이 각 항목을 slot props로 노출 -->
    <slot name="row" {it} {i}>
      <!-- 부모가 row 슬롯을 제공하지 않으면 기본 렌더 -->
      <li>{i+1}. {it}</li>
    </slot>
  {/each}
</ul>
```

```svelte
<script>
  import List from '$lib/components/List.svelte';
  const colors = ['red', 'green', 'blue'];
</script>

<List {items}= {colors}>
  <!-- parent는 let:으로 slot props를 받는다 -->
  <svelte:fragment slot="row" let:it let:i>
    <li style="color:{it}">{i+1}. {it.toUpperCase()}</li>
  </svelte:fragment>
</List>
```

- 슬롯은 **단방향**: 자식이 “슬롯에 노출할 값”을 정의하고, 부모가 `let:`으로 받는다.
- 이 패턴으로 **완전히 커스터마이즈 가능한 리스트/테이블**을 만들 수 있다.

### 2.2.4 프래그먼트 응용: 여러 조각을 한 슬롯에
```svelte
<!-- Tabs.svelte -->
<nav class="tabs">
  <slot name="tablist" />
</nav>
<section class="panel">
  <slot />
</section>
```

```svelte
<script>
  import Tabs from '$lib/components/Tabs.svelte';
</script>

<Tabs>
  <svelte:fragment slot="tablist">
    <button>Tab1</button>
    <button>Tab2</button>
    <button>Tab3</button>
  </svelte:fragment>

  <p>패널 콘텐츠 1</p>
  <p>패널 콘텐츠 2</p>
</Tabs>
```

> **참고**: Svelte 컴포넌트 파일은 **여러 최상위 노드**를 가질 수 있다(특별한 `<Fragment>` 컴포넌트가 필요하지 않음). “프래그먼트”라는 표현은 **슬롯 전송 시** 여러 노드를 하나의 슬롯에 담아 보내는 `<svelte:fragment>` 용도를 가리킨다.

---

## 2.3 라이프사이클 훅 — onMount / beforeUpdate / afterUpdate / onDestroy

Svelte 컴포넌트의 DOM 수명에 맞춰 실행되는 훅:

- **`onMount(fn)`**: **클라이언트에서** 컴포넌트가 마운트된 직후(첫 렌더 뒤) 한 번 실행
- **`beforeUpdate(fn)`**: DOM 업데이트 **직전**(상태 반영되기 전) 매번 실행
- **`afterUpdate(fn)`**: DOM 업데이트 **직후** 매번 실행 (레이아웃 측정 등)
- **`onDestroy(fn)`**: 컴포넌트가 파괴되기 직전 한 번 (정리/해제)

> **중요**: 서버 렌더링(SSR) 중에는 **`onMount`가 실행되지 않는다**. 브라우저에서만 돌기 때문에 **브라우저 API** 사용은 `onMount` 내부에서 하라.

### 2.3.1 onMount — 브라우저 API/비동기 초기화
```svelte
<script>
  import { onMount } from 'svelte';

  let width = 0;

  onMount(() => {
    const resize = () => { width = window.innerWidth; };
    resize(); // 초기 측정
    window.addEventListener('resize', resize);
    // 정리 함수 반환 → onDestroy와 동등
    return () => window.removeEventListener('resize', resize);
  });
</script>

<p>Width: {width}px</p>
```

- **클린업 함수**를 반환하면 `onDestroy` 시점에 자동 호출된다.
- 네트워크 데이터 **초기 fetch**도 보통 여기서 수행한다.

### 2.3.2 beforeUpdate / afterUpdate — DOM 변경 전·후 타이밍
```svelte
<script>
  import { beforeUpdate, afterUpdate } from 'svelte';

  let count = 0;
  let before = 0;
  let after = 0;

  beforeUpdate(() => { before++; });
  afterUpdate(() => { after++; });
</script>

<button on:click={() => count++}>+1</button>
<p>count: {count}</p>
<p>beforeUpdate called: {before}</p>
<p>afterUpdate called: {after}</p>
```

- **측정·로그** 등 변경 타이밍을 알아야 하는 경우 유용하다.
- 레이아웃/크기 측정은 **afterUpdate**가 안전하다(실제 DOM 반영 이후).

### 2.3.3 onDestroy — 타이머/옵저버/구독 해제
```svelte
<script>
  import { onDestroy } from 'svelte';

  const id = setInterval(() => {
    console.log('tick');
  }, 1000);

  onDestroy(() => {
    clearInterval(id);
  });
</script>

<p>콘솔에 매초 tick</p>
```

- 이벤트/타이머/옵저버/스토어 구독 등 **누수 가능성**이 있으면 **반드시 정리**한다.

### 2.3.4 `tick` — 업데이트 플러시 이후를 기다리기
```svelte
<script>
  import { tick } from 'svelte';

  let open = false;
  let el;

  async function showAndFocus() {
    open = true;   // 상태 변경 → DOM 업데이트 예약
    await tick();  // DOM 업데이트가 끝날 때까지 대기
    el?.focus();   // 이제 접근 가능
  }
</script>

<button on:click={showAndFocus}>Open</button>
{#if open}
  <input bind:this={el} placeholder="now focusable" />
{/if}
```

- `tick()`은 “지금 큐에 쌓인 DOM 변경이 **실제로 반영된 뒤**를 기다리는 Promise”다.

### 2.3.5 SSR 주의 요약
- **DOM/윈도우 접근**은 `onMount` 안에서만.
- `beforeUpdate/afterUpdate`는 클라이언트 렌더링 과정에서만 의미가 있다.
- 데이터는 SvelteKit의 `load`에서 미리 받아 SSR로 **완성된 HTML**을 보내는 게 일반적(본 장의 포커스는 컴포넌트 생명주기).

---

## 2.4 액션(actions) 소개 — `use:action`

**액션**은 “DOM 요소에 **부착되어 동작**하는 함수”다.
형식:
```ts
function action(node: HTMLElement, params?: any) {
  // 초기화
  return {
    update(newParams) { /* 파라미터가 바뀌면 호출 */ },
    destroy() { /* 노드 제거 시 정리 */ }
  };
}
```

컴포넌트 마크업에서 `use:action={params}`로 **부착**한다.

### 2.4.1 가장 단순한 액션 — 자동 포커스
```svelte
<!-- actions/autoFocus.js -->
export function autoFocus(node) {
  node.focus();
  // 반환이 없어도 됨(정리할 게 없으면)
}
```

```svelte
<script>
  import { autoFocus } from '$lib/actions/autoFocus.js';
</script>

<input use:autoFocus placeholder="mounted → focus" />
```

### 2.4.2 파라미터가 있는 액션 — 클릭 바깥 감지
```svelte
<!-- actions/clickOutside.ts -->
export function clickOutside(node: HTMLElement, callback: () => void) {
  const onDoc = (e: MouseEvent) => {
    if (!node.contains(e.target as Node)) callback();
  };
  document.addEventListener('click', onDoc);

  return {
    destroy() {
      document.removeEventListener('click', onDoc);
    }
  };
}
```

```svelte
<script lang="ts">
  import { clickOutside } from '$lib/actions/clickOutside';
  let open = true;
</script>

<div class="menu" use:clickOutside={() => open = false}>
  <p>메뉴</p>
</div>
{#if !open}<button on:click={() => open = true}>Open</button>{/if}

<style>
  .menu { border: 1px solid #ccc; padding: .6rem; }
</style>
```

- `destroy()`에서 **리스너 해제**로 누수를 방지한다.

### 2.4.3 반응 파라미터 — `update`로 재설정
```svelte
<!-- actions/tooltip.ts -->
type Params = { text: string; placement?: 'top'|'right'|'bottom'|'left' };

export function tooltip(node: HTMLElement, params: Params) {
  let title = document.createElement('div');
  title.className = 'tooltip';
  node.addEventListener('mouseenter', show);
  node.addEventListener('mouseleave', hide);
  apply(params);

  function apply(p: Params) {
    title.textContent = p.text;
  }
  function show() {
    document.body.appendChild(title);
    const rect = node.getBoundingClientRect();
    title.style.position = 'fixed';
    title.style.top = rect.top - 28 + 'px';
    title.style.left = rect.left + 'px';
  }
  function hide() {
    title.remove();
  }

  return {
    update(p: Params) { apply(p); },
    destroy() {
      hide();
      node.removeEventListener('mouseenter', show);
      node.removeEventListener('mouseleave', hide);
    }
  };
}
```

{% raw %}
```svelte
<script lang="ts">
  import { tooltip } from '$lib/actions/tooltip';
  let msg = '툴팁입니다';
</script>

<button use:tooltip={{ text: msg }}>Hover</button>
<input bind:value={msg} placeholder="툴팁 메시지" />

<style>
  .tooltip {
    background: #222; color: #fff; padding: .2rem .4rem; border-radius: .3rem;
    font-size: .85rem; pointer-events: none;
  }
</style>
```
{% endraw %}

- 부모에서 `msg`가 바뀌면 **액션의 `update`**가 호출되어 DOM을 갱신한다.

### 2.4.4 고급 액션 예: IntersectionObserver로 **무한 스크롤**
```svelte
<!-- actions/intersect.ts -->
export function intersect(node: Element, onEnter: () => void) {
  const io = new IntersectionObserver((entries) => {
    if (entries.some(e => e.isIntersecting)) onEnter();
  });
  io.observe(node);

  return {
    destroy() { io.disconnect(); }
  }
}
```

```svelte
<script>
  import { intersect } from '$lib/actions/intersect';

  let items = Array.from({ length: 20 }, (_, i) => `row-${i+1}`);

  async function loadMore() {
    await new Promise(r => setTimeout(r, 400));
    const next = items.length;
    items = [...items, ...Array.from({ length: 20 }, (_, i) => `row-${next + i + 1}`)];
  }
</script>

<ul>
  {#each items as it (it)}
    <li>{it}</li>
  {/each}
</ul>

<!-- sentinel: 화면에 들어오면 loadMore 실행 -->
<div use:intersect={loadMore} style="height:1px"></div>
```

- 스크롤 **끝에 닿을 때마다** 더 로드하는 패턴을 **한 줄(use:intersect)** 로 재사용할 수 있다.

### 2.4.5 여러 액션 동시 부착
{% raw %}
```svelte
<input use:autoFocus use:tooltip={{ text: '자동 포커스 됩니다' }} />
```
{% endraw %}
- 여러 `use:`를 **띄어쓰기**로 나란히 적으면 된다.

### 2.4.6 액션의 장점 요약
- **DOM 지향 동작**(포커스, 드래그, 관찰자, 바깥클릭 등)을 **캡슐화**
- 다른 컴포넌트에서도 **그대로 재사용**
- **정리(destroy)** 타이밍이 명확하므로 누수 위험 낮음

---

## 2.5 종합 실습 — “Modal + Slot + Action + Lifecycle”

아래는 지금까지 배운 개념을 모두 쓰는 **모달 컴포넌트**다.

### 2.5.1 액션: ESC 키 닫기
```svelte
<!-- actions/esc.ts -->
export function esc(node, callback) {
  const onKey = (e) => { if (e.key === 'Escape') callback(); };
  window.addEventListener('keydown', onKey);
  return { destroy: () => window.removeEventListener('keydown', onKey) };
}
```

### 2.5.2 컴포넌트: Modal.svelte
```svelte
<script>
  import { onMount, onDestroy } from 'svelte';
  import { esc } from '$lib/actions/esc';

  export let open = false;
  export let title = 'Modal';

  let dlg;

  // onMount: 열린 상태면 showModal
  onMount(() => {
    if (open) dlg?.showModal();
  });

  // open이 외부에서 바뀌면 반응
  $: if (dlg) {
    if (open && !dlg.open) dlg.showModal();
    if (!open && dlg.open) dlg.close();
  }

  function closeByBackdrop(e) {
    if (e.target === dlg) {
      open = false;
    }
  }

  onDestroy(() => {
    if (dlg?.open) dlg.close();
  });
</script>

<dialog bind:this={dlg} class="modal" on:click={closeByBackdrop} use:esc={() => open = false}>
  <header class="hd">
    <slot name="header">
      <h3>{title}</h3>
    </slot>
    <button class="x" on:click={() => open = false}>×</button>
  </header>

  <section class="body">
    <slot>
      <p>내용이 없습니다.</p>
    </slot>
  </section>

  <footer class="ft">
    <slot name="footer">
      <button on:click={() => open = false}>확인</button>
    </slot>
  </footer>
</dialog>

<style>
  .modal { border: none; border-radius: .6rem; padding: 0; width: min(560px, 92vw); }
  .hd, .ft { display: flex; justify-content: space-between; align-items: center; padding: .8rem 1rem; background: #f6f6f6; }
  .body { padding: 1rem; }
  .x { background: transparent; border: none; font-size: 1.2rem; cursor: pointer; }
</style>
```

- **라이프사이클**: `onMount`에서 초기 상태 반영, `onDestroy`에서 닫기 정리
- **액션**: `use:esc`로 ESC 키 닫기
- **슬롯**: `header`/기본/`footer` 이름 슬롯 지원
- **조건부 로직**: `$:`로 `open` 변화 → `showModal/close` 호출

### 2.5.3 사용 예
```svelte
<script>
  import Modal from '$lib/components/Modal.svelte';
  let open = false;
</script>

<button on:click={() => open = true} class:active={open}>Open modal</button>

<Modal bind:open {open} title="알림">
  <svelte:fragment slot="header">
    <h3>사용자 알림</h3>
  </svelte:fragment>

  <p>이것은 모달 본문입니다. ESC 또는 바깥 클릭으로 닫습니다.</p>

  <svelte:fragment slot="footer">
    <button on:click={() => open = false}>닫기</button>
    <button style="background:#0c7;color:#fff" on:click={() => (open = false)}>확인</button>
  </svelte:fragment>
</Modal>
```

- 버튼의 `class:active={open}`으로 상태와 스타일을 연동
- 슬롯으로 헤더/푸터를 자유롭게 커스터마이즈

---

## 2.6 흔한 함정 · 베스트 프랙티스

1) **조건부 클래스 계산이 지저분**
   → `class:foo={cond}`를 **여러 줄**로 쓰거나 **배열 → join** 패턴 사용.

2) **슬롯 props 방향 착각**
   → 데이터는 **자식이 제공**, 부모가 `let:`으로 **수신**.

3) **브라우저 API를 SSR에서 호출**
   → `onMount`에서만 사용. 또는 `typeof window !== 'undefined'` 가드.

4) **액션 정리 누락(메모리 누수)**
   → 이벤트/옵저버는 **반드시 destroy**로 해제.

5) **라이프사이클 타이밍 오용**
   - 레이아웃 측정/DOM 의존 작업은 **afterUpdate**나 `await tick()` 이후.
   - `beforeUpdate`는 “렌더 직전” 상태를 보고 싶을 때만.

6) **스타일 바인딩 단위 누락**
   → `style:width={`${n}%`}` 처럼 **단위 포함**.

---

## 2.7 체크리스트 (요점 정리)

- [ ] `class:NAME={cond}` / `class={expr}` 로 조건부/동적 클래스
- [ ] `style:prop={value}` / `style="…"` 로 반응 스타일
- [ ] `<slot>` 기본 + `<slot name="…">` 이름 슬롯, **대체 콘텐츠** 설계
- [ ] 부모에서 `<svelte:fragment slot="…">` 로 **여러 노드** 전달
- [ ] 슬롯 props: 자식 → 부모(`let:…`) 데이터 전달
- [ ] 라이프사이클: `onMount`(브라우저 전용), `beforeUpdate/afterUpdate`, `onDestroy`
- [ ] `tick()`으로 DOM 업데이트 완료를 기다린 뒤 조작
- [ ] 액션: `use:action={params}` / `update` / `destroy` 패턴, 재사용 가능한 DOM 동작
