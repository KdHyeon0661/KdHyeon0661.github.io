---
layout: post
title: 영상처리 - ImageTool 프로그램
date: 2025-09-18 16:25:23 +0900
category: 영상처리
---
# ImageTool 프로그램

## 1) 프로젝트 생성

1. Visual Studio → **Create a new project** → **Windows Desktop Application**(C++/Empty Project) 선택  
2. 프로젝트 이름: `ImageTool`  
3. 솔루션 생성 후, **소스 파일**에 아래 2개 파일 추가:
   - `IppDib.hpp` (영상 클래스)
   - `ImageTool.cpp` (WinMain/윈도우/메뉴/그리기)

> 속성 권장
> - C/C++ → Language → **/std:c++20**
> - C/C++ → General → **SDL checks: Yes**(선택)
> - C/C++ → Code Generation → **/MD(d)**
> - 사용 문자 집합: **유니코드(Unicode)**

---

## 2) 영상 클래스 추가하기 — `IppDib.hpp`

다음 헤더는 **BI_RGB(무압축) 8/24bpp** BMP 입출력, **Top→Down 내부 저장**, **그리기(StretchDIBits)** 를 지원합니다.

```cpp
// IppDib.hpp
#pragma once
#define UNICODE
#define _UNICODE
#include <windows.h>
#include <cstdint>
#include <vector>
#include <string>
#include <fstream>
#include <algorithm>
#include <stdexcept>

class IppDib {
public:
    // 메타
    int  width  = 0;
    int  height = 0;   // 내부는 Top->Down(양수 보관), 그리기는 음수 높이로 전달
    int  bpp    = 0;   // 8 또는 24
    int  stride = 0;   // 바이트 단위, 4바이트 정렬 (DIB 규칙)
    bool valid() const { return width>0 && height>0 && (bpp==8||bpp==24) && !pixels.empty(); }

    // 픽셀/팔레트
    std::vector<uint8_t> pixels;   // Top->Down
    std::vector<RGBQUAD> palette;  // 8bpp면 256개 그레이, 24bpp면 비어 있음

public:
    void Destroy() {
        width = height = bpp = stride = 0;
        pixels.clear();
        palette.clear();
    }

    // 8bpp Gray
    bool CreateGray(int w, int h) {
        if (w<=0 || h<=0) return false;
        Destroy();
        width=w; height=h; bpp=8; stride = ((w*bpp + 31)/32)*4;
        palette.resize(256);
        for (int i=0;i<256;++i) palette[i] = RGBQUAD{ (BYTE)i,(BYTE)i,(BYTE)i,0 };
        pixels.assign((size_t)stride*height, 0);
        return true;
    }

    // 24bpp BGR
    bool CreateColor(int w, int h) {
        if (w<=0 || h<=0) return false;
        Destroy();
        width=w; height=h; bpp=24; stride = ((w*bpp + 31)/32)*4;
        pixels.assign((size_t)stride*height, 0);
        return true;
    }

    // 간단한 접근자 (Top->Down)
    uint8_t*       RowPtr(int y)       { return pixels.data() + (size_t)stride*y; }
    const uint8_t* RowPtr(int y) const { return pixels.data() + (size_t)stride*y; }

    // BMP 저장 (Bottom->Up으로 기록)
    bool SaveBmp(const std::wstring& path) const {
        if (!valid()) return false;

        BITMAPFILEHEADER bmf{};
        BITMAPINFOHEADER bmi{};
        bmi.biSize = sizeof(BITMAPINFOHEADER);
        bmi.biWidth  = width;
        bmi.biHeight = height;               // Bottom->Up (양수)
        bmi.biPlanes = 1;
        bmi.biBitCount = (WORD)bpp;
        bmi.biCompression = BI_RGB;
        bmi.biSizeImage  = (DWORD)(stride * height);
        bmi.biClrUsed    = (bpp==8)?256:0;

        DWORD offBits = sizeof(BITMAPFILEHEADER) + sizeof(BITMAPINFOHEADER) + (DWORD)palette.size()*sizeof(RGBQUAD);
        bmf.bfType    = 0x4D42;  // 'BM'
        bmf.bfOffBits = offBits;
        bmf.bfSize    = offBits + bmi.biSizeImage;

        std::ofstream ofs(path, std::ios::binary);
        if (!ofs) return false;

        ofs.write((const char*)&bmf, sizeof(bmf));
        ofs.write((const char*)&bmi, sizeof(bmi));
        if (!palette.empty())
            ofs.write((const char*)palette.data(), (std::streamsize)palette.size()*sizeof(RGBQUAD));

        // Bottom->Up으로 뒤집어 내보내기
        for (int y=height-1; y>=0; --y) {
            ofs.write((const char*)RowPtr(y), stride);
        }
        return true;
    }

    // BMP 불러오기 (BI_RGB, 8/24bpp)
    bool LoadBmp(const std::wstring& path) {
        Destroy();

        std::ifstream ifs(path, std::ios::binary);
        if (!ifs) return false;

        BITMAPFILEHEADER bmf{};
        BITMAPINFOHEADER bmi{};

        ifs.read((char*)&bmf, sizeof(bmf));
        if (!ifs || bmf.bfType != 0x4D42) return false;

        ifs.read((char*)&bmi, sizeof(bmi));
        if (!ifs || (bmi.biCompression!=BI_RGB) || !(bmi.biBitCount==8 || bmi.biBitCount==24))
            return false;

        const int W = bmi.biWidth;
        const int H = (bmi.biHeight<0)? -bmi.biHeight : bmi.biHeight;
        const int B = bmi.biBitCount;
        const int S = ((W*B + 31)/32)*4;

        // 팔레트
        size_t palCount = 0;
        if (B==8) {
            palCount = bmi.biClrUsed ? bmi.biClrUsed : 256;
            palette.resize(palCount);
            ifs.read((char*)palette.data(), (std::streamsize)palCount*sizeof(RGBQUAD));
            if (!ifs) return false;
        } else {
            palette.clear();
        }

        // 픽셀 데이터 위치로 이동
        ifs.seekg(bmf.bfOffBits, std::ios::beg);
        if (!ifs) return false;

        std::vector<uint8_t> raw((size_t)S*H);
        ifs.read((char*)raw.data(), raw.size());
        if (!ifs) return false;

        // 내부는 Top->Down으로 보관
        width=W; height=H; bpp=B; stride=S;
        pixels.assign((size_t)stride*height, 0);

        if (bmi.biHeight>0) {
            // 입력이 Bottom->Up → 뒤집어서 Top->Down
            for (int y=0; y<H; ++y) {
                std::copy_n(raw.data() + (size_t)S*(H-1-y), S, RowPtr(y));
            }
        } else {
            // 입력이 이미 Top->Down
            for (int y=0; y<H; ++y) {
                std::copy_n(raw.data() + (size_t)S*y, S, RowPtr(y));
            }
        }
        return true;
    }

    // 화면 출력(1:1), (x,y) 좌표에 Top->Down으로 그리기
    bool DrawAt(HDC hdc, int x, int y) const {
        if (!valid()) return false;

        // 그리기용 BITMAPINFO (Top->Down: biHeight 음수)
        struct BI_PACK {
            BITMAPINFOHEADER hdr;
            RGBQUAD pal[256]; // 8bpp만 사용
        } bi{};
        bi.hdr.biSize = sizeof(BITMAPINFOHEADER);
        bi.hdr.biWidth = width;
        bi.hdr.biHeight = -height; // Top->Down
        bi.hdr.biPlanes = 1;
        bi.hdr.biBitCount = (WORD)bpp;
        bi.hdr.biCompression = BI_RGB;
        bi.hdr.biSizeImage = 0;
        bi.hdr.biClrUsed = (bpp==8)?256:0;
        if (bpp==8) {
            for (int i=0;i<256;++i) bi.pal[i]=palette[i];
        }

        int ret = ::StretchDIBits(
            hdc,
            x, y, width, height,     // dst
            0, 0, width, height,     // src
            pixels.data(),
            reinterpret_cast<BITMAPINFO*>(&bi),
            DIB_RGB_COLORS,
            SRCCOPY
        );
        return ret != GDI_ERROR;
    }
};
```

