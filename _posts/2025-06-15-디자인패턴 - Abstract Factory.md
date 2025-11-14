---
layout: post
title: 디자인패턴 - Abstract Factory
date: 2025-06-15 19:20:23 +0900
category: 디자인패턴
---
# Abstract Factory(추상 팩토리 패턴)

## 정의(Definition)

**추상 팩토리(Abstract Factory)**는 **서로 관련된 객체들의 제품군(family of products)**을 **일관되게 생성**하기 위한 **인터페이스**를 제공하는 생성 패턴이다.
클라이언트는 **구체 클래스**를 알 필요가 없고, **추상 타입에만 의존**한다. 제품군의 예: “다크 테마 위젯 세트(버튼/체크박스/라벨)”, “DB 제품군(커넥션/커맨드/트랜잭션)”, “클라우드 제품군(스토리지/큐/시크릿)”.

---

## 의도(Intent)·적용 시점

- 관련 객체들을 **한 세트**로 **일관되게** 생성/교체하고 싶다.
- 플랫폼/테마/벤더에 따라 **여러 객체를 함께 바꿔야** 한다.
- 클라이언트가 **구체 클래스 대신 인터페이스**만 보도록 결합도를 낮추고 싶다.
- **Factory Method**의 확장형: *하나*가 아니라 **여러 제품 종류**를 동시에 만든다.

**적용 체크리스트**
- (필수) 서로 **궁합이 맞는** 객체 묶음을 항상 같이 써야 하는가?
- (권장) 구현 교체가 **환경/설정/런타임 조건**에 의해 자주 필요한가?
- (권장) “A는 다크, B는 라이트” 같은 **불일치 조합**을 원천 차단하고 싶은가?

---

## 구조(Structure) — 제공한 그림을 보존

```
          ┌─────────────────────────┐
          │   AbstractFactory       │◄────────────────────┐
          ├─────────────────────────┤                     │
          │ + createProductA()      │                     │
          │ + createProductB()      │                     │
          └─────────────────────────┘                     │
                   ▲                                      │
                   │                                      │
     ┌─────────────────────────┐        ┌─────────────────────────┐
     │ ConcreteFactory1        │        │ ConcreteFactory2        │
     ├─────────────────────────┤        ├─────────────────────────┤
     │ + createProductA()      │        │ + createProductA()      │
     │ + createProductB()      │        │ + createProductB()      │
     └─────────────────────────┘        └─────────────────────────┘
              ▲                                      ▲
              │                                      │
   ┌───────────────────────┐            ┌───────────────────────┐
   │ AbstractProductA      │            │ AbstractProductB      │
   └───────────────────────┘            └───────────────────────┘
              ▲                                      ▲
   ┌───────────────────────┐            ┌───────────────────────┐
   │ ProductA1             │            │ ProductB1             │
   └───────────────────────┘            └───────────────────────┘
```

**참여자 역할**
- **AbstractFactory**: 제품군 생성을 위한 **계약**.
- **ConcreteFactoryX**: 특정 테마/벤더에 맞는 **일관된 제품군** 생성.
- **AbstractProductA/B**: 제품군을 구성하는 **각 제품 타입의 계약**.
- **ProductAX/BX**: **구체 제품**.

**효과(조합 제약의 수학적 관점)**
제품 타입이 \(k\)개, 각 타입의 변형(테마/벤더)이 \(t\)개라면, 아무 제약 없이 섞으면 가능한 조합은 $$t^k$$ 이다.
추상 팩토리로 “일관된 세트만 허용”하면 합법 조합은 **정확히 \(t\)**(각 팩토리 1조합)로 줄어 **불일치 조합을 구조적으로 차단**한다.

---

## 최소 구현 — Python(GUI 위젯 제품군)

```python
from abc import ABC, abstractmethod

# 제품 계약

class Button(ABC):
    @abstractmethod
    def render(self) -> None: ...

class Checkbox(ABC):
    @abstractmethod
    def render(self) -> None: ...

# 구체 제품 (Windows)

class WindowsButton(Button):
    def render(self) -> None:
        print("윈도우 버튼 렌더링")

class WindowsCheckbox(Checkbox):
    def render(self) -> None:
        print("윈도우 체크박스 렌더링")

# 구체 제품 (Mac)

class MacButton(Button):
    def render(self) -> None:
        print("맥 버튼 렌더링")

class MacCheckbox(Checkbox):
    def render(self) -> None:
        print("맥 체크박스 렌더링")

# 추상 팩토리

class GUIFactory(ABC):
    @abstractmethod
    def create_button(self) -> Button: ...
    @abstractmethod
    def create_checkbox(self) -> Checkbox: ...

# 구체 팩토리

class WindowsFactory(GUIFactory):
    def create_button(self) -> Button: return WindowsButton()
    def create_checkbox(self) -> Checkbox: return WindowsCheckbox()

class MacFactory(GUIFactory):
    def create_button(self) -> Button: return MacButton()
    def create_checkbox(self) -> Checkbox: return MacCheckbox()

# 클라이언트 (팩토리만 의존)

def render_ui(factory: GUIFactory) -> None:
    btn = factory.create_button()
    chk = factory.create_checkbox()
    btn.render(); chk.render()

render_ui(WindowsFactory())
render_ui(MacFactory())
```

