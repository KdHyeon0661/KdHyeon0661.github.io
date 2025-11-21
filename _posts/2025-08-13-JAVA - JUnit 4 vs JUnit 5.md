---
layout: post
title: Java - JUnit 4 vs JUnit 5
date: 2025-08-13 16:20:23 +0900
category: Java
---
# JUnit 4 vs JUnit 5 — 자세 비교 가이드 (2025-11 업데이트 포함)

JUnit은 자바 생태계의 사실상 표준 테스트 프레임워크다. 이 글은 **JUnit 4와 JUnit 5(Jupiter)** 를 **개념/애너테이션 매핑/확장 모델/파라미터화/병렬·조건 실행/의존성/마이그레이션** 관점에서 세세히 비교한다.
또한 **2025-11 현재 JUnit 6.0.1 GA**가 공개되어, *JUnit 5의 아키텍처(Platform/Jupiter/Vintage)를 계승*하면서 **기본 JDK 요구치(17)** 상향, 일부 기능 추가/정리되었음을 먼저 알린다. 실무에서는 본문에서 “JUnit 5”로 설명하는 **Jupiter API**를 **JUnit 6에서도 동일하게** 사용한다. (의존성 버전만 6.x로 조정)

---

## 큰 그림 — 아키텍처·개념 비교

| 항목 | JUnit 4 | JUnit 5 (Jupiter) |
|---|---|---|
| 아키텍처 | 단일 프레임워크 | **Platform(런처)** + **Engine** 구조. Jupiter(5), Vintage(4), 다른 엔진 공존 가능 |
| 확장 방법 | `Runner` 1개 선택, `@Rule/@ClassRule` | **Extension** 모델(`@ExtendWith`) — 여러 확장 동시 적용 |
| 실행 단위 | Runner 기반 | **Platform**가 **태깅/필터/선택자/병렬** 등을 통합 제공 |
| 가시성 | 테스트 `public` 요구 | `public` 불필요(패키지 프라이빗 가능) |
| 파라미터화 | `@RunWith(Parameterized)` | **`@ParameterizedTest`** + 다양한 소스 제공 |
| 예외/타임아웃 | `@Test(expected, timeout)` 또는 Rule | **`assertThrows`**, `@Timeout`, `assertTimeout(Preemptive)` |
| 동적/중첩 테스트 | X | **`@Nested`, `@TestFactory`** (동적) |
| 조건 실행 | `Assume.*` | **`@EnabledOnOs/@EnabledOnJre`**, `@EnabledIfEnvironmentVariable` 등 |
| 병렬 실행 | 빌드 도구 의존 | **Platform 레벨 병렬**(`junit-platform.properties`) 설정 지원 |

> JUnit 5의 전체 구조(Platform/Jupiter/Vintage), 확장 모델, 병렬/조건 실행, 파라미터화/동적 테스트 등은 **JUnit 사용자 가이드**에서 공식 규격으로 문서화되어 있다.

---

## 애너테이션 매핑: 4 → 5 (자주 쓰는 것)

| 목적 | JUnit 4 | JUnit 5 (Jupiter) |
|---|---|---|
| 테스트 | `@Test` | `@Test`, `@DisplayName`(표기) |
| 전/후(각 테스트) | `@Before`, `@After` | `@BeforeEach`, `@AfterEach` |
| 전/후(클래스) | `@BeforeClass`, `@AfterClass`(static) | `@BeforeAll`, `@AfterAll` (보통 static, **`@TestInstance(PER_CLASS)`면 인스턴스 메서드 가능**) |
| 비활성화 | `@Ignore` | `@Disabled("이유")` |
| 예외 | `@Test(expected=...)`, `ExpectedException` Rule | **`assertThrows`** |
| 타임아웃 | `@Test(timeout=...)` | `@Timeout`, `assertTimeout(Preemptively)` |
| 카테고리/태그 | `@Category`(외부) | **`@Tag`** |
| 파라미터화 | Runner `@RunWith(Parameterized)` | **`@ParameterizedTest`** (+ `@ValueSource/@CsvSource/@MethodSource` 등) |
| 임시 디렉터리 | `@Rule TemporaryFolder` | **`@TempDir`** (내장 확장) |

