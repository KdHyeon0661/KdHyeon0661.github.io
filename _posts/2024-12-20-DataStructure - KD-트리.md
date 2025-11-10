---
layout: post
title: Data Structure - 펜윅 트리
date: 2024-12-20 19:20:23 +0900
category: Data Structure
---
# KD-트리 (K-Dimensional Tree)

## 1) 개요와 동기

**KD-트리(KD-Tree)** 는 \(K\)차원 점 집합에서 **최근접 이웃(Nearest Neighbor, NN)**, **K-최근접 이웃(KNN)**, **반지름 질의(Range/Radius)**, **직교 범위 질의(Orthogonal Range)** 등을 **평균 \(O(\log n)\)** 으로 처리하기 위해 고안된 **이진 분할 트리**다.  
루트에서 특정 축(예: \(x\)) 기준으로 분할하고, 다음 수준에서는 다른 축(예: \(y\))으로 분할하는 식으로 **축을 순환**하며 내려간다.

- **장점**: 평균적으로 빠른 NN/KNN/범위 질의, 구현 난이도 적당, 실무에서도 충분히 빠름  
- **주의**: 편향 입력(정렬된 순서로 삽입) 시 **퇴화 \(O(n)\)** 가능 → **균형 빌드**(median split)나 **주기적 재빌드** 권장

---

## 2) 핵심 아이디어 & 수학적 직관

각 노드가 한 점 \(\mathbf{p}\in\mathbb{R}^K\)과 분할 축 \(a\in\{0,\dots,K-1\}\) 을 가진다.  
한 노드의 왼쪽 서브트리는 \(\mathbf{x}[a]\le \mathbf{p}[a]\), 오른쪽은 \(\mathbf{x}[a]>\mathbf{p}[a]\) 를 만족하는 점들로 구성한다.

- **거리**: 최근접 탐색은 유클리드 거리(주로 **제곱거리**)로 비교:
  \[
  d^2(\mathbf{x},\mathbf{y})=\sum_{i=0}^{K-1}(x_i-y_i)^2.
  \]
  제곱근을 계산하지 않으므로 빠르고 부동소수 오차가 줄어든다.
- **가지치기(Pruning)**: 질의점 \(\mathbf{q}\) 과 현재 **최적 거리** \(D^2\) 가 있을 때,  
  분할 초평면까지의 거리제곱 \( (q_a - p_a)^2 \ge D^2 \) 이면 **반대쪽 서브트리**는 볼 필요가 없다.  
  더 강력하게, 각 서브트리에 대해 **경계상자(AABB)** 를 유지하면:
  \[
  \mathrm{dist}^2(\mathbf{q}, \mathrm{AABB}) \ge D^2 \ \Rightarrow\ \text{해당 서브트리 전체 가지치기}.
  \]

---

## 3) 자료구조 설계(실전형)

- 점: `array<T, K>` (T는 `float` 또는 `double`)
- 노드: 점, 왼/오른 자식, **분할축**, **AABB**(각 축의 min/max)  
  AABB는 NN/KNN/범위/반지름 질의에서 강력한 가지치기를 제공한다.

---

## 4) 빌드 전략: 균형 KD-트리 (Median Split)

정적 데이터(배치 구축)라면, 각 레벨에서 **해당 축으로 median** 을 선택해 분할하면 평균적으로 **높이가 \(O(\log n)\)** 인 균형 트리를 얻는다.

- 구현: `nth_element` 로 축 기준 **중앙 원소**를 \(O(n)\)에 뽑고 좌/우 재귀 → 전체 빌드 \(O(n \log n)\)
- 분할축 선택: **round-robin**(depth%K) 또는 **분산이 큰 축**(variance 최대)을 선택(가지치기 효율 ↑)

---

## 5) 질의 API 레시피

- **NN**: 한쪽(near) 먼저, 갱신된 \(D^2\)로 far를 **조건부 탐색**(분할평면 거리 or AABB 거리)
- **KNN**: **최대힙**(size≤K)으로 현재 상위 K개 유지, 힙 Top(가장 먼 후보)보다 AABB가 더 가깝지 않으면 가지치기
- **직교 범위**: \([\mathbf{L},\mathbf{H}]\) 직사각형(AABB)과 노드/자식 AABB 교차 테스트 후 내려가며 **포함 점 수집**
- **반지름 질의**: \(d^2(\mathbf{q},\mathbf{x})\le R^2\) 인 점 수집, **AABB-ball** 교차로 가지치기

