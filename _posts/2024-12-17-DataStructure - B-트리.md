---
layout: post
title: Data Structure - B-Tree
date: 2024-12-17 19:20:23 +0900
category: Data Structure
---
# B-Tree 자료구조

## B-Tree란?

**B-Tree**는 **균형 다진 탐색 트리**다. 각 노드가 **여러 키와 최대 2t 자식**을 가지며, **모든 리프가 같은 깊이**에 존재한다.
외부 메모리(디스크/SSD)에서 **노드=페이지**로 맞추어 **I/O 횟수를 최소화**하도록 설계되었다.

- 탐색/삽입/삭제: **항상** \(O(\log n)\)
- 한 번의 페이지 로드로 **여러 키**를 비교 → I/O 효율 최적화

---

## 용어와 차수 t

| 용어 | 의미 |
|---|---|
| **차수 t (minimum degree)** | 각 비루트 노드의 최소 키 개수를 결정하는 핵심 파라미터 |
| 키(key) | 노드 내부에 **정렬** 상태로 저장되는 값 |
| 자식 포인터 | 하위 노드를 가리키는 포인터(또는 페이지 ID) |
| 루트 | 최상단 노드(키 최소 개수 예외) |
| 리프 | 자식이 없는 노드(모든 리프의 깊이 동일) |

---

## B-Tree의 성질(불변식)

차수 \(t \ge 2\) 에 대해:

- 한 노드의 키 수: **최소 \(t-1\), 최대 \(2t-1\)** (루트 예외: 최소 1 또는 0)
- 자식 수: **키+1**, 즉 **최소 \(t\), 최대 \(2t\)**
- 노드 내부 키는 **오름차순**, 키 \(k_i\) 는 **경계** 역할:
  - 왼쪽 자식의 모든 키 \< \(k_i\) \< 오른쪽 자식의 모든 키
- **모든 리프의 깊이 동일**(균형)

### 높이와 원소 수 관계

높이를 \(h\)라 하면(루트 깊이 0), 전체 키 수 \(n\)에 대해(루트에 최소 1키 가정):

\[
n \;\ge\; 2\,t^{\,h} - 1
\quad\Longrightarrow\quad
h \;\le\; \left\lfloor \log_{t} \frac{n+1}{2} \right\rfloor
\]

따라서 \(h = O(\log_t n)\), 즉 **항상 로그 높이**를 보장한다.

---

## 왜 B-Tree인가? (I/O 관점)

- 디스크/SSD는 **랜덤 접근 비용**이 크다.
- **노드=페이지(블록) 크기**로 맞춰 한 번 로드에 **수십~수백 키**를 검사 → I/O 횟수 \(\approx\) 트리 높이
- 노드 내부 검색은 **이진 탐색**(또는 분기 예측 최적화) 적용이 가능

### 차수(t) 선택 가이드

페이지 크기를 \(P\), 포인터 크기 \(p\) 바이트, 키 크기 \(k\) 바이트라고 할 때,
한 내부 노드는 대략 \((2t-1)\cdot k + (2t)\cdot p \le P\) 를 만족해야 한다.
여기에 **슬롯/메타데이터** 여유를 포함시켜 \(t\)를 잡는다.

---

## 탐색(Search)

루트에서 시작 → 노드 내부에서 **이진 탐색**으로 분기 결정 → 자식으로 재귀.
리프에 도달하거나 노드 내에서 키를 찾으면 종료. 복잡도 \(O(h \cdot \log(2t)) = O(\log n)\) (노드 내부 이진 탐색 포함).

---

## 삽입 알고리즘(정확한 절차)

핵심 아이디어: **항상 리프에 삽입**. 내려가다 **가득 찬(=2t-1키) 노드**를 만나면 **미리 분할**(split)해서 공간 확보.

1) **Top-down split**: 루트부터 내려가며, 자식이 가득 차 있으면 **먼저 분할**하여 내려간다.
2) 리프에서 적절 위치에 키를 삽입(정렬 유지).

