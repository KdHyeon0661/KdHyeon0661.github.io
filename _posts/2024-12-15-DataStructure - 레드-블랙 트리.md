---
layout: post
title: Data Structure - 레드-블랙 트리
date: 2024-12-15 20:20:23 +0900
category: Data Structure
---
# 레드-블랙 트리 (Red-Black Tree)

## 0. 핵심 불변식과 용어

### 레드-블랙 5규칙
| 규칙 | 내용 |
|---|---|
| R1 | 각 노드는 **RED** 또는 **BLACK** |
| R2 | **루트는 BLACK** |
| R3 | 모든 **리프(NIL 센티넬)**는 BLACK |
| R4 | **RED 노드의 자식은 둘 다 BLACK** (RED-RED 연속 금지) |
| R5 | 임의의 노드에서 **리프(NIL)**까지 가는 모든 단순 경로의 **BLACK 노드 수(black-height)**가 같다 |

> **black-height** \(bh(x)\): 노드 `x`에서 아래쪽 NIL(리프 센티넬)까지의 경로 중 **BLACK 노드 개수**. (x 자신이 BLACK이면 포함)

### BST 조건
모든 노드에 대해 `left.key < key < right.key` (중복 정책이 필요하면 `<`/`<=` 규칙을 명확히 택한다. 여기선 **중복 금지**로 진행).

---

## 1. 노드 구조와 NIL 센티넬

NIL(리프)까지도 하나의 **공유된 BLACK 노드** 포인터로 다루면, 회전/삭제 Fix-up에서 **널 분기**가 크게 줄어든다.

```cpp
// ===== rbtree.hpp =====
#pragma once
#include <bits/stdc++.h>
using namespace std;

enum Color : uint8_t { RED = 0, BLACK = 1 };

struct RBNode {
    int key;
    Color color;
    RBNode *left, *right, *parent;
    RBNode(int k = 0, Color c = BLACK, RBNode* l=nullptr, RBNode* r=nullptr, RBNode* p=nullptr)
        : key(k), color(c), left(l), right(r), parent(p) {}
};

struct RBTree {
    RBNode* root;
    RBNode* NIL; // shared sentinel, BLACK

    RBTree() {
        NIL = new RBNode();               // key=0, BLACK
        NIL->left = NIL->right = NIL->parent = NIL;
        root = NIL;
    }

    ~RBTree() { clear(root); delete NIL; }

    bool empty() const { return root == NIL; }
```

---

## 2. 좌/우 회전(rotateLeft/rotateRight)

회전은 **부모/자식 포인터**를 빠짐없이 재연결해야 한다.

### 좌회전 (x-y)

```
x                y
 \              / \
  y    →       x   γ
 / \            \
β   γ            β
```

```cpp
    void rotateLeft(RBNode* x) {
        RBNode* y = x->right;
        x->right = y->left;
        if (y->left != NIL) y->left->parent = x;
        y->parent = x->parent;

        if (x->parent == NIL) root = y;
        else if (x == x->parent->left) x->parent->left = y;
        else x->parent->right = y;

        y->left = x;
        x->parent = y;
    }

    void rotateRight(RBNode* y) {
        RBNode* x = y->left;
        y->left = x->right;
        if (x->right != NIL) x->right->parent = y;
        x->parent = y->parent;

        if (y->parent == NIL) root = x;
        else if (y == y->parent->left) y->parent->left = x;
        else y->parent->right = x;

        x->right = y;
        y->parent = x;
    }
```

---

## 3. 삽입(Insert) — BST 삽입 → Fix-up

1) **BST 규칙**으로 RED 노드 삽입
2) **부모가 RED**면 R4 위반 → **삼촌 색**에 따라 **재색칠/회전**

### 삽입 Fix-up 3케이스 (좌측 기준, 우측은 대칭)

- **Case 1 (삼촌 RED)**: 부모/삼촌을 BLACK, 조부모를 RED로 칠하고, **조부모로 올라가** 반복
- **Case 2 (삼촌 BLACK, z가 부모의 오른쪽)**: **좌회전**으로 Case 3으로 변형
- **Case 3 (삼촌 BLACK, z가 부모의 왼쪽)**: **우회전** 후, 새 부모를 BLACK, 조부모를 RED

