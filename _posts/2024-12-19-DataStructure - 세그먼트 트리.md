---
layout: post
title: Data Structure - 펜윅 트리
date: 2024-12-19 19:20:23 +0900
category: Data Structure
---
# 세그먼트 트리 (Segment Tree)

## 1) 왜 세그먼트 트리인가?

배열의 **구간 정보(합·최솟값·최댓값·GCD 등)** 를 반복적으로 물어보면서 **중간에 값이 갱신**되기도 하는 문제는 매우 흔하다.  
단순 누적합(prefix)으로는 **질의 O(1)** 이 가능하지만, **갱신이 O(n)** 이라 비효율적이다.  
**세그먼트 트리**는 트리를 이용해 **질의·갱신 모두 O(log n)** 으로 처리한다.

- 지원 연산: 합, 최솟값/최댓값, GCD/LCM(모듈러), 비트 OR/AND, 구조체 결합(예: “합+최솟값+최댓값” 동시 유지) 등  
- 형태: **완전 이진 트리를 배열로 표현**(vector 1-index 또는 0-index)  
- 공간: 일반적으로 **4n** 정도(안전 상수), **bottom-up(반복형)** 은 **2·N(상한 power-of-two)**

---

## 2) 아이디어와 정의 (간단 수식)

길이 \(n\) 배열 \(A[0..n-1]\).  
세그트리는 각 노드가 어떤 구간 \([l,r]\) 의 정보를 보관한다. 루트는 \([0,n-1]\), 왼쪽 자식 \([l,m]\), 오른쪽 자식 \([m+1,r]\) (단, \(m=\lfloor (l+r)/2 \rfloor\)).

- 노드 값(예: 합)의 결합 법칙:
  \[
  \text{node}(l,r) = \text{merge}\big(\text{node}(l,m),\ \text{node}(m+1,r)\big).
  \]
- 질의(예: 합 \([L,R]\))는 \([l,r]\) 노드가 **겹치면 내려가며** 필요한 부분만 합친다.  
- 점 갱신(혹은 구간 갱신)은 해당 원소/구간을 포함하는 **모든 조상 노드**를 갱신한다.

결합 연산 `merge` 가 **결합법칙(associative)** 을 만족하면 세그트리로 효율 처리 가능하다.

---

## 3) 가장 기본: “구간 합 + 점 갱신(덧셈/치환)” (재귀형)

> 아래 구현은 **합 세그트리**. `update_add`(증분)와 `update_set`(치환)을 모두 제공한다.

```cpp
#include <bits/stdc++.h>
using namespace std;

struct SegSum {
    int n;
    vector<long long> t; // tree

    SegSum(const vector<long long>& a) { build(a); }
    SegSum(int n=0) { init(n); }

    void init(int n_) { n=n_; t.assign(4*n, 0); }

    void build(const vector<long long>& a){ init((int)a.size()); build(1,0,n-1,a); }
    void build(int v,int l,int r,const vector<long long>& a){
        if(l==r){ t[v]=a[l]; return; }
        int m=(l+r)/2;
        build(v*2,l,m,a); build(v*2+1,m+1,r,a);
        t[v]=t[v*2]+t[v*2+1];
    }

    // query sum on [L,R]
    long long query(int L,int R){ return query(1,0,n-1,L,R); }
    long long query(int v,int l,int r,int L,int R){
        if(R<l || r<L) return 0;
        if(L<=l && r<=R) return t[v];
        int m=(l+r)/2;
        return query(v*2,l,m,L,R)+query(v*2+1,m+1,r,L,R);
    }

    // point: add delta at idx
    void update_add(int idx,long long delta){ update_add(1,0,n-1,idx,delta); }
    void update_add(int v,int l,int r,int idx,long long d){
        if(l==r){ t[v]+=d; return; }
        int m=(l+r)/2;
        if(idx<=m) update_add(v*2,l,m,idx,d);
        else       update_add(v*2+1,m+1,r,idx,d);
        t[v]=t[v*2]+t[v*2+1];
    }

    // point: set A[idx]=val  (치환)
    void update_set(int idx,long long val){ update_set(1,0,n-1,idx,val); }
    void update_set(int v,int l,int r,int idx,long long val){
        if(l==r){ t[v]=val; return; }
        int m=(l+r)/2;
        if(idx<=m) update_set(v*2,l,m,idx,val);
        else       update_set(v*2+1,m+1,r,idx,val);
        t[v]=t[v*2]+t[v*2+1];
    }
};

int main(){
    vector<long long> a={1,3,5,7,9,11};
    SegSum seg(a);
    cout << seg.query(1,3) << "\n"; // 3+5+7=15
    seg.update_add(1, +2);          // a[1]=3->5
    cout << seg.query(1,3) << "\n"; // 5+5+7=17
    seg.update_set(2, 10);          // a[2]=10
    cout << seg.query(0,5) << "\n"; // 합 확인
}
```

