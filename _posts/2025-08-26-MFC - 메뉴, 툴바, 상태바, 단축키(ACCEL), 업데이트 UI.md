---
layout: post
title: MFC - 메뉴·툴바·상태바, 단축키(ACCEL), 업데이트 UI
date: 2025-08-26 14:25:23 +0900
category: MFC
---

# 메뉴 / 툴바 / 상태바 / 단축키 / 업데이트 UI

> 본 글은 두 문서를 하나로 통합한 버전입니다. 각 개념 섹션 뒤에 바로 **빌드 가능한 예제**를 배치해
> `A.1 → B.1 → A.2 → B.2 …` 흐름으로 읽기-실습을 교차 진행할 수 있게 했습니다.

---

## 0. 큰 그림: “명령 ID로 4가지 입력을 통합” (A)
MFC에서는 **하나의 명령 ID**(예: `ID_FILE_OPEN`)가 아래 **4개의 입력 경로**를 동시에 대표합니다.

1) **메뉴 항목** → `WM_COMMAND`
2) **툴바 버튼** → `WM_COMMAND`
3) **단축키(Accelerator)** → `WM_COMMAND`
4) **상태바 힌트** → **문자열 리소스**의 “상태바 설명” 부분 사용

즉, “메뉴를 누르든/툴바를 클릭하든/단축키를 치든” **같은 ID**가 전달되어 **동일한 핸들러**가 실행됩니다. 이것이 MFC 명령 라우팅의 핵심 철학입니다.

---

## 1. 메뉴(Menu) (A)

### 1-1. 리소스 에디터에서 만드는 법
- **리소스 뷰 → Menu → 새 메뉴** 생성.
- 항목마다 **Command ID**(예: `ID_FILE_OPEN`)를 부여.
- **문자열 리소스**에 동일 ID의 문자열을 등록하면,
  - **앞부분**: 메뉴 캡션(예: `&Open...`)
  - **개행(`\n`) 이후**: **상태바 도움말 텍스트** (예: `Open a document`)
  → 메뉴에 마우스를 올리면 상태바에 자동 표시됩니다(`OnSetMessageString` 메커니즘).

### 1-2. 액세스 키/구분선/서브메뉴
- `&` 문자로 **Alt + 문자** 액세스 키 설정(예: `&File` → Alt+F).
- 구분선은 **Separator** 항목 사용.
- 항목에 **Submenu**를 연결하면 드롭다운 구조 생성.

### 1-3. 명령 라우팅 순서
문서/뷰 기반 앱에서 **명령/업데이트**는 아래 순서로 핸들러를 찾습니다.

`활성 CView → CFrameWnd(프레임) → CDocument → CWinApp`

> 같은 ID의 핸들러가 여러 곳에 있으면 **먼저 찾은 쪽이 우선**합니다.
> 데이터 조작은 `CDocument`, 화면 편집/도구는 `CView`, 전역 기능은 `CWinApp`가 자연스럽습니다.

### 1-4. 동적 메뉴/컨텍스트 메뉴
- **동적 메뉴**: `OnInitMenuPopup` 시점에 항목 삽입/제거 가능(단, 매 프레임 조작은 금물).
- **컨텍스트 메뉴(우클릭)**: `TrackPopupMenu`로 화면 좌표에 표시. 컨텍스트 전용 ID도 **동일 라우팅**을 따릅니다.

### 1-5. 흔한 실수/체크포인트
- **ID 중복**: 의도하지 않은 핸들러 호출. ID 네이밍 규칙을 지키고 범위를 관리하세요.
- **문자열 리소스 미등록**: 상태바 힌트가 비거나 이전 텍스트가 남습니다.
- **대화상자 기반 앱**: 메뉴 업데이트가 자동으로 빈번하지 않으므로, 필요 시 **명시적 UI 업데이트**(아래 §5-4 참조)가 필요합니다.

---

### ▶ 실전 예제: SDI 메뉴 리소스 & 문자열 테이블 (B)
## 예시 A) SDI 앱: 메뉴/툴바/상태바 + 단축키 + 업데이트 UI

### 1. `resource.h` (ID 정의)
```cpp
#pragma once

// 프레임/리소스
#define IDR_MAINFRAME          128
#define IDR_MAINMENU           129

// 커맨드 (표준 ID는 afxres.h에도 정의되어 있으나, 예시를 위해 나열)
#define ID_APP_ABOUT           0xE140
// 표준: ID_FILE_NEW, ID_FILE_OPEN, ID_FILE_SAVE, ID_APP_EXIT, ID_EDIT_COPY, ID_EDIT_PASTE, ID_VIEW_TOOLBAR, ID_VIEW_STATUS_BAR 등

// 상태바 커스텀 판넬(Optional)
#define ID_INDICATOR_READY     59100
```

### 2. 메뉴 리소스(`.rc`) — 상태바 힌트(문자열 리소스와 연동)
```rc
IDR_MAINMENU MENU
BEGIN
    POPUP "&File"
    BEGIN
        MENUITEM "&New\tCtrl+N",              ID_FILE_NEW
        MENUITEM "&Open...\tCtrl+O",          ID_FILE_OPEN
        MENUITEM "&Save\tCtrl+S",             ID_FILE_SAVE
        MENUITEM SEPARATOR
        MENUITEM "E&xit\tCtrl+Q",             ID_APP_EXIT
    END
    POPUP "&Edit"
    BEGIN
        MENUITEM "&Copy\tCtrl+C",             ID_EDIT_COPY
        MENUITEM "&Paste\tCtrl+V",            ID_EDIT_PASTE
    END
    POPUP "&View"
    BEGIN
        MENUITEM "&Toolbar",                  ID_VIEW_TOOLBAR
        MENUITEM "&Status Bar",               ID_VIEW_STATUS_BAR
    END
    POPUP "&Help"
    BEGIN
        MENUITEM "&About\tF1",                ID_APP_ABOUT
    END
END
```

---

## 2. 툴바(Toolbar) (A)

### 2-1. 고전 `CToolBar` vs Feature Pack `CMFCToolBar`
- **`CToolBar`(클래식)**: 가볍고 전통적인 툴바. 이미지 스트립(비트맵) 기반.
- **`CMFCToolBar`(Feature Pack)**: 테마/커스터마이즈/드롭다운/명령 관리가 강화된 현대식 툴바.
  → **둘 중 하나**를 선택해 사용하는 것이 유지보수에 유리합니다(혼용 지양).

