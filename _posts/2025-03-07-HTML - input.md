---
layout: post
title: HTML - input
date: 2025-03-07 22:20:23 +0900
category: HTML
---
# HTML `<input>` 태그

## `<input>`이란?

- 문서의 **단일 self-contained 입력 컨트롤**.
- 표시/동작은 `type`으로 결정(기본값: `text`).
- 닫는 태그 없음: `<input ...>`.

```html
<input type="text" name="username" placeholder="아이디" required>
```

---

## `type`별 총정리(실전 메모 포함)

### 텍스트 계열

| 타입 | 용도 | 핵심 포인트 |
|---|---|---|
| `text` | 일반 텍스트 | `maxlength`/`pattern`/`autocapitalize`/`spellcheck` |
| `password` | 비밀번호 | `autocomplete="current-password"`/`new-password"` 권장 |
| `email` | 이메일 형식 | 다중 허용: `multiple` (쉼표 구분) |
| `url` | URL 형식 | `https://` 포함 유도 |
| `search` | 검색 입력 | 모바일 엔터 라벨: `enterkeyhint="search"` |
| `tel` | 전화번호 | 형식 검사는 직접, `inputmode="tel"` |

```html
<input type="email" name="email" required placeholder="you@example.com" autocomplete="email">
<input type="password" name="pw" required minlength="8" autocomplete="current-password">
```

### 숫자/범위/날짜·시간

| 타입 | 용도 | 핵심 포인트 |
|---|---|---|
| `number` | 수치 입력 | `min`/`max`/`step`·증감버튼, 지역 소수점 유의 |
| `range` | 슬라이더 | 시각적 입력, 실제 값 표시용 `<output>` 조합 |
| `date` | 날짜 | 브라우저 UI 상이, ISO YYYY-MM-DD |
| `time` | 시간 | HH:MM |
| `datetime-local` | 날짜+시간(로컬) | 타임존 없음(주의) |
| `month`/`week` | 월/주 선택 | 리포팅/일정 주기 입력에 유용 |

```html
<label>수량 <input type="number" name="qty" min="1" max="99" step="1" value="1"></label>
<label>진행도 <input type="range" name="progress" min="0" max="100" step="5"></label>
```

> **국제화 팁**: 가격/실수 입력은 `type="number"`가 로케일 소수점 이슈를 일으킬 수 있습니다. 대안으로 `type="text" inputmode="decimal" pattern="^\d+([.,]\d+)?$"`를 고려하세요.

### 선택 계열

| 타입 | 용도 | 핵심 포인트 |
|---|---|---|
| `checkbox` | 다중 선택 | 같은 `name` → 다중 값 전송 |
| `radio` | 단일 선택 | 같은 `name` 그룹에서 1개만 선택, 반드시 라벨링 |

```html
<fieldset>
  <legend>관심사</legend>
  <label><input type="checkbox" name="interests" value="dev">개발</label>
  <label><input type="checkbox" name="interests" value="music">음악</label>
</fieldset>

<fieldset>
  <legend>수신 동의</legend>
  <label><input type="radio" name="consent" value="yes" required>예</label>
  <label><input type="radio" name="consent" value="no">아니오</label>
</fieldset>
```

### 파일/버튼/기타

| 타입 | 용도 | 핵심 포인트 |
|---|---|---|
| `file` | 파일 업로드 | `accept`/`multiple`/`capture`, 폼 `enctype="multipart/form-data"` 필수 |
| `hidden` | 비표시 값 | 변조 가능(신뢰 금지), 서버에서 검증 |
| `submit`/`reset`/`button` | 제출/초기화/일반 버튼 | 버튼별 `formaction` 등 파생 속성 지원 |
| `color` | 색상 | HEX 반환 |
| `image` | 이미지 제출 버튼 | 좌표 전송(특수 케이스) |

```html
<form action="/upload" method="POST" enctype="multipart/form-data">
  <input type="file" name="photos" accept="image/*" multiple>
  <button type="submit">업로드</button>
</form>
```

---

## 필수·중요 속성 — 작게 보이지만 큰 차이를 만든다

### 값/이름/상태

| 속성 | 설명 |
|---|---|
| `name` | 서버에 전달되는 키 |
| `value` | 초기값 |
| `checked` | 체크박스/라디오 초기 선택 |
| `placeholder` | 힌트(접근성 관점에선 라벨 대체 아님) |
| `readonly` / `disabled` | 읽기 전용(전송됨)/비활성(전송 안 됨) |

