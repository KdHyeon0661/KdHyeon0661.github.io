---
layout: post
title: MFC - Thread 역할 분리
date: 2025-09-06 15:25:23 +0900
category: MFC
---
# CWinThread와 UI/Worker Thread 역할 분리 (MFC 멀티스레딩 완전 가이드 + 예제 다수)

이 글은 **MFC 응용프로그램**에서 스레드를 올바르게 설계·구현하는 방법을
**`CWinThread`**, **UI 스레드 vs Worker 스레드**, **메시지 교환**, **동기화**, **취소/타임아웃**, **COM 초기화**, **진행률/취소 UI**, **작업 큐/스레드 풀**까지 **생략 없이** 정리합니다.
모든 코드는 ``` 로 감싸며, 그대로 붙여서 실험할 수 있는 **작은 단위 예제**로 구성되어 있습니다.

> 환경: Visual C++(x64/유니코드), Windows 10/11, MFC(Feature Pack 포함)
> 표기: `UI 스레드` = 메시지 펌프(윈도우/대화상자/리본)를 가진 스레드, `Worker 스레드` = 메시지 펌프 없음(기본)
> 핵심 원칙: **UI는 UI 스레드에서만**, **비용 큰 일은 Worker에서**, **스레드 간 통신은 메시지/동기화로 안전하게**

---

## 0. 큰 그림: MFC 스레드 모델 요약

| 구분 | CWinThread 생성 방식 | 메시지 펌프 | UI 생성 가능 | 주 사용처 |
|---|---|---|---|---|
| **UI Thread** | `AfxBeginThread(RUNTIME_CLASS(CMyThread))` (CWinThread 파생) | **있음** (`Run()` 루프) | 가능 (창/대화상자/컨트롤) | Top-level 보조 UI, 모델리스 대화상자, 프리뷰/도구창 |
| **Worker Thread** | `AfxBeginThread(MyProc, pParam)` (함수 포인터) | 없음(기본) | 불가(원칙) | 긴 계산/IO/네트워크/압축/DB/이미지 처리 등 |

**핵심 규칙**
1. **UI 객체(창/컨트롤) 생성·수정은 해당 객체를 만든 스레드**에서만.
2. **UI 스레드**만 가속기/메뉴/대화상자 같은 **메시지 루프**를 자동 보유.
3. **Worker 스레드**는 기본적으로 **메시지 펌프 없음** → `PostThreadMessage` 사용 가능(큐는 있음), `PeekMessage` 루프를 직접 돌려야 메시지를 소진.

---

## 1. 스레드 시작하기: 두 가지 API

### 1-1. Worker Thread (가장 흔함)

```cpp
// 1) 함수 시그니처: UINT AFX_CDECL ThreadProc(LPVOID pParam)
UINT AFX_CDECL Work_HeavyJob(LPVOID pParam) {
    // COM 필요 시: CoInitializeEx(nullptr, COINIT_MULTITHREADED);  // or APARTMENT
    // RAII로 해제: CoUninitialize();

    // 긴 작업 (예: 파일 스캔)
    // 주기적으로 취소 플래그 확인, 진행률 보고는 PostMessage/SendMessageTimeout 등으로
    return 0;
}

// 2) 시작 (UI 스레드에서)
CWinThread* p = AfxBeginThread(Work_HeavyJob, pParam, THREAD_PRIORITY_NORMAL, 0, 0);
// 반환된 CWinThread*는 Worker를 나타내지만, 이 스레드는 메시지펌프를 돌리지 않음
```

- 장점: 단순, 빠르게 도입 가능.
- 제약: 창/컨트롤 생성 금지(원칙). UI 갱신은 **UI 스레드로 Post**.

### 1-2. UI Thread (CWinThread 파생)

```cpp
// UI 스레드 클래스
class CPreviewThread : public CWinThread {
    DECLARE_DYNCREATE(CPreviewThread)
public:
    CWnd m_wnd; // 또는 CDialogEx
    virtual BOOL InitInstance() override {
        // 이 스레드의 메시지 펌프가 시작되기 전에 호출됨
        // 모델리스 대화상자 생성/표시 가능
        m_wnd.CreateEx(0, AfxRegisterWndClass(0), _T("Preview Pane"), WS_OVERLAPPEDWINDOW,
                       CRect(100,100,600,400), nullptr, 0);
        m_wnd.ShowWindow(SW_SHOW);
        return TRUE; // Run() 루프 시작
    }
    virtual int ExitInstance() override {
        // 정리
        return CWinThread::ExitInstance();
    }
};

