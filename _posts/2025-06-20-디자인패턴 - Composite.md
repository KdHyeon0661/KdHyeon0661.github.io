---
layout: post
title: 디자인패턴 - Composite
date: 2025-06-20 20:20:23 +0900
category: 디자인패턴
---
# Composite (컴포지트 패턴)

## ✅ 정의

**컴포지트 패턴(Composite Pattern)**은 객체들을 **트리 구조로 구성**하여,  
부분-전체 계층 구조(whole-part hierarchy)를 동일하게 다룰 수 있게 해주는 **구조 패턴(Structural Pattern)**입니다.

즉, 단일 객체와 복합 객체를 **동일한 방식으로 처리**할 수 있도록 구성하여, 클라이언트 코드가 객체의 구조를 알 필요 없이 일관된 방식으로 조작할 수 있게 합니다.

> “부분과 전체를 동일하게 다룬다”

---

## 🎯 의도 (Intent)

- 객체들의 계층 구조를 표현 (트리 구조)
- 단일 객체와 복합 객체를 **동일한 인터페이스로 다루기**
- 재귀적으로 구성된 구조를 효과적으로 처리

---

## 📦 구조 (UML)

```
             ┌─────────────────────┐
             │     Component       │
             └─────────────────────┘
                 ▲           ▲
                 │           │
     ┌─────────────────┐   ┌──────────────────┐
     │     Leaf        │   │   Composite       │
     └─────────────────┘   └──────────────────┘
                                │
                  ┌────────────┴────────────┐
                  ▼                         ▼
         ┌─────────────┐         ┌────────────────┐
         │    Leaf     │         │   Composite    │
         └─────────────┘         └────────────────┘
```

- `Component`: 공통 인터페이스 (Leaf와 Composite가 구현)
- `Leaf`: 실제 객체 (말단 노드)
- `Composite`: 자식 객체들을 포함하는 복합 객체 (내부 노드)

---

## 🧑‍💻 구현 예시 (Python)

```python
from abc import ABC, abstractmethod

# 공통 인터페이스
class Graphic(ABC):
    @abstractmethod
    def draw(self):
        pass

# Leaf 클래스
class Line(Graphic):
    def draw(self):
        print("선을 그림")

class Circle(Graphic):
    def draw(self):
        print("원을 그림")

# Composite 클래스
class Picture(Graphic):
    def __init__(self):
        self.children = []

    def add(self, graphic: Graphic):
        self.children.append(graphic)

    def remove(self, graphic: Graphic):
        self.children.remove(graphic)

    def draw(self):
        print("[그림 시작]")
        for child in self.children:
            child.draw()
        print("[그림 끝]")

# 사용 예
circle = Circle()
line = Line()
picture = Picture()
picture.add(circle)
picture.add(line)

picture.draw()
# 출력:
# [그림 시작]
# 원을 그림
# 선을 그림
# [그림 끝]
```

---

## ✅ 장점

- **재귀적 구조**(트리 구조)를 쉽게 표현 가능
- 클라이언트가 복합 객체와 단일 객체를 **동일하게 처리**할 수 있음
- 구조가 동적으로 변경 가능한 유연한 설계
- 새로운 타입의 노드를 추가하더라도 기존 코드 수정이 거의 필요 없음

---

## ⚠️ 단점

- **설계가 복잡해질 수 있음** (특히 Composite 내부에 여러 타입의 자식이 있는 경우)
- 전체 구조를 명확하게 이해하기 어려워질 수 있음
- Leaf와 Composite의 구분이 흐려져 관리가 어려울 수 있음

---

## 📌 사용 사례

| 사용 사례 | 설명 |
|-----------|------|
| **그래픽 편집기** | 도형 그룹(Composite)과 단일 도형(Leaf)을 동일하게 처리 |
| **파일 시스템 구조** | 폴더(Composite)와 파일(Leaf)을 트리 구조로 관리 |
| **UI 트리** | 버튼, 텍스트, 컨테이너 등을 계층적으로 구성 |
| **조직도/회사 구조** | 부서(Composite)와 사원(Leaf)을 동일 인터페이스로 관리 |
| **문서 요소 처리기** | 문단(Composite), 텍스트(Leaf) 등 문서 구조 표현 |

---

## 🧠 Composite vs Decorator vs Bridge

| 패턴       | 목적 | 구조 | 사용 시점 |
|------------|------|------|-----------|
| **Composite** | 트리 구조 표현 | 재귀적 트리 | 구조를 나타낼 때 |
| **Decorator** | 기능 확장 | 중첩 구조 | 런타임 기능 추가 |
| **Bridge** | 추상화/구현 분리 | 두 계층 분리 | 구현과 기능 분리 시 |

---

## 🧠 마무리

**컴포지트 패턴**은 **복합 객체와 단일 객체를 일관되게 처리**하고 싶을 때 매우 유용한 구조 패턴입니다.  
트리 구조로 구성되는 시스템(파일 시스템, 그래픽, UI, 조직도 등)에서 자주 사용되며,  
복잡한 계층 구조를 클라이언트가 신경 쓰지 않고 **동일한 방식으로 조작 가능**하게 해줍니다.

단순한 구조에서는 과도한 설계가 될 수 있으므로, **재귀적 계층이 존재할 때 전략적으로 도입**하는 것이 좋습니다.
