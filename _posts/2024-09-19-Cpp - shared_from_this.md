---
layout: post
title: C++ - shared_from_this
date: 2024-09-19 19:20:23 +0900
category: Cpp
---
# `shared_from_this`와 `enable_shared_from_this`

## 0) 문제의 본질 — “자기 자신을 소유하는 포인터”의 위험
다음 코드는 **절대 금지**다.

```cpp
struct Bad {
    std::shared_ptr<Bad> getSelf() {
        return std::shared_ptr<Bad>(this); // ❌ 새로운 컨트롤 블록 생성
    }
};
```

- `std::shared_ptr<Bad>(this)`는 **새 컨트롤 블록**을 만든다.  
- 동일 객체를 **두 개의 서로 다른 컨트롤 블록**이 관리 → **이중 삭제(double delete)** 위험.

상태를 수식으로 표현하면:

$$
\text{same object} \Rightarrow \text{two control blocks } C_1, C_2 \Rightarrow
\exists\, t_1,t_2: \text{destroy}(t_1),\text{destroy}(t_2) \Rightarrow \text{double delete}
$$

**해결책**: 객체가 **이미 연결된 컨트롤 블록**을 재사용하여 `shared_ptr`를 만들어야 한다.

---

## 1) 해법 — `enable_shared_from_this<T>`
표준 라이브러리의 믹스인(Mixin) 베이스:

```cpp
#include <memory>

struct My : std::enable_shared_from_this<My> {
    std::shared_ptr<My> getSelf() {
        return shared_from_this(); // ✅ 같은 컨트롤 블록 공유
    }
};
```

### 핵심 아이디어
- `enable_shared_from_this<T>`는 내부에 **약한 참조(weak_ptr)** 를 보관한다.
- 객체가 **어떤 `shared_ptr`로 관리되기 시작**할 때, 라이브러리가 그 컨트롤 블록과 내부 약참조를 연결한다.
- 이후 `shared_from_this()`는 그 약참조를 **안전하게 승격(lock)** 하여 **같은 컨트롤 블록**을 참조하는 `shared_ptr`를 반환한다.

그림으로 보면:

```
[ shared_ptr<My> p ] --> [ Control Block ] --> [ My ]
                             ↑                 (this)
                      weak_ptr in enable_shared_from_this
```

---

## 2) 전제 조건 — “이미 shared_ptr로 관리 중”이어야 한다
다음은 **정상**:

```cpp
auto obj = std::make_shared<My>();
auto self = obj->getSelf();  // use_count == 2
```

다음은 **UB/예외**:

```cpp
My* raw = new My;
raw->getSelf();  // ❌ std::bad_weak_ptr 예외 (컨트롤 블록 연결 전)
```

### 왜? (사실 모델)
`shared_from_this()`는 내부 weak_ptr을 **승격**한다:

$$
\text{shared\_from\_this}() \equiv \text{weak\_ptr.lock()} \\
\text{lock}() \Rightarrow 
\begin{cases}
\text{shared\_ptr} & \text{if use\_count}>0 \\
\emptyset & \text{if use\_count}=0
\end{cases}
$$

컨트롤 블록 미연결 시 **use_count** 개념 자체가 없다 → 예외.

---

## 3) 생성자에서 `shared_from_this()` 호출 금지
**생성자/소멸자 내부**에서 `shared_from_this()`를 부르면 **컨트롤 블록이 아직 연결되지 않았을 수 있음**.

```cpp
struct C : std::enable_shared_from_this<C> {
    C() {
        // ❌ 위험: make_shared<C>() 호출 도중에는 아직 연결 전일 수 있음
        // auto self = shared_from_this();
    }
};
```

**규칙**:  
- 생성자에서는 **절대 호출하지 말고**,  
- 생성 직후 외부에서 초기화 루틴을 호출하거나(팩토리 패턴),  
- `post_construct()` 같은 별도 멤버에서 사용.

안전 패턴 예:

```cpp
struct C : std::enable_shared_from_this<C> {
    void start() {
        auto self = shared_from_this(); // ✅ 이미 shared_ptr로 소유 중인 시점
        // ...
    }
};

auto c = std::make_shared<C>();
c->start();
```

---

## 4) 콜백/비동기에서의 올바른 캡처
비동기 콜백은 “객체 수명이 이벤트 시점까지 유지돼야” 안전하다.

### 4.1 소유를 연장하려면 `shared_from_this()`
```cpp
struct Conn : std::enable_shared_from_this<Conn> {
    void start() {
        auto self = shared_from_this();   // 소유 연장
        async_read([self](std::string msg){ self->on_msg(msg); });
    }
    void on_msg(const std::string& m) { /* ... */ }
};
```

- 람다가 `self`를 **값으로 캡처** → 콜백 생존 동안 소유 유지.

### 4.2 순환을 피하려면 `weak_ptr` 캡처 후 `lock()`
콜백 리스트가 객체를 오래 붙잡는 구조라면 **순환 참조**가 생길 수 있다.

