---
layout: post
title: Data Structure - 정렬
date: 2024-12-25 19:20:23 +0900
category: Data Structure
---
# 📚 정렬(Sorting) 알고리즘 정리

정렬이란, **데이터를 일정한 순서(오름차순/내림차순 등)**로 나열하는 알고리즘입니다.  
데이터를 정렬함으로써 **검색, 탐색, 비교 연산**이 효율적으로 이루어집니다.

---

## ✅ 1. 정렬 알고리즘 분류

| 분류 기준 | 종류 |
|-----------|------|
| **기초 정렬** | 선택 정렬, 버블 정렬, 삽입 정렬 |
| **고급 정렬** | 병합 정렬, 퀵 정렬, 힙 정렬 |
| **비교 기반 X** | 기수 정렬, 계수 정렬, 버킷 정렬 |
| **안정 정렬 여부** | 병합/삽입/기수(✔), 퀵/힙/선택(❌) |
| **제자리 정렬 여부** | 퀵/삽입/버블(✔), 병합(❌) |

---

## 🧮 2. 기본 정렬 3종 (O(n²))

### 🔸 선택 정렬 (Selection Sort)

- 매번 **가장 작은 값을 선택하여** 앞으로 보냄
- 시간복잡도: O(n²)

```cpp
#include <iostream>
#include <vector>
using namespace std;

void selectionSort(vector<int>& arr) {
    int n = arr.size();
    for (int i = 0; i < n - 1; ++i) {
        int minIdx = i;
        for (int j = i + 1; j < n; ++j)
            if (arr[j] < arr[minIdx]) minIdx = j;
        swap(arr[i], arr[minIdx]);
    }
}

int main() {
    vector<int> arr = {20, 7, 1, 34, 12, 3};
    cout << "정렬 전 배열: ";
    for (int num : arr) cout << num << " ";
    cout << endl;

    selectionSort(arr);

    cout << "정렬 후 배열: ";
    for (int num : arr) cout << num << " ";
    cout << endl;

    return 0;
}
```

---

### 🔸 버블 정렬 (Bubble Sort)

- 인접한 두 값을 비교하며 스왑
- 가장 큰 값이 거품처럼 뒤로 밀려감
- 시간복잡도: O(n²)

```cpp
#include <iostream>
#include <vector>
using namespace std;

void bubbleSort(vector<int>& arr) {
    int n = arr.size();
    for (int i = 0; i < n - 1; ++i)
        for (int j = 0; j < n - 1 - i; ++j)
            if (arr[j] > arr[j + 1]) swap(arr[j], arr[j + 1]);
}

int main() {
    vector<int> arr = {9, 1, 6, 3, 5, 2};
    cout << "정렬 전 배열: ";
    for (int num : arr) cout << num << " ";
    cout << endl;

    bubbleSort(arr);

    cout << "정렬 후 배열: ";
    for (int num : arr) cout << num << " ";
    cout << endl;

    return 0;
}
```

---

### 🔸 삽입 정렬 (Insertion Sort)

- 앞에서부터 차례대로 **정렬된 영역에 삽입**
- 시간복잡도: O(n²), 정렬된 데이터에 매우 빠름

```cpp
#include <iostream>
#include <vector>
using namespace std;

void insertionSort(vector<int>& arr) {
    int n = arr.size();
    for (int i = 1; i < n; ++i) {
        int key = arr[i], j = i - 1;
        // key 보다 큰 값들은 오른쪽으로 한 칸씩 이동
        while (j >= 0 && arr[j] > key)
            arr[j + 1] = arr[j--];
        arr[j + 1] = key;
    }
}

int main() {
    vector<int> arr = {12, 4, 5, 3, 8, 7};
    cout << "정렬 전 배열: ";
    for (int num : arr) cout << num << " ";
    cout << endl;

    insertionSort(arr);

    cout << "정렬 후 배열: ";
    for (int num : arr) cout << num << " ";
    cout << endl;

    return 0;
}
```

---

## 🚀 3. 고급 정렬 (O(n log n))

### ⚡ 퀵 정렬 (Quick Sort)

- 분할 정복
- 피벗을 기준으로 좌우로 분할
- 평균 O(n log n), 최악 O(n²)

