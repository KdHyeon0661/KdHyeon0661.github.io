---
layout: post
title: Data Structure - 펜윅 트리
date: 2024-12-21 19:20:23 +0900
category: Data Structure
---
# 고성능 문자열 자료구조: Adaptive Radix Tree(ART)와 HAT-Trie

## 왜 ART와 HAT-Trie인가

일반적인 해시 테이블은 평균 \(O(1)\) 탐색을 제공하지만 **사전식 순회**, **접두사 검색**, **범위 질의**가 어렵다. 전통적 Trie/압축 Trie(radix/patricia)는 이를 잘 지원하지만, **자식 포인터 낭비**와 **캐시 비효율**이 성능을 깎는다.
**ART**와 **HAT-Trie**는 다음 목표로 설계된 고성능 문자열 인덱스다.

- **접두사/사전식 순회/범위 질의**가 자연스럽다.
- **CPU 캐시 친화** 설계(연속 메모리·작은 노드·분기 예측 개선).
- **공간 적응**(키 분포에 따라 노드 레이아웃이 자동 확장/축소).

요약 비교:

| 항목 | ART | HAT-Trie |
|---|---|---|
| 핵심 아이디어 | 압축 Trie + **적응형 노드 크기**(4/16/48/256) | Trie 상단 + **Bucket(해시/배열)** 하단, **버킷 스플릿** |
| 순회/범위 질의 | 매우 용이(바이트 사전 순) | 용이(버킷 정렬/체인) |
| 메모리 효율 | 우수(노드 크기 적응) | 우수(버킷으로 서픽스 집약) |
| 구현 난이도 | 높음(노드 업/다운그레이드, partial 관리) | 높음(버킷 분할 정책/정렬 유지) |
| 적합 사례 | DB 인덱스, 키-값 저장, 접두사 검색 | 대량 문자열 사전, 오토컴플리트, 로그 색인 |

---

## ART(Adaptive Radix Tree) 자세히 보기

### 핵심 구조

- **압축 경로(partial prefix)**: 연속 단일 경로는 문자열 덩어리로 압축. 내부 노드가 `partial`(최대 몇 바이트)와 그 길이를 가진다.
- **적응형 자식 표현**: 자식 수에 따라 4/16/48/256 형식을 **동적으로 승급/강등**.
  - **Node4**: 키 4개와 자식 4개 배열(선형 탐색).
  - **Node16**: 키 16개 배열(정렬 유지) + 자식 16개(바이트 비교를 16바이트 일괄 비교로 최적화 가능).
  - **Node48**: 인덱스 테이블 256바이트(값은 0..47 또는 -1) + 자식 48개.
  - **Node256**: 자식 256개 포인터 테이블(바이트 직접 인덱싱).
- **리프**: 실제 key(원문 바이트열)와 payload를 보관(접두사/완전일치 판별용).

주의: **바이트 단위 Trie**이므로 UTF-8 다바이트 문자는 **그냥 바이트열**로 처리한다.

### 탐색 알고리즘(요약)

1. 현재 노드의 **partial**을 키의 현재 위치와 비교(불일치 시 실패).
2. 다음 분기 바이트 `c = key[pos]`로 자식 검색(Node4/16/48/256 방식).
3. 리프 도달 시 전체 키 비교로 최종 확인.

### 삽입 알고리즘(요약)

1. 내려가다 **partial 불일치**가 생기면 내부 노드를 **분할**:
   - 공통 접두사까지를 상위 노드로 남기고, 나머지를 두 갈래(기존 경로/신규 경로)로 분기.
2. 자식 포인터가 꽉 차면 **승급**(4→16→48→256).
3. 리프 생성(키 원문 저장) 또는 기존 리프 중복 키 정책 처리.

### 삭제 알고리즘(요약)

- 리프 제거 후 부모 노드 자식 수가 임계치보다 줄면 **강등**(256→48→16→4).
- 강등 후 **연속 단일 경로**가 되면 **partial 병합**(압축 유지).

### 시간·공간 직관

