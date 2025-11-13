---
layout: post
title: MFC - Custom/Owner-Draw & Custom-Draw
date: 2025-08-26 21:25:23 +0900
category: MFC
---
# Custom/Owner-Draw & Custom-Draw: 리스트/트리 커스터마이징 (MFC 완전 정리 · 실전 예제 다수)

이 글은 MFC에서 **리스트/트리(및 고전 리스트박스/콤보)**를 “내 스타일대로” 그리기 위해 필요한 **Owner-Draw**와 **Custom-Draw**를 **개념 → 설계 → 코드 → 성능/UX → 체크리스트** 순서로, 생략 없이 정리합니다.
모든 코드는 바로 붙여쓸 수 있도록 **독립 스니펫**을 제공합니다.

---

## 0. 큰 그림 요약

- **Owner-Draw**
  - 컨트롤이 **그리기 책임을 ‘소유자(부모)’에게 전가**.
  - 대화상자/뷰(혹은 컨트롤 파생 클래스)의 **`DrawItem`/`MeasureItem`** 오버라이드로 **완전한 커스텀**.
  - 대상: **ListBox/ComboBox**(고전 컨트롤), **Button** 등. (ListView/TreeView는 Owner-Draw가 아닌 **Custom-Draw** 사용이 일반적)
- **Custom-Draw**
  - 공용 컨트롤(예: **ListView=`CListCtrl`**, **TreeView=`CTreeCtrl`**)이 **그리기 단계별 훅**을 노출
  - **`NM_CUSTOMDRAW`** 통지로 **전처리/항목/서브아이템/후처리**를 단계적으로 커스터마이징
  - 장점: **가상 모드(LVS_OWNERDATA)**, **체크박스/그룹/서브아이템** 등과 **궁합 최고**

> 선택 기준
> - **ListBox/Combo** = Owner-Draw
> - **ListView/TreeView** = Custom-Draw (필요 시 Owner-Data=가상 모드와 병행)

---

## 1. Owner-Draw (ListBox/Combo) — 기본과 고급

### 1-1. 스타일/흐름
- **리소스 스타일**
  - ListBox: `LBS_OWNERDRAWFIXED` / `LBS_OWNERDRAWVARIABLE`(+ `LBS_HASSTRINGS`)
  - ComboBox: `CBS_OWNERDRAWFIXED` / `CBS_OWNERDRAWVARIABLE`(+ `CBS_HASSTRINGS`)
- **핵심 가상 함수**
  - `void DrawItem(LPDRAWITEMSTRUCT)` — **실제 그리기**
  - `void MeasureItem(LPMEASUREITEMSTRUCT)` — **항목 높이 측정**(VARIABLE일 때 필수)
  - `void DeleteItem(LPDELETEITEMSTRUCT)` — (선택) 항목 삭제 후 정리
- **상태 플래그**: `ODS_SELECTED`, `ODS_FOCUS`, `ODS_DISABLED`, `ODS_COMBOBOXEDIT` …

### 1-2. 고정/가변 높이 ListBox 예제

