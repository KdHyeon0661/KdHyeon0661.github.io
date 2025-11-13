---
layout: post
title: 영상처리 - Dithering
date: 2025-10-18 17:25:23 +0900
category: 영상처리
---
# Dithering (RGB565 같은 **저비트 표면**을 위한 디더링 총정리)
> 목표
> - 24/32bpp(8bit/channel) 소스 이미지를 **RGB565(5-6-5)** 같은 저비트 표면으로 보낼 때, **계조 밴딩/블로킹**을 줄인다.
> - **Ordered(규칙) 디더링**과 **Error Diffusion(오차 확산)**(Floyd–Steinberg 등)의 C++ 실전 코드를 제공한다.
> - **감마/전달함수 정합**(sRGB↔선형)·알파 합성·성능 팁까지 “빠뜨림 없이” 설명한다.
> - 코드 블록은 한 번만 ` ``` `로 감싼다. 수학은 MathJax 사용.

---

## 1. 왜 디더링이 필요한가?
- RGB565는 **R=5bit(32단계), G=6bit(64단계), B=5bit(32단계)** 에 불과해, 8bit 채널 대비 **계조 단계가 크게 줄어** 밴딩/계단 현상이 나타난다.
- 디더링은 **양자화(quantization)** 로 생기는 오차를 **공간적으로 흩뿌려** 시각적으로 **균질한 노이즈**로 바꾸는 기술이다.
  - **Ordered(규칙) 디더링**: 작은 타일(예: 8×8 Bayer) 임계치 행렬로 **빠르고 일정한** 노이즈.
  - **Error Diffusion**: 픽셀 오차를 이웃으로 **전파**(Floyd–Steinberg, JJN, Stucki). **시각 품질↑**지만 비용↑/패턴 의존.

---

## 2. RGB565로의 양자화(기본식)
- R,G,B 원 신호를 \(r,g,b \in [0,1]\) 로 두고, 5/6bit에 매핑:
\[
R_5 = \left\lfloor 32 \cdot r + \tfrac{1}{2} \right\rfloor \;\;(\text{clamp }0..31), \quad
G_6 = \left\lfloor 64 \cdot g + \tfrac{1}{2} \right\rfloor \;\;(\text{clamp }0..63), \quad
B_5 = \left\lfloor 32 \cdot b + \tfrac{1}{2} \right\rfloor \;\;(\text{clamp }0..31)
\]
- 16비트 포맷으로 패킹:
\[
\text{RGB565} = (R_5 \ll 11) \;|\; (G_6 \ll 5) \;|\; (B_5)
\]
- **디더링**은 양자화 직전에 **작은 섭동**을 더하거나, 양자화 후 **오차를 이웃으로 확산**하여 밴딩을 줄인다.

---

## 3. 감마(전달함수)와 디더링 — 어디에서 할까?
- 대부분의 565 LCD는 **sRGB 유사 감마**(≈2.2)를 내장 가정.
- **연산(리사이즈/블렌드/컨볼루션)** 은 **선형광**에서 하는 것이 맞다.
- **디더링 단계**는 보통 **최종 코드값 경계**에 맞춰 **sRGB 코드공간**에서 하는 것이 실용적이다.
  - 즉, 입력을 sRGB(0..1)로 보고 **코드 레벨 기반** 임계치/오차를 사용 → 실제 하드웨어의 **코드 경계**와 일치.

아래 예제는 **sRGB 코드공간**(0..1)에서 디더링을 수행하여 5/6bit 단계 경계를 직접 겨냥한다.
(필요 시 §11에서 **선형광 기반 디더링**도 간단히 변형하는 법을 안내한다.)

---

## 4. 공통 유틸 (sRGB 변환, 패킹 등)
```cpp
#include <algorithm>
#include <cmath>
#include <cstdint>

// sRGB<->Linear (정확한 sRGB 전달함수) — 필터/블렌딩이 필요할 때 사용
inline float srgb_to_linear(float v) {
    return (v <= 0.04045f) ? (v/12.92f) : std::pow((v + 0.055f)/1.055f, 2.4f);
}
inline float linear_to_srgb(float l) {
    l = std::clamp(l, 0.0f, 1.0f);
    return (l <= 0.0031308f) ? (12.92f*l) : (1.055f*std::pow(l, 1.0f/2.4f) - 0.055f);
}
inline uint8_t to_u8(float v01) {
    int iv = (int)std::floor(std::clamp(v01,0.0f,1.0f)*255.0f + 0.5f);
    return (uint8_t)std::clamp(iv,0,255);
}

