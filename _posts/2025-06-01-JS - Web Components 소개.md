---
layout: post
title: JavaScript - Web Components 소개
date: 2025-06-01 22:20:23 +0900
category: JavaScript
---
# Web Components 소개

## 0) 한눈에 보는 핵심 3총사

| 기술 | 무엇을 해결? | 핵심 포인트 |
|---|---|---|
| **Custom Elements** | “나만의 HTML 태그” 정의 | `class MyEl extends HTMLElement`, `customElements.define()` |
| **Shadow DOM** | DOM/스타일 **캡슐화** | `attachShadow({mode:'open'})`, `:host`, `::slotted`, CSS Shadow Parts |
| **HTML Template** | 정적 템플릿 재사용 | `<template>` + `.content.cloneNode(true)` |

> 표준만으로도 **컴포넌트화, 캡슐화, 재사용**이 가능하다. 프레임워크는 “선언적 렌더링/상태/생태계”를 더해주는 선택지다.

---

## 1) Custom Elements — 사용자 정의 태그의 뼈대

### 1-1. 가장 작은 예제

```js
class MyButton extends HTMLElement {
  connectedCallback() {
    this.innerHTML = `<button type="button">Click me</button>`;
  }
}
customElements.define('my-button', MyButton);
```

```html
<my-button></my-button>
```

**포인트**
- `connectedCallback()`: DOM에 삽입될 때 호출.
- 반드시 **하이픈(-)** 이 포함된 이름을 써야 네이티브 태그와 충돌을 피한다(예: `x-card`, `app-modal`).

### 1-2. 라이프사이클 콜백 정리

| 훅 | 언제 불리나 | 활용 |
|---|---|---|
| `connectedCallback` | 문서에 추가될 때 | 초기 렌더, 이벤트 바인딩 |
| `disconnectedCallback` | 문서에서 제거될 때 | 이벤트 정리, 타이머 해제 |
| `attributeChangedCallback(name, oldV, newV)` | **관찰된 속성** 변경 시 | 리렌더, 동기화 |
| `adoptedCallback` | 다른 Document로 옮겨질 때 | 포털/iframe 전송 대응 |

관찰하려는 속성은 `static get observedAttributes(){ return ['disabled','open'] }` 로 명시한다.

---

## 2) Shadow DOM — 스타일과 DOM을 안전하게 격리

