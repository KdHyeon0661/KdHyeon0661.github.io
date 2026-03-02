---
layout: post
title: PowerBuilder - 기본 이벤트 처리
date: 2025-12-12 22:30:23 +0900
category: PowerBuilder
---
# 기본 이벤트 처리 (Open, Clicked 등) 및 메시지 박스 사용, 사용자 정의 객체(User Object)

PowerBuilder 애플리케이션은 사용자 동작이나 시스템 상태 변화에 반응하는 **이벤트 기반(Event-Driven)** 구조로 동작합니다. 각 객체(윈도우, 컨트롤)는 미리 정의된 이벤트들을 가지고 있으며, 개발자는 이 이벤트에 스크립트를 연결하여 원하는 기능을 구현합니다. 또한 사용자와의 상호작용에서 자주 사용되는 **메시지 박스**는 간단한 정보 표시나 확인 대화상자로 활용됩니다. 나아가 **사용자 정의 객체(User Object)**를 활용하면 반복되는 UI 요소나 비즈니스 로직을 모듈화하여 재사용성과 유지보수성을 높일 수 있습니다. 이 글에서는 이 세 가지 주제를 상세히 살펴보겠습니다.

---

## 1. 기본 이벤트 처리

### 이벤트(Event)란?

이벤트는 객체에서 발생하는 특정 사건을 의미합니다. 예를 들어, 사용자가 버튼을 클릭하면 해당 버튼의 `Clicked` 이벤트가 발생하고, 윈도우가 열리면 `Open` 이벤트가 발생합니다. 개발자는 각 이벤트에 PowerScript 코드를 작성하여 애플리케이션이 적절히 반응하도록 합니다.

### 주요 이벤트와 발생 시점

#### 윈도우(Window) 이벤트

| 이벤트 | 발생 시점 | 주요 용도 |
|--------|----------|----------|
| **Open** | 윈도우가 열릴 때 | 초기화 작업(데이터베이스 연결, DataWindow 데이터 조회, 컨트롤 초기값 설정) |
| **Close** | 윈도우가 닫힐 때 | 정리 작업(트랜잭션 해제, 임시 파일 삭제, 객체 파괴) |
| **CloseQuery** | 윈도우가 닫히기 직전에 발생 | 닫기 전 확인 메시지 표시, 닫기 취소 가능 |
| **Resize** | 윈도우 크기가 변경될 때 | 컨트롤 위치/크기 재조정 |
| **Timer** | 타이머 설정 시 일정 간격으로 발생 | 주기적 작업(실시간 갱신, 자동 저장) |
| **Activate / Deactivate** | 윈도우가 활성화/비활성화될 때 | 포커스 관련 처리 |

#### 컨트롤(Control) 이벤트

| 이벤트 | 발생 시점 | 주요 용도 |
|--------|----------|----------|
| **Clicked** | 컨트롤을 클릭했을 때 | 버튼 클릭 처리, 항목 선택 등 |
| **DoubleClicked** | 더블클릭했을 때 | 상세 보기, 편집 모드 진입 |
| **RightClicked** | 마우스 오른쪽 버튼 클릭 | 컨텍스트 메뉴 표시 |
| **Modified** | 컨트롤의 내용이 변경되고 포커스를 잃을 때 | 입력값 검증, 자동 저장 |
| **GetFocus** | 컨트롤이 포커스를 받을 때 | 도움말 표시, 초기화 |
| **LoseFocus** | 컨트롤이 포커스를 잃을 때 | 입력 완료 처리 |
| **Key** | 키보드 입력이 있을 때 | 단축키 처리, 입력 제한 |
| **SelectionChanged** | (리스트 계열) 선택 항목이 변경될 때 | 선택된 항목에 따른 화면 갱신 |

### 이벤트 스크립트 작성 위치

각 객체의 Painter에서 이벤트 목록을 열고 해당 이벤트를 더블클릭하면 스크립트 편집기가 열립니다. 여기에 코드를 작성하면 됩니다.

```
// w_main의 Open 이벤트
dw_customer.SetTransObject(SQLCA)
dw_customer.Retrieve()
```

### 이벤트 전달 방식: Trigger vs Post

이벤트를 프로그래밍 방식으로 발생시킬 때는 두 가지 방법이 있습니다.

#### TRIGGER EVENT
- **동기적(Synchronous)**으로 실행됩니다.
- 이벤트 핸들러가 완료될 때까지 현재 스크립트의 실행이 중단됩니다.
- 즉시 처리가 필요한 상황에 사용합니다.

