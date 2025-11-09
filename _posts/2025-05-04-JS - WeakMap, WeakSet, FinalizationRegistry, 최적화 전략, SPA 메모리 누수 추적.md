---
layout: post
title: JavaScript - WeakMap, WeakSet, FinalizationRegistry, 최적화 전략, SPA 메모리 누수 추적
date: 2025-04-28 23:20:23 +0900
category: JavaScript
---
# 고급 메모리 관리: WeakMap / WeakSet, FinalizationRegistry, 최적화 전략, SPA 메모리 누수 추적

**요약**: 자동 GC가 있어도 **참조를 잡고 있으면** 메모리는 해제되지 않습니다. 이 글은 현대 JS에서 제공하는 **약참조 컨테이너(WeakMap/WeakSet/WeakRef)**, **수거 시 후처리(FinalizationRegistry)**를 심층 설명하고, **SPA(단일 페이지 앱)**에서의 누수 패턴, **DevTools 기반 추적 절차**, **안전한 클린업 코드 레시피**까지 실무 중심으로 정리합니다.

---

## 1. 개요

### 1.1 왜 “약한 참조(weak reference)”가 필요한가
- 강한 참조(Map/Set/객체 프로퍼티 등)는 키/값이 **어딘가에서 참조되는 한** GC 대상이 아닙니다.
- 장수 객체(캐시/전역 이벤트 버스/싱글톤)와 결합하면 **“숨은 뿌리”**가 되어 누수를 유발합니다.
- **WeakMap/WeakSet/WeakRef**는 **GC가 가능하도록**(키 또는 대상이 더 이상 강하게 참조되지 않으면) **자동으로 끊어지는** 약참조를 제공합니다.

### 1.2 언제 약참조를 쓰는가
- 외부 라이브러리/프레임워크가 **객체 생명주기를 주도**할 때, 여기에 **부가 정보/캐시를 덧붙이되 수명은 따라가야** 할 때.
- **DOM 노드 메타데이터**, **프록시 → 원본 매핑**, **컴포넌트 인스턴스별 비공개 상태**, **이미지/파싱 결과 캐시(Soft cache)** 등에 유리.

---

## 2. WeakMap / WeakSet 심층

### 2.1 공통 성질
- 키(WeakMap)·값(WeakSet)으로 **오직 객체만** 허용(원시값 불가).
- **약참조이므로 GC에 영향 없음**: 외부에서 더 이상 강하게 참조하지 않을 때 컨테이너 항목이 자동 제거.
- **열거 불가/크기 불명**: `keys()/values()/forEach/size`가 없습니다. (GC 타이밍이 비결정적이라 일관된 열거·크기 보장이 불가능)
- **메모리 누수 방지**: 장수 컨테이너가 대상 수명에 얽매이지 않습니다.

### 2.2 WeakMap 기본 사용
```js
const meta = new WeakMap();

function attachMeta(target, info) {
  meta.set(target, info);
}

function getMeta(target) {
  return meta.get(target); // target이 GC되면 자동 사라짐
}

// 예: DOM 노드에 메타데이터 부착
const el = document.createElement('div');
attachMeta(el, { createdAt: Date.now() });
console.log(getMeta(el)); // { createdAt: ... }
```

#### 패턴 A — 인스턴스 비공개 상태(프라이빗 에뮬레이션)
```js
const _state = new WeakMap();

class Counter {
  constructor() { _state.set(this, { value: 0 }); }
  inc() { _state.get(this).value++; }
  get value() { return _state.get(this).value; }
}
```
> ES13의 클래스 프라이빗 필드(`#x`)가 표준이지만, 프라이빗 필드를 쓰기 어려운 환경이라면 WeakMap으로 에뮬레이션 가능합니다.

#### 패턴 B — 캐시(Soft cache)
```js
const parseCache = new WeakMap();
export function parseJsonBound(objWithJsonStr) {
  if (parseCache.has(objWithJsonStr)) return parseCache.get(objWithJsonStr);
  const parsed = JSON.parse(objWithJsonStr.text);
  parseCache.set(objWithJsonStr, parsed);
  return parsed;
}
```

### 2.3 WeakSet 기본 사용
```js
const seen = new WeakSet();
function visit(node) {
  if (seen.has(node)) return;
  seen.add(node);
  // ... 처리
}
```
> 객체 방문 여부/중복 처리 플래그를 누수 없이 유지할 때 유용합니다.

### 2.4 반패턴과 주의
- **열거가 필요**하거나 **크기 제한/트림 정책**이 필요하면 WeakMap/WeakSet만으로는 불충분(LRU 등은 보조 구조 필요).
- **원시 키**를 쓰고 싶을 때는 그냥 `Map/Set`을 사용.
- 약참조 특성상 **항목이 언제 사라질지 믿고 로직을 짜면 안 됨**(비결정성). 항시 **폴백**을 둬야 합니다.

