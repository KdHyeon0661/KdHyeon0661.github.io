---
layout: post
title: 형식언어와 오토마타 - Hopcroft’s Algorithm vs Moore’s Algorithm
date: 2025-06-18 23:20:23 +0900
category: 형식언어와 오토마타
---
# DFA 최소화를 위한 알고리즘: Hopcroft’s Algorithm vs Moore’s Algorithm — 개선본(심화)

DFA(Deterministic Finite Automaton)는 **같은 언어를 인식**하는 서로 다른 구조(상태 수/전이)를 가질 수 있다.
**DFA 최소화**는 언어를 바꾸지 않으면서 **상태 수를 가능한 한 줄이는** 과정이며, 최소 DFA는 **동형(isomorphism)까지 유일**하다(= Myhill–Nerode 동치류 개수와 동일).

본 글은 실무·교육 모두에서 널리 쓰이는 두 최소화 알고리즘을 **정의 → 절차 → 복잡도 → 예제 전개 → 코드**로 정리한다.

- **Hopcroft(1971)**: 이론/실무 표준. **\(O(|\Sigma|\,n\log n)\)** 에 가까운 최적 복잡도
- **Moore(1956)**: 가장 직관적인 **반복 분할**(partition refinement). **\(O(|\Sigma|\,n^2)\)** 근방

> 보너스: 표 채움(table-filling, Myhill–Nerode 표기) 방식은 교육에서 자주 보지만 대형 DFA에는 비실용적이라 부록 참고로만 짚는다.

---

## 전제(Preliminaries)

- DFA \(D=(Q,\Sigma,\delta,q_0,F)\) 는 **완전(complete)** 하다고 가정(모든 \((q,a)\)에 대해 전이 정의).
  완전하지 않으면 **sink(죽은) 상태**를 추가해 완전화하고 최소화해야 **여집합/차집합** 등 연산이 정확해진다.
- **도달 불가 상태 제거**: 최소화 전, \(q_0\)에서 도달 불가한 상태는 삭제(언어에 영향 없음).
- **최소 DFA**는 Myhill–Nerode 동치류와 1:1로 대응하며, 동형까지 **유일**.

---

## Hopcroft’s Algorithm

### 핵심 아이디어

> “**작은 분할부터** 효율적으로 쪼개며(Refinement) 한 번의 작업에서 **최대한 많은 상태를 한꺼번에 분리**한다.”

- 초기 분할 \(P=\{F,\ Q\setminus F\}\)
- worklist \(W\)에 보통 **더 작은 블록**부터 푸시(분할 비용 최소화)
- 어떤 블록 \(A\in W\)와 문자 \(a\)에 대해 **역상(Pre)**
  \(\mathrm{Pre}(A,a)=\{\,q\in Q\mid \delta(q,a)\in A\,\}\) 를 계산,
  이 집합이 다른 블록 \(Y\)를 **둘로** 쪼개면 \(Y\cap \mathrm{Pre}\), \(Y\setminus \mathrm{Pre}\)로 분할

### 절차(요약)

1) \(P\gets\{F,\ Q\setminus F\}\), \(W\gets\) (작은 쪽부터)
2) while \(W\neq\emptyset\):
   꺼낸 \(A\)에 대해 모든 \(a\in\Sigma\)에 대해 \(X=\mathrm{Pre}(A,a)\) 계산
   각 블록 \(Y\in P\)에 대해 \(Y\cap X\)와 \(Y\setminus X\)가 모두 비어있지 않으면 **분할**
   worklist에는 **원래 Y가 들어있었으면** 둘 다, 아니면 **더 작은 쪽**을 추가
3) 더 이상 분할이 안 되면 종료. \(P\)의 각 블록이 **최소 DFA**의 상태

### 복잡도

- 전형적으로 **\(O(|\Sigma|\,n\log n)\)**
- 상태 수 \(n=|Q|\), 알파벳 크기 \(|\Sigma|\)
- 대형 DFA/정규식 엔진/컴파일러에서 **사실상 표준**

### 장점/주의

- 빠르고 확장성 우수.
- 구현 시 **역전이 리스트(Reverse edges)** 또는 \(\mathrm{Pre}\) 계산 최적화가 성능에 중요.
- 불완전 DFA(전이 누락)는 최소화 전에 **sink 추가**로 완전화 필수.