// 8bit → 정규화 sRGB (0..1)
inline float u8_to_srgb01(uint8_t u) { return u/255.0f; }

// 5/6/5 비트 패킹
inline uint16_t pack_rgb565(uint8_t r8, uint8_t g8, uint8_t b8) {
    uint16_t r5 = (uint16_t)((r8 * 31 + 127) / 255);
    uint16_t g6 = (uint16_t)((g8 * 63 + 127) / 255);
    uint16_t b5 = (uint16_t)((b8 * 31 + 127) / 255);
    return (uint16_t)((r5<<11) | (g6<<5) | b5);
}
```

---

## 5. **Ordered Dithering** (8×8 Bayer) — 빠르고 간단, 패턴 일정

### 5.1 원리
- M×M 임계치 행렬 \(B[y,x]\in[0..M^2-1]\) 로 **정규화 임계치** \(t=\frac{B+0.5}{M^2}\in[0,1)\) 를 만들고,
\[
q = \left\lfloor s \cdot L + t \right\rfloor,\qquad L=\text{레벨 수}(5bit\to 32,\;6bit\to 64)
\]
- 즉, **코드 경계**에 소량의 임계치를 더해 **올리기/내리기**를 번갈아 결정한다.

### 5.2 코드
```cpp
// 8x8 Bayer threshold matrix (0..63)
static const uint8_t BAYER8[8][8] = {
    { 0,48,12,60, 3,51,15,63 },
    { 32,16,44,28,35,19,47,31 },
    { 8,56, 4,52,11,59, 7,55 },
    { 40,24,36,20,43,27,39,23 },
    { 2,50,14,62, 1,49,13,61 },
    { 34,18,46,30,33,17,45,29 },
    {10,58, 6,54, 9,57, 5,53 },
    {42,26,38,22,41,25,37,21 }
};

// src: BGRA8 sRGB, dst: RGB565
void DitherOrdered_RGB565(const uint8_t* srcBGRA, int w, int h, int sstrideBytes,
                          uint16_t* dst565, int dstridePixels /*in uint16_t*/)
{
    for (int y=0; y<h; ++y) {
        const uint8_t* srow = srcBGRA + (size_t)sstrideBytes*y;
        uint16_t* drow = dst565 + (size_t)dstridePixels*y;

        for (int x=0; x<w; ++x) {
            const uint8_t* p = srow + x*4;
            float sr = u8_to_srgb01(p[2]); // sRGB code space (0..1)
            float sg = u8_to_srgb01(p[1]);
            float sb = u8_to_srgb01(p[0]);

            // Threshold in [0,1)
            float t = (BAYER8[y & 7][x & 7] + 0.5f) / 64.0f;

            // Quantize in code space with threshold — levels: R,B=32, G=64
            int R5 = (int)std::floor(sr * 32.0f + t);
            int G6 = (int)std::floor(sg * 64.0f + t);
            int B5 = (int)std::floor(sb * 32.0f + t);

            R5 = std::clamp(R5, 0, 31);
            G6 = std::clamp(G6, 0, 63);
            B5 = std::clamp(B5, 0, 31);

            uint16_t px = (uint16_t)((R5<<11) | (G6<<5) | B5);
            drow[x] = px;
        }
    }
}
```

> **특징**
> - 빠름, 분기/메모리 거의 없음.
> - 타일 패턴이 **정적**이므로 큰 평탄 영역에서 **약한 체커/망점**이 보일 수 있음.
> - **UI/문서**에서 자연스러운 결과. **사진**엔 Error Diffusion이 더 좋을 때가 많다.

---

## 6. **Error Diffusion (Floyd–Steinberg)** — 품질 우선

### 6.1 원리
- 픽셀마다 양자화 후 **오차** \(e = s - \hat{s}\) 를 **이웃 픽셀로 분배**하여, 전체적으로 평균이 맞게 한다.
- Floyd–Steinberg 커널(좌→우 스캔):
\[
\begin{matrix}
\cdot & \;\; & \tfrac{7}{16}\\
\tfrac{3}{16} & \tfrac{5}{16} & \tfrac{1}{16}
\end{matrix}
\]
- **Serpentine(지그재그)** 스캔으로 방향성 패턴을 완화.

### 6.2 코드 (sRGB 코드공간에서 레벨 오차 확산)
```cpp
#include <vector>

