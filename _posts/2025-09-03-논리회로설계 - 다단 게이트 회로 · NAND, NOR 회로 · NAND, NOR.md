---
layout: post
title: 논리회로설계 - 다단 게이트 회로 · NAND, NOR 회로 · NAND, NOR
date: 2025-09-03 15:25:23 +0900
category: 논리회로설계
---
# 다단 게이트 회로 · NAND/NOR 회로 · NAND/NOR로 구현하는 2단(2-level) 회로 설계

> 표기: \(+\)=OR, \(\cdot\) 또는 생략=AND, \(\overline{X}\)=NOT \(X\), \(\oplus\)=XOR.  
> 목표: (1) **다단 게이트 회로**의 개념·지연·부하 특성을 정리하고, (2) **NAND/NOR**의 트랜지스터/논리 관점과 보편성, (3) **SOP→NAND–NAND**, **POS→NOR–NOR** 매핑 규칙과 **버블 푸시(bubble pushing)**까지 포함한 2단 회로 설계를 단계별로 설명합니다.

---

## 1) 다단 게이트 회로(Multilevel Logic)

### 1.1 정의와 비용 지표
- **2단(2-level)**: 입력 → (1단)곱항/합항 → (2단)최종 OR/AND. 예: **SOP**(곱의 합), **POS**(합의 곱).  
- **다단(>2-level)**: 공통인수 인수화, 보조 신호 생성 등으로 중간 단을 추가한 구조.
- **비용 지표**  
  - **리터럴 수**(PLA/ROM 구현에 유리), **게이트 수**, **팬인/팬아웃**, **지연 \(t_{pd}\)**, **전력**.
  - 2단은 논리적으로 짧지만(=깊이 2), **팬인 폭**이 커져 한 게이트에 신호가 많이 몰릴 수 있음.

### 1.2 지연/타이밍(조합 경로)
- 경로 지연 상한(간단 모델):  
  \[
  t_{pd}\approx \sum_{\text{경로의 게이트 }g} t_{pd}(g) \;+\; t_{\text{배선}}
  \]
- **팬인↑ → 지연↑**: 특히 CMOS에서 **NOR의 상부 pMOS 직렬**이 느리게 함(§2.2).  
- **오염 지연 \(t_{cd}\)**, **정적 해저드**(글리치) 고려: 인접 묶음 **중첩**(합의항)으로 완화.

### 1.3 다단 vs 2단 트레이드오프
- **2단**: 합성·검증 간편, PLA/ROM 매핑 용이.  
- **다단**: 인수화로 리터럴/팬인/배선 감소 → 지연·전력 개선 가능. (예: \(AB+AC = A(B+C)\))

---

## 2) NAND/NOR 회로의 성질

### 2.1 보편 게이트(Functional Completeness)
- **NAND만으로**  
  \[
  \overline{A}=A\ \text{NAND}\ A,\quad
  AB=\overline{(A\ \text{NAND}\ B)},\quad
  A+B=\overline{\overline{A}\cdot \overline{B}}
  \]
- **NOR만으로**  
  \[
  \overline{A}=A\ \text{NOR}\ A,\quad
  A+B=\overline{A\ \text{NOR}\ B},\quad
  AB=\overline{\overline{A}+\overline{B}}
  \]

### 2.2 CMOS 트랜지스터 관점(속도 직관)
- **NAND\(n\)-입력**: **nMOS 직렬(풀다운)**, **pMOS 병렬(풀업)** → 보통 **NOR보다 빠름**.  
- **NOR\(n\)-입력**: **pMOS 직렬(풀업)**, **nMOS 병렬(풀다운)** → 상승 지연↑, 팬인 큰 NOR은 **느리기 쉬움**.  
- 설계 팁: **높은 팬인**이 필요하면 **NAND 우선**, NOR는 가능하면 2~3입력으로 제한.

