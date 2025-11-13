---
layout: post
title: 선형대수 - Singular Value Decomposition
date: 2025-06-22 19:20:23 +0900
category: 선형대수
---
# SVD (Singular Value Decomposition)

> **핵심 한 줄**
> 임의의 행렬 $$A\in\mathbb{R}^{m\times n}$$는 항상 **직교(정규직교) 회전 + 축 방향 스케일링 + 직교 회전**의 합성으로 분해된다:
> $$\boxed{A=U\,\Sigma\,V^\top}$$
> 여기서 $$U\in\mathbb{R}^{m\times m},\ V\in\mathbb{R}^{n\times n}$$는 **직교 행렬**, $$\Sigma\in\mathbb{R}^{m\times n}$$는 대각(비음수 특이값) 행렬.

---

## 1. SVD의 정식 정의와 구성요소

- $$A=U\Sigma V^\top$$
  - $$U=[\vec{u}_1,\ldots,\vec{u}_m]$$: **좌측 특이벡터(Left singular vectors)**, $$U^\top U=I_m$$
  - $$V=[\vec{v}_1,\ldots,\vec{v}_n]$$: **우측 특이벡터(Right singular vectors)**, $$V^\top V=I_n$$
  - $$\Sigma=\operatorname{diag}(\sigma_1,\ldots,\sigma_r)\in\mathbb{R}^{m\times n}$$ (대각선만 비영),
    $$\sigma_1\ge\cdots\ge\sigma_r>0,\ r=\operatorname{rank}(A)$$

> **Thin/경제 SVD(실무 기본형)**
> $$A=U_r\,\Sigma_r\,V_r^\top$$,
> $$U_r\in\mathbb{R}^{m\times r},\ \Sigma_r\in\mathbb{R}^{r\times r},\ V_r\in\mathbb{R}^{n\times r}$$

---

## 2. 고유값 분해와 무엇이 다른가?

| 구분 | 고유값 분해 | SVD |
|---|---|---|
| 대상 | 정방행렬만 | **모든** 직사각형 행렬 |
| 형태 | $$A=PDP^{-1}$$ (일반), $$A=Q\Lambda Q^\top$$ (대칭) | $$A=U\Sigma V^\top$$ (항상 가능) |
| 수치안정성 | 비대칭/결손 시 민감 | **매우 안정적**(직교 기저) |
| 값의 의미 | 고유값: 방향 보존 배율 | **특이값**: 축 길이(스케일) |

> 연결고리: $$A^\top A\,\vec{v}_i=\lambda_i\vec{v}_i,\ \ \sigma_i=\sqrt{\lambda_i}$$,
> $$A A^\top \vec{u}_i=\lambda_i\vec{u}_i$$

---

## 3. 기하학적 의미—원판을 타원으로

- 단위 구(원판) $$\{x:\|x\|_2=1\}$$를 $$A$$로 사상하면 **타원(타원체)**가 된다.
  - 주축 방향: **우측 특이벡터** $$\vec{v}_i$$
  - 이미지의 축 방향: **좌측 특이벡터** $$\vec{u}_i$$
  - 축 길이: **특이값** $$\sigma_i$$
- 즉, $$A= \underbrace{V^\top}_{\text{회전}}\ \underbrace{\Sigma}_{\text{축별 스케일}}\ \underbrace{U}_{\text{회전}}$$(표기 관점에 따라 순서가 바뀌어 보일 수 있으나 본질은 “회전–스케일–회전”).

---

## 4. SVD가 주는 핵심 정보

- **스펙트럴 노름**: $$\|A\|_2=\sigma_{\max}=\sigma_1$$
- **프로베니우스 노름**: $$\|A\|_F=\sqrt{\sum_i\sigma_i^2}$$
- **조건수**(정사각, 풀랭크): $$\kappa_2(A)=\frac{\sigma_{\max}}{\sigma_{\min}}$$ → 수치적 민감도
- **랭크**: 양의 특이값의 개수
- **영공간/열공간**:
  - $$\text{Null}(A)=\{\vec{x}:V\Sigma^\top U^\top \vec{x}=\vec{0}\}=\operatorname{span}\{\vec{v}_{r+1},\ldots,\vec{v}_n\}$$
  - $$\text{Col}(A)=\operatorname{span}\{\vec{u}_1,\ldots,\vec{u}_r\}$$

---

## 5. 최적 저랭크 근사 (Eckart–Young–Mirsky 정리)

- **최적의 랭크-$$k$$ 근사** $$A_k$$(스펙트럴/프로베니우스 노름 모두에서 최적):
  $$
  \boxed{A_k=\sum_{i=1}^k \sigma_i\,\vec{u}_i\vec{v}_i^\top=U_k\,\Sigma_k\,V_k^\top}
  $$
- 오차:
  $$
  \|A-A_k\|_2=\sigma_{k+1},\quad
  \|A-A_k\|_F=\sqrt{\sum_{i>k}\sigma_i^2}
  $$
- **압축/노이즈 제거/주성분 보존**의 이론적 근거.

---

## 6. 최소제곱, 유사역행렬, 정규화

