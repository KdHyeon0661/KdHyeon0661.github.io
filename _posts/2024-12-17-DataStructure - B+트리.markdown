---
layout: post
title: Data Structure - B+트리
date: 2024-12-09 19:20:23 +0900
category: Data Structure
---
# 🌳 B+ Tree 자료구조 - 개념, 비교, 인덱싱 예제

B+ 트리는 **데이터베이스, 파일 시스템** 등에서 널리 사용되는 **균형 다진 탐색 트리**입니다.  
B-Tree와 유사하지만, 인덱싱 및 범위 검색에서 더 뛰어난 성능을 보입니다.

---

## 📌 1. B+ 트리란?

> B+ 트리는 **B-Tree에서 발전한 구조**로, 모든 실제 데이터를 **리프 노드에만 저장**하며, 내부 노드는 오직 인덱스 역할만 합니다.

### 🔑 특징

- 모든 **실제 값은 리프 노드**에만 저장
- 리프 노드들은 **오른쪽으로 연결된 Linked List 형태**
- **범위 검색에 효율적**
- **균형 트리**로 항상 로그 시간 내 탐색 가능

---

## ⚖️ 2. B-Tree vs B+Tree 비교

| 항목 | B-Tree | B+Tree |
|------|--------|--------|
| 데이터 저장 위치 | 모든 노드 | 리프 노드에만 |
| 내부 노드의 역할 | 인덱스 + 값 | 인덱스만 |
| 리프 노드 연결 | 연결 없음 | 오른쪽으로 연결 리스트 구성 |
| 범위 검색 성능 | 느림 | 빠름 (리프 순회로 가능) |
| 디스크 I/O 비용 | 보통 | 효율적 (인덱스 작아짐) |
| 사용 예시 | 일부 메모리 트리 | **데이터베이스 인덱스**, **파일 시스템** 등 |

---

## 🌿 3. B+ 트리 구조 예시 (차수 3, fanout = 3)

데이터 삽입 순서: `10, 20, 5, 6, 12, 30, 7, 17`

```
Internal:
           [10, 20]
          /   |    \ 
         /    |     \
      [5,6,7] [10,12,17] [20,30]  <- 리프 노드들
```

### 🔗 리프 노드 연결
```
[5,6,7] → [10,12,17] → [20,30]
```

---

## 🔍 4. B+ 트리 인덱싱 예제 (가상의 레코드 ID 사용)

### 📁 레코드 구성
```
ID:   5   6   7   10   12   17   20   30
Name: A   B   C    D    E    F    G    H
```

### 🔎 인덱싱 과정

1. **리프 노드에 데이터 저장**  
    ```
    [5:A, 6:B, 7:C] → [10:D, 12:E, 17:F] → [20:G, 30:H]
    ```

2. **내부 노드에 인덱스 키 저장**  
   - 자식 리프 노드의 **첫 번째 키**를 사용  
    ```
    Root → [10, 20]
        → [5:A, 6:B, 7:C], [10:D, 12:E, 17:F], [20:G, 30:H]
    ```

3. **검색 예시**
   - `12` 검색:
     - Root에서 10 ≤ 12 < 20 → 중간 블록 진입
     - `[10:D, 12:E, 17:F]` 탐색 → `E` 반환

4. **범위 검색: 7 ~ 20**
   - 시작: `[7:C]` → 다음 블록 순차 탐색
   - 결과: C, D, E, F, G

---

## 기본 코드

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
using namespace std;

const int T = 3; // 최소 차수 t (노드당 최대 2t-1 keys, 2t children)

class BPlusNode {
public:
    bool isLeaf;
    vector<int> keys;
    vector<BPlusNode*> children;
    BPlusNode* next;

    BPlusNode(bool leaf) : isLeaf(leaf), next(nullptr) {}
};

class BPlusTree {
private:
    BPlusNode* root;

