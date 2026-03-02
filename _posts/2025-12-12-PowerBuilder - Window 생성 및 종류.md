---
layout: post
title: PowerBuilder - Window 생성 및 종류
date: 2025-12-12 21:30:23 +0900
category: PowerBuilder
---
# Window 생성 및 종류 (Main, Response 등)

PowerBuilder 애플리케이션의 사용자 인터페이스는 윈도우(Window) 객체를 기반으로 구성됩니다. 윈도우는 컨트롤을 배치하는 컨테이너이자, 사용자와 시스템 간의 상호작용이 이루어지는 핵심 공간입니다. 이 글에서는 윈도우의 개념부터 종류별 특징, 생성 방법, 주요 속성과 이벤트, 그리고 실전 활용 예제까지 자세히 살펴보겠습니다.

---

## 윈도우(Window)란?

윈도우는 PowerBuilder에서 화면에 표시되는 모든 시각적 요소의 기본 단위입니다. 윈도우 위에는 버튼, 텍스트 상자, DataWindow 등 다양한 컨트롤을 배치할 수 있으며, 윈도우 자체도 크기, 위치, 제목, 스타일 등의 속성을 가집니다. 각 윈도우는 고유한 이벤트(Open, Close, Resize 등)를 가지고 있어 사용자 동작이나 시스템 상태 변화에 반응할 수 있습니다.

---

## 윈도우의 종류

PowerBuilder는 윈도우의 용도와 동작 방식에 따라 여섯 가지 유형을 제공합니다. 각 유형은 `WindowType` 속성으로 지정하며, 애플리케이션 설계 목적에 맞게 선택해야 합니다.

| 윈도우 유형 | 설명 | WindowType 값 |
|------------|------|---------------|
| **Main** | 독립적인 최상위 윈도우, 다른 윈도우와 종속 관계 없음 | main! |
| **Popup** | 부모 윈도우에 종속되지만 부모 밖으로 나올 수 있음, 부모가 최소화되면 함께 최소화 | popup! |
| **Response** | 모달(modal) 윈도우, 사용자가 반드시 응답해야 하며 부모 윈도우는 입력 대기 | response! |
| **Child** | 부모 윈도우 내에서만 존재하며 부모 내에서만 이동 가능, 부모와 함께 이동/크기변경 | child! |
| **MDI Frame** | 여러 개의 MDI 시트를 포함할 수 있는 프레임 (메뉴, 툴바 포함) | mdi! |
| **MDI Frame with MicroHelp** | MDI Frame에 상태 표시줄(마이크로헬프)이 추가된 형태 | mdihelp! |

각 유형의 특징을 자세히 알아보겠습니다.

### 1. Main 윈도우

가장 기본적인 윈도우 형태입니다. 다른 윈도우와 종속 관계 없이 독립적으로 존재하며, 작업 표시줄에 별도의 버튼이 표시됩니다. 일반적인 애플리케이션의 메인 화면으로 사용됩니다.

- **특징**: 
  - 최상위 레벨 윈도우
  - 다른 윈도우의 영향을 받지 않음
  - 여러 개의 Main 윈도우를 동시에 띄울 수 있음
- **용도**: 애플리케이션의 주 화면, 독립적인 기능 창

### 2. Popup 윈도우

부모 윈도우에 종속되지만, 부모 윈도우 영역 밖으로 나올 수 있는 윈도우입니다. 부모 윈도우가 최소화되면 팝업 윈도우도 함께 최소화되며, 부모가 닫히면 팝업도 닫힙니다. 주로 도움말 창, 찾기/바꾸기 대화상자 등에 사용됩니다.

- **특징**:
  - 항상 부모 윈도우 위에 표시되지는 않음 (일반적인 팝업)
  - 부모 윈도우와 독립적으로 사용자 입력 가능
- **용도**: 보조 기능 창, 도움말, 검색 창

### 3. Response 윈도우

