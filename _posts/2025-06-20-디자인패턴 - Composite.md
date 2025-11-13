---
layout: post
title: 디자인패턴 - Composite
date: 2025-06-20 20:20:23 +0900
category: 디자인패턴
---
# Composite(컴포지트 패턴)

## 정의

**컴포지트 패턴(Composite Pattern)**은 객체들을 **트리 구조로 구성**하여, 부분-전체 계층(whole–part hierarchy)을 **동일한 인터페이스**로 다루도록 하는 **구조 패턴**이다.
클라이언트는 개별 객체(Leaf)와 복합 객체(Composite)를 **구분하지 않고** 동일한 연산으로 조작한다.

> “부분과 전체를 동일하게 다룬다.”

---

## 의도 (Intent)

- 재귀적 **계층 구조(트리)**를 표현한다.
- **Leaf**와 **Composite**를 동일 인터페이스(**Component**)로 다룬다.
- 구조/자식 관리, 순회/집계 로직을 **일관화**한다.

---

## 구조 (UML)

```
             ┌─────────────────────┐
             │     Component       │  ← 공통 인터페이스(연산/순회/집계)
             └─────────────────────┘
                 ▲           ▲
                 │           │
     ┌─────────────────┐   ┌──────────────────┐
     │     Leaf        │   │    Composite      │
     └─────────────────┘   └──────────────────┘
                                │  children: List<Component>
                  ┌─────────────┴─────────────┐
                  ▼                           ▼
         ┌─────────────┐              ┌────────────────┐
         │    Leaf     │              │    Composite    │
         └─────────────┘              └────────────────┘
```

- **Component**: 공통 연산(예: `draw()`, `size()`, `evaluate()`, `accept(visitor)`).
- **Leaf**: 자식이 없는 말단 노드(실제 업무 로직 보유).
- **Composite**: 여러 자식을 보관하고 재귀적으로 연산을 위임/집계.

---

## 기본 구현 예시 (Python) — 그래픽 씬 트리

```python
from abc import ABC, abstractmethod
from typing import List, Iterable

# 공통 인터페이스
class Graphic(ABC):
    @abstractmethod
    def draw(self) -> None:
        ...

    # 선택: 공통 순회(DFS) 제공
    def iter(self) -> Iterable["Graphic"]:
        yield self  # 기본: 자신만

# Leaf
class Line(Graphic):
    def __init__(self, x1, y1, x2, y2):
        self.x1, self.y1, self.x2, self.y2 = x1, y1, x2, y2

    def draw(self) -> None:
        print(f"Line(({self.x1},{self.y1})→({self.x2},{self.y2}))")

class Circle(Graphic):
    def __init__(self, cx, cy, r):
        self.cx, self.cy, self.r = cx, cy, r

    def draw(self) -> None:
        print(f"Circle(center=({self.cx},{self.cy}), r={self.r})")

# Composite
class Picture(Graphic):
    def __init__(self):
        self._children: List[Graphic] = []

    def add(self, g: Graphic) -> None:
        self._children.append(g)

    def remove(self, g: Graphic) -> None:
        self._children.remove(g)

    def draw(self) -> None:
        print("[Picture begin]")
        for c in self._children:
            c.draw()
        print("[Picture end]")

    def iter(self) -> Iterable[Graphic]:
        yield self
        for c in self._children:
            yield from c.iter()

# 사용
scene = Picture()
scene.add(Circle(0, 0, 10))
scene.add(Line(0, 0, 5, 5))
scene.draw()
# [Picture begin] ... [Picture end]

# 트리 전체 순회
for node in scene.iter():
    pass  # 로그/검색/집계에 활용
```

---

## 설계 변형: “투명(Transparent) vs 안전(Safe) 컴포지트”

- **투명 컴포지트**: `Component`에 `add/remove` 포함 → Leaf도 노출하지만 Leaf에서는 예외 발생.
  장점: 클라이언트 코드가 **완전히 동일한 인터페이스** 사용.
  단점: Leaf가 **의미 없는 메서드**를 가진다.

- **안전 컴포지트**: `add/remove`는 **Composite에만** 둔다.
  장점: 타입 안전, 의미 명확.
  단점: 클라이언트가 **Composite 타입**을 알아야 자식 관리 가능.

실무 팁:
- **읽기 연산(업무 연산)**은 `Component`에,
- **구조 변경(자식 관리)**은 `Composite`에 두는 **혼합 전략**이 가장 흔하다.

---

## 순회(Traversal)·집계(Fold) 패턴

