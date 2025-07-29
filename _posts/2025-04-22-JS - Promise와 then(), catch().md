---
layout: post
title: JavaScript - Promise와 then(), catch()
date: 2025-04-22 20:20:23 +0900
category: JavaScript
---
# Promise와 `.then()`, `.catch()` 완전 정복

JavaScript에서 비동기 처리를 다루는 대표적인 방식이 바로 **`Promise`**입니다.  
과거 콜백(callback) 지옥의 대안으로 등장한 Promise는 더 명확한 비동기 흐름과 에러 처리를 제공합니다.

---

## ✅ 1. Promise란?

> **미래에 완료될 수도 있는 비동기 작업의 결과**를 나타내는 객체

```js
const promise = new Promise((resolve, reject) => {
  // 비동기 작업 수행
  const success = true;

  if (success) {
    resolve("작업 성공!");
  } else {
    reject("작업 실패!");
  }
});
```

### 생성자 구조

```js
new Promise((resolve, reject) => {
  // 비동기 로직
});
```

- `resolve(value)` → 성공 시 호출
- `reject(reason)` → 실패 시 호출

---

## ✅ 2. `.then()` – 성공 처리

`resolve`로 전달된 값을 받아 **성공 콜백 실행**

```js
promise
  .then(result => {
    console.log(result); // "작업 성공!"
  });
```

---

## ✅ 3. `.catch()` – 에러 처리

`reject` 또는 내부 에러가 발생했을 때 호출됨

```js
promise
  .then(result => {
    console.log(result);
  })
  .catch(error => {
    console.error(error); // "작업 실패!" 또는 에러 메시지
  });
```

---

## ✅ 4. 체이닝 (Chaining)

`.then()`은 **Promise를 반환**하므로 연속해서 연결할 수 있음

```js
getUser()
  .then(user => getPosts(user.id))
  .then(posts => display(posts))
  .catch(err => console.error("에러 발생:", err));
```

> 각 `then`은 이전 결과를 다음으로 전달합니다.

---

## ✅ 5. 예제: 비동기 API 흉내

```js
function asyncJob() {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      resolve("완료됨!");
    }, 1000);
  });
}

asyncJob()
  .then(msg => {
    console.log(msg); // 1초 뒤 "완료됨!"
    return "다음 작업";
  })
  .then(msg2 => {
    console.log(msg2); // "다음 작업"
  })
  .catch(err => {
    console.error("에러:", err);
  });
```

---

## ✅ 6. 중첩된 `.then()` ❌ vs 체이닝 ✅

### ❌ 중첩 사용 (가독성 ↓)

```js
login().then(user => {
  getProfile(user.id).then(profile => {
    console.log(profile);
  });
});
```

### ✅ 체이닝 사용

```js
login()
  .then(user => getProfile(user.id))
  .then(profile => console.log(profile))
  .catch(err => console.error(err));
```

---

## ✅ 7. 예외 처리

- `throw`로 예외를 발생시키면 `.catch()`에서 잡힘

```js
new Promise((resolve, reject) => {
  throw new Error("예외 발생!");
})
.catch(err => {
  console.error(err.message); // "예외 발생!"
});
```

- `.then()` 내부에서 발생한 에러도 `.catch()`로 전달됨

```js
Promise.resolve("시작")
  .then(() => {
    throw new Error("중간 에러");
  })
  .catch(err => console.error("잡힘:", err.message));
```

---

## ✅ 8. `finally()` – 성공/실패 무관하게 실행

```js
doSomething()
  .then(result => console.log("성공:", result))
  .catch(err => console.error("실패:", err))
  .finally(() => console.log("항상 실행됨"));
```

> 리소스 정리, 로딩 상태 해제 등에 유용

---

## ✅ 9. 상태 변화 요약

| 상태          | 설명                            | 처리 메서드      |
|---------------|----------------------------------|------------------|
| Pending       | 대기 중 (초기 상태)             | -                |
| Fulfilled     | 성공 완료                        | `.then()`        |
| Rejected      | 실패                             | `.catch()`       |
| 완료(무관)    | 성공이든 실패든 마무리됨        | `.finally()`     |

---

## ✅ 10. 실전 예시: 사용자 정보 가져오기

```js
function getUser() {
  return new Promise((resolve) => {
    setTimeout(() => resolve({ id: 1, name: "Alice" }), 500);
  });
}

function getPosts(userId) {
  return new Promise((resolve) => {
    setTimeout(() => resolve([`Post1 for ${userId}`, `Post2 for ${userId}`]), 500);
  });
}

getUser()
  .then(user => {
    console.log("User:", user);
    return getPosts(user.id);
  })
  .then(posts => {
    console.log("Posts:", posts);
  })
  .catch(err => {
    console.error("Error:", err);
  })
  .finally(() => {
    console.log("요청 완료");
  });
```

---

## ✅ 마무리

- `Promise`는 비동기 처리를 체계적으로 제어할 수 있게 해주는 객체
- `.then()`은 성공, `.catch()`는 실패, `.finally()`는 마무리 처리용
- 콜백 지옥을 피하고 가독성을 높이기 위해 체이닝을 적극 활용
- ES2017의 `async/await` 문법과 함께 사용하면 더욱 직관적