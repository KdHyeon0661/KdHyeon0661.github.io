---
layout: post
title: Data Structure - 동적 배열
date: 2024-12-08 20:20:23 +0900
category: Data Structure
---
# 직접 구현하는 동적 배열 (Dynamic Array) in C++
## 1. 동적 배열이란?

정적 배열과 달리, **요소 수에 맞춰 자동 확장/축소 가능**한 **연속 메모리** 기반 컨테이너입니다.

### 핵심 특성

- **랜덤 접근 O(1)** (`operator[]`, `data()`로 인덱싱)
- **끝 삽입 amortized O(1)** (재할당 발생 시 O(n))
- **중간 삽입/삭제 O(n)** (시프트 비용)
- **연속 메모리**로 캐시 친화적(실행시간 상수항이 작음)

---

## 2. 수학적 배경: 상환분석(Amortized Analysis)

용량을 매번 2배로 늘리는 정책에서, n번의 `push_back`에 대한 총 이동 비용은

$$
\sum_{i=0}^{\lfloor\log_2 n\rfloor} \frac{n}{2^i} \le 2n = O(n)
$$

이므로 **평균 1회 삽입 비용은 상수 시간**입니다:

$$
\text{amortized } O(1).
$$

> 성장 비율을 1.5로 줄이면 **메모리 오버헤드는 감소**하지만 **재할당 빈도**가 늘어 평균 상수항이 커질 수 있습니다. 반대로 2배는 재할당 빈도는 적으나 순간 피크 메모리가 더 큽니다.

---

## 3. 우리가 만들 클래스의 외형(요구사항 확장)

초안의 인터페이스를 **일반 템플릿 + 안전한 메모리 관리 + 반복자**로 확장합니다.

```cpp
// mini_vector.hpp
#pragma once
#include <memory>
#include <iterator>
#include <stdexcept>
#include <algorithm>
#include <initializer_list>
#include <type_traits>
#include <utility>

template <class T, class Alloc = std::allocator<T>>
class mini_vector {
public:
    using value_type             = T;
    using allocator_type         = Alloc;
    using size_type              = std::size_t;
    using difference_type        = std::ptrdiff_t;
    using reference              = value_type&;
    using const_reference        = const value_type&;
    using pointer                = typename std::allocator_traits<Alloc>::pointer;
    using const_pointer          = typename std::allocator_traits<Alloc>::const_pointer;
    using iterator               = value_type*;              // contiguous
    using const_iterator         = const value_type*;
    using reverse_iterator       = std::reverse_iterator<iterator>;
    using const_reverse_iterator = std::reverse_iterator<const_iterator>;

private:
    allocator_type alloc_;
    pointer data_ = nullptr;     // [data_, data_ + size_) : constructed
    size_type size_ = 0;
    size_type cap_  = 0;

public:
    // 3.1 생성/소멸
    mini_vector() noexcept(std::is_nothrow_default_constructible<Alloc>::value) = default;
    explicit mini_vector(size_type n, const T& val = T(), const Alloc& a = Alloc());
    explicit mini_vector(const Alloc& a) noexcept;
    mini_vector(std::initializer_list<T> il, const Alloc& a = Alloc());

    mini_vector(const mini_vector& other);
    mini_vector(const mini_vector& other, const Alloc& a);

    mini_vector(mini_vector&& other) noexcept;
    mini_vector(mini_vector&& other, const Alloc& a);

    ~mini_vector();

    // 3.2 대입/스왑
    mini_vector& operator=(const mini_vector& rhs);
    mini_vector& operator=(mini_vector&& rhs) noexcept(
        std::allocator_traits<Alloc>::propagate_on_container_move_assignment::value ||
        std::is_nothrow_move_assignable<T>::value);
    mini_vector& operator=(std::initializer_list<T> il);
    void swap(mini_vector& other) noexcept;

    // 3.3 용량
    size_type size() const noexcept { return size_; }
    size_type capacity() const noexcept { return cap_; }
    bool empty() const noexcept { return size_ == 0; }
    void reserve(size_type new_cap);
    void shrink_to_fit() noexcept; // best-effort
    void clear() noexcept;

    // 3.4 접근
    reference operator[](size_type i) noexcept { return data_[i]; }
    const_reference operator[](size_type i) const noexcept { return data_[i]; }
    reference at(size_type i);
    const_reference at(size_type i) const;
    reference front() { return data_[0]; }
    const_reference front() const { return data_[0]; }
    reference back() { return data_[size_-1]; }
    const_reference back() const { return data_[size_-1]; }
    pointer data() noexcept { return data_; }
    const_pointer data() const noexcept { return data_; }

    // 3.5 반복자
    iterator begin() noexcept { return data_; }
    const_iterator begin() const noexcept { return data_; }
    const_iterator cbegin() const noexcept { return data_; }
    iterator end() noexcept { return data_ + size_; }
    const_iterator end() const noexcept { return data_ + size_; }
    const_iterator cend() const noexcept { return data_ + size_; }
    reverse_iterator rbegin() noexcept { return reverse_iterator(end()); }
    const_reverse_iterator rbegin() const noexcept { return const_reverse_iterator(end()); }
    reverse_iterator rend() noexcept { return reverse_iterator(begin()); }
    const_reverse_iterator rend() const noexcept { return const_reverse_iterator(begin()); }

    // 3.6 수정자
    void push_back(const T& v);
    void push_back(T&& v);
    template <class... Args> reference emplace_back(Args&&... args);

    void pop_back();

    iterator insert(const_iterator pos, const T& v);
    iterator insert(const_iterator pos, T&& v);
    template <class... Args> iterator emplace(const_iterator pos, Args&&... args);

    iterator erase(const_iterator pos);
    iterator erase(const_iterator first, const_iterator last);

    void resize(size_type new_size);
    void resize(size_type new_size, const T& value);

private:
    // 내부 유틸
    void reallocate_strong(size_type new_cap);
    void destroy_range(pointer first, pointer last) noexcept;
    void deallocate_storage() noexcept;
    static size_type grow_capacity(size_type current, size_type need);
};
```