트리 연산의 다수는 **“자식들의 결과를 결합(집계)”**하는 형태다.
예: 도형 개수, 총 길이/면적, 파일 크기 합, UI의 레이아웃 측정 등.

- 전체 노드 수를 \(N\)이라 하면, 순수 순회/집계의 시간 복잡도는
  $$ T_{\text{fold}} = \Theta(N) $$

일반적인 **폴드 템플릿**(Python):

```python
from typing import Callable, TypeVar
T = TypeVar("T")

class Component(ABC):
    @abstractmethod
    def children(self) -> Iterable["Component"]:
        ...

    def fold(self, f: Callable[[T, "Component"], T], init: T) -> T:
        acc = f(init, self)
        for ch in self.children():
            acc = ch.fold(f, acc)
        return acc
```

이 폴드 기반으로 “개수 세기”, “바운딩 박스 합치기”, “속성 검색” 등을 쉽게 구현 가능.

---

## 예: 파일 시스템 크기 합산 (Composite로 모델링)

```python
from abc import ABC, abstractmethod
from typing import List

class Node(ABC):
    @abstractmethod
    def size(self) -> int: ...
    def children(self): return []

class File(Node):
    def __init__(self, name: str, bytes_: int):
        self.name, self._bytes = name, bytes_
    def size(self) -> int: return self._bytes

class Dir(Node):
    def __init__(self, name: str):
        self.name, self._children = name, []
    def add(self, n: Node): self._children.append(n)
    def children(self): return self._children
    def size(self) -> int:  # 집계
        return sum(ch.size() for ch in self._children)

# 테스트
root = Dir("root")
root.add(File("a.txt", 10))
img = Dir("img"); img.add(File("logo.png", 120))
root.add(img)
assert root.size() == 130
```

복잡도:
- 노드 수 \(N\), 엣지 수 \(E\)에 대해, 한 번의 총합 계산은
  $$ T_{\text{size}} = \Theta(N + E) \approx \Theta(N) $$

---

## 캐싱과 무효화(Invalidation)

큰 트리에서 집계를 **매 호출마다** 재귀하면 비싸다.
**Composite**에 캐시를 두고, 구조 변경 시 **무효화**하면 성능을 크게 개선할 수 있다.

```python
class CachedDir(Dir):
    def __init__(self, name):
        super().__init__(name)
        self._cached_size = None

    def add(self, n: Node):
        super().add(n)
        self._cached_size = None  # 무효화

    def size(self) -> int:
        if self._cached_size is None:
            self._cached_size = sum(ch.size() for ch in self.children())
        return self._cached_size
```

주의:
- 삽입/삭제 시 **상위로 전파**하여 캐시 무효화가 필요(부모 참조 or 이벤트).
- 병렬 구조에서는 **락/원자성** 고려.

---

## 스레드 안전성 / 불변 트리

- **가변 트리 + 캐시**: 락(읽기-쓰기 락) 또는 원자적 교체가 필요.
- **불변 트리(권장)**: 자식 교체 시 **새 Composite**를 생성(퍼시스턴트 구조).
  - 장점: 스레드 안전·스냅샷 용이, 테스트 쉬움.
  - 단점: 대형 트리에서는 쓰기 비용 증가(구조 공유로 완화 가능).

---

## 순환 방지

컴포지트는 **트리**가 전제이므로 **사이클**이 없어야 한다.
- `Composite.add(child)` 시 **자기 자신/조상** 추가 방지 체크.
- 순환이 허용되면 **그래프 패턴(Visitor with memo)**으로 전환해야 한다.

간단 체크:

```python
def add_child_safe(parent, child):
    # child가 parent의 조상인지 검사
    for n in parent.iter():  # parent의 모든 하위 탐색
        if n is child:
            raise ValueError("cycle detected")
    parent.add(child)
```

---

## 테스트 전략

1) **계약 테스트**: Component가 제공하는 API(`draw`, `size`, `iter`)가 Leaf/Composite 모두에서 동일하게 동작하는지.
2) **구조 변경**: `add/remove` 수행 후 집계/순회가 일관적인지, 캐시 무효화가 제대로 되는지.
3) **엣지 케이스**: 빈 Composite, 깊은 트리, 매우 넓은 트리(스택/메모리), 중복 제거, 순환 방지.
4) **성능 가드**: 대형 트리에서 집계 시간/메모리 상한의 회귀 테스트.

간단 PyTest 예:

