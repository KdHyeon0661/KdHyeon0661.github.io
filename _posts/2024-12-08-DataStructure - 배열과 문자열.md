---
layout: post
title: Data Structure - 배열과 문자열
date: 2024-12-08 19:20:23 +0900
category: Data Structure
---
# 배열과 문자열

## 1. 배열(Array)

### 1.1 정의와 메모리 모델

배열은 **같은 타입의 요소들이 연속된 메모리**에 저장된 구조다. 연속성 덕분에 임의 인덱스 접근이 상수 시간에 가능하다.

```cpp
int arr[5] = {1, 2, 3, 4, 5}; // 스택/정적 영역에 연속 배치
```

- 인덱스 i의 주소는 `base + i * sizeof(T)`로 계산된다.
- 캐시 지역성이 좋아 순차 순회가 매우 빠르다.

### 1.2 시간/공간 특성

- **접근**: $$O(1)$$
- **검색(선형)**: $$O(n)$$
- **삽입/삭제(중간)**: 시프트 필요 → $$O(n)$$
- **메모리**: 연속 블록 필요(큰 배열은 할당 실패 가능)

### 1.3 정적 배열 vs 동적 배열

```cpp
// 정적(스택) 크기 고정
int a[10];

// 동적(힙) new/delete
int n = 1000;
int* b = new int[n];   // 할당
// ... 사용 ...
delete[] b;            // 해제 (반드시)
```

- 정적 배열은 크기 변경 불가.
- 동적 배열은 크기 선택이 유연하나 **수동 관리 오류**(누수/이중 해제/경계 초과)에 주의.

### 1.4 경계/예외: UB(Undefined Behavior)

```cpp
int x[3] = {1,2,3};
int v = x[3];   // UB: 범위 밖 읽기
x[-1] = 7;     // UB: 음수 인덱스
```

경계 체크가 필요하면 `std::vector::at()`처럼 예외를 던지는 접근을 사용하라.

### 1.5 기본 예제: 선형 검색

```cpp
int find_index(const int* arr, int n, int target) {
    for (int i = 0; i < n; ++i)
        if (arr[i] == target) return i;
    return -1;
}
```

### 1.6 이진 검색(정렬 가정)

```cpp
int lower_bound_index(const std::vector<int>& v, int key){
    int lo=0, hi=(int)v.size();
    while (lo<hi){
        int mid=(lo+hi)/2;
        if (v[mid] < key) lo=mid+1;
        else hi=mid;
    }
    return lo; // key 이상 첫 위치
}
```

- 시간: $$O(\log n)$$
- 전제: **정렬** 유지

### 1.7 배열 변형 패턴

#### (1) 회전(rotate) — 세 번 뒤집기

```cpp
void reverse(int* a, int l, int r){ // [l, r] inclusive
    while(l<r) std::swap(a[l++], a[r--]);
}
void rotate_right(int* a, int n, int k){
    k%=n; if(k<0) k+=n;
    reverse(a, 0, n-1);
    reverse(a, 0, k-1);
    reverse(a, k, n-1);
}
```

#### (2) 중복 제거(정렬 배열, in-place)

```cpp
int unique_inplace(std::vector<int>& v){
    if (v.empty()) return 0;
    int w=1;
    for (int i=1;i<(int)v.size();++i)
        if (v[i]!=v[w-1]) v[w++]=v[i];
    v.resize(w);
    return w;
}
```

#### (3) 파티션(quickselect/퀵정렬 핵심)

```cpp
int partition(std::vector<int>& a, int l, int r){
    int pivot=a[r], i=l;
    for(int j=l;j<r;++j) if(a[j]<=pivot) std::swap(a[i++], a[j]);
    std::swap(a[i], a[r]);
    return i;
}
```

### 1.8 누적합 / 차분 배열

- 누적합 $$P[i] = \sum_{k=0}^{i-1} A[k]$$ → 구간합 $$[l,r)$$은 $$P[r]-P[l]$$