---

## 4) 다양한 연산으로 바꾸기 (모노이드 템플릿)

세그트리는 “**합치는 연산**”만 바꾸면 된다. 예:

- **최솟값**: `merge = min`, 항등원 \(+\infty\)
- **최댓값**: `merge = max`, 항등원 \(-\infty\)
- **GCD/LCM**: `merge = gcd/lcm`
- **비트 OR/AND**: `merge = | / &`

간단 템플릿 예시(최솟값 RMQ):

```cpp
struct SegMin {
    int n;
    const long long INF = (1LL<<60);
    vector<long long> t;

    SegMin(const vector<long long>& a){ build(a); }
    SegMin(int n=0){ init(n); }
    void init(int n_){ n=n_; t.assign(4*n, INF); }

    void build(const vector<long long>& a){ init((int)a.size()); build(1,0,n-1,a); }
    void build(int v,int l,int r,const vector<long long>& a){
        if(l==r){ t[v]=a[l]; return; }
        int m=(l+r)/2;
        build(v*2,l,m,a); build(v*2+1,m+1,r,a);
        t[v]=min(t[v*2], t[v*2+1]);
    }

    long long query(int L,int R){ return query(1,0,n-1,L,R); }
    long long query(int v,int l,int r,int L,int R){
        if(R<l||r<L) return INF;
        if(L<=l && r<=R) return t[v];
        int m=(l+r)/2;
        return min(query(v*2,l,m,L,R), query(v*2+1,m+1,r,L,R));
    }

    void point_set(int idx,long long val){ point_set(1,0,n-1,idx,val); }
    void point_set(int v,int l,int r,int idx,long long val){
        if(l==r){ t[v]=val; return; }
        int m=(l+r)/2;
        if(idx<=m) point_set(v*2,l,m,idx,val);
        else       point_set(v*2+1,m+1,r,idx,val);
        t[v]=min(t[v*2], t[v*2+1]);
    }
};
```

---

## 5) **Lazy Propagation** — 구간 갱신 + 구간 질의

구간에 한 번에 값을 더하는 **range add [L,R]+=v** 와 **range sum** 을 함께 처리하려면,  
“아직 자식으로 밀어내지 않은 갱신”을 **lazy 배열**에 저장해 **O(log n)** 을 유지한다.

```cpp
struct SegLazyAddSum {
    int n;
    vector<long long> t, lazy; // t: sum, lazy: pending add

    SegLazyAddSum(int n=0){ init(n); }
    SegLazyAddSum(const vector<long long>& a){ build(a); }

    void init(int n_){ n=n_; t.assign(4*n,0); lazy.assign(4*n,0); }

    void build(const vector<long long>& a){ init((int)a.size()); build(1,0,n-1,a); }
    void build(int v,int l,int r,const vector<long long>& a){
        if(l==r){ t[v]=a[l]; return; }
        int m=(l+r)/2;
        build(v*2,l,m,a); build(v*2+1,m+1,r,a);
        t[v]=t[v*2]+t[v*2+1];
    }

    void apply(int v,int l,int r,long long add){
        t[v] += add * (r-l+1);
        lazy[v] += add;
    }
    void push(int v,int l,int r){
        if(lazy[v]==0 || l==r) return;
        int m=(l+r)/2;
        apply(v*2,l,m,lazy[v]);
        apply(v*2+1,m+1,r,lazy[v]);
        lazy[v]=0;
    }

    void range_add(int L,int R,long long val){ range_add(1,0,n-1,L,R,val); }
    void range_add(int v,int l,int r,int L,int R,long long val){
        if(R<l||r<L) return;
        if(L<=l && r<=R){ apply(v,l,r,val); return; }
        push(v,l,r);
        int m=(l+r)/2;
        range_add(v*2,l,m,L,R,val);
        range_add(v*2+1,m+1,r,L,R,val);
        t[v]=t[v*2]+t[v*2+1];
    }

    long long range_sum(int L,int R){ return range_sum(1,0,n-1,L,R); }
    long long range_sum(int v,int l,int r,int L,int R){
        if(R<l||r<L) return 0;
        if(L<=l && r<=R) return t[v];
        push(v,l,r);
        int m=(l+r)/2;
        return range_sum(v*2,l,m,L,R)+range_sum(v*2+1,m+1,r,L,R);
    }
};
```