- 바이트 길이 \(k\) 에 대해 탐색/삽입/삭제 **\(O(k)\)**.
- 노드 수준에서 분기 선택이 **상수 시간**에 가깝게 최적화(Node48/256).

메모리(대략):
- Node4/16은 **작은 배열 + 정렬된 키 바이트**.
- Node48은 **256 인덱스 배열 + 48 자식 슬롯**으로 **희소/밀집 절충**.
- Node256은 **직접 인덱싱**으로 분기비용 최소화.

---

## ART-Lite C++ 레퍼런스 구현

아래는 학습·프로토타입용 **간단화된 ART**다(핵심 아이디어를 담았고, 일부 최적화/에러처리는 생략).
키는 `std::string`(임의 바이트), 값은 `int` payload로 예시한다.

```cpp
#include <bits/stdc++.h>

using namespace std;

// ---------- 공통 ----------
enum class Kind : uint8_t { Node4, Node16, Node48, Node256, Leaf };

struct Leaf {
    string key;
    int value;
    Leaf(string k, int v): key(move(k)), value(v) {}
};

struct Node {
    Kind kind;
    uint8_t partial_len;            // 저장된 압축 경로 길이
    array<uint8_t, 8> partial{};    // 데모: 최대 8바이트만 저장(더 길면 압축 일부만 저장하고 나머지는 런타임 비교)
    Node(Kind k): kind(k), partial_len(0) {}
    virtual ~Node() = default;
};

struct Node4  : Node { array<uint8_t,4> keys{}; array<Node*,4> child{}; uint8_t num{0}; Node4():Node(Kind::Node4){ child.fill(nullptr);} };
struct Node16 : Node { array<uint8_t,16> keys{}; array<Node*,16> child{}; uint8_t num{0}; Node16():Node(Kind::Node16){ child.fill(nullptr);} };
struct Node48 : Node { array<int8_t,256> index{}; array<Node*,48> child{}; uint8_t num{0}; Node48():Node(Kind::Node48){ index.fill(-1); child.fill(nullptr);} };
struct Node256: Node { array<Node*,256> child{}; Node256():Node(Kind::Node256){ child.fill(nullptr);} };

struct ART {
    Node* root = nullptr;

    // 유틸
    static bool matchPartial(Node* n, const string& key, size_t& depth){
        size_t i=0, maxp = n->partial_len;
        for(; i<maxp; ++i){
            if(depth+i >= key.size()) return false;
            if(n->partial[i] != (uint8_t)key[depth+i]) return false;
        }
        depth += i;
        return true;
    }

    static Leaf* asLeaf(Node* n){ return (Leaf*)n; }
    static bool isLeaf(Node* n){ return n && n->kind==Kind::Leaf; }

    // 리프 생성
    static Node* makeLeaf(const string& k, int v){ return (Node*)new Leaf(k,v); }

    // Node4 helpers
    static int findPos(Node4* n, uint8_t c){
        for(int i=0;i<n->num;i++) if(n->keys[i]==c) return i;
        return -1;
    }
    static void insertChild(Node4* n, uint8_t c, Node* ch){
        int i=n->num;
        // 키 정렬 유지(단순 삽입정렬)
        while(i>0 && n->keys[i-1]>c){
            n->keys[i]=n->keys[i-1];
            n->child[i]=n->child[i-1];
            --i;
        }
        n->keys[i]=c; n->child[i]=ch; n->num++;
    }

    // 승급: 4 -> 16
    static Node16* grow4to16(Node4* n4){
        auto* n16 = new Node16();
        n16->partial_len = n4->partial_len; n16->partial = n4->partial;
        for(int i=0;i<n4->num;i++){ n16->keys[i]=n4->keys[i]; n16->child[i]=n4->child[i]; }
        n16->num = n4->num;
        delete n4;
        return n16;
    }

    // 16 -> 48
    static Node48* grow16to48(Node16* n16){
        auto* n48=new Node48(); n48->partial_len=n16->partial_len; n48->partial=n16->partial;
        for(int i=0;i<n16->num;i++){
            uint8_t k=n16->keys[i]; n48->index[k]=i; n48->child[i]=n16->child[i];
        }
        n48->num=n16->num; delete n16; return n48;
    }

    // 48 -> 256
    static Node256* grow48to256(Node48* n48){
        auto* n256=new Node256(); n256->partial_len=n48->partial_len; n256->partial=n48->partial;
        for(int c=0;c<256;c++){ int8_t idx=n48->index[c]; if(idx>=0) n256->child[c]=n48->child[idx]; }
        delete n48; return n256;
    }

    // Node16 helpers
    static int findPos(Node16* n, uint8_t c){
        for(int i=0;i<n->num;i++) if(n->keys[i]==c) return i; // 데모: 선형. 실제는 SIMD 비교 최적화 가능.
        return -1;
    }
    static void insertChild(Node16* n, uint8_t c, Node* ch){
        int i=n->num;
        while(i>0 && n->keys[i-1]>c){ n->keys[i]=n->keys[i-1]; n->child[i]=n->child[i-1]; --i; }
        n->keys[i]=c; n->child[i]=ch; n->num++;
    }

    // Node48 helpers
    static Node* getChild(Node48* n, uint8_t c){
        int8_t idx=n->index[c]; return idx>=0? n->child[idx]: nullptr;
    }
    static void insertChild(Node48* n, uint8_t c, Node* ch){
        n->child[n->num]=ch; n->index[c]=n->num; n->num++;
    }

    // 삽입(상위 API)
    void insert(const string& key, int value){
        root = insertRec(root, key, 0, value);
    }

    // 검색(정확 일치) — payload 반환 포인터 또는 nullptr
    const int* find(const string& key){
        Node* n=root; size_t depth=0;
        while(n){
            if(isLeaf(n)){
                Leaf* lf=asLeaf(n);
                return (lf->key==key)? &lf->value : nullptr;
            }
            if(!matchPartial(n,key,depth)) return nullptr;
            uint8_t c = depth<key.size()? (uint8_t)key[depth] : 0; // 키 끝은 0 sentinel처럼 다루기도 함(옵션)
            switch(n->kind){
                case Kind::Node4:{
                    auto* x=(Node4*)n; int pos=findPos(x,c);
                    if(pos<0) return nullptr; n=x->child[pos]; depth++; break;
                }
                case Kind::Node16:{
                    auto* x=(Node16*)n; int pos=findPos(x,c);
                    if(pos<0) return nullptr; n=x->child[pos]; depth++; break;
                }
                case Kind::Node48:{
                    auto* x=(Node48*)n; Node* ch=getChild(x,c);
                    if(!ch) return nullptr; n=ch; depth++; break;
                }
                case Kind::Node256:{
                    auto* x=(Node256*)n; Node* ch=x->child[c];
                    if(!ch) return nullptr; n=ch; depth++; break;
                }
                default: return nullptr;
            }
        }
        return nullptr;
    }

    // 접두사 순회: prefix로 시작하는 모든 (key,value)에 대해 콜백
    template<class F>
    void forEachWithPrefix(const string& prefix, F visit){
        // prefix 끝까지 내려간 후 서브트리 DFS
        Node* n=root; size_t depth=0;
        // 내려가기
        while(n){
            if(isLeaf(n)){
                Leaf* lf=asLeaf(n);
                if(lf->key.rfind(prefix,0)==0) visit(lf->key, lf->value);
                return;
            }
            // partial 비교: prefix가 더 짧은 경우도 허용
            size_t i=0, maxp=((Node*)n)->partial_len;
            for(; i<maxp && depth+i<prefix.size(); ++i){
                if(((Node*)n)->partial[i] != (uint8_t)prefix[depth+i]) return;
            }
            depth += min(i, maxp);
            if(depth==prefix.size()){ // 이 노드 이하 전부 방문
                dfsAll(n, visit);
                return;
            }
            // 아직 내려가야 함
            uint8_t c = (uint8_t)prefix[depth];
            Node* ch=nullptr;
            switch(n->kind){
                case Kind::Node4:{
                    auto* x=(Node4*)n; int pos=findPos(x,c);
                    if(pos<0) return; ch=x->child[pos]; break;
                }
                case Kind::Node16:{
                    auto* x=(Node16*)n; int pos=findPos(x,c);
                    if(pos<0) return; ch=x->child[pos]; break;
                }
                case Kind::Node48:{
                    auto* x=(Node48*)n; ch=getChild(x,c); if(!ch) return; break;
                }
                case Kind::Node256:{
                    auto* x=(Node256*)n; ch=x->child[c]; if(!ch) return; break;
                }
                default: return;
            }
            n=ch; depth++;
        }
    }

private:
    template<class F>
    static void dfsAll(Node* n, F visit){
        if(!n) return;
        if(isLeaf(n)){ auto* lf=asLeaf(n); visit(lf->key, lf->value); return; }
        switch(n->kind){
            case Kind::Node4:{
                auto* x=(Node4*)n;
                for(int i=0;i<x->num;i++) dfsAll(x->child[i], visit);
                break;
            }
            case Kind::Node16:{
                auto* x=(Node16*)n;
                for(int i=0;i<x->num;i++) dfsAll(x->child[i], visit);
                break;
            }
            case Kind::Node48:{
                auto* x=(Node48*)n;
                for(int i=0;i<256;i++){ int8_t idx=x->index[i]; if(idx>=0) dfsAll(x->child[idx], visit); }
                break;
            }
            case Kind::Node256:{
                auto* x=(Node256*)n;
                for(int i=0;i<256;i++) dfsAll(x->child[i], visit);
                break;
            }
            default: break;
        }
    }

    // 내부: 자식 조회/삽입 도우미(간소화)
    static Node*& childRef(Node* n, uint8_t c){
        if(n->kind==Kind::Node4)   { auto* x=(Node4*)n; int pos=findPos(x,c); if(pos<0) throw; return x->child[pos]; }
        if(n->kind==Kind::Node16)  { auto* x=(Node16*)n; int pos=findPos(x,c); if(pos<0) throw; return x->child[pos]; }
        if(n->kind==Kind::Node48)  { auto* x=(Node48*)n; int8_t idx=((Node48*)n)->index[c]; if(idx<0) throw; return x->child[idx]; }
        auto* x=(Node256*)n; return x->child[c];
    }

    static Node* addChild(Node* n, uint8_t c, Node* ch){
        switch(n->kind){
            case Kind::Node4:{
                auto* x=(Node4*)n;
                if(x->num<4){ insertChild(x,c,ch); return x; }
                // 승급
                auto* g=grow4to16(x); insertChild(g,c,ch); return g;
            }
            case Kind::Node16:{
                auto* x=(Node16*)n;
                if(x->num<16){ insertChild(x,c,ch); return x; }
                auto* g=grow16to48(x); insertChild(g,c,ch); return g;
            }
            case Kind::Node48:{
                auto* x=(Node48*)n;
                if(x->num<48){ insertChild(x,c,ch); return x; }
                auto* g=grow48to256(x); g->child[c]=ch; return g;
            }
            case Kind::Node256:{
                auto* x=(Node256*)n; x->child[c]=ch; return x;
            }
            default: return n;
        }
    }

    // 분할: 현재 노드 n의 partial과 key[depth..]가 diverge할 때
    static Node* splitNode(Node* n, const string& key, size_t depth, uint8_t mismatch_at){
        // 기존 노드의 partial = P, 공통 접두사 길이 = i
        // 상위에 새 내부노드(공통) 만들고, 기존 n은 partial를 줄여 하위로 내리고, 새 분기(키) 추가
        auto* parent = new Node4();
        // 공통 부분 저장
        parent->partial_len = mismatch_at;
        for(size_t i=0;i<mismatch_at && i<parent->partial.size(); ++i) parent->partial[i]=n->partial[i];

        // 기존 노드 n의 partial를 잘라서 하위로 이동
        size_t old_len = n->partial_len;
        size_t remain = old_len - (size_t)mismatch_at;
        // n.partial = n.partial[mismatch_at ..]
        array<uint8_t,8> newp{};
        size_t copyLen = min(remain, newp.size());
        for(size_t i=0;i<copyLen;i++) newp[i]=n->partial[mismatch_at+i];
        n->partial = newp;
        n->partial_len = (uint8_t)copyLen;

        // 기존 분기 바이트
        uint8_t old_c = (n->partial_len>0)? n->partial[0] : 0;
        // 새 키의 분기 바이트
        uint8_t new_c = (depth+mismatch_at<key.size())? (uint8_t)key[depth+mismatch_at] : 0;

        // parent에 기존 n 연결
        insertChild((Node4*)parent, old_c, n);

        // parent에 새 리프 연결
        Node* lf = makeLeaf(key, 0); // 데모: value는 0, 실제는 적절히
        parent = addChild(parent, new_c, lf);
        return parent;
    }

    Node* insertRec(Node* n, const string& key, size_t depth, int value){
        if(!n) return makeLeaf(key, value);

        if(isLeaf(n)){
            Leaf* lf = asLeaf(n);
            if(lf->key==key){ lf->value = value; return n; }
            // 리프와 새 키의 공통 접두사까지 내부 노드 생성
            // 내부 노드 만들고 두 리프를 분기
            auto* pn = new Node4();
            // partial = 공통 접두사 최대 8바이트 저장
            size_t i=0;
            while(i<lf->key.size() && i<key.size() && i< pn->partial.size() && lf->key[depth+i]==key[depth+i]) i++;
            pn->partial_len = (uint8_t)i;
            for(size_t j=0;j<i;j++) pn->partial[j]=(uint8_t)key[depth+j];

            uint8_t c1 = (depth+i<lf->key.size())? (uint8_t)lf->key[depth+i] : 0;
            uint8_t c2 = (depth+i<key.size())? (uint8_t)key[depth+i] : 0;

            insertChild(pn, c1, n);
            insertChild(pn, c2, makeLeaf(key,value));
            return pn;
        }

        // 내부 노드: partial 비교
        size_t i=0, maxp=n->partial_len;
        while(i<maxp && depth+i<key.size() && n->partial[i]==(uint8_t)key[depth+i]) i++;
        if(i<maxp){
            // partial 중간에서 갈라짐 -> 분할
            return splitNode(n, key, depth, (uint8_t)i);
        }
        depth += i;
        uint8_t c = (depth<key.size())? (uint8_t)key[depth] : 0;

        // 자식 찾기 or 추가
        Node* child=nullptr;
        switch(n->kind){
            case Kind::Node4:{
                auto* x=(Node4*)n; int pos=findPos(x,c);
                if(pos<0){ n = addChild(n,c, makeLeaf(key,value)); return n; }
                child = x->child[pos]; break;
            }
            case Kind::Node16:{
                auto* x=(Node16*)n; int pos=findPos(x,c);
                if(pos<0){ n = addChild(n,c, makeLeaf(key,value)); return n; }
                child = x->child[pos]; break;
            }
            case Kind::Node48:{
                auto* x=(Node48*)n; int8_t idx=x->index[c];
                if(idx<0){ n = addChild(n,c, makeLeaf(key,value)); return n; }
                child = x->child[idx]; break;
            }
            case Kind::Node256:{
                auto* x=(Node256*)n; if(!x->child[c]){ x->child[c]=makeLeaf(key,value); return n; }
                child = x->child[c]; break;
            }
            default: break;
        }
        Node* newChild = insertRec(child, key, depth+1, value);
        // 자식 포인터가 바뀌었을 수 있으니 다시 꽂기(간소화)
        switch(n->kind){
            case Kind::Node4:{ auto* x=(Node4*)n; int pos=findPos(x,c); x->child[pos]=newChild; break; }
            case Kind::Node16:{ auto* x=(Node16*)n; int pos=findPos(x,c); x->child[pos]=newChild; break; }
            case Kind::Node48:{ auto* x=(Node48*)n; int8_t idx=x->index[c]; x->child[idx]=newChild; break; }
            case Kind::Node256:{ auto* x=(Node256*)n; x->child[c]=newChild; break; }
            default: break;
        }
        return n;
    }
};

// 데모
int main(){
    ART art;
    art.insert("apple", 1);
    art.insert("applet", 2);
    art.insert("apricot", 3);
    art.insert("banana", 4);
    art.insert("band", 5);

    if(auto v=art.find("apricot")) cout<<"apricot="<<*v<<"\n";
    if(auto v=art.find("apple"))   cout<<"apple="<<*v<<"\n";

    cout<<"prefix ap: ";
    art.forEachWithPrefix("ap", [](const string& k, int v){ cout<<k<<"("<<v<<") "; });
    cout<<"\n";
    return 0;
}
```

