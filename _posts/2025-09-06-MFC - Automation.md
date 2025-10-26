---
layout: post
title: MFC - Automation
date: 2025-09-06 21:25:23 +0900
category: MFC
---
# Automation(Dispatch) 개요와 스크립팅 통합 아이디어  
(MFC/Win32/ATL 기준: `IDispatch` 늦은 바인딩, Type Library, 이벤트 싱크, Active Scripting/PowerShell/Python 연동까지)

이 글은 **COM Automation(IDispatch)**의 동작 원리와 **애플리케이션을 스크립팅 가능**하게 만드는 전체적인 설계·구현 패턴을 정리합니다.  
MFC/ATL에서 **클라이언트로서 외부 앱(예: Excel)을 자동화**하는 방법과, **서버로서 우리 앱을 노출**하는 방법, 그리고 **스크립팅 엔진(VBScript/JScript/PowerShell/Python)**과 **양방향 통합** 아이디어까지 예제와 함께 다룹니다.

> 전제: C++17, Unicode, Win32/COM 기초, x86/x64. 예제 코드는 모두 ``` 로 감싸서 제공합니다.

---

## 0) 큰 그림: Automation이 뭐고, 왜 쓰는가?

- **COM Automation** = 인터페이스 `IDispatch` 기반의 **늦은 바인딩** 모델.  
  - **클라이언트**: `IDispatch::GetIDsOfNames`로 **이름 → DISPID**를 얻고, `Invoke`로 **메서드/프로퍼티 호출**.  
  - **서버**: **Type Library(typelib)**와 `IDispatch` 구현을 통해 **객체 모델**을 외부에 노출.  
- 장점  
  - 언어 중립: VBScript, JScript, VBA, PowerShell, Python(pywin32), C# interop 등에서 호출 가능  
  - **이름 기반**이라 스크립팅이 쉽고, **개체 모델 문서화**와 **인텔리센스**(typelib) 제공 가능  
- 핵심 구성요소  
  - `IDispatch`, `VARIANT`, `BSTR`, `SAFEARRAY`, `ITypeInfo`  
  - 등록정보: **ProgID**, **CLSID**, **AppID**, **ThreadingModel**  
  - **Connection Point**(이벤트 소스/싱크), **Active Scripting** 호스팅(선택)

---

## 1) 클라이언트(Consumer)로 쓰기: 외부 앱 자동화

### 1-1. COM 초기화와 아파트먼트
- 기본: **STA**(Single-Threaded Apartment) 권장 — 대부분의 Office/스크립팅은 STA 가정  
- `CoInitializeEx(nullptr, COINIT_APARTMENTTHREADED);` + 스레드 종료 시 `CoUninitialize();`

```cpp
#include <comutil.h>   // _bstr_t, _variant_t (link: comsuppw.lib)
#pragma comment(lib, "comsuppw.lib")

struct ComInit {
    ComInit()  { CoInitializeEx(nullptr, COINIT_APARTMENTTHREADED); }
    ~ComInit() { CoUninitialize(); }
};
```

### 1-2. #import 기반(조기 바인딩/래퍼 생성) — Excel 예제
`#import`는 Type Library로부터 **스마트 포인터**와 **래퍼**를 생성합니다.

```cpp
#import "progid:Excel.Application" rename("DialogBox", "ExcelDialogBox") \
    rename("RGB", "ExcelRGB") no_auto_exclude
// 또는 #import "C:\\Program Files\\Microsoft Office\\root\\Office16\\EXCEL.EXE"

void AutomateExcelEarlyBinding() {
    ComInit cx; // STA
    Excel::ApplicationPtr app;  app.CreateInstance(L"Excel.Application");
    app->Visible[0] = VARIANT_TRUE;

    Excel::WorkbooksPtr books = app->Workbooks;
    Excel::WorkbookPtr  book  = books->Add(Excel::xlWorksheet);
    Excel::WorksheetPtr sheet = app->ActiveSheet;

    sheet->Cells->Item[1][1] = _variant_t(L"Hello");
    sheet->Cells->Item[1][2] = _variant_t(123.45);

    _bstr_t path = L"C:\\temp\\demo.xlsx";
    book->SaveAs(path);
    // app->Quit();  // 사용 후 종료(필요 시)
}
```

