---
layout: post
title: JavaScript - 스코프(Scope)와 클로저(Closure)
date: 2025-04-12 22:20:23 +0900
category: JavaScript
---
# 스코프(Scope)와 클로저(Closure)

## 용어 및 수학적 관점(간단 정의)

엔진이 관리하는 렉시컬 환경을 다음 튜플로 볼 수 있다:

$$
\text{LexEnv} = (\text{EnvRecord}, \text{Outer})
$$

- `EnvRecord`: 현재 스코프에 선언된 바인딩(식별자 → 값/슬롯)
- `Outer`: 바깥 렉시컬 환경 참조(스코프 체인)

이 체인을 따라 **이름 해석(name resolution)**이 이뤄진다.

---

## 스코프(Scope) — 범위의 종류와 해석 규칙

### 스코프의 종류

| 스코프 | 설명 | 대표 선언 |
|---|---|---|
| 전역(Global) | 프로그램 전체에서 접근 | 전역 `var`/`let`/`const`(모듈과 구분) |
| 함수(Function) | 함수 내부에서만 접근 | `var`, 함수 파라미터, 함수 선언 |
| 블록(Block) | `{}` 내부에서만 접근 | `let`, `const`, `class`, `try/catch` 바인딩 |
| 모듈(Module) | 각 모듈 파일 단위로 독립 | `import`/`export`, 최상단 `this===undefined` |

> **렉시컬 스코프**: “함수가 **어디서 호출**되었는지”가 아니라, **어디서 정의**되었는지로 스코프가 결정된다.

### 전역 vs 함수 vs 블록 스코프

```js
let globalVar = "global";

function test() {
  let localVar = "local";
  console.log(globalVar); // OK
  console.log(localVar);  // OK
}

test();
console.log(globalVar);   // OK
// console.log(localVar); // ReferenceError
```

```js
{
  let a = 1;
  const b = 2;
}
// console.log(a, b); // ReferenceError (블록 밖에서 접근 불가)
```

### 렉시컬 스코프(정의 위치 기준)

```js
let x = 10;
function outer() {
  let x = 20;
  function inner() {
    console.log(x); // 20 — 정의된 위치(outer)의 스코프를 본다
  }
  inner();
}
outer();
```

---

## 실행 컨텍스트·렉시컬 환경 — 엔진이 스코프를 다루는 방식

자바스크립트는 코드를 실행할 때 **실행 컨텍스트(Execution Context)**를 만들고, 그 안에 **렉시컬 환경**을 구성한다.

- **호이스팅 단계**: 선언을 환경에 **등록**
  - `var`: 선언 + **`undefined` 초기화**
  - `let`/`const`: 선언만, 초기화 전 구간은 **TDZ(Temporal Dead Zone)**
- **평가/실행 단계**: 실제 코드 수행, 바인딩 읽기/쓰기

관찰 코드:

```js
console.log(a); // undefined — var는 호이스팅 + 초기화
var a = 1;

// console.log(b); // ReferenceError — TDZ
let b = 2;

// console.log(c); // ReferenceError — TDZ
const c = 3;
```

TDZ 범위는 **해당 선언이 평가되어 초기화되기 전까지**다.

---

## 클로저(Closure) — 정의, 동작, 메모리

> “클로저는 **함수가 생성될 때의 렉시컬 환경을 기억**하여, 함수가 살아있는 동안 그 환경의 식별자에 접근할 수 있게 해주는 기능”이다.

### 기본 예제(상태 유지)

```js
function makeCounter() {
  let count = 0;        // 캡처되는 상태
  return function () {  // 클로저
    count++;
    return count;
  };
}

const counter = makeCounter();
console.log(counter()); // 1
console.log(counter()); // 2
console.log(counter()); // 3
```

### 정보 은닉/캡슐화(게터/세터)

```js
function createUser(name) {
  let privateName = name;
  return {
    getName() { return privateName; },
    setName(n) { privateName = n; }
  };
}

const u = createUser("Alice");
console.log(u.getName()); // "Alice"
u.setName("Bob");
console.log(u.getName()); // "Bob"
```

외부에서 `privateName`에 직접 접근할 수 없고, 오직 제공된 인터페이스로만 제어한다.

### 메모리 관점

- **클로저가 참조하는 바깥 변수**는 함수가 살아있는 동안 **GC 대상에서 제외**된다(참조가 남아있으므로).
- 이벤트 핸들러/타이머/전역 배열에 **클로저를 오래 저장**하면 해당 환경도 오래 유지된다 → *누수처럼 보일 수 있음*.

**누수 완화 팁**

