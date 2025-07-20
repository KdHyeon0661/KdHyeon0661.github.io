---
layout: post
title: Data Structure - 펜윅 트리
date: 2024-12-09 19:20:23 +0900
category: Data Structure
---
# 📐 세그먼트 트리 (Segment Tree) - 구간 질의와 갱신을 빠르게!

## 🔍 1. 세그먼트 트리란?

**세그먼트 트리(Segment Tree)** 는 **구간 합, 구간 최솟값, 최댓값, 최소 공배수** 등  
**범위에 대한 질의(Range Query)** 와 **값 갱신(Update)** 를 **O(log n)** 시간에 수행하는 고급 자료구조입니다.

- 배열의 특정 구간에 대해 반복적으로 질의하는 문제에 유리합니다.
- 펜윅 트리보다 **더 다양한 연산(최댓값, 최솟값 등)** 을 지원합니다.
- **리시트**에 기반한 **트리 형태의 배열**로 구현됩니다.

---

## 🧩 2. 특징 요약

| 항목 | 설명 |
|------|------|
| 구조 | 완전 이진 트리(배열로 표현) |
| 지원 연산 | 합, 최솟값, 최댓값 등 범위 연산 |
| 쿼리 시간 | O(log n) |
| 갱신 시간 | O(log n) |
| 공간 복잡도 | O(n × 4) 정도 |

---

## 🛠️ 3. C++로 구현 (구간 합 예제)

### 📌 기본 아이디어
- 루트는 전체 구간 `[0, n-1]`
- 각 노드는 구간 `[l, r]` 을 표현
- 왼쪽 자식은 `[l, mid]`, 오른쪽 자식은 `[mid+1, r]`

---

### ✅ 세그먼트 트리 클래스 구현

```cpp
#include <iostream>
#include <vector>
using namespace std;

class SegmentTree {
    vector<int> tree;
    int n;

public:
    SegmentTree(const vector<int>& data) {
        n = data.size();
        tree.resize(n * 4);
        build(data, 1, 0, n - 1);
    }

    // 트리 빌드
    void build(const vector<int>& data, int node, int start, int end) {
        if (start == end) {
            tree[node] = data[start];
        } else {
            int mid = (start + end) / 2;
            build(data, node * 2, start, mid);
            build(data, node * 2 + 1, mid + 1, end);
            tree[node] = tree[node * 2] + tree[node * 2 + 1]; // 합 연산
        }
    }

    // 구간합 쿼리 [l, r]
    int query(int l, int r) {
        return query(1, 0, n - 1, l, r);
    }

    int query(int node, int start, int end, int l, int r) {
        if (r < start || end < l) return 0; // 겹치지 않음
        if (l <= start && end <= r) return tree[node]; // 완전히 포함
        int mid = (start + end) / 2;
        return query(node * 2, start, mid, l, r)
             + query(node * 2 + 1, mid + 1, end, l, r);
    }

    // 값 변경: arr[idx] += diff
    void update(int idx, int diff) {
        update(1, 0, n - 1, idx, diff);
    }

    void update(int node, int start, int end, int idx, int diff) {
        if (idx < start || idx > end) return;
        tree[node] += diff;
        if (start != end) {
            int mid = (start + end) / 2;
            update(node * 2, start, mid, idx, diff);
            update(node * 2 + 1, mid + 1, end, idx, diff);
        }
    }
};
```

---

## 🔍 4. 사용 예제

```cpp
int main() {
    vector<int> arr = {1, 3, 5, 7, 9, 11};
    SegmentTree seg(arr);

    cout << "Sum of range [1, 3]: " << seg.query(1, 3) << endl; // 3 + 5 + 7 = 15

    seg.update(1, 2); // arr[1] += 2 → 3 -> 5
    cout << "After update, sum of range [1, 3]: " << seg.query(1, 3) << endl; // 5 + 5 + 7 = 17

    return 0;
}
```

### 🔽 출력 예시
```
Sum of range [1, 3]: 15
After update, sum of range [1, 3]: 17
```

---

## ⚙️ 5. 다양한 연산을 위한 트리 구성

| 연산 종류 | 트리 노드 구성 |
|-----------|----------------|
| 구간 합   | `tree[node] = sum(left) + sum(right)` |
| 최댓값    | `tree[node] = max(left, right)` |
| 최솟값    | `tree[node] = min(left, right)` |
| 구간 곱   | `tree[node] = mul(left) * mul(right)` (모듈러 필요 시 `% MOD`) |

> 연산 방식만 바꾸면 다양한 문제에 대응할 수 있습니다.

---

## 🧠 6. 세그먼트 트리 vs 펜윅 트리

| 항목 | 세그먼트 트리 | 펜윅 트리 |
|------|----------------|------------|
| 구조 | 트리 | 배열 기반 트리 |
| 공간 | O(n * 4) | O(n) |
| 연산 종류 | 다양한 연산 지원 | 주로 구간 합 |
| 구현 난이도 | 다소 복잡 | 간단 |
| 구간 갱신 | 가능 | 어려움 (단일점 갱신만 쉬움) |

---

## ✅ 정리

| 키워드 | 설명 |
|--------|------|
| Segment Tree | 구간 기반 트리 자료구조 |
| 쿼리/갱신 | 모두 O(log n) |
| 구현 | 재귀적 트리 배열 |
| 장점 | 다양한 구간 연산 가능 |