---
layout: post
title: 디자인패턴 - Gang of Four
date: 2025-06-09 20:20:23 +0900
category: 디자인패턴
---
# GoF(Gang of Four) 디자인 패턴

## GoF 디자인 패턴이란?

- 1994년 고전 『Design Patterns: Elements of Reusable Object-Oriented Software』에서 정리된 **객체지향 설계 패턴 23개**를 말한다.
- 저자: **Erich Gamma, Richard Helm, Ralph Johnson, John Vlissides**(일명 GoF).
- 목표: 반복적으로 등장하는 설계 문제에 대한 **재사용 가능한 설계 해법**을 **언어 중립적**으로 제공.

**핵심 특징**
- OOP 원리(추상화, 캡슐화, 상속, 다형성)와 **느슨한 결합·높은 응집**을 촉진
- **유지보수성, 확장성, 재사용성**을 높이는 구조 제공
- 실전 경험에서 나온 **검증된 설계 지침**

---

## GoF 23개 패턴 분류(요약표)

| 분류 | 패턴 |
|---|---|
| **생성(Creational)** | Singleton, Factory Method, Abstract Factory, Builder, Prototype |
| **구조(Structural)** | Adapter, Bridge, Composite, Decorator, Facade, Flyweight, Proxy |
| **행위(Behavioral)** | Observer, Strategy, Command, State, Template Method, Iterator, Mediator, Chain of Responsibility, Visitor, Memento, Interpreter |

---

## 학습·적용 가이드(빠른 나침반)

- **생성 복잡/다양?** → Builder(단계 분리), Factory Method(타입별), Abstract Factory(제품군)
- **호환 안 되는 인터페이스?** → Adapter
- **추상과 구현 독립 변화?** → Bridge
- **트리 구조/전체=부분 동일?** → Composite
- **런타임 기능 조합/횡단 관심사?** → Decorator
- **서브시스템 단순 진입점?** → Facade
- **동일 객체 대량/메모리 압박?** → Flyweight
- **원격/지연/접근 제어?** → Proxy
- **알고리즘 교체?** → Strategy
- **상태에 따른 행위 변화?** → State
- **이벤트 알림/관찰?** → Observer
- **요청 캡슐화/작업 큐/Undo?** → Command
- **연쇄 필터/미들웨어?** → Chain of Responsibility
- **알고리즘 골격 고정+커스터마이즈?** → Template Method
- **내부구조 은닉 순회?** → Iterator
- **상호의존 복잡 상호작용 조정?** → Mediator
- **새 연산 자주 추가(AST 등)?** → Visitor
- **스냅샷/복원?** → Memento
- **도메인 작은 언어 해석?** → Interpreter

---

## SOLID ↔ 패턴 맵(실전 감각)

- **SRP**: Facade(한 책임으로 단순화), Builder(생성과 표현 분리)
- **OCP**: Strategy/Decorator/Factory Method(확장으로 수용)
- **LSP**: Template Method/Bridge(치환 가능한 서브타입 계약)
- **ISP**: Adapter(좁은 인터페이스로 수축), Abstract Factory(제품군 인터페이스 분리)
- **DIP**: Factory 계열/DI 컨테이너(상위 수준이 추상에 의존)

---

## 생성 패턴(Creational)

### Singleton — 인스턴스 1개 보장

**의도**: **단일 인스턴스**와 전역 접근점 보장
**구조**
```
Client -> Singleton.getInstance() -> Singleton
```
**Python**
```python
class Config:
    _inst = None
    def __new__(cls):
        if cls._inst is None:
            cls._inst = super().__new__(cls)
            cls._inst.region = "ap-northeast-2"
        return cls._inst
```
**C#**
```csharp
public sealed class Config {
    private static readonly Lazy<Config> _inst = new(() => new Config());
    public static Config Instance => _inst.Value;
    private Config() {}
}
```
**테스트 포인트**: 동일 참조, 스레드 안전.
**주의**: 테스트 어려움(전역 상태). 가능하면 DI로 대체.

---

### Factory Method — 하위가 생성 책임

