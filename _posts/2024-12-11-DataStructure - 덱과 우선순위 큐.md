---
layout: post
title: Data Structure - 덱과 우선순위 큐
date: 2024-12-11 19:20:23 +0900
category: Data Structure
---
# 덱(Deque)과 우선순위 큐(Priority Queue)

## 1. Deque — 양끝 입출력의 왕

> **Deque (Double-Ended Queue)**: 앞·뒤 **둘 다** `push/pop` 가능한 큐.

### 1.1 주요 연산과 복잡도

| 연산 | 의미 | 원형배열 | 연결리스트(이중) |
|---|---|---|---|
| `push_front/back` | 양끝 삽입 | 평균 O(1)\* | O(1) |
| `pop_front/back` | 양끝 삭제 | 평균 O(1)\* | O(1) |
| `front/back` | 양끝 조회 | O(1) | O(1) |
| `size/empty` | 크기/공백 | O(1) | O(1) |

\* 동적 확장 시 재배치가 **가끔** O(n)이지만 **평균(Amortized) O(1)**.

암ortized 직관:
$$
\text{총 이동} \approx 1+2+4+\cdots+2^k = O(2^k),\quad
\text{삽입 수} \approx 2^k \Rightarrow \text{평균} = O(1)
$$

---

### 1.2 STL `std::deque` 한 줄 요약

- **블록 기반 동적 배열**(segmented array): 중앙 확장도 저렴, 양끝 O(1)
- 반복자 무효화 규칙이 `vector`와 다름(블록 재배치 시 일부만 무효화)

```cpp
#include <deque>
#include <iostream>

std::deque<int> dq;
dq.push_back(10);
dq.push_front(20);     // [20,10]
dq.pop_back();         // [20]
std::cout << dq.front(); // 20
```

---

## 2. 연결 리스트 기반 덱(센티넬, 이중 링크)

경계(빈/단일 노드) 분기 제거를 위해 **센티넬 원형**을 권장.

```cpp
#include <stdexcept>

template <class T>
class DequeList {
    struct Node { T v; Node* prev; Node* next;
        template<class U> explicit Node(U&& x): v(std::forward<U>(x)), prev(nullptr), next(nullptr) {}
        DequeList::Node(): v(), prev(this), next(this) {} // sentinel
    };
    Node* s_; // sentinel: s_->next = front, s_->prev = back
    size_t n_{};

public:
    DequeList(){ s_ = new Node(); }
    ~DequeList(){ clear(); delete s_; }

    bool empty() const { return n_==0; }
    size_t size() const { return n_; }

    void clear(){
        for(Node* c=s_->next; c!=s_;){ Node* nx=c->next; delete c; c=nx; }
        s_->next=s_->prev=s_; n_=0;
    }

    template<class U> void push_front(U&& x){
        Node* nd = new Node(std::forward<U>(x));
        nd->next = s_->next; nd->prev = s_;
        s_->next->prev = nd; s_->next = nd; ++n_;
    }
    template<class U> void push_back(U&& x){
        Node* nd = new Node(std::forward<U>(x));
        nd->prev = s_->prev; nd->next = s_;
        s_->prev->next = nd; s_->prev = nd; ++n_;
    }
    void pop_front(){
        if (empty()) throw std::underflow_error("empty");
        Node* f = s_->next;
        s_->next = f->next; f->next->prev = s_;
        delete f; --n_;
    }
    void pop_back(){
        if (empty()) throw std::underflow_error("empty");
        Node* b = s_->prev;
        s_->prev = b->prev; b->prev->next = s_;
        delete b; --n_;
    }
    T& front(){ if (empty()) throw std::runtime_error("empty"); return s_->next->v; }
    T& back (){ if (empty()) throw std::runtime_error("empty"); return s_->prev->v; }
};
```

- 장점: **항상 중간 삽입 형태**로 처리 → 분기 단순·견고
- 단점: 노드 할당/포인터 추적 → 캐시 비우호

---

## 3. 동적 원형 배열 기반 Deque(템플릿) — 실전형

**front, back 인덱스**를 원형으로 굴리고, 가득 차면 **2배 확장** + **front부터 연속 복사**.