```
      (gp)B                   (gp)B                     (gp)B
      /    \                  /    \                    /    \
   (p)R    (u)R   C1→     (p)R    (u)R   C2→       (p)R    (u)R
    /                       \                         /
  (z)R                       (z)R                   (z)R

C1: recolor p,u black; gp red; z←gp
C2: rotateLeft(p): z가 삼각형이면 line으로
C3: rotateRight(gp); 색 재조정 (새 root 후보 black)
```

### 코드: BST 삽입 + Fix-up

```cpp
    RBNode* bstInsert(int key) {
        RBNode* z = new RBNode(key, RED, NIL, NIL, NIL);
        RBNode* y = NIL;
        RBNode* x = root;
        while (x != NIL) {
            y = x;
            if (key == x->key) { delete z; return NIL; } // 중복 금지
            x = (key < x->key) ? x->left : x->right;
        }
        z->parent = y;
        if (y == NIL) root = z;
        else if (key < y->key) y->left = z;
        else y->right = z;
        return z;
    }

    void insert(int key) {
        RBNode* z = bstInsert(key);
        if (z == NIL) return; // dup

        while (z->parent->color == RED) {
            RBNode* gp = z->parent->parent;
            if (z->parent == gp->left) {
                RBNode* u = gp->right; // uncle
                if (u->color == RED) {                 // Case 1
                    z->parent->color = BLACK;
                    u->color = BLACK;
                    gp->color = RED;
                    z = gp;
                } else {
                    if (z == z->parent->right) {       // Case 2
                        z = z->parent;
                        rotateLeft(z);
                    }                                   // Case 3
                    z->parent->color = BLACK;
                    gp->color = RED;
                    rotateRight(gp);
                }
            } else {
                // symmetric (parent is right child)
                RBNode* u = gp->left;
                if (u->color == RED) {
                    z->parent->color = BLACK;
                    u->color = BLACK;
                    gp->color = RED;
                    z = gp;
                } else {
                    if (z == z->parent->left) {
                        z = z->parent;
                        rotateRight(z);
                    }
                    z->parent->color = BLACK;
                    gp->color = RED;
                    rotateLeft(gp);
                }
            }
        }
        root->color = BLACK; // R2
    }
```

---

## 4. 삭제(Delete) — BST 삭제 → Double-Black Fix-up

### 개요
1) **BST 삭제**: (0/1/2자식) — 2자식은 **후속자(successor)**로 치환
2) **삭제된 BLACK**이 경로 검정 수를 줄이면 **Double-Black** 문제 → **형제(s) 색/자식 색**에 따라 **회전/재색칠**

### 삭제 보정 4케이스(좌측 기준, 우측 대칭)

> `x` = double-black이 발생한 위치(혹은 NIL), `s` = 형제

| Case | 조건 | 조치 |
|---|---|---|
| C1 | `s` RED | `s`를 BLACK, 부모를 RED; **부모 기준 회전** → BLACK 형제 케이스로 변환 |
| C2 | `s` BLACK, `s`의 두 자식 모두 BLACK | `s`를 RED; `x`를 부모로 끌어올려 반복 (상위로 DB 전파) |
| C3 | `s` BLACK, `s`의 **가까운** 자식 RED, **먼** 자식 BLACK | `s` 기준 **내측 회전**으로 C4로 변환 (형제/자식 색 재배치) |
| C4 | `s` BLACK, `s`의 **먼** 자식 RED | 부모 색을 `s`로 이관, 부모 BLACK, `s` BLACK, **부모 기준 회전**, `x`의 DB 해소 |

ASCII(좌측 기준):

```
C1: p?B            p?B                 p?B
    / \            / \                 / \
   x  sR   →      x  sB   rotateLeft  x   ...
      / \            / \
     a   b          a   b

C2: sB의 자식 둘 다 B → s를 R로, x↑p로 전파 (p가 R이면 pB로 끝)

C3: sB, s의 "가까운" 자식(왼쪽)이 R → rotateRight(s)로 C4로 정규화

C4: sB, s의 "먼" 자식(오른쪽)이 R → rotateLeft(p) + 재색칠로 종료
```

