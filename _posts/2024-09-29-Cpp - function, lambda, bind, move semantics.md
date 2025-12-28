---
layout: post
title: C++ - function, lambda, bind, move semantics
date: 2024-09-29 19:20:23 +0900
category: Cpp
---
# 함수 객체와 이동 의미론: C++의 현대적 추상화와 성능

## 서문: 함수를 객체로, 객체를 함수처럼

C++은 함수를 일급 객체(first-class citizen)로 다루는 강력한 메커니즘을 제공합니다. 이는 단순한 코드 구조화를 넘어, 런타임 다형성, 지연 실행, 그리고 성능 최적화의 핵심 도구가 됩니다. 람다, `std::function`, 이동 의미론은 이러한 패러다임의 중심에 있습니다.

---

## 제1장: `std::function` - 타입을 지우는 래퍼의 마법

### 왜 `std::function`이 필요한가?

다양한 종류의 호출 가능 객체(함수 포인터, 람다, 함수 객체)를 하나의 통일된 타입으로 저장하고 싶을 때가 있습니다. 콜백 시스템, 이벤트 핸들러, 전략 패턴 등이 대표적인 예입니다.

```cpp
#include <functional>
#include <iostream>
#include <vector>

// 다양한 호출자를 저장하는 컨테이너
std::vector<std::function<void()>> callbacks;

// 함수 포인터
void hello() { std::cout << "Hello\n"; }

// 함수 객체
struct Goodbye {
    void operator()() const { std::cout << "Goodbye\n"; }
};

int main() {
    // 다양한 타입을 한 컨테이너에 저장
    callbacks.push_back(hello);
    callbacks.push_back(Goodbye{});
    callbacks.push_back([] { std::cout << "Lambda\n"; });
    
    // 모두 동일한 방식으로 호출
    for (const auto& callback : callbacks) {
        callback();
    }
}
```

### 내부 동작: 타입 소거의 비밀

`std::function`은 타입 소거(Type Erasure) 패턴을 사용합니다. 이는 서로 다른 타입의 객체를 동일한 인터페이스 뒤에 감추는 기술입니다.

```cpp
// 개념적인 내부 구조
class function_impl_base {
public:
    virtual ~function_impl_base() = default;
    virtual void call() const = 0;
    virtual std::unique_ptr<function_impl_base> clone() const = 0;
};

template<typename Callable>
class function_impl : public function_impl_base {
    Callable callable_;
public:
    explicit function_impl(Callable c) : callable_(std::move(c)) {}
    void call() const override { callable_(); }
    std::unique_ptr<function_impl_base> clone() const override {
        return std::make_unique<function_impl>(*this);
    }
};
```

**핵심**: 모든 호출 가능 객체는 내부 구현 클래스로 감싸지고, 기본 클래스 포인터를 통해 접근됩니다.

### 성능 고려사항

`std::function`은 편리하지만 비용이 있습니다:
1. **간접 호출**: 가상 함수 호출 오버헤드
2. **메모리 할당**: 큰 객체는 힙에 할당될 수 있음
3. **인라인 최적화 어려움**: 컴파일러가 호출을 인라인화하기 어려움

**성능 팁**:
- 작은 객체에 대한 SBO(Small Buffer Optimization)를 활용하세요
- 성능이 중요한 루프에서는 템플릿을 사용하세요

```cpp
// 성능이 중요한 경우: 템플릿 사용
template<typename F>
void fast_process(F&& func) {
    for (int i = 0; i < 1000000; ++i) {
        func(i);  // 인라인 최적화 가능
    }
}

// 유연성이 필요한 경우: std::function 사용
void flexible_process(std::function<void(int)> func) {
    for (int i = 0; i < 1000000; ++i) {
        func(i);  // 간접 호출
    }
}
```

---

## 제2장: 람다 - 익명 함수의 힘

### 람다의 진화: C++11에서 C++20까지

