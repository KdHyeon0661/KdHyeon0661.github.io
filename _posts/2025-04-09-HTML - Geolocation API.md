---
layout: post
title: HTML - Geolocation API
date: 2025-04-09 21:20:23 +0900
category: HTML
---
# HTML5 Geolocation API 완벽 가이드

## 빠른 시작 — 현재 위치 한 번 가져오기

```html
<button id="btn">내 위치 가져오기</button>
<pre id="out"></pre>

<script>
const $out = document.getElementById('out');

document.getElementById('btn').addEventListener('click', () => {
  if (!('geolocation' in navigator)) {
    $out.textContent = '이 브라우저는 Geolocation을 지원하지 않습니다.';
    return;
  }

  navigator.geolocation.getCurrentPosition(
    pos => {
      const { latitude, longitude, accuracy } = pos.coords;
      $out.textContent = `위도: ${latitude}\n경도: ${longitude}\n정확도(±m): ${accuracy}`;
    },
    err => {
      $out.textContent = `오류: ${err.code} / ${err.message}`;
    },
    { enableHighAccuracy: true, timeout: 10_000, maximumAge: 0 }
  );
});
</script>
```

**옵션**
- `enableHighAccuracy`: 더 정확한 센서(GPS 등) 사용 시도. **배터리/지연 증가** 가능.
- `timeout`: 응답 대기 시간(ms).
- `maximumAge`: 캐시된 위치 허용 시간(ms). `0`이면 **항상 최신 측정**만.

---

## 반환 객체 구조 (좌표·정확도·속도)

```js
// success 콜백의 position
{
  coords: {
    latitude,          // 위도 (deg)
    longitude,         // 경도 (deg)
    accuracy,          // 수평 정확도 (미터)
    altitude,          // 고도(미터, 장치/환경 따라 null)
    altitudeAccuracy,  // 고도 정확도(미터, null 가능)
    heading,           // 진행 방향(도, 북=0, 동=90) / 정지 시 null
    speed              // 속도(m/s, 정지 시 null)
  },
  timestamp: 1731081600000 // Epoch(ms)
}
```

> **정확도(accuracy)**는 “오차 반경(±m)”에 가깝습니다. 즉, 실제 위치는 **반경 accuracy 미터** 원 내부에 있을 확률이 높습니다.

---

## 연속 추적 — 이동 시마다 콜백 받기

```html
<button id="start">트래킹 시작</button>
<button id="stop">트래킹 중지</button>
<pre id="log"></pre>

<script>
let watchId = null;
const $log = document.getElementById('log');

document.getElementById('start').addEventListener('click', () => {
  if (watchId !== null) return; // 중복 방지
  watchId = navigator.geolocation.watchPosition(
    pos => {
      const { latitude, longitude, speed } = pos.coords;
      $log.textContent = `lat=${latitude}, lon=${longitude}, speed=${speed ?? 0}m/s`;
    },
    err => {
      $log.textContent = `오류: ${err.code}/${err.message}`;
    },
    { enableHighAccuracy: true, maximumAge: 2000 }
  );
});

document.getElementById('stop').addEventListener('click', () => {
  if (watchId !== null) {
    navigator.geolocation.clearWatch(watchId);
    watchId = null;
    $log.textContent += '\n트래킹 종료';
  }
});
</script>
```

**실무 팁**
- `watchPosition`은 **콜백 빈도**가 환경에 따라 상이. 필요 시 **쓰로틀링**으로 UI/네트워크 부하를 줄이세요.
- 장시간 추적은 **배터리 소모↑**. 상황에 맞게 **정밀도/주기**를 조절하고, **유휴 시 중지**하세요.

---

## 에러 처리 패턴

```js
function handleGeoError(err) {
  switch (err.code) {
    case err.PERMISSION_DENIED:
      alert('사용자가 위치 접근을 거부했습니다. (브라우저 주소창의 자물쇠/권한 설정을 확인)');
      break;
    case err.POSITION_UNAVAILABLE:
      alert('위치 정보를 사용할 수 없습니다. (실내/기내/센서 비활성)');
      break;
    case err.TIMEOUT:
      alert('위치 요청이 시간 초과되었습니다.');
      break;
    default:
      alert(`알 수 없는 오류: ${err.message}`);
  }
}
```

**현실적인 대응**
- 권한 거부 시: **권한 안내 UI**(도움말, OS/브라우저 설정 이동 안내).
- 획득 실패 시: **IP 기반 대략 위치**(정밀도 낮음)로 폴백하거나, **직접 동네/주소 입력** UX 제공.

---

## Promise/`async` 스타일로 깔끔하게

