---
layout: post
title: 디자인패턴 - Singleton vs DI 컨테이너 비교
date: 2025-07-05 20:20:23 +0900
category: 디자인패턴
---
# Singleton vs DI 컨테이너

## 1. 개요와 용어

| 용어 | 설명 |
|---|---|
| Singleton | 특정 클래스의 인스턴스를 애플리케이션 전체에서 1개로 **강제**하는 패턴 |
| DI (Dependency Injection) | 객체가 자신의 의존을 **외부에서 주입**받는 설계 원리 |
| DI 컨테이너 | 의존성 그래프와 수명주기를 **설정 기반**으로 관리·해결하는 런타임 도구(예: Spring, .NET) |

핵심 차이:
- **Singleton**은 “**하나만** 만들겠다”는 **객체 내부의 제약**.
- **DI 컨테이너**는 “**언제·얼마나** 만들고 **어떻게 주입**할지”를 **외부 구성(Composition Root)** 으로 이관.

---

## 2. Singleton 심화: 구현과 함정

### 2.1 대표 구현

#### Java — 올바른 DCL(Double-Checked Locking)
```java
public final class Config {
  private static volatile Config INSTANCE;
  private Config() {}
  public static Config getInstance() {
    Config local = INSTANCE;
    if (local == null) {
      synchronized (Config.class) {
        local = INSTANCE;
        if (local == null) {
          local = INSTANCE = new Config();
        }
      }
    }
    return local;
  }
}
```
- `volatile` 누락 시 재정렬로 인한 반쯤 초기화된 인스턴스 노출 위험.

#### C# — 정적 초기화(스레드 안전)
```csharp
public sealed class AppConfig {
    private AppConfig() {}
    public static AppConfig Instance { get; } = new AppConfig(); // CLR이 안전 보장
}
```

#### C++ — Meyers Singleton (C++11+)
```cpp
class Config {
public:
  static Config& Instance() {
    static Config instance; // thread-safe
    return instance;
  }
private:
  Config() = default;
};
```

#### Kotlin — 언어 차원의 싱글턴
```kotlin
object Config { var mode: String = "prod" }
```

#### Python — 모듈 단위(사실상 싱글턴)
```python
# config.py
value = "prod"  # import한 모듈은 프로세스 내에서 1회 로딩
```

### 2.2 장점과 비용
- 장점: 간단·빠름·의존성 구성 없이 즉시 사용.
- 비용: 전역 상태, **테스트 곤란(모킹·격리 어려움)**, **수명 역전(captive dependency)**, 동시성/초기화 순서 문제.

### 2.3 전형적 함정
- **가변 전역 상태**: 테스트 간 교차 오염.
- **숨은 의존**: import 또는 `getInstance()` 호출이 실제 의존을 감춘다.
- **초기화 순서**: 정적 초기화 순환 의존.
- **멀티테넌시 곤란**: 요청·세션별 구성이 불가.

---

## 3. DI 컨테이너 심화: 수명주기·조립·고급 기능

### 3.1 .NET 예제 — 기본 수명주기
```csharp
// Program.cs (Composition Root)
builder.Services.AddSingleton<IClock, UtcClock>();      // 프로세스 내 1개
builder.Services.AddScoped<IUnitOfWork, EfUnitOfWork>(); // 요청(스코프)당 1개
builder.Services.AddTransient<IHasher, Sha256Hasher>();  // 해결(resolve)마다 1개
```

```csharp
public class ReportService {
    private readonly IClock _clock;          // Singleton
    private readonly IUnitOfWork _uow;       // Scoped
    private readonly IHasher _hasher;        // Transient
    public ReportService(IClock clock, IUnitOfWork uow, IHasher hasher) {
        _clock = clock; _uow = uow; _hasher = hasher;
    }
    public Task GenerateAsync() { /* ... */ return Task.CompletedTask; }
}
```

