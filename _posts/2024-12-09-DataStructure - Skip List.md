---
layout: post
title: Data Structure - Skip List
date: 2024-12-09 22:20:23 +0900
category: Data Structure
---
# Skip List (스킵 리스트)

## Skip List란?

**Skip List**는 정렬된 연결 리스트에 **여러 개의 층(level)**을 추가하여, 탐색·삽입·삭제를 평균적으로 빠르게 수행할 수 있도록 만든 자료구조입니다.

- 각 노드는 무작위로 결정된 **높이(level)**를 가지며, 높이가 높을수록 더 먼 노드를 건너뛸 수 있습니다.
- 상위 레벨은 **고속도로**처럼 멀리 점프하고, 하위 레벨은 **일반 도로**처럼 세세하게 이동합니다.
- 탐색, 삽입, 삭제의 평균 시간 복잡도는 **$$O(\log n)$$**입니다.
- 구현이 비교적 간단하고, 확률적 균형을 유지합니다.

---

## 왜 필요한가?

| 자료구조 | 장점 | 단점 |
|----------|------|------|
| **정렬된 배열** | 이진 탐색 $$O(\log n)$$ | 삽입/삭제 $$O(n)$$ (밀기) |
| **연결 리스트** | 삽입/삭제 $$O(1)$$ (위치 알 때) | 탐색 $$O(n)$$ |
| **균형 트리 (AVL, Red-Black)** | 모든 연산 $$O(\log n)$$ | 구현 복잡 (회전 등) |
| **Skip List** | 모든 연산 평균 $$O(\log n)$$, 구현 단순 | 최악의 경우 $$O(n)$$ (확률적으로 드묾) |

Skip List는 **균형 트리**의 복잡한 회전 없이도 무작위성을 이용해 비슷한 성능을 얻을 수 있는 실용적인 선택입니다.

---

## 구조: 여러 층의 연결 리스트

아래 그림은 Skip List의 개념을 보여줍니다.

```
Level 3:  head ----------------------------------------------> 70
Level 2:  head -------------------> 40 -------------------> 70
Level 1:  head ---------> 20 -------> 40 -------> 60 -----> 70
Level 0:  head -> 10 -> 20 -> 30 -> 40 -> 50 -> 60 -> 70 -> 80
```

- **Level 0**은 모든 노드를 포함하는 일반적인 정렬 연결 리스트입니다.
- **Level 1**은 일부 노드만 포함하며, Level 0보다 더 긴 간격으로 건너뜁니다.
- **Level 2, 3**은 더 적은 노드를 포함하며, 더 큰 도약을 합니다.

탐색은 가장 높은 레벨에서 시작하여, 목표 값을 넘지 않는 선에서 최대한 전진한 후, 한 레벨 내려와 다시 전진하는 방식으로 진행됩니다.

---

## 노드와 레벨

각 노드는 자신이 속한 모든 레벨의 **다음 노드 포인터**를 배열로 저장합니다.

```cpp
struct Node {
    int key;                       // 저장하는 값 (정수 셋 기준)
    std::vector<Node*> forward;    // forward[i] : 이 노드의 i-레벨 다음 노드
};
```

노드의 레벨(높이)는 **확률적으로** 결정됩니다.  
보통 **동전 던지기** 방식으로, 앞면이 나오면 레벨을 하나 높이고, 뒷면이 나오면 멈춥니다.

```cpp
int random_level() {
    int level = 0;
    while (level < MAX_LEVEL && (rand() % 2) == 0) {
        ++level;
    }
    return level;
}
```

- `MAX_LEVEL`은 미리 정해둔 최대 레벨 (보통 $$\lceil \log_2 N_{max} \rceil$$ 정도)
- 평균 레벨은 약 2 (앞면 확률 1/2일 때)

---

## 기본 연산

### 탐색 (Search)

탐색은 가장 높은 레벨에서 시작하여, 각 레벨에서 **목표보다 작은 마지막 노드**까지 전진한 후, 한 레벨 내려옵니다. 최종적으로 Level 0에서 목표 노드를 찾습니다.

```cpp
bool contains(int key) {
    Node* cur = head;
    for (int i = level; i >= 0; --i) {
        while (cur->forward[i] && cur->forward[i]->key < key) {
            cur = cur->forward[i];
        }
    }
    cur = cur->forward[0];  // Level 0의 다음 노드
    return (cur && cur->key == key);
}
```

