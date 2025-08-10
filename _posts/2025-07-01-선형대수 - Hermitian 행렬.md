---
layout: post
title: 선형대수 - Hermitian 행렬
date: 2025-07-01 20:20:23 +0900
category: 선형대수
---
# 🧮 Hermitian 행렬 (에르미트 행렬)이란?

**Hermitian 행렬(에르미트 행렬)**은 실수 공간에서의 **대칭 행렬**에 해당하는 복소수 공간의 행렬입니다.  
양자역학, 신호처리, 머신러닝 등 수많은 분야에서 핵심적 역할을 하며,  
**모든 고유값이 실수**이고 **유니터리 대각화**가 가능한 특별한 행렬입니다.

---

## 🧩 1. 정의

### 📌 Hermitian 행렬의 정의

정사각 복소수 행렬 \( A \in \mathbb{C}^{n \times n} \)가 다음 조건을 만족할 때 **Hermitian 행렬**이라 합니다:

\[
A = A^*
\]

여기서 \( A^* \)는 **켤레 전치(conjugate transpose)**, 즉  
전치(transpose)한 후 복소수 켤레(conjugate)를 취한 것입니다.

---

### 🔍 예시

\[
A =
\begin{bmatrix}
2 & i \\
-i & 3
\end{bmatrix}
,\quad
A^* =
\begin{bmatrix}
2 & -i \\
i & 3
\end{bmatrix}
\Rightarrow A = A^*
\]

→ Hermitian 행렬

---

## 🧠 2. 성질

| 성질 | 설명 |
|------|------|
| **실수 대각 성분** | \( A_{ii} \in \mathbb{R} \) |
| **켤레 대칭** | \( A_{ij} = \overline{A_{ji}} \) |
| **정규 행렬** | 항상 \( A A^* = A^* A \) 만족 |
| **고유값** | 모두 **실수(real)** |
| **고유벡터** | 서로 **직교 가능** |
| **유니터리 대각화 가능** | \( A = U \Lambda U^* \) with unitary \( U \) |

---

## 🔬 3. 기하학적 의미

Hermitian 행렬은 복소수 벡터 공간에서 **내적을 보존하는 자기수반 연산자** 역할을 하며,  
벡터 공간을 ‘왜곡 없이’ 변형하는 구조로 이해할 수 있습니다.

- 실수 대칭 행렬은 실수 공간에서의 특별한 Hermitian
- **양자역학**에서는 **관측 가능한 물리량**은 항상 Hermitian 연산자로 표현됩니다

---

## 🔢 4. 고유값과 고유벡터

### ✅ 성질 요약

- 모든 고유값 \( \lambda \in \mathbb{R} \)
- 고유벡터들은 서로 직교 가능
- **유니터리 행렬** \( U \)로 대각화 가능:

\[
A = U \Lambda U^*, \quad \Lambda = \text{diag}(\lambda_1, \dots, \lambda_n)
\]

### 📘 예제

\[
A = 
\begin{bmatrix}
2 & i \\
-i & 2
\end{bmatrix}
\Rightarrow \lambda_1 = 3, \quad \lambda_2 = 1
\]

---

## 🧮 Python 예제: Hermitian 행렬 고유값

```python
import numpy as np

# Hermitian 행렬
A = np.array([
    [2, 1j],
    [-1j, 2]
])

# 확인
print("A Hermitian?", np.allclose(A, A.conj().T))

# 고유값, 고유벡터
eigvals, eigvecs = np.linalg.eigh(A)  # Hermitian에 최적화된 함수

print("고유값:", eigvals)
print("고유벡터:\n", eigvecs)
```

---

## 📚 5. 응용 분야

| 분야 | 활용 예 |
|------|---------|
| **양자역학** | 관측량 → Hermitian 연산자 (예: 해밀토니안) |
| **스펙트럴 그래프 이론** | 그래프 라플라시안은 Hermitian |
| **PCA/통계** | 공분산 행렬은 항상 Hermitian |
| **신호 처리** | 복소수 신호의 자기수반 필터링 연산 |
| **딥러닝** | 양자 머신러닝, GNN, EigenEmbedding 등에서 응용됨 |

---

## 🧭 6. 대칭 행렬 vs Hermitian 행렬

| 구분 | 대칭 행렬 | Hermitian 행렬 |
|------|------------|----------------|
| 정의 | \( A^T = A \) | \( A^* = A \) |
| 공간 | 실수 | 복소수 |
| 고유값 | 실수 | 실수 |
| 고유벡터 | 직교 가능 | 직교 가능 |
| 대각화 | 직교 대각화 | 유니터리 대각화 |

---

## ✅ 요약

| 핵심 개념 | 설명 |
|------------|------|
| Hermitian 행렬 | \( A = A^* \), 대칭 행렬의 복소수 확장 |
| 고유값 | 항상 실수 |
| 대각화 | 유니터리 행렬로 대각화 가능 |
| 응용 | 물리학, 그래프이론, 머신러닝 등 |