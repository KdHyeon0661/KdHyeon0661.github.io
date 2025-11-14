---
layout: post
title: JavaScript - 모듈 시스템 (import, export)
date: 2025-05-04 20:20:23 +0900
category: JavaScript
---
# 자바스크립트 모듈 시스템 (import / export)

## 모듈 개념과 기본 규칙

### 모듈이란?

- **독립 파일 단위**의 코드 블록. 파일마다 **자체 스코프**를 가지며 전역 오염을 막습니다.
- 모듈 그래프(Module Graph): `import`가 만든 **유향 그래프**로, 엔진은 이를 정적으로 분석합니다.

### 핵심 특징

- **정적 import/export**: 구문이 정적이라 빌드/로더가 **의존성 분석·트리 셰이킹** 가능.
- **strict mode 자동**: 각 모듈 파일은 ‘use strict’ 상태.
- **라이브 바인딩(live binding)**: `export` 값은 참조로 연결되어 **값 변경이 반영**됩니다(재할당은 불가).
- **단 한 번 평가**: 동일 URL(해시/쿼리 포함) 모듈은 한 번만 실행되고 **캐시 공유**.

---

## export — 내보내기 문법

### Named Export (여러 개 가능)

```js
// math.js
export const PI = 3.1415926535;
export function add(a, b) { return a + b; }
const hidden = 42;
export { hidden as SECRET };
```

### Default Export (파일당 1개)

```js
// logger.js
export default function log(msg) { console.log(`[LOG] ${msg}`); }
```
- 파일당 **최대 1개**. 가져올 때 **이름은 자유**.

### 재수출(Re-export)

```js
// index.js
export * from './math.js';                 // math의 내보내기 모두 재수출(기본값 제외)
export { add as sum } from './math.js';    // 선택적 재수출(이름 변경)
export { default as log } from './logger.js';
```

### 선언과 동시에 export vs 묶어서 export

```js
// 동시
export const E = 2.718;
// 묶어서
const TAU = Math.PI * 2;
function circle(r){ return TAU * r; }
export { TAU, circle };
```

---

## import — 가져오기 문법

### Named Import

```js
import { PI, add } from './math.js';
import { SECRET as HIDDEN } from './math.js';
```

### Default Import

```js
import log from './logger.js';
log('hello');
```

### 네임스페이스 Import

```js
import * as math from './math.js';
console.log(math.PI, math.add(2,3));
```

### 혼합 Import

```js
import log, { PI, add as sum } from './pkg/index.js';
```

### 주의: 정적 구조

- `import`/`export`는 **문서 최상위(top level)** 에서만 사용.
- 조건부/동적 경로는 **동적 import**를 사용(아래 §7).

---

## 라이브 바인딩과 재할당 규칙

### 라이브 바인딩 동작

```js
// counter.js
export let count = 0;
export function inc(){ count += 1; }

// main.js
import { count, inc } from './counter.js';
console.log(count); // 0
inc();
console.log(count); // 1  ← 라이브 바인딩 반영
```

### 재할당 금지(읽기 전용 바인딩)

```js
import { count } from './counter.js';
// count = 10;  // TypeError: read-only binding
```
- **값 변경은 export 모듈 내부**에서만 수행하고, 가져오는 쪽은 **참조만** 보유.

---

## 브라우저에서의 ESM

### 기본 사용

```html
<!-- index.html -->
<script type="module" src="/main.js"></script>
```
- 모듈 스크립트는 **defer처럼 동작**(HTML 파싱 후 실행).
- **CORS 규칙** 준수, 같은 출처 또는 허용된 Origin만 로드.
- **확장자/정확한 URL 필요**: `./utils.js` (생략 불가).

### 모듈 스코프의 this / import.meta

```js
console.log(this);          // undefined (strict)
console.log(import.meta.url); // 현재 모듈 URL(절대경로)
```

### 캐시/단일 평가와 싱글톤 패턴

```js
// store.js
let id = 0;
export function nextId(){ return ++id; }

// a.js / b.js 각각에서 nextId() 호출 → 같은 카운터 공유(모듈은 한 번만 평가)
```

### Import Maps (경로 별칭)

```html
<script type="importmap">
{
  "imports": {
    "lib/": "/static/lib/",
    "lodash": "https://cdn.example.com/lodash-es.js"
  }
}
</script>
<script type="module">
  import { debounce } from 'lodash';
  import { foo } from 'lib/foo.js';
</script>
```
- **정식 표준**. 브라우저가 **경로 별칭/버전 고정**을 이해.

### JSON/CSS 등 Import(실험/환경 별)

- 일부 환경/번들러는 `import json from './data.json' assert { type: 'json' }` 를 지원.
- 브라우저 네이티브 지원 상황은 환경에 따라 다르므로 배포 타겟을 확인.

---

## Node.js에서의 모듈

### 두 체계 공존

