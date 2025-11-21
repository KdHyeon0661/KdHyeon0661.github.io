---
layout: post
title: 영상처리 - Visual Studio 개발 환경 & 첫 프로그램
date: 2025-09-12 20:25:23 +0900
category: 영상처리
---
# Visual Studio 개발 환경 & 첫 프로그램

---

## | Visual Studio 개발 환경 구축

### A. Windows 프로그래밍과 Visual C++

#### 왜 Visual C++인가?

- **Win32 API / GDI / Direct2D / Direct3D / WIC / Media Foundation / MFC** 등 **네이티브 Windows** 스택을 **가장 잘 지원**합니다.
- **디버거/프로파일러/리소스 편집기**(아이콘·비트맵·다이얼로그·문자열·가속기 등)와 **MSBuild/Clang-CL/CMake** 등 **도구 통합**이 탁월합니다.
- 영상처리 실습에서 필요한 **DIB/StretchDIBits**, **BITMAPINFO**, **메모리 정렬/stride**, **고해상도 타이머(QueryPerformanceCounter)** 등을 빠르게 검증하기 좋습니다.

#### 프로젝트 유형 비교(요약)

- **Win32 API (순수 C/C++)**: 가장 가벼움. 소스가 적고, 창/메시지 루프/그리기를 직접 제어.
- **MFC (C++)**: 문서/뷰, 프레임워크 이벤트 모델, 리소스 UI 디자이너. **대규모 Win 데스크톱**에 적합.
- **CMake 프로젝트**: 크로스 플랫폼 지향. VS가 CMakeLists.txt를 인식해 빌드/디버깅 지원.
- **콘솔 프로젝트**: 영상 알고리즘만 빠르게 만들 때 유용(표시만 별도 툴로).

---

### B. Visual Studio Community 에디션 설치하기

> **권장 대상**: Visual Studio 2022 Community (무료, 개인/교육/소규모 팀). 이후 **Enterprise**와 기능 차이는 있지만, 본 장 실습에는 충분합니다.

#### 설치 준비

- **권장 OS**: Windows 10/11 (x64).
- **디스크 공간**: MFC/ATL, SDK, C++ 툴체인 포함 시 **10~20GB+** 여유 권장.
- **관리자 권한**: 설치/업데이트 시 필요할 수 있음.

#### 설치 절차(요약)

1. **Visual Studio Installer** 실행 → *Install* 선택.
2. **Workloads(작업 부하)** 탭에서 다음 체크:
   - **Desktop development with C++** ✅
     - 이 안에 포함된 선택(우측 Summary에서) 중 **MSVC v14x**, **Windows 10/11 SDK**, **C++ CMake tools**, **C++ ATL/MFC**를 확인.
3. **Individual components(개별)** 탭에서 누락된 항목 체크:
   - **MSVC v14x (x86/x64) build tools**
   - **Windows 10/11 SDK (10.0.x)**
   - **C++ MFC for latest v14x build tools (x86 & x64)** ✅
   - **C++ ATL for latest v14x build tools** (선택)
   - **C++ Clang tools for Windows** (선택)
   - **C++ AddressSanitizer/Static Analysis** (선택)
   - **Git for Windows**, **Python development**(스크립팅 필요 시)
4. **언어 팩**: 한국어/영어 동시 설치를 권장(문서/Stack Overflow 혼용 시 편리).
5. **설치 시작** → 완료 후 재부팅 권장.

> 🔎 **MFC 마법사가 보이지 않으면?**
> “**C++ MFC for latest v14x build tools**”가 **체크**되어 있는지 확인 → 설치 후 VS 재시작.

#### 설치 팁

- **vsconfig 내보내기**: 설치 구성을 `파일 → 계정 설정 → 설치 구성 내보내기`로 저장(팀 공유/재설치 편의).
- **오프라인 설치**: 네트워크 제한 환경에서는 layout 옵션으로 오프라인 캐시 생성 가능.
- **성능 팁**: SSD 설치, 백신 실시간 검사 예외 폴더(신중), RAM 16GB+ 권장.

---

### C. Visual Studio 첫 실행 — 필수 설정

#### 도구 → 옵션(Options)