장점: 타입 안전/인텔리센스/간결. 단점: **타입/버전 의존성**이 있고 배포 시 typelib 접근이 필요.

### 1-3. 늦은 바인딩(IDispatch 직접 호출) — Excel 예제
모든 스크립팅 언어와 동일한 경로. **버전 변화에 강함**.

```cpp
HRESULT AutoExcelLateBinding() {
    ComInit cx;
    CLSID cls; CLSIDFromProgID(L"Excel.Application", &cls);
    CComPtr<IDispatch> app;
    RETURN_IF_FAILED(CoCreateInstance(cls, nullptr, CLSCTX_LOCAL_SERVER, IID_PPV_ARGS(&app)));

    auto GetDispID = [&](IDispatch* p, LPCOLESTR name, DISPID* pid) {
        return p->GetIDsOfNames(IID_NULL, const_cast<LPOLESTR*>(&name), 1, LOCALE_USER_DEFAULT, pid);
    };
    auto PutProperty = [&](IDispatch* p, LPCOLESTR name, VARIANT v) {
        DISPID dispid, dispidPut = DISPID_PROPERTYPUT;
        GetDispID(p, name, &dispid);
        DISPPARAMS dp{ &v, &dispidPut, 1, 1 };
        return p->Invoke(dispid, IID_NULL, LOCALE_USER_DEFAULT, DISPATCH_PROPERTYPUT, &dp, nullptr, nullptr, nullptr);
    };
    auto Call = [&](IDispatch* p, LPCOLESTR name, VARIANT* args, UINT argc, VARIANT* ret = nullptr) {
        DISPID dispid; GetDispID(p, name, &dispid);
        DISPPARAMS dp{ args, nullptr, argc, 0 };
        return p->Invoke(dispid, IID_NULL, LOCALE_USER_DEFAULT, DISPATCH_METHOD, &dp, ret, nullptr, nullptr);
    };

    // Visible = true
    VARIANT vTrue; VariantInit(&vTrue); vTrue.vt = VT_BOOL; vTrue.boolVal = VARIANT_TRUE;
    PutProperty(app, L"Visible", vTrue);

    // Workbooks = app.Workbooks
    DISPID idWorkbooks; GetDispID(app, L"Workbooks", &idWorkbooks);
    VARIANT res; VariantInit(&res);
    DISPPARAMS noargs{}; app->Invoke(idWorkbooks, IID_NULL, LOCALE_USER_DEFAULT, DISPATCH_PROPERTYGET, &noargs, &res, nullptr, nullptr);
    IDispatch* workbooks = res.pdispVal;

    // book = Workbooks.Add()
    VariantClear(&res);
    Call(workbooks, L"Add", nullptr, 0, &res);
    IDispatch* book = res.pdispVal;

    // sheet = app.ActiveSheet
    VariantClear(&res);
    DISPID idActiveSheet; GetDispID(app, L"ActiveSheet", &idActiveSheet);
    app->Invoke(idActiveSheet, IID_NULL, LOCALE_USER_DEFAULT, DISPATCH_PROPERTYGET, &noargs, &res, nullptr, nullptr);
    IDispatch* sheet = res.pdispVal;

    // sheet.Cells.Item(1,1) = "Hello"
    VARIANT vCellArgs[2]; // 역순: COM Invoke는 디폴트가 오른쪽부터 push
    vCellArgs[1] = _variant_t(1L);
    vCellArgs[0] = _variant_t(1L);
    VariantInit(&res);
    Call(sheet, L"Cells", vCellArgs, 2, &res);
    IDispatch* cell = res.pdispVal;

    DISPID idItem; GetDispID(cell, L"Item", &idItem);
    // Item은 이미 (1,1)로 가져왔으니 바로 Value2 설정 가능
    VARIANT vHello; vHello = _variant_t(L"Hello");
    DISPID dispidPut = DISPID_PROPERTYPUT;
    DISPPARAMS dpPut{ &vHello, &dispidPut, 1, 1 };
    DISPID idValue2; GetDispID(cell, L"Value2", &idValue2);
    cell->Invoke(idValue2, IID_NULL, LOCALE_USER_DEFAULT, DISPATCH_PROPERTYPUT, &dpPut, nullptr, nullptr, nullptr);

    // 정리
    cell->Release(); sheet->Release(); book->Release(); workbooks->Release(); 
    return S_OK;
}
```

