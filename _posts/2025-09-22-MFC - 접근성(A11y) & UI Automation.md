---
layout: post
title: MFC - 접근성(A11y) & UI Automation
date: 2025-09-22 15:25:23 +0900
category: MFC
---
# 접근성(A11y) & UI Automation 실전 가이드
**UIA 패턴 노출 · High-Contrast/컬러 필터 대응 · 키보드 내비 · 스크린리더(내레이터) 호환 체크리스트**


## 0. 큰 그림: “표준 컨트롤 + 올바른 이름 + 키보드 + 시스템 색상”이 80%를 만든다

- **표준/공용 컨트롤 사용**: `BUTTON`, `EDIT`, `STATIC`, `LISTVIEW`, `TREEVIEW`, `REBAR`, `RIBBON(MFC Feature Pack)` 등은 OS가 **IAccessible/UIA Provider**를 내장합니다.  
  → **먼저 표준 컨트롤로 구성**하고, 스타일/테마만 얹어도 기본 접근성은 확보됩니다.
- **이름/역할**: 텍스트 레이블과 컨트롤을 **프로그램적으로 연결** (`LabelFor/LabeledBy`), 접근키(Access Key, `&`), `AutomationId`(식별자) 제공.
- **키보드 내비**: `WS_TABSTOP`/`WS_GROUP`, 포커스 표시(`DrawFocusRect`), 가속기/단축키.  
- **시스템 색상/High-Contrast**: 색상 하드코딩 금지. `GetSysColor/UxTheme` 사용, `SPI_GETHIGHCONTRAST` 대응.  
- **커스텀 컨트롤**: 필요할 때만 UIA Provider를 구현(패턴 최소화: `Invoke`, `Value`, `RangeValue`, `Selection`, `Toggle`, `Text`, `Scroll` 등).

---

## 1. UI Automation 핵심 용어 5분 요약

- **UIA 트리**: Control View / Content View / Raw View(세분화). 스크린리더는 **Control/Content** 중심으로 순회.  
- **ControlType**: `Button`, `Edit`, `List`, `ListItem`, `Tree`, `Menu`… **역할(Role)**.  
- **필수 속성**: `Name`, `AutomationId`, `ControlType`, `IsEnabled`, `BoundingRectangle`, `FrameworkId`.  
- **패턴(Pattern)**: 상호작용 계약. 예) `Invoke`, `Value`, `RangeValue`, `Selection(Item)`, `Toggle`, `ExpandCollapse`, `Text`, `Grid`, `Table`, `Scroll`, `Transform`, `Window`, `VirtualizedItem`, `ItemContainer`, `SynchronizedInput`, `Annotation`, `Spreadsheet` 등.  
- **이벤트**: `AutomationFocusChanged`, `PropertyChanged`, `StructureChanged`, `Invoke_Invoked`, `Text_TextSelectionChanged` 등.

---

## 2. “기본기”부터: 이름, 라벨, 접근키(Access Key), 탭 순서

### 2.1 레이블-컨트롤 연결 (Win32)
```cpp
// 리소스 예시 (Dialog)
CONTROL "이름(&N):", IDC_LBL_NAME, "STATIC", WS_CHILD|WS_VISIBLE, 10,10,60,12
EDITTEXT               IDC_EDIT_NAME,                   80,8,160,14, ES_AUTOHSCROLL

// 코드: LABEL의 텍스트에서 &로 접근키 지정(Alt+N)
// 내레이터가 "이름 편집" 처럼 읽게 만들려면, 레이블과 에디트를 논리적으로 연결
// Win32 순정은 자동 연결이 약하므로 UIA Provider 또는 IAccessibleEx를 쓰는 게 정석이지만,
// 최소한 NAME을 에디트에 직접 설정하는 것도 방법:
SetWindowTextW(GetDlgItem(hDlg, IDC_EDIT_NAME), L""); // 사용자 텍스트 영역
// 에디트에 접근성 Name을 강제하려면 UIA Provider를 쓰거나, 별도로 "placeholder"가 Name을 덮지 않도록 주의.
```

