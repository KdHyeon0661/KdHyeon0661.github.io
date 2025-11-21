---
layout: post
title: 디자인패턴 - Decorator
date: 2025-06-20 21:20:23 +0900
category: 디자인패턴
---
# Decorator(데코레이터 패턴)

## 정의

**데코레이터 패턴(Decorator Pattern)**은 객체에 **추가 기능을 동적으로 부여**하는 **구조 패턴**이다.
상속 없이 **구성(Composition)**으로 기능을 확장하며, 여러 데코레이터를 **중첩**해 다양한 조합을 만들 수 있다.

> “기존 코드를 변경하지 않고 기능을 확장한다.”

---

## 의도 (Intent)

- 런타임에 **새 기능을 부착/조합**한다.
- **OCP(개방-폐쇄 원칙)**를 지켜 기존 코드를 수정하지 않고 확장한다.
- 상속 폭발을 **합성 기반 조합**으로 대체한다.

---

## 구조 (UML)

```
          ┌────────────────┐
          │  Component     │◄────────────┐
          └────────────────┘             │
                   ▲                     │
                   │                     │
     ┌─────────────────────┐   ┌─────────────────────┐
     │ ConcreteComponent   │   │     Decorator       │
     └─────────────────────┘   └─────────────────────┘
                                       ▲
                                       │
                        ┌──────────────────────────┐
                        │   ConcreteDecorator      │
                        └──────────────────────────┘
```

- **Component**: 공통 인터페이스
- **ConcreteComponent**: 기본 기능 구현
- **Decorator**: `Component`를 구현하며 내부에 `Component`를 **래핑**
- **ConcreteDecorator**: 실제 부가기능을 구현

---

## 핵심 아이디어와 규칙

1) **인터페이스 동일성**: 데코레이터는 항상 **Component 계약**을 준수해야 한다.
2) **위임 + 부가로직**: `call()` 전/후에 교차 관심사(로깅, 캐시, 리트라이 등)를 적용한다.
3) **순서 중요**: `Retry→Timeout→Cache→Log` 등 **조합 순서**가 결과/성능을 좌우한다.
4) **얇고 단일 책임**: 데코레이터 하나는 하나의 관심사만. 체인으로 조합한다.

---

## — 커피 가격

> 사용자 예시를 살리되, **타입힌트와 확장 포인트**를 추가했다.

```python
from abc import ABC, abstractmethod

# Component

class Coffee(ABC):
    @abstractmethod
    def cost(self) -> int: ...
    def description(self) -> str: return "Coffee"

# ConcreteComponent

class BasicCoffee(Coffee):
    def cost(self) -> int: return 3000
    def description(self) -> str: return "Basic Coffee"

# Decorator

class CoffeeDecorator(Coffee):
    def __init__(self, coffee: Coffee): self._coffee = coffee
    def cost(self) -> int: return self._coffee.cost()
    def description(self) -> str: return self._coffee.description()

# ConcreteDecorators

class Milk(CoffeeDecorator):
    def cost(self) -> int: return self._coffee.cost() + 500
    def description(self) -> str: return self._coffee.description() + " + Milk"

class Whip(CoffeeDecorator):
    def cost(self) -> int: return self._coffee.cost() + 700
    def description(self) -> str: return self._coffee.description() + " + Whip"

class Caramel(CoffeeDecorator):
    def cost(self) -> int: return self._coffee.cost() + 1000
    def description(self) -> str: return self._coffee.description() + " + Caramel"

# 사용

coffee = Caramel(Whip(Milk(BasicCoffee())))
print(coffee.description(), coffee.cost())  # Basic Coffee + Milk + Whip + Caramel 5200
```

---

## — Repository에 로깅·리트라이·캐시 조합

> **순서 의존성**과 **예외 전파**를 보여준다.

```
Client → Logging → Retry → Cache → RealRepository
```

