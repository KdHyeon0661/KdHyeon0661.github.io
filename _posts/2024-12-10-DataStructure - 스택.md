---
layout: post
title: Data Structure - 스택
date: 2024-12-10 19:20:23 +0900
category: Data Structure
---
# 스택(Stack)

## 스택이란?

> 스택은 **후입선출(LIFO: Last In, First Out)** 구조다.

- 삽입(`push`)과 삭제(`pop`)는 **맨 위(top)**에서만 수행
- 대표적 활용: **함수 호출 스택**, **웹 브라우저 방문 기록**, **문자열 파싱**, **DFS**

---

## 기본 연산과 복잡도

| 연산      | 의미                      | 평균 시간 |
|-----------|---------------------------|-----------|
| `push(x)` | 맨 위에 x 삽입            | O(1)\*    |
| `pop()`   | 맨 위 요소 제거+반환      | O(1)      |
| `top()`   | 맨 위 요소 조회(제거 X)   | O(1)      |
| `empty()` | 비었는가                  | O(1)      |
| `size()`  | 요소 개수                 | O(1)      |

\* 동적 배열 기반일 때, **리사이즈 순간**은 O(n)이나 **평균(Amortized) O(1)**.
수식으로 직관을 보면, 용량이 두 배로 늘 때마다 복사 비용이 분산되어
$$
\text{평균 비용} \approx \frac{1+1+2+4+\cdots+2^{k}}{2^{k}} = O(1)
$$

---

## 배열 기반 스택 (정적 용량)

초안의 정적 배열 버전을 유지하되, 몇 가지 세부 조언을 포함한다.

```cpp
#include <stdexcept>

class StackArray {
private:
    int* data;
    int  topIndex;
    int  capacity;

public:
    explicit StackArray(int cap = 100)
        : data(new int[cap]), topIndex(-1), capacity(cap) {}

    ~StackArray() { delete[] data; }

    void push(int val) {
        if (topIndex + 1 == capacity) throw std::overflow_error("Stack Overflow");
        data[++topIndex] = val;
    }

    void pop() {
        if (empty()) throw std::underflow_error("Stack Underflow");
        --topIndex;
    }

    int top() const {
        if (empty()) throw std::runtime_error("Stack is empty");
        return data[topIndex];
    }

    bool empty() const { return topIndex == -1; }
    int  size()  const { return topIndex + 1; }
};
```

- **장점**: 메모리 연속 → **캐시 친화적**, 성능 예측 용이
- **단점**: **고정 용량**으로 오버플로우 처리 필요

---

## 동적 배열 기반 스택 (템플릿, Amortized O(1))

실전에서는 **제네릭 템플릿 + 자동 확장**이 편리하다.

```cpp
#include <stdexcept>
#include <utility>
#include <new> // std::bad_alloc

template <class T>
class DynStack {
    T*   buf_      = nullptr;
    int  top_      = -1;
    int  capacity_ = 0;

    void grow() {
        int newCap = capacity_ ? capacity_ * 2 : 4;
        T*  nb = static_cast<T*>(::operator new[](sizeof(T) * newCap));
        // 기존 요소 이동(예외 안전: 이동 중 예외 발생 시 원상 복구 가능)
        try {
            for (int i = 0; i <= top_; ++i) {
                new (&nb[i]) T(std::move(buf_[i]));
                buf_[i].~T();
            }
        } catch (...) {
            // 부분 구성 해제
            for (int i = 0; i <= top_; ++i) nb[i].~T();
            ::operator delete[](nb);
            throw;
        }
        ::operator delete[](buf_);
        buf_      = nb;
        capacity_ = newCap;
    }

public:
    DynStack() = default;

    ~DynStack() {
        for (int i = 0; i <= top_; ++i) buf_[i].~T();
        ::operator delete[](buf_);
    }

    bool empty() const { return top_ == -1; }
    int  size()  const { return top_ + 1;  }

    void push(const T& v) {
        if (top_ + 1 == capacity_) grow();
        new (&buf_[++top_]) T(v);
    }
    void push(T&& v) {
        if (top_ + 1 == capacity_) grow();
        new (&buf_[++top_]) T(std::move(v));
    }

    void pop() {
        if (empty()) throw std::underflow_error("Stack Underflow");
        buf_[top_--].~T();
    }

    T& top() {
        if (empty()) throw std::runtime_error("Stack is empty");
        return buf_[top_];
    }
    const T& top() const {
        if (empty()) throw std::runtime_error("Stack is empty");
        return buf_[top_];
    }
};
```

