---
layout: post
title: Data Structure - 큐
date: 2024-12-10 19:20:23 +0900
category: Data Structure
---
# 큐(Queue)

## 큐란?

- **FIFO(First-In First-Out)**: 먼저 들어온 데이터가 먼저 나간다.
- 연산은 뒤에 **삽입(push/enqueue)**, 앞에서 **삭제(pop/dequeue)**.
- 대표 응용: **BFS**, **OS 스케줄링**, **비동기 이벤트 처리**, **파이프라인 버퍼**.

---

## 기본 연산과 복잡도

| 연산        | 의미                   | 평균 시간 |
|-------------|------------------------|-----------|
| `push(x)`   | 뒤에 삽입              | O(1)\*    |
| `pop()`     | 앞에서 제거            | O(1)\*    |
| `front()`   | 맨 앞 조회             | O(1)      |
| `back()`    | 맨 뒤 조회             | O(1)      |
| `empty()`   | 비었는지               | O(1)      |
| `size()`    | 원소 개수              | O(1)      |

\* **원형 배열**이나 **연결 리스트**, **deque** 기반일 때 O(1).
동적 원형 버퍼에서 **리사이즈 시** 개별 `push`는 O(n)이지만 **평균(Amortized) O(1)** 이다.

수식 스냅샷(용량을 두 배로 키운다면):
$$
\text{총 이동 비용} = 1+2+4+\dots+2^k \approx 2^{k+1}-1, \quad
\text{삽입 횟수} \approx 2^k \Rightarrow \text{평균} \approx 2 = O(1)
$$

---

## 배열 기반 큐(비원형) — 문제점 복습

초안에 제시된 간단 구현은 `pop`이 진행될수록 앞쪽 **빈 공간 낭비**가 누적된다.
실전에서는 **원형 인덱싱**(ring/circular)으로 해결한다.

---

## 원형 큐(Circular Queue) — 고정 용량

### 핵심 아이디어

- 고정 배열에서 `front`, `rear`를 **모듈로 연산**으로 순환.
- **가득 참/비어 있음** 구분 방법:
  1) `count`를 유지, 또는
  2) **한 칸 비움** 규칙(용량+1 배열), 또는
  3) `front == rear` 이면 비어있고, **가득 참**은 `(rear+1)%cap == front`.

### 구현(고정 용량, count 방식)

```cpp
#include <stdexcept>

class CircularQueue {
    int* data_;
    int  cap_;
    int  front_;
    int  rear_;
    int  cnt_;

public:
    explicit CircularQueue(int cap=8)
        : data_(new int[cap]), cap_(cap), front_(0), rear_(0), cnt_(0) {}

    ~CircularQueue(){ delete[] data_; }

    bool empty() const { return cnt_==0; }
    bool full()  const { return cnt_==cap_; }
    int  size()  const { return cnt_; }
    int  capacity() const { return cap_; }

    void push(int x){
        if (full()) throw std::overflow_error("Queue Overflow");
        data_[rear_] = x;
        rear_ = (rear_ + 1) % cap_;
        ++cnt_;
    }
    void pop(){
        if (empty()) throw std::underflow_error("Queue Underflow");
        front_ = (front_ + 1) % cap_;
        --cnt_;
    }
    int& front(){
        if (empty()) throw std::runtime_error("Queue is empty");
        return data_[front_];
    }
    const int& front() const{
        if (empty()) throw std::runtime_error("Queue is empty");
        return data_[front_];
    }
    int& back(){
        if (empty()) throw std::runtime_error("Queue is empty");
        int idx = (rear_ + cap_ - 1) % cap_;
        return data_[idx];
    }
};
```

---

## 동적 확장 원형 큐(템플릿) — 실전형 구현

- 고정 용량이 꽉 차면 **용량 2배**로 재할당 후 **연속 복사**.
- 제네릭 타입(T) 지원, 예외 안전 고려.

