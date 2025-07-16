---
layout: post
title: Data Structure - 펜윅 트리
date: 2024-12-09 19:20:23 +0900
category: Data Structure
---
# 🔗 그래프(Graph) 자료구조 - 이론부터 구현까지

---

## 📌 1. 그래프란?

그래프는 **정점(Vertex, 노드)**과 **간선(Edge, 연결선)**으로 이루어진 자료구조입니다.

- **정점 (Vertex)**: 정보를 담는 단위 (예: 도시, 사람)
- **간선 (Edge)**: 정점 간의 관계 (예: 도로, 친구 관계)

---

## 🧩 2. 그래프의 분류

| 분류 기준 | 종류 | 설명 |
|-----------|------|------|
| 방향성 | **방향 그래프 (Directed)** | 간선에 방향 존재 (u → v) |
|        | **무방향 그래프 (Undirected)** | 간선에 방향 없음 (u — v) |
| 가중치 | **가중 그래프 (Weighted)** | 간선에 비용/거리 등 가중치 있음 |
|        | **비가중 그래프 (Unweighted)** | 가중치 없음 (간선 존재만 중요) |
| 사이클 | **사이클 그래프** | 순환 경로 존재 |
|        | **비순환 그래프 (DAG)** | 방향성 + 순환 없음 |
| 연결성 | **연결 그래프** | 모든 정점이 연결됨 |
|        | **비연결 그래프** | 일부 정점이 고립됨 |

---

## 🔧 3. 그래프 표현 방식

### ✅ 인접 행렬 (Adjacency Matrix)

- 2차원 배열 `adj[u][v]` = 간선 존재 여부
- 메모리: O(V²)
- 밀집 그래프에 유리

```cpp
const int V = 5;
int adj[V][V] = {0};
adj[0][1] = 1; // 0 → 1 간선
```

---

### ✅ 인접 리스트 (Adjacency List)

- 각 정점마다 연결된 정점 리스트를 저장
- 메모리: O(V + E)
- 희소 그래프에 유리

```cpp
vector<vector<int>> adj(V);
adj[0].push_back(1); // 0 → 1 간선
```

---

## 🛠️ 4. 그래프 구현 (C++, 인접 리스트 기반)

```cpp
#include <iostream>
#include <vector>
using namespace std;

class Graph {
    int V;
    vector<vector<int>> adj;

public:
    Graph(int V) : V(V), adj(V) {}

    void addEdge(int u, int v, bool directed = false) {
        adj[u].push_back(v);
        if (!directed)
            adj[v].push_back(u);
    }

    void print() {
        for (int u = 0; u < V; ++u) {
            cout << u << " → ";
            for (int v : adj[u])
                cout << v << " ";
            cout << endl;
        }
    }
};
```

---

## 🧭 5. 그래프 탐색 알고리즘

### 🔹 깊이 우선 탐색 (DFS)

```cpp
void dfs(int u, vector<vector<int>>& adj, vector<bool>& visited) {
    visited[u] = true;
    cout << u << " ";
    for (int v : adj[u])
        if (!visited[v])
            dfs(v, adj, visited);
}
```

---

### 🔹 너비 우선 탐색 (BFS)

```cpp
#include <queue>

void bfs(int start, vector<vector<int>>& adj) {
    vector<bool> visited(adj.size(), false);
    queue<int> q;
    visited[start] = true;
    q.push(start);

    while (!q.empty()) {
        int u = q.front(); q.pop();
        cout << u << " ";
        for (int v : adj[u]) {
            if (!visited[v]) {
                visited[v] = true;
                q.push(v);
            }
        }
    }
}
```

---

## 📊 6. 응용 분야

| 분야 | 활용 |
|------|------|
| 길찾기 | DFS, BFS, 다익스트라, A* |
| 소셜 네트워크 | 연결, 군집 탐지 |
| 게임 개발 | 맵 탐색, AI 추적 |
| 웹 | 페이지 링크, 크롤링 |
| 전자회로 | 방향 그래프 분석 |
| 데이터베이스 | 조인 경로, 인덱스 최적화 |

---

## 🔍 7. 시간 복잡도 비교

| 표현 방식 | 공간 복잡도 | 간선 확인 | 인접 노드 탐색 |
|-----------|--------------|------------|----------------|
| 인접 행렬 | O(V²) | O(1) | O(V) |
| 인접 리스트 | O(V+E) | O(V) | O(degree) |

---

## ✅ 마무리 요약

| 키워드 | 설명 |
|--------|------|
| 정점 & 간선 | 그래프의 기본 구성 |
| 인접 리스트 | 일반적으로 가장 효율적 |
| DFS/BFS | 모든 노드 탐색 |
| 표현 방식 선택 | 그래프의 밀도에 따라 달라짐 |