    void insertInternal(int key, BPlusNode* cursor, BPlusNode* child);
    BPlusNode* findParent(BPlusNode* cursor, BPlusNode* child);
    void deleteEntry(BPlusNode* node, int key);
    void merge(BPlusNode* parent, int index);
    void borrowFromLeft(BPlusNode* parent, int index);
    void borrowFromRight(BPlusNode* parent, int index);

public:
    BPlusTree() : root(nullptr) {}
    void insert(int key);
    void remove(int key);
    void display();
    void rangeSearch(int start, int end);
};

void BPlusTree::insert(int key) {
    if (!root) {
        root = new BPlusNode(true);
        root->keys.push_back(key);
        return;
    }

    BPlusNode* cursor = root;
    BPlusNode* parent = nullptr;

    while (!cursor->isLeaf) {
        parent = cursor;
        int i = 0;
        while (i < cursor->keys.size() && key >= cursor->keys[i]) i++;
        cursor = cursor->children[i];
    }

    cursor->keys.push_back(key);
    sort(cursor->keys.begin(), cursor->keys.end());

    if (cursor->keys.size() < 2 * T - 1) return;

    BPlusNode* newLeaf = new BPlusNode(true);
    int mid = cursor->keys.size() / 2;
    newLeaf->keys.assign(cursor->keys.begin() + mid, cursor->keys.end());
    cursor->keys.resize(mid);

    newLeaf->next = cursor->next;
    cursor->next = newLeaf;

    if (cursor == root) {
        BPlusNode* newRoot = new BPlusNode(false);
        newRoot->keys.push_back(newLeaf->keys[0]);
        newRoot->children.push_back(cursor);
        newRoot->children.push_back(newLeaf);
        root = newRoot;
    } else {
        insertInternal(newLeaf->keys[0], parent, newLeaf);
    }
}

void BPlusTree::insertInternal(int key, BPlusNode* cursor, BPlusNode* child) {
    cursor->keys.push_back(key);
    sort(cursor->keys.begin(), cursor->keys.end());
    int idx = find(cursor->keys.begin(), cursor->keys.end(), key) - cursor->keys.begin();
    cursor->children.insert(cursor->children.begin() + idx + 1, child);

    if (cursor->keys.size() < 2 * T - 1) return;

    int mid = cursor->keys.size() / 2;
    BPlusNode* newInternal = new BPlusNode(false);
    newInternal->keys.assign(cursor->keys.begin() + mid + 1, cursor->keys.end());
    newInternal->children.assign(cursor->children.begin() + mid + 1, cursor->children.end());
    int upKey = cursor->keys[mid];

    cursor->keys.resize(mid);
    cursor->children.resize(mid + 1);

    if (cursor == root) {
        BPlusNode* newRoot = new BPlusNode(false);
        newRoot->keys.push_back(upKey);
        newRoot->children.push_back(cursor);
        newRoot->children.push_back(newInternal);
        root = newRoot;
    } else {
        insertInternal(upKey, findParent(root, cursor), newInternal);
    }
}

BPlusNode* BPlusTree::findParent(BPlusNode* cursor, BPlusNode* child) {
    if (cursor->isLeaf || cursor->children[0]->isLeaf) return nullptr;
    for (auto c : cursor->children) {
        if (c == child) return cursor;
        BPlusNode* p = findParent(c, child);
        if (p) return p;
    }
    return nullptr;
}

void BPlusTree::remove(int key) {
    BPlusNode* cursor = root;
    BPlusNode* parent = nullptr;
    int index = -1;

    while (!cursor->isLeaf) {
        parent = cursor;
        index = 0;
        while (index < cursor->keys.size() && key >= cursor->keys[index]) index++;
        cursor = cursor->children[index];
    }

    auto it = find(cursor->keys.begin(), cursor->keys.end(), key);
    if (it != cursor->keys.end()) {
        cursor->keys.erase(it);
        if (cursor == root) {
            if (cursor->keys.empty()) {
                delete cursor;
                root = nullptr;
            }
            return;
        }

        if (cursor->keys.size() < T - 1) {
            deleteEntry(cursor, key);
        }
    }
}

