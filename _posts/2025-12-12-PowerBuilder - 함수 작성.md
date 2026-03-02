---
layout: post
title: PowerBuilder - 함수 작성
date: 2025-12-12 18:30:23 +0900
category: PowerBuilder
---
# 함수 작성 (전역 함수 / 로컬 함수)

PowerScript에서 함수는 특정 작업을 수행하고 결과를 반환하는 재사용 가능한 코드 블록입니다. 함수를 적절히 사용하면 코드 중복을 줄이고, 유지보수성을 높이며, 애플리케이션의 구조를 체계적으로 만들 수 있습니다. PowerBuilder에서는 함수의 범위(Scope)에 따라 **전역 함수**, **객체 수준 함수(로컬 함수)**로 구분하며, 객체 수준 함수는 다시 **객체 함수**와 **윈도우 함수** 등으로 세분화됩니다. 이 글에서는 함수의 개념부터 작성 방법, 호출 방식, 고급 기능까지 상세히 알아보겠습니다.

---

## 함수의 개념과 종류

### 함수란?

함수는 입력값(인자, Argument)을 받아 특정 로직을 수행한 후 결과값을 반환하는 서브프로그램입니다. PowerScript에서 함수는 다음과 같은 특징을 가집니다.

- **반환값**: 함수는 항상 하나의 값을 반환합니다. 반환값이 없는 함수는 `SUBROUTINE`이라고 별도로 구분하기도 하지만, PowerBuilder에서는 반환형을 지정하지 않으면 `(None)`으로 간주합니다.
- **재사용성**: 한 번 작성한 함수는 여러 곳에서 호출하여 사용할 수 있습니다.
- **모듈화**: 복잡한 로직을 기능 단위로 분리하여 코드 가독성과 유지보수성을 향상시킵니다.

### 함수의 종류

PowerBuilder에서 함수는 정의되는 위치에 따라 다음과 같이 분류됩니다.

| 함수 종류 | 정의 위치 | 호출 범위 | 주요 특징 |
|----------|----------|----------|----------|
| **전역 함수** | 별도의 Function Painter | 애플리케이션 전체 | 어디서나 호출 가능, 공통 유틸리티에 적합 |
| **객체 함수** | 특정 객체 내부(Window, Menu, User Object) | 해당 객체 및 상속 객체 | 객체의 동작을 캡슐화, 객체 지향적 설계 |
| **윈도우 함수** | 윈도우 내부 | 해당 윈도우 내부 | 윈도우 전용 로직 처리 |
| **메뉴 함수** | 메뉴 내부 | 해당 메뉴 내부 | 메뉴 관련 로직 처리 |
| **사용자 객체 함수** | 사용자 객체 내부 | 해당 객체 및 상속 객체 | 재사용 가능한 비즈니스 컴포넌트 |

---

## 전역 함수 (Global Function)

### 전역 함수의 개념

전역 함수는 애플리케이션 전체에서 접근 가능한 함수입니다. 주로 여러 곳에서 공통으로 사용되는 유틸리티 기능(문자열 처리, 수학 계산, 데이터 변환 등)을 구현할 때 사용합니다.

- 저장 위치: 별도의 `.pbl` 라이브러리에 저장
- 호출 범위: 모든 객체(윈도우, 메뉴, 사용자 객체 등)의 스크립트에서 직접 호출 가능
- 네이밍: 일반적으로 `f_` 접두사 사용 (예: `f_round`, `f_format_date`)

### 전역 함수 생성 방법

1. **Function Painter 열기**
   - 메뉴: **File → New**
   - **PB Object** 탭에서 **Function** 아이콘 선택
   - 또는 도구 모음에서 **New Function** 아이콘 클릭

2. **함수 속성 정의**
   Function Painter가 열리면 다음과 같은 정보를 입력합니다.

   ```
   [함수 속성]
   ┌─────────────────────────────────┐
   │ Function Name: f_calc_tax       │
   │ Access: Public                   │
   │ Returns: Decimal                 │
   │ Arguments:                       │
   │   - amount (Decimal) : 과세 금액 │
   │   - rate (Decimal)   : 세율      │
   └─────────────────────────────────┘
   ```

   - **Function Name**: 함수 이름 (고유해야 함)
   - **Access**: 접근 지정자 (Public/Private/Protected) - 전역 함수는 보통 Public
   - **Returns**: 반환 데이터 타입 (없으면 (None) 선택)
   - **Arguments**: 인자 목록 (이름, 타입, 전달 방식)

   > **📸 참고**: Function Painter 하단의 Arguments 입력 영역에서 **Pass By** 드롭다운을 통해 값 전달(value), 참조 전달(reference), 읽기 전용(readonly)을 선택할 수 있습니다. (이 부분은 스크린샷으로 첨부하면 초보자에게 큰 도움이 됩니다.)

