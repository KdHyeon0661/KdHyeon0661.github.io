---
layout: post
title: PowerBuilder - 웹 서비스 연동 및 PFC 프레임워크
date: 2025-12-14 15:30:23 +0900
category: PowerBuilder
---
# 웹 서비스 연동 (SOAP/REST) 및 PFC 프레임워크 이해

현대 비즈니스 애플리케이션 개발에서 외부 시스템과의 연동은 필수적인 요소가 되었습니다. PowerBuilder는 웹 서비스를 통한 시스템 통합을 지원하며, 대규모 프로젝트의 생산성과 유지보수성을 높이기 위해 **PFC(PowerBuilder Foundation Class)** 프레임워크를 제공합니다. 이 글에서는 SOAP/REST 기반 웹 서비스 연동 방법과 PFC 프레임워크의 상속 구조 및 기본 흐름을 상세히 알아보겠습니다.

---

## 1. 웹 서비스 연동 (SOAP/REST)

### 1.1 웹 서비스 개념

웹 서비스는 네트워크를 통해 서로 다른 시스템 간에 데이터를 교환할 수 있게 해주는 소프트웨어 인터페이스입니다. PowerBuilder는 두 가지 주요 웹 서비스 방식을 지원합니다.

| 방식 | 특징 | 프로토콜 | 데이터 형식 |
|------|------|----------|------------|
| **SOAP** | 정형화된 메시지 구조, WSDL 정의 필요 | HTTP/HTTPS, SMTP 등 | XML |
| **REST** | 리소스 중심, 경량화 | HTTP/HTTPS | JSON, XML |

### 1.2 SOAP 웹 서비스 연동

PowerBuilder에서 SOAP 웹 서비스를 사용하려면 **웹 서비스 프록시(Proxy) 객체**를 생성해야 합니다.

#### 웹 서비스 프록시 생성 단계

1. **Web Service Proxy Wizard 실행**
   - **File → New** → **Project** 탭 → **Web Service Proxy** 선택

2. **WSDL 파일 또는 URL 지정**
   - 로컬 WSDL 파일 경로 또는 원격 WSDL URL 입력
   ```
   http://example.com/service?wsdl
   ```

3. **프록시 라이브러리 생성**
   - 생성될 프록시 객체를 저장할 PBL 파일 지정
   - 프록시 이름과 네임스페이스 설정

4. **프록시 객체 생성 완료**
   - 생성된 프록시 객체는 PowerBuilder 라이브러리에 저장됨

#### SOAP 웹 서비스 호출 예제

```powerbuilder
// 웹 서비스 프록시 객체 생성
soap_connection lnv_soap
ws_customer_service lnv_service   // 생성된 프록시 객체

lnv_service = CREATE ws_customer_service
lnv_service.SetOptions( "SoapLog=TRUE" )  // 디버깅 옵션

// SOAP 연결 설정
lnv_soap = CREATE soap_connection
lnv_soap.SetOptions( "SoapVersion=1.1" )
lnv_service.SetSoapConnection( lnv_soap )

// 웹 서비스 메서드 호출
String ls_result
ls_result = lnv_service.getCustomerInfo("C001")

IF lnv_service.GetLastError() <> 0 THEN
   MessageBox("SOAP 오류", lnv_service.GetLastErrorMessage())
ELSE
   // 결과 처리
   MessageBox("고객 정보", ls_result)
END IF

// 객체 소멸
DESTROY lnv_service
DESTROY lnv_soap
```

### 1.3 REST 웹 서비스 연동

REST 서비스는 PowerBuilder 2017 이상에서 **HttpClient** 객체를 통해 사용할 수 있습니다. HttpClient는 `SendRequest()` 메서드를 사용하여 HTTP 메서드(GET, POST, PUT, DELETE 등)를 실행하고, 응답은 `GetResponseBody()`로 받아야 합니다.

#### HttpClient를 이용한 REST 호출 (올바른 문법)