### 2.3 실전 매핑의 핵심 아이디어
- **드모르간 + 버블 푸시**: 인버전(버블)을 **입·출력으로 이동**해 **NAND–NAND**, **NOR–NOR** 2단 구조로 정규화.
- **인버터 수 최소화**: 같은 위치의 **짝 버블**은 상쇄.  
- 입력에서 **보수 리터럴**(예: \(\overline{A}\))이 필요하면 **입력 버블**로 처리.

---

## 3) 2단 회로 설계 절차

### 3.1 SOP → **NAND–NAND** (OR-of-AND를 NAND-of-NAND로)
**규칙**  
1) \(F = P_1 + P_2 + \cdots + P_k\) (각 \(P_i\)는 곱항).  
2) 드모르간:
   \[
   F=\overline{\ \overline{P_1}\cdot \overline{P_2}\cdots \overline{P_k}\ } 
   \]
3) 1단: 각 \(P_i\)를 **NAND**로 만들어 \(\overline{P_i}\) 생성(리터럴 보수는 입력 버블).  
4) 2단: 1단 출력들을 **NAND**로 결합 → 최종 \(F\).

**예제 A**  \(F=AB+\overline{A}C+BC\)
\[
F=\overline{(AB)'\cdot (\overline{A}C)'\cdot (BC)'}
\]
- 1단: \(N_1=AB\ \text{NAND}\), \(N_2=\overline{A},C\ \text{NAND}\), \(N_3=B,C\ \text{NAND}\).  
- 2단: \(F = N_1\ \text{NAND}\ N_2\ \text{NAND}\ N_3\).  
- **보수 리터럴** \(\overline{A}\)는 **입력에 인버터**(또는 \(A\ \text{NAND}\ A\))로 생성.

**버블 푸시 팁**: 입력에 이미 버블이 있으면 **앞단 게이트 유형을 바꾸지 않고** 버블을 **한 칸 앞/뒤로 밀어** 인버터를 상쇄.

---

### 3.2 POS → **NOR–NOR** (AND-of-OR를 NOR-of-NOR로)
**규칙**  
1) \(F = S_1 S_2 \cdots S_k\) (각 \(S_i\)는 합항).  
2) 드모르간:
   \[
   F=\overline{\ \overline{S_1} + \overline{S_2} + \cdots + \overline{S_k}\ }
   \]
3) 1단: 각 \(S_i\)를 **NOR**로 만들어 \(\overline{S_i}\) 생성(보수 리터럴은 입력 버블).  
4) 2단: 1단 출력들을 **NOR**로 결합 → 최종 \(F\).

**예제 B**  \(F=(A+B)(A+\overline{C})(B+D)\)
\[
F=\overline{(A+B)'\ +\ (A+\overline{C})'\ +\ (B+D)'}
\]
- 1단: \(N_1=A,B\ \text{NOR}\), \(N_2=A,\overline{C}\ \text{NOR}\), \(N_3=B,D\ \text{NOR}\).  
- 2단: \(F = N_1\ \text{NOR}\ N_2\ \text{NOR}\ N_3\).

---

### 3.3 혼성 2단(**NAND–NOR** 또는 **NOR–NAND**)
- 특정 함수에서 인버전 수를 더 줄이거나 팬인을 맞추려면 **혼합형**이 유리할 수 있음.  
- 예) \(F=\overline{P_1+P_2+\cdots}\) 형태면 **NOR–NOR** 대신 **NAND–NOR**로 직결.

---

### 3.4 인버터/버블 최소화 규칙(핵심 4문장)
1) **SOP→NAND–NAND**, **POS→NOR–NOR**가 기본.  
2) **보수 리터럴**은 **입력 버블**로 처리(게이트 안에서 \(\overline{X}\) 만들지 말 것).  
3) **연속된 동일 게이트**의 **마주보는 버블**은 상쇄.  
4) 팬인 제한(예: 4-입력 초과) 시 곱항/합항을 **분할**하거나 **다단 인수화**로 조정.

---

## 4) NAND/NOR 회로 설계 — 완전 예제

