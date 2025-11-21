---
layout: post
title: Java - Java 모듈 선언, 의존성, exports
date: 2025-08-09 21:20:23 +0900
category: Java
---
# Java 모듈 선언, 의존성, exports

## 한눈에 핵심(요약)

- **module-info.java**에 모듈의 **의존성**(`requires`), **공개 범위**(`exports`), **리플렉션 허용**(`opens`), **서비스 노출/소비**(`provides/uses`)를 선언한다.
- `requires transitive`로 **전이 의존성**을 공개할 수 있고, `requires static`은 **컴파일 전용** 의존성이다.
- 런타임 리플렉션(예: Jackson/Hibernate)은 `exports`가 아니라 **`opens`** 필요. (선언적 캡슐화 vs 리플렉션 허용의 차이)
- **클래스패스**와 **모듈패스**는 다르다. 초기 마이그레이션은 **자동/이름 없는 모듈**을 이해하면 한층 수월하다.
- 서비스(플러그인) 아키텍처는 `uses` + `provides ... with` + `ServiceLoader`로 모듈 경계에서 깔끔하게 구현 가능.

---

## 모듈 선언 `module` — 구조·규칙·레이아웃

### module-info.java 기본

```java
module com.example.myapp {
    requires java.sql;                 // 의존성 선언
    exports com.example.myapp.api;     // 외부에 공개할 패키지
}
```

**규칙/관례**
- 모듈 이름은 **역도메인** 형태 권장(e.g., `com.company.project`).
- 한 모듈은 **하나의 `module-info.java`**만 가진다.
- `module-info.java`는 **소스 루트**(모듈 루트)에 위치한다.

### 권장 폴더 레이아웃(멀티 모듈 예)

```
root/
 ├─ app/                     (module: com.example.app)
 │   └─ src/main/java/
 │       ├─ module-info.java
 │       └─ com/example/app/...
 ├─ core/                    (module: com.example.core)
 │   └─ src/main/java/
 │       ├─ module-info.java
 │       └─ com/example/core/...
 └─ impl/                    (module: com.example.impl)
     └─ src/main/java/
         ├─ module-info.java
         └─ com/example/impl/...
```

---

## 의존성 `requires` — 기본/전이/컴파일 전용

### 기본 의존성

```java
module com.example.myapp {
    requires java.sql;
    requires com.example.util;
}
```
- 선언된 모듈의 **공개 패키지**에 접근 가능.

### 전이 의존성 `requires transitive`

```java
module com.example.service {
    requires transitive com.example.core;
    exports com.example.service.api;
}
```
- **`com.example.service`를 사용하는 모듈**은 별도 선언 없이 **`com.example.core`에도 접근 가능**.
- 상위 레이어가 **하위 레이어 API를 외부에 재노출**해야 할 때 유용하나 **남용 금지**.

### 컴파일 전용 `requires static`

```java
module com.example.compiler {
    requires static com.example.annotations; // 컴파일 시에만 필요
    exports com.example.compiler.api;
}
```
- 예: 애너테이션/애너테이션 프로세서 등 **런타임 불필요** 의존성.

### JDK 기본 모듈 예시

| 모듈명       | 설명                                                   |
|--------------|--------------------------------------------------------|
| `java.base`  | **암시적으로 포함**. `java.lang`, `java.util` 등 필수 |
| `java.sql`   | JDBC API                                               |
| `java.xml`   | XML 처리                                               |
| `java.logging` | JUL(logging)                                        |

---

## 패키지 공개 `exports` — 전체 공개 vs 선택적 공개

### 기본 공개

```java
module com.example.myapp {
    exports com.example.myapp.api; // public 타입 외부 접근 허용
}
```
- 공개하지 않으면 **모듈 내부에서만** 접근 가능(강한 캡슐화).

### 특정 모듈에만 공개(친구 모듈)

```java
module com.example.myapp {
    exports com.example.myapp.internal to com.example.test; // test 모듈만 접근
}
```
- 테스트/툴링 모듈에만 제한적 공개가 가능.

---

## `exports` vs `opens` — 리플렉션 차이(매우 중요)

### 차이 요약

- `exports`: **컴파일·런타임의 정적 접근** 허용(소스/바이트코드 참조). 리플렉션으로 **비공개 멤버 접근 허용 아님**.
- `opens`: **런타임 리플렉션**(예: `setAccessible(true)`)을 허용. 프레임워크(Jackson/Hibernate/MapStruct 등) 호환에 필수.