**의도**: 생성 책임을 **서브클래스**에 위임, 클라이언트는 추상에 의존
**구조(제공 도식과 동일 형태)**
```
Creator
 ├─ + factoryMethod(): Product  ◄── 확장 지점
 └─ + operation()               ── 공통 흐름
ConcreteCreatorA/B → ConcreteProductA/B
```
**Python**
```python
class Button:
    def render(self): raise NotImplementedError

class WinButton(Button):
    def render(self): print("Windows Button")

class MacButton(Button):
    def render(self): print("Mac Button")

class Dialog:
    def create_button(self) -> Button: raise NotImplementedError
    def render_window(self):
        btn = self.create_button()
        btn.render()

class WinDialog(Dialog):
    def create_button(self) -> Button: return WinButton()

class MacDialog(Dialog):
    def create_button(self) -> Button: return MacButton()

Dialog().render_window()  # 추상은 직접 사용하지 않고, 구체 Creator 사용
```
**리팩토링 경로**: 거대한 `switch` 분기 → Creator 서브클래스 분리
**주의**: 단순 생성엔 과도, 정적 팩토리/직접 생성이 더 낫기도 함.

---

### Abstract Factory — 제품군 일관 생성

**의도**: 관련된 **제품군**을 일관되게 생성
**구조**
```
AbstractFactory
 ├─ createButton(): Button
 └─ createTextbox(): Textbox
ConcreteFactoryDark/Light → DarkButton/LightButton, ...
```
**C#**
```csharp
interface IButton { void Draw(); }
interface ITextbox { void Draw(); }

interface IWidgetFactory { IButton Button(); ITextbox Textbox(); }

class DarkFactory : IWidgetFactory {
    public IButton Button() => new DarkButton();
    public ITextbox Textbox() => new DarkTextbox();
}
```
**선택 팁**: “세트” 일관성이 중요하면 추상 팩토리, 단일 축이면 팩토리 메서드.

---

### Builder — 복잡 생성 단계 분리

**의도**: 복잡한 객체의 **구성 단계를 분리**해 다양한 표현
**Python**
```python
class SqlBuilder:
    def __init__(self): self.parts = ["SELECT * FROM users"]
    def where(self, cond): self.parts.append("WHERE " + cond); return self
    def order(self, col): self.parts.append("ORDER BY " + col); return self
    def build(self): return " ".join(self.parts)
q = SqlBuilder().where("age>=18").order("name").build()
```
**주의**: 과도한 Fluent는 가독성 저하.

---

### Prototype — 복제로 생성

**의도**: 기존 객체를 **복제**해 새 객체 생성
**Python**
```python
import copy
class Node:
    def __init__(self, text, style): self.text, self.style = text, style
    def clone(self): return copy.deepcopy(self)
```
**주의**: 얕은/깊은 복사 명확히, 식별자 재사용 금지.

---

## 구조 패턴(Structural)

### Adapter — 인터페이스 변환

**의도**: 호환되지 않는 인터페이스 연결
**구조**
```
Client → Target
        ↑
     Adapter → Adaptee
```
**C#**
```csharp
interface INewPay { void Pay(decimal amount); }
class OldGateway { public void Send(int cents) { /* ... */ } }

class PaymentAdapter : INewPay {
    private readonly OldGateway _old = new();
    public void Pay(decimal amount) => _old.Send((int)(amount * 100));
}
```
**주의**: 에러/예외 매핑 명확히.

---

### Bridge — 추상과 구현 분리

**의도**: 추상(Abstraction)과 구현(Implementor) **독립 변화**
**구조**
```
Abstraction ── uses ──> Implementor
   ▲                         ▲
RefinedAbstraction      ConcreteImplementor
```
**C#**
```csharp
interface IRenderer { void DrawCircle(float x, float y, float r); }
abstract class Shape { protected IRenderer R; protected Shape(IRenderer r){R=r;} public abstract void Draw(); }
class Circle : Shape { float x,y,r; public Circle(IRenderer r,float x,float y,float r):base(r){this.x=x;this.y=y;this.r=r;} public override void Draw()=>R.DrawCircle(x,y,r); }
```
**장점**: 조합 폭발 억제(상속 대신 합성).

---

### Composite — 전체/부분 동일 취급

**의도**: 트리 구조에서 Leaf와 Composite를 **동일 인터페이스**로
**구조**
```
Component
 ├─ Leaf
 └─ Composite → children: Component*
```
**Python**
```python
class Component:
    def render(self): raise NotImplementedError
class Leaf(Component):
    def __init__(self, text): self.text = text
    def render(self): return self.text
class Composite(Component):
    def __init__(self): self.children=[]
    def add(self, c): self.children.append(c)
    def render(self): return "".join(ch.render() for ch in self.children)
```

---

### Decorator — 동적 기능 추가