---

## 6) 정확성 직관(스케치)

최근접 후보 거리가 \(D\)일 때, 분할축에 수직인 초평면/혹은 서브트리 AABB 까지의 **최소 거리**가 \(D\)보다 크면, 그 서브트리 내부의 어떤 점이라도 \(\mathbf{q}\) 보다 더 가까울 수 없다.  
이는 **삼각부등식**과 AABB의 최소거리 정의로 즉시 성립한다:
\[
\mathrm{dist}^2(\mathbf{q}, \mathrm{AABB})=\sum_{i=0}^{K-1}
\begin{cases}
(L_i-q_i)^2 & q_i<L_i\\
(q_i-H_i)^2 & q_i>H_i\\
0 & \text{그 외}
\end{cases}
\]

---

## 7) C++ — 범용 템플릿 KD-트리 (빌드/NN/KNN/범위/반지름/삽입/지연삭제/재빌드)

> **목표**: 실무/대회에서 곧바로 쓰는 **균형 빌드 + 강력 가지치기(AABB) + 동적 삽입/삭제(지연) + 재빌드** 버전.

```cpp
// kd_tree.hpp
#include <bits/stdc++.h>
using namespace std;

template<int K, class T = double>
struct KDTree {
    using Pt = array<T, K>;

    struct Node {
        Pt p, mn, mx;
        int axis;
        bool deleted;
        Node *l, *r;
        Node(const Pt& _p, int a=0): p(_p), mn(_p), mx(_p), axis(a), deleted(false), l(nullptr), r(nullptr) {}
    };

    Node* root = nullptr;
    int alive = 0;          // 삭제되지 않은 점 수
    int tomb = 0;           // deleted=true 수(지연 삭제)
    double rebuild_ratio = 0.3; // 무덤 비율 넘으면 재빌드

    // ==== 유틸 ====
    static inline T sq(T x){ return x*x; }

    static T dist2(const Pt& a, const Pt& b){
        T s=0; for(int i=0;i<K;++i){ T d=a[i]-b[i]; s += d*d; } return s;
    }

    static T dist2_box(const Pt& q, const Pt& mn, const Pt& mx){
        T s=0;
        for(int i=0;i<K;++i){
            if(q[i]<mn[i]) s += sq(mn[i]-q[i]);
            else if(q[i]>mx[i]) s += sq(q[i]-mx[i]);
        }
        return s;
    }

    static void pull(Node* u){
        for(int i=0;i<K;++i){
            u->mn[i] = u->mx[i] = u->p[i];
        }
        if(u->l){
            for(int i=0;i<K;++i){
                u->mn[i] = min(u->mn[i], u->l->mn[i]);
                u->mx[i] = max(u->mx[i], u->l->mx[i]);
            }
        }
        if(u->r){
            for(int i=0;i<K;++i){
                u->mn[i] = min(u->mn[i], u->r->mn[i]);
                u->mx[i] = max(u->mx[i], u->r->mx[i]);
            }
        }
    }

    // ==== 균형 빌드 ====
    Node* build(vector<Pt>& a, int l, int r, int depth){
        if(l>=r) return nullptr;
        int axis = depth % K;
        int m = (l+r)/2;
        nth_element(a.begin()+l, a.begin()+m, a.begin()+r,
            [&](const Pt& x, const Pt& y){ return x[axis] < y[axis]; });
        Node* u = new Node(a[m], axis);
        u->l = build(a, l, m, depth+1);
        u->r = build(a, m+1, r, depth+1);
        pull(u);
        return u;
    }

    KDTree() = default;
    KDTree(const vector<Pt>& pts){ build(pts); }

    void build(const vector<Pt>& pts){
        vector<Pt> a=pts;
        clear(root);
        root = build(a, 0, (int)a.size(), 0);
        alive = (int)pts.size();
        tomb  = 0;
    }

    // ==== 메모리 해제 ====
    void clear(Node* u){
        if(!u) return;
        clear(u->l); clear(u->r); delete u;
    }
    ~KDTree(){ clear(root); }

    // ==== 동적 삽입(편향 가능) ====
    Node* insert(Node* u, const Pt& p, int depth){
        if(!u){ ++alive; return new Node(p, depth%K); }
        int a=u->axis;
        if(p[a] <= u->p[a]) u->l = insert(u->l, p, depth+1);
        else                u->r = insert(u->r, p, depth+1);
        pull(u);
        return u;
    }
    void insert(const Pt& p){
        root = insert(root, p, 0);
        // 선택적으로: 편향 완화 위해 일정 삽입 수마다 재빌드 고려
    }

    // ==== 지연 삭제 ====
    bool erase_mark(Node* u, const Pt& p, int depth=0){
        if(!u) return false;
        int a=u->axis;
        if(equal_pt(u->p, p)){
            if(!u->deleted){ u->deleted=true; --alive; ++tomb; }
            return true;
        }
        if(p[a] <= u->p[a]) return erase_mark(u->l, p, depth+1);
        else                return erase_mark(u->r, p, depth+1);
    }
    // 부동소수 비교 허용 오차
    static bool equal_pt(const Pt& a, const Pt& b, T eps = (T)1e-9){
        for(int i=0;i<K;++i) if(fabsl((long double)(a[i]-b[i]))>eps) return false;
        return true;
    }
    void erase(const Pt& p){
        if(erase_mark(root, p) && tomb>0 && (double)tomb/ (tomb+alive) >= rebuild_ratio){
            vector<Pt> pts; pts.reserve(alive);
            collect(root, pts);
            build(pts); // 균형 재빌드
        }
    }
    void collect(Node* u, vector<Pt>& out){
        if(!u) return;
        if(!u->deleted) out.push_back(u->p);
        collect(u->l,out); collect(u->r,out);
    }

    // ==== NN (1-최근접) ====
    // 반환: (최소제곱거리, 최근접점)
    pair<T, Pt> nearest(const Pt& q) const {
        pair<T, Pt> ans{numeric_limits<T>::infinity(), Pt{}};
        nearest(root, q, ans);
        return ans;
    }
    void nearest(Node* u, const Pt& q, pair<T,Pt>& best) const {
        if(!u) return;
        // AABB 가지치기
        T boxd = dist2_box(q, u->mn, u->mx);
        if(boxd >= best.first) return;

        // 현재 노드
        if(!u->deleted){
            T d = dist2(q, u->p);
            if(d < best.first) best = {d, u->p};
        }

        // 자식 방문 순서: 분할축 기준으로 가까운 쪽 먼저
        Node* nearc = (q[u->axis] <= u->p[u->axis]) ? u->l : u->r;
        Node* farc  = (nearc==u->l)? u->r : u->l;

        nearest(nearc, q, best);
        // far 서브트리 AABB가 개선 가능성 있으면 방문
        if(farc){
            T farBox = dist2_box(q, farc->mn, farc->mx);
            if(farBox < best.first) nearest(farc, q, best);
        }
    }

    // ==== KNN ====
    // 가장 가까운 K개 (거리제곱 오름차순)
    vector<pair<T,Pt>> knearest(const Pt& q, int Kq) const {
        // 최대힙(가장 먼 후보가 top)
        auto cmp = [](const pair<T,Pt>& A, const pair<T,Pt>& B){ return A.first < B.first; };
        priority_queue<pair<T,Pt>, vector<pair<T,Pt>>, decltype(cmp)> heap(cmp);
        knearest(root, q, Kq, heap);
        vector<pair<T,Pt>> out;
        out.reserve(heap.size());
        while(!heap.empty()){ out.push_back(heap.top()); heap.pop(); }
        reverse(out.begin(), out.end());
        return out;
    }
    void knearest(Node* u, const Pt& q, int Kq,
                  priority_queue<pair<T,Pt>, vector<pair<T,Pt>>,
                                 function<bool(const pair<T,Pt>&,const pair<T,Pt>&)>>& heap) const {
        if(!u) return;
        // AABB 가지치기
        T worst = heap.empty()? numeric_limits<T>::infinity() : heap.top().first;
        T boxd = dist2_box(q, u->mn, u->mx);
        if(boxd > worst) return;

        if(!u->deleted){
            T d = dist2(q, u->p);
            if((int)heap.size() < Kq) heap.push({d, u->p});
            else if(d < heap.top().first){ heap.pop(); heap.push({d,u->p}); }
        }
        Node* nearc = (q[u->axis] <= u->p[u->axis]) ? u->l : u->r;
        Node* farc  = (nearc==u->l)? u->r : u->l;

        knearest(nearc, q, Kq, heap);
        // far 가지치기 (최신 worst 사용)
        T worst2 = heap.empty()? numeric_limits<T>::infinity() : heap.top().first;
        if(farc){
            T farBox = dist2_box(q, farc->mn, farc->mx);
            if(farBox <= worst2) knearest(farc, q, Kq, heap);
        }
    }

    // ==== 직교 범위 질의: [L..H] AABB 내부의 점 수집 ====
    vector<Pt> range_rect(const Pt& L, const Pt& H) const {
        vector<Pt> out;
        range_rect(root, L, H, out);
        return out;
    }
    static bool box_disjoint(const Pt& aL, const Pt& aH, const Pt& bL, const Pt& bH){
        for(int i=0;i<K;++i) if(aH[i]<bL[i] || bH[i]<aL[i]) return true;
        return false;
    }
    static bool inside(const Pt& p, const Pt& L, const Pt& H){
        for(int i=0;i<K;++i) if(p[i]<L[i] || p[i]>H[i]) return false;
        return true;
    }
    void range_rect(Node* u, const Pt& L, const Pt& H, vector<Pt>& out) const {
        if(!u) return;
        // u의 AABB가 [L..H]와 전혀 겹치지 않으면 종료
        if(box_disjoint(u->mn, u->mx, L, H)) return;
        if(!u->deleted && inside(u->p,L,H)) out.push_back(u->p);
        range_rect(u->l, L, H, out);
        range_rect(u->r, L, H, out);
    }

    // ==== 반지름 질의: d^2 <= R^2 ====
    vector<Pt> radius(const Pt& q, T R) const {
        vector<Pt> out; T R2=R*R;
        radius(root, q, R2, out);
        return out;
    }
    void radius(Node* u, const Pt& q, T R2, vector<Pt>& out) const {
        if(!u) return;
        if(dist2_box(q, u->mn, u->mx) > R2) return;
        if(!u->deleted && dist2(q, u->p) <= R2) out.push_back(u->p);
        radius(u->l, q, R2, out);
        radius(u->r, q, R2, out);
    }
};
```

