---
layout: post
title: C++ - string
date: 2024-09-04 19:20:23 +0900
category: Cpp
---
# 문자열(String) 처리 가이드

## 개요: 현대 C++ 문자열 처리 방식

문자열 처리는 프로그래밍에서 가장 기본적이면서도 중요한 작업 중 하나입니다. C++에서는 역사적으로 여러 접근 방식이 발전해왔으며, 현재는 다음과 같은 도구들을 중심으로 문자열을 다룹니다:

- **소유 문자열**: `std::string` - 메모리를 소유하며 수정 가능한 문자열
- **비소유 뷰**: `std::string_view` - 문자열 데이터를 참조만 하는 읽기 전용 뷰
- **유니코드 지원**: `char8_t`(UTF-8), `char16_t`(UTF-16), `char32_t`(UTF-32)
- **고급 처리**: `<format>`, `<charconv>`, `<regex>` 등의 라이브러리

현대 C++의 핵심 철학은 명확합니다: 문자열 데이터를 소유해야 할 때는 `std::string`을 사용하고, 읽기만 필요한 경우에는 `std::string_view`를 통해 불필요한 복사를 피하십시오. 인코딩은 프로젝트 전반에 걸쳐 UTF-8을 기본으로 통일하고, 성능은 알고리즘 선택과 객체 수명 관리에서 얻어집니다.

---

## `std::string`: 기본 사용법과 특징

### 생성과 기본 연산

```cpp
#include <string>
#include <iostream>

int main() {
    std::string greeting = "Hello";
    greeting += " World";  // 문자열 연결
    std::cout << greeting << "\n";  // "Hello World" 출력
}
```

### 문자열 정보 접근

```cpp
std::string text = "example";
size_t length = text.size();        // 문자열 길이 (7)
bool isEmpty = text.empty();        // 빈 문자열 여부
char firstChar = text[0];           // 인덱스 접근 (경계 검사 없음)
char secondChar = text.at(1);       // 안전한 접근 (경계 검사 수행)
```

### 문자열 검색과 수정

```cpp
std::string data = "abcdef";
std::string part = data.substr(1, 3);   // 위치 1부터 3글자: "bcd"

size_t position = data.find("cd");      // "cd" 찾기
if (position != std::string::npos) {
    data.replace(position, 2, "XY");    // "abXYef"로 변경
}
```

성능 특성을 이해하는 것이 중요합니다: `substr`은 추출하는 길이에 비례한 시간이 걸리고, `find`는 일반적으로 선형 시간이 소요되며, `replace`, `insert`, `erase` 같은 수정 연산은 데이터 재배치로 인해 문자열 길이에 비례하는 시간이 필요합니다.

---

## 문자열 조작: 삽입, 삭제, 변환

### 기본 조작 연산

```cpp
std::string message = "hello world";
message.insert(5, ",");           // "hello, world"
message.erase(5, 1);              // "hello world" 복원
message.replace(0, 5, "hi");      // "hi world"
message.append("!!!");            // "hi world!!!"
```

### 성능 최적화 팁

여러 문자열을 반복적으로 연결해야 할 경우, 미리 충분한 공간을 예약하는 것이 효율적입니다:

```cpp
std::string result;
result.reserve(1024);  // 예상되는 최종 크기로 미리 공간 확보
for (const auto& fragment : fragments) {
    result += fragment;
}
```

`shrink_to_fit()`은 메모리 사용량을 줄이기 위한 힌트일 뿐이며, 실제로 메모리가 반환될 것이라는 보장은 없다는 점을 명심하세요.

---

## 문자열 비교와 정렬

```cpp
std::string first = "apple";
std::string second = "banana";

bool areEqual = (first == second);     // 동등성 비교
bool isLess = (first < second);        // 사전식 비교
```

기본 문자열 비교는 바이트 단위로 사전식 순서를 따릅니다. 지역화된 비교나 대소문자를 구분하지 않는 비교가 필요하다면, `std::locale`이나 전문적인 국제화 라이브러리를 고려해야 합니다.

---

## 숫자와 문자열 간 변환