**핵심**: 클라이언트는 **GUIFactory 인터페이스만** 알고, 제품의 **구체 클래스**는 모른다.
**효과**: 테마 전환시 `WindowsFactory ↔ MacFactory`만 바꾸면 된다.

---

## C# 예시 — DB 제품군(Conn/Command/Tx)

```csharp
public interface IDbConnectionX { void Open(); }
public interface IDbCommandX { int Execute(string sql); }
public interface ITransactionX : IDisposable { void Commit(); void Rollback(); }

public interface IDbFactory {
    IDbConnectionX CreateConnection(string cs);
    IDbCommandX CreateCommand(IDbConnectionX conn);
    ITransactionX CreateTransaction(IDbConnectionX conn);
}

// ConcreteFactory: MySQL
public class MySqlFactory : IDbFactory {
    public IDbConnectionX CreateConnection(string cs) => new MySqlConn(cs);
    public IDbCommandX CreateCommand(IDbConnectionX c) => new MySqlCmd(c);
    public ITransactionX CreateTransaction(IDbConnectionX c) => new MySqlTx(c);
}

// ConcreteFactory: PostgreSQL
public class PgFactory : IDbFactory {
    public IDbConnectionX CreateConnection(string cs) => new PgConn(cs);
    public IDbCommandX CreateCommand(IDbConnectionX c) => new PgCmd(c);
    public ITransactionX CreateTransaction(IDbConnectionX c) => new PgTx(c);
}

// Client
public class Repository {
    private readonly IDbFactory _f; private readonly string _cs;
    public Repository(IDbFactory f, string cs){ _f=f; _cs=cs; }

    public int Save(string sql){
        var conn = _f.CreateConnection(_cs);
        conn.Open();
        using var tx = _f.CreateTransaction(conn);
        var cmd = _f.CreateCommand(conn);
        var rows = cmd.Execute(sql);
        tx.Commit();
        return rows;
    }
}
```

**효과**: DB 벤더 교체 시 **팩토리 바인딩만 변경**.
**테스트**: `FakeDbFactory`를 주입하여 **순수 단위 테스트** 가능.

---

## Java 예시 — 테마 위젯 + ServiceLoader(선택)

```java
public interface Button { void render(); }
public interface Checkbox { void render(); }

public interface GuiFactory {
  Button createButton();
  Checkbox createCheckbox();
}

public class DarkFactory implements GuiFactory {
  public Button createButton(){ return new DarkButton(); }
  public Checkbox createCheckbox(){ return new DarkCheckbox(); }
}

// 선택: META-INF/services/GuiFactory 등록 후 동적 로딩
// ServiceLoader<GuiFactory> loader = ServiceLoader.load(GuiFactory.class);
// GuiFactory f = loader.iterator().next();
```

**주의**: `ServiceLoader`나 리플렉션 기반 로딩 시 **신뢰 경계/예외 처리**를 명확히.

---

## 변형(Variants)·구현 기법

### 등록(Registry) 기반 추상 팩토리

런타임에 **키→팩토리**를 등록/선택(플러그인 친화).

```python
class FactoryRegistry:
    _map = {}
    @classmethod
    def register(cls, key: str, factory: GUIFactory) -> None:
        if key in cls._map: raise KeyError("duplicate key")
        cls._map[key] = factory
    @classmethod
    def resolve(cls, key: str) -> GUIFactory:
        return cls._map[key]

FactoryRegistry.register("win", WindowsFactory())
FactoryRegistry.register("mac", MacFactory())
factory = FactoryRegistry.resolve("win")
```

### 프로토타입 기반 추상 팩토리

**프로토타입 레지스트리**에서 복제하여 제품군 생성.

