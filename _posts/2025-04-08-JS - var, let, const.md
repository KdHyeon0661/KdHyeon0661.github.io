---
layout: post
title: JavaScript - var, let, const
date: 2025-04-08 20:20:23 +0900
category: JavaScript
---
# 변수 선언 방식 비교: `var`, `let`, `const`

## 배경 지식: 실행 컨텍스트와 변수 환경

자바스크립트 엔진은 **실행 컨텍스트(Execution Context)**를 만들고, 그 안에 **변수 환경(Variable Environment)**과 **렉시컬 환경(Lexical Environment)**을 유지한다. 선언문은 **호이스팅(Declaration Hoisting)** 단계에서 등록되고, 초기화 시점과 가시성은 선언 종류에 따라 다르다.

- `var` **선언(Declaration)** 은 **환경에 등록 + 즉시 `undefined` 초기화** → 선언 전 읽기 시 `undefined`.
- `let`/`const`는 **환경에 등록되지만 초기화 전 상태(TDZ)** → 선언 전 읽기/쓰기 **ReferenceError**.
- `const`는 **초기화와 선언이 동시에** 이뤄져야 함(초기화 없이 선언 불가).

---

## `var`: ES5 이전 방식

### 특징 정리

- **함수 스코프(Function Scope)** — 블록(`{}`)을 무시하고 **가장 가까운 함수**를 스코프로 사용.
- **선언 호이스팅 + `undefined` 초기화** — 선언 전 접근 가능(값은 `undefined`).
- **재선언 가능** — 같은 스코프에서 다시 `var` 선언해도 에러 없음.
- **전역에서 사용 시**: 전역 `var`는 **전역 객체(window/globalThis)의 프로퍼티**가 될 수 있음(모듈 스코프는 예외).

```js
console.log(x); // undefined (호이스팅 + 초기화)
var x = 10;
console.log(x); // 10

var x = 20;     // 재선언 허용
console.log(x); // 20
```

### 블록 스코프 무시 사례

```js
if (true) {
  var msg = "hello";
}
console.log(msg); // "hello" (블록을 탈출함)
```

### 함수 스코프 동작

```js
function f() {
  if (true) {
    var a = 1;
  }
  console.log(a); // 1 (함수 스코프)
}
f();
// console.log(a); // ReferenceError: a is not defined
```

### 전역 오염 및 삭제 불가 이슈

```js
var g = 1;               // 비모듈 스크립트에서 window.g 생성
console.log("g" in window); // true

// var로 만든 전역은 configurable=false 여서 delete 불가
// delete window.g; // false (삭제 실패)
```

> **결론**: 전역 오염과 재선언, 루프 캡처 문제 때문에 **새 코드에서 `var`는 지양**.

---

## `let`: ES6의 블록 스코프 변수

### 특징 정리

- **블록 스코프(Block Scope)** — `{}` 범위에 갇힘.
- **중복 선언 불가** — 같은 스코프에서 재선언 시 에러.
- **호이스팅되나 TDZ 존재** — 초기화 전 접근은 **ReferenceError**.
- **재할당 가능**.

```js
let y = 10;
y = 20; // OK
```

### 예제

```js
// TDZ 구간에서 접근 → ReferenceError
console.log(a); // ReferenceError
let a = 5;
```

### 블록 스코프 예제

```js
if (true) {
  let msg = "hello";
}
// console.log(msg); // ReferenceError
```

---

## + 블록 스코프

### 특징 정리

- **블록 스코프**
- **재선언 불가**
- **호이스팅 + TDZ** — 선언 전 접근 불가.
- **재할당 불가**(초기화 필수).

```js
const PI = 3.14;
// PI = 3.1415; // TypeError: Assignment to constant variable.
```

### 객체/배열은 “참조가 상수일 뿐”

```js
const user = { name: "Alice" };
user.name = "Bob"; // OK (객체 내용 변경은 가능)

// user = {}; // TypeError (참조 자체 재대입은 불가)
```

#### 불변 데이터가 필요하면

- **얕은 불변**: `Object.freeze(obj)` (중첩 객체는 여전히 변경됨)
- **깊은 불변**: 재귀 `deepFreeze` 또는 Immer 같은 라이브러리 사용

```js
function deepFreeze(obj) {
  if (obj && typeof obj === "object" && !Object.isFrozen(obj)) {
    Object.freeze(obj);
    for (const k of Reflect.ownKeys(obj)) deepFreeze(obj[k]);
  }
  return obj;
}
```

---

## 호이스팅과 TDZ — 세밀 동작 비교

### 관찰 코드

```js
// case A: var
console.log(a); // undefined
var a = 1;

// case B: let
// console.log(b); // ReferenceError (TDZ)
let b = 2;

// case C: const
// console.log(c); // ReferenceError (TDZ)
const c = 3;
```

