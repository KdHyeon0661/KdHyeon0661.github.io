---
layout: post
title: Java - module-info.java 구조
date: 2025-08-09 19:20:23 +0900
category: Java
---
# `module-info.java` 구조

## 1. `module-info.java`가 담는 것: 개념 맵

- **모듈 선언**: `module` 또는 `open module`  
- **의존성**: `requires`, `requires transitive`, `requires static`  
- **공개 API**: `exports` / `exports … to` (정적 타입 접근; 컴파일·런타임 모두)  
- **리플렉션 허용**: `opens` / `opens … to` (런타임만; `setAccessible`류 허용)  
- **서비스 구성**: `uses`, `provides … with` (ServiceLoader)  

간단 형태:
```java
module com.example.app {
    requires com.example.core;
    requires transitive com.example.api;
    requires static com.example.annotations;

    exports com.example.app.api;
    exports com.example.app.spi to com.example.plugin;

    opens com.example.app.model to com.fasterxml.jackson.databind;

    uses com.example.api.PaymentService;
    provides com.example.api.PaymentService with com.example.app.CardPaymentService;
}
```

---

## 2. 키워드별 역할과 차이

| 구분 | 키워드 | 적용 시점/의미 | 접근 범위 |
|---|---|---|---|
| 모듈 선언 | `module` | 일반 모듈 | 패키지별로 `exports/opens` 선택 |
| 모듈 선언 | `open module` | **모든 패키지**를 리플렉션에 open | 각 패키지를 개별 `opens`하지 않아도 됨(보안상 신중) |
| 의존성 | `requires` | 컴파일·런타임 의존성 | 해당 모듈의 **exports된 패키지** 접근 |
| 의존성 | `requires transitive` | 의존성 **전파** | 이 모듈을 `requires`한 모듈이 자동으로 그 의존성도 읽음 |
| 의존성 | `requires static` | **컴파일 전용** | 런타임 불필요(애너테이션/IDE 헬퍼 등) |
| 공개 | `exports` | 정적 접근(컴파일/런타임) 허용 | **public 타입**만 노출, 패키지 내부는 은닉 |
| 공개(지정) | `exports … to` | 특정 모듈에만 공개 | 선택적 API 공유 |
| 반사 | `opens` | 런타임 리플렉션 허용 | private 포함 멤버 접근(직렬화/프레임워크) |
| 반사(지정) | `opens … to` | 특정 모듈에만 리플렉션 허용 | Jackson/Hibernate 등에 최소 개방 |
| 서비스 | `uses` | ServiceLoader **소비자** 선언 | 런타임 구현체 탐색 |
| 서비스 | `provides … with` | 구현 **제공자** 선언 | JAR `META-INF/services` 대체/보완 |

**핵심 차이**  
- `exports` = **정적 타입 접근** 허용(컴파일·런타임). `public` API만 노출.  
- `opens` = **리플렉션** 접근 허용(런타임). `private`까지 주입/직렬화 가능.  
- 둘은 목적과 보안 수준이 다르므로, **API는 `exports`**, 프레임워크 직렬화 등 **반사가 필요한 모델 패키지에만 `opens`**를 쓰는 것이 일반적이다.

---

## 3. 모듈 이름, 패키지, JAR의 관계

- **모듈 이름**: 보통 패키지 네이밍과 유사(역도메인; 소문자·`.`).  
- **한 모듈 = 하나의 `module-info.java`**.  
- **Split Package 금지**: **같은 패키지**를 **여러 모듈**에 나누어 담으면 에러. 패키지 경계는 모듈 내부에 **단일 소유**되도록 리팩터링(패키지 이름 세분화 권장).  
- **자동 모듈**: 모듈화되지 않은 JAR에 `Automatic-Module-Name` 매니페스트가 있으면 그 이름 사용, 없으면 파일명에서 유추. 자동 모듈은 **모든 패키지 공개**라는 부작용이 있으니 과도 의존 금지.  
- **Unnamed Module(클래스패스)**: 클래스패스의 코드는 **모듈 경계 밖**이며, **모든 exports된 패키지**를 읽을 수 있다. 반대로 **이름 있는 모듈은 unnamed을 읽지 못함**(전이적 접근 불가). 점진 마이그레이션 시 주의.