```js
class MyCard extends HTMLElement {
  constructor() {
    super();
    this.attachShadow({ mode: 'open' }).innerHTML = `
      <style>
        :host { display: block; border: 1px solid #ddd; padding: 12px; border-radius: 8px; }
        .title { font-weight: 700; margin-bottom: 8px; }
        ::slotted(img) { max-width: 100%; border-radius: 8px; }
      </style>
      <div class="title"><slot name="title"></slot></div>
      <slot></slot>
    `;
  }
}
customElements.define('my-card', MyCard);
```

```html
<my-card>
  <span slot="title">Hello</span>
  <img src="cover.jpg" alt="cover" />
  <p>Shadow DOM은 외부 CSS로부터 보호된다.</p>
</my-card>
```

### 2-1. 스타일 키워드

- `:host` — 컴포넌트 **자신**을 선택  
- `:host([variant="primary"])` — **호스트의 속성** 기반 스타일  
- `::slotted(p)` — **슬롯으로 투입된** Light DOM 노드에 스타일 (단, **슬롯 루트의 직계 자식**만)  
- **CSS Shadow Parts**: 내부 노드에 `part="thumb"` 부여 → 바깥에서 `::part(thumb)`로 스타일 허용  
- **Custom Properties(변수)**: 테마 통일에 최적 — `--color-primary` 등

### 2-2. open vs closed
- `open`: `el.shadowRoot` 접근 가능(디버깅·테스트 편리)  
- `closed`: 외부 접근 불가(진정한 캡슐화 필요 시)  
> 실무에선 **open 권장**(디버깅/테스트/스토리북∙플레이라이트 연동이 쉬움)

---

## 3) Template — 값싼 DOM 복제

```html
<template id="x-tmpl">
  <style>.hello { color: seagreen; }</style>
  <div class="hello"><slot></slot></div>
</template>
```

```js
class XHello extends HTMLElement {
  constructor(){
    super();
    const root = this.attachShadow({mode:'open'});
    const tpl = document.getElementById('x-tmpl');
    root.appendChild(tpl.content.cloneNode(true));
  }
}
customElements.define('x-hello', XHello);
```

> 템플릿은 **파싱은 되지만 렌더되지 않음**. clone하여 Shadow DOM에 삽입하면 빠르고 가벼움.

---

## 4) 속성(attributes) vs 프로퍼티(properties) 동기화

- **속성(HTML 문자열)**: `<x-todo done="true">`  
- **프로퍼티(JS 값)**: `el.done = true`

베스트 프랙티스: **리플렉션(양방향 동기화)**

```js
class XToggle extends HTMLElement {
  static get observedAttributes(){ return ['on']; }

  get on(){ return this.hasAttribute('on'); }
  set on(v){
    if(v) this.setAttribute('on',''); else this.removeAttribute('on');
    this.#render();
  }

  attributeChangedCallback(){ this.#render(); }

  constructor(){
    super();
    this.attachShadow({mode:'open'});
    this.shadowRoot.innerHTML = `
      <button id="btn" type="button" part="button"></button>
      <style>
        :host([on]) { outline: 2px solid dodgerblue }
      </style>`;
  }

  connectedCallback(){
    this.shadowRoot.getElementById('btn').addEventListener('click', () => {
      this.on = !this.on; // 프로퍼티 변경 → 속성 반영
      this.dispatchEvent(new CustomEvent('toggle', { detail: { on: this.on }, bubbles: true, composed: true }));
    });
    this.#render();
  }

  #render(){
    const btn = this.shadowRoot.getElementById('btn');
    btn.textContent = this.on ? 'ON' : 'OFF';
    btn.setAttribute('aria-pressed', String(this.on));
  }
}
customElements.define('x-toggle', XToggle);
```

**포인트**
- 불리언 속성은 **존재 자체가 true**(`checked`, `disabled`) 패턴을 따른다.  
- 이벤트는 `CustomEvent` + `{bubbles:true, composed:true}`로 **상위 DOM**까지 전파(프레임워크 연동 시 필수).

---

## 5) 실전 1 — 접근성 갖춘 ⭐️별점 컴포넌트 (폼 연동 포함)

### 5-1. 요구사항
- 키보드/스크린리더 지원(Arrow/Enter/Space, `role="radiogroup"`).  
- **Form-Associated**: `<form>` submit 시 값 포함.  
- 테마 가능(Shadow Parts / CSS 변수).

```js
class XRating extends HTMLElement {
  static formAssociated = true; // 폼 연동 활성화
  static get observedAttributes(){ return ['value', 'max', 'readonly']; }

  #internals = this.attachInternals?.(); // ElementInternals
  #value = 0;

  get value(){ return this.#value; }
  set value(v){
    const int = Math.max(0, Math.min(this.max, Number(v)||0));
    this.#value = int;
    this.setAttribute('value', String(int));
    this.#internals?.setFormValue?.(String(int)); // form 데이터 동기화
    this.#render();
  }

  get max(){ return Number(this.getAttribute('max') ?? 5); }
  set max(v){ this.setAttribute('max', String(v)); }

  get readOnly(){ return this.hasAttribute('readonly'); }
  set readOnly(v){ v ? this.setAttribute('readonly','') : this.removeAttribute('readonly'); }

  constructor(){
    super();
    const root = this.attachShadow({mode:'open'});
    root.innerHTML = `
      <style>
        :host { --star-size: 24px; --star-color: #ccc; --star-active: gold; display: inline-block; }
        .wrap { display: inline-flex; gap: 4px; }
        button.star {
          all: unset; width: var(--star-size); height: var(--star-size); cursor: pointer;
          background: var(--star-color); clip-path: polygon(50% 0%, 61% 35%, 98% 35%, 68% 56%, 79% 91%, 50% 70%, 21% 91%, 32% 56%, 2% 35%, 39% 35%);
        }
        button.star.active { background: var(--star-active); }
        :host([readonly]) button.star { cursor: default; opacity: .6; }
      </style>
      <div class="wrap" role="radiogroup" aria-label="Rating"></div>
    `;
  }

  connectedCallback(){
    if(!this.hasAttribute('max')) this.max = 5;
    if(!this.hasAttribute('value')) this.value = 0;

    const wrap = this.shadowRoot.querySelector('.wrap');
    wrap.addEventListener('click', (e) => {
      const i = e.target?.dataset?.i;
      if(i && !this.readOnly){ this.value = Number(i); this.#fire(); }
    });

    wrap.addEventListener('keydown', (e) => {
      if(this.readOnly) return;
      if(['ArrowRight','ArrowUp'].includes(e.key)){ this.value = Math.min(this.value+1, this.max); this.#fire(); e.preventDefault(); }
      if(['ArrowLeft','ArrowDown'].includes(e.key)){ this.value = Math.max(this.value-1, 0); this.#fire(); e.preventDefault(); }
      if([' ','Enter'].includes(e.key)){ this.#fire(); e.preventDefault(); }
    });

    this.#render();
  }

  attributeChangedCallback(name, _o, _n){
    if(name==='value') this.#value = Number(this.getAttribute('value')||0);
    this.#render();
  }

  #render(){
    const wrap = this.shadowRoot.querySelector('.wrap');
    wrap.innerHTML = '';
    for(let i=1; i<=this.max; i++){
      const btn = document.createElement('button');
      btn.className = 'star' + (i<=this.value ? ' active' : '');
      btn.setAttribute('role','radio');
      btn.setAttribute('aria-checked', String(i===this.value));
      btn.setAttribute('tabindex', i===this.value ? '0' : '-1');
      btn.dataset.i = String(i);
      wrap.appendChild(btn);
    }
  }

  #fire(){
    this.dispatchEvent(new CustomEvent('change', { detail: { value: this.value }, bubbles:true, composed:true }));
  }

  // form 인터페이스
  formDisabledCallback(disabled){ disabled ? this.setAttribute('readonly','') : this.removeAttribute('readonly'); }
  formResetCallback(){ this.value = 0; }
  formStateRestoreCallback(state){ this.value = Number(state||0); }
}
customElements.define('x-rating', XRating);
```

```html
<form onsubmit="event.preventDefault(); alert(new FormData(this).get('score'))">
  <x-rating name="score" value="3" max="5"></x-rating>
  <button>Submit</button>
</form>
```

**핵심**
- `static formAssociated = true` + `attachInternals()`로 **네이티브 폼과 연동**.  
- 키보드/스크린리더 가능: `radiogroup/radio` 역할, `aria-checked`, `tabindex` 관리.  
- 외부 테마: `--star-*` 변수 or `::part()` 전략(내부 버튼에 `part="star"` 추가 가능).

---

## 6) 실전 2 — 접근성/포커스 트랩 포함 모달

```js
class XModal extends HTMLElement {
  static get observedAttributes(){ return ['open']; }

  get open(){ return this.hasAttribute('open'); }
  set open(v){ v ? this.setAttribute('open','') : this.removeAttribute('open'); }

  constructor(){
    super();
    this.attachShadow({mode:'open'}).innerHTML = `
      <style>
        :host { display: contents; }
        .backdrop {
          position: fixed; inset: 0; background: rgba(0,0,0,.4);
          display: none; align-items: center; justify-content: center;
        }
        :host([open]) .backdrop { display: flex; }
        .panel { background: #fff; color: #222; min-width: 320px; max-width: 90vw; border-radius: 12px; padding: 16px; }
      </style>
      <div class="backdrop" part="backdrop" role="dialog" aria-modal="true" aria-labelledby="x-title">
        <div class="panel" part="panel">
          <slot name="title" id="x-title"></slot>
          <slot></slot>
          <button id="close" part="close">Close</button>
        </div>
      </div>
    `;
  }

  connectedCallback(){
    const root = this.shadowRoot;
    root.getElementById('close').addEventListener('click', () => this.open = false);
    root.querySelector('.backdrop').addEventListener('click', (e) => {
      if(e.target === e.currentTarget) this.open = false; // 바깥 클릭 닫기
    });
    this.addEventListener('keydown', (e)=> {
      if(e.key === 'Escape') this.open = false;
    });
  }

  attributeChangedCallback(name){
    if(name==='open'){
      this.dispatchEvent(new CustomEvent(this.open?'open':'close', { bubbles:true, composed:true }));
      if(this.open) this.#trapFocus();
    }
  }

  #trapFocus(){
    const focusables = this.shadowRoot.querySelectorAll('button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])');
    focusables[0]?.focus();
  }
}
customElements.define('x-modal', XModal);
```

```html
<button onclick="document.querySelector('x-modal').open=true">Open</button>
<x-modal>
  <h2 slot="title">Title</h2>
  <p>내용</p>
</x-modal>
```

> `role="dialog"` + `aria-modal="true"` + **Escape/Backdrop/포커스 트랩** 포함. `::part(panel)` 로 외부 스타일 커스터마이즈 가능.

---

## 7) 이벤트 모델 & 프레임워크 연동

### 7-1. 이벤트 전파
- Shadow DOM은 기본적으로 **이벤트 리타게팅**을 한다(내부 노드 대신 호스트가 타깃처럼 보임).  
- 프레임워크로 올리는 이벤트는 **`bubbles: true` + `composed: true`** 로 만들어야 상위로 도달.

```js
this.dispatchEvent(new CustomEvent('change', {
  detail: { value: this.value },
  bubbles: true, composed: true
}));
```

### 7-2. React 연동 팁
- React는 **커스텀 이벤트를 Synthetic Event로 래핑하지 않는다**.  
- `<x-rating onChange={...}>` 로는 안 잡힐 수 있음 → **ref로 DOM** 잡고 `addEventListener('change', handler)` 사용.

```jsx
import { useEffect, useRef } from 'react';

export default function RatingWrapper(){
  const ref = useRef(null);
  useEffect(() => {
    const el = ref.current;
    const onChange = e => console.log('value:', e.detail.value);
    el.addEventListener('change', onChange);
    return () => el.removeEventListener('change', onChange);
  }, []);
  return <x-rating ref={ref} value="2" max="5" />;
}
```

### 7-3. 속성 전달(문자열↔JS 값)
- JSX는 **문자열 속성**만 HTML로 박는다. 복잡한 객체/함수는 **마운트 후 프로퍼티 할당** 필요.

```jsx
function Host(){
  const ref = useRef(null);
  useEffect(()=>{ ref.current.options = { step: 5 }; },[]);
  return <x-range ref={ref} min="0" max="100" />;
}
```

> Vue/Svelte는 비교적 자연스럽지만, **불리언/객체/이벤트**는 프레임워크별 바인딩 규칙을 확인하라.

---

## 8) 스타일 커스터마이징 전략

1) **CSS Custom Properties**(권장)  
   - 제공 측: `--color-primary`, `--radius` 노출  
   - 사용 측: 호스트에 변수 주입

```css
x-rating { --star-active: #ff7a00; }
```

2) **Shadow Parts**  
   - 내부 노드에 `part="thumb"` → 외부 `x-slider::part(thumb){ ... }`

3) **exportparts**  
   - 중첩된 컴포넌트의 part를 상위로 전달  
   - `<x-card exportparts="header,footer">`

