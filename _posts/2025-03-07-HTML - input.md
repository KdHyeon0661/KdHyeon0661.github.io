---
layout: post
title: HTML - input
date: 2025-03-07 22:20:23 +0900
category: HTML
---
# HTML `<input>` 태그 완벽 가이드 – 종류, 속성, 접근성, 보안까지

웹에서 사용자의 입력을 받는 가장 기본적이고 강력한 태그가 바로 `<input>`입니다.

로그인, 회원가입, 검색, 체크박스, 파일 업로드 등 대부분의 입력은 `<input>` 하나로 해결됩니다.

이 글에서는 `<input>`의 **모든 타입과 속성**, 그리고 **실전 활용 방법과 보안/접근성 팁**까지 자세히 알아봅니다.

---

## ✅ 1. `<input>` 태그란?

`<input>` 태그는 사용자의 데이터를 입력받기 위한 **단일 self-closing 태그**입니다.  
입력 타입은 `type` 속성으로 제어하며, **HTML5 이후 다양한 유형**이 추가되었습니다.

```html
<input type="text" name="username">
```

---

## 🧾 2. `type` 속성별 정리

### 📄 텍스트 입력 계열

| 타입            | 설명 |
|------------------|------|
| `text`           | 한 줄 텍스트 입력 (기본값) |
| `password`       | 비밀번호 (입력 내용 가림) |
| `email`          | 이메일 형식 자동 검사 |
| `search`         | 검색 필드 (브라우저별 스타일) |
| `tel`            | 전화번호 (모바일 키보드 지원) |
| `url`            | URL 형식 자동 검사 |

```html
<input type="email" placeholder="you@example.com" required>
```

---

### 🔢 숫자 / 범위 / 날짜 계열

| 타입         | 설명 |
|--------------|------|
| `number`     | 숫자 입력, 증감 버튼 제공 |
| `range`      | 슬라이더 형태로 범위 입력 |
| `date`       | 날짜 선택기 (yyyy-mm-dd) |
| `time`       | 시간 선택기 (hh:mm) |
| `datetime-local` | 날짜 + 시간 선택 (로컬 기준) |
| `month`      | 월 단위 선택 |
| `week`       | 주 단위 선택 |

```html
<input type="number" min="1" max="100">
<input type="range" min="0" max="10" step="0.5">
```

---

### 🔘 선택 입력 계열

| 타입       | 설명 |
|------------|------|
| `checkbox` | 다중 선택 가능 |
| `radio`    | 그룹 내 단일 선택 |

```html
<input type="checkbox" name="agree" checked>
<input type="radio" name="gender" value="male">
<input type="radio" name="gender" value="female">
```

**💡 radio 버튼은 동일한 `name` 값을 공유해야 하나만 선택됩니다.**

---

### 📁 파일 및 숨김 필드

| 타입       | 설명 |
|------------|------|
| `file`     | 파일 업로드 창 |
| `hidden`   | 사용자에게 보이지 않지만 값 전달 가능 |
| `submit`   | 폼 제출 버튼 |
| `reset`    | 입력 초기화 |
| `button`   | 일반 버튼 (JS와 함께 사용) |

```html
<input type="file" accept="image/*">
<input type="hidden" name="token" value="abc123">
```

---

## ⚙️ 3. 주요 속성 정리

| 속성           | 설명 |
|----------------|------|
| `name`         | 서버에 전송될 키 이름 |
| `value`        | 초기값 |
| `placeholder`  | 힌트 텍스트 (text류에만 적용) |
| `required`     | 필수 입력 여부 |
| `readonly`     | 읽기 전용 (사용자 수정 불가) |
| `disabled`     | 비활성화 (전송도 안 됨) |
| `checked`      | 체크 상태 설정 (`checkbox`, `radio`) |
| `min`, `max`   | 값의 범위 설정 (숫자, 날짜 등) |
| `maxlength`    | 최대 글자 수 |
| `pattern`      | 정규표현식 입력 제한 |

### 예시: 유효성 제한

```html
<input type="text" name="zipcode" pattern="[0-9]{5}" title="5자리 숫자를 입력하세요" required>
```

---

## ♿ 4. 접근성 (a11y) 고려

### ✅ `label`과 연결하기

```html
<label for="email">이메일</label>
<input type="email" id="email" name="email" required>
```

- `label`은 `for` 속성으로 `input`과 연결
- 스크린 리더 사용자에게 읽기 쉬움
- 클릭 영역도 확장됨

### ✅ 시각적 힌트 + 접근성

```html
<input aria-label="검색어 입력" placeholder="검색어 입력">
```

- `aria-label`은 보이지 않는 텍스트 설명
- 접근성 점수 향상

---

## 🔐 5. 보안 관련 팁

| 위험 요소 | 대응 방법 |
|-----------|-----------|
| 자동완성 민감 정보 | `autocomplete="off"` 사용 |
| 입력값 신뢰 | **서버에서 반드시 유효성 검사** 수행 |
| XSS 공격 | 사용자 입력을 DOM에 넣기 전에 **이스케이프 처리** |
| 파일 업로드 위험 | 파일 확장자 검사, MIME 타입 검증, 업로드 제한 필요 |

---

## 🧪 6. 실전 예제 – 회원가입 입력 폼

```html
<form action="/register" method="POST">
  <label for="username">아이디</label><br>
  <input type="text" id="username" name="username" required minlength="4"><br>

  <label for="email">이메일</label><br>
  <input type="email" id="email" name="email" required><br>

  <label for="password">비밀번호</label><br>
  <input type="password" id="password" name="password" required minlength="8"><br>

  <label for="birth">생년월일</label><br>
  <input type="date" id="birth" name="birth"><br>

  <input type="submit" value="가입하기">
</form>
```

---

## 🧾 요약표

| 타입          | 사용 목적           |
|---------------|---------------------|
| `text`, `password` | 일반 텍스트 입력 |
| `email`, `tel`, `url` | HTML5 검증 포함 입력 |
| `checkbox`, `radio` | 선택형 옵션 |
| `file`        | 파일 첨부 |
| `hidden`      | 보이지 않는 값 전달 |
| `date`, `range`, `number` | 범위/수치 입력 |
| `submit`, `reset`, `button` | 버튼 역할 수행 |

---

## ✅ 마무리

`<input>` 태그는 단순해 보여도 수많은 기능을 포함한 **폼의 핵심 요소**입니다.

- `type` 속성으로 다양한 입력 방식 처리  
- `required`, `pattern` 등 HTML5 유효성 검사 지원  
- 보안과 접근성도 함께 고려해야 함

---

**입력값은 항상 서버에서 다시 검증해야 하고**,  
JavaScript와 연동하면 실시간 입력 검증, 자동완성, 조건부 필드 표시 같은 **인터랙티브 폼**도 만들 수 있습니다.