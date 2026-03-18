---
layout: post
title: Data Structure - 연결 리스트 고급
date: 2024-12-09 20:20:23 +0900
category: Data Structure
---
# 특수한 연결 리스트들

## 왜 다양한 연결 리스트가 필요한가?

기본 연결 리스트(단일/이중)는 중간 삽입/삭제가 빠르지만, 탐색이 느리고 캐시 효율이 낮습니다.  
상황에 따라 이런 단점을 보완하거나 특수한 목적을 위해 다양한 변형이 등장했습니다.

- **다중 연결 리스트**: 2차원 구조나 여러 방향 탐색이 필요할 때
- **스킵 리스트**: 탐색 속도를 \(O(\log n)\)으로 높이고 싶을 때
- **XOR 리스트**: 메모리를 절약하고 싶을 때 (하지만 위험)
- **센티넬 리스트**: 경계 조건 처리를 단순화할 때
- **Unrolled 리스트**: 캐시 효율을 높이고 싶을 때
- **순환 탐지 리스트**: 사이클 검출 실험용

---

## 다중 연결 리스트 (Multi‑level Linked List)

### 개념

하나의 노드가 여러 개의 포인터(예: `right`, `down`)를 가져서 2차원 이상의 연결 구조를 만듭니다.

```
[data] -> [data] -> [data]   (right 방향)
  |         |         |
  v         v         v
[data] -> [data] -> [data]   (down 방향)
```

- **장점**: 희소 행렬, 게임 맵 등에서 행/열 삽입/삭제가 빠릅니다. (포인터만 변경)
- **단점**: 구현이 복잡하고, 임의 접근은 느립니다.

### 간단한 2차원 격자 구현 (핵심만)

```cpp
#include <iostream>
#include <vector>

struct Cell {
    int data;
    Cell* right;
    Cell* down;
    explicit Cell(int val) : data(val), right(nullptr), down(nullptr) {}
};

// rows x cols 크기의 격자 생성 (오른쪽, 아래로 연결)
Cell* createGrid(int rows, int cols) {
    std::vector<std::vector<Cell*>> grid(rows, std::vector<Cell*>(cols));
    for (int i = 0; i < rows; ++i)
        for (int j = 0; j < cols; ++j)
            grid[i][j] = new Cell(i * cols + j);

    for (int i = 0; i < rows; ++i) {
        for (int j = 0; j < cols; ++j) {
            if (j + 1 < cols) grid[i][j]->right = grid[i][j + 1];
            if (i + 1 < rows) grid[i][j]->down  = grid[i + 1][j];
        }
    }
    return grid[0][0]; // 맨 왼쪽 위 노드
}

// 첫 번째 행 출력
void printFirstRow(Cell* head) {
    for (Cell* cur = head; cur; cur = cur->right)
        std::cout << cur->data << " ";
    std::cout << "\n";
}
```

**활용 예**: 스프레드시트에서 행/열 삽입이 빈번할 때 유용합니다.

---

## 스킵 리스트 (Skip List)

### 아이디어

정렬된 연결 리스트 위에 **여러 층의 고속도로**를 만듭니다. 위층일수록 노드가 드물게 배치되어 멀리 점프할 수 있습니다.

```
Level 2: head ------------------------------> 70
Level 1: head ---------> 40 ---------------> 70
Level 0: head -> 10 -> 20 -> 30 -> 40 -> 50 -> 70
```

- 탐색은 최상위 레벨에서 시작하여 목표 값을 넘지 않는 선에서 전진, 아래층으로 내려오며 반복합니다.
- 각 노드의 레벨은 확률적으로 결정(보통 0.5 확률로 한 층 더 승격). 평균 높이는 \( \log_2 n \) 정도.
- 삽입, 삭제, 탐색 모두 평균 \(O(\log n)\)입니다.

### 핵심 구현 (정수 저장)

