---
layout: post
title: Data Structure - 가중치 그래프
date: 2024-12-26 20:20:23 +0900
category: Data Structure
---
# 가중치 그래프 완전 정리

## 문제 정의와 표기

- 정점 집합 \(V\), 간선 집합 \(E\)로 이루어진 방향 가중치 그래프 \(G=(V,E,w)\).
- 간선 \( (u\to v) \in E \)의 가중치 \( w(u,v) \in \mathbb{R} \) (일반적으로 정수/실수).
- **경로 길이**: 경로상의 가중치 합.
- **최단 경로 문제**: 주어진 출발 \(s\)에서의 \( \text{dist}[v] \) 최솟값 계산.

**핵심 연산 — 완화(Relaxation)**
간선 \((u\to v)\)에 대해
\[
\text{if } \text{dist}[u] + w(u,v) < \text{dist}[v] \text{ then }
\begin{cases}
\text{dist}[v]\gets \text{dist}[u]+w(u,v)\\
\text{parent}[v]\gets u
\end{cases}
\]

---

## 예제 그래프(방향 + 가중치)

정점: A, B, C, D, E
간선:

| 출발 | 도착 | 가중치 |
|------|------|--------|
| A    | B    | 4      |
| A    | C    | 2      |
| B    | C    | 5      |
| B    | D    | 10     |
| C    | E    | 3      |
| E    | D    | 4      |

인덱스 매핑: A=0, B=1, C=2, D=3, E=4.

---

## 그래프 표현 (C++ 인접 리스트)

```cpp
#include <bits/stdc++.h>

using namespace std;

struct Edge { int to; long long w; };

struct Graph {
    int n;
    vector<vector<Edge>> adj;
    explicit Graph(int n): n(n), adj(n) {}
    void addEdge(int u, int v, long long w) {
        adj[u].push_back({v, w});
    }
};
```

예제 그래프 구성과 출력:

```cpp
Graph makeExample() {
    Graph g(5); // A=0, B=1, C=2, D=3, E=4
    g.addEdge(0,1,4);   // A->B
    g.addEdge(0,2,2);   // A->C
    g.addEdge(1,2,5);   // B->C
    g.addEdge(1,3,10);  // B->D
    g.addEdge(2,4,3);   // C->E
    g.addEdge(4,3,4);   // E->D
    return g;
}

void printAdj(const Graph& g){
    for(int u=0; u<g.n; ++u){
        cout << "Node " << u << " -> ";
        for(auto &e: g.adj[u]) cout << "(" << e.to << "," << e.w << ") ";
        cout << "\n";
    }
}
```

---

## 경로 복원(Parent 배열)

최단 경로를 구한 뒤, `parent`를 역추적하여 경로를 복원한다.

```cpp
vector<int> buildPath(int s, int t, const vector<int>& parent){
    vector<int> path;
    if (t<0 || t>=(int)parent.size()) return path;
    for(int v=t; v!=-1; v=parent[v]) path.push_back(v);
    reverse(path.begin(), path.end());
    if(path.empty() || path.front()!=s) path.clear();
    return path;
}
```

---

## Dijkstra — 음수 간선이 없는 경우의 단일 시작점 최단 경로

### 원리와 조건

- **가중치가 모두 비음수**일 때 동작 보장.
- 우선순위 큐(최소 힙)로 미방문 정점 중 현재 가장 작은 거리 정점을 선택.

복잡도:
- 인접 리스트 + 이진 힙: \(O((n+m)\log n)\).

### 구현 (경로 복원 포함)

