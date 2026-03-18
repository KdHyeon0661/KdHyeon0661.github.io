---
layout: post
title: Data Structure - XOR Linked List
date: 2024-12-09 21:20:23 +0900
category: Data Structure
---
# XOR Linked List (XOR 연결 리스트)

## XOR 연결 리스트란?

XOR 연결 리스트는 **이중 연결 리스트**의 메모리 사용량을 줄이기 위한 기법입니다.  
이중 연결 리스트는 각 노드가 `prev`(이전 노드)와 `next`(다음 노드) 두 개의 포인터를 가집니다.  
XOR 연결 리스트는 이 두 포인터를 **하나의 포인터 필드**로 합칩니다.

- 각 노드는 `npx`라는 하나의 포인터만 저장합니다.
- `npx`는 **이전 노드 주소**와 **다음 노드 주소**를 XOR(배타적 논리합)한 값입니다.
  \[
  \text{npx} = \text{prev} \oplus \text{next}
  \]
  (여기서 \(\oplus\)는 비트 단위 XOR)

- 다음 노드 주소는 이전 노드 주소를 알고 있을 때 계산할 수 있습니다.
  \[
  \text{next} = \text{npx} \oplus \text{prev}
  \]
- 마찬가지로 이전 노드 주소는 다음 노드 주소를 알고 있을 때 계산할 수 있습니다.
  \[
  \text{prev} = \text{npx} \oplus \text{next}
  \]

### 장점과 단점

- **장점**:  
  - 포인터 하나만 저장하므로 메모리 사용량이 이중 연결 리스트의 절반입니다.  
    (64비트 시스템에서 포인터 8바이트 → 노드당 8바이트 절약)
  - 기본 연산(앞/뒤 삽입, 삭제)은 여전히 \(O(1)\)입니다.

- **단점**:  
  - 코드가 복잡하고 이해하기 어렵습니다.  
  - 디버깅이 매우 까다롭습니다.  
  - 포인터 값을 직접 조작하므로 실수하기 쉽고, 실수로 인한 오류를 찾기 어렵습니다.  
  - 가비지 컬렉션이 있는 언어나 주소가 변경되는 환경(예: 압축 포인터)에서는 사용할 수 없습니다.  
  - 멀티스레드 환경에서 안전하게 사용하기 매우 어렵습니다.  
  - 실제로 메모리 절약 효과는 노드의 다른 필드나 메모리 할당 오버헤드 때문에 체감하기 어려울 수 있습니다.

> **결론**: XOR 연결 리스트는 흥미로운 아이디어이지만, 실무에서는 거의 사용되지 않습니다. 학습 목적으로 포인터 연산을 이해하는 데 도움이 될 수 있습니다.

---

## 구조 이해하기

### 이중 연결 리스트

일반적인 이중 연결 리스트의 노드 구조:

```
[prev] <-> Node <-> [next]
```

- `prev`: 이전 노드 주소
- `next`: 다음 노드 주소

### XOR 연결 리스트

XOR 연결 리스트의 노드 구조:

```
[npx = prev XOR next]
```

- `npx`만 저장합니다.

#### 예시

노드 A, B, C가 순서대로 연결되어 있다고 가정합시다.

- A의 `npx` = `NULL XOR B` = B (이전 없음, 다음은 B)
- B의 `npx` = `A XOR C`
- C의 `npx` = `B XOR NULL` = B

이제 A에서 B로 이동하려면:  
A의 이전 노드는 NULL이므로, B = A->npx XOR NULL = A->npx.

B에서 C로 이동하려면:  
B의 이전 노드는 A이므로, C = B->npx XOR A = (A XOR C) XOR A = C.

C에서 B로 역방향 이동하려면:  
C의 다음 노드는 NULL이므로, B = C->npx XOR NULL = C->npx.

이처럼 **현재 노드**와 **이전(또는 다음) 노드**를 알고 있으면 다음 노드를 계산할 수 있습니다.

---

## 주의사항 (반드시 읽어보세요)

- XOR 연결 리스트는 **C/C++** 같은 저수준 언어에서만 가능합니다. (포인터를 정수로 변환하여 XOR 연산해야 하므로)
- 포인터를 정수로 변환할 때는 `uintptr_t` 타입을 사용해야 합니다. (표준에서 보장)
- 메모리 오류 탐지 도구(예: AddressSanitizer)는 포인터를 변조하는 것을 감지하지 못할 수 있으며, 오히려 오탐을 일으킬 수 있습니다.
- 이 코드를 디버깅할 때는 포인터 값 자체를 출력해보기 어렵기 때문에, **연결 상태를 검증하는 함수**를 함께 작성하는 것이 좋습니다.
- **멀티스레드 환경에서는 절대 사용하지 마세요.** 원자적 연산으로도 안전성을 보장하기 어렵습니다.

