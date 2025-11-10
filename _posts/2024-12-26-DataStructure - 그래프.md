---
layout: post
title: Data Structure - 그래프
date: 2024-12-26 19:20:23 +0900
category: Data Structure
---
# 그래프(Graph) 자료구조

## 1. 그래프란?

그래프 \(G=(V,E)\)는 **정점(Vertex)** 집합 \(V\)와 **간선(Edge)** 집합 \(E\)로 구성된 구조다.  
정점은 도시·웹페이지·사용자 등 **개체**를, 간선은 도로·하이퍼링크·팔로우 등 **관계**를 나타낸다.

- **방향 그래프**: 간선이 순서쌍 \((u,v)\)로, 흐름이 **일방향**.
- **무방향 그래프**: 간선이 집합 \(\{u,v\}\)로, 연결만을 의미.
- **가중치 그래프**: 각 간선 \(e\)에 비용/거리 \(w(e)\)가 부여.
- **비가중 그래프**: 간선 존재만 중요(모든 비용=1로 취급 가능).
- **DAG**(Directed Acyclic Graph): 방향 + 사이클 없음.
- **연결성**: 무방향에서 한 컴포넌트로 모두 이어지면 연결 그래프.

### 기본 표기와 성질

- 정점 수 \(n=|V|\), 간선 수 \(m=|E|\).
- 무방향에서 **차수(degree)** \(deg(v)\)는 \(v\)에 접한 간선 수.  
  \[
  \sum_{v\in V} deg(v) = 2m
  \]
- 방향 그래프에서 **진입차수/진출차수**:  
  \[
  \sum_{v\in V} indeg(v) = \sum_{v\in V} outdeg(v) = m
  \]

---

## 2. 그래프의 분류 (요약 표)

| 분류 | 종류 | 핵심 포인트 |
|---|---|---|
| 방향성 | Directed / Undirected | 흐름 여부 |
| 가중치 | Weighted / Unweighted | 거리/비용 모델 |
| 사이클 | Cyclic / DAG | 위상정렬/DP 가능성 |
| 연결성 | Connected / Disconnected | 컴포넌트 개수 |
| 밀도 | 희소 \(m=O(n)\) / 밀집 \(m=\Theta(n^2)\) | 표현·알고리즘 선택에 영향 |

---

## 3. 그래프 표현

### 3.1 인접 행렬 (Adjacency Matrix)

- \(n\times n\) 배열 \(A\), \(A[u][v]\)가 간선 존재/가중치를 담음.
- 공간 \(O(n^2)\) — **밀집 그래프**에서 단순·빠름.
- 임의 간선 조회 O(1), 한 정점의 인접 탐색 O(n).

```cpp
const int N = 5;
const long long INF = (1LL<<60);
long long adj[N][N];
```

### 3.2 인접 리스트 (Adjacency List)

- 각 정점에 연결된 간선 목록.
- 공간 \(O(n+m)\) — **희소 그래프**에 적합.
- 한 정점의 인접 탐색 O(degree).

```cpp
#include <vector>
struct Edge { int to; long long w; };
std::vector<std::vector<Edge>> adj; // adj[u] = { (v,w), ... }
```

### 3.3 간선 리스트 (Edge List)

- 간선만 모아둔 배열 \((u,v,w)\).
- **크루스칼**(정렬→유니온파인드) 등에서 유용.

---

## 4. C++ 기본 그래프 클래스 (가중/무방향 옵션)

```cpp
#include <bits/stdc++.h>
using namespace std;

struct Edge { int to; long long w; };

class Graph {
public:
    int n; bool directed;
    vector<vector<Edge>> adj;
    Graph(int n, bool directed=false): n(n), directed(directed), adj(n) {}

    void addEdge(int u, int v, long long w=1) {
        adj[u].push_back({v,w});
        if (!directed) adj[v].push_back({u,w});
    }
};
```

입력(1-indexed)을 0-index로 매핑하는 습관:

```cpp
int n,m; cin>>n>>m;
Graph g(n, /*directed=*/true);
for(int i=0;i<m;++i){
    int u,v; long long w; cin>>u>>v>>w;
    --u; --v;
    g.addEdge(u,v,w);
}
```

---

## 5. 그래프 탐색: DFS / BFS

### 5.1 DFS (재귀)

```cpp
void dfs_rec(int u, const Graph& g, vector<char>& vis) {
    vis[u]=1;
    // 방문 처리 (예: 출력)
    for (auto &e: g.adj[u]) if (!vis[e.to]) dfs_rec(e.to, g, vis);
}
```

### 5.2 DFS (반복, 스택)

```cpp
void dfs_iter(int s, const Graph& g) {
    vector<char> vis(g.n,0);
    stack<int> st; st.push(s);
    while(!st.empty()){
        int u=st.top(); st.pop();
        if(vis[u]) continue; vis[u]=1;
        for (auto &e: g.adj[u]) if(!vis[e.to]) st.push(e.to);
    }
}
```

### 5.3 BFS (최단 간선 수 = 거리)

- 비가중 그래프에서 **최단 간선 수**는 BFS로 계산.

```cpp
void bfs(int s, const Graph& g, vector<int>& dist, vector<int>& parent){
    dist.assign(g.n, INT_MAX); parent.assign(g.n, -1);
    queue<int> q; dist[s]=0; q.push(s);
    while(!q.empty()){
        int u=q.front(); q.pop();
        for(auto &e: g.adj[u]){
            int v=e.to;
            if(dist[v]==INT_MAX){
                dist[v]=dist[u]+1; parent[v]=u; q.push(v);
            }
        }
    }
}
```

경로 복원:

```cpp
vector<int> buildPath(int s, int t, const vector<int>& parent){
    vector<int> path;
    for(int v=t; v!=-1; v=parent[v]) path.push_back(v);
    reverse(path.begin(), path.end());
    if(path.empty() || path.front()!=s) path.clear();
    return path;
}
```

---

## 6. 연결 요소(컴포넌트)

무방향 그래프에서 **연결 요소 수**와 각 요소를 찾기:

```cpp
int components(const Graph& g, vector<int>& compId){
    compId.assign(g.n, -1);
    int cid=0;
    for(int i=0;i<g.n;++i) if(compId[i]==-1){
        queue<int> q; q.push(i); compId[i]=cid;
        while(!q.empty()){
            int u=q.front(); q.pop();
            for(auto &e: g.adj[u]) if(compId[e.to]==-1){
                compId[e.to]=cid; q.push(e.to);
            }
        }
        ++cid;
    }
    return cid;
}
```

---

## 7. 사이클 탐지

### 7.1 무방향 사이클 (DFS, 부모 추적)

```cpp
bool hasCycleUndirected(const Graph& g){
    vector<char> vis(g.n,0);
    function<bool(int,int)> dfs=[&](int u,int p){
        vis[u]=1;
        for(auto &e: g.adj[u]){
            int v=e.to;
            if(v==p) continue;
            if(vis[v]) return true;
            if(dfs(v,u)) return true;
        }
        return false;
    };
    for(int i=0;i<g.n;++i) if(!vis[i] && dfs(i,-1)) return true;
    return false;
}
```

### 7.2 방향 사이클 (색 배열: 0/1/2 = 미방문/스택/완료)

```cpp
bool hasCycleDirected(const Graph& g){
    vector<int> color(g.n,0);
    function<bool(int)> dfs=[&](int u){
        color[u]=1;
        for(auto &e: g.adj[u]){
            int v=e.to;
            if(color[v]==1) return true;        // back edge
            if(color[v]==0 && dfs(v)) return true;
        }
        color[u]=2; return false;
    };
    for(int i=0;i<g.n;++i) if(color[i]==0 && dfs(i)) return true;
    return false;
}
```

---

## 8. 위상 정렬 (DAG 전용)

### 8.1 Kahn(진입차수 0 큐)

