---
layout: post
title: C++ - string
date: 2024-09-04 19:20:23 +0900
category: Cpp
---
# 문자열(string)

C++에서는 문자열을 다룰 때 예전에는 `char[]`이나 `C-style 문자열 (char*)` 을 사용했지만,  
지금은 대부분 **`std::string` 클래스**를 사용합니다.

---

## 1. string의 기본 사용

```cpp
#include <string>
#include <iostream>
using namespace std;

int main() {
    string s = "Hello";
    s += " World";  // 문자열 덧붙이기
    cout << s;      // Hello World
}
```

---

## 2. 주요 기능

### 문자열 길이

```cpp
s.length();   // 또는 s.size();
```

### 부분 문자열 추출

```cpp
string s = "abcdef";
string sub = s.substr(1, 3);  // "bcd"
```

### 문자열 비교

```cpp
if (s1 == s2) { ... }  // ==, !=, <, > 가능
```

### 문자 접근

```cpp
char c = s[0];      // 인덱스로 접근
s[1] = 'a';         // 수정 가능
```

---

## 3. 문자열 찾기 & 변경

### 문자열 찾기: `find()`

```cpp
#include <iostream>
#include <string>
using namespace std;

int main() {
    string s = "hello world";

    // "world"라는 부분 문자열을 s에서 찾음
    size_t pos = s.find("world");  // pos = 6

    // 찾았는지 확인
    if (pos != string::npos) {
        cout << "찾음" << endl;
    } else {
        cout << "못 찾음" << endl;
    }

    return 0;
}
```

### 문자열 교체: `replace()`

```cpp
string s = "hello world";
s.replace(0, 5, "hi");  // "hi world"
```

---

## 4. 문자열 삭제 및 삽입

```cpp
string s = "hello world";
s.replace(0, 5, "hi");  // "hi world"
```

---

## 5. 숫자 <-> 문자열 변환

```cpp
#include <iostream>
#include <string>
using namespace std;

int main() {
    int x = 42;
    string s = to_string(x);  // 정수 → 문자열 ("42")
    cout << "s: " << s << endl;

    double d = stod("3.14");  // 문자열 → double (3.14)
    cout << "d: " << d << endl;

    int i = stoi("123");      // 문자열 → int (123)
    cout << "i: " << i << endl;

    return 0;
}
```

---

## 6. 문자열 반복 & 전처리

### 문자 하나씩 순회

```cpp
for (char c : s) {
    cout << c;
}
```

### 공백 기준으로 자르기 (istringstream)

```cpp
#include <sstream>
string line = "abc def ghi";
istringstream iss(line);
string word;
while (iss >> word) {
    cout << word << endl;
}
```

---

## 7. string과 C 문자열(char*) 비교

| 항목         | char* 또는 char[] | std::string     |
|--------------|--------------------|-----------------|
| 안전성        | 낮음 (버퍼 초과 등 위험) | 높음 (자동 메모리 관리) |
| 기능성        | 제한적             | 풍부한 메서드 제공 |
| 편리성        | 수동 처리 필요       | 직관적인 연산 가능 |

---

## 8. 마무리

- `std::string`은 C++에서 문자열 처리의 핵심 도구입니다.
- 대부분의 경우 `char*` 대신 `string`을 사용하는 것이 **안전하고 효율적**입니다.
- STL 알고리즘 (`sort`, `find_if`) 와 함께 사용할 수도 있습니다.