---
layout: post
title: Data Structure - B-Tree
date: 2024-12-09 19:20:23 +0900
category: Data Structure
---
# 🌳 B-Tree 자료구조 - 개념, 원리, 삽입/삭제 알고리즘 완전 정복

---

## 📌 1. B-Tree란?

> **B-Tree**는 **균형 잡힌 다진(多枝) 탐색 트리**로, **디스크 기반 저장 장치**에서 효율적인 데이터 처리를 위해 설계된 자료구조입니다.

- 균형 잡힌 구조로 **항상 O(log n)** 시간에 검색, 삽입, 삭제 가능
- 각 노드는 여러 개의 키(key)와 자식(child)을 가질 수 있음
- **한 번의 디스크 접근으로 많은 키를 검사**할 수 있어, 외부 메모리 환경에 최적화

---

## 🧱 2. B-Tree 용어 정리

| 용어 | 설명 |
|------|------|
| 차수(t) | 하나의 노드가 가질 수 있는 최소/최대 자식 수를 결정하는 수 |
| 키(key) | 노드가 저장하는 정렬된 값들 |
| 자식 포인터 | 다음 하위 노드를 가리키는 포인터 |
| 루트 | 트리의 최상단 노드 |
| 리프 | 자식이 없는 노드 |

---

## ✅ 3. B-Tree의 성질 (차수 t 기준)

- 한 노드는 **최대 2t - 1개 키**, **최대 2t 자식**을 가질 수 있음
- 모든 노드는 **최소 t - 1개 키**, **최소 t 자식** (단, 루트는 예외)
- 키는 노드 내부에서 항상 **오름차순 정렬**
- **중간 노드의 키는 자식 노드를 분할하는 경계 역할**
- **리프는 항상 동일한 깊이**

---

## 🔍 4. 왜 B-Tree가 필요한가?

### ✅ 이진 탐색 트리의 한계

- 메모리 기반에서는 빠르지만, **디스크에서는 비효율적**
- 디스크 I/O는 **랜덤 접근보다 연속 블록 읽기**가 훨씬 빠름
- B-Tree는 한 노드에 많은 키를 넣어 **디스크 접근을 최소화**

### ✅ 디스크 구조 친화적

- 노드 하나가 디스크 블록과 일치
- 한 번에 수십~수백 개의 키 탐색 가능
- 대량의 데이터에 적합

---

## 🧠 5. B-Tree 삽입 알고리즘 (요약)

1. **항상 리프 노드에 삽입**
2. 삽입 위치의 노드가 **가득 찼을 경우**, 중간 키를 위로 올리며 **노드 분할(split)**
3. **재귀적으로 루트까지 분할** 가능 → 트리 높이가 1 증가

---

### 🔧 삽입 과정 예시 (t = 2)

- 노드에 최대 3개 키 저장 가능 (2t-1 = 3)
- `Insert(10)` → 루트에 삽입  
- `Insert(20)` → 루트에 삽입  
- `Insert(5)` → [5, 10, 20]  
- `Insert(6)` → 노드 가득 참 → **분할 발생**  
  → 중간 값 10을 위로 올리고 `[5]` `[20]`이 자식이 됨

---

## 🔨 6. 삭제 알고리즘 (요약)

1. **리프 노드에서 직접 삭제**
2. **중간 노드에서 삭제할 경우**,  
   - 왼쪽 서브트리의 **최댓값** 또는  
   - 오른쪽 서브트리의 **최솟값**으로 대체
3. 삭제 후, 노드의 키 수가 **t-1보다 작아질 경우** → **병합 또는 차용**

---

### 병합 & 차용 개념

- **형제 노드로부터 키를 차용**하거나
- **형제와 병합(merge)**하여 트리 높이를 줄임
- **균형 유지 보장**: 항상 O(log n)

---

## 🛠️ 7. C++ 구조 정의 (기본 틀)