// Error diffusion in sRGB code space (0..1). Serpentine scanning.
void DitherFS_RGB565(const uint8_t* srcBGRA, int w, int h, int sstrideBytes,
                     uint16_t* dst565, int dstridePixels /*uint16_t*/)
{
    // 행별 오차 버퍼 (각 채널 0..1 스케일)
    std::vector<float> errR_curr(w+2,0.f), errR_next(w+2,0.f);
    std::vector<float> errG_curr(w+2,0.f), errG_next(w+2,0.f);
    std::vector<float> errB_curr(w+2,0.f), errB_next(w+2,0.f);

    auto clamp01 = [](float v){ return std::clamp(v, 0.0f, 1.0f); };
    auto quantN = [](float s, int levels) {
        // s in [0,1], quantize to N levels (0..levels-1), return index
        int q = (int)std::floor(s * levels + 0.5f); // round-to-nearest
        return std::clamp(q, 0, levels-1);
    };

    for (int y=0; y<h; ++y) {
        const uint8_t* srow = srcBGRA + (size_t)sstrideBytes*y;
        uint16_t* drow = dst565 + (size_t)dstridePixels*y;

        bool leftToRight = (y % 2 == 0);
        int xStart = leftToRight ? 0 : (w-1);
        int xEnd   = leftToRight ? w : -1;
        int step   = leftToRight ? 1 : -1;

        // 포인터로 현재/다음 행 선택
        auto &eRc = errR_curr, &eRn = errR_next;
        auto &eGc = errG_curr, &eGn = errG_next;
        auto &eBc = errB_curr, &eBn = errB_next;

        for (int x=xStart; x!=xEnd; x+=step) {
            int xi = x + 1; // 경계 여유(좌측에 1칸)
            const uint8_t* p = srow + x*4;

            float sr = u8_to_srgb01(p[2]) + eRc[xi];
            float sg = u8_to_srgb01(p[1]) + eGc[xi];
            float sb = u8_to_srgb01(p[0]) + eBc[xi];

            sr = clamp01(sr); sg = clamp01(sg); sb = clamp01(sb);

            // 5/6/5 레벨로 양자화
            int R5 = quantN(sr, 32);
            int G6 = quantN(sg, 64);
            int B5 = quantN(sb, 32);

            // 복원된 sRGB 코드값(0..1)로 재구성—오차 계산
            float rQ = (float)R5 / 31.0f;
            float gQ = (float)G6 / 63.0f;
            float bQ = (float)B5 / 31.0f;

            float er = sr - rQ;
            float eg = sg - gQ;
            float eb = sb - bQ;

            // 패킹
            uint16_t px = (uint16_t)((R5<<11) | (G6<<5) | B5);
            drow[x] = px;

            // 오차 확산 (좌→우 기준 좌표, Serpentine 시 좌우 반전)
            // 가중치: 7/16(right), 3/16(down-left), 5/16(down), 1/16(down-right)
            if (leftToRight) {
                eRc[xi+1] += er * 7.f/16.f; eGc[xi+1] += eg * 7.f/16.f; eBc[xi+1] += eb * 7.f/16.f;
                eRn[xi-1] += er * 3.f/16.f; eGn[xi-1] += eg * 3.f/16.f; eBn[xi-1] += eb * 3.f/16.f;
                eRn[xi  ] += er * 5.f/16.f; eGn[xi  ] += eg * 5.f/16.f; eBn[xi  ] += eb * 5.f/16.f;
                eRn[xi+1] += er * 1.f/16.f; eGn[xi+1] += eg * 1.f/16.f; eBn[xi+1] += eb * 1.f/16.f;
            } else {
                // 오른→왼 (가중치 좌우 반전)
                eRc[xi-1] += er * 7.f/16.f; eGc[xi-1] += eg * 7.f/16.f; eBc[xi-1] += eb * 7.f/16.f;
                eRn[xi+1] += er * 3.f/16.f; eGn[xi+1] += eg * 3.f/16.f; eBn[xi+1] += eb * 3.f/16.f;
                eRn[xi  ] += er * 5.f/16.f; eGn[xi  ] += eg * 5.f/16.f; eBn[xi  ] += eb * 5.f/16.f;
                eRn[xi-1] += er * 1.f/16.f; eGn[xi-1] += eg * 1.f/16.f; eBn[xi-1] += eb * 1.f/16.f;
            }
        }

        // 다음 행으로 롤
        std::fill(errR_curr.begin(), errR_curr.end(), 0.f);
        std::fill(errG_curr.begin(), errG_curr.end(), 0.f);
        std::fill(errB_curr.begin(), errB_curr.end(), 0.f);
        errR_curr.swap(errR_next); errG_curr.swap(errG_next); errB_curr.swap(errB_next);
        std::fill(errR_next.begin(), errR_next.end(), 0.f);
        std::fill(errG_next.begin(), errG_next.end(), 0.f);
        std::fill(errB_next.begin(), errB_next.end(), 0.f);
    }
}
```

> **특징**
> - 평탄 영역에서도 **패턴이 덜 보임**(자연스러운 필름 그레인 느낌).
> - 비용이 더 큼(메모리 접근/부동소수점). **Serpentine** 을 꼭 켜서 방향성 줄이기.

---

## 7. **랜덤/블루 노이즈 Ordered** — 패턴 저감, 캐시 친화
- **Bayer**의 규칙 패턴이 거슬리면, **블루 노이즈 타일(예: 64×64)** 을 t로 써서 `t∈[0,1)` 로 정규화해 §5의 공식을 그대로 적용한다.
- 블루 노이즈는 **저주파 성분이 적어** 눈에 덜 띄는 장점.
- 간단 대안: **크게 섞은 LCG/PCG 난수 타일**을 미리 생성해 반복 사용(프레임 간 고정하면 깜빡임 없음).

---

## 8. 알파가 있는 입력을 565로 내려야 할 때
- RGB565엔 **알파 없음** → 565로 **내리기 전에** **배경색과 선형 프리멀티 합성**을 끝낸다(§5/감마 절 참고).
- 절차: BGRA(sRGB) → **sRGB→Linear**, 프리멀티 합성 → **Linear→sRGB** → 디더/양자화 → RGB565.

---

## 9. 성능 팁
- **정규화 LUT**: `u8→s(0..1)` 는 `lut[256]` 로, `s→code` 는 `(u*levels + offs)>>8` 같은 정수 근사로 가속.
- **SIMD**: SSE2/NEON으로 4~16픽셀 병렬. Ordered는 **분기 없음** → SIMD 적합.
- **타일링**: 캐시/버퍼 스와핑 줄이기 위해 **행 타일** 단위 처리.
- **고스트 방지**: 에러 버퍼 초기화/경계 처리(위 코드처럼 좌우에 1칸 여유) 필수.

---

## 10. MFC/Win32에서 **RGB565 DIBSection** 만들기(화면 출력)
```cpp
#include <windows.h>

