---
layout: post
title: 형식언어와 오토마타 - Chomsky 정규형과 Greibach 정규형
date: 2025-07-13 19:20:23 +0900
category: 형식언어와 오토마타
---
# 정규형: Chomsky 정규형(CNF)과 Greibach 정규형(GNF)

## 왜 정규형(Normal Form)이 중요한가?

- **표준화**: 임의의 CFG를 간단한 규칙 형태로 **정규화**해 두면, 이론/증명/파서 설계가 쉬워진다.
- **알고리즘 친화성**
  - **CNF**: CYK(Chomsky–Younger–Kasami) 알고리즘의 **필수 전제**. 내부 DP 테이블 구조가 깨끗해진다.
  - **GNF**: 모든 생산 규칙이 **터미널로 시작** → **Top-down/예측 파싱**에 유리. 유도 단계마다 최소 1문자를 **즉시 소비**하므로 분석이 단순해진다.
- **동치 유지**: 변환은 원래 언어와 **언어적으로 동치**(ε 허용에 관한 **시작기호 특례**만 주의)다.

---

## Chomsky Normal Form (CNF)

### 정의

**모든** 생성 규칙이 다음 중 하나여야 한다.

1. $$A \to BC \quad(\text{단 } B,C \in V,\; B,C \neq S_0 \text{ 제약은 없음})$$
2. $$A \to a \quad(\text{단 } a \in \Sigma)$$
3. $$S_0 \to \varepsilon \quad(\text{언어에 } \varepsilon \text{이 포함될 때만 허용})$$

여기서 \(S_0\)은 (필요 시 도입하는) **새 시작 기호**다. 요컨대 **우변 길이는 2(전부 비터미널) 또는 1(단말 하나)**만 허용된다.

### 특징 & 효과

- **이항화(binary)**: 긴 우변을 2개씩 끊어내면 DP(예: CYK)가 구간 분할로 자연스럽게 동작.
- **단말 분리**: \(A\to aB\) 같은 혼합은 금지 → 단말을 **전용 비터미널**을 통해 간접 도입.
- **ε-규칙/단위(unit) 규칙 제거**와 **쓸모없는 기호 제거**가 전처리 핵심.

### CNF 변환 절차(표준 버전)

> 입력: CFG \(G=(V,\Sigma,R,S)\)
> 출력: 동치 CFG \(G_\text{CNF}\)

0. **새 시작기호 도입**: \(S_0\notin V\)에 대해 \(S_0\to S\) 추가. (이후 \(S_0\)를 시작기호로 사용)
1. **ε-규칙 제거**: \(S_0\to\varepsilon\) **가능성만 보존**, 그 외 \(A\to\varepsilon\) 제거.
   - **Nullable 집합** \( \mathsf{Null}=\{A\mid A\Rightarrow^*\varepsilon\} \) 계산.
   - 각 \(A\to \alpha\)에서 \(\alpha\)의 nullable 비터미널을 **모든 부분집합 조합**으로 삭제한 파생을 추가(단, 빈 우변은 허용하지 않고, \(A=S_0\)인 경우에만 \(\varepsilon\) 허용).
2. **단위 규칙 제거**: \(A\to B\) 꼴 제거.
   - **Unit-closure** \((A,B)\) 구해 \(A\to \gamma\) (단 \(\gamma\)가 단위가 아닌 우변) 추가.
3. **쓸모없는 기호 제거**:
   - **비생성(non-generating)** 제거: 단말만으로 유도 가능한 비터미널만 남김.
   - **비도달(unreachable)** 제거: \(S_0\)에서 도달 가능한 기호만 남김.
4. **단말 분리**: 우변 길이 ≥2인 규칙에서 등장하는 단말 \(a\)는 **전용 비터미널** \(A_a\to a\)를 만들고 \(a\)를 \(A_a\)로 치환.
5. **이항화**: \(A\to X_1X_2\cdots X_k\) (k≥3) 이면 새 기호 도입:
   - \(A\to X_1Y_1\), \(Y_1\to X_2Y_2\), …, \(Y_{k-3}\to X_{k-2}X_{k-1}\).

> **복잡도/크기 증가**: CNF 변환은 보통 **선형–다항**(규칙 수 기준 **\(O(|R|)\)~\(O(|R|+|V||\Sigma|)\)**) 수준 증가. 실무 크기에서도 감당 가능.

---

## Greibach Normal Form (GNF)

### 정의