---

## 3) 애플리케이션 — `ImageTool.cpp`

- **시작 시 빈 창을 띄우지 않음**: 창을 **숨긴 상태로 생성** → 파일 열기 대화상자 → 성공 시 **창 크기를 영상 크기에 맞춘 뒤** 표시. 취소하면 바로 종료.
- **BMP 불러오기/저장**: 메뉴 **파일→열기(O)… / 다른 이름으로 저장(S)…**
- **화면 출력 & 해칭**: 이미지가 창보다 작으면 **여백 4면을 해칭**(비스듬 빗금)으로 칠함.

```cpp
// ImageTool.cpp
#define UNICODE
#define _UNICODE
#include <windows.h>
#include <commdlg.h>
#include <string>
#include "IppDib.hpp"

#pragma comment(lib, "Comdlg32.lib")

// -------------------- 전역 상태 --------------------
static IppDib   g_img;
static HWND     g_hWnd = nullptr;
static std::wstring g_title = L"ImageTool";

// -------------------- 유틸 --------------------
static void SetWindowClientSize(HWND hWnd, int clientW, int clientH) {
    RECT rc = {0,0, clientW, clientH};
    DWORD style = (DWORD)GetWindowLongPtr(hWnd, GWL_STYLE);
    DWORD ex    = (DWORD)GetWindowLongPtr(hWnd, GWL_EXSTYLE);
    AdjustWindowRectEx(&rc, style, TRUE, ex);
    int winW = rc.right - rc.left;
    int winH = rc.bottom - rc.top;
    SetWindowPos(hWnd, nullptr, 0,0, winW, winH, SWP_NOMOVE|SWP_NOZORDER);
}

static bool OpenBmpDialog(HWND hWnd, std::wstring& outPath) {
    wchar_t fileName[MAX_PATH] = L"";
    OPENFILENAMEW ofn{ sizeof(ofn) };
    ofn.hwndOwner = hWnd;
    ofn.lpstrFilter = L"BMP Files (*.bmp)\0*.bmp\0All Files (*.*)\0*.*\0";
    ofn.lpstrFile   = fileName;
    ofn.nMaxFile    = MAX_PATH;
    ofn.lpstrTitle  = L"열기";
    ofn.Flags       = OFN_FILEMUSTEXIST | OFN_PATHMUSTEXIST;
    if (GetOpenFileNameW(&ofn)) {
        outPath = fileName;
        return true;
    }
    return false;
}

static bool SaveBmpDialog(HWND hWnd, std::wstring& outPath) {
    wchar_t fileName[MAX_PATH] = L"image.bmp";
    OPENFILENAMEW ofn{ sizeof(ofn) };
    ofn.hwndOwner = hWnd;
    ofn.lpstrFilter = L"BMP Files (*.bmp)\0*.bmp\0All Files (*.*)\0*.*\0";
    ofn.lpstrFile   = fileName;
    ofn.nMaxFile    = MAX_PATH;
    ofn.lpstrTitle  = L"다른 이름으로 저장";
    ofn.Flags       = OFN_OVERWRITEPROMPT | OFN_PATHMUSTEXIST;
    if (GetSaveFileNameW(&ofn)) {
        outPath = fileName;
        return true;
    }
    return false;
}

static void UpdateWindowTitle(HWND hWnd, const std::wstring& path) {
    std::wstring name = path.empty()? L"(새 이미지)" : path.substr(path.find_last_of(L"\\/")+1);
    std::wstring title = g_title + L" - " + name +
        (g_img.valid()? L"  [" + std::to_wstring(g_img.width) + L"x" + std::to_wstring(g_img.height) + L" " + std::to_wstring(g_img.bpp) + L"bpp]" : L"");
    SetWindowTextW(hWnd, title.c_str());
}

static void PaintHatchedBackground(HDC hdc, const RECT& rcClient, const RECT& rcImage) {
    // rcClient에서 rcImage를 제외한 4개 영역을 해칭
    HBRUSH hatch = CreateHatchBrush(HS_BDIAGONAL, RGB(180,180,180));
    COLORREF oldBk = SetBkColor(hdc, RGB(245,245,245));
    HBRUSH old = (HBRUSH)SelectObject(hdc, hatch);

    // 위
    if (rcImage.top > rcClient.top) {
        RECT r = { rcClient.left, rcClient.top, rcClient.right, rcImage.top };
        PatBlt(hdc, r.left, r.top, r.right-r.left, r.bottom-r.top, PATCOPY);
    }
    // 아래
    if (rcImage.bottom < rcClient.bottom) {
        RECT r = { rcClient.left, rcImage.bottom, rcClient.right, rcClient.bottom };
        PatBlt(hdc, r.left, r.top, r.right-r.left, r.bottom-r.top, PATCOPY);
    }
    // 좌
    if (rcImage.left > rcClient.left) {
        RECT r = { rcClient.left, rcImage.top, rcImage.left, rcImage.bottom };
        PatBlt(hdc, r.left, r.top, r.right-r.left, r.bottom-r.top, PATCOPY);
    }
    // 우
    if (rcImage.right < rcClient.right) {
        RECT r = { rcImage.right, rcImage.top, rcClient.right, rcImage.bottom };
        PatBlt(hdc, r.left, r.top, r.right-r.left, r.bottom-r.top, PATCOPY);
    }

    SelectObject(hdc, old);
    SetBkColor(hdc, oldBk);
    DeleteObject(hatch);
}

// -------------------- 메뉴/커맨드 --------------------
enum : UINT {
    ID_FILE_OPEN  = 1001,
    ID_FILE_SAVEAS,
    ID_FILE_EXIT
};

static HMENU BuildMenuBar() {
    HMENU hMenuBar = CreateMenu();
    HMENU hFile = CreatePopupMenu();
    AppendMenuW(hFile, MF_STRING, ID_FILE_OPEN,  L"열기(&O)...");
    AppendMenuW(hFile, MF_STRING, ID_FILE_SAVEAS, L"다른 이름으로 저장(&S)...");
    AppendMenuW(hFile, MF_SEPARATOR, 0, nullptr);
    AppendMenuW(hFile, MF_STRING, ID_FILE_EXIT,  L"끝내기(&X)");
    AppendMenuW(hMenuBar, MF_POPUP, (UINT_PTR)hFile, L"파일(&F)");
    return hMenuBar;
}

// -------------------- 윈도우 프로시저 --------------------
static std::wstring g_currentPath;

LRESULT CALLBACK WndProc(HWND hWnd, UINT msg, WPARAM wParam, LPARAM lParam) {
    switch (msg) {
    case WM_CREATE: {
        HMENU hMenu = BuildMenuBar();
        SetMenu(hWnd, hMenu);
        UpdateWindowTitle(hWnd, L"");
        return 0;
    }
    case WM_COMMAND: {
        switch (LOWORD(wParam)) {
        case ID_FILE_OPEN: {
            std::wstring path;
            if (OpenBmpDialog(hWnd, path)) {
                if (!g_img.LoadBmp(path)) {
                    MessageBoxW(hWnd, L"BMP 로드 실패(무압축 8/24bpp만 지원).", L"오류", MB_ICONERROR);
                    break;
                }
                g_currentPath = path;
                UpdateWindowTitle(hWnd, path);
                // 창 크기를 영상 크기에 맞추기
                SetWindowClientSize(hWnd, g_img.width, g_img.height);
                InvalidateRect(hWnd, nullptr, TRUE);
            }
        } break;
        case ID_FILE_SAVEAS: {
            if (!g_img.valid()) { MessageBoxW(hWnd, L"저장할 이미지가 없습니다.", L"안내", MB_OK|MB_ICONINFORMATION); break; }
            std::wstring path;
            if (SaveBmpDialog(hWnd, path)) {
                if (!g_img.SaveBmp(path)) {
                    MessageBoxW(hWnd, L"BMP 저장 실패.", L"오류", MB_ICONERROR);
                } else {
                    g_currentPath = path;
                    UpdateWindowTitle(hWnd, path);
                }
            }
        } break;
        case ID_FILE_EXIT:
            PostMessageW(hWnd, WM_CLOSE, 0, 0);
            break;
        }
        return 0;
    }
    case WM_PAINT: {
        PAINTSTRUCT ps{};
        HDC hdc = BeginPaint(hWnd, &ps);

        RECT rc; GetClientRect(hWnd, &rc);
        HBRUSH bg = CreateSolidBrush(RGB(245,245,245));
        FillRect(hdc, &rc, bg);
        DeleteObject(bg);

        if (g_img.valid()) {
            // 중앙 배치
            int imgW = g_img.width, imgH = g_img.height;
            int clientW = rc.right - rc.left;
            int clientH = rc.bottom - rc.top;
            int x = (clientW > imgW) ? (rc.left + (clientW - imgW)/2) : rc.left;
            int y = (clientH > imgH) ? (rc.top  + (clientH - imgH)/2) : rc.top;

            RECT rcImg = { x, y, x+imgW, y+imgH };
            // 영상 바깥 영역에 해칭
            PaintHatchedBackground(hdc, rc, rcImg);
            // 1:1 출력
            g_img.DrawAt(hdc, x, y);
            // 테두리
            FrameRect(hdc, &rcImg, (HBRUSH)GetStockObject(BLACK_BRUSH));
        } else {
            const wchar_t* tip = L"[파일] → [열기]로 BMP(무압축 8/24bpp)를 띄워보세요.";
            TextOutW(hdc, 10, 10, tip, lstrlenW(tip));
        }

        EndPaint(hWnd, &ps);
        return 0;
    }
    case WM_DESTROY:
        PostQuitMessage(0); return 0;
    }
    return DefWindowProcW(hWnd, msg, wParam, lParam);
}

// -------------------- WinMain --------------------
int APIENTRY wWinMain(HINSTANCE hInst, HINSTANCE, LPWSTR, int nCmdShow) {
    const wchar_t* cls = L"ImageToolWindow";

    WNDCLASSW wc{};
    wc.style         = CS_HREDRAW | CS_VREDRAW;
    wc.lpfnWndProc   = WndProc;
    wc.hInstance     = hInst;
    wc.hCursor       = LoadCursor(nullptr, IDC_ARROW);
    wc.hbrBackground = (HBRUSH)(COLOR_WINDOW+1);
    wc.lpszClassName = cls;
    if (!RegisterClassW(&wc)) return 0;

    // 창은 일단 "숨김"으로 생성 → 파일을 성공적으로 로드하면 보여줌
    g_hWnd = CreateWindowW(cls, g_title.c_str(),
                           WS_OVERLAPPEDWINDOW,
                           CW_USEDEFAULT, CW_USEDEFAULT, 800, 600,
                           nullptr, nullptr, hInst, nullptr);
    if (!g_hWnd) return 0;

    // 프로그램 구동 시 빈 창 띄우지 않기 → 먼저 '열기' 시도
    std::wstring path;
    if (OpenBmpDialog(g_hWnd, path)) {
        if (g_img.LoadBmp(path)) {
            g_currentPath = path;
            UpdateWindowTitle(g_hWnd, path);
            // 창 크기를 영상 크기에 맞춤
            SetWindowClientSize(g_hWnd, g_img.width, g_img.height);
            ShowWindow(g_hWnd, nCmdShow);
            UpdateWindow(g_hWnd);
        } else {
            MessageBoxW(g_hWnd, L"BMP 로드 실패(무압축 8/24bpp만 지원).", L"오류", MB_ICONERROR);
            DestroyWindow(g_hWnd);
            return 0;
        }
    } else {
        // 파일을 고르지 않으면 바로 종료(빈 창 표시하지 않음)
        DestroyWindow(g_hWnd);
        return 0;
    }

    MSG m;
    while (GetMessageW(&m, nullptr, 0, 0)) {
        TranslateMessage(&m);
        DispatchMessageW(&m);
    }
    return (int)m.wParam;
}
```