void BPlusTree::deleteEntry(BPlusNode* node, int key) {
    BPlusNode* parent = findParent(root, node);
    if (!parent) return;

    int index = 0;
    while (index < parent->children.size() && parent->children[index] != node) index++;

    if (index > 0 && parent->children[index - 1]->keys.size() > T - 1) {
        borrowFromLeft(parent, index);
    } else if (index < parent->children.size() - 1 && parent->children[index + 1]->keys.size() > T - 1) {
        borrowFromRight(parent, index);
    } else {
        if (index > 0) merge(parent, index);
        else merge(parent, index + 1);
    }

    if (parent == root && parent->keys.empty()) {
        root = parent->children[0];
        delete parent;
    }
}

void BPlusTree::merge(BPlusNode* parent, int index) {
    BPlusNode* left = parent->children[index - 1];
    BPlusNode* right = parent->children[index];

    left->keys.insert(left->keys.end(), right->keys.begin(), right->keys.end());
    if (left->isLeaf) left->next = right->next;
    else left->children.insert(left->children.end(), right->children.begin(), right->children.end());

    parent->keys.erase(parent->keys.begin() + index - 1);
    parent->children.erase(parent->children.begin() + index);
    delete right;
}

void BPlusTree::borrowFromLeft(BPlusNode* parent, int index) {
    BPlusNode* left = parent->children[index - 1];
    BPlusNode* node = parent->children[index];

    node->keys.insert(node->keys.begin(), left->keys.back());
    left->keys.pop_back();
    parent->keys[index - 1] = node->keys.front();
}

void BPlusTree::borrowFromRight(BPlusNode* parent, int index) {
    BPlusNode* right = parent->children[index + 1];
    BPlusNode* node = parent->children[index];

    node->keys.push_back(right->keys.front());
    right->keys.erase(right->keys.begin());
    parent->keys[index] = right->keys.front();
}

void BPlusTree::display() {
    BPlusNode* cursor = root;
    while (cursor && !cursor->isLeaf) cursor = cursor->children[0];
    cout << "B+ Tree Leaf Nodes: \n";
    while (cursor) {
        for (int key : cursor->keys) cout << key << " ";
        cout << "| ";
        cursor = cursor->next;
    }
    cout << endl;
}

void BPlusTree::rangeSearch(int start, int end) {
    BPlusNode* cursor = root;
    while (!cursor->isLeaf) {
        int i = 0;
        while (i < cursor->keys.size() && start >= cursor->keys[i]) i++;
        cursor = cursor->children[i];
    }

    cout << "Range Search [" << start << " ~ " << end << "]: ";
    while (cursor) {
        for (int key : cursor->keys) {
            if (key >= start && key <= end) cout << key << " ";
            else if (key > end) return;
        }
        cursor = cursor->next;
    }
    cout << endl;
}

int main() {
    BPlusTree tree;
    vector<int> vals = {10, 20, 5, 6, 12, 30, 7, 17};
    for (int v : vals) tree.insert(v);
    tree.display();

    tree.rangeSearch(7, 20);

    tree.remove(12);
    tree.remove(10);
    tree.display();

    return 0;
}
```

## 💡 5. B+ 트리가 사용되는 곳

- 📂 **데이터베이스 인덱스 (MySQL, PostgreSQL 등)**
- 📁 **파일 시스템 (NTFS, XFS 등)**
- 📈 **키-값 저장소 (RocksDB, LevelDB 등)**
- 📚 **사전, 사전형 검색**

---

## ✅ 요약

| 항목 | 설명 |
|------|------|
| B+ Tree | B-Tree의 확장 구조, 인덱스와 데이터를 분리 |
| 리프 노드 | 모든 데이터 저장, 연결 리스트로 구성 |
| 내부 노드 | 인덱스 전용 |
| 장점 | 범위 검색 우수, 인덱스 크기 작음 |
| 용도 | 데이터베이스, 파일 시스템 |
