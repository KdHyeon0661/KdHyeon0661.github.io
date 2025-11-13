---
layout: post
title: Data Structure - 집합
date: 2024-12-23 19:20:23 +0900
category: Data Structure
---
# C++ STL `set`

## 1. 개요 — 왜 `set`인가?

`std::set<T>`는 **중복 없는 원소**를 **정렬된 상태**로 보관하는 연관 컨테이너다. 핵심 특성:

- 내부적으로 **Red-Black Tree(RBT)** 기반의 **자체 균형 이진 탐색 트리**
- 정렬 기준은 기본적으로 `std::less<T>` (사용자 정의 비교자 지정 가능)
- **삽입/탐색/삭제: `O(log n)`**, **정렬 순회: `O(n)`**
- 원소 자체가 **키 == 값**(즉, `map`의 `pair<key,value>` 대신 `key` 단독)

```cpp
#include <set>
#include <iostream>
int main() {
    std::set<int> s = {20, 5, 10, 20}; // 20 중복은 자동 제거
    for (int x : s) std::cout << x << " "; // 5 10 20
}
```

---

## 2. 가족들: `set` vs `multiset` vs `unordered_set` vs `unordered_multiset`

| 항목               | `set`                             | `multiset`                      | `unordered_set`                     | `unordered_multiset`              |
|--------------------|-----------------------------------|----------------------------------|-------------------------------------|-----------------------------------|
| 정렬               | 예                                 | 예                                | 아니오                               | 아니오                             |
| 중복               | 불가                               | 허용                              | 불가                                 | 허용                               |
| 내부 구조          | Red-Black Tree                    | Red-Black Tree                   | 해시 테이블                          | 해시 테이블                        |
| 탐색/삽입/삭제     | `O(log n)`                        | `O(log n)`                       | 평균 `O(1)`, 최악 `O(n)`             | 평균 `O(1)`, 최악 `O(n)`          |
| 순회 순서          | 정렬 순                            | 정렬 순                           | 정의 없음                            | 정의 없음                          |
| 비교자/해시 커스텀 | 비교자                             | 비교자                            | 해시/동등자                          | 해시/동등자                        |

> 정렬이 필요하면 `set/multiset`, 최대 속도(평균상수) 위주면 `unordered_*`가 적합.

---

## 3. 시간 복잡도 스케치 — RBT 높이 경계

Red-Black Tree 불변식(요지):
1. 각 노드는 **빨강/검정** 중 하나.
2. 루트는 **검정**.
3. 모든 NIL(센티넬) 노드는 **검정**.
4. **빨강 노드의 자식은 모두 검정**.
5. 임의 노드에서 아래 모든 리프까지의 **검정 노드 수(black-height)** 는 동일.

흑높이를 \( bh \)라 하면 최소 노드 수는
$$
n \ge 2^{bh} - 1
$$
경로 길이 \( h \)는
$$
h \le 2\cdot bh \le 2 \log_2(n+1)
$$
즉, 탐색/삽입/삭제의 경로 길이는 **\( O(\log n) \)**.

---

## 4. 핵심 인터페이스 요약

| 분류 | 멤버/함수 | 메모 |
|---|---|---|
| 생성 | `set()`/`set(comp)`/`set(first,last)` | 비교자 지정 및 범위 생성 가능 |
| 조회 | `find`, `contains(C++20)`, `count` | `contains`는 bool, `find`는 iterator |
| 범위 | `lower_bound`, `upper_bound`, `equal_range` | 정렬 기반 범위 탐색 |
| 삽입 | `insert`, `emplace` | 중복이면 무시(`insert`는 `pair<it,bool>` 반환) |
| 삭제 | `erase(key)`, `erase(it)`, `erase(first,last)` | 반복자 삭제 안전 |
| 합성 | `merge(C++17)` | 다른 set에서 노드 **이동** |
| 노드 | `extract(C++17)` | 노드 핸들 추출/재삽입(키 수정 가능) |
| 반복 | `begin/end`, `rbegin/rend` | 정렬 순회/역순 순회 |

실전 예:

```cpp
#include <set>
#include <string>
#include <iostream>
int main(){
    std::set<std::string> s;

    // 1) insert
    auto [it, inserted] = s.insert("alpha");
    s.insert("beta");
    s.emplace("gamma");

    // 2) contains/find
    if (s.contains("beta")) std::cout << "beta ok\n";
    if (auto it = s.find("delta"); it == s.end()) std::cout << "delta none\n";

    // 3) lower/upper
    auto lb = s.lower_bound("b");
    if (lb != s.end()) std::cout << "lower_bound(b): " << *lb << "\n";

    // 4) erase
    s.erase("alpha");

    // 5) iterate
    for (auto const& x : s) std::cout << x << " "; // beta gamma
    std::cout << "\n";
}
```

---

## 5. 범위 탐색 패턴 — `lower_bound`/`upper_bound`/`equal_range`

정렬 특성을 살린 구간 추출:

```cpp
#include <set>
#include <iostream>
int main(){
    std::set<int> s = {1,3,5,7,9,11,13};
    auto itL = s.lower_bound(4);  // 4 이상 → 5
    auto itR = s.upper_bound(10); // 10 초과 → 11
    for (auto it = itL; it != itR; ++it)
        std::cout << *it << " ";  // 5 7 9
}
```

`equal_range(x)` 는 `[lower_bound(x), upper_bound(x))` 반환. `set`은 중복이 없으니 길이 0 또는 1 구간.

---

## 6. 사용자 정의 비교자 — 정렬 기준 커스터마이징

### 6.1 내림차순 / 절댓값 기반 등

```cpp
struct Desc {
    bool operator()(int a, int b) const { return a > b; }
};
std::set<int, Desc> sdesc = {3,1,2};   // 3 2 1
```

주의: **엄격 약순서(strict weak ordering)** 를 지켜야 한다.
- 추이성/반사성/비대칭 등을 만족하지 않으면 트리 불변식 파괴 → UB.

### 6.2 대소문자 무시 비교 (문자열)

```cpp
#include <cctype>
struct CiLess {
    bool operator()(std::string const& a, std::string const& b) const {
        auto f = [](unsigned char c){ return std::tolower(c); };
        size_t n = std::min(a.size(), b.size());
        for (size_t i=0; i<n; ++i) {
            auto ac = f(a[i]), bc = f(b[i]);
            if (ac < bc) return true;
            if (ac > bc) return false;
        }
        return a.size() < b.size();
    }
};
std::set<std::string, CiLess> ci;
ci.insert("ALPHA"); ci.insert("alpha"); // 중복 취급 → 하나만 저장
```

---

## 7. 투명 비교자 — 이형(heterogeneous) 조회 최적화

복사 없이 `string_view`로 검색하려면 **투명 비교자**:

```cpp
struct TransparentLess {
    using is_transparent = void; // 중요: 투명 태그
    template<class L, class R>
    bool operator()(L const& l, R const& r) const { return l < r; }
};

#include <string>
#include <string_view>
#include <set>

int main(){
    std::set<std::string, TransparentLess> s;
    s.insert("zeta");
    auto it = s.find(std::string_view{"zeta"}); // 임시 string 생성 없이 탐색
}
```

---

## 8. 반복자 무효화·예외 안전성

- **삽입**: 기존 반복자 **유효**(RBT 노드는 연결구조라 재배치 없음)
- **삭제**: 삭제된 원소의 반복자만 무효, 그 외 **유효**
- 예외 안전성: 노드 할당/비교자에서 예외 가능. 표준 컨테이너는 일반적으로 **strong/basic guarantee**를 준수

반복자 삭제 패턴:

```cpp
for (auto it = s.begin(); it != s.end(); ) {
    if (*it % 2 == 0) it = s.erase(it); // C++11: erase(it) -> 다음 반복자
    else ++it;
}
```

---

## 9. 표준 알고리즘과의 조합 — 집합 연산

정렬 컨테이너이므로 `<algorithm>`의 집합 알고리즘과 친화적:

```cpp
#include <algorithm>
#include <vector>
#include <set>
#include <iostream>

int main(){
    std::set<int> A = {1,2,3,7};
    std::set<int> B = {3,4,5,7};

    std::vector<int> uni, inter, diff, sym;

    std::set_union(A.begin(), A.end(), B.begin(), B.end(), std::back_inserter(uni));
    std::set_intersection(A.begin(), A.end(), B.begin(), B.end(), std::back_inserter(inter));
    std::set_difference(A.begin(), A.end(), B.begin(), B.end(), std::back_inserter(diff));
    std::set_symmetric_difference(A.begin(), A.end(), B.begin(), B.end(), std::back_inserter(sym));

    auto print = [](auto const& v){ for (int x: v) std::cout<<x<<" "; std::cout<<"\n"; };
    print(uni);   // 1 2 3 4 5 7
    print(inter); // 3 7
    print(diff);  // 1 2
    print(sym);   // 1 2 4 5
}
```

---

## 10. 실전 예제 1 — 로그 유니크 및 범위 질의

시나리오: 로그에서 **중복 라인 제거** + **접두 조건 범위 조회**.

```cpp
#include <set>
#include <string>
#include <iostream>

int main(){
    std::set<std::string> logs;
    for (std::string line; std::getline(std::cin, line); )
        logs.insert(std::move(line)); // 자동 유니크 + 정렬

    // "WARN" 이상(사전순)만 출력
    for (auto it = logs.lower_bound("WARN"); it != logs.end(); ++it)
        std::cout << *it << "\n";
}
```

---

## 11. 실전 예제 2 — 구조체 키와 비교자, 투명 조회 결합

```cpp
#include <set>
#include <string>
#include <string_view>
#include <iostream>

struct User {
    std::string id;   // 키
    std::string name; // 기타 데이터
};

struct ById {
    using is_transparent = void; // 투명 비교자
    bool operator()(User const& a, User const& b) const { return a.id < b.id; }
    bool operator()(User const& a, std::string_view b) const { return a.id < b; }
    bool operator()(std::string_view a, User const& b) const { return a < b.id; }
};

int main(){
    std::set<User, ById> users;
    users.insert({"u001","Alice"});
    users.insert({"u010","Bob"});
    users.insert({"u005","Cara"});

    if (auto it = users.find("u005"); it != users.end())
        std::cout << it->name << "\n"; // Cara

    // id 범위 [u005, u010)
    for (auto it = users.lower_bound("u005");
         it != users.lower_bound("u010"); ++it)
        std::cout << it->id << ":" << it->name << "\n";
}
```

---

## 12. 내부 구조 심화 — 교육용 Red-Black Tree 기반 `set` 구현

> 표준 구현은 훨씬 복잡(할당자, 예외 안전, 반복자, 노드 핸들 등).
> 아래 코드는 **학습용**으로 RBT의 핵심(삽입/삭제/회전/수선/중위순회)을 담은 간단 `rb_set`이다.
> 실사용은 반드시 `std::set`을 쓰자.

### 12.1 설계 포인트

- **센티넬 `NIL` 노드(검정)** 를 사용해 `nullptr` 분기를 최소화
- 삽입 수선(`insert_fix`), 삭제 수선(`erase_fix`)은 표준 RBT 알고리즘
- 키 중복 시 무시(= `set`语義)

### 12.2 코드

