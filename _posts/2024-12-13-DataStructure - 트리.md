---
layout: post
title: Data Structure - 트리
date: 2024-12-09 19:20:23 +0900
category: Data Structure
---
# 🌳 트리(Tree) 자료구조

트리는 계층적인 구조를 표현하는 데 매우 유용한 자료구조입니다.  
운영체제, 파일 시스템, 게임 엔진, 데이터베이스 인덱스, 인공지능 탐색 등 다양한 분야에서 사용됩니다.

---

## 📌 1. 트리(Tree)란?

> 트리는 **사이클이 없는 비선형 계층 구조**로, **노드(Node)**들이 **부모-자식 관계**로 연결된 구조입니다.

### 🧩 기본 용어

| 용어 | 설명 |
|------|------|
| 노드 (Node) | 데이터를 담는 기본 단위 |
| 루트 (Root) | 트리의 시작 노드 |
| 부모 (Parent) | 자식 노드를 갖는 노드 |
| 자식 (Child) | 부모로부터 연결된 노드 |
| 형제 (Sibling) | 같은 부모를 가진 노드 |
| 리프 (Leaf) | 자식이 없는 노드 |
| 서브트리 (Subtree) | 한 노드를 루트로 하는 트리 |
| 높이 (Height) | 루트에서 가장 먼 리프까지의 거리 |
| 깊이 (Depth) | 루트로부터 현재 노드까지의 거리 |

---

## 🌲 2. 트리의 종류

- **일반 트리 (General Tree)**: 자식 수 제한 없음
- **이진 트리 (Binary Tree)**: 자식이 최대 2개
- **이진 탐색 트리 (BST)**: 왼쪽 < 루트 < 오른쪽
- **완전 이진 트리 (Complete Binary Tree)**
- **균형 이진 트리 (Balanced Binary Tree)**: AVL, Red-Black Tree
- **힙 (Heap)**: 우선순위 기반 완전 이진 트리
- **트라이 (Trie)**: 문자열 검색 특화

> 📌 이번 글에서는 자식 수에 제한이 없는 **일반 트리(General Tree)**의 두 가지 대표적인 구현 방법을 소개합니다.

---

## 🛠️ 3. 일반 트리 구현 방식

일반 트리는 보통 다음 두 가지 방식으로 구현됩니다:

| 방식 | 설명 | 장점 |
|------|------|------|
| `vector<TreeNode*>` 기반 | 자식 노드들을 vector로 저장 | 직관적, STL 활용 가능 |
| `firstChild` / `nextSibling` 기반 | 트리를 이진 트리처럼 변환하여 표현 | 메모리 절약, 순회 최적화 가능 |

---

## ✅ 4. vector 기반 일반 트리 구현

```cpp
#include <iostream>
#include <vector>
using namespace std;

struct TreeNode {
    int data;
    vector<TreeNode*> children;

    TreeNode(int val) : data(val) {}
};

void printTree(TreeNode* node, int depth = 0) {
    if (!node) return;
    cout << string(depth * 2, '-') << node->data << endl;
    for (TreeNode* child : node->children) {
        printTree(child, depth + 1);
    }
}

int main() {
    TreeNode* root = new TreeNode(1);
    TreeNode* child1 = new TreeNode(2);
    TreeNode* child2 = new TreeNode(3);
    TreeNode* child3 = new TreeNode(4);
    TreeNode* grandchild = new TreeNode(5);

    root->children.push_back(child1);
    root->children.push_back(child2);
    child2->children.push_back(child3);
    child3->children.push_back(grandchild);

    printTree(root);
    return 0;
}
```

### 🔽 출력 예시
```
1
--2
--3
----4
------5
```

---

## ✅ 5. Child-Sibling 기반 일반 트리 구현

```cpp
#include <iostream>
using namespace std;

// 연결 리스트의 노드를 정의
struct ChildNode {
    struct TreeNode* child;
    ChildNode* next;
};

// 트리 노드 정의
struct TreeNode {
    int data;
    ChildNode* firstChild;
};

// 새로운 트리 노드 생성 함수
TreeNode* createTreeNode(int data) {
    TreeNode* node = new TreeNode;
    node->data = data;
    node->firstChild = nullptr;
    return node;
}

// 자식 노드 추가 함수
void addChild(TreeNode* parent, TreeNode* child) {
    ChildNode* newChild = new ChildNode;
    newChild->child = child;
    newChild->next = nullptr;

    if (parent->firstChild == nullptr) {
        parent->firstChild = newChild;
    } else {
        ChildNode* temp = parent->firstChild;
        while (temp->next != nullptr) {
            temp = temp->next;
        }
        temp->next = newChild;
    }
}

// 트리 구조 출력 함수 (깊이값 이용, 들여쓰기)
void printTree(TreeNode* node, int depth = 0) {
    for (int i = 0; i < depth; ++i) cout << "--";
    cout << node->data << "\n";
    ChildNode* curr = node->firstChild;
    while (curr != nullptr) {
        printTree(curr->child, depth + 1);
        curr = curr->next;
    }
}

int main() {
    // root 및 1번째 레벨 자식 4개 생성
    TreeNode* root = createTreeNode(1);
    TreeNode* child1 = createTreeNode(2);
    TreeNode* child2 = createTreeNode(3);
    TreeNode* child3 = createTreeNode(4);
    TreeNode* child4 = createTreeNode(5);

    addChild(root, child1);
    addChild(root, child2);
    addChild(root, child3);
    addChild(root, child4);

    // child1의 자식 두 개
    TreeNode* child1_1 = createTreeNode(6);
    TreeNode* child1_2 = createTreeNode(7);
    addChild(child1, child1_1);
    addChild(child1, child1_2);

    // child2의 자식 두 개
    TreeNode* child2_1 = createTreeNode(8);
    TreeNode* child2_2 = createTreeNode(9);
    addChild(child2, child2_1);
    addChild(child2, child2_2);

    // 트리 구조 출력
    printTree(root);

    return 0;
}
```

### 🔽 출력 예시
```
1
--2
----6
----7
--3
----8
----9
--4
--5
```

---

## 🧠 6. 일반 트리의 응용 분야

- **파일 시스템**: 디렉토리 구조
- **컴파일러**: 문법 분석 트리(AST)
- **UI 시스템**: 계층적 메뉴 및 씬 그래프
- **조직도 / 계보도**: 계층 구조 표현

---

## ✅ 마무리 요약

| 키워드 | 요약 |
|--------|------|
| 일반 트리 | 자식 수 제한 없는 트리 |
| vector 방식 | 구현이 직관적이며 STL 활용 가능 |
| Child-Sibling 방식 | 포인터 두 개로 메모리 절약 |
| 출력 | DFS 순회로 출력 (재귀 기반) |
| 응용 | 파일 시스템, UI, AI, 컴파일러 등 |