```cpp
#include <vector>
#include <random>

struct Node {
    int key;
    std::vector<Node*> next;  // next[i] : i-레벨에서 다음 노드
    Node(int k, int level) : key(k), next(level + 1, nullptr) {}
};

class SkipList {
    Node* head;
    int level;          // 현재 최고 레벨
    int maxLevel;
    double prob;
    std::mt19937 rng;
    std::uniform_real_distribution<double> dist;

    int randomLevel() {
        int lvl = 0;
        while (lvl < maxLevel && dist(rng) < prob) ++lvl;
        return lvl;
    }

public:
    SkipList(int maxL = 32, double p = 0.5)
        : level(0), maxLevel(maxL), prob(p), dist(0, 1), rng(std::random_device{}()) {
        head = new Node(0, maxLevel); // 더미 헤드
    }

    ~SkipList() { clear(); delete head; }

    void insert(int key) {
        std::vector<Node*> update(maxLevel + 1, nullptr);
        Node* cur = head;
        for (int i = level; i >= 0; --i) {
            while (cur->next[i] && cur->next[i]->key < key)
                cur = cur->next[i];
            update[i] = cur;
        }
        cur = cur->next[0];
        if (cur && cur->key == key) return; // 중복 방지

        int newLevel = randomLevel();
        if (newLevel > level) {
            for (int i = level + 1; i <= newLevel; ++i)
                update[i] = head;
            level = newLevel;
        }
        Node* newNode = new Node(key, newLevel);
        for (int i = 0; i <= newLevel; ++i) {
            newNode->next[i] = update[i]->next[i];
            update[i]->next[i] = newNode;
        }
    }

    bool contains(int key) {
        Node* cur = head;
        for (int i = level; i >= 0; --i) {
            while (cur->next[i] && cur->next[i]->key < key)
                cur = cur->next[i];
        }
        cur = cur->next[0];
        return (cur && cur->key == key);
    }

    void clear() {
        Node* cur = head->next[0];
        while (cur) {
            Node* tmp = cur->next[0];
            delete cur;
            cur = tmp;
        }
        for (int i = 0; i <= maxLevel; ++i) head->next[i] = nullptr;
        level = 0;
    }
};
```

**주의**: 스킵 리스트는 구현이 비교적 간단하지만, 최악의 경우 \(O(n)\)이 될 수 있습니다(매우 드묾). Redis의 Sorted Set 등에서 사용됩니다.

---

## XOR 연결 리스트 (XOR Linked List)

### 원리

이중 연결 리스트에서 `prev`와 `next` 포인터 대신 **두 포인터의 XOR 값** 하나만 저장합니다.

\[
\text{npx} = \text{prev} \oplus \text{next}
\]

다음 노드 주소는 이전 노드를 알고 있을 때 계산합니다.

\[
\text{next} = \text{npx} \oplus \text{prev}
\]

- **장점**: 노드당 포인터 1개만 사용 → 메모리 절약
- **단점**:
  - 코드가 복잡하고 이해하기 어려움
  - 디버깅이 거의 불가능
  - 가비지 컬렉터, 주소 공간 무작위화(ASLR) 등과 충돌 가능
  - 실무에서 거의 사용되지 않음 (학습용으로만)

### 간단한 앞쪽 삽입 예시

```cpp
#include <cstdint>

template <typename T>
T* xor_ptr(T* a, T* b) {
    return reinterpret_cast<T*>(
        reinterpret_cast<std::uintptr_t>(a) ^
        reinterpret_cast<std::uintptr_t>(b)
    );
}

struct Node {
    int data;
    Node* npx;
    explicit Node(int val) : data(val), npx(nullptr) {}
};

void push_front(Node*& head, int val) {
    Node* newNode = new Node(val);
    newNode->npx = xor_ptr<Node>(nullptr, head);
    if (head) {
        Node* next = xor_ptr<Node>(nullptr, head->npx); // head의 다음 노드
        head->npx = xor_ptr<Node>(newNode, next);
    }
    head = newNode;
}
```

**결론**: 이런 기법이 있다는 것만 알고, 실제 프로젝트에서는 절대 쓰지 않는 것이 좋습니다.

---

## 센티넬 연결 리스트 (Sentinel Linked List)

### 개념