### 2-2. 생성·이미지·아이디 연결
- **리소스 에디터**에서 툴바 리소스 추가(버튼 순서/ID 지정).
- 이미지 크기(전통적으로 16×16 또는 24×24)와 **DPI 스케일링**을 고려해 준비.
- 버튼 ID는 **메뉴와 동일한 명령 ID**를 사용하면 **핸들러 공유**가 자동입니다.

### 2-3. 확장 스타일/툴팁/상태
- 확장 스타일(클래식): **플랫 모양**, **더블버퍼링** 등(Feature Pack은 각종 옵션 제공).
- 툴팁은 **문자열 리소스**의 캡션/둘째 줄을 활용하거나 별도의 툴팁 텍스트를 할당.
- 버튼 활성/비활성/체크 상태는 `ON_UPDATE_COMMAND_UI`에서 **CCmdUI**를 통해 제어(아래 §5).

### 2-4. DPI/테마/비활성 아이콘
- **고해상도**에서 흐릿함을 피하려면 **여러 스케일**(예: 100/150/200%) 이미지 준비 또는 벡터 아이콘 사용 가능한 툴킷 사용.
- **비활성(Disabled) 상태** 아이콘을 별도로 준비하면 가독성↑(자동 디밍은 대비가 낮을 수 있음).
- Feature Pack 사용 시 **Visual Manager**로 다크/현대 테마를 적용할 수 있습니다.

### 2-5. 드롭다운/스플릿 버튼(Feature Pack)
- `CMFCToolBarButton`의 **드롭다운**을 활용하면 메뉴와 툴바의 복합 UX를 제공.
- 기본 버튼 클릭과 드롭다운 화살표 클릭을 **서로 다른 동작**으로 매핑 가능.

---

### ▶ 실전 예제: 툴바 리소스 & MainFrame 생성/토글/업데이트 (B)
### 3. 툴바 리소스(`.rc`)
```rc
IDR_MAINFRAME TOOLBAR 16,15
BEGIN
    BUTTON      ID_FILE_NEW
    BUTTON      ID_FILE_OPEN
    BUTTON      ID_FILE_SAVE
    SEPARATOR
    BUTTON      ID_EDIT_COPY
    BUTTON      ID_EDIT_PASTE
    SEPARATOR
    BUTTON      ID_APP_ABOUT
END
```

### 6. 메인 프레임(`MainFrm.h/.cpp`) — 툴바/상태바 생성, 토글 + 업데이트 UI
```cpp
// MainFrm.h
#pragma once
#include <afxwin.h>
#include <afxext.h>

class CMainFrame : public CFrameWnd
{
protected:
    DECLARE_DYNCREATE(CMainFrame)
public:
    CMainFrame() noexcept = default;

    // 구현
    virtual BOOL PreCreateWindow(CREATESTRUCT& cs);

protected:
    CStatusBar  m_wndStatusBar;
    CToolBar    m_wndToolBar;

    afx_msg int OnCreate(LPCREATESTRUCT lpCreateStruct);
    afx_msg void OnViewToolbar();
    afx_msg void OnViewStatusBar();
    afx_msg void OnUpdateViewToolbar(CCmdUI* pCmdUI);
    afx_msg void OnUpdateViewStatusBar(CCmdUI* pCmdUI);

    DECLARE_MESSAGE_MAP()
};
```

```cpp
// MainFrm.cpp
#include "pch.h"
#include "resource.h"
#include "MainFrm.h"

#ifdef _DEBUG
#define new DEBUG_NEW
#endif

IMPLEMENT_DYNCREATE(CMainFrame, CFrameWnd)

BEGIN_MESSAGE_MAP(CMainFrame, CFrameWnd)
    ON_WM_CREATE()
    ON_COMMAND(ID_VIEW_TOOLBAR, &CMainFrame::OnViewToolbar)
    ON_COMMAND(ID_VIEW_STATUS_BAR, &CMainFrame::OnViewStatusBar)
    ON_UPDATE_COMMAND_UI(ID_VIEW_TOOLBAR, &CMainFrame::OnUpdateViewToolbar)
    ON_UPDATE_COMMAND_UI(ID_VIEW_STATUS_BAR, &CMainFrame::OnUpdateViewStatusBar)
END_MESSAGE_MAP()

static UINT indicators[] =
{
    ID_SEPARATOR,           // 기본 메시지 영역(가변 폭)
    ID_INDICATOR_CAPS,
    ID_INDICATOR_NUM,
    ID_INDICATOR_SCRL
    // 필요 시 커스텀: ID_INDICATOR_READY
};

BOOL CMainFrame::PreCreateWindow(CREATESTRUCT& cs)
{
    if (!CFrameWnd::PreCreateWindow(cs)) return FALSE;
    cs.style |= FWS_ADDTOTITLE; // 문서명 타이틀에 추가
    return TRUE;
}

int CMainFrame::OnCreate(LPCREATESTRUCT lpCreateStruct)
{
    if (CFrameWnd::OnCreate(lpCreateStruct) == -1) return -1;

    // 툴바
    if (!m_wndToolBar.CreateEx(this, TBSTYLE_FLAT, WS_CHILD|WS_VISIBLE|CBRS_TOP|CBRS_TOOLTIPS|CBRS_FLYBY|CBRS_SIZE_DYNAMIC) ||
        !m_wndToolBar.LoadToolBar(IDR_MAINFRAME))
        return -1;

    // 상태바
    if (!m_wndStatusBar.Create(this) ||
        !m_wndStatusBar.SetIndicators(indicators, (sizeof(indicators)/sizeof(UINT))))
        return -1;

    // 초기 상태 텍스트
    m_wndStatusBar.SetPaneText(0, _T("Ready"));

    // 도킹 허용(옵션)
    m_wndToolBar.EnableDocking(CBRS_ALIGN_ANY);
    EnableDocking(CBRS_ALIGN_ANY);
    DockControlBar(&m_wndToolBar);

    return 0;
}

void CMainFrame::OnViewToolbar()
{
    const BOOL bShow = !m_wndToolBar.IsWindowVisible();
    m_wndToolBar.ShowWindow(bShow ? SW_SHOW : SW_HIDE);
    RecalcLayout();
}

void CMainFrame::OnViewStatusBar()
{
    const BOOL bShow = !m_wndStatusBar.IsWindowVisible();
    m_wndStatusBar.ShowWindow(bShow ? SW_SHOW : SW_HIDE);
    RecalcLayout();
}

void CMainFrame::OnUpdateViewToolbar(CCmdUI* pCmdUI)
{
    pCmdUI->SetCheck(m_wndToolBar.IsWindowVisible());
}

void CMainFrame::OnUpdateViewStatusBar(CCmdUI* pCmdUI)
{
    pCmdUI->SetCheck(m_wndStatusBar.IsWindowVisible());
}
```