### 삭제 구현: `transplant`, `deleteFixup`

```cpp
    void transplant(RBNode* u, RBNode* v) {
        if (u->parent == NIL) root = v;
        else if (u == u->parent->left) u->parent->left = v;
        else u->parent->right = v;
        v->parent = u->parent;
    }

    RBNode* minNode(RBNode* x) {
        while (x->left != NIL) x = x->left;
        return x;
    }

    void erase(int key) {
        RBNode* z = root;
        while (z != NIL && z->key != key)
            z = (key < z->key) ? z->left : z->right;
        if (z == NIL) return; // not found

        RBNode* y = z;          // to be removed/ moved
        Color yOrig = y->color;
        RBNode* x;              // child moved into y's original position

        if (z->left == NIL) {
            x = z->right;
            transplant(z, z->right);
        } else if (z->right == NIL) {
            x = z->left;
            transplant(z, z->left);
        } else {
            y = minNode(z->right);      // successor
            yOrig = y->color;
            x = y->right;
            if (y->parent == z) {
                x->parent = y;
            } else {
                transplant(y, y->right);
                y->right = z->right; y->right->parent = y;
            }
            transplant(z, y);
            y->left = z->left;  y->left->parent = y;
            y->color = z->color;
        }
        delete z;

        if (yOrig == BLACK) deleteFixup(x);
    }

    void deleteFixup(RBNode* x) {
        while (x != root && x->color == BLACK) {
            if (x == x->parent->left) {
                RBNode* s = x->parent->right;
                if (s->color == RED) {                    // C1
                    s->color = BLACK;
                    x->parent->color = RED;
                    rotateLeft(x->parent);
                    s = x->parent->right;
                }
                if (s->left->color == BLACK && s->right->color == BLACK) { // C2
                    s->color = RED;
                    x = x->parent;
                } else {
                    if (s->right->color == BLACK) {       // C3
                        s->left->color = BLACK;
                        s->color = RED;
                        rotateRight(s);
                        s = x->parent->right;
                    }
                    // C4
                    s->color = x->parent->color;
                    x->parent->color = BLACK;
                    s->right->color = BLACK;
                    rotateLeft(x->parent);
                    x = root;
                }
            } else {
                // symmetric (x is right child)
                RBNode* s = x->parent->left;
                if (s->color == RED) {
                    s->color = BLACK;
                    x->parent->color = RED;
                    rotateRight(x->parent);
                    s = x->parent->left;
                }
                if (s->right->color == BLACK && s->left->color == BLACK) {
                    s->color = RED;
                    x = x->parent;
                } else {
                    if (s->left->color == BLACK) {
                        s->right->color = BLACK;
                        s->color = RED;
                        rotateLeft(s);
                        s = x->parent->left;
                    }
                    s->color = x->parent->color;
                    x->parent->color = BLACK;
                    s->left->color = BLACK;
                    rotateRight(x->parent);
                    x = root;
                }
            }
        }
        x->color = BLACK;
    }
```

> **포인트**: NIL은 **BLACK**이므로 `NIL->color == BLACK` 가정이 케이스 분기에서 자연스럽게 동작한다.

---

## 5. 순회/검색/유틸(검증 포함)

```cpp
    RBNode* find(int key) const {
        RBNode* cur = root;
        while (cur != NIL && cur->key != key)
            cur = (key < cur->key) ? cur->left : cur->right;
        return (cur == NIL) ? nullptr : cur;
    }

    RBNode* lower_bound(int key) const { // 첫 >= key
        RBNode* cur = root; RBNode* ans = NIL;
        while (cur != NIL) {
            if (cur->key >= key) { ans = cur; cur = cur->left; }
            else cur = cur->right;
        }
        return (ans==NIL)? nullptr : ans;
    }

    RBNode* floorKey(int key) const { // 첫 <= key
        RBNode* cur = root; RBNode* ans = NIL;
        while (cur != NIL) {
            if (cur->key == key) return cur;
            if (cur->key < key) { ans = cur; cur = cur->right; }
            else cur = cur->left;
        }
        return (ans==NIL)? nullptr : ans;
    }

    void inorderPrint() const { inorder(root); cout << "\n"; }
    void inorder(RBNode* n) const {
        if (n == NIL) return;
        inorder(n->left);
        cout << n->key << (n->color==RED?'R':'B') << ' ';
        inorder(n->right);
    }

    void clear(RBNode* n) {
        if (n == NIL) return;
        clear(n->left); clear(n->right);
        delete n;
    }
};
```

