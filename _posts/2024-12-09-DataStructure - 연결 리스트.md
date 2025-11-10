---
layout: post
title: Data Structure - 연결 리스트
date: 2024-12-09 19:20:23 +0900
category: Data Structure
---
# 연결 리스트 (Linked List)

기존 초안의 **핵심(정의·기본 구조·시간 복잡도·간단 구현)**을 바탕으로, 본 글은 **실전 구현과 확장**을 모두 다룬다:

- 단일/이중/원형 리스트의 **클래스형 템플릿 구현**
- **센티넬 노드**(dummy head/tail)로 경계 단순화
- **반복자 설계**(전/양방향), 예외 안전, 리소스 관리(RAII)
- 고급 연산: `splice`(노드 재연결), `reverse`, `remove_if`, `unique`, `merge`(정렬), 사이클 검출(Floyd), 중간/뒤에서 k번째
- **LRU 캐시**(이중 리스트 + 해시) 예제
- 원형 리스트의 tail 포인터 기법
- **복잡도·캐시 지역성**·반복자 무효화 규칙·테스트 전략

---

## 0. 왜 배열 대신 연결 리스트인가?

- **장점**: 중간 삽입/삭제가 **포인터 재배선만으로 O(1)** (해당 지점까지 가는 비용은 O(n))
- **단점**: 랜덤 접근이 **O(n)**, 데이터가 **비연속** → **캐시 비우호**, 상수항이 큼
- **정리**: “중간 조작이 아주 잦다 + 큰 객체의 이동/복사가 비싼” 상황에 적합. 그 외엔 `vector`/`deque`가 대개 더 빠름.

---

## 1. 단일 연결 리스트 (Singly Linked List)

### 1.1 핵심 구조

```cpp
struct SNode {
    int data;
    SNode* next;
};
```

- 임의 노드 **앞에 삽입**은 해당 노드의 **직전 노드를 알아야** O(1).
- 헤드 삽입/삭제는 O(1).

### 1.2 안전한 도우미 (C++17)

```cpp
#include <iostream>
#include <functional>
#include <stdexcept>

struct SNode {
    int data;
    SNode* next;
    explicit SNode(int v, SNode* n=nullptr) : data(v), next(n) {}
};

void push_front(SNode*& head, int v) {
    head = new SNode(v, head);
}

void pop_front(SNode*& head) {
    if (!head) throw std::out_of_range("pop_front on empty");
    SNode* old = head; head = head->next; delete old;
}

void print(SNode* head) {
    for (SNode* cur = head; cur; cur = cur->next) std::cout << cur->data << " -> ";
    std::cout << "NULL\n";
}

void clear(SNode*& head) {
    while (head) { SNode* t = head; head = head->next; delete t; }
}
```

### 1.3 고급 연산

#### (1) 역순 뒤집기 — 반복

```cpp
SNode* reverse_iter(SNode* head) {
    SNode* prev=nullptr; SNode* cur=head;
    while (cur) {
        SNode* nxt = cur->next;
        cur->next  = prev;
        prev = cur; cur = nxt;
    }
    return prev;
}
```

#### (2) 중간 노드 찾기 — runner(토끼/거북)

```cpp
SNode* middle(SNode* head){
    SNode* slow=head; SNode* fast=head;
    while (fast && fast->next) { slow=slow->next; fast=fast->next->next; }
    return slow;
}
```

#### (3) 뒤에서 k번째

```cpp
SNode* kth_from_end(SNode* head, int k){
    SNode* a=head; SNode* b=head;
    while (k-- && b) b=b->next;
    if (k>=0) return nullptr; // 길이 부족
    while (b){ a=a->next; b=b->next; }
    return a;
}
```

#### (4) 사이클 검출 — Floyd

```cpp
bool has_cycle(SNode* head){
    SNode* slow=head; SNode* fast=head;
    while (fast && fast->next){
        slow=slow->next; fast=fast->next->next;
        if (slow==fast) return true;
    }
    return false;
}
```

---

## 2. 이중 연결 리스트 (Doubly Linked List)

### 2.1 구조 & 장점

```cpp
struct DNode {
    int data;
    DNode* prev;
    DNode* next;
};
```

- 양방향 이동 가능. **삭제가 O(1)** (자기 포인터만 있으면 직전 탐색 불필요)
- 단점: 포인터 2개 → 더 많은 메모리

### 2.2 센티넬 기반 템플릿 구현 (RAII)

센티넬(더미) 노드 두 개(head/tail)을 두면 **경계 처리가 단순**해진다.

