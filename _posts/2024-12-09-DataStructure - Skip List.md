---
layout: post
title: Data Structure - Skip List
date: 2024-12-09 22:20:23 +0900
category: Data Structure
---
# Skip List (스킵 리스트)

## Skip List란?

**Skip List**는 정렬된 단일 연결 리스트에 **다층 레벨**을 덧대어, 상위 레벨일수록 **멀리 점프**할 수 있게 만든 자료구조다.
무작위 높이(레벨)를 부여하여 **평균적으로 균형**을 유지하므로, 탐색/삽입/삭제가 모두 평균 $$O(\log n)$$에 동작한다.

- 평균 동작: **BST(균형 트리)** 수준의 $$O(\log n)$$
- 구현 복잡도: **회전·재배치가 필요한 트리**보다 단순
- 추가 장점: 정렬 유지 + 양방향(수준별) 순회가 자연스러움

---

## 핵심 개념 정리

- 각 노드는 `forward[level]` 포인터들을 가진다. `level`이 클수록 더 긴 도약을 담당.
- 노드의 높이는 확률 $$P$$(보통 1/2)로 **동전 던지기**하며 정한다.
- `MAX_LEVEL`을 상한으로 둔다(보통 $$\lceil \log_{1/P}(N_{\max}) \rceil$$ 근처).

**평균 성능 직관**
각 레벨에 등장하는 노드 수가 기하급수적으로 줄어 **탐색 경로 길이**가 로그가 된다.

---

## 수학적 스냅샷(확률 분석 스케치)

- 각 노드가 레벨 $$\ell$$ 이상일 확률은 $$P^\ell$$.
- 레벨 $$\ell$$의 기대 노드 수는 $$n P^\ell$$.
- 최상위 레벨은 대략 $$\log_{1/P} n$$ 개.
  탐색 시 레벨마다 평균 **상수**개의 전진 후 한 레벨 하강 →
  $$\Theta(\log_{1/P} n) = \Theta(\log n)$$ 기대 시간.

> 자세한 기대값/분산 표준 증명은 본문 말미에 스케치를 덧붙인다.

---

## 기본 구조 (템플릿 + 정렬 Key/Value)

- 비교 가능한 키 `Key`, 값 `T`(맵 형태) 혹은 `void`(셋 형태)를 지원하도록 **템플릿 일반화**.
- 간단성을 위해 **원시 포인터**를 사용하되, **명확한 소멸자**에서 해제한다.
- 예외 안전은 삽입/삭제 도중 노드 연결 순서를 주의해서 제공.

```cpp
// skip_list.hpp
#pragma once
#include <vector>
#include <functional>
#include <random>
#include <limits>
#include <iostream>
#include <cassert>

template <class Key, class T = void, class Compare = std::less<Key>>
class skip_list {
public:
    static constexpr int DEFAULT_MAX_LEVEL = 32; // log2(4e9) 정도, 넉넉
    using mapped_type   = T;
    using key_type      = Key;
    using value_type    = std::conditional_t<std::is_void<T>::value, Key, std::pair<const Key, T>>;
    using size_type     = std::size_t;

private:
    struct node {
        value_type      val;
        std::vector<node*> forward; // forward[0] = next in base level

        // set: value_type=Key, map: pair<const Key,T>
        template <class... Args>
        node(int level, Args&&... args)
            : val(std::forward<Args>(args)...), forward(level+1, nullptr) {}
    };

    Compare comp_;
    float   prob_;
    int     max_level_;
    int     level_;     // 현재 skip list의 최고 레벨
    node*   head_;
    size_type size_;

    // RNG
    std::mt19937_64 rng_;
    std::uniform_real_distribution<float> dist_;

public:
    explicit skip_list(float p = 0.5f, int max_level = DEFAULT_MAX_LEVEL,
                       Compare cmp = Compare(),
                       uint64_t seed = std::random_device{}())
        : comp_(cmp), prob_(p), max_level_(max_level), level_(0),
          size_(0), rng_(seed), dist_(0.0f, 1.0f)
    {
        head_ = new node(max_level_, dummy_init_tag{}); // 헤더 노드: 값 무의미
    }

    ~skip_list() { clear(); delete head_; }

    skip_list(const skip_list&) = delete;
    skip_list& operator=(const skip_list&) = delete;

    // 기본 연산
    bool empty() const noexcept { return size_ == 0; }
    size_type size() const noexcept { return size_; }

    // 삽입/탐색/삭제/경계 탐색
    template <class... Args>
    bool insert(Args&&... args);      // set: insert(key), map: insert(key, value)
    bool contains(const key_type& k) const;
    node* find_node(const key_type& k) const;
    bool erase(const key_type& k);

    // lower_bound / upper_bound
    node* lower_bound_node(const key_type& k) const;
    node* upper_bound_node(const key_type& k) const;

    void clear() noexcept;

    // 디버그 출력
    void print_levels(std::ostream& os = std::cout) const;

    // 반복자(최하위 레벨 선형 순회)
    class iterator {
        node* cur_;
    public:
        explicit iterator(node* p=nullptr): cur_(p) {}
        value_type& operator*() const { return cur_->val; }
        value_type* operator->() const { return &cur_->val; }
        iterator& operator++(){ cur_ = cur_->forward[0]; return *this; }
        bool operator==(const iterator& o) const { return cur_==o.cur_; }
        bool operator!=(const iterator& o) const { return cur_!=o.cur_; }
        friend class skip_list;
    };
    iterator begin() const { return iterator(head_->forward[0]); }
    iterator end()   const { return iterator(nullptr); }

private:
    // 유틸
    struct dummy_init_tag{}; // 헤더 val 초기화용 태그
    template <class K>
    static const K& key_of(const K& k) { return k; }

    template <class K, class V>
    static const K& key_of(const std::pair<const K, V>& kv) { return kv.first; }

    int random_level() {
        int lv = 0;
        while (lv < max_level_ && dist_(rng_) < prob_) ++lv;
        return lv;
    }

    // key 비교: comp_(a,b)==true 이면 a<b
    bool key_less(const key_type& a, const key_type& b) const { return comp_(a,b); }
    bool key_equal(const key_type& a, const key_type& b) const { return !comp_(a,b) && !comp_(b,a); }
};
```

