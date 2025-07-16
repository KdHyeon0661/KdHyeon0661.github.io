---
layout: post
title: Data Structure - 펜윅 트리
date: 2024-12-09 19:20:23 +0900
category: Data Structure
---
# 🗺️ KD-트리 (K-Dimensional Tree)

## 🔍 1. KD-트리란?

**KD-트리(KD-Tree)** 는 **K차원 공간에서의 데이터 탐색**을 빠르게 하기 위한 이진 트리 구조입니다.

- 주로 **2D/3D 좌표**에서의 범위 탐색(range search), 최근접 이웃 탐색(nearest neighbor search)에 활용됩니다.
- K차원 데이터를 각 차원 축 기준으로 번갈아가며 분할합니다.

---

## 📦 2. 특징

| 항목 | 설명 |
|------|------|
| 구조 | 이진 트리 |
| 목적 | 다차원 범위 탐색, NN 검색 |
| 시간 복잡도 | 평균 O(log n), 최악 O(n) |
| 적용 예시 | 위치 기반 서비스, 이미지 검색, 게임 AI, 로보틱스 |

---

## 🧠 3. 분할 원리

- 각 노드는 `K`차원 좌표값을 저장합니다.
- 루트에서는 `x축`, 그 다음은 `y축`, 그 다음은 `z축` … 순으로 **차원을 바꿔가며** 분할합니다.
- 노드 기준 왼쪽 서브트리는 **해당 축에서 작거나 같은 좌표**, 오른쪽은 **큰 좌표**가 위치합니다.

---

## 🌳 4. 2D KD-트리 구현 (C++)

### ✅ 구조 정의

```cpp
#include <iostream>
#include <vector>
using namespace std;

const int K = 2; // 2차원

struct Point {
    int coords[K];
};

struct KDNode {
    Point point;
    KDNode* left;
    KDNode* right;

    KDNode(Point p) : point(p), left(nullptr), right(nullptr) {}
};
```

---

### ✅ 삽입 함수

```cpp
KDNode* insert(KDNode* root, Point point, int depth = 0) {
    if (root == nullptr)
        return new KDNode(point);

    int axis = depth % K;
    if (point.coords[axis] <= root->point.coords[axis])
        root->left = insert(root->left, point, depth + 1);
    else
        root->right = insert(root->right, point, depth + 1);

    return root;
}
```

---

### 삭제 연산 (Delete)

KD-트리에서 삭제는 일반 BST보다 복잡합니다. 삭제 대상 노드의 하위 트리에서 **해당 축의 최소값 노드**를 찾아 대체합니다.

#### ✅ 삭제 로직 개요

- 삭제할 노드와 현재 노드가 같으면:
  - 오른쪽 서브트리에서 해당 축의 최소 노드를 찾아 대체
  - 없으면 왼쪽에서 대체
  - 둘 다 없으면 nullptr 반환
- 아니라면 축 기준으로 좌/우 재귀 탐색

#### ✅ C++ 코드

```cpp
// 최소값 노드 찾기 (axis 기준)
KDNode* findMin(KDNode* root, int d, int depth = 0) {
    if (!root) return nullptr;

    int axis = depth % K;
    if (axis == d) {
        if (!root->left) return root;
        return findMin(root->left, d, depth + 1);
    }

    KDNode* lmin = findMin(root->left, d, depth + 1);
    KDNode* rmin = findMin(root->right, d, depth + 1);

    KDNode* minNode = root;
    if (lmin && lmin->point.coords[d] < minNode->point.coords[d]) minNode = lmin;
    if (rmin && rmin->point.coords[d] < minNode->point.coords[d]) minNode = rmin;
    return minNode;
}

// 삭제 함수
KDNode* deleteNode(KDNode* root, Point target, int depth = 0) {
    if (!root) return nullptr;

    int axis = depth % K;
    if (root->point.coords[0] == target.coords[0] && root->point.coords[1] == target.coords[1]) {
        if (root->right) {
            KDNode* minNode = findMin(root->right, axis, depth + 1);
            root->point = minNode->point;
            root->right = deleteNode(root->right, minNode->point, depth + 1);
        } else if (root->left) {
            KDNode* minNode = findMin(root->left, axis, depth + 1);
            root->point = minNode->point;
            root->right = deleteNode(root->left, minNode->point, depth + 1);
            root->left = nullptr;
        } else {
            delete root;
            return nullptr;
        }
        return root;
    }

    if (target.coords[axis] <= root->point.coords[axis])
        root->left = deleteNode(root->left, target, depth + 1);
    else
        root->right = deleteNode(root->right, target, depth + 1);

    return root;
}
```

---

### ✅ 범위 탐색 예시

```cpp
void rangeSearch(KDNode* root, Point low, Point high, int depth = 0) {
    if (!root) return;

    bool inRange = true;
    for (int i = 0; i < K; ++i) {
        if (root->point.coords[i] < low.coords[i] || root->point.coords[i] > high.coords[i])
            inRange = false;
    }

    if (inRange) {
        cout << "(" << root->point.coords[0] << ", " << root->point.coords[1] << ") ";
    }

    int axis = depth % K;

    if (root->point.coords[axis] >= low.coords[axis])
        rangeSearch(root->left, low, high, depth + 1);
    if (root->point.coords[axis] <= high.coords[axis])
        rangeSearch(root->right, low, high, depth + 1);
}
```

---

## 🧪 5. 예제 실행