| 체계 | 확장자 | 활성화 방법 | 불러오기 |
|---|---|---|---|
| **CommonJS** | `.cjs` / (기본 `.js`) | 기본(또는 `"type":"commonjs"`) | `require`, `module.exports` |
| **ESM** | `.mjs` / `.js` | `"type":"module"` | `import`, `export` |

#### package.json로 ESM 활성화

```json
{
  "type": "module"
}
```
- 이제 `.js`도 ESM로 파싱.

### Node ESM 예시

```js
// math.js (ESM)
export const add = (a,b)=>a+b;

// main.js (ESM)
import { add } from './math.js';
console.log(add(1,2));
```

### CommonJS ↔ ESM 상호운용

#### ESM에서 CommonJS 불러오기

```js
// ESM 파일
import { createRequire } from 'node:module';
const require = createRequire(import.meta.url);
const cjs = require('./legacy.cjs');
```

#### CommonJS에서 ESM 불러오기

```js
// CJS 파일
(async () => {
  const { add } = await import('./math.js');
  console.log(add(1,2));
})();
```

### 패키지의 내보내기 제어 (exports 필드)

```json
{
  "name": "pkg",
  "type": "module",
  "exports": {
    ".": {
      "import": "./dist/index.mjs",
      "require": "./dist/index.cjs"
    }
  }
}
```
- Node는 **조건부 내보내기**로 **ESM/CJS 소비자**를 구분해 적절한 파일을 제공합니다.

---

## 동적 import와 코드 스플리팅

### 동적 import 기본

```js
button.addEventListener('click', async () => {
  const { track } = await import('./analytics.js');
  track('clicked');
});
```
- **Promise** 반환, 런타임 조건에 따라 모듈을 로드.
- 번들러는 이를 **코드 스플리팅** 포인트로 사용.

### 매개변수화 경로(안전 패턴)

```js
const pages = { home: './pages/home.js', about: './pages/about.js' };
const mod = await import(pages[route]);
```
- 완전 동적인 문자열은 정적 분석이 어려워 **화이트리스트 매핑**을 권장.

---

## Top-Level Await(TLA)

### 기본

```js
// data.mjs
const res = await fetch('https://example.com/data.json');
export const data = await res.json();
```
- **모듈 전체가 비동기**가 되어, 이를 import하는 상위 모듈의 평가가 **해당 await 완료까지 대기**합니다.

### 주의

- **모듈 그래프에 비동기 경로가 전파**되어 초기 로딩이 지연될 수 있음.
- SSR/번들 환경에서 TLA 지원 여부를 확인.

---

## 순환 의존(cyclic dependency)과 초기화 순서

### 문제 사례

```js
// a.js
import { b } from './b.js';
export const a = 'A';
console.log('in a, b =', b);

// b.js
import { a } from './a.js';
export const b = 'B';
console.log('in b, a =', a);
```
- ESM은 순환을 지원하지만 **평가 타이밍**에 따라 `undefined` 관측 가능.

### 안전 패턴

- **함수로 디퍼(지연) 참조**:
```js
// a.js
import { getB } from './b.js';
export function useB(){ return `use ${getB()}`; }

// b.js
import { useB } from './a.js';
export function getB(){ return 'B'; }
console.log(useB()); // 사이클이더라도 사용 시점에 값이 준비
```

---

## 트리 셰이킹과 사이드이펙트

### 트리 셰이킹 전제

- **정적 import/export**와 **사이드이펙트 없는 모듈**이 전제.
- 번들러는 미사용 export를 제거하여 번들 크기를 줄입니다.

### package.json에서 힌트 제공

```json
{
  "sideEffects": false
}
```
- 모듈 최상위 부작용이 없음을 선언. CSS/폴리필 등 부작용 파일이 있다면 배열 예외 설정:
```json
{ "sideEffects": ["./polyfills.js", "*.css"] }
```

### 흔한 함정

- 모듈 최상위에서 **전역 등록/DOM 접근/콘솔/데이터 변경**은 부작용으로 간주 → 트리 셰이킹에 방해.

---

## 브라우저 네트워킹/보안 세부

### CORS/동일 출처

- 모듈 로드는 **CORS 정책**을 따릅니다. CDN/서브도메인 사용 시 **CORS 헤더** 설정 필요.

### MIME/요청

- 올바른 MIME(`text/javascript`)과 존재하는 URL, 확장자 명시가 필요.

### 서브리소스 무결성(SRI, 번들 시 주로)

```html
<script type="module" src="/main.js" integrity="sha384-..."></script>
```

---

## 모듈 범용 유틸리티

### import.meta

```js
console.log(import.meta.url);        // 현재 모듈의 절대 URL
// 일부 빌드도구는 import.meta.env.* 등을 주입(Vite 등)
```

### URL 기반 리소스 로딩

```js
const img = new URL('./assets/pic.png', import.meta.url);
document.querySelector('img').src = img.href;
```

---

## 실전 폴더 구조와 예시

### 구조

```
src/
  lib/
    math.js
    logger.js
  features/
    report/
      index.js
      view.js
  main.js
index.html
```

### 코드

