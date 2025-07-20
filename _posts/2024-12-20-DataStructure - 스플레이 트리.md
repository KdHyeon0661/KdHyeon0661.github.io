---
layout: post
title: Data Structure - 펜윅 트리
date: 2024-12-09 19:20:23 +0900
category: Data Structure
---
# 🌳 스플레이 트리 (Splay Tree) - 자가 조정 이진 탐색 트리

## 📌 1. 스플레이 트리란?

**스플레이 트리(Splay Tree)** 는 **자가 조정(Self-Adjusting)** 기능이 있는 **이진 탐색 트리(BST)**입니다.

- 자주 접근되는 노드를 루트에 가깝게 끌어올려 탐색 속도를 향상시킵니다.
- **최근 사용된 노드를 루트로 올리는 스플레이(Splay) 연산**이 핵심입니다.

---

## 🧠 2. 핵심 아이디어

### ✅ 주요 목적
- **탐색 시간 최적화**: 자주 쓰는 데이터는 더 빠르게 접근
- **균형 유지**: 명시적인 균형 재조정은 없지만 평균 성능은 로그 수준 유지

### ✅ 시간 복잡도
| 연산 | 평균 시간 | 최악 시간 (단일 연산) | 전체 시퀀스 |
|------|------------|-----------------------|--------------|
| 탐색, 삽입, 삭제 | O(log n) | O(n) | O(m log n) for m ops |

---

## 🔁 3. 스플레이 연산 종류

스플레이 트리는 다음과 같은 **회전(Rotation)** 을 조합해 **노드를 루트로 끌어올립니다**.

| 유형 | 조건 | 동작 |
|------|------|------|
| Zig | 부모만 존재 | 단일 회전 |
| Zig-Zig | 부모-조부모 방향 동일 | 2회 연속 회전 |
| Zig-Zag | 부모-조부모 방향 다름 | 교차 회전 (좌우) |

---

## 🛠️ 4. C++ 구현

### ✅ 노드 구조

```cpp
struct Node {
    int key;
    Node* left;
    Node* right;
    Node* parent;

    Node(int val) : key(val), left(nullptr), right(nullptr), parent(nullptr) {}
};
```

---

### ✅ 회전 함수

```cpp
void rotate(Node*& root, Node* x) {
    Node* p = x->parent;
    Node* g = p ? p->parent : nullptr;

    if (p->left == x) {
        p->left = x->right;
        if (x->right) x->right->parent = p;
        x->right = p;
    } else {
        p->right = x->left;
        if (x->left) x->left->parent = p;
        x->left = p;
    }

    x->parent = g;
    p->parent = x;

    if (g) {
        if (g->left == p) g->left = x;
        else g->right = x;
    } else {
        root = x;
    }
}
```

---

### ✅ 스플레이 연산

```cpp
void splay(Node*& root, Node* x) {
    while (x->parent) {
        Node* p = x->parent;
        Node* g = p->parent;
        if (!g) {
            rotate(root, x); // Zig
        } else if ((g->left == p && p->left == x) || (g->right == p && p->right == x)) {
            rotate(root, p); // Zig-Zig
            rotate(root, x);
        } else {
            rotate(root, x); // Zig-Zag
            rotate(root, x);
        }
    }
}
```

---

## ✏️ 5. 삽입, 탐색, 삭제

### 🔍 탐색 + 스플레이

```cpp
Node* search(Node*& root, int key) {
    Node* x = root;
    while (x) {
        if (key == x->key) break;
        x = (key < x->key) ? x->left : x->right;
    }
    if (x) splay(root, x);
    return x;
}
```

---

### ➕ 삽입

```cpp
void insert(Node*& root, int key) {
    Node* parent = nullptr;
    Node* cur = root;
    while (cur) {
        parent = cur;
        cur = (key < cur->key) ? cur->left : cur->right;
    }

    Node* newNode = new Node(key);
    newNode->parent = parent;

    if (!parent) root = newNode;
    else if (key < parent->key) parent->left = newNode;
    else parent->right = newNode;

    splay(root, newNode);
}
```

---

### ❌ 삭제

```cpp
void deleteNode(Node*& root, int key) {
    Node* target = search(root, key);
    if (!target) return;

    splay(root, target);

    Node* left = root->left;
    Node* right = root->right;

    if (left) left->parent = nullptr;
    if (right) right->parent = nullptr;

    delete root;

    if (!left) {
        root = right;
    } else {
        Node* maxLeft = left;
        while (maxLeft->right) maxLeft = maxLeft->right;
        splay(left, maxLeft);
        left->right = right;
        if (right) right->parent = left;
        root = left;
    }
}
```

---

## 🔬 6. 예제 실행

```cpp
int main() {
    Node* root = nullptr;

    insert(root, 10);
    insert(root, 20);
    insert(root, 5);
    insert(root, 30);

    search(root, 5); // 루트로 이동

    deleteNode(root, 10); // 삭제 후 재구성

    cout << "Root after operations: " << root->key << endl;

    return 0;
}
```

---

## 🧮 7. 스플레이 트리 vs 다른 트리

| 기준 | 스플레이 트리 | AVL 트리 | Red-Black 트리 |
|------|----------------|-----------|-----------------|
| 균형 유지 방식 | 자가 조정 | 엄격한 높이 균형 | 느슨한 균형 |
| 삽입/삭제 시간 | O(log n) (암시적) | O(log n) | O(log n) |
| 탐색 최적화 | 자주 쓰는 키 빠름 | 일반 탐색 | 일반 탐색 |
| 구현 난이도 | 보통 | 높음 | 높음 |

---

## ✅ 정리

| 항목 | 설명 |
|------|------|
| 구조 | 자가 조정 이진 탐색 트리 |
| 핵심 | Splay 연산으로 루트 끌어올림 |
| 시간 복잡도 | 평균 O(log n), 시퀀스 성능 우수 |
| 장점 | 자주 쓰는 키 빠르게 접근 |
| 단점 | 최악 시간 O(n), 구현 복잡도 |