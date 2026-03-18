---
layout: post
title: Data Structure - 덱과 우선순위 큐
date: 2024-12-11 19:20:23 +0900
category: Data Structure
---
# 덱과 우선순위 큐

## 덱 (Deque)

### 덱이란?

덱은 **Double-Ended Queue**의 줄임말로, **앞과 뒤 양쪽에서 삽입과 삭제가 모두 가능한 큐**입니다.

- `push_front`: 앞에 삽입
- `push_back`: 뒤에 삽입
- `pop_front`: 앞에서 제거
- `pop_back`: 뒤에서 제거
- `front`, `back`: 앞/뒤 원소 조회

### 주요 연산과 시간 복잡도

| 연산 | 설명 | 평균 시간 |
|------|------|-----------|
| `push_front(x)` | 앞에 삽입 | O(1) |
| `push_back(x)`  | 뒤에 삽입 | O(1) |
| `pop_front()`   | 앞에서 제거 | O(1) |
| `pop_back()`    | 뒤에서 제거 | O(1) |
| `front()`       | 앞 원소 조회 | O(1) |
| `back()`        | 뒤 원소 조회 | O(1) |
| `empty()`       | 비었는지 확인 | O(1) |

### 구현 방식

#### 1. 원형 배열 기반 (고정 크기)

배열을 원형으로 사용하여 앞과 뒤 인덱스를 관리합니다.  
크기가 고정되어 있으므로 꽉 찼을 때는 더 이상 삽입할 수 없습니다.

```cpp
#include <stdexcept>

class CircularDeque {
private:
    int* data;
    int capacity;
    int frontIdx;
    int rearIdx;
    int count;

public:
    CircularDeque(int cap = 8)
        : data(new int[cap]), capacity(cap), frontIdx(0), rearIdx(0), count(0) {}

    ~CircularDeque() { delete[] data; }

    bool empty() const { return count == 0; }
    bool full() const { return count == capacity; }
    int size() const { return count; }

    void push_front(int value) {
        if (full()) throw std::overflow_error("Deque full");
        frontIdx = (frontIdx - 1 + capacity) % capacity;
        data[frontIdx] = value;
        ++count;
    }

    void push_back(int value) {
        if (full()) throw std::overflow_error("Deque full");
        data[rearIdx] = value;
        rearIdx = (rearIdx + 1) % capacity;
        ++count;
    }

    void pop_front() {
        if (empty()) throw std::underflow_error("Deque empty");
        frontIdx = (frontIdx + 1) % capacity;
        --count;
    }

    void pop_back() {
        if (empty()) throw std::underflow_error("Deque empty");
        rearIdx = (rearIdx - 1 + capacity) % capacity;
        --count;
    }

    int& front() {
        if (empty()) throw std::runtime_error("Deque empty");
        return data[frontIdx];
    }

    int& back() {
        if (empty()) throw std::runtime_error("Deque empty");
        int idx = (rearIdx - 1 + capacity) % capacity;
        return data[idx];
    }
};
```

#### 2. 이중 연결 리스트 기반

각 노드가 `prev`와 `next` 포인터를 가지며, 양끝 삽입/삭제가 모두 O(1)입니다.  
크기 제한이 없고 구현이 비교적 간단하지만, 노드 할당 오버헤드와 캐시 비효율이 있습니다.

```cpp
#include <stdexcept>

template <typename T>
class DequeList {
    struct Node {
        T data;
        Node* prev;
        Node* next;
        Node(const T& val) : data(val), prev(nullptr), next(nullptr) {}
    };
    Node* head;
    Node* tail;
    int count;

public:
    DequeList() : head(nullptr), tail(nullptr), count(0) {}
    ~DequeList() { while (!empty()) pop_front(); }

    bool empty() const { return head == nullptr; }
    int size() const { return count; }

    void push_front(const T& val) {
        Node* newNode = new Node(val);
        if (empty()) {
            head = tail = newNode;
        } else {
            newNode->next = head;
            head->prev = newNode;
            head = newNode;
        }
        ++count;
    }

    void push_back(const T& val) {
        Node* newNode = new Node(val);
        if (empty()) {
            head = tail = newNode;
        } else {
            newNode->prev = tail;
            tail->next = newNode;
            tail = newNode;
        }
        ++count;
    }

    void pop_front() {
        if (empty()) throw std::underflow_error("Deque empty");
        Node* temp = head;
        head = head->next;
        if (head) head->prev = nullptr;
        else tail = nullptr;
        delete temp;
        --count;
    }

    void pop_back() {
        if (empty()) throw std::underflow_error("Deque empty");
        Node* temp = tail;
        tail = tail->prev;
        if (tail) tail->next = nullptr;
        else head = nullptr;
        delete temp;
        --count;
    }

    T& front() {
        if (empty()) throw std::runtime_error("Deque empty");
        return head->data;
    }

    T& back() {
        if (empty()) throw std::runtime_error("Deque empty");
        return tail->data;
    }
};
```