핵심:  
- **PROPERTYGET/PUT/PUTREF** 구분, `DISPID_PROPERTYPUT` 네임드 인수 사용  
- 인수 배열은 **오른쪽→왼쪽** 순서로 배치  
- 문자열은 **`BSTR`**( `_bstr_t` 사용 권장 ), 수치/논리는 **`VARIANT`**로 포장

### 1-4. SAFEARRAY(배열) 전달 예
```cpp
// 2x2 배열을 Excel Range.Value2에 한 번에 대입
SAFEARRAYBOUND bnd[2] = { {2,1},{2,1} }; // [row, col], lLbound=1
SAFEARRAY* psa = SafeArrayCreate(VT_VARIANT, 2, bnd);

LONG idx[2];
VARIANT v;
idx[0]=1; idx[1]=1; v = _variant_t(L"A"); SafeArrayPutElement(psa, idx, &v);
idx[0]=1; idx[1]=2; v = _variant_t(10);   SafeArrayPutElement(psa, idx, &v);
idx[0]=2; idx[1]=1; v = _variant_t(L"B"); SafeArrayPutElement(psa, idx, &v);
idx[0]=2; idx[1]=2; v = _variant_t(20);   SafeArrayPutElement(psa, idx, &v);

VARIANT arr; VariantInit(&arr); arr.vt = VT_ARRAY|VT_VARIANT; arr.parray = psa;

// Range("A1:B2").Value2 = arr;  (Late Binding일 때 Invoke로 PUT)
```

---

## 2) 서버(Provider)로 노출: 우리 앱을 자동화 가능하게 만들기

### 2-1. MFC OLE Automation(레거시) 빠른 길
- `AfxOleInit()`, `DECLARE_DISPATCH_MAP`, `BEGIN_DISPATCH_MAP` 매크로 기반  
- `COleDispatchDriver`/`CCmdTarget` 파생으로 **메서드/프로퍼티**를 스크립트에 노출

```cpp
class CMyAuto : public CCmdTarget {
    DECLARE_DYNCREATE(CMyAuto)
    DECLARE_DISPATCH_MAP()
    DECLARE_INTERFACE_MAP()
public:
    CMyAuto() { EnableAutomation(); }
    afx_msg BSTR GetVersion();
    afx_msg void  DoWork(LPCTSTR path, long flags);
};

BEGIN_DISPATCH_MAP(CMyAuto, CCmdTarget)
    DISP_PROPERTY_EX_ID(CMyAuto, "Version", 1, GetVersion, nullptr, VT_BSTR)
    DISP_FUNCTION_ID(CMyAuto, "DoWork", 2, DoWork, VT_EMPTY, VTS_BSTR VTS_I4)
END_DISPATCH_MAP()

BSTR CMyAuto::GetVersion() { return ::SysAllocString(L"1.2.3"); }
void CMyAuto::DoWork(LPCTSTR path, long flags) { /* ... */ }
```

- **등록/ProgID**: `COleObjectFactory`와 리소스 문자열 기반으로 CLSID/ProgID 등록  
- 장점: 빠르고 간단. 단점: 근대적 툴링/IDL 제약, 대규모 모델엔 한계

### 2-2. ATL + `IDispatchImpl` (권장)
- **IDL(Type Library)** 기반으로 명확한 계약 제공 → IntelliSense/문서화/상호 운용성 ↑

