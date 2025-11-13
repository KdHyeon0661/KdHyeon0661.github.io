---
layout: post
title: 선형대수 - Hermitian 행렬
date: 2025-07-01 20:20:23 +0900
category: 선형대수
---
# Hermitian 행렬(에르미트 행렬)

> **핵심 한 줄**
> Hermitian(에르미트) 행렬은 $$A=A^*$$ 를 만족하는 복소 정사각 행렬로, **모든 고유값이 실수**이고 **유니터리 대각화** $$A=U\Lambda U^*$$ 가 가능합니다. 공분산/그래프 라플라시안/양자 해밀토니안 등 실전의 “핵심 행렬” 대부분이 여기에 속합니다.

---

## 1. 정의와 예시

### 1.1 정의
정사각 복소 행렬 $$A\in\mathbb{C}^{n\times n}$$ 에 대해
$$
\boxed{A=A^*}
$$
이면 **Hermitian(에르미트)** 행렬이라 부릅니다. 여기서 $$A^*$$ 는 **켤레 전치**(transpose 후 원소별 복소수 켤레)입니다.

### 1.2 빠른 예시
$$
A=
\begin{bmatrix}
2 & i\\
-i& 3
\end{bmatrix}
\qquad\Rightarrow\qquad
A^*=
\begin{bmatrix}
2 & -i\\
i& 3
\end{bmatrix}
\ \ \text{이므로}\ \ A=A^*
$$

- **대칭 행렬**은 실수에서의 **Hermitian** 특수 경우(복소 켤레가 의미 없으므로 $$A^T=A$$).
- **공분산 행렬**, **그래프 라플라시안**, **해밀토니안**(양자역학), **그람(Gram) 행렬** 등은 기본적으로 Hermitian.

---

## 2. 기본 성질(증명 스케치 포함)

| 성질 | 내용 |
|---|---|
| 대각 성분 | $$A_{ii}\in\mathbb{R}$$ |
| 켤레 대칭 | $$A_{ij}=\overline{A_{ji}}$$ |
| 정규성 | 항상 $$AA^*=A^*A$$(즉, **정규 행렬**) |
| 고유값 | **전부 실수** |
| 고유벡터 | 서로 **직교 가능** (서로 다른 고유값 대응 벡터는 직교) |
| 대각화 | **유니터리 대각화** 가능: $$A=U\Lambda U^*$$ |

### 2.1 고유값이 실수인 이유(스케치)
$$
A\vec v=\lambda \vec v,\ \ \vec v\neq 0.
$$
내적을 이용하면
$$
\langle A\vec v,\vec v\rangle=\langle \lambda\vec v,\vec v\rangle=\lambda\langle \vec v,\vec v\rangle,
$$
한편 Hermitian 성질로
$$
\langle A\vec v,\vec v\rangle=\langle \vec v,A\vec v\rangle=\overline{\lambda}\langle \vec v,\vec v\rangle.
$$
따라서 $$\lambda=\overline{\lambda}$$, 즉 실수.

### 2.2 서로 다른 고유값의 고유벡터는 직교
$$
A\vec v=\lambda \vec v,\ A\vec w=\mu \vec w,\ \lambda\neq \mu
$$ 에 대해
$$
\lambda\langle \vec v,\vec w\rangle=\langle A\vec v,\vec w\rangle=\langle \vec v,A\vec w\rangle=\mu\langle \vec v,\vec w\rangle
$$
이므로 $$\langle \vec v,\vec w\rangle=0$$.

---

## 3. 스펙트럼 정리(에르미트)

**스펙트럼 정리**: Hermitian $$A$$ 에 대해 **유니터리** $$U$$ 와 **실수 대각** $$\Lambda$$ 가 존재하여
$$
\boxed{A=U\Lambda U^*},\qquad \Lambda=\operatorname{diag}(\lambda_1,\dots,\lambda_n),\ \lambda_i\in\mathbb{R}.
$$
- 열벡터 $$u_i$$ 는 **정규직교** 고유벡터, 즉 $$U^*U=UU^*=I$$.
- 함수 계산(**functional calculus**): 적당한 함수 $$f$$ 에 대해
  $$
  \boxed{f(A)=U\,f(\Lambda)\,U^*=\sum_i f(\lambda_i)\,u_i u_i^*}.
  $$
  예: $$\exp(A),\ \sqrt{A},\ (A+\alpha I)^{-1}$$ 등.