### 사용 예시(2D)

```cpp
// demo_kdtree.cpp
#include "kd_tree.hpp"
int main(){
    using K2 = KDTree<2,double>;
    using Pt  = K2::Pt;

    vector<Pt> pts = { Pt{3,6}, Pt{17,15}, Pt{13,15}, Pt{6,12}, Pt{9,1}, Pt{2,7}, Pt{10,19} };
    K2 kd(pts);

    // 1) 직교 범위: [(0,0)..(10,15)]
    Pt L{0,0}, H{10,15};
    auto in = kd.range_rect(L,H);
    cout << "In-rectangle: ";
    for(auto& p: in) cout << "("<<p[0]<<","<<p[1]<<") ";
    cout << "\n"; // (10,19)는 제외되어야 함

    // 2) 1-최근접
    Pt q{10,9};
    auto [d2, best] = kd.nearest(q);
    cout << "NN to (10,9): (" << best[0] << "," << best[1] << "), dist=" << sqrt(d2) << "\n";

    // 3) 3-최근접
    auto KNN = kd.knearest(q, 3);
    cout << "3-NN:\n";
    for(auto& e: KNN){
        cout << "  ("<<e.second[0]<<","<<e.second[1]<<") d="<< sqrt(e.first) << "\n";
    }

    // 4) 반지름 질의 (R=6)
    auto near6 = kd.radius(q, 6.0);
    cout << "Within R=6 of (10,9): ";
    for(auto& p: near6) cout << "("<<p[0]<<","<<p[1]<<") ";
    cout << "\n";

    // 5) 동적 삽입/삭제 + 자동 재빌드
    kd.insert(Pt{8,10});
    kd.insert(Pt{11,8});
    kd.erase(Pt{6,12}); // 지연삭제, 무덤 비율 넘으면 자동 재빌드
    auto [d2b, bestb] = kd.nearest(q);
    cout << "NN after updates: ("<<bestb[0]<<","<<bestb[1]<<")\n";
}
```