// 시작
CWinThread* pThread = AfxBeginThread(RUNTIME_CLASS(CPreviewThread));
```

- 장점: 독립 UI(도구창/모델리스/프리뷰) 운영.
- 제약: UI 스레드 간 상호 호출 주의(교착/재진입). **스레드별 HWND 소유** 개념 확실히.

---

## 2. 스레드와 메시지: 안전한 교신 패턴

### 2-1. 사용자 정의 메시지와 `PostMessage`

```cpp
// 헤더(공통)
constexpr UINT WM_APP_PROGRESS  = WM_APP + 1;
constexpr UINT WM_APP_FINISHED  = WM_APP + 2;

// Worker → UI: 진행률 보고
// hwndTarget은 UI 스레드의 창 핸들
::PostMessage(hwndTarget, WM_APP_PROGRESS, (WPARAM)percent, 0);

// UI 핸들러
afx_msg LRESULT OnProgress(WPARAM w, LPARAM l) {
    int percent = (int)w;
    m_progressCtrl.SetPos(percent);
    return 0;
}
BEGIN_MESSAGE_MAP(CMyDlg, CDialogEx)
    ON_MESSAGE(WM_APP_PROGRESS, &CMyDlg::OnProgress)
END_MESSAGE_MAP()
```

- **PostMessage** = 비동기, **SendMessage** = 동기(교착 위험).
- **SendMessageTimeout** 으로 Hang 방어 가능.

### 2-2. `PostThreadMessage` 로 스레드 큐 사용

- **Worker 스레드**도 **스레드 메시지 큐**는 있다(최초 `PostThreadMessage` 전에 `PeekMessage` 호출 권장 X — MFC는 자동 생성됨).
- Worker가 자체 명령을 받고 싶으면 **간단한 메시지 펌프**를 돌린다.

```cpp
// Worker 쪽: 간단 메시지 루프 (취소/일시정지 등 명령 수신)
UINT AFX_CDECL Work_Looping(LPVOID p) {
    bool running = true, paused=false;
    while (running) {
        // 1) 펌프: 들어온 스레드 메시지 처리
        MSG msg;
        while (PeekMessage(&msg, nullptr, 0, 0, PM_REMOVE)) {
            if (msg.message == WM_QUIT) { running=false; break; }
            if (msg.message == WM_APP+100) { paused = (msg.wParam!=0); }  // 예: 일시정지
        }
        if (!running) break;
        if (paused) { Sleep(50); continue; }
        // 2) 실제 작업
        DoOneUnitOfWork();
    }
    return 0;
}

// UI → Worker 스레드로 명령 보내기
::PostThreadMessage(pWorkerThread->m_nThreadID, WM_APP+100, TRUE, 0); // pause
```

> **Tip**: Worker가 메시지 루프를 돌리면 APC/Timer(QueueTimer)도 적용할 수 있습니다.

---

## 3. 스레드 안전한 UI 갱신

**절대 금지**: Worker에서 직접 `SetWindowText`, `ListCtrl.InsertItem` 등 UI 호출.
**올바른 방법**: `PostMessage`/`SendMessageTimeout`/`CWnd::PostMessage`로 **UI 스레드에 위임**.

```cpp
// 안전: Worker → UI로 데이터 전달
struct RowData { CString name; int size; };
auto* p = new RowData{ _T("Report.docx"), 24 };
::PostMessage(m_hListOwner, WM_APP + 201, 0, (LPARAM)p);

