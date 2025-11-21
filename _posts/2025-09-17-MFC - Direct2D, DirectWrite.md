---
layout: post
title: MFC - Direct2D, DirectWrite
date: 2025-09-17 23:25:23 +0900
category: MFC
---
# Direct2D/DirectWrite 전환 가이드

**GDI 텍스트 → DirectWrite** 클리어타입 / 컬러 폰트·이모지 / OpenType 특성 / 복합 스크립트(Complex Script) 까지

> 목표: 기존 **GDI(+GDI+) 기반 텍스트 렌더링**을 **Direct2D + DirectWrite(DWrite)** 로 전환하면서 품질(클리어타입, 힌팅, 커닝), **컬러 폰트/이모지**, **OpenType 기능(합자·대체·스타일셋)**, **복합 스크립트(아랍어·힌디어·태국어 등)** 의 **정확한 조합/형상화(shaping)** 까지 모두 다룹니다.
> 예제는 **C++/Win32** 기준, **ComPtr(WRL)** 과 **스마트 포인터**로 구성합니다.

---

## 큰 그림: 왜 DirectWrite인가?

- **일관된 텍스트 엔진**: GDI의 `TextOut`/`ExtTextOut` 대비, **서브픽셀 클리어타입 렌더링**, **정확한 글립·커닝**, **스크립트 shaping**, **OpenType 전체 기능**을 **OS 표준 엔진**으로 제공합니다.
- **컬러 폰트/이모지**: DWrite 1.3+ (Win 8.1+)부터 COLR/CPAL, CBDT/CBLC, SVG-in-OpenType 등 **색상 글립** 표시를 지원합니다.
- **고 DPI·Per-Monitor-V2**와 **픽셀 스냅/미세 커닝**: 고해상도에서 또렷하고 왜곡 없는 텍스트.
- **Direct2D와 자연스러운 결합**: DX11/SwapChain·합성, 투명도·효과와의 조합, 벡터·비트맵과 동일 파이프라인.

---

## 아키텍처 개요

```
[App] ──▶ [D2D1Factory] ─┬─▶ [ID2D1Device] ─▶ [ID2D1DeviceContext] ─▶ DrawText/Layout
                         │
                         └─▶ [IDWriteFactory] ─▶ TextFormat / TextLayout / Typography / FontFallback
                                                     ▲
                                                     └── [FontCollection/Loader]
```

- **D2D**는 *그림을 그리는 컨텍스트*를 제공, **DWrite**는 *문자를 글립으로 분석/배치/형상화*합니다.
- **ID2D1DeviceContext::DrawText / DrawTextLayout** 이 **DWrite 레이아웃**을 실제 픽셀로 렌더링합니다.

---

## 기본 초기화: D2D/DWrite 팩토리, 디바이스 컨텍스트

### 2-1. 최소 초기화 (HWND Render Target, 쉬운 길)

가장 간단한 레거시 경로입니다. 새로운 앱이라면 *Device/DeviceContext* 경로를 권장하지만, 전환기에 부담이 적습니다.

```cpp
#include <d2d1.h>
#include <dwrite.h>
#include <wrl.h>

using Microsoft::WRL::ComPtr;

ComPtr<ID2D1Factory>        g_d2dFactory;
ComPtr<IDWriteFactory>       g_dwFactory;
ComPtr<ID2D1HwndRenderTarget> g_rt;
ComPtr<ID2D1SolidColorBrush> g_textBrush;

void InitD2D(HWND hWnd) {
    D2D1_FACTORY_OPTIONS opt{}; // 디버그시 D2D1_DEBUG_LEVEL_INFORMATION
    D2D1CreateFactory(D2D1_FACTORY_TYPE_SINGLE_THREADED, opt, &g_d2dFactory);

    DWriteCreateFactory(DWRITE_FACTORY_TYPE_SHARED, __uuidof(IDWriteFactory),
                        reinterpret_cast<IUnknown**>(g_dwFactory.GetAddressOf()));

    RECT rc{}; GetClientRect(hWnd, &rc);
    g_d2dFactory->CreateHwndRenderTarget(
        D2D1::RenderTargetProperties(
            D2D1_RENDER_TARGET_TYPE_DEFAULT,
            D2D1::PixelFormat(DXGI_FORMAT_UNKNOWN, D2D1_ALPHA_MODE_PREMULTIPLIED)),
        D2D1::HwndRenderTargetProperties(hWnd,
            D2D1::SizeU(rc.right-rc.left, rc.bottom-rc.top)),
        &g_rt);

    g_rt->CreateSolidColorBrush(D2D1::ColorF(D2D1::ColorF::White), &g_textBrush);
}
```