---

## 6. 검증: 레드-블랙 불변식 검사

- R4: 어떤 RED도 RED 자식을 가지면 안 됨
- R5: 모든 경로의 black-height 동일

```cpp
// ===== rbcheck.hpp =====
#pragma once
#include "rbtree.hpp"

struct RBChecker {
    const RBTree& T;
    RBChecker(const RBTree& t): T(t) {}

    bool checkRootBlack() const { return T.root == T.NIL || T.root->color == BLACK; }

    bool checkNoRedRed(RBNode* n) const {
        if (n == T.NIL) return true;
        if (n->color == RED) {
            if (n->left->color == RED || n->right->color == RED) return false;
        }
        return checkNoRedRed(n->left) && checkNoRedRed(n->right);
    }

    // 반환: (isValid, blackHeight)
    pair<bool,int> checkBlackHeight(RBNode* n) const {
        if (n == T.NIL) return {true, 1}; // NIL은 black-height 1로 셈(관례)
        auto L = checkBlackHeight(n->left);
        auto R = checkBlackHeight(n->right);
        if (!L.first || !R.first || L.second != R.second) return {false, 0};
        int add = (n->color == BLACK) ? 1 : 0;
        return {true, L.second + add};
    }

    bool run() const {
        if (!checkRootBlack()) return false;
        if (!checkNoRedRed(T.root)) return false;
        return checkBlackHeight(T.root).first;
    }
};
```

> **주의**: black-height의 정의에서 NIL을 1로 볼지 0으로 볼지는 관례 문제. 여기선 **NIL=1**로 두었다. (일관성 유지가 핵심)

---

## 7. 전체 사용 예시

```cpp
// ===== main.cpp =====
#include "rbtree.hpp"
#include "rbcheck.hpp"

int main() {
    ios::sync_with_stdio(false); cin.tie(nullptr);

    RBTree t;
    for (int x : {10,20,30,15,25,5,1,17,16,19}) t.insert(x);

    cout << "[삽입 후 중위] "; t.inorderPrint();
    auto lb = t.lower_bound(18);
    cout << "lower_bound(18) = " << (lb ? to_string(lb->key) : string("null")) << "\n";

    RBChecker chk(t);
    cout << "RB valid? " << (chk.run() ? "true" : "false") << "\n\n";

    for (int del : {15,10,1,19,16}) {
        t.erase(del);
        cout << "[삭제 " << del << " 후] "; t.inorderPrint();
        cout << "RB valid? " << (chk.run() ? "true" : "false") << "\n";
    }
}
```

**출력 예시(형식 예)**

```
[삽입 후 중위] 1R 5B 10R 15B 16R 17B 19R 20B 25B 30B
lower_bound(18) = 19
RB valid? true

[삭제 15 후] 1R 5B 10R 16B 17B 19R 20B 25B 30B
RB valid? true
[삭제 10 후] 1R 5B 16B 17B 19R 20B 25B 30B
RB valid? true
...
```

> 색상은 디버깅 가독성용. 실제 출력은 구현에 따라 달라질 수 있다.

---

## 8. 삽입/삭제 케이스 상세 ASCII

### 삽입 — Case 1 (삼촌 RED)
```
   gp(B)           gp(R)
   /   \    →     /   \
 p(R)  u(R)     p(B)  u(B)
 /                \
z(R)               z(R)
```
- 해결: `p,u` BLACK, `gp` RED, `z ← gp`로 승격(루트면 종료), 루트는 마지막에 BLACK.

### 삽입 — Case 2 → 3 변형
```
Case2:
   gp(B)                gp(B)
   /   \                /   \
 p(R)  u(B)   z=p->R  p(R)  u(B)
   \       →          /
   z(R)              z(R)
        rotateLeft(p)

Case3:
   gp(B)                  p(B)
   /   \       →         /   \
 p(R)  u(B)            z(R)  gp(R)
 /                            /
z(R)                         (..)
        rotateRight(gp) + 색변경
```

