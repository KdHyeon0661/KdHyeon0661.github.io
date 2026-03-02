---
layout: post
title: PowerBuilder - 객체
date: 2025-12-12 20:30:23 +0900
category: PowerBuilder
---
# 객체(Object) 개념

PowerBuilder는 객체 지향 프로그래밍(Object-Oriented Programming, OOP)을 지원하는 4GL 언어입니다. 객체란 현실 세계의 사물이나 개념을 소프트웨어적으로 모델링한 단위로, 데이터(속성)와 그 데이터를 처리하는 행위(메서드, 이벤트)를 하나로 묶은 것입니다. PowerBuilder의 모든 구성 요소(윈도우, 버튼, DataWindow, 메뉴 등)는 객체이며, 객체 지향의 개념을 이해하면 더 효율적이고 유지보수하기 쉬운 애플리케이션을 개발할 수 있습니다.

---

## 객체(Object)란 무엇인가?

객체는 **상태(state)**와 **행동(behavior)**을 가지는 독립적인 실체입니다. 

- **상태**: 객체가 가지고 있는 데이터(속성, 프로퍼티)로, 예를 들어 윈도우 객체의 너비, 높이, 제목 등이 상태입니다.
- **행동**: 객체가 수행할 수 있는 동작(메서드, 함수, 이벤트)으로, 예를 들어 버튼 객체가 클릭되었을 때 실행되는 스크립트가 행동입니다.

PowerBuilder에서 객체는 **클래스(class)**라는 설계도를 기반으로 생성됩니다. 클래스는 같은 종류의 객체들이 공통적으로 가지는 속성과 행동을 정의한 틀입니다. 예를 들어, `CommandButton` 클래스는 모든 명령 버튼이 공통으로 가지는 속성(Text, Enabled 등)과 이벤트(Clicked 등)를 정의합니다. 실제 애플리케이션에서 사용되는 각각의 버튼은 이 클래스의 **인스턴스(instance)**입니다.

---

## PowerBuilder 객체의 종류

PowerBuilder의 객체는 크게 시각적 객체, 비시각적 객체, 시스템 객체로 나눌 수 있습니다.

### 1. 시각적 객체 (Visual Objects)

사용자 인터페이스를 구성하는 객체로, 화면에 보이고 사용자와 상호작용합니다.

- **Window (윈도우)**: 애플리케이션의 기본 화면 단위
- **Control (컨트롤)**: 윈도우 위에 배치되는 요소들
  - CommandButton, CheckBox, RadioButton, StaticText, SingleLineEdit, MultiLineEdit, ListBox, DropDownListBox, DataWindow, Picture, Tab, GroupBox 등
- **Menu (메뉴)**: 윈도우 상단이나 팝업 메뉴

시각적 객체는 대부분 상속을 통해 확장하여 재사용할 수 있습니다.

### 2. 비시각적 객체 (Non-Visual Objects)

화면에 보이지 않지만 비즈니스 로직, 데이터 처리, 공통 기능을 담당하는 객체입니다.

- **Transaction Object (트랜잭션 객체)**: 데이터베이스 연결 관리 (SQLCA 등)
- **DataStore**: DataWindow의 비시각적 버전, 백그라운드 데이터 처리
- **User Object (사용자 객체)**: 개발자가 직접 정의하는 객체로, 시각적/비시각적 유형이 있음
  - **Visual User Object**: 여러 컨트롤을 묶어 재사용 가능한 컴포넌트로 만든 것
  - **Class User Object (Non-Visual)**: 비즈니스 로직을 캡슐화한 비시각적 객체 (예: 계산기, 데이터 검증 등)
- **Structure (구조체)**: 데이터를 그룹화하지만 메서드는 없음 (엄밀히 객체는 아니지만 데이터 컨테이너)

### 3. 시스템 객체

PowerBuilder 실행 환경에서 제공하는 내장 객체들입니다.

- **Message**: 윈도우 메시지 정보를 전달하는 객체
- **Error**: 런타임 오류 정보를 담는 객체
- **Environment**: 운영체제, 화면 해상도 등 환경 정보

---

## 객체의 구성 요소

모든 객체는 다음과 같은 세 가지 주요 요소로 구성됩니다.

### 속성 (Properties)

객체의 상태를 나타내는 데이터입니다. 속성은 객체의 특징(크기, 색상, 활성화 여부 등)을 저장합니다.

- 점(`.`) 표기법으로 접근합니다.
- 읽기/쓰기 가능한 속성과 읽기 전용 속성이 있습니다.

예제:
```
// 윈도우의 제목 속성 변경
w_main.Title = "고객 관리 시스템"

// 버튼의 활성화 여부 설정
cb_save.Enabled = FALSE

// DataWindow의 배경색 속성 읽기
Long ll_color = dw_list.BackColor
```