// UI 핸들러
afx_msg LRESULT OnAddRow(WPARAM, LPARAM lp) {
    std::unique_ptr<RowData> row((RowData*)lp);
    int i = m_list.InsertItem(m_list.GetItemCount(), row->name);
    m_list.SetItemText(i, 1, std::to_wstring(row->size).c_str());
    return 0;
}
```

- 포인터 전달 시 **소유권**을 명확히(예: UI에서 `delete`).
- 크고 빈번한 데이터는 **고정 큐**로 배치 전송(섬세한 배압/드롭 정책).

---

## 4. 동기화 객체와 RAII

MFC 래퍼와 Win32 원본을 혼용 가능. **RAII가 중요**.

- **뮤텍스**: 다중 프로세스까지 락 → `CMutex`, `std::mutex`(동일 프로세스 한정)
- **크리티컬 섹션**: 프로세스 내부, 빠름 → `CCriticalSection`
- **이벤트**: 시그널/대기 → `CEvent`
- **세마포어**: 자원 카운트 → `CSemaphore`
- **다중 대기**: `CMultiLock` (여러 동기화 객체에 Wait)

```cpp
// 간단 RAII
class CAutoLock {
    CCriticalSection& m_cs;
public:
    CAutoLock(CCriticalSection& cs) : m_cs(cs) { m_cs.Lock(); }
    ~CAutoLock() { m_cs.Unlock(); }
};

// 사용
CCriticalSection g_cs;
std::vector<int> g_data;

void PushSafe(int v) {
    CAutoLock lock(g_cs);
    g_data.push_back(v);
}
```

---

## 5. 취소/타임아웃/중단 가능한 작업 패턴

### 5-1. 취소 플래그 + 배리어

```cpp
class CCancelable {
    std::atomic<bool> m_cancel{false};
public:
    void Cancel() { m_cancel = true; }
    bool IsCanceled() const { return m_cancel.load(std::memory_order_relaxed); }
};

UINT AFX_CDECL Work_Cancelable(LPVOID ctx) {
    CCancelable* c = (CCancelable*)ctx;
    for (int i=0; i<1000000; ++i) {
        if (c->IsCanceled()) break;
        // chunk 단위로 일함
    }
    return 0;
}
```

### 5-2. Waitable Event로 즉시 중단

```cpp
struct WorkCtx {
    HANDLE hCancel; // CreateEvent(NULL, TRUE, FALSE, NULL)
};

UINT AFX_CDECL Work_EventCancelable(LPVOID lp) {
    WorkCtx* w = (WorkCtx*)lp;
    while (true) {
        // IO or Sleep 교대
        DWORD rc = WaitForSingleObject(w->hCancel, 0);
        if (rc == WAIT_OBJECT_0) break;
        DoChunk();
    }
    return 0;
}
```

### 5-3. 타임아웃

- 작업 단위마다 `GetTickCount64()` 체크, 혹은 `WaitForSingleObject(hJob, timeout)` 내부 설계.
- UI에서 **취소 버튼** → `SetEvent(hCancel)` → Worker 즉시 탈출.

---

## 6. COM 초기화와 스레딩 모델

- 스레드에서 COM 사용 시 **반드시** `CoInitializeEx` 호출 후 `CoUninitialize`.
- UI 스레드는 **STA**가 일반적: `CoInitializeEx(nullptr, COINIT_APARTMENTTHREADED)`
- CPU 바운드/네트워크 Worker는 **MTA**가 유리: `COINIT_MULTITHREADED`
- COM 객체의 스레드 선호/마샬링 규칙 숙지.

```cpp
UINT AFX_CDECL Work_ComMta(LPVOID) {
    CoInitializeEx(nullptr, COINIT_MULTITHREADED);
    // ... COM 사용 ...
    CoUninitialize();
    return 0;
}
```

---

## 7. “진행률 & 취소 버튼” 대화상자 + Worker 예제

### 7-1. 대화상자 골격

```cpp
// ProgressDlg.h
class CProgressDlg : public CDialogEx {
    DECLARE_DYNAMIC(CProgressDlg)
public:
    CProgressDlg() : CDialogEx(IDD_PROGRESS) {}
    CProgressCtrl m_bar;
    CButton m_btnCancel;
    HANDLE m_hCancel = nullptr;
    CWinThread* m_pWorker = nullptr;

