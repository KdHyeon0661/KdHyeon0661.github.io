---
layout: post
title: Data Structure - 연결 리스트 고급
date: 2024-12-09 20:20:23 +0900
category: Data Structure
---
# 특수한 연결 리스트들

앞서 단일(Singly), 이중(Doubly), 원형(Circular) 연결 리스트를 살펴봤습니다.  
이번 글에서는 **보다 고급이거나 특수한 연결 리스트 구조들**을 소개합니다.

---

## 1. 다중 연결 리스트 (Multi-level / Multi-pointer Linked List)

### 구조

노드가 **2개 이상의 방향(포인터)**으로 연결되는 리스트입니다. 주로 **2차원 이상의 데이터**를 표현할 때 사용됩니다.

예:  
- 표 형태 데이터 (스프레드시트)
- 트리의 리스트 표현
- 지형 맵, 마인크래프트 월드 구성 등

### 구조 예시 (2D 연결 리스트)

```cpp
struct Node {
    int data;
    Node* right;
    Node* down;
};
```

### 사용 예

```cpp
Node* createMatrix(int rows, int cols) {
    Node* head = nullptr;
    std::vector<Node*> prevRow(cols, nullptr);

    for (int i = 0; i < rows; ++i) {
        Node* rowHead = nullptr;
        Node* prev = nullptr;
        for (int j = 0; j < cols; ++j) {
            Node* newNode = new Node{i * cols + j, nullptr, nullptr};
            if (!rowHead) rowHead = newNode;
            if (prev) prev->right = newNode;
            if (i > 0) prevRow[j]->down = newNode;
            prevRow[j] = newNode;
            prev = newNode;
        }
        if (!head) head = rowHead;
    }
    return head;
}
```

---

## 2. 스킵 리스트 (Skip List)

### 개념

정렬된 연결 리스트에 **다단계 포인터를 추가**하여, 이진 탐색처럼 빠르게 탐색이 가능한 구조입니다.  
삽입/삭제/탐색 모두 평균적으로 `O(log n)` 시간에 수행됩니다.

> 정렬된 데이터가 많은 경우, 트리보다 구현이 간단하면서도 높은 성능을 기대할 수 있습니다.

### 구조

각 노드는 **다수의 포인터**(다음 레벨 포인터 포함)를 가집니다.

```cpp
struct SkipNode {
    int val;
    std::vector<SkipNode*> forward;
    SkipNode(int level, int val) : val(val), forward(level + 1, nullptr) {}
};
```

### 간단한 개념도

```
Level 2:  A ------------> E
Level 1:  A --> B --> C --> E
Level 0:  A -> B -> C -> D -> E
```

### 장점

- 동적 트리 구조 없이도 `O(log n)` 탐색 가능
- 해시 테이블보다 순차 접근이 용이

### 실사용 예

- Redis 내부에서 스코어 정렬을 위해 사용
- MemTable (LSM-tree 기반 DB)

---

## 3. XOR 연결 리스트 (XOR Linked List)

### 개념

**prev**와 **next** 포인터를 하나로 줄여 메모리를 절약하려는 특수 구조입니다.

- `node->npx = prev ^ next` (XOR 연산 사용)
- 단방향 탐색만 가능하며, 이전 주소를 반드시 기억하고 있어야 다음 노드를 알 수 있음

### 구조

```cpp
struct Node {
    int data;
    Node* npx;  // XOR(prev, next)
};
```

### 주의 사항

- 실제 사용은 드뭅니다
- 디버깅 및 유지보수 매우 어려움
- C++에선 `uintptr_t` 등을 사용해야 안전하게 구현 가능

---

## 4. 센티넬 노드를 가진 연결 리스트 (Sentinel Nodes)

### 개념

**헤드/테일 노드에 더미 노드**를 미리 생성해두어, 삽입/삭제 로직을 단순화하는 테크닉입니다.

### 장점

- 엣지 케이스를 줄임 (빈 리스트, 머리/꼬리 삽입 등)
- 항상 중간 삽입처럼 처리 가능

### 구조

```cpp
struct Node {
    int data;
    Node* prev;
    Node* next;
};

struct List {
    Node* sentinel;
    List() {
        sentinel = new Node{-1, nullptr, nullptr};
        sentinel->next = sentinel->prev = sentinel;
    }
};
```

---

## 5. Unrolled Linked List (분해된 연결 리스트)

### 개념

각 노드가 데이터를 하나만 가지지 않고 **여러 개의 요소(작은 배열)**를 저장하는 구조입니다.

### 특징

- **캐시 친화적**
- 일반 연결 리스트보다 공간 효율과 접근 효율이 좋음

```cpp
struct UnrolledNode {
    std::vector<int> values;
    UnrolledNode* next;
};
```

### 사용 예

- Python의 deque 내부 구현
- 일부 텍스트 편집기 (줄 단위 캐시 저장)

---

## 6. Self-referential / Looping List

### 개념

노드가 자기 자신을 참조하거나 순환을 형성한 리스트입니다.  
이는 일반적인 원형 리스트와 달리 **특정 조건 하에 일부 노드만 루프를 형성**할 수 있습니다.

### 주 용도

- 루프 탐지 알고리즘 (Floyd’s cycle detection 등)
- 메모리 검사 테스트

---

## 🔚 정리

| 이름 | 특징 | 사용 용도 |
|------|------|-----------|
| 다중 연결 리스트 | 노드가 상/하/좌/우 등 다방향 연결 | 2D 데이터, 지형 등 |
| 스킵 리스트 | 빠른 탐색용 계층형 리스트 | Redis 등 DB 내부 |
| XOR 리스트 | 포인터 하나로 prev/next 동시 표현 | 이론적 흥미, 메모리 최적화 |
| 센티넬 리스트 | 더미 노드로 삽입/삭제 단순화 | 안정성 있는 구현 |
| Unrolled 리스트 | 한 노드에 여러 값 저장 | 캐시 최적화 |
| 루프 리스트 | 순환 검출 목적 | 알고리즘 테스트 |