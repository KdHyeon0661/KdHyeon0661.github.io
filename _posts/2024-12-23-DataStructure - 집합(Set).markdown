---
layout: post
title: Data Structure - 펜윅 트리
date: 2024-12-09 19:20:23 +0900
category: Data Structure
---
# 🌿 C++ STL Set 완전 정리 (with 내부 구조)

---

## 1️⃣ Set이란?

> Set은 **중복되지 않는 원소**를 **자동 정렬된 상태**로 저장하는 컨테이너입니다. C++에는 set, multiset, unordered_set, unordered_multiset이 있습니다.

- 키만 저장 (`Key == Value`)
- 삽입 시 자동 정렬됨
- 중복된 값 허용 ❌
- 내부적으로 **Red-Black Tree**로 구현됨

---

## 2️⃣ Set의 특징

| 항목 | 내용 |
|------|------|
| 중복 허용 | ❌ 불가능 |
| 정렬 | 자동 (기본은 `<`) |
| 삽입/삭제/탐색 | O(log n) |
| 순회 순서 | 정렬된 순서 |
| 내부 구조 | Red-Black Tree |
| 반복자 | bidirectional iterator 지원 |

---

## 3️⃣ Set vs Multiset

| 항목 | set | multiset |
|------|-----|----------|
| 중복 허용 | ❌ | ✅ |
| 동일 값 삽입 | 무시됨 | 모두 저장됨 |
| 내부 정렬 | O(log n) | O(log n) |
| 삭제 | `erase(value)` 1개 삭제 | `erase(value)` 모든 복제 삭제 |

---

## Set vs unordered_set

| 항목 | set | unordered_set |
|------|-----|----------------|
| 정렬 | O | ❌ |
| 탐색/삽입 | O(log n) | 평균 O(1), 최악 O(n) |
| 내부 구조 | Red-Black Tree | Hash Table |
| 순서 보장 | ✅ | ❌ |
| 메모리 | 적음 | 더 큼 |
| 커스텀 정렬 | 가능 | 불가능 |

---

## Set vs unordered_multiset

| 항목            | `set`                                | `unordered_set`                        |
|-----------------|----------------------------------------|----------------------------------------|
| 정렬 여부    | ✅ 오름차순 정렬 (기본 `std::less`)     | ❌ 없음 (순서 무작위)                   |
| 탐색/삽입/삭제 | `O(log n)` (트리 깊이에 비례)          | 평균 `O(1)`, 최악 `O(n)` (해시 충돌 시) |
| 내부 구조    | Red-Black Tree (이진 균형 탐색 트리)   | Hash Table (버킷 + 해시 함수)          |
| 순서 보장    | ✅ 정렬된 순서 유지                     | ❌ 입력 순서나 정렬 순서 모두 보장 X     |
| 중복 허용    | ❌ 중복 허용 안 함                     | ❌ 중복 허용 안 함                     |
| 메모리 사용  | 더 적음 (트리 구조)                    | 더 큼 (버킷과 해시 구조로 인한 오버헤드) |
| 커스텀 정렬  | ✅ `std::greater` 등 비교자 지정 가능   | ❌ 정렬 개념 없음 → 커스텀 불가         |
| 사용 예시    | 정렬된 고유 값 목록                    | 빠른 검색이 필요한 고유 값 집합         |

---

## 4️⃣ 내부 구조는 Red-Black Tree

- `std::set`도 `std::map`처럼 **자체 균형 BST(Red-Black Tree)** 기반
- 모든 연산은 O(log n)
- 값이 키 역할을 하며 정렬 순서를 유지

---

## 5️⃣ 사용 예제 (삽입, 탐색, 삭제)

```cpp
#include <iostream>
#include <set>
using namespace std;

int main() {
    set<int> s;

    s.insert(10);
    s.insert(5);
    s.insert(20);
    s.insert(10); // 중복 무시

    cout << "Set contains: ";
    for (int x : s) cout << x << " ";
    cout << endl;

    if (s.find(10) != s.end())
        cout << "10 found" << endl;

    s.erase(10); // 삭제

    cout << "After erase: ";
    for (int x : s) cout << x << " ";
    cout << endl;

    return 0;
}
```

📌 출력:
```
Set contains: 5 10 20  
10 found  
After erase: 5 20
```

---

## 6️⃣ 사용자 정의 정렬 (Comparator)

```cpp
struct Descending {
    bool operator()(int a, int b) const {
        return a > b;
    }
};

set<int, Descending> s;  // 내림차순 정렬
```

- `operator()`를 통해 사용자 지정 정렬 기준을 정의 가능
- 정렬 방식만 바꾸고 내부는 여전히 Red-Black Tree

---

## 구현 예제