    enum { WM_APP_PROGRESS = WM_APP + 100, WM_APP_DONE = WM_APP + 101 };

    BOOL OnInitDialog() override {
        CDialogEx::OnInitDialog();
        m_bar.SubclassDlgItem(IDC_PROGRESS, this);
        m_btnCancel.SubclassDlgItem(IDCANCEL, this);
        m_bar.SetRange(0,100);
        m_hCancel = CreateEvent(NULL, TRUE, FALSE, NULL);
        // 시작
        m_pWorker = AfxBeginThread(&RunWork, this);
        return TRUE;
    }

    static UINT AFX_CDECL RunWork(LPVOID pParam) {
        auto* dlg = static_cast<CProgressDlg*>(pParam);
        for (int i=0;i<=100;i++) {
            if (WaitForSingleObject(dlg->m_hCancel, 0)==WAIT_OBJECT_0) break;
            ::PostMessage(dlg->m_hWnd, WM_APP_PROGRESS, i, 0);
            Sleep(30);
        }
        ::PostMessage(dlg->m_hWnd, WM_APP_DONE, 0, 0);
        return 0;
    }

    afx_msg LRESULT OnProgress(WPARAM w, LPARAM) {
        m_bar.SetPos((int)w);
        return 0;
    }
    afx_msg LRESULT OnDone(WPARAM, LPARAM) {
        if (m_hCancel) CloseHandle(m_hCancel), m_hCancel=nullptr;
        EndDialog(IDOK);
        return 0;
    }
    void OnCancel() override {
        if (m_hCancel) SetEvent(m_hCancel);
        // 대화상자 즉시 닫지 않고, Worker가 Done 포스트하면 종료
    }

    DECLARE_MESSAGE_MAP()
};

BEGIN_MESSAGE_MAP(CProgressDlg, CDialogEx)
    ON_MESSAGE(WM_APP_PROGRESS, &CProgressDlg::OnProgress)
    ON_MESSAGE(WM_APP_DONE, &CProgressDlg::OnDone)
END_MESSAGE_MAP()
```

- 사용자 취소 → 이벤트 세트 → Worker가 빠르게 인지하고 종료 → UI는 **Done 메시지**로 정리 후 종료.

---

## 8. 작업 큐/스레드 풀(경량 구현)

### 8-1. 고정 스레드 N + 작업 큐

```cpp
struct Task { std::function<void()> fn; };

class CTaskQueue {
    CCriticalSection m_cs;
    CEvent m_ev; // auto-reset
    std::deque<Task> m_q;
    std::vector<CWinThread*> m_workers;
    bool m_stop=false;

