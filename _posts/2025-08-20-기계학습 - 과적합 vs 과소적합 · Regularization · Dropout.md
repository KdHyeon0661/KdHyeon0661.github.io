---
layout: post
title: 기계학습 - 과적합 vs 과소적합 · Regularization · Dropout
date: 2025-08-20 17:25:23 +0900
category: 기계학습
---
# 과적합 vs 과소적합 · Regularization · Dropout

## 1. 과적합(Overfitting) vs 과소적합(Underfitting)

### 1.1 개념 요약
- **과적합**: 학습 데이터의 **노이즈/우연 패턴**까지 외워 **훈련 성능↑·검증/테스트 성능↓**.
- **과소적합**: 모델 용량이 부족하거나 학습이 불충분해 **훈련 성능부터 낮음**.

### 1.2 학습/검증 곡선 징후
| 상황 | 훈련 손실 | 검증 손실 | 특징 |
|---|---|---|---|
| 과적합 | 지속 하락 | 어느 시점 이후 **상승** | 모델 복잡/데이터 적음/정규화 약함 |
| 과소적합 | **높게 유지** | 높게 유지 | 모델 단순/학습률 부적절/학습 부족 |
| 적정 | 낮음 | 낮음(또는 평행) | 일반화 양호 |

### 1.3 바이어스–분산 트레이드오프(회귀 MSE의 분해)
입력 \(x\), 정답 \(y=f^\*(x)+\epsilon\) (잡음 \(\epsilon\) 평균 0, 분산 \(\sigma^2\))에서,
모형 \(\hat f\)의 기대 MSE는
$$
\mathbb{E}\big[(y-\hat f(x))^2\big]
=\underbrace{\big(\mathbb{E}[\hat f(x)]-f^\*(x)\big)^2}_{\text{Bias}^2}
+\underbrace{\mathbb{Var}[\hat f(x)]}_{\text{Variance}}
+\underbrace{\sigma^2}_{\text{Irreducible Noise}}.
$$
- **용량↑** → **분산↑·바이어스↓**(과적합 위험)  
- **정규화↑** → **바이어스↑·분산↓**(과소적합 위험)

### 1.4 원인↔처방 매핑
- 과적합: **모델 복잡**·데이터 적음·노이즈 큼·정규화 약함  
  → **정규화 강화, 데이터 증가/증강, 드롭아웃, 조기 종료, 모델 단순화**
- 과소적합: **모델 용량↓**·학습 불충분·표현력 부족  
  → **모델 확대, 더 오래/더 잘 학습, 특성 공학, 정규화 완화**

---

## 2. Regularization(정규화) — 수학·직관·레시피

### 2.1 목적 함수(일반형)
$$
\min_{\theta}\ \frac{1}{n}\sum_{i=1}^n L\!\big(y_i,\hat{y}(x_i;\theta)\big)\ +\ \lambda\,\Omega(\theta),
$$
- \(\lambda>0\): 정규화 강도. **크면 규제↑(바이어스↑·분산↓)**

### 2.2 L2 (Ridge, Weight Decay)
$$
\Omega(\theta)=\tfrac{1}{2}\|\theta\|_2^2,\quad 
w \leftarrow (1-\alpha\lambda)\,w\ -\ \alpha\nabla_w L.
$$
- 큰 가중치 억제 → **매끄러운 함수** 선호, 과적합 완화  
- **베이지안 해석**: \(w\sim \mathcal N(0,\tau^2 I)\) 사전과 동치

### 2.3 L1 (Lasso) — 희소성
$$
\Omega(\theta)=\|w\|_1=\sum_j |w_j|
$$
- **가중치 0 유도**(특성 선택 효과)  
- 좌표 강하/서브그래디언트 최적화 필요

### 2.4 Elastic Net
$$
\Omega(\theta)=\alpha\|w\|_1 + (1-\alpha)\tfrac{1}{2}\|w\|_2^2.
$$
- L1의 선택성 + L2의 안정성 **절충**

### 2.5 기타 구조적/암묵적 정규화
- **Early Stopping**: 검증 성능 악화 직전 중단(가끔 **암묵적 L2**와 유사 효과)
- **Data Augment**: 입력 다양화로 **가상 데이터** 확대 (이미지·텍스트·오디오·표)
- **Label Smoothing**: 원-핫 대신 \((1-\varepsilon,\varepsilon/(K-1))\) → 과도 확신 억제
- **BatchNorm/LayerNorm**: 통계 안정화 + 약한 정규화 효과
- **Noise Injection**: 입력/은닉/가중치에 노이즈 주입(티호노프 정규화와 연결)
- **Adversarial/Sharpness-Aware**: 경계 근방 강건화(고급)

