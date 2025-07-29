---
layout: post
title: HTML - Web Storage API
date: 2025-04-18 19:20:23 +0900
category: HTML
---
# 💾 Web Storage API 완전 정리 (localStorage vs sessionStorage)

## ✅ Web Storage란?

**Web Storage API**는 클라이언트 측(브라우저)에 **간단한 키-값 데이터를 저장할 수 있는 API**입니다.

> 서버가 아닌 **브라우저 내에서 데이터를 보존**하기 위한 기술로, 쿠키보다 **더 쉽고 빠르고 용량도 큽니다.**

---

## 🔧 종류

| 저장소 | 설명 |
|--------|------|
| `localStorage` | 브라우저에 **영구적으로 저장**됨 (브라우저/탭을 꺼도 유지) |
| `sessionStorage` | **세션(탭/창)이 살아있는 동안만** 저장됨 (탭 닫으면 사라짐) |

---

## 📦 Web Storage와 쿠키 차이

| 항목 | Web Storage | 쿠키 |
|------|-------------|------|
| 용량 | 약 5MB | 약 4KB |
| 전송 | HTTP 요청에 포함되지 않음 | 매 요청마다 서버로 전송됨 |
| 수명 | localStorage는 영구, sessionStorage는 탭 세션 | 만료일 설정 가능 |
| 접근성 | 클라이언트에서만 접근 가능 | 클라이언트 & 서버 모두 가능 |
| 저장 구조 | Key-Value 문자열 | Key-Value 문자열 |

---

## 🧪 localStorage 사용법

```js
// 저장
localStorage.setItem("username", "Alice");

// 조회
const name = localStorage.getItem("username"); // "Alice"

// 삭제
localStorage.removeItem("username");

// 모두 제거
localStorage.clear();
```

---

## 🧪 sessionStorage 사용법

```js
// 저장
sessionStorage.setItem("token", "abc123");

// 조회
const token = sessionStorage.getItem("token");

// 삭제
sessionStorage.removeItem("token");

// 모두 제거
sessionStorage.clear();
```

> 문법은 `localStorage`와 완전히 동일합니다. **저장 범위(수명)**만 다릅니다.

---

## 🧠 주요 메서드 정리

| 메서드 | 설명 |
|--------|------|
| `setItem(key, value)` | 키-값 저장 |
| `getItem(key)` | 키에 대한 값 가져오기 |
| `removeItem(key)` | 해당 키 삭제 |
| `clear()` | 전체 저장소 비우기 |
| `length` | 저장된 항목 수 |
| `key(index)` | 인덱스로 key 조회 |

예:

```js
localStorage.setItem("color", "blue");
console.log(localStorage.key(0)); // "color"
console.log(localStorage.length); // 1
```

---

## 📂 저장 데이터는 문자열(String)

Web Storage는 **항상 문자열만 저장**합니다.  
객체나 배열은 `JSON.stringify()`로 변환해야 합니다.

```js
const user = { name: "Kim", age: 25 };

// 저장
localStorage.setItem("user", JSON.stringify(user));

// 가져오기
const stored = JSON.parse(localStorage.getItem("user"));
console.log(stored.name); // "Kim"
```

---

## 🔐 보안 및 주의사항

| 주의 사항 | 설명 |
|-----------|------|
| ❗ 민감한 정보 저장 금지 | Web Storage는 **암호화되지 않음**. 누구나 접근 가능 |
| ❗ XSS 공격에 취약 | 악성 스크립트가 storage 접근 가능 → 반드시 입력값 필터링 필요 |
| ❗ 도메인 단위 저장 | 같은 도메인에서만 접근 가능 (서브도메인도 분리됨) |
| ❗ 용량 제한 있음 | 브라우저마다 약 5~10MB 정도의 제한 (초과 시 예외 발생)

---

## 🌐 사용 가능한 환경

| 브라우저 | 지원 여부 |
|----------|------------|
| Chrome | ✅ |
| Firefox | ✅ |
| Safari | ✅ |
| Edge | ✅ |
| IE 8+ | ✅ (localStorage만 부분 지원) |

> 📱 모바일 브라우저도 대부분 지원됩니다.

---

## 💡 실전 활용 예시

| 분야 | 예시 |
|------|------|
| 로그인 유지 | 로그인 정보 or 토큰 임시 저장 (단, 민감한 정보는 ❌) |
| 사용자 설정 | 테마 색상, 언어, 글꼴 크기 저장 |
| 장바구니 | 로그인하지 않은 사용자도 장바구니 유지 |
| 임시 폼 데이터 | 작성 중인 설문지나 글 임시 저장 |
| 방문 기록 | 첫 방문 여부 확인, 환영 메시지 등

---

## ✅ 요약 정리

| 항목 | localStorage | sessionStorage |
|------|--------------|----------------|
| 수명 | 영구 저장 | 탭 종료 시 삭제 |
| 저장 위치 | 브라우저 내 | 브라우저 내 |
| 전송 여부 | 서버에 전송 ❌ | 서버에 전송 ❌ |
| 주요 용도 | 사용자 설정, 장기 저장 | 임시 폼 데이터, 일시적 상태 저장 |

---

## 📚 참고 자료

- [MDN Web Docs - Web Storage](https://developer.mozilla.org/ko/docs/Web/API/Web_Storage_API)
- [W3Schools Web Storage Tutorial](https://www.w3schools.com/html/html5_webstorage.asp)
- [Can I Use - localStorage](https://caniuse.com/?search=localStorage)