> **마이그레이션 가이드** 및 **내장 확장(@TempDir)** 은 사용자 가이드에 정리되어 있다.

---

## 예제 — 동일 시나리오를 4/5로 비교

### 기본 테스트

```java
// JUnit 4
import org.junit.*;

public class CalculatorTest {
    @Before public void setUp() { /* ... */ }
    @After  public void tearDown() { /* ... */ }

    @Test
    public void add() {
        Assert.assertEquals(4, 2 + 2);
    }
}
```

```java
// JUnit 5 (Jupiter)
import org.junit.jupiter.api.*;

class CalculatorTest {
    @BeforeEach void setUp() { /* ... */ }
    @AfterEach  void tearDown() { /* ... */ }

    @Test
    void add() {
        Assertions.assertEquals(4, 2 + 2);
    }
}
```

### 예외/타임아웃

```java
// JUnit 4
@Test(expected = IllegalArgumentException.class)
public void invalid() {
    throw new IllegalArgumentException();
}
```

```java
// JUnit 5
@Test
void invalid() {
    Assertions.assertThrows(IllegalArgumentException.class,
        () -> { throw new IllegalArgumentException(); });
}
```

```java
// JUnit 5: 타임아웃
@Test
void slow() {
    Assertions.assertTimeout(java.time.Duration.ofMillis(200),
        () -> someService.call());
}
```

> `assertThrows`, `@Timeout`/`assertTimeout`은 Jupiter API로 제공된다.

### 파라미터화 테스트

```java
// JUnit 5
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.*;

class LengthTest {
    @ParameterizedTest
    @ValueSource(strings = {"a", "bb", "ccc"})
    void lengthAtLeastOne(String s) {
        Assertions.assertTrue(s.length() >= 1);
    }

    @ParameterizedTest
    @CsvSource({
        "alice,5",
        "bob,3"
    })
    void lengthMatches(String s, int expected) {
        Assertions.assertEquals(expected, s.length());
    }
}
```

> `@ParameterizedTest`와 `@ValueSource/@CsvSource` 등은 Jupiter Params 모듈에서 제공된다.

### 중첩/동적 테스트

```java
// 중첩
@Nested
class WhenUserExists {
    @Test void canLoad() { /* ... */ }
}
```

```java
// 동적 테스트
@TestFactory
java.util.stream.Stream<DynamicTest> dynamic() {
    return java.util.stream.Stream.of(1,2,3)
      .map(n -> DynamicTest.dynamicTest("case " + n, () -> Assertions.assertTrue(n > 0)));
}
```

> 중첩/동적 테스트는 Jupiter가 공식 지원한다.

### 조건부/환경 의존 테스트

```java
import org.junit.jupiter.api.condition.*;

@EnabledOnOs({OS.LINUX, OS.MAC})
@Test
void onlyOnUnixLike() { /* ... */ }

@EnabledIfEnvironmentVariable(named="CI", matches="true")
@Test
void onlyOnCi() { /* ... */ }
```

> 조건부 실행 애너테이션은 OS/JRE/환경변수/시스템프로퍼티 기반으로 필터링한다.

### 임시 디렉터리 (Rule → Extension)

```java
// JUnit 5
class FileTest {
    @org.junit.jupiter.api.io.TempDir
    java.nio.file.Path temp;

    @Test
    void writeFile() throws Exception {
        java.nio.file.Files.writeString(temp.resolve("a.txt"), "hello");
    }
}
```

> Jupiter의 `@TempDir` 내장 확장은 JUnit 4의 `TemporaryFolder` Rule 대체 수단이다.

---

## 병렬 실행 & 설정 파일

JUnit 5/6에서는 **`junit-platform.properties`** 로 병렬 실행을 포함한 다양한 동작을 설정한다.

**병렬 관련 주요 속성(예시)**

```properties
# 병렬 실행 활성화

junit.jupiter.execution.parallel.enabled=true
# 모드: same_thread / concurrent

junit.jupiter.execution.parallel.mode.default=concurrent
# 클래스/메서드 레벨 정책

junit.jupiter.execution.parallel.mode.classes.default=concurrent
```

> 병렬 실행을 제어하는 공식 속성 키는 사용자 가이드에 명시돼 있다.