```powerbuilder
// HttpClient 객체 생성
HttpClient lnv_http
lnv_http = CREATE HttpClient

// 요청 헤더 설정
lnv_http.SetRequestHeader("Content-Type", "application/json")
lnv_http.SetRequestHeader("Authorization", "Bearer " + ls_token)

// GET 요청
Integer li_rtn
String ls_response

li_rtn = lnv_http.SendRequest("GET", "https://api.example.com/customers/C001")
IF li_rtn = 1 AND lnv_http.GetResponseStatusCode() = 200 THEN
    lnv_http.GetResponseBody(ls_response)
    // ls_response에 JSON 응답이 저장됨
END IF

// POST 요청 (JSON 데이터 전송)
String ls_json = '{"name":"홍길동", "email":"hong@example.com"}'
li_rtn = lnv_http.SendRequest("POST", "https://api.example.com/customers", ls_json)
IF li_rtn = 1 AND lnv_http.GetResponseStatusCode() = 200 THEN
    lnv_http.GetResponseBody(ls_response)
END IF

// 응답 처리
IF lnv_http.GetLastError() <> 0 THEN
   MessageBox("REST 오류", lnv_http.GetLastErrorMessage())
ELSE
   // JSON 파싱 (아래 JSONParser 참조)
END IF

DESTROY lnv_http
```

#### WinHTTP API를 이용한 REST 호환 방식 (하위 호환)

```powerbuilder
// 외부 함수 선언 (WinHTTP)
FUNCTION long WinHttpOpen(REF string szUserAgent, long dwAccessType, 
                         string szProxyName, string szProxyBypass, long dwFlags) 
                         LIBRARY "winhttp.dll"
// ... 추가 API 선언 필요

// 간편한 사용을 위한 HTTP 서비스 객체 구현 가능
```

### 1.4 JSON 데이터 처리

REST 서비스에서 주로 사용하는 JSON 데이터는 **JSONParser** 객체로 처리합니다. JSON 문자열을 로드한 후, **루트 핸들(Root Item Handle)**을 얻고, 핸들을 기준으로 타입별 값을 추출합니다.

```powerbuilder
// JSONParser 객체 생성
JSONParser lnv_json
lnv_json = CREATE JSONParser

// JSON 문자열 로드 (Parse 대신 LoadString 사용 권장)
String ls_json = '{"id":"C001","name":"홍길동","age":30}'
String ls_error

ls_error = lnv_json.LoadString(ls_json)
IF ls_error <> "" THEN
    MessageBox("JSON 오류", ls_error)
    RETURN
END IF

// 최상위 루트 핸들 가져오기
Long ll_root
ll_root = lnv_json.GetRootItem()

// 핸들을 통해 타입별 값 추출
String ls_id = lnv_json.GetItemString(ll_root, "id")
String ls_name = lnv_json.GetItemString(ll_root, "name")
Integer li_age = Integer(lnv_json.GetItemNumber(ll_root, "age"))

// 객체 배열 처리
String ls_json_array = '[{"id":"C001"},{"id":"C002"}]'
lnv_json.LoadString(ls_json_array)
ll_root = lnv_json.GetRootItem()
Integer li_count = lnv_json.GetArrayLength(ll_root)   // 배열 길이
FOR i = 0 TO li_count - 1   // 배열 인덱스는 0부터 시작
    Long ll_elem = lnv_json.GetArrayItem(ll_root, i)
    ls_id = lnv_json.GetItemString(ll_elem, "id")
NEXT

DESTROY lnv_json
```

### 1.5 웹 서비스 연동 시 고려사항

| 고려사항 | 설명 |
|---------|------|
| **인증 처리** | Basic Auth, OAuth 2.0, API Key 등 인증 방식 구현 |
| **타임아웃 설정** | 네트워크 지연 대비 적절한 타임아웃 값 설정 |
| **오류 처리** | HTTP 상태 코드 및 서비스별 오류 코드 처리 |
| **로깅** | 요청/응답 로깅으로 디버깅 및 모니터링 |
| **비동기 처리** | 장시간 소요되는 호출은 PostEvent로 비동기 처리 |

---

## 2. PFC(PowerBuilder Foundation Class) 프레임워크

### 2.1 PFC란?

PFC는 PowerBuilder 개발 환경에서 제공하는 **기본 클래스 라이브러리**로, 애플리케이션 개발에 자주 사용되는 객체와 함수를 미리 구현해 놓은 프레임워크입니다. PFC를 사용하면 다음과 같은 이점이 있습니다.

