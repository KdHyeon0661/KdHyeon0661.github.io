---
layout: post
title: PowerBuilder - MDI 및 사용자 정의 클래스
date: 2025-12-13 23:30:23 +0900
category: PowerBuilder
---
# MDI (Multiple Document Interface) 구조 및 사용자 정의 클래스 (Custom Class User Object)

PowerBuilder로 애플리케이션을 개발할 때, 사용자 인터페이스 아키텍처는 애플리케이션의 성격에 따라 달라집니다. 단일 윈도우에서 모든 작업을 처리하는 SDI(Single Document Interface) 방식도 있지만, 여러 개의 문서를 동시에 열어 작업해야 하는 경우에는 **MDI(Multiple Document Interface)** 구조가 적합합니다. 또한 애플리케이션의 비즈니스 로직을 체계적으로 관리하고 재사용성을 높이기 위해 **사용자 정의 클래스(Custom Class User Object)**를 활용할 수 있습니다. 이 글에서는 MDI 구조의 개념과 구현 방법, 그리고 비시각적 사용자 정의 클래스의 작성 및 활용법을 상세히 알아보겠습니다.

---

## 1. MDI (Multiple Document Interface) 구조

### 1.1 MDI란?

MDI는 **하나의 프레임 윈도우(Frame Window)** 안에 여러 개의 **자식 윈도우(Child Window)** 또는 **시트(Sheet)**를 띄워 동시에 작업할 수 있는 사용자 인터페이스 구조입니다. 대표적인 예로 마이크로소프트 엑셀, 워드, 포토샵 등이 있습니다. 사용자는 여러 문서를 동시에 열고 각각을 독립적으로 편집할 수 있으며, 프레임 윈도우는 전체적인 메뉴와 도구 모음을 제공합니다.

### 1.2 PowerBuilder의 MDI 지원

PowerBuilder는 MDI 애플리케이션 개발을 위해 두 가지 유형의 윈도우를 제공합니다.

| 윈도우 유형 | 설명 |
|------------|------|
| **MDI Frame (mdi!)** | MDI 프레임 윈도우. 내부에 여러 개의 MDI 시트를 띄울 수 있으며, 메뉴와 도구 모음을 가질 수 있습니다. |
| **MDI Frame with MicroHelp (mdihelp!)** | MDI 프레임에 상태 표시줄(MicroHelp)이 추가된 형태. 프레임 하단에 도움말 메시지를 표시할 수 있습니다. |

MDI 프레임 내부에 띄워지는 시트는 일반적으로 **Main!** 타입의 윈도우이지만, MDI 프레임 안에서는 시트로 동작합니다. 시트는 프레임 내에서만 이동, 크기 조절이 가능하며, 프레임 밖으로 나갈 수 없습니다.

### 1.3 MDI Frame 윈도우 생성

**MDI Frame 생성 단계**:
1. **File → New** → **PB Object** 탭 → **Window** 선택
2. 속성 시트에서 **WindowType**을 `mdi!` 또는 `mdihelp!`로 설정
3. **MenuName** 속성에 메뉴 객체를 반드시 연결합니다.
4. 마이크로헬프를 사용하려면 `mdihelp!` 선택

> **⚠️ 중요**: MDI 프레임 윈도우는 **반드시 메뉴(Menu) 객체가 연결되어 있어야 정상적으로 동작**합니다. 메뉴를 연결하지 않으면 윈도우가 열릴 때 런타임 오류가 발생하거나, 시트가 프레임 밖으로 튀어나가는 심각한 버그가 발생할 수 있습니다.

**예: MDI Frame 윈도우 정의**
```
Window: w_frame
WindowType: mdihelp!
Title: "주문 관리 시스템"
MenuName: m_main
```

### 1.4 MDI Sheet 열기

MDI 프레임 내에 시트를 열려면 **OpenSheet()** 함수를 사용합니다.

**문법**:
```powerbuilder
OpenSheet ( sheetref, mdiframe {, position {, arrangeopen } } )
```