사용자의 응답을 강제로 요구하는 모달(Modal) 윈도우입니다. Response 윈도우가 열리면 부모 윈도우는 모든 입력을 받지 못하고 대기 상태가 됩니다. 사용자가 Response 윈도우를 닫아야만 부모 윈도우로 돌아갈 수 있습니다. **또한, Response 윈도우를 여는 순간부터 닫힐 때까지 부모 윈도우의 스크립트 실행은 완전히 일시 중지(Blocking)됩니다.**

- **특징**:
  - 애플리케이션 실행 흐름을 일시 중단
  - 중요한 확인이나 입력이 필요할 때 사용
  - 일반적으로 크기가 작고 버튼(확인, 취소)이 포함됨
- **용도**: 로그인 창, 메시지 박스, 설정 저장 확인

### 4. Child 윈도우

부모 윈도우 내에서만 존재하는 윈도우입니다. 부모 윈도우의 클라이언트 영역을 벗어날 수 없으며, 부모 윈도우가 이동하거나 크기가 변경되면 자식 윈도우도 함께 영향을 받습니다.

- **특징**:
  - 부모 윈도우의 일부처럼 동작
  - 제목 표시줄이 있을 수 있지만, 독립적인 작업 표시줄 항목은 없음
  - 주로 MDI 환경에서 사용되나, 일반 윈도우에서도 제한적으로 사용 가능
- **용도**: MDI 시트, 부모 내부의 보조 창

### 5. MDI Frame 윈도우

Multiple Document Interface(MDI)를 구현하는 프레임 윈도우입니다. 메뉴, 툴바, 상태바 등을 포함할 수 있으며, 내부에 여러 개의 MDI 시트(MDI Sheet)를 띄울 수 있습니다. MDI 시트는 일반적으로 Main 윈도우이지만 MDI Frame에 의해 관리됩니다.

- **특징**:
  - 클라이언트 영역에 여러 개의 시트를 배치 가능
  - 시트는 프레임 내에서만 이동/크기조절 가능
  - 프레임 메뉴가 시트에 따라 변경될 수 있음 (메뉴 병합)
- **용도**: 문서 편집기, 통합 개발 환경, 다중 작업이 필요한 프로그램

### 6. MDI Frame with MicroHelp

MDI Frame에 상태 표시줄(마이크로헬프)이 추가된 형태입니다. 마이크로헬프는 일반적으로 프레임 하단에 위치하며, 메뉴 항목 설명이나 현재 상태를 표시하는 데 사용됩니다.

- **특징**:
  - `mdihelp!` 타입 사용
  - `SetMicroHelp()` 함수로 상태 표시줄 텍스트 변경 가능
- **용도**: 도움말 메시지가 필요한 MDI 애플리케이션

### 윈도우 유형 간 관계 다이어그램

```
[Main] <-- 부모 -- [Popup]
   |
   | 부모
   ▼
[MDI Frame] <-- 포함 -- [MDI Sheet (Main)]   // MDI 시트는 보통 Main 윈도우
   |
   | (MDI Frame 내부)
   ▼
[Child]  // MDI 시트로 사용되기도 함
```

Response 윈도우는 독립적이지만 부모가 지정될 수 있으며, 부모는 입력 대기 상태가 됩니다.

---

## 윈도우 생성 방법

### 1. Window Painter를 이용한 시각적 생성

PowerBuilder IDE에서 새로운 윈도우를 생성하는 가장 일반적인 방법입니다.

1. **File → New** (또는 도구 모음 New 아이콘)
2. **PB Object** 탭에서 **Window** 아이콘 더블클릭
3. 빈 윈도우가 열리면 원하는 컨트롤을 배치하고 속성을 설정합니다.
4. 윈도우의 속성 시트(Properties)에서 **WindowType**을 선택합니다.
5. 이벤트 탭에서 필요한 이벤트(Open, Close, Clicked 등)에 스크립트를 작성합니다.
6. 저장할 때는 일반적으로 `w_` 접두사를 사용합니다. (예: `w_main`, `w_login`)

