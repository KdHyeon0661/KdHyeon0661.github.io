---
layout: post
title: Data Structure - 배열과 문자열
date: 2024-12-08 19:20:23 +0900
category: Data Structure
---
# 배열과 문자열

## 배열

### 배열이란?

배열은 **같은 종류의 데이터를 연속된 메모리 공간에 저장하는 자료구조**입니다.  
마치 번호가 붙은 우편함과 같아서, 몇 번째 상자인지만 알면 내용물을 바로 꺼낼 수 있습니다.

```cpp
#include <vector>
#include <iostream>

int main() {
    std::vector<int> arr = {10, 20, 30, 40, 50};   // C++ 동적 배열 (std::vector)
    std::cout << arr[2] << std::endl;              // 30
    return 0;
}
```

- 첫 번째 칸(인덱스 0)에는 10이 들어있고, 두 번째 칸(인덱스 1)에는 20이 들어있습니다.
- 인덱스로 원하는 위치의 값을 **즉시** 읽거나 쓸 수 있습니다.  
  예: `arr[2]`는 30을 바로 반환합니다.

### 배열의 특징

- **빠른 읽기/쓰기**: 위치(인덱스)만 알면 $$O(1)$$ 시간에 접근 가능합니다.
- **느린 중간 삽입/삭제**: 중간에 값을 넣거나 빼려면 그 뒤의 데이터를 모두 한 칸씩 밀어야 하므로 $$O(n)$$ 시간이 걸립니다.
- **캐시 친화적**: 데이터가 메모리에 연속으로 저장되어 있어 순서대로 읽을 때 속도가 매우 빠릅니다.

### 배열의 기본 연산 (C++ 예시)

```cpp
#include <vector>
#include <iostream>

int main() {
    std::vector<int> arr = {1, 2, 3, 4, 5};

    // 접근
    std::cout << arr[2] << std::endl;        // 3

    // 끝에 추가
    arr.push_back(6);                         // [1,2,3,4,5,6]

    // 중간에 삽입 (2번 위치에 99 삽입)
    arr.insert(arr.begin() + 2, 99);          // [1,2,99,3,4,5,6]

    // 삭제 (1번 위치 삭제)
    arr.erase(arr.begin() + 1);                // [1,99,3,4,5,6]

    // 값으로 검색 (처음 나오는 위치)
    auto it = std::find(arr.begin(), arr.end(), 99);
    if (it != arr.end()) {
        int idx = it - arr.begin();            // 1
        std::cout << idx << std::endl;
    }
    return 0;
}
```

### 자주 쓰는 배열 패턴

#### 누적합 (Prefix Sum)
배열의 각 위치까지의 합을 미리 계산해두면, 특정 구간의 합을 빠르게 구할 수 있습니다.

```cpp
#include <vector>
#include <iostream>

int main() {
    std::vector<int> arr = {2, 4, 1, 7, 3};
    std::vector<int> prefix(arr.size() + 1, 0);
    for (size_t i = 0; i < arr.size(); ++i)
        prefix[i + 1] = prefix[i] + arr[i];

    // arr[1]부터 arr[3]까지 합 = 4+1+7 = 12
    int sum_1_to_3 = prefix[4] - prefix[1];   // 12
    std::cout << sum_1_to_3 << std::endl;
    return 0;
}
```

#### 슬라이딩 윈도우
연속된 구간(윈도우)을 이동시키면서 문제를 해결하는 방법입니다.  
예: 길이가 k인 부분 배열의 최대 합 구하기

```cpp
#include <vector>
#include <iostream>
#include <algorithm>

int max_sum_k(const std::vector<int>& arr, int k) {
    if (k > (int)arr.size()) return 0;
    int window_sum = 0;
    for (int i = 0; i < k; ++i) window_sum += arr[i];
    int max_sum = window_sum;
    for (int i = k; i < (int)arr.size(); ++i) {
        window_sum += arr[i] - arr[i - k];
        max_sum = std::max(max_sum, window_sum);
    }
    return max_sum;
}

int main() {
    std::vector<int> arr = {1, 4, 2, 10, 23, 3, 1, 0, 20};
    std::cout << max_sum_k(arr, 4) << std::endl;  // 39
    return 0;
}
```