- **sheetref**: 열 시트 윈도우 (예: `w_customer`)
- **mdiframe**: MDI 프레임 윈도우 참조 (예: `w_frame`)
- **position**: (선택) 시트 제목이 표시될 메뉴 항목의 위치 (기본값 0)
- **arrangeopen**: (선택) 시트 배열 방식 (`Cascade!`, `Layer!`, `Original!`, `Tile!`)

**예제**:
```powerbuilder
// w_frame의 메뉴에서 "고객 관리" 시트 열기
OpenSheet(w_customer, w_frame, 0, Cascade!)
```

여러 개의 시트를 열면 프레임 내에 계단식(Cascade) 또는 타일(Tile) 형태로 배열할 수 있습니다.

### 1.5 메뉴 병합 (Menu Merging)

MDI 애플리케이션에서는 프레임 메뉴와 시트 메뉴가 동적으로 결합(병합)될 수 있습니다. 시트가 활성화되면 해당 시트에 정의된 메뉴 항목이 프레임 메뉴에 추가되거나 대체됩니다.

**메뉴 병합 설정**:
- 각 메뉴 항목의 **MenuMerge** 속성을 설정합니다.
  - `insert!`: 프레임 메뉴의 지정된 위치에 삽입
  - `replace!`: 프레임 메뉴의 해당 위치를 대체
  - `hide!`: 시트 활성화 시 숨김

**예**:
프레임 메뉴에 "파일", "편집" 메뉴가 있고, 시트 메뉴에 "고객" 메뉴가 있는 경우, 시트가 활성화되면 "고객" 메뉴가 추가됩니다.

### 1.6 MicroHelp 활용

`mdihelp!` 타입 프레임에서는 **MicroHelp**라는 상태 표시줄을 사용할 수 있습니다. 주로 메뉴 항목에 마우스를 올렸을 때 간단한 설명을 표시하거나, 현재 작업 상태를 표시하는 데 사용됩니다.

**MicroHelp 설정 방법**:
- 메뉴 항목의 **MicroHelp** 속성에 문자열을 입력합니다.
- 코드에서 `SetMicroHelp()` 함수를 사용하여 동적으로 변경할 수 있습니다.

```powerbuilder
// 메뉴 항목에 마우스를 올리면 자동으로 표시됨
// 수동으로 변경하려면
w_frame.SetMicroHelp("고객 데이터를 조회 중입니다...")
```

### 1.7 현재 활성 시트 확인 및 제어

MDI 프레임에서 현재 활성화된 시트를 얻거나 제어하려면 다음과 같은 방법을 사용합니다.

```powerbuilder
// 현재 활성 시트의 윈도우 핸들(참조) 가져오기
Window lw_active
lw_active = w_frame.GetActiveSheet()

IF IsValid(lw_active) THEN
    // 활성 시트가 있으면
    lw_active.Title = "변경된 제목"
END IF
```

**모든 시트 순회 (파워빌더 표준 방식)**:
```powerbuilder
Window lw_sheet

// 1. 첫 번째 시트를 가져옵니다.
lw_sheet = w_frame.GetFirstSheet()

// 2. 시트가 존재하는 동안 루프를 돕니다.
DO WHILE IsValid(lw_sheet)
    // (이곳에서 lw_sheet를 제어하는 작업 수행)
    // 예: lw_sheet.TriggerEvent("ue_save")
    
    // 3. 현재 시트를 기준으로 '다음 시트'를 가져옵니다.
    lw_sheet = w_frame.GetNextSheet(lw_sheet)
LOOP
```

### 1.8 MDI 애플리케이션 설계 시 고려사항

- **시트 간 데이터 공유**: 여러 시트가 동일한 데이터를 공유해야 한다면, DataStore를 이용한 공유 또는 전역 변수/객체를 활용합니다.
- **메모리 관리**: 열린 시트는 닫힐 때 메모리에서 해제되지만, 필요에 따라 `Close()` 호출을 명시적으로 해야 합니다.
- **사용자 경험**: 시트가 많아질 경우 네비게이션이 복잡해질 수 있으므로, 윈도우 메뉴에 열린 시트 목록을 표시하는 기능을 추가하는 것이 좋습니다.