#### 3. STL `std::deque`

C++ 표준 라이브러리는 `std::deque`를 제공합니다. 내부적으로는 여러 블록으로 나누어진 동적 배열 구조로, 양끝 삽입/삭제가 O(1)이며 중간 삽입도 비교적 효율적입니다.

```cpp
#include <deque>
#include <iostream>

int main() {
    std::deque<int> dq;
    dq.push_back(10);    // [10]
    dq.push_front(20);   // [20, 10]
    dq.push_back(30);    // [20, 10, 30]

    std::cout << dq.front() << '\n'; // 20
    std::cout << dq.back() << '\n';  // 30

    dq.pop_front();      // [10, 30]
    dq.pop_back();       // [10]

    return 0;
}
```

### 실전 응용: 슬라이딩 윈도우 최대값 (모노토닉 덱)

주어진 배열에서 길이 `k`인 모든 연속 부분 배열의 최대값을 구하는 문제입니다.  
덱에 인덱스를 저장하며, 값이 **내림차순**이 되도록 유지합니다.

```cpp
#include <vector>
#include <deque>

std::vector<int> slidingWindowMax(const std::vector<int>& nums, int k) {
    std::deque<int> dq; // 인덱스 저장, 값은 내림차순 유지
    std::vector<int> result;
    for (int i = 0; i < nums.size(); ++i) {
        // 뒤에서 nums[i]보다 작은 값들은 제거 (내림차순 유지)
        while (!dq.empty() && nums[dq.back()] <= nums[i])
            dq.pop_back();
        dq.push_back(i);

        // 윈도우 범위를 벗어난 앞쪽 인덱스 제거
        if (dq.front() <= i - k)
            dq.pop_front();

        // 윈도우가 완성된 경우 결과 추가
        if (i >= k - 1)
            result.push_back(nums[dq.front()]);
    }
    return result;
}
```

시간 복잡도: 각 인덱스는 최대 한 번 push, 한 번 pop → O(n)

---

## 우선순위 큐 (Priority Queue)

### 우선순위 큐란?

우선순위 큐는 **들어간 순서와 관계없이 우선순위가 가장 높은 요소가 먼저 나오는 큐**입니다.  
내부적으로는 보통 **힙(heap)**이라는 자료구조를 사용합니다.

### 힙의 개념

힙은 **완전 이진 트리**이며, 다음 성질을 만족합니다.

- **최대 힙**: 부모 노드의 값이 자식 노드의 값보다 항상 크거나 같다.
- **최소 힙**: 부모 노드의 값이 자식 노드의 값보다 항상 작거나 같다.

힙은 배열로 쉽게 표현할 수 있습니다.  
루트는 인덱스 0, 어떤 노드의 인덱스가 `i`이면:
- 왼쪽 자식: `2*i + 1`
- 오른쪽 자식: `2*i + 2`
- 부모: `(i-1)/2`

### 주요 연산과 시간 복잡도

| 연산 | 설명 | 시간 복잡도 |
|------|------|-------------|
| `push(x)` | 새 원소 삽입 | O(log n) |
| `pop()`   | 최우선 원소 제거 | O(log n) |
| `top()`   | 최우선 원소 조회 | O(1) |
| `empty()` | 비었는지 확인 | O(1) |

### 간단한 최대 힙 구현 (정수)

```cpp
#include <vector>
#include <stdexcept>

class MaxHeap {
private:
    std::vector<int> data;

    void siftUp(int i) {
        while (i > 0) {
            int parent = (i - 1) / 2;
            if (data[i] <= data[parent]) break;
            std::swap(data[i], data[parent]);
            i = parent;
        }
    }

    void siftDown(int i) {
        int n = data.size();
        while (true) {
            int left = 2 * i + 1;
            int right = 2 * i + 2;
            int largest = i;
            if (left < n && data[left] > data[largest])
                largest = left;
            if (right < n && data[right] > data[largest])
                largest = right;
            if (largest == i) break;
            std::swap(data[i], data[largest]);
            i = largest;
        }
    }

public:
    bool empty() const { return data.empty(); }
    int size() const { return data.size(); }

    void push(int val) {
        data.push_back(val);
        siftUp(data.size() - 1);
    }

    void pop() {
        if (empty()) throw std::runtime_error("Heap empty");
        data[0] = data.back();
        data.pop_back();
        if (!empty()) siftDown(0);
    }

    int top() const {
        if (empty()) throw std::runtime_error("Heap empty");
        return data[0];
    }
};
```

