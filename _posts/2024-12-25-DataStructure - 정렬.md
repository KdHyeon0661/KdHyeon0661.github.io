---
layout: post
title: Data Structure - 정렬
date: 2024-12-25 19:20:23 +0900
category: Data Structure
---
# 정렬(Sorting) 알고리즘

## 0. 정렬이 왜 중요한가 — 하한선과 문제 정의

정렬은 **순서가 없는 데이터**를 특정 **전순서(total order)** 혹은 **엄격 약순서(strict weak ordering)** 로 배치하는 문제다.
비교 기반 정렬은 **결정 트리 모델** 하에서 최악 시간복잡도의 하한이 존재한다.

- 가능한 순열: \( n! \)
- 결정 트리 높이 \( h \) 는 모든 순열을 식별해야 하므로 \( 2^h \ge n! \)
- 따라서
  $$
  h \ge \log_2(n!) \approx n\log_2 n - 1.4427\,n + O(\log n)
  $$
  (스털링 근사)

**결론:** 비교 기반 정렬의 **최악 복잡도 하한**은 \( \Omega(n\log n) \).
→ 평균·최악 모두 \(O(n\log n)\)을 달성하는 **힙/병합/인트로 정렬**이 실전 기준선이 된다.

---

## 1. 용어·속성 체크리스트

- **안정성(stability)**: 동치 키의 **상대 순서 보존** 여부
- **제자리(in-place)**: 추가 메모리 \(O(1)\) (스택은 \(O(\log n)\) 허용 관행)
- **분할 정복**: 서브문제로 분해 후 병합
- **비교 기반 vs 비비교 기반**: 전자는 하한 \( \Omega(n\log n) \), 후자는 **도메인 제약**(정수 범위 등)으로 \(O(n)\)도 가능

**C++ 비교자 요구:** `StrictWeakOrdering`
- 반대칭/추이성/추이적 비교 가능성(삼항관계) 충족
- `!(a<b) && !(b<a)` 를 **동치**로 간주한다

---

## 2. O(n²) 기초 정렬 — 학습용 + 소규모 최적화

### 2.1 선택 정렬 (Selection Sort) — 스왑 최소
```cpp
#include <vector>
#include <utility>
void selectionSort(std::vector<int>& a){
    const int n = (int)a.size();
    for(int i=0;i<n-1;++i){
        int m=i;
        for(int j=i+1;j<n;++j) if(a[j]<a[m]) m=j;
        if(m!=i) std::swap(a[i],a[m]);
    }
}
```
- 비교 \( \Theta(n^2) \), 스왑 \( \le n-1 \)
- **안정성 X**, **제자리 O**

### 2.2 버블 정렬 (Bubble) — 조기 종료 최적화
```cpp
#include <vector>
#include <utility>
void bubbleSort(std::vector<int>& a){
    int n=(int)a.size();
    bool swapped=true;
    while(swapped){
        swapped=false;
        for(int i=1;i<n;i++){
            if(a[i-1]>a[i]){ std::swap(a[i-1],a[i]); swapped=true; }
        }
        --n; // 마지막 원소는 고정됨
    }
}
```
- 최선 \(O(n)\) (이미 정렬), 평균/최악 \(O(n^2)\)
- **안정성 O**, **제자리 O**

### 2.3 삽입 정렬 (Insertion) — 작거나 거의 정렬된 입력에 강함
```cpp
#include <vector>
void insertionSort(std::vector<int>& a){
    for(int i=1;i<(int)a.size();++i){
        int key=a[i], j=i-1;
        while(j>=0 && a[j]>key){ a[j+1]=a[j]; --j; }
        a[j+1]=key;
    }
}
```
- 최선 \(O(n)\): 거의 정렬
- **안정성 O**, **제자리 O**