### 이벤트 (Events)

객체에서 발생하는 특정 사건(사용자 동작, 시스템 알림)입니다. 각 이벤트에는 스크립트를 연결하여 해당 사건 발생 시 실행할 코드를 작성할 수 있습니다.

- 시스템 정의 이벤트: `Clicked`, `Open`, `Close`, `Timer`, `Key` 등
- 사용자 정의 이벤트: 개발자가 필요에 따라 추가할 수 있는 이벤트 (`ue_` 접두사 사용)

예제:
```
// 버튼 cb_ok의 Clicked 이벤트
cb_ok.Text = "확인"
MessageBox("알림", "버튼이 클릭되었습니다.")
```

### 함수 (Functions)

객체가 수행할 수 있는 동작(메서드)입니다. PowerBuilder에서는 함수(Function)라고 부릅니다.

- 내장 함수: 객체가 기본적으로 제공하는 함수 (예: `Print()`, `SetRedraw()`)
- 사용자 정의 함수: 객체 수준에서 추가로 정의한 함수

예제:
```
// DataWindow의 내장 함수 호출
dw_list.SetTransObject(SQLCA)
dw_list.Retrieve()

// 사용자 정의 함수 호출 (윈도우에 정의한 uf_refresh)
THIS.uf_refresh()
```

---

## 객체 지향 프로그래밍의 3대 요소

PowerBuilder는 객체 지향 언어로서 캡슐화, 상속, 다형성을 지원합니다.

### 1. 캡슐화 (Encapsulation)

캡슐화는 객체의 내부 데이터(인스턴스 변수)를 외부에서 직접 접근하지 못하게 보호하고, 공개된 인터페이스(함수, 이벤트)를 통해서만 조작하도록 하는 개념입니다.

PowerBuilder에서는 인스턴스 변수에 **접근 지정자(Access Modifier)**를 사용하여 캡슐화를 구현합니다.

- **PUBLIC**: 모든 스크립트에서 접근 가능
- **PROTECTED**: 해당 객체와 상속받은 객체에서만 접근 가능
- **PRIVATE**: 해당 객체 내에서만 접근 가능

```
// 윈도우의 인스턴스 변수 선언 예
PRIVATE Integer ii_counter
PROTECTED String is_internal
PUBLIC String gs_visible
```

또한 속성(Property)을 직접 노출하는 대신, Get/Set 함수를 제공하여 데이터 무결성을 유지할 수 있습니다.

### 2. 상속 (Inheritance)

상속은 기존 클래스(부모)의 속성, 이벤트, 함수를 그대로 물려받아 새로운 클래스(자식)를 정의하는 것입니다. PowerBuilder는 윈도우, 메뉴, 사용자 객체 등에서 상속을 지원합니다.

**상속의 장점**:
- 코드 재사용성 증가
- 공통 기능을 부모에 정의하여 일관성 유지
- 유지보수 용이 (부모 수정 시 자식에 반영)

**상속 관계에서의 객체**:
- 부모 객체의 모든 Public/Protected 멤버는 자식에게 상속됩니다.
- 자식 객체는 부모의 이벤트 스크립트나 함수를 **오버라이드(Override)**하여 재정의할 수 있습니다.
- `CALL` 문을 사용하여 부모의 원본 함수나 이벤트를 호출할 수 있습니다.

```
// 부모 윈도우 w_parent에 정의된 함수 uf_display
FUNCTION uf_display()
   MessageBox("정보", "부모 윈도우")
END FUNCTION

// 자식 윈도우 w_child에서 오버라이드
FUNCTION uf_display()
   MessageBox("정보", "자식 윈도우")
   CALL w_parent::uf_display   // 부모 함수 호출 (선택)
END FUNCTION
```

**상속 계층도** (PowerBuilder Object Browser에서 시각적으로 확인 가능):
```
   [Window] (시스템 기본 클래스)
       ↑
   [w_parent] (사용자 정의 부모)
       ↑
   [w_child]  (자식)
       ↑
   [w_grandchild] (손자)
```
> **📸 추천 스크린샷**: Object Browser에서 상속 트리를 보여주는 화면을 캡처하여 이 부분에 첨부하면 독자의 이해가 훨씬 쉬워집니다.

### 3. 다형성 (Polymorphism)

다형성은 같은 이름의 메서드나 이벤트가 객체에 따라 다르게 동작하는 능력입니다. PowerBuilder에서는 다음과 같은 방식으로 다형성을 구현합니다.