```cpp
// C++11: 기본 람다
auto add = [](int a, int b) { return a + b; };

// C++14: 제네릭 람다와 초기화 캡처
auto generic_add = [](auto a, auto b) { return a + b; };
auto with_init = [value = std::make_unique<int>(42)] { return *value; };

// C++17: constexpr 람다
constexpr auto constexpr_square = [](int n) { return n * n; };
static_assert(constexpr_square(5) == 25);

// C++20: 템플릿 람다와 명시적 객체 매개변수
auto template_lambda = []<typename T>(T a, T b) { return a + b; };
auto explicit_object = [](this auto&& self, int x) { return x * 2; };
```

### 캡처: 값과 참조의 균형

캡처는 람다의 가장 강력한 기능이자 가장 위험한 함정입니다.

```cpp
#include <iostream>
#include <memory>
#include <functional>

void demonstrate_captures() {
    int local_value = 42;
    int& ref = local_value;
    
    // 값 캡처: 안전하지만 비용이 있음
    auto value_capture = [local_value]() {
        return local_value;  // 복사본 사용
    };
    
    // 참조 캡처: 위험하지만 효율적
    auto reference_capture = [&ref]() {
        return ref;  // 원본 참조 - 수명 주의!
    };
    
    // 초기화 캡처: 이동 의미론 활용
    auto expensive = std::make_unique<int>(100);
    auto move_capture = [data = std::move(expensive)]() {
        return *data;  // 소유권 이전
    };
    
    // 멤버 변수 캡처
    struct Widget {
        int id = 10;
        std::function<int()> get_id() const {
            return [this] { return id; };  // this 캡처 - 위험!
        }
    };
}
```

### 람다의 실제 타입 이해하기

```cpp
auto lambda1 = [] { return 1; };
auto lambda2 = [] { return 1; };

// 각 람다는 고유한 타입을 가집니다
static_assert(!std::is_same_v<decltype(lambda1), decltype(lambda2)>);

// std::function으로 타입을 통일할 수 있습니다
std::function<int()> f1 = lambda1;
std::function<int()> f2 = lambda2;
static_assert(std::is_same_v<decltype(f1), decltype(f2)>);
```

---

## 제3장: 이동 의미론 - 성능 혁명

### 이동 의미론이 왜 중요한가?

C++11 이전에는 큰 객체를 반환하거나 전달할 때 불필요한 복사가 발생했습니다. 이동 의미론은 이러한 비효율을 해결합니다.

```cpp
#include <vector>
#include <string>
#include <chrono>
#include <iostream>

class HeavyObject {
    std::vector<int> data_;
public:
    HeavyObject(size_t size) : data_(size) {
        std::iota(data_.begin(), data_.end(), 0);
    }
    
    // 이동 생성자
    HeavyObject(HeavyObject&& other) noexcept 
        : data_(std::move(other.data_)) {}
    
    // 이동 대입 연산자
    HeavyObject& operator=(HeavyObject&& other) noexcept {
        if (this != &other) {
            data_ = std::move(other.data_);
        }
        return *this;
    }
    
    // 복사 생성자/대입 연산자는 비활성화
    HeavyObject(const HeavyObject&) = delete;
    HeavyObject& operator=(const HeavyObject&) = delete;
};

// 이동을 활용한 효율적인 반환
HeavyObject create_heavy_object() {
    HeavyObject obj(1'000'000);
    // RVO(Return Value Optimization) 또는 이동 발생
    return obj;
}
```

### 이동 vs 복사: 올바른 선택

```cpp
void process_objects() {
    std::vector<std::string> names;
    
    // 복사: 원본 유지 필요
    std::string original = "Hello";
    names.push_back(original);  // 원본 유지
    
    // 이동: 원본 더 이상 필요 없음
    std::string temporary = "World";
    names.push_back(std::move(temporary));  // 소유권 이전
    // temporary는 더 이상 사용하면 안 됨
}
```

### `noexcept` 이동의 중요성

컨테이너는 `noexcept` 이동 생성자를 선호합니다. 예외를 던질 수 있는 이동 연산은 컨테이너 재배치 시 비효율적인 복사로 대체될 수 있습니다.