#### (옵션) 이진 삽입 정렬 — 비교 수 감소
```cpp
#include <vector>
int lowerBound(const std::vector<int>& a, int hi, int key){
    int lo=0;
    while(lo<hi){
        int mid=(lo+hi)>>1;
        if(a[mid]<key) lo=mid+1; else hi=mid;
    }
    return lo;
}
void binaryInsertionSort(std::vector<int>& a){
    for(int i=1;i<(int)a.size();++i){
        int key=a[i], pos=lowerBound(a,i,key);
        for(int j=i;j>pos;--j) a[j]=a[j-1];
        a[pos]=key;
    }
}
```

### 2.4 셸 정렬 (Shell Sort) — O(n^1.2~1.5) 실전 성능
```cpp
#include <vector>
void shellSort(std::vector<int>& a){
    int n=(int)a.size();
    // Knuth 간격: 1,4,13,40,...
    int h=1; while(h<n/3) h=3*h+1;
    while(h>=1){
        for(int i=h;i<n;++i){
            int key=a[i], j=i;
            while(j>=h && a[j-h]>key){ a[j]=a[j-h]; j-=h; }
            a[j]=key;
        }
        h/=3;
    }
}
```
- **안정성 X(일반적으로)**, **제자리 O**

---

## 3. O(n log n) 정렬 — 실전 주력군

### 3.1 병합 정렬 (Merge Sort) — 안정 + 외부정렬 친화
**Top-Down (재귀)**
```cpp
#include <vector>
template<class It>
void merge(It l, It m, It r, std::vector<typename It::value_type>& buf){
    auto i=l, j=m; auto k=buf.begin();
    while(i<m && j<r) *k++ = (*j < *i) ? *j++ : *i++; // 안정성 보장(<=)
    k = std::copy(i,m,k); std::copy(j,r,k);
    std::move(buf.begin(), buf.begin()+(r-l), l);
}
template<class It>
void mergeSort(It l, It r){
    const auto n = r-l; if(n<=32){ // 작은 구간 삽입정렬로 하이브리드
        for(It i=l+1;i<r;++i){ auto key=*i; It j=i;
            while(j>l && *(j-1)>key){ *j=*(j-1); --j; } *j=key; }
        return;
    }
    It m = l + n/2;
    mergeSort(l,m); mergeSort(m,r);
    std::vector<typename It::value_type> buf(n);
    merge(l,m,r,buf);
}
```
- **항상 \(O(n\log n)\)**, **안정 O**, **추가 메모리 O(n)**

**Bottom-Up (반복)**
```cpp
#include <vector>
template<class T>
void mergeBottomUp(std::vector<T>& a){
    int n=(int)a.size();
    std::vector<T> buf(n);
    for(int sz=1; sz<n; sz<<=1){
        for(int l=0; l<n-sz; l+=sz<<1){
            int m=l+sz, r=std::min(l+(sz<<1), n);
            int i=l, j=m, k=l;
            while(i<m && j<r) buf[k++] = (a[j] < a[i]) ? a[j++] : a[i++];
            while(i<m) buf[k++]=a[i++]; while(j<r) buf[k++]=a[j++];
            for(int t=l;t<r;++t) a[t]=std::move(buf[t]);
        }
    }
}
```

### 3.2 힙 정렬 (Heap Sort) — 제자리, 최악도 \(O(n\log n)\)
```cpp
#include <vector>
#include <utility>
void siftDown(std::vector<int>& a, int n, int i){
    while(true){
        int l=2*i+1, r=2*i+2, big=i;
        if(l<n && a[l]>a[big]) big=l;
        if(r<n && a[r]>a[big]) big=r;
        if(big==i) break;
        std::swap(a[i],a[big]); i=big;
    }
}
void heapSort(std::vector<int>& a){
    int n=(int)a.size();
    for(int i=n/2-1;i>=0;--i) siftDown(a,n,i);
    for(int i=n-1;i>0;--i){
        std::swap(a[0],a[i]); siftDown(a,i,0);
    }
}
```
- **안정성 X**, **제자리 O**, **브랜치 예측에 다소 불리**

