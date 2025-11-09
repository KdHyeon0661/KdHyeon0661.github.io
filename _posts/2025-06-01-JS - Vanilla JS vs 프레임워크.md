---
layout: post
title: JavaScript - Vanilla JS vs 프레임워크
date: 2025-06-01 19:20:23 +0900
category: JavaScript
---
# Vanilla JS vs 프레임워크: 언제, 왜, 무엇을 선택할까?

## 0. 판단 가이드(요약)

- **작고 단순**: Vanilla JS → 빠른 납기, 의존성 0
- **중~대형/기능 다각화**: 프레임워크 → 구조·재사용·협업·생태계
- **SEO/SSR/라우팅/상태 난도↑**: 프레임워크(Next/Nuxt 등 메타 프레임워크)
- **성능 최상·제어권 100%**: Vanilla + 선택적 유틸(라우터/스토어 등 소형 라이브러리)
- **팀 표준·온보딩 속도**: 프레임워크(관례와 도구가 답)

---

## 1️⃣ Vanilla JS란?

> **프레임워크/라이브러리 없이** 순수 자바스크립트 + 브라우저 표준 API만으로 구현하는 방식.

### 장점
- **의존성 0**: 번들 크기 최소, 보안/라이선스 리스크↓
- **러닝 커브 낮음**: JS/DOM만 알면 시작 가능
- **미세 제어**: 성능·접근성·렌더링 타이밍까지 직접 조절 가능

### 단점
- **구조/패턴 부재**: 규모↑ → 이벤트/상태/DOM 업데이트 난맥상
- **재사용/조직화 비용↑**: 컴포넌트·라우팅·i18n 등 직접 설계
- **테스트/빌드/SSR/코드스플릿**: 수작업 구성 필요

### 가장 단순한 예시
```html
<p id="text">Hello</p>
<button onclick="changeText()">변경</button>
<script>
  function changeText() {
    document.getElementById("text").innerText = "Hi";
  }
</script>
```

---

## 2️⃣ 프레임워크란?

> UI를 **컴포넌트 단위**로 쪼개고, **상태·라우팅·빌드·테스트·SSR**까지 생태계를 제공하는 개발 틀.

### 대표 프레임워크
| 이름 | 특성 |
|---|---|
| React | 선언적 UI, Hooks, 거대 생태계(Next.js, React Query 등) |
| Vue | 직관적 문법, SFC, 반응성 시스템(Composition API) |
| Angular | TS 중심 정통 프레임워크(라우팅/DI/폼/빌드 일체) |
| Svelte | 컴파일 타임 최적화, 런타임 오버헤드↓ |

### React 스타일 예시
```jsx
import { useState } from 'react';
export default function App() {
  const [text, setText] = useState("Hello");
  return (
    <>
      <p>{text}</p>
      <button onClick={() => setText("Hi")}>변경</button>
    </>
  );
}
```

---

## 3️⃣ 핵심 비교 (한눈표)

| 항목 | Vanilla JS | 프레임워크 |
|---|---|---|
| 개념 | 순수 JS/DOM | UI 라이브러리/프레임워크 |
| 학습 곡선 | 낮음 | 중~높음(컴포넌트/상태/라우팅 등) |
| 초기 생산성 | 작고 단순한 앱에 유리 | 템플릿/CLI/HMR로 빠른 반복 |
| 규모 확장 | 구조 설계 필요 | 관례/패턴 내장 → 협업 유리 |
| 성능 | 최적화 자유도 100% | 런타임 오버헤드 있으나 최적화 도구 풍부 |
| 생태계 | 직접 선택/조합 | 표준화된 플러그인/툴/문서 |
| SSR/SEO | 수작업/별도 세팅 | Next/Nuxt/SvelteKit 등으로 수월 |
| 테스트/빌드 | 직접 구성 | CRA/Vite/Angular CLI 등 내장 혹은 공식 가이드 |

---

## 4️⃣ 상황별 선택 기준

| 상황 | 추천 |
|---|---|
| **사이드프로젝트/PoC/학습** | Vanilla JS |
| **단일 페이지·단순 DOM 조작** | Vanilla JS |
| **SPA, 라우팅/상태 다수** | React/Vue/Svelte |
| **대규모 팀·정책/가드레일 필요** | React/Angular |
| **SEO/SSR/ISR** | Next.js/Nuxt/SvelteKit |
| **초소형·저사양·오프라인 Kiosk** | Vanilla + 소형 유틸(알파) |

---

## 5️⃣ 동일 기능, 다른 접근 — 실전 비교

### A. 양방향 입력 미러링

**Vanilla**
```html
<input id="input" />
<p id="result"></p>
<script>
  const $in = document.getElementById("input");
  const $out = document.getElementById("result");
  $in.addEventListener("input", e => { $out.textContent = e.target.value; });
</script>
```

