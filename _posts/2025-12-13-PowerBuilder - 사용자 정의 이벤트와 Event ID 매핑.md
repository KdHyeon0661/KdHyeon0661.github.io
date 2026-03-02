---
layout: post
title: PowerBuilder - 사용자 정의 이벤트와 Event ID 매핑
date: 2025-12-13 14:30:23 +0900
category: PowerBuilder
---
# 사용자 정의 이벤트(Custom Event)와 Event ID 매핑

PowerBuilder는 객체 지향 이벤트 기반 환경에서 풍부한 내장 이벤트를 제공하지만, 때로는 표준 이벤트만으로는 특정 상황을 처리하기 어렵습니다. 이때 **사용자 정의 이벤트(Custom Event)**를 활용하면 개발자가 원하는 시점에 고유한 이벤트를 발생시킬 수 있습니다. 특히 **Event ID 매핑**을 통해 윈도우 메시지(예: 키보드 입력, 마우스 동작 등)를 사용자 정의 이벤트와 연결하면, 저수준의 시스템 이벤트도 쉽게 처리할 수 있습니다. 이 글에서는 사용자 정의 이벤트의 개념, Event ID 매핑 방법, 그리고 실제 CRUD 화면에서 단축키를 구현하는 예제를 자세히 알아보겠습니다.

---

## 1. 사용자 정의 이벤트란?

사용자 정의 이벤트는 개발자가 필요에 따라 객체(윈도우, 컨트롤, 사용자 객체)에 추가하는 이벤트입니다. 내장 이벤트(예: `Clicked`, `Open`)와 달리 이름과 동작을 개발자가 직접 정의할 수 있습니다.

### 사용자 정의 이벤트가 필요한 상황

- 특정 조건에서만 발생시켜야 하는 비즈니스 로직 (예: 데이터 검증 후 "데이터 검증 완료" 이벤트)
- 여러 객체에서 공통으로 처리해야 하는 동작 (예: 모든 윈도우에 "새로고침" 이벤트 추가)
- 윈도우 메시지(키보드, 마우스 등)를 가로채서 처리해야 할 때 (예: F2 키를 누르면 수정 모드 진입)
- DataWindow 내부에서 발생하는 세부 이벤트(예: 셀 단위 키 입력)를 캐치해야 할 때

---

## 2. 사용자 정의 이벤트 생성 방법

### 2.1 이벤트 선언

1. 윈도우나 컨트롤의 Painter를 엽니다.
2. 메뉴에서 **Declare → User Events**를 선택하거나, System Tree에서 객체를 우클릭하고 **Edit → User Events**를 선택합니다.
3. **User Events** 대화상자가 열리면 다음 정보를 입력합니다.

```
[User Events 대화상자]
┌─────────────────────────────────────┐
│ Event Name      | Event ID          │
│─────────────────────────────────────│
│ ue_refresh      | (None)            │
│ ue_keydown      | pbm_keydown       │
│ ue_custom_01    | pbm_custom01      │
└─────────────────────────────────────┘
```

- **Event Name**: 사용자 정의 이벤트 이름 (보통 `ue_` 접두사 사용)
- **Event ID**: 이벤트에 매핑할 윈도우 메시지 ID (선택 사항, 매핑하지 않으면 수동으로 트리거해야 함)

> **⚠️ 주의**: 하나의 객체 내에서 동일한 Event ID(예: `pbm_keydown`)를 여러 User Event에 중복해서 매핑할 수 없습니다. 따라서 키보드 입력처럼 다양한 키를 처리해야 한다면, 하나의 통합 이벤트(예: `ue_keydown`)에만 매핑하고 내부에서 키 코드에 따라 분기해야 합니다.

### 2.2 Event ID란?

Event ID는 PowerBuilder가 인식하는 미리 정의된 윈도우 메시지 식별자입니다. 예를 들어, `pbm_keydown`은 키보드 키가 눌렸을 때 발생하는 윈도우 메시지에 해당합니다. Event ID를 사용자 정의 이벤트에 매핑하면, 해당 메시지가 발생할 때마다 자동으로 사용자 정의 이벤트가 트리거됩니다.