모든 규칙이 **항상 터미널로 시작**한다.

$$
A \to a\,\alpha \quad(\text{단 } a\in\Sigma,\; \alpha\in V^*)
$$

- \(\alpha\)는 0개 이상의 비터미널. 즉 **첫 기호는 반드시 터미널**.
- \(S\to\varepsilon\)은 언어에 빈문자열이 포함될 때에 한해 허용(특례).

### 특징 & 효과

- **Top-down/LL 파싱 친화**: 다음 입력 심볼만 보고 규칙 후보를 좁히기 쉬움.
- **유도 단계마다 1문자 소비**: 길이 \(n\)의 입력을 정확히 **\(n\)단계**(+ε특례)로 유도 가능 → 시간/증명에 유리.
- **변환이 더 어렵고, 최악 크기 증가가 커질 수 있음(지수적 증가 가능)**.

### GNF 변환 절차(전형적 개요)

> 입력: ε-제거/단위 제거/쓸모없는 기호 제거가 끝난 CFG, 새 시작기호 \(S_0\) 포함
> 목적: 모든 규칙이 \(A\to a\alpha\)가 되도록

1. **비터미널 순서화**: \(V=\{A_1,\dots,A_n\}\).
2. **좌측 재귀 제거를 겸한 상향 치환**:
   - \(i=1..n\)에 대해, 각 \(A_i\)의 규칙을 검사.
   - \(A_i\to A_j\beta\) (단 \(j<i\)) 꼴이면, \(A_j\)의 우변으로 **치환 확장**하여 \(A_j\)를 제거(반복).
   - 그 결과 \(A_i\to A_k\beta\)이면 반드시 \(k\ge i\)가 되도록 정리.
3. **즉시 좌재귀 제거**(필요 시): \(A_i\to A_i\alpha\)와 \(A_i\to \beta\) 형태이면,
   - \(A_i\to \beta R_i\), \(R_i\to \alpha R_i \mid \varepsilon\).
4. **맨 앞을 터미널로 강제**: \(A_i\to B\gamma\) (비터미널이 맨 앞) 꼴이 남아 있으면 \(B\)의 규칙을 앞에서부터 다시 **적용/대체**하여 **맨 앞이 단말이 될 때까지** 교체.

> **주의**: 2–4단계가 서로 얽혀 반복될 수 있다. 구현 난이도가 CNF보다 높은 이유.

---

## 완전 예제: \(S\to aSb \mid \varepsilon\)

### CNF 변환

원문법:
```
S → a S b | ε
```

**Step 0 (새 시작기호)**: \(S_0\to S\)

**Step 4 (단말 분리)**: 단말 전용 비터미널 도입
\(A\to a,\; B\to b\)
```
S0 → S
S  → A S B | ε
A  → a
B  → b
```
- \(S\to ASB\)는 이미 이항(비터미널 3개라서 이항 아님!) → **이항화 필요**

**Step 5 (이항화)**: \(S\to A S B\)는 3개이므로 중간기호 \(D\) 도입
\(S\to A D,\; D\to S B\)

**최종 CNF**:
```
S0 → S | ε   (ε 허용 시에만. 원래 언어는 ε 포함하므로 OK)
S  → A D
D  → S B
A  → a
B  → b
```
- 모든 우변이 길이 1(단말) 또는 길이 2(비터미널×2). CNF 완료.

> **메모**: 일부 교재는 \(S_0\to\varepsilon\)만 허용하고 \(S\)에는 ε 금지되도록 조정한다. 위 구성은 \(S_0\)를 시작으로 사용함으로써 표준을 준수.

### GNF 변환

목표: 모든 규칙이 \(A\to a\alpha\).

현재(ε/단위 제거 가정 후) 규칙 집합:
```
S → a S b | ε
```
- ε을 유지하려면 \(S_0\) 특례를 쓰는 게 일반적: \(S_0→S \mid ε\), \(S→a S b\)만 남김.

터미널 분리 + 앞쪽 터미널 유지:
- \(a\)가 이미 앞에 있어 GNF 형태에 **근접**.
- 뒤쪽 \(b\)는 단말이지만 **맨 앞만 단말**이면 되므로 **GNF 요건 만족**.

**GNF(ε 제외)**
```
S → a S B
B → b
```
- 또는 \(S\to a S b\) 그대로 두고, “맨 앞이 터미널” 요건만 확인해도 GNF 합격.
- ε을 허용해야 하면 시작기호 특례로 \(S_0\to ε\mid S\) 추가.