---

## 4) 동작 확인 체크리스트

1. **실행 즉시** 파일 열기 대화상자 표시 → **BMP 선택**  
   - 취소하면 **종료**(빈 창 없음)  
   - 선택 시 **창 표시 + 크기 자동 맞춤**
2. 창을 **키워 보세요** → 영상 바깥 영역이 **비스듬 해칭(HS_BDIAGONAL)** 으로 칠해집니다.
3. **파일 → 다른 이름으로 저장** → 저장 파일을 아무 뷰어로 열어 정상 표시되는지 확인  
4. **8bpp(그레이) / 24bpp(컬러)** BMP 모두 테스트  
   - 8bpp: 팔레트 256개(R=G=B)  
   - 24bpp: 팔레트 없음(BGR 순서)

---

## 5) 구현 포인트(생략 없이 요약)

- **Top→Down 내부표현**: 그리기 시 `biHeight = -height`(음수)로 전달 → `StretchDIBits`가 **뒤집지 않고** 그려줌.  
  저장 시엔 **Bottom→Up**으로 뒤집어 씁니다(전통 BMP 호환).
- **stride(4바이트 정렬)**: `((w*bpp + 31)/32)*4`  
- **팔레트**: 8bpp는 `RGBQUAD[256]`(R=G=B=i) 유지  
- **빈 창 방지**: `CreateWindow`만 먼저, **ShowWindow 전**에 파일 열기→성공 시에만 `ShowWindow`  
- **창 크기=영상 크기**: `AdjustWindowRectEx`로 **클라이언트 영역**을 정확히 맞춤  
- **해칭 영역**: 이미지 **바깥 4면**을 계산해 `CreateHatchBrush + PatBlt`로 채움  
- **메뉴/대화상자**: 리소스 없이 **코드에서 동적 생성**(휴대성↑)

---

## 6) 다음 확장 아이디어

- **스케일(확대/축소) 보기** 옵션(Nearest/Bilinear)  
- **드래그&드롭** 파일 열기, MRU(최근 파일)  
- **8bpp↔24bpp 변환**, **히스토그램 뷰**, **간단 필터**(블러/샤프)  
- **IppImage**와의 변환(알고리즘 파트 붙이기)
