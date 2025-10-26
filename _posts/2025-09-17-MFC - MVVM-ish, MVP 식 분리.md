---
layout: post
title: MFC - MVVM-ish, MVP 식 분리
date: 2025-09-17 14:25:23 +0900
category: MFC
---
# MVVM-ish / MVP 식 분리: **MFC 앱을 테스트 가능한 설계로 가볍게 전환**하기

> “코드를 전부 갈아엎지 않고, **조금씩** 역할 분리를 시작해도 충분합니다.”  
> 이 글은 **MFC/Win32** 기반 앱에서 **MVVM-ish / MVP** 스타일로 가볍게 전환하는 **실전 로드맵**입니다.  
> 핵심은 **View(창/컨트롤)** 와 **로직(뷰모델/프레젠터)** 사이에 **인터페이스/바인딩**을 두고,  
> **단위테스트 가능한 구조**로 바꾸는 것입니다.

---

## 0) 왜 MVVM-ish/MVP인가? (문제 → 목표)

### 기존 문제
- `CDialog::OnBnClickedOk` 안에 **검증/계산/파일 I/O**가 뒤섞여 테스트가 어렵고 유지보수도 고통.
- **뷰 로직+도메인 로직**이 한 파일에 섞여 **재사용/모킹/병렬 개발**이 힘듦.
- UI 이벤트(버튼 클릭/리스트 변경)에 따라 **상태 업데이트**가 산발적으로 퍼짐.

### 목표
- **View**(MFC 창/컨트롤) 는 **바인딩/입출력 위임만**.  
- **ViewModel/Presenter** 는 **상태·명령·검증** 담당, **UI 프레임워크 독립**(테스트 용이).  
- **Model/Service** 는 파일/네트워크/DB 등 **외부 연동** 추상화.

> 우리는 **MVVM 100% 구현**이 아니라, **MVP+MVVM의 실용 혼합(MVVM-ish)** 으로 **작게 시작**합니다.

---

## 1) 용어/역할 요약

| 레이어 | 책임 | 의존 |
|---|---|---|
| **View** (MFC `CDialog`/`CView`/컨트롤) | 화면, 이벤트 발생, 바인딩 호출 | **ViewModel 인터페이스** |
| **ViewModel / Presenter** | 상태 보유, 검증, Command 실행, 비동기/취소 | **Model/Service 인터페이스**, (옵션) 메시지버스 |
| **Model/Service** | 데이터(파일/네트워크/DB), 비즈니스 규칙 | 없음(하위 의존) |

핵심 포인트
- **View는 얇게**, `GetDlgItemText/SetDlgItemText` 같은 **순수 I/O**만.
- **ViewModel/Presenter는 UI 프레임워크에 독립** (테스트/재사용 가능).
- **서비스는 인터페이스**로 숨기고 DI(주입)로 교체 가능.

---

## 2) 최소 골격: 인터페이스 → 바인딩 → 테스트

### 2-1. ViewModel 인터페이스(상태 + 명령)
```cpp
// ISettingsVM.h : ViewModel 인터페이스
#pragma once
#include <string>
#include <functional>
#include <optional>

struct ISettingsVM {
    virtual ~ISettingsVM() = default;

    // 상태(속성)
    virtual std::wstring userName() const = 0;
    virtual void setUserName(std::wstring v) = 0;

    virtual int refreshIntervalSec() const = 0;
    virtual void setRefreshIntervalSec(int v) = 0;

    virtual bool darkMode() const = 0;
    virtual void setDarkMode(bool v) = 0;

    // 유효성/오류 표시
    virtual std::optional<std::wstring> lastError() const = 0;

    // Command
    virtual bool canSave() const = 0;
    virtual void save() = 0;
    virtual void cancel() = 0;

    // 속성 변경 통지(옵저버)
    using Changed = std::function<void(std::wstring prop)>;
    virtual void subscribe(Changed cb) = 0;
};
```

### 2-2. 서비스 인터페이스
```cpp
// ISettingsService.h
#pragma once
#include <string>
#include <optional>

struct SettingsData {
    std::wstring userName;
    int intervalSec = 60;
    bool dark = false;
};

struct ISettingsService {
    virtual ~ISettingsService() = default;
    virtual std::optional<SettingsData> load() = 0;
    virtual bool save(const SettingsData& s, std::wstring& outError) = 0;
};
```