```python
def test_composite_size_and_iter():
    root = Picture()
    c1, c2 = Circle(0,0,1), Circle(1,1,2)
    root.add(c1); root.add(c2)
    nodes = list(root.iter())
    assert len(nodes) == 3  # root 포함
    # draw() 호출 smoke 테스트
    root.draw()
```

---

## Composite의 변형과 인터페이스 설계

- **Component에 어떤 연산을 둘 것인가?**
  - 꼭 공통인 것(“그리기/측정/이벤트 처리”)만 둔다.
  - 구조 변경(자식 관리)은 Composite로 분리(안전 컴포지트).

- **반복자 제공 위치**
  - `Component.iter()`에 DFS 기본 제공(간결).
  - 또는 외부 유틸로 순회자 제공(관심사 분리).

- **Visitor와의 결합**
  - 새로운 연산을 구조 수정 없이 추가하려면 **Visitor**가 유리(행위 추가/확장).

---

## Visitor 연동 스케치 (옵션)

```python
class Visitor(ABC):
    @abstractmethod
    def visit_line(self, line: Line): ...
    @abstractmethod
    def visit_circle(self, circle: Circle): ...
    @abstractmethod
    def visit_picture(self, pic: "Picture"): ...

class Visitable(ABC):
    @abstractmethod
    def accept(self, v: Visitor): ...

class Line(Graphic, Visitable):
    # ...
    def accept(self, v: Visitor): v.visit_line(self)

class Circle(Graphic, Visitable):
    def accept(self, v: Visitor): v.visit_circle(self)

class Picture(Graphic, Visitable):
    # ...
    def accept(self, v: Visitor):
        v.visit_picture(self)
        for ch in self._children:
            ch.accept(v)
```

Visitor로 “바운딩 박스 계산/히트 테스트/시리얼라이즈” 등 **행위 추가**가 쉬워진다.

---

## 실제 시나리오 1 — UI 트리(측정/배치/그리기)

UI는 대개 **Composite**이다. 각 위젯(Leaf/Composite)은 `measure()`로 **자연 크기**, `arrange(rect)`로 **배치**, `render()`로 **그리기**를 수행한다.

```python
class Widget(ABC):
    @abstractmethod
    def measure(self) -> tuple[int, int]: ...
    @abstractmethod
    def arrange(self, x:int, y:int, w:int, h:int) -> None: ...
    @abstractmethod
    def render(self) -> None: ...

class Text(Widget):
    def __init__(self, s: str): self.s = s; self.bounds=(0,0,0,0)
    def measure(self): return (len(self.s)*8, 16)
    def arrange(self, x,y,w,h): self.bounds = (x,y,w,h)
    def render(self): print(f"Text '{self.s}' at {self.bounds}")

class StackPanel(Widget):
    def __init__(self): self.children: list[Widget] = []
    def add(self, w: Widget): self.children.append(w)
    def measure(self):
        w = h = 0
        for c in self.children:
            cw, ch = c.measure()
            w = max(w, cw); h += ch
        return (w, h)
    def arrange(self, x,y,w,h):
        cy = y
        for c in self.children:
            cw, ch = c.measure()
            c.arrange(x, cy, w, ch)
            cy += ch
    def render(self):
        for c in self.children: c.render()

root = StackPanel()
root.add(Text("Hello"))
root.add(Text("Composite"))
W,H = root.measure()
root.arrange(0,0,W,H)
root.render()
```

핵심: **Leaf/Composite 동일 인터페이스**로 **재귀적 측정/배치/렌더링**을 구현.

---

## 실제 시나리오 2 — 수식/IR 트리(폴드로 평가)

수식(또는 DSL/쿼리)은 전형적인 Composite. 폴드로 평가/단순화/코드 생성 가능.

```python
class Expr(ABC):
    @abstractmethod
    def eval(self) -> float: ...

class Lit(Expr):
    def __init__(self, v: float): self.v = v
    def eval(self) -> float: return self.v

class Add(Expr):
    def __init__(self, l: Expr, r: Expr): self.l, self.r = l, r
    def eval(self) -> float: return self.l.eval() + self.r.eval()

class Mul(Expr):
    def __init__(self, l: Expr, r: Expr): self.l, self.r = l, r
    def eval(self) -> float: return self.l.eval() * self.r.eval()

# (3+4)*5 = 35
expr = Mul(Add(Lit(3), Lit(4)), Lit(5))
assert expr.eval() == 35.0
```

---

##  성능·복잡도 분석

