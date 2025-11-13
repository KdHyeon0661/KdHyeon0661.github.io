---
layout: post
title: Java - Jigsaw 프로젝트
date: 2025-08-09 20:20:23 +0900
category: Java
---
# Jigsaw 프로젝트 개요

## 1. 배경 — 클래스패스의 구조적 한계와 Jigsaw의 필요

### 1.1 클래스패스(Classpath) 한계
- **충돌(=Classpath Hell)**: 동일 FQCN이 여러 JAR에 존재 → 로딩 순서 의존.
- **의존성 비가시성**: *필요 JAR 목록*을 타입 시스템 차원에서 표현할 수 없음.
- **캡슐화 붕괴**: `public`이면 누구나 접근(내부 구현 노출).
- **거대 배포물**: 작은 앱도 전체 JRE/JDK 동반.

### 1.2 Jigsaw의 목표(설계 원칙)
| 목표 | 의도 |
|---|---|
| **모듈화** | 코드·플랫폼을 **모듈 단위**로 쪼개고 **의존 그래프**로 관리 |
| **강한 캡슐화** | `exports`로 공개를 명시, 나머지는 감추어 내부 API를 보호 |
| **명시적 의존성** | `requires`/`requires transitive`로 **타입-체크 가능한 의존성** |
| **서비스 디스커버리** | `uses/provides`로 느슨한 결합의 플러그인 구조 |
| **경량 런타임** | `jlink`로 **필요 모듈만** 묶은 런타임 생성(크기·보안·기동 최적화) |

---

## 2. 핵심 개념 — 모듈, 모듈 그래프, 접근성·가시성

### 2.1 모듈과 `module-info.java`
- **모듈** = 패키지 묶음 + 메타데이터(이름, 의존성, 공개/개방 규칙, 서비스).
- 모듈 선언 예:
```java
module com.example.app {
    requires java.sql;                     // 의존
    requires transitive com.example.core;  // 전이 의존(사용자에게도 읽기 허용)
    exports com.example.app.api;           // 외부 공개
    opens com.example.app.json to com.fasterxml.jackson.databind; // 리플렉션 허용
    uses com.example.core.spi.Payment;     // 서비스 사용
    provides com.example.core.spi.Payment with com.example.app.pay.CardPayment; // 구현 제공
}
```

### 2.2 읽기(Readability) vs 접근성(Accessibility)
- **읽기**: A 모듈이 B 모듈을 `requires`로 **읽을 수 있는가**(타입 사용 가능).
- **접근성**: 읽을 수 있어도, 대상 **패키지가 `exports` 되었는가**가 별개.
- **리플렉션**: `exports`만으로는 비공개 멤버 접근 불가 → **`opens`** 필요.

### 2.3 모듈 종류
| 종류 | 설명 |
|---|---|
| **Named Module** | `module-info.class` 포함, 모듈 이름 명시 |
| **Automatic Module** | `module-info` 없음, 모듈패스에 두면 JAR 파일명으로 모듈명 유추(매니페스트 `Automatic-Module-Name` 권장) |
| **Unnamed Module** | 클래스패스의 모든 코드가 속하는 특수 모듈(느슨한 규칙) |

---

## 3. JDK 모듈화 — Platform as Modules

### 3.1 예시 구조(발췌)
```
java.base        # 모든 모듈의 기저 (java.lang, java.util, ...)
java.logging     # java.util.logging
java.sql         # JDBC API
java.xml         # XML
jdk.crypto.ec    # JDK 구현 모듈(elliptic curves)
```

### 3.2 실습: 내 런타임의 모듈 나열
```bash
java --list-modules
```

### 3.3 특정 JAR(또는 모듈) 분석
```bash
jar --describe-module --file libs/some-lib.jar
jdeps --module-path mods --list-deps mods/com.example.app.jar
```

---

## 4. 문법 상세 — `requires`/`exports`/`opens`/서비스