---

## Moore’s Algorithm

### 핵심 아이디어

> “현재 분할에서 **같은 블록**에 있는 상태들이 **다음 전이의 블록 시퀀스**가 다르면 **분리**한다.”

- 초기 분할 \(P_0=\{F, Q\setminus F\}\)
- 각 상태 \(q\)에 대해 **서명(signature)** \(S(q)=(\mathrm{blk}(\delta(q,a_1)),\dots,\mathrm{blk}(\delta(q,a_k)))\)
  (여기서 \(\mathrm{blk}(x)\)는 \(x\)가 속한 블록 ID)
- 같은 블록 내에서 서명이 다르면 그 블록을 **쪼갬**
- 서명 안정화(분리가 더 이상 발생하지 않음)까지 반복

### 절차(요약)

1) \(P\gets\{F,\ Q\setminus F\}\)
2) do:
   각 상태에 대해 현재 \(P\) 기준 서명 계산 → 블록 내에서 서명 기준으로 재분할
   분할이 변했으면 반복
   (보통 ‘변화 없음’이 될 때까지)
3) 종료 시 \(P\)의 각 블록이 최소 DFA의 상태

### 복잡도/특징

- **\(O(|\Sigma|\,n^2)\)** 근방(상황 따라 더 느릴 수 있음)
- **구현이 매우 직관적**이고 코드가 간결 → 학습용/소형 DFA에 적합

---

## 동작 예제(완전 전개)

예제 DFA:

- 상태 \(Q=\{A,B,C,D,E\}\), 입력 \(\Sigma=\{0,1\}\)
- 수용 상태 \(F=\{C,E\}\)
- 전이:

| 상태 | 0 → | 1 → |
|---|---|---|
| A | B | C |
| B | A | D |
| C | E | C |
| D | E | D |
| E | E | E |

### 선행 처리

- 도달 불가 상태 없음(표에서 전부 연결)
- 완전 DFA(모든 전이 정의)

### Hopcroft로 최소화(핵심 스텝)

- 초기 \(P=\{\{C,E\},\{A,B,D\}\}\), \(W=\{\{C,E\}\}\) (작은 블록 우선)

1) \(A:=\{C,E\}\) 꺼냄.
**a=0:** \(\mathrm{Pre}(A,0)=\{q\mid \delta(q,0)\in\{C,E\}\}=\{C,D,E\}\)
  - \(Y=\{A,B,D\}\)와 교차 → \(Y\cap\mathrm{Pre}=\{D\}\), \(Y\setminus\mathrm{Pre}=\{A,B\}\) ⇒ **분할**
  - \(P=\{\{C,E\},\{D\},\{A,B\}\}\), \(W\)에는 작은 \(\{D\}\) 추가

**a=1:** \(\mathrm{Pre}(A,1)=\{A,C,E\}\)
  - \(Y=\{A,B\}\)와 교차 → \(\{A\}\) vs \(\{B\}\)로 **분할**
  - \(P=\{\{C,E\},\{D\},\{A\},\{B\}\}\), \(W\)에 (원래 Y가 W에 없으니) 작은 쪽 한 개 추가(임의로 \(\{A\}\))

2) 이제 \(W=\{\{D\},\{A\}\}\).
\(\{A\}\)를 꺼내도 더 이상 유효 분할 없음(모두 싱글턴). \(\{D\}\)도 동일.

**최종 \(P=\{\{C,E\},\{D\},\{A\},\{B\}\} \Rightarrow 4\) 상태**
- \(C,E\)는 **동치**(둘 다 이후 모든 입력을 항상 수용)
- \(A,B,D\)는 서로 구별됨(예: ‘1’은 \(A\to C\) 수용, \(B\to D\) 비수용 등)

### Moore로 최소화(핵심 스텝)

- \(P_0=\{\{C,E\},\{A,B,D\}\}\)

**서명 계산(블록 ID를 0:수용블록, 1:비수용블록로 가정)**