> **노트**
> - 반복자는 포인터여서 `std::vector`처럼 **contiguous iterator**를 가집니다.
> - `grow_capacity`에서 성장 정책(예: 2x, 혹은 1.5x)을 통일적으로 적용합니다.
> - `reallocate_strong`은 **강한 예외 안전 보장**을 제공하기 위해 새 버퍼에 **이동/복사-구성** 후 교체합니다.

---

## 4. 구현: 생성/소멸/대입/스왑

```cpp
// mini_vector_impl.hpp
#pragma once
#include "mini_vector.hpp"

template <class T, class Alloc>
mini_vector<T,Alloc>::mini_vector(size_type n, const T& val, const Alloc& a)
: alloc_(a), data_(nullptr), size_(0), cap_(0)
{
    if (n == 0) return;
    data_ = std::allocator_traits<Alloc>::allocate(alloc_, n);
    cap_ = n;
    pointer cur = data_;
    try {
        for (; size_ < n; ++size_, ++cur)
            std::allocator_traits<Alloc>::construct(alloc_, cur, val);
    } catch (...) {
        destroy_range(data_, cur);
        std::allocator_traits<Alloc>::deallocate(alloc_, data_, cap_);
        data_ = nullptr; size_ = cap_ = 0;
        throw;
    }
}

template <class T, class Alloc>
mini_vector<T,Alloc>::mini_vector(const Alloc& a) noexcept
: alloc_(a) {}

template <class T, class Alloc>
mini_vector<T,Alloc>::mini_vector(std::initializer_list<T> il, const Alloc& a)
: alloc_(a)
{
    reserve(il.size());
    for (const auto& x : il) push_back(x);
}

template <class T, class Alloc>
mini_vector<T,Alloc>::mini_vector(const mini_vector& other)
: alloc_(std::allocator_traits<Alloc>::
         select_on_container_copy_construction(other.alloc_))
{
    if (other.size_ == 0) return;
    data_ = std::allocator_traits<Alloc>::allocate(alloc_, other.size_);
    cap_ = other.size_;
    pointer cur = data_;
    try {
        for (; size_ < other.size_; ++size_, ++cur)
            std::allocator_traits<Alloc>::construct(alloc_, cur, other.data_[size_]);
    } catch (...) {
        destroy_range(data_, cur);
        std::allocator_traits<Alloc>::deallocate(alloc_, data_, cap_);
        data_ = nullptr; size_ = cap_ = 0;
        throw;
    }
}

template <class T, class Alloc>
mini_vector<T,Alloc>::mini_vector(const mini_vector& other, const Alloc& a)
: alloc_(a)
{
    if (other.size_ == 0) return;
    data_ = std::allocator_traits<Alloc>::allocate(alloc_, other.size_);
    cap_ = other.size_;
    pointer cur = data_;
    try {
        for (; size_ < other.size_; ++size_, ++cur)
            std::allocator_traits<Alloc>::construct(alloc_, cur, other.data_[size_]);
    } catch (...) {
        destroy_range(data_, cur);
        std::allocator_traits<Alloc>::deallocate(alloc_, data_, cap_);
        data_ = nullptr; size_ = cap_ = 0;
        throw;
    }
}

template <class T, class Alloc>
mini_vector<T,Alloc>::mini_vector(mini_vector&& other) noexcept
: alloc_(std::move(other.alloc_)), data_(other.data_), size_(other.size_), cap_(other.cap_)
{
    other.data_ = nullptr; other.size_ = other.cap_ = 0;
}

template <class T, class Alloc>
mini_vector<T,Alloc>::mini_vector(mini_vector&& other, const Alloc& a)
: alloc_(a)
{
    // allocator-propagation 정책 고려
    if (alloc_ == other.alloc_) {
        data_ = other.data_; size_ = other.size_; cap_ = other.cap_;
        other.data_ = nullptr; other.size_ = other.cap_ = 0;
    } else {
        // 다른 allocator면 move-construct
        reserve(other.size_);
        for (size_type i = 0; i < other.size_; ++i) push_back(std::move(other.data_[i]));
        other.clear();
    }
}

template <class T, class Alloc>
mini_vector<T,Alloc>::~mini_vector() {
    deallocate_storage();
}

template <class T, class Alloc>
mini_vector<T,Alloc>& mini_vector<T,Alloc>::operator=(const mini_vector& rhs) {
    if (this == &rhs) return *this;
    if constexpr (std::allocator_traits<Alloc>::propagate_on_container_copy_assignment::value) {
        if (alloc_ != rhs.alloc_) {
            deallocate_storage();
            alloc_ = rhs.alloc_;
        }
    }
    if (rhs.size_ > cap_) {
        mini_vector tmp(rhs, alloc_);
        swap(tmp);
    } else {
        // 재사용: 크기 조정
        size_type i = 0;
        // 공통 prefix 복사
        for (; i < std::min(size_, rhs.size_); ++i) data_[i] = rhs.data_[i];
        // 부족분 생성
        for (; i < rhs.size_; ++i)
            std::allocator_traits<Alloc>::construct(alloc_, data_ + i, rhs.data_[i]);
        // 초과분 파괴
        for (; i < size_; ++i)
            std::allocator_traits<Alloc>::destroy(alloc_, data_ + i);
        size_ = rhs.size_;
    }
    return *this;
}

template <class T, class Alloc>
mini_vector<T,Alloc>& mini_vector<T,Alloc>::operator=(mini_vector&& rhs) noexcept(
    std::allocator_traits<Alloc>::propagate_on_container_move_assignment::value ||
    std::is_nothrow_move_assignable<T>::value)
{
    if (this == &rhs) return *this;
    if constexpr (std::allocator_traits<Alloc>::propagate_on_container_move_assignment::value) {
        deallocate_storage();
        alloc_ = std::move(rhs.alloc_);
        data_ = rhs.data_; size_ = rhs.size_; cap_ = rhs.cap_;
        rhs.data_ = nullptr; rhs.size_ = rhs.cap_ = 0;
    } else {
        if (alloc_ == rhs.alloc_) {
            deallocate_storage();
            data_ = rhs.data_; size_ = rhs.size_; cap_ = rhs.cap_;
            rhs.data_ = nullptr; rhs.size_ = rhs.cap_ = 0;
        } else {
            // 다른 allocator면 move-assign by element
            clear();
            reserve(rhs.size_);
            for (size_type i=0; i<rhs.size_; ++i) push_back(std::move(rhs.data_[i]));
            rhs.clear();
        }
    }
    return *this;
}

template <class T, class Alloc>
mini_vector<T,Alloc>& mini_vector<T,Alloc>::operator=(std::initializer_list<T> il) {
    clear(); reserve(il.size());
    for (const auto& x : il) push_back(x);
    return *this;
}

template <class T, class Alloc>
void mini_vector<T,Alloc>::swap(mini_vector& other) noexcept {
    using std::swap;
    if constexpr (std::allocator_traits<Alloc>::propagate_on_container_swap::value) {
        swap(alloc_, other.alloc_);
    }
    swap(data_, other.data_);
    swap(size_, other.size_);
    swap(cap_,  other.cap_);
}
```