> 구현 포인트
> - `value_type`을 **셋(Key)** 혹은 **맵(pair<const Key,T>)**으로 일반화.
> - 헤더 `head_`는 `forward`만 가지는 더미 노드(값 무의미).
> - 반복자는 최하위 레벨(0)만 선형 순회한다.

---

## 삽입/탐색/삭제/경계탐색 구현

```cpp
// skip_list_impl.hpp
#pragma once
#include "skip_list.hpp"

template <class Key, class T, class Compare>
template <class... Args>
bool skip_list<Key,T,Compare>::insert(Args&&... args) {
    // 1) 경로(업데이트 포인터) 수집
    std::vector<node*> update(max_level_+1, nullptr);
    node* cur = head_;
    key_type k_candidate;

    // 임시 노드 생성 없이 키만 뽑아내기
    if constexpr (std::is_void<T>::value) {
        // set: Args... = (key)
        const key_type& k = std::get<0>(std::forward_as_tuple(args...));
        k_candidate = k;
    } else {
        // map: Args... = (key, value) 또는 (pair)
        if constexpr (sizeof...(Args) == 1 &&
                      std::is_same_v<std::tuple_element_t<0, std::tuple<Args...>>, value_type>) {
            const value_type& kv = std::get<0>(std::forward_as_tuple(args...));
            k_candidate = kv.first;
        } else {
            const key_type& k = std::get<0>(std::forward_as_tuple(args...));
            k_candidate = k;
        }
    }

    for (int i = level_; i >= 0; --i) {
        while (cur->forward[i]) {
            const key_type& nxtk = key_of(cur->forward[i]->val);
            if (key_less(nxtk, k_candidate)) cur = cur->forward[i];
            else break;
        }
        update[i] = cur;
    }

    cur = cur->forward[0];
    if (cur && key_equal(key_of(cur->val), k_candidate)) {
        // 이미 존재: set은 중복 금지, map이라면 overwrite 원하면 별도 정책
        return false;
    }

    // 2) 새 노드 생성 + 레벨 반영
    int newLevel = random_level();
    if (newLevel > level_) {
        for (int i = level_ + 1; i <= newLevel; ++i) update[i] = head_;
        level_ = newLevel;
    }

    node* nn = nullptr;
    if constexpr (std::is_void<T>::value) {
        nn = new node(newLevel, k_candidate);
    } else {
        if constexpr (sizeof...(Args) == 1 &&
                      std::is_same_v<std::tuple_element_t<0, std::tuple<Args...>>, value_type>) {
            const value_type& kv = std::get<0>(std::forward_as_tuple(args...));
            nn = new node(newLevel, kv);
        } else {
            // (key, val) 형태
            const key_type& k   = std::get<0>(std::forward_as_tuple(args...));
            const T&        val = std::get<1>(std::forward_as_tuple(args...));
            nn = new node(newLevel, std::piecewise_construct,
                          std::forward_as_tuple(k),
                          std::forward_as_tuple(val));
        }
    }

    // 3) 포인터 연결(하위→상위 순서로 안전하게)
    for (int i = 0; i <= newLevel; ++i) {
        nn->forward[i] = update[i]->forward[i];
        update[i]->forward[i] = nn;
    }
    ++size_;
    return true;
}

template <class Key, class T, class Compare>
bool skip_list<Key,T,Compare>::contains(const key_type& k) const {
    return find_node(k) != nullptr;
}

template <class Key, class T, class Compare>
typename skip_list<Key,T,Compare>::node*
skip_list<Key,T,Compare>::find_node(const key_type& k) const {
    const node* cur = head_;
    for (int i = level_; i >= 0; --i) {
        while (cur->forward[i]) {
            const key_type& nxtk = key_of(cur->forward[i]->val);
            if (comp_(nxtk, k)) cur = cur->forward[i];
            else break;
        }
    }
    cur = cur->forward[0];
    if (cur && !comp_(k, key_of(cur->val)) && !comp_(key_of(cur->val), k))
        return const_cast<node*>(cur);
    return nullptr;
}

template <class Key, class T, class Compare>
bool skip_list<Key,T,Compare>::erase(const key_type& k) {
    std::vector<node*> update(max_level_+1, nullptr);
    node* cur = head_;
    for (int i = level_; i >= 0; --i) {
        while (cur->forward[i]) {
            const key_type& nxtk = key_of(cur->forward[i]->val);
            if (comp_(nxtk, k)) cur = cur->forward[i];
            else break;
        }
        update[i] = cur;
    }
    cur = cur->forward[0];
    if (!cur || !key_equal(key_of(cur->val), k)) return false;

    // 연결 해제(상위→하위 모두)
    for (int i = 0; i <= level_; ++i) {
        if (update[i]->forward[i] == cur) update[i]->forward[i] = cur->forward[i];
    }
    delete cur; --size_;

    // 필요시 레벨 감소
    while (level_ > 0 && head_->forward[level_] == nullptr) --level_;
    return true;
}

template <class Key, class T, class Compare>
typename skip_list<Key,T,Compare>::node*
skip_list<Key,T,Compare>::lower_bound_node(const key_type& k) const {
    const node* cur = head_;
    for (int i = level_; i >= 0; --i) {
        while (cur->forward[i]) {
            const key_type& nxtk = key_of(cur->forward[i]->val);
            if (comp_(nxtk, k)) cur = cur->forward[i];
            else break;
        }
    }
    cur = cur->forward[0]; // 첫 번째 not less than k
    return const_cast<node*>(cur);
}

template <class Key, class T, class Compare>
typename skip_list<Key,T,Compare>::node*
skip_list<Key,T,Compare>::upper_bound_node(const key_type& k) const {
    const node* cur = head_;
    for (int i = level_; i >= 0; --i) {
        while (cur->forward[i]) {
            const key_type& nxtk = key_of(cur->forward[i]->val);
            // strictly greater: nxtk <= k 면 앞으로
            if (!comp_(k, nxtk)) cur = cur->forward[i];
            else break;
        }
    }
    cur = cur->forward[0]; // 첫 번째 greater than k
    return const_cast<node*>(cur);
}

template <class Key, class T, class Compare>
void skip_list<Key,T,Compare>::clear() noexcept {
    node* cur = head_->forward[0];
    while (cur) {
        node* nxt = cur->forward[0];
        delete cur;
        cur = nxt;
    }
    for (int i=0;i<=level_;++i) head_->forward[i] = nullptr;
    size_ = 0;
    level_ = 0;
}

template <class Key, class T, class Compare>
void skip_list<Key,T,Compare>::print_levels(std::ostream& os) const {
    for (int i = level_; i >= 0; --i) {
        os << "Level " << i << ": ";
        const node* cur = head_->forward[i];
        while (cur) {
            os << key_of(cur->val) << " ";
            cur = cur->forward[i];
        }
        os << "\n";
    }
}
```