---

## C++ 구현 (기초 도우미와 노드)

먼저 포인터 XOR 연산을 도와줄 유틸리티 함수를 만듭니다.

```cpp
#include <cstdint>  // uintptr_t
#include <iostream>
#include <stdexcept>

// 두 포인터를 XOR한 결과를 void*로 반환
inline void* xor_ptr(void* a, void* b) noexcept {
    return reinterpret_cast<void*>(
        reinterpret_cast<std::uintptr_t>(a) ^
        reinterpret_cast<std::uintptr_t>(b)
    );
}

// 타입이 있는 포인터 버전 (편의용)
template <typename T>
inline T* xor_ptr(T* a, T* b) noexcept {
    return static_cast<T*>(xor_ptr(static_cast<void*>(a), static_cast<void*>(b)));
}
```

노드 구조체는 데이터와 `npx` 하나만 가집니다.

```cpp
template <typename T>
struct Node {
    T data;
    Node* npx;  // prev XOR next

    template <typename... Args>
    explicit Node(Args&&... args)
        : data(std::forward<Args>(args)...), npx(nullptr) {}
};
```

---

## XOR 연결 리스트 클래스 설계

리스트 클래스는 `head`(첫 노드)와 `tail`(마지막 노드), 그리고 크기를 관리합니다.  
또한 양방향 순회를 지원하기 위해 **반복자(iterator)**를 제공합니다.

```cpp
template <typename T>
class XorList {
    using Node = Node<T>;

    Node* head_ = nullptr;
    Node* tail_ = nullptr;
    size_t size_ = 0;

public:
    XorList() = default;
    ~XorList() { clear(); }

    // 복사는 금지 (간단히 하기 위해)
    XorList(const XorList&) = delete;
    XorList& operator=(const XorList&) = delete;

    bool empty() const { return size_ == 0; }
    size_t size() const { return size_; }

    T& front() {
        if (empty()) throw std::out_of_range("empty");
        return head_->data;
    }
    T& back() {
        if (empty()) throw std::out_of_range("empty");
        return tail_->data;
    }

    // 기본 연산
    void push_front(const T& value) { emplace_front(value); }
    void push_back(const T& value)  { emplace_back(value); }

    template <typename... Args>
    void emplace_front(Args&&... args);

    template <typename... Args>
    void emplace_back(Args&&... args);

    void pop_front();
    void pop_back();

    void clear();

    // 반복자 (간단한 양방향 반복자)
    class iterator {
        Node* prev_ = nullptr;
        Node* cur_  = nullptr;
        Node* next_ = nullptr;

    public:
        using iterator_category = std::bidirectional_iterator_tag;
        using value_type = T;
        using difference_type = std::ptrdiff_t;
        using pointer = T*;
        using reference = T&;

        iterator() = default;
        iterator(Node* p, Node* c, Node* n) : prev_(p), cur_(c), next_(n) {}

        reference operator*() const { return cur_->data; }
        pointer operator->() const { return &cur_->data; }

        bool operator==(const iterator& other) const { return cur_ == other.cur_; }
        bool operator!=(const iterator& other) const { return cur_ != other.cur_; }

        // 전진 (++it)
        iterator& operator++() {
            if (!cur_) throw std::out_of_range("increment end()");
            prev_ = cur_;
            cur_  = next_;
            next_ = (cur_ ? xor_ptr(prev_, cur_->npx) : nullptr);
            return *this;
        }
        iterator operator++(int) {
            auto tmp = *this;
            ++*this;
            return tmp;
        }

        // 후진 (--it)
        iterator& operator--() {
            if (!cur_) { // end()인 경우 tail로 이동
                cur_  = prev_;   // prev_에 tail을 저장해둠 (end 생성 시)
                if (!cur_) throw std::out_of_range("decrement begin()");
                next_ = nullptr;
                prev_ = xor_ptr(next_, cur_->npx);
                return *this;
            }
            next_ = cur_;
            cur_  = prev_;
            prev_ = (cur_ ? xor_ptr(next_, cur_->npx) : nullptr);
            return *this;
        }
        iterator operator--(int) {
            auto tmp = *this;
            --*this;
            return tmp;
        }

        friend class XorList;
    };

    iterator begin() {
        if (!head_) return end();
        Node* next = xor_ptr<Node>(nullptr, head_->npx);
        return iterator(nullptr, head_, next);
    }
    iterator end() {
        // end는 cur_=nullptr, prev_=tail_ 로 설정하여 --end()가 tail을 가리키게 함
        return iterator(tail_, nullptr, nullptr);
    }

    // 삽입/삭제 (반복자 위치)
    iterator insert(iterator pos, const T& value) {
        return emplace(pos, value);
    }
    template <typename... Args>
    iterator emplace(iterator pos, Args&&... args);

    iterator erase(iterator pos);
};
```