```cpp
struct DijkstraResult {
    vector<long long> dist;
    vector<int> parent;
};

const long long INF = (1LL<<60);

DijkstraResult dijkstra(const Graph& g, int s) {
    vector<long long> dist(g.n, INF);
    vector<int> parent(g.n, -1);
    vector<char> visited(g.n, 0);

    using P = pair<long long,int>; // (dist, node)
    priority_queue<P, vector<P>, greater<P>> pq;

    dist[s] = 0;
    pq.push({0, s});

    while(!pq.empty()){
        auto [d,u] = pq.top(); pq.pop();
        if (visited[u]) continue;
        visited[u] = 1;
        if (d != dist[u]) continue; // stale entry

        for(auto &e: g.adj[u]){
            int v = e.to; long long w = e.w;
            if(dist[u]!=INF && dist[u]+w < dist[v]){
                dist[v] = dist[u] + w;
                parent[v] = u;
                pq.push({dist[v], v});
            }
        }
    }
    return {dist, parent};
}
```

### 예제 검증 (A→D)

- 후보 경로
  - A→C→E→D = 2 + 3 + 4 = 9
  - A→B→D = 4 + 10 = 14
따라서 최단 경로는 비용 9.

예제 실행:

```cpp
int main(){
    ios::sync_with_stdio(false);
    cin.tie(nullptr);

    auto g = makeExample();
    // printAdj(g);

    auto res = dijkstra(g, /*A*/0);
    int s = 0, t = 3; // A->D
    cout << "Dijkstra dist[A->D] = " << res.dist[t] << "\n"; // 9
    auto path = buildPath(s, t, res.parent);
    cout << "Path: ";
    for(int v: path) cout << v << " ";
    cout << "\n"; // 0 2 4 3  (A C E D)

    return 0;
}
```

---

## Bellman–Ford — 음수 간선/사이클 처리

### 원리와 조건

- 간선 완화를 **정확히 \(n-1\)번** 수행하면, 음수 사이클이 없으면 모든 최단 경로가 수렴.
- \(n\)번째 완화에서 갱신이 일어나는 정점은 **음수 사이클의 영향**을 받는다.

복잡도: \(O(nm)\).

### 구현 (음수 사이클 감지 + 영향 전파)

```cpp
struct BFResult {
    vector<long long> dist;
    vector<int> parent;
    vector<char> negInf; // 음수 사이클 영향으로 -∞ 취급되는 노드 표시
    bool hasNegCycle;
};

BFResult bellmanFord(const Graph& g, int s){
    vector<long long> dist(g.n, INF);
    vector<int> parent(g.n, -1);
    dist[s] = 0;

    // n-1 라운드 완화
    for(int iter=0; iter<g.n-1; ++iter){
        bool any=false;
        for(int u=0; u<g.n; ++u){
            if(dist[u]==INF) continue;
            for(auto &e: g.adj[u]){
                int v=e.to; long long w=e.w;
                if(dist[u]+w < dist[v]){
                    dist[v] = dist[u]+w;
                    parent[v] = u;
                    any=true;
                }
            }
        }
        if(!any) break;
    }

    // n번째 라운드: 음수사이클 영향 탐지
    vector<char> negInf(g.n, 0);
    bool hasNegCycle = false;
    for(int u=0; u<g.n; ++u){
        if(dist[u]==INF) continue;
        for(auto &e: g.adj[u]){
            int v=e.to; long long w=e.w;
            if(dist[u]+w < dist[v]){
                hasNegCycle=true;
                // v와 v로부터 도달 가능한 정점들은 -∞로 전파
                // BFS/DFS로 전파
            }
        }
    }
    if(hasNegCycle){
        queue<int> q;
        vector<char> inq(g.n, 0);
        // 초기 큐: n번째에서도 완화 가능한 간선의 도착점들
        for(int u=0; u<g.n; ++u){
            if(dist[u]==INF) continue;
            for(auto &e: g.adj[u]){
                int v=e.to; long long w=e.w;
                if(dist[u]+w < dist[v]){
                    if(!inq[v]){ q.push(v); inq[v]=1; }
                }
            }
        }
        while(!q.empty()){
            int u=q.front(); q.pop();
            negInf[u]=1;
            for(auto &e: g.adj[u]){
                int v=e.to;
                if(!negInf[v]){
                    negInf[v]=1; q.push(v);
                }
            }
        }
    }
    return {dist, parent, negInf, hasNegCycle};
}
```

