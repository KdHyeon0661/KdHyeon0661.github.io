---
layout: post
title: PowerBuilder - 주요 컨트롤
date: 2025-12-12 22:30:23 +0900
category: PowerBuilder
---
# 주요 컨트롤 (CommandButton, EditMask, StaticText, CheckBox, RadioButton, DropDownListBox 등)

PowerBuilder에서 윈도우는 사용자 인터페이스의 기본 컨테이너이며, 실제 사용자와의 상호작용은 윈도우 위에 배치된 다양한 컨트롤들을 통해 이루어집니다. 컨트롤은 버튼, 텍스트 표시, 입력 필드, 선택 옵션 등 각각 고유한 역할을 가지며, 애플리케이션의 기능과 직관성을 결정짓는 중요한 요소입니다. 이 글에서는 PowerBuilder에서 가장 많이 사용되는 주요 컨트롤들의 특징, 속성, 이벤트, 그리고 활용 방법을 상세히 알아보겠습니다.

---

## 컨트롤의 공통 속성과 이벤트

모든 컨트롤은 기본적으로 공통된 속성과 이벤트를 가집니다. 이를 이해하면 컨트롤을 일관되게 다룰 수 있습니다.

### 공통 속성

| 속성 | 설명 | 예시 |
|------|------|------|
| `Name` | 컨트롤의 고유 식별자 | `cb_ok`, `sle_name` |
| `Text` | 컨트롤에 표시되는 문자열 | `"확인"`, `"이름"` |
| `Visible` | 컨트롤 표시 여부 (`TRUE`/`FALSE`) | `cb_ok.Visible = FALSE` |
| `Enabled` | 컨트롤 활성화 여부 (`TRUE`/`FALSE`) | `sle_name.Enabled = FALSE` |
| `Tag` | 개발자가 임의로 사용할 수 있는 추가 정보 저장 | `"user_id"` |
| `BackColor` | 배경색 (RGB 함수 사용) | `RGB(255,0,0)` |
| `Border` | 테두리 스타일 | `TRUE`, `FALSE` |
| `Width`, `Height` | 컨트롤의 너비와 높이 | `200`, `80` |
| `X`, `Y` | 부모 윈도우 내에서의 위치 좌표 | `100`, `50` |
| `Pointer` | 마우스 포인터 모양 | `HandPoint!`, `HourGlass!` |

### 공통 이벤트

| 이벤트 | 발생 시점 | 주요 용도 |
|--------|----------|----------|
| `Clicked` | 컨트롤을 클릭했을 때 | 버튼 클릭 처리, 항목 선택 등 |
| `DoubleClicked` | 더블클릭했을 때 | 상세 보기, 편집 모드 진입 등 |
| `RightClicked` | 마우스 오른쪽 버튼 클릭 | 컨텍스트 메뉴 표시 |
| `Modified` | 컨트롤의 내용이 변경되고 포커스를 잃을 때 | 입력값 검증, 자동 저장 등 |
| `GetFocus` | 컨트롤이 포커스를 받을 때 | 도움말 표시, 초기화 |
| `LoseFocus` | 컨트롤이 포커스를 잃을 때 | 입력 완료 처리 |
| `Key` | 키보드 입력이 있을 때 | 단축키, 입력 제한 |

---

## 주요 컨트롤 상세 설명

### 1. CommandButton (명령 버튼)

가장 기본적인 컨트롤로, 사용자의 클릭에 의해 특정 동작을 실행합니다.

#### 주요 속성

| 속성 | 설명 |
|------|------|
| `Default` | `TRUE`이면 Enter 키 입력 시 버튼 클릭으로 간주 (폼의 기본 버튼) |
| `Cancel` | `TRUE`이면 Esc 키 입력 시 버튼 클릭으로 간주 (취소 버튼) |
| `Picture` | 버튼에 표시할 이미지 파일 |
| `PictureName` | 내장 이미지 이름 (예: `Save!`, `Print!`) |