### 3.2 Spring Boot 예제 — 스코프와 주입
```java
@Configuration
public class AppConfig {
  @Bean @Scope(ConfigurableBeanFactory.SCOPE_SINGLETON)
  public Clock clock() { return Clock.systemUTC(); }

  @Bean @Scope(WebApplicationContext.SCOPE_REQUEST)
  public RequestContext requestContext() { return new RequestContext(); }

  @Bean
  public ReportService reportService(Clock clock, RequestContext rc, Hasher hasher) {
    return new ReportService(clock, rc, hasher);
  }
}
```

### 3.3 Python 예제 — dependency_injector
```python
from dependency_injector import containers, providers

class Container(containers.DeclarativeContainer):
    config = providers.Configuration()
    clock = providers.Singleton(lambda: "utc-clock")
    hasher = providers.Factory(lambda: "sha256")
    report_service = providers.Factory(lambda c, h: (c, h), clock, hasher)
```

### 3.4 고급 기능
- **Open Generic** 등록, **Decorator**/Interceptor(로깅·벨리데이션), **Typed/Named HttpClient**, **Options 패턴**, **Factory 주입**(지연 생성).

---

## 4. 수명주기 제약을 수식으로 표현하기

의존 그래프를 유향 그래프로 두자.
- 정점: 컴포넌트 집합 \( V \).
- 간선: 의존 \( E \subseteq V \times V \) (u→v는 u가 v에 의존).
- 수명주기 함수 \( L: V \to \{\text{Singleton}, \text{Scoped}, \text{Transient}\} \).
- 부분순서 \( \preceq \) 를 다음과 같이 정의:
  \[
  \text{Transient} \preceq \text{Scoped} \preceq \text{Singleton}
  \]
안전 조건:
\[
(u \to v) \in E \;\Rightarrow\; L(u) \preceq L(v)
\]
즉 **더 오래 사는 객체가 더 짧게 사는 객체에 의존하면 위험**(captured dependency).
Singleton이 Transient에 직접 의존하면, Transient가 사실상 Singleton처럼 살아버려 리소스/상태가 갇힌다.

---

## 5. 정밀 비교 표

| 항목 | Singleton 패턴 | DI 컨테이너 |
|---|---|---|
| 수명주기 | 전역 1개(고정) | Singleton/Scoped/Transient 등 **유연 설정** |
| 의존 표현 | 코드 내부에서 `getInstance()` | **생성자 주입**(명시적) |
| 테스트 | 모킹 어렵고 전역 상태 오염 | 인터페이스 치환 쉬움, 테스트 친화 |
| 스레드 안전 | 직접 보장 필요 | 컨테이너/프레임워크가 지원(전제: 구현 주의) |
| 교체 가능성 | 낮음(강결합) | 높음(구성만 교체) |
| 다중 테넌트/요청별 상태 | 곤란 | Scoped로 자연스러운 격리 |
| 학습/준비 | 매우 쉬움 | 설정·조립 필요(학습 비용) |
| 안티패턴 위험 | 전역 상태, Service Locator로 변질 | 과도 DI, 컨테이너 침투(Service Locator) |

---

## 6. 성능과 동시성 관점

- **해결(Resolve) 비용**: 현대 컨테이너는 1~수십 마이크로초 수준(구성·범위·Open Generic·Decorator 사용에 따라 달라짐).
- **싱글턴의 잠금 경합**: 전역 가변 상태·공유 락을 두면 부하 시 병목. **불변/락-프리** 구조 또는 샤딩을 고려.
- **스코프 관리**: 연결·파일 핸들 같은 리소스는 Scoped/Transient로 생성하고 **명확한 종료**를 보장.

