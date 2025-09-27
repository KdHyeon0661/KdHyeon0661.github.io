---
layout: post
title: 기계학습 - 과적합 vs 과소적합 · Regularization · Dropout
date: 2025-08-20 17:25:23 +0900
category: 기계학습
---
# 과적합 vs 과소적합 · Regularization · Dropout 개념 정리

## 1) 과적합(Overfitting) vs 과소적합(Underfitting)

### 개념
- **과적합**: 학습 데이터의 **노이즈/우연한 패턴**까지 외워서, **훈련 성능↑·검증/테스트 성능↓**인 상태.
- **과소적합**: 모델이 너무 단순하거나 학습이 부족해 **훈련 성능조차 낮은** 상태.

### 전형적 징후 (학습/검증 곡선)
| 상황 | 훈련 손실 | 검증 손실 | 특징 |
|---|---|---|---|
| 과적합 | 지속 하락 | 어느 시점 이후 **상승** | 모델 복잡/학습 오래/데이터 적음 |
| 과소적합 | **높게 유지** | 높게 유지 | 모델 단순/학습률 부적절/학습 부족 |
| 적정 | 낮음 | 낮음(또는 평행) | 일반화 양호 |

### 원인과 처방 요약
- 과적합 원인: 모델 복잡도↑, 데이터 적음/노이즈↑, 정규화 약함  
  → **정규화 강화, 데이터 증가/증강, 드롭아웃, 조기 종료, 모델 단순화**
- 과소적합 원인: 모델 용량↓, 학습 불충분, 특징 부족  
  → **모델 확장, 더 오래 학습, 특성 공학/표현력 강화**

---

## 2) Regularization(정규화) — 수학과 직관

### 2.1 목적 함수
일반적으로 정규화는 **복잡도 패널티**를 더해 과적합을 억제합니다.
\[
\min_{\theta}\;\; \underbrace{\frac{1}{n}\sum_{i=1}^n L\big(y_i,\hat{y}(x_i;\theta)\big)}_{\text{데이터 손실}}
\;+\;
\underbrace{\lambda\;\Omega(\theta)}_{\text{정규화(규제) 항}}
\]
- \(\lambda>0\): 정규화 강도. **크면 규제↑(바이어스↑·분산↓)**

### 2.2 L2 (Ridge, Weight Decay)
\[
\Omega(\theta)=\frac{1}{2}\|\theta\|_2^2
\]
- **큰 가중치에 불이익** → 매끄러운 함수 선호, 과적합 억제.
- 가중치 업데이트(학습률 \(\alpha\))에 **weight decay**가 포함:
\[
w \leftarrow (1-\alpha\lambda)\,w \;-\; \alpha\;\nabla_w L
\]
- **베이지안 관점**: 가우시안(평균 0) 사전분포를 가정한 MLE와 동등.

### 2.3 L1 (Lasso)
\[
\Omega(\theta)=\|\theta\|_1=\sum_j |w_j|
\]
- **희소성(sparsity)** 유도 → 불필요한 가중치가 **0**이 되기도 함(특성 선택 효과).
- 라쏘 회귀·희소 모델에서 유리. (비분화 지점 → 서브그래디언트/좌표 강하)

### 2.4 Elastic Net
\[
\Omega(\theta)=\alpha\|\theta\|_1 + (1-\alpha)\tfrac{1}{2}\|\theta\|_2^2
\]
- L1의 선택성과 L2의 안정성을 **절충**.

### 2.5 기타 “암묵적/구조적” 정규화
- **Early Stopping**: 검증 성능이 악화되기 직전에 학습 중단 → 손쉬운 강력 규제.
- **Data Augmentation**: 입력 다양화로 **데이터 공간을 부풀려** 일반화↑ (이미지 플립/크롭, 텍스트 노이즈, MixUp/CutMix 등).
- **Label Smoothing**: 라벨을 0/1 대신 \((1-\varepsilon, \varepsilon/(K-1))\)로 부드럽게 → 과도한 확신 억제.
- **BatchNorm**: 통계 안정화 + 약한 정규화 효과.
- **Dropout**: (다음 절 참조) 뉴런 무작위 제거로 앙상블 비슷한 효과.

> 실무 팁: **\(\lambda\)**, 드롭아웃 비율, 조기 종료 에포크는 **교차검증**으로 선택하세요.

---

## 3) Dropout — 개념, 수학, 실무 포인트

### 3.1 무엇인가?
- 학습 시 각 층의 활성값(뉴런)을 **확률 \(p\)** 로 무작위 **비활성화(0)** 하여 **공적응(co-adaptation)** 을 줄이고 **앙상블 효과**를 내는 기법.
- 테스트 시에는 **전체 네트워크**를 사용.