### 1.9 간단한 MDI 예제: 문서 편집기

**구성**:
- 프레임 윈도우: `w_frame` (mdihelp!, 메뉴: `m_frame`)
- 시트 윈도우: `w_edit` (main!, 제목: "문서1", MultiLineEdit 컨트롤 포함)

**m_frame 메뉴**:
- "파일" → "새 문서" (이벤트: `cb_new`)
- "파일" → "닫기" (이벤트: `cb_close`)

**스크립트**:
```powerbuilder
// 새 문서 메뉴 클릭
OpenSheet(w_edit, w_frame, 0, Cascade!)

// 닫기 메뉴 클릭
Window lw_active
lw_active = w_frame.GetActiveSheet()
IF IsValid(lw_active) THEN
    Close(lw_active)
END IF
```

**w_edit의 CloseQuery 이벤트**에서 저장 확인 로직 추가 가능.

---

## 2. 사용자 정의 클래스 (Custom Class User Object)

### 2.1 개념

**사용자 정의 클래스**는 PowerBuilder에서 제공하는 **비시각적 사용자 객체(Non-Visual User Object)**의 한 종류로, 순수하게 비즈니스 로직, 데이터 처리, 공통 기능을 캡슐화하는 데 사용됩니다. 이는 객체 지향 프로그래밍의 **클래스** 개념에 해당하며, 속성(인스턴스 변수)과 메서드(함수, 이벤트)를 가질 수 있습니다.

### 2.2 생성 방법

1. **File → New** → **PB Object** 탭 → **User Object** 선택
2. **Class** 유형에서 **Custom** 선택 (또는 표준 클래스를 상속받으려면 해당 클래스 선택)
3. User Object Painter에서 다음을 정의합니다.
   - **인스턴스 변수 (Instance Variables)**: 객체의 상태를 저장
   - **함수 (Functions)**: 객체의 동작 정의
   - **사용자 정의 이벤트 (User Events)**: 객체 내부 이벤트
   - **Constructor/Destructor**: 생성자/소멸자 이벤트

**저장** 시 보통 `n_` 접두사를 사용합니다 (예: `n_customer_manager`, `n_utility`).

### 2.3 구성 요소

#### 인스턴스 변수 (Instance Variables)

객체의 데이터를 저장합니다. 접근 지정자를 사용하여 캡슐화할 수 있습니다.

```powerbuilder
// 인스턴스 변수 선언 예
PRIVATE:
    String is_connection_string
    Boolean ib_connected

PUBLIC:
    String is_last_error
```

#### 함수 (Functions)

객체의 메서드를 정의합니다. 함수는 반환값과 인자를 가질 수 있습니다.

```powerbuilder
// n_calculator의 함수 of_add
FUNCTION Decimal of_add (Decimal ad_a, Decimal ad_b)
    RETURN ad_a + ad_b
END FUNCTION
```

#### 사용자 정의 이벤트 (User Events)

객체 내부에서 발생시키거나 외부에서 트리거할 수 있는 이벤트입니다. `ue_` 접두사를 주로 사용합니다.

```powerbuilder
// 객체 내부에서 이벤트 발생
This.TriggerEvent("ue_process_completed")
```

#### 생성자(Constructor) / 소멸자(Destructor)

객체가 생성될 때(`CREATE`)와 소멸될 때(`DESTROY`) 자동으로 실행되는 특수 이벤트입니다. 초기화 및 정리 코드를 작성합니다.

```powerbuilder
// Constructor 이벤트
is_connection_string = "DSN=MyDB;UID=sa;PWD=1234"
ib_connected = FALSE

// Destructor 이벤트
IF ib_connected THEN
    DISCONNECT USING SQLCA;
END IF
```

### 2.4 사용자 정의 클래스 사용

**객체 생성 및 소멸**:
```powerbuilder
n_calculator lnv_calc
lnv_calc = CREATE n_calculator
// 사용
Decimal ld_result = lnv_calc.of_add(10, 20)
// 소멸
DESTROY lnv_calc
```