---

## 핵심 연산 구현

### 앞/뒤 삽입 (`emplace_front`, `emplace_back`)

```cpp
template <typename T>
template <typename... Args>
void XorList<T>::emplace_front(Args&&... args) {
    Node* newNode = new Node(std::forward<Args>(args)...);
    newNode->npx = xor_ptr<Node>(nullptr, head_);

    if (!head_) {
        // 빈 리스트
        head_ = tail_ = newNode;
    } else {
        // 기존 head의 npx는 (nullptr ^ next)였음. 새 노드가 앞에 오므로
        // 기존 head의 npx = newNode ^ next
        Node* headNext = xor_ptr<Node>(nullptr, head_->npx);
        head_->npx = xor_ptr<Node>(newNode, headNext);
        head_ = newNode;
    }
    ++size_;
}

template <typename T>
template <typename... Args>
void XorList<T>::emplace_back(Args&&... args) {
    Node* newNode = new Node(std::forward<Args>(args)...);
    newNode->npx = xor_ptr<Node>(tail_, nullptr);

    if (!tail_) {
        head_ = tail_ = newNode;
    } else {
        Node* tailPrev = xor_ptr<Node>(tail_->npx, nullptr);
        tail_->npx = xor_ptr<Node>(tailPrev, newNode);
        tail_ = newNode;
    }
    ++size_;
}
```

### 앞/뒤 삭제 (`pop_front`, `pop_back`)

```cpp
template <typename T>
void XorList<T>::pop_front() {
    if (empty()) throw std::out_of_range("pop_front on empty");

    Node* oldHead = head_;
    Node* next = xor_ptr<Node>(nullptr, head_->npx);

    if (!next) {
        // 노드가 하나뿐
        head_ = tail_ = nullptr;
    } else {
        Node* nextNext = xor_ptr<Node>(oldHead, next->npx);
        next->npx = xor_ptr<Node>(nullptr, nextNext);
        head_ = next;
    }
    delete oldHead;
    --size_;
}

template <typename T>
void XorList<T>::pop_back() {
    if (empty()) throw std::out_of_range("pop_back on empty");

    Node* oldTail = tail_;
    Node* prev = xor_ptr<Node>(tail_->npx, nullptr);

    if (!prev) {
        head_ = tail_ = nullptr;
    } else {
        Node* prevPrev = xor_ptr<Node>(prev->npx, oldTail);
        prev->npx = xor_ptr<Node>(prevPrev, nullptr);
        tail_ = prev;
    }
    delete oldTail;
    --size_;
}
```

### 리스트 비우기 (`clear`)

```cpp
template <typename T>
void XorList<T>::clear() {
    Node* prev = nullptr;
    Node* cur  = head_;
    while (cur) {
        Node* next = xor_ptr(prev, cur->npx);
        delete cur;
        prev = cur;
        cur  = next;
    }
    head_ = tail_ = nullptr;
    size_ = 0;
}
```

### 반복자 위치에 삽입 (`emplace`)

`std::list::insert`와 동일하게 **`pos` 앞에** 삽입합니다.

```cpp
template <typename T>
template <typename... Args>
typename XorList<T>::iterator
XorList<T>::emplace(iterator pos, Args&&... args) {
    // pos가 begin()이면 앞에 삽입
    if (pos.cur_ == head_) {
        emplace_front(std::forward<Args>(args)...);
        return begin();
    }
    // pos가 end()이면 뒤에 삽입 (end 앞 = 마지막 뒤)
    if (!pos.cur_) {
        emplace_back(std::forward<Args>(args)...);
        auto it = end();
        --it;  // 방금 추가된 노드로 이동
        return it;
    }

    Node* next = pos.cur_;
    Node* prev = pos.prev_;  // pos가 알고 있는 이전 노드

    // 새 노드 생성
    Node* newNode = new Node(std::forward<Args>(args)...);
    newNode->npx = xor_ptr(prev, next);

    // prev 갱신
    if (prev) {
        Node* prevPrev = xor_ptr(prev->npx, next);
        prev->npx = xor_ptr(prevPrev, newNode);
    } else {
        // prev가 nullptr인 경우는 없음 (위에서 begin 케이스 처리)
    }

    // next 갱신
    Node* nextNext = xor_ptr(next->npx, prev);
    next->npx = xor_ptr(newNode, nextNext);

    ++size_;
    return iterator(prev, newNode, next);
}
```

### 반복자 위치 삭제 (`erase`)

