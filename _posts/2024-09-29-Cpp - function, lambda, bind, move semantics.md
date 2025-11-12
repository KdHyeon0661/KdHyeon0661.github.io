---
layout: post
title: C++ - function, lambda, bind, move semantics
date: 2024-09-29 19:20:23 +0900
category: Cpp
---
# C++ 함수 객체와 이동 의미론: std::function, lambda, bind, move semantics

현대 C++는 **함수 객체(Functor)**, **람다(lambda)**, **타입 소거 기반 래퍼(std::function)**, **std::bind / std::bind_front**, 그리고 **이동 의미론(move semantics)**을 통해
표현력과 성능을 모두 잡습니다.  
이 글은 네 가지 축을 **원리 → 올바른 사용 → 함정과 대응 → 실전 패턴** 순으로 정리합니다.

---

## 1. `std::function` — 범용 호출자(Callable) 래퍼

`std::function<R(Args...)>`는 **함수 포인터, 람다, 멤버 함수 포인터, 함수 객체** 등 서로 다른 호출자를 *하나의 타입*으로 담는 래퍼입니다.  
핵심은 **타입 소거(Type Erasure)**: 내부에 “호출 대상과 호출 방법”을 보관하고, 외부에는 `R(Args...)` 시그니처만 노출합니다.

```cpp
#include <functional>
#include <iostream>

void greet() { std::cout << "Hello!\n"; }

int main() {
    std::function<void()> f = greet;  // 함수 포인터 저장
    f();                               // Hello!
}
```

### 1.1 다양한 호출자 수용

```cpp
#include <functional>
#include <iostream>

void hello() { std::cout << "Hello\n"; }

struct Bye {
    void operator()() const { std::cout << "Bye\n"; }
};

int main() {
    std::function<void()> f1 = hello;           // 함수
    std::function<void()> f2 = Bye{};           // 함수 객체
    std::function<void()> f3 = []{ std::cout << "Lambda!\n"; }; // 람다

    f1(); f2(); f3();
}
```

### 1.2 성능 관점(소규모 최적화와 할당)
- 구현에 따라 **Small Buffer Optimization(SBO)** 가 있을 수 있으나, *표준이 보장하진 않습니다*.  
  캡처가 큰 람다/펑터는 **동적 할당**(heap)이 발생할 수 있습니다.
- 성능 민감 경로(핫 루프)에서는  
  - 캡처를 줄이거나,  
  - 호출자 타입을 템플릿 매개변수로 받는 **정적 다형성(템플릿)**,  
  - `auto` 템플릿 매개변수(CTAD/제네릭 람다) 등을 고려하세요.

### 1.3 상태 확인 및 소거된 시그니처의 제약
```cpp
std::function<int(int)> f;      // empty 상태
if (!f) { /* 미설정 */ }

f = [](int x){ return x * 2; }; // 시그니처는 int(int)로 고정
// f에 다른 인자 구조의 Callable은 대입 불가
```

---

## 2. 람다(lambda) — 캡처 가능한 익명 함수

람다는 `[]` 문법으로 **캡처(capture)** 를 지정하고, **익명 함수**를 즉석에서 정의합니다.

```cpp
auto add = [](int a, int b) { return a + b; };
std::cout << add(3, 4) << '\n';  // 7
```

### 2.1 캡처 방법
```cpp
int x = 10, y = 20;

auto f1 = [x]      { std::cout << x << "\n"; };     // 값 캡처(읽기 전용)
auto f2 = [&x]     { ++x; };                        // 참조 캡처(쓰기 가능)
auto f3 = [=]      { return x + y; };               // 주변 모두 값 캡처(읽기)
auto f4 = [&]      { x += y; };                     // 주변 모두 참조 캡처
auto f5 = [z = std::move(x)] { /* 초기화 캡처 C++14+ */ };
// f5 안에서 z는 x에서 이동한 값으로 초기화됨
```

