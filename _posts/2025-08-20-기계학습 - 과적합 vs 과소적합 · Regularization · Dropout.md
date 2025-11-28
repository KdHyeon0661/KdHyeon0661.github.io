---
layout: post
title: 기계학습 - 과적합 vs 과소적합 · Regularization · Dropout
date: 2025-08-20 17:25:23 +0900
category: 기계학습
---
# 과적합 vs 과소적합 · Regularization · Dropout

## 1. 학습/검증/테스트와 일반화

### 1.1 데이터 분할과 일반화 오류

기계학습에서 우리가 최소화하고 싶은 것은 단순히 **훈련오차**가 아니라,  
보이지 않는 새로운 데이터에 대한 **일반화오차**다.

- **훈련셋(Train)**: 파라미터 학습에 사용
- **검증셋(Validation)**: 하이퍼파라미터(정규화 강도, 드롭아웃 비율 등) 선택
- **테스트셋(Test)**: 최종 성능 평가(한 번만 사용)

모형 \(\hat f\)에 대해, 진짜로 줄이고 싶은 것은

$$
R(\hat f) = \mathbb{E}_{(x,y)\sim \mathcal{D}}[L(y,\hat f(x))],
$$

이지만, 실제로는 분포 \(\mathcal{D}\)를 모르므로 **경험위험(ERM)**

$$
\hat R(\hat f) = \frac{1}{n}\sum_{i=1}^n L(y_i, \hat f(x_i))
$$

을 대신 최소화한다. 과적합은 **\(\hat R\)는 낮은데 \(R\)는 큰 상태**라 볼 수 있다.

---

## 2. 과적합(Overfitting) vs 과소적합(Underfitting)

### 2.1 개념 요약

- **과적합**: 학습 데이터의 **노이즈/우연 패턴**까지 외워  
  **훈련 성능은 높지만 검증/테스트 성능이 떨어지는** 상태.
- **과소적합**: 모델 용량이 부족하거나 학습이 불충분해  
  **훈련 성능부터 낮은** 상태.

정성적으로는 다음과 같다고 볼 수 있다.

- 과적합: “시험에 나온 예제만 달달 외웠는데, 응용문제는 못 푸는 학생”
- 과소적합: “기본 개념 자체를 제대로 이해하지 못한 학생”

### 2.2 학습/검증 곡선 징후

훈련/검증 손실 곡선을 보면 두 상황을 비교적 명확하게 구분할 수 있다.

| 상황 | 훈련 손실 | 검증 손실 | 특징 |
|---|---|---|---|
| 과적합 | 계속 감소 | 어느 시점 이후 **상승** | 모델 복잡, 데이터 적음, 정규화 약함 |
| 과소적합 | **높게 유지** | 높게 유지 | 모델 단순, 학습률 부적절, 학습 부족 |
| 적정 학습 | 낮음 | 낮음(또는 훈련과 비슷한 수준으로 평행) | 일반화 양호 |

실제 프로젝트에서는 **에포크에 따른 손실/정확도 곡선을 항상 그려보는 것**이 중요하다.

### 2.3 바이어스–분산 트레이드오프

회귀 문제에서, 입력 \(x\), 정답 \(y=f^\*(x)+\epsilon\) (잡음 \(\epsilon\) 평균 0, 분산 \(\sigma^2\))라고 하자.  
모형 \(\hat f\)의 기대 MSE는 다음처럼 분해된다.

$$
\mathbb{E}\big[(y-\hat f(x))^2\big]
=\underbrace{\big(\mathbb{E}[\hat f(x)]-f^\*(x)\big)^2}_{\text{Bias}^2}
+\underbrace{\mathbb{Var}[\hat f(x)]}_{\text{Variance}}
+\underbrace{\sigma^2}_{\text{Irreducible Noise}}.
$$

- **Bias(바이어스)**: 모델이 **너무 단순**해서 구조를 제대로 표현하지 못하는 정도  
  → 과소적합과 연결
- **Variance(분산)**: 같은 데이터 분포에서도 **훈련셋이 조금만 바뀌어도 모델이 크게 바뀌는 정도**  
  → 과적합과 연결
- **Irreducible Noise**: 어떤 모델로도 줄일 수 없는 데이터 고유의 잡음

**모델 용량을 키우면:**

- 바이어스는 감소(더 복잡한 함수를 표현할 수 있음)
- 분산은 증가(훈련 데이터에 민감해짐, 과적합 위험)

**정규화 강도를 키우면:**

- 모델을 더 단순하게 만들어 바이어스를 증가
- 대신 분산을 줄여 일반화를 향상시킬 수 있음

