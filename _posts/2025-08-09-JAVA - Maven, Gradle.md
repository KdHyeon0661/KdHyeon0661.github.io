---
layout: post
title: Java - Maven, Gradle
date: 2025-08-09 23:20:23 +0900
category: Java
---
# Maven, Gradle 개요 및 비교

Java/JVM 생태계에서 가장 널리 쓰이는 빌드 도구는 **Maven**과 **Gradle**입니다.  
두 도구는 공통적으로 **의존성 관리, 빌드/테스트/패키징/배포 자동화, 멀티모듈 프로젝트**를 지원하지만, 철학과 사용성, 성능 최적화 방식에서 차이가 큽니다.

---

## 1) 한눈에 개요

- **Maven**
  - **선언형(Convention over Configuration)** 중심. `pom.xml`에 표준화된 모델로 기술.
  - 일관된 디렉터리 구조·수명주기(phase)·플러그인 중심 확장.
  - 학습 곡선 낮고, 문서/예제 풍부. “정형화된” 빌드에 강함.

- **Gradle**
  - **스크립트(DSL) 기반(구로비/Groovy 또는 코틀린/Kotlin)**, **작업(Task) 지향**.
  - 증분 빌드, 빌드캐시, 병렬화 등 **성능 최적화**에 강함.
  - 복잡한 빌드 로직·커스터마이징에 유연.

---

## 2) 기본 구조 & 핵심 개념

### Maven
- **주요 파일**
  - `pom.xml` : 프로젝트 객체 모델(Project Object Model). 그룹/아티팩트/버전(GAV), 의존성, 플러그인, 빌드 설정.
  - `settings.xml` : 레포지토리 자격증명, 미러, 프로필 등 사용자/전역 설정.
- **기본 디렉터리**
  ```
  src
   ├─ main
   │   ├─ java
   │   └─ resources
   └─ test
       ├─ java
       └─ resources
  ```
- **수명주기(대표)**
  - `validate` → `compile` → `test` → `package` → `verify` → `install` → `deploy`
- **의존성 범위(scope)**: `compile`, `provided`, `runtime`, `test`, `system`, `import(BOM)`
- **확장**: 플러그인(goal). 예) `maven-compiler-plugin`, `maven-surefire-plugin`, `maven-deploy-plugin`

예시 `pom.xml`:
```xml
<project>
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.example</groupId>
  <artifactId>demo</artifactId>
  <version>1.0.0</version>

  <properties>
    <java.version>17</java.version>
  </properties>

  <dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
      <version>3.3.0</version>
    </dependency>
    <!-- BOM 가져오기(import) -->
    <dependencyManagement>
      <dependencies>
        <dependency>
          <groupId>org.springframework.boot</groupId>
          <artifactId>spring-boot-dependencies</artifactId>
          <version>3.3.0</version>
          <type>pom</type>
          <scope>import</scope>
        </dependency>
      </dependencies>
    </dependencyManagement>
  </dependencies>

  <build>
    <plugins>
      <plugin>
        <artifactId>maven-compiler-plugin</artifactId>
        <version>3.11.0</version>
        <configuration>
          <source>${java.version}</source>
          <target>${java.version}</target>
        </configuration>
      </plugin>
    </plugins>
  </build>
</project>
```

### Gradle
- **주요 파일**
  - `build.gradle` / `build.gradle.kts` : 빌드 스크립트(DSL).
  - `settings.gradle` / `settings.gradle.kts` : 루트명/모듈 포함 설정.
  - `gradle.properties` : 공통 속성, JVM 옵션 등.
- **핵심 개념**
  - **Task**: 작업 단위(`compileJava`, `test`, `jar` 등). 의존관계 그래프로 실행.
  - **플러그인**: 기능 번들(`java`, `application`, `maven-publish`, `spring-boot` 등).
  - **구성(configuration)**: 의존성 구성(`implementation`, `api`, `runtimeOnly`, `testImplementation`).
  - **증분 빌드**와 **빌드 캐시**: 입력/출력 스냅샷 기반 변경 감지로 빠른 빌드.
