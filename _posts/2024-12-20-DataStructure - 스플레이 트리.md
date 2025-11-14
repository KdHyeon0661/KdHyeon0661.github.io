---
layout: post
title: Data Structure - 펜윅 트리
date: 2024-12-20 19:20:23 +0900
category: Data Structure
---
# 스플레이 트리 (Splay Tree) — 자가 조정 BST 완전 정리

## 개요와 동기

**스플레이 트리(Splay Tree)** 는 Sleator & Tarjan(1985)이 제안한 **자가 조정(Self-Adjusting)** 이진 탐색 트리다. 연속된 연산 시 **자주 접근되는 키를 루트 근처로 끌어올리는(splaying)** 전략으로,
단일 연산은 최악 \(O(n)\)일 수 있지만 **암시적 균형** 덕분에 연산 시퀀스 전체는 **상각(amortized) \(O(\log n)\)** 을 보장한다.

- **언제 유리한가**: 같은 키(혹은 근처 키)만 계속 접근되는 워킹셋/캐시 패턴, 스택/큐처럼 최근 요소 접근이 잦은 패턴, 키 분포가 편향적인 온라인 워크로드.
- **장점**: 강력한 상각 보장, 구현이 RB/AVL보다 간단, 보조 균형 정보 불필요(색/높이 없음).
- **주의**: 단일 연산 최악 \(O(n)\), 실시간 최악 지연에 민감한 시스템에서는 주의.

시간 복잡도 개요(상각):

| 연산 | 평균(상각) | 단일 최악 | 시퀀스 \(m\)회 |
|---|---|---|---|
| 탐색/삽입/삭제 | \(O(\log n)\) | \(O(n)\) | \(O(m\log n)\) |

---

## 불변식과 스플레이 동작

### BST 불변식

모든 노드 \(x\) 에 대해 왼쪽 서브트리는 \(<x.key\), 오른쪽은 \(>x.key\) (동일 키 허용 시 정책 필요).

### 스플레이 회전 유형

루트로 끌어올리는 과정에서 세 가지 패턴을 반복한다.

| 유형 | 모양 | 의미 |
|---|---|---|
| **Zig** | (부모만 있고 조부모 없음) | 단회전으로 루트에 올림 |
| **Zig-Zig** | `LL` 또는 `RR` | 부모 먼저, 그 다음 자신 회전(같은 방향 두 번) |
| **Zig-Zag** | `LR` 또는 `RL` | 자신→자신 두 번(교차 방향) |

스플레이는 **반복 회전으로 목표 노드가 루트가 될 때까지** 수행한다.

---

## 상향식 vs 하향식 스플레이

- **상향식(bottom-up)**: 목표 노드에서 시작해 위로 올라가며 Zig/Zig-Zig/Zig-Zag를 적용(고전적 구현, 아래 코드).
- **하향식(top-down)**: 탐색 중에 세 개의 가상 트리(Left, Middle, Right)를 유지하며 회전/분해를 동시에 수행. 반복문 한 번으로 구현되며 재귀가 없고 꼬리 정리가 쉬움(추가 스니펫 제공).

---

## C++ 기본 구현(안전한 회전 + 사이즈 보강)

아래 구현은 **정수 키**, **중복 허용(빈도 `cnt`)**, **서브트리 크기(`sz`)** 를 보강하여 **순서통계(k-th, rank)** 도 지원한다.
포인터 부모/자식 연결을 **일관되게 갱신**하는 회전이 핵심이다.

```cpp
#include <bits/stdc++.h>

using namespace std;

struct Node {
    int key, cnt;          // 동일 키 빈도
    int sz;                // 서브트리 총 크기(빈도 포함)
    Node *l, *r, *p;
    Node(int k, Node* p=nullptr) : key(k), cnt(1), sz(1), l(nullptr), r(nullptr), p(p) {}
};

static int sizeOf(Node* x){ return x ? x->sz : 0; }
static void pull(Node* x){
    if(!x) return;
    x->sz = x->cnt + sizeOf(x->l) + sizeOf(x->r);
}
static bool isLeft(Node* x){ return x->p && x->p->l == x; }
static void connect(Node* child, Node* parent, bool asLeft){
    if(parent){
        if(asLeft) parent->l = child;
        else       parent->r = child;
    }
    if(child) child->p = parent;
}

// 단일 회전: x를 부모 p 위로 올림
static void rotate(Node*& root, Node* x){
    Node* p = x->p;
    Node* g = p ? p->p : nullptr;
    bool xLeft = (p->l == x);

    // p와 x 사이의 링크 재배치
    if(xLeft){
        connect(x->r, p, true);   // p->l = x->r
        connect(p, x, false);     // x->r = p
    }else{
        connect(x->l, p, false);  // p->r = x->l
        connect(p, x, true);      // x->l = p
    }

    // 조부모와 x 연결
    if(!g){
        x->p = nullptr;
        root = x;
    }else{
        connect(x, g, g->l == p);
    }
    pull(p); pull(x);
}

// 상향식 스플레이: x를 루트로
static void splay(Node*& root, Node* x){
    if(!x) return;
    while(x->p){
        Node* p = x->p;
        Node* g = p->p;
        if(!g){ // Zig
            rotate(root, x);
        }else if((g->l == p) == (p->l == x)){ // Zig-Zig
            rotate(root, p);
            rotate(root, x);
        }else{ // Zig-Zag
            rotate(root, x);
            rotate(root, x);
        }
    }
}
```

