---
layout: post
title: 웹해킹 - DOM Clobbering
date: 2025-10-05 19:25:23 +0900
category: 웹해킹
---
# 13. DOM Clobbering

## 0. 핵심 요약 (Executive Summary)

- **문제(정의)**
  HTML 문서에서 `id`/`name`을 가진 요소들이 **글로벌 네임스페이스**(특히 `window`/`document`/`form`)에 **프로퍼티처럼 노출**됩니다.
  개발자가 이를 몰라서 **전역 참조(예: `window.foo`, `document.bar`), 폼 속성/메서드(예: `form.submit`)**에 의존하면,
  공격자는 **같은 이름의 요소**를 주입해 *참조를 “덮어써”* 보안 체크를 우회/교란할 수 있습니다.

- **대표 증상**
  - `form.submit`이 **함수가 아니라 요소**로 바뀌어 정상 흐름이 깨짐
  - `if (isAdmin) …` 같은 **암묵 전역/글로벌 프로퍼티**가 **요소로 평가**되어 truthy/객체가 됨
  - `document.foo`/`window.bar` 식 접근이 **예상과 다른 타입**(HTML 요소/컬렉션)으로 변함

- **핵심 방어**
  1) **요소 접근은 항상 `document.getElementById` / `querySelector`**로 **명시적** 참조
  2) **모듈 범위 `const`(ES modules)** 로 **참조를 캡처**하여 전역 룩업 회피
  3) **폼/전역 예약어와 `id`/`name` 충돌 금지**(빌드/테스트에 포함)
  4) **템플릿/리치 텍스트 삽입 시 `id`/`name` 제거**(Sanitize/정규화)
  5) **`'use strict'`/ESLint**로 **암묵 전역**(implicit global) 생성 금지
  6) **타입 검증**: 사용 전 **타입/메서드 존재** 확인(가드)

---

# 1. DOM Clobbering이 왜 생기나 — 브라우저의 “이름 → 프로퍼티” 노출 규칙

브라우저는 호환성 때문에 다음을 제공합니다.

- **Window Named Properties**: 문서 내 `id`/`name`을 가진 요소들이 `window.<name>`로 **노출**될 수 있음.
- **Document Collections**: `document.forms.foo`, `document.images.logo` 등 **컬렉션에 이름 접근**.
- **Form Named Access**: `<form>` 내부 컨트롤의 `name`으로 `form.<name>` 접근.
  → 특히 **`form.submit` 메서드가 `<input name="submit">`에 덮이기** 쉬움.

> 표준/브라우저별 세부 차이는 있지만, **“이름이 전역/폼의 프로퍼티 자리에 나타난다”**는 공통 행동이 본 취약점의 뿌리입니다.

---

# 2. 위험 시나리오 & 안전한 재현

## 2.1 폼 메서드 덮기: `form.submit` 우회/교란

**취약 HTML**
```html
<form id="pay" action="/pay" method="post">
  <!-- 개발자는 'form.submit()'을 호출한다고 가정 -->
  <!-- 공격자가 주입한 필드 (예: 댓글/프로필 HTML 인젝션 지점) -->
  <input name="submit" value="clobbered">
</form>
<script>
  const form = document.getElementById('pay');
  // ❌ 의도: 네이티브 submit 호출 → 실제: 'submit'이 요소여서 함수가 아님
  form.submit(); // TypeError: form.submit is not a function (또는 침묵 실패)
</script>
```

**영향**
- CSRF 방지 등 **검증 흐름이 깨져** 전송이 되지 않거나,
  보안 로직이 “submit 호출”을 신뢰하는 경우 **우회를 유발**.

**수정(방어)**
```html
<form id="pay" action="/pay" method="post">
  <!-- 'name="submit"' 금지 (린트/테스트로 막기) -->
  <button type="submit" id="btnPay">Pay</button>
</form>
<script type="module">
  'use strict';
  const form = document.getElementById('pay');        // 명시적 참조
  HTMLFormElement.prototype.submit.call(form);        // 네이티브 메서드 직접 호출
</script>
```

---

## 2.2 전역 변수/플래그 덮기: auth 플래그 우회

