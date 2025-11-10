---
layout: post
title: Data Structure - 펜윅 트리
date: 2024-12-21 19:20:23 +0900
category: Data Structure
---
# Link-Cut Tree (LCT)

## 1. 왜 Link-Cut Tree인가

정적 트리에서는 세그먼트 트리·HLD(Heavy-Light Decomposition) 같은 기법으로 경로 쿼리를 빠르게 처리할 수 있다. 하지만 **간선이 동적으로 추가/제거**되는 상황(포레스트에서 간선 link/cut)에서는 정적 분해가 깨진다. **Link-Cut Tree(LCT)** 는 **스플레이 트리(splay tree)** 로 **선호 경로(preferred path)** 를 유지하여, 다음 연산들을 **아몰타이즈 \(O(\log n)\)** 에 처리한다.

- `link(u, v)` : 서로 다른 트리를 **간선 (u, v)** 로 연결(포레스트 유지 전제)
- `cut(u, v)` : 간선 (u, v) 제거
- `makeRoot(u)` : u를 속한 트리의 새로운 루트로 만듦(경로 방향 반전)
- `findRoot(u)` : u가 속한 트리의 루트 탐색
- `connected(u, v)` : 두 노드가 같은 트리인지 판별
- `split(u, v)` : 경로 \(u \leadsto v\) 를 하나의 스플레이로 “노출”
- `pathQuery(u, v)` : 경로 질의(합/최솟값/최댓값 등)
- `pathUpdate(u, v, Δ)` : 경로 전가산/전최솟값 등 **lazy** 갱신(옵션)

핵심 직관: LCT는 각 정점이 속한 **보조 트리(auxiliary tree)** 를 스플레이로 관리하여, **`access(x)`** 라는 연산으로 **루트→x 경로**를 한 개의 스플레이로 노출한다. 이때 스플레이의 루트가 경로의 말단(x)이며, 스플레이 전체가 바로 **질의/갱신 대상 구간**이 된다.

---

## 2. 기본 수학과 시간복잡도 개요

스플레이 트리의 고전 결과를 그대로 물려받는다. 임의의 연산 시퀀스에 대해(스플레이 회전들이 포함된) 총 비용은 다음과 같다:

- 임의의 길이 \(m\) 연산 시퀀스에 대해  
  $$ T(m) = O\big((n + m)\log n\big) $$
- 즉 **아몰타이즈 \(O(\log n)\)**.

이는 스플레이의 **잠재 함수(potential function)** 분석(무게 균형에 대한 rank potential)으로 얻는다. LCT는 `access` 가 **스플레이 연산 \(O(\log n)\)** 를 유한 번 호출하는 구조라 전체도 \(O(\log n)\) 로 누적된다.

---

## 3. 자료구조 핵심 아이디어

- 각 정점 \(x\) 는 포인터 `left`, `right`, `parent` 를 가진다.  
- `access(x)` 는 “루트→x” 선호 경로를 마지막으로 **x가 속한 스플레이의 루트**가 되도록 만든다. 과정에서 x의 오른쪽 자식은 “다음 선호 경로로 넘어가는 접합부” 역할을 한다.
- `makeRoot(x)` 는 `access(x)` 후에 “경로 방향”을 **뒤집는 lazy 태그(`rev`)** 를 적용한다.
- `split(u, v)` 는 `makeRoot(u)` → `access(v)` 를 통해 **경로 \(u \leadsto v\)** 를 하나의 스플레이로 노출한다. 그러면 스플레이 루트 v의 서브트리 집계 값이 곧 **경로 집계**가 된다.
- `link(u, v)` : 서로 다른 트리일 때 `makeRoot(u)` 후 `u.parent = v`.
- `cut(u, v)` : `makeRoot(u)`, `access(v)` 후 `(v.left == u && !u.right)` 를 확인하고 `v.left = u.parent = nullptr`.

---

## 4. 안전하고 확장 가능한 C++ 구현 (경로 합/경로 가산 지원)

> 설명을 위해 “경로 합”과 “경로 전가산(Δ 더하기)”를 넣은 버전이다. 필요에 따라 min/max로 바꾸거나, xor 등 교환/결합 가능한 연산으로 교체 가능하다.

