---
layout: post
title: 형식언어와 오토마타 - Moore 기계와 Mealy 기계의 동치성
date: 2025-08-15 19:20:23 +0900
category: 형식언어와 오토마타
---
# Moore 기계와 Mealy 기계의 **동치성**

Moore와 Mealy는 **유한 상태 변환기(FST)** 의 두 대표 모델이다.
핵심 차이는 **출력을 언제 내느냐**(Mealy: 전이 시점, Moore: 도착 상태)뿐이고, **고정 지연(shift) 보정**을 허용하면 **동일한 변환 능력**을 갖는다.

---

## 0. 표기와 실행 의미

- **Mealy**
  $$
  M=(Q,\Sigma,\Gamma,\delta,\lambda,q_0),\quad
  \delta:Q\times\Sigma\to Q,\;\; \lambda:Q\times\Sigma\to\Gamma
  $$
  입력 $a$ 를 읽는 **그 스텝**에 출력 $\lambda(q,a)$ 방출.

- **Moore**
  $$
  N=(P,\Sigma,\Gamma,\delta',\rho,p_0),\quad
  \delta':P\times\Sigma\to P,\;\; \rho:P\to\Gamma
  $$
  $p\xrightarrow{a}p'$ 로 이동한 **다음 상태**의 출력 $\rho(p')$ 방출. (관례) **초기 출력** $\rho(p_0)$ 포함.

### 출력 스트림의 길이
- Mealy(전통 1:1): 입력 길이 $n$ 이면 **출력도 $n$**.
- Moore(초기 출력 포함 관례): **출력 길이 $n+1$**. 비교 시 초기 출력 처리(§4)가 중요.

---

## 1. 동치의 의미 — 엄밀 vs 지연(shift) 동치

입력 $x=a_1\cdots a_n$ 에 대해 Mealy 출력 $y_M\in\Gamma^n$, Moore 출력 $y_N\in\Gamma^{n+1}$.

- **엄밀 동치(strict)**: 두 기계의 출력 스트림이 **완전히 동일**(길이·위치 모두 일치).
- **지연 동치(shift-equivalence)**: 고정 오프셋 $k\in\mathbb{Z}$ (실무에선 $k\in\{0,\pm1\}$)에 대해
  $$
  \mathrm{Shift}_k(y_A)=y_B
  $$
  가 모든 입력에서 성립. Moore vs Mealy 비교에선 보통 **$k=+1$**(Moore가 한 스텝 길다).

> 본 글의 변환·검증은 **지연 동치**를 표준으로 가정한다. 즉 **Mealy 출력**과 **Moore의 (초기 출력 제거) 출력**이 같으면 동치로 본다.

---

## 2. **Moore → Mealy** 변환 — 즉시 반응형으로 (상태 수 보존)

**핵심 아이디어**: Mealy 전이 출력은 “도착 상태의 출력”과 동일하게 두면 된다.

