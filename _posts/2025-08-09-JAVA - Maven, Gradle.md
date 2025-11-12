---
layout: post
title: Java - Maven, Gradle
date: 2025-08-09 23:20:23 +0900
category: Java
---
# Maven, Gradle 개요 및 비교

## 0. 개요

- **Maven**: 선언형(XML)·관례 중심. 표준 수명주기/플러그인으로 **안정적**이고 **예측 가능한** 빌드. 온보딩 용이.
- **Gradle**: DSL 기반(Task 그래프)·증분/캐시·병렬화로 **빠르고 유연**. 복잡한 빌드/대규모 멀티모듈에서 강력.

선택 기준(요약):

| 상황 | 권장 |
|---|---|
| 정형화된 프로젝트, 표준 우선, 팀 경험이 Maven 중심 | **Maven** |
| 빌드 시간 절감, 복잡한 커스터마이징, 대규모 멀티모듈/모노레포 | **Gradle** |
| Spring Boot/Android 대규모, 캐시/병렬 극대화 | **Gradle** |
| 사내 표준이 Maven, 규정 준수가 최우선 | **Maven** |

---

## 1. 철학과 동작 모델

### 1.1 Maven — Convention over Configuration
- **POM 모델**로 GAV(Group/Artifact/Version), 의존성, 플러그인을 선언.
- **수명주기(phase)**에 **goal**(플러그인 실행)을 바인딩하여 실행.
- 디렉터리 구조·패키징 타입·플러그인 구성이 **관례로 표준화**.

간단 구조:
```
validate → compile → test → package → verify → install → deploy
```

### 1.2 Gradle — Task Graph & Incremental
- **Task** 간 의존 그래프를 구성해 필요한 작업만 실행(증분).
- **입출력 스냅샷**, **빌드 캐시(로컬/원격)**, **병렬 실행**으로 속도 최적화.
- Groovy/Kotlin DSL로 로직을 **코드처럼** 표현(조건/반복/함수화).

Task 예(개념):
```
:compileJava → :processResources → :classes → :jar → :check → :build
```

---

## 2. 기본 골격 비교(예제 포함)

### 2.1 디렉터리 관례(공통)
```
src
 ├─ main
 │   ├─ java
 │   └─ resources
 └─ test
     ├─ java
     └─ resources
```

### 2.2 Maven 최소 POM
```xml
<!-- pom.xml -->
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
                             http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.example</groupId>
  <artifactId>demo</artifactId>
  <version>1.0.0</version>

  <properties>
    <maven.compiler.release>17</maven.compiler.release>
  </properties>

  <dependencies>
    <dependency>
      <groupId>org.slf4j</groupId>
      <artifactId>slf4j-api</artifactId>
      <version>2.0.13</version>
    </dependency>
  </dependencies>

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
</project>
```

### 2.3 Gradle 최소 Kotlin DSL
```kotlin
// settings.gradle.kts
rootProject.name = "demo"
```

```kotlin
// build.gradle.kts
plugins {
    java
    application
}

java {
    toolchain { languageVersion.set(JavaLanguageVersion.of(17)) }
}

repositories { mavenCentral() }

dependencies {
    implementation("org.slf4j:slf4j-api:2.0.13")
    testImplementation("org.junit.jupiter:junit-jupiter:5.10.2")
}

tasks.test { useJUnitPlatform() }

application {
    mainClass.set("com.example.Main")
}
```

---

## 3. 의존성 관리 — 범위/구성, 전이, BOM/카탈로그, 충돌 해결

### 3.1 Maven
- **범위(scope)**: `compile`(기본), `provided`, `runtime`, `test`, `system`, `import(BOM)`
- **전이 의존성**: 상위 의존의 의존성을 자동 포함
- **충돌 해결**: **가장 가까운(Nearest) 선언 우선** → 명시 버전 고정은 `dependencyManagement`
- **BOM**: 여러 아티팩트 버전을 **일괄 고정**