```cpp
#include <stdexcept>
#include <new>
#include <utility>

template <class T>
class RingQueue {
    T*   buf_ = nullptr;
    int  cap_ = 0;
    int  f_   = 0;   // front index
    int  r_   = 0;   // rear index (next write)
    int  cnt_ = 0;

    void grow_(){
        int ncap = cap_? cap_*2 : 4;
        T* nbuf = static_cast<T*>(::operator new[](sizeof(T)*ncap));
        // front..front+cnt-1 순서대로 이동
        try{
            for (int i=0;i<cnt_;++i){
                new (&nbuf[i]) T(std::move(buf_[(f_+i)%cap_]));
                buf_[(f_+i)%cap_].~T();
            }
        } catch(...){
            for (int i=0;i<cnt_;++i) nbuf[i].~T();
            ::operator delete[](nbuf);
            throw;
        }
        ::operator delete[](buf_);
        buf_ = nbuf;
        cap_ = ncap;
        f_ = 0;
        r_ = cnt_;
    }

public:
    RingQueue() = default;
    ~RingQueue(){
        for (int i=0;i<cnt_;++i) buf_[(f_+i)%cap_].~T();
        ::operator delete[](buf_);
    }

    bool empty() const { return cnt_==0; }
    int  size()  const { return cnt_; }
    int  capacity() const { return cap_; }

    void push(const T& x){
        if (cnt_==cap_) grow_();
        new (&buf_[r_]) T(x);
        r_ = cap_? (r_+1)%cap_ : 0;
        ++cnt_;
    }
    void push(T&& x){
        if (cnt_==cap_) grow_();
        new (&buf_[r_]) T(std::move(x));
        r_ = cap_? (r_+1)%cap_ : 0;
        ++cnt_;
    }
    void pop(){
        if (empty()) throw std::underflow_error("Queue Underflow");
        buf_[f_].~T();
        f_ = (f_+1)%cap_;
        --cnt_;
    }
    T& front(){
        if (empty()) throw std::runtime_error("Queue is empty");
        return buf_[f_];
    }
    const T& front() const{
        if (empty()) throw std::runtime_error("Queue is empty");
        return buf_[f_];
    }
    T& back(){
        if (empty()) throw std::runtime_error("Queue is empty");
        int idx = (r_+cap_-1)%cap_;
        return buf_[idx];
    }
};
```

**장점**: `std::queue`처럼 간편한 인터페이스 + **배열의 캐시 친화**와 **동적 확장**.
**복잡도**: `push` 평균 O(1), `pop` O(1), 리사이즈 시 O(n).

---

## 연결 리스트 기반 큐 — 템플릿 개선판

- 매 삽입마다 동적 할당 비용이 있지만 **크기 제한 없음**.

```cpp
#include <stdexcept>
#include <utility>

template <class T>
class ListQueue {
    struct Node {
        T data;
        Node* next;
        template<class U>
        explicit Node(U&& v) : data(std::forward<U>(v)), next(nullptr) {}
    };
    Node* head_ = nullptr;
    Node* tail_ = nullptr;
    int   cnt_  = 0;

public:
    ~ListQueue(){ while(!empty()) pop(); }

    bool empty() const { return cnt_==0; }
    int  size()  const { return cnt_; }

    void push(const T& x){
        Node* n = new Node(x);
        if (!tail_) head_=tail_=n;
        else { tail_->next = n; tail_ = n; }
        ++cnt_;
    }
    void push(T&& x){
        Node* n = new Node(std::move(x));
        if (!tail_) head_=tail_=n;
        else { tail_->next = n; tail_ = n; }
        ++cnt_;
    }
    void pop(){
        if (empty()) throw std::underflow_error("Queue Underflow");
        Node* t=head_;
        head_ = head_->next;
        if (!head_) tail_=nullptr;
        delete t; --cnt_;
    }
    T& front(){
        if (empty()) throw std::runtime_error("Queue is empty");
        return head_->data;
    }
    const T& front() const{
        if (empty()) throw std::runtime_error("Queue is empty");
        return head_->data;
    }
    T& back(){
        if (empty()) throw std::runtime_error("Queue is empty");
        return tail_->data;
    }
};
```

---

## 두 스택으로 큐 만들기 — 이론과 구현

**아이디어**: 입력 스택(S1), 출력 스택(S2).
`pop/front` 시 S2가 비어 있으면 **S1을 모두 S2로 옮김**(역순 유지). 이 작업은 요소당 **한 번만** 일어나므로 평균 O(1).

