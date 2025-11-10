---
layout: post
title: Data Structure - 자료구조란
date: 2024-12-07 19:20:23 +0900
category: Data Structure
---
# 자료구조란 무엇인가?

## 1. 자료구조의 정의

**자료구조(Data Structure)**란 데이터를 효율적으로 저장하고 관리하며, 필요한 작업(검색, 삽입, 삭제 등)을 빠르게 수행할 수 있도록 구성한 **데이터의 조직 방식**이다.  
즉, **데이터를 저장하는 그릇**이며, 이 그릇의 형태에 따라 가능한 연산과 성능 특성이 달라진다.

---

## 2. 왜 중요한가? (근거 중심)

- **시간 복잡도**: 같은 문제도 자료구조 선택에 따라 $$O(n) \to O(\log n)$$, $$O(n^2) \to O(n \log n)$$로 급감한다.
- **공간 복잡도**: 메모리 레이아웃(연속/불연속)에 따라 캐시 히트율이 달라지고 실제 런타임이 크게 변한다.
- **불변식(Invariant)**을 통해 **정확성**을 보장한다(예: 힙 속성, BST 정렬 순서, 해시 부하율 등).
- **확장성**: 대규모 데이터(스트리밍·분산)로 갈수록 적절한 자료구조가 필수이다.

---

## 3. 분류 (기존 초안 확장)