---

## 3. FinalizationRegistry 심층

### 3.1 개념
- 특정 객체가 **GC로 수거된 이후**(정확한 시점 보장 없음) **콜백을 “언젠가” 실행**해 주는 메커니즘.
- 주 용도: **진단/로깅/추적/약한 리소스 정리 힌트**.  
  DB 커넥션/파일 핸들 등 **즉시성·결정성이 필요한 정리**에는 **부적합**.

### 3.2 기본 사용과 토큰 관리
```js
const registry = new FinalizationRegistry((heldValue) => {
  // heldValue로 무엇이 수거됐는지 식별(문자열/아이디/약한 메타)
  console.log('collected:', heldValue);
});

function track(target, id, unregisterToken) {
  registry.register(target, id, unregisterToken);
}

const obj = {};
const token = {};
track(obj, 'obj#1', token);

// 참조 해제
// obj = null; // (예시 상황: 스코프 종료 등)

// 필요 시 수동 해제(더는 추적하지 않음)
registry.unregister(token);
```

#### 포인트
- `heldValue`는 수거 후 콜백으로 전달될 **작은 메타값**. 대용량 데이터나 순환 참조를 넣지 마세요.
- `unregister(token)`으로 **더는 필요 없는 추적을 해제**하여 불필요한 콜백을 방지.

### 3.3 WeakRef와 조합(Soft reference 캐시)
```js
const cache = new Map(); // 키는 문자열 등 원시 가능

function getCachedHeavy(key, factory) {
  const ref = cache.get(key);
  const cached = ref?.deref();
  if (cached) return cached;
  const fresh = factory();
  cache.set(key, new WeakRef(fresh));
  return fresh;
}
```
> WeakRef는 대상이 살아있을 땐 접근 가능하지만, **언제든 GC로 사라질 수 있으므로** `deref()` 결과가 `undefined`일 수 있다는 전제하에 사용해야 합니다. FinalizationRegistry로 **수거된 키를 로깅**하거나 **부수 캐시에서 제거**하는 힌트를 받는 식으로 보완할 수 있습니다.

### 3.4 금지/주의 사례
- **트랜잭션/파일/소켓 등 즉시 정리 필수 리소스의 종결**을 FinalizationRegistry에 맡기지 말 것.
- 콜백은 **메인 로직과 시간 결합**을 가지면 안 됩니다(지연/미실행 가능). **로깅/통계/샘플링** 수준에 한정.

---

## 4. WeakRef 실무 패턴

### 4.1 사용 전 체크
- 일부 환경/브라우저에서 **비활성**일 수 있음. 폴백 경로를 마련하세요.

### 4.2 미니 “소프트 캐시” 구현
```js
class SoftCache {
  #map = new Map(); // key -> WeakRef(value)
  get(key) { return this.#map.get(key)?.deref(); }
  set(key, value) { this.#map.set(key, new WeakRef(value)); }
  getOrSet(key, factory) {
    const v = this.get(key);
    if (v !== undefined) return v;
    const fresh = factory();
    this.set(key, fresh);
    return fresh;
  }
}
```

### 4.3 LRU와 혼합
- **핫 데이터는 강참조(LRU Map)**, **웜 데이터는 WeakRef**로 내리는 **2단계 캐시**가 현실적입니다.

---

## 5. SPA 메모리 최적화 전략(체크리스트 포함)

### 5.1 핵심 원칙
| 범주 | 권장 |
|---|---|
| 전역 참조 | 최소화. 디버깅용 전역 핸들러/상태를 반드시 해제. |
| 이벤트 | 컴포넌트 언마운트 시 **removeEventListener**/`off` 필수. `AbortController`로 일괄 취소. |
| 타이머 | `clearTimeout`/`clearInterval`은 “모듈 종료 훅에서” 확실히. |
| 옵저버 | `IntersectionObserver`/`ResizeObserver`/`MutationObserver`는 **disconnect()**. |
| DOM 참조 | DOM 제거 시 JS 레벨 참조도 **null**/삭제. |
| 캐시 | 용량/만료 정책 명시. 장수 Map/Set → WeakMap/WeakRef 검토. |
| 이미지/Blob | `URL.revokeObjectURL()` 호출 습관화. 캔버스는 `width/height` 재설정으로 버퍼 해제. |

