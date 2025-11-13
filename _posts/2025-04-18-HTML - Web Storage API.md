---
layout: post
title: HTML - Web Storage API
date: 2025-04-18 19:20:23 +0900
category: HTML
---
# Web Storage API (localStorage vs sessionStorage)

## 0. Web Storage 한눈에 보기

- **정의**: 브라우저에 **Key-Value(문자열)** 쌍을 저장하는 동기 API.
- **종류**
  - `localStorage`: 브라우저를 꺼도 **지속**. (도메인 단위)
  - `sessionStorage`: **탭(세션) 생명주기** 동안만 유효. 탭 닫으면 소멸, 탭 간 공유 없음.
- **특징**
  - 용량: 보통 **5~10MB** 수준(브라우저/플랫폼별 상이)
  - 네트워크 전송 없음(쿠키와 다름), **동기 API**(메인 스레드 블록 주의)
  - 값은 **항상 문자열** → 객체는 `JSON.stringify/parse` 필수

---

## 1. 기본 사용법 리마인드

```js
// localStorage
localStorage.setItem('username', 'Alice');
const name = localStorage.getItem('username'); // "Alice"
localStorage.removeItem('username');
localStorage.clear();

// sessionStorage
sessionStorage.setItem('token', 'abc123');
sessionStorage.getItem('token');
sessionStorage.removeItem('token');
sessionStorage.clear();
```

> 문법은 동일, **수명과 범위**만 다릅니다.

---

## 2. 문자열 전용 → JSON 안전 래퍼

```js
const storage = {
  get(key, fallback = null) {
    const raw = localStorage.getItem(key);
    if (raw == null) return fallback;
    try { return JSON.parse(raw); } catch { return raw; } // 원시 문자열도 허용
  },
  set(key, value) {
    const raw = (typeof value === 'string') ? value : JSON.stringify(value);
    localStorage.setItem(key, raw);
  },
  del(key) { localStorage.removeItem(key); },
  has(key) { return localStorage.getItem(key) !== null; }
};

// 사용
storage.set('user', { name: 'Kim', age: 25 });
console.log(storage.get('user').name); // "Kim"
```

---

## 3. TTL(만료) 기능 직접 구현하기

Web Storage에는 만료 개념이 없습니다. **메타데이터를 함께 저장**해 만료를 흉내 냅니다.

```js
const ttlStore = {
  set(key, value, ttlMs) {
    const expiresAt = Date.now() + ttlMs;
    localStorage.setItem(key, JSON.stringify({ value, expiresAt }));
  },
  get(key) {
    const raw = localStorage.getItem(key);
    if (!raw) return null;
    try {
      const { value, expiresAt } = JSON.parse(raw);
      if (typeof expiresAt === 'number' && Date.now() > expiresAt) {
        localStorage.removeItem(key);
        return null;
      }
      return value;
    } catch {
      return null;
    }
  }
};

// 예: 10분 캐시
ttlStore.set('profile', { id: 1, name: 'Alice' }, 10 * 60 * 1000);
```

---

## 4. 네임스페이스/버전/마이그레이션 전략

여러 기능이 한 스토리지를 공유하면 **키 충돌/청소**가 힘들어집니다. **접두사/버전**을 두세요.

```js
const NS = 'myapp:v2:';            // 버전 업 시 접두사 교체
const keyOf = (k) => `${NS}${k}`;

function setNs(k, v) { localStorage.setItem(keyOf(k), JSON.stringify(v)); }
function getNs(k, fb=null) {
  const raw = localStorage.getItem(keyOf(k));
  if (!raw) return fb;
  try { return JSON.parse(raw); } catch { return fb; }
}
function wipeNamespace(ns = NS) {
  for (let i=localStorage.length-1; i>=0; i--) {
    const k = localStorage.key(i);
    if (k && k.startsWith(ns)) localStorage.removeItem(k);
  }
}

// 마이그레이션 예시
(function migrate() {
  const OLD = 'myapp:v1:';
  if (localStorage.getItem(`${NS}migrated`)) return;
  for (let i=0; i<localStorage.length; i++) {
    const k = localStorage.key(i);
    if (k?.startsWith(OLD)) {
      const v = localStorage.getItem(k);
      const nk = k.replace(OLD, NS);
      localStorage.setItem(nk, v);
      localStorage.removeItem(k);
    }
  }
  localStorage.setItem(`${NS}migrated`, '1');
})();
```

