---
layout: post
title: C++ - shared_from_this
date: 2024-09-19 19:20:23 +0900
category: Cpp
---
# `shared_from_this`와 `enable_shared_from_this`

스마트 포인터로 객체를 관리하다 보면, 객체 **자기 자신에 대한 `shared_ptr`을 생성하고 싶은 상황**이 생깁니다.  
이럴 때 사용하는 것이 바로 `shared_from_this()`입니다.

---

## 왜 필요한가?

```cpp
struct MyClass {
    shared_ptr<MyClass> getSelf() {
        return shared_ptr<MyClass>(this);  // ❌ 위험한 코드!
    }
};
```

### 문제점:

- 이미 `shared_ptr`로 관리 중인 객체에서 다시 `shared_ptr(this)`를 만들면, **참조 카운트가 꼬여서 이중 삭제(double free)** 발생 가능!
- 그 이유는 같은 객체를 관리하는 shared_ptr가 2개 생깁니다. 각 shared_ptr는 서로 독립적인 참조 카운트를 가집니다.

```cpp
shared_ptr<MyClass> p1 = make_shared<MyClass>();
shared_ptr<MyClass> p2 = p1->getSelf(); // p2도 p1과 같은 객체를 가리키지만, 참조 카운트가 공유되지 않음

// p1, p2가 각각 소멸될 때 같은 객체를 두 번 delete!
```

---

## 안전한 방법: `enable_shared_from_this`

> C++ 표준 라이브러리에 포함된 `enable_shared_from_this`를 상속하면,  
> 객체 자신을 안전하게 `shared_ptr`로 가져올 수 있습니다.

```cpp
#include <memory>
using namespace std;

struct MyClass : enable_shared_from_this<MyClass> {
    shared_ptr<MyClass> getSelf() {
        return shared_from_this();  // 안전하게 자기 자신 반환
    }
};
```

---

## 3. 사용 예시

```cpp
int main() {
    auto obj = make_shared<MyClass>();
    auto self = obj->getSelf();

    cout << "use_count: " << self.use_count();  // 2
}
```

- `obj`와 `self`가 같은 객체를 가리킴
- 참조 카운트는 2가 되며, 공유 소유 상태가 유지됨

---

## 주의: `shared_from_this()`는 shared_ptr로 만들어야만 동작

```cpp
MyClass* raw = new MyClass();
auto self = raw->shared_from_this();  // ❌ 예외 발생!
```

- `shared_from_this()`는 내부적으로 `weak_ptr`을 통해 참조 카운트를 확인
- `shared_ptr`로 생성하지 않으면 작동할 수 없음

---

## 요약

| 개념 | 설명 |
|------|------|
| `shared_ptr(this)` | 직접 사용하면 참조 카운트 오류 가능 |
| `enable_shared_from_this` | 안전한 자기 참조를 위해 상속 |
| `shared_from_this()` | 내부에서 자기 자신을 `shared_ptr`로 반환 |
| 조건 | 반드시 `shared_ptr`로 객체를 생성해야 함 (`make_shared` 등) |

---

## 사용 예시: 콜백 전달 등

```cpp
#include <iostream>
#include <memory>
#include <functional>
#include <string>
using namespace std;

// 모의 asyncRead 함수: 콜백에 "Hello, World!"를 전달
void asyncRead(function<void(const string&)> callback) {
    // 실제 네트워크에서는 비동기로 데이터가 도착할 때 콜백이 호출됨
    callback("Hello, World!");
}

class Connection : public enable_shared_from_this<Connection> {
public:
    void start() {
        auto self = shared_from_this();  // 안전하게 자기 자신을 shared_ptr로 보관
        asyncRead([self](const string& data) {
            self->handle(data);
        });
    }

    void handle(const string& msg) {
        cout << "Got: " << msg << endl;
    }
};

int main() {
    // Connection 객체를 shared_ptr로 생성
    auto conn = make_shared<Connection>();

    // 비동기 작업 시작
    conn->start();

    // 실제 네트워크라면 이벤트 루프가 필요하겠지만,
    // 여기서는 콜백이 즉시 실행되므로 바로 종료됨
    return 0;
}
```

---

## 마무리

- `shared_from_this`는 **객체 자신을 안전하게 참조**해야 할 때 필수 도구입니다.
- 반드시 `make_shared`로 생성해야 안전하게 동작합니다.
- 직접 `shared_ptr(this)`를 만드는 건 절대 피해야 합니다.