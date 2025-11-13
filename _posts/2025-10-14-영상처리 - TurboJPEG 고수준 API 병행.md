---
layout: post
title: 영상처리 - TurboJPEG 고수준 API 병행
date: 2025-10-14 17:25:23 +0900
category: 영상처리
---
# 9. **TurboJPEG 고수준 API 병행 (성능·단순화)**
> jpeglib의 유연함은 그대로 두고, **썸네일·미리보기·일반 디코드/인코드** 경로는 **TurboJPEG**로 단순화/가속!

- **왜**: `tjDecompress2()`/`tjCompress2()`는 **BGRA 버퍼를 그대로** 넣고 빼기 쉽고, SIMD(AVX2/NEON)로 **매우 빠릅니다**.
- **무엇**: `TJPF_BGRA` 픽셀 포맷, `TJSAMP_420/444` 서브샘플링, `TJFLAG_FASTUPSAMPLE`/`TJFLAG_FASTDCT`(속도)·`TJFLAG_ACCURATEDCT`(정확), **스케일 디코드**(tjGetScalingFactors), **무손실 변환**(`tjTransform`)까지.

---

## 0. 빠른 개념 맵
- **디코드(파일→BGRA)**: `tjInitDecompress` → `tjDecompressHeader3`(원본 W/H/서브샘플링 파악) → (옵션) 스케일 팩터 선택 → `tjDecompress2`.
- **인코드(BGRA→JPEG)**: `tjInitCompress` → `tjCompress2` (포맷=BGRA, 샘플링=420/444, quality, progressive).
- **무손실 변환(JPEG→JPEG)**: `tjInitTransform` → `tjTransform` (회전/반전/트림/크롭: MCU 경계 기반).
- **ROI 디코드**: TurboJPEG 자체는 **부분 디코드(read ROI)** API가 제한적. **썸네일·타일**은 **스케일 디코드**로 충분히 빠르며, **무손실 크롭**은 `tjTransform`으로 해결.

---

## 1. 빌드 & 준비
- vcpkg(Windows) 예:
  ```powershell
  .\vcpkg\vcpkg install libjpeg-turbo:x64-windows
  ```
- CMake 예:
  ```cmake
  find_package(JPEG REQUIRED) # vcpkg의 libjpeg-turbo가 JPEG로 노출
  target_link_libraries(yourapp PRIVATE JPEG::JPEG)
  ```
- 헤더:
  ```cpp
  #include <turbojpeg.h>
  ```

> **주의**
> - TurboJPEG 버퍼는 **`tjFree()`로 해제**해야 합니다(표준 `free()` 금지).
> - 예외 대신 **리턴값**과 `tjGetErrorStr2(handle)`로 오류 문자열 확인.

---