간단 마이크로벤치(개념 예시, C#):
```csharp
var sw = Stopwatch.StartNew();
for (int i = 0; i < 1_000_000; i++) {
    using var scope = provider.CreateScope();
    _ = scope.ServiceProvider.GetRequiredService<IHasher>();
}
sw.Stop();
Console.WriteLine(sw.Elapsed);
```
의미: 트래픽·구성에 맞춰 **DI 해상도 비용**을 관찰하고 Hot Path는 **Factory 주입** 등으로 최적화.

---

## 7. 안티패턴과 예방책

### 7.1 Service Locator (지양)
```csharp
public class Bad {
  private readonly IServiceProvider _sp;
  public Bad(IServiceProvider sp) { _sp = sp; }
  public void Do() {
    var repo = _sp.GetService<IRepo>(); // 런타임 의존 숨김
  }
}
```
- 문제: 의존이 코드에 **명시되지 않음**, 테스트·검토 어려움.
- 해결: **생성자 주입**으로 명시하거나, 정말 동적 필요 시 **Abstract Factory**를 명시 의존으로 주입.

### 7.2 Singleton이 Transient 잡아두기(captive)
```csharp
public class BadSingleton {
  public BadSingleton(TransientRepo repo) { /* repo가 사실상 싱글턴처럼 */ }
}
```
- 해결: 수명 규칙 \( L(u) \preceq L(v) \) 준수. 필요하면 **ScopeFactory**로 한시적 범위 생성.
```csharp
public class ReportService {
  private readonly IServiceScopeFactory _sf;
  public ReportService(IServiceScopeFactory sf) { _sf = sf; }
  public void Generate() {
    using var scope = _sf.CreateScope();
    var repo = scope.ServiceProvider.GetRequiredService<IRepo>();
    // ...
  }
}
```

### 7.3 전역 가변 상태
- 해결: 불변(Immutable) 데이터, Copy-on-write, 스레드 안전 컬렉션, 명시적 동기화 최소화.

---

## 8. 사용 기준(의사결정 매트릭스)

| 상황 | 권장 |
|---|---|
| 단순 도구(무상태, 순수 함수에 가까움) | Singleton 또는 정적 유틸 |
| 요청/세션/트랜잭션 경계가 중요 | DI 컨테이너 + Scoped |
| 다양한 구현 교체·A/B·테스트 필수 | DI 컨테이너(인터페이스 주입) |
| 멀티테넌시·플러그인 구조 | DI 컨테이너(구성 격리) |
| 고성능 Hot Path에서 해석 비용 부담 | Factory 주입, Pre-wire, 수동 DI 일부 혼용 |

---

## 9. 실무 시나리오 베스트 프랙티스

### 9.1 로깅
- .NET: `ILogger<T>`는 컨테이너 Singleton 팩토리가 제공, 각 T별 카테고리 부여.
- Spring: `@Slf4j` 또는 `LoggerFactory` 주입.
- 전역 로거 싱글턴보다 **DI 주입 로거**가 교체·테스트 용이.

### 9.2 구성(Configuration)
- .NET: `IOptions<T>`(Singleton 소스) + Scoped/Transient 소비자.
- 변경 감지 필요 시 `IOptionsMonitor<T>`.

### 9.3 DB 컨텍스트
- EF DbContext는 **Scoped**가 권장(요청·트랜잭션 경계).
- 싱글턴으로 두면 동시성·캐시 오염.

### 9.4 HttpClient
- .NET: `IHttpClientFactory` 사용(소켓 재사용, DNS 회전). Singleton 직접 보관은 누수 위험.

---

## 10. 테스트 전략

### 10.1 Singleton의 한계
```python
class Logger:
    _instance = None
    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
            cls._instance.level = "INFO"
        return cls._instance

# 테스트간 전역 상태 공유 → 독립성 저하
```

### 10.2 DI로 치환(파이썬 예)
```python
class Logger:
    def __init__(self, level="INFO"): self.level = level

class Service:
    def __init__(self, logger: Logger): self.logger = logger

# 테스트
fake = Logger(level="DEBUG")
sut = Service(logger=fake)  # 손쉬운 치환
```

### 10.3 .NET 테스트 구성
```csharp
var services = new ServiceCollection();
services.AddSingleton<IClock, FakeClock>();
services.AddTransient<IHasher, FakeHasher>();
var provider = services.BuildServiceProvider();
var sut = provider.GetRequiredService<ReportService>();
```

---

## 11. 마이그레이션: Singleton → DI

1) **의존 노출**: `getInstance()` 호출부를 생성자 인자로 치환.
2) **인터페이스 추출**: 구현에 인터페이스 도입(IClock 등).
3) **Composition Root 신설**: 등록 코드로 전역 의존을 외부로 이동.
4) **테스트 정비**: Fake/Mock 주입으로 격리 테스트.
5) **수명 재설계**: 그래프 상 수명 제약 \( L(u) \preceq L(v) \) 검증.
6) **점진 전환**: 경계 어댑터를 두고 모듈별로 대체.

