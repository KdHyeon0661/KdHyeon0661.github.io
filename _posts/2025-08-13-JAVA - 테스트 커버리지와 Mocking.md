---
layout: post
title: Java - 테스트 커버리지와 Mocking
date: 2025-08-13 18:20:23 +0900
category: Java
---
# 테스트 커버리지와 Mocking — 실전 가이드

**테스트 커버리지(coverage)**는 “테스트가 코드의 어느 부분을 실행했는가”를 수치화한 지표입니다.  
**Mocking**은 외부 협력 객체를 **대체(Double)**하여 **빠르고 결정적인(unit) 테스트**를 가능하게 하는 기법입니다.  
아래에서 커버리지 **종류·도구·설정·해석법**과 Mocking **원칙·프레임워크 활용·안티패턴**을 함께 정리합니다.

---

## 1) 커버리지 기본 개념과 지표

- **라인/명령 커버리지(Line/Instruction)**: 실행된 소스/바이트코드 비율.
- **브랜치 커버리지(Branch)**: 분기(if/switch/삼항)의 각 갈래를 실행했는가.
- **조건 커버리지(Condition)**: 복합 조건식의 각 조건이 참/거짓을 모두 가졌는가.
- **메서드/클래스 커버리지**: 호출된 메서드/로딩된 클래스 비율.
- **경로 커버리지(Path)**: 가능한 모든 경로(조합)를 실행 — 실무에선 비현실적.
- **뮤테이션 커버리지(Mutation)**: 코드를 의도적으로 변형(뮤턴트)했을 때 **테스트가 실패로 잡아내는 비율** → **테스트의 질**을 가장 잘 반영.

> **주의**: 높은 라인 커버리지는 “안전”을 보장하지 않습니다.  
> 분기·경계·예외 경로가 검증되지 않으면 **숫자는 높아도 품질은 낮을 수** 있습니다.  
> 핵심 로직은 **브랜치 커버리지**와 **뮤테이션 테스트**까지 보는 것을 권장합니다.

---

## 2) JaCoCo로 커버리지 측정

### 2.1 Maven 설정
```xml
<build>
  <plugins>
    <plugin>
      <groupId>org.jacoco</groupId>
      <artifactId>jacoco-maven-plugin</artifactId>
      <version>0.8.11</version>
      <executions>
        <!-- 에이전트 주입 -->
        <execution>
          <goals><goal>prepare-agent</goal></goals>
        </execution>
        <!-- 리포트 생성 (HTML/XML) -->
        <execution>
          <id>report</id>
          <phase>test</phase>
          <goals><goal>report</goal></goals>
        </execution>
        <!-- 커버리지 기준 미달 시 빌드 실패 -->
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

### 2.2 Gradle(Kotlin DSL) 설정
```kotlin
plugins { jacoco }          // plugins { id("jacoco") } 도 동일
tasks.test { useJUnitPlatform() }

jacoco { toolVersion = "0.8.11" }

tasks.jacocoTestReport {
    dependsOn(tasks.test)
    reports {
        html.required.set(true)
        xml.required.set(true)
        csv.required.set(false)
    }
    // 생성/DTO 등 제외
    classDirectories.setFrom(
        files(classDirectories.files.map {
            fileTree(it) {
                exclude("**/dto/**", "**/generated/**")
            }
        })
    )
}