### 3.3 퀵 정렬(Three-Way + Tail Call) — 평균 최강, 나쁜 피벗 방지
```cpp
#include <vector>
#include <utility>
#include <algorithm>

// Dijkstra 3-way partition: 중복 키에 강함
std::pair<int,int> partition3(std::vector<int>& a, int l, int r){
    int i=l, lt=l, gt=r;
    int pivot=a[l + (r-l)/2];
    while(i<=gt){
        if(a[i]<pivot) std::swap(a[i++],a[lt++]);
        else if(a[i]>pivot) std::swap(a[i],a[gt--]);
        else ++i;
    }
    return {lt, gt};
}

void quickSort3(std::vector<int>& a, int l, int r){
    while(l<r){
        if(r-l+1<=24){ // 작은 구간 삽입정렬 전환
            for(int i=l+1;i<=r;++i){
                int key=a[i], j=i-1;
                while(j>=l && a[j]>key){ a[j+1]=a[j]; --j; }
                a[j+1]=key;
            }
            return;
        }
        auto [m1,m2]=partition3(a,l,r);
        // 꼬리재귀 제거: 더 작은 쪽 먼저 처리
        if(m1-l < r-m2){
            quickSort3(a,l,m1-1);
            l=m2+1;
        }else{
            quickSort3(a,m2+1,r);
            r=m1-1;
        }
    }
}
```
- 평균 \(O(n\log n)\), 최악 \(O(n^2)\) → **인트로소트**로 보강

### 3.4 인트로소트(IntroSort) — `std::sort`의 핵심 아이디어
- 시작은 **퀵정렬**, 재귀 깊이가 \(2\lfloor \log_2 n\rfloor\) 를 넘으면 **힙정렬**로 전환 (최악 \(O(n\log n)\) 보장)
- 소구간은 **삽입정렬**로 마무리 → 실전 최강 하이브리드

#### 인트로소트 스케치
```cpp
#include <vector>
#include <cmath>
#include <utility>

int median3(std::vector<int>& a, int l, int m, int r){
    if(a[l]>a[m]) std::swap(a[l],a[m]);
    if(a[m]>a[r]) std::swap(a[m],a[r]);
    if(a[l]>a[m]) std::swap(a[l],a[m]);
    return m;
}

int partition(std::vector<int>& a, int l, int r){
    int m = median3(a,l,l+(r-l)/2,r);
    int pivot=a[m]; std::swap(a[m],a[r]);
    int i=l;
    for(int j=l;j<r;++j) if(a[j]<pivot) std::swap(a[i++],a[j]);
    std::swap(a[i],a[r]); return i;
}

void heapify(std::vector<int>& a, int n, int i){
    while(true){
        int l=2*i+1, r=2*i+2, big=i;
        if(l<n && a[l]>a[big]) big=l;
        if(r<n && a[r]>a[big]) big=r;
        if(big==i) break; std::swap(a[i],a[big]); i=big;
    }
}
void heapSortRange(std::vector<int>& a, int l, int r){
    int n=r-l+1;
    for(int i=l+n/2-1;i>=l;--i) heapify(a, r-l+1, i-l); // 인덱스 대응 주의
    for(int i=r;i>l;--i){
        std::swap(a[l],a[i]);
        // 부분 힙화: 편의상 임시 배열 사용하는 구현이 쉬움(여기선 스케치)
    }
}

void insertionRange(std::vector<int>& a, int l, int r){
    for(int i=l+1;i<=r;++i){
        int key=a[i], j=i-1;
        while(j>=l && a[j]>key){ a[j+1]=a[j]; --j; }
        a[j+1]=key;
    }
}

void introsortImpl(std::vector<int>& a, int l, int r, int depth){
    while(r-l+1 > 24){
        if(depth==0){ heapSortRange(a,l,r); return; }
        --depth;
        int p = partition(a,l,r);
        if(p-l < r-p){
            introsortImpl(a,l,p-1,depth); l=p+1;
        }else{
            introsortImpl(a,p+1,r,depth); r=p-1;
        }
    }
    insertionRange(a,l,r);
}

void introSort(std::vector<int>& a){
    int depth = 2 * (int)std::log2(std::max(1,(int)a.size()));
    if(!a.empty()) introsortImpl(a,0,(int)a.size()-1,depth);
}
```
> 실제 `std::sort` 구현은 보다 정교한 피벗 선택(예: median-of-3, Tukey ninther), 분기 예측, 메모리 접근 최적화가 들어간다.

