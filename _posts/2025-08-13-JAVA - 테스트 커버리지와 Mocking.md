---
layout: post
title: Java - 테스트 커버리지와 Mocking
date: 2025-08-13 18:20:23 +0900
category: Java
---
# 테스트 커버리지와 Mocking

**테스트 커버리지(coverage)**는 “테스트가 코드의 어느 부분을 *실행했는가*”를 수치화한 지표다.
**Mocking**은 외부 협력 객체를 **테스트 더블(Test Double)** 로 대체해 **빠르고 결정적인 단위 테스트**를 가능하게 한다.
이 글은 **커버리지 종류 → JaCoCo 설정/리포트 해석 → 커버리지 품질을 올리는 전략 → 뮤테이션 테스트(PIT)** 와 **Mocking 원칙/Mockito 실전/안티패턴 → 팀 운영 규칙**까지 한 번에 정리한다. (모든 코드는 JUnit 5/Jupiter 문법 기준)

---

## 0. 2025-11 기준 빠른 버전 안내

실무 예시에서 사용하는 대표 도구의 *최신 안정* 라인(2025-11 기준):

| 항목 | 최신 버전(예시) | 근거 |
|---|---:|---|
| **JaCoCo Maven Plugin** | **0.8.14** | mvnrepository 최신 표기 확인.  |
| **Mockito Core** | **5.20.0** | mvnrepository 최신 릴리스.  |
| **PIT Maven Plugin** | **1.21.1** | Sonatype Central 메타데이터.  |
| **Gradle PIT Plugin** (`id "info.solidsoft.pitest"`) | **1.15.0** | Sonatype Central 메타데이터.  |

> 표의 버전은 “예시”다. 팀 표준 BOM/버전 카탈로그에 맞춰 일괄 관리하자.

---

## 1. 커버리지 지표 — 무엇을 ‘덮’었는가?

| 지표 | 설명 | 해석 팁 |
|---|---|---|
| **라인/명령(Instruction)** | 실행된 소스/바이트코드 비율 | 수치가 빠르게 올라가지만, 분기·예외 경로를 놓칠 수 있음 |
| **브랜치(Branch)** | `if/switch/?:` 각 갈래 실행 비율 | **조건/경계 버그**를 잡으려면 반드시 확인 |
| **조건(Condition)** | 복합 조건의 각 항목이 T/F를 모두 가졌는가 | 브랜치보다 더 세밀(도구 지원 제약 有) |
| **메서드/클래스** | 호출된 메서드/로딩된 클래스 비율 | 패키지별 커버리지 슬라이스 분석에 유용 |
| **경로(Path)** | 가능한 모든 경로 | 현실적으로 불가(조합 폭발) — 이론적 참고치 |
| **뮤테이션(Mutation)** | 의도적 변이(뮤턴트)를 테스트가 잡아내는 비율 | **테스트 ‘질’의 최강 지표**(빌드 비용↑) |

> **수치만으로 안심 금지.** 핵심 로직은 **브랜치 커버리지**와 **뮤테이션 테스트**로 본다.

---

## 2. JaCoCo로 커버리지 측정 — 설정 스니펫

### 2.1 Maven (기본 + 기준 미달 시 실패)

```xml
<!-- pom.xml -->
<build>
  <plugins>
    <plugin>
      <groupId>org.jacoco</groupId>
      <artifactId>jacoco-maven-plugin</artifactId>
      <version>0.8.14</version> <!-- 2025-11 최신 -->
      <executions>
        <!-- 테스트 시 에이전트 주입 -->
        <execution>
          <goals><goal>prepare-agent</goal></goals>
        </execution>

        <!-- 리포트 생성 (test 단계에서) -->
        <execution>
          <id>report</id>
          <phase>test</phase>
          <goals><goal>report</goal></goals>
        </execution>

        <!-- 커버리지 기준 체크 (verify 단계) -->
        <execution>
          <id>check</id>
          <phase>verify</phase>
          <goals><goal>check</goal></goals>
          <configuration>
            <rules>
              <rule>
                <element>PACKAGE</element>
                <limits>
                  <limit>
                    <counter>LINE</counter>
                    <value>COVEREDRATIO</value>
                    <minimum>0.80</minimum>
                  </limit>
                  <limit>
                    <counter>BRANCH</counter>
                    <value>COVEREDRATIO</value>
                    <minimum>0.70</minimum>
                  </limit>
                </limits>
              </rule>
            </rules>
            <excludes>
              <exclude>**/dto/**</exclude>
              <exclude>**/generated/**</exclude>
            </excludes>
          </configuration>
        </execution>
      </executions>
    </plugin>
  </plugins>
</build>
```

