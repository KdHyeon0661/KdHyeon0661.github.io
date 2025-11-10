---
layout: post
title: Data Structure - 연결 리스트 고급
date: 2024-12-09 20:20:23 +0900
category: Data Structure
---
# 특수한 연결 리스트들

## 0) 공통 배경: 왜 다양한 “링크”가 필요한가?

- **목표가 다르다**: 정렬 유지/빠른 탐색(스킵), 공간 절약(XOR), 경계 단순화(센티넬), 캐시 최적화(Unrolled), 2D 구조(다중 포인터) 등.
- **단점 보완**: 연결 리스트는 랜덤 접근이 약하지만, 설계에 따라 **순회 비용/상수항/캐시 미스**를 줄일 수 있다.
- **표준 컨테이너와 비교**: 사용 전 `std::list`/`std::forward_list`/`std::deque`/`std::vector`의 장단을 선검토. 특수 리스트는 **특정 요구**가 있을 때만 도입하는 게 합리적.

---

## 1) 다중 연결 리스트 (Multi-level / Multi-pointer)

### 1.1 개념 & 동기

하나의 노드가 **여러 방향 포인터**(예: `right`, `down`)를 갖는 구조. 2D 이상의 격자형/행렬형 데이터를 “링크”로 엮는다.

- **장점**: 희소 행렬/지도/GUI 그리드 등 **국소적 수정**이 많은 경우 유리(행/열 삽입·삭제 O(1) 재배선).
- **단점**: 임의 접근이 느림, 포인터 관리 복잡.

### 1.2 2D 리스트 기본 구현

```cpp
#include <vector>
#include <iostream>

struct Cell {
    int   val;
    Cell* right;
    Cell* down;
    explicit Cell(int v) : val(v), right(nullptr), down(nullptr) {}
};

// rows x cols 2D linked grid 생성: 각 셀은 right/down으로 연결
Cell* createGrid(int rows, int cols) {
    if (rows<=0 || cols<=0) return nullptr;
    std::vector<std::vector<Cell*>> nodes(rows, std::vector<Cell*>(cols));
    for (int i=0;i<rows;++i)
        for (int j=0;j<cols;++j)
            nodes[i][j] = new Cell(i*cols + j);

    for (int i=0;i<rows;++i){
        for (int j=0;j<cols;++j){
            if (j+1<cols) nodes[i][j]->right = nodes[i][j+1];
            if (i+1<rows) nodes[i][j]->down  = nodes[i+1][j];
        }
    }
    return nodes[0][0]; // head of grid
}

void printRow(Cell* rowHead){
    for (Cell* c=rowHead; c; c=c->right) std::cout<<c->val<<" ";
    std::cout<<"\n";
}

// 행 앞에 새 행 삽입: 새 행의 각 노드를 기존 첫 행 앞쪽에 붙임 (O(cols))
Cell* insertRowFront(Cell* head, int cols, int baseVal){
    // 새 행 노드 생성
    std::vector<Cell*> row(cols);
    for (int j=0;j<cols;++j) row[j]=new Cell(baseVal + j);
    for (int j=0;j<cols-1;++j) row[j]->right = row[j+1];
    // 수직 연결
    Cell* old = head;
    for (int j=0;j<cols && old; ++j){
        row[j]->down = old;
        old = old->right;
    }
    return row[0]; // 새 head
}

void freeGrid(Cell* head){
    // 행단위로 내려가며 해제 (방문 체크 없이 full grid 가정)
    for(Cell* r=head; r; ){
        Cell* nextRow = r->down;
        for(Cell* c=r; c; ){
            Cell* next = c->right;
            delete c; c=next;
        }
        r = nextRow;
    }
}
```

**적용 예시**:  
- **희소 행렬 편집기**: 필요 셀만 동적 할당 → 메모리 효율  
- **게임 맵**: 타일 삽입/삭제 빈번  
- **스프레드시트**: 행/열 삽입이 많으면 배열보다 유리