4) **::slotted**  
   - Light DOM 슬롯 자식만. 후손까지는 불가(제약 주의).

---

## 9) 성능·아키텍처 베스트 프랙티스

- **렌더 배치**: 속성 여러 개 바뀔 때마다 렌더 X → **마이크로태스크 큐**로 모아서 1회 렌더

```js
#pending=false;
#schedule(){
  if(this.#pending) return;
  this.#pending = true;
  queueMicrotask(() => { this.#pending = false; this.#render(); });
}
```

- **큰 리스트**: 가상 스크롤(Lazy/IntersectionObserver), `content-visibility: auto;`  
- **Constructable Stylesheets**(크롬/에지/사파리): 스타일을 객체로 재사용

```js
const sheet = new CSSStyleSheet();
sheet.replaceSync(`:host{display:block}`);
this.shadowRoot.adoptedStyleSheets = [sheet];
```

- **이벤트 핸들러 정리**: `disconnectedCallback` 에서 removeEventListener  
- **메모리**: `closed` ShadowRoot는 디버깅 어려움 + 릭 원인 추적 난이도↑ → `open` 권장  
- **CSP**: Shadow DOM 내부 `<style>` 는 허용되지만, 인라인 스크립트는 CSP에 걸릴 수 있음(외부 모듈/ESM 권장)

