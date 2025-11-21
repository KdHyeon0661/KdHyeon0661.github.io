---
layout: post
title: MFC - DPI-Aware, 다국어·현지화, 배포
date: 2025-09-11 17:25:23 +0900
category: MFC
---
# · 배포(재배포 패키지, 매니페스트/UAC, 코드 서명, WiX/Inno 팁) 완전 정리

**대상**: Win32/MFC(Feature Pack 포함), x64, Windows 10/11
**목표**:
1) 고DPI에서 **또렷하고 정확한 레이아웃**
2) 다국어 지원을 **안전하고 유지보수 가능**하게
3) **배포/업데이트/서명/UAC**까지 한 번에 설계

---

## DPI-Aware: Per-Monitor v2까지 제대로

### 1-1. 핵심 개념

- **DPI 가상화(DPI Virtualization)**: DPI-미인지 앱은 OS가 강제로 확대(블러) → 글자/아이콘 번짐.
- **DPI Awareness** 단계
  - `DPI Unaware` (기본): 흐릿함
  - `System-DPI Aware`: 로그인 당시 DPI만 인지(다른 모니터 이동 시 크기 불일치)
  - `Per-Monitor Aware (v1)`: 모니터별 DPI 대응(이동 시 `WM_DPICHANGED`)
  - **Per-Monitor v2**: 비클라이언트 영역(제목표시줄·메뉴·툴팁 등)까지 정교 지원

**정답**: **Per-Monitor v2**로 선언하고, **WM_DPICHANGED**에서 **크기/이미지/폰트**를 재계산.

---

### 1-2. 매니페스트로 DPI 선언 (권장)

```xml
<!-- app.manifest -->
<?xml version="1.0" encoding="utf-8"?>
<assembly xmlns="urn:schemas-microsoft-com:asm.v1" manifestVersion="1.0">
  <assemblyIdentity version="1.0.0.0" processorArchitecture="*" name="Vendor.App" type="win32"/>
  <dependency>
    <dependentAssembly>
      <assemblyIdentity type="win32" name="Microsoft.Windows.Common-Controls"
                        version="6.0.0.0" processorArchitecture="*"
                        publicKeyToken="6595b64144ccf1df" language="*"/>
    </dependentAssembly>
  </dependency>

  <!-- DPI Awareness (Per-Monitor v2) -->
  <application xmlns="urn:schemas-microsoft-com:asm.v3">
    <windowsSettings>
      <dpiAware xmlns="http://schemas.microsoft.com/SMI/2005/WindowsSettings">PerMonitorV2</dpiAware>
      <dpiAwareness xmlns="http://schemas.microsoft.com/SMI/2016/WindowsSettings">PerMonitorV2</dpiAwareness>
    </windowsSettings>
  </application>

  <!-- UAC는 3-1절 참고 -->
</assembly>
```

> Visual Studio: **프로젝트 → 링커 → 매니페스트 파일** 활성화 + 위 내용을 **app.manifest**로 포함.

---

### 1-3. 런타임 보강(하위호환/조건부)

```cpp
// 초기화 시점(WinMain/InitInstance 초반)
typedef DPI_AWARENESS_CONTEXT (WINAPI *PSetCtx)(DPI_AWARENESS_CONTEXT);
auto pSet = (PSetCtx)::GetProcAddress(::GetModuleHandleW(L"user32"), "SetThreadDpiAwarenessContext");
if (pSet) {
    pSet(DPI_AWARENESS_CONTEXT_PER_MONITOR_AWARE_V2); // 실패해도 기존 매니페스트로 OK
}
```

---

### 1-4. DPI 도우미: 스케일 함수 & DPI 질의

```cpp
// 공통 스케일 함수(정수 안전한 MulDiv 사용)
inline int ScaleByDpi(int logical, UINT dpi /*e.g., 96,120,144*/) {
    return ::MulDiv(logical, (int)dpi, 96);
}

inline UINT GetDpiForWindowSafe(HWND h) {
    HMODULE u32 = ::GetModuleHandleW(L"user32");
    auto pGet = (UINT (WINAPI*)(HWND))::GetProcAddress(u32, "GetDpiForWindow");
    if (pGet) return pGet(h);
    HDC hdc = ::GetDC(h);
    UINT dpi = ::GetDeviceCaps(hdc, LOGPIXELSX);
    ::ReleaseDC(h, hdc);
    return dpi ? dpi : 96;
}
```

---

### 1-5. `WM_DPICHANGED` 처리: 크기·폰트·이미지 리프레시

