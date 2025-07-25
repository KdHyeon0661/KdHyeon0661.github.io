---
layout: post
title: HTML - input 요소 속성
date: 2025-03-26 19:20:23 +0900
category: HTML
---
# 🧾 HTML & HTML5의 `<input>` 요소 속성 완전 정리

HTML에서 `<input>` 요소는 가장 많이 사용되는 폼 요소 중 하나입니다.  
이 요소는 다양한 **속성(attribute)**을 통해 입력값의 제어, 제출 동작, 유효성 검사, 사용자 경험 향상 등을 지원합니다.

이 글에서는 HTML과 HTML5에서 사용할 수 있는 `<input>` 요소의 주요 속성들을 **카테고리별로 나누어** 상세히 설명합니다.

---

## 📌 1. HTML 기본 `<input>` 속성

### 1️⃣ `value`
- **입력 필드의 초기값 또는 현재 값**을 지정합니다.
```html
<input type="text" value="기본값">
```

### 2️⃣ `readonly`
- 필드를 **읽기 전용**으로 만들어 수정은 불가능하지만, 복사할 수는 있습니다.
```html
<input type="text" value="읽기 전용" readonly>
```

### 3️⃣ `disabled`
- 해당 필드를 **비활성화**하여 사용자 입력 및 전송을 차단합니다.
```html
<input type="text" value="비활성화됨" disabled>
```

### 4️⃣ `maxlength`
- 입력할 수 있는 **최대 글자 수**를 제한합니다.
```html
<input type="text" maxlength="10">
```

### 5️⃣ `size`
- **입력창의 너비**를 글자 수 기준으로 지정합니다.
```html
<input type="text" size="30">
```

---

## 🧩 2. HTML5에서 추가된 `<form>` 요소 속성

### 1️⃣ `autocomplete`
- 브라우저가 이전에 입력한 값을 기억하고 자동완성할 수 있도록 합니다.
```html
<form autocomplete="off">
```

### 2️⃣ `novalidate`
- **폼 제출 시 HTML5 유효성 검사를 무시**합니다.
```html
<form novalidate>
```

---

## 🚀 3. HTML5에서 추가된 `<input>` 속성들

| 속성 | 설명 |
|------|------|
| `autocomplete` | 자동완성 허용 여부 (`on` / `off`) |
| `autofocus` | 페이지 로드 시 자동으로 포커스 |
| `form` | 특정 `<form id="...">`과 연결 (폼 외부에 위치할 경우 유용) |
| `formaction` | 버튼 클릭 시 제출할 URL 지정 (`submit` 전용) |
| `formenctype` | 폼 전송 시 인코딩 방식 (`multipart/form-data` 등) |
| `formmethod` | 폼 제출 방식 (`get` / `post`) |
| `formnovalidate` | 해당 입력 필드만 유효성 검사 생략 |
| `formtarget` | 폼 응답을 열 위치 지정 (`_blank`, `_self`, 등) |
| `height`, `width` | `<input type="image">`에서 사용되는 이미지 크기 지정 |
| `list` | `<datalist>`와 연결하여 자동완성 제안 목록 제공 |
| `min`, `max` | 숫자, 날짜 등의 최소/최대값 지정 |
| `multiple` | 여러 개의 값 입력 허용 (`file`, `email` 등) |
| `pattern` | 정규표현식으로 유효성 검사 |
| `placeholder` | 힌트 텍스트 표시 |
| `required` | 필수 입력 필드 지정 |
| `step` | 증가 간격 지정 (숫자, 날짜 등) |

---

## 📌 주요 속성 예시

### 🔹 `autofocus`

```html
<input type="text" autofocus placeholder="자동 포커스">
```

### 🔹 `placeholder`

```html
<input type="text" placeholder="입력 힌트 표시">
```

### 🔹 `pattern`

```html
<input type="text" pattern="[A-Za-z]{3}" title="영문 3자만 입력하세요">
```

### 🔹 `min`, `max`, `step`

```html
<input type="number" min="0" max="100" step="10">
```

### 🔹 `multiple`

```html
<input type="email" multiple placeholder="여러 이메일 입력 (쉼표로 구분)">
<input type="file" multiple>
```

---

## 🧠 유효성 검사 관련 속성 요약

| 속성 | 설명 |
|------|------|
| `required` | 필수 입력 여부 지정 |
| `pattern` | 정규표현식 기반 입력값 검사 |
| `min`, `max`, `step` | 수치/날짜 범위 제한 |
| `maxlength`, `size` | 입력 가능 글자 수 제한 |
| `type` 속성 | 자동 유효성 검사 적용 (email, url 등) |

---

## ✅ 실무 팁

- `required`, `pattern`, `min/max` 등을 조합하면 **JS 없이도 강력한 유효성 검사 가능**
- `autofocus`는 **한 페이지당 하나만 사용** (중복 시 동작 보장 안됨)
- `form` 속성은 **입력 요소가 form 외부에 위치할 경우 매우 유용**
- `formaction`, `formmethod` 등은 **다양한 제출 버튼 구성 시** 활용

---

## 🧾 마무리

HTML5는 `<input>` 요소에 다양한 속성을 추가해 **웹 폼의 기능성과 사용자 경험을 극대화**시켰습니다.  
이러한 속성들을 잘 이해하고 조합하면, **자바스크립트 없이도 유효성 검사, 자동완성, 사용자 편의 제공**이 가능합니다.