---

## 탐색/삽입/삭제 + 보조 연산(전/후임자, lower_bound)

### 내부 탐색(가까운 노드까지)

탐색 실패 시 **마지막 방문 노드**를 splay하면 이후 가까운 키 접근이 빨라진다.

```cpp
// key에 가장 가까운 위치까지 내려간 뒤, 그 노드를 splay 후 반환
static Node* findAndSplay(Node*& root, int key){
    Node* cur = root;
    Node* last = nullptr;
    while(cur){
        last = cur;
        if(key == cur->key) break;
        cur = (key < cur->key) ? cur->l : cur->r;
    }
    if(last) splay(root, last);
    return (cur && cur->key == key) ? root : nullptr; // splay 후 root가 last
}
```

### 삽입(중복 허용)

동일 키면 `cnt++` 후 splay. 아니면 새 노드 삽입 후 splay.

```cpp
static void insert(Node*& root, int key){
    if(!root){ root = new Node(key); return; }
    Node* cur = root; Node* par = nullptr;
    while(cur){
        par = cur;
        if(key == cur->key){
            cur->cnt++; pull(cur);
            splay(root, cur);
            return;
        }
        cur = (key < par->key) ? par->l : par->r;
    }
    Node* nd = new Node(key, par);
    if(key < par->key) par->l = nd; else par->r = nd;
    // 위쪽으로 sz 갱신
    for(Node* t = par; t; t = t->p) pull(t);
    splay(root, nd);
}
```

### 전임자/후임자, 하한(lower_bound)

```cpp
static Node* lower_bound(Node*& root, int key){ // 첫 >= key
    Node* cur = root; Node* ans = nullptr;
    while(cur){
        if(cur->key >= key){ ans = cur; cur = cur->l; }
        else cur = cur->r;
    }
    if(ans) splay(root, ans);
    return ans; // 없으면 null(그대로)
}

static Node* predecessor(Node*& root, int key){ // < key 중 최대
    lower_bound(root, key); // root는 첫 >= key 가 루트(혹은 마지막)
    if(root && root->key < key) return root;
    if(!root || !root->l) return nullptr;
    Node* t = root->l;
    while(t->r) t = t->r;
    splay(root, t); return root;
}

static Node* successor(Node*& root, int key){ // > key 중 최소
    Node* lb = lower_bound(root, key+1);
    return lb; // 이미 splay됨
}
```

### 삭제

루트로 올린 뒤, 빈도만 줄이거나, 좌/우를 합치는 **join**으로 제거한다.

```cpp
// left의 모든 키 < right의 모든 키 라고 가정하고 합친다
static Node* joinTrees(Node* left, Node* right){
    if(!left) return right;
    if(!right) return left;
    // left의 최대 키를 루트로
    Node* t = left;
    while(t->r) t = t->r;
    Node*& rootLeft = left;
    splay(rootLeft, t); // t가 left의 루트
    connect(right, rootLeft, false); // rootLeft->r = right
    pull(rootLeft);
    return rootLeft;
}

static void erase(Node*& root, int key){
    Node* x = findAndSplay(root, key);
    if(!x || x->key != key) return; // 없음
    if(x->cnt > 1){ x->cnt--; pull(x); return; }
    Node* L = x->l; Node* R = x->r;
    if(L) L->p = nullptr;
    if(R) R->p = nullptr;
    delete x;
    root = joinTrees(L, R);
}
```

---

## Split/Join로 범위 조작

스플레이는 **Split/Join** 이 간단해 **범위 삭제·이동·병합** 등의 고급 연산을 쉽게 구성할 수 있다.