---

## 4. 양의 정부호/반정부호, 이차형식, 레일리 몫

### 4.1 부호성과 이차형식
Hermitian $$A$$ 와 임의 벡터 $$x$$ 에 대해 **이차형식**
$$
\boxed{x^* A x \in \mathbb{R}}
$$
입니다(실수!). 이를 통해
- **양의 정부호(PSD)**: $$x^*Ax\ge 0\ \forall x$$  ⇔ 모든 고유값 $$\lambda_i\ge 0$$  ⇔ **Cholesky** 분해 가능(수치적으로 대칭/PSD 판단 기준).
- **양의 정정보호(PD)**: $$x^*Ax>0\ \forall x\ne 0$$ ⇔ 모든 고유값 $$\lambda_i>0$$.

### 4.2 레일리 몫(Rayleigh quotient)과 최대/최소
$$
\boxed{R_A(x)=\frac{x^*Ax}{x^*x}}
$$
은 $$A$$ 의 스펙트럼을 “스캔”합니다:
$$
\boxed{\lambda_{\min}=\min_{x\ne 0} R_A(x)\le R_A(x)\le \max_{x\ne 0} R_A(x)=\lambda_{\max}.}
$$

### 4.3 Courant–Fischer(최소최대 정리, 개요)
$$
\lambda_k=\min_{\dim S=k}\ \max_{x\in S\setminus\{0\}} R_A(x)
\ =\ \max_{\dim T=n-k+1}\ \min_{x\in T\setminus\{0\}} R_A(x).
$$
→ **주성분/최소제곱/스펙트럴 클러스터링** 등 최적화 해석의 바탕.

---

## 5. 자주 쓰는 예와 해석

### 5.1 공분산/그람(Gram) 행렬
데이터 행렬 $$X\in\mathbb{R}^{m\times n}$$ 의 평균 중심화 후 공분산:
$$
C=\frac{1}{m-1}X^T X\ \ (\text{실대칭}=\text{Hermitian}).
$$
모든 고유값이 **비음수**(PSD). PCA의 이론적 기반.

### 5.2 그래프 라플라시안
무향 그래프에서
$$
L=D-A
$$
(정규화 버전도), $$L=L^*\ (\text{실대칭})$$, 고유값 **비음수**. 두 번째 고유값(Fiedler 값)은 연결성/분할력.

### 5.3 양자역학(해밀토니안)
$$
H=H^*
$$
→ 고유값(에너지 준위)은 **실수**, 시간발전은 유니터리. **관측가능량**은 항상 Hermitian.

---

## 6. 수치 계산과 실전 팁

- **eigh vs eig**: Hermitian에는 **`np.linalg.eigh`**(실수 스펙트럼/정규직교 벡터 보장, 수치안정) 사용 권장. 일반 `eig`는 복소 스펙트럼 전용.
- **정렬**: `eigh`는 보통 오름차순으로 고유값 반환.
- **조건수**: PD인 Hermitian은 Cholesky 분해로 안정적으로 역/선형계 풀이.
- **대규모**: 희소/대형은 **Lanczos**(대칭 전용), LOBPCG 등 반복법.

---

## 7. 손으로 푸는 2×2 복합 예제

다음 Hermitian:
$$
A=
\begin{bmatrix}
2 & i\\
-i& 2
\end{bmatrix}.
$$

### 7.1 고유값
특성다항식:
$$
\det(A-\lambda I)=
\begin{vmatrix}
2-\lambda & i\\
-i & 2-\lambda
\end{vmatrix}
=(2-\lambda)^2-(-i)(i)=(2-\lambda)^2-1.
$$
따라서
$$
(2-\lambda)^2=1\ \Rightarrow\ \lambda\in\{1,3\}.
$$