#### 주요 이벤트
- `Clicked`: 버튼이 클릭될 때 발생. 가장 많이 사용됨.

#### 예제
```
// cb_save의 Clicked 이벤트
IF sle_name.Text = "" THEN
   MessageBox("경고", "이름을 입력하세요.")
   RETURN
END IF

// 저장 로직 실행
dw_customer.Update()
```

---

### 2. StaticText (정적 텍스트)

사용자와 상호작용하지 않고 단순히 텍스트를 표시하는 컨트롤입니다. 레이블 역할을 합니다.

#### 주요 속성

| 속성 | 설명 |
|------|------|
| `Text` | 표시할 문자열 |
| `Alignment` | 텍스트 정렬 (Left!, Center!, Right!) |
| `BorderStyle` | 테두리 스타일 (None!, Box!, etc.) |
| `FocusRectangle` | 포커스 사각형 표시 여부 |

#### 주요 이벤트
- `Clicked`: 클릭 가능하도록 설정할 수 있지만, 주로 표시용이므로 잘 사용하지 않음.

#### 예제
```
st_message.Text = "현재 처리 중입니다..."
st_message.Visible = TRUE
```

---

### 3. SingleLineEdit / MultiLineEdit (한 줄 입력 / 여러 줄 입력)

사용자로부터 텍스트를 입력받는 컨트롤입니다.

#### SingleLineEdit
- 한 줄의 텍스트 입력
- 주요 속성: `Password` (TRUE이면 입력 문자를 *로 표시), `MaxLength` (최대 입력 길이), `DisplayOnly` (읽기 전용)

#### MultiLineEdit
- 여러 줄의 텍스트 입력
- 주요 속성: `VScrollBar` (수직 스크롤바), `HScrollBar` (수평 스크롤바), `Alignment`

#### 주요 이벤트
- `Modified`: 내용이 변경되고 포커스를 잃을 때
- `GetFocus` / `LoseFocus`

#### 예제
```
// sle_id의 Modified 이벤트 (입력 완료 시 검증)
IF Len(sle_id.Text) < 4 THEN
   MessageBox("오류", "ID는 4자 이상이어야 합니다.")
   sle_id.SetFocus()   // 다시 입력하도록 포커스 이동
END IF
```

---

### 4. EditMask (형식화된 입력)

입력 형식을 지정하여 사용자가 정해진 패턴대로만 입력할 수 있도록 하는 컨트롤입니다. 전화번호, 주민등록번호, 날짜 등 형식이 정해진 데이터를 입력받을 때 유용합니다.

#### 주요 속성

| 속성 | 설명 |
|------|------|
| `Mask` | 입력 형식 문자열 (예: `!(###) ###-####`, `##/##/####`) |
| `MaskDataType` | 데이터 타입 (String!, Date!, Time!, DateTime!, Numeric!) |
| `MaskDisp` | 마스크 문자를 화면에 표시할지 여부 |
| `Spin` | 스핀 컨트롤(증감 버튼) 사용 여부 |
| `Increment` / `Decrement` | 스핀 시 증감 값 |

#### 마스크 형식 문자

| 문자 | 의미 |
|------|------|
| `#` | 숫자만 입력 가능 |
| `!` | 대문자로 변환 |
| `^` | 소문자로 변환 |
| `x` | 모든 문자 허용 |
| `a` | 알파벳 문자만 허용 |
| `*` | 마스크 문자를 화면에 표시하지 않음 |

#### 예제
- 전화번호: `!(###) ###-####`
- 주민등록번호: `######-#######`
- 날짜: `##/##/####`

#### 주요 이벤트
- `Modified`: 입력 완료 시

#### 코드에서 값 읽기/쓰기
```
String ls_phone = em_phone.Text   // 입력된 텍스트 (마스크 포함)
Date ld_date = Date(em_date.Text) // 날짜로 변환
```

