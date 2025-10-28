---
layout: post
title: Svelte - Svelte 기초 문법 (1)
date: 2025-09-28 16:25:23 +0900
category: Svelte
---
# 2. Svelte 기초 문법 (1)
**파일 구조와 단일 파일 컴포넌트 · 반응성(할당 기반, `$:` 반응 구문) · 템플릿 구문(`{}`/`if`/`each`/`await`) · 이벤트 바인딩 · `bind:` 양방향 바인딩**

> 이 장은 **Svelte 파일 구조와 단일 파일 컴포넌트(SFC)**를 출발점으로, **반응성의 핵심(할당 기반 업데이트, `$:` 반응 구문)**, 그리고 **템플릿 구문**을 촘촘히 다룬다. 모든 코드는 **곧장 붙여넣어 실행**해볼 수 있게 작성한다.

---

## 2.1 파일 구조와 단일 파일 컴포넌트(SFC)

### 2.1.1 Svelte 프로젝트에서의 기본 디렉터리(Kit 기준)
```
my-app/
  src/
    lib/                 # 공용 컴포넌트/유틸(=별칭 $lib)
      components/
        Button.svelte
      utils/
        format.ts
    routes/              # 라우트(페이지)
      +layout.svelte
      +page.svelte       # 홈(/)
      about/
        +page.svelte     # /about
  static/                # 정적 파일(이미지, 폰트 등)
  svelte.config.ts
  vite.config.ts
  tsconfig.json
  package.json
```
- **SvelteKit**: 라우팅은 `src/routes` 폴더 구조로 결정된다.
- **공용 컴포넌트**는 `src/lib`에 두고 `$lib/*` 별칭으로 임포트하는 관례를 쓴다.

### 2.1.2 단일 파일 컴포넌트 구조
Svelte의 컴포넌트 파일은 `.svelte` 확장자를 가지며, **필수는 아니지만** 보통 다음 3부로 구성된다.
1) `<script>`: 로직, 상태, props 선언  
2) **마크업(HTML 템플릿)**: DOM 구조, `{}` 표현식  
3) `<style>`: 해당 컴포넌트에 스코프된 CSS

```svelte
<!-- src/lib/components/Hello.svelte -->
<script>
  export let name = 'world'; // prop
</script>

<h1>Hello {name}!</h1>

<style>
  h1 { color: #0c7; }
</style>
```

- **스코프 CSS**: 기본적으로 이 컴포넌트의 `<h1>`에만 적용된다. Svelte가 빌드시 내부적으로 해시 클래스를 부여해 충돌을 막아준다.
- **prop 선언**: `export let ...` 으로 외부에서 전달받을 수 있는 값을 정의한다.

### 2.1.3 컴포넌트 사용
```svelte
<!-- src/routes/+page.svelte -->
<script>
  import Hello from '$lib/components/Hello.svelte';
</script>

<Hello name="Svelte" />
```

- 태그처럼 **사용 시 대문자**로 시작해야 한다(HTML 태그와 구분).

---

## 2.2 반응성(reactivity) 기본

> Svelte의 반응성은 **“할당이 신호”**다. 값이 **재할당**되면 해당 값을 **읽는 템플릿/반응식만** 업데이트된다. 가상 DOM이 아니라 **정확히 필요한 DOM 부분**만 패치한다.

### 2.2.1 할당 기반 업데이트(핵심 규칙)
```svelte
<!-- Counter.svelte -->
<script>
  let count = 0;

  function inc() {
    count = count + 1; // 재할당(= 업데이트 트리거)
  }

  function reset() {
    count = 0;         // 재할당
  }
</script>

<button on:click={inc}>+1</button>
<button on:click={reset}>reset</button>
<p>Count: {count}</p>
```
- `count++`도 결국 재할당(`count = count + 1`)이므로 반응 업데이트가 트리거된다.
- **주의**: 객체/배열을 **직접 변형**하면(예: `arr.push`) 재할당이 아닌 변경이라 **업데이트가 안 될 수 있다**. 이때는 **새 참조를 재할당** 하거나 v5의 deep reactivity(아래 참고)를 활용한다.

