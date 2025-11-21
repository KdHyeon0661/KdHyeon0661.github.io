---
layout: post
title: Data Structure - 이진 트리
date: 2024-12-15 19:20:23 +0900
category: Data Structure
---
# AVL 트리 — 자가 균형 이진 탐색 트리 (Self-Balancing BST)

## 왜 AVL 트리인가?

일반 BST는 **삽입 순서에 따라 편향**될 수 있다.

```text
삽입: 10 → 20 → 30 → 40 → 50

    10
      \
      20
        \
        30
          \
          40
            \
            50
```

- 사실상 **연결 리스트**와 동일.
- 탐색/삽입/삭제 시간: \(O(n)\) 로 **퇴화**.

**자가 균형 트리**(AVL)는 삽입/삭제 후 **자동 회전(rotation)**으로 **높이 \(O(\log n)\)**을 보장한다.

---

## 정의와 불변식

**AVL 트리**는 모든 노드 \(v\)에 대해

$$
\mathrm{bf}(v) \;=\; \mathrm{height}(v_\text{left}) - \mathrm{height}(v_\text{right}) \;\in\; \{-1, 0, +1\}
$$

을 유지한다.
여기서 빈 서브트리 높이는 \(0\) 또는 \(-1\) 중 하나로 정의할 수 있지만, 이 글에서는 **빈 노드 높이 0**, 실제 노드 높이는

$$
\mathrm{height}(v) \;=\; 1 + \max(\mathrm{height}(v_\text{left}), \mathrm{height}(v_\text{right}))
$$

를 사용한다.

> 관례: 구현에서 높이 필드를 1-기준으로 저장(리프=1, 널=0).
> 수학 서술과 구현의 기준만 **일관**되게 유지하면 된다.

---

## 회전(Rotation)

불균형이 발생한 **최상위 조상**(불균형 첫 노드)을 \(z\)라 하자.
\(z\)의 **무거운 쪽** 자식을 \(y\), \(y\)의 무거운 쪽 자식을 \(x\)라 두면 아래 4가지 케이스로 귀결된다.

### LL (Left-Left)

- 상황: 왼쪽 서브트리의 **왼쪽**에 삽입/높이 증가
- 처방: **오른쪽 회전** \( \text{RightRotate}(z) \)

```
    z                      y
   / \                   /   \
  y   C     →           A     z
 / \                         / \
A   B                       B   C
```

### RR (Right-Right)

- 상황: 오른쪽 서브트리의 **오른쪽**에 삽입/높이 증가
- 처방: **왼쪽 회전** \( \text{LeftRotate}(z) \)

```
  z                          y
 / \                       /   \
A   y        →            z     C
   / \                   / \
  B   C                 A   B
```

### LR (Left-Right)

- 상황: 왼쪽 서브트리의 **오른쪽**에 삽입/높이 증가
- 처방: \(y\)에 **LeftRotate**, 이어서 \(z\)에 **RightRotate**

```
    z                        z                         x
   / \                      / \                      /   \
  y   D      →            x   D       →            y       z
 / \                      / \                      / \     / \
A   x                    y   C                    A   B   C   D
   / \                  / \
  B   C                A   B
```

### RL (Right-Left)

- 상황: 오른쪽 서브트리의 **왼쪽**에 삽입/높이 증가
- 처방: \(y\)에 **RightRotate**, 이어서 \(z\)에 **LeftRotate**

```
  z                          z                           x
 / \                        / \                        /   \
A   y        →             A   x        →            z       y
   / \                        / \                    / \     / \
  x   D                      B   y                  A   B   C   D
 / \                            / \
B   C                          C   D
```

---

## C++ 기본 뼈대

```cpp
#include <bits/stdc++.h>

using namespace std;

struct Node {
    int key;
    Node *left, *right;
    int height; // 1-based (leaf=1, null=0)
    explicit Node(int k): key(k), left(nullptr), right(nullptr), height(1) {}
};

int h(Node* n) { return n ? n->height : 0; }

int bf(Node* n) { return n ? h(n->left) - h(n->right) : 0; }

void pull(Node* n) {
    if (n) n->height = 1 + max(h(n->left), h(n->right));
}
```

### 회전 구현

```cpp
Node* rotateRight(Node* y) {
    Node* x = y->left;
    Node* T2 = x->right;

    // 회전
    x->right = y;
    y->left = T2;

    // 높이 갱신 (아래→위)
    pull(y);
    pull(x);
    return x;
}

Node* rotateLeft(Node* x) {
    Node* y = x->right;
    Node* T2 = y->left;

    // 회전
    y->left = x;
    x->right = T2;

    // 높이 갱신
    pull(x);
    pull(y);
    return y;
}
```

