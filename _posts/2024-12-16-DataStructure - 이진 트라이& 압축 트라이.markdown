---
layout: post
title: Data Structure - 트라이
date: 2024-12-09 20:20:23 +0900
category: Data Structure
---
# 🧠 고급 Trie 구조 - 이진 Trie와 압축 Trie(Radix Trie)

---

## 📌 1. 이진 Trie (Bitwise Trie)

### ✅ 개념

이진 Trie는 **정수(Integer)**를 **이진수 비트 단위로 저장**하는 트라이입니다.

- 각 노드는 `0` 또는 `1`의 비트를 따라가며 구성됩니다.
- 주로 **최댓값 XOR 쌍**, **IP 라우팅**, **집합 간 관계** 등에 사용됩니다.

---

### 🛠️ 구조 정의 및 삽입 (C++)

```cpp
struct BinaryTrieNode {
    BinaryTrieNode* child[2];

    BinaryTrieNode() {
        child[0] = child[1] = nullptr;
    }
};

void insert(BinaryTrieNode* root, int num) {
    BinaryTrieNode* node = root;
    for (int i = 31; i >= 0; --i) {
        int bit = (num >> i) & 1;
        if (!node->child[bit])
            node->child[bit] = new BinaryTrieNode();
        node = node->child[bit];
    }
}
```

---

### 🔍 예제: 최대 XOR 쌍 찾기

#### 🎯 문제 설명

> **정수 배열 nums가 주어질 때, 두 수의 XOR 값 중 가장 큰 값을 반환**하라.

예:  
`nums = [3, 10, 5, 25, 2, 8]`  
최대 XOR은 `5 ^ 25 = 28` → **출력: 28**

---

#### 💡 아이디어 - 이진 트라이(Bitwise Trie) 이용

- 각 숫자를 32비트 이진수로 보고, 이진 트라이에 삽입
- 최댓값을 만들기 위해 가능한 한 **상반된 비트(0↔1)**를 선택하며 탐색

---

#### 🛠️ C++ 구현

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
using namespace std;

struct Node {
    Node* child[2];
    Node() {
        child[0] = child[1] = nullptr;
    }
};

void insert(Node* root, int num) {
    Node* cur = root;
    for (int i = 31; i >= 0; --i) {
        int bit = (num >> i) & 1;
        if (!cur->child[bit])
            cur->child[bit] = new Node();
        cur = cur->child[bit];
    }
}

int getMaxXOR(Node* root, int num) {
    Node* cur = root;
    int maxXor = 0;
    for (int i = 31; i >= 0; --i) {
        int bit = (num >> i) & 1;
        int opp = 1 - bit;
        if (cur->child[opp]) {
            maxXor |= (1 << i);
            cur = cur->child[opp];
        } else {
            cur = cur->child[bit];
        }
    }
    return maxXor;
}

int findMaximumXOR(vector<int>& nums) {
    Node* root = new Node();
    int result = 0;

    for (int num : nums)
        insert(root, num);

    for (int num : nums)
        result = max(result, getMaxXOR(root, num));

    return result;
}
```

---

#### ✅ 실행 예제

```cpp
int main() {
    vector<int> nums = {3, 10, 5, 25, 2, 8};
    cout << "Maximum XOR = " << findMaximumXOR(nums) << endl;
    return 0;
}
```

🔽 출력:

```
Maximum XOR = 28
```

---

#### 🧠 정리

| 항목 | 설명 |
|------|------|
| 시간 복잡도 | O(N * 32) |
| 공간 복잡도 | O(N * 32) |
| 핵심 전략 | XOR 최대화는 상반된 비트 선택 |

---

#### ✨ 이진 Trie 응용

| 문제 유형 | 설명 |
|-----------|------|
| XOR 최대쌍 | 최대 XOR 계산 (`Leetcode 421`) |
| 정수 집합 포함 검사 | 중복 비트 조합 확인 |
| IPv4 라우팅 | Prefix 기반 탐색 |

---

## 📌 2. 압축 Trie (Radix Trie)

### ✅ 개념

압축 Trie는 **불필요한 단일 경로를 압축**하여 공간을 절약하는 Trie입니다.

- 노드에 단일 문자가 아닌 **문자열 전체를 저장** 가능
- 삽입/탐색 시 문자열 비교가 필요
- **메모리 사용 감소**, **탐색 트리 깊이 축소**

---

### 🛠️ 구조 정의 (간단 버전)

```cpp
struct RadixNode {
    string label;
    bool isEnd;
    vector<RadixNode*> children;

    RadixNode(string s = "", bool end = false)
        : label(move(s)), isEnd(end) {}
};
```

---

### 🛠️ 핵심 삽입 로직 (요약)

```cpp
void insert(RadixNode* node, const string& word) {
    for (RadixNode* child : node->children) {
        int i = 0;
        while (i < child->label.size() && i < word.size() && child->label[i] == word[i])
            ++i;

        if (i == 0) continue;

        if (i < child->label.size()) {
            // 분할 노드 생성
            RadixNode* split = new RadixNode(child->label.substr(i), child->isEnd);
            split->children = move(child->children);
            child->label = child->label.substr(0, i);
            child->isEnd = false;
            child->children = { split };
        }

        insert(child, word.substr(i));
        return;
    }

    // 새로운 자식 노드 추가
    node->children.push_back(new RadixNode(word, true));
}
```

---

### 🔍 검색 함수

```cpp
bool search(RadixNode* node, const string& word) {
    for (RadixNode* child : node->children) {
        if (word.starts_with(child->label)) {
            if (word == child->label) return child->isEnd;
            return search(child, word.substr(child->label.size()));
        }
    }
    return false;
}
```

> 문자열 접두사 기반으로 label을 잘라가며 탐색합니다.

---

## 🎯 비교 요약

| 항목 | 이진 Trie | 압축 Trie |
|------|-----------|-----------|
| 단위 | 비트 (0,1) | 문자열 |
| 데이터 타입 | 정수 | 문자열 |
| 공간 효율 | 낮음 (최대 32레벨) | 높음 (압축됨) |
| 응용 | XOR, IP 탐색 | 사전, 문자열 압축 |
| 구현 난이도 | 낮음 | 높음 (문자열 분할 필요) |

---

## ✅ 마무리

- **이진 Trie**는 정수 기반 문제 (비트 연산)에 강력
- **압축 Trie**는 사전 구조를 보다 메모리 효율적으로 처리 가능
- 둘 모두 일반 Trie보다 특화된 목적에 적합