### 5.2 React 예시 — 안전한 클린업
```jsx
import { useEffect, useRef } from "react";

export function Widget({ url }) {
  const abortRef = useRef(new AbortController());

  useEffect(() => {
    const controller = new AbortController();
    abortRef.current = controller;

    fetch(url, { signal: controller.signal })
      .then(r => r.json())
      .catch(e => { if (e.name !== 'AbortError') console.error(e); });

    return () => controller.abort(); // 언마운트 시 네트워크 취소
  }, [url]);

  useEffect(() => {
    const onScroll = () => {/* ... */};
    window.addEventListener('scroll', onScroll, { passive: true });
    return () => window.removeEventListener('scroll', onScroll);
  }, []);

  return <div>...</div>;
}
```

### 5.3 Vue 예시 — 옵저버/타이머 정리
```js
export default {
  mounted() {
    this.io = new IntersectionObserver(this.onIntersect);
    this.io.observe(this.$refs.target);
    this.timer = setInterval(this.tick, 1000);
  },
  beforeUnmount() {
    this.io?.disconnect();
    this.io = null;
    clearInterval(this.timer);
  },
  methods: {
    onIntersect(entries) { /* ... */ },
    tick() { /* ... */ }
  }
}
```

### 5.4 EventTarget 래퍼 — 범위 기반 등록/해제
```js
class ScopedEvents {
  constructor() { this._list = []; }
  on(target, type, handler, opts) {
    target.addEventListener(type, handler, opts);
    this._list.push(() => target.removeEventListener(type, handler, opts));
  }
  clear() { for (const off of this._list.splice(0)) off(); }
}
```
> 컴포넌트 단위로 생성해두고 **unmount에서 `clear()`** 한 번으로 정리.

### 5.5 URL.createObjectURL 관리
```js
const urls = new Set();
export function showBlob(blob) {
  const url = URL.createObjectURL(blob);
  urls.add(url);
  const img = new Image();
  img.onload = () => { URL.revokeObjectURL(url); urls.delete(url); };
  img.src = url;
}
export function cleanupAll() {
  for (const u of urls) URL.revokeObjectURL(u);
  urls.clear();
}
```

---

## 6. 전형적 누수 유형과 재현 코드

### 6.1 Detached DOM 참조
```js
let leaked = [];
function makeLeak() {
  const el = document.createElement('div');
  document.body.appendChild(el);
  document.body.removeChild(el); // DOM에서는 제거했지만…
  leaked.push(el);               // JS 배열이 계속 참조 → 누수
}
```
**대안**: 참조도 끊기.
```js
leaked = null; // 또는 leaked.length=0
```

### 6.2 이벤트 리스너 해제 누락
```js
function mount() {
  const el = document.getElementById('btn');
  const onClick = () => {/* ... 대용량 클로저 캡처 ... */};
  el.addEventListener('click', onClick);
  return () => el.removeEventListener('click', onClick); // 잊지 말 것
}
```

### 6.3 전역에 의한 숨은 뿌리
```js
window.DEBUG_LAST = someLargeGraph; // 디버깅 후 해제 잊음 → 장수
```

### 6.4 이벤트 버스 누적
```js
const bus = new Map(); // topic -> Set(handlers)
function on(topic, fn) {
  (bus.get(topic) ?? bus.set(topic, new Set()).get(topic)).add(fn);
}
function off(topic, fn) {
  bus.get(topic)?.delete(fn);
}
```
> **언마운트에서 `off` 호출** 또는 **Scopes/AbortSignal**로 자동화.

### 6.5 캐시 무한 성장
```js
const cache = new Map();
function heavy(key) {
  if (cache.has(key)) return cache.get(key);
  const v = computeHeavy(key);
  cache.set(key, v); // 만료 정책 없음 → 영구 증가
  return v;
}
```
**대안**: LRU/TTL 또는 WeakRef 캐시 사용.

---

## 7. 메모리 추적(Chrome DevTools/Node)

### 7.1 크롬 DevTools — Heap Snapshot
1) **Memory 탭 → Heap snapshot** 촬영  
2) 유저 플로우 수행(탭 전환/리스트 스크롤 등)  
3) 다시 스냅샷 후 **Comparison** 보기  
4) **Dominators**로 “누수 뿌리” 찾기, **Retainers**로 보유 체인 분석  
5) **Class filter**로 `Detached HTMLDivElement`/리스트 항목 누적 확인

### 7.2 Allocation instrumentation on timeline
- **Allocation sampling/recording**을 켜고 시나리오를 재현 → **누가 언제 할당했는지** 스택과 함께 확인.

### 7.3 Performance 탭 메모리 라인
- 장시간 **Record** 후 그래프가 **계단식 상승 후 떨어지지 않으면** 의심.  
- 강제 GC(설정 필요: Expose GC) 뒤에도 내려가지 않으면 실제 누수 가능성↑.

### 7.4 강제 GC (실험용)
```js
// chrome://flags → Expose garbage collection
window.gc(); // 테스트 환경에서만
```

