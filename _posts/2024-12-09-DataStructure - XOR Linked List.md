---
layout: post
title: Data Structure - XOR Linked List
date: 2024-12-09 21:20:23 +0900
category: Data Structure
---
# XOR Linked List

## 1. XOR Linked List란?

**XOR 연결 리스트**는 이중 연결 리스트의 `prev`/`next` 두 포인터를 **단 하나의 필드**로 합친다.

- 노드가 가진 포인터:  
  \[
  \text{npx} = \text{prev} \oplus \text{next}
  \]
  여기서 \(\oplus\)는 비트 XOR.

- **공간 절약**: 64비트 환경에서 이중 리스트는 포인터 2개(16B), XOR 리스트는 1개(8B)로, **노드당 8B 절약**.
- **탐색**: 다음 노드를 얻으려면 **이전 노드를 알고 있어야** 한다.
  \[
  \text{next} = \text{npx} \oplus \text{prev}
  \]

> 평균 시간 복잡도는 이중 리스트와 동일하다(단일 스텝 전진/후진은 \(O(1)\), 임의 위치 검색은 \(O(n)\)).

---

## 2. 구조 개념 복습

### 이중 리스트

```
[prev] <- Node -> [next]
```

### XOR 리스트

```
[npx = prev ⊕ next]
```

- 헤드에서의 전진: `prev = nullptr`로 시작하여 `next = XOR(prev, cur->npx)`.
- 테일부터 역방향도 동일(대칭).

---

## 3. 안전성·이식성 경고(매우 중요)

- C/C++ 이외 언어(특히 **이동/압축 GC**)에서는 **주소가 바뀌면 XOR 값이 무효** → 금지.
- **AddressSanitizer(ASan)**, **HWASan** 등 도구가 붙은 환경에서는 포인터 태깅/레드존으로 인해 **예상치 못한 동작**을 유발 가능.
- 메모리 덮어쓰기 버그가 나면 **디버깅 난이도 급상승** (정상 포인터와 달리 체인 복원이 어렵다).
- 멀티스레드에선 **원자적 갱신**/메모리 모델을 세심히 설계해야 한다(본 글은 단일 스레드 가정).

> 학습/문제풀이/임베디드 특수 상황 외에는 **일반 이중 연결 리스트**가 훨씬 안전하고 유지보수성이 높다.

---

## 4. 기초 도우미와 노드 정의

```cpp
// xor_list.hpp
#pragma once
#include <cstdint>
#include <iostream>
#include <stdexcept>
#include <utility>

namespace xorll {

inline void* xor_ptr(void* a, void* b) noexcept {
    return reinterpret_cast<void*>(
        reinterpret_cast<std::uintptr_t>(a) ^
        reinterpret_cast<std::uintptr_t>(b)
    );
}
template <class T> inline T* xor_ptr(T* a, T* b) noexcept {
    return reinterpret_cast<T*>( xor_ptr(static_cast<void*>(a), static_cast<void*>(b)) );
}

template <class T>
struct Node {
    T      data;
    Node*  npx;  // prev ^ next
    template <class... Args>
    explicit Node(Args&&... args)
      : data(std::forward<Args>(args)...), npx(nullptr) {}
};
} // namespace xorll
```

- 포인터 XOR은 반드시 **`uintptr_t`**를 경유해야 정의된 동작.
- 노드는 `data`와 `npx`만 가진다.

---

## 5. 컨테이너 설계(양끝·사이즈·반복자)

- `head_`, `tail_`을 들고 있으면 **양방향 순회/삽입/삭제**가 편해진다.
- 반복자는 **(prev, cur, next)** 3튜플을 유지하면 `++/--`를 안전하게 구현할 수 있다.
- `end()` 반복자는 **`cur=nullptr`**로 표현하고, **`prev=tail_`**를 들고 있게 하면 `--end()`가 자연스럽다.

