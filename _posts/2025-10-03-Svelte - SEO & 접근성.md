---
layout: post
title: Svelte - SEO & 접근성
date: 2025-10-03 17:30:23 +0900
category: Svelte
---
# 11. SEO & 접근성
**메타/오픈그래프/구조화 데이터 · i18n 다국어 라우팅 · A11y 체크리스트(포커스/키보드/ARIA)**

> 이 장은 SvelteKit로 **검색 친화적(SEO)**이고 **접근 가능한(A11y)** 웹을 만드는 데 필요한 실무 지식을 한곳에 모은다.
> - **메타/OG/Twitter/구조화 데이터(JSON-LD)**를 라우트별로 정확히 출력
> - **다국어(i18n) 라우팅**과 `hreflang`/정규화(canonical)
> - **접근성 체크리스트**: 포커스 흐름, 키보드 내비, ARIA, 라이브 영역, 색 대비 등

---

## 11.1 SvelteKit에서 **메타 태그** 관리하기

### 11.1.1 라우트별 `<svelte:head>` 기본
```svelte
<!-- src/routes/about/+page.svelte -->
<script lang="ts">
  export let data: {
    title: string;
    description: string;
    url: string;
    image: string;
  };
</script>

<svelte:head>
  <title>{data.title}</title>
  <meta name="description" content={data.description} />

  <!-- Open Graph -->
  <meta property="og:type" content="website" />
  <meta property="og:title" content={data.title} />
  <meta property="og:description" content={data.description} />
  <meta property="og:url" content={data.url} />
  <meta property="og:image" content={data.image} />

  <!-- Twitter Card -->
  <meta name="twitter:card" content="summary_large_image" />
  <meta name="twitter:title" content={data.title} />
  <meta name="twitter:description" content={data.description} />
  <meta name="twitter:image" content={data.image} />

  <!-- Canonical / Hreflang(후술) -->
  <link rel="canonical" href={data.url} />
</svelte:head>

<article class="prose">
  <h1>About</h1>
  <p>…</p>
</article>
```

### 11.1.2 `+page.ts`에서 메타 데이터 구성
```ts
// src/routes/about/+page.ts
import type { PageLoad } from './$types';

export const load: PageLoad = async ({ url }) => {
  const title = 'About – MyApp';
  const description = 'MyApp에 대해 소개합니다.';
  const image = `${url.origin}/og/about.png`; // 정적 또는 동적 생성
  const canonical = `${url.origin}${url.pathname}`; // 쿼리 제거

  return { title, description, image, url: canonical };
};
```

> **권장**: 쿼리스트링이 SEO에 의미 없으면 **canonical**을 쿼리 제거 버전으로 고정.

### 11.1.3 전역 기본값(루트 레이아웃)
```svelte
<!-- src/routes/+layout.svelte -->
<script lang="ts">
  export let data: { site: { name: string; baseUrl: string; defaultImage: string } };
</script>

<svelte:head>
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <meta name="theme-color" content="#0ea5e9" />
  <meta name="generator" content="SvelteKit" />

  <!-- 기본 OG -->
  <meta property="og:site_name" content={data.site.name} />
  <meta property="og:image" content={data.site.defaultImage} />
</svelte:head>

<slot />
```

```ts
// src/routes/+layout.server.ts
import type { LayoutServerLoad } from './$types';
export const load: LayoutServerLoad = async ({ url }) => {
  return {
    site: {
      name: 'MyApp',
      baseUrl: url.origin,
      defaultImage: `${url.origin}/og/default.png`
    }
  };
};
```

### 11.1.4 성능 관련 `<link>` 힌트
```svelte
<svelte:head>
  <!-- 폰트/API 도메인 연결 -->
  <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin />
  <link rel="dns-prefetch" href="https://cdn.example.com" />
  <!-- 중요 자원 프리로드 -->
  <link rel="preload" as="image" href="/hero.webp" imagesrcset="/hero@2x.webp 2x" />
</svelte:head>
```

---

## 11.2 **Open Graph 이미지** 동적 생성(서버)

### 11.2.1 텍스트 → 이미지(간단 SVG → PNG/WebP 변환 예시)
```ts
// src/routes/og/[slug]/+server.ts
import type { RequestHandler } from './$types';

const svg = (title: string) => `
<svg xmlns="http://www.w3.org/2000/svg" width="1200" height="630">
  <rect width="100%" height="100%" fill="#0ea5e9"/>
  <text x="60" y="340" font-size="72" fill="#fff" font-family="system-ui,Segoe UI" >
    ${title.replace(/</g,'&lt;')}
  </text>