```cpp
#include <bits/stdc++.h>
using namespace std;

/*** Link-Cut Tree (Path Sum + Path Add) ***/
struct Node {
    Node *left = nullptr, *right = nullptr, *parent = nullptr;
    bool rev = false;          // 경로 반전 lazy
    long long add = 0;         // 경로 전가산 lazy
    int sz = 1;                // 스플레이 크기 (보조트리 크기)
    long long val = 0;         // 노드 자체 값
    long long sum = 0;         // 스플레이 집계(합)
    int id = -1;               // 디버깅용

    Node(int _id=0, long long _val=0) : id(_id), val(_val), sum(_val) {}
};

static inline bool isRoot(Node* x) {
    return !x->parent || (x->parent->left != x && x->parent->right != x);
}
static inline int size(Node* x){ return x ? x->sz : 0; }
static inline long long ssum(Node* x){ return x ? x->sum : 0LL; }

// lazy 적용: 경로 반전
static inline void applyRev(Node* x){
    if(!x) return;
    x->rev ^= 1;
    swap(x->left, x->right);
}

// lazy 적용: 전가산 Δ
static inline void applyAdd(Node* x, long long d){
    if(!x) return;
    x->add += d;
    x->val += d;
    x->sum += 1LL * d * x->sz; // 스플레이 전체에 더해짐
}

// lazy 전파
static inline void push(Node* x){
    if(!x) return;
    if(x->rev){
        if(x->left)  applyRev(x->left);
        if(x->right) applyRev(x->right);
        x->rev = false;
    }
    if(x->add){
        if(x->left)  applyAdd(x->left, x->add);
        if(x->right) applyAdd(x->right, x->add);
        x->add = 0;
    }
}

// 상향 재계산
static inline void pull(Node* x){
    x->sz  = 1 + size(x->left) + size(x->right);
    x->sum = x->val + ssum(x->left) + ssum(x->right);
}

// 조상까지 lazy를 미리 내려서 splay의 회전 안정화
static void pushAll(Node* x){
    if(!x || isRoot(x)) { push(x); return; }
    vector<Node*> stk;
    Node* cur = x;
    while(!isRoot(cur)){ stk.push_back(cur); cur = cur->parent; }
    stk.push_back(cur);
    for(int i=(int)stk.size()-1;i>=0;--i) push(stk[i]);
}

static void rotate(Node* x){
    Node* p = x->parent;
    Node* g = p->parent;
    push(p); push(x);
    bool xIsLeft = (p->left == x);
    Node* b = xIsLeft ? x->right : x->left;

    if(!isRoot(p)){
        if(g->left == p) g->left = x; else if(g->right == p) g->right = x;
    }
    x->parent = g;

    if(xIsLeft){
        x->right = p; p->parent = x;
        p->left  = b; if(b) b->parent = p;
    } else {
        x->left = p;  p->parent = x;
        p->right = b; if(b) b->parent = p;
    }
    pull(p); pull(x);
}

static void splay(Node* x){
    pushAll(x);
    while(!isRoot(x)){
        Node* p = x->parent;
        Node* g = p->parent;
        if(!isRoot(p)){
            bool zigzig = ( (g->left==p) == (p->left==x) );
            rotate( zigzig ? p : x );
        }
        rotate(x);
    }
}

// 루트→x 경로를 선호 경로로 만들고 x를 그 스플레이의 루트로
static void access(Node* x){
    Node* last = nullptr;
    for(Node* y = x; y; y = y->parent){
        splay(y);
        y->right = last;       // y의 preferred path의 오른쪽 끝을 갱신
        pull(y);
        last = y;
    }
    splay(x); // x가 루트(경로의 말단)
}

// x가 속한 트리의 루트가 되도록(경로 방향 반전)
static void makeRoot(Node* x){
    access(x);
    applyRev(x);
    push(x);
    pull(x);
}

// x의 실제 트리 루트를 찾음(가장 왼쪽으로)
static Node* findRoot(Node* x){
    access(x);
    while(x->left){ push(x); x = x->left; }
    splay(x);
    return x;
}

static bool connected(Node* u, Node* v){
    if(u==v) return true;
    return findRoot(u) == findRoot(v);
}

// u-v 경로를 하나의 스플레이로 노출: 이후 v 스플레이의 sum/size 등이 경로 집계
static void split(Node* u, Node* v){
    makeRoot(u);
    access(v); // v가 루트이며, v의 스플레이가 경로 전체
}

// 간선(u, v) 추가(포레스트 유지: u의 루트를 v와 다르게 유지)
static bool link(Node* u, Node* v){
    makeRoot(u);
    if(findRoot(v) == u) return false; // 이미 연결(사이클 방지)
    u->parent = v;
    return true;
}

// 간선(u, v) 제거
static bool cut(Node* u, Node* v){
    makeRoot(u);
    access(v);
    // 이제 v의 스플레이에서 v.left == u 이고 u.right == nullptr 이어야 간선 (u,v)
    if(v->left != u || (u->right != nullptr)) return false;
    v->left->parent = nullptr;
    v->left = nullptr;
    pull(v);
    return true;
}

// 경로 전가산
static void pathAdd(Node* u, Node* v, long long d){
    split(u, v);
    applyAdd(v, d); // v의 스플레이 전체에 적용
    push(v); pull(v);
}

// 경로 합
static long long pathSum(Node* u, Node* v){
    split(u, v);
    return v->sum;
}

// 점 갱신: x의 값을 newVal로 설정
static void setVal(Node* x, long long newVal){
    access(x);
    x->val = newVal;
    pull(x);
}

// 디버그 도우미
static void printPathSum(Node* u, Node* v, const string& msg=""){
    cout << (msg.empty()? "" : msg + " ") << pathSum(u, v) << "\n";
}

int main(){
    ios::sync_with_stdio(false);
    cin.tie(nullptr);

    int N = 6;
    vector<Node*> a(N+1);
    for(int i=1;i<=N;i++) a[i] = new Node(i, i); // 값=노드번호

    // 트리 1: 1-2-3-4
    link(a[1], a[2]);
    link(a[2], a[3]);
    link(a[3], a[4]);

    // 트리 2: 5-6
    link(a[5], a[6]);

    // 연결 여부
    cout << "connected(1,4) = " << connected(a[1], a[4]) << "\n"; // 1
    cout << "connected(1,6) = " << connected(a[1], a[6]) << "\n"; // 0

    // 경로 합(1..4) = 1+2+3+4 = 10
    cout << "sum(1,4) = " << pathSum(a[1], a[4]) << "\n";

    // 경로 전가산(2..4, +10): 2->12,3->13,4->14
    pathAdd(a[2], a[4], 10);
    cout << "sum(1,4) after add(2..4,+10) = " << pathSum(a[1], a[4]) << "\n"; // 1 + (12+13+14) = 40

    // 점 갱신: 3 := 100
    setVal(a[3], 100);
    cout << "sum(1,4) after set(3=100) = " << pathSum(a[1], a[4]) << "\n"; // 1 + (12+100+14) = 127

    // 간선 절단: (2,3)
    cout << "cut(2,3) = " << cut(a[2], a[3]) << "\n";
    cout << "connected(1,4) after cut = " << connected(a[1], a[4]) << "\n"; // 0

    // 재연결: (1,5)
    cout << "link(1,5) = " << link(a[1], a[5]) << "\n";
    cout << "connected(4,6) now = " << connected(a[4], a[6]) << "\n"; // 1

    // 새로운 경로 합: (4..6)
    cout << "sum(4,6) = " << pathSum(a[4], a[6]) << "\n";

    // 자원 정리
    for(int i=1;i<=N;i++) delete a[i];
    return 0;
}
```