```cpp
#include <iostream>
#include <vector>
using namespace std;

int partition(vector<int>& arr, int low, int high) {
    int pivot = arr[high], i = low - 1;
    for (int j = low; j < high; ++j)
        if (arr[j] < pivot) swap(arr[++i], arr[j]);
    swap(arr[i + 1], arr[high]);
    return i + 1;
}

void quickSort(vector<int>& arr, int low, int high) {
    if (low < high) {
        int pi = partition(arr, low, high);
        quickSort(arr, low, pi - 1);
        quickSort(arr, pi + 1, high);
    }
}

int main() {
    vector<int> arr = {29, 10, 14, 37, 13};
    cout << "정렬 전 배열: ";
    for (int num : arr) cout << num << " ";
    cout << endl;

    quickSort(arr, 0, arr.size() - 1);

    cout << "정렬 후 배열: ";
    for (int num : arr) cout << num << " ";
    cout << endl;

    return 0;
}
```

---

### ⚙ 병합 정렬 (Merge Sort)

- 분할 정복
- **안정 정렬**, 공간 O(n)
- 항상 O(n log n)

```cpp
#include <iostream>
#include <vector>
using namespace std;

void merge(vector<int>& arr, int l, int m, int r) {
    vector<int> left(arr.begin() + l, arr.begin() + m + 1);
    vector<int> right(arr.begin() + m + 1, arr.begin() + r + 1);
    int i = 0, j = 0, k = l;
    while (i < left.size() && j < right.size())
        arr[k++] = (left[i] <= right[j]) ? left[i++] : right[j++];
    while (i < left.size()) arr[k++] = left[i++];
    while (j < right.size()) arr[k++] = right[j++];
}

void mergeSort(vector<int>& arr, int l, int r) {
    if (l < r) {
        int m = (l + r) / 2;
        mergeSort(arr, l, m);
        mergeSort(arr, m + 1, r);
        merge(arr, l, m, r);
    }
}

int main() {
    vector<int> arr = {38, 27, 43, 3, 9, 82, 10};
    cout << "정렬 전 배열: ";
    for (int num : arr) cout << num << " ";
    cout << endl;

    mergeSort(arr, 0, arr.size()-1);

    cout << "정렬 후 배열: ";
    for (int num : arr) cout << num << " ";
    cout << endl;

    return 0;
}
```

---

### 🧱 힙 정렬 (Heap Sort)

- 최대 힙(또는 최소 힙)을 이용
- O(n log n), 정렬 후 힙 소멸
- 제자리 정렬 가능, **안정 X**

```cpp
#include <iostream>
#include <vector>
using namespace std;

void heapify(vector<int>& arr, int n, int i) {
    int largest = i, l = 2*i + 1, r = 2*i + 2;
    if (l < n && arr[l] > arr[largest]) largest = l;
    if (r < n && arr[r] > arr[largest]) largest = r;
    if (largest != i) {
        swap(arr[i], arr[largest]);
        heapify(arr, n, largest);
    }
}

void heapSort(vector<int>& arr) {
    int n = arr.size();
    for (int i = n / 2 - 1; i >= 0; --i)
        heapify(arr, n, i);
    for (int i = n - 1; i > 0; --i) {
        swap(arr[0], arr[i]);
        heapify(arr, i, 0);
    }
}

int main() {
    vector<int> arr = {12, 11, 13, 5, 6, 7};
    cout << "정렬 전 배열: ";
    for (int num : arr) cout << num << " ";
    cout << endl;

    heapSort(arr);

    cout << "정렬 후 배열: ";
    for (int num : arr) cout << num << " ";
    cout << endl;

    return 0;
}

```

---

### ✅ 기수 정렬 (Radix Sort)

#### 🔎 개념

기수 정렬은 **자릿수를 기준으로 정렬**하는 알고리즘입니다.  
가장 낮은 자릿수(일의 자리)부터 높은 자릿수(천, 만의 자리)까지 차례대로 정렬합니다.

- 안정 정렬
- 정수, 고정된 길이의 문자열 등에 유리
- **비교 없이 정렬** (`O(d * n)` → d: 자릿수, n: 원소 수)

---

#### 📌 작동 방식 (예: LSD 방식)

정렬 대상: `170, 45, 75, 90, 802, 24, 2, 66`

1. 일의 자리 기준 정렬 →  
   `170, 90, 802, 2, 24, 45, 75, 66`
2. 십의 자리 기준 정렬 →  
   `802, 2, 24, 45, 66, 170, 75, 90`
3. 백의 자리 기준 정렬 →  
   `2, 24, 45, 66, 75, 90, 170, 802`

결과: **오름차순 정렬 완료**

---

#### 🛠️ C++ 예제

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
using namespace std;