### 구문

```java
module com.example.app {
    exports com.example.app.api;                    // 정적 공개
    opens   com.example.app.model to com.fasterxml.jackson.databind; // jackson에만 리플렉션 허용
}
```

### 모두 열기(권장 X)

```java
open module com.example.app {  // 모듈 내 모든 패키지를 리플렉션에 개방
    requires com.fasterxml.jackson.databind;
}
```
- 초기 마이그레이션 시 편하지만 **보안/캡슐화 약화**. 필요 패키지에 **선택적 `opens`**가 바람직.

---

## — `uses` + `provides ... with` + `ServiceLoader`

### API 모듈

```java
// module-info.java (com.example.calc.api)
module com.example.calc.api {
    exports com.example.calc.api;
    uses com.example.calc.api.Calculator; // 소비자들을 위해 공개(소비 선언은 보통 앱/라이브러리 쪽)
}
```

```java
// com/example/calc/api/Calculator.java
package com.example.calc.api;
public interface Calculator {
    String name();
    int compute(int a, int b);
}
```

### 구현 모듈(플러그인)

```java
// module-info.java (com.example.calc.add)
module com.example.calc.add {
    requires com.example.calc.api;
    provides com.example.calc.api.Calculator
        with com.example.calc.add.AddCalculator; // 구현 클래스 등록
}
```

```java
// com/example/calc/add/AddCalculator.java
package com.example.calc.add;
import com.example.calc.api.Calculator;
public class AddCalculator implements Calculator {
    public String name() { return "add"; }
    public int compute(int a, int b) { return a + b; }
}
```

### 소비자(애플리케이션)

```java
// module-info.java (com.example.app)
module com.example.app {
    requires com.example.calc.api;
    uses com.example.calc.api.Calculator; // 서비스 사용 선언
}
```

```java
// com/example/app/Main.java
package com.example.app;
import com.example.calc.api.Calculator;
import java.util.ServiceLoader;

public class Main {
    public static void main(String[] args) {
        for (Calculator c : ServiceLoader.load(Calculator.class)) {
            System.out.printf("[%s] 3 ? 5 = %d%n", c.name(), c.compute(3,5));
        }
    }
}
```

**실행 시** 모듈패스에 `com.example.calc.add`가 존재하면 자동으로 **발견/주입**된다.

---

## 모듈 패스 vs 클래스패스, 자동/이름 없는 모듈

### 용어

- **모듈패스**: 모듈(JAR 안에 `module-info.class`가 있거나 **Automatic-Module-Name** 매니페스트로 이름이 지정된 JAR)을 배치.
- **클래스패스**: 기존 방식. 모듈 경계/검증 없음.

### 자동 모듈(automatic module)

- `module-info.class`가 없어도, JAR 파일명이 **모듈 이름**으로 간주되어 **모듈패스**에서 사용 가능.
- `MANIFEST.MF`에 `Automatic-Module-Name: com.example.lib`를 넣어 **안정적 이름** 부여 권장.

### 이름 없는 모듈(unnamed module)

- **클래스패스**에 놓인 모든 코드 → **하나의 이름 없는 모듈**로 취급.
- 이름 없는 모듈은 **모든 모듈을 읽을 수 있으나**, **모듈들은 이름 없는 모듈을 읽지 못함**.
- 점진적 마이그레이션 시 흔히 발생. 처음엔 앱을 **클래스패스**로, 라이브러리/부분을 모듈패스로 두는 혼합 형태가 가능하지만, 최종적으로 **모듈패스 정착**을 권장.

---

## 컴파일/패키징/실행(순수 JDK 도구)

예시는 `core`, `add`(플러그인), `app` 3모듈 구성.

### 컴파일

```bash
# 모듈 소스 컴파일

javac -d out/core $(find core/src/main/java -name "*.java")

# add 모듈(의존: core)

javac --module-path out -d out/add  $(find add/src/main/java -name "*.java")

# app 모듈(의존: core)

javac --module-path out -d out/app  $(find app/src/main/java -name "*.java")
```

### 모듈 JAR 만들기

```bash
jar --create --file mods/com.example.core.jar -C out/core .
jar --create --file mods/com.example.calc.add.jar -C out/add .
jar --create --file mods/com.example.app.jar -C out/app .
```

### 실행

```bash
java --module-path mods \
     --module com.example.app/com.example.app.Main
```

### 서비스 누락 시