```cpp
// rb_set.cpp — 교육용 예제 (단일 translation unit)
// 빌드: g++ -std=c++20 -O2 rb_set.cpp
#include <bits/stdc++.h>
using namespace std;

enum Color { RED, BLACK };

template<class Key, class Comp = std::less<Key>>
class rb_set {
    struct Node {
        Key   key;
        Color color;
        Node *p, *l, *r;
        Node(Key k, Color c, Node* nil)
            : key(std::move(k)), color(c), p(nil), l(nil), r(nil) {}
    };

    Node* root;
    Node* NIL;        // 공용 센티넬
    size_t sz = 0;
    Comp comp;

public:
    rb_set(): comp() {
        NIL = (Node*)::operator new(sizeof(Node)); // dummy memory
        NIL->color = BLACK; NIL->p = NIL->l = NIL->r = NIL;
        root = NIL;
    }
    ~rb_set(){ clear_node(root); ::operator delete(NIL); }

    rb_set(rb_set const&)            = delete;
    rb_set& operator=(rb_set const&) = delete;

    bool empty() const { return sz==0; }
    size_t size() const { return sz; }

    // 삽입: 존재하면 무시, 없으면 삽입
    bool insert(Key key){
        Node* z = new Node(std::move(key), RED, NIL);
        Node* y = NIL; Node* x = root;
        while (x!=NIL){
            y = x;
            if (!comp(x->key, z->key) && !comp(z->key, x->key)){
                delete z; return false; // 중복
            }
            x = comp(z->key, x->key) ? x->l : x->r;
        }
        z->p = y;
        if (y==NIL) root=z;
        else if (comp(z->key, y->key)) y->l = z; else y->r = z;
        ++sz;
        insert_fix(z);
        return true;
    }

    // 삭제: 존재하면 삭제
    bool erase(Key const& k){
        Node* z = root;
        while (z!=NIL){
            if (!comp(z->key, k) && !comp(k, z->key)) break;
            z = comp(k, z->key) ? z->l : z->r;
        }
        if (z==NIL) return false;

        Node* y = z; Node* x = NIL; Color yorig = y->color;
        if (z->l==NIL) { x=z->r; transplant(z, z->r); }
        else if (z->r==NIL) { x=z->l; transplant(z, z->l); }
        else {
            y = minimum(z->r); yorig = y->color; x = y->r;
            if (y->p == z) x->p = y;
            else {
                transplant(y, y->r);
                y->r = z->r; y->r->p = y;
            }
            transplant(z, y);
            y->l = z->l; y->l->p = y;
            y->color = z->color;
        }
        delete z; --sz;
        if (yorig == BLACK) erase_fix(x);
        return true;
    }

    // 조회
    bool contains(Key const& k) const {
        Node* x = root;
        while (x!=NIL){
            if (!comp(x->key, k) && !comp(k,x->key)) return true;
            x = comp(k,x->key) ? x->l : x->r;
        }
        return false;
    }

    // 디버그 출력
    void debug_inorder() const { inorder(root); cout << "\n"; }

    // 불변식 검사(루트 검정, red-parent rule, black-height equal)
    bool check_invariants() const {
        if (root==NIL) return true;
        if (root->color != BLACK) return false;
        int target=-1;
        return dfs_check(root, 0, target);
    }

private:
    void clear_node(Node* x){
        if (x==NIL) return;
        clear_node(x->l); clear_node(x->r);
        delete x;
    }
    Node* minimum(Node* x){ while (x->l!=NIL) x=x->l; return x; }

    void left_rotate(Node* x){
        Node* y = x->r;
        x->r = y->l; if (y->l!=NIL) y->l->p = x;
        y->p = x->p;
        if (x->p==NIL) root=y;
        else if (x==x->p->l) x->p->l=y; else x->p->r=y;
        y->l = x; x->p = y;
    }
    void right_rotate(Node* x){
        Node* y = x->l;
        x->l = y->r; if (y->r!=NIL) y->r->p = x;
        y->p = x->p;
        if (x->p==NIL) root=y;
        else if (x==x->p->r) x->p->r=y; else x->p->l=y;
        y->r = x; x->p = y;
    }
    void insert_fix(Node* z){
        while (z->p->color == RED){
            if (z->p == z->p->p->l){
                Node* y = z->p->p->r;
                if (y->color == RED){
                    z->p->color = BLACK; y->color = BLACK;
                    z->p->p->color = RED; z = z->p->p;
                } else {
                    if (z == z->p->r){ z = z->p; left_rotate(z); }
                    z->p->color = BLACK; z->p->p->color = RED;
                    right_rotate(z->p->p);
                }
            } else {
                Node* y = z->p->p->l;
                if (y->color == RED){
                    z->p->color = BLACK; y->color = BLACK;
                    z->p->p->color = RED; z = z->p->p;
                } else {
                    if (z == z->p->l){ z = z->p; right_rotate(z); }
                    z->p->color = BLACK; z->p->p->color = RED;
                    left_rotate(z->p->p);
                }
            }
        }
        root->color = BLACK;
    }
    void transplant(Node* u, Node* v){
        if (u->p==NIL) root=v;
        else if (u==u->p->l) u->p->l=v; else u->p->r=v;
        v->p = u->p;
    }
    void erase_fix(Node* x){
        while (x!=root && x->color==BLACK){
            if (x==x->p->l){
                Node* w=x->p->r;
                if (w->color==RED){
                    w->color=BLACK; x->p->color=RED; left_rotate(x->p);
                    w=x->p->r;
                }
                if (w->l->color==BLACK && w->r->color==BLACK){
                    w->color=RED; x=x->p;
                } else {
                    if (w->r->color==BLACK){
                        w->l->color=BLACK; w->color=RED; right_rotate(w);
                        w=x->p->r;
                    }
                    w->color = x->p->color; x->p->color = BLACK; w->r->color = BLACK;
                    left_rotate(x->p); x = root;
                }
            } else {
                Node* w=x->p->l;
                if (w->color==RED){
                    w->color=BLACK; x->p->color=RED; right_rotate(x->p);
                    w=x->p->l;
                }
                if (w->r->color==BLACK && w->l->color==BLACK){
                    w->color=RED; x=x->p;
                } else {
                    if (w->l->color==BLACK){
                        w->r->color=BLACK; w->color=RED; left_rotate(w);
                        w=x->p->l;
                    }
                    w->color = x->p->color; x->p->color = BLACK; w->l->color = BLACK;
                    right_rotate(x->p); x = root;
                }
            }
        }
        x->color = BLACK;
    }
    void inorder(Node* x) const {
        if (x==NIL) return;
        inorder(x->l);
        cout << x->key << "(" << (x->color==RED?'R':'B') << ") ";
        inorder(x->r);
    }
    bool dfs_check(Node* x, int blacks, int& target) const {
        if (x==NIL){
            if (target<0) target=blacks;
            return target==blacks;
        }
        if (x->color==BLACK) ++blacks;
        if (x->color==RED){
            if (x->l->color==RED || x->r->color==RED) return false;
        }
        return dfs_check(x->l,blacks,target) && dfs_check(x->r,blacks,target);
    }
};

int main(){
    rb_set<int> S;
    for (int v: {10,5,20,15,10,7,3}) S.insert(v); // 10 중복 무시
    S.debug_inorder(); // 3(B) 5(R/B) 7(...) 10(...) 15(...) 20(...)

    S.erase(10); S.erase(100); // 존재하지 않으면 무시
    S.debug_inorder();

    cout << boolalpha << S.contains(7) << " " << S.contains(10) << "\n";
    cout << "RBT invariants: " << S.check_invariants() << "\n";
}
```