핵심 구현 포인트만 담은 **ART-Lite**다. 실제 프로덕션에서는 다음을 추가/개선한다.

- partial 길이 확장(현재 8바이트 데모) 및 **장거리 비교 최적화**.
- Node16 키 비교 **SIMD**/비트트릭(예: `_mm_cmpeq_epi8`) 최적화.
- 삭제 + 강등(Node256→48→16→4) 로직.
- **값 타입 제네릭**, **메모리 풀/RCU/Epoch GC**, **스레드 안전**.

---

## HAT-Trie 자세히 보기

### 핵심 구조

- 상단은 **Trie 레벨**: 공통 prefix 단위로 분기.
- 하단은 **Bucket**: 동일 prefix 아래 **서픽스 집합**을 해시/배열로 보관.
  버킷이 커지면 **split**: 다음 바이트 기준으로 트라이 노드로 승격.

장점:
- 동일 prefix 하의 문자열을 **연속 메모리**에 모아 캐시 효율.
- **삽입/탐색 빠름**(버킷 내부는 해시/선형 스캔, 매우 짧음).
- **사전 순 순회**: 버킷을 정렬하거나 child 순서대로 방문하면 가능.

### 간소 HAT-Trie C++ 구현(학습용)

- 각 내부 노드는 `depth`(prefix 길이), `children[256]`.
- 버킷은 현재 노드 아래 **서픽스 전체 문자열**을 저장(여기선 `unordered_map<string,bool>` 데모).
- 버킷 크기 초과 시 **splitBucket**: 다음 바이트로 나눠 자식 생성.