```python
import copy

class ProtoFactory(GUIFactory):
    def __init__(self, p_btn: Button, p_chk: Checkbox):
        self._p_btn = p_btn; self._p_chk = p_chk
    def create_button(self) -> Button: return copy.deepcopy(self._p_btn)
    def create_checkbox(self) -> Checkbox: return copy.deepcopy(self._p_chk)
```

### 함수형/람다 기반(경량)

팩토리 자체를 **생성자 함수들의 모음**으로 본다.

```python
from typing import Callable

class FuncFactory(GUIFactory):
    def __init__(self, mk_btn: Callable[[], Button], mk_chk: Callable[[], Checkbox]):
        self._mk_btn = mk_btn; self._mk_chk = mk_chk
    def create_button(self) -> Button: return self._mk_btn()
    def create_checkbox(self) -> Checkbox: return self._mk_chk()
```

### DI/IoC와의 결합

- **조립은 컨테이너**, **선택은 팩토리**: 환경/테넌트/플래그에 따라 팩토리를 **주입/선택**.
- ASP.NET Core 예(개념):
  - `services.AddSingleton<IDbFactory, MySqlFactory>();`
  - 환경별 프로필에서 다른 팩토리 바인딩.

---

## 제품 추가 vs 제품군 추가 — OCP 트레이드오프

| 변화 | 영향 | 비고 |
|---|---|---|
| **새 변형(테마/벤더) 추가** | **ConcreteFactoryX** 추가(기존 코드 수정 X) | OCP 준수 |
| **새 제품 타입(C를 추가)** | `AbstractFactory`에 `createC()` **추가** + 모든 ConcreteFactory 수정 | 인터페이스 변경 → OCP **부분 위배** |

- 추상 팩토리는 “**세트의 일관성**”을 위해 **인터페이스가 넓어지기 쉬움**.
- 제품 추가가 잦다면 **Factory of Factories**(제품별 팩토리를 모아 조립)이나 **컴포지션**(각 제품용 팩토리를 멤버로) 고려.

---

## 실제 시나리오 설계 템플릿 3선

### UI 테마(다크/라이트/하이 콘트라스트)

- 제품군: `Button/Checkbox/Label/Panel`
- 제약: 테마 **일관성** 필수 → 추상 팩토리 적합
- 변형: 등록 기반 + A/B 테스트 토글과 연계

### DB 벤더(MySQL/PostgreSQL/SQLite)

- 제품군: `Connection/Command/Transaction`
- 교체: CI에서 **모든 팩토리 계약 테스트** 수행

### 클라우드 벤더(AWS/Azure/GCP)

- 제품군: `StorageClient/QueueClient/SecretClient`
- 보안: 자격증명/엔드포인트 주입, 서명 전략은 **Strategy**로 분리 → 팩토리가 전략을 조합해 생성

---

## 테스트 전략(계약 테스트 & 모킹)

- **계약 테스트**: 제품군 계약(예: `Button.render`가 예외 없이 호출 가능)을 공통 테스트로 만들고, **모든 ConcreteFactory**에 대해 **동일 테스트 재사용**.
- **InMemory/TestFactory**: 실제 I/O 없는 테스트용 팩토리를 주입.
- **불일치 조합 방지 검증**: 일부러 “다크 버튼 + 라이트 체크박스”를 만들 방법이 **구조적으로 없는지**를 확인.

**Python 계약 테스트 스켈레톤**
```python
import pytest

@pytest.mark.parametrize("factory", [WindowsFactory(), MacFactory()])
def test_gui_family_contract(factory: GUIFactory):
    btn = factory.create_button()
    chk = factory.create_checkbox()
    btn.render()
    chk.render()
```

---

## 성능·동시성·수명 고려

- **지연 로딩**: 팩토리가 내부 캐시/풀과 결합될 때 초기 지연 vs 호출 지연을 트레이드오프.
- **수명 관리**: C#에서는 `IDisposable` 제품을 팩토리가 만든다면 **소유권**과 **해제 시점**을 명시(클라이언트가 해제? 팩토리가 풀 관리?).
- **동시성**: 팩토리는 **무상태(stateless)**가 이상적. 내부 캐시가 필요하면 락 범위를 최소화하고 **불변 객체**/원자적 참조 교체를 고려.

---

## 리팩토링 가이드 — 흩어진 생성 → 추상 팩토리

**냄새**
- 여러 곳에 `if (theme==dark) new DarkButton() else new LightButton()`
- “교체 정책”과 “사용 로직”이 뒤섞임

**절차**
1) 제품별 인터페이스(추상 제품)를 정의.
2) 현재 생성 코드를 **임시 팩토리**로 모아 `createA/B/...`로 캡슐화.
3) 변형별 구현을 **ConcreteFactoryX**로 분리.
4) 클라이언트는 **팩토리만** 의존하도록 의존성 역전(DIP).
5) 구성/환경에 따라 적절한 팩토리를 바인딩(등록/DI/플래그).