```cpp
// 목록 항목에 색상칩 + 텍스트를 그리는 Owner-Draw ListBox
class CColorListBox : public CListBox {
    DECLARE_MESSAGE_MAP()
public:
    // 가변 높이 모드라면 MeasureItem 필수
    void MeasureItem(LPMEASUREITEMSTRUCT mis) override {
        // DPI 96 기준 20px → 폰트/DPI에 맞게 조정해도 좋음
        mis->itemHeight = 20;
    }
    void DrawItem(LPDRAWITEMSTRUCT dis) override {
        CDC dc; dc.Attach(dis->hDC);
        CRect r = dis->rcItem;
        const bool sel = (dis->itemState & ODS_SELECTED) != 0;
        const bool focus = (dis->itemState & ODS_FOCUS) != 0;

        // 배경
        COLORREF bk = sel ? RGB(210, 230, 255) : RGB(255, 255, 255);
        dc.FillSolidRect(r, bk);

        // 항목 데이터에서 색/텍스트 가져오기
        CString text;
        if (dis->itemID != (UINT)-1)
            GetText(dis->itemID, text);
        COLORREF c = (COLORREF)GetItemData(dis->itemID);

        // 색상칩
        CRect chip = r; chip.right = chip.left + 24; chip.DeflateRect(3, 3);
        dc.FillSolidRect(chip, c);
        dc.FrameRect(chip, &CBrush(RGB(120, 120, 120)));

        // 텍스트
        r.left = chip.right + 8;
        dc.SetBkMode(TRANSPARENT);
        dc.SetTextColor(RGB(30, 30, 30));
        dc.DrawText(text, r, DT_SINGLELINE | DT_VCENTER | DT_END_ELLIPSIS);

        // 포커스 사각
        if (focus && (dis->itemAction & (ODA_DRAWENTIRE | ODA_FOCUS)))
            dc.DrawFocusRect(r);

        dc.Detach();
    }
};
```

**포인트**
- `MeasureItem`은 **가변 높이**일 때만 호출. (FIXED면 ListBox의 `ItemHeight` 속성 사용)
- **더블 버퍼링**은 보통 필요 없음(항목 단위 그리기). 깜빡임이 보이면 `WM_ERASEBKGND` 최적화 검토.

### 1-3. 오너드로우 콤보(미리보기 포함)

```cpp
class CColorCombo : public CComboBox {
public:
    void MeasureItem(LPMEASUREITEMSTRUCT mis) override { mis->itemHeight = 20; }
    void DrawItem(LPDRAWITEMSTRUCT dis) override {
        CDC dc; dc.Attach(dis->hDC);
        CRect r = dis->rcItem;
        bool sel = (dis->itemState & ODS_SELECTED) != 0;

        dc.FillSolidRect(r, sel ? RGB(232,244,255) : RGB(255,255,255));

        CString text;
        if (dis->itemID != (UINT)-1) GetLBText(dis->itemID, text);
        COLORREF c = (COLORREF)GetItemData(dis->itemID);

        CRect chip = r; chip.right = chip.left + 24; chip.DeflateRect(3, 3);
        dc.FillSolidRect(chip, c); dc.FrameRect(chip, &CBrush(RGB(100,100,100)));

        r.left = chip.right + 8; dc.SetBkMode(TRANSPARENT);
        dc.DrawText(text, r, DT_SINGLELINE | DT_VCENTER);

        dc.Detach();
    }
};
```

> 콤보의 “에디트 부분”은 `ODS_COMBOBOXEDIT`로 구분 가능.
> 드롭다운 리스트 영역과 에디트 영역을 다르게 그리려면 조건 분기.

---

## 2. Custom-Draw (ListView=`CListCtrl`) — 단계/기법 총정리

### 2-1. 단계(DWORD `dwDrawStage`)
- `CDDS_PREPAINT` → **아이템 단위 알림 허용 요청**: `*pResult = CDRF_NOTIFYITEMDRAW`
- `CDDS_ITEMPREPAINT` → 행 단위 커스터마이즈
  - 여기서 `*pResult = CDRF_NOTIFYSUBITEMDRAW` 반환 시 `CDDS_SUBITEM|CDDS_ITEMPREPAINT` 단계가 옴
- `CDDS_SUBITEM|CDDS_ITEMPREPAINT` → **서브아이템(열)** 단위 커스터마이즈
- `CDDS_POSTPAINT`/`CDDS_ITEMPOSTPAINT` → 후처리(오버레이/선 그리기 등)

### 2-2. 메시지 맵

```cpp
BEGIN_MESSAGE_MAP(CMyDlg, CDialogEx)
    ON_NOTIFY(NM_CUSTOMDRAW, IDC_LISTCTRL, &CMyDlg::OnListCustomDraw)
END_MESSAGE_MAP()
```

