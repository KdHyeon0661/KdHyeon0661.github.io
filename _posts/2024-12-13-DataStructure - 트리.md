---
layout: post
title: Data Structure - 트리
date: 2024-12-13 19:20:23 +0900
category: Data Structure
---
# 트리(Tree)

## 0. 수학적 정의(스냅샷)

- **트리** \(T=(V,E)\): **연결**이고 **사이클 없음**. \(|E|=|V|-1\).
- 임의 루트 \(r\)를 잡으면 **부모/자식**·**깊이/높이**가 정의된다.
- 트리의 **높이** \(h\)는 루트에서 가장 먼 리프까지의 간선 수.
- 일반 트리는 각 노드의 자식 수가 **무제한(가변)**.

---

## 1. 구현 전략 1: `vector<TreeNode*>` — 직관·STL 친화

### 1.1 기본 구조

```cpp
```cpp
#include <bits/stdc++.h>
using namespace std;

struct NodeV {
    int data;
    vector<NodeV*> children;  // N-ary 자식
    explicit NodeV(int v) : data(v) {}
};

void addChild(NodeV* parent, NodeV* child) {
    parent->children.push_back(child);
}
```
```

- 장점: **직관적**, STL과 잘 맞음, 순회/조작이 간단
- 단점: 각 노드가 **가변 길이 컨테이너**(vector)를 가져서 **할당 단편화**·캐시 비우호 가능

### 1.2 출력/순회(전위 DFS)

```cpp
```cpp
void printTree(const NodeV* u, int depth=0){
    if (!u) return;
    cout << string(depth*2, '-') << u->data << "\n";
    for (auto* ch : u->children) printTree(ch, depth+1);
}
```
```

### 1.3 반복(스택 기반, 전위)

```cpp
```cpp
void printTreeIter(const NodeV* root){
    if (!root) return;
    vector<pair<const NodeV*,int>> st; // (node, depth)
    st.push_back({root,0});
    while(!st.empty()){
        auto [u,d]=st.back(); st.pop_back();
        cout << string(d*2,'-') << u->data << "\n";
        // 전위: children을 역순 push → 왼쪽부터 방문
        for (int i=(int)u->children.size()-1; i>=0; --i)
            st.push_back({u->children[i], d+1});
    }
}
```
```

### 1.4 레벨 순회(BFS)

```cpp
```cpp
void levelOrder(const NodeV* root){
    if (!root) return;
    queue<pair<const NodeV*,int>> q;
    q.push({root,0});
    int cur = 0;
    while(!q.empty()){
        auto [u,d]=q.front(); q.pop();
        if (d!=cur){ cout << "\n"; cur=d; }
        cout << u->data << " ";
        for (auto* ch : u->children) q.push({ch,d+1});
    }
    cout << "\n";
}
```
```

### 1.5 유틸: 높이/리프 수/서브트리 크기

```cpp
```cpp
int height(const NodeV* u){
    if (!u) return -1; // 공백 트리 관례
    int h=-1;
    for (auto* ch : u->children) h = max(h, height(ch));
    return h+1;
}
int leafCount(const NodeV* u){
    if (!u) return 0;
    if (u->children.empty()) return 1;
    int s=0; for (auto* ch : u->children) s += leafCount(ch); return s;
}
int subtreeSize(const NodeV* u){
    if (!u) return 0;
    int s=1; for (auto* ch : u->children) s += subtreeSize(ch); return s;
}
```
```

---

## 2. 구현 전략 2: Child–Sibling 표현 — 이진화 트릭

**모든 일반 트리**는 다음의 두 포인터로 **이진 트리**처럼 표현 가능:

- `firstChild`: 첫 번째 자식
- `nextSibling`: 동일 부모의 다음 형제

```cpp
```cpp
struct NodeCS {
    int data;
    NodeCS* firstChild{nullptr};
    NodeCS* nextSibling{nullptr};
    explicit NodeCS(int v): data(v) {}
};

void addChild(NodeCS* parent, NodeCS* child){
    if (!parent->firstChild) { parent->firstChild=child; return; }
    NodeCS* c = parent->firstChild;
    while (c->nextSibling) c=c->nextSibling;
    c->nextSibling = child;
}
```
```

- 장점: 각 노드 **두 포인터** → **메모리 예측 가능**, 연결형 연산에 유리
- 단점: 모든 자식 접근 시 **리스트 순회** 필요(랜덤 접근 비우호)