---

## 3. 상태바(Status Bar) (A)

### 3-1. 구조와 “인디케이터” 개념
- 상태바는 보통 **여러 개의 “Pane(인디케이터)”**로 분할됩니다.
  - `ID_SEPARATOR` : 가변 폭(주 메시지 영역)
  - `ID_INDICATOR_CAPS/NUM/SCRL` : 키보드 상태 표시
  - **사용자 정의 Pane**: 네트워크/진행률/모드 등

### 3-2. 텍스트 갱신
- **메뉴 항목 위에 마우스를 올리면** 해당 ID의 문자열 리소스 **둘째 줄**이 자동 표시(`OnSetMessageString`).
- 그 외 상황에서는 `SetPaneText`로 특정 Pane의 텍스트를 갱신합니다.
- 장시간 표시가 필요 없는 **일시 메시지**는 타이머로 **초 뒤 초기화**하는 패턴이 깔끔합니다.

### 3-3. 사용자 경험(UX) 팁
- **오버플로** 방지: 긴 텍스트는 줄임표 처리하고 핵심 정보만 남기기.
- **접근성**: 색/아이콘만으로 상태 전달을 피하고 **텍스트** 또는 **툴팁**을 병행.
- **진행 상태**는 상태바보다는 **모달·모델리스 진행 대화상자**나 **리본/툴바 진행 UI**가 더 눈에 띕니다.

---

### ▶ 실전 예제: 상태바 Pane + 동적 텍스트 갱신 (B)
static UINT indicators[] =
{
    ID_SEPARATOR,           // 기본 메시지 영역(가변 폭)
    ID_INDICATOR_CAPS,
    ID_INDICATOR_NUM,
    ID_INDICATOR_SCRL
    // 필요 시 커스텀: ID_INDICATOR_READY
};

int CMainFrame::OnCreate(LPCREATESTRUCT lpCreateStruct)
{
    if (CFrameWnd::OnCreate(lpCreateStruct) == -1) return -1;

    // 툴바
    if (!m_wndToolBar.CreateEx(this, TBSTYLE_FLAT, WS_CHILD|WS_VISIBLE|CBRS_TOP|CBRS_TOOLTIPS|CBRS_FLYBY|CBRS_SIZE_DYNAMIC) ||
        !m_wndToolBar.LoadToolBar(IDR_MAINFRAME))
        return -1;

    // 상태바
    if (!m_wndStatusBar.Create(this) ||
        !m_wndStatusBar.SetIndicators(indicators, (sizeof(indicators)/sizeof(UINT))))
        return -1;

    // 초기 상태 텍스트
    m_wndStatusBar.SetPaneText(0, _T("Ready"));

    // 도킹 허용(옵션)
    m_wndToolBar.EnableDocking(CBRS_ALIGN_ANY);
    EnableDocking(CBRS_ALIGN_ANY);
    DockControlBar(&m_wndToolBar);

    return 0;
}

---

## 4. 단축키(Accelerator, ACCEL) (A)

### 4-1. 작동 원리(메시지 루프)
- 메시지 루프에서 `TranslateAccelerator`가 **키 입력을 명령(=ID)**로 변환합니다.
- 변환된 명령은 **메뉴/툴바 클릭과 동일**하게 라우팅됩니다.

### 4-2. 리소스 에디터에서 추가하기
- **Accelerator 리소스**를 추가하고 각 항목에
  - 키 조합(예: `Ctrl+O`)
  - **명령 ID**(예: `ID_FILE_OPEN`)
  - 플래그(Alt/Ctrl/Shift, VirtKey 등)를 지정.
- **이미 존재하는 메뉴/툴바의 ID**를 그대로 재사용하면 핸들러를 공유합니다.

### 4-3. 우선순위/충돌
- **액셀러레이터가 먼저** 명령으로 변환되며, **텍스트 입력 컨트롤 포커스**일 때는 일부 키가 컨트롤에 소비될 수 있습니다(예: `Ctrl+C` 복사).
- **중복 키**를 지정하면 예기치 않은 명령이 실행될 수 있으니, **앱 전반에서 일관성**을 확인하세요.

### 4-4. 지역화/키보드 레이아웃
- 비영문 키보드(한글/IME) 환경에서 **문자 키 가속기**는 혼선을 줄 수 있습니다.
- **기능 키/조합 키 중심**의 가속기 구성이 안전합니다.

---

### ▶ 실전 예제: 가속기 리소스 (SDI) & 다이얼로그에서 TranslateAccelerator (B)
### 4. 가속기(Accelerator) 리소스(`.rc`)
```rc
IDR_MAINFRAME ACCELERATORS
BEGIN
    "N",    ID_FILE_NEW,        VIRTKEY, CONTROL
    "O",    ID_FILE_OPEN,       VIRTKEY, CONTROL
    "S",    ID_FILE_SAVE,       VIRTKEY, CONTROL
    "Q",    ID_APP_EXIT,        VIRTKEY, CONTROL
    VK_F1,  ID_APP_ABOUT,       VIRTKEY
    "C",    ID_EDIT_COPY,       VIRTKEY, CONTROL
    "V",    ID_EDIT_PASTE,      VIRTKEY, CONTROL
END
```

