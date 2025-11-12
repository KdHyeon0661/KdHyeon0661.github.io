---
layout: post
title: MFC - SDI, MDI 구조
date: 2025-08-26 17:25:23 +0900
category: MFC
---
# SDI/MDI 구조 이해: `CDocument` / `CView` / `CFrameWnd`, DocTemplate, 명령 라우팅 완전 정리
이 글은 MFC의 **문서/뷰 구조(도큐먼트-뷰-프레임)** 를 **SDI/MDI** 관점에서 끝까지 파고듭니다.  
핵심 클래스의 역할, 생성·수명 주기, **DocTemplate 연결**, **명령/업데이트 라우팅**, **멀티 뷰 구성**, **프린팅 파이프라인**, **MDI 전용 이슈**를 **개념→코드→체크리스트** 순으로 정리합니다.  
(본문은 ~~~markdown, 코드는 ```로 감싸 제공합니다.)

---

## 0. 큰 그림: 왜 Doc/View인가?

- **분리(Separation of Concerns)**
  - **`CDocument`**: 데이터·모델(파일 로드/저장, 직렬화, 변경 여부, 뷰에 방송)
  - **`CView`**: 표시·편집(그리기, 입력 처리, 도구/상태 갱신)
  - **`CFrameWnd`**: 컨테이너(메뉴·툴바·상태바·도킹, 활성 뷰 관리)
- **확장성**
  - 한 문서에 **여러 뷰**(표/그래프/속성)를 동시에 띄워 **동일 데이터**를 다른 관점으로 표현
- **표준화**
  - 새로 만들기/열기/저장/인쇄/MRU/프리뷰 등 **파일 기반 앱의 정석 흐름**을 프레임워크가 지원

---

## 1. 핵심 클래스의 역할과 경계

### 1-1. `CDocument` (모델)
- **데이터 보유** 및 **직렬화**
- **수정 플래그**: `SetModifiedFlag(TRUE)` → 종료/닫기 시 저장 확인을 자동화
- **브로드캐스트**: `UpdateAllViews(pSender, lHint, pHint)`로 뷰 갱신 통지
- **수명**: 문서를 표시하는 **모든 프레임/뷰가 닫혀야** 소멸

```cpp
// MyDoc.h
class CMyDoc : public CDocument
{
protected: DECLARE_DYNCREATE(CMyDoc)
public:
    std::vector<CString> m_items;

    virtual BOOL OnNewDocument() override {
        if (!CDocument::OnNewDocument()) return FALSE;
        m_items = { _T("alpha"), _T("beta"), _T("gamma") };
        SetModifiedFlag(FALSE);
        return TRUE;
    }
    virtual void Serialize(CArchive& ar) override {
        if (ar.IsStoring()) {
            UINT n = (UINT)m_items.size(); ar << n;
            for (auto& s : m_items) ar << s;
        } else {
            UINT n=0; ar >> n; m_items.clear(); m_items.reserve(n);
            for (UINT i=0;i<n;++i){ CString s; ar>>s; m_items.push_back(s); }
        }
    }

    void AddItem(const CString& s){
        m_items.push_back(s);
        SetModifiedFlag(TRUE);
        UpdateAllViews(nullptr, 1 /*HINT_ITEM_ADDED*/, (CObject*)&m_items.back());
    }
};
```

### 1-2. `CView` (뷰)
- **그리기**: `OnDraw(CDC*)`가 화면 렌더의 중심
- **입력 처리**: 메뉴/툴바/단축키 명령(커맨드)과 마우스/키보드 메시지
- **문서 접근**: `GetDocument()`
- **초기화**: `OnInitialUpdate()`(스크롤 범위/컨트롤 배치 등)
- **갱신 수신**: `OnUpdate(pSender, lHint, pHint)`

```cpp
// MyView.h
class CMyView : public CView
{
protected: DECLARE_DYNCREATE(CMyView)
public:
    virtual void OnDraw(CDC* pDC) override {
        const auto* pDoc = GetDocument();
        int y=10; for (auto& s : pDoc->m_items) { pDC->TextOut(10, y, s); y+=20; }
    }
    virtual void OnInitialUpdate() override {
        CView::OnInitialUpdate();
        // 스크롤뷰라면 SetScrollSizes 등
    }
    virtual void OnUpdate(CView* pSender, LPARAM lHint, CObject* pHint) override {
        // 힌트 기반 부분 갱신 가능. 여기선 전체 무효화
        Invalidate();
    }
    CMyDoc* GetDocument() const { return static_cast<CMyDoc*>(m_pDocument); }

    afx_msg void OnCmdAddAlpha(){ GetDocument()->AddItem(_T("added via view")); }
    DECLARE_MESSAGE_MAP()
};

BEGIN_MESSAGE_MAP(CMyView, CView)
    ON_COMMAND(ID_EDIT_ADDALPHA, &CMyView::OnCmdAddAlpha)
END_MESSAGE_MAP()
```

### 1-3. `CFrameWnd` / `CMDIFrameWnd` / `CMDIChildWnd`
- 메뉴/툴바/상태바/도킹·리본 관리
- 활성 뷰 포커스/전환
- **SDI**: `CFrameWnd(Ex)` 1개가 문서/뷰를 담음  
- **MDI**: 메인 `CMDIFrameWnd(Ex)` + 문서마다 `CMDIChildWnd(Ex)`로 분리

```cpp
// MainFrm.h (SDI 예시)
class CMainFrame : public CFrameWndEx
{
protected: DECLARE_DYNCREATE(CMainFrame)
public:
    CMFCRibbonBar m_wndRibbonBar;
    virtual BOOL PreCreateWindow(CREATESTRUCT& cs) override { return CFrameWndEx::PreCreateWindow(cs); }
};
```

---

## 2. DocTemplate: 문서-프레임-뷰를 연결하는 허브

### 2-1. 역할
- **런타임 클래스 3종**(Doc/Frame/View) 연결
- **파일 형식 메타**(DocString) 제공
- **생성 파이프라인**(새 문서/열기)에서 객체를 **공장 패턴**으로 생성·결합

### 2-2. 유형
- **`CSingleDocTemplate`**: SDI용(문서 1개)
- **`CMultiDocTemplate`**: MDI용(문서 여러 개, Child Frame 생성)

### 2-3. 초기화 코드 (SDI/MDI 공통 패턴)

```cpp
// MyApp.cpp
BOOL CMyApp::InitInstance()
{
    CWinAppEx::InitInstance();
    SetRegistryKey(_T("Acme"));
    LoadStdProfileSettings(10);

    // SDI:
    auto pDocTemplate = new CSingleDocTemplate(
        IDR_MAINFRAME,
        RUNTIME_CLASS(CMyDoc),
        RUNTIME_CLASS(CMainFrame),
        RUNTIME_CLASS(CMyView));
    AddDocTemplate(pDocTemplate);

    // 첫 문서 자동 생성(SDI 기본)
    CCommandLineInfo cmdInfo; ParseCommandLine(cmdInfo);
    if (!ProcessShellCommand(cmdInfo)) return FALSE;
    m_pMainWnd->ShowWindow(SW_SHOW); m_pMainWnd->UpdateWindow();
    return TRUE;
}
```

```cpp
// MDI의 경우
auto pMDITemplate = new CMultiDocTemplate(
    IDR_MyDocTYPE,
    RUNTIME_CLASS(CMyDoc),
    RUNTIME_CLASS(CChildFrame),   // CMDIChildWndEx
    RUNTIME_CLASS(CMyView));
AddDocTemplate(pMDITemplate);

auto* pMainFrame = new CMainFrame; // CMDIFrameWndEx
if (!pMainFrame->LoadFrame(IDR_MAINFRAME)) return FALSE;
m_pMainWnd = pMainFrame;
```

### 2-4. DocString (리소스의 \n 구획)
- 메뉴 이름/파일 필터/확장자/타이틀 템플릿 등
- **파일 열기/저장 대화상자**의 필터, **창 타이틀** 포맷 등에 사용

---

## 3. 수명 주기: 생성·열기·저장·닫기

### 3-1. 새 문서(SDI 기본 흐름)
1) `InitInstance` → DocTemplate 등록  
2) `ProcessShellCommand` → `ID_FILE_NEW` → `OpenDocumentFile(nullptr)`  
3) **문서 생성** → **프레임 생성** → **뷰 생성/부착** → `OnInitialUpdate`  
4) 프레임 표시/활성화

### 3-2. 파일 열기
- `OpenDocumentFile(L"path")` → 문서 생성 → `CDocument::OnOpenDocument` 로드 → 프레임/뷰 결합

### 3-3. 저장/다른 이름으로 저장
- `CDocument::DoFileSave()` / `OnSaveDocument(L"path")`  
- 저장 성공 후 `SetModifiedFlag(FALSE)`

### 3-4. 닫기/종료
- 문서가 수정됨 → `SaveModified()`로 저장 여부 확인  
- **SDI**: 창 닫기 = 문서 종료  
- **MDI**: 활성 Child 닫기 = 해당 문서 종료

---

## 4. 명령/업데이트 라우팅 (핵심)

### 4-1. 검색 순서

**SDI**
1) 활성 **`CView`**  
2) **`CFrameWnd`**  
3) **`CDocument`**  
4) **`CWinApp`**

**MDI**
1) 활성 Child의 **`CView`**  
2) 그 Child의 **`CMDIChildWnd`**  
3) 그 Child의 **`CDocument`**  
4) **`CMDIFrameWnd`**  
5) **`CWinApp`**

> 데이터 조작은 **문서**, 표시/도구는 **뷰**, 전역/창은 **프레임/앱**에 핸들러를 배치하면 자연스럽게 작동합니다.

```cpp
// 동일 ID가 여러 클래스에 있어도, 라우팅 순서에 따라 "가장 가까운" 곳이 실행됨
BEGIN_MESSAGE_MAP(CMyDoc, CDocument)
    ON_COMMAND(ID_EDIT_ADDALPHA, &CMyDoc::OnAddFromDoc) // View에 없으면 여기로 떨어짐
END_MESSAGE_MAP()
```

### 4-2. `ON_UPDATE_COMMAND_UI` (메뉴/툴바 상태)
- Enable/Disable, Check/Radio, Text 업데이트
- **Idle 타임**마다 호출 → 비싼 연산/파일 I/O 금지, **캐시된 상태**만 빠르게 반영

```cpp
// View: 선택 항목이 있어야 삭제 가능하도록
afx_msg void CMyView::OnUpdateEditDelete(CCmdUI* pCmdUI)
{
    bool hasSel = !GetDocument()->m_items.empty();
    pCmdUI->Enable(hasSel);
}
BEGIN_MESSAGE_MAP(CMyView, CView)
    ON_UPDATE_COMMAND_UI(ID_EDIT_DELETE, &CMyView::OnUpdateEditDelete)
END_MESSAGE_MAP()
```

---

## 5. 멀티 뷰/스플리터/Hint 패턴

### 5-1. 한 문서—여러 뷰
- `UpdateAllViews`로 **방송**
- 각 뷰는 `OnUpdate(lHint, pHint)`에서 **관심 있는 부분만** 갱신

```cpp
// Hint 상수
enum { HINT_ITEM_ADDED=1, HINT_SELECTION=2 };

void CMyDoc::AddItem(const CString& s)
{
    m_items.push_back(s); SetModifiedFlag(TRUE);
    UpdateAllViews(nullptr, HINT_ITEM_ADDED, (CObject*)&m_items.back());
}
```

```cpp
// 뷰 A/B가 서로 다른 방식으로 반응
void CMyView::OnUpdate(CView* s, LPARAM hint, CObject* p)
{
    if (hint == HINT_ITEM_ADDED) {
        // 마지막 줄만 무효화 등 부분 갱신 가능
    }
    Invalidate();
}
```

### 5-2. 런타임 뷰 교체(고급)
- `CFrameWnd::SetActiveView` + 기존 뷰 파괴/새 뷰 생성
- **소유권은 프레임**: 뷰를 직접 `delete`하지 말고 프레임 API 경유

```cpp
// 간단 스케치: 기존 뷰 교체
void ReplaceView(CFrameWnd* pFrame, CRuntimeClass* pNewViewClass)
{
    CView* pOld = pFrame->GetActiveView();
    CCreateContext cc{}; cc.m_pCurrentDoc = pOld->GetDocument();
    auto* pNew = (CView*)pNewViewClass->CreateObject();
    pNew->Create(nullptr, nullptr, AFX_WS_DEFAULT_VIEW, CRect(), pFrame, AFX_IDW_PANE_FIRST, &cc);
    pFrame->SetActiveView(pNew);
    pNew->OnInitialUpdate();
    pOld->DestroyWindow(); // 프레임이 소유, 여기서 파괴
    pFrame->RecalcLayout();
}
```

### 5-3. 스플리터 `CSplitterWnd`
- 한 프레임 안에 **행/열로 여러 뷰** 배치

```cpp
// Child/Frame에서 OnCreateClient 재정의 (MDI도 동일)
BOOL CMainFrame::OnCreateClient(LPCREATESTRUCT, CCreateContext* pCC)
{
    if (!m_wndSplit.CreateStatic(this, 1, 2)) return FALSE;
    m_wndSplit.CreateView(0, 0, RUNTIME_CLASS(CMyView), CSize(400,0), pCC);
    m_wndSplit.CreateView(0, 1, RUNTIME_CLASS(CMyListView), CSize(400,0), pCC);
    return TRUE;
}
```

---

## 6. 프린팅 파이프라인 (요약 & 연결)

1) `CView::OnPreparePrinting` → 페이지 수 계산  
2) `OnBeginPrinting` → 폰트/펜/브러시  
3) `OnPrepareDC` → 맵핑 모드/오리진  
4) `OnPrint` → **문서 데이터**를 **뷰 방식으로** 렌더  
5) `OnEndPrinting` → 정리

> 프린트 프리뷰는 동일 경로에서 **`CPreviewDC`** 를 사용할 뿐입니다.

---

## 7. MDI 전용 이슈(SDI와 다른 지점)

- **Child Frame**: 문서마다 독립 창 (`CMDIChildWndEx`)  
- **명령/업데이트 기준**: **활성 Child**의 뷰/문서가 우선  
- **메뉴 병합**: 활성 Child 메뉴가 메인 메뉴와 **동적 병합**  
- **창 관리**: Window 메뉴에 Child 목록/전환, 타일/캐스케이드

```cpp
// MDI: 문서 추가 생성(새 Child 띄우기)
void CMainFrame::OnFileNewDoc()
{
    POSITION pos = AfxGetApp()->GetFirstDocTemplatePosition();
    auto* pTpl = AfxGetApp()->GetNextDocTemplate(pos); // 첫 템플릿
    pTpl->OpenDocumentFile(nullptr); // 새 문서 → 새 Child 생성
}
```

---

## 8. 명령 설계 베스트 프랙티스

1. **데이터 변경 명령**은 **문서**에, **표시/선택/도구 명령**은 **뷰**에
2. **ON_UPDATE_COMMAND_UI**는 빠르게: 상태 비트/캐시 읽기만
3. **Hint 패턴**으로 부분 갱신: 대형 캔버스/리스트 성능 확보
4. **DocTemplate**와 **표준 ID**(열기/저장/인쇄)를 적극 활용
5. **MDI**: 활성 Child 개념을 전제로 단축키/업데이트 로직 설계
6. **문서/뷰 수명**은 프레임 흐름에 따르며 **수동 delete 금지**

---

## 9. 문제 해결 가이드

| 증상 | 원인 | 해결 |
|---|---|---|
| 명령 핸들러가 안 불림 | 핸들러가 라우팅 순서 바깥 클래스에 위치 | 핸들러 위치 재배치(뷰/문서/프레임/앱 순서) |
| 메뉴/툴바 상태가 이상 | Update 핸들러가 무거움, 활성 프레임 오인 | O(1) 처리, 활성 Child/뷰 기준으로 상태 결정 |
| 뷰 간 갱신 지연 | `UpdateAllViews` 누락 또는 너무 광범위 | Hint로 **부분 갱신**, 필요 시 해당 영역만 무효화 |
| 문서/뷰 소멸 충돌 | 뷰를 직접 `delete` | 프레임이 소유. `DestroyWindow`/프레임 API 사용 |
| DocString 필터 오동작 | 리소스 문자열 토큰 오류 | DocString 형식(`\n` 구획) 재검증 |

---

## 10. SDI vs MDI 선택 가이드

| 항목 | SDI | MDI |
|---|---|---|
| 사용 패턴 | 단일 문서 집중 | 다문서 동시 비교/편집 |
| UI 복잡도 | 낮음 | 높음(Child/메뉴 병합) |
| 멀티 뷰 | 스플리터/패널로 대응 | Child 여러 개 + 다양한 뷰 |
| 유지보수 | 쉬움 | 상대적으로 어려움 |
| 예시 | 메모장형/뷰어 | IDE/그래픽 툴/탭형 에디터 |

---

## 11. 한 장으로 보는 전체 시퀀스

```
[사용자] --(ID_FILE_OPEN)--> [App]
      └--> [DocTemplate.OpenDocumentFile]
             ├─[CDocument 생성 → OnOpenDocument]
             ├─[CFrameWnd/CMDIChildWnd 생성]
             ├─[CView 생성 → OnInitialUpdate]
             └─[프레임 활성/표시]

(편집/명령) → [CView 핸들러] → [CDocument 데이터 변경/SetModifiedFlag]
                          └─→ [CDocument.UpdateAllViews(hint)]

(종료/닫기) → [CDocument.SaveModified] → 저장/폐기/취소
```

---

## 12. 실무용 코드 모음 (스니펫)

### 12-1. 문서→뷰 방송(선택 변경 힌트)

```cpp
// Doc
struct SelHint : public CObject { int index=-1; };
void CMyDoc::SelectIndex(int idx)
{
    auto* h = new SelHint; h->index = idx;
    UpdateAllViews(nullptr, HINT_SELECTION, h); // 프레임워크가 delete
}
```

```cpp
// View
void CMyView::OnUpdate(CView*, LPARAM hint, CObject* p)
{
    if (hint == HINT_SELECTION) {
        auto* h = static_cast<SelHint*>(p);
        m_curSel = h->index;  // 부분 렌더만
        Invalidate(FALSE);
        return;
    }
    Invalidate();
}
```

### 12-2. MRU/표준 파일 명령 연결

```cpp
BEGIN_MESSAGE_MAP(CMyApp, CWinAppEx)
    ON_COMMAND(ID_FILE_NEW, &CWinAppEx::OnFileNew)
    ON_COMMAND(ID_FILE_OPEN, &CWinAppEx::OnFileOpen)
    ON_UPDATE_COMMAND_UI(ID_FILE_MRU_FILE1, &CWinAppEx::OnUpdateRecentFileMenu)
    ON_COMMAND_RANGE(ID_FILE_MRU_FILE1, ID_FILE_MRU_FILE16, &CWinAppEx::OnOpenRecentFile)
END_MESSAGE_MAP()
```

### 12-3. 가속기(단축키) 등록

```cpp
// 리소스에서 IDR_MAINFRAME에 Accelerator 정의
// 프레임워크가 자동 로드. 커맨드는 동일 라우팅을 따름.
```

---

## 13. 체크리스트 (요약)

- [ ] Doc/View/Frame **역할 분리**가 명확한가  
- [ ] **DocTemplate** 등록/DocString 구성 적절한가  
- [ ] **명령/업데이트 라우팅**을 기준으로 핸들러 위치를 설계했는가  
- [ ] **Hint 기반** 부분 갱신을 사용하여 성능을 확보했는가  
- [ ] 멀티 뷰/스플리터/동적 교체에서 **프레임 소유권 규칙**을 지켰는가  
- [ ] 프린팅 파이프라인(Prepare/Begin/PrepareDC/Print/End)을 재사용 가능한 **레이아웃-렌더** 구조로 설계했는가  
- [ ] MDI라면 **활성 Child** 개념을 UI/단축키/업데이트에 반영했는가

---

### 마무리

- **Doc/View/Frame**은 여전히 강력한 **데스크톱 아키텍처**입니다.  
- **DocTemplate**는 **결합도를 낮춘 생성 허브**, **명령/업데이트 라우팅**은 **핸들러 배치의 규율**입니다.  
- 본문의 스니펫을 바탕으로 SDI/MDI, 멀티 뷰, 프린팅까지 한 흐름으로 통일하면 **확장성과 유지보수성**을 동시에 얻을 수 있습니다.