### 간편한 방법 (예외 기반)

```cpp
std::string numberText = std::to_string(42);    // 정수에서 문자열로
int value = std::stoi("123");                   // 문자열에서 정수로
double pi = std::stod("3.14");                  // 문자열에서 실수로
```

이 방법은 사용하기 쉽지만 변환 실패 시 예외를 발생시킵니다. `std::stoi` 같은 함수들은 선행 공백을 무시하고 유효한 숫자 부분만 변환합니다(예: "123abc" → 123).

### 고성능 방법 (예외 없음, C++17)

```cpp
#include <charconv>
#include <system_error>

int parseInteger(std::string_view input) {
    int result = 0;
    auto [endPtr, errorCode] = std::from_chars(
        input.data(), 
        input.data() + input.size(), 
        result
    );
    
    if (errorCode == std::errc{}) {
        return result;  // 성공적으로 변환됨
    }
    // 에러 처리
}
```

`<charconv>` 함수들은 지역 설정에 영향을 받지 않으며 예외를 발생시키지 않아 성능이 우수합니다.

---

## 문자열 입력과 출력

```cpp
#include <iostream>
#include <string>

int main() {
    std::string fullLine;
    
    // 공백을 포함한 전체 라인 읽기
    std::getline(std::cin, fullLine);
    
    std::cout << "입력받은 내용: " << fullLine << "\n";
}
```

`std::cin >> variable`은 공백에서 입력이 끊기므로, 전체 라인을 읽으려면 반드시 `std::getline`을 사용해야 합니다.

---

## 문자열 분할과 토큰화

C++ 표준 라이브러리에는 문자열 분할을 위한 전용 함수가 없으므로, 필요에 따라 직접 구현해야 합니다.

### 공백으로 분할

```cpp
#include <sstream>
#include <vector>
#include <string>

std::vector<std::string> splitByWhitespace(const std::string& text) {
    std::istringstream stream(text);
    std::vector<std::string> tokens;
    std::string token;
    
    while (stream >> token) {
        tokens.push_back(token);
    }
    
    return tokens;
}
```

### 구분자로 분할 (고성능 `string_view` 버전)

```cpp
#include <string_view>
#include <vector>

std::vector<std::string_view> splitByDelimiter(std::string_view text, char delimiter) {
    std::vector<std::string_view> parts;
    size_t start = 0;
    
    while (true) {
        size_t end = text.find(delimiter, start);
        
        if (end == std::string_view::npos) {
            parts.push_back(text.substr(start));
            break;
        }
        
        parts.push_back(text.substr(start, end - start));
        start = end + 1;
    }
    
    return parts;
}
```

`string_view`를 사용한 구현은 메모리를 할당하거나 복사하지 않아 효율적이지만, 원본 문자열이 파괴되지 않도록 수명 관리에 주의해야 합니다.

---

## `std::string_view`: 효율적인 문자열 뷰

`std::string_view`는 문자열 데이터를 소유하지 않고 참조만 하는 경량 객체입니다. 포인터와 길이만을 유지하여 불필요한 복사를 방지합니다.

```cpp
#include <string>
#include <string_view>

// 문자열의 앞부분을 반환하는 함수
std::string_view getPrefix(std::string_view text, size_t length) {
    return text.substr(0, std::min(length, text.size()));
}

int main() {
    std::string original = "Hello, World!";
    std::string_view view = getPrefix(original, 5);  // "Hello" 참조
    
    // 원본 문자열이 수정되면 view는 영향을 받을 수 있음
    original += " More text";
    // 이제 view가 참조하는 메모리는 무효화되었을 수 있음
}
```

### 사용 지침

- **읽기 전용 API**의 매개변수로 적극 활용하세요
- **저장소에는 사용하지 마세요** - 보관이 필요하면 `std::string`으로 복사하세요
- **수명 관리에 주의하세요** - 원본보다 오래 살아남지 않도록 하세요

---

## C 문자열과의 호환성

C 라이브러리와 상호작용할 때는 `c_str()` 메서드를 사용하세요:

```cpp
// C 스타일 함수
void legacyFunction(const char* cString);

std::string modernString = "Modern C++";
legacyFunction(modernString.c_str());  // 안전한 변환
```

C++17부터 `data()` 메서드도 널 종단 문자를 보장하지만, C API와의 호환성을 위해서는 `c_str()`을 사용하는 것이 안전합니다. `string_view::data()`는 널 종단을 보장하지 않으므로 주의가 필요합니다.

---

## 문자열 리터럴과 인코딩

| 리터럴 형식 | 자료형 | 용도 |
|------------|--------|------|
| `"text"` | `const char[N]` | 일반 문자열 |
| `u8"text"` | `const char8_t[N]` | UTF-8 인코딩 (C++20) |
| `u"text"` | `const char16_t[N]` | UTF-16 인코딩 |
| `U"text"` | `const char32_t[N]` | UTF-32 인코딩 |
| `L"text"` | `const wchar_t[N]` | 플랫폼별 와이드 문자열 |
| `R"(raw\ntext)"` | `const char[N]` | 이스케이프 처리 없는 원시 문자열 |
| `"text"s` | `std::string` | `std::string` 객체 (리터럴 접미사) |

```cpp
using namespace std::string_literals;

auto str = "Hello"s;           // std::string 객체
auto view = "World"sv;         // std::string_view 객체
auto utf8 = u8"안녕하세요";    // UTF-8 문자열
```

프로젝트에서는 소스 코드 인코딩과 런타임 문자열 처리를 UTF-8로 통일하는 것이 바람직합니다.

---

## 대소문자 변환과 지역화

단순한 ASCII 문자열의 경우 `std::toupper`와 `std::tolower`를 사용할 수 있습니다:

```cpp
#include <algorithm>
#include <cctype>

std::string toUpperCase(std::string text) {
    std::transform(text.begin(), text.end(), text.begin(),
        [](unsigned char ch) { return std::toupper(ch); });
    return text;
}
```

하지만 유니코드나 복잡한 지역화 규칙이 필요한 경우에는 ICU(International Components for Unicode)나 Boost.Locale 같은 전문 라이브러리를 고려해야 합니다.

---

## 고급 문자열 처리 기능

### 현대적 문자열 포맷팅 (C++20)

```cpp
#include <format>
#include <string>
#include <iostream>

int main() {
    std::string name = "Alice";
    int score = 95;
    
    std::string message = std::format("{}님의 점수는 {}점입니다.", name, score);
    std::cout << message << "\n";
}
```

`<format>` 라이브러리는 타입 안전성과 가독성을 모두 갖춘 현대적인 포맷팅 방식을 제공합니다.

### 정규표현식 (Regex)

```cpp
#include <regex>
#include <string>
#include <iostream>

int main() {
    std::string text = "이메일: user@example.com, 전화: 010-1234-5678";
    std::regex emailPattern(R"(\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b)");
    
    std::smatch matches;
    if (std::regex_search(text, matches, emailPattern)) {
        std::cout << "발견된 이메일: " << matches[0] << "\n";
    }
}
```

표준 정규표현식은 편리하지만 성능과 메모리 사용 측면에서 비용이 클 수 있으니, 고성능이 요구되는 상황에서는 RE2나 PCRE2 같은 대안을 고려해보세요.

---

## 파일 시스템 경로와 문자열

파일 시스템 작업에는 `<filesystem>` 라이브러리를 사용하는 것이 안전하고 이식성이 좋습니다:

```cpp
#include <filesystem>
#include <iostream>

namespace fs = std::filesystem;

int main() {
    fs::path directory = "docs";
    fs::path filename = "report.txt";
    fs::path fullPath = directory / filename;  // 경로 결합
    
    std::cout << "경로: " << fullPath.string() << "\n";
    
    // UTF-8 기반 프로젝트에서는 u8string()을 사용
    std::cout << "UTF-8 경로: " << fullPath.u8string() << "\n";
}
```

운영체제별로 경로 인코딩이 다를 수 있으므로(Windows는 UTF-16, POSIX는 바이트 시퀀스), 파일 시스템 API를 직접 사용하는 것이 문자열 조작보다 안전합니다.

