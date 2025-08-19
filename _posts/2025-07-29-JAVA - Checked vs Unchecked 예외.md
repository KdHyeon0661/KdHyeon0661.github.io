---
layout: post
title: Java - Checked vs Unchecked 예외
date: 2025-07-29 15:20:23 +0900
category: Java
---
# Checked vs Unchecked 예외(예외 타입) — Java에서의 차이와 활용법 (자세 정리)

Java에서 예외는 `Throwable`을 루트로 하는 계층 구조를 갖습니다. 그 중 실무에서 중요한 구분은 **Checked Exception**(체크 예외)과 **Unchecked Exception**(언체크 예외)입니다. 이 문서는 두 종류의 차이, 동작 방식, 코드 예시, 설계 권장사항과 실무 패턴(예외 전환, 예외 체이닝 등)을 모두 다룹니다.

---

## 1. 개념 정리 (요약)

- **Checked Exception**  
  - `Exception`을 상속하지만 `RuntimeException`을 상속하지 않는 예외들.  
  - 컴파일러가 **처리(try-catch)** 하거나 **선언(throws)** 하도록 강제함.  
  - 주로 **복구 가능한(혹은 호출자에게 처리 책임이 있는)** 조건을 표현.

- **Unchecked Exception**  
  - `RuntimeException`과 그 하위 클래스.  
  - 컴파일러가 처리나 선언을 요구하지 않음.  
  - 주로 **프로그래밍 오류(논리 버그, 잘못된 API 사용)** 를 나타냄.

- **Error** (`java.lang.Error`)  
  - JVM 수준의 치명적 오류(OutOfMemoryError 등). 일반적으로 잡지 않음.

---

## 2. 예외 계층(간단)

```
Throwable
 ├─ Error (X: 거의 잡지 않음)
 └─ Exception
     ├─ RuntimeException (Unchecked)
     │   ├─ NullPointerException, IllegalArgumentException, ...
     └─ (Checked 예외)
         ├─ IOException, SQLException, ClassNotFoundException, ...
```

---

## 3. 코드 예시

### 3.1 Checked 예외 (파일 읽기 — `IOException`)
```java
import java.io.*;

public class CheckedExample {
    public static void readFile(String path) throws IOException {
        BufferedReader br = new BufferedReader(new FileReader(path));
        try {
            System.out.println(br.readLine());
        } finally {
            br.close(); // 또는 try-with-resources 권장
        }
    }

    public static void main(String[] args) {
        try {
            readFile("not_exists.txt");
        } catch (IOException e) {
            System.err.println("파일 읽기 실패: " + e.getMessage());
            // 복구 로직 또는 사용자 안내
        }
    }
}
```

### 3.2 Unchecked 예외 (프로그래밍 오류)
```java
public class UncheckedExample {
    public static int divide(int a, int b) {
        return a / b; // b==0 이면 ArithmeticException (unchecked) 발생
    }

    public static void main(String[] args) {
        System.out.println(divide(10, 0)); // 예외를 잡지 않으면 런타임에 터짐
    }
}
```

### 3.3 사용자 정의 예외 예시
```java
// Checked
public class InsufficientFundsException extends Exception {
    public InsufficientFundsException(String msg) { super(msg); }
}

// Unchecked
public class InvalidStateException extends RuntimeException {
    public InvalidStateException(String msg) { super(msg); }
}
```

---

## 4. 처리(try-catch) vs 선언(throws)

- **Checked** 예외는 메서드가 해당 예외를 처리하지 않으면 `throws`로 선언해야 함:
  ```java
  void foo() throws IOException { ... }
  ```
- **Unchecked** 예외는 `throws` 없이도 메서드 내부에서 `throw` 가능하며, 호출자에서 잡을 필요 없음.

---

## 5. 언제 어떤 타입을 선택할까? (설계 지침)

> **전통적 권장사항(많은 문헌에 나오는 관례)**  
> - **Checked**: *호출자가 합리적으로 복구하거나 조치를 취할 수 있는 상황* (예: I/O 실패, 네트워크 장애, 파일 없음).  
> - **Unchecked**: *프로그래밍 실수로 간주되는 상황* (null 인자, 잘못된 인덱스, 불변 규약 위반 등) — 호출자가 예외를 "항상" 처리해야 할 필요는 없음.

**그러나 현실적 고려사항**
- Checked 예외를 남발하면 API 사용이 번거로워지고, 호출자들이 예외를 무의미하게 캡처해서 무시하는 코드(`catch(Exception e) {}`)가 늘어날 수 있음.
- 그래서 일부 라이브러리/프레임워크(특히 최근 추세)는 **주로 Unchecked** 예외를 사용하고, 필요시 문서화로 안내하는 접근을 택합니다.