**취약 코드**
```html
<!-- 개발자 가정: SSR이 window.isAdmin = false 를 세팅 -->
<!-- 실제로는 누락되거나, 코드가 전역(window.isAdmin)을 직접 참조 -->
<script>
  // ❌ 전역 프로퍼티 룩업(안티패턴)
  if (window.isAdmin) {
    enableDangerousButton();
  }
</script>

<!-- 공격자 HTML 주입 -->
<div id="isAdmin"></div> <!-- truthy 객체로 전역 프로퍼티 생성/섀도잉 -->
```

**결과**
`window.isAdmin`이 **HTMLElement**로 평가 → truthy → **보안 체크 우회**.

**수정(방어)**
```html
<script type="module">
  'use strict';

  // 1) 모듈 스코프 상수로 '권한' 캡처(전역 프로퍼티 의존 X)
  const IS_ADMIN = Boolean(window.__BOOTSTRAP__?.user?.isAdmin ?? false);

  // 2) 사용 전 타입 확인(보너스)
  if (typeof IS_ADMIN === 'boolean' && IS_ADMIN) {
    enableDangerousButton();
  }
</script>
```
> 모듈 스코프 `const`는 **전역 프로퍼티가 아니며** DOM 네임드 요소가 이를 덮을 수 없습니다.

---

## 2.3 폼 속성 혼동: `form.action`/`form.target` 조작

**취약 코드**
```html
<form id="f" action="/transfer" method="post" target="_self">
  <!-- 공격자 주입 -->
  <input name="target" value="_blank">    <!-- form.target 을 요소로 클로버 -->
</form>
<script>
  const f = document.getElementById('f');
  // ❌ 보안 점검: target이 _self인지 확인
  if (f.target !== '_self') throw new Error('no'); // 여기서 f.target은 HTMLInputElement
  // 또는 어딘가에서 f.target 사용 시 다른 타입으로 오동작
</script>
```

**수정(방어)**
```html
<script type="module">
  const f = document.getElementById('f');

  // 1) form.getAttribute로 문자열을 직접 읽거나
  if (f.getAttribute('target') !== '_self') throw new Error('no');

  // 2) 또는 안전한 스냅샷
  const TARGET = '_self';  // 정책 상수: 모듈 스코프
  if (f.getAttribute('target') !== TARGET) throw new Error('no');
</script>
```

---

## 2.4 `document.foo`/`window.bar`로 요소 접근(레거시 관용) → 조작 취약

**취약 코드(레거시)**
```html
<div id="panel">...</div>
<script>
  // ❌ 오래된 관용: document.panel 으로 접근 (네임드 엘리먼트)
  document.panel.textContent = 'hi'; // 공격자가 <form id="panel">… 같은 걸 주입하면?
</script>
```

**수정(방어)**
```html
<script type="module">
  const panel = document.getElementById('panel'); // ✅ 항상 명시적 조회
  if (!(panel instanceof HTMLElement)) throw new Error('panel missing');
  panel.textContent = 'hi';
</script>
```

---

# 3. 방어 전략 — 코드/템플릿/도구 레벨

## 3.1 요소 참조 원칙 (필수)
- **반드시** `document.getElementById`, `querySelector`(또는 프레임워크 `ref`)로 **로컬 변수**에 참조를 저장하고 사용.
- 전역/폼의 **이름 기반 프로퍼티**(`window.foo`, `document.bar`, `form.baz`)에 **의존 금지**.

**패턴**
```js
// ❌ 나쁜 예
if (window.loginForm && window.loginForm.submit) window.loginForm.submit();

// ✅ 좋은 예
const loginForm = document.getElementById('login-form');
HTMLFormElement.prototype.submit.call(loginForm);
```

## 3.2 모듈 범위 `const`로 **참조 스냅샷**
- ES Modules를 사용하면 **전역 오염**/DOM 네임드 충돌에서 **자연스럽게 격리**됩니다.
```html
<script type="module">
  const el = /** @type {HTMLButtonElement} */ (document.getElementById('danger'));
  const submit = HTMLFormElement.prototype.submit; // 네이티브 메서드 스냅샷
  // … 나중에 submit.call(form)
</script>
```

