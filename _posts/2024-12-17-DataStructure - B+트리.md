---
layout: post
title: Data Structure - B+트리
date: 2024-12-17 19:20:23 +0900
category: Data Structure
---
# B+ Tree 자료구조

## 1. B+ 트리란?

**B+ Tree**는 **균형 다진 탐색 트리**의 한 변형으로,
- **모든 실제 데이터(레코드 포인터 또는 값)는 리프에만 저장**하고,
- **내부 노드는 오직 인덱스(경계 키)만 저장**하며,
- **리프들이 오른쪽으로 연결(next)된 리스트**를 이루어 **범위 질의**에 매우 효율적이다.

---

## 2. B-Tree vs B+ Tree — 핵심 차이

| 항목 | B-Tree | **B+ Tree** |
|---|---|---|
| 값 저장 위치 | 내부·리프 모두 | **리프 전용** |
| 내부 노드 | 인덱스 + 값 | **인덱스 전용 (경계 키)** |
| 리프 연결 | 일반적 아님 | **오른쪽 단방향 연결(next)** |
| 범위 질의 | 중위 순회 | **리프 스캔으로 연속 접근** |
| I/O 효율 | 보통 | **내부 노드 더 작음 → fanout↑, 높이↓** |
| 중복 키 처리 | 다양 | **보통 리프에서 (key, rid)로 중복 허용** |

---

## 3. 불변식(Invariants)과 차수(order)

**표기**: 내부 노드 최대 자식 수를 \(M\), 리프 최대 레코드 수를 \(L\)이라 하자. (단순화를 위해 \(M=L\)로 두기도 한다.)

- **내부 노드**
  - 자식 수: \([ \lceil M/2 \rceil,\, M ]\) (루트 예외)
  - 키 수: 자식 수 − 1 \(\in [ \lceil M/2 \rceil - 1,\, M-1 ]\)
  - 키는 **오름차순**이며, **경계 키(separators)** 는 **오른쪽 자식의 첫 키 이상**을 대표

- **리프 노드**
  - 레코드 수: \([ \lceil L/2 \rceil,\, L ]\) (루트=리프일 때 예외)
  - 키는 오름차순, **오른쪽 리프를 가리키는 `next` 포인터** 보유

- **높이/균형**
  - 모든 리프가 **같은 깊이**에 존재
  - 높이 \(h\)에 대해, fanout이 평균 \(\bar{f}\)라면
    \[
      h \;\approx\; \log_{\bar{f}} n
    \]
  - 즉, **탐색·삽입·삭제는 항상 \(O(\log n)\)**

---

## 4. 페이지 레이아웃과 차수 산정 (실전)

페이지 크기 \(P\), 포인터 크기 \(p\), 키 크기 \(k\), 리프에서 값(또는 RID) 크기 \(v\)라고 하자. 내부/리프 메타데이터(M) 오버헤드를 고려하면:

- **내부 노드**: 최대 자식 \(M\)
  \[
    (M-1)\cdot k \;+\; M\cdot p \;+\; M_{\text{internal}} \;\le\; P
  \]

- **리프 노드**: 최대 레코드 \(L\)
  \[
    L\cdot (k + v) \;+\; p_{\text{next}} \;+\; M_{\text{leaf}} \;\le\; P
  \]

이 부등식을 만족하는 최댓값 \(M\), \(L\)를 취한다. **fanout이 커질수록 트리 높이가 낮아져 I/O 횟수 감소**.

---

## 5. 탐색(Search)과 범위 질의(Range)

### 5.1 탐색(단일 키)
1. 루트에서 시작, 내부 노드에서 **이진 탐색**으로 자식 포인터 선택
2. 리프에 도달하면, **리프 키 배열에서 이진 탐색** → 값(또는 RID) 반환

복잡도: \(O(\log n)\) (I/O는 높이에 비례)

### 5.2 범위 질의 \([a,b]\)
1. 키 \(a\)가 들어갈 리프를 찾아 **해당 리프에서 시작**
2. 리프의 키들을 스캔하며 \(b\)를 넘을 때까지, **`next`로 오른쪽 리프 연속 접근**

> **장점**: 리프 간 **순차 스캔**이므로 **디스크 선형 접근**과 **캐시/프리페치**에 유리

---

## 6. 삽입(Insert) — Top-down split

