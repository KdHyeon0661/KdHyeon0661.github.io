---
layout: post
title: Data Structure - 연결 리스트
date: 2024-12-09 19:20:23 +0900
category: Data Structure
---
# 연결 리스트 (Linked List)

## 1. 연결 리스트란?

**연결 리스트(Linked List)**는 데이터를 담은 노드들이 **포인터로 서로 연결된 구조**입니다. 배열처럼 연속된 메모리 공간이 아닌, **동적으로 연결된 노드들**로 이루어져 있어 **삽입과 삭제가 빠른** 장점이 있습니다.

---

## 2. 단일 연결 리스트 (Singly Linked List)

### 구조

각 노드는 **데이터 + 다음 노드를 가리키는 포인터**로 구성됩니다.

```cpp
struct Node {
    int data;
    Node* next;
};
```

### 기본 동작 구현

```cpp
#include <iostream>

struct Node {
    int data;
    Node* next;
};

void insertFront(Node*& head, int val) {
    Node* newNode = new Node{val, head};
    head = newNode;
}

void printList(Node* head) {
    Node* cur = head;
    while (cur) {
        std::cout << cur->data << " -> ";
        cur = cur->next;
    }
    std::cout << "NULL\n";
}

void deleteList(Node*& head) {
    Node* cur = head;
    while (cur) {
        Node* temp = cur;
        cur = cur->next;
        delete temp;
    }
    head = nullptr;
}
```

### 시간 복잡도

| 연산 | 시간 |
|------|------|
| 앞에 삽입 | O(1) |
| 중간/끝 삽입 | O(n) |
| 삭제 | O(n) |
| 검색 | O(n) |

---

## 3. 이중 연결 리스트 (Doubly Linked List)

### 구조

각 노드가 **이전 노드(prev)**와 **다음 노드(next)**를 모두 가리킵니다.

```cpp
struct DNode {
    int data;
    DNode* prev;
    DNode* next;
};
```

### 예제 구현

```cpp
#include <iostream>

struct DNode {
    int data;
    DNode* prev;
    DNode* next;
};

void insertFront(DNode*& head, int val) {
    DNode* newNode = new DNode{val, nullptr, head};
    if (head) head->prev = newNode;
    head = newNode;
}

void printList(DNode* head) {
    DNode* cur = head;
    while (cur) {
        std::cout << cur->data << " <-> ";
        cur = cur->next;
    }
    std::cout << "NULL\n";
}

void deleteList(DNode*& head) {
    DNode* cur = head;
    while (cur) {
        DNode* temp = cur;
        cur = cur->next;
        delete temp;
    }
    head = nullptr;
}
```

### 장점

- 양방향 이동 가능
- 노드 삭제 시 이전 노드를 알기 때문에 빠르게 제거 가능

---

## 4. 원형 연결 리스트 (Circular Linked List)

### 구조

리스트의 **마지막 노드가 첫 번째 노드를 가리키는 구조**입니다.

#### 단일 원형 리스트

```cpp
struct CNode {
    int data;
    CNode* next;
};
```

```cpp
#include <iostream>

struct CNode {
    int data;
    CNode* next;
};

void insertEnd(CNode*& tail, int val) {
    CNode* newNode = new CNode{val, nullptr};
    if (!tail) {
        newNode->next = newNode;
        tail = newNode;
    } else {
        newNode->next = tail->next;
        tail->next = newNode;
        tail = newNode;
    }
}

void printList(CNode* tail) {
    if (!tail) return;
    CNode* cur = tail->next;
    do {
        std::cout << cur->data << " -> ";
        cur = cur->next;
    } while (cur != tail->next);
    std::cout << "(head)\n";
}

void deleteList(CNode*& tail) {
    if (!tail) return;
    CNode* head = tail->next;
    CNode* cur = head;
    do {
        CNode* temp = cur;
        cur = cur->next;
        delete temp;
    } while (cur != head);
    tail = nullptr;
}
```

---

## 5. 비교 요약

| 종류 | 양방향 | 원형 | 장점 | 단점 |
|------|--------|------|------|------|
| 단일 연결 리스트 | X | X | 구조 간단, 삽입 빠름 | 삭제 시 이전 노드 추적 필요 |
| 이중 연결 리스트 | O | X | 앞뒤 이동 가능, 삭제 용이 | 메모리 추가 사용 |
| 원형 연결 리스트 | X/O | O | 꼬리 → 머리로 순환 가능 | 끝 판단이 어려움 |

---

## 6. 사용 사례

- 스택/큐 기반 구현 (연결 리스트로도 구현 가능)
- LRU 캐시 (이중 연결 리스트 + 해시)
- 음악 재생 목록, 라운드 로빈 스케줄러 (원형 리스트)