결국, 정규화는 **Bias–Variance의 균형점을 찾기 위한 도구**다.

### 2.4 원인↔처방 매핑

#### 과적합일 때 (훈련↑·검증↓)

가능한 원인

- 모델이 너무 복잡(깊은/넓은 네트워크, 파라미터 수 과다)
- 데이터가 적거나, 노이즈가 많음
- 정규화(Weight Decay, Dropout 등)가 약함
- 데이터 증강이 부족
- 학습을 너무 오래, 또는 너무 공격적으로 진행

대표 처방

- **정규화 강화** (L2/L1/ElasticNet, Dropout, Label Smoothing 등)
- **데이터 증가/증강**
- **조기 종료(Early Stopping)**
- **모델 단순화**(층 수/폭 줄이기, 파라미터 공유, 지식 증류)
- **학습률 스케줄 활용**(고 LR로 빠르게 수렴 후 점진 감소)

#### 과소적합일 때 (훈련부터 성능 낮음)

가능한 원인

- 모델 용량 자체가 너무 작음
- 학습률이 너무 작아서 수렴이 충분히 안 됨
- 학습 횟수(에포크)가 부족
- 피처/표현력이 부족
- 정규화가 너무 강함(Dropout, Weight Decay 과도)

대표 처방

- **모델 확대**(층/폭 증가, 더 리치한 아키텍처)
- **학습 더 오래**(에포크 수 증가), 학습률 튜닝
- **정규화 완화**(Dropout 비율 감소, WD 감소)
- **특성 공학/전이학습** 활용

---

## 3. 정규화(Regularization)의 수학적 기본 틀

### 3.1 목적 함수(일반형)

대부분의 정규화 기법은 다음과 같은 목적 함수를 공유한다.

$$
\min_{\theta}\ \frac{1}{n}\sum_{i=1}^n L\!\big(y_i,\hat{y}(x_i;\theta)\big)\ +\ \lambda\,\Omega(\theta),
$$

- 첫 번째 항: 경험손실(훈련 데이터에 대한 평균 손실)
- 두 번째 항: **정규화 항(Regularizer)** \(\Omega(\theta)\)
- \(\lambda>0\): 정규화 강도 하이퍼파라미터  
  \(\lambda\)가 클수록 규제가 강해져 **모델이 더 단순해지고, 바이어스↑·분산↓** 쪽으로 이동한다.

여기서 \(\Omega(\theta)\)를 어떻게 정의하느냐에 따라  
L2, L1, Elastic Net, Group Lasso, Nuclear Norm 등 다양한 정규화가 생긴다.

---

## 4. L2(Largest 사용) 정규화: Ridge, Weight Decay

### 4.1 정의와 업데이트 식

L2 정규화(릿지)는

$$
\Omega(\theta)=\tfrac{1}{2}\|\theta\|_2^2
$$

를 사용하는 형태다.  
가중치 \(w\)에 대한 SGD 업데이트는

$$
w \leftarrow (1-\alpha\lambda)\,w\ -\ \alpha\nabla_w L
$$

이 되는데, 여기서 \(\alpha\)는 학습률이다.

- 첫 항 \((1-\alpha\lambda)\,w\): 가중치를 **조금 0 쪽으로 당기는 효과**  
  → **Weight Decay**라고 부른다.
- 둘째 항 \(-\alpha\nabla_w L\): 원래 손실에 따른 그라디언트 업데이트

즉, L2 정규화는 매 스텝마다 “조금씩 원점 쪽으로 당기는 힘”을 추가하는 것과 같다.

### 4.2 선형 회귀에서의 Ridge 해

정규화 없는 선형 회귀의 최소제곱 해는

$$
\hat w_{\text{OLS}} = (X^\top X)^{-1}X^\top y
$$

인데, Ridge 회귀는

$$
\hat w_{\text{Ridge}} = (X^\top X + \lambda I)^{-1}X^\top y
$$

가 된다.  
\(\lambda I\)를 더해줌으로써

- 특잇값이 작은 방향을 축소 → **불안정한 방향(고분산 방향)을 억제**
- 역행렬을 안정적으로 계산할 수 있게 해준다.

### 4.3 베이지안 관점

L2 정규화를 베이지안 관점에서 보면,  
가중치에 대해

$$
w \sim \mathcal{N}(0, \tau^2 I)
$$

라는 가우시안 사전을 둔 것과 같다.  
정규화 항 \(\frac{\lambda}{2}\|w\|^2\)는 로그 사전분포에 해당하고,  
MLE 대신 MAP 추정으로 해석된다.