---

## 4. 비교 기반이 아닌 정렬 — 선형 시간까지

### 4.1 카운팅 정렬 (Counting Sort) — 작은 키 범위 정수
```cpp
#include <vector>
#include <algorithm>
std::vector<int> countingSort(const std::vector<int>& a, int K /*max value*/){
    std::vector<int> cnt(K+1), out(a.size());
    for(int x: a) ++cnt[x];
    for(int i=1;i<=K;++i) cnt[i]+=cnt[i-1];    // 누적 → 안정
    for(int i=(int)a.size()-1;i>=0;--i){       // 뒤에서 앞으로 → 안정성
        out[--cnt[a[i]]] = a[i];
    }
    return out;
}
```
- 시간 \(O(n+K)\), 공간 \(O(n+K)\), **안정 O**

### 4.2 기수 정렬 (Radix Sort) — 정수·고정길이 키
**LSD(하위 자릿수부터)** — 32-bit 비음수 정수 예:
```cpp
#include <vector>
#include <cstdint>
std::vector<uint32_t> radixLSD(std::vector<uint32_t> a){
    const int B=256; // 바이트 단위
    std::vector<uint32_t> tmp(a.size());
    for(int shift=0; shift<32; shift+=8){
        int cnt[B]={0};
        for(auto x: a) ++cnt[(x>>shift) & 0xFF];
        for(int i=1;i<B;++i) cnt[i]+=cnt[i-1];
        for(int i=(int)a.size()-1;i>=0;--i){
            uint32_t key=(a[i]>>shift)&0xFF;
            tmp[--cnt[key]]=a[i];
        }
        a.swap(tmp);
    }
    return a;
}
```
- 시간 \(O(d\cdot (n + B))\) (d=자릿수), **안정 O**, 비교 없음
- **부호 있는 정수**는 **바이트 순서를 부호 비트 기준으로 조정**(최상위 바이트에서 `xor 0x80` 등)하여 자연 순서 보장

### 4.3 버킷 정렬 (Bucket Sort) — [0,1) 균일 분포 실수 등
```cpp
#include <vector>
#include <algorithm>
std::vector<double> bucketSort(std::vector<double> a){
    int n=(int)a.size(); if(n==0) return a;
    std::vector<std::vector<double>> B(n);
    for(double x: a){
        int idx = std::min(n-1, (int)(x*n));
        B[idx].push_back(x);
    }
    int k=0;
    for(auto& b : B){
        std::sort(b.begin(), b.end());
        for(double x: b) a[k++]=x;
    }
    return a;
}
```
- 평균 \(O(n)\) (균일, 버킷 수 \(=n\)), **안정(내부 안정 정렬 시)**

---

## 5. C++에서 **정렬을 올바르게** 쓰는 법

### 5.1 비교자(Comparator) — 올바른 정의가 먼저

{% raw %}
```cpp
#include <vector>
#include <string>
#include <algorithm>
struct Record{ std::string name; int score; int id; };

// 1) 점수 내림차순, 2) 이름 오름차순, 3) id 오름차순
auto comp = [](Record const& a, Record const& b){
    if(a.score!=b.score) return a.score>b.score;
    if(a.name!=b.name)   return a.name<b.name;
    return a.id<b.id;
};

int main(){
    std::vector<Record> v = {{"kim",90,1},{"lee",90,2},{"kim",80,3}};
    std::sort(v.begin(), v.end(), comp); // strict-weak-ordering 준수
}
```
{% endraw %}