---

## 8) 삭제(딥 다이브): 즉시 삭제 vs 지연 삭제

- **즉시 삭제**(classic): 삭제 대상의 분할축 \(a\) 에 대해, **오른쪽 서브트리의 축 \(a\) 최소값**(또는 왼쪽 최대)을 올려 치환 후 재귀 삭제.  
  구현은 가능하지만, **AABB 갱신**과 **균형 유지**가 번거롭다.
- **지연 삭제**(lazy): 노드 플래그 `deleted=true` 만 세팅하고, **무덤 비율**이 임계치를 넘으면 **살아있는 점만 수집 → 재빌드**.  
  실전에서 **간단+빠름**. 위 코드가 이 방식을 구현.

> 데이터가 **많이 변하는 온라인 시스템**이라면 **주기적 재빌드**를 권장(예: 삽입/삭제 합이 n의 30%를 넘었을 때).

---

## 9) 수치 안정성과 팁

- **제곱거리**로 비교해 **sqrt** 피하기  
- `double` 권장(특히 K가 크거나 좌표가 큰 경우)  
- **EPS**(작은 허용오차) 로 점 비교 → `equal_pt`  
- 스케일이 크게 다른 축이 섞이면 **전처리 정규화**(표준화/최대값 스케일링)로 가지치기 효율 상승  
- 입력이 순차적으로 정렬돼 있으면, **삽입형 KD-트리**는 편향 → **배치 빌드** 또는 **주기 재빌드**