BOM 적용:
```xml
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

<dependencies>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
  </dependency>
</dependencies>
```

### 3.2 Gradle
- **구성(configuration)**: `api`/`implementation`/`compileOnly`/`runtimeOnly`/`testImplementation` …
  - `api`는 상위 모듈의 **공개 API**(하위 의존 전파), `implementation`은 내부 구현(은닉)
- **버전 카탈로그**(`libs.versions.toml`)로 버전·별칭 중앙 관리
- **플랫폼(BOM)** 적용: `implementation(platform("group:artifact:ver"))`
- **충돌 전략**: 기본 선출 + `resolutionStrategy` 커스터마이징 가능

버전 카탈로그 예:
```toml
# gradle/libs.versions.toml
[versions]
junit = "5.10.2"
slf4j = "2.0.13"

[libraries]
junit = { group = "org.junit.jupiter", name = "junit-jupiter", version.ref = "junit" }
slf4j = { group = "org.slf4j", name = "slf4j-api", version.ref = "slf4j" }
```

```kotlin
// build.gradle.kts
dependencies {
    implementation(libs.slf4j)
    testImplementation(libs.junit)
}
```

---

## 4. 멀티모듈/모노레포

### 4.1 Maven 멀티모듈
루트 POM:
```xml
<project>
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.example</groupId>
  <artifactId>demo-parent</artifactId>
  <version>1.0.0</version>
  <packaging>pom</packaging>

  <modules>
    <module>core</module>
    <module>api</module>
    <module>app</module>
  </modules>
</project>
```

모듈 간 의존:
```xml
<dependency>
  <groupId>com.example</groupId>
  <artifactId>core</artifactId>
  <version>${project.version}</version>
</dependency>
```

### 4.2 Gradle 멀티프로젝트
```
demo
 ├─ settings.gradle.kts
 ├─ build.gradle.kts (루트 공통)
 ├─ core/build.gradle.kts
 ├─ api/build.gradle.kts
 └─ app/build.gradle.kts
```

```kotlin
// settings.gradle.kts
rootProject.name = "demo"
include("core", "api", "app")
```

루트에서 공통 설정:
```kotlin
// build.gradle.kts (root)
subprojects {
    apply(plugin = "java")

    repositories { mavenCentral() }

    extensions.configure<JavaPluginExtension> {
        toolchain { languageVersion.set(JavaLanguageVersion.of(17)) }
    }

    dependencies {
        testImplementation("org.junit.jupiter:junit-jupiter:5.10.2")
    }
    tasks.test { useJUnitPlatform() }
}
```

모듈 간 의존:
```kotlin
// api/build.gradle.kts
dependencies {
    implementation(project(":core"))
}
```

**팁**: Gradle은 **convention plugin**(build-logic)으로 공통 설정을 캡슐화하면 유지보수가 쉬워집니다.

---

## 5. 성능 — 증분·캐시·병렬·Configuration Cache

| 항목 | Maven | Gradle |
|---|---|---|
| 증분 | 모듈 단위(플러그인 의존) | **입출력 스냅샷 기반** Task 증분 |
| 캐시 | 제한적 | **로컬/원격 빌드 캐시** |
| 병렬 | `-T` 옵션(모듈 병렬) | **Task/Project 병렬**(Worker API) |
| 구성 캐시 | - | **Configuration Cache**(초기 구성 시간 단축) |
| 빌드 스캔 | 플러그인 | **--scan**으로 프로파일/공유 |

Gradle 성능 플래그(gradle.properties):
```
org.gradle.parallel=true
org.gradle.caching=true
org.gradle.configuration-cache=true
org.gradle.jvmargs=-Xmx2g -Dfile.encoding=UTF-8
```

---

## 6. 테스트 전략(단위/통합), 소스셋 확장