- 블록 1(\{A,B,D\}) 내:
  - \(A\): \((\mathrm{blk}(\delta(A,0))=\mathrm{blk}(B)=1,\ \mathrm{blk}(\delta(A,1))=\mathrm{blk}(C)=0)\Rightarrow (1,0)\)
  - \(B\): \((\mathrm{blk}(A)=1,\ \mathrm{blk}(D)=1)\Rightarrow (1,1)\)
  - \(D\): \((\mathrm{blk}(E)=0,\ \mathrm{blk}(D)=1)\Rightarrow (0,1)\)
  → 서로 서명이 달라 **세 개로 분할**: \(\{A\},\{B\},\{D\}\)

- 블록 0(\{C,E\}) 내:
  - \(C\): \((\mathrm{blk}(E)=0,\ \mathrm{blk}(C)=0)\Rightarrow (0,0)\)
  - \(E\): \((\mathrm{blk}(E)=0,\ \mathrm{blk}(E)=0)\Rightarrow (0,0)\)
  → 동일 서명이므로 **그대로 유지**

서명 안정화 ⇒ **최종 \(P=\{\{C,E\},\{A\},\{B\},\{D\}\}** (Hopcroft와 동일)

---

## 어떤 알고리즘을 언제 쓰나?

| 항목 | Hopcroft | Moore |
|---|---|---|
| 시간복잡도 | \(O(|\Sigma|\,n\log n)\) | \(O(|\Sigma|\,n^2)\) 근방 |
| 구현 난이도 | 중 | 쉬움 |
| 스케일 | 대형 DFA(정규식 엔진/컴파일러) | 소형 DFA, 교육·프로토타입 |
| 성능 팁 | 역전이/작은 블록 우선 | 간결한 서명·재분할 루프 |

> **실무 팁**
> 1) **도달 불가 제거 → 완전화(sink) → 최소화(Hopcroft) → (선택) canonical rename**
> 2) 보완: 곱구성(합·교·차) 후 **다시 최소화**하면 상태가 크게 줄어든다.

---

## Python 구현(교육용, 실행 가능)

> 주의: 학습용으로 간결하게 작성. 실서비스는 입력 검증·예외처리·메모리/속도 최적화 필요.

### 공통 DFA 유틸(도달 가능 제거, 완전화, 시뮬레이터)

```python
from collections import deque, defaultdict
from typing import Dict, Set, Tuple, Iterable

State = str
Sym = str

class DFA:
    def __init__(self,
                 states: Iterable[State],
                 alphabet: Iterable[Sym],
                 delta: Dict[Tuple[State, Sym], State],
                 start: State,
                 accepts: Iterable[State]):
        self.states: Set[State] = set(states)
        self.alphabet: Set[Sym] = set(alphabet)
        self.delta: Dict[Tuple[State, Sym], State] = dict(delta)
        self.start: State = start
        self.accepts: Set[State] = set(accepts)

    def run(self, s: str) -> bool:
        q = self.start
        for ch in s:
            if (q, ch) not in self.delta:
                return False
            q = self.delta[(q, ch)]
        return q in self.accepts

    def reachable(self) -> Set[State]:
        seen, dq = {self.start}, deque([self.start])
        adj = defaultdict(list)
        for (q,a), q2 in self.delta.items():
            adj[q].append(q2)
        while dq:
            u = dq.popleft()
            for v in adj[u]:
                if v not in seen:
                    seen.add(v); dq.append(v)
        return seen

    def prune_unreachable(self) -> "DFA":
        R = self.reachable()
        delta2 = {(q,a): q2 for (q,a), q2 in self.delta.items() if q in R and q2 in R}
        accepts2 = self.accepts & R
        return DFA(R, self.alphabet, delta2, self.start, accepts2)

    def complete_with_sink(self, sink_name: State = "__SINK__") -> "DFA":
        states2 = set(self.states)
        delta2 = dict(self.delta)
        if sink_name in states2:
            sink_name = max({sink_name} | {f"{sink_name}_{i}" for i in range(1000)}, key=len)  # avoid collision
        need_sink = False
        for q in list(states2):
            for a in self.alphabet:
                if (q,a) not in delta2:
                    need_sink = True
        if need_sink:
            states2.add(sink_name)
            for a in self.alphabet:
                delta2[(sink_name, a)] = sink_name
            for q in list(states2):
                for a in self.alphabet:
                    delta2.setdefault((q,a), sink_name)
        return DFA(states2, self.alphabet, delta2, self.start, self.accepts)
```

### Hopcroft 최소화

