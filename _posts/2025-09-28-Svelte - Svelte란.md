---
layout: post
title: Svelte - Svelte란
date: 2025-09-28 14:25:23 +0900
category: Svelte
---
# Svelte란

> **한 줄 요약**: Svelte는 “런타임 프레임워크”보다 **컴파일러**에 더 가깝다. 빌드 타임에 템플릿과 반응성을 **정적 분석**해 **작고 빠른(lean) 자바스크립트**로 바꿔치기한다. 그래서 **가상 DOM 없이**도 반응 UI를 만든다.

---

## 0.1 Svelte가 뭔가요? — 철학과 원리

### 0.1.1 “프레임워크 없는 프레임워크”
- **컴파일러 철학**: Svelte는 문법(템플릿/반응성)을 **빌드 시** 해석하여 실제 DOM 조작 코드로 **컴파일**한다. 런타임에 무거운 추상화 계층(예: 가상 DOM diff)을 들고 다니지 않는다. 결과적으로 전송 자바스크립트가 적고 렌더/업데이트가 빠르다.
- **효과**:
  - 번들 크기↓, 런타임 오버헤드↓
  - 기본기(HTML/CSS/JS)에 가까운 DX(개발 경험)
  - 특정 패턴(양방향 바인딩, transition 등)을 **문법 차원**에서 제공

> **핵심 개념**
> - **정적 분석**: 템플릿에서 데이터 흐름을 추적, 변경 지점만 DOM 패치
> - **반응성(reactivity)**: 값이 바뀌면, 관련된 바인딩과 DOM에 즉시 반영
> - **스코프 CSS**: 컴포넌트 CSS는 기본적으로 컴포넌트 내부에만 적용(해시 기반)

### 0.1.2 리액트/뷰와의 차이 — 가상 DOM 없는 반응성
- **React/Vue**: 컴포넌트 트리를 메모리에 유지(가상 DOM 또는 Proxy 기반), **런타임**에서 변경을 계산.
- **Svelte**: **컴파일 단계**에서 변경 지점을 알고, **필요한 DOM 지시문**을 미리 생성. 런타임엔 “정답지”만 실행.
  → **업데이트 경로가 짧고 예측 가능**. 복잡한 diff가 필요 없다.

#### 작은 실험
```html
<!-- Counter.svelte -->
<script>
  let count = 0;
  const inc = () => count++;
</script>

<button on:click={inc}>
  Count: {count}
</button>
```
- 위 코드는 **컴파일**되며, `count`가 변할 때 **해당 텍스트 노드만** 업데이트하는 JS로 변환된다(가상 DOM 없음).

### 0.1.3 “컴파일러다”가 주는 이점과 트레이드오프
- **장점**
  - 성능과 번들 크기가 좋다(특히 상호작용이 적당한 규모에서 돋보임).
  - 반응성 문법이 간단해 **학습 곡선이 완만**.
  - CSS 스코프/트랜지션/액션 등 **일상 기능이 내장**.
- **트레이드오프**
  - **생태계 규모**와 **채용 시장**은 React 대비 상대적으로 작음.
  - **런타임 플러그인**보다 **빌드 파이프라인**을 이해해야 하는 경우가 있다.
  - 대규모 조직의 기존 표준(React/Next)과의 정합성 고려 필요.

---

## 0.2 언제 Svelte/SvelteKit을 쓰면 좋은가

### 0.2.1 추천 시나리오
- **콘텐츠/문서 중심 SSG**: 블로그/문서 사이트 — 빠른 **SSG**와 작은 클라이언트 번들이 장점.
- **상호작용은 적당, 레이턴시는 민감**: 마케팅/랜딩/대시보드의 초기 로드 최적화.
- **Form 중심 UX**: SvelteKit **폼 액션**(progressive enhancement)으로 JS 없이도 동작하는 흐름을 만들기 쉬움.
- **Edge/서버리스**: 다양한 **어댑터**로 Node/서버리스/Edge/정적 배포를 유연하게 구성 가능(Cloudflare/Vercel/Netlify 등).

### 0.2.2 고민이 필요한 시나리오
- **거대한 컴포넌트 생태계**(수천 개 npm UI 패키지 전제)나 **팀 표준이 이미 React**: 온보딩/공유 자산 측면에서 React 쏠림이 현실적.
- **사내 공용 런타임 추상화**(예: 기업 공통 위젯 SDK)가 **가상 DOM** 가정에 강하게 묶여 있다면 적합성 검토 필요.