```cpp
// xor_list.hpp (계속)
namespace xorll {

template <class T>
class XorList {
    using NodeT = Node<T>;

    NodeT* head_ = nullptr;
    NodeT* tail_ = nullptr;
    std::size_t size_ = 0;

public:
    XorList() = default;
    ~XorList() { clear(); }

    XorList(const XorList&) = delete;
    XorList& operator=(const XorList&) = delete;

    bool empty() const noexcept { return size_==0; }
    std::size_t size() const noexcept { return size_; }

    T& front() { if(empty()) throw std::out_of_range("empty"); return head_->data; }
    T& back()  { if(empty()) throw std::out_of_range("empty"); return tail_->data; }
    const T& front() const { if(empty()) throw std::out_of_range("empty"); return head_->data; }
    const T& back()  const { if(empty()) throw std::out_of_range("empty"); return tail_->data; }

    // 기본 연산
    void push_front(const T& v) { emplace_front(v); }
    void push_back (const T& v) { emplace_back (v); }

    template <class... Args> void emplace_front(Args&&... args);
    template <class... Args> void emplace_back (Args&&... args);

    void pop_front();
    void pop_back();
    void clear() noexcept;

    // 반복자
    class iterator {
        NodeT* prev_ = nullptr;
        NodeT* cur_  = nullptr;
        NodeT* next_ = nullptr;
        const XorList* owner_ = nullptr;
    public:
        using difference_type = std::ptrdiff_t;
        using value_type = T;
        using reference = T&;
        using pointer = T*;
        using iterator_category = std::bidirectional_iterator_tag;

        iterator() = default;
        iterator(const XorList* o, NodeT* p, NodeT* c, NodeT* n)
            : prev_(p), cur_(c), next_(n), owner_(o) {}

        reference operator*()  const { return cur_->data; }
        pointer   operator->() const { return &cur_->data; }

        bool operator==(const iterator& o) const { return cur_==o.cur_ && owner_==o.owner_; }
        bool operator!=(const iterator& o) const { return !(*this==o); }

        iterator& operator++(){ // forward
            if (!cur_) throw std::out_of_range("increment end()");
            prev_ = cur_;
            cur_  = next_;
            next_ = (cur_ ? xor_ptr(prev_, cur_->npx) : nullptr);
            return *this;
        }
        iterator operator++(int){ auto t=*this; ++*this; return t; }

        iterator& operator--(){ // backward
            // if at end(), step to tail
            if (!cur_) {
                cur_  = prev_;            // prev_ holds tail at end()
                if (!cur_) throw std::out_of_range("decrement begin()");
                next_ = nullptr;
                // recompute prev_ from tail
                prev_ = xor_ptr(next_, cur_->npx);
                return *this;
            }
            next_ = cur_;
            cur_  = prev_;
            prev_ = (cur_ ? xor_ptr(next_, cur_->npx) : nullptr);
            return *this;
        }
        iterator operator--(int){ auto t=*this; --*this; return t; }

        friend class XorList;
    };

    iterator begin() const {
        if (!head_) return end();
        NodeT* next = xor_ptr<NodeT>(nullptr, head_->npx);
        return iterator(this, nullptr, head_, next);
    }
    iterator end() const { // past-the-end: cur=nullptr, prev=tail
        return iterator(this, tail_, nullptr, nullptr);
    }

    // 삽입/삭제(반복자 기반)
    iterator insert(iterator pos, const T& v) { return emplace(pos, v); }
    template <class... Args> iterator emplace(iterator pos, Args&&... args);
    iterator erase(iterator pos);
};

} // namespace xorll
```

---

## 6. 핵심 연산 구현

### 6.1 양끝 삽입/삭제