> **💡 실무 팁: 마스크 기호를 빼고 순수 데이터만 가져오려면?**  
> `em_phone.Text`를 사용하면 `(010) 1234-5678`처럼 마스크 기호가 포함된 문자열을 반환합니다. DB에 순수 숫자만 저장하고 싶다면 `GetData()` 함수를 사용해야 합니다.
> ```powerbuilder
> String ls_raw_phone
> em_phone.GetData(ls_raw_phone)   // ls_raw_phone = "01012345678"
> ```

---

### 5. CheckBox (체크박스)

사용자가 두 가지 상태(선택/해제) 중 하나를 선택할 수 있는 컨트롤입니다. 여러 개의 독립적인 옵션을 제공할 때 사용합니다.

#### 주요 속성

| 속성 | 설명 |
|------|------|
| `Checked` | 체크 여부 (`TRUE`/`FALSE`) |
| `ThreeState` | 3상태 지원 여부 (선택/해제/혼합) |
| `ThirdState` | 세 번째 상태일 때 `TRUE` |
| `LeftText` | 텍스트를 체크박스 왼쪽에 표시 |

#### 주요 이벤트
- `Clicked`: 상태가 변경될 때 발생

#### 예제
```
// cb_agree (약관 동의) Clicked 이벤트
IF This.Checked THEN
   cb_next.Enabled = TRUE
ELSE
   cb_next.Enabled = FALSE
END IF
```

#### 상태 확인
```
IF cb_option.Checked THEN
   // 옵션 선택됨
ELSE
   // 선택 안 됨
END IF
```

---

### 6. RadioButton (라디오 버튼)

여러 옵션 중 하나만 선택해야 할 때 사용합니다. 라디오 버튼은 일반적으로 **GroupBox** 안에 배치하여 그룹을 형성합니다. 같은 그룹 내에서는 하나만 선택 가능합니다.

#### 주요 속성

| 속성 | 설명 |
|------|------|
| `Checked` | 선택 여부 (`TRUE`/`FALSE`) |
| `Automatic` | 자동 그룹핑 (기본 `TRUE`, 그룹박스 없이도 같은 부모 내에서 자동 그룹) |
| `LeftText` | 텍스트를 버튼 왼쪽에 표시 |

#### 그룹박스(GroupBox)와 함께 사용

GroupBox는 시각적 그룹핑과 함께 라디오 버튼의 논리적 그룹을 형성합니다. 같은 GroupBox 내의 라디오 버튼은 자동으로 하나만 선택됩니다.

```
[GroupBox: 성별]
   (•) rb_male   남성
   ( ) rb_female 여성

[GroupBox: 연령대]
   ( ) rb_10s    10대
   (•) rb_20s    20대
   ( ) rb_30s    30대
```

#### 주요 이벤트
- `Clicked`: 선택될 때 발생

#### 예제
```
// rb_male Clicked 이벤트
IF This.Checked THEN
   st_gender.Text = "남성"
END IF

// 다른 라디오 버튼들은 자동으로 해제됨
```

#### 선택된 라디오 버튼 확인
```
String ls_gender
IF rb_male.Checked THEN
   ls_gender = "M"
ELSEIF rb_female.Checked THEN
   ls_gender = "F"
END IF
```

---

### 7. ListBox / DropDownListBox (리스트 박스 / 드롭다운 리스트)

여러 항목 중 하나(또는 다중)를 선택할 수 있는 컨트롤입니다.

#### ListBox
- 항상 목록이 펼쳐져 있음
- 주요 속성: `Sorted` (정렬), `MultiSelect` (다중 선택), `ExtendedSelect` (Shift/Ctrl 다중 선택)

#### DropDownListBox
- 평소에는 한 줄만 보이고 클릭 시 드롭다운 목록이 펼쳐짐
- 유형: `DropDownList` (선택만 가능, 직접 입력 불가), `DropDownEdit` (직접 입력 가능)

