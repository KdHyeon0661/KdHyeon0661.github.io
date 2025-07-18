---
layout: post
title: Data Structure - 큐
date: 2024-12-10 19:20:23 +0900
category: Data Structure
---
# 큐(Queue) — 선입선출(FIFO) 구조의 이해

큐는 **선입선출(First In, First Out, FIFO)** 구조로 작동하는 선형 자료구조입니다.  
은행 대기열, 프린터 작업 큐, 프로세스 스케줄링 등 실생활에서 흔히 사용됩니다.

---

## 📌 1. 큐란?

- 먼저 들어간 데이터가 먼저 나오는 구조
- 데이터를 **뒤에서 추가(push)**하고, **앞에서 제거(pop)**

### 📍 대표 응용
- BFS (너비 우선 탐색)
- OS의 작업 스케줄러
- 비동기 이벤트 처리

---

## 📐 2. 큐의 주요 연산

| 연산 | 설명 |
|------|------|
| `push(x)` | 뒤에 요소 x 삽입 |
| `pop()` | 앞에서 요소 제거 |
| `front()` | 앞 요소 확인 |
| `back()` | 뒤 요소 확인 |
| `empty()` | 비었는지 확인 |
| `size()` | 요소 개수 확인 |

---

## 🧱 3. 배열 기반 큐 (비효율적인 구현)

```cpp
class QueueArray {
private:
    int* data;
    int capacity;
    int frontIdx;
    int rearIdx;
    int count;

public:
    QueueArray(int cap = 100) : capacity(cap), frontIdx(0), rearIdx(0), count(0) {
        data = new int[capacity];
    }

    ~QueueArray() {
        delete[] data;
    }

    void push(int val) {
        if (count == capacity)
            throw std::overflow_error("Queue Overflow");
        data[rearIdx++] = val;
        count++;
    }

    void pop() {
        if (empty())
            throw std::underflow_error("Queue Underflow");
        frontIdx++;
        count--;
    }

    int front() const {
        if (empty())
            throw std::runtime_error("Queue is empty");
        return data[frontIdx];
    }

    bool empty() const {
        return count == 0;
    }

    int size() const {
        return count;
    }
};
```

> ❗ 위 구현은 pop이 진행되면 **앞쪽 공간이 낭비**되므로, 실제 구현에서는 **원형 큐(Circular Queue)**를 사용합니다.

---

## 🔄 4. 원형 큐 구현

```cpp
class CircularQueue {
private:
    int* data;
    int capacity;
    int frontIdx;
    int rearIdx;
    int count;

public:
    CircularQueue(int cap = 100) : capacity(cap), frontIdx(0), rearIdx(0), count(0) {
        data = new int[capacity];
    }

    ~CircularQueue() {
        delete[] data;
    }

    void push(int val) {
        if (count == capacity)
            throw std::overflow_error("Queue Overflow");
        data[rearIdx] = val;
        rearIdx = (rearIdx + 1) % capacity;
        count++;
    }

    void pop() {
        if (empty())
            throw std::underflow_error("Queue Underflow");
        frontIdx = (frontIdx + 1) % capacity;
        count--;
    }

    int front() const {
        if (empty())
            throw std::runtime_error("Queue is empty");
        return data[frontIdx];
    }

    bool empty() const {
        return count == 0;
    }

    int size() const {
        return count;
    }
};
```

---

## 🔗 5. 연결 리스트 기반 큐 구현

```cpp
struct Node {
    int data;
    Node* next;
    Node(int val) : data(val), next(nullptr) {}
};

class QueueList {
private:
    Node* frontNode;
    Node* rearNode;
    int count;

public:
    QueueList() : frontNode(nullptr), rearNode(nullptr), count(0) {}

    ~QueueList() {
        while (!empty()) pop();
    }

    void push(int val) {
        Node* newNode = new Node(val);
        if (empty())
            frontNode = rearNode = newNode;
        else {
            rearNode->next = newNode;
            rearNode = newNode;
        }
        count++;
    }

    void pop() {
        if (empty())
            throw std::underflow_error("Queue Underflow");
        Node* temp = frontNode;
        frontNode = frontNode->next;
        delete temp;
        count--;
        if (frontNode == nullptr)
            rearNode = nullptr;
    }

    int front() const {
        if (empty())
            throw std::runtime_error("Queue is empty");
        return frontNode->data;
    }

    bool empty() const {
        return count == 0;
    }

    int size() const {
        return count;
    }
};
```

---

## ⚙️ 6. STL `std::queue` 사용 예시

```cpp
#include <queue>
#include <iostream>

std::queue<int> q;
q.push(10);
q.push(20);
std::cout << q.front();  // 10
q.pop();
std::cout << q.front();  // 20
```

> STL `std::queue`는 기본적으로 `deque`을 기반으로 동작하며, 내부 컨테이너를 변경할 수도 있습니다.

---

## 💡 7. 큐 응용: BFS (너비 우선 탐색)

```cpp
#include <queue>
#include <vector>
#include <iostream>

void bfs(int start, const std::vector<std::vector<int>>& graph, std::vector<bool>& visited) {
    std::queue<int> q;
    q.push(start);
    visited[start] = true;

    while (!q.empty()) {
        int cur = q.front(); q.pop();
        std::cout << cur << " ";
        for (int neighbor : graph[cur]) {
            if (!visited[neighbor]) {
                visited[neighbor] = true;
                q.push(neighbor);
            }
        }
    }
}
```

---

## 📊 8. 각 구현 비교

| 구현 방식 | 장점 | 단점 |
|-----------|------|------|
| 배열 (비원형) | 단순 | 메모리 낭비 |
| 원형 큐 | 공간 효율 | 포인터 연산 복잡 |
| 연결 리스트 | 크기 제한 없음 | 포인터 관리 필요 |
| STL `queue` | 편의성 최고 | 내부 제어 어려움 |

---

## ✅ 정리

- 큐는 선입선출(FIFO) 구조
- 배열 기반, 원형 큐, 연결 리스트 기반 구현 가능
- BFS, 이벤트 처리 등에 필수적으로 사용됨