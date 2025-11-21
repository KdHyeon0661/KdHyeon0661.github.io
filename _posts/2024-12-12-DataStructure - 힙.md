---
layout: post
title: Data Structure - 힙
date: 2024-12-12 19:20:23 +0900
category: Data Structure
---
# — 우선순위를 다루는 강력한 트리 구조

기존 초안의 핵심(정의·종류·배열 인덱싱·간단 MaxHeap/MinHeap·`priority_queue`·힙정렬·응용)을 유지하면서 다음을 대폭 확장한다.

- **정확한 배열 인덱싱 규칙**과 **완전 이진 트리** 조건
- **템플릿 이진 힙**(최대/최소/커스텀 비교자), **`build-heap` O(n)** 정식 구현
- **STL 힙 알고리즘(`make_heap/push_heap/pop_heap/sort_heap`)**과 in-place 정렬
- **d-ary 힙**(4-ary 추천)의 설계 포인트
- **Indexed Priority Queue**(decrease-key 지원) — 다익스트라 실전형
- **스트리밍 Top-K/중간값(running median)**, **다익스트라**, **이벤트 스케줄링** 예제
- **암ortized/합산 분석**과 자주 생기는 버그/테스트 전략

---

## 힙이란?

> **힙**은 **완전 이진 트리(Complete Binary Tree)** 기반의 자료구조로, **부모와 자식 간 우선순위 관계**(힙 성질)를 만족한다.

- **최대 힙(Max-Heap)**: 부모 ≥ 자식 → `top()`이 최댓값
- **최소 힙(Min-Heap)**: 부모 ≤ 자식 → `top()`이 최솟값

예시(최대 힙):

```
      50
     /  \
   30    20
  / \   /
10 15  5
```

---

## 힙 vs 우선순위 큐

| 항목 | 힙(Heap) | 우선순위 큐(Priority Queue) |
|---|---|---|
| 개념 | 우선순위 관계를 만족하는 **트리 구조** | 우선순위 높은 원소부터 꺼내는 **추상 자료형** |
| 구현 | 보통 **완전 이진 트리**를 **배열**로 | 내부 구현으로 힙을 주로 사용 |
| 연산 | `push/pop/top` 중심, `heapify` 제공 | `push/pop/top` 인터페이스 제공 |
| C++ 표준 | `std::make_heap` 등 **알고리즘** 제공 | `std::priority_queue` **컨테이너 어댑터** 제공 |

> 정리: **힙은 구현체**, **우선순위 큐는 개념/인터페이스**. 일반적으로 우선순위 큐는 힙으로 구현된다.

---

## 배열 인덱스 규칙(0-based)

완전 이진 트리는 **왼쪽부터 빈틈없이** 채워진다.

- 왼쪽 자식: `L(i) = 2*i + 1`
- 오른쪽 자식: `R(i) = 2*i + 2`
- 부모: `P(i) = (i - 1) / 2`

---

## 이진 힙의 연산과 복잡도

| 연산 | 설명 | 시간 |
|---|---|---|
| `push(x)` | 말단에 추가 후 **상향 힙화(sift-up)** | \(O(\log n)\) |
| `pop()` | 루트 제거 → 말단을 루트로 옮긴 뒤 **하향 힙화(sift-down)** | \(O(\log n)\) |
| `top()` | 루트 조회(최대/최소) | \(O(1)\) |
| `build-heap` | 무작위 배열을 힙으로 변환 | \(O(n)\) |

### 왜 `build-heap`이 \(O(n)\)인가?

아래쪽 레벨의 노드가 훨씬 많아서 **총 이동 거리의 합**이 선형으로 수렴한다.

$$
\sum_{\ell=0}^{h} \frac{n}{2^{\ell+1}} \cdot O(h-\ell) \in O(n)
$$

---

## 템플릿 이진 힙 구현(최대/최소/커스텀 비교자)

### 핵심 구현

```cpp
```cpp
#include <vector>
#include <functional>
#include <stdexcept>
#include <utility>

