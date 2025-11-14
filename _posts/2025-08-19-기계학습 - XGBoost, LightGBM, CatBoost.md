---
layout: post
title: 기계학습 - XGBoost, LightGBM, CatBoost
date: 2025-08-19 21:25:23 +0900
category: 기계학습
---
# XGBoost / LightGBM / CatBoost

## 한 눈에 보는 선택 가이드 (요약)

| 상황 | 추천 | 핵심 이유 |
|---|---|---|
| **수치형 위주 + 표준 GBDT** | **XGBoost** | 2차(헤시안) 최적화 + 다양한 정규화/제약(모노토닉/인터랙션) + 안정성 |
| **대용량(행·열 수↑), 빠른 학습** | **LightGBM** | 히스토그램 학습 + GOSS/EFB + **Leaf-wise** 성장(손실 급감) |
| **범주형(카테고리) 많음/전처리 최소화** | **CatBoost** | Ordered Boosting + Target Statistics로 **누수 방지** + 대칭트리 |
| **랭킹(LTR)** | XGBoost/LightGBM/CatBoost Ranker | 세 라이브러리 모두 LambdaMART/Pairwise 지원 |
| **모노토닉 제약/해석성 강화** | XGBoost/LightGBM/CatBoost | `monotone_constraints` 지원(세 라이브러리 모두 가능) |

---

## 공통 수학: GBDT의 목적함수와 분할 이득

### 목적함수(2차 테일러 근사)

트리 \(f_t\)를 순차적으로 더하는 가법모형:
$$
\hat{y}_i^{(t)} = \hat{y}_i^{(t-1)} + f_t(x_i).
$$

손실의 2차 근사(샘플 \(i\)의 그래디언트/헤시안 \(g_i,h_i\)):
$$
\mathcal{L}^{(t)} \approx \sum_{i=1}^n \Big[ g_i f_t(x_i) + \frac12 h_i f_t(x_i)^2 \Big] + \Omega(f_t).
$$

리프 \(j\)에 속한 샘플 집합 \(I_j\), 리프 값 \(w_j\)일 때:
$$
\mathcal{L}^{(t)} \approx \sum_{j} \Big[ G_j w_j + \frac12 (H_j+\lambda) w_j^2 \Big] + \gamma T,
$$
여기서
$$
G_j=\sum_{i\in I_j} g_i,\quad H_j=\sum_{i\in I_j} h_i,\quad \Omega(f_t)=\gamma T + \tfrac12\lambda \sum_j w_j^2.
$$

리프 최적값과 리프 점수:
$$
w_j^* = -\frac{G_j}{H_j+\lambda},\qquad
\mathcal{L}^{(t)}_{\text{leaf}} = -\frac12 \sum_j \frac{G_j^2}{H_j+\lambda} + \gamma T.
$$

### 분할 이득(Gain)

왼/오 리프로 분할 시 이득:
$$
\text{Gain}=\frac12\Big(\frac{G_L^2}{H_L+\lambda} + \frac{G_R^2}{H_R+\lambda} - \frac{G^2}{H+\lambda}\Big) - \gamma.
$$

> **해석**: \(G\)가 크게 갈라지고 \(H\)가 충분할수록 이득이 크다. \(\lambda\)는 리프 L2 정규화, \(\gamma\)는 분할 최소 이득(threshold) 역할.

---

## XGBoost — 디테일 & 실전

### 핵심 특징

- **정규화**: \(\lambda\)(L2, `reg_lambda`), \(\alpha\)(L1, `reg_alpha`), \(\gamma\)(분할 최소 이득, `min_split_loss`).
- **2차 최적화**: 헤시안 기반 분할 품질 평가(안정적).
- **서브샘플링**: `subsample`, `colsample_bytree/by_level/by_node`.
- **트리 방법**: `tree_method` = `"hist"`, `"approx"`, `"gpu_hist"`,(대규모/고속), `"exact"`(소규모).
- **결측값**: **학습 중 자동 기본방향(default direction)** 추정(명시 처리 불필요).
- **카테고리**: 파이썬 1.6+에서 `enable_categorical=True` + pandas `category` dtype 지원(원핫 불필요).
- **제약**: `monotone_constraints`, `interaction_constraints`(해석/규정 준수).
- **부스터 변형**: `"dart"`(dropout boosting; 과적합 억제), `"gbtree"`(기본).