```
// TRIGGER 예시: 즉시 실행
Parent.TriggerEvent("ue_refresh")
```

#### POST EVENT
- **비동기적(Asynchronous)**으로 실행됩니다.
- 현재 스크립트는 계속 실행되고, 이벤트는 나중에(현재 이벤트 핸들러 종료 후) 메시지 큐에 쌓여 처리됩니다.
- 시간이 오래 걸리는 작업이나 현재 작업에 영향을 주지 않으면서 추가 처리가 필요할 때 유용합니다.

```
// POST 예시: 메시지 큐에 넣고 바로 다음 줄 실행
Parent.PostEvent("ue_long_task")
```

### 이벤트 상속과 CALL 문

상속 관계에서 자식 객체는 부모 객체의 이벤트 스크립트를 상속받습니다. 자식 객체에서 동일한 이벤트에 스크립트를 작성하면 부모의 스크립트는 실행되지 않습니다(오버라이드). 만약 부모의 스크립트를 먼저 실행하고 싶다면 `CALL` 문을 사용합니다.

```
// 자식 윈도우의 Open 이벤트
CALL w_parent::Open   // 부모의 Open 이벤트 먼저 실행
// 이후 자식의 추가 코드 실행
```

### 실전 예제: 윈도우 닫기 확인

```
// w_main의 CloseQuery 이벤트
Integer li_rtn
li_rtn = MessageBox("종료 확인", "프로그램을 종료하시겠습니까?", Question!, YesNo!)
IF li_rtn = 2 THEN   // No 버튼 선택
   RETURN 1          // 1을 반환하면 닫기가 취소됨
END IF
```

---

## 2. 메시지 박스 사용

메시지 박스는 사용자에게 정보를 표시하거나 선택을 요구하는 간단한 대화상자입니다. PowerBuilder에서는 `MessageBox()` 함수를 사용합니다.

### MessageBox 함수 문법

```
MessageBox ( title, text {, icon {, button {, default } } } )
```

| 인자 | 설명 | 필수 여부 |
|------|------|----------|
| `title` | 메시지 박스의 제목 표시줄 문자열 | 필수 |
| `text` | 본문에 표시할 메시지 문자열 | 필수 |
| `icon` | 표시할 아이콘 종류 | 선택 |
| `button` | 표시할 버튼 종류 | 선택 |
| `default` | 기본 포커스를 가질 버튼 번호 | 선택 |

### 아이콘 종류 (icon)

| 값 | 설명 |
|------|------|
| `Information!` | 정보 아이콘 (파란색 원 안에 i) |
| `StopSign!` | 오류/중지 아이콘 (빨간색 엑스) |
| `Exclamation!` | 경고 아이콘 (노란색 느낌표) |
| `Question!` | 질문 아이콘 (물음표) |
| `None!` | 아이콘 없음 |

### 버튼 종류 (button)

| 값 | 표시되는 버튼 |
|------|-------------|
| `OK!` | 확인 버튼 하나 |
| `OKCancel!` | 확인, 취소 버튼 |
| `YesNo!` | 예, 아니오 버튼 |
| `YesNoCancel!` | 예, 아니오, 취소 버튼 |
| `RetryCancel!` | 다시 시도, 취소 버튼 |
| `AbortRetryIgnore!` | 중단, 다시 시도, 무시 버튼 |

### 반환값

사용자가 클릭한 버튼에 따라 정수값이 반환됩니다.

| 반환값 | 클릭한 버튼 |
|--------|------------|
| 1 | 확인(OK), 예(Yes) |
| 2 | 취소(Cancel), 아니오(No) |
| 3 | 중단(Abort), 다시 시도(Retry), 무시(Ignore) |

### 예제

#### 간단한 정보 표시
```
MessageBox("알림", "저장이 완료되었습니다.", Information!)
```

#### 예/아니오 선택
```
Integer li_ans
li_ans = MessageBox("삭제 확인", "선택한 항목을 삭제하시겠습니까?", Question!, YesNo!)
IF li_ans = 1 THEN
   // 예 선택 시 삭제 실행
   dw_list.DeleteRow(0)
END IF
```

#### 오류 메시지와 함께 종료 선택
```
Integer li_ans
li_ans = MessageBox("시스템 오류", "치명적인 오류가 발생했습니다. 프로그램을 종료합니다.", StopSign!, OKCancel!)
IF li_ans = 1 THEN
   HALT CLOSE   // 애플리케이션 종료
ELSE
   RETURN
END IF
```