```cpp
// B-Tree 삽입 및 삭제 구현 (차수 T)
#include <iostream>
using namespace std;

const int T = 3; // 최소 차수 t (노드당 최대 2t-1 keys, 2t children)

struct BTreeNode {
    int keys[2 * T - 1];
    BTreeNode* children[2 * T];
    int n; // 현재 key 개수
    bool leaf;

    BTreeNode(bool isLeaf) {
        leaf = isLeaf;
        n = 0;
        for (int i = 0; i < 2 * T; i++) children[i] = nullptr;
    }

    void traverse() {
        for (int i = 0; i < n; i++) {
            if (!leaf) children[i]->traverse();
            cout << keys[i] << " ";
        }
        if (!leaf) children[n]->traverse();
    }

    BTreeNode* search(int k) {
        int i = 0;
        while (i < n && k > keys[i]) i++;
        if (i < n && keys[i] == k) return this;
        if (leaf) return nullptr;
        return children[i]->search(k);
    }

    void insertNonFull(int k);
    void splitChild(int i, BTreeNode* y);
    void remove(int k);
    int findKey(int k);
    void removeFromLeaf(int idx);
    void removeFromNonLeaf(int idx);
    int getPredecessor(int idx);
    int getSuccessor(int idx);
    void fill(int idx);
    void borrowFromPrev(int idx);
    void borrowFromNext(int idx);
    void merge(int idx);
};

struct BTree {
    BTreeNode* root;

    BTree() : root(nullptr) {}

    void traverse() {
        if (root != nullptr) root->traverse();
        cout << endl;
    }

    BTreeNode* search(int k) {
        return (root == nullptr) ? nullptr : root->search(k);
    }

    void insert(int k);
    void remove(int k);
};

void BTree::insert(int k) {
    if (root == nullptr) {
        root = new BTreeNode(true);
        root->keys[0] = k;
        root->n = 1;
    } else {
        if (root->n == 2 * T - 1) {
            BTreeNode* s = new BTreeNode(false);
            s->children[0] = root;
            s->splitChild(0, root);
            int i = (s->keys[0] < k) ? 1 : 0;
            s->children[i]->insertNonFull(k);
            root = s;
        } else {
            root->insertNonFull(k);
        }
    }
}

void BTreeNode::insertNonFull(int k) {
    int i = n - 1;
    if (leaf) {
        while (i >= 0 && keys[i] > k) {
            keys[i + 1] = keys[i];
            i--;
        }
        keys[i + 1] = k;
        n++;
    } else {
        while (i >= 0 && keys[i] > k) i--;
        if (children[i + 1]->n == 2 * T - 1) {
            splitChild(i + 1, children[i + 1]);
            if (keys[i + 1] < k) i++;
        }
        children[i + 1]->insertNonFull(k);
    }
}

void BTreeNode::splitChild(int i, BTreeNode* y) {
    BTreeNode* z = new BTreeNode(y->leaf);
    z->n = T - 1;

    for (int j = 0; j < T - 1; j++)
        z->keys[j] = y->keys[j + T];
    if (!y->leaf) {
        for (int j = 0; j < T; j++)
            z->children[j] = y->children[j + T];
    }
    y->n = T - 1;

    for (int j = n; j >= i + 1; j--)
        children[j + 1] = children[j];
    children[i + 1] = z;

    for (int j = n - 1; j >= i; j--)
        keys[j + 1] = keys[j];
    keys[i] = y->keys[T - 1];
    n++;
}

void BTree::remove(int k) {
    if (!root) return;
    root->remove(k);
    if (root->n == 0) {
        BTreeNode* tmp = root;
        root = root->leaf ? nullptr : root->children[0];
        delete tmp;
    }
}

int BTreeNode::findKey(int k) {
    int idx = 0;
    while (idx < n && keys[idx] < k) ++idx;
    return idx;
}

void BTreeNode::remove(int k) {
    int idx = findKey(k);
    if (idx < n && keys[idx] == k) {
        if (leaf)
            removeFromLeaf(idx);
        else
            removeFromNonLeaf(idx);
    } else {
        if (leaf) return;
        bool flag = (idx == n);
        if (children[idx]->n < T)
            fill(idx);
        if (flag && idx > n)
            children[idx - 1]->remove(k);
        else
            children[idx]->remove(k);
    }
}

void BTreeNode::removeFromLeaf(int idx) {
    for (int i = idx + 1; i < n; ++i)
        keys[i - 1] = keys[i];
    n--;
}

void BTreeNode::removeFromNonLeaf(int idx) {
    int k = keys[idx];
    if (children[idx]->n >= T) {
        int pred = getPredecessor(idx);
        keys[idx] = pred;
        children[idx]->remove(pred);
    } else if (children[idx + 1]->n >= T) {
        int succ = getSuccessor(idx);
        keys[idx] = succ;
        children[idx + 1]->remove(succ);
    } else {
        merge(idx);
        children[idx]->remove(k);
    }
}

int BTreeNode::getPredecessor(int idx) {
    BTreeNode* cur = children[idx];
    while (!cur->leaf)
        cur = cur->children[cur->n];
    return cur->keys[cur->n - 1];
}

int BTreeNode::getSuccessor(int idx) {
    BTreeNode* cur = children[idx + 1];
    while (!cur->leaf)
        cur = cur->children[0];
    return cur->keys[0];
}

void BTreeNode::fill(int idx) {
    if (idx != 0 && children[idx - 1]->n >= T)
        borrowFromPrev(idx);
    else if (idx != n && children[idx + 1]->n >= T)
        borrowFromNext(idx);
    else {
        if (idx != n)
            merge(idx);
        else
            merge(idx - 1);
    }
}

void BTreeNode::borrowFromPrev(int idx) {
    BTreeNode* child = children[idx];
    BTreeNode* sibling = children[idx - 1];

    for (int i = child->n - 1; i >= 0; --i)
        child->keys[i + 1] = child->keys[i];
    if (!child->leaf) {
        for (int i = child->n; i >= 0; --i)
            child->children[i + 1] = child->children[i];
    }
    child->keys[0] = keys[idx - 1];
    if (!child->leaf)
        child->children[0] = sibling->children[sibling->n];
    keys[idx - 1] = sibling->keys[sibling->n - 1];
    child->n += 1;
    sibling->n -= 1;
}

void BTreeNode::borrowFromNext(int idx) {
    BTreeNode* child = children[idx];
    BTreeNode* sibling = children[idx + 1];

    child->keys[child->n] = keys[idx];
    if (!child->leaf)
        child->children[child->n + 1] = sibling->children[0];
    keys[idx] = sibling->keys[0];

    for (int i = 1; i < sibling->n; ++i)
        sibling->keys[i - 1] = sibling->keys[i];
    if (!sibling->leaf) {
        for (int i = 1; i <= sibling->n; ++i)
            sibling->children[i - 1] = sibling->children[i];
    }
    child->n += 1;
    sibling->n -= 1;
}

void BTreeNode::merge(int idx) {
    BTreeNode* child = children[idx];
    BTreeNode* sibling = children[idx + 1];

    child->keys[T - 1] = keys[idx];
    for (int i = 0; i < sibling->n; ++i)
        child->keys[i + T] = sibling->keys[i];
    if (!child->leaf) {
        for (int i = 0; i <= sibling->n; ++i)
            child->children[i + T] = sibling->children[i];
    }

    for (int i = idx + 1; i < n; ++i)
        keys[i - 1] = keys[i];
    for (int i = idx + 2; i <= n; ++i)
        children[i - 1] = children[i];

    child->n += sibling->n + 1;
    n--;
    delete sibling;
}

// 예제 사용
int main() {
    BTree tree;
    for (int k : {10, 20, 5, 6, 12, 30, 7, 17})
        tree.insert(k);

    cout << "트리 순회: ";
    tree.traverse();

    tree.remove(6);
    cout << "\n6 삭제 후: ";
    tree.traverse();

    tree.remove(13); // 없는 값
    tree.remove(7);
    cout << "\n7 삭제 후: ";
    tree.traverse();

    tree.remove(4); // 없는 값
    tree.remove(2); // 없는 값
    tree.remove(16); // 없는 값

    return 0;
}
```

---

## 📦 8. 실제 사용 사례

| 분야 | 설명 |
|------|------|
| 파일 시스템 | NTFS, ext4, HFS+ |
| 데이터베이스 | MySQL (InnoDB), PostgreSQL |
| 키-값 저장소 | RocksDB, LevelDB |
| 인덱싱 | B+Tree로 구현됨 (다음 편) |

---

## ✅ 9. B-Tree vs 다른 트리

| 항목 | BST | AVL | B-Tree |
|------|-----|-----|--------|
| 균형 보장 | × | ○ | ○ |
| 자식 수 | 2 | 2 | 2~2t |
| 외부 메모리 최적화 | × | × | ✅ |
| 디스크 접근 최소화 | × | × | ✅ |
| 사용 용도 | 메모리 탐색 | 빠른 탐색 | 대용량 저장소 |

---

## ✅ 마무리 요약

| 키워드 | 설명 |
|--------|------|
| B-Tree | 외부 메모리 최적화 다진 균형 트리 |
| 시간복잡도 | O(log n) 탐색/삽입/삭제 |
| 삽입 | 항상 리프에 삽입, 분할 발생 가능 |
| 삭제 | 대체 + 병합/차용 |
| 사용처 | DB, 파일 시스템, 인덱스 등 |