### 7.5 Node.js 측면
- `node --inspect`로 크롬 DevTools 연결 → Heap snapshot/Allocation 프로파일링 가능.
- `heapdump` 패키지로 스냅샷 덤프 → 오프라인 분석.

### 7.6 자동 회귀 테스트(간단)
- Puppeteer 등으로 **특정 시나리오 수행 후 성능 측정**.  
- 사용자 액션 N회 후 **JS heap used** 값이 기준선보다 지속 상승하면 경고.

---

## 8. 운영 체크리스트

- [ ] 언마운트 훅에서 **이벤트/타이머/옵저버/웹소켓** 정리
- [ ] **AbortController**로 fetch/스트림 취소
- [ ] **URL.revokeObjectURL** 호출 누락 없음
- [ ] 장수 Map/Set 캐시에 **TTL/LRU/WeakRef** 적용
- [ ] 전역 디버깅 핸들/DOM 참조 제거
- [ ] DevTools 스냅샷 **비교 분석 루틴** 정착
- [ ] Third-party 위젯 **destroy/teardown** 명시 호출
- [ ] 이미지/캔버스 **오프스크린 정리**(캔버스 리사이즈로 버퍼 초기화)

---

## 9. 레시피 모음

### 9.1 AbortController로 “일괄 정리”되는 fetch
```js
class ScopedAbort {
  constructor(){ this.ctrl = new AbortController(); }
  get signal(){ return this.ctrl.signal; }
  abort(){ this.ctrl.abort(); }
}

const scope = new ScopedAbort();
fetch('/api', { signal: scope.signal }).catch(e => {
  if (e.name !== 'AbortError') console.error(e);
});
// 언마운트
scope.abort();
```

### 9.2 Observer 묶음 관리
```js
class Observers {
  #list = [];
  add(o, target, method = 'observe') {
    o[method](target);
    this.#list.push(() => o.disconnect?.());
  }
  clear() { for (const off of this.#list.splice(0)) off(); }
}
```

### 9.3 WeakMap으로 DOM 메타 안전 부착
```js
const meta = new WeakMap();
export function setMeta(node, data){ meta.set(node, data); }
export function getMeta(node){ return meta.get(node); }
// DOM 제거 시 별도 정리 불필요 — GC 대상이 되면 메타도 함께 사라짐
```

### 9.4 WeakRef + FinalizationRegistry로 소프트 캐시
```js
const cache = new Map(); // id -> WeakRef(value)
const reg = new FinalizationRegistry(id => {
  // 힌트: 필요 시 부가 인덱스/로그에서 정리
  // (여기서 주요 로직을 수행하지 말 것)
});

export function getOrCreate(id, factory) {
  const wr = cache.get(id);
  const v = wr?.deref();
  if (v) return v;
  const fresh = factory();
  cache.set(id, new WeakRef(fresh));
  reg.register(fresh, id);
  return fresh;
}
```

---

## 10. FAQ/오해 바로잡기

**Q1. WeakMap이면 누수가 “절대” 없나요?**  
A. **WeakMap 내부 항목은** 대상 객체의 생존과 함께 하므로 “WeakMap 자체”가 원인이 되는 누수는 줄지만, **키를 밖에서 강하게 잡고 있으면** GC되지 않습니다.

**Q2. FinalizationRegistry로 파일/소켓을 닫아도 되나요?**  
A. **권장하지 않습니다.** 실행 시점이 비결정적이어서 리소스 고갈/경쟁 상태를 야기할 수 있습니다. 명시적 `close()`/`dispose()`를 우선.

**Q3. WeakRef는 캐시에 만능인가요?**  
A. 약참조는 **언제든 사라질 수 있어** 히트율이 낮아질 수 있습니다. LRU/TTL 등과 **혼합 설계**가 현실적입니다.

**Q4. DevTools에서 “Detached DOM”이 뜨는데 꼭 버그인가요?**  
A. 대부분 버그지만, 특정 툴/프레임워크가 내부적으로 가진 참조일 수도 있습니다. **Retainers** 체인을 따라 **진짜 뿌리**를 확인하세요.

---

## 11. 결론

- **WeakMap/WeakSet/WeakRef**는 **대상 수명에 종속된 부가 상태/캐시**를 다룰 때 강력합니다.  
- **FinalizationRegistry**는 **디버깅/로깅 수준의 힌트**에 한정하세요. 즉시성 정리는 **명시적 클린업**이 원칙입니다.  
- SPA 환경에서는 **이벤트/타이머/옵저버/리소스**의 생명주기 관리를 표준화하고, **DevTools 스냅샷 비교/타임라인**으로 **누수 회귀 테스트**를 자동화하는 것이 가장 확실한 방어입니다.