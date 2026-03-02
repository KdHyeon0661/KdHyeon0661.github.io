---
layout: post
title: PowerBuilder - 기본 문법
date: 2025-12-12 17:30:23 +0900
category: PowerBuilder
---
# 기본 문법 (PowerScript)

PowerBuilder 애플리케이션의 핵심 로직은 PowerScript라는 전용 스크립트 언어로 작성됩니다. PowerScript는 4GL 언어로서 직관적이고 강력한 기능을 제공하며, 데이터베이스 처리, 화면 제어, 비즈니스 로직 구현 등을 효율적으로 수행할 수 있도록 설계되었습니다. 이 글에서는 PowerScript의 기본 문법을 변수 선언, 데이터 타입, 연산자, 조건문, 반복문으로 나누어 자세히 살펴보겠습니다.

---

## 변수 선언 및 데이터 타입

### 변수 선언

PowerScript에서 변수를 사용하려면 먼저 선언해야 합니다. 변수 선언은 일반적으로 다음과 같은 형식을 따릅니다.

```
데이터타입 변수이름 [= 초기값]
```

여러 변수를 한 줄에 선언할 수도 있습니다.

```
Integer li_count, li_total
String ls_name, ls_address
```

#### 변수의 스코프(Scope)

변수는 선언되는 위치와 방식에 따라 스코프(유효 범위)가 결정됩니다.

- **지역 변수 (Local Variable)**: 스크립트나 함수 내부에서 선언되며, 해당 블록 내에서만 사용 가능합니다.
  ```
  // 버튼 Clicked 이벤트 내부
  Integer li_local
  li_local = 10
  ```

- **인스턴스 변수 (Instance Variable)**: 객체(윈도우, 사용자 객체 등)의 정의 부분에 선언되며, 해당 객체의 모든 스크립트에서 사용할 수 있습니다. 객체마다 별도의 값을 가집니다.
  ```
  // 윈도우의 인스턴스 변수 선언 영역
  Integer ii_count
  String is_name
  ```

- **공유 변수 (Shared Variable)**: 객체의 모든 인스턴스가 공유하는 변수입니다. 같은 객체 타입의 여러 인스턴스가 하나의 변수를 공유합니다.
  ```
  // 윈도우의 공유 변수 선언 영역
  SHARED Integer si_sharedCount
  ```

- **전역 변수 (Global Variable)**: 애플리케이션 전체에서 접근 가능한 변수입니다. 모든 객체의 스크립트에서 사용할 수 있습니다.
  ```
  // 전역 변수 선언 화면 (Application Object 또는 별도 전역 변수 선언)
  GLOBAL String gs_userID
  ```

### 데이터 타입

PowerScript는 다양한 데이터 타입을 제공합니다. 주요 데이터 타입은 다음과 같습니다.

| 데이터 타입 | 설명 | 예시 |
|------------|------|------|
| **Integer** | 16비트 부호 있는 정수 (-32768 ~ 32767) | `Integer li_val = 100` |
| **Long** | 32비트 부호 있는 정수 (-2,147,483,648 ~ 2,147,483,647) | `Long ll_large = 999999` |
| **Real** | 6자리 정밀도의 부동소수점 | `Real lr_rate = 1.2345` |
| **Double** | 15자리 정밀도의 부동소수점 | `Double ldbl_value = 3.14159265358979` |
| **Decimal** | 최대 18자리의 십진 고정소수점 (금융 계산에 유용) | `Decimal{2} ldc_price = 19.95` |
| **Boolean** | True 또는 False 값 | `Boolean lb_flag = TRUE` |
| **String** | 가변 길이 문자열 (최대 2,147,483,647자) | `String ls_name = "홍길동"` |
| **Char** | 단일 문자 | `Char lc_grade = 'A'` |
| **Date** | 날짜 (년-월-일) | `Date ld_today = 2025-03-15` |
| **Time** | 시간 (시:분:초.밀리초) | `Time lt_now = 14:30:25.123` |
| **DateTime** | 날짜와 시간의 조합 | `DateTime ldt_created` |
| **Blob** | Binary Large Object (이미지, 파일 등) | `Blob lb_image` |
| **Any** | 모든 타입의 데이터를 저장할 수 있는 특수 타입 | `Any la_data` |

#### Decimal 타입 선언 시 정밀도 지정
```
Decimal{2} ld_price     // 소수점 이하 2자리까지
Decimal{4} ld_rate      // 소수점 이하 4자리까지
```

### 배열(Array)

동일한 데이터 타입의 여러 값을 저장할 수 있는 자료 구조입니다.

