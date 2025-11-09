---
layout: post
title: HTML - Web Storage API
date: 2025-04-09 19:20:23 +0900
category: HTML
---
# Web Storage API 완전 정리 (localStorage vs sessionStorage) — 실무 패턴, 보안, 성능까지

**Web Storage API**는 브라우저에 **키–값 문자열**을 저장하는 간단하고 빠른 저장소입니다.  
쿠키보다 용량이 크고(≈5–10MB), 요청마다 서버로 전송되지 않으며, 프론트엔드 상태/설정/캐시를 손쉽게 유지할 수 있습니다.  
이 글은 당신이 정리한 초안을 **확장**하여 API, 호환성, 보안 주의점, 동기화, 만료(TTL), 네임스페이스, 마이그레이션, 테스트, 에러처리까지 실무에서 꼭 쓰는 패턴을 예제와 함께 모았습니다.

---

## 0. 큰 그림: 언제 무엇을 쓰나?

| 시나리오 | 권장 |
|---|---|
| 사용자 선호(테마/언어), 간단한 플래그, 가벼운 캐시(수십 KB) | **localStorage** |
| 탭/창이 닫히면 사라져야 하는 임시 상태 | **sessionStorage** |
| 수백 KB–수MB 데이터, 구조화된 쿼리/트랜잭션 | **IndexedDB** |
| 오프라인 정적 자원 캐시(HTML/CSS/JS/이미지) | **Cache API + Service Worker** |
| 서버와 함께 보안 컨트롤이 필요한 세션 식별자 | **HttpOnly 쿠키** |

> 핵심: **민감정보(JWT/리프레시 토큰/개인정보)는 Web Storage에 보관하지 않음**. XSS에 노출됩니다.

---

## 1. API 빠른 복습

### 1.1 저장/조회/삭제/비우기

```js
// 저장
localStorage.setItem('username', 'Alice');

// 조회 (키가 없으면 null)
const name = localStorage.getItem('username'); // "Alice"

// 삭제
localStorage.removeItem('username');

// 전체 비우기 (주의!)
localStorage.clear();
```

`sessionStorage`도 동일한 메서드를 가집니다. **차이는 수명(스코프)** 뿐입니다.

### 1.2 보조 속성

```js
localStorage.length;      // 저장된 항목 수
localStorage.key(0);      // 인덱스로 key 조회 (구현별 순서는 보장되지 않음)
```

### 1.3 문자열만 저장

```js
const user = { name: 'Kim', age: 25 };

localStorage.setItem('user', JSON.stringify(user));

const restored = JSON.parse(localStorage.getItem('user') || 'null');
console.log(restored?.name); // "Kim"
```

> 문자열 이외는 `JSON.stringify`/`JSON.parse`. Date/Map/Set 등은 **직렬화 커스텀**이 필요.

---

## 2. 에러 처리 & 용량 제한

브라우저·사설탭마다 한도(대략 5–10MB)가 다릅니다. 초과시 `QuotaExceededError` 발생.

```js
function safeSetItem(key, value) {
  try {
    localStorage.setItem(key, value);
    return true;
  } catch (e) {
    if (e && (e.name === 'QuotaExceededError' || e.code === 22)) {
      // 대응: LRU 삭제, 압축, IndexedDB로 이동 등
      console.warn('Storage quota exceeded');
    } else {
      console.error('Storage error', e);
    }
    return false;
  }
}
```

### 2.1 사파리 프라이빗 모드 이슈
프라이빗 모드에서 `localStorage` 접근 자체가 예외를 던지거나 용량이 0일 수 있습니다.  
→ **기능 감지** + **폴백**(메모리 저장소) 준비.

```js
function hasLocalStorage() {
  try {
    const k = '__t__';
    localStorage.setItem(k, '1');
    localStorage.removeItem(k);
    return true;
  } catch { return false; }
}
```

---

## 3. 실무 유틸: 네임스페이스, 버전, TTL, 스키마

### 3.1 네임스페이스로 충돌 방지

```js
const NS = 'myapp:';

const storage = {
  set: (k, v) => localStorage.setItem(NS + k, v),
  get: (k) => localStorage.getItem(NS + k),
  del: (k) => localStorage.removeItem(NS + k),
  clearNS: () => {
    Object.keys(localStorage)
      .filter(k => k.startsWith(NS))
      .forEach(k => localStorage.removeItem(k));
  }
};
```