---

## 4. 설계 패턴: API / 구현 / 리플렉션

### 4.1 API/구현 분리
```
com.example.api      → exports com.example.api.* (인터페이스·DTO)
com.example.impl     → exports com.example.impl.spi to com.example.app (선택 공개)
com.example.app      → requires transitive com.example.api, requires com.example.impl
```

API는 폭넓게 `exports`, 구현은 최대한 은닉(필요 시 `exports … to`).

### 4.2 직렬화/바인딩(리플렉션) 최소 개방
```java
module com.example.bind {
    requires com.fasterxml.jackson.databind;
    exports com.example.bind.api; // 컴파일/런타임 정적 접근
    opens com.example.bind.model to com.fasterxml.jackson.databind; // 반사 허용 최소화
}
```

### 4.3 서비스 플러그인 구조
```java
module com.example.spi {
    exports com.example.spi;
    uses com.example.spi.Plugin;
}
module com.example.plugins.json {
    requires com.example.spi;
    provides com.example.spi.Plugin with com.example.plugins.json.JsonPlugin;
}
```
런타임에 `ServiceLoader.load(Plugin.class)`로 자동 검색.

---

## 5. 빌드·실행·패키징(명령행, Maven, Gradle)

### 5.1 명령행(단일 모듈 예)
```bash
# 컴파일
javac --module-source-path src -d out $(find src -name "*.java")

# 실행
java --module-path out -m com.example.app/com.example.app.Main
```

### 5.2 Maven (JPMS 인지)
```xml
<!-- pom.xml -->
<properties>
  <maven.compiler.release>17</maven.compiler.release>
</properties>
<build>
  <plugins>
    <plugin>
      <artifactId>maven-compiler-plugin</artifactId>
      <version>3.11.0</version>
      <configuration>
        <release>${maven.compiler.release}</release>
      </configuration>
    </plugin>
  </plugins>
</build>
```
- 자동으로 모듈 경로 인식.  
- 리플렉션 프레임워크 테스트 시 Surefire/Failsafe에 `--add-opens` 전달이 필요할 수 있다.

### 5.3 Gradle (Kotlin DSL)
```kotlin
plugins { java; application }

java { toolchain { languageVersion.set(JavaLanguageVersion.of(17)) } }

tasks.withType<JavaCompile>().configureEach {
    options.compilerArgs.addAll(listOf("--release", "17"))
}
tasks.test {
    // 리플렉션용 opens가 필요한 경우:
    jvmArgs("--add-opens", "com.example.app/com.example.app.model=org.junit.platform.commons")
}
application { mainClass.set("com.example.app.Main") }
```

### 5.4 jdeps + jlink 파이프라인
```bash
# 필요한 플랫폼 모듈 자동 추론
jdeps --print-module-deps --ignore-missing-deps out/com.example.app > deps.txt

# 경량 런타임 생성
jlink --module-path "$JAVA_HOME/jmods:out" \
      --add-modules com.example.app,$(cat deps.txt) \
      --strip-debug --no-header-files --no-man-pages \
      --compress=2 --output runtime-image
```

---

## 6. 자주 쓰는 문법 패턴(정리 예제)

### 6.1 `requires` 변형
```java
module com.example.compiler {
    // 런타임 불필요: 컴파일 시 애너테이션만
    requires static com.example.annotations;

    // API 전파: 이 모듈을 쓰는 쪽에서도 core가 자동 가시
    requires transitive com.example.core;
}
```

### 6.2 선택적 공개(정교한 exports/opens)
```java
module com.example.selective {
    exports com.example.selective.api; // 일반 공개
    exports com.example.selective.internal to com.example.friend; // 제한 공개

    opens com.example.selective.model to com.fasterxml.jackson.databind, org.hibernate.orm.core;
}
```

### 6.3 서비스 다중 제공자
```java
module com.example.pay {
    exports com.example.pay.spi;
    uses com.example.pay.spi.PaymentService;

    provides com.example.pay.spi.PaymentService
        with com.example.pay.impl.CardService,
             com.example.pay.impl.WalletService;
}
```

---

## 7. 테스트·프레임워크와의 상호작용

