---
layout: post
title: Svelte - SvelteKit 기본 (2)
date: 2025-09-28 23:30:23 +0900
category: Svelte
---
# SvelteKit 기본 (2)

**`load` 함수(클라이언트/서버), 실행 순서와 캐싱/무효화 · 폼 액션(`actions`)과 점진적 향상(Progressive Enhancement)**

> 이 장은 **데이터 로딩의 생애주기**와 **폼 제출 흐름**을 중심으로, SvelteKit 앱을 실무적으로 튜닝하는 방법을 예제와 함께 정리한다.
> - `+layout/+page`의 **server/universal `load`**가 *언제/어디서* 실행되는지,
> - **의존성(deps)**과 **무효화(invalidate)**, **캐싱 헤더**로 새로고침·탭 이동·뒤/앞으로가기를 올바르게 다루는 법,
> - **폼 액션**으로 **서버 우선(SSR)** 제출을 구현하고, `enhance`로 **점진적 향상**까지 덧입히는 법을 다룬다.

---

## `load` 함수 총정리 — “무엇을 어디에서, 어떤 순서로?”

SvelteKit의 `load`는 “**라우트 전환마다 데이터 계약을 수행**하는 함수”다. 파일 위치와 확장자에 따라 **실행 위치**가 달라진다.

### 종류와 실행 위치

- **서버 전용(Server load)**
  - `+layout.server.ts` / `+page.server.ts`
  - **항상 서버에서만** 실행. 쿠키/세션/DB/비밀키 사용 가능.
  - 클라이언트 사이드 전환 시 **다시 서버 호출**(필요 시).
- **유니버설(Universal load)**
  - `+layout.ts` / `+page.ts`
  - 최초 SSR 때 **서버에서 1회 실행**해 결과를 HTML에 주입 →
    이후 **클라이언트 내비게이션**에서는 **브라우저에서 실행**.

> **결과 병합**: 같은 레벨에서 `server` → `universal` 순으로 실행되고 **반환 객체가 병합**되어 하위로 **캐스케이딩**된다.

### 실행 순서(중첩 레이아웃 포함)

루트에서 하위로 내려가며 **서버 → 유니버설** 순서로 실행:

1. `src/routes/+layout.server.ts`
2. `src/routes/+layout.ts`
3. `src/routes/(세그먼트)/+layout.server.ts`
4. `src/routes/(세그먼트)/+layout.ts`
5. `src/routes/(세그먼트)/+page.server.ts`
6. `src/routes/(세그먼트)/+page.ts`

> 각 `load`의 결과는 **누적 병합**되어 최종 `data`로 페이지 컴포넌트에 전달된다.

### `load` 시그니처와 매개변수

```ts
// +page.ts (universal)
import type { PageLoad } from './$types';

export const load: PageLoad = async ({ fetch, params, url, parent, depends, data, setHeaders }) => {
  // fetch: 래핑된 fetch (SSR↔CSR 모두 쿠키/자격 전달 일관)
  // params: [slug] 등 경로 파라미터
  // url: 현재 URL (searchParams 등)
  // parent(): 상위 레이아웃 load 결과를 await로 가져옴
  // depends('키' | URL): 무효화 키/리소스 등록
  // data: 같은 레벨의 .server.ts load 반환이 이미 병합되어 들어옴
  // setHeaders(): SSR 응답 헤더 설정(클라이언트에서 호출 시 no-op)
};
```

> **팁**: 유니버설 `load`에서 `setHeaders`를 호출해도 **SSR 렌더링 시에만 적용**된다(클라이언트 전환에서는 무시됨).

---

## `fetch`와 캐싱 — 서버·클라이언트 모두에서 “같이” 동작

SvelteKit이 `load`에 주입하는 `fetch`는 **다음 특성**을 가진다.

1) **쿠키/자격 자동 전파**
   - 서버에서 내부 API(`/api/*`) 호출 시 **요청자 쿠키/세션**이 전파된다.
   - “SSR 때는 서버-서버 호출, CSR 때는 브라우저 fetch”가 **동일 시그니처**로 동작.

