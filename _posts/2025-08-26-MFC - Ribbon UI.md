---
layout: post
title: MFC - Ribbon UI
date: 2025-08-26 22:25:23 +0900
category: MFC
---
# Ribbon UI / 현대 UI 요소: 리본, 갤러리, 퀵액세스, 다크 모드 접근법 (MFC Feature Pack 실전 총정리 · 예제 대량)

이 글은 **MFC Feature Pack**(리본·갤러리·QAT·백스테이지 등)을 **프로젝트에 처음 붙이는 단계부터**  
**핵심 위젯 구축·명령 라우팅·커스터마이징·영구 저장·DPI/다크 모드 대응**까지 **생략 없이** 정리합니다.  
코드는 **복붙**이 가능하도록 독립 스니펫 위주로 제공합니다.

> 적용 대상  
> - Visual Studio의 **MFC Feature Pack** 기반 프로젝트(SDI/MDI/대화상자 + 프레임)  
> - `CMFCRibbonBar`, `CMFCRibbonCategory`, `CMFCRibbonPanel`, `CMFCRibbonButton`, `CMFCRibbonGallery` 등 사용

---

## 0) 로드맵: 무엇을 구현할 건가?

1. **리본 바** 골격 만들기 (앱 버튼/탭/패널/버튼/스플릿/체크/콤보)  
2. **갤러리**(아이콘/색/스타일 미리보기) + **라이브 프리뷰** 패턴  
3. **Quick Access Toolbar(QAT)** 기본/사용자 커스터마이징/영구 저장  
4. **백스테이지/애플리케이션 메뉴**(파일·최근 문서·옵션)  
5. **키팁/액세스키/키보드 커스터마이징**  
6. **아이콘/HiDPI**(32비트 PNG, 스케일)  
7. **다크 모드/테마**(비주얼 매니저, 시스템/앱 전환)  
8. **상태 저장/복원**(레지스트리)  
9. **성능/UX/접근성** 체크리스트

---

## 1) 프로젝트 준비 & 기본 골격

### 1-1. App/Frame 기본 세팅 (CWinAppEx 추천)

```cpp
// App.h
class CMyApp : public CWinAppEx {
public:
    BOOL InitInstance() override;
    int  ExitInstance() override;

    // 상태 저장/복원에 필요
    void PreLoadState() override;
    void LoadCustomState() override;
    void SaveCustomState() override;

public:
    BOOL m_bHiColorIcons = TRUE;      // 32bpp 아이콘/PNG 사용
};

// App.cpp
BOOL CMyApp::InitInstance()
{
    CWinAppEx::InitInstance();

    // 리본/도킹 등 상태 저장 루트
    SetRegistryKey(L"MyCompany");
    SetRegistryBase(L"Settings"); // (CWinAppEx 전용) 하위 키 커스텀

    InitContextMenuManager();
    InitKeyboardManager();
    InitTooltipManager();

    // 툴팁 현대화
    CMFCToolTipInfo params;
    params.m_bVislManagerTheme = TRUE;
    theApp.GetTooltipManager()->SetDefaultTooltipParams(AFX_TOOLTIP_TYPE_ALL, RUNTIME_CLASS(CMFCToolTipCtrl), &params);

    return TRUE;
}

int CMyApp::ExitInstance()
{
    return CWinAppEx::ExitInstance();
}

void CMyApp::PreLoadState() { /* 필요 시 사전 상태 로드 */ }
void CMyApp::LoadCustomState(){ /* Ribbon Customize 등 */ }
void CMyApp::SaveCustomState(){ /* Ribbon Customize 등 */ }
```

### 1-2. Main Frame에 리본 바 생성

```cpp
// MainFrm.h
class CMainFrame : public CFrameWndEx {
public:
    CMainFrame() noexcept;
protected:
    int OnCreate(LPCREATESTRUCT lpCreateStruct);
    DECLARE_DYNAMIC(CMainFrame)
    DECLARE_MESSAGE_MAP()

private:
    CMFCRibbonBar         m_wndRibbonBar;
    CMFCRibbonApplicationButton m_AppButton;  // 앱(파일) 버튼
    CMFCRibbonStatusBar   m_wndStatusBar;
    CMFCToolBarImages     m_PanelImages;      // 패널 헤더 이미지 (선택)
};
```

