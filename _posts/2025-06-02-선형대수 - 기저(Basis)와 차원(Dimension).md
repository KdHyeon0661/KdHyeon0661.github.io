---
layout: post
title: 선형대수 - 기저(Basis)와 차원(Dimension)
date: 2025-06-02 21:20:23 +0900
category: 선형대수
---
# 기저(Basis)와 차원(Dimension)

> **핵심 요약**
>
> - **기저(Basis)**: 벡터 공간 $$V$$를 **전부 생성**(span)하면서 **서로 선형 독립**인 벡터들의 최소 집합
> - **차원(Dimension)**: $$V$$의 **어느 기저든 그 벡터의 개수는 같다** → 그 공통 개수를 $$\dim(V)$$라 정의

---

## 벡터 공간 짧은 복습

벡터 공간 $$V$$은 덧셈과 스칼라곱에 대해 닫혀 있고 0벡터를 가지는 집합입니다. 대표 예:
- $$\mathbb{R}^n$$, 모든 $$m\times n$$ 행렬의 집합, 다항식 공간, 연속함수 공간 등.

> 본 글의 모든 수식은 **MathJax의 $$…$$ 표기**만 사용합니다.

---

## 기저(Basis)의 정확한 정의와 직관

### 정의

집합 $$B=\{\vec{v}_1,\dots,\vec{v}_k\}\subset V$$가 기저가 되려면

$$
\boxed{
\text{(i) 선형 독립} \quad \land \quad \text{(ii) } \text{span}(B)=V
}
$$

- **선형 독립**: $$c_1\vec{v}_1+\cdots+c_k\vec{v}_k=\vec{0} \Rightarrow c_1=\cdots=c_k=0$$
- **생성(Span)**: 모든 $$\vec{x}\in V$$가 어떤 계수로 $$\vec{x}=\sum_i c_i\vec{v}_i$$ 로 표현 가능

### 직관

- **중복 없이** 공간 전체를 덮는 최소 벡터 모음
- 기저를 알면 **좌표 표현**이 가능: 임의의 $$\vec{x}$$는 계수 벡터 $$[\vec{x}]_B=(c_1,\dots,c_k)^\top$$ 로 유일하게 기록

---

## 차원(Dimension)의 정의와 기본 정리

### 정의

$$
\boxed{\dim(V)=\text{어느 기저에서나의 벡터 개수}}
$$

- 서로 다른 기저라도 **벡터 개수는 항상 동일** (Replacement/Steinitz 정리의 귀결)

### 기본 성질

- **부분공간의 차원**은 원공간보다 크지 않음: $$U\le V \Rightarrow \dim(U)\le \dim(V)$$
- **직합의 차원**: $$U\oplus W \Rightarrow \dim(U\oplus W)=\dim(U)+\dim(W)$$
- **합과 교집합 공식**:
  $$
  \boxed{\dim(U+W)=\dim(U)+\dim(W)-\dim(U\cap W)}
  $$

---

## 선형 독립과 생성 — 실전 판정법

### 유한 차원에서의 표준 절차(행렬화 + 가우스 소거)

벡터들을 열로 쌓아 행렬 $$A=[\vec{v}_1\ \cdots\ \vec{v}_k]$$를 만든 뒤,
- **RREF의 피벗 열**이 기저를 준다(선형 독립한 대표 열)
- **피벗 개수 = rank = 기저 크기**
- **열공간/행공간/영공간**과의 연결은 랭크-널리티로 정리됨

### 내적공간에서의 직교 기저(Gram–Schmidt)

내적이 정의된 공간(예: $$\mathbb{R}^n$$, 함수공간)에서는 **그람–슈미트**로 직교(정규직교) 기저를 만들면 좌표 계산과 투영이 쉬워짐.

---

## 예제로 보는 기저와 차원

### $$\mathbb{R}^2$$의 표준기저

$$
\vec{e}_1=\begin{bmatrix}1\\0\end{bmatrix},\quad
\vec{e}_2=\begin{bmatrix}0\\1\end{bmatrix}
\quad\Rightarrow\quad
\text{span}\{\vec{e}_1,\vec{e}_2\}=\mathbb{R}^2,\ \dim(\mathbb{R}^2)=2
$$

### 선형 종속 예

$$
\vec{v}_1=\begin{bmatrix}1\\2\end{bmatrix},\quad
\vec{v}_2=\begin{bmatrix}2\\4\end{bmatrix}=2\vec{v}_1
\quad\Rightarrow\quad \{\vec{v}_1,\vec{v}_2\}\ \text{는 종속} \Rightarrow \text{기저 아님}
$$

### 다항식 공간

$$
\mathcal{P}_2=\{a+bx+cx^2\mid a,b,c\in\mathbb{R}\},\quad
\text{표준기저 } \{1,x,x^2\},\ \dim(\mathcal{P}_2)=3
$$

