---
layout: post
title: Data Structure - 트라이
date: 2024-12-16 19:20:23 +0900
category: Data Structure
---
# 자료구조

## 빠른 리마인드 — Trie란?

- **Trie(트라이)**는 문자열(보통 알파벳)의 **공통 접두사(prefix)**를 공유하는 **트리 기반 자료구조**다.
- 노드: “해당 접두사까지 도달”을 의미, 간선: “다음 문자”.
- **탐색 시간** \(= O(L)\), \(L\)은 문자열 길이. 해시보다 **정렬·접두사 검색**·**사전순 순회**가 편리.

---

## 노드 설계와 문자 집합

### 전용 — 고정 배열 방식

```cpp
struct TrieNode {
    std::array<TrieNode*, 26> next{};
    int pass = 0;      // 이 노드를 경유(이 접두사를 가지는) 단어 수
    int end  = 0;      // 이 노드에서 끝나는 단어 수(중복 허용)
    TrieNode() { next.fill(nullptr); }
};
```
- `pass`는 “접두사 개수(count of words with this prefix)”, `end`는 “정확히 이 단어 개수”.
- **장점**: 매우 빠른 인덱싱(상수 시간). **단점**: 희소한 경우 포인터 낭비.

### — 해시/맵 방식

```cpp
struct TrieNodeX {
    std::unordered_map<char32_t, TrieNodeX*> next; // 유니코드 코드포인트
    int pass = 0, end = 0;
};
```
- **장점**: 메모리 효율(희소), 임의 문자 지원. **단점**: 상수 계수 커짐(해시/맵 오버헤드).
- 실전: **하이브리드**(작을 때 vector, 커지면 map로 승격)도 유효.

> 본문 코드는 26자 소문자 버전으로 설명하되, 동일 아이디어를 유니코드에도 적용 가능.

---

## 기본 연산 — 삽입/검색/접두사 검사/삭제(중복·가지치기)

```cpp
#include <bits/stdc++.h>

using namespace std;

struct TrieNode {
    array<TrieNode*, 26> next{};
    int pass = 0;  // prefix count
    int end  = 0;  // word multiplicity
    TrieNode(){ next.fill(nullptr); }
};

struct Trie {
    TrieNode* root = new TrieNode();

    ~Trie(){ clear(root); }
    void clear(TrieNode* n){
        if(!n) return;
        for(auto* c : n->next) clear(c);
        delete n;
    }

    static int idx(char ch){ return ch - 'a'; } // 전처리에서 소문자 가정

    void insert(const string& s){
        TrieNode* cur = root;
        cur->pass++;
        for(char ch : s){
            int i = idx(ch);
            if(!cur->next[i]) cur->next[i] = new TrieNode();
            cur = cur->next[i];
            cur->pass++;
        }
        cur->end++;
    }

    bool search(const string& s) const {
        const TrieNode* cur = root;
        for(char ch : s){
            int i = idx(ch);
            cur = cur->next[i];
            if(!cur) return false;
        }
        return cur->end > 0;
    }

    bool startsWith(const string& p) const {
        const TrieNode* cur = root;
        for(char ch : p){
            int i = idx(ch);
            cur = cur->next[i];
            if(!cur) return false;
        }
        return true;
    }

    // s가 존재하면 1개 삭제(중복 허용), 경로의 pass/end 갱신 및 가지치기
    bool erase(const string& s){
        if(!search(s)) return false; // 없으면 실패
        TrieNode* cur = root;
        cur->pass--;
        for(char ch : s){
            int i = idx(ch);
            TrieNode* nxt = cur->next[i];
            nxt->pass--;
            // 아래 노드가 더 이상 아무 단어도 지나지 않으면 가지치기
            if(nxt->pass == 0){
                clear(nxt);
                cur->next[i] = nullptr;
                return true;
            }
            cur = nxt;
        }
        cur->end--;
        return true;
    }

    // 접두사 p를 가지는 단어 수
    int countPrefix(const string& p) const {
        const TrieNode* cur = root;
        for(char ch : p){
            int i = idx(ch);
            cur = cur->next[i];
            if(!cur) return 0;
        }
        return cur->pass; // 해당 노드를 지나는 모든 단어 수
    }

    // 정확히 s와 동일한 단어 수(중복 개수)
    int countWord(const string& s) const {
        const TrieNode* cur = root;
        for(char ch : s){
            int i = idx(ch);
            cur = cur->next[i];
            if(!cur) return 0;
        }
        return cur->end;
    }
};
```