```cpp
#include <bits/stdc++.h>

using namespace std;

struct HNode {
    bool isBucket=false;
    int depth=0; // 이 노드가 담당하는 prefix 길이
    array<HNode*,256> child{};
    unordered_map<string, bool> bucket; // suffix 전체 저장(데모)
    HNode(int d=0): depth(d){ child.fill(nullptr); }
    ~HNode(){ for(auto* c: child) delete c; }
};

struct HATTrie {
    HNode* root = new HNode(0);
    size_t BUCKET_MAX = 4; // 데모 임계

    ~HATTrie(){ delete root; }

    void insert(const string& s){
        HNode* n = root;
        while(true){
            if(n->isBucket){
                n->bucket[s] = true;
                if(n->bucket.size() > BUCKET_MAX) splitBucket(n);
                return;
            }
            if(n->depth == (int)s.size()){
                // 정확히 이 prefix에서 종료: 이 위치를 버킷으로 전환
                n->isBucket = true;
                n->bucket[s] = true;
                if(n->bucket.size()>BUCKET_MAX) splitBucket(n);
                return;
            }
            uint8_t c = (uint8_t)s[n->depth];
            if(!n->child[c]) n->child[c] = new HNode(n->depth+1);
            n = n->child[c];
        }
    }

    bool search(const string& s) const {
        const HNode* n = root;
        while(n){
            if(n->isBucket) return n->bucket.count(s)>0;
            if(n->depth == (int)s.size()) return false;
            uint8_t c = (uint8_t)s[n->depth];
            n = n->child[c];
        }
        return false;
    }

    // 접두사 순회
    template<class F>
    void forEachWithPrefix(const string& prefix, F visit) const {
        const HNode* n = root;
        // 내려가기
        while(n){
            if(n->isBucket){
                for(auto& kv : n->bucket) if(kv.first.rfind(prefix,0)==0) visit(kv.first);
                return;
            }
            if(n->depth==(int)prefix.size()){
                // 이 노드 아래 전부 방문
                dfs(n, visit, prefix);
                return;
            }
            if(n->depth>(int)prefix.size()) { dfs(n, visit, prefix); return; }
            uint8_t c = (uint8_t)prefix[n->depth];
            n = n->child[c];
            if(!n) return;
        }
    }

private:
    void splitBucket(HNode* n){
        // 버킷의 모든 문자열을 자식으로 재분배
        unordered_map<string,bool> tmp; tmp.swap(n->bucket);
        n->isBucket=false;
        for(auto& kv: tmp){
            const string& s = kv.first;
            if(n->depth==(int)s.size()){
                // 정확히 이 위치에서 끝나면 '종료'를 표현하려면 별도 마커 필요.
                // 데모: 빈문자(0)로 child[0]로 보냄.
                uint8_t c=0;
                if(!n->child[c]) n->child[c]=new HNode(n->depth+1);
                n->child[c]->isBucket=true;
                n->child[c]->bucket[s]=true;
            }else{
                uint8_t c = (uint8_t)s[n->depth];
                if(!n->child[c]) n->child[c] = new HNode(n->depth+1);
                HNode* ch = n->child[c];
                if(!ch->isBucket) ch->isBucket=true;
                ch->bucket[s]=true;
            }
        }
        // 2차 분할—자식 버킷이 임계 넘으면 재귀 분할
        for(auto* ch: n->child){
            if(ch && ch->isBucket && ch->bucket.size()>BUCKET_MAX) splitBucket(ch);
        }
    }

    template<class F>
    static void dfs(const HNode* n, F visit, const string& prefix){
        if(!n) return;
        if(n->isBucket){
            for(auto& kv : n->bucket)
                if(kv.first.rfind(prefix,0)==0) visit(kv.first);
            return;
        }
        for(int c=0;c<256;c++) dfs(n->child[c], visit, prefix);
    }
};

// 데모
int main(){
    HATTrie ht;
    for(auto&s: {"apple","apricot","apex","apply","banana","band","bandana","bar"})
        ht.insert(s);

    cout<<"search(apricot)="<<ht.search("apricot")<<"\n";
    cout<<"prefix ap: ";
    ht.forEachWithPrefix("ap", [](const string& s){ cout<<s<<" "; });
    cout<<"\n";
    return 0;
}
```