### 2-3. 대표 시나리오: **지브라(홀짝 줄무늬) + 선택/핫트래킹 + 특정 열 색상**

```cpp
void CMyDlg::OnListCustomDraw(NMHDR* pNMHDR, LRESULT* pResult)
{
    LPNMLVCUSTOMDRAW cd = reinterpret_cast<LPNMLVCUSTOMDRAW>(pNMHDR);

    switch (cd->nmcd.dwDrawStage)
    {
    case CDDS_PREPAINT:
        *pResult = CDRF_NOTIFYITEMDRAW; // 아이템 단계 통지 원함
        return;

    case CDDS_ITEMPREPAINT:
    {
        const int row = (int)cd->nmcd.dwItemSpec;
        const bool selected = (cd->nmcd.uItemState & CDIS_SELECTED) != 0;
        const bool hot = (cd->nmcd.uItemState & CDIS_HOT) != 0;

        if (selected) {
            cd->clrText = RGB(255,255,255);
            cd->clrTextBk = RGB(51, 102, 204);  // 선택 배경
        } else {
            cd->clrText = RGB(30,30,30);
            cd->clrTextBk = (row % 2 == 0) ? RGB(248,250,253) : RGB(255,255,255);
            if (hot) cd->clrTextBk = RGB(230, 240, 255); // 호버
        }
        *pResult = CDRF_NOTIFYSUBITEMDRAW; // 서브아이템 단계도 요청
        return;
    }

    case CDDS_SUBITEM | CDDS_ITEMPREPAINT:
    {
        const int col = cd->iSubItem;
        // 예: 1번째 열(“크기”)만 파랑 텍스트
        if (col == 1 && !(cd->nmcd.uItemState & CDIS_SELECTED)) {
            cd->clrText = RGB(0, 80, 160);
        }
        *pResult = CDRF_DODEFAULT;
        return;
    }
    }
    *pResult = CDRF_DODEFAULT;
}
```

**포인트**
- **선택 행**은 보통 시스템 색을 쓰는 것이 접근성/테마 호환에 좋지만, **일관된 스타일**을 원하면 직접 색 지정.
- 깜빡임 최소화를 위해 `LVS_EX_DOUBLEBUFFER` 사용 추천.

```cpp
m_list.SetExtendedStyle(m_list.GetExtendedStyle() | LVS_EX_DOUBLEBUFFER | LVS_EX_FULLROWSELECT);
```

### 2-4. “진짜 그리기”가 필요한 경우(텍스트 대신 커스텀 요소)

#### (A) 진행 막대/게이지를 서브아이템에 직접 렌더

```cpp
void DrawProgress(CDC& dc, const CRect& cell, int percent) {
    CRect bar = cell; bar.DeflateRect(4, 4);
    dc.FillSolidRect(bar, RGB(240,240,240));
    dc.Draw3dRect(bar, RGB(200,200,200), RGB(200,200,200));
    CRect fill = bar; fill.right = bar.left + MulDiv(bar.Width(), percent, 100);
    fill.DeflateRect(1,1);
    dc.FillSolidRect(fill, RGB(76, 175, 80));
}
```

```cpp
case CDDS_SUBITEM | CDDS_ITEMPREPAINT:
{
    const int row = (int)cd->nmcd.dwItemSpec;
    const int col = cd->iSubItem;
    if (col == 2) { // 예: 3열에 진행률
        // 직접 그릴려면 기본 텍스트 드로잉 차단 후 수동 드로잉
        *pResult = CDRF_SKIPDEFAULT; // 기본 그리기 생략
        CDC* pDC = CDC::FromHandle(cd->nmcd.hdc);
        CRect rc; m_list.GetSubItemRect(row, col, LVIR_BOUNDS, rc);
        int progress = GetProgressForRow(row); // 0~100
        DrawProgress(*pDC, rc, progress);
        return;
    }
    *pResult = CDRF_DODEFAULT;
    return;
}
```