- \(\lambda\)가 크다 = \(\tau^2\)가 작다 = “가중치는 0 근처일 것”이라는 **강한 사전 신념**

이 관점은 딥러닝에서도 그대로 확장된다.

---

## 5. L1(Lasso)와 희소성, Elastic Net

### 5.1 L1 정규화: Lasso

L1 정규화는

$$
\Omega(\theta)=\|w\|_1=\sum_j |w_j|
$$

를 사용하는 형태다.  
L2와 달리 미분가능하지 않기 때문에, 좌표강하법이나 서브그래디언트를 사용한다.

특징:

- 많은 계수를 **정확히 0으로 만들 수 있음** → **특징 선택 효과**
- 다중공선성이 있는 피처들 중 일부를 선택해 나머지는 0으로 만든다.
- 고차원 희소 모델(예: 수천 개의 피처 중 몇 개만 중요)을 만들 때 유용하다.

### 5.2 Elastic Net

L1만 쓰면 상관이 높은 피처들 중 하나만 남기고 나머지는 0으로 만들어  
불안정한 선택이 될 수 있다. 이를 보완하는 것이 Elastic Net이다.

$$
\Omega(\theta)=\alpha\|w\|_1 + (1-\alpha)\tfrac{1}{2}\|w\|_2^2,\quad 0\le\alpha\le1.
$$

- L1의 **희소성**과
- L2의 **안정성/그룹화 효과**를 함께 얻을 수 있다.

### 5.3 scikit-learn 예제: L1 vs L2 vs Elastic Net

```python
import numpy as np
from sklearn.linear_model import LogisticRegressionCV
from sklearn.preprocessing import StandardScaler
from sklearn.pipeline import Pipeline
from sklearn.datasets import make_classification
from sklearn.metrics import classification_report

# 가상 데이터 생성: 실제로는 유럽/북미 고객 이탈 여부 같은 이진 분류를 상상해도 된다.
X, y = make_classification(
    n_samples=4000,
    n_features=50,
    n_informative=10,
    n_redundant=10,
    weights=[0.7, 0.3],
    random_state=42
)

pipelines = {
    "L2": Pipeline([
        ("scaler", StandardScaler()),
        ("clf", LogisticRegressionCV(
            Cs=20, cv=5, penalty="l2", solver="lbfgs", max_iter=5000,
            scoring="f1", n_jobs=-1
        ))
    ]),
    "L1": Pipeline([
        ("scaler", StandardScaler()),
        ("clf", LogisticRegressionCV(
            Cs=20, cv=5, penalty="l1", solver="saga", max_iter=5000,
            scoring="f1", n_jobs=-1
        ))
    ]),
    "ElasticNet": Pipeline([
        ("scaler", StandardScaler()),
        ("clf", LogisticRegressionCV(
            Cs=10, cv=5, penalty="elasticnet", solver="saga",
            l1_ratios=[0.3, 0.5, 0.7], max_iter=5000,
            scoring="f1", n_jobs=-1
        ))
    ]),
}

for name, pipe in pipelines.items():
    pipe.fit(X, y)
    print(f"\n== {name} ==")
    yhat = pipe.predict(X)
    print(classification_report(y, yhat, digits=3))
```

- L1 모델의 계수들을 보면 많은 피처가 0이 된다.
- Elastic Net은 L1만 썼을 때보다 조금 더 안정적인 선택을 한다.

---

## 6. 딥러닝에서의 정규화: Weight Decay, Early Stopping, Data Augmentation, Label Smoothing

### 6.1 Weight Decay와 AdamW

딥러닝에서 L2 정규화는 보통 **Weight Decay** 형태로 구현된다.  
다만, 적응형 옵티마(Adam 등)에서는 단순히 L2 항을 추가하는 것이  
“진짜 Weight Decay”와 수학적으로 동일하지 않다.

그래서 등장한 것이 **AdamW**(Decoupled Weight Decay)이다.

- Adam + L2: 그라디언트를 적응스케일링한 뒤 WD 효과가 섞인다.
- AdamW: **그라디언트 업데이트와 별개로, 파라미터에 일정 비율을 곱해 감소**시키는 방식

실무에서는

- `weight_decay ≈ 1e-4 ~ 5e-2` 범위에서 시작해서 튜닝하는 경우가 많다.
- 큰 비전 모델/언어 모델에서도 여전히 기본 요소로 쓰인다.

### 6.2 Early Stopping (암묵적 정규화)

Early Stopping은 정규화 항을 직접 추가하지 않지만,  
사실상 **정규화와 비슷한 역할**을 한다.