### 7.1 Jackson/Hibernate 등 리플렉션 라이브러리
- 원칙: **모델 패키지에만 `opens`** (혹은 `open module`은 지양).
- 테스트에서만 필요하면 **런처 옵션**으로 보완:
```bash
java --add-opens com.example.app/com.example.app.model=org.junit.platform.commons ...
```
Maven Surefire:
```xml
<plugin>
  <artifactId>maven-surefire-plugin</artifactId>
  <configuration>
    <argLine>
      --add-opens com.example.app/com.example.app.model=org.junit.platform.commons
    </argLine>
  </configuration>
</plugin>
```
Gradle(Test):
```kotlin
tasks.test {
    jvmArgs("--add-opens", "com.example.app/com.example.app.model=ALL-UNNAMED")
}
```

### 7.2 Unnamed(클래스패스)와 혼용
- **이름 있는 모듈**은 **클래스패스(unnamed)** 를 **자동으로 읽지 않는다**.  
- 마이그레이션 중엔 **라이브러리를 자동 모듈**로 두거나, 임시로 `--add-reads` 등으로 연결.

---

## 8. 오류·경고 트러블슈팅 매트릭스

| 증상/메시지 | 원인 | 해결 |
|---|---|---|
| `package ... is declared in module A and module B` | **Split Package** | 패키지 이름 분리/병합, 하나의 모듈로 통합 |
| `module not found: X` | 모듈 경로 누락 | `--module-path`, Maven/Gradle 의존성 확인 |
| `cannot access class ... because module ... does not export ...` | exports 없음 | 대상 패키지 `exports` 추가(또는 API 설계 변경) |
| `InaccessibleObjectException` | 리플렉션 차단 | `opens` 추가 혹은 실행 시 `--add-opens` |
| 자동 모듈 이름 충돌 | 두 JAR의 유사 이름 | 매니페스트 `Automatic-Module-Name` 부여/대체 |
| 테스트만 실패(런타임 직렬화) | 테스트 런처서 모듈 옵션 미전달 | Surefire/Gradle Test에 `--add-opens` 추가 |
| 순환 의존으로 시작 지연/복잡 | **requires** 사이클 | 설계상 제거 권장(허용은 되지만 복잡도↑) |

> **참고**: 모듈 간 **순환 의존**(A↔B)은 언어/런타임 상 **허용**되지만, 해석과 툴링이 복잡해지고 유지보수/테스트/배포에 악영향을 준다. **계층화(상향 의존 금지)**로 끊는 것이 바람직하다.

---

## 9. 서비스 로더(ServiceLoader)와 모듈

### 9.1 인터페이스/구현/소비자
```java
// com.example.spi module
package com.example.spi;
public interface Plugin { String name(); }

// 제공자 모듈
module com.example.plugin.json {
    requires com.example.spi;
    provides com.example.spi.Plugin with com.example.plugin.json.JsonPlugin;
}

// 소비자 모듈
module com.example.app {
    requires com.example.spi;
    uses com.example.spi.Plugin;
}
```

### 9.2 런타임 로딩
```java
ServiceLoader<Plugin> loader = ServiceLoader.load(Plugin.class);
for (Plugin p : loader) {
    System.out.println(p.name());
}
```
- **장점**: 모듈 경계에서 구현 은닉 유지, 플러그인 교체 용이.  
- **주의**: 제공자 JAR이 **모듈 경로**에 있어야 탐색된다(클래스패스는 unnamed로 동작).

---

## 10. 보안·유지보수 관점 가이드

1. **기본은 닫고 필요한 최소만 연다**  
   - 공개: `exports` 최소  
   - 리플렉션: `opens` 최소(가능하면 `… to`로 한정)
2. **API vs 구현** 경계 명확화  
   - 인터페이스/DTO 전용 패키지를 `exports`  
   - 구현체는 모듈 내부 은닉, 필요 시 선택 공개
3. **전이 의존성 남용 금지**  
   - `requires transitive`는 진입점·파사드 모듈에서만
4. **자동 모듈 의존 최소화**  
   - 장기적으로 모두 **명시 모듈화**(descriptor 포함)로 이전
5. **빌드·런치 옵션 관리**  
   - 테스트/운영의 `--add-opens`, `--add-exports`, `--add-reads`를 환경별 표준 스크립트에 명시