### 분할(splitChild) 규칙

가득 찬 노드 \(Y\)의 **중간 키** \(Y.keys[t-1]\) 를 **부모로 올리고**, 나머지를 좌/우로 분배:

- 왼쪽 \(Y_L\): 처음 \(t-1\)개 키
- 오른쪽 \(Z\): 뒤 \(t-1\)개 키
- 자식 포인터도 좌/우로 나눔
- 부모에 중간 키 삽입 및 자식 포인터 갱신

---

## 삭제 알고리즘(정확한 절차)

탐색하면서 **부족(=키 수 < t-1) 상태가 생기지 않도록** **top-down**으로 처리한다.

케이스는 다음과 같다(현재 노드 \(X\), 삭제 대상 키 \(k\)):

1) **리프에 \(k\)** 가 있으면: **그 자리에서 삭제**(쉬움).

2) **내부 노드에 \(k\)** 가 있으면:
   - **왼쪽 자식 \(Y\)** 가 **\(t\)개 이상** 키 보유 → **선행자**(왼쪽 서브트리 최댓값)로 대체 후, \(Y\)에서 그 키 삭제
   - 그렇지 않고 **오른쪽 자식 \(Z\)** 가 **\(t\)개 이상** → **후속자**(오른쪽 서브트리 최솟값)로 대체 후, \(Z\)에서 삭제
   - 둘 다 **\(t-1\)** 개이면: **\(Y\), \(Z\), 그리고 경계 키**를 **병합(merge)** → 하나의 노드로 만든 뒤, 그 노드에서 삭제 계속

3) **내려가야 하는데 자식이 \(t-1\)** 개라면 **미리 보정**:
   - **좌/우 형제 중 하나가 \(t\)개 이상**이면 → **차용(borrow)**: 부모 경계 키와 형제의 키를 재배치
   - **양쪽 형제가 모두 \(t-1\)** 개이면 → **병합** 후 내려감

이 보정 덕분에 **삭제 도중 어떤 노드도 \(t-2\) 미만이 되지 않는다**(불변식 유지).

---

## C++ 구현(교육용, 정수 키·고정 차수)

아래 구현은 교육 목적으로 **정수 키**와 **고정 차수 T**(minimum degree)를 사용한다.
노드 내부 검색은 선형 이동을 보였던 초안에서 한 걸음 나아가, **간단 이진 탐색** 헬퍼를 추가한다.