```cpp
struct Conn : std::enable_shared_from_this<Conn> {
    void start() {
        std::weak_ptr<Conn> weak = shared_from_this();
        async_read([weak](std::string msg){
            if (auto self = weak.lock()) { self->on_msg(msg); } // 객체가 살아있을 때만
        });
    }
};
```

- **수명을 연장하지 않아야** 할 때는 `weak_ptr` 캡처가 정답.

---

## 5) C++17: `weak_from_this()`로 더욱 안전하게
C++17부터 `enable_shared_from_this`는 **`weak_from_this()`** 제공:

```cpp
struct S : std::enable_shared_from_this<S> {
    std::weak_ptr<S> observe() noexcept {
        return weak_from_this(); // 예외 없이 약참조 반환
    }
};
```

- 컨트롤 블록이 없으면 **빈 weak_ptr** 반환(예외 없음).  
- 콜백 등록 시점에 “살아있으면 연결, 아니면 무시” 패턴에 유용.

---

## 6) 다중 상속/가상 상속 시 주의점
### 6.1 동일 베이스 `enable_shared_from_this`를 여러 경로로 상속 ❌
다중 상속 구조에서 **여러 번** `enable_shared_from_this`를 포함하면 **각 경로별로 다른 내부 weak_ptr**을 갖게 될 수 있다.

**규칙**:
- **최상위 공통 베이스**에서 **단 한 번** 상속하거나,  
- 가상 상속으로 **하나의 베이스 인스턴스만** 두기.

```cpp
struct Base : std::enable_shared_from_this<Base> { /*...*/ };

struct Left  : virtual Base { /*...*/ };
struct Right : virtual Base { /*...*/ };

struct Derived : Left, Right { /*...*/ }; // Base는 가상 상속으로 1개만 존재
```

### 6.2 파생 타입에서 `shared_from_this<Derived>()`를 쓰고 싶다?
C++20부터 `shared_from_this()`가 **파생 타입 안전 반환**을 지원한다(표준 구현에 따라 다를 수 있다).  
범용 안전 패턴:

```cpp
struct Base : std::enable_shared_from_this<Base> {
    template<class D>
    std::shared_ptr<D> shared_from_this_as() {
        return std::dynamic_pointer_cast<D>(shared_from_this());
    }
};
```

---

## 7) 별칭(aliased) `shared_ptr`와 결합 — “부분 뷰”를 안전하게
**같은 컨트롤 블록**을 공유하되, **다른 포인터**를 가리키는 `shared_ptr`를 만들 수 있다.

```cpp
struct Big { int header; int payload[1024]; };

auto big = std::make_shared<Big>();
int* slice = big->payload + 100;

// aliasing: 컨트롤 블록은 big과 같지만, 대상 포인터는 slice를 가리킴
std::shared_ptr<int> view(big, slice);

// view가 살아있는 한 big도 파괴되지 않음.
```

- `enable_shared_from_this`와 합치면, **자기 자신의 부분 메모리**를 안전하게 외부 API로 내보낼 수 있다.

---

## 8) 실전 패턴 모음

### 8.1 팩토리 + post-construct 초기화
```cpp
struct Service : std::enable_shared_from_this<Service> {
    void start(); // 내부에서 shared_from_this 사용
    static std::shared_ptr<Service> make() {
        auto s = std::shared_ptr<Service>(new Service); // or make_shared<Service>()
        s->start(); // 생성자 바깥에서 안전 호출
        return s;
    }
};
```

### 8.2 옵저버(리스너) 관리
```cpp
struct Observer { virtual void on_event() = 0; virtual ~Observer() = default; };

struct Subject {
    std::vector<std::weak_ptr<Observer>> list;
    void add(const std::shared_ptr<Observer>& o) { list.push_back(o); }
    void notify() {
        for (auto it = list.begin(); it != list.end();) {
            if (auto s = it->lock()) { s->on_event(); ++it; }
            else it = list.erase(it); // 죽은 관찰자 정리
        }
    }
};
```

### 8.3 비동기 작업 큐
```cpp
struct Task : std::enable_shared_from_this<Task> {
    void schedule(Executor& ex) {
        std::weak_ptr<Task> weak = shared_from_this();
        ex.post([weak]{
            if (auto self = weak.lock()) self->run();
        });
    }
    void run();
};
```

---

## 9) 테스트/디버깅 팁

- **`use_count()`는 진단용**: 멀티스레드 상황에서 즉시 정확 보장을 기대하지 말 것.
- **ASan/UBSan/Valgrind**: 이중 삭제/댕글링/누수 잡는 데 탁월.
- **로그로 수명 추적**: 생성/복사/이동/소멸자에 로그 추가해 **흐름 시각화**.
- **가짜 이벤트 루프**: 콜백 연쇄에서 `weak.lock()`이 반복적으로 실패/성공하는 흐름을 의도적으로 시뮬레이션.

---

## 10) 흔한 함정 체크리스트

