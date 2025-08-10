---
layout: post
title: 디자인패턴 - Template Method
date: 2025-06-30 19:20:23 +0900
category: 디자인패턴
---
# Template Method (템플릿 메서드 패턴)

## ✅ 정의

**템플릿 메서드 패턴(Template Method Pattern)**은 **알고리즘의 골격(Template)**을 상위 클래스에 정의하고,  
일부 단계를 하위 클래스에서 **재정의(Override)** 하도록 하는 **행위 패턴**입니다.

> “알고리즘의 구조는 그대로 두고, 세부 단계는 서브클래스에서 정의한다.”

---

## 🎯 의도 (Intent)

- 알고리즘의 구조(순서, 흐름)는 고정하고
- 세부적인 처리 내용만 서브클래스에서 유연하게 변경할 수 있도록 함

---

## 📦 구조 (UML)

```
┌─────────────────┐
│  AbstractClass  │
├─────────────────┤
│ templateMethod()│  → 알고리즘 골격
│ step1()         │  → 기본 구현 or 추상 메서드
│ step2()         │
└──────┬──────────┘
       ▼
┌──────────────────┐
│  ConcreteClassA   │ → 일부 단계 오버라이딩
└──────────────────┘
┌──────────────────┐
│  ConcreteClassB   │
└──────────────────┘
```

---

## 🧑‍💻 구현 예시 (Python)

```python
from abc import ABC, abstractmethod

# 상위 클래스: 템플릿 정의
class CoffeeTemplate(ABC):
    def prepare(self):
        self.boil_water()
        self.brew()
        self.pour_in_cup()
        self.add_condiments()

    def boil_water(self):
        print("1️⃣ 물을 끓입니다.")

    def pour_in_cup(self):
        print("3️⃣ 컵에 따릅니다.")

    @abstractmethod
    def brew(self):
        pass

    @abstractmethod
    def add_condiments(self):
        pass

# 하위 클래스 1: 아메리카노
class Americano(CoffeeTemplate):
    def brew(self):
        print("2️⃣ 원두를 드립합니다.")
    def add_condiments(self):
        print("4️⃣ 아무것도 넣지 않습니다.")

# 하위 클래스 2: 라떼
class Latte(CoffeeTemplate):
    def brew(self):
        print("2️⃣ 에스프레소를 추출합니다.")
    def add_condiments(self):
        print("4️⃣ 우유를 추가합니다.")

# 사용 예시
coffee = Americano()
coffee.prepare()

print("-----------")

latte = Latte()
latte.prepare()
```

**출력 예시:**
```
1️⃣ 물을 끓입니다.
2️⃣ 원두를 드립합니다.
3️⃣ 컵에 따릅니다.
4️⃣ 아무것도 넣지 않습니다.
-----------
1️⃣ 물을 끓입니다.
2️⃣ 에스프레소를 추출합니다.
3️⃣ 컵에 따릅니다.
4️⃣ 우유를 추가합니다.
```

---

## ✅ 장점

- 알고리즘 구조를 **재사용** 가능
- **서브클래스 확장**만으로 다양한 동작 정의 가능
- 공통된 알고리즘 흐름을 **중복 없이 유지** 가능
- **프레임워크 설계**에 적합

---

## ⚠️ 단점

- **상속에 의존** → 유연성이 낮음 (런타임 교체 불가)
- 너무 많은 훅 메서드가 생기면 **복잡성 증가**
- 상위 클래스가 너무 커질 수 있음

---

## 📌 사용 사례

| 분야 | 예시 |
|------|------|
| 프레임워크 | Spring Framework의 추상 컨트롤러 |
| 게임 | 턴 처리 방식의 공통 알고리즘 + 캐릭터 별 특수 행동 |
| 데이터 파싱 | 템플릿 구조는 고정, 파싱 로직만 다름 |
| UI 렌더링 | 공통 렌더링 구조 + 커스터마이징된 스타일 |

---

## 🧠 Template Method vs Strategy

| 항목 | Template Method | Strategy |
|------|------------------|----------|
| **기반** | 상속(Inheritance) | 위임(Composition) |
| **알고리즘 변경 시기** | 컴파일 시(정적) | 런타임(동적) |
| **사용 방식** | 추상 메서드 오버라이드 | 전략 객체 교체 |
| **장점** | 코드 중복 제거 | 유연한 확장성 |
| **단점** | 클래스 간 강한 결합 | 객체 관리 복잡 |

---

## ✅ 실무 팁

- 알고리즘이 일정하고, **변경 가능한 단계만 다를 때** 유용
- 상속보다는 조합(Strategy)을 우선 고려하되, **코드 흐름을 고정해야 하는 경우에 Template Method가 적합**
- Template Method + Factory Method 함께 쓰이는 경우 많음

---

## 🧠 마무리

**Template Method 패턴은 “틀은 고정, 내용은 변동”이라는 개념으로**,  
공통 알고리즘 구조를 유지하면서 **확장성과 재사용성을 높이는 효과적인 방법**입니다.

특히 프레임워크나 게임, 데이터 처리 흐름 같은 **일정한 로직이 반복되는 상황에서 유용**하게 활용됩니다.
