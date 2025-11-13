---
layout: post
title: Data Structure - 해시 테이블
date: 2024-12-24 19:20:23 +0900
category: Data Structure
---
# 해시 테이블(Hash Table)

## 1. 개요

해시 테이블은 **키 → 인덱스**로 사상하는 **해시 함수(hash function)** 와 **버킷 배열(bucket array)** 을 이용해, 평균적으로 **삽입/탐색/삭제가 O(1)** 인 빠른 연관 자료구조다.
핵심은 다음 두 가지다.

- **좋은 해시 함수**: 키를 **균일하게** 흩뿌린다.
- **충돌 처리 전략**: 같은 버킷에 여러 키가 오는 **충돌(collision)** 을 효율적으로 해결한다.

이 글은 기존 초안의 뼈대를 유지하되, 다음을 대폭 보강한다.

- 로드 팩터(α), 리해시(rehash), 기대 연산 비용의 수식
- 개방 주소법(선형/이차/이중 해싱), Robin HoodHash, Cuckoo 개요
- 체이닝/개방 주소법 **직접 C++ 구현**(리사이즈 포함)
- `std::unordered_map`/**투명 해시(equal/hash is_transparent)**, 예외/반복자 안정성, 보안 해시

---

## 2. 해시 함수의 요구 조건

해시 함수 \( h: \mathcal{K} \to \{0,\dots,m-1\} \) 는 다음을 만족해야 한다.

- **균일성(uniformity)**: 키가 버킷에 **고르게** 분포
- **결정성(determinism)**: 같은 키 → 항상 같은 값
- **속도(speed)**: 매우 빠른 산술(분기·메모리 접근 적음)
- **혼합성(mixing)**: 비슷한 키라도 결과가 충분히 달라짐

정수 키의 간단 혼합(64-bit):

```cpp
// SplitMix64: 빠르고 균일한 비암호 해시(시드와 함께 쓰면 더 좋다)
static inline uint64_t splitmix64(uint64_t x) {
    x += 0x9e3779b97f4a7c15ull;
    x = (x ^ (x >> 30)) * 0xbf58476d1ce4e5b9ull;
    x = (x ^ (x >> 27)) * 0x94d049bb133111ebull;
    return x ^ (x >> 31);
}
```

문자열 해시(대표적 예):
- **FNV-1a**, **MurmurHash**(3/64-bit), **xxHash**: 빠른 비암호 해시
- **SipHash**: 충돌 유도 공격 방어용(보안 강화)

C++ 표준 `std::hash`는 구현마다 다르다. **사용자 정의 타입**은 직접 해시를 제공하자.

```cpp
struct Point { int x, y; };

struct PointHash {
    size_t operator()(const Point& p) const noexcept {
        // 64-bit 혼합을 2개 좌표에 적용 후 결합
        uint64_t hx = splitmix64(static_cast<uint64_t>(static_cast<uint32_t>(p.x)));
        uint64_t hy = splitmix64(static_cast<uint64_t>(static_cast<uint32_t>(p.y)));
        // hash_combine 유사 기법
        return static_cast<size_t>(hx ^ (hy + 0x9e3779b97f4a7c15ull + (hx<<6) + (hx>>2)));
    }
};
struct PointEq {
    bool operator()(const Point& a, const Point& b) const noexcept {
        return a.x==b.x && a.y==b.y;
    }
};
```

---

## 3. 충돌과 해결 전략

충돌은 **피할 수 없다**. 해결은 크게 **체이닝(Chaining)** 과 **개방 주소법(Open Addressing)** 으로 나뉜다.

### 3.1 체이닝(Chaining)

각 버킷이 키-값을 **연결 리스트(혹은 동적 배열)** 로 보관.

- 구현 간단, 삭제 쉬움, 로드 팩터(α) > 1 허용
- 단점: 포인터 추적 → 캐시 비우호, 메모리 오버헤드

기대 비용(균등분포 가정):
- 성공 탐색: \( \Theta(1 + \alpha/2) \)
- 실패 탐색: \( \Theta(1 + \alpha) \)
- 여기서 \( \alpha = \frac{n}{m} \) (원소 수 n, 버킷 수 m)

### 3.2 개방 주소법(Open Addressing)

빈 슬롯을 **탐사(probing)** 로 찾아 **배열 내부에** 저장.

- 장점: 포인터 없음 → 캐시 친화
- 단점: **로드 팩터**가 1 미만이어야 하고 삭제가 까다롭다(무덤표시 Tombstone 필요).

종류:

| 탐사 | 규칙 | 특징 |
|---|---|---|
| 선형(Linear) | \( i_k = (h + k) \bmod m \) | 구현 간단, **1차 클러스터링** 발생 |
| 이차(Quadratic) | \( i_k = (h + c_1 k + c_2 k^2) \bmod m \) | 1차 클러스터링 완화, 매개변수 주의 |
| 이중 해싱(Double) | \( i_k = (h_1 + k \cdot h_2) \bmod m \) | 분산 효과 가장 큼, \(h_2\)와 m의 서로소 조건 |

선형 탐사의 기대 비용(평균, 무작위 해시 가정, 로드 팩터 \( \alpha \)):
- **성공 탐색**: \( \frac{1}{2}\left(1 + \frac{1}{1-\alpha}\right) \)
- **실패 탐색**: \( \frac{1}{2}\left(1 + \frac{1}{(1-\alpha)^2}\right) \)

\( \alpha \) 가 0.75에 근접하면 급격히 악화 → **0.5~0.75** 유지 권고.

#### Robin Hood Hashing (개방 주소법의 개량)

- 삽입 시 **탐사 거리**(현재 위치 - 이상적 위치)가 **더 큰** 원소가 **버킷을 '빼앗아'** 차지.
- 클러스터 길이의 분산이 줄어 **최악 탐사 길이**가 감소하는 경향.

#### Cuckoo Hashing (개요)

- 두 개 해시 함수 \( h_1, h_2 \). 삽입 시 둘 중 하나에 넣고, 자리가 차면 기존 원소를 **쫓아내 다른 해시에 재삽입**.
- 매우 빠른 조회(최대 2회 조회)이나, **재배치 루프**와 **재해시** 필요.

---

## 4. 리사이즈(리해시)와 로드 팩터

- **로드 팩터** \( \alpha = \frac{n}{m} \).
- 체이닝: α 제한이 느슨, 개방 주소법: α<1 필수, 보통 0.5~0.75 관리.
- **리해시(rehash)**: 임계치 초과 시 **버킷 수 증가**(보통 2배) 후 **모든 원소 재배치**. 비용 \( O(n) \)이지만 **상수분할 상환**으로 평균 O(1) 유지.

---

## 5. 체이닝 기반 C++ 해시 테이블 (동적 리사이즈 포함)

기존 초안(고정 크기)에서 **로드 팩터 관리 + 리해시 + 일반화 템플릿** 으로 확장.

```cpp
#include <vector>
#include <forward_list>
#include <optional>
#include <functional>
#include <stdexcept>
#include <utility>
#include <string>
#include <cassert>

template<class K, class V,
         class Hash = std::hash<K>,
         class Eq   = std::equal_to<K>>
class HashTableChaining {
    using Bucket = std::forward_list<std::pair<K,V>>;
    std::vector<Bucket> table;
    size_t n = 0;                 // 원소 수
    float  max_alpha = 0.75f;     // 임계 로드 팩터
    Hash   hasher;
    Eq     eq;

    size_t idx_for(K const& k, size_t mod) const {
        return hasher(k) % mod;
    }
    void maybe_rehash() {
        if (load_factor() > max_alpha) rehash(table.size() * 2);
    }
public:
    explicit HashTableChaining(size_t buckets = 8,
                               float max_load_factor = 0.75f,
                               Hash h = Hash{}, Eq e = Eq{})
        : table(std::max<size_t>(buckets, 1)), max_alpha(max_load_factor),
          hasher(h), eq(e) {}

    bool insert_or_assign(K key, V value) {
        maybe_rehash();
        size_t m = table.size();
        size_t i = idx_for(key, m);
        for (auto& kv : table[i]) {
            if (eq(kv.first, key)) { kv.second = std::move(value); return false; }
        }
        table[i].push_front({std::move(key), std::move(value)});
        ++n;
        return true;
    }

    std::optional<V> find(K const& key) const {
        size_t m = table.size();
        size_t i = idx_for(key, m);
        for (auto const& kv : table[i])
            if (eq(kv.first, key)) return kv.second;
        return std::nullopt;
    }

    bool erase(K const& key) {
        size_t i = idx_for(key, table.size());
        auto& lst = table[i];
        auto prev = lst.before_begin();
        for (auto it = lst.begin(); it != lst.end(); ++it) {
            if (eq(it->first, key)) {
                lst.erase_after(prev); --n; return true;
            }
            prev = it;
        }
        return false;
    }

    void rehash(size_t new_buckets) {
        std::vector<Bucket> newtab(std::max<size_t>(new_buckets, 1));
        for (auto& bucket : table) {
            for (auto& kv : bucket) {
                size_t i = idx_for(kv.first, newtab.size());
                newtab[i].push_front(std::move(kv));
            }
        }
        table.swap(newtab);
    }

    void reserve(size_t expected_size) {
        size_t target = table.size();
        while ((float)expected_size / (float)target > max_alpha) target *= 2;
        if (target != table.size()) rehash(target);
    }

    float load_factor() const { return (float)n / (float)table.size(); }
    size_t size() const { return n; }
    size_t bucket_count() const { return table.size(); }
};
```

**포인트**
- `forward_list` 로 체이닝 → 삽입/삭제 O(1)
- `reserve`/`rehash` 로 **상수분할 상환 O(1)** 보장
- 해시/동등 비교자 템플릿으로 커스터마이즈

사용 예:

```cpp
#include <iostream>

int main() {
    HashTableChaining<int, std::string> ht(4);
    ht.insert_or_assign(1, "A");
    ht.insert_or_assign(5, "E"); // 충돌
    ht.insert_or_assign(9, "I"); // 충돌
    ht.insert_or_assign(1, "A*"); // update

    std::cout << "size=" << ht.size()
              << " buckets=" << ht.bucket_count()
              << " load=" << ht.load_factor() << "\n";

    if (auto v = ht.find(5)) std::cout << *v << "\n";
    ht.erase(9);
    std::cout << (ht.find(9).has_value() ? "found" : "not found") << "\n";
}
```

---

## 6. 개방 주소법 C++ 구현 — 선형 탐사 + Tombstone + 리사이즈

```cpp
#include <vector>
#include <optional>
#include <string>
#include <functional>

template<class K, class V,
         class Hash = std::hash<K>,
         class Eq   = std::equal_to<K>>
class LinearProbingHT {
    enum class State : uint8_t { Empty, Occupied, Deleted };
    struct Slot {
        K key; V val; State st = State::Empty;
    };

    std::vector<Slot> tab;
    size_t n = 0;        // Occupied 개수
    size_t tomb = 0;     // Deleted 개수
    float  max_alpha = 0.5f;
    Hash   hasher; Eq eq;

    size_t mask() const { return tab.size() - 1; } // 크기 2^k일 때 용이
    void maybe_rehash() {
        if ((float)(n + tomb) / (float)tab.size() > max_alpha)
            rehash(tab.size() * 2);
    }
    size_t find_slot(K const& key) const {
        size_t m = tab.size(), i = hasher(key) & (m - 1);
        while (true) {
            if (tab[i].st == State::Empty) return i; // 빈 자리 → 최초 삽입 위치/실패 종료
            if (tab[i].st == State::Occupied && eq(tab[i].key, key)) return i;
            i = (i + 1) & (m - 1);
        }
    }
public:
    explicit LinearProbingHT(size_t cap = 8, float max_load = 0.5f,
                             Hash h = Hash{}, Eq e = Eq{})
        : max_alpha(max_load), hasher(h), eq(e)
    {
        // 용이한 마스크 연산을 위해 2의 거듭제곱으로 맞춘다.
        size_t m = 1; while (m < cap) m <<= 1;
        tab.resize(m);
    }

    bool insert_or_assign(K key, V value) {
        maybe_rehash();
        size_t m = tab.size();
        size_t i = hasher(key) & (m - 1);
        size_t first_del = m; // 최초 tombstone 위치
        while (true) {
            if (tab[i].st == State::Empty) {
                size_t pos = (first_del < m) ? first_del : i;
                if (tab[pos].st == State::Deleted) --tomb;
                tab[pos].key = std::move(key);
                tab[pos].val = std::move(value);
                tab[pos].st  = State::Occupied;
                ++n;
                return true;
            }
            if (tab[i].st == State::Deleted) {
                if (first_del == m) first_del = i; // tombstone 재사용 후보
            } else if (eq(tab[i].key, key)) {
                tab[i].val = std::move(value); // update
                return false;
            }
            i = (i + 1) & (m - 1);
        }
    }

    std::optional<V> find(K const& key) const {
        size_t m = tab.size();
        size_t i = hasher(key) & (m - 1);
        while (true) {
            if (tab[i].st == State::Empty) return std::nullopt; // 탐색 중단
            if (tab[i].st == State::Occupied && eq(tab[i].key, key))
                return tab[i].val;
            i = (i + 1) & (m - 1);
        }
    }

    bool erase(K const& key) {
        size_t m = tab.size();
        size_t i = hasher(key) & (m - 1);
        while (true) {
            if (tab[i].st == State::Empty) return false;
            if (tab[i].st == State::Occupied && eq(tab[i].key, key)) {
                tab[i].st = State::Deleted; --n; ++tomb;
                return true;
            }
            i = (i + 1) & (m - 1);
        }
    }

    void rehash(size_t newcap) {
        size_t m = 1; while (m < newcap) m <<= 1;
        std::vector<Slot> old = std::move(tab);
        tab.clear(); tab.resize(m);
        size_t oldn = n; n = tomb = 0;
        for (auto& s : old) if (s.st == State::Occupied)
            insert_or_assign(std::move(s.key), std::move(s.val));
        (void)oldn;
    }

    size_t size() const { return n; }
    size_t capacity() const { return tab.size(); }
    float load_factor() const { return (float)(n + tomb) / (float)tab.size(); }
};
```

**포인트**
- 테이블 크기를 **2의 거듭제곱** → `%` 대신 `& (m-1)` 로 빠르게
- **Tombstone** 로 삭제 처리(탐색 연속성 보장)
- \( \alpha \) 는 **(n + tomb) / m** 기준으로 관리

---

## 7. Robin Hood Hashing (핵심 로직 미니 구현)

```cpp
#include <vector>
#include <optional>
#include <functional>
#include <utility>
#include <cstdint>

template<class K, class V, class Hash=std::hash<K>, class Eq=std::equal_to<K>>
class RobinHoodHT {
    struct Slot { K key; V val; bool used=false; bool tomb=false; };
    std::vector<Slot> tab;
    size_t n=0, tomb=0;
    Hash H; Eq E;

    size_t mask() const { return tab.size()-1; }
    static size_t dist(size_t from, size_t to, size_t m){
        return (to + m - from) & (m-1);
    }
    void rehash(size_t cap){
        size_t m=1; while(m<cap) m<<=1;
        auto old = std::move(tab);
        tab.assign(m, {});
        n=tomb=0;
        for (auto& s: old) if (s.used && !s.tomb) insert(std::move(s.key), std::move(s.val));
    }
    void maybe_rehash(){
        if ((n + tomb) * 1.0f / tab.size() > 0.7f) rehash(tab.size()*2);
    }
public:
    explicit RobinHoodHT(size_t cap=8){ size_t m=1; while(m<cap) m<<=1; tab.resize(m); }

    bool insert(K key, V val){
        maybe_rehash();
        size_t m=tab.size();
        size_t i = H(key) & (m-1);
        size_t d = 0;
        for(;; i=(i+1)&(m-1), ++d){
            if(!tab[i].used || tab[i].tomb){
                tab[i].key=std::move(key); tab[i].val=std::move(val);
                tab[i].used=true; tab[i].tomb=false; ++n; if(tab[i].tomb) --tomb;
                return true;
            }
            if (E(tab[i].key, key)){ tab[i].val=std::move(val); return false; }

            size_t di = dist(H(tab[i].key)&(m-1), i, m);
            if (di < d) { // Robin Hood swap
                std::swap(key, tab[i].key);
                std::swap(val, tab[i].val);
                d = di;
            }
        }
    }

    std::optional<V> find(K const& key) const{
        size_t m=tab.size();
        size_t i = H(key)&(m-1);
        size_t d = 0;
        for(;; i=(i+1)&(m-1), ++d){
            if(!tab[i].used) return std::nullopt; // 빈 칸 → 미존재
            size_t di = dist(H(tab[i].key)&(m-1), i, m);
            if (di < d) return std::nullopt;       // 조기 중단
            if (!tab[i].tomb && E(tab[i].key,key)) return tab[i].val;
        }
    }

    bool erase(K const& key){
        size_t m=tab.size();
        size_t i = H(key)&(m-1);
        size_t d = 0;
        for(;; i=(i+1)&(m-1), ++d){
            if(!tab[i].used) return false;
            size_t di = dist(H(tab[i].key)&(m-1), i, m);
            if (di < d) return false;
            if (!tab[i].tomb && E(tab[i].key,key)) { tab[i].tomb=true; --n; ++tomb; return true; }
        }
    }
};
```

**핵심**: 더 멀리 온(=탐사 거리가 큰) 원소가 **우선권**을 가진다 → 긴 클러스터의 **최악 탐사 길이** 감소.

---

## 8. `std::unordered_map` / `unordered_set` 실전 팁

- **체이닝 기반**(노드 기반 컨테이너) — 대부분 구현에서 **참조/포인터는 rehash 후에도 유효**(표준 규정)
- **반복자(iterator)** 는 **`rehash/reserve` 시 모두 무효화**
- 기본 API:
  - `max_load_factor`, `load_factor`, `rehash`, `reserve`, `bucket_count`
  - `find`, `contains(C++20)`, `insert`, `insert_or_assign(C++17)`, `try_emplace(C++17)`
- 대량 삽입 전 **`reserve(n)`** 로 리해시 비용을 줄여라.
- **사용자 정의 타입**은 **hash/equal** 를 함께 제공.
- **투명 해시/동등자**(C++20): 다른 키 타입으로도 조회 가능.

```cpp
#include <unordered_map>
#include <string>
#include <string_view>

struct TransparentHash {
    using is_transparent = void;
    size_t operator()(std::string_view sv) const noexcept {
        // 매우 단순: 데모용(실전은 더 강한 해시 권장)
        size_t h=1469598103934665603ull;
        for (unsigned char c : sv) { h ^= c; h *= 1099511628211ull; }
        return h;
    }
    size_t operator()(std::string const& s) const noexcept { return (*this)(std::string_view{s}); }
};

struct TransparentEq {
    using is_transparent = void;
    bool operator()(std::string_view a, std::string_view b) const noexcept { return a==b; }
};

int main(){
    std::unordered_map<std::string, int, TransparentHash, TransparentEq> mp;
    mp.reserve(1024); // 리해시 최소화
    mp.emplace("alpha", 1);
    // 이형 조회: string_view로 검색(복사 없음)
    auto it = mp.find(std::string_view{"alpha"});
    if (it != mp.end()) { /* ... */ }
}
```

---

## 9. 보안: 적대적 충돌과 솔트

- 공격자가 의도적으로 충돌을 유발하면 해시 테이블은 **O(n)** 로 추락.
- 대응: **솔트(salt)를 섞은 해시**(실행마다 임의 시드), **SipHash** 같은 보안 해시 채택.

---

## 10. 캐시·메모리 관점

- 체이닝: 노드 할당/포인터 추적 → **캐시 미스** 증가
- 개방 주소법: **연속 메모리** → 캐시 히트율 높음
  단, **리해시 시 이동 비용**과 **삭제 복잡성** 존재
- Robin Hood: 분산 균등, 최악 탐사 길이 감소 경향

---

## 11. 수학 정리(기대 비용)

체이닝(균등 분포 가정, 로드 팩터 \( \alpha=n/m \)):
- 성공 탐색 평균 비교 횟수 \( \approx 1 + \alpha/2 \)
- 실패 탐색 평균 비교 횟수 \( \approx 1 + \alpha \)

선형 탐사(균등 해시 가정):
- 성공:
  $$ E[\text{probes}] \approx \frac{1}{2}\left(1 + \frac{1}{1-\alpha}\right) $$
- 실패:
  $$ E[\text{probes}] \approx \frac{1}{2}\left(1 + \frac{1}{(1-\alpha)^2}\right) $$

이중 해싱은 선형보다 **클러스터링이 적어** 실전 성능이 좋은 편.

---

## 12. 시나리오 예시 — 톱 N 조회 로그 카운팅

- 키: `std::string_view`(로그 라인), 값: 카운트
- 컨테이너: `unordered_map<string, int>` + 투명 해시
- 튜닝: `reserve(라인수/α)`, `max_load_factor(0.7)`, 입력 버퍼링

```cpp
#include <unordered_map>
#include <string>
#include <string_view>
#include <vector>
#include <algorithm>
#include <iostream>