### 6.1 Maven
- **Surefire**(단위), **Failsafe**(통합) 분리:
```xml
<plugin>
  <artifactId>maven-surefire-plugin</artifactId>
  <version>3.2.5</version>
  <configuration>
    <includes>
      <include>**/*Test.java</include>
    </includes>
  </configuration>
</plugin>

<plugin>
  <artifactId>maven-failsafe-plugin</artifactId>
  <version>3.2.5</version>
  <executions>
    <execution>
      <goals><goal>integration-test</goal><goal>verify</goal></goals>
    </execution>
  </executions>
</plugin>
```

### 6.2 Gradle
- **소스셋**으로 통합 테스트 분리:
```kotlin
// build.gradle.kts
sourceSets {
    val integrationTest by creating {
        compileClasspath += sourceSets.main.get().output + configurations.testRuntimeClasspath.get()
        runtimeClasspath += output + compileClasspath
        java.srcDir("src/integrationTest/java")
        resources.srcDir("src/integrationTest/resources")
    }
}
configurations {
    val integrationTestImplementation by getting { extendsFrom(configurations.testImplementation.get()) }
    val integrationTestRuntimeOnly by getting { extendsFrom(configurations.testRuntimeOnly.get()) }
}
tasks.register<Test>("integrationTest") {
    description = "Runs integration tests."
    group = "verification"
    testClassesDirs = sourceSets["integrationTest"].output.classesDirs
    classpath = sourceSets["integrationTest"].runtimeClasspath
    useJUnitPlatform()
    shouldRunAfter("test")
}
tasks.check { dependsOn("integrationTest") }
```

---

## 7. 패키징·실행·배포

### 7.1 공통 개념
- **JAR**: 라이브러리/실행 JAR
- **Fat/Shadow JAR**: 의존 JAR을 하나로 병합(충돌/라이선스 주의)
- **Spring Boot**: Boot 플러그인(Gradle `bootJar`, Maven `spring-boot-maven-plugin`)
- **퍼블리시**: 사내/중앙 리포지토리(Nexus, Artifactory)

### 7.2 Maven 실행형 JAR
```xml
<plugin>
  <artifactId>maven-jar-plugin</artifactId>
  <version>3.3.0</version>
  <configuration>
    <archive>
      <manifest>
        <mainClass>com.example.Main</mainClass>
      </manifest>
    </archive>
  </configuration>
</plugin>
```

### 7.3 Gradle 실행형 JAR
```kotlin
tasks.jar {
    manifest { attributes["Main-Class"] = "com.example.Main" }
}
```

### 7.4 Fat JAR(Gradle Shadow 예)
```kotlin
plugins { id("com.github.johnrengelman.shadow") version "8.1.1" }
tasks.shadowJar {
    archiveClassifier.set("all")
    mergeServiceFiles() // ServiceLoader 리소스 병합
}
```

### 7.5 퍼블리시

**Maven**
```xml
<distributionManagement>
  <repository>
    <id>releases</id>
    <url>https://repo.example.com/releases</url>
  </repository>
</distributionManagement>
```

**Gradle**
```kotlin
plugins { `maven-publish`; signing }

publishing {
    publications {
        create<MavenPublication>("mavenJava") {
            from(components["java"])
            groupId = "com.example"; artifactId = "demo"; version = "1.0.0"
        }
    }
    repositories {
        maven {
            url = uri("https://repo.example.com/releases")
            credentials { username = findProperty("repoUser") as String; password = findProperty("repoPass") as String }
        }
    }
}
signing { sign(publishing.publications["mavenJava"]) }
```

---

## 8. JPMS(모듈 시스템)·jlink·jdeps와의 연계

### 8.1 Maven
```xml
<properties>
  <maven.compiler.release>17</maven.compiler.release>
</properties>
```
JPMS 사용 시 `module-info.java` 포함, 테스트/프레임워크 리플렉션은 `--add-opens` 혹은 설계상 `opens` 사용.