**객체 참조 전달**:
함수의 인자나 반환값으로 객체를 전달할 수 있습니다.

### 2.5 상속과 다형성

사용자 정의 클래스도 상속이 가능합니다. 기본 클래스를 만들고 이를 상속받아 기능을 확장하거나 오버라이드할 수 있습니다.

**예: 기본 계산기 클래스를 상속받은 공학용 계산기**
```powerbuilder
// n_adv_calculator (n_calculator 상속)
// 추가 함수 of_power
FUNCTION Decimal of_power (Decimal ad_base, Integer ai_exp)
    Decimal ld_result = 1
    FOR i = 1 TO ai_exp
        ld_result *= ad_base
    NEXT
    RETURN ld_result
END FUNCTION
```

### 2.6 실전 활용 예제

#### 예제 1: 데이터베이스 연결 관리 클래스

**n_db_manager** 클래스:
- 데이터베이스 연결, 연결 상태 관리, 오류 처리 등을 캡슐화

**인스턴스 변수**:
```powerbuilder
PRIVATE:
    Transaction itr_trans
    Boolean ib_connected
PUBLIC:
    String is_last_error
```

**Constructor**:
```powerbuilder
itr_trans = SQLCA   // 또는 CREATE Transaction
```

**함수 of_connect**:
```powerbuilder
FUNCTION Boolean of_connect (String as_dbms, String as_server, String as_db, String as_id, String as_pwd)
    itr_trans.DBMS = as_dbms
    itr_trans.ServerName = as_server
    itr_trans.Database = as_db
    itr_trans.LogID = as_id
    itr_trans.LogPass = as_pwd
    CONNECT USING itr_trans;
    IF itr_trans.SQLCode = 0 THEN
        ib_connected = TRUE
        RETURN TRUE
    ELSE
        is_last_error = itr_trans.SQLErrText
        RETURN FALSE
    END IF
END FUNCTION
```

**함수 of_disconnect**:
```powerbuilder
FUNCTION Boolean of_disconnect ()
    DISCONNECT USING itr_trans;
    IF itr_trans.SQLCode = 0 THEN
        ib_connected = FALSE
        RETURN TRUE
    ELSE
        is_last_error = itr_trans.SQLErrText
        RETURN FALSE
    END IF
END FUNCTION
```

**Destructor**:
```powerbuilder
IF ib_connected THEN
    of_disconnect()
END IF
```

**사용**:
```powerbuilder
n_db_manager lnv_db
lnv_db = CREATE n_db_manager
IF lnv_db.of_connect("ODBC", "", "MyDB", "sa", "1234") THEN
    // 데이터베이스 작업
ELSE
    MessageBox("오류", lnv_db.is_last_error)
END IF
DESTROY lnv_db
```

#### 예제 2: 고객 관리 비즈니스 로직 클래스

**n_customer_manager** 클래스:
- 고객 데이터 조회, 저장, 삭제 로직 포함
- 내부적으로 DataStore 사용

**인스턴스 변수**:
```powerbuilder
PRIVATE:
    DataStore ids_cust
    Transaction itr_trans
```

**Constructor**:
```powerbuilder
itr_trans = SQLCA
ids_cust = CREATE DataStore
ids_cust.DataObject = "d_customer"
ids_cust.SetTransObject(itr_trans)
```

**함수 of_get_customer**:
```powerbuilder
FUNCTION Long of_get_customer (String as_cust_id)
    RETURN ids_cust.Retrieve(as_cust_id)
END FUNCTION
```

**함수 of_save_customer**:
```powerbuilder
FUNCTION Boolean of_save_customer ()
    IF ids_cust.Update() = 1 THEN
        COMMIT USING itr_trans;
        RETURN TRUE
    ELSE
        ROLLBACK USING itr_trans;
        RETURN FALSE
    END IF
END FUNCTION
```

**함수 of_get_data (DataStore 반환)**:
```powerbuilder
FUNCTION DataStore of_get_data ()
    RETURN ids_cust
END FUNCTION
```

