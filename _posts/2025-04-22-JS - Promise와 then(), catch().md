---
layout: post
title: JavaScript - Promise와 then(), catch()
date: 2025-04-22 20:20:23 +0900
category: JavaScript
---
# Promise란? — “미래에 결정될 값(또는 실패 이유)을 담는 상자”

```js
const p = new Promise((resolve, reject) => {
  // executor(실행자)는 즉시 동기 호출된다
  try {
    // 비동기 작업 시작
    const ok = true;
    if (ok) resolve("작업 성공!");
    else reject(new Error("작업 실패!"));
  } catch (e) {
    reject(e);
  }
});
```

- **상태(state)**: `pending` → `fulfilled`(성공) 또는 `rejected`(실패)로 **단 한 번** 전이되며, 이후 **불변**.
- **executor**는 **동기적으로** 실행된다. 내부에서 비동기 API를 호출해 **나중에** `resolve/reject`를 트리거한다.
- **thenable 동화(assimilation)**: `resolve(x)`에서 `x`가 Promise-같은 객체(thenable)이면 **그 결과를 따라간다**.

---

# `.then()` — 성공/실패 핸들러와 “평탄화(flattening)”

```js
p.then(
  value => { /* fulfilled 시 호출 */ },
  reason => { /* rejected 시 호출(잘 쓰지 않음) */ }
);
```

- 반환값: **항상 새로운 Promise**.
- `value` 핸들러가 **값**을 반환 → 그 값으로 **fulfilled**.
- `value` 핸들러가 **Promise/thenable** 반환 → **평탄화**되어 그 결과를 **연결**.
- 핸들러에서 **예외 throw** → **rejected**로 전파.

> 실패 처리는 보통 두 번째 인자 대신 **`.catch()`로 분리**하는 것을 권장(가독성/흐름 제어).

---

# `.catch()` — 실패 전용 핸들러(전파/복구/재throw)

```js
p
  .then(doWork)
  .catch(err => {
    // 복구하거나
    if (canRecover(err)) return fallbackValue();
    // 못하면 재throw하여 다음 catch에 맡긴다
    throw err;
  });
```

- `.catch(onRejected)`는 `.then(undefined, onRejected)`와 동일.
- **어디에서 발생한 에러든** 체인 아래의 첫 `.catch`가 **가로챈다**.
- `.catch` 내부에서 **값**을 반환하면 **정상 흐름으로 복귀**한다(그 값으로 fulfilled).

---

# `.finally()` — 성공/실패 무관 “청소 영역”

```js
doSomething()
  .then(result => use(result))
  .catch(err => handle(err))
  .finally(() => teardown());
```

- 반환: **원래 결과**를 그대로 전달.
  (단, `finally`에서 예외가 발생하면 그 예외로 대체되어 reject)

---

# 체이닝 & 에러 전파 규칙 — “항상 Promise를 반환한다”

```js
login()
  .then(user => getProfile(user.id))      // Promise 반환 → 평탄화
  .then(profile => render(profile))        // 값 반환 → 다음 then에 값
  .catch(err => {                          // 위에서 터진 모든 에러 수거
    console.error("에러:", err);
    return emptyProfile();                 // 복구 후 정상 흐름 복귀
  })
  .then(profile => use(profile))           // catch 이후도 계속 진행
  .catch(finalErr => alert(finalErr));
```

핵심 규칙 요약:
- **값 반환** → 다음 then에 값 전달(fulfilled)
- **Promise 반환** → 그 Promise 결과를 따름(평탄화)
- **throw** → 다음 catch로 이동(rejected)
- **catch에서 값 반환** → 정상 복귀
- **catch에서 재throw** → 다음 catch로 전파

---

# 실전 예제 — 모의 비동기 API

```js
function asyncJob() {
  return new Promise((resolve) => {
    setTimeout(() => resolve("완료됨!"), 1000);
  });
}

asyncJob()
  .then(msg => {
    console.log(msg);     // 1초 뒤 "완료됨!"
    return "다음 작업";
  })
  .then(next => console.log(next))
  .catch(err => console.error("에러:", err))
  .finally(() => console.log("항상 실행"));
```

---

# 중첩 then ❌ vs 체이닝 ✅

```js
// ❌ 중첩(콜백 지옥과 유사)
login().then(user => {
  getProfile(user.id).then(profile => {
    console.log(profile);
  });
});

// ✅ 체이닝
login()
  .then(user => getProfile(user.id))
  .then(profile => console.log(profile))
  .catch(console.error);
```

---

# Promise 조합기(Combinators) — 병렬/경쟁/종료 대기

## `Promise.all(iterable)`

- 모든 Promise가 **성공**해야 전체가 성공. 하나라도 실패 → 즉시 reject.
- 결과는 **입력 순서**를 보장한 배열.

```js
const pa = fetchA();
const pb = fetchB();
Promise.all([pa, pb])
  .then(([a, b]) => useBoth(a, b))
  .catch(err => console.error("둘 중 하나 실패:", err));
```

## `Promise.allSettled(iterable)`

