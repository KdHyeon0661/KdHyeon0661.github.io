---
layout: post
title: HTML - input 요소 속성
date: 2025-03-30 19:20:23 +0900
category: HTML
---
# HTML & HTML5의 `<input>` 요소 속성

## 0. 제출(submit)과 전송(payload)의 기본 원리 한 장 요약

- 폼 전송 시 **서버로 전송되는 키**는 `name` 속성이다. `id`는 **전송되지 않는다**(라벨 연결/JS 선택용).  
- 전송 포함 조건:
  - 포함: `name`이 있고, `disabled`가 **아닌** 필드
  - 제외: `disabled`(항상 제외), `type="submit"/"button"/"reset"`(컨트롤), 체크되지 않은 `checkbox/radio`  
- `readonly`는 **전송된다**(값 수정 불가일 뿐).  
- `disabled`는 **완전히 무시**(전송 제외 + 포커스 불가 + CSS 접근성 고려 필요).

---

## 1. HTML 기본 `<input>` 속성 (핵심 & 차이점 포함)

### 1.1 `name` (전송 키)
```html
<input type="text" name="username" value="kimdh">
```
- 서버는 `username=kimdh`로 수신. **폼 데이터 설계의 최중요 속성**.

### 1.2 `value` (현재값/초깃값)
```html
<input type="text" name="nickname" value="기본값">
```
- 초기 렌더 시 값. 사용자 입력/JS로 수정되면 현재값으로 반영되어 전송.

### 1.3 `readonly` (읽기 전용, 전송됨)
```html
<input type="text" name="orderId" value="ORD-123" readonly>
```
- 사용자는 수정 불가. **전송에는 포함**(잠금된 참조값 제공 시 유용).

### 1.4 `disabled` (비활성, 전송 제외)
```html
<input type="text" name="debug" value="hidden" disabled>
```
- 포커스/탭 이동 불가, 전송 또한 **완전히 제외**. 접근성상 시각적 표시와 안내 필요.

### 1.5 `maxlength` / `minlength`
```html
<input type="text" name="code" minlength="6" maxlength="6" required>
```
- `minlength`는 HTML5. **제출 시 검증**(입력 중에는 안내문구로 보조).

### 1.6 `size` (가시 너비, 문자 단위)
```html
<input type="text" size="30" placeholder="가시 너비만 조절">
```
- **표시 폭**일 뿐, 실제 입력 길이 제한 아님(제한은 `maxlength`로).

### 1.7 `type`
- 유명 타입: `text`, `password`, `email`, `number`, `url`, `tel`, `search`, `date`, `time`, `datetime-local`, `month`, `week`, `range`, `color`, `checkbox`, `radio`, `file`, `hidden`, `submit`, `reset`, `button`, `image`.

> 타입별 유효성/키보드/모바일 UX가 달라진다(아래 “유효성 검증” & “모바일 UX” 참고).

---

## 2. HTML5/현대 브라우저 확장 속성 (입력 UX/검증/연결)

### 2.1 `placeholder` (힌트)
```html
<input type="text" name="query" placeholder="검색어를 입력하세요">
```
- 라벨을 대체하지 않는다(접근성상 **라벨 필수**, 플레이스홀더는 힌트/예시).

### 2.2 `required` (필수)
```html
<input type="email" name="email" required>
```
- 제출 시 공란이면 브라우저 기본 오류(Constraint Validation).

### 2.3 `pattern` (정규식)
```html
<input type="text" name="initials"
       pattern="[A-Za-z]{2,3}" title="영문 대/소문자 2~3자">
```
- 암시적 앵커(`^…$`)가 붙은 것처럼 **전체 일치** 기준.
- 사용자 안내 문구는 `title`로 제공.

### 2.4 `min` / `max` / `step` (수치/날짜/시간)
```html
<input type="number" name="qty" min="1" max="10" step="1" value="1">
<input type="date" name="start" min="2025-01-01" max="2025-12-31">
<input type="time" name="when" step="1"> <!-- 초 단위 허용 -->
```
- `step`은 기본 값 1. 소수/초 단위는 `step="0.1"`, `step="1"` 등으로 제어.

### 2.5 `multiple` (다중 값)
```html
<input type="email" name="recipients" multiple placeholder="쉼표로 여러 이메일">
<input type="file" name="photos" multiple accept="image/*">
```