#### 패턴 A: 새 배열/객체로 재할당
```svelte
<script>
  let items = ['a', 'b'];

  function addItem(v) {
    // ✅ 새 배열을 만들어 재할당
    items = [...items, v];
  }
</script>
```

#### (Svelte 5) 패턴 B: `$state`의 깊은 반응성(개념 소개)
> v5에서는 `$state`를 쓰면 **중첩 속성 변경**도 반응적으로 추적된다.
```svelte
<script>
  // v5 전용 문법 예시(설명 목적)
  let cart = $state([{ name: 'Book', price: 12 }]);
  function add() {
    cart.push({ name: 'Pen', price: 3 }); // push도 반응
  }
</script>

<button on:click={add}>Add</button>
<p>Total: {cart.reduce((s, it) => s + it.price, 0)}</p>
```
- 본 장에서는 **v4 스타일(재할당 중심)**을 기본으로 설명하고, v5 확장 포인트를 필요 시 병기한다.

### 2.2.2 `$:` 반응 구문(reactive declarations)
**특정 값이 바뀔 때 함께 재계산**되어야 하는 파생값/부작용은 `$:` 라벨을 사용한다.

```svelte
<script>
  let count = 0;

  $: double = count * 2; // count가 바뀔 때마다 double 재계산

  function inc() {
    count += 1;
  }
</script>

<button on:click={inc}>+1</button>
<p>{count} (x2 = {double})</p>
```
- `$:` 선언은 **의존하는 변수**가 바뀔 때마다 다시 실행된다.
- `$:` 는 **선언형**이다. “count가 바뀌면 double을 재계산해” 라고 **의미**를 적는다고 생각하면 된다.

#### 2.2.2.1 반응 구문에서의 부작용
부작용(예: 콘솔 로그, 네트워크 요청)은 별도 `$:` 블록으로 명시한다.
```svelte
<script>
  let q = '';

  // 파생값: 부작용 없는 계산
  $: trimmed = q.trim();

  // 부작용: 값 변화 로그
  $: {
    console.log('query changed:', q);
  }
</script>

<input bind:value={q} placeholder="Type..." />
<p>Trimmed: {trimmed}</p>
```
- **권장**: 계산과 부작용을 **분리**해 읽기 좋게 유지한다.

#### 2.2.2.2 실행 순서/배치
- 같은 틱(이벤트 루프 사이클) 내에서 **여러 할당이 일어나면** Svelte가 **최소한의 재계산**만 수행한다(배치).  
- `$:`는 **아래쪽에 있어도** Svelte가 의존성 그래프를 알고 있기 때문에 **정확한 순서**로 평가된다.

```svelte
<script>
  let a = 1, b = 2;

  function bump() {
    a = a + 1;
    b = b + 1;
    // 두 값 모두 이 틱 내에서 바뀌며,
    // 아래 반응식은 한번의 적절한 타이밍으로 재평가된다.
  }

  $: sum = a + b;
</script>

<button on:click={bump}>bump</button>
<p>{a} + {b} = {sum}</p>
```

### 2.2.3 구조분해/계산 시 주의점
- **구조분해 후 재할당 누락**: 반응성을 유지하려면 **원본 상태를 재할당**하거나 v5에서 `$state`의 deep reactivity를 사용.
```svelte
<script>
  let user = { name: 'Ada', age: 30 };

  function birthdayWrong() {
    const { age } = user;
    // age++ 해도 user가 바뀌는 게 아니라서 템플릿 갱신 X
  }

  function birthdayRight() {
    user = { ...user, age: user.age + 1 }; // ✅ 새 객체로 재할당
  }
</script>
```

---

## 2.3 템플릿 구문 — `{}`/제어 블록/비동기/이벤트/바인딩

> Svelte는 템플릿 내에서 **JS 표현식**을 `{}`로 렌더링하고, **제어 블록**(`{#if}`, `{#each}`, `{#await}`)과 **이벤트 바인딩**, **`bind:` 바인딩**을 통해 UI-상태를 일관되게 연결한다.

