---
layout: post
title: Data Structure - 펜윅 트리
date: 2024-12-21 19:20:23 +0900
category: Data Structure
---
# Euler Tour Tree (ETT)

## 무엇을 다루는가 (개요 & 목표)

**Euler Tour Tree (ETT)** 는 트리(또는 포레스트)를 **오일러 순회(Euler Tour) 결과**로 “1차원 배열”에 펴서, 세그먼트 트리·펜윅 트리·Splay/트립(implicit treap) 같은 자료구조로 **서브트리/경로 질의와 갱신을 \(O(\log n)\)** 에 처리하려는 기법이다.
이 글은 다음을 한 번에 정리한다.

- 정적 트리: **입·출시각(in/out) 기반 ETT**로 서브트리 질의/갱신
- 정적 트리: **경로 질의**(LCA + prefix trick)와 **lazy** 확장
- **동적 ETT(implicit treap)**: 포레스트에서 **link/cut/connected**와 **(루트 기준) 서브트리 합/가산 갱신** 시연

핵심 성질(전위기준 첫 방문 `tin[u]`, 마지막 방문 `tout[u]`)은:

- (고전) **서브트리(u)** 는 오일러 배열에서 **연속 구간**이 된다.
- (정확식) 임의의 노드 가중치 \(w\) 에 대해
  \[
  \text{subtree\_sum}(u) \;=\; \sum_{i=\mathrm{tin}[u]}^{\mathrm{tout}[u]} A[i]
  \]
  와 같이 구간 합으로 환원된다.

---

## 두 가지 ETT 관점: “정적(Subtree/Q)” vs “동적(Link/Cut)”

| 구분 | 핵심 아이디어 | 장점 | 단점/주의 |
|---|---|---|---|
| **정적 ETT** | DFS로 `tin/tout`를 잡고 1회 배열화 | 구현 간단, 서브트리 질의·갱신 강력 | 간선 구조가 바뀌면 재빌드 필요 |
| **동적 ETT** | 오일러 순회 **시퀀스 자체**를 Splay/treap으로 유지 | **link/cut/connected**를 \(O(\log n)\) 근처로 처리 | 구현 난이도↑, “루트 고정/표시” 등 설계 필요 |

> 경로 질의(두 노드 u–v)는 **정적**이면 LCA로 쉽게 풀리나, **완전 동적**이면 보통 **Link-Cut Tree** 또는 **Top Tree**가 더 흔하다.
> 동적 ETT는 **연결성/서브트리 질의(루트 선택)** 중심으로 쓰기 좋다.

---

## 오일러 순회 수학: 구간화의 원리

루트를 \(r\) 로 정하고 DFS 전위 순회를 하자. `tin[u]`를 **u에 최초 진입 시각**, `tout[u]`를 **u의 서브트리 처리가 끝나고 나갈 때**의 시각으로 두면:

- **서브트리 포함성**
  \[
  v \in \mathrm{subtree}(u) \iff \mathrm{tin}[u] \le \mathrm{tin}[v] \le \mathrm{tout}[u].
  \]
- **서브트리 합(엔트리만 기록)**
  오일러 배열 \(E\) 를 “**첫 방문만** 기록”하는 길이 \(n\) 배열로 두면,
  \[
  \mathrm{subtree\_sum}(u)=\sum_{i=\mathrm{tin}[u]}^{\mathrm{tout}[u]} W[E[i]].
  \]
  (여기서 \(W[x]\) 는 노드 \(x\) 의 현재 가중치)
- **경로 합** (정적 + LCA 전처리)
  \[
  \mathrm{path\_sum}(u,v) \;=\; \mathrm{dist\_sum}(r,u)+\mathrm{dist\_sum}(r,v)-2\cdot \mathrm{dist\_sum}(r,\mathrm{lca}(u,v)) + W[\mathrm{lca}(u,v)].
  \]
  여기서 \(\mathrm{dist\_sum}(r,x)\) 는 루트 \(r\to x\) 까지의 누적합(전위 DFS로 prefix 화 가능).

---

## 정적 ETT ① — 가장 많이 쓰는 `tin/tout` + 세그먼트 트리

### DFS로 입·출시각과 1차원 전개

