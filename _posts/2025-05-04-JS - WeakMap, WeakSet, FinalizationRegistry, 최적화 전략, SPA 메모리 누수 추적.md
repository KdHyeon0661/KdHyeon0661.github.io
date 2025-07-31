---
layout: post
title: JavaScript - WeakMap, WeakSet, FinalizationRegistry, 최적화 전략, SPA 메모리 누수 추적
date: 2025-04-28 23:20:23 +0900
category: JavaScript
---
# 🧠 고급 메모리 관리: WeakMap / WeakSet, FinalizationRegistry, 최적화 전략, SPA 메모리 누수 추적

자바스크립트는 자동으로 메모리를 관리하지만, **참조가 끊어지지 않으면 메모리 누수(Memory Leak)**가 발생할 수 있습니다.  
이런 문제를 해결하기 위해 ECMAScript에서는 **WeakMap**, **WeakSet**, 그리고 **FinalizationRegistry** 같은 **고급 메모리 도구**가 도입되었습니다.

또한, SPA처럼 **메모리를 오래 사용하는 애플리케이션**에서 어떻게 메모리를 최적화하고, 추적할 수 있는지도 함께 정리합니다.

---

## 📌 1. WeakMap / WeakSet

### ✅ WeakMap

> `WeakMap`은 키가 반드시 **객체**여야 하며, **약한 참조(weak reference)**로 유지됩니다.  
> 키가 GC 대상이 되면, `WeakMap`에서도 자동 제거됩니다.

```js
let wm = new WeakMap();
let obj = { name: "Alice" };

wm.set(obj, "Hello");
console.log(wm.get(obj)); // "Hello"

obj = null; // obj가 참조되지 않으면 GC 대상 → wm에서도 제거됨
```

### 🔒 특징

- 키는 **객체만 허용**
- **반복(iteration)** 불가 (`forEach`, `keys`, `values` 없음)
- 자동 메모리 관리 → **메모리 누수 방지**

### ✅ 사용 예시: 객체에 메타데이터 저장

```js
const privateData = new WeakMap();

function User(name) {
  privateData.set(this, { name });
}

User.prototype.getName = function () {
  return privateData.get(this).name;
};

const user = new User("Alice");
console.log(user.getName()); // "Alice"
```

---

### ✅ WeakSet

> 객체만 저장 가능한 `Set`의 변형.  
> 값은 약한 참조로 저장되며, 반복 불가, 자동 GC 적용.

```js
let ws = new WeakSet();
let obj = { id: 123 };

ws.add(obj);
console.log(ws.has(obj)); // true

obj = null; // obj가 참조되지 않으면 자동 제거
```

---

## 📌 2. FinalizationRegistry

> 객체가 GC에 의해 수거되기 직전에 **콜백을 등록**하여 정리 작업을 할 수 있도록 하는 기능

```js
const registry = new FinalizationRegistry((value) => {
  console.log(`객체가 수거됨: ${value}`);
});

let user = { name: "Bob" };
registry.register(user, "user1");

user = null; // GC가 발생하면 콜백 실행됨
```

### ⚠️ 주의

- **GC가 언제 일어날지는 보장되지 않음**
- **디버깅, 로깅, 리소스 정리용**으로 사용
- 실시간 로직(예: DB 커넥션 정리)에는 적합하지 않음

---

## 📌 3. 메모리 최적화 전략

| 전략 | 설명 |
|------|------|
| ❌ 전역 변수 최소화 | `window.obj = ...` 같은 전역 참조는 GC 되지 않음 |
| ✅ 이벤트 리스너 정리 | 컴포넌트 해제 시 `removeEventListener()` |
| ✅ 타이머/인터벌 정리 | `clearTimeout()`, `clearInterval()` 사용 |
| ✅ 클로저 정리 | 불필요한 참조는 `null`로 제거 |
| ✅ DOM 참조 해제 | DOM 노드 제거 시 JS에서도 참조 제거 |
| ✅ WeakMap 활용 | 메모리 자동 관리가 필요한 객체 저장 시 사용 |

---

## 📌 4. SPA에서의 메모리 누수 추적

SPA(Single Page Application)는 페이지를 전환하지 않고 뷰만 바꾸기 때문에 **오래 살아있는 상태, 이벤트, DOM 노드**가 누수의 주요 원인입니다.

### 💥 자주 발생하는 누수

1. 컴포넌트가 사라졌는데 이벤트 리스너가 남음
2. 해제되지 않은 setInterval / setTimeout
3. 클로저 내부에 DOM 또는 대용량 데이터가 유지됨
4. 캐시된 상태가 계속 누적됨

---

### 🛠️ 추적 방법 (Chrome DevTools 기준)

#### ① Performance 탭으로 메모리 타임라인 확인

1. `Record` 시작 → 앱 사용 → 중지
2. 메모리 선이 계속 오르면 누수 가능성 있음

#### ② Memory 탭의 Heap Snapshot 사용

1. Snapshot 찍기
2. 실행 후 다시 Snapshot
3. **Retainers**나 **Detached DOM Tree** 탐색
4. 수거되지 않은 객체가 있으면 GC가 실패했을 가능성

#### ③ Console에서 강제 GC 실행

```js
// Chrome 전용
window.gc();
```

> `chrome://flags → Enable 'Expose garbage collection'` 활성화 필요

---

## ✅ 5. 요약

| 항목 | 설명 |
|------|------|
| WeakMap | 객체 키만 가능, 약한 참조, GC 자동 |
| WeakSet | 객체만 저장, 반복 불가, 자동 수거 |
| FinalizationRegistry | 객체 수거 시 콜백 등록 (비결정적) |
| 메모리 누수 추적 | DevTools의 Memory, Performance 탭 활용 |
| SPA 누수 방지 | 이벤트 해제, 클로저 정리, 타이머 제거 필수 |

---

## 🔚 마무리

- 자바스크립트는 자동으로 메모리를 관리하지만, **참조가 남아있는 한 수거되지 않습니다.**
- `WeakMap`과 `FinalizationRegistry`는 **메모리 민감한 상황에서 강력한 도구**가 될 수 있습니다.
- 특히 SPA나 장시간 사용되는 웹앱에서는 **의도적인 정리 전략과 도구 활용**이 필수입니다.