사용 예:
```cpp
// 예제 그래프는 음수 간선이 없으므로 hasNegCycle=false, 결과는 Dijkstra와 동일
auto bf = bellmanFord(g, 0);
if(bf.hasNegCycle){
    cout << "Negative cycle exists and affects some nodes.\n";
}else{
    cout << "BF dist[A->D] = " << bf.dist[3] << "\n"; // 9
}
```

---

## Floyd–Warshall — 모든 쌍 최단 경로(APSP)

### 동적 계획법 점화식

정점 집합을 \(\{1,\dots,n\}\)이라 하면,
\[
\text{dist}^{(k)}[i][j] = \min\Big(\text{dist}^{(k-1)}[i][j],\;
\text{dist}^{(k-1)}[i][k] + \text{dist}^{(k-1)}[k][j]\Big)
\]
초기값: \(\text{dist}^{(0)}[i][j] = w(i,j)\), 자기 자신 \(0\), 간선 없으면 \(\infty\).

복잡도: \(O(n^3)\), 공간 \(O(n^2)\).

### 구현(경로 복원 포함)

```cpp
struct FW {
    vector<vector<long long>> dist;
    vector<vector<int>> next; // 경로 복원: i->j에서 다음 정점
};

FW floydWarshall(const Graph& g){
    int n=g.n;
    vector<vector<long long>> dist(n, vector<long long>(n, INF));
    vector<vector<int>> nxt(n, vector<int>(n, -1));

    for(int i=0;i<n;++i){
        dist[i][i]=0; nxt[i][i]=i;
        for(auto &e: g.adj[i]){
            dist[i][e.to]=min(dist[i][e.to], e.w);
            nxt[i][e.to]=e.to;
        }
    }

    for(int k=0;k<n;++k){
        for(int i=0;i<n;++i){
            if(dist[i][k]==INF) continue;
            for(int j=0;j<n;++j){
                if(dist[k][j]==INF) continue;
                long long cand = dist[i][k]+dist[k][j];
                if(cand < dist[i][j]){
                    dist[i][j]=cand;
                    nxt[i][j]=nxt[i][k];
                }
            }
        }
    }
    return {dist, nxt};
}

vector<int> fwPath(int s, int t, const FW& fw){
    if(fw.next[s][t]==-1) return {};
    vector<int> path = {s};
    while(s!=t){
        s = fw.next[s][t];
        if(s==-1){ path.clear(); return path; }
        path.push_back(s);
    }
    return path;
}
```

예제:
```cpp
auto fw = floydWarshall(g);
cout << "FW dist[A->D] = " << fw.dist[0][3] << "\n"; // 9
auto p = fwPath(0,3,fw);
for(int v: p) cout << v << " ";
cout << "\n"; // 0 2 4 3
```

---

## DAG에서의 최단/최장 경로 — 위상정렬 + DP

### DAG 최단 경로

- 사이클이 없으므로 **위상순서**로 한 번씩 완화하면 충분.
- 복잡도 \(O(n+m)\).

```cpp
vector<int> topoSort(const Graph& g){
    vector<int> indeg(g.n,0);
    for(int u=0;u<g.n;++u) for(auto &e: g.adj[u]) ++indeg[e.to];
    queue<int> q;
    for(int i=0;i<g.n;++i) if(!indeg[i]) q.push(i);
    vector<int> order;
    while(!q.empty()){
        int u=q.front(); q.pop(); order.push_back(u);
        for(auto &e: g.adj[u]) if(--indeg[e.to]==0) q.push(e.to);
    }
    return order; // DAG가 아니면 크기<n
}

pair<vector<long long>, vector<int>> dagShortest(const Graph& g, int s){
    auto order = topoSort(g);
    vector<long long> dist(g.n, INF);
    vector<int> parent(g.n, -1);
    dist[s]=0;
    for(int u: order){
        if(dist[u]==INF) continue;
        for(auto &e: g.adj[u]){
            if(dist[u]+e.w < dist[e.to]){
                dist[e.to]=dist[u]+e.w;
                parent[e.to]=u;
            }
        }
    }
    return {dist, parent};
}
```