template <class T, class Comp = std::less<T>> // less => max-heap
class BinaryHeap {
    std::vector<T> a; // 0-based array
    Comp cmp;         // cmp(x,y)==true → x<y (기본: less)

    // "higher"는 힙의 루트에 가까울수록 '우선'이라는 뜻.
    bool higher(int i, int j) const {
        // max-heap: a[i]가 a[j]보다 '더 큰'가? → cmp(a[j],a[i])
        return cmp(a[j], a[i]);
    }

    void sift_up(int i){
        while(i > 0){
            int p = (i - 1) / 2;
            if (!higher(i, p)) break;
            std::swap(a[i], a[p]);
            i = p;
        }
    }

    void sift_down(int i){
        int n = (int)a.size();
        while(true){
            int l = 2*i + 1, r = l + 1, best = i;
            if (l < n && higher(l, best)) best = l;
            if (r < n && higher(r, best)) best = r;
            if (best == i) break;
            std::swap(a[i], a[best]);
            i = best;
        }
    }

public:
    BinaryHeap() = default;
    explicit BinaryHeap(const Comp& c): cmp(c) {}

    bool empty() const { return a.empty(); }
    int  size () const { return (int)a.size(); }

    const T& top() const {
        if (empty()) throw std::runtime_error("heap empty");
        return a[0];
    }

    void push(const T& x){ a.push_back(x); sift_up((int)a.size()-1); }
    void push(T&& x)     { a.push_back(std::move(x)); sift_up((int)a.size()-1); }

    void pop(){
        if (empty()) throw std::runtime_error("heap empty");
        a[0] = std::move(a.back()); a.pop_back();
        if (!a.empty()) sift_down(0);
    }

    template <class It>
    void build(It first, It last){
        a.assign(first, last);
        for (int i=(int)a.size()/2 - 1; i>=0; --i) sift_down(i); // O(n)
    }
};
```
```

- **최대 힙**: `Comp = std::less<T>` (기본)
- **최소 힙**: `Comp = std::greater<T>`

### 구조체와 커스텀 비교자

```cpp
```cpp
struct Job { int id; int pri; int dur; }; // 우선순위 pri, tie-breaker dur

struct JobCmp {
    bool operator()(const Job& a, const Job& b) const {
        if (a.pri != b.pri) return a.pri < b.pri;   // pri 큰게 '우선' (max-heap)
        return a.dur > b.dur;                        // dur 짧은게 '우선'
    }
};

// BinaryHeap<Job, JobCmp> heap;
```
```

---

## STL 힙 알고리즘 & `priority_queue`

### 힙 알고리즘(컨테이너는 vector 등 임의 접근 필요)

```cpp
```cpp
#include <algorithm>
#include <vector>

// v가 힙이 아니면 힙으로 만든다 (O(n))
std::vector<int> v {3,1,5,2,4};
std::make_heap(v.begin(), v.end());     // 최대 힙

// 원소 추가: push_back 후 push_heap (O(log n))
v.push_back(6);
std::push_heap(v.begin(), v.end());

// pop: 루트와 끝 교환 + pop_back + heapify-down (O(log n))
std::pop_heap(v.begin(), v.end()); // 루트가 맨 뒤로 이동
v.pop_back();

// 정렬: 힙을 이용해 in-place 내림차순(기본) 정렬 (O(n log n))
std::sort_heap(v.begin(), v.end());
```
```

- 최소 힙을 원한다면 **역비교자**를 쓰거나 원소에 부호를 바꾸는 방식.

### `std::priority_queue`

```cpp
```cpp
#include <queue>
#include <vector>
#include <functional>

std::priority_queue<int> maxpq; // 최대 힙(default: less)
std::priority_queue<int, std::vector<int>, std::greater<int>> minpq; // 최소 힙
```
```

- `priority_queue`는 반복자가 없고 **top/pop/push**만 제공 → **우선순위 큐 추상화**에 충실

---

## — 팬아웃으로 상수 줄이기