### 2. 대화상자 구현(`MainDlg.cpp`)
```cpp
#include "pch.h"
#include "resource.h"
#include "MainDlg.h"

BEGIN_MESSAGE_MAP(CMainDlg, CDialogEx)
    ON_COMMAND(ID_FILE_SAVE, &CMainDlg::OnFileSave)
    ON_COMMAND(ID_VIEW_TOOLBAR, &CMainDlg::OnToggleFlag) // 예시: 플래그 토글
    ON_UPDATE_COMMAND_UI(ID_FILE_SAVE, &CMainDlg::OnUpdateFileSave)
END_MESSAGE_MAP()

BOOL CMainDlg::OnInitDialog()
{
    CDialogEx::OnInitDialog();

    // 메뉴 연결(리소스의 메뉴를 다이얼로그에 붙이려면 CreateMenu/SetMenu or 리소스에서 지정)
    CMenu menu; menu.LoadMenu(IDR_MAINMENU);
    SetMenu(&menu); menu.Detach(); // Detach: 다이얼로그가 소유

    // 가속기 로드(다이얼로그는 프레임이 없으므로 직접 TranslateAccelerator)
    m_hAccel = ::LoadAccelerators(AfxGetResourceHandle(), MAKEINTRESOURCE(IDR_MAINFRAME));

    // 초기 UI 갱신
    UpdateDialogControls(this, FALSE);
    return TRUE;
}

BOOL CMainDlg::PreTranslateMessage(MSG* pMsg)
{
    if (m_hAccel && ::TranslateAccelerator(m_hWnd, m_hAccel, pMsg))
        return TRUE;

    // 대화상자에서 ON_UPDATE_COMMAND_UI를 동작시키기 위한 트리거(과도한 호출은 피하기)
    if (pMsg->message == WM_KEYUP || pMsg->message == WM_LBUTTONUP)
        UpdateDialogControls(this, FALSE);

    return CDialogEx::PreTranslateMessage(pMsg);
}

void CMainDlg::OnFileSave()
{
    AfxMessageBox(_T("Saved!"));
    m_bCanSave = FALSE;
    UpdateDialogControls(this, FALSE);
}

void CMainDlg::OnToggleFlag()
{
    m_bCanSave = !m_bCanSave;
    UpdateDialogControls(this, FALSE);
}

void CMainDlg::OnUpdateFileSave(CCmdUI* pCmdUI)
{
    pCmdUI->Enable(m_bCanSave == TRUE);
    pCmdUI->SetText(m_bCanSave ? _T("&Save Now") : _T("Save (Disabled)"));
}
```

> 포인트
> - 대화상자에서 **가속기**는 `TranslateAccelerator`를 **직접 호출**해야 동작합니다.
> - `ON_UPDATE_COMMAND_UI` 갱신은 **`UpdateDialogControls`로 트리거**합니다(이벤트/타이머 등 적절한 시점에 1회 호출).

---

## 5. ON_UPDATE_COMMAND_UI — “지금 이 순간의 UI 상태” (A)

### 5-1. 무엇을 하는가
- 메뉴/툴바/리본 항목의 **활성화(Enable)**, **체크/라디오(상태)**, **캡션(Text)** 등을 **실시간**으로 갱신합니다.
- 핸들러의 시그니처는 `void OnUpdateX(CCmdUI* pCmdUI)` 형태이며, 내부에서
  - `pCmdUI->Enable(BOOL)`
  - `pCmdUI->SetCheck(int)` 또는 `SetRadio(BOOL)`
  - `pCmdUI->SetText(LPCTSTR)`
  를 호출합니다.

### 5-2. 언제 호출되는가
- 프레임 기반 앱(SDI/MDI)에서는 **Idle 시점**(메시지가 없을 때) 자동 호출됩니다.
- 메뉴가 펼쳐질 때에도 해당 항목의 `OnUpdate…`가 호출되어 **열기 직전 상태**를 반영합니다.

### 5-3. 어디에 두는가(라우팅 순서 동일)
- 명령 라우팅과 동일한 순서(활성 `CView` → `CFrameWnd` → `CDocument` → `CWinApp`).
- **상태 판단 주체**에 따라 **핸들러 위치**를 결정하세요.
  - 예: 편집 가능 여부는 `CView`, 문서 변경 여부는 `CDocument` 등.

### 5-4. 대화상자 기반 앱의 특수성
- 대화상자 앱은 **Idle 기반 자동 업데이트가 약함**.
- 일반적으로 **이벤트 변화 시점**에 `UpdateDialogControls(this, FALSE)`를 **직접 호출**하여 버튼/메뉴 상태를 재평가합니다.
- 너무 자주 호출하면 성능 저하 → **상태 변화가 발생했을 때만** 호출하는 것이 좋습니다.

### 5-5. 패턴별 모범 사례
- **Enable/Disable**: 기능 사용 가능 상태를 **명확한 조건식**으로 빠르게 평가.
- **Toggle/Check**: 현재 모드/옵션을 **체크**로 표현(저장 아이콘 대신 **상태 표시**가 더 직관적일 때).
- **Radio 그룹**: 동일 그룹의 여러 항목에 대해 **하나만 `SetRadio(TRUE)`**, 나머지는 `FALSE`.
- **Text 변경**: 빈번하면 가독성↓ → **상태바**나 **툴팁**으로 옮기는 것을 고려.

### 5-6. 성능/안정성 주의
- `ON_UPDATE_COMMAND_UI`는 **빈번히 호출**됩니다(특히 프레임 앱).
- **I/O, DB, 복잡한 계산 금지**: 캐시된 상태 변수를 읽는 **O(1) 로직**만 수행하세요.
- 예외 발생 시 UI가 멎은 듯 보일 수 있으므로, 내부에서 **예외/에러 처리**는 철저하게.

---

### ▶ 실전 예제: SDI(문서 수정 여부로 Save 활성) + Dialog(UpdateDialogControls 트리거) (B)
### 7. 문서 클래스(`MyDoc.h/.cpp`) — **저장 가능 상태**에 따라 `Save` 버튼 활성화
```cpp
// MyDoc.h
#pragma once
#include <afxwin.h>
class CMyDoc : public CDocument {
    DECLARE_DYNCREATE(CMyDoc)
public:
    BOOL OnNewDocument() override;
    void Serialize(CArchive& ar) override;

    // 업데이트 UI: Save 활성 조건(수정됨)
    afx_msg void OnUpdateFileSave(CCmdUI* pCmdUI);

    DECLARE_MESSAGE_MAP()
};
```