---

## 11. 마이그레이션 단계 로드맵

1. **의존성 지도 작성**: `jdeps`로 JAR 간 참조 파악  
2. **Split Package 제거**: 패키지 리팩터링  
3. **핵심 모듈 선정**: API/도메인부터 모듈화  
4. **프레임워크 호환**: Jackson/Hibernate 대상 패키지에 `opens`  
5. **빌드 통합**: Maven/Gradle 설정에 모듈 경로 반영, 테스트 런처 옵션 추가  
6. **점진적 모듈화**: 외부 라이브러리는 **자동 모듈→정식 모듈**로 교체  
7. **jlink 최적화**: 런타임 이미지 최소화, 배포 아티팩트 경량화

---

## 12. 모듈 레이어(고급): 플러그인 격리 로딩

여러 버전의 플러그인을 **격리된 로더**로 올릴 때:
```java
ModuleLayer parent = ModuleLayer.boot();
Configuration conf = parent.configuration()
   .resolve(ModuleFinder.of(Path.of("plugins")), ModuleFinder.of(), Set.of("com.example.plugin.json"));
ClassLoader scl = ClassLoader.getSystemClassLoader();
ModuleLayer layer = parent.defineModulesWithOneLoader(conf, scl);

// layer에서 ServiceLoader 실행
ServiceLoader<Plugin> loader = ServiceLoader.load(layer, Plugin.class);
```
- 각 레이어는 모듈 가시성·격리를 제공. 대형 IDE/서버 플러그인 구조에서 유용.

---

## 13. 리소스·리플렉션·도구 상식

- **리소스 로드**(`Class.getResource*`)는 모듈화와 무관하게 동작하나, 모듈 경로 상 JAR의 구조를 따른다.  
- **`opens`는 컴파일에는 영향 없음**(오직 런타임 반사). 반대로 `exports`는 컴파일 가시성에도 영향.  
- **`jar --module-version`**으로 JAR에 모듈 버전을 기록하면 `ModuleDescriptor.version()`에서 조회 가능.  
- **멀티릴리스 JAR(MRJAR)**와 `module-info.class`는 **루트에 위치**해야 한다(버전별 디렉터리 아래에 두지 않는다).

---

## 14. 실전 예시: 세 모듈 구성

### 14.1 디렉터리
```
src/
 ├─ com.example.core/
 │   ├─ module-info.java
 │   └─ com/example/core/...
 ├─ com.example.api/
 │   ├─ module-info.java
 │   └─ com/example/api/...
 └─ com.example.app/
     ├─ module-info.java
     └─ com/example/app/...
```

### 14.2 module-info.java

**core**
```java
module com.example.core {
    exports com.example.core.util;
    // 구현 패키지는 exports하지 않음
}
```

**api**
```java
module com.example.api {
    requires transitive com.example.core;     // API 사용자에게 core 노출
    exports com.example.api.dto;
    exports com.example.api.spi;              // SPI는 필요 시 선택 공개도 가능
}
```

**app**
```java
module com.example.app {
    requires com.example.api;                 // api가 core를 전이하므로 이곳에는 불필요
    requires com.fasterxml.jackson.databind;

    exports com.example.app.api;              // 앱의 외부 API
    opens com.example.app.model to com.fasterxml.jackson.databind; // 직렬화 대상

    uses com.example.api.spi.Plugin;          // 플러그인 소비
}
```

### 14.3 컴파일·실행
```bash
javac --module-source-path src -d out $(find src -name "*.java")
java  --module-path out -m com.example.app/com.example.app.Main
```

---

## 15. 체크리스트

- [ ] **Split Package**가 없는가? (패키지 경계 = 모듈 경계)  
- [ ] `exports`는 **필요 최소**로만 열었는가?  
- [ ] 리플렉션 대상만 **정밀 `opens … to`**로 지정했는가?  
- [ ] `requires transitive`는 파사드/진입점에서만 사용했는가?  
- [ ] 자동 모듈 의존은 **임시**로만 쓰고 있는가?  
- [ ] 테스트 런처에 필요한 `--add-opens`/`--add-exports`가 반영됐는가?  
- [ ] `jdeps`로 의존성 무결성을 주기적으로 점검하는가?  
- [ ] `jlink`로 배포 런타임을 경량화할 계획이 있는가?