int main(){
    std::unordered_map<std::string, int> freq;
    freq.reserve(1<<20); // 약 100만 라인 예상
    std::ios::sync_with_stdio(false);
    std::cin.tie(nullptr);

    for (std::string line; std::getline(std::cin, line); )
        ++freq[line];

    std::vector<std::pair<std::string,int>> v(freq.begin(), freq.end());
    std::partial_sort(v.begin(), v.begin()+std::min<size_t>(10, v.size()), v.end(),
                      [](auto const& a, auto const& b){ return a.second>b.second; });

    for (size_t i=0; i<std::min<size_t>(10, v.size()); ++i)
        std::cout << v[i].first << " : " << v[i].second << "\n";
}
```

---

## 13. FAQ & 체크리스트

- **왜 `reserve`?** 대량 삽입 중 리해시를 줄여 **상수 배** 빠르다.
- **정렬된 결과가 필요하면?** 해시는 **순서가 없다**. 끝에서 `std::vector`로 옮겨 정렬하라.
  정렬 유지가 필요하면 처음부터 `std::map`/`std::set`.
- **삭제가 느려지는 이유(개방 주소법)**: Tombstone이 쌓여 **탐색 길이 증가** → 주기적 **재해시** 필요.
- **버킷 수는 소수? 2의 거듭제곱?**
  - 모듈로 비용/마스킹 최적화와 해시 특성에 따라 선택.
  - 2의 거듭제곱이면 하위 비트가 중요 → **해시 혼합**이 좋아야 한다.
- **스레드 안전?** 표준 컨테이너는 기본적으로 **비동시성**. 외부 락이나 분할 락 기법 필요.

---

## 14. 요약

| 항목 | 핵심 |
|---|---|
| 목적 | 평균 O(1) 삽입/탐색/삭제 |
| 구성 | 해시 함수 + 버킷 배열 + 충돌 처리 |
| 충돌 | 체이닝(단순/삭제 쉬움), 개방 주소(캐시 친화) |
| 로드 팩터 | 체이닝: α>1도 가능, 개방 주소: α<1 유지(보통 0.5~0.75) |
| 리해시 | 임계 초과 시 버킷 확장 + 재배치(상수분할 상환 O(1)) |
| 고급 기법 | Robin Hood, Cuckoo, 보안 해시(SipHash) |
| STL | `unordered_map/set` — `reserve`, `max_load_factor`, 커스텀 hash/equal, 이형 조회 |

---

## 15. 간단 벤치마크 드라이버(데모)

```cpp
#include <chrono>
#include <random>
#include <iostream>
#include <unordered_map>
#include <vector>