### 2-2. 권장 초기화 (D3D11 + DXGI + D2D DeviceContext)

- **고급 합성/효과**, **색상 글립(Enable Color Font 옵션)**, **하드웨어 가속**, **프레임 동기화** 등을 활용하기 좋은 경로.

```cpp
ComPtr<ID2D1Factory1>       d2dFactory;
ComPtr<IDWriteFactory2>     dwFactory;  // 1.2/1.3 이상이면 *2, *3, *5 등 최신 인터페이스 활용
ComPtr<ID2D1Device>         d2dDevice;
ComPtr<ID2D1DeviceContext>  d2dCtx;
// (생략: D3D11Device + DXGI SwapChain 생성)
void InitDeviceContextFromDXGI(IDXGIDevice* dxgiDevice) {
    D2D1CreateFactory(D2D1_FACTORY_TYPE_SINGLE_THREADED, d2dFactory.GetAddressOf());
    d2dFactory->CreateDevice(dxgiDevice, &d2dDevice);
    d2dDevice->CreateDeviceContext(D2D1_DEVICE_CONTEXT_OPTIONS_NONE, &d2dCtx);

    DWriteCreateFactory(DWRITE_FACTORY_TYPE_SHARED, __uuidof(IDWriteFactory2),
        reinterpret_cast<IUnknown**>(dwFactory.GetAddressOf()));

    d2dCtx->SetTextAntialiasMode(D2D1_TEXT_ANTIALIAS_MODE_CLEARTYPE);
}
```

> **버전 팁**
> - **DWrite 1.3+**: 컬러 폰트, `IDWriteFactory2/3` 인터페이스, `IDWriteFontFallback` 향상.
> - **Windows 10/11**: 최신 DWrite가 기본. 레거시 OS는 DWriteCore 또는 OS 조건부 기능 분기 고려.

---

## 텍스트 포맷과 레이아웃

### 3-1. `IDWriteTextFormat` – 글꼴/크기/정렬/줄 간격

```cpp
ComPtr<IDWriteTextFormat> fmt;
dwFactory->CreateTextFormat(
    L"Segoe UI",            // 폰트명(로컬/콜렉션)
    nullptr,                // 기본 폰트 컬렉션
    DWRITE_FONT_WEIGHT_NORMAL,
    DWRITE_FONT_STYLE_NORMAL,
    DWRITE_FONT_STRETCH_NORMAL,
    14.0f,                  // DIP 단위 포인트(1/96 inch)
    L"ko-KR",               // locale
    &fmt);

fmt->SetTextAlignment(DWRITE_TEXT_ALIGNMENT_LEADING);
fmt->SetParagraphAlignment(DWRITE_PARAGRAPH_ALIGNMENT_NEAR);
```

### 3-2. `IDWriteTextLayout` – 줄바꿈/커닝/형상화 결과(핵심)

`DrawText`는 내부적으로 레이아웃을 즉시 만들지만, **복잡한 텍스트(커스텀 기능, 히트테스트, 하이라이트)** 는 **TextLayout**을 캐시해 쓰세요.

```cpp
const wchar_t* text = L"Hello, DirectWrite! こんにちはمرحبا 😀";
ComPtr<IDWriteTextLayout> layout;
dwFactory->CreateTextLayout(text, (UINT32)wcslen(text), fmt.Get(),
                            800.0f /* layoutWidth */,
                            0.0f   /* layoutHeight: 0=auto */, &layout);
```