이 구현은 **개념 시연**에 초점을 맞춘 간소 버전이다. 실전에서는:

- 버킷을 **오픈 어드레싱 해시**나 **정렬 벡터**로 구현하여 **순서/캐시** 개선.
- **정확 종료 마커**를 별도 비트로 보관(위 데모의 child[0] 트릭 대신).
- 버킷 분할 정책: **부하율/분산 균형**, **스필오버** 등 튜닝.
- 사전 순 순회를 위해 버킷을 **정렬 상태 유지** 또는 **키 포인터 배열 정렬**.

---

## 순회·범위·자동완성 패턴

두 구조 모두 접두사 기반 질의가 자연스럽다.

- **자동완성**: `forEachWithPrefix(prefix, visit)` 로 즉시 후보 열거.
- **범위 질의** \([L, R]\):
  prefix 구간으로 변환하거나, lower_bound(L)에서 시작해 R 초과 시 중단.
- **사전 순 페이지네이션**: 이터레이터를 리프/버킷 체인으로 이어서 구현.

---

## 성능·메모리·수학적 메모

- ART 노드 적응은 **희소→밀집** 구간을 부드럽게 커버.
  평균 자식 수를 \(\bar{d}\) 라 할 때, 노드당 인덱싱 비용은 대략 상수에 가깝다.
