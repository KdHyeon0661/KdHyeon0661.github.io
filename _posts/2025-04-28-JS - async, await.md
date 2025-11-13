---
layout: post
title: JavaScript - async, await
date: 2025-04-28 19:20:23 +0900
category: JavaScript
---
# async / await

## 1. async 함수란?

### 1.1 정의와 반환 규칙
- 함수 앞에 `async`를 붙이면 **항상 `Promise`를 반환**합니다.
- `return 값`은 자동으로 `Promise.resolve(값)`에 **포장**됩니다.
- `throw 에러`는 자동으로 `Promise.reject(에러)`가 됩니다.

```js
async function greet() {
  return "Hello"; // => Promise.resolve("Hello")
}
greet().then(console.log); // "Hello"
```

```js
async function fail() {
  throw new Error("boom"); // => Promise.reject(Error)
}
fail().catch(e => console.log(e.message)); // "boom"
```

### 1.2 `thenable` 지원
- `await`는 진짜 `Promise`뿐 아니라 **then 메서드가 있는 객체(thenable)** 도 기다립니다.

```js
const thenable = { then: (res) => setTimeout(() => res(42), 10) };
async function demo() {
  const v = await thenable;
  console.log(v); // 42
}
demo();
```

---

## 2. await의 의미와 스케줄링

### 2.1 await의 동작
- `await P`는 **해당 async 함수의 실행만** 중단하고, `P`가 해결되면 **다음 줄**로 이어갑니다.
- **메인 스레드를 블로킹하지 않습니다.** (이벤트 루프가 다른 작업을 계속 처리)

```js
async function getData() {
  const res = await fetch("/api");
  const json = await res.json();
  return json;
}
```

### 2.2 마이크로태스크 순서
- `await` 뒤 코드는 **마이크로태스크 큐**에서 실행됩니다.
- `Promise.then` 콜백과 같은 우선순위를 가집니다.

```js
console.log("1");
setTimeout(() => console.log("2"), 0);
(async () => {
  await Promise.resolve();
  console.log("3");
})();
console.log("4");
// 1 → 4 → 3 → 2
```

---

## 3. 콜백 → Promise → async/await 변환 감각

### 3.1 콜백 기반
```js
getUser((uErr, user) => {
  if (uErr) return console.error(uErr);
  getProfile(user.id, (pErr, profile) => {
    if (pErr) return console.error(pErr);
    console.log(profile);
  });
});
```

### 3.2 Promise 체인
```js
getUserP()
  .then(user => getProfileP(user.id))
  .then(profile => console.log(profile))
  .catch(console.error);
```

### 3.3 async/await
```js
async function showProfile() {
  try {
    const user = await getUserP();
    const profile = await getProfileP(user.id);
    console.log(profile);
  } catch (e) {
    console.error(e);
  }
}
showProfile();
```

> **가독성·제어 흐름·예외 처리** 측면에서 async/await이 가장 직관적입니다.

---

## 4. 예외 처리 패턴 (try/catch/finally)

### 4.1 전체 래핑 vs. 부분 래핑
```js
// 전체 래핑: 간단하지만 에러 지점 파악이 어렵기도
async function loadAll() {
  try {
    const a = await A();
    const b = await B(a);
    return await C(b);
  } catch (e) {
    // A/B/C 어디에서 실패했는지 모호
    throw e;
  }
}

// 부분 래핑: 원인 분리
async function loadAll2() {
  const a = await A().catch(e => { e.step = "A"; throw e; });
  const b = await B(a).catch(e => { e.step = "B"; throw e; });
  return C(b).catch(e => { e.step = "C"; throw e; });
}
```

### 4.2 finally로 정리 보장
```js
async function withLock(task) {
  let lock;
  try {
    lock = await acquire();
    return await task();
  } finally {
    if (lock) await lock.release();
  }
}
```

---

## 5. 동시성(병렬) 패턴

### 5.1 직렬 vs 병렬
```js
// ❌ 직렬(느림)
const a = await fetchA(); // 1s
const b = await fetchB(); // 또 1s → 총 2s

// ✅ 병렬
const [a2, b2] = await Promise.all([fetchA(), fetchB()]); // 1s 내 완료
```

### 5.2 집합 대기 유틸
```js
// 모두 성공해야 결과: 하나라도 실패하면 reject
await Promise.all([p1, p2, p3]);

// 모두의 결과를 안전하게 수집(성공/실패 포함)
const results = await Promise.allSettled([p1, p2]);

// 하나라도 먼저 성공하면 resolve
const fastest = await Promise.any([p1, p2, p3]);

// 가장 먼저 settle(성공/실패)되는 것 반환
const first = await Promise.race([p1, p2]);
```

### 5.3 태스크 수 제한(세마포어)
```js
function limit(concurrency) {
  let running = 0, queue = [];
  const run = async (fn, resolve, reject) => {
    running++;
    try { resolve(await fn()); }
    catch (e) { reject(e); }
    finally {
      running--;
      if (queue.length) queue.shift()();
    }
  };
  return fn => new Promise((resolve, reject) => {
    const task = () => run(fn, resolve, reject);
    running < concurrency ? task() : queue.push(task);
  });
}

// 사용
const lim2 = limit(2);
const jobs = urls.map(u => lim2(() => fetch(u).then(r => r.text())));
const texts = await Promise.all(jobs);
```

---

## 6. 타임아웃·취소(AbortController)