### 삭제 — Case 1 (형제 RED → 회전으로 C2~C4로 정규화)
```
    p(?)               s(B)
   /   \     →        /   \
  x(B) s(R)          p(R)  γ
      / \            / \
     α  γ          x(B) α
rotateLeft(p)  (좌측 기준)
```

### 삭제 — Case 2 (형제/자식이 모두 BLACK → 재색칠로 DB 상향)
```
    p(C)            p'(C?)
   /   \    →      /     \
  x(DB) s(B)     x(B)   s(R)
      /  \             /   \
     α(B)β(B)       α(B)  β(B)
```

### 삭제 — Case 3 → 4 (가까운 RED를 먼 RED로 바꾸는 사전 회전)
```
C3:
   p(?)                p(?)
  /   \               /   \
 x   s(B)   →       x    α(B)
    /  \                 \
  α(R) β(B)               s(R)
          rotateRight(s)

C4:
   p(C)                     s(C)
  /   \         →          /   \
 x   s(B)                p(B)  β(B)
     /  \               /  \
   α(B) β(R)          x(B) α(B)
           rotateLeft(p) + 색조정
```

---

## 9. 수학 스냅샷 — 높이 상계 \(h \le 2\log_2(n+1)\)

- 임의 노드의 left/right 중 최소 한 쪽에는 **적어도** half의 black-height가 존재한다.
- R4로 인해 RED는 **연속될 수 없으므로**, 루트→리프 경로에서 **RED는 최대 BLACK 수만큼** 낄 수 있다.
- 즉, 경로 길이 \(h \le 2 \cdot bh(\text{root})\).
- 한편, 크기 \(n\)인 RB-Tree의 최소 노드 수는 black-height \(bh\)에 대해 \(n \ge 2^{bh}-1\) 정도로 하한이 성립한다(귀납/압축 논증).
  따라서
  $$
  bh \le \log_2(n+1) \quad\Rightarrow\quad
  h \le 2\,bh \le 2\log_2(n+1).
  $$
- 결론: **항상 \(h = O(\log n)\)**. 탐색/삽입/삭제 모두 \(O(\log n)\).

---

## 10. 직렬화/역직렬화 (전위 + NIL 토큰)

디버깅/테스트에 유용하다. (색상까지 포함)

```cpp
// 전위 순회 직렬화: key,color 혹은 # (NIL)
void serializeRB(const RBTree& T, RBNode* n, ostream& os) {
    if (n == T.NIL) { os << "# "; return; }
    os << n->key << ":" << (n->color==RED?'R':'B') << " ";
    serializeRB(T, n->left, os);
    serializeRB(T, n->right, os);
}

RBNode* deserializeRB_core(RBTree& T, istringstream& iss) {
    string tok; if (!(iss >> tok)) return T.NIL;
    if (tok == "#") return T.NIL;
    auto pos = tok.find(':');
    int key = stoi(tok.substr(0, pos));
    char c = tok[pos+1];

    RBNode* n = new RBNode(key, (c=='R'?RED:BLACK), T.NIL, T.NIL, T.NIL);
    n->left  = deserializeRB_core(T, iss); if (n->left  != T.NIL) n->left->parent  = n;
    n->right = deserializeRB_core(T, iss); if (n->right != T.NIL) n->right->parent = n;
    return n;
}
// 사용 시: 색 불변식이 깨질 수 있으므로 테스트 용도에 한정(검증기로 점검)
```

---

## 11. 실전 팁/버그 포인트

1. **NIL 센티넬**을 **BLACK**으로 유지, 모든 leaf/child가 NIL을 가리키게 하라.
2. **회전 시 부모 포인터** 업데이트 누락 금지. `root`/`parent->left/right` 재연결 주의.
3. **삽입 Fix-up**는 루트 도달 전까지 **부모 RED** 조건으로만 반복.
4. **삭제 Fix-up**는 `x`가 루트가 되거나 BLACK이 될 때까지 반복. `s` 재평가 필요.
5. **대칭 처리**(좌/우)는 꼭 **복붙 후 필드 좌우를 바꾸는** 방식으로 실수 줄이기.
6. **검증기**(R4 + black-height)로 단계별 테스트. 퍼징에서는 `std::set`과 **정렬 결과/원소 존재**를 계속 비교.
7. **중복 정책**을 초기에 확정(< / <=). 여기서는 **중복 금지**.