```cpp
// MainFrm.cpp
BEGIN_MESSAGE_MAP(CMainFrame, CFrameWndEx)
    ON_WM_CREATE()
END_MESSAGE_MAP()

int CMainFrame::OnCreate(LPCREATESTRUCT cs)
{
    if (CFrameWndEx::OnCreate(cs) == -1) return -1;

    // 리본 바 생성
    if (!m_wndRibbonBar.Create(this)) return -1;
    m_wndRibbonBar.EnableDocking(CBRS_ALIGN_ANY);

    // 애플리케이션(파일) 버튼
    m_AppButton.SetImage(IDB_APPBTN_32); // 32x32 PNG (이미지 리소스)
    m_wndRibbonBar.SetApplicationButton(&m_AppButton, CSize(45, 45));

    // 카테고리(탭) 생성
    CMFCRibbonCategory* pCatHome = m_wndRibbonBar.AddCategory(L"홈", IDB_RIBBON_HOME_16, IDB_RIBBON_HOME_32);
    // (IDB_RIBBON_HOME_16/32는 16/32픽셀 이미지 스트립, 없으면 -1로)

    // 패널 추가 & 버튼들
    CMFCRibbonPanel* pPanelClipboard = pCatHome->AddPanel(L"클립보드");
    pPanelClipboard->Add(new CMFCRibbonButton(ID_EDIT_PASTE, L"붙여넣기", 0, 0));
    pPanelClipboard->Add(new CMFCRibbonButton(ID_EDIT_CUT,   L"잘라내기", 1, 1));
    pPanelClipboard->Add(new CMFCRibbonButton(ID_EDIT_COPY,  L"복사",     2, 2));

    CMFCRibbonPanel* pPanelStyles = pCatHome->AddPanel(L"스타일");
    auto* pTglBold = new CMFCRibbonButton(ID_FMT_BOLD, L"굵게", 3, 3);
    pTglBold->SetCheck();  // 토글 가능(체크)로 동작
    pPanelStyles->Add(pTglBold);
    pPanelStyles->Add(new CMFCRibbonButton(ID_FMT_ITALIC, L"기울임", 4, 4));

    // 갤러리(아래에서 상세)
    CMFCRibbonPanel* pPanelTheme = pCatHome->AddPanel(L"테마");
    // 갤러리 추가는 아래 2장에서…

    // 상태바
    if (!m_wndStatusBar.Create(this)) return -1;
    m_wndStatusBar.AddElement(new CMFCRibbonStatusBarPane(ID_INDICATOR_READY, L"준비", TRUE), L"ReadyPane");

    // QAT(퀵액세스) 기본 항목 (시작에 표시)
    m_wndRibbonBar.AddToQuickAccessToolbar(new CMFCRibbonButton(ID_FILE_NEW,   L"새로 만들기"));
    m_wndRibbonBar.AddToQuickAccessToolbar(new CMFCRibbonButton(ID_FILE_OPEN,  L"열기"));
    m_wndRibbonBar.AddToQuickAccessToolbar(new CMFCRibbonButton(ID_FILE_SAVE,  L"저장"));

    return 0;
}
```

> 리소스 스트립(16/32) 없이도 동작하지만, **HiDPI/다크 모드**를 고려해 **32bpp PNG**를 권장합니다.

---

## 2) 갤러리(Gallery) 구축 — 미리보기/라이브 프리뷰 패턴

### 2-1. 아이콘 세트 준비

```cpp
// 아이콘 세트(16/32) 준비
CMFCToolBarImages g_GalleryImages16, g_GalleryImages32;

void LoadGalleryImages()
{
    g_GalleryImages16.SetImageSize(CSize(16,16));
    g_GalleryImages16.Load(IDB_GALLERY_16, TRUE); // 32bpp PNG 스트립
    g_GalleryImages32.SetImageSize(CSize(32,32));
    g_GalleryImages32.Load(IDB_GALLERY_32, TRUE);
}
```

> `TRUE`는 투명(알파) 사용. 리소스는 32bpp PNG 스트립이 이상적.

### 2-2. 갤러리 생성 & 항목 추가

```cpp
// MainFrm.cpp (OnCreate 내부)
CMFCRibbonPanel* pPanelTheme = pCatHome->AddPanel(L"테마");

auto* pGallery = new CMFCRibbonGallery(ID_GAL_THEME, L"테마",  -1,  // 버튼 라벨, Large 이미지(-1: 없음)
                                       g_GalleryImages32.GetImageSize().cx); // 큰 아이콘 너비 힌트
pGallery->SetIcons(&g_GalleryImages16, &g_GalleryImages32);
pGallery->EnableMenuResize(TRUE, TRUE); // 팝업 크기 사용자 조절 허용
pGallery->EnableMenuScroll(TRUE);
pGallery->SetButtonMode();              // 버튼 모드(아니면 드롭다운 갤러리)

// 항목 채우기
static const wchar_t* kThemes[] = { L"기본", L"차분", L"선명", L"모노", L"밤" };
for (int i=0; i<_countof(kThemes); ++i)
{
    pGallery->AddItem(kThemes[i], i); // 텍스트, 이미지 인덱스
}
pGallery->SetIconsInRow(4);  // 한 행에 4개씩
pPanelTheme->Add(pGallery);
```