```cpp
#include <stdexcept>
#include <new>
#include <utility>

template <class T>
class DequeRing {
    T* buf_ = nullptr;
    int cap_ = 0;
    int f_ = 0; // front index
    int r_ = 0; // one past back
    int n_ = 0;

    void grow_(){
        int ncap = cap_? cap_*2 : 4;
        T* nb = static_cast<T*>(::operator new[](sizeof(T)*ncap));
        try{
            for(int i=0;i<n_;++i){
                new (&nb[i]) T(std::move(buf_[(f_+i)%cap_]));
                buf_[(f_+i)%cap_].~T();
            }
        }catch(...){
            for(int i=0;i<n_;++i) nb[i].~T();
            ::operator delete[](nb); throw;
        }
        ::operator delete[](buf_);
        buf_ = nb; cap_ = ncap; f_ = 0; r_ = n_;
    }

public:
    DequeRing() = default;
    ~DequeRing(){
        for(int i=0;i<n_;++i) buf_[(f_+i)%cap_].~T();
        ::operator delete[](buf_);
    }
    bool empty() const { return n_==0; }
    int  size () const { return n_; }

    void push_front(const T& x){
        if (n_==cap_) grow_();
        f_ = cap_? (f_ + cap_ - 1) % cap_ : 0;
        new (&buf_[f_]) T(x);
        ++n_; if (!cap_) { cap_=4; r_=n_; } // 초기화 보정(최초 grow_을 스킵하려면)
    }
    void push_front(T&& x){
        if (n_==cap_) grow_();
        f_ = cap_? (f_ + cap_ - 1) % cap_ : 0;
        new (&buf_[f_]) T(std::move(x));
        ++n_;
    }
    void push_back(const T& x){
        if (n_==cap_) grow_();
        new (&buf_[r_]) T(x);
        r_ = cap_? (r_+1)%cap_ : 0; ++n_;
    }
    void push_back(T&& x){
        if (n_==cap_) grow_();
        new (&buf_[r_]) T(std::move(x));
        r_ = cap_? (r_+1)%cap_ : 0; ++n_;
    }
    void pop_front(){
        if (empty()) throw std::underflow_error("empty");
        buf_[f_].~T(); f_ = (f_+1)%cap_; --n_;
    }
    void pop_back(){
        if (empty()) throw std::underflow_error("empty");
        r_ = (r_ + cap_ - 1)%cap_; buf_[r_].~T(); --n_;
    }
    T& front(){ if (empty()) throw std::runtime_error("empty"); return buf_[f_]; }
    T& back (){
        if (empty()) throw std::runtime_error("empty");
        int idx = (r_ + cap_ - 1) % cap_; return buf_[idx];
    }
};
```

- **모듈로 최적화**: cap을 2의 거듭제곱으로 고정하면 `(i & (cap-1))`로 더 빠르게 가능
- **실전 팁**: 구조체 내부에서 **소형 객체**를 담으면 캐시 친화성 상승

---

## 4. 모노토닉 덱 — 슬라이딩 윈도우 최댓값/최솟값

값(또는 인덱스)을 **단조** 상태로 유지: **뒤에서 더 나쁜 값 제거** → 각 원소가 **한 번** 들어오고 **한 번** 나가므로 O(n).

```cpp
#include <vector>
#include <deque>

std::vector<int> slidingMax(const std::vector<int>& a, int k){
    std::deque<int> dq; // store indices, values decreasing
    std::vector<int> ans;
    for (int i=0;i<(int)a.size();++i){
        while(!dq.empty() && a[dq.back()] <= a[i]) dq.pop_back();
        dq.push_back(i);
        if (dq.front() <= i-k) dq.pop_front();
        if (i >= k-1) ans.push_back(a[dq.front()]);
    }
    return ans;
}
```

- **최솟값**: 부등호 반대로
- **응용**: **정규화/필터링**, **스케줄링 가시창치 계산**, **주가/센서 스트림 처리**

---

## 5. 0–1 BFS — 덱의 대표 알고리즘

간선 가중치가 0 또는 1일 때, **덱**에 0-간선은 `push_front`, 1-간선은 `push_back`.
다익스트라 없이 O(V+E)로 최단 거리.

```cpp
#include <deque>
#include <vector>
#include <limits>

struct Edge{ int to; int w; }; // w ∈ {0,1}

std::vector<int> zero_one_bfs(int n, const std::vector<std::vector<Edge>>& g, int s){
    const int INF = std::numeric_limits<int>::max()/4;
    std::vector<int> d(n, INF);
    std::deque<int> dq;
    d[s]=0; dq.push_back(s);
    while(!dq.empty()){
        int u=dq.front(); dq.pop_front();
        for(auto [v,w]: g[u]){
            if (d[u]+w < d[v]){
                d[v]=d[u]+w;
                if (w==0) dq.push_front(v); else dq.push_back(v);
            }
        }
    }
    return d;
}
```

---

# Priority Queue — 우선순위가 먼저다

> 일반 큐는 FIFO, **우선순위 큐**는 비교 기준에 따라 **가장 높은 우선순위**가 먼저 나온다.
> 실전은 대부분 **힙(Heap)** 기반.