### 2.6 `list` + `<datalist>` (제안 목록)
```html
<input type="text" name="fruit" list="fruits">
<datalist id="fruits">
  <option value="Apple"><option value="Banana"><option value="Cherry">
</datalist>
```
- **제안**일 뿐 강제 선택 아님(자유 입력 가능).

### 2.7 `autocomplete` (자동완성) — **토큰 기반 제어**
```html
<!-- 폼 수준 전체 끄기 -->
<form autocomplete="off">

<!-- 필드별 토큰(권장): username/password, email, one-time-code 등 -->
<input type="text"   name="username" autocomplete="username">
<input type="password" name="password" autocomplete="current-password">
<input type="text"   name="otp"      autocomplete="one-time-code">
```
- 토큰 예: `name`, `email`, `username`, `new-password`, `current-password`,  
  주소 계열 `address-line1`, `address-level1`(도/광역), `postal-code`, `country`.  
- **암호관리자/브라우저와의 협업을 위해 정확한 토큰을 적극 사용**.

### 2.8 `autofocus` (자동 포커스)
```html
<input type="search" name="q" autofocus>
```
- 문서당 실질적으로 **하나만** 의미 있음(여러 개면 브라우저가 임의 선택).

### 2.9 `inputmode` (모바일 키보드 힌트)
```html
<input type="text" name="amount" inputmode="numeric" pattern="\d*">
```
- 값: `text`, `numeric`, `decimal`, `tel`, `email`, `url`, `search`.  
- 타입과 별개로 **가상 키보드 레이아웃** 힌트를 주어 UX 개선.

### 2.10 `enterkeyhint` (모바일 Enter 라벨)
```html
<input type="search" enterkeyhint="search" placeholder="검색어">
```
- 값: `enter`, `done`, `go`, `next`, `previous`, `search`, `send`.  
- 모바일 키보드의 리턴/전송 버튼 라벨을 UX적으로 맞춤.

### 2.11 `spellcheck` / `autocapitalize` (텍스트 교정)
```html
<input type="text" name="title" spellcheck="true" autocapitalize="sentences">
```
- 플랫폼/브라우저 지원 가변. **라벨/문장 입력 필드**에 주로 유용.

### 2.12 `dirname` (텍스트 방향성 전송)
```html
<input type="text" name="comment" dir="auto" dirname="comment.dir">
```
- 서버로 `comment`와 함께 `comment.dir=ltr|rtl` 제출(국제화 폼에서 유용).

---

## 3. `<form>` 연계 속성 (폼 외부·버튼별 제어)

### 3.1 `form` (연결) — 폼 밖의 필드/버튼을 특정 폼에 연결
```html
<form id="reg" action="/register" method="post"></form>

<!-- 문서 어디서든 form 속성으로 연결 가능 -->
<input type="text" name="email" form="reg" placeholder="이메일">
<input type="password" name="pw" form="reg" placeholder="비밀번호">

<!-- 제출 버튼도 외부에서 연결 -->
<input type="submit" value="가입" form="reg">
```

### 3.2 버튼 전용 `formaction` / `formmethod` / `formenctype` / `formnovalidate` / `formtarget`
```html
<form id="f" action="/submit" method="post">
  <input type="text" name="q" required>
  <input type="submit" value="기본 제출">
  <input type="submit" value="새 창 검색"
         formaction="/search" formmethod="get" formtarget="_blank">
  <input type="submit" value="검증 생략"
         formnovalidate>
</form>
```
- 버튼마다 **제출 대상/방법/인코딩/검증 여부/창 대상**을 **세분화** 가능.
- `type="image"`(이미지 버튼)에도 적용 가능(`height`/`width`/`alt` 병행).

---

## 4. 타입별 특화 속성 (파일/이미지/비밀번호 등)

### 4.1 파일 업로드: `type="file"` — `accept`/`multiple`/`capture`
```html
<!-- 이미지/동영상만 -->
<input type="file" name="media" accept="image/*,video/*">

<!-- 여러 파일 -->
<input type="file" name="photos" accept="image/*" multiple>

<!-- 모바일에서 카메라 선호(UX 힌트) -->
<input type="file" name="shoot" accept="image/*" capture="environment">
```
- **보안상** `value`는 설정 불가(JS로도). 선택은 **사용자만** 가능.
- `capture`는 모바일 브라우저에서 **카메라 앱 선호** 힌트(필수 아님).