#### 투 포인터
두 개의 인덱스(포인터)를 움직이며 조건을 만족하는 구간을 찾습니다.  
예: 정렬된 배열에서 두 수의 합이 target이 되는 쌍 찾기

```cpp
#include <vector>
#include <iostream>
#include <utility>

std::pair<int, int> two_sum_sorted(const std::vector<int>& arr, int target) {
    int left = 0, right = (int)arr.size() - 1;
    while (left < right) {
        int current = arr[left] + arr[right];
        if (current == target)
            return {left, right};
        else if (current < target)
            ++left;
        else
            --right;
    }
    return {-1, -1}; // 찾지 못함
}

int main() {
    std::vector<int> arr = {2, 7, 11, 15};
    auto [l, r] = two_sum_sorted(arr, 9);
    std::cout << l << ", " << r << std::endl;   // 0, 1
    return 0;
}
```

### 배열 사용 시 주의점

- **인덱스 범위**: 존재하지 않는 인덱스에 접근하면 **미정의 동작**이 발생합니다.  
  `arr[arr.size()]`는 범위를 벗어납니다 (마지막 인덱스는 size()-1).  
  안전한 접근을 위해 `at()` 멤버 함수를 사용할 수 있습니다 (범위 검사).
- **반복자 무효화**: `std::vector`에서 삽입/삭제 시 기존 반복자, 참조, 포인터가 무효화될 수 있으므로 주의해야 합니다.
- **메모리 관리**: `std::vector`는 자동으로 메모리를 관리하지만, `new[]`로 만든 동적 배열은 직접 `delete[]`로 해제해야 합니다.

---

## 문자열

### 문자열이란?

문자열은 **문자의 배열**입니다. C++에서는 두 가지 형태로 자주 사용됩니다:  
- C 스타일 문자열: `char` 배열로 표현되고 `\0`(널 문자)로 끝납니다.  
- C++ 스타일 문자열: `std::string` 클래스로 제공되며 사용이 편리하고 안전합니다.

```cpp
#include <string>
#include <iostream>

int main() {
    std::string s = "hello";
    std::cout << s[1] << std::endl;      // 'e' (인덱스 접근 가능)
    return 0;
}
```

### 문자열의 특징 (`std::string` 기준)

- **가변성**: `std::string`은 내용을 변경할 수 있습니다 (mutable).  
  예: `s[0] = 'H';` 로 첫 글자를 바꿀 수 있습니다.
- **인덱스 접근 가능**: 배열처럼 특정 위치의 문자를 읽고 쓸 수 있습니다.
- **자동 메모리 관리**: 필요에 따라 내부 버퍼가 확장/축소됩니다.
- **다양한 멤버 함수**: 검색, 비교, 부분 문자열 등 풍부한 기능을 제공합니다.

### 문자열 기본 연산 (C++)

```cpp
#include <string>
#include <iostream>

int main() {
    std::string s = "hello";

    // 길이
    std::cout << s.size() << std::endl;      // 5 (또는 s.length())

    // 연결
    std::string t = s + " world";             // "hello world"

    // 반복
    // C++에는 문자열 반복 연산자가 없으므로 반복문 사용 또는 string 생성자 활용
    std::string repeated(3, 'x');             // "xxx" (문자 반복)

    // 포함 여부 (C++23에는 contains()가 추가됨)
    if (s.find('e') != std::string::npos)
        std::cout << "found e" << std::endl;

    // 부분 문자열
    std::string sub = s.substr(1, 3);          // "ell" (위치 1부터 3글자)

    // 분할 (간단한 예: 특정 문자로 분할)
    std::string data = "a,b,c";
    size_t pos = 0;
    std::string token;
    while ((pos = data.find(',')) != std::string::npos) {
        token = data.substr(0, pos);
        std::cout << token << std::endl;       // a, b
        data.erase(0, pos + 1);
    }
    std::cout << data << std::endl;             // c

    // 대체
    s.replace(1, 2, "LL");                      // "hLLlo" (위치 1부터 2글자를 "LL"로 대체)

    // 대소문자 변환 (직접 구현)
    for (char& c : s) c = std::toupper(c);
    std::cout << s << std::endl;                // "HLLLO"

    return 0;
}
```