- 훈련 손실은 계속 감소하지만
- 검증 손실이 어느 시점 이후 증가하기 시작하면
- **검증 손실이 최소가 된 직후의 파라미터로 멈추는 것이다.**

수학적으로는, 특정 형태의 선형 모델에서 Early Stopping이  
L2 정규화와 유사한 효과를 가져온다는 결과가 알려져 있다.

### 6.3 데이터 증강(Data Augmentation)

데이터 증강은 **입력을 변형해서 가상의 신규 데이터**를 만드는 것이다.

- 원본 이미지 \(x\)에 대해 변환 \(T\)를 적용해 \(T(x)\)를 얻고,
- 레이블은 그대로 유지(또는 MixUp처럼 혼합)

이는 사실상 “모델이 **변환에 대해 불변/강건**하도록” 요구하는 정규화 효과를 갖는다.

자세한 증강 기법은 뒤에서 별도 섹션으로 정리한다.

### 6.4 Label Smoothing

분류에서 원-핫 타깃 대신,

- 정답 클래스: \(1-\varepsilon\)
- 오답 클래스들: \(\varepsilon/(K-1)\)

로 타깃을 부드럽게 만드는 기법이다.

- 모델이 **과도한 확신**을 갖지 못하게 한다.
- 특히 큰 클래스 수(수천, 수만)일 때 유용하다.
- Calibration 관점에서도 예측 확률을 더 현실적인 값으로 만들 수 있다.

Cross-Entropy with Label Smoothing은

$$
L = -(1-\varepsilon)\log p_{y}
 - \sum_{k\ne y}\frac{\varepsilon}{K-1}\log p_k
$$

형태가 되며, 각 클래스에 대한 로그확률을 조금씩 섞어준다.

---

## 7. Dropout: 원리·수학·변형·주의점

### 7.1 정의(학습 시 뉴런 무작위 비활성화)

은닉 표현 \(h\) (예: Linear/Conv 이후 활성화)와  
마스크 \(m\sim \mathrm{Bernoulli}(1-p)\)에 대해

$$
\tilde h=\frac{m\odot h}{1-p}
$$

로 정의하는 것이 **Inverted Dropout**이다.

- \(m\)은 각 뉴런(또는 채널)에 대해 0 또는 1을 샘플한다.
- 기대값은

  $$
  \mathbb{E}[\tilde h] = \mathbb{E}\Big[\frac{m\odot h}{1-p}\Big] = h
  $$

  이므로, 테스트 시에는 단순히 모든 뉴런을 켠 상태에서  
  별도의 스케일 조정 없이 사용할 수 있다.

### 7.2 왜 잘 작동하는가? (앙상블 관점)

Dropout은 다음과 같은 여러 서브모델을 동시에 학습하는 것과 비슷하다.

- 각 미니배치마다, **다른 서브네트워크**가 활성화된다.
- 파라미터는 공유되지만, 활성 경로가 다르다.
- 테스트 시에는 모든 경로를 동시에 사용하는 셈이므로  
  **암묵적으로 많은 모델을 평균내는 효과**를 낸다.

또한, 특정 뉴런/경로에 과도하게 의존하는 것을 막아

- **co-adaptation(공적응)**을 줄이고
- 보다 **강건한 특징**을 학습하는 데 도움을 준다.

### 7.3 어디·얼마나 쓸까?

실무에서의 예시 범위:

- **MLP/FC 레이어**: Dropout 비율 \(p \approx 0.3\sim0.5\)
- **Conv 네트워크**
  - 보통 Conv에는 Dropout을 적게 쓰거나,  
    **SpatialDropout/DropBlock** 같이 패턴 보존에 더 적합한 변형을 쓴다.
  - 비율은 0.1~0.3 정도가 일반적이다.
- **Transformer 계열**
  - `dropout ≈ 0.1`, `attention_dropout ≈ 0.1`
  - 깊은 네트워크에는 **Stochastic Depth(DropPath)**를 함께 적용하기도 한다.

과도한 Dropout은 **학습 속도 저하·과소적합**으로 이어질 수 있으므로  
작은 값에서 시작해 점차 늘려 보면서 성능을 확인하는 것이 좋다.

### 7.4 변형들

- **DropConnect**: 뉴런이 아니라 **가중치** 자체를 확률적으로 드롭
- **SpatialDropout**: Conv 출력의 채널 차원 전체를 같이 드롭
- **DropBlock**: 연속된 공간 블록 단위로 드롭 (객체 일부를 가리는 효과)
- **Variational Dropout**: RNN에서 시퀀스 전체에 대해 고정된 마스크 사용
- **MC Dropout**: 추론 시에도 드롭아웃을 켜서  
  **예측 불확실성**을 추정하는 기법