---

## 5. 접근/용량/수정자 구현

```cpp
template <class T, class Alloc>
typename mini_vector<T,Alloc>::reference
mini_vector<T,Alloc>::at(size_type i) {
    if (i >= size_) throw std::out_of_range("mini_vector::at");
    return data_[i];
}

template <class T, class Alloc>
typename mini_vector<T,Alloc>::const_reference
mini_vector<T,Alloc>::at(size_type i) const {
    if (i >= size_) throw std::out_of_range("mini_vector::at");
    return data_[i];
}

template <class T, class Alloc>
void mini_vector<T,Alloc>::reserve(size_type new_cap) {
    if (new_cap <= cap_) return;
    reallocate_strong(new_cap);
}

template <class T, class Alloc>
void mini_vector<T,Alloc>::shrink_to_fit() noexcept {
    if (size_ == cap_) return;
    try {
        reallocate_strong(size_);
    } catch (...) {
        // best-effort: 실패해도 상태 보존
    }
}

template <class T, class Alloc>
void mini_vector<T,Alloc>::clear() noexcept {
    destroy_range(data_, data_ + size_);
    size_ = 0;
}

template <class T, class Alloc>
void mini_vector<T,Alloc>::push_back(const T& v) {
    if (size_ == cap_) reserve(grow_capacity(cap_, size_ + 1));
    std::allocator_traits<Alloc>::construct(alloc_, data_ + size_, v);
    ++size_;
}

template <class T, class Alloc>
void mini_vector<T,Alloc>::push_back(T&& v) {
    if (size_ == cap_) reserve(grow_capacity(cap_, size_ + 1));
    std::allocator_traits<Alloc>::construct(alloc_, data_ + size_, std::move(v));
    ++size_;
}

template <class T, class Alloc>
template <class... Args>
typename mini_vector<T,Alloc>::reference
mini_vector<T,Alloc>::emplace_back(Args&&... args) {
    if (size_ == cap_) reserve(grow_capacity(cap_, size_ + 1));
    std::allocator_traits<Alloc>::construct(alloc_, data_ + size_, std::forward<Args>(args)...);
    ++size_;
    return back();
}

template <class T, class Alloc>
void mini_vector<T,Alloc>::pop_back() {
    if (size_ == 0) throw std::out_of_range("mini_vector::pop_back");
    --size_;
    std::allocator_traits<Alloc>::destroy(alloc_, data_ + size_);
    // 축소 정책: 여기서는 자동 축소는 하지 않음(힙 스래싱 방지)
}

template <class T, class Alloc>
typename mini_vector<T,Alloc>::iterator
mini_vector<T,Alloc>::insert(const_iterator pos, const T& v) {
    auto idx = static_cast<size_type>(pos - cbegin());
    if (size_ == cap_) reserve(grow_capacity(cap_, size_ + 1));
    // 뒤에서부터 한 칸씩 민 후 자리 생성
    if (idx < size_) {
        std::allocator_traits<Alloc>::construct(alloc_, data_ + size_, std::move(data_[size_-1]));
        for (size_type i = size_-1; i > idx; --i) data_[i] = std::move(data_[i-1]);
        data_[idx] = v;
    } else {
        std::allocator_traits<Alloc>::construct(alloc_, data_ + size_, v);
    }
    ++size_;
    return begin() + idx;
}

template <class T, class Alloc>
typename mini_vector<T,Alloc>::iterator
mini_vector<T,Alloc>::insert(const_iterator pos, T&& v) {
    auto idx = static_cast<size_type>(pos - cbegin());
    if (size_ == cap_) reserve(grow_capacity(cap_, size_ + 1));
    if (idx < size_) {
        std::allocator_traits<Alloc>::construct(alloc_, data_ + size_, std::move(data_[size_-1]));
        for (size_type i = size_-1; i > idx; --i) data_[i] = std::move(data_[i-1]);
        data_[idx] = std::move(v);
    } else {
        std::allocator_traits<Alloc>::construct(alloc_, data_ + size_, std::move(v));
    }
    ++size_;
    return begin() + idx;
}

template <class T, class Alloc>
template <class... Args>
typename mini_vector<T,Alloc>::iterator
mini_vector<T,Alloc>::emplace(const_iterator pos, Args&&... args) {
    auto idx = static_cast<size_type>(pos - cbegin());
    if (size_ == cap_) reserve(grow_capacity(cap_, size_ + 1));
    if (idx < size_) {
        std::allocator_traits<Alloc>::construct(alloc_, data_ + size_, std::move(data_[size_-1]));
        for (size_type i=size_-1; i>idx; --i) data_[i] = std::move(data_[i-1]);
        data_[idx].~T();
        std::allocator_traits<Alloc>::construct(alloc_, data_ + idx, std::forward<Args>(args)...);
    } else {
        std::allocator_traits<Alloc>::construct(alloc_, data_ + size_, std::forward<Args>(args)...);
    }
    ++size_;
    return begin() + idx;
}

template <class T, class Alloc>
typename mini_vector<T,Alloc>::iterator
mini_vector<T,Alloc>::erase(const_iterator pos) {
    auto idx = static_cast<size_type>(pos - cbegin());
    if (idx >= size_) return end();
    std::allocator_traits<Alloc>::destroy(alloc_, data_ + idx);
    for (size_type i = idx; i + 1 < size_; ++i) data_[i] = std::move(data_[i+1]);
    --size_;
    std::allocator_traits<Alloc>::destroy(alloc_, data_ + size_);
    return begin() + idx;
}

template <class T, class Alloc>
typename mini_vector<T,Alloc>::iterator
mini_vector<T,Alloc>::erase(const_iterator first, const_iterator last) {
    auto f = static_cast<size_type>(first - cbegin());
    auto l = static_cast<size_type>(last  - cbegin());
    if (f >= l || f >= size_) return begin() + std::min(f, size_);
    l = std::min(l, size_);

    // 파괴
    for (size_type i=f; i<l; ++i) std::allocator_traits<Alloc>::destroy(alloc_, data_ + i);
    // 앞으로 당김
    size_type cnt = l - f;
    for (size_type i=l; i<size_; ++i) data_[i - cnt] = std::move(data_[i]);

    // 뒤 꼬리 파괴
    for (size_type i=size_-cnt; i<size_; ++i)
        std::allocator_traits<Alloc>::destroy(alloc_, data_ + i);
    size_ -= cnt;
    return begin() + f;
}

template <class T, class Alloc>
void mini_vector<T,Alloc>::resize(size_type new_size) {
    if (new_size < size_) {
        for (size_type i=new_size; i<size_; ++i)
            std::allocator_traits<Alloc>::destroy(alloc_, data_ + i);
        size_ = new_size;
    } else if (new_size > size_) {
        reserve(new_size);
        for (; size_ < new_size; ++size_)
            std::allocator_traits<Alloc>::construct(alloc_, data_ + size_);
    }
}

template <class T, class Alloc>
void mini_vector<T,Alloc>::resize(size_type new_size, const T& value) {
    if (new_size < size_) {
        for (size_type i=new_size; i<size_; ++i)
            std::allocator_traits<Alloc>::destroy(alloc_, data_ + i);
        size_ = new_size;
    } else if (new_size > size_) {
        reserve(new_size);
        for (; size_ < new_size; ++size_)
            std::allocator_traits<Alloc>::construct(alloc_, data_ + size_, value);
    }
}
```