---

## 복합 예제: 괄호 올바른 열 \(S\to SS \mid (S) \mid ε\)

이 문법은 **좌공통**, **길이>2 우변**, **ε**가 모두 있어서 정규화 훈련용으로 적합.

### CNF 변환

0) **새 시작**: \(S_0\to S\)

1) **ε-제거(특례 보존)**
- Nullable: \(S\) (왜냐하면 \(S\Rightarrow^*\varepsilon\)).
- \(S→SS\)에서 nullable 조합: \(S→S\) (단위), \(S→S\) (중복), \(S→\varepsilon\) (제거 대상, 시작 특례 제외) …
- 체계적으로 하면 **ε는 \(S_0\)에서만 허용**하고 나머지는 제거.

2) **단위 제거**: \(S→S\)류 제거.

3) **쓸모없는 제거**: (이 예에서는 대부분 생성/도달 가능)

4) **단말 분리**: `(`, `)`를 \(L→(\), \(R→)\)로 대체. \( (S) \)는 \(L S R\)로 치환.

5) **이항화**: \(S→SS\)는 이항이라 OK. \(S→LSR\)는 3개이므로
- \(S→L X\), \(X→S R\)

6) **정리**(ε 특례는 \(S_0\)에서만):
```
S0 → S | ε
S  → S S | L X
X  → S R
L  → (
R  → )
```
- 모두 우변 길이 2(전부 비터미널) 또는 1(단말). CNF 완료.

### GNF 변환(핵심만)

- \(S→SS\)는 **비터미널로 시작**하므로 GNF 위배 → **맨 앞에 단말이 오도록** 반복 치환 필요.
- \(S→(S)\)처럼 **‘(’로 시작**하는 파생을 전개해서 \(S\)의 규칙이 전부 `(`로 시작하도록 만들 수 있다.
- 실무적 GNF 변환은 치환 폭발이 있을 수 있으므로, 보통 **증명/이론적 용도**에서만 수행.

**직관적 GNF 형태 예시(개념)**:
```
S → ( S ) S | ε
```
- 실제 GNF 정의에 맞추려면 단말 전용 비터미널을 넣고, 모든 규칙의 **첫 심볼이 단말**이 되게끔 \(S\) 오른쪽의 선두를 끝까지 전개한다.
- 완전한 전개는 길어지므로 여기서는 절차만 요약:
  1) \(S→SS\)를 우선 \(S\)의 `( … )`로 시작하는 대안들로 치환
  2) 좌재귀/단위 제거
  3) 모든 규칙 선두가 `(`가 되도록 반복 치환

---

## 이론 스냅샷 및 증명 스케치

### CNF 가능 정리

> **정리**: 임의의 CFG \(G\)에 대해, \(L(G)=\{\varepsilon\}\) 이면 \(S\to\varepsilon\) 하나만 갖는 CNF,
> 아니면 시작기호 특례를 둔 **CNF** \(G'\)가 존재하며 \(L(G')=L(G)\)이다.

- **스케치**: ε/단위/쓸모없는 제거 후, 단말 치환 및 이항화를 통해 규칙을 표준화. 시작기호 특례로 ε 보존.

### GNF 가능 정리

> **정리(Greibach)**: ε을 제외(또는 시작 특례로 분리)하고 단위/쓸모없는 제거를 마친 CFG는 **GNF로 변환** 가능.

- **핵심 아이디어**: 비터미널 순서화 + 상향 치환으로 선두 비터미널을 **낮은 인덱스의 규칙으로만** 바꾸고, 결국 선두 터미널이 오게끔 반복. 좌재귀 제거가 중간중간 반드시 필요.

### 복잡도

- **CNF**: 규칙 수가 보통 **선형~다항 증가**. 실무에서 안전.
- **GNF**: 최악 **지수적 증가** 가능(치환 폭발). 주로 **이론 용도**.

---

## 실전 체크리스트

### CNF 검사 체크리스트

- [ ] 시작기호 \(S_0\)만 \(\varepsilon\) 허용?
- [ ] 모든 규칙이 **(비터미널×비터미널)** 혹은 **(단말 하나)**?
- [ ] 단말이 우변 길이≥2 규칙에 **직접** 나오지 않는가?
- [ ] 단위 규칙/ε-규칙(특례 제외) 없는가?
- [ ] 도달 불가능/비생성 비터미널 제거했는가?

