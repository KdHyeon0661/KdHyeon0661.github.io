---
layout: post
title: PowerBuilder - Debug 모드 사용 및 메시지 로그
date: 2025-12-14 16:30:23 +0900
category: PowerBuilder
---
# Debug 모드 사용 및 메시지 로그 남기기

PowerBuilder 애플리케이션 개발에서 디버깅은 예상치 못한 오류를 찾고 수정하는 핵심 과정입니다. PowerBuilder IDE는 강력한 디버깅 도구를 내장하고 있으며, 적절한 로깅 기법과 결합하면 문제 해결 시간을 획기적으로 단축할 수 있습니다. 이 글에서는 Debug 모드의 사용법(중단점, 변수 감시, 호출 스택 등)과 메시지 로그를 남기는 다양한 방법을 상세히 알아보겠습니다.

---

## 1. PowerBuilder 디버깅 개요

PowerBuilder의 디버거는 다음과 같은 기능을 제공합니다.

- **중단점(Breakpoint)** 설정: 특정 코드 라인에서 실행을 일시 중지
- **단계별 실행**: 한 줄씩 코드 실행 (Step Into, Step Over, Step Out)
- **변수 감시(Watch)**: 특정 변수의 값 변화 실시간 추적
- **호출 스택(Call Stack)**: 현재 실행 지점까지의 함수 호출 경로 확인
- **즉시 실행 창(Immediate Window)**: 실행 중간에 코드 실행 및 변수 값 변경

디버깅은 개발 중인 애플리케이션을 **Debug 모드**로 실행해야 가능합니다.

---

## 2. Debug 모드 사용하기

### 2.1 Debug 모드 진입

디버그 모드를 시작하는 방법은 두 가지입니다.

- **도구 모음**에서 **Debug** 버튼 클릭 (벌레 아이콘)
- 메뉴에서 **Run → Debug** 선택 (단축키: `Shift+F5`)

디버그 모드로 실행하면 애플리케이션이 시작되지만, 중단점을 만나면 실행이 일시 중지됩니다.

### 2.2 Breakpoint 설정

중단점은 코드의 특정 라인에서 실행을 멈추도록 표시하는 지점입니다.

**설정 방법**:
- 스크립트 편집기에서 중단점을 설정할 라인의 왼쪽 회색 여백을 클릭합니다.
- 또는 해당 라인에 커서를 놓고 **Insert Breakpoint** 버튼 클릭 또는 `F9` 키를 누릅니다.

중단점이 설정된 라인은 빨간색 원으로 표시됩니다.

```
// 예시: 버튼 Clicked 이벤트에서 중단점 설정
Integer li_count
li_count = Integer(sle_count.Text)   // [여기에 중단점 설정]
FOR li_i = 1 TO li_count
   // ...
NEXT
```

### 2.3 단계별 실행

중단점에서 멈춘 후 다음 명령어로 코드를 한 단계씩 실행할 수 있습니다.

| 명령 | 단축키 | 설명 |
|------|--------|------|
| **Step Into** | `F8` | 현재 라인의 함수나 이벤트 내부로 들어감 |
| **Step Over** | `F10` | 현재 라인을 실행하고 다음 라인으로 이동 (함수 내부로 들어가지 않음) |
| **Step Out** | `Shift+F8` | 현재 함수/이벤트의 나머지를 실행하고 호출자로 돌아감 |
| **Continue** | `F5` | 다음 중단점까지 계속 실행 |

### 2.4 변수 값 확인 (Watch)

실행 중인 변수의 값을 확인하는 방법은 여러 가지입니다.

**QuickWatch**:
- 코드 창에서 변수 위에 마우스 커서를 올리면 현재 값이 툴팁으로 표시됩니다.
- 또는 변수를 선택한 후 `Shift+F9`를 누르면 QuickWatch 대화상자가 열립니다.

**Watch 창**:
- **Debug** 메뉴에서 **Watch** 창을 엽니다. (또는 `Ctrl+W`)
- 빈 줄에 변수 이름을 입력하면 해당 변수의 현재 값이 실시간으로 표시됩니다.
- 객체의 속성도 점(.) 표기법으로 감시할 수 있습니다.

```
// Watch 창에 등록 예
li_count
ls_name
dw_1.RowCount()
```

**변수 값 변경**:
디버그 중에 Watch 창에서 변수 값을 직접 변경할 수 있습니다. 값을 더블클릭하고 새 값을 입력하면 됩니다.

### 2.5 Call Stack 확인

**Call Stack** 창은 현재 실행 지점까지 호출된 함수/이벤트의 목록을 보여줍니다. 이를 통해 어떤 경로로 현재 코드에 도달했는지 추적할 수 있습니다.

- **Debug** 메뉴에서 **Call Stack** 창을 엽니다.
- 목록의 각 항목을 더블클릭하면 해당 호출 지점의 코드로 이동합니다.

