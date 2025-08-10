---
layout: post
title: 디자인패턴 - Interpreter
date: 2025-07-05 19:20:23 +0900
category: 디자인패턴
---
# Interpreter (인터프리터 패턴)

## ✅ 정의

**Interpreter 패턴**은 **문법이 정의된 언어(표현)**를 해석하기 위한 클래스를 구성하여,  
그 **문장을 해석(interpret)**하는 역할을 하는 **행위 패턴(Behavioral Pattern)**입니다.

> “언어의 문법을 객체 구조로 표현하고, 그 구조를 이용해 문장을 해석”

---

## 🎯 의도 (Intent)

- 특정 **언어의 문법을 정의하고**, 그 언어로 쓰인 문장을 **해석하는 구조를 클래스 기반으로 구현**
- 복잡한 표현식을 **객체 트리(파스 트리)로 표현**하고 재귀적으로 해석

---

## 📦 구조 (UML)

```
             ┌────────────────┐
             │  AbstractExpr  │◄────────┐
             └──────┬─────────┘         │
      ┌─────────────┴────────────┐      │
      ▼                          ▼      │
┌────────────┐           ┌────────────┐ │
│ Terminal   │           │ NonTerminal│◄┘
│ Expression │           │ Expression │
└────┬───────┘           └────┬───────┘
     │                        │
     ▼                        ▼
 ┌──────────────┐     ┌────────────────┐
 │ Context      │     │ Client         │
 └──────────────┘     └────────────────┘
```

- **AbstractExpression**: 모든 표현식의 공통 인터페이스 (`interpret(Context)` 포함)
- **TerminalExpression**: 최종 노드(숫자, 문자 등), 더 이상 분해되지 않음
- **NonTerminalExpression**: 다른 표현식을 조합하여 구성 (예: 더하기, 빼기 등)
- **Context**: 해석 과정에서 필요한 외부 정보를 담는 환경
- **Client**: 파스 트리를 만들고 해석을 요청하는 사용자

---

## 🧑‍💻 구현 예시 (Python): 간단한 수식 해석기

예: 문자열 `"5 + 3 - 2"`를 해석하여 계산하기

```python
from abc import ABC, abstractmethod

# 추상 표현식
class Expression(ABC):
    @abstractmethod
    def interpret(self):
        pass

# 숫자 표현식 (터미널 노드)
class Number(Expression):
    def __init__(self, value):
        self.value = int(value)

    def interpret(self):
        return self.value

# 덧셈 표현식
class Add(Expression):
    def __init__(self, left, right):
        self.left = left
        self.right = right

    def interpret(self):
        return self.left.interpret() + self.right.interpret()

# 뺄셈 표현식
class Subtract(Expression):
    def __init__(self, left, right):
        self.left = left
        self.right = right

    def interpret(self):
        return self.left.interpret() - self.right.interpret()

# 수식 해석기
def parse_expression(expression: str):
    tokens = expression.split()
    stack = []

    i = 0
    while i < len(tokens):
        token = tokens[i]

        if token.isdigit():
            stack.append(Number(token))
        elif token == "+":
            left = stack.pop()
            right = Number(tokens[i + 1])
            stack.append(Add(left, right))
            i += 1
        elif token == "-":
            left = stack.pop()
            right = Number(tokens[i + 1])
            stack.append(Subtract(left, right))
            i += 1
        i += 1

    return stack.pop()

# 사용 예시
expr = "5 + 3 - 2"
parsed = parse_expression(expr)
print(f"🧮 수식: {expr}")
print(f"🔎 결과: {parsed.interpret()}")
```

**출력 예시:**
```
🧮 수식: 5 + 3 - 2
🔎 결과: 6
```

---

## ✅ 장점

- **문법과 해석을 분리**하여 코드 구조가 명확해짐
- **재귀적 문법 구조**에 자연스럽게 적용 가능 (ex. 수식, 언어 문법)
- 새로운 문법 추가 시 **클래스만 추가하면 됨** (확장 용이)

---

## ⚠️ 단점

- 문법이 복잡하거나 표현식이 많아질수록 **클래스 수가 급증**
- 실행 성능이 좋지 않음 → **파싱 최적화에 부적합**
- AST 기반의 컴파일러나 인터프리터가 더 나은 경우도 많음

---

## 📌 사용 사례

| 분야 | 예시 |
|------|------|
| 수식 계산기 | 수학식 계산, 논리 연산 |
| SQL 해석기 | 질의문 해석 및 실행 |
| 정규식 엔진 | 패턴 해석 및 실행 |
| Rule Engine | 도메인 언어(DSL)를 해석 |
| 컴파일러 프론트엔드 | AST 생성 및 해석 단계 |

---

## 🧠 Interpreter vs Visitor vs Composite

| 패턴 | 목적 | 유사점 | 차이점 |
|------|------|--------|--------|
| Interpreter | 문법 해석 | 구조를 트리로 구성 | 해석이 목적 |
| Visitor | 연산 분리 | 트리 구조 순회 | 연산이 목적 |
| Composite | 계층 구조 표현 | 트리 구조 활용 | 구조 조합 중심 |

---

## ✅ 실무 팁

- 실제 사용 시, 인터프리터 패턴보다는 **파서 생성기(ANTLR, yacc, PEG)** 등의 도구를 많이 사용
- 하지만 DSL(도메인 특화 언어) 구현이나 규칙 해석기 등에선 여전히 유용
- 간단한 수식이나 조건문(예: "금액 > 5000")을 런타임에 동적으로 해석할 때 활용 가능

---

## 🧠 마무리

**Interpreter 패턴은 문법이 존재하는 언어나 수식 구조를 객체로 모델링하고,**  
그 구조를 해석하여 동작하게 만드는 데 강력한 유연성을 제공합니다.

복잡한 표현식 처리에는 부적합할 수 있지만,  
**작고 반복되는 문법 구조의 해석 작업에선 매우 직관적이고 효과적인 솔루션**입니다.