</svg>`;

export const GET: RequestHandler = async ({ params, url }) => {
  const title = params.slug.replace(/-/g, ' ').slice(0, 80);
  const body = svg(title);
  // 대부분의 소셜은 SVG도 미리보기 지원(안정 위해 PNG 변환을 권장: sharp/renderer)
  return new Response(body, { headers: { 'content-type': 'image/svg+xml', 'cache-control':'public, max-age=86400' } });
};
```

> **팁**: 기본은 SVG로 간단 구현 → 운영에선 **PNG/WebP** 변환과 폰트 임베드로 일관성 확보.

---

## 11.3 **구조화 데이터(JSON-LD)**

### 11.3.1 Article(블로그 글)
```svelte
<!-- src/routes/blog/[slug]/+page.svelte -->
<script lang="ts">
  export let data: {
    post: {
      title: string; description: string; slug: string;
      datePublished: string; dateModified: string; author: string; image: string;
    };
    site: { name: string; baseUrl: string };
  };

  const ld = {
    '@context': 'https://schema.org',
    '@type': 'BlogPosting',
    headline: data.post.title,
    image: [`${data.site.baseUrl}${data.post.image}`],
    datePublished: data.post.datePublished,
    dateModified: data.post.dateModified,
    author: [{ '@type': 'Person', name: data.post.author }],
    publisher: { '@type': 'Organization', name: data.site.name },
    mainEntityOfPage: `${data.site.baseUrl}/blog/${data.post.slug}`,
    description: data.post.description
  };
</script>

<svelte:head>
  <title>{data.post.title} – {data.site.name}</title>
  <meta name="description" content={data.post.description}/>
  <script type="application/ld+json">
    {JSON.stringify(ld)}
  </script>
</svelte:head>

<article class="prose">
  <h1>{data.post.title}</h1>
  <!-- ... -->
</article>
```

### 11.3.2 Organization + Breadcrumbs
```svelte
<svelte:head>
  <script type="application/ld+json">
    {JSON.stringify({
      '@context':'https://schema.org','@type':'Organization',
      name:'MyApp', url: data.site.baseUrl,
      logo: `${data.site.baseUrl}/logo.png`
    })}
  </script>

  <script type="application/ld+json">
    {JSON.stringify({
      '@context': 'https://schema.org',
      '@type': 'BreadcrumbList',
      itemListElement: [
        { '@type':'ListItem', position:1, name:'Home', item: data.site.baseUrl },
        { '@type':'ListItem', position:2, name:'Blog', item: `${data.site.baseUrl}/blog` },
        { '@type':'ListItem', position:3, name:data.post.title, item: `${data.site.baseUrl}/blog/${data.post.slug}` }
      ]
    })}
  </script>
</svelte:head>
```

> **주의**: JSON-LD는 **실제 페이지 컨텐츠**와 어긋나면 감점/무시될 수 있다.

---

## 11.4 **i18n 다국어 라우팅**

### 11.4.1 폴더 구조 패턴
```
src/routes/
  [[lang]]/                 # 선택적 언어 세그먼트
    +layout.server.ts       # 언어 결정/검증
    +layout.svelte          # <html lang="..">와 hreflang 링크
    +page.svelte
    about/
      +page.svelte
```

- `/about` (기본 언어)
- `/en/about`, `/ja/about` (명시 언어)

### 11.4.2 언어 결정 로직(서버)
```ts
// src/routes/[[lang]]/+layout.server.ts
import type { LayoutServerLoad } from './$types';

const SUPPORTED = ['ko', 'en', 'ja'] as const;
type Lang = (typeof SUPPORTED)[number];

export const load: LayoutServerLoad = async ({ params, request, url, cookies }) => {
  let lang = (params.lang as Lang) || cookies.get('lang') as Lang || 'ko';

  if (!SUPPORTED.includes(lang)) {
    // 알 수 없는 언어 → 기본으로
    lang = 'ko';
  }
  // remember
  cookies.set('lang', lang, { path:'/', httpOnly:false, sameSite:'lax', secure:true, maxAge:60*60*24*365 });

  const base = url.origin;
  const path = url.pathname.replace(/^\/(ko|en|ja)(\/|$)/, '/'); // 언어 프리픽스 제거

  // hreflang 목록
  const alternates = SUPPORTED.map((l) => ({
    lang: l,
    href: `${base}${l==='ko' ? '' : `/${l}`}${path}`
  }));

  return { lang, alternates, canonical: `${base}${lang==='ko' ? '' : `/${lang}`}${path}` };
};
```