### 2.6 Optimizer와 Weight Decay
- **Adam의 L2**는 실제로 적응 스케일과 섞여 **진짜 Weight Decay와 다름**  
  → **AdamW**(decoupled weight decay) 사용 권장  
- 튜닝 초깃값: `weight_decay ≈ 1e-4 ~ 5e-2` (모델·데이터에 의존)  
- 학습률 스케줄과 함께 **cosine/step**로 조합

---

## 3. Dropout — 원리·수학·변형·주의점

### 3.1 정의(학습 시 뉴런 무작위 비활성)
은닉 \(h\), 마스크 \(m\sim \mathrm{Bernoulli}(1-p)\):
$$
\tilde h=\frac{m\odot h}{1-p}.
$$
- 분모 \(1-p\)로 **기대값 보정**(Inverted Dropout)  
- 서로 다른 서브모델의 **암묵적 앙상블** 효과

### 3.2 왜 잘 작동하나?
- 특정 경로에 **과도 의존(co-adaptation)** 억제 → **강건 특징** 학습
- 모델 평균화와 유사한 일반화 이점 (테스트는 풀 네트워크 사용)

### 3.3 어디·얼마나?
- **MLP/FC**: 0.3–0.5 자주  
- **Conv**: **SpatialDropout/DropBlock** 0.1–0.3 (채널/블록 단위)  
- **Transformer**: `dropout, attention_dropout ≈ 0.1`, **Stochastic Depth(DropPath)** 적용(깊은 네트)

> 과도한 비율은 **학습 불안정/과소적합**. 작은 값에서 점진 튜닝.

### 3.4 변형/활용
- **DropConnect**: 가중치 자체를 드롭  
- **SpatialDropout/DropBlock**: Conv에 적합  
- **Variational Dropout**: RNN 시퀀스 고정 마스크  
- **MC Dropout**: 추론 시에도 드롭아웃 활성 → **불확실성 추정**(평균·분산)

### 3.5 BN과의 순서
일반적 권장: **Conv/Linear → Norm → 활성화 → Dropout**  
(특정 프레임워크/구현에 따라 예외 가능)

---

## 4. 데이터 증강(Data Augmentation) 레시피

### 4.1 Computer Vision
- 기본: Random Crop/Flip/ColorJitter, Resize, Cutout  
- 강력: **MixUp**  
  \( \tilde x=\lambda x_i+(1-\lambda)x_j,\ \tilde y=\lambda y_i+(1-\lambda)y_j,\ \lambda\sim\mathrm{Beta}(\alpha,\alpha) \)  
- **CutMix**: 패치 섞기 → 라벨도 면적 비율로 혼합

### 4.2 NLP
- 동의어 치환, 랜덤 삭제/스와프, 백트랜슬레이션, **EDA**  
- 사전학습 임베딩/언어모델 기반 마스킹

### 4.3 오디오/음성
- **SpecAugment**(주파수/시간 마스킹), 시간 왜곡

### 4.4 표/시계열
- 잡음 주입, 부트스트랩, 랜덤 임계/구간 왜곡, 시계열 윈도우 시프트

---

## 5. Regularization 튜닝 절차(실무형 Checklists)

### 5.1 과적합(훈련↑·검증↓)일 때
1) **Weight Decay↑** (AdamW)  
2) **Dropout 도입/상향** (소량→점진)  
3) **Augment 강화**(CV: RandAugment/MixUp/CutMix)  
4) **Early Stopping** + **LR 스케줄**(Cosine/ReduceLROnPlateau)  
5) **모델 단순화**(깊이/너비↓, 파라 공유, Distillation)  
6) **데이터 증가**(수집/합성)

### 5.2 과소적합(훈련도 낮음)일 때
1) **모델 용량↑**(층/너비)·학습 더 오래  
2) **정규화/Dropout 약화**  
3) **특징 공학/전이학습**  
4) **학습률/옵티마** 재설계(warmup, momentum, batch size)

### 5.3 선택 지침(작업별)
- **Tabular(불균형/잡음多)**: L2 + 소량 Dropout/Label Smoothing, 트리계열 고려  
- **CV**: Weight Decay + 강한 Aug + (필요 시)Dropout/DropBlock + EMA  
- **NLP**: Label Smoothing(0.1±), LayerDrop/DropPath, 작은 WD, 강한 스케줄링  
- **시계열**: 입력 잡음/윈도우 증강, 과도한 Dropout 금지