- **고정 크기 배열**:
  ```
  Integer li_numbers[10]           // 10개 요소
  String ls_names[5] = {"Kim", "Lee", "Park", "Choi", "Jung"}
  ```

- **가변 크기 배열**:
  ```
  Integer li_scores[]               // 빈 배열
  li_scores[1] = 90                 // 첫 번째 요소 할당 (크기 자동 조정)
  li_scores[5] = 100                // 중간 빈 요소는 기본값(0)으로 채워짐
  ```

- **2차원 배열**:
  ```
  Integer li_matrix[10, 5]
  Integer li_table[][]
  ```

### 상수(Constant)

값이 변하지 않는 변수를 선언할 수 있습니다. `CONSTANT` 키워드를 사용합니다.

```
CONSTANT String GS_COMPANY_NAME = "한국전력"
CONSTANT Real PI = 3.14159
```

### 변수 초기화

변수 선언 시 초기값을 지정할 수 있습니다. 초기화하지 않으면 데이터 타입에 따라 기본값이 할당됩니다.

- 숫자형: 0
- Boolean: FALSE
- String: "" (빈 문자열)
- Date: 1900-01-01
- DateTime: 1900-01-01 00:00:00
- Blob: 빈 Blob

```
Integer li_count = 0
String ls_name = "Guest"
Boolean lb_active = TRUE
Date ld_birth = 1990-05-20
```

---

## 연산자

PowerScript에서 사용되는 주요 연산자들을 살펴보겠습니다.

### 산술 연산자

| 연산자 | 의미 | 예시 |
|--------|------|------|
| `+` | 덧셈 | `result = a + b` |
| `-` | 뺄셈 | `result = a - b` |
| `*` | 곱셈 | `result = a * b` |
| `/` | 나눗셈 | `result = a / b` (실수 나눗셈) |
| `^` | 거듭제곱 | `result = a ^ b` |
| `MOD` | 나머지 | `remainder = 10 MOD 3` (결과: 1) |

### 비교 연산자

| 연산자 | 의미 | 예시 |
|--------|------|------|
| `=` | 같음 | `IF a = b THEN` |
| `<>` | 다름 | `IF a <> b THEN` |
| `<` | 작음 | `IF a < b THEN` |
| `>` | 큼 | `IF a > b THEN` |
| `<=` | 작거나 같음 | `IF a <= b THEN` |
| `>=` | 크거나 같음 | `IF a >= b THEN` |

### 논리 연산자

| 연산자 | 의미 | 예시 |
|--------|------|------|
| `AND` | 논리곱 | `IF a > 0 AND b > 0 THEN` |
| `OR` | 논리합 | `IF a = 1 OR b = 1 THEN` |
| `NOT` | 논리 부정 | `IF NOT (a = b) THEN` |

### 문자열 연산자

- **연결(Concatenation)**: `+` 연산자를 사용하여 문자열을 연결합니다.
  ```
  String ls_first = "Hello"
  String ls_last = "World"
  String ls_message = ls_first + " " + ls_last   // 결과: "Hello World"
  ```

> **💡 실무 팁: 문자열 결합 시 Null 주의보!**  
> 파워빌더에서 `+` 연산자로 문자열을 합칠 때, 합치는 변수 중 단 하나라도 `NULL` 값을 가지고 있다면 전체 결과가 `NULL`이 되어버립니다.
> ```powerbuilder
> String ls_name = "홍길동"
> String ls_title = SetNull(ls_title) // Null 값 할당
> String ls_result
> ls_result = ls_name + " 님 " + ls_title
> // 결과: "홍길동 님 "이 아니라 NULL이 됨!
> ```
> 데이터베이스에서 읽어온 값이 `NULL`일 수 있다면, 반드시 `IsNull()` 함수로 먼저 체크하거나 `NullToEmpty()` 함수를 사용하여 안전하게 처리해야 합니다.
> ```powerbuilder
> IF IsNull(ls_title) THEN
>     ls_title = ""
> END IF
> ls_result = ls_name + " 님 " + ls_title   // 안전한 결합
> ```

### 기타 연산자

- **대입 연산자**: `=` (예: `a = b + c`)
- **증감 연산자**: PowerScript에서는 C나 Java처럼 `++`, `--` 증감 연산자를 지원합니다. (PowerBuilder 8.0부터 도입)
  ```
  li_count++        // li_count = li_count + 1
  li_count--        // li_count = li_count - 1
  ```
  또한 확장 대입 연산자(`+=`, `-=`, `*=`, `/=`, `^=` 등)도 사용할 수 있습니다.
  ```
  li_count += 5     // li_count = li_count + 5
  li_total *= 2     // li_total = li_total * 2
  ```