### 2-3. 라이브 프리뷰(마우스 hover 시 임시 적용)

- **핵심**: 갤러리가 **Hover** 이벤트를 보낼 때 현재 선택 후보 스타일을 **미리 적용**, 마우스가 떠나면 **되돌리기**.  
- 구현 포인트:  
  - `ON_REGISTERED_MESSAGE(AFX_WM_ON_HIGHLIGHT_RIBBON_LIST_ITEM, OnHighlightRibbonItem)`  
  - `lParam`으로 **리스트 컨트롤 포인터**, `wParam`에 **인덱스** 전달 (MFC 내부 패턴)

```cpp
// MainFrm.h
protected:
    afx_msg LRESULT OnHighlightRibbonItem(WPARAM wParam, LPARAM lParam);
    int m_iPreviewTheme = -1;
    DECLARE_MESSAGE_MAP()
```

```cpp
// MainFrm.cpp
BEGIN_MESSAGE_MAP(CMainFrame, CFrameWndEx)
    ON_WM_CREATE()
    ON_REGISTERED_MESSAGE(AFX_WM_ON_HIGHLIGHT_RIBBON_LIST_ITEM, &CMainFrame::OnHighlightRibbonItem)
END_MESSAGE_MAP()

LRESULT CMainFrame::OnHighlightRibbonItem(WPARAM wParam, LPARAM lParam)
{
    int nIndex = (int)wParam;
    CMFCRibbonBaseElement* pEl = (CMFCRibbonBaseElement*)lParam;

    if (pEl != nullptr && pEl->GetID() == ID_GAL_THEME)
    {
        if (nIndex >= 0) {
            // 미리보기 적용
            ApplyThemePreview(nIndex);
            m_iPreviewTheme = nIndex;
        } else {
            // 미리보기 종료 → 원래 테마로 복귀
            CancelThemePreview();
            m_iPreviewTheme = -1;
        }
        return 1;
    }
    return 0;
}
```

> 최종 선택은 `ON_COMMAND(ID_GAL_THEME, OnThemeSelected)` 경로에서 `CMFCRibbonGallery::GetSelectedItem()`로 확인.

```cpp
// 명령 핸들러
void CMainFrame::OnThemeSelected()
{
    CMFCRibbonGallery* pGal = DYNAMIC_DOWNCAST(CMFCRibbonGallery, m_wndRibbonBar.FindByID(ID_GAL_THEME));
    if (!pGal) return;
    int sel = pGal->GetSelectedItem();
    if (sel >= 0) {
        ApplyThemeFinal(sel); // 진짜 적용 + 상태 저장
    }
}
```

---

## 3) Quick Access Toolbar(QAT) — 기본/사용자 커스터마이징/영구 저장

### 3-1. 기본 항목 추가

```cpp
// OnCreate에서:
m_wndRibbonBar.AddToQuickAccessToolbar(new CMFCRibbonButton(ID_FILE_NEW,  L"새 문서"));
m_wndRibbonBar.AddToQuickAccessToolbar(new CMFCRibbonButton(ID_FILE_OPEN, L"열기"));
m_wndRibbonBar.AddToQuickAccessToolbar(new CMFCRibbonButton(ID_FILE_SAVE, L"저장"));
```

### 3-2. 사용자 커스터마이징 허용

- 리본 바에 **커스터마이즈** 활성화: **마우스 우클릭 → QAT에 추가/제거**, 탭/명령 사용자 편집
- `CWinAppEx`의 **상태 저장/복원** API로 레지스트리에 자동 보존

```cpp
// App.cpp (종료 시점에 자동 호출됨)
void CMyApp::SaveCustomState()
{
    // CWinAppEx가 내부적으로 Ribbon/QAT 상태를 저장
    CWinAppEx::SaveCustomState();
}

void CMyApp::LoadCustomState()
{
    // 시작 시 자동 로드
    CWinAppEx::LoadCustomState();
}
```

> 필요 시 `CMFCRibbonCustomizePropertySheet`를 직접 띄워 **키보드/명령/탭 편집** UI 제공 가능.