```cpp
// dlist.hpp
#pragma once
#include <cstddef>
#include <iterator>
#include <stdexcept>
#include <utility>
#include <functional>

template <class T>
class dlist {
    struct node {
        T     data;
        node* prev;
        node* next;
        template <class... Args>
        explicit node(Args&&... args) : data(std::forward<Args>(args)...), prev(nullptr), next(nullptr) {}
    };
    node* head_; // sentinel head
    node* tail_; // sentinel tail
    std::size_t sz_{};

public:
    dlist() {
        head_ = new node(); tail_ = new node();
        head_->next = tail_; tail_->prev = head_;
    }
    ~dlist(){ clear(); delete head_; delete tail_; }

    dlist(const dlist&) = delete;
    dlist& operator=(const dlist&) = delete;

    bool empty() const noexcept { return sz_==0; }
    std::size_t size() const noexcept { return sz_; }

    T& front(){ if(empty()) throw std::out_of_range("empty"); return head_->next->data; }
    T& back (){ if(empty()) throw std::out_of_range("empty"); return tail_->prev->data; }
    const T& front() const { if(empty()) throw std::out_of_range("empty"); return head_->next->data; }
    const T& back () const { if(empty()) throw std::out_of_range("empty"); return tail_->prev->data; }

    // 반복자 (양방향)
    class iterator {
        node* n_=nullptr;
    public:
        using difference_type = std::ptrdiff_t;
        using value_type = T;
        using reference = T&;
        using pointer = T*;
        using iterator_category = std::bidirectional_iterator_tag;

        iterator()=default; explicit iterator(node* n):n_(n){}
        reference operator*() const { return n_->data; }
        pointer   operator->() const { return &n_->data; }
        iterator& operator++(){ n_=n_->next; return *this; }
        iterator  operator++(int){ auto t=*this; ++*this; return t; }
        iterator& operator--(){ n_=n_->prev; return *this; }
        iterator  operator--(int){ auto t=*this; --*this; return t; }
        bool operator==(const iterator& o) const { return n_==o.n_; }
        bool operator!=(const iterator& o) const { return !(*this==o); }
        friend class dlist;
    };

    iterator begin() const { return iterator(head_->next); }
    iterator end()   const { return iterator(tail_); }

    // 기본 조작
    template <class... Args> iterator emplace(iterator pos, Args&&... args){
        node* p = pos.n_;
        node* x = new node(std::forward<Args>(args)...);
        node* a = p->prev;
        a->next = x; x->prev = a;
        x->next = p; p->prev = x;
        ++sz_; return iterator(x);
    }
    template <class... Args> void emplace_front(Args&&... args){ emplace(begin(), std::forward<Args>(args)...); }
    template <class... Args> void emplace_back (Args&&... args){ emplace(end(),   std::forward<Args>(args)...);  }

    void push_front(const T& v){ emplace_front(v); }
    void push_back (const T& v){ emplace_back (v); }

    iterator erase(iterator pos){
        node* p = pos.n_;
        if (p==head_ || p==tail_) throw std::out_of_range("erase sentinel");
        node* a = p->prev; node* b = p->next;
        a->next = b; b->prev = a;
        delete p; --sz_;
        return iterator(b);
    }

    void pop_front(){ if(empty()) throw std::out_of_range("empty"); erase(begin()); }
    void pop_back (){ if(empty()) throw std::out_of_range("empty"); auto it=end(); --it; erase(it); }

    void clear() noexcept {
        node* cur = head_->next;
        while (cur != tail_) { node* nx=cur->next; delete cur; cur=nx; }
        head_->next = tail_; tail_->prev = head_; sz_=0;
    }

    // 유틸 연산
    void reverse(){
        if (sz_<2) return;
        node* cur = head_;
        while (cur){
            std::swap(cur->prev, cur->next);
            cur = cur->prev; // 기존 next
        }
        std::swap(head_, tail_);
    }

    template <class Pred>
    void remove_if(Pred pred){
        for(auto it=begin(); it!=end(); ){
            if (pred(*it)) it=erase(it);
            else ++it;
        }
    }

    void unique(){ // 인접 중복 제거(정렬 가정)
        if (sz_<2) return;
        auto it=begin(); auto jt=it; ++jt;
        while (jt!=end()){
            if (*it==*jt) jt=erase(jt);
            else { it=jt; ++jt; }
        }
    }

    // 정렬된 두 리스트 병합 (자기 안으로 흡수)
    template <class Less=std::less<T>>
    void merge(dlist& other, Less less = Less()){
        if (&other==this || other.empty()) return;
        auto a=begin(), ae=end();
        auto b=other.begin(), be=other.end();
        while (b!=be){
            while (a!=ae && !less(*b,*a)) ++a;
            // b를 *a 앞에 splice 1개 이동
            node* bn = b.n_;
            ++b;
            // other에서 제거
            bn->prev->next = bn->next;
            bn->next->prev = bn->prev;
            --other.sz_;
            // this에 삽입 (a 앞)
            node* ap = a.n_;
            node* ap_prev = ap->prev;
            ap_prev->next = bn; bn->prev = ap_prev;
            bn->next  = ap;    ap->prev = bn;
            ++sz_;
        }
    }

    // 구간 이동(splice): [first,last) 를 pos 앞에 이동 (동일 컨테이너/타 컨테이너 지원)
    void splice(iterator pos, dlist& from, iterator first, iterator last){
        if (first==last) return;
        // 구간 크기 계산
        std::size_t moved=0; for(auto it=first; it!=last; ++it) ++moved;

        // from에서 잘라내기
        node* A = first.n_; node* B = last.n_->prev; // [A..B]
        node* ap = A->prev; node* bn = B->next;
        ap->next = bn; bn->prev = ap;
        from.sz_ -= moved;

        // this에 연결: pos 앞
        node* P = pos.n_; node* pp = P->prev;
        pp->next = A; A->prev = pp;
        B->next  = P; P->prev = B;
        sz_ += moved;
    }
};
```