> **원칙**  
> - 실제 입력 대상(에디트)의 **Name**이 비어 있지 않게 하라. 레이블 텍스트를 **복제**해 Name으로 노출하면 효과적.  
> - 접근키 `&`를 **레이블**에 넣으면 **Alt+Key**로 해당 레이블의 **다음 포커스 가능 컨트롤**에 포커스가 갑니다.  
> - `WS_TABSTOP`/`WS_GROUP`으로 **Tab 순서**를 자연스럽게 구성.

### 2.2 MFC에서 접근키 예시
```cpp
// 리소스: "이름(&N):" 라벨 + IDC_EDIT_NAME
BOOL CMainDlg::OnInitDialog() {
    CDialogEx::OnInitDialog();
    // Tab 순서에서 라벨 다음에 에디트가 오도록 배치
    // 포커스 표시가 보이도록 스타일/그리기 확인
    return TRUE;
}
```

---

## 3. High-Contrast / 컬러 필터 / 시스템 색상 대응

### 3.1 High-Contrast 감지 & 반응
```cpp
static bool IsHighContrast() {
    HIGHCONTRASTW hc{ sizeof(hc) };
    SystemParametersInfoW(SPI_GETHIGHCONTRAST, sizeof(hc), &hc, 0);
    return (hc.dwFlags & HCF_HIGHCONTRASTON) != 0;
}

LRESULT WndProc(HWND h, UINT m, WPARAM w, LPARAM l) {
    switch (m) {
    case WM_SETTINGCHANGE:
    case WM_THEMECHANGED:
        // 팔레트/브러시 재구성
        InvalidateRect(h, nullptr, TRUE);
        break;
    }
    return DefWindowProc(h, m, w, l);
}
```

- 색상은 **시스템 색상**을 사용: `GetSysColor(COLOR_WINDOW)`, `GetSysColor(COLOR_HIGHLIGHT)`, `GetSysColor(COLOR_WINDOWTEXT)` 등.  
- 커스텀 다크/라이트 테마라도, High-Contrast일 땐 **시스템 팔레트 우선**.

### 3.2 컬러 필터(Win+Ctrl+C) 고려
- 앱이 직접 감지/제어할 API는 제한적. **색 구분에만 의존하지 말 것** (아이콘/패턴/텍스트 보조).  
- 정보전달은 **색 + 텍스트/아이콘/패턴**으로 **冗長성** 확보.

### 3.3 대비(Contrast) 권장
- 최소 **4.5:1**(WCAG AA) 권장. 시스템 팔레트를 따르면 대체로 충족.  
- 임의 색 하드코딩은 **고대비 모드에서 보이지 않을 수 있음** → 시스템 색 기반 브러시/펜/문자색.

---

## 4. 키보드 내비게이션: 탭/화살표/Enter/Esc/공간/메뉴

- `WS_TABSTOP`와 **Tab Order**로 포커스 흐름.  
- 그룹 구분: **첫 컨트롤에 `WS_GROUP`**.  
- 가속기/Access Key: `&` + Alt 조합, 메뉴/툴바/리본과 **일관**.  
- **포커스 표시**: `DrawFocusRect`, `WM_UPDATEUISTATE`(마우스/키보드 포커스 힌트) 대응.  
- **표/리스트**: 화살표/홈/엔드/PgUp/PgDn, Space/Enter 동작 표준화.

```cpp
BOOL CMainDlg::PreTranslateMessage(MSG* p) {
    if (p->message == WM_KEYDOWN && p->wParam == VK_ESCAPE) {
        // Esc: 닫기 금지/확인 등 정책
        return TRUE;
    }
    return CDialogEx::PreTranslateMessage(p);
}
```

---

## 5. “이름/역할/상태”를 바르게: UIA Name/AutomationId/HelpText/ItemStatus