---

## 10) 폼-연동(Form-Associated) 심화

- `static formAssociated = true` + `const internals = this.attachInternals()`  
- `internals.setFormValue(value, state?)` 로 제출 값 설정  
- 콜백: `formDisabledCallback`, `formResetCallback`, `formStateRestoreCallback`  
- 네이티브 유효성 검사: `internals.setValidity({ customError: true }, '메시지', inputLikeElem)`

```js
if(invalid) this.#internals.setValidity({ customError: true }, '값을 입력하세요');
else this.#internals.setValidity({});
```

---

## 11) 접근성(A11y) 체크리스트

- **문서 구조 역할/이름/상태**: `role`, `aria-*`, `aria-live`  
- **키보드 내비게이션**: Tab 순서, Arrow/Space/Enter 대응  
- **포커스 관리**: 모달/팝오버의 포커스 트랩 + 복귀  
- **라벨 연결**: `aria-labelledby` / `aria-label`  
- **컨트라스트/모션 감도**: 테마 변수, `prefers-reduced-motion` 대응  
- **스크린리더**: `aria-pressed`, `aria-expanded`, `aria-checked` 정확히 업데이트

---

## 12) 테스트/스토리/문서화

- **유닛/DOM 테스트**: `vitest` + `@testing-library/dom`  
- **E2E**: `Playwright` (Shadow DOM 셀렉터 지원: `shadow=`/강력한 Locator)  
- **스토리북**: Web Components 모드 지원  
- **비주얼 리그레션**: Chromatic/Playwright Screenshot