### 4.1 `requires`
- `requires M;` : 컴파일/런타임 모두 **M 읽기** 필요.
- `requires transitive M;` : 나를 **읽는 자**도 **M을 읽을 수 있게**(API 노출).
- `requires static M;` : **컴파일 시만** 필요(애너테이션 등), 런타임 optional.

### 4.2 `exports`와 `opens`
| 키워드 | 목적 | 효과 |
|---|---|---|
| `exports p` | **컴파일/런타임 API 공개** | `public` 타입 접근 허용 |
| `exports p to X` | 선택적 공개 | 지정 모듈 X만 접근 |
| `opens p` | **리플렉션 허용(런타임)** | `setAccessible` 없이도 프레임워크 바인딩 가능 |
| `opens p to X` | 선택적 리플렉션 허용 | 특정 모듈에만 |

> **정리**: API 노출은 `exports`, 리플렉션 기반 프레임워크(Jackson/Hibernate/JAXB 등)에는 `opens`를 함께 설계.

### 4.3 서비스 — `uses` / `provides ... with`
- **설계 의도**: 모듈 간 **느슨한 결합**과 **플러그인 구조**.
- SPI 정의:
```java
package com.example.core.spi;
public interface Payment { boolean pay(int amount); }
```
- 서비스 소비자:
```java
module com.example.app {
    requires com.example.core;
    uses com.example.core.spi.Payment;
}
```
- 서비스 제공자:
```java
module com.example.pay.card {
    requires com.example.core;
    provides com.example.core.spi.Payment
        with com.example.pay.card.CardPayment;
}
```
- 런타임 로딩:
```java
ServiceLoader<Payment> loader = ServiceLoader.load(Payment.class);
for (Payment p : loader) {
    if (p.pay(100)) break;
}
```

---

## 5. 도구 체인 — `jdeps`, `jar --describe-module`, `jlink`, `jmod`

### 5.1 `jdeps` — 의존성 분석
```bash
# JAR가 어떤 모듈/패키지에 의존하는지
jdeps --multi-release 9 --print-module-deps app.jar
jdeps --module-path mods --list-deps mods/com.example.app.jar
```

### 5.2 `jar --describe-module`
```bash
jar --describe-module --file mods/com.example.app.jar
```

### 5.3 `jlink` — 경량 런타임 이미지 구성
- 목적: 앱이 **필요로 하는 모듈만** 포함한 **런타임 이미지** 생성.
```bash
# 의존 모듈 확인
jdeps --print-module-deps --ignore-missing-deps mods/com.example.app.jar

# 예: 최소 런타임 생성
jlink \
  --module-path "$JAVA_HOME/jmods:mods" \
  --add-modules com.example.app \
  --strip-debug \
  --no-header-files --no-man-pages \
  --compress=2 \
  --output dist/runtime

# 실행
dist/runtime/bin/java -m com.example.app/com.example.app.Main
```

### 5.4 `jmod` — 모듈 패키징(고급)
- 네이티브 라이브러리/설치 스크립트 포함 가능한 **.jmod** 포맷 생성·검증.
```bash
jmod describe mods/com.example.core.jmod
```

---

## 6. 마이그레이션 전략 — Classpath → Modulepath

### 6.1 단계적 접근(권장)
1. **현행 유지 + 준비**
   - 모든 라이브러리에 `Automatic-Module-Name`(매니페스트) 지정 → **안정 모듈명** 확보.
   - 예: `Automatic-Module-Name: com.fasterxml.jackson.databind`.
2. **앱에 `module-info.java` 도입**
   - 가장 바깥(애플리케이션)부터 명시적 `requires/exports` 설계.
   - **split package**(하나의 패키지를 둘 모듈에 분산) 제거/통합.
3. **리플렉션 프레임워크 정리**
   - Jackson/Hibernate 등: `opens` 또는 런처 `--add-opens`로 조치.
   - 테스트: `opens ... to org.junit.platform.commons`.
