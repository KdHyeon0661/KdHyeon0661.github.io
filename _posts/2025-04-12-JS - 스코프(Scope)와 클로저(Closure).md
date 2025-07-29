---
layout: post
title: JavaScript - 스코프(Scope)와 클로저(Closure)
date: 2025-04-12 22:20:23 +0900
category: JavaScript
---
# 스코프(Scope)와 클로저(Closure) 기초

자바스크립트에서 **스코프(Scope)**는 변수에 접근할 수 있는 범위를 의미하고,  
**클로저(Closure)**는 함수가 선언된 시점의 스코프를 기억하는 기능입니다.  

이 개념은 **변수의 생명주기**, **메모리 관리**, **캡슐화** 등과 깊은 관련이 있습니다.

---

## ✅ 1. 스코프(Scope)란?

**스코프**는 변수를 **선언한 위치에 따라 접근할 수 있는 범위**를 결정합니다.

### 🔸 종류

| 스코프 종류     | 설명                                         |
|------------------|----------------------------------------------|
| 전역 스코프      | 코드 전체에서 접근 가능                       |
| 함수 스코프      | 함수 내부에서만 접근 가능 (`var`)             |
| 블록 스코프      | `{}` 내부에서만 접근 가능 (`let`, `const`)    |
| 렉시컬 스코프    | 선언된 위치 기준으로 결정되는 스코프 구조     |

---

### 📌 전역 스코프 vs 함수 스코프

```js
let globalVar = "global";

function test() {
  let localVar = "local";
  console.log(globalVar); // 접근 가능
  console.log(localVar);  // 접근 가능
}

console.log(globalVar);  // 접근 가능
console.log(localVar);   // ReferenceError
```

---

### 📌 블록 스코프 (`let`, `const`)

```js
{
  let a = 1;
  const b = 2;
}
console.log(a); // ReferenceError
console.log(b); // ReferenceError
```

> `var`는 블록을 무시하고 함수 전체에서 접근 가능 → 사용 지양

---

### 📌 렉시컬 스코프 (Lexical Scope)

**함수가 어디서 호출되었는지가 아니라, 어디서 정의되었는지가 중요**

```js
let x = 10;

function outer() {
  let x = 20;
  function inner() {
    console.log(x); // 20 ← 정의된 위치의 스코프를 참조
  }
  inner();
}

outer();
```

---

## ✅ 2. 클로저(Closure)란?

> "클로저는 함수가 자신이 선언된 렉시컬 환경을 **기억하는 기능**이다."

함수가 반환된 후에도, **자신이 선언될 당시의 스코프를 기억**하여  
그 내부 변수에 접근할 수 있는 함수 = **클로저**

---

### 🔹 예제 1: 기본 클로저

```js
function outer() {
  let count = 0;

  function inner() {
    count++;
    return count;
  }

  return inner;
}

const counter = outer(); // outer 실행 → inner 반환
console.log(counter());  // 1
console.log(counter());  // 2
console.log(counter());  // 3
```

> `outer()`는 이미 끝났지만, `count` 변수는 `inner()`가 계속 기억하고 있음 → 클로저!

---

### 🔹 예제 2: 클로저를 이용한 정보 은닉

```js
function createUser(name) {
  let privateName = name;

  return {
    getName: function() {
      return privateName;
    },
    setName: function(newName) {
      privateName = newName;
    }
  };
}

const user = createUser("Alice");
console.log(user.getName()); // Alice
user.setName("Bob");
console.log(user.getName()); // Bob
```

> 외부에서 `privateName`에 직접 접근은 불가능하지만, getter/setter를 통해 제어 가능 → 캡슐화 구현 가능

---

## 📦 클로저의 특징

- 함수가 종료되어도 **내부 변수 상태 유지 가능**
- **정보 은닉**, **상태 유지**, **커링(Currying)** 등에 사용
- 클로저가 많아지면 **메모리 누수** 가능성도 있음 → 불필요한 참조는 해제 필요

---

## ⚠️ 주의: 반복문과 클로저

```js
for (var i = 0; i < 3; i++) {
  setTimeout(function () {
    console.log(i);
  }, 100);
}
// 3, 3, 3
```

> 모든 `setTimeout` 콜백은 동일한 `i`를 참조

### ✅ 해결 방법: `let` 사용

```js
for (let i = 0; i < 3; i++) {
  setTimeout(function () {
    console.log(i);
  }, 100);
}
// 0, 1, 2
```

> `let`은 블록 스코프이므로 각 반복마다 `i`가 새로 바인딩됨

---

## ✅ 스코프와 클로저 정리

| 항목             | 설명                                             |
|------------------|--------------------------------------------------|
| 스코프           | 변수에 접근 가능한 유효 범위                     |
| 렉시컬 스코프     | 선언된 위치를 기준으로 스코프 결정               |
| 클로저           | 함수가 자신이 선언된 스코프를 기억하는 특성     |
| 주 용도          | 상태 유지, 캡슐화, 커링                         |

---

## 📌 실전 활용: 버튼 클릭 수 카운트

```js
function createCounterButton() {
  let count = 0;

  return function () {
    count++;
    console.log(`Clicked ${count} times`);
  };
}

const clickHandler = createCounterButton();

document.querySelector("button").addEventListener("click", clickHandler);
```

---

## 🧠 마무리

- 스코프를 이해하면 변수 충돌과 의도치 않은 참조를 피할 수 있습니다.
- 클로저는 **함수를 함수 안에서 만들고 반환**할 때 자연스럽게 생깁니다.
- 자바스크립트의 핵심 개념 중 하나이며, **비동기 처리**, **모듈화**, **라이브러리 구현** 등에 필수적입니다.