#### 주요 속성 (공통)

| 속성 | 설명 |
|------|------|
| `Items` | 항목 목록 (디자인 시 추가하거나 코드로 추가) |
| `Sorted` | 항목 정렬 여부 |
| `Text` | 현재 선택된 항목의 텍스트 |

#### 주요 이벤트
- `SelectionChanged`: 선택 항목이 변경될 때
- `DoubleClicked`: 더블클릭 시

#### 주요 함수

| 함수 | 설명 |
|------|------|
| `AddItem ( item )` | 항목 추가 |
| `InsertItem ( item, index )` | 특정 위치에 항목 삽입 |
| `DeleteItem ( index )` | 항목 삭제 |
| `Reset ()` | 모든 항목 삭제 |
| `FindItem ( text, index )` | 항목 검색 |
| `SelectItem ( index )` / `SelectItem ( text )` | 항목 선택 |
| `TotalItems ()` | 전체 항목 수 반환 |
| `SelectedItem ()` | 선택된 항목의 텍스트 반환 |
| `State ( index )` | 특정 인덱스의 선택 상태 반환 (0=선택안됨, 1=선택됨) |

#### 예제
```
// 리스트 박스에 항목 추가 (윈도우 오픈 이벤트)
lb_department.AddItem("영업부")
lb_department.AddItem("개발부")
lb_department.AddItem("인사부")
lb_department.AddItem("총무부")

// 선택 변경 시 (lb_department의 SelectionChanged 이벤트)
String ls_dept
ls_dept = lb_department.SelectedItem()
IF ls_dept <> "" THEN
   st_dept.Text = "선택: " + ls_dept
END IF

// 드롭다운 리스트에서 항목 가져오기
String ls_city = ddlb_city.Text
```

---

### 8. Picture (그림)

이미지를 표시하는 컨트롤입니다.

#### 주요 속성
- `PictureName`: 이미지 파일 경로 또는 내장 이미지 이름
- `OriginalSize`: 원본 크기 유지 여부
- `Invert`: 색상 반전

#### 주요 이벤트
- `Clicked`: 이미지 클릭 시 (아이콘 버튼으로 활용 가능)

---

### 9. Tab (탭 컨트롤)

여러 페이지를 탭으로 구분하여 표시하는 컨트롤입니다. 복잡한 정보를 체계적으로 구성할 때 사용합니다.

- 각 탭 페이지는 별도의 컨트롤들을 배치할 수 있는 컨테이너 역할을 합니다.
- 주요 속성: `TabPosition` (탭 위치), `SelectedTab` (현재 선택된 탭 인덱스)

---

## 컨트롤 이름 규칙 (접두사)

일관된 이름 규칙은 코드 가독성과 유지보수에 도움이 됩니다. 일반적으로 다음과 같은 접두사를 사용합니다.

| 컨트롤 | 접두사 | 예시 |
|--------|--------|------|
| CommandButton | `cb_` | `cb_save`, `cb_exit` |
| StaticText | `st_` | `st_name_label` |
| SingleLineEdit | `sle_` | `sle_userid` |
| MultiLineEdit | `mle_` | `mle_address` |
| EditMask | `em_` | `em_phone`, `em_birth` |
| CheckBox | `cbx_` 또는 `ck_` | `cbx_agree` |
| RadioButton | `rb_` | `rb_male`, `rb_female` |
| ListBox | `lb_` | `lb_department` |
| DropDownListBox | `ddlb_` | `ddlb_country` |
| Picture | `p_` | `p_logo` |
| DataWindow | `dw_` | `dw_customer` |
| GroupBox | `gb_` | `gb_gender` |
| Tab | `tab_` | `tab_main` |

---

## 컨트롤 배치와 관리 팁

