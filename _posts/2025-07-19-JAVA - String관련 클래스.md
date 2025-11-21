---
layout: post
title: Java - String관련 클래스
date: 2025-07-19 22:20:23 +0900
category: Java
---
# String, StringBuilder, StringBuffer

## 한눈에 보는 차이

| 항목 | String | StringBuilder | StringBuffer |
|---|---|---|---|
| 변경 가능성 | **불변(immutable)** | 가변(mutable) | 가변(mutable) |
| 쓰레드 안전성 | 불변 특성상 **안전** | **비보장**(싱글 스레드용) | 메서드 **synchronized** (기본 쓰레드 안전) |
| 주 용도 | 식별자/키/메시지/로그, 변하지 않는 텍스트 | 루프에서 반복 연결·삽입·삭제 | 여러 스레드가 **같은 인스턴스**를 건드리는 특수 상황 |
| 성능 특성 | 변경 시 새 객체(할당↑, GC↑) | 변경 누적에 효율적 | 동기화 비용으로 Builder보다 느림 |
| 내부 표현 | **JDK 9+ Compact Strings**: `byte[]` + coder(Latin1/UTF-16) | `char[]` 기반 버퍼(추가 시 확장) | `char[]` 기반 버퍼 + 동기화 |

> 바이트코드 수준에서 `a + b`(컴파일타임 상수 결합 제외)는 **대부분 `StringBuilder`로 변환**됩니다. 다만 **루프 내부의 `+=`** 는 **매 반복 새 Builder**가 만들어질 수 있어 비용이 큽니다.

---

## String — 불변(Immutable)의 의미와 이점

### 불변의 이점

- **안전성**: 공유해도 값이 변하지 않음 → 동시성 버그 감소.
- **해시 캐시** 가능: `hashCode()`를 캐시해 **Map 키**로 적합.
- **문자열 상수 풀(String Pool)** 운용: 같은 리터럴은 재사용되어 **메모리 절감**.
- **보안/캐싱**: 파라미터 검증/로그/메시지에 적합.

### 내부 표현(요약)

- **JDK 9+**: Compact Strings — 문자열이 **Latin-1**로 표현 가능하면 1바이트 배열 + coder(플래그), 아니면 UTF-16(2바이트).
  (이 최적화는 개발자가 직접 제어하지 않지만, 평균 메모리 효율을 개선합니다.)

### `==` vs `equals()`

```java
String a = "Hello";
String b = "Hello";
String c = new String("Hello"); // 강제 새 인스턴스

System.out.println(a == b);       // true (풀 공유)
System.out.println(a == c);       // false (참조 다름)
System.out.println(a.equals(c));  // true (내용 같음)
```
- **항상 내용 비교는 `equals`** 를 사용.

### String Pool / `intern()`

```java
String x = new String("java");
String y = x.intern();        // 풀에 있는 "java" 참조 반환
System.out.println(y == "java"); // true
```
- `intern()`은 **풀에 등록**하므로 과도한 사용은 메모리/시간 비용. **특수 케이스**(매우 반복적인 동일 문자열)에서만.

### 코드 포인트 주의(이모지, 한자 확장 등)

- `length()`는 **UTF-16 코드 유닛 개수**, 사람이 인식하는 “문자 수”와 다를 수 있음.
- 보조 평면 문자 처리:
```java
String s = "A😊B";
System.out.println(s.length());                   // 4 (😊가 2 유닛)
System.out.println(s.codePointCount(0, s.length())); // 3
```
- 안전한 순회:
```java
for (int i = 0; i < s.length(); ) {
    int cp = s.codePointAt(i);
    // cp 처리
    i += Character.charCount(cp);
}
```

---

## StringBuilder — 단일 스레드에서의 고성능 가변 문자열

### 기본 패턴

```java
StringBuilder sb = new StringBuilder(256); // 예상 용량 지정 권장
sb.append("id=").append(42).append("&name=").append("kim");
String result = sb.toString();
```
- 체이닝으로 **중간 객체 생성 없음**.
- **초기 용량(capacity)** 를 넉넉히 주면 확장 비용 감소.

### 주요 API

```java
append(..)         // 뒤에 붙이기
insert(pos, ..)    // 삽입
delete(start,end)  // 구간 삭제
replace(s,e,str)   // 구간 치환
reverse()          // 역순
setCharAt(i,ch)    // 단일 문자 수정
ensureCapacity(n)  // 최소 용량 보장
trimToSize()       // 실제 길이에 맞게 줄이기
```

### 용량 확장 전략

- 확장 시 내부적으로 **대략 2배(+2)** 수준으로 버퍼를 키움(구현 세부는 JDK에 따라 상이할 수 있으나 “지수적 증가”가 핵심).
- 예상 크기를 아는 경우 **`new StringBuilder(expected)`** 또는 **`ensureCapacity`** 사용.