### 2.2 Gradle (Kotlin DSL)

```kotlin
// build.gradle.kts
plugins { jacoco }               // = id("jacoco")
tasks.test { useJUnitPlatform() }

jacoco { toolVersion = "0.8.14" } // 에이전트/리포터 버전

tasks.jacocoTestReport {
    dependsOn(tasks.test)
    reports {
        html.required.set(true)
        xml.required.set(true)
        csv.required.set(false)
    }
    // 예외 대상(생성/DTO/매핑 등)
    classDirectories.setFrom(
        files(classDirectories.files.map {
            fileTree(it) {
                exclude("**/dto/**", "**/generated/**")
            }
        })
    )
}

tasks.register<JacocoCoverageVerification>("jacocoVerify") {
    dependsOn(tasks.test)
    violationRules {
        rule {
            // 패키지 전체 기준
            limit { minimum = "0.80".toBigDecimal() } // 라인
        }
        rule {
            element = "CLASS"
            excludes = listOf("**/config/**", "**/*Application*")
            limit {
                counter = "BRANCH"
                value = "COVEREDRATIO"
                minimum = "0.70".toBigDecimal()
            }
        }
    }
}

// CI에서 실패 조건으로 연결
// tasks.check { dependsOn("jacocoTestReport", "jacocoVerify") }
```

### 2.3 리포트 읽는 법(핵심 포인트)
- **Missed Instructions/Branches**: 놓친 라인/분기를 먼저 본다.
- **복잡도(Cyclomatic Complexity)**가 높은 클래스는 브랜치 기준을 특히 강화.
- **변경 라인(diff coverage)** 중심 정책(예: “변경된 라인 90% 이상”)이 **전체 N%**보다 현실적.

수식으로 보는 복잡도(참고):
$$
M = E - N + 2P
$$
- \(M\): 사이클로매틱 복잡도, \(E\): 에지 수, \(N\): 노드 수, \(P\): 연결된 컴포넌트 수

---

## 3. 커버리지 ‘숫자’가 아니라 ‘위험’을 낮춰라 — 품질 중심 전략

1. **경계/예외 경로**: 정상 플로우만 덮지 말고 **빈 컬렉션/null/극단값/예외**를 반드시 포함.
2. **시간/난수/환경 분리**: `Clock`/`Random`/`Supplier<T>` 주입으로 **결정성** 확보.
3. **Fake 적극 활용**: 복잡한 Mock 연쇄보다 **인메모리 Fake**가 가독성·안정성 모두 우수.
4. **죽은 코드 제거**: 억지 테스트 대신 **미사용·중복 로직 삭제**가 더 큰 품질 개선.
5. **테스트 데이터 빌더**: 픽스처 중복을 줄여 **가독성**과 **유지보수성**을 확보.

---

## 4. 뮤테이션 테스트(PIT) — 테스트 ‘힘’ 점검

일반 커버리지는 “실행” 여부만 알려준다. **PIT**는 산술/조건 연산자 등을 변형해 **테스트가 실패를 일으키는지**로 *질*을 측정한다.

### 4.1 Maven 설정(핵심만)
```xml
<!-- pom.xml -->
<plugin>
  <groupId>org.pitest</groupId>
  <artifactId>pitest-maven</artifactId>
  <version>1.21.1</version> <!-- 2025-11 최신 -->
  <configuration>
    <targetClasses>
      <param>com.example.*</param>
    </targetClasses>
    <targetTests>
      <param>com.example.*Test</param>
    </targetTests>
    <mutationThreshold>75</mutationThreshold> <!-- 미달 시 실패 옵션 -->
    <threads>4</threads>
  </configuration>
</plugin>
```
(버전 출처: Sonatype Central)