> **📸 추천 스크린샷 위치**: 이 부분에서 Window Painter 화면과 속성 창에서 WindowType을 선택하는 드롭다운 메뉴를 캡처하여 첨부하면 초보자에게 큰 도움이 됩니다.

### 2. 코드를 통한 윈도우 열기

디자인한 윈도우를 실행 중에 열려면 `Open()` 함수를 사용합니다.

#### 기본 형식
```
Open ( windowvar {, parent } )
```

- `windowvar`: 열려는 윈도우 객체 변수 (보통 윈도우 이름 자체를 사용)
- `parent`: (선택) Popup, Response, Child 윈도우의 경우 부모 윈도우 지정

#### 예제
```
// Main 윈도우 열기
Open(w_main)

// Popup 윈도우 열기 (부모: w_main)
Open(w_popup, w_main)

// Response 윈도우 열기 (부모 생략 가능, 생략하면 활성 윈도우가 부모)
Open(w_login)
```

#### MDI 시트 열기
MDI 프레임에서 시트를 열 때는 `OpenSheet()` 함수를 사용합니다.

```
OpenSheet( sheetref, mdiframe {, position {, arrangeopen } } )
```

- `sheetref`: 열 시트 윈도우
- `mdiframe`: MDI 프레임 윈도우
- `position`: 시트가 표시될 위치 (선택)
- `arrangeopen`: 시트 배열 방식 (선택)

예:
```
OpenSheet(w_customer, w_frame)
```

> **📸 추천 스크린샷 위치**: MDI 프레임과 그 안에 여러 시트가 열린 모습을 캡처하여 이 부분에 첨부하면 MDI 개념을 직관적으로 이해하는 데 도움이 됩니다.

### 3. 윈도우 닫기

열린 윈도우는 `Close()` 함수로 닫습니다.

```
Close ( w_main )
```

Response 윈도우에서 닫을 때는 `Close()` 호출 시점에 부모 윈도우로 제어가 돌아갑니다.

---

## 윈도우의 주요 속성

윈도우는 다양한 속성을 통해 외관과 동작을 제어할 수 있습니다. 주요 속성은 다음과 같습니다.

| 속성 | 설명 | 예시 값 |
|------|------|---------|
| `Title` | 윈도우 제목 표시줄 문자열 | `"고객 관리"` |
| `WindowType` | 윈도우 유형 | `main!`, `response!` 등 |
| `Visible` | 윈도우 표시 여부 | `TRUE` / `FALSE` |
| `Enabled` | 윈도우 활성화 여부 | `TRUE` / `FALSE` |
| `Border` | 테두리 스타일 | `TRUE` / `FALSE` |
| `ControlMenu` | 시스템 메뉴 표시 여부 | `TRUE` / `FALSE` |
| `MaxBox` | 최대화 버튼 표시 여부 | `TRUE` / `FALSE` |
| `MinBox` | 최소화 버튼 표시 여부 | `TRUE` / `FALSE` |
| `Resizable` | 사용자 크기 조절 가능 여부 | `TRUE` / `FALSE` |
| `Center` | 윈도우 열릴 때 화면 중앙에 위치 | `TRUE` / `FALSE` |
| `BackColor` | 배경색 (RGB 값 또는 설정) | `RGB(255,255,255)` |
| `Icon` | 윈도우 아이콘 | `"MyApp.ico"` |

속성은 코드에서도 변경 가능합니다.

```
w_main.Title = "새 제목"
w_main.BackColor = RGB(240, 240, 240)
```

---

## 윈도우의 주요 이벤트

윈도우는 여러 이벤트를 가지고 있으며, 각 이벤트에 스크립트를 작성하여 특정 시점에 원하는 동작을 수행할 수 있습니다.