---

## + 재균형

AVL 삽입은 **BST 삽입 → 올라오며 높이/균형 검사 → 필요한 회전** 순서로 진행된다.
중복 정책은 본 구현에서 “**금지**”로 둔다(필요 시 카운트 보강으로 확장).

```cpp
Node* rebalance(Node* n) {
    pull(n);
    int B = bf(n);

    // LL
    if (B > 1 && bf(n->left) >= 0) return rotateRight(n);

    // LR
    if (B > 1 && bf(n->left) < 0) {
        n->left = rotateLeft(n->left);
        return rotateRight(n);
    }

    // RR
    if (B < -1 && bf(n->right) <= 0) return rotateLeft(n);

    // RL
    if (B < -1 && bf(n->right) > 0) {
        n->right = rotateRight(n->right);
        return rotateLeft(n);
    }

    return n; // already balanced
}

Node* insert(Node* root, int key) {
    if (!root) return new Node(key);
    if (key < root->key)      root->left  = insert(root->left, key);
    else if (key > root->key) root->right = insert(root->right, key);
    else return root; // 중복 금지

    return rebalance(root);
}
```

> 사실 삽입 시 **최대 한 번의 단/복합 회전**(single/double)만 발생한다.

### 삽입 단계 예시(LL)

```
빈 → 30 삽입
    30

→ 20 삽입
    30
   /
  20

→ 10 삽입 (z=30, y=20, x=10 → LL)
      30                20
     /        →        /  \
    20               10    30
   /
  10
```

---

## + 재균형

삭제는 **BST 삭제 로직**으로 노드를 제거한 뒤, **재균형**을 수행한다.
삭제에서는 **경로 상 여러 위치에서 회전이 반복**될 수 있다(최대 \(O(\log n)\)회).

```cpp
Node* minNode(Node* n) { while (n && n->left) n = n->left; return n; }

Node* erase(Node* root, int key) {
    if (!root) return nullptr;

    if (key < root->key)      root->left  = erase(root->left, key);
    else if (key > root->key) root->right = erase(root->right, key);
    else {
        // 0 or 1 child
        if (!root->left || !root->right) {
            Node* t = root->left ? root->left : root->right;
            delete root;
            return t;
        }
        // 2 children: in-order successor로 대체
        Node* s = minNode(root->right);
        root->key = s->key;
        root->right = erase(root->right, s->key);
    }
    return rebalance(root);
}
```

### 삭제 회전 케이스 표

| Case | 조건(노드 `v`) | 조치 |
|---|---|---|
| LL | `bf(v) = +2` and `bf(v->left) ≥ 0` | `rotateRight(v)` |
| LR | `bf(v) = +2` and `bf(v->left) < 0` | `v->left=rotateLeft(v->left)` → `rotateRight(v)` |
| RR | `bf(v) = -2` and `bf(v->right) ≤ 0` | `rotateLeft(v)` |
| RL | `bf(v) = -2` and `bf(v->right) > 0` | `v->right=rotateRight(v->right)` → `rotateLeft(v)` |

> 삭제는 삽입과 달리 **여러 조상**에서 연쇄적으로 균형이 깨질 수 있으므로 루트까지 **되올라가며** 검사한다.

---

## 순회/검증/유틸

```cpp
void inorder(Node* n){ if(!n) return; inorder(n->left); cout<<n->key<<" "; inorder(n->right); }
void preorder(Node* n){ if(!n) return; cout<<n->key<<" "; preorder(n->left); preorder(n->right); }
void postorder(Node* n){ if(!n) return; postorder(n->left); postorder(n->right); cout<<n->key<<" "; }

bool isAVL(Node* n, int& outHeight) {
    if (!n) { outHeight = 0; return true; }
    int hl, hr;
    if (!isAVL(n->left, hl)) return false;
    if (!isAVL(n->right, hr)) return false;
    int bal = hl - hr;
    bool ok = (bal >= -1 && bal <= 1);
    outHeight = 1 + max(hl, hr);
    return ok && (n->height == outHeight);
}

void levelOrder(Node* root){
    if(!root) return;
    queue<Node*> q; q.push(root);
    while(!q.empty()){
        auto* u=q.front(); q.pop();
        cout<<u->key<<" ";
        if(u->left) q.push(u->left);
        if(u->right) q.push(u->right);
    }
}
```