- 이진 힙의 자식 2개 → **d개**로 확장
- **장점**: 트리 높이 감소 → `sift_down` 비교 단계 수 감소
- **단점**: `sift_up`에서 부모 검색/비교 비용은 증가
- 인덱스(0-based):
  - 부모: \(\text{par}(i) = \left\lfloor\frac{i-1}{d}\right\rfloor\)
  - k번째 자식: \(\text{child}(i,k) = d\cdot i + 1 + k, \ 0 \le k < d\)
- 실전에서는 **4-ary**가 캐시/분기 예측 관점에서 대체로 좋은 타협점

> 대용량/고성능 스케줄러·네트워킹 큐에서 4-ary 힙을 선택하는 사례가 많다.

---

## Indexed Priority Queue — decrease-key가 필요할 때

**다익스트라/A\***에서 핵심은 **키 감소(decrease-key)**. 항목별 **id**를 고정하고, `pos[id]`가 **힙 내 위치**를 추적한다.

```cpp
```cpp
#include <vector>
#include <functional>
#include <stdexcept>
#include <utility>

template <class Key, class Comp = std::less<Key>> // less => min-heap을 원하면 Comp=greater
class IndexedPQ {
    // min-heap를 예로 설명 (Comp=greater<Key>)
    std::vector<Key> key_;   // id -> key
    std::vector<int> heap_;  // heap of ids
    std::vector<int> pos_;   // id -> heap index ( -1 if not present )
    Comp cmp;                // heap order comparator on 'Key'

    bool higher(int i, int j) const { // compare by key
        // heap_[i], heap_[j]는 id. key_로 비교.
        return cmp(key_[heap_[j]], key_[heap_[i]]);
    }
    void swap_heap(int i, int j){
        std::swap(heap_[i], heap_[j]);
        pos_[heap_[i]] = i; pos_[heap_[j]] = j;
    }
    void sift_up(int i){
        while(i>0){
            int p=(i-1)/2;
            if (!higher(i,p)) break;
            swap_heap(i,p); i=p;
        }
    }
    void sift_down(int i){
        int n=(int)heap_.size();
        while(true){
            int l=2*i+1, r=l+1, best=i;
            if (l<n && higher(l,best)) best=l;
            if (r<n && higher(r,best)) best=r;
            if (best==i) break;
            swap_heap(i,best); i=best;
        }
    }

public:
    explicit IndexedPQ(int n, const Comp& c = Comp())
        : key_(n), pos_(n, -1), cmp(c) {}

    bool contains(int id) const { return pos_[id] != -1; }
    int  size() const { return (int)heap_.size(); }
    bool empty() const { return heap_.empty(); }

    void push(int id, const Key& k){
        key_[id] = k;
        heap_.push_back(id);
        pos_[id] = (int)heap_.size()-1;
        sift_up(pos_[id]);
    }

    int top_id() const {
        if (empty()) throw std::runtime_error("empty");
        return heap_[0];
    }
    const Key& top_key() const { return key_[top_id()]; }

    void pop(){
        if (empty()) throw std::runtime_error("empty");
        int root = heap_[0];
        int last = heap_.back();
        heap_.pop_back();
        pos_[root] = -1;
        if (!heap_.empty()){
            heap_[0] = last; pos_[last] = 0; sift_down(0);
        }
    }

    // 키를 더 '좋은' 값으로 갱신 (min-heap에선 더 작은 값)
    void decrease_key(int id, const Key& k){
        key_[id] = k;
        int i = pos_[id];
        // 변경 방향을 모르면 sift_up/sift_down 모두 호출 가능(미세 최적화 여지)
        sift_up(i); sift_down(i);
    }
};
```
```

- **장점**: `decrease-key`가 \(O(\log n)\) 보장 → **다익스트라/스케줄링**에서 표준
- `priority_queue`는 decrease-key가 없어 **lazy deletion**(타임스탬프 부여 후 오래된 항목 무시)로 우회한다.

---

## — 불안정, In-place \(O(n \log n)\)

### `std::make_heap`/`sort_heap` 활용(in-place)

```cpp
```cpp
#include <vector>
#include <algorithm>