### 예제 C: **NAND–NAND**로 2단 구현
주어진 최소 SOP \(F=\sum m(1,2,5,7)\) → \(F=\overline{A}\,\overline{B}C + \overline{A}BC + A\overline{B}C\)

1) 곱항을 **NAND**로 출력 뒤집기:
- \(N_1=\text{NAND}(\overline{A},\overline{B},C)=(\overline{A}\,\overline{B}C)'\)  
- \(N_2=\text{NAND}(\overline{A},B,C)=(\overline{A}BC)'\)  
- \(N_3=\text{NAND}(A,\overline{B},C)=(A\overline{B}C)'\)

2) 최종:
\[
F = \text{NAND}(N_1,N_2,N_3)
\]
**입력 버블만**으로 \(\overline{A},\overline{B}\) 생성(인버터 2개). 중간 인버터 불필요.

---

### 예제 D: **NOR–NOR**로 2단 구현
최소 POS \(F=\prod M(0,2,8)=(A+B)(\overline{A}+C)(\overline{B}+D)\)

1) 합항을 **NOR**로 뒤집기:
- \(M_1=\text{NOR}(A,B)=(A+B)'\)  
- \(M_2=\text{NOR}(\overline{A},C)=(\overline{A}+C)'\)  
- \(M_3=\text{NOR}(\overline{B},D)=(\overline{B}+D)'\)

2) 최종:
\[
F=\text{NOR}(M_1,M_2,M_3)
\]

---

## 5) 검증·최적화·해저드

### 5.1 타이밍/팬인 점검
- 2단은 지연이 \(t_{pd}\approx t_{\text{1단}} + t_{\text{2단}}\).  
- 각 게이트의 **팬인 한계**를 넘지 않도록 곱항/합항을 **분할**하거나 다단화.

### 5.2 해저드(정적-1/0) 완화
- K-map에서 인접 1(또는 0)들이 **중첩 없이** 다른 묶음으로만 덮이면 글리치 가능.  
- **합의항 추가**(예: \(AB+\overline{A}C \Rightarrow +BC\)) 또는 **게이트 공유**로 중첩 확보.

### 5.3 NAND vs NOR 선택
- **속도**: 같은 팬인이라면 **NAND가 유리**(§2.2).  
- **구현 제약**: 공정/셀 라이브러리, 팬아웃, 와이어 부하에 따라 실제 최적 선택.

---

## 6) 요약 포켓 카드

- **SOP→NAND–NAND**, **POS→NOR–NOR** (드모르간 + 버블 푸시).  
- **보수 리터럴은 입력 버블**로, **인버터 최소화**.  
- **NAND가 일반적으로 NOR보다 빠름**(팬인 동일 시).  
- 2단은 간단하지만 팬인 커질 수 있음 → 필요 시 **다단 인수화**.  
- 해저드는 **합의항 중첩**으로 완화.

---

## 7) 연습문제

1) \(F=AB+\overline{A}C+BC\) 를 **NAND–NAND** 2단으로 그려라. (필요한 인버터 수, 각 게이트 팬인 명시)  
2) \(F=(A+B)(A+\overline{C})(B+D)\) 를 **NOR–NOR** 2단으로 그려라. (버블 푸시로 인버터 최소화)  
3) K-map 최소화 결과 \(F=AC+BD\) 를 **(a) NAND–NAND**, **(b) NOR–NOR** 두 방식으로 구현하고, 어느 쪽이 팬인/지연 면에서 유리한지 논하라.  
4) \(F=AB+\overline{A}D+\overline{B}C\) 의 2단 **NAND–NAND** 구현에서 정적-1 해저드가 우려된다. K-map으로 **합의항**을 찾아 추가하고 회로 변화를 설명하라.  
5) 라이브러리가 **3-입력 NOR 금지** 제약을 가진다. 예제 B의 회로를 **다단 인수화**로 재설계하라(게이트 팬인≤2).
