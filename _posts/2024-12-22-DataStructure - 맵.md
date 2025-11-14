---
layout: post
title: Data Structure - 맵
date: 2024-12-22 19:20:23 +0900
category: Data Structure
---
# C++ `std::map` 완전 정복

## `std::map` 한눈에 보기

`std::map<Key, T>`는 **정렬된 키**에 대해 **유일한 매핑**을 제공하는 연관 컨테이너다.

- 내부: **Red-Black Tree(RBT)** 기반(대부분의 표준 라이브러리 구현)
- 정렬: 기본은 `std::less<Key>` — 즉 `operator<` (커스텀 비교자 사용 가능)
- 복잡도: 삽입/탐색/삭제 모두 **O(log n)**
- 반복: **정렬 순서**로 순회
- 값 접근: `operator[]`, `at`, `insert`, `emplace`, `try_emplace`, `insert_or_assign` 등

```cpp
#include <map>
#include <string>
#include <iostream>

int main(){
    std::map<std::string,int> m;
    m["apple"]  = 5;            // 존재 없으면 노드 생성(값 기본생성) 후 대입
    m.insert({"banana", 2});    // 삽입 (중복키면 무시)
    m.emplace("orange", 7);     // 제자리 생성

    for (auto const& [k,v] : m) // 키 정렬 순서
        std::cout << k << ": " << v << "\n";
}
```

---

## 핵심 연산 빠르게 훑기

| 분류 | 멤버 | 설명/특징 |
|---|---|---|
| 조회 | `find`, `contains(C++20)`, `count`, `at` | `at`은 실패 시 예외 |
| 범위 | `lower_bound`, `upper_bound`, `equal_range` | 정렬기반 범위 탐색 |
| 삽입 | `insert`, `emplace`, `try_emplace(C++17)`, `insert_or_assign(C++17)` | 키 존재 시 행동이 다름 |
| 수정 | `operator[]`, `erase`, `extract(C++17)`, `merge(C++17)` | `extract`로 node handle 이동 |
| 반복 | `begin/end`, `rbegin/rend` | 정렬 순서(양방향 반복자) |
| 기타 | `key_comp`, `value_comp`, `get_allocator` | 비교자/할당자 조회 |

> **팁**: 키가 존재할 수도 없을 수도 있을 때
>
> - “없으면 생성, 있으면 건드리지 않기” → `try_emplace(k, ctor_args...)`
> - “없으면 생성, 있으면 값만 바꾸기” → `insert_or_assign(k, value)`

---

## 정렬·비교자와 “strict weak ordering”

비교자는 기본적으로 `std::less<Key>`이며 **엄격 약순서(strict weak ordering)** 를 만족해야 한다.
그렇지 않으면(예: 비추이/비정합 비교) **트리 불변식이 깨지고 UB**(정의되지 않은 동작)가 발생할 수 있다.

### 이형(heterogeneous) 조회 — 투명 비교자(C++14)

대규모 문자열 키에서 “`std::string` vs `std::string_view`”를 복사 없이 비교하려면 **투명 비교자** 사용:

```cpp
#include <map>
#include <string>
#include <string_view>

struct transparent_less {
    using is_transparent = void; // 핵심: 투명 태그
    template<class L, class R>
    bool operator()(L const& l, R const& r) const { return l < r; }
};

int main(){
    std::map<std::string, int, transparent_less> m;
    m.emplace("alpha", 1);
    // string_view로 이형 탐색 (키 복사 없음)
    auto it = m.find(std::string_view{"alpha"});
}
```

---

## RBT 높이 경계와 시간복잡도(스케치)

RBT는 다음 불변식을 갖는다:

1. 각 노드는 **빨강 또는 검정**.
2. 루트는 검정.
3. 모든 NIL(리프 센티넬)은 검정.
4. 빨강 노드의 자식은 둘 다 **검정**.
5. 임의 노드에서 내려가는 **모든 단순 경로**의 **검정 노드 수(black-height)** 는 동일.

이때, 흑높이를 \(bh\) 라 하면 최소 노드 수는
\[
n \ge 2^{bh}-1
\]
을 만족한다. 또한 “빨강은 연속될 수 없음”에서 **경로 길이 \(h\)** 는
\[
h \le 2 \cdot bh \le 2 \log_2(n+1)
\]
따라서 탐색/삽입/삭제의 단일 연산 경로 길이는 **O(log n)** 이고, 회전은 O(1)이므로 총 **O(log n)**.

---

## 반복자와 예외/유효성 규칙