- **생산성 향상**: 검증된 공통 기능을 재사용하여 개발 시간 단축
- **일관성 유지**: 표준화된 객체 구조로 일관된 코드 작성 가능
- **유지보수성**: 프레임워크 레벨에서 버그 수정 및 기능 개선 효과 공유

PFC는 크게 **기본 클래스(PFC)**와 **확장 클래스(PFE)**로 구성됩니다. 애플리케이션 개발자는 PFC를 직접 수정하지 않고 PFE에서 상속받아 확장합니다.

### 2.2 PFC 라이브러리 구성

PFC 애플리케이션을 개발하려면 다음 라이브러리를 참조해야 합니다.

| 라이브러리 | 설명 |
|-----------|------|
| **PFCAPSRV.PBL** | 애플리케이션 서비스 (AppManager, Debug, Error 등) |
| **PFCDWSRV.PBL** | 데이터윈도우 서비스 (Sort, Filter, Retrieve 등) |
| **PFCMAIN.PBL** | 기본 윈도우 객체 (w_main, w_frame, w_sheet 등) |
| **PFCWNSRV.PBL** | 윈도우 서비스 (Resize, StatusBar, SheetManager 등) |
| **PFCUTIL.PBL** | 유틸리티 서비스 (File, INI, DateTime 등) |
| **PFEAPSRV.PBL** | 애플리케이션 서비스 확장 |
| **PFEDWSRV.PBL** | 데이터윈도우 서비스 확장 |
| **PFEMAIN.PBL** | 기본 윈도우 객체 확장 |
| **PFEWNSRV.PBL** | 윈도우 서비스 확장 |
| **PFEUTIL.PBL** | 유틸리티 서비스 확장 |

### 2.3 PFC 상속 구조 이해

PFC는 계층적인 상속 구조를 가지고 있습니다. 각 객체는 다음과 같은 단계로 상속됩니다.

```
기본 시스템 객체 (예: window)
    ↑
PFC 기본 클래스 (pfc_window)
    ↑
PFC 확장 클래스 (pfe_window)
    ↑
사용자 정의 클래스 (w_my_window)
```

**주요 객체 상속 구조 예시**:

| 기본 객체 | PFC 기본 | PFC 확장 | 사용자 정의 |
|----------|----------|----------|------------|
| window | pfc_w_main, pfc_w_frame, pfc_w_sheet | pfe_w_main, pfe_w_frame, pfe_w_sheet | w_customer, w_order |
| commandbutton | pfc_cmd | pfe_cmd | cb_save, cb_search |
| transaction | pfc_n_tr | pfe_n_tr | n_tr (SQLCA 타입) |
| datastore | pfc_n_ds | pfe_n_ds | n_customer_ds |

### 2.4 PFC 애플리케이션의 핵심: n_cst_appmanager

PFC 애플리케이션의 가장 중요한 객체는 **n_cst_appmanager**입니다. 이 객체는 기존 애플리케이션 객체의 역할을 대체하며, 애플리케이션의 전체 생명주기를 관리합니다.

#### n_cst_appmanager 상속 구조

```
pfc_n_base → n_base → pfc_n_cst_appmanager → n_cst_appmanager
```

#### 주요 이벤트

| 이벤트 | 설명 |
|-------|------|
| **Constructor** | 객체 생성 시 초기화 (버전, 회사명, INI 파일 등 설정) |
| **pfc_Open** | 애플리케이션 시작 시 실행 (애플리케이션 서비스 활성화, 초기 윈도우 열기) |
| **pfc_Close** | 애플리케이션 종료 시 실행 |
| **pfc_Logon** | 로그인 처리 (of_LogonDlg() 호출 시 실행) |
| **pfc_PreAbout** | About 대화상자 표시 전 호출 |
| **pfc_PreSplash** | Splash 화면 표시 전 호출 |
| **pfc_SystemError** | 시스템 오류 발생 시 호출 |

#### 주요 함수

| 함수 | 설명 |
|------|------|
| **of_SetAppIniFile()** | 애플리케이션 INI 파일 설정 |
| **of_SetVersion()** | 버전 정보 설정 |
| **of_SetCopyright()** | 저작권 정보 설정 |
| **of_GetAppIniFile()** | INI 파일 경로 반환 |
| **of_LogonDlg()** | 로그인 대화상자 표시 |
| **of_Splash()** | 스플래시 화면 표시 |
| **of_SetFrame()** | MDI 프레임 윈도우 설정 |

