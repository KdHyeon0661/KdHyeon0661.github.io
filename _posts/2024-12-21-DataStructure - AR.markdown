---
layout: post
title: Data Structure - 펜윅 트리
date: 2024-12-09 19:20:23 +0900
category: Data Structure
---
# ⚡ 고성능 문자열 자료구조: Adaptive Radix Tree (ART) & HAT-Trie

---

## 📌 개요

많은 문자열 탐색/저장 알고리즘에서 속도와 메모리 효율 사이의 **트레이드오프**가 존재합니다.  
**ART**와 **HAT-Trie**는 이 문제를 해결하기 위해 고안된 **고성능 문자열 기반 트리**입니다.

---

## 🌳 1. Adaptive Radix Tree (ART)

### 🔍 정의

> ART는 "Trie"와 "Radix Tree(압축 트라이)"를 결합한 자료구조로,  
> 노드의 자식 수에 따라 내부 구조를 **적응형으로 동적으로 바꾸는** 특성이 있습니다.

- 공간 효율성과 탐색 성능의 균형
- 많은 키를 가진 트라이의 **성능 저하**를 방지
- **CPU 캐시 친화적** 구조

---

### 🧩 ART 노드 종류

| 노드 유형 | 설명 | 최대 자식 수 |
|-----------|------|---------------|
| Node4     | 배열 4개 | 4 |
| Node16    | 배열 16개 | 16 |
| Node48    | 인덱스 + 포인터 배열 | 48 |
| Node256   | 256개 포인터 배열 | 256 |

> 노드들은 키 개수에 따라 자동으로 업그레이드/다운그레이드 됩니다.

---

### 🔁 ART 주요 연산

#### 🔍 탐색 (Search)

- 각 문자마다 트리 내려감
- 자식 수에 따라 `Node4`, `Node16`, ... 등에서 **선형 or 이진 탐색**

#### ➕ 삽입 (Insert)

- 존재하지 않는 경로라면 노드 생성
- 자식이 꽉 차면 자동으로 **더 큰 타입으로 승급**

#### ❌ 삭제 (Delete)

- 자식 수가 줄어들면 자동으로 **작은 타입으로 다운그레이드**

---

### 예제코드
```cpp
#include <iostream>
#include <map>
#include <vector>
using namespace std;

struct ARTNode {
    map<char, ARTNode*> children;
    bool isEnd = false;

    ~ARTNode() {
        for (auto& [_, child] : children)
            delete child;
    }
};

class ART {
    ARTNode* root;

public:
    ART() { root = new ARTNode(); }
    ~ART() { delete root; }

    void insert(const string& word) {
        ARTNode* node = root;
        for (char c : word) {
            if (!node->children.count(c))
                node->children[c] = new ARTNode();
            node = node->children[c];
        }
        node->isEnd = true;
    }

    bool search(const string& word) {
        ARTNode* node = root;
        for (char c : word) {
            if (!node->children.count(c)) return false;
            node = node->children[c];
        }
        return node->isEnd;
    }

    bool remove(const string& word) {
        return remove(root, word, 0);
    }

private:
    bool remove(ARTNode* node, const string& word, int depth) {
        if (!node) return false;

        if (depth == word.size()) {
            if (!node->isEnd) return false;
            node->isEnd = false;
            return node->children.empty();
        }

        char c = word[depth];
        if (!node->children.count(c)) return false;

        bool shouldDelete = remove(node->children[c], word, depth + 1);
        if (shouldDelete) {
            delete node->children[c];
            node->children.erase(c);
            return !node->isEnd && node->children.empty();
        }
        return false;
    }
};

int main() {
    ART art;
    art.insert("apple");
    art.insert("apricot");
    cout << "ART search apricot: " << art.search("apricot") << endl;
    art.remove("apricot");
    cout << "ART search apricot after delete: " << art.search("apricot") << endl;

    return 0;
}

```

### 🧠 ART의 특징 요약

| 항목 | 내용 |
|------|------|
| 구조 | 압축된 Prefix Tree |
| 노드 전환 | Node4 ↔ Node16 ↔ Node48 ↔ Node256 |
| 탐색 시간 | O(k), 캐시 효율 높음 |
| 메모리 사용 | 매우 효율적 |
| 문자열 키 정렬 | 지원 (범위 질의 가능) |
| 실제 사용 | MariaDB, Redis, LevelDB 등

---

## 🌿 2. HAT-Trie (Hash-Array-Mapped Trie)

### 🔍 정의

> HAT-Trie는 **Trie**와 **Hash Table**을 결합한 구조로,  
> 긴 문자열을 효율적으로 저장하면서도 탐색, 삽입 속도를 높입니다.

---

### ⚙️ 구조 개요

1. **Trie 노드**
   - 각 레벨에서 공통 prefix를 저장

2. **Bucket 노드 (Hash Table)**
   - 일정 prefix까지 탐색하면 이후는 Hash Table로 관리
   - 문자열이 많아지면 Bucket → Trie로 다시 분리