- `var`: 선언 단계에서 **환경에 등록 + 초기화(undefined)** → 접근 가능
- `let`/`const`: 선언은 등록되나 **초기화 전 상태(TDZ)** → 접근 시 **ReferenceError**

### TDZ는 어디까지인가?

```js
{
  // TDZ 시작
  // console.log(v); // ReferenceError
  let v = 10;      // TDZ 종료(초기화 지점)
  console.log(v);  // 10
}
```

---

## 정리

### 재선언과 재할당

```js
// 같은 스코프
var a = 1;
var a = 2; // 재선언 OK

let b = 1;
// let b = 2; // SyntaxError (같은 스코프 재선언 불가)

const c = 1;
// c = 2;     // TypeError (재할당 불가)
```

### 섀도잉: 내부 스코프에서 이름 재사용

```js
let x = 1;
{
  let x = 2;    // 바깥 x를 가림(섀도잉)
  console.log(x); // 2
}
console.log(x);   // 1
```

---

## 전역 스코프와 모듈 스코프

### 비모듈 스크립트(브라우저)

```html
<script>
  var g1 = "var-global";
  let g2 = "let-global";
  const g3 = "const-global";

  console.log(window.g1); // "var-global"
  console.log("g2" in window, "g3" in window); // false false
</script>
```

- 전역 `var`는 **전역 객체 프로퍼티**가 됨.
- 전역 `let`/`const`는 **전역 렉시컬 환경**에 만들어지고 전역 객체에 붙지 않음.

### ES 모듈

모듈 파일(`type="module"` 또는 `.mjs`)은 **모듈 스코프**를 갖고, 최상단 `this`는 `undefined`. 전역 `var`라도 `window`에 붙지 않는다.

```html
<script type="module">
  var g = 1;
  console.log("g" in window); // false (모듈 스코프)
</script>
```

---

## 루프와 클로저 캡처 — `var` vs `let`

### `var`로 인한 흔한 버그

```js
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0);
}
// 출력: 3 3 3 (하나의 i를 공유)
```

### `let`은 반복마다 새로운 바인딩

```js
for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0);
}
// 출력: 0 1 2
```

### `for` 헤드에서의 `const`

루프마다 **새 바인딩**이 생성되므로 다음도 가능(단, 같은 반복 안에서는 재할당 불가).

```js
for (const v of [10, 20, 30]) {
  // v = 99; // TypeError
  setTimeout(() => console.log(v), 0); // 10 20 30
}
```

### `var` 사용 시 IIFE로 우회(구형 코드)

```js
for (var i = 0; i < 3; i++) {
  (function (j) {
    setTimeout(() => console.log(j), 0);
  })(i);
}
```

---

## try/catch, 함수 선언과 블록, strict 모드

### `catch` 바인딩은 블록 스코프

```js
try {
  throw 1;
} catch (e) {
  let a = e;
}
// console.log(a); // ReferenceError
```

### 블록 내 함수 선언(브라우저 비표준/표준 차이 주의)

ES2015+에서 블록 내 함수 선언은 환경에 따라 처리 방식이 다를 수 있다. **안전하게는 함수 표현식 + `let/const`**를 권장.

```js
if (cond) {
  const helper = function () { /* ... */ };
}
```

### strict 모드

- 엄격 모드에서 실수로 전역에 **암묵적 변수 생성**이 방지됨.
- `var`도 엄격 모드라 해서 스코프가 바뀌진 않지만, 전반적 오류 탐지에 유리.

```js
"use strict";
function f() {
  // o = 1; // ReferenceError (암묵 전역 금지)
}
```

---

## 패턴·베스트 프랙티스

### 기본 규칙

- **기본은 `const`**, 바꾸는 값만 **의도적으로 `let`**.
- **`var`는 사용하지 않기**(전역 오염, 루프 버그, 재선언 등).
- 블록 스코프에 변수 **최소화**하여 가독성과 안전성 확보.

```js
const TAX_RATE = 0.1;

function calc(price) {
  let result = price * (1 + TAX_RATE);
  return result;
}
```

### 린트/포맷 설정 예시(ESLint)

```json
{
  "rules": {
    "no-var": "error",
    "prefer-const": ["warn", { "destructuring": "all" }],
    "no-redeclare": "error",
    "no-shadow": "warn",
    "block-scoped-var": "error"
  }
}
```

### 구조 분해 + `const`/`let` 전략

```js
// 불변 파트는 const, 변하는 파트만 let
const { id, name } = user;
let { page = 1 } = options ?? {};
page++;
```

---

## 미묘한 함정과 팁

### 호이스팅 순서 상관관계

동일 스코프에서 `function` 선언이 `var`보다 우선순위가 높다(함수 선언문이 먼저 바인딩). 그러나 블록 내 함수 선언은 환경 차이 가능 → **함수 표현식 권장**.

```js
foo(); // OK
function foo() {}

console.log(bar); // undefined (var 호이스팅)
var bar = 1;
```

### TDZ는 평가 타이밍과도 관련