### 2.3.1 `{}` 표현식
- **JS 표현식**을 텍스트 노드/속성/어트리뷰트에 바인딩 가능
- **문**이 아니라 **표현식**만 가능(예: `if (...) { ... }` 불가, 삼항은 가능)
```svelte
<script>
  let first = 'Ada';
  let last = 'Lovelace';
  let n = 3;
</script>

<p>Hello, {first + ' ' + last}!</p>
<p>{n} + 1 = {n + 1}</p>
<p>Upper: {first.toUpperCase()}</p>

<!-- 속성에서도 가능 -->
<div title={`User: ${first}`}>Hover me</div>
```

#### 2.3.1.1 안전하지 않은 HTML 넣기 — `@html`
- 기본 `{}`는 **HTML을 이스케이프**한다(안전).
- 신뢰된 HTML만 **직접 삽입**하려면 `{@html ...}`을 쓴다(**XSS 주의**).
```svelte
<script>
  // 서버에서 온 HTML을 그대로 넣고 싶을 때(출처를 신뢰할 때만!)
  let raw = '<b>bold</b> & <i>italic</i>';
</script>

<p>{@html raw}</p>
```

---

### 2.3.2 `{#if}` / `{:else if}` / `{:else}` — 조건부 렌더링
```svelte
<script>
  let loggedIn = false;
  let user = { name: 'Ada' };
</script>

{#if loggedIn}
  <p>Welcome, {user.name}!</p>
{:else}
  <p>Please log in</p>
{/if}
```

```svelte
<script>
  let score = 87;
</script>

{#if score >= 90}
  <p>Grade: A</p>
{:else if score >= 80}
  <p>Grade: B</p>
{:else}
  <p>Keep going!</p>
{/if}
```

- Svelte는 **블록 시작/끝 태그**가 명시적이라 가독성이 좋다.
- 조건이 자주 바뀌면 **해당 영역만 DOM 패치**된다.

---

### 2.3.3 `{#each}` — 반복 렌더링(키드/비키드)
```svelte
<script>
  let items = [
    { id: 1, name: 'a' },
    { id: 2, name: 'b' }
  ];
</script>

<ul>
  {#each items as it}
    <li>{it.name}</li>
  {/each}
</ul>
```

#### 2.3.3.1 keyed each(권장)
- 아이템에 **고유 키**를 부여하면 DOM 재사용이 안정적이다.
```svelte
<ul>
  {#each items as it (it.id)}
    <li>{it.name}</li>
  {/each}
</ul>
```
- **키가 없는** `each`는 인덱스 기반으로 요소를 재사용해, **중간 삽입/삭제** 시 의도치 않은 DOM 재사용이 생길 수 있다. 가능한 키를 지정하자.

#### 2.3.3.2 `index` / `else` 절
```svelte
<script>
  let items = [];
</script>

{#each items as it, i (it.id)}
  <p>{i}: {it.name}</p>
{:else}
  <p>No items</p>
{/each}
```

---

### 2.3.4 `{#await}` — 비동기 데이터 처리
```svelte
<script>
  let userPromise = fetch('/api/user').then(r => r.json());
</script>

{#await userPromise}
  <p>Loading...</p>
{:then user}
  <p>Hello {user.name}</p>
{:catch err}
  <p style="color:red">Error: {err.message}</p>
{/await}
```
- Promise 상태에 따라 **로딩/성공/실패 UI**를 분기한다.
- **주의**: SvelteKit SSR에서 `load`로 데이터를 받으면 `await` 없이 서버에서 받아 렌더하는 게 일반적이다. 하지만 컴포넌트 내부에서 비동기를 처리해야 할 때 `{#await}`가 유용하다.

---

### 2.3.5 이벤트 바인딩 — `on:click` 등 + **이벤트 수정자**
#### 2.3.5.1 기본
```svelte
<script>
  function greet() { alert('Hello'); }
  let name = 'Ada';
  function hi(n) { alert('Hi ' + n); }
</script>

<button on:click={greet}>Greet</button>
<button on:click={() => hi(name)}>Hi</button>
```

#### 2.3.5.2 이벤트 객체 사용
```svelte
<script>
  function handle(e) {
    console.log('type:', e.type);
  }
</script>

<div on:mousemove={handle} style="height:60px;background:#eee">
  Move here
</div>
```