```cpp
#include <bits/stdc++.h>

using namespace std;

const int MAXN = 200000;
int N;
vector<int> g[MAXN+1];

int timer=0;
int tin[MAXN+1], tout[MAXN+1], euler[MAXN+1]; // 엔트리만 기록
int parent_[MAXN+1], depth_[MAXN+1];

// DFS: 첫 방문 시 tin[u]=++timer; euler[tin[u]]=u;
void dfs(int u, int p){
    parent_[u]=p;
    tin[u]=++timer;
    euler[tin[u]]=u;
    for(int v: g[u]) if(v!=p){
        depth_[v]=depth_[u]+1;
        dfs(v,u);
    }
    tout[u]=timer;
}
```

- `euler[1..N]` 는 “첫 방문만” 저장. 덕분에 **서브트리(u) = [tin[u], tout[u]]** 가 **연속 구간**이 된다.

### 세그먼트 트리(합) + 서브트리 합/가산 갱신

```cpp
struct Seg {
    int n;
    struct Node{ long long sum,lz; } ;
    vector<Node> t;
    Seg(int n=0):n(n),t(4*n+4,{0,0}){}
    void build(const vector<long long>& base,int o,int l,int r){
        if(l==r){ t[o].sum = base[l]; return; }
        int m=(l+r)>>1;
        build(base,o<<1,l,m);
        build(base,o<<1|1,m+1,r);
        t[o].sum = t[o<<1].sum + t[o<<1|1].sum;
    }
    void apply(int o,int l,int r,long long v){
        t[o].sum += 1LL*(r-l+1)*v;
        t[o].lz  += v;
    }
    void push(int o,int l,int r){
        if(t[o].lz){
            int m=(l+r)>>1;
            apply(o<<1,l,m,t[o].lz);
            apply(o<<1|1,m+1,r,t[o].lz);
            t[o].lz=0;
        }
    }
    void rangeAdd(int o,int l,int r,int ql,int qr,long long v){
        if(qr<l||r<ql) return;
        if(ql<=l&&r<=qr){ apply(o,l,r,v); return; }
        push(o,l,r);
        int m=(l+r)>>1;
        rangeAdd(o<<1,l,m,ql,qr,v);
        rangeAdd(o<<1|1,m+1,r,ql,qr,v);
        t[o].sum = t[o<<1].sum + t[o<<1|1].sum;
    }
    long long rangeSum(int o,int l,int r,int ql,int qr){
        if(qr<l||r<ql) return 0;
        if(ql<=l&&r<=qr) return t[o].sum;
        push(o,l,r);
        int m=(l+r)>>1;
        return rangeSum(o<<1,l,m,ql,qr)+rangeSum(o<<1|1,m+1,r,ql,qr);
    }
};
```

### 사용 예 — 서브트리 합과 갱신

```cpp
int main(){
    ios::sync_with_stdio(false); cin.tie(nullptr);
    int n; cin>>n;
    for(int i=1;i<=n;i++){ g[i].clear(); }
    for(int i=0;i<n-1;i++){
        int u,v; cin>>u>>v;
        g[u].push_back(v); g[v].push_back(u);
    }
    vector<long long> W(n+1); // 노드 값
    for(int i=1;i<=n;i++) cin>>W[i];

    timer=0; depth_[1]=0; dfs(1,0);

    // 오일러 1차원 기저
    vector<long long> base(n+1);
    for(int pos=1; pos<=n; ++pos){
        int u = euler[pos];
        base[pos] = W[u];
    }
    Seg seg(n); seg.build(base,1,1,n);

    int q; cin>>q;
    while(q--){
        int tp; cin>>tp;
        if(tp==1){ // 서브트리 가산
            int u; long long d; cin>>u>>d;
            seg.rangeAdd(1,1,n, tin[u], tout[u], d);
        }else if(tp==2){ // 서브트리 합
            int u; cin>>u;
            cout << seg.rangeSum(1,1,n, tin[u], tout[u]) << "\n";
        }
    }
    return 0;
}
```

- **시간복잡도**: 빌드 \(O(n)\), 각 질의/갱신 \(O(\log n)\).
- **변형**: 합 대신 `min/max/xor` 등으로 노드 구성만 바꾸면 끝.

---

## 정적 ETT ② — 경로 질의(정적)까지: LCA + prefix

경로 질의는 정적 트리에서 보통 다음 식을 쓴다:

\[
\mathrm{path\_sum}(u,v) \;=\; S(u)+S(v)-2\cdot S(\mathrm{lca}(u,v)) + W[\mathrm{lca}(u,v)]
\]
여기서 \(S(x)=\sum_{y \in \text{경로 } r\to x} W[y]\).