### 공식
$$
\begin{aligned}
Q&:=P,\quad q_0:=p_0,\\
\delta(q,a)&:=\delta'(q,a),\\
\lambda(q,a)&:=\rho\!\big(\delta'(q,a)\big).
\end{aligned}
$$

- 어떤 입력 $a_i$ 에서 Mealy가 내는 $\lambda(q_{i-1},a_i)$ 는, Moore가 **그 스텝 후** 내는 $\rho(q_i)$ 와 동일.
- **상태 수**: 그대로. **시간/공간**: 전이 수만큼 상수 작업 → $O(|Q||\Sigma|)$.

### 정확성(식)
$$
\forall x=a_1\cdots a_n:\;
\underbrace{\lambda(q_0,a_1)\cdots\lambda(q_{n-1},a_n)}_{\text{Mealy}}
=
\underbrace{\rho(q_1)\cdots\rho(q_n)}_{\text{Moore(초기 제외)}}.
$$
→ **Moore의 초기 출력** $\rho(p_0)$ 만 버리면 스트림 정확히 일치.

---

## 3. **Mealy → Moore** 변환 — 안정 출력형으로 (상태 세분화 필요)

**도전점**: Mealy의 출력은 $(\text{상태},\text{입력})$에 의존. Moore는 **상태만**으로 출력이 정해져야 한다.
따라서 “**다음에 내보낼 출력**”을 상태에 **흡수**한다.

### 표준 구성
새 상태 집합
$$
P:=\{(q,y)\mid q\in Q,\ y\in\Gamma\}\quad\text{(도달 쌍만 생성)}
$$
출력 $\rho((q,y)):=y$, 전이
$$
\delta'((q,y),a):=\big(\delta(q,a),\ \lambda(q,a)\big).
$$
초기 상태는 보통 **더미 출력** $\bot$ 로 $(q_0,\bot)$ 를 두거나, 지연 동치를 가정해 **초기 출력은 무시**.

- **상태 수 경계**: 최악에 $|P|\le |Q|\cdot|\Gamma|$ (타이트, §6 예).
- **정확성**: Mealy의 $i$번째 출력 $\lambda(q_{i-1},a_i)$ 가, Moore의 도착 상태 $(q_i,\lambda(q_{i-1},a_i))$ 의 $\rho$ 와 일치.

### 의사코드
```text
queue := [(q0, ⊥)]; mark ρ(q0,⊥)=⊥
while queue not empty:
  (q,y) := pop()
  for a in Σ:
    q' := δ(q,a); y' := λ(q,a)
    add transition ( (q,y) --a--> (q',y') ), and set ρ(q',y') = y'
    if (q',y') unseen: push((q',y'))
# (초기 출력 ⊥는 지연 동치 관례상 비교에서 버리거나, 상수 프리엠블로 처리)
```

---

## 4. 지연(shift) 정렬 — 출력 길이·시점 맞추기

### 4.1 정렬 함수
$$
\mathrm{Shift}_{+1}(b_0 b_1\cdots b_n)=b_1\cdots b_n,\quad
\mathrm{Shift}_{-1}(c_1\cdots c_n)=\_\,c_1\cdots c_n
$$
(필요 시 더미 기호 $\_$).

### 4.2 비교 규칙(권장)
- **Moore vs Mealy**: Moore의 **초기 출력 제거**(= $\mathrm{Shift}_{+1}$) 후 비교.
- **Moore vs Moore**: 두 쪽 모두 같은 초기 출력 관례를 쓰거나, 둘 다 **초기 제거**.
- **Mealy vs Mealy**: 엄밀(shift 0) 비교.

---

## 5. 상태 수 **경계**(상·하한)와 의미

- **Moore→Mealy**: **상태 수 보존**. (일반적으로는 **최소 상태 수가 같거나 더 작아질 수** 있다 — 출력 다양성을 전이로 이동)
- **Mealy→Moore**: **최대 $|Q|\cdot|\Gamma|$**.
  - **상한 타이트**: 각 상태 $q\in Q$ 에 대해 모든 $a\in\Sigma$ 에서 **서로 다른** 출력이 나오고, 그 패턴이 상태별로 구분되어야 하는 예를 만들면 실제로 $|Q||\Gamma|$ 까지 필요.
  - **실무 의미**: 출력 알파벳이 크거나(태그·벡터 출력), 상태별 전이 출력 다양성이 크면 **Moore 전환 시 상태 폭증**이 온다.
    이때는 **Mealy 유지** 또는 **부분-하이브리드**(중간 스테이지만 Moore) 전략이 낫다.

---

## 6. 작동 예시(소형 케이스 2종)

### 6.1 Moore → Mealy: 패리티 발생기
```text
Moore
Σ=Γ={0,1}
상태/출력: qE/0, qO/1
전이: qE--0→qE, qE--1→qO, qO--0→qO, qO--1→qE
```
- 변환: Mealy의 λ(q,a)=ρ(δ'(q,a)) → **초기 출력(0)** 을 버리면 스트림 동일.

### 6.2 Mealy → Moore: '01' 검출기
- **Mealy**: 직전이 0이고 현재 1이면 **바로** 1 출력.
- **Moore**: “다음에 낼 출력”을 상태에 담아 **한 스텝 뒤**에 1 출력(지연 +1).
- 상태는 $(\text{원상태},\text{직전 출력})$ 형태로 세분 → 보통 2~3배 이내, 최악 $|\Gamma|$ 배.

---

## 7. 동치성(Equivalence) 판정 — 제품 기계 + 반례 산출

### 7.1 알고리즘(지연 동치 버전)
1) **정렬**: Moore 쪽에서 **초기 출력 제거**(또는 Mealy에 더미 프리엠블 추가).
2) **제품 상태** $(p,q)$ 위에서 같은 입력을 동시에 주입.
3) 매 스텝 도착 시점에 **출력 비교**. 다르면 **최단 반례 입력**을 보고.
4) 전체 상태 공간 탐색에 차이 없음 → **동치**.

- 시간/공간: 결정적·총기계 기준 **선형**(상태·전이 수에 비례).