- **환경 → 문서**: 자동 다시 로드, 파일 변경 감지 옵션 활성화.
- **텍스트 편집기 → C/C++ → 서식**: **clang-format** 사용 시 *Use .clang-format* 체크.
- **프로젝트 및 솔루션 → VC++ 디렉터리**: 3rd-party 라이브러리 경로(예: `C:\dev\include`, `C:\dev\lib`) 사전 준비.
- **디버깅**:
  - *Just My Code* 해제(프레임워크 내부 추적 시)
  - **네이티브/관리 코드 혼합 디버깅** 필요 시 설정
  - *Symbol* 캐시 경로 지정(빠른 PDB 로드)
- **IntelliSense**: 최대 동시 프로세스, 인덱싱 설정 점검.

#### 개발자 명령 프롬프트

- 시작 메뉴 → **x64 Native Tools Command Prompt for VS**
  - `cl`, `link`, `dumpbin`, `editbin`, `mt`, `rc`, `msbuild` 등 사용 가능.
- **MSBuild**:
  ```bat
  msbuild MyApp.sln /m /p:Configuration=Release;Platform=x64
  ```

#### Git & 코드 규칙

- **Git 통합**: VS 상단 *Git* 메뉴 → Clone/Create → `.gitignore(VisualStudio)`
- **.editorconfig / .clang-format**: 팀 규칙 고정(탭/스페이스, 줄바꿈, 랩핑, include 순서 등)

---

## | First 프로그램 예제

> 여기서는 **두 가지**를 만든 뒤, 차이를 비교합니다.
> **(1) Win32 API 최소 예제** — 창 생성 + **영상 버퍼(DIB)**를 그려보기
> **(2) MFC 응용 프로그램 마법사** — SDI 기반 스켈레톤 + 뷰에서 그리기

---

### A. 새 프로젝트 만들기

#### 생성

1. **파일 → 새로 만들기 → 프로젝트**
2. 템플릿 검색: “*Windows Desktop Application*” 또는 “*Win32 Project*”
   - 최신 VS에서는 **“Windows Desktop Application (C++)”** 템플릿 사용
3. **프로젝트 이름**: `FirstWin32DIB`
4. **솔루션**: 새 솔루션(체크박스) → 위치 지정
5. **Create** 클릭 → 옵션에서 **Empty project** 선택(불필요 파일 없는 상태)

#### MFC 프로젝트 생성

1. **파일 → 새로 만들기 → 프로젝트**
2. 템플릿: **MFC App** 또는 **MFC Application** (보이지 않으면 MFC 컴포넌트 설치 필수)
3. **프로젝트 이름**: `FirstMFCApp`
4. **Create** → **MFC 응용 프로그램 마법사** 실행

---

### B. MFC 응용 프로그램 마법사 — 옵션 설명(상세)

> 마법사는 다이얼로그 **여러 페이지**로 구성됩니다. 주요 체크 포인트를 정리합니다.

#### 애플리케이션 유형(Application Type)

- **장치 프레임워크**:
  - **SDI**(Single Document Interface): 하나의 문서/뷰. 영상뷰 실습에 적합.
  - **MDI**(Multiple Document Interface): 탭/다중 문서.
  - **Dialog-based**: 대화상자 중심 UI(간단 툴/설정 위주 앱에 적합).
- **문서/뷰 아키텍처**: **문서(데이터)** ↔ **뷰(렌더링/인터랙션)** ↔ **프레임(메뉴/툴바/상태바)**

#### Use of MFC

- **Use MFC in a Shared DLL**(권장): exe 크기 작고 시스템 공유 DLL 사용
- **Use MFC in a Static Library**: 배포 단순(외부 DLL 의존↓), exe가 커짐

#### 프로젝트 스타일 & 기능

- **Visual Style and Colors**: Windows 10/11 UX와 조화되는 테마
- **Command Bar/Menu**: 리본/전통 메뉴/툴바 선택
- **Docking/Tabbed**: 도킹 패널, 탭 문서
- **Advanced Features**: OLE/Automation, ActiveX, Context Help, Restart Manager 등

#### 데이터/문서 기능