```js
// @testing-library/dom 예시
import { getByRole } from '@testing-library/dom';
test('x-toggle toggles', async () => {
  document.body.innerHTML = `<x-toggle></x-toggle>`;
  const el = document.querySelector('x-toggle');
  const btn = el.shadowRoot.querySelector('button');
  btn.click();
  expect(el.on).toBe(true);
});
```

---

## 13) 배포·번들·버전 전략

- **ESM 배포** + 타입 선언(d.ts) 제공(개발자 친화)  
- `package.json`의 `"exports"` 로 ESM 경로 지정  
- **이름 충돌 회피**: 프리픽스(예: `acme-`)  
- **Side Effects**: webpack tree-shaking 친화 설정  
- **폴리필 최소화**: 최신 브라우저 타깃 권장(IE 미지원)  
- **문서**: 속성/프로퍼티/이벤트/파트/슬롯/폼 연동 표 제공(디자인 시스템 수준)

---

## 14) 프레임워크와의 관계 설정

- **내부 구현은 Web Components**, 프레임워크 앱에서 **호출/조립**  
- 장점: 멀티 프레임워크/노프레임워크 재사용(디자인 시스템 중심)  
- 한계: 선언적 상태관리/SSR/HMR처럼 **앱 아키텍처**는 프레임워크가 편함  
- 타협: **복잡한 페이지는 프레임워크**, “버튼/토글/모달/셀렉트/달력” 같은 **UI 원자들은 Web Components**로 표준화

---