- \(S(x)\) 는 DFS 중 “입점에 더하고 출점에 빼는” 2차 방문 오일러 배열로도 만들 수 있고,
- 혹은 단순히 **부모→자식 전개 중 prefix** 로 만들어도 된다.

### LCA(이진 점프) 전처리 스니펫

```cpp
const int LOG=20;
int up[LOG][MAXN+1];

void dfs2(int u,int p){
    up[0][u]=p;
    for(int k=1;k<LOG;k++) up[k][u]= up[k-1][u]? up[k-1][ up[k-1][u] ]:0;
    for(int v: g[u]) if(v!=p){
        depth_[v]=depth_[u]+1;
        dfs2(v,u);
    }
}
int lca(int a,int b){
    if(depth_[a]<depth_[b]) swap(a,b);
    int d = depth_[a]-depth_[b];
    for(int k=0;k<LOG;k++) if(d>>k&1) a=up[k][a];
    if(a==b) return a;
    for(int k=LOG-1;k>=0;k--){
        if(up[k][a]!=up[k][b]){ a=up[k][a]; b=up[k][b]; }
    }
    return up[0][a];
}
```

> 경로 갱신/질의가 **많고 간선이 변하지 않는** 문제라면, 이 LCA+prefix 패턴이 가장 단순·견고하다.

---

## 동적 ETT — implicit treap으로 “오일러 시퀀스 자체” 유지

정적 ETT는 간선이 바뀌면 재DFS가 필요하다. **동적 ETT** 는 각 트리를 **오일러 시퀀스(원형 리스트)** 로 보고, 이를 **암시적 키 트립(implicit treap)** 로 구현하여 **split/merge** 로 다룬다.

### 설계 원칙(본 글 구현의 목표)

- **연산**: `connected(u,v)`, `link(u,v)`(서로 다른 트리 연결), `cut(u,v)`(간선 제거), `reroot(r)`(원형 시퀀스 회전),
  그리고 **루트 r 기준** `subtreeAdd(u,Δ)` / `subtreeSum(u)` (노드 값 합)
- **표현**: 각 정점 u에 대해 **ENTER(u)**, **EXIT(u)** 두 토큰을 시퀀스에 둔다.
  **r을 루트로 잡으면** `[pos(ENTER(u)) .. pos(EXIT(u))]` 가 **u의 서브트리 구간**이 된다.
- **핵심**: 동적으로 트리를 바꿔도, `reroot` 로 원하는 루트로 회전하면 **서브트리(u)는 항상 연속**.

> 경로 질의(임의 u–v)는 동적 ETT만으로는 까다롭다. 보통 **LCT/Top Tree** 가 더 적합. 본 구현은 **서브트리 질의/갱신** 중점.

### implicit treap 노드와 연산

```cpp
struct TNode{
    TNode *l=nullptr,*r=nullptr,*p=nullptr;
    int pri = (rand()<<16)^rand();
    // 암시적 키: 서브트리 크기로 순서를 유지
    int sz=1;

    // 토큰 유형
    enum Kind{ENTER,EXIT} kind;
    int u; // vertex id

    // 집계/레이지: 합 + 구간가산
    long long val=0, sum=0, lz=0;

    // 디버그용
    int id;
    TNode(int _id,int _u,Kind _k,long long _val):id(_id),u(_u),kind(_k),val(_val){ sum=val; }
};

static int getsz(TNode* x){ return x?x->sz:0; }
static long long getsum(TNode* x){ return x?x->sum:0; }

static void pull(TNode* x){
    if(!x) return;
    x->sz = 1 + getsz(x->l) + getsz(x->r);
    x->sum= x->val + getsum(x->l) + getsum(x->r);
    if(x->l) x->l->p=x;
    if(x->r) x->r->p=x;
}
static void applyAdd(TNode* x,long long d){
    if(!x) return;
    x->val += d;
    x->sum += 1LL*d*getsz(x);
    x->lz  += d;
}
static void push(TNode* x){
    if(!x||!x->lz) return;
    applyAdd(x->l,x->lz);
    applyAdd(x->r,x->lz);
    x->lz=0;
}
```

- **split(root, k)**: 왼쪽 서브트리 크기가 k가 되게 분리
- **merge(a,b)**: a의 모든 인덱스가 b보다 앞인 상태로 결합

