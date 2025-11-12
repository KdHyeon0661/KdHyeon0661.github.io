---
layout: post
title: 디자인패턴 - Interpreter
date: 2025-07-05 19:20:23 +0900
category: 디자인패턴
---
# Interpreter (인터프리터 패턴)

## 1. 핵심 개념과 의도

- **의도**: 문법이 정의된 언어에 대해, 그 문장을 해석할 수 있도록 **클래스(객체 트리)**로 표현하고, 각 노드가 자신을 **해석(interpret)**한다.
- **핵심 구조**
  - `AbstractExpression`(공통 인터페이스: `interpret(context)`)
  - `TerminalExpression`(숫자, 문자열, 식별자 등 더 분해되지 않는 단말)
  - `NonTerminalExpression`(이항/단항 연산, 함수 호출, 제어식 등 다른 표현식 조합)
  - `Context`(변수/함수 테이블 등 해석에 필요한 외부 상태)
- **언제 쓰나**
  - 작고 비교적 간단한 **도메인 전용 언어(DSL)** 를 빠르게 내장하고 싶을 때
  - 반복되는 규칙/조건을 안전하게 런타임 해석하고 싶을 때
- **언제 피하나**
  - 문법이 크고 자주 변하며 성능이 중요한 경우(컴파일러 수준) → 파서 생성기(ANTLR/PEG)나 JIT/바이트코드 방식을 고려

---

## 2. 인터프리터 패턴과 트리 구조(UML)

```
             ┌────────────────┐
             │ AbstractExpr   │  interpret(ctx)
             └──────┬─────────┘
      ┌─────────────┴─────────────┐
      ▼                           ▼
┌────────────┐            ┌──────────────────┐
│ Terminal   │            │ NonTerminal      │
│ Expression │            │ Expression       │
└────────────┘            └──────────────────┘

┌──────────────┐
│   Context     │  변수·함수 테이블, 실행 옵션, 에러 정책 등
└──────────────┘
```

---

## 3. 우리가 만들 미니 언어(DSL)

### 3.1 지원 요소

- **리터럴**: 정수/실수, 문자열(`"..."`), 불리언(`true`/`false`)
- **식별자**: 변수 이름
- **연산자**(우선순위 낮음→높음)
  - 논리: `or`, `and`, `not`
  - 비교: `==`, `!=`, `<`, `<=`, `>`, `>=`
  - 산술: `+`, `-`, `*`, `/`, `%`
  - 단항: `-x`, `not x`
- **그룹핑**: 괄호 `( ... )`
- **대입**: `name = expr`
- **함수 호출**: `name(arg1, arg2, ...)`
- **문장 구분**: 세미콜론 `;` (선택적, 여러 문장 가능)

### 3.2 예시

- 수식: `1 + 2 * 3 - 4 / 2`
- 규칙: `amount > 10000 and country == "KR" and vip`
- 변수/함수: `x = 10; y = max(x*2, 15); y + 1`

---

## 4. 파이썬으로 구현: 토크나이저 → 파서 → AST → 인터프리터

아래 코드는 하나의 파일로 동작하도록 작성했다.  
입출력 예제와 단위 테스트 격의 검증 코드까지 포함한다.

