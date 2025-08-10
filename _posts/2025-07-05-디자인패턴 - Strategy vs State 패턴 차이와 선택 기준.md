---
layout: post
title: 디자인패턴 - Strategy vs State 패턴 차이와 선택 기준
date: 2025-07-05 21:20:23 +0900
category: 디자인패턴
---
# Strategy vs State 패턴 차이와 선택 기준

---

## ✅ 개요

**Strategy 패턴**과 **State 패턴**은 모두 객체의 행위를 캡슐화하고  
동적으로 변경할 수 있게 해주는 **행위(Behavioral) 패턴**입니다.

구조는 매우 비슷하지만 **목적과 쓰임새**가 다릅니다.

| 공통점 |
|--------|
| 인터페이스(또는 추상 클래스)를 기반으로 하위 구현 클래스를 정의 |
| 런타임에 객체의 행위를 교체할 수 있음 |
| 클래스 수가 많아짐 (OOP의 단점 중 하나) |

---

## 🧩 공통 구조 비교

```plaintext
Context ────────┐
                ▼
          ┌────────────┐
          │ Strategy A │
          │ or         │
          │ State A    │
          └────────────┘
```

- **Context**: 현재 전략 또는 상태를 참조하며 위임
- **Strategy/State**: 인터페이스 또는 추상 클래스
- **Concrete Class**: 실제 알고리즘 또는 상태 구현

---

## 🎯 Strategy 패턴이란?

> **"동일한 작업을 서로 다른 방식으로 처리해야 할 때, 알고리즘을 캡슐화"**

### ✅ 핵심 특징

- 목적: **알고리즘(행위)**을 런타임에 쉽게 교체
- 클라이언트가 **직접 전략을 선택하거나 주입**
- 전략 간 전환은 클라이언트나 외부에서 제어

### ✅ 예시

```python
class Strategy:
    def execute(self, data): pass

class SortAsc(Strategy):
    def execute(self, data):
        return sorted(data)

class SortDesc(Strategy):
    def execute(self, data):
        return sorted(data, reverse=True)

class Context:
    def __init__(self, strategy):
        self.strategy = strategy

    def run(self, data):
        return self.strategy.execute(data)

ctx = Context(SortAsc())
print(ctx.run([3,1,2]))  # [1,2,3]

ctx.strategy = SortDesc()
print(ctx.run([3,1,2]))  # [3,2,1]
```

---

## 🎯 State 패턴이란?

> **"객체가 상태에 따라 서로 다른 행동을 하도록 만들며, 상태 전환을 내부에서 처리"**

### ✅ 핵심 특징

- 목적: **객체의 상태에 따라 행동을 다르게 정의**
- 상태 전환은 **객체 내부에서 발생**
- 상태 클래스는 다음 상태를 **직접 지정**할 수 있음

### ✅ 예시

```python
class State:
    def handle(self, context): pass

class LoggedOut(State):
    def handle(self, context):
        print("🔒 로그아웃 상태입니다.")
        context.state = LoggedIn()

class LoggedIn(State):
    def handle(self, context):
        print("🔓 로그인 상태입니다.")
        context.state = LoggedOut()

class Context:
    def __init__(self):
        self.state = LoggedOut()

    def request(self):
        self.state.handle(self)

ctx = Context()
ctx.request()  # 🔒 → 🔓
ctx.request()  # 🔓 → 🔒
```

---

## 🆚 Strategy vs State: 비교표

| 항목 | Strategy 패턴 | State 패턴 |
|------|----------------|-------------|
| **목적** | 알고리즘을 선택 가능하게 | 상태에 따른 행위 전환 |
| **전환 주체** | 외부에서 직접 전략 교체 | 내부에서 상태 전환 |
| **전략/상태의 관계** | 서로 독립적 | 서로 유기적으로 연결됨 |
| **클라이언트 개입** | 전략을 직접 지정 | 상태 전환은 내부에서 발생 |
| **상태 기억** | 없음 (단순한 전략 실행) | 상태 전환의 흐름이 존재 |
| **예시** | 정렬 방식, 결제 방법 등 | 게임 캐릭터 상태, 로그인 상태 등 |

---

## 🧪 실무 사용 예

| 도메인 | Strategy | State |
|--------|----------|-------|
| 정렬, 인코딩 | ✔️ | ❌ |
| 로그인/로그아웃 상태 | ❌ | ✔️ |
| 결제 수단 (카드/포인트) | ✔️ | ❌ |
| 게임 캐릭터 행동 | ❌ | ✔️ |
| 데이터 압축 방식 | ✔️ | ❌ |
| 문서 상태 (초안/검토/완료) | ❌ | ✔️ |

---

## 🧠 선택 기준 요약

| 조건 | 추천 패턴 |
|------|------------|
| 같은 동작을 다른 방식으로 구현하고 싶다 | ✅ **Strategy** |
| 객체의 내부 상태가 시간에 따라 변하고, 각 상태마다 동작이 다르다 | ✅ **State** |
| 상태 전환 흐름이 없다 (단순 교체) | ✅ **Strategy** |
| 상태에 따라 전이(Transition)가 일어나야 한다 | ✅ **State** |

---

## ✅ 마무리

- **Strategy**는 "무엇을 하느냐"보다 "어떻게 할 것인가"를 캡슐화
- **State**는 "지금 어떤 상태냐"에 따라 행동이 바뀌고, 상태 전이가 중요

두 패턴은 구조가 매우 유사하므로 **목적이 무엇인지**를 명확히 이해하는 것이  
실무에서 **적절한 패턴을 선택하는 핵심 기준**이 됩니다.