```cpp
// xor_list_impl.hpp
#pragma once
#include "xor_list.hpp"

namespace xorll {

template <class T>
template <class... Args>
void XorList<T>::emplace_front(Args&&... args){
    NodeT* n = new NodeT(std::forward<Args>(args)...);
    n->npx = xor_ptr<NodeT>(nullptr, head_);
    if (!head_) {
        head_ = tail_ = n;
    } else {
        // head_->npx originally = prev(nullptr) ^ next(head_next)
        NodeT* head_next = xor_ptr<NodeT>(nullptr, head_->npx);
        head_->npx = xor_ptr<NodeT>(n, head_next);
        head_ = n;
    }
    ++size_;
}

template <class T>
template <class... Args>
void XorList<T>::emplace_back(Args&&... args){
    NodeT* n = new NodeT(std::forward<Args>(args)...);
    n->npx = xor_ptr<NodeT>(tail_, nullptr);
    if (!tail_) {
        head_ = tail_ = n;
    } else {
        NodeT* tail_prev = xor_ptr<NodeT>(tail_->npx, nullptr);
        tail_->npx = xor_ptr<NodeT>(tail_prev, n);
        tail_ = n;
    }
    ++size_;
}

template <class T>
void XorList<T>::pop_front(){
    if (empty()) throw std::out_of_range("pop_front on empty");
    NodeT* old = head_;
    NodeT* next = xor_ptr<NodeT>(nullptr, head_->npx);
    if (!next) {
        head_ = tail_ = nullptr;
    } else {
        NodeT* nextnext = xor_ptr<NodeT>(head_, next->npx);
        // new head is next; its prev becomes nullptr
        next->npx = xor_ptr<NodeT>(nullptr, nextnext);
        head_ = next;
    }
    delete old; --size_;
}

template <class T>
void XorList<T>::pop_back(){
    if (empty()) throw std::out_of_range("pop_back on empty");
    NodeT* old = tail_;
    NodeT* prev = xor_ptr<NodeT>(tail_->npx, nullptr);
    if (!prev) {
        head_ = tail_ = nullptr;
    } else {
        NodeT* prevprev = xor_ptr<NodeT>(prev->npx, tail_);
        prev->npx = xor_ptr<NodeT>(prevprev, nullptr);
        tail_ = prev;
    }
    delete old; --size_;
}

template <class T>
void XorList<T>::clear() noexcept {
    NodeT* prev = nullptr;
    NodeT* cur  = head_;
    while (cur) {
        NodeT* next = xor_ptr<NodeT>(prev, cur->npx);
        prev = cur;
        delete cur;
        cur = next;
    }
    head_ = tail_ = nullptr;
    size_ = 0;
}

} // namespace xorll
```

### 6.2 반복자 기반 `emplace/insert`(pos 앞에 삽입)

- 표준 `std::list::insert(pos, v)`와 동일하게 **`pos` 앞**에 들어간다고 가정한다.
- `pos`가 `begin()`이면 `emplace_front`, `end()`면 `emplace_back`.

```cpp
// xor_list_impl.hpp (계속)
namespace xorll {

template <class T>
template <class... Args>
typename XorList<T>::iterator
XorList<T>::emplace(iterator pos, Args&&... args){
    if (pos.owner_ != this) throw std::runtime_error("iterator mismatch");
    if (pos.cur_ == head_) { // 맨 앞
        emplace_front(std::forward<Args>(args)...);
        return begin();
    }
    if (!pos.cur_) { // end() 앞 = 맨 뒤
        std::size_t oldsz = size_;
        emplace_back(std::forward<Args>(args)...);
        auto it = end(); --it; // 새 tail
        (void)oldsz;
        return it;
    }

    NodeT* next = pos.cur_;
    NodeT* prev = pos.prev_; // pos는 prev/cur/next를 보유
    // 새 노드
    NodeT* n = new NodeT(std::forward<Args>(args)...);
    // 링크: prev - n - next
    // prev 업데이트
    if (prev) {
        NodeT* prevprev = xor_ptr<NodeT>(prev->npx, next);
        prev->npx = xor_ptr<NodeT>(prevprev, n);
    } else {
        // prev==nullptr면 head 앞에 삽입이므로 head 갱신
        n->npx = xor_ptr<NodeT>(nullptr, next);
        NodeT* nextnext = xor_ptr<NodeT>(nullptr, next->npx);
        next->npx = xor_ptr<NodeT>(n, nextnext);
        head_ = n;
        ++size_;
        return iterator(this, nullptr, n, next);
    }
    // next 업데이트
    NodeT* nextnext = xor_ptr<NodeT>(next->npx, prev);
    next->npx = xor_ptr<NodeT>(n, nextnext);
    // n 설정
    n->npx = xor_ptr<NodeT>(prev, next);

    ++size_;
    // 삽입된 노드에 대한 반복자 반환
    NodeT* new_prev = prev ? xor_ptr<NodeT>(next, prev->npx) : nullptr; // 의미 없음, 재계산
    (void)new_prev;
    return iterator(this, prev, n, next);
}

template <class T>
typename XorList<T>::iterator
XorList<T>::erase(iterator pos){
    if (pos.owner_ != this) throw std::runtime_error("iterator mismatch");
    if (!pos.cur_) throw std::out_of_range("erase end()");
    NodeT* cur  = pos.cur_;
    NodeT* prev = pos.prev_;
    NodeT* next = pos.next_;

    // 경계 처리
    if (!prev && !next) {
        // single node
        delete cur; head_=tail_=nullptr; size_=0;
        return end();
    }
    if (!prev) {
        // 제거 노드가 head
        NodeT* nextnext = xor_ptr<NodeT>(cur, next->npx);
        next->npx = xor_ptr<NodeT>(nullptr, nextnext);
        head_ = next;
        delete cur; --size_;
        return iterator(this, nullptr, head_, xor_ptr<NodeT>(nullptr, head_->npx));
    }
    if (!next) {
        // 제거 노드가 tail
        NodeT* prevprev = xor_ptr<NodeT>(prev->npx, cur);
        prev->npx = xor_ptr<NodeT>(prevprev, nullptr);
        tail_ = prev;
        delete cur; --size_;
        return end(); // tail 뒤
    }

    // 일반 케이스: prev - cur - next
    NodeT* prevprev = xor_ptr<NodeT>(prev->npx, cur);
    NodeT* nextnext = xor_ptr<NodeT>(next->npx, cur);
    prev->npx = xor_ptr<NodeT>(prevprev, next);
    next->npx = xor_ptr<NodeT>(prev,     nextnext);

    delete cur; --size_;
    // cur 자리에 next가 오므로, 새 반복자는 (prev, next, nextnext)
    return iterator(this, prev, next, nextnext);
}

} // namespace xorll
```

