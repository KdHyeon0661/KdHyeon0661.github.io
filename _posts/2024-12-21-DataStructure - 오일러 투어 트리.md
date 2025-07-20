---
layout: post
title: Data Structure - 펜윅 트리
date: 2024-12-09 19:20:23 +0900
category: Data Structure
---
# 🌀 Euler Tour Tree (ETT) - 오일러 순회를 활용한 동적 트리 표현

## 📌 1. 개요

**Euler Tour Tree (오일러 투어 트리)** 는 트리를 **DFS 순회 결과로 표현**하여, **경로 정보와 서브트리 정보**를 빠르게 쿼리할 수 있도록 만든 구조입니다.

특히, **세그먼트 트리, 펜윅 트리(Fenwick Tree), Splay Tree**와 함께 사용되어  
- **부분 합/최소/최대 쿼리**
- **서브트리 갱신**
- **간선 추가/삭제 (연결/절단)**  
등을 **O(log n)** 에 처리할 수 있게 합니다.

---

## 🌲 2. 핵심 개념

- 트리를 **DFS 전위순회**하여 **오일러 경로(Euler Tour)**를 구함
- 각 노드를 **"들어올 때"**, **"나갈 때"** 2번 방문
- DFS 순서를 배열로 저장 → **세그먼트 트리 / 펜윅 트리** 등으로 관리 가능

### ✅ 예시

```
    1
   / \
  2   3
     / \
    4   5

Euler Tour: [1, 2, 2, 1, 3, 4, 4, 5, 5, 3, 1]
```

---

## 🧠 3. 적용 방식

| 항목 | 설명 |
|------|------|
| DFS 순회 결과 | 오일러 경로로 배열화 |
| 각 노드의 시작/종료 인덱스 | `in[u]`, `out[u]` 로 기록 |
| 트리 전체 → 배열 | → 세그먼트 트리 등으로 갱신/질의 가능 |
| 연결/절단 | 임의 트리 변경 시 오일러 경로 갱신 필요 |

---

## 🛠️ 4. 기본 구현 (C++)

### ✅ 변수 정의

```cpp
const int MAXN = 100005;
vector<int> tree[MAXN];
vector<int> euler;
int in[MAXN], out[MAXN];
int timer = 0;
```

---

### ✅ Euler Tour DFS

```cpp
void dfs(int u, int p) {
    in[u] = euler.size();
    euler.push_back(u);

    for (int v : tree[u]) {
        if (v == p) continue;
        dfs(v, u);
        euler.push_back(u);  // Back edge (나올 때 다시 기록)
    }

    out[u] = euler.size() - 1;
}
```

---

## ⚙️ 5. 세그먼트 트리 예제 연동

노드에 값을 할당하고, **노드 서브트리의 합**을 구한다고 가정합시다.

### ✅ 트리 노드 값 정의

```cpp
int value[MAXN];        // 노드 값
int seg[MAXN * 4];      // 세그먼트 트리

void build(int node, int l, int r) {
    if (l == r) {
        seg[node] = value[euler[l]];
        return;
    }
    int mid = (l + r) / 2;
    build(node * 2, l, mid);
    build(node * 2 + 1, mid + 1, r);
    seg[node] = seg[node * 2] + seg[node * 2 + 1];
}
```

---

### ✅ 서브트리 쿼리

```cpp
int query(int node, int l, int r, int ql, int qr) {
    if (qr < l || r < ql) return 0;
    if (ql <= l && r <= qr) return seg[node];
    int mid = (l + r) / 2;
    return query(node * 2, l, mid, ql, qr) + query(node * 2 + 1, mid + 1, r, ql, qr);
}

// 특정 노드 u의 서브트리 합
int getSubtreeSum(int u, int N) {
    return query(1, 0, N - 1, in[u], out[u]);
}
```

---

## 🧪 6. 예제 실행

```cpp
int main() {
    int N = 5;
    // 트리 구성
    tree[1] = {2, 3};
    tree[2] = {1};
    tree[3] = {1, 4, 5};
    tree[4] = {3};
    tree[5] = {3};

    // 노드 값 설정
    value[1] = 10;
    value[2] = 20;
    value[3] = 30;
    value[4] = 40;
    value[5] = 50;

    dfs(1, -1);
    int M = euler.size();
    build(1, 0, M - 1);

    cout << "Sum of subtree rooted at 3: " << getSubtreeSum(3, M) << "\n"; // 30+40+50 = 120
    cout << "Sum of subtree rooted at 1: " << getSubtreeSum(1, M) << "\n"; // 10+20+30+40+50 = 150
}
```

---

## ⛓️ 7. Euler Tour Tree vs Link-Cut Tree

| 항목 | Euler Tour Tree | Link-Cut Tree |
|------|------------------|----------------|
| 기반 구조 | DFS 기반 + 배열 | Splay Tree |
| 갱신 방식 | 전체 DFS 필요 | 부분적 갱신 가능 |
| 연결/절단 | 느림 (재 DFS 필요) | 빠름 (O(log n)) |
| 구현 난이도 | 쉬움 | 어려움 |
| 응용 | 정적 트리 질의 | 동적 연결 그래프 |

---

## ✅ 정리

| 항목 | 설명 |
|------|------|
| 목적 | 트리를 배열로 표현하여 질의 최적화 |
| 핵심 | 오일러 순회 + 범위 쿼리 |
| 응용 | 서브트리 합, 갱신, 경로 질의 등 |
| 한계 | 간선 변경이 빈번한 경우 비효율적 |