### 구현 체크포인트

1. **`pushAll`** 로 조상들의 lazy(`rev`, `add`)를 내려준 뒤에 회전(splay)해야 한다.
2. **`pull`** 은 회전 이후 항상 호출되어야 집계값(`sz`, `sum`)이 보정된다.
3. **`cut(u,v)`** 는 `makeRoot(u)`, `access(v)` 한 뒤에 **정확히 간선 (u,v)만** 제거하는 조건을 확인한다.
4. **`link(u,v)`** 는 사이클 방지를 위해 `findRoot(v) != u` 를 먼저 확인한다.
5. **경로 전가산/합** 은 `split(u,v)` 후 **스플레이 루트 v** 전체에 lazy를 적용/집계한다.

---

## 5. API 설계(요약)

- `makeRoot(u)`: 루트→u 경로를 뒤집어 u를 트리 루트로.  
- `findRoot(u)`: `access(u)` 후 가장 왼쪽으로 전개, splay.  
- `connected(u, v)`: `findRoot(u) == findRoot(v)`.  
- `link(u, v)`: 서로 다른 트리일 때만 연결.  
- `cut(u, v)`: 해당 간선만 제거.  
- `split(u, v)`: 경로를 한 스플레이로 노출.  
- `pathAdd(u, v, d)`, `pathSum(u, v)`, `setVal(u, newVal)` : 질의/갱신.