void heap_sort(std::vector<int>& v){
    std::make_heap(v.begin(), v.end()); // 최대 힙
    std::sort_heap(v.begin(), v.end()); // 오름차순 결과
}
```
```

### 직접 MaxHeap 객체로 정렬(단계적 pop)

```cpp
```cpp
void heap_sort2(std::vector<int>& arr){
    BinaryHeap<int> hp;
    hp.build(arr.begin(), arr.end());      // O(n)
    for (int i=(int)arr.size()-1; i>=0; --i){
        arr[i] = hp.top();
        hp.pop();
    }
}
```
```

> **불안정**: 같은 값의 상대 순서가 보존되지 않는다(안정 정렬이 아님).

---

## 실전 응용 레시피

### — 최소 힙 크기 K

```cpp
```cpp
#include <queue>
#include <vector>
#include <functional>

std::vector<int> topK(const std::vector<int>& a, int K){
    std::priority_queue<int, std::vector<int>, std::greater<int>> pq;
    for (int x: a){
        if ((int)pq.size() < K) pq.push(x);
        else if (x > pq.top()){ pq.pop(); pq.push(x); }
    }
    std::vector<int> out;
    while(!pq.empty()){ out.push_back(pq.top()); pq.pop(); }
    return out; // 내림차순 정렬은 아님
}
```
```

### — 두 힙

- **max-heap**(왼쪽 절반), **min-heap**(오른쪽 절반) 유지
- 균형을 맞춰 두 힙 사이 크기 차가 1 이하가 되도록

```cpp
```cpp
#include <queue>
#include <vector>
#include <functional>

class RunningMedian {
    std::priority_queue<int> leftMax; // lower half
    std::priority_queue<int, std::vector<int>, std::greater<int>> rightMin; // upper half

public:
    void add(int x){
        if (leftMax.empty() || x <= leftMax.top()) leftMax.push(x);
        else rightMin.push(x);

        // rebalance
        if ((int)leftMax.size() > (int)rightMin.size() + 1){
            rightMin.push(leftMax.top()); leftMax.pop();
        } else if ((int)rightMin.size() > (int)leftMax.size()){
            leftMax.push(rightMin.top()); rightMin.pop();
        }
    }
    double median() const {
        if (leftMax.size() == rightMin.size()){
            if (leftMax.empty()) return 0.0;
            return (leftMax.top() + rightMin.top()) / 2.0;
        }
        return leftMax.top();
    }
};
```
```

### 다익스트라(Indexed PQ 버전, 인접 리스트)

```cpp
```cpp
#include <vector>
#include <limits>

struct Edge { int to; int w; };

std::vector<int> dijkstra_indexedPQ(int n, const std::vector<std::vector<Edge>>& g, int s){
    const int INF = std::numeric_limits<int>::max()/4;
    std::vector<int> dist(n, INF);
    // min-heap: Comp=greater<int> → smaller key is 'higher'
    IndexedPQ<int, std::greater<int>> pq(n);
    dist[s] = 0; pq.push(s, 0);

    while(!pq.empty()){
        int u = pq.top_id();
        int du = pq.top_key();
        pq.pop();
        if (du != dist[u]) continue; // stale guard
        for (auto [v, w] : g[u]){
            if (du + w < dist[v]){
                dist[v] = du + w;
                if (pq.contains(v)) pq.decrease_key(v, dist[v]);
                else pq.push(v, dist[v]);
            }
        }
    }
    return dist;
}
```
```

### 작업 스케줄링(우선순위·소요시간 기준)

```cpp
```cpp
#include <queue>
#include <vector>
#include <algorithm>

struct Task { int arrival, pri, dur; };
struct Cmp {
    bool operator()(const Task& a, const Task& b) const {
        if (a.pri != b.pri) return a.pri < b.pri;  // pri 큰게 먼저
        return a.dur > b.dur;                      // dur 짧은게 먼저
    }
};