- `return a.key < b.key;` 규칙을 지키며 **비교의 추이성** 깨지지 않게 주의
- 부동소수점에 **NaN** 포함 시, 비교 불능(반사성 위배) → 정렬 전 필터링/정의 처리

### 5.2 다중 키 정렬 — 안정 정렬 체이닝 vs 하나의 비교자
- **안정 정렬을 뒤에서부터** 적용하면 중첩 키 효과
```cpp
#include <algorithm>
#include <vector>
#include <string>
struct R{ std::string name; int c1,c2; };
void multiKeyStable(std::vector<R>& v){
    std::stable_sort(v.begin(), v.end(), [](auto& a, auto& b){ return a.name<b.name; });
    std::stable_sort(v.begin(), v.end(), [](auto& a, auto& b){ return a.c2<b.c2; });
    std::stable_sort(v.begin(), v.end(), [](auto& a, auto& b){ return a.c1<b.c1; });
}
```
- 한 번의 비교자로 끝내도 되지만, **가독성/안정성 의도**가 중요

### 5.3 부분 정렬·Top-K — `std::nth_element`/`partial_sort`
```cpp
#include <vector>
#include <algorithm>
#include <iostream>

int main(){
    std::vector<int> a = {9,1,7,3,5,8,2,6,4};
    size_t k=5;
    std::nth_element(a.begin(), a.begin()+k, a.end()); // 앞 k개는 "k번째 이하"
    a.resize(k);
    std::sort(a.begin(), a.end()); // k개만 정렬
    for(int x: a) std::cout<<x<<" ";
}
```
- `nth_element` 평균 \(O(n)\), Top-K나 분위수 계산에 최적

### 5.4 안정 정렬 — `std::stable_sort`
- 대부분의 구현에서 **병합 정렬 기반**(안정, \(O(n\log n)\), \(O(n)\) 메모리)
- 동치 요소의 **입력 순서 보존**

### 5.5 병렬 정렬 — C++17 실행 정책
```cpp
#include <algorithm>
#include <execution>
#include <vector>

int main(){
    std::vector<int> v(1<<20);
    // ... fill v ...
    std::sort(std::execution::par, v.begin(), v.end()); // 구현/환경 의존
}
```
- 큰 데이터·비용 큰 비교에서 유리. 데이터/RAM/스케줄러 특성에 따라 성능 상이

---

## 6. 외부 정렬(External Sort) — 메모리 초과 데이터

### 6.1 전략
1) RAM에 들어가는 크기 \(M\)로 **런(run)** 생성(각 덩어리 정렬)
2) **k-way merge** 로 최종 병합 (우선순위 큐)

### 6.2 k-way merge 스케치
```cpp
#include <queue>
#include <vector>
#include <utility>
struct Node {
    int value; int runId;
    bool operator<(Node const& o) const { return value > o.value; } // min-heap
};
std::vector<int> kWayMerge(std::vector<std::vector<int>>& runs){
    std::priority_queue<Node> pq;
    std::vector<size_t> idx(runs.size());
    for(int i=0;i<(int)runs.size();++i) if(!runs[i].empty())
        pq.push({runs[i][0], i});
    std::vector<int> out;
    while(!pq.empty()){
        auto [val,id]=pq.top(); pq.pop();
        out.push_back(val);
        if(++idx[id] < runs[id].size())
            pq.push({runs[id][idx[id]], id});
    }
    return out;
}
```

---

## 7. 실무 최적화 포인트 — 캐시·분기·하이브리드

- **소구간 삽입 정렬**: \( n\le 16\sim 32 \) 구간은 삽입정렬이 더 빠름
- **3-way 파티션**: 중복 키 많은 데이터는 필수
- **피벗 선택**: median-of-3, **Tukey ninther**(3×3 median)
- **브랜치 줄이기**: 비교 결과를 정수로 변환해 **분기 예측 실패 최소화**
- **메모리 레이아웃**: AoS → SoA로 바꾸면 정렬 후 접근 locality 향상
- **안정성 vs 성능**: 안정이 꼭 필요하지 않다면 `std::sort`가 더 빠름