- **핵심**: **수동 메모리 관리**로 불필요한 디폴트 생성 방지, 강건한 예외 안전
- **복잡도**: `push` **평균 O(1)**, 리사이즈 시 O(n)

---

## 연결 리스트 기반 스택 (템플릿)

초안의 int 고정형을 **템플릿**으로 일반화.

```cpp
#include <stdexcept>

template <class T>
class ListStack {
    struct Node {
        T     data;
        Node* next;
        template <class U>
        explicit Node(U&& v, Node* n=nullptr)
            : data(std::forward<U>(v)), next(n) {}
    };
    Node* head_ = nullptr; // top
    int   cnt_  = 0;

public:
    ~ListStack() { while (!empty()) pop(); }

    bool empty() const { return head_ == nullptr; }
    int  size()  const { return cnt_; }

    void push(const T& v) {
        head_ = new Node(v, head_);
        ++cnt_;
    }
    void push(T&& v) {
        head_ = new Node(std::move(v), head_);
        ++cnt_;
    }

    void pop() {
        if (empty()) throw std::underflow_error("Stack Underflow");
        Node* t = head_;
        head_ = head_->next;
        delete t;
        --cnt_;
    }

    T& top() {
        if (empty()) throw std::runtime_error("Stack is empty");
        return head_->data;
    }
    const T& top() const {
        if (empty()) throw std::runtime_error("Stack is empty");
        return head_->data;
    }
};
```

- **장점**: **용량 제한 없음**, push/pop 모두 O(1)
- **단점**: 노드 할당 비용, **캐시 비우호**

---

## 표준 `std::stack` 올바르게 쓰기

```cpp
#include <stack>
#include <vector>
#include <deque>

std::stack<int> s1;                      // 기본: deque<int>
std::stack<int, std::vector<int>> s2;    // vector 기반으로 지정 가능
std::stack<int, std::deque<int>>  s3;    // deque 명시
```

- `std::stack`은 **컨테이너 어댑터**: 내부 컨테이너(기본 `deque`/선택 `vector`, `list`)의 `push_back/pop_back/back`을 래핑
- **임의 접근/순회 API 없음**: 스택의 **추상화**를 지키기 위함

---

## 필수 응용 문제

### 괄호 검사(다종 괄호)

초안의 `()`만이 아니라 `()[]{}` 전부.

```cpp
#include <stack>
#include <string>

bool isValidBrackets(const std::string& s) {
    std::stack<char> st;
    for (char c : s) {
        if (c=='(' || c=='[' || c=='{') st.push(c);
        else if (c==')' || c==']' || c=='}') {
            if (st.empty()) return false;
            char o = st.top(); st.pop();
            if ((c==')' && o!='(') || (c==']' && o!='[') || (c=='}' && o!='{'))
                return false;
        }
    }
    return st.empty();
}
```

### 중위식 → 후위식(Shunting-yard)

연산자 우선순위/결합 법칙을 스택으로 처리.

```cpp
#include <string>
#include <vector>
#include <cctype>

int precedence(char op){
    if (op=='+'||op=='-') return 1;
    if (op=='*'||op=='/') return 2;
    return 0;
}
bool rightAssoc(char op){ return false; } // ^라면 true

std::vector<std::string> infixToPostfix(const std::string& s){
    std::vector<std::string> out;
    std::stack<char> ops;
    for (size_t i=0;i<s.size();){
        if (std::isspace((unsigned char)s[i])) { ++i; continue; }
        if (std::isdigit((unsigned char)s[i])){
            size_t j=i;
            while (j<s.size() && std::isdigit((unsigned char)s[j])) ++j;
            out.push_back(s.substr(i, j-i));
            i=j;
        } else if (s[i]=='('){ ops.push('('); ++i; }
        else if (s[i]==')'){
            while (!ops.empty() && ops.top()!='('){ out.push_back(std::string(1,ops.top())); ops.pop(); }
            if (!ops.empty() && ops.top()=='(') ops.pop();
            ++i;
        } else { // operator
            char op = s[i++];
            while (!ops.empty() && ops.top()!='(' &&
                  (precedence(ops.top()) > precedence(op) ||
                  (precedence(ops.top()) == precedence(op) && !rightAssoc(op)))){
                out.push_back(std::string(1, ops.top())); ops.pop();
            }
            ops.push(op);
        }
    }
    while (!ops.empty()){ out.push_back(std::string(1, ops.top())); ops.pop(); }
    return out;
}
```

