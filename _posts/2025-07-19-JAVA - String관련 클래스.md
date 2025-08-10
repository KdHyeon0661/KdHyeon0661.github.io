---
layout: post
title: Java - String관련 클래스
date: 2025-07-19 22:20:23 +0900
category: Java
---
# String, StringBuilder, StringBuffer

Java에서 문자열을 다루는 클래스는 크게 `String`, `StringBuilder`, `StringBuffer`가 있습니다. 이들은 모두 문자열을 표현하지만, 내부 구현 방식과 성능 특성, 쓰레드 안정성(Thread Safety) 측면에서 차이가 있습니다. 각각을 자세히 살펴보겠습니다.

---

## 1. `String`

### 특징
- `String`은 **불변(immutable)** 객체입니다.  
  한 번 생성된 문자열은 변경할 수 없습니다.
- 문자열을 수정하는 것처럼 보여도, 실제로는 새로운 `String` 객체가 생성됩니다.

### 예제
```java
String str = "Hello";
str = str + " World";
System.out.println(str); // "Hello World"
```

위 코드는 실제로 `"Hello"`라는 문자열 객체를 만들고, `" World"`를 추가하면서 **새로운 문자열 객체**를 생성하여 `str`이 가리키게 됩니다.

### 장점
- 불변이므로 **보안, 동기화, 캐싱에 유리**합니다.
- **상수풀(String pool)**을 이용하여 메모리를 효율적으로 사용할 수 있음.

### 단점
- 문자열을 자주 수정해야 하는 상황에서는 비효율적 (매번 객체 생성)

---

## 2. `StringBuilder`

### 특징
- **가변(mutable)** 객체입니다. 문자열을 수정할 수 있습니다.
- **싱글 스레드 환경에서 빠름** (동기화를 지원하지 않음)
- Java 5부터 도입되었습니다.

### 예제
```java
StringBuilder sb = new StringBuilder("Hello");
sb.append(" World");
System.out.println(sb.toString()); // "Hello World"
```

### 주요 메서드
```java
append(String str)   // 문자열 덧붙이기
insert(int offset, String str)  // 문자열 삽입
delete(int start, int end)  // 문자열 삭제
reverse()   // 문자열 뒤집기
```

### 장점
- 문자열을 자주 수정해야 하는 상황에 적합
- 성능이 좋음 (String보다 빠름)

---

## 3. `StringBuffer`

### 특징
- `StringBuilder`와 거의 동일한 기능 제공
- 차이점: **스레드 안전(thread-safe)**  
  내부 메서드에 `synchronized` 키워드를 사용하여 **멀티스레드 환경**에서도 안전하게 동작
- Java 1.0부터 존재

### 예제
```java
StringBuffer sbf = new StringBuffer("Hello");
sbf.append(" World");
System.out.println(sbf.toString()); // "Hello World"
```

### 장점
- 멀티스레드 환경에서 안전
- 문자열 수정이 빈번할 경우 유리

### 단점
- 동기화 때문에 `StringBuilder`보다 성능이 낮음

---

## 4. 비교 요약

| 특징              | String      | StringBuilder     | StringBuffer      |
|-------------------|-------------|-------------------|-------------------|
| 변경 가능 여부     | 불변         | 가변              | 가변              |
| 스레드 안전        | X           | X                 | O                 |
| 성능               | 느림         | 빠름              | 중간              |
| 사용 환경          | 일반 문자열 | 단일 스레드       | 멀티 스레드       |

---

## 5. 어떤 것을 언제 사용해야 할까?

- **문자열이 불변이어야 하고, 자주 바뀌지 않는 경우** → `String`
- **문자열을 반복적으로 조작(추가/삭제 등)하는 경우** → `StringBuilder`
- **멀티스레드 환경에서 문자열을 조작해야 하는 경우** → `StringBuffer`

---

## 6. 성능 비교 예시

```java
public class StringTest {
    public static void main(String[] args) {
        int n = 100000;

        // String
        long start = System.currentTimeMillis();
        String str = "";
        for (int i = 0; i < n; i++) {
            str += i;
        }
        long end = System.currentTimeMillis();
        System.out.println("String: " + (end - start) + "ms");

        // StringBuilder
        start = System.currentTimeMillis();
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < n; i++) {
            sb.append(i);
        }
        end = System.currentTimeMillis();
        System.out.println("StringBuilder: " + (end - start) + "ms");

        // StringBuffer
        start = System.currentTimeMillis();
        StringBuffer sbf = new StringBuffer();
        for (int i = 0; i < n; i++) {
            sbf.append(i);
        }
        end = System.currentTimeMillis();
        System.out.println("StringBuffer: " + (end - start) + "ms");
    }
}
```

결과적으로 `StringBuilder`가 가장 빠르고, `StringBuffer`는 그 다음, `String`은 매우 느리게 동작합니다.

---

## 결론

Java에서 문자열 처리는 매우 중요합니다. 아래처럼 정리할 수 있습니다.

- **변경이 적은 문자열** → `String`
- **단일 스레드에서 자주 수정** → `StringBuilder`
- **멀티 스레드 환경에서 자주 수정** → `StringBuffer`

각 클래스의 특성을 잘 파악하고 적절한 상황에서 사용하는 것이 성능과 안정성을 높이는 핵심입니다.