```python
import time
from abc import ABC, abstractmethod

# Component

class Repo(ABC):
    @abstractmethod
    def get(self, key: str) -> str: ...

# Concrete

class RealRepo(Repo):
    def __init__(self, store: dict[str, str]): self._db = store
    def get(self, key: str) -> str:
        # 예시로 불안정 네트워크/스토리지를 가정
        if key not in self._db: raise KeyError(key)
        return self._db[key]

# Decorators

class LoggingRepo(Repo):
    def __init__(self, inner: Repo): self._inner = inner
    def get(self, key: str) -> str:
        print(f"[LOG] get({key}) - begin")
        try:
            v = self._inner.get(key)
            print(f"[LOG] get({key}) - ok -> {v}")
            return v
        except Exception as e:
            print(f"[LOG] get({key}) - fail: {e}")
            raise

class RetryRepo(Repo):
    def __init__(self, inner: Repo, retries=3, backoff=0.05):
        self._inner, self._retries, self._backoff = inner, retries, backoff
    def get(self, key: str) -> str:
        last = None
        for i in range(self._retries):
            try:
                return self._inner.get(key)
            except Exception as e:
                last = e
                time.sleep(self._backoff * (2**i))  # 지수 백오프
        raise last

class CacheRepo(Repo):
    def __init__(self, inner: Repo):
        self._inner = inner
        self._cache: dict[str, str] = {}
    def get(self, key: str) -> str:
        if key in self._cache:
            return self._cache[key]
        v = self._inner.get(key)
        self._cache[key] = v
        return v

# 조합: Logging→Retry→Cache→Real

repo: Repo = LoggingRepo(RetryRepo(CacheRepo(RealRepo({"a":"A","b":"B"}))))
print(repo.get("a"))
```

### 순서 가이드

- **읽기 경로**: `Logging → Retry → Cache → Real`
  - 캐시 미스 시에만 리트라이가 작동.
  - 로깅은 항상 가장 바깥(가시성↑).
- **쓰기 경로**: `Logging → Retry → (옵션: CircuitBreaker/RateLimit) → Real`
- **보안/권한**: `AuthZ`는 바깥쪽, **입력 검증**은 더 바깥쪽.

---

## 예외 전파 정책

- 데코레이터는 **계약을 보존**해야 한다. 즉, 예외 타입/의미를 바꾸지 않는다(필요 시 **래핑**하되 원인을 체인에 남긴다).
- Retry/Timeout은 **복구 불가능한 예외**를 즉시 전파하고, **일시적 오류**에만 재시도한다.

---

## 성능·복잡도 분석

데코레이터가 \(k\)개일 때, 한 호출의 총 지연은

\[
T_{\text{total}} \;=\; T_{\text{base}} \;+\; \sum_{i=1}^{k} T_{\text{decorator},i}
\]

- I/O 바운드(네트워크/디스크)에서는 \(T_{\text{decorator},i}\)가 **무시 가능한 수준**인 경우가 많다.
- CPU 핫패스에서는 체인을 **얕게** 유지하고, 비용 큰 정책(직렬화/압축/암호화)은 **끝점 인접**에 배치.

메모리 오버헤드: 래핑 수 \(k\)에 비례하는 작은 객체/참조 비용.
캐시/버퍼 등을 쓰는 상태형 데코레이터는 **상태 규모**와 **무효화 비용**을 고려한다.

---

## 상태/동시성 설계

- **무상태 데코레이터**(로깅/권한/측정): 다중 스레드 친화적.
- **상태형 데코레이터**(캐시/버퍼/레이트리미트): 동시성 제어 필요.
  - Python: `threading.Lock`, C#: `ConcurrentDictionary`, 비동기 환경에선 **async** 버전 고려.
  - **TTL/무효화** 전략 문서화(오염 데이터 방지).

---

## 해부/진단: 체인 확인과 Unwrap

디버깅 시 **체인 순서**를 관찰하거나 특정 데코레이터를 **해제**할 필요가 있다.