`ServiceLoader`로 아무 구현을 찾지 못하면 루프가 비어 있음. 구현 JAR을 **모듈패스에 추가**해야 한다.

---

## `exports` + 테스트: 제한 공개와 테스트 모듈

테스트 전용 API 접근이 필요할 때:
```java
module com.example.core {
    exports com.example.core.internal to com.example.core.tests; // 친구 모듈
}
```
테스트 모듈에서:
```java
module com.example.core.tests {
    requires com.example.core;
}
```

**대안**: 테스트 시에만 JVM 옵션으로 강제 열기(초기 마이그레이션에 유용).
```bash
java --add-exports com.example.core/com.example.core.internal=ALL-UNNAMED ...
```

---

## 리플렉션 호환 — `opens`, `--add-opens`, `open module`

- 프레임워크(예: Jackson/Hibernate/Spring)에서 **프라이빗 필드/생성자 접근**이 필요하다면 `opens`가 필수.
- 선택적 개방:
  ```java
  module com.example.app {
      opens com.example.app.model to com.fasterxml.jackson.databind;
  }
  ```
- 임시 우회(테스트/디버깅용):
  ```bash
  java --add-opens com.example.app/com.example.app.model=com.fasterxml.jackson.databind ...
  ```
- 전체 개방(비권장):
  ```java
  open module com.example.app { ... }
  ```

---

## 트러블슈팅(현장에서 자주 만나는 문제)

| 증상/에러 | 원인 | 해결 |
|---|---|---|
| `java.lang.module.FindException: Module not found` | 모듈패스에 JAR 누락 | `--module-path`에 해당 모듈 JAR 추가 |
| `java.lang.IllegalAccessException: module ... does not "opens"` | 리플렉션 대상 패키지가 `opens`되지 않음 | `opens` 선언 또는 `--add-opens` 옵션 사용 |
| `java.lang.module.ResolutionException: Conflicting module` | 같은 모듈 이름 중복(두 JAR 모두 `Automatic-Module-Name` 같음) | 모듈 이름 충돌 해결(하나 제거/이름 변경) |
| Split-package 경고/에러 | **같은 패키지**가 **서로 다른 모듈**로 분할됨 | 패키지 재조정(패키지 경계 = 모듈 경계) |
| 순환 의존성 | `A` ↔ `B`가 서로 `requires` | 의존성 방향 재설계 또는 인터페이스 분리 |

---

## 베스트 프랙티스

1. **exports 최소화**: API만 공개, 구현은 패키지 캡슐화.
2. **transitive 절제 사용**: 외부에 불필요한 의존성 누수 방지.
3. **필요 패키지에만 opens**: 리플렉션 최소 범위에서 허용.
4. **Automatic-Module-Name** 지정: 타사 JAR도 안정적 이름을 가지게(라이브러리 제작 시 강추).
5. **Split-package 회피**: 패키지 경계와 모듈 경계를 일치시킨다.
6. **서비스 로더로 플러그인 구조**: `uses/provides` + `ServiceLoader`로 확장 포인트 제공.

---

## 실전 예제 — API/구현/앱 + 리플렉션 + 테스트 제한 공개

### API 모듈

```java
// calc-api/module-info.java
module com.example.calc.api {
    exports com.example.calc.api;
    uses com.example.calc.api.Calculator;
}
```

```java
// calc-api/com/example/calc/api/Calculator.java
package com.example.calc.api;
public interface Calculator {
    String name();
    int compute(int a, int b);
}
```

### 구현 모듈(반사 필요 모델 포함)

```java
// calc-impl/module-info.java
module com.example.calc.impl {
    requires com.example.calc.api;
    provides com.example.calc.api.Calculator
            with com.example.calc.impl.MulCalculator;
    opens com.example.calc.impl.model to com.fasterxml.jackson.databind; // 리플렉션 허용
}
```

```java
// calc-impl/com/example/calc/impl/MulCalculator.java
package com.example.calc.impl;
import com.example.calc.api.Calculator;
public class MulCalculator implements Calculator {
    public String name() { return "mul"; }
    public int compute(int a, int b) { return a * b; }
}
```

```java
// calc-impl/com/example/calc/impl/model/Factor.java
package com.example.calc.impl.model;
public class Factor { private int a; private int b; public int getA(){return a;} public int getB(){return b;} }
```

### 앱 모듈(테스트용 제한 공개)