### 제약(유효성)

| 속성 | 대상 | 설명 |
|---|---|---|
| `required` | 대부분 | 필수 입력 |
| `min`/`max`/`step` | number/date 등 | 허용 범위/증분 |
| `minlength`/`maxlength` | text/password | 글자 수 범위 |
| `pattern` | text 등 | 정규표현식 제약(`title`로 안내) |

```html
<input type="text" name="zipcode" inputmode="numeric"
       pattern="^\d{5}$" title="5자리 숫자를 입력하세요" required>
```

### 자동완성·키보드·IME

| 속성 | 용도 | 예 |
|---|---|---|
| `autocomplete` | 브라우저/비밀번호 관리자 힌트 | `username`, `current-password`, `email`, `one-time-code`, `cc-number` |
| `inputmode` | 모바일 키패드 힌트 | `numeric`, `decimal`, `tel`, `email`, `url` |
| `enterkeyhint` | 가상 키보드 엔터 라벨 | `search`, `go`, `next`, `done` |
| `autocapitalize` | 첫 글자 대문자화 등(iOS) | `off`, `none`, `sentences` |

```html
<input type="text" name="otp" inputmode="numeric" autocomplete="one-time-code" enterkeyhint="done">
```

### 라우팅/멀티 폼 제어(파생 속성)

- 어느 버튼이 눌렸는지에 따라 **폼 전송 속성**을 덮어쓰기

| 버튼 속성 | 의미 |
|---|---|
| `formaction` | 전송 URL 덮어쓰기 |
| `formmethod` | `GET/POST` 덮어쓰기 |
| `formenctype` | 인코딩 덮어쓰기 |
| `formnovalidate` | 해당 제출만 검증 건너뜀 |
| `form` | 같은 문서 내 다른 위치의 `<form id="...">`와 연결 |

```html
<form id="post" action="/posts" method="POST">
  <input name="title" required>
  <button type="submit">저장</button>
  <button type="submit" formaction="/posts/preview" formnovalidate>미리보기</button>
</form>
```

### `list` + `<datalist>`

```html
<label>기술 <input name="skill" list="skills"></label>
<datalist id="skills">
  <option value="HTML"><option value="CSS"><option value="JavaScript">
</datalist>
```

### 접근성 보조

| 속성 | 설명 |
|---|---|
| `id` + `<label for>` | 기본 라벨링 |
| `aria-describedby` | 힌트/오류 텍스트 연결 |
| `aria-invalid="true"` | 오류 상태 명시 |
| `aria-required="true"` | 필수 표시(시맨틱은 `required`) |

---

## HTML5 Constraint Validation — 브라우저 내장 검증 + JS API

### 상태 의사클래스와 CSS

```html
<style>
  input:required:invalid { border: 2px solid #ef4444; }
  input:required:valid   { border: 2px solid #10b981; }
  .error { color:#b91c1c; font-size:.9rem; }
</style>

<label for="em">이메일</label>
<input id="em" type="email" required aria-describedby="em-err">
<small id="em-err" class="error" role="alert"></small>
```

### JS API

- `checkValidity()` / `reportValidity()`
- `setCustomValidity(message)` + `validity` 상태(예: `patternMismatch`, `tooShort`)

```html
<form id="f" novalidate>
  <input id="p1" type="password" minlength="8" required>
  <input id="p2" type="password" minlength="8" required>
  <button>가입</button>
</form>
<script>
const f = document.getElementById('f');
f.addEventListener('submit', e => {
  if (!f.checkValidity()) { e.preventDefault(); f.reportValidity(); return; }
  if (p1.value !== p2.value) {
    e.preventDefault();
    p2.setCustomValidity('비밀번호가 일치하지 않습니다.');
    p2.reportValidity();
    p2.addEventListener('input', () => p2.setCustomValidity(''), { once:true });
  }
});
</script>
```

---

## 파일 업로드 — UX와 보안 체크리스트

### 클라이언트 UX

```html
<input type="file" name="avatar" accept="image/*" capture="environment">
<!-- 다중 -->
<input type="file" name="docs" accept=".pdf,.doc,.docx" multiple>
```

- `accept`는 **힌트**일 뿐(우회 가능). 서버에서 재검증 필수.
- 모바일 카메라 우선 요청: `capture`(브라우저/기기 의존).

### 서버 보안