```cpp
std::vector<long long> prefix(const std::vector<int>& a){
    std::vector<long long> p(a.size()+1,0);
    for(size_t i=0;i<a.size();++i) p[i+1]=p[i]+a[i];
    return p;
}
```

- 차분 배열로 구간 업데이트를 상수 시간으로 축적 후 한 번에 복원 가능.

### 1.9 투 포인터 / 슬라이딩 윈도우

#### (1) 고유 원소 최대 길이 부분배열

```cpp
#include <unordered_map>
int longest_distinct(const std::vector<int>& a){
    std::unordered_map<int,int> last;
    int best=0, l=0;
    for(int r=0;r<(int)a.size();++r){
        if(last.count(a[r])) l = std::max(l, last[a[r]]+1);
        last[a[r]]=r;
        best = std::max(best, r-l+1);
    }
    return best;
}
```

#### (2) 합이 S 이하인 최대 길이(양수 배열)

```cpp
int max_len_leq_sum(const std::vector<int>& a, long long S){
    int n=a.size(), l=0, best=0; long long sum=0;
    for(int r=0;r<n;++r){
        sum+=a[r];
        while(sum>S) sum-=a[l++];
        best=std::max(best, r-l+1);
    }
    return best;
}
```

---

## 2. 문자열(String)

### 2.1 C 문자열 vs C++ `std::string`

- C 문자열: `char*` + **널 종료 `\0`**. 길이 계산이 $$O(n)$$.
- `std::string`: 길이/용량 보관, 동적 확장, 예외/대입/이동 지원.

```c
char s[] = "hello"; // {'h','e','l','l','o','\0'}
```

```cpp
std::string s = "hello";
s += " world";            // 편의 + 자동 확장
std::cout << s.size();    // O(1)
```

### 2.2 대표 연산 복잡도

| 연산 | 설명 | 시간 |
|---|---|---|
| `size()` | 길이 | $$O(1)$$ |
| 인덱스 접근 | `s[i]` | $$O(1)$$ |
| 비교 | 사전순/동등 | $$O(\min(n,m))$$ |
| 연결 | `+`, `append` | 평균 $$O(n+m)$$ |
| `substr(pos,k)` | 부분 문자열 복사 | $$O(k)$$ |
| `find` | 기본 검색 | $$O(n\cdot m)$$ 최악 |

> 구현체에 따라 **SSO(Small String Optimization)** 등 상수항 최적화가 있을 수 있으나 표준에서 보장되지는 않는다.

### 2.3 `std::string_view` — 복사 없이 참조

```cpp
#include <string_view>
void log_line(std::string_view sv){
    // 문자열/리터럴/부분범위를 모두 비용 없이 참조
}
```

- 비소유 참조이므로 수명 주의.

### 2.4 유니코드 기본기

- UTF-8은 **가변 길이** 인코딩. `s[i]`는 **바이트 i**이지 문자 i가 아니다.
- 문자의 수를 세거나 자르려면 **코드 포인트 파싱**이 필요.
- 결합 문자/정규화(NFC/NFD) 이슈 존재. 단순 바이트 비교는 사용자 관점과 다를 수 있다.

---

## 3. 문자열 알고리즘 실전

### 3.1 회문(팰린드롬)

#### (1) 투 포인터

```cpp
bool is_pal(const std::string& s){
    int l=0, r=(int)s.size()-1;
    while(l<r) if(s[l++]!=s[r--]) return false;
    return true;
}
```

#### (2) 중심 확장으로 최장 회문 부분문자열 길이

```cpp
int expand(const std::string& s,int L,int R){
    while(0<=L && R<(int)s.size() && s[L]==s[R]){--L;++R;}
    return R-L-1;
}
int longest_pal_substr(const std::string& s){
    int best=0;
    for(int i=0;i<(int)s.size();++i){
        best=std::max({best, expand(s,i,i), expand(s,i,i+1)});
    }
    return best;
}
```

> 선형 시간 알고리즘 **Manacher**도 있으나 구현이 길어 개요만 소개.

### 3.2 아나그램 판단