- **Document/View architecture support**: 체크 유지
- **File extension/Document string**: 나중에 파일 연동(더블클릭) 시 활용
- **Serialization**: 문서 저장/불러오기(CArchive) 필요 시 체크

#### 보안/네트워킹

- **Security**: DEP/ASLR 기본 활성
- **Windows Sockets**: 네트워크 기능 필요 시 체크(실습에서는 불필요)

> ✅ **최소 추천 구성**: *SDI + Shared MFC DLL + 기본 메뉴/툴바/상태바 + Doc/View + Serialization(선택)*

---

### C. Visual Studio 구조 — 솔루션/프로젝트/필터

#### & 프로젝트(.vcxproj)

- **솔루션**: 여러 프로젝트 묶음(예: 앱 + 라이브러리 + 테스트)
- **프로젝트**: 빌드 단위(소스/헤더/리소스/설정 포함)

#### 필터(.vcxproj.filters)

- *가상 폴더* 개념(디스크 구조와 별개). `Source Files`, `Header Files`, `Resource Files` 등 정리용.

#### 리소스(.rc)와 res 폴더

- **리소스 편집기**로 아이콘, 비트맵, 커서, 대화상자, 문자열, 가속기 등을 시각 편집.
- 헤더(`resource.h`)에 심볼 ID가 정의됨(예: `IDR_MAINFRAME`).

---

### D. 프로젝트 속성 — 필수 체크리스트

> **솔루션 탐색기 → 프로젝트 → 속성**에서 구성/플랫폼 별로 설정합니다.
> (상단 드롭다운: `Debug/Release`, `Win32/x64/ARM64/ARM64EC`)

#### 일반(General)

- **플랫폼 도구 집합**: 최신(v14x)
- **Windows SDK 버전**: 설치된 최신(예: 10.0.22621.x)

#### C/C++ → 언어/전처리기/최적화

- **C++ 표준**: `/std:c++17` 이상
- **경고 수준**: `/W4` 권장, 가능하면 **경고를 오류로** `/WX`
- **전처리기**: `_UNICODE`, `UNICODE` 사용(문자 집합은 “Use Unicode Character Set”)
- **SDL 체크**: `/sdl`(안전성)
- **전처컴파일 헤더(PCH)**: *사용* 권장(빌드 속도 개선)

#### 링커 → 시스템

- **서브시스템**: `Windows (/SUBSYSTEM:WINDOWS)` (콘솔은 `CONSOLE`)
- **최적화**: LTO(/GL) + LTCG(/LTCG) 조합은 대규모 프로젝트에서 고려

#### 코드 생성

- **런타임 라이브러리**:
  - Debug: `/MDd`(멀티스레드 DLL, 디버그)
  - Release: `/MD`
- **Spectre 완화 라이브러리**: 보안 요구 시 선택

#### 고급

- **/permissive-**: 표준 준수 엄격 모드
- **멀티프로세서 컴파일**: `/MP`

---

### Win32 API + DIB로 그라데이션 버퍼 그리기

> 목적: **영상 버퍼(2D 배열)**를 만들어 창에 **StretchDIBits**로 출력합니다.
> 포인트: **stride(패딩)**, **BITMAPINFO**, **WM_PAINT** 처리.

#### 소스 추가

- `FirstWin32DIB.cpp` 파일 생성 후 아래 코드 삽입:

```cpp
#include <windows.h>
#include <cstdint>
#include <vector>
#include <string>

// 창 크기
constexpr int WIN_W = 640;
constexpr int WIN_H = 480;

// 영상 버퍼 크기 (그레이 8-bit)
constexpr int IMG_W = 256;
constexpr int IMG_H = 256;

// 전역(샘플 단순화를 위함)
HINSTANCE g_hInst = nullptr;
HWND      g_hWnd  = nullptr;
std::vector<uint8_t> g_image; // size = IMG_W * IMG_H, stride = IMG_W

// 간단한 그라데이션 이미지 생성
void FillGradient(std::vector<uint8_t>& img, int w, int h) {
    img.resize(w * h);
    for (int y = 0; y < h; ++y) {
        for (int x = 0; x < w; ++x) {
            // 대각선 그라데이션: (x + y) % 256
            img[y * w + x] = static_cast<uint8_t>((x + y) & 0xFF);
        }
    }
}

// WM_PAINT에서 StretchDIBits로 그리기
void PaintImage(HDC hdc, const std::vector<uint8_t>& img, int w, int h, RECT client) {
    // BITMAPINFO (그레이 8-bit → 팔레트가 필요한데, 여기서는 8-bit indexed 대신 24-bit로 변환하여 그립니다)
    // 간단화를 위해 화면 출력 직전에 24-bit BGR 버퍼로 변환
    const int dstW = client.right - client.left;
    const int dstH = client.bottom - client.top;

    // 24-bit BGR: 3바이트/픽셀, 행 패딩은 4바이트 배수
    int bpp   = 24;
    int bytesPerPixel = bpp / 8;
    int stride = ((w * bytesPerPixel + 3) / 4) * 4;

    std::vector<uint8_t> bgr(stride * h);

    for (int y = 0; y < h; ++y) {
        uint8_t* row = bgr.data() + y * stride;
        for (int x = 0; x < w; ++x) {
            uint8_t g = img[y * w + x];
            row[x * 3 + 0] = g; // B
            row[x * 3 + 1] = g; // G
            row[x * 3 + 2] = g; // R
        }
    }

    BITMAPINFO bmi{};
    bmi.bmiHeader.biSize        = sizeof(BITMAPINFOHEADER);
    bmi.bmiHeader.biWidth       = w;
    bmi.bmiHeader.biHeight      = -h; // top-down
    bmi.bmiHeader.biPlanes      = 1;
    bmi.bmiHeader.biBitCount    = bpp; // 24-bit
    bmi.bmiHeader.biCompression = BI_RGB;
    bmi.bmiHeader.biSizeImage   = stride * h;

    StretchDIBits(
        hdc,
        0, 0, dstW, dstH,      // dst rect (client area scaling)
        0, 0, w, h,            // src rect
        bgr.data(),
        &bmi,
        DIB_RGB_COLORS,
        SRCCOPY
    );
}

// WndProc
LRESULT CALLBACK WndProc(HWND hwnd, UINT msg, WPARAM wParam, LPARAM lParam) {
    switch (msg) {
    case WM_PAINT: {
        PAINTSTRUCT ps{};
        HDC hdc = BeginPaint(hwnd, &ps);
        RECT rc{};
        GetClientRect(hwnd, &rc);
        PaintImage(hdc, g_image, IMG_W, IMG_H, rc);
        EndPaint(hwnd, &ps);
        return 0;
    }
    case WM_SIZE:
        InvalidateRect(hwnd, nullptr, FALSE);
        return 0;
    case WM_KEYDOWN:
        if (wParam == VK_SPACE) { // 스페이스로 패턴 변경
            FillGradient(g_image, IMG_W, IMG_H);
            InvalidateRect(hwnd, nullptr, FALSE);
        }
        return 0;
    case WM_DESTROY:
        PostQuitMessage(0);
        return 0;
    default:
        return DefWindowProc(hwnd, msg, wParam, lParam);
    }
}

int APIENTRY wWinMain(HINSTANCE hInstance, HINSTANCE, LPWSTR, int nCmdShow) {
    g_hInst = hInstance;
    const wchar_t CLASS_NAME[] = L"FirstWin32DIBClass";
    const wchar_t TITLE[]      = L"First Win32 DIB — StretchDIBits";

    // 윈도우 클래스 등록
    WNDCLASSEXW wc{};
    wc.cbSize        = sizeof(wc);
    wc.style         = CS_HREDRAW | CS_VREDRAW | CS_OWNDC;
    wc.lpfnWndProc   = WndProc;
    wc.hInstance     = g_hInst;
    wc.hCursor       = LoadCursor(nullptr, IDC_ARROW);
    wc.hbrBackground = (HBRUSH)(COLOR_WINDOW + 1);
    wc.lpszClassName = CLASS_NAME;

    if (!RegisterClassExW(&wc)) return 0;

    // 창 생성
    g_hWnd = CreateWindowExW(
        0, CLASS_NAME, TITLE,
        WS_OVERLAPPEDWINDOW | WS_VISIBLE,
        CW_USEDEFAULT, CW_USEDEFAULT, WIN_W, WIN_H,
        nullptr, nullptr, g_hInst, nullptr);

    if (!g_hWnd) return 0;

    // 영상 버퍼 준비
    FillGradient(g_image, IMG_W, IMG_H);

    // 메시지 루프
    ShowWindow(g_hWnd, nCmdShow);
    UpdateWindow(g_hWnd);

    MSG msg{};
    while (GetMessageW(&msg, nullptr, 0, 0)) {
        TranslateMessage(&msg);
        DispatchMessageW(&msg);
    }
    return (int)msg.wParam;
}
```

