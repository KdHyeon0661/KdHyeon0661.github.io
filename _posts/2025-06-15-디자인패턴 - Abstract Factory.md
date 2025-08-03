---
layout: post
title: 디자인패턴 - Abstract Factory
date: 2025-06-15 19:20:23 +0900
category: 디자인패턴
---
# Abstract Factory (추상 팩토리 패턴)

## ✅ 정의

**추상 팩토리(Abstract Factory) 패턴**은 관련된 객체들의 **제품군(family of products)**을 생성하기 위한 **인터페이스**를 제공하며, **구체적인 클래스는 서브클래스가 정의**하도록 만드는 **생성 패턴(Creational Pattern)**입니다.

즉, 서로 관련 있는 객체들을 **일관되게 생성**하고 싶을 때 사용하며, 클라이언트 코드가 객체 생성 방식과 구체적인 클래스에 의존하지 않도록 해줍니다.

---

## 🎯 의도 (Intent)

- 관련된 객체들을 하나의 제품군으로 묶어 일관되게 생성한다.
- 구체적인 클래스 대신 **인터페이스(추상 타입)**만을 사용하여 **느슨한 결합**을 유지한다.
- **팩토리 메서드 패턴의 확장된 형태**로, 여러 제품을 동시에 생성할 수 있다.

---

## 📦 구조 (Structure)

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

---

## 🧑‍💻 구현 예시 (Python)

```python
# 제품 A 계열
class Button:
    def render(self): pass

class WindowsButton(Button):
    def render(self):
        print("윈도우 버튼 렌더링")

class MacButton(Button):
    def render(self):
        print("맥 버튼 렌더링")

# 제품 B 계열
class Checkbox:
    def render(self): pass

class WindowsCheckbox(Checkbox):
    def render(self):
        print("윈도우 체크박스 렌더링")

class MacCheckbox(Checkbox):
    def render(self):
        print("맥 체크박스 렌더링")

# 추상 팩토리
class GUIFactory:
    def create_button(self): pass
    def create_checkbox(self): pass

# 구체 팩토리들
class WindowsFactory(GUIFactory):
    def create_button(self):
        return WindowsButton()

    def create_checkbox(self):
        return WindowsCheckbox()

class MacFactory(GUIFactory):
    def create_button(self):
        return MacButton()

    def create_checkbox(self):
        return MacCheckbox()

# 클라이언트 코드
def render_ui(factory: GUIFactory):
    button = factory.create_button()
    checkbox = factory.create_checkbox()
    button.render()
    checkbox.render()

# 사용 예
render_ui(WindowsFactory())
render_ui(MacFactory())
```

---

## ✅ 장점

- 관련된 객체들을 **일관성 있게 생성** 가능
- 클라이언트 코드가 **구체 클래스에 의존하지 않음**
- 새로운 제품군을 추가할 때 유연한 구조
- 테스트 시 **Mock 팩토리**로 대체 가능

---

## ⚠️ 단점

- 클래스가 많아지고 구조가 복잡해질 수 있음
- 제품을 추가하는 경우에는 기존 팩토리를 **수정**해야 함 → **OCP 위배 가능**
- 모든 제품군이 반드시 동시에 쓰일 필요가 없을 경우 **불필요한 추상화**가 될 수 있음

---

## 📌 사용 사례

| 사용 사례               | 설명 |
|------------------------|------|
| GUI 라이브러리         | 플랫폼(Mac/Windows/Linux)에 따라 위젯 제품군을 교체 |
| 게임 엔진 테마 시스템  | 스킨, 배경, 효과 등 테마별 리소스를 일관되게 생성 |
| 데이터베이스 커넥터    | RDBMS별 커넥션, 쿼리, 트랜잭션 객체를 생성 |
| 테스트 환경 대체       | 실제 객체 대신 테스트용 팩토리 주입 |

---

## 🧠 팩토리 메서드 vs 추상 팩토리

| 항목              | 팩토리 메서드                   | 추상 팩토리                         |
|-------------------|----------------------------------|-------------------------------------|
| 목적              | 하나의 객체 생성                 | 관련된 객체들의 제품군 생성         |
| 클래스 수         | 상대적으로 적음                  | 상대적으로 많음                    |
| 유연성            | 일반적인 생성 패턴               | 제품군 간의 일관성 확보 가능        |
| 사용 예           | 단일 객체 선택적으로 생성        | GUI, 테마, 연결 등 묶음 생성 필요 시 |

---

## 🧠 마무리

**추상 팩토리 패턴**은 관련 있는 객체들을 일관되게 생성하고 싶을 때 유용한 패턴입니다.  
**프레임워크나 라이브러리**처럼 확장이 잦고 다양한 구현을 필요로 하는 시스템에서 강력한 유연성을 제공합니다.

하지만, 제품군이 간단하거나 하나의 객체만 필요하다면 오히려 **오버엔지니어링**이 될 수 있으므로, 복잡한 시스템 설계 시에만 적용하는 것이 좋습니다.
