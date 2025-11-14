---
layout: post
title: HTML - input 타입 정리
date: 2025-03-26 21:20:23 +0900
category: HTML
---
# HTML5에서 추가된 `<input>` 타입 정리 – 총 12가지 완전 해설

## HTML5로 추가된 12가지 타입 한눈에 보기

1) `number` — 숫자
2) `range` — 슬라이더
3) `color` — 색상 선택
4) `date` — 날짜
5) `time` — 시간
6) `datetime-local` — 날짜+시간(로컬)
7) `month` — 연-월
8) `week` — 연-주(ISO)
9) `email` — 이메일
10) `url` — 웹 주소
11) `tel` — 전화번호
12) `search` — 검색

> **원칙:** 클라이언트 검증은 **편의**일 뿐, **최종 검증은 반드시 서버**에서 수행한다.

---

## 공통 준비 코드(데모 프레임)

아래 템플릿에 원하는 섹션만 끼워 넣어 실험할 수 있다.

```html
<!DOCTYPE html>
<html lang="ko">
<head>
<meta charset="UTF-8" />
<meta name="viewport" content="width=device-width, initial-scale=1" />
<title>HTML5 Input Types Playground</title>
<style>
  body { font-family: system-ui, -apple-system, Segoe UI, Roboto, sans-serif; margin: 2rem; line-height: 1.6; }
  fieldset { margin-bottom: 1.5rem; padding: 1rem; border: 1px solid #ddd; border-radius: .5rem; }
  label { display: block; margin: .5rem 0 .25rem; font-weight: 600; }
  .row { display: flex; gap: .75rem; align-items: center; flex-wrap: wrap; }
  .hint { color: #666; font-size: .9rem; }
  output { padding: .2rem .5rem; background: #f4f6fb; border-radius: .25rem; }
  .error { color: #b00020; }
</style>
</head>
<body>

<h1>HTML5 Input Types Demo</h1>

<!-- 여기에 각 섹션 코드 조립 -->

</body>
</html>
```

---

## `type="number"` — 숫자만 입력

### 핵심

- **숫자 전용** 입력. 브라우저가 **스핀 버튼(▲▼)** 제공.
- `min`, `max`, `step`으로 범위/증분 제어.
- **주의:** 일부 브라우저/OS에서 마우스 휠로 값이 변할 수 있음(의도치 않은 변경).

```html
<fieldset>
  <legend>number</legend>
  <label for="qty">수량(1~100, 5단위)</label>
  <input id="qty" name="qty" type="number" min="1" max="100" step="5" inputmode="numeric" autocomplete="off" />
  <p class="hint">스핀/키보드로 조정. 모바일 키보드는 숫자 전용으로 유도합니다.</p>
</fieldset>

<script>
  // 마우스 휠로 값이 바뀌지 않도록 포커스 상태에서만 입력 허용(선택)
  const qty = document.getElementById('qty');
  qty.addEventListener('wheel', (e) => {
    if (document.activeElement !== qty) e.preventDefault();
  }, { passive: false });
</script>
```

**검증 팁**
- 음수/소수/공란 등 엣지 케이스를 **서버에서 재검증**.
- `step="any"`로 소수 허용 가능.

---

## `type="range"` — 슬라이더 형태로 숫자 선택

### 핵심

- 슬라이더 UI. 값은 **숫자**지만 시각적으로 조절.
- 즉시 피드백을 위해 `<output>`과 함께 사용 권장.

```html
<fieldset>
  <legend>range</legend>
  <label for="vol">볼륨(0~100)</label>
  <div class="row">
    <input id="vol" type="range" min="0" max="100" step="1" value="50" oninput="volOut.value = vol.value" />
    <output id="volOut" name="volOut" for="vol">50</output>
  </div>
</fieldset>
```

**UX 팁**
- 슬라이더만으로는 정확한 값 입력이 어려울 수 있으므로 **동일 값을 number와 동기화**하는 패턴도 자주 사용.

---

## `type="color"` — 색상 선택기

### 핵심

- 시스템/브라우저 기본 **컬러 피커** 제공.
- 값은 `#RRGGBB` 16진수.

```html
<fieldset>
  <legend>color</legend>
  <label for="favcolor">좋아하는 색</label>
  <input id="favcolor" type="color" value="#1e90ff" />
  <span id="colorHex">#1e90ff</span>
</fieldset>

<script>
  const color = document.getElementById('favcolor');
  const hex = document.getElementById('colorHex');
  color.addEventListener('input', () => { hex.textContent = color.value; });
</script>
```