### STL `std::priority_queue`

C++ 표준 라이브러리는 `std::priority_queue`를 제공합니다. 기본적으로 **최대 힙**입니다.

```cpp
#include <queue>
#include <vector>
#include <iostream>

int main() {
    std::priority_queue<int> maxPQ; // 최대 힙

    maxPQ.push(30);
    maxPQ.push(10);
    maxPQ.push(50);
    maxPQ.push(20);

    std::cout << maxPQ.top() << '\n'; // 50
    maxPQ.pop();
    std::cout << maxPQ.top() << '\n'; // 30
}
```

#### 최소 힙으로 사용하기

```cpp
std::priority_queue<int, std::vector<int>, std::greater<int>> minPQ;
```

#### 사용자 정의 비교

```cpp
struct Job {
    int id;
    int priority;
};

struct CompareJob {
    bool operator()(const Job& a, const Job& b) const {
        return a.priority < b.priority; // priority 큰 것이 먼저 (최대 힙)
    }
};

std::priority_queue<Job, std::vector<Job>, CompareJob> jobQueue;
```

### 실전 응용: Top K 빈도수

가장 빈도가 높은 K개의 원소를 찾는 문제입니다.  
최소 힙을 사용하여 크기 K를 유지하면 됩니다.

```cpp
#include <queue>
#include <vector>
#include <unordered_map>

std::vector<int> topKFrequent(const std::vector<int>& nums, int k) {
    std::unordered_map<int, int> freq;
    for (int x : nums) freq[x]++;

    // 최소 힙 (pair<빈도, 값>)
    auto cmp = [](auto& a, auto& b) { return a.first > b.first; };
    std::priority_queue<std::pair<int,int>, std::vector<std::pair<int,int>>, decltype(cmp)> pq(cmp);

    for (auto& [value, count] : freq) {
        pq.push({count, value});
        if (pq.size() > k) pq.pop();
    }

    std::vector<int> result;
    while (!pq.empty()) {
        result.push_back(pq.top().second);
        pq.pop();
    }
    return result;
}
```

### 실전 응용: K개 정렬 리스트 병합

K개의 정렬된 리스트를 하나의 정렬된 리스트로 합칩니다.  
각 리스트의 첫 원소를 최소 힙에 넣고, 하나씩 꺼내면서 다음 원소를 추가합니다.

```cpp
#include <queue>
#include <vector>

struct Element {
    int value;
    int listIdx;
    int elementIdx;
};

struct Compare {
    bool operator()(const Element& a, const Element& b) {
        return a.value > b.value; // 최소 힙
    }
};

std::vector<int> mergeKLists(const std::vector<std::vector<int>>& lists) {
    std::priority_queue<Element, std::vector<Element>, Compare> pq;
    for (int i = 0; i < lists.size(); ++i) {
        if (!lists[i].empty())
            pq.push({lists[i][0], i, 0});
    }

    std::vector<int> result;
    while (!pq.empty()) {
        Element e = pq.top(); pq.pop();
        result.push_back(e.value);
        if (e.elementIdx + 1 < lists[e.listIdx].size()) {
            pq.push({lists[e.listIdx][e.elementIdx + 1], e.listIdx, e.elementIdx + 1});
        }
    }
    return result;
}
```

---

## 정리

- **덱**: 양끝 삽입/삭제가 자주 필요한 경우 사용. STL의 `std::deque`가 범용적으로 좋다. 슬라이딩 윈도우 문제에서 모노토닉 덱 기법이 유용하다.
- **우선순위 큐**: 항상 우선순위가 가장 높은 요소를 빠르게 꺼내야 할 때 사용. STL의 `std::priority_queue`가 편리하며, 힙 정렬, 다익스트라, K-way 병합 등 다양한 알고리즘에 활용된다.

초보자라면 STL 컨테이너를 먼저 익히고, 내부 동작을 이해한 후 필요에 따라 직접 구현해보는 것을 추천한다.