### 루프에서의 `+=` vs Builder

```java
// 비효율: 매 반복 새 Builder 생성되어 연결
String s = "";
for (int i = 0; i < 100_000; i++) s += i;

// 효율적: 하나의 Builder에 누적
StringBuilder sb = new StringBuilder(600_000);
for (int i = 0; i < 100_000; i++) sb.append(i);
String s2 = sb.toString();
```

---

## StringBuffer — 동기화된 가변 문자열

### 언제 쓰나?

- 다수의 스레드가 **하나의 동일 인스턴스**를 공유·수정하는 경우.
- 메서드 수준 동기화(`synchronized`) 제공 → **원자적 복합 연산은 아님**에 유의.
  - 예: `if (buf.length() > 0) buf.delete(0,1);`는 **두 호출 사이에 끼어들기 가능**.
    복합 작업은 **외부에서 추가 동기화** 필요.

### 대안: 스레드 지역 빌더

- 대개 멀티스레드 환경에서도 **스레드마다 별도 `StringBuilder`** 를 쓰면 충분.
- 고급: `ThreadLocal<StringBuilder>` 로 재사용(사용 후 `setLength(0)`).

---

## 포맷팅·결합·분해 — 주변 API와의 협업

### `String.format` / `Formatter`

```java
String m = String.format("name=%s, score=%.2f", "kim", 98.123);
```
- 가독성↑, 다국어·포맷 제어 유리.
- 빈번한 루프에서는 **포맷터 비용으로 느릴 수 있음**.

### `StringJoiner` / `Collectors.joining`

```java
// 1) StringJoiner
var joiner = new java.util.StringJoiner(", ", "[", "]");
joiner.add("a").add("b");
System.out.println(joiner.toString()); // [a, b]

// 2) Stream + joining
var s = java.util.stream.Stream.of("a","b","c")
        .collect(java.util.stream.Collectors.joining(","));
```

### 분해/치환

- `split(regex)` 는 **정규식 비용**. 단순 구분자는 `indexOf`/수동 파싱 고려.
- 대량 치환은 `String.replace`(리터럴), 정규식은 `replaceAll`.

---

## 파일·인코딩·문자셋

### 인코딩 명시

```java
var s = java.nio.file.Files.readString(path, java.nio.charset.StandardCharsets.UTF_8);
java.nio.file.Files.writeString(path, s, java.nio.charset.StandardCharsets.UTF_8);
```
- 네트워크/파일 I/O에서는 **항상 문자셋 지정**. 플랫폼 기본값 의존 지양.

### `getBytes(Charset)` / `new String(bytes, Charset)`

```java
byte[] utf8 = "한글".getBytes(java.nio.charset.StandardCharsets.UTF_8);
String back = new String(utf8, java.nio.charset.StandardCharsets.UTF_8);
```

---

## 성능 비교(설명 + 예제)

> 정확한 비교는 **JMH** 같은 마이크로벤치마크 프레임워크로 **워밍업/옵트레이션 제어** 하에 수행해야 합니다. 아래 코드는 개념 예시입니다.

```java
public class StringPerf {
    public static void main(String[] args) {
        int n = 100_000;

        // 1) String (비효율)
        long t0 = System.nanoTime();
        String s = "";
        for (int i = 0; i < n; i++) s += i;
        long t1 = System.nanoTime();

        // 2) StringBuilder (권장)
        long t2 = System.nanoTime();
        StringBuilder sb = new StringBuilder(n * 6);
        for (int i = 0; i < n; i++) sb.append(i);
        String s2 = sb.toString();
        long t3 = System.nanoTime();

        // 3) StringBuffer (동기화 비용 추가)
        long t4 = System.nanoTime();
        StringBuffer sf = new StringBuffer(n * 6);
        for (int i = 0; i < n; i++) sf.append(i);
        String s3 = sf.toString();
        long t5 = System.nanoTime();

        System.out.printf("String      : %d ms%n", (t1 - t0)/1_000_000);
        System.out.printf("StringBuilder: %d ms%n", (t3 - t2)/1_000_000);
        System.out.printf("StringBuffer: %d ms%n", (t5 - t4)/1_000_000);
    }
}
```

일반적으로: **Builder가 가장 빠름** > Buffer(동기화 비용) >> String(루프 연결 시 매우 느림).
단, **컴파일타임 상수 결합**은 String도 비용이 거의 없습니다:
```java
String s = "abc" + "def"; // 컴파일 시 "abcdef" 로 상수 폴딩
```

---

## 자주 쓰는 코드 스니펫

### 안전한 코드포인트 기반 잘라내기(이모지 깨짐 방지)

