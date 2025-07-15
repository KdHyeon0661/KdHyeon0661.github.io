---
layout: post
title: Data Structure - 이진 트리
date: 2024-12-09 19:20:23 +0900
category: Data Structure
---
# 🌲 AVL 트리 - 자가 균형 이진 탐색 트리 (Self-Balancing BST)

**AVL 트리**는 1962년에 G. M. Adelson-Velsky와 E. M. Landis가 고안한 트리입니다.  
기본적인 **이진 탐색 트리(BST)**의 삽입/삭제 연산 후에도 **항상 균형을 유지**하도록 만들어진 트리입니다.

---

## ⚠️ 왜 AVL 트리가 필요한가?

### ❗ BST의 문제점

일반적인 **이진 탐색 트리(BST)**는 다음 규칙만 만족합니다:

> 왼쪽 자식 < 부모 < 오른쪽 자식

하지만 **삽입 순서에 따라 트리가 심하게 한쪽으로 기울 수 있습니다**:

```text
삽입 순서: 10 → 20 → 30 → 40 → 50
BST 구조:

    10
      \
      20
        \
        30
          \
          40
            \
            50
```

- 이 구조는 **사실상 연결 리스트**와 같고,  
- 탐색/삽입/삭제의 시간 복잡도는 O(**n**)으로 퇴화합니다.

---

## 💡 자가 균형 트리란?

> 삽입/삭제 이후에도 **자동으로 균형을 맞추는 이진 탐색 트리**

**AVL 트리**는 그 대표적인 예입니다.  
트리의 **모든 노드**에 대해 왼쪽과 오른쪽 서브트리 높이 차이가 ±1을 넘지 않도록 유지합니다.

| 트리 종류 | 균형 조건 유지 방식 | 예시 |
|-----------|--------------------|------|
| 일반 BST | 없음               | 불균형 가능 |
| AVL 트리 | 높이 차이 ≤ 1     | 회전으로 균형 유지 |
| Red-Black 트리 | 색상 규칙 기반 | 로그 높이 보장 |

---

## 📌 1. AVL 트리란?

> AVL 트리는 **모든 노드의 왼쪽 서브트리와 오른쪽 서브트리 높이 차이(균형 계수)**가 **-1, 0, +1**을 유지하는 이진 탐색 트리입니다.

### 🔸 균형 계수 (Balance Factor)

```
balance = height(left subtree) - height(right subtree)
```

- 균형 계수 ∈ { -1, 0, 1 } → 정상
- 이 범위를 벗어나면 **회전(rotation)**을 통해 균형을 되찾습니다.

---

## 🔁 2. 회전(Rotation)의 종류

| 유형 | 상황 | 처리 방법 |
|------|------|-----------|
| LL (Left-Left) | 왼쪽 자식의 왼쪽에 삽입됨 | **오른쪽 회전(Right Rotation)** |
| RR (Right-Right) | 오른쪽 자식의 오른쪽에 삽입됨 | **왼쪽 회전(Left Rotation)** |
| LR (Left-Right) | 왼쪽 자식의 오른쪽에 삽입됨 | **왼쪽-오른쪽 이중 회전 (Left-Right)** |
| RL (Right-Left) | 오른쪽 자식의 왼쪽에 삽입됨 | **오른쪽-왼쪽 이중 회전 (Right-Left)** |

---

## 🧱 3. 노드 구조와 높이 계산 (C++)

```cpp
#include <iostream>
#include <algorithm>
using namespace std;

struct Node {
    int data;
    Node* left;
    Node* right;
    int height;

    Node(int val) : data(val), left(nullptr), right(nullptr), height(1) {}
};

int getHeight(Node* node) {
    return node ? node->height : 0;
}

int getBalance(Node* node) {
    return node ? getHeight(node->left) - getHeight(node->right) : 0;
}

void updateHeight(Node* node) {
    if (node)
        node->height = 1 + max(getHeight(node->left), getHeight(node->right));
}
```

---

## 🔄 4. 회전 함수들

```cpp
Node* rightRotate(Node* y) {
    Node* x = y->left;
    Node* T2 = x->right;

    x->right = y;
    y->left = T2;

    updateHeight(y);
    updateHeight(x);

    return x;
}

Node* leftRotate(Node* x) {
    Node* y = x->right;
    Node* T2 = y->left;

    y->left = x;
    x->right = T2;

    updateHeight(x);
    updateHeight(y);

    return y;
}
```

---

## ➕ 5. 삽입 연산 with 재균형

```cpp
Node* insert(Node* node, int key) {
    if (!node) return new Node(key);

    if (key < node->data)
        node->left = insert(node->left, key);
    else if (key > node->data)
        node->right = insert(node->right, key);
    else
        return node; // 중복 허용 안함

    updateHeight(node);
    int balance = getBalance(node);

    // LL Case
    if (balance > 1 && key < node->left->data)
        return rightRotate(node);

    // RR Case
    if (balance < -1 && key > node->right->data)
        return leftRotate(node);

    // LR Case
    if (balance > 1 && key > node->left->data) {
        node->left = leftRotate(node->left);
        return rightRotate(node);
    }

    // RL Case
    if (balance < -1 && key < node->right->data) {
        node->right = rightRotate(node->right);
        return leftRotate(node);
    }

    return node;
}
```