### 2-3. ViewModel 구현(프레임워크 비의존)
```cpp
// SettingsVM.h/.cpp
#pragma once
#include "ISettingsVM.h"
#include "ISettingsService.h"
#include <vector>

class SettingsVM : public ISettingsVM {
public:
    explicit SettingsVM(ISettingsService* svc) : m_svc(svc) {
        if (auto d = m_svc->load()) {
            m_user = d->userName;
            m_interval = d->intervalSec;
            m_dark = d->dark;
        }
    }
    // 상태
    std::wstring userName() const override { return m_user; }
    void setUserName(std::wstring v) override { if (m_user != v){ m_user = std::move(v); notify(L"userName"); }}
    int refreshIntervalSec() const override { return m_interval; }
    void setRefreshIntervalSec(int v) override { if (m_interval != v){ m_interval = v; notify(L"refreshIntervalSec"); }}
    bool darkMode() const override { return m_dark; }
    void setDarkMode(bool v) override { if (m_dark != v){ m_dark = v; notify(L"darkMode"); }}

    // 오류/검증
    std::optional<std::wstring> lastError() const override { return m_error; }

    // Command
    bool canSave() const override {
        return !m_user.empty() && m_interval >= 5; // 예시: 최소 5초
    }
    void save() override {
        if (!canSave()) { setError(L"이름 또는 간격이 유효하지 않습니다."); return; }
        SettingsData d{ m_user, m_interval, m_dark };
        std::wstring err;
        if (!m_svc->save(d, err)) { setError(err); return; }
        setError(std::nullopt);
        notify(L"saved"); // 임의의 신호
    }
    void cancel() override {
        if (auto d = m_svc->load()) {
            m_user = d->userName; m_interval = d->intervalSec; m_dark = d->dark;
            setError(std::nullopt);
            notifyAll();
        }
    }

    void subscribe(Changed cb) override { m_subs.push_back(std::move(cb)); }

private:
    ISettingsService* m_svc;
    std::wstring m_user;
    int m_interval = 60;
    bool m_dark = false;
    std::optional<std::wstring> m_error;
    std::vector<Changed> m_subs;

    void setError(std::optional<std::wstring> e){ m_error = std::move(e); notify(L"lastError"); }
    void notify(const std::wstring& prop){ for (auto& f : m_subs) f(prop); }
    void notifyAll(){ notify(L"userName"); notify(L"refreshIntervalSec"); notify(L"darkMode"); notify(L"lastError"); }
};
```

### 2-4. 가짜 서비스(테스트/프로토타입)
```cpp
// FakeSettingsService.h
#pragma once
#include "ISettingsService.h"

class FakeSettingsService : public ISettingsService {
public:
    std::optional<SettingsData> load() override { return data; }
    bool save(const SettingsData& s, std::wstring& outError) override {
        // 간단 검증
        if (s.userName.size() > 64) { outError = L"이름이 너무 깁니다."; return false; }
        data = s; return true;
    }
private:
    SettingsData data{ L"Guest", 60, false };
};
```