- **삽입**: 기존 반복자는 **무효화되지 않음** (트리 노드가 재배치되지 않음)
- **삭제**: 삭제된 원소에 대한 반복자만 무효화. 다른 반복자는 유효
- **예외 안전성**:
  - 비교자/할당자/노드 생성에서 던져질 수 있음
  - 표준 컨테이너는 대체로 **strong/commit 또는 basic guarantee** 유지

---

## 값 접근·삽입 패턴 정리

```cpp
std::map<std::string, int> m;

// 1) []: 없으면 default-construct 후 참조 반환
m["cat"] += 1;         // 키 없으면 0으로 생성 후 1

// 2) at: 없으면 std::out_of_range
try { m.at("dog") = 5; } catch (...) { /* no key */ }

// 3) insert: 중복키 무시
m.insert({"ant", 2});  // pair<iterator,bool> 반환

// 4) emplace: 제자리 생성(불필요 복사 감소)
m.emplace("bee", 3);

// 5) try_emplace: 키 없을 때만 생성자 호출
m.try_emplace("cow", 7);         // cow 없으면 생성
m.try_emplace("bee", 9);         // bee 있으므로 아무 것도 안 함

// 6) insert_or_assign: 있으면 대입
m.insert_or_assign("bee", 9);    // bee -> 9
```

---

## 범위 탐색 — lower/upper/equal_range

```cpp
auto it = m.lower_bound("fox");  // fox 이상 첫 원소
auto jt = m.upper_bound("fox");  // fox 초과 첫 원소
auto [lb, ub] = m.equal_range("fox"); // [lb, ub) = 키==fox 범위
```

---

## 노드 핸들 — `extract` / `merge` (C++17)

컨테이너 간에 **노드를 이동**하며 비교자/할당자 제약을 피하기 좋다.

```cpp
std::map<std::string,int> a, b;
a.emplace("x", 1);
auto nh = a.extract("x");  // a에서 노드 분리 (비어 있으면 empty)
if (nh) {
    nh.key() = "y";        // 키 수정 가능!
    b.insert(std::move(nh)); // b로 이동
}
```

---

## 성능 팁과 함정

- **비교자 비용** 줄이기: 문자열 → 투명 비교자 + `string_view` 입력
- `operator[]`는 **없으면 생성**이므로 단순 조회는 `find/contains` 를 쓰자
- 커스텀 비교자에서 **엄격 약순서** 위반 금지
- 대량 삽입은 **정렬된 입력**일 때도 `std::map`은 균형유지로 O(n log n) — 대량 빌드는 `std::vector` 정렬 + `std::map` 교체(또는 `std::pmr`/커스텀 트리) 고려

---

## 교육용 Red-Black Tree 기반 미니 `map` 구현

> 학습용으로 **삽입/검색/순회/삭제** 가 되는 간단 구현을 제시한다.
> 실사용 목적이면 표준 컨테이너를 쓰자(예외·경계·이동·노드핸들 등 미비).

### 전체 코드

