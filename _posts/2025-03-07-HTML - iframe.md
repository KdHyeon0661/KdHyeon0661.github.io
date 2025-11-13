---
layout: post
title: HTML - iframe
date: 2025-03-07 19:20:23 +0900
category: HTML
---
# HTML `<iframe>`

## 1. `<iframe>` 기본 구조와 핵심 속성

```html
<iframe
  src="https://example.com"
  width="600"
  height="400"
  title="Example 임베드"
  loading="lazy"
  referrerpolicy="no-referrer-when-downgrade"
  sandbox
></iframe>
```

| 속성 | 역할/비고 |
|---|---|
| `src` | 불러올 문서(페이지) URL |
| `width`/`height` | iframe 뷰포트 크기(픽셀/비율 단위 가능) |
| `title` | 접근성(스크린리더)용 **의미 있는 설명** 필수 |
| `loading` | `"lazy"`로 **지연 로딩**(뷰포트 근접 시 네트워크 시작) |
| `referrerpolicy` | 프레임 내 네비게이션에 전달할 **Referrer** 정책 |
| `sandbox` | 프레임 권한 제한. **보안 핵심**(토큰으로 필요한 권한만 허용) |
| `allow` | **Permissions Policy**: 카메라/마이크/지오로케이션 등 기능 허용 |
| `allowfullscreen` | 전체 화면 전환 허용(권장: `allow="fullscreen"`) |
| `name` | 타깃 네비게이션(`target="name"`)과 메시징에서 식별자 역할 |
| `srcdoc` | 외부 URL 없이 **인라인 HTML**을 직접 삽입 |

### 1.1 `srcdoc`로 인라인 문서 임베드

```html
<iframe
  title="인라인 데모"
  srcdoc="
    <!doctype html>
    <meta charset='utf-8'>
    <h1>안녕하세요</h1>
    <p>이 콘텐츠는 부모 문서 안의 srcdoc에서 왔습니다.</p>
  "
  sandbox="allow-same-origin"
  loading="lazy"
></iframe>
```

- `srcdoc`은 파일 없이 빠른 데모/문서 템플릿을 넣을 때 유용.
- `sandbox`와 결합 시 권한을 세밀히 제어.

---

## 2. 활용 예시

### 2.1 YouTube 영상 임베드

```html
<iframe
  width="560"
  height="315"
  src="https://www.youtube.com/embed/dQw4w9WgXcQ"
  title="YouTube video"
  allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; fullscreen"
  loading="lazy"
  referrerpolicy="strict-origin-when-cross-origin"
></iframe>
```

- 최신 권장: `allow="...; fullscreen"`로 전체 화면 권한 포함.

### 2.2 Google 지도 임베드

```html
<iframe
  src="https://www.google.com/maps/embed?pb=..."
  width="600" height="450" style="border:0;"
  allow="fullscreen"
  loading="lazy"
  referrerpolicy="no-referrer-when-downgrade"
  title="회사 위치 지도"
></iframe>
```

### 2.3 문서/대시보드 임베드 팁

- PDF, 대시보드 서비스(예: Looker Studio, Tableau) 등은 **서비스 제공자가 허용**해야 표시됩니다.
- 상대 사이트가 `X-Frame-Options: DENY/SAMEORIGIN` 또는 `Content-Security-Policy: frame-ancestors`로 **차단**하면 임베드 불가.

---

## 3. 레이아웃/반응형/리사이징

### 3.1 반응형 비율 상자(16:9 예)

```html
<style>
.embed {
  position: relative;
  width: 100%;
  aspect-ratio: 16 / 9; /* 현대 브라우저 */
  background: #000;
}
.embed iframe {
  position: absolute; inset: 0; width: 100%; height: 100%; border: 0;
}
</style>

<div class="embed">
  <iframe src="https://example.com" title="반응형 임베드" loading="lazy" allow="fullscreen"></iframe>
</div>
```

- 구형 브라우저는 `padding-top` 트릭(56.25%)로 대체 가능.

### 3.2 콘텐츠 높이에 맞춰 자동 리사이즈(postMessage)

자식 프레임(동일 출처 또는 메시지 합의)에서 **높이**를 부모에게 보내고, 부모가 iframe 높이를 조정.

```html
<!-- 부모 페이지 -->
<div id="wrap" style="max-width: 800px;">
  <iframe id="child" src="/child" title="자동 높이" style="width:100%; border:0;"></iframe>
</div>
<script>
const child = document.getElementById('child');
window.addEventListener('message', (e) => {
  if (e.origin !== location.origin) return; // 출처 검증
  const { type, height } = e.data || {};
  if (type === 'resize' && Number.isFinite(height)) {
    child.style.height = height + 'px';
  }
});
</script>
```

