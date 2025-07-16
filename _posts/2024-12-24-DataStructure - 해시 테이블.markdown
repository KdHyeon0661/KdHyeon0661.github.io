---
layout: post
title: Data Structure - 펜윅 트리
date: 2024-12-09 19:20:23 +0900
category: Data Structure
---
# 🔢 해시 테이블(Hash Table) - C++ 구현과 원리

---

## 📌 1. 해시 테이블이란?

해시 테이블은 **키(Key)**를 **해시 함수(Hash Function)**에 넣어 나온 해시 값을 배열의 인덱스로 사용하여 값을 저장하는 자료구조입니다.  
탐색, 삽입, 삭제가 평균 `O(1)`의 시간복잡도로 매우 빠릅니다.

---

## 🧠 2. 기본 동작 원리

1. **해시 함수(Hash Function)**: `key → index` 로 매핑  
2. **배열(Bucket)**에 저장  
3. **충돌(Collision)** 발생 시 처리 필요 (같은 index에 다른 key가 매핑될 때)

---

## ⚠️ 3. 해시 충돌(Collision)이란?

서로 다른 키가 같은 해시 값을 가질 수 있습니다.  
이를 **충돌**이라 하며, 이를 해결하는 방식이 필요합니다.

---

## 🔧 4. 충돌 해결 방법

| 방법 | 설명 | 특징 |
|------|------|------|
| **체이닝 (Chaining)** | 같은 버킷에 여러 값을 연결 리스트로 저장 | 구현 간단, 메모리 추가 필요 |
| **개방 주소법 (Open Addressing)** | 비어 있는 다음 위치를 찾아 저장 | 메모리 효율적, 삭제 어려움 |

### 개방 주소법 - 탐사(probing)

개방 주소법은 탐사(probing)라는 과정을 통해 빈 공간을 탐색합니다

| 방법                             | 설명                                                    | 특징                                    |
|----------------------------------|---------------------------------------------------------|-----------------------------------------|
| **선형 탐사 (Linear Probing)**     | 다음 빈 칸을 한 칸씩 순차적으로 탐색하여 저장              | 구현이 간단, 클러스터링(집중현상) 발생 가능    |
| **이차 탐사 (Quadratic Probing)**  | 빈 칸을 1², 2², 3², ... 거리로 건너뛰며 탐색                 | 탐사 패턴이 분산되어 클러스터링 완화, 삭제 어려움 |
| **이중 해싱 (Double Hashing)**      | 별도의 두 번째 해시 함수로 이동 거리를 결정                    | 충돌 분산 효과 큼, 구현 복잡, 성능 우수         |

---

## 🔍 5. C++로 해시 테이블 직접 구현 (체이닝 방식)

```cpp
#include <iostream>
#include <list>
#include <vector>
using namespace std;

class HashTable {
    static const int SIZE = 7;
    vector<list<pair<int, string>>> table;

    int hash(int key) {
        return key % SIZE;
    }

public:
    HashTable() : table(SIZE) {}

    void insert(int key, const string& value) {
        int idx = hash(key);
        for (auto& pair : table[idx]) {
            if (pair.first == key) {
                pair.second = value; // update
                return;
            }
        }
        table[idx].emplace_back(key, value);
    }

    string search(int key) {
        int idx = hash(key);
        for (auto& pair : table[idx]) {
            if (pair.first == key) return pair.second;
        }
        return "Not found";
    }

    void erase(int key) {
        int idx = hash(key);
        auto& chain = table[idx];
        for (auto it = chain.begin(); it != chain.end(); ++it) {
            if (it->first == key) {
                chain.erase(it);
                return;
            }
        }
    }

    void print() {
        for (int i = 0; i < SIZE; ++i) {
            cout << i << ": ";
            for (auto& pair : table[i])
                cout << "(" << pair.first << ", " << pair.second << ") ";
            cout << endl;
        }
    }
};

int main() {
    HashTable ht;
    ht.insert(1, "Apple");
    ht.insert(8, "Banana"); // 충돌 발생 (1 % 7 == 1, 8 % 7 == 1)
    ht.insert(15, "Cherry");

    ht.print();

    cout << "Search key 8: " << ht.search(8) << endl;

    ht.erase(8);
    cout << "After erase key 8:\n";
    ht.print();

    return 0;
}
```

