---
layout: post
title: CSS - DevTools
date: 2025-05-20 19:20:23 +0900
category: CSS
---
# 디버깅을 위한 DevTools(개발자 도구) 활용 가이드

## 0. DevTools 빠르게 열기 & 환경 준비

- 단축키
  - **열기**: `F12`, `Ctrl+Shift+I`(Win/Linux), `Cmd+Opt+I`(macOS)
  - **요소 선택기**: `Ctrl+Shift+C` / `Cmd+Shift+C`
  - **명령 메뉴(Command Palette)**: `Ctrl+Shift+P` / `Cmd+Shift+P`
- 기본 옵션
  - **Disable cache(개발자 도구 열려 있을 때만)**: Network 탭 상단 체크
  - **Preserve log**: 페이지 리로드 시 로그 유지
  - **Dock side**: 우측/하단 도킹 전환

---

## 1. Elements 패널 — DOM·CSS 실시간 수정의 중심

### 1.1 필수 기능
- **DOM 트리 편집**: 더블클릭/Enter로 태그/텍스트/속성 수정
- **강제 상태**: `:hov` → `:hover/:focus/:active/:visited` 강제 적용
- **클래스 토글**: `.cls` 버튼에서 클래스 추가/체크
- **Box Model**: margin/padding/border 시각화
- **Grid/Flex 시각화**: Layout 도우미로 영역/그리드 라인 표시

### 1.2 레시피: “클릭 안 되는 버튼” CSS 원인 추적
1) **요소 선택기**로 버튼 선택
2) **Styles**에서 `pointer-events: none;` 또는 `z-index`/`position` 상위 오버레이 확인
3) **Computed**에서 `pointer-events` 최종 계산값 확인
4) 문제 속성 토글/수정 후 원인 규명

### 1.3 예제: z-index·포지셔닝 충돌
```html
<div class="overlay"></div>
<button class="btn">Buy</button>
```

```css
.overlay {
  position: absolute;
  inset: 0;
  background: rgba(0,0,0,.01); /* 실수로 거의 투명 */
}
.btn { position: relative; z-index: 1; }
```

**조치**:
- `.overlay { pointer-events:none; }` 또는 오버레이 z-index/크기 조건 조정

---

## 2. Styles / Computed / Layout — CSS 충돌과 우선순위 끝장내기

- **Styles**: 규칙 정의 순서/특이성/오버라이드 흐름 확인, 체크박스로 즉시 토글
- **Computed**: 최종 계산값(색상/크기/폰트 등) 및 **상속 경로** 확인
- **Layout(Chrome)**: Flex/Grid 오버레이, gap/정렬/축 정보 시각화

### 레시피: “정렬이 틀어진 Flex 박스” 즉시 진단
- **Layout**에서 Flex 오버레이 활성화 → 메인/교차축 정렬 상태 파악
- `min-width: 0`/`min-height: 0` 누락 시 콘텐츠 넘침 → **Flex 자식**에 보정

```css
.flex { display:flex; gap:1rem; }
.flex > * { min-width: 0; } /* 긴 텍스트 줄바꿈 허용 */
```

---

## 3. Console — JS 가시성 극대화 & 순간 분석

### 3.1 실전 단축 기능
- `$0`: Elements에서 마지막 선택 요소
- `copy(obj)`: 객체를 클립보드로 복사
- `console.table(arr)`: 테이블 형태 출력
- `console.time/console.timeEnd('label')`: 구간 시간 측정
- `dir(obj)`: 속성 트리 보기

### 3.2 예제: 로그·타임라인·그룹
```js
console.group('fetch /api/products');
console.time('fetch');
const res = await fetch('/api/products');
const data = await res.json();
console.table(data.slice(0,5));
console.timeEnd('fetch');
console.groupEnd();
```

### 3.3 라이브 편집 & 일시 패치
- `Sources → Overrides`(또는 **Local Overrides**)로 CDN/정적 파일을 DevTools에서 저장 후 요청 가로채어 적용

---

## 4. Sources — 중단점 & 실행 흐름 완전 제어

### 4.1 브레이크포인트 종류
- **Line**: 특정 줄에서 정지
- **Conditional**: 조건식이 참일 때만 정지
- **Logpoint**: 정지 대신 메시지 로깅 (실행 흐름 멈추지 않음)
- **XHR/Fetch**: 특정 URL 패턴 요청 시 정지
- **Event**: `click`, `keydown` 등 이벤트 발생 시 정지
- **DOM 변경**: 속성/노드 추가·삭제 감지 시 정지

### 4.2 예제: 조건부 브레이크포인트 & 로그포인트
```js
function total(items) {
  let sum = 0;
  for (const it of items) {
    sum += it.price * it.qty; // 여기에 조건부 BP: it.qty < 0
  }
  return sum;
}
```
- 라인 번호 우클릭 → **Add conditional breakpoint…** → `it.qty < 0`
- 혹은 **Add logpoint…** → `'NEGATIVE_QTY:' , it`