### 7.2 고유벡터
- $$\lambda=3$$:
  $$
  (A-3I)=\begin{bmatrix}-1 & i\\ -i & -1\end{bmatrix},\quad
  -x+iy=0\Rightarrow x=iy.
  $$
  하나의 선택: $$\vec v_1=\begin{bmatrix}i\\1\end{bmatrix}$$.
  정규화:
  $$
  \|\vec v_1\|^2=i\overline{i}+1\cdot \overline{1}=1+1=2\Rightarrow
  u_1=\frac{1}{\sqrt2}\begin{bmatrix}i\\1\end{bmatrix}.
  $$
- $$\lambda=1$$:
  $$
  (A-I)=\begin{bmatrix}1 & i\\ -i & 1\end{bmatrix},\quad
  x+iy=0\Rightarrow x=-iy.
  $$
  $$\vec v_2=\begin{bmatrix}-i\\1\end{bmatrix},\ \ \|v_2\|=\sqrt2,\ \ u_2=\frac{1}{\sqrt2}\begin{bmatrix}-i\\1\end{bmatrix}.
  $$

유니터리 $$U=[u_1\ u_2]$$, 대각 $$\Lambda=\operatorname{diag}(3,1)$$ 로
$$
A=U\Lambda U^*.
$$

---

## 8. Python 실습(NumPy): 판정·대각화·최소최대·Cholesky

```python
import numpy as np

def is_hermitian(A, tol=1e-12):
    return np.allclose(A, A.conj().T, atol=tol)

# 1. Hermitian 판정과 고유분해
A = np.array([[2, 1j],
              [-1j, 2]], dtype=complex)

print("Hermitian?", is_hermitian(A))

# Hermitian 전용 고유분해
lam, U = np.linalg.eigh(A)
print("eigenvalues (real):", lam)      # [1. 3.]
print("U^*U≈I ?", np.allclose(U.conj().T @ U, np.eye(A.shape[0])))

# 복원 확인
Lambda = np.diag(lam)
A_rec = U @ Lambda @ U.conj().T
print("||A - UΛU^*||_F =", np.linalg.norm(A - A_rec))

# 2. Rayleigh quotient로 λ_min/λ_max 확인
def rayleigh(A, x):
    x = x.astype(complex)
    return (x.conj().T @ A @ x) / (x.conj().T @ x)

rng = np.random.default_rng(0)
xs = rng.normal(size=(1000, 2)) + 1j*rng.normal(size=(1000, 2))
rq_vals = np.array([rayleigh(A, x) for x in xs])
print("min R_A(x) ~", np.min(rq_vals.real), "  max R_A(x) ~", np.max(rq_vals.real))

# 3. PSD 판정과 Cholesky (A가 PSD면 가능)
# 예: B = [[2,1],[1,2]] 는 실대칭(=Hermitian)이고 PD
B = np.array([[2.,1.],
              [1.,2.]])
L = np.linalg.cholesky(B)  # PD면 성공
print("Cholesky L:\n", L)
```

**포인트**
- Hermitian은 `eigh` 사용 → 실수 고유값/정규직교 벡터 보장.
- Rayleigh 몫 샘플의 최소/최대가 고유값 범위를 근사.
- PD면 Cholesky가 존재(빠르고 안정적인 선형계/최적화에 필수).

---

## 9. 실전 응용 시나리오

### 9.1 PCA/통계
공분산 $$C=X^TX/(m-1)$$ 은 Hermitian(실대칭) & PSD.
주성분은 $$C$$ 의 고유벡터, 분산은 고유값.

### 9.2 그래프·네트워크
무향 그래프 라플라시안 $$L=L^*$$, **비음수 스펙트럼**.
두 번째 고유값은 연결성/커뮤니티 분할을 반영(스펙트럴 클러스터링).

### 9.3 신호·이미지
필터/커널이 자기수반이면 위상과 에너지를 다루기 유리.
위너 필터/정규화/역문제에서 Hermitian 정규화 항이 흔함.

### 9.4 양자 물리/머신러닝
해밀토니안 $$H=H^*$$, 관측량은 Hermitian.
양자 머신러닝/변분 알고리즘에서 기대값 $$\langle \psi|H|\psi\rangle$$ 계산은 Hermitian이기에 실수.