### 7.5 BN과의 순서

일반적인 권장 순서는

- **Conv/Linear → Norm(BN/LN) → 활성화(ReLU 등) → Dropout**

이다.  
다만, 특정 논문/라이브러리에서는 약간씩 변형된 순서를 사용하기도 한다.

---

## 8. 정규화 레시피: 작업별 직관 요약

### 8.1 Computer Vision

- 기본 증강: Random Crop/Flip/Color Jitter, Resize, Cutout 등
- 강력 증강:
  - **MixUp**

    $$
    \tilde x=\lambda x_i+(1-\lambda)x_j,\quad
    \tilde y=\lambda y_i+(1-\lambda)y_j,\quad
    \lambda\sim\mathrm{Beta}(\alpha,\alpha)
    $$

  - **CutMix**: 이미지 일부 패치를 다른 이미지 패치로 교체하고,  
    라벨도 면적 비율만큼 혼합
- Weight Decay + Dropout/DropBlock + Label Smoothing + EMA 등을 조합

### 8.2 NLP

- 간단 증강: 동의어 치환, 랜덤 삭제/스와프, 백트랜슬레이션, EDA
- 사전학습 언어모델 기반: 토큰 마스킹/노이즈 주입
- 정규화: Dropout 0.1, Label Smoothing 0.1 내외, LayerDrop/DropPath

### 8.3 오디오/음성

- **SpecAugment**(주파수/시간 마스킹)
- 시간 축 늘리기/줄이기, 피치 변환 등

### 8.4 표·시계열 데이터

- 잡음 주입(정규분포 노이즈), 부트스트랩 샘플링
- 시계열 윈도우 시프트, 랜덤 컷/붙이기
- 과도한 Dropout은 신호 자체를 해치기 쉬우므로 주의

---

## 9. 과적합 vs 과소적합 시뮬레이션 (scikit-learn 예제)

### 9.1 다항 회귀에서의 직관적 예

간단한 실험:  
1D 입력에 대해 \(\sin(x)\) 패턴을 가진 데이터를 만들고,  
다항차수(degree)를 1, 3, 9로 바꾸면서  
훈련/검증 MSE를 비교해보자.

```python
import numpy as np
import matplotlib.pyplot as plt
from sklearn.preprocessing import PolynomialFeatures
from sklearn.linear_model import LinearRegression, Ridge
from sklearn.pipeline import Pipeline
from sklearn.metrics import mean_squared_error
from sklearn.model_selection import train_test_split

# 데이터 생성
rng = np.random.RandomState(42)
n = 200
X = np.linspace(0, 2*np.pi, n)
y_true = np.sin(X)
y = y_true + rng.normal(scale=0.2, size=n)

X = X.reshape(-1, 1)
X_tr, X_va, y_tr, y_va = train_test_split(X, y, test_size=0.3, random_state=42)

def fit_and_eval(degree, alpha=0.0):
    if alpha == 0.0:
        model = Pipeline([
            ("poly", PolynomialFeatures(degree=degree)),
            ("lr", LinearRegression())
        ])
    else:
        model = Pipeline([
            ("poly", PolynomialFeatures(degree=degree)),
            ("lr", Ridge(alpha=alpha))
        ])
    model.fit(X_tr, y_tr)
    y_tr_hat = model.predict(X_tr)
    y_va_hat = model.predict(X_va)
    return (
        mean_squared_error(y_tr, y_tr_hat),
        mean_squared_error(y_va, y_va_hat),
        model
    )

for degree in [1, 3, 9]:
    tr_mse, va_mse, _ = fit_and_eval(degree)
    print(f"degree={degree}, train MSE={tr_mse:.3f}, val MSE={va_mse:.3f}")
```

- degree=1: 선형 → **과소적합** (훈련/검증 모두 MSE 큼)
- degree=9: 복잡한 다항 → 훈련 MSE는 매우 낮지만, 검증 MSE는 커질 수 있음  
  → **과적합**
- degree=3: 적절한 수준의 유연성 → 훈련과 검증 MSE가 모두 적당히 낮음

이제 `degree=9`일 때 Ridge 정규화를 넣어보자.

```python
for alpha in [0.0, 0.1, 1.0, 10.0]:
    tr_mse, va_mse, _ = fit_and_eval(degree=9, alpha=alpha)
    print(f"alpha={alpha:.1f}, train MSE={tr_mse:.3f}, val MSE={va_mse:.3f}")
```

- alpha=0: 강한 과적합
- alpha가 커질수록 훈련 MSE는 약간 증가하지만,  
  검증 MSE가 줄어드는 지점이 존재한다.