```
Call Stack 예시:
   [0] w_customer.cb_save.Clicked (line 10)
   [1] w_customer.uf_validate (line 25)
   [2] w_customer.uf_save (line 40)
```

### 2.6 조건부 중단점

특정 조건에서만 멈추도록 중단점에 조건을 설정할 수 있습니다.

**설정 방법**:
- 중단점이 설정된 라인을 마우스 오른쪽 클릭 → **Breakpoints...** 선택
- **Condition** 필드에 조건식을 입력합니다. (예: `li_count > 10`)
- 조건이 참일 때만 중단됩니다.

조건부 중단점은 반복문에서 특정 반복 회차에만 멈추고 싶을 때 유용합니다.

---

## 3. 실전 디버깅 예제

다음 시나리오로 디버깅 과정을 연습해 보겠습니다.

**문제**: 고객 정보를 저장하는 버튼을 클릭하면 가끔 "데이터 저장 실패" 오류가 발생합니다.

**디버깅 단계**:

1. **중단점 설정**: `cb_save`의 Clicked 이벤트 첫 줄에 중단점 설정.
2. **Debug 모드 실행**: 애플리케이션 실행 후 저장 버튼 클릭 → 중단점에서 멈춤.
3. **Step Over로 진행**: 한 줄씩 실행하면서 각 변수의 값 확인.
4. **Watch 추가**: `ls_name`, `ls_phone`, `li_rtn` 변수 등록.
5. **문제 발견**: `ls_name`이 빈 문자열인 경우 발견. (사용자 입력 누락)
6. **Call Stack 확인**: 이벤트가 어떻게 호출되었는지 추적 (특별한 문제 없음).
7. **조건부 중단점 활용**: 반복문이 있는 경우 특정 조건에서 멈추도록 설정.

이러한 과정을 통해 오류 원인을 신속히 파악할 수 있습니다.

---

## 4. 메시지 로그 남기기

디버깅은 개발 중에만 가능하지만, 운영 환경에서 발생하는 문제를 추적하려면 **로깅(Logging)**이 필수적입니다. PowerBuilder에서는 다양한 방법으로 로그를 남길 수 있습니다.

### 4.1 MessageBox를 이용한 디버깅

가장 간단한 방법으로, 중간 결과를 MessageBox로 표시합니다.

```powerbuilder
MessageBox("디버그", "현재 값: " + String(li_value))
```

**장점**: 간편함  
**단점**: 운영 시 제거해야 함, 많은 메시지가 사용자 경험을 해침

### 4.2 파일에 로그 남기기

파일 입출력 함수를 사용하여 로그 파일에 기록합니다.

**로그 함수 예제**:
```powerbuilder
// 전역 함수 f_log_write
FUNCTION f_log_write (String as_message)
   Integer li_file
   String ls_logfile = "C:\temp\app.log"
   String ls_timestamp
   
   ls_timestamp = String(Today(), "yyyy-mm-dd") + " " + String(Now())
   li_file = FileOpen(ls_logfile, LineMode!, Write!, LockWrite!, Append!)
   IF li_file > 0 THEN
      FileWriteEx(li_file, ls_timestamp + " : " + as_message)
      FileClose(li_file)
   END IF
END FUNCTION
```

**사용**:
```powerbuilder
f_log_write("저장 버튼 클릭됨")
f_log_write("사용자 ID: " + sle_id.Text)
```

### 4.3 Output 윈도우 활용 (DB 쿼리 디버깅)

PowerBuilder의 스크립트 편집기에서 직접 하단 Output 창에 텍스트를 찍는 함수(Print나 Console.WriteLine 같은 함수)는 내장되어 있지 않습니다. 대신, 실무에서 Output 창을 가장 유용하게 쓰는 방법은 **데이터베이스 쿼리 추적(Trace)**입니다.

**방법**: 데이터베이스 연결 문자열에 `TRACE` 키워드를 추가하거나, `SQLCA.DBMS` 속성에 `TRACE`를 포함시키면 DataWindow가 실행하는 모든 SQL 문이 하단 Output 윈도우에 실시간으로 출력됩니다.

```powerbuilder
// 예: ODBC 연결 시 TRACE 추가
SQLCA.DBMS = "TRACE ODBC"
SQLCA.DBParm = "ConnectString='DSN=MyDB;UID=sa;PWD=1234'"
```

이렇게 설정하면 애플리케이션 실행 중에 DataWindow가 생성한 INSERT, UPDATE, DELETE, SELECT 문이 Output 창에 나타나므로, 잘못된 쿼리나 데이터 문제를 신속히 파악할 수 있습니다. 추적 결과는 파일로도 저장할 수 있습니다.

### 4.4 PFC의 로깅 서비스 활용

PFC 애플리케이션을 사용한다면 내장된 **오류 서비스(Error Service)**를 활용할 수 있습니다.

**설정**:
```powerbuilder
// n_cst_appmanager의 pfc_Open 이벤트에서
this.of_SetError(TRUE)   // 오류 서비스 활성화
```