> **주의**: 참조 캡처는 원본 객체의 수명이 더 길다는 보장이 있어야 합니다.  
> 비동기/지연 실행 콜백에서 지역 변수를 참조 캡처하면 **댕글링 참조** 위험이 큽니다.  
> 이 경우 **값 캡처** 또는 `std::shared_ptr` + **값 캡처**(또는 `enable_shared_from_this`) 패턴이 안전합니다.

### 2.2 `mutable`, `constexpr`, 제네릭 람다
```cpp
int c = 0;
auto inc = [c]() mutable { ++c; return c; }; // 캡처 사본 수정
std::cout << inc() << " " << inc() << "\n";  // 1 2 (외부 c는 그대로)

// 제네릭 람다(C++14+)
auto twice = [](auto v) { return v + v; }; // 템플릿처럼 동작
static_assert(std::is_same_v<decltype(twice(1)), int>);
```

### 2.3 람다와 수명 관리: 안전한 비동기 콜백
```cpp
#include <memory>
#include <functional>

struct Worker : std::enable_shared_from_this<Worker> {
    void start(std::function<void(std::string)> post) {
        auto self = shared_from_this(); // 안전한 자기 생존 보장
        // 비동기 완료 시점에 self를 캡처하여 생존
        asyncRead([post, self](const std::string& s) { post(">> " + s); });
    }
};
```

---

## 3. `std::bind`와 `std::bind_front` — 인자 고정/재배열

람다가 보편화되며 `std::bind`는 **대부분 람다로 대체** 가능합니다.  
다만 레거시 코드와의 결합, 멤버 함수 포인터 바인딩, 자리 지정자 조합 등에서 여전히 볼 수 있습니다.

```cpp
#include <functional>
#include <iostream>

int add(int a, int b) { return a + b; }

int main() {
    auto add5 = std::bind(add, 5, std::placeholders::_1); // 앞 인자 5로 고정
    std::cout << add5(3) << '\n';  // 8
}
```

### 3.1 멤버 함수 포인터 바인딩
```cpp
struct Calc {
    int mul(int a, int b) const { return a * b; }
};

Calc c;
auto f = std::bind(&Calc::mul, &c, 3, std::placeholders::_1);
std::cout << f(7) << "\n"; // 21
```

### 3.2 `std::bind`의 함정과 대안
- **지연 참조/복사 시맨틱**이 헷갈려 *생명주기 버그*를 만들기 쉽습니다.  
  - 원본을 참조로 넘기려면 `std::ref(x)`, `std::cref(x)` 필요
  - 복사/이동 의도를 명시하기 어려움
- 대안: **람다** 또는 C++20의 `std::bind_front`  
  ```cpp
  auto g = std::bind_front(add, 5); // 앞 인자 고정(간결)
  ```
- 규칙: *새로 작성하는 코드라면 람다/`bind_front` 권장*. `bind`는 유지보수/레거시 호환용.

---

## 4. 이동 의미론(Move Semantics) — 값의 “소유권 이전”

**복사(copy)** 는 값을 *복제*하지만, **이동(move)** 은 내부 자원의 **소유권을 이전**합니다.  
큰 버퍼/파일/소켓/뮤텍스 같은 자원형에 치명적으로 중요하며, 컨테이너 성능에도 직접적입니다.

```cpp
#include <string>
#include <vector>

std::string a = "Hello";
std::string b = std::move(a); // a의 내부 버퍼 소유권 이동 → a는 비어질 수 있음

std::vector<std::string> v;
v.push_back(std::string("Tom"));              // (임시) 복사 또는 이동(소멸 직전)
v.push_back(std::move(std::string("Jerry"))); // 명시적 이동(대개 동일 결과)
```