```cpp
void CMainFrame::OnRibbonCustomize()
{
    CMFCRibbonCustomizePropertySheet dlg(&m_wndRibbonBar);
    // 명령 그룹/카테고리 등록(예: 파일/편집/보기…)
    // dlg.AddCategories(...), dlg.AddButton(ID_CMD, L"이름", L"설명");
    dlg.DoModal();
}
```

---

## 4) 애플리케이션 메뉴(파일 메뉴) / Backstage 접근

### 4-1. 기본 애플리케이션 메뉴(리본 방식)

```cpp
// 리본 앱 버튼 메뉴 구성
CMFCRibbonMainPanel* pMainPanel = m_wndRibbonBar.AddMainCategory(L"파일", IDB_FILE_16, IDB_FILE_32);
pMainPanel->Add(new CMFCRibbonButton(ID_FILE_NEW,  L"새로 만들기", 0/*icon*/, 0/*small*/));
pMainPanel->Add(new CMFCRibbonButton(ID_FILE_OPEN, L"열기", 1, 1));
pMainPanel->Add(new CMFCRibbonButton(ID_FILE_SAVE, L"저장", 2, 2));
pMainPanel->Add(new CMFCRibbonButton(ID_FILE_SAVE_AS, L"다른 이름으로 저장", 3, 3));

pMainPanel->AddRecentFilesList(L"최근 문서"); // MRU 자동 연결
```

> `AddRecentFilesList`는 `CWinAppEx::LoadStdProfileSettings()` + `AddToRecentFileList()`가 제대로 작동할 때 자동 표시.

### 4-2. Backstage 스타일

MFC 버전에 따라 **Backstage** API 지원이 다릅니다.  
공통적인 방법은 **앱 버튼**에 **메인 패널**을 구성하고, “옵션/정보/계정” 등 **자체 대화상자/뷰**를 **호출**하는 형태로 유사 UX를 구현합니다.  
(최신 MFC에서 `CMFCRibbonBackstage*` 클래스가 있다면 해당 패턴을 사용하세요. 없다면 메인 패널 + 커스텀 시트를 권장)

---

## 5) 리본 요소 집합 (버튼/스플릿/체크/라디오/콤보/스핀/색 선택/갤러리)

### 5-1. Split Button (드롭다운 + 메인 동작)

```cpp
auto* pSplit = new CMFCRibbonSplitButton(ID_EXPORT, L"내보내기", 5, 5);
CMFCRibbonButton* pCSV  = new CMFCRibbonButton(ID_EXPORT_CSV,  L"CSV");
CMFCRibbonButton* pJSON = new CMFCRibbonButton(ID_EXPORT_JSON, L"JSON");
pSplit->AddSubItem(pCSV);
pSplit->AddSubItem(pJSON);
pPanel->Add(pSplit);
```

### 5-2. 체크/라디오 그룹

```cpp
auto* pChkGrid = new CMFCRibbonCheckBox(ID_VIEW_GRID, L"그리드");
auto* pChkSnap = new CMFCRibbonCheckBox(ID_VIEW_SNAP, L"스냅");
pPanel->Add(pChkGrid);
pPanel->Add(pChkSnap);

// 라디오는 서로 다른 ID를 같은 “옵션 묶음”으로 취급(핸들러에서 상호배타 처리)
```

### 5-3. 콤보/에딧/스핀

```cpp
auto* pCombo = new CMFCRibbonComboBox(ID_VIEW_SCALE, TRUE/*edit*/, -1, 80/*width*/);
pCombo->AddItem(L"50%");
pCombo->AddItem(L"100%");
pCombo->AddItem(L"200%");
pCombo->SelectItem(1);
pPanel->Add(pCombo);

auto* pEdit = new CMFCRibbonEdit(ID_FIND_TEXT, 150, L"검색");
pPanel->Add(pEdit);

auto* pSpin = new CMFCRibbonSpinButton(ID_PARAM_LEVEL, 0, 10); // 0~10
pSpin->SetPos(5);
pPanel->Add(pSpin);
```

### 5-4. 색 선택(Color Button)

```cpp
auto* pColor = new CMFCRibbonColorButton(ID_FMT_COLOR, L"글꼴 색", 6, 6);
pColor->EnableAutomaticButton(L"자동", RGB(0,0,0));
pColor->EnableOtherButton(L"기타 색...", L"색 선택");
pPanel->Add(pColor);

// 선택 핸들링: ON_COMMAND(ID_FMT_COLOR, OnColor), CMFCRibbonColorButton::GetColor()
```

