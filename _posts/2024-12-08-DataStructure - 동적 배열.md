---
layout: post
title: Data Structure - 동적 배열
date: 2024-12-08 20:20:23 +0900
category: Data Structure
---
# 동적 배열

## 동적 배열이란?

**동적 배열(dynamic array)**은 실행 시간 동안 크기가 변할 수 있는 배열입니다.  
일반 배열(정적 배열)은 선언할 때 크기를 고정해야 하지만, 동적 배열은 필요에 따라 자동으로 크기가 늘어나거나 줄어듭니다.

### 정적 배열 vs 동적 배열

```cpp
// 정적 배열 (크기 고정)
int arr[5] = {1, 2, 3, 4, 5};  // 항상 5칸만 사용 가능

// 동적 배열 (std::vector 사용)
#include <vector>
std::vector<int> vec;           // 처음에는 비어 있음
vec.push_back(10);               // 크기가 1로 증가
vec.push_back(20);               // 크기가 2로 증가
```

- **정적 배열**: 메모리가 스택에 할당되고, 크기를 바꿀 수 없습니다.
- **동적 배열**: 메모리가 힙에 할당되고, 요소 추가 시 자동으로 공간을 확보합니다.

---

## C++의 동적 배열: `std::vector`

C++ 표준 라이브러리는 `std::vector`라는 템플릿 클래스를 제공합니다.  
가장 많이 사용하는 컨테이너 중 하나이며, 성능과 편의성 모두 뛰어납니다.

### `std::vector` 기본 사용법

#### 헤더 포함

```cpp
#include <vector>
```

#### 생성

```cpp
std::vector<int> v1;                // 빈 벡터
std::vector<int> v2(10);            // 10개의 0으로 초기화된 벡터
std::vector<int> v3(5, 100);        // 5개의 100으로 채워진 벡터
std::vector<int> v4 = {1, 2, 3, 4}; // 초기화 리스트
```

#### 요소 추가 (끝에 삽입)

```cpp
std::vector<int> v;
v.push_back(10);    // v: [10]
v.push_back(20);    // v: [10, 20]
v.push_back(30);    // v: [10, 20, 30]
```

#### 요소 접근

```cpp
int x = v[1];           // 20 (범위 검사 없음)
int y = v.at(1);        // 20 (범위 검사 있음, 예외 발생 가능)
int first = v.front();  // 첫 요소 (10)
int last = v.back();    // 마지막 요소 (30)
```

#### 요소 삭제

```cpp
v.pop_back();           // 마지막 요소 제거 → [10, 20]
v.clear();              // 모든 요소 제거 → []
```

#### 크기와 용량

- `size()` : 현재 저장된 요소의 개수
- `capacity()` : 현재 할당된 메모리 공간이 저장할 수 있는 요소의 최대 개수 (재할당 없이)

```cpp
std::vector<int> v;
std::cout << v.size() << ", " << v.capacity(); // 0, 0

v.push_back(1);
std::cout << v.size() << ", " << v.capacity(); // 1, 1 (컴파일러마다 다를 수 있음)

v.push_back(2);
// size:2, capacity:2 (또는 3, 구현에 따라 다름)
```

#### 미리 공간 예약하기

```cpp
std::vector<int> v;
v.reserve(100);         // 최소 100개의 공간을 미리 확보
std::cout << v.capacity(); // 100 (또는 그 이상)
```

`reserve`를 사용하면 잦은 재할당을 피할 수 있습니다.

#### 불필요한 용량 줄이기

```cpp
std::vector<int> v(100);   // size=100, capacity>=100
v.clear();                 // size=0, capacity는 그대로
v.shrink_to_fit();         // capacity를 size에 맞춤 (요청)
```

---

## 용량과 재할당

`std::vector`는 내부적으로 연속된 메모리 블록을 사용합니다.  
새 요소를 추가할 때 현재 용량이 부족하면 더 큰 메모리 블록을 새로 할당하고, 기존 요소를 모두 옮깁니다. 이 과정을 **재할당(reallocation)**이라고 합니다.

재할당은 비용이 큰 작업이므로, `vector`는 한 번에 많은 공간을 추가로 할당하여 재할당 횟수를 줄입니다. 일반적으로 현재 용량의 **1.5배 또는 2배**를 새 용량으로 삼습니다.

### 상환 분석 (Amortized Analysis)

재할당 비용을 모든 삽입에 분산시키면, 한 번의 삽입 평균 비용은 **상수 시간**이 됩니다.

예를 들어, 용량을 2배씩 늘리는 정책에서 n번의 `push_back`에 드는 총 요소 이동 횟수는 대략

$$
\sum_{i=0}^{\lfloor \log_2 n \rfloor} \frac{n}{2^i} \le 2n = O(n)
$$