### 후위식 계산(RPN)

```cpp
#include <stack>
#include <string>
#include <vector>

int evalRPN(const std::vector<std::string>& toks){
    std::stack<int> st;
    for (auto &t : toks){
        if (t=="+"||t=="-"||t=="*"||t=="/"){
            int b=st.top(); st.pop();
            int a=st.top(); st.pop();
            if (t=="+") st.push(a+b);
            else if (t=="-") st.push(a-b);
            else if (t=="*") st.push(a*b);
            else st.push(a/b);
        } else {
            st.push(std::stoi(t));
        }
    }
    return st.top();
}
```

---

## 확장 스택: `MinStack` (O(1) 최소값 조회)

보조 스택에 **최소값 누적** 저장.

```cpp
#include <stack>
#include <stdexcept>

template <class T>
class MinStack {
    std::stack<T> s, mins;
public:
    void push(const T& x){
        s.push(x);
        if (mins.empty() || x <= mins.top()) mins.push(x);
    }
    void pop(){
        if (s.empty()) throw std::underflow_error("empty");
        if (s.top() == mins.top()) mins.pop();
        s.pop();
    }
    const T& top() const { if (s.empty()) throw std::underflow_error("empty"); return s.top(); }
    const T& getMin() const { if (mins.empty()) throw std::underflow_error("empty"); return mins.top(); }
    bool empty() const { return s.empty(); }
    int  size()  const { return (int)s.size(); }
};
```

- **변형**: 최대값 스택, **두 스택으로 큐** 구현 등

---

## 모노토닉 스택(Monotonic Stack) — 실전 핵심

### 다음 큰 원소(Next Greater Element)

오른쪽으로 **자신보다 큰 첫 원소**의 인덱스/값을 구한다. 스택에 **내림차순**을 유지.

```cpp
#include <vector>
#include <stack>

std::vector<int> nextGreater(const std::vector<int>& a){
    int n=(int)a.size();
    std::vector<int> ans(n, -1);
    std::stack<int> st; // index stack, values strictly decreasing
    for(int i=0;i<n;++i){
        while(!st.empty() && a[st.top()] < a[i]){
            ans[st.top()] = a[i];
            st.pop();
        }
        st.push(i);
    }
    return ans;
}
```

- **복잡도**: 각 인덱스가 **한 번 push, 한 번 pop** → 총 O(n)

### 히스토그램의 최대 직사각형 (LeetCode 84)

높이 배열에서 **가장 큰 직사각형** 넓이 구하기. **증가 스택** 유지.

```cpp
#include <vector>
#include <stack>
#include <algorithm>

long long largestRectangle(const std::vector<int>& h){
    std::stack<int> st; // indices, heights increasing
    long long best=0;
    int n=(int)h.size();
    for (int i=0;i<=n;++i){
        int cur = (i==n? 0 : h[i]);
        while(!st.empty() && h[st.top()] > cur){
            int height = h[st.top()]; st.pop();
            int left = st.empty()? -1 : st.top();
            int width = i - left - 1;
            best = std::max(best, 1LL*height*width);
        }
        st.push(i);
    }
    return best;
}
```

- **응용**: **이진 행렬의 최대 사각형**(행별 누적 히스토그램으로 변환) 등

---

## 브라우저 뒤로/앞으로 — 두 스택 패턴