### 4.1 사용자 정의 타입에서의 이동 생성자/대입자
```cpp
class Buffer {
    char* data;
    std::size_t n;
public:
    Buffer(std::size_t size) : data(new char[size]), n(size) {}
    ~Buffer() { delete[] data; }

    // 이동 생성자
    Buffer(Buffer&& other) noexcept
      : data(other.data), n(other.n) {
        other.data = nullptr; other.n = 0;
    }

    // 이동 대입 연산자
    Buffer& operator=(Buffer&& other) noexcept {
        if (this != &other) {
            delete[] data;
            data = other.data; n = other.n;
            other.data = nullptr; other.n = 0;
        }
        return *this;
    }

    // 복사 금지(선택)
    Buffer(const Buffer&)            = delete;
    Buffer& operator=(const Buffer&) = delete;
};
```

**핵심 포인트**
- 이동 연산은 가급적 `noexcept`로 표기하세요.  
  컨테이너 재배치(예: `std::vector` reallocation)에서 예외 없는 이동 타입을 **선호**합니다.
- 이동 후 원본은 **소멸 가능한 유효하지만 특정 값 없음** 상태여야 합니다(널 포인터 등).

### 4.2 `std::move`는 “캐스팅”일 뿐
- `std::move(x)`는 **우측값 참조로 캐스팅**합니다. 실제 이동은 **수신자 쪽의 이동 생성/대입**이 결정합니다.
- 마구 `std::move`를 남발하면 **이후에 쓸 값이 비게 되어** 버그가 나기도 합니다.  
  규칙: *더 이상 x를 사용하지 않을 때만* `std::move(x)`.

### 4.3 완벽 전달(Perfect Forwarding) & `std::forward`
템플릿에서 매개변수의 값/참조 범주를 보존해 **복사/이동을 자동 선택**시키는 기법입니다.

```cpp
#include <utility>

template <class F, class... Args>
decltype(auto) call(F&& f, Args&&... args) {
    // f와 args...의 값/참조 범주를 보존하여 전달
    return std::forward<F>(f)(std::forward<Args>(args)...);
}
```

- `T&&`가 **전달 참조(Forwarding Reference)** 로 쓰일 때만 동작(템플릿 타입 추론 맥락).
- 라이브러리 래퍼/팩토리/데코레이터 패턴에서 필수.

### 4.4 복사 생략(Copy Elision) & RVO/NRVO
- 최신 컴파일러는 **반환 값 최적화**(RVO)를 광범위하게 적용합니다.
- `return obj;`에서 `obj`가 지역 변수면 NRVO가, `return T{...};`면 RVO가 일어납니다.  
  → 대체로 `std::move`를 *굳이 쓰지 않는 편이* 최적입니다(오히려 RVO를 방해할 수 있음).

---

## 5. 실전 패턴 모음

### 5.1 콜백 저장 — `std::function` vs 템플릿
```cpp
// 쉽고 유연: 런타임 다형성
struct Button {
    std::function<void()> onClick;
};

// 빠르고 제네릭: 컴파일타임 다형성(헤더에 구현 필요)
template <class F>
struct ButtonT {
    F onClick;
};
```
- **빈번히 교체/저장해야 하면** `std::function`이 편리.
- **성능 임계 경로**에서, 특정 호출자만 쓸 거면 템플릿로 고정(할당/간접 호출 감소).

### 5.2 안전한 비동기 콜백 — 캡처 전략
```cpp
// ❌ 지역 변수 참조 캡처: 비동기 완료 시점에 지역이 파괴되어 댕글링
void bad() {
    int x = 42;
    async([&]{ use(x); }); // 위험!
}

// ✅ 값 캡처 또는 shared_ptr
void good() {
    auto sp = std::make_shared<int>(42);
    async([sp]{ use(*sp); });
}
```

### 5.3 `bind`보다 람다
```cpp
// 가독성 떨어짐
auto f1 = std::bind(&Calc::mul, &c, 3, std::placeholders::_1);
// 선호: 람다
auto f2 = [&c](int x){ return c.mul(3, x); };
```

