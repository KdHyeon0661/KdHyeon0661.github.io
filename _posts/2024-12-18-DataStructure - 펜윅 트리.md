---
layout: post
title: Data Structure - 펜윅 트리
date: 2024-12-18 19:20:23 +0900
category: Data Structure
---
# 펜윅 트리 (Fenwick Tree, BIT)

## 개요와 동기

**펜윅 트리(Fenwick Tree)** 또는 **BIT(Binary Indexed Tree)**는
**누적 합(prefix sum)**·**구간 합(range sum)**·**빈도 누적** 등을 **\(O(\log n)\)** 으로 처리하는 **배열 기반 트리**다.

- 단순 **누적 합 배열**은 질의 \(O(1)\)이지만 **갱신이 \(O(n)\)**.
- **세그먼트 트리**는 질의·갱신 모두 \(O(\log n)\)이지만 구현이 비교적 무겁다.
- **Fenwick**은 **가볍고 코드가 짧으면서** 질의·갱신 모두 \(O(\log n)\). 메모리도 적게 든다.

**주요 응용**
- 누적 합/구간 합, 누적 카운트(빈도)
- **인버전 수** 계산
- **온라인 순위/질의 응답**(예: k번째 원소 찾기)
- **좌표 압축**과 함께 큰 값 범위 처리
- 2D 누적 합(행렬 구간 합)까지 확장 가능

---

## 핵심 아이디어 — LSB로 덩어리 묶기

- BIT는 **인덱스 i의 최하위 1비트(LSB)** 길이만큼을 **하나의 덩어리**로 관리한다.
- 비트 연산 **`i & -i`** 는 정수 \(i\)의 **LSB(2-adic 분해)** 를 반환:

  $$
  \operatorname{LSB}(i) = i \,\&\, (-i).
  $$

  예) \( i=12_{(2)}=1100_2 \Rightarrow i\&-i = 0100_2=4 \).

- **prefix sum** \([1..i]\)를 계산할 때, \(i\)에서 시작해
  \(i -= \operatorname{LSB}(i)\)로 **상위 블록으로 점프**하면서 덩어리 합을 더한다.
- **update**는 반대로 \(i\)에서 시작해
  \(i += \operatorname{LSB}(i)\)로 **상위 덩어리들에 변화 전파**.

이 설계로 질의·갱신이 모두 \(O(\log n)\)에 끝난다.

---

## 인터페이스 설계 — 1-based vs 0-based

- 전통 구현은 **1-based 인덱스**를 쓴다. (가장 간단)
- 0-based가 꼭 필요하면 **내부는 1-based로, 외부에서 ±1 시프트** 래핑을 권장.

**수 자료형**
- 합/누적은 쉽게 **int를 넘친다**. 실전은 **`long long`** 또는 **모듈러**를 사용.

---

## 기본 템플릿 구현 (C++17)

### 단일 트리: 점 갱신 + prefix/구간 합

```cpp
```cpp
#include <bits/stdc++.h>

using namespace std;

template <class T = long long>
struct Fenwick {
    int n;
    vector<T> bit; // 1-based

    explicit Fenwick(int n = 0) { init(n); }

    void init(int n_) {
        n = n_;
        bit.assign(n + 1, T(0));
    }

    // arr[i] += delta  (1-based i)
    void add(int i, T delta) {
        for (; i <= n; i += (i & -i)) bit[i] += delta;
    }

    // sum of arr[1..i]
    T sumPrefix(int i) const {
        T s = 0;
        for (; i > 0; i -= (i & -i)) s += bit[i];
        return s;
    }

    // sum of arr[l..r] (1 <= l <= r <= n)
    T sumRange(int l, int r) const {
        return sumPrefix(r) - sumPrefix(l - 1);
    }

    // O(n) 빌드: arr는 1-based로 전달 (arr[0] dummy)
    void build(const vector<T>& arr) {
        init((int)arr.size() - 1);
        for (int i = 1; i <= n; ++i) bit[i] += arr[i];
        for (int i = 1; i <= n; ++i) {
            int j = i + (i & -i);
            if (j <= n) bit[j] += bit[i];
        }
    }
};
```
```

