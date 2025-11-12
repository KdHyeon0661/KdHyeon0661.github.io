---
layout: post
title: Data Structure - 이진 트리
date: 2024-12-14 19:20:23 +0900
category: Data Structure
---
# 이진 트리 & 이진 탐색 트리(BST)

## 1. 이진 트리란?

> **이진 트리(Binary Tree)** 는 각 노드가 **최대 2개의 자식 노드**(왼쪽, 오른쪽)를 갖는 트리다.

---

## 2. 이진 트리의 종류

| 분류 | 설명 |
|---|---|
| **완전 이진 트리 (Complete Binary Tree)** | 마지막 레벨을 제외한 모든 레벨이 가득 차 있으며, 마지막 레벨은 **왼쪽부터** 채워진다. |
| **포화 이진 트리 (Full Binary Tree)** | 모든 내부 노드의 자식 수가 **정확히 2**(즉, 자식이 1개인 노드는 없다). |
| **정 포화 이진 트리 (Perfect Binary Tree)** | 모든 리프의 깊이가 동일하고, 모든 내부 노드가 2개의 자식을 갖는다. (완전 + 포화) |

### 예시 비교

```
        1
       / \
      2   3
     / \ / \
    4  5 6  7
```

- → **완전 이진 트리** ✅  
- → **포화 이진 트리** ✅  
- → **정 포화 이진 트리** ✅

```
        1
       / \
      2   3
     / \
    4   5
```

- → **완전** ✅ (왼쪽부터 빈틈없이 채움)  
- → **포화** ❌ (노드 3은 자식 0, 노드 2는 자식 2로 일관성이 깨지진 않지만, “모든 내부 노드가 2자식” 조건을 **전체**가 만족해야 한다. 여기서 내부 노드 1은 자식 2이지만, 내부 노드 3은 내부 노드가 아니다. 혼동을 피하려면 **모든 내부 노드가 자식 2** 조건을 구조 전체에서 확인해야 하고, 상도표현에 따라 오해가 생길 수 있어, 포화 트리의 예시로는 첫 그림이 안전하다.)  
- → **정 포화** ❌ (리프 깊이 불일치)

> 주의: 교재·블로그마다 “포화”를 엄밀히 정의하는 문장 차이로 예시 해석이 엇갈리기 쉽다. 안전하게는 “**모든 내부 노드가 자식 2, 모든 리프의 깊이 동일**”까지 만족하는 **정 포화**를 엄밀한 포화로 소개하기도 한다. 본 문서는 위 표의 정의를 따른다.

---

## 3. 노드 정의 & 표현 방식

### 3.1 포인터 기반(클래식)

```cpp
struct Node {
    int data;
    Node* left;
    Node* right;
    explicit Node(int x) : data(x), left(nullptr), right(nullptr) {}
};
```

- 장점: 일반적인 BST, 다양한 비완전 트리 표현에 적합.  
- 단점: 연속 메모리가 아니라 캐시 친화성 ↓.

### 3.2 배열 기반(완전/힙형 레이아웃)

완전 이진 트리는 배열로도 표현이 쉽다(인덱스 `i`에서 `L=2i+1`, `R=2i+2`, `P=(i-1)/2`).  
BST는 **완전**일 필요가 없으므로 배열 표현의 **널 슬롯 관리 비용**이 커서 일반적이지 않다(힙과 달리 **BST는 포인터형**이 보편).

---

## 4. 순회(Traversal)

트리는 선형이 아니므로 순회 순서에 따라 방문 결과가 달라진다.  
대표 3순회(전/중/후위) + 레벨 순회(BFS) + 반복/스택 기반 + **Morris**(O(1) 보조 메모리).

아래 예시를 위해 동일한 `Node` 정의를 가정한다.

```cpp
#include <bits/stdc++.h>
using namespace std;

struct Node {
    int data; Node* left; Node* right;
    explicit Node(int x): data(x), left(nullptr), right(nullptr) {}
};
```

### 4.1 전위/중위/후위(재귀)

```cpp
void preorder(Node* n){ if(!n) return; cout<<n->data<<" "; preorder(n->left); preorder(n->right); }
void inorder (Node* n){ if(!n) return; inorder(n->left); cout<<n->data<<" "; inorder(n->right); }
void postorder(Node* n){ if(!n) return; postorder(n->left); postorder(n->right); cout<<n->data<<" "; }
```