```cpp
LRESULT CMainFrame::OnDpiChanged(WPARAM w, LPARAM l) {
    UINT dpiNew = LOWORD(w);
    // 1) 추천 직사각형으로 윈도우 움직이기
    RECT* prcNew = (RECT*)l;
    ::SetWindowPos(m_hWnd, nullptr,
                   prcNew->left, prcNew->top,
                   prcNew->right - prcNew->left,
                   prcNew->bottom - prcNew->top,
                   SWP_NOZORDER | SWP_NOACTIVATE);

    // 2) 폰트/이미지/컨트롤 재스케일
    RecreateScaledFonts(dpiNew);     // 포인트→픽셀 변환 재계산
    ResizeControlsForDpi(dpiNew);    // 위치/크기 조정(Anchor 패턴과 조합)
    ReloadScaledImages(dpiNew);      // 아이콘/비트맵 교체(16/24/32/48/64px 등 준비)

    Invalidate(FALSE);
    return 0;
}
```

**폰트 스케일**(포인트→픽셀): `height_px = -MulDiv(point, dpi, 72)`로 `LOGFONT.lfHeight` 세팅.

---

### 1-6. 아이콘/이미지 스케일 전략

- **Multi-size ICO**(16/20/24/32/48/64/256) → `LoadIconMetric`(가능 시)
- `ImageList`는 `ImageList_SetIconSize`로 DPI당 크기 조정
- PNG/비트맵은 **여러 해상도 버전**을 리소스로 포함하거나 **WIC 스케일** 사용

```cpp
CSize IconSizeForDpi(UINT dpi) {
    int px = ScaleByDpi(16, dpi);
    if      (px <= 16) return {16,16};
    else if (px <= 20) return {20,20};
    else if (px <= 24) return {24,24};
    else if (px <= 32) return {32,32};
    else               return {48,48};
}
```

---

### 1-7. 대화상자 레이아웃(DLU→픽셀)와 DPI

- RC 대화상자 단위는 **DLU(Dialog Unit)** — 폰트 기준 상대 단위
- 실제 픽셀은 `MapDialogRect`로 변환 (폰트 바뀌면 DLU→픽셀도 달라짐)
- 고DPI에서 컨트롤 간격/폭은 **스케일** + **자동 줄바꿈 고려**

```cpp
CRect DluToPx(CWnd* dlg, int x, int y, int w, int h) {
    RECT rc = {x, y, x+w, y+h};
    dlg->MapDialogRect(&rc);
    return rc;
}
```

---

### 1-8. 체크리스트(DPI)

- [ ] 매니페스트에 **PerMonitorV2**
- [ ] `WM_DPICHANGED` 처리(추천 rect 적용)
- [ ] 폰트/아이콘/이미지 **재생성**
- [ ] `OnEraseBkgnd` 차단 + 더블 버퍼
- [ ] `GetDpiForWindow` 기반 **정수 스케일**
- [ ] 멀티 모니터 **이동 테스트** (125%↔200%)

---

## 다국어/현지화: 리소스 전환 전략

### 2-1. 기본 원칙

- 모든 UI 문자열은 **리소스(STRINGTABLE/RC/Ribbon XML)**로 분리
- **리소스 전용 DLL(위성 DLL)**로 **언어별 분리**(앱 본체는 중립)
- 앱 시작 시 **사용자 UI 언어**를 탐지 → 해당 언어 **위성 DLL 로드**

---

### 구조

```
App.exe                 // 중립(영문) 리소스 포함해도 OK
ko-KR\App.resources.dll // 리소스 전용 DLL (LANG_KOREAN, SUBLANG_KOREAN)
ja-JP\App.resources.dll
zh-CN\App.resources.dll
```

**생성 방법(요약)**
- MFC/Win32 프로젝트를 **리소스 전용 DLL**로 설정(`/NOENTRY`)
- `LANGUAGE LANG_KOREAN, SUBLANG_KOREAN`
- 문자열/대화상자/메뉴 등을 해당 언어로 번역하여 포함

---

### 2-3. 런타임 리소스 전환(MFC)

```cpp
// 1) 기본 리소스 핸들은 exe
HMODULE g_hResExe = AfxGetResourceHandle();
HMODULE g_hResLoc = nullptr;

bool LoadLangResource(LPCWSTR dllPath) {
    HMODULE h = ::LoadLibraryExW(dllPath, nullptr, LOAD_LIBRARY_AS_DATAFILE_EXCLUSIVE);
    if (!h) return false;
    g_hResLoc = h;
    AfxSetResourceHandle(g_hResLoc); // 이후부터 리소스 로드 시 여기서 찾음
    return true;
}

void RestoreNeutralResource() {
    if (g_hResLoc) { ::FreeLibrary(g_hResLoc); g_hResLoc = nullptr; }
    AfxSetResourceHandle(g_hResExe);
}
```