**의도**: **상속 대신 합성**으로 기능을 런타임에 덧입힘
**구조**
```
Component <─ Decorator ─ Decorator2 ... → ConcreteComponent
```
**C#**
```csharp
interface IService { string Get(); }
class Core : IService { public string Get()=>"data"; }
class Logging : IService { private readonly IService _next; public Logging(IService n){_next=n;} public string Get(){ Console.WriteLine("call"); return _next.Get(); } }
class Caching : IService { private readonly IService _next; private string _v; public Caching(IService n){_next=n;} public string Get()=> _v ??= _next.Get(); }
var svc = new Caching(new Logging(new Core()));
```
**주의**: 체인 순서·디버깅 난이도.

---

### Facade — 단순한 진입점

**의도**: 복잡한 서브시스템에 **간단한 인터페이스** 제공
**구조**
```
Client → Facade → SubsystemA/B/C
```
**Python**
```python
class MediaFacade:
    def process(self, path):
        # decode -> filter -> encode 등 내부 복잡도 은닉
        return "ok"
```

---

### Flyweight — 공유로 메모리 절약

**의도**: **내재 상태**를 공유해 대량 객체 메모리 절감
**Python**
```python
class IconFactory:
    _pool={}
    @classmethod
    def get(cls, name):
        if name not in cls._pool: cls._pool[name]=load_icon(name)
        return cls._pool[name]
```
**주의**: 외재 상태(좌표·색상 등)는 호출자가 제공.

---

### Proxy — 대리 접근/지연/원격

**의도**: 접근 제어, 지연 로딩, 원격 호출
**구조**
```
Client → Proxy → RealSubject
```
**C#**
```csharp
class LazyImage : IImage {
    private RealImage _real;
    public void Draw(){ (_real ??= new RealImage("a.png")).Draw(); }
}
```

---

## 행위 패턴(Behavioral)

### Strategy — 알고리즘 교체 가능

**의도**: 알고리즘 군을 인터페이스로 **캡슐화**해 교체
**C#**
```csharp
interface ICompression { byte[] Compress(byte[] src); }
class Zip : ICompression { public byte[] Compress(byte[] s){ /* ... */ return s; } }
class Gzip : ICompression { public byte[] Compress(byte[] s){ /* ... */ return s; } }

class Archiver {
    private ICompression _algo;
    public Archiver(ICompression a){ _algo=a; }
    public void Set(ICompression a)=>_algo=a;
    public byte[] Run(byte[] s)=>_algo.Compress(s);
}
```
**수식 예(조합 수)**: 데코레이터와 함께 k개 전략/옵션을 조합하면 가능한 경우의 수는 $$2^{k}$$ (선택/비선택 기준).

---

### Observer — 상태 변화 통지

**의도**: 발행자 상태 변화 → 구독자에 자동 통지(느슨한 결합)
**C#**
```csharp
class Publisher {
    public event Action<int> Changed;
    private int _v;
    public void Set(int v){ _v = v; Changed?.Invoke(_v); }
}
class Subscriber { public void OnChanged(int v){ /* ... */ } }
```
**주의**: 구독 해지 누락 → 메모리 누수. 약한 참조/해지 규약 마련.

---

### Command — 요청을 객체로 캡슐화

**의도**: 요청을 객체화하여 큐잉/로그/Undo 지원
**Python**
```python
class Command:
    def execute(self): ...
    def undo(self): ...
class Insert(Command):
    def __init__(self, buf, text): self.buf, self.text = buf, text
    def execute(self): self.buf.append(self.text)
    def undo(self): self.buf.pop()

hist=[]; buf=[]
cmd=Insert(buf,"A"); cmd.execute(); hist.append(cmd)
hist.pop().undo()
```

---

### State — 상태에 따른 행위 변경

**의도**: 분기 폭발을 **상태 객체**로 치환
**Python**
```python
class State:
    def pay(self, o): raise NotImplementedError
class New(State):
    def pay(self,o): o.state = Paid()
class Paid(State):
    def pay(self,o): raise Exception("already")
class Order:
    def __init__(self): self.state = New()
    def pay(self): self.state.pay(self)
```

---

### Template Method — 알고리즘 골격 고정

**의도**: 알고리즘 **뼈대**를 상위가 정의, **세부 단계**는 하위가 결정
**C#**
```csharp
abstract class Pipeline {
    public void Run(){ Load(); Transform(); Save(); }
    protected abstract void Load(); protected abstract void Transform(); protected abstract void Save();
}
```