### 6.1 모어–펜로즈 유사역행렬
- $$A^+=V\,\Sigma^+\,U^\top,\quad \Sigma^+=\operatorname{diag}(\sigma_1^{-1},\ldots,\sigma_r^{-1})$$
- **최소제곱 해** $$\vec{x}_\star=A^+\vec{b}$$ (해가 없을 때 잔차 최소)

### 6.2 Tikhonov(릿지) 정규화의 SVD 표현
- $$\vec{x}_\lambda=(A^\top A+\lambda I)^{-1}A^\top \vec{b}
=V\,\operatorname{diag}\!\Big(\frac{\sigma_i}{\sigma_i^2+\lambda}\Big)U^\top \vec{b}$$
- 작은 $$\sigma_i$$에 대한 **필터링 효과**로 과적합/불안정 완화.

---

## 7. PCA와의 직접 연결

- 데이터 행렬 $$X\in\mathbb{R}^{n\times d}$$(평균 제거) **SVD**: $$X=U\Sigma V^\top$$
- 공분산: $$C=\frac{1}{n}X^\top X = V\,\frac{\Sigma^\top \Sigma}{n}\,V^\top$$
- **주성분 방향**: $$V$$, **분산(설명력)**: $$\sigma_i^2/n$$
- 누적 설명률: $$\sum_{i=1}^k \sigma_i^2 \Big/ \sum_j \sigma_j^2$$

---

## 8. 계산법·복잡도·대규모 데이터

- 고전 알고리즘: **Golub–Kahan bidiagonalization** → QR/Divide-and-Conquer
- 복잡도(대략): $$\mathcal{O}(mn\min\{m,n\})$$
- **큰 행렬**:
  - **Truncated/Partial SVD** (상위 $$k$$만): Lanczos/IRLBA/PROPACK
  - **Randomized SVD**: $$\mathcal{O}(mn\log k + (m+n)k^2)$$ 근사, 대규모에서 효율
- 수치팁: 스케일링/중심화, 정칙화, **full_matrices=False**(경제 SVD) 사용

---

## 9. 자주 놓치는 디테일

- **부호(또는 복소위상) 불일정성**: $$\vec{u}_i,\vec{v}_i$$는 동시에 부호 반전 가능(결과는 동일).
- **중복 특이값**: 해당 부분공간의 기저(특이벡터)는 **비고유적**.
- **랭크 판정 임계치**: $$\sigma_i>\tau$$(예: $$\tau=\max(m,n)\,\sigma_{\max}\,\epsilon_{\text{machine}}$$).
- **정방행렬의 SVD**와 **고유값 분해**는 다름(특히 비대칭일 때).

---

## 10. 손계산 감각—작은 예제

### 10.1 간단 2×2
$$
A=\begin{bmatrix}3&1\\0&2\end{bmatrix}.
$$
- $$A^\top A=\begin{bmatrix}9&3\\3&5\end{bmatrix}$$의 고유값을 풀면
  $$\lambda_{1,2}=\frac{14\pm\sqrt{14^2-4\cdot(45-9)}}{2}=\frac{14\pm\sqrt{160}}{2}=7\pm2\sqrt{10}$$
  ⇒ 특이값 $$\sigma_{1,2}=\sqrt{7\pm 2\sqrt{10}}$$
- 대응 우측 특이벡터 $$\vec{v}_i$$를 구하고, $$\vec{u}_i=A\vec{v}_i/\sigma_i$$로 좌측을 구해
  $$A=U\Sigma V^\top$$ 완성(실무에선 라이브러리 사용 권장).

### 10.2 랭크 결손 3×2
$$
A=\begin{bmatrix}1&2\\2&4\\3&6\end{bmatrix}=\begin{bmatrix}1\\2\\3\end{bmatrix}\begin{bmatrix}1&2\end{bmatrix}.
$$
- $$\operatorname{rank}(A)=1$$, 특이값 하나만 양수.
- **최적 랭크-1 근사**는 사실상 자기 자신.

---

## 11. 파이썬 코드 (NumPy / PyTorch)

### 11.1 기본 SVD와 복원 (NumPy)
```python
import numpy as np

A = np.array([[3., 1.],
              [0., 2.]])

U, S, VT = np.linalg.svd(A, full_matrices=False)  # 경제 SVD
A_rec = U @ np.diag(S) @ VT

print("U=\n", U)
print("S=", S)
print("VT=\n", VT)
print("Reconstruction error ‖A - UΣVᵀ‖_F =", np.linalg.norm(A - A_rec, 'fro'))
```

### 11.2 Truncated SVD (상위 k만, NumPy)
```python
import numpy as np

# 예시 행렬 (m >> n일 수도, 여기선 6x4)
A = np.array([[1., 0., 2., 0.],
              [0., 3., 0., 0.],
              [1., 1., 0., 0.],
              [0., 0., 0., 4.],
              [1., 0., 0., 0.],
              [0., 0., 5., 0.]])

U, S, VT = np.linalg.svd(A, full_matrices=False)
k = 2
Uk, Sk, VTk = U[:, :k], S[:k], VT[:k, :]
A_k = Uk @ np.diag(Sk) @ VTk

err2 = np.linalg.norm(A - A_k, 2)      # ~= σ_{k+1}
errF = np.linalg.norm(A - A_k, 'fro')  # = sqrt(sum_{i>k} σ_i^2)

print("Top-2 singular values:", Sk)
print("Spectral-norm error ≈", err2)
print("Frobenius-norm error =", errF)
```