#### (B) 체크박스/상태 아이콘 직접 렌더
- 기본 체크박스(`LVS_EX_CHECKBOXES`) 대신 **상태이미지** 또는 사용자 지정 그래픽을 쓰려면 `CDRF_SKIPDEFAULT` 후 직접 그리기.

```cpp
void DrawCheck(CDC& dc, const CRect& cell, bool on) {
    CRect box = cell; box.DeflateRect(6, (cell.Height()-16)/2);
    box.right = box.left + 16;
    dc.FillSolidRect(box, RGB(255,255,255));
    dc.Draw3dRect(box, RGB(160,160,160), RGB(160,160,160));
    if (on) {
        CPen pen(PS_SOLID, 2, RGB(34,139,34));
        CPen* old = dc.SelectObject(&pen);
        dc.MoveTo(box.left+3, box.CenterPoint().y);
        dc.LineTo(box.left+7, box.bottom-3);
        dc.LineTo(box.right-3, box.top+3);
        dc.SelectObject(old);
    }
}
```

> 클릭 처리: `NM_CLICK`에서 **히트테스트** 후 해당 셀 범위 안이면 상태 토글.

```cpp
void CMyDlg::OnListClick(NMHDR* pNMHDR, LRESULT* pResult) {
    LPNMITEMACTIVATE ia = (LPNMITEMACTIVATE)pNMHDR;
    if (ia->iItem >= 0 && ia->iSubItem == 0) { // 예: 첫 열이 체크
        CRect rc; m_list.GetSubItemRect(ia->iItem, 0, LVIR_BOUNDS, rc);
        CPoint pt(ia->ptAction);
        if (rc.PtInRect(pt)) {
            ToggleCheck(ia->iItem);
            m_list.RedrawItems(ia->iItem, ia->iItem);
        }
    }
    *pResult = 0;
}
```

### 2-5. 가상 리스트(`LVS_OWNERDATA`)와 Custom-Draw

- **항목 갯수만** 지정하고, 표시 시점에 **텍스트/이미지 공급** (`LVN_GETDISPINFO`)
- Custom-Draw는 **셀 단위 색/배경/수동 그림**에 그대로 적용 가능
- 주의: **항목 인덱스 → 모델 인덱스** 매핑을 확실히(정렬/필터 시)

```cpp
// 가상 리스트 필수: OnGetDispInfo
ON_NOTIFY(LVN_GETDISPINFO, IDC_LIST, &CMyDlg::OnGetDispInfo)

void CMyDlg::OnGetDispInfo(NMHDR* n, LRESULT* r) {
    auto di = (NMLVDISPINFO*)n;
    if (di->item.mask & LVIF_TEXT) {
        auto row = di->item.iItem, col = di->item.iSubItem;
        auto s = Model_Text(row, col);
        _tcsncpy_s(di->item.pszText, di->item.cchTextMax, s, _TRUNCATE);
    }
    *r = 0;
}
```

---

## 3. Custom-Draw (TreeView=`CTreeCtrl`) — 전체 행/노드 스타일링

### 3-1. 필수 스타일/확장 스타일
- `TVS_HASBUTTONS`, `TVS_HASLINES`, `TVS_LINESATROOT`, `TVS_FULLROWSELECT`, `TVS_TRACKSELECT`
- 확장: `TVS_EX_DOUBLEBUFFER`(테마 환경에서 깜빡임 저감; OS/버전에 따라 다름)

```cpp
m_tree.ModifyStyle(0, TVS_FULLROWSELECT | TVS_TRACKSELECT);
```

### 3-2. 커스텀드로우 단계 (`NMTVCUSTOMDRAW`)
- `CDDS_PREPAINT` → `CDRF_NOTIFYITEMDRAW`
- `CDDS_ITEMPREPAINT` → 노드별 색/배경 지정
- (`CDDS_ITEMPOSTPAINT` → 후처리 가능)