---

## 6. 코드 스니펫 모음

### 6.1 Scikit-learn — L1/L2/ElasticNet와 정규화 경로
```python
import numpy as np
from sklearn.linear_model import LogisticRegressionCV
from sklearn.preprocessing import StandardScaler
from sklearn.pipeline import Pipeline
from sklearn.datasets import make_classification
from sklearn.metrics import classification_report

X, y = make_classification(n_samples=4000, n_features=50, n_informative=10,
                           weights=[0.7,0.3], random_state=42)

# L1 (saga)와 L2를 교차검증으로 비교
pipelines = {
    "L2": Pipeline([
        ("scaler", StandardScaler()),
        ("clf", LogisticRegressionCV(
            Cs=20, cv=5, penalty="l2", solver="lbfgs", max_iter=5000,
            scoring="f1", n_jobs=-1))
    ]),
    "L1": Pipeline([
        ("scaler", StandardScaler()),
        ("clf", LogisticRegressionCV(
            Cs=20, cv=5, penalty="l1", solver="saga", max_iter=5000,
            scoring="f1", n_jobs=-1))
    ])
}

for name, pipe in pipelines.items():
    pipe.fit(X, y)
    print(f"\n== {name} ==")
    yhat = pipe.predict(X)
    print(classification_report(y, yhat, digits=3))
```

### 6.2 PyTorch — AdamW(Decoupled WD) + Dropout + EarlyStopping + MixUp/LabelSmoothing
```python
import torch, torch.nn as nn, torch.optim as optim
from torch.utils.data import DataLoader, TensorDataset
import numpy as np

# Dummy data
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
            nn.Linear(in_dim, hid), nn.ReLU(),
            nn.Dropout(p),
            nn.Linear(hid, hid), nn.ReLU(),
            nn.Dropout(p),
            nn.Linear(hid, 2)
        )
    def forward(self, x): return self.net(x)

def mixup(x, y, alpha=0.4):
    if alpha <= 0: return x, y.float()
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
for epoch in range(100):
    model.train()
    for xb, yb in dl_tr:
        xb_mix, yb_mix = mixup(xb, yb, alpha=0.2)
        logits = model(xb_mix)
        loss = - (yb_mix * nn.functional.log_softmax(logits, dim=-1)).sum(dim=-1).mean()  # CE with soft targets
        opt.zero_grad()
        loss.backward()
        nn.utils.clip_grad_norm_(model.parameters(), 5.0)
        opt.step()
    sched.step()

    # validation
    model.eval()
    with torch.no_grad():
        vloss, n = 0.0, 0
        for xb, yb in dl_va:
            logits = model(xb)
            vloss += crit(logits, yb).item()*xb.size(0)
            n += xb.size(0)
        vloss /= n
    if vloss < best - 1e-4:
        best, wait = vloss, 0
        best_state = {k:v.cpu().clone() for k,v in model.state_dict().items()}
    else:
        wait += 1
        if wait >= patience:
            break  # Early Stopping

model.load_state_dict(best_state)
model.eval()
```

### 6.3 PyTorch — MC Dropout로 **불확실성** 추정(ECE 평가 예시 포함)
```python
import torch, numpy as np

def mc_predict(model, x, T=20):
    model.train()  # Dropout 활성화
    probs = []
    with torch.no_grad():
        for _ in range(T):
            p = torch.softmax(model(x), dim=-1)
            probs.append(p.unsqueeze(0))
    return torch.cat(probs, 0).mean(0), torch.cat(probs, 0).var(0).mean(-1)

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

# usage:
# p_mean, p_var = mc_predict(model, X_te, T=30)
# ece = expected_calibration_error(p_mean, y_te)
```

### 6.4 TensorFlow/Keras — Label Smoothing + Dropout/BatchNorm + ReduceLROnPlateau
```python
import tensorflow as tf
from tensorflow.keras import layers as L, regularizers as R, callbacks as C

inputs = L.Input(shape=(100,))
x = L.Dense(256, activation=None, kernel_regularizer=R.l2(1e-4))(inputs)
x = L.BatchNormalization()(x); x = L.ReLU()(x); x = L.Dropout(0.3)(x)
x = L.Dense(256, activation=None, kernel_regularizer=R.l2(1e-4))(x)
x = L.BatchNormalization()(x); x = L.ReLU()(x); x = L.Dropout(0.3)(x)
outputs = L.Dense(10, activation="softmax")(x)

model = tf.keras.Model(inputs, outputs)
model.compile(optimizer=tf.keras.optimizers.AdamW(1e-3, weight_decay=1e-3),
              loss=tf.keras.losses.CategoricalCrossentropy(label_smoothing=0.1),
              metrics=["accuracy"])
cb = [
  C.EarlyStopping(monitor="val_loss", patience=10, restore_best_weights=True),
  C.ReduceLROnPlateau(monitor="val_loss", factor=0.5, patience=3)
]
# model.fit(X_tr, y_tr_onehot, validation_data=(X_val, y_val_onehot), epochs=200, batch_size=256, callbacks=cb)
```