---

## 5. 교차 탭 동기화 — `storage` 이벤트

같은 도메인의 **다른 탭**에서 `localStorage`가 변경되면 이벤트가 발생합니다. (단, **변경을 수행한 탭에는 트리거되지 않음**)

```js
window.addEventListener('storage', (e) => {
  if (!e.key) return; // clear() 에서는 key가 null
  if (e.key.startsWith('myapp:v2:')) {
    // e.oldValue, e.newValue 로 변경 전/후 값 확인
    console.log('다른 탭에서 변경:', e.key, e.newValue);
    // 상태/캐시 리프레시 등
  }
});
```

> `sessionStorage`는 탭 간 격리되어 **이 이벤트로 공유되지 않습니다**.

---

## 6. 예외/쿼터(용량 초과) 처리 — 안전한 저장

모바일/사파리/프라이빗 모드에선 **쿼터가 작은 경우**가 있습니다. 반드시 try/catch.

```js
function safeSetItem(k, v) {
  try {
    localStorage.setItem(k, v);
    return true;
  } catch (err) {
    if (err?.name === 'QuotaExceededError' || err?.code === 22) {
      // LRU 정책, 오래된 캐시 제거 등
      evictCacheEntries(); // 사용자 정의
      try { localStorage.setItem(k, v); return true; } catch { /* 여전히 실패 */ }
    }
    // 읽기 전용 스토리지/프라이빗 모드 등
    console.warn('storage set 실패:', err);
    return false;
  }
}
```

**권장 정책**
- **캐시 우선 제거**: 프로필/설정보다 **재생성 가능한 데이터**부터 삭제
- **최대 항목 수/사이즈**를 내부적으로 제한
- 실패 시 **메모리 캐시**로 폴백하고 다음 방문에 재시도

---

## 7. 성능/동기성 주의 (메인 스레드)

- API는 **동기**이므로 대량 쓰기/루프는 프레임 드랍 유발 가능 → **배치/디바운스** 권장
- 큰 JSON을 자주 `stringify/parse` 하면 CPU 비용 큼 → 필요한 필드만 분리 저장 고려

```js
const queue = [];
let flushing = false;

function batchedSet(key, value) {
  queue.push([key, value]);
  if (!flushing) requestIdleCallback(flush, { timeout: 500 }); // 폴백은 setTimeout
}

function flush() {
  flushing = true;
  try {
    for (const [k, v] of queue.splice(0)) {
      localStorage.setItem(k, typeof v === 'string' ? v : JSON.stringify(v));
    }
  } finally {
    flushing = false;
  }
}
```

---

## 8. 보안 고려(중요)

| 이슈 | 설명/가이드 |
|---|---|
| XSS | Web Storage는 **암호화 없음/HttpOnly 아님** → 스크립트 유출에 취약. **민감 정보(토큰/PII) 저장 금지**. CSP/출력 인코딩/라이브러리 검증 필수. |
| ITP/Private 모드 | 사파리 ITP/모바일 프라이빗 모드에서 용량/영속성 제약이 크거나 “세션 스토리지처럼” 동작할 수 있음. **가용성 체크 + 실패 폴백** 구현. |
| 도메인 경계 | 동일 출처 정책(SOP) 적용. **서브도메인도 분리**. SSO/멀티 서브도메인 공유가 필요하면 서버 측 세션/쿠키 전략 검토. |
| 전송 | 요청에 자동 포함되지 않지만, **XSS로 읽혀 외부 송신** 가능 → 네트워크 DLP/콘텐츠 보안 고려. |
| 암호화 | WebCrypto로 암호화 후 저장은 가능하나 **키 관리**가 관건(키를 JS에 넣으면 의미 약함). **서버 세션/HttpOnly 쿠키**가 민감데이터에 더 적합. |

간단 암호화 예(교육용 — 키/초기화 벡터 관리 별도):

```js
// WebCrypto AES-GCM 예시 (민감데이터는 가급적 저장하지 말 것)
async function encryptAndStore(key, name, obj) {
  const iv = crypto.getRandomValues(new Uint8Array(12));
  const data = new TextEncoder().encode(JSON.stringify(obj));
  const cipher = await crypto.subtle.encrypt({ name: 'AES-GCM', iv }, key, data);
  localStorage.setItem(name, JSON.stringify({ iv: Array.from(iv), c: Array.from(new Uint8Array(cipher)) }));
}
```