```cpp
BEGIN_MESSAGE_MAP(CMyDlg, CDialogEx)
    ON_NOTIFY(NM_CUSTOMDRAW, IDC_TREE, &CMyDlg::OnTreeCustomDraw)
END_MESSAGE_MAP()

void CMyDlg::OnTreeCustomDraw(NMHDR* n, LRESULT* r){
    auto cd = (NMTVCUSTOMDRAW*)n;
    switch (cd->nmcd.dwDrawStage) {
    case CDDS_PREPAINT: *r = CDRF_NOTIFYITEMDRAW; return;
    case CDDS_ITEMPREPAINT: {
        HTREEITEM hItem = (HTREEITEM)cd->nmcd.dwItemSpec;
        bool sel = (cd->nmcd.uItemState & CDIS_SELECTED)!=0;
        bool hot = (cd->nmcd.uItemState & CDIS_HOT)!=0;

        if (sel) { cd->clrText = RGB(255,255,255); cd->clrTextBk = RGB(51, 102, 204); }
        else     { cd->clrText = RGB(30,30,30);    cd->clrTextBk = hot?RGB(235,244,255):RGB(255,255,255); }

        // 예: 특정 노드는 빨갛게
        if (IsAlertNode(hItem) && !sel) cd->clrText = RGB(200,30,30);

        *r = CDRF_DODEFAULT; return;
    }}
    *r = CDRF_DODEFAULT;
}
```

### 3-3. “진짜 그리기”로 아이콘/배지/진행률/라인 커스터마이즈
- Tree는 기본적으로 **텍스트/아이콘**을 OS가 그림
- 더 복잡한 배지를 붙이거나, **전체 행 배경**을 칠하려면 **클라이언트 영역 직도**가 필요
  - 방법 1) `CDDS_ITEMPOSTPAINT`에서 `GetItemRect(..., TRUE)`로 행 영역 구한 뒤 **오버레이**
  - 방법 2) `NM_CUSTOMDRAW`에서 `CDRF_SKIPDEFAULT`로 **완전 수동 그림** (다만 접근성/테마 호환 주의)

```cpp
// CDDS_ITEMPOSTPAINT에서 오른쪽 끝에 배지/수치 그리기
case CDDS_ITEMPOSTPAINT:
{
    CDC* pDC = CDC::FromHandle(cd->nmcd.hdc);
    CRect rText; m_tree.GetItemRect((HTREEITEM)cd->nmcd.dwItemSpec, &rText, TRUE);
    CRect badge(rText); badge.left = badge.right + 6; badge.right += 60;
    pDC->FillSolidRect(badge, RGB(250,250,200));
    pDC->DrawText(L"99+", badge, DT_SINGLELINE|DT_VCENTER|DT_CENTER);
    *r = CDRF_DODEFAULT; return;
}
```

### 3-4. 테마 API(UXTheme)와의 조합
- `OpenThemeData(hWnd, L"TreeView")` + `DrawThemeBackground`로 **현대 테마**에 맞춰 드로잉
- 필요시 `SetWindowTheme(m_hWnd, L"Explorer", nullptr)`로 익스플로러 스타일 라인/확장 단추 사용

---

## 4. 공통 고급 패턴

### 4-1. 더블 버퍼링 공통 헬퍼

```cpp
class CMemDC {
    CDC m_dc, *m_pOldDC; CBitmap m_bmp, *m_pOldBmp; CRect m_rc;
public:
    CMemDC(CDC* pDC, const CRect& rc) : m_pOldDC(pDC) {
        m_dc.CreateCompatibleDC(pDC);
        m_bmp.CreateCompatibleBitmap(pDC, rc.Width(), rc.Height());
        m_pOldBmp = m_dc.SelectObject(&m_bmp);
        m_rc = rc;
    }
    ~CMemDC() {
        m_pOldDC->BitBlt(m_rc.left, m_rc.top, m_rc.Width(), m_rc.Height(),
                         &m_dc, 0, 0, SRCCOPY);
        m_dc.SelectObject(m_pOldBmp);
    }
    CDC* GetDC() { return &m_dc; }
};
```