```cpp
#include <iostream>
using namespace std;

enum Color { RED, BLACK };

struct Node {
    int key;
    Color color;
    Node *left, *right, *parent;
    Node(int k) : key(k), color(RED), left(nullptr), right(nullptr), parent(nullptr) {}
};

class RBSet {
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

    bool contains(Node* node, int key) {
        if (!node) return false;
        if (key == node->key) return true;
        if (key < node->key) return contains(node->left, key);
        return contains(node->right, key);
    }

    void inorder(Node* node) {
        if (!node) return;
        inorder(node->left);
        cout << node->key << "(" << (node->color == RED ? "R" : "B") << ") ";
        inorder(node->right);
    }

    void free(Node* node) {
        if (!node) return;
        free(node->left);
        free(node->right);
        delete node;
    }

    void transplant(Node* u, Node* v) {
        if (!u->parent) root = v;
        else if (u == u->parent->left) u->parent->left = v;
        else u->parent->right = v;
        if (v) v->parent = u->parent;
    }

    Node* minimum(Node* node) {
        while (node->left) node = node->left;
        return node;
    }

    void fixDelete(Node* x) {
        while (x != root && (!x || x->color == BLACK)) {
            if (x == x->parent->left) {
                Node* w = x->parent->right;
                if (w && w->color == RED) {
                    w->color = BLACK;
                    x->parent->color = RED;
                    leftRotate(x->parent);
                    w = x->parent->right;
                }
                if ((!w->left || w->left->color == BLACK) &&
                    (!w->right || w->right->color == BLACK)) {
                    w->color = RED;
                    x = x->parent;
                } else {
                    if (!w->right || w->right->color == BLACK) {
                        if (w->left) w->left->color = BLACK;
                        w->color = RED;
                        rightRotate(w);
                        w = x->parent->right;
                    }
                    w->color = x->parent->color;
                    x->parent->color = BLACK;
                    if (w->right) w->right->color = BLACK;
                    leftRotate(x->parent);
                    x = root;
                }
            } else {
                Node* w = x->parent->left;
                if (w && w->color == RED) {
                    w->color = BLACK;
                    x->parent->color = RED;
                    rightRotate(x->parent);
                    w = x->parent->left;
                }
                if ((!w->right || w->right->color == BLACK) &&
                    (!w->left || w->left->color == BLACK)) {
                    w->color = RED;
                    x = x->parent;
                } else {
                    if (!w->left || w->left->color == BLACK) {
                        if (w->right) w->right->color = BLACK;
                        w->color = RED;
                        leftRotate(w);
                        w = x->parent->left;
                    }
                    w->color = x->parent->color;
                    x->parent->color = BLACK;
                    if (w->left) w->left->color = BLACK;
                    rightRotate(x->parent);
                    x = root;
                }
            }
        }
        if (x) x->color = BLACK;
    }

public:
    ~RBSet() { free(root); }

    void insert(int key) {
        Node* z = new Node(key);
        Node* y = nullptr;
        Node* x = root;

        while (x) {
            y = x;
            if (key == x->key) { delete z; return; } // 중복 무시
            if (key < x->key) x = x->left;
            else x = x->right;
        }

        z->parent = y;
        if (!y) root = z;
        else if (key < y->key) y->left = z;
        else y->right = z;

        fixInsert(z);
    }

    void erase(int key) {
        Node* z = root;
        while (z && z->key != key) {
            if (key < z->key) z = z->left;
            else z = z->right;
        }
        if (!z) return;

        Node* y = z;
        Node* x = nullptr;
        Color y_original_color = y->color;

        if (!z->left) {
            x = z->right;
            transplant(z, z->right);
        } else if (!z->right) {
            x = z->left;
            transplant(z, z->left);
        } else {
            y = minimum(z->right);
            y_original_color = y->color;
            x = y->right;
            if (y->parent == z) {
                if (x) x->parent = y;
            } else {
                transplant(y, y->right);
                y->right = z->right;
                y->right->parent = y;
            }
            transplant(z, y);
            y->left = z->left;
            y->left->parent = y;
            y->color = z->color;
        }

        delete z;

        if (y_original_color == BLACK)
            fixDelete(x);
    }

    bool contains(int key) { return contains(root, key); }
    void print() { inorder(root); cout << endl; }
};

int main() {
    RBSet s;
    s.insert(10);
    s.insert(5);
    s.insert(20);
    s.insert(15);
    s.insert(10); // 중복 무시

    cout << "Set contents (before erase): ";
    s.print();

    s.erase(10);
    s.erase(20);

    cout << "Set contents (after erase 10 and 20): ";
    s.print();

    cout << (s.contains(15) ? "Contains 15" : "Does not contain 15") << endl;
    cout << (s.contains(10) ? "Contains 10" : "Does not contain 10") << endl;
    cout << (s.contains(100) ? "Contains 100" : "Does not contain 100") << endl;

    return 0;
}
```
---

## ✅ 요약

| 키워드 | 설명 |
|--------|------|
| set | 중복 없이 정렬된 값 저장 |
| 내부 구현 | Red-Black Tree |
| 복잡도 | 삽입/삭제/탐색 O(log n) |
| 멀티셋 | 중복 허용 |
| 해시 기반 대안 | unordered_set |