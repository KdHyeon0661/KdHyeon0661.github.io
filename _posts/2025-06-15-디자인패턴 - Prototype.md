---
layout: post
title: 디자인패턴 - Prototype
date: 2025-06-15 21:20:23 +0900
category: 디자인패턴
---
# Prototype (프로토타입 패턴)

## ✅ 정의

**프로토타입 패턴(Prototype Pattern)**은 **기존 객체를 복제(clone)**하여 새로운 객체를 생성하는 **생성 패턴(Creational Pattern)**입니다.  
즉, **객체를 새로 생성하는 것이 아니라, 기존 객체를 복사하여 새로운 인스턴스를 만드는 방식**입니다.

이 패턴은 복잡한 객체의 생성 비용이 크거나, 객체 간의 차이가 적고 대부분 비슷한 구조를 가진 경우에 유용합니다.

---

## 🎯 의도 (Intent)

- **객체를 복제하여 생성**하고, 생성 비용을 줄이거나 생성 과정을 단순화한다.
- **기존 인스턴스를 복사(clone)**하여 새로운 인스턴스를 쉽게 만든다.
- 생성할 객체의 구조나 구체적인 타입을 클라이언트가 몰라도 된다.

---

## 📦 구조 (Structure)

```
        ┌────────────────────────┐
        │   Prototype (인터페이스)│
        └────────────────────────┘
                  ▲
                  │
        ┌────────────────────────┐
        │ ConcretePrototype      │
        └────────────────────────┘
                  │
                  ▼
           ┌──────────────┐
           │ Client       │
           └──────────────┘
```

- `Prototype`: 복제 기능을 가진 인터페이스 또는 추상 클래스 (`clone()` 메서드)
- `ConcretePrototype`: 복제 메서드를 구현한 클래스
- `Client`: 복제된 인스턴스를 사용하는 코드

---

## 🧑‍💻 구현 예시 (Python)

```python
import copy

# 프로토타입 인터페이스
class Shape:
    def clone(self):
        return copy.deepcopy(self)

# 구체 프로토타입
class Circle(Shape):
    def __init__(self, radius, color):
        self.radius = radius
        self.color = color

    def __str__(self):
        return f"Circle(radius={self.radius}, color='{self.color}')"

class Rectangle(Shape):
    def __init__(self, width, height, color):
        self.width = width
        self.height = height
        self.color = color

    def __str__(self):
        return f"Rectangle({self.width}x{self.height}, color='{self.color}')"

# 클라이언트 코드
circle1 = Circle(10, "red")
circle2 = circle1.clone()
circle2.color = "blue"

print(circle1)  # Circle(radius=10, color='red')
print(circle2)  # Circle(radius=10, color='blue')
```

---

## ✅ 장점

- 복잡한 객체를 **쉽게 복제** 가능
- 생성 비용이 높은 객체를 효율적으로 재사용
- **런타임에 동적으로 객체 생성 가능**
- 객체의 내부 상태를 보존한 채 복사 가능

---

## ⚠️ 단점

- **객체 간 깊은 복사(Deep Copy)**가 필요할 경우 구현이 복잡해질 수 있음
- 복사 과정에서 참조 타입(리스트, 딕셔너리 등)의 처리 주의 필요
- `clone()` 구현을 모든 클래스에 반복해야 할 수 있음

---

## 📌 사용 사례

| 사용 사례               | 설명 |
|------------------------|------|
| 게임 캐릭터 템플릿     | 특정 능력치를 가진 캐릭터를 여러 개 생성 |
| UI 컴포넌트 복사       | 기존 컴포넌트를 그대로 복제하여 재사용 |
| DB 레코드 복제         | 기존 레코드를 기반으로 새로운 엔티티 생성 |
| 프로토타이핑 도구      | 도형, 박스, 그래픽 요소 등 반복 생성 |
| 설정 객체 복제         | 복사 후 일부 속성만 변경하여 사용 |

---

## 🧠 Prototype vs new vs Factory

| 항목            | new 연산자     | Factory 패턴         | Prototype 패턴        |
|------------------|----------------|------------------------|------------------------|
| 객체 생성 방식   | 직접 생성      | 서브클래스에서 생성 위임 | 기존 객체에서 복사     |
| 비용             | 일반적         | 중간                   | 가장 낮음              |
| 유연성           | 낮음           | 중간                   | 매우 높음              |
| 런타임 확장성    | 낮음           | 중간                   | 높음                   |

---

## 🧠 마무리

**프로토타입 패턴**은 생성 비용이 높거나 복잡한 객체를 **쉽고 빠르게 복사**할 수 있는 효과적인 방법입니다.  
또한, 동적으로 객체를 생성하거나 다양한 객체를 유사한 방식으로 처리해야 할 때 유용합니다.

단, **깊은 복사와 얕은 복사의 차이**를 명확히 이해하고 상황에 맞게 사용하는 것이 중요합니다.