```python
# interpreter_dsl.py
# Python 3.10+

from __future__ import annotations
from dataclasses import dataclass
from typing import Any, Callable, Dict, List, Optional, Sequence, Tuple
import re

# ---------------------------
# 1. 토큰 정의와 토크나이저
# ---------------------------

@dataclass(frozen=True)
class Token:
    type: str
    lexeme: str
    line: int
    col: int

class LexerError(Exception): ...
class ParseError(Exception): ...
class RuntimeEvalError(Exception): ...

KEYWORDS = {
    "true": "TRUE",
    "false": "FALSE",
    "and": "AND",
    "or": "OR",
    "not": "NOT",
}

TOKEN_SPEC = [
    ("NUMBER",   r'\d+(\.\d+)?'),
    ("STRING",   r'"(?:\\.|[^"\\])*"'),
    ("ID",       r'[A-Za-z_][A-Za-z0-9_]*'),
    ("EQ",       r'=='),
    ("NE",       r'!='),
    ("LE",       r'<='),
    ("GE",       r'>='),
    ("LT",       r'<'),
    ("GT",       r'>'),
    ("ASSIGN",   r'='),
    ("PLUS",     r'\+'),
    ("MINUS",    r'-'),
    ("STAR",     r'\*'),
    ("SLASH",    r'/'),
    ("PERCENT",  r'%'),
    ("LPAREN",   r'\('),
    ("RPAREN",   r'\)'),
    ("COMMA",    r','),
    ("SEMI",     r';'),
    ("WS",       r'[ \t]+'),
    ("NEWLINE",  r'\n'),
    ("MISMATCH", r'.'),
]
MASTER_RE = re.compile("|".join(f"(?P<{n}>{p})" for n, p in TOKEN_SPEC))

def lex(source: str) -> List[Token]:
    tokens: List[Token] = []
    line, col = 1, 1
    i = 0
    while i < len(source):
        m = MASTER_RE.match(source, i)
        if not m:
            raise LexerError(f"Invalid char at {line}:{col}")
        kind = m.lastgroup
        text = m.group()
        if kind == "WS":
            pass
        elif kind == "NEWLINE":
            line += 1
            col = 0
        elif kind == "ID":
            t = KEYWORDS.get(text, "ID")
            tokens.append(Token(t, text, line, col))
        elif kind == "STRING":
            tokens.append(Token("STRING", text, line, col))
        elif kind == "NUMBER":
            tokens.append(Token("NUMBER", text, line, col))
        elif kind == "MISMATCH":
            raise LexerError(f"Unexpected token '{text}' at {line}:{col}")
        else:
            tokens.append(Token(kind, text, line, col))
        i = m.end()
        col += len(text)
    tokens.append(Token("EOF", "", line, col))
    return tokens

# ---------------------------
# 2. AST 노드(인터프리터 패턴)
# ---------------------------

class Context:
    """실행에 필요한 외부 상태(변수/함수)."""
    def __init__(self,
                 variables: Optional[Dict[str, Any]] = None,
                 functions: Optional[Dict[str, Callable[..., Any]]] = None):
        self.variables: Dict[str, Any] = variables or {}
        self.functions: Dict[str, Callable[..., Any]] = functions or {}

class Expr:
    def interpret(self, ctx: Context) -> Any:
        raise NotImplementedError()

@dataclass
class NumberLit(Expr):
    value: float
    def interpret(self, ctx: Context) -> Any:
        return self.value

@dataclass
class StringLit(Expr):
    value: str
    def interpret(self, ctx: Context) -> Any:
        return self.value

@dataclass
class BoolLit(Expr):
    value: bool
    def interpret(self, ctx: Context) -> Any:
        return self.value

@dataclass
class Variable(Expr):
    name: str
    line: int
    col: int
    def interpret(self, ctx: Context) -> Any:
        if self.name not in ctx.variables:
            raise RuntimeEvalError(f"Undefined variable '{self.name}' at {self.line}:{self.col}")
        return ctx.variables[self.name]

@dataclass
class Assign(Expr):
    name: str
    expr: Expr
    def interpret(self, ctx: Context) -> Any:
        val = self.expr.interpret(ctx)
        ctx.variables[self.name] = val
        return val

@dataclass
class Unary(Expr):
    op: str  # 'MINUS' or 'NOT'
    right: Expr
    line: int
    col: int
    def interpret(self, ctx: Context) -> Any:
        v = self.right.interpret(ctx)
        if self.op == "MINUS":
            if isinstance(v, (int, float)):
                return -v
            raise RuntimeEvalError(f"Unary '-' expects number at {self.line}:{self.col}")
        elif self.op == "NOT":
            return not bool(v)
        raise RuntimeEvalError(f"Unknown unary op {self.op} at {self.line}:{self.col}")

@dataclass
class Binary(Expr):
    left: Expr
    op: str
    right: Expr
    line: int
    col: int
    def interpret(self, ctx: Context) -> Any:
        lv = self.left.interpret(ctx)
        rv = self.right.interpret(ctx)
        try:
            if self.op == "PLUS":    return lv + rv
            if self.op == "MINUS":   return lv - rv
            if self.op == "STAR":    return lv * rv
            if self.op == "SLASH":   return lv / rv
            if self.op == "PERCENT": return lv % rv
            if self.op == "EQ":      return lv == rv
            if self.op == "NE":      return lv != rv
            if self.op == "LT":      return lv <  rv
            if self.op == "LE":      return lv <= rv
            if self.op == "GT":      return lv >  rv
            if self.op == "GE":      return lv >= rv
            if self.op == "AND":     return bool(lv) and bool(rv)
            if self.op == "OR":      return bool(lv) or bool(rv)
        except Exception as e:
            raise RuntimeEvalError(f"Operator error {self.op} at {self.line}:{self.col}: {e}")
        raise RuntimeEvalError(f"Unknown binary op {self.op} at {self.line}:{self.col}")

@dataclass
class Call(Expr):
    callee: str
    args: List[Expr]
    line: int
    col: int
    def interpret(self, ctx: Context) -> Any:
        if self.callee not in ctx.functions:
            raise RuntimeEvalError(f"Unknown function '{self.callee}' at {self.line}:{self.col}")
        fn = ctx.functions[self.callee]
        vals = [a.interpret(ctx) for a in self.args]
        try:
            return fn(*vals)
        except Exception as e:
            raise RuntimeEvalError(f"Call error {self.callee} at {self.line}:{self.col}: {e}")

@dataclass
class Program:
    statements: List[Expr]
    def interpret(self, ctx: Context) -> Any:
        last = None
        for s in self.statements:
            last = s.interpret(ctx)
        return last

# ---------------------------
# 3. 파서(재귀하강, 우선순위 보장)
# ---------------------------

class Parser:
    def __init__(self, tokens: List[Token]) -> None:
        self.toks = tokens
        self.i = 0

    def peek(self) -> Token: return self.toks[self.i]
    def prev(self) -> Token: return self.toks[self.i-1]
    def is_at_end(self) -> bool: return self.peek().type == "EOF"

    def match(self, *types: str) -> bool:
        if self.peek().type in types:
            self.i += 1
            return True
        return False

    def consume(self, t: str, msg: str):
        if self.peek().type == t:
            tok = self.peek()
            self.i += 1
            return tok
        p = self.peek()
        raise ParseError(f"{msg} at {p.line}:{p.col}, got {p.type}")

    def parse(self) -> Program:
        stmts: List[Expr] = []
        while not self.is_at_end():
            if self.peek().type == "SEMI":
                self.i += 1
                continue
            stmts.append(self.statement())
            self.match("SEMI")  # optional
        return Program(stmts)

    def statement(self) -> Expr:
        # assignment: ID '=' expr
        if self.peek().type == "ID" and self._is_assign_lookahead():
            name_tok = self.peek()
            self.i += 1  # consume ID
            self.consume("ASSIGN", "Expect '=' after identifier")
            expr = self.expression()
            return Assign(name_tok.lexeme, expr)
        return self.expression()

    def _is_assign_lookahead(self) -> bool:
        return (self.i + 1 < len(self.toks)) and (self.toks[self.i+1].type == "ASSIGN")

    # precedence: or > and > equality > comparison > term > factor > unary > call > primary

    def expression(self) -> Expr:
        return self.or_()

    def or_(self) -> Expr:
        expr = self.and_()
        while self.match("OR"):
            op = self.prev()
            right = self.and_()
            expr = Binary(expr, "OR", right, op.line, op.col)
        return expr

    def and_(self) -> Expr:
        expr = self.equality()
        while self.match("AND"):
            op = self.prev()
            right = self.equality()
            expr = Binary(expr, "AND", right, op.line, op.col)
        return expr

    def equality(self) -> Expr:
        expr = self.comparison()
        while self.match("EQ", "NE"):
            op = self.prev()
            right = self.comparison()
            expr = Binary(expr, op.type, right, op.line, op.col)
        return expr

    def comparison(self) -> Expr:
        expr = self.term()
        while self.match("LT", "LE", "GT", "GE"):
            op = self.prev()
            right = self.term()
            expr = Binary(expr, op.type, right, op.line, op.col)
        return expr

    def term(self) -> Expr:
        expr = self.factor()
        while self.match("PLUS", "MINUS"):
            op = self.prev()
            right = self.factor()
            expr = Binary(expr, op.type, right, op.line, op.col)
        return expr

    def factor(self) -> Expr:
        expr = self.unary()
        while self.match("STAR", "SLASH", "PERCENT"):
            op = self.prev()
            right = self.unary()
            expr = Binary(expr, op.type, right, op.line, op.col)
        return expr

    def unary(self) -> Expr:
        if self.match("NOT", "MINUS"):
            op = self.prev()
            right = self.unary()
            return Unary(op.type, right, op.line, op.col)
        return self.call()

    def call(self) -> Expr:
        expr = self.primary()
        while self.match("LPAREN"):
            lpar = self.prev()
            args: List[Expr] = []
            if not self.match("RPAREN"):
                args.append(self.expression())
                while self.match("COMMA"):
                    args.append(self.expression())
                self.consume("RPAREN", "Expect ')' after arguments")
            # callee는 식별자여야 한다
            if isinstance(expr, Variable):
                expr = Call(expr.name, args, lpar.line, lpar.col)
            else:
                raise ParseError(f"Only identifier can be called at {lpar.line}:{lpar.col}")
        return expr

    def primary(self) -> Expr:
        tok = self.peek()
        if self.match("NUMBER"):
            v = float(tok.lexeme) if "." in tok.lexeme else int(tok.lexeme)
            return NumberLit(v)
        if self.match("STRING"):
            # 따옴표 제거 및 이스케이프 처리
            raw = tok.lexeme[1:-1]
            s = bytes(raw, "utf-8").decode("unicode_escape")
            return StringLit(s)
        if self.match("TRUE"):  return BoolLit(True)
        if self.match("FALSE"): return BoolLit(False)
        if self.match("ID"):
            return Variable(tok.lexeme, tok.line, tok.col)
        if self.match("LPAREN"):
            expr = self.expression()
            self.consume("RPAREN", "Expect ')'")
            return expr
        raise ParseError(f"Unexpected token {tok.type} '{tok.lexeme}' at {tok.line}:{tok.col}")

# ---------------------------
# 4. 유틸: 실행 헬퍼
# ---------------------------

def default_functions() -> Dict[str, Callable[..., Any]]:
    return {
        "max": max,
        "min": min,
        "len": lambda x: len(x),
        "upper": lambda s: str(s).upper(),
        "lower": lambda s: str(s).lower(),
        "abs": abs,
    }

def run(source: str,
        variables: Optional[Dict[str, Any]] = None,
        functions: Optional[Dict[str, Callable[..., Any]]] = None) -> Tuple[Program, Any, Context]:
    tokens = lex(source)
    ast = Parser(tokens).parse()
    ctx = Context(variables=variables or {}, functions=functions or default_functions())
    result = ast.interpret(ctx)
    return ast, result, ctx

# ---------------------------
# 5. 데모
# ---------------------------

if __name__ == "__main__":
    # 1) 산술/우선순위
    src1 = "1 + 2 * 3 - 4 / 2"
    _, r1, _ = run(src1)
    print("[ex1]", src1, "=>", r1)  # 1 + 6 - 2 = 5

    # 2) 변수/함수/대입/세미콜론
    src2 = "x = 10; y = max(x*2, 15); y + 1"
    _, r2, ctx2 = run(src2)
    print("[ex2]", src2, "=>", r2, "| vars:", ctx2.variables)

    # 3) 규칙 엔진 스타일
    vars3 = {"amount": 12000, "country": "KR", "vip": True}
    src3 = 'amount > 10000 and country == "KR" and vip'
    _, r3, _ = run(src3, variables=vars3)
    print("[ex3]", src3, "=>", r3)

    # 4) 문자열/불리언/함수 조합
    src4 = 'greet = "hello"; upper(greet) == "HELLO" and not false'
    _, r4, ctx4 = run(src4)
    print("[ex4]", src4, "=>", r4, "| vars:", ctx4.variables)
```