## 2. 디코드: **JPEG → BGRA(IppDib)**
### 2.1 스케일 팩터 선택(1/1, 1/2, 1/4, 1/8 등)
```cpp
#include <turbojpeg.h>
#include <vector>
#include <string>
#include <cstdio>
#include <algorithm>
#include "IppDib.h"

struct TJDecodeOpts {
    bool fast_upsample = true;  // TJFLAG_FASTUPSAMPLE
    bool fast_dct      = true;  // TJFLAG_FASTDCT (vs ACCURATEDCT)
    bool limit_scans   = true;  // TJFLAG_LIMITSCANS (진행형 폭탄 완화)
    int  maxThumb      = 0;     // 0=원본, 양수=긴 변 이 크기 이하로 스케일
};

static int chooseScaledSize(int w, int h, int maxThumb, int& outW, int& outH, int& scaleNum, int& scaleDen)
{
    if (maxThumb <= 0){ outW=w; outH=h; scaleNum=1; scaleDen=1; return 1; }

    int ns=0; tjscalingfactor* sfs = tjGetScalingFactors(&ns);
    // 가장 큰 쪽을 maxThumb 이하로 만드는 '가장 큰' 스케일 팩터 선택
    double bestScale=0.0; scaleNum=1; scaleDen=1; outW=w; outH=h;
    for (int i=0;i<ns;++i){
        int sw = TJSCALED(w, sfs[i]);
        int sh = TJSCALED(h, sfs[i]);
        int larger = std::max(sw, sh);
        if (larger <= maxThumb){
            double sc = (double)sw / w; // 비율
            if (sc > bestScale){ bestScale=sc; outW=sw; outH=sh; scaleNum=sfs[i].num; scaleDen=sfs[i].denom; }
        }
    }
    return (bestScale>0.0)? 1:0;
}

bool TJ_DecodeToBGRA(const std::wstring& path, const TJDecodeOpts& opts, IppDib& out, std::string& err)
{
    err.clear(); out.destroy();

    // 1) 파일 읽기
    FILE* fp=nullptr; _wfopen_s(&fp, path.c_str(), L"rb");
    if(!fp){ err="open failed"; return false; }
    fseek(fp, 0, SEEK_END); long sz = ftell(fp); fseek(fp, 0, SEEK_SET);
    if (sz<=0){ fclose(fp); err="empty file"; return false; }

    std::vector<unsigned char> jpg(sz);
    if (fread(jpg.data(), 1, jpg.size(), fp) != (size_t)sz){ fclose(fp); err="read error"; return false; }
    fclose(fp);

    // 2) 디코더 생성
    tjhandle td = tjInitDecompress();
    if (!td){ err = tjGetErrorStr2(td); return false; }

    int w=0,h=0,subsamp=0,colorspace=0;
    if (tjDecompressHeader3(td, jpg.data(), (unsigned long)jpg.size(), &w,&h,&subsamp,&colorspace) != 0){
        err = tjGetErrorStr2(td); tjDestroy(td); return false;
    }

    // 3) 스케일 결정
    int outW=w, outH=h, sn=1, sd=1;
    if (opts.maxThumb>0){
        chooseScaledSize(w,h, opts.maxThumb, outW,outH, sn,sd);
        // tjDecompress2는 width/height를 직접 넘기면 내부가 가장 가까운 스케일 팩터로 동작
    }

    // 4) 출력 버퍼 준비
    if (!out.create(outW, outH, 32)){ tjDestroy(td); err="alloc failed"; return false; }

    // 5) 플래그
    int flags=0;
    if (opts.fast_upsample) flags |= TJFLAG_FASTUPSAMPLE;
    if (opts.fast_dct)      flags |= TJFLAG_FASTDCT; else flags |= TJFLAG_ACCURATEDCT;
    if (opts.limit_scans)   flags |= TJFLAG_LIMITSCANS; // 프로그레시브 폭탄 완화

    // 6) 디코드 (BGRA 직출)
    if (tjDecompress2(td,
        jpg.data(), (unsigned long)jpg.size(),
        (unsigned char*)out.bits(), outW, out.stride(), outH,
        TJPF_BGRA, flags) != 0)
    {
        err = tjGetErrorStr2(td); tjDestroy(td); out.destroy(); return false;
    }

    tjDestroy(td);
    return true;
}
```

**포인트**
- `TJPF_BGRA`로 **바로 BGRA** 복원 → 수동 채널 스왑 불필요.
- `tjGetScalingFactors()`로 **스케일 디코드**(썸네일/프리뷰) 간단.
- `TJFLAG_LIMITSCANS`(권장): 진행형 JPEG의 **악성 스캔 폭탄** 방어.

---

## 3. 인코드: **BGRA(IppDib) → JPEG**
```cpp
struct TJEncodeOpts {
    int quality = 85;            // 1~100
    int subsamp = TJSAMP_420;    // TJSAMP_444 / 422 / 420 / GRAY
    bool progressive = false;    // TJFLAG_PROGRESSIVE
    bool optimize   = true;      // TJFLAG_OPTIMIZE_CODING
    bool fast_dct   = true;      // TJFLAG_FASTDCT (vs ACCURATEDCT)
};

bool TJ_EncodeFromBGRA(const IppDib& img, const TJEncodeOpts& o,
                       std::vector<unsigned char>& outJpeg, std::string& err)
{
    err.clear(); outJpeg.clear();
    if (!img){ err="invalid image"; return false; }

    tjhandle tc = tjInitCompress();
    if (!tc){ err=tjGetErrorStr2(tc); return false; }

    int flags=0;
    if (o.progressive) flags |= TJFLAG_PROGRESSIVE;
    if (o.optimize)    flags |= TJFLAG_OPTIMIZE_CODING;
    if (o.fast_dct)    flags |= TJFLAG_FASTDCT; else flags |= TJFLAG_ACCURATEDCT;

    unsigned char* jpegBuf=nullptr;
    unsigned long  jpegSize=0;

    if (tjCompress2(tc,
        (unsigned char*)img.bits(), img.width(), img.stride(), img.height(),
        TJPF_BGRA, &jpegBuf, &jpegSize,
        o.subsamp, o.quality, flags) != 0)
    {
        err=tjGetErrorStr2(tc); tjDestroy(tc); return false;
    }

    outJpeg.assign(jpegBuf, jpegBuf + jpegSize);
    tjFree(jpegBuf);
    tjDestroy(tc);
    return true;
}
```