### 2.1 전위 출력(Child–Sibling)

```cpp
```cpp
void printTreeCS(const NodeCS* u, int depth=0){
    if (!u) return;
    cout << string(depth*2,'-') << u->data << "\n";
    // 자식들
    for (auto* c=u->firstChild; c; c=c->nextSibling)
        printTreeCS(c, depth+1);
}
```
```

### 2.2 레벨 순회(BFS)

```cpp
```cpp
void levelOrderCS(const NodeCS* root){
    if (!root) return;
    queue<pair<const NodeCS*,int>> q;
    q.push({root,0}); int cur=0;
    while(!q.empty()){
        auto [u,d]=q.front(); q.pop();
        if (d!=cur){ cout << "\n"; cur=d; }
        cout << u->data << " ";
        for (auto* c=u->firstChild; c; c=c->nextSibling)
            q.push({c,d+1});
    }
    cout << "\n";
}
```
```

---

## 3. 두 표현의 **상호 변환**

### 3.1 Vector → Child–Sibling

```cpp
```cpp
NodeCS* convertVtoCS(const NodeV* u){
    if (!u) return nullptr;
    NodeCS* r = new NodeCS(u->data);
    NodeCS* prev=nullptr;
    for (auto* ch : u->children){
        NodeCS* c = convertVtoCS(ch);
        if (!r->firstChild) r->firstChild = c;
        else prev->nextSibling = c;
        prev = c;
    }
    return r;
}
```
```

### 3.2 Child–Sibling → Vector

```cpp
```cpp
NodeV* convertCStoV(const NodeCS* u){
    if (!u) return nullptr;
    NodeV* r = new NodeV(u->data);
    for (auto* c=u->firstChild; c; c=c->nextSibling)
        r->children.push_back(convertCStoV(c));
    return r;
}
```
```

- **복잡도**: 두 변환 모두 **O(n)**

---

## 4. 빌더: 입력으로부터 트리 만들기

### 4.1 부모 배열(parent[i] = 부모 인덱스, root의 parent=-1)

```cpp
```cpp
vector<NodeV*> buildFromParent(const vector<int>& parent){
    int n=(int)parent.size();
    vector<NodeV*> nodes(n,nullptr);
    for(int i=0;i<n;++i) nodes[i]=new NodeV(i);
    int root=-1;
    for(int i=0;i<n;++i){
        if (parent[i]==-1) { root=i; continue; }
        nodes[parent[i]]->children.push_back(nodes[i]);
    }
    // 반환은 루트 노드 하나면 충분하나, 여기선 전체 반환
    return nodes; // nodes[root]가 루트
}
```
```

### 4.2 간선 리스트(부모→자식)에서 루트 추정

```cpp
```cpp
NodeV* buildFromEdges(int n, const vector<pair<int,int>>& edges){
    vector<NodeV*> nodes(n); for(int i=0;i<n;++i) nodes[i]=new NodeV(i);
    vector<int> indeg(n,0);
    for(auto [p,c]: edges){ nodes[p]->children.push_back(nodes[c]); indeg[c]++; }
    int root=0; while(root<n && indeg[root]) ++root; // 간단 루트 추정
    return (root<n)? nodes[root] : nullptr;
}
```
```

---

## 5. 순회 패턴/Iterator

### 5.1 전위/후위/레벨(개념)

- **전위(Preorder)**: `node → children`
- **후위(Postorder)**: `children → node`
- **레벨(Level-order)**: BFS

### 5.2 후위(삭제/집계에 유용)

```cpp
```cpp
void postorder(const NodeV* u){
    if (!u) return;
    for (auto* ch : u->children) postorder(ch);
    // u 처리
}
```
```

### 5.3 전위 반복자(간단 구현)

```cpp
```cpp
struct PreorderIter {
    vector<NodeV*> st;
    explicit PreorderIter(NodeV* root){ if(root) st.push_back(root); }
    NodeV* next(){
        if (st.empty()) return nullptr;
        NodeV* u=st.back(); st.pop_back();
        for (int i=(int)u->children.size()-1;i>=0;--i) st.push_back(u->children[i]);
        return u;
    }
};
```
```

---

## 6. 트리 알고리즘: 필수 루틴

### 6.1 트리 지름(Diameter) — 임의 루트 DFS 2회