### 실행 결과 예

```
[ex1] 1 + 2 * 3 - 4 / 2 => 5.0
[ex2] x = 10; y = max(x*2, 15); y + 1 => 21 | vars: {'x': 10, 'y': 20}
[ex3] amount > 10000 and country == "KR" and vip => True
[ex4] greet = "hello"; upper(greet) == "HELLO" and not false => True | vars: {'greet': 'hello'}
```

---

## 5. 설계 논점과 인터프리터 패턴의 적용 포인트

- **터미널 표현식**: `NumberLit`, `StringLit`, `BoolLit`, `Variable`  
  각 노드는 더 이상 분해되지 않으며, `interpret(ctx)`가 즉시 값을 돌려준다.
- **논터미널 표현식**: `Unary`, `Binary`, `Call`, `Assign`  
  자식 노드의 `interpret`를 재귀적으로 호출하며 의미 규칙을 수행한다.
- **Context**: 변수/함수 테이블을 캡슐화하여 **해석 시 필요한 외부 상태**를 주입한다.
- **Program**: 여러 문장(표현식)을 순차 실행. 마지막 표현식 값 반환.

---

## 6. 오류 처리와 진단 품질

- **어휘 단계**: 알 수 없는 문자 `LexerError`
- **구문 단계**: 기대 토큰이 없을 때 `ParseError`(행/열 포함)
- **실행 단계**: 미정의 변수, 잘못된 연산/호출 시 `RuntimeEvalError`
- **에러 메시지 원칙**
  - 가능한 한 **행/열** 제공
  - 연산자/함수 이름 등 **문맥 정보** 포함
  - 사용자 DSL에 초점을 맞춘 메시지(호출 인자 수 등도 검사 가능)