**Destructor**:
```powerbuilder
DESTROY ids_cust
```

**윈도우에서 사용**:
```powerbuilder
n_customer_manager lnv_cust
lnv_cust = CREATE n_customer_manager

lnv_cust.of_get_customer("C001")

// DataWindow와 데이터 공유 (옵션)
dw_list.ShareData(lnv_cust.of_get_data())

// 저장 버튼
IF lnv_cust.of_save_customer() THEN
    MessageBox("성공", "저장되었습니다.")
ELSE
    MessageBox("오류", "저장 실패")
END IF

DESTROY lnv_cust
```

### 2.7 실무 팁

- **싱글톤 패턴**: 애플리케이션 전체에서 하나의 인스턴스만 필요한 클래스는 전역 변수로 선언하고, 필요 시 생성하여 공유합니다.
- **객체 풀링**: 자주 생성/소멸되는 객체는 풀링 기법을 고려할 수 있습니다.
- **인터페이스 분리**: 너무 큰 클래스보다는 작은 단위의 책임을 가진 여러 클래스로 분리하는 것이 유지보수에 유리합니다.
- **오류 처리 일관화**: 모든 메서드에서 오류 발생 시 일관된 방식(예: is_last_error 설정)으로 처리합니다.

---

## 3. MDI와 사용자 정의 클래스를 결합한 아키텍처

MDI 애플리케이션에서 사용자 정의 클래스를 활용하면 각 시트가 공통 비즈니스 로직을 공유하거나, 데이터 접근 계층을 분리할 수 있습니다. 예를 들어, 모든 시트에서 사용할 **공통 데이터 관리 객체**를 프레임 윈도우의 인스턴스 변수로 선언하고, 각 시트가 이를 참조하여 사용하는 방식입니다.

**예**:
- 프레임 윈도우에 `n_db_manager` 타입의 인스턴스 변수 `inv_db` 선언
- 프레임 오픈 시 `inv_db` 생성 및 연결
- 각 시트를 열 때 `inv_db`를 시트의 공용 객체로 전달 (또는 전역 변수로 접근)

```powerbuilder
// w_frame의 인스턴스 변수
n_db_manager inv_db

// w_frame의 Open 이벤트
inv_db = CREATE n_db_manager
inv_db.of_connect(...)

// 시트를 열 때 inv_db 참조를 전달 (시트의 인스턴스 변수로 받도록 설계)
OpenSheetWithParm(w_customer, inv_db, w_frame)

// w_customer의 Open 이벤트
n_db_manager lnv_db
lnv_db = Message.PowerObjectParm   // 객체 참조 수신
// 이제 lnv_db를 사용하여 데이터베이스 작업
```

이러한 아키텍처는 **관심사의 분리**를 명확히 하고, 코드 중복을 줄이며, 유지보수성을 향상시킵니다.

---

## 결론

MDI 구조는 여러 문서를 동시에 다뤄야 하는 애플리케이션에 적합한 사용자 인터페이스 패턴이며, PowerBuilder는 `mdi!`와 `mdihelp!` 윈도우 타입과 `OpenSheet()` 함수를 통해 이를 완벽하게 지원합니다. MDI 프레임 윈도우는 반드시 메뉴가 연결되어야 하며, 시트 순회는 `GetFirstSheet()`와 `GetNextSheet()` 함수를 사용해야 합니다.

사용자 정의 클래스는 비즈니스 로직과 데이터 처리를 객체 지향적으로 캡슐화하는 강력한 도구로, 코드의 재사용성과 유지보수성을 크게 향상시킵니다.

이 두 가지 개념을 결합하면 확장 가능하고 견고한 엔터프라이즈 애플리케이션을 구축할 수 있습니다. MDI 프레임이 전체 UI를 관리하고, 사용자 정의 클래스들이 백엔드 비즈니스 로직을 담당하는 구조는 전형적인 PowerBuilder 애플리케이션 아키텍처의 모범 사례 중 하나입니다.

다음 글에서는 PowerBuilder의 **배포(Deployment)와 패키징**에 대해 알아보겠습니다.