> 지름: 두 노드 사이의 **최장 경로** 길이(간선 수).
> 1) 임의 루트에서 가장 먼 A, 2) A에서 가장 먼 B, 3) dist(A,B)가 지름.

```cpp
```cpp
pair<int,int> farthest(const NodeV* u){
    // (dist, node)
    // 일반 트리에서 간선 가중치=1, children 방향만 있으면 부모 추적 필요
    // 여기선 parent를 인자로 전달하지 않는 구조: 보조 그래프로 변환 또는 Euler tour.
    // 간단화를 위해 NodeV*를 index화했다고 가정하거나, 아래처럼 '부모 전달' DFS 사용.
    return {0, u->data}; // 자리표시자. 실제로는 인접 리스트가 편리.
}
```
```

일반 트리(NodeV)만으로 지름을 정확하게 하려면 **부모 포인터**가 없으므로
**양방향 인접 리스트**를 만드는 편이 안전/보편적이다.

```cpp
```cpp
// NodeV 트리를 adjacency로 풀어 지름 계산
void buildAdj(const NodeV* u, vector<vector<int>>& g){
    for (auto* ch : u->children){
        g[u->data].push_back(ch->data);
        g[ch->data].push_back(u->data);
        buildAdj(ch, g);
    }
}

pair<int,int> bfs_far(const vector<vector<int>>& g, int s){
    int n=g.size(); vector<int> dist(n,-1); queue<int> q;
    dist[s]=0; q.push(s); int far=s;
    while(!q.empty()){
        int u=q.front(); q.pop();
        if (dist[u]>dist[far]) far=u;
        for(int v:g[u]) if(dist[v]==-1){ dist[v]=dist[u]+1; q.push(v); }
    }
    return {far, dist[far]};
}

int treeDiameter(const NodeV* root, int n){
    vector<vector<int>> g(n);
    buildAdj(root, g);
    auto [a,_] = bfs_far(g, root->data);
    auto [b,D] = bfs_far(g, a);
    return D;
}
```
```

### 6.2 Euler Tour(전위/후위 타임스탬프) — 서브트리 질의 기반

```cpp
```cpp
void eulerTour(const NodeV* u, vector<int>& in, vector<int>& out, int& t){
    if(!u) return;
    in[u->data]=++t;              // 진입
    for(auto* ch: u->children) eulerTour(ch, in, out, t);
    out[u->data]=t;               // 이 노드의 서브트리 종료 시각
}
// 성질: v가 u의 서브트리라면  in[u] <= in[v] <= out[u]
```
```

### 6.3 LCA(개요)

- 일반 트리 루트 기준 **LCA**(최소 공통 조상)는 Euler tour + RMQ, 또는 바이너리 리프팅으로 \(O(\log n)\).
- 여기서는 **개요**만: 실제 구현은 별도 챕터 추천.

---

## 7. 직렬화/역직렬화

### 7.1 괄호 표기(전위): `value(children...)` 형식

예:
`1(2() 3(4(5())))` ← 공백은 가독용

간단 직렬화(전위):

```cpp
```cpp
void serialize(const NodeV* u, ostream& os){
    if(!u){ os << "# "; return; } // #=null sentinel(선택)
    os << u->data << " ( ";
    for(auto* ch: u->children) serialize(ch, os);
    os << ") ";
}
```
```

역직렬화는 파서가 필요. 실무에서는 **JSON** 혹은 **Protocol Buffers** 권장.

---

## 8. 메모리/소유권/예외 안정성

- **Row 포인터 원시 관리**(new/delete)는 **소유권 불명확**·예외에 취약 → **스마트 포인터 권장**.
- `unique_ptr` 기반(단일 소유) 일반 트리:

```cpp
```cpp
struct NodeU {
    int data;
    vector<unique_ptr<NodeU>> children;
    explicit NodeU(int v): data(v) {}
};

void addChild(unique_ptr<NodeU>& parent, unique_ptr<NodeU> child){
    parent->children.push_back(std::move(child));
}
```
```

- **장점**: 소유권이 **부모→자식 단일 경로**, 파괴가 자동
- **주의**: 순환 참조는 `shared_ptr`/`weak_ptr` 설계 필요(일반 트리는 순환 없음)

---

## 9. 성능 관점: 캐시/할당/순회 비용