---

## 6. 내부 유틸: 재할당(강한 보장), 파괴/해제, 성장 정책

```cpp
template <class T, class Alloc>
void mini_vector<T,Alloc>::reallocate_strong(size_type new_cap) {
    pointer new_data = std::allocator_traits<Alloc>::allocate(alloc_, new_cap);
    size_type i = 0;
    try {
        for (; i < size_; ++i)
            std::allocator_traits<Alloc>::construct(
                alloc_, new_data + i, std::move_if_noexcept(data_[i]));
    } catch (...) {
        // 성공한 만큼만 파괴 후 새 버퍼 해제
        for (size_type j = 0; j < i; ++j)
            std::allocator_traits<Alloc>::destroy(alloc_, new_data + j);
        std::allocator_traits<Alloc>::deallocate(alloc_, new_data, new_cap);
        throw;
    }
    // 기존 파괴/해제
    destroy_range(data_, data_ + size_);
    if (data_) std::allocator_traits<Alloc>::deallocate(alloc_, data_, cap_);
    data_ = new_data; cap_ = new_cap;
}

template <class T, class Alloc>
void mini_vector<T,Alloc>::destroy_range(pointer first, pointer last) noexcept {
    for (; first != last; ++first)
        std::allocator_traits<Alloc>::destroy(alloc_, first);
}

template <class T, class Alloc>
void mini_vector<T,Alloc>::deallocate_storage() noexcept {
    destroy_range(data_, data_ + size_);
    if (data_) std::allocator_traits<Alloc>::deallocate(alloc_, data_, cap_);
    data_ = nullptr; size_ = cap_ = 0;
}

template <class T, class Alloc>
typename mini_vector<T,Alloc>::size_type
mini_vector<T,Alloc>::grow_capacity(size_type current, size_type need) {
    // 정책: max(need, max(2*current, 8))  (초기 최소 cap 8)
    size_type doubled = current ? current * 2 : size_type(8);
    if (doubled < need) doubled = need;
    return doubled;
}
```