---

## 0.3 버전 한눈에 보기 — Svelte 4 vs 5 (Runes 간단 비교)

> **Svelte 5의 가장 큰 변화**: 반응성 모델을 “룬(Runes)”으로 **명시적**이고 **타입 친화적**으로 정리. `$state`, `$derived`, `$effect` 등.

### 0.3.1 왜 Runes?
- v3/4에서는 `let count = 0`처럼 **변수 자체가 반응형**(재할당 기반 업데이트)이었지만, 복잡도가 커질수록 **의도/의존성**이 모호할 수 있었다.
- v5는 반응 상태/파생 값/부작용을 **룬으로 분리**해 코드의 의미가 더 명확해졌다(타입 추론 및 에디터 지원 향상 보고 사례 다수).

### 0.3.2 핵심 Runes 한눈에
- **`$state(initial)`**: *상태*를 선언. 깊은(Deep) 반응성 지원.
- **`$derived(expr)`**: *파생 값*을 선언(의존 값이 변하면 자동 갱신, 부작용 금지 권장).
- **`$effect(fn)`**: *부작용*을 수행. 마운트 시 1회 + 의존 변경마다 실행(SSR에서는 실행되지 않음).

#### v4 스타일(감 잡기)
```svelte
<script>
  // v4: 변수 자체가 반응형 — 재할당이 신호
  let count = 0;
  $: double = count * 2; // 반응식
</script>

<button on:click={() => count++}>
  {count} (x2={double})
</button>
```

#### v5 스타일(룬)
```svelte
<script>
  // v5: 의도를 명시 — 상태/파생/부작용을 분리
  let count = $state(0);
  let double = $derived(count * 2);

  $effect(() => {
    console.log('count 변경됨:', count);
  });
</script>

<button on:click={() => count++}>
  {count} (x2={double})
</button>
```
- **차이점 요약**
  - v4: *“재할당 기반 반응성”*이 기본.
  - v5: 상태/파생/효과를 **명시적 API**로 선언해 **가독성/타이핑**이 개선.

### 0.3.3 깊은(Deep) 반응성의 실감 예시
```svelte
<script>
  // 중첩 객체도 반응적으로 추적
  let profile = $state({ name: 'Ada', skills: ['Svelte', 'TypeScript'] });

  function addSkill(s) {
    profile.skills.push(s); // 중첩 변경도 반응
  }

  let skillCount = $derived(profile.skills.length);
</script>

<p>{profile.name} — {skillCount} skills</p>
<button on:click={() => addSkill('Runes')}>Add</button>
```
- v5의 `$state`는 **중첩 속성 변경**도 자동 추적 → 파생 값/DOM이 갱신된다(“deep reactivity”).

### 0.3.4 마이그레이션 관점
- v4 문법은 **대부분 호환**되며 점진적 전환 가능.
- 팀 합의: **신규 코드**는 Runes, 기존은 **필요 시 리팩터링**.

---

## 0.4 로드맵과 생태계 개관

### 0.4.1 Svelte/SvelteKit 현황(2025)
- **지속적 릴리스**: 언어 도구/어댑터/런타임의 잦은 버그 픽스와 개선 보고. 공식 블로그의 월간 “What’s new”에서 추적 가능.
- **어댑터 다양성**: Node, 서버리스(AWS Lambda), Edge(CF Workers 등), 정적 사이트 — **배포 타깃을 바꾸는 일**이 비교적 수월. 실전 가이드도 풍부.
- **문법/개념 통일**: v5에서 반응성 모델이 Runes로 **명료화**되어, 문서/튜토리얼/커뮤니티 글이 빠르게 정돈되는 추세.

### 0.4.2 에코시스템 지도(간단 버킷)
- **UI/스타일**: Tailwind, Skeleton/Flowbite/Svelte-Material류
- **데이터**: REST/GraphQL 클라이언트, `load`/서버 라우트, 폼 액션
- **상태**: 기본 스토어(간단) + Runes, 필요 시 외부 상태 라이브러리
- **빌드/배포**: Vite 기반, Adapter 교체로 Node/서버리스/Edge/SSG
- **테스트**: Vitest(유닛/컴포넌트), Playwright(E2E)
- **관측**: Sentry/Logflare/Cloud tooling

---

# 1. Svelte의 핵심 감각 — “반응성은 값의 **의미**다”

## 1.1 템플릿과 상태 — DOM을 ‘정확히’ 업데이트