> 일반적으로 ListView/TreeView는 **자체 더블버퍼**가 있어 필요 없지만, **Owner-Draw 컨트롤**을 윈도우 전체 레벨로 복잡하게 그릴 때 유용.

### 4-2. 히트 테스트 & 인플레이스 편집
- ListView: `SubItemRect`로 셀 영역 얻고, 클릭 좌표로 **어느 열인지** 판별
- Tree: `TVHITTESTINFO`로 아이콘/텍스트/버튼 영역 구분
- 인플레이스 편집은 **동적 `CEdit`/`CComboBox` 생성** → 완료 시 내용 반영 + 삭제

```cpp
void CMyDlg::BeginEditCell(int row, int col) {
    CRect rc; m_list.GetSubItemRect(row, col, LVIR_BOUNDS, rc);
    if (!m_inplace.m_hWnd) m_inplace.Create(WS_CHILD|WS_BORDER|ES_AUTOHSCROLL, rc, &m_list, 1001);
    m_inplace.MoveWindow(rc); m_inplace.SetWindowText(m_list.GetItemText(row, col));
    m_inplace.ShowWindow(SW_SHOW); m_inplace.SetFocus();
    m_editRow=row; m_editCol=col;
}
```

### 4-3. 상태 이미지/오버레이 이미지
- 이미지 리스트에 **상태 이미지**를 등록 후 `LVIS_STATEIMAGEMASK`를 사용하면 체크/경고/오프라인 등 표현 용이
- Tree도 `TVS_CHECKBOXES`(레지스터 스타일)나 상태 이미지 사용 가능

```cpp
UINT StateImageMask(int idx1Based) { return INDEXTOSTATEIMAGEMASK(idx1Based); } // 1=unchecked, 2=checked
m_list.SetItemState(row, StateImageMask(2), LVIS_STATEIMAGEMASK);
```

### 4-4. 다크 테마/하이 콘트라스트 대응
- **색 하드코딩 최소화**, 시스템 색/테마 질의 → `GetSysColor`, UxTheme 색상
- 하이 콘트라스트 모드(`SystemParametersInfo(SPI_GETHIGHCONTRAST, ...)`) 감지 후 **기본 그리기 우선**

---

## 5. 통합 예제 ①: `CListCtrlEx` — 지브라/호버/체크/진행률/가상 모드