- 모두 **정착(settled)**할 때까지 기다림(성공/실패 포함).
- 각 항목 `{ status: "fulfilled", value }` 또는 `{ status: "rejected", reason }`.

```js
Promise.allSettled([fetchA(), fetchB()])
  .then(results => {
    const oks = results.filter(r => r.status === "fulfilled").map(r => r.value);
    const errs = results.filter(r => r.status === "rejected").map(r => r.reason);
    console.log(oks, errs);
  });
```

## `Promise.race(iterable)`

- **가장 먼저 정착**한 Promise의 결과를 그대로 반영.

```js
Promise.race([slow(), timeoutAfter(1000)])
  .then(console.log)
  .catch(console.error);
```

## `Promise.any(iterable)`

- **첫 성공**을 반환. 모든 항목이 실패하면 `AggregateError`로 reject.

```js
Promise.any([tryPrimary(), trySecondary()])
  .then(result => console.log("최초 성공:", result))
  .catch(e => console.error("전부 실패", e.errors));
```

---

# 직렬/병렬/동시성 제어 패턴

## 직렬(순차) 실행

```js
const tasks = [t1, t2, t3]; // () => Promise
tasks.reduce((p, task) => p.then(task), Promise.resolve());
```

## 제한된 동시성(concurrency) — 간단한 세마포어

```js
function withLimit(limit, tasks) {
  const queue = [...tasks];         // () => Promise
  let running = 0;
  const results = [];
  return new Promise((resolve, reject) => {
    const next = () => {
      if (!queue.length && running === 0) return resolve(results);
      while (running < limit && queue.length) {
        const i = results.length;
        const task = queue.shift();
        running++;
        Promise.resolve()
          .then(task)
          .then(r => { results[i] = { status: 'fulfilled', value: r }; })
          .catch(e => { results[i] = { status: 'rejected', reason: e }; })
          .finally(() => { running--; next(); });
      }
    };
    next();
  });
}
```

---

# 타임아웃/취소/리트라이 — 견고한 네트워킹

## 타임아웃 래퍼

```js
const timeout = (ms, name="timeout") =>
  new Promise((_, reject) => setTimeout(() => reject(new Error(name)), ms));

Promise.race([fetch(url), timeout(5000, "5s exceeded")])
  .then(handle)
  .catch(console.error);
```

## AbortController로 취소(fetch)

```js
function fetchWithTimeout(url, { ms = 5000 } = {}) {
  const ac = new AbortController();
  const t = setTimeout(() => ac.abort(), ms);
  return fetch(url, { signal: ac.signal })
    .finally(() => clearTimeout(t));
}

fetchWithTimeout("/api/data", { ms: 3000 })
  .then(r => r.json())
  .catch(err => {
    if (err.name === "AbortError") console.warn("취소됨");
    else console.error(err);
  });
```

## 리트라이 + 지수 백오프

```js
async function retry(fn, { attempts = 3, baseMs = 300 } = {}) {
  let last;
  for (let i = 0; i < attempts; i++) {
    try { return await fn(); }
    catch (e) {
      last = e;
      const backoff = baseMs * 2 ** i;
      await new Promise(res => setTimeout(res, backoff));
    }
  }
  throw last;
}
```

---

# 콜백 API를 Promise로 — promisify

```js
const fs = require("fs");
const readFileP = (path, enc="utf8") =>
  new Promise((resolve, reject) =>
    fs.readFile(path, enc, (err, data) => err ? reject(err) : resolve(data)));

readFileP("a.txt")
  .then(console.log)
  .catch(console.error);
```

---

# 마이크로태스크 큐와 이벤트 루프 — 실행 타이밍 이해

- `then/catch/finally` 콜백은 **마이크로태스크 큐**에 들어가며,
  현재 **콜스택**이 비고 **매크로태스크**(timer, I/O 등)보다 **우선** 처리된다.

```js
Promise.resolve().then(() => console.log("microtask"));
setTimeout(() => console.log("macrotask"), 0);
console.log("sync");
// 출력 순서: "sync" → "microtask" → "macrotask"
```

---

# `unhandledrejection` — 놓친 에러 잡기

브라우저:
```js
window.addEventListener("unhandledrejection", e => {
  console.error("Unhandled:", e.reason);
});
```

Node.js:
```js
process.on("unhandledRejection", (reason, promise) => {
  console.error("Unhandled:", reason);
});
```

> 체인 **마지막에는 반드시 `.catch()`** 를 붙여 누락된 에러를 방지하자.

---

# 안티패턴 & 베스트 프랙티스

## “Promise 생성자 오남용(new Promise 안에서 또 Promise)”

```js
// ❌ 불필요한 포장
function foo() {
  return new Promise((resolve, reject) => {
    asyncOp().then(resolve, reject);
  });
}
// ✅ 그대로 반환(평탄화)
function foo() {
  return asyncOp();
}
```

## `then` 안에서 `return` 누락