---

## 7. 사용 예제 (기본 동작 확인)

```cpp
// main_basic.cpp
#include <iostream>
#include "mini_vector.hpp"
#include "mini_vector_impl.hpp"

int main() {
    mini_vector<int> v;
    for (int i=1;i<=5;i++) v.push_back(i*10); // 10 20 30 40 50
    auto it = v.insert(v.begin()+2, 25);      // 10 20 25 30 40 50
    v.erase(it+2);                            // 10 20 25 40 50
    v.emplace_back(60);                       // 10 20 25 40 50 60
    v.pop_back();                             // 10 20 25 40 50

    std::cout << "size=" << v.size()
              << " cap=" << v.capacity() << "\n";
    for (auto x: v) std::cout << x << " ";
    std::cout << "\n";

    v.shrink_to_fit();
    std::cout << "after shrink cap=" << v.capacity() << "\n";

    try {
        std::cout << v.at(100) << "\n"; // 예외 테스트
    } catch (const std::exception& e) {
        std::cout << "ex: " << e.what() << "\n";
    }
}
```

---

## 8. 예외 안전 보장(Strong vs Basic)

- `reallocate_strong`은 **새 버퍼에 먼저 성공적으로 구성**(construct) 후, **교체**하므로 **강한 보장**: 실패 시 **원래 컨테이너 불변**.
- `insert`/`erase`는 시프트 중 예외가 날 수 있으므로, 위처럼 **뒤에서 하나 구성 후 앞으로 move**하는 패턴을 사용해 **기본 보장** 또는 조건부 **강한 보장**을 근접하게 달성합니다.
- `push_back`/`emplace_back`은 재할당 시 `reallocate_strong`을 경유하여 **강한 보장**.