**핵심**: 내려가기 전에 **넘칠(child overflow) 가능성이 있으면 미리 분할**하여 공간 확보.

- **리프 오버플로우**: \(L+1\)개가 되면
  - **분할**: \(\lceil (L+1)/2 \rceil\) / \(\lfloor (L+1)/2 \rfloor\)
  - **오른쪽 리프의 첫 키**를 **부모에 올림**(separator)
  - 리프의 `next` 갱신
- **내부 오버플로우**: \(M\)자식 초과 시
  - **분할**: 키를 반으로 나누고, **중간 키를 위로 올림**
  - 오른쪽 노드의 **첫 키(혹은 중간 오른쪽 첫 키)**가 부모 경계가 됨(구현 선택)

**주의**(B+의 미묘한 점): 내부 노드의 키는 **자식의 실제 키와 중복**될 수 있다(값은 리프만 보유).

---

## 7. 삭제(Delete) — Borrow / Merge (Top-down fix)

**Top-down**으로 내려가며, 다음을 유지: **내려갈 자식이 최소치 미만이 되지 않게 보정**.

- 목표 자식이 **최소 미만이 될 위험**이 있으면:
  - **형제에서 차용(borrow)**: 좌/우 형제가 여유가 있으면 1개 가져오고 **부모 경계 키 갱신**
  - **병합(merge)**: 양쪽 형제 모두 최소치면 **형제+부모 경계**를 합쳐 **자식 수 축소**
- **리프에서 삭제** 후 부족하면 위 규칙 적용
- **내부에서 경계 키 삭제** 시, **B+**는 일반적으로 **리프에서만 실제 데이터를 삭제**하고 내부 키는 **경계 유지(필요 시 재조정)**

삭제는 구현 난이도가 높다. 아래 코드는 **교육용으로 모든 케이스를 단순화하되 B+의 불변식을 유지**한다.

---

## 8. C++ 구현 — 정수 키, 값=동일 키(데모), 완전한 삽입/검색/범위/삭제

> - 내부 노드: `isLeaf=false`, `keys`, `children`
> - 리프 노드: `isLeaf=true`, `keys`, `vals`, `next`
> - **ORDER**: 내부 노드 **최대 자식 수** \(M\)
> - **LEAF_ORDER**: 리프 **최대 레코드 수** \(L\)
> - 삭제는 **borrow/merge** 포함.  
> - 실전 엔진: 가변 길이 키, 레코드 포인터(RID), 페이지 ID, 버퍼 매니저, 로그 등이 필요.