예시(전환 전)
```csharp
public class ReportService {
  public void Run() {
    var log = Logger.Instance; // 전역 호출
    log.Info("start");
  }
}
```
전환 후
```csharp
public class ReportService {
  private readonly ILogger _log;
  public ReportService(ILogger log) { _log = log; }
  public void Run() { _log.Info("start"); }
}
```

---

## 12. 보안·운영 관점

- **비밀/토큰**: 컨테이너 등록 시 지연 조회(Secret Manager, Vault). 싱글턴에 평문 보관 금지.
- **순환 의존**: 컨테이너 경고를 해결(포트/어댑터로 분리).
- **관찰 가능성**: Interceptor/Decorator로 로깅·메트릭 주입(핸들러 파이프라인).
- **Warmup**: 고비용 싱글턴은 스타트업 시 프리워밍하거나 Lazy<T>로 지연.

---

## 13. 코드 레시피 모음

### 13.1 .NET: Decorator로 교차 관심사 추가
```csharp
public interface IHasher { string Hash(string s); }

public sealed class Sha256Hasher : IHasher {
  public string Hash(string s) => Convert.ToHexString(
      System.Security.Cryptography.SHA256.HashData(System.Text.Encoding.UTF8.GetBytes(s)));
}

public sealed class TimingHasher : IHasher {
  private readonly IHasher _inner; private readonly ILogger<TimingHasher> _log;
  public TimingHasher(IHasher inner, ILogger<TimingHasher> log) { _inner = inner; _log = log; }
  public string Hash(string s) {
    var sw = Stopwatch.StartNew();
    var h = _inner.Hash(s);
    _log.LogInformation("hash in {Elapsed} ms", sw.Elapsed.TotalMilliseconds);
    return h;
  }
}

// 등록
services.AddTransient<IHasher, Sha256Hasher>();
services.Decorate<IHasher, TimingHasher>(); // Scrutor 등 사용
```

### 13.2 Java: Provider/Factory로 지연 생성
```java
class ReportService {
  private final Provider<Repository> repoProvider; // javax.inject.Provider
  ReportService(Provider<Repository> repoProvider) { this.repoProvider = repoProvider; }
  void run() {
    Repository repo = repoProvider.get(); // 필요 시점에 생성
  }
}
```

---

## 14. 다이어그램(ASCII)

### 14.1 Singleton vs DI 흐름
```
[Caller] --> getInstance() ---------> [Singleton Instance]
[Caller] --ctor(args from container)-> [Object built by DI]
                         ^ register
                   [Composition Root]
```

### 14.2 수명 제약 위반 사례
```
[Singleton S] ---> [Transient T]   // T가 사실상 전역으로 붙잡힘(captive)
```

---

## 15. 결론

- **Singleton**: 간단·저비용이지만 전역 상태·테스트·수명 역전의 구조적 리스크가 크다.
- **DI 컨테이너**: 수명주기·의존을 **외부 구성으로 명시**하고, 테스트·교체·확장을 체계화한다.
- 규모가 커질수록 다음 원칙을 따르는 것이 장기적으로 유리하다.
  1) **생성은 밖에서(Composition Root)**, **사용은 안에서(의존 주입)**.
  2) 수명 제약 \( L(u) \preceq L(v) \) 준수, 전역 가변 상태 금지.
  3) 안티패턴(Service Locator, Captive Dependency) 차단.
  4) 교차 관심사는 Decorator/Interceptor로 횡단 주입.

단순 도구성 컴포넌트에는 Singleton도 실용적일 수 있다.
그러나 **변화·테스트·멀티테넌시·운영성**이 중요한 일반적 애플리케이션에서는 **DI 컨테이너 기반 수명주기 관리**가 훨씬 일관되고 미래지향적이다.