```js
function getCurrentPositionAsync(options) {
  return new Promise((resolve, reject) => {
    navigator.geolocation.getCurrentPosition(resolve, reject, options);
  });
}

async function locate() {
  if (!('geolocation' in navigator)) throw new Error('미지원');
  const pos = await getCurrentPositionAsync({ enableHighAccuracy: true, timeout: 8000 });
  return pos.coords;
}

// 사용
locate().then(console.log).catch(console.error);
```

**타임아웃 레이스(경쟁)로 안전성↑**

```js
async function getCurrentPositionWithHardTimeout(ms, options) {
  const geo = new Promise((res, rej) => {
    navigator.geolocation.getCurrentPosition(res, rej, options);
  });
  const timer = new Promise((_, rej) => setTimeout(() => rej(new Error('HARD_TIMEOUT')), ms));
  return Promise.race([geo, timer]);
}
```

---

## 정확도·지연·배터리 — 트레이드오프 설계

- **enableHighAccuracy=true**
  - 장점: 더 정확(특히 실외 GPS)
  - 단점: **배터리 소모↑**, **획득 지연↑**
- **maximumAge**
  - 캐시 허용. **재측정 없이** 빠르게 응답 가능(배터리 절약).
  - 단, **너무 오래된 위치**는 UX 저하.

**권장 전략**
1) 첫 화면: `maximumAge: 60_000` 정도로 **빠른 캐시 위치** 우선 제공
2) 백그라운드로 **정밀 재측정**(enableHighAccuracy=true)하여 지도/거리 재계산

---

## — Haversine

두 위경도 \((\varphi_1,\lambda_1),(\varphi_2,\lambda_2)\) 사이의 대권거리 \(d\) (반지름 \(R\)):

$$
\Delta\varphi=\varphi_2-\varphi_1,\quad
\Delta\lambda=\lambda_2-\lambda_1 \\
a=\sin^2\left(\frac{\Delta\varphi}{2}\right)+\cos\varphi_1\cos\varphi_2\sin^2\left(\frac{\Delta\lambda}{2}\right) \\
d=2R\arcsin\left(\sqrt{a}\right)
$$

```js
function haversineDistance(lat1, lon1, lat2, lon2, radius = 6371000) {
  const toRad = d => (d * Math.PI) / 180;
  const φ1 = toRad(lat1), φ2 = toRad(lat2);
  const Δφ = toRad(lat2 - lat1);
  const Δλ = toRad(lon2 - lon1);
  const a = Math.sin(Δφ/2)**2 + Math.cos(φ1)*Math.cos(φ2)*Math.sin(Δλ/2)**2;
  return 2 * radius * Math.asin(Math.sqrt(a)); // meters
}
```

> 반지름 \(R\)은 평균 지구반경 **6,371,000m** 근사치를 사용합니다.

---

## 예

```html
<link
  rel="stylesheet"
  href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css"
  integrity="sha256-p4NxAoJBhIIN+hmNHrzRCf9tD/miZyoHS5obTRR9BMY="
  crossorigin=""
/>
<div id="map" style="height:360px;border:1px solid #ccc"></div>

<script
  src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"
  integrity="sha256-20nQCchB9co0qIjJZRGuk2/Z9VM+kNiyxNV1lvTlZBo="
  crossorigin=""
></script>

<script>
async function initLeaflet() {
  if (!('geolocation' in navigator)) {
    alert('Geolocation 미지원');
    return;
  }
  const pos = await new Promise((res, rej) =>
    navigator.geolocation.getCurrentPosition(res, rej, { enableHighAccuracy: true })
  );
  const { latitude: lat, longitude: lng, accuracy } = pos.coords;

  const map = L.map('map').setView([lat, lng], 15);
  L.tileLayer('https://tile.openstreetmap.org/{z}/{x}/{y}.png', {
    attribution: '&copy; OpenStreetMap contributors'
  }).addTo(map);

  L.marker([lat, lng]).addTo(map).bindPopup('현재 위치');
  L.circle([lat, lng], { radius: accuracy, color: 'blue', fillOpacity: 0.1 }).addTo(map);
}
initLeaflet().catch(console.error);
</script>
```

- 파란 원은 **정확도(accuracy)** 오차 반경을 시각화합니다.
- Google Maps를 쓸 때는 **API 키**와 요금·이용약관을 반드시 확인하세요.

---

## 고급: `watchPosition` + 쓰로틀 + 이탈 감지