### 5.4 이동만 허용하는 타입과 컨테이너
```cpp
std::vector<std::unique_ptr<int>> v;
v.emplace_back(std::make_unique<int>(7)); // 이동만
```
- 이동 전용 타입(파일, 소켓, mutex 등)은 컨테이너와 궁합이 좋습니다.
- `noexcept` 이동을 제공하면 컨테이너에서 더 효율적입니다.

---

## 6. 요약 표

| 주제 | 핵심 | 주의점/가이드 |
|---|---|---|
| `std::function` | 이기종 호출자 **타입 소거** 래퍼 | SBO 불명확, 큰 캡처는 할당 발생 가능. 성능 민감 시 템플릿 대안 고려 |
| 람다 | 캡처/익명 함수로 **간결한 호출자** 생성 | 참조 캡처 수명 주의, 비동기엔 값 캡처/`shared_ptr` 활용 |
| `std::bind` | 인자 고정/재배열(레거시 친화) | 새 코드엔 람다/`bind_front` 선호, `ref/cref` 명시 |
| 이동 의미론 | **소유권 이전**으로 대용량 객체 최적화 | `noexcept` 이동, `std::move` 남용 금지, RVO에 맡기기 |
| 완벽 전달 | `std::forward`로 값 범주 보존 | 전달 참조 문맥에서만 동작, 래퍼/팩토리에 유용 |

---

## 7. 체크리스트

- [ ] 비동기 콜백에 **참조 캡처**를 쓰지 않았는가? (값/`shared_ptr` 캡처로 전환)  
- [ ] 성능 임계 경로에서 `std::function`이 **불필요한 할당**을 유발하지 않는가?  
- [ ] 새 코드에서 `std::bind` 대신 **람다/`bind_front`**를 사용했는가?  
- [ ] 이동 생성/대입에 **`noexcept`** 를 붙였는가?  
- [ ] 반환 시 **RVO를 방해하는 `std::move`** 를 넣지 않았는가?

---

## 부록 A) 실전 예제: 이벤트 파이프라인

```cpp
#include <functional>
#include <string>
#include <vector>
#include <iostream>

class Pipeline {
public:
    using Stage = std::function<std::string(std::string)>;

    void add(Stage s) { stages_.push_back(std::move(s)); }

    std::string run(std::string input) const {
        for (auto const& s : stages_) input = s(input);
        return input;
    }
private:
    std::vector<Stage> stages_;
};

int main() {
    Pipeline p;
    p.add([](std::string s){ return "[trim]" + s; });
    p.add([](std::string s){ return s + "[lower]"; });

    // 멤버 함수 바인딩은 람다로 권장
    struct S { std::string tag(std::string s) const { return s + "[tag]"; } } obj;
    p.add([&obj](std::string s){ return obj.tag(std::move(s)); });

    std::cout << p.run("  DATA  ") << "\n";
}
```

---

## 부록 B) 이동 전용 타입과 콜렉션

```cpp
#include <vector>
#include <memory>
#include <iostream>

struct Big {
    Big(size_t n) : data(new int[n]), n(n) {}
    ~Big(){ delete[] data; }
    Big(Big&& o) noexcept : data(o.data), n(o.n) { o.data=nullptr; o.n=0; }
    Big& operator=(Big&& o) noexcept {
        if (this != &o) { delete[] data; data=o.data; n=o.n; o.data=nullptr; o.n=0; }
        return *this;
    }
    Big(const Big&) = delete;
    Big& operator=(const Big&) = delete;
    int* data{}; size_t n{};
};

int main() {
    std::vector<Big> v;
    v.emplace_back(1<<20); // no-throw move면 재할당 최적
    v.emplace_back(1<<22);
    std::cout << v.size() << "\n";
}
```

---

## 마무리

- **람다 + `std::function`** 으로 유연한 콜백을, **이동 의미론**으로 대용량/자원형의 성능을 끌어올리세요.  
- 새 코드라면 **람다/`bind_front`**를 우선, 레거시 연결에는 `std::bind`를 제한적으로.  
- 비동기/지연 실행에서는 **캡처 수명**을 최우선으로 점검하세요.