2) **요청 디듀플리케이션**
   - 동일 라우팅 사이클에서 **같은 URL**을 `fetch`하면 **한 번만** 요청(중복 제거).

3) **의존성 추적**
   - `depends(URL)` 또는 `fetch(URL)` 자체로 **해당 URL**이 **무효화 대상**에 등록된다.

4) **브라우저/중간 캐시와 결합**
   - API 응답에서 설정한 **`cache-control` 헤더**를 그대로 존중.
   - SSR의 `setHeaders()`로 페이지 응답 자체의 캐시를 제어할 수도 있다.

### 예: 서버 캐시 헤더 + API 캐시 헤더

```ts
// +page.server.ts
import type { PageServerLoad } from './$types';

export const load: PageServerLoad = async ({ fetch, setHeaders }) => {
  // 페이지(문서) 자체 캐시: 30초
  setHeaders({ 'cache-control': 'public, max-age=30' });

  const res = await fetch('/api/products'); // 내부 API
  // /api/products 가 자체적으로 max-age=60 등을 보낼 수 있음
  const items = await res.json();

  return { items };
};
```

### 외부 API와 캐시 무력화

```ts
// +page.ts
export const load = async ({ fetch, depends }) => {
  // 의존성 키(임의 문자열). 나중에 클라이언트에서 invalidate('app:rates')로 무효화 가능
  depends('app:rates');

  // 외부 API의 캐시를 무시하고 항상 최신
  const r = await fetch('https://api.example.com/rates', { cache: 'no-store' });
  return { rates: await r.json() };
};
```

---

## `depends` / `invalidate` / `invalidateAll` — “언제 다시 불러올까?”

SvelteKit의 무효화 시스템은 “**어떤 `load`가 어떤 리소스에 의존하는지**”를 기억했다가,
**클라이언트에서** 해당 리소스가 바뀌었다고 신호하면 필요한 `load`만 다시 실행한다.

### 의존성 등록

- **자동 등록**: `fetch('/api/x')`로 가져오면 **그 URL**이 자동 등록
- **수동 등록**: `depends('app:key')`로 **추상 키** 등록 (API 아닌 로컬 변화도 트리거 가능)

### 클라이언트에서 무효화

```svelte
<script lang="ts">
  import { invalidate, invalidateAll } from '$app/navigation';

  async function refreshProducts() {
    await invalidate('/api/products'); // 해당 URL을 의존한 load만 재실행
  }

  async function hardRefresh() {
    await invalidateAll(); // 현재 페이지의 모든 load 전부 재실행
  }
</script>

<button on:click={refreshProducts}>재로드(선택적)</button>
<button on:click={hardRefresh}>강제 재로드</button>
```

### 예: 폼 제출 후 정확히 필요한 것만 갱신

폼 액션에서 서버가 데이터를 바꿨다면, **성공 후** 클라이언트에서 `invalidate('app:key')` 혹은 해당 API URL을 **선택적**으로 무효화한다.

---

## `parent()`와 데이터 병합 — 상하위 계약

유니버설/서버 `load`에서 `parent()`를 `await`하면 **상위 레이아웃의 반환값**을 가져올 수 있다.

```ts
// src/routes/dashboard/+layout.ts
export const load = async ({ fetch }) => {
  const profile = await fetch('/api/me').then(r => r.json());
  return { profile };
};

// src/routes/dashboard/reports/+page.ts
export const load = async ({ parent, fetch }) => {
  const { profile } = await parent(); // 상위 레이아웃 값
  const reports = await fetch(`/api/reports?u=${profile.id}`).then(r => r.json());
  return { reports };
};
```

> **주의**: 상위/하위가 **같은 키 이름**을 반환하면 하위가 **덮어쓴다**.
> 키 네이밍 충돌을 피하는 습관이 좋다(예: `layoutUser`, `pageData`).

---

## 스트리밍/지연 데이터(간단 패턴)

가벼운 방법: `load`에서 **Promise 그대로 반환**하고 컴포넌트에서 `{#await}`로 처리한다.
(대용량/세밀 스트리밍은 프레임워크 버전에 따라 전용 유틸이 있으나, 여기선 **범용 패턴**만 소개한다.)

