---
layout: post
title: Data Structure - 스택
date: 2024-12-10 19:20:23 +0900
category: Data Structure
---
# 스택(Stack) — 자료구조의 기초

스택은 가장 단순하면서도 중요한 **선형 자료구조**입니다.  
컴파일러, 웹 브라우저, 재귀 호출 등 수많은 시스템에서 사용됩니다.

---

## 📌 1. 스택이란?

> 스택은 **후입선출(LIFO: Last In, First Out)** 구조입니다.

- 데이터를 **맨 위(top)**에 넣고, **맨 위에서 꺼냄**
- 실생활 예시: 책 더미, 웹 페이지 뒤로 가기

---

## 📐 2. 기본 연산

| 연산 | 설명 |
|------|------|
| `push(x)` | 요소 x를 스택에 넣기 |
| `pop()` | 가장 최근 요소 꺼내기 |
| `top()` | 가장 위의 요소 확인 (제거하지 않음) |
| `empty()` | 스택이 비었는지 확인 |
| `size()` | 현재 요소 개수 확인 |

---

## 🧱 3. 배열 기반 스택 구현 (C++)

```cpp
class StackArray {
private:
    int* data;
    int topIndex;
    int capacity;

public:
    StackArray(int cap = 100) {
        capacity = cap;
        data = new int[capacity];
        topIndex = -1;
    }

    ~StackArray() {
        delete[] data;
    }

    void push(int val) {
        if (topIndex + 1 == capacity)
            throw std::overflow_error("Stack Overflow");
        data[++topIndex] = val;
    }

    void pop() {
        if (empty())
            throw std::underflow_error("Stack Underflow");
        --topIndex;
    }

    int top() const {
        if (empty())
            throw std::runtime_error("Stack is empty");
        return data[topIndex];
    }

    bool empty() const {
        return topIndex == -1;
    }

    int size() const {
        return topIndex + 1;
    }
};
```

---

## 🔗 4. 연결 리스트 기반 스택 구현

```cpp
struct Node {
    int data;
    Node* next;
    Node(int val) : data(val), next(nullptr) {}
};

class StackList {
private:
    Node* head;
    int count;

public:
    StackList() : head(nullptr), count(0) {}

    ~StackList() {
        while (!empty()) pop();
    }

    void push(int val) {
        Node* newNode = new Node(val);
        newNode->next = head;
        head = newNode;
        count++;
    }

    void pop() {
        if (empty())
            throw std::underflow_error("Stack Underflow");
        Node* temp = head;
        head = head->next;
        delete temp;
        count--;
    }

    int top() const {
        if (empty())
            throw std::runtime_error("Stack is empty");
        return head->data;
    }

    bool empty() const { return head == nullptr; }

    int size() const { return count; }
};
```

---

## ⚙️ 5. STL `std::stack` 사용 예시

```cpp
#include <stack>

std::stack<int> s;
s.push(10);
s.push(20);
s.pop();           // 20 제거
int x = s.top();   // x = 10
```

> `std::stack`은 내부적으로 **`deque` 또는 `vector`를 기반**으로 구현됩니다.

---

## 💡 6. 스택 응용 예시

### ✅ 괄호 검사

```cpp
#include <stack>
#include <string>

bool isValid(const std::string& s) {
    std::stack<char> st;
    for (char c : s) {
        if (c == '(') st.push(c);
        else if (c == ')') {
            if (st.empty()) return false;
            st.pop();
        }
    }
    return st.empty();
}
```

---

## 📊 7. 배열 vs 연결리스트 vs STL 비교

| 구현 방식 | 장점 | 단점 |
|-----------|------|------|
| 배열 기반 | 빠른 접근, 캐시 효율 ↑ | 고정 크기 또는 동적 할당 필요 |
| 연결 리스트 | 유연한 크기 | 메모리 오버헤드, 포인터 필요 |
| STL stack | 안정성, 사용 편의성 ↑ | 세부 제어 불가능 |

---

## ✅ 정리

- 스택은 후입선출(LIFO)의 대표 구조
- 배열, 연결리스트, STL 방식으로 모두 구현 가능
- 괄호 검사, 수식 계산, DFS 등 다양한 알고리즘에 활용