```python
def hopcroft_minimize(dfa: DFA) -> DFA:
    # 1) 전처리: 도달 불가 제거 + 완전화(여집합/차집합 등 위해 권장)
    D = dfa.prune_unreachable().complete_with_sink()

    Q, Σ = D.states, D.alphabet
    F, NF = set(D.accepts), D.states - set(D.accepts)

    # 초기 분할
    P = [F, NF] if NF else [F]
    # worklist: 작은 블록부터. 리스트로 두되 pop() 사용(스택처럼)
    W = [min(F, NF, key=len)] if F and NF else [F]

    # 역전이(pre) 효율화를 위한 reverse adjacency
    rev = { (q,a): set() for q in Q for a in Σ }
    for (q,a), q2 in D.delta.items():
        rev[(q2, a)].add(q)

    while W:
        A = W.pop()
        for a in Σ:
            # X = Pre(A, a)
            X = set()
            for q in A:
                X |= rev[(q, a)]

            newP = []
            for Y in P:
                inter = Y & X
                diff  = Y - X
                if inter and diff:
                    newP.extend([inter, diff])
                    # worklist 갱신
                    if Y in W:
                        W.remove(Y)
                        W.extend([inter, diff])
                    else:
                        W.append(inter if len(inter) <= len(diff) else diff)
                else:
                    newP.append(Y)
            P = newP

    # 블록 대표를 새 상태로
    rep = {}
    for block in P:
        b_rep = next(iter(block))
        for q in block:
            rep[q] = b_rep

    new_states = {rep[q] for q in Q}
    new_start  = rep[D.start]
    new_accepts= {rep[q] for q in F}
    new_delta  = {}
    for (q,a), q2 in D.delta.items():
        new_delta[(rep[q], a)] = rep[q2]

    return DFA(new_states, Σ, new_delta, new_start, new_accepts)
```

### Moore 최소화

```python
def moore_minimize(dfa: DFA) -> DFA:
    D = dfa.prune_unreachable().complete_with_sink()
    Q, Σ = D.states, D.alphabet

    # 초기 분할: 수용 / 비수용
    F, NF = set(D.accepts), D.states - set(D.accepts)
    P = [F, NF] if NF else [F]

    changed = True
    while changed:
        changed = False
        newP = []
        for block in P:
            # block 내부에서 signature로 분할
            sig_map = {}
            for q in block:
                sig = tuple(next((i for i,B in enumerate(P) if D.delta[(q,a)] in B), -1) for a in Σ)
                sig_map.setdefault(sig, set()).add(q)
            if len(sig_map) == 1:
                newP.append(block)
            else:
                # 여러 조각으로 쪼개짐
                parts = list(sig_map.values())
                newP.extend(parts)
                changed = True
        P = newP

    # 블록 대표로 축약
    rep = {}
    for block in P:
        r = next(iter(block))
        for q in block:
            rep[q] = r

    new_states = {rep[q] for q in Q}
    new_start  = rep[D.start]
    new_accepts= {rep[q] for q in D.accepts}
    new_delta  = {(rep[q], a): rep[D.delta[(q,a)]] for (q,a) in D.delta}

    return DFA(new_states, Σ, new_delta, new_start, new_accepts)
```

### 예제 DFA에 적용(두 알고리즘 결과 비교)

```python
# 예제 DFA 구성

states = {'A','B','C','D','E'}
alphabet = {'0','1'}
delta = {
    ('A','0'):'B', ('A','1'):'C',
    ('B','0'):'A', ('B','1'):'D',
    ('C','0'):'E', ('C','1'):'C',
    ('D','0'):'E', ('D','1'):'D',
    ('E','0'):'E', ('E','1'):'E',
}
start = 'A'
accepts = {'C','E'}

D = DFA(states, alphabet, delta, start, accepts)

Dh = hopcroft_minimize(D)
Dm = moore_minimize(D)

print("Hopcroft:", Dh.states, Dh.start, Dh.accepts)
print("Moore   :", Dm.states, Dm.start, Dm.accepts)

# 간단 동등성 체크: 길이 ≤ 6의 모든 문자열에서 수용 결과 비교

def eq_on_prefix(d1: DFA, d2: DFA, depth=6):
    from itertools import product
    Σ = sorted(d1.alphabet)
    for L in range(depth+1):
        for w in map("".join, product(Σ, repeat=L)):
            if d1.run(w) != d2.run(w):
                return False, w
    return True, ""

ok, witness = eq_on_prefix(Dh, Dm, 6)
print("두 결과 DFA 동등성:", ok, "| 반례:", witness or "(없음)")
```