```ts
// +page.ts
export const load = async ({ fetch }) => {
  // 무거운 목록은 Promise로 그대로
  const list = fetch('/api/heavy').then(r => r.json());
  return { list };
};
```

```svelte
<!-- +page.svelte -->
<script lang="ts">
  export let data: { list: Promise<any[]> };
</script>

{#await data.list}
  <p>Loading…</p>
{:then items}
  <ul>{#each items as it}<li>{it.name}</li>{/each}</ul>
{:catch e}
  <p style="color:#b00020">Error: {e.message}</p>
{/await}
```

---

## 에러/리다이렉트 — `error`, `redirect`, `fail`

- **`error(status, message)`**: 해당 라우트에서 에러 페이지로 분기
- **`redirect(status, location)`**: 즉시 리다이렉트 (307/308/303 등)
- **`fail(status, data)`**: **폼 액션**에서 유효성 실패 시 페이지로 **데이터와 함께** 돌아감

```ts
// +page.server.ts
import { error, redirect } from '@sveltejs/kit';

export const load = async ({ locals }) => {
  if (!locals.user) throw redirect(303, '/login');
  // ...
};

export const actions = {
  safe: async ({ request }) => {
    const body = await request.formData();
    if (!body.get('email')) {
      // 400 상태 + 폼에 다시 채워넣을 데이터
      return fail(400, { message: 'email is required', values: Object.fromEntries(body) });
    }
    // 저장 실패
    // throw error(500, 'DB down');
    return { success: true };
  }
};
```

---

# 7.B 폼 액션(actions)과 점진적 향상(PE)

## 7.B.1 “서버 우선” 폼

SvelteKit은 **서버 우선** 폼 제출을 기본 제공한다.
`+page.server.ts`에 `export const actions = { ... }`를 정의하고, 페이지에 **표준 `<form method="POST">`**를 작성하면 된다.

### 7.B.1.1 가장 단순한 폼 액션

```ts
// +page.server.ts
import type { Actions } from './$types';

export const actions: Actions = {
  default: async ({ request, cookies }) => {
    const form = await request.formData();
    const name = String(form.get('name') ?? '');

    // 처리(검증/저장 등)
    cookies.set('name', name, { path: '/', httpOnly: true });

    return { ok: true, name }; // 페이지로 되돌아가 data.form 아래에 머지됨
  }
};
```

```svelte
<!-- +page.svelte -->
<script lang="ts">
  export let form; // 액션의 반환값이 들어오는 자리 (SvelteKit 예약)
</script>

<form method="POST">
  <input name="name" placeholder="Your name" />
  <button type="submit">Save</button>
</form>

{#if form?.ok}
  <p>Saved: {form.name}</p>
{/if}
```

> **핵심**: JS가 꺼져 있어도 **서버로 POST → 페이지 리렌더**가 되므로 **접근성/내결함성**이 높다.

### 7.B.1.2 유효성 실패: `fail(status, data)`

```ts
// +page.server.ts
import { fail } from '@sveltejs/kit';
export const actions = {
  default: async ({ request }) => {
    const fd = await request.formData();
    const email = String(fd.get('email') ?? '');
    if (!email.includes('@')) {
      // HTTP 400 + 폼 데이터
      return fail(400, { message: 'Email is invalid', values: Object.fromEntries(fd) });
    }
    return { ok: true };
  }
};
```

```svelte
<script lang="ts"> export let form; </script>

<form method="POST">
  <input name="email" value={form?.values?.email ?? ''} aria-invalid={!!form?.message} />
  {#if form?.message}<p style="color:#b00020">{form.message}</p>{/if}
  <button>Submit</button>
</form>
```

> **포인트**: 실패시에도 입력값을 잃지 않게 `values`를 돌려주면 UX가 좋아진다.

### 7.B.1.3 여러 액션 핸들러 사용

```ts
// +page.server.ts
export const actions = {
  create: async ({ request }) => { /* ... */ return { created: true }; },
  remove: async ({ request }) => { /* ... */ return { removed: true }; }
} satisfies Actions;
```