---

## 10) 복잡도 정리

| 연산 | 평균 | 최악(퇴화) |
|---|---|---|
| 빌드(균형, median) | \(O(n\log n)\) | — |
| NN / KNN / 범위 / 반지름 | \(O(\log n)\) ~ \(O(n^{1-1/K})\) | \(O(n)\) |
| 삽입(동적) | 평균 \(O(\log n)\) | \(O(n)\) |
| 지연 삭제 마킹 | 평균 \(O(\log n)\) | \(O(n)\) |
| 재빌드 | \(O(n\log n)\) | — |

> 고차원에서는 **차원의 저주**로 \(K\)가 커질수록 KD-트리 이점이 줄어든다(대략 \(K\lesssim 20\) 권장). 그 이상은 **Ball-Tree/Annoy/HNSW** 등 ANN이 실전적.

---

## 11) 변형/확장

- **가중 거리**, **맨해튼 거리** 등도 가능(단, AABB 최소거리 정의만 해당 노름에 맞게 수정)  
- **Best-Bin-First(BBF)**: KNN에서 **우선순위 기반 서치(노드 by AABB 거리)** 로 근사/정확 탐색 속도 ↑  
- **동형 좌표/affine 변환**: 사전 변환 후 KD-트리 구축  
- **멀티스레드 빌드**: 좌/우 재귀 분기 병렬화(주의: 메모리 할당 contention)

---

## 12) KD-트리 vs 다른 공간 트리

| 구조 | 핵심 아이디어 | 강점 | 약점 | 쓰임 |
|---|---|---|---|---|
| **KD-Tree** | 축 순환 분할 | NN/KNN/범위 평균 빠름 | 고차원 취약, 동적 균형 어려움 | 2D/3D/저차원 검색, 로보틱스 |
| **Ball-Tree** | 구체(센터+반지름) 분할 | 고차원 비교적 유리 | 구현 복잡 | ML, 고차원 |
| **VP-Tree** | vantage point 반径 분할 | 임의 거리함수 | 균형/튜닝 필요 | 커스텀 거리 |
| **R-Tree** | 사각형(MBR) 묶음 | 사각형 교차/범위 | NN 최적 아님 | GIS/DB 인덱스 |
| **Oct/Quad-Tree** | 3D/2D 정규 분할 | 그리드 친화 | NN 비효율 | 렌더링/충돌 |

---

## 13) 수학 스냅샷(마크다운 수식)

- 분할평면 가지치기 필요충분은 아니지만 충분조건:
  \[
  (q_a - p_a)^2 \ge D^2 \ \Rightarrow\ \text{반대 서브트리 무시 가능.}
  \]