### GNF 검사 체크리스트

- [ ] 모든 규칙이 **터미널로 시작**하는가?
- [ ] 이어지는 심볼은 0개 이상의 비터미널뿐인가?
- [ ] (언어가 ε 포함 시) 시작기호 특례 \(S_0\to\varepsilon\)만 존재하는가?
- [ ] 좌재귀/단위 규칙이 제거되었는가?

---

## 알고리즘 의사코드 (간결판)

### CNF 변환(의사코드)

```text
CNF-Transform(G=(V,Σ,R,S)):
  introduce S0 → S

  // ε-제거
  Null := computeNullable(V,R)
  R' := {}
  for each A→α in R:
    for each subset Z of positions in α that are nullable:
      β := α with symbols at Z removed
      if β ≠ ε: add A→β to R'
      else if A==S0: add S0→ε to R'
  R := R'

  // 단위 제거
  Unit := computeUnitPairs(V,R) // A⇒* B via unit rules
  R'' := {}
  for each (A,B) in Unit:
    for each B→γ in R with |γ|≠1 or γ ∉ V:
      add A→γ to R''
  R := R''

  // 쓸모없는 제거: non-generating, unreachable
  R := removeUseless(V,Σ,R,S0)

  // 단말 분리
  for each rule A→X1...Xk, k≥2:
    replace each terminal a by a fresh Va with Va→a
    update rule with Va inplace

  // 이항화
  for each A→X1...Xk with k≥3:
    introduce Y1,...,Yk-3
    add A→X1 Y1; Y1→X2 Y2; ...; Yk-3→Xk-1 Xk
    remove A→X1...Xk

  return (V∪fresh, Σ, R, S0)
```

### GNF 변환(의사코드; 개요)

```text
GNF-Transform(G):
  S0→S introduce, remove ε (S0 특례 외), remove unit, remove useless

  order nonterminals A1,...,An

  for i in 1..n:
    // 상향 치환: 선두 Aj (j<i) 제거
    repeat
      for each rule Ai→Aj β with j<i:
        replace Ai→Aj β by { Ai→δ β | Aj→δ in R }
    until no change

    // 즉시 좌재귀 제거(있을 때만)
    let Rα = { α | Ai→Ai α } and Rβ = { β | Ai→β and β does not start with Ai }
    if Rα nonempty:
      create new Ri
      Ai→ (for each β in Rβ) β Ri
      Ri→ (for each α in Rα) α Ri | ε

  // 선두를 터미널로 강제
  repeat
    for each rule A→B γ with B∈V:
      replace by { A→δ γ | B→δ in R }
  until all rules start with terminal

  return G in GNF
```

> 구현 시 **중복 폭발** 방지(메모이제이션/집합화)와 **순환** 안전장치가 중요.

---

## 파이썬 샘플 코드(학습/실험용; 축약/교육용 구현)

> **목표**: (1) 간단한 CFG 표현, (2) CNF 변환 핵심 단계(ε/단위/쓸모없는/단말분리/이항화), (3) CNF/GNF 검사기
> **주의**: 실전 완전성보다 **학습 가독성**에 초점. 대형 문법/복잡 사례는 추가 보강 필요.