> 실제 서비스에선 **민감정보는 HttpOnly 쿠키 기반 세션**을 권장.

---

## 9. 환경/가용성 체크

```js
function storageAvailable(type = 'localStorage') {
  try {
    const s = window[type];
    const x = '__test__';
    s.setItem(x, x); s.removeItem(x);
    return true;
  } catch { return false; }
}
```

실패 시 **메모리 캐시/IndexedDB/쿠키** 등 폴백.

---

## 10. React 훅/Vanilla 유틸

### 10.1 React: `useLocalStorage` 훅

```jsx
import { useEffect, useState } from 'react';

export function useLocalStorage(key, initialValue) {
  const read = () => {
    try {
      const raw = localStorage.getItem(key);
      return raw ? JSON.parse(raw) : initialValue;
    } catch { return initialValue; }
  };
  const [state, setState] = useState(read);

  useEffect(() => {
    try { localStorage.setItem(key, JSON.stringify(state)); } catch {}
  }, [key, state]);

  // 다른 탭에서 변경되면 반영
  useEffect(() => {
    const onStorage = (e) => {
      if (e.key === key && e.storageArea === localStorage) {
        setState(e.newValue ? JSON.parse(e.newValue) : initialValue);
      }
    };
    window.addEventListener('storage', onStorage);
    return () => window.removeEventListener('storage', onStorage);
  }, [key]);

  return [state, setState];
}

// 사용
// const [theme, setTheme] = useLocalStorage('myapp:v2:theme', 'light');
```

### 10.2 Vanilla 모듈(네임스페이스+TTL+안전 저장 종합)

```js
export function createStore(ns='app:v1:') {
  const K = (k) => `${ns}${k}`;

  const get = (k, fb=null) => {
    const raw = localStorage.getItem(K(k));
    if (!raw) return fb;
    try {
      const obj = JSON.parse(raw);
      if (obj?.expiresAt && Date.now() > obj.expiresAt) {
        localStorage.removeItem(K(k));
        return fb;
      }
      return ('value' in obj) ? obj.value : obj; // 평문/TTL 혼용 지원
    } catch { return fb; }
  };

  const set = (k, v, ttlMs=0) => {
    const payload = ttlMs > 0
      ? { value: v, expiresAt: Date.now() + ttlMs }
      : v;
    try {
      localStorage.setItem(K(k), JSON.stringify(payload));
      return true;
    } catch (e) {
      // 쿼터 초과 등
      console.warn('store set fail', e);
      return false;
    }
  };

  const del = (k) => localStorage.removeItem(K(k));
  const clearNS = () => {
    for (let i=localStorage.length-1; i>=0; i--) {
      const key = localStorage.key(i);
      if (key?.startsWith(ns)) localStorage.removeItem(key);
    }
  };

  return { get, set, del, clearNS, key:K };
}
```

---

## 11. 실전 예시 시나리오

### 11.1 “로그인 유지(비민감) + 사용자 설정 + 장바구니”

- **로그인 토큰**: 민감 → **HttpOnly 쿠키/서버 세션** 권장.
  단, 공개 토큰(예: 비민감 읽기 전용 API Key)은 TTL과 함께 보관 가능.
- **사용자 설정(테마/언어/최근탭)**: `localStorage` 적합.
- **장바구니**: 비로그인 사용자의 임시 상태 → TTL 7일 등.

```js
const store = createStore('shop:v3:');
store.set('cart', { items:[{id:1,qty:2}] }, 7*24*60*60*1000);
store.set('ui', { theme: 'dark', locale: 'ko' });
```

### 11.2 “폼 임시 저장(세션 범위)”

```js
// 페이지별 form-state
sessionStorage.setItem('form:/profile', JSON.stringify(formState));
window.addEventListener('beforeunload', () => {
  sessionStorage.removeItem('form:/profile'); // 필요 시 정리
});
```

### 11.3 “교차 탭 로그아웃 동기화”

```js
// 로그아웃 시
localStorage.setItem('myapp:v2:logout', String(Date.now()));

// 모든 탭
window.addEventListener('storage', (e) => {
  if (e.key === 'myapp:v2:logout') {
    // 즉시 세션 종료/리다이렉트
    location.href = '/login?reason=logout';
  }
});
```

---

