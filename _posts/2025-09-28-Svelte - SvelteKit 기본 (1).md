---
layout: post
title: Svelte - SvelteKit 기본 (1)
date: 2025-09-28 23:25:23 +0900
category: Svelte
---
# SvelteKit 기본 (1)

> 이 장은 SvelteKit의 **파일 기반 라우팅과 렌더링 모델**을 이해하는 데 필요한 핵심을, **작동 예제**와 함께 한 번에 훑는다.
> - “**어떤 파일을 어디에 두면 무슨 역할을 하나?**”를 먼저 잡고
> - **동적 파라미터/중첩 레이아웃**의 흐름을 익힌 뒤
> - **SSR/SPA/SSG**와 **프리렌더링**을 조합해 배포 전략을 세운다.

---

## 폴더 & 파일 구조 — 어떤 파일이 무엇을 하나?

SvelteKit는 `src/routes` 폴더로 **라우트(경로)**를 구성한다. 폴더/파일 이름이 **URL**이 된다.

```
my-app/
  src/
    routes/
      +layout.svelte        # 모든 하위 경로에 공통 레이아웃
      +layout.ts            # (선택) 공통 load 옵션/프리렌더/SSR 토글 등
      +layout.server.ts     # (선택) 서버에서만 실행되는 레이아웃 load

      +page.svelte          # 루트(/) 페이지
      +page.ts              # (선택) 페이지용 universal load
      +page.server.ts       # (선택) 페이지용 server load

      api/
        +server.ts          # /api (HTTP endpoint 라우트)
      about/
        +page.svelte        # /about
      blog/
        +layout.svelte      # /blog/* 구간 공통 레이아웃
        +page.svelte        # /blog (목록)
        [slug]/
          +page.svelte      # /blog/hello 같은 동적 페이지
          +page.ts          # slug 기반 데이터 로딩 (universal)
          +page.server.ts   # slug 기반 서버 전용 로딩/보호
```

### 각 파일의 책임 요약

- **`+layout.svelte`**: 공통 UI(헤더/푸터/토스트 등), **데이터는 상위에서 하위로 캐스케이딩**
- **`+layout.ts`**: 클라이언트/서버 **모두에서 실행되는(load) 함수**와 **페이지 옵션**(예: `export const ssr = false`)
- **`+layout.server.ts`**: **서버에서만** 실행되는 load (쿠키/세션/DB 조회 등)

- **`+page.svelte`**: 해당 경로의 **페이지 컴포넌트**
- **`+page.ts`**: 페이지용 **universal load** (CSR 전환에도 실행)
- **`+page.server.ts`**: 페이지용 **server load** (SSR만, CSR 전환 시에는 호출되지 않음)

- **`+server.ts`**: **API 라우트**. `GET/POST/PUT/DELETE` 핸들러를 export하여 **HTTP 엔드포인트**를 만든다.

> 기억법
> - `.server.ts`가 붙으면 “**서버 전용**”
> - `.ts`만 있으면 “**유니버설(universal)** = 서버SSR + 클라이언트 라우팅 전환 모두”
> - `.svelte`는 “**뷰**”

---

## 레이아웃 & 페이지의 데이터 흐름 — **캐스케이딩**

상위 레이아웃의 `load`가 리턴한 값은 **하위 레이아웃/페이지**에 **자동으로 병합**된다. 이를 **캐스케이딩**이라 한다.

### 레이아웃에서 사용자 세션 주입하기

```ts
// src/routes/+layout.server.ts
import type { LayoutServerLoad } from './$types';

export const load: LayoutServerLoad = async ({ cookies }) => {
  const session = cookies.get('session');
  // 실제로는 DB/세션 검증 로직…
  const user = session ? { name: 'Ada', role: 'admin' } : null;

  return { user }; // 모든 하위 라우트에서 사용 가능
};
```

```svelte
<!-- src/routes/+layout.svelte -->
<script lang="ts">
  export let data: { user: { name: string; role: string } | null };
</script>

<header class="topbar">
  <a href="/">MyApp</a>
  {#if data.user}
    <span>Hi, {data.user.name}</span>
  {:else}
    <a href="/login">Login</a>
  {/if}
</header>

<slot /> <!-- 하위 레이아웃/페이지 -->
```