```python
# cfg_cnf_gnf.py — 교육용 축약 구현

from collections import defaultdict, deque

class CFG:
    def __init__(self, V, Sigma, R, S):
        self.V = set(V)           # nonterminals
        self.Sigma = set(Sigma)   # terminals
        self.R = defaultdict(set) # A -> set of tuples(rhs symbols)
        for A, rhss in R.items():
            for rhs in rhss:
                self.R[A].add(tuple(rhs))
        self.S = S                # start (will inject S0)

    def clone(self):
        return CFG(self.V.copy(), self.Sigma.copy(),
                   {A: [list(r) for r in rhs] for A, rhs in self.R.items()},
                   self.S)

def introduce_S0(g):
    S0 = "_S0"
    while S0 in g.V or S0 in g.Sigma:
        S0 = "_" + S0
    g.V.add(S0)
    g.R[S0].add((g.S,))
    g.S = S0
    return g

def compute_nullable(g):
    nullable = set()
    changed = True
    while changed:
        changed = False
        for A in g.V:
            if A in nullable: continue
            for rhs in g.R[A]:
                if len(rhs)==0 or all((X in nullable) for X in rhs if X in g.V):
                    nullable.add(A); changed = True; break
    return nullable

def remove_epsilon(g):
    # keep S0->ε if original language had ε; here we conservatively allow S0->ε iff S0 nullable
    nullable = compute_nullable(g)
    newR = defaultdict(set)
    for A in g.V:
        for rhs in g.R[A]:
            # enumerate subsets of nullable nonterminals in rhs
            positions = [i for i,X in enumerate(rhs) if X in nullable]
            # bitmask enumeration
            m = len(positions)
            for mask in range(1<<m):
                out = []
                for i,X in enumerate(rhs):
                    if i in positions:
                        j = positions.index(i)
                        if (mask>>j)&1:  # drop
                            continue
                    out.append(X)
                if len(out)>0:
                    newR[A].add(tuple(out))
                else:
                    if A == g.S:  # only start may produce ε
                        newR[A].add(tuple())
    g.R = newR
    return g

def remove_unit(g):
    # unit closure
    unit = {A:{A} for A in g.V}
    changed = True
    while changed:
        changed = False
        for A in g.V:
            for rhs in g.R[A]:
                if len(rhs)==1 and rhs[0] in g.V:
                    B = rhs[0]
                    if B not in unit[A]:
                        unit[A].add(B); changed = True
                        unit[A] |= unit[B]
    newR = defaultdict(set)
    for A in g.V:
        for B in unit[A]:
            for rhs in g.R[B]:
                if not (len(rhs)==1 and rhs[0] in g.V):  # skip unit
                    newR[A].add(rhs)
    g.R = newR
    return g

def remove_useless(g):
    # generating
    gen = set()
    changed = True
    while changed:
        changed = False
        for A in g.V:
            if A in gen: continue
            for rhs in g.R[A]:
                if all((X in g.Sigma) or (X in gen) for X in rhs):
                    gen.add(A); changed = True; break
    # reachable
    reach = {g.S}
    q = deque([g.S])
    while q:
        A = q.popleft()
        for rhs in g.R[A]:
            for X in rhs:
                if X in g.V and X not in reach:
                    reach.add(X); q.append(X)
    # prune
    g.V = g.V & gen & reach
    g.R = {A: set(r for r in rhss if all((X in g.Sigma) or (X in g.V) for X in r))
           for A, rhss in g.R.items() if A in g.V}
    return g

def terminal_split(g):
    # introduce Va -> a for terminals inside long RHS
    mapping = {}
    for a in g.Sigma:
        Va = f"_T_{a}"
        while Va in g.V or Va in g.Sigma:
            Va = "_" + Va
        mapping[a] = Va
    added = {}
    for a,Va in mapping.items():
        added[Va] = {(a,)}
    # rewrite
    newR = defaultdict(set)
    for A, rhss in g.R.items():
        for rhs in rhss:
            if len(rhs)>=2:
                rhs2 = []
                for X in rhs:
                    if X in g.Sigma:
                        rhs2.append(mapping[X])
                    else:
                        rhs2.append(X)
                newR[A].add(tuple(rhs2))
            else:
                newR[A].add(rhs)
    g.R = newR
    for Va, rhss in added.items():
        g.V.add(Va)
        if Va not in g.R: g.R[Va] = set()
        g.R[Va] |= rhss
    return g

def binarize(g):
    newR = defaultdict(set)
    fresh_id = 0
    def fresh():
        nonlocal fresh_id
        while True:
            Y = f"_B{fresh_id}"
            fresh_id += 1
            if Y not in g.V and Y not in g.Sigma:
                return Y
    for A, rhss in g.R.items():
        for rhs in rhss:
            if len(rhs)<=2:
                newR[A].add(rhs)
            else:
                Xs = list(rhs)
                Yprev = None
                # chain A->X1 Y1, Y1->X2 Y2, ..., Yk-3->Xk-1 Xk
                Y = fresh(); g.V.add(Y)
                newR[A].add((Xs[0], Y))
                for i in range(1, len(Xs)-2):
                    Y2 = fresh(); g.V.add(Y2)
                    newR[Y].add((Xs[i], Y2))
                    Y = Y2
                newR[Y].add((Xs[-2], Xs[-1]))
    g.R = newR
    return g

def to_cnf(g0):
    g = g0.clone()
    introduce_S0(g)
    remove_epsilon(g)
    remove_unit(g)
    remove_useless(g)
    terminal_split(g)
    binarize(g)
    return g

def is_cnf(g):
    for A, rhss in g.R.items():
        for rhs in rhss:
            if len(rhs)==0:
                if A!=g.S: return False
            elif len(rhs)==1:
                if rhs[0] not in g.Sigma: return False
            elif len(rhs)==2:
                if rhs[0] not in g.V or rhs[1] not in g.V: return False
            else:
                return False
    return True

def is_gnf(g):
    # 허용: 시작기호 A→ε (특례)
    for A, rhss in g.R.items():
        for rhs in rhss:
            if len(rhs)==0:
                if A!=g.S: return False
            else:
                if rhs[0] not in g.Sigma: return False
                for X in rhs[1:]:
                    if X not in g.V: return False
    return True

# --- quick demo ---

if __name__ == "__main__":
    # S -> a S b | ε
    V = {"S"}
    Sigma = {"a","b"}
    R = {"S":[["a","S","b"], []]}
    G = CFG(V,Sigma,R,"S")
    Gcnf = to_cnf(G)
    print("Start:", Gcnf.S)
    print("CNF?")
    print(is_cnf(Gcnf))
    for A in sorted(Gcnf.R):
        print(A, "->", [" ".join(r) if r else "ε" for r in Gcnf.R[A]])
```