**포인트**
- **서브샘플링**: 텍스트·GUI 캡처는 `TJSAMP_444`, 일반 사진은 `TJSAMP_420`.
- **progressive**는 용량↓ 경향, **optimize**(허프만 최적화)는 대부분 이득.
- 고품질/색정확도가 중요하면 `TJFLAG_ACCURATEDCT` 사용.

---

## 4. (옵션) **타깃 사이즈** 맞추기 — TurboJPEG로도 “이진 탐색” 간단
```cpp
bool TJ_Encode_ToTarget(const IppDib& img, size_t targetBytes, size_t tol,
                        const TJEncodeOpts& base, std::vector<unsigned char>& best, int& usedQ)
{
    TJEncodeOpts cur = base;
    int lo=1, hi=100; usedQ=-1; std::vector<unsigned char> tmp, bestBuf;
    for (int it=0; it<10 && lo<=hi; ++it){
        int mid=(lo+hi)/2;
        cur.quality = mid;
        std::string e;
        if (!TJ_EncodeFromBGRA(img, cur, tmp, e)) return false;
        if (tmp.size() <= targetBytes + tol){ usedQ=mid; bestBuf=tmp; lo=mid+1; }
        else hi=mid-1;
    }
    if (usedQ<0) return false;
    best.swap(bestBuf); return true;
}
```

---

## 5. **무손실 변환(회전/반전/크롭)** — `tjTransform`
> `jpegtran`/`transupp` 없이 TurboJPEG만으로 **MCU 경계 기반** 회전/반전/크롭 가능.

```cpp
struct TJLossless {
    int op = TJXOP_ROT90;   // TJXOP_NONE, TJXOP_ROT90/180/270, TJXOP_HFLIP/VFLIP, TJXOP_TRANSPOSE/TRANSVERSE
    bool trim = true;       // TJXOPT_TRIM: 가장자리 MCU 버림
    bool crop = false;      // TJXOPT_CROP 사용
    // crop rect (픽셀 기준, 내부적으로 MCU 정렬)
    int x=0,y=0,w=0,h=0;
};

bool TJ_LosslessTransform(const std::vector<unsigned char>& inJpeg,
                          const TJLossless& opt,
                          std::vector<unsigned char>& outJpeg, std::string& err)
{
    tjhandle tt = tjInitTransform(); if (!tt){ err=tjGetErrorStr2(tt); return false; }

    tjtransform t{};
    t.op = opt.op;
    t.options = 0;
    if (opt.trim) t.options |= TJXOPT_TRIM;
    if (opt.crop){
        t.options |= TJXOPT_CROP;
        t.r.left = opt.x;  t.r.top = opt.y;
        t.r.width= opt.w;  t.r.height= opt.h;
    }

    unsigned char* dstBuf=nullptr; unsigned long dstSize=0;
    int ret = tjTransform(tt,
        inJpeg.data(), (unsigned long)inJpeg.size(),
        1, &dstBuf, &dstSize, &t, 0);

    if (ret!=0){ err=tjGetErrorStr2(tt); tjDestroy(tt); return false; }

    outJpeg.assign(dstBuf, dstBuf+dstSize);
    tjFree(dstBuf);
    tjDestroy(tt);
    return true;
}
```

**포인트**
- `TJXOPT_CROP`는 **MCU 정렬**로 실제 사각형이 약간 조정될 수 있습니다(진정한 무손실 조건).
- EXIF Orientation을 “픽셀 반영 + Orientation=1 패치”하려면
  - **무손실 회전/반전** 후 **메타데이터 재주입**(jpeglib 경로) 추천.

---

## 6. **시나리오 통합** (ImageTool에 넣기)

### 6.1 썸네일 경로
- 목록/그리드: `TJ_DecodeToBGRA(maxThumb=256, FASTUPSAMPLE, FASTDCT, LIMITSCANS)`.
- 결과 `IppDib`를 `StretchDIBits()`로 즉시 그리기.
- 클릭/확대 시 원본(또는 1/2) 재디코드.

### 6.2 일괄 리사이즈·저장
- 디코드(`TJ_DecodeToBGRA`) → (선택) 리사이즈 → 인코드(`TJ_EncodeFromBGRA`).
- 옵션: 사진=420/90Q/optimize/progressive, UI 캡처=444/85Q/accurate DCT.