> **동적 전환**(설정에서 언어 바꾸기) 시: **창/리본 재생성** 또는 **문자열 재바인딩** 필요.

---

### 2-4. UI 언어 탐지 & Fallback

```cpp
LANGID ui = ::GetUserDefaultUILanguage(); // 예: 0x0412(ko-KR)
WCHAR tag[16]{}; ::GetLocaleInfoW(ui, LOCALE_SNAME, tag, 16); // "ko-KR"

// 폴더 존재 체크 후 ko-KR → ko → en-US 순으로 폴백
```

---

### 2-5. RC/리소스 작성 팁

```rc
// resource_ko.rc
LANGUAGE LANG_KOREAN, SUBLANG_KOREAN
STRINGTABLE
BEGIN
    IDS_APP_TITLE       "사진 도구"
    IDS_FILE_OPEN       "열기(&O)…"
    IDS_SAVE            "저장(&S)"
END

// resource_en.rc
LANGUAGE LANG_ENGLISH, SUBLANG_ENGLISH_US
STRINGTABLE
BEGIN
    IDS_APP_TITLE       "Photo Tool"
    IDS_FILE_OPEN       "&Open…"
    IDS_SAVE            "&Save"
END
```

**리본/툴팁**: MFC 리본은 **리본 리소스(XML)** 내 텍스트도 현지화 필요 → 각 언어 리소스에 **XML별 사본** 제공.

---

### 2-6. 날짜/숫자/통화/서식

- 포맷은 **사용자 로캘**에 맞춰 `GetDateFormatEx / GetTimeFormatEx / GetNumberFormatEx` 사용
- 문자열 비교/정렬: `CompareStringEx`, `LCMAP` 계열(컬레이션 주의)

```cpp
CStringW FormatDate(const SYSTEMTIME& st) {
    WCHAR buf[64];
    GetDateFormatEx(LOCALE_NAME_USER_DEFAULT, 0, &st, L"yyyy-MM-dd", buf, 64, nullptr);
    return buf;
}
```

---

### 2-7. 유니코드/입력기/우측→좌측(RTL)

- 프로젝트는 **유니코드(UTF-16)**
- **RTL** 언어(아랍어/히브리어)는 `WS_EX_LAYOUTRTL` + `SetProcessDefaultLayout` 고려
- 텍스트 렌더링은 **GDI+ 또는 DirectWrite** 권장(복잡 스크립트/단축키 병기)

---

### 2-8. 체크리스트(현지화)

- [ ] 모든 문자열 **리소스화**(하드코딩 금지)
- [ ] 언어별 **리소스 전용 DLL** 구조
- [ ] 앱 시작 시 **UI 언어 탐지 → 위성 DLL 로드**
- [ ] 날짜/숫자 포맷 **로캘 API** 사용
- [ ] 리본/툴팁/가속키 문자열도 번역
- [ ] 동적 전환 시 **윈도우 재생성/문자열 리바인드**

---

## 배포: 재배포 패키지, 매니페스트/UAC, 코드 서명, WiX/Inno

### 3-1. 매니페스트/UAC

- **UAC 실행 권한**:
  - `asInvoker`(기본) – 일반 권한
  - `highestAvailable` – 관리자면 관리자, 아니면 일반
  - `requireAdministrator` – 항상 관리자(설치/서비스 관리용)

```xml
<!-- UAC (app.manifest 안에) -->
<trustInfo xmlns="urn:schemas-microsoft-com:asm.v3">
  <security>
    <requestedPrivileges>
      <requestedExecutionLevel level="asInvoker" uiAccess="false"/>
    </requestedPrivileges>
  </security>
</trustInfo>
```

> **중요**: 관리자 권한이 필요한 작업은 **설치기/서비스/도우미**로 분리. 일반 실행은 `asInvoker` 유지가 UX/보안 모두 좋습니다.

---

### 3-2. Visual C++ 재배포 패키지(VC++ Redist)

- CRT/UCRT/ConCRT/MFC 사용 시 대상 PC에 **vcredist** 필요
- 배포 방식
  1) **전제 조건(Prerequisite)**: 설치기의 **체크 후 설치**(권장)
  2) 앱 폴더에 **로컬 배포**(일부 제한/권장 아님)