### DAG 최장 경로

- 동일한 위상순서에서 **최대화**를 사용.
- 다만 **양의 사이클이 없다는 보장** 필요(=DAG).

```cpp
pair<vector<long long>, vector<int>> dagLongest(const Graph& g, int s){
    auto order = topoSort(g);
    vector<long long> dist(g.n, -INF);
    vector<int> parent(g.n, -1);
    dist[s]=0;
    for(int u: order){
        if(dist[u]==-INF) continue;
        for(auto &e: g.adj[u]){
            if(dist[u]+e.w > dist[e.to]){
                dist[e.to]=dist[u]+e.w;
                parent[e.to]=u;
            }
        }
    }
    return {dist, parent};
}
```

---

## 검증: 예제 그래프 A→D

- Dijkstra: 9, 경로 [A,C,E,D]
- Bellman–Ford: 9, 음수 사이클 없음
- Floyd–Warshall: 9, 경로 복원 동일
- DAG가 아닌 그래프(일반 방향 그래프)이므로 DAG DP는 바로 적용되지 않음(위상정렬 결과 크기가 n 미만이면 사이클 존재).

---

## 인접 행렬 vs 인접 리스트

| 표현 | 공간 | 간선 접근 | 최단경로 사용 |
|------|------|-----------|---------------|
| 인접 리스트 | \(O(n+m)\) | 정점 u의 간선만 순회 | Dijkstra/BF에 적합 |
| 인접 행렬   | \(O(n^2)\) | 임의 간선 \((u,v)\) O(1) | Floyd–Warshall에 적합 |

밀집 그래프( \(m\approx n^2\) )는 행렬 기반이 단순·빠를 수 있다.

---

## 자료형/오버플로/불능 상태

- 가중치 합이 `int` 범위를 넘을 수 있으므로 **`long long`** 사용 권장.
- **INF**는 충분히 큰 값으로 설정(예: \( \text{INF}=2^{60} \)).
- 도달 불가능: `dist[v]==INF`로 표현.
- Bellman–Ford의 `negInf[v]==1`이면 **음수 사이클 영향으로 최단의 개념이 무의미**.

---

## 알고리즘 선택 가이드