```js
// src/lib/math.js
export const sum = (a,b)=>a+b;
export const mean = arr => arr.reduce((a,b)=>a+b,0)/arr.length;

// src/lib/logger.js
export default function log(...args){ console.log('[app]', ...args); }

// src/features/report/index.js
import log from '../../lib/logger.js';
import { mean } from '../../lib/math.js';
export function report(title, numbers){
  log(title, mean(numbers));
}

// src/main.js
import { report } from './features/report/index.js';
report('avg', [10,20,30]);
```

```html
<!-- index.html -->
<script type="module" src="/src/main.js"></script>
```

---

## 번들러(webpack/rollup/vite)와 ESM

### 번들링 이유

- **HTTP 요청 수 감소**, 레거시 브라우저 트랜스파일, **코드 스플리팅/캐싱/해시 버전**.

### 코드 스플리팅(동적 import)

```js
// routes.js
export async function loadPage(name){
  const mod = await import(`./pages/${name}.js`); // 빌드 도구가 청크 분할
  return mod.default();
}
```

### 트리 셰이킹 팁

- Named export 사용 선호, 최상위 **부작용 최소화**, `sideEffects:false` 설정.

---

## 자주 하는 실수/FAQ

### “import는 최상위에서만?”

- 정적 `import`/`export`는 **최상위에서만**. 런타임 조건은 `await import()` 사용.

### “브라우저에서 확장자 생략?”

- 네이티브 ESM은 **확장자 필수**(`.js`, `.mjs`). Node의 해석 규칙과 다를 수 있음.

### “Default vs Named 뭐가 좋나?”

- **트리 셰이킹/자동 리팩토링** 측면에서 **Named** 선호. 라이브러리 외부 API 명확성↑.
- 단일 주력 기능을 내보내는 파일은 **default**도 가독성 좋음.

### “모듈이 두 번 실행되나요?”

- 같은 URL은 **한 번만 평가**. 쿼리/해시가 다르면 다른 모듈로 간주.

### “순환 의존으로 undefined 나와요”

- **지연 참조(함수/게터) 도입**, 초기화 순서 정리, 의존 분리/합치기.

---

## 보너스: 모듈 테스트/배포 체크리스트

- [ ] 브라우저: `<script type="module">`, 올바른 경로/확장자
- [ ] CORS: CDN/서브도메인 자원은 **Access-Control-Allow-Origin**
- [ ] Node: `"type":"module"` 또는 `.mjs`/`.cjs` 명시
- [ ] 패키지: `exports`/`types`(TS) 필드 구성, `sideEffects` 선언
- [ ] 번들: 코드 스플리팅 포인트(동적 import) 설정
- [ ] 트리 셰이킹: 부작용 제거, Named export 선호
- [ ] 순환 의존: 구조 점검(ESLint import/no-cycle 등)
- [ ] 캐시 전략: 해시 파일명/Cache-Control

---

## 미니 퀴즈

```js
// Q1: 다음 중 문법 오류는?
// A)
import x from './a.js';
if (cond) { import { y } from './b.js'; }  // ?
// B)
export { foo as default, bar } from './m.js';
// C)
import * as ns from './n.js'; ns = {};     // ?
// D)
export default 123;

// Q2: 다음 코드의 출력?
// counter.js
export let c = 0;
export function inc(){ c += 1; }

// main.js
import { c, inc } from './counter.js';
console.log(c);
inc();
console.log(c);

// Q3: 브라우저 네이티브 ESM에서 허용되는가?
// import data from './data.json';

// Q4: 순환 의존 시 안전한 패턴은?
// (보기) A) 초기화 전에 값 사용  B) 함수로 지연 참조  C) default 덮어쓰기

// Q5: Node에서 ESM 파일이 CJS를 불러오려면?
// (보기) A) require() 바로 사용  B) createRequire 사용  C) import로 가능
```

**힌트**
- Q1: A(정적 import는 최상위에서만), C(네임스페이스 바인딩은 읽기 전용).
- Q2: 0, 1 (라이브 바인딩).
- Q3: 환경에 따라 다름. 표준적으로는 `import ... assert { type:'json' }`가 필요하며 지원 여부 확인.
- Q4: B.
- Q5: B(`createRequire(import.meta.url)`).

---

## 결론

- **ESM은 현대 JS의 표준 모듈 시스템**으로, 정적 구조·라이브 바인딩·단일 평가·트리 셰이킹 친화성을 제공합니다.
- 브라우저/Node의 **활성화 규칙과 URL·확장자·CORS**를 이해하면 배포 문제를 크게 줄일 수 있습니다.
- **동적 import/Top-Level Await**로 유연한 로딩 전략을 취하되 초기 로딩 지연에 유의하세요.
- 레거시 **CommonJS와의 상호운용**은 `createRequire`/동적 import로 해결하고, 패키지는 `exports`로 양 체계를 안전하게 지원합니다.
- 마지막으로, **순환 참조/사이드이펙트**를 관리하고 **트리 셰이킹**을 극대화하는 설계가 번들 크기와 성능을 좌우합니다.
