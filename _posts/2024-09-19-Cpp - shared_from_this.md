---
layout: post
title: C++ - shared_from_this
date: 2024-09-19 19:20:23 +0900
category: Cpp
---
# `shared_from_this`와 `enable_shared_from_this`: 자기 참조의 안전한 관리법

## 서론: 왜 이 도구가 필요한가?

객체가 자신에 대한 `shared_ptr`을 만들어야 할 때가 있습니다. 비동기 콜백, 이벤트 시스템, 관찰자 패턴 등에서 객체 스스로 "나를 살려둬"라고 요청해야 하는 상황이죠. 그러나 순진하게 `shared_ptr<T>(this)`를 호출하면 재앙이 벌어집니다.

```cpp
class NetworkConnection {
public:
    std::shared_ptr<NetworkConnection> get_self() {
        return std::shared_ptr<NetworkConnection>(this);  // 치명적인 실수!
    }
};
```

이 코드의 문제는 **같은 객체에 대해 두 개의 독립적인 제어 블록**을 생성한다는 점입니다. 첫 번째 `shared_ptr`이 소멸되어 객체를 삭제한 후, 두 번째 `shared_ptr`이 또다시 삭제를 시도하면 **이중 삭제(double delete)** 가 발생합니다.

## 해결책: `enable_shared_from_this`

C++ 표준 라이브러리는 바로 이 문제를 해결하기 위해 `std::enable_shared_from_this`를 제공합니다:

```cpp
#include <memory>

class NetworkConnection : public std::enable_shared_from_this<NetworkConnection> {
public:
    std::shared_ptr<NetworkConnection> get_self() {
        return shared_from_this();  // 안전한 방법
    }
};
```

### 내부 작동 원리

`enable_shared_from_this`는 마법 같은 도구가 아닙니다. 내부적으로 단순히 약한 포인터(`weak_ptr`)를 유지하면서 동작합니다:

1. 객체가 `shared_ptr`에 의해 처음 관리될 때, 그 제어 블록이 객체 내부의 약한 포인터와 연결됩니다.
2. 이후 `shared_from_this()`가 호출되면, 이 약한 포인터를 통해 이미 존재하는 제어 블록을 찾아 같은 객체를 가리키는 새로운 `shared_ptr`을 생성합니다.
3. 결과적으로 모든 `shared_ptr`이 같은 제어 블록을 공유하게 되어 안전합니다.

## 올바른 사용법: 반드시 지켜야 할 규칙들

### 규칙 1: 이미 `shared_ptr`로 관리되고 있어야 한다

```cpp
// 올바른 사용
auto conn = std::make_shared<NetworkConnection>();
auto self = conn->get_self();  // OK: 이미 shared_ptr로 관리 중

// 잘못된 사용
NetworkConnection* raw = new NetworkConnection;
raw->get_self();  // 런타임 예외 발생!
```

객체가 `shared_ptr`에 의해 관리되기 시작하기 전에는 `shared_from_this()`를 호출할 수 없습니다. 이 규칙을 위반하면 `std::bad_weak_ptr` 예외가 발생합니다.

### 규칙 2: 생성자와 소멸자에서는 호출 금지

생성자에서는 객체가 아직 `shared_ptr`에 완전히 전달되지 않은 상태일 수 있고, 소멸자에서는 객체가 이미 파괴되는 중이기 때문입니다.

```cpp
class Problematic : public std::enable_shared_from_this<Problematic> {
public:
    Problematic() {
        // auto self = shared_from_this();  // 위험: 아직 안전하지 않음
    }
    
    ~Problematic() {
        // auto self = shared_from_this();  // 위험: 파괴 중인 객체
    }
};
```

### 규칙 3: 팩토리 패턴과 함께 사용하기

생성 직후에 자기 참조가 필요한 경우, 팩토리 함수를 사용하는 것이 안전합니다:

```cpp
class Service : public std::enable_shared_from_this<Service> {
private:
    Service() {}  // private 생성자
    
public:
    static std::shared_ptr<Service> create() {
        auto instance = std::make_shared<Service>();
        instance->initialize();  // 생성 후 초기화
        return instance;
    }
    
    void initialize() {
        // 이제 안전하게 shared_from_this() 사용 가능
        auto self = shared_from_this();
        // 초기화 작업 수행...
    }
};
```