### 3.2 수학적 형태 (Inverted Dropout)
은닉 벡터 \(h\)와 드롭아웃 마스크 \(m \sim \text{Bernoulli}(1-p)\):
\[
\tilde{h} = \frac{m \odot h}{1-p}
\]
- 분모 \(1-p\) 로 나눠 **기대 활성값 보정**(훈련/추론 시 스케일 일치).  
- 결과적으로 학습 동안 **\(2^{\#\text{뉴런}}\)** 개에 달하는 얕은 앙상블을 암묵적으로 평균하는 효과.

### 3.3 왜 효과적인가?
- 일부 뉴런을 랜덤하게 끊어 **특정 경로 의존**을 약화 → **강건한 특징** 학습.
- **모델 평균화**와 유사한 일반화 이점.

### 3.4 어디에, 얼마나?
- **완전연결(FC)**: 0.3–0.5 자주 사용(클래식).  
- **합성곱 층**: 공간적 상관성 고려해 **SpatialDropout/DropBlock**(특정 채널/블록 단위 드롭) 0.1–0.3.  
- **Transformer**: attention/ffn/dropout 0.1 전후가 흔함.
- 너무 크게 주면 **학습 불안정/과소적합** 유발 → 점진 튜닝.

### 3.5 변형
- **DropConnect**: 가중치 자체를 무작위로 drop.  
- **SpatialDropout**: 채널 전체를 드롭(Conv에 적합).  
- **MC Dropout**: 추론 시에도 드롭아웃을 켜서 **불확실성 추정**(샘플 반복 평균·분산).

### 3.6 한계/주의
- BN과 동시 사용 시 **순서/강도**가 중요(일반적으로 `Conv → BN → ReLU → Dropout`).  
- 거대한 데이터/강한 증강에서는 드롭아웃 이득이 작을 수 있음(대신 **Weight Decay + Aug** 조합이 더 효과적일 때도 많음).

---

## 4) 실무 처방전 (체크리스트)

### 과적합일 때 (훈련↑·검증↓)
1) **정규화 강화**: L2(Weight Decay)↑, L1/ElasticNet 고려  
2) **Dropout 도입/증가**  
3) **Data Augmentation** 강화 (이미지: RandAugment/MixUp)  
4) **Early Stopping** + **러닝레이트 스케줄**  
5) **모델 단순화**(층/너비↓), 파라미터 공유/티칭(KD)  
6) **더 많은 데이터**(수집 또는 합성)

### 과소적합일 때 (훈련도 낮음)
1) **모델 용량↑**(층/너비↑), **학습 더 오래**  
2) **정규화/Dropout 약화**  
3) **특징 공학/사전학습(Pretraining)**  
4) **학습률/옵티마 조정**(warmup·cosine 등)

---

## 5) 간단 코드 예시

### (A) PyTorch — L2(Weight Decay) + Dropout + Early Stopping
```python
import torch, torch.nn as nn, torch.optim as optim

class MLP(nn.Module):
    def __init__(self, in_dim, hid=256, p=0.5):
        super().__init__()
        self.net = nn.Sequential(
            nn.Linear(in_dim, hid), nn.ReLU(),
            nn.Dropout(p),                 # Dropout
            nn.Linear(hid, hid), nn.ReLU(),
            nn.Dropout(p),
            nn.Linear(hid, 1)             # binary logit
        )
    def forward(self, x): return self.net(x)

model = MLP(in_dim=100, p=0.3)
# L2 정규화는 optimizer의 weight_decay로 지정
opt = optim.AdamW(model.parameters(), lr=1e-3, weight_decay=1e-4)
crit = nn.BCEWithLogitsLoss()

best_val, patience, wait = 1e9, 10, 0
for epoch in range(200):
    model.train()
    # ... forward/backward on train batches ...
    # val loss 계산
    val_loss = ...  # evaluate on validation set
    if val_loss < best_val - 1e-4:
        best_val, wait = val_loss, 0
        best_state = {k:v.cpu().clone() for k,v in model.state_dict().items()}
    else:
        wait += 1
        if wait >= patience: break  # Early Stopping

model.load_state_dict(best_state)
model.eval()  # 추론 시 Dropout 비활성화
```

### (B) Keras — Dropout/BatchNorm와 규제
```python
import tensorflow as tf
from tensorflow.keras import layers as L, regularizers as R

inputs = L.Input(shape=(100,))
x = L.Dense(256, activation="relu",
            kernel_regularizer=R.l2(1e-4))(inputs)  # L2
x = L.Dropout(0.3)(x)
x = L.Dense(256, activation="relu")(x)
x = L.BatchNormalization()(x)
x = L.Dropout(0.3)(x)
outputs = L.Dense(1, activation="sigmoid")(x)

model = tf.keras.Model(inputs, outputs)
model.compile(optimizer=tf.keras.optimizers.Adam(1e-3),
              loss="binary_crossentropy",
              metrics=["AUC", "Precision", "Recall"])

es = tf.keras.callbacks.EarlyStopping(monitor="val_auc", mode="max",
                                      patience=10, restore_best_weights=True)
model.fit(X_tr, y_tr, validation_data=(X_val, y_val),
          epochs=200, batch_size=256, callbacks=[es])
```

---

## 6) 요약
- **과적합 vs 과소적합**은 **바이어스–분산 트레이드오프**의 양극단.  
- **Regularization**은 손실에 패널티를 더해 **복잡도 제어**(L2/Weight Decay, L1, Elastic Net, Early Stopping, Augmentation, Label Smoothing 등).  
- **Dropout**은 뉴런을 확률적으로 끊어 **공적응 방지·앙상블 효과**로 일반화 향상(훈련 시만 적용, inverted scaling).  
- 실무에서는 **데이터/문제 특성**에 맞춰 **규제 세트(Weight Decay + Aug + Early Stop + (필요 시) Dropout)** 를 조합하여 가장 단순한 방법부터 점진적으로 강화하는 전략이 효과적입니다.