**삭제 포인트**
- `pass`는 경로상의 **모든 노드**에서 감소시키며, 0이 되면 **하위 전체를 해제**(가지치기).
- `end`는 마지막 노드에서 **중복 개수**를 감소.

---

## — 접두사 기반 제안

- 절차: **(1) 접두사 노드까지 이동 → (2) DFS/BFS로 후보 수집 → (3) 필요 시 k개 제한/정렬**.

```cpp
struct Suggest {
    string word; int freq;
    bool operator<(const Suggest& o) const {
        if(freq != o.freq) return freq > o.freq; // 높은 빈도 우선
        return word < o.word;                    // 사전순 보조
    }
};

void dfsCollect(TrieNode* u, string& cur, vector<Suggest>& out){
    if(!u) return;
    if(u->end > 0) out.push_back({cur, u->end});
    for(int i=0;i<26;++i){
        if(u->next[i]){
            cur.push_back('a'+i);
            dfsCollect(u->next[i], cur, out);
            cur.pop_back();
        }
    }
}

vector<Suggest> autocomplete(Trie& T, const string& prefix, int k = 10){
    TrieNode* cur = T.root;
    for(char ch : prefix){
        int i = ch-'a';
        if(!cur->next[i]) return {}; // 후보 없음
        cur = cur->next[i];
    }
    vector<Suggest> cand;
    string buf = prefix;
    dfsCollect(cur, buf, cand);
    sort(cand.begin(), cand.end());           // freq desc, lex asc
    if((int)cand.size() > k) cand.resize(k);  // 상위 k개
    return cand;
}
```

**응용**
- 검색창 자동완성, “자주 찾는 쿼리” 추천(빈도 = `end` 누적 또는 별도 가중치).

---

## 와일드카드 검색 — `?`(1글자)과 `*`(0+글자)

```cpp
bool matchWildcard(TrieNode* u, const string& pat, int pos){
    if(!u) return false;
    if(pos == (int)pat.size()) return u->end > 0;

    char c = pat[pos];
    if(c == '?'){
        for(int i=0;i<26;++i)
            if(u->next[i] && matchWildcard(u->next[i], pat, pos+1))
                return true;
        return false;
    }
    if(c == '*'){
        // 1) *가 빈문자열
        if(matchWildcard(u, pat, pos+1)) return true;
        // 2) *가 1글자 이상
        for(int i=0;i<26;++i)
            if(u->next[i] && matchWildcard(u->next[i], pat, pos))
                return true;
        return false;
    }
    int i = c - 'a';
    return u->next[i] && matchWildcard(u->next[i], pat, pos+1);
}
```

> 최악 복잡도는 지수적이지만, **패턴이 제한적**이거나 **짧은 접두사를 먼저 고정**하면 실전에서도 충분히 빠름.
> 많은 패턴을 **동시에** 찾을 땐 Aho–Corasick가 적합(별도 글 추천).

---

## 사전순 K번째 단어 (k-th in lexicographic order)

- 아이디어: 각 노드에서 **서브트리 단어 수 = pass** (해당 접두사를 가지는 단어 수).
- 사전순으로 내려가며 “왼쪽 가지 크기”를 누적해 k를 소모.