```cpp
class CListCtrlEx : public CListCtrl {
    DECLARE_MESSAGE_MAP()
public:
    BOOL Init() {
        ModifyStyle(0, LVS_REPORT);
        SetExtendedStyle(GetExtendedStyle()|LVS_EX_FULLROWSELECT|LVS_EX_DOUBLEBUFFER);
        InsertColumn(0, L"이름", LVCFMT_LEFT, 200);
        InsertColumn(1, L"크기", LVCFMT_RIGHT, 80);
        InsertColumn(2, L"진행", LVCFMT_LEFT, 160);
        return TRUE;
    }

    afx_msg void OnCustomDraw(NMHDR* n, LRESULT* r) {
        LPNMLVCUSTOMDRAW cd = (LPNMLVCUSTOMDRAW)n;
        switch (cd->nmcd.dwDrawStage) {
        case CDDS_PREPAINT: *r = CDRF_NOTIFYITEMDRAW; return;
        case CDDS_ITEMPREPAINT: {
            int row = (int)cd->nmcd.dwItemSpec;
            bool sel = (cd->nmcd.uItemState & CDIS_SELECTED)!=0;
            bool hot = (cd->nmcd.uItemState & CDIS_HOT)!=0;
            cd->clrText = sel?RGB(255,255,255):RGB(34,34,34);
            cd->clrTextBk = sel?RGB(51,102,204):(row%2?RGB(255,255,255):RGB(248,250,253));
            if (!sel && hot) cd->clrTextBk = RGB(235,244,255);
            *r = CDRF_NOTIFYSUBITEMDRAW; return; }
        case CDDS_SUBITEM|CDDS_ITEMPREPAINT: {
            int row = (int)cd->nmcd.dwItemSpec;
            int col = cd->iSubItem;
            if (col==2) {
                *r = CDRF_SKIPDEFAULT;
                CDC* dc = CDC::FromHandle(cd->nmcd.hdc);
                CRect rc; GetSubItemRect(row, col, LVIR_BOUNDS, rc);
                int p = GetProgress(row); // 0~100
                DrawProgress(*dc, rc, p);
                return;
            }
            *r = CDRF_DODEFAULT; return; }
        }
        *r = CDRF_DODEFAULT;
    }

    static void DrawProgress(CDC& dc, const CRect& cell, int pct) {
        CRect bar = cell; bar.DeflateRect(6, 6);
        dc.FillSolidRect(bar, RGB(240,240,240));
        dc.Draw3dRect(bar, RGB(210,210,210), RGB(210,210,210));
        CRect fill = bar; fill.right = bar.left + MulDiv(bar.Width(), pct, 100);
        fill.DeflateRect(1,1);
        dc.FillSolidRect(fill, RGB(76,175,80));
        CString s; s.Format(L"%d%%", pct);
        dc.SetBkMode(TRANSPARENT);
        dc.DrawText(s, bar, DT_CENTER|DT_VCENTER|DT_SINGLELINE);
    }

    int GetProgress(int row) {
        // 데모: 행 번호 기반 가짜 값
        return (row*13)%101;
    }
};

BEGIN_MESSAGE_MAP(CListCtrlEx, CListCtrl)
    ON_NOTIFY_REFLECT(NM_CUSTOMDRAW, &CListCtrlEx::OnCustomDraw)
END_MESSAGE_MAP()
```

> **사용**: 대화상자에서 `m_listEx.SubclassDlgItem(IDC_LIST, this); m_listEx.Init();`
> 가상 모드가 필요하면 `ModifyStyle(0, LVS_OWNERDATA)` + `SetItemCountEx` + `LVN_GETDISPINFO` 구현.

---

## 6. 통합 예제 ②: `CTreeCtrlEx` — 풀로우 선택/호버/배지

```cpp
class CTreeCtrlEx : public CTreeCtrl {
    DECLARE_MESSAGE_MAP()
public:
    BOOL Init() {
        ModifyStyle(0, TVS_FULLROWSELECT|TVS_TRACKSELECT);
        return TRUE;
    }

    afx_msg void OnCustomDraw(NMHDR* n, LRESULT* r) {
        auto cd = (NMTVCUSTOMDRAW*)n;
        switch (cd->nmcd.dwDrawStage) {
        case CDDS_PREPAINT: *r = CDRF_NOTIFYITEMDRAW; return;
        case CDDS_ITEMPREPAINT: {
            bool sel = (cd->nmcd.uItemState & CDIS_SELECTED)!=0;
            bool hot = (cd->nmcd.uItemState & CDIS_HOT)!=0;
            cd->clrText   = sel?RGB(255,255,255):RGB(34,34,34);
            cd->clrTextBk = sel?RGB(51,102,204):(hot?RGB(235,244,255):RGB(255,255,255));
            *r = CDRF_DODEFAULT; return; }
        }
        *r = CDRF_DODEFAULT;
    }

    // 배지 그리기(ITEMPOSTPAINT) 등을 추가하고 싶다면:
    // ON_NOTIFY_REFLECT(NM_CUSTOMDRAW, ...)에서 CDDS_ITEMPOSTPAINT 분기 후
    // GetItemRect(hItem, &rect, TRUE)로 텍스트 영역 얻어 우측에 오버레이
};

BEGIN_MESSAGE_MAP(CTreeCtrlEx, CTreeCtrl)
    ON_NOTIFY_REFLECT(NM_CUSTOMDRAW, &CTreeCtrlEx::OnCustomDraw)
END_MESSAGE_MAP()
```