**포인트**
- `build()`는 **O(n)** 에 초기화(클래식) → 초기 로딩이 많은 경우 유용.
- 그 외 일반적 초기화는 `add(i, arr[i])`를 \(i=1..n\)에 수행(총 \(O(n\log n)\)).

### 사용 예

```cpp
```cpp
int main() {
    Fenwick<long long> ft(10);
    vector<long long> arr = {0, 3, 2, -1, 6, 5, 4, -3, 3, 7, 2}; // 1-based

    // O(n) build
    ft.build(arr);

    cout << ft.sumPrefix(5) << "\n";        // 3+2-1+6+5 = 15
    cout << ft.sumRange(3, 7) << "\n";      // -1+6+5+4-3 = 11

    ft.add(4, 3); // arr[4] += 3  (6 -> 9)
    cout << ft.sumPrefix(5) << "\n";        // 18

    return 0;
}
```
```

---

## 왜 맞는가? — 분해 아이디어 스냅샷

정수 \(i\)를 이진 전개하면,
\[
i = \sum_{k} 2^{b_k} \quad(b_k\text{는 켜진 비트 위치}).
\]
BIT는 각 \(i\)에 대해 길이 \(\operatorname{LSB}(i)\)의 구간 **\([i-\operatorname{LSB}(i)+1,\, i]\)** 합을 저장한다.
따라서 prefix 합은 다음 블록들의 합으로 분해된다:
\[
\sum_{x=1}^{i} a_x \;=\;
\sum_{\text{경로 }i\to 0} \text{block}(i).
\]
업데이트는 반대로 **해당 원소를 포함하는 모든 상위 블록**에 동일 델타를 더한다.

---

## 변형 A — **구간 덧셈 + 점 질의**(RUPQ)

- 목표: \([l, r]\)에 \(+v\)를 더하고, 최종 배열 \(A\)의 **점 값 \(A[i]\)** 를 즉시 질의.
- 트릭: 차분 배열 \(D\)를 사용하면 \(A = \operatorname{prefix}(D)\).
  - \([l,r]\)에 \(+v\) 추가는 \(D[l]+=v, D[r+1]-=v\).
  - 그러면 \(A[i] = \sum_{x=1}^{i} D[x]\) ⇒ **BIT 하나로 점 질의** 가능.

```cpp
```cpp
template <class T = long long>
struct FenwickRUPQ {
    int n; Fenwick<T> fw; // D에 대한 fenwick
    FenwickRUPQ(int n=0): n(n), fw(n) {}

    // add v on [l..r]
    void rangeAdd(int l, int r, T v) {
        fw.add(l, v);
        if (r+1 <= n) fw.add(r+1, -v);
    }

    // point value A[i]
    T pointQuery(int i) const {
        return fw.sumPrefix(i);
    }
};
```
```

---

## 변형 B — **구간 덧셈 + 구간 합 질의**(RURQ) (BIT 2개)

유명 공식:
\[
\text{prefix}(x) = \big(\sum B_1[1..x]\big)\cdot x - \sum B_2[1..x].
\]
- \([l,r]\)에 \(+v\) 추가 시
  - \(B_1[l]+=v,\; B_1[r+1]-=v\)
  - \(B_2[l]+=v\cdot(l-1),\; B_2[r+1]-=v\cdot r\)

```cpp
```cpp
template <class T = long long>
struct FenwickRURQ {
    int n; Fenwick<T> B1, B2;
    FenwickRURQ(int n=0): n(n), B1(n), B2(n) {}

    void _add(Fenwick<T>& F, int i, T v) { F.add(i, v); }

    // add v on [l..r]
    void rangeAdd(int l, int r, T v) {
        _add(B1, l, v);
        if (r+1<=n) _add(B1, r+1, -v);
        _add(B2, l, v*(l-1));
        if (r+1<=n) _add(B2, r+1, -v*r);
    }

