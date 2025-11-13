---
layout: post
title: 디자인패턴 - Flyweight
date: 2025-06-23 19:20:23 +0900
category: 디자인패턴
---
# Flyweight (플라이웨이트 패턴)

## 정의

**플라이웨이트 패턴(Flyweight Pattern)**은 **많은 수의 유사한 객체**를 **공유(내부 상태 Intrinsic state)하여 메모리를 줄이는** **구조 패턴**이다.
공유 가능한 **불변 객체**를 만들고, 매번 달라지는 **외부 상태(Extrinsic state)**는 호출 시점에 전달한다.

> “**같은 것은 공유**하고, **다른 것만 외부에서 공급**하자.”

---

## 의도 (Intent)

- 대량 객체의 **중복 데이터를 공유**해 메모리 사용량을 최소화한다.
- 객체를 **내부 상태(공유/불변)**와 **외부 상태(매 호출 인자)**로 분리한다.
- **생성 비용/GC 부담**을 낮추고, **캐시 적중**을 통해 성능을 높인다.

---

## 구조 (UML)

사용자 제공 UML을 **보존**한다.

```
┌────────────┐
│   Client   │
└────┬───────┘
     │  (외부 상태 전달: 위치, 문맥 등)
     ▼
┌────────────┐       uses       ┌────────────────┐
│ Flyweight  │◄────────────────│ FlyweightFactory│
└────────────┘                 └────────────────┘
     ▲
     │  (공유되는 내부 상태: 글꼴, 색, 질감 등)
┌────────────┐
│ConcreteFly │
└────────────┘
```

- **Flyweight**: 공유 객체의 인터페이스(=동작).
- **ConcreteFlyweight**: **불변** 내부 상태를 가진 실제 공유 객체.
- **FlyweightFactory**: 키(내부 상태)로 객체를 **캐싱·재사용**.
- **Client**: 매번 바뀌는 **외부 상태**(좌표, 크기, 컨텍스트 등)를 전달.

---

## 핵심 개념

- **내부 상태(Intrinsic)**: 공유 가능한 불변 속성(예: 글리프의 윤곽, 타일 텍스처, 색 팔레트).
- **외부 상태(Extrinsic)**: 호출 때마다 달라지는 값(예: (x,y)좌표, 회전, 스케일, 문맥).
- **불변성(Immutability)**: 공유 객체는 **절대 변경하지 않는다** → 스레드-세이프 공유 가능.
- **캐시 키(Canonicalization)**: 내부 상태를 **정규화된 키**로 매핑하여 **동일한 인스턴스**를 돌려준다.

---

## 구현 예시 1 — Python(사용자 예시 보강): 숲 렌더링

> 예시를 살리되, **타입힌트/동등성/식별성 테스트/캐시 통계**를 추가했다.

```python
from __future__ import annotations
from dataclasses import dataclass
from typing import Dict, Tuple

# Flyweight (내부 상태: 이름/색상/텍스처) — 불변 객체
@dataclass(frozen=True)
class TreeType:
    name: str
    color: str
    texture: str

    def draw(self, x: int, y: int) -> None:   # 외부 상태는 호출 인자로 전달
        print(f"[{self.name}] ({x},{y}) 색:{self.color}, 텍스처:{self.texture}")

# Factory (캐시)
class TreeFactory:
    _cache: Dict[Tuple[str, str, str], TreeType] = {}

    @classmethod
    def get_tree_type(cls, name: str, color: str, texture: str) -> TreeType:
        key = (name, color, texture)
        if key not in cls._cache:
            cls._cache[key] = TreeType(*key)
        return cls._cache[key]

    @classmethod
    def stats(cls) -> int:
        return len(cls._cache)

# Client (외부 상태 보유)
@dataclass
class Tree:
    x: int
    y: int
    type: TreeType

    def draw(self) -> None:
        self.type.draw(self.x, self.y)

# 사용
forest: list[Tree] = []
for i in range(0, 10_000, 1_000):
    forest.append(Tree(i, i,   TreeFactory.get_tree_type("자작나무", "초록",   "거칠게")))
    forest.append(Tree(i+5, i, TreeFactory.get_tree_type("소나무",  "진초록", "가늘게")))

# 확인
forest[0].draw()
forest[1].draw()
print("캐시된 TreeType 개수:", TreeFactory.stats())  # 2개
# 동일 TreeType 공유 여부(식별성)
assert forest[0].type is TreeFactory.get_tree_type("자작나무", "초록", "거칠게")
```

---

## 메모리 절감 효과 계산

내부 상태를 공유하면, 총 메모리 사용량은 대략:

\[
M_{\text{fly}} \;\approx\; N_{\text{ext}} \cdot S_{\text{ext}}
\;+\; K_{\text{int}} \cdot S_{\text{int}}
\]

기존(공유 없음):

\[
M_{\text{plain}} \;\approx\; N \cdot (S_{\text{ext}} + S_{\text{int}})
\]