```cpp
// MyDoc.cpp
#include "pch.h"
#include "MyDoc.h"

IMPLEMENT_DYNCREATE(CMyDoc, CDocument)

BEGIN_MESSAGE_MAP(CMyDoc, CDocument)
    ON_UPDATE_COMMAND_UI(ID_FILE_SAVE, &CMyDoc::OnUpdateFileSave)
END_MESSAGE_MAP()

BOOL CMyDoc::OnNewDocument()
{
    if (!CDocument::OnNewDocument()) return FALSE;
    SetModifiedFlag(FALSE);
    return TRUE;
}

void CMyDoc::Serialize(CArchive& ar)
{
    if (ar.IsStoring()) {
        // TODO: 저장
    } else {
        // TODO: 불러오기
    }
}

void CMyDoc::OnUpdateFileSave(CCmdUI* pCmdUI)
{
    pCmdUI->Enable(IsModified()); // 수정되었을 때만 Save 가능
}
```

## 예시 B) 대화상자 기반: 메뉴 + 단축키 + 대화상자에서의 `ON_UPDATE_COMMAND_UI`

대화상자 앱은 프레임 기반 자동 업데이트가 약하므로, **`UpdateDialogControls(this, FALSE)`를 직접 호출**해 UI를 갱신합니다.

### 1. 대화상자 헤더(`MainDlg.h`)
```cpp
#pragma once
#include <afxwin.h>

class CMainDlg : public CDialogEx {
public:
    CMainDlg() : CDialogEx(IDD_MAIN_DIALOG) {}

protected:
    HACCEL m_hAccel = nullptr;
    BOOL OnInitDialog() override;
    BOOL PreTranslateMessage(MSG* pMsg) override;

    // 상태 플래그(예: 저장 가능 여부)
    BOOL m_bCanSave = FALSE;

    // 커맨드/업데이트 핸들러
    afx_msg void OnFileSave();
    afx_msg void OnUpdateFileSave(CCmdUI* pCmdUI);
    afx_msg void OnToggleFlag();

    DECLARE_MESSAGE_MAP()
};
```

### 2. 대화상자 구현(`MainDlg.cpp`)
```cpp
#include "pch.h"
#include "resource.h"
#include "MainDlg.h"

BEGIN_MESSAGE_MAP(CMainDlg, CDialogEx)
    ON_COMMAND(ID_FILE_SAVE, &CMainDlg::OnFileSave)
    ON_COMMAND(ID_VIEW_TOOLBAR, &CMainDlg::OnToggleFlag) // 예시: 플래그 토글
    ON_UPDATE_COMMAND_UI(ID_FILE_SAVE, &CMainDlg::OnUpdateFileSave)
END_MESSAGE_MAP()

BOOL CMainDlg::OnInitDialog()
{
    CDialogEx::OnInitDialog();

    // 메뉴 연결(리소스의 메뉴를 다이얼로그에 붙이려면 CreateMenu/SetMenu or 리소스에서 지정)
    CMenu menu; menu.LoadMenu(IDR_MAINMENU);
    SetMenu(&menu); menu.Detach(); // Detach: 다이얼로그가 소유

    // 가속기 로드(다이얼로그는 프레임이 없으므로 직접 TranslateAccelerator)
    m_hAccel = ::LoadAccelerators(AfxGetResourceHandle(), MAKEINTRESOURCE(IDR_MAINFRAME));

    // 초기 UI 갱신
    UpdateDialogControls(this, FALSE);
    return TRUE;
}

BOOL CMainDlg::PreTranslateMessage(MSG* pMsg)
{
    if (m_hAccel && ::TranslateAccelerator(m_hWnd, m_hAccel, pMsg))
        return TRUE;

    // 대화상자에서 ON_UPDATE_COMMAND_UI를 동작시키기 위한 트리거(과도한 호출은 피하기)
    if (pMsg->message == WM_KEYUP || pMsg->message == WM_LBUTTONUP)
        UpdateDialogControls(this, FALSE);

    return CDialogEx::PreTranslateMessage(pMsg);
}

void CMainDlg::OnFileSave()
{
    AfxMessageBox(_T("Saved!"));
    m_bCanSave = FALSE;
    UpdateDialogControls(this, FALSE);
}

void CMainDlg::OnToggleFlag()
{
    m_bCanSave = !m_bCanSave;
    UpdateDialogControls(this, FALSE);
}

void CMainDlg::OnUpdateFileSave(CCmdUI* pCmdUI)
{
    pCmdUI->Enable(m_bCanSave == TRUE);
    pCmdUI->SetText(m_bCanSave ? _T("&Save Now") : _T("Save (Disabled)"));
}
```

> 포인트
> - 대화상자에서 **가속기**는 `TranslateAccelerator`를 **직접 호출**해야 동작합니다.
> - `ON_UPDATE_COMMAND_UI` 갱신은 **`UpdateDialogControls`로 트리거**합니다(이벤트/타이머 등 적절한 시점에 1회 호출).

---

## 6. 통합 작동 시나리오 (A)

1) 사용자가 **문서를 열어 편집** → 앱 내부 상태 `isDirty = true`.
2) Idle 타임에 `ON_UPDATE_COMMAND_UI`가 호출됨 →
   - `ID_FILE_SAVE`에 대해 `pCmdUI->Enable(isDirty)` → **저장 버튼 활성화**
   - `ID_VIEW_STATUSBAR`에 대해 `pCmdUI->SetCheck(isStatusBarVisible)` → **메뉴 체크 표시**
3) 사용자가 **Ctrl+S**(가속기) 또는 **툴바 저장 버튼** 또는 **메뉴 저장** 클릭 →
   - 모두 `ID_FILE_SAVE`로 라우팅 → **동일 핸들러** 실행
4) 성공적으로 저장하면 `isDirty = false` → 다음 Idle에 **저장 버튼 비활성화**
5) 메뉴에 마우스 오버 시, 문자열 리소스의 둘째 줄이 **상태바에 힌트**로 표시

---

---

## 7. 디버깅/트러블슈팅 (A)

- **업데이트가 안 된다**
  - 프레임 앱: `ON_UPDATE_COMMAND_UI`가 정의된 **클래스와 라우팅 경로**가 맞는지 확인.
  - 대화상자 앱: **이벤트 후 `UpdateDialogControls` 호출** 누락 여부 확인.