---

### Iterator — 내부구조 은닉 순회

**Python**
```python
class Range:
    def __init__(self,n): self.n=n
    def __iter__(self):
        for i in range(self.n): yield i
```

---

### Mediator — 상호작용 중재

**의도**: 객체 간 상호참조/의존을 **중앙 중재자**로 흡수
**Python**
```python
class Bus:
    def __init__(self): self._subs={}
    def on(self, evt, fn): self._subs.setdefault(evt,[]).append(fn)
    def emit(self, evt, *a): [fn(*a) for fn in self._subs.get(evt,[])]
```

---

### Chain of Responsibility — 연쇄 처리

**의도**: 요청을 **연결된 처리자**에 전달
**C#**
```csharp
abstract class Handler {
    protected Handler Next;
    public Handler SetNext(Handler n){ Next=n; return n; }
    public virtual void Handle(Request r){ Next?.Handle(r); }
}
```
**주의**: 누락 시 무처리 위험 → 기본 처리/종료 조건 명확히.

---

### Visitor — 구조 고정, 새 연산 추가

**의도**: 자료 구조는 그대로, **연산을 외부로 추가**
**Java(개요)**
```java
interface Visitor { void visit(Foo f); void visit(Bar b); }
interface Node { void accept(Visitor v); }
class Foo implements Node { public void accept(Visitor v){ v.visit(this);} }
class Bar implements Node { public void accept(Visitor v){ v.visit(this);} }
```
**주의**: 새 노드 타입 추가는 어려움(더블 디스패치 목록 갱신 필요).

---

### Memento — 스냅샷/복원

**의도**: 캡슐화 유지한 채 **상태 저장·복원**
**주의**: 스냅샷 비용/빈도 관리, 보안상 민감 필드 암호화 고려.

---

### Interpreter — 작은 언어 해석

**의도**: 도메인 전용 미니 언어의 문법·해석기 구현
**Python(주의: 실전은 안전 파서 필요)**
```python
def calc(expr: str) -> int:
    return eval(expr)  # 실제 서비스엔 미사용! 학습용 데모
```

---

## 비교표(핵심 차이 빠르게 보기)

| 구분 | Factory Method | Abstract Factory | Builder | Strategy | Decorator | Facade |
|---|---|---|---|---|---|---|
| 목적 | 단일 제품 생성 확장 | 제품군 일관 생성 | 복잡 생성 단계 분리 | 알고리즘 교체 | 기능 동적 조합 | 서브시스템 단순화 |
| 변화 축 | 타입 | 제품군 | 생성 단계/옵션 | 알고리즘 | 횡단 관심사 | API 표면 |
| 복잡도 | 중 | 높음 | 중~높음 | 낮음 | 중 | 낮음 |

---

## 테스트 전략 요약

- **Factory/Abstract Factory**: 새 구현 추가 시 **계약 테스트** 재사용
- **Strategy/State/Command**: 동일 입력→일관 결과, Undo/Redo 가드
- **Decorator**: **기저 동작 보존** + 부가효과 검증(스파이/프록시)
- **Observer**: 구독/해지, 순서 보장, 누수 여부
- **Proxy**: 원격/지연 실패 경로, 타임아웃
- **Composite**: 리프/컴포지트 교환성(다형성) 확인

---

## 리팩토링 레시피(냄새 → 패턴)

- 거대한 생성 `switch` → **Factory Method**
- 생성자 매개변수 과다/순서 의존 → **Builder**
- 레거시 API 호환 요구 → **Adapter**
- 공용 초기화 코드 난립 → **Facade**
- 기능 추가·조합 폭발 → **Decorator**
- 상태 분기 폭발 → **State**
- 로그/권한/캐시 등 횡단 관심사 → **Decorator/Proxy**
- 연속 필터/검증 → **Chain of Responsibility**

---

## 성능·동시성·보안 노트

- **Flyweight**: 공유 캐시의 수명·경합·메모리 상한
- **Proxy**: 원격 호출 재시도/백오프/서킷브레이커(Decorator로 결합)
- **Observer**: 백프레셔/버퍼링 정책
- **Builder**: 대규모 객체 생성 시 불필요한 중간물 최소화
- **Memento**: 스냅샷 민감 정보 암호화

---

## 실전 시나리오 3개

### API 클라이언트 SDK

