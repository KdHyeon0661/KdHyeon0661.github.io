---
layout: post
title: JavaScript - 전개 연산자(Spread)와 나머지 파라미터(Rest)
date: 2025-04-19 22:20:23 +0900
category: JavaScript
---
# 전개 연산자(Spread)와 나머지 파라미터(Rest)

- **Spread(펼침)**: 이터러블/객체의 **요소·프로퍼티를 펼쳐** 전달/복사/병합
- **Rest(수집)**: 남은 인자/프로퍼티를 **배열/객체로 모아** 받음

---

## 한눈에 보는 규칙

- **배열/함수 Spread**는 **이터러블만** 가능(배열, 문자열, Set, Map, 제너레이터…).
  `Math.max(...nums)`, `fn(...args)`
- **객체 Spread**는 **own + enumerable(열거 가능) 프로퍼티**만 복사(문자열/심볼 모두).
  접근자(getter)는 **값으로 평가되어** 복사(접근자 자체는 사라짐).
- Spread/Rest는 **얕은(shallow)** 동작(중첩 객체/배열은 참조 공유).
- **Rest**는 **마지막 위치**에서만(함수 인자/구조 분해 모두).

---

## 배열에서의 Spread — 복사·병합·전달

### 복사(얕은 복사)

```js
const arr1 = [1, 2, { n: 3 }];
const copy = [...arr1];
copy[2].n = 99;
console.log(arr1[2].n); // 99  ← 얕은 복사(참조 공유)
```

### 병합/삽입

```js
const a = [1, 2], b = [3, 4];
const merged = [...a, 0, ...b]; // [1,2,0,3,4]
```

### 인자 전달 — `apply` 대체

```js
function sum(x, y, z) { return x + y + z; }
const nums = [1, 2, 3];
console.log(sum(...nums)); // 6
```

### 문자열·이터러블

```js
[...'hello'];            // ['h','e','l','l','o']
const s = new Set([1,2,3]);
[...s];                  // [1,2,3]

function* g(){ yield 10; yield 20; }
[...g()];                // [10,20]
```

### 처리

```js
const a = [, 2, , 4];   // length=4, 홀(비어있는 슬롯) 포함
[...a];                 // [undefined, 2, undefined, 4]
```
> 배열 spread는 **이터레이터를 통해 값 시퀀스를 생성**하며 **holes를 `undefined`로 채운** **밀집 배열**을 만든다.
> 반면 `map`/`forEach` 등은 holes를 **건너뛴다**.

---

## 객체에서의 Spread — 복사·병합(ES2018+)

### 얕은 복사

```js
const user = { name: "Alice", meta: { score: 10 } };
const copy = { ...user };
copy.meta.score = 42;
console.log(user.meta.score); // 42  ← 얕은 복사
```

### 병합(뒤에 오는 것이 우선)

```js
const a = { name: "Alice", role: "user" };
const b = { role: "admin", active: true };
const merged = { ...a, ...b };
// { name: "Alice", role: "admin", active: true }
```

### 접근자/프로퍼티 기술자 주의

```js
const src = {
  get x(){ return 7; }
};
const dst = { ...src };
const desc = Object.getOwnPropertyDescriptor(dst, 'x');
console.log(desc.get); // undefined  ← getter는 값으로 평가되어 복사
console.log(dst.x);    // 7
```
- 객체 spread는 **데이터 프로퍼티**로 붙는다(`writable/configurable/enumerable: true`).
- 원본의 **접근자(get/set)**, 세밀한 **속성 기술자**는 **보존되지 않는다**.

### 프로토타입/비열거·상속 프로퍼티

- **프로토타입**은 복사되지 않는다(새 객체의 [[Prototype]]은 `Object.prototype`).
- **own + enumerable**만 대상(상속/비열거는 제외). **심볼 enumerable**도 포함.

### 순서 규칙

복사/열거 순서는: **정수형 키 → 문자열 키 → 심볼 키**(각 그룹은 정의 순서).

---

## vs Spread(펼침)

### Rest 파라미터(선언부에서 수집)