int main(){
    const size_t N = 1'000'000;
    std::vector<uint64_t> keys(N);
    std::mt19937_64 rng(12345);
    for (auto& k: keys) k = rng();

    std::unordered_map<uint64_t, uint64_t> mp;
    mp.reserve(N);

    auto t0 = std::chrono::high_resolution_clock::now();
    for (auto k: keys) mp.emplace(k, k);
    auto t1 = std::chrono::high_resolution_clock::now();
    size_t hit=0;
    for (auto k: keys) if (mp.find(k)!=mp.end()) ++hit;
    auto t2 = std::chrono::high_resolution_clock::now();

    std::chrono::duration<double> ins = t1-t0;
    std::chrono::duration<double> q   = t2-t1;

    std::cout << "insert " << N << " in " << ins.count() << "s\n";
    std::cout << "query  " << N << " in " << q.count()   << "s, hit="<<hit<<"\n";
}
```

---

## 부록 A. 이중 해싱 파라미터 주의

- \( h_2(k) \) 는 **0이 아니고**, 버킷 수 \( m \) 과 **서로소**여야 모든 슬롯을 순회.
- 예: \( m \) 이 2의 거듭제곱일 때 \( h_2(k) \) 를 **홀수**로 강제.

```cpp
size_t h1 = hasher(key) & (m-1);
size_t h2 = (splitmix64(hasher(key)) | 1ull) & (m-1); // 홀수 보장
i = (h1 + t*h2) & (m-1);
```

---

## 부록 B. Cuckoo Hashing(개요 코드)

{% raw %}
```cpp
// 간략 개념 스케치(실전용 아님)
template<class K, class V, class H1, class H2>
struct Cuckoo {
    std::vector<std::optional<std::pair<K,V>>> t1, t2;
    H1 h1; H2 h2;

    bool insert(K k, V v){
        size_t i = h1(k) % t1.size();
        for (int step=0; step<64; ++step){
            if (!t1[i]) { t1[i] = {{std::move(k), std::move(v)}}; return true; }
            std::swap(k, t1[i]->first); std::swap(v, t1[i]->second);
            size_t j = h2(k) % t2.size();
            if (!t2[j]) { t2[j] = {{std::move(k), std::move(v)}}; return true; }
            std::swap(k, t2[j]->first); std::swap(v, t2[j]->second);
            i = h1(k) % t1.size();
        }
        // 루프 감지 → 재해시 필요
        return false;
    }
};
```
{% endraw %}

실전 구현은 **재해시**, **사이클 탐지**, **로드 팩터 관리**가 중요.