- AABB 가지치기:
  \[
  \mathrm{dist}^2(\mathbf{q}, \mathrm{AABB}[\mathbf{m},\mathbf{M}]) =
  \sum_{i=0}^{K-1}
  \begin{cases}
  (m_i-q_i)^2 & q_i<m_i\\
  (q_i-M_i)^2 & q_i>M_i\\
  0 & \text{otherwise}
  \end{cases}
  \]
  이 값이 현재 최적 거리제곱 \(D^2\) 이상이면 **서브트리 전체 배제**.

---

## 14) 2D 간단 삽입/삭제/범위/최근접 “교육용” 버전(축만, AABB 없음)

아래는 **분할축/평면 조건**만 이용하는 간단 버전(학습/디버깅에 유익):

```cpp
struct KD2 {
    struct P{ double x,y; };
    struct Node{
        P p; int axis; Node *l,*r; bool del=false;
        Node(P _p,int a):p(_p),axis(a),l(nullptr),r(nullptr){}
    }*root=nullptr;

    static double d2(const P&a,const P&b){ double dx=a.x-b.x, dy=a.y-b.y; return dx*dx+dy*dy; }

    Node* ins(Node* u, P p, int dep){
        if(!u) return new Node(p, dep&1);
        int a=u->axis; double key = (a? p.y : p.x), cut=(a? u->p.y : u->p.x);
        if(key <= cut) u->l = ins(u->l, p, dep+1); else u->r = ins(u->r, p, dep+1);
        return u;
    }
    void insert(P p){ root=ins(root,p,0); }

    void nn(Node* u, P q, pair<double,P>& best){
        if(!u) return;
        if(!u->del){
            double d=d2(q,u->p); if(d<best.first) best={d,u->p};
        }
        int a=u->axis;
        double qv = (a? q.y:q.x), pv=(a? u->p.y:u->p.x);
        Node* nearc = (qv<=pv)? u->l: u->r;
        Node* farc  = (nearc==u->l)? u->r: u->l;
        nn(nearc, q, best);
        if((qv-pv)*(qv-pv) < best.first) nn(farc, q, best); // 평면 기준 가지치기
    }
    pair<double,P> nearest(P q){
        pair<double,P> best{1e300, P{}};
        nn(root,q,best); return best;
    }

    // 지연삭제
    bool eq(P a,P b,double eps=1e-9){ return fabs(a.x-b.x)<eps && fabs(a.y-b.y)<eps; }
    bool del(Node* u,P t){
        if(!u) return false;
        if(eq(u->p,t)){ u->del=true; return true; }
        int a=u->axis; double tv=(a? t.y:t.x), pv=(a? u->p.y:u->p.x);
        if(tv<=pv) return del(u->l,t); else return del(u->r,t);
    }
    void erase(P t){ del(root,t); }

    void range(Node* u, P L, P H, vector<P>& out){
        if(!u) return;
        if(!u->del && u->p.x>=L.x && u->p.x<=H.x && u->p.y>=L.y && u->p.y<=H.y) out.push_back(u->p);
        int a=u->axis; double low=(a? L.y:L.x), high=(a? H.y:H.x), cut=(a? u->p.y:u->p.x);
        if(low<=cut) range(u->l,L,H,out);
        if(high>=cut) range(u->r,L,H,out);
    }
    vector<P> range(P L,P H){ vector<P> out; range(root,L,H,out); return out; }
};
```

---

## 15) 검증·프로파일링 가이드

- **브루트포스**와 결과 일치 테스트(NN/KNN/범위/반지름)  
- 난수 점 1e5~1e6로 **빌드 시간/질의 처리량** 측정  
- AABB 가지치기 **히트율**(방문 노드 수 / 총 노드 수) 로그  
- 입력 분포(격자/균일/클러스터)와 분할축 전략(라운드로빈 vs 분산 최대) 비교

---

## 16) 마무리 요약

- KD-트리는 **저차원 좌표**에서 **NN/KNN/범위/반지름**을 빠르게 처리하는 표준 도구.  
- 정적 데이터는 **median-split 균형 빌드**로, 동적 데이터는 **지연 삭제 + 재빌드**로 실전 운영.  
- **AABB 가지치기**는 성능의 핵심. 수치 안정성을 위해 **제곱거리**와 **EPS** 비교를 권장.  
- 고차원/거대한 데이터셋은 **ANN 구조(Annoy/FAISS/HNSW)** 도 검토.