- **Name**: 시각적 라벨과 동일/근접.  
- **AutomationId**: **자동화 스크립트**가 요소를 찾을 수 있는 **안정된 ID**(리소스 ID/고정 문자열).  
- **HelpText**: 보조 설명(필요 시).  
- **ItemStatus**: 요소 상태 “동기화됨/읽기전용/오프라인” 같은 간단 상태 문자열.

> **금지**: 플래시 메시지/Toaster만 띄우고 **화면 리더에 알리지 않기**.  
> **대안**: **Live Region**(아래)이나 **Notification** API로 내레이터에 **공식 알림**.

---

## 6. UIA Live Region / 알림(Announcement)

### 6.1 UiaRaiseAutomationNotification (Windows 10+)
```cpp
#include <uiautomationcoreapi.h>

// 중요/오류/상태 업데이트 등 알림
void Announce(HWND hwnd, const wchar_t* msg) {
    UiaRaiseAutomationNotification(
        hwnd,
        AutomationNotificationKind_ActionCompleted, // 또는 ActionAborted, Other, …
        AutomationNotificationProcessing_MostRecent, // 압축 정책
        msg, wcslen(msg), L"status-area"           // 메시지, 토큰(동일 그룹)
    );
}
```

- **언제?** 진행률 끝, 저장 완료, 오류 발생 등 **비시각적 사용자에게 꼭 알려야 할 때**.  
- 내레이터가 **읽어줌**(현재 포커스 방해 최소).

---

## 7. UI Automation Provider: 커스텀 컨트롤 노출

### 7.1 최소 골격 (Host Raw Provider from HWND)
```cpp
#include <uiautomationcore.h>
#include <wrl.h>
using Microsoft::WRL::ComPtr;

class MyControlProvider :
    public IRawElementProviderSimple,
    public IInvokeProvider
{
    LONG _ref = 1;
    HWND _hwnd = nullptr;

public:
    MyControlProvider(HWND hwnd) : _hwnd(hwnd) {}

    // IUnknown
    IFACEMETHODIMP QueryInterface(REFIID riid, void** pp) override {
        if (!pp) return E_POINTER;
        if (riid == __uuidof(IUnknown) ||
            riid == __uuidof(IRawElementProviderSimple))
            { *pp = (IRawElementProviderSimple*)this; AddRef(); return S_OK; }
        if (riid == __uuidof(IInvokeProvider))
            { *pp = (IInvokeProvider*)this; AddRef(); return S_OK; }
        *pp = nullptr; return E_NOINTERFACE;
    }
    ULONG STDMETHODCALLTYPE AddRef() override { return InterlockedIncrement(&_ref); }
    ULONG STDMETHODCALLTYPE Release() override { auto r = InterlockedDecrement(&_ref); if (!r) delete this; return r; }

    // IRawElementProviderSimple
    IFACEMETHODIMP get_ProviderOptions(ProviderOptions* p) override {
        *p = ProviderOptions_ServerSideProvider; return S_OK;
    }
    IFACEMETHODIMP GetPatternProvider(PATTERNID pid, IUnknown** pp) override {
        if (pid == UIA_InvokePatternId) { *pp = static_cast<IInvokeProvider*>(this); AddRef(); return S_OK; }
        *pp = nullptr; return S_OK;
    }
    IFACEMETHODIMP GetPropertyValue(PROPERTYID pid, VARIANT* p) override {
        VariantInit(p);
        if (pid == UIA_ControlTypePropertyId) { p->vt = VT_I4; p->lVal = UIA_ButtonControlTypeId; return S_OK; }
        if (pid == UIA_NamePropertyId)        { p->vt = VT_BSTR; p->bstrVal = SysAllocString(L"실행"); return S_OK; }
        if (pid == UIA_IsEnabledPropertyId)   { p->vt = VT_BOOL; p->boolVal = VARIANT_TRUE; return S_OK; }
        return S_OK;
    }
    IFACEMETHODIMP get_HostRawElementProvider(IRawElementProviderSimple** pp) override {
        return UiaHostProviderFromHwnd(_hwnd, pp);
    }

    // IInvokeProvider
    IFACEMETHODIMP Invoke() override {
        // 버튼과 동일한 액션
        PostMessageW(_hwnd, WM_COMMAND, MAKEWPARAM(IDOK, BN_CLICKED), 0);
        return S_OK;
    }
};

// 사용: CreateWindowEx로 만든 커스텀 class에 대해 WM_GETOBJECT 처리
LRESULT OnGetObject(HWND h, WPARAM w, LPARAM l) {
    if (static_cast<long>(l) == UiaRootObjectId) {
        ComPtr<IRawElementProviderSimple> prov = new MyControlProvider(h);
        return UiaReturnRawElementProvider(h, w, l, prov.Get());
    }
    return 0;
}

LRESULT CALLBACK WndProc(HWND h, UINT m, WPARAM w, LPARAM l) {
    if (m == WM_GETOBJECT) return OnGetObject(h, w, l);
    // ...
    return DefWindowProc(h,m,w,l);
}
```