---

## 6. 다른 기법과의 비교

| 기준 | Link-Cut Tree | Heavy-Light Decomposition | Euler Tour + 세그트리 |
|---|---|---|---|
| 간선 동적 추가/삭제 | **가능** | 어려움(재분해 필요) | 일부 가능(재빌드 비용 큼) |
| 경로 쿼리 | **자연스러움** | 자연스러움 | 자연스러움 |
| 서브트리 쿼리 | 까다로움(확장 필요) | 자연스러움 | 자연스러움 |
| 구현 난이도 | 높음(스플레이+lazy) | 중간 | 중간 |
| 시간복잡도 | 아몰타이즈 \(O(\log n)\) | \(O(\log^2 n)\) 전후 | \(O(\log n)\) |

- **서브트리 질의**는 LCT의 전통 구현에서는 직접적이지 않다. **Evert(makeRoot) + access** 패턴과 보조 태그를 섞어 구현하는 변종도 있으나, 일반적으로 **경로 질의**에 특화되어 있다고 이해하면 정확하다.

---

## 7. 전형적 실전 시나리오

1. **동적 MST 유지보수**  
   간선 추가 시 사이클이 생기면 경로 최대 가중치 간선과 비교해 더 큰 쪽을 제거. LCT의 **경로 최대값 쿼리**가 핵심.
2. **네트워크 연결성 실시간 판정**  
   링크/절단 이벤트 스트림에서 `connected(u, v)` 연속 판정.  
3. **게임·시뮬레이션 트리 구조**  
   장비/버프를 트리형으로 붙였다 떼는 구조에서 경로 누적 효과 계산.

---

## 8. 디버깅·안정성 체크리스트

- `isRoot` 정의가 바르게 되어 있는가? (부모의 진짜 자식인지 검사)
- 회전 전 **`push(p)`, `push(x)`** 를 잊지 않았는가?
- `cut` 조건 `(v->left == u && !u->right)` 을 체크하는가?
- lazy(`rev`, `add`)가 **중복 전파/누락** 없이 작동하는가?
- 테스트:
  - 단일 노드·두 노드·선형 체인에서 link/cut 반복
  - 임의 경로에 `pathAdd` 후 `pathSum` 검증
  - 대량 랜덤 시퀀스(fuzz)로 **포레스트 불변식** 확인

---

## 9. 자주 쓰는 변형

- **경로 최대/최소**: `sum` 대신 `mx`/`mn` 유지, lazy는 전최대/전최소에 맞게 설계.
- **경로 XOR**: 교환·결합 법칙이 성립하는 연산이면 동일 패턴.
- **값이 간선에 있을 때**: 간선을 노드로 표현(예: (u,v) 간선 값을 v 쪽 노드에 저장 후 `makeRoot(u)`와 `access(v)`로 경로 노출 등 전형적 트릭).

---

## 10. 요약

- **핵심**: `access` 로 경로 노출 → 스플레이 루트 하나에 **경로 집계/갱신**을 일괄 적용.  
- **시간복잡도**: 모든 연산이 아몰타이즈 \(O(\log n)\).  
- **장점**: 동적 link/cut 에 특화된 경로 질의/갱신.  
- **주의**: 구현 난이도·lazy 전파·정확한 cut/link 조건.

위 C++ 템플릿을 바탕으로, 여러분의 문제에 맞는 **집계(합/최대/최소/비트연산)** 와 **lazy 정책**을 넣어 **동적 트리 문제**를 안정적으로 처리할 수 있다.