- **자동 줄바꿈/스크립트 분석/커닝/숫자 대체/서브픽셀 위치**가 레이아웃에 담깁니다.
- **텍스트 범위 속성 부여**: 색·폰트·특성(합자/스타일셋)·언어 범위를 지정해 **혼합 스타일**을 표현.

---

## 렌더링: 클리어타입/안티앨리어싱/픽셀 스냅

### 4-1. 텍스트 안티앨리어싱 모드

```cpp
// 그레이스케일(투명 배경 UI에 유리), 클리어타입(불투명 배경에 선명)
d2dCtx->SetTextAntialiasMode(D2D1_TEXT_ANTIALIAS_MODE_GRAYSCALE);
// or
d2dCtx->SetTextAntialiasMode(D2D1_TEXT_ANTIALIAS_MODE_CLEARTYPE);
```

### 4-2. 렌더링/측정 모드(클래식 vs 내추럴)

```cpp
DWRITE_RENDERING_MODE rm = DWRITE_RENDERING_MODE_NATURAL;      // LCD 서브픽셀 자연스러운 품질
DWRITE_MEASURING_MODE mm = DWRITE_MEASURING_MODE_NATURAL;      // 부동소수 커닝/측정

ComPtr<IDWriteRenderingParams> defParams;
dwFactory->CreateRenderingParams(&defParams);

ComPtr<IDWriteRenderingParams> custom;
dwFactory->CreateCustomRenderingParams(
    defParams->GetGamma(), defParams->GetEnhancedContrast(),
    defParams->GetClearTypeLevel(), rm, mm, &custom);

// HwndRenderTarget인 경우
g_rt->SetTextRenderingParams(custom.Get());

// DeviceContext인 경우
ComPtr<ID2D1DrawingStateBlock1> ds;
d2dFactory->CreateDrawingStateBlock(nullptr, &ds);
ds->SetTextRenderingParams(custom.Get());
d2dCtx->SetDrawingStateBlock(ds.Get());
```

> **픽셀 스냅**
> - 텍스트 **원점을 정수 픽셀**에 맞추면 가독성이 향상됩니다.
> - 단, DWrite는 부동소수 위치 커닝이 품질을 살리는 경우가 많아, **UI 요소(레이블)** 는 스냅, **장문/뷰어**는 자연 측정 권장.

---

## 그리기 예제: DrawText vs DrawTextLayout

```cpp
void RenderHello() {
    d2dCtx->BeginDraw();
    d2dCtx->Clear(D2D1::ColorF(0.1f,0.1f,0.12f));

    static const wchar_t* s = L"ClearType + Emoji 😀 + 아랍어 مرحبا + हिन्दी";
    D2D1_RECT_F rc = D2D1::RectF(20.f, 20.f, 760.f, 0.f);

    // 간단: DrawText (내부 레이아웃 생성)
    d2dCtx->DrawTextW(s, (UINT32)wcslen(s), fmt.Get(), rc, g_textBrush.Get(),
        D2D1_DRAW_TEXT_OPTIONS_ENABLE_COLOR_FONT, // 컬러 폰트 사용
        DWRITE_MEASURING_MODE_NATURAL);

    d2dCtx->EndDraw();
}
```

```cpp
void RenderWithLayout() {
    d2dCtx->BeginDraw();
    d2dCtx->Clear(D2D1::ColorF(0,0,0));

    // 사전 생성된 layout 사용
    d2dCtx->DrawTextLayout(
        D2D1::Point2F(20.f, 20.f),
        layout.Get(),
        g_textBrush.Get(),
        D2D1_DRAW_TEXT_OPTIONS_ENABLE_COLOR_FONT); // 😀/컬러 글립 표현

    d2dCtx->EndDraw();
}
```

> **옵션 팁**
> - `D2D1_DRAW_TEXT_OPTIONS_ENABLE_COLOR_FONT` 를 **반드시** 전달해야 컬러 폰트(이모지)가 실제 색상으로 렌더됩니다.
> - 배경이 반투명/합성일 때는 **GRAYSCALE AA** 가 더 보기 좋은 경우가 많습니다.