- HAT-Trie 버킷 탐색 기대 비용은 부하율 \(\alpha\) 에 대해(선형 탐사 가정)
  $$ \mathbb{E}[\text{probe}] \approx \frac{1}{1-\alpha} $$
  \(\alpha\) 를 0.7~0.8로 유지하면 수 프로브 이내.

- 메모리 대략식(키 바이트 길이 \(k\), 노드 수 \(N\)):
  $$ \text{Mem} \approx \sum_{\text{nodes}} \text{node\_overhead} + \sum_{\text{leaves}} (k + \text{payload}) $$
  ART는 **자식 수가 적을수록 작은 노드**를 사용해 오버헤드를 낮춘다.
  HAT-Trie는 **버킷에 서픽스를 밀집**시켜 파편화를 줄인다.

---

## 실무 팁과 함정

1. **바이트 vs 유니코드**
   UTF-8은 바이트 Trie로 자연 처리되지만, **정규화**와 **대소문** 정책을 먼저 결정.
2. **널 바이트**
   바이트 0x00도 유효하게 취급. 종료 마커 전략은 구현별로 일관되게.
3. **삭제와 축소**
   ART는 강등을 잊지 말 것(공간 회수). HAT-Trie는 버킷 underflow 시 병합 고려.
4. **동시성**
   읽기 다수 환경은 **Optimistic Lock Coupling(OLC)**, **Epoch GC**가 실전에서 많이 쓰인다.