> 확장: “구간 치환(set) + 구간 합”은 **set-lazy가 add-lazy를 덮어쓰는** 규칙이 필요(두 종류 lazy, `hasSet` 플래그). 구현 복잡도가 증가하니 필요할 때만 도입.

---

## 6) **반복형(bottom-up)** 세그트리 — 빠르고 간결한 실전 코드

반복형은 **배열 하나**에 리프를 `base..base+n-1` 에 두고 위로 올린다.  
빠르고 캐시 친화적이며 Codeforces/ICPC에서 자주 쓰인다.

```cpp
struct SegItSum {
    int N;                 // 내부 베이스(2의 거듭제곱)
    vector<long long> t;   // 크기 2*N

    SegItSum(int n=0){ init(n); }
    void init(int n){
        N=1; while(N<n) N<<=1;
        t.assign(2*N,0);
    }
    void build(const vector<long long>& a){
        init((int)a.size());
        for(int i=0;i<(int)a.size();++i) t[N+i]=a[i];
        for(int i=N-1;i>=1;--i) t[i]=t[i<<1]+t[i<<1|1];
    }
    void point_add(int idx,long long d){
        int p=idx+N;
        t[p]+=d;
        for(p>>=1; p>=1; p>>=1) t[p]=t[p<<1]+t[p<<1|1];
    }
    long long range_sum(int l,int r){ // [l,r]
        long long L=0, R=0;
        for(l+=N, r+=N; l<=r; l>>=1, r>>=1){
            if(l&1) L+=t[l++];
            if(!(r&1)) R+=t[r--];
        }
        return L+R;
    }
};
```

---

## 7) **인덱스까지** 알고 싶다면? (예: 최솟값의 위치)

결합 값을 `(value, index)` 로 들고 다니면 된다. tie-break를 정해 주면 안정적.

```cpp
struct NodeMinIdx {
    long long val; int idx;
};
NodeMinIdx merge(NodeMinIdx a, NodeMinIdx b){
    if(a.val!=b.val) return (a.val<b.val)?a:b;
    return (a.idx<b.idx)?a:b; // 같은 값이면 더 왼쪽
}
```

---

## 8) 세그트리로 **이분 탐색**: “최초로 누적합 ≥ K” 위치 찾기

합 세그트리에서 루트부터 내려가며 **왼쪽 합 ≥ K ? 왼쪽으로 : K-=왼쪽합, 오른쪽으로**.

```cpp
// a[]는 비음수라고 가정(누적이 단조 증가)
int first_prefix_ge(const SegItSum& seg, long long K){
    if(seg.t[1] < K) return -1; // 전체 합 부족
    int p=1;
    while(p < seg.N){
        long long left = seg.t[p<<1];
        if(left >= K) p=p<<1;
        else { K -= left; p=p<<1|1; }
    }
    return p - seg.N; // 리프 인덱스
}
```

---

## 9) **좌표가 10^9**? → 동적 세그트리 / 좌표 압축

- **좌표 압축**: 질의에 등장하는 좌표만 압축해 `0..M-1` 로 만들고 일반 세그트리를 쓴다.  
- **동적 세그트리**: [0, 1e9) 같은 큰 범위를 **필요한 노드만 동적 할당**.

