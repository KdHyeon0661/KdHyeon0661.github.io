---
layout: post
title: C++ - string
date: 2024-09-04 19:20:23 +0900
category: Cpp
---
# 문자열(string)

## 개요 — 오늘날 문자열은 무엇으로 다루나?

- 과거: `char*`, `char[]`, `std::strlen/std::strcpy` 등 **C-스타일 문자열**
- 현재: **`std::string`(가변/소유)**, **`std::string_view`(비소유/뷰)** 중심
- 유니코드: `char8_t`(UTF-8), `char16_t`(UTF-16), `char32_t`(UTF-32), `wchar_t`(플랫폼 가변)
- 포맷팅/파싱: `<format>`, `<charconv>`(to/from_chars), `<regex>` 등

**핵심 철학**:
- **소유는 `std::string`**, **비소유는 `std::string_view`**
- **인코딩은 기본 UTF-8**(프로젝트 정책으로 명확히)
- **성능은 알고리즘 & 수명 관리**로 얻는다

---

## `std::string` 기초

### 기본 사용

```cpp
#include <string>
#include <iostream>

int main() {
    std::string s = "Hello";
    s += " World";
    std::cout << s << "\n"; // Hello World
}
```

### 길이/비어 있음/접근

```cpp
std::string s = "abcdef";
auto n = s.size();         // 또는 s.length()
bool empty = s.empty();
char c0 = s[0];            // 경계 검사 없음 (UB 위험)
char c1 = s.at(1);         // 경계 검사, 예외 발생 가능
```

### 부분 문자열/탐색/치환

```cpp
std::string s = "abcdef";
std::string sub = s.substr(1, 3);   // "bcd"

auto pos = s.find("cd");            // 2, 못 찾으면 std::string::npos
if (pos != std::string::npos) {
    s.replace(pos, 2, "XY");        // "abXYef"
}
```

**복잡도 스케치**

- `substr`: $$\mathcal{O}(k)$$ (k는 추출 길이)
- `find`: 평균 $$\mathcal{O}(n)$$(단순), 구현에 따라 최적화 가능
- `replace/insert/erase`: **재배치/복사**로 $$\mathcal{O}(n)$$

---

## 삽입/삭제/변형 — 안전·성능 팁 포함

```cpp
std::string s = "hello world";
s.insert(5, ",");                 // "hello, world"
s.erase(5, 1);                    // "hello world"
s.replace(0, 5, "hi");            // "hi world"
s.append("!!!");                  // "hi world!!!"
```

- **여러 번 덧붙이기**가 예상되면 `reserve`로 용량 선할당:

```cpp
std::string out;
out.reserve(1024);  // 예상 크기
for (...) { out += piece; }
```

- `shrink_to_fit()`은 **비보장 힌트**(필요 시 메모리 회수 시도)임을 인지.

---

## 비교/정렬/사전식

```cpp
std::string a = "abc", b = "abd";
bool eq = (a == b);   // 사전식 비교 지원(==, !=, <, >, <=, >=)
```

- 기본 비교는 **바이트 시퀀스 사전식**.
- **현지화/대소문자 무시 비교**가 필요하면 `std::locale`/`std::toupper`(주의: 코드포인트/언어 규칙의 복잡성) 또는 ICU 같은 라이브러리 고려.

---

## 숫자 ↔ 문자열 변환

### 쉬운 방법 (예외 기반)

```cpp
std::string s = std::to_string(42); // "42"
int i = std::stoi("123");           // 123
double d = std::stod("3.14");       // 3.14
```

- `stoi/stol/stoll/stof/stod/stold`는 **예외**(invalid_argument, out_of_range)를 던진다.
- 부분 파싱 규칙: `"123abc"` → 123 까지 읽고 멈춤(뒤에 문자가 있어도 일부 구현에선 예외 없이 파싱됨).

### 고성능/비예외 `<charconv>` (C++17)

```cpp
#include <charconv>
#include <system_error>

int i;
auto src = std::string("123");
auto [ptr, ec] = std::from_chars(src.data(), src.data()+src.size(), i);
if (ec == std::errc{}) {
    // 성공: i == 123, ptr는 파싱 끝 위치
}
```