### 4.2 Gradle 설정
```kotlin
// build.gradle.kts
plugins { id("info.solidsoft.pitest") version "1.15.0" } // 2025-11 기준
pitest {
    targetClasses.set(listOf("com.example.*"))
    targetTests.set(listOf("com.example.*Test"))
    mutationThreshold.set(75)
    junit5PluginVersion.set("1.2.3") // JUnit5 플러그인
}
// ./gradlew pitest
```
(Gradle 플러그인 최신: Central 메타데이터)

> **운영 팁**: 전체 모듈에 매번 돌리면 느리다. **핵심 모듈만 야간/주간 배치**로 실행하자.

---

## 5. Mocking — 원칙, 선택, 경계

### 5.1 Test Double 분류 요약

| 종류 | 목적 | 예 |
|---|---|---|
| **Dummy** | 채우기용, 사용 안 함 | 사용되지 않는 콜백/핸들러 |
| **Stub** | 고정 응답 제공 | 항상 동일 응답의 리포지토리 |
| **Fake** | 단순 대체 구현 | 인메모리 DB/캐시 |
| **Spy** | 실제 + 호출 기록 | 부분 모킹 |
| **Mock** | **행위 검증** | “`save`가 1회 호출되어야 한다” |

**원칙**: *Don’t mock what you don’t own*.
도메인/값 객체는 **실제 구현**을, **경계(네트워크/DB/파일/시간/난수)** 만 **Mock/Fake**로 치환하라.

### 5.2 언제 Mock/Fake인가?
- **단위 테스트**: DB/네트워크/파일 → **Mock/Fake** (속도·결정성↑)
- **통합 단계**: **Testcontainers/WireMock**으로 실제 흐름도 검증(여기서는 간단 언급만).

---

## 6. Mockito 실전 스니펫 (JUnit 5/Jupiter)

> 의존성(예시): `org.mockito:mockito-junit-jupiter:5.20.0` (Core最新版 릴리스 기준)

### 6.1 기본 Stubbing/Verify
```java
import static org.mockito.Mockito.*;
import static org.junit.jupiter.api.Assertions.*;
import org.junit.jupiter.api.Test;

interface Mailer { void send(String to, String body); }
interface Repo { String findName(long id); }

final class Service {
    private final Repo repo; private final Mailer mailer;
    Service(Repo r, Mailer m){ this.repo=r; this.mailer=m; }
    void notifyUser(long id){
        String name = repo.findName(id);
        if (name == null) throw new IllegalArgumentException("no user");
        mailer.send(name + "@ex.com", "hi " + name);
    }
}

class ServiceTest {
    @Test
    void notify_sendsMail() {
        Repo repo = mock(Repo.class);
        Mailer mail = mock(Mailer.class);
        when(repo.findName(1L)).thenReturn("alice");

        new Service(repo, mail).notifyUser(1L);

        verify(mail).send(eq("alice@ex.com"), startsWith("hi "));
        verifyNoMoreInteractions(mail);
    }
}
```

### 6.2 예외/void/Answer
```java
when(repo.findName(99L)).thenThrow(new NoSuchElementException());
doThrow(new RuntimeException()).when(mail).send(any(), any());
when(repo.findName(anyLong())).thenAnswer(inv -> "user-" + inv.getArgument(0, Long.class));
```

### 6.3 ArgumentCaptor로 값 검증
```java
import org.mockito.ArgumentCaptor;

ArgumentCaptor<String> to = ArgumentCaptor.forClass(String.class);
ArgumentCaptor<String> body = ArgumentCaptor.forClass(String.class);

verify(mail).send(to.capture(), body.capture());
assertEquals("bob@ex.com", to.getValue());
assertTrue(body.getValue().contains("bob"));
```

