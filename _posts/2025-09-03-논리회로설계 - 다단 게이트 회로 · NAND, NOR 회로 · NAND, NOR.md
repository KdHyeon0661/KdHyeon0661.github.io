---
layout: post
title: 논리회로설계 - 다단 게이트 회로 · NAND, NOR 회로 · NAND, NOR
date: 2025-09-03 15:25:23 +0900
category: 논리회로설계
---
# 회로 설계 — 완전 정리

> 표기: \(+\)=OR, \(\cdot\) 또는 생략=AND, \(\overline{X}\)=NOT \(X\), \(\oplus\)=XOR.
> 목표: (1) **다단 게이트 회로**의 개념·지연·부하 특성, (2) **NAND/NOR**의 트랜지스터/논리 관점과 보편성, (3) **SOP→NAND–NAND**, **POS→NOR–NOR** 매핑 규칙과 **버블 푸시(bubble pushing)**까지 포함한 **2단 회로** 설계를 단계별로 설명한다.
> 메모: CMOS 표준셀 흐름을 기준으로 최신 관행(6/7/8 입력 대형 게이트 제한, AOI/OAI 활용, 동기 경계로 글리치 흡수)을 반영한다.

---

## 개요(왜 2단·다단을 구분하는가)

2-level(2단) 회로는 **깊이 2**(곱항→최종 OR 또는 합항→최종 AND)로, **논리 깊이 최소**이지만 한 게이트의 **팬인**이 커지기 쉽다. 반대로 다단(>2) 회로는 **인수화·공유**로 팬인/배선을 줄여 **지연·전력**을 개선할 여지가 있다.

- **SOP(곱의 합)** ↔ **NAND–NAND**로 직결
- **POS(합의 곱)** ↔ **NOR–NOR**로 직결
- **버블 푸시**로 인버터를 최소화하면, NAND/NOR **두 층**으로 깨끗하게 구현 가능

---

## 다단 게이트 회로(Multilevel Logic)

### 정의와 비용 지표

- **2단(2-level)**: 입력 → (1단)곱항/합항 → (2단)최종 OR/AND. (예: \(F=\sum\) 형태)
- **다단(>2-level)**: 공통인수 인수화, 중간 신호 공유, AOI/OAI 등 복합 게이트 활용.

주요 지표:
- **리터럴 수**(PLA/ROM/면적 근사), **게이트 수**, **팬인/팬아웃**, **배선 길이**, **지연 \(t_{pd}\)**, **전력**.

### 지연 모델(직관)

조합 경로 지연 근사:
\[
t_{pd} \approx \sum_{g\in\text{경로}} t_{pd}(g) + t_{\text{배선}}
\]
- **팬인↑ → 지연↑**: CMOS에서 NOR는 **pMOS 직렬**로 상승지연이 커지므로 큰 팬인은 불리.
- 길고 얇은 배선은 RC가 커져 **배선 지연** 우세.

### 관점(요지)

- **기준 인버터**의 논리적 노력 \(g=1\), 기생 \(p\approx1\).
- 대략적 경향(공정·셀에 따라 다소 상이):
  - **NAND2**: \(g \approx 4/3,\ p \approx 2\)
  - **NOR2**: \(g \approx 5/3,\ p \approx 2\)
  - 입력 수가 늘수록 \(g\)·\(p\) 증가 → **큰 팬인 NOR**는 특히 불리.
→ 설계 팁: 2단이라도 **큰 OR/AND를 다단 균형 트리**로 쪼개면 지연이 줄 수 있다.

---

## NAND/NOR 회로의 성질

### 보편 게이트(Functional Completeness)

**NAND만으로**
\[
\overline{A}=A\,\text{NAND}\,A,\quad
AB=\overline{(A\,\text{NAND}\,B)},\quad
A+B=\overline{\overline{A}\cdot \overline{B}}
\]

**NOR만으로**
\[
\overline{A}=A\,\text{NOR}\,A,\quad
A+B=\overline{A\,\text{NOR}\,B},\quad
AB=\overline{\overline{A}+\overline{B}}
\]

### CMOS 트랜지스터 직관