---

## 비교 — Factory Method / Builder / Prototype

| 항목 | Factory Method | **Abstract Factory** | Builder | Prototype |
|---|---|---|---|---|
| 초점 | 단일 제품 생성 확장 | **제품군(세트) 일관 생성** | 복잡 생성 단계 분리 | 복제 기반 빠른 생성 |
| 인터페이스 크기 | 작음 | **큼(제품 수만큼 메서드)** | 1개 빌더 + 다수 단계 | clone 계약 |
| 유연성 | 중 | 중~높음(세트 단위 교체) | 높음(옵션/순서) | 중 |
| OCP(새 제품 추가) | 영향 적음 | **인터페이스 변경** 필요 | 영향 없음 | 영향 없음 |

---

## 안티패턴·흔한 실수

- **불필요한 세트화**: 실제로 함께 쓰지 않는 제품을 억지로 묶음 → 과추상화.
- **인터페이스 비대화**: 제품 수가 늘며 `AbstractFactory`가 점점 커짐 → **분할/컴포지션** 고려.
- **리플렉션 남용**: 설정 이름→클래스 매핑을 전가 → 타입 안정성/보안 취약.
- **수명 혼동**: 팩토리가 만든 자원의 **소유권/해제 책임** 불명확.

---

## 추가 예제 — 함수형 조합과 전략 결합

**전송 클라이언트 제품군 + 전략(서명/재시도) 조합**

```python
class Transport(ABC):
    @abstractmethod
    def send(self, data: bytes) -> None: ...

class RetryStrategy(ABC):
    @abstractmethod
    def run(self, fn): ...

class NoRetry(RetryStrategy):
    def run(self, fn): return fn()

class ExponentialBackoff(RetryStrategy):
    def run(self, fn):
        # 재시도 로직(간소화)
        for _ in range(3):
            try: return fn()
            except Exception: pass
        raise

class HttpTransport(Transport):
    def __init__(self, retry: RetryStrategy): self._retry = retry
    def send(self, data: bytes) -> None: self._retry.run(lambda: print("HTTP send"))

class GrpcTransport(Transport):
    def __init__(self, retry: RetryStrategy): self._retry = retry
    def send(self, data: bytes) -> None: self._retry.run(lambda: print("gRPC send"))

class NetFactory(ABC):
    @abstractmethod
    def create_transport(self) -> Transport: ...

class HttpFactory(NetFactory):
    def __init__(self, retry: RetryStrategy): self._retry = retry
    def create_transport(self) -> Transport: return HttpTransport(self._retry)

class GrpcFactory(NetFactory):
    def __init__(self, retry: RetryStrategy): self._retry = retry
    def create_transport(self) -> Transport: return GrpcTransport(self._retry)
```

- **전략(Strategy)**을 팩토리에 주입 → 제품군 생성 시 **일관된 정책**을 공유.

---

## 체크리스트(최종)

- 세트 일관성(테마/벤더)이 **핵심 제약**인가?
- 제품 **변형 추가**가 흔하고, 제품 **종류 추가**는 드문가? (반대면 다른 설계 고려)
- DI/구성/플러그인으로 팩토리를 **안전하게 선택**할 수 있는가?
- 테스트에서 **계약 테스트**와 **InMemory 팩토리**를 준비했는가?
- 자원 **수명/소유권**과 **동시성**을 명확히 했는가?

---

## 장단점 요약

**장점**
- 세트 단위 **일관 생성/교체**
- 클라이언트의 **구체 클래스 의존 제거**
- 테스트 대체(모킹) 용이, 제품군 불일치 조합 차단

**단점**
- 인터페이스 비대화(새 제품 추가 시 **모든 팩토리 변경**)
- 클래스 수 증가, 복잡도 상승 가능
- 세트화가 과하면 **오버엔지니어링**

---

## 마무리

추상 팩토리는 “**세트로 움직이는 변화**”를 **추상화된 공장** 하나로 수렴시켜 **일관성·교체 용이성**을 보장한다. 실제 시스템에서 **Factory Method/Strategy/DI**와 함께 조합해 쓰면 가장 강력하다. 다만, **제품 타입 추가의 OCP 트레이드오프**와 **인터페이스 비대화**를 항상 염두에 두고, 억지로 세트화하지 말 것. **현실의 변화 축**을 정확히 읽는 것이 패턴 성공의 핵심이다.