```cpp
#include <array>
bool is_anagram_latin(const std::string& a, const std::string& b){
    if(a.size()!=b.size()) return false;
    std::array<int,256> cnt{}; // 바이트 범위
    for(unsigned char c: a) ++cnt[c];
    for(unsigned char c: b) if(--cnt[c]<0) return false;
    return true;
}
```

### 3.3 슬라이딩 윈도우: 모든 아나그램 시작 위치

```cpp
#include <vector>
std::vector<int> find_anagrams(const std::string& s, const std::string& p){
    if(s.size()<p.size()) return {};
    std::array<int,26> need{}, win{};
    for(char c: p) ++need[c-'a'];
    int m=p.size(), diff=0;
    for(int i=0;i<26;++i) if(need[i]) ++diff;

    auto upd=[&](int idx, int delta){
        int before = (win[idx]==need[idx]);
        win[idx]+=delta;
        int after  = (win[idx]==need[idx]);
        if(before&&!after) ++diff;
        else if(!before&&after) --diff;
    };

    for(int i=0;i<(int)s.size();++i){
        upd(s[i]-'a', +1);
        if(i>=m) upd(s[i-m]-'a', -1);
        if(i>=m-1 && diff==0) { /* 일치 */ }
    }

    // 더 간결한 구현(직접 비교)도 가능하나 위 방식은 상수항 최적화에 유리
    std::vector<int> res;
    win.fill(0); diff=0; for(int i=0;i<26;++i) if(need[i]) ++diff;
    for(int i=0;i<(int)s.size();++i){
        int id = s[i]-'a';
        int before = (win[id]==need[id]);
        win[id]++;
        int after  = (win[id]==need[id]);
        if(before&&!after) ++diff; else if(!before&&after) --diff;
        if (i>=m){
            int jd=s[i-m]-'a';
            before = (win[jd]==need[jd]);
            win[jd]--;
            after  = (win[jd]==need[jd]);
            if(before&&!after) ++diff; else if(!before&&after) --diff;
        }
        if(i>=m-1 && diff==0) res.push_back(i-m+1);
    }
    return res;
}
```

### 3.4 KMP(접두사 함수/실패 함수)

#### 아이디어
패턴의 **자기 접두사=접미사** 길이를 이용해 텍스트 인덱스를 되돌리지 않고 이동.

- 접두사 함수(파이 함수) 정의:
  $$
  \pi[i] = \max\{\,k< i+1 \mid P[0..k-1] = P[i-k+1..i] \,\}
  $$

#### 구현

```cpp
std::vector<int> prefix_function(const std::string& p){
    int n=p.size();
    std::vector<int> pi(n);
    for(int i=1;i<n;++i){
        int j=pi[i-1];
        while(j>0 && p[i]!=p[j]) j=pi[j-1];
        if(p[i]==p[j]) ++j;
        pi[i]=j;
    }
    return pi;
}

std::vector<int> kmp_search(const std::string& s, const std::string& p){
    if(p.empty()) return {};
    auto pi = prefix_function(p);
    std::vector<int> res;
    for(int i=0,j=0;i<(int)s.size();++i){
        while(j>0 && s[i]!=p[j]) j=pi[j-1];
        if(s[i]==p[j]) ++j;
        if(j==(int)p.size()){
            res.push_back(i-j+1);
            j=pi[j-1];
        }
    }
    return res;
}
```

- 시간: $$O(n+m)$$
- 공간: $$O(m)$$

### 3.5 Z-알고리즘

- 문자열 `S`에 대해 `Z[i]` = `S`와 `S[i..]`의 최장 공통 접두사 길이.
- 패턴 매칭을 위해 `P + '#' + T`에 Z를 적용하는 방식 사용.

```cpp
std::vector<int> z_function(const std::string& s){
    int n=s.size(); std::vector<int> z(n);
    for(int i=1,l=0,r=0;i<n;++i){
        if(i<=r) z[i]=std::min(r-i+1, z[i-l]);
        while(i+z[i]<n && s[z[i]]==s[i+z[i]]) ++z[i];
        if(i+z[i]-1>r) l=i, r=i+z[i]-1;
    }
    z[0]=n; // 관례
    return z;
}
```