#### (A) IDL 정의 (샘플)
```idl
// MyApp.idl
import "oaidl.idl";
import "ocidl.idl";

[uuid(XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX), version(1.0)]
library MyAppLib {
  importlib("stdole2.tlb");

  [uuid(AAAAAAAA-AAAA-AAAA-AAAA-AAAAAAAAAAAA)]
  dispinterface IApp {
    properties:
      [id(1), propget] BSTR Version;
    methods:
      [id(2)] void DoWork([in] BSTR path, [in, defaultvalue(0)] LONG flags);
      [id(3)] VARIANT GetStats();
  };

  [uuid(BBBBBBBB-BBBB-BBBB-BBBB-BBBBBBBBBBBB)]
  coclass App {
    [default] dispinterface IApp;
  };
};
```

#### (B) C++ 클래스(ATL)
```cpp
class ATL_NO_VTABLE CApp :
  public CComObjectRootEx<CComSingleThreadModel>,
  public CComCoClass<CApp, &CLSID_App>,
  public IDispatchImpl<IApp, &IID_IApp, &LIBID_MyAppLib, /*wMajor*/1, /*wMinor*/0>
{
public:
  STDMETHOD(get_Version)(BSTR* p) { *p = SysAllocString(L"1.2.3"); return S_OK; }
  STDMETHOD(DoWork)(BSTR path, LONG flags) { /* ... */ return S_OK; }
  STDMETHOD(GetStats)(VARIANT* pv) {
      CComSafeArray<BSTR> sa(2);
      sa.SetAt(0, SysAllocString(L"ok"));
      sa.SetAt(1, SysAllocString(L"42 items"));
      pv->vt = VT_ARRAY|VT_BSTR; pv->parray = sa.Detach();
      return S_OK;
  }

  DECLARE_NO_REGISTRY() // 또는 RGS 파일 사용
  BEGIN_COM_MAP(CApp)
    COM_INTERFACE_ENTRY(IApp)
    COM_INTERFACE_ENTRY(IDispatch)
  END_COM_MAP()
};
OBJECT_ENTRY_AUTO(__uuidof(App), CApp)
```

- **레지스트레이션**: MSI/`regsvr32`/RGS로 CLSID/ProgID/TypeLib 등록  
- **ThreadingModel=Apartment** 지정(STA)

### 2-3. 이벤트(Outbound) — Connection Point / `IDispatch` 이벤트 싱크
서버가 **이벤트**를 내보내면(예: 진행률, 완료), 스크립트가 핸들링 가능.

#### ATL 서버: 이벤트 소스
- IDL에 `dispinterface _IAppEvents` 정의 → coclass에 `uuid, source`로 연결  
- 코드에서 `IDispEventSimpleImpl` 또는 `IConnectionPointContainer`를 통해 **Fire_OnProgress(percent)** 등 호출

```idl
dispinterface _IAppEvents {
  methods:
    [id(1)] void OnProgress([in] LONG percent);
    [id(2)] void OnDone([in] VARIANT_BOOL ok);
};

coclass App {
  [default] dispinterface IApp;
  [source]  dispinterface _IAppEvents;
};
```

```cpp
// ATL: CProxy_IAppEvents (wizards가 생성) → Fire_OnProgress(…)
m_vec; // IConnectionPoint::EnumConnections 를 통해 구독자 나열
```

#### 클라이언트(ATL)에서 이벤트 싱크
```cpp
class CAppSink : public IDispEventSimpleImpl<1, CAppSink, &__uuidof(_IAppEvents)> {
public:
  BEGIN_SINK_MAP(CAppSink)
    SINK_ENTRY_INFO(1, __uuidof(_IAppEvents), 1 /*OnProgress*/, OnProgress, &__uuidof(VT_I4))
    SINK_ENTRY_INFO(1, __uuidof(_IAppEvents), 2 /*OnDone*/,     OnDone,     &__uuidof(VT_BOOL))
  END_SINK_MAP()

  void __stdcall OnProgress(LONG pct) { /* ... */ }
  void __stdcall OnDone(VARIANT_BOOL ok) { /* ... */ }
};

// 연결
CComPtr<IApp> app; app.CoCreateInstance(CLSID_App);
CAppSink sink; AtlAdvise(app, &sink, __uuidof(_IAppEvents), &cookie);
// … 해제 시 AtlUnadvise
```