> **BST에서 `inorder`는 오름차순 출력**을 보장한다.

### 4.2 중위(반복, 스택 활용)

```cpp
void inorderIter(Node* root){
    stack<Node*> st; Node* cur = root;
    while(cur || !st.empty()){
        while(cur){ st.push(cur); cur = cur->left; }
        cur = st.top(); st.pop();
        cout<<cur->data<<" ";
        cur = cur->right;
    }
}
```

### 4.3 레벨 순회(BFS, 큐)

```cpp
void levelOrder(Node* root){
    if(!root) return;
    queue<Node*> q; q.push(root);
    while(!q.empty()){
        Node* u=q.front(); q.pop();
        cout<<u->data<<" ";
        if(u->left)  q.push(u->left);
        if(u->right) q.push(u->right);
    }
}
```

### 4.4 **Morris 중위 순회**(O(1) 보조 메모리)

임시 스레딩을 이용해 스택/재귀 없이 중위 순회:

```cpp
void inorderMorris(Node* root){
    Node* cur = root;
    while(cur){
        if(!cur->left){
            cout<<cur->data<<" ";
            cur = cur->right;
        }else{
            Node* pred = cur->left;
            while(pred->right && pred->right!=cur) pred = pred->right;
            if(!pred->right){ pred->right = cur; cur = cur->left; }
            else{ pred->right=nullptr; cout<<cur->data<<" "; cur = cur->right; }
        }
    }
}
```

> 동시 접근 금지(순회 중 구조를 임시로 바꾼다). 순회 종료 시 원형 복원됨.

---

## 5. 이진 탐색 트리(BST): 삽입/탐색/삭제

> 정의: 각 노드 `x`에 대해 **왼쪽 서브트리 < x < 오른쪽 서브트리**(키 기준).  
> 본문에서는 키가 `data` 필드에 저장된다고 가정한다.

### 5.1 중복 처리 정책

- **금지**: 동일 키 삽입 시 무시(본 예시 기본값)  
- **왼쪽 허용**: `<=`는 왼쪽으로  
- **오른쪽 허용**: `<`는 왼쪽, `>=`는 오른쪽  
- **카운트 보강**: 노드가 `(key, count)` 를 보유(중복 키 빈도 저장)

### 5.2 삽입/탐색

```cpp
Node* insert(Node* root, int val){
    if(!root) return new Node(val);
    if(val < root->data) root->left  = insert(root->left, val);
    else if(val > root->data) root->right = insert(root->right, val);
    return root; // 중복 금지 정책
}

bool search(Node* root, int val){
    if(!root) return false;
    if(val == root->data) return true;
    return (val < root->data) ? search(root->left, val) : search(root->right, val);
}
```

### 5.3 선행자/후속자 (in-order predecessor/successor)

```cpp
Node* findMin(Node* n){ while(n && n->left ) n = n->left;  return n; }
Node* findMax(Node* n){ while(n && n->right) n = n->right; return n; }

Node* successor(Node* root, int k){
    Node* cur=root; Node* ans=nullptr;
    while(cur){
        if(k < cur->data){ ans=cur; cur=cur->left; }
        else cur=cur->right;
    }
    return ans;
}
Node* predecessor(Node* root, int k){
    Node* cur=root; Node* ans=nullptr;
    while(cur){
        if(k > cur->data){ ans=cur; cur=cur->right; }
        else cur=cur->left;
    }
    return ans;
}
```

### 5.4 삭제(재귀) — 후속자 교체

```cpp
Node* deleteNode(Node* root, int val){
    if(!root) return nullptr;
    if(val < root->data) root->left  = deleteNode(root->left, val);
    else if(val > root->data) root->right = deleteNode(root->right, val);
    else{
        if(!root->left){
            Node* r = root->right; delete root; return r;
        }else if(!root->right){
            Node* l = root->left;  delete root; return l;
        }
        Node* s = findMin(root->right);    // in-order successor
        root->data = s->data;
        root->right = deleteNode(root->right, s->data);
    }
    return root;
}
```

> **선행자 교체** 버전은 `findMax(root->left)` 를 써서 동일하게 구현 가능.

