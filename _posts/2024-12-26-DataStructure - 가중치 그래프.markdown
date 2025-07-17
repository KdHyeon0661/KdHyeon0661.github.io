---
layout: post
title: Data Structure - 펜윅 트리
date: 2024-12-09 20:20:23 +0900
category: Data Structure
---
# 💰 가중치 그래프 예시

---

## ✅ 예제 그래프 (방향 + 가중치)

- 정점: A, B, C, D, E
- 간선과 가중치:

| 출발 | 도착 | 가중치 |
|------|------|--------|
| A    | B    | 4      |
| A    | C    | 2      |
| B    | C    | 5      |
| B    | D    | 10     |
| C    | E    | 3      |
| E    | D    | 4      |

```
    A
   / \
 4/   \2
 /     \
B       C
|\       \
| \       \3
|  \10     E
|    \     |
 \     \   |
  \     \  |4
   ------> D
```

---

## 🧾 인접 리스트 표현 (C++)

```cpp
#include <iostream>
#include <vector>
using namespace std;

struct Edge {
    int to, weight;
};

int main() {
    const int V = 5;
    vector<vector<Edge>> graph(V);

    auto addEdge = [&](int from, int to, int weight) {
        graph[from].push_back({to, weight});
    };

    // 정점: A=0, B=1, C=2, D=3, E=4
    addEdge(0, 1, 4);  // A → B (4)
    addEdge(0, 2, 2);  // A → C (2)
    addEdge(1, 2, 5);  // B → C (5)
    addEdge(1, 3, 10); // B → D (10)
    addEdge(2, 4, 3);  // C → E (3)
    addEdge(4, 3, 4);  // E → D (4)

    // 출력
    for (int u = 0; u < V; ++u) {
        cout << "Node " << u << " → ";
        for (auto& e : graph[u]) {
            cout << "(" << e.to << ", " << e.weight << ") ";
        }
        cout << '\n';
    }
    return 0;
}
```

---

## 🌟 이 그래프에서 어떤 걸 할 수 있나요?

| 알고리즘 | 설명 |
|----------|------|
| Dijkstra | A → D까지의 최소 비용 경로 찾기 |
| Bellman-Ford | 음수 가중치가 있는 경우 사용 |
| Floyd-Warshall | 모든 쌍의 최단 경로 |
| Topological Sort + DP | DAG에서 최단/최장 거리 |

---

## 📌 예시 경로

- A → C → E → D  
  = 2 + 3 + 4 = **9**
- A → B → D  
  = 4 + 10 = **14**

✅ 따라서 A → D의 최단 경로는 `A → C → E → D` (비용: 9)

---

## ✅ 마무리

- 가중치 그래프는 **현실적인 거리/비용**을 반영할 수 있음