---

## 10. 자주 하는 실수와 체크리스트

- [ ] 복소 데이터에서 **대칭 $$A^T=A$$** 와 **Hermitian $$A=A^*$$** 를 혼동하지 않기.
- [ ] Hermitian이 아닌데 `eigh` 사용 → 결과 보증 불가. 일반 행렬은 `eig`.
- [ ] PSD 추정 시 수치 오차(작은 음수 고유값) → **클리핑**(예: $$\max(\lambda_i,0)$$) 후 재구성.
- [ ] 대규모 문제에서 전치/켤레 전치의 비용과 메모리 주의(희소 구조 활용).

---

## 11. 심화 토막지식

- **Gershgorin 원반**: Hermitian이면 모든 원반이 실축에 대해 대칭 → 스펙트럼이 **실수축**에만 놓임.
- **Weyl 부등식(섭동)**: $$A=H+E$$, 작은 Hermitian 섭동에서 고유값 이동이 제한됨(안정성 근거).
- **Interlacing(끼워넣기)**: 서브행렬의 고유값이 원행렬 고유값 사이에 끼어듦(샘플/서브그래프 분석에 유용).
- **Lanczos 반복**: 대규모 Hermitian의 극값 고유쌍 근사에 특화.

---

## 12. 연습문제(스스로 해보기)

1) 무작위 복소 행렬 $$M$$ 에 대해 $$A=\tfrac12(M+M^*)$$ 를 만들고 `eigh`로 고유분해. $$A$$ 가 Hermitian/실수 스펙트럼임을 확인.
2) 임의의 PD Hermitian $$A$$ 에 대해 Cholesky로 $$A=LL^*$$ 를 구하고, `solve(L^* y=b)`→`solve(L x=y)` 로 선형계 풀이.
3) 레일리 몫을 이용해 **파워 메서드**로 $$\lambda_{\max}$$, **역파워**로 $$\lambda_{\min}$$ 근사.

---

## 13. 요약 표

| 항목 | Hermitian 핵심 |
|---|---|
| 정의 | $$A=A^*$$ (켤레 전치 대칭) |
| 스펙트럼 | 고유값 **모두 실수**, 고유벡터 **직교 가능** |
| 대각화 | $$A=U\Lambda U^*$$ (**유니터리 대각화**) |
| 부호성 | $$x^*Ax\ge 0 \Leftrightarrow A\succeq 0$$ (PSD) |
| 실전 | 공분산, 라플라시안, 해밀토니안, 그람, 필터 |
| 수치 | `eigh` 사용, PD면 Cholesky, 대규모는 Lanczos |

---

## 14. 추가 Python 예제: 무작위 Hermitian 생성·검증

```python
import numpy as np

rng = np.random.default_rng(42)
n = 5

# 무작위 복소 행렬에서 Hermitian 생성
M = rng.normal(size=(n,n)) + 1j*rng.normal(size=(n,n))
A = (M + M.conj().T) / 2  # Hermitian 보장

# 검증
print("Hermitian?", np.allclose(A, A.conj().T))

# eigh
lam, U = np.linalg.eigh(A)
print("eigs(real):", np.round(lam, 6))
print("U^*U≈I ?", np.allclose(U.conj().T @ U, np.eye(n)))

# 재구성
A_rec = U @ np.diag(lam) @ U.conj().T
print("||A - UΛU^*||_F =", np.linalg.norm(A - A_rec))

# PSD로 만들고 싶다면(고유값 클리핑)
lam_psd = np.clip(lam, 0, None)
A_psd = U @ np.diag(lam_psd) @ U.conj().T
w_psd = np.linalg.eigvalsh(A_psd)
print("A_psd min eig:", np.min(w_psd))  # ≥ 0 이어야 함
```

---

### 마무리
에르미트 행렬은 “복소수 세계의 대칭”입니다. **실수 스펙트럼 + 유니터리 대각화**라는 강력한 구조 덕분에, 이론·알고리즘·실무(데이터/신호/그래프/양자)까지 **하나의 공통 언어**로 연결해 줍니다.
