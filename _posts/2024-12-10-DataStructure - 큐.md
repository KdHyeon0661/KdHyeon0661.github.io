---
layout: post
title: Data Structure - 큐
date: 2024-12-10 19:20:23 +0900
category: Data Structure
---
# 큐

## 큐란?

큐는 **선입선출(FIFO: First-In First-Out)** 구조입니다.  
먼저 들어온 데이터가 먼저 나갑니다.  
일상적인 예: 매표소 줄, 프린터 대기열, 작업 스케줄링.

- **push** : 데이터를 뒤쪽에 삽입
- **pop**  : 앞쪽 데이터를 제거
- **front**: 앞쪽 데이터를 조회 (제거하지 않음)
- **back** : 뒤쪽 데이터를 조회

---

## 기본 연산과 시간 복잡도

| 연산 | 설명 | 평균 시간 |
|------|------|-----------|
| `push(x)` | 뒤에 삽입 | O(1) |
| `pop()`   | 앞에서 제거 | O(1) |
| `front()` | 맨 앞 조회 | O(1) |
| `back()`  | 맨 뒤 조회 | O(1) |
| `empty()` | 비었는지 확인 | O(1) |

> 배열 기반 큐는 `pop`이 발생하면 앞쪽 공간이 낭비됩니다.  
> 이를 해결하기 위해 **원형 큐(circular queue)**를 사용합니다.

---

## 원형 배열 기반 큐 (고정 크기)

배열을 원형으로 사용하여 공간을 재활용합니다.  
`front`와 `rear` 인덱스를 % 연산으로 순환시킵니다.  
빈 상태와 가득 찬 상태를 구분하기 위해 **요소 개수(`count`)**를 따로 저장합니다.

```
초기 상태 (count=0, front=0, rear=0):
[ ][ ][ ][ ]

push 10 → rear=1, count=1:
[10][ ][ ][ ]

push 20 → rear=2, count=2:
[10][20][ ][ ]

pop → front=1, count=1:
[ ][20][ ][ ]
```

```cpp
#include <stdexcept>

class CircularQueue {
private:
    int* data;
    int capacity;
    int frontIdx;
    int rearIdx;
    int count;

public:
    CircularQueue(int cap = 8)
        : data(new int[cap]), capacity(cap), frontIdx(0), rearIdx(0), count(0) {}

    ~CircularQueue() { delete[] data; }

    bool empty() const { return count == 0; }
    bool full() const  { return count == capacity; }
    int size() const   { return count; }

    void push(int value) {
        if (full())
            throw std::overflow_error("Queue is full");
        data[rearIdx] = value;
        rearIdx = (rearIdx + 1) % capacity;
        ++count;
    }

    void pop() {
        if (empty())
            throw std::underflow_error("Queue is empty");
        frontIdx = (frontIdx + 1) % capacity;
        --count;
    }

    int front() const {
        if (empty())
            throw std::runtime_error("Queue is empty");
        return data[frontIdx];
    }

    int back() const {
        if (empty())
            throw std::runtime_error("Queue is empty");
        int idx = (rearIdx + capacity - 1) % capacity;
        return data[idx];
    }
};
```

- **장점**: 구현이 단순하고 메모리 효율이 좋습니다.
- **단점**: 크기가 고정되어 미리 예측해야 합니다.

---

## 연결 리스트 기반 큐

크기 제한이 없고, `push`와 `pop`이 항상 O(1)입니다.  
단, 각 노드의 동적 할당으로 인해 오버헤드가 있고 캐시 효율은 떨어집니다.

```cpp
#include <stdexcept>

template <typename T>
class ListQueue {
    struct Node {
        T data;
        Node* next;
        Node(const T& val, Node* nxt = nullptr) : data(val), next(nxt) {}
    };
    Node* head;  // 앞쪽 노드
    Node* tail;  // 뒤쪽 노드
    int count;

public:
    ListQueue() : head(nullptr), tail(nullptr), count(0) {}
    ~ListQueue() { while (!empty()) pop(); }

    bool empty() const { return head == nullptr; }
    int size() const { return count; }

    void push(const T& value) {
        Node* newNode = new Node(value);
        if (empty()) {
            head = tail = newNode;
        } else {
            tail->next = newNode;
            tail = newNode;
        }
        ++count;
    }

    void pop() {
        if (empty())
            throw std::underflow_error("Queue is empty");
        Node* temp = head;
        head = head->next;
        if (!head) tail = nullptr;
        delete temp;
        --count;
    }

    T& front() {
        if (empty())
            throw std::runtime_error("Queue is empty");
        return head->data;
    }

    T& back() {
        if (empty())
            throw std::runtime_error("Queue is empty");
        return tail->data;
    }
};
```

---

## 표준 라이브러리 `std::queue`

C++ 표준 라이브러리는 `std::queue` 컨테이너 어댑터를 제공합니다.  
기본적으로 `std::deque`를 내부 컨테이너로 사용하며, 필요에 따라 다른 컨테이너(`std::list` 등)로 변경할 수 있습니다.

```cpp
#include <queue>
#include <iostream>

int main() {
    std::queue<int> q;

    q.push(10);
    q.push(20);
    q.push(30);

    std::cout << "front: " << q.front() << '\n'; // 10
    std::cout << "back:  " << q.back() << '\n';  // 30

    q.pop(); // 10 제거
    std::cout << "front after pop: " << q.front() << '\n'; // 20

    return 0;
}
```

`std::queue`는 **반복자**를 제공하지 않습니다. 큐의 추상화를 유지하기 위해 앞과 뒤만 접근할 수 있습니다.

---

## 대표 응용: 너비 우선 탐색 (BFS)

그래프에서 최단 경로나 모든 노드를 레벨 순으로 탐색할 때 큐를 사용합니다.

```cpp
#include <queue>
#include <vector>
#include <iostream>

void bfs(int start, const std::vector<std::vector<int>>& graph) {
    std::vector<bool> visited(graph.size(), false);
    std::queue<int> q;

    visited[start] = true;
    q.push(start);

    while (!q.empty()) {
        int u = q.front();
        q.pop();
        std::cout << u << ' ';

        for (int v : graph[u]) {
            if (!visited[v]) {
                visited[v] = true;
                q.push(v);
            }
        }
    }
}
```

---

## 구현 선택 가이드

| 구현 방식 | 장점 | 단점 | 추천 상황 |
|-----------|------|------|-----------|
| 원형 배열 (고정) | 단순, 빠름, 캐시 효율 | 크기 제한 | 크기를 미리 알 수 있는 경우 |
| 연결 리스트 | 크기 제한 없음 | 노드 할당 오버헤드, 캐시 비효율 | 크기 변동이 심한 경우 |
| `std::queue` | 표준, 안전, 편리 | 내부 컨테이너에 의존 | 일반적인 대부분의 상황 |

---

## 주의사항

- `front()`나 `pop()` 호출 전에 반드시 `empty()`를 확인해야 합니다.
- 원형 큐에서 `full()`과 `empty()`를 구분하는 방법을 명확히 해야 합니다. (위 예제는 `count`를 사용)
- 멀티스레드 환경에서 `std::queue`를 그대로 사용하면 데이터 경쟁이 발생합니다. 별도의 동기화가 필요합니다.

---

## 마무리

큐는 **선입선출**이 필요한 모든 곳에 사용되는 기본 자료구조입니다.  
구현 방식에 따라 성능 특성이 다르므로, 상황에 맞게 선택하는 것이 중요합니다.  
초보자라면 `std::queue`로 시작하고, 내부 동작을 이해한 후 필요에 따라 직접 구현해보는 것을 추천합니다.