## 6. 이진 힙(Binary Heap) — 핵심만 단단하게

- **완전 이진 트리**를 **배열**로 저장 (루트 index 0)
- `push`: 말단 삽입 → **상향 힙화**(heapify-up)
- `pop`: 루트 제거 → 말단을 루트로 올림 → **하향 힙화**(heapify-down)

### 6.1 템플릿 최대 힙(커스텀 비교자)

```cpp
#include <vector>
#include <functional>
#include <stdexcept>
#include <utility>

template <class T, class Comp = std::less<T>> // less => max-heap
class BinaryHeap {
    std::vector<T> a;
    Comp cmp; // cmp(x,y)==true 이면 x<y (기본: less)

    bool higher(int i, int j) const { // i가 j보다 "우선"이면 true
        return cmp(a[j], a[i]); // max-heap: a[i] > a[j]
    }
    void sift_up(int i){
        while(i>0){
            int p=(i-1)/2;
            if (!higher(i,p)) break;
            std::swap(a[i],a[p]); i=p;
        }
    }
    void sift_down(int i){
        int n=(int)a.size();
        while(true){
            int l=i*2+1, r=l+1, best=i;
            if (l<n && higher(l,best)) best=l;
            if (r<n && higher(r,best)) best=r;
            if (best==i) break;
            std::swap(a[i],a[best]); i=best;
        }
    }

public:
    BinaryHeap() = default;
    explicit BinaryHeap(const Comp& c): cmp(c) {}

    bool empty() const { return a.empty(); }
    int  size () const { return (int)a.size(); }
    const T& top() const{
        if (empty()) throw std::runtime_error("empty");
        return a[0];
    }
    void push(const T& x){ a.push_back(x); sift_up((int)a.size()-1); }
    void push(T&& x){ a.push_back(std::move(x)); sift_up((int)a.size()-1); }
    void pop(){
        if (empty()) throw std::runtime_error("empty");
        a[0] = std::move(a.back()); a.pop_back();
        if (!a.empty()) sift_down(0);
    }
    // O(n) heapify from range
    template <class It>
    void build(It first, It last){
        a.assign(first,last);
        for(int i=(int)a.size()/2-1;i>=0;--i) sift_down(i);
    }
};
```

- **최대 힙**: `Comp = std::less<T>`
- **최소 힙**: `Comp = std::greater<T>`
- STL `std::priority_queue<T, vector<T>, Comp>`와 동일한 인터페이스 감각

---

### 6.2 `heapify`가 O(n)인 이유(직관)

완전 이진 트리에서 **아래 레벨** 노드가 압도적으로 많아 **작은 거리만 이동**:

$$
\sum_{\ell=0}^{h} \frac{n}{2^{\ell+1}} \cdot O(h-\ell) \in O(n)
$$

---

## 7. `std::priority_queue` 정확 사용

```cpp
#include <queue>
#include <vector>
#include <functional>

std::priority_queue<int> maxpq; // 최대 힙(기본)
std::priority_queue<int, std::vector<int>, std::greater<int>> minpq; // 최소 힙

struct Job { int id; int pri; };
struct Cmp { bool operator()(const Job& a, const Job& b) const { return a.pri < b.pri; } }; // max by pri
std::priority_queue<Job, std::vector<Job>, Cmp> jobs;
```

- **주의**: `priority_queue`는 **decrease-key 없음**. 값 갱신은 보통 **새 값을 push**하고 **오래된 값은 무시**(lazy deletion with timestamp) 전략 사용.

---

## 8. d-ary 힙 — 팬아웃으로 상수 개선

이진 힙의 자식 2개 대신 **d개**로 확장.
- **장점**: `sift_down` 단계 수 감소(높이 ↓)
- **단점**: `sift_up` 시 부모 계산/비교 증가

인덱스 규칙(0-based):
- 부모: \(\text{par}(i) = \lfloor (i-1)/d \rfloor\)
- 자식: \(\text{child}(i,k) = d\cdot i + 1 + k,\ 0 \le k < d\)

`d`는 **분기因子**. 캐시/분기 예측과 타협해 **4-ary**가 실전에서 자주 쓰인다.

---

## 9. Indexed Priority Queue — decrease-key가 필요할 때

다익스트라, A* 등에서 **키 감소**가 핵심. 항목에 **고유 id**를 두고, `pos[id]`로 **힙 내 위치 추적**.