```svelte
<form method="POST">
  <!-- 어떤 액션으로 보낼지 버튼 name 지정 -->
  <button name="create" value="1">Create</button>
  <button name="remove" value="1">Remove</button>
</form>
```

---

## 7.B.2 점진적 향상: `$app/forms`의 `enhance`

`enhance`를 쓰면 **페이지 이동 없이 AJAX로 제출**하고, 반환 결과를 **현재 페이지 상태에 적용**한다.

```svelte
<script lang="ts">
  import { enhance } from '$app/forms';
  export let form; // 서버 반환이 여전히 여기로 들어옴
</script>

<form method="POST" use:enhance>
  <input name="task" required />
  <button>Add</button>
</form>

{#if form?.ok}<p>Added!</p>{/if}
```

> JS가 없는 환경에선 그냥 **기본 POST**가 일어나고, JS가 있으면 AJAX로 변환된다. → **프로그레시브**.

### 7.B.2.1 `enhance` 커스터마이즈(로딩/에러/무효화)

```svelte
<script lang="ts">
  import { enhance } from '$app/forms';
  import { invalidate } from '$app/navigation';

  let pending = false;

  const enhanceOpts = ({ form }) => {
    return {
      // 제출 전 (옵션으로 fetch를 바꿔치기 가능)
      async pending({ data, cancel }) {
        pending = true;
        // 예: optimistic UI. 필요 시 cancel()로 취소 가능
      },
      // 서버 응답 성공
      async result({ result, update }) {
        pending = false;

        if (result.type === 'success') {
          // form 스토어 갱신 (update()는 서버가 돌려준 form 데이터로 page를 갱신)
          await update();
          // 선택적: 특정 데이터 무효화
          await invalidate('/api/tasks');
        }
        else if (result.type === 'failure') {
          // 실패 시에도 update()로 form 값/오류를 반영
          await update();
        }
        else if (result.type === 'redirect') {
          // 리다이렉트는 자동 처리됨(여기선 주로 로깅 정도)
        }
        else if (result.type === 'error') {
          console.error(result.error);
        }
      }
    };
  };
</script>

<form method="POST" use:enhance={enhanceOpts}>
  <input name="task" required />
  <button disabled={pending}>{pending ? 'Saving…' : 'Add'}</button>
</form>
```

### 7.B.2.2 `applyAction` — 폼 없이도 액션 결과를 적용

SPA 상호작용 중 **직접 FormData를 만들어 액션으로 보내고**, 반환값을 현재 페이지에 **적용**한다.

```svelte
<script lang="ts">
  import { applyAction } from '$app/forms';

  async function addQuick(task: string) {
    const fd = new FormData();
    fd.set('/?/create', '1');  // "create" 액션을 직접 지정하는 관례
    fd.set('task', task);

    const res = await fetch('?/create', { method:'POST', body: fd });
    const result = await res.json(); // 프레임워크가 감싸서 돌려줌
    await applyAction(result);       // 폼과 동일하게 form 스토어/페이지 업데이트
  }
</script>
```

> **언제 유용한가**: 버튼/메뉴 클릭 같은 **비-폼 UI**에서 서버 액션을 재사용하고 싶을 때.

---

## 7.B.3 업로드/멀티파트/파일

`request.formData()`는 파일도 함께 받는다. **액션에서 파일을 받고 저장**한 뒤, 성공 시 **무효화** 또는 **리다이렉트**한다.

```ts
// +page.server.ts
import { fail, redirect } from '@sveltejs/kit';

export const actions = {
  upload: async ({ request }) => {
    const fd = await request.formData();
    const file = fd.get('file');
    if (!(file instanceof File) || file.size === 0) {
      return fail(400, { message: '파일이 비었습니다' });
    }
    // 파일 저장 로직…
    // 저장 뒤 목록 페이지로
    throw redirect(303, '/files');
  }
};
```

```svelte
<form method="POST" enctype="multipart/form-data">
  <input type="file" name="file" />
  <button name="upload">Upload</button>
</form>
```