### 4.2 비밀번호: `type="password"` — 자동완성 토큰/보안
```html
<input type="password" name="current" autocomplete="current-password" required>
<input type="password" name="new" autocomplete="new-password" minlength="8">
```
- 암호관리자/브라우저와 **협력**하려면 토큰을 정확히.  
- 민감 정보라도 `autocomplete="off"`를 강제 무시하는 브라우저가 있음(보안은 **서버에서**).

### 4.3 이미지 버튼: `type="image"` — 좌표 제출
```html
<input type="image" src="/img/submit.png" alt="제출" height="40" width="120">
```
- 클릭 좌표가 `name.x`, `name.y`로 함께 전송됨(특수 케이스에서만 사용 권장).

---

## 5. 유효성 검사(Constraint Validation) — HTML만으로도 강력

### 5.1 기본 흐름
- 브라우저는 `required`, `type`, `min/max/step`, `pattern`, `minlength/maxlength` 등을 종합해 **제출 시 자동 검증**.
- 실패 시 **기본 툴팁** + 해당 필드 포커스.

### 5.2 CSS 의사클래스
```css
input:required:invalid { border-color: #e53935; }
input:required:valid   { border-color: #43a047; }
```
- 입력 중 실시간 스타일 피드백 가능.

### 5.3 JS API (정밀 제어)
```html
<form id="pay">
  <input type="text" id="card" name="card" required pattern="\d{16}" title="숫자 16자리">
  <button id="btn">결제</button>
</form>
<script>
  const card = document.getElementById('card');
  const btn  = document.getElementById('btn');

  // 실시간 커스텀 메시지
  card.addEventListener('input', () => {
    if (card.validity.patternMismatch) {
      card.setCustomValidity('카드번호는 공백 없이 숫자 16자리여야 합니다.');
    } else {
      card.setCustomValidity('');
    }
  });

  btn.addEventListener('click', (e) => {
    const form = document.getElementById('pay');
    if (!form.checkValidity()) {
      e.preventDefault();      // 제출 중단
      form.reportValidity();   // 브라우저 기본 툴팁 노출
    }
  });
</script>
```
- 주요 프로퍼티: `willValidate`, `validity`(여러 플래그), `validationMessage`.
- 메서드: `checkValidity()`, `reportValidity()`, `setCustomValidity()`.

### 5.4 흔한 함정
- `pattern`은 **전체 문자열 매칭**. 필요한 경우 `^…$`를 명시하는 습관.
- `type="number"`에서 로케일/소수점 기호 차이(일부 환경의 `,` vs `.`) 주의.
- `type="email"`은 **간단 검증**(RFC 전체 아님). 엄격 검증은 서버에서.

---

## 6. 접근성(a11y) & 국제화(i18n)

### 6.1 라벨 연결(필수)
```html
<label for="email">이메일</label>
<input id="email" name="email" type="email" required>
```
- `<label>`과 `for`/`id` 연결로 **클릭 영역 확장** + 스크린 리더 명확화.
- 시각적 라벨 숨김이 필요하면 **접근성 친화** 클래스(스크린리더 전용) 사용.

### 6.2 설명/오류 연결
```html
<label for="amount">금액</label>
<input id="amount" name="amount" aria-describedby="amountHint amountErr">
<div id="amountHint">원화 기준, 숫자만 입력.</div>
<div id="amountErr"  class="sr-only" aria-live="polite"></div>
```
- 오류가 생기면 JS로 `amountErr` 내용 갱신 → 스크린 리더가 공지.

### 6.3 방향/언어
```html
<input name="comment" dir="auto" dirname="comment.dir" lang="ar">
```
- **텍스트 방향/언어**를 필드 단위로 명시 → 다국어 폼 품질 향상.

---

## 7. 보안(Security) — 클라이언트 검증은 **보조**일 뿐

| 위험 | 요약 | 대응 |
|---|---|---|
| XSS | 입력을 그대로 DOM/HTML에 삽입 시 스크립트 주입 | 출력 시 **이스케이프/정화**(서버/템플릿/프론트) |
| CSRF | 사용자가 의도치 않은 제출 | **CSRF 토큰** hidden 필드 + 서버 검증 |
| 스니핑 | 전송 중 도청 | **HTTPS** 필수 |
| 자동완성 노출 | 민감정보 자동완성 | 적절한 `autocomplete` 토큰(예: `one-time-code`, `new-password`) |
| 파일 업로드 | 악성파일/확장자 위장 | **서버에서 MIME/헤더 검사**, 경로 분리, 실행권한 차단 |