- **NAND(n입력)**: **nMOS 직렬**(풀다운)·**pMOS 병렬**(풀업) → 풀업 강함 → **NOR보다 유리**.
- **NOR(n입력)**: **pMOS 직렬**(풀업)로 **상승 지연↑** → 큰 팬인에 취약.
- 실무: 큰 팬인은 **NAND 선호**, NOR는 **2~3입력** 권장.

### 버블 푸시(bubble pushing)

- 드모르간으로 **인버전(버블)** 을 게이트 **전후로 이동**해 같은 기능을 유지.
- 마주보는 버블은 **상쇄**. 입력의 보수(\(\overline{A}\))는 **입력 버블**로 표현해 인버터를 줄인다.

---

## 2단 회로 설계 절차(핵심 레시피)

### SOP → **NAND–NAND**

규칙:
1) \(F = P_1 + P_2 + \cdots + P_k\) (각 \(P_i\): 곱항)
2) 드모르간:
\[
F=\overline{\,\overline{P_1}\cdot\overline{P_2}\cdots\overline{P_k}\,}
\]
3) **1단**: 각 \(P_i\)를 **NAND**로 만들어 \(\overline{P_i}\) 생성(보수 리터럴은 입력 버블로 처리).
4) **2단**: 위 출력을 **NAND**로 결합 → 최종 \(F\).

**예제 A** \(\;F=AB+\overline{A}C+BC\)
\[
F=\overline{(AB)'\cdot(\overline{A}C)'\cdot(BC)'}
\]
- 1단: \(N_1=\text{NAND}(A,B)\), \(N_2=\text{NAND}(\overline{A},C)\), \(N_3=\text{NAND}(B,C)\)
- 2단: \(F=\text{NAND}(N_1,N_2,N_3)\)

### POS → **NOR–NOR**

규칙:
1) \(F=S_1 S_2 \cdots S_k\) (각 \(S_i\): 합항)
2) 드모르간:
\[
F=\overline{\,\overline{S_1}+\overline{S_2}+\cdots+\overline{S_k}\,}
\]
3) **1단**: 각 \(S_i\)를 **NOR**로 \(\overline{S_i}\) 생성.
4) **2단**: 위 출력을 **NOR**로 결합.

**예제 B** \(\;F=(A+B)(A+\overline{C})(B+D)\)
\[
F=\overline{(A+B)' + (A+\overline{C})' + (B+D)'}
\]
- 1단: \(M_1=\text{NOR}(A,B)\), \(M_2=\text{NOR}(A,\overline{C})\), \(M_3=\text{NOR}(B,D)\)
- 2단: \(F=\text{NOR}(M_1,M_2,M_3)\)

### 혼성 2단(NAND–NOR / NOR–NAND)

- 목표 팬인·인버전 수를 맞추기 위해 혼성 구조가 유리할 수 있다.
- 예) \(\overline{P_1+P_2+\cdots}\) 형태는 **NAND–NOR** 직결로 추가 인버터 없이 구현.

### 인버터/버블 최소화 4원칙

1) **SOP→NAND–NAND**, **POS→NOR–NOR**를 기본.
2) **보수 리터럴은 입력 버블**로 표현해 중간 인버터 제거.
3) **마주보는 버블**은 상쇄.
4) 팬인 제한이 있으면 곱항/합항을 **분할**하거나 **다단 인수화**.

---

## 팬인과 트리 구조(속도·전력 최적화)

### 큰 OR/AND를 트리로

- 큰 OR(예: 8입력)은 **NAND–NAND**에서 **1단 NAND**의 팬인이 커짐 →
  **균형 트리**(예: 2→4→8)로 분해하면 각 게이트 부하·기생이 감소.

### 활용

- 표준셀에 **AOI21/22, OAI21/22**가 있다면, SOP/POS 일부를 **직접 1단**으로 매핑해 지연을 줄인다.
  - 예: \(Y=\overline{AB+C}\)는 **AOI21** 1개.
  - 버블 푸시로 같은 기능을 AOI/OAI에 끼워 맞추면 **레벨 단축**.

---

## — 분석과 완화

### 정적-1/0 해저드

- 이론상 출력이 1(또는 0)로 유지되어야 하는 **입력 전이**에서 순간 0(또는 1)이 튀는 현상.
- K-map에서 **인접 1(또는 0)** 들이 **중첩 없이 다른 묶음**으로만 덮일 때 발생 가능.