```cpp
#include <stack>
#include <stdexcept>

template <class T>
class TwoStackQueue {
    std::stack<T> in_, out_;
    void shift_(){
        if (out_.empty()){
            while(!in_.empty()){ out_.push(std::move(in_.top())); in_.pop(); }
        }
    }
public:
    bool empty() const { return in_.empty() && out_.empty(); }
    int  size()  const { return (int)in_.size() + (int)out_.size(); }

    void push(const T& x){ in_.push(x); }
    void push(T&& x){ in_.push(std::move(x)); }

    void pop(){
        shift_();
        if (out_.empty()) throw std::underflow_error("Queue Underflow");
        out_.pop();
    }
    T& front(){
        shift_();
        if (out_.empty()) throw std::runtime_error("Queue is empty");
        return out_.top();
    }
    const T& front() const{
        // const 버전은 구현 단순화를 위해 const_cast로 재사용해도 됨(주의).
        return const_cast<TwoStackQueue*>(this)->front();
    }
};
```

**암ortized 분석**(직관): 각 요소는 **in → out 이동**을 **최대 한 번**만 겪는다.
따라서 총 이동 비용은 O(n), 각 연산 평균 O(1).

---

## STL `std::queue` — 제대로 알기

```cpp
#include <queue>
#include <deque>
#include <vector>

std::queue<int> q1;                               // 기본: deque<int>
std::queue<int, std::deque<int>> q2;              // 명시적 deque
std::queue<int, std::vector<int>> q3;             // vector 기반(가능)
```

- `std::queue`는 **컨테이너 어댑터**: 내부 컨테이너의 `push_back/pop_front`(또는 양 끝 연산)를 감싼다.
- **반복자/범위 순회 없음**: 큐의 추상화 유지(앞/뒤만 접근).
- **대안**: 임의 접근/양끝 삽입이 필요하면 `std::deque` 사용.

---

## 모노토닉 큐(Monotonic Queue) — 슬라이딩 윈도우 최대

**문제**: 길이 `k` 윈도우에서 **각 윈도우의 최대값**을 O(n)에 구하라.
**해법**: 큐 안을 **내림차순**으로 유지(뒤에서 작은 값 제거). 앞의 인덱스가 윈도우에서 벗어나면 제거.

```cpp
#include <deque>
#include <vector>

std::vector<int> slidingWindowMax(const std::vector<int>& a, int k){
    std::deque<int> dq; // store indices, values decreasing
    std::vector<int> ans;
    for (int i=0;i<(int)a.size();++i){
        // 1) 뒤에서 a[i]보다 작은 값 제거
        while(!dq.empty() && a[dq.back()] <= a[i]) dq.pop_back();
        dq.push_back(i);
        // 2) 윈도우 밖 제거 (i-k+1이 시작)
        if (!dq.empty() && dq.front() <= i-k) dq.pop_front();
        // 3) 유효 윈도우 시작 이후부터 결과 수집
        if (i >= k-1) ans.push_back(a[dq.front()]);
    }
    return ans;
}
```

- **복잡도**: 각 인덱스가 **한 번 push, 한 번 pop** → O(n).
- **변형**: **모노토닉 증가 큐**로 **최솟값**도 동일하게 구한다.

---

## BFS/토폴로지/최단거리(무가중) — 큐의 대표 응용

### BFS

```cpp
#include <queue>
#include <vector>
#include <iostream>

void bfs(int s, const std::vector<std::vector<int>>& g){
    std::vector<int> vis(g.size());
    std::queue<int> q; q.push(s); vis[s]=1;
    while(!q.empty()){
        int u=q.front(); q.pop();
        std::cout<<u<<" ";
        for(int v: g[u]) if(!vis[v]){ vis[v]=1; q.push(v); }
    }
}
```

### 위상 정렬(Kahn)

```cpp
#include <queue>
#include <vector>

std::vector<int> topoSort(int n, const std::vector<std::vector<int>>& g){
    std::vector<int> indeg(n,0);
    for(int u=0;u<n;++u) for(int v: g[u]) ++indeg[v];

    std::queue<int> q;
    for(int i=0;i<n;++i) if(indeg[i]==0) q.push(i);

    std::vector<int> order;
    while(!q.empty()){
        int u=q.front(); q.pop();
        order.push_back(u);
        for(int v: g[u]) if(--indeg[v]==0) q.push(v);
    }
    return order; // DAG 아닐 경우 모든 노드가 나오지 않음
}
```

### 최단거리(무가중 그래프)