    T prefixSum(int x) const {
        auto s1 = B1.sumPrefix(x);
        auto s2 = B2.sumPrefix(x);
        return s1 * x - s2;
    }

    // sum of A[l..r]
    T rangeSum(int l, int r) const {
        return prefixSum(r) - prefixSum(l-1);
    }
};
```
```

---

## 2D Fenwick — 행렬 구간 합

- 2D 배열 \(A[r][c]\)에 대해 **점 업데이트 + 직사각형 합** 또는
  **직사각형 업데이트 + 점/직사각형 질의**를 2D BIT로 확장 가능.
- 핵심은 인덱스 두 축에 대해 **LSB 이동을 중첩**:

```cpp
```cpp
template <class T = long long>
struct Fenwick2D {
    int n, m;
    vector<vector<T>> bit; // 1-based [1..n][1..m]
    Fenwick2D(int n=0, int m=0): n(n), m(m), bit(n+1, vector<T>(m+1, T(0))) {}

    void add(int r, int c, T delta) {
        for (int i=r; i<=n; i+= (i & -i))
            for (int j=c; j<=m; j+= (j & -j))
                bit[i][j] += delta;
    }

    T sumPrefix(int r, int c) const {
        T s=0;
        for (int i=r; i>0; i-= (i & -i))
            for (int j=c; j>0; j-= (j & -j))
                s += bit[i][j];
        return s;
    }

    T sumRect(int r1, int c1, int r2, int c2) const {
        return sumPrefix(r2,c2) - sumPrefix(r1-1,c2) - sumPrefix(r2,c1-1) + sumPrefix(r1-1,c1-1);
    }
};
```
```

---

## 순서 통계 — **k번째(이하 누적) 찾기**

BIT를 **빈도 배열**로 쓰면,
`find_kth(k)` (누적 합이 처음으로 \( \ge k \)가 되는 최소 인덱스)를 **이진 리프 보행**으로 \(O(\log n)\)에 찾는다.

```cpp
```cpp
// bit[]는 각 인덱스의 덩어리 합(빈도 누적)을 보유 (1-based)
int find_kth(const Fenwick<long long>& fw, long long k) {
    int pos = 0;
    // maxPow: n 이하의 최대 2^p
    int maxPow = 1;
    while ((maxPow<<1) <= fw.n) maxPow <<= 1;

    for (int step = maxPow; step > 0; step >>= 1) {
        int next = pos + step;
        if (next <= fw.n && fw.bit[next] < k) {
            k -= fw.bit[next];
            pos = next;
        }
    }
    return pos + 1; // 첫 인덱스 with prefix >= original k
}
```
```

**용도**
- 온라인 **순위표**: 점수 빈도 BIT → k등의 점수 위치
- 동적 멀티셋의 **order_of_key** / **find_by_order** 유사 기능

---

## 인버전 수 — 좌표 압축과 BIT

배열 \(A\)의 인버전 수는
\[
\#\{(i,j)\mid i<j,\ A[i] > A[j]\}
\]
- 값 범위가 크면 **좌표 압축** 먼저 수행.
- 왼→오른으로 순회하며, **현재보다 큰 값** 개수 합산:
  - BIT에 **등장 빈도**를 누적.
  - 현재 \(A[i]\)보다 큰 값 개수 = `totalSoFar - sumPrefix(rank(A[i]))`.

```cpp
```cpp
long long inversion_count(vector<int> a) {
    // 1) 좌표 압축
    vector<int> sorted = a;
    sort(sorted.begin(), sorted.end());
    sorted.erase(unique(sorted.begin(), sorted.end()), sorted.end());
    auto rid = [&](int x){
        return int(lower_bound(sorted.begin(), sorted.end(), x) - sorted.begin()) + 1; // 1-based
    };

    int n = (int)a.size();
    Fenwick<long long> fw((int)sorted.size());
    long long inv = 0;

    long long seen = 0;
    for (int x : a) {
        int r = rid(x);
        long long le = fw.sumPrefix(r);         // <= x
        inv += (seen - le);                      // > x
        fw.add(r, 1);
        ++seen;
    }
    return inv;
}
```
```