> 위 코드는 **학습용 스켈레톤**이다.
> - CNF 경로는 실무 소형 문법에서 잘 동작한다.
> - GNF 변환은 구현량이 많아(치환 폭발 제어, 순환/중복 관리) 여기선 **검사기**만 제공했다.

---

## 실전 팁/자주 겪는 함정

- **\(S\)가 우변에 등장**하는 경우: 꼭 **새 시작기호 \(S_0\)**를 도입해 \(S_0\to S\)로 시작. ε-제거/단위-제거 시 **언어 보존**에 필수.
- **ε-제거**: 시작기호 **특례**를 제외하면 ε을 모두 제거. Nullable 조합 생성 시 **공집합/중복** 주의.
- **단위 규칙 사이클**: \(A\Rightarrow^*A\) 같은 사이클이 있으면 **Unit-closure**에서 반드시 막아야 함(이미 방문한 쌍은 스킵).
- **단말 분리**: \(A\to aB\)는 CNF 위반. 반드시 \(A_a\to a\), \(A\to A_a B\)로 대치.
- **이항화**: 체인 도중 새 비터미널 이름 충돌 방지.
- **GNF**: 변환이 복잡하고 **규칙 수 폭증** 가능. 보통 **증명**이나 **이론적 성질** 확인용.

---

## 요약 표

| 항목 | CNF | GNF |
|---|---|---|
| 규칙 형태 | \(A\to BC\), \(A\to a\), \(S_0\to\varepsilon\) | \(A\to a\alpha\) (선두는 항상 단말), \(S_0\to\varepsilon\) 특례 |
| 주 용도 | **CYK**/정형 증명/하한·상한 논증 | **Top-down/예측 파싱**, 길이-단계 대응 |
| 변환 난이도 | 비교적 쉬움 | 상대적으로 어려움 |
| 크기 증가 | 선형~다항 | 최악 지수 |
| 실무 체감 | 파서·검증기 내부에서 빈번 | 교과/증명·특수 파서 연구 |

---

## 추가 연습 과제

1. 다음 문법을 **CNF**로 바꾸고, 문자열 `(()())`의 CYK **테이블을 손으로** 채워 membership을 보이기.
   \(S\to SS\mid (S)\mid \varepsilon\).
2. 아래 문법을 **GNF**로 바꾸는 과정을 단계별로 써보기(선두 터미널 강제):
   \(E\to E+T\mid T,\; T\to T*F\mid F,\; F\to (E)\mid id\).
3. **반례 찾기**: CNF로 변환한 뒤에도 언어가 같음을, 임의 짧은 길이 문자열 집합(길이 ≤ 4)으로 **양방향 포함**을 점검(테스트 케이스 설계).

---

## 마무리

- **CNF**: “DP/증명에 좋은 형태” — ε/단위/쓸모없는 제거 → **단말 분리** → **이항화**.
- **GNF**: “Top-down 친화형” — 선두를 터미널로 만들 때까지 **치환/좌재귀 제거**를 반복.
- 두 정규형 모두 **언어 동치**를 유지(단, ε은 시작 특례로 관리).
- 실전 구현은 **중복 폭발/순환**에 유의하고, 작은 도우미(검사기, CNF 변환기)부터 확장하는 전략이 안전하다.