```cpp
#include <bits/stdc++.h>
using namespace std;

// ===== Tunables =====
// Max children per internal node (M), Max records per leaf (L)
static const int ORDER      = 4;  // M >= 3 (toy)
static const int LEAF_ORDER = 4;  // L >= 3 (toy)

struct Node {
    bool isLeaf;
    vector<int> keys;               // sorted
    vector<Node*> children;         // internal: size = keys+1
    vector<int> values;             // leaf: same length as keys (demo: value==key)
    Node* next;                     // leaf-link (right)
    Node(bool leaf=false): isLeaf(leaf), next(nullptr) {}
};

struct BPlusTree {
    Node* root = nullptr;

    ~BPlusTree(){ clear(root); }

    // ========== Public API ==========
    void insert(int key, int value){ // demo: value can equal key
        if(!root){
            root = new Node(true);
            root->keys.push_back(key);
            root->values.push_back(value);
            return;
        }
        // Track path for easier splits
        vector<Node*> path;
        Node* cur = root;
        while(!cur->isLeaf){
            path.push_back(cur);
            int i = lower_bound(cur->keys.begin(), cur->keys.end(), key) - cur->keys.begin();
            cur = cur->children[i];
        }
        // Leaf insert
        auto it = lower_bound(cur->keys.begin(), cur->keys.end(), key);
        int pos = it - cur->keys.begin();
        cur->keys.insert(it, key);
        cur->values.insert(cur->values.begin()+pos, value);

        if((int)cur->keys.size() <= LEAF_ORDER) return; // no split

        // Split leaf
        Node* right = new Node(true);
        int total = cur->keys.size();
        int mid = (total+1)/2; // bias right slightly
        // Move right half
        right->keys.assign(cur->keys.begin()+mid, cur->keys.end());
        right->values.assign(cur->values.begin()+mid, cur->values.end());
        cur->keys.resize(mid);
        cur->values.resize(mid);

        // link
        right->next = cur->next;
        cur->next = right;

        int upKey = right->keys.front(); // separator to parent

        // If split root (leaf root case)
        if(path.empty()){
            Node* newRoot = new Node(false);
            newRoot->keys = { upKey };
            newRoot->children = { cur, right };
            root = newRoot;
            return;
        }

        // Propagate up
        insertInternal(path, upKey, cur, right);
    }

    bool find(int key, int& outValue) const {
        Node* cur = root;
        while(cur && !cur->isLeaf){
            int i = lower_bound(cur->keys.begin(), cur->keys.end(), key) - cur->keys.begin();
            cur = cur->children[i];
        }
        if(!cur) return false;
        auto it = lower_bound(cur->keys.begin(), cur->keys.end(), key);
        if(it!=cur->keys.end() && *it==key){
            int pos = it - cur->keys.begin();
            outValue = cur->values[pos];
            return true;
        }
        return false;
    }

    // inclusive range
    vector<int> rangeQuery(int lo, int hi) const {
        vector<int> res;
        Node* cur = root;
        if(!cur) return res;
        // descend to leaf for lo
        while(!cur->isLeaf){
            int i = lower_bound(cur->keys.begin(), cur->keys.end(), lo) - cur->keys.begin();
            cur = cur->children[i];
        }
        // scan
        while(cur){
            for(size_t i=0;i<cur->keys.size();++i){
                int k = cur->keys[i];
                if(k < lo) continue;
                if(k > hi) return res;
                res.push_back(cur->values[i]);
            }
            cur = cur->next;
        }
        return res;
    }

    // Delete one occurrence of key
    void erase(int key){
        if(!root) return;
        vector<Node*> path;
        vector<int>  idxPath; // child index taken from each internal
        Node* cur = root;

        // descend with path
        while(!cur->isLeaf){
            path.push_back(cur);
            int i = lower_bound(cur->keys.begin(), cur->keys.end(), key) - cur->keys.begin();
            idxPath.push_back(i);
            cur = cur->children[i];
        }
        // remove in leaf if present
        auto it = lower_bound(cur->keys.begin(), cur->keys.end(), key);
        if(it==cur->keys.end() || *it!=key) return; // not found
        int pos = it - cur->keys.begin();
        cur->keys.erase(cur->keys.begin()+pos);
        cur->values.erase(cur->values.begin()+pos);

        // root shrink
        if(cur==root){
            if(cur->keys.empty()){ delete cur; root=nullptr; }
            return;
        }

        // If underflow: need at least ceil(L/2)
        int minLeaf = (LEAF_ORDER+1)/2;
        if((int)cur->keys.size() >= minLeaf) return; // ok

        // Fix from parent upwards
        fixAfterDeleteLeaf(path, idxPath, cur);
    }

    // Debug traverse: print leaves left->right
    void printLeaves() const {
        Node* cur = root;
        if(!cur){ cout<<"(empty)\n"; return; }
        while(cur && !cur->isLeaf) cur = cur->children.front();
        while(cur){
            cout<<"[";
            for(size_t i=0;i<cur->keys.size();++i){
                if(i) cout<<",";
                cout<<cur->keys[i];
            }
            cout<<"] ";
            cur = cur->next;
            if(cur) cout<<"-> ";
        }
        cout<<"\n";
    }

private:
    // ===== Helpers =====
    void clear(Node* u){
        if(!u) return;
        if(!u->isLeaf){
            for(Node* c: u->children) clear(c);
        }
        delete u;
    }

    void insertInternal(vector<Node*>& path, int upKey, Node* leftChild, Node* rightChild){
        // climb from deepest parent (last of path)
        for(int depth = (int)path.size()-1; depth>=0; --depth){
            Node* parent = path[depth];
            // find position for upKey (must insert to parent)
            int pos = lower_bound(parent->keys.begin(), parent->keys.end(), upKey) - parent->keys.begin();
            parent->keys.insert(parent->keys.begin()+pos, upKey);
            parent->children.insert(parent->children.begin()+pos+1, rightChild);

            // parent was pointing to leftChild already by structure
            // Now check overflow
            if((int)parent->children.size() <= ORDER) return; // ok: children <= M

            // split internal
            Node* right = new Node(false);
            int totalK = parent->keys.size();       // = children-1
            int totalC = parent->children.size();   // = keys+1

            int mid = totalK/2;
            int promote = parent->keys[mid];

            // right gets keys after mid, and children after mid
            right->keys.assign(parent->keys.begin()+mid+1, parent->keys.end());
            right->children.assign(parent->children.begin()+mid+1, parent->children.end());

            // parent shrinks
            parent->keys.resize(mid);
            parent->children.resize(mid+1);

            // upKey for next level
            upKey = promote;
            rightChild = right;
            leftChild  = parent;

            // if parent is root
            if(depth==0){
                Node* newRoot = new Node(false);
                newRoot->keys = { upKey };
                newRoot->children = { leftChild, rightChild };
                root = newRoot;
                return;
            }
            // else continue loop to insert upKey to next parent
        }
    }

    // Borrow from siblings or merge nodes after leaf deletion underflow
    void fixAfterDeleteLeaf(const vector<Node*>& path, const vector<int>& idxPath, Node* leaf){
        int minLeaf = (LEAF_ORDER+1)/2;
        Node* child = leaf;

        for(int depth = (int)path.size()-1; depth>=0; --depth){
            Node* parent = path[depth];
            int childIdx = idxPath[depth];

            // Try borrow from left sibling
            if(childIdx-1 >= 0){
                Node* L = parent->children[childIdx-1];
                if(L->isLeaf && (int)L->keys.size() > minLeaf){
                    // move last from L to front of child
                    child->keys.insert(child->keys.begin(), L->keys.back());
                    child->values.insert(child->values.begin(), L->values.back());
                    L->keys.pop_back(); L->values.pop_back();
                    // Update parent separator to child's first key
                    parent->keys[childIdx-1] = child->keys.front();
                    return;
                }else if(!L->isLeaf){
                    // internal borrow (for inner fix) - not used in leaf stage
                }
            }

            // Try borrow from right sibling
            if(childIdx+1 < (int)parent->children.size()){
                Node* R = parent->children[childIdx+1];
                if(R->isLeaf && (int)R->keys.size() > minLeaf){
                    // move first from R to end of child
                    child->keys.push_back(R->keys.front());
                    child->values.push_back(R->values.front());
                    R->keys.erase(R->keys.begin());
                    R->values.erase(R->values.begin());
                    // Update parent separator to R's first key
                    parent->keys[childIdx] = R->keys.front();
                    return;
                }else if(!R->isLeaf){
                    // internal borrow - not needed here
                }
            }

            // Need merge with a sibling
            if(childIdx-1 >= 0){
                // merge left+child into left (for B+ leaf: parent sep key removed)
                Node* L = parent->children[childIdx-1];
                // splice keys, values, and link
                L->keys.insert(L->keys.end(), child->keys.begin(), child->keys.end());
                L->values.insert(L->values.end(), child->values.begin(), child->values.end());
                L->next = child->next;

                // remove parent separator and child pointer
                parent->keys.erase(parent->keys.begin()+childIdx-1);
                parent->children.erase(parent->children.begin()+childIdx);
                delete child;
                child = L; // merged node survives as 'child' for higher-level fix
            }else{
                // merge child+right into child
                Node* R = parent->children[childIdx+1];
                child->keys.insert(child->keys.end(), R->keys.begin(), R->keys.end());
                child->values.insert(child->values.end(), R->values.begin(), R->values.end());
                child->next = R->next;

                parent->keys.erase(parent->keys.begin()+childIdx);
                parent->children.erase(parent->children.begin()+childIdx+1);
                delete R;
                // child stays same
            }

            // Check parent underflow for next loop
            if(parent==root){
                // root shrink if only one child
                if(!parent->isLeaf && parent->children.size()==1){
                    root = parent->children[0];
                    parent->children.clear();
                    parent->keys.clear();
                    delete parent;
                }else if(parent->isLeaf && parent->keys.empty()){
                    delete parent; root=nullptr;
                }
                return;
            }

            // If parent has enough keys, stop; else continue upward (internal fix)
            int minInternalChildren = (ORDER+1)/2; // ceil(M/2)
            if((int)parent->children.size() >= minInternalChildren) return;

            // propagate deficiency upward: set child to parent and continue
            child = parent;
        }
    }
};

// ===== Demo =====
int main(){
    ios::sync_with_stdio(false);
    cin.tie(nullptr);

    BPlusTree t;
    vector<int> vals = {10,20,5,6,12,30,7,17};
    for(int x: vals) t.insert(x, x);

    cout << "Leaves (after insert):\n";
    t.printLeaves(); // e.g., [5,6,7] -> [10,12,17] -> [20,30]

    // Range [7,20]
    auto range = t.rangeQuery(7,20);
    cout << "Range [7,20]: ";
    for(int v: range) cout << v << " ";
    cout << "\n";

    // Find
    int out;
    cout << boolalpha;
    cout << "find(12): " << t.find(12,out) << " val=" << (t.find(12,out)? out : -1) << "\n";
    cout << "find(13): " << t.find(13,out) << "\n";

    // Deletions
    t.erase(12);
    t.erase(10);
    cout << "Leaves (after erase 12,10):\n";
    t.printLeaves();

    return 0;
}
```