---

## 조건문

조건에 따라 실행 흐름을 분기하는 데 사용됩니다.

### IF ... THEN ... ELSE

가장 기본적인 조건문입니다.

**형식 1: 단일 조건**
```
IF 조건 THEN
    // 조건이 참일 때 실행
END IF
```

**형식 2: IF-ELSE**
```
IF 조건 THEN
    // 조건이 참일 때 실행
ELSE
    // 조건이 거짓일 때 실행
END IF
```

**형식 3: ELSEIF (다중 조건)**
```
IF 조건1 THEN
    // 조건1 참일 때 실행
ELSEIF 조건2 THEN
    // 조건2 참일 때 실행
ELSE
    // 모든 조건이 거짓일 때 실행
END IF
```

**예제**:
```
Integer li_score = 85
String ls_grade

IF li_score >= 90 THEN
    ls_grade = "A"
ELSEIF li_score >= 80 THEN
    ls_grade = "B"
ELSEIF li_score >= 70 THEN
    ls_grade = "C"
ELSEIF li_score >= 60 THEN
    ls_grade = "D"
ELSE
    ls_grade = "F"
END IF

MessageBox("학점", "당신의 학점은 " + ls_grade + "입니다.")
```

### CHOOSE CASE

여러 값 중 하나를 선택해야 할 때 사용하며, `IF ... ELSEIF`보다 가독성이 좋습니다.

**형식**:
```
CHOOSE CASE 표현식
    CASE 값1
        // 실행문
    CASE 값2
        // 실행문
    CASE 값3, 값4       // 여러 값 (OR 조건)
        // 실행문
    CASE IS > 10        // 비교 조건 (IS는 표현식의 값을 나타냄)
        // 실행문
    CASE 5 TO 10        // 범위 지정 (5 ~ 10)
        // 실행문
    CASE ELSE
        // 위 조건에 모두 해당하지 않을 때
END CHOOSE
```

**예제 1: 단일 값 비교**
```
String ls_day = "MON"
CHOOSE CASE ls_day
    CASE "MON"
        MessageBox("요일", "월요일입니다.")
    CASE "TUE"
        MessageBox("요일", "화요일입니다.")
    CASE "WED"
        MessageBox("요일", "수요일입니다.")
    CASE ELSE
        MessageBox("요일", "기타 요일입니다.")
END CHOOSE
```

**예제 2: 범위와 조건**
```
Integer li_month = 7
String ls_season

CHOOSE CASE li_month
    CASE 3, 4, 5
        ls_season = "봄"
    CASE 6, 7, 8
        ls_season = "여름"
    CASE 9, 10, 11
        ls_season = "가을"
    CASE 12, 1, 2
        ls_season = "겨울"
    CASE ELSE
        ls_season = "알 수 없음"
END CHOOSE

MessageBox("계절", ls_season)
```

**예제 3: IS와 TO 사용**
```
Integer li_score = 85
String ls_result

CHOOSE CASE li_score
    CASE IS >= 90
        ls_result = "우수"
    CASE 70 TO 89
        ls_result = "보통"
    CASE IS < 70
        ls_result = "미흡"
END CHOOSE
```

---

## 반복문

동일한 코드 블록을 여러 번 실행해야 할 때 사용합니다.

### FOR ... NEXT

지정된 횟수만큼 반복 실행합니다.

**형식**:
```
FOR 카운터변수 = 시작값 TO 종료값 [STEP 증가값]
    // 반복 실행할 코드
NEXT
```

- `STEP`을 생략하면 기본값 1로 증가합니다.
- 감소하는 반복문을 만들려면 `STEP`에 음수를 사용합니다.

**예제 1: 1부터 10까지 합계**
```
Integer li_i, li_sum = 0
FOR li_i = 1 TO 10
    li_sum += li_i   // li_sum = li_sum + li_i
NEXT
MessageBox("결과", "1~10 합계: " + String(li_sum))
```

**예제 2: STEP 2로 홀수만 출력**
```
Integer li_i
FOR li_i = 1 TO 10 STEP 2
    // li_i = 1, 3, 5, 7, 9
NEXT
```

**예제 3: 감소하는 반복문**
```
Integer li_i
FOR li_i = 10 TO 1 STEP -1    // STEP에 음수를 주어 감소
    // 10, 9, 8, ... , 1
NEXT
```

### DO ... LOOP

조건에 따라 반복 실행합니다. 다양한 변형이 있습니다.