```cpp
char buf[64];
auto [ptr2, ec2] = std::to_chars(buf, buf+sizeof(buf), 3.14159, std::chars_format::general, 6);
std::string s(buf, ptr2);
```

- **빠르고 locale-free**, 예외 없음. 실패는 `ec`로 확인.

---

## 입력/출력/라인 처리

```cpp
#include <iostream>
#include <string>

int main() {
    std::string line;
    std::getline(std::cin, line);  // 공백 포함 한 줄 입력
    std::cout << ">" << line << "\n";
}
```

- `std::cin >> word;`는 **공백에서 끊김**. 라인 전체가 필요하면 `getline`.

---

## 토큰화/스플릿(표준 유틸 부재 → 직접 구현)

### 공백 기준

```cpp
#include <sstream>
#include <vector>
#include <string>

std::vector<std::string> split_ws(const std::string& s) {
    std::istringstream iss(s);
    std::vector<std::string> out;
    for (std::string w; iss >> w; ) out.push_back(w);
    return out;
}
```

### 구분자 기준 (단일 문자)

```cpp
#include <string_view>
#include <vector>
#include <string>

std::vector<std::string_view> split_sv(std::string_view s, char delim) {
    std::vector<std::string_view> out;
    size_t pos = 0;
    while (true) {
        size_t next = s.find(delim, pos);
        if (next == std::string_view::npos) {
            out.emplace_back(s.substr(pos));
            break;
        }
        out.emplace_back(s.substr(pos, next - pos));
        pos = next + 1;
    }
    return out;
}
```

> `string_view` 버전은 **비할당/비복사**이지만 **수명에 주의**(원본이 살아 있어야 함).

---

## `std::string_view` — 비소유 뷰(매우 중요)

- “문자열 데이터를 **복사하지 않고** 바라보기” 위한 **얇은 포인터+길이** 뷰.
- 소유가 없으므로 **댕글링** 위험: 원본이 소멸/재배치되면 **모든 view 무효**.

```cpp
#include <string>
#include <string_view>

std::string_view head(std::string_view s, size_t n) {
    return s.substr(0, std::min(n, s.size()));
}

int main() {
    std::string base = "abcdef";
    std::string_view v = head(base, 3); // "abc"
    base += "XYZ"; // 재할당이 일어날 수 있음 → v가 가리키던 메모리 무효화 가능
}
```

**가이드**

- **입력 전용** API에 `std::string_view` 적극 사용 (파일 경로, 키, 검색 패턴 등).
- 저장/보관 금지(특히 컨테이너에): 보관하려면 **복사하여 std::string**으로 소유.

---

## C 문자열과 상호운용

- C API 호출 시 `c_str()` 사용:

```cpp
extern "C" void c_api(const char*);

std::string s = "hi";
c_api(s.c_str()); // NUL-terminated 보장
```

- `data()`는 C++17부터 NUL 종단 포함(문자열에 따라 다르지만 일반적으로 `data()==c_str()`와 동일한 메모리 시작 주소).
- **주의**: `string_view::data()`는 NUL 보장 없음.

---

## 문자열 리터럴 — 인코딩·접두/접미사

| 리터럴 예 | 의미 |
|---|---|
| `"text"` | `const char[N]` |
| `u8"text"` | `const char8_t[N]` (UTF-8, C++20) |
| `u"text"` | `const char16_t[N]` (UTF-16) |
| `U"text"` | `const char32_t[N]` (UTF-32) |
| `L"text"` | `const wchar_t[N]` (플랫폼 가변) |
| `R"(raw\ntext)"` | **Raw literal**(이스케이프 무시) |
| `"text"s` | `std::string` 리터럴(네임스페이스 `std::literals`) |

```cpp
using namespace std::string_literals;
auto s  = "hello"s;      // std::string
auto sv = "hello"sv;     // std::string_view
auto u8 = u8"안녕";       // UTF-8 (char8_t[])
```

