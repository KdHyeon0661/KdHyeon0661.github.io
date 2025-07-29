---
layout: post
title: JavaScript - Fetch API
date: 2025-05-13 22:20:23 +0900
category: JavaScript
---
# 🌐 Fetch API 사용법 완전 정복

`Fetch API`는 자바스크립트에서 네트워크 요청을 수행하기 위한 **비동기 통신 도구**입니다.  
과거의 `XMLHttpRequest`보다 훨씬 간결하며, Promise 기반으로 작성되어 **가독성과 유지보수성**이 뛰어납니다.

---

## ✅ 1. 기본 사용법 (GET 요청)

```js
fetch("https://jsonplaceholder.typicode.com/posts/1")
  .then((response) => response.json())
  .then((data) => console.log(data))
  .catch((error) => console.error("Error:", error));
```

### 설명:

- `fetch()`는 `Promise`를 반환
- `response.json()`으로 JSON 본문 파싱
- `.then()`으로 결과 처리
- `.catch()`로 에러 처리

---

## ✅ 2. JSON 응답 받기

```js
fetch("/api/user")
  .then((res) => {
    if (!res.ok) throw new Error("응답 실패");
    return res.json();
  })
  .then((data) => {
    console.log("사용자 정보:", data);
  })
  .catch((err) => console.error("에러:", err));
```

### 🔍 주의: `fetch()`는 **HTTP 에러(404, 500)**도 "성공"으로 간주하므로  
**반드시 `res.ok`를 체크**해야 합니다!

---

## ✅ 3. POST 요청 (JSON 데이터 전송)

```js
fetch("/api/login", {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
  },
  body: JSON.stringify({
    username: "alice",
    password: "1234",
  }),
})
  .then((res) => res.json())
  .then((data) => console.log("로그인 결과:", data))
  .catch((err) => console.error("에러:", err));
```

---

## ✅ 4. 요청 옵션 전체

| 옵션 | 설명 |
|------|------|
| `method` | HTTP 메서드 (`GET`, `POST`, `PUT`, `DELETE` 등) |
| `headers` | 요청 헤더 |
| `body` | 전송 데이터 (`string` 또는 `FormData`) |
| `mode` | `'cors'`, `'no-cors'`, `'same-origin'` |
| `credentials` | `'omit'`, `'same-origin'`, `'include'` |
| `cache` | `'default'`, `'no-cache'`, `'reload'`, `'force-cache'` 등 |
| `redirect` | `'follow'`, `'error'`, `'manual'` |

---

## ✅ 5. 기타 요청 예시

### 🔸 PUT 요청

```js
fetch("/api/user/123", {
  method: "PUT",
  headers: {
    "Content-Type": "application/json",
  },
  body: JSON.stringify({ name: "Bob" }),
});
```

### 🔸 DELETE 요청

```js
fetch("/api/user/123", {
  method: "DELETE",
});
```

---

## ✅ 6. async/await 버전

```js
async function fetchUser() {
  try {
    const res = await fetch("/api/user");
    if (!res.ok) throw new Error("서버 응답 실패");

    const data = await res.json();
    console.log("유저 정보:", data);
  } catch (e) {
    console.error("에러 발생:", e);
  }
}

fetchUser();
```

> `async/await`는 코드 흐름을 동기처럼 보여줘 가독성이 좋아집니다.

---

## ✅ 7. FormData 전송 (파일 업로드 등)

```js
const formData = new FormData();
formData.append("file", fileInput.files[0]);

fetch("/upload", {
  method: "POST",
  body: formData, // Content-Type 생략 (자동 처리)
});
```

---

## ✅ 8. CORS (Cross-Origin Resource Sharing)

다른 도메인으로 요청 시 **CORS 정책**이 적용됩니다. 서버가 `Access-Control-Allow-Origin` 헤더를 포함해야만 허용됩니다.

```js
fetch("https://api.example.com/data", {
  method: "GET",
  mode: "cors",
});
```

> ❗ 브라우저 보안 정책 때문에 서버 설정이 필요할 수 있습니다.

---

## ✅ 9. fetch vs axios 비교

| 항목 | fetch | axios |
|------|-------|-------|
| 기본 지원 | 브라우저 내장 | 외부 라이브러리 필요 |
| 응답 처리 | `res.json()` | 자동 파싱 |
| HTTP 에러 처리 | 수동 (`res.ok`) | 자동 throw |
| 요청 취소 | 지원 X (AbortController 필요) | 지원 O |
| 브라우저 지원 | 최신 브라우저 | IE 포함 (Polyfill 필요) |

---

## ✅ 10. 요청 취소: AbortController

```js
const controller = new AbortController();

fetch("/api/data", { signal: controller.signal })
  .then((res) => res.json())
  .then((data) => console.log(data))
  .catch((err) => {
    if (err.name === "AbortError") {
      console.log("요청이 취소되었습니다.");
    }
  });

// 1초 후 요청 취소
setTimeout(() => controller.abort(), 1000);
```

---

## ✅ 11. try-catch로 안전하게 감싸기

```js
async function safeFetch(url) {
  try {
    const res = await fetch(url);
    if (!res.ok) throw new Error(`HTTP 오류: ${res.status}`);
    return await res.json();
  } catch (err) {
    console.error("fetch 실패:", err.message);
    return null;
  }
}
```

---

## 🧠 마무리

- `fetch()`는 네트워크 요청을 간결하게 처리할 수 있는 **Promise 기반 API**
- **GET, POST, PUT, DELETE** 등 다양한 메서드를 지원
- **에러는 반드시 `res.ok` 확인** 필요
- **async/await과 함께 사용하면 가독성 향상**
- **파일 전송, CORS, 요청 취소(AbortController)** 등 다양한 기능 활용 가능