## 3.3 폼/전역 **예약어 블랙리스트**
실제 프로젝트에서 **아래 이름은 `id`/`name`으로 금지**하세요(일부만 예시).

- **폼 메서드/속성**: `submit`, `reset`, `action`, `target`, `method`, `length`, `elements`
- **Window/Document 흔한 전역**: `name`, `status`, `event`, `history`, `open`, `close`, `location`, `frames`, `length`
- **프레임워크에서 쓰는 전역 키**: 앱마다 정리(예: `__BOOTSTRAP__`, `__DATA__` 등)

**검사 스크립트(빌드 단계)**
```js
// tools/check-id-name-collisions.mjs
import { readFileSync } from 'node:fs';
const RESERVED = new Set([
  'submit','reset','action','target','method','length','elements',
  'name','status','event','history','open','close','location','frames','length'
]);
const html = readFileSync(process.argv[2], 'utf8');
const idNames = [...html.matchAll(/\s(id|name)=["']([^"']+)["']/g)].map(m=>m[2]);
const bad = idNames.filter(n => RESERVED.has(n));
if (bad.length) {
  console.error('❌ Reserved id/name detected:', bad.join(', '));
  process.exit(1);
}
console.log('✅ No reserved id/name');
```

## 3.4 템플릿/리치 텍스트 **Sanitization**
- **외부/사용자 입력 HTML**에서는 **`id`/`name`/`form`** 같은 **네임스페이스 영향 속성 제거**가 안전합니다.
- HTML Sanitizer(DOMPurify 등) 설정에서 `FORBID_ATTR: ['id','name','form']` 등.

**예: DOMPurify**
```js
const clean = DOMPurify.sanitize(userHtml, {
  FORBID_ATTR: ['id','name','form'],
  ALLOWED_URI_REGEXP: /^https?:/i
});
container.innerHTML = clean;
```

## 3.5 타입/존재 **가드**
- 사용 전 **타입과 메서드 존재**를 확인하세요.

```js
const form = document.getElementById('pay');
if (!(form instanceof HTMLFormElement)) throw new Error('form missing');
// 'submit'이 진짜 함수인지 확인
const submit = form.submit;
if (typeof submit !== 'function') throw new Error('submit clobbered');
submit.call(form);
```

## 3.6 린트/컴파일러 설정
- **ESLint**:
  - `no-undef`, `no-implicit-globals`, `no-restricted-globals`(예: `event`, `name`)
  - `no-restricted-properties`로 `document.foo`/`window.bar` 사용 금지
- **TypeScript**:
  - `noImplicitAny`, `noImplicitThis`, `strictNullChecks`, `dom` 라이브러리 활성
  - 전역 타입에 의존하지 않도록 **모듈/ESNext 타깃** 사용

**ESLint 샘플**
```js
// .eslintrc.cjs
module.exports = {
  env: { browser: true, es2023: true },
  rules: {
    'no-undef': 'error',
    'no-implicit-globals': 'error',
    'no-restricted-globals': ['error', 'event', 'name', 'status'],
    'no-restricted-properties': ['error',
      { object: 'document', property: /.*/, message: 'Use getElementById/querySelector' },
      { object: 'window', property: /.*/, message: 'Avoid window.* lookups' }
    ],
  }
};
```

---

# 4. 프레임워크별 팁

## 4.1 React/Vue/Svelte 등
- 요소는 **ref**로 접근하고, **전역 네임드 접근 금지**.
- 리치 텍스트 렌더링: `dangerouslySetInnerHTML`/`v-html` 사용 시 **Sanitize + id/name 제거**.

**React 예**
```tsx
function PayForm() {
  const ref = useRef<HTMLFormElement>(null);
  const onClick = () => {
    const f = ref.current!;
    HTMLFormElement.prototype.submit.call(f);
  };
  return (
    <form id="pay" ref={ref} action="/pay" method="post">
      <button type="button" onClick={onClick}>Pay</button>
    </form>
  );
}
```

## 4.2 서버 템플릿(Thymeleaf, EJS, Pug 등)
- **예약어 리스트**를 템플릿 빌드 파이프라인에 포함.
- 동적 블록에 사용자 HTML 삽입 금지(필요 시 Sanitize).
- **`id` 생성기**로 **충돌 방지 프리픽스** 도입(예: `app-<slug>-<hash>`).