    static UINT AFX_CDECL WorkerProc(LPVOID p) {
        auto* self = (CTaskQueue*)p;
        while (true) {
            Task t;
            { // CRIT
                CAutoLock L(self->m_cs);
                while (self->m_q.empty() && !self->m_stop) {
                    L.~CAutoLock(); // unlock
                    self->m_ev.Lock(); // wait
                    new(&L) CAutoLock(self->m_cs); // relock
                }
                if (self->m_stop && self->m_q.empty()) break;
                t = std::move(self->m_q.front()); self->m_q.pop_front();
            }
            if (t.fn) t.fn();
        }
        return 0;
    }
public:
    CTaskQueue(int n=2) : m_ev(FALSE, FALSE) { // auto-reset, nonsignaled
        for (int i=0;i<n;i++) m_workers.push_back(AfxBeginThread(&WorkerProc, this));
    }
    ~CTaskQueue() {
        { CAutoLock L(m_cs); m_stop=true; }
        m_ev.SetEvent(); m_ev.SetEvent(); // 여러 번 깨우기
        for (auto* t : m_workers) WaitForSingleObject(t->m_hThread, INFINITE);
    }
    void Post(Task&& t) {
        { CAutoLock L(m_cs); m_q.push_back(std::move(t)); }
        m_ev.SetEvent();
    }
};
```

- UI에서 `queue.Post({[=]{ /*heavy*/ ::PostMessage(hwnd, WM_APP_DONE,0,0); }});` 식으로 사용.
- MFC의 `CThreadPool`은 없으니, 표준 C++ `std::thread`/`std::async`/`concurrency::task` 등도 선택지.

---

## 9. 메시지 펌프와 Wait: `MsgWaitForMultipleObjects`

UI 스레드에서 **핵심 핸들(Event/Process) 대기**와 **메시지 처리**를 병행할 때:

```cpp
void WaitWithPump(HANDLE hEvent, DWORD timeout) {
    DWORD end = GetTickCount() + timeout;
    for (;;) {
        DWORD remain = end - GetTickCount();
        DWORD rc = MsgWaitForMultipleObjects(1, &hEvent, FALSE, remain, QS_ALLINPUT);
        if (rc == WAIT_OBJECT_0) break; // event signaled
        if (rc == WAIT_TIMEOUT) break;
        if (rc == WAIT_OBJECT_0 + 1) {
            MSG msg;
            while (PeekMessage(&msg, nullptr, 0, 0, PM_REMOVE)) {
                TranslateMessage(&msg);
                DispatchMessage(&msg);
            }
        }
    }
}
```

- 긴 동기 작업 중에도 **UI 응답성** 유지(진행률 애니메이션, 취소 버튼 반응 등).

---

## 10. 타이머: UI vs Worker

- **UI 타이머**: `SetTimer`(WM_TIMER). **UI 스레드**에서만 자연스럽게 동작.
- **Worker 타이머**: `CreateWaitableTimer` or `RegisterWaitForSingleObject` or **메시지 루프를 갖춘 Worker에서 `SetTimer`**.

```cpp
// Worker에서 고해상도 주기 작업 (Waitable Timer)
HANDLE hTimer = CreateWaitableTimer(NULL, FALSE, NULL);
LARGE_INTEGER duetime{}; duetime.QuadPart = -10'000'000LL; // 1초 뒤
SetWaitableTimer(hTimer, &duetime, 1000, NULL, NULL, FALSE);

for(;;) {
    DWORD rc = WaitForSingleObject(hTimer, INFINITE);
    if (rc==WAIT_OBJECT_0) DoTick();
}
```

---

## 11. 네트워크/IO + 스레드 (아주 짧게)

- `CAsyncSocket`은 UI 스레드 메시지 기반이므로 **UI 또는 전용 UI thread**와 궁합.
- 블로킹 소켓/파일 IO는 Worker에 배치. UI와는 **메시지**로 교신.
- 고성능 서버/프록시는 IOCP(별도 아키텍처).

---

## 12. 데드락/재진입/수명관리 체크리스트

1. **SendMessage** 교차 호출 금지: UI ↔ Worker 간에는 가급적 **PostMessage**.
2. **락 순서** 통일: 다중 락 취득 시 순서 정의.
3. **객체 수명**: Worker가 UI 포인터 잡고 있을 때 UI가 먼저 파괴되면 위험 → **약한 참조/핸들 유효성 검사**.
4. **종료 동기화**: `WaitForSingleObject(pThread->m_hThread, INFINITE)` 로 종료 보장.
5. **예외/종료 경로**: 모든 핸들/이벤트/COM 해제.

---

## 13. 고급: UI Thread 안에 모델리스 폼 & 작업자 결합

- **UI Thread**(CWinThread 파생)가 **도구창/프리뷰**를 표시하고, 그 내부에서 **Worker**를 구동.
- 큰 도면/이미지 처리 중에도 프리뷰 스레드 UI는 부드럽게 반응.

```cpp
class CToolUiThread : public CWinThread {
    DECLARE_DYNCREATE(CToolUiThread)
public:
    CMyToolDlg m_dlg;
    BOOL InitInstance() override {
        m_dlg.Create(IDD_TOOL, nullptr);
        m_dlg.ShowWindow(SW_SHOW);
        return TRUE;
    }
    int ExitInstance() override { return CWinThread::ExitInstance(); }
};
```

---

## 14. 실무 자주 쓰는 패턴 모음

### 14-1. 큰 파일 처리(체크섬/변환) + 진행률

- Worker에서 **청크 단위**로 읽기/처리/보고.
- UI는 **누적/ETA** 표시, 취소 이벤트 감시.

### 14-2. DB/네트워크 파이프라인

- Worker A: Fetch → Queue → Worker B: Parse/Transform → Queue → Worker C: Save
- 각 큐는 **유한 버퍼**(배압), 종료 시 **Poison Pill**(특수 메시지)로 깔끔히 정리.

### 14-3. 스레드별 로그 버퍼

- contention 줄이려고 스레드 로컬 `std::vector<char>`에 쌓고 일정 단위마다 한 번에 파일 append.

---

## 15. 실수·버그 예방 리스트

- [ ] UI 스레드 외부에서 UI API 호출 X
- [ ] 취소 가능 지점 충분히 배치(루프 중 break)
- [ ] SendMessageTimeout 사용으로 Hang 방어
- [ ] 동기화 남용 금지(락 범위 최소화, 조건변수/이벤트 혼합)
- [ ] COM 초기화/해제 누락 X
- [ ] 스레드 종료 대기 및 리소스 해제 보장
- [ ] 크래시 덤프/로그로 사후 분석 가능하게

---

## 16. 빠른 스타터: 템플릿 세트

### 16-1. Worker + 진행률 + 취소

```cpp
// 헤더
constexpr UINT WM_APP_WORKPROG = WM_APP + 501;
constexpr UINT WM_APP_WORKDONE = WM_APP + 502;