```cpp
// 주의: 여기서 pass는 "이 노드를 지나가는 단어 수"로 정의했으므로
// root의 pass = 전체 단어 수. (insert/erase가 pass를 정확히 유지해야 함)

bool kth(TrieNode* u, long long& k, string& out){
    // 현재 노드에서 끝나는 단어들 먼저 소비(사전순에서 접두사 자체가 앞선다)
    if(u->end){
        if(k <= u->end){ return true; } // out 그대로(현재 접두사)
        k -= u->end;
    }
    for(int i=0;i<26;++i){
        TrieNode* v = u->next[i];
        if(!v) continue;
        long long sub = v->pass; // 이 가지의 단어 수
        if(k > sub){ k -= sub; continue; }
        out.push_back('a'+i);
        bool ok = kth(v, k, out);
        if(ok) return true;
        out.pop_back();
    }
    return false;
}

string kthWord(Trie& T, long long k){
    if(k <= 0 || k > T.root->pass) return "";
    string out;
    long long kk = k;
    bool ok = kth(T.root, kk, out);
    return ok ? out : "";
}
```

---

## 직렬화·역직렬화 — 저장/전송/테스트 픽스처

### 간단 텍스트 직렬화(전위 순회)

- 형식: `(<end> <child_count> <char child> ... [child_subtree] ...)`
- 실제 환경에선 바이너리/압축 권장.

```cpp
void serialize(TrieNode* u, ostream& os){
    if(!u){ os << "# "; return; }           // 안전장치(루트 null 방지)
    os << u->end << " ";
    // 자식 집계
    vector<int> idxs;
    for(int i=0;i<26;++i) if(u->next[i]) idxs.push_back(i);
    os << idxs.size() << " ";
    for(int i : idxs){
        os << char('a'+i) << " ";
        serialize(u->next[i], os);
    }
}

TrieNode* deserialize(istream& is){
    string tok; if(!(is>>tok)) return nullptr; // EOF
    if(tok == "#") return nullptr;
    int end = stoi(tok);
    int m; is >> m;
    TrieNode* u = new TrieNode();
    u->end = end;
    for(int j=0;j<m;++j){
        char ch; is >> ch;
        TrieNode* c = deserialize(is);
        u->next[ch-'a'] = c;
    }
    // pass 다시 계산(역직렬화 후 한 번에)
    // DFS로 pass 합산
    function<int(TrieNode*)> pull = [&](TrieNode* x)->int{
        if(!x) return 0;
        int s = x->end;
        for(auto* p : x->next) if(p) s += pull(p);
        x->pass = s;
        return s;
    };
    pull(u);
    return u;
}
```

---

## 종합 예제 — 사용 시나리오

```cpp
int main(){
    ios::sync_with_stdio(false);
    cin.tie(nullptr);

    Trie T;
    vector<string> words = {"app","apple","application","apply","bat","bad","bag","banner"};
    for(auto& w: words) T.insert(w);

    cout << boolalpha;
    cout << "search(app): "   << T.search("app")   << "\n"; // true
    cout << "search(appl): "  << T.search("appl")  << "\n"; // false
    cout << "startsWith(ap): "<< T.startsWith("ap")<< "\n"; // true

    cout << "countPrefix(ap): "<< T.countPrefix("ap") << "\n"; // app* 개수
    cout << "countWord(apple): "<< T.countWord("apple") << "\n";

    // 자동완성
    auto sug = autocomplete(T, "app", 5);
    cout << "autocomplete(app):\n";
    for(auto& s : sug) cout << "  " << s.word << " (freq=" << s.freq << ")\n";

    // 와일드카드
    cout << "matchWildcard 'ba?': ";
    cout << matchWildcard(T.root, "ba?", 0) << "\n"; // bad, bag 매칭 → true

    // K번째(사전순)
    for(int k=1;k<=T.root->pass;++k){
        cout << k << "th: " << kthWord(T, k) << "\n";
    }

    // 삭제
    cout << "erase(apple): " << T.erase("apple") << "\n";
    cout << "search(apple): " << T.search("apple") << "\n"; // false
    cout << "search(app):   " << T.search("app")   << "\n"; // true

    // 직렬화/역직렬화
    {
        stringstream ss;
        serialize(T.root, ss);
        TrieNode* rebuilt = deserialize(ss);
        cout << "rebuilt startsWith(app): " << Trie{rebuilt}.startsWith("app") << "\n";
        // cleanup: Trie{rebuilt}는 임시이므로 명시적 clear가 필요하면 별도 관리
    }
}
```