3. **함수 본문 작성**
   하단의 스크립트 편집기에 함수 로직을 작성합니다.

   ```
   // f_calc_tax (세금 계산 함수)
   Decimal ld_tax
   
   ld_tax = amount * (rate / 100)
   
   RETURN ld_tax
   ```

4. **저장**
   함수 이름으로 저장됩니다. (예: `f_calc_tax`)

### 전역 함수 호출 예제

어느 객체의 스크립트에서든 함수 이름과 인자를 전달하여 호출합니다.

```
// 버튼 Clicked 이벤트에서 세금 계산
Decimal ld_price = 10000
Decimal ld_taxRate = 10
Decimal ld_tax

ld_tax = f_calc_tax(ld_price, ld_taxRate)   // 결과: 1000

MessageBox("세금", "부가세: " + String(ld_tax))
```

---

## 객체 함수 (Object-Level Function)

### 객체 함수의 개념

객체 함수는 특정 객체(윈도우, 메뉴, 사용자 객체 등)에 소속된 함수입니다. 객체 지향 개념에서 해당 객체의 행위(Behavior)를 정의하며, 객체의 데이터(인스턴스 변수)에 접근하여 작업을 수행할 수 있습니다.

- **윈도우 함수**: 특정 윈도우 내에서만 사용되는 함수 (예: `w_customer::uf_refresh_data`)
- **사용자 객체 함수**: 사용자 객체(Visual/Non-Visual)의 메서드 (예: `n_calculator::of_calculate`)
- **메뉴 함수**: 메뉴 아이템의 공통 로직 처리

### 객체 함수의 장점

- **캡슐화**: 객체의 내부 데이터를 보호하면서 필요한 기능만 외부에 노출
- **상속 지원**: 부모 객체의 함수를 자식 객체에서 재사용하거나 오버라이드 가능
- **네임스페이스 구분**: 함수 이름이 객체에 종속되므로 이름 충돌 방지

### 객체 함수 생성 방법

1. **객체 열기**
   - 해당 객체(윈도우, 사용자 객체 등)를 Painter로 엽니다.

2. **함수 선언 화면 열기**
   - **Declare → Window Functions** (윈도우의 경우)
   - 또는 **Insert → Function** 메뉴 선택
   - 또는 System Tree에서 객체 아래 **Functions** 노드를 오른쪽 클릭하고 **Add** 선택

3. **함수 속성 정의**
   전역 함수와 유사하지만, **Access** 설정에 차이가 있습니다.

   ```
   [윈도우 함수 속성 예시]
   ┌─────────────────────────────────┐
   │ Function Name: uf_refresh_data   │
   │ Access: Public                    │
   │ Returns: Integer                   │
   │ Arguments:                         │
   │   - customer_id (String) : 고객ID  │
   └─────────────────────────────────┘
   ```

   - **Access**: 
     - **Public**: 외부에서 객체명을 통해 호출 가능
     - **Private**: 해당 객체 내부에서만 호출 가능 (상속 객체에서도 접근 불가)
     - **Protected**: 해당 객체와 상속받은 객체에서만 호출 가능

4. **함수 본문 작성 및 저장**

### 객체 함수 호출 방법

객체 함수는 해당 객체의 인스턴스를 통해 호출합니다.

- **같은 객체 내부에서 호출**: 함수 이름만 사용
  ```
  // 같은 윈도우의 다른 이벤트에서
  uf_refresh_data("CUST001")
  ```

- **외부 객체에서 호출**: 객체 참조 변수 사용
  ```
  // 다른 윈도우에서 w_main 윈도우의 함수 호출
  w_main w_mywin
  w_mywin = w_main          // 열려 있는 윈도우 참조 가져오기
  w_mywin.uf_refresh_data("CUST001")
  ```

- **자기 자신의 함수 호출**: `THIS` 키워드 사용 가능 (선택 사항)
  ```
  THIS.uf_refresh_data("CUST001")
  ```

---

## 함수의 구성 요소

### 함수 이름 규칙

일반적으로 다음과 같은 접두사를 사용하여 함수의 종류와 범위를 구분합니다.