5. **직렬화/스냅샷**
   정해진 순서로 노드/버킷을 덤프(버전·엔디언·키코덱 명시).
6. **부정 탐색 가속**
   대용량 로그/텍스트에서 부정 조회 빈도가 높다면 앞단에 **Bloom Filter**를 얹어 낭비를 줄일 수 있다.
7. **캐시 최적화**
   - Node16 비교는 **SIMD**로 16바이트를 한 번에 비교.
   - 포인터보다 **오프셋 인덱스**를 쓰면 구조체 크기를 줄이고 locality가 좋아진다.

---

## 언제 무엇을 선택할까

- **접두사 검색/범위 질의**가 핵심이고, 키 분포가 다양 — **ART** 추천.
- **동일 prefix가 매우 많은 대량 문자열**(예: 로그 접두사, URL 도메인) — **HAT-Trie**가 버킷으로 매우 강하다.
- **복합 키**(예: userId|timestamp)에는 첫 필드 단위로 계층화하거나, 바이트 시퀀스로 그대로 넣어도 된다.

---

## 빠른 실험 시나리오

- 1천만 라인의 접두사 검색 워크로드:
  1) 키 로딩(ART/HAT-Trie/`unordered_set` 비교)
  2) 10만 개의 무작위 prefix로 자동완성 길이 20 제한
  3) 쿼리 QPS·p95 레이턴시·메모리 비교