---

## 7. 테스트와 검증

- **정확성**: 다양한 입력 케이스(우선순위, 괄호, 단항/이항 조합, 문자열/불리언)
- **음수/경계**: 슬래시 제로, 타입 불일치, 미정의 식별자
- **성질 기반**: 예를 들어, 상수식에 대해 파스-실행 결과가 파이썬의 동일 식과 일치하는지(허용 가능한 범위에서)
- **스냅샷**: 오류 메시지 형식의 안정성 보장

간단 검증 스니펫:

```python
def _eval(src, vars=None):
    return run(src, variables=vars)[1]

assert _eval("1+2*3") == 7
assert _eval("(1+2)*3") == 9
assert _eval('not false and true') is True
assert _eval('x=5; x*2') == 10
assert _eval('len("abc")') == 3
assert _eval('max(1,2,3)') == 3
assert _eval('10 % 3') == 1
```

---

## 8. 성능·캐싱·스레드 안전

- **시간 복잡도**: 토크나이즈/파싱은 입력 길이 n에 대해 대체로 **$$O(n)$$**, 실행은 AST 크기 m에 대해 **$$O(m)$$**.
- **캐싱**: 동일 표현식을 반복 평가한다면, **파싱 결과(AST)를 캐시**하고 Context만 바꿔 실행하라.
- **함수 호출 비용**: 미리 바인딩된 내장 함수 테이블(딕셔너리)로 O(1) 조회.
- **스레드 안전**: `Context`를 쓰기 공유하지 말고, 요청 단위로 **분리된 Context**를 사용한다.