**주의**
- 일부 구형 브라우저에서 **텍스트로 대체**되기도 하므로, **PE** 관점에서 폴백 허용.

---

## `type="date"` — 날짜 선택

### 핵심

- 달력 위젯 제공. 값은 **`YYYY-MM-DD`**.
- `min`/`max`로 범위 제한.

```html
<fieldset>
  <legend>date</legend>
  <label for="dob">생년월일</label>
  <input id="dob" type="date" name="dob"
         min="1900-01-01" max="2099-12-31" autocomplete="bday" />
  <p class="hint">서버에서는 반드시 유효 날짜(존재하는 달/일) 재검증.</p>
</fieldset>
```

**국제화**
- 표시는 로케일에 따라 달라져도 **전송 값 포맷은 고정**.

---

## `type="time"` — 시간 선택

### 핵심

- 시간 전용. `step`으로 **초 단위** 제어 가능.
- 로케일에 따라 12/24시간 **표현이 다를 수 있으나 값은 동일 포맷**.

```html
<fieldset>
  <legend>time</legend>
  <label for="appt">예약 시간</label>
  <input id="appt" type="time" name="appt" step="60" autocomplete="off" />
  <p class="hint">step=60 → 분 단위, step=1 → 초 단위 입력 허용.</p>
</fieldset>
```

---

## `type="datetime-local"` — 날짜 + 시간(로컬)

### 핵심

- 로컬 타임존 기준. **UTC가 아니다**.
- 시간대 혼동이 잦으므로, 서버 저장 시 **UTC 변환**을 강력 권장.

```html
<fieldset>
  <legend>datetime-local</legend>
  <label for="meet">미팅(로컬)</label>
  <input id="meet" type="datetime-local" name="meet" />
  <p class="hint">서버에서는 수신 즉시 타임존 정보와 함께 UTC로 변환 저장.</p>
</fieldset>
```

**주의(반패턴)**
- 국제 서비스에서 `datetime-local`을 그대로 저장/비교하면 **타임존 혼선**.
- **대안:** 날짜/시간을 별 필드(+타임존)로 분리해 전송하거나, 클라이언트에서 **ISO-8601(UTC)** 로 전송.

---

## `type="month"` — 연월(YYYY-MM)

### 핵심

- **월 단위** 선택. 결제 주기/월간 리포트에 적합.

```html
<fieldset>
  <legend>month</legend>
  <label for="billing">청구 월</label>
  <input id="billing" type="month" name="billing" min="2020-01" />
</fieldset>
```

---

## `type="week"` — 연-주(ISO Week)

### 핵심

- 값 형식 **`YYYY-W##`** (예: `2025-W29`).
- ISO-8601 기준: **첫 주는 그 해의 첫 목요일을 포함한 주**.

```html
<fieldset>
  <legend>week</legend>
  <label for="sprint">스프린트(ISO 주)</label>
  <input id="sprint" type="week" name="sprint" />
  <p class="hint">서버 변환 로직에서 ISO 주 → 실제 날짜 범위로 해석.</p>
</fieldset>
```

---

## `type="email"` — 이메일 형식 검사

### 핵심

- `@`와 도메인 구조 등 기본 형식을 **브라우저가 검증**.
- `multiple`로 **쉼표/공백 구분** 다중 입력 가능.

```html
<fieldset>
  <legend>email</legend>
  <label for="email">이메일</label>
  <input id="email" type="email" name="email" required autocomplete="email" />
  <p class="hint">기본 형식만 확인하므로, 서버에서 MX 등 정교 검증 추천.</p>

  <label for="emails">참조 이메일(다중)</label>
  <input id="emails" type="email" name="emails" multiple placeholder="a@x.com, b@y.com" />
</fieldset>
```

**주의**
- `type=email`은 **호스트·TLD 형식만** 거른다. Disposable/존재 확인은 서버 영역.

---

## `type="url"` — URL 형식 입력

### 핵심

- `http://` 또는 `https://` 등의 **스킴 포함 URL** 형식을 검사.
- 스킴 없는 `example.com`은 **유효하지 않음**.

```html
<fieldset>
  <legend>url</legend>
  <label for="homepage">홈페이지</label>
  <input id="homepage" type="url" name="homepage" required placeholder="https://example.com" autocomplete="url" />
  <p class="hint">스킴 필수. 서버에서 스킴 화이트리스트/리다이렉트 검증 권장.</p>
</fieldset>
```