> 연결 갱신의 핵심 패턴  
> **기존**: `X->npx = A ^ B`  
> **중간 노드 C 제거 후** `X->npx = (A ^ C) ^ C ^ D = A ^ D` 로 바꾸기 →  
> 구현에서는 `xor_ptr(x->npx, C)`로 C를 제거하고, 이어서 새 이웃을 XOR한다.

---

## 7. 사용 예제

### 7.1 기본 동작

```cpp
// main_basic.cpp
#include "xor_list_impl.hpp"
using namespace xorll;

int main(){
    XorList<int> xs;
    xs.push_front(30);  // [30]
    xs.push_front(20);  // [20,30]
    xs.push_front(10);  // [10,20,30]
    xs.push_back(40);   // [10,20,30,40]

    // 순회
    for (auto it=xs.begin(); it!=xs.end(); ++it) {
        std::cout << *it << " ";
    }
    std::cout << "\n"; // 10 20 30 40

    // 뒤에서 앞으로
    auto it = xs.end();
    while (true) {
        try { --it; std::cout << *it << " "; }
        catch (...) { break; }
    }
    std::cout << "\n"; // 40 30 20 10

    // 중간 삽입: 25를 30 앞에
    for (auto it=xs.begin(); it!=xs.end(); ++it){
        if (*it==30){ xs.insert(it, 25); break; }
    }
    // 지우기: 20
    for (auto it=xs.begin(); it!=xs.end(); ++it){
        if (*it==20){ xs.erase(it); break; }
    }

    for (auto v: xs) std::cout << v << " ";
    std::cout << "\n"; // 10 25 30 40
}
```

### 7.2 예외 테스트(경계)

```cpp
// main_edge.cpp
#include "xor_list_impl.hpp"
using namespace xorll;

int main(){
    XorList<int> xs;
    try { xs.pop_front(); } catch(const std::exception& e){ std::cout << e.what() << "\n"; }
    xs.push_back(1);
    xs.pop_back(); // empty
    std::cout << xs.size() << "\n"; // 0

    xs.push_back(5);
    xs.push_back(6);
    auto it = xs.begin(); // points to 5
    xs.erase(it);         // erase head
    std::cout << xs.front() << "\n"; // 6
}
```

---

## 8. 복잡도·메모리·수학 스냅샷

- **시간**
  - 전/후진 1스텝: $$O(1)$$
  - 맨앞/맨뒤 삽입/삭제: $$O(1)$$
  - 중간 위치 반복자 기반 `insert/erase`: 이웃만 갱신 → $$O(1)$$  
    (단, 그 위치까지 가는 비용은 순회이므로 평균 $$O(n)$$)
- **공간**
  - 이중 리스트: 노드당 포인터 2개 → \(2 \cdot \text{ptr\_size}\)
  - XOR 리스트: 포인터 1개 → \(\text{ptr\_size}\)
  - **절감률**(64비트 가정):  
    \[
    \frac{2-1}{2} = 50\%
    \]
  - 실제로는 **할당자 메타데이터/정렬 패딩**으로 인해 체감 절감률은 더 낮을 수 있다.