- **핵심**: `WM_GETOBJECT`에서 `UiaReturnRawElementProvider` 로 **Provider 제공**.  
- **패턴**: 버튼형이면 `IInvokeProvider`만으로 충분. 입력창이면 `IValueProvider` 추가.

### 7.2 ValuePattern (Edit 유사 컨트롤)
```cpp
class MyEditProvider :
  public IRawElementProviderSimple,
  public IValueProvider
{
    // … IUnknown/IRawElementProviderSimple 유사 …
    IFACEMETHODIMP GetPatternProvider(PATTERNID pid, IUnknown** pp) override {
        if (pid == UIA_ValuePatternId) { *pp = static_cast<IValueProvider*>(this); AddRef(); return S_OK; }
        *pp = nullptr; return S_OK;
    }
    // IValueProvider
    IFACEMETHODIMP get_IsReadOnly(BOOL* v) override { *v = FALSE; return S_OK; }
    IFACEMETHODIMP SetValue(LPCWSTR s) override {
        // 내부 모델 갱신 + 화면 반영
        SetWindowTextW(_hwnd, s);
        // PropertyChanged 이벤트를 올리면 더 좋음
        UiaRaiseAutomationPropertyChangedEvent(this, UIA_ValueValuePropertyId, _old, _new);
        return S_OK;
    }
    IFACEMETHODIMP get_Value(BSTR* s) override {
        wchar_t buf[256]; GetWindowTextW(_hwnd, buf, 256);
        *s = SysAllocString(buf);
        return S_OK;
    }
};
```

- **이벤트**: 값 변화 시 `UiaRaiseAutomationPropertyChangedEvent`.  
- **텍스트 조작이 복잡**하면 `ITextProvider`(텍스트 범위/캐럿/선택) 지원을 고려.

### 7.3 Scroll/ScrollItem 패턴 (가상 리스트/캔버스)
- **컨테이너**: `IScrollProvider` — 수평/수직 스크롤 양/가시 영역/스크롤 명령.  
- **항목**: `IScrollItemProvider` — 자신이 보이도록 `ScrollIntoView`.

---

## 8. 텍스트(에디터/뷰어) 고급: TextPattern

- `ITextProvider`/`ITextRangeProvider` 를 구현하면 **스크린리더의 읽기/선택/이동**이 정확해집니다.  
- **HitTest**: 좌표→문자 범위, **Range 이동**: 단어/문장/행 경계, **속성**: 굵게/밑줄/링크.  
- 이미 **DirectWrite**로 조합/커서/히트테스트를 구현했다면, 그 로직을 **TextRangeProvider**가 호출하도록 연결.