---

## 12. 범위/순서 쿼리

RB-Tree는 기본적으로 **순서 통계**가 없다. `size`를 보강하면 가능(AVL과 동일 아이디어).

- 범위 출력: 중위 순회에서 `L ≤ key ≤ R`만 출력
- `lower_bound(x)`, `floor(x)`, `ceil(x)`는 위 유틸 참조
- k-th element: 각 노드에 `sz` 보강(본 글에서는 생략)

---

## 13. 퍼징 테스트 스케치

```cpp
// 의사코드
RBTree T; multiset<int> M;
for (int step=0; step<100000; ++step) {
    int op = rng()%3;
    int x  = rng()%10000;
    if (op==0) { T.insert(x); M.insert(x); }
    else if (op==1) { T.erase(x); auto it=M.find(x); if(it!=M.end()) M.erase(it); }
    else {
        vector<int> a; // inorder 수집
        // T.inorderCollect(a) 구현해서 수집 (NIL 제외)
        vector<int> b(M.begin(), M.end());
        assert(a==b);
        RBChecker chk(T); assert(chk.run());
    }
}
```

---

## 14. AVL vs Red-Black — 선택 기준

| 항목 | Red-Black Tree | AVL Tree |
|---|---|---|
| 균형 강도 | 느슨(색 규칙) | 강함(높이 차 ≤ 1) |
| 탐색 평균 | 약간 불리 | 유리 (높이 더 낮음) |
| 삽입/삭제 회전 | **적음** (삽·삭 모두 평균 0~2회) | 삽입/삭제 시 회전 많을 수 있음 |
| 삭제 난이도 | **복잡**(Double-Black 케이스多) | 비교적 단순 |
| 쓰임새 | 표준 컨테이너, 런타임 맵 | 인덱싱/읽기 성능 중시 |
| 높이 상계 | \(h \le 2\log_2(n+1)\) | \(h \approx 1.44\log_2 n\) 근사 |

**가이드**
- **갱신(삽/삭)이 매우 빈번** → **Red-Black** 권장
- **탐색(look-up) 비중↑, 읽기 성능 극대화** → **AVL** 고려
- 범용/표준 라이브러리 호환성 → **Red-Black**(사실상 디폴트)

---

## 15. 요약

- 레드-블랙 트리는 **색 규칙(5규칙)**과 회전으로 **로그 높이**를 보장한다.
- **삽입**: `부모 RED`일 때 삼촌 색에 따라 **재색칠/회전**(Case 1-3).
- **삭제**: **Double-Black**을 **형제/자식 색**으로 해소(Case 1-4).
- **NIL 센티넬**을 적극 활용하면 구현이 깔끔해진다.
- 검증기(R4, black-height)와 퍼징으로 **신뢰성**을 높이고, 실전에서는 `lower_bound`/`floor/ceil`/직렬화 등 **도구화**가 필수다.
- 선택 기준: **갱신 빈번 → RB**, **탐색 성능 극대화 → AVL**.