```html
<!-- /child (프레임 내부 페이지) -->
<script>
function sendHeight(){
  const h = document.documentElement.scrollHeight;
  parent.postMessage({ type: 'resize', height: h }, window.location.origin);
}
new ResizeObserver(sendHeight).observe(document.body);
window.addEventListener('load', sendHeight);
</script>
```

> 교차 출처인 경우에도 **postMessage**는 가능하지만, 반드시 `origin`을 화이트리스트로 검증하세요.

---

## 4. 보안 리스크와 대응

### 4.1 대표 리스크

- **Clickjacking**: 로그인/결제 등 중요한 버튼 위에 투명 iframe을 겹쳐 클릭 유도.
- **XSS/CSRF 전파**: 취약한 페이지를 프레임으로 임베드할 때 악성 스크립트 실행/요청 유도.
- **피싱/세션 탈취**: 프레임 내부에서 입력값 탈취/토큰 재사용.
- **Same-Origin Policy 혼동**: 동일 출처가 아니면 DOM 접근 불가. `allow-same-origin` 샌드박스는 신중히.

### 4.2 `sandbox` — 최소 권한 원칙

```html
<!-- 가장 안전: 전부 차단한 뒤 필요한 것만 허용 -->
<iframe src="https://example.com" sandbox></iframe>

<!-- 스크립트만 허용(폼/팝업 등은 여전히 차단) -->
<iframe src="https://example.com" sandbox="allow-scripts"></iframe>

<!-- 동일출처 취급 + 스크립트 (강력 권한) → 신중히 사용 -->
<iframe src="https://app.example.com" sandbox="allow-same-origin allow-scripts"></iframe>
```

자주 쓰는 토큰(대표):

| 토큰 | 의미 |
|---|---|
| `allow-scripts` | JavaScript 실행 허용(폼 제출·팝업 등은 여전히 제한) |
| `allow-same-origin` | 프레임 문서를 **동일 출처처럼** 취급 |
| `allow-forms` | 폼 제출 허용 |
| `allow-popups` | 새 창/탭 열기 허용 |
| `allow-popups-to-escape-sandbox` | 팝업에 sandbox 상속 금지(탈출) |
| `allow-modals` | `alert/prompt/confirm` 허용 |
| `allow-pointer-lock` | Pointer Lock API 허용 |
| `allow-presentation` | 프레젠테이션 API 허용 |
| `allow-top-navigation` | 최상위로의 네비게이션 허용(위험) |
| `allow-top-navigation-by-user-activation` | 사용자 상호작용 시에만 최상위 이동 |
| `allow-storage-access-by-user-activation` | Storage Access API 사용 허용 |

> 원칙: **`sandbox` 기본** + 필요한 권한만 열기. `allow-same-origin`과 `allow-scripts`를 동시에 주면 **강력 권한**이므로 꼭 필요한 경우에 한정.

### 4.3 Permissions Policy(`allow` 속성)

프레임 내부에서 사용할 수 있는 기능을 **출처/조건**과 함께 제한.

```html
<!-- geolocation은 현재 문서 출처에서만, 카메라/마이크는 금지, 전체화면만 허용 -->
<iframe
  src="https://maps.example.com"
  allow="geolocation 'self'; microphone 'none'; camera 'none'; fullscreen"
  title="지도"
></iframe>
```

- 페이지 전역 정책은 **HTTP 응답 헤더 `Permissions-Policy`**로도 설정 가능.
- `allowfullscreen` 대신 `allow="fullscreen"` 사용을 권장.

### 4.4 CSP(콘텐츠 보안 정책)

- **`frame-src`**: 이 페이지가 **어떤 출처**에서 프레임을 로드할 수 있는지.
- **`frame-ancestors`**: **이 페이지를 누가** 프레임으로 **임베드할 수 있는지**(Clickjacking 방지, 임베드 당하는 쪽).

```http
Content-Security-Policy: frame-src https://trusted.cdn.com https://player.youtube.com; frame-ancestors 'self' https://partner.example;
```

> `frame-ancestors`는 **임베드 당하는 페이지**가 설정해야 효과가 있습니다.

### 4.5 X-Frame-Options(레거시)

```http
X-Frame-Options: DENY              # 임베드 금지
X-Frame-Options: SAMEORIGIN        # 동일 출처만 허용
```

- 최신 환경은 CSP `frame-ancestors` 권장. 구형 브라우저 대비로 **겹쳐서** 쓰기도 합니다.

### 4.6 postMessage 보안 패턴