### 7.2 최소화와의 연결
- **Mealy 최소화**: 출력이 **전이**에 붙으므로, 같은 전이를 취해도 출력이 다르면 **그 전이 라벨**만 다르고 상태는 동일할 수 있다.
- **Moore 최소화**: 출력이 **상태**에 붙으므로, “같은 목적 상태이지만 서로 다른 출력”을 위해 **상태를 세분화**해야 한다.
→ 바로 이 차이가 Mealy→Moore 시 상태 수 급증의 원인.

---

## 8. 파이썬 구현 — 모델, 변환, 동치(shift), 최소화(요약)

> 교육·검증용 간결 구현. 부분 기계는 **Sink** 로 총기계화 권장. (초기 출력 포함 관례)

```python
from collections import deque, defaultdict
from typing import Dict, Tuple, Set, Hashable, List, Optional

State = Hashable; Sym = Hashable; Out = Hashable

# ---------- Moore ----------
class DMoore:
    def __init__(self, Q:Set[State], Σ:Set[Sym], Γ:Set[Out],
                 δ:Dict[Tuple[State,Sym], State],
                 ρ:Dict[State,Out], q0:State):
        self.Q, self.Σ, self.Γ = set(Q), set(Σ), set(Γ)
        self.δ, self.ρ, self.q0 = dict(δ), dict(ρ), q0

    def reachable(self) -> Set[State]:
        seen, dq = set(), deque([self.q0])
        while dq:
            q = dq.popleft()
            if q in seen: continue
            seen.add(q)
            for a in self.Σ:
                nq = self.δ.get((q,a))
                if nq is not None: dq.append(nq)
        return seen

    def totalize(self, sink:State="⟂", sink_out:Out=("<sink>",)):
        if sink not in self.Q:
            self.Q.add(sink); self.ρ[sink]=sink_out
        for a in self.Σ:
            self.δ.setdefault((sink,a), sink)
        for q in list(self.Q):
            for a in self.Σ:
                self.δ.setdefault((q,a), sink)

    def minimize(self):
        # Trim
        R = self.reachable()
        Q = [q for q in self.Q if q in R]; Σ = list(self.Σ)
        # 초기 분할: 출력
        groups = defaultdict(list)
        for q in Q: groups[self.ρ[q]].append(q)
        blocks = list(groups.values())
        # 정제
        changed = True
        while changed:
            changed, new_blocks = False, []
            where = {}
            for i,B in enumerate(blocks):
                for s in B: where[s]=i
            for B in blocks:
                bucket = defaultdict(list)
                for q in B:
                    sig = tuple(where[self.δ[(q,a)]] for a in Σ)
                    bucket[sig].append(q)
                if len(bucket)==1:
                    new_blocks.append(B)
                else:
                    changed = True
                    new_blocks.extend(bucket.values())
            blocks = new_blocks
        # 몫 기계
        rep = {s:i for i,B in enumerate(blocks) for s in B}
        δm, ρm = {}, {}
        for i,B in enumerate(blocks):
            q0 = B[0]; ρm[i] = self.ρ[q0]
            for a in Σ: δm[(i,a)] = rep[self.δ[(q0,a)]]
        return DMoore(set(range(len(blocks))), set(Σ), set(ρm.values()), δm, ρm, rep[self.q0])

# ---------- Mealy ----------
class DMealy:
    def __init__(self, Q:Set[State], Σ:Set[Sym], Γ:Set[Out],
                 δ:Dict[Tuple[State,Sym], State],
                 λ:Dict[Tuple[State,Sym], Out], q0:State):
        self.Q, self.Σ, self.Γ = set(Q), set(Σ), set(Γ)
        self.δ, self.λ, self.q0 = dict(δ), dict(λ), q0

# ---------- Conversions ----------
def moore_to_mealy(N:DMoore) -> DMealy:
    δ, λ = {}, {}
    for (q,a), nq in N.δ.items():
        δ[(q,a)] = nq
        λ[(q,a)] = N.ρ[nq]  # 도착 상태 출력
    Γ = set(N.ρ.values())
    return DMealy(set(N.Q), set(N.Σ), Γ, δ, λ, N.q0)

def mealy_to_moore(M:DMealy, init_out:Out=("<init>",)) -> DMoore:
    Σ = list(M.Σ)
    Qp, ρp, δp = set(), {}, {}
    # (q,y) 생성 및 전이 구성
    def add(q,y):
        if (q,y) not in Qp:
            Qp.add((q,y)); ρp[(q,y)] = y
            for a in Σ:
                q2 = M.δ[(q,a)]; y2 = M.λ[(q,a)]
                δp[((q,y),a)] = (q2,y2)
                add(q2,y2)
    # 시작
    add(M.q0, init_out)
    return DMoore(Qp, set(Σ), set(ρp.values()), δp, ρp, (M.q0, init_out))

# ---------- Equivalence with shift ----------
def equal_moore_vs_mealy(N:DMoore, M:DMealy, shift:int=+1):
    """
    shift=+1: Moore 초기 출력 제거 후 Mealy와 비교(권장).
    shift=0 : 엄밀 비교(길이 동일 관례에서만).
    반환: (is_equal, counterexample_input or None)
    """
    from collections import deque
    dq, seen = deque(), set()
    # 초기 출력 정렬 체크
    if shift==0:
        # 엄밀: Moore의 초기 출력과 '가상의 Mealy 사전 출력'을 비교해야 한다.
        # 보통은 shift=+1을 쓰자. 여기서는 초기 출력은 무시(지연 동치 전제).
        pass
    dq.append((N.q0, M.q0, []))
    while dq:
        p, q, w = dq.popleft()
        if (p,q) in seen: continue
        seen.add((p,q))
        for a in N.Σ:
            p2, q2 = N.δ[(p,a)], M.δ[(q,a)]
            outN = N.ρ[p2]                        # Moore: 도착 출력
            outM = M.λ[(q,a)]                     # Mealy: 즉시 출력
            if outN != outM:
                return False, w+[a]               # 최단 반례
            dq.append((p2,q2,w+[a]))
    return True, None
```