---

## 복잡도 분석

문자열 집합 \(S=\{s_1,\dots,s_n\}\)의 총 길이를 \(N=\sum_i |s_i|\)라 할 때,

- **공간**: \(O(N \cdot \alpha)\) — \(\alpha\)=문자 집합 밀도에 따라 달라짐.
  고정 배열(26)은 상수 26 포인터×노드 수, 희소 시 낭비↑. 해시/맵은 상수 계수↑.
- **삽입/검색/접두사**: \(O(L)\), \(L=|s|\).
- **삭제**: \(O(L)\) (경로 카운트 감소 + 필요 시 가지치기).
- **자동완성**: 접두사 노드 도달 \(O(P)\) + 수집 \(O(K \cdot A)\) (K개 결과, 평균 분기 A).
- **와일드카드**: 최악 지수. 실전에서는 접두사 고정·가지치기·제한 K로 제어.
- **K번째**: \(O(\Sigma)\) — 알파벳 크기 \(\Sigma\)에 비례하여 한 단계씩 내려감(보통 상수 또는 26).

---

## 실전 팁 — 정확성·성능·메모리

1. **전처리**: 입력을 **소문자화**·공백 제거·정규화(NFC/NFD) 일관성 유지.
2. **중복 단어**: `end`를 **카운터**로 관리(빈도 기반 자동완성에 바로 활용).
3. **삭제**: `pass` 0 가지치기 누락 시 **유령 경로**가 남음(메모리 누수·검색 오류).
4. **메모리**:
   - 희소 입력: `unordered_map`/`vector<pair<char,node*>>` 하이브리드.
   - 대량 단어: **메모리 풀/arena**로 `new` 오버헤드 제거.
   - 긴 체인: **압축 트라이(Radix/Patricia)** 고려(경로 압축).
5. **유니코드**:
   - 문자 경계는 **코드포인트**와 **사용자 인식 글자(grapheme)**가 다를 수 있음.
   - 정렬·대소문자 변환은 로케일 의존(영문 외 언어면 ICU 같은 라이브러리 권장).
6. **대량 패턴 검색**: 와일드카드/다중 문자열 매칭은 **Aho–Corasick**로 전환.

---

## 수학 스냅샷 — 접두사 공유의 이점

문자열 집합 \(S\)의 모든 접두사를 **공유**하므로, 트라이 노드 수는 “**서로 다른 접두사 수**”와 같다.
따라서 순수 배열 보관 대비, 공통 접두사 비율이 높을수록 공간 이점이 커진다.

- 전체 문자 수 \(N\)에 대해, 트라이의 **간선 수**는 \( \le N \).
- **탐색 시간**은 해시의 평균 \(O(1)\)과 비슷한 수준이나, **정렬성과 접두사 연산**에서 우위.

---

## 확장: 압축 Trie(Radix)·비트 트라이(Bitwise)

- **Radix(압축)**: 단일 경로를 **라벨 문자열**로 압축 → **깊이 감소/메모리 절감**.
  (문자열 사전에 효율적. 분할(split)/병합 구현 필요)
- **Bitwise(이진 트라이)**: 정수 키를 비트 단위로 → **최대 XOR**·**IPv4 LPM** 등에 특화.

> 이들 변형은 별도 글(“압축 Trie/Radix”, “이진 Trie와 XOR/LPM”)에서 상세 구현.

---

## — 삽입/검색/삭제/자동완성/K번째