**복잡도**:  
- 행/열 **국소 삽입/삭제**: 링크 재배선 O(1)~O(k)  
- 전체 순회: O(rows·cols)  
- 인덱스 접근: O(rows + cols) (보통 직접 포인터 보관으로 상쇄)

---

## 2) 스킵 리스트 (Skip List)

> 자세 구현은 별도 글에서 풀었다(삽입/삭제/경계/반복자/퍼징 검증 등). 여기선 **요지 + 핵심 코드 요약**을 제공.

### 2.1 아이디어

정렬된 레벨-0 리스트 위에 **상위 레벨들이 희소하게** 존재. 레벨이 높을수록 **멀리 점프**, 평균 **탐색 O(log n)**.

- 노드 레벨은 확률 \(P\) (보통 0.5)로 승격:
  \[
  \Pr[\text{level} \ge \ell] = P^\ell
  \]
- 최상위 레벨 \(L \approx \log_{1/P} n\)

### 2.2 기본 골격 (요약 템플릿)

```cpp
#include <vector>
#include <random>
#include <functional>

template <class Key, class Compare=std::less<Key>>
class SkipList {
    struct Node {
        Key key;
        std::vector<Node*> next; // level 0..lv
        explicit Node(int lv, const Key& k): key(k), next(lv+1,nullptr) {}
    };
    Node* head_;
    int   level_;
    int   maxLevel_;
    float p_;
    Compare comp_;
    std::mt19937_64 rng_; std::uniform_real_distribution<float> dist_{0,1};

    int randomLevel(){ int lv=0; while (lv<maxLevel_ && dist_(rng_)<p_) ++lv; return lv; }

public:
    explicit SkipList(int maxL=32, float prob=0.5): level_(0), maxLevel_(maxL), p_(prob){
        head_ = new Node(maxLevel_, Key{}); // sentinel-like head
    }
    ~SkipList(){ clear(); delete head_; }

    bool insert(const Key& k){
        std::vector<Node*> upd(maxLevel_+1,nullptr);
        Node* cur=head_;
        for(int lv=level_; lv>=0; --lv){
            while (cur->next[lv] && comp_(cur->next[lv]->key, k)) cur=cur->next[lv];
            upd[lv]=cur;
        }
        cur = cur->next[0];
        if (cur && !comp_(k,cur->key) && !comp_(cur->key,k)) return false; // dup
        int nl = randomLevel();
        if (nl>level_) for(int i=level_+1;i<=nl;++i) upd[i]=head_, level_=nl;
        Node* nn = new Node(nl, k);
        for (int i=0;i<=nl;++i){ nn->next[i]=upd[i]->next[i]; upd[i]->next[i]=nn; }
        return true;
    }
    bool contains(const Key& k) const {
        const Node* cur=head_;
        for(int lv=level_; lv>=0; --lv)
            while (cur->next[lv] && comp_(cur->next[lv]->key, k)) cur=cur->next[lv];
        cur = cur->next[0];
        return cur && !comp_(k,cur->key) && !comp_(cur->key,k);
    }
    void clear(){
        Node* c=head_->next[0];
        while(c){ Node* nx=c->next[0]; delete c; c=nx; }
        for(int i=0;i<=level_;++i) head_->next[i]=nullptr;
        level_=0;
    }
};
```

**장점**: 정렬 유지 + 평균 `O(log n)` 탐색/삽입/삭제, 코드 난이도가 균형 트리보다 낮음  
**단점**: 확률적 구조(최악 O(n)), 포인터 오버헤드, 캐시 친화성 제한

**현업 적용**: Redis의 **sorted set**(점수 기반), 여러 LSM-tree 구현에서 **메모리 상의 정렬 구조** 등

---

## 3) XOR 연결 리스트 (XOR Linked List)

> 별도 글에서 완전 구현/반복자/테스트까지 제공했다. 여기선 **핵심 개념+요지 코드**만 복습.

### 3.1 핵심

이중 리스트의 `prev`와 `next` 대신 **하나의 필드** `npx = prev ⊕ next`.