```js
// N초마다 한 번만 서버에 업로드(쓰로틀)
function throttle(fn, ms) {
  let last = 0;
  let pending = null;
  return (...args) => {
    const now = Date.now();
    if (now - last >= ms) {
      last = now;
      fn(...args);
    } else {
      clearTimeout(pending);
      pending = setTimeout(() => { last = Date.now(); fn(...args); }, ms - (now - last));
    }
  };
}

const uploadPosition = throttle(async pos => {
  const { latitude, longitude, accuracy } = pos.coords;
  // 예: 서버 전송 (에러/재시도·백오프 필요)
  await fetch('/api/track', {
    method: 'POST',
    headers: { 'content-type': 'application/json' },
    body: JSON.stringify({ latitude, longitude, accuracy, ts: pos.timestamp })
  });
}, 5000);

let lastLat = null, lastLon = null;
const THRESHOLD_M = 30;

function onMove(pos) {
  const { latitude: lat, longitude: lon } = pos.coords;
  if (lastLat != null) {
    const d = haversineDistance(lastLat, lastLon, lat, lon);
    if (d < THRESHOLD_M) return; // 30m 이내면 무시(정지/드리프트)
  }
  lastLat = lat; lastLon = lon;
  uploadPosition(pos);
}

const id = navigator.geolocation.watchPosition(onMove, console.error, {
  enableHighAccuracy: true,
  maximumAge: 2000
});
```

- **드리프트 필터**(임계 거리 미만 무시)로 불필요한 업데이트 억제.
- **쓰로틀**로 서버/네트워크 부하 완화.

---

## 권한·보안·정책 (반드시 숙지)

- **HTTPS 필수(또는 localhost)**: 보안 문맥(Secure Context)에서만 동작.
- **권한 프롬프트**: 사용자의 **명시적 허용**이 필요. UX 상 **행동 유도 버튼** 뒤에 요청하세요.
- **권한 사전 점검 (Permissions API)**

```js
if ('permissions' in navigator) {
  const status = await navigator.permissions.query({ name: 'geolocation' });
  // status.state: 'granted' | 'prompt' | 'denied'
}
```

- **iframe** 내에서: 상위 문서가 **Permissions-Policy**로 `geolocation`을 허용해야 함.
  예) `<iframe src="..." allow="geolocation">`
- **Service Worker**에서는 Geolocation 사용 불가(문서 컨텍스트에서만).
- **개인정보 보호**:
  - 최소 수집(필요한 시점·필드만),
  - 저장 시 **익명화/가명화**,
  - 전송 시 **TLS**,
  - 보관 기간·파기 정책 명시.
  - 국내 **개인정보보호법(PIPA)**, EU **GDPR** 등 **법적 근거**와 **동의 고지** 준수.

---

## 브라우저/플랫폼 주의점

- **iOS Safari**:
  - iOS 13+에서 HTTPS 요구 엄격.
  - PWA 백그라운드 지속 추적은 **큰 제약**(OS 정책).
- **Android WebView**: 앱 단 권한(위치) 허용 필요.
- **데스크톱**: Wi-Fi/네트워크 기반 위치 — 실외 GPS보다 오차 큼.
- **실내 환경**: 위치 흔들림(Drift) 빈번 → **정확도/드리프트 필터** 필수.

---

## 테스트·모킹

- **DevTools** > Sensors(Chrome) → Location을 **Custom**으로 설정해 경로 시뮬레이션.
- **권한 상태**: 브라우저 주소창 자물쇠 아이콘에서 사이트 권한 초기화/변경.
- **오프라인 시나리오**: 네트워크 탭에서 Offline 전환 후 폴백 UX 확인.

---

## 실전 사례 패턴

### 위치 → 날씨 API

```js
async function fetchWeatherByGeolocation() {
  const pos = await getCurrentPositionAsync({ timeout: 6000, maximumAge: 60000 });
  const { latitude, longitude } = pos.coords;
  const resp = await fetch(`/api/weather?lat=${latitude}&lon=${longitude}`);
  return await resp.json();
}
```

### 반경 검색(예: 2km 내 카페)

```js
function withinRadius(poi, center, r) {
  return haversineDistance(poi.lat, poi.lon, center.lat, center.lon) <= r;
}
```

> 대규모 데이터에는 **지오해시(GeoHash)**, **R-Tree**, **S2 Geometry** 등 공간 인덱싱을 고려.

---

## 반응형 지도 + 위치 버튼 UI 예시(접근성 포함)

```html
<button id="locate" aria-label="현재 위치로 이동">현재 위치</button>
<div id="map" style="height: 360px"></div>

<script>
async function onLocateClick() {
  try {
    const pos = await getCurrentPositionAsync({ enableHighAccuracy: true, timeout: 5000 });
    const { latitude: lat, longitude: lng } = pos.coords;
    // 지도 SDK로 이동/마커 갱신
  } catch (e) {
    alert('위치 접근 실패: 권한/네트워크/센서 상태를 확인하세요.');
  }
}
document.getElementById('locate').addEventListener('click', onLocateClick);
</script>
```

- 버튼으로 **명시적 행동**을 유도한 뒤 위치 요청 → 권한 프롬프트 전환 **맥락이 자연스러움**.
- 스크린리더를 위한 **`aria-label`** 제공.