---

## 16. 자주 묻는 질문(요약)

**Q1. `exports`와 `opens`를 둘 다 써야 하나요?**  
A. 목적이 다르다. **API 공개**는 `exports`, **반사(직렬화/프레임워크)**는 `opens`. 필요에 따라 병행 가능.

**Q2. 클래스패스(unnamed)에서 모듈을 읽을 수 있나요?**  
A. 클래스패스 코드는 **exports된 패키지**를 읽을 수 있다. 반대로 **이름 있는 모듈**은 클래스패스를 읽지 못한다.

**Q3. 순환 의존이 가능한가요?**  
A. 가능하지만 권장하지 않는다. 해석/툴링 복잡·테스트 어려움. 설계로 제거하라.

**Q4. Hibernate/Jackson이 동작하지 않아요.**  
A. 대상 모델 패키지에 `opens`가 필요(또는 실행 옵션 `--add-opens`).

**Q5. 자동 모듈로 충분한가요?**  
A. 단기 우회책일 뿐이다. 보안/가시성/버전 관리 측면에서 **명시 모듈화**로 이행하라.

---

## 17. 완성형 예제: 최소 앱 + 서비스 제공자

**`com.example.spi/module-info.java`**
```java
module com.example.spi {
    exports com.example.spi;
}
```

**`com.example.spi/com/example/spi/Greeting.java`**
```java
package com.example.spi;
public interface Greeting { String say(String name); }
```

**`com.example.provider/module-info.java`**
```java
module com.example.provider {
    requires com.example.spi;
    provides com.example.spi.Greeting with com.example.provider.KoreanGreeting;
}
```

**`com.example.provider/com/example/provider/KoreanGreeting.java`**
```java
package com.example.provider;
import com.example.spi.Greeting;
public class KoreanGreeting implements Greeting {
    public String say(String name) { return "안녕하세요, " + name; }
}
```

**`com.example.app/module-info.java`**
```java
module com.example.app {
    requires com.example.spi;
    uses com.example.spi.Greeting;
}
```

**`com.example.app/com/example/app/Main.java`**
```java
package com.example.app;
import com.example.spi.Greeting;
import java.util.ServiceLoader;

public class Main {
    public static void main(String[] args) {
        String name = args.length > 0 ? args[0] : "World";
        ServiceLoader<Greeting> loader = ServiceLoader.load(Greeting.class);
        loader.findFirst()
              .ifPresentOrElse(
                  g -> System.out.println(g.say(name)),
                  () -> System.out.println("No provider found"));
    }
}
```

**빌드/실행(모듈 경로 포함)**
```bash
javac --module-source-path src -d out $(find src -name "*.java")
java  --module-path out -m com.example.app/com.example.app.Main "개발자"
```

---

## 18. 한 장 표 — 문법 요약

| 문법 | 목적 | 컴파일 시 영향 | 런타임 영향 | 노출 범위 |
|---|---|---|---|---|
| `exports p` | API 공개 | O | O | `public` 타입만 |
| `exports p to M` | 선택 API 공개 | O | O | M에만 |
| `opens p` | 리플렉션 허용 | X | O | 모든 멤버(프라이빗 포함) |
| `opens p to M` | 선택 리플렉션 | X | O | M에만 |
| `requires X` | 의존성 | O | O | X의 exports 접근 |
| `requires transitive X` | 의존 전파 | O | O | 소비자도 X 접근 |
| `requires static X` | 컴파일 전용 | O | X | 컴파일 도움 |
| `uses S` | 서비스 소비 | - | O | ServiceLoader 기반 |
| `provides S with I` | 서비스 제공 | - | O | 구현 등록 |

---

## 19. 마무리

`module-info.java`는 단순 선언 파일이 아니라 **아키텍처의 계약서**다.  
- **API/구현/리플렉션/서비스**를 구분하고,  
- **최소 공개·정밀 개방** 원칙으로 보안을 유지하며,  
- **빌드·런타임·테스트**의 모듈 옵션을 일관되게 관리하면,  

대규모 코드베이스의 **유지보수성·성능·보안성**을 장기적으로 끌어올릴 수 있다.