### 2.3 주요 Event ID 종류

| Event ID | 설명 |
|----------|------|
| **pbm_keydown** | 키보드 키를 눌렀을 때 발생 (인자: key, keyflags) |
| **pbm_keyup** | 키보드 키를 놓았을 때 발생 |
| **pbm_char** | 키 입력 문자가 전달될 때 발생 |
| **pbm_mousemove** | 마우스가 움직일 때 발생 |
| **pbm_lbuttondown** | 마우스 왼쪽 버튼을 눌렀을 때 |
| **pbm_lbuttonup** | 마우스 왼쪽 버튼을 놓았을 때 |
| **pbm_rbuttondown** | 마우스 오른쪽 버튼을 눌렀을 때 |
| **pbm_timer** | 타이머 메시지 (Timer 함수로 설정) |
| **pbm_paint** | 화면 다시 그리기 필요 시 |
| **pbm_size** | 윈도우 크기 변경 시 |
| **pbm_dwnkey** | DataWindow에서 키 입력 시 (DataWindow 전용) |
| **pbm_dwnprocessenter** | DataWindow에서 Enter 키 처리 시 |
| **pbm_custom01 ~ pbm_custom15** | 사용자 정의 메시지를 위한 예약 ID |

#### DataWindow 관련 주요 Event ID

| Event ID | 설명 |
|----------|------|
| **pbm_dwnkey** | DataWindow에서 키가 눌릴 때 발생 (컬럼 편집 중) |
| **pbm_dwnprocessenter** | DataWindow에서 Enter 키가 눌렸을 때 발생 |
| **pbm_dwnitemchange** | DataWindow 아이템이 변경될 때 |
| **pbm_dwnbuttonclicking** | DataWindow 버튼 컬럼 클릭 시 |
| **pbm_dwnscroll** | DataWindow 스크롤 시 |

### 2.4 Event ID 매핑 예시

**예제 1: 윈도우에서 F2 키 눌림 처리**
- 윈도우의 User Events에서 이벤트 이름 `ue_keydown`을 추가하고 Event ID를 `pbm_keydown`으로 설정합니다.
- `ue_keydown` 이벤트 스크립트에서 전달된 `key` 인자를 확인하여 F2 키일 때 원하는 동작을 수행합니다.

**예제 2: DataWindow에서 Enter 키 처리**
- DataWindow 컨트롤의 User Events에서 `ue_enter`를 추가하고 Event ID를 `pbm_dwnprocessenter`로 설정합니다.
- `ue_enter` 이벤트에 Enter 키 처리 로직(예: 다음 컬럼 이동)을 작성합니다.

---

## 3. 사용자 정의 이벤트 호출 방법

Event ID에 매핑하지 않은 사용자 정의 이벤트는 코드에서 명시적으로 호출해야 합니다.

### 3.1 TriggerEvent

```powerbuilder
// 동기적 호출: 이벤트 처리가 끝날 때까지 기다림
Parent.TriggerEvent("ue_refresh")
```

### 3.2 PostEvent

```powerbuilder
// 비동기적 호출: 메시지 큐에 넣고 즉시 다음 줄 실행
Parent.PostEvent("ue_long_task")
```

### 3.3 인자 전달

이벤트에 인자를 전달하려면 `TriggerEvent`나 `PostEvent`와 함께 `Message` 객체를 사용할 수 있습니다.

```powerbuilder
// 이벤트 트리거 전에 Message 객체에 값 설정
Message.StringParm = "추가 데이터"
Parent.TriggerEvent("ue_custom")

// 이벤트 내에서 수신
String ls_data = Message.StringParm
```

---

## 4. 실전 예제: CRUD 화면에서 단축키 구현

간단한 고객 관리 CRUD 화면을 예로 들어, 단축키와 윈도우 메시지 이벤트 처리를 구현해 보겠습니다.