**주의 포인트**

- **업데이트 배열**(`update[i]`)로 각 레벨에서 새 노드가 들어갈 앞 노드를 기억해 둔다.
- 삽입 시 **하위 레벨부터 연결**하면 예외가 터질 경우 상위 레벨 구조가 망가질 수 있다. 보통은 코드 전체가 예외를 던질 부분이 거의 없지만, 확장 시 생성자 예외 등을 고려해 **연결 순서/생명주기**에 유의한다.
- 삭제 후 상위 레벨이 비면 `level_`을 낮춰 불필요한 레벨을 제거한다.

---

## 사용 예제(셋/맵)

```cpp
// main_set.cpp  (셋 형태: T=void)
#include "skip_list_impl.hpp"
#include <iostream>

int main(){
    skip_list<int> s; // set<int>
    s.insert(10); s.insert(1); s.insert(7); s.insert(20); s.insert(15);

    std::cout << "contains 7? " << s.contains(7) << "\n";
    s.print_levels();

    auto lb = s.lower_bound_node(8);
    if (lb) std::cout << "lower_bound(8)=" << (*lb).val << "\n";

    s.erase(10);
    s.print_levels();

    // 순회
    for (auto it=s.begin(); it!=s.end(); ++it) {
        std::cout << *it << " ";
    }
    std::cout << "\n";
}
```

