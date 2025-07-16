---
layout: post
title: Data Structure - 펜윅 트리
date: 2024-12-09 19:20:23 +0900
category: Data Structure
---
# 🔗 Link-Cut Tree (LCT)

## 📌 1. 개요

**Link-Cut Tree**는 **동적 트리(Dynamic Tree)** 구조를 표현하고 관리하기 위한 고급 자료구조입니다.  
**포인터 조작과 Splay Tree**를 기반으로 하며, **동적으로 연결(연결/절단)** 되는 트리 간의 질의를 빠르게 수행할 수 있습니다.

---

## 🧠 2. 주요 특징

| 기능 | 설명 |
|------|------|
| 동적 트리 | 간선 추가/제거가 가능 |
| 질의 속도 | O(log n) |
| 내부 구조 | Splay Tree 기반 |
| 대표 연산 | `link`, `cut`, `findRoot`, `pathQuery` 등 |

---

## 🧩 3. 지원 연산

| 연산 | 설명 |
|------|------|
| `link(x, y)` | 노드 x를 y의 자식으로 연결 |
| `cut(x)` | x와 부모 사이의 간선을 절단 |
| `findRoot(x)` | x가 속한 트리의 루트 노드를 반환 |
| `connected(x, y)` | x와 y가 같은 트리에 속하는지 확인 |
| `pathQuery(x, y)` | 경로상의 노드 값들에 대한 질의 |

---

## 🛠️ 4. 핵심 아이디어: Splay Tree 기반 표현

각 노드는 Splay Tree의 형태로 관리되며, **Preferred Path Decomposition** 을 사용합니다.

### ✅ Push / Update

```cpp
void push(Node* x) {
    if (x && x->rev) {
        swap(x->left, x->right);
        if (x->left) x->left->rev ^= 1;
        if (x->right) x->right->rev ^= 1;
        x->rev = false;
    }
}
```

---

## 🏗️ 5. 구조 정의 및 기본 연산 (C++)

### ✅ 노드 정의

```cpp
struct Node {
    Node* left = nullptr;
    Node* right = nullptr;
    Node* parent = nullptr;
    bool rev = false;
    int id; // node identifier

    Node(int _id) : id(_id) {}
};
```

---

### ✅ Splay Tree 연산

```cpp
bool isRoot(Node* x) {
    return !x->parent || (x->parent->left != x && x->parent->right != x);
}

void rotate(Node* x) {
    Node* p = x->parent;
    Node* g = p->parent;
    if (!isRoot(p)) {
        if (g->left == p) g->left = x;
        else g->right = x;
    }
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
}

void splay(Node* x) {
    push(x);
    while (!isRoot(x)) {
        Node* p = x->parent;
        Node* g = p->parent;
        if (!isRoot(p)) push(g);
        push(p); push(x);
        if (!isRoot(p)) {
            if ((g->left == p) == (p->left == x))
                rotate(p);
            else
                rotate(x);
        }
        rotate(x);
    }
}
```

---

### ✅ 액세스 및 경로 반전

```cpp
void access(Node* x) {
    Node* last = nullptr;
    for (Node* y = x; y; y = y->parent) {
        splay(y);
        y->right = last;
        last = y;
    }
    splay(x);
}

void makeRoot(Node* x) {
    access(x);
    x->rev ^= 1;
    push(x);
}
```

---

## 🔗 6. 연결/절단 연산

```cpp
void link(Node* x, Node* y) {
    makeRoot(x);
    x->parent = y;
}

void cut(Node* x, Node* y) {
    makeRoot(x);
    access(y);
    if (y->left == x && !x->right) {
        y->left = x->parent = nullptr;
    }
}

bool connected(Node* x, Node* y) {
    if (x == y) return true;
    makeRoot(x);
    access(y);
    return x->parent != nullptr;
}
```

---

## 🧪 7. 사용 예시

```cpp
int main() {
    const int N = 6;
    vector<Node*> tree(N + 1);
    for (int i = 1; i <= N; ++i)
        tree[i] = new Node(i);

    link(tree[1], tree[2]);
    link(tree[2], tree[3]);
    link(tree[3], tree[4]);

    cout << "1 and 4 connected? " << connected(tree[1], tree[4]) << "\n"; // 1
    cut(tree[2], tree[3]);
    cout << "1 and 4 connected after cut? " << connected(tree[1], tree[4]) << "\n"; // 0

    return 0;
}
```

---

## 📌 8. 요약

| 항목 | 설명 |
|------|------|
| 구조 | Splay Tree 기반 동적 트리 |
| 연산 | 링크, 절단, 루트 찾기 등 |
| 시간 복잡도 | O(log n) |
| 응용 분야 | 네트워크 연결, 최소 신장 트리, 동적 그래프 등 |