### 3.2 버전 태깅 & 마이그레이션

```js
const META_KEY = 'myapp:meta';

function readMeta() {
  try { return JSON.parse(localStorage.getItem(META_KEY)) || { version: 1 }; }
  catch { return { version: 1 }; }
}

function migrate() {
  const meta = readMeta();
  if (meta.version < 2) {
    // v1 → v2 마이그레이션 로직
    // 예: 키명 변경, 값 포맷 변환
    meta.version = 2;
    localStorage.setItem(META_KEY, JSON.stringify(meta));
  }
}
migrate();
```

### 3.3 TTL(만료) 지원 래퍼

```js
const ttlStore = {
  set(key, data, ttlMs) {
    const record = { data, expires: ttlMs ? Date.now() + ttlMs : null };
    localStorage.setItem(key, JSON.stringify(record));
  },
  get(key) {
    const raw = localStorage.getItem(key);
    if (!raw) return null;
    try {
      const { data, expires } = JSON.parse(raw);
      if (expires && Date.now() > expires) {
        localStorage.removeItem(key);
        return null;
      }
      return data;
    } catch {
      localStorage.removeItem(key);
      return null;
    }
  }
};

// 사용
ttlStore.set('weather:seoul', { temp: 9 }, 5 * 60 * 1000); // 5분 캐시
```

### 3.4 JSON 직렬화 커스텀(날짜/맵/셋)

```js
function stringify(value) {
  return JSON.stringify(value, (_, v) => {
    if (v instanceof Date) return { __type: 'Date', value: v.toISOString() };
    if (v instanceof Map)  return { __type: 'Map',  value: [...v] };
    if (v instanceof Set)  return { __type: 'Set',  value: [...v] };
    return v;
  });
}

function parse(text) {
  return JSON.parse(text, (_, v) => {
    if (v?.__type === 'Date') return new Date(v.value);
    if (v?.__type === 'Map')  return new Map(v.value);
    if (v?.__type === 'Set')  return new Set(v.value);
    return v;
  });
}
```

---

## 4. 탭 간 동기화: `storage` 이벤트

다른 탭에서 변경된 항목을 실시간 반영할 수 있습니다.

```js
window.addEventListener('storage', (e) => {
  if (!e.key) return; // clear 호출 시 null
  if (e.key.startsWith('myapp:')) {
    // e.oldValue, e.newValue 활용하여 상태 동기화
    console.log('changed', e.key, e.oldValue, e.newValue);
  }
});
```

> 같은 탭에서는 `storage` 이벤트가 발생하지 않습니다. 같은 탭 내 동기화는 **Pub/Sub** 직접 호출.

---

## 5. 보안 — 반드시 알아야 할 6가지

1) **민감정보 저장 금지**  
   액세스 토큰/리프레시 토큰/개인정보/결제수단은 **XSS에 탈취**됩니다.  
   인증은 가능하면 **HttpOnly, Secure 쿠키 + 서버 검증**(CSRF 방지는 `SameSite`/CSRF 토큰)로.

2) **XSS 차단**  
   모든 외부 입력을 **이스케이프/검증**. 템플릿에 삽입 시 `textContent` 사용. CSP 설정.

3) **키 네임스페이스 은닉**  
   예측 가능한 키 이름을 피하고, 필요하면 **난수 접두사**/유저별 명시적 네임스페이스.

4) **암호화에 기대지 말 것**  
   클라이언트 키로 암호화해도 **같은 영역**에서 복호화 가능합니다. 민감데이터는 서버/세션으로.

5) **만료/재검증**  
   로컬 캐시는 TTL과 함께 저장하고, **백엔드 ETag/Last-Modified**로 재검증.

6) **로그아웃 처리**  
   네임스페이스 단위 삭제 + 탭 간 `storage` 이벤트로 **즉시 반영**.

---

## 6. 패턴 모음

### 6.1 사용자 설정(테마/언어) 유지

```js
const PFX = 'settings:';

function setSetting(k, v) {
  localStorage.setItem(PFX + k, JSON.stringify(v));
}
function getSetting(k, fallback) {
  try { return JSON.parse(localStorage.getItem(PFX + k)) ?? fallback; }
  catch { return fallback; }
}

// 예: 다크모드
setSetting('theme', 'dark');
document.documentElement.dataset.theme = getSetting('theme', 'light');
```