> **반드시 서버에서 재검증**: 길이/형식/허용 범위/권한/속도 제한(레이트 리밋/캡차) 등.

---

## 8. 모바일 UX 강화 속성/패턴

- `inputmode`, `enterkeyhint`, `autocomplete` 토큰 조합으로 **가상 키보드 품질** 향상.
- `type="tel"` + `pattern`으로 지역 포맷 힌트 제공.
- `type="file" accept="image/*" capture="environment"`로 **카메라 선호**.

```html
<label for="phone">전화번호</label>
<input id="phone" name="phone" type="tel"
       inputmode="tel" autocomplete="tel"
       pattern="[0-9\-]{9,13}" placeholder="010-1234-5678">
```

---

## 9. “버튼별 다른 제출” 실전 패턴 (검색 vs 저장)

```html
<form id="postForm" action="/posts" method="post">
  <label>제목 <input type="text" name="title" required></label>
  <label>내용 <textarea name="body" required></textarea></label>

  <!-- 저장(POST /posts) -->
  <input type="submit" value="저장">

  <!-- 미리보기(GET /preview, 새 창, 검증 생략) -->
  <input type="submit" value="미리보기"
         formaction="/preview" formmethod="get"
         formtarget="_blank" formnovalidate>
</form>
```

---

## 10. 대형 폼 최적화 체크리스트

- [ ] **라벨**은 모든 컨트롤과 연결되어 있는가?  
- [ ] `name`은 서버 모델과 1:1 매핑되는가?  
- [ ] `autocomplete` **토큰**을 올바르게 지정했는가?  
- [ ] `required/pattern/min/max/step/minlength/maxlength` 조합으로 **HTML만으로 가능한 검증**을 충분히 활용했는가?  
- [ ] `checkValidity()/reportValidity()`로 **제출 전 클라이언트 피드백**을 제공하는가?  
- [ ] `disabled` 남용 없이, 필요 시 `readonly`로 **전송 유지** 설계를 했는가?  
- [ ] 파일 업로드는 `accept` 힌트 + 서버 측 **철저 검증**을 적용했는가?  
- [ ] 모바일 입력 환경(`inputmode`, `enterkeyhint`)을 고려했는가?  
- [ ] 국제화(`dir`, `dirname`, `lang`)와 접근성(`aria-*`, 설명 연결)을 챙겼는가?

---

## 11. 속성 레퍼런스 테이블(요약)

| 분류 | 속성 | 비고 |
|---|---|---|
| 전송 | `name`, `value`, `disabled`, `readonly` | `disabled`는 전송 제외, `readonly`는 전송 |
| 힌트/라벨 | `placeholder`, `title`, `aria-*`, `list` | 라벨을 대체하지 않음 |
| 유효성 | `required`, `pattern`, `min`, `max`, `step`, `minlength`, `maxlength`, `type` | 브라우저 기본 검증 + CSS 의사클래스 |
| 자동완성 | `autocomplete` | 토큰 기반(이메일/암호/주소/OTP 등) |
| 모바일 | `inputmode`, `enterkeyhint`, `spellcheck`, `autocapitalize` | UX 향상 |
| 폼연결 | `form`, `formaction`, `formmethod`, `formenctype`, `formnovalidate`, `formtarget` | 버튼별 별도 제출 경로/방식 |
| 미디어/파일 | `accept`, `multiple`, `capture`, `height`, `width`(image) | 카메라/갤러리 힌트 |
| 국제화 | `dir`, `dirname`, `lang` | 방향성/언어 전송 |

---

## 12. 실전 통합 예제 (제출·검증·접근성·모바일)