---

## 3. 사용자 정의 객체 (User Object)

사용자 정의 객체(User Object)는 PowerBuilder에서 개발자가 직접 정의하는 객체로, 반복적으로 사용되는 UI 요소나 비즈니스 로직을 하나의 단위로 묶어 재사용할 수 있게 해줍니다. 크게 **시각적 사용자 객체(Visual User Object)**와 **비시각적 사용자 객체(Class User Object)**로 나뉩니다.

### 사용자 객체의 장점

- **재사용성**: 한 번 만들어 놓은 객체를 여러 윈도우에서 사용 가능
- **일관성**: 동일한 기능과 디자인을 여러 곳에 적용 가능
- **유지보수성**: 객체 하나만 수정하면 이를 사용하는 모든 곳에 변경 내용 반영
- **모듈화**: 복잡한 로직을 캡슐화하여 코드 복잡도 감소

---

### 3.1 시각적 사용자 객체 (Visual User Object)

여러 컨트롤을 하나로 묶어 마치 하나의 컨트롤처럼 사용할 수 있도록 한 객체입니다. 예를 들어, "조회/저장/삭제" 버튼 세트를 사용자 객체로 만들면 여러 윈도우에서 간편히 배치할 수 있습니다.

#### 생성 방법

1. **File → New** → **PB Object** 탭 → **User Object** 선택
2. **Visual** 유형에서 상속받을 기본 컨트롤을 선택합니다.
   - `Custom` : 완전히 새로운 사용자 객체(여러 컨트롤을 조합할 때)
   - `CommandButton`, `DataWindow` 등 기존 컨트롤을 확장할 수도 있음
3. User Object Painter가 열리면 필요한 컨트롤을 배치하고 속성을 설정합니다.
4. 필요에 따라 사용자 정의 이벤트나 함수를 추가합니다.
5. 저장 시 이름은 보통 `u_` 접두사 사용 (예: `u_button_set`)

#### 예제: 조회/저장/삭제 버튼 세트

**u_button_set** 사용자 객체:
- CommandButton 3개: `cb_retrieve`, `cb_save`, `cb_delete`
- 각 버튼의 크기와 위치를 정렬하고, 적절한 텍스트 설정

**사용자 객체의 이벤트 처리**:
각 버튼의 Clicked 이벤트에서 바로 로직을 구현할 수도 있지만, 일반적으로는 이 사용자 객체를 사용하는 윈도우에서 필요한 처리를 하도록 **사용자 정의 이벤트**를 발생시키는 방식으로 설계합니다.

```
// u_button_set 내부 - cb_retrieve의 Clicked 이벤트
Parent.TriggerEvent("ue_retrieve")
```

이렇게 하면 이 사용자 객체를 배치한 윈도우에서 `ue_retrieve` 이벤트에 스크립트를 작성하여 각 윈도우에 맞는 조회 로직을 구현할 수 있습니다.

#### 윈도우에서 사용자 객체 사용

1. 윈도우를 열고 컨트롤 팔레트에서 **User Object** 아이콘을 선택합니다.
2. 생성한 사용자 객체(`u_button_set`)를 선택하여 윈도우에 배치합니다.
3. 배치된 사용자 객체의 이벤트(예: `ue_retrieve`)에 스크립트를 작성합니다.

```
// 윈도우 내에서 u_button_set의 ue_retrieve 이벤트
dw_customer.Retrieve()
```

#### 시각적 사용자 객체 상속

한 번 만든 사용자 객체를 다시 상속받아 기능을 추가하거나 변경할 수 있습니다. 예를 들어, `u_button_set`을 상속받아 `u_button_set_with_excel`을 만들고 엑셀 내보내기 버튼을 추가할 수 있습니다.

---

### 3.2 비시각적 사용자 객체 (Class User Object)

화면에 보이지 않으며, 비즈니스 로직, 데이터 처리, 공통 기능을 캡슐화하는 객체입니다. 전역 함수보다 더 객체 지향적인 방법으로 관련 함수와 데이터를 묶을 수 있습니다.

#### 생성 방법

1. **File → New** → **PB Object** 탭 → **User Object** 선택
2. **Class** 유형에서 상속받을 기본 클래스를 선택합니다.
   - `Custom` : 완전히 새로운 비시각적 객체
   - `Connection`, `Transaction`, `DataStore` 등 시스템 클래스를 확장할 수도 있음