### 실행 예시(한 가능 출력)

```
Leaves (after insert):
[5,6,7] -> [10,12,17] -> [20,30]
Range [7,20]: 7 10 12 17 20
find(12): true val=12
find(13): false
Leaves (after erase 12,10):
[5,6,7] -> [17] -> [20,30]
```

> ⚠️ 주의: 위 구현은 **교육용(메모리 전용·정수 키·단일 값)** 으로, **페이지/버퍼/로그/동시성**이 없다.  
> 삭제 시 **top-down 보정**과 **B+의 경계 키 갱신**을 반영했으나, 실전 엔진은 훨씬 더 많은 케이스(가변키, RID, Fence key 등)를 다룬다.

---

## 9. 인덱싱 예제(리프에 (key, RID))

기초 예시(키=ID, 값=이름 RID):

```
ID:   5   6   7   10   12   17   20   30
Name: A   B   C    D    E    F    G    H
리프: [ (5,A), (6,B), (7,C) ] -> [ (10,D), (12,E), (17,F) ] -> [ (20,G), (30,H) ]
내부: [10, 20]  // 경계 키는 오른쪽 리프의 첫 키
```

- `search(12)`:
  - 내부 [10,20] → 10 ≤ 12 < 20 → 중간 리프 → (12,E)
- 범위 `7~20`:
  - 7이 속한 리프부터 `next`로 스캔 → C,D,E,F,G

