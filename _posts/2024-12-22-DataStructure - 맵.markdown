---
layout: post
title: Data Structure - 펜윅 트리
date: 2024-12-09 19:20:23 +0900
category: Data Structure
---
# 📌 C++ std::map과 내부 구조 구현

---

## 1️⃣ std::map 개념

> `std::map<Key, Value>`는 키-값 쌍을 저장하는 **연관 컨테이너**입니다.

- 모든 키는 **유일(unique)** 해야 함
- 내부적으로는 **Red-Black Tree(RBT)** 를 사용
- 자동으로 **키 기준 정렬** (기본은 `<` 연산자 기준)

```cpp
#include <map>
#include <iostream>
using namespace std;

int main() {
    map<string, int> m;
    m["apple"] = 5;
    m["banana"] = 2;
    m["orange"] = 7;

    for (auto& [key, value] : m)
        cout << key << ": " << value << endl;
    return 0;
}
```

> 출력은 키의 **정렬 순서**:  
```
apple: 5  
banana: 2  
orange: 7
```

---

## 2️⃣ std::map 특징 요약

| 항목 | 내용 |
|------|------|
| 키 중복 | ❌ 불가능 (자동 덮어쓰기) |
| 정렬 기준 | 기본 `operator<` |
| 삽입/탐색/삭제 | O(log n) (트리 높이) |
| 순회 순서 | 키의 정렬 순서 |
| 내부 구현 | **Red-Black Tree** |

---

## 3️⃣ 내부 구조는 Red-Black Tree?

`std::map`은 일반적으로 **Red-Black Tree (균형 이진 탐색 트리)** 로 구현됩니다.

> 🎯 Red-Black Tree는 다음 조건을 만족하는 **자체 균형 이진 탐색 트리**입니다:

- 노드는 빨간색(Red) 또는 검정색(Black)
- 루트 노드는 항상 검정색
- 모든 리프(NIL)는 검정색
- 빨간 노드의 자식은 모두 검정색
- 임의 노드에서 리프까지의 경로에 있는 검정 노드 수는 동일

✅ 이렇게 하면 **최악의 경우에도 O(log n)** 보장!

---

## 4️⃣ 내부 구조 구현

> 🛠 아래 코드는 간단한 map 기능(BST + Red Black Tree 기반 삽입/검색/순회)을 구현한 예입니다.

```cpp
#include <iostream>
using namespace std;

enum Color { RED, BLACK };

struct Node {
    string key;
    int value;
    Color color;
    Node *left, *right, *parent;

    Node(string k, int v) : key(k), value(v), color(RED), left(nullptr), right(nullptr), parent(nullptr) {}
};

class RBMap {
    Node* root = nullptr;

    void leftRotate(Node* x) {
        Node* y = x->right;
        x->right = y->left;
        if (y->left) y->left->parent = x;
        y->parent = x->parent;
        if (!x->parent) root = y;
        else if (x == x->parent->left) x->parent->left = y;
        else x->parent->right = y;
        y->left = x;
        x->parent = y;
    }

    void rightRotate(Node* x) {
        Node* y = x->left;
        x->left = y->right;
        if (y->right) y->right->parent = x;
        y->parent = x->parent;
        if (!x->parent) root = y;
        else if (x == x->parent->right) x->parent->right = y;
        else x->parent->left = y;
        y->right = x;
        x->parent = y;
    }

    void fixInsert(Node* z) {
        while (z->parent && z->parent->color == RED) {
            Node* gp = z->parent->parent;
            if (z->parent == gp->left) {
                Node* y = gp->right;
                if (y && y->color == RED) {
                    z->parent->color = BLACK;
                    y->color = BLACK;
                    gp->color = RED;
                    z = gp;
                } else {
                    if (z == z->parent->right) {
                        z = z->parent;
                        leftRotate(z);
                    }
                    z->parent->color = BLACK;
                    gp->color = RED;
                    rightRotate(gp);
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
                        rightRotate(z);
                    }
                    z->parent->color = BLACK;
                    gp->color = RED;
                    leftRotate(gp);
                }
            }
        }
        root->color = BLACK;
    }

    void inorder(Node* node) {
        if (!node) return;
        inorder(node->left);
        cout << node->key << " (" << (node->color == RED ? "R" : "B") << "): " << node->value << endl;
        inorder(node->right);
    }

public:
    void insert(string key, int value) {
        Node* z = new Node(key, value);
        Node* y = nullptr;
        Node* x = root;

        while (x) {
            y = x;
            if (key < x->key) x = x->left;
            else if (key > x->key) x = x->right;
            else {
                x->value = value;
                delete z;
                return;
            }
        }

        z->parent = y;
        if (!y) root = z;
        else if (key < y->key) y->left = z;
        else y->right = z;

        fixInsert(z);
    }

    bool get(string key, int& out) {
        Node* node = root;
        while (node) {
            if (key == node->key) {
                out = node->value;
                return true;
            } else if (key < node->key) node = node->left;
            else node = node->right;
        }
        return false;
    }

    void print() {
        inorder(root);
    }
};

// -------------------------
// 🚀 테스트 예시
// -------------------------
int main() {
    RBMap map;
    map.insert("banana", 3);
    map.insert("apple", 1);
    map.insert("cherry", 2);
    map.insert("banana", 5); // 업데이트

    int v;
    if (map.get("banana", v)) cout << "banana = " << v << endl;

    map.print();
    return 0;
}

```

---

## 5️⃣ std::map vs std::unordered_map

| 항목 | map | unordered_map |
|------|-----|----------------|
| 내부 구조 | Red-Black Tree | Hash Table |
| 시간 복잡도 | O(log n) | 평균 O(1), 최악 O(n) |
| 키 정렬 | ✅ 지원 | ❌ 없음 |
| 반복 순서 | 정렬 순 | 임의 순 |
| 메모리 사용 | 보통 | 더 큼 |
| 커스텀 정렬 | 가능 (comparator) | 불가능 |

---

## ✅ 요약

| 키워드 | 요약 |
|--------|------|
| std::map | 정렬된 키 기반, RBT로 구현된 연관 컨테이너 |
| 시간 복잡도 | 삽입/탐색/삭제 O(log n) |
| 구현 | 일반적으로 Red-Black Tree 기반 |
| 대안 | 정렬 필요 없다면 `unordered_map` 고려 |