---

## & 폰트 폴백

### 6-1. 컬러 폰트 활성화 포인트

- Windows 8.1+ / DWrite 1.3+
- **렌더 옵션**: `D2D1_DRAW_TEXT_OPTIONS_ENABLE_COLOR_FONT`
- 폰트: **Segoe UI Emoji**, 컬러 레이어(COLR/CPAL), 비트맵(CBDT/CBLC), SVG-in-OT
- **투명 합성**: 그레이스케일 AA가 더 자연스럽기도 합니다.

```cpp
d2dCtx->SetTextAntialiasMode(D2D1_TEXT_ANTIALIAS_MODE_GRAYSCALE);
d2dCtx->DrawTextLayout(pt, layout.Get(), brush.Get(),
    D2D1_DRAW_TEXT_OPTIONS_ENABLE_COLOR_FONT);
```

### 6-2. 폰트 폴백(Font Fallback)

복합 문자열에서 **한 폰트로 모든 글립이 존재하지 않을 수 있음** → DWrite 폰트 폴백을 통해 **적절한 대체 폰트**가 자동 선택됩니다.

```cpp
ComPtr<IDWriteFontFallback> fallback;
ComPtr<IDWriteFactory2> dw2; dwFactory.As(&dw2);
dw2->GetSystemFontFallback(&fallback); // 시스템 기본 폴백

ComPtr<IDWriteTextLayout3> layout3;
layout.As(&layout3);
layout3->SetFontFallback(fallback.Get());
```

> **커스텀 폴백**
> - `IDWriteFontFallbackBuilder` 로 특정 유니코드 범위를 지정해 회사 전용 폰트 우선 사용 등도 가능합니다.

---

## OpenType 기능(합자, 대체 글립, 스타일셋, OldStyle 숫자 등)

### 7-1. `IDWriteTypography` 로 기능 주입

```cpp
ComPtr<IDWriteTypography> typo;
dwFactory->CreateTypography(&typo);

// 표준 합자(liga), 선택적 합자(dlig), 대체(ss01), OldStyle 숫자(onum) 등
DWRITE_FONT_FEATURE features[] = {
    { DWRITE_FONT_FEATURE_TAG_STANDARD_LIGATURES, 1 },   // liga
    { DWRITE_FONT_FEATURE_TAG_CONTEXTUAL_ALTERNATES, 1 },// calt
    { DWRITE_FONT_FEATURE_TAG_STYLISTIC_SET_1, 1 },      // ss01
    { DWRITE_FONT_FEATURE_TAG_OLD_STYLE_FIGURES, 1 },    // onum
};
for (auto& f : features) typo->AddFontFeature(f);

// 레이아웃 범위에 적용
DWRITE_TEXT_RANGE rng{ 0, (UINT32)wcslen(text) };
ComPtr<IDWriteTextLayout2> layout2; layout.As(&layout2);
layout2->SetTypography(typo.Get(), rng);
```

### 7-2. 커닝/합자/대체 효과 확인

- OpenType 지원 폰트(예: **Segoe UI**, **Bahnschrift**, **Source Serif** 등)를 사용하면 **fi, fl 합자**, **스타일셋** 등이 반영됩니다.
- **GDI**는 기본적으로 합자/커닝을 온전히 지원하지 않거나 제한적임 → **DWrite로 이관 시 즉시 품질 향상**.

---

## & BiDi

### 8-1. 언어/로케일 설정

- `CreateTextFormat` 의 **locale** (예: `ar-SA`, `he-IL`, `hi-IN`, `th-TH`) 를 정확히 지정하면 **숫자 대체/단어 단위 줄바꿈**이 향상됩니다.

```cpp
dwFactory->CreateTextFormat(L"Segoe UI", nullptr,
    DWRITE_FONT_WEIGHT_NORMAL, DWRITE_FONT_STYLE_NORMAL, DWRITE_FONT_STRETCH_NORMAL,
    16.0f, L"ar-SA", &fmt); // 아랍어
```