struct WorkParam {
    HWND hNotify{};
    HANDLE hCancel{};
    std::wstring path;
};

UINT AFX_CDECL Work_Filescan(LPVOID lp) {
    auto* p = (WorkParam*)lp;
    uint64_t processed=0, total=Estimate(p->path);
    for (auto& f : EnumerateFiles(p->path)) {
        if (WaitForSingleObject(p->hCancel,0)==WAIT_OBJECT_0) break;
        processed += ProcessOne(f); // 비용 큰 작업
        int percent = (int)(processed*100/total);
        PostMessage(p->hNotify, WM_APP_WORKPROG, percent, 0);
    }
    PostMessage(p->hNotify, WM_APP_WORKDONE, 0, 0);
    return 0;
}
```

### 16-2. UI에서 실행

```cpp
void CMainDlg::OnBnClickedStart() {
    m_hCancel = CreateEvent(NULL, TRUE, FALSE, NULL);
    auto* wp = new WorkParam{ m_hWnd, m_hCancel, L"C:\\Data" };
    m_pThread = AfxBeginThread(&Work_Filescan, wp);
}
void CMainDlg::OnBnClickedCancel() {
    if (m_hCancel) SetEvent(m_hCancel);
}
LRESULT CMainDlg::OnWorkProg(WPARAM w, LPARAM) { m_prog.SetPos((int)w); return 0; }
LRESULT CMainDlg::OnWorkDone(WPARAM, LPARAM)   { /*버튼/상태 복구*/ return 0; }
```

---

## 17. 디버깅·프로파일링 팁

- **스레드 이름** 설정: `SetThreadDescription(pThread->m_hThread, L"Worker-FileScan");`
- **디버거**: Threads 창에서 스택 확인, **Deadlock** 시 **Wait Chain Traversal**(WCT) 도구.
- **ETW**/Xperf/WPA로 CPU 바운드/컨텍스트 스위치/디스크 IO 확인.
- 로그에 **ThreadId** 출력.

---

## 18. 요약

- **역할 분리**: **UI는 빠르고 반응성 유지**, **Worker가 무거운 일**.
- **통신은 메시지/큐**로, **SendMessage 교착** 금지.
- **취소/타임아웃/에러 경로**를 처음부터 포함.
- 필요시 **UI Thread**(CWinThread 파생)로 보조 UI를 붙이고, 내부에서 Worker를 안전하게 구동.
- **COM/동기화/수명관리**의 기초를 지키면, 복잡한 앱도 안정적이고 테스트 가능한 구조를 유지할 수 있습니다.