**React**
```jsx
import { useState } from 'react';
export default function Mirror() {
  const [text, setText] = useState("");
  return (
    <>
      <input onChange={(e) => setText(e.target.value)} />
      <p>{text}</p>
    </>
  );
}
```

**논의**  
- Vanilla: 직관적이지만 **상태가 흩어짐**(여러 요소·페이지로 확장 시 복잡).  
- React: 상태→UI 단방향 흐름으로 **예측가능성↑**, 테스트/리팩터링 유리.

---

### B. Todo(추가·토글·삭제·영속화) — 구조 차이 체감

**Vanilla(컴포넌트 흉내 + 이벤트 위임 + Store)**
```html
<ul id="list"></ul>
<form id="form"><input id="text" required/><button>추가</button></form>
<script>
  // 간단 Store
  const store = (() => {
    let todos = JSON.parse(localStorage.getItem('todos')||'[]');
    const subs = new Set();
    const notify = () => subs.forEach(f => f(todos));
    const set = (next) => { todos = next; localStorage.setItem('todos', JSON.stringify(todos)); notify(); }
    return {
      subscribe(f){ subs.add(f); f(todos); return () => subs.delete(f); },
      add(text){ set([...todos, {id:crypto.randomUUID(), text, done:false}]); },
      toggle(id){ set(todos.map(t => t.id===id? {...t,done:!t.done}:t)); },
      remove(id){ set(todos.filter(t => t.id!==id)); }
    };
  })();

  // 렌더 함수
  const $list = document.getElementById('list');
  store.subscribe((todos) => {
    $list.innerHTML = todos.map(t => `
      <li data-id="${t.id}">
        <label><input type="checkbox" ${t.done?'checked':''}/> ${t.text}</label>
        <button class="del">삭제</button>
      </li>`).join('');
  });

  // 이벤트 위임
  $list.addEventListener('change', e => {
    if(e.target.matches('input[type="checkbox"]')){
      const id = e.target.closest('li').dataset.id;
      store.toggle(id);
    }
  });
  $list.addEventListener('click', e => {
    if(e.target.matches('.del')){
      const id = e.target.closest('li').dataset.id;
      store.remove(id);
    }
  });

  // 추가
  document.getElementById('form').addEventListener('submit', e => {
    e.preventDefault();
    const text = document.getElementById('text').value.trim();
    if(text) store.add(text);
    e.target.reset();
  });
</script>
```

**React(상태 상향/컨텍스트/컴포넌트화)**
```jsx
import { createContext, useContext, useMemo, useState } from "react";

const TodoCtx = createContext(null);

function TodoProvider({ children }) {
  const [todos, setTodos] = useState(() => JSON.parse(localStorage.getItem('todos')||'[]'));
  const api = useMemo(() => ({
    todos,
    add: (text)=> setTodos(prev => {
      const next = [...prev, {id:crypto.randomUUID(), text, done:false}];
      localStorage.setItem('todos', JSON.stringify(next)); return next;
    }),
    toggle:(id)=> setTodos(prev => {
      const next = prev.map(t => t.id===id? {...t,done:!t.done}:t);
      localStorage.setItem('todos', JSON.stringify(next)); return next;
    }),
    remove:(id)=> setTodos(prev => {
      const next = prev.filter(t=>t.id!==id);
      localStorage.setItem('todos', JSON.stringify(next)); return next;
    })
  }), [todos]);
  return <TodoCtx.Provider value={api}>{children}</TodoCtx.Provider>;
}

function useTodos(){ return useContext(TodoCtx); }

function List(){
  const { todos, toggle, remove } = useTodos();
  return (
    <ul>
      {todos.map(t =>
        <li key={t.id}>
          <label>
            <input type="checkbox" checked={t.done} onChange={()=>toggle(t.id)} />
            {t.text}
          </label>
          <button onClick={()=>remove(t.id)}>삭제</button>
        </li>
      )}
    </ul>
  );
}

function AddForm(){
  const { add } = useTodos();
  const onSubmit = e => {
    e.preventDefault();
    const text = new FormData(e.currentTarget).get('text').trim();
    if(text) add(text);
    e.currentTarget.reset();
  };
  return (
    <form onSubmit={onSubmit}>
      <input name="text" required />
      <button>추가</button>
    </form>
  );
}

export default function App(){
  return (
    <TodoProvider>
      <AddForm />
      <List />
    </TodoProvider>
  );
}
```

**논의**  
- Vanilla: 가볍고 빠르지만 **렌더/상태/이벤트 연결부**를 개발자가 직접 체계화해야 한다.  
- React: 초기 보일러플레이트는 많지만 **역할 분리·테스트·확장(검색/필터/서버동기화)**이 명확.

---

## 6️⃣ 성능 관점의 의사결정

