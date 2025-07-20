---
layout: post
title: Data Structure - Skip List
date: 2024-12-09 22:20:23 +0900
category: Data Structure
---
# Skip List (스킵 리스트)

## 1. Skip List란?

**Skip List**는 **정렬된 연결 리스트에 계층 구조를 추가**하여 이진 탐색처럼 빠르게 탐색이 가능하도록 만든 자료구조입니다.

> 평균 성능이 `O(log n)`으로 **이진 탐색 트리(BST)**에 대응하며, 동적 배열이나 해시 테이블보다 구조적으로 더 단순할 수 있습니다.

---

## 2. 기본 개념

- 각 노드는 여러 수준(Level)의 포인터를 가짐
- 높이가 높은 노드는 **더 먼 곳까지 연결**되어 탐색 효율을 높임
- 무작위로 높이를 정하여 **균형 유지**

---

## 3. 구조 정의 (C++)

```cpp
#include <iostream>
#include <vector>
#include <cstdlib>
#include <ctime>

const int MAX_LEVEL = 4;
const float P = 0.5;

struct Node {
    int val;
    std::vector<Node*> forward;

    Node(int level, int val) : val(val), forward(level + 1, nullptr) {}
};
```

---

## 4. 높이 결정 함수

```cpp
int randomLevel() {
    int level = 0;
    while (((float)std::rand() / RAND_MAX) < P && level < MAX_LEVEL)
        level++;
    return level;
}
```

---

## 5. SkipList 클래스 정의

```cpp
class SkipList {
private:
    Node* head;
    int level;

public:
    SkipList() {
        level = 0;
        head = new Node(MAX_LEVEL, -1);
        std::srand(std::time(0));
    }

    void insert(int val);
    bool search(int val);
    void print();
};
```

---

## 6. 삽입 구현

```cpp
void SkipList::insert(int val) {
    std::vector<Node*> update(MAX_LEVEL + 1);
    Node* cur = head;

    for (int i = level; i >= 0; i--) {
        while (cur->forward[i] && cur->forward[i]->val < val)
            cur = cur->forward[i];
        update[i] = cur;
    }

    cur = cur->forward[0];

    if (!cur || cur->val != val) {
        int newLevel = randomLevel();
        if (newLevel > level) {
            for (int i = level + 1; i <= newLevel; i++)
                update[i] = head;
            level = newLevel;
        }

        Node* newNode = new Node(newLevel, val);
        for (int i = 0; i <= newLevel; i++) {
            newNode->forward[i] = update[i]->forward[i];
            update[i]->forward[i] = newNode;
        }
    }
}
```

---

## 7. 검색 구현

```cpp
bool SkipList::search(int val) {
    Node* cur = head;
    for (int i = level; i >= 0; i--) {
        while (cur->forward[i] && cur->forward[i]->val < val)
            cur = cur->forward[i];
    }
    cur = cur->forward[0];
    return cur && cur->val == val;
}
```

---

## 8. 출력 함수

```cpp
void SkipList::print() {
    for (int i = level; i >= 0; i--) {
        Node* cur = head->forward[i];
        std::cout << "Level " << i << ": ";
        while (cur) {
            std::cout << cur->val << " ";
            cur = cur->forward[i];
        }
        std::cout << "\n";
    }
}
```

---

## 9. 장단점 요약

| 장점 | 단점 |
|------|------|
| 구현이 BST보다 단순 | 랜덤성에 따라 성능 편차 존재 |
| 평균 `O(log n)` 삽입/삭제/탐색 | 메모리 사용량 ↑ (다중 포인터) |
| 정렬 + 탐색 + 순회에 모두 강함 | 균형 유지를 확률에 의존 |