| 이벤트 | 발생 시점 | 주요 용도 |
|--------|----------|----------|
| **Open** | 윈도우가 열릴 때 | 초기화 작업 (데이터베이스 연결, 컨트롤 초기값 설정) |
| **Close** | 윈도우가 닫힐 때 | 정리 작업 (트랜잭션 해제, 임시 파일 삭제) |
| **CloseQuery** | 윈도우가 닫히기 직전에 발생 | 닫기 전 확인 메시지, 닫기 취소 가능 |
| **Resize** | 윈도우 크기가 변경될 때 | 컨트롤 위치/크기 재조정 |
| **Timer** | 타이머 설정 시 일정 간격으로 발생 | 주기적 작업 (실시간 갱신) |
| **Activate** / **Deactivate** | 윈도우가 활성화/비활성화될 때 | 포커스 관련 처리 |
| **Key** | 키보드 입력이 있을 때 | 단축키 처리 |

#### CloseQuery 예제
```
// 윈도우 CloseQuery 이벤트
Integer li_rtn
li_rtn = MessageBox("확인", "저장하지 않은 데이터가 있습니다. 종료하시겠습니까?", Question!, YesNo!)
IF li_rtn = 2 THEN  // No 선택 시
   RETURN 1  // 1을 반환하면 닫기가 취소됨
END IF
```

---

## 윈도우 간 데이터 전달

윈도우 간에 데이터를 전달해야 하는 경우가 많습니다. PowerBuilder는 여러 가지 방법을 제공합니다.

### 1. OpenWithParm 함수

`OpenWithParm()` 함수를 사용하면 윈도우를 열면서 매개변수를 전달할 수 있습니다.

**문법**:
```
OpenWithParm ( windowvar, parameter {, parent } )
```

- `parameter`: 전달할 데이터 (String, Numeric, PowerObject 등)

**전달 예**:
```
// 메인 윈도우에서 상세 윈도우 열기
OpenWithParm(w_detail, "C001")
```

**수신 예 (w_detail의 Open 이벤트)**:
```
String ls_cust_id
ls_cust_id = Message.StringParm   // 문자열 파라미터 수신
// 또는 Message.DoubleParm (숫자), Message.PowerObjectParm (객체)
```

### 2. 전역 변수 사용

전역 변수를 선언하여 값을 공유할 수 있지만, 캡슐화를 해치므로 신중히 사용해야 합니다.

### 3. 공유 객체 또는 사용자 객체

복잡한 데이터 교환은 비시각적 사용자 객체를 만들어 참조를 전달하는 방법이 좋습니다.

> **💡 실무 팁: 여러 개의 값을 전달하고 싶다면?**  
> `OpenWithParm`은 단일 값만 전달할 수 있습니다. 2개 이상의 값을 전달하려면 앞서 배운 **구조체(Structure)**나 **비시각적 객체(NVO)**에 값들을 담은 뒤, 객체 자체를 넘기고 `Message.PowerObjectParm`으로 받아야 합니다.
> 
> ```powerbuilder
> // 보내는 쪽 (w_main)
> s_customer_param lstr_param
> lstr_param.cust_id = "C001"
> lstr_param.cust_name = "홍길동"
> OpenWithParm(w_detail, lstr_param)
> 
> // 받는 쪽 (w_detail의 Open 이벤트)
> s_customer_param lstr_receive
> lstr_receive = Message.PowerObjectParm
> ```

---

## 실전 예제

### 예제 1: Main 윈도우와 Response 윈도우(로그인) 구현 및 데이터 통신

**w_main (Main 윈도우)의 로그인 버튼 Clicked 이벤트**:
```powerbuilder
// 1. Response 윈도우를 열며 파라미터 전달
OpenWithParm(w_login, "환영합니다! 아이디를 입력하세요.")

// 2. Response 윈도우가 열려 있는 동안 아래 코드는 실행되지 않고 대기(Blocking)함
// 3. w_login이 닫히면 비로소 아래 코드가 실행되며, 반환된 값을 Message 객체에서 꺼냄
String ls_return_id
ls_return_id = Message.StringParm

IF ls_return_id <> "" THEN
    MessageBox("로그인 성공", ls_return_id + "님 환영합니다.")
ELSE
    MessageBox("로그인 취소", "로그인이 취소되었습니다.")
END IF
```