- **툴바 버튼만 비활성**
  - 동일 ID의 `ON_UPDATE_COMMAND_UI`가 **툴바/메뉴 모두**에 적용되지만, 일부 프레임/Feature Pack 구성에서는 **메뉴 펼침 시점**에만 평가될 수 있습니다. **Idle 호출**이 동작하는지 점검.
- **단축키가 먹지 않는다**
  - **Accelerator 테이블**이 프레임에 올바르게 연결되어 있는지, 포커스가 **에디트 컨트롤**에 있지 않은지 확인.
  - 이미 **다른 가속기**가 선점했는지(중복) 체크.
- **상태바 힌트가 안 보임**
  - 문자열 리소스의 **개행(`\n`)** 이후 설명 문구가 비어있지 않은지 확인.
  - 커스텀 상태바 사용 시 `OnSetMessageString` 메시지 흐름을 가로막지 않았는지 점검.

---

---

## 8. 품질 체크리스트 (A)

- 명령 ID/문자열 리소스/가속기 테이블 간 **ID 일관성 유지**
- `ON_UPDATE_COMMAND_UI`는 **빠르고 결정적**인 로직만 수행
- 대화상자 앱은 **명시적 UI 업데이트 트리거**를 도입
- 툴바/아이콘은 **DPI 대비**(여러 스케일 or 벡터)
- 상태바 텍스트는 **핵심 정보 우선**, 과도한 길이 금지
- 단축키는 **표준 관례**(Ctrl+N/O/S, Ctrl+Z/Y/C/V 등)를 가능한 준수
- 지역화 시 **액세스 키(`&`) 충돌** 점검(메뉴 전반 일관성)

---

---

## 9. 요약 (A)

- **메뉴·툴바·단축키**는 **하나의 명령 ID**로 수렴하며,
- **상태바**는 그 명령의 **설명 텍스트**를 사용자에게 피드백합니다.
- **`ON_UPDATE_COMMAND_UI`**는 “지금 이 순간, 이 명령을 보여주고/활성화할지/체크할지”를 결정하는 핵심 후크입니다.
- 프레임 앱은 **Idle 자동 업데이트**, 대화상자 앱은 **명시적 트리거**가 실전 포인트입니다.

---

## 부록: 전체 예제 모음(B) — 한 곳에서 보기
# 메뉴 / 툴바 / 상태바 예시 모음 (SDI & Dialog 기반)

아래는 **실전에서 바로 붙여 넣어 빌드**할 수 있도록 준비한 예시들입니다.
- 예시 A: **SDI(문서/뷰) 앱** — 메뉴/툴바/상태바 + 단축키 + `ON_UPDATE_COMMAND_UI`
- 예시 B: **대화상자 기반 앱** — 메뉴 + 단축키 + `UpdateDialogControls` 기반 UI 업데이트

---

## 예시 A) SDI 앱: 메뉴/툴바/상태바 + 단축키 + 업데이트 UI

### 1. `resource.h` (ID 정의)
```cpp
#pragma once

// 프레임/리소스
#define IDR_MAINFRAME          128
#define IDR_MAINMENU           129

// 커맨드 (표준 ID는 afxres.h에도 정의되어 있으나, 예시를 위해 나열)
#define ID_APP_ABOUT           0xE140
// 표준: ID_FILE_NEW, ID_FILE_OPEN, ID_FILE_SAVE, ID_APP_EXIT, ID_EDIT_COPY, ID_EDIT_PASTE, ID_VIEW_TOOLBAR, ID_VIEW_STATUS_BAR 등

// 상태바 커스텀 판넬(Optional)
#define ID_INDICATOR_READY     59100
```

### 2. 메뉴 리소스(`.rc`) — 상태바 힌트(문자열 리소스와 연동)
```rc
IDR_MAINMENU MENU
BEGIN
    POPUP "&File"
    BEGIN
        MENUITEM "&New\tCtrl+N",              ID_FILE_NEW
        MENUITEM "&Open...\tCtrl+O",          ID_FILE_OPEN
        MENUITEM "&Save\tCtrl+S",             ID_FILE_SAVE
        MENUITEM SEPARATOR
        MENUITEM "E&xit\tCtrl+Q",             ID_APP_EXIT
    END
    POPUP "&Edit"
    BEGIN
        MENUITEM "&Copy\tCtrl+C",             ID_EDIT_COPY
        MENUITEM "&Paste\tCtrl+V",            ID_EDIT_PASTE
    END
    POPUP "&View"
    BEGIN
        MENUITEM "&Toolbar",                  ID_VIEW_TOOLBAR
        MENUITEM "&Status Bar",               ID_VIEW_STATUS_BAR
    END
    POPUP "&Help"
    BEGIN
        MENUITEM "&About\tF1",                ID_APP_ABOUT
    END
END
```

### 3. 툴바 리소스(`.rc`)
```rc
IDR_MAINFRAME TOOLBAR 16,15
BEGIN
    BUTTON      ID_FILE_NEW
    BUTTON      ID_FILE_OPEN
    BUTTON      ID_FILE_SAVE
    SEPARATOR
    BUTTON      ID_EDIT_COPY
    BUTTON      ID_EDIT_PASTE
    SEPARATOR
    BUTTON      ID_APP_ABOUT
END
```

### 4. 가속기(Accelerator) 리소스(`.rc`)
```rc
IDR_MAINFRAME ACCELERATORS
BEGIN
    "N",    ID_FILE_NEW,        VIRTKEY, CONTROL
    "O",    ID_FILE_OPEN,       VIRTKEY, CONTROL
    "S",    ID_FILE_SAVE,       VIRTKEY, CONTROL
    "Q",    ID_APP_EXIT,        VIRTKEY, CONTROL
    VK_F1,  ID_APP_ABOUT,       VIRTKEY
    "C",    ID_EDIT_COPY,       VIRTKEY, CONTROL
    "V",    ID_EDIT_PASTE,      VIRTKEY, CONTROL
END
```

