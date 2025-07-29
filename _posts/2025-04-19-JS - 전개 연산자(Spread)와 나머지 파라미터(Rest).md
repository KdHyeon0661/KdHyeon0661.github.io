---
layout: post
title: JavaScript - 전개 연산자(Spread)와 나머지 파라미터(Rest)
date: 2025-04-19 22:20:23 +0900
category: JavaScript
---
# 전개 연산자(Spread)와 나머지 파라미터(Rest)

자바스크립트에서 `...`은 문맥에 따라 다르게 해석됩니다.  
- **전개 연산자(Spread)**는 **배열이나 객체를 펼쳐서 복사/병합**할 때 사용하고,  
- **나머지 파라미터(Rest)**는 **여러 인자를 하나의 배열로 수집**할 때 사용됩니다.

---

## ✅ 1. 전개 연산자 (Spread Operator)

> 배열, 객체, 문자열 등을 **하나씩 펼쳐서 개별 요소로 분해**합니다.

### 🔹 배열 복사

```js
const arr1 = [1, 2, 3];
const copy = [...arr1];
console.log(copy); // [1, 2, 3]
```

### 🔹 배열 병합

```js
const a = [1, 2];
const b = [3, 4];
const merged = [...a, ...b]; // [1, 2, 3, 4]
```

### 🔹 함수 인자에 전달 (기존: apply 대체)

```js
function sum(x, y, z) {
  return x + y + z;
}

const nums = [1, 2, 3];
console.log(sum(...nums)); // 6
```

### 🔹 문자열 분해

```js
const str = "hello";
const chars = [...str]; // ['h', 'e', 'l', 'l', 'o']
```

---

## ✅ 2. 객체에서의 Spread (ES2018+)

### 🔹 객체 복사

```js
const user = { name: "Alice", age: 25 };
const copy = { ...user };
console.log(copy); // { name: "Alice", age: 25 }
```

### 🔹 객체 병합

```js
const a = { name: "Alice" };
const b = { age: 30 };
const merged = { ...a, ...b }; // { name: "Alice", age: 30 }
```

> 동일한 key가 있으면 **나중에 오는 값으로 덮어씌워짐**

---

## ✅ 3. 나머지 파라미터 (Rest Parameter)

> 함수 정의 시, **여러 인자를 하나의 배열로 수집**

### 🔹 기본 사용법

```js
function printAll(...args) {
  console.log(args);
}

printAll(1, 2, 3); // [1, 2, 3]
```

### 🔹 고정 파라미터 + 나머지

```js
function logLevels(first, ...rest) {
  console.log("first:", first);
  console.log("rest:", rest);
}

logLevels("info", "warn", "error");
// first: info
// rest: ["warn", "error"]
```

---

## ✅ 4. 구조 분해에서의 Rest

### 🔹 배열 구조 분해

```js
const [first, ...others] = [10, 20, 30, 40];
console.log(first);  // 10
console.log(others); // [20, 30, 40]
```

### 🔹 객체 구조 분해

```js
const user = {
  name: "Bob",
  age: 27,
  job: "developer"
};

const { name, ...info } = user;
console.log(name); // Bob
console.log(info); // { age: 27, job: "developer" }
```

---

## ✅ 5. Spread vs Rest 비교

| 항목            | 전개 연산자 (Spread)                         | 나머지 파라미터 (Rest)                         |
|------------------|----------------------------------------------|-------------------------------------------------|
| 문맥             | **사용 시점**에 따라 결정됨                | **함수 인자나 구조 분해** 시 사용               |
| 배열             | 배열을 펼쳐서 전달                          | 여러 값을 배열로 수집                           |
| 함수 인자        | 전달 시 사용: `fn(...arr)`                  | 선언 시 사용: `function fn(...args)`            |
| 구조 분해        | 배열/객체를 펼쳐서 복사/병합                | 나머지 요소 수집                                |
| 위치 제한        | 아무 위치나 가능                            | **항상 마지막 인자**로만 사용 가능              |

---

## ✅ 6. 활용 예시

### 🔹 최댓값 구하기

```js
const nums = [1, 7, 3];
console.log(Math.max(...nums)); // 7
```

### 🔹 함수에서 가변 인자 처리

```js
function sumAll(...nums) {
  return nums.reduce((a, b) => a + b, 0);
}

sumAll(1, 2, 3, 4); // 10
```

### 🔹 배열 비우지 않고 push

```js
const base = [1, 2];
const extra = [3, 4];

base.push(...extra);
console.log(base); // [1, 2, 3, 4]
```

---

## ✅ 마무리

- `...`은 상황에 따라 **Spread(펼침)** 또는 **Rest(수집)** 역할을 합니다.
- **Spread**는 복사, 병합, 함수 인자 전달에 매우 유용하며,  
  **Rest**는 가변 인자나 구조 분해에 자주 쓰입니다.
- 가독성과 재사용성 측면에서 **함수형 스타일** 코딩에 필수 도구입니다.