```cpp
// B-Tree (minimum degree T) — Insert/Delete/Search with top-down corrections
#include <bits/stdc++.h>

using namespace std;

#ifndef T
#define T 3            // minimum degree (each node: [t-1 .. 2t-1] keys), t >= 2
#endif

struct BTreeNode {
    int keys[2*T-1];
    BTreeNode* ch[2*T];
    int n;            // number of keys used
    bool leaf;

    BTreeNode(bool isLeaf): n(0), leaf(isLeaf) {
        for(int i=0;i<2*T;i++) ch[i]=nullptr;
    }

    // binary search: first index i s.t. keys[i] >= k (in [0..n])
    int lower_bound_idx(int k) const {
        int lo=0, hi=n;
        while(lo<hi){
            int mid=(lo+hi)>>1;
            if(keys[mid] < k) lo = mid+1; else hi = mid;
        }
        return lo;
    }

    // Traverse (inorder-like)
    void traverse() const {
        for(int i=0;i<n;i++){
            if(!leaf) ch[i]->traverse();
            cout<<keys[i]<<" ";
        }
        if(!leaf) ch[n]->traverse();
    }

    // Search in subtree
    BTreeNode* search(int k){
        int i = lower_bound_idx(k);
        if(i<n && keys[i]==k) return this;
        if(leaf) return nullptr;
        return ch[i]->search(k);
    }

    // Core subroutines used by BTree wrapper:
    void insertNonFull(int k);
    void splitChild(int i, BTreeNode* y);
    void removeKey(int k);

    // deletion helpers
    int  findKey(int k){
        // first i with keys[i] >= k
        return lower_bound_idx(k);
    }
    void removeFromLeaf(int idx){
        for(int i=idx+1;i<n;i++) keys[i-1]=keys[i];
        n--;
    }
    void removeFromNonLeaf(int idx){
        int k = keys[idx];
        if(ch[idx]->n >= T){
            int pred = getPredecessor(idx);
            keys[idx] = pred;
            ch[idx]->removeKey(pred);
        } else if(ch[idx+1]->n >= T){
            int succ = getSuccessor(idx);
            keys[idx] = succ;
            ch[idx+1]->removeKey(succ);
        } else {
            merge(idx);
            ch[idx]->removeKey(k);
        }
    }
    int getPredecessor(int idx){
        BTreeNode* cur = ch[idx];
        while(!cur->leaf) cur = cur->ch[cur->n];
        return cur->keys[cur->n-1];
    }
    int getSuccessor(int idx){
        BTreeNode* cur = ch[idx+1];
        while(!cur->leaf) cur = cur->ch[0];
        return cur->keys[0];
    }
    void fill(int idx){
        if(idx!=0 && ch[idx-1]->n >= T)      borrowFromPrev(idx);
        else if(idx!=n && ch[idx+1]->n >= T) borrowFromNext(idx);
        else{
            if(idx!=n) merge(idx);
            else       merge(idx-1);
        }
    }
    void borrowFromPrev(int idx){
        BTreeNode* C = ch[idx];
        BTreeNode* S = ch[idx-1];
        // shift C to right
        for(int i=C->n-1;i>=0;i--) C->keys[i+1]=C->keys[i];
        if(!C->leaf){
            for(int i=C->n;i>=0;i--) C->ch[i+1]=C->ch[i];
        }
        // bring down separator
        C->keys[0] = keys[idx-1];
        if(!C->leaf) C->ch[0] = S->ch[S->n];
        keys[idx-1] = S->keys[S->n-1];
        C->n++;
        S->n--;
    }
    void borrowFromNext(int idx){
        BTreeNode* C = ch[idx];
        BTreeNode* S = ch[idx+1];
        // separator moves down to C
        C->keys[C->n] = keys[idx];
        if(!C->leaf) C->ch[C->n+1] = S->ch[0];
        keys[idx] = S->keys[0];
        // shift S to left
        for(int i=1;i<S->n;i++) S->keys[i-1]=S->keys[i];
        if(!S->leaf){
            for(int i=1;i<=S->n;i++) S->ch[i-1]=S->ch[i];
        }
        C->n++;
        S->n--;
    }
    void merge(int idx){
        // merge ch[idx], separator keys[idx], ch[idx+1] into left child
        BTreeNode* C = ch[idx];
        BTreeNode* S = ch[idx+1];
        C->keys[T-1] = keys[idx];
        for(int i=0;i<S->n;i++) C->keys[i+T] = S->keys[i];
        if(!C->leaf){
            for(int i=0;i<=S->n;i++) C->ch[i+T] = S->ch[i];
        }
        // shrink current node
        for(int i=idx+1;i<n;i++) keys[i-1]=keys[i];
        for(int i=idx+2;i<=n;i++) ch[i-1]=ch[i];
        C->n += S->n + 1;
        n--;
        delete S;
    }
};

struct BTree {
    BTreeNode* root = nullptr;

    ~BTree(){ clear(root); }

    void clear(BTreeNode* u){
        if(!u) return;
        for(int i=0;i<=u->n;i++) clear(u->ch[i]);
        delete u;
    }

    void traverse() const {
        if(root) root->traverse();
        cout<<"\n";
    }

    BTreeNode* search(int k){
        return root? root->search(k) : nullptr;
    }

    void insert(int k){
        if(!root){
            root = new BTreeNode(true);
            root->keys[0]=k; root->n=1;
            return;
        }
        if(root->n == 2*T-1){
            // grow height
            BTreeNode* s = new BTreeNode(false);
            s->ch[0] = root;
            s->splitChild(0, root);
            int i = (s->keys[0] < k)? 1:0;
            s->ch[i]->insertNonFull(k);
            root = s;
        } else {
            root->insertNonFull(k);
        }
    }

    void remove(int k){
        if(!root) return;
        root->removeKey(k);
        if(root->n == 0){
            BTreeNode* old = root;
            root = root->leaf? nullptr : root->ch[0];
            delete old;
        }
    }
};

// === Node methods (insertion) ===

void BTreeNode::splitChild(int i, BTreeNode* y){
    // split full child y at index i
    BTreeNode* z = new BTreeNode(y->leaf);
    z->n = T-1;
    // move last T-1 keys to z
    for(int j=0;j<T-1;j++) z->keys[j] = y->keys[j+T];
    // move children if internal
    if(!y->leaf){
        for(int j=0;j<T;j++) z->ch[j] = y->ch[j+T];
    }
    y->n = T-1;

    // make room in current node for new child and separator
    for(int j=n; j>=i+1; j--) ch[j+1]=ch[j];
    ch[i+1] = z;

    for(int j=n-1; j>=i; j--) keys[j+1]=keys[j];
    keys[i] = y->keys[T-1];
    n++;
}

void BTreeNode::insertNonFull(int k){
    int i = n-1;
    if(leaf){
        // shift and insert (keep sorted)
        while(i>=0 && keys[i] > k){ keys[i+1]=keys[i]; i--; }
        keys[i+1] = k; n++;
    }else{
        // descend to child
        int pos = lower_bound_idx(k);
        if(ch[pos]->n == 2*T-1){
            splitChild(pos, ch[pos]);
            if(keys[pos] < k) pos++;
        }
        ch[pos]->insertNonFull(k);
    }
}

// === Node methods (deletion) ===

void BTreeNode::removeKey(int k){
    int idx = findKey(k);

    // case: key found in this node
    if(idx<n && keys[idx]==k){
        if(leaf){
            removeFromLeaf(idx);
        }else{
            removeFromNonLeaf(idx);
        }
        return;
    }
    // key not found here
    if(leaf) return; // not present

    bool lastChild = (idx==n);
    if(ch[idx]->n < T) fill(idx);
    // after fill, structure may change: if lastChild and merged, go to idx-1
    if(lastChild && idx>n) ch[idx-1]->removeKey(k);
    else                   ch[idx]->removeKey(k);
}
```