## 15) 고급 주제 스냅샷

- **ElementInternals의 `aria*`**: 호스트에 접근성 속성 반영  
- **Adopted Stylesheets**: 런타임 스타일 교체/테마 스위치 O(브라우저 지원 확인)  
- **Event Retargeting 주의**: 내부 노드 참조가 필요한 관측/로깅은 `open` 모드 사용  
- **서버 렌더링(SSR)**: 표준만으로는 하이드레이션 부재 → *프레임워크/툴(예: Lit SSR)* 분석 필요  
- **i18n**: `Intl.*`, Light DOM 슬롯에 번역 문자열 투입 or 속성으로 메시지 전달  
- **보안**: 슬롯 컨텐츠 Sanitization(필요 시), `sandboxed iframes` 고려

---

## 16) 미니 레시피 모음

### 16-1. IntersectionObserver로 지연 로딩 카드
```js
class XLazy extends HTMLElement {
  constructor(){ super(); this.attachShadow({mode:'open'}).innerHTML = `<slot></slot>`; }
  connectedCallback(){
    this.#io = new IntersectionObserver(([e]) => {
      if(e.isIntersecting){ this.dispatchEvent(new Event('enter',{bubbles:true, composed:true})); this.#io.disconnect(); }
    });
    this.#io.observe(this);
  }
  disconnectedCallback(){ this.#io?.disconnect(); }
}
customElements.define('x-lazy', XLazy);
```

### 16-2. 토스트(알림) 관리자
```js
class XToastHost extends HTMLElement {
  constructor(){ super(); this.attachShadow({mode:'open'}).innerHTML = `
    <style>
      :host { position: fixed; right: 12px; bottom: 12px; display: grid; gap: 8px; z-index: 9999; }
      .toast { background:#333;color:#fff;padding:8px 12px;border-radius:8px; }
    </style>
    <slot></slot>
  `;}
  show(msg, ms=2000){
    const div = document.createElement('div');
    div.className='toast'; div.textContent=msg;
    this.append(div);
    setTimeout(()=>div.remove(), ms);
  }
}
customElements.define('x-toast-host', XToastHost);
```

```html
<x-toast-host id="toasts"></x-toast-host>
<script>
  document.getElementById('toasts').show('Saved!');
</script>
```

---

## 17) 체크리스트(요약)

- [ ] 이름에 `-` 포함, 문서화(속성/프로퍼티/이벤트/파트/슬롯)  
- [ ] Shadow DOM `open`, 파트/변수 기반 테마 제공  
- [ ] 속성↔프로퍼티 리플렉션, 불리언 속성 패턴 준수  
- [ ] 이벤트 `bubbles+composed` 로 상향 전파  
- [ ] A11y: 역할/라벨/키보드/포커스/aria 업데이트  
- [ ] Form-Associated 적용 가능한 입력형 컴포넌트는 Internals 연동  
- [ ] 렌더 배치(마이크로태스크), 대량 DOM은 가상화/지연  
- [ ] 테스트(단위/스토리/E2E) & CI  
- [ ] 패키징(ESM), 버전/변경로그/마이그레이션 가이드

---

## 18) 결론

Web Components는 **브라우저 표준만으로** 캡슐화·재사용·테마 가능성을 제공합니다.  
**UI 원자/분자(Design System)** 층위에 특히 강력하며, 프레임워크와 **보완 관계**를 형성합니다.  
작게 시작해 **속성/이벤트/파트/폼/A11y** 를 갖춘 컴포넌트를 쌓아가면, 어느 스택에서도 통하는 **장기 자산**이 된다.

---

## 참고 문서
- MDN Web Components: https://developer.mozilla.org/en-US/docs/Web/Web_Components  
- Custom Elements: https://developer.mozilla.org/en-US/docs/Web/API/CustomElementRegistry/define  
- Shadow DOM: https://developer.mozilla.org/en-US/docs/Web/Web_Components/Using_shadow_DOM  
- HTML Templates: https://developer.mozilla.org/en-US/docs/Web/HTML/Element/template