#### 2.3.5.3 이벤트 수정자(modifiers)
- `|preventDefault` : 기본 동작 방지  
- `|stopPropagation` : 이벤트 전파 중지  
- `|once` : 한 번만 실행  
- `|capture` : 캡처 단계 리스닝  
- `|passive` : 스크롤 성능 최적화(수정자가 브라우저에 passive 알림)
- `|self` : 이벤트 타겟이 **자기 자신**일 때만

```svelte
<form on:submit|preventDefault={() => console.log('submit')}>
  <input placeholder="Enter" />
  <button type="submit">Go</button>
</form>

<div on:click={() => console.log('outer')}>
  <button on:click|stopPropagation={() => console.log('inner')}>
    inner
  </button>
</div>

<button on:click|once={() => alert('clicked once')}>Once</button>
```

---

### 2.3.6 `bind:` 양방향 바인딩(폼/요소/컴포넌트/요소 참조)

#### 2.3.6.1 `bind:value` — 텍스트 입력
```svelte
<script>
  let name = '';
</script>

<input bind:value={name} placeholder="Your name" />
<p>Hello {name || '...'}</p>
```

#### 2.3.6.2 `bind:checked` — 체크박스
```svelte
<script>
  let agree = false;
</script>

<label>
  <input type="checkbox" bind:checked={agree} />
  I agree
</label>

{#if agree}<p>Thanks!</p>{/if}
```

#### 2.3.6.3 `bind:group` — 라디오/체크박스 그룹
```svelte
<script>
  let lang = 'ts';
</script>

<label><input type="radio" bind:group={lang} value="js" /> JS</label>
<label><input type="radio" bind:group={lang} value="ts" /> TS</label>
<p>Selected: {lang}</p>
```

```svelte
<script>
  let selected = [];
  const all = ['red', 'green', 'blue'];
</script>

{#each all as c}
  <label>
    <input type="checkbox" bind:group={selected} value={c} />
    {c}
  </label>
{/each}

<p>Selected: {selected.join(', ')}</p>
```

#### 2.3.6.4 `bind:this` — DOM 요소 참조
```svelte
<script>
  let inputRef;
  function focus() {
    inputRef.focus();
  }
</script>

<input bind:this={inputRef} placeholder="Focus me with button" />
<button on:click={focus}>Focus</button>
```
- DOM API를 사용할 때 유용하다(스크롤, 사이즈 측정 등).

#### 2.3.6.5 기타 바인딩 — 미디어/상세/파일
```svelte
<script>
  let open = false;
  let files; // FileList
  let t = 0; // currentTime
</script>

<details bind:open>
  <summary>More</summary>
  <p>Detail content...</p>
</details>

<input type="file" bind:files />

<video src="/video.mp4" bind:currentTime={t} controls />
<p>Time: {t.toFixed(1)}s</p>
```

#### 2.3.6.6 컴포넌트에 대한 바인딩(Props/이벤트)
**부모 → 자식**: props로 값 전달  
**자식 → 부모**: 이벤트로 값 통지(or 바인딩 지원 컴포넌트)

```svelte
<!-- Child.svelte -->
<script>
  export let value = '';
  // 필요하면 컴포넌트에서 change 이벤트를 발생시켜 부모가 bind 가능하게 만들 수 있다.
  import { createEventDispatcher } from 'svelte';
  const dispatch = createEventDispatcher();

  function onInput(e) {
    dispatch('change', e.target.value);
  }
</script>

<input value={value} on:input={onInput} />
```

```svelte
<!-- Parent.svelte -->
<script>
  import Child from './Child.svelte';
  let v = 'hello';

  function changed(e) {
    v = e.detail; // 자식 이벤트 payload
  }
</script>

<Child {value} on:change={changed} />
<p>Parent: {v}</p>
```

> **고급**: 컴포넌트가 `export let value`와 함께 **`bind:value`**를 지원하도록 작성할 수 있다(컴포넌트 설계 주제이므로 후술).

---

## 2.4 미니 실습 — “폼 + 리스트 + 파생값 + 키드 each”

다음은 본 절의 개념을 모두 엮은 작은 예제다.

