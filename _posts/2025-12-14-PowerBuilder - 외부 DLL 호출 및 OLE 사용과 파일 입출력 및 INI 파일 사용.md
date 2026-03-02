---
layout: post
title: PowerBuilder - 외부 DLL 호출 및 OLE 사용과 파일 입출력 및 INI 파일 사용
date: 2025-12-14 14:30:23 +0900
category: PowerBuilder
---
# 외부 DLL 호출 및 OLE 사용과 파일 입출력 및 INI 파일 사용

PowerBuilder는 기본 기능만으로도 강력한 애플리케이션을 개발할 수 있지만, 때로는 운영체제의 기능이나 다른 애플리케이션의 자원을 활용해야 할 때가 있습니다. 이때 **외부 DLL 호출**과 **OLE 자동화**가 유용하게 사용됩니다. 또한 애플리케이션에서 데이터를 영구적으로 저장하거나 설정을 관리하기 위해 **파일 입출력**과 **INI 파일**을 사용할 수 있습니다. 이 글에서는 이러한 외부 연동 및 파일 처리 기술을 상세히 알아보겠습니다.

---

## 1. 외부 DLL 호출

### 1.1 개념

DLL(Dynamic Link Library)은 Windows 운영체제에서 공통으로 사용되는 함수들을 모아놓은 파일입니다. PowerBuilder에서는 표준 DLL에 포함된 함수를 호출하여 운영체제 기능이나 타사 라이브러리의 기능을 사용할 수 있습니다.

### 1.2 Declare 문을 이용한 외부 함수 선언

외부 DLL 함수를 사용하려면 먼저 PowerScript에서 해당 함수를 **선언(Declare)**해야 합니다. 선언 위치는 함수를 사용할 객체(윈도우, 메뉴, 사용자 객체)의 선언부나 전역 함수 선언부에 할 수 있습니다.

**문법**:
```powerbuilder
// 지역 외부 함수 (특정 객체 내에서만 사용)
FUNCTION 반환형 함수명 (인자목록) LIBRARY "DLL파일명" [ALIAS 외부이름]

// 전역 외부 함수 (애플리케이션 전체에서 사용)
GLOBAL FUNCTION 반환형 함수명 (인자목록) LIBRARY "DLL파일명" [ALIAS 외부이름]
```

**인자 전달 방식**:
- **BY VALUE**: 값 전달 (기본값)
- **REF**: 참조 전달 (포인터)

**예제: Windows API MessageBox 호출**:
```powerbuilder
// 윈도우의 Local External Functions 선언부
FUNCTION int MessageBoxA (uint hWnd, string lpText, string lpCaption, uint uType) LIBRARY "user32.dll"

// 호출
MessageBoxA(0, "안녕하세요", "외부 DLL 호출", 0)
```

### 1.3 인자 타입 매핑

PowerBuilder 데이터 타입과 C 언어 데이터 타입의 매핑은 중요합니다.

| PowerBuilder 타입 | C 타입 | 설명 |
|------------------|--------|------|
| `Char` | `char` | 1바이트 문자 |
| `String` | `LPSTR`, `LPCSTR` | null 종료 문자열 (REF로 전달) |
| `Integer` | `int`, `short` | 16비트 정수 |
| `Long` | `long`, `int` (32비트) | 32비트 정수 |
| `Uint` | `unsigned int` | 부호 없는 32비트 정수 |
| `Boolean` | `BOOL` | 0/1 값 |

**문자열 전달 시 주의**:
- `string` 타입은 REF로 전달해야 합니다.
- API가 문자열을 수정하는 경우 충분한 버퍼 크기를 미리 확보해야 합니다.

### 1.4 예제: 시스템 정보 가져오기 (GetComputerName)

```powerbuilder
// 외부 함수 선언
FUNCTION boolean GetComputerNameA (REF string lpBuffer, REF long lpnSize) LIBRARY "kernel32.dll"

// 사용
string ls_buffer
long ll_size

ls_buffer = Space(256)   // 256바이트 버퍼 할당
ll_size = 256
IF GetComputerNameA(ls_buffer, ll_size) THEN
    ls_buffer = Trim(ls_buffer)
    MessageBox("컴퓨터 이름", ls_buffer)
END IF
```

### 1.5 실전 팁