- 하위의 `+page.svelte`에서는 **`export let data`**로 같이 받는다(병합된 값).

### 페이지 전용 데이터

```ts
// src/routes/blog/[slug]/+page.ts
import type { PageLoad } from './$types';

export const load: PageLoad = async ({ params, fetch }) => {
  const post = await fetch(`/api/posts/${params.slug}`).then(r => r.json());
  return { post };
};
```

```svelte
<!-- src/routes/blog/[slug]/+page.svelte -->
<script lang="ts">
  export let data: { post: { title: string; html: string } };
</script>

<article class="prose">
  <h1>{data.post.title}</h1>
  {@html data.post.html}
</article>
```

- `+layout.server.ts`에서 받은 **전역 데이터(user)**와, `+page.ts`의 **지역 데이터(post)**가 **병합**되어 `data`로 들어온다.

---

## 라우팅 — 파일과 폴더가 URL이 된다

### 기본 규칙

- 폴더: 경로 세그먼트(`/blog` → `routes/blog/`)
- `+page.svelte`: **그 경로의 페이지 뷰**
- `+server.ts`: **그 경로의 API 엔드포인트**(`GET/POST` 등)
- **인덱스**: 폴더에 `+page.svelte`가 있으면 해당 세그먼트 루트(예: `routes/blog/+page.svelte` → `/blog`)

### 동적 파라미터

- **`[slug]`**: `/blog/[slug]/+page.svelte` → `/blog/hello`에서 `params.slug === 'hello'`
- **캣치올(`[...]`)**: `/docs/[...path]/+page.svelte` → `/docs/guide/getting-started`에서 `params.path === 'guide/getting-started'`
- **옵셔널(`[[lang]]`)**: `/[[lang]]/about/+page.svelte` → `/about` 또는 `/en/about`

```ts
// src/routes/docs/[...path]/+page.ts
import type { PageLoad } from './$types';

export const load: PageLoad = async ({ params }) => {
  // params.path가 'guide/getting-started' 같은 문자열
  return { path: params.path.split('/') };
};
```

```svelte
<!-- src/routes/docs/[...path]/+page.svelte -->
<script lang="ts">
  export let data: { path: string[] };
</script>

<h1>Docs path</h1>
<ol>
  {#each data.path as seg}
    <li>{seg}</li>
  {/each}
</ol>
```

### 중첩 레이아웃

폴더마다 `+layout.svelte`를 두면, **중첩된 레이아웃**이 된다.

```
routes/
  +layout.svelte        # App Shell
  blog/
    +layout.svelte      # Blog Shell
    +page.svelte        # /blog (목록)
    [slug]/
      +page.svelte      # /blog/:slug (상세)
```

상위 레이아웃 → 하위 레이아웃 → 페이지 순으로 **겹겹이 감싸진다**.

```svelte
<!-- src/routes/blog/+layout.svelte -->
<script lang="ts"> export let data; </script>
<div class="blog-shell">
  <h2>Blog</h2>
  <slot />
</div>
```

---

## API 라우트 — `+server.ts`로 HTTP 엔드포인트 만들기

SvelteKit은 라우트 파일에 `+server.ts`를 두면 **백엔드 핸들러**가 된다.

```ts
// src/routes/api/posts/+server.ts
import type { RequestHandler } from './$types';

export const GET: RequestHandler = async () => {
  const posts = [{ slug:'hello', title:'Hello SvelteKit' }];
  return new Response(JSON.stringify(posts), {
    headers: { 'content-type': 'application/json' }
  });
};

export const POST: RequestHandler = async ({ request }) => {
  const body = await request.json();
  // 저장 로직…
  return new Response('ok', { status: 201 });
};
```

- 클라이언트에서 `fetch('/api/posts')`로 호출하면 된다.
- **CORS/세션/쿠키** 등 서버 관심사는 이 파일에서 처리.

---

## `load` 함수 — **언제 어디서 실행될까?**

### 세 가지 `load`