---

## 🔍 5. C++로 해시 테이블 직접 구현 (선형탐사 방식)

```cpp
#include <iostream>
#include <vector>
using namespace std;

enum State { EMPTY, OCCUPIED, DELETED };

struct Entry {
    int key;
    string value;
    State state;
    Entry() : key(0), value(""), state(EMPTY) {}
};

class LinearProbingHashTable {
private:
    static const int SIZE = 11; // 소수 권장
    vector<Entry> table;

    int hash(int key) {
        return key % SIZE;
    }

public:
    LinearProbingHashTable() : table(SIZE) {}

    void insert(int key, const string& value) {
        int idx = hash(key);
        int start = idx;
        do {
            if (table[idx].state == EMPTY || table[idx].state == DELETED) {
                table[idx].key = key;
                table[idx].value = value;
                table[idx].state = OCCUPIED;
                return;
            } else if (table[idx].state == OCCUPIED && table[idx].key == key) {
                table[idx].value = value; // update
                return;
            }
            idx = (idx + 1) % SIZE;
        } while (idx != start);
        cout << "Hash Table is full\n";
    }

    string search(int key) {
        int idx = hash(key);
        int start = idx;
        do {
            if (table[idx].state == EMPTY)
                return "Not found";
            if (table[idx].state == OCCUPIED && table[idx].key == key)
                return table[idx].value;
            idx = (idx + 1) % SIZE;
        } while (idx != start);
        return "Not found";
    }

    void erase(int key) {
        int idx = hash(key);
        int start = idx;
        do {
            if (table[idx].state == EMPTY)
                return;
            if (table[idx].state == OCCUPIED && table[idx].key == key) {
                table[idx].state = DELETED;
                return;
            }
            idx = (idx + 1) % SIZE;
        } while (idx != start);
    }

    void print() {
        for (int i = 0; i < SIZE; ++i) {
            cout << i << ": ";
            if (table[i].state == OCCUPIED)
                cout << "(" << table[i].key << ", " << table[i].value << ")";
            else if (table[i].state == DELETED)
                cout << "Deleted";
            else
                cout << "Empty";
            cout << endl;
        }
    }
};

int main() {
    LinearProbingHashTable ht;

    ht.insert(10, "Apple");
    ht.insert(21, "Banana"); // 충돌: 10 % 11 == 10, 21 % 11 == 10
    ht.insert(32, "Cherry"); // 또 충돌 발생
    ht.insert(43, "Durian");

    ht.print();

    cout << "\nSearch 21: " << ht.search(21) << endl;

    ht.erase(21);
    cout << "\nAfter erasing 21:\n";
    ht.print();

    return 0;
}
```

---

## 📊 7. 시간 복잡도 분석

| 연산 | 평균 시간 | 최악 시간 |
|------|-----------|-----------|
| 삽입 | O(1) | O(n) (충돌 모두 같은 곳에 몰릴 경우) |
| 탐색 | O(1) | O(n) |
| 삭제 | O(1) | O(n) |

---

## 🧠 8. 해시 테이블 장단점

### 👍 장점
- 매우 빠른 평균 시간복잡도
- 키 기반 접근
- 삽입/삭제/탐색 모두 O(1) 가능

### 👎 단점
- 충돌 가능성
- 메모리 사용 비효율적 (버킷 낭비)
- 정렬 불가

---

## 🧩 9. C++ STL의 `unordered_map` / `unordered_set`

C++ 표준 라이브러리에서 해시 기반 컨테이너는 다음과 같습니다:

- `unordered_map<Key, Value>`  
- `unordered_set<Key>`  
- 평균 `O(1)` 속도 보장 (구현에 따라 다를 수 있음)
- 내부적으로 체이닝 방식 사용

```cpp
#include <unordered_map>
unordered_map<int, string> mp;
mp[42] = "Hello";
cout << mp[42]; // "Hello"
```

---

## ✅ 마무리 요약

| 요소 | 설명 |
|------|------|
| 키 기반 자료구조 | 해시 테이블 |
| 속도 | 평균 O(1) |
| 충돌 해결 | 체이닝 or 개방 주소 |
| STL | `unordered_map`, `unordered_set` |
| 정렬 | 불가 |
| 삽입/탐색/삭제 | 빠름 (but 충돌 시 저하) |