핵심 요점:
- **센티넬 `NIL`** 로 `nullptr` 분기 방지 → 코드 간결/안전
- **회전 + 색 재배치** 로 불변식 유지
- 단일 연산의 높이는 \( O(\log n) \)

---

## 13. 노드 핸들/머지/추출 — `std::set` 고급 API

C++17 이후, 노드 이동이 쉬워졌다:

```cpp
#include <set>
#include <string>

int main(){
    std::set<std::string> a = {"a","b"}, b = {"b","c"};

    // 1) merge: b의 노드를 a로 이동(중복은 이동 안 됨)
    a.merge(b); // a: {a,b,c}  b: { }

    // 2) extract: 노드 핸들을 빼내 키 수정 후 재삽입 가능
    auto nh = a.extract("c");
    nh.value() = "z";      // set은 key==value, value()로 직접 수정
    a.insert(std::move(nh)); // a: {a,b,z}
}
```

> `extract`는 컨테이너 노드의 소유권을 넘겨 **키 수정**을 가능하게 한다(`map`에서 특히 유용).

---

## 14. 메모리/성능 관점

- `set`은 **노드 기반**이라 `vector`보다 **메모리 오버헤드**가 크다(포인터 3개+색상 비트 등).
- 대량 데이터의 **순차 처리**나 **인덱스 접근**에는 부적합.
- 문자열 키의 잦은 탐색 → **투명 비교자 + `string_view`** 로临时 객체 생성 비용 절감.
- 할당자 커스텀/메모리 풀(예: `std::pmr::polymorphic_allocator`)로 **할당 비용** 최적화 가능.