1) 음수 간선 없음 + 단일 시작점 → **Dijkstra**
2) 음수 간선 가능 + 음수 사이클 탐지 필요 → **Bellman–Ford**
3) 모든 쌍 → **Floyd–Warshall** (혹은 Johnson, 아래)
4) DAG → **위상정렬 + DP**
5) 대규모 희소 그래프의 모든 쌍 → **Johnson**:
   - 보조 정점 \(q\) 추가, \(q\to v\) 가중치 0
   - BF로 잠재치 \(h(v)\) 계산
   - 재가중치 \( w'(u,v)=w(u,v)+h(u)-h(v) \ge 0\)
   - 각 정점에서 Dijkstra 수행 → 원래 거리 복구
   - 복잡도 \(O(nm\log n)\) 수준, 모든 쌍 최단 경로에 적합(희소 그래프)

---

## 단위 테스트/데모 메인

```cpp
int main(){
    ios::sync_with_stdio(false);
    cin.tie(nullptr);

    auto g = makeExample();

    // 1) Dijkstra
    auto dj = dijkstra(g, 0);
    cout << "[Dijkstra] dist(A->D) = " << dj.dist[3] << "\n";
    auto p1 = buildPath(0,3,dj.parent);
    cout << "Path: "; for(int v: p1) cout << v << " "; cout << "\n";

    // 2) Bellman–Ford
    auto bf = bellmanFord(g, 0);
    if(bf.hasNegCycle){
        cout << "[BF] negative cycle affects some nodes\n";
    }else{
        cout << "[BF] dist(A->D) = " << bf.dist[3] << "\n";
        auto p2 = buildPath(0,3,bf.parent);
        cout << "Path: "; for(int v: p2) cout << v << " "; cout << "\n";
    }

    // 3) Floyd–Warshall
    auto fw = floydWarshall(g);
    cout << "[FW] dist(A->D) = " << fw.dist[0][3] << "\n";
    auto p3 = fwPath(0,3,fw);
    cout << "Path: "; for(int v: p3) cout << v << " "; cout << "\n";

    return 0;
}
```

출력 예(인덱스는 A=0, C=2, E=4, D=3):
```
[Dijkstra] dist(A->D) = 9
Path: 0 2 4 3
[BF] dist(A->D) = 9
Path: 0 2 4 3
[FW] dist(A->D) = 9
Path: 0 2 4 3
```

---

## 수학적 관점: Dijkstra의 올바름(스케치)

모든 가중치가 비음수일 때, Dijkstra는 우선순위 큐에서 꺼내는 정점 \(u\)에 대해 **해당 시점의 \(\text{dist}[u]\)가 최단거리**임을 보장한다.
만약 더 짧은 경로가 존재한다면 그 경로의 중간 정점 중 하나가 아직 미확정으로 큐에 남아 있어야 하며, 그 정점의 거리 상한이 \(\text{dist}[u]\)보다 작아야 한다. 비음수 조건 때문에 **먼저 꺼내졌어야 하는 모순**이 발생한다.

---

## 추가 고급 토픽

- **A\***: 휴리스틱 \(h(v)\)를 사용해서 탐색을 가속. \(h\)가 **적법(admissible)**이면 최단성 보장.
- **SPFA**: Bellman–Ford의 큐 최적화 변종. 최악 복잡도는 여전히 나쁠 수 있어 실전에서는 신중히 선택.
- **경로 개수/경로 복원 고급**: 최단 경로 그래프(DAG) 위에서 경로 수 DP 가능.
- **엣지 케이스**:
  - 매우 큰 가중치 → 오버플로 주의
  - 다중 간선/자기 루프 허용 여부 명확화
  - 도달 불능/음수 사이클 보고 형식 결정

---

## 복잡도 요약

| 알고리즘           | 조건                    | 시간복잡도                         | 공간 |
|-------------------|-------------------------|------------------------------------|------|
| Dijkstra           | \(w\ge 0\)              | \(O((n+m)\log n)\)                 | \(O(n+m)\) |
| Bellman–Ford       | 임의 가중치             | \(O(nm)\)                          | \(O(n)\) |
| Floyd–Warshall     | 모든 쌍                 | \(O(n^3)\)                         | \(O(n^2)\) |
| DAG shortest/long  | DAG                     | \(O(n+m)\)                         | \(O(n)\) |
| Johnson            | 모든 쌍, 희소 그래프     | \(O(nm\log n)\) (Dijkstra n회 + BF) | \(O(n+m)\) |

---

## 실전 체크리스트

- 입력 파싱 시 **정점 매핑**(문자↔인덱스) 일관성 유지
- **long long** 사용, `INF` 충분히 크게
- Dijkstra 사용 전 **음수 간선 존재 여부** 확인
- 경로 복원 실패 시(도달 불능/사이클 영향) 출력 정책 정의
- 모든 쌍 문제에서 희소 그래프면 **Johnson** 고려
- 단위 테스트: 작은 그래프에서 수작업 검증 경로와 결과 대조

---

## 결론

- 가중치 그래프의 기본은 **완화**와 **경로 복원**이다.
- 입력 특성(음수 유무, 규모, 밀도, 모든쌍/단일원천, DAG 여부)을 파악하면 **정답 알고리즘**이 자연스럽게 결정된다.
- 본문 예제 그래프에서는 Dijkstra/BF/FW 모두 동일하게 **A→D 최단 거리 9, 경로 A→C→E→D**를 확인했다.