- 부분공간 예: **홀수차 부분공간** $$\{bx\mid b\in\mathbb{R}\}$$ 의 기저는 $$\{x\}$$, 차원은 1
- 조건식으로 정의된 부분공간
  예: $$\{a+bx+cx^2\in\mathcal{P}_2\mid a+b+c=0\}$$ 은 제약 1개 → 차원 $$=3-1=2$$

### 행렬 공간

$$
\mathbb{R}^{2\times 2}=\left\{\begin{bmatrix}a&b\\c&d\end{bmatrix}\right\},\quad
\text{표준기저}\ \left\{
\begin{bmatrix}1&0\\0&0\end{bmatrix},
\begin{bmatrix}0&1\\0&0\end{bmatrix},
\begin{bmatrix}0&0\\1&0\end{bmatrix},
\begin{bmatrix}0&0\\0&1\end{bmatrix}
\right\}
$$
따라서 $$\dim(\mathbb{R}^{2\times 2})=4$$.

### 함수 공간의 기저

구간 $$[-\pi,\pi]$$에서의 삼각기저:
$$
\{\ 1,\ \cos x,\ \sin x,\ \cos 2x,\ \sin 2x,\ \dots\ \}
$$
적절한 완비성을 가정하면(푸리에 이론), **직교성**을 바탕으로 부분공간의 기저를 구성 가능.

---

## 좌표와 기저 변화(Change of Basis)

### 좌표 벡터

기저 $$B=\{\vec{b}_1,\dots,\vec{b}_n\}$$에 대해 $$\vec{x}=\sum_i c_i\vec{b}_i$$라면
$$[\vec{x}]_B=(c_1,\dots,c_n)^\top\ \text{(유일)}$$.

### 기저 변화 행렬

두 기저 $$B=\{\vec{b}_i\},\ C=\{\vec{c}_i\}$$, 기저행렬을
$$
B=[\vec{b}_1\ \cdots\ \vec{b}_n],\quad
C=[\vec{c}_1\ \cdots\ \vec{c}_n]
$$
라 두면
$$
\boxed{[\vec{x}]_C=P_{C\leftarrow B}\,[\vec{x}]_B,\quad
P_{C\leftarrow B}=C^{-1}B}
$$
(각 기저행렬은 가역이어야 함: 기저는 선형 독립)

---

## 차원 공식들 — 실전에서 자주 쓰는 식

### 합-교집합 공식

$$
\boxed{\dim(U+W)=\dim(U)+\dim(W)-\dim(U\cap W)}
$$
- **증명 스케치**: 교집합 기저를 공통 씨앗으로 확장하여 두 공간의 기저를 만든 뒤, 합공간의 생성집합에서 중복(교집합) 차원을 한 번 빼준다.

### 랭크-널리티(행렬과의 연결)

$$
\boxed{\operatorname{rank}(A)+\dim\text{Null}(A)=n},\qquad
\boxed{\operatorname{rank}(A)+\dim\text{Null}(A^\top)=m}
$$
- 여기서 $$\operatorname{rank}(A)=\dim\text{Col}(A)=\dim\text{Row}(A)$$

---

## 기저를 계산하는 알고리즘(유한차원)

### 가우스 소거/행기약형(RREF)

- 열 벡터 집합의 **선형 독립 판정** 및 **기저 추출**에 표준적
- **피벗 열**이 원래 행렬의 **열공간 기저**를 준다
- **비영 행**이 **행공간 기저**를 준다

### 그람–슈미트(내적공간)

- 직교(정규직교) 기저 생성
- 수치해석에서는 **Householder/Modified GS**가 안정적

---

## 손계산 예제

### 열공간·영공간 기저 구하기

$$
A=\begin{bmatrix}
1&2&3\\
0&1&4\\
1&3&7
\end{bmatrix}
$$

1) **열공간**: 가우스 소거로 첫 두 열이 피벗이라고 하자(실제로 3열은 1·col1+2·col2).
따라서
$$
\text{Col}(A)=\text{span}\left\{
\begin{bmatrix}1\\0\\1\end{bmatrix},
\begin{bmatrix}2\\1\\3\end{bmatrix}
\right\},\quad \dim=2
$$

2) **영공간**: $$A\vec{x}=\vec{0}$$을 풀면 자유변수 1개 →
$$
\text{Null}(A)=\text{span}\left\{\begin{bmatrix}1\\-2\\1\end{bmatrix}\right\},\quad \dim=1
$$

3) 차원 체크: $$n=3,\ \operatorname{rank}=2,\ \text{nullity}=1\Rightarrow \text{OK}$$

### 다항식 부분공간의 차원

제약 $$a+b+c=0$$을 만족하는 $$\mathcal{P}_2$$의 부분공간:
$$
\mathcal{W}=\{a+bx+cx^2\in\mathcal{P}_2\mid a+b+c=0\}
$$
즉 $$a=-(b+c)$$이므로
$$
p(x)=-(b+c)+bx+cx^2=b(x-1)+c(x^2-1)
$$
따라서 기저는 $$\{x-1,\ x^2-1\}$$, 차원은 2.