이 간단한 실험은 **“정규화가 고차원 모델의 분산을 줄여 일반화를 향상시킨다”**는 것을 눈으로 확인하게 해준다.

---

## 10. PyTorch 예제: Dropout과 MixUp/Label Smoothing 활용

### 10.1 MLP 분류 모델 + 정규화 세트

```python
import torch, torch.nn as nn, torch.optim as optim
from torch.utils.data import DataLoader, TensorDataset
import numpy as np

# 가짜 데이터: 2클래스 분류 (실제로는 북미/유럽 고객 이탈/잔존 여부 등을 떠올릴 수 있다.)
X = torch.randn(6000, 100)
y = torch.randint(0, 2, (6000,))

ds_tr = TensorDataset(X[:5000], y[:5000])
ds_va = TensorDataset(X[5000:], y[5000:])
dl_tr = DataLoader(ds_tr, batch_size=256, shuffle=True)
dl_va = DataLoader(ds_va, batch_size=512)

class MLP(nn.Module):
    def __init__(self, in_dim, hid=256, p=0.3):
        super().__init__()
        self.net = nn.Sequential(
            nn.Linear(in_dim, hid),
            nn.ReLU(),
            nn.Dropout(p),
            nn.Linear(hid, hid),
            nn.ReLU(),
            nn.Dropout(p),
            nn.Linear(hid, 2),
        )
    def forward(self, x): return self.net(x)

def mixup(x, y, alpha=0.4):
    if alpha <= 0:
        return x, nn.functional.one_hot(y, num_classes=2).float()
    lam = np.random.beta(alpha, alpha)
    idx = torch.randperm(x.size(0))
    x2, y2 = x[idx], y[idx]
    y1 = nn.functional.one_hot(y, num_classes=2).float()
    y2 = nn.functional.one_hot(y2, num_classes=2).float()
    return lam*x + (1-lam)*x2, lam*y1 + (1-lam)*y2

model = MLP(100, p=0.3)
opt = optim.AdamW(model.parameters(), lr=1e-3, weight_decay=1e-3)
sched = optim.lr_scheduler.CosineAnnealingLR(opt, T_max=30)
crit = nn.CrossEntropyLoss(label_smoothing=0.1)  # Label Smoothing

best, patience, wait = 1e9, 10, 0
history = {"train_loss": [], "val_loss": []}

for epoch in range(100):
    model.train()
    tr_loss = 0.0
    for xb, yb in dl_tr:
        xb_mix, yb_mix = mixup(xb, yb, alpha=0.2)
        logits = model(xb_mix)
        # soft target을 사용하는 Cross-Entropy
        loss = - (yb_mix * nn.functional.log_softmax(logits, dim=-1)).sum(dim=-1).mean()
        opt.zero_grad()
        loss.backward()
        nn.utils.clip_grad_norm_(model.parameters(), 5.0)
        opt.step()
        tr_loss += loss.item()*xb.size(0)
    sched.step()
    tr_loss /= len(ds_tr)

    # validation
    model.eval()
    with torch.no_grad():
        vloss, n = 0.0, 0
        for xb, yb in dl_va:
            logits = model(xb)
            vloss += crit(logits, yb).item()*xb.size(0)
            n += xb.size(0)
        vloss /= n

    history["train_loss"].append(tr_loss)
    history["val_loss"].append(vloss)

    if vloss < best - 1e-4:
        best, wait = vloss, 0
        best_state = {k: v.cpu().clone() for k, v in model.state_dict().items()}
    else:
        wait += 1
        if wait >= patience:
            break  # Early Stopping

model.load_state_dict(best_state)
model.eval()
```

위 코드에는 다음 정규화 요소가 모두 들어 있다.

- **Weight Decay** (`AdamW`)
- **Dropout** (은닉층)
- **MixUp** (입력/레이블 혼합)
- **Label Smoothing**
- **Cosine LR 스케줄**
- **Early Stopping**

실제 데이터에 적용하면, 정규화가 거의 없는 모델 대비  
검증 성능과 일반화가 훨씬 안정적인 것을 확인할 수 있다.

---

## 11. MC Dropout으로 불확실성 추정

Dropout을 추론 시에도 켜고 여러 번 예측한 뒤  
평균과 분산을 계산하면 **예측 불확실성**을 대략적으로 얻을 수 있다.

### 11.1 PyTorch 코드 스니펫