```js
function logLevels(first, ...rest) {
  console.log(first); // 첫 인자
  console.log(rest);  // 나머지 배열
}
logLevels("info","warn","error");
// "info", ["warn","error"]
```
- **항상 마지막 매개변수**여야 함.
- `arguments`와 달리 **진짜 배열**이고 **나머지만** 담는다.

### 가변 인자 합계

```js
const sumAll = (...nums) => nums.reduce((a,b)=>a+b, 0);
sumAll(1,2,3,4); // 10
```

### Spread로 인자 전달(호출부에서 펼침)

```js
const args = [1, 2, 3];
fn(...args);
```

---

## 구조 분해와 Rest — 나머지 수집 패턴

### 배열

```js
const [first, ...others] = [10, 20, 30, 40];
// first=10, others=[20,30,40]
```

### 객체

```js
const user = { name: "Bob", age: 27, job: "dev" };
const { name, ...info } = user;
// name: "Bob", info: { age:27, job:"dev" }
```
- 객체 Rest는 **이미 바인딩한 키를 제외한 나머지**(own+enumerable)만 모은다.
- **얕은 복사**로 수집된다.

---

## Spread vs Rest 비교 요약

| 구분 | Spread(펼침) | Rest(수집) |
|---|---|---|
| 문맥 | **호출/리터럴/초기화 시점** | **선언/패턴 좌변** |
| 배열 | 펼쳐서 전달/복사/병합 | 남은 요소를 배열로 수집 |
| 객체 | 프로퍼티를 복사/병합 | 나머지 프로퍼티를 객체로 수집 |
| 함수 | `fn(...args)` | `function fn(...args){}` |
| 제약 | 배열/함수는 **이터러블** 필요, 객체는 **own+enumerable**만 | **마지막 위치**만 가능(함수/패턴) |

---

## 실전 레시피

### 최댓값/최솟값

```js
const nums = [1, 7, 3];
Math.max(...nums); // 7
Math.min(...nums); // 1
```

### 배열 비우지 않고 `push`(여러 개)

```js
const base = [1, 2];
const extra = [3, 4];
base.push(...extra);        // base = [1,2,3,4]
```

### 디폴트 + 오버라이드(얕은 병합)

```js
const defaults = { timeout: 5000, retries: 2, headers: { "X-Req": "base" } };
const cfg = { retries: 5, headers: { "X-Req": "custom" } };
const final = { ...defaults, ...cfg };
// headers는 통째로 덮임(얕은 병합)
```

### 중첩 병합이 필요하면

```js
function deepMerge(a, b){
  if (Array.isArray(a) && Array.isArray(b)) return [...a, ...b];
  if (a && typeof a==='object' && b && typeof b==='object') {
    const out = { ...a };
    for (const k of Reflect.ownKeys(b)) {
      out[k] = k in a ? deepMerge(a[k], b[k]) : b[k];
    }
    return out;
  }
  return b;
}
```

### Set/Map 변환

```js
const set = new Set([1,2,2,3]);
const arr = [...set];                 // [1,2,3]

const map = new Map([["a",1],["b",2]]);
const obj = Object.fromEntries(map);  // { a:1, b:2 }
```

### 안전한 옵션 패턴(함수 인자 + Rest)

```js
function connect({ host="localhost", port=3306, ...rest } = {}) {
  return { host, port, rest };
}
connect({ port: 5432, ssl: true });
// { host:"localhost", port:5432, rest:{ ssl:true } }
```

---

## 성능·가독성 팁

- **큰 배열 복사/병합**을 빈번히 하면 비용↑.
  성능이 중요하면 **원본 재사용** 또는 **단일 루프**를 고려.
- `filter(...).map(...).reduce(...)` 체이닝 vs **단일 패스** `reduce`는 **측정 후 결정**.
- 객체 spread는 **접근자 평가**와 **기술자 재설정**이 일어나므로,
  원본의 고급 기술자를 보존하려면 `Object.defineProperties`/`Reflect` 기반 복사가 필요.

---

## 자주 하는 오해/함정

### 깊은 복사 아님

```js
const o1 = { nested: { v: 1 } };
const o2 = { ...o1 };
o2.nested.v = 9;
console.log(o1.nested.v); // 9
```
> 깊은 복사가 필요하면 `structuredClone(o1)`(지원 환경) 또는 커스텀/라이브러리 사용.