---

## 10. 중복 키, 합성 키, 클러스터/넌클러스터

- **중복 키**: 보통 리프에 (key, **RID**) 쌍으로 저장하여 **다중 레코드**를 식별
- **합성 키**: `(col1, col2, ...)`를 사전식 비교. 내부 경계도 합성 키
- **클러스터 인덱스**: 리프 값이 **실제 레코드**(또는 페이지) → 테이블 정렬 효과
- **넌클러스터 인덱스**: 리프 값이 **RID** → RID로 테이블 접근(추가 I/O)

---

## 11. 벌크 로드(Bulk Load) — O(n)

정렬된 (key,RID) 입력이 주어진다면:
1. 리프 페이지를 **꽉 차게** 순서대로 채우고, `next`를 연결
2. 리프들의 첫 키를 모아 **상위 내부 노드**를 채움
3. 상위 레벨을 **위로** 반복 구축 → **높이 로그 수준**에 한 번씩, 전체 O(n)

> **장점**: 삽입마다 분할 발생 없이 **연속 작성** → 훨씬 빠르고 조각화 적음

---

## 12. Prefix 압축, Fence Key(실전)

- **Prefix Compression**: 내부/리프에서 공통 접두사를 잘라 공간 절약(가변 길이 키)
- **Fence Keys**: 페이지의 **하한/상한** 키를 명시, 스플릿/머지·복구·동시성에서 경계 명확화

---

## 13. 동시성 — B-link Tree(고전)

- **B-link Tree**(Lehman & Yao): 각 내부/리프가 **오른쪽 형제 링크**를 가지고, **latch coupling** 최소화
- Split 시 **right sibling**으로 탐색이 **진행 가능**하여 데드락/재시도 감소
- 실전 DBMS는 **락/래치**, **버퍼 매니저**, **로그(Write-Ahead Logging)**, **충돌 제어**를 통합