### 6.3 EXIF 회전 반영 + 무손실
- 원본 JPEG(바이트) + Orientation 읽기 → `TJ_LosslessTransform`으로 회전/반전 →
  메타 재주입(앞에서 만든 jpeglib 메타 모듈) → **Orientation=1**로 정정 후 저장.

---

## 7. 성능/안전 플래그 가이드
- **FASTUPSAMPLE + FASTDCT**: 가장 빠름(썸네일/프리뷰).
- **ACCURATEDCT**: 고품질(모던 CPU에서도 충분히 빠름).
- **LIMITSCANS**: 진행형 JPEG **스캔 폭탄** 완화(권장).
- **BOTTOMUP**: BMP 같은 **하단→상단 메모리**를 직접 채우고 싶을 때 사용(일반적으로 비권장).

---

## 8. 주의 사항 & 팁
- TurboJPEG는 **APP 마커 편집/보존 제어가 제한**됩니다. 메타데이터(EXIF/XMP/ICC) 정밀 제어가 필요하면
  **jpeglib 경로**(이전 섹션)로 **추출/재주입**하세요.
- CMYK/YCCK JPEG 디코드는 TurboJPEG로도 가능하지만 **표시 경로**는 **RGB/BGRA**로 통일하세요(ICC 변환은 lcms2).
- 스케일 디코드는 **해상도 기반 속도 이득**이 큽니다. 썸네일/미리보기는 **반드시 스케일**로.
- `tjDecompressHeader3`로 원본 **subsamp**를 확인하고, 인코드 시 **444/420**을 **의도적으로** 선택하세요(텍스트/선화=444 권장).

---

## 9. 테스트 체크리스트
- [ ] 동일 이미지에 대해 **tjDecompress2** vs **libjpeg 디코드** 품질/속도 비교.
- [ ] **스케일 팩터**(1/2,1/4,1/8)에서 크기 정확성, 썸네일 FPS 측정.
- [ ] **LIMITSCANS** 켠/끈 성능/안전 차이(악성 프로그레시브 샘플).
- [ ] **무손실 회전/크롭** 결과가 `jpegtran`과 **동일**한지 확인.
- [ ] 인코드에서 **420/444**, **progressive/optimize**, **FASTDCT/ACCURATEDCT** 조합별 용량·품질 비교.

---

## 10. 미니 CLI 예제 (소스→썸네일→저장)
```cpp
// tj_cli.cpp
#include <iostream>
int wmain(int argc, wchar_t** argv){
    if (argc<4){ std::wcout<<L"usage: tjcli <in.jpg> <out.jpg> <maxThumb>\n"; return 0; }
    std::wstring in=argv[1], out=argv[2]; int maxThumb=_wtoi(argv[3]);

    TJDecodeOpts dopt; dopt.maxThumb=maxThumb; dopt.fast_upsample=true; dopt.fast_dct=true; dopt.limit_scans=true;
    IppDib dib; std::string err;
    if (!TJ_DecodeToBGRA(in, dopt, dib, err)){ std::wcerr<<L"decode: "<<err.c_str()<<L"\n"; return 1; }

    TJEncodeOpts eopt; eopt.quality=85; eopt.subsamp=TJSAMP_420; eopt.progressive=true; eopt.optimize=true;
    std::vector<unsigned char> jpg;
    if (!TJ_EncodeFromBGRA(dib, eopt, jpg, err)){ std::wcerr<<L"encode: "<<err.c_str()<<L"\n"; return 1; }

    FILE* fp=nullptr; _wfopen_s(&fp, out.c_str(), L"wb");
    if (!fp){ std::wcerr<<L"open out failed\n"; return 1; }
    fwrite(jpg.data(),1,jpg.size(),fp); fclose(fp);
    std::wcout<<L"OK "<<jpg.size()<<L" bytes\n"; return 0;
}
```

---

## 11. 결론
- **TurboJPEG**는 “**BGRA 직결·스케일 디코드·간단 인코드**”가 강력해서
  **미리보기/썸네일/일반 파이프라인**의 복잡도를 크게 줄여줍니다.
- **무손실 변환**도 `tjTransform` 한 방.
- **메타데이터/색관리** 등 정밀 제어는 기존 **jpeglib 경로**와 **병행**하세요.
- 결과적으로, ImageTool은
  - **읽기/프리뷰**: TurboJPEG,
  - **메타/무손실+메타/특수색공간**: jpeglib,
  - 을 **하이브리드**로 써서 **간단·빠름·정확**을 동시에 달성할 수 있습니다.