```html
<script>
window.addEventListener('message', (event) => {
  // 1) 출처 검증
  const ALLOW = new Set(['https://trusted.example', 'https://app.partner.com']);
  if (!ALLOW.has(event.origin)) return;

  // 2) 메시지 구조 점검
  const msg = event.data || {};
  if (msg.type === 'PING') {
    event.source.postMessage({ type: 'PONG', t: Date.now() }, event.origin);
  }
});
</script>
```

- 항상 **origin 화이트리스트** + **메시지 스키마 검증**(type 필드 등).

---

## 5. 접근성(a11y)·SEO·포커스

- **`title`**은 **반드시** 의미 있게 작성(“동영상 플레이어”, “회사 위치 지도” 등).
- 프레임 내부 포커스 이동은 별도 문맥이므로, **키보드 트랩**이 생기지 않도록 설계.
- SEO 측면에서 iframe 콘텐츠는 **부모 페이지의 직접 컨텐츠로 간주되지 않음**(검색 가시성에 영향 미미). 대체 요약/링크 제공 고려.
- 시각적 숨김 텍스트로 **설명 보강**:

```html
<iframe src="..." title="월간 대시보드" aria-describedby="dash-desc"></iframe>
<p id="dash-desc" class="visually-hidden">이 프레임은 월간 KPI 추이(매출, 전환, 이탈)를 시각화합니다.</p>

<style>
.visually-hidden { position:absolute; width:1px; height:1px; margin:-1px; border:0; padding:0; clip:rect(0 0 0 0); overflow:hidden; white-space:nowrap; }
</style>
```

---

## 6. 성능 최적화

- **`loading="lazy"`**로 폴드 밖 프레임의 초기 네트워크 비용 감소.
- 가능한 한 **가벼운 임베드 URL** 사용(광고/추적 스크립트 과다 페이지 지양).
- **일시 표시 상자**(스켈레톤/플레이스홀더)로 사용자 체감 개선.
- 많은 프레임은 **탭/아코디언**로 분할하여 **동적 로딩**(필요 시 `src` 세팅).

```html
<button id="load">보고서 열기</button>
<div id="host" class="embed" hidden></div>
<script>
document.getElementById('load').addEventListener('click', () => {
  const host = document.getElementById('host');
  host.innerHTML = '<iframe src="https://report.example.com" title="보고서" allow="fullscreen"></iframe>';
  host.hidden = false;
});
</script>
```

---

## 7. 동일 출처 판단과 스크립트 접근

```html
<!-- 부모 JS -->
<script>
const f = document.querySelector('iframe');
try {
  // 같은 출처이면서 sandbox에 allow-same-origin이 있을 때만 DOM 접근 가능
  console.log(f.contentWindow.document.title);
} catch (e) {
  console.warn('동일 출처가 아니거나 sandbox로 차단됨', e);
}
</script>
```

- **동일 출처**(스킴·호스트·포트가 동일) + sandbox가 DOM 접근을 막지 않아야 가능합니다.
- 교차 출처라면 **postMessage** 기반 통신만 신뢰.

---

## 8. 고급 시나리오: 결제/인증/크로스 오리진 격리

- **민감 작업(결제·인증)**: 제공 사업자의 **보안 가이드**(필수 헤더, sandbox/allow 조합)를 따르세요.
- **COOP/COEP(교차 출처 격리)**: SharedArrayBuffer 등의 고급 API가 필요하거나, 강한 프로세스 격리를 요구할 때 서버측에서 설정.
  - `Cross-Origin-Opener-Policy: same-origin`
  - `Cross-Origin-Embedder-Policy: require-corp`
  - 상호 임베드시 각 리소스의 **CORS/CORP** 처리 필요.
- **익명 프레임(credentialless)**: 일부 브라우저에서 쿠키 없이 임베드(실험/한정 지원). **브라우저 지원 확인** 후 사용.

> 이 영역은 정책/헤더 상호작용이 복잡하므로, 서비스/브라우저 공식 문서를 반드시 확인하세요.

---

## 9. 폴백/대체 링크 제공

임베드가 차단되거나 장애가 있을 때를 대비해 **대체 링크/요약**을 제공하세요.

```html
<figure>
  <div class="embed">
    <iframe src="https://charts.example.com/kpi" title="KPI 대시보드" loading="lazy" allow="fullscreen"></iframe>
  </div>
  <figcaption>
    문제가 있나요? <a href="https://charts.example.com/kpi" rel="noopener" target="_blank">새 창으로 열기</a>
  </figcaption>
</figure>
```

---

## 10. 종합 예제 — 안전한 임베드 + 통신 + 반응형