---

## 3) 스크립팅 엔진과의 통합

### 3-1. Windows Script Host(WSH)에서 우리 객체 사용 (VBScript/JScript)

#### VBScript 예 (파일: `test.vbs`)
```vb
Dim app : Set app = CreateObject("MyApp.App")
WScript.Echo app.Version
app.DoWork "C:\temp\in.txt", 1
Dim s : s = app.GetStats()
WScript.Echo s(0) & " / " & s(1)
```

#### PowerShell 예
```powershell
$app = New-Object -ComObject "MyApp.App"
$app.Version
$app.DoWork("C:\temp\in.txt", 1)
$stats = $app.GetStats()
$stats
```

#### Python(pywin32) 예
```python
import win32com.client as win32
app = win32.Dispatch("MyApp.App")
print(app.Version)
app.DoWork(r"C:\temp\in.txt", 1)
print(app.GetStats())
```

> 핵심: **ProgID**가 등록되어 있고, **Type Library**가 있으면 언어 측 IntelliSense/도움말에도 우호적.

### 3-2. 우리 앱이 **스크립트 엔진을 호스팅**(Active Scripting)

- COM 인터페이스: `IActiveScript`, `IActiveScriptParse`  
- VBS/JScript 엔진을 로드하여 **호스트 객체**(우리 `IApp`)를 Script에 주입 → 스크립트에서 `App.DoWork` 호출 가능

#### 최소 예(개념 코드)
```cpp
CComPtr<IActiveScript>       engine;
CComPtr<IActiveScriptParse>  parse;
CLSID clsid; CLSIDFromProgID(L"VBScript", &clsid);
engine.CoCreateInstance(clsid);
engine->QueryInterface(&parse);
parse->InitNew();

// 호스트 객체 주입: IActiveScriptSite 구현 필요 (변수명 "App")
CComPtr<IActiveScriptSite> site = new CMyScriptSite(/* exposes IDispatch* App */);
engine->SetScriptSite(site);

// 전역 코드 실행
parse->ParseScriptText(L"App.DoWork \"C:\\temp\\a.txt\", 0", nullptr, nullptr, nullptr, 0, 0, 0, nullptr, nullptr);
engine->SetScriptState(SCRIPTSTATE_CONNECTED);
```

- 장점: **내장 매크로 엔진**처럼 사용자 스크립트를 실행  
- 주의: 보안(샌드박스), 장기적으로 **Windows Active Scripting(JScript/VBScript) 폐기 이슈** 고려 → **PowerShell/JS(ChakraCore)/Lua/Python 임베딩** 대안 검토

### 3-3. PowerShell 호스팅(권장 대안)
- .NET Hosting or External PowerShell 호출, **COM 객체를 PS에 노출**  
- 스크립트 실행 시 **App**을 자동 바인딩해주면 친화적인 개발자 경험 제공

---

## 4) 설계: “스크립팅 가능한 애플리케이션”의 객체 모델

### 4-1. 오브젝트 트리 예(도면 앱 가정)
```
Application
 ├─ Documents (collection)
 │   └─ Document
 │       ├─ Shapes (collection)
 │       │   └─ Shape (methods: Move, Rotate, FillColor, …)
 │       └─ Selection
 └─ Dialogs / UI helpers
```

- **컬렉션 패턴**: `Count`, `Item(index)`, **_NewEnum**(DISPID = -4)로 **For Each** 지원
- **프로퍼티**: `Name`, `Path`, `ActiveDocument`
- **메서드**: `Open(path)`, `SaveAs(path)`, `ExportPng(path, dpi)`

```idl
dispinterface IDocuments {
  methods:
    [id(1)] long Count();
    [id(2)] IDispatch* Item([in] VARIANT index);
    [id(DISPID_NEWENUM), propget] IUnknown* _NewEnum(); // IEnumVARIANT
};
```