```cpp
#include <queue>
#include <vector>
#include <limits>

std::vector<int> shortest_unweighted(int n, const std::vector<std::vector<int>>& g, int s){
    const int INF = std::numeric_limits<int>::max()/4;
    std::vector<int> dist(n, INF);
    std::queue<int> q; q.push(s); dist[s]=0;
    while(!q.empty()){
        int u=q.front(); q.pop();
        for(int v: g[u]) if(dist[v]==INF){
            dist[v]=dist[u]+1;
            q.push(v);
        }
    }
    return dist;
}
```

---

## 블로킹 큐(Blocking Queue) — 생산자–소비자

멀티스레드 환경에서 **한 쪽은 push**, **다른 쪽은 pop**을 수행.
빈 큐에서 `pop`은 **대기**, 가득 찬 큐에서 `push`는 **대기**.

```cpp
#include <mutex>
#include <condition_variable>
#include <queue>

template <class T>
class BlockingQueue {
    std::mutex m_;
    std::condition_variable cv_not_empty_, cv_not_full_;
    std::queue<T> q_;
    std::size_t cap_;

public:
    explicit BlockingQueue(std::size_t cap) : cap_(cap) {}

    void push(T x){
        std::unique_lock<std::mutex> lk(m_);
        cv_not_full_.wait(lk, [&]{ return q_.size()<cap_; });
        q_.push(std::move(x));
        lk.unlock();
        cv_not_empty_.notify_one();
    }
    T pop(){
        std::unique_lock<std::mutex> lk(m_);
        cv_not_empty_.wait(lk, [&]{ return !q_.empty(); });
        T v = std::move(q_.front()); q_.pop();
        lk.unlock();
        cv_not_full_.notify_one();
        return v;
    }
    bool try_pop(T& out){
        std::lock_guard<std::mutex> g(m_);
        if (q_.empty()) return false;
        out = std::move(q_.front()); q_.pop();
        cv_not_full_.notify_one();
        return true;
    }
};
```

- **주의**: 스풀 종료/취소를 위해 **폐쇄 플래그/notify_all** 추가 설계가 필요할 수 있다.

---

## (스케치) SPSC 링 버퍼 — 한 생산자·한 소비자 무락 패턴

- **Single-Producer Single-Consumer**에서는 원형 버퍼와 `std::atomic<size_t>` 인덱스로 **락 없이** 구현 가능.
- 여기선 안전 스케치만 제시(메모리 순서 보장 핵심: `memory_order_release/acquire`).

```cpp
#include <atomic>
#include <vector>
#include <optional>

template <class T>
class SPSCQueue {
    std::vector<T> buf_;
    const size_t cap_;
    std::atomic<size_t> head_{0}; // 소비 인덱스
    std::atomic<size_t> tail_{0}; // 생산 인덱스
public:
    explicit SPSCQueue(size_t cap) : buf_(cap), cap_(cap) {}

    bool push(const T& x){
        size_t t = tail_.load(std::memory_order_relaxed);
        size_t h = head_.load(std::memory_order_acquire);
        if ((t+1)%cap_ == h) return false; // full
        buf_[t] = x;
        tail_.store((t+1)%cap_, std::memory_order_release);
        return true;
    }
    bool pop(T& out){
        size_t h = head_.load(std::memory_order_relaxed);
        size_t t = tail_.load(std::memory_order_acquire);
        if (h == t) return false; // empty
        out = std::move(buf_[h]);
        head_.store((h+1)%cap_, std::memory_order_release);
        return true;
    }
};
```

> **주의**: MPMC(다중 생산자·다중 소비자) 무락 큐는 훨씬 복잡(ABA, 메모리 순서, 캐시 라인 패딩 등).

---

## 구현 비교 & 선택 가이드

| 구현             | 장점                                  | 단점/주의                         | 권장 시나리오 |
|------------------|---------------------------------------|-----------------------------------|---------------|
| 원형 배열(고정)  | O(1) 연산, 캐시 친화, 단순            | 용량 제한, 오버플로 체크 필요     | 임베디드, 고정 크기 버퍼 |
| 원형 배열(동적)  | 평균 O(1), 실용적, 템플릿 가능        | 리사이즈 복사 비용(가끔)          | 범용 |
| 연결 리스트      | 크기 제한 없음, `push/pop` O(1)       | 할당 비용, 캐시 비우호            | 크기 큰/변동 심한 큐 |
| 두 스택 큐       | 평균 O(1), 구현 단순                  | 최악 시 이동 비용, 디버깅 주의     | 알고리즘 연습/면접 |
| 블로킹 큐        | 스레드 간 안전한 통신                 | 잠금·대기, 종료 플래그 설계 필요   | 생산자–소비자 |
| SPSC 링 버퍼     | 무락, 극저지연                        | SPSC로 제한, 메모리 순서 주의     | RT/게임/드라이버 |