### 1.1.1 이벤트/바인딩 기초
```svelte
<script>
  let name = $state('world');
</script>

<input bind:value={name} placeholder="Your name" />
<h1>Hello {name}!</h1>
```
- `bind:value`는 입력 변경을 상태에 반영, 상태가 바뀌면 DOM 텍스트가 **정밀 패치**된다.

### 1.1.2 조건/반복/비동기 블록
```svelte
<script>
  let items = $state(['A', 'B', 'C']);
  let loading = $state(false);

  async function loadMore() {
    loading = true;
    await new Promise(r => setTimeout(r, 300));
    items.push('D');
    loading = false;
  }
</script>

{#if loading}
  <p>Loading...</p>
{:else}
  <ul>{#each items as it}<li>{it}</li>{/each}</ul>
  <button on:click={loadMore}>More</button>
{/if}
```

---

# 2. SvelteKit을 곁들인 선택 — “페이지부터 데이터까지”

## 2.1 라우팅/레이아웃/로드

### 2.1.1 파일 기반 라우팅
```
src/routes/
  +layout.svelte       # 공통 레이아웃
  +page.svelte         # / 페이지
  about/
    +page.svelte       # /about
  items/[id]/
    +page.svelte       # /items/:id
```

### 2.1.2 서버와 클라이언트 `load`
```ts
// src/routes/items/[id]/+page.ts
export const load = async ({ fetch, params }) => {
  const res = await fetch(`/api/items/${params.id}`);
  return { item: await res.json() };
};
```

## 2.2 폼 액션 — JS 없이도 동작하는 UX
SvelteKit은 **서버 액션**으로 폼을 처리한다. **점진적 향상**을 기본으로 삼아, JS가 꺼져도 동작하며, 켜져 있으면 **부분 갱신**과 **낙관적 UI**를 유연히 구성한다.
```svelte
<!-- src/routes/signup/+page.svelte -->
<form method="POST">
  <input name="email" type="email" required />
  <button type="submit">Sign up</button>
</form>
```

```ts
// src/routes/signup/+page.server.ts
import { fail, redirect } from '@sveltejs/kit';

export const actions = {
  default: async ({ request }) => {
    const form = await request.formData();
    const email = String(form.get('email') || '');
    if (!email.includes('@')) return fail(400, { error: 'Invalid email' });
    // ...save to DB
    throw redirect(303, '/welcome');
  }
};
```

---

# 3. Svelte 4 → 5 사고 전환 가이드(살짝 더 깊게)

## 3.1 상태/파생/부작용의 분리

### 3.1.1 `$state` — 상태 선언
```svelte
<script>
  let todo = $state({ title: '', done: false });

  function toggle() {
    todo.done = !todo.done;  // 중첩 필드 변경도 추적
  }
</script>

<label><input type="checkbox" bind:checked={todo.done} on:change={toggle} /> {todo.title}</label>
```
- 중첩 객체 변경도 반응. **깊은 추적**으로 파생/UI가 자연 갱신.

### 3.1.2 `$derived` — 부작용 없는 계산 값
```svelte
<script>
  let cart = $state([{ name: 'Book', price: 12 }, { name: 'Pen', price: 3 }]);
  let total = $derived(cart.reduce((sum, it) => sum + it.price, 0));
</script>

<p>Total: {total}</p>
```
- `$derived(...)` 내부는 **부작용이 없어야** 하며, 의존 상태가 변할 때만 다시 계산된다.

### 3.1.3 `$effect` — DOM 이후 타이밍의 부작용
```svelte
<script>
  let query = $state('');
  $effect(() => {
    console.debug('Search query changed:', query);
    // fetch/observe/analytics 등: DOM 업데이트 이후 타이밍
  });
</script>

<input bind:value={query} placeholder="Search..." />
```
- 마운트 직후와 의존 변경 때마다 실행. **SSR에서는 실행되지 않음**.

---

# 4. 실전 체크리스트 — “언제 SvelteKit을 고를까?”

- **초기 TTFB/CLS를 줄이고 싶다**: SSR/프리렌더로 **즉시 콘텐츠**.
- **Form가 많은 사이트**: SvelteKit **폼 액션** + 점진적 향상.
- **배포 타깃이 자주 바뀐다**: 어댑터로 Node/서버리스/Edge/SSG 전환.
- **복잡한 전역 상태는 적고, 페이지 내 상호작용이 잦다**: Runes/스토어로 충분.
- **팀이 HTML/CSS 친화적**: Svelte의 템플릿 문법이 적합.

