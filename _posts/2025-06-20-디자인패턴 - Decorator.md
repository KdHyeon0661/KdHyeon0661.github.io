---
layout: post
title: 디자인패턴 - Decorator
date: 2025-06-20 21:20:23 +0900
category: 디자인패턴
---
# Decorator (데코레이터 패턴)

## ✅ 정의

**데코레이터 패턴(Decorator Pattern)**은 객체에 추가적인 기능을 동적으로 부여할 수 있게 해주는 **구조 패턴(Structural Pattern)**입니다.  
클래스의 상속을 사용하지 않고도 **기존 객체에 기능을 확장**할 수 있으며, 여러 데코레이터를 **중첩**하여 다양한 기능 조합이 가능합니다.

> “기존 코드를 변경하지 않고 기능을 확장한다.”

---

## 🎯 의도 (Intent)

- 객체에 동적으로 새로운 기능을 추가한다.
- 서브클래스를 만들지 않고도 객체의 행동을 확장한다.
- **개방-폐쇄 원칙(OCP)**: 기존 코드는 수정하지 않고 확장만 가능하게 설계한다.

---

## 📦 구조 (UML)

```
          ┌────────────────┐
          │  Component     │◄────────────┐
          └────────────────┘             │
                   ▲                     │
                   │                     │
     ┌─────────────────────┐   ┌─────────────────────┐
     │   ConcreteComponent │   │    Decorator        │
     └─────────────────────┘   └─────────────────────┘
                                       ▲
                                       │
                        ┌──────────────────────────┐
                        │   ConcreteDecorator       │
                        └──────────────────────────┘
```

- `Component`: 기본 기능을 정의한 인터페이스
- `ConcreteComponent`: 기본 기능을 실제로 구현한 클래스
- `Decorator`: `Component`를 구현하고, 내부에 `Component`를 포함하는 클래스 (기능 확장용 래퍼)
- `ConcreteDecorator`: 실제 기능을 추가하는 데코레이터 클래스

---

## 🧑‍💻 구현 예시 (Python)

```python
from abc import ABC, abstractmethod

# Component
class Coffee(ABC):
    @abstractmethod
    def cost(self):
        pass

# ConcreteComponent
class BasicCoffee(Coffee):
    def cost(self):
        return 3000

# Decorator
class CoffeeDecorator(Coffee):
    def __init__(self, coffee: Coffee):
        self._coffee = coffee

    def cost(self):
        return self._coffee.cost()

# ConcreteDecorator
class Milk(CoffeeDecorator):
    def cost(self):
        return self._coffee.cost() + 500

class Whip(CoffeeDecorator):
    def cost(self):
        return self._coffee.cost() + 700

class Caramel(CoffeeDecorator):
    def cost(self):
        return self._coffee.cost() + 1000

# 사용 예
coffee = BasicCoffee()
print("기본 커피:", coffee.cost())  # 3000

milk_coffee = Milk(coffee)
whip_milk_coffee = Whip(milk_coffee)
caramel_whip_milk_coffee = Caramel(whip_milk_coffee)

print("카페 주문:", caramel_whip_milk_coffee.cost())  # 3000 + 500 + 700 + 1000 = 5200
```

---

## ✅ 장점

- **런타임에 객체 기능을 유연하게 추가 가능**
- 클래스 수 증가 없이 **조합 가능한 기능 확장**
- **상속보다 유연한 구조** (기능 추가 시 클래스 변경 불필요)
- 여러 데코레이터를 **중첩**하여 다양한 기능 조합 가능

---

## ⚠️ 단점

- 중첩이 많아질수록 **구조 추적이 어려움**
- **디버깅이 어려울 수 있음**
- 구성 객체와 데코레이터의 인터페이스가 일치해야 하므로, **인터페이스 설계가 중요**

---

## 📌 사용 사례

| 사용 사례 | 설명 |
|-----------|------|
| **Java I/O 스트림** | `BufferedReader`, `InputStreamReader`, `FileReader` 등 데코레이터 방식 |
| **웹 요청 처리기** | Django/Python의 미들웨어 또는 Express.js의 미들웨어 |
| **GUI 컴포넌트** | 버튼에 스크롤, 그림자 등 동적 추가 기능 부여 |
| **의류 코디 시스템** | 기본 옷에 모자, 장갑, 액세서리 등을 조합 |
| **알림 시스템** | 메시지 전송에 로그, 이메일, 슬랙 등 부가 기능 조합 |

---

## 🧠 Decorator vs Inheritance vs Strategy

| 패턴         | 목적 | 장점 | 단점 |
|--------------|------|------|------|
| **Decorator** | 런타임 기능 확장 | 유연한 기능 조합 | 중첩 시 복잡 |
| **Inheritance** | 컴파일타임 확장 | 구조 단순 | 조합 유연성 부족 |
| **Strategy** | 알고리즘 변경 | 캡슐화 명확 | 단일 역할 초점 |

---

## 🧠 마무리

**데코레이터 패턴**은 기존 클래스에 **기능을 유연하게 확장**하고 싶을 때 유용한 구조 패턴입니다.  
특히 상속 대신 **구성(composition)**을 활용함으로써 더 유연한 코드 구조를 설계할 수 있습니다.

단, **중첩된 데코레이터가 많아지면 코드 추적이 어려울 수 있으므로**, 적절한 조합과 명확한 명명 규칙이 중요합니다.