## 6. AVL 트리의 삭제 연산

AVL 트리에서 삭제 연산은 다음 두 가지 과정을 포함합니다:

1. **일반 BST 삭제 방식**으로 노드를 제거
2. **균형이 깨진 노드에 대해 회전**을 수행하여 자가 균형 유지

---

### BST 삭제 요약

삭제할 노드 `x`를 찾은 후 다음 중 하나로 처리합니다:

| 상황 | 처리 방식 |
|------|-----------|
| 자식 없음 (리프) | 그냥 제거 |
| 자식 1개 | 자식을 부모와 연결 |
| 자식 2개 | 오른쪽 서브트리의 최솟값(또는 왼쪽 서브트리의 최댓값)으로 교체 후 삭제 |

---

### 삭제 시 필요한 회전 처리

삭제로 인해 서브트리의 높이 차이가 ±1을 넘는 경우, 다음 회전이 필요할 수 있습니다:

| Case | 조건 | 처리 |
|------|------|------|
| **LL** | 왼쪽이 2 이상 크고, 왼쪽 자식의 왼쪽이 더 큼 | **오른쪽 회전** |
| **LR** | 왼쪽이 2 이상 크고, 왼쪽 자식의 오른쪽이 더 큼 | **왼쪽 후 오른쪽 회전** |
| **RR** | 오른쪽이 2 이상 크고, 오른쪽 자식의 오른쪽이 더 큼 | **왼쪽 회전** |
| **RL** | 오른쪽이 2 이상 크고, 오른쪽 자식의 왼쪽이 더 큼 | **오른쪽 후 왼쪽 회전** |

---

### AVL 트리 삭제 함수

```cpp
Node* minValueNode(Node* node) {
    Node* current = node;
    while (current && current->left)
        current = current->left;
    return current;
}

Node* deleteNode(Node* root, int key) {
    if (!root) return nullptr;

    // BST 삭제 로직
    if (key < root->data)
        root->left = deleteNode(root->left, key);
    else if (key > root->data)
        root->right = deleteNode(root->right, key);
    else {
        // Case 1/2/3: 삭제 대상 노드 찾음
        if (!root->left || !root->right) {
            Node* temp = root->left ? root->left : root->right;
            delete root;
            return temp;
        }
        // Case 4: 두 자식이 있는 경우
        Node* temp = minValueNode(root->right);  // 또는 maxValueNode(root->left)
        root->data = temp->data;
        root->right = deleteNode(root->right, temp->data);
    }

    // 높이 갱신 및 균형 확인
    updateHeight(root);
    int balance = getBalance(root);

    // LL Case
    if (balance > 1 && getBalance(root->left) >= 0)
        return rightRotate(root);

    // LR Case
    if (balance > 1 && getBalance(root->left) < 0) {
        root->left = leftRotate(root->left);
        return rightRotate(root);
    }

    // RR Case
    if (balance < -1 && getBalance(root->right) <= 0)
        return leftRotate(root);

    // RL Case
    if (balance < -1 && getBalance(root->right) > 0) {
        root->right = rightRotate(root->right);
        return leftRotate(root);
    }

    return root;
}
```

---

## 📤 6. 중위 순회 (Inorder Traversal)

```cpp
void inorder(Node* root) {
    if (!root) return;
    inorder(root->left);
    cout << root->data << " ";
    inorder(root->right);
}
```

---


## 🧪 4. 전체 사용 예시

```cpp
int main() {
    Node* root = nullptr;
    int values[] = { 30, 20, 40, 10, 25, 35, 50 };
    for (int val : values)
        root = insert(root, val);

    cout << "Inorder before deletion: ";
    inorder(root);
    cout << endl;

    root = deleteNode(root, 20);

    cout << "Inorder after deletion: ";
    inorder(root);
    cout << endl;

    return 0;
}
```

### 🔽 출력 예시
```
Inorder before deletion: 10 20 25 30 35 40 50  
Inorder after deletion: 10 25 30 35 40 50
```
---

## ✅ 8. AVL 트리의 시간 복잡도

| 연산 | 시간 복잡도 |
|------|-------------|
| 탐색 | O(log n) |
| 삽입 | O(log n) |
| 삭제 | O(log n) |

AVL 트리는 항상 **O(log n)** 높이를 유지하기 때문에,  
불균형한 BST보다 훨씬 안정적입니다.

---

## 🧠 마무리 요약

| 키워드 | 설명 |
|--------|------|
| BST의 단점 | 삽입 순서에 따라 선형 구조로 퇴화 가능 |
| AVL 트리 | 자가 균형 이진 탐색 트리 |
| 균형 조건 | 각 노드의 왼쪽·오른쪽 서브트리 높이 차 ≤ 1 |
| 회전 종류 | LL, RR, LR, RL |
| 삭제 원리 | 일반 BST 삭제 방식 + 균형 유지 |
| 회전 종류 | LL, RR, LR, RL |
| 장점 | 탐색, 삽입, 삭제 모두 O(log n) |
| 단점 | 삽입/삭제가 일반 BST보다 약간 느릴 수 있음 (회전 필요) |
| 주의 | 회전 후 `height` 갱신 필수 |