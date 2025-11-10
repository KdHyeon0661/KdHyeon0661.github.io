---
layout: post
title: Data Structure - 트라이
date: 2024-12-16 20:20:23 +0900
category: Data Structure
---
# 고급 Trie 구조 — 이진 Trie(Bitwise) & 압축 Trie(Radix/Compressed)

## 0) 개요와 비교

| 구조 | 키 단위 | 주 용도 | 깊이 상한 | 장점 | 단점 |
|---|---|---|---|---|---|
| 일반 Trie | 문자 | 사전, 자동완성 | 문자열 길이 | 단순·예측 가능 | 희소 시 메모리 낭비 |
| **이진 Trie** | **비트(0/1)** | **정수 XOR 문제**, IP LPM | 고정(보통 32/64) | 비트 문제 최적 | 삭제/중복 처리 주의 |
| **압축(Radix)** | **문자열 덩어리** | 사전/검색/라우팅 | ≤ 키 길이 | 깊이 얕음, 메모리 절감 | 삽입 시 분할 로직 복잡 |

---

## 1) 이진 Trie (Bitwise Trie)

### 1.1 핵심 아이디어
- 키를 고정 길이 **비트열**(예: 32비트 `uint32_t`)로 보고, 매 레벨에서 **0/1**로 분기.
- 경로 카운트(`cnt`)를 두면 **중복 허용**·**삭제**·**빈 노드 정리**가 가능.

> 표기: 최상위 비트(MSB)에서 시작해 `i = MAX_BIT .. 0` 순으로 내려간다.

---

### 1.2 완전 구현 (삽입/삭제/탐색/최대·최소 XOR)

```cpp
#include <bits/stdc++.h>
using namespace std;

struct BNode {
    BNode* ch[2];
    int cnt; // 이 노드를 경유(또는 종단)하는 키 개수
    BNode() : ch{nullptr,nullptr}, cnt(0) {}
};

struct BinaryTrie {
    BNode* root;
    int MAX_BIT; // 예: 31(= 32bit), 또는 63(= 64bit)
    BinaryTrie(int max_bit = 31) : MAX_BIT(max_bit) { root = new BNode(); }

    ~BinaryTrie(){ clear(root); }
    void clear(BNode* n){
        if(!n) return;
        clear(n->ch[0]); clear(n->ch[1]);
        delete n;
    }

    void insert(uint64_t x){
        BNode* cur = root;
        for(int i = MAX_BIT; i >= 0; --i){
            int b = (x >> i) & 1;
            if(!cur->ch[b]) cur->ch[b] = new BNode();
            cur = cur->ch[b];
            cur->cnt++;
        }
    }

    bool contains(uint64_t x) const {
        const BNode* cur = root;
        for(int i = MAX_BIT; i >= 0; --i){
            int b = (x >> i) & 1;
            if(!cur->ch[b] || cur->ch[b]->cnt == 0) return false;
            cur = cur->ch[b];
        }
        return cur && cur->cnt > 0;
    }

    // 중복 허용 삭제: 경로 cnt 감소, 0이 되면 가지치기
    bool erase(uint64_t x){
        array<BNode*, 66> path{}; // MAX_BIT<=63 가정
        BNode* cur = root;
        for(int i = MAX_BIT; i >= 0; --i){
            int b = (x >> i) & 1;
            if(!cur->ch[b] || cur->ch[b]->cnt == 0) return false; // 없음
            path[i+1] = cur;
            cur = cur->ch[b];
        }
        // cur: leaf(종단)
        // 아래에서 위로 cnt 감소 & 필요시 delete
        for(int i = 0; i <= MAX_BIT; ++i){
            int bit = (x >> i) & 1;
            BNode* parent = path[i];
            BNode* node   = (i==0 ? root : path[i]); // path[i]는 i에서의 parent, path[i+1]가 child
        }
        // 정확히 감소/정리 수행
        cur = root;
        for(int i = MAX_BIT; i >= 0; --i){
            int b = (x >> i) & 1;
            BNode* nxt = cur->ch[b];
            if(--nxt->cnt == 0){
                // 하위 모두 비었으면 가지치기
                deleteSubtree(nxt);
                cur->ch[b] = nullptr;
                return true;
            }
            cur = nxt;
        }
        return true;
    }

    void deleteSubtree(BNode* n){
        if(!n) return;
        deleteSubtree(n->ch[0]); deleteSubtree(n->ch[1]);
        delete n;
    }

    // x와 XOR을 최대화하는 값의 XOR 결과
    uint64_t maxXorWith(uint64_t x) const {
        const BNode* cur = root;
        if(!cur) return 0;
        uint64_t ans = 0;
        for(int i = MAX_BIT; i >= 0; --i){
            int b = (x >> i) & 1;
            int want = 1 - b; // 반대 비트가 유리
            if(cur->ch[want] && cur->ch[want]->cnt > 0){
                ans |= (1ULL << i);
                cur = cur->ch[want];
            } else if(cur->ch[b] && cur->ch[b]->cnt > 0){
                cur = cur->ch[b];
            } else break; // 비정상(빈 트리 등)
        }
        return ans;
    }

    // x와 XOR을 최소화
    uint64_t minXorWith(uint64_t x) const {
        const BNode* cur = root;
        uint64_t ans = 0;
        for(int i = MAX_BIT; i >= 0; --i){
            int b = (x >> i) & 1;
            if(cur->ch[b] && cur->ch[b]->cnt > 0){
                // 같은 비트 선택 → XOR 0
                cur = cur->ch[b];
            } else if(cur->ch[1-b] && cur->ch[1-b]->cnt > 0){
                ans |= (1ULL << i);
                cur = cur->ch[1-b];
            } else break;
        }
        return ans;
    }
};
```