**포인트**

- 센티넬로 `head_`와 `tail_` 사이만 **실데이터**. 빈 리스트/양끝 처리 코드가 단순해짐.
- `splice`/`merge`는 **노드 재연결만** 수행하므로 **각 노드의 이동이 O(1)**, 전체 O(k).
- 반복자 무효화 규칙:
  - 특정 노드가 **삭제/이동**되면 그 노드에 대한 반복자는 무효.
  - 그 외 노드들의 반복자는 유지(연결만 바뀌므로).

### 2.3 사용 예

```cpp
#include "dlist.hpp"
#include <iostream>

int main(){
    dlist<int> a, b;
    a.push_back(1); a.push_back(3); a.push_back(5);
    b.push_back(2); b.push_back(4); b.push_back(6);

    // 정렬 병합
    a.merge(b); // a: 1 2 3 4 5 6, b: empty
    for(auto x: a) std::cout<<x<<" "; std::cout<<"\n";

    // 부분 스플라이스: 뒤 3개를 앞으로
    auto it=a.begin(); ++it; ++it; // 3 가리킴
    auto it_end=a.end();
    a.splice(a.begin(), a, it, it_end); // [3..6]을 맨 앞으로
    for(auto x: a) std::cout<<x<<" "; std::cout<<"\n";

    // unique 테스트(연속 중복 제거)
    dlist<int> c;
    c.push_back(1); c.push_back(1); c.push_back(1);
    c.push_back(2); c.push_back(2); c.push_back(3);
    c.unique();
    for(auto x: c) std::cout<<x<<" "; std::cout<<"\n";
}
```

---

## 3. 원형 연결 리스트 (Circular Linked List)

### 3.1 단일 원형 — tail 포인터만으로 O(1) 끝삽입

```cpp
struct CNode {
    int data;
    CNode* next;
    explicit CNode(int v): data(v), next(this) {} // 자기 자신
};

void push_back(CNode*& tail, int v){
    CNode* n = new CNode(v);
    if (!tail) { tail = n; return; }
    n->next = tail->next;   // head
    tail->next = n;
    tail = n;               // 새 tail
}

void print_c(CNode* tail){
    if (!tail) { std::cout<<"(empty)\n"; return; }
    CNode* cur = tail->next; // head
    do { std::cout<<cur->data<<" -> "; cur=cur->next; } while(cur!=tail->next);
    std::cout<<"(head)\n";
}

void clear_c(CNode*& tail){
    if (!tail) return;
    CNode* head = tail->next; CNode* cur=head;
    do{ CNode* t=cur; cur=cur->next; delete t; } while(cur!=head);
    tail=nullptr;
}
```

- 장점: `tail`만 알면 `push_back`이 **O(1)**.
- 순회 종료 조건을 `cur == head`로 둔다(무한 루프 주의).

### 3.2 이중 원형 + 센티넬 = `std::list`의 전형