### 5.5 삭제(반복) — 스택 없이

```cpp
Node* deleteIter(Node* root, int val){
    Node* parent=nullptr; Node* cur=root;
    while(cur && cur->data!=val){
        parent=cur;
        cur = (val<cur->data)?cur->left:cur->right;
    }
    if(!cur) return root; // not found

    auto link = [&](Node* child){
        if(!parent) root=child;
        else if(parent->left==cur) parent->left=child;
        else parent->right=child;
    };

    if(!cur->left || !cur->right){ // 0 or 1 child
        Node* child = cur->left ? cur->left : cur->right;
        link(child); delete cur; return root;
    }
    // two children: find successor
    Node* ps=cur; Node* s=cur->right;
    while(s->left){ ps=s; s=s->left; }
    cur->data = s->data;
    if(ps->left==s) ps->left=s->right; else ps->right=s->right;
    delete s; 
    return root;
}
```

---

## 6. 예시 — 구성·순회·삭제

```cpp
int main(){
    Node* root=nullptr;
    int values[] = {8,3,10,1,6,14,4,7};

    for(int v: values) root = insert(root, v);

    cout<<"Inorder before delete: "; inorder(root); cout<<"\n";
    root = deleteNode(root, 3);   // 두 자식(1,6)을 가진 노드 3 제거: 후속자로 교체
    cout<<"Inorder after delete:  "; inorder(root); cout<<"\n";
}
```

**출력 예시**
```
Inorder before delete: 1 3 4 6 7 8 10 14
Inorder after delete:  1 4 6 7 8 10 14
```

---

## 7. 검증/유틸: 높이·노드/리프 수·isBST

```cpp
int height(Node* n){ if(!n) return -1; return 1 + max(height(n->left), height(n->right)); }
int countNodes(Node* n){ return n?1+countNodes(n->left)+countNodes(n->right):0; }
int countLeaves(Node* n){ if(!n) return 0; if(!n->left && !n->right) return 1; return countLeaves(n->left)+countLeaves(n->right); }
```

**BST 검증**(범위 전파 방식: 중복 금지 기준 `<`/`>` 사용)

```cpp
bool isBST(Node* n, long long lo=LLONG_MIN, long long hi=LLONG_MAX){
    if(!n) return true;
    if(!(lo < n->data && n->data < hi)) return false;
    return isBST(n->left, lo, n->data) && isBST(n->right, n->data, hi);
}
```

> 중복 허용 정책을 쓰면 비교식(예: `<=`/`>=`)을 정책에 맞춰 수정해야 한다.

---

## 8. 직렬화·역직렬화

전위 + 널 토큰(`#`) 방식을 사용한다.

```cpp
void serialize(Node* n, ostream& os){
    if(!n){ os << "# "; return; }
    os << n->data << " ";
    serialize(n->left, os);
    serialize(n->right, os);
}
Node* deserialize(istringstream& iss){
    string tok; if(!(iss>>tok)) return nullptr;
    if(tok=="#") return nullptr;
    Node* n = new Node(stoi(tok));
    n->left  = deserialize(iss);
    n->right = deserialize(iss);
    return n;
}
```

**레벨 순서 직렬화**도 가능(큐 사용)하나, `#` 관리와 후단의 불필요한 `#` 제거 정책을 함께 정해야 한다. 간결성 측면에서 전위+널 토큰이 실전에서도 자주 쓰인다.

---

## 9. 범위 쿼리/집계

### 9.1 범위 출력 \([L,R]\)

```cpp
void rangePrint(Node* n, int L, int R){
    if(!n) return;
    if(n->data > L) rangePrint(n->left, L, R);
    if(L <= n->data && n->data <= R) cout<<n->data<<" ";
    if(n->data < R) rangePrint(n->right, L, R);
}
```

### 9.2 범위 합(키 합계)

```cpp
long long rangeSum(Node* n, int L, int R){
    if(!n) return 0;
    if(n->data < L) return rangeSum(n->right, L, R);
    if(n->data > R) return rangeSum(n->left,  L, R);
    return n->data + rangeSum(n->left, L, R) + rangeSum(n->right, L, R);
}
```

### 9.3 floor/ceil(lower_bound/upper_bound 유사)