---

## 9. 디버깅·테스트 전략

1. **일관성 체크**: 순방향으로 수집한 시퀀스와 역방향 시퀀스가 서로 역순인지 비교.
2. **브루트 대조**: 동일 연산 시퀀스를 `std::list`와 동시 실행 후 결과 비교.
3. **퍼징**: 랜덤 `insert/erase/pop`/순회 후 불변식 확인.

```cpp
// test_fuzz.cpp
#include "xor_list_impl.hpp"
#include <list>
#include <random>
#include <cassert>
using namespace xorll;

int main(){
    XorList<int> xs;
    std::list<int> ys;

    std::mt19937 rng(1234);
    std::uniform_int_distribution<int> op(0,4), val(0,1000);

    for (int t=0;t<20000;++t){
        int o = op(rng);
        if (o==0){ // push_front
            int x=val(rng); xs.push_front(x); ys.push_front(x);
        } else if (o==1){ // push_back
            int x=val(rng); xs.push_back(x); ys.push_back(x);
        } else if (o==2 && !ys.empty()){ // pop_front
            xs.pop_front(); ys.pop_front();
        } else if (o==3 && !ys.empty()){ // pop_back
            xs.pop_back(); ys.pop_back();
        } else if (o==4 && !ys.empty()){ // erase random
            int k = val(rng) % (int)ys.size();
            auto itx = xs.begin();
            auto ity = ys.begin();
            for (int i=0;i<k;++i){ ++itx; ++ity; }
            xs.erase(itx); ys.erase(ity);
        }
        // compare
        auto itx = xs.begin(); auto ity = ys.begin();
        for(; itx!=xs.end() && ity!=ys.end(); ++itx, ++ity) assert(*itx==*ity);
        assert(itx==xs.end() && ity==ys.end());
    }
}
```

---

## 10. 왜 실무에서 거의 안 쓰일까?

- **가독성/유지보수성**: 포인터 XOR은 직관적이지 않아 협업 난이도↑.
- **디버깅 지옥**: 중간 상태를 디버거로 보기 어려움. 크래시 시 체인 복원이 힘듦.
- **도구 호환성**: ASan/Valgrind/검증기와 상호작용이 나쁠 수 있음.
- **메모리 절감의 한계**: 실제 시스템에선 노드 내부 다른 필드, 할당자 메타데이터, 캐시 라인 정렬 등으로 **실제 절약률이 작음**.
- **대안 풍부**: 공간이 정말 문제면 **압축 포인터**, **풀 할당자**, **SoA(Structure of Arrays)**, **컨테이너 재설계**가 보통 더 낫다.

---

## 11. 추가 팁/변형

- **정렬 유지 리스트**: 삽입 시 순회 비용은 동일. XOR의 이점은 **포인터 1개**뿐.
- **스레드 안전**: 단일 원자 갱신만으로 충분하지 않다. **락/RCU/해저드 포인터** 설계가 필요.
- **포인터 태깅(tagging)**: 하위 비트를 플래그로 사용하는 트릭과 결합 가능하지만, 정렬 보장/UB 위험이 크다.

---

## 12. 전체 빌드

```bash
g++ -std=c++17 -O2 -Wall -Wextra -pedantic main_basic.cpp -o basic
./basic

g++ -std=c++17 -O2 -Wall -Wextra -pedantic main_edge.cpp  -o edge
./edge

g++ -std=c++17 -O2 -fsanitize=address,undefined -g test_fuzz.cpp -o fuzz
./fuzz
```

> 주의: ASan/UBSan 환경에서도 본 구현은 정의된 동작을 지키지만, **도구가 주입하는 포인터 변형/레드존**이 있는 플랫폼에서는 false positive/성능 저하가 있을 수 있다.

---

## 13. 요약

- XOR 리스트는 **이중 리스트의 공간을 절반으로 줄이는** 고전 테크닉이다.
- 구현은 간단해 보이나, **연결 갱신/반복자/예외/디버깅**에서 난이도가 높다.
- 실전 대안: **일반 이중 리스트 + 맞춤 할당자/메모리 풀**이 대부분 더 낫다.
- 학습 관점에선 **포인터 연산/메모리 모델** 감각을 키우는 훌륭한 연습 주제다.