### 1. 컨트롤 정렬 및 크기 조정
Window Painter의 도구 모음에는 여러 컨트롤을 정렬하거나 동일한 크기로 맞추는 버튼들이 있습니다.
- `Align Left`, `Align Right`, `Align Top`, `Align Bottom`
- `Make Same Width`, `Make Same Height`, `Make Same Size`

### 2. 탭 순서(Tab Order) 설정
Tab 키를 누를 때 포커스가 이동하는 순서를 지정할 수 있습니다. Window Painter에서 **Format → Tab Order** 메뉴를 선택하거나 `Ctrl+D`를 누르면 각 컨트롤에 빨간 숫자가 표시되며, 이를 클릭하여 순서를 변경할 수 있습니다.

> **📸 추천 스크린샷**: 탭 순서 편집 모드에서 빨간 숫자가 표시된 화면을 캡처하여 첨부하면 초보자도 쉽게 이해할 수 있습니다.

### 3. 컨트롤 배열 (Control Array)
PowerBuilder에서 동일한 이름의 컨트롤을 배열처럼 자동으로 묶어주는 기능은 없습니다. 하지만 윈도우의 `Control[]` 배열 속성을 이용하여 모든 컨트롤을 순회하면서 특정 이름 패턴이나 타입에 따라 일괄 처리할 수 있습니다.

예를 들어, 이름이 "cb_1", "cb_2", "cb_3"인 버튼들의 텍스트를 일괄 변경하려면 다음과 같이 작성합니다.

```powerbuilder
Integer i
CommandButton lcb_temp

FOR i = 1 TO UpperBound(Parent.Control)
    // 컨트롤 타입이 CommandButton인지 확인
    IF Parent.Control[i].TypeOf() = CommandButton! THEN
        // CommandButton으로 형변환
        lcb_temp = Parent.Control[i]
        // 이름이 "cb_"로 시작하는 경우에만 처리
        IF Left(lcb_temp.ClassName(), 3) = "cb_" THEN
            lcb_temp.Text = "일괄 변경"
        END IF
    END IF
NEXT
```

`Parent.Control`은 현재 윈도우에 배치된 모든 컨트롤의 배열입니다. `TypeOf()`로 컨트롤 유형을 확인한 후, 원하는 타입으로 캐스팅하여 속성을 제어할 수 있습니다.

### 4. 동적 컨트롤 생성
실행 중에 컨트롤을 동적으로 생성하려면, 먼저 표준 시각적 사용자 객체(Standard Visual User Object)로 해당 컨트롤을 상속받아 정의한 후, `OpenUserObject()` 함수를 사용하여 윈도우에 추가합니다.

**1단계: 사용자 객체 생성**
- `File → New` → `PB Object` 탭 → `Standard Visual` 선택
- `Types` 목록에서 `commandbutton` 선택
- 이름을 `u_my_button`으로 저장 (선택사항)

**2단계: 실행 중에 윈도우에 동적으로 추가**
```powerbuilder
// 윈도우 스크립트에서
u_my_button lu_btn
Parent.OpenUserObject(lu_btn, 100, 200)   // X=100, Y=200 위치에 생성
lu_btn.Text = "동적 버튼"
```

`OpenUserObject()`로 생성된 컨트롤은 윈도우가 닫힐 때 자동으로 소멸되지만, 중간에 제거하려면 `CloseUserObject()`를 사용합니다.
```powerbuilder
CloseUserObject(lu_btn)
```

> **참고**: PowerBuilder의 기본 컨트롤(CommandButton 등)을 직접 `CREATE`로 생성하는 것은 지원되지 않습니다. 시각적 객체는 반드시 사용자 객체로 정의한 후 `OpenUserObject()`로 생성해야 합니다.

---

## 실전 예제: 간단한 사용자 입력 폼

다음은 이름, 성별, 연령대, 전화번호를 입력받는 윈도우 예제입니다.