### 11.3 최소제곱/유사역행렬 (NumPy)
```python
import numpy as np

A = np.array([[1., 1.],
              [0., 1.],
              [1., 0.]])
b = np.array([2., 1., 3.])

U, S, VT = np.linalg.svd(A, full_matrices=False)
A_pinv = VT.T @ np.diag(1.0 / S) @ U.T    # 모어-펜로즈
x_ls = A_pinv @ b                         # 최소제곱 해
res = b - A @ x_ls

print("x_ls =", x_ls)
print("Residual norm:", np.linalg.norm(res))
print("Q-orthogonality (Uᵀ res):", U.T @ res)  # 앞 r개는 거의 0
```

### 11.4 릿지 정규화의 필터 계수 (NumPy)
```python
import numpy as np

A = np.random.randn(100, 10)
x_true = np.random.randn(10)
b = A @ x_true + 0.05*np.random.randn(100)

U, S, VT = np.linalg.svd(A, full_matrices=False)

lam = 1e-1
f = S / (S**2 + lam)               # 필터
x_ridge = VT.T @ (f * (U.T @ b))   # V diag(f) Uᵀ b

print("ridge solution norm:", np.linalg.norm(x_ridge))
```

### 11.5 PyTorch 버전 (GPU 가속 가능)
```python
import torch

A = torch.tensor([[3., 1.],
                  [0., 2.]])

# full_matrices=False 가 경제 SVD
U, S, Vh = torch.linalg.svd(A, full_matrices=False)
A_rec = U @ torch.diag(S) @ Vh

print("U=\n", U)
print("S=", S)
print("Vh=\n", Vh)
print("‖A - UΣVᵀ‖_F =", torch.linalg.matrix_norm(A - A_rec, ord='fro').item())
```

> **Tip**: 대규모 데이터에서는 `sklearn.utils.extmath.randomized_svd`나
> PyTorch의 `torch.pca_lowrank`(PCA 근사) 같은 **근사/부분 SVD** 사용을 검토.

---

## 12. 이미지/문서 예시 시나리오

### 12.1 이미지 압축(회색조 행렬)
- 이미지 행렬 $$I\in\mathbb{R}^{m\times n}$$에 대해 상위 $$k$$ 특이값만으로
  $$I_k=U_k\Sigma_k V_k^\top$$ 을 저장하면 **저용량 근사**.
- 품질–용량 트레이드오프: $$k$$↑ ⇒ 품질↑, 용량↑.

#### 코드 스케치(파일 IO는 환경에 맞게 대체)
```python
import numpy as np
# img: m x n 실수 행렬(0~255 등)이라 가정
U, S, VT = np.linalg.svd(img, full_matrices=False)
k = 50
img_k = (U[:, :k] @ np.diag(S[:k]) @ VT[:k, :]).clip(0, 255)
```

### 12.2 LSA(잠재 의미 분석)
- 단어–문서 행렬 $$X$$에 SVD 적용: $$X=U\Sigma V^\top$$
  상위 $$k$$ 성분만 유지: **잠재 의미 공간**에서 유사도 계산.

---

## 13. 체크리스트—SVD를 고를 때

1. **형태**: 정방이든 아니든 **항상 SVD 가능**.
2. **안정성**: 직교 기저 → 수치안정 우수.
3. **랭크/조건수**: 특이값 스펙트럼으로 즉시 파악.
4. **근사**: Eckart–Young–Mirsky로 최적 저랭크 보장.
5. **해결**: 최소제곱/유사역행렬/릿지 모두 SVD 기반이 명료.
6. **규모**: 커지면 **부분·랜덤라이즈드 SVD** 사용.

---

## 14. 연습문제(핸즈온 권장)

1) 무작위 $$A\in\mathbb{R}^{200\times 50}$$에 노이즈를 섞고 상위 $$k$$로 복원할 때
   $$\|A-A_k\|_F/\|A\|_F$$가 $$k$$에 따라 어떻게 변하는지 그려보라.
2) 컬러 이미지(RGB) 압축: 채널별 SVD 후 결합, PSNR 비교.
3) 다중공선성이 강한 회귀 데이터에서 **릿지**와 **단순 최소제곱**의 차이를
   특이값 필터링 관점에서 설명하고 실험으로 확인하라.
4) 작은 문서–단어 행렬로 LSA 공간을 만들고, 유사한 문서를 코사인 유사도로 찾아보라.

---

## 마무리

**SVD는 “모든 행렬”을 가장 잘 보이게 정렬해 주는 확대경**이다.
스케일(특이값)과 방향(특이벡터)을 분해해 **랭크·조건·근사·정규화**를 한눈에 풀어주며,
PCA·압축·추천·NLP·신호처리에서 **표준 도구**로 쓰인다.