```cpp
class SafeToMove {
    std::unique_ptr<int[]> data_;
    size_t size_;
public:
    // noexcept 이동 생성자
    SafeToMove(SafeToMove&& other) noexcept 
        : data_(std::move(other.data_))
        , size_(other.size_) {
        other.size_ = 0;
    }
    
    // noexcept 이동 대입 연산자
    SafeToMove& operator=(SafeToMove&& other) noexcept {
        if (this != &other) {
            data_ = std::move(other.data_);
            size_ = other.size_;
            other.size_ = 0;
        }
        return *this;
    }
};
```

---

## 제4장: 실전 패턴과 최적화

### 패턴 1: 이벤트 시스템 구현

```cpp
#include <functional>
#include <vector>
#include <memory>
#include <algorithm>

class EventSystem {
    struct EventHandler {
        size_t id;
        std::function<void(int)> callback;
    };
    
    std::vector<EventHandler> handlers_;
    size_t next_id_ = 0;
    
public:
    // 템플릿을 사용한 등록 (성능 최적화)
    template<typename F>
    size_t register_handler(F&& callback) {
        handlers_.push_back({
            next_id_++,
            std::forward<F>(callback)
        });
        return handlers_.back().id;
    }
    
    // 이벤트 발행
    void emit(int event_data) {
        // 핸들러 호출 중 삭제를 안전하게 처리
        for (size_t i = 0; i < handlers_.size(); ) {
            try {
                handlers_[i].callback(event_data);
                ++i;
            } catch (...) {
                // 예외 발생 시 핸들러 제거
                handlers_.erase(handlers_.begin() + i);
            }
        }
    }
    
    // 핸들러 제거
    void unregister_handler(size_t id) {
        handlers_.erase(
            std::remove_if(handlers_.begin(), handlers_.end(),
                [id](const EventHandler& h) { return h.id == id; }),
            handlers_.end()
        );
    }
};
```

### 패턴 2: 지연 계산과 메모이제이션

```cpp
#include <functional>
#include <unordered_map>
#include <mutex>

template<typename Result, typename... Args>
class Memoizer {
    mutable std::mutex mutex_;
    mutable std::unordered_map<std::tuple<Args...>, Result> cache_;
    std::function<Result(Args...)> function_;
    
public:
    Memoizer(std::function<Result(Args...)> func) 
        : function_(std::move(func)) {}
    
    Result operator()(Args... args) const {
        auto key = std::make_tuple(args...);
        
        {
            std::lock_guard lock(mutex_);
            auto it = cache_.find(key);
            if (it != cache_.end()) {
                return it->second;
            }
        }
        
        Result result = function_(args...);
        
        {
            std::lock_guard lock(mutex_);
            cache_.emplace(std::move(key), result);
        }
        
        return result;
    }
};

// 사용 예시
auto expensive_computation = [](int x) -> int {
    // 무거운 계산
    return x * x;
};

auto memoized = Memoizer<int, int>(expensive_computation);
int result1 = memoized(5);  // 계산 수행
int result2 = memoized(5);  // 캐시에서 반환
```

### 패턴 3: 빌더 패턴과 이동 의미론

```cpp
class QueryBuilder {
    std::string select_;
    std::string from_;
    std::vector<std::string> conditions_;
    
public:
    QueryBuilder& select(std::string columns) {
        select_ = std::move(columns);
        return *this;
    }
    
    QueryBuilder& from(std::string table) {
        from_ = std::move(table);
        return *this;
    }
    
    QueryBuilder& where(std::string condition) {
        conditions_.push_back(std::move(condition));
        return *this;
    }
    
    std::string build() {
        std::string query = "SELECT " + select_ + " FROM " + from_;
        if (!conditions_.empty()) {
            query += " WHERE ";
            for (size_t i = 0; i < conditions_.size(); ++i) {
                if (i > 0) query += " AND ";
                query += conditions_[i];
            }
        }
        return query;
    }
};

// 사용
auto query = QueryBuilder{}
    .select("id, name, email")
    .from("users")
    .where("active = 1")
    .where("created > '2023-01-01'")
    .build();
```

---

## 제5장: 함정과 안전 조치

