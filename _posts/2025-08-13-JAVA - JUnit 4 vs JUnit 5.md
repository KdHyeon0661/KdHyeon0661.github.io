---
layout: post
title: Java - JUnit 4 vs JUnit 5
date: 2025-08-13 16:20:23 +0900
category: Java
---
# JUnit 4 vs JUnit 5 — 자세 비교 가이드

JUnit은 Java 테스트의 표준입니다. **JUnit 5**는 “JUnit Platform + Jupiter + Vintage”로 재설계되어 **확장성, 표현력, 실행 성능/유연성**이 크게 향상되었습니다. 아래에서는 **핵심 개념·애너테이션 매핑·기능 차이·의존성·마이그레이션 팁**을 체계적으로 정리합니다.

---

## 1) 큰 그림 비교

| 항목 | JUnit 4 | JUnit 5 |
|---|---|---|
| 아키텍처 | 단일 프레임워크 | **Platform(런처) + Engine** 구조 (Jupiter=JUnit5, Vintage=JUnit4 호환) |
| 최소 Java | 5+ | 8+ (엔진/도구에 따라 상향 가능) |
| 실행 단위 | Runner(@RunWith) 1개만 지정 | **여러 엔진/런처 통합**, 조건/필터/태깅 강화 |
| 확장 | `@Rule`, `@ClassRule`, Runner 교체 | **Extension 모델** (`@ExtendWith`, 내장 조건/파라미터 리졸버) |
| 가시성 제약 | 테스트 클래스/메서드 **public** 필요 | **public 불필요**(패키지 프라이빗 가능) |
| 파라미터화 | `@RunWith(Parameterized)` 불편 | **`@ParameterizedTest`** + 다양한 소스 제공 |
| 예외/타임아웃 | `@Test(expected, timeout)` 또는 `ExpectedException` Rule | **`assertThrows`**, `@Timeout`, `assertTimeout` |
| 중첩/동적 테스트 | 미지원 | **`@Nested`**, **동적 테스트**(`@TestFactory`) |
| 태깅/카테고리 | `@Category`(외부 lib) | **`@Tag`** (엔진/빌드에서 필터링 용이) |
| 비활성화 | `@Ignore` | **`@Disabled`** (+ 조건부 `@EnabledOnOs` 등) |
| 병렬 실행 | 빌드 도구 의존 | **Platform 레벨 병렬**(설정 파일로 제어) |

---

## 2) 애너테이션 매핑(가장 많이 쓰는 것)

| 목적 | JUnit 4 | JUnit 5 (Jupiter) |
|---|---|---|
| 테스트 | `@Test` | `@Test`, `@DisplayName`(이름 꾸미기) |
| 사전/사후(각 테스트) | `@Before`, `@After` | `@BeforeEach`, `@AfterEach` |
| 사전/사후(클래스 전체) | `@BeforeClass`, `@AfterClass` (static 필요) | `@BeforeAll`, `@AfterAll` (기본 static, 단 `@TestInstance(PER_CLASS)`면 인스턴스 메서드 가능) |
| 비활성화 | `@Ignore` | `@Disabled("이유")` |
| 예외 | `@Test(expected=...)`, `ExpectedException` Rule | `Assertions.assertThrows(...)` |
| 타임아웃 | `@Test(timeout=...)` | `@Timeout`, `assertTimeout(Preemptively)` |
| 카테고리/태그 | (외부 라이브러리 `@Category`) | `@Tag("slow")` |
| 파라미터화 | Runner `@RunWith(Parameterized)` | `@ParameterizedTest` + `@ValueSource`/`@CsvSource`/`@MethodSource`… |
| 중첩 | 없음 | `@Nested` |
| 조건부 | `Assume.*` | `Assumptions.*`, `@EnabledOnOs`, `@EnabledIfEnvironmentVariable` 등 |

---

## 3) 핵심 기능 차이 상세

### 3.1 확장 모델: Rule/Runner → Extension
- **JUnit 4**: 기능 확장은 `Runner` 교체(1개만 가능) 또는 `@Rule`/`@ClassRule` 조합.
- **JUnit 5**: **Extension** 포인트(파라미터 주입, 라이프사이클 훅, 예외/타임아웃/조건 처리 등)로 통합. 여러 확장을 **동시에** 적용 가능.
```java
// JUnit 5: Mockito 예 — 확장 등록
@ExtendWith(MockitoExtension.class)
class UserServiceTest { /* ... */ }
```

### 3.2 파라미터화 테스트
```java
// JUnit 4 (간략)
@RunWith(Parameterized.class)
public class ParamTest {
  @Parameters public static Object[] data() { return new Object[]{1,2,3}; }
  private final int n;
  public ParamTest(int n){ this.n = n; }
  @Test public void ok(){ /* ... */ }
}
```
```java
// JUnit 5
@ParameterizedTest
@ValueSource(strings = {"a","bb","ccc"})
void lengthAtLeastOne(String s) {
  assertTrue(s.length() >= 1);
}
```

### 3.3 예외/타임아웃
```java
// JUnit 4
@Test(expected = IllegalArgumentException.class)
public void invalid() { throw new IllegalArgumentException(); }
```
```java
// JUnit 5
@Test
void invalid() {
  assertThrows(IllegalArgumentException.class, () -> { throw new IllegalArgumentException(); });
}
```
```java
// 타임아웃 (JUnit 5)
@Test
void slow() {
  assertTimeout(Duration.ofMillis(200), () -> service.call());
}
```