여기서,
- \(N\): 전체 개체 수(예: 나무 10,000그루)
- \(N_{\text{ext}} \approx N\): 외부 상태 객체 수(좌표 등)
- \(K_{\text{int}}\): **서로 다른 내부 상태 조합의 수**(예: 나무 종류×색×텍스처)
- \(S_{\text{ext}}\)/\(S_{\text{int}}\): 외부/내부 상태의 평균 메모리 크기

**절감량**:

\[
\Delta M \;=\; M_{\text{plain}} - M_{\text{fly}}
\;\approx\; (N - K_{\text{int}})\cdot S_{\text{int}}
\]

즉, **내부 상태 크기**가 클수록, 그리고 **K_int ≪ N**일수록 이득이 커진다.

**예시 수치**
- \(N=10{,}000\), \(K_{\text{int}}=2\) (자작나무/소나무),
- \(S_{\text{int}}=4\,\text{KB}\), \(S_{\text{ext}}=16\,\text{B}\).

그렇다면,
- 공유 없음: \(10{,}000 \times (4\,\text{KB} + 16\,\text{B}) \approx 40\,\text{MB}\) **+** 외부 160KB
- 플라이웨이트: \(10{,}000 \times 16\,\text{B} \approx 160\,\text{KB}\) **+** \(2 \times 4\,\text{KB}=8\,\text{KB}\) ⇒ **168KB**
- **절감 비율**: 약 240배 이상

---

## 구현 예시 2 — 텍스트 렌더러(글리프 공유)

> **문자 ‘글리프(윤곽, 메트릭)’를 공유**하고, 위치/크기/색상은 외부 상태로.

```python
from dataclasses import dataclass
import weakref

@dataclass(frozen=True)
class Glyph:        # 내재 상태: 불변
    char: str
    font: str
    size_px: int
    # 윤곽/커닝/메트릭 등 (실제론 바이너리 데이터)
    metrics_id: int

class GlyphFactory:
    _cache = weakref.WeakValueDictionary()  # 메모리 압박 시 자동 파기 가능
    _seq = 0

    @classmethod
    def get(cls, char: str, font: str, size_px: int) -> Glyph:
        key = (char, font, size_px)
        g = cls._cache.get(key)
        if g is None:
            cls._seq += 1
            g = Glyph(char, font, size_px, metrics_id=cls._seq)
            cls._cache[key] = g
        return g

@dataclass
class PositionedGlyph:    # 외부 상태: 위치/색/회전 등
    glyph: Glyph
    x: int
    y: int
    color: str

def draw_text(text: str, x: int, y: int, font: str, size: int, color: str):
    cursor = x
    for ch in text:
        g = GlyphFactory.get(ch, font, size)
        # 렌더러에 (g, cursor, y, color) 전달
        print(f"draw '{g.char}'@({cursor},{y})/{color} metrics#{g.metrics_id}")
        cursor += size // 2  # 단순 가정

draw_text("HELLO", 100, 200, "NotoSans", 16, "#333")
```

**포인트**
- `Glyph`는 **불변**이므로 스레드 공유 가능.
- `WeakValueDictionary`로 **약한 참조 캐시** 구현(메모리 압박 시 해제 용이).
- 외부 상태는 **좌표/색상/회전/스케일** 등 렌더링 컨텍스트.

---

## 구현 예시 3 — C# 색 팔레트/이미지 타일

```csharp
using System.Collections.Concurrent;

public sealed record TileType(string TextureName, int TileSize); // 불변 레코드

public static class TileFactory
{
    private static readonly ConcurrentDictionary<TileType, TileType> Cache = new();

    public static TileType Get(string texture, int size)
        => Cache.GetOrAdd(new TileType(texture, size), t => t);

    public static int Count => Cache.Count;
}

public sealed class Tile
{
    public int X { get; }
    public int Y { get; }
    public TileType Type { get; }  // 공유(내부 상태)
    public Tile(int x, int y, TileType type) { X = x; Y = y; Type = type; }
}
```

- **ConcurrentDictionary**로 스레드-세이프 캐시.
- `record`(값동등)로 키 충돌을 방지하고 **정규화**된 인스턴스만 보관.

---

## 스레드 안전 & 캐시 전략

- **불변 Flyweight**: 읽기 전용이라 **공유 안전**.
- **Factory**는 **동시성 안전**해야 한다.
  - Python: `threading.Lock`, **double-checked** 접근 지양(파이썬 GIL과는 별개).
  - C#: `ConcurrentDictionary.GetOrAdd`.
- **Strong Cache** vs **Weak Cache**:
  - Strong: 재사용률↑, 메모리 사용 고정↗.
  - Weak: 메모리 압박 시 회수, 재생성 비용 존재.
- **Eviction 정책**: LRU/LFU가 필요하면 **Object Pool**과 혼동 금지(플라이웨이트는 **불변 공유**, 풀은 **가변 재사용**).

---

## 동등성/식별성 규약

- **Flyweight 인스턴스는 “내부 상태” 기준으로 동일**해야 한다.
  - Python: `@dataclass(frozen=True)` + 팩토리 **정규화**.
  - C#: `record`/`IEquatable<T>`.