### 2-5. View(MFC Dialog) — 바인딩만
```cpp
// SettingsDlg.h
class CSettingsDlg : public CDialogEx {
public:
    CSettingsDlg(ISettingsVM* vm) : CDialogEx(IDD_SETTINGS), m_vm(vm) {}
protected:
    ISettingsVM* m_vm;
    CEdit   m_editName;
    CSpinButtonCtrl m_spinInterval;
    CButton m_chkDark, m_btnSave;

    virtual BOOL OnInitDialog() {
        CDialogEx::OnInitDialog();
        m_editName.SubclassDlgItem(IDC_EDIT_NAME, this);
        m_spinInterval.SubclassDlgItem(IDC_SPIN_INTERVAL, this);
        m_chkDark.SubclassDlgItem(IDC_CHK_DARK, this);
        m_btnSave.SubclassDlgItem(IDOK, this);

        // 초기 값 → UI
        SetDlgItemTextW(IDC_EDIT_NAME, m_vm->userName().c_str());
        SetDlgItemInt(IDC_EDIT_INTERVAL, m_vm->refreshIntervalSec());
        m_chkDark.SetCheck(m_vm->darkMode());

        // VM 변경 → UI 반영
        m_vm->subscribe([this](std::wstring prop){ PostMessageW(WM_APP+1, 0, 0); });

        return TRUE;
    }

    afx_msg LRESULT OnVmChanged(WPARAM, LPARAM) {
        // 간단: 전체 리바인딩 (최적화는 후술)
        SetDlgItemTextW(IDC_EDIT_NAME, m_vm->userName().c_str());
        SetDlgItemInt(IDC_EDIT_INTERVAL, m_vm->refreshIntervalSec());
        m_chkDark.SetCheck(m_vm->darkMode());
        // Save 버튼 상태
        GetDlgItem(IDOK)->EnableWindow(m_vm->canSave());
        // 오류 메시지
        if (auto e = m_vm->lastError()) SetDlgItemTextW(IDC_STATIC_ERROR, e->c_str());
        else SetDlgItemTextW(IDC_STATIC_ERROR, L"");
        return 0;
    }

    // UI → VM
    afx_msg void OnEnChangeName() {
        CString s; m_editName.GetWindowTextW(s);
        m_vm->setUserName((LPCWSTR)s);
    }
    afx_msg void OnEnChangeInterval() {
        BOOL ok = FALSE;
        int n = GetDlgItemInt(IDC_EDIT_INTERVAL, &ok);
        if (ok) m_vm->setRefreshIntervalSec(n);
    }
    afx_msg void OnBnClickedDark() {
        m_vm->setDarkMode(m_chkDark.GetCheck() == BST_CHECKED);
    }
    afx_msg void OnBnClickedSave() {
        m_vm->save(); // 성공 시 'saved' 통지로 버튼/메시지 갱신
    }
    afx_msg void OnBnClickedCancel() {
        m_vm->cancel();
    }

    // Idle 에서 Save 버튼 활성화 갱신(대화상자는 자동 Update가 없으므로)
    virtual BOOL PreTranslateMessage(MSG* pMsg) {
        UpdateDialogControls(this, FALSE); // ON_UPDATE_COMMAND_UI 연동 시
        return CDialogEx::PreTranslateMessage(pMsg);
    }
    afx_msg void OnUpdateSave(CCmdUI* p) { p->Enable(m_vm->canSave()); }

    DECLARE_MESSAGE_MAP()
};

// SettingsDlg.cpp
BEGIN_MESSAGE_MAP(CSettingsDlg, CDialogEx)
    ON_MESSAGE(WM_APP+1, &CSettingsDlg::OnVmChanged)
    ON_EN_CHANGE(IDC_EDIT_NAME, &CSettingsDlg::OnEnChangeName)
    ON_EN_CHANGE(IDC_EDIT_INTERVAL, &CSettingsDlg::OnEnChangeInterval)
    ON_BN_CLICKED(IDC_CHK_DARK, &CSettingsDlg::OnBnClickedDark)
    ON_BN_CLICKED(IDOK, &CSettingsDlg::OnBnClickedSave)
    ON_BN_CLICKED(IDCANCEL, &CSettingsDlg::OnBnClickedCancel)
    ON_UPDATE_COMMAND_UI(IDOK, &CSettingsDlg::OnUpdateSave)
END_MESSAGE_MAP()
```

> **핵심**: 대화상자는 기본적으로 `ON_UPDATE_COMMAND_UI`가 자동 호출되지 않으므로  
> `PreTranslateMessage`에서 `UpdateDialogControls`를 **명시 호출**하거나, 상태 변화 시 **PostMessage → 한 번 갱신**.

---

## 3) 얇은 바인더로 “양방향” 베이스 마련 (확장 가능)

실무에서 **컨트롤마다 수동 핸들러**를 쓰면 보일러플레이트가 늘어납니다.  
아주 **얇은 바인더** 유틸로 반복을 줄일 수 있습니다.