```cpp
#include <string>
#include <stack>

class Browser {
    std::string cur;
    std::stack<std::string> backSt, fwdSt;
public:
    explicit Browser(std::string home): cur(std::move(home)) {}
    void visit(const std::string& url){
        backSt.push(cur);
        cur = url;
        while(!fwdSt.empty()) fwdSt.pop();
    }
    bool back(){
        if (backSt.empty()) return false;
        fwdSt.push(cur);
        cur = backSt.top(); backSt.pop();
        return true;
    }
    bool forward(){
        if (fwdSt.empty()) return false;
        backSt.push(cur);
        cur = fwdSt.top(); fwdSt.pop();
        return true;
    }
    const std::string& current() const { return cur; }
};
```

- **일반화**: **명령 취소/재실행(Undo/Redo)**, 편집기 히스토리

---

## 재귀 제거 / DFS

### 재귀 → 명시적 스택 치환

```cpp
// 재귀 DFS
void dfs_rec(int u, const std::vector<std::vector<int>>& g, std::vector<int>& vis){
    vis[u]=1;
    for(int v: g[u]) if(!vis[v]) dfs_rec(v,g,vis);
}

// 반복 DFS
#include <stack>

void dfs_iter(int s, const std::vector<std::vector<int>>& g, std::vector<int>& vis){
    std::stack<int> st; st.push(s);
    while(!st.empty()){
        int u=st.top(); st.pop();
        if (vis[u]) continue;
        vis[u]=1;
        for (int v: g[u]) if(!vis[v]) st.push(v);
    }
}
```

- **스택**은 **시스템 호출 스택의 역할**을 직접 수행한다.

### 문자열 역전/수식 평가 등 미니 예제

```cpp
#include <stack>
#include <string>

std::string reverseStr(const std::string& s){
    std::stack<char> st; for(char c: s) st.push(c);
    std::string t; t.reserve(s.size());
    while(!st.empty()){ t.push_back(st.top()); st.pop(); }
    return t;
}
```

---

## 수학적 뒷받침 (암ORTIZED 분석 스냅샷)

동적 배열 기반 스택에서 용량을 두 배로 늘린다면, `push`의 평균 비용은 상수로 수렴한다.

- 총 `m`번 `push`, 복사 비용은 **용량 확장 시점**에만 발생
- 확장 시 복사 수열: \(1 + 2 + 4 + \dots + 2^{k} \approx 2^{k+1}-1\), 전체 삽입 수 \(m \approx 2^{k}\)
- 평균:
  $$
  \frac{2^{k+1}-1}{2^k} \approx 2 \Rightarrow O(1)
  $$

---

## 경계/예외/성능 팁

- **예외 안전**: `pop()`은 비어 있으면 던진다. API 설계 시 **사전 검사** vs **예외** 선택 명확히.
- **성능**:
  - **배열 기반**이 **연결 리스트**보다 보통 빠름(캐시 지역성, 할당 비용 없음).
  - 대용량 환경에서는 **reserve**(vector 기반)로 리사이즈 빈도 감소.
- **반복자 제공?** 스택은 **추상**을 유지하는 게 원칙. 디버깅/테스트용으로만 순회 노출을 고려.
- **스레드 안전**: 단일 스택에 다중 스레드 접근 시 **락** 또는 **lock-free**(복잡) 필요. 대부분의 경우 **락**이 현실적.

---

## 추가 연습 과제

1. **MinStack**을 1개 스택/차이 저장 방식으로 구현해 보라(메모리 더 절약).
2. 모노토닉 스택으로 **다음 작은 원소**/ **이전 큰 원소**도 구해 보라.
3. 괄호 검사에 `*`(와일드카드)를 도입(LeetCode 678 변형).
4. Shunting-yard에 **단항 부호**(`-x`)와 **지수**(`^`)를 추가해 보라(우결합).
5. 히스토그램 문제를 **원형 배열**(끝과 처음 연결)로 확장.

---

## 요약

- 스택은 가장 기본적이면서 **파싱/탐색/최적화**에 광범위하게 쓰인다.
- **배열 기반**은 빠르고 간단, **연결 리스트**는 유연.
- **STL `std::stack`**은 컨테이너 어댑터로 목적에 충실.
- **응용의 폭**: 괄호/파서/후위식/MinStack/모노토닉/히스토그램/브라우저/Undo-Redo/DFS…
실무에 바로 넣을 수 있는 **패턴**을 익히면, 많은 “스택스러운” 문제들이 **한 줄기**로 보인다.