### 2.5 PFC 애플리케이션 기본 흐름

#### 단계 1: 애플리케이션 객체 설정

**전역 변수 선언**
```powerbuilder
// Application 객체의 Global Variables 선언부
n_cst_appmanager gnv_app   // 반드시 gnv_app이라는 이름 사용
```

**Application Open 이벤트**
```powerbuilder
// Application Open 이벤트
gnv_app = CREATE n_cst_appmanager   // 또는 상속받은 사용자 정의 객체
gnv_app.EVENT pfc_Open(commandline)
```

**Application Close 이벤트**
```powerbuilder
// Application Close 이벤트
gnv_app.EVENT pfc_Close()
DESTROY gnv_app
```

**Application SystemError 이벤트**
```powerbuilder
// Application SystemError 이벤트
gnv_app.EVENT pfc_SystemError()
```

#### 단계 2: n_cst_appmanager 커스터마이징

**Constructor 이벤트에서 기본 정보 설정**
```powerbuilder
// n_cst_appmanager의 Constructor 이벤트
this.of_SetAppIniFile("app.ini")
this.of_SetVersion("버전 1.0")
this.of_SetCopyright("Copyright 2025 MyCompany")
this.of_SetLogo("logo.bmp")
this.of_SetMicrohelp(TRUE)
```

**pfc_Open 이벤트에서 서비스 활성화 및 초기 화면 열기**
```powerbuilder
// n_cst_appmanager의 pfc_Open 이벤트
// 애플리케이션 서비스 활성화
this.of_SetAppPreference(TRUE)     // 환경설정 서비스
this.of_SetDebug(TRUE)              // 디버그 서비스
this.of_SetDwCache(TRUE)            // DataWindow 캐시 서비스
this.of_SetError(TRUE)              // 오류 처리 서비스
this.of_SetMRU(TRUE)                // 최근 사용 객체 서비스
this.of_SetSecurity(TRUE)           // 보안 서비스

// 데이터베이스 연결
String ls_inifile = this.of_GetAppIniFile()
IF SQLCA.of_Init(ls_inifile, "Database") = -1 THEN
   this.inv_error.of_Message("오류", "INI 파일을 찾을 수 없습니다.")
   HALT CLOSE
   RETURN
END IF

IF SQLCA.of_Connect() = -1 THEN
   this.inv_error.of_Message("연결 오류", "데이터베이스 연결에 실패했습니다.")
   HALT CLOSE
   RETURN
END IF

// 스플래시 화면 표시 (2초)
this.of_Splash(2)

// 로그인 대화상자 표시
IF this.of_LogonDlg() <> 1 THEN
   HALT CLOSE
   RETURN
END IF

// MDI 프레임 윈도우 열기
Open(w_frame)
```

#### 단계 3: 로그인 처리

**pfc_Logon 이벤트에서 사용자 인증**
```powerbuilder
// n_cst_appmanager의 pfc_Logon 이벤트
// as_userid, as_password는 이벤트 파라미터로 전달됨
String ls_inifile
Integer li_rtn

ls_inifile = this.of_GetAppIniFile()

// SQLCA의 타입은 n_tr로 설정되어 있어야 함 (트랜잭션 객체 타입 변경)
li_rtn = SQLCA.of_Init(ls_inifile, "Database")
IF li_rtn = -1 THEN RETURN -1

// 사용자 ID/비밀번호 설정
SQLCA.of_SetUser(as_userid, as_password)

IF SQLCA.of_Connect() = -1 THEN
   RETURN -1   // 로그인 실패
ELSE
   this.of_SetUserID(as_userid)   // 사용자 ID 저장
   RETURN 1    // 로그인 성공
END IF
```

### 2.6 MDI 프레임과 시트(Sheet) 구현

#### 프레임 윈도우 생성

`w_frame` 윈도우 생성 (pfe_w_frame 상속):
- WindowType: `mdihelp!`
- 적절한 메뉴 연결