### 자주 쓰는 파라미터 가이드

| 파라미터 | 역할/권장 범위 |
|---|---|
| `n_estimators` | 200~1500(early stopping과 함께) |
| `learning_rate` | 0.02~0.2(작을수록 트리↑, 일반화↑ 경향) |
| `max_depth` | 3~10(깊을수록 패턴↑, 과적합 위험) |
| `min_child_weight` | 리프의 H 합 최소(값↑ → 보수적) |
| `subsample` / `colsample_bytree` | 0.6~1.0(과적합 억제) |
| `reg_alpha`, `reg_lambda` | 0~(데이터에 따라 튜닝) |
| `gamma` | 0~(분할 최소 이득 요구치) |
| `tree_method` | `"hist"`/`"gpu_hist"` 권장(대규모) |

### 분류 예제 (GPU/모노토닉/조기중단)

```python
import xgboost as xgb
from sklearn.datasets import make_classification
from sklearn.model_selection import train_test_split
from sklearn.metrics import roc_auc_score

X, y = make_classification(n_samples=120000, n_features=40, n_informative=12, random_state=42)
X_tr, X_te, y_tr, y_te = train_test_split(X, y, test_size=0.2, stratify=y, random_state=42)

clf = xgb.XGBClassifier(
    n_estimators=1200,
    learning_rate=0.05,
    max_depth=6,
    subsample=0.8,
    colsample_bytree=0.8,
    reg_lambda=1.0,
    tree_method="gpu_hist",      # GPU 없으면 "hist"
    monotone_constraints="(0,0,0,1,-1,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0)"
)
clf.fit(X_tr, y_tr, eval_set=[(X_te, y_te)], eval_metric="auc", verbose=False,
        early_stopping_rounds=100)
proba = clf.predict_proba(X_te)[:,1]
print("AUC:", roc_auc_score(y_te, proba))
```

### 랭킹 예제 (XGBRanker; LambdaMART)

```python
import numpy as np, xgboost as xgb
from sklearn.model_selection import train_test_split

# toy ranking data (group sizes)

X = np.random.randn(1000, 20)
y = np.random.randint(0, 5, size=1000)     # relevance
group = np.random.randint(5, 15, size=50)  # 50 queries, varying lengths
X_tr, X_te, y_tr, y_te = train_test_split(X, y, test_size=0.2, random_state=42)

rk = xgb.XGBRanker(
    objective="rank:ndcg",
    n_estimators=400, learning_rate=0.1, max_depth=6, tree_method="hist"
)
rk.fit(X_tr, y_tr, group=np.array([len(X_tr)//50]*50))
pred = rk.predict(X_te)  # 평가엔 별도 랭킹 metric 필요
```

---

## LightGBM — 디테일 & 실전

### 핵심 특징

- **Histogram 기반**: 연속형 값을 bin으로 압축 → 메모리↓ 속도↑, 분할 카운팅 빠름.
- **GOSS**(Gradient-based One-Side Sampling): 그래디언트 상위 \(a\%\)는 유지, 하위 중 \(b\%\)만 샘플링하고 **재가중**하여 편향 보정.
- **EFB**(Exclusive Feature Bundling): 서로 상호배타적(대부분 0)인 희소 피처를 묶어 차원↓.
- **Leaf-wise 성장**: 가장 이득 큰 리프를 계속 분할 → 손실 급감. 단, **과적합 위험↑** → `max_depth`/`min_data_in_leaf`로 제어.
- **카테고리 지원**: 카테고리를 int로 인코딩하고 `categorical_feature` 등록 or pandas `category` dtype.