### 11.4.3 `<html lang>`과 `hreflang`
```svelte
<!-- src/routes/[[lang]]/+layout.svelte -->
<script lang="ts">
  export let data: { lang: string; alternates: { lang:string; href:string }[]; canonical: string };
</script>

<svelte:head>
  <link rel="canonical" href={data.canonical} />
  {#each data.alternates as a}
    <link rel="alternate" hreflang={a.lang} href={a.href} />
  {/each}
  <!-- x-default(선택): 기본 언어 -->
  <link rel="alternate" hreflang="x-default" href={data.alternates.find(a=>a.lang==='ko')?.href} />
</svelte:head>

<!-- SvelteKit는 <html> 엘리먼트를 app.html에서 관리.
     lang 속성은 서버 훅 transformPageChunk 또는 아래처럼 body의 data-attr을 이용해 제어 -->
<div lang={data.lang} data-app-lang={data.lang} style="display:contents">
  <slot />
</div>
```

> **참고**: `app.html`에서 서버 훅(`transformPageChunk`)로 `<html lang="">`을 동적으로 바꾸는 패턴도 가능.

### 11.4.4 번역 문자열 관리(간단 키-값)
```
src/lib/i18n/
  messages.ts
  index.ts
```

```ts
// src/lib/i18n/messages.ts
export const dict = {
  ko: { title: '안녕하세요', about: '소개' },
  en: { title: 'Hello', about: 'About' },
  ja: { title: 'こんにちは', about: '概要' }
} as const;

export type Lang = keyof typeof dict;
```

```ts
// src/lib/i18n/index.ts
import { dict, type Lang } from './messages';
export function t(lang: Lang, key: keyof (typeof dict)['ko']) { return dict[lang][key]; }
```

사용:
```svelte
<script lang="ts">
  import { t } from '$lib/i18n';
  export let data: { lang: 'ko'|'en'|'ja' };
</script>

<h1>{t(data.lang, 'title')}</h1>
<nav><a href="/about">{t(data.lang, 'about')}</a></nav>
```

### 11.4.5 리다이렉트/언어 협상(최초 방문)
- 첫 방문 시 `Accept-Language`를 참고해 추천 언어로 **302** 리다이렉트
- 이미 쿠키에 `lang` 있으면 존중

```ts
// src/hooks.server.ts (일부)
import type { Handle } from '@sveltejs/kit';
const SUPPORTED = ['ko','en','ja'];

export const handle: Handle = async ({ event, resolve }) => {
  const { url, request, cookies } = event;
  const hasLang = /^\/(ko|en|ja)(\/|$)/.test(url.pathname);
  const saved = cookies.get('lang');

  if (!hasLang && !saved) {
    const accept = request.headers.get('accept-language') || '';
    const preferred = SUPPORTED.find(l => accept.toLowerCase().includes(l)) || 'ko';
    return Response.redirect(`${url.origin}${preferred==='ko' ? '' : `/${preferred}`}${url.pathname}`, 302);
  }
  return resolve(event);
};
```

---

## 11.5 검색 크롤링 관련 파일: `robots.txt`/`sitemap.xml`

### 11.5.1 robots.txt
```ts
// src/routes/robots.txt/+server.ts
import type { RequestHandler } from './$types';
export const GET: RequestHandler = ({ url }) => {
  const body = `User-agent: *
Allow: /
Sitemap: ${url.origin}/sitemap.xml`;
  return new Response(body, { headers: { 'content-type': 'text/plain' } });
};
```

### 11.5.2 sitemap.xml(단순)
```ts
// src/routes/sitemap.xml/+server.ts
import type { RequestHandler } from './$types';

const staticPages = ['/', '/about', '/blog'];

export const GET: RequestHandler = ({ url }) => {
  const base = url.origin;
  const xml =
    `<?xml version="1.0" encoding="UTF-8"?>
     <urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
       ${staticPages.map(p=>`<url><loc>${base}${p}</loc></url>`).join('')}
     </urlset>`;
  return new Response(xml.trim(), { headers: { 'content-type': 'application/xml', 'cache-control':'public, max-age=86400' } });
};
```

> **실무**: DB/파일 시스템에서 **모든 공개 글**을 조회하여 동적으로 생성하거나, 빌드 단계에서 프리렌더.

---

## 11.6 A11y(접근성) **기본 원칙**