```java
// app/module-info.java
module com.example.app {
    requires com.example.calc.api;
    uses com.example.calc.api.Calculator;
    exports com.example.app.internal to com.example.app.tests; // 테스트 모듈만 접근
}
```

```java
// app/com/example/app/Main.java
package com.example.app;
import com.example.calc.api.Calculator;
import java.util.ServiceLoader;

public class Main {
    public static void main(String[] args) {
        ServiceLoader.load(Calculator.class)
                     .forEach(c -> System.out.println(c.name() + ": " + c.compute(6,7)));
    }
}
```

### 테스트 모듈

```java
// app-tests/module-info.java
module com.example.app.tests {
    requires com.example.app;
    // com.example.app의 internal 패키지를 exports to로 열어 두었기 때문에 접근 가능
}
```

---

## Maven/Gradle 팁(요약)

### Maven

- `maven-compiler-plugin`은 JDK 9+에서 **모듈패스**를 인식.
- 라이브러리 JAR에 모듈 이름을 고정하려면 `Automatic-Module-Name` 매니페스트를 설정.
```xml
<plugin>
  <artifactId>maven-compiler-plugin</artifactId>
  <version>3.11.0</version>
  <configuration>
    <release>17</release>
  </configuration>
</plugin>
```

### Gradle

- Java 플러그인 사용 시 Gradle이 모듈패스/클래스패스를 자동 구성하려 시도.
- 초기 이슈가 있으면 `--patch-module`, `--add-reads`, `--add-opens`를 `tasks.withType(JavaExec)` 등에 추가해 점진 적용.

---

## 마이그레이션 전략(단계별)

1. **현황 파악**: split-package, 리플렉션 사용 지점(프레임워크/라이브러리) 식별.
2. **Automatic-Module-Name** 부여(라이브러리) → 안정적 모듈 이름 확보.
3. **앱/핵심 라이브러리부터 모듈화**: `module-info.java` 작성, `exports` 최소화.
4. 리플렉션 필요한 패키지만 **선택적으로 `opens`** (또는 초기엔 `--add-opens` 옵션으로 점진 이행).
5. **서비스 경계**로 API/구현 분리: `uses/provides`.
6. CI에 **모듈 모드 빌드/테스트** 추가, JVM 옵션 제거 방향으로 정리.

---

## 치트시트

```text
module M { ... }                         # 모듈 선언
requires A;                              # A 모듈을 사용
requires transitive B;                   # M을 쓰는 모듈도 B에 접근 가능
requires static C;                       # 컴파일 시만 필요, 런타임 불필요
exports p.q;                             # p.q 패키지를 공개
exports p.q to X,Y;                      # X,Y 모듈에만 공개
opens p.q;                               # p.q 리플렉션 허용
opens p.q to X;                          # X에게만 리플렉션 허용
open module M { ... }                    # 모듈 전체 리플렉션 허용(비권장)
uses a.b.Service;                        # 서비스 사용 선언
provides a.b.Service with impl.Cls;      # 구현체 등록
```

**명령어**
```bash
javac --module-path mods -d out/m  $(find m/src/main/java -name "*.java")
jar --create --file mods/m.jar -C out/m .
java --module-path mods --module m/pkg.Main
```

**임시 옵션(트러블슈팅)**
```bash
--add-reads m=n                 # m 모듈이 n 모듈을 강제로 읽도록
--add-exports m/p.q=x           # m의 p.q 패키지를 x 모듈에 강제 export
--add-opens m/p.q=x             # m의 p.q 패키지를 x에 리플렉션 허용
```

---

## 결론

- **`requires`/`exports`/`opens`/`uses`/`provides`**를 이해하면, 모듈 경계에서 **명확한 캡슐화와 제어된 개방**을 동시에 달성할 수 있다.
- 마이그레이션은 **모듈패스/클래스패스 혼용**과 **자동/이름 없는 모듈**을 적절히 활용해 단계적으로 진행하라.
- 프레임워크 리플렉션은 **`opens`(선택적)** 로 최소 범위만 허용하고, 플러그인 구조는 **서비스 로더**로 우아하게 해결하자.

---
```java
// 빠른 참고: 대표 선언 모음
module com.example.app {
    requires transitive com.example.core;
    requires static com.example.annotations;
    exports com.example.app.api;
    exports com.example.app.internal to com.example.app.tests;
    opens com.example.app.model to com.fasterxml.jackson.databind;
    uses com.example.spi.Plugin;
    provides com.example.spi.Plugin with com.example.app.impl.DefaultPlugin;
}
```