3. User Object Painter에서 인스턴스 변수, 함수, 사용자 정의 이벤트를 정의합니다.
4. 저장 시 이름은 보통 `n_` 접두사 사용 (예: `n_calculator`, `n_customer_manager`)

#### 주요 구성 요소

- **인스턴스 변수 (Instance Variables)**: 객체 내부에서 사용할 데이터 (접근 지정자: PUBLIC, PROTECTED, PRIVATE)
- **함수 (Functions)**: 객체의 메서드
- **사용자 정의 이벤트 (User Events)**: 객체 내부에서 발생시키거나 외부에서 호출할 수 있는 이벤트
- **생성자(Constructor) / 소멸자(Destructor)**: 객체 생성/소멸 시 자동 실행되는 특수 이벤트

#### 예제: 간단한 계산기 객체

**n_calculator** 비시각적 사용자 객체 생성:

**인스턴스 변수 선언**
```
PRIVATE:
   Decimal id_last_result
```

**함수 of_add (Public)**
```
FUNCTION Decimal of_add (Decimal ad_a, Decimal ad_b)
   id_last_result = ad_a + ad_b
   RETURN id_last_result
END FUNCTION
```

**함수 of_multiply (Public)**
```
FUNCTION Decimal of_multiply (Decimal ad_a, Decimal ad_b)
   id_last_result = ad_a * ad_b
   RETURN id_last_result
END FUNCTION
```

**함수 of_get_last_result (Public)**
```
FUNCTION Decimal of_get_last_result ()
   RETURN id_last_result
END FUNCTION
```

**생성자(Constructor) 이벤트**
```
id_last_result = 0
MessageBox("알림", "계산기 객체가 생성되었습니다.")
```

**소멸자(Destructor) 이벤트**
```
MessageBox("알림", "계산기 객체가 소멸됩니다.")
```

#### 비시각적 객체 사용 방법

객체를 사용할 윈도우나 다른 객체의 스크립트에서 `CREATE` 문으로 인스턴스를 생성하고, `DESTROY`로 소멸시킵니다.

```
// 윈도우의 Open 이벤트
n_calculator lnv_calc
lnv_calc = CREATE n_calculator

// 계산기 사용
Decimal ld_result
ld_result = lnv_calc.of_add(10.5, 20.3)
MessageBox("결과", String(ld_result))   // 30.8

ld_result = lnv_calc.of_multiply(5, 4)
MessageBox("곱셈 결과", String(ld_result))   // 20

// 마지막 결과 조회
ld_result = lnv_calc.of_get_last_result()
MessageBox("마지막 결과", String(ld_result))   // 20

// 객체 소멸
DESTROY lnv_calc
```

#### 비시각적 객체의 장점

- 관련 함수들을 하나의 객체로 묶어 네임스페이스처럼 사용 가능
- 인스턴스 변수를 통해 상태 유지 가능
- 여러 곳에서 동일한 로직을 사용할 때 코드 중복 방지
- 상속을 통해 기능 확장 가능

#### DataStore를 활용한 데이터 관리 객체 (수정된 예제)

비시각적 객체는 데이터베이스 연동 로직을 캡슐화하는 데 자주 사용됩니다. 아래 예제는 DataStore를 감싸는 NVO를 만들고, 윈도우의 DataWindow와 데이터를 공유하는 올바른 방법을 보여줍니다.

**n_customer_ds** 객체:
- 내부에 DataStore 인스턴스 변수 보유
- 생성자에서 DataStore 생성 및 DataObject 할당
- Retrieve, Update 등 메서드 제공
- DataStore 참조를 반환하는 함수 제공

```
// n_customer_ds의 인스턴스 변수
PRIVATE:
   DataStore ids_cust

// Constructor 이벤트
ids_cust = CREATE DataStore
ids_cust.DataObject = "d_customer_list"
ids_cust.SetTransObject(SQLCA)

// of_retrieve 함수
FUNCTION Long of_retrieve (String as_cust_id)
   RETURN ids_cust.Retrieve(as_cust_id)
END FUNCTION

// of_get_data 함수 (DataStore 참조 반환)
FUNCTION DataStore of_get_data ()
   RETURN ids_cust
END FUNCTION

// Destructor 이벤트
DESTROY ids_cust
```

