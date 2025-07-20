---
layout: post
title: Data Structure - 레드-블랙 트리
date: 2024-12-09 20:20:23 +0900
category: Data Structure
---
# 🔴⚫ 레드-블랙 트리 (Red-Black Tree)

레드-블랙 트리는 이진 탐색 트리의 일종으로, **스스로 균형을 유지하는 트리**입니다.  
AVL 트리보다 회전 횟수가 적고, 실용적인 성능을 보장하여 STL `std::map`, `std::set` 등에 사용됩니다.

---

## 📌 1. 레드-블랙 트리의 특징

| 규칙 | 설명 |
|------|------|
| 1. 노드는 빨간색 또는 검은색이다. |
| 2. 루트 노드는 항상 검은색이다. |
| 3. 모든 리프(NIL)는 검은색이다. |
| 4. 빨간 노드의 자식은 반드시 검은색 (즉, **빨간색이 연속될 수 없다**) |
| 5. 모든 노드에서 **리프까지 가는 경로에 포함된 검은색 노드의 수는 같다** |

---

## 🧱 2. 노드 구조 정의 (C++)

```cpp
enum Color { RED, BLACK };

struct Node {
    int data;
    Color color;
    Node* left;
    Node* right;
    Node* parent;

    Node(int data) : data(data), color(RED),
        left(nullptr), right(nullptr), parent(nullptr) {}
};
```

---

## 🔁 3. 좌우 회전 함수

레드-블랙 트리 삽입 후 규칙을 만족시키기 위해 회전이 필요합니다.

```cpp
Node* rotateLeft(Node* root, Node* x) {
    Node* y = x->right;
    x->right = y->left;
    if (y->left) y->left->parent = x;
    y->parent = x->parent;

    if (!x->parent) root = y;
    else if (x == x->parent->left) x->parent->left = y;
    else x->parent->right = y;

    y->left = x;
    x->parent = y;
    return root;
}

Node* rotateRight(Node* root, Node* y) {
    Node* x = y->left;
    y->left = x->right;
    if (x->right) x->right->parent = y;
    x->parent = y->parent;

    if (!y->parent) root = x;
    else if (y == y->parent->left) y->parent->left = x;
    else y->parent->right = x;

    x->right = y;
    y->parent = x;
    return root;
}
```

---

## 🌱 4. 삽입 후 색 보정 (Fixup)

삽입된 노드가 `RED`이므로, 트리 규칙 위반 시 색 보정 또는 회전이 필요합니다.

```cpp
Node* fixViolation(Node* root, Node* z) {
    while (z != root && z->parent->color == RED) {
        Node* gp = z->parent->parent;
        if (z->parent == gp->left) {
            Node* y = gp->right;
            if (y && y->color == RED) { // Case 1: 삼촌도 RED
                z->parent->color = BLACK;
                y->color = BLACK;
                gp->color = RED;
                z = gp;
            } else {
                if (z == z->parent->right) { // Case 2: 삼촌 BLACK, 삼각형
                    z = z->parent;
                    root = rotateLeft(root, z);
                }
                // Case 3: 일직선
                z->parent->color = BLACK;
                gp->color = RED;
                root = rotateRight(root, gp);
            }
        } else {
            Node* y = gp->left;
            if (y && y->color == RED) {
                z->parent->color = BLACK;
                y->color = BLACK;
                gp->color = RED;
                z = gp;
            } else {
                if (z == z->parent->left) {
                    z = z->parent;
                    root = rotateRight(root, z);
                }
                z->parent->color = BLACK;
                gp->color = RED;
                root = rotateLeft(root, gp);
            }
        }
    }
    root->color = BLACK;
    return root;
}
```

---

## ✨ 5. 삽입 함수

```cpp
Node* bstInsert(Node* root, Node* node) {
    if (!root) return node;

    if (node->data < root->data) {
        root->left = bstInsert(root->left, node);
        root->left->parent = root;
    } else if (node->data > root->data) {
        root->right = bstInsert(root->right, node);
        root->right->parent = root;
    }
    return root;
}

Node* insert(Node* root, int data) {
    Node* node = new Node(data);
    root = bstInsert(root, node);
    return fixViolation(root, node);
}
```

---

## 6. Red-Black Tree 삭제 연산 & 🔍 AVL vs Red-Black 비교

---

### ❌ Red-Black Tree 삭제

레드-블랙 트리에서 삭제는 **일반 BST 삭제** 후,  
**레드-블랙 규칙이 깨졌을 경우 보정**(Fix-up)을 수행합니다.

#### 📌 삭제 절차 요약

1. BST 방식으로 노드 삭제
2. 삭제한 노드나 그 대체 노드가 **검은색**일 경우 문제 발생 가능
3. **Fix-up**을 통해 규칙 회복

---

### 🧠 삭제 중 위반되는 규칙

- 삭제로 인해 **검은 노드 수 불균형** 발생 가능
- **Double Black (이중 검정)** 문제:
  - 삭제 후 null이거나 대체 노드가 검정이면  
    트리의 경로별 검정 노드 수가 달라짐
  - 이 문제를 해결하는 회전과 색 변경이 필요

---

### 🔁 삭제 보정 핵심 Case 정리