### 4-2. 이름있는/옵셔널 인수
- `DISPID_PROPERTYPUT`, `DISPID_PROPERTYPUTREF`, `DISPATCH_PROPERTYPUT`  
- IDL에서 `defaultvalue`, `optional` 지정 → VBScript/PowerShell에서 자연스럽게 사용

```idl
[id(10)] void Export([in] BSTR path, [in, optional, defaultvalue(300)] LONG dpi);
```

### 4-3. 에러 전달
- 실패 시 `HRESULT` + `EXCEPINFO` 채움 → 스크립트에서 **런타임 오류**로 인식  
- 의미 있는 **에러 코드/메시지/소스** 제공

---

## 5) 메모리/형 변환 규칙(실무 필수)

- 문자열: **BSTR** ( `SysAllocString` / `_bstr_t` )  
- 값: **VARIANT** ( `_variant_t` )  
- 배열: **SAFEARRAY** / `CComSafeArray`  
- `VARIANT_BOOL`: `VARIANT_TRUE/VARIANT_FALSE` (C++ `bool`과 다른 타입)  
- 스레딩: 대부분 **STA**. 워커에서 호출하려면 **`CoMarshalInterThreadInterfaceInStream`**로 마샬링

---

## 6) 배포/레지스트리/64비트

- ProgID → CLSID 매핑: `HKCR\MyApp.App\CLSID`  
- CLSID → 서버: `HKCR\CLSID\{...}\LocalServer32`(EXE) / `InprocServer32`(DLL)  
- TypeLib 등록: `HKCR\TypeLib\{...}`  
- **x86/x64 분리**: 32비트 Office 자동화는 32비트 프로세스 필요. 가능하면 **아키텍처 일치**  
- 권한: 등록은 관리자 필요(MSI 권장)

---

## 7) 이벤트/콜백(스크립트→앱, 앱→스크립트) 활용 아이디어

- **진행률 이벤트**: 긴 작업 중 `OnProgress(pct)` 발행 → 스크립트에서 UI/로그로 반영  
- **취소 토큰**: 스크립트가 `App.Cancel` 호출 → 앱에서 작업 중단  
- **데이터 파이프**: 앱이 **IStream을 반환**하여 스크립트에서 바로 읽게 (파일 I/O 피함)

---

## 8) 보안/안전 가이드

- 자동화는 사실상 **원격 코드 실행과 유사**: 신뢰할 수 있는 스크립트만 허용  
- 샌드박스: 제한된 오브젝트만 노출, 파일시스템/레지스트리 접근 최소화  
- 서명된 스크립트, 정책 기반 허용 목록(경로/해시)  
- COM ACL/DCOM 설정(원격 자동화 시)

---

## 9) 문제해결/디버깅 팁

- `oleview.exe`(OLE/COM Object Viewer)로 **TypeLib 확인**  
- 스크립트 오류 → **EXCEPINFO** 내용과 `HRESULT` 로깅  
- `IDispatch::Invoke`에 들어온 **`DISPID`/인수 VT**를 트레이스  
- 0x80040154(클래스 미등록), 0x80029C4A(typelib 문제) 흔함  
- 이벤트가 안 옴 → `AtlAdvise/Unadvise` 성공/쿠키/아파트 확인

---

## 10) “양방향 샘플” — 우리 앱을 노출하고 PowerShell에서 자동화

### 10-1. 서버(IDL/ATL) — 핵심만
```idl
dispinterface IApp {
  properties:
    [id(1), propget] BSTR Version;
  methods:
    [id(2)] void DoWork([in] BSTR path, [in, defaultvalue(0)] LONG flags);
    [id(3)] void Cancel();
};
dispinterface _IAppEvents {
  methods:
    [id(1)] void OnProgress([in] LONG pct, [in] BSTR msg);
    [id(2)] void OnDone([in] VARIANT_BOOL ok);
};
coclass App {
  [default] dispinterface IApp;
  [source]  dispinterface _IAppEvents;
};
```