- `vector<Node*>`는 자식 벡터가 **분산 할당** → 캐시 미스 증가 가능
  → **풀 할당**/**arena**(메모리 풀)로 **연속 할당**시 유리
- child–sibling은 각 노드 2포인터 고정 → **메모리 예측성** 높음,
  하지만 **형제 순회**가 리스트형이라 **무작위 인덱싱**은 비효율
- 재귀 DFS는 깊이 \(h\)가 큰 입력에서 스택 한계 → **반복(스택)**으로 대체

---

## 10. 실전 예시: 파일 시스템(간단), 씬 그래프

### 10.1 파일 시스템(디렉토리=내부 노드, 파일=리프)

```cpp
```cpp
struct File {
    string name; bool isDir=false;
    vector<unique_ptr<File>> children;
};

unique_ptr<File> makeFile(string n, bool d){ auto p=make_unique<File>(); p->name=n; p->isDir=d; return p; }

void add(unique_ptr<File>& dir, unique_ptr<File> f){ dir->children.push_back(std::move(f)); }

void printFS(const File* f, int depth=0){
    cout << string(depth*2,' ') << (f->isDir?"[D] ":"[F] ") << f->name << "\n";
    if (f->isDir) for (auto& ch : f->children) printFS(ch.get(), depth+1);
}
```
```

### 10.2 씬 그래프(트랜스폼 합성: 전위 순회로 누적)

```cpp
```cpp
struct Mat { /* 4x4 변환 행렬, 여기선 생략 */ };
Mat mul(const Mat& A, const Mat& B); // 행렬 곱

struct SceneNode {
    Mat local, world;
    vector<unique_ptr<SceneNode>> children;
};

void updateWorld(SceneNode* u, const Mat& parentWorld){
    u->world = mul(parentWorld, u->local);
    for (auto& ch : u->children) updateWorld(ch.get(), u->world);
}
```
```

---

## 11. 종합 예제: 두 표현 모두 제공 + 구조 변환 + 기본 알고리즘

```cpp
```cpp
#include <bits/stdc++.h>
using namespace std;

// --------- Vector-based ----------
struct NodeV {
    int data;
    vector<NodeV*> children;
    explicit NodeV(int v): data(v) {}
};
void addChild(NodeV* p, NodeV* c){ p->children.push_back(c); }
void printV(const NodeV* u, int d=0){
    if(!u) return;
    cout << string(d*2,'-') << u->data << "\n";
    for(auto* ch: u->children) printV(ch, d+1);
}

// --------- Child–Sibling ----------
struct NodeCS {
    int data;
    NodeCS* firstChild{nullptr};
    NodeCS* nextSibling{nullptr};
    explicit NodeCS(int v): data(v) {}
};
void addChildCS(NodeCS* p, NodeCS* c){
    if(!p->firstChild) { p->firstChild=c; return; }
    NodeCS* t=p->firstChild; while(t->nextSibling) t=t->nextSibling; t->nextSibling=c;
}
void printCS(const NodeCS* u, int d=0){
    if(!u) return;
    cout << string(d*2,'-') << u->data << "\n";
    for(auto* c=u->firstChild;c;c=c->nextSibling) printCS(c, d+1);
}

// --------- Conversion ----------
NodeCS* toCS(const NodeV* u){
    if(!u) return nullptr;
    NodeCS* r=new NodeCS(u->data);
    NodeCS* prev=nullptr;
    for(auto* ch: u->children){
        NodeCS* c = toCS(ch);
        if(!r->firstChild) r->firstChild=c;
        else prev->nextSibling=c;
        prev=c;
    }
    return r;
}
NodeV* toV(const NodeCS* u){
    if(!u) return nullptr;
    NodeV* r=new NodeV(u->data);
    for(auto* c=u->firstChild;c;c=c->nextSibling) r->children.push_back(toV(c));
    return r;
}

// --------- Utilities ----------
int heightV(const NodeV* u){
    if(!u) return -1;
    int h=-1; for(auto* ch: u->children) h=max(h, heightV(ch)); return h+1;
}
int leavesV(const NodeV* u){
    if(!u) return 0;
    if(u->children.empty()) return 1;
    int s=0; for(auto* ch: u->children) s+=leavesV(ch); return s;
}
void buildAdj(const NodeV* u, vector<vector<int>>& g){
    for(auto* ch: u->children){
        g[u->data].push_back(ch->data);
        g[ch->data].push_back(u->data);
        buildAdj(ch,g);
    }
}
pair<int,int> bfsFar(const vector<vector<int>>& g, int s){
    int n=g.size(); vector<int> dist(n,-1); queue<int> q; dist[s]=0; q.push(s); int far=s;
    while(!q.empty()){
        int u=q.front(); q.pop();
        if(dist[u]>dist[far]) far=u;
        for(int v:g[u]) if(dist[v]==-1){ dist[v]=dist[u]+1; q.push(v); }
    }
    return {far, dist[far]};
}
int diameterV(const NodeV* root, int n){
    vector<vector<int>> g(n);
    buildAdj(root,g);
    auto [a,_] = bfsFar(g, root->data);
    auto [b,D] = bfsFar(g, a);
    return D;
}