### 8-2. 숫자 대체/방향성

```cpp
ComPtr<IDWriteNumberSubstitution> numSubst;
dwFactory->CreateNumberSubstitution(
    DWRITE_NUMBER_SUBSTITUTION_METHOD_CONTEXTUAL, // locale 기반
    L"ar-SA", TRUE, &numSubst);

layout->SetNumberSubstitution(numSubst.Get(), DWRITE_TEXT_RANGE{0, len});
```

- **BiDi**(Mixed LTR/RTL)는 DWrite가 자동 분석합니다.
- 커서 이동/히트테스트는 `IDWriteTextLayout::HitTestPoint/HitTestTextPosition` 으로 **글립 클러스터 단위**로 정확히 처리됩니다.

---

## 히트 테스트, 캐럿, 선택(에디터 구현 핵심)

```cpp
DWRITE_HIT_TEST_METRICS m{};
BOOL isTrailing=FALSE, isInside=FALSE;
layout->HitTestPoint(x, y, &isTrailing, &isInside, &m);

// m.textPosition / length 로 캐럿 위치 결정
// selection: HitTestTextRange 로 사각형 목록 획득 → 커스텀 하이라이트 렌더
UINT32 count = 0;
layout->HitTestTextRange(selStart, selLength, xOffset, yOffset, nullptr, 0, &count);
std::vector<DWRITE_HIT_TEST_METRICS> rects(count);
layout->HitTestTextRange(selStart, selLength, xOffset, yOffset, rects.data(), count, &count);

// 각 rect에 대해 d2dCtx->FillRectangle(하이라이트 브러시)
```

> **GDI에서의 좌표 계산 난점**(UTF-16 surrogate pair, 조합문자, Complex Script)은 DWrite가 **클러스터 경계**로 해결합니다.

---

## 성능 전략: 캐싱/배치/Invalidation

- **TextFormat/Typo/FontFallback**: **재사용** (매 프레임 생성 금지).
- **TextLayout**: 문자열 변경이 없으면 **캐시**. 뷰포트 스크롤 시 **오프셋만 변경**.
- **그리기 배치**: `BeginDraw`/`EndDraw` 사이 **DrawTextLayout 다수 호출** OK.
- **장문 뷰어**: 페이징/가상화(화면에 보이는 레인지만 레이아웃 생성/갱신).

---

## DPI/픽셀 정렬/스냅

- D2D/DWrite는 `DIP(1/96inch)` 기준. Per-Monitor DPI에서 **자동 스케일**.
- **픽셀 스냅 팁**
  - 텍스트 앵커(좌측 위)를 `floor(x)+0.5f` 처럼 조정해 **수평 0.5 픽셀** 스냅을 쓰면 심미적으로 좋을 때가 많습니다(서브픽셀 AA 기준).
  - 라벨/UI 텍스트는 정수 스냅, 본문/장문은 자연 측정.

---

## GDI → DirectWrite 매핑 참고표

| GDI | DirectWrite/Direct2D |
|---|---|
| `HFONT` | `IDWriteTextFormat` (크기/가중/스타일/스트레치) |
| `TextOut/ExtTextOut` | `DrawText/DrawTextLayout` |
| `GetTextExtentPoint32` | `IDWriteTextLayout::DetermineMinWidth/HitTestTextRange` |
| `SetBkMode(TRANSPARENT)` | 기본 투명 합성(브러시 알파/AA 모드로 제어) |
| `SelectObject(hFont)` | `SetTextFormat` 또는 레이아웃 생성 시 지정 |
| 커닝/합자 제한 | **완전한 OpenType 지원** |
| BiDi/복합 스크립트 난점 | **자동 shaping + HitTest** |

---

## GDI+ → Direct2D 전환 메모

- 이미지/WIC 파이프라인은 그대로 사용 가능: `IWICBitmapDecoder` → `ID2D1Bitmap1`.
- 텍스트는 **DWrite가 정답**. GDI+의 `DrawString` 는 품질·기능 모두 열세.

---