4. **서비스 구조로 느슨화**
   - 플러그인/확장은 `uses/provides`로 모듈 경계 유지.
5. **배포 최적화**
   - `jlink`로 경량 런타임 생성, 컨테이너 이미지 축소.

### 6.2 빈출 이슈와 해결
| 이슈 | 원인 | 해결 |
|---|---|---|
| **Split package** | 동일 패키지가 여러 모듈에 | 패키지 리네임/합치기(모듈=패키지 경계 원칙) |
| **Illegal reflective access** | 모듈 캡슐화 차단 | `opens`(영구) 또는 `--add-opens`(임시) |
| **Automatic 모듈명 충돌** | 파일명으로 유추된 이름 충돌 | `Automatic-Module-Name` 명시 |
| **서비스 미발견** | `provides` 누락/모듈패스 미배치 | `provides` 선언·모듈패스 점검 |

---

## 7. 실전 예 — 코어/API/앱/플러그인(서비스) 구성

### 7.1 모듈 레이아웃
```
src/
 ├─ com.example.core/
 │   ├─ module-info.java
 │   └─ com/example/core/spi/Payment.java
 ├─ com.example.app/
 │   ├─ module-info.java
 │   └─ com/example/app/Main.java
 └─ com.example.pay.card/
     ├─ module-info.java
     └─ com/example/pay/card/CardPayment.java
```

### 7.2 선언 파일
```java
// src/com.example.core/module-info.java
module com.example.core {
    exports com.example.core.spi;
}
```
```java
// src/com.example.app/module-info.java
module com.example.app {
    requires com.example.core;
    uses com.example.core.spi.Payment;
    exports com.example.app.api;
}
```
```java
// src/com.example.pay.card/module-info.java
module com.example.pay.card {
    requires com.example.core;
    provides com.example.core.spi.Payment
        with com.example.pay.card.CardPayment;
}
```

### 7.3 구현 & 실행
```java
// com.example.core.spi.Payment
package com.example.core.spi;
public interface Payment { boolean pay(int amount); }
```
```java
// com.example.pay.card.CardPayment
package com.example.pay.card;
import com.example.core.spi.Payment;
public class CardPayment implements Payment {
    public boolean pay(int amount) {
        System.out.println("CARD OK: " + amount);
        return true;
    }
}
```
```java
// com.example.app.Main
package com.example.app;
import com.example.core.spi.Payment;
import java.util.ServiceLoader;
public class Main {
    public static void main(String[] args) {
        for (Payment p : ServiceLoader.load(Payment.class)) {
            if (p.pay(100)) break;
        }
    }
}
```

컴파일·패키징·실행:
```bash
# 컴파일(모듈 소스 경로 한 번에)
javac -d out --module-source-path src $(find src -name "*.java")

# 모듈 JAR로 포장
jar --create --file mods/com.example.core.jar       -C out/com.example.core .
jar --create --file mods/com.example.pay.card.jar   -C out/com.example.pay.card .
jar --create --file mods/com.example.app.jar        -C out/com.example.app .

# 실행(모듈패스 지정)
java -p mods -m com.example.app/com.example.app.Main
```

---

## 8. 리플렉션·프레임워크·테스트 — `opens`와 런처 플래그

### 8.1 프레임워크(Jackson/Hibernate) 매핑
```java
module com.example.app {
    // POJO가 있는 패키지를 리플렉션 허용
    opens com.example.app.model to com.fasterxml.jackson.databind, org.hibernate.orm.core;
}
```

### 8.2 런처 임시 플래그(마이그레이션 단계)
```bash
java --add-opens com.example.app/com.example.app.model=ALL-UNNAMED -jar app.jar
```