### 4.3 비동기 콜스택 & Blackboxing
- **Async stack traces** 켜기: 비동기 경로 추적
- **Blackbox** 라이브러리: `node_modules`·프레임워크 코드를 스택에서 감추어 사용자 코드만 집중

---

## 5. Network — API·리소스·캐시 문제의 진원지

### 5.1 핵심 시나리오
- **CORS/인증**: 응답 헤더/상태코드 확인, `www-authenticate`/`set-cookie` 확인
- **캐시**: `cache-control`, `etag`, `age` 분석, Disable cache로 재현
- **Waterfall**: DNS→TCP→TLS→TTFB→Content Download 구간별 병목 파악

### 5.2 예제: 실패하는 파일 업로드 디버그
1) `Network`에서 `fetch`/`XHR` 필터
2) Request Payload(FormData) 구조/헤더(특히 `Content-Type`) 확인
3) 응답 본문/에러 메시지/상태코드로 서버측 원인 파악
4) CORS라면 **Response 헤더**의 `Access-Control-Allow-*` 검증

---

## 6. Performance — 레이아웃/페인트/스크립팅 병목 추적

### 6.1 절차
1) **Record** 시작 → 실사용 동작 재현(스크롤/탭 전환/검색)
2) **Stop** → 타임라인 상단에서 구간 확대
3) **Main** 스레드에서 `Recalculate Style`, `Layout`, `Paint`, `Composite` 비중 확인
4) **Bottom-Up/Call Tree**로 무거운 함수/작업 식별

### 6.2 CSS 관점 최적화 포인트
- 레이아웃 잦으면 → 변화 속성 `transform/opacity`로 전환
- 긴 리스트 → `content-visibility: auto; contain-intrinsic-size` 적용
- 애니메이션 → `will-change`를 짧게, 필요한 시점에만

```css
.card { transition: transform .2s ease, opacity .2s ease; }
.card.enter { transform: translateY(8px); opacity:0; }
.card.enter-active { transform:none; opacity:1; }
```

---

## 7. Lighthouse & Core Web Vitals — 품질 지표 자동 리포트

- **Performance/SEO/Accessibility/Best Practices** 점수와 **LCP/FID/CLS** 지표 제공
- **기능 제안(To-do)**을 항목별로 확인 → 실전 리팩토링 항목 선정
- **Throttling**(CPU/네트워크)로 저사양·저속 상황 재현

> 점수 자체보다 **항목별 개선 제안**을 실행 목록으로 삼는다.

---

## 8. Application — 스토리지·서비스워커·PWA

### 8.1 스토리지
- **Local/SessionStorage, Cookies, IndexedDB, Cache Storage** 관찰·수정·삭제
- 로그인 이슈 재현 시 **토큰/쿠키 만료**와 `SameSite/Secure/HttpOnly` 플래그 체크

```js
// 콘솔에서 임시 토큰 주입/검증
localStorage.setItem('token', 'dev-override');
document.cookie = "exp=0; path=/";
```

### 8.2 Service Worker & Cache
- **Update on reload**, **Unregister**로 구버전 SW 청소
- **Cache Storage**에서 잘못 캐싱된 응답 제거(드문 304/프리플라이트 꼬임 해결)

---

## 9. Rendering / Sensors / Layers / Coverage — 숨은 보석들

### 9.1 Rendering
- **Paint flashing**: 페인트 영역 시각화
- **Layout Shift Regions**: CLS 발생 영역 표시
- **FPS meter**: 프레임 드롭 구간 파악

### 9.2 Sensors
- **Geolocation**, **Device orientation**, **Touch**, **Network Throttling** 시뮬레이션

### 9.3 Layers
- 합성 레이어 수/경계 시각화 → 과도한 레이어(will-change 남용) 탐지

### 9.4 Coverage
- 로드된 **CSS/JS 사용률** 측정 → 미사용 바이트 제거 목록 생성
- CSS 성능 최적화의 **출발점**으로 활용

---

## 10. 보너스: DevTools로 재현 가능한 대표 버그 8선

### 10.1 Margin Collapsing
```html
<div class="parent">
  <h1 class="title">Title</h1>
</div>
```

```css
.parent { background:#eee; }
.title { margin-top:20px; }

/* 해결: 부모에 새로운 BFC 생성 */
.parent { overflow:hidden; /* 또는 display:flow-root */ }
```

### 10.2 이미지 하단 공백
```css
img { display:block; } /* 또는 vertical-align: middle; */
```

### 10.3 인라인 요소 width/height 미적용
```css
a { display:inline-block; width:120px; height:40px; }
```

### 10.4 Flex 줄바꿈 안 됨
```css
.wrap { display:flex; flex-wrap:wrap; }
```

### 10.5 Absolute 기준 엇갈림
```css
.parent { position:relative; }
.child  { position:absolute; inset:0; }
```