##### 빌드/실행

- 상단 툴바: `Debug | x64` 선택 → **로컬 Windows 디버거(▶)**
- 창이 열리면 **스페이스바**를 눌러 갱신. 창 크기 조절 시 **StretchDIBits**로 스케일링됨.

##### 코드 해설(핵심)

- **DIB 출력**: `BITMAPINFO + StretchDIBits`
- **top-down 비트맵**: `biHeight = -h` (상하 반전 방지)
- **stride 정렬**: 24-bit는 행이 **4바이트 배수**가 되도록 패딩(`((w*3+3)/4)*4`)
- **WM_PAINT**에서만 GDI 호출 → 깜빡임/성능 문제 최소화

> 📌 **실전 팁**: 그레이 8-bit를 **팔레트 없이** 바로 그리려면 **32-bit BGRX**로 변환하여 `biBitCount=32`로 출력하는 패턴이 흔합니다(변환은 약간 더 빠르고 코드가 단순).

---

### MFC SDI — 뷰에서 픽셀 렌더링

> 마법사로 **SDI** 프로젝트를 생성한 뒤, **뷰 클래스**(`CYourProjectView`)의 `OnDraw` 또는 `OnPaint`에서 DIB 출력 패턴을 적용합니다.
> 여기서는 **뷰의 OnDraw**에서 24-bit BGR 버퍼를 그리는 간단 예를 보입니다.

#### 마법사로 프로젝트 생성

- `FirstMFCApp` → **SDI**, **Use MFC in a Shared DLL**, 기본 메뉴/툴바/상태바 유지 → Finish.

생성 후 주요 파일:
- `FirstMFCApp.cpp / .h` — `CFirstMFCApp` 애플리케이션 클래스
- `MainFrm.cpp / .h` — `CMainFrame` 프레임 창
- `FirstMFCAppDoc.cpp / .h` — 문서 클래스
- `FirstMFCAppView.cpp / .h` — 뷰 클래스(그리기 구현 위치)
- `resource.h`, `FirstMFCApp.rc`, `res\*` — 리소스

#### 뷰에 렌더링 코드 추가

- `FirstMFCAppView.h`에 **이미지 버퍼** 멤버 추가:

```cpp
// FirstMFCAppView.h
class CFirstMFCAppView : public CView
{
protected:
    CFirstMFCAppView() noexcept;
    DECLARE_DYNCREATE(CFirstMFCAppView)

public:
    enum { IMG_W = 256, IMG_H = 256 };
    std::vector<uint8_t> m_gray; // IMG_W * IMG_H
    std::vector<uint8_t> m_bgr;  // 24-bit BGR (stride 포함)
    int m_stride = 0;

    void PrepareImage();
    void DrawImage(CDC* pDC, const CRect& client);

    // ...
protected:
    virtual void OnDraw(CDC* pDC);
    virtual BOOL PreCreateWindow(CREATESTRUCT& cs);
    DECLARE_MESSAGE_MAP()
};
```

- `FirstMFCAppView.cpp`에 구현:

```cpp
#include "pch.h"
#include "framework.h"
#include "FirstMFCApp.h"
#include "FirstMFCAppDoc.h"
#include "FirstMFCAppView.h"
#include <algorithm>

#ifdef _DEBUG
#define new DEBUG_NEW
#endif

IMPLEMENT_DYNCREATE(CFirstMFCAppView, CView)

BEGIN_MESSAGE_MAP(CFirstMFCAppView, CView)
    // 필요 시 메시지 추가
END_MESSAGE_MAP()

CFirstMFCAppView::CFirstMFCAppView() noexcept
{
    // 초기 버퍼 생성
    PrepareImage();
}

BOOL CFirstMFCAppView::PreCreateWindow(CREATESTRUCT& cs)
{
    return CView::PreCreateWindow(cs);
}

void CFirstMFCAppView::PrepareImage()
{
    m_gray.resize(IMG_W * IMG_H);

    // 간단 그라데이션
    for (int y = 0; y < IMG_H; ++y) {
        for (int x = 0; x < IMG_W; ++x) {
            m_gray[y * IMG_W + x] = static_cast<uint8_t>((x ^ y) & 0xFF);
        }
    }
    int bytesPerPixel = 3;
    m_stride = ((IMG_W * bytesPerPixel + 3) / 4) * 4;
    m_bgr.assign(m_stride * IMG_H, 0);

    for (int y = 0; y < IMG_H; ++y) {
        uint8_t* row = m_bgr.data() + y * m_stride;
        for (int x = 0; x < IMG_W; ++x) {
            uint8_t g = m_gray[y * IMG_W + x];
            row[x * 3 + 0] = g; // B
            row[x * 3 + 1] = g; // G
            row[x * 3 + 2] = g; // R
        }
    }
}

void CFirstMFCAppView::DrawImage(CDC* pDC, const CRect& client)
{
    // BITMAPINFO 구성
    BITMAPINFO bmi{};
    bmi.bmiHeader.biSize        = sizeof(BITMAPINFOHEADER);
    bmi.bmiHeader.biWidth       = IMG_W;
    bmi.bmiHeader.biHeight      = -IMG_H; // top-down
    bmi.bmiHeader.biPlanes      = 1;
    bmi.bmiHeader.biBitCount    = 24;
    bmi.bmiHeader.biCompression = BI_RGB;
    bmi.bmiHeader.biSizeImage   = m_stride * IMG_H;

    ::StretchDIBits(
        pDC->m_hDC,
        client.left, client.top, client.Width(), client.Height(),
        0, 0, IMG_W, IMG_H,
        m_bgr.data(),
        &bmi,
        DIB_RGB_COLORS,
        SRCCOPY
    );
}

void CFirstMFCAppView::OnDraw(CDC* pDC)
{
    CFirstMFCAppDoc* pDoc = GetDocument();
    ASSERT_VALID(pDoc);
    if (!pDoc) return;

    CRect rc;
    GetClientRect(&rc);
    DrawImage(pDC, rc);
}
```

- **빌드 & 실행**: `Debug | x64` → ▶
  창이 열리면 **뷰 영역에 그라데이션**이 나타납니다. 창 크기 변경 시 자동 리스케일.

> 🧭 **Doc/View 연결**: 실제로는 `CDocument`에 **영상 버퍼**(예: `std::vector<uint8_t>`)를 두고, `CView::OnDraw`에서 **문서의 데이터를 읽어 렌더링**하는 패턴이 정석입니다. 이렇게 하면 **열기/저장(Serialization)**, **멀티 뷰** 확장에 유리합니다.

---

### G. Visual Studio 창/도구 — “어디서 무엇을 하나요?”

- **Solution Explorer**: 파일/프로젝트/참조/빌드 구성 관리
- **Properties**: 구성별 컴파일러/링커/디버거 설정
- **Resource View**: `.rc` 리소스 트리(아이콘/대화상자/문자열/가속기)
- **Output / Error List**: 빌드/링커 메시지, 경고/에러
- **Task List / TODO**: 주석 기반 태스크 추적
- **Graphics Diagnostics(선택)**: DirectX 디버깅
- **Performance Profiler**: CPU 샘플링, File I/O, UI 응답성
- **Memory / Registers / Disassembly**: 네이티브 심층 디버깅
- **Watch / Autos / Locals / Call Stack**: 상태 추적

---

### H. 프로그램 빌드 및 실행 — 체크리스트

#### 구성/플랫폼

- `Debug/Release`, `x86/x64/ARM64/ARM64EC`를 **명확히 인지**
- 영상처리 실습은 보통 **x64**를 권장(큰 버퍼/성능)

#### 전처컴파일 헤더(PCH)

- **헤더 포함 순서**가 꼬이면 컴파일 오류가 나기 쉬움.
- MFC 템플릿의 `pch.h`/`pch.cpp` 관례 준수.