```svelte
<!-- src/routes/+page.svelte -->
<script>
  let text = '';
  let items = [];

  function add() {
    const v = text.trim();
    if (!v) return;
    // 재할당 패턴으로 반응성 보장
    items = [...items, { id: crypto.randomUUID(), title: v, done: false }];
    text = '';
  }

  function toggle(id) {
    items = items.map(it => it.id === id ? { ...it, done: !it.done } : it);
  }

  function remove(id) {
    items = items.filter(it => it.id !== id);
  }

  $: doneCount = items.filter(it => it.done).length;
  $: leftCount = items.length - doneCount;
</script>

<h1>Todos</h1>

<form on:submit|preventDefault={add}>
  <input bind:value={text} placeholder="What needs to be done?" />
  <button type="submit">Add</button>
</form>

{#if items.length === 0}
  <p>Nothing here.</p>
{:else}
  <ul>
    {#each items as it (it.id)}
      <li>
        <label>
          <input type="checkbox" checked={it.done} on:change={() => toggle(it.id)} />
          {it.title}
        </label>
        <button on:click={() => remove(it.id)}>x</button>
      </li>
    {/each}
  </ul>
{/if}

<p>Done: {doneCount} · Left: {leftCount}</p>

<style>
  form { margin: 1rem 0; display: flex; gap: .5rem; }
  ul { padding-left: 1rem; }
  li { display: flex; align-items: center; gap: .5rem; }
  button { cursor: pointer; }
</style>
```

- **핵심 요약**
  - 입력은 `bind:value`로 상태와 연결.
  - 리스트는 `each (key)`로 렌더링.
  - 상태 변경은 **새 참조로 재할당**하여 반응성 보장.
  - 파생값(`doneCount`, `leftCount`)은 `$:` 반응 선언.

---

## 2.5 흔한 함정과 모범 사례

1) **배열/객체 직접 변경** → UI 미갱신  
   - 해결: **새 객체/배열로 재할당**(또는 v5 `$state` 심화 기능 사용).
2) **DOM에 직접 접근 남용**  
   - 가급적 **상태 → 템플릿** 바인딩으로 해결. 꼭 필요할 때 `bind:this`/액션 사용.
3) **중복 키 없는 `each`**  
   - 중간 삽입/삭제 시 DOM 재사용 문제가 발생할 수 있다. **고유 키를 제공**.
4) **반응식에서 부작용과 계산 혼합**  
   - 유지보수를 위해 **계산**과 **부작용**을 **분리**.
5) **`{@html}` 남용**  
   - XSS 위험. 신뢰된 콘텐츠에만 제한적으로 사용.

---

## 2.6 보너스: `{#key}` 블록과 반응적 재마운트

특정 값이 바뀔 때 **컴포넌트를 통째로 재마운트**하고 싶다면 `{#key}`를 쓴다.
```svelte
<script>
  let seed = 0;
  function reshuffle() { seed = Math.random(); }
</script>

{#key seed}
  <!-- seed가 바뀔 때 이 블록의 하위가 새로 마운트 -->
  <Randomized />
{/key}

<button on:click={reshuffle}>Reshuffle</button>
```
- 애니메이션 초기화, 3rd-party 위젯 재설정 등에서 유용하다.

---

## 2.7 정리 체크리스트

- [ ] 컴포넌트는 `.svelte` 하나에 **script/markup/style**이 공존(스코프 CSS)  
- [ ] 반응성은 **재할당이 신호** — 파생값/부작용은 `$:`  
- [ ] 템플릿 `{}`엔 **표현식**만, HTML 삽입은 `{@html}`(주의)  
- [ ] 조건 `{#if}`, 반복 `{#each (key)}`, 비동기 `{#await}`  
- [ ] 이벤트 `on:click` + **수정자**(`|preventDefault`, `|once` …)  
- [ ] 양방향 `bind:` — `value/checked/group/this/currentTime/files/open` …  
- [ ] 배열/객체 변경은 **새 참조로 재할당**(v4), 또는 v5의 deep reactivity  
- [ ] 필요 시 `{#key}`로 **재마운트** 패턴 활용