```cpp
vector<int> topoKahn(const Graph& g){
    vector<int> indeg(g.n,0);
    for(int u=0;u<g.n;++u) for(auto &e: g.adj[u]) ++indeg[e.to];
    queue<int> q;
    for(int i=0;i<g.n;++i) if(!indeg[i]) q.push(i);
    vector<int> order; order.reserve(g.n);
    while(!q.empty()){
        int u=q.front(); q.pop(); order.push_back(u);
        for(auto &e: g.adj[u]) if(--indeg[e.to]==0) q.push(e.to);
    }
    if((int)order.size()!=g.n) order.clear(); // cycle 존재
    return order;
}
```

### 8.2 DFS 후역 순서 역순

```cpp
vector<int> topoDFS(const Graph& g){
    vector<char> vis(g.n,0);
    vector<int> stk;
    function<void(int)> dfs=[&](int u){
        vis[u]=1;
        for(auto &e: g.adj[u]) if(!vis[e.to]) dfs(e.to);
        stk.push_back(u);
    };
    for(int i=0;i<g.n;++i) if(!vis[i]) dfs(i);
    reverse(stk.begin(), stk.end());
    return stk;
}
```

---

## 9. 이분 그래프(이분성 검사)

- 무방향 그래프가 **2-컬러링** 가능 ⇔ **홀수 사이클 없음**.

```cpp
bool isBipartite(const Graph& g, vector<int>& color){
    color.assign(g.n, -1);
    for(int s=0;s<g.n;++s) if(color[s]==-1){
        queue<int> q; q.push(s); color[s]=0;
        while(!q.empty()){
            int u=q.front(); q.pop();
            for(auto &e: g.adj[u]){
                int v=e.to;
                if(color[v]==-1){ color[v]=color[u]^1; q.push(v); }
                else if(color[v]==color[u]) return false;
            }
        }
    }
    return true;
}
```

---

## 10. 최단 경로

### 10.1 비가중 그래프: BFS

- 간선 비용이 모두 1인 경우 **BFS**로 최단 간선 수 구함(§5.3 참고).

### 10.2 Dijkstra (비음수 가중치)

```cpp
struct DijRes { vector<long long> dist; vector<int> par; };
const long long INF = (1LL<<60);

DijRes dijkstra(const Graph& g, int s){
    vector<long long> dist(g.n, INF); vector<int> par(g.n, -1);
    using P=pair<long long,int>;
    priority_queue<P, vector<P>, greater<P>> pq;
    dist[s]=0; pq.push({0,s});
    vector<char> done(g.n,0);
    while(!pq.empty()){
        auto [d,u]=pq.top(); pq.pop();
        if(done[u]) continue; done[u]=1;
        if(d!=dist[u]) continue;
        for(auto &e: g.adj[u]){
            int v=e.to; long long w=e.w;
            if(dist[u]+w<dist[v]){
                dist[v]=dist[u]+w; par[v]=u; pq.push({dist[v],v});
            }
        }
    }
    return {dist, par};
}
```

### 10.3 Bellman–Ford (음수 가중치/사이클 감지)

```cpp
struct BFRes { vector<long long> dist; vector<int> par; bool negCycle; };

BFRes bellmanFord(const Graph& g, int s){
    vector<long long> dist(g.n, INF); vector<int> par(g.n,-1);
    dist[s]=0;
    for(int it=0; it<g.n-1; ++it){
        bool upd=false;
        for(int u=0;u<g.n;++u){
            if(dist[u]==INF) continue;
            for(auto &e: g.adj[u]){
                if(dist[u]+e.w < dist[e.to]){
                    dist[e.to]=dist[u]+e.w; par[e.to]=u; upd=true;
                }
            }
        }
        if(!upd) break;
    }
    // n번째 완화 시도 → 음수 사이클
    for(int u=0;u<g.n;++u){
        if(dist[u]==INF) continue;
        for(auto &e: g.adj[u]){
            if(dist[u]+e.w < dist[e.to]){
                return {dist, par, true};
            }
        }
    }
    return {dist, par, false};
}
```

---

## 11. 최소 신장 트리 (MST, 무방향)

### 11.1 Kruskal (정렬 + 유니온파인드)