1) **의미론적 HTML**: 가능한 `<button>`, `<a>`, `<label>`, `<fieldset>`, `<legend>`, `<nav>` 등 사용
2) **포커스 가능/순서**: 키보드(Tab/Shift+Tab)로 모든 인터랙션 가능
3) **명확한 포커스 표시**: 커스텀 스타일 시 기본 outline 제거하면 **대체 포커스** 제공
4) **대체 텍스트**: 의미 있는 이미지 `alt` 제공, 장식은 `alt=""`
5) **색 대비**: 최소 4.5:1(본문), 3:1(굵은 큰 텍스트)
6) **ARIA는 마지막 수단**: 의미론이 부족할 때만 `role`/`aria-*` 보강
7) **애니메이션 배려**: `prefers-reduced-motion` 준수
8) **오류/상태 전달**: `aria-live`/`role="alert"`로 즉시 전달

---

## 11.7 포커스 관리 & 키보드 내비

### 11.7.1 스킵 링크
```svelte
<!-- src/routes/+layout.svelte -->
<a href="#main" class="skip">본문 바로가기</a>
<header>…</header>
<main id="main"><slot/></main>

<style>
.skip{position:absolute;left:-9999px;top:auto;width:1px;height:1px;overflow:hidden}
.skip:focus{left:12px;top:12px;width:auto;height:auto;background:#fff;padding:.5rem;border:2px solid #0ea5e9;border-radius:8px}
</style>
```

### 11.7.2 키보드 트랩(모달)
```svelte
<!-- src/lib/components/Modal.svelte -->
<script lang="ts">
  import { onMount, onDestroy } from 'svelte';
  export let open = false;
  let dialogEl: HTMLDivElement;

  function keydown(e: KeyboardEvent) {
    if (e.key === 'Escape') open = false;
    if (e.key === 'Tab') {
      const focusables = dialogEl.querySelectorAll<HTMLElement>('[tabindex],a,button,input,select,textarea');
      const list = Array.from(focusables).filter(el => !el.hasAttribute('disabled') && el.tabIndex >= 0);
      const first = list[0]; const last = list[list.length-1];
      if (e.shiftKey && document.activeElement === first) { e.preventDefault(); last?.focus(); }
      else if (!e.shiftKey && document.activeElement === last) { e.preventDefault(); first?.focus(); }
    }
  }

  onMount(() => document.addEventListener('keydown', keydown));
  onDestroy(() => document.removeEventListener('keydown', keydown));
</script>

{#if open}
  <div class="backdrop" role="dialog" aria-modal="true" aria-labelledby="title" aria-describedby="desc">
    <div class="panel" bind:this={dialogEl} tabindex="-1">
      <h2 id="title">제목</h2>
      <p id="desc">설명</p>
      <slot />
      <button on:click={() => open=false}>닫기</button>
    </div>
  </div>
{/if}

<style>
.backdrop{position:fixed;inset:0;background:rgba(0,0,0,.45);display:grid;place-items:center}
.panel{background:#fff;border-radius:12px;padding:1rem;min-width:320px;outline:0}
</style>
```

> **포인트**: `role="dialog"` + `aria-modal="true"` + label/description 연결, Tab 트랩, ESC 닫기.

### 11.7.3 포커스 표시 커스텀
```css
:focus-visible { outline: 3px solid #0ea5e9; outline-offset: 2px; }
button:focus-visible { box-shadow: 0 0 0 3px rgba(14,165,233,.5); }
```

---

## 11.8 ARIA & 라이브 영역

### 11.8.1 라이브 알림
```svelte
<!-- src/lib/components/Announcer.svelte -->
<script lang="ts">
  import { writable } from 'svelte/store';
  export const announce = writable<string>('');

  $: if ($announce) {
    // 화면리더에만 읽히고 시각적으로 숨김
    setTimeout(() => announce.set(''), 1000);
  }
</script>

<div aria-live="polite" aria-atomic="true" class="sr-only">{$announce}</div>

<style>
.sr-only{position:absolute;width:1px;height:1px;overflow:hidden;clip:rect(1px,1px,1px,1px);white-space:nowrap}
</style>
```

사용:
```svelte
<script lang="ts">
  import { announce } from '$lib/components/Announcer.svelte';
  function saved(){ announce.set('저장되었습니다'); }
</script>

<button on:click={saved}>저장</button>
```

### 11.8.2 토글 가능한 요소의 ARIA
```svelte
<script>
  let expanded = false;
</script>

<button aria-expanded={expanded} aria-controls="sect" on:click={() => expanded=!expanded}>
  세부 정보 {expanded ? '접기' : '펼치기'}
</button>
<section id="sect" hidden={!expanded}>…</section>
```