### 3.4 중첩/동적 테스트
```java
// 중첩 테스트 (JUnit 5)
@Nested
class WhenUserExists {
  @Test void canLoad() { /* ... */ }
}
```
```java
// 동적 테스트 (JUnit 5)
@TestFactory
Stream<DynamicTest> dynamic() {
  return Stream.of(1,2,3)
    .map(n -> DynamicTest.dynamicTest("case " + n, () -> assertTrue(n > 0)));
}
```

### 3.5 태그 & 필터링
- **JUnit 5**: `@Tag("integration")` → Maven/Gradle/IDE에서 **포함/제외** 필터링 쉬움.
- **JUnit 4**: `@Category`(외부 라이브러리) 사용 빈번.

### 3.6 병렬 실행
- **JUnit 5**: `junit-platform.properties`에서 병렬 전략 설정 가능.
- **JUnit 4**: Surefire/Gradle 설정에 의존.

---

## 4) 의존성(빌드 설정) 예

### Maven

```xml
<!-- JUnit 4 -->
<dependency>
  <groupId>junit</groupId>
  <artifactId>junit</artifactId>
  <version><!-- 최신 4.x --></version>
  <scope>test</scope>
</dependency>

<!-- JUnit 5 (Jupiter) -->
<dependency>
  <groupId>org.junit.jupiter</groupId>
  <artifactId>junit-jupiter-api</artifactId>
  <version><!-- 최신 5.x --></version>
  <scope>test</scope>
</dependency>
<dependency>
  <groupId>org.junit.jupiter</groupId>
  <artifactId>junit-jupiter-engine</artifactId>
  <version><!-- 최신 5.x --></version>
  <scope>test</scope>
</dependency>

<!-- JUnit 4 테스트를 JUnit 5 플랫폼에서 돌리기 (Vintage) -->
<dependency>
  <groupId>org.junit.vintage</groupId>
  <artifactId>junit-vintage-engine</artifactId>
  <version><!-- 5.x --></version>
  <scope>test</scope>
</dependency>

<!-- 파라미터화 -->
<dependency>
  <groupId>org.junit.jupiter</groupId>
  <artifactId>junit-jupiter-params</artifactId>
  <version><!-- 5.x --></version>
  <scope>test</scope>
</dependency>
```

**Surefire/Failsafe** 플러그인은 JUnit 5를 인식할 수 있는 버전을 사용하세요(최신 권장).

### Gradle (Kotlin DSL)

```kotlin
dependencies {
    // JUnit 4
    testImplementation("junit:junit:<4.x>")

    // JUnit 5
    testImplementation(platform("org.junit:junit-bom:<5.x>"))
    testImplementation("org.junit.jupiter:junit-jupiter-api")
    testRuntimeOnly("org.junit.jupiter:junit-jupiter-engine")
    testImplementation("org.junit.jupiter:junit-jupiter-params") // 파라미터화

    // Vintage (선택: JUnit4 실행)
    testRuntimeOnly("org.junit.vintage:junit-vintage-engine")
}

tasks.test {
    useJUnitPlatform()   // JUnit 5 플랫폼 활성화
}
```

---

## 5) 마이그레이션 가이드 (4 → 5)

1. **플랫폼 전환**: 빌드에서 **JUnit Platform**을 활성화(Maven Surefire/Gradle `useJUnitPlatform()`).
2. **점진 전환**: 기존 JUnit 4 테스트는 **Vintage 엔진**으로 계속 실행 → 새 테스트부터 Jupiter로 작성.
3. **애너테이션/기능 치환**(대표 매핑):
   - `@Before/@After` → `@BeforeEach/@AfterEach`
   - `@BeforeClass/@AfterClass` → `@BeforeAll/@AfterAll`
   - `@Ignore` → `@Disabled`
   - `@Rule(TemporaryFolder)` → `@TempDir`
   - `ExpectedException` / `@Test(expected=..) ` → `assertThrows`
   - `@Category` → `@Tag`
   - `@RunWith(MockitoJUnitRunner.class)` → `@ExtendWith(MockitoExtension.class)`
4. **가시성 정리**: `public` 강제 제거 가능(더 깔끔한 테스트 클래스).
5. **테스트 이름/리포트 품질**: `@DisplayName`, `@DisplayNameGeneration`으로 가독성 향상.
6. **조건부/환경 의존 테스트**: `@EnabledOnOs`, `@EnabledIfEnvironmentVariable` 등으로 명시.

---

## 6) 실용 팁

- **Assertions**: JUnit 5의 `assertAll`, `assertIterableEquals`, `assertLinesMatch` 등 **표현력↑**.  
- **DI & 인젝션**: `TestInfo`, `TestReporter`, 커스텀 `ParameterResolver`로 컨텍스트 주입 가능.  
- **Suite 실행**: 5에서는 `junit-platform-suite`로 선택자/필터 기반 수트 구성.  
- **Timeout 전략**: 어플리케이션 특성에 따라 `assertTimeout`(코드 완료까지 대기) vs `assertTimeoutPreemptively`(선제 중단) 선택.  
- **병렬**: `junit-platform.properties`에서 `junit.jupiter.execution.parallel.enabled=true` 등으로 제어.  
- **IDE/CI**: 최신 IDE(인텔리J/이클립스)·CI는 기본적으로 Jupiter 엔진 지원.

---

## 7) 결론

- **새 프로젝트**라면 **JUnit 5 (Jupiter)**가 기본 선택입니다.  
- 기존 JUnit 4 자산은 **Vintage 엔진**으로 “그대로” 돌리면서, **확장/표현/성능** 이점이 큰 **Jupiter**로 점진 전환하세요.  
- 핵심 전환 포인트는 **Extension 모델**, **파라미터화 테스트**, **태깅/조건**, **예외/타임아웃 API**입니다.