```cpp
struct DynSeg {
    struct Node { long long sum; Node* L; Node* R; Node():sum(0),L(nullptr),R(nullptr){} };
    Node* root=nullptr;
    static const int64_t LO=0, HI=1'000'000'000; // [LO,HI)

    void add(Node*& cur,int64_t l,int64_t r,int64_t idx,long long d){
        if(!cur) cur=new Node();
        if(l+1==r){ cur->sum += d; return; }
        int64_t m=(l+r)>>1;
        if(idx<m) add(cur->L,l,m,idx,d);
        else      add(cur->R,m,r,idx,d);
        cur->sum = (cur->L?cur->L->sum:0) + (cur->R?cur->R->sum:0);
    }
    long long sum(Node* cur,int64_t l,int64_t r,int64_t ql,int64_t qr){
        if(!cur || qr<=l || r<=ql) return 0;
        if(ql<=l && r<=qr) return cur->sum;
        int64_t m=(l+r)>>1;
        return sum(cur->L,l,m,ql,qr)+sum(cur->R,m,r,ql,qr);
    }

    void add(int64_t idx,long long d){ add(root,LO,HI,idx,d); }
    long long range_sum(int64_t l,int64_t r){ return sum(root,LO,HI,l,r); } // [l,r)
};
```

---

## 10) **퍼시스턴트(Immutable) 세그트리** — 버전별 쿼리

배열의 상태가 시점마다 바뀌는 **버전 K에서의 질의**가 필요하면,  
갱신할 때 **경로상의 노드만 새로 만들어** 루트 포인터를 보관한다(나머지는 공유).  
대표 응용: **구간 K번째 수**, **구간 내 ≤ x 개수** 등(오프라인).

아래는 “버전별 **빈도 누적**” 간단 형태(카운트 트리):

```cpp
struct PST {
    struct Node{ int sum; Node* L; Node* R; Node(int s=0,Node*L=nullptr,Node*R=nullptr):sum(s),L(L),R(R){} };
    int N; // 좌표 수(압축 뒤)
    vector<Node*> ver; // 각 버전 루트

    PST(int n=0):N(n){ ver.clear(); ver.push_back(build(0,N)); }

    Node* build(int l,int r){
        if(l+1==r) return new Node(0);
        int m=(l+r)/2;
        return new Node(0, build(l,m), build(m,r));
    }

    Node* update(Node* cur,int l,int r,int idx,int d){
        if(l+1==r) return new Node(cur->sum + d, nullptr, nullptr);
        int m=(l+r)/2;
        if(idx<m) {
            Node* L = update(cur->L,l,m,idx,d);
            return new Node(L->sum + cur->R->sum, L, cur->R);
        } else {
            Node* R = update(cur->R,m,r,idx,d);
            return new Node(cur->L->sum + R->sum, cur->L, R);
        }
    }

    // 새 버전 생성: 이전 버전 v 기반으로 idx 위치에 +d
    int new_version(int v,int idx,int d){
        Node* nv = update(ver[v],0,N,idx,d);
        ver.push_back(nv);
        return (int)ver.size()-1;
    }

    // [0..idx] 합 (버전 v)
    int prefix(Node* cur,int l,int r,int idx){
        if(!cur) return 0;
        if(idx<l) return 0;
        if(r-1<=idx) return cur->sum;
        int m=(l+r)/2;
        return prefix(cur->L,l,m,idx) + prefix(cur->R,m,r,idx);
    }
    int prefix(int v,int idx){ return prefix(ver[v],0,N,idx); }
};
```

응용:
- 값들을 좌표 압축해 **삽입 시 1 증가** 버전을 만들어 두면,  
  “**[L,R]에서 ≤ x 개수**” = `prefix(ver[R], rank(x)) - prefix(ver[L-1], rank(x))`.  
- 이를 이용해 **[L,R] K번째 값**은 이분 탐색(압축 값 범위)으로 찾는다.

---

## 11) 2D 세그트리(스케치)

2D 배열에서 직사각형 합/최솟값 등은  
1D 트리를 **행마다** 두거나, **세그트리의 각 노드에 또 하나의 세그트리**를 얹는 기법으로 처리한다(공간·구현 난도 높음).  
대부분의 실전은 **2D 펜윅**이나 **오프라인 스위핑 + 1D 세그**로 해결하는 편이 단순/효율적이다.

---

## 12) 정확성 스케치

- 루트가 \([0,n-1]\) 전체를 표현하고, 모든 \([l,r]\) 에 대해
  \[
  \text{merge}\big(\text{node}(l,m),\text{node}(m+1,r)\big)=\text{node}(l,r)
  \]
  가 성립하므로, 임의의 질의 \([L,R]\)는 **겹치는 부분들의 분할정복 합성**으로 정확히 계산된다.  