---

## 9. 반복자 무효화 규칙

- **재할당 발생 시**: 모든 반복자/참조/포인터 **무효화**.
- **중간 삽입/삭제**: 삽입 지점 이후의 반복자 **무효화** 가능.
- **끝 삽입(재할당 없음)**: `end()`만 변하고 기존 요소 참조는 유효.

> 실제 `std::vector`와 동일한 직관을 갖도록 설계했습니다.

---

## 10. 성능/캐시/성장 정책 논의

- **연속 메모리**는 연결 리스트보다 **캐시 지역성**이 훨씬 좋습니다. 단순한 `for` 루프 순회가 매우 빠릅니다.
- **성장 비율**: 2x는 재할당 빈도를 낮추고 상수항이 작아지지만 **메모리 오버헤드** 증가. 1.5x는 메모리 효율이 좋으나 재할당 빈도 증가.
  실무에서는 **작업 부하**(삽입 패턴, 최대 용량 예상, 메모리 한계)에 따라 선택합니다.
- **축소 정책**: `pop_back` 때마다 바로 `shrink`하면 **힙 스래싱**.
  대개 **명시적 `shrink_to_fit()`** 또는 **재할당 기준(1/4 이하)**을 신중 적용.

---

## 11. 고급: 사용자 정의 타입/Move/Noexcept

