---
layout: post
title: JavaScript - async, await
date: 2025-04-28 19:20:23 +0900
category: JavaScript
---
# async / await 제대로 이해하기

`async / await`는 자바스크립트에서 **비동기 코드를 동기 코드처럼 작성**할 수 있게 해주는 문법입니다.  
`Promise`의 `.then()` 체이닝을 더 깔끔하게 대체하며, **가독성과 예외 처리 측면에서 훨씬 유리**합니다.

---

## ✅ 1. async 함수란?

> 함수 앞에 `async` 키워드를 붙이면 항상 **Promise를 반환**하는 **비동기 함수**가 됩니다.

```js
async function greet() {
  return "Hello";
}

greet().then(msg => console.log(msg)); // "Hello"
```

- 내부에서 `return`을 사용하면 자동으로 `Promise.resolve()`로 감싸짐
- 예외를 던지면 `Promise.reject()`로 처리됨

---

## ✅ 2. await란?

> `await`는 **Promise가 처리될 때까지 기다렸다가** 결과값을 반환합니다.

```js
async function getData() {
  const res = await fetch("https://api.example.com/data");
  const json = await res.json();
  console.log(json);
}
```

- 반드시 `async` 함수 안에서만 사용 가능
- `await`은 Promise가 완료될 때까지 **비동기적으로 기다린 후 결과를 반환**

---

## ✅ 3. 예제: 콜백 → Promise → async/await 비교

### 🔸 콜백 기반

```js
getUser(function(user) {
  getProfile(user.id, function(profile) {
    console.log(profile);
  });
});
```

### 🔸 Promise 체이닝

```js
getUser()
  .then(user => getProfile(user.id))
  .then(profile => console.log(profile));
```

### 🔸 async/await

```js
async function showProfile() {
  const user = await getUser();
  const profile = await getProfile(user.id);
  console.log(profile);
}

showProfile();
```

> ✔️ 가장 **간결하고 가독성 좋은 형태**가 `async/await`입니다.

---

## ✅ 4. 예외 처리 – try/catch

```js
async function loadData() {
  try {
    const user = await getUser();
    const posts = await getPosts(user.id);
    console.log(posts);
  } catch (err) {
    console.error("에러 발생:", err);
  }
}
```

- `try/catch` 블록을 사용해 **동기 코드처럼 에러 처리 가능**
- `await`된 Promise가 `reject`되면 `catch`로 넘어감

---

## ✅ 5. 여러 비동기 처리 병렬 실행

### 🔹 잘못된 방식: 순차 실행 (느림)

```js
const a = await fetchA(); // 1초
const b = await fetchB(); // 1초 → 총 2초
```

### 🔹 올바른 병렬 실행

```js
const [a, b] = await Promise.all([fetchA(), fetchB()]); // 병렬로 1초
```

---

## ✅ 6. async 함수 내부 흐름 요약

```js
async function process() {
  const res1 = await step1(); // Promise 완료까지 대기
  const res2 = await step2(res1);
  return res2; // 자동으로 Promise로 래핑됨
}
```

- 내부에서 `await`는 실행을 일시 중단하고, Promise 완료 후 다음 줄로 넘어감
- 전체 async 함수는 `Promise`로 감싸진 결과를 반환

---

## ✅ 7. await은 실제로 "동기처럼 보이지만 비동기"

```js
console.log("1");

setTimeout(() => console.log("2"), 0);

(async () => {
  await Promise.resolve();
  console.log("3");
})();

console.log("4");

// 출력 순서: 1 → 4 → 3 → 2
```

> ✔️ `await`는 **마이크로태스크 큐**에 들어가기 때문에 `setTimeout`보다 먼저 실행됨

---

## ✅ 8. async/await vs then/catch 비교

| 항목         | then/catch                           | async/await                            |
|--------------|--------------------------------------|----------------------------------------|
| 문법 스타일   | 체이닝 기반                          | 동기 코드처럼 작성                     |
| 에러 처리     | `.catch()` 사용                      | `try/catch` 사용                       |
| 가독성        | 복잡한 체인일수록 어려움            | 매우 좋음                              |
| 디버깅        | 스택 트레이스 어려움                | 브레이크포인트 설정 용이               |
| 병렬 처리     | `Promise.all()` 사용                | 동일하게 사용 (`await Promise.all()`) |

---

## ✅ 9. async 함수는 항상 Promise 반환

```js
async function getValue() {
  return 123;
}

getValue().then(console.log); // 123
```

---

## ✅ 10. 주의사항

- `await`는 반드시 **`async` 함수 내부**에서만 사용 가능
- `forEach` 안에서는 `await`가 **의미 없음** → `for...of` 사용 권장
  ```js
  const list = [1, 2, 3];
  for (const item of list) {
    await doSomething(item);
  }
  ```

- 동시에 실행할 수 있는 작업은 `Promise.all()`로 묶는 것이 성능상 이점 있음

---

## ✅ 마무리

- `async/await`는 Promise를 **더 깔끔하고 동기식처럼 작성**할 수 있게 도와줍니다.
- `try/catch`로 에러 처리가 편리하며, 유지보수성과 디버깅에도 유리합니다.
- `Promise.all`, `await`, `try/catch`, `for...of` 등과 함께 쓰는 패턴을 익히면 실무에서 매우 강력한 도구가 됩니다.