## 샘플: 색 혼합 텍스트 + 범위별 스타일 + 컬러 이모지

```cpp
struct BrushPack {
    ComPtr<ID2D1SolidColorBrush> normal, accent, highlight;
} g;

void BuildBrushes() {
    d2dCtx->CreateSolidColorBrush(D2D1::ColorF(0xEEEEEE), &g.normal);
    d2dCtx->CreateSolidColorBrush(D2D1::ColorF(0x88C0D0), &g.accent);
    d2dCtx->CreateSolidColorBrush(D2D1::ColorF(0xEBCB8B, 0.5f), &g.highlight);
}

void DrawStyled() {
    const wchar_t* s = L"[Title] OpenType fi/fl 합자 + 😀 + العربية हिन्दी — ss01";
    ComPtr<IDWriteTextLayout> lay;
    dwFactory->CreateTextLayout(s, (UINT32)wcslen(s), fmt.Get(), 900.f, 0.f, &lay);

    // 범위별 색상/폰트 크기
    ComPtr<IDWriteTextLayout2> lay2; lay.As(&lay2);
    DWRITE_TEXT_RANGE title{0, 7};
    lay2->SetFontSize(24.f, title);

    // Typography: 합자/스타일셋
    ComPtr<IDWriteTypography> typo; dwFactory->CreateTypography(&typo);
    DWRITE_FONT_FEATURE feats[] = {
        { DWRITE_FONT_FEATURE_TAG_STANDARD_LIGATURES, 1 },
        { DWRITE_FONT_FEATURE_TAG_STYLISTIC_SET_1, 1 }
    };
    for (auto& f: feats) typo->AddFontFeature(f);
    DWRITE_TEXT_RANGE whole{0, (UINT32)wcslen(s)};
    lay2->SetTypography(typo.Get(), whole);

    // 배경 하이라이트 영역(임의 범위)
    DWRITE_TEXT_METRICS tm{}; lay->GetMetrics(&tm);
    UINT32 count=0;
    lay->HitTestTextRange(0, 7, 20.f, 20.f, nullptr, 0, &count);
    std::vector<DWRITE_HIT_TEST_METRICS> rects(count);
    lay->HitTestTextRange(0, 7, 20.f, 20.f, rects.data(), count, &count);

    d2dCtx->BeginDraw();
    d2dCtx->Clear(D2D1::ColorF(0x2E3440));

    // 하이라이트 배경
    for (auto& r: rects) {
        D2D1_RECT_F rf = D2D1::RectF(20.f + r.left, 20.f + r.top,
                                     20.f + r.left + r.width, 20.f + r.top + r.height);
        d2dCtx->FillRectangle(rf, g.highlight.Get());
    }

    // 텍스트(컬러 폰트 허용)
    d2dCtx->DrawTextLayout(D2D1::Point2F(20.f, 20.f), lay.Get(),
                           g.normal.Get(), D2D1_DRAW_TEXT_OPTIONS_ENABLE_COLOR_FONT);

    d2dCtx->EndDraw();
}
```

---

## 국제화 체크리스트

1. **Locale**: 포맷 생성 시 정확히 지정 (`ko-KR`, `en-US`, `ar-SA` …)
2. **Number substitution**: 로케일 문맥 기반 대체
3. **BiDi**: 오른쪽→왼쪽 혼합 텍스트 테스트(아랍어 + 영어)
4. **단어 분리/줄바꿈**: 일본어/중국어/태국어 줄바꿈 규칙 확인
5. **대체 폰트**: Emoji/한자 확장 영역(CJK Ext B~) 폴백 정상 작동 확인
6. **색 글립**: 시스템/테마(다크/라이트)에서 대비/가시성 재확인

---

## 트러블슈팅