### floor / ceil (lower/upper bound)

```cpp
Node* floorKey(Node* n, int x){ // <= x 중 최대
    Node* ans=nullptr;
    while(n){
        if(n->key==x) return n;
        if(n->key < x){ ans=n; n=n->right; }
        else n=n->left;
    }
    return ans;
}
Node* ceilKey(Node* n, int x){ // >= x 중 최소
    Node* ans=nullptr;
    while(n){
        if(n->key==x) return n;
        if(n->key > x){ ans=n; n=n->left; }
        else n=n->right;
    }
    return ans;
}
```

---

## 예제: 삽입/삭제 동작

```cpp
int main(){
    Node* root=nullptr;
    for(int x: {30, 20, 40, 10, 25, 35, 50, 5, 4}) {
        root = insert(root, x);
    }

    cout<<"Inorder: "; inorder(root); cout<<"\n";
    cout<<"Level:   "; levelOrder(root); cout<<"\n";

    int hh=0;
    cout<<"isAVL? "<<(isAVL(root, hh) ? "true" : "false")<<", height="<<hh<<"\n";

    // 삭제 테스트
    for(int del: {20, 40, 30}) {
        root = erase(root, del);
        cout<<"After erase("<<del<<") Inorder: "; inorder(root); cout<<"\n";
        cout<<"Level: "; levelOrder(root); cout<<"\n";
        cout<<"AVL? "<<(isAVL(root, hh) ? "true" : "false")<<", height="<<hh<<"\n";
    }
}
```

---

## 반복형 삽입(경로 스택 이용)

부모 포인터 없이도, 삽입 경로를 스택에 기록해 **되올라가며** `pull+rebalance`를 수행할 수 있다.

```cpp
Node* insertIter(Node* root, int key){
    if(!root) return new Node(key);

    // 1) BST 삽입 + 경로 스택
    vector<Node*> path;
    Node* cur=root;
    while(cur){
        path.push_back(cur);
        if(key == cur->key) return root; // 중복 금지
        cur = (key < path.back()->key) ? path.back()->left : path.back()->right;
    }

    // 실제 삽입 지점 찾기
    Node* parent = path.back();
    Node* leaf = new Node(key);
    if(key < parent->key) parent->left = leaf; else parent->right = leaf;

    // 2) 되올라가며 재균형
    for(int i=(int)path.size()-1; i>=0; --i){
        Node* p = path[i];

        // p의 자식 링크를 rebalance 결과로 치환해줘야 함.
        // 부모가 있으면 부모의 좌/우를 갱신, 없으면 root를 갱신.
        Node* pp = (i>0) ? path[i-1] : nullptr;

        Node* newp = rebalance(p);
        if(!pp) root = newp;
        else {
            if(pp->left == p)  pp->left = newp;
            else               pp->right = newp;
        }
    }
    return root;
}
```

> 반복형 삭제도 동일 아이디어로 구현 가능(경로 스택 또는 부모 포인터로 되올라가기).

---

## 정렬 배열 → AVL 빌드

정렬 배열로부터 **완전 균형 BST**를 만들면, 이는 자동으로 AVL 조건을 만족한다.

```cpp
Node* buildFromSorted(const vector<int>& a, int l, int r){
    if(l>r) return nullptr;
    int m = (l+r)/2;
    Node* n = new Node(a[m]);
    n->left  = buildFromSorted(a, l, m-1);
    n->right = buildFromSorted(a, m+1, r);
    pull(n);
    return n;
}
```

---

## 범위 쿼리 예시

```cpp
void rangePrint(Node* n, int L, int R){
    if(!n) return;
    if(n->key > L) rangePrint(n->left, L, R);
    if(L <= n->key && n->key <= R) cout<<n->key<<" ";
    if(n->key < R) rangePrint(n->right, L, R);
}
```

---

## 중복 키 정책 확장(카운트 보강)

간단히 `count` 필드를 추가하면 **멀티셋**처럼 동작한다.

```cpp
struct MNode {
    int key, count, height;
    MNode *left, *right;
    explicit MNode(int k): key(k), count(1), height(1), left(nullptr), right(nullptr) {}
};
// insert: 동일 키면 count++, erase: count>1이면 count-- 후 반환
// 나머지 rebalance 로직은 동일 (높이 갱신은 key와 무관)
```

---

## 수학: 높이 상계와 회전 수

### 최소 노드 수 재귀

높이가 \(h\)인 AVL 트리의 **최소 노드 수**를 \(N(h)\)라 하자(리프=높이 1, 널=0 기준).
균형계수 \(\in\{-1,0,+1\}\) 조건으로부터