---

## TypeScript 타입 안전 예시

```ts
type Coords = { lat: number; lon: number; accuracy?: number };

export async function currentCoords(): Promise<Coords> {
  const pos = await new Promise<GeolocationPosition>((res, rej) =>
    navigator.geolocation.getCurrentPosition(res, rej, { enableHighAccuracy: true })
  );
  return {
    lat: pos.coords.latitude,
    lon: pos.coords.longitude,
    accuracy: pos.coords.accuracy
  };
}
```

---

## UX 체크리스트

- [ ] 권한 요청은 **사용자 동작 직후**(버튼) 트리거.
- [ ] 실패 시 **대체 경로**(수동 주소 입력, IP 기반 대략 위치).
- [ ] **정확도/배터리/지연** 균형: 초기 캐시 → 정밀 재측정.
- [ ] **오차 시각화**(정확도 원), **마지막 업데이트 시간** 표기.
- [ ] **프라이버시 고지**(수집 목적·범위·보관·제3자 공유 여부).
- [ ] **CLS 방지**: 지도/카드 영역 높이 예약.
- [ ] **네트워크 오류/타임아웃** 재시도 UI(백오프).

---

## 한눈에 보는 API 표

| 함수 | 용도 | 비고 |
|---|---|---|
| `getCurrentPosition(success, error?, options?)` | 한 번 측위 | 캐시·정밀 옵션으로 지연/정확도 조정 |
| `watchPosition(success, error?, options?)` | 연속 추적 | `clearWatch(id)`로 종료 |
| `Permissions API` | 권한 상태 사전 점검 | `navigator.permissions.query({ name: 'geolocation' })` |

**옵션**
| 옵션 | 의미 | 주의 |
|---|---|---|
| `enableHighAccuracy` | 고정밀 센서 시도 | 배터리·지연 증가 |
| `timeout` | 응답 대기 한계 | 0/미설정은 브라우저 기본 |
| `maximumAge` | 캐시 허용 시간 | 0은 항상 최신 측정 |

**에러 코드**
| 코드 | 상수 | 설명 |
|---|---|---|
| 1 | `PERMISSION_DENIED` | 사용자 거부 |
| 2 | `POSITION_UNAVAILABLE` | 사용 불가 |
| 3 | `TIMEOUT` | 시간 초과 |

---

## 예제 모음 — 상황별 스니펫

### + 마지막 업데이트 UI

```html
<div id="status">-</div>
<script>
let lastTs = null;
function stamp(ts) {
  const t = new Date(ts);
  document.getElementById('status').textContent = `업데이트: ${t.toLocaleTimeString()}`;
}
navigator.geolocation.watchPosition(pos => {
  lastTs = pos.timestamp; stamp(lastTs);
  // 지도 상에서 circle 반경 pos.coords.accuracy로 업데이트...
}, handleGeoError, { enableHighAccuracy: true, maximumAge: 3000 });
</script>
```

### 입력 대체 UX

```html
<input id="manual" placeholder="구/동이나 주소를 입력">
<button id="use-address">이 위치 사용</button>
<script>
document.getElementById('use-address').onclick = async () => {
  const q = document.getElementById('manual').value.trim();
  if (!q) return;
  // 지도/지오코딩 API로 q → (lat,lon) 변환 후 이동
};
</script>
```

---

## 흔한 질문(FAQ)

- **Q. 왜 HTTP에서 안 되나요?**
  A. 위치는 민감정보라 **Secure Context(HTTPS/localhost)**에서만 허용됩니다.

- **Q. 실내에서 정확도가 왜 낮죠?**
  A. GPS 신호 감쇠/반사, Wi-Fi 측위 품질 차이. **정밀 옵션+재시도** 또는 **수동 입력**을 보완 제공하세요.

- **Q. 백그라운드로 계속 추적 가능한가요?**
  A. 브라우저/PWA는 **OS 정책으로 강하게 제한**합니다. 네이티브 앱 수준의 백그라운드 추적은 어렵습니다.

---

## 결론

- **보안(HTTPS)·권한(UX)·정확도(배터리)**의 균형이 핵심입니다.
- **`getCurrentPosition` → 캐시 응답 즉시 표시**, 이후 **정밀 재측정/연속 추적**으로 보완.
- **오류·폴백·프라이버시 고지**를 갖춘 제품 수준 UX를 구현하면, 지도·근처검색·배달·헬스·물류 등 다양한 도메인에서 **신뢰도 높은 위치 경험**을 제공할 수 있습니다.

---

## 참고 링크

- MDN: Geolocation API
- W3C Geolocation API (최신 스펙)
- Can I use: geolocation
- 각 지도/지오코딩 제공자 문서(Google Maps, Mapbox, OpenStreetMap/Leaflet 등)