- 설계: **Facade(ApiClient)** + **Decorator(로깅/리트라이/캐시)** + **Strategy(서명/인증)** + **Factory Method(전송층 선택)**

### 데이터 수집 파이프라인

- 설계: **Factory Method(리더 선택)** + **Template Method(파이프라인 골격)** + **Chain(필터)** + **Observer(이벤트)**

### 에디터 Undo/Redo

- 설계: **Command(명령 캡슐화)** + **Memento(스냅샷)** + **Composite(문서 트리)**

---

## 자주 하는 질문(FAQ)

- **항상 패턴을 써야 하나?** 아니오. 단순한 곳엔 단순한 해법이 최선.
- **여러 패턴을 섞어도 되나?** 가능. 단, **역할 경계**를 명확히 하고 테스트로 검증.
- **함수형에서도 의미가 있나?** Strategy=함수 값, Observer=스트림, Command=이벤트 등 개념은 그대로 적용.

---

## 연습 과제(스켈레톤 제공)

### 과제 A: 이미지 인코더 선택기(Factory Method)

```python
class Encoder:
    def encode(self, img, **opts): raise NotImplementedError
class Jpeg(Encoder): ...
class Png(Encoder): ...
class Webp(Encoder): ...

class Saver:
    def create_encoder(self, kind: str) -> Encoder:
        raise NotImplementedError
    def save(self, img, kind: str, **opts):
        enc = self.create_encoder(kind)
        return enc.encode(img, **opts)

class MySaver(Saver):
    def create_encoder(self, kind: str) -> Encoder:
        if kind == "jpeg": return Jpeg()
        if kind == "png": return Png()
        if kind == "webp": return Webp()
        raise ValueError("unknown")
```

### 과제 B: 전송 미들웨어 체인(Decorator + Chain)

```csharp
interface IHandler { Task<Response> HandleAsync(Request r); }
class CoreHandler : IHandler { public Task<Response> HandleAsync(Request r){ /* ... */ return Task.FromResult(new Response()); } }
class Logging : IHandler { private readonly IHandler _n; public Logging(IHandler n){_n=n;} public async Task<Response> HandleAsync(Request r){ Console.WriteLine("in"); return await _n.HandleAsync(r);} }
class Retry : IHandler { /* ... 재시도 후 _n 호출 ... */ }
```

---

## 패턴별 미니 체크리스트

- **Factory Method**: 새 타입 추가 시 기존 코드 수정 최소?
- **Abstract Factory**: 제품군 일관성이 중요한가?
- **Builder**: 생성 단계·옵션이 복잡한가?
- **Adapter**: 외부/레거시 인터페이스 호환이 필요한가?
- **Decorator**: 횡단 관심사를 조합/분리하고 싶은가?
- **Observer**: 비동기 이벤트 전파가 필요한가?
- **State**: 상태 분기가 복잡한가?

---

## 보완: 23개 전 패턴 한 줄 요약과 예

- **Singleton**: 전역 구성·캐시
- **Factory Method**: 타입별 객체 생성
- **Abstract Factory**: 테마별 위젯 세트
- **Builder**: 문서/쿼리 조립
- **Prototype**: 템플릿 복제
- **Adapter**: 레거시 API ↔ 신규 인터페이스
- **Bridge**: 도형 ↔ 렌더러 분리
- **Composite**: 파일시스템 트리
- **Decorator**: 스트림 필터 체인
- **Facade**: 복잡 API의 간단한 포털
- **Flyweight**: 글리프 공유
- **Proxy**: 가상/원격/보호 대리
- **Observer**: 이벤트 버스
- **Strategy**: 압축/정렬 교체
- **Command**: Undo 가능한 명령
- **State**: 주문/세션 상태 흐름
- **Template Method**: 파이프라인 골격
- **Iterator**: 컬렉션 순회
- **Mediator**: 채팅/폼 중재
- **Chain of Responsibility**: 필터/검증 체인
- **Visitor**: AST에 새 연산
- **Memento**: 스냅샷/복원
- **Interpreter**: 간이 DSL 해석

---

## 마무리

**GoF 디자인 패턴**은 “코드 트릭”이 아닌 **협력 구조의 어휘**다.
현재 문제의 본질을 먼저 명확히 하고, **가장 단순한 해법**으로 풀 수 없다면 그때 패턴을 선택하라. 패턴 간 조합은 강력하지만, **역할 경계·테스트 가능성·수명/동시성·보안**을 항상 함께 고려해야 한다.