- **오버라이딩(Overriding)**: 상속 관계에서 자식 클래스가 부모의 메서드를 재정의
- **오버로딩(Overloading)**: 같은 클래스 내에서 함수 이름은 같지만 인자의 개수나 타입이 다른 여러 함수를 정의

```
// 오버로딩 예제 (같은 클래스 내)
FUNCTION Integer f_calc (Integer a, Integer b)
   RETURN a + b
END FUNCTION

FUNCTION Integer f_calc (Integer a, Integer b, Integer c)
   RETURN a + b + c
END FUNCTION

// 호출 시 인자에 따라 적절한 함수 선택
li_result = f_calc(10, 20)      // 첫 번째 함수
li_result = f_calc(10, 20, 30)  // 두 번째 함수
```

---

## 객체의 생성과 소멸

### 시각적 객체 생성

윈도우와 같은 시각적 객체는 일반적으로 `OPEN` 함수를 사용하여 생성하고 화면에 표시합니다.

```
// 단일 윈도우 열기
Open(w_main)
```

동일한 윈도우 클래스의 여러 인스턴스를 띄우려면 각각 다른 변수에 할당하여 `OPEN`합니다.

```
// 같은 윈도우(w_customer_detail)를 여러 개 인스턴스로 생성
w_customer_detail lw_detail1, lw_detail2
Open(lw_detail1)   // 첫 번째 상세 창 열기
Open(lw_detail2)   // 두 번째 상세 창 열기
```

윈도우를 닫을 때는 `CLOSE` 함수를 사용합니다.

```
CLOSE(w_main)
```

### 비시각적 객체 생성

비시각적 객체(사용자 객체, 트랜잭션 객체 등)는 `CREATE` 문으로 생성하고, `DESTROY` 문으로 소멸시킵니다.

```
// 사용자 객체 생성
n_calculator lnv_calc
lnv_calc = CREATE n_calculator

// 객체 사용
lnv_calc.of_calculate(100, 200)

// 객체 소멸 (메모리 해제)
DESTROY lnv_calc
```

트랜잭션 객체는 기본적으로 `SQLCA`라는 전역 객체가 자동 생성되지만, 추가 트랜잭션 객체가 필요하면 직접 생성합니다.

```
// 보조 트랜잭션 객체 생성
transaction ltr_second
ltr_second = CREATE transaction
ltr_second.DBMS = "ODBC"
// ... 연결 설정
CONNECT USING ltr_second;

// 사용 후
DISCONNECT USING ltr_second;
DESTROY ltr_second
```

### 참조 카운팅과 메모리 관리

PowerBuilder는 객체 참조 카운트를 관리하여 더 이상 사용되지 않는 객체를 자동으로 소멸시키는 경우도 있습니다. 하지만 `CREATE`로 생성한 비시각적 객체는 반드시 `DESTROY`로 명시적으로 해제해야 메모리 누수를 방지할 수 있습니다.

---

## 객체 간 통신

객체들은 서로 메시지를 주고받으며 협력합니다. PowerBuilder에서 객체 간 통신 방법은 다음과 같습니다.

### 직접 함수/이벤트 호출

한 객체가 다른 객체의 함수나 이벤트를 직접 호출합니다.

```
// 다른 윈도우의 함수 호출
w_other.uf_refresh()

// 다른 윈도우의 사용자 정의 이벤트 트리거
w_other.TriggerEvent("ue_update")
```

### 메시지 전달 (이벤트)

`TRIGGER EVENT`와 `POST EVENT`를 사용하여 이벤트를 발생시킵니다.

- `TRIGGER EVENT`: 동기적 호출, 이벤트 처리가 끝날 때까지 기다림
- `POST EVENT`: 비동기적 호출, 이벤트를 메시지 큐에 넣고 즉시 다음 코드 실행

```
// 다른 객체의 이벤트를 트리거
Parent.TriggerEvent("ue_refresh")

// 이벤트 포스트
Parent.PostEvent("ue_long_task")
```

### 객체 참조 전달

함수 호출 시 객체 참조를 인자로 전달하여 해당 객체의 메서드를 호출할 수 있게 합니다.

```
// 함수 정의: 다른 윈도우 참조를 받아 그 윈도우의 함수 호출
FUNCTION uf_call_other (window aw_win)
   aw_win.uf_something()
END FUNCTION

// 호출
uf_call_other(w_main)
```

---

## 객체 브라우저(Object Browser)

PowerBuilder IDE는 **Object Browser**라는 강력한 도구를 제공하여 애플리케이션에 정의된 모든 객체와 시스템 객체의 구조를 탐색할 수 있습니다.