입니다. 따라서 **평균적으로 1회 삽입당 O(1)**의 시간이 걸린다고 말합니다.

---

## 반복자 (Iterator)

`vector`는 반복자를 제공하여 요소들을 순회할 수 있습니다.

```cpp
std::vector<int> v = {10, 20, 30};

// begin()은 첫 요소를 가리키는 반복자, end()는 마지막 다음을 가리킴
for (auto it = v.begin(); it != v.end(); ++it) {
    std::cout << *it << " ";   // 10 20 30
}

// 범위 기반 for 문 (C++11)
for (int x : v) {
    std::cout << x << " ";
}
```

---

## 실전 예제

### 예제 1: 학생 점수 관리

```cpp
#include <iostream>
#include <vector>
#include <numeric>   // accumulate

int main() {
    std::vector<int> scores;

    // 점수 입력
    scores.push_back(85);
    scores.push_back(92);
    scores.push_back(78);
    scores.push_back(94);
    scores.push_back(88);

    // 총점 계산
    int sum = 0;
    for (int s : scores) {
        sum += s;
    }
    double average = static_cast<double>(sum) / scores.size();
    std::cout << "평균: " << average << std::endl;

    // 90점 이상인 학생 수
    int cnt = 0;
    for (int s : scores) {
        if (s >= 90) ++cnt;
    }
    std::cout << "90점 이상: " << cnt << "명" << std::endl;

    return 0;
}
```

### 예제 2: 짝수만 남기기

```cpp
#include <vector>
#include <iostream>

int main() {
    std::vector<int> v = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};

    // 짝수만 남기고 홀수는 제거 (뒤에서부터 순회하며 erase)
    for (auto it = v.begin(); it != v.end(); ) {
        if (*it % 2 != 0) {
            it = v.erase(it);   // erase는 다음 요소 반복자를 반환
        } else {
            ++it;
        }
    }

    for (int x : v) {
        std::cout << x << " ";   // 2 4 6 8 10
    }
    return 0;
}
```

### 예제 3: 동적 배열을 함수에 전달

```cpp
#include <vector>
#include <iostream>

// 값 복사 (벡터 전체가 복사됨)
void print_vector(std::vector<int> v) {
    for (int x : v) std::cout << x << " ";
    std::cout << std::endl;
}

// 참조 전달 (복사 없음)
void add_one(std::vector<int>& v) {
    for (int& x : v) ++x;
}

int main() {
    std::vector<int> data = {1, 2, 3};
    add_one(data);
    print_vector(data);   // 2 3 4
    return 0;
}
```

---

## 주의할 점

### 반복자 무효화 (Iterator Invalidation)

`vector`에 요소를 추가하거나 제거하면, 기존에 가지고 있던 반복자, 포인터, 참조가 무효화될 수 있습니다.

- 재할당이 일어나면 모든 반복자가 무효화됩니다.
- 중간에 삽입/삭제가 일어나면 그 위치 이후의 반복자가 무효화됩니다.

```cpp
std::vector<int> v = {1, 2, 3, 4};
auto it = v.begin() + 2;  // 3을 가리킴
v.push_back(5);            // 재할당 가능성 → it 무효화 위험
// *it 를 사용하면 미정의 동작!
```

안전하게 사용하려면 삽입/삭제 후 반복자를 다시 얻어야 합니다.

### 인덱스 범위

`operator[]`는 범위를 검사하지 않으므로, 유효하지 않은 인덱스 접근은 미정의 동작을 일으킵니다.  
범위를 벗어날 위험이 있다면 `at()` 멤버 함수를 사용하세요.

```cpp
std::vector<int> v(10);
int x = v[20];   // 위험! (미정의 동작)
int y = v.at(20); // std::out_of_range 예외 발생
```

---

## 요약

- `std::vector`는 크기가 변할 수 있는 연속 메모리 컨테이너입니다.
- **랜덤 접근**: 인덱스로 요소에 $$O(1)$$에 접근할 수 있습니다.
- **끝 삽입/삭제**: 평균적으로 $$O(1)$$ (상환 분석).
- **중간 삽입/삭제**: $$O(n)$$ (이동 비용).
- `size()`는 현재 요소 수, `capacity()`는 현재 할당된 메모리 크기입니다.
- `reserve()`로 재할당 횟수를 줄일 수 있습니다.
- 반복자 무효화에 주의해야 합니다.

동적 배열은 프로그래밍에서 가장 널리 쓰이는 자료구조 중 하나이며, C++의 `std::vector`는 그 강력하고 안전한 구현체입니다. 기본 사용법을 익히고, 필요에 따라 고급 기능(예: 사용자 정의 할당자, 이동 의미론)을 학습해 나가면 됩니다.