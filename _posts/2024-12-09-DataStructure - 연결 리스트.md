---
layout: post
title: Data Structure - 연결 리스트
date: 2024-12-09 19:20:23 +0900
category: Data Structure
---
# 연결 리스트 (Linked List)

연결 리스트는 **노드(node)**들이 **포인터**로 연결된 자료구조입니다.  
각 노드는 데이터를 저장하고, 다음(혹은 이전) 노드를 가리키는 포인터를 가지고 있습니다.

배열과 달리 **연속된 메모리**를 사용하지 않기 때문에, 중간에 데이터를 삽입하거나 삭제할 때 **데이터를 이동시킬 필요 없이 포인터만 변경**하면 됩니다.  
하지만 **임의 접근(random access)**이 불가능하고, 특정 위치에 접근하려면 처음부터 순서대로 따라가야 합니다.

---

## 왜 배열 대신 연결 리스트를 쓸까?

| 특징 | 배열 | 연결 리스트 |
|------|------|-------------|
| **메모리 구조** | 연속된 공간 | 불연속, 노드 단위로 흩어짐 |
| **임의 접근** | $$O(1)$$ (인덱스로 즉시 접근) | $$O(n)$$ (순차 탐색 필요) |
| **중간 삽입/삭제** | $$O(n)$$ (뒤로 밀거나 당겨야 함) | **위치를 알고 있으면** $$O(1)$$ (포인터만 변경) |
| **캐시 효율** | 좋음 (연속 메모리) | 나쁨 (노드가 여기저기 흩어짐) |

**결론**:  
- 데이터를 **자주 중간에 넣거나 빼야 하고**,  
- 데이터 크기가 커서 **복사 비용이 클 때**  
연결 리스트가 유리합니다.  
그 외에는 대부분 배열(또는 `std::vector`)이 더 빠릅니다.

---

## 단일 연결 리스트 (Singly Linked List)

가장 단순한 형태로, 각 노드는 **데이터**와 **다음 노드 포인터(`next`)**만 가집니다.

```
[data | next] -> [data | next] -> [data | next] -> null
```

- **헤드(head)** : 첫 번째 노드를 가리키는 포인터.
- **끝 노드**의 `next`는 `nullptr`.

### 기본 구현 (C++)

```cpp
#include <iostream>

struct Node {
    int data;
    Node* next;

    explicit Node(int val, Node* nxt = nullptr)
        : data(val), next(nxt) {}
};

// 앞에 삽입
void push_front(Node*& head, int value) {
    head = new Node(value, head);
}

// 앞에서 삭제
void pop_front(Node*& head) {
    if (!head) return;                 // 빈 리스트
    Node* temp = head;
    head = head->next;
    delete temp;
}

// 출력
void print(Node* head) {
    for (Node* cur = head; cur; cur = cur->next)
        std::cout << cur->data << " -> ";
    std::cout << "null\n";
}

// 전체 메모리 해제
void clear(Node*& head) {
    while (head) {
        Node* temp = head;
        head = head->next;
        delete temp;
    }
}
```

**주의**: 위 코드는 간단한 예시로, 실제로는 클래스로 감싸서 RAII(생성/소멸 관리)를 적용하는 것이 좋습니다.

### 단일 연결 리스트의 연산별 시간 복잡도

| 연산 | 시간 복잡도 |
|------|------------|
| 머리(head)에 삽입/삭제 | $$O(1)$$ |
| 꼬리(tail)에 삽입 | tail 포인터를 유지하면 $$O(1)$$, 아니면 $$O(n)$$ |
| 중간 삽입/삭제 (해당 노드의 **직전 노드를 알고 있을 때**) | $$O(1)$$ |
| 특정 값을 가진 노드 탐색 | $$O(n)$$ |
| 인덱스로 접근 | $$O(n)$$ |

---

## 이중 연결 리스트 (Doubly Linked List)

각 노드가 **이전 노드 포인터(`prev`)**와 **다음 노드 포인터(`next`)**를 모두 가집니다.

```
null <- [prev | data | next] <-> [prev | data | next] <-> [prev | data | next] -> null
```

- 장점: **양방향 탐색** 가능, **임의의 노드 삭제**가 더 쉽다(직전 노드를 몰라도 됨).
- 단점: 포인터를 2개 저장하므로 메모리를 더 사용합니다.

### 센티넬(sentinel) 노드를 이용한 구현