```cpp
#include <bits/stdc++.h>

using namespace std;

struct Node{ array<Node*,26> nx{}; int pass=0, end=0; Node(){ nx.fill(nullptr);} };
struct Trie {
    Node* r = new Node();
    ~Trie(){ clear(r); }
    void clear(Node* u){ if(!u) return; for(auto*p:u->nx) clear(p); delete u; }
    static int id(char c){ return c-'a'; }

    void insert(const string&s){ Node* u=r; u->pass++; for(char c:s){int i=id(c); if(!u->nx[i]) u->nx[i]=new Node(); u=u->nx[i]; u->pass++;} u->end++; }
    bool search(const string&s)const{ const Node* u=r; for(char c:s){int i=id(c); u=u->nx[i]; if(!u) return false;} return u->end>0; }
    bool erase(const string&s){
        if(!search(s)) return false;
        Node* u=r; u->pass--;
        for(char c:s){
            int i=id(c); Node* v=u->nx[i]; v->pass--;
            if(v->pass==0){ clear(v); u->nx[i]=nullptr; return true; }
            u=v;
        } u->end--; return true;
    }
    int countPrefix(const string&p)const{ const Node* u=r; for(char c:p){int i=id(c); u=u->nx[i]; if(!u) return 0;} return u->pass; }
};

struct S{ string w; int f; bool operator<(const S&o)const{ if(f!=o.f) return f>o.f; return w<o.w; } };
void dfs(Node* u, string&cur, vector<S>&out){
    if(!u) return; if(u->end) out.push_back({cur, u->end});
    for(int i=0;i<26;++i) if(u->nx[i]){ cur.push_back('a'+i); dfs(u->nx[i],cur,out); cur.pop_back(); }
}
vector<S> autocomplete(Trie& T, const string& p, int k){
    Node* u=T.r; for(char c:p){int i=c-'a'; if(!u->nx[i]) return {}; u=u->nx[i];}
    vector<S> res; string cur=p; dfs(u,cur,res); sort(res.begin(),res.end()); if((int)res.size()>k) res.resize(k); return res;
}

bool wild(Node* u, const string& pat, int pos){
    if(!u) return false; if(pos==(int)pat.size()) return u->end>0;
    char c=pat[pos];
    if(c=='?'){ for(int i=0;i<26;++i) if(u->nx[i] && wild(u->nx[i],pat,pos+1)) return true; return false; }
    if(c=='*'){ if(wild(u,pat,pos+1)) return true; for(int i=0;i<26;++i) if(u->nx[i] && wild(u->nx[i],pat,pos)) return true; return false; }
    int i=c-'a'; return u->nx[i] && wild(u->nx[i],pat,pos+1);
}

bool kth(Node* u, long long&k, string&out){
    if(u->end){ if(k<=u->end) return true; k-=u->end; }
    for(int i=0;i<26;++i){ Node* v=u->nx[i]; if(!v) continue; long long sub=v->pass; if(k>sub){ k-=sub; continue; } out.push_back('a'+i); bool ok=kth(v,k,out); if(ok) return true; out.pop_back(); }
    return false;
}
string kthWord(Trie& T, long long k){ if(k<=0||k>T.r->pass) return ""; string o; bool ok=kth(T.r,k,o); return ok?o:""; }

int main(){
    Trie T; for(string w:{"app","apple","application","apply","bat","bad","bag","banner"}) T.insert(w);
    cout<<boolalpha<<"search(app): "<<T.search("app")<<"\n";
    auto s=autocomplete(T,"app",5); for(auto& e:s) cout<<e.w<<" "<<e.f<<"\n";
    cout<<"wild ba?: "<<wild(T.r,"ba?",0)<<"\n";
    for(int k=1;k<=T.r->pass;++k) cout<<k<<":"<<kthWord(T,k)<<"\n";
}
```

---

## 마무리 요약

- **삭제는 `pass`/`end`의 불변식을 지키는가?**(경로 카운트와 중복 수)
- **자동완성/와일드카드/K번째** 등 **실전 기능**은 기본 트라이 위에 간단한 로직을 얹으면 된다.
- **희소 데이터**·**긴 체인**에서는 **압축 Trie(Radix/Patricia)**·메모리 풀·하이브리드 자식 컨테이너를 고려하라.
- 유니코드/로케일 환경에서는 **정규화·대소문자 규칙**을 명확히 정의하고 일관되게 처리하라.