```java
static String safeCut(String s, int codePoints) {
    int i = 0, count = 0;
    while (i < s.length() && count < codePoints) {
        i += Character.charCount(s.codePointAt(i));
        count++;
    }
    return s.substring(0, i);
}
```

### 대용량 로그 메시지 구성

```java
StringBuilder sb = new StringBuilder(1024);
sb.append("reqId=").append(reqId)
  .append(" user=").append(user)
  .append(" items=").append(items.size());
log.info(sb.toString());
```

### ThreadLocal 재사용(고급)

```java
final class ReusableBuilder {
    private static final ThreadLocal<StringBuilder> TL =
        ThreadLocal.withInitial(() -> new StringBuilder(1024));

    static String build(java.util.function.Function<StringBuilder, StringBuilder> f) {
        StringBuilder sb = TL.get();
        sb.setLength(0);
        f.apply(sb);
        return sb.toString();
    }
}
// 사용
String msg = ReusableBuilder.build(sb -> sb.append("id=").append(10));
```

---

## 함정 & 베스트 프랙티스 체크리스트

| 주제 | 권장/주의 |
|---|---|
| 루프에서 연결 | **`StringBuilder` 사용**. `+=`는 피함 |
| 초기 용량 | 예상 크기 알면 **생성 시 capacity 지정** |
| 다중 스레드 | 각 스레드가 **자기 Builder** 사용. 정말 공유해야 하면 `StringBuffer` + **외부 동기화로 복합 연산 보호** |
| `format` 성능 | 빈번한 루프에선 **Builder**. 포맷은 가독성/국제화 용도 |
| 분해/치환 | **정규식 비용** 고려. 단순 구분은 수동 파싱/`indexOf` |
| 인코딩 | 파일/네트워크 I/O는 **항상 Charset 명시(UTF-8 권장)** |
| 이모지/합성 문자 | **코드포인트 API** 사용(`codePointAt`, `charCount`, `codePointCount`) |
| `intern()` | 특수 케이스 외 남용 금지(메모리/시간 비용) |
| 상수 결합 | 리터럴 결합은 **상수 폴딩** → 성능 문제 없음 |
| 보안 | `toString()`에 **민감정보** 포함 금지, 로그 레드액션 고려 |

---

## 확장 비교 표(심화)

| 항목 | String | StringBuilder | StringBuffer |
|---|---|---|---|
| 동등성 비교 | `equals` 내용 기준 | 동일 인스턴스 비교가 일반적(가변) | 동일 인스턴스 |
| 기본 해시 전략 | 캐시된 해시(불변이므로 안전) | 가변이라 보통 **키로 부적절** | 가변이라 부적절 |
| 메모리 사용 | 짧은 라틴 텍스트에서 효율적(Compact Strings) | capacity에 따라 여유 공간 존재 | capacity + 동기화 오버헤드 |
| API 유연성 | 풍부한 유틸(포맷, joiner 등) | 조작 중심 API(append/insert/replace) | Builder와 유사 |
| JIT 최적화 | 상수 폴딩/바이트코드 수준 Builder 변환 | 반복 조작 최적 | 메서드 동기화로 JIT 제약 가능 |

---

## 요약

- **String**: 불변. 안전/간결. **변경이 적은 텍스트**, Map 키, 메시지에 최적.
- **StringBuilder**: 가변. **단일 스레드에서 반복 조작**이 빠름. 루프에서 **무조건 사용**.
- **StringBuffer**: 가변 + 동기화. **공유 인스턴스**를 다중 스레드가 함께 다룰 때만.

**원칙**: *불변을 기본으로, 성능 병목 지점에서만 가변을 도입하되, 스레드 모델에 맞춰 선택*.

---

## 간단 실습: JSON 유사 문자열 빠르게 만들기

```java
public class Jsonish {
    public static String toJsonish(String key, String value, int id) {
        StringBuilder sb = new StringBuilder(key.length() + value.length() + 32);
        sb.append('{')
          .append("\"id\":").append(id).append(',')
          .append("\"").append(key).append("\":\"").append(value).append("\"")
          .append('}');
        return sb.toString();
    }

    public static void main(String[] args) {
        System.out.println(toJsonish("name", "kim", 7));
    }
}
```

---

## 코드포인트 안전 슬라이싱/대문자화(국제화 예시)

```java
// 첫 N 코드포인트를 upper-case로 변환 후 반환
static String headUpper(String s, int n, java.util.Locale loc) {
    int i = 0, cnt = 0;
    while (i < s.length() && cnt < n) {
        i += Character.charCount(s.codePointAt(i));
        cnt++;
    }
    String head = s.substring(0, i).toUpperCase(loc);
    return head + s.substring(i);
}
```

위와 같은 패턴으로 **이모지/합성 문자 손상 없는** 처리 가능.