센티넬(더미 노드)을 사용하면 **경계 조건**을 단순화할 수 있습니다.  
`head`와 `tail`을 각각 더미 노드로 만들고, 실제 데이터는 그 사이에 위치시킵니다.

```
head(더미) <-> [data] <-> [data] <-> ... <-> tail(더미)
```

이렇게 하면 빈 리스트도 `head`와 `tail`이 서로 연결되어 있어, 항상 같은 코드로 삽입/삭제를 처리할 수 있습니다.

#### 템플릿을 사용한 이중 연결 리스트 클래스

```cpp
// dlist.hpp
#pragma once
#include <cstddef>
#include <iterator>
#include <stdexcept>
#include <utility>

template <typename T>
class dlist {
private:
    struct Node {
        T data;
        Node* prev;
        Node* next;

        template <typename... Args>
        explicit Node(Args&&... args)
            : data(std::forward<Args>(args)...), prev(nullptr), next(nullptr) {}
    };

    Node* head_;   // 더미 헤드 (데이터 없음)
    Node* tail_;   // 더미 테일 (데이터 없음)
    size_t size_;  // 실제 노드 개수

public:
    // 생성자: head와 tail을 만들고 서로 연결
    dlist() : head_(new Node()), tail_(new Node()), size_(0) {
        head_->next = tail_;
        tail_->prev = head_;
    }

    // 소멸자: 모든 노드 삭제
    ~dlist() {
        clear();
        delete head_;
        delete tail_;
    }

    // 복사 생성자와 대입 연산자는 일단 금지 (간단하게)
    dlist(const dlist&) = delete;
    dlist& operator=(const dlist&) = delete;

    bool empty() const { return size_ == 0; }
    size_t size() const { return size_; }

    // 맨 앞/뒤 원소 접근
    T& front() {
        if (empty()) throw std::out_of_range("list is empty");
        return head_->next->data;
    }
    const T& front() const {
        if (empty()) throw std::out_of_range("list is empty");
        return head_->next->data;
    }
    T& back() {
        if (empty()) throw std::out_of_range("list is empty");
        return tail_->prev->data;
    }
    const T& back() const {
        if (empty()) throw std::out_of_range("list is empty");
        return tail_->prev->data;
    }

    // 반복자 (양방향 반복자)
    class iterator {
    private:
        Node* node_;
    public:
        using iterator_category = std::bidirectional_iterator_tag;
        using value_type        = T;
        using difference_type   = std::ptrdiff_t;
        using pointer           = T*;
        using reference         = T&;

        iterator(Node* n = nullptr) : node_(n) {}

        reference operator*() const { return node_->data; }
        pointer   operator->() const { return &node_->data; }

        iterator& operator++() { node_ = node_->next; return *this; }
        iterator  operator++(int) { iterator tmp = *this; ++(*this); return tmp; }
        iterator& operator--() { node_ = node_->prev; return *this; }
        iterator  operator--(int) { iterator tmp = *this; --(*this); return tmp; }

        bool operator==(const iterator& other) const { return node_ == other.node_; }
        bool operator!=(const iterator& other) const { return !(*this == other); }

        friend class dlist;
    };

    iterator begin() { return iterator(head_->next); }
    iterator end()   { return iterator(tail_); }

    // 삽입: pos 위치 앞에 새 노드를 삽입하고, 삽입된 노드의 반복자 반환
    template <typename... Args>
    iterator emplace(iterator pos, Args&&... args) {
        Node* p = pos.node_;               // 삽입 위치 다음 노드
        Node* prev = p->prev;               // 삽입 위치 앞 노드

        Node* newNode = new Node(std::forward<Args>(args)...);
        // 앞 노드와 새 노드 연결
        prev->next = newNode;
        newNode->prev = prev;
        // 새 노드와 다음 노드 연결
        newNode->next = p;
        p->prev = newNode;

        ++size_;
        return iterator(newNode);
    }

    void push_front(const T& v) { emplace(begin(), v); }
    void push_back(const T& v)  { emplace(end(), v); }

    // 삭제: pos가 가리키는 노드 삭제, 다음 노드의 반복자 반환
    iterator erase(iterator pos) {
        Node* p = pos.node_;
        if (p == head_ || p == tail_)  // 더미 노드는 삭제 불가
            throw std::out_of_range("cannot erase sentinel");

        Node* prev = p->prev;
        Node* next = p->next;

        prev->next = next;
        next->prev = prev;

        delete p;
        --size_;
        return iterator(next);
    }

    void pop_front() { erase(begin()); }
    void pop_back()  { erase(--end()); }

    void clear() {
        Node* cur = head_->next;
        while (cur != tail_) {
            Node* next = cur->next;
            delete cur;
            cur = next;
        }
        head_->next = tail_;
        tail_->prev = head_;
        size_ = 0;
    }
};
```