---

## 성능 고려사항과 최적화 기법

### SSO (Small String Optimization)

대부분의 현대 C++ 구현체는 짧은 문자열을 힙에 할당하지 않고 객체 내부에 저장하는 최적화를 수행합니다. 이 최적화의 세부 사항은 구현에 따라 다르지만, 일반적으로 15-23바이트 정도의 문자열은 추가 메모리 할당 없이 처리됩니다.

### 효율적인 문자열 연결

여러 문자열 조각을 연결할 때는 다음 패턴을 따르세요:

```cpp
std::string assembleString(const std::vector<std::string>& parts) {
    std::string result;
    
    // 총 필요한 크기 계산
    size_t totalSize = 0;
    for (const auto& part : parts) {
        totalSize += part.size();
    }
    
    // 한 번에 메모리 할당
    result.reserve(totalSize);
    
    // 조각 추가
    for (const auto& part : parts) {
        result += part;
    }
    
    return result;
}
```

`operator+`를 연속적으로 사용하면 중간 임시 객체가 생성될 수 있으므로, `reserve`와 `append`를 조합하는 것이 더 효율적입니다.

---

## 실전 예제 패턴

### 문자열 치환 유틸리티

```cpp
bool replaceSubstring(std::string& text, std::string_view oldPart, std::string_view newPart) {
    size_t position = text.find(oldPart);
    if (position == std::string::npos) {
        return false;  // 찾지 못함
    }
    
    text.replace(position, oldPart.size(), newPart);
    return true;
}
```

### 문자열 트리밍 (공백 제거)

```cpp
#include <cctype>
#include <string>

std::string trim(const std::string& text) {
    auto isSpace = [](unsigned char ch) { return std::isspace(ch); };
    
    // 앞쪽 공백 찾기
    auto front = std::find_if_not(text.begin(), text.end(), isSpace);
    // 뒤쪽 공백 찾기
    auto back = std::find_if_not(text.rbegin(), text.rend(), isSpace).base();
    
    if (back <= front) {
        return "";  // 빈 문자열 또는 공백만 있는 경우
    }
    
    return std::string(front, back);
}
```

### 문자열 연결 (Join)

```cpp
std::string joinStrings(const std::vector<std::string>& strings, std::string_view separator) {
    if (strings.empty()) {
        return "";
    }
    
    std::string result;
    
    // 필요한 총 공간 계산
    size_t totalLength = (strings.size() - 1) * separator.size();
    for (const auto& str : strings) {
        totalLength += str.size();
    }
    
    result.reserve(totalLength);
    result = strings[0];
    
    for (size_t i = 1; i < strings.size(); ++i) {
        result += separator;
        result += strings[i];
    }
    
    return result;
}
```

### 고성능 CSV 파서 (단순 버전)

```cpp
std::vector<std::string_view> parseCSVLine(std::string_view line) {
    std::vector<std::string_view> fields;
    size_t start = 0;
    
    while (start < line.size()) {
        size_t end = line.find(',', start);
        
        if (end == std::string_view::npos) {
            fields.push_back(line.substr(start));
            break;
        }
        
        fields.push_back(line.substr(start, end - start));
        start = end + 1;
    }
    
    return fields;
}
```

실제 CSV 형식은 따옴표 처리, 이스케이프 시퀀스, 빈 필드 등 더 복잡한 규칙을 포함하므로, 프로덕션 환경에서는 전문 라이브러리 사용을 고려하세요.

---

## 로그 라인 파서: 종합 예제

로그 라인에서 정보를 추출하는 실용적인 예제를 살펴보겠습니다:

```cpp
#include <string>
#include <string_view>
#include <optional>
#include <iostream>

struct LogEntry {
    std::string_view level;
    std::string_view timestamp;
    std::string_view message;
    std::string_view user;
};

std::optional<LogEntry> parseLogLine(std::string_view line) {
    // 형식: "LEVEL timestamp=... message=... user=..."
    
    // 레벨 추출 (첫 번째 공백까지)
    size_t firstSpace = line.find(' ');
    if (firstSpace == std::string_view::npos) {
        return std::nullopt;
    }
    
    std::string_view level = line.substr(0, firstSpace);
    
    // 키-값 쌍 추출을 위한 헬퍼 함수
    auto extractValue = [&line](std::string_view key) -> std::optional<std::string_view> {
        std::string_view searchKey = std::string(key) + "=";
        size_t keyStart = line.find(searchKey);
        
        if (keyStart == std::string_view::npos) {
            return std::nullopt;
        }
        
        size_t valueStart = keyStart + searchKey.size();
        size_t valueEnd = line.find(' ', valueStart);
        
        if (valueEnd == std::string_view::npos) {
            return line.substr(valueStart);
        }
        
        return line.substr(valueStart, valueEnd - valueStart);
    };
    
    auto timestamp = extractValue("timestamp");
    auto message = extractValue("message");
    auto user = extractValue("user");
    
    if (!timestamp || !message || !user) {
        return std::nullopt;
    }
    
    return LogEntry{level, *timestamp, *message, *user};
}

int main() {
    std::string logLine = "INFO timestamp=2023-10-05T14:30:00 message=시스템시작 user=admin";
    
    if (auto entry = parseLogLine(logLine)) {
        std::cout << "레벨: " << entry->level << "\n";
        std::cout << "시간: " << entry->timestamp << "\n";
        std::cout << "메시지: " << entry->message << "\n";
        std::cout << "사용자: " << entry->user << "\n";
    }
}
```

이 예제는 `std::string_view`를 사용하여 메모리 복사 없이 효율적으로 문자열을 파싱하는 방법을 보여줍니다. 다만, 반환된 뷰 객체들은 원본 문자열이 유효한 동안만 사용할 수 있다는 점을 주의하세요.

---

## 주의사항과 모범 사례

1. **수명 관리**: `std::string_view`는 원본 데이터의 수명을 초과해서 사용하지 마세요. 저장이 필요하다면 `std::string`으로 복사하세요.

2. **메모리 무효화**: `std::string`에 대한 연산(삽입, 삭제, 재할당 등)은 기존 반복자, 포인터, 참조를 무효화할 수 있습니다.

3. **C 문자열 포인터**: `c_str()`로 얻은 포인터는 문자열 객체가 수정되거나 파괴된 후에는 사용하지 마세요.

4. **안전한 접근**: 경계를 벗어난 접근이 우려된다면 `at()` 메서드를 사용하여 예외를 통한 안전한 접근을 하세요.

5. **동시성**: 여러 스레드에서 동일한 문자열을 수정할 때는 적절한 동기화 메커니즘을 사용하세요.

---

## 핵심 요약

C++에서 효과적인 문자열 처리를 위해서는 몇 가지 기본 원칙을 따르는 것이 중요합니다:

- **적절한 도구 선택**: 데이터 소유가 필요하면 `std::string`을, 읽기 전용 뷰만 필요하면 `std::string_view`를 사용하세요.
- **인코딩 일관성**: 프로젝트 전체에서 UTF-8 인코딩을 표준으로 채택하면 많은 문제를 예방할 수 있습니다.
- **성능 인식**: 문자열 연결은 `reserve()`로 사전 할당하고, 숫자 변환은 `<charconv>`를 우선 고려하세요.
- **수명 주의**: `string_view`의 수명을 관리하고, 특히 저장소에는 사용하지 마세요.
- **테스트 철저성**: 빈 문자열, 긴 문자열, 특수 문자, 유니코드 등 다양한 경계 조건을 테스트하세요.

문자열 처리는 C++ 프로그래밍의 기본이자 핵심입니다. 이 가이드의 원칙과 패턴을 이해하고 적용하면 더 안전하고 효율적이며 유지보수하기 쉬운 코드를 작성할 수 있을 것입니다. 각 프로젝트의 요구사항에 맞추어 인코딩 정책, API 디자인, 성능 목표를 명확히 정의하고 일관되게 적용하는 것이 장기적인 성공의 열쇠입니다.