- WiX/Inno에서 **런타임 버전**에 맞는 vcredist를 **사일런트 설치**하도록 구성

**Inno 예시(다운로드/동봉)**
```pascal
[Run]
; 64비트 예: Visual C++ 2015-2022 x64
Filename: "{tmp}\VC_redist.x64.exe"; Parameters: "/install /quiet /norestart";
StatusMsg: "Microsoft Visual C++ 재배포 패키지 설치 중...";
Flags: waituntilterminated runhidden
```

**WiX Burn(Bootstrapper)로 번들**: 아래 3-5절.

---

### 3-3. 코드 서명(Code Signing)

- **목적**: SmartScreen 경고 완화, 무결성/신뢰 확보
- 인증서: **기업용 OV** 또는 **EV 코드서명**(EV가 SmartScreen 신뢰 축적 빠름)
- **시간 스탬프(TSA)** 필수(인증서 만료 이후에도 서명 유효)

**signtool 예시**
```bat
:: SHA-256, 타임스탬프 포함
signtool sign /fd SHA256 /tr http://timestamp.digicert.com /td SHA256 ^
    /a "bin\Release\App.exe"

signtool sign /fd SHA256 /tr http://timestamp.digicert.com /td SHA256 ^
    /a "setup\Installer.exe"
```

> **권장**: 빌드 파이프라인(CI)에서 **자동 서명**.
> MSIX/스토어 배포가 아니면 **서명+스마트스크린 평판**이 사용자 경험에 매우 중요.

---

### 3-4. 설치기: Inno Setup(스크립트 간단·강력)

**기본 스크립트**
```pascal
; MyApp.iss
[Setup]
AppName=Photo Tool
AppVersion=1.2.3
DefaultDirName={pf}\Vendor\PhotoTool
DefaultGroupName=Photo Tool
OutputBaseFilename=PhotoToolSetup
ArchitecturesInstallIn64BitMode=x64
PrivilegesRequired=admin
Compression=lzma
SolidCompression=yes
SetupIconFile=setup.ico

[Languages]
Name: "korean"; MessagesFile: "compiler:Languages\Korean.isl"
Name: "english"; MessagesFile: "compiler:Default.isl"

[Files]
Source: "bin\x64\Release\App.exe"; DestDir: "{app}"; Flags: ignoreversion
Source: "bin\x64\Release\ko-KR\App.resources.dll"; DestDir: "{app}\ko-KR"; Flags: ignoreversion
; vcredist 동봉 시
Source: "redist\VC_redist.x64.exe"; DestDir: "{tmp}"

[Icons]
Name: "{group}\Photo Tool"; Filename: "{app}\App.exe"
Name: "{commondesktop}\Photo Tool"; Filename: "{app}\App.exe"; Tasks: desktopicon

[Tasks]
Name: "desktopicon"; Description: "바탕화면 바로가기 만들기"; GroupDescription: "추가 작업"

[Run]
; VC++ redist (필요 시)
Filename: "{tmp}\VC_redist.x64.exe"; Parameters: "/install /quiet /norestart";
Flags: waituntilterminated runhidden
; 앱 실행
Filename: "{app}\App.exe"; Description: "실행"; Flags: nowait postinstall skipifsilent
```

**서명 연동**
- `SignTool` 파라미터를 Inno `SignTool` 옵션에 등록하거나 **사후 빌드 배치**에서 서명.

---

### 3-5. 설치기: WiX Toolset(MSI, 기업/정책 친화)