### 5. 문자열 테이블 — **메뉴 캡션 + 상태바 힌트**(개행 `\n` 뒤 힌트가 상태바에 자동 표시)
```rc
STRINGTABLE
BEGIN
    ID_FILE_NEW            "New...\nCreate a new document."
    ID_FILE_OPEN           "Open...\nOpen an existing document."
    ID_FILE_SAVE           "Save\nSave the current document."
    ID_APP_EXIT            "Exit\nClose the application."
    ID_EDIT_COPY           "Copy\nCopy selection to clipboard."
    ID_EDIT_PASTE          "Paste\nPaste from clipboard."
    ID_VIEW_TOOLBAR        "Toolbar\nShow or hide the toolbar."
    ID_VIEW_STATUS_BAR     "Status Bar\nShow or hide the status bar."
    ID_APP_ABOUT           "About\nAbout this application."
END
```

### 6. 메인 프레임(`MainFrm.h/.cpp`) — 툴바/상태바 생성, 토글 + 업데이트 UI
```cpp
// MainFrm.h
#pragma once
#include <afxwin.h>
#include <afxext.h>

class CMainFrame : public CFrameWnd
{
protected:
    DECLARE_DYNCREATE(CMainFrame)
public:
    CMainFrame() noexcept = default;

    // 구현
    virtual BOOL PreCreateWindow(CREATESTRUCT& cs);

protected:
    CStatusBar  m_wndStatusBar;
    CToolBar    m_wndToolBar;

    afx_msg int OnCreate(LPCREATESTRUCT lpCreateStruct);
    afx_msg void OnViewToolbar();
    afx_msg void OnViewStatusBar();
    afx_msg void OnUpdateViewToolbar(CCmdUI* pCmdUI);
    afx_msg void OnUpdateViewStatusBar(CCmdUI* pCmdUI);

    DECLARE_MESSAGE_MAP()
};
```

```cpp
// MainFrm.cpp
#include "pch.h"
#include "resource.h"
#include "MainFrm.h"

#ifdef _DEBUG
#define new DEBUG_NEW
#endif

IMPLEMENT_DYNCREATE(CMainFrame, CFrameWnd)

BEGIN_MESSAGE_MAP(CMainFrame, CFrameWnd)
    ON_WM_CREATE()
    ON_COMMAND(ID_VIEW_TOOLBAR, &CMainFrame::OnViewToolbar)
    ON_COMMAND(ID_VIEW_STATUS_BAR, &CMainFrame::OnViewStatusBar)
    ON_UPDATE_COMMAND_UI(ID_VIEW_TOOLBAR, &CMainFrame::OnUpdateViewToolbar)
    ON_UPDATE_COMMAND_UI(ID_VIEW_STATUS_BAR, &CMainFrame::OnUpdateViewStatusBar)
END_MESSAGE_MAP()

static UINT indicators[] =
{
    ID_SEPARATOR,           // 기본 메시지 영역(가변 폭)
    ID_INDICATOR_CAPS,
    ID_INDICATOR_NUM,
    ID_INDICATOR_SCRL
    // 필요 시 커스텀: ID_INDICATOR_READY
};

BOOL CMainFrame::PreCreateWindow(CREATESTRUCT& cs)
{
    if (!CFrameWnd::PreCreateWindow(cs)) return FALSE;
    cs.style |= FWS_ADDTOTITLE; // 문서명 타이틀에 추가
    return TRUE;
}

int CMainFrame::OnCreate(LPCREATESTRUCT lpCreateStruct)
{
    if (CFrameWnd::OnCreate(lpCreateStruct) == -1) return -1;

    // 툴바
    if (!m_wndToolBar.CreateEx(this, TBSTYLE_FLAT, WS_CHILD|WS_VISIBLE|CBRS_TOP|CBRS_TOOLTIPS|CBRS_FLYBY|CBRS_SIZE_DYNAMIC) ||
        !m_wndToolBar.LoadToolBar(IDR_MAINFRAME))
        return -1;

    // 상태바
    if (!m_wndStatusBar.Create(this) ||
        !m_wndStatusBar.SetIndicators(indicators, (sizeof(indicators)/sizeof(UINT))))
        return -1;

    // 초기 상태 텍스트
    m_wndStatusBar.SetPaneText(0, _T("Ready"));

    // 도킹 허용(옵션)
    m_wndToolBar.EnableDocking(CBRS_ALIGN_ANY);
    EnableDocking(CBRS_ALIGN_ANY);
    DockControlBar(&m_wndToolBar);

    return 0;
}

void CMainFrame::OnViewToolbar()
{
    const BOOL bShow = !m_wndToolBar.IsWindowVisible();
    m_wndToolBar.ShowWindow(bShow ? SW_SHOW : SW_HIDE);
    RecalcLayout();
}

void CMainFrame::OnViewStatusBar()
{
    const BOOL bShow = !m_wndStatusBar.IsWindowVisible();
    m_wndStatusBar.ShowWindow(bShow ? SW_SHOW : SW_HIDE);
    RecalcLayout();
}

void CMainFrame::OnUpdateViewToolbar(CCmdUI* pCmdUI)
{
    pCmdUI->SetCheck(m_wndToolBar.IsWindowVisible());
}

void CMainFrame::OnUpdateViewStatusBar(CCmdUI* pCmdUI)
{
    pCmdUI->SetCheck(m_wndStatusBar.IsWindowVisible());
}
```

### 7. 문서 클래스(`MyDoc.h/.cpp`) — **저장 가능 상태**에 따라 `Save` 버튼 활성화
```cpp
// MyDoc.h
#pragma once
#include <afxwin.h>
class CMyDoc : public CDocument {
    DECLARE_DYNCREATE(CMyDoc)
public:
    BOOL OnNewDocument() override;
    void Serialize(CArchive& ar) override;

    // 업데이트 UI: Save 활성 조건(수정됨)
    afx_msg void OnUpdateFileSave(CCmdUI* pCmdUI);

    DECLARE_MESSAGE_MAP()
};
```

```cpp
// MyDoc.cpp
#include "pch.h"
#include "MyDoc.h"

IMPLEMENT_DYNCREATE(CMyDoc, CDocument)

BEGIN_MESSAGE_MAP(CMyDoc, CDocument)
    ON_UPDATE_COMMAND_UI(ID_FILE_SAVE, &CMyDoc::OnUpdateFileSave)
END_MESSAGE_MAP()

BOOL CMyDoc::OnNewDocument()
{
    if (!CDocument::OnNewDocument()) return FALSE;
    SetModifiedFlag(FALSE);
    return TRUE;
}

void CMyDoc::Serialize(CArchive& ar)
{
    if (ar.IsStoring()) {
        // TODO: 저장
    } else {
        // TODO: 불러오기
    }
}

void CMyDoc::OnUpdateFileSave(CCmdUI* pCmdUI)
{
    pCmdUI->Enable(IsModified()); // 수정되었을 때만 Save 가능
}
```

