---
layout: post
title: Data Structure - 덱과 우선순위 큐
date: 2024-12-11 19:20:23 +0900
category: Data Structure
---
# 덱(Deque)과 우선순위 큐(Priority Queue) — 큐의 진화형 구조들

큐(Queue)는 선입선출(FIFO) 구조이지만, 다양한 응용에 맞춰 여러 확장 구조가 등장했습니다.  
이번 글에서는 **덱(Deque)**과 **우선순위 큐(Priority Queue)**를 중심으로 구조적 차이와 구현, 활용법을 살펴봅니다.

---

## 🧭 1. 덱(Deque)이란?

> **Deque**는 **Double-Ended Queue**의 약자로, **양쪽 끝에서 삽입과 삭제가 모두 가능한 큐**입니다.

| 연산 | 설명 |
|------|------|
| `push_front(x)` | 앞쪽에 요소 삽입 |
| `push_back(x)`  | 뒤쪽에 요소 삽입 |
| `pop_front()`   | 앞에서 제거 |
| `pop_back()`    | 뒤에서 제거 |
| `front()`       | 앞 요소 확인 |
| `back()`        | 뒤 요소 확인 |

### 📦 STL `std::deque` 예시

```cpp
#include <deque>
#include <iostream>

std::deque<int> dq;
dq.push_back(10);
dq.push_front(20);   // dq = 20, 10
dq.pop_back();       // dq = 20
std::cout << dq.front();  // 20
```

### ✅ 특징

- 양방향 입출력
- 내부적으로 **블록 기반 동적 배열** 사용 (일반 배열보다 유연)
- `std::deque`는 `std::vector`보다 삽입/삭제가 더 유리할 수 있음

---

## ⚙️ 2. 직접 덱 구현 (연결 리스트 기반)

```cpp
struct Node {
    int data;
    Node* prev;
    Node* next;
    Node(int val) : data(val), prev(nullptr), next(nullptr) {}
};

class Deque {
private:
    Node* frontNode;
    Node* backNode;
    int count;

public:
    Deque() : frontNode(nullptr), backNode(nullptr), count(0) {}

    ~Deque() {
        while (!empty()) pop_front();
    }

    void push_front(int val) {
        Node* newNode = new Node(val);
        newNode->next = frontNode;
        if (frontNode) frontNode->prev = newNode;
        frontNode = newNode;
        if (!backNode) backNode = newNode;
        count++;
    }

    void push_back(int val) {
        Node* newNode = new Node(val);
        newNode->prev = backNode;
        if (backNode) backNode->next = newNode;
        backNode = newNode;
        if (!frontNode) frontNode = newNode;
        count++;
    }

    void pop_front() {
        if (empty()) throw std::runtime_error("Empty deque");
        Node* temp = frontNode;
        frontNode = frontNode->next;
        if (frontNode) frontNode->prev = nullptr;
        else backNode = nullptr;
        delete temp;
        count--;
    }

    void pop_back() {
        if (empty()) throw std::runtime_error("Empty deque");
        Node* temp = backNode;
        backNode = backNode->prev;
        if (backNode) backNode->next = nullptr;
        else frontNode = nullptr;
        delete temp;
        count--;
    }

    int front() const {
        if (empty()) throw std::runtime_error("Empty deque");
        return frontNode->data;
    }

    int back() const {
        if (empty()) throw std::runtime_error("Empty deque");
        return backNode->data;
    }

    bool empty() const { return count == 0; }

    int size() const { return count; }
};
```

---

## 🚀 3. 우선순위 큐(Priority Queue)

> 일반 큐는 선입선출이지만, 우선순위 큐는 **우선순위가 높은 요소를 먼저 꺼냅니다.**

### 🧮 동작 방식

- 내부적으로 **힙(heap)** 구조 사용 (대부분 이진 최대 힙)
- 가장 큰 값이 항상 루트에 위치

### 🧪 STL `std::priority_queue` 기본 사용

```cpp
#include <queue>
#include <vector>
#include <iostream>

std::priority_queue<int> pq;
pq.push(10);
pq.push(20);
pq.push(5);

std::cout << pq.top();  // 20
pq.pop();               // 제거 후 top = 10
```

### 🧭 커스텀 우선순위 (작은 수가 먼저 나오게)

```cpp
// 최소 힙으로 만들기
std::priority_queue<int, std::vector<int>, std::greater<int>> minHeap;
minHeap.push(5);
minHeap.push(1);
minHeap.push(7);
// top = 1
```

---

## ⚙️ 4. 직접 최대 힙 기반 우선순위 큐 구현

```cpp
class PriorityQueue {
private:
    std::vector<int> heap;

    void heapify_up(int idx) {
        while (idx > 0 && heap[idx] > heap[(idx - 1) / 2]) {
            std::swap(heap[idx], heap[(idx - 1) / 2]);
            idx = (idx - 1) / 2;
        }
    }

    void heapify_down(int idx) {
        int size = heap.size();
        while (2 * idx + 1 < size) {
            int left = 2 * idx + 1;
            int right = 2 * idx + 2;
            int largest = idx;

            if (left < size && heap[left] > heap[largest]) largest = left;
            if (right < size && heap[right] > heap[largest]) largest = right;
            if (largest == idx) break;
            std::swap(heap[idx], heap[largest]);
            idx = largest;
        }
    }

public:
    void push(int val) {
        heap.push_back(val);
        heapify_up(heap.size() - 1);
    }

    void pop() {
        if (heap.empty()) throw std::runtime_error("Empty queue");
        heap[0] = heap.back();
        heap.pop_back();
        heapify_down(0);
    }

    int top() const {
        if (heap.empty()) throw std::runtime_error("Empty queue");
        return heap[0];
    }

    bool empty() const {
        return heap.empty();
    }

    int size() const {
        return heap.size();
    }
};
```

---

## 📊 5. 비교: `deque` vs `priority_queue`

| 항목 | `deque` | `priority_queue` |
|------|---------|------------------|
| 구조 | 양끝 입출력 | 우선순위 기반 정렬 |
| 내부 구현 | 블록 배열 | 힙(Heap) |
| 삽입/삭제 | 앞뒤 가능 | 정렬 기준 삽입/삭제 |
| 접근 | 양쪽 끝 | top만 확인 가능 |
| STL 컨테이너 | `std::deque` | `std::priority_queue` |
| 주요 용도 | 슬라이딩 윈도우, 캐시 | 작업 스케줄링, 탐욕 알고리즘 |

---

## ✅ 마무리

- **Deque**: 앞뒤 모두 삽입/삭제 가능한 유연한 큐
- **Priority Queue**: 우선순위가 높은 데이터부터 꺼내는 큐
- STL에서 강력한 도구로 제공되며, 내부 원리를 직접 구현해보면 훨씬 깊이 이해할 수 있음