### 간단 사용 예

```cpp
int main(){
    BTree tree;
    for(int k: {10,20,5,6,12,30,7,17}) tree.insert(k);

    cout<<"트리 순회: "; tree.traverse();

    tree.remove(6);
    cout<<"6 삭제 후: "; tree.traverse();

    tree.remove(13); // 없음
    tree.remove(7);
    cout<<"7 삭제 후: "; tree.traverse();

    tree.remove(4); // 없음
    tree.remove(2); // 없음
    tree.remove(16); // 없음

    // 더 많은 랜덤 테스트/퍼징을 권장
    return 0;
}
```

---

## 삽입/삭제 워크스루 (t=2, 최대 3키/노드)

### 삽입 예시

초기 빈 트리에 순서대로 삽입: 10, 20, 5, 6

1) [10], insert 20 → [10,20]
2) insert 5 → [5,10,20] (가득 참)
3) insert 6 시 **분할**:
   - [5,10,20] 분할 → 중간 10이 위로, 좌 [5], 우 [20]
   - 루트 새로 생성: [10] / 자식: [5], [20]
   - 6은 [5] 쪽으로 내려가 [5,6]

ASCII 개략:
```
      [10]
     /    \
  [5,6]  [20]
```

### 삭제 예시

현재 트리에서 10 삭제(내부 키):

- 왼쪽 자식 [5,6] 는 t=2 ⇒ 키 2개(≥t) → **선행자=6** 선택
- 루트의 10을 6으로 대체, 그리고 [5,6]에서 6 삭제 → [5]
- 결과:
```
      [6]
     /   \
   [5]  [20]
```