---

## 7. 추가: 정규화와 일반화에 관한 심화 개념

### 7.1 마진과 일반화(선형 분류)
로지스틱/힌지 손실에서 **큰 마진**을 선호하는 해는 일반화 경향이 강함.  
뉴럴넷에서도 **대규모 배치/스케줄**이 **implicit margin**에 영향.

### 7.2 라벨 스무딩과 칼리브레이션
- **Label Smoothing**은 예측 확률의 **과도 확신**을 완화 → **ECE 감소** 경향  
- 다만 지나치면 **분류 경계** 약화(미세 구분 저하)

### 7.3 Double Descent(현대 통계)
용량을 늘릴수록 오류가 다시 증가 후 감소하는 **이중 하강** 현상 가능 →  
**강한 정규화·증강·스케줄링**이 필요

### 7.4 일반화 관점의 잡음 주입
입력 가우시안 노이즈는 특정 선형 모형에서 **티호노프(L2) 정규화**와 연결

---

## 8. 시각화/진단 예시 코드

### 8.1 학습·검증 곡선 그리기
```python
import matplotlib.pyplot as plt

def plot_curves(history):
    plt.figure(figsize=(6,4))
    plt.plot(history["train_loss"], label="train")
    plt.plot(history["val_loss"], label="val")
    plt.xlabel("epoch"); plt.ylabel("loss"); plt.legend(); plt.title("Learning Curves")
    plt.show()
```

### 8.2 정규화 경로(λ에 따른 계수 크기) — 개념 스케치
- L1: \(\lambda\uparrow\) → **계수 0**로 수축(희소)  
- L2: \(\lambda\uparrow\) → **연속적 수축**, 0은 드뭄

---

## 9. 작업 유형별 **정규화 세트** 추천

| 작업 | 기본 세트 | 필요 시 추가 |
|---|---|---|
| CV 분류 | AdamW(1e-3, wd 1e-4~1e-3), Aug(RandAug/MixUp), Cosine, Early Stop | DropBlock 0.1~0.3, Label Smoothing 0.1 |
| NLP 텍스트 | AdamW(wd 작게), Dropout 0.1, Warmup+Cosine | Label Smoothing 0.05~0.2, LayerDrop/DropPath |
| 표(Tabular) | L2, 소량 Dropout(0.1~0.2), 다소 작은 네트 | 강한 L1/ElasticNet(특성 선택), MixUp(연속형) |
| 시계열 | 잡음 주입·윈도우 시프트, L2, 적당 Dropout | TCN/Transformer에 Stochastic Depth |

---

## 10. 종합 체크리스트

- [ ] 학습/검증 곡선 모니터링(과적합/과소적합 판별)  
- [ ] Weight Decay/Dropout/증강/스케줄 조합 설정  
- [ ] Label Smoothing/칼리브레이션(ECE) 확인  
- [ ] Early Stopping 기준/버퍼(plateau 마진) 정의  
- [ ] 정규화 강도/드롭아웃 비율/증강 강도 **교차검증**  
- [ ] 재현성(시드/버전/데이터 스냅샷)·로깅  
- [ ] 배포 전 **견고성(Shift/Noise/Adversarial) 스팟 테스트**

---

## 11. 요약

- **과적합 vs 과소적합**은 **바이어스–분산** 균형 문제.  
- **Regularization**은 손실에 **패널티**(L1/L2/ElasticNet)와 **암묵적 기법**(Early Stop, Aug, Label Smoothing, Noise)을 통해 **복잡도 제어**.  
- **Dropout**은 뉴런을 확률적으로 끊어 **공적응 방지·앙상블 효과**로 일반화 향상(훈련 시만 적용, inverted scaling).  
- 실무에서는 **AdamW + 적절한 증강 + Early Stopping + (필요 시) Dropout/Label Smoothing**을 기본 세트로, 데이터/모델 특성에 따라 강도를 튜닝하는 전략이 가장 효율적입니다.