```cpp
Node* floorBST(Node* root, int x){ // <= x 중 가장 큰 키
    Node* cur=root; Node* ans=nullptr;
    while(cur){
        if(cur->data==x) return cur;
        if(cur->data < x){ ans=cur; cur=cur->right; }
        else cur=cur->left;
    }
    return ans;
}
Node* ceilBST(Node* root, int x){ // >= x 중 가장 작은 키
    Node* cur=root; Node* ans=nullptr;
    while(cur){
        if(cur->data==x) return cur;
        if(cur->data > x){ ans=cur; cur=cur->left; }
        else cur=cur->right;
    }
    return ans;
}
```

---

## 10. 순서 통계(Order Statistics): k번째 & rank

노드에 **서브트리 크기 `sz`** 를 보강한다.

```cpp
struct OSNode{
    int data, sz; OSNode *left,*right;
    explicit OSNode(int x): data(x), sz(1), left(nullptr), right(nullptr) {}
};
int size(OSNode* n){ return n? n->sz:0; }
void pull(OSNode* n){ if(n) n->sz = 1 + size(n->left) + size(n->right); }

OSNode* insert(OSNode* r, int x){
    if(!r) return new OSNode(x);
    if(x < r->data) r->left = insert(r->left, x);
    else if(x > r->data) r->right= insert(r->right,x);
    pull(r); return r;
}
OSNode* findMin(OSNode* n){ while(n&&n->left) n=n->left; return n; }

OSNode* erase(OSNode* r, int x){
    if(!r) return nullptr;
    if(x < r->data) r->left = erase(r->left,x);
    else if(x > r->data) r->right= erase(r->right,x);
    else{
        if(!r->left || !r->right){
            OSNode* t = r->left? r->left : r->right;
            delete r; return t;
        }else{
            OSNode* s = findMin(r->right);
            r->data = s->data;
            r->right = erase(r->right, s->data);
        }
    }
    pull(r); return r;
}

// 1-indexed kth
OSNode* kth(OSNode* r, int k){
    if(!r || k<=0 || k>size(r)) return nullptr;
    int L = size(r->left);
    if(k == L+1) return r;
    if(k <= L)   return kth(r->left, k);
    return kth(r->right, k-(L+1));
}

// rank: 키 x의 "오름차순 위치"(1-indexed)
int rankOf(OSNode* r, int x){
    if(!r) return 0;
    if(x < r->data) return rankOf(r->left, x);
    if(x > r->data) return size(r->left)+1 + rankOf(r->right, x);
    return size(r->left)+1;
}
```

> 실전에서는 **균형 트리(AVL/RB)** 위에 `sz`를 보강해 최악도 \(O(\log n)\) 보장.

---

## 11. 정렬 배열 ↔ 균형 BST

### 11.1 정렬 배열 → 높이 균형 BST

```cpp
Node* buildBalanced(const vector<int>& a, int l, int r){
    if(l>r) return nullptr;
    int m = (l+r)/2;
    Node* n = new Node(a[m]);
    n->left  = buildBalanced(a, l, m-1);
    n->right = buildBalanced(a, m+1, r);
    return n;
}
```

### 11.2 BST → 정렬 배열(중위 수집)

```cpp
void toSortedArray(Node* n, vector<int>& out){
    if(!n) return;
    toSortedArray(n->left, out);
    out.push_back(n->data);
    toSortedArray(n->right, out);
}
```

---

## 12. 테스트·퍼징 & 실전 팁

- **중복 정책**을 코드로 **일관**되게 강제(비교식/경계 포함 여부 통일).  
- 삭제 2자식 케이스: 후속자/선행자 교체 후 **해당 서브트리에서** 삭제를 수행.  
- `deleteIter`: 부모-자식 링크 치환 실수 주의(왼/오른쪽 분기).  
- **isBST** 검증으로 구조 불변식 점검.  
- **퍼징 아이디어**: 랜덤 시드로 (삽입·삭제·탐색) 시퀀스를 생성하고, 표준 `multiset<int>`와 **정렬 결과/카디널리티**가 항상 일치하는지 비교.