- `head_`만 센티넬로 두고, `head_.next`부터 `head_`로 돌아오면 종료.
- `reverse`/`splice`/`merge` 모두 **포인터 교환**만으로 처리 가능.

---

## 4. 실전: LRU 캐시(이중 리스트 + 해시)

- 규칙:
  - 조회/삽입 시 노드를 **맨 앞으로 이동**(가장 최근).
  - 용량 초과 시 **맨 뒤**(가장 오래된) 제거.
- 자료구조:
  - **이중 리스트**: 사용 순서 유지
  - **해시**: 키 → 노드 반복자(또는 포인터)로 O(1) 조회

```cpp
#include <unordered_map>
#include <optional>
#include "dlist.hpp"

template <class Key, class Val, class Hash=std::hash<Key>, class Eq=std::equal_to<Key>>
class LRU {
    using Pair = std::pair<Key, Val>;
    dlist<Pair> list_; // front: MRU, back: LRU
    std::unordered_map<Key, typename dlist<Pair>::iterator, Hash, Eq> map_;
    std::size_t cap_;

public:
    explicit LRU(std::size_t cap): cap_(cap) {}

    std::optional<Val> get(const Key& k){
        auto it = map_.find(k);
        if (it==map_.end()) return std::nullopt;
        // to front (splice 1개 노드)
        auto node = it->second;
        list_.splice(list_.begin(), list_, node, std::next(node));
        return node->second;
    }

    void put(const Key& k, Val v){
        auto it = map_.find(k);
        if (it!=map_.end()){
            // 업데이트 + 이동
            it->second->second = std::move(v);
            list_.splice(list_.begin(), list_, it->second, std::next(it->second));
            return;
        }
        // 신규
        list_.emplace_front(k, std::move(v));
        map_[k] = list_.begin();
        if (map_.size() > cap_){
            // remove LRU (back)
            auto tail_it = list_.end(); --tail_it;
            map_.erase(tail_it->first);
            list_.pop_back();
        }
    }
};
```

테스트:

```cpp
#include <iostream>
int main(){
    LRU<int, std::string> cache(2);
    cache.put(1, "one");
    cache.put(2, "two");
    std::cout << cache.get(1).value_or("-") << "\n"; // use 1 -> (1,2)
    cache.put(3,"three"); // evict 2
    std::cout << cache.get(2).value_or("-") << "\n"; // -
    std::cout << cache.get(3).value_or("-") << "\n"; // three
}
```

---

## 5. 알고리즘 팁 & 패턴 모음

### 5.1 리스트 반 나누기 + 병합 정렬(O(n log n), 안정)

- 단일 리스트에서도 포인터만으로 정렬 가능(추가 배열 불필요)
- **중간 찾기** + **두 반 병합** 반복

```cpp
SNode* merge_sorted(SNode* a, SNode* b){
    SNode dummy(0), *t=&dummy;
    while (a && b){
        if (a->data <= b->data){ t->next=a; a=a->next; }
        else { t->next=b; b=b->next; }
        t=t->next;
    }
    t->next = a?a:b; return dummy.next;
}

SNode* merge_sort(SNode* h){
    if (!h || !h->next) return h;
    // split
    SNode* slow=h; SNode* fast=h->next;
    while (fast && fast->next){ slow=slow->next; fast=fast->next->next; }
    SNode* mid = slow->next; slow->next=nullptr;
    return merge_sorted(merge_sort(h), merge_sort(mid));
}
```

### 5.2 노드 삭제 without prev (단일 리스트 트릭)

- **삭제할 노드 포인터만 있고 prev가 없을 때**: 다음 노드의 데이터를 복사해 덮고, 다음 노드를 제거  
  (단, **마지막 노드에는 불가**)

```cpp
bool erase_without_prev(SNode* x){
    if (!x || !x->next) return false;
    SNode* n = x->next;
    x->data  = n->data;
    x->next  = n->next;
    delete n;
    return true;
}
```

---

## 6. 성능·복잡도·캐시

### 6.1 시간 복잡도

| 연산 | 단일 | 이중 | 원형 |
|---|---|---|---|
| head 삽입/삭제 | O(1) | O(1) | O(1) |
| tail 삽입 | O(n)\* | O(1)\*\* | O(1) (tail 보유) |
| 중간 삽입/삭제 (노드 지점 제공) | O(1) | O(1) | O(1) |
| 임의 접근 | O(n) | O(n) | O(n) |