```cpp
struct DSU{
    vector<int> p, r;
    DSU(int n): p(n), r(n,0){ iota(p.begin(),p.end(),0); }
    int find(int x){ return p[x]==x?x:p[x]=find(p[x]); }
    bool unite(int a,int b){
        a=find(a); b=find(b); if(a==b) return false;
        if(r[a]<r[b]) swap(a,b);
        p[b]=a; if(r[a]==r[b]) r[a]++; return true;
    }
};

long long kruskal(int n, vector<tuple<int,int,long long>>& edges){
    sort(edges.begin(), edges.end(), [](auto &a, auto &b){
        return get<2>(a)<get<2>(b);
    });
    DSU dsu(n); long long cost=0; int used=0;
    for(auto &[u,v,w]: edges){
        if(dsu.unite(u,v)){ cost+=w; used++; if(used==n-1) break; }
    }
    return (used==n-1)? cost : (long long)-1; // 연결 안 되면 실패
}
```

### 11.2 Prim (하나의 컴포넌트에서 자라나기)

```cpp
long long prim(const Graph& g){
    vector<char> vis(g.n,0);
    using P=pair<long long,int>;
    priority_queue<P,vector<P>,greater<P>> pq;
    long long cost=0; int cnt=0;
    pq.push({0,0});
    while(!pq.empty()){
        auto [w,u]=pq.top(); pq.pop();
        if(vis[u]) continue; vis[u]=1; cost+=w; cnt++;
        for(auto &e: g.adj[u]) if(!vis[e.to]) pq.push({e.w, e.to});
    }
    return (cnt==g.n)? cost : (long long)-1;
}
```

---

## 12. 강연결요소(SCC, 방향 그래프)

### 12.1 Kosaraju (역방향 그래프 + 2-pass)

```cpp
void dfs1(int u, const Graph& g, vector<char>& vis, vector<int>& order){
    vis[u]=1;
    for(auto &e: g.adj[u]) if(!vis[e.to]) dfs1(e.to,g,vis,order);
    order.push_back(u);
}
void dfs2(int u, const Graph& rg, vector<int>& comp, int cid){
    comp[u]=cid;
    for(auto &e: rg.adj[u]) if(comp[e.to]==-1) dfs2(e.to, rg, comp, cid);
}

int scc_kosaraju(const Graph& g, vector<int>& comp){
    vector<char> vis(g.n,0); vector<int> order;
    for(int i=0;i<g.n;++i) if(!vis[i]) dfs1(i,g,vis,order);
    // 역그래프
    Graph rg(g.n, true);
    for(int u=0;u<g.n;++u) for(auto &e: g.adj[u]) rg.addEdge(e.to,u,e.w);
    comp.assign(g.n,-1);
    int cid=0;
    for(int i=g.n-1;i>=0;--i){
        int v=order[i];
        if(comp[v]==-1){ dfs2(v,rg,comp,cid++); }
    }
    return cid; // SCC 개수
}
```

---

## 13. 브리지(단절간선) & 단절점 (무방향, Tarjan)

```cpp
void bridgesAP(const Graph& g, vector<pair<int,int>>& bridges, vector<int>& art){
    vector<int> tin(g.n,-1), low(g.n,-1), parent(g.n,-1);
    vector<int> isArt(g.n,0);
    int timer=0;

    function<void(int)> dfs=[&](int u){
        tin[u]=low[u]=timer++;
        int child=0;
        for(auto &e: g.adj[u]){
            int v=e.to;
            if(tin[v]==-1){
                parent[v]=u; ++child; dfs(v);
                low[u]=min(low[u], low[v]);
                if(low[v] > tin[u]) bridges.push_back({u,v});
                if(parent[u]==-1 && child>1) isArt[u]=1;
                if(parent[u]!=-1 && low[v]>=tin[u]) isArt[u]=1;
            }else if(v!=parent[u]){
                low[u]=min(low[u], tin[v]);
            }
        }
    };

    for(int i=0;i<g.n;++i) if(tin[i]==-1) dfs(i);
    art.clear();
    for(int i=0;i<g.n;++i) if(isArt[i]) art.push_back(i);
}
```