// 16bpp RGB565 비트필드 DIBSection 생성
HBITMAP CreateRGB565DIBSection(HDC hdc, int w, int h, void** outBits) {
    BITMAPINFO bmi{};
    bmi.bmiHeader.biSize        = sizeof(BITMAPINFOHEADER);
    bmi.bmiHeader.biWidth       = w;
    bmi.bmiHeader.biHeight      = -h; // top-down
    bmi.bmiHeader.biPlanes      = 1;
    bmi.bmiHeader.biBitCount    = 16;
    bmi.bmiHeader.biCompression = BI_BITFIELDS;
    // RGB565 masks
    ((DWORD*)bmi.bmiColors)[0] = 0xF800; // R
    ((DWORD*)bmi.bmiColors)[1] = 0x07E0; // G
    ((DWORD*)bmi.bmiColors)[2] = 0x001F; // B

    HBITMAP hbmp = CreateDIBSection(hdc, &bmi, DIB_RGB_COLORS, outBits, NULL, 0);
    return hbmp;
}

// 예) OnPaint에서 StretchDIBits로 그리기 (이미 bits에 565 픽셀 채움)
void BlitRGB565(HDC hdc, int dx, int dy, int w, int h, const void* bits, int pitchBytes) {
    BITMAPINFO bmi{};
    bmi.bmiHeader.biSize        = sizeof(BITMAPINFOHEADER);
    bmi.bmiHeader.biWidth       = w;
    bmi.bmiHeader.biHeight      = -h;
    bmi.bmiHeader.biPlanes      = 1;
    bmi.bmiHeader.biBitCount    = 16;
    bmi.bmiHeader.biCompression = BI_BITFIELDS;
    ((DWORD*)bmi.bmiColors)[0] = 0xF800;
    ((DWORD*)bmi.bmiColors)[1] = 0x07E0;
    ((DWORD*)bmi.bmiColors)[2] = 0x001F;

    StretchDIBits(hdc, dx, dy, w, h, 0, 0, w, h,
                  bits, &bmi, DIB_RGB_COLORS, SRCCOPY);
}
```

---

## 11. (선택) **선형광 기반 디더링**으로 더 정확히
- 디더링을 **선형광**에서 하면 밝기 보존이 더 좋다. 절차:
  1) sRGB(0..1) → **Linear**(0..1)
  2) (Ordered 또는 FS) 디더링을 **Linear** 값에 적용
  3) 최종적으로 **sRGB로 다시 올린 뒤** 5/6bit **코드 경계**로 스냅
- 구현은 §5/§6에서 `sr,sg,sb` 를 `srgb_to_linear(...)` 로 치환하고, 복원 `rQ,gQ,bQ` 를 **역매핑**할 때도 `linear_to_srgb` 경유.
- 다만 이 경우, “코드 경계” 자체가 sRGB 공간임을 감안해 **에러 계산/확산을 어느 공간 기준으로 둘지** 정책을 정해야 한다.
  - **실용안**: **Linear에서 디더·에러 계산**, **양자화는 최종에 sRGB로 복귀하여 코드 스냅**.
  - 구현 복잡도↑. 대부분의 565 LCD/임베디드에선 **sRGB 코드공간 디더링**도 충분히 우수.

---

## 12. 비교 가이드 (언제 어떤 기법?)
| 상황 | 추천 |
|---|---|
| 사진/그라디언트 넓은 영역 | **Floyd–Steinberg**(Serpentine) 또는 **블루 노이즈 Ordered** |
| UI/문서, 선/글자 | **8×8 Bayer Ordered** (빠르고 선명) |
| 매우 저성능(임베디드) | **Bayer Ordered** (정수/분기 없음) |
| 노이즈 패턴 억제 | **블루 노이즈 타일** (프리컴퓨트) |

---

## 13. 품질 검증 체크리스트
- 밴딩 감소: 8/16 px 간격 그라디언트에서 **밴딩→노이즈**로 바뀌는지.
- 얇은 글꼴 테두리/아이콘: 수평/수직/대각선 라인에서 **색 흔들림** 최소화.
- 패턴성: Bayer 체커가 보이면 **블루 노이즈/FS** 로 전환.
- 성능: 프레임 타임/메모리 대역폭 확인. Ordered→SIMD 가속.

---

## 14. 통합 예시 — “모드 스위치” 함수
```cpp
enum class DitherMode { None, OrderedBayer8, FloydSteinberg };