헤드와 테일에 실제 데이터를 가지지 않는 **더미 노드(sentinel)**를 둡니다.

```
head(더미) <-> [data] <-> [data] <-> [data] <-> tail(더미)
```

- 빈 리스트일 때도 head와 tail이 서로 연결되어 있어, 경계 조건(맨 앞/맨 뒤 삽입/삭제)을 일반 로직과 동일하게 처리할 수 있습니다.
- 코드가 단순해지고 오류 가능성이 줄어듭니다.
- `std::list`도 내부적으로 원형 + 센티넬 구조를 사용합니다.

### 이중 연결 리스트 센티넬 버전 (핵심)

```cpp
template <typename T>
class List {
    struct Node {
        T data;
        Node* prev;
        Node* next;
        Node() : prev(this), next(this) {} // 더미용
        Node(const T& val) : data(val), prev(nullptr), next(nullptr) {}
    };
    Node* sentinel; // head이자 tail 역할
    size_t sz;

public:
    List() : sentinel(new Node()), sz(0) {}
    ~List() { clear(); delete sentinel; }

    void push_front(const T& val) {
        Node* newNode = new Node(val);
        Node* first = sentinel->next;
        // sentinel <-> newNode <-> first
        sentinel->next = newNode;
        newNode->prev = sentinel;
        newNode->next = first;
        first->prev = newNode;
        ++sz;
    }

    void push_back(const T& val) {
        Node* newNode = new Node(val);
        Node* last = sentinel->prev;
        last->next = newNode;
        newNode->prev = last;
        newNode->next = sentinel;
        sentinel->prev = newNode;
        ++sz;
    }

    // ... 삭제, 순회 등
};
```

센티넬 덕분에 빈 리스트에서도 `push_front`가 문제없이 동작합니다.

---

## Unrolled 연결 리스트 (Unrolled Linked List)

### 동기

연결 리스트의 가장 큰 단점은 **캐시 비효율**입니다. 노드가 메모리 곳곳에 흩어져 있어 순회할 때마다 캐시 미스가 발생합니다.  
Unrolled 리스트는 **각 노드가 작은 배열(블록)**을 가지게 하여 한 번에 여러 데이터를 연속적으로 접근하게 만듭니다.

```
[블록1 | 4개] -> [블록2 | 3개] -> [블록3 | 4개] ...
```

- 블록 크기 \(B\)를 적절히 선택(예: 32~128)하면 순회 시 캐시 히트율이 높아집니다.
- 삽입/삭제 시 블록 내에서 데이터를 이동하거나, 블록이 가득 차면 분할, 너무 비면 병합하는 로직이 필요합니다.

### 핵심 구현 (간략)

```cpp
#include <vector>
#include <cassert>

template <typename T, int BLOCK = 32>
class UnrolledList {
    struct Block {
        T data[BLOCK];
        int count = 0;
        Block* prev = nullptr;
        Block* next = nullptr;
    };
    Block* head = nullptr;
    Block* tail = nullptr;
    size_t total = 0;

    void split(Block* b) {
        // 블록이 가득 찼을 때 반으로 나눔
        if (b->count < BLOCK) return;
        Block* nb = new Block;
        int move = b->count / 2;
        for (int i = 0; i < move; ++i)
            nb->data[i] = b->data[b->count - move + i];
        nb->count = move;
        b->count -= move;
        // 연결
        nb->next = b->next;
        if (b->next) b->next->prev = nb;
        else tail = nb;
        b->next = nb;
        nb->prev = b;
    }

public:
    ~UnrolledList() { clear(); }
    void clear() {
        Block* cur = head;
        while (cur) { Block* nxt = cur->next; delete cur; cur = nxt; }
        head = tail = nullptr;
        total = 0;
    }

    void insert(size_t pos, const T& val) {
        assert(pos <= total);
        if (!head) {
            head = tail = new Block;
            head->data[0] = val;
            head->count = 1;
            total = 1;
            return;
        }
        // pos에 해당하는 블록과 블록 내 위치 찾기
        Block* b = head;
        size_t offset = pos;
        while (b && offset > (size_t)b->count) {
            offset -= b->count;
            b = b->next;
        }
        if (!b) { // 맨 끝
            tail->data[tail->count++] = val;
            ++total;
            return;
        }
        // 블록 내 offset 위치에 삽입 (밀기)
        for (int i = b->count; i > (int)offset; --i)
            b->data[i] = b->data[i - 1];
        b->data[offset] = val;
        ++b->count;
        ++total;
        if (b->count == BLOCK) split(b);
    }

    // 삭제, 순회 등 생략
};
```