- 트리 순회/집계의 기본 비용은 노드 수 \(N\)에 대해
  $$ T = \Theta(N) $$
- **캐시** 도입 시: 조회 \(O(1)\) 가까이 단축, 단 **구조 변경시 무효화 비용** 발생.
- **깊이 제한**: 재귀 깊이가 큰 트리는 **스택 오버플로** 위험 → 반복/스택 사용 or TCO.
- **메모리**: 포인터/참조 오버헤드를 고려, **Flyweight**와의 결합으로 불변 공유 최적화 가능.

---

## 함정·안티패턴

- **무의미한 균일 인터페이스**: Leaf에 불필요 메서드 강요(투명 컴포지트 남용).
- **순환 참조 허용**: 트리가 그래프로 변질 → 순회 무한루프/중복 계산.
- **거대 Composite**: 자식 관리/추가 행위가 비대해지면 **Visitor/전략**으로 행위 분리.
- **캐시와 일관성**: 무효화 누락으로 **오염된 값** 유지.

---

## 🧠 Composite vs Decorator vs Bridge (확장 표)

| 패턴        | 목적                      | 구조                      | 사용 시점                                   | 핵심 포인트                         |
|-------------|---------------------------|---------------------------|---------------------------------------------|-------------------------------------|
| Composite   | 트리(부분-전체) 표현      | 재귀적 트리               | 트리 기반 집계/순회/구조 변경               | Leaf/Composite를 동일 인터페이스   |
| Decorator   | 기능의 동적 추가          | 중첩(래핑)                | 런타임에 횡단 관심(로깅/캐싱/검사) 부착      | 조합으로 클래스 폭발 억제          |
| Bridge      | 추상/구현 독립적 확장     | 2계층(기능×구현)          | 기능×구현 조합 폭발 억제                    | 합성으로 런타임 교체/확장          |

---

## 🤝 다른 패턴과의 조합

- **Composite + Visitor**: 트리 구조는 고정, **새 연산은 Visitor**로 확장.
- **Composite + Flyweight**: 반복되는 불변 서브트리(폰트 글리프/메시) 공유로 메모리 절감.
- **Composite + Builder**: 복잡한 트리 구성을 **Builder**로 단계적 생성.
- **Composite + Prototype**: 서브트리를 빠르게 복제하여 템플릿 기반 편집.
- **Composite + Facade**: 트리 최상위에 단순 API를 제공.

---

## 🧩 C# 제너릭 버전(간단 스케치)

```csharp
public interface INode<T>
{
    IEnumerable<INode<T>> Children { get; }
    T Evaluate(); // 혹은 void Draw(), long Size() 등 도메인별
}

public sealed class Leaf<T> : INode<T>
{
    private readonly T _value;
    public Leaf(T value) => _value = value;
    public IEnumerable<INode<T>> Children => Array.Empty<INode<T>>();
    public T Evaluate() => _value;
}

public sealed class Composite<T> : INode<T>
{
    private readonly List<INode<T>> _children = new();
    private readonly Func<IEnumerable<T>, T> _aggregate;
    public Composite(Func<IEnumerable<T>, T> aggregate) => _aggregate = aggregate;
    public IEnumerable<INode<T>> Children => _children;
    public void Add(INode<T> c) => _children.Add(c);
    public T Evaluate() => _aggregate(_children.Select(c => c.Evaluate()));
}
```

---

## 체크리스트(요약)

- 트리인가? **사이클 없음** 보장하는가?
- **인터페이스 최소화**: 공통 연산만 `Component`에.
- **자식 관리**는 Composite에(안전), 필요 시 예외 기반(투명) 혼합.
- **순회/집계**는 폴드로 일관화.
- **캐시** 사용 시 무효화/부모 전파 설계.
- **불변/락**으로 스레드 안전성 고려.
- **테스트 스위트**(계약/성능/엣지) 준비.

---

## 마무리

**Composite**는 트리 구조를 **표현·조작·집계**하는 표준 해법이다.
핵심은 **Leaf/Composite의 균일한 계약**과 **재귀적 위임/집계**이며,
현실 문제(파일 시스템, UI 레이아웃, 씬 그래프, 수식/쿼리 IR)에 그대로 대응한다.

과설계를 경계하되, **순회/폴드 템플릿·캐시 무효화·순환 방지·스레드 안전**을 의식하면
대형 트리에서도 성능·유지보수성을 동시에 달성할 수 있다.
필요 시 **Visitor/Builder/Flyweight/Prototype**과 조합해 표현력과 효율을 끌어올려라.