Vintage(=JUnit 4)를 Platform에서 실행할 때 병렬 비활성화 등 **엔진별 설정 키**도 존재한다.

```properties
# Vintage 엔진 병렬 비활성화 예

junit.vintage.execution.parallel.enabled=false
```

> 엔진별 설정 키 역시 사용자 가이드의 설정 섹션에 정의돼 있다.

---

## 의존성(2025-11 최신 예시)

> **JUnit 6.0.1 GA** 기준. Jupiter API/Params/Engine 등 아티팩트 구성은 **5 → 6** 으로 버전만 상향되었다. (기본 JDK는 **17**)

### Maven (Jupiter 6.x)

```xml
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.junit</groupId>
      <artifactId>junit-bom</artifactId>
      <version>6.0.1</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>

<dependencies>
  <!-- 테스트 API -->
  <dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter-api</artifactId>
    <scope>test</scope>
  </dependency>
  <!-- 실행 엔진 -->
  <dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter-engine</artifactId>
    <scope>test</scope>
  </dependency>
  <!-- 파라미터화 -->
  <dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter-params</artifactId>
    <scope>test</scope>
  </dependency>
  <!-- 선택: JUnit 4 테스트도 함께 실행 (Vintage) -->
  <dependency>
    <groupId>org.junit.vintage</groupId>
    <artifactId>junit-vintage-engine</artifactId>
    <scope>test</scope>
  </dependency>
</dependencies>
```

- **Bill of Materials(BOM)** 로 버전을 일괄 관리하는 방식은 사용자 가이드에 공식화되어 있다.
- Maven Surefire/FailSafe는 **최신(≥ 3.0.0)** 권장. 사용자 가이드에서 최소권장 버전이 안내된다.

### Gradle (Kotlin DSL, 6.x)

```kotlin
dependencies {
    testImplementation(platform("org.junit:junit-bom:6.0.1"))
    testImplementation("org.junit.jupiter:junit-jupiter-api")
    testRuntimeOnly("org.junit.jupiter:junit-jupiter-engine")
    testImplementation("org.junit.jupiter:junit-jupiter-params")
    // 선택: JUnit4 실행
    testRuntimeOnly("org.junit.vintage:junit-vintage-engine")
}
tasks.test {
    useJUnitPlatform()
}
```

---

## 확장 모델: Runner/Rule → Extension

JUnit 4의 `Runner`(단일 선택), `@Rule/@ClassRule` 조합은 JUnit 5에서 **Extension** 포인트로 통합·일원화되었다. 파라미터 리졸버, 라이프사이클 인터셉터, 조건/예외/타임아웃, 임시 디렉터리 등 공통 패턴이 **확장 API**로 제공된다.

```java
// 예) MockitoExtension
@org.junit.jupiter.api.extension.ExtendWith(org.mockito.junit.jupiter.MockitoExtension.class)
class UserServiceTest {
    // @Mock, @InjectMocks 등과 결합
}
```

---

## 파라미터화 테스트 — Jupiter Params 모듈

- **소스**: `@ValueSource`, `@CsvSource`, `@CsvFileSource`, `@EnumSource`, `@MethodSource`, `@ArgumentsSource`
- **널/빈 처리**: `@NullSource`, `@EmptySource`, `@NullAndEmptySource` (원시형은 변환 필요)

실무 팁
- **표현력**: `@DisplayName`로 읽기 좋은 케이스명 제공
- **데이터 분리**: `@MethodSource`에 스트림/컬렉션을 반환해 복잡한 케이스 구성
- **대규모 데이터**: CSV 파일 외부화(`@CsvFileSource`)

---

## 조건/태깅/선택자

- **태그**: `@Tag("integration")` → 빌드/IDE/CI에서 **포함/제외** 필터링
- **선택자**: 패키지/클래스/메서드/태그 기반 실행은 **JUnit Platform**의 런처에서 처리
- **조건부**: OS/JRE/환경변수/시스템 프로퍼티/표현식 기반 활성/비활성화 애너테이션 제공

---

## 마이그레이션 가이드 (4 → 5/6)