### 8.3 테스트에서 내부 접근(JUnit)
```java
// 테스트 전용으로만 리플렉션 허용
opens com.example.app.internal to org.junit.platform.commons;
```
Maven Surefire 예(선택):
```xml
<plugin>
  <artifactId>maven-surefire-plugin</artifactId>
  <configuration>
    <argLine>
      --add-opens com.example.app/com.example.app.internal=org.junit.platform.commons
    </argLine>
  </configuration>
</plugin>
```

---

## 9. 빌드 도구 통합(요약) — Maven/Gradle

### 9.1 Maven (핵심만)
```xml
<plugin>
  <artifactId>maven-compiler-plugin</artifactId>
  <version>3.11.0</version>
  <configuration>
    <release>17</release>
    <compilerArgs>
      <arg>--module-path</arg><arg>${project.build.outputDirectory}</arg>
    </compilerArgs>
  </configuration>
</plugin>
```
- 멀티모듈은 **모듈 간 의존**을 POM으로 관리.
- 테스트/리플렉션은 Surefire/FailSafe의 `argLine`로 `--add-opens` 조정.

### 9.2 Gradle (핵심만)
```groovy
java {
  toolchain { languageVersion = JavaLanguageVersion.of(17) }
  modularity.inferModulePath = true
}
test {
  jvmArgs '--add-opens', 'com.example.app/com.example.app.internal=org.junit.platform.commons'
}
```

---

## 10. jlink로 경량 런타임 만들기 — 컨테이너/임베디드 배포

### 10.1 절차
1. 앱 모듈 JAR과 의존 모듈 준비(`mods/`).
2. `jdeps --print-module-deps`로 필요한 플랫폼 모듈 확인.
3. `jlink`로 이미지 생성.

### 10.2 예시
```bash
jdeps --print-module-deps mods/com.example.app.jar
# 출력 예: java.base,java.logging,com.example.core,com.example.pay.card

jlink \
  --module-path "$JAVA_HOME/jmods:mods" \
  --add-modules com.example.app \
  --strip-debug --compress=2 --no-man-pages --no-header-files \
  --output image/app

image/app/bin/java -m com.example.app/com.example.app.Main
```

- 이 결과물(`image/`)만 컨테이너에 복사 → **작고 빠른** 배포 이미지.

---

## 11. 아키텍처와 성능·보안의 영향

- **보안**: 내부 API 접근 차단(`sun.*` 등), `opens`로 **정상 선언된** 리플렉션만 허용.
- **성능**: 클래스 경로 스캔 감소, 런타임 크기 축소(`jlink`)로 **기동/메모리 개선**.
- **유지보수**: 모듈 그래프를 통해 **의존성 사이클**/누수 조기 발견.

---

## 12. 고급 주제 — 동적 모듈 로딩(`ModuleLayer`), 계층 구성

### 12.1 동적 로딩 개념
- JPMS는 **계층형(ModuleLayer)** 로 모듈을 동적 로드 가능(플러그인 시스템 등).
- 기본 계층 위에 **새 레이어**를 구성하여 격리된 환경 제공.

### 12.2 코드 스케치
```java
ModuleLayer parent = ModuleLayer.boot();
ModuleFinder finder = ModuleFinder.of(Path.of("plugins"));

Configuration config = parent.configuration()
    .resolve(finder, ModuleFinder.of(), Set.of("com.example.plugin.foo"));

ClassLoader scl = ClassLoader.getSystemClassLoader();
ModuleLayer layer = parent.defineModulesWithOneLoader(config, scl);

// 레이어 내부의 클래스 로드
Class<?> c = layer.findLoader("com.example.plugin.foo")
    .loadClass("com.example.plugin.foo.Plugin");
Object plugin = c.getDeclaredConstructor().newInstance();
```

---

## 13. 트러블슈팅 표(빠른 대응)