---

## 0-based 래핑, 모듈러, 실전 유틸

### 0-based 외부 API

```cpp
```cpp
struct Fenwick0 {
    Fenwick<long long> fw;
    Fenwick0(int n=0): fw(n) {}
    // add on a[idx] += delta  (0-based)
    void add(int idx, long long delta){ fw.add(idx+1, delta); }
    // sum of a[0..idx]
    long long sumPrefix(int idx){ return fw.sumPrefix(idx+1); }
    // sum of a[l..r]
    long long sumRange(int l, int r){ return fw.sumRange(l+1, r+1); }
};
```
```

### 모듈러 연산 (예: \(10^9+7\))

- 누적 시 `% MOD` 적용
- 음수 델타는 `(x%MOD+MOD)%MOD` normalize

---

## 디버깅/테스트 체크리스트

- **경계**: `l=1`, `r=n`, 공배열, 단일 원소
- **자료형**: 합이 `int` 범위를 초과 → `long long`
- **1-based** 실수: `bit[0]` 접근 금지
- **range API**: 포함 범위 `[l..r]` 일치 확인
- **build() vs add**: 초기화 경로 혼합 금지
- **find_kth**: `k` 범위(1..total) 사전검증
- **2D**: 이중 for의 증감 방향을 정확히

**간이 퍼징**
- 난수로 `add`/`sumRange`를 반복해 **브루트 포스(prefix 배열)**와 결과 비교

---

## 복잡도/공간/캐시

- 시간: 모든 변형이 **\(O(\log n)\)**
- 초기화: `build()`는 **\(O(n)\)**, `n`번 `add`는 **\(O(n\log n)\)**
- 메모리: `n+1` 크기 벡터(상수 계수 작음)
- 캐시 지역성: **연속 메모리**라 세그트리보다 캐시 친화적

---

## 예제 — 통합 데모

```cpp
```cpp
#include <bits/stdc++.h>

using namespace std;

// 앞서 정의한 Fenwick / FenwickRUPQ / FenwickRURQ / find_kth / inversion_count를 가정

int main(){
    // 1) 기본 합
    Fenwick<long long> ft(10);
    for(int i=1;i<=10;++i) ft.add(i, i); // a[i] = i
    cout << "sumPrefix(10)=" << ft.sumPrefix(10) << "\n";  // 55
    cout << "sumRange(4,7)=" << ft.sumRange(4,7) << "\n";  // 4+5+6+7=22

    // 2) 구간 덧셈 + 점 질의 (RUPQ)
    FenwickRUPQ<long long> rupq(8);
    rupq.rangeAdd(2,5, 10);  // a[2..5]+=10
    rupq.rangeAdd(4,7, -3);
    cout << "point(3)=" << rupq.pointQuery(3) << "\n";  // 10
    cout << "point(6)=" << rupq.pointQuery(6) << "\n";  // -3

    // 3) 구간 덧셈 + 구간 합 (RURQ)
    FenwickRURQ<long long> rurq(8);
    rurq.rangeAdd(2,5, 10);
    rurq.rangeAdd(4,7, -3);
    cout << "sum(1..8)=" << rurq.rangeSum(1,8) << "\n";
    cout << "sum(3..6)=" << rurq.rangeSum(3,6) << "\n";

    // 4) k번째 찾기 (빈도 누적)
    Fenwick<long long> freq(10);
    // {1:2회, 3:1회, 5:3회, 9:1회} 라고 하자.
    freq.add(1,2); freq.add(3,1); freq.add(5,3); freq.add(9,1);
    auto kth = [&](long long k){
        // find_kth에서 bit 벡터 접근 필요 → 같은 번들에 넣어 사용 가정
        // 여기선 간략하게 내부 접근 허용 또는 find_kth를 friend로 선언했다고 치자.
        int pos = 0;
        int maxPow = 1; while ((maxPow<<1) <= freq.n) maxPow <<= 1;
        long long kk = k;
        for (int step=maxPow; step>0; step>>=1) {
            int next = pos + step;
            if (next <= freq.n && freq.bit[next] < kk) {
                kk -= freq.bit[next];
                pos = next;
            }
        }
        return pos+1;
    };
    cout << "k=1 -> idx " << kth(1) << "\n"; // 1
    cout << "k=3 -> idx " << kth(3) << "\n"; // 3
    cout << "k=5 -> idx " << kth(5) << "\n"; // 5

    // 5) 인버전 수
    vector<int> a = {3, 4, 1, 2, 5};
    cout << "inversions=" << inversion_count(a) << "\n"; // 3:(3,1),(3,2),(4,1),(4,2) → 4개? 실제 계산: (3,1),(3,2),(4,1),(4,2)=4

    return 0;
}
```
```