### 8.2 Gradle
```kotlin
tasks.withType<JavaCompile>().configureEach {
    options.compilerArgs.addAll(listOf("--release", "17"))
}
```

**jlink** 파이프라인(공통 아이디어):
1) `jdeps --print-module-deps`로 필요한 모듈 파악  
2) `jlink --module-path "$JAVA_HOME/jmods:mods" --add-modules com.example.app ...`

---

## 9. CI/CD — 재현 가능 빌드

### 공통 원칙
- **Wrapper**로 도구 버전 고정: `mvnw` / `gradlew`
- 중앙 리포 미러/캐시, 사내 프록시, 의존성 잠금(BOM/Locking)
- 병렬/캐시 활성화

**Maven**
- `maven-enforcer-plugin`으로 JDK/플러그인 버전 규칙 강제
- `-T 4` 병렬 빌드

**Gradle**
- `--scan`으로 성능/문제 리포트
- 원격 캐시(예: Gradle Enterprise 또는 사내 캐시 서버)로 **CI 빌드 재사용**

---

## 10. 마이그레이션 가이드

### 10.1 Maven → Gradle
1. `gradle init`으로 기본 변환(의존성/소스세트 반영)
2. BOM → `platform(...)` 또는 버전 카탈로그로 치환
3. Maven profile → Gradle **플래그/프로퍼티/전역 조건**로 매핑
4. 멀티모듈은 `settings.gradle.kts` + convention plugin으로 공통화
5. CI에서 캐시/병렬/Configuration Cache 활성화

### 10.2 Gradle → Maven
1. 의존성/버전/플러그인을 POM으로 반영 (`dependencyManagement`로 버전 중앙화)
2. 커스텀 Task 로직은 Maven 플러그인/스크립트(또는 Exec 플러그인)로 전환
3. 프로필 기반 빌드 조건 설정

---

## 11. Android·Spring Boot·Kotlin·Annotation Processing

- **Android**: Gradle 표준. Android Gradle Plugin(AGP) 사용.
- **Spring Boot**: Maven/Gradle 모두 원활. Gradle의 `bootRun`/`bootJar`가 생산적.
- **Kotlin**: Gradle Kotlin DSL과 궁합↑. Maven도 `kotlin-maven-plugin`으로 가능.
- **Annotation Processing**:
  - Maven: `maven-compiler-plugin`의 `annotationProcessorPaths`
  - Gradle: `annotationProcessor("...")` / Kotlin은 `kapt("...")`

Maven APT 예:
```xml
<plugin>
  <artifactId>maven-compiler-plugin</artifactId>
  <configuration>
    <annotationProcessorPaths>
      <path>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <version>1.18.34</version>
      </path>
    </annotationProcessorPaths>
  </configuration>
</plugin>
```

Gradle APT 예:
```kotlin
dependencies {
    annotationProcessor("org.projectlombok:lombok:1.18.34")
    compileOnly("org.projectlombok:lombok:1.18.34")
}
```

---

## 12. 트러블슈팅 매트릭스

| 증상 | Maven 원인/해결 | Gradle 원인/해결 |
|---|---|---|
| 의존성 버전 충돌 | `mvn dependency:tree`, `dependencyManagement`로 버전 고정 | `gradle dependencyInsight`, `platform()`/카탈로그/`resolutionStrategy` |
| 빌드 느림 | 불필요 플러그인 제거, `-T` 병렬 | 캐시/병렬/Configuration Cache 활성화, 무효화 Task 점검 |
| 테스트가 모듈 접근 실패 | `--add-opens` 또는 설계상 `opens` | `jvmArgs("--add-opens", "...")` 또는 `opens` |
| Fat JAR 실행 실패 | 리소스 충돌/Service 파일 병합 누락 | Shadow `mergeServiceFiles()` 사용, 충돌 전략 설정 |
| 릴리즈 서명/배포 실패 | settings/server 크리덴셜·GPG | `publishing/signing` 자격 증명·환경 변수 확인 |