### 함정 1: 댕글링 참조

```cpp
// 위험한 코드
std::function<int()> create_dangerous_function() {
    int local = 42;
    return [&local] { return local; };  // 지역 변수 참조 캡처
}

// 안전한 코드
std::function<int()> create_safe_function() {
    int local = 42;
    return [local] { return local; };  // 값 캡처
    
    // 또는 shared_ptr 사용
    auto shared = std::make_shared<int>(42);
    return [shared] { return *shared; };
}
```

### 함정 2: 이동 후 사용

```cpp
void misuse_of_move() {
    std::string important = "Important data";
    
    // 잘못된 사용: 이동 후 접근
    std::string stolen = std::move(important);
    std::cout << important;  // 정의되지 않은 동작!
    
    // 올바른 사용
    std::string another = std::move(stolen);
    // stolen은 더 이상 접근하지 않음
}
```

### 함정 3: 성능 오버헤드 무시

```cpp
// 성능이 중요한 루프
void performance_sensitive_loop() {
    // 나쁜 예: 매번 std::function 생성
    for (int i = 0; i < 1000000; ++i) {
        std::function<int(int)> f = [](int x) { return x * x; };
        int result = f(i);
    }
    
    // 좋은 예: 루프 밖에서 한 번 생성
    auto lambda = [](int x) { return x * x; };
    for (int i = 0; i < 1000000; ++i) {
        int result = lambda(i);  // 인라인 최적화 가능
    }
}
```

---

## 결론: 현명한 선택을 위한 가이드라인

### 1. **도구 선택의 지혜**

각 상황에 맞는 도구를 선택하세요:
- **간단한 콜백**: 인라인 람다
- **저장해야 하는 콜백**: `std::function` 또는 템플릿
- **성능이 중요**: 템플릿과 정적 다형성
- **다양성과 편의성**: `std::function`과 타입 소거

### 2. **수명 관리의 원칙**

메모리 안전성을 위한 원칙:
1. 람다에서 참조 캡처는 원본 객체의 수명을 신중히 고려하세요
2. 비동기 콜백에서는 값 캡처 또는 `shared_ptr`을 사용하세요
3. 이동 후 객체는 접근하지 마세요

### 3. **성능 최적화의 비결**

성능을 위한 실용적인 조언:
1. 작은 객체는 값으로, 큰 객체는 참조나 이동으로 전달하세요
2. `noexcept` 이동 생성자를 제공하세요
3. 성능이 중요한 경로에서는 가상 함수 호출을 피하세요
4. RVO(Return Value Optimization)를 신뢰하고 방해하지 마세요

### 4. **코드 품질의 지표**

읽기 좋고 유지보수 가능한 코드를 위한 지침:
1. 람다의 본문이 짧게 유지되도록 하세요 (10줄 이내 권장)
2. 복잡한 로직은 명명된 함수나 함수 객체로 분리하세요
3. 캡처 목록을 명시적으로 작성하세요 (`[=]`, `[&]` 피하기)
4. 이동 의미론은 명확한 의도가 있을 때만 사용하세요

### 5. **현대적 관용구 활용**

C++17/20의 새로운 기능을 활용하세요:
- `std::bind` 대신 람다나 `std::bind_front` 사용
- 제네릭 람다로 템플릿 간소화
- `constexpr` 람다로 컴파일 타임 계산

함수 객체와 이동 의미론은 C++의 표현력과 성능을 결합하는 강력한 패러다임입니다. 이들을 올바르게 이해하고 적용하면, 더 안전하고 빠르며 표현력 있는 코드를 작성할 수 있습니다. 가장 중요한 것은 각 도구의 장단점을 이해하고, 상황에 맞게 현명하게 선택하는 것입니다.

기억하세요: 최고의 코드는 단순히 동작하는 코드가 아니라, **왜 그렇게 동작하는지 이해할 수 있는 코드**입니다. 함수 객체와 이동 의미론을 깊이 이해하면, C++의 추상화와 성능 모두를 잡을 수 있는 진정한 전문가로 성장할 수 있습니다.