int getMax(const vector<int>& arr) {
    return *max_element(arr.begin(), arr.end());
}

void countingSort(vector<int>& arr, int exp) {
    vector<int> output(arr.size());
    int count[10] = {0};

    for (int i = 0; i < arr.size(); i++)
        count[(arr[i] / exp) % 10]++;

    for (int i = 1; i < 10; i++)
        count[i] += count[i - 1];

    for (int i = arr.size() - 1; i >= 0; i--) {
        int idx = (arr[i] / exp) % 10;
        output[count[idx] - 1] = arr[i];
        count[idx]--;
    }

    arr = output;
}

void radixSort(vector<int>& arr) {
    int maxVal = getMax(arr);
    for (int exp = 1; maxVal / exp > 0; exp *= 10)
        countingSort(arr, exp);
}

int main() {
    vector<int> arr = {170, 45, 75, 90, 802, 24, 2, 66};
    radixSort(arr);

    for (int num : arr)
        cout << num << " ";
    cout << endl;
    return 0;
}
```

---

### ✅ 버킷 정렬 (Bucket Sort)

#### 🔎 개념

버킷 정렬은 데이터를 여러 개의 **구간(버킷)**으로 나누고, 각 버킷을 정렬한 뒤 합치는 방식입니다.

- 실수, 균일 분포된 데이터에 효과적
- 평균 시간복잡도: **O(n + k)**  
- 최악: O(n²) (한 버킷에 몰릴 경우)

---

#### 🛠️ 작동 예시

입력: `0.78, 0.17, 0.39, 0.26, 0.72, 0.94, 0.21, 0.12, 0.23, 0.68`

1. 10개의 버킷 [0.0~0.1), [0.1~0.2), ... [0.9~1.0)
2. 각 값 분류 후 내부 정렬
3. 모든 버킷을 순서대로 합침

결과: 오름차순 정렬

---

#### 🛠️ C++ 예제

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
using namespace std;

void bucketSort(vector<float>& arr) {
    int n = arr.size();
    vector<vector<float>> buckets(n);

    for (float num : arr) {
        int idx = num * n; // 버킷 인덱스
        buckets[idx].push_back(num);
    }

    for (int i = 0; i < n; i++)
        sort(buckets[i].begin(), buckets[i].end());

    int index = 0;
    for (int i = 0; i < n; i++)
        for (float num : buckets[i])
            arr[index++] = num;
}

int main() {
    vector<float> arr = {0.78, 0.17, 0.39, 0.26, 0.72, 0.94, 0.21, 0.12, 0.23, 0.68};
    bucketSort(arr);

    for (float num : arr)
        cout << num << " ";
    cout << endl;
    return 0;
}
```

---

## 📊 정렬 알고리즘 비교 요약

| 알고리즘   | 최선 시간복잡도 | 평균 시간복잡도 | 최악 시간복잡도 | 공간복잡도 | 안정성 | 제자리 정렬 |
|------------|----------------|-----------------|----------------|------------|--------|-------------|
| 선택 정렬   | O(n²)          | O(n²)           | O(n²)          | O(1)       | ❌     | ✅          |
| 버블 정렬   | O(n)           | O(n²)           | O(n²)          | O(1)       | ✅     | ✅          |
| 삽입 정렬   | O(n)           | O(n²)           | O(n²)          | O(1)       | ✅     | ✅          |
| 퀵 정렬     | O(n log n)     | O(n log n)      | O(n²)          | O(log n)   | ❌     | ✅          |
| 병합 정렬   | O(n log n)     | O(n log n)      | O(n log n)     | O(n)       | ✅     | ❌          |
| 힙 정렬     | O(n log n)     | O(n log n)      | O(n log n)     | O(1)       | ❌     | ✅          |
| 버킷 정렬   | O(n + k)       | O(n + k)        | O(n²)          | O(n + k)   | 조건부*  | ❌          |       
| 기수 정렬   | O(nk)          | O(nk)           | O(nk)          | O(n + k)   | 조건부*  | ❌          |

> 안정성: 같은 값의 원소 순서가 정렬 후에도 보장되는가 (안정정렬=✅)
> 제자리 정렬: 추가 메모리 없이 정렬 가능한가 (제자리=✅)

---

## ✅ 마무리

- 🔎 탐색 효율을 위해 정렬은 필수
- ⚖ 상황에 맞게 정렬 알고리즘 선택
- 📦 STL에서는 `std::sort`, `std::stable_sort` 지원