### 객체 spread는 접근자/프로토타입 미보존

- getter/setter → **값**으로 복사, 접근자 사라짐
- `Object.getPrototypeOf(copy) === Object.prototype`

### `null`/`undefined`는 펼칠 수 없음

```js
const x = null;
// [...x];          // TypeError: null is not iterable
// { ...x };        // TypeError: Cannot convert undefined or null to object
```

### Rest 위치 제약

```js
// function f(...a, b) {} // SyntaxError
function f(a, ...rest) { /* ok */ }
```

### 희소 배열 기대와 달리

- `[...Array(3)]` → `[undefined, undefined, undefined]` (밀집)
- `Array(3).map(()=>1)`은 콜백이 **호출되지 않음**(holes 건너뜀)

### 객체 키 충돌 순서

```js
const merged = { a:1, ...{ a:2 } };
console.log(merged.a); // 2 (뒤에 오는 값 우선)
```

---

## 디버깅/점검 체크리스트

- [ ] **대상 타입** 확인: 배열/함수 Spread는 이터러블? 객체 Spread는 null/undefined 아님?
- [ ] **얕은 복사**임을 기억: 중첩 변경 전파 위험.
- [ ] **접근자/기술자/프로토타입** 필요? → spread 대신 정교한 복사 사용.
- [ ] **대량 데이터**에서 spread 반복 사용 시 성능 측정.
- [ ] 구조 분해 Rest는 **마지막**에만.

---

## 실습 스니펫

### holes와 spread 차이 체감

```js
const a = Array(3);                 // [ <3 empty items> ]
console.log(a.map(x => 1));         // [ <3 empty items> ] (콜백 미호출)
console.log([...a]);                // [ undefined, undefined, undefined ]
```

### getter가 값으로 복사됨

```js
const src = { get t(){ console.log("hit"); return 5; } };
const dst = { ...src };             // "hit" (복사 시 평가됨)
console.log(Object.getOwnPropertyDescriptor(dst,"t").get); // undefined
```

### 얕은 병합에서의 덮어쓰기

```js
const base = { cfg:{ a:1, b:2 } };
const over = { cfg:{ b:9 } };
const merged = { ...base, ...over }; // cfg는 통째로 교체
console.log(merged.cfg);             // { b:9 }
```

---

## 미니 퀴즈

```js
// Q1: 결과는?
const arr = [,2,,4];
console.log([...arr]);

// Q2: 아래 코드의 문제점과 수정?
function g(...args, tail) { return args.length; }

// Q3: 값은?
const o = { get x(){ return 1 }, y:2 };
const c = { ...o };
console.log(Object.getOwnPropertyDescriptor(c,'x').get, c.x);

// Q4: TypeError가 나는 줄은?
// A) [...null]
// B) { ...undefined }
// C) { ...{ a:1 } }

// Q5: 깊은 병합으로 { a:{x:1,y:2} }와 { a:{y:9,z:3} }를 합치면?
// 얕은 spread 결과와 깊은 병합 결과의 차이를 서술.
```

**힌트**
- Q1: `[undefined, 2, undefined, 4]`.
- Q2: Rest는 마지막에만. `function g(tail, ...args){}` 또는 `function g(...args){ const tail=args.at(-1) }`.
- Q3: getter는 값으로 복사 → `get`는 `undefined`, `c.x===1`.
- Q4: A,B는 TypeError, C는 정상.
- Q5: 얕은: `{ a:{ y:9, z:3 } }`(통째 교체). 깊은: `{ a:{ x:1, y:9, z:3 } }`.

---

## 결론

- `...`은 **펼침(Spread)**과 **수집(Rest)**이라는 상반된 역할을 한다.
- Spread는 **전달/복사/병합**, Rest는 **가변 수집/나머지 캡처**를 간결하게 만든다.
- 다만 둘 다 **얕은 동작**이며, **객체 spread는 접근자/프로토타입을 보존하지 않는다**.
- **희소 배열**, **순서 규칙**, **타입 제약**만 정확히 기억하면 실무에서 안전하고 빠르게 쓸 수 있다.