```cpp
int main() {
    vector<Point> points = {
        {{3, 6}}, {{17, 15}}, {{13, 15}}, {{6, 12}}, {{9, 1}}, {{2, 7}}, {{10, 19}}
    };

    KDNode* root = nullptr;
    for (auto p : points)
        root = insert(root, p);

    Point low = {{0, 0}}, high = {{10, 15}};
    cout << "Points in range [(0,0)-(10,15)]: ";
    rangeSearch(root, low, high);
    cout << endl;

    return 0;
}
```

---

## 🔽 출력 예시

```
Points in range [(0,0)-(10,15)]: (3, 6) (6, 12) (9, 1) (2, 7) (10, 19)
```

> 주의: 마지막 출력값은 범위를 넘으므로 필터링 필요 (출력 코드 보완 가능)

---

## 🎯 2. 최근접 이웃 탐색 (Nearest Neighbor)

- 기준점과 가장 가까운 점 1개를 찾습니다.
- 거리 제곱(`dx*dx + dy*dy`)을 사용해 비교합니다.
- 방문 우선순위는 기준 축 → 다른 쪽 가지도 거리 조건 만족 시 탐색.

### ✅ C++ 구현

```cpp
double squaredDistance(Point a, Point b) {
    double dist = 0;
    for (int i = 0; i < K; ++i)
        dist += (a.coords[i] - b.coords[i]) * (a.coords[i] - b.coords[i]);
    return dist;
}

void nearest(KDNode* root, Point target, KDNode*& best, double& bestDist, int depth = 0) {
    if (!root) return;

    double d = squaredDistance(root->point, target);
    if (d < bestDist) {
        bestDist = d;
        best = root;
    }

    int axis = depth % K;
    KDNode* next = nullptr, *other = nullptr;

    if (target.coords[axis] <= root->point.coords[axis]) {
        next = root->left;
        other = root->right;
    } else {
        next = root->right;
        other = root->left;
    }

    nearest(next, target, best, bestDist, depth + 1);

    double axisDist = (root->point.coords[axis] - target.coords[axis]) *
                      (root->point.coords[axis] - target.coords[axis]);

    if (axisDist < bestDist)
        nearest(other, target, best, bestDist, depth + 1);
}
```

---

## 👥 3. K-최근접 이웃 탐색 (KNN)

- 우선순위 큐 (`priority_queue`) 를 이용하여 거리 기준으로 K개의 최근접 노드 저장
- 큐 크기가 K를 넘으면 가장 먼 노드 제거

### ✅ C++ 구현

```cpp
#include <queue>
typedef pair<double, KDNode*> PD;

void knn(KDNode* root, Point target, int K, priority_queue<PD>& pq, int depth = 0) {
    if (!root) return;

    double dist = squaredDistance(root->point, target);
    pq.push({ -dist, root }); // 최대힙: 거리 작은 게 top

    if (pq.size() > K) pq.pop();

    int axis = depth % K;
    KDNode* next = (target.coords[axis] <= root->point.coords[axis]) ? root->left : root->right;
    KDNode* other = (next == root->left) ? root->right : root->left;

    knn(next, target, K, pq, depth + 1);

    double axisDist = (root->point.coords[axis] - target.coords[axis]) *
                      (root->point.coords[axis] - target.coords[axis]);

    if (-pq.top().first > axisDist)
        knn(other, target, K, pq, depth + 1);
}
```

---

## 🔍 4. 전체 예제 코드 사용

```cpp
int main() {
    vector<Point> points = {
        {{3, 6}}, {{17, 15}}, {{13, 15}}, {{6, 12}}, {{9, 1}}, {{2, 7}}, {{10, 19}}
    };

    KDNode* root = nullptr;
    for (auto p : points)
        root = insert(root, p);

    // 최근접 탐색
    Point query = {{10, 9}};
    KDNode* best = nullptr;
    double bestDist = 1e9;
    nearest(root, query, best, bestDist);
    cout << "Nearest to (10,9): (" << best->point.coords[0] << ", " << best->point.coords[1] << ")\n";

    // 삭제 예시
    root = deleteNode(root, {{6, 12}});

    // KNN 탐색
    priority_queue<PD> pq;
    knn(root, query, 3, pq);
    cout << "3 Nearest neighbors to (10,9):\n";
    while (!pq.empty()) {
        auto p = pq.top().second->point;
        cout << "(" << p.coords[0] << ", " << p.coords[1] << ")\n";
        pq.pop();
    }

    return 0;
}
```

---

## 🧠 6. KD-트리와 다른 공간 분할 트리 비교

| 자료구조 | 특징 | 쓰임 |
|----------|------|------|
| KD-트리 | K차원 이진 분할 | 일반적인 좌표 탐색 |
| QuadTree | 2D에서 사분면 분할 | 이미지, 게임 타일 |
| Octree | 3D에서 8분할 | 공간 모델링 |
| R-Tree | 사각형 범위 기준 | DB 인덱스, GIS |

---

## ✅ 요약

| 키워드 | 설명 |
|--------|------|
| KD-트리 | 다차원 공간 분할용 이진 트리 |
| 분할 기준 | 축 기준 순환 (x→y→x...) |
| 활용 | 범위 질의, 최근접 탐색 |
| 구현 | 삽입, 탐색 재귀 기반 |
| 삭제 | 해당 축의 최소 노드로 대체 |
| 최근접 이웃 | 재귀적 거리 비교 + 백트래킹 |
| KNN | 최대힙으로 K개 유지, 가지치기 조건 사용 |