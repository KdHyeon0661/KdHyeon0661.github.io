---
layout: post
title: Data Structure - 펜윅 트리
date: 2024-12-09 19:20:23 +0900
category: Data Structure
---
# 📊 펜윅 트리 (Fenwick Tree, BIT) - 구간 합을 빠르게!

## 🔍 1. 개요

**펜윅 트리(Fenwick Tree)** 또는 **BIT(Binary Indexed Tree)** 는  
**구간 합(range sum)** 또는 **누적 합(prefix sum)** 을 **O(log n)** 시간에 처리할 수 있는 트리 기반 자료구조입니다.

- 일반적인 `prefix_sum[i]` 구간합은 배열을 사용하면 `O(1)`에 가능하지만, **값 수정이 O(n)** 으로 느립니다.
- 세그먼트 트리처럼 **업데이트와 쿼리 둘 다 O(log n)** 을 보장하면서도, 구현이 **더 간단하고 메모리 사용량도 적습니다.**

---

## ⚙️ 2. 사용 목적

- 누적 합 / 구간 합
- 누적 카운트 (빈도수)
- 인버전 수 계산
- 온라인 순위, 랭킹 시스템
- 낮은 수준의 세그먼트 트리 대체

---

## 🧠 3. 동작 원리 요약

- 배열 기반 트리 구조
- 인덱스의 **LSB(Least Significant Bit)** 를 이용
- `i += (i & -i)` → 부모 노드  
- `i -= (i & -i)` → 자식 노드

---

## 🔧 4. 핵심 연산

### ✅ 구간합 (prefix sum)

```cpp
int query(int i) {
    int sum = 0;
    while (i > 0) {
        sum += bit[i];
        i -= (i & -i); // 이동
    }
    return sum;
}
```

---

### ✅ 값 갱신 (update)

```cpp
void update(int i, int delta) {
    while (i <= n) {
        bit[i] += delta;
        i += (i & -i); // 다음 노드
    }
}
```

---

## 💻 5. C++ 전체 구현 예제

```cpp
#include <iostream>
#include <vector>
using namespace std;

class FenwickTree {
    vector<int> bit;
    int n;

public:
    FenwickTree(int size) : n(size) {
        bit.assign(n + 1, 0);
    }

    // arr[1] ~ arr[i] 합
    int query(int i) {
        int sum = 0;
        while (i > 0) {
            sum += bit[i];
            i -= (i & -i);
        }
        return sum;
    }

    // arr[i] += delta
    void update(int i, int delta) {
        while (i <= n) {
            bit[i] += delta;
            i += (i & -i);
        }
    }

    // 구간합 [l, r]
    int rangeSum(int l, int r) {
        return query(r) - query(l - 1);
    }
};
```

---

## 🔎 6. 예제 실행

```cpp
int main() {
    FenwickTree ft(10);
    vector<int> arr = {0, 3, 2, -1, 6, 5, 4, -3, 3, 7, 2}; // 1-based

    for (int i = 1; i <= 10; ++i) {
        ft.update(i, arr[i]);
    }

    cout << "Sum of first 5 elements: " << ft.query(5) << endl;
    cout << "Sum from 3 to 7: " << ft.rangeSum(3, 7) << endl;

    ft.update(4, 3); // arr[4] += 3
    cout << "After update, sum of first 5 elements: " << ft.query(5) << endl;

    return 0;
}
```

### 🧾 출력 예시

```
Sum of first 5 elements: 15
Sum from 3 to 7: 11
After update, sum of first 5 elements: 18
```

---

## ⏱️ 7. 시간 복잡도

| 연산 | 시간복잡도 |
|------|------------|
| query(i) | O(log n) |
| update(i, delta) | O(log n) |
| 전체 초기화 | O(n log n) |

---

## ⚠️ 8. 주의사항

- 이 구현은 **1-based 인덱스**를 사용합니다 (즉, `bit[0]` 사용 금지)
- 원본 배열과 BIT는 별개로 유지되므로 `arr`를 따로 관리해야 함
- `i & -i`는 비트 트릭으로 LSB 추출

---

## ✅ 요약

| 항목 | 설명 |
|------|------|
| 펜윅 트리 | 구간합을 효율적으로 계산하는 배열 기반 구조 |
| 목적 | 누적 합, 빈도 누적 등 |
| 구현 | 간단한 비트 연산 + 배열 |
| 성능 | 쿼리와 갱신 모두 O(log n) |