### 4.1 화면 구성

- 윈도우: `w_customer`
- DataWindow 컨트롤: `dw_list` (고객 목록)
- 버튼: `cb_add` (추가), `cb_modify` (수정), `cb_delete` (삭제), `cb_save` (저장), `cb_search` (조회)

### 4.2 단축키 요구사항

- **F2**: 선택된 고객 수정 모드 (cb_modify 클릭 효과)
- **F3**: 조회 실행 (cb_search 클릭 효과)
- **Ctrl+S**: 저장 실행 (cb_save 클릭 효과)
- **Enter**: DataWindow에서 다음 컬럼으로 이동 (기본 동작 대신 사용자 정의 처리)

### 4.3 사용자 정의 이벤트 생성

**윈도우 `w_customer`의 User Events**

| Event Name | Event ID | 설명 |
|------------|----------|------|
| `ue_keydown` | `pbm_keydown` | 윈도우의 모든 키 입력을 통합 처리 |

**DataWindow 컨트롤 `dw_list`의 User Events**

| Event Name | Event ID | 설명 |
|------------|----------|------|
| `ue_enter` | `pbm_dwnprocessenter` | Enter 키 처리 |

### 4.4 이벤트 스크립트 작성

#### 윈도우의 `ue_keydown` 이벤트 (pbm_keydown 매핑)

`pbm_keydown` 이벤트는 기본적으로 `key`와 `keyflags`라는 두 개의 인자를 제공합니다. `key`는 눌린 키의 코드이며, `keyflags`는 Shift, Ctrl 등의 상태를 나타냅니다. 이를 활용하여 단축키를 분기합니다.

```powerbuilder
// ue_keydown 이벤트 (pbm_keydown 매핑)
// 인자: key (UnsignedLong), keyflags (UnsignedLong)

CHOOSE CASE key
   CASE KeyF2!
      cb_modify.TriggerEvent(Clicked!)
   CASE KeyF3!
      cb_search.TriggerEvent(Clicked!)
   CASE KeyS!
      // Ctrl 키가 함께 눌렸는지 확인
      IF KeyControlDown() THEN
         cb_save.TriggerEvent(Clicked!)
      END IF
END CHOOSE
```

#### DataWindow의 `ue_enter` 이벤트 (pbm_dwnprocessenter 매핑)

```powerbuilder
// DataWindow에서 Enter 키 처리
// 다음 컬럼으로 이동하고, 마지막 컬럼이면 다음 행으로 이동

Integer li_row, li_col, li_cols
li_row = dw_list.GetRow()
li_col = dw_list.GetColumn()

// 다음 컬럼으로 이동 시도
IF dw_list.SetColumn(li_col + 1) < 0 THEN
   // 더 이상 컬럼이 없으면 다음 행으로
   dw_list.SetRow(li_row + 1)
   dw_list.SetColumn(1)
END IF

RETURN 1   // 1을 반환하면 기본 Enter 처리(행 이동)를 막음
```

#### 조회 버튼 `cb_search`의 Clicked 이벤트

```powerbuilder
// DataWindow 조회
dw_list.SetTransObject(SQLCA)
dw_list.Retrieve()
```

#### 추가/수정/삭제/저장 버튼 이벤트는 생략 (일반적인 CRUD 로직)

### 4.5 윈도우 Open 이벤트에서 포커스 설정

```powerbuilder
// 초기 포커스를 DataWindow로 설정
dw_list.SetFocus()
```

---

## 5. 고급 활용: pbm_custom01 ~ pbm_custom15

`pbm_custom01`부터 `pbm_custom15`까지의 Event ID는 사용자 정의 메시지를 위해 예약되어 있습니다. 이를 활용하면 다른 애플리케이션이나 외부 DLL에서 보내는 윈도우 메시지를 처리할 수 있습니다.

### 예제: 외부에서 보낸 사용자 메시지 처리