> **메모**  
> - `cnt`는 **경로상 모든 노드**에 저장하는 대신, 본 구현은 **하위 노드에만 카운트**(종단 포함)를 두고 하향하면서 `cnt--`로 정리한다.  
> - 삽입/삭제는 모두 \(O(\text{비트길이})\) — 32/64 고정이므로 사실상 **상수 시간**.

---

### 1.3 실전 패턴 ① — 배열 최대 XOR 쌍

> 문제: 정수 배열 `nums`에서 **서로 다른 두 수의 XOR 최대값**을 구하라.

```cpp
uint64_t maxPairXOR(const vector<uint32_t>& a){
    BinaryTrie T(31);
    uint64_t ans = 0;
    for(uint32_t x : a) T.insert(x);
    for(uint32_t x : a) ans = max(ans, T.maxXorWith(x));
    return ans;
}

int main(){
    vector<uint32_t> v = {3, 10, 5, 25, 2, 8};
    cout << "maxPairXOR = " << maxPairXOR(v) << "\n"; // 28 (5^25)
}
```

**복잡도**: \(O(n \cdot W)\), \(W=32\).

---

### 1.4 실전 패턴 ② — 부분배열 최대 XOR

아이디어: **prefix XOR**를 이용해  
\[
\max_{i<j} (pref[j]\oplus pref[i])
\]
가 되므로, prefix들을 이진 Trie에 누적하며 매 스텝 **최대 XOR** 갱신.

```cpp
uint32_t maxSubarrayXOR(const vector<uint32_t>& a){
    BinaryTrie T(31);
    uint32_t pref = 0, best = 0;
    T.insert(0); // 공집합 prefix
    for(uint32_t x : a){
        pref ^= x;
        best = max(best, (uint32_t)T.maxXorWith(pref));
        T.insert(pref);
    }
    return best;
}
```

---

### 1.5 실전 패턴 ③ — IPv4 LPM(가장 긴 접두사 일치)

- 라우팅 테이블: `prefix/len → nextHop`
- 탐색: 주소의 비트를 따라가며 **마지막으로 만난 매칭 엔트리**(nextHop 있는 노드)를 기억.

```cpp
struct RouteNode {
    RouteNode* ch[2]{};
    bool has = false;
    string nextHop;
    RouteNode(){ ch[0]=ch[1]=nullptr; }
};

struct IPv4Trie {
    RouteNode* root = new RouteNode();
    static uint32_t parseIP(const string& s){
        unsigned a,b,c,d; char dot;
        stringstream ss(s);
        ss>>a>>dot>>b>>dot>>c>>dot>>d;
        return (a<<24)|(b<<16)|(c<<8)|d;
    }
    void insert(const string& ipPrefix, int len, string hop){
        uint32_t ip = parseIP(ipPrefix);
        RouteNode* cur = root;
        for(int i=31;i>=32-len;--i){
            int b = (ip>>i)&1;
            if(!cur->ch[b]) cur->ch[b]=new RouteNode();
            cur=cur->ch[b];
        }
        cur->has = true; cur->nextHop = move(hop);
    }
    // 주소 ip에 대한 LPM
    string lookup(const string& ipStr){
        uint32_t ip = parseIP(ipStr);
        RouteNode* cur=root;
        string last="";
        for(int i=31;i>=0 && cur;i--){
            if(cur->has) last = cur->nextHop;
            int b=(ip>>i)&1;
            cur=cur->ch[b];
        }
        if(cur && cur->has) last = cur->nextHop;
        return last; // 없으면 빈 문자열
    }
};

int main(){
    IPv4Trie rt;
    rt.insert("10.0.0.0", 8,  "NH-A");
    rt.insert("10.1.0.0", 16, "NH-B");
    rt.insert("10.1.2.0", 24, "NH-C");

    cout<< rt.lookup("10.1.2.3")  <<"\n"; // NH-C
    cout<< rt.lookup("10.1.9.1")  <<"\n"; // NH-B
    cout<< rt.lookup("10.9.1.1")  <<"\n"; // NH-A
    cout<< rt.lookup("11.0.0.1")  <<"\n"; // ""
}
```