- Lazy의 정확성: 자식으로 미루지 않은 증가량 \(\Delta\) 를 저장해 두었다가 필요 시 `push()` 로 분배하면,  
  노드 값은 항상 “**현재 구간 길이 × 증가량**”만큼 보정되어 불변식 유지:
  \[
  \text{node.sum} = \sum A[i] + \Delta \cdot |[l,r]|.
  \]

---

## 13) 복잡도·메모리·실전 팁

- 시간: **질의 O(log n)**, **점/구간 갱신 O(log n)**, **빌드 O(n)**
- 공간: 재귀형 약 **4n**; 반복형 **2·N (N=파워오브투)**  
- 실전 팁
  1. **자료형**: 합은 `long long`. 모듈러가 필요하면 노드·lazy에 `%MOD` 적용.
  2. **경계**: 질의 [L,R] 포함 범위를 명확히(0-based/1-based 혼용 금지).
  3. **Lazy**: `push()` 호출 타이밍(질의/부분겹침 갱신 시) 실수 주의.
  4. **함수형/동적**: new/delete 관리 대신 **풀(pool) 할당**을 쓰면 빠르고 안전.
  5. **성능**: 반복형(bottom-up)이 상수 계수 유리. 다만 구간 갱신은 재귀+lazy가 코드가 직관적.

---

## 14) 통합 데모

```cpp
#include <bits/stdc++.h>
using namespace std;

// 1) 합 + 점 갱신/질의
// 2) Lazy: 구간 더하기 + 구간 합
// 3) 반복형(bottom-up) 합

struct SegSum {
    int n; vector<long long> t;
    SegSum(const vector<long long>& a){ build(a); }
    SegSum(int n=0){ init(n); }
    void init(int n_){ n=n_; t.assign(4*n,0); }
    void build(const vector<long long>& a){ init((int)a.size()); build(1,0,n-1,a); }
    void build(int v,int l,int r,const vector<long long>& a){
        if(l==r){ t[v]=a[l]; return; }
        int m=(l+r)/2; build(v*2,l,m,a); build(v*2+1,m+1,r,a); t[v]=t[v*2]+t[v*2+1];
    }
    long long query(int L,int R){ return query(1,0,n-1,L,R); }
    long long query(int v,int l,int r,int L,int R){
        if(R<l||r<L) return 0;
        if(L<=l&&r<=R) return t[v];
        int m=(l+r)/2; return query(v*2,l,m,L,R)+query(v*2+1,m+1,r,L,R);
    }
    void point_add(int idx,long long d){ point_add(1,0,n-1,idx,d); }
    void point_add(int v,int l,int r,int idx,long long d){
        if(l==r){ t[v]+=d; return; }
        int m=(l+r)/2; if(idx<=m) point_add(v*2,l,m,idx,d); else point_add(v*2+1,m+1,r,idx,d);
        t[v]=t[v*2]+t[v*2+1];
    }
};

struct SegLazyAddSum {
    int n; vector<long long> t,lz;
    SegLazyAddSum(int n=0){ init(n); }
    SegLazyAddSum(const vector<long long>& a){ build(a); }
    void init(int n_){ n=n_; t.assign(4*n,0); lz.assign(4*n,0); }
    void build(const vector<long long>& a){ init((int)a.size()); build(1,0,n-1,a); }
    void build(int v,int l,int r,const vector<long long>& a){
        if(l==r){ t[v]=a[l]; return; }
        int m=(l+r)/2; build(v*2,l,m,a); build(v*2+1,m+1,r,a); t[v]=t[v*2]+t[v*2+1];
    }
    void apply(int v,int l,int r,long long add){ t[v]+=add*(r-l+1); lz[v]+=add; }
    void push(int v,int l,int r){
        if(lz[v]==0||l==r) return;
        int m=(l+r)/2; apply(v*2,l,m,lz[v]); apply(v*2+1,m+1,r,lz[v]); lz[v]=0;
    }
    void range_add(int L,int R,long long val){ range_add(1,0,n-1,L,R,val); }
    void range_add(int v,int l,int r,int L,int R,long long val){
        if(R<l||r<L) return;
        if(L<=l&&r<=R){ apply(v,l,r,val); return; }
        push(v,l,r); int m=(l+r)/2;
        range_add(v*2,l,m,L,R,val); range_add(v*2+1,m+1,r,L,R,val);
        t[v]=t[v*2]+t[v*2+1];
    }
    long long range_sum(int L,int R){ return range_sum(1,0,n-1,L,R); }
    long long range_sum(int v,int l,int r,int L,int R){
        if(R<l||r<L) return 0;
        if(L<=l&&r<=R) return t[v];
        push(v,l,r); int m=(l+r)/2;
        return range_sum(v*2,l,m,L,R)+range_sum(v*2+1,m+1,r,L,R);
    }
};

struct SegItSum {
    int N; vector<long long> t;
    void init(int n){ N=1; while(N<n) N<<=1; t.assign(2*N,0); }
    void build(const vector<long long>& a){
        init((int)a.size());
        for(int i=0;i<(int)a.size();++i) t[N+i]=a[i];
        for(int i=N-1;i>=1;--i) t[i]=t[i<<1]+t[i<<1|1];
    }
    void point_add(int idx,long long d){
        int p=idx+N; t[p]+=d; for(p>>=1;p>=1;p>>=1) t[p]=t[p<<1]+t[p<<1|1];
    }
    long long range_sum(int l,int r){ long long L=0,R=0;
        for(l+=N,r+=N;l<=r;l>>=1,r>>=1){ if(l&1) L+=t[l++]; if(!(r&1)) R+=t[r--]; }
        return L+R;
    }
};

int first_prefix_ge(const SegItSum& seg,long long K){
    if(seg.t[1]<K) return -1;
    int p=1;
    while(p<seg.N){
        long long left=seg.t[p<<1];
        if(left>=K) p=p<<1;
        else { K-=left; p=p<<1|1; }
    }
    return p-seg.N;
}

int main(){
    vector<long long> a={1,3,5,7,9,11};
    // 1) 재귀 합
    SegSum s1(a);
    cout << "sum[1,3]=" << s1.query(1,3) << "\n";
    s1.point_add(1,+2);
    cout << "sum[1,3] after +2 at idx1=" << s1.query(1,3) << "\n";

    // 2) lazy: range add + range sum
    SegLazyAddSum sl(a);
    sl.range_add(1,4, +10);
    cout << "lazy sum[0,5]=" << sl.range_sum(0,5) << "\n";
    cout << "lazy sum[3,4]=" << sl.range_sum(3,4) << "\n";

    // 3) 반복형
    SegItSum si; si.build(a);
    cout << "it sum[0,5]=" << si.range_sum(0,5) << "\n";
    si.point_add(2, +5);
    cout << "it sum[0,5] after +5@2=" << si.range_sum(0,5) << "\n";

    // 4) prefix >= K 위치
    cout << "first idx with prefix>=16: " << first_prefix_ge(si, 16) << "\n";
}
```