```cpp
// rbt_map.cpp (교육용 예제 — 단일 파일)
// 빌드: g++ -std=c++20 -O2 rbt_map.cpp

#include <bits/stdc++.h>

using namespace std;

enum Color { RED, BLACK };

template <class Key, class T, class Cmp = std::less<Key>>
class rb_map {
    struct Node {
        Key   key;
        T     val;
        Color color;
        Node *p, *l, *r;
        Node(Key k, T v, Color c, Node* nil)
            : key(std::move(k)), val(std::move(v)), color(c), p(nil), l(nil), r(nil) {}
    };

    Node* root;
    Node* NIL;            // 공용 센티넬(검정)
    size_t sz = 0;
    Cmp cmp;

public:
    rb_map(): cmp() {
        NIL = (Node*)::operator new(sizeof(Node));
        // NIL 내용은 쓰지 않지만 포인터/색만 활용
        NIL->color = BLACK; NIL->p = NIL->l = NIL->r = NIL;
        root = NIL;
    }
    ~rb_map(){ clear_node(root); ::operator delete(NIL); }

    rb_map(rb_map const&) = delete;
    rb_map& operator=(rb_map const&) = delete;

    // --- public API ---
    size_t size()  const { return sz; }
    bool   empty() const { return sz==0; }

    // 삽입: 키 존재 시 값 갱신 (std::map과 다르게 insert_or_assign 형태)
    void insert_or_assign(Key key, T val){
        Node* z = new Node(std::move(key), std::move(val), RED, NIL);
        Node* y = NIL; Node* x = root;

        while (x != NIL) {
            y = x;
            if (cmp(z->key, x->key)) x = x->l;
            else if (cmp(x->key, z->key)) x = x->r;
            else {
                // 키 동일 — 갱신 후 삭제
                x->val = std::move(z->val);
                delete z;
                return;
            }
        }
        z->p = y;
        if (y == NIL) root = z;
        else if (cmp(z->key, y->key)) y->l = z;
        else y->r = z;
        ++sz;
        insert_fix(z);
    }

    // 탐색
    T* get(Key const& k){
        Node* x = root;
        while (x != NIL) {
            if (cmp(k, x->key)) x = x->l;
            else if (cmp(x->key, k)) x = x->r;
            else return &x->val;
        }
        return nullptr;
    }

    // 하한(>= key)
    pair<Key const*, T*> lower_bound_ptr(Key const& k){
        Node* x = root; Node* ans = NIL;
        while (x != NIL){
            if (!cmp(x->key, k)) { ans = x; x = x->l; }
            else x = x->r;
        }
        if (ans==NIL) return {nullptr,nullptr};
        return { &ans->key, &ans->val };
    }

    // 삭제: 키 존재 시 한 개 삭제
    bool erase(Key const& k){
        Node* z = root;
        while (z != NIL){
            if (cmp(k, z->key)) z = z->l;
            else if (cmp(z->key, k)) z = z->r;
            else break;
        }
        if (z==NIL) return false;

        Node* y = z;
        Node* x = NIL;
        Color yorig = y->color;

        if (z->l == NIL) {
            x = z->r;
            transplant(z, z->r);
        } else if (z->r == NIL) {
            x = z->l;
            transplant(z, z->l);
        } else {
            y = minimum(z->r);
            yorig = y->color;
            x = y->r;
            if (y->p == z) {
                x->p = y;
            } else {
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

    // 중위순회 출력(디버깅)
    void debug_inorder() const {
        inorder(root); std::cout << "\n";
    }

    // RBT 불변식 검사(디버깅): 모든 경로의 black-height 동일?
    bool check_invariants() const {
        if (root==NIL) return true;
        if (root->color != BLACK) return false;
        int target=-1;
        return dfs_check(root, 0, target);
    }

private:
    // --- 내부 유틸 ---
    void clear_node(Node* x){
        if (x==NIL) return;
        clear_node(x->l); clear_node(x->r);
        delete x;
    }
    void inorder(Node* x) const {
        if (x==NIL) return;
        inorder(x->l);
        std::cout << x->key << "(" << (x->color==RED?'R':'B') << "):" << x->val << " ";
        inorder(x->r);
    }
    Node* minimum(Node* x){ while (x->l!=NIL) x=x->l; return x; }

    void left_rotate(Node* x){
        Node* y = x->r;
        x->r = y->l;
        if (y->l != NIL) y->l->p = x;
        y->p = x->p;
        if (x->p==NIL) root=y;
        else if (x==x->p->l) x->p->l=y; else x->p->r=y;
        y->l = x; x->p=y;
    }
    void right_rotate(Node* x){
        Node* y = x->l;
        x->l = y->r;
        if (y->r != NIL) y->r->p = x;
        y->p = x->p;
        if (x->p==NIL) root=y;
        else if (x==x->p->r) x->p->r=y; else x->p->l=y;
        y->r = x; x->p=y;
    }
    void insert_fix(Node* z){
        while (z->p->color == RED){
            if (z->p == z->p->p->l){
                Node* y = z->p->p->r; // 삼촌
                if (y->color == RED){
                    z->p->color = BLACK;
                    y->color    = BLACK;
                    z->p->p->color = RED;
                    z = z->p->p;
                }else{
                    if (z == z->p->r){ z = z->p; left_rotate(z); }
                    z->p->color = BLACK;
                    z->p->p->color = RED;
                    right_rotate(z->p->p);
                }
            }else{
                Node* y = z->p->p->l;
                if (y->color == RED){
                    z->p->color = BLACK;
                    y->color    = BLACK;
                    z->p->p->color = RED;
                    z = z->p->p;
                }else{
                    if (z == z->p->l){ z = z->p; right_rotate(z); }
                    z->p->color = BLACK;
                    z->p->p->color = RED;
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
                }else{
                    if (w->r->color==BLACK){
                        w->l->color=BLACK; w->color=RED; right_rotate(w);
                        w=x->p->r;
                    }
                    w->color = x->p->color;
                    x->p->color=BLACK;
                    w->r->color=BLACK;
                    left_rotate(x->p);
                    x=root;
                }
            }else{
                Node* w=x->p->l;
                if (w->color==RED){
                    w->color=BLACK; x->p->color=RED; right_rotate(x->p);
                    w=x->p->l;
                }
                if (w->r->color==BLACK && w->l->color==BLACK){
                    w->color=RED; x=x->p;
                }else{
                    if (w->l->color==BLACK){
                        w->r->color=BLACK; w->color=RED; left_rotate(w);
                        w=x->p->l;
                    }
                    w->color = x->p->color;
                    x->p->color=BLACK;
                    w->l->color=BLACK;
                    right_rotate(x->p);
                    x=root;
                }
            }
        }
        x->color=BLACK;
    }

    bool dfs_check(Node* x, int blacks, int& target) const {
        if (x==NIL){
            if (target<0) target=blacks;
            return target==blacks;
        }
        if (x->color==BLACK) ++blacks;
        // 빨강 부모-자식 연속 금지
        if (x->color==RED){
            if (x->l->color==RED || x->r->color==RED) return false;
        }
        return dfs_check(x->l,blacks,target) && dfs_check(x->r,blacks,target);
    }
};

// --- 데모 ---
int main(){
    rb_map<std::string,int> M;
    M.insert_or_assign("banana", 3);
    M.insert_or_assign("apple",  1);
    M.insert_or_assign("cherry", 2);
    M.insert_or_assign("banana", 5); // 갱신

    if (auto p = M.get("banana")) std::cout << "banana=" << *p << "\n";

    auto [kptr, vptr] = M.lower_bound_ptr("blue");
    if (kptr) std::cout << "lower_bound(blue) -> " << *kptr << ":" << *vptr << "\n";

    std::cout << "inorder: "; M.debug_inorder();
    std::cout << "erase apple -> " << M.erase("apple") << "\n";
    std::cout << "inorder: "; M.debug_inorder();

    std::cout << "RBT invariants: " << (M.check_invariants() ? "OK" : "BROKEN") << "\n";
}
```