### 6.4 Spy(부분 모킹) — 안전 패턴
```java
List<String> real = new ArrayList<>();
List<String> spy = spy(real);

// when(spy.size())는 실제 호출되어 부작용 가능 → doReturn 사용
doReturn(42).when(spy).size();
spy.add("x");
assertEquals(42, spy.size());
```

### 6.5 정적 메서드 모킹 (Mockito 3.4+)
```java
import java.util.UUID;
import org.mockito.MockedStatic;
try (MockedStatic<UUID> mocked = mockStatic(UUID.class)) {
    mocked.when(UUID::randomUUID).thenReturn(UUID.fromString("00000000-0000-0000-0000-000000000000"));
    // 코드 내부의 UUID.randomUUID() → 고정 값 반환
}
```

---

## 7. 파일/시간/난수/동시성 — 결정성 확보 패턴

### 7.1 파일 — `@TempDir` (Jupiter)
```java
import org.junit.jupiter.api.io.TempDir;
import java.nio.file.*;
import static org.junit.jupiter.api.Assertions.*;

class FileTest {
    @TempDir Path tempDir;
    @Test
    void writesFile() throws Exception {
        Path out = tempDir.resolve("out.txt");
        Files.writeString(out, "hello");
        assertTrue(Files.exists(out));
    }
}
```

### 7.2 시간 — `Clock` 주입
- 프로덕션: `LocalDate.now(clock)`
- 테스트: `Clock.fixed(Instant.parse("2025-01-01T00:00:00Z"), ZoneOffset.UTC)`

### 7.3 난수 — `Random`/UUID 주입 또는 정적 모킹

### 7.4 동시성 — 타이밍 의존 제거
```java
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;
import static org.junit.jupiter.api.Assertions.*;

final class Counter {
    private final AtomicInteger v = new AtomicInteger();
    int inc(){ return v.incrementAndGet(); }
    int get(){ return v.get(); }
}

@Test
void inc_threadSafe() throws Exception {
    Counter c = new Counter();
    int threads = 8, iters = 5_000;
    ExecutorService es = Executors.newFixedThreadPool(threads);
    CountDownLatch start = new CountDownLatch(1), done = new CountDownLatch(threads);
    for (int t=0; t<threads; t++) {
        es.submit(() -> { start.await(); for (int i=0;i<iters;i++) c.inc(); done.countDown(); return null; });
    }
    start.countDown(); done.await(); es.shutdown();
    assertEquals(threads * iters, c.get());
}
```

> **금지**: `Thread.sleep(...)`으로 타이밍을 “맞추는” 테스트 → **플레이키(flaky)** 의 주범.

---

## 8. Mocking 안티패턴 — 이렇게 쓰지 말자

- **오버모킹**: 값/도메인 객체까지 전부 Mock → 설계 변경에 **취약**.
- **네거티브 검증 남발**: `verifyNoMoreInteractions`를 광범위하게 → 사소한 변경에도 깨짐.
- **정적/싱글톤 남발**: 주입이 어려워 **테스트불가 코드** 양산. 래핑/DI로 구조를 바꿔라.
- **`sleep()` 의존**: 동시성·비동기 타이밍은 `Latch/Barrier/Answer`로 제어.
- **`reset(mock)` 남발**: 테스트 격리 원칙(각 테스트에 *새* Mock)으로 해결.

---

## 9. 커버리지+Mocking을 팀에서 ‘운영’하는 법

**정책 제안(예시)**
- 변경 라인(**diff coverage**) **라인 ≥ 90%**, **브랜치 ≥ 80%**.
- 핵심 도메인/알고리즘: **뮤테이션 ≥ 70%**.
- DTO/생성 코드는 제외 패턴 유지.

**실행/분리**
- **단위 테스트**: Mock/Fake로 빠르게(초단위).
- **통합 테스트**: Testcontainers/WireMock는 태그 분리, 파이프라인 후반에 실행.

**리포트/가시성**
- JaCoCo HTML/XML과 PIT 리포트는 CI 아티팩트 보관, PR 코멘트에 *변경 커버리지* 요약.