이후 리프에서 키 삭제는 그대로 제거하면 된다.

---

## 중복 키·값 저장 정책

- **중복 금지**: 탐색 중 발견 시 **삽입 무시** 또는 카운터 증가.
- **중복 허용**:
  - (권장) **키는 유니크**, 값(레코드 포인터) 목록을 **리프**에 연결
  - 또는 **(key, record_id)** 를 키로 간주하여 정렬 삽입
- 데이터베이스/파일시스템은 보통 **키는 정렬된 인덱스 키**, 값은 **레코드/페이지 포인터**

---

## 노드 내부 검색과 메모리 배치

- **이진 탐색**: 노드당 \(2t-1\) 키를 `lower_bound`로 로그 단계 비교
- **분기 예측**을 위해 **각 키를 64B 정렬 슬롯** 등에 정렬해 배치(엔지니어링 팁)
- 내부 노드에서 **키 배열과 자식 배열**을 **SoA**로 분리하면 프리페치에 유리할 수 있다.

---

## 범위 질의(range scan)

B-Tree 자체도 내부 노드에 값이 있으므로, **중위 순회**로 정렬 순서를 얻는다.
다만 **B+Tree**처럼 **모든 값이 리프**에 있고 **리프 간 연결 리스트**가 있지는 않으므로, 많은 시스템은 **범위 질의에 B+Tree**를 선호한다.

---

## 검증기(checker) — 불변식 점검

테스트에서 반드시 자동 검증을 수행하자.

```cpp
struct CheckRes { bool ok=true; string msg; };

// returns (minKey, maxKey, height)
tuple<int,int,int> checkNode(BTreeNode* u, CheckRes& r){
    if(!u){ r.ok=false; r.msg="null node"; return {0,0,-1}; }

    // 1) key order
    for(int i=1;i<u->n;i++){
        if(!(u->keys[i-1] < u->keys[i])){ r.ok=false; r.msg="keys not sorted"; }
    }
    // 2) key/child counts
    if(!u->leaf){
        for(int i=0;i<=u->n;i++) if(!u->ch[i]){ r.ok=false; r.msg="missing child"; }
    }

    int minK = u->keys[0], maxK = u->keys[u->n-1];
    int height = 0;

    if(!u->leaf){
        // children order constraints: ch[i].max < keys[i] < ch[i+1].min
        auto [lmin,lmax,lh] = checkNode(u->ch[0], r); minK = min(minK, lmin); height = lh;
        if(!(lmax < u->keys[0])){ r.ok=false; r.msg="left child max >= key0"; }

        for(int i=1;i<=u->n;i++){
            auto [cmin,cmax,chh] = checkNode(u->ch[i], r);
            if(chh!=height){ r.ok=false; r.msg="leaf levels differ"; }
            minK = min(minK, cmin); maxK = max(maxK, cmax);
            if(i<u->n){
                if(!(u->keys[i-1] < cmin)) { r.ok=false; r.msg="key not less than right child min"; }
                if(!(cmax < u->keys[i])) { r.ok=false; r.msg="right child max >= next key"; }
            }
        }
        height += 1;
    }
    return {minK, maxK, height};
}

CheckRes checkTree(BTree& T){
    CheckRes r;
    if(!T.root) return r;
    // root minimality can be relaxed when leaf
    checkNode(T.root, r);
    return r;
}
```

---

## 퍼징 아이디어

- **난수 삽입/삭제** 대량 수행 → `std::multiset<int>` 등 **참조 구현**과 결과 세트 비교
- 매 스텝 후 `checkTree`로 불변식 점검
- 작은 `T`(2~4)와 큰 `T`(페이지 크기 기반) 모두 실험

---

## B-Tree vs B+Tree vs B\*Tree