---

## 15) 세그트리 vs 펜윅 (요약)

| 항목 | 세그먼트 트리 | 펜윅 트리 |
|---|---|---|
| 연산 다양성 | 매우 높음(임의 모노이드, 구조체) | 주로 합/카운트 |
| 구간 갱신 | Lazy로 자연스럽게 지원 | 변형은 가능하나 복잡(2 BIT 등) |
| 구현 난도 | 다소 높음(특히 Lazy) | 낮음 |
| 상수 계수 | 큼 | 작음 |
| 메모리 | ~4n (재귀), 2N(반복형) | ~n |

---

## 16) 체크리스트 & 퍼징

- 0/1-based 혼동 금지, 포함 범위 `[L..R]` 일관성  
- `long long` 사용(합·곱·누적)  
- Lazy: `apply`와 `push`의 규칙(덧셈/치환 우선순위) 명확화  
- 퍼징 예: 브루트포스(직접 배열)와 세그트리를 난수 업데이트/질의로 수천 번 대조

간단 퍼징 스케치:

```cpp
// 난수로 point_add/range_sum 교차 실행
// 브루트 배열로도 같은 연산 수행 후 결과 비교
```

---

## 17) 결론

- 세그먼트 트리는 **다양한 구간 연산**을 **질의·갱신 모두 O(log n)** 으로 처리하는 강력한 범용 자료구조다.  
- 문제 성격이 **합/카운트 중심**이고 간단하다면 펜윅이 더 가볍다.  
- **구간 갱신, 복합 정보(값+인덱스), 이분 탐색, 동적/퍼시스턴트** 등 고급 수요가 있다면 세그먼트 트리가 확실한 해답이다.