---

## 6) 명령 라우팅 & UI 업데이트 (리본과의 연결)

### 6-1. 표준 ON_COMMAND / ON_UPDATE_COMMAND_UI

```cpp
BEGIN_MESSAGE_MAP(CMainFrame, CFrameWndEx)
    ON_WM_CREATE()
    ON_COMMAND(ID_EDIT_COPY, &CMainFrame::OnEditCopy)
    ON_UPDATE_COMMAND_UI(ID_EDIT_COPY, &CMainFrame::OnUpdateEditCopy)
END_MESSAGE_MAP()

void CMainFrame::OnEditCopy()
{
    // 문서/뷰에 위임
    GetActiveView()->SendMessage(WM_COMMAND, ID_EDIT_COPY);
}

void CMainFrame::OnUpdateEditCopy(CCmdUI* pCmdUI)
{
    BOOL canCopy = /* 선택 있음? */ TRUE;
    pCmdUI->Enable(canCopy);
}
```

### 6-2. 리본 요소 상태 제어

```cpp
// 리본에서 토글 체크/텍스트 변경 등
auto* pBold = DYNAMIC_DOWNCAST(CMFCRibbonButton, m_wndRibbonBar.FindByID(ID_FMT_BOLD));
if (pBold) pBold->SetCheck(bBoldOn);

// 콤보/에딧 값 읽기
auto* pFind = DYNAMIC_DOWNCAST(CMFCRibbonEdit, m_wndRibbonBar.FindByID(ID_FIND_TEXT));
CString s; if (pFind) s = pFind->GetEditText();
```

---

## 7) HiDPI/아이콘 — 32bpp PNG·스케일·투명

### 7-1. HiColor 아이콘 활성화

```cpp
theApp.m_bHiColorIcons = TRUE; // CWinAppEx 멤버
```

### 7-2. CMFCToolBarImages 스케일

```cpp
CMFCToolBarImages imgs;
imgs.SetScaleRatio(afxGlobalData.GetRibbonImageScale());
imgs.Load(IDB_MY_ICONS_32, TRUE);
```

> 복수 크기(16/24/32/48) 리소스를 상황에 맞춰 **선택**하거나, **SVG**를 벡터로 그려 넣는 커스텀도 고려.  
> (MFC 기본은 PNG 비트맵 기반이 표준적입니다)

---

## 8) 다크 모드 / 테마 접근법

### 8-1. Visual Manager로 전역 테마 스위치

MFC는 전역 **Visual Manager**로 컨트롤의 룩앤필을 제어합니다.

```cpp
// 다크 테마 매니저(버전에 따라 이름이 다를 수 있음)
// (안전 패턴) 윈도우 스타일 매니저 계열을 사용하거나, Office/VS 스타일 중 어두운 스킨을 선택
CMFCVisualManager::SetDefaultManager(RUNTIME_CLASS(CMFCVisualManagerWindows));      // 윈도우 기본
// CMFCVisualManager::SetDefaultManager(RUNTIME_CLASS(CMFCVisualManagerWindows10)); // 가능 시
// CMFCVisualManager::SetDefaultManager(RUNTIME_CLASS(CMFCVisualManagerOffice2007));
// CMFCVisualManager::SetDefaultManager(RUNTIME_CLASS(CMFCVisualManagerVisualStudio));

// 전역 스킨 적용
CMFCVisualManager::GetInstance()->OnUpdateSystemColors();
AfxGetMainWnd()->RedrawWindow(nullptr, nullptr, RDW_INVALIDATE|RDW_ALLCHILDREN);
```

> **주의**: 실제 “완전한” 시스템 다크 모드 동기화를 하려면  
> - OS 설정(Windows 10/11) **AppsUseLightTheme** 값 모니터링  
> - `WM_SETTINGCHANGE` 처리  
> - 리본/상태바/문서 배경/컨텐츠(문서·캔버스) **직접 색**을 다크 팔레트로 스위치  
> 가 필요합니다.

### 8-2. 시스템 다크 모드 감지/토글(간단 패턴)