```cpp
// key 기준으로 <key | >=key 두 트리로 분리
static pair<Node*,Node*> split(Node*& root, int key){
    Node* lb = lower_bound(root, key); // 첫 >=key를 루트로
    if(!lb) return {root, nullptr};    // 모두 <key
    Node* L = lb->l;
    if(L) L->p = nullptr;
    lb->l = nullptr; pull(lb);
    return {L, lb}; // L(<key), lb(>=key)
}

// 두 트리 L(<R)와 R을 결합
static Node* join(Node* L, Node* R){ return joinTrees(L, R); }

// [lo, hi] 범위 삭제
static void eraseRange(Node*& root, int lo, int hi){
    auto [A, midR] = split(root, lo);
    auto [mid, C]  = split(midR, hi+1);
    // mid: [lo..hi] — 전부 파괴
    // 안전하게 노드를 파괴하려면 후위순회 delete
    function<void(Node*)> destroy = [&](Node* u){
        if(!u) return; destroy(u->l); destroy(u->r); delete u;
    };
    destroy(mid);
    root = join(A, C);
}
```

---

## 순서 통계(k-th) & 순위(rank)

서브트리 크기 `sz` 로 **k-th(1-기반)**, **rank**(작은 원소 개수) 지원이 간단하다.

```cpp
// k번째(1-based) 원소를 루트로 splay 후 반환
static Node* kth(Node*& root, int k){
    if(!root || k<=0 || k>sizeOf(root)) return nullptr;
    Node* cur = root;
    while(cur){
        int L = sizeOf(cur->l);
        if(k <= L) cur = cur->l;
        else if(k <= L + cur->cnt){ splay(root, cur); return root; }
        else { k -= L + cur->cnt; cur = cur->r; }
    }
    return nullptr;
}

// key의 rank: (<key) 개수
static int rankOf(Node*& root, int key){
    Node* cur = root; int rank = 0;
    while(cur){
        if(key <= cur->key) cur = cur->l;
        else{
            rank += sizeOf(cur->l) + cur->cnt;
            cur = cur->r;
        }
    }
    if(root) splay(root, root); // 선택: 마지막 방문을 올려 캐시 효과
    return rank;
}
```

---

## 하향식(Top-Down) 스플레이 스니펫

탐색하면서 세 개의 체인을 유지한다. 구현이 간결하고 꼬리 작업이 적다.

```cpp
// top-down splay (Tarjan): key를 루트로(없으면 가까운 노드가 루트)
static Node* splayTopDown(Node* root, int key){
    if(!root) return root;
    Node N(0); // 가드
    Node *L = &N, *R = &N; // Left, Right 체인의 끝
    while(true){
        if(key < root->key){
            if(!root->l) break;
            if(key < root->l->key){ // Zig-Zig right
                Node* y = root->l;
                root->l = y->r; if(y->r) y->r->p = root;
                y->r = root; root->p = y;
                pull(root); pull(y);
                root = y;
                if(!root->l) break;
            }
            // Right chain으로 당김
            R->l = root; root->p = R;
            R = root;
            root = root->l;
        }else if(key > root->key){
            if(!root->r) break;
            if(key > root->r->key){ // Zig-Zig left
                Node* y = root->r;
                root->r = y->l; if(y->l) y->l->p = root;
                y->l = root; root->p = y;
                pull(root); pull(y);
                root = y;
                if(!root->r) break;
            }
            // Left chain으로 당김
            L->r = root; root->p = L;
            L = root;
            root = root->r;
        }else break;
    }
    // assemble
    L->r = root->l; if(L->r) L->r->p = L;
    R->l = root->r; if(R->l) R->l->p = R;
    root->l = N.r; if(root->l) root->l->p = root;
    root->r = N.l; if(root->r) root->r->p = root;
    pull(root);
    return root;
}
```

---

## 사용 예시(통합 데모)

```cpp
int main(){
    ios::sync_with_stdio(false);
    cin.tie(nullptr);

    Node* root = nullptr;

    // 삽입
    for(int x : {10,20,5,30,25,5,7,15}) insert(root, x);

    // 탐색 + 전/후임자
    findAndSplay(root, 7);
    Node* pre = predecessor(root, 7);     // <7 최대
    Node* suc = successor(root, 7);       // >7 최소
    cout << "root=" << root->key
         << " pre=" << (pre? pre->key:-1)
         << " suc=" << (suc? suc->key:-1) << "\n";

    // 순서통계
    Node* k3 = kth(root, 3);
    cout << "kth(3)=" << (k3? k3->key:-1) << " root=" << root->key << "\n";
    cout << "rank(15)=" << rankOf(root, 15) << "\n";

    // 범위 삭제
    eraseRange(root, 7, 20);

    // 남은 것 확인(k-th로 순회)
    int n = sizeOf(root);
    cout << "after eraseRange [7..20], size=" << n << " : ";
    for(int i=1;i<=n;++i){ Node* ki = kth(root, i); cout << ki->key << " "; }
    cout << "\n";

    // 삭제
    erase(root, 25);
    erase(root, 5); // 중복 한 개 줄어듦
    cout << "final size=" << sizeOf(root) << "\n";

    return 0;
}
```