---

## 14. 모든쌍 최단경로 (Floyd–Warshall)

점화식:
\[
D^{(k)}[i][j] = \min\big(D^{(k-1)}[i][j],\; D^{(k-1)}[i][k]+D^{(k-1)}[k][j]\big)
\]

```cpp
vector<vector<long long>> floydWarshall(const Graph& g){
    int n=g.n;
    vector<vector<long long>> d(n, vector<long long>(n, INF));
    for(int i=0;i<n;++i){ d[i][i]=0;
        for(auto &e: g.adj[i]) d[i][e.to]=min(d[i][e.to], e.w);
    }
    for(int k=0;k<n;++k)
        for(int i=0;i<n;++i) if(d[i][k]<INF)
            for(int j=0;j<n;++j) if(d[k][j]<INF)
                d[i][j]=min(d[i][j], d[i][k]+d[k][j]);
    return d;
}
```

---

## 15. 실전 예제: 방향+가중치 그래프

정점: A,B,C,D,E (0..4)  
간선: A→B(4), A→C(2), B→C(5), B→D(10), C→E(3), E→D(4)

```cpp
int main(){
    ios::sync_with_stdio(false);
    cin.tie(nullptr);

    Graph g(5, true);
    g.addEdge(0,1,4); g.addEdge(0,2,2);
    g.addEdge(1,2,5); g.addEdge(1,3,10);
    g.addEdge(2,4,3); g.addEdge(4,3,4);

    // Dijkstra: A(0) -> D(3)
    auto dj = dijkstra(g, 0);
    cout << "Dijkstra dist(A->D) = " << dj.dist[3] << "\n";
    auto path = [&]{
        vector<int> p; for(int v=3; v!=-1; v=dj.par[v]) p.push_back(v);
        reverse(p.begin(), p.end()); return p;
    }();
    cout << "Path: "; for(int v: path) cout << v << " "; cout << "\n";

    // Bellman–Ford
    auto bf = bellmanFord(g, 0);
    cout << "BF dist(A->D) = " << bf.dist[3] << (bf.negCycle?" (negCycle)":"") << "\n";

    // APSP
    auto all = floydWarshall(g);
    cout << "FW dist(A->D) = " << all[0][3] << "\n";
    return 0;
}
```

예상: 최단 비용 = 9 (A→C→E→D = 2+3+4)

---

## 16. 시간/공간 복잡도 요약

| 주제 | 알고리즘 | 시간 | 공간 | 비고 |
|---|---|---|---|---|
| 탐색 | DFS/BFS | \(O(n+m)\) | \(O(n+m)\) | 컴포넌트, 거리, 트리 |
| 사이클(무방향) | DFS | \(O(n+m)\) | \(O(n)\) | 부모 추적 |
| 사이클(방향) | DFS 색 | \(O(n+m)\) | \(O(n)\) | back edge |
| 위상정렬 | Kahn/DFS | \(O(n+m)\) | \(O(n)\) | DAG 필요 |
| 이분성 | BFS 색 | \(O(n+m)\) | \(O(n)\) | 홀수 사이클 검출 |
| 단일최단(비음수) | Dijkstra | \(O((n+m)\log n)\) | \(O(n+m)\) | PQ |
| 단일최단(음수 허용) | Bellman–Ford | \(O(nm)\) | \(O(n)\) | 음수사이클 탐지 |
| MST | Kruskal | \(O(m\log m)\) | \(O(n)\) | DSU |
| MST | Prim | \(O(m\log n)\) | \(O(n)\) | PQ |
| 모든쌍 | Floyd–Warshall | \(O(n^3)\) | \(O(n^2)\) | 밀집/작은 n |
| 브리지/단절점 | Tarjan | \(O(n+m)\) | \(O(n)\) | low-link |
| SCC | Kosaraju/Tarjan | \(O(n+m)\) | \(O(n)\) | 방향 그래프 |

---

## 17. 수학적 관점 몇 가지