### 6.2 임시 폼 자동 저장(sessionStorage)

```js
const KEY = 'draft:post';

const form = document.querySelector('#postForm');
const title = form.querySelector('input[name=title]');
const body  = form.querySelector('textarea[name=body]');

const saved = sessionStorage.getItem(KEY);
if (saved) {
  const { t, b } = JSON.parse(saved);
  if (t) title.value = t;
  if (b) body.value  = b;
}

form.addEventListener('input', () => {
  sessionStorage.setItem(KEY, JSON.stringify({ t: title.value, b: body.value }));
});
```

### 6.3 API 캐시(짧은 TTL, 실패 시 폴백)

```js
async function getWithCache(url, ttlMs) {
  const cached = ttlStore.get('cache:' + url);
  if (cached) return cached;

  try {
    const res = await fetch(url);
    if (!res.ok) throw new Error(res.statusText);
    const json = await res.json();
    ttlStore.set('cache:' + url, json, ttlMs);
    return json;
  } catch (e) {
    if (cached) return cached; // 소프트 폴백
    throw e;
  }
}
```

### 6.4 LRU(최근 미사용) 삭제로 쿼터 극복

간단 구현: 저장 시 접근 시간 업데이트, 용량 초과 시 **가장 오래된** 키 제거.

```js
const LRU_IDX = 'lru:index'; // 키 배열 저장

function lruTouch(key) {
  const idx = JSON.parse(localStorage.getItem(LRU_IDX) || '[]')
    .filter(k => k !== key);
  idx.push(key);
  localStorage.setItem(LRU_IDX, JSON.stringify(idx));
}

function lruEvictIfNeeded(tryFn) {
  try { return tryFn(); }
  catch (e) {
    if (e.name !== 'QuotaExceededError') throw e;
    const idx = JSON.parse(localStorage.getItem(LRU_IDX) || '[]');
    while (idx.length) {
      const victim = idx.shift();
      localStorage.removeItem(victim);
      localStorage.setItem(LRU_IDX, JSON.stringify(idx));
      try { return tryFn(); } catch (e2) {
        if (e2.name !== 'QuotaExceededError') throw e2;
      }
    }
    throw e;
  }
}
```

사용:

```js
lruEvictIfNeeded(() => {
  localStorage.setItem('big:key', largeJson);
  lruTouch('big:key');
});
```

### 6.5 타입 안전(TypeScript) 래퍼

```ts
type StoreValue = string | number | boolean | null | object;

class TypedStore<T extends Record<string, StoreValue>> {
  constructor(private ns: string) {}
  set<K extends keyof T>(k: K, v: T[K]) {
    localStorage.setItem(this.ns + String(k), JSON.stringify(v));
  }
  get<K extends keyof T>(k: K, fallback?: T[K]): T[K] | undefined {
    const raw = localStorage.getItem(this.ns + String(k));
    if (!raw) return fallback;
    try { return JSON.parse(raw) as T[K]; } catch { return fallback; }
  }
}

type Prefs = { theme: 'light' | 'dark'; fontSize: number };
const prefs = new TypedStore<Prefs>('prefs:');
prefs.set('theme', 'dark');
const fs = prefs.get('fontSize', 16);
```

---

## 7. 테스트 & 디버깅 팁

- **개발자도구** → Application(또는 Storage) 패널에서 local/sessionStorage 확인·수정·삭제
- 자동화 테스트에서 **스토리지 목킹**:

```js
function mockStorage() {
  let store = {};
  return {
    getItem: (k) => (k in store ? store[k] : null),
    setItem: (k, v) => { store[k] = String(v); },
    removeItem: (k) => { delete store[k]; },
    clear: () => { store = {}; },
    key: (i) => Object.keys(store)[i] || null,
    get length() { return Object.keys(store).length; }
  };
}

// 예: Vitest/Jest
Object.defineProperty(window, 'localStorage', { value: mockStorage() });
```

- **경계 테스트**: 큰 문자열(예: `'x'.repeat(1024*1024)`), JSON 파싱 실패, `null` 반환 처리

---