```js
function attach() {
  let cache = new Map();
  const onClick = () => { /* cache 사용 */ };
  btn.addEventListener("click", onClick);

  return () => {
    btn.removeEventListener("click", onClick);
    cache.clear(); // 참조 끊기
  };
}

const detach = attach();
// 필요 없어지면
detach();
```

---

## 반복문과 클로저 — 루프 캡처 이슈

### `var` 사용 시 버그

```js
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 10);
}
// 3 3 3 — 단 하나의 i 바인딩을 공유
```

### `let`으로 해결(반복마다 새 바인딩)

```js
for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 10);
}
// 0 1 2
```

### 구형 코드에서의 IIFE 패턴

```js
for (var i = 0; i < 3; i++) {
  (function (j) {
    setTimeout(() => console.log(j), 10);
  })(i);
}
```

### 콜백 파라미터 캡처 활용

```js
[10, 20, 30].forEach((v, idx) => {
  setTimeout(() => console.log(idx, v), 10);
});
```

---

## 모듈 스코프·전역 바인딩·this와의 관계

### 비모듈 스크립트(브라우저)

```html
<script>
  var g1 = 1;               // window.g1 생성
  let g2 = 2; const g3 = 3; // 전역 렉시컬 환경 (window에 붙지 않음)
  console.log(window.g1, "g2" in window, "g3" in window); // 1 false false
</script>
```

### ES 모듈

- 모듈 파일(`type="module"`, `.mjs`)은 **모듈 스코프**를 갖는다.
- 최상단 `this === undefined`, 전역 `var`라도 `window`에 붙지 않는다.
- **캡슐화**와 **의존성 관리**에 유리: `export`로 공개 대상을 명시.

---

## 이름 해석 규칙(변수 탐색 알고리즘)

1. **현재 렉시컬 환경**에서 식별자 탐색
2. 없으면 **Outer**로 이동
3. 전역 환경까지 없다면 **ReferenceError**

예제:

```js
let a = "G";
function f() {
  let a = "L1";
  function g() {
    let a = "L2";
    console.log(a); // "L2"
  }
  g();
}
f();
```

---

## TDZ(Temporal Dead Zone) — 선언 전 접근 금지

```js
{
  // console.log(x); // ReferenceError — TDZ
  let x = 1;          // 여기서 초기화, TDZ 종료
  console.log(x);     // 1
}
```

초기화 표현식에서 같은 블록의 식별자를 참조해도 TDZ에 걸릴 수 있다.

```js
{
  // const v = v; // ReferenceError — 아직 초기화 전
}
```

---

## 클로저 실전 패턴 모음

### once: 한 번만 실행

```js
function once(fn) {
  let called = false, result;
  return (...args) => {
    if (!called) { called = true; result = fn(...args); }
    return result;
  };
}

const init = once(() => ({ db: "connected" }));
console.log(init() === init()); // true
```

### debounce/throttle(상태 유지)

```js
function debounce(fn, wait = 200) {
  let t;
  return (...args) => {
    clearTimeout(t);
    t = setTimeout(() => fn(...args), wait);
  };
}

function throttle(fn, wait = 200) {
  let last = 0;
  return (...args) => {
    const now = Date.now();
    if (now - last >= wait) {
      last = now;
      fn(...args);
    }
  };
}
```

### 커링/부분 적용

```js
const add = (a) => (b) => a + b;
const add10 = add(10);
console.log(add10(5)); // 15
```

### 모듈 패턴(IIFE) — 레거시 환경

```js
const CounterModule = (function () {
  let c = 0;
  function inc() { c++; }
  function get() { return c; }
  return { inc, get };
})();
CounterModule.inc();
console.log(CounterModule.get()); // 1
```

### 클래스의 프라이빗 필드 vs 클로저

```js
// # 프라이빗 필드
class Bank {
  #balance = 0;
  deposit(x) { this.#balance += x; }
  get balance() { return this.#balance; }
}

// 클로저 기반
function Bank2() {
  let balance = 0;
  return {
    deposit(x) { balance += x; },
    get balance() { return balance; }
  };
}
```

### DOM과 데이터 결합(WeakMap로 누수 완화)

```js
const store = new WeakMap();

function bind(node, data) {
  store.set(node, data); // node가 GC되면 엔트리도 수거
  node.addEventListener("click", () => {
    const d = store.get(node);
    console.log("clicked:", d);
  });
}
```

---

## 비동기 + 클로저 — 타이머/Promise/async

### 타이머와 캡처

```js
function scheduleTasks() {
  for (let i = 1; i <= 3; i++) {
    setTimeout(() => console.log("T", i), i * 10);
  }
}
scheduleTasks();
// T 1, T 2, T 3
```