---

## `type="tel"` — 전화번호

### 핵심

- **형식 검사를 하지 않는다**(기본). 자유 입력.
- 모바일에서 **전화 키패드**를 띄우는 효과.
- 검증은 `pattern` 또는 서버에서 수행.

```html
<fieldset>
  <legend>tel</legend>
  <label for="phone">전화번호(예: 010-1234-5678)</label>
  <input id="phone" type="tel" name="phone"
         inputmode="tel" autocomplete="tel-national"
         pattern="[0-9]{2,4}-[0-9]{3,4}-[0-9]{4}"
         placeholder="010-1234-5678"
         title="예: 010-1234-5678 형식" />
  <p class="hint">국가별 규칙 상이 → 서버에서 E.164(+821012345678) 등으로 정규화 권장.</p>
</fieldset>
```

---

## `type="search"` — 검색 입력(시맨틱)

### 핵심

- 기능은 `text`와 유사하나, **검색 UX(지우기 X 버튼 등)** 를 제공.
- 모바일 키보드의 **검색 전용 버튼**을 유도(`enterkeyhint="search"`).

```html
<fieldset>
  <legend>search</legend>
  <label for="q">검색</label>
  <input id="q" name="q" type="search" placeholder="검색어" enterkeyhint="search" autocomplete="off" />
  <button type="submit">검색</button>
</fieldset>
```

---

## 공통 속성/검증/UX 향상

### A) 필수/범위/정규식

```html
<input type="number" required min="0" max="100" step="10" />
<input type="text" required pattern="^[a-zA-Z0-9_]{4,16}$" title="영문/숫자/언더스코어 4~16자" />
```

### B) 자동완성(autocomplete) 토큰(발췌)

- `name`, `email`, `username`, `new-password`, `current-password`,
  `tel`, `tel-national`, `one-time-code`,
  `street-address`, `address-line1`, `postal-code`,
  `bday`, `bday-day`, `bday-month`, `bday-year`,
  `url`, `organization`, `cc-number`, `cc-exp`, `cc-csc`, …
```html
<input type="email" name="email" autocomplete="email" />
<input type="password" name="password" autocomplete="current-password" />
```

### C) 모바일 최적화

- `inputmode` : `numeric`, `decimal`, `tel`, `email`, `url`, `search`
- `enterkeyhint` : `search`, `go`, `done`, `next`, `send`, …

```html
<input type="text" inputmode="numeric" enterkeyhint="next" />
```

### D) 접근성(a11y)

- `label for` ↔ `id` 연결은 **기본 중 기본**.
- 동적 결과는 `aria-live="polite"` 등으로 공지.
- 에러는 **연결된 설명 영역**에 표시하고 `aria-invalid="true"` 설정.

```html
<label for="zip">우편번호</label>
<input id="zip" name="zip" type="text" pattern="\\d{5}" aria-describedby="zip-help" />
<small id="zip-help" class="hint">5자리 숫자</small>
```

### E) 클라이언트-서버 검증 협업

- **클라:** 즉시 피드백(형식/범위) 제공.
- **서버:** 모든 값 재검증(권한/비즈니스 규칙/중복/위조).

---

## 실전 시나리오 1 — 예약 폼(다양 타입 + `<output>`)