```cpp
// Binder.h : 아주 작은 양방향 바인더
#pragma once
#include <functional>
#include <unordered_map>
#include <string>

class Binder {
public:
    // View->VM 바인딩(컨트롤 변경 시 호출할 람다)
    void bindInput(UINT id, std::function<void()> f) { m_inputs[id] = std::move(f); }
    // VM->View 갱신(속성명→한 번에 처리할 람다)
    void bindOutput(std::wstring prop, std::function<void()> f) { m_outputs[std::move(prop)] = std::move(f); }

    void onControlChanged(UINT id){ if (auto it=m_inputs.find(id); it!=m_inputs.end()) it->second(); }
    void onVmChanged(const std::wstring& prop){ if (auto it=m_outputs.find(prop); it!=m_outputs.end()) it->second(); }
    void refreshAll(){ for (auto& kv : m_outputs) kv.second(); }

private:
    std::unordered_map<UINT, std::function<void()>> m_inputs;
    std::unordered_map<std::wstring, std::function<void()>> m_outputs;
};
```

사용 예:
```cpp
// OnInitDialog에서
m_binder.bindInput(IDC_EDIT_NAME, [this]{
    CString s; m_editName.GetWindowTextW(s); m_vm->setUserName((LPCWSTR)s);
});
m_binder.bindOutput(L"userName", [this]{
    SetDlgItemTextW(IDC_EDIT_NAME, m_vm->userName().c_str());
});
m_vm->subscribe([this](std::wstring p){ m_binder.onVmChanged(p); });
```

> 필요 시, **유효성 표시/에러 텍스트/Enable/Visible** 등으로 람다를 확장.

---

## 4) MVP 변형: Presenter가 View 인터페이스를 호출

MVVM은 ViewModel이 **View에 대해 아무것도 몰라야** 합니다.  
MVP는 Presenter가 **IView 인터페이스**를 통해 **명시적으로 View를 제어**합니다.  
둘 중 **팀 취향/프로젝트 상황**에 맞는 쪽을 선택하세요.

### 4-1. View 인터페이스
```cpp
// ISettingsView.h
struct ISettingsView {
    virtual ~ISettingsView() = default;
    virtual void setUserNameText(const std::wstring& s) = 0;
    virtual void setIntervalText(int n) = 0;
    virtual void setDarkChecked(bool on) = 0;
    virtual void setErrorText(const std::wstring& s) = 0;
    virtual void setSaveEnabled(bool on) = 0;
};
```

### 4-2. Presenter
```cpp
class SettingsPresenter {
public:
    SettingsPresenter(ISettingsView* v, ISettingsService* s) : view(v), svc(s) {
        reload();
        updateSaveEnabled();
    }
    // View → Presenter
    void onUserNameChanged(std::wstring v){ user=v; updateSaveEnabled(); }
    void onIntervalChanged(int n){ interval=n; updateSaveEnabled(); }
    void onDarkChanged(bool on){ dark=on; }
    void onSave(){ doSave(); }
    void onCancel(){ reload(); }

private:
    ISettingsView* view;
    ISettingsService* svc;
    std::wstring user; int interval=60; bool dark=false;

    void reload() {
        if (auto d=svc->load()) {
            user=d->userName; interval=d->intervalSec; dark=d->dark;
            view->setUserNameText(user); view->setIntervalText(interval); view->setDarkChecked(dark);
            view->setErrorText(L"");
        }
    }
    void updateSaveEnabled(){ view->setSaveEnabled(!user.empty() && interval>=5); }
    void doSave(){
        SettingsData d{user,interval,dark}; std::wstring err;
        if (!svc->save(d, err)) { view->setErrorText(err); return; }
        view->setErrorText(L"저장되었습니다.");
    }
};
```