**권장 실무 원칙**
1. API 설계 시 **복구 가능성**을 기준으로 결정한다. 호출자가 실질적인 복구(재시도, 다른 자원 사용 등)를 할 수 있으면 Checked.
2. 내부적/프레임워크 수준에서는 Unchecked를 선호하는 경우가 많음(개발 편의성).
3. 예외 타입을 명확하고 구체적으로 만든다(일반 `Exception` 금지).

---

## 6. 예외 전환(Checked → Unchecked) 패턴

라이브러리에서 Checked 예외를 내부적으로 처리하고 호출자에게 Unchecked로 던지고 싶을 때 사용:

```java
try {
    someIoOperation();
} catch (IOException e) {
    throw new RuntimeException("IO 실패: " + e.getMessage(), e); // 예외 래핑
}
```

**장점**: 호출자에게 `throws` 부담을 주지 않음.  
**주의**: 예외 원인을 보존하려면 항상 `cause`로 전달(예: `new RuntimeException(msg, e)`).

---

## 7. 예외 체이닝 (Chaining) — 좋은 관행
- 예외를 래핑할 때 원래 예외를 `cause`로 전달하면 디버깅이 쉬움:
```java
catch (SQLException e) {
    throw new DataAccessException("DB 오류", e); // DataAccessException은 Unchecked 또는 Checked
}
```
- 스택트레이스에 원래 원인이 남음.

---

## 8. 예외 처리의 모범 사례

- **무분별한 catch 금지**: `catch(Exception e)`로 무조건 잡고 무시하면 디버깅이 불가능.
- **로그와 재처리**: 예외를 잡으면 적절히 로깅하고, 필요하면 재시도/대체 경로를 제공.
- **try-with-resources 사용**: 자원 해제는 `try-with-resources`(Java 7+)로 안전하게.
- **API 문서화**: Checked 예외를 사용하는 경우 `@throws`로 문서화하고, 호출자가 어떻게 처리해야 할지 안내.
- **구체적 예외 사용**: `IllegalArgumentException` 같은 범용 예외 대신 더 구체적인 예외를 만드는 것이 유지보수에 유리.
- **예외를 흐름 제어로 사용하지 않음**: 예외는 예외 상황용이지 정상 로직의 흐름 제어용이 아님.

---

## 9. 예시: 라이브러리 설계 시의 결단 예

- 퍼블릭 API에서 파일 읽기 실패에 대해 호출자가 반드시 처리해야 한다고 판단하면 `IOException`(Checked)을 노출.
- 반면 프레임워크 내부에서 I/O 실패는 치명적이므로 `Unchecked`로 래핑하여 던질 수 있음.

---

## 10. 기타 팁 / 주의사항

- **Unchecked 예외에도 문서화가 필요**: throws 선언을 하지 않더라도 API 문서에 어떤 unchecked 예외가 던져질 수 있는지 명시하면 사용자가 안전하게 호출 가능.
- **성능**: 예외는 예외 상황 처리용이고, 빈번한 정상 흐름의 제어로 사용하면 성능·가독성에 악영향.
- **테스트**: 단위 테스트에서 예외 발생 및 전파 경로를 확인하는 테스트를 작성하라.

---

## 11. 비교표 요약

| 항목 | Checked | Unchecked (RuntimeException) |
|------|---------|-------------------------------|
| 상속 | `Exception` (단, `RuntimeException` 제외) | `RuntimeException` |
| 컴파일 검사 | 예 (처리 또는 선언 필요) | 아니오 |
| 사용 의도 | 호출자가 복구 가능(예정) | 프로그래밍 오류/비복구 상황 |
| 선언(throws) 필요 | 예 | 선택적 |
| 예시 | `IOException`, `SQLException` | `NullPointerException`, `IllegalArgumentException` |

---

## 12. 결론 (권장 요약)
- **Checked**: 호출자가 합리적으로 복구할 가능성이 있을 때 사용. (I/O, 네트워크, DB 등)  
- **Unchecked**: API 사용자의 실수나 프로그래밍 오류일 때 사용. (잘못된 인수, 내부 논리 오류)  
- 실무에서는 **균형**이 중요 — Checked 남발은 API 사용자 부담을 늘리므로, 라이브러리 설계 시 신중히 선택하고, 예외 전환과 체이닝을 통해 명확한 에러 흐름을 제공하라.