```cpp
bool IsSystemAppLightTheme()
{
    DWORD v = 1;
    CRegKey key;
    if (key.Open(HKEY_CURRENT_USER, L"Software\\Microsoft\\Windows\\CurrentVersion\\Themes\\Personalize", KEY_READ) == ERROR_SUCCESS)
        key.QueryDWORDValue(L"AppsUseLightTheme", v);
    return v != 0; // 1=라이트, 0=다크
}

// 설정 변경 감지
BOOL CMainFrame::OnWndMsg(UINT msg, WPARAM w, LPARAM l, LRESULT* p)
{
    if (msg == WM_SETTINGCHANGE) {
        bool light = IsSystemAppLightTheme();
        ApplyAppTheme(light ? THEME_LIGHT : THEME_DARK);
        return TRUE;
    }
    return CFrameWndEx::OnWndMsg(msg, w, l, p);
}

void CMainFrame::ApplyAppTheme(AppTheme t)
{
    // 1) VisualManager 전환(가능한 근사 스킨)
    if (t == THEME_DARK) {
        // 다크 계열 비주얼 매니저 등록(사용 가능한 가장 근사한 클래스)
        CMFCVisualManager::SetDefaultManager(RUNTIME_CLASS(CMFCVisualManagerWindows));
        // 또는 커스텀 비주얼 매니저 파생 구현
    } else {
        CMFCVisualManager::SetDefaultManager(RUNTIME_CLASS(CMFCVisualManagerWindows));
    }
    CMFCVisualManager::GetInstance()->OnUpdateSystemColors();

    // 2) 리본/상태바/문서 영역 재색칠 (사용자 정의 팔레트)
    UpdateDarkPalette(t == THEME_DARK);

    // 3) 강제 리드로우
    RedrawWindow(nullptr, nullptr, RDW_INVALIDATE|RDW_ERASE|RDW_ALLCHILDREN);
}
```

### 8-3. 사용자 팔레트(다크/라이트) 설계

```cpp
struct AppPalette {
    COLORREF clrWnd, clrText, clrAccent, clrPane, clrBorder;
} g_PalLight{ RGB(255,255,255), RGB(32,32,32), RGB(51,102,204), RGB(246,248,250), RGB(220,220,220) },
  g_PalDark { RGB(32,32,32),    RGB(230,230,230), RGB(98,141,234), RGB(45,45,48),   RGB(70,70,74) };

static AppPalette g_Pal = g_PalLight;

void UpdateDarkPalette(bool dark) { g_Pal = dark ? g_PalDark : g_PalLight; }
```

- **문서/뷰**의 배경/텍스트/그리드/하이라이트 색은 이 팔레트로 일원화합니다.  
- 리본 자체는 비주얼 매니저가 담당하지만, **갤러리/커스텀 드로우 요소**는 팔레트로 직접 채색.

---

## 9) 상태 저장/복원 (QAT/리본 구성/키보드/창 레이아웃)

`CWinAppEx`는 다음 상태를 자동 저장/복원합니다.

- **리본 커스터마이즈(QAT/탭/명령)**  
- **키보드/가속기 매핑**  
- **최근 파일(MRU)**  
- **도킹/창 레이아웃**(CFrameWndEx 기반)

```cpp
// App.cpp
void CMyApp::SaveCustomState()
{
    CWinAppEx::SaveCustomState(); // 내부적으로 레지스트리에 저장
}

void CMyApp::LoadCustomState()
{
    CWinAppEx::LoadCustomState(); // 시작 시 로드
}
```

> 커스텀 상태(예: 테마 번호/문서 뷰 옵션)는 **별도 레지스트리/JSON**에 저장.  
> (문서 저장과 동일하게 **원자적 저장(임시 파일 → 교체)** 권장)

---

## 10) 성능/UX/접근성 체크리스트

1. **명령 응답 속도**: 리본 버튼 클릭 → 핸들러 O(1), 무거운 작업은 백그라운드 + 진행 표시  
2. **툴팁/키팁**: 툴팁 텍스트/키보드 키팁(Alt 키) 확인, 현지화  
3. **키보드 사용자**: 탭 전환/포커스 이동/검색 에딧/갤러리 화살표 탐색  
4. **HiDPI**: 125/150/200%에서 아이콘 선명(32bpp PNG) 여부  
5. **다크 모드**: 시스템 전환, 사용자 토글, 팔레트 적용 영역 누락 없는지  
6. **저장/복원**: QAT/커스터마이즈가 재실행 시 그대로 복원?  
7. **접근성**: UIA/스크린리더 기본 역할 유지(리본은 기본 지원. 커스텀 드로우 영역은 대체 텍스트 고려)  
8. **국제화**: 긴 문자열/좁은 패널 제목 말줄임/팝업 너비 재조정

---

## 11) 종합 예제: “홈” 탭(클립보드/스타일/테마) + 갤러리 라이브 프리뷰 + QAT + 다크 토글

### 11-1. 리본 구성