int main(){
    // Build: Vector-based
    auto *r=new NodeV(1);
    auto *n2=new NodeV(2), *n3=new NodeV(3), *n4=new NodeV(4), *n5=new NodeV(5);
    addChild(r,n2); addChild(r,n3); addChild(n3,n4); addChild(n4,n5);

    cout<<"[Vector]\n"; printV(r);
    cout<<"height="<<heightV(r)<<", leaves="<<leavesV(r)<<"\n";
    cout<<"diameter="<<diameterV(r, 6)<<"\n\n";

    // Convert to Child–Sibling
    NodeCS* rc = toCS(r);
    cout<<"[Child–Sibling]\n"; printCS(rc);

    // Back to Vector
    NodeV* r2 = toV(rc);
    cout<<"[Round-trip Vector]\n"; printV(r2);

    // (주의: 메모리 해제 생략. 실제에선 소유권 정책/스마트포인터 사용 권장)
    return 0;
}
```
```

---

## 12. 복잡도 및 선택 가이드

| 항목 | `vector<Node*>` | Child–Sibling |
|---|---|---|
| 메모리 | 자식 벡터(가변) + 포인터 | 노드당 2포인터(고정) |
| 자식 접근 | 인덱스 O(1), 임의 접근 우수 | **순차 탐색** 필요 |
| 순회 DFS/BFS | O(n) | O(n) |
| 변환 | ⇄ O(n) | ⇄ O(n) |
| 캐시/할당 | 분산 할당 → 캐시 미스 | 포인터 추적형, 예측성 ↑ |
| 추천 | 알고리즘/문제풀이/간단 구현 | 메모리 예측·연결형 처리, 트리 변환/이진화 트릭 |

---

## 13. 테스트/디버깅 체크리스트

1. **루트 식별**: parent 배열 or indegree로 검증.
2. **메모리 소유권**: new/delete 누수 방지—**스마트 포인터**를 우선.
3. **깊이 한계**: 매우 깊은 트리는 **반복 DFS**로 전환.
4. **출력/순회 검증**: 전위/후위/레벨 순서 예상과 일치 확인.
5. **변환 왕복**: vector → CS → vector 이후 구조 동일성(값/자식수/순서) 확인.
6. **대형 입력**: 프로파일링—할당 수, 캐시 미스 비율, 스택 사용량.

간단 퍼징(왕복 + 순회 크기 검증):

```cpp
```cpp
// 아이디어: 랜덤 트리 생성 → 변환 왕복 → 노드 수/리프 수/높이 일치성 검사
```
```

---

## 14. 수학 스냅샷

### 14.1 트리 기본 성질
노드 수 \(n\), 간선 수 \(m\)에 대하여 트리는 **연결+비순환**이므로
$$
m = n - 1.
$$

### 14.2 Euler Tour 서브트리 포함 관계
루트 \(r\)에서의 Euler tour 타임스탬프 \(in[v], out[v]\)에 대해
$$
v \in \text{subtree}(u) \iff in[u] \le in[v] \le out[u].
$$

---

## 15. 마무리 요약

- **일반 트리**는 자식 수 무제한의 계층을 표현하는 **핵심 구조**.
- 구현은 **vector 기반**(직관/알고리즘 친화)과 **child–sibling**(이진화/메모리 예측) 두 축.
- **순회 패턴**(전위/후위/레벨), **유틸(높이/리프/서브트리)**, **지름**/**Euler tour** 등 필수 알고리즘을 갖추면 대부분의 문제를 다룰 수 있다.
- **스마트 포인터**로 소유권을 명확히 하고, **변환/빌더**로 입력을 유연하게 처리하라.
- 실제 서비스/엔진에서는 **메모리 풀**/**캐시 친화 레이아웃**과 **반복 DFS**를 고려하면 성능/안정성이 오른다.