1. **플랫폼 전환**: 빌드 도구에서 **JUnit Platform** 활성화(Maven Surefire/Gradle `useJUnitPlatform()`).
2. **점진 전환**: 기존 JUnit 4 테스트는 **Vintage 엔진**으로 계속 실행하고, 신규는 **Jupiter**로 작성. (엔진 공존 가능)
3. **애너테이션 치환**
   - `@Before/@After` → `@BeforeEach/@AfterEach`
   - `@BeforeClass/@AfterClass` → `@BeforeAll/@AfterAll` (+ `@TestInstance(PER_CLASS)` 고려)
   - `@Ignore` → `@Disabled`
   - `ExpectedException`/`@Test(expected=...)` → **`assertThrows`**
   - `@Category` → **`@Tag`**
   - `TemporaryFolder` Rule → **`@TempDir`**
4. **확장 모델로 통합**: `Runner/Rule`를 Jupiter **Extension**로 대체. Mockito/AssertJ/Spring 등은 JUnit 5/6 지원 익스텐션 제공.
5. **병렬/조건/태그**: Platform 설정/애너테이션으로 이관.
6. **버전/플러그인 최신화**: BOM(5.x 또는 6.x) + Surefire/FailSafe/Gradle 최신. (JUnit 6은 JDK **17+**)

---

## 테스트 품질·표현력·성능 팁

- **Assertions**: `assertAll`, `assertIterableEquals`, `assertLinesMatch` 등 활용으로 실패 메시지 가독성↑
- **`@DisplayName`**/**`@DisplayNameGeneration`** 으로 리포트 품질 향상
- **DI/컨텍스트 주입**: `TestInfo`, `TestReporter`, 커스텀 `ParameterResolver`
- **Suite**: `junit-platform-suite`로 선택자/필터 기반 수트 구성 (패키지·태그·클래스 조합)
- **Timeout 전략**: `assertTimeout`(대기) vs `assertTimeoutPreemptively`(선제 중단) — 환경/리소스에 맞게 선택
- **병렬화**: I/O 바운드/독립 리소스는 `concurrent`, 전역 상태 공유는 `same_thread` 또는 리소스 격리
- **junit-platform.properties** 를 **프로젝트 루트/test 리소스**에 두고, CI마다 프로파일링해 적정 병렬도/전략 도출

---

## 실전 스니펫 모음

### JUnit 4 테스트를 JUnit 5/6 Platform에서 실행 (Vintage)

```xml
<!-- Maven: JUnit 4 자산 유지 -->
<dependency>
  <groupId>org.junit.vintage</groupId>
  <artifactId>junit-vintage-engine</artifactId>
  <version>6.0.1</version>
  <scope>test</scope>
</dependency>
```

> Vintage 엔진을 추가하면 JUnit 4 테스트를 Platform 상에서 그대로 실행할 수 있다. (병렬·필터링 등 Platform 기능 활용 가능)

### `junit-platform.properties` 예시(부분)

```properties
# 공통

junit.platform.output.capture.stderr=true
junit.platform.output.capture.stdout=true

# 병렬

junit.jupiter.execution.parallel.enabled=true
junit.jupiter.execution.parallel.mode.default=concurrent
junit.jupiter.execution.parallel.mode.classes.default=concurrent

# Vintage 엔진 제어(선택)

junit.vintage.execution.parallel.enabled=false
```

> 공식 속성 키는 사용자 가이드에 상세하게 나열되어 있다.

---

## 2025-11 최신 동향 요약

- **JUnit 6.0.1 GA(2025-10-31)**: JDK **17+** 요구. Jupiter/Platform 체계 유지, CLI/빌드/특성 개선. (기존 Jupiter 테스트 대부분 **그대로 동작**)
- 문서/가이드/설정 키는 **공식 User Guide**를 우선 참조. (5/6 모두 동일 섹션 구조)

---

## 결론

- **새 프로젝트**: **JUnit 6(Jupiter)** 채택을 권장 — 5의 API와 동일한 개발 경험이며, 최신 Platform 기능/성능/설정을 그대로 활용한다. (JDK 17+)
- **기존 JUnit 4**: **Vintage**로 공존 실행 → 점진적으로 Jupiter로 이전. `Runner/Rule` → **Extension/@TempDir/assertThrows** 로 치환.
- **핵심 이득**: 확장 모델/파라미터화/동적·중첩/조건/병렬/태그/런처 통합 — **표현력·속도·유연성** 모두 개선.