| 접두사 | 의미 | 예시 |
|--------|------|------|
| `f_` | 전역 함수 | `f_format_date` |
| `uf_` | 사용자 정의 함수 (User Function) - 객체 함수 | `uf_save_data` |
| `of_` | 객체 함수 (Object Function) - 주로 비시각적 객체 | `of_calculate` |
| `wf_` | 윈도우 함수 (Window Function) | `wf_open_child` |

### 인자(Argument) 정의

함수는 0개 이상의 인자를 가질 수 있습니다. 각 인자는 다음과 같이 정의합니다.

- **Name**: 인자 이름 (함수 내에서 사용)
- **Type**: 데이터 타입 (Integer, String, Decimal, Date 등)
- **Pass By**: 전달 방식
  - **value** (값 전달): 인자의 복사본이 전달되므로 함수 내에서 변경해도 원본 영향 없음
  - **reference** (참조 전달): 인자의 주소가 전달되므로 함수 내에서 변경하면 원본도 변경됨
  - **readonly**: 읽기 전용 참조 (값 변경 불가)

```
// 참조 전달 예시
// 함수 선언: f_swap (ref a Integer, ref b Integer)
// 함수 본문:
Integer li_temp
li_temp = a
a = b
b = li_temp

// 호출부:
Integer li_x = 10, li_y = 20
f_swap(li_x, li_y)   // li_x = 20, li_y = 10
```

> **💡 배열을 인자로 전달하기**  
> 배열을 함수의 인자로 넘길 때는 타입 뒤에 대괄호(`[]`)를 붙여야 합니다.
> ```
> // 함수 선언
> FUNCTION Integer f_sum_array (Integer ai_values[])
>    // 배열 처리
>
> // 호출
> Integer li_numbers[] = {1,2,3,4,5}
> Integer li_total = f_sum_array(li_numbers)
> ```

### 반환값(Return Value)

함수는 항상 하나의 값을 반환합니다. 반환값이 없는 경우 `Returns`를 `(None)`으로 설정합니다.

- `RETURN` 문을 사용하여 값을 반환합니다.
- 함수의 모든 실행 경로에서 반환값을 지정해야 합니다.

```
// 반환값이 있는 함수
Integer li_result
li_result = amount * 2
RETURN li_result

// 반환값이 없는 함수 (서브루틴)
// 함수 선언 시 Returns: (None)
// 본문에서:
MessageBox("알림", "작업 완료")
RETURN   // 값 없이 RETURN만 사용 가능
```

### 지역 변수(Local Variable)

함수 내부에서만 사용되는 변수입니다. 함수가 호출될 때 생성되고 종료되면 소멸됩니다.

```
Integer li_count      // 지역 변수 선언
String ls_temp
li_count = 10
```

---

## 함수 오버로드 (Function Overloading)

PowerBuilder는 **함수 오버로드**를 지원합니다. 같은 이름의 함수를 인자의 개수나 데이터 타입을 다르게 하여 여러 개 정의할 수 있습니다.

### 오버로드 규칙

- 함수 이름은 동일해야 함
- 인자의 개수 또는 데이터 타입이 달라야 함
- 반환 타입만 다른 것은 오버로드로 인정되지 않음

### 예제: 문자열을 다양한 방식으로 포맷팅하는 함수

```
// 전역 함수 f_format
// 버전 1: 날짜 포맷
FUNCTION String f_format (Date ad_date) 
   RETURN String(ad_date, "yyyy-mm-dd")

// 버전 2: 숫자 포맷 (천 단위 구분)
FUNCTION String f_format (Decimal ad_number)
   RETURN String(ad_number, "#,##0")

// 버전 3: 문자열 포맷 (지정된 길이로 자르기)
FUNCTION String f_format (String as_text, Integer ai_length)
   IF Len(as_text) > ai_length THEN
      RETURN Left(as_text, ai_length) + "..."
   ELSE
      RETURN as_text
   END IF
```

### 호출 예제

PowerBuilder는 전달된 인자의 타입에 따라 적절한 함수를 자동으로 선택합니다.

```
String ls_result
Date ld_today = Today()
Decimal ld_price = 1234567.89

ls_result = f_format(ld_today)        // "2025-03-15"
ls_result = f_format(ld_price)         // "1,234,568" (반올림)
ls_result = f_format("Hello World", 5) // "Hello..."
```

---

## 함수의 상속과 오버라이드

### 상속 관계에서의 함수