- **`+layout.ts / +page.ts`의 `load`**: **유니버설**.
  - SSR(첫 요청)에서는 **서버**에서 실행되어 HTML에 주입
  - 클라이언트 라우팅 전환 시에는 **브라우저**에서 실행
- **`+layout.server.ts / +page.server.ts`의 `load`**: **서버 전용**.
  - 쿠키/세션/DB 접근, **비밀키** 사용 가능
  - CSR 전환 시에는 다시 호출되지 않음(서버 요청 없으면)
- **둘 다 있으면**: `server` → `universal` 순서로 실행되고 결과가 병합된다.

### 타입과 반환

```ts
// src/routes/products/+page.ts
import type { PageLoad } from './$types';

export const load: PageLoad = async ({ fetch, url, depends }) => {
  depends('app:products'); // 이 키에 종속 → 무효화 시 재호출
  const page = Number(url.searchParams.get('page') ?? '1');
  const items = await fetch(`/api/products?page=${page}`).then(r => r.json());
  return { items, page };
};
```

```svelte
<!-- src/routes/products/+page.svelte -->
<script lang="ts">
  export let data: { items: any[]; page: number };
</script>

<h1>Products – page {data.page}</h1>
<ul>{#each data.items as it}<li>{it.name}</li>{/each}</ul>
```

- `depends()`는 **캐시 무효화 키**. 후속 장(폼 액션/invalidations)에서 더 다룸.

---

## · SSG

SvelteKit은 **페이지/레이아웃 단위**로 렌더링 모드를 설정할 수 있다.

### SSR (Server-Side Rendering)

- 기본값. 첫 요청에서 서버가 HTML을 만들어 응답.
- SEO/초기 응답/보안(비밀키) 측면에서 유리.
- CSR로 전환하면 이후 페이지 이동은 클라이언트에서.

```ts
// src/routes/+layout.ts
export const ssr = true;    // 기본값이라 보통 생략
export const csr = true;    // CSR 전환 허용(기본값 true)
```

### SPA/CSR Only (서버 렌더링 비활성화)

- 문서가 **클라이언트에서만** 렌더됨(SEO 취약, CDN 캐싱 쉬움).
- 빠른 대시보드/내부 도구에서 사용.

```ts
// src/routes/app/+layout.ts
export const ssr = false;   // 이 구간 이하 SSR 끄기
```

### / 프리렌더링

- 빌드 시 HTML을 **파일로 생성**해 CDN에서 서빙.
- **변하지 않거나, revalidate로 느슨하게 변하는** 페이지에 적합.

```ts
// src/routes/+layout.ts (또는 특정 페이지의 +page.ts)
export const prerender = true;
```

- `prerender = true`인 경로는 **링크로 도달 가능한 모든 하위 페이지를 크롤**해 정적 생성한다.
- **동적 파라미터**가 많거나 무한이면, **entries** 옵션이나 별도의 **endpoint**로 **프리렌더 목록**을 제공해야 한다(후술).

---

## 프리렌더링 세부 — 엔트리/하이브리드

### 특정 경로만 프리렌더

```ts
// svelte.config.js (예시 개념 설명용)
const config = {
  kit: {
    prerender: {
      entries: [
        '/',           // 루트
        '/about',
        '/blog/hello'  // 동적 중 일부
      ]
    }
  }
};
export default config;
```

- `entries: ['*']`는 기본(가능한 모든 페이지). `+layout.ts`에서 `prerender = true`면 해당 구간을 크롤한다.

### 동적 리스트를 SSG하고 상세는 SSR (하이브리드)

- 블로그: `/blog`와 일부 인기 글은 SSG, 나머지는 SSR로 실시간 제공.
- 구현: 상위 `+layout.ts` 에서 **`prerender = true`**를 두되, **특정 하위 페이지**에서 **`export const prerender = false`**로 예외 처리.

```ts
// src/routes/blog/+layout.ts
export const prerender = true;
```

```ts
// src/routes/blog/[slug]/+page.ts
export const prerender = false; // 상세는 SSR
```

- 또는 반대로, 전체는 SSR이고 특정 페이지만 `prerender = true`로 개별 지정 가능.

### Revalidation (SSG + 유효기간)