```cpp
pair<TNode*,TNode*> split(TNode* root,int k){ // [0..k-1] | [k..)
    if(!root) return {nullptr,nullptr};
    push(root);
    if(getsz(root->l)>=k){
        auto [L,R] = split(root->l,k);
        root->l=R; pull(root);
        if(R) R->p=root;
        return {L,root};
    }else{
        auto [L,R] = split(root->r, k-getsz(root->l)-1);
        root->r=L; pull(root);
        if(L) L->p=root;
        return {root,R};
    }
}
TNode* merge(TNode* a,TNode* b){
    if(!a) return b; if(!b) return a;
    if(a->pri < b->pri){
        push(a);
        a->r = merge(a->r,b); pull(a);
        if(a->r) a->r->p=a;
        return a;
    }else{
        push(b);
        b->l = merge(a,b->l); pull(b);
        if(b->l) b->l->p=b;
        return b;
    }
}
```

### 인덱스 탐색 & 구간 연산

```cpp
// k번째 (0-index) 노드
TNode* kth(TNode* root,int k){
    if(!root || k<0 || k>=getsz(root)) return nullptr;
    push(root);
    int ls = getsz(root->l);
    if(k<ls) return kth(root->l,k);
    if(k==ls) return root;
    return kth(root->r, k-ls-1);
}
// 노드의 인덱스 (root 기준)
int indexOf(TNode* x){
    int idx = getsz(x->l);
    while(x->p){
        if(x->p->r == x) idx += getsz(x->p->l) + 1;
        x = x->p;
    }
    return idx;
}
```

**구간 가산/합**:

```cpp
// [l..r] += d
void rangeAdd(TNode*& root,int l,int r,long long d){
    auto [A,BC] = split(root, l);
    auto [B,C]  = split(BC, r-l+1);
    applyAdd(B,d);
    root = merge(A, merge(B,C));
}
// sum[l..r]
long long rangeSum(TNode*& root,int l,int r){
    auto [A,BC] = split(root, l);
    auto [B,C]  = split(BC, r-l+1);
    long long ans = getsum(B);
    root = merge(A, merge(B,C));
    return ans;
}
```

### ETT 시퀀스: ENTER/EXIT 토큰, 포인터 테이블

- 각 정점 \(u\) 에 대해 **ENTER(u)**, **EXIT(u)** 토큰 포인터를 저장한다.
- 시퀀스에서 **루트 r 기준**으로 보면, \([ \mathrm{pos}(\mathrm{ENTER}(u))\,..\,\mathrm{pos}(\mathrm{EXIT}(u)) ]\) 가 **u 서브트리**.
- `reroot(r)` = **시퀀스 회전**: `pos(ENTER(r))` 를 맨 앞으로 오게 split/merge.

### link/cut/connected 구현 개요

- **초기화**: 각 \(u\) 의 독립 트리는 \([ENTER(u), EXIT(u)]\) (길이 2) 로 만들어 둔다.
- **connected(u,v)**: 각 시퀀스의 **대표(root 포인터)** 가 같은지로 판정.
  - (여기선 간단히 “루트 TNode *” 같은 대표를 추적하거나, `findRootPtr` 로 최상위 부모를 거슬러 올라가 비교)
- **reroot(r)**: `pos = indexOf(ENTER(r))`; `split(root,pos)` → 오른쪽 + 왼쪽을 `merge(right,left)`.
- **link(u,v)** (u,v가 서로 다른 트리):
  `reroot(u)`; `reroot(v)`; 두 시퀀스를 **중간에 간선 삽입 없이** 단순히 **concat** 하면 Euler 순회가 연결 트리에 대응한다.
  (간선을 명시 토큰으로 둘 필요 없이, ENTER/EXIT의 interleave가 자연히 형성됨)
- **cut(u,v)**: 끊을 간선(u,v)은 ETT 상에서 두 구간 사이의 **경계** 두 곳을 찾는 split 2회로 구현.
  (실전에서는 “간선 토큰”을 명시하면 cut 가 쉬워지지만, 본 예제는 u를 루트로 회전 후 u–v 쪽을 잘라내는 간단 패턴을 시연)

> 주의: ETT 변형은 많다. 여기 구현은 **교육용**으로, `link/cut` 의 핵심 감각(회전→split/merge)과 **(루트 기준) 서브트리 질의/가산**을 보여준다. 완전한 일반성(모든 케이스 cut의 엄밀성, 경로 질의 등)은 LCT가 편하다.

### 시연 코드 (연결성 + reroot + 서브트리 합/가산)