### 문자열과 배열의 관계

- 문자열은 사실상 **문자의 배열**이므로, 배열에서 쓰는 기법(슬라이딩 윈도우, 투 포인터 등)을 그대로 적용할 수 있습니다.
- `std::string`은 `std::vector<char>`와 유사하게 동작하지만, 문자열 특화 기능이 추가되어 있습니다.

### 자주 쓰는 문자열 패턴

#### 회문(Palindrome) 검사

```cpp
#include <string>
#include <iostream>

bool is_palindrome(const std::string& s) {
    int left = 0, right = (int)s.size() - 1;
    while (left < right) {
        if (s[left] != s[right])
            return false;
        ++left;
        --right;
    }
    return true;
}

int main() {
    std::cout << std::boolalpha;
    std::cout << is_palindrome("racecar") << std::endl;   // true
    std::cout << is_palindrome("hello") << std::endl;     // false
    return 0;
}
```

#### 아나그램(Anagram) 검사
두 문자열이 같은 문자를 같은 개수만큼 가지고 있는지 확인합니다.

```cpp
#include <string>
#include <algorithm>
#include <iostream>

bool is_anagram(std::string s1, std::string s2) {
    if (s1.size() != s2.size()) return false;
    std::sort(s1.begin(), s1.end());
    std::sort(s2.begin(), s2.end());
    return s1 == s2;
}

int main() {
    std::cout << std::boolalpha;
    std::cout << is_anagram("listen", "silent") << std::endl;   // true
    return 0;
}
```

더 효율적인 방법으로는 각 문자의 빈도 수를 배열에 기록하는 방법이 있습니다.

```cpp
#include <string>
#include <array>
#include <iostream>

bool is_anagram_fast(const std::string& s1, const std::string& s2) {
    if (s1.size() != s2.size()) return false;
    std::array<int, 256> count{};   // ASCII 문자 범위
    for (char c : s1) ++count[static_cast<unsigned char>(c)];
    for (char c : s2) {
        if (--count[static_cast<unsigned char>(c)] < 0)
            return false;
    }
    return true;
}
```

#### 접두사/접미사 검사

```cpp
#include <string>
#include <iostream>

int main() {
    std::string s = "hello_world";
    if (s.find("hello") == 0)   // 접두사 검사 (간단한 방법)
        std::cout << "starts with hello" << std::endl;

    // C++20에서는 starts_with(), ends_with() 멤버 함수 제공
    #if __cplusplus >= 202002L
    if (s.starts_with("hello")) std::cout << "yes" << std::endl;
    if (s.ends_with("world")) std::cout << "yes" << std::endl;
    #endif
    return 0;
}
```

### 문자열 사용 시 주의점

- **문자열 연결**: `+` 연산자나 `+=`로 여러 번 연결하면 매번 새로운 문자열이 생성될 수 있습니다.  
  여러 조각을 모아야 한다면 `std::ostringstream`을 사용하는 것이 효율적입니다.

```cpp
#include <sstream>
#include <string>

std::string result;
for (int i = 0; i < 1000; ++i) {
    result += std::to_string(i);   // 매번 재할당 가능성 있음
}

// 더 효율적인 방법
std::ostringstream oss;
for (int i = 0; i < 1000; ++i)
    oss << i;
std::string efficient = oss.str();
```

- **인코딩**: `std::string`은 바이트 시퀀스를 저장합니다. 유니코드 문자열을 다룰 때는 UTF-8 등 인코딩을 고려해야 하며, 문자 단위 처리가 필요하면 `std::wstring` 또는 라이브러리(예: ICU)를 사용할 수 있습니다.