1. [ ] **생성자/소멸자**에서 `shared_from_this()` 호출 ❌  
2. [ ] `new T; std::shared_ptr<T>(this)`로 **직접 감싸기** ❌  
3. [ ] **여러 경로**로 `enable_shared_from_this` 상속(다중 상속 충돌) ❌  
4. [ ] 콜백에서 **강한 캡처**로 순환 참조 형성 → `weak_ptr` 고려  
5. [ ] `weak_from_this()` 반환값 **즉시 lock 확인** 없이 사용 ❌  
6. [ ] 컨트롤 블록이 서로 다른 `shared_ptr` 두 개로 **중복 소유** 중복 해제 위험 ❌

---

## 11) 미세 팁

- **`weak_from_this()` (C++17)**: 예외 없는 관찰자 취득.  
- **별칭 생성자**: 동일 수명 보장 + 다른 포인터 노출.  
- **`owner_less<shared_ptr<T>>`**: weak/shared를 키로 하는 정렬 컨테이너 비교자.  
- **혼합 계층**: 베이스에만 `enable_shared_from_this`를 두고 파생에서는 **사용만** 하라.  
- **도메인 규칙**: “밖으로 내보내는 콜백은 기본 `weak` 캡처, 내부 보장된 짧은 범위는 `shared` 캡처”.

---

## 12) 종합 예제 — 네트워크 연결, 파서, 버퍼 뷰까지
요구사항:
- 연결 수명을 콜백 체인 동안 유지
- 콜백 리스트는 순환을 만들지 않도록 약캡처
- 수신 버퍼 일부를 안전하게 노출(별칭 `shared_ptr`)

```cpp
#include <memory>
#include <vector>
#include <functional>
#include <string>
#include <iostream>

using Handler = std::function<void(const std::string&)>;

struct Reactor {
    void post(Handler h) { h("DATA:HELLO, WORLD!"); } // 모의 비동기
};

struct Buffer {
    std::vector<char> bytes;
    explicit Buffer(size_t n) : bytes(n, 0) {}
};

struct Connection : std::enable_shared_from_this<Connection> {
    Reactor& r;
    std::vector<std::weak_ptr<Connection>> peers;
    std::shared_ptr<Buffer> recv;

    explicit Connection(Reactor& re) : r(re), recv(std::make_shared<Buffer>(4096)) {}

    void start() {
        auto self = shared_from_this(); // 수명 연장
        r.post([self](const std::string& msg) {
            self->on_read(msg);
        });
    }

    void on_read(const std::string& msg) {
        // 버퍼 일부를 별칭으로 노출 (동일 수명 보장)
        char* head = recv->bytes.data();
        std::shared_ptr<char> view(recv, head); // alias: recv의 컨트롤 블록 공유
        (void)view; // 데모

        notify_peers(msg);
    }

    void add_peer(const std::shared_ptr<Connection>& c) {
        peers.emplace_back(c); // 약참조 보관으로 순환 방지
    }

    void notify_peers(const std::string& msg) {
        for (auto it = peers.begin(); it != peers.end();) {
            if (auto p = it->lock()) {
                p->on_peer(msg);
                ++it;
            } else {
                it = peers.erase(it);
            }
        }
    }

    void on_peer(const std::string& m) {
        std::cout << "peer notified: " << m << "\n";
    }
};

int main() {
    Reactor reactor;
    auto a = std::make_shared<Connection>(reactor);
    auto b = std::make_shared<Connection>(reactor);

    a->add_peer(b); // 약참조: 순환 방지
    b->add_peer(a);

    a->start(); // 안전한 자기참조로 콜백 체인 유지
    b->start();
}
```

---

## 13) 결정 트리 (언제 무엇을 쓰나)

| 질문 | 예 | 아니오 |
|---|---|---|
| 객체가 이미 `shared_ptr`로 관리되고 있나? | `shared_from_this()` / `weak_from_this()` 사용 | 먼저 `make_shared<T>()`로 생성 |
| 콜백이 “수명 연장”을 보장해야 하나? | `auto self = shared_from_this();` 캡처 | `weak_ptr` 캡처 후 `lock()` |
| 상속 트리가 복잡/다중인가? | 최상위(또는 가상 베이스) **한 곳**에서만 `enable_shared_from_this` | 단일 상속이면 그대로 사용 |
| 부분 뷰를 노출해야 하나? | 별칭 `shared_ptr` (수명=원본) | 일반 포인터/스팬 반환 시 수명 주의 |

---

## 14) 요약
- `shared_ptr(this)`는 **중복 컨트롤 블록**을 만들어 **이중 삭제** 위험.  
- `enable_shared_from_this<T>` + `shared_from_this()`는 **같은 컨트롤 블록**을 재사용하여 **안전한 자기 참조**를 제공.  
- **생성자/소멸자 호출 금지**, **다중 상속 시 베이스를 하나로**.  
- 비동기/콜백에서는 **목적에 따라** `shared` 캡처(수명 연장) vs `weak` 캡처(순환 방지)를 선택.  
- C++17의 `weak_from_this()`로 **예외 없이** 관찰자 취득 가능.  
- 별칭 `shared_ptr`로 **부분 뷰**를 안전하게 외부에 제공.

이 원칙만 지키면, “자기 자신을 `shared_ptr`로 다뤄야 하는” 대부분의 상황에서 **누수/이중 삭제/순환 참조**를 깔끔하게 회피할 수 있다.