```cpp
template <typename T>
typename XorList<T>::iterator
XorList<T>::erase(iterator pos) {
    if (!pos.cur_) throw std::out_of_range("erase end()");

    Node* cur  = pos.cur_;
    Node* prev = pos.prev_;
    Node* next = pos.next_;

    // 단일 노드
    if (!prev && !next) {
        delete cur;
        head_ = tail_ = nullptr;
        size_ = 0;
        return end();
    }

    // head 삭제
    if (!prev) {
        Node* nextNext = xor_ptr(cur, next->npx);
        next->npx = xor_ptr(nullptr, nextNext);
        head_ = next;
        delete cur;
        --size_;
        return iterator(nullptr, head_, xor_ptr(nullptr, head_->npx));
    }

    // tail 삭제
    if (!next) {
        Node* prevPrev = xor_ptr(prev->npx, cur);
        prev->npx = xor_ptr(prevPrev, nullptr);
        tail_ = prev;
        delete cur;
        --size_;
        return end();
    }

    // 중간 삭제
    Node* prevPrev = xor_ptr(prev->npx, cur);
    Node* nextNext = xor_ptr(next->npx, cur);
    prev->npx = xor_ptr(prevPrev, next);
    next->npx = xor_ptr(prev,     nextNext);

    delete cur;
    --size_;
    return iterator(prev, next, nextNext);
}
```

---

## 사용 예제

```cpp
#include <iostream>
#include "xor_list.hpp"  // 위 코드를 모은 헤더

int main() {
    XorList<int> list;

    list.push_front(30);   // [30]
    list.push_front(20);   // [20,30]
    list.push_front(10);   // [10,20,30]
    list.push_back(40);    // [10,20,30,40]
    list.push_back(50);    // [10,20,30,40,50]

    // 순방향 출력
    std::cout << "순방향: ";
    for (auto it = list.begin(); it != list.end(); ++it) {
        std::cout << *it << " ";
    }
    std::cout << std::endl;

    // 역방향 출력 (--end()로 tail로 이동)
    std::cout << "역방향: ";
    auto it = list.end();
    while (true) {
        try {
            --it;
            std::cout << *it << " ";
        } catch (...) {
            break;
        }
    }
    std::cout << std::endl;

    // 중간 삽입 (25를 30 앞에)
    for (auto it = list.begin(); it != list.end(); ++it) {
        if (*it == 30) {
            list.insert(it, 25);
            break;
        }
    }

    // 20 삭제
    for (auto it = list.begin(); it != list.end(); ++it) {
        if (*it == 20) {
            list.erase(it);
            break;
        }
    }

    std::cout << "수정 후: ";
    for (int x : list) {  // 범위 기반 for 문 (begin/end 필요)
        std::cout << x << " ";
    }
    std::cout << std::endl;

    return 0;
}
```

**예상 출력:**
```
순방향: 10 20 30 40 50 
역방향: 50 40 30 20 10 
수정 후: 10 25 30 40 50
```

---

## 복잡도 및 메모리

- **시간 복잡도**:
  - 앞/뒤 삽입/삭제: \(O(1)\)
  - 반복자 위치 삽입/삭제: \(O(1)\) (이미 위치를 알고 있을 때)
  - 탐색: \(O(n)\) (임의 접근 불가)

- **공간 복잡도**:
  - 이중 연결 리스트: 노드당 포인터 2개
  - XOR 리스트: 노드당 포인터 1개 → **50% 절약**
  - 하지만 노드의 데이터 필드 크기가 작거나, 메모리 할당 오버헤드가 큰 경우 실제 절감 효과는 미미할 수 있습니다.

---

## 왜 실무에서 거의 사용되지 않을까?

1. **가독성과 유지보수성**: 코드를 이해하기 어렵고, 새로운 개발자가 바로 투입되기 힘듭니다.
2. **디버깅의 어려움**: 연결 상태를 파악하기 위해 항상 XOR을 풀어야 하므로 디버거에서 값 확인이 불편합니다.
3. **도구 호환성**: AddressSanitizer 같은 메모리 검사 도구가 포인터 변조를 오탐할 수 있습니다.
4. **실질적 절약 효과 미미**: 현대 시스템에서 메모리는 비교적 저렴하고, 노드당 8바이트 절약보다 코드 복잡성이 더 큰 비용입니다.
5. **안전한 대안**: 대부분의 상황에서는 `std::list`(이중 연결 리스트)를 그대로 사용하는 것이 좋습니다.

---

## 결론

XOR 연결 리스트는 **포인터 연산의 묘미**를 보여주는 흥미로운 주제입니다.  
메모리가 매우 제한된 임베디드 시스템이나 특수한 환경이 아니라면 실제로 사용할 일은 거의 없지만,  
C++의 포인터와 비트 연산을 깊이 이해하는 데 도움이 될 수 있습니다.

**학습 목표**:
- 포인터를 정수로 변환하여 연산하는 방법
- XOR의 성질을 이용한 데이터 구조 설계
- 반복자와 예외 처리의 실제 구현 경험

만약 실제 프로젝트에서 이중 연결 리스트가 필요하다면, 그냥 `std::list`를 사용하세요.