```cpp
// main_map.cpp  (맵 형태: T=std::string)
#include "skip_list_impl.hpp"
#include <iostream>

int main(){
    skip_list<int, std::string> mp; // map<int,string>
    mp.insert(2, "two");
    mp.insert(10, "ten");
    mp.insert(5, "five");

    if (auto nd = mp.find_node(5)) {
        std::cout << "5 -> " << nd->val.second << "\n";
    }
    auto ub = mp.upper_bound_node(5);
    if (ub) std::cout << "upper_bound(5)=" << ub->val.first << "\n";

    mp.print_levels();
    mp.erase(2);
    mp.print_levels();

    for (auto it=mp.begin(); it!=mp.end(); ++it)
        std::cout << it->first << ":" << it->second << " ";
    std::cout << "\n";
}
```

---

## 삭제 알고리즘 상세(예외/코너케이스)

- 경로(`update[]`)를 찾은 뒤, 목표 노드 `x`가 존재하면 **모든 레벨에서 `x`를 건너뛰도록** 포인터를 갱신.
- 그 다음 `x`를 `delete`.
- 상위 레벨이 비면 `level_`을 감소시켜 레벨 수를 유지.

**코너**

- 존재하지 않는 키 삭제: `false` 반환.
- 동일 키 삽입: 정책에 따라 실패 처리(위 코드: **중복 삽입 거부**).

---

## 경계 탐색(`lower_bound`/`upper_bound`)와 범위 질의

- **`lower_bound(k)`**: $$\ge k$$인 첫 노드
- **`upper_bound(k)`**: $$> k$$인 첫 노드

범위 질의(예: $$[L,R)$$)는

```cpp
for (auto nd = s.lower_bound_node(L);
     nd && comp(key_of(nd->val), R)==false && comp(R, key_of(nd->val))==false;
     nd = nd->forward[0]) {
    // nd->val 사용
}
```

처럼 최하위 레벨에서 선형 순회하면 된다.

---

## 확장 1: 인덱스 가능한 Skip List(순서통계)

일부 구현은 각 레벨의 간격 길이( `span[i]` )를 보관하여

- **k번째 원소 접근** $$O(\log n)$$
- **순위(rank) 계산** $$O(\log n)$$

을 지원한다. 아이디어:

- `forward[i]`와 함께 그 링크가 **몇 개의 레벨-0 노드**를 건너뛰는지 `span[i]` 유지.
- 삽입/삭제 시 각 레벨에서 span 갱신.

간단 스케치:

```cpp
struct idx_node {
    value_type val;
    std::vector<idx_node*> forward;
    std::vector<int> span; // forward[i]로 건너뛰는 (level-0) 노드 수
    template <class... Args>
    idx_node(int lv, Args&&... args)
      : val(std::forward<Args>(args)...), forward(lv+1,nullptr), span(lv+1,1) {}
};
```

> 본문 완전 코드는 길어 생략. 필요 시 별도 포스트에서 구현/증명/테스트를 다룬다.

---

## 확장 2: 동시성(참고 개요)

- 다중 스레드 환경에서는 **락 기반** 레벨별 락 또는 **락-프리(skip list-based concurrent set)** 기법을 사용한다.
- ABA 문제/메모리 회수(Hazard pointers, epoch) 등을 다뤄야 하므로 범위가 크다.
- 실무에서는 **검증된 동시 컨테이너**(라이브러리) 사용 추천.

---

## 메모리·파라미터 선택