정적 페이지라도 **주기적으로 갱신**하고 싶을 수 있다.

```ts
// src/routes/news/+page.ts
export const prerender = true;
export const csr = true; // 클라이언트 상호작용 허용
export const trailingSlash = 'always'; // 선택 옵션 예시

export const load = async ({ fetch, setHeaders }) => {
  const articles = await fetch('/api/news').then(r => r.json());
  // 캐시 헤더를 통해 CDN 관리 (개념 예시)
  setHeaders({ 'cache-control': 'max-age=60' }); // 60초 캐시
  return { articles };
};
```

- “ISR(Incremental Static Regeneration)” 같은 개념은 **빌드-후 갱신** 전략과 **캐싱**을 조합해 만들어낸다. SvelteKit은 **revalidate** 전용 키가 아니라, **무효화/캐시 헤더/프리렌더 재실행** 등으로 실현한다(팀 빌드/배포 파이프라인에서 결정).

---

## 예제로 보는 모드 조합

### + 앱(SSR/CSR)”

```
routes/
  +layout.svelte          # 공통 헤더
  +layout.ts              # default: ssr=true, csr=true
  +page.svelte            # 홈 — 마케팅
  pricing/
    +page.svelte          # 마케팅
  app/
    +layout.ts            # 이 구간은 SSR 끔(순수 SPA)
    +layout.svelte
    dashboard/
      +page.svelte
```

```ts
// src/routes/+layout.ts
export const prerender = true; // 마케팅 전체 SSG
```

```ts
// src/routes/app/+layout.ts
export const ssr = false; // 앱은 CSR 전용 (로그인 후 대시보드 등)
```

- `/`와 `/pricing`은 **정적 HTML**로 배포, `/app/*`은 **CSR**.

### + 검색(SSR)”

```ts
// src/routes/blog/+layout.ts
export const prerender = true; // 목록/태그 페이지 SSG
```

```ts
// src/routes/search/+page.ts
export const prerender = false; // 검색은 SSR(검색어마다 다름)
export const ssr = true;
```

---

## + 공통 사용자 세션

### API 라우트(목록/상세)

```ts
// src/routes/api/posts/+server.ts
import type { RequestHandler } from './$types';
const posts = [
  { slug: 'hello', title: 'Hello', excerpt: 'Hi', content: '<p>Hello</p>' },
  { slug: 'sveltekit', title: 'SvelteKit', excerpt: 'Routing', content: '<p>Routing!</p>' }
];

export const GET: RequestHandler = async () => {
  return new Response(JSON.stringify(posts.map(({ content, ...x }) => x)), {
    headers: { 'content-type': 'application/json' }
  });
};
```

```ts
// src/routes/api/posts/[slug]/+server.ts
import type { RequestHandler } from './$types';
const bySlug = (slug: string) => ({
  hello: { slug: 'hello', title: 'Hello', content: '<p>Hello</p>' },
  sveltekit: { slug: 'sveltekit', title: 'SvelteKit', content: '<p>Routing!</p>' }
} as Record<string, any>)[slug];

export const GET: RequestHandler = async ({ params }) => {
  const post = bySlug(params.slug);
  if (!post) return new Response('Not found', { status: 404 });
  return new Response(JSON.stringify(post), { headers: { 'content-type': 'application/json' } });
};
```

### 레이아웃: 사용자 세션

```ts
// src/routes/+layout.server.ts
import type { LayoutServerLoad } from './$types';

export const load: LayoutServerLoad = async ({ cookies }) => {
  const session = cookies.get('session');
  const user = session ? { name: 'Ada' } : null;
  return { user };
};
```

```svelte
<!-- src/routes/+layout.svelte -->
<script lang="ts"> export let data: { user: { name: string } | null };</script>

<nav class="top">
  <a href="/">Home</a>
  <a href="/blog">Blog</a>
  <span style="margin-left:auto" />
  {#if data.user} Hi, {data.user.name} {:else} <a href="/login">Login</a> {/if}
</nav>

<slot />

<style>
  .top { display:flex; gap:1rem; padding:.6rem 1rem; border-bottom:1px solid #e5e7eb; }
</style>
```