**윈도우에서 사용** (수정됨)
```
// 윈도우의 인스턴스 변수 (Instance Variables 영역에 선언)
n_customer_ds inv_cust

// 윈도우의 Open 이벤트
inv_cust = CREATE n_customer_ds
Long ll_count = inv_cust.of_retrieve("C001")

// DataWindow 트랜잭션 연결
dw_list.SetTransObject(SQLCA)

// 원본(DataStore)이 대상(dw_list)에게 데이터를 공유
inv_cust.of_get_data().ShareData(dw_list)

// 윈도우의 Close 이벤트
DESTROY inv_cust
```

> **⚠️ 주의**: `ShareData`는 데이터를 복사하는 것이 아니라 메모리 포인터를 공유하는 기능입니다. 따라서 데이터를 제공하는 원본 객체(NVO 내의 DataStore)는 화면이 유지되는 동안 반드시 살아 있어야 합니다. 위 예제처럼 NVO 객체를 윈도우의 인스턴스 변수로 선언하고 윈도우가 닫힐 때 파괴해야 안전합니다.

---

### 3.3 사용자 객체와 상속

사용자 객체는 상속을 통해 확장할 수 있습니다. 예를 들어, 기본 계산기 객체(`n_calculator`)를 상속받아 공학용 계산기 객체(`n_sci_calculator`)를 만들 수 있습니다.

```
// n_sci_calculator (n_calculator 상속)
// 추가 함수: of_sqrt, of_power 등

FUNCTION Decimal of_sqrt (Decimal ad_val)
   id_last_result = Sqrt(ad_val)
   RETURN id_last_result
END FUNCTION
```

상속받은 객체는 부모의 모든 Public/Protected 함수와 변수를 사용할 수 있습니다.

---

## 실전 예제: 로그인 사용자 객체

간단한 로그인 기능을 비시각적 사용자 객체로 구현해 보겠습니다.

**n_login** 객체 생성:

**인스턴스 변수**
```
PRIVATE:
   String is_user_id
   Boolean ib_authenticated
```

**함수 of_authenticate (Public)**
```
FUNCTION Boolean of_authenticate (String as_id, String as_pwd)
   // 실제로는 데이터베이스에서 검증
   IF as_id = "admin" AND as_pwd = "1234" THEN
      is_user_id = as_id
      ib_authenticated = TRUE
      RETURN TRUE
   ELSE
      ib_authenticated = FALSE
      RETURN FALSE
   END IF
END FUNCTION
```

**함수 of_get_user_id (Public)**
```
FUNCTION String of_get_user_id ()
   RETURN is_user_id
END FUNCTION
```

**함수 of_is_authenticated (Public)**
```
FUNCTION Boolean of_is_authenticated ()
   RETURN ib_authenticated
END FUNCTION
```

**윈도우(w_login)에서 사용**
```
// 전역 변수로 선언하거나 윈도우 인스턴스 변수로 선언
n_login gnv_login

// w_login의 Open 이벤트
gnv_login = CREATE n_login

// 로그인 버튼 Clicked 이벤트
IF gnv_login.of_authenticate(sle_id.Text, sle_pwd.Text) THEN
   MessageBox("성공", "환영합니다, " + gnv_login.of_get_user_id() + "님!")
   Close(Parent)   // 로그인 윈도우 닫기
   Open(w_main)    // 메인 윈도우 열기
ELSE
   MessageBox("실패", "아이디 또는 비밀번호가 틀립니다.")
END IF

// w_main의 Close 이벤트에서 로그인 객체 소멸
DESTROY gnv_login
```

---

## 결론

이벤트 처리는 PowerBuilder 애플리케이션의 동작을 정의하는 핵심이며, 각 객체의 이벤트에 적절한 스크립트를 작성하는 것이 개발의 대부분을 차지합니다. `Trigger`와 `Post`의 차이를 이해하고, 상속 관계에서 `CALL` 문을 활용하면 더 유연한 이벤트 처리가 가능합니다.

메시지 박스는 사용자와의 간단한 상호작용에 필수적인 도구로, 다양한 아이콘과 버튼 조합을 통해 상황에 맞는 대화상을 제공할 수 있습니다.

사용자 정의 객체는 PowerBuilder의 강력한 재사용 메커니즘입니다. 시각적 사용자 객체로 UI 컴포넌트를 표준화하고, 비시각적 사용자 객체로 비즈니스 로직을 캡슐화하면 코드의 중복을 줄이고 유지보수성을 크게 향상시킬 수 있습니다. 특히 비시각적 객체는 3-Tier 아키텍처의 비즈니스 계층을 구현하는 기본 단위가 됩니다.
