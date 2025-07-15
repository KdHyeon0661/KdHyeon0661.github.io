---
layout: post
title: Data Structure - 트라이
date: 2024-12-09 19:20:23 +0900
category: Data Structure
---
# 🔠 Trie (트라이) 자료구조 - 문자열 검색을 위한 트리

---

## 📌 1. Trie란?

**Trie(트라이)**는 문자열을 저장하고 탐색하는 데 최적화된 **트리 기반 자료구조**입니다.  
각 노드는 **문자의 단위**를 저장하며, 루트부터 차례로 이어지는 경로가 **문자열을 구성**합니다.

### ✅ 특징

- 문자열의 **공통 접두사(Prefix)**를 공유하여 공간 절약
- 문자열 검색 시간: O(L) (L = 문자열 길이)
- 해시맵보다 **정렬된 결과**, **접두사 검색** 등에 유리

---

## 🧩 2. 기본 구조

```cpp
struct TrieNode {
    TrieNode* children[26];  // 소문자 알파벳만
    bool isEndOfWord;

    TrieNode() {
        fill(begin(children), end(children), nullptr);
        isEndOfWord = false;
    }
};
```

- `children[26]`: 각 알파벳에 대한 포인터
- `isEndOfWord`: 하나의 단어의 끝을 표시

---

## 🛠️ 3. 삽입 함수 (insert)

```cpp
void insert(TrieNode* root, const string& word) {
    TrieNode* node = root;
    for (char ch : word) {
        int idx = ch - 'a';
        if (!node->children[idx])
            node->children[idx] = new TrieNode();
        node = node->children[idx];
    }
    node->isEndOfWord = true;
}
```

---

## 🔍 4. 검색 함수 (search)

```cpp
bool search(TrieNode* root, const string& word) {
    TrieNode* node = root;
    for (char ch : word) {
        int idx = ch - 'a';
        if (!node->children[idx])
            return false;
        node = node->children[idx];
    }
    return node->isEndOfWord;
}
```

---

## 🧹 5. 삭제 함수 (delete)

```cpp
bool remove(TrieNode* node, const string& word, int depth = 0) {
    if (!node) return false;

    if (depth == word.length()) {
        if (!node->isEndOfWord) return false;
        node->isEndOfWord = false;
        return all_of(begin(node->children), end(node->children),
                      [](TrieNode* child) { return child == nullptr; });
    }

    int idx = word[depth] - 'a';
    if (remove(node->children[idx], word, depth + 1)) {
        delete node->children[idx];
        node->children[idx] = nullptr;

        return !node->isEndOfWord &&
               all_of(begin(node->children), end(node->children),
                      [](TrieNode* child) { return child == nullptr; });
    }

    return false;
}
```

---

## 🧪 6. 사용 예시 (main 함수)

```cpp
int main() {
    TrieNode* root = new TrieNode();

    insert(root, "apple");
    insert(root, "app");
    insert(root, "bat");
    insert(root, "bad");

    cout << boolalpha;
    cout << "search(\"app\"): " << search(root, "app") << endl;   // true
    cout << "search(\"apple\"): " << search(root, "apple") << endl; // true
    cout << "search(\"appl\"): " << search(root, "appl") << endl; // false

    remove(root, "apple");
    cout << "search(\"apple\") after deletion: " << search(root, "apple") << endl; // false
    cout << "search(\"app\") still exists: " << search(root, "app") << endl; // true

    return 0;
}
```

---

## ✨ 7. 트라이의 응용 분야

| 분야 | 설명 |
|------|------|
| 문자열 자동완성 | 접두사 기반 검색 |
| 사전 구현 | 단어 등록/검색 |
| 문자열 필터링 | 비속어 탐지 등 |
| IP 라우팅 | 이진 Trie 활용 |
| 압축 알고리즘 | LZW 등 |

---

## ✅ 요약

| 키워드 | 설명 |
|--------|------|
| 트라이 | 문자열 저장을 위한 트리 구조 |
| 시간복잡도 | 삽입/검색/삭제: O(L) |
| 구현 | 자식 노드 배열 + 끝 표시 플래그 |
| 장점 | 접두사 검색, 정렬된 사전, 고속 탐색 |