객체 함수는 상속 관계에서 다음과 같이 동작합니다.

- **상속**: 자식 객체는 부모 객체의 Public/Protected 함수를 상속받습니다.
- **오버라이드(Override)**: 자식 객체에서 부모와 동일한 이름과 인자 목록을 가진 함수를 다시 정의할 수 있습니다.

### 함수 오버라이드 예제 (비시각적 객체 사용)

**부모 사용자 객체 (n_parent)**
```
// 함수 선언: of_display_message (Public)
// 본문:
MessageBox("정보", "부모 객체 메시지")
```

**자식 사용자 객체 (n_child, n_parent 상속)**
```
// 동일한 이름으로 함수 재정의
// 본문:
MessageBox("정보", "자식 객체 메시지")

// 부모 함수 호출 (CALL 사용 안 함!)
Super::of_display_message()  
// 또는 n_parent::of_display_message()
```

**호출 결과**
```
n_child lnv_child
lnv_child = CREATE n_child
lnv_child.of_display_message()   // "자식 객체 메시지" 출력 후 부모 함수 실행
DESTROY lnv_child
```

> **⚠️ 주의**: 윈도우 같은 시각적 객체는 `CREATE`로 생성하지 않습니다. 시각적 객체는 반드시 `Open()` 함수를 사용하여 인스턴스를 생성하고 화면에 표시해야 합니다. 위 예제는 비시각적 객체(NVO)를 사용한 올바른 예시입니다.

### 부모 함수 호출 문법

자식 함수 내에서 부모 클래스의 원본 함수를 호출할 때는 **`Super::`** 또는 **`부모클래스명::`** 을 사용합니다. (`CALL` 키워드는 이벤트 호출에만 사용됩니다.)

```
// 자식 함수 내에서
Super::부모함수이름(인자)
// 또는
부모클래스명::부모함수이름(인자)
```

---

## 함수 작성 모범 사례

### 1. 함수는 단일 책임 원칙을 따르라

하나의 함수는 하나의 작업만 수행해야 합니다. 예를 들어, 데이터를 가져오고 화면에 표시하는 작업은 별도의 함수로 분리하는 것이 좋습니다.

```
// 좋은 예
uf_fetch_data()    // 데이터 조회만 수행
uf_display_data()  // 화면 표시만 수행

// 나쁜 예
uf_fetch_and_display()  // 두 가지 작업을 동시에 수행
```

### 2. 함수 이름은 동사를 사용하여 명확하게

함수 이름은 기능을 명확히 표현해야 합니다.

- `get_customer_name` (고객명 조회)
- `calculate_total_amount` (총액 계산)
- `validate_input_data` (입력 데이터 검증)

### 3. 인자 개수는 적게 유지

인자가 3~4개를 넘어가면 구조체(Structure)를 사용하는 것을 고려하세요.

```
// 대신 Structure 사용
s_customer_info
   - id (String)
   - name (String)
   - address (String)
   - phone (String)

FUNCTION Boolean f_save_customer (s_customer_info cust_info)
```

### 4. 오류 처리 포함

함수 내에서 발생할 수 있는 오류를 적절히 처리하고, 성공/실패 여부를 반환값으로 전달합니다.

```
FUNCTION Boolean f_delete_customer (String as_id)
   // 고객 삭제 로직
   IF SQLCA.SQLCode <> 0 THEN
      MessageBox("DB 오류", SQLCA.SQLErrText)
      RETURN FALSE
   END IF
   RETURN TRUE
```

### 5. 함수 주석 작성

함수의 목적, 인자 설명, 반환값, 주의사항 등을 주석으로 남겨둡니다.

```
/*******************************************************************
 * 함수명: f_calc_tax
 * 설명:  과세 금액과 세율을 입력받아 세금 계산
 * 인자:  amount - 과세 금액 (Decimal)
 *        rate   - 세율(%) (Decimal)
 * 반환:  계산된 세금 (Decimal)
 * 주의:  rate는 0~100 사이 값이어야 함
 *******************************************************************/
```

---

## 실전 예제: 종합 활용

### 예제 1: 전역 함수를 활용한 데이터 유효성 검사