> 주: 위 인버전 예시는 실제로 **4**가 맞다(출력 확인 권장).

---

## 수학 스냅샷 — LSB와 덩어리 증명

- 각 인덱스 \(i\)는 길이 \(\operatorname{LSB}(i)\) 구간 \([i-\operatorname{LSB}(i)+1, i]\)에 대한 합을 저장.
- 임의의 \(i\)에 대해
  \[
  i \to i-\operatorname{LSB}(i) \to i-\operatorname{LSB}(i)-\operatorname{LSB}(i-\operatorname{LSB}(i)) \to \cdots \to 0
  \]
  경로가 정확히 **상호소 구간 분할**을 만든다. 그러므로 **중복/누락 없이** \([1..i]\)가 합쳐진다.
- 업데이트는 상위 구간의 대표 인덱스들이 모두 **그 원소를 포함**하므로 동일 델타 합산이 정확하다.

---

## 실전 팁/버그 포인트

1. **1-based 강제**: `i=0`에서 `i&-i=0` 무한루프에 걸리지 않게 하라.
2. **자료형**: 합계가 큰 경우 **`long long`** 으로.
3. **범위 포함**: `sumRange(l,r)=sumPrefix(r)-sumPrefix(l-1)` 일관성 유지.
4. **build** vs **add 루프** 섞지 않기. (중복 누적 주의)
5. **find_kth**: `k`가 누적 총합을 넘지 않도록 입력 방어.
6. **2D**: 이중 for의 `+=LSB` / `-=LSB` 방향을 뒤섞지 말 것.
7. **좌표 압축**: 값 범위가 **1e9**급이면 반드시 압축 후 사용.

---

## 연습 과제(권장)

1. **RUPQ**와 **RURQ**를 하나의 클래스로 일반화하라(정적 bool 플래그 or 정책 클래스).
2. `find_kth`를 **템플릿 멤버**로 안전하게 제공하고, k 범위 예외 처리 추가.
3. 2D Fenwick으로 **직사각형 업데이트 + 직사각형 합**(BIT 4개)을 구현하라.
4. **모듈러** 버전을 작성하고 랜덤 퍼징으로 O(10^5) 연산을 검증하라.
5. 온라인으로 들어오는 `(type, l, r, v)` 질의를 **좌표 압축** 후 처리하라.

---

## 요약

| 항목 | 핵심 |
|---|---|
| 구조 | 배열 기반 + LSB로 덩어리 관리 |
| 시간 | 질의·갱신 모두 \(O(\log n)\) |
| 초기화 | `build()` \(O(n)\) 또는 `add` 루프 \(O(n\log n)\) |
| 변형 | RUPQ(구간+=v/점질의), RURQ(구간+=v/구간합) = **BIT 2개** |
| 고급 | 2D 확장, `find_kth`, 좌표 압축, 인버전 수, 온라인 순위 |
| 주의 | 1-based, 자료형, 범위 포함, 빌드·갱신 혼합 금지 |

위의 템플릿과 변형들을 익히면 **세그먼트 트리보다 가벼운 많은 누적형 문제**를 효율적으로 풀 수 있다.
현업/대회 모두에서 여전히 강력한 **실전 도구**다.