```cpp
// 개략 인터페이스
class MyTextProvider : public ITextProvider {
  IFACEMETHODIMP GetSelection(SAFEARRAY** p) override;
  IFACEMETHODIMP GetVisibleRanges(SAFEARRAY** p) override;
  IFACEMETHODIMP RangeFromChild(IRawElementProviderSimple* child, ITextRangeProvider** p) override;
  IFACEMETHODIMP RangeFromPoint(UiaPoint pt, ITextRangeProvider** p) override;
  IFACEMETHODIMP get_DocumentRange(ITextRangeProvider** p) override;
  IFACEMETHODIMP get_SupportedTextSelection(SupportedTextSelection* v) override;
};
```

> **실무 팁**  
> - 문서가 큰 경우: 레인지 내부만 **지연 로딩**/가상화.  
> - 링크/코드 하이라이트: **TextAttribute**(UIA)에 색/스타일/역할을 속성으로 노출.

---

## 9. 구조 변경/속성 변경/포커스 이벤트

- 포커스 이동 시 **AutomationFocusChanged** 이벤트 자동 발생(일반 컨트롤). 커스텀 컨트롤은 필요 시 `UiaRaiseAutomationEvent`.  
- 항목 추가/삭제/이동: **StructureChanged** 이벤트로 알림.  
- 속성 변화(이름/값/토글 상태): **PropertyChanged** 이벤트.

```cpp
// 상태 텍스트 변경 알림
UiaRaiseAutomationPropertyChangedEvent(provider, UIA_NamePropertyId, oldVar, newVar);
```

---

## 10. 가상화(Virtualization) & 수천/수만 항목

- **VirtualizedItemPattern**: 실제 HWND나 실체를 만들지 않고도 항목을 **논리적으로 노출**.  
- 사용자(스크린리더)가 항목에 접근하려 할 때 `Realize()`(또는 `ScrollIntoView`)에서 **행을 생성/데이터 바인딩**.  
- **ItemContainerPattern**: 특정 속성(이름/자동화ID)로 **항목 찾기** 지원.

> 대량 목록에서 **모든 항목을 UIA 트리에 등록하지 말 것**.  
> 가상화 패턴으로 **필요 시점**에만 실체화.

---

## 11. 스크린리더 호환 체크리스트

- [ ] **Inspect.exe** 로 트리/속성/패턴 확인 (Windows SDK 도구).  
- [ ] **AccEvent.exe** 로 이벤트 발생 확인(포커스/속성/구조 변경).  
- [ ] **내레이터(Win+Ctrl+Enter)** 로 Tab 순서/이름/역할/설명을 읽는지 실제 확인.  
- [ ] **키보드 전용 시나리오**: 마우스 없이 모든 주요 기능 수행 가능?  
- [ ] **High-Contrast 모드**: 모든 텍스트/아이콘/포커스 링이 보이나?  
- [ ] **색 구분 의존 없음**: 색맹/필터에서도 상태 인지 가능한가?  
- [ ] **에러/완료 알림**: Live Region/Notification으로 읽히는가?  
- [ ] **리치 텍스트**: 선택/읽기/링크 이동이 가능한가?(TextPattern)

---

## 12. Ribbon/리본 & 공용 컨트롤 접근성

- MFC Feature Pack 리본은 **UIA를 기본 노출**(Windows 10+).  
- 커스텀 갤러리/버튼은 **항목 이름/도구팁**을 정확히 설정.  
- **Quick Access Toolbar**(QAT) 항목은 AutomationId/Name 일관성 유지.

---

## 13. 도형/캔버스형 커스텀 UI의 접근성 설계

- **포커서블 오브젝트**를 **논리적 아이템**으로 모델링: 각 셰이프→`ListItem`/`DataItem`.  
- 선택/토글/편집 상태를 패턴으로 노출 (`SelectionItem`, `Toggle`, `Value`).  
- **히트 테스트**: 마우스만 의존하지 말고 **키보드 이동**을 설계(다음/이전/상하좌우).  
- **레이어/오버레이**: 포커스 링/선택 박스를 `DrawFocusRect` 또는 굵은 테두리로 명확히.

---

## 14. 폼 검증 UX: 메시지박스만으로 끝내지 말고 “화면 리더에게도 알려라”