```python
import torch, numpy as np

def mc_predict(model, x, T=20):
    model.train()  # Dropout 활성화
    probs = []
    with torch.no_grad():
        for _ in range(T):
            p = torch.softmax(model(x), dim=-1)
            probs.append(p.unsqueeze(0))
    probs = torch.cat(probs, 0)  # [T, N, C]
    return probs.mean(0), probs.var(0).mean(-1)  # mean prob, predictive variance

def expected_calibration_error(probs, y, n_bins=15):
    conf, pred = probs.max(-1)
    bins = torch.linspace(0, 1, n_bins+1)
    ece = 0.0
    for i in range(n_bins):
        m = (conf >= bins[i]) & (conf < bins[i+1])
        if m.any():
            acc = (pred[m] == y[m]).float().mean()
            ece += (m.float().mean() * (conf[m].mean() - acc).abs()).item()
    return ece

# 사용 예시:
# p_mean, p_var = mc_predict(model, X_te, T=30)
# ece = expected_calibration_error(p_mean, y_te)
```

- `p_var`가 큰 샘플은 **모델이 자신 없다고 느끼는 샘플**이다.
- ECE(Expected Calibration Error)를 낮추는 것도  
  정규화/Label Smoothing 튜닝 때 중요한 목표가 될 수 있다.

---

## 12. Keras 예제: Dropout + BatchNorm + Label Smoothing

```python
import tensorflow as tf
from tensorflow.keras import layers as L, regularizers as R, callbacks as C

inputs = L.Input(shape=(100,))
x = L.Dense(256, activation=None, kernel_regularizer=R.l2(1e-4))(inputs)
x = L.BatchNormalization()(x)
x = L.ReLU()(x)
x = L.Dropout(0.3)(x)

x = L.Dense(256, activation=None, kernel_regularizer=R.l2(1e-4))(x)
x = L.BatchNormalization()(x)
x = L.ReLU()(x)
x = L.Dropout(0.3)(x)

outputs = L.Dense(10, activation="softmax")(x)

model = tf.keras.Model(inputs, outputs)

model.compile(
    optimizer=tf.keras.optimizers.AdamW(1e-3, weight_decay=1e-3),
    loss=tf.keras.losses.CategoricalCrossentropy(label_smoothing=0.1),
    metrics=["accuracy"]
)

cb = [
    C.EarlyStopping(monitor="val_loss", patience=10, restore_best_weights=True),
    C.ReduceLROnPlateau(monitor="val_loss", factor=0.5, patience=3),
]

# model.fit(X_tr, y_tr_onehot,
#           validation_data=(X_val, y_val_onehot),
#           epochs=200, batch_size=256, callbacks=cb)
```

정규화 요소:

- L2(커널 정규화) + Weight Decay(AdamW)
- BatchNorm
- Dropout
- Label Smoothing
- EarlyStopping + ReduceLROnPlateau

---

## 13. 과적합/과소적합 진단용 시각화 코드

```python
import matplotlib.pyplot as plt

def plot_curves(history):
    plt.figure(figsize=(6,4))
    plt.plot(history["train_loss"], label="train")
    plt.plot(history["val_loss"], label="val")
    plt.xlabel("epoch"); plt.ylabel("loss")
    plt.legend()
    plt.title("Learning Curves")
    plt.show()
```

- 검증 손실이 어느 시점 이후 올라가면: **과적합 시작**
- 훈련 손실도 높은데 검증 손실도 높으면: **과소적합 또는 학습 세팅 문제**

---

## 14. 현대적 이슈 몇 가지: Double Descent, Implicit Bias

### 14.1 Double Descent

최근 이론/실험에서는 모델 용량을 늘릴수록  
오차가 다시 줄어드는 **Double Descent** 현상이 관찰된다.

- 전통적 관점: 파라미터 수가 최적점 이후 증가하면 일반화 오차가 계속 증가
- 현대 딥러닝: “Interpolation Regime”(훈련오차 0 근처)을 지나 더 크게 키우면  
  다시 일반화 오차가 감소하기도 한다.

이 현상은 **정규화, 옵티마, 데이터 규모**에 민감하며,  
정규화 전략을 설계할 때 “너무 단순한 모델은 오히려 성능이 나쁠 수 있다”는  
경고로 받아들일 수 있다.

### 14.2 SGD의 암묵적 정규화(Implicit Regularization)

미니배치 SGD의 특성과  
신경망 구조의 비선형성 때문에

- 명시적인 L2/L1가 없더라도
- SGD가 자연스럽게 **노름이 작은 해** 또는  
  **평탄한 최소(Sharpness가 작은 영역)**를 선호하는 경향이 있다는 연구들이 있다.

이 때문에 **학습률/배치 크기/스케줄** 자체도  
일종의 정규화 하이퍼파라미터로 볼 수 있다.

---