1. 윈도우의 User Events에서 `ue_external_msg`를 추가하고 Event ID를 `pbm_custom01`로 설정합니다.
2. `ue_external_msg` 이벤트에 스크립트를 작성합니다.
3. 다른 애플리케이션에서 `SendMessage` API로 해당 윈도우에 `WM_USER+1` (pbm_custom01에 해당) 메시지를 보내면 이 이벤트가 실행됩니다.

```powerbuilder
// ue_external_msg 이벤트
MessageBox("외부 메시지", "사용자 정의 메시지를 수신했습니다.")
```

---

## 6. 목표 달성: 간단한 CRUD 화면에서 단축키 처리 전체 구조

```
[w_customer] (Main 윈도우)
   ├─ [dw_list] (DataWindow)
   │     └─ ue_enter (pbm_dwnprocessenter)
   ├─ [cb_search] (조회 버튼)
   ├─ [cb_add] (추가 버튼)
   ├─ [cb_modify] (수정 버튼)
   ├─ [cb_delete] (삭제 버튼)
   └─ [cb_save] (저장 버튼)

[User Events]
   w_customer:
      ue_keydown (pbm_keydown)  // 모든 키 입력 통합 처리
   dw_list:
      ue_enter (pbm_dwnprocessenter)
```

### 실행 흐름

1. 윈도우가 열리면 Open 이벤트에서 dw_list에 포커스 설정.
2. 사용자가 dw_list에서 Enter 키를 누르면 → dw_list의 `ue_enter` 이벤트 실행 → 다음 컬럼/행 이동.
3. 사용자가 F2 키를 누르면 → 윈도우의 `ue_keydown` 이벤트 실행 → key = KeyF2! 확인 후 cb_modify 클릭 트리거.
4. 사용자가 F3 키를 누르면 → 윈도우의 `ue_keydown` 이벤트 실행 → key = KeyF3! 확인 후 cb_search 클릭 트리거.
5. 사용자가 Ctrl+S를 누르면 → 윈도우의 `ue_keydown` 이벤트 실행 → key = KeyS! 이고 Ctrl 키가 눌렸는지 확인 후 cb_save 클릭 트리거.

---

## 7. 디버깅 및 주의사항

- **이벤트 중복 매핑 불가**: 파워빌더에서는 하나의 객체 내에서 동일한 Event ID(예: `pbm_keydown`)를 여러 User Event에 중복해서 매핑할 수 없습니다. 따라서 키보드 이벤트를 처리할 때는 반드시 하나의 통합 이벤트(예: `ue_keydown`)만 만들고 내부에서 분기해야 합니다.
- **기본 동작 방지**: DataWindow에서 Enter 키 처리 시 `RETURN 1`을 반환하면 기본 동작(행 이동)을 막을 수 있습니다. 필요한 경우에만 사용하세요.
- **Message 객체 사용 시**: 인자를 전달할 때는 `Message` 객체가 전역이므로, 이벤트 처리 중에 다른 이벤트가 끼어들면 값이 덮어쓰일 수 있습니다. 가능하면 함수 호출을 권장합니다.
- **Event ID의 범위**: `pbm_custom01~15`는 사용자 정의 메시지용이므로, 외부 메시지와 충돌하지 않도록 주의하세요.

---

## 결론

사용자 정의 이벤트와 Event ID 매핑은 PowerBuilder에서 유연하고 강력한 이벤트 처리 메커니즘을 제공합니다. 단순히 버튼 클릭 이상의 사용자 상호작용(단축키, 특수 키, 외부 메시지)을 구현할 수 있으며, 특히 DataWindow와 같은 복잡한 컨트롤에서 세부적인 제어가 가능해집니다.

CRUD 화면 예제에서 본 것처럼, 하나의 통합 키보드 이벤트(`pbm_keydown`)만 추가하고 내부에서 분기 처리하면 여러 단축키를 간결하게 구현할 수 있습니다. 이벤트 기반 프로그래밍의 핵심을 이해하고, 적절한 Event ID를 활용하면 더욱 직관적이고 효율적인 애플리케이션을 개발할 수 있습니다.