### 자주 쓰는 파라미터

| 파라미터 | 역할/권장 |
|---|---|
| `num_leaves` | 모델 용량 핵심(보통 \(2^{\text{max_depth}}\) 근처) |
| `max_depth` | Leaf-wise 과적합 제어(예: 6~12) |
| `min_data_in_leaf` | 리프 최소 샘플(불균형/과적합 방지) |
| `feature_fraction` | 열 서브샘플(0.6~0.9) |
| `bagging_fraction` + `bagging_freq` | 행 서브샘플/빈도 |
| `lambda_l1`, `lambda_l2` | 정규화 |
| `min_gain_to_split` | 분할 최소 이득(γ 유사) |
| `max_bin` | 히스토그램 bin 수(255 기본, 메모리/정밀도 트레이드오프) |
| `device` | `"gpu"`로 GPU 가속 |

### 분류 (sklearn API; 조기중단/범주형)

```python
import lightgbm as lgb
from sklearn.datasets import fetch_openml
from sklearn.model_selection import train_test_split
from sklearn.metrics import roc_auc_score
import pandas as pd

# 예시: 범주형 포함된 데이터라 가정

X, y = fetch_openml("adult", version=2, as_frame=True, parser="auto", return_X_y=True)
cat_cols = X.select_dtypes(include=["category", "object"]).columns
for c in cat_cols: X[c] = X[c].astype("category")

X_tr, X_te, y_tr, y_te = train_test_split(X, y.astype(int), test_size=0.2, random_state=42, stratify=y)

clf = lgb.LGBMClassifier(
    n_estimators=2000, learning_rate=0.03,
    num_leaves=64, max_depth=-1,
    feature_fraction=0.8, bagging_fraction=0.8, bagging_freq=1,
    min_data_in_leaf=50, lambda_l2=1.0, device="cpu" # "gpu" 가능
)
clf.fit(X_tr, y_tr,
        eval_set=[(X_te, y_te)], eval_metric="auc",
        callbacks=[lgb.early_stopping(stopping_rounds=100, verbose=False)])
proba = clf.predict_proba(X_te)[:,1]
print("AUC:", roc_auc_score(y_te, proba))
```

### 랭킹 (LGBMRanker)

```python
import numpy as np, lightgbm as lgb

X = np.random.randn(3000, 30)
y = np.random.randint(0, 5, size=3000)
qids = np.repeat(np.arange(300), 10)  # 300 queries, each 10 docs

rk = lgb.LGBMRanker(
    objective="lambdarank",
    n_estimators=500, learning_rate=0.05,
    num_leaves=63, min_data_in_leaf=20
)
rk.fit(X, y, group=np.bincount(qids), eval_at=[1,3,5])
pred = rk.predict(X)
```

---

## CatBoost — 디테일 & 실전

### 핵심 특징

- **범주형 자동 처리**: 원-핫 없이 **Target Statistics**(스무딩된 타깃 평균) 사용.
  - 누수 방지를 위해 **Ordered** 방식: 데이터 순열을 잡고 현재 포인트 **이전** 관측치로만 통계를 계산.
- **Ordered Boosting**: 각 단계 잔차도 순서 기반으로 누수 방지.
- **대칭 트리(Oblivious, Symmetric)**: 깊이 \(d\)에서 동일한 분기 규칙을 양쪽에 적용 → 예측 빠름/GPU 효율↑/규모화 용이.
- **텍스트/수치/범주 혼합 지원**, 결측 자동 처리.

### Target Statistics(개략)