```python
def unwrap(comp):
    seen = []
    while hasattr(comp, "_coffee") or hasattr(comp, "_inner"):
        seen.append(type(comp).__name__)
        comp = getattr(comp, "_coffee", getattr(comp, "_inner", comp))
    return seen, comp

print(unwrap(coffee))  # (["Caramel","Whip","Milk"], BasicCoffee)
```

실무에서는 **명확한 네이밍**과 **구성(설정)에서의 체인 선언**이 가장 큰 가시성을 준다.

---

## Decorator vs Inheritance vs Strategy vs Proxy (확장 표)

| 패턴            | 목적                          | 조합/순서 | 교체 시점  | 비고 |
|-----------------|-------------------------------|-----------|-----------|-----|
| **Decorator**    | 기능의 동적 부착              | 중요      | 런타임     | 래핑·합성 |
| **Inheritance**  | 타입/행동의 정적 확장         | 불가에 가깝 | 컴파일타임 | 클래스 폭발 위험 |
| **Strategy**     | 알고리즘 교체                 | 미미      | 런타임     | 내부 정책 교환 |
| **Proxy**        | 접근 제어/원격/지연/캐시      | 보통 중요 | 런타임     | 동일 인터페이스, “대리인” |
| **Middleware**   | 요청/응답 파이프라인(웹)      | 매우 중요 | 런타임     | 사실상 함수형 데코레이터 체인 |

> **프록시 vs 데코레이터**: 프록시는 “대리+제어”가 주목적, 데코레이터는 “부가기능 부착”이 주목적이지만, 실무에선 경계가 겹치기도 한다.

---

## C# 실전 예시 — 이메일 전송에 로깅·리트라이·레이트리미트

```csharp
public interface IEmailSender { Task SendAsync(string to, string subject, string body); }

// Concrete
public sealed class SmtpEmailSender : IEmailSender {
    public async Task SendAsync(string to, string subject, string body) {
        // SMTP 전송(생략)
        await Task.CompletedTask;
    }
}

// Decorators
public abstract class EmailSenderDecorator : IEmailSender {
    protected readonly IEmailSender Inner;
    protected EmailSenderDecorator(IEmailSender inner) => Inner = inner;
    public abstract Task SendAsync(string to, string subject, string body);
}

public sealed class LoggingEmailSender : EmailSenderDecorator {
    private readonly ILogger<LoggingEmailSender> _log;
    public LoggingEmailSender(IEmailSender inner, ILogger<LoggingEmailSender> log) : base(inner) { _log = log; }
    public override async Task SendAsync(string to, string subject, string body) {
        _log.LogInformation("Send to {To} : {Subject}", to, subject);
        await Inner.SendAsync(to, subject, body);
    }
}

public sealed class RetryEmailSender : EmailSenderDecorator {
    public RetryEmailSender(IEmailSender inner) : base(inner) {}
    public override async Task SendAsync(string to, string subject, string body) {
        int retries = 3; Exception? last = null;
        for (int i=0;i<retries;i++) {
            try { await Inner.SendAsync(to, subject, body); return; }
            catch (Exception e) { last = e; await Task.Delay(50 << i); }
        }
        throw last!;
    }
}

public sealed class RateLimitEmailSender : EmailSenderDecorator {
    private readonly SemaphoreSlim _sem = new(5); // 동시 5개
    public RateLimitEmailSender(IEmailSender inner) : base(inner) {}
    public override async Task SendAsync(string to, string subject, string body) {
        await _sem.WaitAsync();
        try { await Inner.SendAsync(to, subject, body); }
        finally { _sem.Release(); }
    }
}

// 구성(체인): Logging → Retry → RateLimit → Smtp
// ASP.NET Core DI 예시
// services.AddTransient<IEmailSender>(sp =>
//   new LoggingEmailSender(
//     new RetryEmailSender(
//       new RateLimitEmailSender(
//          new SmtpEmailSender())),
//     sp.GetRequiredService<ILogger<LoggingEmailSender>>()));
```

---