---

## 15. 흔한 함정과 FAQ

- **키 수정 불가**: `set` 요소를 통해 키를 수정하면 정렬 불변식이 깨진다 → 금지. 수정하려면 `extract` 사용 후 `insert`.
- **비교자 규칙 위반**: 엄격 약순서 미충족은 치명적. 특히 사용자 정의 비교자에서 **동등 판단**이 일관되게 작동해야 함.
- **중복 처리**: `insert`는 중복 시 무시. 중복 허용이 필요하면 `multiset`.
- **단순 조회에 `operator[]`?** 없음. `map`과 달리 키만 저장하며, 생성 부작용을 일으키는 `[]` 연산자가 없다. 조회는 `find/contains`.

---

## 16. 부가: 순서 통계(Order Statistics)가 필요할 때

표준 `set`은 **k번째 원소** 질의를 직접 지원하지 않는다.
연구/대회 환경에서는 GNU PBDS(`tree_order_statistics_node_update`)를 활용하기도 한다(표준 아님):

```cpp
// 비표준(GNU 확장). GCC 전용.
// #include <ext/pb_ds/assoc_container.hpp>
// using namespace __gnu_pbds;
// template<class T>
// using ordered_set = tree<T, null_type, less<T>, rb_tree_tag, tree_order_statistics_node_update>;
//
// ordered_set<int> os;
// os.insert(10); os.insert(5); os.insert(20);
// *os.find_by_order(1) == 10         // 0-based k번째
// os.order_of_key(15) == 2           // 15 미만 원소 개수
```

실무/이식성을 중시하면 별도 인덱스 구조(세그먼트 트리, 펜윅 트리, 스킵리스트 등) 고려.

---

## 17. 종합 예제 — 로그 집합 + 이형 검색 + 구간 출력

```cpp
#include <set>
#include <string>
#include <string_view>
#include <iostream>

struct TransparentLess {
    using is_transparent = void;
    template<class L, class R>
    bool operator()(L const& l, R const& r) const { return l < r; }
};

int main(){
    std::set<std::string, TransparentLess> S;

    // 입력에서 중복 제거 + 정렬 저장
    for (std::string line; std::getline(std::cin, line) && !line.empty(); )
        S.insert(std::move(line));

    // 특정 키 존재 검사(복사 없이)
    if (S.contains(std::string_view{"ERROR"}))
        std::cout << "ERROR exists\n";

    // [INFO, WARN) 범위 출력
    auto itL = S.lower_bound(std::string_view{"INFO"});
    auto itR = S.lower_bound(std::string_view{"WARN"});
    for (auto it = itL; it != itR; ++it)
        std::cout << *it << "\n";
}
```

---

## 18. 요약

- `std::set`은 **중복 없는 정렬 컨테이너**, 내부는 **Red-Black Tree**.
- **삽입/탐색/삭제 `O(log n)`**, 정렬 순회 가능.
- **비교자 커스텀**과 **투명 비교자**로 실전 성능 최적화.
- 반복자 무효화 규칙은 단순(삭제된 원소만 무효), 예외 안전성 준수.
- 내부 RBT 메커니즘(회전/색변경)을 이해하면 복잡도와 동작을 직관적으로 파악 가능.
- **키 수정 금지**, 필요 시 **`extract` → 수정 → `insert`**.
- 순서 통계가 필요하면 비표준 PBDS나 다른 자료구조를 고려.

위 내용을 습득하면, `std::set`을 단순 “정렬 컨테이너”가 아니라 **로그 집합, 범위 질의, 중복 제거 파이프라인, 이형 검색 최적화** 등 다양한 시나리오에서 **안정적인 `O(log n)` 도구**로 활용할 수 있다.