다음 타입을 삽입해 **move 경로**와 **예외 발생 시 롤백**을 실험할 수 있습니다.

```cpp
// types.hpp
#pragma once
#include <iostream>
#include <stdexcept>
struct Loud {
    int v;
    explicit Loud(int x=0) : v(x) { std::cout << "ctor " << v << "\n"; }
    Loud(const Loud& o) : v(o.v) { std::cout << "copy " << v << "\n"; }
    Loud(Loud&& o) noexcept : v(o.v) { std::cout << "move " << v << "\n"; o.v=0; }
    Loud& operator=(const Loud& o) { v=o.v; std::cout << "copy= " << v << "\n"; return *this; }
    Loud& operator=(Loud&& o) noexcept { v=o.v; o.v=0; std::cout << "move= " << v << "\n"; return *this; }
    ~Loud(){ std::cout << "dtor " << v << "\n"; }
};

struct ThrowOnCopy {
    int v;
    explicit ThrowOnCopy(int x=0) : v(x) {}
    ThrowOnCopy(const ThrowOnCopy&) { throw std::runtime_error("copy fail"); }
    ThrowOnCopy(ThrowOnCopy&&) noexcept = default;
    ThrowOnCopy& operator=(const ThrowOnCopy&) = delete;
    ThrowOnCopy& operator=(ThrowOnCopy&&) noexcept = default;
};
```

테스트:

```cpp
// main_move.cpp
#include "mini_vector.hpp"
#include "mini_vector_impl.hpp"
#include "types.hpp"

int main(){
    mini_vector<Loud> mv;
    mv.emplace_back(1);
    mv.emplace_back(2);
    mv.emplace_back(3);

    // 재할당 트리거
    mv.reserve(100);
    mv.emplace_back(4);

    mini_vector<ThrowOnCopy> tv;
    tv.emplace_back(1);
    tv.emplace_back(2);
    // 강한 보장 확인: 재할당 중 copy 생성자 대신 move 사용
    tv.reserve(64);
}
```

---

## 12. 단위 테스트 & 퍼징(경계/랜덤)

간단한 **브루트포스 대조**: 표준 `std::vector`와 결과 동일성 검증.

```cpp
// test_fuzz.cpp
#include <vector>
#include <random>
#include <cassert>
#include "mini_vector.hpp"
#include "mini_vector_impl.hpp"

int main(){
    std::mt19937 rng(1234);
    std::uniform_int_distribution<int> op(0,4), val(0,1000);

    mini_vector<int> mv;
    std::vector<int>  sv;

    for (int t=0; t<100000; ++t) {
        int o = op(rng);
        if (o==0) { // push_back
            int x = val(rng);
            mv.push_back(x); sv.push_back(x);
        } else if (o==1 && !sv.empty()) { // pop_back
            mv.pop_back(); sv.pop_back();
        } else if (o==2) { // insert at random pos
            int x = val(rng);
            size_t p = sv.empty()?0: std::uniform_int_distribution<size_t>(0, sv.size())(rng);
            mv.insert(mv.begin()+p, x);
            sv.insert(sv.begin()+p, x);
        } else if (o==3 && !sv.empty()) { // erase one
            size_t p = std::uniform_int_distribution<size_t>(0, sv.size()-1)(rng);
            mv.erase(mv.begin()+p);
            sv.erase(sv.begin()+p);
        } else { // reserve/reshrink
            size_t nc = std::uniform_int_distribution<size_t>(0, 1000)(rng);
            mv.reserve(nc);
            // sv에는 동등 동작 없음, 스킵
        }
        // 동등성 체크
        assert(mv.size() == sv.size());
        for (size_t i=0;i<sv.size();++i) assert(mv[i] == sv[i]);
    }
}
```