| 측면 | Vanilla JS | 프레임워크 |
|---|---|---|
| **초기 로드** | 가장 작음(필요 스크립트만) | 프레임워크 런타임 + 앱 번들 |
| **인터랙션** | 직접 최적화 가능(Idle/RAF/가상화) | 내장 최적화/메모라이즈, 다만 오버헤드 존재 |
| **메모리** | 적음 | 상대적으로 큼(가비지/힙 압력 관리 필요) |
| **코드 스플리팅** | 수작업(동적 import) | 라우트 단위 분할/빌드 체인 지원 |

**팁**
- **Vanilla**: 큰 리스트는 *가상 스크롤*, DOM 배치를 묶어 업데이트, `requestAnimationFrame`/`Idle` 활용.  
- **프레임워크**: 메모화(`useMemo`/`computed`), 리스트 *가상화* 컴포넌트, **SSR/ISR/Streaming**로 체감속도↑.

---

## 7️⃣ 설계/협업/유지보수

| 항목 | Vanilla | 프레임워크 |
|---|---|---|
| 구조/관례 | 직접 설계(문서화 필수) | 관례 확립(파일/상태/테스트 패턴) |
| 테스트 | Vitest/Jest+DOM Testing Library 직접 구성 | 대부분 가이드/예제 풍부 |
| 접근성(A11y) | 표준 준수·aria 수작업 | A11y 린터/컴포넌트 가드레일 활용 가능 |
| 국제화(i18n) | 메시지 테이블/서식기 수동 | 플러그인/패키지 다수 |
| 빌드/배포 | Vite/Webpack/ESBuild를 직접 엮기 | 템플릿/CLI로 일관성 유지 |

---

## 8️⃣ Vanilla의 “컴포넌트화” 패턴(프레임워크 없이도 구조 잡기)

### (1) 템플릿 렌더 + 이벤트 위임
```js
function h(strings, ...vals){ // 템플릿 헬퍼(옵션)
  return strings.reduce((s, str, i) => s + str + (vals[i]??''), '');
}

function mount($root, items){
  $root.innerHTML = h`
    <ul>
      ${items.map((x,i)=> h`<li data-i="${i}">${x}</li>`).join('')}
    </ul>`;
  $root.addEventListener('click', (e) => {
    const li = e.target.closest('li'); if(!li) return;
    console.log('clicked index:', li.dataset.i);
  });
}
```

### (2) Proxy/Signals로 반응성 흉내
```js
function signal(value){
  const subs = new Set();
  const get = () => value;
  const set = v => { value = v; subs.forEach(f=>f(value)); };
  const subscribe = f => { subs.add(f); f(value); return ()=>subs.delete(f); };
  return { get, set, subscribe };
}

const count = signal(0);
count.subscribe(v => document.getElementById('out').textContent = v);
document.getElementById('inc').addEventListener('click', () => count.set(count.get()+1));
```

### (3) 경량 라우팅(History API)
```js
const routes = {
  '/': () => 'Home',
  '/about': () => 'About',
};
function render(){
  const view = routes[location.pathname] || (()=>'Not Found');
  document.getElementById('app').textContent = view();
}
window.addEventListener('popstate', render);
document.addEventListener('click', e => {
  const a = e.target.closest('a[data-link]'); if(!a) return;
  e.preventDefault(); history.pushState(null, '', a.href); render();
});
render();
```

> 이런 식으로 **Vanilla도 컴포넌트/라우팅/스토어**를 조합해 “작은 프레임워크”처럼 운용 가능. 다만 **규모가 커지면** 검증된 프레임워크 전환이 유지보수에 유리.

---

## 9️⃣ 프레임워크 선택 가이드(세부)

- **React**: 라이브러리 조합 자유, 거대 생태계. 팀 내 프론트 규칙/문서화가 중요.
- **Vue**: 온보딩 빠름. SFC와 반응성으로 직관. Nuxt로 SSR/파일 라우팅까지 매끄럽게.
- **Angular**: 일체형 프레임워크(라우팅/DI/폼/테스트). 대규모/정책·규격 필요한 조직.
- **Svelte**: 런타임 오버헤드 적고 코드 양↓. SvelteKit로 라우팅/서버 통합.

**메타 프레임워크**  
- **Next.js/Nuxt/SvelteKit**: 파일 기반 라우팅, 서버 액션/SSR/ISR/Streaming, 이미지 최적화 등 **제품화 기능** 제공.

---

## 🔟 점진적 전환 전략(현실적인 마이그레이션)

1) **Vanilla로 시작**  
- HTML 청사진 + 접근성 준수 + 빌드(ESM/Vite)만 준비.

2) **컴포넌트화/스토어화**  
- 템플릿 함수/이벤트 위임/Store/라우팅으로 질서 부여.

3) **병목 확인**  
- 기능·협업 한계 도달: 설계 문서화, 테스트 도입, 번들 나눔.

