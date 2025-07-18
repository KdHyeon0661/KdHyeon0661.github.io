---
layout: post
title: Data Structure - 힙
date: 2024-12-12 19:20:23 +0900
category: Data Structure
---
# 힙(Heap) — 우선순위를 다루는 강력한 트리 구조

힙은 **우선순위 기반 자료구조**로서, 정렬이나 우선순위 큐 구현에 매우 널리 사용됩니다.  
트리 구조 중 하나지만 **완전 이진 트리**라는 점에서 배열로도 효과적으로 표현할 수 있습니다.

---

## 📌 1. 힙이란?

> 힙은 **완전 이진 트리(Complete Binary Tree)**를 기반으로 한 자료구조로,  
> **부모 노드와 자식 노드 간의 우선순위 관계**를 유지합니다.

### 📎 힙의 종류

- **최대 힙(Max Heap)**: 부모 노드 ≥ 자식 노드  
- **최소 힙(Min Heap)**: 부모 노드 ≤ 자식 노드

예시 (최대 힙):

```
      50
     /  \
   30    20
  / \   / 
10 15  5
```

---

## 📦 2. 힙의 특징

| 특징 | 설명 |
|------|------|
| 완전 이진 트리 | 항상 좌측부터 채우는 트리 |
| 삽입/삭제 O(log n) | 높이가 log n이므로 효율적 |
| 배열로 구현 가능 | 인덱스로 부모/자식 관계 계산 |

---

### 🔍 힙(Heap) vs 우선순위 큐(Priority Queue)

| 항목 | 힙(Heap) | 우선순위 큐(Priority Queue) |
|------|----------|------------------------------|
| 개념 | 자료를 우선순위 기준으로 정렬된 **트리 구조** | 우선순위가 높은 요소를 먼저 꺼내는 **추상 자료구조** |
| 구조 | **완전 이진 트리** 기반 (주로 배열로 구현) | 내부적으로 힙으로 구현됨 |
| 목적 | 정렬이나 트리 기반 알고리즘에 사용 | 데이터 처리 순서를 우선순위로 결정할 때 사용 |
| STL 제공 | 없음 (`std::make_heap`, `std::priority_queue` 등으로 간접적 제공) | `std::priority_queue`로 제공됨 |
| 구현 방식 | Max Heap, Min Heap 직접 구현 | 힙 기반으로 구현되며 인터페이스 제공 |

> ✅ **정리**: "힙은 우선순위 큐를 구현하는 데 사용되는 구조"이고,  
> **우선순위 큐는 추상적 개념**, **힙은 그 개념을 실현하는 도구**입니다.

---

### 배열 인덱스 규칙

| 노드 | 인덱스 |
|------|--------|
| 부모 | `i` |
| 왼쪽 자식 | `2i + 1` |
| 오른쪽 자식 | `2i + 2` |
| 부모 (역) | `(i - 1) / 2` |

---

## ⚙️ 3. 최대 힙(Max Heap) 구현 (C++)

```cpp
#include <vector>
#include <stdexcept>

class MaxHeap {
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
        if (heap.empty()) throw std::runtime_error("Heap is empty");
        heap[0] = heap.back();
        heap.pop_back();
        heapify_down(0);
    }

    int top() const {
        if (heap.empty()) throw std::runtime_error("Heap is empty");
        return heap[0];
    }

    bool empty() const { return heap.empty(); }

    int size() const { return heap.size(); }
};
```

---

## 🔄 4. 최소 힙(Min Heap) 구현

> 위 코드에서 `>`을 `<`으로 바꾸면 최소 힙이 됩니다.

또는 STL에서 `std::priority_queue`에 `std::greater<>`를 적용해 최소 힙을 구현할 수 있습니다:

```cpp
#include <queue>
#include <functional>

std::priority_queue<int, std::vector<int>, std::greater<int>> minHeap;
minHeap.push(10);
minHeap.push(2);
minHeap.push(7);
// top()은 2
```

---

## 🧪 5. STL `priority_queue` 요약

| 형태 | 선언 | 힙 종류 |
|------|------|----------|
| 기본 | `std::priority_queue<int>` | 최대 힙 |
| 최소 힙 | `std::priority_queue<int, std::vector<int>, std::greater<int>>` | 최소 힙 |

---

## 📊 6. 힙 정렬 (Heap Sort)

> 힙을 이용하여 **정렬**할 수 있습니다 (O(n log n), 불안정 정렬).

### 정렬 과정 요약 (최대 힙 기준)

1. 모든 원소를 힙에 삽입 (heapify)
2. 하나씩 꺼내서 배열에 저장 (가장 큰 값부터 제거)

```cpp
void heap_sort(std::vector<int>& arr) {
    MaxHeap heap;
    for (int x : arr) heap.push(x);
    for (int i = arr.size() - 1; i >= 0; --i) {
        arr[i] = heap.top();
        heap.pop();
    }
}
```

---

## 🧠 7. 힙의 응용

- **우선순위 큐 (priority queue)**: 힙이 핵심 자료구조
- **힙 정렬**
- **최단 경로 (다익스트라 알고리즘)**
- **최대/최소 k개 원소 추출**
- **온라인 데이터 흐름 중간값 구하기 (min/max heap 병행)**

---

## 📌 8. 정리

| 항목 | 최대 힙 | 최소 힙 |
|------|---------|---------|
| 정의 | 부모 ≥ 자식 | 부모 ≤ 자식 |
| top() | 최대값 | 최소값 |
| 사용 예 | 우선순위 큐, 힙 정렬 | 최단거리, 데이터 흐름 |

---

## ✅ 결론

- 힙은 정렬, 큐, 그래프 등 다양한 분야에 사용되는 핵심 자료구조입니다.
- 배열로 표현 가능한 **트리 기반 자료구조**라는 점에서 구현이 직관적이며 강력합니다.
- STL `priority_queue`를 적극 활용하되, 내부 작동 원리를 익히는 것이 중요합니다.