---

## 10. 종합 예제 — 서비스 + 리포지토리 + 메일러

### 10.1 대상 코드
```java
package com.example;

import java.time.*;

public final class User {
    private final long id;
    private boolean active;
    private LocalDate activatedAt;
    public User(long id){ this.id=id; }
    public long getId(){ return id; }
    public boolean isActive(){ return active; }
    public LocalDate getActivatedAt(){ return activatedAt; }
    public void activate(LocalDate d){ this.active=true; this.activatedAt=d; }
}

interface UserRepository {
    java.util.Optional<User> findById(long id);
    void save(User user);
}

interface Mailer { void send(String to, String body); }

public final class UserService {
    private final UserRepository repo; private final Mailer mail; private final Clock clock;
    public UserService(UserRepository r, Mailer m, Clock c){ this.repo=r; this.mail=m; this.clock=c; }

    public void activate(long id){
        User u = repo.findById(id).orElseThrow();
        u.activate(LocalDate.now(clock));
        repo.save(u);
        mail.send("user-"+u.getId()+"@ex.com", "activated:"+u.getActivatedAt());
    }
}
```

### 10.2 단위 테스트 (Mockito + 결정성 확보)
```java
package com.example;

import org.junit.jupiter.api.*;
import org.mockito.*;
import java.time.*;
import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

class UserServiceTest {
    @Mock UserRepository repo;
    @Mock Mailer mail;
    Clock fixed;
    UserService sut;

    @BeforeEach
    void setUp() {
        MockitoAnnotations.openMocks(this);
        fixed = Clock.fixed(Instant.parse("2025-01-01T00:00:00Z"), ZoneOffset.UTC);
        sut = new UserService(repo, mail, fixed);
    }

    @Test
    void activate_setsDate_saves_sendsMail() {
        User user = new User(1L);
        when(repo.findById(1L)).thenReturn(java.util.Optional.of(user));

        sut.activate(1L);

        assertTrue(user.isActive());
        assertEquals(LocalDate.of(2025,1,1), user.getActivatedAt());
        verify(repo).save(user);

        ArgumentCaptor<String> to = ArgumentCaptor.forClass(String.class);
        ArgumentCaptor<String> body = ArgumentCaptor.forClass(String.class);
        verify(mail).send(to.capture(), body.capture());
        assertEquals("user-1@ex.com", to.getValue());
        assertTrue(body.getValue().contains("2025-01-01"));
        verifyNoMoreInteractions(mail, repo);
    }

    @Test
    void activate_throws_whenNotFound() {
        when(repo.findById(99L)).thenReturn(java.util.Optional.empty());
        assertThrows(java.util.NoSuchElementException.class, () -> sut.activate(99L));
        verifyNoInteractions(mail);
    }
}
```

이 테스트 묶음만으로도:
- **라인/브랜치** 대부분을 덮고,
- “미존재 사용자” 예외 경로를 포함하며,
- 시간 의존은 **`Clock.fixed`** 로 결정화했다.

---

## 11. 체크리스트

- [ ] 브랜치/예외/경계값이 포함되어 있는가?
- [ ] 변경 라인(디프)의 커버리지를 우선 보나?
- [ ] 외부 자원(네트워크/DB/파일/시간/난수)을 Mock/Fake/주입으로 대체했나?
- [ ] Mock은 **행위 검증이 필요한 협력자**에만 제한했나?
- [ ] 테스트는 **빠르고 결정적**, 서로 **독립적**인가?
- [ ] JaCoCo/PIT 리포트를 **주기적으로** 점검하나?

---

## 12. 결론

- 커버리지는 **양**(어디까지 실행했나), **뮤테이션은 질**(버그를 잡아내는 힘)을 보여준다.
- Mocking은 **경계 의존성**을 대체해 **빠르고 신뢰 가능한** 단위 테스트를 만든다.
- 좋은 팀 규칙은 “**변경 커버리지 높은 상태** + **핵심 모듈 뮤테이션 테스트** + **플레이키 제거**”로 요약된다.