- DLL 함수 이름이 ANSI와 Unicode 버전으로 나뉘는 경우, ANSI 버전(함수명 뒤에 'A')을 사용하거나 PowerBuilder 2017 이상에서는 Unicode 지원 여부를 확인합니다.
- 구조체를 전달해야 한다면 PowerBuilder에서 구조체를 정의하고 `REF`로 전달합니다.
- 잘못된 호출은 애플리케이션을 비정상 종료시킬 수 있으므로 반드시 정확한 프로토타입을 확인해야 합니다.

---

## 2. OLE 자동화

### 2.1 개념

OLE 자동화(OLE Automation)는 애플리케이션 간에 객체를 공유하고 제어할 수 있게 해주는 마이크로소프트의 기술입니다. PowerBuilder에서는 **OLEObject** 객체를 사용하여 Excel, Word, Outlook 등 OLE 서버 애플리케이션을 제어할 수 있습니다.

### 2.2 OLEObject 생성 및 연결

**단계**:
1. `OLEObject` 객체 생성
2. `ConnectToObject()` 또는 `ConnectToNewObject()`로 OLE 서버 연결
3. 서버의 속성과 메서드 호출
4. 사용 후 `DisconnectObject()`로 연결 해제

```powerbuilder
OLEObject ole_excel
ole_excel = CREATE OLEObject

// 실행 중인 Excel 인스턴스에 연결 (없으면 오류)
IF ole_excel.ConnectToObject("Excel.Application") <> 0 THEN
    // 새 Excel 인스턴스 생성
    ole_excel.ConnectToNewObject("Excel.Application")
END IF

// Excel 표시 (보통 False로 숨김 처리)
ole_excel.Application.Visible = TRUE

// 사용 후 해제
ole_excel.DisconnectObject()
DESTROY ole_excel
```

### 2.3 속성 및 메서드 호출

OLEObject를 통해 연결되면, 해당 애플리케이션의 객체 모델을 그대로 사용할 수 있습니다.

**예제: Excel에 데이터 쓰기**
```powerbuilder
OLEObject ole_excel, ole_workbook, ole_sheet
ole_excel = CREATE OLEObject

// Excel 연결
IF ole_excel.ConnectToNewObject("Excel.Application") = 0 THEN
    ole_excel.Visible = TRUE
    
    // 새 워크북 추가
    ole_workbook = ole_excel.Workbooks.Add()
    
    // 첫 번째 시트 참조
    ole_sheet = ole_excel.ActiveSheet
    
    // 셀에 값 쓰기
    ole_sheet.Cells(1, 1).Value = "이름"
    ole_sheet.Cells(1, 2).Value = "점수"
    ole_sheet.Cells(2, 1).Value = "홍길동"
    ole_sheet.Cells(2, 2).Value = 95
    
    // 저장
    ole_workbook.SaveAs("C:\temp\score.xlsx")
    
    // 종료
    ole_excel.Quit()
END IF

ole_excel.DisconnectObject()
DESTROY ole_excel
```

### 2.4 오류 처리

OLE 호출 중 오류가 발생하면 `OLEObject`의 `OLEErrorCode` 속성이나 예외 처리를 통해 확인할 수 있습니다.

```powerbuilder
Integer li_rtn
li_rtn = ole_excel.ConnectToNewObject("Excel.Application")
IF li_rtn < 0 THEN
    MessageBox("OLE 오류", "Excel을 시작할 수 없습니다. 코드: " + String(li_rtn))
    RETURN
END IF
```

### 2.5 주요 OLE 객체 식별자

| 애플리케이션 | 프로그래밍 ID (ProgID) |
|-------------|------------------------|
| Excel | `Excel.Application` |
| Word | `Word.Application` |
| Outlook | `Outlook.Application` |
| Internet Explorer | `InternetExplorer.Application` |

### 2.6 실전 예제: Outlook을 통해 메일 보내기

```powerbuilder
OLEObject ole_outlook, ole_mail
ole_outlook = CREATE OLEObject

IF ole_outlook.ConnectToNewObject("Outlook.Application") = 0 THEN
    // 새 메일 항목 생성
    ole_mail = ole_outlook.CreateItem(0)  // 0 = olMailItem
    
    ole_mail.To = "recipient@example.com"
    ole_mail.Subject = "PowerBuilder에서 보낸 메일"
    ole_mail.Body = "안녕하세요, OLE 자동화 테스트입니다."
    
    // 첨부 파일 추가
    ole_mail.Attachments.Add("C:\temp\report.txt")
    
    // 메일 보내기
    ole_mail.Send()
    
    MessageBox("성공", "메일이 전송되었습니다.")
ELSE
    MessageBox("오류", "Outlook을 시작할 수 없습니다.")
END IF

ole_outlook.DisconnectObject()
DESTROY ole_outlook
```