### 6.1 fetch 타임아웃 + 취소
```js
async function fetchWithTimeout(url, { ms = 3000, ...opts } = {}) {
  const ac = new AbortController();
  const id = setTimeout(() => ac.abort(new Error("timeout")), ms);
  try {
    const res = await fetch(url, { ...opts, signal: ac.signal });
    return res;
  } finally {
    clearTimeout(id);
  }
}
```

### 6.2 Promise.race로 타임아웃
```js
function withTimeout(p, ms = 3000) {
  const t = new Promise((_, rej) =>
    setTimeout(() => rej(new Error("timeout")), ms));
  return Promise.race([p, t]);
}
```

---

## 7. 재시도(리트라이)·백오프

### 7.1 지수 백오프(기본식)
- 대기 시간: $$ t_k = t_0 \cdot 2^{k} $$
- **지터**(무작위성)로 동시 재시도 폭주 예방:
  $$ t_k = U(0,1)\cdot t_0 \cdot 2^{k} $$

```js
async function retry(fn, { attempts = 3, base = 300, jitter = true } = {}) {
  let last;
  for (let k = 0; k < attempts; k++) {
    try { return await fn(); }
    catch (e) {
      last = e;
      const delay = base * (2 ** k);
      const sleep = jitter ? Math.random() * delay : delay;
      await new Promise(r => setTimeout(r, sleep));
    }
  }
  throw last;
}
```

---

## 8. forEach와 await의 함정

### 8.1 `forEach`는 `await`를 못 기다립니다
```js
// ❌ 완료를 기다리지 않음
arr.forEach(async v => {
  await doWork(v);
});
console.log("끝"); // 먼저 출력될 수 있음
```

### 8.2 `for...of` 또는 `map → Promise.all`
```js
// 직렬
for (const v of arr) {
  await doWork(v);
}

// 병렬
await Promise.all(arr.map(v => doWork(v)));
```

---

## 9. 흔한 실수와 디버깅 팁

### 9.1 `await` 누락
```js
async function f() {
  doAsync(); // ❌ 잊음 → 실패가 밖으로 전파 안 됨
}
```

### 9.2 `return` 누락으로 의도치 않은 `undefined`
```js
async function g() {
  const x = await calc();
  // ❌ return x를 잊으면 호출측은 undefined를 받음
  return x; // ✅
}
```

### 9.3 전역 미처리 거부 감지
```js
// 브라우저
window.addEventListener("unhandledrejection", e => {
  console.warn("Unhandled:", e.reason);
});

// Node
process.on("unhandledRejection", (reason) => {
  console.warn("Unhandled:", reason);
});
```

### 9.4 CPU 바운드 작업은 워커로
- `await`는 **I/O 대기**에 적합. 무거운 계산은 **Web Worker/Worker Threads**로 분리.

---

## 10. 실전 유틸 모음

### 10.1 sleep
```js
const sleep = ms => new Promise(r => setTimeout(r, ms));
```

### 10.2 `promisify` (콜백 → Promise)
```js
function promisify(fn){
  return (...args) => new Promise((res, rej) =>
    fn(...args, (err, val) => err ? rej(err) : res(val)));
}
```

### 10.3 withTimeout + withRetry 조합
```js
async function robust(fn) {
  return retry(() => withTimeout(fn(), 5000), { attempts: 4, base: 400 });
}
```

---

## 11. 테스트에서의 async/await

### 11.1 Jest 예시
```js
test("fetches user", async () => {
  const user = await getUser(1);
  expect(user.name).toBe("Alice");
});
```

### 11.2 병렬 테스트 시 주의
- 외부 리소스/포트 사용 시 **충돌 방지**(고유 리소스 할당 또는 직렬화).

---

## 12. 미니 프로젝트: 목록→상세 동시성 최적화

### 12.1 요구
1) `/posts`로 ID 목록 로드
2) 각 ID 상세 `/posts/:id` 병렬 로드
3) 일부 실패해도 전체 결과 목록을 보여주되, 실패 항목은 표시

```js
async function loadPosts() {
  const list = await (await fetch("/posts")).json(); // [{id:1}, {id:2}, ...]
  const tasks = list.map(p => fetch(`/posts/${p.id}`).then(r => r.json()));
  const settled = await Promise.allSettled(tasks);
  return settled.map((r, i) => r.status === "fulfilled"
                     ? { ok: true, data: r.value }
                     : { ok: false, id: list[i].id, reason: r.reason });
}
```

---

## 13. 체크리스트

- [ ] **직렬 vs 병렬**: 독립 작업은 `Promise.all`
- [ ] **에러 범위**: `try/catch` 위치를 최소·명확하게
- [ ] **타임아웃/취소**: `AbortController`/`race`/withTimeout
- [ ] **리트라이**: 지수 백오프 + 지터
  $$ t_k = U(0,1)\cdot t_0 \cdot 2^{k} $$
- [ ] **루프**: `forEach` 대신 `for...of` 또는 `map→Promise.all`
- [ ] **전역 핸들러**: `unhandledrejection` 배선
- [ ] **CPU 바운드**: 워커 사용

---

## 14. 요약

- `async`는 **Promise를 반환**, `await`는 **Promise 완료까지 비동기 대기**합니다.
- `try/catch/finally`로 **동기 수준의 예외·정리**를 표현할 수 있습니다.
- **동시성 최적화**(All/AllSettled/Any/Race), **타임아웃·취소**, **재시도**를 유틸로 표준화하면 실무 품질이 급상승합니다.
- **루프·가독성·성능**의 밸런스를 유지하면서, 흔한 함정(누락된 `await`, forEach, 전역 미처리)만 피하면 `async/await`은 가장 강력한 비동기 도구가 됩니다.