**w_login (Response 윈도우)의 Open 이벤트**:
```powerbuilder
// 전달받은 메시지 표시
st_message.Text = Message.StringParm
```

**w_login의 확인 버튼 Clicked 이벤트**:
```powerbuilder
// 입력값을 부모 윈도우로 안전하게 반환하며 창 닫기
CloseWithReturn(Parent, sle_user.Text)
```

**w_login의 취소 버튼 Clicked 이벤트**:
```powerbuilder
// 빈 값을 반환하며 창 닫기
CloseWithReturn(Parent, "")
```

### 예제 2: MDI Frame과 MDI Sheet

**w_frame (MDI Frame)**:
- WindowType = mdihelp!
- 메뉴: "파일" → "고객 목록"

**메뉴 Clicked 이벤트**:
```
OpenSheet(w_customer_list, w_frame)
```

**w_customer_list (MDI Sheet)**:
- WindowType = main! (MDI 프레임 내에서는 main!이어야 시트로 동작)
- DataWindow 배치

---

## 윈도우 상속

PowerBuilder는 윈도우 상속을 지원합니다. 공통 기능(예: 배경색, 로고, 종료 버튼)을 가진 기본 윈도우를 만들고, 이를 상속받아 자식 윈도우를 생성하면 일관된 디자인과 기능을 유지하면서 개발 생산성을 높일 수 있습니다.

**상속 윈도우 생성**:
1. 기존 윈도우를 우클릭 → **Inherit** 선택
2. 새 윈도우 이름 지정 (예: `w_base_main`을 상속받은 `w_customer_main`)

상속받은 윈도우에서는 부모의 컨트롤과 스크립트를 수정하거나 추가할 수 있으며, `CALL` 문을 사용하여 부모의 이벤트 스크립트를 호출할 수 있습니다.

---

## 윈도우 설계 시 고려사항

1. **윈도우 유형 선택**: 용도에 맞게 Main, Response, MDI 등을 선택하세요. 잘못 선택하면 사용자 경험이 나빠질 수 있습니다.
2. **일관성 유지**: 상속을 활용하여 공통 디자인과 기능을 재사용하세요.
3. **메모리 관리**: 필요 없어진 윈도우는 반드시 `Close`하여 리소스를 해제하세요.
4. **응답성**: Response 윈도우는 꼭 필요한 경우에만 사용하고, 너무 자주 띄우지 마세요.
5. **데이터 전달**: `OpenWithParm`과 `CloseWithReturn`을 적절히 활용하세요.

> **실무 팁: Resize 이벤트 활용**  
> 사용자가 윈도우 크기를 마우스로 드래그해서 늘렸을 때, 화면 안의 DataWindow도 꽉 차게 같이 늘어나게 하려면 윈도우의 `Resize` 이벤트에 다음과 같이 코딩합니다.
> 
> ```powerbuilder
> // 윈도우의 Resize 이벤트
> // newwidth, newheight는 Resize 이벤트가 기본 제공하는 인수(Argument)입니다.
> dw_list.Resize(newwidth - 100, newheight - 150)
> ```

---

## 결론

PowerBuilder의 윈도우는 애플리케이션 UI의 핵심입니다. Main, Popup, Response, Child, MDI Frame 등 다양한 윈도우 유형을 이해하고 상황에 맞게 선택하면 효율적이고 사용자 친화적인 인터페이스를 구축할 수 있습니다. 윈도우의 속성과 이벤트를 잘 활용하고, 상속을 통해 재사용성을 높이면 유지보수하기 쉬운 애플리케이션을 개발할 수 있습니다.

특히 Response 윈도우는 부모 스크립트를 블로킹하고, 값을 반환할 때는 `CloseWithReturn`을 사용해야 한다는 점을 기억하세요. 또한 여러 값을 전달할 때는 구조체나 객체를 활용하는 것이 안전합니다.

다음 글에서는 윈도우에 배치하는 주요 컨트롤들의 사용법과 활용 예제를 다루어 보겠습니다.