---

## 3. 파일 입출력 (실무 표준: FileReadEx / FileWriteEx)

### 3.1 개념

PowerBuilder는 텍스트 파일, 바이너리 파일 등 다양한 파일 형식을 읽고 쓸 수 있는 함수를 제공합니다. 특히 **FileReadEx**와 **FileWriteEx**는 기존 파일 입출력 함수의 한계(32KB 제한)를 극복한 **무제한 용량** 함수로, 대용량 파일 처리에 적합합니다. 유니코드 환경에서도 안정적으로 동작하므로 실무에서는 이 함수들을 표준으로 사용해야 합니다.

### 3.2 파일 입출력 함수

| 함수 | 설명 |
|------|------|
| **FileOpen ( filename, filemode, fileaccess, filelock, writemode, encoding )** | 파일 열기 |
| **FileReadEx ( fileno, variable )** | 파일에서 읽기 (용량 제한 없음) |
| **FileWriteEx ( fileno, text_or_blob )** | 파일에 쓰기 (용량 제한 없음) |
| **FileClose ( fileno )** | 파일 닫기 |
| **FileSeek ( fileno, offset, origin )** | 파일 포인터 이동 |
| **FileExists ( filename )** | 파일 존재 여부 확인 |
| **FileDelete ( filename )** | 파일 삭제 |
| **FileCopy ( source, dest )** | 파일 복사 |
| **FileMove ( source, dest )** | 파일 이동 |

> **💡 실무 팁**: 기존 `FileRead`/`FileWrite` 함수는 최대 32,765바이트(약 32KB)까지만 처리할 수 있습니다. 따라서 대용량 파일이나 유니코드 텍스트를 다룰 때는 반드시 `FileReadEx`/`FileWriteEx`를 사용해야 합니다.

**파일 열기 모드**:

| 모드 | 설명 |
|------|------|
| **LineMode!** | 한 줄씩 읽기/쓰기 (텍스트 파일) |
| **StreamMode!** | 연속적인 데이터 스트림 (바이너리) |

**파일 접근 모드**:

| 모드 | 설명 |
|------|------|
| **Read!** | 읽기 전용 |
| **Write!** | 쓰기 전용 (기존 내용 삭제) |
| **WriteMode** 파라미터로 추가/덮어쓰기 지정 |

### 3.3 텍스트 파일 읽기 예제 (FileReadEx)

```powerbuilder
Integer li_filenum
String ls_line, ls_content

li_filenum = FileOpen("C:\temp\data.txt", LineMode!, Read!)
IF li_filenum > 0 THEN
    // FileReadEx는 파일 끝(EOF)에 도달하면 -100을 반환
    DO WHILE FileReadEx(li_filenum, ls_line) >= 0
        ls_content += ls_line + "~r~n"
    LOOP
    FileClose(li_filenum)
    MessageBox("파일 내용", ls_content)
ELSE
    MessageBox("오류", "파일을 열 수 없습니다.")
END IF
```

### 3.4 텍스트 파일 쓰기 예제 (FileWriteEx)

```powerbuilder
Integer li_filenum

li_filenum = FileOpen("C:\temp\output.txt", LineMode!, Write!, LockWrite!, Append!)
IF li_filenum > 0 THEN
    FileWriteEx(li_filenum, "첫 번째 줄")
    FileWriteEx(li_filenum, "두 번째 줄")
    FileClose(li_filenum)
    MessageBox("성공", "파일이 저장되었습니다.")
END IF
```

### 3.5 바이너리 파일 읽기/쓰기 (StreamMode!)

StreamMode!와 함께 `Blob` 변수를 사용하여 바이너리 데이터를 처리합니다.

```powerbuilder
// 파일 복사 예제 (FileReadEx/FileWriteEx 사용)
Integer li_src, li_dest
Blob lbl_data
Long ll_read

li_src = FileOpen("C:\temp\image.jpg", StreamMode!, Read!)
li_dest = FileOpen("C:\temp\copy.jpg", StreamMode!, Write!, LockWrite!, Replace!)
IF li_src > 0 AND li_dest > 0 THEN
    DO WHILE TRUE
        ll_read = FileReadEx(li_src, lbl_data)
        IF ll_read <= 0 THEN EXIT
        FileWriteEx(li_dest, lbl_data)
    LOOP
    FileClose(li_src)
    FileClose(li_dest)
    MessageBox("성공", "파일이 복사되었습니다.")
END IF
```