#### DO WHILE ... LOOP
조건이 참인 동안 반복합니다. 조건이 거짓이면 루프에 진입하지 않습니다.

```
DO WHILE 조건
    // 조건이 참일 때 반복
LOOP
```

#### DO UNTIL ... LOOP
조건이 참이 될 때까지 반복합니다. 즉, 조건이 거짓인 동안 반복합니다.

```
DO UNTIL 조건
    // 조건이 거짓일 때 반복
LOOP
```

#### DO ... LOOP WHILE
일단 한 번 실행한 후, 조건이 참이면 반복을 계속합니다. 최소 한 번은 실행됩니다.

```
DO
    // 최소 한 번 실행
LOOP WHILE 조건
```

#### DO ... LOOP UNTIL
일단 한 번 실행한 후, 조건이 참이 될 때까지 반복합니다. 최소 한 번 실행됩니다.

```
DO
    // 최소 한 번 실행
LOOP UNTIL 조건
```

**예제 1: DO WHILE (1부터 10까지 합계)**
```
Integer li_i = 1, li_sum = 0
DO WHILE li_i <= 10
    li_sum += li_i
    li_i++                // 증감 연산자 사용 가능
LOOP
MessageBox("합계", String(li_sum))   // 55
```

**예제 2: DO UNTIL (li_i가 10보다 클 때까지 반복)**
```
Integer li_i = 1, li_sum = 0
DO UNTIL li_i > 10
    li_sum += li_i
    li_i++
LOOP
```

**예제 3: 최소 한 번 실행 보장**
```
Integer li_i = 100
DO
    li_i++                // 101이 됨
LOOP WHILE li_i < 10       // 조건이 거짓이므로 바로 종료 (최소 한 번 실행)
// 결과: li_i = 101
```

### EXIT와 CONTINUE

- **EXIT**: 반복문을 즉시 종료하고 빠져나옵니다.
- **CONTINUE**: 현재 반복의 나머지 부분을 건너뛰고 다음 반복으로 넘어갑니다.

**예제: 짝수만 더하기 (CONTINUE 사용)**
```
Integer li_i, li_sum = 0
FOR li_i = 1 TO 10
    IF MOD(li_i, 2) <> 0 THEN    // 홀수이면
        CONTINUE                  // 아래 코드 건너뛰고 다음 li_i로
    END IF
    li_sum += li_i                // 짝수만 더함
NEXT
MessageBox("짝수 합", String(li_sum))   // 2+4+6+8+10 = 30
```

**예제: 5가 되면 반복 종료 (EXIT 사용)**
```
Integer li_i
FOR li_i = 1 TO 10
    IF li_i = 5 THEN EXIT
    MessageBox("숫자", String(li_i))   // 1,2,3,4 만 출력
NEXT
```

---

## 실전 예제: 종합 활용

간단한 프로그램을 작성하여 지금까지 배운 내용을 종합해 보겠습니다. 윈도우에 버튼을 하나 배치하고, 버튼 클릭 이벤트에서 다음 기능을 수행합니다.

- 사용자가 입력한 숫자(1~9)에 해당하는 구구단을 출력합니다.
- 단, 입력값이 없으면 2단을 기본으로 출력합니다.
- 출력 형식: "2 x 1 = 2" 형태로 리스트박스에 추가합니다.

```
// 버튼 cb_calc 의 Clicked 이벤트
Integer li_dan, li_i
String ls_input, ls_line

// 입력값 가져오기 (SingleLineEdit sle_dan 이라고 가정)
ls_input = Trim(sle_dan.Text)

IF ls_input = "" THEN
    li_dan = 2   // 기본값
ELSE
    li_dan = Integer(ls_input)
END IF

// 입력값 유효성 검사 (1~9 사이)
IF li_dan < 1 OR li_dan > 9 THEN
    MessageBox("오류", "1에서 9 사이의 숫자를 입력하세요.")
    RETURN
END IF

// 리스트박스 초기화 (lb_result)
lb_result.Reset()

// 구구단 계산 및 출력
FOR li_i = 1 TO 9
    ls_line = String(li_dan) + " x " + String(li_i) + " = " + String(li_dan * li_i)
    lb_result.AddItem(ls_line)
NEXT
```

---

## 마치며

이상으로 PowerScript의 기본 문법을 살펴보았습니다. 변수 선언부터 조건문, 반복문까지의 개념은 어떤 프로그래밍 언어에서도 핵심적인 부분이며, PowerScript 역시 이러한 기본기를 바탕으로 강력한 데이터베이스 처리 기능과 결합됩니다.