- 클라이언트 외부 상태는 **인스턴스에 저장하지 않고** 호출 인자로 전달한다.
- `is`(식별) 테스트가 통과하도록 **캐시를 단일화**한다(동일 키 → 동일 인스턴스).

---

## 주의점(함정)

- **불변성 위반**: 공유 인스턴스를 수정하면 전체가 오염(절대 금지).
- **키 폭발/고유 조합 증가**: \(K_{\text{int}}\)가 커지면 캐시 이득↓ → 그룹화/정규화.
- **외부 상태 전달 비용**: 매 호출 많은 파라미터가 오히려 부담이면, 구조를 재검토.
- **풀/캐시 혼동**: 플라이웨이트는 **불변 공유**, 풀은 **가변 객체 재사용**. 목적과 정책이 다르다.

---

## 리팩토링 절차: “중복 큰 필드” → Flyweight

1) **프로파일링**으로 대량 객체의 **공통 필드** 파악(텍스처, 글꼴, 스타일).
2) 공통 필드를 **불변 클래스**로 추출(내부 상태).
3) **Factory** 도입: `(공통 속성)` → **정규화된 인스턴스** 반환.
4) 원래 객체는 외부 상태만 보유; 필요 시 호출 인자로 전달.
5) **동등성/식별성/스레드 안전** 테스트.
6) 메모리/지연 개선 측정 및 튜닝(Weak/Strong/Hybrid 캐시).

---

## 테스트 전략

- **정규화 테스트**: 같은 키 → 같은 인스턴스
  ```python
  a = TreeFactory.get_tree_type("자작나무","초록","거칠게")
  b = TreeFactory.get_tree_type("자작나무","초록","거칠게")
  assert a is b
  ```
- **불변성 테스트**: 속성 변경 불가(예외).
- **성능 테스트**: N 증가 시 메모리/지연 추세.
- **회수 테스트(Weak)**: 참조 제거 후 GC 시 캐시 해제 여부.

---

## 다른 패턴과의 조합

- **Composite + Flyweight**: 씬 그래프(Composite)의 **노드 수**는 많고, **메시/머티리얼**은 공유(Flyweight).
- **Prototype + Flyweight**: 큰 공통 구조는 Flyweight로, 개별 차이만 Prototype 복제.
- **Proxy + Flyweight**: Flyweight가 가리키는 리소스를 **지연 로딩**하는 프록시.
- **Factory Method/Abstract Factory**: Flyweight 인스턴스 생성을 **정규화된 팩토리**로 감춘다.

---

## 사용 사례(보강)

| 사례                         | 설명 |
|------------------------------|------|
| 게임의 타일/나무/건물        | 동일 텍스처·메시 공유, 좌표만 외부 상태 |
| 문자 렌더러(글꼴)            | 글리프 윤곽·메트릭 공유, 위치/색만 외부 |
| 아이콘/이모지 세트           | 비트맵/벡터 데이터 공유 |
| 코드 에디터 문법 강조        | 동일 토큰 스타일 공유 |
| 맵/지리 엔진                 | 타일 스펙 공유, 줌/오프셋은 외부 |
| 차트/시각화 마커             | 마커 도형 공유, 데이터 포인트 위치·색만 외부 |

---

## Flyweight vs Singleton vs Pool (사용자 표 유지·보강)

| 패턴           | 목적                 | 공유 방식            | 상태 | 전형 사용처 |
|----------------|----------------------|----------------------|------|-------------|
| **Flyweight**  | 메모리 절약          | **여러 개 공유**     | **불변** | 대량 유사 객체 |
| **Singleton**  | 유일 인스턴스        | 전역 1개             | 보통 가변/불변 | 시스템 단일 리소스 |
| **Object Pool**| 재사용(할당 절약)    | 대여/반납            | **가변** | 커넥션/버퍼 등 |

---

## 성능/복잡도 트레이드오프

- 메모리 절감은 대개 **지연/캐시 조회** 오버헤드보다 **이득**이 크다.
- 단, 외부 상태 전달이 커지면 호출 비용이 증가 → **데이터 전송 크기** 최적화.
- 캐시가 강하면 메모리 상주↑, 약하면 재생성 비용↑ → **워크로드 특성**으로 결정.

---

## 마무리

플라이웨이트는 **“공유 가능한 내부 상태를 불변으로 만들고, 외부 상태만 매 호출 전달”** 하는 설계다.
**대량의 유사 객체**에서 탁월한 메모리 절감과 성능 향상을 제공하지만, 그 대가로 **키 설계/정규화/스레드 안전/테스트**가 필요하다.
Composite·Factory·Proxy 등과 함께 쓰면 **대규모 렌더링/문자/지도/IDE** 같은 시스템의 핵심 최적화 축이 된다.
도입 전후로 **프로파일링과 계약 테스트**로 검증하고, 캐시(Strong/Weak)와 외부 상태 인터페이스를 **업무 부하에 맞게** 다듬어라.