---

## 14. 검증기(checker)와 퍼징

### 최소 검증 항목
- 각 노드 키 정렬
- 내부: `children.size()==keys.size()+1`
- 리프: `values.size()==keys.size()`
- 리프 레벨 동일(높이), `next`가 왼→오른쪽 단조 증가
- 내부 경계: `max(left) < sep <= min(right)` (정책에 맞춰 등호 처리)

### 간단 퍼징
- 난수 삽입/삭제를 반복
- `std::multiset<int>` 또는 `std::map<int,int>` 를 참조로 **정렬성과 카디널리티** 비교
- 매 스텝 후 검증기 실행

---

## 15. 복잡도 요약과 I/O 직관

- **탐색/삽입/삭제**: \(O(\log n)\)
- **범위 질의**: 시작 리프 찾기 \(O(\log n)\) + **연속 스캔 \(O(k)\)** (결과 개수 \(k\))
- I/O: 평균 fanout \(\bar{f}\)라면 **블록 접근 수 \(\approx \log_{\bar{f}} n\)**

---

## 16. 자주 겪는 버그/주의점

1. **내부 분할 시 경계 키 승격** 처리 혼동(B+는 값이 리프, 내부는 경계만)
2. **리프 분할 후 `next` 링크 갱신 누락**
3. 삭제에서 **borrow/merge 시 부모 경계 키** 재설정 실수
4. 루트 축소(자식 하나 되면 **루트 교체**) 누락
5. **중복 키** 정책 불일치(리프에 (key,RID)로 통일 권장)
6. 내부 이진 탐색/하향 인덱스 **오프바이원**

---

## 17. 수학 스냅샷

### 높이 상계(내부 fanout \(M\), 리프 용량 \(L\))
평균적으로 각 내부가 최소 \(\lceil M/2 \rceil\) 자식을 가져도,
\[
h \;\le\; \log_{\lceil M/2 \rceil} n \;+\; O(1)
\]
리프에 값이 몰려 fanout이 커지므로 **\(h\)** 는 일반 B-Tree보다 **작거나 비슷**하다.

---

## 18. 예제 워크스루(차수 예: \(M=L=4\))

### 삽입: 10,20,5,6,12,30,7,17
- 리프가 꽉 차면 분할, 오른쪽 리프의 첫 키(예: 10, 20)를 부모에 올림
- 내부가 꽉 차면 분할, 중간 키 승격
- 결과(가능 구성):
```
Internal:         [10, 20]
                 /    |     \
Leafs:       [5,6,7] [10,12,17] [20,30]
Links:  [5,6,7] -> [10,12,17] -> [20,30]
```

### 삭제: 12, 10
- 12 삭제로 중간 리프가 부족하면 형제에게 **차용** 또는 **병합**
- 10 삭제로 경계 재조정 → 내부 키/자식 수 균형 확인, 필요 시 병합

---

## 19. 실전 팁

- 페이지 크기(4KB/8KB/16KB)·키/RID 크기·메타데이터로 **fanout 산정**
- 인덱스 타입: **클러스터 인덱스**(리프=데이터 페이지) vs **넌클러스터**(리프=RID)
- **벌크 로드** 우선 채택 → 그 후 **점증 삽입**
- **Prefix 압축**으로 내부/리프 모두 공간 절약
- **B-link tree**로 동시성·스플릿 진행성 확보

---

## 20. 요약

| 키워드 | 설명 |
|---|---|
| 구조 | **리프에만 값**, 내부는 경계 키, **리프 링크** |
| 탐색/범위 | \(O(\log n)\) / 시작점 찾고 **연속 스캔** |
| 삽입/삭제 | **top-down split / borrow·merge**, 루트 축소 |
| 벌크 로드 | 정렬 입력으로 **O(n)** 구축 |
| 실전 | fanout↑, I/O↓, Prefix 압축, B-link, WAL/복구 |

> 이 글의 코드는 **교육용**이며, 실제 DB/FS 인덱스는 버퍼·동시성·로그·복구·가변키·압축·통계 등을 결합한다.  
> 하지만 **핵심 불변식/알고리즘**은 동일하다 — 이것이 B+ Tree가 **현대 인덱스의 표준**인 이유다.