---
layout: post
title: Java - module-info.java 구조
date: 2025-08-09 19:20:23 +0900
category: Java
---
# `module-info.java` 구조 — Java 모듈 시스템 (Java 9+)

Java 9부터 도입된 **Java Platform Module System (JPMS)**는  
대규모 애플리케이션에서 **패키지의 명확한 공개/비공개, 의존성 선언, 강력한 캡슐화**를 가능하게 합니다.  
모듈 시스템을 사용하면 **`module-info.java`**라는 파일을 통해 해당 모듈의 메타데이터를 정의합니다.

---

## 1. 기본 구조

`module-info.java`는 **모듈의 이름, 의존하는 모듈, 외부로 공개할 패키지, 서비스 제공/사용 정보**를 기술합니다.

```java
module 모듈이름 {
    // 모듈이 의존하는 다른 모듈
    requires 모듈명;

    // 패키지 공개
    exports 패키지명;

    // 특정 모듈에만 패키지 공개
    exports 패키지명 to 대상모듈명;

    // 다른 모듈에 열어두는 (리플렉션 허용)
    opens 패키지명;

    // 특정 모듈에만 열기
    opens 패키지명 to 대상모듈명;

    // 서비스 사용/제공
    uses 서비스인터페이스;
    provides 서비스인터페이스 with 구현클래스;
}
```

---

## 2. 주요 키워드

| 키워드         | 설명 |
|----------------|------|
| `module`       | 모듈 정의 시작 키워드 |
| `requires`     | 다른 모듈을 의존성으로 선언 |
| `requires transitive` | 의존 모듈을 해당 모듈을 사용하는 다른 모듈에도 전달 |
| `exports`      | 패키지를 외부 모듈에 공개 |
| `exports ... to` | 특정 모듈에만 패키지 공개 |
| `opens`        | 패키지를 리플렉션 용도로 공개 (`setAccessible` 허용) |
| `opens ... to` | 특정 모듈에만 리플렉션 허용 |
| `uses`         | 서비스 로더(ServiceLoader)를 통한 서비스 사용 선언 |
| `provides ... with` | 서비스 인터페이스 구현체 제공 |

---

## 3. 예제 1 — 간단한 모듈 선언

```java
module com.example.app {
    requires java.base;   // java.lang, java.util 등 (기본적으로 생략 가능)
    requires com.example.util; // 다른 모듈 의존

    exports com.example.app.api; // API 패키지 공개
}
```

---

## 4. 예제 2 — 서비스 제공과 사용

```java
module com.example.payment {
    requires com.example.bankapi;

    // 이 모듈에서 제공하는 서비스
    provides com.example.bankapi.PaymentService
        with com.example.payment.impl.CardPaymentService;

    // 서비스 로더를 사용해서 PaymentService 구현을 찾음
    uses com.example.bankapi.PaymentService;
}
```

---

## 5. 예제 3 — 리플렉션 허용

```java
module com.example.reflectiondemo {
    requires com.fasterxml.jackson.databind;

    // Jackson 같은 라이브러리가 리플렉션으로 private 필드를 읽을 수 있게 허용
    opens com.example.reflectiondemo.model to com.fasterxml.jackson.databind;
}
```

---

## 6. 모듈 이름 규칙

- 패키지 네이밍 규칙을 따르는 것이 좋음 (예: `com.company.project`)
- **점(`.`)**으로 구분 가능
- 소문자 권장
- **모듈 이름 = jar 파일의 모듈명** (자동 모듈의 경우 `파일명-버전` 형태)

---

## 7. 강한 캡슐화의 장점

- `exports`로 지정하지 않은 패키지는 외부에서 접근 불가
- `opens` 없이는 리플렉션도 불가 → 보안 강화
- 의존성 명시적 관리 → 컴파일 시 의존성 누락 방지
- JDK 자체도 모듈화되어 필요 없는 모듈 제외 가능 → **JLink**로 경량 런타임 제작 가능

---

## 8. `requires`의 세부 옵션

| 키워드                   | 설명 |
|--------------------------|------|
| `requires`               | 의존성 선언 |
| `requires transitive`    | 의존 모듈을 **다른 모듈에도 자동 전달** |
| `requires static`        | **컴파일 시만 필요**하고 런타임에는 필요 없음 |

예:
```java
module com.example.api {
    requires transitive com.example.core;
}
```
→ `com.example.api`를 사용하는 모듈은 `com.example.core`도 자동 접근 가능.

---

## 9. 자동 모듈 (Automatic Module)

- 기존 Java 8 이하 라이브러리를 그대로 모듈 경로에 두면  
  **자동 모듈 이름**이 생성되어 `module-info.java` 없이 사용 가능.
- 단점: 모든 패키지가 공개되며, 의존성이 명시되지 않음.

---

## 10. unnamed module (이름 없는 모듈)

- 모듈 시스템에 포함되지 않은 클래스패스 기반 코드
- 모든 패키지에 접근 가능 (모듈 경계 무시)
- 점진적 마이그레이션 시 유용

---

## 11. 모듈 경로와 빌드

**javac**
```bash
javac --module-source-path src -d out $(find src -name "*.java")
```

**java 실행**
```bash
java --module-path out -m com.example.app/com.example.app.Main
```

**Maven/Gradle**는 `maven-compiler-plugin` 또는 `--module-path` 옵션을 자동 적용.

---

## 12. 주의 사항

1. **패키지 중복 불가** — 하나의 모듈에 같은 패키지가 여러 번 선언되면 에러.
2. **exports 없이 접근 불가** — 패키지가 비공개이면 public 클래스라도 외부 모듈에서 접근 불가.
3. **opens 필요 시 명시** — Jackson, Hibernate 등 리플렉션 라이브러리 사용 시 반드시 `opens` 지정.
4. `requires transitive`를 남용하면 모듈 경계가 흐려짐.
5. JDK 모듈(`java.base`, `java.sql`, `java.logging` 등)도 모듈 선언으로 관리.

---

## 13. 결론

`module-info.java`는 Java 애플리케이션의 **구조, 의존성, 캡슐화 정책**을 명시적으로 정의하는 설계 문서 역할을 합니다.  
JPMS를 잘 활용하면 **대규모 프로젝트의 유지보수성과 보안성**이 크게 향상됩니다.