---

## 7. 성능 & 안정성 & UX 체크리스트

1. **깜빡임**: `LVS_EX_DOUBLEBUFFER`, `TVS_EX_DOUBLEBUFFER`(가능 시), 필요 시 수동 더블버퍼
2. **GDI 리소스 누수**: `CPen/ CBrush/ CFont` 생성/선택/해제 철저
3. **가상 리스트**: 텍스트/색상 계산은 **O(1)**, I/O 금지(캐시)
4. **히트 테스트**: 셀/노드 별 경계 정확하게, 패딩은 DPI 기준 계산
5. **정렬/필터**: 모델 인덱스 ↔ 뷰 인덱스 매핑 일관성 유지
6. **접근성**: 선택 색/포커스 표시 유지, 하이 콘트라스트 모드 고려
7. **키보드 내비게이션**: ↑↓/Space/Enter 등 기본 동작 훼손 금지
8. **DPI/테마**: `WM_DPICHANGED` 대응 시 열 폭/아이콘/패딩 재계산, 테마 API 사용 시 핸들 수명 관리
9. **마우스 캡처/드래그**: 인플레이스 편집/드래그 수행 시 메시지 라우팅(포커스) 주의
10. **테스트**: 1e5 행 가상 리스트 스크롤, 빠른 정렬/필터 변경, 다중 선택 후 배치 갱신

---

## 8. 문제 해결 가이드

| 증상 | 원인 | 해결 |
|---|---|---|
| 선택 배경이 안 보임 | `CDRF_SKIPDEFAULT`로 기본 드로잉 누락 | 선택/포커스 렌더를 **직접** 구현하거나 `CDRF_DODEFAULT` 사용 |
| 텍스트 잘림/어긋남 | 서브아이템 Rect/패딩 계산 오차 | `GetSubItemRect(LVIR_LABEL/BOUNDS)` 비교, DPI 반영 |
| CPU 사용량 높음 | `NM_CUSTOMDRAW`에서 무거운 연산 | 색/문자열 캐시, GDI 객체 재사용 |
| 가상 목록에서 깜빡임 | 빈번한 `RedrawItems`/정렬 | 더블버퍼 + 배치 업데이트(`SetRedraw(FALSE/TRUE)`) |
| 트리 전체행 배경 안 칠해짐 | `TVS_FULLROWSELECT` 미사용 | 스타일 추가 또는 클라이언트 직도(POSTPAINT) |

---

## 9. 빠른 스타터(요약 스니펫)

```cpp
// 메시지 맵
BEGIN_MESSAGE_MAP(CMyDlg, CDialogEx)
    ON_NOTIFY(NM_CUSTOMDRAW, IDC_LIST, &CMyDlg::OnListCustomDraw)
    ON_NOTIFY(NM_CLICK,       IDC_LIST, &CMyDlg::OnListClick)
    ON_NOTIFY(NM_CUSTOMDRAW,  IDC_TREE, &CMyDlg::OnTreeCustomDraw)
    ON_NOTIFY(LVN_GETDISPINFO,IDC_LIST, &CMyDlg::OnGetDispInfo) // 가상 리스트
END_MESSAGE_MAP()
```

---

## 10. 결론

- **Owner-Draw**(ListBox/Combo)는 **완전 커스텀**이 필요한 단순 목록/선택 UI에 적합.
- **Custom-Draw**(ListView/TreeView)는 **서브아이템/상태/가상/성능**까지 잡는 **현실적 솔루션**.
- 핵심은 **단계별 훅의 의미**와 **기본 드로잉과의 경계**를 정확히 이해하는 것.
- 본문 스니펫(지브라/호버/체크/진행/배지/인플레이스)을 취사 선택해 붙이면, **현대적이고 빠른** 리스트/트리를 만들 수 있습니다.