---

## — 기저·차원·좌표·기저변화

> 숫자 오차 없이 **정확** 계산하려면 SymPy 권장.

```python
from sympy import Matrix, symbols, simplify

# 열공간/행공간/영공간/좌영공간, 차원 확인

A = Matrix([
    [1, 2, 3],
    [0, 1, 4],
    [1, 3, 7],
])

colB = A.columnspace()   # 열공간 기저
rowB = A.rowspace()      # 행공간 기저
nullB = A.nullspace()    # 영공간 기저
leftNullB = A.T.nullspace()  # 좌영공간 기저

print("rank(A) =", A.rank())
print("dim Col(A) =", len(colB))
print("dim Row(A) =", len(rowB))
print("dim Null(A) =", len(nullB))
print("dim Null(A^T) =", len(leftNullB))

# 좌표 표현: 주어진 기저 B에서의 [x]_B 구하기

b1 = Matrix([1, 0, 1])
b2 = Matrix([2, 1, 3])
B = Matrix.hstack(b1, b2)        # 3x2 (부분공간 기저)
x = Matrix([5, 2, 7])            # Col(A)에 실제로 속하는지 확인
# x가 span{b1,b2}에 있으면 B*c = x가 풀림 (최소제곱이 아니라 정확해가 존재)

c = B.gauss_jordan_solve(x)[0]   # [x]_B
print("[x]_B =", c)

# 기저 변화: B -> C

c1 = Matrix([1, 1, 0])
c2 = Matrix([0, 1, 1])
C = Matrix.hstack(c1, c2)
# 두 기저가 같은 2차원 부분공간을 생성한다고 가정할 때 변환행렬

PC_from_B = C.inv()*B
print("P_{C<-B} =\n", PC_from_B)
print("[x]_C =", simplify(PC_from_B * c))
```

**해설**
- `columnspace()/rowspace()/nullspace()`는 각각 기저 벡터를 반환 → **차원 = 길이**
- 좌표는 **기저행렬**과 선형시스템 풀이로 구함: $$B[\vec{x}]_B=\vec{x}$$
- 기저변화 행렬은 $$P_{C\leftarrow B}=C^{-1}B$$

---

## 자주 하는 실수와 팁

- **RREF의 열을 그대로 기저로 쓰지 않기**: 열공간의 기저는 **원본 행렬의 피벗 열**을 선택
- **기저 ≠ 유일**: 기저는 많지만 **차원은 항상 동일**
- **0벡터는 기저에 포함될 수 없음**(독립 위반)
- **함수공간**에서는 수치 표본으로 독립을 판단하면 거짓 양성/음성이 날 수 있음 → 심볼릭/이론적 판정(예: Wronskian, 정리) 병행
- **내적공간**에서는 직교(정규직교) 기저가 계산·투영·좌표화에 유리

---

## 연습 문제(스스로 손·코드로 확인)

1) $$A=\begin{bmatrix}1&2&3&0\\2&4&6&1\\1&1&1&0\end{bmatrix}$$ 에 대해
   - RREF로 랭크를 구하고 **Col/Row/Null/LeftNull**의 차원을 확인
   - 각 공간의 **기저**를 구하라.
2) $$\mathcal{P}_3$$에서 제약 $$p(1)=p(-1)=0$$을 만족하는 부분공간의 기저와 차원은?
3) $$\mathbb{R}^3$$의 두 부분공간 $$U=\text{span}\{(1,0,1),(0,1,1)\}$$, $$W=\text{span}\{(1,1,0),(1,-1,2)\}$$에 대해
   - $$\dim(U),\dim(W),\dim(U\cap W),\dim(U+W)$$를 계산하고 공식이 성립함을 보이라.

---

## 총정리 표

| 항목 | 핵심 식/정의 | 실전 관점 |
|---|---|---|
| **기저** | $$\text{span}(B)=V,\ B\text{ 독립}$$ | 좌표 표현의 기준, 중복 없는 최소 생성집합 |
| **차원** | $$\dim(V)=\text{기저의 벡터 수}$$ | 기저는 여러 개, 차원은 하나 |
| **합-교집합** | $$\dim(U+W)=\dim(U)+\dim(W)-\dim(U\cap W)$$ | 중복(교집합)을 한 번 뺀다 |
| **랭크-널리티** | $$\operatorname{rank}(A)+\dim\text{Null}(A)=n$$ | 행렬의 구조·해의 자유도 |
| **기저변화** | $$[\vec{x}]_C=(C^{-1}B)[\vec{x}]_B$$ | 표현만 바뀌고 벡터 자체는 불변 |

---

### 마무리

기저와 차원은 **벡터 공간을 정량화**하는 핵심 언어입니다.
가우스 소거로 **기저/차원**을 계산하고, 필요 시 그람–슈미트로 **정규직교 기저**를 만들며,
기저변화 행렬로 **좌표계를 바꾸는 법**까지 익히면 대부분의 선형대수 문제는 구조가 즉시 보입니다.
