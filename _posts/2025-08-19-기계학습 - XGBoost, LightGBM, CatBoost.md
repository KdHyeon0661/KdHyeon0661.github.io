---
layout: post
title: 기계학습 - XGBoost, LightGBM, CatBoost
date: 2025-08-19 21:25:23 +0900
category: 기계학습
---
# 🌳 XGBoost / LightGBM / CatBoost 비교 및 상세 정리

부스팅 계열 알고리즘(Gradient Boosting)의 발전된 형태로, **XGBoost**, **LightGBM**, **CatBoost**는 오늘날 머신러닝 실무에서 가장 널리 쓰이는 라이브러리입니다.  
세 가지 모두 **Gradient Boosting Decision Tree(GBDT)** 기반이지만, 구현 방식과 최적화 기법, 데이터 처리 방식에서 차이가 있습니다.

---

## 1. XGBoost (Extreme Gradient Boosting)

### 개요
- **2016년 발표**, 가장 먼저 대중화된 부스팅 알고리즘
- Kaggle과 같은 머신러닝 대회에서 "국민 알고리즘"으로 불릴 정도로 많이 사용됨
- "정확성"과 "안정성"에 강점

### 주요 특징
1. **정규화(Regularization) 추가**
   - 전통적인 GBDT에 L1, L2 규제를 적용하여 과적합 억제
2. **두 번째 미분 사용**
   - Gradient(1차 미분)뿐만 아니라 Hessian(2차 미분)을 이용 → 더 정교한 손실 최적화
3. **Shrinkage (Learning Rate)**
   - 각 트리의 기여도를 줄여 점진적 최적화
4. **Column Subsampling**
   - 랜덤 포레스트처럼 특징을 부분적으로 사용하여 과적합 방지
5. **병렬 처리 지원**
   - 다중 스레드 기반 학습 속도 개선

### 목적 함수 (수학적 형태)
\[
Obj(\theta) = \sum_{i=1}^n l(y_i, \hat{y}_i) + \sum_{k=1}^K \Omega(f_k)
\]

- \(l(y_i, \hat{y}_i)\): 손실 함수 (예: MSE, 로지스틱 손실)  
- \(\Omega(f_k) = \gamma T + \frac{1}{2}\lambda \|w\|^2\): 트리 복잡도 규제 항  

---

## 2. LightGBM (Light Gradient Boosting Machine)

### 개요
- **마이크로소프트에서 개발**
- "속도와 메모리 효율"에 초점을 맞춘 부스팅 프레임워크
- 대규모 데이터셋에서 특히 강력

### 주요 특징
1. **Histogram 기반 학습**
   - 연속형 특징값을 구간(bin)으로 나눠 저장
   - 메모리 사용량 대폭 감소, 연산 속도 개선
2. **GOSS (Gradient-based One-Side Sampling)**
   - 그래디언트가 큰 데이터(어려운 샘플)는 유지하고, 작은 그래디언트를 가진 데이터 일부는 샘플링
   - 연산량 감소, 속도 향상
3. **EFB (Exclusive Feature Bundling)**
   - 서로 상호배타적인 희소(sparse) 피처를 묶어 차원 축소
4. **Leaf-wise Tree Growth**
   - XGBoost는 Level-wise 확장(균형적인 성장)  
   - LightGBM은 Leaf-wise 확장(손실 감소가 큰 리프를 우선 분할) → 더 빠르게 손실 감소  
   - 단, 과적합 위험 ↑

---

## 3. CatBoost (Categorical Boosting)

### 개요
- **Yandex(러시아 검색엔진 회사)에서 개발**
- 범주형 변수(categorical feature) 처리를 자동으로 지원하는 점이 큰 강점
- 데이터 전처리 부담을 크게 줄여줌

### 주요 특징
1. **범주형 변수 자동 처리**
   - One-hot 인코딩이나 Label Encoding 불필요
   - 순차적 통계(Smoothed Target Encoding) 사용
2. **Ordered Boosting**
   - 전통적 부스팅은 target leakage(데이터 누수) 문제가 발생할 수 있음  
   - CatBoost는 데이터를 순서대로 사용하여 누수 방지
3. **대칭 트리(Symmetric Tree) 구조**
   - 각 깊이에서 동일한 분할 규칙 적용 → 예측 시 빠름, GPU 최적화 용이
4. **GPU 학습 강력 지원**
   - 범주형 변수 처리 + GPU = 대규모 데이터에서도 효율적

---

## 4. 비교 요약

| 항목 | XGBoost | LightGBM | CatBoost |
|------|----------|-----------|-----------|
| 개발사 | 오픈소스 (커뮤니티 주도) | Microsoft | Yandex |
| 주요 장점 | 정확성, 안정성, 튜닝 다양 | 빠른 학습, 대규모 데이터 최적화 | 범주형 변수 자동 처리, 데이터 전처리 단순 |
| 학습 속도 | 중간 | 매우 빠름 | 빠름 (특히 카테고리 데이터에서) |
| 메모리 효율 | 중간 | 높음 (Histogram, GOSS, EFB) | 중간 |
| 트리 성장 방식 | Level-wise | Leaf-wise | Symmetric |
| 범주형 변수 처리 | 수동 인코딩 필요 | 수동 인코딩 필요 | 자동 지원 |
| 과적합 위험 | 중간 | 높음 (Leaf-wise) | 낮음 (Ordered Boosting) |

---

## 5. 파이썬 구현 예제

### XGBoost
```python
import xgboost as xgb
from sklearn.datasets import load_breast_cancer
from sklearn.model_selection import train_test_split
from sklearn.metrics import accuracy_score

X, y = load_breast_cancer(return_X_y=True)
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)

model = xgb.XGBClassifier(n_estimators=200, learning_rate=0.1, max_depth=4)
model.fit(X_train, y_train)
print("XGBoost Accuracy:", accuracy_score(y_test, model.predict(X_test)))
```

### LightGBM
```python
import lightgbm as lgb

train_data = lgb.Dataset(X_train, label=y_train)
test_data = lgb.Dataset(X_test, label=y_test, reference=train_data)

params = {
    'objective': 'binary',
    'metric': 'accuracy',
    'boosting_type': 'gbdt',
    'learning_rate': 0.1,
    'num_leaves': 31
}

lgb_model = lgb.train(params, train_data, valid_sets=[test_data], num_boost_round=200, early_stopping_rounds=20)
y_pred = (lgb_model.predict(X_test) > 0.5).astype(int)
print("LightGBM Accuracy:", accuracy_score(y_test, y_pred))
```

### CatBoost
```python
from catboost import CatBoostClassifier

cat_model = CatBoostClassifier(
    iterations=200,
    learning_rate=0.1,
    depth=6,
    verbose=0
)

cat_model.fit(X_train, y_train)
print("CatBoost Accuracy:", accuracy_score(y_test, cat_model.predict(X_test)))
```

---

## 6. 선택 가이드

- **XGBoost**  
  → 기본기 탄탄, 다양한 커스터마이징 필요할 때  
- **LightGBM**  
  → 데이터가 크고, 학습 속도가 중요할 때  
- **CatBoost**  
  → 범주형 데이터가 많고, 전처리를 최소화하고 싶을 때  

---

## 📌 요약
- XGBoost: 정확하고 안정적, 표준적인 선택  
- LightGBM: 빠르고 메모리 효율적, 대규모 데이터 적합  
- CatBoost: 범주형 변수에 최적, 자동화된 전처리  

→ 결국 **데이터 특성과 문제 성격에 따라 선택**하는 것이 가장 현명합니다.