---

# 5. 로드맵/생태계 추적 팁

- **공식 블로그 “What’s new in Svelte (월간)”**: 핵심 이슈/변경사항 빠르게 파악.
- **문서의 Runes 레퍼런스**: `$state`, `$derived`, `$effect` 동작을 정확히 확인.
- **배포 가이드/어댑터**: 플랫폼별 모범 사례(예: AWS Lambda 경유 구성) 확인.

---

# 6. 작은 데모: “SSG 블로그 홈 + 폼 액션 구독”

> 홈은 **프리렌더**로 빌드, `/subscribe`는 **폼 액션**으로 서버에서 이메일 검증 후 리다이렉트.

```bash
# 새 프로젝트(예: SvelteKit 기본)
pnpm create svelte@latest my-blog
cd my-blog
pnpm i
```

```ts
// svelte.config.ts — (예시) 정적 프리렌더 옵션
import adapter from '@sveltejs/adapter-auto';
import { vitePreprocess } from '@sveltejs/vite-plugin-svelte';

const config = {
  preprocess: vitePreprocess(),
  kit: {
    adapter,
    prerender: { entries: ['/'] } // 홈 페이지만 미리 렌더
  }
};

export default config;
```

```svelte
<!-- src/routes/+page.svelte -->
<script>
  let posts = $state([
    { slug: 'svelte-why', title: 'Svelte, 왜 쓰나요?' },
    { slug: 'runes', title: 'Svelte 5 Runes 입문' }
  ]);
</script>

<h1>My Blog</h1>
<ul>
  {#each posts as p}
    <li><a href={`/posts/${p.slug}`}>{p.title}</a></li>
  {/each}
</ul>

<p><a href="/subscribe">뉴스레터 구독</a></p>
```

```svelte
<!-- src/routes/subscribe/+page.svelte -->
<h1>Subscribe</h1>
<form method="POST">
  <input name="email" type="email" placeholder="you@example.com" required />
  <button type="submit">Subscribe</button>
</form>
```

```ts
// src/routes/subscribe/+page.server.ts
import { fail, redirect } from '@sveltejs/kit';

export const actions = {
  default: async ({ request }) => {
    const fd = await request.formData();
    const email = String(fd.get('email') || '').trim();

    if (!email.includes('@')) {
      return fail(400, { error: 'Invalid email' });
    }

    // TODO: 저장/메일 발송
    throw redirect(303, '/thanks');
  }
};
```

```svelte
<!-- src/routes/thanks/+page.svelte -->
<h1>감사합니다! 구독이 완료되었어요.</h1>
```
- **프리렌더된 홈**은 JS 없이도 즉시 표시.
- **구독 폼**은 JS가 없어도 정상 동작, JS가 있으면 더 부드러운 전환.

---

# 7. Q&A — 자주 받는 오해와 답

**Q1. “Svelte는 가상 DOM이 없는데, 그래서 더 빠른가요?”**
A. “항상” 더 빠르다는 뜻은 아니다. 다만 Svelte는 **빌드 타임 지식**으로 업데이트를 생성해, **불필요한 diff 계산을 줄인다**. 패턴에 따라 **작고 빠른** 경향이 뚜렷하다.

**Q2. “v4 프로젝트를 v5로 당장 바꿔야 하나요?”**
A. 필수는 아니다. v4도 견고하다. 다만 **신규 코드에 Runes를 도입**하면 의도가 더 명료해지고, **깊은 반응성**/타이핑이 좋아지는 이점이 있다. 점진적 마이그레이션을 권장.

**Q3. “서버리스/Edge 배포는 까다롭지 않나요?”**
A. **어댑터**로 배포 타깃을 전환하는 것이 일반적 패턴. 각 플랫폼에 맞춘 예제가 다수 존재하며, Node/Lambda/ALB/CloudFront 구성 실전 글도 활발하다.

---

# 8. 마무리 — 선택 기준 한 장 요약

- **개발·운영 단순성**: 템플릿/반응성 내장 + SSG/SSR/Actions 내장
- **성능·용량 목표**: 초반 페인트/상호작용 빠르게, 번들 가볍게
- **팀 배경**: HTML/CSS/JS 친화적이면 러닝커브 완만
- **생태계**: 필요한 범위 내에서 충분, 다만 “압도적” 규모를 원하면 검토 필요
- **버전**: 신규는 v5 Runes(명시적 반응성) 추천, 기존은 점진 전환