#### 구현 포인트

- **센티넬 `NIL`(검정)**: null 대신 공용 노드를 사용해 분기 단순화
- **회전/수선**: 삽입 `insert_fix`, 삭제 `erase_fix` — 표준 RBT 알고리즘
- **높이 보장**: 삽입/삭제 각 \(O(\log n)\)
- **학습용**: 예외/이동/반복자/노드 핸들 등은 생략

---

## `std::map`로 동일 시나리오 작성(참고)

```cpp
#include <map>
#include <iostream>
#include <string>

int main(){
    std::map<std::string,int> m;
    m.insert_or_assign("banana", 3);
    m.insert_or_assign("apple",  1);
    m.insert_or_assign("cherry", 2);
    m.insert_or_assign("banana", 5); // 갱신

    if (auto it = m.find("banana"); it!=m.end())
        std::cout << "banana=" << it->second << "\n";

    auto lb = m.lower_bound("blue");
    if (lb!=m.end())
        std::cout << "lower_bound(blue) -> " << lb->first << ":" << lb->second << "\n";

    for (auto const& [k,v] : m) std::cout << k << ":" << v << " ";
    std::cout << "\n";

    m.erase("apple");
    for (auto const& [k,v] : m) std::cout << k << ":" << v << " ";
    std::cout << "\n";
}
```

---

## 자주 묻는 질문(FAQ)

- **Q. 키를 수정해도 되나요?**
  **안 됩니다.** 키는 정렬 순서를 결정하므로, 요소를 통해 키를 변경하면 트리 불변식이 깨진다. 키를 바꾸려면 **`extract` → `key()` 수정 → `insert`**.

- **Q. 대량 삽입이 느린데요?**
  트리 특성상 균형유지로 \(O(n\log n)\). 미리 데이터를 정렬해도 큰 개선은 없다.
  오프라인 대량 구축은 다른 자료구조/전용 빌더를 고려.

- **Q. `unordered_map`과 무엇이 다른가요?**
  정렬 보장이 필요하면 `std::map`. 해시 테이블은 평균 \(O(1)\) 이지만 순서가 없다.

---

## 마무리 요약

- `std::map`은 **정렬 보장 + 로그 시간 연산**을 제공하는 **RBT 기반** 연관 컨테이너.
- **비교자**는 **엄격 약순서**를 지켜야 하며, **투명 비교자**로 이형 조회 최적화 가능.
- **삽입 패턴**을 올바르게 선택(`try_emplace`, `insert_or_assign` 등)하면 **불필요한 생성/복사**를 줄일 수 있다.
- 내부적으로는 **RBT 회전/수선**으로 **높이 \( \le 2\log_2(n+1) \)** 를 유지 → **O(log n)** 보장.
- 본문 교육용 **RBT 구현**으로 핵심 동작을 추적해보면, 표준 컨테이너의 동작/복잡도 보장이 더 명확해진다.