---

## 9. 실무 예제 시나리오

### 9.1 규칙 엔진(Feature Flag/프로모션)

- 규칙: `amount > 5000 and country == "KR" and (vip or level >= 3)`
- 요청마다 변수만 바꾸고 **AST 재사용**으로 저렴하게 평가
- 위험한 `eval` 없이 안전하게 동작(허용된 연산/함수만 수행)

### 9.2 로그 필터 DSL

- `level >= 3 and message != "" and not service=="billing"`
- 문자열/불리언/비교 연산으로 간결하게 필터 조건 정의

### 9.3 레이팅/가격 정책

- `base = 1000; factor = 1.2; final = base * factor; final`

---

## 10. 확장 아이디어

- **단축평가**: 현재 구현은 불리언 연산에서 피연산자 모두 평가. 필요시 `AND`/`OR`에 단축평가를 적용(왼쪽 값으로 오른쪽 평가 생략).
- **형 변환 규칙**: 문자열↔숫자 자동 변환 정책(명확히 정하지 않으면 런타임 에러를 유지).
- **예약어 추가**: `let`, `if(...) then ... else ...` 등 제어식(간단한 3항 연산으로도 가능).
- **컬렉션 타입**: 리스트/맵 리터럴, 인덱싱 `arr[0]`, 점 표기법 `obj.name`.
- **보안**: 함수 테이블을 최소화하고, 외부 객체 접근을 금지하여 샌드박스 보장.