### 삽입 (Insert)

삽입은 다음 단계로 이루어집니다.

1. 탐색 경로 저장: 삽입할 위치의 **각 레벨에서의 앞 노드**를 기억합니다. (이들을 `update` 배열에 저장)
2. 새 노드의 레벨 결정: `random_level()`로 레벨 생성.
3. 만약 새 노드의 레벨이 현재 최고 레벨보다 높다면, `update` 배열의 나머지를 `head`로 채우고 최고 레벨을 갱신.
4. 새 노드를 생성하고, 각 레벨에서 `update[i]->forward[i]`를 연결.

```cpp
void insert(int key) {
    std::vector<Node*> update(MAX_LEVEL + 1, nullptr);
    Node* cur = head;

    // 1. 삽입 위치 탐색 (각 레벨의 앞 노드를 update에 저장)
    for (int i = level; i >= 0; --i) {
        while (cur->forward[i] && cur->forward[i]->key < key) {
            cur = cur->forward[i];
        }
        update[i] = cur;
    }
    cur = cur->forward[0];

    if (cur && cur->key == key) {
        return;  // 중복 키 허용 안 함
    }

    // 2. 새 노드 레벨 결정
    int newLevel = random_level();

    // 3. 최고 레벨 갱신
    if (newLevel > level) {
        for (int i = level + 1; i <= newLevel; ++i) {
            update[i] = head;
        }
        level = newLevel;
    }

    // 4. 새 노드 생성
    Node* newNode = new Node(key, newLevel);

    // 5. 각 레벨에서 연결
    for (int i = 0; i <= newLevel; ++i) {
        newNode->forward[i] = update[i]->forward[i];
        update[i]->forward[i] = newNode;
    }
}
```

### 삭제 (Erase)

삭제도 비슷한 방식으로 진행됩니다.

1. 탐색 경로 저장 (각 레벨에서의 앞 노드)
2. 삭제할 노드가 존재하는지 확인
3. 각 레벨에서 해당 노드를 가리키는 포인터를 다음 노드로 변경
4. 노드 삭제
5. 필요시 최고 레벨 조정 (가장 높은 레벨에 노드가 없으면 레벨 낮춤)

```cpp
bool erase(int key) {
    std::vector<Node*> update(MAX_LEVEL + 1, nullptr);
    Node* cur = head;

    for (int i = level; i >= 0; --i) {
        while (cur->forward[i] && cur->forward[i]->key < key) {
            cur = cur->forward[i];
        }
        update[i] = cur;
    }
    cur = cur->forward[0];

    if (!cur || cur->key != key) {
        return false;
    }

    // 포인터 재연결
    for (int i = 0; i <= level; ++i) {
        if (update[i]->forward[i] == cur) {
            update[i]->forward[i] = cur->forward[i];
        }
    }

    delete cur;

    // 최고 레벨 조정
    while (level > 0 && head->forward[level] == nullptr) {
        --level;
    }
    return true;
}
```

---

## 복잡도

- **탐색, 삽입, 삭제**: 평균 $$O(\log n)$$, 최악 $$O(n)$$
  - 최악의 경우는 모든 노드의 레벨이 0일 때 (일반 연결 리스트) → 확률이 매우 낮음.
- **공간**: 평균적으로 각 노드는 $$\frac{1}{1-P}$$ 개의 포인터를 가짐 (P=0.5일 때 평균 2개). 총 포인터 수는 약 $$n \times \frac{1}{1-P}$$.

---

## 간단한 구현 예제 (정수 셋)

아래는 위에서 설명한 내용을 하나의 코드로 모은 예제입니다. (C++17 기준)