### 사용 예 (간단 스모크 테스트)
```python
# 1. 패리티 Moore → Mealy → 동치(shift=+1)
Σ={"0","1"}; Γ={"0","1"}
Q={"E","O"}; ρ={"E":"0","O":"1"}
δ={("E","0"):"E",("E","1"):"O",("O","0"):"O",("O","1"):"E"}
Mo = DMoore(Q, Σ, Γ, δ, ρ, "E")
Me = moore_to_mealy(Mo)
ok, cex = equal_moore_vs_mealy(Mo, Me, shift=+1)
assert ok

# 2. '01' Mealy → Moore → 동치(shift=+1)
Q={"S","Z"}; Σ={"0","1"}; Γ={"0","1"}
δm={("S","0"):"Z",("S","1"):"S",("Z","0"):"Z",("Z","1"):"S"}
λm={("S","0"):"0",("S","1"):"0",("Z","0"):"0",("Z","1"):"1"}
Me = DMealy(Q, Σ, Γ, δm, λm, "S")
Mo2 = mealy_to_moore(Me)                # 초기 출력 <init>
ok, cex = equal_moore_vs_mealy(Mo2, Me, shift=+1)
assert ok
```

---

## 9. 최소화 관점 — 언제 누가 더 작아지나?

- **Moore→Mealy**: 보통 **상태 수가 줄거나 동일**.
  - 이유: Moore에서 “출력 다양성” 때문에 쪼개야 했던 상태를 Mealy는 **전이 라벨 차이만으로** 처리 가능.
- **Mealy→Moore**: **늘 수 있음(최대 $|\Gamma|$ 배)**.
  - 이유: 전이별 출력 차이를 상태에 흡수해야 하므로 **상태 세분화**가 필요.
- **유일성**: Trim(도달 상태 제한) 가정하에 각 모델의 **최소형**은 **동형까지 유일**.
- **교차 비교**: “최소 Mealy”와 “최소 Moore”의 상태 수는 일반적으로 **서로 다르되**, 변환 + 최소화 후 **지연 동치**가 성립.

---

## 10. 실전 체크리스트 — 변환·검증·설계

```text
[동치/정렬]
- Moore 초기 출력 포함/생략 관례를 먼저 고정
- Mealy↔Moore 비교는 지연 +1 정렬(초기 출력 제거)로
- 제품 기계 탐색으로 최단 반례 입력까지 산출

[변환 선택]
- 지연 0(즉시 반응) 필요 → Mealy 우선
- 출력 안정/글리치 억제/클럭 경계에서만 변화 → Moore 우선
- Mealy→Moore 시 상태 폭증 위험: |Q|·|Γ| 경계 확인

[최소화/스케일]
- Moore 최소화: "출력으로 초기 분할 → (다음 블록 튜플) 정제"
- Mealy 최소화: "(출력, 다음 블록) 서명으로 정제"
- 대알파벳/희소 전이: 딕셔너리 저장·클래스 묶음(예: 문자 클래스)로 서명 크기 축소

[테스트/검증]
- 총기계화(Sink) 후 동치 검사(정의역 차이 방지)
- 회귀 테스트에 '상태 구분 입력 시퀀스'(W-Method/HSI) 사용
```