---

## 11. Interpreter vs Visitor vs Composite

| 패턴 | 목적 | 유사점 | 차이점 |
|------|------|--------|--------|
| Interpreter | 문법을 객체로 모델링하고 해석 | 트리 구조 사용 | 각 노드가 `interpret` 보유(의미 규칙 포함) |
| Visitor | 연산을 객체 밖으로 분리 | 트리 순회 활용 | 해석 로직을 Visitor로 외부화(연산 추가에 유리) |
| Composite | 계층 구조 표현 | 트리 표현 | 의미/해석보다는 구조적 합성에 초점 |

- 문법이 안정적이고, **연산(평가/출력/검증 등)이 자주 추가**된다면 Visitor로 `EvalVisitor`, `PrettyPrintVisitor` 등을 분리하는 것도 선택지다.

---

## 12. 안티패턴과 주의

- **클래스 폭발**: 문법이 커질수록 노드/클래스가 증가. DSL 범위를 관리하라.
- **무분별한 함수/변수 노출**: 샌드박스를 깨뜨리는 순간 보안 문제가 된다.
- **우선순위/결합 규칙 오류**: 파서에서 누락되면 해석이 틀린다. 반드시 표준 우선순위를 테이블로 고정하고 테스트하라.
- **에러 메시지 빈약**: DSL 사용자는 해석기 내부를 모른다. 위치 정보와 원인 제공은 필수.

---

## 13. 리팩터링 가이드(기존 if/switch 기반 해석 제거)

1) 각 연산/구문을 **전용 AST 노드**로 도출하고 `interpret(ctx)` 메서드를 부여.  
2) 거대한 `if-elif` 분기를 **파서 규칙**으로 이전하여 구조화.  
3) 순회와 연산 혼합을 피하고, 필요하면 Visitor로 연산을 분해.  
4) 에러 처리/메시지 정책을 통합(행·열·토큰 정보).

---

## 14. 마무리

인터프리터 패턴은 **작고 반복적인 문법**을 빠르게 내장하고자 할 때 가장 직관적이고 안전한 해법이다.  
이 글의 구현은 실무에서 바로 사용할 수 있는 최소 골격을 모두 포함한다.

- 토크나이저/파서/AST/인터프리터의 **완결된 파이프라인**
- 변수·함수·대입·우선순위·불리언/비교·문자열
- 에러 보고, 캐싱, 확장 포인트

표현식이 복잡해지거나 성능이 절대적으로 중요해지면, Visitor 도입, 바이트코드/스택머신, 또는 파서 생성기로의 전환을 검토하라.  
그 전까지는, 위의 패턴 자체가 **간결성과 통제 가능한 복잡도**를 보장한다.