---

### 🧩 주요 특징

| 항목 | 설명 |
|------|------|
| Trie + Hash hybrid | 앞부분은 Trie, 뒷부분은 Hash Table |
| Bucket Split | 일정 임계치 초과 시 Bucket → Trie 변환 |
| Insert | O(k), 매우 빠름 |
| Ordered 지원 | 문자열 사전 순서 가능 |
| 캐시 친화성 | 매우 높음 (한 Bucket에 여러 문자열 밀집)

---

### 예제코드
#include <iostream>
#include <map>
#include <vector>
#include <unordered_map>
using namespace std;

class HATTrieNode {
public:
    unordered_map<string, bool> bucket; // using map to simulate a hash bucket
    unordered_map<char, HATTrieNode*> children;
    bool isBucket = false;

    ~HATTrieNode() {
        for (auto& [_, child] : children)
            delete child;
    }
};

class HATTrie {
    HATTrieNode* root;
    static const int BUCKET_SIZE = 3; // arbitrary split threshold

public:
    HATTrie() { root = new HATTrieNode(); }
    ~HATTrie() { delete root; }

    void insert(const string& word) {
        HATTrieNode* node = root;
        string prefix;

        for (char c : word) {
            if (node->isBucket) {
                node->bucket[word] = true;
                if (node->bucket.size() > BUCKET_SIZE)
                    split(node);
                return;
            }

            prefix += c;
            if (!node->children.count(c))
                node->children[c] = new HATTrieNode();
            node = node->children[c];
        }

        node->isBucket = true;
        node->bucket[word] = true;
    }

    bool search(const string& word) {
        HATTrieNode* node = root;
        for (char c : word) {
            if (node->isBucket)
                return node->bucket.count(word) > 0;
            if (!node->children.count(c))
                return false;
            node = node->children[c];
        }
        return node->isBucket && node->bucket.count(word);
    }

private:
    void split(HATTrieNode* node) {
        for (auto& [word, _] : node->bucket) {
            HATTrieNode* cur = node;
            for (char c : word) {
                if (!cur->children.count(c))
                    cur->children[c] = new HATTrieNode();
                cur = cur->children[c];
            }
            cur->isBucket = true;
            cur->bucket[word] = true;
        }
        node->isBucket = false;
        node->bucket.clear();
    }
};

int main() {
    HATTrie hat;
    hat.insert("apple");
    hat.insert("apricot");
    hat.insert("apricon");
    hat.insert("apromax"); // should trigger split
    cout << "HAT search apricot: " << hat.search("apricot") << endl;
    cout << "HAT search apple: " << hat.search("apple") << endl;
    cout << "HAT search apromax: " << hat.search("apromax") << endl;

    return 0;
}

---

### ✅ 장점 vs 단점

| 장점 | 단점 |
|------|------|
| 매우 빠른 삽입/탐색 속도 | 구현 복잡도 |
| 캐시 효율 좋음 | Bucket 튜닝 필요 |
| 사전 순 정렬, 접두사 검색 가능 | 일부 케이스에선 Trie보다 느릴 수 있음 |

---

## 🔍 ART vs HAT-Trie 비교

| 항목 | Adaptive Radix Tree | HAT-Trie |
|------|----------------------|-----------|
| 구조 | 압축 Trie + 적응형 노드 | Trie + HashTable |
| 노드 분기 | 동적 업/다운그레이드 | Bucket split |
| 탐색 속도 | 매우 빠름 (캐시 최적화) | 빠름 (Bucket 효과) |
| 정렬 지원 | 사전 순 탐색 가능 | 사전 순 탐색 가능 |
| 메모리 효율 | 높음 | 높음 |
| 구현 난이도 | 높음 | 매우 높음 |
| 사용 예 | MariaDB, Redis | 일부 검색 엔진, 논문 등 |

---

## 🛠️ 예제 시각화 (개념)

### 🔤 ART 삽입 예

```text
Insert: "abc", "abd", "abe", "xyz"

Node4 ("a") --> Node4 ("b") --> Node4 ("c"/"d"/"e")
           \
            -> ("x") -> ("y") -> ("z")
```

→ 자식 노드 개수 증가 시 `Node4` → `Node16`으로 변경됨

---

### 🧪 HAT-Trie 예

```text
Insert: "apple", "april", "apply", "banana"

Trie:
  a
   └── p
        └── Bucket ["ple", "pril", "ply"]
  b
   └── a
        └── Bucket ["nana"]
```

---

## ✅ 요약

| 키워드 | 설명 |
|--------|------|
| ART | 적응형 Radix Tree로 캐시 최적화와 빠른 탐색 |
| HAT-Trie | Trie + HashTable 하이브리드로 고성능 문자열 처리 |
| 공통점 | 고속, 캐시 친화, 정렬 지원 |
| 차이점 | ART는 Trie 압축 + 노드 변화 / HAT는 Bucket 변환 |