tasks.register<JacocoCoverageVerification>("jacocoTestCoverageVerification") {
    dependsOn(tasks.test)
    violationRules {
        rule {
            limit { minimum = "0.80".toBigDecimal() } // 라인 커버리지
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
// CI에서: check.dependsOn("jacocoTestReport", "jacocoTestCoverageVerification")
```

### 2.3 리포트 읽는 법 (핵심 포인트)
- **Missed Instructions/Branches**: 놓친 라인/분기 수를 우선 확인.
- **복잡도(Cyclomatic Complexity)**가 높은 클래스는 브랜치 커버리지 기준으로 관리.
- **새 코드(diff coverage)** 위주 정책: 전체 80%보다 **변경 라인 90%** 같은 정책이 더 현실적.

---

## 3) 커버리지 올리는 전략 (품질 중심)

1. **경계값/예외 경로 테스트**: 정상 흐름만이 아니라 **실패/에러/빈 컬렉션/null/극단값**.
2. **설계 개선으로 테스트 용이성 확보**:  
   - 시간/난수/환경 의존성 → **`Clock`/`Random` 주입**  
   - 정적 호출/싱글톤/전역 상태 → **인터페이스로 래핑**해 교체 가능하게
3. **Fake 활용**: 복잡한 Mock 다발보다 **인메모리 구현(Fake)**가 더 읽기 쉬움.
4. **죽은 코드 제거**: 커버리지 때문에 테스트를 ‘억지로’ 만들지 말고 **불필요 코드는 삭제**.
5. **테스트 데이터 빌더**로 가독성↑ 중복↓.

---

## 4) 뮤테이션 테스트(PIT)로 “테스트의 힘” 점검

일반 커버리지는 **실행 여부**만 보며, **오류 탐지 능력**은 보장하지 않습니다.  
**PIT(PITest)**는 연산자 뒤바꾸기 등 작은 변이를 만들어 **테스트가 실패로 잡는지** 측정합니다.

### Maven 예시
```xml
<plugin>
  <groupId>org.pitest</groupId>
  <artifactId>pitest-maven</artifactId>
  <version>1.15.10</version>
  <configuration>
    <targetClasses>
      <param>com.example.*</param>
    </targetClasses>
    <targetTests>
      <param>com.example.*Test</param>
    </targetTests>
    <mutationThreshold>75</mutationThreshold>
  </configuration>
</plugin>
```

### Gradle 예시
```kotlin
plugins { id("info.solidsoft.pitest") version "1.15.0" }

pitest {
    targetClasses.set(listOf("com.example.*"))
    targetTests.set(listOf("com.example.*Test"))
    mutationThreshold.set(75)
    junit5PluginVersion.set("1.2.1")
}
```

> **권장**: 핵심 모듈만 정기적으로 뮤테이션 테스트(야간/주간) → 과도한 빌드 시간 방지.

---

## 5) Mocking — 원칙과 선택

### 5.1 Test Double 분류
| 종류 | 목적 | 예 |
|---|---|---|
| **Dummy** | 채우기용, 사용 안 함 | unused 콜백 |
| **Stub** | 미리 정해진 응답 | 고정 리포지토리 |
| **Fake** | 단순 대체 구현 | 인메모리 DB |
| **Spy** | 실제 + 호출 기록 | 부분 모킹 |
| **Mock** | **행위 검증** 중심 | “save가 1번 호출되어야 한다” |

> **원칙**: _Don’t mock what you don’t own_ (자신이 소유하지 않은 외부 타입을 과도하게 모킹하면 취약).  
> 도메인/값 객체는 모킹보다 **실제 객체**를 쓰고, **경계(네트워크/DB/시간/난수/파일)**만 대체하세요.

### 5.2 언제 Mock/Fake를 쓰나
- **DB/네트워크/파일**: 단위 테스트에서는 Mock/Fake로 대체(속도·결정성↑).
- **시간/난수**: `Clock`/`Random` 주입(혹은 UUID 정적 메서드 모킹).
- **외부 API**: 단위 테스트는 Mock, 통합 단계에서 **WireMock/Testcontainers**로 실제 흐름 검증.

---

## 6) Mockito 실전

### 6.1 기본 Stubbing/Verify
```java
import static org.mockito.Mockito.*;
import static org.junit.jupiter.api.Assertions.*;

interface Mailer { void send(String to, String body); }
interface Repo { String findName(long id); }

final class Service {
    private final Repo repo; private final Mailer mailer;
    Service(Repo r, Mailer m){ this.repo=r; this.mailer=m; }
    void notifyUser(long id){
        String name = repo.findName(id);
        if (name == null) throw new IllegalArgumentException();
        mailer.send(name + "@ex.com", "hi " + name);
    }
}

@Test
void notify_sendsMail() {
    Repo repo = mock(Repo.class);
    Mailer mail = mock(Mailer.class);
    when(repo.findName(1L)).thenReturn("alice");

    new Service(repo, mail).notifyUser(1L);

    verify(mail, times(1)).send(eq("alice@ex.com"), startsWith("hi "));
    verifyNoMoreInteractions(mail);
}
```

### 6.2 예외/void 메서드/Answer
```java
// 예외 던지기
when(repo.findName(99L)).thenThrow(new NoSuchElementException());

// void 메서드 예외
doThrow(new RuntimeException()).when(mail).send(any(), any());

// 동적 동작(Answer)
when(repo.findName(anyLong())).thenAnswer(inv ->
    "user-" + inv.getArgument(0, Long.class));
```

### 6.3 ArgumentCaptor로 저장값 검증
```java
import org.mockito.ArgumentCaptor;
import static org.mockito.Mockito.*;
import static org.junit.jupiter.api.Assertions.*;

@Captor ArgumentCaptor<String> to;
@Captor ArgumentCaptor<String> body;

@Test
void capture_args(@Mock Repo repo, @Mock Mailer mail) {
    when(repo.findName(1L)).thenReturn("bob");
    new Service(repo, mail).notifyUser(1L);

    verify(mail).send(to.capture(), body.capture());
    assertEquals("bob@ex.com", to.getValue());
    assertTrue(body.getValue().contains("bob"));
}
```

### 6.4 Spy(부분 모킹) — 주의해서 사용
```java
List<String> real = new ArrayList<>();
List<String> spy = spy(real);

// spy는 기본적으로 실제 메서드 호출
doReturn(42).when(spy).size();     // when(spy.size())는 실제 호출되어 NPE 등 날 수 있음
spy.add("x");

assertEquals(42, spy.size());
```

### 6.5 정적 메서드 모킹(Mockito 3.4+)
```java
import java.util.UUID;
import org.mockito.MockedStatic;

try (MockedStatic<UUID> mocked = Mockito.mockStatic(UUID.class)) {
    mocked.when(UUID::randomUUID).thenReturn(UUID.fromString("00000000-0000-0000-0000-000000000000"));
    // 코드 내부에서 UUID.randomUUID() 호출 → 고정 값 반환
}
```

> **베스트 프랙티스**
> - 테스트마다 **새 Mock**을 생성 (`@ExtendWith(MockitoExtension.class)` + `@Mock`).
> - **하나의 행위**만 검증, 과도한 `verifyNoMoreInteractions` 남용 금지.
> - **구현 세부 호출 순서**에 결합된 검증은 취약 — 상태/결과를 우선 검증.

---

## 7) Mocking 안티패턴

- **오버모킹**: 값/도메인 객체까지 전부 Mock → 리팩터링에 한 줄만 바꿔도 대량 실패.
- **네거티브 검증 남발**: `verifyNoMoreInteractions`를 광범위하게 사용해 **취약**해짐.
- **Thread.sleep 의존**: 동시성/비동기 테스트에서 타이밍 의존 → **플레이키**.  
  → `CountDownLatch`/`Awaitility`/콜백을 **Answer**로 트리거.
- **Reset 모킹**: `reset(mock)` 남발보다 **테스트 격리**(새 인스턴스) 원칙 준수.
- **정적/싱글톤 의존** 방치: 모킹 어려움. **의존성 주입**/랩핑으로 설계를 재고.

---

## 8) 커버리지+Mocking 함께 운영하는 법 (팀 규칙 제안)

- **정책**
  - 변경 라인(신규 코드) **라인 ≥ 90%**, **브랜치 ≥ 80%** 목표.
  - 핵심 도메인/알고리즘은 **뮤테이션 커버리지 ≥ 70%**.
- **분리 실행**
  - **단위 테스트**(Mock/Fake) 빠르게,  
  - **통합 테스트**(Testcontainers/WireMock 등)는 별 태그로 분리하여 CI 후반에 실행.
- **리포트 배포**
  - CI에서 JaCoCo HTML/XML를 아티팩트로 보관, PR에 요약 코멘트(변경 커버리지) 표시.
- **예외 규정**
  - 생성·DTO·보일러플레이트는 제외 패턴으로 관리.
  - 바로 테스트 어려운 레거시는 리팩터링하며 “**커버리지 개선 부채 목록**”로 관리.

---

## 9) 체크리스트

- [ ] 브랜치/예외/경계값 테스트가 포함되어 있는가?  
- [ ] 변경 라인(diff)의 커버리지를 우선 보고 있는가?  
- [ ] 외부 의존(시간/네트워크/DB/파일)을 **Mock/Fake**로 대체했는가?  
- [ ] Mock은 **행위 검증이 필요한 협력자**에만 제한했는가?  
- [ ] 테스트는 **빠르고 결정적**이며 서로 **독립적**인가?  
- [ ] JaCoCo/PIT 리포트를 **정기적으로** 확인하는가?  

---

## 10) 결론

- 커버리지는 **테스트의 양**을, 뮤테이션은 **테스트의 질**을 보여줍니다.  
- Mocking은 **경계**를 대체해 **빠르고 신뢰 가능한** 단위 테스트를 가능하게 합니다.  
- 숫자 자체보다 **위험도가 높은 코드의 시나리오**를 얼마나 잘 잡아내는지가 중요합니다.