범주 \(c\)에 대해(순열에서 i 이전 데이터만):
$$
\text{TS}_i(c)=\frac{\sum_{j<i,\ x_j=c} y_j + a \cdot p}{\#\{j<i:\ x_j=c\}+a},
$$
여기서 \(p\)는 사전(prior), \(a\)는 스무딩 강도. 이로써 희소/저빈도 카테고리의 분산을 낮추고 누수 방지.

### 자주 쓰는 파라미터

| 파라미터 | 설명 |
|---|---|
| `iterations`/`learning_rate`/`depth` | 일반 GBDT와 동일 트레이드오프 |
| `l2_leaf_reg` | L2 정규화 |
| `loss_function` | `Logloss`, `CrossEntropy`, `RMSE`, `MAE`, `QueryRMSE` 등 |
| `eval_metric` | AUC/F1/NDCG 등 |
| `cat_features` | 범주형 열 인덱스 or 이름 |
| `task_type` | `"CPU"`/`"GPU"` |
| `od_type`/`od_wait` | Overfitting detector(조기중단) |

### 분류 예제 (범주형 자동 처리/GPU/조기중단)

```python
import pandas as pd
from sklearn.model_selection import train_test_split
from sklearn.metrics import roc_auc_score
from catboost import CatBoostClassifier, Pool

# 예시: 범주형 열 포함
# df = pd.read_csv("your_data.csv")
# y = df["target"]; X = df.drop(columns=["target"])
# cat_cols = [c for c in X.columns if X[c].dtype == "object"]

# 데모용 (인공 데이터)

X = pd.DataFrame({
    "age":[23,45,31,52,46,57,29,33]*200,
    "job":["A","B","A","C","B","C","A","B"]*200,
    "income":[31,55,43,70,61,80,40,49]*200
})
y = ([0,1,0,1,1,1,0,0]*200)

cat_cols = ["job"]
X_tr, X_te, y_tr, y_te = train_test_split(X, y, test_size=0.2, random_state=42, stratify=y)

train_pool = Pool(X_tr, y_tr, cat_features=cat_cols)
test_pool  = Pool(X_te, y_te, cat_features=cat_cols)

clf = CatBoostClassifier(
    iterations=2000, learning_rate=0.03, depth=6,
    loss_function="Logloss", eval_metric="AUC",
    l2_leaf_reg=3.0, task_type="CPU", # "GPU" 가능
    od_type="Iter", od_wait=100, verbose=0
)
clf.fit(train_pool, eval_set=test_pool, use_best_model=True)
proba = clf.predict_proba(test_pool)[:,1]
print("AUC:", roc_auc_score(y_te, proba))
```

### 랭킹 예제 (CatBoostRanker)

```python
from catboost import CatBoostRanker, Pool
import numpy as np

X = np.random.randn(2000, 15)
y = np.random.randint(0, 5, size=2000)
qid = np.repeat(np.arange(200), 10)

train_pool = Pool(X, y, group_id=qid)
rk = CatBoostRanker(
    iterations=800, learning_rate=0.06, depth=6,
    loss_function="YetiRank", verbose=100
)
rk.fit(train_pool)
pred = rk.predict(X)
```

---

## 모델 비교 요약 (심화 관점)

| 항목 | XGBoost | LightGBM | CatBoost |
|---|---|---|---|
| **트리 성장** | Level-wise(균형) | Leaf-wise(최대 이득 리프 우선) | Symmetric(Oblivious) |
| **핵심 가속** | `hist`/`gpu_hist` | Histogram + **GOSS/EFB** | 대칭트리 + GPU 효율 |
| **정규화** | L1/L2 + `gamma` | L1/L2 + `min_gain_to_split` | `l2_leaf_reg` |
| **카테고리** | 1.6+: enable_categorical 가능(또는 One-Hot) | int 카테고리 + `categorical_feature` | **자동(Ordered TS)** |
| **결측 처리** | 자동 방향 학습 | 자동 | 자동 |
| **과적합** | 중간(튜닝 풍부) | Leaf-wise로 ↑ (깊이/리프제약 필요) | Ordered Boosting 덕에 ↓ 경향 |
| **특이 기능** | 모노토닉/인터랙션 제약, DART | GOSS/EFB, 랭킹/네이티브 훈련 빠름 | Ordered Boosting, 누수 강건, 텍스트/카테고리 우수 |

---

## 하이퍼파라미터 튜닝 레시피

1) **학습률–트리개수 트레이드오프**
- `learning_rate` 낮추면 `n_estimators` ↑, 일반화 개선 경향.
- 조기중단(`early_stopping`)으로 최적 라운드 탐색.