- **DSL**: Groovy 또는 Kotlin(정적 타입, IDE 지원 ↑)

예시 `build.gradle.kts`:
```kotlin
plugins {
    java
    application
}

java {
    toolchain { languageVersion.set(JavaLanguageVersion.of(17)) }
}

repositories {
    mavenCentral()
}

dependencies {
    implementation("org.slf4j:slf4j-api:2.0.13")
    testImplementation("org.junit.jupiter:junit-jupiter:5.10.2")
}

tasks.test {
    useJUnitPlatform()
}

application {
    mainClass.set("com.example.Main")
}
```

---

## 3) 의존성 관리: 범위/구성, 충돌 해결

- **Maven**
  - 범위: `compile`(기본), `provided`, `runtime`, `test`, `system`.
  - **의존성 전이(Transitive)**: 상위 의존성의 의존성을 자동 포함.
  - **충돌 해결**: **근접 우선(Nearest-Wins)** + 선언 순서. `dependencyManagement`에서 버전 고정.
  - **BOM**: `scope=import`로 다수 아티팩트 버전을 일괄 관리.

- **Gradle**
  - 구성(configuration): `api`, `implementation`, `runtimeOnly`, `compileOnly`, `testImplementation` 등.
  - **버전 충돌 전략**: 기본은 **엄격한 상향 선택** 또는 정해진 전략. `resolutionStrategy` 커스터마이징.
  - **버전 카탈로그(Version Catalogs)**: `libs.versions.toml`로 라이브러리/버전 중앙 관리.
  - **Platforms/BOM**: `implementation(platform("group:artifact:version"))`로 BOM 적용.

Gradle BOM 예:
```kotlin
dependencies {
    implementation(platform("org.springframework.boot:spring-boot-dependencies:3.3.0"))
    implementation("org.springframework.boot:spring-boot-starter-web")
}
```

---

## 4) 멀티모듈(다중 프로젝트) 빌드

### Maven
- 루트 `pom.xml`에 **packaging `pom`** + `<modules>` 목록.
```xml
<project>
  <packaging>pom</packaging>
  <modules>
    <module>core</module>
    <module>api</module>
  </modules>
</project>
```

### Gradle
- `settings.gradle(.kts)`에서 `include("core", "api")`.
```kotlin
// settings.gradle.kts
rootProject.name = "demo"
include("core", "api")
```
- 각 서브프로젝트 공통 설정을 루트 `build.gradle(.kts)`에서 `subprojects {}` 또는 플러그인으로 공유.

---

## 5) 성능 & 생산성

| 항목 | Maven | Gradle |
|---|---|---|
| 증분 빌드 | 기본적(모듈 단위) | **고급 증분/입출력 추적**, 변경 시 부분 실행 |
| 빌드 캐시 | 제한적(플러그인별) | **로컬/원격 캐시** 정식 지원 |
| 병렬 | `-T` 옵션(목록/모듈 병렬) | **태스크/프로젝트 병렬** 및 워커 API |
| 스크립팅 | XML 선언형 | **DSL 기반(유연)**, 복잡한 로직 쉽게 구현 |
| 표준화 | 매우 높음 | 높음(+커스터마이징 자유도 ↑) |
| 생태계 | 방대, 예제 풍부 | 빠르게 성장, 현대적 기능 풍부 |
| 보고/진단 | 플러그인 중심 | **Build Scan**, Task 그래프 시각화, 프로파일러 |

> 대규모/빈번한 빌드 환경에서는 Gradle의 **캐시+증분+병렬**로 체감 성능 이점이 크며, 정형화된 표준 빌드·학습비용 최소화가 중요하면 Maven이 유리합니다.

---

## 6) 테스트·패키징·배포

- **Maven**
  - 테스트: Surefire(단위), Failsafe(통합) 플러그인.
  - 패키징: `jar`, `war`, `ear` 등 `packaging` 타입.
  - 배포: `install`(로컬), `deploy`(원격 레포). `maven-deploy-plugin`/`maven-release-plugin`.