---

## 7.B.4 폼 액션 + 무효화(새로고침 없이 목록 갱신)

### 7.B.4.1 서버: 액션 성공 후 **키 통지(선택)**

액션 자체에는 “키 통지”가 없지만, 클라이언트에서 `enhance`의 `result`에서 **무효화**를 호출하면 된다.
(또는 리다이렉트 전략으로 간단화)

```svelte
<script lang="ts">
  import { enhance } from '$app/forms';
  import { invalidate } from '$app/navigation';

  const opts = () => ({
    async result({ result, update }) {
      await update();
      if (result.type === 'success') {
        await invalidate('/api/items'); // 해당 키 의존한 load만 재호출
      }
    }
  });
</script>

<form method="POST" use:enhance={opts}>
  <input name="title" required />
  <button>Add</button>
</form>
```

---

## 7.B.5 접근성/UX 모범사례

1) **서버 우선**: JS가 꺼져도 동작하도록 `<form method="POST">`를 기본으로.
2) **에러 반환은 `fail`**: 상태코드와 함께 **필드 값**을 돌려주어 재입력 방지.
3) **포커스 관리**: 실패 시 **첫 오류 필드**로 포커스 이동.
4) **낙관적 UI**는 **취소/롤백** 경로를 함께 설계.
5) **파일 업로드/장시간 작업**은 **진행률 표시** 또는 **완료 알림** 제공.
6) **무효화 범위 최소화**: `invalidate('/api/…')`로 **정확한 리소스만** 새로고침.

---

# 7.C 실전: “할 일 목록” — load + actions + 무효화 + 점진적 향상

### 7.C.1 API 라우트(데모용 간이 저장)

```ts
// src/routes/api/todos/+server.ts
import type { RequestHandler } from './$types';

let todos = [{ id: 'a', title: 'Read SvelteKit', done: false }];

export const GET: RequestHandler = async () => {
  return new Response(JSON.stringify(todos), { headers: { 'content-type': 'application/json', 'cache-control':'no-store' } });
};

export const POST: RequestHandler = async ({ request }) => {
  const fd = await request.formData();
  const title = String(fd.get('title') ?? '').trim();
  if (!title) return new Response('Bad Request', { status: 400 });
  todos = [...todos, { id: crypto.randomUUID(), title, done: false }];
  return new Response('ok', { status: 201 });
};

export const PATCH: RequestHandler = async ({ request }) => {
  const fd = await request.formData();
  const id = String(fd.get('id'));
  todos = todos.map(t => t.id === id ? { ...t, done: !t.done } : t);
  return new Response('ok');
};

export const DELETE: RequestHandler = async ({ request }) => {
  const fd = await request.formData();
  const id = String(fd.get('id'));
  todos = todos.filter(t => t.id !== id);
  return new Response('ok');
};
```

### + 액션 정의

```ts
// src/routes/todos/+page.ts (universal)
import type { PageLoad } from './$types';

export const load: PageLoad = async ({ fetch, depends }) => {
  depends('/api/todos'); // 이 URL 의존 → invalidate('/api/todos')로 갱신 가능
  const items = await fetch('/api/todos').then(r => r.json());
  return { items };
};
```

```ts
// src/routes/todos/+page.server.ts (actions)
import type { Actions } from './$types';
import { fail } from '@sveltejs/kit';

export const actions: Actions = {
  add: async ({ request, fetch }) => {
    const fd = await request.formData();
    const title = String(fd.get('title') ?? '').trim();
    if (!title) return fail(400, { message:'필수 입력', values:Object.fromEntries(fd) });
    await fetch('/api/todos', { method: 'POST', body: fd });
    return { added: true };
  },
  toggle: async ({ request, fetch }) => {
    await fetch('/api/todos', { method: 'PATCH', body: await request.formData() });
    return { toggled: true };
  },
  remove: async ({ request, fetch }) => {
    await fetch('/api/todos', { method: 'DELETE', body: await request.formData() });
    return { removed: true };
  }
};
```

### 7.C.3 페이지 컴포넌트: `enhance` + 무효화