### `for` + `await` 캡처(반복마다 새 바인딩이므로 안전)

```js
async function run() {
  for (let i = 1; i <= 3; i++) {
    await new Promise(r => setTimeout(r, 10));
    console.log("A", i);
  }
}
run();
// A 1, A 2, A 3
```

### 값 스냅샷이 필요한 경우(기본값 파라미터 트릭)

```js
for (let i = 0; i < 3; i++) {
  setTimeout(((j = i) => () => console.log(j))(), 10);
}
```

---

## 보안/안정성 — `eval`/`with`와 스코프

- `eval`은 현재 스코프에 영향을 줄 수 있어 **지양**. 엄격 모드에서는 영향 축소.
- `with`는 렉시컬 해석을 혼란스럽게 하므로 **사용 금지**(strict 모드 금지).

```js
"use strict";
// eval("var x = 1"); // 현재 스코프 오염 우려 → 피하기
```

---

## 디버깅/프로파일링 팁

- DevTools Sources/Scope 패널에서 **Closures**/`Local`/`Global` 확인.
- 메모리 프로파일러로 **Detached DOM**과 **Listener**를 점검.
- 장시간 유지되는 클로저가 참조하는 대상을 **의도적으로 해제**(배열 `length=0`, `Map.clear()`, 이벤트 제거 등).

---

## 실전 예제 — 버튼 클릭 카운터(확장 버전)

### 기본(단일 버튼)

```js
function createCounterHandler() {
  let count = 0;
  return () => {
    count++;
    console.log(`Clicked ${count} times`);
  };
}

const handler = createCounterHandler();
document.querySelector("button").addEventListener("click", handler);
```

### 여러 버튼에 독립 상태 부여

```js
function attachCounter(btn) {
  let count = 0;
  const onClick = () => btn.textContent = `Clicked ${++count}`;
  btn.addEventListener("click", onClick);
  return () => btn.removeEventListener("click", onClick); // detach
}

const offs = [...document.querySelectorAll("button")].map(attachCounter);
// 나중에 필요 없으면: offs.forEach(off => off());
```

### 요청 취소(AbortController) + 클로저로 상태 보관

```js
function makeFetchOnce(url) {
  let done = false, ctrl;
  return async () => {
    if (done) return;
    ctrl = new AbortController();
    try {
      const res = await fetch(url, { signal: ctrl.signal });
      console.log(await res.text());
      done = true;
    } catch (e) {
      if (e.name !== "AbortError") throw e;
    }
  };
}

const run = makeFetchOnce("/api/data");
// run()을 여러 번 눌러도 최초 1회만 동작
```

---

## 체크리스트(요약)

- [ ] **렉시컬 스코프**: 정의 위치가 곧 스코프.
- [ ] 스코프 종류: 전역/함수/블록/모듈 — 구분해서 사용.
- [ ] **TDZ**: `let`/`const`는 선언 전 접근 금지.
- [ ] **클로저**: 상태 유지/캡슐화/유틸 구현에 적극 활용.
- [ ] 루프 캡처: `let` 사용, 구형 코드는 IIFE 등으로 대응.
- [ ] 이벤트/타이머 해제: **참조 끊기**로 메모리 누수 예방.
- [ ] 모듈 스코프로 전역 오염 방지, `eval`/`with` 지양.
- [ ] DevTools로 클로저와 메모리 상태 점검.

---

## 미니 퀴즈

```js
// Q1: 출력은?
let x = 1;
function a() {
  console.log(x);
  let x = 2;
}
try { a(); } catch (e) { console.log(e.name); }

// Q2: 출력은?
for (var i = 0; i < 2; i++) {
  setTimeout(() => console.log(i), 0);
}

// Q3: 캡슐화한 카운터의 count를 밖에서 바꿀 수 있는가?
function make() {
  let count = 0;
  return { inc(){count++}, get(){return count} };
}
const c = make();
```

정답 힌트:
Q1) `ReferenceError`(TDZ), Q2) `2 2`, Q3) 직접 변경 불가(인터페이스로만).

---

## 결론

- 스코프는 **이름 해석의 경로**이고, 클로저는 **그 경로(환경)를 보존**한다.
- 이 원리를 이해하면 **의도치 않은 참조/전역 오염/루프 버그**를 줄이고, **정보 은닉·상태 유지**·유틸 구현을 견고하게 만들 수 있다.
- 실무에서는 **모듈 스코프**, **`let`/`const` + TDZ**, **이벤트 해제 습관**, **WeakMap/프라이빗 필드**를 조합해 **안정성과 성능**을 동시에 확보하라.
