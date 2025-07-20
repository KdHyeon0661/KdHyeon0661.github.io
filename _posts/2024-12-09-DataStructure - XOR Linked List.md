---
layout: post
title: Data Structure - XOR Linked List
date: 2024-12-09 21:20:23 +0900
category: Data Structure
---
# XOR Linked List

## 1. XOR Linked List란?

**XOR 연결 리스트**는 이중 연결 리스트에서 **prev, next 포인터를 XOR 연산으로 하나로 통합**한 구조입니다.

> 즉, `node->npx = prev ^ next`로 표현되며, 공간을 절약할 수 있습니다.

---

## 2. 구조 개념

### 일반 이중 리스트

```
[prev] <- Node -> [next]
```

### XOR 리스트

```
[npx = prev ⊕ next]
```

> 이전 노드를 알고 있어야 다음 노드를 계산할 수 있습니다.

---

## 3. C++ 구조 정의

```cpp
#include <iostream>
#include <cstdint>

struct Node {
    int data;
    Node* npx; // XOR(prev, next)
    Node(int val) : data(val), npx(nullptr) {}
};

Node* XOR(Node* a, Node* b) {
    return reinterpret_cast<Node*>(
        reinterpret_cast<uintptr_t>(a) ^ reinterpret_cast<uintptr_t>(b)
    );
}
```

---

## 4. 삽입 구현 (앞에 삽입)

```cpp
void insertFront(Node*& head, int val) {
    Node* newNode = new Node(val);
    newNode->npx = XOR(nullptr, head);

    if (head != nullptr) {
        Node* next = XOR(nullptr, head->npx);
        head->npx = XOR(newNode, next);
    }

    head = newNode;
}
```

---

## 5. 출력 함수 (순방향)

```cpp
void printList(Node* head) {
    Node* cur = head;
    Node* prev = nullptr;
    Node* next;

    while (cur != nullptr) {
        std::cout << cur->data << " ";
        next = XOR(prev, cur->npx);
        prev = cur;
        cur = next;
    }
    std::cout << "\n";
}
```

---

## 6. 장단점 비교

| 장점 | 단점 |
|------|------|
| 포인터 수 절감 (메모리 절약) | 코드가 복잡하고 디버깅 어려움 |
| 메모리 제약 환경에서 유리 | 일반 포인터 방식보다 비직관적 |
| 이론적으로 흥미로움 | 실무에서는 거의 사용되지 않음 |

---

## 7. 사용 사례

- **이론적 연구/문제 풀이용**
- **경량 임베디드 시스템**의 메모리 절약이 필요한 경우
- **학습 목적**의 자료구조 설계

---

## 🔒 주의 사항

- 포인터 연산을 비트 연산으로 다루므로, **C/C++에서만 가능**
- 타입 변환 시 **`uintptr_t` 사용 필수**
- 실전 사용보다는 **흥미롭고 창의적인 연습용 자료구조**