```html
<!doctype html>
<html lang="ko">
<head>
  <meta charset="utf-8">
  <title>회원가입</title>
  <style>
    .sr-only{position:absolute;left:-9999px}
    input:required:invalid{border-color:#e53935}
    input:required:valid{border-color:#43a047}
    .field{margin:.5rem 0}
  </style>
</head>
<body>
  <a class="sr-only" href="#main">본문 바로가기</a>

  <main id="main">
    <h1>회원가입</h1>
    <form id="reg" action="/register" method="post" autocomplete="on" novalidate>
      <div class="field">
        <label for="email">이메일</label><br>
        <input id="email" name="email" type="email"
               autocomplete="email" required
               inputmode="email" placeholder="you@example.com"
               aria-describedby="emailHelp">
        <div id="emailHelp" class="sr-only">유효한 이메일 주소를 입력하세요.</div>
      </div>

      <div class="field">
        <label for="pw">비밀번호</label><br>
        <input id="pw" name="password" type="password"
               autocomplete="new-password" minlength="8" required
               aria-describedby="pwHint">
        <div id="pwHint" class="sr-only">최소 8자, 추천: 대/소문자+숫자 조합</div>
      </div>

      <div class="field">
        <label for="phone">휴대폰</label><br>
        <input id="phone" name="tel" type="tel"
               autocomplete="tel" inputmode="tel"
               pattern="[0-9\-]{9,13}" placeholder="010-1234-5678"
               aria-describedby="telHint">
        <div id="telHint" class="sr-only">숫자와 하이픈만, 9~13자</div>
      </div>

      <div class="field">
        <label for="avatar">프로필 이미지</label><br>
        <input id="avatar" name="avatar" type="file"
               accept="image/*" capture="environment">
      </div>

      <div class="field">
        <label for="color">테마 색상</label><br>
        <input id="color" name="themeColor" type="color" value="#3f51b5">
      </div>

      <div class="field">
        <label for="birth">생일</label><br>
        <input id="birth" name="birth" type="date"
               min="1900-01-01" max="2025-12-31">
      </div>

      <div class="field">
        <label for="otp">인증 코드</label><br>
        <input id="otp" name="otp" type="text"
               autocomplete="one-time-code"
               inputmode="numeric" pattern="\d{6}"
               placeholder="6자리" required>
      </div>

      <div class="field">
        <button type="submit">가입하기</button>
        <button type="submit"
                formaction="/register/preview"
                formmethod="get"
                formtarget="_blank"
                formnovalidate>미리보기</button>
      </div>
    </form>
  </main>

  <script>
    const form = document.getElementById('reg');
    const otp  = document.getElementById('otp');
    const tel  = document.getElementById('phone');

    // 커스텀 메시지
    otp.addEventListener('input', () => {
      otp.setCustomValidity(otp.validity.patternMismatch ? '인증 코드는 숫자 6자리입니다.' : '');
    });
    tel.addEventListener('input', () => {
      tel.setCustomValidity(tel.validity.patternMismatch ? '전화번호는 숫자/하이픈만 9~13자입니다.' : '');
    });

    form.addEventListener('submit', (e) => {
      if (!form.checkValidity()) {
        e.preventDefault();
        form.reportValidity();
      }
    });
  </script>
</body>
</html>
```

---

## 13. 흔한 질문(FAQ)

**Q1. `disabled`와 `readonly` 차이?**  
- `disabled`: 전송 제외, 포커스 불가, 보조기술 탐색 어려움.  
- `readonly`: 전송 포함, 포커스 가능(브라우저별), 값 수정만 금지.  
→ **전송해야 하는 값**이라면 가급적 `readonly`.

**Q2. 민감 정보에 자동완성을 끄면 안전한가?**  
- 브라우저/암호관리자가 **무시**할 수 있다. 보안은 **HTTPS + 서버 재검증**이 본질.  
- 자동완성은 **정확한 토큰**으로 유도(예: `current-password`, `new-password`, `one-time-code`).

**Q3. `pattern`은 어떤 시점에 동작?**  
- 제출 시(또는 `reportValidity()`). 입력 중 즉시 피드백이 필요하면 **JS로 `setCustomValidity`**.

**Q4. 숫자 필드에서 소수점/쉼표가 섞일 때?**  
- 로케일에 따라 `,` → **유효성 실패** 가능. 서버는 **정규화**(`,` → `.`) 후 파싱 권장.

---

## 14. 마무리

- `<input>`은 **속성의 조합**으로 UX/검증/보안을 크게 끌어올릴 수 있다.  
- **폼 전송의 핵심은 `name`**이며, `disabled`/`readonly`의 차이, 버튼별 `form*` 속성으로 제출 흐름을 분기하라.  
- **Constraint Validation + CSS 의사클래스 + JS API**로 일관된 검증 UX를 만들고,  
- 모바일/국제화까지 고려해 `inputmode`/`enterkeyhint`/`dirname`/`autocomplete` 토큰을 적극 활용하라.  
- 그리고 무엇보다, **최종 신뢰는 서버 검증**에 있다.

> 구조(HTML) + 검증(HTML/JS) + 보안(HTTPS/서버) + 접근성(라벨/aria) = **견고한 폼**.