```html
<!doctype html>
<meta charset="utf-8">
<title>안전한 iframe 임베드 예제</title>
<style>
  body { font: 14px/1.6 system-ui, sans-serif; margin: 2rem; }
  .embed { position: relative; width: 100%; aspect-ratio: 16 / 9; background: #0b1220; border-radius: 12px; overflow: hidden; }
  .embed iframe { position: absolute; inset: 0; width: 100%; height: 100%; border: 0; }
  .toolbar { display: flex; gap: .5rem; margin-bottom: 1rem; }
  button { padding: .5rem .75rem; border: 1px solid #d1d5db; border-radius: .5rem; background: #fff; cursor: pointer; }
  button:focus-visible { outline: 3px solid #2563eb; outline-offset: 2px; }
</style>

<h1>안전한 iframe 임베드</h1>
<p>지연 로딩, sandbox, Permissions Policy, postMessage 통신, 반응형을 적용한 예제입니다.</p>

<div class="toolbar">
  <button id="ping">프레임에 PING 보내기</button>
  <button id="open">새 창으로 열기</button>
</div>

<div class="embed">
  <iframe
    id="frame"
    title="데모 위젯"
    src="/widget"                       <!-- 동일 출처 데모 가정 -->
    loading="lazy"
    sandbox="allow-scripts allow-same-origin"
    allow="fullscreen; geolocation 'none'; microphone 'none'; camera 'none'"
    referrerpolicy="strict-origin-when-cross-origin"
  ></iframe>
</div>

<script>
const frame = document.getElementById('frame');
const ALLOW = new Set([location.origin]); // 동일 출처만 허용

// 부모 ← 자식: 메시지 수신
window.addEventListener('message', (e) => {
  if (!ALLOW.has(e.origin)) return;
  const msg = e.data || {};
  if (msg.type === 'ready') {
    console.log('[parent] child ready', msg.version);
  } else if (msg.type === 'resize' && Number.isFinite(msg.height)) {
    frame.style.height = msg.height + 'px';
  }
});

// 부모 → 자식: 메시지 전송
document.getElementById('ping').addEventListener('click', () => {
  frame.contentWindow?.postMessage({ type: 'PING', t: Date.now() }, location.origin);
});

// 대체 열기
document.getElementById('open').addEventListener('click', () => {
  window.open(frame.src, '_blank', 'noopener');
});
</script>
```

```html
<!-- /widget (프레임 내부 페이지) -->
<!doctype html>
<meta charset="utf-8">
<title>Widget</title>
<style>
  body { margin: 0; font: 14px/1.6 system-ui, sans-serif; }
  .box { padding: 1rem; }
</style>
<div class="box">
  <h2>위젯</h2>
  <p>이 위젯은 부모와 postMessage로 통신하고, ResizeObserver로 높이를 보냅니다.</p>
</div>
<script>
function send(type, payload={}) {
  parent.postMessage({ type, ...payload }, window.location.origin);
}
send('ready', { version: '1.0.0' });

new ResizeObserver(() => {
  const h = document.documentElement.scrollHeight;
  send('resize', { height: h });
}).observe(document.body);

window.addEventListener('message', (e) => {
  if (e.origin !== window.location.origin) return;
  if ((e.data||{}).type === 'PING') {
    send('PONG', { t: Date.now() });
  }
});
</script>
```

---

## 11. 체크리스트

- [ ] 임베드 대상이 **법적/보안적으로 신뢰 가능한가?**
- [ ] `sandbox`는 기본 적용, **필요 권한만 토큰으로 허용**했는가?
- [ ] `allow`로 **Permissions Policy**를 보수적으로 제한했는가?
- [ ] 임베드 당하는 페이지는 `CSP: frame-ancestors`/`X-Frame-Options`로 정책을 명확히 했는가?
- [ ] 교차 출처 통신은 **postMessage + origin 화이트리스트**로 안전한가?
- [ ] `title`/`aria-describedby` 등 **접근성** 메타가 충분한가?
- [ ] `loading="lazy"`/동적 로딩으로 **성능**을 최적화했는가?
- [ ] 반응형(비율 상자)과 **대체 링크/폴백**을 제공했는가?

---

## 12. 요약

- `<iframe>`은 **외부 문서를 안전하게 포함**하는 표준 수단이지만, 보안/성능/접근성을 반드시 함께 고려해야 합니다.
- **보안 핵심**: `sandbox` 기본 + 최소 권한, `allow`(Permissions Policy), `CSP frame-ancestors`, 출처 검증된 `postMessage`.
- **UX/성능**: 반응형 비율 상자, 지연 로딩, 동적 삽입, 자동 리사이징.
- **접근성/SEO**: 의미 있는 `title`, 설명 연결, 폴백 링크.
- 임베드 실패 가능성을 항상 염두에 두고 **대체 경로**를 준비하세요.