### 4-3. MFC View 구현
```cpp
class CSettingsDlg2 : public CDialogEx, public ISettingsView {
public:
    CSettingsDlg2(ISettingsService* svc) : CDialogEx(IDD_SETTINGS), pSvc(svc) {}
protected:
    SettingsPresenter* p{};
    ISettingsService* pSvc;

    BOOL OnInitDialog() override {
        CDialogEx::OnInitDialog();
        p = new SettingsPresenter(this, pSvc);
        return TRUE;
    }
    void PostNcDestroy() override { delete p; CDialogEx::PostNcDestroy(); }

    // ISettingsView 구현
    void setUserNameText(const std::wstring& s) override { SetDlgItemTextW(IDC_EDIT_NAME, s.c_str()); }
    void setIntervalText(int n) override { SetDlgItemInt(IDC_EDIT_INTERVAL, n); }
    void setDarkChecked(bool on) override { CheckDlgButton(IDC_CHK_DARK, on ? BST_CHECKED : BST_UNCHECKED); }
    void setErrorText(const std::wstring& s) override { SetDlgItemTextW(IDC_STATIC_ERROR, s.c_str()); }
    void setSaveEnabled(bool on) override { GetDlgItem(IDOK)->EnableWindow(on); }

    // UI → Presenter
    afx_msg void OnNameChange(){ CString s; GetDlgItemTextW(IDC_EDIT_NAME, s); p->onUserNameChanged((LPCWSTR)s); }
    afx_msg void OnIntervalChange(){ BOOL ok; int n=GetDlgItemInt(IDC_EDIT_INTERVAL,&ok); if(ok) p->onIntervalChanged(n); }
    afx_msg void OnDarkClick(){ p->onDarkChanged(IsDlgButtonChecked(IDC_CHK_DARK)==BST_CHECKED); }
    afx_msg void OnSave(){ p->onSave(); }
    afx_msg void OnCancelClick(){ p->onCancel(); }

    DECLARE_MESSAGE_MAP()
};

BEGIN_MESSAGE_MAP(CSettingsDlg2, CDialogEx)
    ON_EN_CHANGE(IDC_EDIT_NAME, &CSettingsDlg2::OnNameChange)
    ON_EN_CHANGE(IDC_EDIT_INTERVAL, &CSettingsDlg2::OnIntervalChange)
    ON_BN_CLICKED(IDC_CHK_DARK, &CSettingsDlg2::OnDarkClick)
    ON_BN_CLICKED(IDOK, &CSettingsDlg2::OnSave)
    ON_BN_CLICKED(IDCANCEL, &CSettingsDlg2::OnCancelClick)
END_MESSAGE_MAP()
```

> **MVP 장점**: “Presenter → View” 의존은 있지만 **테스트할 때 View를 모킹**하기 쉽습니다.

---

## 5) 리스트/그리드: **Table Model**로 데이터-뷰 분리

### 5-1. 모델 인터페이스
```cpp
struct ITableModel {
    virtual ~ITableModel() = default;
    virtual int rowCount() const = 0;
    virtual int colCount() const = 0;
    virtual std::wstring cellText(int r, int c) const = 0;
    virtual void sortBy(int c, bool asc) = 0;
};
```

### 5-2. 구현 예 (파일 목록)
```cpp
class FileTableModel : public ITableModel {
public:
    struct Row { std::wstring name; uintmax_t size; std::wstring date; };
    std::vector<Row> rows;
    int rowCount() const override { return (int)rows.size(); }
    int colCount() const override { return 3; }
    std::wstring cellText(int r, int c) const override {
        auto& x = rows[r];
        switch(c){ case 0: return x.name; case 1: return std::to_wstring(x.size); case 2: return x.date; }
        return L"";
    }
    void sortBy(int c, bool asc) override {
        std::sort(rows.begin(), rows.end(), [&](auto& a, auto& b){
            auto key = [&](const Row& z)->std::wstring {
                if (c==0) return z.name;
                if (c==1) return std::to_wstring(z.size);
                return z.date;
            };
            return asc ? key(a)<key(b) : key(b)<key(a);
        });
    }
};
```

### 5-3. View( `CListCtrl` ) 어댑터
```cpp
class ListCtrlAdapter {
public:
    ListCtrlAdapter(CListCtrl& lc, ITableModel* m) : list(lc), model(m) {}
    void init() {
        list.SetExtendedStyle(LVS_EX_FULLROWSELECT|LVS_EX_DOUBLEBUFFER|LVS_EX_GRIDLINES);
        list.InsertColumn(0, L"Name", LVCFMT_LEFT, 240);
        list.InsertColumn(1, L"Size", LVCFMT_RIGHT, 100);
        list.InsertColumn(2, L"Date", LVCFMT_LEFT, 140);
        refresh();
    }
    void refresh() {
        list.SetRedraw(FALSE);
        list.DeleteAllItems();
        for (int r=0;r<model->rowCount();++r) {
            list.InsertItem(r, model->cellText(r,0).c_str());
            for (int c=1;c<model->colCount();++c) list.SetItemText(r, c, model->cellText(r,c).c_str());
        }
        list.SetRedraw(TRUE);
    }
    void sort(int c, bool asc){ model->sortBy(c,asc); refresh(); }
private:
    CListCtrl& list; ITableModel* model;
};
```