void Convert_BGRA8_to_RGB565(const uint8_t* srcBGRA, int w, int h, int sstrideBytes,
                             uint16_t* dst565, int dstridePixels, DitherMode mode)
{
    switch (mode) {
    case DitherMode::None:
        for (int y=0; y<h; ++y) {
            const uint8_t* srow = srcBGRA + (size_t)sstrideBytes*y;
            uint16_t* drow = dst565 + (size_t)dstridePixels*y;
            for (int x=0; x<w; ++x) {
                const uint8_t* p = srow + x*4;
                drow[x] = pack_rgb565(p[2], p[1], p[0]); // 단순 라운딩
            }
        }
        break;
    case DitherMode::OrderedBayer8:
        DitherOrdered_RGB565(srcBGRA, w, h, sstrideBytes, dst565, dstridePixels);
        break;
    case DitherMode::FloydSteinberg:
        DitherFS_RGB565(srcBGRA, w, h, sstrideBytes, dst565, dstridePixels);
        break;
    }
}
```

---

## 15. 흔한 실수 & 해결
| 실수 | 증상 | 해결 |
|---|---|---|
| 디더 전 **리사이즈/블렌드**를 sRGB에서 함 | 어두운 쪽 뭉개짐, 링잉 | **선형광에서 연산** 후 디더링 |
| 오차 확산에서 경계 처리 없음 | 테두리 밝기 끌림 | **여유 인덱스**/클램프, 다음 행 버퍼 초기화 |
| Bayer를 모든 채널에 동일 적용 | 컬러 튐 | **채널별** 동일 t 사용은 보통 괜찮지만, 필요 시 G에 보수적 스케일 |
| 난수 디더 프레임마다 변경 | 깜빡임 | **타일 고정**(frame-stable) |
| 알파 입력을 565로 직접 디더 | 테두리 검게/밝게 | **배경과 합성 후** 디더 |

---

## 16. 요약
- 저비트 표면(RGB565)에서 밴딩을 줄이려면, **Ordered**(빠름, 예측가능) 또는 **Floyd–Steinberg**(품질↑)를 사용하라.
- **연산은 선형**, **디더/양자화는 sRGB 코드 경계 기준**이 현실적이고, 필요 시 선형 기반 디더로 확장 가능.
- 제공한 C++ 함수들을 **ImageTool/MFC**에 그대로 이식하면, **565 대상 장치**(임베디드 LCD, DIBSection 등)에 즉시 적용 가능하다.