- **정책**: 소스 파일 인코딩을 UTF-8로 통일, 유니코드 텍스트는 가급적 **UTF-8 문자열**로 처리.

---

## 인코딩/로케일/대소문자 변환

- `std::toupper`/`std::tolower`는 **단순 ASCII 가정**시 안전. 유니코드 전반 처리엔 부정확.
- **정말로 국제화가 필요**하면 ICU, Boost.Text 등 전문 라이브러리 고려.

```cpp
#include <algorithm>
#include <cctype>

void to_upper_ascii(std::string& s) {
    std::transform(s.begin(), s.end(), s.begin(),
                   [](unsigned char ch){ return std::toupper(ch); });
}
```

---

## 고급 기능: `<format>`(C++20), `<regex>`

### 현대적 포맷팅

```cpp
#include <format>
#include <string>
#include <iostream>

int main() {
    std::string name = "Kim";
    int n = 7;
    std::cout << std::format("Hello {}, you have {} messages\n", name, n);
}
```

- 타입 안전, `printf`보다 가독성 우수.
- 지역화/유니코드 폭 처리 등은 구현에 따라 다름.

### 정규식

```cpp
#include <regex>
#include <string>
#include <iostream>

int main() {
    std::string s = "abc-123-def";
    std::regex re(R"(\d+)");
    std::smatch m;
    if (std::regex_search(s, m, re)) {
        std::cout << m.str() << "\n"; // 123
    }
}
```

- `<regex>`는 **간편**하지만 **성능·메모리 비용**이 큰 편.
- 고성능이 절대적이면 RE2/PCRE2 등 대안 고려.

---

## 파일시스템/경로와 문자열

```cpp
#include <filesystem>
#include <string>
#include <iostream>

namespace fs = std::filesystem;

int main(){
    for (auto& e : fs::directory_iterator(".")) {
        std::cout << e.path().string() << "\n";   // 플랫폼 네이티브 → string
        // UTF-8 프로젝트면: e.path().u8string() (char8_t 기반)
    }
}
```

- **경로 인코딩**은 OS별 차이(Windows: UTF-16 와이드 API, POSIX: 바이트 시퀀스).
- 응용은 **파일시스템 API** 우선 사용, 문자열로 무리한 직접 조작 최소화.

---

## 성능·메모리 — SSO/연결/빌더 패턴

- **SSO(Small String Optimization)**: 짧은 문자열을 **동적 할당 없이** 내부 버퍼에 저장(크기와 정책은 **구현 의존적**).
- **여러 조각을 반복해서 연결**할 땐:
  - 1) `reserve` 로 용량 확보
  - 2) `std::string` + `append`/`push_back`
  - 3) 또는 한 번에 `std::format`/`fmt::format`로 구성

```cpp
std::string out;
out.reserve(4096);
for (const auto& piece : pieces) out += piece;
```

- **`operator+` 체인**은 중간 임시가 생겨 비용↑ → 컴파일러 최적화 기대 가능하지만 명시적 `append`가 안정적.

---

## 안전(수명) 체크리스트 — 댕글링/무효화/참조

- [ ] **string_view 보관 금지**(원본 재할당·소멸 시 치명적)
- [ ] `std::string`에 `push_back`/`append`로 재할당되면 **이전의 `char*`/참조/이터레이터 무효화**
- [ ] C API에 건네준 `c_str()` 포인터 **오래 보관하지 않기**
- [ ] `operator[]`는 **검사 없음** → 안전이 중요하면 `at()`
- [ ] 멀티스레딩 접근 시 **동기화**(공유 변경 금지)

---

## 예제 모음 — 실전 패턴

### 안전한 찾기/치환 유틸

```cpp
#include <string>

bool replace_first(std::string& s, std::string_view from, std::string_view to) {
    auto pos = s.find(from);
    if (pos == std::string::npos) return false;
    s.replace(pos, from.size(), to);
    return true;
}
```

### 트림(trim) — 공백 제거