> **규칙**: 상태는 **`aria-*`로 반영**, 컨텐츠 가시성은 실제 DOM/속성(`hidden`)으로 일치.

---

## 11.9 폼 접근성: 라벨/오류/도움말

### 11.9.1 라벨-컨트롤 연결
```svelte
<label for="email">이메일</label>
<input id="email" name="email" type="email" autocomplete="email" required />
```

### 11.9.2 오류와 도움말 연결
```svelte
<script>
  let error = '';
</script>

<label for="pw">비밀번호</label>
<input id="pw" name="password" type="password"
  aria-invalid={!!error}
  aria-describedby="pw-help {error ? 'pw-err' : ''}" />
<small id="pw-help">8자 이상, 숫자/문자 포함</small>
{#if error}<small id="pw-err" role="alert" style="color:#b00020">{error}</small>{/if}
```

> **중요**: 오류 메시지는 **`role="alert"`** 또는 **`aria-live="assertive"`**로 즉시 고지.

---

## 11.10 색 대비/모션/가독성

### 11.10.1 대비 확인 기준
- 본문/보통 텍스트: **4.5:1 이상**
- 큰 텍스트(≥24px or 19px bold): **3:1 이상**

```css
:root {
  --fg:#0b1220;
  --muted:#475569;
  --bg:#ffffff;
  --primary:#0ea5e9;
}
body { color: var(--fg); background: var(--bg); }
a { color: var(--primary); }
```

### 11.10.2 모션 최소화
```css
@media (prefers-reduced-motion: reduce) {
  * { animation: none !important; transition: none !important; scroll-behavior: auto !important; }
}
```

---

## 11.11 성능 & 인덱싱 실무 팁

- **SSR/SSG**: 마케팅/블로그는 SSR/SSG로 **초기 HTML** 제공
- **LCP 자원 최적화**: Hero 이미지 **preload** + 적절한 크기
- **CLS 방지**: 이미지에 `width/height` 또는 `aspect-ratio` 지정
- **번들 분할**: 라우트 레벨 코드 스플리팅, 큰 라이브러리 지연 로딩
- **Noindex** 필요한 페이지(내부/개발):
```svelte
<svelte:head><meta name="robots" content="noindex, nofollow" /></svelte:head>
```

---

## 11.12 체크리스트(요약)

### SEO
- [ ] 라우트별 **`<title>`/`description`/OG/Twitter** 정확
- [ ] **canonical**(쿼리 정규화), **hreflang**(다국어)
- [ ] **JSON-LD**(Article/Org/Breadcrumbs 등) 컨텐츠와 일치
- [ ] **robots.txt**/**sitemap.xml** 제공
- [ ] SSR/SSG로 **초기 HTML** 제공, 성능 최적화(LCP/CLS)

### i18n
- [ ] `[[lang]]` 라우팅 + 쿠키/헤더 협상
- [ ] `hreflang`/`x-default`/`canonical` 일관
- [ ] 번역 키 관리/빌드 검증(누락 키 탐지)

### A11y
- [ ] 의미론적 태그/랜드마크(`<header><nav><main><footer>`)
- [ ] **스킵 링크** 제공
- [ ] 키보드 내비게이션 완전 지원(Tab/Shift+Tab/ESC)
- [ ] 포커스 표시 가시성(커스텀 시 대체 제공)
- [ ] 이미지 `alt`/장식 `alt=""`
- [ ] 폼 라벨/오류 연결(`aria-invalid`/`aria-describedby`)
- [ ] 모달: `role="dialog"`, `aria-modal`, 포커스 트랩, ESC 닫기
- [ ] 라이브 상태: `aria-live`/`role="alert"`
- [ ] 색 대비 기준 충족(4.5:1), 모션 최소화 지원

---

## 11.13 마무리
- SvelteKit는 **SSR/파일 기반 라우팅**으로 라우트별 **메타/OG/JSON-LD** 관리가 용이하다.
- 다국어는 `[[lang]]` 패턴과 `hreflang`/canonical을 묶어서 설계하고, **언어 협상**(헤더/쿠키/경로)을 일관되게 운영.
- 접근성은 **설계 초반**부터: 의미론, 포커스, 키보드, ARIA, 라이브 영역, 대비/모션을 기본값으로.
- SEO와 A11y는 “**사용자에게 좋은 것 = 검색/도구에도 좋은 것**”이라는 동일 목표를 갖는다.