2) **모델 용량 제어**
- XGB: `max_depth`, `min_child_weight`, `gamma`, `subsample`, `colsample_bytree`
- LGB: `num_leaves`(핵심), `max_depth`, `min_data_in_leaf`, `feature_fraction`, `bagging_*`
- Cat: `depth`, `l2_leaf_reg`, `iterations`, `learning_rate`

3) **불균형 대응**
- 클래스 가중치(XGB: `scale_pos_weight` ≈ 음성/양성 비율, LGB/Cat: `class_weight`)
- PR-AUC, F1, Recall@k 등 **업무 맞춤 지표**로 조기중단.

4) **카테고리 처리 전략**
- CatBoost 우선 고려. LightGBM은 **int 인코딩**+`categorical_feature`. XGBoost는 최신 `enable_categorical=True` 또는 원핫.

5) **모노토닉 제약(규정 준수/해석성)**
- XGB: `monotone_constraints="(1,-1,0,...)"`
- LGB: `monotone_constraints` 리스트
- Cat: `monotone_constraints` 지원(피처별 +1/0/−1)

6) **GPU**
- XGB: `tree_method="gpu_hist"`
- LGB: `device="gpu"`, `gpu_platform_id/gpu_device_id`
- Cat: `task_type="GPU"`

---

## 회귀 예제 (세 라이브러리 비교 스켈레톤)

```python
import numpy as np
from sklearn.datasets import make_regression
from sklearn.model_selection import train_test_split
from sklearn.metrics import mean_absolute_error, r2_score

X, y = make_regression(n_samples=40000, n_features=50, noise=12.0, random_state=42)
X_tr, X_te, y_tr, y_te = train_test_split(X, y, test_size=0.2, random_state=42)

# XGBoost

import xgboost as xgb
xgb_reg = xgb.XGBRegressor(n_estimators=2000, learning_rate=0.03, max_depth=8,
                           subsample=0.8, colsample_bytree=0.8, tree_method="hist")
xgb_reg.fit(X_tr, y_tr, eval_set=[(X_te, y_te)], eval_metric="rmse", verbose=False, early_stopping_rounds=100)
pred_x = xgb_reg.predict(X_te)

# LightGBM

import lightgbm as lgb
lgb_reg = lgb.LGBMRegressor(n_estimators=4000, learning_rate=0.02, num_leaves=127,
                            feature_fraction=0.8, bagging_fraction=0.8, bagging_freq=1,
                            min_data_in_leaf=50)
lgb_reg.fit(X_tr, y_tr, eval_set=[(X_te, y_te)], eval_metric="l1", callbacks=[lgb.early_stopping(100, verbose=False)])
pred_l = lgb_reg.predict(X_te)

# CatBoost

from catboost import CatBoostRegressor
cat_reg = CatBoostRegressor(iterations=5000, learning_rate=0.03, depth=8,
                            loss_function="RMSE", verbose=False, od_type="Iter", od_wait=200)
cat_reg.fit(X_tr, y_tr, eval_set=(X_te, y_te), use_best_model=True)
pred_c = cat_reg.predict(X_te)

def report(name, y_true, y_pred):
    print(f"{name:10s} | MAE={mean_absolute_error(y_true, y_pred):.3f} | R2={r2_score(y_true, y_pred):.3f}")

report("XGBoost", y_te, pred_x)
report("LightGBM", y_te, pred_l)
report("CatBoost", y_te, pred_c)
```