---

### 1.6 복잡도 & 메모리

- 고정 폭 \(W\)에 대해 **삽입/삭제/쿼리**: \(O(W)\) — 보통 \(W\in\{32,64\}\)  
- 노드 수 상한: **\(O(n \cdot W)\)** (중복/공유 경로로 실제는 더 작음)
- **메모리 팁**  
  - **풀링(메모리 풀/arena)**: `new` 오버헤드 감축  
  - **비트셋 압축 Patricia**: 노드에 **분기비트 인덱스**만 저장(고급 IP 라우팅에서 흔함)

---

## 2) 압축 Trie (Radix / Compressed Trie)

### 2.1 개념
- **단독 경로(차일드 1개)**가 연속되는 구간을 **한 노드(label)**로 **압축**한다.  
- 덕분에 깊이가 크게 줄고 메모리 효율이 좋아진다. (문자열 키에 적합)

> 본문은 **문자열 라딕스 트리**를 구현한다. (비트 기반 Patricia는 1장의 확장으로 이해 가능)

---

### 2.2 핵심 연산 — LCP(최장 공통 접두사)에 따른 **분할(splitting)**

케이스(부모→자식 `child.label`, 삽입할 `word`):
1) **공통 접두사 길이 = 0** → 다음 자식 검사
2) **공통 접두사 = child.label 전체**  
   - `word`가 더 길면 **하위로 재귀** (`word.substr(|child|)`)
   - `word`가 정확히 같다면 `child.terminal = true`
3) **공통 접두사가 child.label의 proper prefix**  
   → **분할(split)**: `child.label = lcp`, `child` 아래에 **기존 나머지**와 **새 나머지**를 각각 자식으로

---

### 2.3 완전 구현(삽입/검색/삭제/병합/디버그)