Unrolled 리스트는 텍스트 편집기, 대용량 버퍼 등에서 사용됩니다. C++의 `std::deque`도 유사한 블록 구조를 가집니다.

---

## 순환 검출 리스트 (Cycle Detection)

### 개념

연결 리스트에 **사이클(순환)**이 있는지 검사하는 알고리즘입니다. 유명한 방법으로 **Floyd의 토끼와 거북이** 알고리즘이 있습니다.

- 느린 포인터(slow)는 한 칸씩, 빠른 포인터(fast)는 두 칸씩 이동.
- 만나면 사이클이 존재, 만나지 않고 빠른 포인터가 끝에 도달하면 사이클 없음.

### 구현

```cpp
struct Node {
    int data;
    Node* next;
    Node(int v) : data(v), next(nullptr) {}
};

bool hasCycle(Node* head) {
    Node* slow = head;
    Node* fast = head;
    while (fast && fast->next) {
        slow = slow->next;
        fast = fast->next->next;
        if (slow == fast) return true;
    }
    return false;
}

// 사이클 시작점 찾기
Node* cycleStart(Node* head) {
    Node* slow = head;
    Node* fast = head;
    while (fast && fast->next) {
        slow = slow->next;
        fast = fast->next->next;
        if (slow == fast) break;
    }
    if (!fast || !fast->next) return nullptr; // no cycle
    slow = head;
    while (slow != fast) {
        slow = slow->next;
        fast = fast->next;
    }
    return slow;
}
```

**수학적 배경**:  
리스트 시작부터 사이클 시작점까지 거리 \(a\), 사이클 시작점부터 만난 지점까지 \(b\), 사이클 길이 \(c\)라 할 때, 만난 지점에서 출발점과 동시에 한 칸씩 이동하면 시작점에서 만납니다.

---

## 정리

| 리스트 종류      | 핵심 아이디어                              | 장점                         | 단점                              | 실무 사용 |
|------------------|--------------------------------------------|------------------------------|-----------------------------------|-----------|
| 다중 연결        | 여러 방향 포인터                           | 2차원 조작 용이              | 복잡, 인덱스 접근 느림            | 특수 상황 |
| 스킵 리스트      | 다층 구조로 탐색 가속                      | 평균 \(O(\log n)\) 탐색      | 확률적, 포인터 많음               | Redis 등  |
| XOR 리스트       | prev ⊕ next 저장                           | 메모리 절약                  | 디버깅 불가, 위험                 | 거의 없음 |
| 센티넬 리스트    | 더미 노드로 경계 단순화                    | 코드 단순, 안전              | 약간의 메모리 오버헤드            | 표준 라이브러리 |
| Unrolled 리스트  | 블록 단위 저장으로 캐시 효율↑              | 순회 빠름                    | 분할/병합 로직 필요               | `std::deque` |
| 사이클 검출      | Floyd 알고리즘                             | 사이클 유무 판별             | -                                 | 디버깅/테스트 |

---

## 마무리

특수 연결 리스트들은 각자의 장단점을 가지고 특정 문제를 해결하기 위해 설계되었습니다.  
초중급 단계에서는 **센티넬 리스트**(코드 단순화)와 **스킵 리스트**(탐색 최적화) 정도를 이해하고, 필요할 때 나머지 개념을 참고하면 됩니다.

실무에서는 대부분 표준 컨테이너(`std::list`, `std::deque`, `std::set` 등)를 사용하지만, 내부 동작 원리를 알면 더 효율적인 코드를 작성하는 데 도움이 됩니다.