```cpp
// pseudo:
// for many trials:
//   vector<int> ops = randomOps();
//   BST b; multiset<int> ms;
//   for(op in ops){
//     if(op.type==INS){ b.insert(op.x); ms.insert(op.x); }
//     if(op.type==DEL){ b.erase(op.x);  auto it=ms.find(op.x); if(it!=ms.end()) ms.erase(it); }
//     if(op.type==CHK){
//        vector<int> a1 = b.inorderCollect();
//        vector<int> a2(ms.begin(), ms.end());
//        assert(a1==a2 && isBST(b.root));
//     }
//   }
```

- 메모리 관리: 테스트 종료 시 `delete`로 모든 노드를 해제(리크 방지).  
- 깊은 트리(편향) 입력에서는 **재귀 대신 반복/스택** 또는 **Morris** 사용.

---

## 13. 복잡도 & 간단 근거

### 13.1 평균/최악

| 연산 | 평균 | 최악(편향) |
|---|---|---|
| 탐색/삽입/삭제 | \(O(\log n)\) | \(O(n)\) |

### 13.2 스케치

- 트리 높이를 \(h\)라 하면 경로를 따라 내려가는 연산은 \(O(h)\).  
- 균형 시 \(h=\Theta(\log n)\), 편향 시 \(h=\Theta(n)\).  
- **중위 순회 결과가 정렬**되는 이유(귀납):

$$
\text{inorder}(T) \;=\; \text{inorder}(T_\text{left}) \;\Vert\; \{ \text{key(root)} \} \;\Vert\; \text{inorder}(T_\text{right})
$$

좌/우 서브트리의 모든 키가 각각 root보다 작고/크므로 전체 연결도 정렬을 유지한다.

---

## 14. 미니 데모(종합)

```cpp
int main(){
    // 1) 균형 트리 만들기
    vector<int> a = {1,2,3,4,5,6,7,8,9};
    Node* root = buildBalanced(a, 0, (int)a.size()-1);

    // 2) 순회
    cout<<"Pre: "; preorder(root); cout<<"\n";
    cout<<"In : "; inorder(root);  cout<<"\n";
    cout<<"Post: "; postorder(root); cout<<"\n";
    cout<<"Level: "; levelOrder(root); cout<<"\n";

    // 3) 질의
    cout<<"height="<<height(root)<<", nodes="<<countNodes(root)<<", leaves="<<countLeaves(root)<<"\n";
    cout<<"isBST? "<<(isBST(root)?"true":"false")<<"\n";

    // 4) 직렬화/역직렬화
    stringstream ss; serialize(root, ss);
    string dump = ss.str(); cout<<"SER="<<dump<<"\n";
    istringstream iss(dump);
    Node* root2 = deserialize(iss);
    cout<<"In2: "; inorder(root2); cout<<"\n";

    // 5) 범위 출력/합
    cout<<"[3,7]: "; rangePrint(root2, 3, 7); cout<<"\n";
    cout<<"sum[3,7]="<<rangeSum(root2,3,7)<<"\n";

    // 6) 선행자/후속자
    Node* p = predecessor(root2, 5); Node* s = successor(root2, 5);
    cout<<"pred(5)="<<(p?p->data:-1)<<", succ(5)="<<(s?s->data:-1)<<"\n";

    // 7) 삭제 데모
    root2 = deleteNode(root2, 6);
    cout<<"In after del(6): "; inorder(root2); cout<<"\n";

    // OS 트리는 별도 테스트
    OSNode* r=nullptr;
    for(int x: {5,2,8,1,3,7,9}) r=insert(r,x);
    for(int k=1;k<=size(r);++k) cout<<k<<"th="<<kth(r,k)->data<<" ";
    cout<<"\nrank(7)="<<rankOf(r,7)<<"\n";
}
```

---

## 15. 정리

- **표현**: 포인터형(일반/BST), 배열형(완전).  
- **순회**: 전/중/후위 + BFS + 반복/스택 + **Morris**(O(1) 보조 메모리).  
- **BST**: 중복 정책을 명확히, 삽입/탐색/삭제(후속자/선행자 교체, 반복 구현) 숙지.  
- **유틸/검증**: 높이·노드/리프 수, `isBST`, 직렬화/역직렬화.  
- **질의**: 범위 출력/합계, floor/ceil, **k번째/순위**(size 보강).  
- **전환**: 정렬 배열↔균형 BST.  
- **품질**: 퍼징·메모리 관리·경계/중복 정책 일관성.