```cpp
#include <bits/stdc++.h>
using namespace std;

struct RNode {
    string label;      // 이 간선 구간(압축된 문자열)
    bool terminal;     // 이 노드에서 단어가 종료되는가?
    vector<RNode*> ch; // 자식 목록 (소수 구현: vector, 실전은 맵/해시 고려)

    RNode(string s="", bool end=false): label(move(s)), terminal(end) {}
};

struct RadixTrie {
    RNode* root = new RNode();

    ~RadixTrie(){ clear(root); }
    void clear(RNode* n){
        if(!n) return;
        for(auto* c : n->ch) clear(c);
        delete n;
    }

    static size_t lcpLen(string_view a, string_view b){
        size_t i=0, L=min(a.size(), b.size());
        while(i<L && a[i]==b[i]) ++i;
        return i;
    }

    void insert(const string& word){ insertRec(root, word); }

    void insertRec(RNode* node, const string& word){
        if(word.empty()){ node->terminal = true; return; }

        for(auto*& child : node->ch){
            size_t k = lcpLen(child->label, word);
            if(k==0) continue;

            if(k == child->label.size()){
                // child 전체가 일치 → 아래로 진행
                insertRec(child, word.substr(k));
                return;
            }else{
                // child를 split
                RNode* split = new RNode(child->label.substr(k), child->terminal);
                split->ch = move(child->ch);

                child->label.erase(k); // child.label = child.label[k:] → split로 옮겼으니 여기선 child.label=k? (정리)
                // 위 줄 대신 다음 두 줄:
                // child->label.resize(k);
                // ↑ 하지만 위에서 erase(k) 쓰면 k 이후가 지워짐. resize가 더 명시적.
                child->label.resize(k);
                child->terminal = (word.size()==k); // 단어가 여기서 끝나면 child를 terminal로

                child->ch.clear();
                child->ch.push_back(split);

                if(word.size()>k){
                    RNode* leaf = new RNode(word.substr(k), true);
                    child->ch.push_back(leaf);
                }
                return;
            }
        }
        // 공통 접두사 가진 자식 없음 → 새 자식으로 추가
        node->ch.push_back(new RNode(word, true));
    }

    bool search(const string& word) const { return searchRec(root, word); }
    bool searchRec(RNode* node, const string& word) const {
        if(word.empty()) return node->terminal;
        for(auto* child : node->ch){
            if(word.rfind(child->label, 0)==0){ // starts_with
                return searchRec(child, word.substr(child->label.size()));
            }
        }
        return false;
    }

    bool startsWith(const string& pref) const { return startsRec(root, pref); }
    bool startsRec(RNode* node, const string& pref) const {
        if(pref.empty()) return true;
        for(auto* child : node->ch){
            size_t k = lcpLen(child->label, pref);
            if(k==0) continue;
            if(k == pref.size()) return true;           // prefix 끝
            if(k == child->label.size())                // 더 내려가야
                return startsRec(child, pref.substr(k));
        }
        return false;
    }

    bool erase(const string& word){ return eraseRec(root, word); }
    // 삭제 후 병합: (terminal==false && 자식 1개) → 위로 병합
    bool eraseRec(RNode* node, const string& word){
        for(size_t idx=0; idx<node->ch.size(); ++idx){
            RNode* child = node->ch[idx];
            if(word.rfind(child->label,0)==0){
                string rest = word.substr(child->label.size());
                if(rest.empty()){
                    if(!child->terminal) return false; // 없음
                    child->terminal = false;
                    // 병합/정리
                    compressIfNeeded(node, idx);
                    return true;
                }else{
                    bool ok = eraseRec(child, rest);
                    if(ok) compressIfNeeded(node, idx);
                    return ok;
                }
            }else{
                size_t k = lcpLen(child->label, word);
                if(k>0 && k<child->label.size()){
                    // word가 child.label의 내부에서 끊겼다면 존재 X
                    return false;
                }
            }
        }
        return false;
    }

    void compressIfNeeded(RNode* parent, size_t idx){
        RNode* child = parent->ch[idx];
        // Case A: terminal==false && 자식 0 → 제거
        if(!child->terminal && child->ch.empty()){
            delete child;
            parent->ch.erase(parent->ch.begin()+idx);
            return;
        }
        // Case B: terminal==false && 자식 1 → 병합
        if(!child->terminal && child->ch.size()==1){
            RNode* g = child->ch[0];
            child->label += g->label;
            child->terminal = g->terminal;
            child->ch = move(g->ch);
            g->ch.clear();
            delete g;
        }
    }

    // 디버그 출력
    void dump() const { dumpRec(root, 0); }
    void dumpRec(RNode* n, int depth) const {
        if(n==nullptr) return;
        cout << string(depth*2,'-') << (n==root?"<root>":n->label)
             << (n->terminal?"*":"") << "\n";
        for(auto* c : n->ch) dumpRec(c, depth+1);
    }
};

int main(){
    RadixTrie T;
    // 고전 테스트 단어들(라틴계)
    vector<string> words={
        "romane","romanus","romulus","rubens","ruber","rubicon","rubicundus"
    };
    for(auto& w: words) T.insert(w);

    cout<<"-- dump after insert --\n"; T.dump();

    for(string q: {"romane","rom","rub","rubic","rubicundus","roma"}){
        cout<<q<<": search="<<T.search(q)<<" startsWith="<<T.startsWith(q)<<"\n";
    }

    cout<<"erase rubicon\n";
    T.erase("rubicon");
    T.dump();
}
```

**포인트**
- `insert`의 **split** 구문이 라딕스의 핵심. (기존 자식 분리 + 새 자식 추가)
- `erase`는 **종단 플래그 해제** 후, **(비종단&&자식1) 병합** 또는 **(비종단&&자식0) 제거**로 정리.

---

### 2.4 예시 동작 (개념적)

삽입 순서: `romane`, `romanus`, `romulus`, …  
초기 분기: `ro` → 이후 `ma...`/`mu...`/`bu...` 등으로 **한 번에 내려감** (압축 덕에 깊이 얕음)

```
<root>
-- ro
---- man
------ e*
------ us*
---- mulus*
-- rub
---- e
------ ns*
------ r*
---- icon*
---- icundus*
```

(`*`는 단어 끝/terminal)

---

## 3) 성능·메모리·테스트

### 3.1 복잡도
- 이진 Trie: \(O(W)\) (고정폭 비트) — 사실상 상수
- 압축 Trie: 평균 \(O(m)\) (\(m\)=문자열 길이), 단 각각의 LCP 비교 비용이 추가