---
```cpp
// ===== 단일 파일 버전: rbtree_full.cpp =====
// 컴파일: g++ -std=c++17 -O2 rbtree_full.cpp && ./a.out
#include <bits/stdc++.h>
using namespace std;

enum Color : uint8_t { RED=0, BLACK=1 };

struct RBNode {
    int key; Color color;
    RBNode *left, *right, *parent;
    RBNode(int k=0, Color c=BLACK, RBNode* l=nullptr, RBNode* r=nullptr, RBNode* p=nullptr)
        : key(k), color(c), left(l), right(r), parent(p) {}
};

struct RBTree {
    RBNode *root, *NIL;
    RBTree() {
        NIL = new RBNode(); NIL->left=NIL->right=NIL->parent=NIL; NIL->color = BLACK;
        root = NIL;
    }
    ~RBTree(){ clear(root); delete NIL; }

    void clear(RBNode* n){ if(n==NIL) return; clear(n->left); clear(n->right); delete n; }

    void rotateLeft(RBNode* x){
        RBNode* y = x->right;
        x->right = y->left; if(y->left!=NIL) y->left->parent = x;
        y->parent = x->parent;
        if(x->parent==NIL) root=y;
        else if(x==x->parent->left) x->parent->left=y;
        else x->parent->right=y;
        y->left=x; x->parent=y;
    }
    void rotateRight(RBNode* y){
        RBNode* x = y->left;
        y->left = x->right; if(x->right!=NIL) x->right->parent = y;
        x->parent = y->parent;
        if(y->parent==NIL) root=x;
        else if(y==y->parent->left) y->parent->left=x;
        else y->parent->right=x;
        x->right=y; y->parent=x;
    }

    RBNode* bstInsert(int key){
        RBNode* z = new RBNode(key, RED, NIL, NIL, NIL);
        RBNode* y=NIL; RBNode* x=root;
        while(x!=NIL){
            y=x;
            if(key==x->key){ delete z; return NIL; }
            x = (key<x->key)? x->left : x->right;
        }
        z->parent = y;
        if(y==NIL) root=z;
        else if(key<y->key) y->left=z;
        else y->right=z;
        return z;
    }

    void insert(int key){
        RBNode* z = bstInsert(key);
        if(z==NIL) return;
        while(z->parent->color==RED){
            RBNode* gp = z->parent->parent;
            if(z->parent==gp->left){
                RBNode* u = gp->right;
                if(u->color==RED){ // C1
                    z->parent->color=BLACK; u->color=BLACK; gp->color=RED; z=gp;
                }else{
                    if(z==z->parent->right){ // C2
                        z=z->parent; rotateLeft(z);
                    }
                    z->parent->color=BLACK; gp->color=RED; rotateRight(gp); // C3
                }
            }else{
                RBNode* u = gp->left;
                if(u->color==RED){
                    z->parent->color=BLACK; u->color=BLACK; gp->color=RED; z=gp;
                }else{
                    if(z==z->parent->left){
                        z=z->parent; rotateRight(z);
                    }
                    z->parent->color=BLACK; gp->color=RED; rotateLeft(gp);
                }
            }
        }
        root->color=BLACK;
    }

    void transplant(RBNode* u, RBNode* v){
        if(u->parent==NIL) root=v;
        else if(u==u->parent->left) u->parent->left=v;
        else u->parent->right=v;
        v->parent = u->parent;
    }
    RBNode* minNode(RBNode* x){ while(x->left!=NIL) x=x->left; return x; }

    void erase(int key){
        RBNode* z=root;
        while(z!=NIL && z->key!=key) z = (key<z->key)? z->left : z->right;
        if(z==NIL) return;

        RBNode *y=z, *x; Color yOrig=y->color;
        if(z->left==NIL){
            x = z->right; transplant(z,z->right);
        }else if(z->right==NIL){
            x = z->left; transplant(z,z->left);
        }else{
            y = minNode(z->right);
            yOrig = y->color;
            x = y->right;
            if(y->parent==z){ x->parent=y; }
            else{
                transplant(y, y->right);
                y->right = z->right; y->right->parent = y;
            }
            transplant(z, y);
            y->left = z->left; y->left->parent = y;
            y->color = z->color;
        }
        delete z;
        if(yOrig==BLACK) deleteFixup(x);
    }
    void deleteFixup(RBNode* x){
        while(x!=root && x->color==BLACK){
            if(x==x->parent->left){
                RBNode* s = x->parent->right;
                if(s->color==RED){ // C1
                    s->color=BLACK; x->parent->color=RED; rotateLeft(x->parent);
                    s = x->parent->right;
                }
                if(s->left->color==BLACK && s->right->color==BLACK){ // C2
                    s->color=RED; x = x->parent;
                }else{
                    if(s->right->color==BLACK){ // C3
                        s->left->color=BLACK; s->color=RED; rotateRight(s);
                        s = x->parent->right;
                    }
                    // C4
                    s->color = x->parent->color;
                    x->parent->color=BLACK; s->right->color=BLACK;
                    rotateLeft(x->parent);
                    x=root;
                }
            }else{
                RBNode* s = x->parent->left;
                if(s->color==RED){
                    s->color=BLACK; x->parent->color=RED; rotateRight(x->parent);
                    s = x->parent->left;
                }
                if(s->right->color==BLACK && s->left->color==BLACK){
                    s->color=RED; x=x->parent;
                }else{
                    if(s->left->color==BLACK){
                        s->right->color=BLACK; s->color=RED; rotateLeft(s);
                        s = x->parent->left;
                    }
                    s->color = x->parent->color;
                    x->parent->color=BLACK; s->left->color=BLACK;
                    rotateRight(x->parent);
                    x=root;
                }
            }
        }
        x->color=BLACK;
    }

    RBNode* find(int key) const {
        RBNode* cur=root; while(cur!=NIL && cur->key!=key)
            cur = (key<cur->key)? cur->left:cur->right;
        return (cur==NIL)? nullptr : cur;
    }
    RBNode* lower_bound(int key) const {
        RBNode* cur=root; RBNode* ans=(RBNode*)NIL;
        while(cur!=NIL){
            if(cur->key>=key){ ans=cur; cur=cur->left; }
            else cur=cur->right;
        }
        return (ans==(RBNode*)NIL)? nullptr: ans;
    }
    void inorderPrint() const { inorder(root); cout<<"\n"; }
    void inorder(RBNode* n) const {
        if(n==NIL) return;
        inorder(n->left);
        cout<<n->key<<(n->color==RED?'R':'B')<<" ";
        inorder(n->right);
    }
};

struct RBChecker {
    const RBTree& T;
    RBChecker(const RBTree& t): T(t) {}
    bool rootBlack() const { return T.root==T.NIL || T.root->color==BLACK; }
    bool noRedRed(RBNode* n) const {
        if(n==T.NIL) return true;
        if(n->color==RED)
            if(n->left->color==RED || n->right->color==RED) return false;
        return noRedRed(n->left) && noRedRed(n->right);
    }
    pair<bool,int> blackHeight(RBNode* n) const {
        if(n==T.NIL) return {true,1};
        auto L = blackHeight(n->left);
        auto R = blackHeight(n->right);
        if(!L.first || !R.first || L.second!=R.second) return {false,0};
        int add = (n->color==BLACK)?1:0;
        return {true, L.second+add};
    }
    bool run() const { if(!rootBlack()) return false; if(!noRedRed(T.root)) return false; return blackHeight(T.root).first; }
};

int main(){
    RBTree t;
    for(int x: {10,20,30,15,25,5,1,17,16,19}) t.insert(x);
    cout<<"[ins] "; t.inorderPrint();

    RBChecker chk(t);
    cout<<"valid? "<<(chk.run()?"true":"false")<<"\n";

    for(int del: {15,10,1,19,16}){
        t.erase(del);
        cout<<"[del "<<del<<"] "; t.inorderPrint();
        cout<<"valid? "<<(chk.run()?"true":"false")<<"\n";
    }
}
```
```

---

## 부록 A) 자주 묻는 질문(FAQ)

- **Q. 삽입 Fix-up에서 왜 루트는 마지막에 BLACK으로?**
  A. Case 1 전파로 `z`가 루트까지 올라갈 수 있다. 루트는 R2로 반드시 BLACK이어야 하므로 마지막에 색을 고정한다.

- **Q. black-height에서 NIL을 0/1 중 무엇으로 세나요?**
  A. 관례 차이. 이 글은 **NIL=1**로 취급했다. 다만 구현/증명 내 **일관성**이 중요.

- **Q. `std::map`의 삭제가 생각보다 빠른 이유?**
  A. RB 트리는 **삭제 회전 수가 적고**, 실전 워크로드에서 평균 **색 재도색으로 대부분 해결**되는 경우가 많다.

---

## 부록 B) 더 읽을거리(키워드)

- “J. S. Tarjan”의 균형 BST 노트, CLRS(레드-블랙 장)
- Order-Statistic Tree(레드-블랙에 `size` 보강), Interval Tree
- B-Tree/B+-Tree: 디스크 친화적 범용 색인(데이터베이스/스토리지)