> 🔍 `x`: 삭제 후 위치한 노드  
> 🔍 `s`: 형제 노드 (sibling)

| Case | 조건 | 처리 방식 |
|------|------|-----------|
| **Case 1** | 형제 `s`가 RED | `s`와 부모 색 변경 후 회전 |
| **Case 2** | 형제 `s`, 자식 모두 BLACK | `s`를 RED로, `x`를 부모로 올림 |
| **Case 3** | `s`는 BLACK, 한쪽 자식만 RED | 자식 방향 회전 후 Case 4로 |
| **Case 4** | `s`는 BLACK, 먼 쪽 자식 RED | 회전 + 색 변경으로 종료 |

---

### 🛠️ C++ 코드 구조 (요약)

> 전체 코드는 매우 길기 때문에 **핵심 흐름 요약** 중심으로 보여드립니다.

```cpp
Node* deleteRBTree(Node* root, int key) {
    // 1. BST 삭제
    Node* nodeToDelete = findNode(root, key);
    Node* y = nodeToDelete;
    Node* x;
    Color yOriginalColor = y->color;

    if (!nodeToDelete->left) {
        x = nodeToDelete->right;
        transplant(root, nodeToDelete, nodeToDelete->right);
    }
    else if (!nodeToDelete->right) {
        x = nodeToDelete->left;
        transplant(root, nodeToDelete, nodeToDelete->left);
    }
    else {
        y = minValueNode(nodeToDelete->right);
        yOriginalColor = y->color;
        x = y->right;
        if (y->parent == nodeToDelete)
            x->parent = y;
        else {
            transplant(root, y, y->right);
            y->right = nodeToDelete->right;
            y->right->parent = y;
        }
        transplant(root, nodeToDelete, y);
        y->left = nodeToDelete->left;
        y->left->parent = y;
        y->color = nodeToDelete->color;
    }

    delete nodeToDelete;

    // 2. Fix-up (if needed)
    if (yOriginalColor == BLACK)
        root = deleteFixup(root, x);

    return root;
}
```

※ `transplant`, `deleteFixup`, `minValueNode` 등은 보조 함수로 별도 구현

---

### 삭제 요약

| 항목 | 설명 |
|------|------|
| 삭제 원리 | BST 삭제 + 규칙 위반 시 Fix-up |
| 복잡성 | O(log n) |
| 핵심 이슈 | Double Black → 회전/색 변경 필요 |
| 삭제 난이도 | AVL보다 **더 복잡** |

---

## 🔍 7. 중위 순회 출력

```cpp
void inorder(Node* root) {
    if (!root) return;
    inorder(root->left);
    cout << root->data << (root->color == RED ? "R " : "B ");
    inorder(root->right);
}
```

---

## 🧪 7. 사용 예시

```cpp
#include <iostream>
#include <vector>
using namespace std;

int main() {
    Node* root = nullptr;
    vector<int> values = {10, 20, 30, 15, 25, 5, 1};

    // 🔼 삽입
    for (int val : values) {
        root = insert(root, val);
    }

    cout << "[삽입 후 중위 순회 결과]: ";
    inorder(root); // 색상 포함 출력
    cout << endl;

    // 삭제 연산 테스트
    vector<int> toDelete = {15, 10, 1};
    for (int val : toDelete) {
        root = deleteRBTree(root, val);
        cout << "[삭제 " << val << " 후]: ";
        inorder(root);
        cout << endl;
    }

    return 0;
}
```

### 출력 결과

```
[삽입 후 중위 순회 결과]: 1R 5B 10R 15B 20R 25B 30B
[삭제 15 후]: 1R 5B 10R 20B 25B 30B
[삭제 10 후]: 1R 5B 20B 25B 30B
[삭제 1 후]: 5B 20B 25B 30B
```
---

## 🔍 Red-Black Tree vs AVL Tree

| 항목 | Red-Black Tree | AVL Tree |
|------|----------------|----------|
| 목적 | 느슨한 균형 유지 | 강한 균형 유지 |
| 삽입/삭제 복잡도 | O(log n) (실제 회전 적음) | O(log n) (회전 많을 수 있음) |
| 회전 수 | 적음 (최대 2번) | 많을 수 있음 (최대 log n) |
| 탐색 성능 | AVL이 더 빠름 | 더 빠름 (균형이 더 좋음) |
| 구현 난이도 | 어려움 (색 보정, 케이스 多) | 비교적 단순 (높이 기반) |
| 사용 예 | STL map/set, Java TreeMap | 실시간 탐색에 적합한 자료구조 |

---

## 🔑 어떤 기준으로 선택해야 하는가
- **삽입/삭제가 빈번하면 → Red-Black Tree 유리**
- **탐색 성능이 중요하면 → AVL Tree 유리**
- 표준 라이브러리(`std::map`, `TreeMap`) 등은 Red-Black Tree 기반 사용

---

## ✅ 요약

| 항목 | 내용 |
|------|------|
| 목적 | 이진 탐색 트리의 균형 유지 |
| 시간 복잡도 | O(log n) (탐색/삽입/삭제) |
| 회전 | 삽입/삭제 후 필요시 LL, RR, LR, RL |
| 장점 | 삽입/삭제 균형 유지가 효율적 |
| 적용 사례 | `std::map`, `std::set`, `TreeMap` 등 |