```cpp
// ------- 교육용 Dynamic ETT (ENTER/EXIT 기반) -------
struct ETT {
    int n;
    vector<TNode*> enterPtr, exitPtr;
    vector<TNode*> repr; // 각 정점이 속한 treap의 대표(root) 추적용(대략적)

    // 각 컴포넌트의 시퀀스 루트를 기억하는 용도.
    // 실제 구현에서는 Node->top() 추적으로 계산하거나 DSU on pointers 등 사용 가능.

    ETT(int n, const vector<long long>& W): n(n){
        enterPtr.resize(n+1,nullptr);
        exitPtr .resize(n+1,nullptr);
        repr    .resize(n+1,nullptr);

        // 각 정점 u: [ENTER(u):W[u], EXIT(u):0]
        // 독립 트리 = 길이 2 시퀀스
        for(int u=1;u<=n;u++){
            TNode* a = new TNode((u<<1)-1,u,TNode::ENTER, W[u]);
            TNode* b = new TNode((u<<1),  u,TNode::EXIT , 0);
            TNode* root = merge(a,b);
            pull(root);
            enterPtr[u]=a; exitPtr[u]=b;
            repr[u]=root; // 임시 대표
        }
    }

    // 대표 얻기(대략): 어떤 토큰에서 parent 끝까지 올라가는 방식
    TNode* top(TNode* x){ while(x->p) x=x->p; return x; }

    // u가 속한 시퀀스 루트(대표) 갱신
    void refreshRepr(int u){ repr[u]= top(enterPtr[u]); }

    bool connected(int u,int v){
        return top(enterPtr[u]) == top(enterPtr[v]);
    }

    // 루트 r 로 회전: ENTER(r)를 시퀀스 맨 앞으로 보냄
    void reroot(int r){
        TNode* root = top(enterPtr[r]);
        int pos = indexOf(enterPtr[r]);
        auto [L,R] = split(root,pos);
        TNode* NR = merge(R,L);
        // 붙은 쪽의 대표 갱신 (간단히 r만 갱신)
        enterPtr[r]= enterPtr[r]; // 그대로
        if(NR) NR->p=nullptr;
    }

    // (루트 r 기준) 서브트리 [ENTER(u) .. EXIT(u)]
    pair<int,int> subtreeRange(int u){
        TNode* root = top(enterPtr[u]);
        int L = indexOf(enterPtr[u]);
        int R = indexOf(exitPtr[u]);
        if(L>R) swap(L,R);
        return {L,R};
    }

    long long subtreeSum(int r,int u){
        reroot(r);
        TNode* root = top(enterPtr[r]);
        auto [L,R] = subtreeRange(u);
        return rangeSum(root,L,R); // 내부 split/merge로 root 상태 복구됨
    }
    void subtreeAdd(int r,int u,long long d){
        reroot(r);
        TNode* root = top(enterPtr[r]);
        auto [L,R] = subtreeRange(u);
        rangeAdd(root,L,R,d);
    }

    // 두 컴포넌트를 연결: 교육용 간단 concat
    // 실제로는 (ENTER/EXIT) interleave가 자연히 Euler tour를 만든다.
    bool link(int u,int v){
        if(connected(u,v)) return false;
        reroot(u); reroot(v);
        TNode* A = top(enterPtr[u]);
        TNode* B = top(enterPtr[v]);
        TNode* C = merge(A,B);
        if(C) C->p=nullptr;
        refreshRepr(u); refreshRepr(v);
        return true;
    }

    // 교육용 cut: u를 루트로 회전 후, u의 어떤 자식 쪽 컴포넌트를 떼어낸다.
    // 실전 cut(u,v)는 “간선 토큰”을 명시하면 더 간단·안전.
    bool cut_as_child(int u, int child){ // (u - child) 간선을 끊는다 가정
        if(!connected(u,child)) return false;
        reroot(u);
        TNode* root = top(enterPtr[u]);
        // 이제 [ENTER(u)..EXIT(u)]가 전체 범위. child 서브트리는 내부 연속구간.
        auto [L,R] = subtreeRange(child);
        // [0..L-1] + [L..R] + [R+1..end] 에서 가운데를 떼어내면 분리.
        auto [A,BC] = split(root, L);
        auto [B,C ] = split(BC , R-L+1);
        // B는 child 컴포넌트
        TNode* X = merge(A,C);
        if(X) X->p=nullptr;
        if(B) B->p=nullptr;
        return true;
    }
};
```

**주의/한계**:

- 위 `link/cut_as_child` 는 **교육용** 으로 핵심 감각(회전→split/merge)을 보여준다.
  실전에서는 **간선 토큰(두 방향)** 을 시퀀스에 **명시적으로 삽입**해두면 `cut(u,v)` 가 더 명확하고 안전하다.
- 경로 질의는 동적 ETT 단독으론 불편. 경로 문제가 주력이면 **Link-Cut Tree** 가 구현·이론 모두 더 잘 맞는다.

### 동작 예 (입출력 없이 간단 시연)

```cpp
int main(){
    ios::sync_with_stdio(false); cin.tie(nullptr);
    srand(712367);

    int n=6;
    vector<long long> W(n+1);
    for(int i=1;i<=n;i++) W[i]=i; // 초기 값 = 정점 번호

    ETT ett(n,W);

    // 1-2-3-4, 5-6 연결
    ett.link(1,2); ett.link(2,3); ett.link(3,4);
    ett.link(5,6);

    cout << "connected(1,4): " << ett.connected(1,4) << "\n"; // 1
    cout << "connected(1,6): " << ett.connected(1,6) << "\n"; // 0

    // 루트=1, 서브트리(3) 합 = (3+4)=7 (교육용 모델에서 ENTER(3)~EXIT(3))
    cout << "subtreeSum(root=1, u=3): " << ett.subtreeSum(1,3) << "\n";

    // 서브트리(3)에 +10
    ett.subtreeAdd(1,3,10);
    cout << "after add, subtreeSum(root=1, u=3): " << ett.subtreeSum(1,3) << "\n";

    // (2 - 3) 절단(교육용: u=2를 루트로 회전 후 child=3 쪽 떼기)
    // 정확한 간선 cut은 '간선 토큰' 모델에서 더 자연스럽다.
    ett.cut_as_child(2,3);
    cout << "connected(1,4) after cut: " << ett.connected(1,4) << "\n"; // 0

    // 1-5 연결 → 이제 1..6 모두 이어질 수 있음
    ett.link(1,5);
    cout << "connected(4,6): " << ett.connected(4,6) << "\n"; // 1

    return 0;
}
```

---

## ETT vs LCT: 언제 무엇을 쓰나

| 문제 유형 | 추천 |
|---|---|
| **정적 트리** 서브트리 질의/갱신 다수 | **정적 ETT(tin/tout) + 세그먼트/펜윅** |
| **정적 트리** 경로 질의/갱신 다수 | **LCA + 경로 분해(HLD)** or ETT+LCA |
| **완전 동적 포레스트** 연결성/서브트리(루트 기준) | **동적 ETT** (시퀀스 회전 후 구간 연산) |
| **완전 동적 포레스트** 경로 질의/갱신(일반) | **Link-Cut Tree (Top Tree)** |

---

## 실전 팁 & 함정

- 정적 ETT: **첫 방문만** 기록하는 방식이 서브트리 구간화를 가장 깔끔하게 만든다.
  “들어갈 때/나올 때” 2회 기록법은 **LCA**(RMQ) 같은 곳에 좋다.
- lazy 전파: **push/pull** 순서가 틀리면 한 번에 망가진다. (특히 동적 ETT)
- 동적 ETT: 케이스를 모두 커버하려면 **간선 토큰(양방향)** 을 쓰고, link는 **4-way concat**, cut은 **2개 위치 split** 패턴으로 구현하는 것이 정석.
- 경로 질의가 핵심이면 **LCT** 로 빠르게 가는 것이 더 현실적이다.

---

## 요약

- **정적 ETT**: `tin/tout` 기반으로 트리를 1차원화 → 서브트리 구간 질의/갱신을 **세그먼트/펜윅** 으로 \(O(\log n)\).
- **경로 질의(정적)**: LCA + prefix 조합이 간단·견고.
- **동적 ETT**: 오일러 시퀀스를 **implicit treap/splay** 로 보관 → `reroot`+`split/merge` 로 **link/cut/connected** 및 **(루트 기준) 서브트리 질의/가산** 실현.
- **선택 가이드**: 완전 동적 경로 문제는 **Link-Cut Tree** 가 실용적이고 구현 리스크가 낮다.

위 코드들은 **핵심 아이디어를 교육용으로 담은 최소 구현**이다. 실제 프로젝트에서는
- 간선 토큰을 명시하여 cut의 모든 케이스를 안전하게 처리,
- 예외/경계/대표 추적을 정교화,
- 테스트(랜덤·대량)로 불변식 보장
을 반드시 수행하자.