### 3.2 최적화 팁
- **이진 Trie**
  - **메모리 풀**/슬랩 할당자: `new`/`delete` 비용 절감
  - **Patricia**(분기 비트 인덱스) → 노드 수·메모리 축소
- **라딕스**
  - 자식 컨테이너: `vector`(작은 분기) → `unordered_map<char,int>`(큰 분기) 하이브리드
  - 대소문자/노멀라이징/토크나이저로 LCP 가능성 ↑

### 3.3 퍼징/검증 (스케치)

```cpp
// 이진 Trie vs std::multiset 동치성
// 1) insert/erase 랜덤
// 2) contains 동치 확인
// 3) maxXorWith(x)는 O(n) 완전탐색 결과와 일치 검증(소규모 샘플)
```

---

## 4) 수학 스냅샷: XOR 최대화가 “반대 비트 선호”인 이유

한 비트 위치 \(i\)에서, \(x_i \in \{0,1\}\)이고 트라이에 \(p_i\in \{0,1\}\)가 존재할 때,  
해당 비트의 XOR 기여는
\[
x_i \oplus p_i =
\begin{cases}
1,& p_i = 1 - x_i\\
0,& p_i = x_i
\end{cases}
\]
이므로, **상위 비트부터** 가능한 경우 **반대 비트**를 고르는 것이 전체 XOR(해당 비트의 가중치 \(2^i\))를 최대로 만든다.  
(하위 비트는 상위 비트 결정 이후에야 영향 — 그리디가 성립)

---

## 5) 자주 하는 실수 & 디버깅 포인트

1. **이진 Trie 삭제** 시 `cnt` 감소/가지치기 누락 → 유령 경로(contains 거짓양성)
2. **IPv4 LPM**에서 “마지막 매칭(nextHop)” 갱신을 잊음 → 더 짧은 프리픽스 미반영
3. **라딕스 split**:  
   - 분할 후 **기존 자식의 label/terminal/children 이동** 순서 꼬임  
   - `word.size()==k`(단어가 lcp로 종료)일 때 **child를 terminal=true**로 설정 잊음
4. **삭제 후 병합** 조건 누락 → 불필요하게 깊은 트리 유지

---

## 6) 요약

- **이진 Trie**: 정수 비트 문제(XOR 최대/최소, 부분배열 XOR, IP LPM)에 **정답 같은 선택**.  
- **압축 Trie(Radix)**: 문자열 세상에서 **깊이를 줄이고 메모리를 절약**.  
- 구현의 핵심은 **경계 케이스**(split/erase/병합)와 **경로 카운트 관리**.  
- 실전에서는 **메모리 풀**, **하이브리드 자식 컨테이너**, **Patricia 변형**으로 성능을 한 단계 더 끌어올린다.

---
```cpp
// 단일 파일 스니펫(빌드용 최소 예): BinaryTrie + maxPairXOR
#include <bits/stdc++.h>
using namespace std;
struct BNode{ BNode* ch[2]; int cnt; BNode():ch{nullptr,nullptr},cnt(0){} };
struct BinaryTrie{
    BNode* root; int MAX_BIT; BinaryTrie(int mb=31):MAX_BIT(mb){ root=new BNode(); }
    ~BinaryTrie(){ clear(root); }
    void clear(BNode* n){ if(!n) return; clear(n->ch[0]); clear(n->ch[1]); delete n; }
    void insert(uint32_t x){ BNode* cur=root; for(int i=31;i>=0;--i){int b=(x>>i)&1; if(!cur->ch[b]) cur->ch[b]=new BNode(); cur=cur->ch[b]; cur->cnt++;}}
    uint32_t maxXorWith(uint32_t x) const{ const BNode* cur=root; uint32_t ans=0; for(int i=31;i>=0;--i){int b=(x>>i)&1, w=1-b; if(cur->ch[w]&&cur->ch[w]->cnt){ ans|=(1u<<i); cur=cur->ch[w]; } else if(cur->ch[b]&&cur->ch[b]->cnt){ cur=cur->ch[b]; } else break; } return ans; }
};
uint32_t maxPairXOR(const vector<uint32_t>& a){ BinaryTrie T; for(auto x:a) T.insert(x); uint32_t ans=0; for(auto x:a) ans=max(ans,T.maxXorWith(x)); return ans; }
int main(){ vector<uint32_t> v={3,10,5,25,2,8}; cout<<maxPairXOR(v)<<"\n"; }
```