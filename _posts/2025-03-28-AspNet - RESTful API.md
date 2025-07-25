---
layout: post
title: Avalonia - RESTful API
date: 2025-03-28 20:20:23 +0900
category: Avalonia
---
# 🌐 RESTful API 설계 완전 정리

---

## ✅ 1. REST란?

**REST (Representational State Transfer)**는  
웹 아키텍처 스타일 중 하나로, **HTTP를 기반으로 리소스를 표현하고 상태를 전이**시키는 방식.

### REST의 6가지 제약 조건

| 제약 | 설명 |
|------|------|
| 클라이언트-서버 구조 | 역할 분리 (UI ↔ 데이터) |
| 무상태성 (Stateless) | 각 요청은 독립적으로 처리 |
| 캐시 가능성 | 응답은 명시적으로 캐시 가능해야 함 |
| 계층화된 시스템 | 중간 게이트웨이, 프록시 허용 |
| 일관된 인터페이스 | 표준 URI, 메서드, 포맷 사용 |
| 코드 온 디맨드 (선택적) | 클라이언트에 코드 제공 가능 (거의 사용 안 함) |

---

## 📌 2. RESTful API란?

REST의 원칙을 따르도록 설계된 **웹 API**를 의미하며,  
**자원(Resource)**를 중심으로 URI를 구성하고, **표준 HTTP 메서드**를 통해 동작을 수행함.

---

## 📦 3. 자원(Resource)의 설계

### ✅ URI는 명사, 복수형 사용

- ❌ `GET /getUser`, `POST /createUser` → 비표준
- ✅ `GET /users`, `POST /users` → RESTful

### ✅ 기본 자원 URI 패턴

| 동작 | URI | HTTP 메서드 |
|------|-----|--------------|
| 전체 조회 | `/users` | `GET` |
| 단일 조회 | `/users/{id}` | `GET` |
| 생성 | `/users` | `POST` |
| 수정 (전체) | `/users/{id}` | `PUT` |
| 수정 (일부) | `/users/{id}` | `PATCH` |
| 삭제 | `/users/{id}` | `DELETE` |

### ✅ 중첩 자원 표현

```txt
GET /users/42/posts          # 사용자 42의 게시글 목록
GET /users/42/posts/7        # 사용자 42의 게시글 중 ID 7
```

---

## 🧩 4. HTTP 메서드 정리

| 메서드 | 용도 | 요청 바디 |
|--------|------|-----------|
| GET | 조회 (Read) | ❌ |
| POST | 생성 (Create) | ✅ |
| PUT | 전체 수정 (Replace) | ✅ |
| PATCH | 부분 수정 (Partial Update) | ✅ |
| DELETE | 삭제 (Delete) | ❌ |

> ❗ POST는 항상 새 자원을 만들고, PUT은 전체 덮어씌움.

---

## 📄 5. 응답 형식 (JSON)

RESTful API의 응답은 보통 **JSON** 형식을 사용하며,  
**명확하고 일관된 구조**가 중요함.

### ✅ 예시

```json
{
  "id": 42,
  "name": "John Doe",
  "email": "john@example.com"
}
```

### ✅ 응답 Wrapping 예시 (선택)

```json
{
  "success": true,
  "data": {
    "id": 42,
    "name": "John Doe"
  },
  "message": "조회 성공"
}
```

---

## 📦 6. HTTP 상태 코드

| 코드 | 의미 | 설명 |
|------|------|------|
| `200 OK` | 성공 | 일반 조회 성공 |
| `201 Created` | 생성 성공 | POST 처리 완료 |
| `204 No Content` | 성공, 반환 없음 | DELETE, PUT 등 |
| `400 Bad Request` | 잘못된 요청 | 유효성 검사 실패 |
| `401 Unauthorized` | 인증 필요 | 토큰 없음 |
| `403 Forbidden` | 접근 거부 | 권한 없음 |
| `404 Not Found` | 자원 없음 | ID 틀림 |
| `409 Conflict` | 중복 충돌 | 이미 존재함 |
| `500 Internal Server Error` | 서버 에러 | 예외 발생 시 |

---

## 🧭 7. API 버전 관리

- ✅ URI 버전: `/api/v1/users`
- ❗ 헤더 방식: `Accept: application/vnd.myapp.v1+json`

---

## 🛡 8. 인증 & 보안

| 항목 | 설명 |
|------|------|
| HTTPS | 모든 API는 SSL 암호화 필요 |
| JWT | 인증 토큰 기반 인증 방식 |
| OAuth2 | 제3자 권한 위임 방식 |
| CORS | 크로스 도메인 요청 허용 설정 |
| Rate Limit | API 남용 방지 (요청 제한) |

---

## 🧪 9. 실전 예제 (사용자 API)

### ✅ POST /api/users

```json
POST /api/users
Content-Type: application/json

{
  "name": "Alice",
  "email": "alice@example.com"
}
```

응답:

```json
201 Created
Location: /api/users/101

{
  "id": 101,
  "name": "Alice",
  "email": "alice@example.com"
}
```

---

### ✅ GET /api/users/101

```http
GET /api/users/101
```

응답:

```json
200 OK
{
  "id": 101,
  "name": "Alice",
  "email": "alice@example.com"
}
```

---

### ✅ DELETE /api/users/101

```http
DELETE /api/users/101
```

응답:

```http
204 No Content
```

---

## 🧰 10. RESTful 설계 체크리스트 ✅

| 항목 | 체크 |
|------|------|
| URI에 동사 대신 명사 사용 ✔ |
| 자원은 복수형 명사 사용 ✔ |
| HTTP 메서드 의미에 맞게 사용 ✔ |
| 일관된 상태 코드 반환 ✔ |
| URI는 계층적 구조 유지 ✔ |
| 오류 메시지는 명확한 JSON ✔ |
| 버전 관리는 URI 또는 헤더 기반 ✔ |

---

## ✅ 요약

| 구성 요소 | 내용 |
|-----------|------|
| 자원 URI | 명사, 계층적, 복수형 |
| HTTP 메서드 | GET, POST, PUT, PATCH, DELETE |
| 응답 형식 | JSON + 상태 코드 |
| 상태 코드 | 200, 201, 204, 400, 401, 404, 500 등 |
| 인증 방식 | JWT, OAuth2 |
| 버전 관리 | URI 기반 권장 (/v1/) |

---

## 🔜 추천 다음 주제

- ✅ REST vs GraphQL 비교
- ✅ RESTful한 오류 처리 구조 (Error Wrapper)
- ✅ HATEOAS란?
- ✅ REST API 테스트 방법 (Postman, Swagger)
- ✅ OpenAPI(Swagger) 문서 작성 자동화

---

RESTful API는 단순히 URL을 예쁘게 만드는 것이 아니라,  
**리소스 중심의 설계 철학과 HTTP의 의미를 올바르게 사용하는 것**이 핵심이다.  
이를 잘 지키면 유지보수와 확장에 강한 API를 만들 수 있다!