**로그 기록**:
```powerbuilder
// 오류 메시지 로그
gnv_app.inv_error.of_Message("오류", "데이터 저장 실패")

// 사용자 정의 로그
gnv_app.inv_error.of_Log("사용자 작업: 저장 버튼 클릭")
```

PFC의 오류 서비스는 자동으로 로그 파일을 생성하고 관리합니다. 기본적으로 `app.log` 파일에 기록됩니다.

### 4.5 조건부 로깅

운영 환경에서는 성능을 고려하여 특정 조건에서만 로그를 남기는 것이 좋습니다.

```powerbuilder
// 디버그 모드에서만 로그
IF gnv_app.inv_debug.of_IsActive() THEN
   f_log_write("상세 정보: " + ls_detail)
END IF

// 오류 발생 시에만 로그
IF li_result < 0 THEN
   f_log_write("오류 발생: 코드 " + String(li_result))
END IF
```

---

## 5. 종합 예제: 디버깅과 로깅을 함께 사용한 오류 추적

다음은 실제 애플리케이션에서 발생한 문제를 디버깅과 로깅을 결합하여 해결하는 과정입니다.

### 시나리오
고객 등록 화면에서 가끔 "중복된 ID" 오류가 발생하지만, 재현이 어렵습니다. 사용자가 입력한 값을 추적해야 합니다.

### 해결 방법

**1. 로깅 강화**: 사용자 입력 값을 파일에 기록

```powerbuilder
// 저장 버튼 Clicked 이벤트
String ls_id, ls_name, ls_phone
ls_id = sle_id.Text
ls_name = sle_name.Text
ls_phone = sle_phone.Text

// 사용자 입력 로그
f_log_write("저장 시도 - ID:" + ls_id + ", Name:" + ls_name + ", Phone:" + ls_phone)

// 중복 체크 로직 (예: SELECT COUNT)
// ...
IF ll_count > 0 THEN
   f_log_write("중복 ID 발견: " + ls_id)
   MessageBox("오류", "이미 존재하는 ID입니다.")
   RETURN
END IF

// 저장 프로시저 호출
// ...
```

**2. 운영 중 로그 분석**: 로그 파일에서 중복 오류 발생 패턴 확인

**3. 문제 재현 및 디버깅**: 로그에 기록된 값으로 테스트 환경에서 재현

**4. 중단점 설정 및 디버깅**: 해당 값으로 디버그 모드 실행, 중단점에서 변수 확인

**5. 수정 및 확인**: 버그 수정 후 로그에 정상 저장 기록 확인

---

## 6. 실무 꿀팁: Just-In-Time(JIT) 디버깅

일반적으로는 디버그 모드로 애플리케이션을 시작해야 디버깅이 가능합니다. 하지만 때로는 일반 **Run** 모드로 테스트하다가 예상치 못한 오류가 발생했을 때, 그 순간의 상태를 디버거로 확인하고 싶을 수 있습니다. 이때 유용한 기능이 **Just-In-Time(JIT) 디버깅**입니다.

### JIT 디버깅 설정 방법

1. 메뉴에서 **Tools → System Options**을 엽니다.
2. **Just In Time Debugging** 체크박스를 활성화합니다.

이 옵션을 켜면 애플리케이션 실행 중 오류(SystemError 등)가 발생했을 때, 오류 메시지 박스에 **[Debug]** 버튼이 함께 표시됩니다. 이 버튼을 클릭하면 즉시 디버거가 실행되어 오류가 발생한 정확한 코드 라인으로 진입하며, 그 시점의 변수 값들을 확인할 수 있습니다. 이 기능을 활용하면 재현이 어려운 오류를 효과적으로 추적할 수 있습니다.

---

## 7. 결론 및 모범 사례

효과적인 디버깅과 로깅은 안정적인 애플리케이션 개발의 핵심입니다.

### 디버깅 모범 사례
- 복잡한 로직에는 중단점을 적극 활용하세요.
- 조건부 중단점으로 반복문 내 특정 조건만 확인하세요.
- Watch 창에 주요 변수를 등록하여 값 변화를 추적하세요.
- Call Stack으로 실행 경로를 이해하세요.
- JIT 디버깅을 활성화하여 예기치 않은 오류를 즉시 포착하세요.

### 로깅 모범 사례
- 로그 레벨을 구분하세요 (Info, Warning, Error).
- 운영 환경에서는 필요한 로그만 남기도록 조건을 두세요.
- 로그 파일의 크기 관리(로테이션)를 고려하세요.
- 민감한 정보(비밀번호 등)는 로그에 기록하지 마세요.
- PFC를 사용한다면 내장 오류 서비스를 활용하세요.
- DB 쿼리 추적(TRACE)을 활용하여 DataWindow의 SQL을 모니터링하세요.

디버깅과 로깅은 개발자에게 강력한 도구입니다. 이 도구들을 잘 활용하면 예상치 못한 문제에 빠르게 대응하고, 애플리케이션의 품질을 높일 수 있습니다.