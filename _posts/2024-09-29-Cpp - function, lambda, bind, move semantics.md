---
layout: post
title: C++ - function, lambda, bind, move semantics
date: 2024-09-29 19:20:23 +0900
category: Cpp
---
# C++ 함수 객체와 이동 의미론: std::function, lambda, bind, move semantics

---

## 1. `std::function`: 범용 함수 래퍼

`std::function`은 함수, 람다, 함수 포인터, functor 등을 **하나의 타입으로 통일**하여 다룰 수 있게 해줍니다.

```cpp
#include <functional>
#include <iostream>

void greet() {
    std::cout << "Hello!\n";
}

int main() {
    std::function<void()> f = greet;
    f();  // Hello!
}
```

### 다양한 타입을 래핑 가능

```cpp
void hello() { std::cout << "Hello\n"; }

struct Bye {
    void operator()() { std::cout << "Bye\n"; }
};

int main() {
    std::function<void()> f1 = hello;
    std::function<void()> f2 = Bye();
    std::function<void()> f3 = []() { std::cout << "Lambda!\n"; };

    f1(); f2(); f3();
}
```

---

## 2. 람다(lambda): 가볍고 강력한 익명 함수

람다는 `[]` 문법을 이용해 캡처와 함께 익명 함수를 선언할 수 있습니다.

```cpp
auto add = [](int a, int b) {
    return a + b;
};

std::cout << add(3, 4);  // 7
```

### 캡처(Capture)

```cpp
int x = 10;

auto f1 = [x]() { std::cout << x << "\n"; };   // 값 캡처
auto f2 = [&x]() { x += 5; };                  // 참조 캡처

f1();  // 10
f2();
std::cout << x;  // 15
```

### 사용 예: 정렬 기준

```cpp
std::vector<int> v = {3, 1, 4};

std::sort(v.begin(), v.end(), [](int a, int b) {
    return a > b;  // 내림차순
});
```

---

## 3. `std::bind`: 함수 인자 고정 및 재배열

```cpp
#include <functional>

int add(int a, int b) {
    return a + b;
}

int main() {
    auto add5 = std::bind(add, 5, std::placeholders::_1);
    std::cout << add5(3);  // 8
}
```

### 기능 요약

| 기능            | 설명                           |
|-----------------|--------------------------------|
| 인자 고정       | 특정 인자를 고정 (`bind(f, 5, ...)`) |
| 자리 지정자 사용 | `_1`, `_2`, ... 으로 순서 지정   |
| 멤버 함수 바인딩 | `bind(&Class::func, 객체, ...)`  |

### 주의

- C++11의 유산 스타일, 람다가 더 가독성 좋고 일반적으로 추천됨

---

## 4. 이동 의미론 (Move Semantics)

### 복사 vs 이동

- **복사(copy)**: 원본을 그대로 두고 새 객체에 복제  
- **이동(move)**: 원본의 자원을 **이전**하고 원본은 비워짐

```cpp
std::string a = "Hello";
std::string b = std::move(a);  // a는 비워짐
```

### `std::move`

```cpp
std::vector<std::string> names;
names.push_back(std::string("Tom"));             // 복사
names.push_back(std::move(std::string("Jerry"))); // 이동
```

### 사용자 정의 이동 생성자 예시

```cpp
class Buffer {
    char* data;
public:
    Buffer(size_t size) : data(new char[size]) {}
    ~Buffer() { delete[] data; }

    // 이동 생성자
    Buffer(Buffer&& other) noexcept : data(other.data) {
        other.data = nullptr;
    }

    // 복사 방지
    Buffer(const Buffer&) = delete;
    Buffer& operator=(const Buffer&) = delete;
};
```

---

## 5. 정리표

| 개념            | 설명                                              | 추천 상황                      |
|-----------------|---------------------------------------------------|--------------------------------|
| `std::function` | 다양한 호출 객체를 하나로 묶어 사용               | 콜백, 함수 저장 시             |
| `lambda`        | 캡처와 함께 짧고 간결한 함수 표현                 | 커스텀 정렬, 이벤트 핸들러 등  |
| `std::bind`     | 인자 고정, 멤버 함수 바인딩                       | 레거시 코드 또는 콜백 구성     |
| `move semantics`| 자원을 복사하지 않고 이전                        | 성능 최적화, 자원 소유권 이전 |

---

## 6. 마무리

- 이 네 가지는 현대 C++ 함수형 스타일의 핵심입니다.
- 특히 `lambda` + `std::function` 조합은 콜백, 커스텀 연산자 등에서 가장 많이 사용됩니다.
- `move semantics`는 대용량 객체 처리 시 꼭 필요한 성능 도구입니다.