```svelte
<!-- src/routes/todos/+page.svelte -->
<script lang="ts">
  import { enhance } from '$app/forms';
  import { invalidate } from '$app/navigation';
  export let data: { items: { id:string; title:string; done:boolean }[] };
  export let form;

  // 모든 액션 성공 후 /api/todos만 정밀 무효화
  const opts = () => ({
    async result({ result, update }) {
      await update();
      if (result.type === 'success') await invalidate('/api/todos');
    }
  });
</script>

<h1>Todos</h1>

<form method="POST" use:enhance={opts} class="add">
  <input name="title" value={form?.values?.title ?? ''} placeholder="할 일" />
  {#if form?.message}<small style="color:#b00020">{form.message}</small>{/if}
  <button name="add">추가</button>
</form>

<ul>
  {#each data.items as t (t.id)}
    <li class:done={t.done}>
      <form method="POST" use:enhance={opts}>
        <input type="hidden" name="id" value={t.id} />
        <button name="toggle">{t.done ? '↩︎' : '✓'}</button>
      </form>

      <span>{t.title}</span>

      <form method="POST" use:enhance={opts}>
        <input type="hidden" name="id" value={t.id} />
        <button name="remove">삭제</button>
      </form>
    </li>
  {/each}
</ul>

<style>
  .add { display:flex; gap:.5rem; align-items:center; margin:.5rem 0 1rem; }
  ul { padding:0; list-style:none; display:grid; gap:.4rem; }
  li { display:flex; gap:.5rem; align-items:center; }
  li.done span { text-decoration: line-through; color:#6b7280; }
  button { border:1px solid #e5e7eb; border-radius:10px; padding:.3rem .6rem; background:#f8fafc; }
</style>
```

> - **서버 우선** 폼이 기본이고, JS가 있을 때 `enhance`가 AJAX로 바꿔준다.
> - 성공 시 **해당 API만 무효화**하여 **빠르고 정확하게 갱신**한다.
> - 실패 시 `fail` 데이터가 `form`으로 반영되어 값 보존/에러 표시에 사용된다.

---

## 7.D 성능/안정성 체크리스트

- [ ] **서버 vs 유니버설** 경계: 비밀키/DB/쿠키는 `.server.ts`, 브라우저 API는 유니버설/컴포넌트
- [ ] `depends`로 **의존성 태깅** → `invalidate('키/URL')`로 **정밀 재로딩**
- [ ] `setHeaders`로 **문서 캐시** 또는 API의 **`cache-control`**로 리소스 캐시
- [ ] **중복 fetch**는 자동 디듀플리케이션: 동일 URL 요청을 합치자
- [ ] **URL을 진실원천**으로: 검색/페이지 등은 QueryString에 싣고 `load`에서 읽기
- [ ] 폼은 **서버 우선** + `enhance`로 **점진적 향상**
- [ ] 유효성 실패는 **`fail(status, data)`**로 값 보존/명확한 오류 전달
- [ ] 액션 성공 후 **정밀 무효화**(`invalidate('/api/...')`)로 불필요한 재로딩 방지
- [ ] 스트리밍/지연 데이터는 필요 시 **Promise 반환 + `{#await}`**
- [ ] 뒤/앞으로가기 UX: SvelteKit가 **히스토리 스냅샷**을 보존하지만, 중요한 데이터는 **무효화 정책**을 함께 설계

---

## 7.E 요약

- `load`는 **레이아웃→페이지**, **서버→유니버설** 순으로 실행되어 결과를 병합한다.
- 래핑된 `fetch`는 **SSR/CSR 모두 동일**하고, **의존성/중복제거/쿠키 전파**를 처리한다.
- **의존성 등록(`depends`) + 무효화(`invalidate`)**로 “언제 다시 불러올지”를 제어한다.
- 폼은 **액션**으로 서버 우선 제출 → `enhance`로 **JS 있을 땐 AJAX**.
- 실패는 `fail`, 성공 후엔 **정밀 무효화** 또는 **리다이렉트**로 UX를 깔끔하게.