#### 링크 오류 대처

- **unresolved external symbol**: 라이브러리 누락/정의 불일치
- **mfcXXXu.lib**: MFC 정적/공유 설정(**Use of MFC**)이 현재 구성과 맞는지 확인
- **/MD vs /MT** 혼합 주의

#### 디버깅

- **중단점(F9)**, **단계 실행(F10/F11)**
- **Watch/Memory**로 버퍼 확인(예: `m_bgr[0]`, `m_bgr[1]` …)
- **natvis**로 사용자 정의 타입 시각화 가능

#### MSBuild/CI

- 명령줄 빌드:
  ```bat
  msbuild FirstWin32DIB.sln /m /p:Configuration=Release;Platform=x64
  ```
- 아티팩트: `.\x64\Release\FirstWin32DIB.exe`

---

## I. 보너스 — CMake/Vcpkg로 외부 라이브러리와 병행

> 중급 이상: **CMake**로 프로젝트를 구성하면 VS뿐 아니라 다른 IDE/툴체인과 호환이 쉬워집니다.

- VS 메뉴 **File → Open → CMake**로 바로 열기
- `vcpkg integrate install` 후 `find_package(OpenCV CONFIG REQUIRED)` 등으로 사용
- CMake Presets(`CMakePresets.json`)로 `x64-Release` 등 **구성 프로파일** 표준화

---

## J. 흔한 문제 & 해결(FAQ)

**Q1. MFC 템플릿이 안 보입니다.**
A. VS Installer → **Individual components**에서 **C++ MFC for latest v14x**를 설치하세요. 설치 후 VS 재시작.

**Q2. 빌드 오류: Windows SDK가 없습니다.**
A. “Desktop development with C++” 워크로드와 **Windows 10/11 SDK** 체크. 프로젝트 속성의 **SDK 버전**을 설치된 버전으로 **Retarget**.

**Q3. 실행하면 검은 창만 나옵니다.**
A. `WM_PAINT` 처리에서 실제 그리기 코드가 호출되는지, `biHeight` 부호, `stride` 계산을 확인하세요. `InvalidateRect`로 새로고침 트리거.

**Q4. MFC SDI에서 이미지가 뒤집혀 보입니다.**
A. `BITMAPINFOHEADER.biHeight = -height`로 **top-down**을 사용하세요. 양의 높이는 bottom-up(상하 반전)입니다.

**Q5. x86/ x64 혼동으로 링크 실패합니다.**
A. 라이브러리/Dependencies가 **플랫폼 일치**하는지 확인하세요. 솔루션 구성 관리(대상 플랫폼 통일).

---

## K. 수학 메모 — 스케일링과 보간(간단 정리)

창 크기 변화에 따른 **선형 스케일링**에서, `(u,v)` 소스 좌표와 정수 격자 `(x,y)`의 관계:

$$
\begin{aligned}
u &= \frac{x}{W_\text{dst}} \cdot W_\text{src}, \quad
v  = \frac{y}{H_\text{dst}} \cdot H_\text{src} \\
\text{Nearest: } &\quad f[x,y] = F\big(\lfloor u + 0.5 \rfloor, \lfloor v + 0.5 \rfloor\big) \\
\text{Bilinear: } &\quad f[x,y] = \sum_{i=0}^{1}\sum_{j=0}^{1} w_{ij}\,F(\lfloor u \rfloor+i, \lfloor v \rfloor+j)
\end{aligned}
$$

본 장 예제는 **GDI StretchDIBits**가 내부에서 보간/필터링을 수행하므로, 직접 구현 없이 **개념만** 익혀도 충분합니다.

---

## L. 마무리 — 오늘 얻은 것

1. **Visual Studio 설치/구성**: C++/MFC/SDK/툴체인을 정확히 준비하는 법
2. **Win32 API 최소 예제**: 창/메시지 루프/`StretchDIBits`로 **영상 버퍼 출력**
3. **MFC SDI 스켈레톤**: **뷰에서 그리기**의 표준 위치와 리소스 구조
4. **빌드/디버깅 루틴**: 구성/플랫폼, PCH, 링크/디버그 체크리스트