```cpp
#include <vector>
#include <functional>
#include <stdexcept>

template <class Key, class Comp=std::less<Key>>
class IndexedPQ {
    // stores (id -> key), plus heap of ids, heap position for each id
    std::vector<Key> key_;
    std::vector<int> heap_;   // heap of ids
    std::vector<int> pos_;    // id -> heap index (or -1 if not in)
    Comp cmp;

    bool higher(int i, int j) const { // compare by key
        return cmp(key_[heap_[j]], key_[heap_[i]]);
    }
    void swap_heap(int i, int j){
        std::swap(heap_[i], heap_[j]);
        pos_[heap_[i]] = i; pos_[heap_[j]] = j;
    }
    void sift_up(int i){
        while(i>0){
            int p=(i-1)/2; if (!higher(i,p)) break; swap_heap(i,p); i=p;
        }
    }
    void sift_down(int i){
        int n=(int)heap_.size();
        while(true){
            int l=i*2+1, r=l+1, best=i;
            if (l<n && higher(l,best)) best=l;
            if (r<n && higher(r,best)) best=r;
            if (best==i) break; swap_heap(i,best); i=best;
        }
    }

public:
    explicit IndexedPQ(int n, const Comp& c=Comp())
        : key_(n), pos_(n, -1), cmp(c) {}

    bool contains(int id) const { return pos_[id]!=-1; }

    void push(int id, const Key& k){
        key_[id]=k; heap_.push_back(id); pos_[id]=(int)heap_.size()-1; sift_up(pos_[id]);
    }
    void decrease_key(int id, const Key& k){
        // assume k improves priority for min-heap(Comp=greater) or max-heap accordingly
        key_[id]=k; sift_up(pos_[id]); sift_down(pos_[id]); // 변경 방향 모르면 양쪽 호출(미세 최적화 여지)
    }
    int top_id() const{
        if (heap_.empty()) throw std::runtime_error("empty");
        return heap_[0];
    }
    const Key& top_key() const { return key_[top_id()]; }

    void pop(){
        if (heap_.empty()) throw std::runtime_error("empty");
        int last=heap_.back(); heap_.pop_back();
        if (!heap_.empty()){
            heap_[0]=last; pos_[last]=0; sift_down(0);
        }
        pos_[last] = (heap_.empty()? -1 : pos_[last]); // last가 0으로 내려간 경우 처리 위에서 완료
        // 실제로는 pop 대상 id의 pos를 -1로
        // 간단화를 위해 별도 반환/저장을 권장
        for (int i=0;i<(int)pos_.size();++i) if (pos_[i] >= (int)heap_.size()) pos_[i] = -1;
    }
};
```

> 참고: 위 `pop()` 마지막 루프는 간단화용. 실전에서는 **pop 대상 id를 추적**해 정확히 `pos[id]=-1`만 갱신하자.

- **장점**: `decrease-key` O(log n) 보장
- **응용**: **다익스트라**, **A\***, **이벤트 시뮬레이션**

---

## 10. 응용 레시피

### 10.1 Top-K(스트리밍) — 최소 힙 K 유지

```cpp
#include <queue>
#include <vector>

std::vector<int> topK(const std::vector<int>& a, int K){
    std::priority_queue<int, std::vector<int>, std::greater<int>> pq;
    for (int x: a){
        if ((int)pq.size() < K) pq.push(x);
        else if (x > pq.top()){ pq.pop(); pq.push(x); }
    }
    std::vector<int> out;
    while(!pq.empty()){ out.push_back(pq.top()); pq.pop(); }
    return out; // K개(큰 값들), 정렬은 안 되었음
}
```

### 10.2 K-way 병합 — 최소 힙

K개의 정렬 리스트를 한 번에 병합.

```cpp
#include <queue>
#include <vector>
struct Node { int val, i, j; }; // i: list index, j: pos
struct Cmp { bool operator()(const Node& a, const Node& b) const { return a.val > b.val; } };

std::vector<int> kmerge(const std::vector<std::vector<int>>& lists){
    std::priority_queue<Node, std::vector<Node>, Cmp> pq;
    for (int i=0;i<(int)lists.size();++i) if (!lists[i].empty()) pq.push({lists[i][0], i, 0});
    std::vector<int> out;
    while(!pq.empty()){
        auto [v,i,j]=pq.top(); pq.pop(); out.push_back(v);
        if (j+1<(int)lists[i].size()) pq.push({lists[i][j+1], i, j+1});
    }
    return out;
}
```

### 10.3 작업 스케줄링 — 우선순위 큐

도착 시간/우선순위/소요시간을 기준으로 **가장 중요한 작업**부터 처리.