- **무방향 그래프 합차수 정리**  
  \[
  \sum_{v\in V}deg(v)=2m
  \]
- **BFS의 최단성(무가중)**: 레벨 \(k\)에서 방문되는 모든 정점은 시작점으로부터 간선 \(k\)개로 도달 가능하며, 더 짧은 경로가 있으면 이전 레벨에서 이미 방문되었어야 하므로 모순.

---

## 18. 실전 구현 팁

- **인덱스**: 입력이 1..n이면 반드시 `--u; --v;`로 0-index 통일.
- **가중치/거리형**: `long long` + `INF=2^60` 등 **오버플로 주의**.
- **자기루프/다중간선**: 문제 조건을 확인하고 허용 시 처리(예: `min`으로 병합).
- **재귀 한도**: 깊은 DFS는 스택오버플로 위험 → **반복형** 또는 `-Wl,--stack` 조정(환경에 따라).
- **그래프 밀도**: 희소면 인접 리스트, 밀집이면 행렬(FW) 고려.
- **음수 간선**: Dijkstra 금지. BF 또는 Johnson(FW 대체) 사용.
- **테스트**: 작은 그래프에서 손계산 경로와 결과 비교(경로 복원 필수).

---

## 19. 종합 데모(메뉴 기반)

```cpp
#include <bits/stdc++.h>
using namespace std;

struct Edge{ int to; long long w; };
struct Graph{
    int n; bool directed;
    vector<vector<Edge>> adj;
    Graph(int n,bool directed=false):n(n),directed(directed),adj(n){}
    void addEdge(int u,int v,long long w=1){
        adj[u].push_back({v,w});
        if(!directed) adj[v].push_back({u,w});
    }
};
const long long INF=(1LL<<60);

vector<int> buildPath(int s,int t,const vector<int>& par){
    vector<int> p; for(int v=t; v!=-1; v=par[v]) p.push_back(v);
    reverse(p.begin(), p.end()); if(p.empty()||p.front()!=s) p.clear(); return p;
}

pair<vector<long long>,vector<int>> dijkstra(const Graph& g,int s){
    vector<long long> dist(g.n,INF); vector<int> par(g.n,-1);
    using P=pair<long long,int>;
    priority_queue<P,vector<P>,greater<P>> pq;
    dist[s]=0; pq.push({0,s});
    while(!pq.empty()){
        auto [d,u]=pq.top(); pq.pop();
        if(d!=dist[u]) continue;
        for(auto &e: g.adj[u]) if(dist[u]+e.w<dist[e.to]){
            dist[e.to]=dist[u]+e.w; par[e.to]=u; pq.push({dist[e.to],e.to});
        }
    }
    return {dist,par};
}

int main(){
    ios::sync_with_stdio(false);
    cin.tie(nullptr);

    // 예제: A..E
    Graph g(5,true);
    g.addEdge(0,1,4); g.addEdge(0,2,2);
    g.addEdge(1,2,5); g.addEdge(1,3,10);
    g.addEdge(2,4,3); g.addEdge(4,3,4);

    auto [dist,par]=dijkstra(g,0);
    cout<<"A->D shortest = "<<dist[3]<<"\n";
    auto path=buildPath(0,3,par);
    cout<<"Path: "; for(int v: path) cout<<v<<" "; cout<<"\n";
    return 0;
}
```

---

## 20. 마무리 요약

| 키워드 | 요약 |
|---|---|
| 표현 | 희소: 인접 리스트 / 밀집·APSP: 행렬(FW) |
| 탐색 | DFS/BFS로 컴포넌트·거리·트리·사이클 기초 처리 |
| 위상 | DAG에서 위상정렬 → 선형 시간 DP 가능 |
| 최단경로 | 무가중=BFS / 비음수=Dijkstra / 음수=BF |
| MST | 크루스칼(정렬+DSU), 프림(PQ) |
| 구조 | SCC, 브리지/단절점은 Tarjan/Kosaraju로 \(O(n+m)\) |
| 실무 팁 | 오버플로·인덱스·다중간선·재귀한도·INF 관리 |