---

## 13. 보안·정책·라이선스

- **Checksum 검증**: 사내 리포지토리 프록시(Nexus/Artifactory)에서 검증/미러링
- **서명**: Maven `maven-gpg-plugin`, Gradle `signing`
- **라이선스 감사**: Maven/Gradle 모두 OSS 라이선스 리포트 플러그인 활용

---

## 14. 의사결정 체크리스트

1. 팀의 기존 경험/온보딩 속도는? → Maven 유리
2. 빌드 시간/대규모 멀티모듈? → Gradle 유리
3. 복잡한 커스텀 로직/플러그인 개발? → Gradle 유리
4. 규정 준수/감사 흔한 환경? → Maven 생태계 성숙
5. Android/현대 Spring 대규모? → Gradle 지지

---

## 15. 명령어 치트시트

**Maven**
```bash
mvn -v
mvn clean package -DskipTests
mvn test -Dtest=MyTest
mvn deploy -P release
mvn dependency:tree
```

**Gradle**
```bash
gradle -v
./gradlew clean build -x test
./gradlew test --tests "com.example.MyTest"
./gradlew publish
./gradlew dependencies
./gradlew dependencyInsight --dependency slf4j-api
./gradlew --scan
```

Wrapper 생성:
```bash
# Maven
mvn -N io.takari:maven:wrapper
# Gradle
gradle wrapper
```

---

## 16. 실전 템플릿 — 최소/가벼운 멀티모듈

### 16.1 Maven Parent + Modules
```xml
<!-- parent/pom.xml -->
<project>
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.example</groupId><artifactId>parent</artifactId><version>1.0.0</version>
  <packaging>pom</packaging>
  <modules><module>core</module><module>app</module></modules>
  <properties><maven.compiler.release>17</maven.compiler.release></properties>
</project>
```

```xml
<!-- core/pom.xml -->
<project>
  <parent><groupId>com.example</groupId><artifactId>parent</artifactId><version>1.0.0</version></parent>
  <artifactId>core</artifactId>
</project>
```

```xml
<!-- app/pom.xml -->
<project>
  <parent><groupId>com.example</groupId><artifactId>parent</artifactId><version>1.0.0</version></parent>
  <artifactId>app</artifactId>
  <dependencies>
    <dependency><groupId>com.example</groupId><artifactId>core</artifactId><version>${project.version}</version></dependency>
  </dependencies>
</project>
```

### 16.2 Gradle Root + Subprojects
```kotlin
// settings.gradle.kts
rootProject.name = "demo"
include("core", "app")
```

```kotlin
// build.gradle.kts (root)
subprojects {
    apply(plugin = "java")
    repositories { mavenCentral() }
    extensions.configure<JavaPluginExtension> {
        toolchain { languageVersion.set(JavaLanguageVersion.of(17)) }
    }
    dependencies { testImplementation("org.junit.jupiter:junit-jupiter:5.10.2") }
    tasks.test { useJUnitPlatform() }
}
```

```kotlin
// app/build.gradle.kts
dependencies { implementation(project(":core")) }
```

---

## 결론

- **Maven**은 **표준화·안정성·학습 비용 절감**에, **Gradle**은 **유연성·성능·대규모 확장성**에 강점이 있습니다.  
- 팀의 맥락(규정/경험/규모/성능 요구)에 맞추어 선택하되, 공통적으로 **Wrapper 고정, 의존성 버전 중앙 관리(BOM/카탈로그), 캐시/병렬 활용, CI 재현성**을 확보하는 것이 핵심입니다.  
- 모듈 시스템(JPMS)·jlink·jdeps·컨테이너 최적화까지 고려하면 **장기 유지보수성/보안/배포 효율**을 크게 높일 수 있습니다.