**w_frame의 Open 이벤트**
```powerbuilder
// 상태바 서비스 활성화
this.of_SetStatusBar(TRUE)

// 시트 관리 서비스 활성화
this.of_SetSheetManager(TRUE)

// 초기 메시지 설정
this.SetMicroHelp("준비")
```

#### 시트 윈도우 생성

`w_customer_list` 윈도우 생성 (pfe_w_sheet 상속):
- DataWindow 컨트롤 포함

**w_customer_list의 Open 이벤트**
```powerbuilder
// Resize 서비스 활성화 (컨트롤 자동 크기 조절)
this.of_SetResize(TRUE)

// DataWindow 서비스 활성화
dw_list.of_SetTransObject(SQLCA)
dw_list.of_SetBase(TRUE)           // 기본 DataWindow 서비스
dw_list.of_SetRowManager(TRUE)     // 행 관리 서비스
dw_list.of_SetSort(TRUE)           // 정렬 서비스
dw_list.of_SetFilter(TRUE)         // 필터 서비스
dw_list.of_SetFind(TRUE)           // 찾기 서비스

// 정렬 서비스 스타일 설정 (드롭다운 스타일)
dw_list.inv_Sort.of_SetStyle(dw_list.inv_Sort.DROPDOWNLISTBOX)

// 데이터 조회
this.PostEvent("ue_retrieve")      // 메시지 라우터를 통해 조회 이벤트 호출
```

#### 메뉴와 메시지 라우터

PFC는 메뉴와 윈도우 간 통신을 위해 **메시지 라우터(Message Router)**를 사용합니다.

**메뉴 아이템 Clicked 이벤트**
```powerbuilder
// 메뉴 아이템에서 시트 열기
n_cst_menu lnv_menu
Message.StringParm = "w_customer_list"
lnv_menu.of_SendMessage(this, "pfc_Open")
```

**프레임 윈도우의 pfc_Open 이벤트**
```powerbuilder
// w_frame의 pfc_Open 이벤트
String ls_sheet_name
w_sheet lw_sheet

ls_sheet_name = Message.StringParm
IF ls_sheet_name <> "" THEN
   OpenSheet(lw_sheet, ls_sheet_name, this, 0, Layered!)
END IF
```

**데이터 조회 요청**
```powerbuilder
// 메뉴에서 조회 명령
n_cst_menu lnv_menu
lnv_menu.of_SendMessage(this, "pfc_Retrieve")

// 윈도우의 pfc_Retrieve 이벤트
dw_list.Retrieve()
```

### 2.7 PFC 개발 시 주의사항

| 주의사항 | 설명 |
|---------|------|
| **gnv_app 변수명** | 반드시 `gnv_app`이라는 이름을 사용해야 PFC 객체가 인식 |
| **트랜잭션 객체 타입** | SQLCA의 타입을 `n_tr`로 변경해야 PFC 트랜잭션 서비스 사용 가능 |
| **별도 라이브러리 유지** | 각 애플리케이션은 자체 확장 라이브러리(PFE)를 가지고 있어야 함. 공유 시 문제 발생 |
| **상속 깊이** | 과도한 상속은 성능 저하를 유발할 수 있으므로 적절한 깊이 유지 |
| **서비스 선택적 활성화** | 필요한 서비스만 활성화하여 성능 최적화 |

### 2.8 PFC 확장 예제: 사용자 정의 데이터윈도우 서비스

```powerbuilder
// n_cst_dwsrv_excel (데이터윈도우 엑셀 내보내기 서비스)
// pfc_n_cst_dwsrv에서 상속

// of_export_to_excel 함수
FUNCTION Boolean of_export_to_excel (String as_filename)
   DataStore lds_temp
   lds_temp = CREATE DataStore
   lds_temp.DataObject = this.GetDataObject()
   lds_temp.SetTransObject(SQLCA)
   lds_temp.Retrieve()
   
   // DataStore를 엑셀 파일로 저장
   lds_temp.SaveAs(as_filename, Excel!, TRUE)
   
   DESTROY lds_temp
   RETURN TRUE
END FUNCTION
```