### 완화

- **합의항(consensus)** 추가: \(AB+\overline{A}C\Rightarrow +BC\)
- **게이트 공유/중첩**으로 재합성.
- **동기 경계(레지스터)** 로 후단에서 글리치 흡수.

---

## 완전 예제 — 설계·검증·코드

### 예제 C(최소 SOP → NAND–NAND)

K-map 최소화 결과:
\[
F=\overline{A}\,\overline{B}C + \overline{A}BC + A\overline{B}C
\]

#### 게이트 수준 설계

- 1단 NAND:
  - \(N_1=\text{NAND}(\overline{A},\overline{B},C)\)
  - \(N_2=\text{NAND}(\overline{A},B,C)\)
  - \(N_3=\text{NAND}(A,\overline{B},C)\)
- 2단 NAND: \(F=\text{NAND}(N_1,N_2,N_3)\)

#### Verilog(구조적 스케치)

```verilog
module nand2level(
  input  logic A,B,C,
  output logic F
);
  logic nA = ~A, nB = ~B;
  logic n1, n2, n3;
  assign n1 = ~(nA & nB & C);
  assign n2 = ~(nA &  B & C);
  assign n3 = ~( A & nB & C);
  assign F  = ~(n1 & n2 & n3);
endmodule
```

#### Python 검증(진리표 등가성)

```python
from itertools import product
def F_ref(A,B,C):
    return ((not A and not B and C) or
            (not A and B and C) or
            (A and not B and C))
def F_nand(A,B,C):
    n1 = not ((not A) and (not B) and C)
    n2 = not ((not A) and B and C)
    n3 = not (A and (not B) and C)
    return not (n1 and n2 and n3)
print(all(F_ref(*v)==F_nand(*v) for v in product([0,1],[0,1],[0,1])))
```

### 예제 D(최소 POS → NOR–NOR)

\[
F=(A+B)(\overline{A}+C)(\overline{B}+D)
\]

```verilog
module nor2level(
  input  logic A,B,C,D,
  output logic F
);
  logic nA = ~A, nB = ~B;
  logic m1, m2, m3;
  assign m1 = ~(A | B);       // (A+B)'
  assign m2 = ~(nA | C);      // (A'+C)'
  assign m3 = ~(nB | D);      // (B'+D)'
  assign F  = ~(m1 | m2 | m3);
endmodule
```

### 예제 E(혼성 & AOI/OAI)

\[
Y=\overline{AB+C}\quad(\text{AOI21})
\]
- 단일 **AOI21**로 1레벨 구현 가능.
- NAND–NOR로도 가능하나 레벨↑/지연↑ 가능 → **표준셀 선택**이 성능 좌우.

---

## 큰 팬인 다루기 — 분해와 균형화

### 8입력 OR의 SOP→NAND–NAND 분해

- 곱항 \(P_1,\dots,P_8\)이 많아 2단 최종 NAND의 팬인이 8이면 느릴 수 있음.
- **균형 트리**: (NAND of 4) → (NAND of 2) → (NAND final)로 재구성.
- 또는 **AOI22** 블록으로 4개씩 묶은 뒤 최종 NOR/NAND 결합.

### 배선·부하 균형

- 큰 부하(팬아웃↑)에는 **버퍼/업사이징**.
- 긴 배선은 **게이트 삽입(리피터)** 로 RC 저감.

---

## 표준셀 실전 매핑

### 버블 푸시로 AOI/OAI에 끼워 맞추기

- \(F=\overline{(AB)+(CD)}\) → **AOI22** 1개
- \(F=\overline{AB\cdot(C+D)}\) → **OAI21** 등으로 단축

### 라이브러리 제약

- 많은 라이브러리가 **NOR3/4 상한**, **NAND4 상한** 보유.
- 팬인 초과 시 **분해/인수화** 또는 **복합셀**로 치환.

---

## 해저드 안전 2단 설계(옵션)

### 정적-1 해저드 제거 예

\[
F=AB+\overline{A}C\quad\Rightarrow\quad F'=AB+\overline{A}C+BC
\]
- \(BC\)는 **합의항**.
- NAND–NAND에서 \(BC\) 곱항을 **추가**하면 인접 그룹 중첩 확보 → 글리치 억제.

---