```cpp
int CMainFrame::OnCreate(LPCREATESTRUCT cs)
{
    if (CFrameWndEx::OnCreate(cs) == -1) return -1;

    // Ribbon
    m_wndRibbonBar.Create(this);
    m_AppButton.SetImage(IDB_APPBTN_32);
    m_wndRibbonBar.SetApplicationButton(&m_AppButton, CSize(45,45));

    // 카테고리
    auto* catHome = m_wndRibbonBar.AddCategory(L"홈", IDB_HOME_16, IDB_HOME_32);

    // 클립보드
    auto* pClip = catHome->AddPanel(L"클립보드");
    pClip->Add(new CMFCRibbonButton(ID_EDIT_PASTE, L"붙여넣기", 0,0));
    pClip->Add(new CMFCRibbonButton(ID_EDIT_CUT,   L"잘라내기", 1,1));
    pClip->Add(new CMFCRibbonButton(ID_EDIT_COPY,  L"복사",     2,2));

    // 스타일
    auto* pStyle = catHome->AddPanel(L"스타일");
    auto* pBold  = new CMFCRibbonButton(ID_FMT_BOLD, L"굵게", 3,3);
    pBold->SetCheck();
    pStyle->Add(pBold);
    pStyle->Add(new CMFCRibbonButton(ID_FMT_ITALIC, L"기울임", 4,4));
    auto* pColor = new CMFCRibbonColorButton(ID_FMT_COLOR, L"글자색", 5,5);
    pColor->EnableAutomaticButton(L"자동", RGB(0,0,0));
    pColor->EnableOtherButton(L"기타 색...", L"색 선택");
    pStyle->Add(pColor);

    // 테마(갤러리)
    LoadGalleryImages();
    auto* pTheme = catHome->AddPanel(L"테마");
    auto* gal = new CMFCRibbonGallery(ID_GAL_THEME, L"테마", -1, g_GalleryImages32.GetImageSize().cx);
    gal->SetIcons(&g_GalleryImages16, &g_GalleryImages32);
    for (int i=0;i<5;i++) gal->AddItem(CString(L"테마 ") + (wchar_t)(L'A'+i), i);
    gal->SetIconsInRow(4);
    pTheme->Add(gal);

    // QAT
    m_wndRibbonBar.AddToQuickAccessToolbar(new CMFCRibbonButton(ID_FILE_NEW,  L"새 문서"));
    m_wndRibbonBar.AddToQuickAccessToolbar(new CMFCRibbonButton(ID_FILE_OPEN, L"열기"));
    m_wndRibbonBar.AddToQuickAccessToolbar(new CMFCRibbonButton(ID_FILE_SAVE, L"저장"));

    // 상태바
    m_wndStatusBar.Create(this);
    m_wndStatusBar.AddElement(new CMFCRibbonStatusBarPane(ID_INDICATOR_READY, L"준비", TRUE), L"PaneReady");

    // 다크 모드 토글(보기 탭 예시)
    auto* catView = m_wndRibbonBar.AddCategory(L"보기", IDB_VIEW_16, IDB_VIEW_32);
    auto* pThemePanel = catView->AddPanel(L"테마");
    pThemePanel->Add(new CMFCRibbonCheckBox(ID_VIEW_DARKMODE, L"다크 모드"));

    return 0;
}
```

### 11-2. 명령/업데이트/라이브 프리뷰

```cpp
BEGIN_MESSAGE_MAP(CMainFrame, CFrameWndEx)
    ON_WM_CREATE()
    ON_COMMAND(ID_FMT_BOLD,      &CMainFrame::OnFmtBold)
    ON_UPDATE_COMMAND_UI(ID_FMT_BOLD, &CMainFrame::OnUpdateFmtBold)
    ON_COMMAND(ID_FMT_COLOR,     &CMainFrame::OnFmtColor)
    ON_COMMAND(ID_GAL_THEME,     &CMainFrame::OnThemeSelected)
    ON_COMMAND(ID_VIEW_DARKMODE, &CMainFrame::OnToggleDark)
    ON_REGISTERED_MESSAGE(AFX_WM_ON_HIGHLIGHT_RIBBON_LIST_ITEM, &CMainFrame::OnHighlightRibbonItem)
END_MESSAGE_MAP()

void CMainFrame::OnFmtBold() { /* 토글 → 뷰에 브로드캐스트 */ }
void CMainFrame::OnUpdateFmtBold(CCmdUI* p) { p->SetCheck(IsBold()); }
void CMainFrame::OnFmtColor() {
    auto* p = DYNAMIC_DOWNCAST(CMFCRibbonColorButton, m_wndRibbonBar.FindByID(ID_FMT_COLOR));
    if (p) ApplyTextColor(p->GetColor());
}
void CMainFrame::OnToggleDark() {
    bool dark = IsDark(); SetDark(!dark); ApplyAppTheme(!dark ? THEME_DARK : THEME_LIGHT);
}
```