- 이동:  
  \[
  \text{next} = \text{npx} \oplus \text{prev}
  \]
- 메모리 절감(노드당 포인터 1개 절약) vs **디버깅 난이도↑**, ASan/GC 등의 **호환성 문제**.

### 3.2 안전한 포인터 XOR 도우미

```cpp
#include <cstdint>

template <class T> inline T* pxor(T* a, T* b) noexcept {
    return reinterpret_cast<T*>(
        reinterpret_cast<std::uintptr_t>(a) ^
        reinterpret_cast<std::uintptr_t>(b)
    );
}

struct Node {
    int data;
    Node* npx;
    explicit Node(int v): data(v), npx(nullptr) {}
};

void push_front(Node*& head, int v){
    Node* n = new Node(v);
    n->npx = pxor<Node>(nullptr, head);
    if (head){
        Node* nxt = pxor<Node>(nullptr, head->npx);
        head->npx = pxor<Node>(n, nxt);
    }
    head = n;
}
```

**요약**: 학습/퍼즐용으로 훌륭, **실전은 비추천**(유지보수/도구 호환성/이득 대비 리스크)

---

## 4) 센티넬(Sentinel) 노드를 가진 연결 리스트

### 4.1 아이디어

헤드/테일에 **더미 노드**를 두어, 빈 리스트/양끝 삽입/삭제를 **중간 삽입과 동일한 로직**으로 처리.

**장점**: 엣지 케이스 감소 → **코드가 단순/견고**  
**적용**: `std::list`는 내부적으로 **원형+센티넬** 형태가 전형적

### 4.2 이중 리스트 템플릿(요약)

```cpp
template <class T>
class dlist {
    struct node { T data; node* prev; node* next; node():prev(this),next(this){} 
                  template<class...Args> node(Args&&...a):data(std::forward<Args>(a)...),prev(nullptr),next(nullptr){} };
    node* sent_; std::size_t sz_{};
public:
    dlist(){ sent_=new node(); } ~dlist(){ clear(); delete sent_; }
    bool empty() const { return sz_==0; }
    void clear(){ for(node* c=sent_->next; c!=sent_; ){ node* n=c->next; delete c; c=n; } sent_->next=sent_->prev=sent_; sz_=0; }

    // pos 앞 삽입
    template<class...Args> node* emplace_before(node* pos, Args&&... args){
        node* x=new node(std::forward<Args>(args)...);
        node* a=pos->prev; x->next=pos; x->prev=a; a->next=x; pos->prev=x; ++sz_; return x;
    }
    node* push_front(const T& v){ return emplace_before(sent_->next, v); }
    node* push_back (const T& v){ return emplace_before(sent_, v); }

    void erase(node* p){ if(p==sent_) return; p->prev->next=p->next; p->next->prev=p->prev; delete p; --sz_; }
    node* begin() const { return sent_->next; }
    node* end()   const { return sent_; }
};
```

**응용**: `splice/merge/reverse`도 포인터 교환만으로 깔끔하게 구현 가능(앞선 “연결 리스트” 글 참고)

---

## 5) Unrolled Linked List (분해/언롤드 리스트)

### 5.1 동기

연결 리스트의 **캐시 미스/상수항**을 줄이기 위해, **노드 하나가 작은 배열(버킷)**을 보유하게 한다.  
유사 구조로 **Python `deque`**, 텍스트 편집기(gap buffer/rope/조합) 등.

- 항목 N개, 버킷 용량 B → **노드 수 ≈ N/B**  
- 순회 시 **각 노드에서 B개를 연속 접근** → 캐시 우호, 분할/병합으로 균형 유지

### 5.2 최소 동작 구현 (삽입/삭제/분할/병합)