## 수학 요약(드모르간/쌍대성)

- 드모르간:
\[
\overline{X+Y}=\overline{X}\cdot\overline{Y},\quad
\overline{XY}=\overline{X}+\overline{Y}
\]
- 쌍대성: AND↔OR, 0↔1, NAND↔NOR 교환 시 식의 구조적 대칭 유지.
→ **SOP↔POS**, **NAND–NAND↔NOR–NOR**는 서로 쌍대.

---

## 추가 예제(설계·검증 묶음)

### \(F=AC+BD\)

**(a) NAND–NAND**
\[
F=\overline{(AC)'\cdot(BD)'}
\]
- 1단: \(n1=\text{NAND}(A,C)\), \(n2=\text{NAND}(B,D)\)
- 2단: \(F=\text{NAND}(n1,n2)\)

**(b) NOR–NOR**
\[
F=(A+C)(B+D)\quad\Rightarrow\quad
F=\overline{(A+C)' + (B+D)'}
\]
- 1단: \(m1=\text{NOR}(A,C)\), \(m2=\text{NOR}(B,D)\)
- 2단: \(F=\text{NOR}(m1,m2)\)

**팬인/지연 논평**: 2입력 블록만 사용하므로 두 방식 모두 양호. 공정상 NAND 유리 경향.

### 테스트벤치 스케치

```verilog
module tb;
  logic A,B,C,D, F1,F2;
  nand2level u1(.A(A),.B(B),.C(C),.F(F1));
  nor2level  u2(.A(A),.B(B),.C(C),.D(D),.F(F2));
  initial begin
    foreach ({A,B,C,D}) begin end
    for (int v=0; v<16; v++) begin
      {A,B,C,D} = v[3:0];
      #1;
    end
  end
endmodule
```

---

## 체크리스트(실전용)

- [ ] **SOP→NAND–NAND**, **POS→NOR–NOR**로 시작하되, **AOI/OAI** 대체 가능성 탐색
- [ ] **입력 보수는 입력 버블**로, 중간 인버터 제거
- [ ] **팬인 상한**(셀/공정 제약) 준수, 필요 시 **분해/균형 트리**
- [ ] **합의항**으로 해저드 완화, 후단 **레지스터**로 글리치 흡수
- [ ] **배선 길이**·팬아웃·업사이징/버퍼 삽입으로 **RC** 관리
- [ ] STA로 **경로 지연** 확인, 타이밍 폐루프(hold/setup) 점검

---

## 연습문제

1) \(F=AB+\overline{A}C+BC\) 를 **NAND–NAND** 2단으로 그려라.
   - (a) 필요한 **인버터 수**(입력 버블 포함)
   - (b) 각 게이트 **팬인**
   - (c) 정적-1 해저드 여부와 **합의항** 제안

2) \(F=(A+B)(A+\overline{C})(B+D)\) 를 **NOR–NOR** 2단으로 그려라.
   - 버블 푸시로 **인버터 최소화**
   - 라이브러리에 **NOR3 금지**가 있으면, **다단 분해**로 재구성

3) \(F=AC+BD\) 를 **(a) NAND–NAND**, **(b) NOR–NOR** 로 설계하고, **균형 트리 분해** 시 지연이 왜 줄 수 있는지 **논리적 노력** 관점으로 설명하라.

4) \(F=AB+\overline{A}D+\overline{B}C\) 의 2단 NAND–NAND 구현에서 **정적-1 해저드**를 K-map으로 확인하고, 필요한 **합의항**을 추가해 회로를 갱신하라.

5) 표준셀 제약으로 **NAND4만 허용, NOR2만 허용**일 때, 예제 B의 회로를 **AOI/OAI**와 **분해**를 섞어 타이밍 유리하게 재설계하라.

---

## 요약(포켓 카드)

- **SOP→NAND–NAND**, **POS→NOR–NOR**, **버블 푸시**로 인버터 최소화
- **NAND가 NOR보다 일반적으로 빠름**(팬인 동일 시)
- 2단은 단순하나 **팬인 폭**이 커짐 → **분해/인수화/복합셀**로 보완
- **합의항**과 **동기 경계**로 해저드 안전 설계
- 성능은 **팬인·배선**·셀 선택(AOI/OAI)·파이프라인(동기)로 결정