## 실전 패턴: 비동기 프로그래밍에서의 활용

### 패턴 1: 콜백에서 객체 수명 유지하기

비동기 작업이 완료될 때까지 객체가 살아있어야 하는 경우:

```cpp
class AsyncProcessor : public std::enable_shared_from_this<AsyncProcessor> {
public:
    void start_processing() {
        auto self = shared_from_this();  // 수명 연장
        
        std::async(std::launch::async, [self]() {
            // self가 캡처되었으므로 작업 완료까지 객체 유지
            self->process_data();
        });
    }
    
private:
    void process_data() {
        // 긴 처리 작업...
    }
};
```

### 패턴 2: 순환 참조 방지하기

객체 간 상호 참조가 발생할 수 있는 경우, `weak_ptr`을 사용하여 순환 참조를 방지합니다:

```cpp
class Observer : public std::enable_shared_from_this<Observer> {
public:
    void subscribe_to(Subject& subject) {
        std::weak_ptr<Observer> weak_self = shared_from_this();
        
        subject.add_callback([weak_self](const Event& e) {
            if (auto self = weak_self.lock()) {
                self->handle_event(e);
            }
            // 객체가 이미 삭제되었다면 아무것도 하지 않음
        });
    }
    
private:
    void handle_event(const Event& e) {
        // 이벤트 처리...
    }
};
```

### 패턴 3: C++17의 `weak_from_this()` 활용

C++17부터는 예외를 던지지 않는 `weak_from_this()`를 사용할 수 있습니다:

```cpp
class Resource : public std::enable_shared_from_this<Resource> {
public:
    std::weak_ptr<Resource> get_weak_reference() noexcept {
        return weak_from_this();  // 예외 없음
    }
};
```

## 고급 주제: 상속과 다형성

### 다중 상속 시 주의사항

다중 상속을 사용할 때는 `enable_shared_from_this`가 한 번만 상속되도록 해야 합니다:

```cpp
// 잘못된 예: 두 번 상속됨
class Base1 : public std::enable_shared_from_this<Base1> {};
class Base2 : public std::enable_shared_from_this<Base2> {};
class Derived : public Base1, public Base2 {};  // 문제 발생!

// 올바른 예: 가상 상속 사용
class CommonBase : public std::enable_shared_from_this<CommonBase> {};
class Base1 : virtual public CommonBase {};
class Base2 : virtual public CommonBase {};
class Derived : public Base1, public Base2 {};  // OK
```

### 파생 클래스에서의 사용

파생 클래스에서도 안전하게 사용할 수 있습니다:

```cpp
class Base : public std::enable_shared_from_this<Base> {
public:
    virtual ~Base() = default;
};

class Derived : public Base {
public:
    std::shared_ptr<Derived> get_derived_self() {
        // Base의 shared_from_this()를 사용한 후 dynamic_pointer_cast
        return std::dynamic_pointer_cast<Derived>(shared_from_this());
    }
};
```

## 실제 시나리오: 네트워크 서버 구현

실제 응용 프로그램에서 어떻게 사용되는지 살펴보겠습니다:

```cpp
#include <memory>
#include <vector>
#include <functional>
#include <iostream>

class Session : public std::enable_shared_from_this<Session> {
    std::vector<std::function<void()>> pending_operations_;
    
public:
    void start() {
        // 비동기 읽기 시작
        async_read([self = shared_from_this()](const std::string& data) {
            self->on_data_received(data);
        });
    }
    
    void async_read(std::function<void(const std::string&)> callback) {
        // 비동기 읽기 시뮬레이션
        pending_operations_.push_back([this, callback]() {
            callback("Hello from server");
        });
    }
    
    void on_data_received(const std::string& data) {
        std::cout << "Received: " << data << std::endl;
        
        // 응답 전송
        async_write("ACK", [self = shared_from_this()](bool success) {
            if (success) {
                self->on_write_complete();
            }
        });
    }
    
    void async_write(const std::string& message, 
                     std::function<void(bool)> callback) {
        // 비동기 쓰기 시뮬레이션
        pending_operations_.push_back([callback]() {
            callback(true);
        });
    }
    
    void on_write_complete() {
        std::cout << "Write completed successfully" << std::endl;
    }
    
    void process_pending() {
        for (auto& op : pending_operations_) {
            op();
        }
        pending_operations_.clear();
    }
};

int main() {
    auto session = std::make_shared<Session>();
    session->start();
    session->process_pending();
    
    return 0;
}
```