---

## 흔한 함정(필수 체크)

1. **가득 참/빈 상태 혼동**: 원형 큐에서 `front==rear` 해석 통일.
2. **모듈로 비용**: 아주 고성능이 필요하면 `cap`을 **2의 거듭제곱**으로 두고 `(idx & (cap-1))`로 최적화.
3. **예외 안전**: `front/back` 호출 전 **empty 검사**.
4. **리사이즈 시 순서 보존**: 동적 원형 큐는 **front부터 순서대로** 새 버퍼에 복사.
5. **멀티스레드**: 단일 큐에 다중 스레드 접근 시 **락/원자성** 필요. SPSC/MPMC 제약 구분.
6. **`std::queue` 오해**: 반복자 없음(큐 추상화 유지). 순회가 필요하면 `deque` 쓰기.

---

## 추가 응용 미니 레시피

### 고정 윈도우 평균(스트리밍)

```cpp
#include <queue>

class MovingAverage {
    std::queue<int> q; long long sum=0; int k;
public:
    explicit MovingAverage(int k): k(k) {}
    double put(int x){
        q.push(x); sum += x;
        if ((int)q.size() > k){ sum -= q.front(); q.pop(); }
        return (double)sum / q.size();
    }
};
```

### 라운드 로빈 스케줄러(간단 시뮬)

```cpp
#include <queue>
#include <string>
#include <utility>
#include <iostream>

void round_robin(std::queue<std::pair<std::string,int>> q, int quantum){
    int time=0;
    while(!q.empty()){
        auto [name, rem]=q.front(); q.pop();
        int run = std::min(quantum, rem);
        time += run; rem -= run;
        std::cout<<name<<" runs "<<run<<" (t="<<time<<")\n";
        if (rem>0) q.push({name, rem});
    }
}
```

---

## 테스트 전략(간단 퍼저)

- **불변식**: `size==cnt`, 원소 순서 유지, `empty ⇒ size==0`.
- **랜덤 시나리오**: `push/pop/front` 혼합 후 결과를 **레퍼런스 구현(std::queue)** 과 비교.

```cpp
#include <queue>
#include <random>
#include <cassert>

int main(){
    RingQueue<int> rq; std::queue<int> ref;
    std::mt19937 rng(123); std::uniform_int_distribution<int> op(0,2), val(0,1000);

    for (int t=0;t<100000;++t){
        int o = op(rng);
        if (o==0){ int x=val(rng); rq.push(x); ref.push(x); }
        else if (o==1 && !ref.empty()){ rq.pop(); ref.pop(); }
        else if (o==2 && !ref.empty()){ assert(rq.front()==ref.front()); }
        assert((int)ref.size()==rq.size());
    }
}
```

---

## 수학 코너 — 두 스택 큐의 암ortized O(1)

각 원소는 **삽입 시 S1에 push 1회**, **어느 순간 S2로 이동 시 push+pop 각 1회**, **삭제 시 S2에서 pop 1회**.
총 3~4회의 상수 연산을 **한 번씩만** 겪으므로, 연산 수 대비 평균 비용은 **상수**:

$$
\text{총 이동 연산} \in \Theta(n) \ \Rightarrow\ \text{평균 비용} \in \Theta(1)
$$

---

## 요약

- 큐는 **FIFO**로, 구현 선택에 따라 **캐시/확장/스레드/지연** 특성이 크게 다르다.
- **원형 배열(동적)**은 범용, **연결 리스트**는 유연, **두 스택 큐**는 면접/알고리즘,
  **모노토닉 큐**는 **슬라이딩 윈도우 최적화**, **블로킹 큐**는 **스레딩**, **SPSC 링**은 **극저지연**에 적합.
- 도입 전 **용량/성능/동시성** 요구를 수치화하고, 본문 레시피를 **테스트+프로파일링**으로 검증하자.