- **Gradle**
  - 테스트: `test`(JUnit Platform), `integrationTest` 커스텀 소스셋 확장 용이.
  - 패키징: `jar`, `war`, `bootJar`(Spring Boot).
  - 배포: `maven-publish`로 Maven 리포지토리 퍼블리시, `signing` 연동.

Gradle `maven-publish` 예:
```kotlin
plugins {
    `maven-publish`
    signing
}

publishing {
    publications {
        create<MavenPublication>("mavenJava") {
            from(components["java"])
            groupId = "com.example"
            artifactId = "demo"
            version = "1.0.0"
        }
    }
    repositories {
        maven {
            url = uri("https://repo.example.com/releases")
            credentials { username = "user"; password = "pass" }
        }
    }
}

signing { sign(publishing.publications["mavenJava"]) }
```

---

## 7) 명령어 & 래퍼(Wrapper)

- **Maven**
  - `mvn -v`, `mvn clean package`, `mvn -T 4 install`
  - **Wrapper**: `mvn -N io.takari:maven:wrapper` → `mvnw`/`mvnw.cmd` 고정 버전 사용

- **Gradle**
  - `gradle -v`, `gradle clean build`, `gradle test --info --scan`
  - **Wrapper**: `gradle wrapper` → `gradlew`/`gradlew.bat`로 팀 내 버전 고정

---

## 8) CI/CD & 재현 가능한 빌드

- 공통: **Wrapper로 빌드 도구 버전 고정**, 사내 Nexus/Artifactory 사용, 캐시/병렬 활용.
- **Maven**: `enforcer` 플러그인으로 JDK/플러그인/버전 규칙 강제, `flatten-maven-plugin`으로 POM 정리.
- **Gradle**: **Configuration Cache**, **Build Cache**, **Version Catalog**, `--scan`으로 성능/문제 분석, `dependency-locking`으로 버전 잠금.

---

## 9) 선택 가이드(요약)

- **Maven 권장 상황**
  - 팀에 XML 기반 표준 프로세스가 익숙.
  - 빌드가 비교적 단순/정형화되어 있음.
  - 신규 인원 온보딩을 최대한 쉽게.

- **Gradle 권장 상황**
  - **대규모 멀티모듈**·**고빈도 빌드**로 빌드 시간 절감이 중요.
  - 복잡한 커스텀 빌드 로직 필요.
  - Kotlin DSL로 **정적 타입**/IDE 자동완성 극대화.

---

## 10) 자주 겪는 문제 & 팁

- **의존성 충돌**
  - Maven: `mvn dependency:tree`, `dependencyManagement`로 버전 고정.
  - Gradle: `gradle dependencies`, `dependencyInsight`, `platform()`/카탈로그로 정리.

- **JDK/도구 버전 불일치**
  - Maven: `maven-toolchain-plugin` 또는 `properties`에 `maven.compiler.release`.
  - Gradle: `java.toolchain` 사용(프로젝트별 JDK 자동 관리).

- **빌드 성능**
  - Gradle: `org.gradle.caching=true`, `org.gradle.parallel=true`, `org.gradle.configuration-cache=true`.
  - Maven: `-T` 병렬, 불필요한 플러그인/프로필 최소화.

---

## 11) 가장 짧은 스타터 예제

**Maven**
```bash
mvn -v
mvn archetype:generate -DgroupId=com.example -DartifactId=demo -DarchetypeArtifactId=maven-archetype-quickstart -DinteractiveMode=false
cd demo
mvn package
java -cp target/demo-1.0-SNAPSHOT.jar com.example.App
```

**Gradle (Kotlin DSL)**
```bash
gradle -v
gradle init --type java-application --dsl kotlin --test-framework junit-jupiter
./gradlew build
./gradlew run
```

---

## 결론

- **Maven = 안정적·표준화·학습 쉬움**, **Gradle = 빠름·유연함·현대적 기능**.  
- 팀/프로젝트의 **복잡도, 성능 요구, 커스터마이징 필요성**에 따라 도구를 선택하고, Wrapper와 버전/의존성 정책을 통해 **재현 가능한 빌드**를 보장하세요.