> 기대 결과
> - 두 알고리즘 모두 **4상태**( \(\{C,E\}\) 통합 )의 DFA 도출
> - 짧은 길이 문자열 비교에서 **항상 일치**

---

## 표 채움(Table-Filling) 스케치

- **아이디어**: 상태쌍 \((p,q)\)가 **구별 가능(distinguishable)**한지 표로 표시
  1) 한쪽은 수용, 한쪽은 비수용이면 **표시**
  2) 표에 표시된 쌍으로부터 역추론: 어떤 \(a\)에 대해 \((\delta(p,a),\delta(q,a))\)가 표시되어 있으면 \((p,q)\)도 표시
  3) 고정점에 도달하면, **표시되지 않은 쌍**은 **동치** → 병합
- **복잡도**: \(O(|\Sigma|\,n^2)\) 내외, 간단하지만 대형 DFA에는 비효율

---

## 올바름(정당성)과 Myhill–Nerode 연결

- **정리**: 최소 DFA의 상태 수 = Myhill–Nerode 동치류 수
- Hopcroft/Moore는 모두 **동치관계의 몫집합**을 효과적으로 구해 **파티션 안정화**를 달성하는 절차
- Hopcroft의 worklist 전략은 **불필요한 분할 계산**을 줄여 \(\log n\) 팩터 개선

---

## 자주 하는 실수 & 베스트 프랙티스

1) **불완전 DFA**를 그대로 최소화 → 여집합/차집합에서 오답
   - **sink 추가로 완전화 후** 최소화할 것
2) **도달 불가 상태**를 남김 → 상태 수가 불필요하게 큼
   - 항상 **reachable prune** 먼저
3) **수용 상태를 하나의 블록**으로만 간주 → 오해
   - 초기엔 하나로 시작하지만, 전이 서명/Pre에 의해 **수용 블록도 분할** 가능
4) **상태 이름 비결정성**
   - 결과 DFA는 동형까지 유일. **대표 상태 고정 규칙**(lexicographic 등)을 적용해 **재현 가능한 이름**을 만들면 좋다.

---

## 복잡도·실전 팁 요약

- **Hopcroft**: 대규모(수만~수백만 상태)에도 실용. 역전이 준비 + 작은 블록 우선으로 빠름
- **Moore**: 코드 단순, 학습·테스트·소형 DFA에 매우 적합
- 파이프라인: **정규식 → ε-NFA(Thompson) → DFA(부분집합) → Hopcroft 최소화 → (옵션) canonical rename**

---

## 연습문제(해설 가이드)

1) **복합 DFA 최소화**:
   두 DFA \(D_1,D_2\)에 대해 교집합 DFA(곱구성) 만든 뒤 Hopcroft로 최소화하라. 곱구성 전·후 상태 수를 비교하라.
   *힌트*: 곱 상태 중 다수는 **도달 불가** → prune으로 큰 절감.

2) **수용 블록 분할 사례 만들기**:
   수용 상태가 둘 이상이고, 어떤 입력에서 **다른 블록**으로 향하는 전이가 있어 **수용 블록이 분할**되는 예를 설계하라.

3) **표 채움으로 동치 확인**:
   위 5상태 DFA에서 표 채움으로 \((C,E)\)가 **동치**임을 수작업으로 증명하라.

4) **완전화 중요성 실험**:
   불완전 DFA를 Moore로 최소화한 결과와, 완전화 후 최소화 결과를 비교해 차이를 관찰하라(여집합 적용 시 특히).

---

## 결론

- **Hopcroft**는 대규모 DFA에 최적화된 실무 표준(\(O(|\Sigma|\,n\log n)\)),
- **Moore**는 직관·단순 구현으로 교육/소형 DFA에 적합(\(O(|\Sigma|\,n^2)\)).
- 두 방법 모두 **동치**의 몫을 찾아 **최소 DFA**를 산출하며, 결과는 **언어적으로 동일**하고 **동형까지 유일**하다.