**전역 함수: f_is_valid_email**
```
/*******************************************************************
 * 함수명: f_is_valid_email
 * 설명:  이메일 주소 형식 검증 (간단한 버전)
 * 인자:  as_email - 검증할 이메일 문자열
 * 반환:  유효하면 TRUE, 그렇지 않으면 FALSE
 *******************************************************************/
String ls_email
Integer li_at_pos, li_dot_pos

ls_email = Trim(as_email)

// @ 기호 확인
li_at_pos = Pos(ls_email, "@")
IF li_at_pos = 0 THEN RETURN FALSE

// @ 뒤에 . 기호 확인
li_dot_pos = Pos(ls_email, ".", li_at_pos + 1)
IF li_dot_pos = 0 THEN RETURN FALSE

// 공백 확인
IF Pos(ls_email, " ") > 0 THEN RETURN FALSE

RETURN TRUE
```

**호출 예제 (윈도우에서)**
```
String ls_input
ls_input = sle_email.Text

IF NOT f_is_valid_email(ls_input) THEN
   MessageBox("오류", "올바른 이메일 형식이 아닙니다.")
   RETURN
END IF
```

### 예제 2: 윈도우 함수로 화면 데이터 처리

**윈도우 w_customer의 함수: uf_load_customer**
```
/*******************************************************************
 * 함수명: uf_load_customer
 * 설명:  지정된 고객ID로 고객 정보를 조회하여 화면에 표시
 * 인자:  as_cust_id - 조회할 고객 ID
 * 반환:  성공 시 1, 실패 시 -1, 데이터 없음 0
 *******************************************************************/
Integer li_rtn

// DataWindow(dw_customer)에 고객 데이터 조회
dw_customer.SetTransObject(SQLCA)
li_rtn = dw_customer.Retrieve(as_cust_id)

IF li_rtn < 0 THEN
   MessageBox("오류", "데이터 조회 실패")
   RETURN -1
ELSEIF li_rtn = 0 THEN
   MessageBox("알림", "해당 고객이 없습니다.")
   RETURN 0
END IF

// 화면 컨트롤에 데이터 표시 (추가 처리)
sle_name.Text = dw_customer.GetItemString(1, "cust_name")
sle_phone.Text = dw_customer.GetItemString(1, "phone")

RETURN 1
```

**호출 예제 (조회 버튼 Clicked 이벤트)**
```
String ls_search_id
ls_search_id = sle_search.Text

IF uf_load_customer(ls_search_id) = 1 THEN
   // 성공 시 추가 작업
   cb_modify.Enabled = TRUE
ELSE
   cb_modify.Enabled = FALSE
END IF
```

---

## 함수 관련 디버깅 팁

### 1. 함수 호출 스택 확인
디버그 모드에서 **Call Stack** 창을 열면 현재까지 호출된 함수의 순서를 확인할 수 있습니다. 이를 통해 어떤 함수가 어떤 함수를 호출했는지 추적할 수 있습니다.

### 2. 함수 내 변수 감시
디버거에서 **Watch** 창에 함수의 지역 변수나 인자를 등록하여 값의 변화를 실시간으로 관찰할 수 있습니다.

### 3. 중단점 설정
함수의 특정 지점에 중단점을 설정하여 실행을 일시 중지하고 변수 값을 검사할 수 있습니다.

### 4. 함수 호출 시그니처 확인
컴파일 오류 시 "Function not found" 메시지가 나타나면 함수 이름과 인자 타입이 일치하는지 확인합니다. 오버로드된 함수의 경우 인자 타입이 모호하지 않은지 검토합니다.

---

## 마치며

PowerScript에서 함수는 코드 재사용성과 모듈화의 핵심 요소입니다. 전역 함수는 공통 유틸리티를, 객체 함수는 특정 객체의 행위를 캡슐화하여 객체 지향적인 설계를 가능하게 합니다.

함수를 작성할 때는 명확한 이름, 적절한 인자 개수, 오류 처리, 주석 등을 고려하여 가독성과 유지보수성을 높이는 것이 중요합니다. 또한 상속과 오버로드 기능을 활용하면 더 유연하고 강력한 애플리케이션을 구축할 수 있습니다.

특히 주의할 점은 **부모 함수 호출 시 `CALL` 키워드를 사용하지 않는다**는 것과 **시각적 객체(윈도우)는 `CREATE`가 아닌 `Open()`으로 생성한다**는 점입니다. 이러한 세부 문법을 정확히 이해해야 안정적인 애플리케이션 개발이 가능합니다.

다음 글에서는 PowerBuilder의 가장 강력한 기능인 DataWindow에 대해 더 깊이 있게 다루어 보겠습니다. DataWindow를 사용하면 데이터베이스 연동과 화면 표시를 획기적으로 단순화할 수 있습니다.