### 3.6 롤링 해시(라빈-카프)

- 다항 해시:
  $$
  H(s_0\ldots s_{k-1}) = \left(\sum_{i=0}^{k-1} s_i \cdot B^{k-1-i}\right)\bmod M
  $$
- 해시 충돌 가능 → 이중 모듈러스/64-bit 곱 분산으로 완화.

```cpp
struct RH {
    static const uint64_t B = 1315423911ull; // 임의 큰 베이스
    std::vector<uint64_t> pow, pref;         // 64-bit mod 2^64 (암묵 모듈로)
    RH(const std::string& s){
        int n=s.size(); pow.resize(n+1,1); pref.resize(n+1,0);
        for(int i=1;i<=n;++i){
            pow[i]=pow[i-1]*B;
            pref[i]=pref[i-1]*B + (unsigned char)s[i-1]+1;
        }
    }
    uint64_t hash(int l,int r) const { // [l,r)
        return pref[r] - pref[l]*pow[r-l];
    }
};
```

- 64-bit overflow는 **모듈러 2⁶⁴**처럼 동작하므로 빠르다(충돌 가능성은 남음).
- 안전 요구가 높으면 두 개의 다른 모듈러를 병용.

---

## 4. 배열 <> 문자열: 상호 환원과 실전 문제

- 문자열은 결국 **바이트/문자 코드의 배열**. 투 포인터/슬라이딩 윈도우/누적합류 기법이 그대로 적용된다.
- 대표 문제군:
  - **최장 고유 부분문자열**: 맵으로 최근 위치 관리(슬윈)
  - **아나그램 찾기**: 빈도 벡터 슬윈
  - **회문 관련**: 투 포인터/중심 확장/Manacher
  - **패턴 매칭**: KMP/Z/롤링 해시
  - **중복/압축**: RLE(run-length), 접두사 함수로 반복 패턴 검출

---

## 5. `std::vector`로 확장: 동적 배열 핵심

### 5.1 자동 성장과 상환분석

크기가 꽉 차면 용량을 **c → g·c**로 늘리며 재할당한다. 보통 g≈2.

$$
\underbrace{1+1+2+4+\cdots}_{\le 2n} \Rightarrow \text{amortized } O(1)
$$

### 5.2 공통 실수

- 반복자 무효화: **재할당/중간 삽입/삭제** 후 기존 포인터/참조는 무효 가능.
- `reserve()` 없이 대량 `push_back` → 잦은 재할당.
- `at()` 대신 `operator[]`만 사용 → 경계 예외 놓침.

### 5.3 예제

```cpp
#include <vector>
#include <iostream>

int main(){
    std::vector<int> v = {1,2,3};
    v.reserve(1000);          // 재할당 회수 감소
    v.push_back(4);
    v.insert(v.begin()+1, 10); // 1 10 2 3 4
    v.erase(v.begin()+2);      // 1 10 3 4
    for(int x: v) std::cout<<x<<" ";
}
```

---

## 6. 성능과 캐시, SIMD 힌트

- **연속 메모리** 순차 접근은 **캐시 미스**가 적어 매우 빠르다.
- 조건 분기보다 **분기 예측 친화적** 루프가 유리.
- 문자열 비교/검색에서 구현체는 플랫폼별 **SIMD 가속**을 사용할 수 있다(사용자는 API 레벨에서 이득).

---

## 7. 테스트 전략과 디버깅

- **경계 케이스**: 빈, 길이 1, 모두 동일 문자, 매우 긴 입력.
- **무작위 퍼징**: 느린 정답(브루트포스)과 결과 대조.
- **도구**: ASan/UBSan로 경계 초과/정수 오버플로 감지.

예: KMP vs `std::search` 대조.