```cpp
#include <string>
#include <string_view>
#include <cctype>

inline bool is_space(unsigned char c){ return std::isspace(c); }

std::string trim(std::string s) {
    auto first = s.find_first_not_of(" \t\n\r\f\v");
    if (first == std::string::npos) return {};
    auto last  = s.find_last_not_of(" \t\n\r\f\v");
    return s.substr(first, last - first + 1);
}
```

### 조인(join) — 간단 구현

```cpp
#include <string>
#include <vector>

std::string join(const std::vector<std::string>& v, std::string_view sep){
    if (v.empty()) return {};
    std::string out;
    size_t total = (v.size()-1) * sep.size();
    for (auto& s : v) total += s.size();
    out.reserve(total);
    for (size_t i=0; i<v.size(); ++i){
        if (i) out += sep;
        out += v[i];
    }
    return out;
}
```

### 빠른 수치 파싱 — 부분 실패 감지

```cpp
#include <charconv>
#include <string>
#include <system_error>
#include <optional>

std::optional<int> parse_int(std::string_view s){
    int val{};
    auto [p, ec] = std::from_chars(s.data(), s.data()+s.size(), val);
    if (ec == std::errc{} && p == s.data()+s.size()) return val; // 완전 소비
    return std::nullopt;
}
```

### CSV 라인 파서(간단, 따옴표 미해결 버전)

```cpp
#include <string>
#include <vector>

std::vector<std::string_view> split_csv(std::string_view s){
    std::vector<std::string_view> out;
    size_t pos=0;
    while(true){
        size_t next = s.find(',', pos);
        if (next == std::string_view::npos) { out.emplace_back(s.substr(pos)); break; }
        out.emplace_back(s.substr(pos, next-pos));
        pos = next+1;
    }
    return out;
}
```

> 따옴표/이스케이프/빈 필드 등 **실제 CSV**는 복잡 → 라이브러리 사용 고려.

---

## 해시/키 조회 — `unordered_map`과 이종 조회

- `unordered_map<std::string, V>`에서 **키 조회 시 임시 `std::string` 생성** 비용 회피를 위해
  “투명(transparent) 비교/해시”를 구성하면 `std::string_view`로 바로 조회 가능.

```cpp
#include <string>
#include <string_view>
#include <unordered_map>

struct SvHash {
    using is_transparent = void; // 이종 조회 허용
    size_t operator()(std::string_view s) const noexcept {
        return std::hash<std::string_view>{}(s);
    }
};
struct SvEq {
    using is_transparent = void;
    bool operator()(std::string_view a, std::string_view b) const noexcept {
        return a == b;
    }
};

int main(){
    std::unordered_map<std::string, int, SvHash, SvEq> m;
    m.emplace("alpha", 1);
    auto it = m.find(std::string_view{"alpha"}); // string_view로 직접 조회
}
```

---

## 테스트/디버깅 팁

- **가시화**: 길이/코드포인트/헥사덤프 유틸 준비
- **경계**: 빈 문자열, 매우 긴 문자열, NUL 포함, 유니코드 결합 문자
- **Sanitizer**: ASan/UBSan로 댕글링/오버런 감시
- **속도**: 마이크로벤치(google-benchmark 등)로 `+` vs `append`, `stoi` vs `from_chars` 비교

---

## 작은 종합 예제 — 로그 라인 파서(UTF-8 가정)

> 요구: `"LEVEL ts=... msg=... user=..."` 형식에서 `level/msg/user`를 추출.
> 제약: **복사 최소화**(string_view), **안전한 실패**(optional), **트림**.