**주요 포인트**:

- `emplace`는 생성자 인자를 직접 받아 노드를 **생성**하고, 적절한 위치에 삽입합니다. (perfect forwarding 사용)
- `erase`는 노드 삭제 후 다음 노드의 반복자를 반환하여, 반복문에서 안전하게 사용할 수 있습니다.
- 모든 연산에서 **경계 조건**(맨 앞/맨 뒤)을 센티넬 덕분에 별도 분기 없이 처리합니다.

### 사용 예시

```cpp
#include "dlist.hpp"
#include <iostream>

int main() {
    dlist<int> list;
    list.push_back(10);
    list.push_back(20);
    list.push_front(5);

    for (auto it = list.begin(); it != list.end(); ++it) {
        std::cout << *it << " ";   // 5 10 20
    }
    std::cout << "\n";

    list.pop_front();              // 10 20
    list.pop_back();               // 10

    std::cout << "front: " << list.front() << "\n"; // 10
    return 0;
}
```

---

## 원형 연결 리스트 (Circular Linked List)

마지막 노드의 `next`가 첫 노드를 가리키는 형태입니다.  
단일 원형, 이중 원형 모두 가능합니다.

- 장점: **tail에서 head로 바로 이동**할 수 있어, **라운드 로빈 스케줄러**나 **플레이리스트**에 유용합니다.
- 단점: 무한 루프를 조심해야 하며, 종료 조건을 잘 처리해야 합니다.

### 단일 원형 리스트 구현 (tail 포인터 사용)

```cpp
struct CNode {
    int data;
    CNode* next;
    explicit CNode(int val) : data(val), next(this) {} // 처음엔 자기 자신
};

// tail을 전달받아, 뒤에 새 노드 추가
void push_back(CNode*& tail, int val) {
    CNode* newNode = new CNode(val);
    if (!tail) {
        tail = newNode;
        return;
    }
    newNode->next = tail->next;  // 새 노드의 next를 head로
    tail->next = newNode;         // 기존 tail이 새 노드를 가리킴
    tail = newNode;                // 새 노드가 새로운 tail
}

// 출력 (head부터 시작)
void print(CNode* tail) {
    if (!tail) {
        std::cout << "(empty)\n";
        return;
    }
    CNode* head = tail->next;
    CNode* cur = head;
    do {
        std::cout << cur->data << " -> ";
        cur = cur->next;
    } while (cur != head);
    std::cout << "(head)\n";
}

// 메모리 해제
void clear(CNode*& tail) {
    if (!tail) return;
    CNode* head = tail->next;
    CNode* cur = head;
    do {
        CNode* temp = cur;
        cur = cur->next;
        delete temp;
    } while (cur != head);
    tail = nullptr;
}
```

---

## 고급 연산 (이중 연결 리스트 기준)

### 1. 리스트 뒤집기 (reverse)

각 노드의 `prev`와 `next`를 맞바꾸고, head와 tail도 교체합니다.  
센티넬이 있으면 더 간단합니다.

```cpp
void reverse() {
    if (size_ < 2) return;
    Node* cur = head_;
    while (cur) {
        std::swap(cur->prev, cur->next);
        cur = cur->prev;  // 원래 next였던 곳으로 이동 (swap 후 prev가 원래 next)
    }
    std::swap(head_, tail_); // head와 tail 센티넬 교환
}
```

### 2. 조건에 맞는 원소 제거 (remove_if)

```cpp
template <typename Pred>
void remove_if(Pred pred) {
    for (auto it = begin(); it != end(); ) {
        if (pred(*it))
            it = erase(it);
        else
            ++it;
    }
}
```

### 3. 인접한 중복 제거 (unique)

리스트가 정렬되어 있다고 가정하고, 연속된 중복 값을 하나만 남깁니다.