> 이 패턴을 쓰면, **모델은 테스트 가능**, **View는 반복적인 목록 바인딩만** 수행합니다.

---

## 6) 명령 패턴/ON_UPDATE_COMMAND_UI와 궁합

**명령(Command) ↔ ViewModel 상태**를 매칭하면 **테스트 가능한 버튼/메뉴**를 만들 수 있습니다.  
(이 섹션은 이전 “명령 패턴 + ON_UPDATE_COMMAND_UI” 글과 자연스럽게 맞물립니다.)

```cpp
// ViewModel에 'canSave()'가 있고, View는 Update 핸들러에서 pUI->Enable(vm->canSave());
ON_UPDATE_COMMAND_UI(ID_SAVE, &CMainFrame::OnUpdateSave)
void CMainFrame::OnUpdateSave(CCmdUI* p){ p->Enable(m_vm->canSave()); }
ON_COMMAND(ID_SAVE, &CMainFrame::OnSave)
void CMainFrame::OnSave(){ m_vm->save(); }
```

> 핵심: **조건 로직은 VM의 `canXxx()`로 통일**, View는 그 결과만 **반영**.

---

## 7) 비동기/취소/타임아웃: ViewModel에서 제어, View는 표시만

### 7-1. 작업 스케줄러 추상화
```cpp
struct IScheduler {
    virtual ~IScheduler() = default;
    virtual void post(std::function<void()> f) = 0; // UI 스레드
    virtual void runAsync(std::function<void(std::atomic_bool& cancel)> f) = 0; // 작업 스레드
    virtual void cancelAll() = 0;
};
```

### 7-2. ViewModel에서 사용
```cpp
class ExportVM {
public:
    ExportVM(IScheduler* s, ISettingsService* svc) : sched(s), service(svc) {}
    bool isBusy() const { return busy; }
    void startExport(){
        if (busy) return;
        busy=true; notify(L"isBusy");
        sched->runAsync([this](std::atomic_bool& cancel){
            // 긴 작업: cancel 주기 확인
            for(int i=0;i<100 && !cancel.load();++i){ /* chunk */ std::this_thread::sleep_for(std::chrono::milliseconds(20)); }
            sched->post([this]{ busy=false; notify(L"isBusy"); notify(L"done"); });
        });
    }
    void cancel(){ sched->cancelAll(); }

    using Changed = std::function<void(std::wstring)>;
    void subscribe(Changed f){ subs.push_back(std::move(f)); }
private:
    void notify(const std::wstring& p){ for(auto& f:subs) f(p); }
    IScheduler* sched; ISettingsService* service; bool busy=false; std::vector<Changed> subs;
};
```

### 7-3. View는 바인딩만
```cpp
ON_UPDATE_COMMAND_UI(ID_EXPORT, &CMainFrame::OnUpdateExport)
void CMainFrame::OnUpdateExport(CCmdUI* p){ p->Enable(!m_exportVm->isBusy()); }
ON_COMMAND(ID_EXPORT, &CMainFrame::OnExport){ m_exportVm->startExport(); }
ON_COMMAND(ID_CANCEL, &CMainFrame::OnCancelExport){ m_exportVm->cancel(); }
```

---

## 8) 테스트(googletest 예시): View 없이 ViewModel만 검증

```cpp
#include <gtest/gtest.h>
#include "SettingsVM.h"
#include "FakeSettingsService.h"

TEST(SettingsVM, SaveValidation) {
    FakeSettingsService svc;
    SettingsVM vm(&svc);

    vm.setUserName(L"");           // invalid
    vm.setRefreshIntervalSec(2);   // invalid
    EXPECT_FALSE(vm.canSave());

    vm.setUserName(L"Alice");
    vm.setRefreshIntervalSec(10);
    EXPECT_TRUE(vm.canSave());

    vm.save();
    EXPECT_FALSE(vm.lastError().has_value());
}

TEST(SettingsVM, CancelReloadsData) {
    FakeSettingsService svc;
    SettingsVM vm(&svc);
    vm.setUserName(L"Bob");
    vm.cancel(); // reload "Guest"
    EXPECT_EQ(vm.userName(), L"Guest");
}
```