### 3.1 선형 자료구조 (Linear)
- **배열(Array)**: 연속 메모리. 인덱스 조회 $$O(1)$$, 중간 삽입/삭제 $$O(n)$$.
- **문자열(String)**: 불변/가변 구현에 따른 비용 차이(파이썬 불변, C# `String` 불변 / `StringBuilder` 가변).
- **연결 리스트(Linked List)**: 임의 접근은 느리나, **포인터 조작으로 국소 삽입/삭제 $$O(1)$$**.
- **스택(Stack)**: LIFO. 재귀 제거, 백트래킹.
- **큐(Queue)**: FIFO. BFS, 작업 스케줄링.
- **덱(Deque)**: 양측 삽입/삭제 $$O(1)$$(구현에 따라 상수항 다름).
- **우선순위 큐(Priority Queue)**: 보통 바이너리 힙으로 구현. `push/pop` $$O(\log n)$$.

### 3.2 비선형 자료구조 (Non-Linear)
- **트리(Tree)**: 계층 구조.
  - **BST**: 정렬 상태 유지, 탐색/삽입/삭제 기대 $$O(\log n)$$(균형 시).
  - **균형 트리(AVL, Red-Black)**: 최악 $$O(\log n)$$ 보장.
  - **힙(Heap)**: 최댓값/최솟값 추출 특화.
  - **B-Tree/B+Tree**: 외부 메모리(디스크/SSD) 친화 인덱스.
  - **Segment Tree/Fenwick(BIT)**: 구간 질의/갱신.
  - **Trie(프리픽스 트리)**: 문자열 공통 접두사 공유.
- **그래프(Graph)**: 정점/간선. 인접 리스트/행렬 표현. 탐색(BFS/DFS), 최단경로, MST 등.

### 3.3 해시 기반
- **해시 테이블(Hash Table)**: 평균 $$O(1)$$ 탐색/삽입/삭제. 충돌 해결(체이닝/오픈 어드레싱).

### 3.4 특수/확장
- **Disjoint Set(Union-Find)**: 집합 합치기/대표 찾기(경로 압축 + 랭크).
- **Skip List**: 다층 연결 리스트로 평균 $$O(\log n)$$.
- **Bloom Filter/Counting Bloom**: 확률적 멤버십 테스트.
- **Rope/Splay/Order-Statistic Tree**: 문자열 편집, 순위 쿼리.

---

## 4. 선택 기준 (의사결정 체크리스트)

- **접근 패턴**: 랜덤 인덱스가 많으면 배열/벤치 친화 구조, 순차/스트림이면 큐/덱.
- **연산 빈도**: 삽입/삭제가 빈번하고 위치가 임의면 리스트/균형트리, 끝 삽입이면 동적 배열.
- **정렬 필요**: 정렬 유지 필요 시 BST/균형 트리/우선순위 큐.
- **키-값 조회**: 해시(평균 $$O(1)$$) vs 정렬 맵(정렬 순회/범위 질의는 트리).
- **메모리/캐시 로컬리티**: 배열/벡터 기반이 유리.
- **동시성**: 락 경합/ABA 문제/무잠금 구조 고려.
- **외부 메모리**: B-Tree 류(페이지 크기 최적화).

---

## 5. 자료구조 × 알고리즘 (그릇과 요리)

- 정렬: 배열/벡터에서 in-place 최적화.
- 그래프 탐색: 인접 리스트 + 큐/스택 조합.
- 다익스트라: 인접 리스트 + 우선순위 큐.
- 문자열 검색: Trie/스파스 테이블/접미사 구조.
- 온라인 통계: Fenwick/Segment Tree.

---

## 6. 복잡도 모델과 표기

- **점근 표기**: $$O(\cdot), \Omega(\cdot), \Theta(\cdot)$$
- **평균 vs 최악**: 해시는 평균 $$O(1)$$, 최악 $$O(n)$$ 가능.
- **암달 법칙/캐시 효과**: 단순 $$O$$ 비교 외 상수항·메모리 계층 구조를 고려.

---

## 7. 자료구조 카탈로그 & 실전 예제

아래는 각 구조별 **개념 → 핵심 불변식 → 주요 연산 → 복잡도 → 실수 포인트 → 테스트 전략 → 코드 예제(파이썬/CPP/C#)** 순서로 제시한다.

---

### 7.1 배열 & 문자열

#### 개념/불변식
- 배열: 고정 크기 또는 동적(벡터) 확장.
- 문자열: 불변/가변 여부가 성능을 좌우.

#### 주요 연산 & 복잡도
- 인덱스 접근: $$O(1)$$
- 중간 삽입/삭제: $$O(n)$$ (시프트)
- 동적 배열 확장: 상환분석 $$O(1)$$ amortized

#### 파이썬 예제
```python
# 슬라이싱, 확장, 상환 분석 감각
arr = [1,2,3]
arr.append(4)      # amortized O(1)
arr.insert(1, 99)  # O(n)
s = "hello"
t = s + " world"   # 새 문자열 할당 (불변)
```

#### C++ 예제
```cpp
#include <bits/stdc++.h>
using namespace std;
int main() {
    vector<int> v; 
    for (int i=0;i<10;i++) v.push_back(i); // amortized O(1)
    v.insert(v.begin()+5, 42);             // O(n)
    // 문자열
    string s="hello";
    s += " world"; // COW 구현 여부는 표준 이후 구현에 따라 다름
    cout << v[5] << " " << s << "\n";
}
```

#### C# 예제
```csharp
using System;
using System.Text;
class Program {
    static void Main() {
        var list = new System.Collections.Generic.List<int>();
        for (int i=0;i<10;i++) list.Add(i); // amortized O(1)
        list.Insert(5, 42);                 // O(n)
        string s = "hello";
        var sb = new StringBuilder(s);
        sb.Append(" world");                 // 가변 버퍼 → 잦은 연결 최적
        Console.WriteLine(sb.ToString());
    }
}
```

**디버깅 팁**: 인덱스 범위/오프바이원, 슬라이스 경계 확인.  
**테스트**: 빈 배열/한 원소/대량/경계 인덱스.

---

### 7.2 연결 리스트 (단일/이중/원형)

#### 핵심 아이디어
- 포인터로 노드를 연결하여 **국소 삽입/삭제 $$O(1)$$**.
- 임의 접근은 $$O(n)$$.

#### 불변식
- 포인터 연결 무결성(끊김/사이클) 유지.
- 이중 리스트: `prev.next == node && next.prev == node`.

#### 파이썬 (단일 리스트 직접 구현)
```python
class Node:
    def __init__(self, val, nxt=None):
        self.val = val; self.next = nxt

class SinglyList:
    def __init__(self): self.head=None
    def push_front(self, x):
        self.head = Node(x, self.head)  # O(1)
    def pop_front(self):
        if not self.head: raise IndexError("empty")
        x = self.head.val; self.head = self.head.next; return x
    def insert_after(self, node, x):
        node.next = Node(x, node.next)  # O(1) at node
```

#### C++ (이중 연결 리스트)
```cpp
struct Node {
    int val; Node *prev,*next;
    Node(int v):val(v),prev(nullptr),next(nullptr){}
};
struct DList {
    Node *head=nullptr, *tail=nullptr;
    void push_front(int x){
        Node* n=new Node(x);
        n->next=head; if(head) head->prev=n; else tail=n; head=n;
    }
    void erase(Node* n){
        if(!n) return;
        if(n->prev) n->prev->next=n->next; else head=n->next;
        if(n->next) n->next->prev=n->prev; else tail=n->prev;
        delete n;
    }
};
```

**실수**: 삭제 시 포인터 업데이트 누락, 메모리 해제 누락.  
**테스트**: 빈/1개/양끝/중간/연속 삭제.

---

### 7.3 스택/큐/덱

#### 스택 (LIFO)
- 용도: 괄호검사, DFS 비재귀, 되돌리기.

```python
stack=[]
stack.append(1)  # push
x=stack.pop()    # pop
```

#### 큐 (FIFO)
- BFS, 생산자-소비자.

```python
from collections import deque
q=deque()
q.append(1)
x=q.popleft()
```

#### 덱
- 양끝 삽입/삭제 $$O(1)$$.
- 슬라이딩 윈도우 최대값(모노토닉 덱) 활용.

```python
from collections import deque
def sliding_max(a, k):
    dq=deque(); ans=[]
    for i,x in enumerate(a):
        while dq and a[dq[-1]] <= x: dq.pop()
        dq.append(i)
        if dq[0] == i-k: dq.popleft()
        if i>=k-1: ans.append(a[dq[0]])
    return ans
```

---

### 7.4 해시 테이블

#### 개념
- 해시함수 $$h(k)$$로 버킷 인덱스 계산.
- **충돌 해결**: 체이닝(리스트/벡터), 오픈 어드레싱(선형/이차/더블해싱).

#### 불변식
- **부하율** $$\alpha = n / m$$ 관리(재해시 기준).
- 오픈 어드레싱: 삭제 마커/탐사 불변 유지.

#### 파이썬 (dict 기본 예)
```python
d={}
d["tom"]=90
d["jane"]=85
d["tom"]=95        # 갱신
del d["jane"]
```

#### C++ (unordered_map)
```cpp
#include <bits/stdc++.h>
using namespace std;
int main(){
    unordered_map<string,int> mp;
    mp["tom"]=90; mp["jane"]=85;
    mp["tom"]=95;
    if (auto it=mp.find("jane"); it!=mp.end()) mp.erase(it);
}
```

**실수**: 사용자 정의 키의 해시/동치 연산자 불일치, 부하율 방치.  
**테스트**: 충돌 유도 키 세트, 삭제/재삽입, 대량 재해시.

---

### 7.5 트리와 균형 트리 (BST, AVL)

#### BST 기본
- 불변식: 왼쪽 < 루트 < 오른쪽 (정렬).
- 탐색/삽입/삭제: 평균 $$O(\log n)$$, 최악 $$O(n)$$.

#### AVL(균형)
- 각 노드 **높이 차이(|bf| ≤ 1)** 유지 → 항상 $$O(\log n)$$.
- 회전(LL/LR/RL/RR).

##### C++ 간단 AVL (핵심 아이디어만)
```cpp
struct Node{
    int key,h; Node* l; Node* r;
    Node(int k):key(k),h(1),l(nullptr),r(nullptr){}
};
int H(Node* n){return n? n->h:0;}
int BF(Node* n){return H(n->l)-H(n->r);}
void upd(Node* n){ if(n) n->h = max(H(n->l),H(n->r))+1; }

Node* rotR(Node* y){
    Node* x=y->l; Node* T2=x->r;
    x->r=y; y->l=T2; upd(y); upd(x); return x;
}
Node* rotL(Node* x){
    Node* y=x->r; Node* T2=y->l;
    y->l=x; x->r=T2; upd(x); upd(y); return y;
}
Node* balance(Node* n){
    upd(n);
    int b=BF(n);
    if(b>1){ if(BF(n->l)<0) n->l=rotL(n->l); return rotR(n); }
    if(b<-1){ if(BF(n->r)>0) n->r=rotR(n->r); return rotL(n); }
    return n;
}
Node* insert(Node* n,int key){
    if(!n) return new Node(key);
    if(key<n->key) n->l=insert(n->l,key);
    else if(key>n->key) n->r=insert(n->r,key);
    else return n; // no dup
    return balance(n);
}
```

**테스트**: 단조 증가 입력(회전 발생), 무작위 삽입/삭제, 중복 키.

---

### 7.6 힙(Heap) & 우선순위 큐

- **배열 기반 완전 이진 트리**.
- Heapify: $$O(n)$$, push/pop: $$O(\log n)$$.

#### 파이썬
```python
import heapq
pq=[]
heapq.heappush(pq, (priority, item))
p,it = heapq.heappop(pq)
```

#### C++
```cpp
#include <bits/stdc++.h>
using namespace std;
int main(){
    priority_queue<int> maxh;
    maxh.push(3); maxh.push(5); maxh.push(1);
    cout<<maxh.top()<<"\n"; // 5
}
```

**응용**: 다익스트라, k-way merge, 스트림 상위 k개.

---

### 7.7 B-Tree / B+Tree (DB 인덱스)

- **페이지 단위** 노드, 높은 branching factor → 트리 높이 작음.
- 디스크 I/O 최소화.  
- B+Tree: 모든 키는 리프에, 리프 간 연결 리스트로 범위 스캔 효율적.

**핵심 아이디어**만 서술(코드 구현은 방대):
- 삽입/삭제 시 **분할(split)과 병합(merge)**.
- 페이지 크기(4KB/8KB)에 맞춘 키/포인터 수 조절.

---

### 7.8 세그먼트 트리 & Fenwick(BIT)

#### 기능
- **구간 질의/갱신**: 합/최소/최대/곱/최대 연속합 등.
- Segment Tree: $$O(\log n)$$ 질의/갱신, 구현 쉬움, 메모리 약 4n.
- Fenwick: 합/차량형 연산에 특화, 구현 간단, 메모리 n.

#### Fenwick 파이썬
```python
class BIT:
    def __init__(self, n):
        self.n=n; self.f=[0]*(n+1)
    def add(self, i, delta):
        while i<=self.n:
            self.f[i]+=delta
            i += i & -i
    def sum(self, i):
        s=0
        while i>0:
            s+=self.f[i]
            i -= i & -i
        return s
    def range_sum(self, l, r):
        return self.sum(r)-self.sum(l-1)
```

#### 세그먼트 트리 C++
```cpp
struct Seg {
    int n; vector<long long> t;
    Seg(int n):n(n),t(4*n,0){}
    void build(vector<int>&a,int v,int tl,int tr){
        if(tl==tr) t[v]=a[tl];
        else{
            int tm=(tl+tr)/2;
            build(a,v*2,tl,tm);
            build(a,v*2+1,tm+1,tr);
            t[v]=t[v*2]+t[v*2+1];
        }
    }
    void update(int v,int tl,int tr,int pos,int val){
        if(tl==tr)t[v]=val;
        else{
            int tm=(tl+tr)/2;
            if(pos<=tm) update(v*2,tl,tm,pos,val);
            else update(v*2+1,tm+1,tr,pos,val);
            t[v]=t[v*2]+t[v*2+1];
        }
    }
    long long query(int v,int tl,int tr,int l,int r){
        if(l>r) return 0;
        if(l==tl && r==tr) return t[v];
        int tm=(tl+tr)/2;
        return query(v*2,tl,tm,l,min(r,tm))
             + query(v*2+1,tm+1,tr,max(l,tm+1),r);
    }
};
```

**실수**: 인덱싱(1/0 기반) 혼용, 구간 경계.  
**테스트**: 랜덤 빌드/랜덤 업데이트/브루트포스 검증.

---

### 7.9 Trie (Prefix Tree)

#### 용도
- 사전/자동완성/접두사 카운트.
- 문자열 길이를 m이라 할 때 삽입/탐색 $$O(m)$$.

#### 파이썬
```python
class Trie:
    def __init__(self): self.trie={}
    def insert(self, w):
        node=self.trie
        for ch in w:
            node=node.setdefault(ch,{})
        node['#']=True
    def search(self, w):
        node=self.trie
        for ch in w:
            if ch not in node: return False
            node=node[ch]
        return '#' in node
    def startswith(self, p):
        node=self.trie
        for ch in p:
            if ch not in node: return False
            node=node[ch]
        return True
```

**확장**: 알파벳 고정 시 배열(고정 26/128/256)로 가속.

---

### 7.10 그래프 (표현·탐색·경로)

#### 표현
- **인접 리스트**: 희소 그래프 권장.
- **인접 행렬**: 밀집 그래프/상수 시간 간선 존재 확인.

#### BFS/DFS 파이썬
```python
from collections import deque, defaultdict
def bfs(n, edges, s):
    g=defaultdict(list)
    for u,v in edges: g[u].append(v); g[v].append(u)
    dist=[-1]*n; dist[s]=0
    q=deque([s])
    while q:
        u=q.popleft()
        for w in g[u]:
            if dist[w]==-1:
                dist[w]=dist[u]+1; q.append(w)
    return dist

def dfs_rec(g,u,vis):
    vis[u]=True
    for w in g[u]:
        if not vis[w]: dfs_rec(g,w,vis)
```

#### 다익스트라 (비음수 가중치)
```python
import heapq
def dijkstra(n, adj, s):
    INF=10**18
    dist=[INF]*n; dist[s]=0
    pq=[(0,s)]
    while pq:
        d,u=heapq.heappop(pq)
        if d!=dist[u]: continue
        for v,w in adj[u]:
            if dist[v]>d+w:
                dist[v]=d+w
                heapq.heappush(pq,(dist[v],v))
    return dist
```

#### MST (크루스칼 + Union-Find)
```python
def find(p,x):
    if p[x]!=x: p[x]=find(p,p[x])
    return p[x]
def unite(p,r,x,y):
    x=find(p,x); y=find(p,y)
    if x==y: return False
    if r[x]<r[y]: x,y=y,x
    p[y]=x
    if r[x]==r[y]: r[x]+=1
    return True

def kruskal(n, edges): # edges: (w,u,v)
    p=list(range(n)); r=[0]*n
    res=0; edges.sort()
    for w,u,v in edges:
        if unite(p,r,u,v): res+=w
    return res
```

---

### 7.11 Union-Find (Disjoint Set)

- 경로 압축 + 랭크/사이즈 → 거의 상수 시간(아커만 역함수).
- 연결성 질의, 사이클 검사, 크루스칼.

**테스트**: 무작위 unite/find 시 브루트포스 연결성 비교.

---

### 7.12 Skip List (개념 + 파이썬 스케치)

- 레벨 당 절반 확률로 승격 → 기대 높이 $$O(\log n)$$.
- 정렬 맵 대안(평균 성능), 구현 간단.

```python
import random
MAXLV=16
class Node:
    def __init__(self, key, lv):
        self.key=key; self.next=[None]*(lv+1)
class SkipList:
    def __init__(self):
        self.h=0; self.head=Node(None, MAXLV)
    def _randlv(self):
        lv=0
        while lv<MAXLV and random.getrandbits(1): lv+=1
        return lv
    def insert(self, key):
        update=[None]*(MAXLV+1)
        x=self.head
        for lv in range(self.h,-1,-1):
            while x.next[lv] and x.next[lv].key<key:
                x=x.next[lv]
            update[lv]=x
        x=x.next[0]
        if x and x.key==key: return
        lv=self._randlv()
        if lv>self.h:
            for i in range(self.h+1, lv+1): update[i]=self.head
            self.h=lv
        x=Node(key, lv)
        for i in range(lv+1):
            x.next[i]=update[i].next[i]
            update[i].next[i]=x
```

---

### 7.13 Bloom Filter

- **거짓 양성** 허용, 거짓 음성 없음.  
- $$m$$ 비트 배열, $$k$$ 해시, $$n$$ 원소일 때 거짓 양성률:
  $$
  \left(1 - e^{-kn/m}\right)^k
  $$
- 캐시/저장공간 절약형 멤버십 테스트, DB 앞단 캐시.

---

## 8. 문자열/배열 고급 주제

- **Two-pointer/Sliding Window**: 부분 문자열/서브어레이 최적화.
- **Prefix Function/KMP**: 패턴 매칭 $$O(n+m)$$.
- **Rolling Hash**: 라빈-카프, 서브스트링 비교/중복 탐지.
- **Rope**: 대규모 문자열 편집(편집기).

---

## 9. 동시성·멀티스레드 고려 (개요)

- 락 기반 큐/스택 vs 무잠금(atomic, CAS, ABA).
- 생산자-소비자: 고정 길이 **원형 버퍼**(ring buffer) + 세마포어.
- 메모리 모델(C#/C++): volatile/atomic, happens-before.

*C# ConcurrentQueue/Bag, C++ `<atomic>`/folly, .NET Channel 등의 고수준 활용 권장.*

---

## 10. 테스트 & 검증 습관

- **브루트포스 크로스체크**: 작은 N에서 느린 구현과 결과 비교.
- **퍼징**: 랜덤 연산 시퀀스 → 불변식 검증.
- **경계 케이스**: 빈 구조, 1개, 최대 용량, 중복 키.
- **시간측정/프로파일링**: 캐시 효과까지 함께 관찰.

---

## 11. 면접/코테 실전 팁

- **필수 암기**: 복잡도 표, 불변식(힙/AVL/Union-Find), 표준 라이브러리 API.
- **선택 기준 설명력**: “왜 이 구조를 썼는가?”를 상황과 데이터 패턴으로 설명.
- **실수 방지**: 인덱스/경계, 삭제 후 포인터/참조 무효화, overflow.

---

## 12. 종합 예제: 로그 스트림 Top-K, 구간 질의, 최단경로

### 12.1 스트림에서 최근 N 중 상위 K (우선순위 큐)
```python
import heapq
def topk_stream(stream, k):
    pq=[]
    for x in stream:
        if len(pq)<k: heapq.heappush(pq, x)
        else:
            if x>pq[0]:
                heapq.heapreplace(pq, x)
    return sorted(pq, reverse=True)
```

### 12.2 구간 합 온라인 업데이트 (Fenwick)
```python
# 위 BIT 클래스 참고
bit=BIT(10)
for i,val in enumerate([3,1,4,1,5,9,2,6,5,3], start=1):
    bit.add(i,val)
print(bit.range_sum(3,7))  # 4+1+5+9+2
bit.add(5, +10)            # a[5]+=10
```

### 12.3 도로망 최단경로 (다익스트라 + 힙)
```python
# 위 dijkstra 참고
n=5
adj=[[] for _ in range(n)]
def add(u,v,w): adj[u].append((v,w)); adj[v].append((u,w))
add(0,1,2); add(1,2,2); add(0,3,5); add(3,4,1); add(4,2,1)
print(dijkstra(n, adj, 0))
```

---

## 13. 복잡도 요약표 (자주 쓰는 연산)

| 구조 | 접근 | 검색 | 삽입 | 삭제 | 비고 |
|---|---:|---:|---:|---:|---|
| 배열 | $$O(1)$$ | $$O(n)$$ | 중간 $$O(n)$$ | 중간 $$O(n)$$ | 끝 삽입 amortized $$O(1)$$ |
| 연결 리스트 | $$O(n)$$ | $$O(n)$$ | 위치 주어지면 $$O(1)$$ | 위치 주어지면 $$O(1)$$ | 임의 접근 비효율 |
| 스택/큐 | - | - | $$O(1)$$ | $$O(1)$$ | 덱도 양끝 $$O(1)$$ |
| 해시 | - | 평균 $$O(1)$$ | 평균 $$O(1)$$ | 평균 $$O(1)$$ | 최악 $$O(n)$$ |
| BST | $$O(\log n)$$ | $$O(\log n)$$ | $$O(\log n)$$ | $$O(\log n)$$ | 불균형 시 최악 $$O(n)$$ |
| AVL/RB | $$O(\log n)$$ | $$O(\log n)$$ | $$O(\log n)$$ | $$O(\log n)$$ | 최악 보장 |
| 힙 | - | - | $$O(\log n)$$ | $$O(\log n)$$ | top 조회 $$O(1)$$ |
| 세그트리 | - | $$O(\log n)$$ | $$O(\log n)$$ | $$O(\log n)$$ | 구간 연산 |
| Fenwick | - | $$O(\log n)$$ | $$O(\log n)$$ | $$O(\log n)$$ | 누적합 특화 |
| Trie | 길이 m | $$O(m)$$ | $$O(m)$$ | $$O(m)$$ | 알파벳 고정 시 상수 가속 |

---

## 14. 학습/실전 로드맵 (확장版)

1. **배열/문자열/슬라이딩 윈도우**
2. **스택/큐/덱 + 모노토닉 구조**
3. **해시(충돌·부하율·커스텀 키)**
4. **트리(BST → AVL/RB) + 힙**
5. **그래프(표현/탐색/경로/MST)**
6. **세그먼트 트리/Fenwick/스파스 테이블**
7. **Trie/접미사 구조(개요)**
8. **Union-Find/Skip List/Bloom Filter**
9. **동시성 구조(개요) & DB 인덱스 관점(B-Tree)**

---

## 15. 부록: 수학/상환분석 스냅샷

- **동적 배열 상환분석**: 용량 2배씩 확장 시, n번 `push_back`의 총 복사 횟수는  
  $$
  \sum_{i=0}^{\lfloor \log n \rfloor} \frac{n}{2^i} = O(n)
  $$
  → **평균 1회당 $$O(1)$$ 상환**.

- **Union-Find** 시간**:  
  $$
  O((n+m)\,\alpha(n))
  $$
  여기서 $$\alpha(\cdot)$$는 아커만 함수의 역함수(실무상 거의 상수).

- **Bloom Filter 최적 k** (주어진 $$m,n$$):  
  $$
  k = \frac{m}{n}\ln 2
  $$