---

## 배열과 문자열을 다루는 기본 알고리즘

### 이진 탐색 (정렬된 배열에서)

```cpp
#include <vector>
#include <iostream>
#include <algorithm>

int binary_search(const std::vector<int>& arr, int target) {
    int left = 0, right = (int)arr.size() - 1;
    while (left <= right) {
        int mid = left + (right - left) / 2;
        if (arr[mid] == target)
            return mid;
        else if (arr[mid] < target)
            left = mid + 1;
        else
            right = mid - 1;
    }
    return -1;
}

int main() {
    std::vector<int> arr = {1, 3, 5, 7, 9, 11};
    std::cout << binary_search(arr, 7) << std::endl;   // 3
    return 0;
}
```

C++ 표준 라이브러리에는 `std::binary_search`, `std::lower_bound` 등이 있습니다.

### 문자열 패턴 찾기 (간단한 naive 방법)

초보자 단계에서는 간단한 방법으로 충분할 때가 많습니다.

```cpp
#include <string>
#include <vector>
#include <iostream>

std::vector<int> naive_search(const std::string& text, const std::string& pattern) {
    std::vector<int> positions;
    int n = text.size(), m = pattern.size();
    for (int i = 0; i <= n - m; ++i) {
        if (text.substr(i, m) == pattern)
            positions.push_back(i);
    }
    return positions;
}

int main() {
    std::string text = "ababcabcabababd";
    std::string pat = "ababd";
    auto res = naive_search(text, pat);
    for (int pos : res) std::cout << pos << " ";   // 10
    return 0;
}
```

---

## 실전 예제

### 예제 1: 배열에서 두 수의 합 (Two Sum)

```cpp
#include <vector>
#include <unordered_map>
#include <iostream>

std::vector<int> two_sum(const std::vector<int>& nums, int target) {
    std::unordered_map<int, int> seen;
    for (int i = 0; i < (int)nums.size(); ++i) {
        int complement = target - nums[i];
        if (seen.count(complement))
            return {seen[complement], i};
        seen[nums[i]] = i;
    }
    return {};
}

int main() {
    std::vector<int> nums = {2, 7, 11, 15};
    auto res = two_sum(nums, 9);
    if (!res.empty())
        std::cout << res[0] << ", " << res[1] << std::endl;   // 0, 1
    return 0;
}
```

### 예제 2: 문자열에서 반복되지 않는 첫 번째 문자 찾기

```cpp
#include <string>
#include <unordered_map>
#include <iostream>

int first_unique_char(const std::string& s) {
    std::unordered_map<char, int> count;
    for (char c : s) ++count[c];
    for (int i = 0; i < (int)s.size(); ++i) {
        if (count[s[i]] == 1)
            return i;
    }
    return -1;
}

int main() {
    std::string s = "loveleetcode";
    std::cout << first_unique_char(s) << std::endl;   // 2 ('v')
    return 0;
}
```

### 예제 3: 배열 회전

```cpp
#include <vector>
#include <algorithm>
#include <iostream>

void rotate(std::vector<int>& nums, int k) {
    k %= nums.size();
    std::reverse(nums.begin(), nums.end());
    std::reverse(nums.begin(), nums.begin() + k);
    std::reverse(nums.begin() + k, nums.end());
}

int main() {
    std::vector<int> arr = {1,2,3,4,5,6,7};
    rotate(arr, 3);
    for (int x : arr) std::cout << x << " ";   // 5 6 7 1 2 3 4
    return 0;
}
```

---

## 마무리

배열과 문자열은 프로그래밍의 가장 기본이 되는 자료구조입니다.  
배열은 **인덱스 접근의 빠름**과 **중간 삽입/삭제의 느림**이라는 트레이드오프를 가지고 있고,  
문자열은 배열의 성질을 가지면서도 문자열 특화 연산들을 제공합니다.