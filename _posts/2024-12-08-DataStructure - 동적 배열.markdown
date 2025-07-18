---
layout: post
title: Data Structure - 동적 배열
date: 2024-12-08 20:20:23 +0900
category: Data Structure
---
# 직접 구현하는 동적 배열 (Dynamic Array) in C++

`std::vector`는 매우 강력한 컨테이너이지만, 그 **동작 원리를 이해하려면 직접 구현해보는 것이 최고**입니다.  
이번 글에서는 **C++로 동적 배열을 직접 구현**하면서 내부 확장 로직, 삽입/삭제 기능 등을 자세히 설명합니다.

---

## 📌 1. 동적 배열이란?

정적인 배열과 달리, **크기를 동적으로 늘리거나 줄일 수 있는 배열**입니다.  
메모리는 처음에는 작게 할당하고, 공간이 부족해질 때 더 큰 배열을 새로 할당하고 복사합니다.

### 특징 요약

- **연속된 메모리 공간** 사용
- **O(1)** 시간의 임의 접근
- 자동으로 **메모리 확장**
- 삽입/삭제는 평균적으로 빠르지만 **최악의 경우 O(n)**

---

## 🧱 2. 구조 설계

```cpp
class DynamicArray {
private:
    int* data;       // 실제 데이터 저장소
    int capacity;    // 현재 할당된 공간
    int length;      // 실제 저장된 요소 수

    void resize(int newCapacity);  // 내부 확장 함수

public:
    DynamicArray();
    ~DynamicArray();

    void push_back(int val);
    void pop_back();
    int size() const;
    int operator[](int index) const;
    void insert(int index, int val);
    void erase(int index);
    void print() const;
};
```

---

## ⚙️ 3. 생성자와 기본 기능

```cpp
DynamicArray::DynamicArray() {
    capacity = 4;        // 초기 크기
    length = 0;
    data = new int[capacity];
}

DynamicArray::~DynamicArray() {
    delete[] data;
}

int DynamicArray::size() const {
    return length;
}

int DynamicArray::operator[](int index) const {
    if (index < 0 || index >= length)
        throw std::out_of_range("Index out of range");
    return data[index];
}
```

---

## ➕ 4. 삽입(push_back)과 자동 확장(resize)

```cpp
void DynamicArray::push_back(int val) {
    if (length == capacity)
        resize(capacity * 2);  // 자동 확장
    data[length++] = val;
}

void DynamicArray::resize(int newCapacity) {
    int* newData = new int[newCapacity];
    for (int i = 0; i < length; i++)
        newData[i] = data[i];
    delete[] data;
    data = newData;
    capacity = newCapacity;
}
```

---

## ➖ 5. 삭제(pop_back)

```cpp
void DynamicArray::pop_back() {
    if (length == 0)
        throw std::out_of_range("Array is empty");
    length--;

    // 메모리 줄이기 (optional)
    if (length <= capacity / 4 && capacity > 4)
        resize(capacity / 2);
}
```

---

## 🔧 6. 중간 삽입 및 삭제

```cpp
void DynamicArray::insert(int index, int val) {
    if (index < 0 || index > length)
        throw std::out_of_range("Index out of range");
    if (length == capacity)
        resize(capacity * 2);
    for (int i = length; i > index; i--)
        data[i] = data[i - 1];
    data[index] = val;
    length++;
}

void DynamicArray::erase(int index) {
    if (index < 0 || index >= length)
        throw std::out_of_range("Index out of range");
    for (int i = index; i < length - 1; i++)
        data[i] = data[i + 1];
    length--;
    if (length <= capacity / 4 && capacity > 4)
        resize(capacity / 2);
}
```

---

## 📤 7. 전체 출력 함수

```cpp
void DynamicArray::print() const {
    for (int i = 0; i < length; i++)
        std::cout << data[i] << " ";
    std::cout << "\n";
}
```

---

## ✅ 8. 사용 예제

```cpp
int main() {
    DynamicArray arr;
    arr.push_back(10);
    arr.push_back(20);
    arr.push_back(30);
    arr.insert(1, 15);  // 10 15 20 30
    arr.print();

    arr.erase(2);       // 10 15 30
    arr.print();

    arr.pop_back();     // 10 15
    arr.print();

    std::cout << "arr[1] = " << arr[1] << "\n";
}
```

---

## 📌 9. 정리

| 기능 | 시간 복잡도 (평균) |
|------|---------------------|
| 접근 | O(1) |
| 삽입/삭제 (끝) | O(1) ~ O(n) |
| 삽입/삭제 (중간) | O(n) |
| 자동 확장 | amortized O(1) |