```
윈도우: w_input
컨트롤 배치:
   [StaticText] st_name: "이름"
   [SingleLineEdit] sle_name
   
   [GroupBox] gb_gender: "성별"
      [RadioButton] rb_male: "남성"  (Checked = TRUE)
      [RadioButton] rb_female: "여성"
   
   [GroupBox] gb_age: "연령대"
      [RadioButton] rb_10s: "10대"
      [RadioButton] rb_20s: "20대"   (Checked = TRUE)
      [RadioButton] rb_30s: "30대"
      [RadioButton] rb_40s: "40대 이상"
   
   [StaticText] st_phone: "전화번호"
   [EditMask] em_phone (Mask: "!(###) ###-####")
   
   [CommandButton] cb_save: "저장"
   [CommandButton] cb_cancel: "취소"
```

**cb_save의 Clicked 이벤트**
```
String ls_name, ls_gender, ls_age, ls_phone

// 이름 검증
ls_name = Trim(sle_name.Text)
IF ls_name = "" THEN
   MessageBox("오류", "이름을 입력하세요.")
   sle_name.SetFocus()
   RETURN
END IF

// 성별 확인
IF rb_male.Checked THEN
   ls_gender = "M"
ELSE
   ls_gender = "F"
END IF

// 연령대 확인
CHOOSE CASE TRUE
   CASE rb_10s.Checked
      ls_age = "10"
   CASE rb_20s.Checked
      ls_age = "20"
   CASE rb_30s.Checked
      ls_age = "30"
   CASE rb_40s.Checked
      ls_age = "40"
END CHOOSE

// 전화번호 (마스크 기호 제외한 원시 데이터)
em_phone.GetData(ls_phone)
IF Len(ls_phone) < 10 THEN   // 숫자만 있을 때 길이 체크
   MessageBox("오류", "전화번호를 완전히 입력하세요.")
   em_phone.SetFocus()
   RETURN
END IF

// 여기서 저장 로직 호출 (예: 데이터베이스 저장)
// f_save_user(ls_name, ls_gender, ls_age, ls_phone)

MessageBox("완료", "저장되었습니다.")
Close(Parent)
```

**cb_cancel의 Clicked 이벤트**
```
Close(Parent)
```

---

## 컨트롤의 상속과 사용자 객체

자주 사용되는 컨트롤 조합이나 특정 기능을 가진 컨트롤은 **사용자 객체(User Object)**로 만들어 재사용할 수 있습니다.

예를 들어, "조회", "저장", "삭제" 버튼이 항상 함께 다니는 경우, 이 세 버튼을 포함하는 시각적 사용자 객체를 만들고 각 버튼의 기능을 캡슐화할 수 있습니다. 그러면 여러 윈도우에서 이 사용자 객체를 배치하기만 하면 동일한 동작을 일관되게 사용할 수 있습니다.

사용자 객체는 **상속**을 통해 기본 컨트롤의 기능을 확장할 수도 있습니다. 예를 들어, `CommandButton`을 상속받아 항상 일정한 크기와 색상을 가지는 버튼 클래스를 만들 수 있습니다.

---

## 마치며

PowerBuilder의 다양한 컨트롤들은 윈도우 기반 애플리케이션 개발에 필요한 거의 모든 UI 요소를 제공합니다. 각 컨트롤의 특성과 용도를 정확히 이해하고, 적절한 이벤트와 함수를 활용하면 사용자 친화적이고 기능적인 인터페이스를 쉽게 구축할 수 있습니다. 특히 EditMask와 같은 컨트롤은 입력 형식을 자동으로 제어하여 데이터 무결성을 높이는 데 큰 도움이 됩니다.

컨트롤의 이름 규칙을 일관되게 사용하고, 그룹박스로 관련 컨트롤을 시각적으로 묶어주면 사용자 경험도 향상되고 코드 관리도 수월해집니다. 또한 자주 사용되는 컨트롤 조합은 사용자 객체로 만들어 재사용하면 개발 생산성을 더욱 높일 수 있습니다.