```html
<form id="booking" method="post" action="/api/booking" onsubmit="return validateBooking()">
  <fieldset>
    <legend>예약 정보</legend>
    <label for="name">이름</label>
    <input id="name" name="name" type="text" required autocomplete="name" />

    <label for="email2">이메일</label>
    <input id="email2" name="email" type="email" required autocomplete="email" />

    <div class="row">
      <div>
        <label for="date">날짜</label>
        <input id="date" name="date" type="date" required min="2025-01-01" />
      </div>
      <div>
        <label for="time">시간</label>
        <input id="time" name="time" type="time" required step="900" />
      </div>
    </div>

    <div class="row">
      <div>
        <label for="people">인원(1~8)</label>
        <input id="people" name="people" type="number" required min="1" max="8" value="2" />
      </div>
      <div>
        <label for="price">1인 단가(원)</label>
        <input id="price" name="price" type="number" required min="0" step="1000" value="15000" />
      </div>
    </div>

    <p>예상 결제 금액: <output id="total" name="total" for="people price">0</output> 원</p>
    <p id="msg" role="status" aria-live="polite"></p>

    <button type="submit">예약 요청</button>
    <p id="err" class="error" aria-live="assertive"></p>
  </fieldset>
</form>

<script>
  const people = document.getElementById('people');
  const price  = document.getElementById('price');
  const total  = document.getElementById('total');
  const msg    = document.getElementById('msg');
  const err    = document.getElementById('err');

  function recalc() {
    const p = Number(people.value);
    const u = Number(price.value);
    if (Number.isFinite(p) && Number.isFinite(u) && p >= 1 && u >= 0) {
      const sum = p * u;
      total.value = sum.toLocaleString('ko-KR');
      msg.textContent = `현재 금액은 ₩${total.value} 입니다.`;
      err.textContent = '';
    } else {
      total.value = '0';
      msg.textContent = '';
      err.textContent = '입력 값을 확인하세요.';
    }
  }
  people.addEventListener('input', recalc);
  price.addEventListener('input', recalc);
  recalc();

  function validateBooking() {
    // 클라이언트 일차 확인 (서버가 최종 검증)
    if (!document.getElementById('name').value.trim()) { err.textContent = '이름을 입력하세요.'; return false; }
    err.textContent = '';
    return true;
  }
</script>
```

---

## 실전 시나리오 2 — 연락처 폼(autocomplete + inputmode + pattern)

```html
<form method="post" action="/api/contact">
  <fieldset>
    <legend>연락처</legend>
    <label for="contact-name">이름</label>
    <input id="contact-name" name="name" type="text" autocomplete="name" required />

    <label for="contact-email">이메일</label>
    <input id="contact-email" name="email" type="email" autocomplete="email" required />

    <label for="contact-tel">전화번호</label>
    <input id="contact-tel" name="tel" type="tel" inputmode="tel"
           autocomplete="tel-national"
           pattern="[0-9]{2,4}-[0-9]{3,4}-[0-9]{4}"
           placeholder="010-1234-5678" />
    <button type="submit">보내기</button>
  </fieldset>
</form>
```

---

## 기능 감지 & 폴백(프로그레시브 인행스먼트)

브라우저가 특정 타입을 지원하지 않으면 **text로 대체**된다. 간단히 감지 후 **폴백 UI**를 제공할 수 있다.

```html
<script>
// input type 지원 감지
function isTypeSupported(type) {
  const ipt = document.createElement('input');
  ipt.setAttribute('type', type);
  return ipt.type === type;
}

if (!isTypeSupported('date')) {
  // 구형 브라우저: date polyfill 안내/라이브러리 로딩
  console.warn('date type not supported: load a datepicker polyfill if needed');
}
</script>
```

---

## 보안/반패턴/주의사항

1) **서버 검증 필수**
   - `number`, `email`, `url` 등은 **형식을 돕는 도구**. 금액/재고/권한 등 **비즈니스 로직은 서버에서**.

2) **`datetime-local`와 타임존**
   - 저장/비교는 **UTC**. 사용자 표시만 로컬로 포맷.

3) **마우스 휠 오작동**
   - `number` 포커스 아웃 시 휠 값 변경 방지(위 예제 참고).

4) **`tel` 형식 강제 금지**
   - 국가별 규칙과 기업 번호 형식이 다르므로, **지나치게 엄격한 정규식은 UX 저해**.
   - 서버에서 E.164 표준으로 정규화/검사 추천.

5) **`url`의 리다이렉트 취약점**
   - 리다이렉트/프록시/콜백 URL은 **화이트리스트**로 제한.

6) **자동완성/비번 필드**
   - `autocomplete="new-password"`/`current-password` 정확히 표기.
   - 임의 차단(`autocomplete="off"`)은 브라우저 정책에 따라 무시될 수 있음.

7) **마스킹 vs 검증**
   - 입력 마스킹(예: 010-xxxx-xxxx)은 **검증이 아니라 포맷 보조**. 서버에서 숫자만 추출하여 검증/정규화.

---

## 📱 모바일 최적화 치트시트

| 목적 | 권장 타입 | 보조 속성 |
|---|---|---|
| 순수 숫자 | `number` or `text` | `inputmode="numeric"`, `pattern="[0-9]*"` |
| 소수점 숫자 | `text` | `inputmode="decimal"` |
| 전화번호 | `tel` | `inputmode="tel"`, `autocomplete="tel-national"` |
| 이메일 | `email` | `autocomplete="email"` |
| URL | `url` | `autocomplete="url"` |
| 검색 | `search` | `enterkeyhint="search"` |