| 증상 | 원인 | 해결 |
|---|---|---|
| 이모지가 흑백으로 뜸 | 컬러 폰트 옵션 미지정 | `DrawText/Layout` 호출에 `D2D1_DRAW_TEXT_OPTIONS_ENABLE_COLOR_FONT` 추가 |
| 가장자리 흐릿 | 알파 배경 + 클리어타입 | **GRAYSCALE AA** 로 변경 또는 불투명 배경 확보 |
| 커서 위치/선택 범위 틀림 | surrogate/조합 문자 미고려 | **HitTest APIs** 사용, UTF-16 코드 유닛이 아닌 **클러스터** 단위 처리 |
| 한 폰트로 일부 문자가 ■ | 폰트 폴백 미적용 | `IDWriteFontFallback` 설정(시스템 기본 or 커스텀) |
| DPI에서 크기 불일치 | DIP↔픽셀 혼동 | DIP 기반 좌표 사용, 필요 시 `SetDpi`/픽셀스냅 |
| GDI 대비 폭/줄바꿈 차이 | 측정·커닝 모드 차이 | DWrite **NATURAL** 모드 특성 이해, 필요 시 **GDI_CLASSIC** 렌더링 모드로 타협 |

---

## 단계별 전환 전략(레거시 앱)

1. **텍스트 렌더 경로만 대체**: GDI 배경/도형은 유지, **텍스트만 DrawTextLayout** 로 먼저 교체.
2. **포맷/레이아웃 캐시**: 컨트롤/뷰 단위로 `TextFormat`/`Typography`/`Layout` 객체 **수명 관리**.
3. **색 글립·OpenType**: 기능별 Feature Flag를 옵션화(설정 UI).
4. **국제화·BiDi**: 실제 데이터로 **자동 테스트**(줄바꿈/커서/선택).
5. **성능**: Invalidate 최소화, 스크롤 시 오프셋 이동 → 레이아웃 재생성은 내용 변경 시에만.
6. **DPI**: Per-Monitor-V2 매니페스트 후, 텍스트 DIP 기준 배치 재점검.

---

## 작은 레퍼런스(태그/상수)

- **AA 모드**: `D2D1_TEXT_ANTIALIAS_MODE_{DEFAULT, CLEARTYPE, GRAYSCALE, ALIASED}`
- **DWrite 렌더링 모드**: `DWRITE_RENDERING_MODE_{DEFAULT, NATURAL, NATURAL_SYMMETRIC, GDI_CLASSIC, OUTLINE}`
- **측정 모드**: `DWRITE_MEASURING_MODE_{NATURAL, GDI_CLASSIC, GDI_NATURAL}`
- **OpenType Feature Tags**(일부):
  - `liga`(표준 합자), `dlig`(선택 합자), `calt`(문맥 대체),
  - `onum`(OldStyle 숫자), `lnum`(Lining 숫자), `tnum`(Tabular), `pnum`(Proportional),
  - `ss01`~`ss20`(스타일셋), `kern`(커닝), `frac`(분수), `smcp`(스몰캡), `salt`(대체)
- **컬러 폰트 옵션**: `D2D1_DRAW_TEXT_OPTIONS_ENABLE_COLOR_FONT`
- **BiDi/숫자 대체**: `IDWriteNumberSubstitution`

---

## 보너스: GDI 텍스트와 시각 비교 샘플 (A/B 토글)

- 동일 UI에 **GDI(TextOut)** vs **DWrite(DrawTextLayout)** 를 **체크박스**로 교차 표시하여
  - 스템 두께, 커닝, 합자, 이모지, 아랍어 shaping, 고 DPI 선명도 차이를 **눈으로 확인** → 전환 설득 자료로 유용.

---

## 결론

- **DirectWrite** 는 **현대 윈도우 텍스트 렌더링의 표준 엔진**입니다.
- **클리어타입/그레이스케일 AA**, **컬러 폰트(이모지)**, **완전한 OpenType**, **복합 스크립트 shaping**, **정확한 히트테스트**까지 **GDI의 한계를 해소**합니다.
- 전환은 **텍스트 경로부터** 단계적으로: **포맷·레이아웃 캐시**, **AA/렌더 모드 선택**, **국제화·BiDi 테스트**를 체크리스트로 삼으세요.
- 한 번 품질을 맞춰두면, 고 DPI·다국어·현대 UI에서 **유지보수 비용이 급감**합니다.