- [ ] `enctype="multipart/form-data"`
- [ ] **MIME + 매직넘버** 이중 검증(확장자 신뢰 금지)
- [ ] 업로드 디렉터리 **웹 루트 밖**
- [ ] 무작위 파일명/경로, 크기/종류/페이지 수 제한
- [ ] 이미지 처리 시 EXIF 제거, 재인코딩
- [ ] 안티바이러스/샌드박스/타임아웃

---

## 접근성(a11y) — 라벨·오류·그룹·포커스

### 라벨 연결

```html
<label for="email">이메일</label>
<input id="email" name="email" type="email" required>
```
- `placeholder`는 **라벨 대체 아님**.

### 오류 안내 링크

```html
<input id="zip" name="zip" required pattern="^\d{5}$" aria-describedby="zip-hint zip-err">
<small id="zip-hint">5자리 숫자</small>
<div id="zip-err" class="error" role="alert" aria-live="polite"></div>
```

### 그룹화

```html
<fieldset>
  <legend>연락 방법</legend>
  <label><input type="radio" name="contact" value="email" required>이메일</label>
  <label><input type="radio" name="contact" value="tel">전화</label>
</fieldset>
```

### 키보드/포커스

- 자연스러운 탭 순서 유지(레이아웃으로 바꾸지 말 것).
- 포커스 링 제거 금지(`:focus-visible` 스타일 개선은 OK).

---

## 보안 — XSS/CSRF/자동완성·민감정보

| 이슈 | 설명 | 대응 |
|---|---|---|
| XSS | 입력을 그대로 DOM/HTML에 삽입 | 서버/템플릿에서 이스케이프, CSP |
| CSRF | 의도치 않은 요청 전송 | CSRF 토큰(hidden) + 서버 검증 |
| 자동완성 민감정보 | 비밀번호/카드번호 등 | 표준 토큰 사용(`username`, `current-password`, `cc-number`), 필요한 경우 제한적 `autocomplete="off"` |
| Hidden 변조 | 클라이언트 값 신뢰 금지 | **서버 권한 검증**·서명 토큰 사용 |

```html
<!-- CSRF 토큰 포함 예 -->
<input type="hidden" name="csrf_token" value="{{token_from_server}}">
```

---

## 모바일/국제화 UX

- `inputmode="numeric|decimal|tel|email|url"`로 키패드 최적화.
- `enterkeyhint="next|done|search|go|send"`로 행동 힌트.
- 자동완성 토큰(아래)으로 입력 자동완성 신뢰도 ↑.

### 자동완성 토큰 예시(선별)

- 계정: `username`, `new-password`, `current-password`, `one-time-code`
- 연락처: `email`, `tel`, `postal-code`, `address-line1/2`, `country`
- 결제: `cc-name`, `cc-number`, `cc-exp`, `cc-csc`

```html
<input type="text" name="username" autocomplete="username">
<input type="password" name="password" autocomplete="current-password">
```

---

## 고급 패턴 — `datalist`, `output`, 실시간 계산

```html
<label>상품 <input name="product" list="products"></label>
<datalist id="products">
  <option value="Laptop"><option value="Monitor"><option value="Keyboard">
</datalist>

<label>수량 <input id="qty" type="number" min="1" value="1"></label>
<label>단가(원) <input id="price" type="text" inputmode="decimal" value="129000"></label>
합계: <output id="sum" for="qty price"></output>

<script>
function calc() {
  const q = Number(qty.value || 0);
  const p = Number((price.value || '0').replace(/[^\d.]/g,'')); // 단순 파서
  sum.value = (q * p).toLocaleString();
}
qty.addEventListener('input', calc);
price.addEventListener('input', calc);
calc();
</script>
```

---

## 종합 예제 — 회원가입(검증·접근성·모바일/보안 동시 반영)