$$
N(h) \;=\; 1 + N(h-1) + N(h-2),\quad N(1)=1,\; N(0)=0
$$

을 얻는다. 이는 사실상 **피보나치**와 같은 점화식이다.
귀납적으로

$$
N(h) \;\ge\; F_{h+1} \;\;\Rightarrow\;\; h \;\le\; \mathcal{O}(\log n)
$$

이며, 좀 더 정밀히는

$$
h \;\le\; 1.44 \,\log_2(n + 2) \;-\; 0.328 \quad \text{(근사)}
$$

를 얻는다. 즉 **항상 로그 높이**가 보장된다.

### 회전 수

- **삽입**: 불균형 **최상위 한 곳**에서 **단/복합 회전 1회**면 충분. ⇒ **회전 수 \(O(1)\)**
- **삭제**: 높이가 감소하며 **여러 조상**에서 연쇄적으로 불균형이 날 수 있어, **최대 \(O(\log n)\)** 회전.

---

## 복잡도

| 연산 | 시간 | 비고 |
|---|---|---|
| 탐색 | \(O(\log n)\) | BST와 동일(항상 로그 높이) |
| 삽입 | \(O(\log n)\) | 경로 갱신 + **회전 \(O(1)\)** |
| 삭제 | \(O(\log n)\) | 경로 갱신 + **회전 \(O(\log n)\)** 가능 |
| 공간 | \(O(n)\) | 노드 1개당 `key,left,right,height` |

---

## 직렬화/역직렬화(전위 + 널 토큰)

BST/AVL 모두에 사용 가능한 간결한 방법이다.

```cpp
void serialize(Node* n, ostream& os){
    if(!n){ os<<"# "; return; }
    os<<n->key<<" ";
    serialize(n->left, os);
    serialize(n->right, os);
}
Node* deserialize(istringstream& iss){
    string t; if(!(iss>>t)) return nullptr;
    if(t=="#") return nullptr;
    Node* n=new Node(stoi(t));
    n->left  = deserialize(iss);
    n->right = deserialize(iss);
    pull(n);
    return n;
}
```

---

## 실전 팁과 버그 포인트

1. **pull 순서**: 회전 직후 **아래 노드부터** 높이를 갱신하고, 그 다음 상위 노드를 갱신한다.
2. **bf 분기**: `>=, <=` 경계 실수 주의(삽입/삭제 케이스 표와 일치시킬 것).
3. **중복 정책**: 금지/카운트 보강 중 하나를 **일관되게**.
4. **메모리 관리**: 테스트 종료 시 모든 노드 `delete`(리크 검사).
5. **검증 도구**: `isAVL`로 높이/균형/저장 height 일치까지 확인.
6. **반복 구현**: 경로 스택으로 부모 포인터 없이도 회전 결과를 부모에 **재연결**하는 코드가 핵심.

---

## 종합 예제(하나의 파일)