### 블로그: 리스트 SSG / 상세 SSR

```ts
// src/routes/blog/+layout.ts
export const prerender = true;  // /blog, /blog/* 크롤 (세부는 예외로)
```

```ts
// src/routes/blog/+page.ts
import type { PageLoad } from './$types';
export const load: PageLoad = async ({ fetch }) => {
  const posts = await fetch('/api/posts').then(r => r.json());
  return { posts };
};
```

```svelte
<!-- src/routes/blog/+page.svelte -->
<script lang="ts">
  export let data: { posts: { slug:string; title:string; excerpt:string }[] };
</script>

<h1>Blog</h1>
<ul>
  {#each data.posts as p}
    <li><a href={`/blog/${p.slug}`}>{p.title}</a> — {p.excerpt}</li>
  {/each}
</ul>
```

```ts
// src/routes/blog/[slug]/+page.ts
export const prerender = false; // 상세는 SSR (또는 CSR fetch)
export const ssr = true;
```

```ts
// src/routes/blog/[slug]/+page.ts
import type { PageLoad } from './$types';

export const load: PageLoad = async ({ params, fetch }) => {
  const post = await fetch(`/api/posts/${params.slug}`).then(r => r.json());
  return { post };
};
```

```svelte
<!-- src/routes/blog/[slug]/+page.svelte -->
<script lang="ts">
  export let data: { post: { title:string; content:string } };
</script>

<article class="prose">
  <h1>{data.post.title}</h1>
  {@html data.post.content}
</article>
```

- `/blog`는 **SSG**, `/blog/[slug]`는 **SSR**으로 **하이브리드** 구현.

---

## 페이지 옵션 모음 — 파일 단위로 제어

아래 상수들을 **`+layout.ts` 또는 `+page.ts`**에 export 하면 해당 구간/페이지에 적용된다.

```ts
export const ssr = true | false;        // 서버 렌더링 on/off
export const csr = true | false;        // 클라이언트 라우팅 on/off
export const prerender = true | false;  // 정적 생성 on/off
export const trailingSlash = 'never' | 'always' | 'ignore';
```

- `ssr=false`면 해당 구간은 **CSR 전용 SPA**
- `csr=false`면 **클라이언트 전환 비활성화**(항상 **완전한 페이지 새로고침**으로 이동)
- `prerender=true`면 **정적 생성**(가능한 경로에 한해)

---

## 흔한 함정 & 체크리스트

1) **`.server.ts`에서만 가능한 일**과 **유니버설에서 가능한 일**을 혼동
   - 비밀키/DB/쿠키 민감 로직은 **`.server.ts`**에서.
   - 브라우저 API는 **유니버설 load**나 컴포넌트의 **onMount**에서.

2) **캐스케이딩 데이터 덮어쓰기**
   - 상위/하위 `load`의 반환이 **병합**된다는 점 기억(키 충돌 주의).
   - 이름을 네임스페이스처럼 나눠(`layoutUser`, `pageData`) 혼동을 줄이자.

3) **동적 파라미터 프리렌더링**
   - 무한/대규모 slug를 `prerender=true`로 그대로 두면 **크롤 불가**.
   - `entries`로 **부분만**, 나머지는 **SSR** 또는 **CSR**.

4) **CSR만으로는 SEO 어려움**
   - 마케팅/문서 페이지는 **SSG/SSR**을 고려.

5) **라우팅 파일 위치 오타**
   - 반드시 `src/routes` 아래에 파일을 두고, 파일명은 `+page.svelte` 등 **정확하게**.

---

## 요약

- **파일명으로 역할 고정**: `+layout(.server).ts`/`+page(.server).ts`/`+server.ts`
- **캐스케이딩 데이터**로 상위 레이아웃 → 하위 페이지로 상태 전파
- **파일 기반 라우팅**: `[slug]`, `[...path]`, `[[lang]]` 등으로 유연한 경로
- **렌더링 모드**를 **페이지 단위**로 조합: SSR/SPA/SSG + 프리렌더 하이브리드
- 배포/SEO/보안 요구에 맞춰 **전략적으로 섞어** 쓰는 것이 핵심