### 10.6 z-index 미적용
```css
.box { position:relative; z-index:10; }
```

### 10.7 iOS 100vh 이슈
```css
.hero { height: 100dvh; min-height:-webkit-fill-available; }
```

### 10.8 폼 요소 브라우저별 차이
```css
input, button {
  appearance: none;
  -webkit-appearance: none;
  font: inherit;
}
```

---

## 11. 팀 생산성을 높이는 DevTools 루틴

1) **Issue 템플릿**: “어떤 URL/해상도/브라우저/재현 절차/기대/실제/스크린샷/Har/프로파일”
2) **Command Palette**로 기능 검색 → `:screenshots`, `:show rulers`, `:coverage` 등 즉시 실행
3) **Workspace** 연결: 로컬 프로젝트와 DevTools 동기화 → 저장 시 핫 리로드
4) **CI에 Lighthouse**: 최소 점수/최대 CSS 바이트 게이트 설정

---

## 12. 실전 시나리오 3종 — 처음부터 끝까지

### 12.1 “검색창 입력 시 렉이 걸린다”
- **Performance** 녹화 → `Recalculate Style` & `Layout` 급증 확인
- **Elements→Styles**: 입력마다 바뀌는 속성이 `width/left` 등 레이아웃 트리거
- **해결**: CSS 변경을 `transform`/`opacity` 기반으로, 또는 스타일 변경 빈도 축소(디바운스)

```js
// 입력 이벤트 디바운스
input.addEventListener('input', debounce(onInput, 120));
```

### 12.2 “이미지 로딩이 매우 느리다”
- **Network**: TTFB/O(size) 확인, 큰 JPEG/PNG 탐지
- **해결**: `srcset/sizes`/AVIF/WebP 도입, **Cache-Control** 향상, `preload` 남용 지양
- **Coverage**로 CSS 미사용 바이트 제거 → 초기 페인트 앞당김

```html
<img
  src="thumb.jpg"
  srcset="img-640.avif 640w, img-1280.avif 1280w"
  sizes="(max-width: 700px) 640px, 1280px"
  alt=""
/>
```

### 12.3 “로그인 후 옛 UI가 계속 보인다”
- **Application**: Service Worker **Unregister/Update on reload**, Cache Storage 비우기
- **Network**: `Cache-Control`/ETag/`Vary` 설정 검토
- **해결**: SW 버전 전략/정적 자산 해시(`app.a1b2.css`) 도입

---

## 13. 자주 쓰는 콘솔 스니펫 모음

```js
// 1) 이벤트 리스너 목록 확인
getEventListeners($0);

// 2) DOM 변화 감시
const obs = new MutationObserver(console.log);
obs.observe($0, { attributes:true, childList:true, subtree:true });

// 3) 성능 마킹
performance.mark('start');
// ... some heavy work ...
performance.mark('end');
performance.measure('work', 'start', 'end');
console.table(performance.getEntriesByType('measure'));

// 4) 네트워크 재시도 래퍼 디버그
async function retry(fn, n=3){
  let err;
  for(let i=0;i<n;i++){
    try { return await fn(); } catch(e){ err=e; console.warn('retry',i+1); }
  }
  throw err;
}
```

---

## 14. 단축키·팁 총정리

| 항목 | 단축키/팁 |
|---|---|
| 요소 선택 | `Ctrl+Shift+C` / `Cmd+Shift+C` |
| 커맨드 팔레트 | `Ctrl+Shift+P` / `Cmd+Shift+P` |
| 다음/이전 요소 선택 | 위/아래 화살표 |
| 콘솔 멀티라인 | `Shift+Enter` |
| 네트워크 로그 유지 | Preserve log |
| 캐시 끄기 | Disable cache |
| 파일 열기 | `Ctrl+P` / `Cmd+P` |
| 파라미터 캡쳐 | Network → Headers/Preview |
| 리팩 전 스냅 | Performance screenshot/filmstrip |

---

## 15. 체크리스트 요약

- DOM/CSS: **Elements + Styles/Computed/Layout**로 충돌/우선순위/겹침 해소
- JS: **Console/Sources**로 로그·중단점·흐름 제어
- 네트워크: **Network**로 실패/캐시/워터폴 분석
- 성능: **Performance/Lighthouse**로 LCP/CLS 병목 제거
- 스토리지/PWA: **Application**로 토큰/쿠키/SW/캐시 관리
- 보조 패널: **Rendering/Sensors/Layers/Coverage**로 페인트·장치·레이어·사용률 점검

---

## 참고 링크

- Chrome DevTools 문서
- Google Web Fundamentals — Performance/Debugging
- Firefox DevTools 가이드

> DevTools는 **측정 → 가설 → 수정 → 재측정**의 반복이 핵심이다.
> 이 글의 레시피들을 팀의 **디버깅 플레이북**으로 재사용해, 이슈를 “재현 가능한 형태”로 남기고 **같은 문제를 두 번 고치지 말자.**