**사용자 정의 데이터윈도우 객체에서 서비스 등록**
```powerbuilder
// u_dw_excel (u_dw 상속)
// of_SetExcelService 함수
FUNCTION Boolean of_SetExcelService (Boolean ab_activate)
   IF ab_activate THEN
      This.inv_excel = CREATE n_cst_dwsrv_excel
      This.inv_excel.of_Register(This)
      RETURN TRUE
   ELSE
      DESTROY This.inv_excel
      RETURN TRUE
   END IF
END FUNCTION
```

---

## 3. 실전 통합 예제: 웹 서비스를 활용한 PFC 애플리케이션

다음은 외부 REST API에서 고객 정보를 가져와 PFC 기반 애플리케이션에서 표시하는 예제입니다.

### 3.1 웹 서비스 통신 객체 생성

**n_rest_client** 사용자 객체 생성:
```powerbuilder
// n_rest_client (비시각적 객체)
// 인스턴스 변수
PRIVATE:
   HttpClient inv_http
   String is_base_url

// Constructor 이벤트
inv_http = CREATE HttpClient
is_base_url = "https://api.example.com"

// of_get_customer 함수 (REST GET 호출)
FUNCTION String of_get_customer (String as_cust_id)
   String ls_url, ls_response
   Integer li_rtn
   
   ls_url = is_base_url + "/customers/" + as_cust_id
   li_rtn = inv_http.SendRequest("GET", ls_url)
   IF li_rtn = 1 AND inv_http.GetResponseStatusCode() = 200 THEN
       inv_http.GetResponseBody(ls_response)
       RETURN ls_response
   ELSE
       RETURN ""
   END IF
END FUNCTION

// Destructor 이벤트
DESTROY inv_http
```

### 3.2 PFC 애플리케이션에 통합

**n_cst_appmanager 확장**:
```powerbuilder
// n_appmanager (n_cst_appmanager 상속)
// 인스턴스 변수
PUBLIC:
   n_rest_client inv_rest

// Constructor 이벤트 추가
inv_rest = CREATE n_rest_client
```

**시트 윈도우에서 사용**:
```powerbuilder
// w_customer_detail의 Open 이벤트
// 전달받은 고객 ID로 REST API 호출
String ls_cust_id, ls_json
ls_cust_id = Message.StringParm
ls_json = gnv_app.inv_rest.of_get_customer(ls_cust_id)

// JSON 파싱 후 화면에 표시
JSONParser lnv_json
lnv_json = CREATE JSONParser
String ls_error = lnv_json.LoadString(ls_json)
IF ls_error = "" THEN
   Long ll_root = lnv_json.GetRootItem()
   sle_id.Text = lnv_json.GetItemString(ll_root, "id")
   sle_name.Text = lnv_json.GetItemString(ll_root, "name")
   sle_email.Text = lnv_json.GetItemString(ll_root, "email")
END IF

DESTROY lnv_json
```

---

## 결론

웹 서비스 연동과 PFC 프레임워크는 현대적인 PowerBuilder 애플리케이션 개발의 핵심 기술입니다.

- **SOAP/REST 웹 서비스**를 통해 외부 시스템과 데이터를 교환하고, 기존 시스템을 현대적인 아키텍처로 통합할 수 있습니다. 특히 REST 연동 시 `HttpClient`의 `SendRequest()`와 `JSONParser`의 핸들 기반 접근법을 정확히 사용해야 합니다.
- **PFC 프레임워크**는 검증된 객체 지향 구조를 제공하여 대규모 프로젝트의 생산성과 유지보수성을 크게 향상시킵니다. 특히 `n_cst_appmanager`를 중심으로 한 애플리케이션 생명주기 관리, 메시지 라우터를 통한 객체 간 통신, 다양한 서비스를 통한 기능 확장은 PFC의 핵심 가치입니다.

PFC 기반 프로젝트에서는 **상속 구조를 이해하고, 필요한 서비스를 선택적으로 활성화하며, 확장 계층(PFE)에 사용자 정의 코드를 추가**하는 방식으로 개발해야 합니다. 초기 학습 곡선이 있지만, 일단 익숙해지면 매우 효율적인 개발이 가능합니다.

다음 글에서는 PowerBuilder 애플리케이션의 **배포(Deployment)와 패키징**, 그리고 성능 최적화 기법에 대해 알아보겠습니다.