## 흔한 실수와 해결책

### 실수 1: 생성자에서 호출하기

```cpp
// 잘못된 코드
class Wrong : public std::enable_shared_from_this<Wrong> {
public:
    Wrong() {
        auto self = shared_from_this();  // 예외 발생!
    }
};

// 해결책: 초기화 메서드 사용
class Correct : public std::enable_shared_from_this<Correct> {
public:
    static std::shared_ptr<Correct> create() {
        auto instance = std::make_shared<Correct>();
        instance->initialize();
        return instance;
    }
    
    void initialize() {
        auto self = shared_from_this();  // 안전
    }
};
```

### 실수 2: 순환 참조 만들기

```cpp
// 잘못된 코드: 순환 참조
class Node {
    std::shared_ptr<Node> next_;  // 강한 참조
public:
    void set_next(std::shared_ptr<Node> next) { next_ = next; }
};

// 해결책: weak_ptr 사용
class SafeNode {
    std::weak_ptr<SafeNode> next_;  // 약한 참조
public:
    void set_next(std::shared_ptr<SafeNode> next) { next_ = next; }
};
```

## 디버깅과 테스트 팁

### use_count() 모니터링

```cpp
auto obj = std::make_shared<MyClass>();
std::cout << "Initial use_count: " << obj.use_count() << std::endl;

auto another_ref = obj->get_self();
std::cout << "After get_self: " << obj.use_count() << std::endl;
```

### 커스텀 삭제자로 메모리 누수 감지

```cpp
auto deleter = [](MyClass* ptr) {
    std::cout << "Deleting MyClass@" << ptr << std::endl;
    delete ptr;
};

std::shared_ptr<MyClass> obj(new MyClass, deleter);
```

## 결론: 현명하게 사용하기 위한 원칙

`enable_shared_from_this`는 강력한 도구이지만, 올바르게 사용해야 합니다:

### 1. **필요할 때만 사용하세요**
   - 단순한 객체라면 `shared_ptr`만으로 충분합니다.
   - 객체가 자기 자신을 참조해야 하는 특별한 경우에만 사용하세요.

### 2. **수명 관리 전략을 명확히 하세요**
   - 비동기 콜백에서는 `shared_from_this()`로 수명을 연장하세요.
   - 관찰자 패턴에서는 `weak_ptr`로 순환 참조를 방지하세요.

### 3. **안전한 생성 패턴을 사용하세요**
   - 생성자에서는 절대 호출하지 마세요.
   - 팩토리 함수와 초기화 메서드를 활용하세요.

### 4. **상속 구조를 단순하게 유지하세요**
   - 다중 상속 시 가상 상속을 사용하세요.
   - 기본 클래스 하나에서만 상속받도록 하세요.

### 5. **모던 C++ 기능을 활용하세요**
   - C++17의 `weak_from_this()`를 사용하세요.
   - `std::async`나 스레드와 함께 사용할 때는 특히 주의하세요.

이 도구의 본질은 **객체 수명 관리의 명시성과 안전성**에 있습니다. 올바르게 사용하면 복잡한 비동기 프로그램에서도 메모리 누수와 댕글링 포인터로부터 안전하게 보호받을 수 있습니다. 하지만 마치 강력한 약처럼, 적절한 경우에만 신중하게 사용해야 합니다.

기억하세요: 가장 좋은 코드는 `shared_from_this`가 필요 없는 코드입니다. 그러나 필요할 때는 두려움 없이 사용하되, 이 글의 원칙들을 명심하세요.