> 일부 환경에서 `number`는 스핀 버튼/휠 이슈가 있어 **엄격 제어가 필요**하면 `text + inputmode` 조합이 실무적으로 더 낫기도 하다.

---

## 브라우저 지원 요약(2025)

- `number`, `range`, `color`, `date`, `time`, `datetime-local`, `month`, `week`, `email`, `url`, `tel`, `search`
  → **현대 브라우저 전반 지원**, 구형 IE 등은 일부 미지원 → **PE로 안전하게 폴백**.

---

## 요약 테이블(기능/검증/주의)

| 타입 | 형식검사 | 주 UI | 흔한 옵션 | 주의/팁 |
|---|---|---|---|---|
| number | 숫자 | 스핀/키보드 | min/max/step | 휠 변경, step=any 소수 |
| range | 숫자 | 슬라이더 | min/max/step | `<output>`와 동기화 |
| color | #RRGGBB | 컬러피커 | value | 구형 폴백 |
| date | YYYY-MM-DD | 달력 | min/max | 로케일표시 vs 전송값 |
| time | HH:MM(:SS) | 시간선택 | step | 초 단위 step |
| datetime-local | YYYY-MM-DDThh:mm | 날짜+시간 | min/max | **UTC 아님**, 서버 변환 |
| month | YYYY-MM | 월선택 | min/max | 월 기준 로직 |
| week | YYYY-W## | ISO 주 | min/max | 주→날짜범위 변환 |
| email | 이메일 | 텍스트 | multiple, required | 서버 MX/실존 확인 |
| url | 스킴 포함 URL | 텍스트 | required | 스킴필수, 리다이렉트 검증 |
| tel | 자유 | 전화 키패드 | pattern | 국제 규칙 다양 |
| search | 텍스트 | 검색 스타일 | enterkeyhint | 지우기 버튼 등 UX |

---

## 부록: 실무 폼 베이스라인(검증/상태/접근성)

```html
<form id="base" novalidate>
  <fieldset>
    <legend>폼 베이스라인</legend>

    <label for="username">아이디</label>
    <input id="username" name="username" type="text" required minlength="4" maxlength="16"
           pattern="^[a-zA-Z0-9_]{4,16}$" autocomplete="username" />
    <small id="uhelp" class="hint">영문/숫자/언더스코어 4~16자</small>

    <label for="pw">비밀번호</label>
    <input id="pw" name="pw" type="password" required minlength="8" autocomplete="new-password" />

    <button type="submit" id="submitBtn">제출</button>
    <p id="formErr" class="error" role="alert" aria-live="assertive"></p>
  </fieldset>
</form>

<script>
  const f = document.getElementById('base');
  const err = document.getElementById('formErr');
  const submitBtn = document.getElementById('submitBtn');

  f.addEventListener('submit', (e) => {
    // 클라이언트 선검증
    if (!f.checkValidity()) {
      e.preventDefault();
      err.textContent = '입력값을 다시 확인하세요.';
      // 브라우저 기본 검증 UI를 활용하려면 reportValidity 사용
      f.reportValidity();
      return;
    }
    // 중복 제출 방지
    submitBtn.disabled = true;
  });
</script>
```

- `novalidate`를 제거하면 **브라우저 기본 검증 UI**를 우선 사용.
- 커스텀 메시지를 원하면 `setCustomValidity`/`reportValidity` 조합.

---

## 마무리

- HTML5 `<input>` 타입은 **UX·모바일·기본 검증**을 크게 향상시킨다.
- 그러나 **최종 검증/권한/정합성은 서버 책임** — 금액/시간대/주차/URL 리다이렉트 등은 반드시 서버 검증.
- 국제 서비스에서는 **시간/날짜의 타임존 처리**를 특히 주의(저장은 UTC, 표시는 로컬).
- 모바일 최적화는 `inputmode`/`enterkeyhint`/`autocomplete`로 완성도를 끌어올린다.
- 폴백/프로그레시브 인행스먼트 관점에서 설계하면 **구형 환경에서도 우아한 저하**가 가능하다.

> 익숙한 `text` 하나로 모두 처리하기보다, **목적에 맞는 타입**을 고르는 것이 **사용성·정확성·유지보수성**을 동시에 높이는 지름길이다.