```js
{
  // 초기화 표현식에서 같은 블록의 let 변수를 참조하면 TDZ
  // const v = v; // TDZ ReferenceError
}
```

### 순환 의존 및 모듈 바인딩

ES 모듈은 **라이브 바인딩**을 제공. 단, 초기화 타이밍에 TDZ가 개입할 수 있으므로 **상호 참조 구조** 설계에 유의.

---

## 실전 시나리오

### DOM 이벤트 루프 캡처

```html
<ul id="menu">
  <li>0</li><li>1</li><li>2</li>
</ul>
<script>
const items = document.querySelectorAll("#menu li");

// 나쁜 예: var
for (var i = 0; i < items.length; i++) {
  items[i].addEventListener("click", () => {
    console.log("clicked:", i); // 모두 items.length 출력
  });
}

// 좋은 예: let
for (let j = 0; j < items.length; j++) {
  items[j].addEventListener("click", () => {
    console.log("clicked:", j); // 각 인덱스 출력
  });
}
</script>
```

### 비동기 콜백에서의 인덱스 보존

```js
const urls = ["/a", "/b", "/c"];

// let으로 인덱스 캡처
for (let i = 0; i < urls.length; i++) {
  fetch(urls[i]).then(() => console.log("done", i));
}
```

### 값이 절대 바뀌지 않는 구성은 `const`

```js
const DEFAULT_TIMEOUT_MS = 5000;
const ENDPOINTS = Object.freeze({
  users: "/api/users",
  posts: "/api/posts"
});
```

### 계산 과정에서만 변하는 누산기는 `let`

```js
function sum(arr) {
  let acc = 0;
  for (const x of arr) acc += x;
  return acc;
}
```

---

## 비교 요약 표(확장)

| 특징                 | `var`                          | `let`                                | `const`                               |
|----------------------|--------------------------------|--------------------------------------|---------------------------------------|
| 스코프               | 함수 스코프                    | 블록 스코프                           | 블록 스코프                            |
| 호이스팅             | 선언 + **`undefined` 초기화**  | 선언만(초기화 전 **TDZ**)             | 선언만(초기화 전 **TDZ**)              |
| 선언 전 접근         | 가능(`undefined`)              | **ReferenceError**                    | **ReferenceError**                     |
| 재선언               | 허용                           | 불가                                  | 불가                                   |
| 재할당               | 가능                           | 가능                                  | **불가**                               |
| 전역 바인딩          | 전역 객체에 붙을 수 있음       | 전역 객체에 붙지 않음                 | 전역 객체에 붙지 않음                  |
| 루프 캡처            | 하나의 바인딩 공유(버그 유발)  | 반복마다 새 바인딩(의도대로 캡처)     | 반복마다 새 바인딩(재할당만 불가)      |
| 권장 여부            | 지양                            | 추천(변하는 값)                        | 추천(변하지 않는 값, 상수/의미 고정)   |

---

## 미니 퀴즈(자기 점검)

```js
// Q1
console.log(a);
var a = 1;           // ?

// Q2
// console.log(b);
let b = 2;           // ?

// Q3
for (var i = 0; i < 2; i++) {
  setTimeout(() => console.log(i), 0);
}                     // ?

// Q4
for (let j = 0; j < 2; j++) {
  setTimeout(() => console.log(j), 0);
}                     // ?

// Q5
const cfg = { dark: false };
cfg.dark = true;      // ?
```

정답 힌트: `undefined`, ReferenceError, `2 2`, `0 1`, 객체 내용 변경은 가능.

---

## 실무 체크리스트

- [ ] **`no-var`**: 새 코드에서 `var` 사용 금지.
- [ ] 기본은 **`const`**, 변경 의도가 분명할 때만 `let`.
- [ ] 선언은 **가장 좁은 블록**에 배치(스코프 최소화).
- [ ] 루프 인덱스/클로저 캡처는 **`let`** 사용.
- [ ] 전역 노출 금지: 모듈 스코프/즉시 실행 함수/번들 스코프 사용.
- [ ] 상수/설정은 **동결(Object.freeze)** 또는 **읽기 전용 래퍼** 고려.
- [ ] 블록 내 함수는 **함수 표현식 + const**로 안전하게.
- [ ] 린트 규칙: `no-var`, `prefer-const`, `no-redeclare`, `no-shadow`.

---

## 결론

`var`/`let`/`const`의 차이는 단순한 문법이 아니라 **실행 모델**(스코프·호이스팅·TDZ)과 **전역 오염/루프 캡처** 같은 **버그 양상**을 결정한다.
모던 자바스크립트에서는 **기본 `const` → 필요한 곳만 `let`**, **`var` 지양**을 원칙으로 삼고, 블록 스코프를 적극 활용하라.
이 원칙만 확실히 지켜도 가독성, 리팩터링 안전성, 런타임 오류 예방에서 즉각적인 이득을 얻을 수 있다.