```cpp
#include <vector>
#include <cassert>

template <class T, int BLOCK=32>
class unrolled_list {
    struct block {
        int cnt=0;
        T data[BLOCK];
        block* prev=nullptr; block* next=nullptr;
    };
    block* head_=nullptr; block* tail_=nullptr; std::size_t sz_=0;

    void ensure_head(){
        if (!head_) head_=tail_=new block();
    }
    // 분할: 꽉 찬 블록을 두 개로 나눔
    void split(block* b){
        if (b->cnt < BLOCK) return;
        block* nb = new block();
        int move = b->cnt/2;
        for(int i=0;i<move;++i) nb->data[i]=b->data[b->cnt-move+i];
        nb->cnt = move; b->cnt -= move;
        // 연결
        nb->next=b->next; nb->prev=b;
        if (b->next) b->next->prev=nb; else tail_=nb;
        b->next=nb;
    }
    // 병합: 너무 비어 있으면 이웃과 합침
    void maybe_merge(block* b){
        if (!b) return;
        if (b->cnt > 0) return;
        // 완전 비면 제거
        if (b==head_ && b==tail_){ delete b; head_=tail_=nullptr; return; }
        if (b==head_){ head_=b->next; head_->prev=nullptr; delete b; return; }
        if (b==tail_){ tail_=b->prev; tail_->next=nullptr; delete b; return; }
        b->prev->next=b->next; b->next->prev=b->prev; delete b;
    }

public:
    ~unrolled_list(){ clear(); }
    void clear(){
        block* c=head_; while(c){ block* n=c->next; delete c; c=n; }
        head_=tail_=nullptr; sz_=0;
    }
    std::size_t size() const { return sz_; }
    bool empty() const { return sz_==0; }

    // i번째 위치 앞에 삽입 (0<=i<=size)
    void insert(std::size_t i, const T& x){
        assert(i<=sz_);
        ensure_head();
        block* b=head_; std::size_t ofs=i;
        while (b && ofs > (std::size_t)b->cnt){ ofs -= b->cnt; b=b->next; }
        if (!b){ b=tail_; ofs = b->cnt; } // 꼬리에 삽입
        // 공간 확보
        if (b->cnt==BLOCK){ split(b); if (ofs> (std::size_t)b->cnt){ ofs -= b->cnt; b=b->next; } }
        // 뒤로 밀기
        for(int k=b->cnt; k>(int)ofs; --k) b->data[k]=b->data[k-1];
        b->data[ofs]=x; ++b->cnt; ++sz_;
    }

    // i번째 원소 삭제
    void erase(std::size_t i){
        assert(i<sz_);
        block* b=head_; std::size_t ofs=i;
        while (b && ofs >= (std::size_t)b->cnt){ ofs -= b->cnt; b=b->next; }
        for(int k=(int)ofs; k<b->cnt-1; ++k) b->data[k]=b->data[k+1];
        --b->cnt; --sz_;
        // 병합 정책(간단 버전): 비면 제거
        if (b->cnt==0) maybe_merge(b);
    }

    // 순회(간단 출력)
    void print() const {
        for(block* b=head_; b; b=b->next){
            std::cout<<"[";
            for(int i=0;i<b->cnt;++i) std::cout<<b->data[i]<<(i+1<b->cnt?" ":"");
            std::cout<<"] ";
        }
        std::cout<<"\n";
    }
};
```