---

## 11. 자주 묻는 질문(FAQ)

**Q1. Moore 초기 출력은 꼭 넣어야 하나?**
A. 문헌마다 다르다. **비교·검증** 단계에서 **둘 다 초기 출력 제거**로 정렬하면 혼란이 없다.

**Q2. Mealy→Moore 에서 상태 수가 정말 $|Q||\Gamma|$ 까지 필요할 수 있나?**
A. 가능하다(타이트). 상태 $q$ 마다 입력에 따라 **서로 모든 출력**이 발생하도록 만들면 각 $(q,\gamma)$ 가 구분되어야 한다.

**Q3. 일반화(전이/최종에 문자열 출력)에서도 동치가 유지되나?**
A. **순차 변환기(subsequential transducer)** 로 보면 yes. 상수 길이의 프리엠블/꼬리는 지연/접미로 흡수된다.

**Q4. 확률/가중을 얹으면 달라지나?**
A. 동치/변환의 코어는 동일하나, 검증 대상이 “출력 스트림”이 아니라 “가중 합”이라면 비교 조건이 달라진다(WFST 동등성).

---

## 12. 한 페이지 요약

- **표현력**: Moore·Mealy는 **동기(길이-보존) 변환**에 대해 **동등**.
- **변환**:
  - Moore→Mealy: $\lambda(p,a)=\rho(\delta'(p,a))$ — **상태 보존**, 지연 −1.
  - Mealy→Moore: 상태 $(q,\text{다음 출력})$ — **최대 $|Q||\Gamma|$** , 지연 +1.
- **정렬**: Moore 초기 출력 제거(Shift +1) 후 비교.
- **검증**: 제품 기계로 선형 시간 내 동치 판정, **최단 반례 입력** 산출.
- **최소화**: Mealy는 전이 라벨, Moore는 상태 라벨 기준. 변환 후 최소화해도 **지연 동치** 보장.

---

## 13. 부록 — 더 큰 예제(상태 폭증을 눈으로 확인)

### Mealy(출력 다양성 큰 경우)

```text
Σ = {a,b}, Γ = {X,Y,Z}
상태: q0,q1,q2 (시작 q0)
전이/출력 λ:
q0 --a/X--> q1   q0 --b/Y--> q2
q1 --a/Y--> q1   q1 --b/Z--> q2
q2 --a/Z--> q1   q2 --b/X--> q2
```

- 같은 목적 상태라도 입력에 따라 **모든 출력**(X,Y,Z)이 번갈아 나옴 → Mealy에서는 상태 3개로 충분.
- **Mealy→Moore**: 상태가 $(q,\gamma)$ 꼴로 **최대 $3\times 3=9$** 까지 늘 수 있다(Trim 후 6~9개 도달).

이후 `mealy_to_moore` 적용 → `minimize()` 로 실제 도달 상태수를 확인하라.
동치 검증은 `equal_moore_vs_mealy(shift=+1)` 로 즉시 가능.

---
```python
# 빠른 실험 스니펫 (학습/블로그 독자용)
Me_Q = {"q0","q1","q2"}; Σ={"a","b"}; Γ={"X","Y","Z"}
δm = {("q0","a"):"q1",("q0","b"):"q2",
      ("q1","a"):"q1",("q1","b"):"q2",
      ("q2","a"):"q1",("q2","b"):"q2"}
λm = {("q0","a"):"X",("q0","b"):"Y",
      ("q1","a"):"Y",("q1","b"):"Z",
      ("q2","a"):"Z",("q2","b"):"X"}
Me  = DMealy(Me_Q, Σ, Γ, δm, λm, "q0")
Mo  = mealy_to_moore(Me)
MoT = Mo.minimize()
ok, cex = equal_moore_vs_mealy(MoT, Me, shift=+1)
print("equal?", ok, "counterexample:", cex)  # 기대: True, None
```

---

이 예제는 **상태 폭증 경계가 현실적**임을 보여 준다. 출력 알파벳이 커지거나, 전이별 출력 다양성이 크면 Moore로의 직접 변환 대신 **Mealy 유지** 또는 **하이브리드 설계**를 고려하라.