## 15. 작업 유형별 정규화 세트 추천

| 작업 | 기본 세트 | 필요 시 추가 |
|---|---|---|
| Vision 분류 | AdamW(1e-3, wd 1e-4~1e-3), 강한 Aug(RandAug/MixUp/CutMix), Cosine, EarlyStopping 여유 | DropBlock 0.1~0.3, Label Smoothing 0.1, EMA |
| NLP 텍스트 | AdamW(wd 작게), Dropout 0.1, Warmup+Cosine 스케줄 | Label Smoothing 0.05~0.2, LayerDrop/DropPath, 데이터 노이즈 필터링 |
| Tabular | L2, 소량 Dropout(0.1~0.2), 적당한 용량의 MLP | Elastic Net, 강한 트리계열 모델과 앙상블, MixUp(연속형 타깃) |
| 시계열 | 입력 잡음/윈도우 증강, L2, 적당 Dropout | TCN/Transformer에 Stochastic Depth, Adversarial Regularization |

---

## 16. 실무형 튜닝 체크리스트

### 16.1 과적합(훈련↑·검증↓)일 때

1. **Weight Decay 증가** (예: 1e-4 → 5e-4 → 1e-3)
2. **Dropout 도입/상향** (0.0 → 0.1 → 0.3)
3. **데이터 증강 강화**
   - CV: RandAugment, MixUp, CutMix
   - NLP: EDA, back-translation, noise
4. **Label Smoothing 추가** (예: 0.1)
5. **Early Stopping** 기준 설정  
   (검증 손실이 몇 에포크 연속 개선 안 되면 정지)
6. **모델 단순화**  
   (레이어 수/채널 수 감소, 헤드 수 줄이기)
7. **데이터 품질 점검**  
   (극단적 노이즈/레이블 오류 제거)

### 16.2 과소적합(훈련부터 낮음)일 때

1. **모델 용량 증가** (깊이/폭, 더 표현력 있는 아키텍처)
2. **정규화 완화**
   - Dropout 비율 감소
   - WD 감소
3. **학습률/스케줄 수정**
   - 너무 작은 학습률을 키우거나
   - 초기 학습률 warmup + 충분히 긴 T_max
4. **훈련을 더 오래**  
   (단, 철저히 학습/검증 곡선을 보며)
5. **전이학습/사전학습 모델 도입**  
   (특히 데이터가 적을 때)

### 16.3 공통

- [ ] 학습/검증 곡선 시각화
- [ ] 정규화 강도(λ)/Dropout 비율/증강 강도 교차검증
- [ ] 재현성 보장(랜덤 시드, 데이터 스냅샷, 코드 버전)
- [ ] 배포 전 다양한 입력 분포(Shift/Noise)를 넣어 **강건성 테스트**

---

## 17. 요약

- **과적합 vs 과소적합**
  - 과적합: 훈련↑·검증↓, 분산↑, 바이어스↓
  - 과소적합: 훈련부터 낮음, 바이어스↑, 분산↓
  - 바이어스–분산 트레이드오프 관점에서 이해하면 전체 그림이 잡힌다.
- **정규화(Regularization)**
  - 명시적: L2(Largest 사용), L1, Elastic Net, Group/Structured 정규화
  - 암묵적: Early Stopping, Data Augmentation, Label Smoothing, Noise Injection
  - 딥러닝에서는 **AdamW(Weight Decay) + Dropout + 증강 + 스케줄**이 기본 조합이다.
- **Dropout**
  - 학습 시 뉴런/채널을 무작위 비활성화
  - Inverted Dropout으로 기대값 유지
  - 공적응 억제 + 암묵적 앙상블 효과
  - 변형: DropConnect, SpatialDropout, DropBlock, Variational Dropout, MC Dropout
- **실전**
  - 항상 **학습/검증 곡선**을 먼저 본다.
  - 과적합이면 정규화/증강/EarlyStopping을 강화하고,  
    과소적합이면 모델/학습을 키우고 정규화를 완화한다.
  - Weight Decay, Dropout, Data Augmentation, Label Smoothing, Early Stopping,  
    Optimizer/스케줄을 **하나의 패키지**로 보고 튜닝해야 한다.
- **장기적인 관점**
  - 정규화는 단순히 과적합 방지 장치가 아니라  
    **“어떤 함수 클래스를 선호할 것인가”를 설계하는 도구**이다.  
  - 프로젝트마다 데이터 특성·모델 구조·운영 환경이 다르므로,  
    이 글의 레시피를 기본으로 두고 **자신의 문제에 맞는 정규화 패턴**을 쌓아 가는 것이 중요하다.