### 8. 뷰 클래스(`MyView.cpp`) — **상태바 텍스트 갱신** & 편의 예시
```cpp
// 마우스 클릭 위치를 상태바에 표시하는 간단 예
void CMyView::OnLButtonDown(UINT nFlags, CPoint point)
{
    CFrameWnd* pFrame = GetParentFrame();
    if (pFrame) {
        CStatusBar* pSB = dynamic_cast<CStatusBar*>(pFrame->GetMessageBar());
        if (pSB) {
            CString msg; msg.Format(_T("Clicked: (%d, %d)"), point.x, point.y);
            pSB->SetPaneText(0, msg);
        }
    }
    CView::OnLButtonDown(nFlags, point);
}
```

> 포인트
> - **가속기**는 `IDR_MAINFRAME`에 정의되어 있으면 MFC가 자동 로드합니다.
> - **상태바 힌트**는 **문자열 테이블**의 `\n` 이후 텍스트가 자동으로 표시됩니다(메뉴에 마우스 오버/항목 포커스 시).
> - `ON_UPDATE_COMMAND_UI`는 **Idle 시점**마다 불리므로 **가벼운 조건 체크**만 수행하세요.

---

## 예시 B) 대화상자 기반: 메뉴 + 단축키 + 대화상자에서의 `ON_UPDATE_COMMAND_UI`

대화상자 앱은 프레임 기반 자동 업데이트가 약하므로, **`UpdateDialogControls(this, FALSE)`를 직접 호출**해 UI를 갱신합니다.

### 1. 대화상자 헤더(`MainDlg.h`)
```cpp
#pragma once
#include <afxwin.h>

class CMainDlg : public CDialogEx {
public:
    CMainDlg() : CDialogEx(IDD_MAIN_DIALOG) {}

protected:
    HACCEL m_hAccel = nullptr;
    BOOL OnInitDialog() override;
    BOOL PreTranslateMessage(MSG* pMsg) override;

    // 상태 플래그(예: 저장 가능 여부)
    BOOL m_bCanSave = FALSE;

    // 커맨드/업데이트 핸들러
    afx_msg void OnFileSave();
    afx_msg void OnUpdateFileSave(CCmdUI* pCmdUI);
    afx_msg void OnToggleFlag();

    DECLARE_MESSAGE_MAP()
};
```

### 2. 대화상자 구현(`MainDlg.cpp`)
```cpp
#include "pch.h"
#include "resource.h"
#include "MainDlg.h"

BEGIN_MESSAGE_MAP(CMainDlg, CDialogEx)
    ON_COMMAND(ID_FILE_SAVE, &CMainDlg::OnFileSave)
    ON_COMMAND(ID_VIEW_TOOLBAR, &CMainDlg::OnToggleFlag) // 예시: 플래그 토글
    ON_UPDATE_COMMAND_UI(ID_FILE_SAVE, &CMainDlg::OnUpdateFileSave)
END_MESSAGE_MAP()

BOOL CMainDlg::OnInitDialog()
{
    CDialogEx::OnInitDialog();

    // 메뉴 연결(리소스의 메뉴를 다이얼로그에 붙이려면 CreateMenu/SetMenu or 리소스에서 지정)
    CMenu menu; menu.LoadMenu(IDR_MAINMENU);
    SetMenu(&menu); menu.Detach(); // Detach: 다이얼로그가 소유

    // 가속기 로드(다이얼로그는 프레임이 없으므로 직접 TranslateAccelerator)
    m_hAccel = ::LoadAccelerators(AfxGetResourceHandle(), MAKEINTRESOURCE(IDR_MAINFRAME));

    // 초기 UI 갱신
    UpdateDialogControls(this, FALSE);
    return TRUE;
}

BOOL CMainDlg::PreTranslateMessage(MSG* pMsg)
{
    if (m_hAccel && ::TranslateAccelerator(m_hWnd, m_hAccel, pMsg))
        return TRUE;

    // 대화상자에서 ON_UPDATE_COMMAND_UI를 동작시키기 위한 트리거(과도한 호출은 피하기)
    if (pMsg->message == WM_KEYUP || pMsg->message == WM_LBUTTONUP)
        UpdateDialogControls(this, FALSE);

    return CDialogEx::PreTranslateMessage(pMsg);
}

void CMainDlg::OnFileSave()
{
    AfxMessageBox(_T("Saved!"));
    m_bCanSave = FALSE;
    UpdateDialogControls(this, FALSE);
}

void CMainDlg::OnToggleFlag()
{
    m_bCanSave = !m_bCanSave;
    UpdateDialogControls(this, FALSE);
}

void CMainDlg::OnUpdateFileSave(CCmdUI* pCmdUI)
{
    pCmdUI->Enable(m_bCanSave == TRUE);
    pCmdUI->SetText(m_bCanSave ? _T("&Save Now") : _T("Save (Disabled)"));
}
```

> 포인트
> - 대화상자에서 **가속기**는 `TranslateAccelerator`를 **직접 호출**해야 동작합니다.
> - `ON_UPDATE_COMMAND_UI` 갱신은 **`UpdateDialogControls`로 트리거**합니다(이벤트/타이머 등 적절한 시점에 1회 호출).

---

## 보너스 팁

- **툴바 툴팁**: 문자열 테이블의 첫 줄(캡션) 또는 별도 TTN_NEEDTEXT 처리로 제공됩니다.
- **상태바 사용자 Pane**: `SetIndicators` 대신 `m_wndStatusBar.AddPane(...)`(Feature Pack) 또는 기존 Pane에 `SetPaneText/Width`로 커스터마이징.
- **다크 테마**(Feature Pack): `CMFCVisualManager::SetDefaultManager(...)`로 현대적 룩&필.
- **문서/뷰 라우팅**: **데이터 관련 명령**은 `CDocument`에, **표시/편집 명령**은 `CView`에 둬야 갱신/핸들러가 직관적으로 동작.