| 증상 | 원인 | 해결 |
|---|---|---|
| `java.lang.module.FindException: Module X not found` | `-p` 경로 누락/오타 | 모듈패스 설정·JAR 존재 확인 |
| `InaccessibleObjectException` | 모듈 캡슐화로 리플렉션 차단 | `opens`(모듈 선언) 또는 `--add-opens`(런처) |
| `java.lang.LayerInstantiationException` | split-package | 패키지 경계 재설계 |
| 서비스 로딩 실패 | `provides` 누락·모듈패스 문제 | 선언·패스 점검, `ServiceLoader` 사용 모듈에서 `uses` 선언 |
| 자동 모듈명 충돌 | JAR 파일명 유추가 중복 | 매니페스트 `Automatic-Module-Name` 부여 |
| 테스트 실패(JUnit 접근) | 내부 패키지 비공개 | 테스트에 `opens ... to org.junit.*` |

---

## 14. FAQ — 자주 묻는 차이·오해 정리

- **`exports` vs `opens`**
  - `exports`: **컴파일/런타임 API 접근** 허용(타입 사용).
  - `opens`: **리플렉션 접근** 허용(런타임 전용). 보통 직렬화/ORM/바인딩 프레임워크용.

- **`requires transitive` 언제?**
  - 상위 모듈의 **공개 API에서 하위 모듈 타입**을 노출할 때(사용자에게도 하위 모듈 읽기 권한 부여).

- **자동/이름 없는 모듈은 안 써도 되나?**
  - **마이그레이션 완충**으로 매우 유용. 최종적으로는 **Named Module**로 수렴 권장.

- **OSGi와 차이점**
  - JPMS는 JDK 표준 모듈 시스템(정적 해석 중심), OSGi는 런타임 동적 모듈화·라이프사이클 관리 등 풍부한 기능. 목적·스코프가 다름.

---

## 15. 베스트 프랙티스(체크리스트)

1. **모듈=API 경계**: 내부는 감추고(`exports` 최소화), 필요한 패키지만 `opens`.
2. **전이 의존 최소화**: `requires transitive`는 **API 노출 의도**가 있을 때만.
3. **split-package 금지**: 패키지=모듈 경계 원칙 유지.
4. **서비스로 느슨화**: 플러그인/확장은 `uses/provides`.
5. **자동 모듈 이름 고정**: 라이브러리는 `Automatic-Module-Name` 제공.
6. **리플렉션 정책 명시**: 프레임워크·테스트 대상에만 구체적 `opens`.
7. **배포 최적화**: `jlink`로 경량 런타임, 컨테이너 이미지 축소.
8. **의존 분석 습관화**: `jdeps`로 의존 그래프 점검, 불필요 의존 제거.

---

## 16. 부록 — 빠른 실습 레시피

### 16.1 내 프로젝트의 모듈 의존 탐색
```bash
jdeps --print-module-deps --ignore-missing-deps target/app.jar
```

### 16.2 모듈 JAR에서 메타데이터 확인
```bash
jar --describe-module --file mods/com.example.app.jar
```

### 16.3 런타임 이미지 생성/실행
```bash
jlink --module-path "$JAVA_HOME/jmods:mods" \
      --add-modules com.example.app \
      --output image/app \
      --strip-debug --no-man-pages --no-header-files --compress=2

image/app/bin/java -m com.example.app/com.example.app.Main
```

---

## 17. 결론

Project Jigsaw는 **플랫폼과 애플리케이션을 모듈 단위로 재구성**해 **안정성(명시적 의존성)**, **보안(강한 캡슐화)**, **배포 효율(경량 런타임)** 을 동시에 끌어올렸습니다.
**마이그레이션의 핵심**은:
- **경계 정의(exports/opens)**,
- **서비스 기반 느슨화(uses/provides)**,
- **테스트·프레임워크 호환 설계**,
- **`jdeps`/`jlink` 도구 내재화**입니다.

이 원칙을 따르면 **대규모 코드베이스의 구조적 복잡도**를 줄이고, **운영·배포**에서 실질적 이득(크기·기동·보안)을 체감할 수 있습니다.