- **열기 방법**: 도구 모음에서 **Browse** 아이콘 클릭 또는 **Tools → Browser** 메뉴
- **기능**:
  - 객체 유형별(윈도우, 메뉴, 함수, 구조체, 사용자 객체 등) 목록 표시
  - 객체의 속성, 이벤트, 함수 시그니처 확인
  - 상속 계층도 표시 (부모-자식 관계를 트리로 시각화)
  - 객체 간 참조 관계 확인
  - 코드로 드래그 앤 드롭하여 자동 코드 생성

Object Browser를 활용하면 상속 구조를 파악하고, 객체가 제공하는 인터페이스를 빠르게 확인할 수 있어 개발 생산성이 향상됩니다.

---

## 실전 예제: 사용자 객체(User Object) 만들기

비시각적 사용자 객체를 만들어 비즈니스 로직을 캡슐화해 보겠습니다.

### 1. 비시각적 사용자 객체 생성

**파일 → New → PB Object 탭 → Standard Class** 선택  
Standard Class 유형에서 `nonvisualobject`를 선택합니다.

### 2. 객체 이름과 인스턴스 변수 정의

```
// n_customer_manager (고객 관리자 객체)
// 인스턴스 변수 선언
PRIVATE:
   transaction itr_trans   // 데이터베이스 트랜잭션 객체
   DataStore ids_cust      // 고객 데이터 저장소
```

### 3. 생성자(Constructor) 이벤트에 초기화 코드 작성

```
// Constructor 이벤트
// 트랜잭션 객체 연결 (SQLCA 사용)
itr_trans = SQLCA
ids_cust = CREATE DataStore
ids_cust.DataObject = "d_customer_list"
ids_cust.SetTransObject(itr_trans)
```

### 4. 비즈니스 메서드 추가

```
// 함수 of_get_customer_count: 고객 수 반환
FUNCTION Long of_get_customer_count ()
   Long ll_count
   ll_count = ids_cust.Retrieve()
   RETURN ll_count
END FUNCTION

// 함수 of_get_customer_name: 고객ID로 이름 조회
FUNCTION String of_get_customer_name (String as_cust_id)
   Long ll_row
   ll_row = ids_cust.Find("cust_id = '" + as_cust_id + "'", 1, ids_cust.RowCount())
   IF ll_row > 0 THEN
      RETURN ids_cust.GetItemString(ll_row, "cust_name")
   ELSE
      RETURN ""
   END IF
END FUNCTION
```

### 5. 소멸자(Destructor) 이벤트에서 정리

```
// Destructor 이벤트
DESTROY ids_cust
```

### 6. 객체 사용 (윈도우 스크립트에서)

```
// 인스턴스 생성
n_customer_manager lnv_mgr
lnv_mgr = CREATE n_customer_manager

// 메서드 호출
Long ll_count = lnv_mgr.of_get_customer_count()
String ls_name = lnv_mgr.of_get_customer_name("C001")

MessageBox("결과", "고객 수: " + String(ll_count) + "~r~n고객명: " + ls_name)

// 객체 소멸
DESTROY lnv_mgr
```

이 예제에서 `n_customer_manager` 객체는 고객 관련 비즈니스 로직을 캡슐화하고 있으며, 윈도우는 이 객체의 인터페이스만 알면 됩니다. 내부 구현이 변경되어도 윈도우 코드는 영향을 받지 않습니다.

---

## 객체 지향 설계 팁

1. **단일 책임 원칙**: 하나의 객체는 하나의 명확한 책임만 가져야 합니다.
2. **낮은 결합도**: 객체 간 의존성을 최소화하고 인터페이스를 통해 통신합니다.
3. **높은 응집도**: 관련된 속성과 메서드는 하나의 객체에 모읍니다.
4. **상속은 is-a 관계일 때만 사용**: "자동차는 탈것이다"와 같은 명확한 관계일 때 상속을 고려합니다. 단순 코드 재사용을 위해서는 포함(Composition)을 고려합니다.
5. **사용자 객체를 적극 활용**: 반복되는 비즈니스 로직은 비시각적 사용자 객체로 만들어 재사용합니다.

---

## 결론

PowerBuilder에서 객체 개념은 단순히 화면의 컨트롤을 넘어, 애플리케이션 전체를 구성하는 핵심 단위입니다. 객체 지향의 3대 요소(캡슐화, 상속, 다형성)를 이해하고 활용하면 코드의 재사용성과 유지보수성을 크게 향상시킬 수 있습니다.

특히 비시각적 사용자 객체를 통해 비즈니스 로직을 캡슐화하면 프레젠테이션 계층(윈도우)과 비즈니스 계층을 분리하여 향후 3-Tier 아키텍처로의 전환도 용이해집니다. 객체의 생성과 소멸을 명확히 관리하고, 객체 간 통신 방식을 이해하는 것이 안정적인 애플리케이션 개발의 기초입니다.