**제품 정의(.wxs) 최소 예**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<Wix xmlns="http://schemas.microsoft.com/wix/2006/wi">
  <Product Id="*" Name="Photo Tool" Language="1033" Version="1.2.3" Manufacturer="Vendor"
           UpgradeCode="{A1234567-89AB-4CDE-ABCD-0123456789AB}">
    <Package InstallerVersion="500" Compressed="yes" InstallScope="perMachine" />

    <MajorUpgrade DowngradeErrorMessage="이전 버전을 제거한 후 설치하세요." />
    <MediaTemplate />

    <Feature Id="Main" Level="1">
      <ComponentGroupRef Id="AppFiles" />
    </Feature>
  </Product>

  <Fragment>
    <Directory Id="TARGETDIR" Name="SourceDir">
      <Directory Id="ProgramFiles64Folder">
        <Directory Id="INSTALLDIR" Name="Photo Tool" />
      </Directory>
      <Directory Id="ProgramMenuFolder">
        <Directory Id="AppShortcutFolder" Name="Photo Tool" />
      </Directory>
    </Directory>
  </Fragment>

  <Fragment>
    <ComponentGroup Id="AppFiles" Directory="INSTALLDIR">
      <Component Id="cmp_AppExe" Guid="{B2345678-90AB-4CDE-BCDE-1234567890AB}">
        <File Id="fil_AppExe" Source="bin\x64\Release\App.exe" KeyPath="yes" />
      </Component>
      <Component Id="cmp_LangKo" Guid="{C2345678-90AB-4CDE-BCDE-1234567890AB}">
        <File Source="bin\x64\Release\ko-KR\App.resources.dll" />
        <CreateFolder />
        <RegistryValue Root="HKCU" Key="Software\Vendor\PhotoTool" Name="Installed" Type="integer" Value="1" KeyPath="yes"/>
      </Component>
      <Component Id="cmp_Shortcut" Guid="{D2345678-90AB-4CDE-BCDE-1234567890AB}">
        <Shortcut Id="ApplicationStartMenuShortcut" Directory="AppShortcutFolder" Name="Photo Tool"
                  WorkingDirectory="INSTALLDIR" Icon="app.ico" Target="[INSTALLDIR]App.exe" />
        <RemoveFolder Id="RemoveAppShortcutFolder" Directory="AppShortcutFolder" On="uninstall"/>
        <RegistryValue Root="HKCU" Key="Software\Vendor\PhotoTool" Name="HasShortcut" Type="integer" Value="1" KeyPath="yes"/>
      </Component>
    </ComponentGroup>
  </Fragment>
</Wix>
```

**Burn(부트스트래퍼)로 VC++ Redist 포함**
```xml
<Bundle Name="Photo Tool Bundle" Version="1.2.3" Manufacturer="Vendor" UpgradeCode="{E234...}">
  <BootstrapperApplicationRef Id="WixStandardBootstrapperApplication.HyperlinkLicense" />
  <Chain>
    <PackageGroupRef Id="VC140_Redist_x64" />
    <MsiPackage SourceFile="PhotoTool.msi" />
  </Chain>
</Bundle>
```

> WiX는 **기업 배포/그룹 정책/롤백/업그레이드** 관리에 유리. Inno는 **개발자 친화/경량**.

---

### 3-6. 파일 연결/컨텍스트 메뉴/방화벽(설치기에서)

**파일 연결(HKCR)**
- MSI: `ProgId` + `Extension` 요소
- Inno: `Assoc` 스크립트 또는 레지스트리

```xml
<!-- WiX: .ptl 확장자 연결 -->
<Fragment>
  <Component Id="cmp_Assoc" Guid="{A0B1...}" Directory="INSTALLDIR">
    <ProgId Id="Vendor.PhotoTool" Description="Photo Tool Document" Icon="app.ico" Advertise="yes">
      <Extension Id="ptl" ContentType="application/x-photo-tool" />
    </ProgId>
    <RegistryValue Root="HKCR" Key=".ptl" Value="Vendor.PhotoTool" Type="string" KeyPath="yes" />
  </Component>