| 항목 | B-Tree | B+Tree | B\*Tree |
|---|---|---|---|
| 값 저장 | 내부·리프 모두 가능 | **리프에만 값**, 내부는 경계키 | B+ 변형(노드 이용률↑) |
| 리프 연결 | 일반적 아님 | **리프 간 링크로 순차 스캔 우수** | 동일 |
| 노드 이용률 | ≥ 50% | ≥ 50% | **≥ 66%**(형제 재분배 선호) |
| 범위 질의 | 중위 순회 | **매우 우수** | **매우 우수** |
| 구현 복잡 | 보통 | 보통 | 다소 높음 |
| DB/FS 채택 | 과거/특수 | **주류** | 일부 엔진/교과서 변형 |

대부분의 RDB/파일시스템 인덱스는 **B+Tree**를 쓴다(리프 값 저장 + 리프 연결 리스트).

---

## I/O·시간 복잡도 요약

- **시간**: 탐색/삽입/삭제 **\(O(\log n)\)**
- **I/O(블록 접근)**: 노드=페이지 가정 시 **트리 높이 \(\approx O(\log_{2t} n)\)** 만큼의 페이지 접근
- **노드 내부**: \(\log(2t)\) 차수의 이진 탐색(혹은 선형 탐색 + SIMD 최적화)

---

## 자주 하는 실수와 디버깅 팁

1. **splitChild 인덱스** 오프바이원, 키/자식 이동 범위 실수
2. **borrow/merge** 후 **부모 키와 자식 포인터** 재배치 누락
3. **root 수축**(키 0개가 되면 자식 1개로 루트 교체) 처리 누락
4. 삭제 시 **top-down 보정**을 잊고 내려갔다가 **t-2** 위반 발생
5. 내부 노드 삭제에서 선행자/후속자 선택 후 **그 서브트리로 재귀 삭제** 누락
6. **검증기**를 매 연산 후 돌리면 **빠르게 오류를 포착**할 수 있다.

---

## 수학 스냅샷(증명 스케치)

### 높이 상계

루트가 1키 이상, 모든 비루트가 \(\ge t-1\) 키, \(\ge t\) 자식을 갖는 B-Tree에서, 높이를 \(h\)라 하면:

- 깊이 0: 루트 ≥ 1키, 자식 ≥ 2 (루트가 내부일 때)
- 깊이 1부터: 각 내부 노드는 자식 수 ≥ \(t\)
- 리프까지 깊이 \(h\): 노드 수 ≥ \(2 \cdot t^{h-1}\) (대략)
- 키 수 하한 \(n \ge 2 t^{h} - 1\)
  \(\Rightarrow h \le \left\lfloor \log_t \frac{n+1}{2} \right\rfloor\)

즉, 항상 **로그 높이**를 보장하므로 모든 연산이 \(O(\log n)\).

---

## 실전 체크리스트

- 페이지 크기·키/포인터 크기 기반으로 **t 산정**
- 노드 내부는 **이진 탐색** 또는 **분기 최적화**(SIMD) 적용
- **WAL/Redo/Undo**(DBMS)나 **저널링**(FS)과 함께 쓰기 일관성 확보
- **B+Tree**가 범위 질의/스토리지 친화 측면에서 대체로 유리
- **테스트**: 무작위 삽입/삭제 + 검증기 + 참조 컨테이너 동치성 확인

---

## 요약

| 키워드 | 설명 |
|---|---|
| 불변식 | 각 노드 키 \([t-1..2t-1]\), 자식 \([t..2t]\), 리프 높이 동일 |
| 높이 | \(h \le \lfloor \log_t((n+1)/2)\rfloor\) |
| 삽입 | **top-down split**, 리프 삽입 |
| 삭제 | **top-down 보정**(차용/병합) + 리프/내부 케이스 처리 |
| I/O | 노드=페이지 → 탐색/갱신 I/O ≈ \(O(\log_{2t} n)\) |
| 변형 | **B+Tree**(리프 값, 리프 링크), **B\*Tree**(노드 이용률↑) |

위 구현은 교육용 기준으로 **핵심 아이디어와 불변식**을 모두 담았다.
실전 엔진에서는 **가변 키 길이**, **스플릿 정책**, **로그/버퍼 관리**, **동시성 제어**가 추가된다.