- 검증 실패 시:  
  1) **문자열**로 사용자에게 설명(레이블 하단/ToolTip).  
  2) **포커스를 에러 필드로 이동**.  
  3) **UiaRaiseAutomationNotification** 으로 “이름은 필수입니다” 읽어주기.  
  4) `HelpText`/`ItemStatus` 로 상태 유지.

---

## 15. 국제화/다국어 접근성

- **Locale**에 맞는 **날짜/숫자/시간** 읽기(문자열 포맷).  
- **RTL**(아랍어/히브리어) 레이아웃/포커스 순서 재점검.  
- **문자 조합/확장**: 텍스트 컨트롤은 **Unicode**(UTF-16) 기반, 이모지/합자/복합 스크립트 테스트.

---

## 16. 성능과 안정성

- UIA Provider는 **필요한 패턴만 구현**(YAGNI).  
- **대량 항목 = 가상화**.  
- 이벤트 남발 금지: **상태가 실제로 바뀐 경우에만** 올리기.  
- Provider 수명 관리: 참조계수/Release 누수 방지, 크래시 시 덤프 수집.

---

## 17. 미니 레시피 모음

### 17.1 버튼형 커스텀 캔버스 “셀” 접근화
- ControlType: `ListItem`  
- 패턴: `SelectionItem`, `Invoke`  
- Name: “행 3, 열 2, 사과” 등 맥락 포함  
- 이벤트: 선택 변경 → `SelectionItem_ElementSelectedEvent`

### 17.2 “확장/접기” 패널
- ControlType: `Group`  
- 패턴: `ExpandCollapse`  
- `ExpandCollapseState` 유지, Space/Enter로 토글, `PropertyChanged` 이벤트

### 17.3 진행률/배지
- ControlType: `ProgressBar` + `RangeValue`(읽기전용)  
- **중요 단계 완료 시**: `UiaRaiseAutomationNotification`(ActionCompleted)로 안내

---

## 18. 빠른 점검 스니펫: High-Contrast + 시스템 색 브러시

```cpp
struct ThemeBrushes {
    HBRUSH wnd{}, txt{}, hl{}, hltxt{};
    void Rebuild() {
        Destroy();
        wnd   = CreateSolidBrush(GetSysColor(COLOR_WINDOW));
        txt   = CreateSolidBrush(GetSysColor(COLOR_WINDOWTEXT));
        hl    = CreateSolidBrush(GetSysColor(COLOR_HIGHLIGHT));
        hltxt = CreateSolidBrush(GetSysColor(COLOR_HIGHLIGHTTEXT));
    }
    void Destroy() {
        for (HBRUSH b : {wnd, txt, hl, hltxt}) if (b) DeleteObject(b);
        wnd = txt = hl = hltxt = nullptr;
    }
} g_theme;

LRESULT CALLBACK WndProc(HWND h, UINT m, WPARAM w, LPARAM l) {
    switch (m) {
    case WM_CREATE: case WM_THEMECHANGED: case WM_SETTINGCHANGE:
        g_theme.Rebuild(); InvalidateRect(h, nullptr, TRUE); break;
    case WM_DESTROY: g_theme.Destroy(); PostQuitMessage(0); break;
    }
    return DefWindowProc(h,m,w,l);
}
```

---

## 19. 최종 체크리스트(요약)

- **표준 컨트롤** 우선, **커스텀은 UIA Provider 최소 패턴**으로  
- **Name/AutomationId/ControlType** 제대로  
- **키보드 100%** 가능(탭/화살표/단축키/포커스 표시)  
- **High-Contrast/시스템 색** 사용, 색만으로 정보 전달 금지  
- **알림은 말로도**(UiaRaiseAutomationNotification)  
- **대량 목록은 가상화** + `Scroll/ScrollItem/VirtualizedItem`  
- **텍스트 컨트롤은 TextPattern** 고려(선택/읽기/링크)  
- **Inspect/AccEvent/내레이터**로 실제 체감 테스트