라이브 프리뷰는 2-3절 코드 재사용.

---

## 12) 문제 해결 가이드

| 증상 | 원인 | 해결 |
|---|---|---|
| 리본 아이콘 흐릿/깨짐 | 8bpp/16bpp BMP, 잘못된 마스크 | **32bpp PNG** 리소스 사용, `m_bHiColorIcons=TRUE` |
| 갤러리 프리뷰 지연 | 핸들러에서 무거운 작업 | **미리보기는 가벼운 모습만** 적용, 최종 선택 때만 무거운 초기화 |
| QAT 저장 안 됨 | `CWinAppEx` 상태 저장 미연결 | `SetRegistryKey/SetRegistryBase` + `Load/SaveCustomState()` 호출확인 |
| 다크 모드 일부만 바뀜 | 팔레트 적용 누락 | 사용자 정의 드로우(리스트/트리/캔버스)에 **팔레트 일괄 적용** |
| DPI에서 버튼 잘림 | 이미지/패딩 고정값 | `GetRibbonImageScale`/상대 패딩, 큰 아이콘 스트립 제공(16/24/32/48) |
| 백스테이지 없음 | MFC 버전/클래스 부재 | 앱 버튼 **Main Panel**로 유사 UX 구성, 옵션/정보는 대화상자/뷰로 호출 |

---

## 13) 확장 아이디어

- **명령 팔레트(Ctrl+P)**: 커맨드 검색 UI → `CMFCRibbonButton` 목록에서 텍스트 매칭  
- **Ribbon-Document 인사이트**: 문서 상태에 따라 패널 Enabled/Disabled 동적 제어  
- **라이브 프리뷰 취소/적용 패턴**: Hover→Preview, MouseLeave→Cancel, Click→Apply  
- **사용자 스킨**: `CMFCVisualManager` 파생으로 색/윤곽/선/그라디언트 일괄 커스터마이즈  
- **SVG 아이콘 파이프라인**: 빌드시 PNG로 래스터라이즈(여러 크기) → 리소스 포함

---

## 14) 요약

- **리본**: `CMFCRibbonBar` → **카테고리/패널/버튼/갤러리**를 계층적으로 구성  
- **갤러리**: 이미지 스트립 + `AFX_WM_ON_HIGHLIGHT_RIBBON_LIST_ITEM`으로 **라이브 프리뷰**  
- **QAT**: 기본 항목 + 사용자 커스터마이징 + `CWinAppEx` 상태 저장으로 **영구화**  
- **다크 모드**: **Visual Manager** + **사용자 팔레트**로 **전역/컨텐츠** 모두 전환  
- **HiDPI**: 32bpp PNG/여러 크기/스케일 API로 선명도 유지

필요하시면 위 코드를 **프로젝트 스캐폴딩**으로 묶은 “리본 스타터” 템플릿(갤러리/프리뷰/다크 토글 포함)을 만들어 드릴게요. 🙂

---

## 부록 A) 리소스 제안

- `IDB_APPBTN_32`: 32×32 PNG  
- `IDB_HOME_16`, `IDB_HOME_32`: 16/32 아이콘 스트립  
- `IDB_VIEW_16`, `IDB_VIEW_32`  
- `IDB_GALLERY_16`, `IDB_GALLERY_32` (가로 스트립)  
- 각 스트립은 동일 크기 타일로 구성(배경 투명)

## 부록 B) 커맨드 ID 예시

```cpp
#define ID_EDIT_CUT         32771
#define ID_EDIT_COPY        32772
#define ID_EDIT_PASTE       32773
#define ID_FMT_BOLD         32780
#define ID_FMT_ITALIC       32781
#define ID_FMT_COLOR        32782
#define ID_GAL_THEME        32790
#define ID_VIEW_DARKMODE    32800
```

## 부록 C) 다크 팔레트 적용 예시(리스트/트리 커스텀 드로우 연계)

```cpp
// 리스트/트리 커스텀 드로우에서 g_Pal 사용
cd->clrText   = g_Pal.clrText;
cd->clrTextBk = g_Pal.clrWnd; // 선택/핫 상태는 변형
```