</Fragment>
```

**방화벽 예**(Inno, netsh)
```pascal
[Run]
Filename: "{cmd}"; Parameters: "/c netsh advfirewall firewall add rule name=""PhotoTool"" dir=in action=allow program=""{app}\App.exe"" enable=yes"; Flags: runhidden
```

---

### 3-7. 업데이트/업그레이드 전략

- **버전/호환성 정책**: 데이터/설정 **마이그레이션**(파일 포맷/레지스트리 `ConfigVersion`)
- **자동 업데이트 채널**: 서명된 패키지 다운로드 + 무중단 업데이트(재시작 필요 시 안내)
- **WiX MajorUpgrade** 또는 Inno `AppVersion` 비교로 **자동 제거 후 설치**

---

### 3-8. 로그/문제 해결

- MSI: `msiexec /i PhotoTool.msi /l*v install.log`
- Inno: `/LOG="setup.log"`
- 설치 실패 시 **권한/존재 파일/잠금 프로세스**(Restart Manager) 확인

---

### 3-9. 배포 보안/법무 체크리스트

- [ ] **코드 서명 + TSA**
- [ ] 서드파티 라이선스/OSS 공지(about box/NOTICE)
- [ ] **개인정보/원격 전송**(크래시/사용 통계) 고지 및 옵트아웃
- [ ] 설치 경로: **Program Files**(Per-machine) 또는 **LocalAppData**(Per-user)
- [ ] 최소 권한 원칙(UAC `asInvoker`)
- [ ] 제거 시 **깨끗이 정리**(레지/캐시/서비스)

---

## 종합 샘플: “DPI-Aware + 다국어 + 설치/서명” 스캐폴딩

### 4-1. 프로젝트 구성

```
App.exe
app.manifest (PerMonitorV2 + asInvoker)
res\resource_en.rc
res\resource_ko.rc
ko-KR\App.resources.dll
ja-JP\App.resources.dll
installer\wix\PhotoTool.wxs
installer\inno\PhotoTool.iss
scripts\sign.bat
```

### 4-2. 초기화 코드 요약

```cpp
BOOL CMyApp::InitInstance() {
    // DPI
    auto pSet = (decltype(&SetThreadDpiAwarenessContext))GetProcAddress(GetModuleHandleW(L"user32"), "SetThreadDpiAwarenessContext");
    if (pSet) pSet(DPI_AWARENESS_CONTEXT_PER_MONITOR_AWARE_V2);

    // 언어: ko-KR 우선 로드 시도
    WCHAR lang[16]{}; GetLocaleInfoW(LOCALE_USER_DEFAULT, LOCALE_SNAME, lang, 16);
    CStringW dll; dll.Format(L"%s\\%s\\App.resources.dll", GetModuleDirectory(), lang);
    LoadLangResource(dll); // 실패 시 중립

    // MFC/Ribbon 초기화 등...
    return CWinAppEx::InitInstance();
}
```

### 4-3. WM_DPICHANGED 핸들러

```cpp
BEGIN_MESSAGE_MAP(CMainFrame, CFrameWndEx)
    ON_MESSAGE(WM_DPICHANGED, &CMainFrame::OnDpiChanged)
END_MESSAGE_MAP()
```

---

## 베스트 프랙티스 요약(한 장)

- **DPI**
  - Per-Monitor v2 선언 + `WM_DPICHANGED` 처리
  - 폰트/아이콘/이미지 재생성, DLU→픽셀 변환 주의
  - 멀티모니터에서 이격/잘림/블러 없는지 수동 테스트

- **현지화**
  - 리소스 전용 위성 DLL(+폴백)
  - 리본/툴팁/가속키까지 번역 세트
  - 날짜/숫자 포맷은 로캘 API

- **배포**
  - UAC `asInvoker`, 관리자 작업은 설치/서비스 분리
  - VC++ Redist 프리리퀴짓(버전 고정)
  - 코드 서명 + TSA + 자동화(CI)
  - WiX(기업/정책) 또는 Inno(간단/빠름)
  - 파일 연결/방화벽/서비스/업데이트까지 설계

---

## 부록: 실용 스니펫 모음

### 6-1. 현재 모니터 DPI로 픽셀 보정

```cpp
CRect ScaleRectByDpi(const CRect& rc, UINT dpi) {
    return CRect(ScaleByDpi(rc.left, dpi), ScaleByDpi(rc.top, dpi),
                 ScaleByDpi(rc.right, dpi), ScaleByDpi(rc.bottom, dpi));
}
```

### 6-2. 리소스 문자열 로딩(예외 안전)

```cpp
CStringW LoadStr(UINT id) {
    CStringW s; s.LoadStringW(id);
    return s;
}
```

### 6-3. 언어 전환(동적)

```cpp
bool SwitchLanguage(LPCWSTR tag /*ex: L"ja-JP"*/) {
    CStringW dll; dll.Format(L"%s\\%s\\App.resources.dll", GetModuleDirectory(), tag);
    RestoreNeutralResource(); // 이전 unload
    if (!LoadLangResource(dll)) return false;

    // 리본 재구성, 메뉴/대화상자 텍스트 갱신
    RebuildRibbon();
    RebindStrings();
    Invalidate(TRUE);
    return true;
}
```

### 6-4. signtool 자동화 배치

```bat
@echo off
set FILE=%1
signtool sign /fd SHA256 /tr http://timestamp.digicert.com /td SHA256 /a "%FILE%"
if %ERRORLEVEL% neq 0 exit /b %ERRORLEVEL%
echo Signed %FILE%
```

---

### 마무리

**DPI-Aware + 현지화 + 배포**는 앱의 **완성도**를 결정합니다.
Per-Monitor v2로 또렷한 UI, **위성 DLL**로 확장 가능한 현지화,
**서명/UAC/재배포/설치기**로 신뢰도 높은 배포 체인을 구축하세요.