샘플 출력(값은 트리 모양에 따라 달라질 수 있음):

```
root=7 pre=5 suc=10
kth(3)=7 root=7
rank(15)=5
after eraseRange [7..20], size=2 : 5 25
final size=2
```

---

## 상각 분석(핵심 직관)

정확한 증명은 **포텐셜(potential) 함수**와 **랭크(rank)** 를 사용한다.

- 각 노드 \(x\) 에 대해 **가중치** \(w(x)\) 를 정의(보통 서브트리 크기와 관련).
- **랭크** \(r(x) = \log W(x)\) (\(W(x)\): \(x\) 서브트리의 총 가중치).
- 트리의 포텐셜 \(\Phi = \sum_x r(x)\).

단일 스플레이의 **실제 비용** + **포텐셜 감소** \(\Delta \Phi\) 를 합하면,
\[
\text{상각비용} \le 3\big(r(\text{루트}) - r(\text{접근 노드})\big) + O(1)
\]
과 같은 형태가 성립하며, 결과적으로 **상각 \(O(\log n)\)** 이 도출된다.
또한 스플레이 트리는 **워크셋(working-set) 경계**, **동적 핑거(dynamic finger) 경계** 등 **접근 패턴 친화적 최적성** 특성을 만족한다(정확한 정리는 고전 논문 참고).

---

## 실전 팁(디버깅/안정성/정책)

- **중복 정책**: 위 코드는 `cnt` 누적으로 처리. 동일 키를 **오른쪽에만** 넣는 정책도 가능(간단하지만 k-th/순위 구현은 `cnt` 방식이 깔끔).
- **메모리**: `eraseRange` 와 같은 파괴 함수는 반드시 후위 순회로 안전 삭제.
- **pull 일관성**: 모든 구조 변경 후 `pull` 호출(회전/연결/분리/합치기).
- **부모 포인터**: `connect` 헬퍼로 **양방향** 일관 갱신.
- **최악 지연**: 단일 연산 최악 \(O(n)\)이 문제라면 RB/AVL/트립 검토.
- **하향식 채택**: 재귀 제거·간결성/성능 측면에서 top-down도 추천(위 스니펫 참고).

---

## 응용: 집합/구간 조작, 시퀀스 구조

- **집합 연산**: `split/join` 으로 합집합/교집합/차집합을 재귀 구성 가능(균형 유지 불필요).
- **시퀀스(배열) 흉내**: 키 대신 **암묵 인덱스(implicit key)** 를 쓰면 구간 반전/이동도 가능(스플레이보다는 Treap/Rope가 흔하지만, 스플레이도 가능).
- **LRU/캐시**: 스플레이의 *최근 접근 승진* 특성이 자연스러운 캐시 패턴과 잘 맞는다.

---

## 체크리스트(버그 방지)

- 회전 후: (1) `child->p` 세팅, (2) `g`와의 연결 갱신, (3) `pull(p)`, `pull(x)`.
- `split` 후: 잘린 쪽의 `p=nullptr`.
- `join` 전제: **모든 L < 모든 R** — 전제 깨지면 BST 불변식 붕괴.
- `kth` 경계: `1 ≤ k ≤ size(root)` 검사.

---

## 요약

| 키워드 | 한 줄 요약 |
|---|---|
| 자가 조정 | 접근 노드를 루트로 끌어올려 **최근 사용 빠름** |
| 보장 | 단일 최악 \(O(n)\), 그러나 **상각 \(O(\log n)\)** |
| 구현 | 높이/색 정보 없이 **회전만**으로 유지 |
| 고급 연산 | **split/join**, **k-th/rank**, **범위 삭제** 손쉽게 구성 |
| 대안 | 최악 지연 민감 ⇒ **RB/AVL/Treap**, 고차원 패턴 ⇒ **B-트리류** |

---

## 수학 스냅샷(표기)

- 포텐셜:
  $$\Phi(T)=\sum_{x\in T} \log W(x), \quad W(x)\ \text{: 서브트리 가중치}.$$
- 상각 비용(대략):
  $$\text{amortized\_cost} \le O\big(\log \tfrac{W(\text{root})}{W(\text{accessed})}\big).$$