---

## 13. 복잡도 요약

| 연산 | 평균/상환 시간복잡도 | 최악 | 비고 |
|---|---|---|---|
| `operator[]`, `at` | $$O(1)$$ | $$O(1)$$ | 연속 메모리 |
| `push_back` | amortized $$O(1)$$ | $$O(n)$$ | 재할당 시 전체 move/copy |
| `pop_back` | $$O(1)$$ | $$O(1)$$ | 파괴만 수행 |
| `insert(pos,v)` | $$O(n)$$ | $$O(n)$$ | 시프트 비용 |
| `erase(pos)` | $$O(n)$$ | $$O(n)$$ | 시프트 비용 |
| `reserve(k)` | $$O(n)$$ | $$O(n)$$ | 재할당+이동 |
| `shrink_to_fit()` | $$O(n)$$ | $$O(n)$$ | best-effort |

---

## 14. 실무 체크리스트

1. **예외 안전**: 새 버퍼에서 먼저 **성공적 구성** → 실패 시 롤백.
2. **성장 정책**: 1.5x vs 2x — 작업 부하와 메모리 한도를 보고 결정.
3. **축소 정책**: 자동 축소는 신중. 일반적으론 명시적 `shrink_to_fit()`.
4. **반복자 무효화**: 재할당/삽입/삭제 시의 규칙을 문서화.
5. **move/noexcept**: 요소 타입이 `noexcept move`일수록 재할당 비용 ↓.
6. **테스트**: 경계·랜덤 퍼징·ASan/UBSan로 미검증 오류 제거.
7. **프로파일**: 캐시미스, 분기예측, 성장 비율별 상수항 비교.

---

## 15. 확장 과제

- **범위 삽입/삭제**(iterator 범위 인자)
- **SBO(Small Buffer Optimization)**: 작은 N에서 heap-free 최적화(문자열에서 흔함)
- **커스텀 Allocator**(풀/arena) 적용
- **동기화 래퍼**: 생산자 단일, 소비자 단일 등의 제한 하에 락 경량화
- **C++20**: `std::span`/`std::ranges` 친화 API

---

## 16. 결론

초안의 “단일형 `int` 동적 배열”을 **일반 템플릿 컨테이너**로 확장하여,
- 연속 메모리/상환분석,
- 예외 안전과 move,
- 반복자 모델과 무효화 규칙,
- 성장/축소 정책과 실무 트레이드오프
까지 **std::vector의 본질**을 바닥부터 재현했습니다.

이제 이 `mini_vector`로 각종 타입/워크로드에서 **재할당 빈도/시간/메모리**를 **실측**해보면,
표준 컨테이너 선택과 튜닝의 감각이 훨씬 빨라집니다.

---

## 17. 부록: 전체 빌드 스크립트 & 실행

```bash
g++ -std=c++17 -O2 -Wall -Wextra -pedantic main_basic.cpp -o basic
./basic

g++ -std=c++17 -O2 -Wall -Wextra -pedantic main_move.cpp -o move
./move

g++ -std=c++17 -O2 -fsanitize=address,undefined -g test_fuzz.cpp -o fuzz
./fuzz
```

- 실패/경계 상황에서의 동작을 **ASan/UBSan**으로 점검하세요.

---

## 18. 요약 표 (핵심 규칙)

- **재할당은 강한 보장**으로 설계하라(새 버퍼 성공 → 교체).
- **성장 정책은 한 곳**(`grow_capacity`)에서 통제하라.
- **연속 메모리**는 생각보다 상수항 이득이 크다(캐시).
- **반복자 무효화**는 반드시 문서화/주석화하라.
- **테스트는 브루트포스 + 퍼징**으로: “실제로 깨먹어 보자”.