```cpp
class CApp :
  public CComObjectRootEx<CComSingleThreadModel>,
  public CComCoClass<CApp, &CLSID_App>,
  public IDispatchImpl<IApp, &IID_IApp, &LIBID_MyAppLib, 1, 0>,
  public IConnectionPointContainerImpl<CApp>,
  public CProxy_IAppEvents<CApp> // ATL 이벤트 프록시
{
    CEvent m_cancel;
public:
    STDMETHOD(get_Version)(BSTR* p) { *p = SysAllocString(L"2.0"); return S_OK; }
    STDMETHOD(DoWork)(BSTR path, LONG flags) {
        ResetEvent(m_cancel);
        for (int i=0;i<=100;++i) {
            if (WaitForSingleObject(m_cancel, 0)==WAIT_OBJECT_0) break;
            Fire_OnProgress(i, CComBSTR(L"working..."));
            Sleep(30);
        }
        Fire_OnDone(VARIANT_TRUE);
        return S_OK;
    }
    STDMETHOD(Cancel)(){ SetEvent(m_cancel); return S_OK; }

    BEGIN_COM_MAP(CApp)
      COM_INTERFACE_ENTRY(IApp)
      COM_INTERFACE_ENTRY(IDispatch)
      COM_INTERFACE_ENTRY(IConnectionPointContainer)
    END_COM_MAP()
    BEGIN_CONNECTION_POINT_MAP(CApp)
      CONNECTION_POINT_ENTRY(__uuidof(_IAppEvents))
    END_CONNECTION_POINT_MAP()
};
```

### 10-2. PowerShell 클라이언트
```powershell
$app = New-Object -ComObject "MyApp.App"
# 이벤트 구독(등록형식은 PS 버전에 따라)
Register-ObjectEvent -InputObject $app -EventName OnProgress -Action {
    param($sender,$e)  # COM dispid 기반으로 args가 들어옴(PS 버전에 따라 다름)
    Write-Host ("Progress: " + $EventArgs[0] + " " + $EventArgs[1])
}
Register-ObjectEvent -InputObject $app -EventName OnDone -Action {
    Write-Host "Done!"
}
$app.DoWork("C:\temp\in.txt", 0)
```

---

## 11) 통합 아이디어(제품화 관점)

1. **스크립트 콘솔** 내장:  
   - PowerShell 또는 JS/Lua 콘솔 패널, **App** 객체 바인딩, 즉시 실행/히스토리  
2. **매크로 기록기**:  
   - 사용자의 UI 조작을 **Automation 호출 시퀀스**로 기록 → 스크립트로 내보내기  
3. **작업 파이프라인**:  
   - 스크립트로 **배치 처리/스케줄러** 실행 (타이머/작업 예약 가이드 참고)  
4. **플러그인 시스템**:  
   - COM 자동화 + 스크립트 모듈로 기능 확장. TypeLib로 문서화/인텔리센스 제공  
5. **테스트 자동화**:  
   - 기능 검증을 PowerShell 스크립트로 돌려 **회귀 테스트**에 활용

---

## 12) 체크리스트 요약

- [ ] `CoInitializeEx(COINIT_APARTMENTTHREADED)` / STA  
- [ ] 서버: **IDL(TypeLib)**, `IDispatchImpl`, `ConnectionPoint`(이벤트)  
- [ ] 클라: #import(조기)/`IDispatch::Invoke`(늦은) 모두 이해  
- [ ] **BSTR/VARIANT/SAFEARRAY** 메모리 규칙  
- [ ] `HRESULT` + `EXCEPINFO` 오류 보고  
- [ ] ProgID/CLSID/TypeLib **등록**(x86/x64 구분)  
- [ ] 보안/샌드박스/허용 목록 정책  
- [ ] 문서화(개체 모델, 예제, 샘플 스크립트)

---

### 마무리

Automation은 **앱을 하나의 플랫폼**으로 바꿉니다.  
`IDispatch`와 TypeLib를 중심으로 **객체 모델을 설계**하고, **이벤트/스크립팅 호스팅**을 더하면  
사용자는 PowerShell/VBScript/Python 등으로 손쉽게 **자동화·배치·통합**을 구현할 수 있습니다.