### 3.6 실전 팁

- 파일 작업 후 반드시 `FileClose`로 핸들을 닫아야 리소스 누수를 방지할 수 있습니다.
- 대용량 파일 처리 시에는 버퍼 크기를 적절히 조절하거나 `FileSeek`를 활용합니다.
- 파일 경로는 역슬래시를 두 번(`\\`) 쓰거나 슬래시(`/`)를 사용합니다.

---

## 4. INI 파일 사용

### 4.1 개념

INI 파일은 초기화 설정을 저장하는 간단한 텍스트 파일 형식입니다. 섹션(Section), 키(Key), 값(Value)의 구조로 되어 있습니다.

```
[Section1]
Key1=Value1
Key2=Value2

[Section2]
Key3=Value3
```

### 4.2 INI 파일 관련 함수

| 함수 | 설명 |
|------|------|
| **ProfileString ( filename, section, key, default )** | INI 파일에서 문자열 값 읽기 |
| **ProfileInt ( filename, section, key, default )** | INI 파일에서 정수 값 읽기 |
| **SetProfileString ( filename, section, key, value )** | INI 파일에 문자열 값 쓰기 |

### 4.3 INI 파일 읽기 예제

```powerbuilder
String ls_dbms, ls_server, ls_database

ls_dbms = ProfileString("C:\config.ini", "Database", "DBMS", "")
ls_server = ProfileString("C:\config.ini", "Database", "Server", "localhost")
ls_database = ProfileString("C:\config.ini", "Database", "Database", "")

SQLCA.DBMS = ls_dbms
SQLCA.ServerName = ls_server
SQLCA.Database = ls_database
```

### 4.4 INI 파일 쓰기 예제

```powerbuilder
Integer li_rtn

li_rtn = SetProfileString("C:\config.ini", "Database", "DBMS", "ODBC")
IF li_rtn = -1 THEN
    MessageBox("오류", "INI 파일 쓰기 실패")
END IF
```

### 4.5 실전 예제: 애플리케이션 설정 저장/불러오기

```powerbuilder
// 설정 저장
SetProfileString("app.ini", "User", "ID", sle_user.Text)
SetProfileString("app.ini", "User", "Theme", ddlb_theme.Text)
SetProfileString("app.ini", "Window", "Left", String(w_main.X))
SetProfileString("app.ini", "Window", "Top", String(w_main.Y))

// 설정 불러오기 (윈도우 Open 이벤트)
sle_user.Text = ProfileString("app.ini", "User", "ID", "")
ddlb_theme.Text = ProfileString("app.ini", "User", "Theme", "Basic")

Integer li_left, li_top
li_left = ProfileInt("app.ini", "Window", "Left", 100)
li_top = ProfileInt("app.ini", "Window", "Top", 100)
w_main.Move(li_left, li_top)
```

### 4.6 실무 팁

- INI 파일은 간단한 설정 저장에 유용하지만, 보안이 중요한 정보(비밀번호 등)는 암호화하여 저장해야 합니다.
- ProfileString 함수는 값을 찾지 못하면 기본값을 반환하므로, 기본값을 적절히 설정해야 합니다.
- INI 파일 경로는 절대 경로보다 실행 파일 경로를 기준으로 상대 경로를 사용하는 것이 좋습니다.

---

## 결론

외부 DLL 호출과 OLE 자동화는 PowerBuilder 애플리케이션의 기능을 무한히 확장할 수 있는 강력한 도구입니다. Windows API를 직접 호출하여 시스템 수준의 기능을 사용하거나, Excel, Word 등 다른 애플리케이션을 제어하여 복잡한 작업을 자동화할 수 있습니다.

파일 입출력과 INI 파일 관리는 애플리케이션에서 데이터를 영구적으로 저장하거나 설정을 관리하는 기본적이면서도 필수적인 기술입니다. 특히 파일 입출력 시에는 **FileReadEx/FileWriteEx**를 사용하여 32KB 제한을 극복하고 안정적인 대용량 파일 처리를 구현해야 합니다.