```js
// ❌ 다음 then에 값 전달 안 됨
get().then(v => { process(v); });

// ✅ 반드시 값/Promise 반환
get().then(v => process(v));
```

## `then`/`catch`와 `async/await` 혼용 시 흐름 분리

- 한 체인에서는 **하나의 스타일**을 유지하라. 필요 시 가장 바깥에서 변환.

## 중첩 then

- 중첩하지 말고 **체이닝**으로 펼쳐라(가독성, 에러 전파 일관성).

## 무한 대기 방지

- 네트워크/IO에는 **타임아웃**이나 **AbortController**를 결합하라.

---

# then ↔ async/await 대응 관계(응용)

```js
// then/catch
task()
  .then(x => step1(x))
  .then(y => step2(y))
  .catch(err => recover(err))
  .then(z => done(z))
  .catch(final);

// async/await 동등한 흐름
async function flow() {
  try {
    const x = await task();
    const y = await step1(x);
    const z = await step2(y);
    const r = await z;
    return done(r);
  } catch (e) {
    try {
      const rec = await recover(e);
      return done(rec);
    } catch (e2) {
      return final(e2);
    }
  }
}
```

> `await`은 **Promise의 fulfilled 값을 꺼내는 문법 설탕**이다.
> 에러는 `try/catch`로 동기처럼 다룬다.

---

# 실전 시나리오 — 사용자/게시글 가져오기

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

# 패턴 모음 — 실무에서 바로 쓰는 스니펫

## 조건부 체이닝(가드)

```js
fetchConfig()
  .then(cfg => cfg.enabled ? run(cfg) : skip())
  .catch(log);
```

## 캐싱(메모이즈) Promise

```js
const loadOnce = (() => {
  let memo;
  return () => memo ?? (memo = fetchData());
})();
```

## 결과/오류 튜플화(try/catch 대체)

```js
const to = p => p.then(v => [null, v]).catch(e => [e]);
const [err, data] = await to(fetch(...));
```

## 파이프라인(체이닝으로 데이터 흐름 구성)

```js
getA()
  .then(a => toB(a))
  .then(b => toC(b))
  .then(c => finalize(c))
  .catch(handle);
```

---

# 디버깅 팁

- 체인의 각 단계에 **이름 있는 함수**를 사용하면 스택과 로깅이 명확해진다.
- `.catch()`를 **가까운 곳**과 **끝**에 각각 두어,
  1) 단계별 복구, 2) 최종 안전망 역할을 병행할 수 있다.
- DevTools의 **Async stack traces** 옵션을 켜면 비동기 경로가 연결되어 추적이 쉬워진다.

---

# 체크리스트 요약

- [ ] 핸들러는 **값/Promise 반환**, 실패는 **throw**.
- [ ] 에러는 아래 **첫 `.catch()`** 가 받는다(복구/재throw 선택).
- [ ] `.finally()`는 ** 청소만**, 결과를 바꾸지 말자(예외시만 예외로 대체).
- [ ] **조합기**: `all`(전부 성공), `allSettled`(전부 종료), `race`(가장 빠른 종료), `any`(첫 성공).
- [ ] **타임아웃/취소/리트라이**를 네트워크에 결합해 견고성 확보.
- [ ] **마이크로태스크 큐** 동작을 이해하고 로딩 표시/순서 문제를 교정.
- [ ] 체인의 **마지막에는 `.catch()`**.

---

# 미니 퀴즈

```js
// Q1: 아래의 출력 순서는?
Promise.resolve().then(() => console.log("A"));
setTimeout(() => console.log("B"), 0);
console.log("C");

// Q2: 무엇이 출력될까?
Promise.resolve("X")
  .then(v => { throw new Error("boom"); })
  .then(() => "Y")
  .catch(e => "Z")
  .then(console.log);

// Q3: all vs any 차이 설명 및 다음 코드 결과?
Promise.any([Promise.reject("e1"), Promise.resolve(42), Promise.reject("e2")])
  .then(console.log);

// Q4: finally가 결과를 바꾸는가?
Promise.resolve(1)
  .finally(() => 99)
  .then(console.log);

// Q5: 왜 아래는 냄새가 나는가? 개선하라.
new Promise((resolve, reject) => {
  asyncOp().then(resolve, reject);
});
```

**해설 힌트**
- Q1: `C` → `A` → `B` (sync → microtask → macrotask).
- Q2: `boom`으로 reject → catch에서 `"Z"` 반환 → 콘솔 `"Z"`.
- Q3: 첫 성공 `42`.
- Q4: finally 반환값은 무시 → 최종 `1`.
- Q5: 불필요한 포장. `return asyncOp();`로 충분.

---

## 결론

Promise는 “**값이 나중에 결정되는 제어 흐름**”을 **선언적으로 모델링**한다.
핵심 규칙(평탄화, 에러 전파, finally, 조합기, 마이크로태스크)을 정확히 이해하고
타임아웃/취소/리트라이 같은 **신뢰성 패턴**을 결합하면,
복잡한 비동기 요구사항도 **가독성, 안정성, 성능**을 모두 잡을 수 있다.