---

# 5. 보안 헤더/정책(보조 완화)

- **CSP**: 인젝션 경로 축소(`script-src 'self'` …). DOM Clobbering 자체를 직접 막지는 못하지만 **HTML 인젝션**을 어렵게 합니다.
- **X-Frame-Options / frame-ancestors**: 클릭재킹 경감(클로버링과 함께 쓰이는 체인 공격 방지).
- **Trusted Types**: 위험 sink(`innerHTML` 등)에 **팩토리 강제** → 임의 HTML 삽입 억제.

---

# 6. 자동 테스트/관측

## 6.1 런타임 자가 점검 스니펫(디버그/스테이징)
```js
// 페이지 로드 후 한 번: 네임드 충돌 스캔
(() => {
  const reserved = new Set(['submit','reset','action','target','method','length','elements',
                            'name','status','event','history','open','close','location','frames','length']);
  const seen = new Set();
  const bad = [];
  document.querySelectorAll('[id],[name]').forEach(el => {
    const id = el.getAttribute('id'); const nm = el.getAttribute('name');
    for (const v of [id, nm]) {
      if (!v) continue;
      if (reserved.has(v)) bad.push({ v, el });
      if (seen.has(v)) bad.push({ v, el, dup: true }); else seen.add(v);
      // 전역/폼 프로퍼티로 노출되었는지 간단 확인
      if (v in window && window[v] === el) bad.push({ v, el, clobbered: 'window' });
    }
  });
  if (bad.length) console.warn('DOM clobbering risks:', bad);
})();
```

## 6.2 E2E(Playwright/Cypress) — “막혀야 정상”
- **시나리오**: 사용자 콘텐츠 영역에 `<input name="submit">` 삽입 후 결제 버튼 클릭 →
  **네이티브 submit 경로**가 여전히 성공해야 테스트 통과.

**Playwright 예시**
```ts
await page.setContent(`
  <form id="f" action="/ok" method="post">
    <div id="user-content"><input name="submit"></div>
    <button id="btn" type="button">Go</button>
  </form>
  <script>
    document.getElementById('btn').onclick = () => {
      const f = document.getElementById('f');
      HTMLFormElement.prototype.submit.call(f);
    };
  </script>
`);
await page.click('#btn');
// 서버 목으로 POST 수신 확인 …
```

---

# 7. 안티패턴 요약 (피해야 할 것)

- `document.foo` / `window.bar` 로 **요소/데이터 접근**
- **암묵 전역**(선언 없이 식별자 사용) 및 전역 프로퍼티 의존
- 폼 요소에 `name="submit"`, `name="action"` 등 **예약어 사용**
- 사용자/서드파티 HTML을 **Sanitize 없이** 삽입
- 전역 부트스트랩 데이터(예: `window.user`)를 **HTML과 같은 이름**으로 둠

---

# 8. 체크리스트 (현장용)

- [ ] 요소 접근은 **항상 `getElementById`/`querySelector` + 로컬 변수**
- [ ] **ES Modules**로 모듈 스코프 `const` 사용(전역 의존 X)
- [ ] 폼/전역 **예약어 리스트**를 린트·빌드에서 **금지**
- [ ] 리치 텍스트/외부 HTML **Sanitize**(`id`/`name` 제거)
- [ ] **네이티브 메서드 스냅샷** 후 호출(예: `HTMLFormElement.prototype.submit.call(form)`)
- [ ] ESLint/TS의 **암묵 전역 금지** 규칙 적용
- [ ] 스테이징에서 **클로버링 스캔 스크립트** 실행
- [ ] 보안 리뷰에서 **폼/전역 이름 충돌** 항목 포함

---

## 맺음말

DOM Clobbering은 **브라우저의 유서 깊은 “이름→프로퍼티” 편의 기능**이
현대 애플리케이션의 **전역/폼 의존 패턴**과 만나 생기는 문제입니다.
**명시적 요소 조회 → 모듈 스코프 상수 캡처 → 예약어 충돌 차단 → Sanitization**의
4단계만 지켜도 대부분의 클로버링 리스크를 **구조적으로 제거**할 수 있습니다.