```html
<!doctype html>
<meta charset="utf-8">
<title>회원가입</title>
<style>
  body { font: 14px/1.6 system-ui, sans-serif; margin: 2rem; }
  .grid { display:grid; grid-template-columns: 12ch 1fr; gap:.75rem 1rem; max-width:640px; }
  .grid > label { justify-self:end; margin-top:.4rem; }
  input { width:100%; padding:.5rem .75rem; border:1px solid #ccc; border-radius:.5rem; }
  input:required:invalid { border-color:#ef4444; }
  input:required:valid { border-color:#10b981; }
  .error { color:#b91c1c; font-size:.9rem; }
  :focus-visible { outline:3px solid #2563eb; outline-offset:2px; }
</style>

<h1>회원가입</h1>

<form id="reg" action="/register" method="POST" autocomplete="on" novalidate>
  <input type="hidden" name="csrf_token" value="{{server_csrf}}">

  <div class="grid" aria-describedby="form-hint">
    <label for="id">아이디*</label>
    <input id="id" name="username" type="text" required minlength="4" maxlength="20"
           autocomplete="username" spellcheck="false">

    <label for="em">이메일*</label>
    <input id="em" name="email" type="email" required autocomplete="email" enterkeyhint="next">

    <label for="pw1">비밀번호*</label>
    <input id="pw1" name="password" type="password" required minlength="8" autocomplete="new-password">

    <label for="pw2">비밀번호 확인*</label>
    <input id="pw2" type="password" required minlength="8" autocomplete="new-password">

    <label for="tel">전화</label>
    <input id="tel" name="tel" type="tel" inputmode="tel" placeholder="010-1234-5678" autocomplete="tel">

    <label for="zip">우편번호</label>
    <input id="zip" name="zip" type="text" inputmode="numeric"
           pattern="^\d{5}$" title="5자리 숫자" aria-describedby="zip-hint zip-err">
  </div>

  <small id="form-hint">별표(*)는 필수 입력입니다.</small><br>
  <small id="zip-hint">예: 06236</small>
  <div id="zip-err" class="error" role="alert" aria-live="polite"></div>

  <p style="margin-top:1rem;">
    <label><input type="checkbox" name="tos" value="yes" required> 이용약관에 동의합니다</label>
  </p>

  <p style="margin-top:1rem;">
    <button id="submitBtn" type="submit">가입</button>
    <button type="reset">초기화</button>
  </p>
</form>

<script>
const form = document.getElementById('reg');
const btn  = document.getElementById('submitBtn');
const pw1  = document.getElementById('pw1');
const pw2  = document.getElementById('pw2');
const zipErr = document.getElementById('zip-err');

form.addEventListener('submit', async (e) => {
  zipErr.textContent = '';
  // 1) 브라우저 기본 검증
  if (!form.checkValidity()) { e.preventDefault(); form.reportValidity(); return; }
  // 2) 교차 필드 검증
  if (pw1.value !== pw2.value) {
    e.preventDefault();
    pw2.setCustomValidity('비밀번호가 일치하지 않습니다.');
    pw2.reportValidity();
    pw2.addEventListener('input', () => pw2.setCustomValidity(''), { once:true });
    return;
  }
  // 3) 이중 제출 방지
  btn.disabled = true;

  // 4) Ajax 전송(프로그레시브 강화)
  e.preventDefault();
  try {
    const fd = new FormData(form);
    const res = await fetch(form.action, { method: 'POST', body: fd, credentials: 'include' });
    if (!res.ok) throw new Error('가입 실패');
    alert('가입 완료!');
  } catch (err) {
    btn.disabled = false;
    alert('오류가 발생했습니다. 다시 시도해주세요.');
  }
});
</script>
```

---

## 흔한 문제와 해결책

| 문제 | 원인 | 해결 |
|---|---|---|
| 파일 업로드 실패 | `enctype` 누락 | `multipart/form-data` 지정 |
| 소수 입력 불가/오류 | `number` 로케일 차이 | `text + inputmode="decimal"` + 서버 정규화 |
| 자동완성 오동작 | 잘못된 `autocomplete` | 표준 토큰 사용 |
| 이중 제출 | 더블클릭/네트워크 지연 | 제출 시 버튼 `disabled`, 서버 토큰 |
| 라벨 미연결 | `for/id` 누락 | `<label for="...">` 필수 |
| 보안 우회 | 클라이언트 검증만 의존 | **서버 검증** 필수 + CSP/CSRF/XSS 방어 |

---

## 요약

- `<input>`은 폼의 **핵심 단위**: 타입/속성 조합으로 대부분의 사용자 입력 처리 가능.
- **유효성**: HTML5 속성 + CSS 상태 + JS API(`setCustomValidity`)의 삼박자.
- **접근성**: 라벨·오류 연결, 그룹화, 포커스 가시성.
- **보안**: 서버 재검증·CSRF 토큰·XSS 방지·파일 업로드 방어.
- **모바일/국제화**: `inputmode`/`enterkeyhint`/자동완성 토큰으로 UX 개선.