```cpp
#include <string>
#include <string_view>
#include <optional>
#include <iostream>

struct LogRec {
    std::string_view level, msg, user;
};

static std::string_view trim_sv(std::string_view s){
    auto issp = [](unsigned char c){ return c==' '||c=='\t'||c=='\n'||c=='\r'; };
    size_t a=0, b=s.size();
    while (a<b && issp((unsigned char)s[a])) ++a;
    while (b>a && issp((unsigned char)s[b-1])) --b;
    return s.substr(a, b-a);
}

std::optional<LogRec> parse_log(std::string_view line){
    auto sp1 = line.find(' ');
    if (sp1 == std::string_view::npos) return std::nullopt;
    std::string_view level = line.substr(0, sp1);

    auto get_kv = [](std::string_view s, std::string_view key)->std::optional<std::string_view>{
        auto p = s.find(key);
        if (p==std::string_view::npos) return std::nullopt;
        p += key.size();
        auto end = s.find(' ', p);
        return s.substr(p, end==std::string_view::npos ? s.size()-p : end-p);
    };

    std::string_view rest = line.substr(sp1+1);
    auto msg  = get_kv(rest, "msg=");
    auto user = get_kv(rest, "user=");
    if (!msg || !user) return std::nullopt;

    return LogRec{ trim_sv(level), trim_sv(*msg), trim_sv(*user) };
}

int main(){
    std::string line = "INFO ts=2025-11-10 msg=Hello user=kim";
    if (auto rec = parse_log(line)){
        std::cout << "level=" << rec->level << " msg=" << rec->msg << " user=" << rec->user << "\n";
    }
}
```

- 입력은 `std::string`(소유), 파서는 **`string_view`**로 **비복사** 파싱.
- `line`이 **살아 있는 동안만** `rec->xxx` 사용 가능(수명 준수).

---

## 요약 체크리스트

- [ ] **소유**: `std::string` / **비소유**: `std::string_view`
- [ ] **성능**: 반복 연결은 `reserve` + `append`; 파싱은 `<charconv>` 우선
- [ ] **안전**: view 보관 금지, 재할당 후 포인터/참조/이터레이터 무효화 주의
- [ ] **인코딩**: UTF-8 정책, 리터럴 접두/접미사 이해(`u8`, `"..."s`, `"..."sv`)
- [ ] **포맷팅**: `<format>` 사용, 레거시는 점진 교체
- [ ] **경계 사례**: 빈 문자열, 긴 입력, NUL/비ASCII, 잘못된 숫자 문자열
- [ ] **테스트/툴링**: Sanitizer/정적 분석/벤치마크

---

## 부록 A) 자주 하는 실수와 교정

1) `string_view`로 반환하고 원본을 바로 파괴
```cpp
std::string_view f(){
    std::string s = "x";
    return s;              // ❌ 수명 종료 → 댕글링
}
```

2) `operator+` 체인 과다
```cpp
auto r = a + b + c + d; // ❌ 임시 다수
// ✅ reserve + append 또는 format 사용
```

3) `stoi` 예외 미처리
```cpp
try { int x = std::stoi(s); } catch(...) { /* 처리 */ }
```
또는 `<charconv>`로 전환해 실패를 코드로 처리.

4) 경로를 문자열로 무리하게 조작
- **`std::filesystem::path`**로 합성/변환 후 `.string()`/`.u8string()` 사용.

---

## 부록 B) 빠른 레퍼런스

- 생성/대입: 기본/복사/이동/리터럴(suffix `s`)
- 접근: `size()`, `empty()`, `operator[]`, `at()`, `front()`, `back()`, `c_str()`, `data()`
- 변경: `clear()`, `push_back()`, `append()`, `insert()`, `erase()`, `replace()`, `resize()`, `reserve()`, `shrink_to_fit()`
- 탐색: `find`, `rfind`, `find_first_of/not_of`, `starts_with`, `ends_with`(C++20)
- 비교: 연산자/`compare`/`<=>`(C++20 3-way)
- 뷰: `std::string_view`(비소유, 수명 주의)
- 변환: `stoi/stol/stof/stod`(예외) / `from_chars/to_chars`(비예외)

---

# 결론

- `std::string`은 **안전한 소유 문자열**의 기본 단위, `std::string_view`는 **고성능 읽기 뷰**.
- **인코딩/수명/성능**을 명확히 하면 문자열 처리는 **예측 가능하고 빠르게** 된다.
- 이 글의 유틸/패턴을 기반으로 프로젝트 규약(인코딩·API 시그니처·포맷팅·파싱)을 문서화해 일관성을 유지하자.
