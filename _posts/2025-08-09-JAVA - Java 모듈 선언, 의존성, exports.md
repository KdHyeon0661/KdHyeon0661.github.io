---
layout: post
title: Java - Java 모듈 선언, 의존성, exports
date: 2025-08-09 21:20:23 +0900
category: Java
---
# Java 모듈 선언, 의존성, exports

Java 9에서 도입된 **Java Platform Module System (JPMS)**는  
모듈 단위로 코드의 공개 범위와 의존성을 명시적으로 관리할 수 있게 해줍니다.  
`module-info.java` 파일에서 **모듈 선언**, **의존성(require)**, **공개(exports)**를 정의합니다.

---

## 1. 모듈 선언 (`module`)

모듈 선언은 **`module-info.java`** 파일에서 `module` 키워드로 시작합니다.

```java
module com.example.myapp {
    // 의존성 선언
    requires java.sql;

    // 패키지 공개
    exports com.example.myapp.api;
}
```

- **`module` 키워드** 뒤에는 **모듈 이름**을 작성합니다.
- 모듈 이름 규칙:
  - 패키지 네이밍 규칙과 유사 (예: `com.company.project`)
  - 소문자 권장
  - 점(`.`)으로 구분 가능
- 한 모듈은 **하나의 `module-info.java`**만 가질 수 있습니다.
- `module-info.java`는 모듈의 루트 패키지에 위치해야 합니다.

---

## 2. 의존성 선언 (`requires`)

`requires` 키워드는 해당 모듈이 다른 모듈을 **사용**한다고 선언합니다.  
즉, 해당 모듈의 **공개 패키지**에 접근할 수 있습니다.

### 기본 형식
```java
module com.example.myapp {
    requires java.sql;
    requires com.example.util;
}
```

---

### 2.1. `requires transitive`

```java
module com.example.service {
    requires transitive com.example.core;
}
```

- **전이 의존성**(Transitive Dependency)
- `com.example.service`를 사용하는 모듈은 **자동으로** `com.example.core`에도 접근 가능.
- API 계층에서 하위 계층을 공개하고 싶을 때 유용.

---

### 2.2. `requires static`

```java
module com.example.compiler {
    requires static com.example.annotations;
}
```

- **컴파일 시에만** 필요한 의존성.
- 런타임에는 불필요 (예: 애너테이션 프로세서, 테스트용 라이브러리).

---

### 2.3. JDK 기본 모듈 예시

| 모듈명          | 포함 패키지 예시            |
|----------------|--------------------------|
| `java.base`    | java.lang, java.util 등 (기본 모듈, 모든 모듈에 암시적 포함) |
| `java.sql`     | JDBC API                 |
| `java.xml`     | XML 처리 API             |
| `java.logging` | java.util.logging        |

---

## 3. 패키지 공개 (`exports`)

`exports`는 모듈의 **특정 패키지**를 외부 모듈에서 사용할 수 있게 공개합니다.

### 기본 형식
```java
module com.example.myapp {
    exports com.example.myapp.api;
}
```
- `com.example.myapp.api` 패키지 안의 **public 클래스/인터페이스**가 외부에서 접근 가능.
- 공개하지 않은 패키지는 같은 모듈 안에서만 접근 가능 → **강한 캡슐화**.

---

### 3.1. 특정 모듈에만 공개

```java
module com.example.myapp {
    exports com.example.myapp.internal to com.example.test;
}
```
- `com.example.test` 모듈에서만 접근 가능.
- 나머지 모듈에서는 접근 불가.

---

### 3.2. `exports`와 리플렉션의 차이

- `exports`는 **컴파일·런타임** 모두 접근 허용.
- 리플렉션을 통한 비공개 필드/메서드 접근은 `exports`만으로 불가 → `opens` 필요.

---

## 4. 예제 — 모듈 선언 + 의존성 + exports

```java
module com.example.app {
    // 의존성 선언
    requires java.sql;                 // JDBC API
    requires transitive com.example.core; // 하위 모듈 전이 공개

    // 패키지 공개
    exports com.example.app.api;       // API 패키지 전체 공개
    exports com.example.app.util to com.example.test; // 특정 모듈에만 공개
}
```

---

## 5. 정리

| 구분        | 설명 |
|-------------|------|
| `module`    | 모듈 선언 |
| `requires`  | 다른 모듈 사용 선언 |
| `transitive`| 전이 의존성, 해당 모듈을 사용하는 모듈도 접근 가능 |
| `static`    | 컴파일 시에만 필요한 의존성 |
| `exports`   | 외부 모듈에 패키지 공개 |
| `exports ... to` | 특정 모듈에만 패키지 공개 |

---

## 6. 베스트 프랙티스

1. **exports 최소화** → API만 공개, 내부 구현 패키지는 감추기.
2. **transitive 최소 사용** → 의존성 전파 범위를 최소화.
3. **static 의존성** 활용 → 불필요한 런타임 의존성 제거.
4. 모듈 이름은 **역도메인 네이밍 규칙** 권장.