## 12. Web Storage vs IndexedDB vs 쿠키 vs Service Worker

| 항목 | Web Storage | IndexedDB | 쿠키 | Service Worker/Cache |
|---|---|---|---|---|
| 구조 | Key-Value 문자열 | 객체 저장/인덱스/쿼리 | Key-Value 문자열 | 요청/응답 캐시 |
| 용량 | ~5–10MB | 수백MB 가능 | ~4KB | 자산/응답 대용량 가능 |
| 동기/비동기 | **동기** | **비동기**(Promise) | 전송 포함 | SW 이벤트기반 |
| 전송 | ❌ | ❌ | **요청에 자동 포함** | ❌ |
| 민감 정보 | 비권장 | 상대적으로 나음(여전히 XSS 주의) | HttpOnly로 보호 가능 | ❌ |
| 사용 예 | 설정/작은 캐시 | 대형 캐시/오프라인 DB | 인증 세션/서버통신 | 정적자산/응답 캐시 |

> **민감한 인증**: **HttpOnly 쿠키**로!
> **대용량/구조화 데이터**: **IndexedDB**.
> **정적/응답 캐시**: **Service Worker + Cache API**.
> **가벼운 상태/설정**: **Web Storage**.

---

## 13. 테스트/디버깅 팁

- DevTools → Application(Storage) 패널에서 **키/값/크기** 확인/삭제
- **유닛 테스트**: JSDOM/테스트 러너에서 `localStorage` 목/스텁 주입
- **로그/버전**: 시작 시 `console.info('storage usage', estimate())` 형태로 용량 리포트

간단 용량 추정:

```js
function approximateStorageSize() {
  let total = 0;
  for (let i=0; i<localStorage.length; i++) {
    const k = localStorage.key(i);
    const v = localStorage.getItem(k);
    total += (k?.length ?? 0) + (v?.length ?? 0);
  }
  return total; // 문자 수 (바이트 근사 아님)
}
```

---

## 14. 브라우저 호환/주의

- 지원: Chrome/Firefox/Safari/Edge(모바일 포함).
- IE8+ 일부 지원(레거시 환경에선 폴리필 고려).
- **사파리 Private 모드**: 저장 실패 또는 세션성 동작 → **가용성 체크 + 폴백**.

---

## 15. 보일러플레이트: 안전한 “Config 저장소”

```js
// 앱 전역 설정/토글/최신 사용 문서 등
const cfg = createStore('cfg:v1:');

export const AppConfig = {
  get theme() { return cfg.get('theme', 'light'); },
  set theme(v) { cfg.set('theme', v); },

  get locale() { return cfg.get('locale', 'ko'); },
  set locale(v) { cfg.set('locale', v); },

  // 최근 문서 아이디 10개 유지
  pushRecent(id) {
    const arr = cfg.get('recentDocs', []);
    const next = [id, ...arr.filter(x => x !== id)].slice(0, 10);
    cfg.set('recentDocs', next);
  }
};
```

---

## 16. 요약 표(확장)

| 항목 | localStorage | sessionStorage |
|---|---|---|
| 수명 | 브라우저 종료 후에도 지속 | **탭** 종료 시 삭제 |
| 범위 | 도메인 단위 공유(동일 브라우저) | 탭(윈도우) 단위 격리 |
| 이벤트 | `storage`로 **다른 탭 변경 감지** | 감지 불가 |
| 용도 | 설정/장바구니/캐시(비민감) | 임시 폼/임시 상태 |
| 보안 | **민감정보 금지**, XSS 취약 | 동일 |
| 성능 | 동기. 대량 쓰기 주의 | 동일 |

---

## 17. 체크리스트

- [ ] **민감정보 저장 금지**(토큰/PII)
- [ ] XSS 방어(CSP/템플릿 인코딩/라이브러리 검증)
- [ ] **가용성/쿼터 예외 처리**, 프라이빗/사파리 대응
- [ ] 네임스페이스/버전/마이그레이션 계획
- [ ] TTL/캐시 청소 정책
- [ ] 교차 탭 동기화(storage 이벤트) 활용
- [ ] 필요한 경우 IndexedDB/Service Worker로 역할 분리

---

## 참고 자료

- MDN Web Docs — Web Storage API
- Can I Use — localStorage / sessionStorage
- OWASP Cheat Sheet — DOM 기반 XSS 방어