long long simulate(std::vector<Task> tasks){
    std::sort(tasks.begin(), tasks.end(), [](auto& a, auto& b){ return a.arrival < b.arrival; });
    std::priority_queue<Task, std::vector<Task>, Cmp> pq;
    long long t=0; size_t i=0;
    while(i<tasks.size() || !pq.empty()){
        if (pq.empty() && t < tasks[i].arrival) t = tasks[i].arrival;
        while(i<tasks.size() && tasks[i].arrival <= t) pq.push(tasks[i++]);
        if (!pq.empty()){
            auto cur = pq.top(); pq.pop();
            t += cur.dur;
        }
    }
    return t;
}
```
```

---

## 수학/복잡도 스냅샷

### `build-heap`의 \(O(n)\)

레벨 \(\ell\)의 노드 수는 \(\approx n/2^{\ell+1}\), 각 노드는 최대 \(O(h-\ell)\)만큼 내려간다:

$$
\sum_{\ell=0}^{h} \frac{n}{2^{\ell+1}} (h-\ell) \in O(n)
$$

### 두 힙 중간값의 균형

좌우 힙 크기 차를 \(\le 1\)로 유지하면, 삽입마다 재배치는 상수 번 비교/이동으로 처리되어 **평균 \(O(\log n)\)** 로 안정적으로 유지된다.

---

## 자주 나오는 버그 & 체크리스트

1. **비교자 반전**: `less`/`greater` 의미를 혼동 → `higher()` 구현을 **명시적**으로 점검.
2. **sift_down 종료 조건**: 자식 인덱스 경계 체크, `best==i`일 때 종료.
3. **`build-heap` 루프 시작점**: **마지막 내부 노드** `n/2 - 1`부터 역순.
4. **`priority_queue` decrease-key 부재**: 갱신 시 **lazy deletion** or **Indexed PQ** 사용.
5. **중간값 두 힙 균형**: 삽입 후 **크기 차 1 이하**로 맞추기.
6. **정렬 안정성 오해**: 힙 정렬은 **불안정**.
7. **경계/예외**: `top/pop` 호출 전 **empty** 확인, 테스트에서 **랜덤 퍼징** 권장.

---

## — 힙 vs `priority_queue`

```cpp
```cpp
#include <queue>
#include <random>
#include <cassert>

int main(){
    BinaryHeap<int> bh;
    std::priority_queue<int> ref;
    std::mt19937 rng(123);
    std::uniform_int_distribution<int> op(0,2), val(0,100000);

    for (int t=0; t<100000; ++t){
        int o = op(rng);
        if (o==0){ int x = val(rng); bh.push(x); ref.push(x); }
        else if (o==1 && !ref.empty()){ bh.pop(); ref.pop(); }
        else if (o==2 && !ref.empty()){ assert(bh.top() == ref.top()); }
    }
}
```
```

---

## 확장: 페어링 힙/피보나치 힙(개요)

- **페어링 힙(Pairing Heap)**: 단순 링크 기반, 실제 성능 우수, decrease-key가 **상대적으로 빠름**(분석은 복잡).
- **피보나치 힙(Fibonacci Heap)**: 이론상 `decrease-key`가 \(O(1)\) amortized, `pop`이 \(O(\log n)\).
  상수항이 커 실전에서는 **Indexed 이진 힙**이 더 빠른 경우 많음.

실무에서 **데이터 크기/업데이트 패턴/상수 비용**을 프로파일링해 선택한다.

---

## 요약

- **힙**은 완전 이진 트리 기반으로 `push/pop/top`에 강하며, `build-heap`이 \(O(n)\)이라 초기화도 빠르다.
- **템플릿 이진 힙**으로 최대/최소/커스텀 우선순위를 일관되게 지원하고,
  **Indexed Priority Queue**로 **decrease-key**가 필요한 그래프/스케줄링 문제를 견고하게 해결한다.
- **힙정렬**은 in-place \(O(n\log n)\)지만 **불안정**.
- 응용은 **Top-K**, **중간값**, **다익스트라**, **이벤트 스케줄링** 등 폭넓다.
- 도입 전 **요구조건(성능/갱신/메모리)**을 수치화하고, 본문 코드로 **퍼징+프로파일링**하여 현장에 맞게 다듬자.