```cpp
#include <iostream>
#include <vector>
#include <cstdlib>
#include <ctime>

const int MAX_LEVEL = 16;  // 적당히 큰 값 (2^16 정도)

struct Node {
    int key;
    std::vector<Node*> forward;

    Node(int k, int level) : key(k), forward(level + 1, nullptr) {}
};

class SkipList {
private:
    Node* head;
    int level;          // 현재 최고 레벨
    double probability; // 레벨 상승 확률 (보통 0.5)

    int random_level() {
        int lvl = 0;
        while (lvl < MAX_LEVEL && (rand() / (double)RAND_MAX) < probability) {
            ++lvl;
        }
        return lvl;
    }

public:
    SkipList(double p = 0.5) : level(0), probability(p) {
        head = new Node(-1, MAX_LEVEL);  // 헤더는 더미 키 (모든 레벨의 시작)
        std::srand(std::time(nullptr));
    }

    ~SkipList() {
        Node* cur = head->forward[0];
        while (cur) {
            Node* next = cur->forward[0];
            delete cur;
            cur = next;
        }
        delete head;
    }

    void insert(int key) {
        std::vector<Node*> update(MAX_LEVEL + 1, nullptr);
        Node* cur = head;

        for (int i = level; i >= 0; --i) {
            while (cur->forward[i] && cur->forward[i]->key < key) {
                cur = cur->forward[i];
            }
            update[i] = cur;
        }
        cur = cur->forward[0];

        if (cur && cur->key == key) {
            return; // 중복 무시
        }

        int newLevel = random_level();

        if (newLevel > level) {
            for (int i = level + 1; i <= newLevel; ++i) {
                update[i] = head;
            }
            level = newLevel;
        }

        Node* newNode = new Node(key, newLevel);

        for (int i = 0; i <= newLevel; ++i) {
            newNode->forward[i] = update[i]->forward[i];
            update[i]->forward[i] = newNode;
        }
    }

    bool contains(int key) {
        Node* cur = head;
        for (int i = level; i >= 0; --i) {
            while (cur->forward[i] && cur->forward[i]->key < key) {
                cur = cur->forward[i];
            }
        }
        cur = cur->forward[0];
        return (cur && cur->key == key);
    }

    bool erase(int key) {
        std::vector<Node*> update(MAX_LEVEL + 1, nullptr);
        Node* cur = head;

        for (int i = level; i >= 0; --i) {
            while (cur->forward[i] && cur->forward[i]->key < key) {
                cur = cur->forward[i];
            }
            update[i] = cur;
        }
        cur = cur->forward[0];

        if (!cur || cur->key != key) {
            return false;
        }

        for (int i = 0; i <= level; ++i) {
            if (update[i]->forward[i] == cur) {
                update[i]->forward[i] = cur->forward[i];
            }
        }

        delete cur;

        while (level > 0 && head->forward[level] == nullptr) {
            --level;
        }
        return true;
    }

    void print() {
        for (int i = level; i >= 0; --i) {
            std::cout << "Level " << i << ": ";
            Node* cur = head->forward[i];
            while (cur) {
                std::cout << cur->key << " ";
                cur = cur->forward[i];
            }
            std::cout << std::endl;
        }
    }
};

int main() {
    SkipList sl;

    sl.insert(10);
    sl.insert(30);
    sl.insert(20);
    sl.insert(5);
    sl.insert(15);

    std::cout << "Skip List after inserts:" << std::endl;
    sl.print();

    std::cout << "Contains 20? " << sl.contains(20) << std::endl;
    std::cout << "Contains 25? " << sl.contains(25) << std::endl;

    sl.erase(20);
    std::cout << "After erasing 20:" << std::endl;
    sl.print();

    return 0;
}
```

**실행 예시:**
```
Skip List after inserts:
Level 2: 10 30 
Level 1: 5 10 15 20 30 
Level 0: 5 10 15 20 30 
Contains 20? 1
Contains 25? 0
After erasing 20:
Level 2: 10 30 
Level 1: 5 10 15 30 
Level 0: 5 10 15 30 
```

---

## 템플릿 버전과 맵으로 확장

위 코드는 정수 키만 다루지만, 템플릿을 사용하면 다양한 타입을 지원할 수 있습니다.  
또한, 값을 저장하는 맵으로 확장하려면 노드에 `value`를 추가하고, `std::pair<const Key, Value>`를 저장하는 식으로 변경하면 됩니다.

C++ 표준 라이브러리에는 없지만, 많은 라이브러리에서 Skip List를 구현하여 제공합니다. (예: `boost::intrusive::skip_list`)

---

## 마무리

Skip List는 확률적 균형을 이용해 트리보다 단순한 구조로 로그 시간 연산을 제공하는 매력적인 자료구조입니다.

- **장점**: 구현이 비교적 쉽고, 삽입/삭제 시 복잡한 회전이 필요 없습니다.
- **단점**: 최악의 경우 성능이 나빠질 수 있으나, 실용적으로는 거의 발생하지 않습니다.

초보자라면 먼저 위의 간단한 구현을 직접 작성해보고, 다양한 연산을 테스트해보며 동작 원리를 익히는 것을 추천합니다.