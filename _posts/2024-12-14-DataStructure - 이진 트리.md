---
layout: post
title: Data Structure - 이진 트리
date: 2024-12-09 19:20:23 +0900
category: Data Structure
---
# 🌳 이진 트리 & 이진 탐색 트리(BST)

이진 트리는 노드당 최대 2개의 자식을 가지는 구조로, 다양한 트리 기반 알고리즘의 핵심입니다.  
그 중 **이진 탐색 트리(BST)**는 효율적인 탐색/삽입/삭제가 가능하여 널리 사용됩니다.

---

## 📌 1. 이진 트리란?

> **이진 트리(Binary Tree)**는 각 노드가 **최대 2개의 자식 노드**(왼쪽, 오른쪽)를 갖는 트리입니다.

---

## 🌲 2. 이진 트리의 종류

| 분류 | 설명 |
|------|------|
| **완전 이진 트리 (Complete Binary Tree)** | 마지막 레벨을 제외한 모든 레벨이 노드로 가득 차 있으며, 마지막 레벨은 왼쪽부터 차례로 채워짐 |
| **포화 이진 트리 (Full Binary Tree)** | 모든 노드가 **0개 또는 2개의 자식**만 가짐 (즉, 자식이 1개인 노드는 없음) |
| **정 포화 이진 트리 (Perfect Binary Tree)** | 모든 리프가 같은 깊이에 있으며, 모든 내부 노드가 2개의 자식을 가짐 (완전 + 포화의 결합 형태) |

### 📝 예시 비교

```
        1
       / \
      2   3
     / \ / \
    4  5 6  7

→ 완전 이진 트리 ✅ 
→ 포화 이진 트리 ✅  
→ 정 포화 이진 트리 ✅
```

```
        1
       / \
      2   3
     / \
    4  5

→ 완전 ✅ (왼쪽부터 순차적으로 채워져 있으며, 마지막 레벨을 제외하면 노드로 가득 차 있음)
→ 포화 ✅ (3은 자식 없음, 2는 2개 → OK / 불균형 아님)
→ 정 포화 ❌ (리프 깊이가 다름)
```

---

## 🔄 3. 이진 트리 순회(Tree Traversal)

트리는 **선형이 아니기 때문에** 순회 방법에 따라 방문 순서가 달라집니다. 대표적인 순회 방법은 다음 세 가지입니다:

### ✅ 전위 순회 (Preorder): Root → Left → Right

```cpp
void preorder(Node* node) {
    if (!node) return;
    cout << node->data << " ";
    preorder(node->left);
    preorder(node->right);
}
```

### ✅ 중위 순회 (Inorder): Left → Root → Right

```cpp
void inorder(Node* node) {
    if (!node) return;
    inorder(node->left);
    cout << node->data << " ";
    inorder(node->right);
}
```

### ✅ 후위 순회 (Postorder): Left → Right → Root

```cpp
void postorder(Node* node) {
    if (!node) return;
    postorder(node->left);
    postorder(node->right);
    cout << node->data << " ";
}
```

---

### 4. 이진 트리 예시 구성

```cpp
int main() {
    Node* root = new Node(1);
    root->left = new Node(2);
    root->right = new Node(3);
    root->left->left = new Node(4);
    root->left->right = new Node(5);

    cout << "Preorder: "; preorder(root); cout << endl;
    cout << "Inorder: "; inorder(root); cout << endl;
    cout << "Postorder: "; postorder(root); cout << endl;

    return 0;
}
```

### 🔽 출력 결과
```
Preorder: 1 2 4 5 3  
Inorder: 4 2 5 1 3  
Postorder: 4 5 2 3 1  
```

---

## 🌳 4. 이진 탐색 트리 (BST: Binary Search Tree)

> BST는 이진 트리의 일종으로 **왼쪽 서브트리는 루트보다 작고**, **오른쪽 서브트리는 루트보다 큼**을 만족합니다.

### ⚙️ 삽입

```cpp
Node* insert(Node* root, int val) {
    if (!root) return new Node(val);
    if (val < root->data)
        root->left = insert(root->left, val);
    else if (val > root->data)
        root->right = insert(root->right, val);
    return root;
}
```

### 🔍 탐색

```cpp
bool search(Node* root, int val) {
    if (!root) return false;
    if (val == root->data) return true;
    return val < root->data ? search(root->left, val) : search(root->right, val);
}
```

---

## ❌ 5. 삭제 (Delete) 연산

BST에서 삭제는 총 3가지 경우로 나뉩니다:

1. **삭제할 노드가 리프(leaf)**: 그냥 제거
2. **자식이 1개인 경우**: 자식 노드를 끌어올림
3. **자식이 2개인 경우**:  
   → **중위 후속 노드(오른쪽 서브트리의 최솟값)**로 대체 후 그 노드 삭제(오른쪽 서브트리에서 가장 작은 값은 가장 왼족에 있기에)
   → 그렇기에 **중위 선행 노드(왼쪽 서브트리의 최댓값)**를 사용해도 문제가 되지 않음

### 🧩 구현 코드

```cpp
Node* findMin(Node* node) {
    while (node && node->left)
        node = node->left;
    return node;
}

Node* deleteNode(Node* root, int val) {
    if (!root) return nullptr;

    if (val < root->data)
        root->left = deleteNode(root->left, val);
    else if (val > root->data)
        root->right = deleteNode(root->right, val);
    else {
        // 1. 리프 또는 1자식
        if (!root->left) {
            Node* temp = root->right;
            delete root;
            return temp;
        }
        else if (!root->right) {
            Node* temp = root->left;
            delete root;
            return temp;
        }

        // 2. 자식이 둘 다 있는 경우
        Node* successor = findMin(root->right);
        root->data = successor->data;
        root->right = deleteNode(root->right, successor->data);
    }
    return root;
}
```

---

## 🔬 6. 예시 - 삽입, 삭제, 순회

```cpp
int main() {
    Node* root = nullptr;
    int values[] = { 8, 3, 10, 1, 6, 14, 4, 7 };

    for (int v : values)
        root = insert(root, v);

    cout << "Inorder before delete: ";
    inorder(root); cout << endl;

    root = deleteNode(root, 3);  // 노드 3 삭제 (자식 2개)

    cout << "Inorder after delete: ";
    inorder(root); cout << endl;

    return 0;
}
```

### 🔽 출력 결과
```
Inorder before delete: 1 3 4 6 7 8 10 14  
Inorder after delete: 1 4 6 7 8 10 14  
```

---

## 🧠 7. 시간 복잡도 비교

| 연산 | 평균 시간 | 최악 시간 (편향 트리) |
|------|------------|------------------------|
| 삽입 | O(log n)   | O(n)                   |
| 탐색 | O(log n)   | O(n)                   |
| 삭제 | O(log n)   | O(n)                   |

> 🔸 **완전/포화 이진 트리** 구조를 유지할 수 있다면 성능을 **O(log n)**으로 보장할 수 있습니다.  
> 🔸 이를 위해 AVL, Red-Black Tree 등의 **자기 균형 트리**가 사용됩니다.

---

## ✅ 정리

| 키워드 | 요약 |
|--------|------|
| 완전 이진 트리 | 마지막 레벨을 제외하고 가득 차 있음 |
| 포화 이진 트리 | 모든 노드가 0 또는 2개의 자식 |
| BST | 왼쪽 < 루트 < 오른쪽 |
| 삭제 | 리프 / 1자식 / 2자식에 따라 처리 방식 다름 |
| 순회 | 전위 / 중위 / 후위 |