- 기대: 해시는 단건 조회는 빠르나 **접두사/순회는 비효율**, ART/HAT-Trie는 **접두사·순회 압도**.

---

## 정리 표

| 키워드 | 요점 |
|---|---|
| ART | 압축 + 적응형 노드(4/16/48/256), 바이트 단위 분기, 접두사·범위·순회 강력 |
| HAT-Trie | 상단 Trie + 하단 버킷, 버킷 스플릿으로 균형, 캐시 친화 |
| 공통 | 사전 순 순회 자연, 문자열 인덱스의 정석급 대안 |
| 구현 팁 | 삭제/강등/스플릿, UTF-8·널 처리, 동시성/GC, 직렬화 |

---

## 부록 A. ART 노드 승급/강등 임계 요약

- 승급: 4→16(5번째), 16→48(17번째), 48→256(49번째 자식)
- 강등: 반대 방향. 예: Node48 자식 수 < 17이면 Node16로 내림(재배치 비용 고려).

---

## 부록 B. 사소하지만 중요한 테스트 체크리스트

- 빈 문자열 `""` 키(허용 여부 명시).
- 공통 접두사만 다른 두 키 삽입/삭제 시 partial 분할과 병합이 일관적인가.
- HAT-Trie 버킷 스플릿 후 **정확 종료 키**(prefix 자체가 키) 보존.
- 대량 삭제 후 메모리 회수/강등이 제대로 발생.
