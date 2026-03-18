---
layout: post
title: Data Structure - 스택
date: 2024-12-10 19:20:23 +0900
category: Data Structure
---
# 스택

## 스택이란?

스택은 **후입선출(LIFO: Last In, First Out)** 구조입니다.  
가장 마지막에 넣은 데이터가 가장 먼저 나옵니다.

- **push**: 데이터를 맨 위에 쌓음
- **pop**: 맨 위 데이터를 꺼냄 (제거)
- **top**: 맨 위 데이터를 확인만 함 (제거하지 않음)

---

## 기본 연산과 시간 복잡도

| 연산 | 설명 | 시간 복잡도 |
|------|------|------------|
| `push(x)` | 맨 위에 x 삽입 | O(1) |
| `pop()`   | 맨 위 요소 제거 | O(1) |
| `top()`   | 맨 위 요소 조회 | O(1) |
| `empty()` | 비었는지 확인 | O(1) |

---

## 구현 방법 비교

### 1. 배열 기반 (정적 크기)

```cpp
#include <stdexcept>

class Stack {
    int data[100];
    int topIndex = -1;
public:
    void push(int val) {
        if (topIndex + 1 == 100) throw std::overflow_error("Stack overflow");
        data[++topIndex] = val;
    }
    void pop() {
        if (topIndex == -1) throw std::underflow_error("Stack underflow");
        --topIndex;
    }
    int top() const {
        if (topIndex == -1) throw std::runtime_error("Stack is empty");
        return data[topIndex];
    }
    bool empty() const { return topIndex == -1; }
};
```

### 2. 연결 리스트 기반

```cpp
#include <stdexcept>

class Stack {
    struct Node {
        int data;
        Node* next;
        Node(int val, Node* nxt = nullptr) : data(val), next(nxt) {}
    };
    Node* head = nullptr;
public:
    ~Stack() { while (!empty()) pop(); }
    void push(int val) { head = new Node(val, head); }
    void pop() {
        if (empty()) throw std::underflow_error("Stack underflow");
        Node* temp = head;
        head = head->next;
        delete temp;
    }
    int top() const {
        if (empty()) throw std::runtime_error("Stack is empty");
        return head->data;
    }
    bool empty() const { return head == nullptr; }
};
```

### 3. 표준 라이브러리 사용 (`std::stack`)

```cpp
#include <stack>
#include <iostream>

int main() {
    std::stack<int> s;
    s.push(10);
    s.push(20);
    std::cout << s.top() << '\n'; // 20
    s.pop();
    std::cout << s.top() << '\n'; // 10
    return 0;
}
```

---

## 핵심 예제 3개

### 예제 1: 괄호 검사

```cpp
#include <stack>
#include <string>

bool checkBrackets(const std::string& s) {
    std::stack<char> st;
    for (char c : s) {
        if (c == '(' || c == '[' || c == '{')
            st.push(c);
        else if (c == ')' || c == ']' || c == '}') {
            if (st.empty()) return false;
            char top = st.top(); st.pop();
            if ((c == ')' && top != '(') ||
                (c == ']' && top != '[') ||
                (c == '}' && top != '{'))
                return false;
        }
    }
    return st.empty();
}
```

### 예제 2: 최소값 조회 스택

```cpp
#include <stack>

class MinStack {
    std::stack<int> data;
    std::stack<int> mins;
public:
    void push(int x) {
        data.push(x);
        if (mins.empty() || x <= mins.top())
            mins.push(x);
    }
    void pop() {
        if (data.top() == mins.top()) mins.pop();
        data.pop();
    }
    int top() { return data.top(); }
    int getMin() { return mins.top(); }
};
```

### 예제 3: 후위 표기식 계산

```cpp
#include <stack>
#include <string>
#include <vector>
#include <cctype>

int evalRPN(const std::vector<std::string>& tokens) {
    std::stack<int> st;
    for (const auto& tok : tokens) {
        if (isdigit(tok[0]) || (tok.size() > 1 && tok[0] == '-')) {
            st.push(std::stoi(tok));
        } else {
            int b = st.top(); st.pop();
            int a = st.top(); st.pop();
            if (tok == "+") st.push(a + b);
            else if (tok == "-") st.push(a - b);
            else if (tok == "*") st.push(a * b);
            else if (tok == "/") st.push(a / b);
        }
    }
    return st.top();
}
```

---

## 마무리

스택은 **후입선출** 구조로, 함수 호출, 실행 취소, 괄호 검사 등에 사용됩니다.  
실무에서는 `std::stack`으로 충분하고, 특수 목적(최소값 추적 등)에 따라 변형해서 사용합니다.