> UI 없는 **순수 C++ 테스트**만으로 핵심 로직을 검증할 수 있습니다.

---

## 9) DI(의존성 주입)와 컴포지션 루트

**앱 초기화 시** 실제 구현을 조립합니다.

```cpp
// Composition.cpp (앱 시작 지점)
std::unique_ptr<ISettingsService> g_settingsSvc;
std::unique_ptr<ISettingsVM> g_settingsVm;

BOOL CMyApp::InitInstance() {
    CWinApp::InitInstance();

    g_settingsSvc = std::make_unique<FileSettingsService>(/* path */);
    g_settingsVm  = std::make_unique<SettingsVM>(g_settingsSvc.get());

    CSettingsDlg dlg(g_settingsVm.get());
    m_pMainWnd = &dlg;
    dlg.DoModal();
    return FALSE;
}
```

> 테스트/개발/운영에서 **서비스/스케줄러**를 **교체** 가능:  
> `FakeSettingsService` ↔ `FileSettingsService` / `HttpSettingsService` 등.

---

## 10) 기존 코드(레거시) **단계적 치환** 전략

1. **가장 자주 쓰는 대화상자 한 개**를 선택해 VM/P 모델로 변환.  
2. 컨트롤 1~2개에만 **바인딩**을 적용해 ROI를 확인.  
3. **유효성 검증/저장** 로직을 VM으로 이관.  
4. **서비스 인터페이스**로 파일/네트워크 I/O 분리.  
5. **테스트 추가** → 회귀 안정성 확보.  
6. 점차 **목록/정렬/검색** 등으로 VM 확장.  
7. **비동기 작업**을 스케줄러 추상화로 안전 이관.  
8. 문서/뷰 구조(SDI/MDI)에도 **VM/Presenter**를 적용.

---

## 11) 고급 토픽(필요 시 도입)

- **Validation Layer**: VM 속성을 업데이트할 때 규칙을 순차 적용, 에러 사유 수집.  
- **Undo/Redo**: VM에 **Command 스택**(Insert/Delete/PropertyChange)으로 도입.  
- **메시지 버스(Event Aggregator)**: VM 간 loosely coupled 통신.  
- **국제화(i18n)**: VM은 **키**만 내고 View/리소스에서 텍스트 변환.  
- **테마/다크 모드**: VM에 `theme()` 속성, View는 테마 변경 시 색/이미지 교체.  
- **상태 영속화**: VM 상태를 `settings.json`에 저장/로드.

---

## 12) 체크리스트 (요약)

- [ ] **View**는 바인딩과 이벤트 포워딩만; **로직 금지**  
- [ ] **ViewModel/Presenter**는 UI 프레임워크 **독립**  
- [ ] **서비스/모델**은 인터페이스로 추상화, DI 구성  
- [ ] 변경 통지는 **subscribe/notify**로 일원화  
- [ ] `ON_UPDATE_COMMAND_UI`는 **VM의 canXxx()** 기반  
- [ ] 비동기/취소는 **스케줄러 추상화** 사용  
- [ ] **단위테스트**는 VM/서비스에 집중  
- [ ] 레거시는 **점진적으로** 치환 (작은 성공을 반복)  
- [ ] 바인더 유틸로 **보일러플레이트 감소**  
- [ ] 국제화/테마/상태 저장은 VM 레벨에서 전략화

---

## 13) 마무리

**MVVM-ish/MVP** 전환의 핵심은 **역할 분리**와 **테스트 가능성**입니다.  
MFC에서조차 View를 **얇은 바인더**로 만들고, 핵심 로직은 **ViewModel/Presenter**로 옮기면:

- UI 교체(리본/다크 모드/새 스킨)에도 **로직 재사용**  
- 파일/네트워크/DB 교체에도 **서비스 주입**으로 대응  
- 대형 기능 추가에도 **리스크/회귀 감소**