## 체인 설계 체크리스트

- **순서**: Log(가시성↑) → AuthZ/Validation → Metrics → Cache → Retry/Timeout → Real
- **단일 책임**: 데코레이터 하나는 하나의 concern만.
- **계약 보존**: 반환형/예외 계약을 바꾸지 않는다.
- **구성 중심**: 체인은 코드가 아닌 **설정/DI**로 선언(가시성/테스트↑).
- **추적성**: 요청 ID/코릴레이션 ID를 가장 바깥에서 주입, 내부로 전파.

---

## 테스트 전략

- **계약 테스트**: 데코레이터 체인 유무와 무관하게 `Component` 계약이 유지되는지.
- **순서 검증**: 스파이(Spy) 컴포넌트로 호출 순서를 기록·검증.
- **페일 패스**: 예외가 올바르게 전파·변환되는지, Retry/Timeout 정책이 의도대로 동작하는지.
- **성능 가드**: 체인 깊이 증가에 따른 지연 상한을 모니터링.

Python 간단 검증:

```python
class SpyRepo(Repo):
    def __init__(self): self.calls=[]
    def get(self, key:str)->str: self.calls.append(key); return key.upper()

spy = SpyRepo()
wrapped = LoggingRepo(CacheRepo(spy))
assert wrapped.get("x") == "X"
assert wrapped.get("x") == "X"  # 캐시 히트, spy는 1회만 호출
assert spy.calls == ["x"]
```

---

## 흔한 함정과 리팩토링

- **거대 데코레이터**: 여러 관심사를 한 클래스에 몰아넣지 말고 **쪼개서 조합**한다.
- **중복 적용/이중 로깅**: 체인 구성 중복 방지, **구성 루트**(한 곳)에서만 정의.
- **계약 파괴**: 반환/예외 계약을 바꾸면 LSP 위반. 어댑터나 별도 컴포넌트로 분리.
- **상태 오염**: 캐시/버퍼는 **무효화/TTL/동시성** 정책을 문서화.
- **과도한 체인 깊이**: 핫패스에서는 **정적 합성(소수만)**, 나머지는 외부에서 처리(예: API Gateway/미들웨어).

---

## 사용 사례(보강)

| 시나리오                | 데코레이터 예                         | 비고 |
|-------------------------|---------------------------------------|------|
| I/O 스트림              | Buffering, Compression, Encryption     | 순서 중요(압축↔암호화) |
| 저장소/레포지토리       | Logging, Caching, Retry, CircuitBreak | 데이터 일관성 주의 |
| 웹/마이크로서비스       | AuthZ, RateLimit, Metrics, Tracing     | 미들웨어 체인과 유사 |
| GUI 위젯                | Border, Scroll, Shadow                 | 렌더 파이프라인 |
| 알림 전송               | Log, Fan-out(Email/SMS/Slack)          | 멱등성 고려 |

---

## 함수 데코레이터와의 구분(파이썬)

파이썬의 `@decorator` 문법은 **함수/메서드**에 부가기능을 부착하는 **언어 레벨 기능**이다.
패턴으로서의 **객체 데코레이터**와 목적은 유사하지만, **적용 대상과 생애주기**가 다르다.
- 함수 데코레이터: 호출 전후 래핑(예: 캐싱/로그).
- 객체 데코레이터: **인터페이스 전체**를 래핑, 다른 구현으로 **조합·교체**가 쉬움.

---

## 마무리

데코레이터는 **합성 기반의 동적 확장**으로 상속 폭발을 치유한다.
핵심은 **얇고 단일 책임의 래퍼**를 **올바른 순서**로 조합하고, **계약 보존·예외 전파·상태/동시성**을 명확히 하는 것이다.
DI/설정으로 체인을 선언해 가시성을 높이고, 테스트로 **순서·성능·페일 패스**를 가드하라.
이 원칙을 지키면, 복잡한 시스템에서도 데코레이터는 **가볍고 강력한 확장 도구**가 된다.