- \* 단일에서 tail 포인터를 유지하면 tail 삽입도 O(1).
- \*\* 이중은 tail 포인터가 자연스럽게 존재(센티넬의 prev).

### 6.2 캐시/상수항

- 연결 리스트는 **노드가 흩어져** 있어 **캐시 미스 빈번** → **상수항이 큼**.
- 같은 O(n)이라도 `vector` 순차순회가 훨씬 빠를 수 있음.
- 큰 객체를 **이동 없이 연결**시키고 싶을 때 여전히 가치가 있다(예: 거대한 트리 노드의 리스트 유지).

---

## 7. 메모리·안전성·반복자 규칙

- 모든 `new`는 정확히 대응되는 `delete` 필요(RAII 권장).
- **예외 안전**: 삽입 시 노드 생성 성공 이후에만 연결 업데이트. 실패 시 원복.
- 반복자 무효화:
  - `erase`된 노드의 반복자만 무효. 나머지는 유효(트리/벡터와 비교되는 장점).
  - `splice`로 **다른 컨테이너로 옮겨진 노드**의 기존 반복자는 **원 컨테이너**에 대해 무효, **새 컨테이너**에 대해 유효(반복자가 노드 주소를 가리키므로).

---

## 8. 테스트 전략

1. **브루트 대조**: 동일 시나리오를 `std::list`/`std::forward_list`와 동시 실행 & 결과 비교.
2. **경계**: 빈 컨테이너, 단일 노드, 양 끝에서의 삽입/삭제, self-splice 금지.
3. **퍼징**: 랜덤 연산 시퀀스(삽입/삭제/스플라이스/머지) 후 불변식 검사(크기, 양방향 일치).
4. **ASan/UBSan**: 런타임 메모리 오류 조기 발견.

간단 퍼저:

```cpp
#include "dlist.hpp"
#include <list>
#include <random>
#include <cassert>

int main(){
    dlist<int> my; std::list<int> ref;
    std::mt19937 rng(123);
    std::uniform_int_distribution<int> op(0,5), val(0,1000);

    for (int t=0;t<50000;++t){
        int o=op(rng);
        if (o==0){ int x=val(rng); my.push_front(x); ref.push_front(x); }
        else if (o==1){ int x=val(rng); my.push_back(x); ref.push_back(x); }
        else if (o==2 && !ref.empty()){ my.pop_front(); ref.pop_front(); }
        else if (o==3 && !ref.empty()){ my.pop_back();  ref.pop_back();  }
        else if (o==4 && !ref.empty()){ // erase random
            int k = val(rng) % (int)ref.size();
            auto it=my.begin(); auto jt=ref.begin();
            while(k--){ ++it; ++jt; }
            my.erase(it); ref.erase(jt);
        } else if (o==5){ my.reverse(); ref.reverse(); }

        // compare
        auto it=my.begin(); auto jt=ref.begin();
        for(; it!=my.end() && jt!=ref.end(); ++it, ++jt) assert(*it==*jt);
        assert(it==my.end() && jt==ref.end());
    }
}
```

---

## 9. 자주 하는 질문(FAQ)

- **Q. 리스트가 항상 느린가요?**  
  랜덤 접근·순차 스캔 위주면 `vector`가 유리. **노드 이동/병합/스플라이스** 같은 **연결 재배선**이 많으면 리스트가 강함.

- **Q. 왜 센티넬이 좋나요?**  
  빈/양끝 케이스 분기가 줄어 코드가 **짧고 견고**해진다. 특히 `splice/merge/reverse` 구현이 깔끔.

- **Q. 단일 vs 이중 선택?**  
  삭제에 **직전 노드**가 필요 없게 만드는 이중이 실전에서 더 편하고 안전. 메모리 여유가 있다면 이중 추천.

- **Q. 원형의 장점은?**  
  tail→head로 **순환**이 자연스러워 **라운드 로빈 스케줄러**, **플레이리스트** 등에서 편리. 종료 조건 주의.

---

## 10. 마무리

연결 리스트는 **포인터만으로 자료를 엮는** 가장 단순한 동적 구조다.  
센티넬·양방향 반복자·스플라이스·머지·LRU 등 실전 패턴을 익히면, “언제 리스트가 이기는지”를 명확히 판단할 수 있다.  

다음 단계 제안:
- `dlist<T>`에 **정렬(merge sort)**, **stable_partition** 추가
- **intrusive list**(노드가 데이터 안에 prev/next를 내장)로 할당 비용 제거
- **lock-free singly list**의 기본(ABA/hazard pointers) 맛보기