```cpp
void unique() {
    if (size_ < 2) return;
    auto it = begin();
    auto jt = it; ++jt;
    while (jt != end()) {
        if (*it == *jt)
            jt = erase(jt);
        else {
            it = jt;
            ++jt;
        }
    }
}
```

### 4. 정렬된 두 리스트 병합 (merge)

두 리스트가 모두 정렬되어 있다고 가정하고, 하나로 합칩니다.  
`std::list::merge`와 유사하게, **상대 리스트는 비게 됩니다**.

```cpp
template <typename Less = std::less<T>>
void merge(dlist& other, Less less = Less()) {
    if (this == &other || other.empty()) return;

    auto a = begin();
    auto b = other.begin();
    while (b != other.end()) {
        // a가 가리키는 값보다 b가 크거나 같을 때까지 a 이동
        while (a != end() && !less(*b, *a))
            ++a;
        // b를 a 앞에 삽입 (즉, splice)
        Node* bn = b.node_;           // other의 현재 노드
        ++b;                           // 다음으로 이동
        // other에서 bn 분리
        bn->prev->next = bn->next;
        bn->next->prev = bn->prev;
        --other.size_;

        // this에 bn 삽입 (a 앞)
        Node* ap = a.node_;            // a가 가리키는 노드
        Node* ap_prev = ap->prev;
        ap_prev->next = bn;
        bn->prev = ap_prev;
        bn->next = ap;
        ap->prev = bn;
        ++size_;
    }
}
```

### 5. 구간 이동 (splice)

한 리스트의 노드 구간 `[first, last)`를 잘라내어 다른 리스트의 `pos` 앞에 붙입니다.

```cpp
void splice(iterator pos, dlist& from, iterator first, iterator last) {
    if (first == last) return;

    // 구간의 크기 계산 (first부터 last 직전까지)
    size_t moved = 0;
    for (auto it = first; it != last; ++it) ++moved;

    // from에서 구간 분리
    Node* A = first.node_;
    Node* B = last.node_->prev;  // 구간의 마지막 노드
    Node* ap = A->prev;
    Node* bn = B->next;

    ap->next = bn;
    bn->prev = ap;
    from.size_ -= moved;

    // this에 연결 (pos 앞)
    Node* P = pos.node_;
    Node* pp = P->prev;
    pp->next = A;
    A->prev = pp;
    B->next = P;
    P->prev = B;
    size_ += moved;
}
```

`splice`는 노드를 **복사하지 않고** 포인터만 조작하므로 매우 빠릅니다.

---

## 실전 예제: LRU 캐시 (Least Recently Used)

LRU 캐시는 **가장 최근에 사용된 항목**을 유지하고, 용량이 초과되면 **가장 오래된 항목**을 제거합니다.

- **이중 연결 리스트**: 사용 순서를 유지 (맨 앞이 MRU, 맨 뒤가 LRU)
- **해시 테이블** (`unordered_map`): 키로 노드의 반복자를 빠르게 찾음

```cpp
#include <unordered_map>
#include <optional>
#include "dlist.hpp"

template <typename Key, typename Value>
class LRUCache {
    using List = dlist<std::pair<Key, Value>>;
    using Iterator = typename List::iterator;

    List list_;
    std::unordered_map<Key, Iterator> map_;
    size_t capacity_;

public:
    LRUCache(size_t cap) : capacity_(cap) {}

    std::optional<Value> get(const Key& key) {
        auto it = map_.find(key);
        if (it == map_.end())
            return std::nullopt;

        // 해당 노드를 맨 앞으로 이동 (splice)
        list_.splice(list_.begin(), list_, it->second, std::next(it->second));
        return it->second->second;
    }

    void put(const Key& key, const Value& value) {
        auto it = map_.find(key);
        if (it != map_.end()) {
            // 이미 존재하면 값 갱신하고 맨 앞으로 이동
            it->second->second = value;
            list_.splice(list_.begin(), list_, it->second, std::next(it->second));
            return;
        }

        // 새 항목 삽입 (맨 앞)
        list_.emplace_front(key, value);
        map_[key] = list_.begin();

        if (map_.size() > capacity_) {
            // 가장 오래된 항목(맨 뒤) 제거
            auto last = --list_.end();
            map_.erase(last->first);
            list_.pop_back();
        }
    }
};
```

사용 예:

```cpp
#include <iostream>

int main() {
    LRUCache<int, std::string> cache(2);
    cache.put(1, "one");
    cache.put(2, "two");
    std::cout << cache.get(1).value_or("-") << "\n"; // one (1이 최근이 됨)
    cache.put(3, "three");                            // 2는 가장 오래됐으므로 제거
    std::cout << cache.get(2).value_or("-") << "\n"; // - (없음)
    std::cout << cache.get(3).value_or("-") << "\n"; // three
    return 0;
}
```

---

## 성능 및 캐시 고려사항

- **캐시 지역성**: 연결 리스트의 노드는 메모리 여기저기 흩어져 있어, 순회할 때마다 캐시 미스가 발생합니다. 따라서 **순회 속도**는 배열보다 훨씬 느립니다.
- **메모리 오버헤드**: 각 노드마다 포인터(들)를 저장해야 하므로, 작은 데이터를 많이 저장할수록 오버헤드가 커집니다.
- **적합한 상황**:
  - 대량의 삽입/삭제가 빈번하고, 그 위치를 이미 알고 있을 때.
  - 데이터 크기가 커서 복사 비용이 매우 클 때 (예: 큰 문자열, 객체).
  - 안정적인 반복자 유지가 중요할 때 (리스트는 삽입/삭제 시 다른 노드의 반복자는 무효화되지 않음).

---

## 반복자 무효화 규칙

연결 리스트의 반복자는 **노드의 주소**를 직접 가리키므로, 해당 노드가 삭제되거나 다른 리스트로 이동(`splice`)되면 무효화됩니다.  
하지만 **삽입**은 기존 노드에 영향을 주지 않으므로, 삽입 전에 얻은 반복자도 계속 유효합니다. (단, `begin()`이나 `end()`는 삽입 후 변경될 수 있습니다.)

---

## 테스트 전략

1. **기능 테스트**: 각 연산(삽입, 삭제, 역순, 병합 등)이 예상대로 동작하는지 확인.
2. **경계 조건 테스트**: 빈 리스트, 노드가 하나만 있는 경우, 맨 앞/뒤 조작 등.
3. **퍼징 테스트**: 무작위 연산을 반복하며 표준 컨테이너(`std::list`)와 결과를 비교.

퍼징 예시 (앞서 본 `dlist` 클래스에 대해):

```cpp
#include <list>
#include <random>
#include <cassert>

void test_dlist() {
    dlist<int> my;
    std::list<int> ref;
    std::mt19937 rng(123);
    std::uniform_int_distribution<int> op(0, 5), val(0, 1000);

    for (int t = 0; t < 100000; ++t) {
        int o = op(rng);
        if (o == 0) { // push_front
            int x = val(rng);
            my.push_front(x);
            ref.push_front(x);
        } else if (o == 1) { // push_back
            int x = val(rng);
            my.push_back(x);
            ref.push_back(x);
        } else if (o == 2 && !ref.empty()) { // pop_front
            my.pop_front();
            ref.pop_front();
        } else if (o == 3 && !ref.empty()) { // pop_back
            my.pop_back();
            ref.pop_back();
        } else if (o == 4 && !ref.empty()) { // erase random
            size_t k = val(rng) % ref.size();
            auto it = my.begin(); auto jt = ref.begin();
            for (size_t i = 0; i < k; ++i) { ++it; ++jt; }
            my.erase(it);
            ref.erase(jt);
        } else if (o == 5) { // reverse
            my.reverse();
            ref.reverse();
        }

        // 두 컨테이너의 내용이 같은지 확인
        assert(my.size() == ref.size());
        auto it1 = my.begin();
        auto it2 = ref.begin();
        for (; it1 != my.end() && it2 != ref.end(); ++it1, ++it2)
            assert(*it1 == *it2);
        assert(it1 == my.end() && it2 == ref.end());
    }
}
```

---

## 마무리

연결 리스트는 포인터를 직접 다루는 가장 기초적인 동적 자료구조입니다.  
단일/이중/원형 리스트의 구현을 익히고, 고급 연산(병합, 뒤집기, 스플라이스)을 추가하면 **자료구조의 동작 원리**를 깊이 이해할 수 있습니다.

실무에서는 `std::forward_list`(단일)나 `std::list`(이중)를 사용하는 것이 일반적이지만, 내부 동작을 이해하면 더 효율적인 코드를 작성하는 데 도움이 됩니다.

다음 단계로는 **intrusive list**(노드 구조체 안에 포인터를 두는 방식)나 **lock-free 리스트** 등을 공부해보세요.