**복잡도**(평균):  
- `insert/erase`는 블록 내 이동 O(B) + 블록 탐색 O(#blocks) ≈ O(B + N/B)  
- 적절한 B(예: 32~128) 선택 시 **캐시 친화적** 성능 확보

**적용**: 텍스트 편집(줄/토큰 단위), **덱/버퍼**류, 큰 데이터 이동이 비싼 환경

---

## 6) Looping / Self-referential List (루프/자가참조)

### 6.1 개념

일반 원형 리스트와 달리, **일부만 루프를 형성**하거나 **자기 자신을 next로 가리키는 노드**가 존재하는 특수 상황.

- **테스트/실험용**: 루프 검출 알고리즘(Floyd, Brent) 검증
- **버그 재현**: 잘못된 연결로 생기는 루프를 찾아내기

### 6.2 루프 검출 (Floyd’s Cycle Detection)

```cpp
struct Node { int v; Node* next; explicit Node(int x): v(x), next(nullptr){} };

bool has_cycle(Node* head){
    Node* slow=head; Node* fast=head;
    while (fast && fast->next){
        slow=slow->next; fast=fast->next->next;
        if (slow==fast) return true;
    }
    return false;
}

// 루프 시작점 찾기
Node* cycle_entry(Node* head){
    Node* slow=head; Node* fast=head;
    while (fast && fast->next){
        slow=slow->next; fast=fast->next->next;
        if (slow==fast) break;
    }
    if (!fast || !fast->next) return nullptr; // no cycle
    slow=head;
    while (slow!=fast){ slow=slow->next; fast=fast->next; }
    return slow; // entry
}
```

**수학 스케치**:  
만남 지점에서 헤드로부터 entry까지 거리 \(a\), entry→만남까지 \(b\), 루프 길이 \(c\).  
빠른 포인터는 느린 포인터보다 2배 속도 → 만남 시  
\[
2(a+b) \equiv a+b \pmod{c} \Rightarrow a \equiv -b \pmod{c}
\]  
따라서 헤드와 만남 지점에서 **동속 전진**하면 entry에서 만난다.

---

## 7) (보너스) Intrusive Linked List 한 줄 개념

- **intrusive**: 데이터 객체 자체가 `prev/next`를 **내장**하여 별도 노드 할당이 없다.  
- 장점: **할당/해제 비용 제거**, 캐시 친화  
- 단점: 타입 결합, 다중 리스트에 같은 객체를 담기 어려움(필드 충돌)  
- Linux 커널의 `list_head` 매크로가 대표적

```c
// C 스타일 요약(커널 스타일)
struct list_head { struct list_head *prev, *next; };

#define INIT_LIST_HEAD(ptr) do { \
  (ptr)->next = (ptr); (ptr)->prev = (ptr); \
} while(0)

static inline void __list_add(struct list_head *n, struct list_head *prev, struct list_head *next){
  next->prev = n; n->next = next; n->prev = prev; prev->next = n;
}
```

---

## 8) 종합 비교표

| 구조 | 포인터 수(노드당) | 주 용도 | 평균 연산 특성 | 장점 | 단점 |
|---|---:|---|---|---|---|
| Multilevel(2D) | 2+ | 2D/희소 그리드 | 국소 수정 O(1)~O(k), 전체 순회 O(N) | 행/열 삽입 유리 | 인덱스 접근 느림 |
| Skip List | ~\(1/(1-P)\) | 정렬+검색 | 탐색/삽입/삭제 O(log n) | 구현 단순(트리 대비) | 확률 구조, 포인터 오버헤드 |
| XOR List | 1 | 이론/퍼즐 | 이중 리스트와 동일 | 포인터 1개 절약 | 디버깅/도구 호환 악몽 |
| Sentinel List | 2 | 경계 단순화 | 표준 리스트와 동일 | 빈/양끝 처리 간단 | 여전히 캐시 비우호 |
| Unrolled List | 1(블록 링크) | 캐시 최적/편집기 | O(B + N/B) | 캐시 친화, 공간 효율↑ | 분할/병합 로직 필요 |
| Looping/Self | 1 | 루프 검출 실험 | - | 테스트/교육용 | 실서비스엔 위험 |

---

## 9) 실전 미니 프로젝트

### 9.1 희소 스프레드시트(2D + Sentinel 행/열 Header)

- 행/열 header를 **센티넬 노드**로 두고, 각 행/열은 **정렬 삽입**.
- 셀 삭제 시 행/열에서 모두 제거.
- 장점: 빈 행/열 제거가 O(1) 재배선, 확장 용이.

**핵심 아이디어 스케치**

```cpp
struct Col;
struct Row {
    int idx;
    Row* up; Row* down;
    // 행 내 셀 연결(오름차순 col)
    struct Cell* right;
};

struct Col {
    int idx;
    Col* left; Col* right;
    struct Cell* down;
};

struct Cell {
    int r, c; int val;
    Cell* row_next; // 행내
    Cell* col_next; // 열내
};
```

- 삽입: (r,c) 위치의 행/열을 찾아 없으면 만들어 연결 → 행/열 리스트에 **정렬 삽입**  
- 삭제: 두 방향 링크에서 제거. header에 고아가 되면 header 재연결

### 9.2 LRU 캐시(센티넬 이중 리스트 + 해시)

- 앞선 연결 리스트 글의 LRU 예제를 센티넬 구조로 이식하면 양끝/빈 케이스 분기가 사라진다.
- `splice` 한 번으로 **O(1) 재배선**.

---

## 10) 테스트 전략 & 디버깅

- **퍼징**: 표준 컨테이너(`std::list`, `std::set`)와 동시 실행하여 결과 일치성 검증
- **불변식 체크**: 사이즈, 원형/센티넬 고리, 역방향 포인터 일치
- **ASan/UBSan**: 포인터 실수, 경계 초과 조기 발견
- **성능 실험**: `unrolled_list`는 B(버킷 크기) 변화에 따른 실제 성능 곡선을 측정

간단 퍼저 예(언롤드 vs `std::vector` 부분 비교):

```cpp
#include <vector>
#include <random>
#include <cassert>
int main(){
    unrolled_list<int, 32> ul;
    std::vector<int>      vr;
    std::mt19937 rng(123); std::uniform_int_distribution<int> op(0,1), val(0,1000);

    for(int t=0;t<20000;++t){
        int o=op(rng), x=val(rng);
        if (o==0){ // insert
            std::size_t i = vr.empty()?0: (val(rng)% (vr.size()+1));
            ul.insert(i, x); vr.insert(vr.begin()+i, x);
        } else if (!vr.empty()) {
            std::size_t i = val(rng)% vr.size();
            ul.erase(i); vr.erase(vr.begin()+i);
        }
    }
    // 값 비교 (단순 출력/스캔 작성 가능)
    // 실전엔 iterator 지원을 추가해 정확히 비교한다.
}
```

---

## 11) 수학/복잡도 스냅샷

- **스킵 리스트** 레벨 기대치:
  \[
  \mathbb{E}[L] \approx \log_{1/P} n \quad (\text{탐색/삽입/삭제 기대 } \Theta(\log n))
  \]
- **Unrolled** 최적 B 선택 직감:
  \[
  \text{비용} \approx c_1\cdot B + c_2\cdot \frac{N}{B}
  \Rightarrow B^\* \propto \sqrt{\frac{c_2 N}{c_1}}
  \]
  상수는 플랫폼 캐시/메모리 계층에 의존 → **실측**이 답.

---

## 12) 언제 무엇을 쓰는가? (의사결정 체크리스트)

- **정렬된 범위 + 빠른 탐색/경계**: 스킵 리스트 (또는 균형 트리)  
- **메모리 극한 절약 필요**: XOR (실전은 신중하게)  
- **양끝/빈 리스트/경계가 잦음**: 센티넬 리스트  
- **대용량 편집/캐시 친화**: Unrolled (또는 Rope/GAP Buffer)  
- **2D 구조/국소 삽입·삭제**: Multilevel(2D)  
- **루프 검출/실험**: Looping/Self-referential

---

## 13) 마무리

특수 연결 리스트들은 “**기본 리스트의 약점**(캐시·경계·탐색·공간)을 특정 전술로 보완”한 구조다.  
도입 전 **요구 사항을 수치화**(탐색 vs 삽입/삭제 비율, 캐시 미스, 메모리 상한, 코드 복잡도)하고,  
**표준 컨테이너 + 보조 기법(센티넬/스플라이스/맞춤 할당자)**로 충분한지 먼저 검토하자.  

필요 시 본 글의 **구현 스케치**를 바탕으로 프로젝트에 맞춰 **테스트/프로파일링/경계 강화**를 수행하면 된다.