---

## 8. 알고리즘 선택 가이드

1) **정수·범위 제한**: 카운팅/기수
2) **문자열(고정 길이)**: MSD/LSD 기수, 또는 비교 기반 + 캐시된 키
3) **중복 키 많음**: 3-way 퀵/인트로소트
4) **안정성 필요**: `std::stable_sort`(병합)
5) **메모리 타이트 + 최악 보장**: 힙/인트로
6) **Top-K/분위수**: `nth_element` + 정렬
7) **외부 데이터**: 외부 병합정렬

---

## 9. 정확도 체크 — 테스트 유틸

```cpp
#include <vector>
#include <algorithm>
#include <cassert>

template<class T>
bool isSorted(const std::vector<T>& v){
    for(size_t i=1;i<v.size();++i) if(v[i]<v[i-1]) return false;
    return true;
}
template<class T, class Cmp>
bool isSorted(const std::vector<T>& v, Cmp cmp){
    for(size_t i=1;i<v.size();++i) if(cmp(v[i],v[i-1])) return false;
    return true;
}
```

---

## 10. 복잡도·속성 요약표 (확장)

| 알고리즘     | 최선 | 평균 | 최악 | 공간 | 안정 | 제자리 | 비고 |
|---|---:|---:|---:|---:|:--:|:--:|---|
| 선택 정렬    | \(n^2\) | \(n^2\) | \(n^2\) | 1 | ✗ | ✓ | 스왑 최소 |
| 버블 정렬    | \(n\) | \(n^2\) | \(n^2\) | 1 | ✓ | ✓ | 조기 종료 |
| 삽입 정렬    | \(n\) | \(n^2\) | \(n^2\) | 1 | ✓ | ✓ | 거의 정렬에 최강 |
| 셸 정렬      | — | \(n^{1.2\sim1.5}\)* | \(n^2\) | 1 | (대체로)✗ | ✓ | 간격 의존 |
| 병합 정렬    | \(n\log n\) | \(n\log n\) | \(n\log n\) | \(n\) | ✓ | ✗ | 외부정렬 적합 |
| 힙 정렬      | \(n\log n\) | \(n\log n\) | \(n\log n\) | 1 | ✗ | ✓ | 최악 보장 |
| 퀵 정렬      | \(n\log n\) | \(n\log n\) | \(n^2\) | \(\log n\) | ✗ | ✓ | 3-way·인트로 권장 |
| 인트로소트   | \(n\log n\) | \(n\log n\) | \(n\log n\) | \(\log n\) | ✗ | ✓ | `std::sort` 사상 |
| 카운팅       | \(n+K\) | \(n+K\) | \(n+K\) | \(n+K\) | ✓ | ✗ | 정수 범위 K |
| 기수(LSD)    | \(d(n+B)\) | \(d(n+B)\) | \(d(n+B)\) | \(n+B\) | ✓ | ✗ | 정수/고정 문자열 |
| 버킷         | \(n+k\) | \(n+k\) | \(n^2\) | \(n+k\) | (내부 안정) | ✗ | 분포 의존 |

\* 셸 정렬의 정확한 평균 차수는 간격 수열에 따라 다르다.

---

## 11. 통합 예시 — 입력 스펙에 따라 자동 선택