- **P (승격 확률)**: 0.5가 보편적.
  $$E[\text{level}] \approx \log_{1/P}(n)$$,
  너무 작으면(=P↑) 레벨이 커져 오버헤드, 너무 크면(=P↓) 레벨이 낮아 탐색 비용 증가.
- **MAX_LEVEL**: 최대 원소 수의 로그를 넘어 넉넉히. 32~40이면 수억까지 충분.
- **공간 오버헤드**: 각 노드는 평균적으로 $$\sum_{\ell\ge 0} P^\ell = \frac{1}{1-P}$$ 개의 forward 포인터를 기대.
  예: P=0.5 → 평균 2개.

---

## 테스트: 퍼징/정확성 검증

### 간단 퍼징(셋 vs `std::set` 비교)

```cpp
// test_fuzz.cpp
#include "skip_list_impl.hpp"
#include <set>
#include <random>
#include <cassert>

int main(){
    skip_list<int> s;
    std::set<int>   g;
    std::mt19937 rng(123);
    std::uniform_int_distribution<int> op(0,2), val(0,10000);

    for(int t=0;t<200000;++t){
        int o=op(rng), x=val(rng);
        if(o==0){
            bool si=s.insert(x);
            bool gi=g.insert(x).second;
            assert(si==gi);
        }else if(o==1){
            bool se=s.erase(x);
            bool ge = (g.erase(x)>0);
            assert(se==ge);
        }else{
            bool sc=s.contains(x);
            bool gc=(g.find(x)!=g.end());
            assert(sc==gc);
        }
    }
    // 순회 일치성
    auto it1=s.begin(); auto it2=g.begin();
    for(; it1!=s.end() && it2!=g.end(); ++it1, ++it2) {
        if constexpr (std::is_void<typename decltype(s)::mapped_type>::value) {
            assert(*it1 == *it2);
        }
    }
    assert(it1==s.end() && it2==g.end());
}
```

---

## 성능 팁

- 비교 함수는 **빠르고 순수**해야 한다.
- 대량 삽입 전 **시드 고정**으로 재현성 있는 벤치 수행.
- `P=0.5`가 대체로 무난하지만 **워크로드**에 따라 `P∈[0.25, 0.75]` 범위 탐색.

---

## 복잡도 요약

| 연산 | 평균 시간 | 최악 시간(희박) | 공간 |
|---|---|---|---|
| `search/contains` | $$O(\log n)$$ | $$O(n)$$ | $$O(n/(1-P))$$ |
| `insert` | $$O(\log n)$$ | $$O(n)$$ | 추가 포인터 평균 $$1/(1-P)$$ |
| `erase` | $$O(\log n)$$ | $$O(n)$$ | - |
| `lower_bound/upper_bound` | $$O(\log n)$$ | $$O(n)$$ | - |

> 최악은 모든 노드가 레벨 0일 때 등 **확률적으로 매우 드묾**.

---

## 수학 스케치(조금 더)

- 레벨 $$\ell$$에 있는 노드 수의 기대값은 $$nP^\ell$$.
- 최상위 비공허 레벨 $$L$$은 $$nP^L \approx 1 \Rightarrow L \approx \log_{1/P}n$$.
- 탐색 중 **레벨당 전진 단계 수**의 기대값은 상수(자기유사성) → 전체 기대 경로 길이 $$\Theta(\log n)$$.

---

## 참고 구현 선택지

- **전통 구현**: `std::vector<node*> forward`. 단순·명확.
- **성능 지향**: 고정 최대 레벨 배열(소형 최적화) + 커스텀 `allocator`(풀/아레나).
- **인덱스형**: `span[]` 유지하여 `select(k)/rank(k)` 지원.

---

## 전체 빌드

```bash
g++ -std=c++17 -O2 -Wall -Wextra main_set.cpp   -o set_demo
g++ -std=c++17 -O2 -Wall -Wextra main_map.cpp   -o map_demo
g++ -std=c++17 -O2 -Wall -Wextra test_fuzz.cpp  -o fuzz && ./fuzz
```

---

## 마무리

이 글은 초안(간단 구조/삽입/탐색)을 **완성형 스킵 리스트**로 끌어올려,
- **삭제/경계 탐색/반복자**
- **예외와 연결 순서**
- **확률 분석/파라미터 선택**
- **퍼징으로 정확성 검증**
- **인덱스형 변종 개요**
까지 실전에서 필요한 요소를 모두 채웠다.

다음 단계로는 **인덱스 가능 스킵 리스트**를 별도 포스트에서 풀 구현하며,
`select(k)`/`rank(x)`/`range_count(L,R)`까지 확장해 보자.