```cpp
#include <bits/stdc++.h>

using namespace std;

struct Node {
    int key; Node *left,*right; int height;
    explicit Node(int k): key(k), left(nullptr), right(nullptr), height(1) {}
};

int h(Node* n){ return n? n->height:0; }
void pull(Node* n){ if(n) n->height = 1 + max(h(n->left), h(n->right)); }
int bf(Node* n){ return n? h(n->left)-h(n->right):0; }

Node* rotateRight(Node* y){
    Node* x=y->left; Node* T2=x->right;
    x->right=y; y->left=T2;
    pull(y); pull(x);
    return x;
}
Node* rotateLeft(Node* x){
    Node* y=x->right; Node* T2=y->left;
    y->left=x; x->right=T2;
    pull(x); pull(y);
    return y;
}

Node* rebalance(Node* n){
    pull(n);
    int B = bf(n);
    if(B > 1 && bf(n->left) >= 0) return rotateRight(n);              // LL
    if(B > 1 && bf(n->left) <  0){ n->left=rotateLeft(n->left); return rotateRight(n);} // LR
    if(B < -1 && bf(n->right)<= 0) return rotateLeft(n);              // RR
    if(B < -1 && bf(n->right)>  0){ n->right=rotateRight(n->right); return rotateLeft(n);} // RL
    return n;
}

Node* insert(Node* root, int key){
    if(!root) return new Node(key);
    if(key < root->key)      root->left  = insert(root->left, key);
    else if(key > root->key) root->right = insert(root->right, key);
    else return root; // dup forbid
    return rebalance(root);
}

Node* minNode(Node* n){ while(n&&n->left) n=n->left; return n; }

Node* erase(Node* root, int key){
    if(!root) return nullptr;
    if(key < root->key)      root->left  = erase(root->left, key);
    else if(key > root->key) root->right = erase(root->right, key);
    else{
        if(!root->left || !root->right){
            Node* t = root->left? root->left : root->right;
            delete root; return t;
        }
        Node* s = minNode(root->right);
        root->key = s->key;
        root->right = erase(root->right, s->key);
    }
    return rebalance(root);
}

void inorder(Node* n){ if(!n) return; inorder(n->left); cout<<n->key<<" "; inorder(n->right); }
void levelOrder(Node* root){
    if(!root) return; queue<Node*> q; q.push(root);
    while(!q.empty()){ auto* u=q.front(); q.pop(); cout<<u->key<<" ";
        if(u->left) q.push(u->left); if(u->right) q.push(u->right); }
}
bool isAVL(Node* n, int& outH){
    if(!n){ outH=0; return true; }
    int hl,hr;
    if(!isAVL(n->left, hl)) return false;
    if(!isAVL(n->right, hr)) return false;
    outH = 1+max(hl,hr);
    if(n->height != outH) return false;
    int B = hl-hr;
    return (B>=-1 && B<=1);
}

int main(){
    Node* root=nullptr;
    vector<int> seq = {30, 20, 40, 10, 25, 35, 50, 5, 4, 45, 42, 43};
    for(int x: seq){ root = insert(root, x); }

    cout<<"Inorder : "; inorder(root); cout<<"\n";
    cout<<"Level   : "; levelOrder(root); cout<<"\n";
    int hh=0; cout<<"isAVL? "<<(isAVL(root, hh)?"true":"false")<<", height="<<hh<<"\n";

    for(int del : {20, 40, 30}) {
        root = erase(root, del);
        cout<<"After erase("<<del<<")\n";
        cout<<"  Inorder: "; inorder(root); cout<<"\n";
        cout<<"  Level  : "; levelOrder(root); cout<<"\n";
        cout<<"  AVL? "<<(isAVL(root, hh)?"true":"false")<<", height="<<hh<<"\n";
    }
}
```

---

## Red-Black 트리와 비교(요지)

| 항목 | AVL | Red-Black |
|---|---|---|
| 균형 기준 | 높이 차이 \(\le 1\) (더 엄격) | 색 규칙(느슨) |
| 탐색 성능 | **더 좋음**(평균 높이 낮음) | 다소 불리 |
| 삽입 회전 | \(O(1)\) (최대 2회) | 최대 2회 |
| 삭제 회전 | **최대 \(O(\log n)\)** | 평균 적음(색 재도색多) |
| 적용 | **읽기 빈번**(DB 인덱스/캐시 등) | **갱신 빈번**(OS 맵/언어 런타임) |

---

## 퍼징/테스트 아이디어

- 무작위 연산열(삽입/삭제/탐색)을 생성해 `std::multiset<int>`과 **정렬 결과**를 비교.
- 매 스텝 `isAVL`로 높이/균형/저장 height 일치 확인.
- 삭제 반복 시 **루트까지 재균형**이 잘 전파되는지 확인.

```cpp
// pseudo
// for t in trials:
//   Node* root=nullptr; multiset<int> ms;
//   for step in steps:
//     pick op in {ins, del, chk}:
//       if ins: x=rand(); root=insert(root,x); ms.insert(x);
//       if del: x=rand(); root=erase(root,x); auto it=ms.find(x); if(it!=ms.end()) ms.erase(it);
//       if chk: vector<int> v1; inorderCollect(root,v1);
//               vector<int> v2(ms.begin(), ms.end());
//               assert(v1==v2); int hh; assert(isAVL(root,hh));
```

---

## 정리

- **AVL 불변식**: 모든 노드에서 \( \mathrm{bf} \in \{-1,0,+1\} \).
- **삽입**: BST 삽입 후 **단/복합 회전 1회**로 복구.
- **삭제**: BST 삭제 후 경로 따라 **여러 회전** 가능.
- **항상 로그 높이**(피보나치 하계로 도출), 탐색/삽입/삭제는 \(O(\log n)\).
- **검증/유틸**(isAVL, level, floor/ceil, 직렬화)과 **반복형 삽입**으로 실전성을 강화.
- 읽기 성능이 중요한 시나리오에서 **AVL**은 여전히 강력한 선택지다.