```cpp
#include <cassert>
#include <algorithm>

void kmp_selftest(){
    std::string s="abcababcabxabcab", p="abcab";
    auto idxs = kmp_search(s,p);
    auto it = std::search(s.begin(), s.end(), p.begin(), p.end());
    assert(!idxs.empty() && idxs[0] == (int)std::distance(s.begin(), it));
}
```

---

## 8. 종합 예제 모음

### 8.1 부분합으로 고정 길이 k의 최대 합

```cpp
long long max_sum_k(const std::vector<int>& a, int k){
    if(k>(int)a.size()) return 0;
    long long cur=0, best=LLONG_MIN;
    for(int i=0;i<(int)a.size();++i){
        cur += a[i];
        if(i>=k) cur -= a[i-k];
        if(i>=k-1) best = std::max(best, cur);
    }
    return best;
}
```

### 8.2 문자열 내 최소 윈도우 커버(필수 문자 모두 포함)

```cpp
#include <unordered_map>

std::pair<int,int> min_window_substr(const std::string& s, const std::string& t){
    if(t.empty()) return {0,0};
    std::unordered_map<char,int> need, have;
    for(char c: t) ++need[c];
    int l=0, haveKinds=0, needKinds=need.size();
    int bestL=0, bestLen=INT_MAX;

    for(int r=0;r<(int)s.size();++r){
        if(need.count(s[r])){
            if(++have[s[r]]==need[s[r]]) ++haveKinds;
        }
        while(haveKinds==needKinds){
            if(r-l+1 < bestLen){ bestLen=r-l+1; bestL=l; }
            if(need.count(s[l]) && --have[s[l]]<need[s[l]]) --haveKinds;
            ++l;
        }
    }
    if(bestLen==INT_MAX) return {-1,-1};
    return {bestL, bestLen}; // s.substr(bestL,bestLen)
}
```

### 8.3 패턴 매칭 벤치: KMP vs Z vs find

문자열 길이가 크고 패턴 반복 구조가 있으면 **KMP/Z**가 유리. 일반 텍스트에서는 구현/플랫폼에 따라 `std::string::find`가 매우 빠를 수도 있다(최적화/어셈 수준).

---

## 9. 복잡도 요약표

| 구조/연산 | 시간 | 비고 |
|---|---|---|
| 배열 랜덤 접근 | $$O(1)$$ | 연속 메모리 |
| 배열 중간 삽입/삭제 | $$O(n)$$ | 시프트 |
| 문자열 연결 | 평균 $$O(n)$$ | `reserve()`로 완화 |
| KMP 검색 | $$O(n+m)$$ | 접두사 함수 |
| Z-알고리즘 | $$O(n)$$ | 전체 문자열 처리 |
| 롤링 해시 비교 | $$O(1)$$ 평균 | 충돌 가능 |
| 슬라이딩 윈도우 | 입력당 $$O(1)$$ 이동 | 총 $$O(n)$$ |

---

## 10. 자주 하는 질문(FAQ)

- **Q. `std::string`은 유니코드 안전한가?**
  바이트 시퀀스를 담는 컨테이너다. 유니코드 문자를 단위로 다루려면 UTF 디코딩이 필요.

- **Q. 왜 연결 리스트보다 배열이 빠른가?**
  캐시 지역성/분기 예측/메모리 할당 비용에서 배열이 유리. 리스트는 포인터 추적 비용이 크다.

- **Q. 대량 문자열 연결이 느릴 때?**
  미리 `reserve` 하거나 `std::ostringstream`를 사용.

---

## 11. 마무리

배열·문자열은 **연속 메모리 모델** 위에 세워진 가장 기본 단위다. 여기서 출발해 **슬라이딩 윈도우/투 포인터/누적합** 같은 테크닉과 **KMP/Z/롤링 해시** 같은 문자열 알고리즘을 익히면, 대부분의 문자열/배열 문제를 강력하게 해결할 수 있다. 다음 글에서는 **접미사 배열/트리**, **Manacher**, **문자열 해시 충돌 완화** 등을 더 깊이 다룬다.