```cpp
#include <queue>
#include <vector>
#include <tuple>

struct Task{ int t, pri, dur; }; // 도착t, 우선순위pri, 실행dur
struct Cmp { bool operator()(const Task& a, const Task& b) const {
    if (a.pri != b.pri) return a.pri < b.pri; // pri 큰게 먼저
    return a.dur > b.dur; // dur 짧은게 먼저
}};

long long simulate(std::vector<Task> tasks){
    std::sort(tasks.begin(), tasks.end(), [](auto& a, auto& b){ return a.t < b.t; });
    std::priority_queue<Task, std::vector<Task>, Cmp> pq;
    long long time=0; size_t i=0;
    while(i<tasks.size() || !pq.empty()){
        if (pq.empty() && time < tasks[i].t) time = tasks[i].t;
        while(i<tasks.size() && tasks[i].t <= time) pq.push(tasks[i++]);
        if (!pq.empty()){
            auto cur = pq.top(); pq.pop();
            time += cur.dur;
        }
    }
    return time; // 마지막 종료 시각
}
```

---

## 11. 복잡도 총정리 & 선택 가이드

### Deque

| 구현 | 장점 | 단점 | 추천 |
|---|---|---|---|
| `std::deque`(블록) | 양끝 O(1), 중간 삽입도 덜 비쌈 | 연속 메모리 아님 | 대부분의 범용 |
| 원형 배열(동적) | 캐시 친화·단순·빠름 | 재배치 비용 가끔 발생 | 순수 C++ 구현, 고성능 |
| 이중 리스트+센티넬 | 경계 단순·항상 O(1) | 캐시 비우호·할당 비용 | 빈번한 splice/접합 |

### Priority Queue

| 구조 | 연산 | 장점 | 단점 | 추천 |
|---|---|---|---|---|
| 이진 힙 | push/pop O(log n), top O(1), build O(n) | 구현 단순, 상수 낮음 | decrease-key 없음 | 일반적 전용 |
| d-ary 힙 | push/pop O((log n)/(log d)) | sift_down 단계 감소 | sift_up 비교↑ | 4-ary 실전 빈도 |
| Indexed PQ | dec-key O(log n) | 다익스트라 등 필수 | 구현 복잡 | 알고리즘 실전 |

---

## 12. 수학 코너

### 12.1 모노토닉 덱의 O(n)
각 인덱스는 덱에 **최대 한 번 push**, **최대 한 번 pop**:
$$
\sum_{i=1}^{n} (1_{\text{push}(i)} + 1_{\text{pop}(i)}) \le 2n \Rightarrow O(n)
$$

### 12.2 Heapify O(n)
레벨 $\ell$의 노드 수 $\approx n/2^{\ell+1}$, 높이 $h-\ell$ 만큼 하향 이동:
$$
\sum_{\ell=0}^{h} \frac{n}{2^{\ell+1}} \cdot O(h-\ell) \in O(n)
$$

---

## 13. 테스트/디버깅 체크리스트

- **Deque**: 원형 인덱스 오버플로(모듈로)/front-back 일관성/리사이즈 후 순서 보존
- **Heap**: `higher()` 비교자 논리(부호 반전 버그 잦음), build 후 힙 성질 유지
- **IndexedPQ**: `pos[id]` 갱신 누락 금지, 삭제 시 **정확히** `pos=-1`
- 퍼징: 레퍼런스(`std::deque`, `std::priority_queue`)와 랜덤 시나리오 비교

```cpp
// 간단 퍼저 예: BinaryHeap vs std::priority_queue
#include <queue>
#include <random>
#include <cassert>

int main(){
    BinaryHeap<int> bh; std::priority_queue<int> ref;
    std::mt19937 rng(123); std::uniform_int_distribution<int> op(0,2), val(0,100000);

    for(int t=0;t<100000;++t){
        int o=op(rng);
        if (o==0){ int x=val(rng); bh.push(x); ref.push(x); }
        else if (o==1 && !ref.empty()){ bh.pop(); ref.pop(); }
        else if (o==2 && !ref.empty()){ assert(bh.top()==ref.top()); }
    }
}
```

---

## 14. 정리

- **Deque**: 양끝 O(1) 입출력. **원형 배열/블록/이중 리스트** 중 요구에 맞게 선택. **모노토닉 덱/0–1 BFS** 등 강력한 알고리즘적 무기.
- **Priority Queue**: **이진 힙**이 표준. **heapify O(n)**, **d-ary/Indexed**로 확장 가능. 응용은 **Top-K, 병합, 스케줄링, 최단경로**까지 폭넓다.
- 도입 전 **데이터 크기/업데이트 패턴/동시성/캐시 특성**을 수치화하고, 본문 구현을 **퍼징+프로파일링**으로 검증하여 자신만의 실전 컨테이너로 다듬자.