```cpp
#include <bits/stdc++.h>
using namespace std;

enum class Algo { Auto, Stable, Counting32, Radix32, Intro };

bool smallRange(const vector<int>& a){
    auto [mn,mx]=minmax_element(a.begin(), a.end());
    long long K = 1LL + *mx - *mn;
    return (K>0 && K <= (long long)2*a.size()); // 단순 휴리스틱
}

void autoSort(vector<int>& a, Algo pref=Algo::Auto){
    if(a.size()<32){ insertionSort(a); return; } // 앞에서 정의했다고 가정
    if(pref==Algo::Stable){ stable_sort(a.begin(), a.end()); return; }
    if(pref==Algo::Counting32 && smallRange(a)){
        int mn=*min_element(a.begin(), a.end());
        int mx=*max_element(a.begin(), a.end());
        int K=mx-mn; vector<int> cnt(K+1), out(a.size());
        for(int x: a) ++cnt[x-mn];
        for(int i=1;i<=K;++i) cnt[i]+=cnt[i-1];
        for(int i=(int)a.size()-1;i>=0;--i) out[--cnt[a[i]-mn]]=a[i];
        a.swap(out); return;
    }
    if(pref==Algo::Radix32){
        // 부호 있는 정수의 자연순서를 위해 상위 비트 xor
        vector<uint32_t> u(a.size());
        transform(a.begin(), a.end(), u.begin(),
                  [](int x){ return (uint32_t)(x ^ 0x80000000u); });
        u = radixLSD(std::move(u)); // 앞에서 구현
        transform(u.begin(), u.end(), a.begin(),
                  [](uint32_t x){ return (int)(x ^ 0x80000000u); });
        return;
    }
    introSort(a); // 기본값
}

int main(){
    vector<int> a = {170,45,75,90,802,24,2,66,-5,0,9999,-1};
    autoSort(a, Algo::Intro);
    cout<<"sorted: ";
    for(int x: a) cout<<x<<" ";
    cout<<"\n";
}
```

---

## 12. 실전 팁 — 성능·정확성 유지

- **데이터 특성 파악**: 중복/분포/도메인/안정성 필요 여부
- **소구간 임계값**을 조정(16~32): 프로파일 후 확정
- **대형 구조체 정렬**: **키만 따로 벡터로 뽑아** 간접 정렬(인덱스 배열) → 캐시 효율
- **부동소수점**: NaN 처리 정책 확정(후행·선행·제외)
- **다국어 문자열**: 단순 `operator<` 대신 로케일 기반 비교 필요 시 ICU/OS API 고려
- **예외 안전**: 비교자에서 예외 던지지 않기(표준 정렬은 강한 보장 전제 어려움)
- **병렬**: 충분히 큰 n + 비싼 비교에서만 이득. 작은 n은 오버헤드가 더 큼

---

## 13. 학습/면접 포인트 요약

1) 비교 기반 하한 \( \Omega(n\log n) \) 유도
2) `std::sort` = **인트로소트** (퀵 + 힙 + 삽입)
3) `std::stable_sort` = **병합 기반**(안정, \(O(n)\) 추가 메모리)
4) **Top-K** = `nth_element` → 부분 정렬
5) **중복 키** 많을 때 3-way 파티션
6) **정수 범위 제한** → 카운팅/기수
7) **외부 정렬** → 런 + k-way 머지

---

## 14. 빠른 레퍼런스 — STL 정렬 API

```cpp
#include <algorithm>
#include <vector>

std::sort(v.begin(), v.end());                          // 평균 O(n log n), 불안정
std::sort(v.begin(), v.end(), comp);                    // 사용자 비교자
std::stable_sort(v.begin(), v.end());                   // 안정 O(n log n), O(n) 메모리
std::partial_sort(v.begin(), v.begin()+k, v.end());     // 앞 k개만 정렬
std::nth_element(v.begin(), v.begin()+k, v.end());      // k번째 원소 기준 분할
std::is_sorted(v.begin(), v.end());                     // 정렬 여부 체크
```

---

## 15. 마무리

- 비교 기반 정렬의 하한은 피할 수 없다 → **하이브리드 설계**가 핵심
- **데이터 특성**(분포·중복·도메인)을 알면 **선형급**까지 가능
- C++에서는 `std::sort`/`std::stable_sort`/`nth_element` 를 중심으로
  **소구간 삽입정렬 + 3-way 파티션 + 적절한 피벗 선택**으로 성능을 끌어올려라.