4) **부분 채택**  
- **마이크로 프런트엔드** 또는 **위젯 단위**로 React/Vue 도입.

5) **완전 전환**  
- 라우팅/상태/빌드/테스트/SSR를 프레임워크 기준으로 재배치.  
- 데이터 페칭 규칙(SWR/React Query/Vue Query) 표준화.

---

## 11️⃣ 성능/품질 체크리스트

- **초기 로드**: 코드분할(dynamic import), 이미지 지연, critical CSS
- **상호작용**: 긴 작업은 Web Worker, 리스트 가상화, RAF 배치
- **상태 관리**: 불변 업데이트(또는 신중한 가변), 메모화 전략
- **접근성**: 시맨틱 태그, aria, 포커스 트랩, 키보드 내비게이션
- **테스트**: 유닛(Jest/Vitest), 컴포넌트 테스트(Testing Library), e2e(Playwright/Cypress)
- **관측성**: Web Vitals(LCP/CLS/INP), Sentry/LogRocket, 사용자 로그 설계

---

## 12️⃣ 결정 트리(문장형)

- 요구사항이 **폼 몇 개 + 단순 목록/상세**인가? → **Vanilla**  
- 라우팅/상태/권한/역할별 화면? **React/Vue**  
- **검색/필터/정렬** + **무한스크롤/캐싱**? → 프레임워크 + 데이터 라이브러리  
- **SEO/퍼포먼스** 민감? → **Next/Nuxt/SvelteKit**  
- **2명 이하**/단기? → Vanilla 또는 Svelte/Vue(온보딩 빠름)  
- **10명 이상**/장기? → React/Angular(+규범/가드레일)

---

## 13️⃣ 자주 하는 실수 & 예방책

| 실수 | 예방책 |
|---|---|
| Vanilla로 **규모 확대**하며 암묵 규칙만 쌓음 | 문서화/코드리뷰/테스트/모듈 경계 설정 |
| 프레임워크 도입 후 **패턴 남용**(전역 상태 과다 등) | 우선 로컬 상태, 필요시 스토어. 데이터 페칭 규칙 합의 |
| **렌더-성능** 미관심 | 리스트 가상화, 메모화, key 안정성, 프로파일링 |
| **접근성/국제화** 미반영 | 설계 단계에 A11y/i18n 포함, 검사 도구(Lighthouse/axe) |
| 빌드/배포 **일관성 부족** | CI 파이프라인 + 린트/테스트/타입체커 강제 |

---

## 14️⃣ 결론

- **Vanilla**: 빠르고 가볍다. 그러나 **질서**를 개발자가 만든다.  
- **프레임워크**: 보일러플레이트↑, 하지만 **협업/확장/테스트/제품화**에서 강력.  
- **정답은 맥락**: 팀 규모, 유지 기간, 기능 복잡도, 성능/SEO 목표, 배포 인프라…  
→ “**작게 시작해서, 신호가 오면 구조를 끌어올리는**” 전략이 가장 실용적이다.

---

## 부록 A. 마이크로 위젯: Vanilla ↔ React 공존

**Vanilla 앱 내부에 React 위젯 삽입**
```html
<div id="react-widget"></div>
<script type="module">
  import React from 'https://esm.sh/react';
  import ReactDOM from 'https://esm.sh/react-dom/client';
  function Widget(){ return <button onClick={()=>alert('Hi')}>Hi</button>; }
  const root = ReactDOM.createRoot(document.getElementById('react-widget'));
  root.render(<Widget />);
</script>
```
- 레거시/서드파티 환경에 **부분 도입**하는 스텝으로 유용.

---

## 부록 B. 라우팅/상태/템플릿 경량 라이브러리 제안(중립)
- 라우팅: Navigo, tiny-router
- 상태: Nano Stores, Zustand(React), Pinia(Vue)
- 템플릿: lit-html, uhtml  
> 목적은 “프레임워크 없이도 **부분적 구성요소**를 안정적으로 얻는 것”.

---

## 마무리 요약(테이블)

| 선택 기준 | Vanilla JS | 프레임워크 |
|---|---|---|
| 규모/기간 | 소규모/단기 | 중~대형/중장기 |
| 학습·온보딩 | 쉽다 | 개념 많음(그러나 팀 온보딩은 빠를 수 있음) |
| 성능/번들 | 최저 수준 유지 가능 | 초기 오버헤드 있지만 최적화 수단 풍부 |
| 구조/협업 | 스스로 규범 수립 | 관례·도구·생태계 제공 |
| 제품화(SSR/SEO) | 수작업/복잡 | Meta 프레임워크로 간단 |

> **권장 전략**: Vanilla로 시작 → 컴포넌트화/스토어화 → 병목 시 프레임워크 “부분 도입” → 전체 전환.