---

## 랭킹/다중분류/확률보정 팁

- **랭킹**: 세 라이브러리 모두 LambdaMART/Pairwise/Query* 손실 지원. 쿼리 그룹 길이 지정 필수.
- **다중분류**: XGB/Cat/LGB 모두 softmax/multiclass 지원(평가 지표: macro/micro-F1).
- **확률 보정**: 트리 부스팅은 확률 과신 경향 → `CalibratedClassifierCV`(Platt/Isotonic)로 보정 권장(운영 임계값 민감 시).

---

## 해석과 검증

- **중요도/SHAP**:
  - XGB/LGB/Cat 모두 **SHAP TreeExplainer** 호환.
  - 피처 중요도는 split/gain/cover 방식 차이 → **SHAP**으로 보완.
- **데이터 누수 점검**:
  - CatBoost 외의 단순 타깃인코딩은 누수 위험.
  - 시계열/랭킹은 **시간/쿼리 단위**로 분할.
- **조기중단/검증셋**:
  - 반드시 **업무 지표**로 early stopping.
  - 데이터 드리프트 여부 모니터링.

---

## 자주 겪는 함정과 해결책

1. **LightGBM 과적합**: `num_leaves` ↓, `min_data_in_leaf` ↑, `max_depth` 설정, `feature_fraction/bagging_fraction` < 1.0, `min_gain_to_split` ↑.
2. **CatBoost 시간↑**: `depth` ↓, `rsm`(열 서브샘플) 활용, `task_type="GPU"`, `iterations`/`learning_rate` 조절.
3. **XGBoost 느림**: `tree_method="hist"`/`"gpu_hist"`, `max_bin`↓(DMatrix), `subsample/colsample` 도입.
4. **범주형 폭발**(고유값↑): CatBoost 사용 또는 빈도 cutoff/rare category 묶기.
5. **클래스 불균형**: 가중치/샘플링 + **PR-AUC** 최적화.
6. **모노토닉 위반**: 제약 파라미터 설정 후 **부분의존(ICE)**로 검증.

---

## 핵심 수식 모음 (요약)

- **Leaf 값**:
  $$
  w_j^*=-\frac{G_j}{H_j+\lambda}
  $$
- **Split Gain**:
  $$
  \text{Gain}=\frac12\Big(\frac{G_L^2}{H_L+\lambda} + \frac{G_R^2}{H_R+\lambda} - \frac{G^2}{H+\lambda}\Big) - \gamma
  $$
- **GOSS 재가중(개념)**: 큰 \(|g|\) 샘플은 모두 유지, 작은 \(|g|\) 중 일부만 유지하고 잔여의 기여를 스케일로 보정.
- **CatBoost TS(Ordered)**:
  $$
  \text{TS}_i=\frac{\sum_{j<i:\ x_j=c} y_j + a\cdot p}{\#\{j<i:\ x_j=c\}+a}
  $$

---

## 최종 체크리스트

- [ ] 업무 지표와 **조기중단** 기준을 먼저 정했다
- [ ] 교차검증은 **그룹/시간** 단위로 분리했다
- [ ] LightGBM은 `num_leaves`와 `min_data_in_leaf`를 함께 튜닝했다
- [ ] 범주형은 CatBoost 우선, 그렇지 않다면 누수 없는 인코딩을 썼다
- [ ] 확률이 중요하면 **보정**(Platt/Isotonic)을 적용했다
- [ ] SHAP으로 중요도/정책 제약(모노토닉)을 검증했다
- [ ] 배포 전 입력 스케일/카테고리 사전 drift 감시를 설정했다

---

## 마무리

- **XGBoost**: 안정적·정교한 표준.
- **LightGBM**: 대용량·고속 최적.
- **CatBoost**: 범주형 최강 + 누수 방지.