## 8. 브라우저·도메인 스코프

- **도메인별 격리**: `https://a.example.com` 과 `https://b.example.com` 은 서로 접근 불가
- **포트/프로토콜 포함**: `http` vs `https` 저장소는 별개
- PWA/사설탭은 **용량 정책**이 다를 수 있음

---

## 9. 성능 최적화

- 큰 데이터를 여러 키로 쪼개지 말고 **IndexedDB** 사용 고려
- 잦은 쓰기는 **배치/디바운스**:

```js
const queue = new Map();
const flush = () => {
  queue.forEach((v, k) => localStorage.setItem(k, v));
  queue.clear();
};
const scheduleSet = (k, v) => {
  queue.set(k, v);
  clearTimeout(scheduleSet.t);
  // @ts-ignore
  scheduleSet.t = setTimeout(flush, 100);
};
```

- **압축**: 텍스트 데이터가 크면 `lz-string` 등으로 압축(단, CPU 비용 고려)

---

## 10. 비교표 (쿠키/IndexedDB/Cache API)

| 항목 | localStorage | sessionStorage | 쿠키 | IndexedDB | Cache API |
|---|---|---|---|---|---|
| 데이터 크기 | 5–10MB | 탭 생존 | ~4KB | 수십~수백 MB | 대형 바이너리/응답 |
| 전송 | 클라 전용 | 클라 전용 | 매 요청에 포함 | X | X |
| 보안 | XSS 취약 | XSS 취약 | HttpOnly 가능 | 비교적 안전(그래도 XSS로 접근 가능) | SW 스코프 |
| 만료 | 직접 구현 | 탭 종료 | Max-Age/Expires | 앱 로직 | Cache 헤더/정책 |
| 쿼리 | 키 단위 | 키 단위 | 키 단위 | 인덱스/트랜잭션 | URL/요청 매칭 |
| 대표 용도 | 설정/가벼운 캐시 | 임시 상태 | 서버 세션/식별 | 대량 구조 데이터 | 오프라인 자원 |

---

## 11. 예제 모음

### 11.1 첫 방문 환영 배너(7일간 숨기기)

```js
const KEY = 'welcome:hidden';
const hidden = ttlStore.get(KEY);

if (!hidden) {
  document.querySelector('#welcome').style.display = 'block';
}
document.querySelector('#welcome .close').addEventListener('click', () => {
  ttlStore.set(KEY, true, 7 * 24 * 60 * 60 * 1000);
  document.querySelector('#welcome').style.display = 'none';
});
```

### 11.2 탭 간 로그아웃 동기화

```js
function logoutEverywhere() {
  localStorage.setItem('auth:logout', String(Date.now()));
  // 현재 탭 처리
  doLogoutUI();
}
window.addEventListener('storage', (e) => {
  if (e.key === 'auth:logout') doLogoutUI();
});
```

### 11.3 A/B 테스트 배정(안정적 랜덤)

```js
function getBucket() {
  const k = 'ab:bucket';
  let b = localStorage.getItem(k);
  if (!b) {
    b = Math.random() < 0.5 ? 'A' : 'B';
    localStorage.setItem(k, b);
  }
  return b;
}
document.body.dataset.bucket = getBucket();
```

---

## 12. 체크리스트

- [ ] 민감정보 저장 금지 (토큰/PII/결제정보)  
- [ ] TTL/버전/네임스페이스 설계  
- [ ] `storage` 이벤트로 탭 동기화  
- [ ] 쿼터 예외 처리 + LRU/압축/폴백  
- [ ] 사파리 프라이빗/ITP/용량 정책 대비  
- [ ] e2e/단위테스트에서 스토리지 목킹  
- [ ] 필요 시 IndexedDB/Cache API로 승격

---

## 결론

`localStorage`/`sessionStorage`는 **얕지만 넓게 유용한** 브라우저 내 상태 저장 수단입니다.  
**보안 경계(XSS)**를 명확히 인지하고, **TTL/네임스페이스/버전/동기화/쿼터 관리** 같은 실무 패턴을 적용하면,  
설정 유지·임시 저장·가벼운 캐시·UX 향상에 매우 큰 가치를 제공합니다.  
그 이상의 규모/요구가 생기면 **IndexedDB/Cache API**로 자연스럽게 확장하세요.