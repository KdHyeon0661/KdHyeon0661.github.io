---
layout: post
title: 영상처리 -  JPEG XL, AVIF, WebP 변환 파이프라인
date: 2025-10-18 18:25:23 +0900
category: 영상처리
---
# JPEG XL / AVIF / WebP **변환 파이프라인** (비교 · 마이그레이션 대비)

> 목표  
> - 기존 JPEG/PNG/RAW 자산을 **AVIF / JPEG XL(JXL) / WebP**로 변환·전송하는 **엔드투엔드 파이프라인**을 설계하고,  
> - **품질/용량/속도**를 데이터로 비교하며,  
> - **브라우저/앱 호환**을 고려한 **점진적 마이그레이션 전략**과 **실전 코드**(CLI & C/C++)를 제시합니다.  
> - 코드는 한 번만 ``` 로 감싸고, 수학식은 MathJax를 씁니다.

---

## 1) 세 코덱/포맷 한눈에 보기

| 항목 | AVIF | JPEG XL (JXL) | WebP |
|---|---|---|---|
| 코덱/컨테이너 | AV1 Image(HEIF 컨테이너 계열) | 전용 비트스트림/컨테이너 | VP8(Lossy)/WebP Lossless |
| 색심도 | 8/10/12bpc, HDR/와이드컬러 | 8~16bpc, HDR/와이드컬러 | 8bpc(일반), 알파 지원 |
| 크로마 | 4:2:0/4:2:2/4:4:4 | 4:2:0/4:2:2/4:4:4 | 4:2:0/4:2:2(일부), 4:4:4 |
| 알파/애니메 | 알파 O / 애니메 O | 알파 O / 애니메 O | 알파 O / 애니메 O |
| 강점(요약) | **용량↓**/고화질, 10bit/HDR | **손실없는 JPEG 래핑**·고속 디코드, 16bpc | **광범위한 실전 호환성**, 인코딩 빠름 |
| 라이브러리 | libavif(+aom/rv1e/SVT-AV1) | libjxl | libwebp |

> **요약 운영 철학**  
> - **사진/캡처 장문서**: AVIF 또는 JXL(특히 **JPEG→JXL 무손실 래핑**)  
> - **UI/일러스트/스티커**: WebP(444/손실/손실없는), AVIF 444  
> - **애니메 이미지**: WebP/AVIF/JXL 모두 가능(파이프라인/플레이어 호환성에 맞춤)

---

## 2) 파이프라인 큰 그림

```
[입력: JPEG/PNG/RAW(+ICC/EXIF)]
        │ ①디코드/정규화(sRGB 확정, 선형연산)
        ▼
[전처리: 리사이즈·샤픈·워터마크·메타 유지정책]
        │
        ├─▶ [AVIF 변환]  : avifenc / libavif
        ├─▶ [JXL 변환]   : cjxl   / libjxl
        └─▶ [WebP 변환]  : cwebp  / libwebp
        │
        ▼ ②품질계측(PSNR/SSIM/MS-SSIM) + 바이트/시간 로깅
[아티팩트 검사·룰(문턱 넘으면 재인코딩)]
        │
        ▼ ③배포(원본과 공존) + 캐시/Accept 네고 + <picture> 폴백
```

---

## 3) 설치/의존(요약)

- **Windows(vcpkg)**: `vcpkg install libavif libjxl libwebp aom svt-av1 lcms`  
- **macOS(brew)**: `brew install libavif libjxl webp aom svt-av1`  
- **Linux(예)**: `apt install libavif-bin libjxl-tools webp`

---

## 4) CLI **배치 변환 예제(권장 시작점)**

### 4.1 AVIF (libavif, aom/rav1e/SVT 선택)

```bash
# 사진(4:2:0) — 품질/속도 밸런스
avifenc input.png out.avif \
  --speed 6 --min 0 --max 63 --cq-level 28 \
  --yuv 420 --depth 10 --jobs 8 \
  --icc input.icc --exif input.exif --xmp input.xmp

# UI/문서(4:4:4) — 엣지 보존
avifenc input.png out_ui.avif \
  --speed 6 --cq-level 20 --yuv 444 --depth 10 --jobs 8
```

- 핵심 파라미터  
  - `--cq-level`: 낮을수록 고화질(대략 18~32 튜닝)  
  - `--speed`: 낮을수록 느리지만 고화질(0~10/코덱별)  
  - `--yuv`: 420/422/444(텍스트·UI는 444 선호)  
  - 메타: `--icc --exif --xmp` 로 **명시적 부착** 권장

### 4.2 JPEG XL (cjxl)

```bash
# 일반 사진 (거리 기반 d: 낮을수록 고화질) + 노력도
cjxl input.png out.jxl -d 1.2 --effort=7 --num_threads=8

# (매우 중요) JPEG 무손실 래핑: JPEG 원본 → JXL 컨테이너 (복원 가능)
cjxl input.jpg out_lossless_wrap.jxl --lossless_jpeg=1
```

- 품질 파라미터  
  - `-d <distance>` 또는 `-q <quality>`; `d≈1~1.5`면 시각상 무손실 근접  
  - `--lossless`(픽셀 무손실) / `--lossless_jpeg=1`(**바이트단위 복원**)  

### 4.3 WebP (cwebp)

```bash
# 사진(손실, 420)
cwebp -q 80 -m 4 -mt -metadata all input.jpg -o out.webp

# UI/문서(444) — 선명 유지
cwebp -q 88 -m 4 -mt -alpha_q 100 -metadata all -af -sns 0 -sharpness 3 \
      -o out_ui.webp input.png

# 무손실(일러스트/스티커)
cwebp -lossless -m 6 -mt -metadata all input.png -o out_ll.webp
```

- 품질 파라미터  
  - `-q`: 0~100  
  - `-m`: 0~6(속도↔품질)  
  - `-metadata all`: ICC/EXIF/XMP 보존(가능한 경우)  

---

## 5) **C++ 라이브러리** 직접 호출 (핵심 스니펫)

### 5.1 libavif — RGB(A) → AVIF

```cpp
#include <avif/avif.h>
#include <vector>
#include <cstdio>

bool EncodeAVIF_RGBA(const uint8_t* rgba, int w, int h, int stride,
                     int depthBits, bool yuv444, int cqLevel, int speed,
                     const uint8_t* icc, size_t iccSize,
                     const uint8_t* exif, size_t exifSize,
                     std::vector<uint8_t>& outBytes)
{
    avifImage* image = avifImageCreate(w, h, depthBits, yuv444 ? AVIF_PIXEL_FORMAT_YUV444 : AVIF_PIXEL_FORMAT_YUV420);
    if (!image) return false;

    // 색역/범위(일반 sRGB): 제한 범위(YUV) vs 전체 범위 정책
    image->colorPrimaries = AVIF_COLOR_PRIMARIES_BT709;
    image->transferCharacteristics = AVIF_TRANSFER_CHARACTERISTICS_SRGB;
    image->matrixCoefficients = AVIF_MATRIX_COEFFICIENTS_BT601; // 보편 601/709, 필요 시 709
    image->yuvRange = AVIF_RANGE_FULL;

    if (icc && iccSize) avifImageSetProfileICC(image, icc, iccSize);
    if (exif && exifSize) {
        avifRWData exifData = AVIF_DATA_EMPTY;
        avifRWDataSet(&exifData, exif, exifSize);
        avifImageSetMetadataExif(image, &exifData);
        avifRWDataFree(&exifData);
    }

    // RGBA → AVIF 내부 버퍼 채우기
    avifRGBImage rgb;
    avifRGBImageSetDefaults(&rgb, image);
    rgb.format = AVIF_RGB_FORMAT_RGBA;
    rgb.depth = 8;
    rgb.rowBytes = stride;
    rgb.pixels = const_cast<uint8_t*>(rgba);
    avifImageRGBToYUV(image, &rgb);

    avifEncoder* enc = avifEncoderCreate();
    enc->maxThreads = 8;
    enc->speed = speed;          // 0(느림/고화질)~10(빠름)
    enc->minQuantizer = 0;       // 범위 0..63
    enc->maxQuantizer = cqLevel; // 품질
    enc->tileColsLog2 = 0;       // 필요시 자동 타일링
    enc->tileRowsLog2 = 0;

    avifRWData out = AVIF_DATA_EMPTY;
    avifResult r = avifEncoderWrite(enc, image, &out);
    if (r == AVIF_RESULT_OK) {
        outBytes.assign(out.data, out.data + out.size);
        avifRWDataFree(&out);
    }

    avifEncoderDestroy(enc);
    avifImageDestroy(image);
    return (r == AVIF_RESULT_OK);
}
```

### 5.2 libwebp — RGBA → WebP (손실/무손실)

```cpp
#include <webp/encode.h>
#include <vector>

bool EncodeWebP_RGBA(const uint8_t* rgba, int w, int h, int stride,
                     float quality, bool lossless, std::vector<uint8_t>& out)
{
    WebPConfig cfg; WebPConfigInit(&cfg);
    if (lossless) {
        cfg.lossless = 1; cfg.method = 6;
    } else {
        cfg.quality = quality; cfg.method = 4; cfg.alpha_quality = 100;
    }
    WebPPicture pic; WebPPictureInit(&pic);
    pic.use_argb = 1; pic.width = w; pic.height = h;

    if (!WebPPictureImportRGBA(&pic, rgba, stride)) return false;
    WebPMemoryWriter wr; WebPMemoryWriterInit(&wr);
    pic.writer = WebPMemoryWrite; pic.custom_ptr = &wr;

    bool ok = WebPEncode(&cfg, &pic);
    if (ok) { out.assign(wr.mem, wr.mem + wr.size); }
    WebPMemoryWriterClear(&wr);
    WebPPictureFree(&pic);
    return ok;
}
```

### 5.3 libjxl — RGBA → JXL (기본 스케치)

```cpp
#include <jxl/encode.h>
#include <vector>

bool EncodeJXL_RGBA(const uint8_t* rgba, int w, int h, int stride,
                    float distance/*1~2 고화질*/, int effort, std::vector<uint8_t>& out)
{
    JxlEncoder* enc = JxlEncoderCreate(nullptr);
    JxlEncoderUseContainer(enc, JXL_TRUE);

    JxlEncoderFrameSettings* fs = JxlEncoderFrameSettingsCreate(enc, nullptr);
    JxlEncoderFrameSettingsSetOption(fs, JXL_ENC_FRAME_SETTING_EFFORT, effort); // 1..9
    JxlEncoderFrameSettingsSetFloatOption(fs, JXL_ENC_FRAME_SETTING_DISTANCE, distance); // 0=무손실

    JxlPixelFormat pf{4, JXL_TYPE_UINT8, JXL_NATIVE_ENDIAN, 0};
    JxlBasicInfo bi; JxlEncoderInitBasicInfo(&bi);
    bi.xsize = (uint32_t)w; bi.ysize = (uint32_t)h;
    bi.bits_per_sample = 8; bi.uses_original_profile = 1;
    JxlColorEncoding cme; JxlColorEncodingSetToSRGB(&cme, /*isGray=*/JXL_FALSE);
    JxlEncoderSetBasicInfo(enc, &bi);
    JxlEncoderSetColorEncoding(enc, &cme);

    JxlEncoderAddImageFrame(fs, &pf, rgba, (size_t)stride*h);
    JxlEncoderCloseInput(enc);

    uint8_t* outbuf = nullptr; size_t outsize = 0;
    JxlEncoderStatus st = JxlEncoderProcessOutput(enc, &outbuf, &outsize); // 스트리밍 반복 필요(단순화)
    // 실제 구현: while(st==NEED_MORE_OUTPUT){버퍼 증가} 루프
    if (st == JXL_ENC_SUCCESS) {
        out.assign(outbuf, outbuf + outsize); // 예시 단순화
        JxlEncoderDestroy(enc);
        return true;
    }
    JxlEncoderDestroy(enc);
    return false;
}
```

> 위 JXL 스니펫은 “출력 버퍼 루프”를 단순화했습니다. 실제로는 `JxlEncoderProcessOutput` 를 루프 돌며 outsize를 늘려가야 합니다.

---

## 6) **품질·속도 계측**(내장 도구 + 자가 지표)

### 6.1 PSNR/SSIM 간단 구현

- PSNR:
\[
\mathrm{PSNR} = 10 \log_{10} \frac{(MAX)^2}{\mathrm{MSE}},\quad MAX=255
\]
- SSIM(간단형):
\[
\mathrm{SSIM}(x,y)=\frac{(2\mu_x\mu_y+C_1)(2\sigma_{xy}+C_2)}{(\mu_x^2+\mu_y^2+C_1)(\sigma_x^2+\sigma_y^2+C_2)}
\]

```cpp
#include <vector>
#include <cmath>
double PSNR_RGB(const uint8_t* a, const uint8_t* b, int w, int h, int strideA, int strideB) {
    double mse=0; long long N=(long long)w*h*3;
    for(int y=0;y<h;++y){
        auto pa=a+y*strideA, pb=b+y*strideB;
        for(int x=0;x<w;++x){
            for(int c=0;c<3;++c){
                int d=int(pa[2-c]) - int(pb[2-c]); // RGB
                mse += double(d*d);
            }
            pa+=4; pb+=4;
        }
    }
    mse/=N; if(mse<=1e-12) return 99.0;
    return 10.0*std::log10((255.0*255.0)/mse);
}
```

> 운영 팁  
> - **MS-SSIM** 같이 시각적 지표를 함께 로깅.  
> - **인코드 시간/디코드 시간/바이트**를 함께 로그 파일/DB에 저장하여 **코호트별 비교**.

---

## 7) **메타데이터/색관리**(ICC/EXIF/XMP) 보존

- **AVIF**: `--icc/--exif/--xmp` 로 명시 부착 권장(소스에서 추출 후 주입).  
- **JXL**: 기본 보존 경향이나, 파이프라인 일관성을 위해 **명시 정책**(보존/익명화) 도입, 필요 시 `--strip`(툴 기준) 옵션.  
- **WebP**: `-metadata all|icc|exif|xmp` 로 선택 보존.  
- sRGB 확정: 파일에 **ICC(sRGB)** 또는 PNG의 `sRGB`/`gAMA`를 명시해 **Color Shift 방지**.

---

## 8) **프리셋 전략**(현장 검증된 스타팅 포인트)

| 용도 | AVIF | JXL | WebP |
|---|---|---|---|
| 사진(일반) | `--yuv 420 --depth 10 --cq-level 28 --speed 6` | `-d 1.2 --effort 7` | `-q 80 -m 4 -mt` |
| UI/문서 | `--yuv 444 --depth 10 --cq-level 20 --speed 6` | `-d 1.0 --effort 7` | `-q 88 -m 4 -af -sns 0 -sharpness 3` |
| 무손실 필요 | `--lossless` (코덱별 지원) | `--lossless` 또는 `--lossless_jpeg=1` | `-lossless -m 6` |
| 애니메 | GOP/프레임 옵션 튜닝 | 프레임 레이어 | `-loop`, 프레임 딜레이 |

> 실제 값은 **콘텐츠 특성**과 **SLO(용량/시간)** 에 맞춰 튜닝하세요.

---

## 9) **배포 & 폴백** 설계

### 9.1 HTML `<picture>` 폴백

```html
<picture>
  <source type="image/avif" srcset="/img/sample.avif">
  <source type="image/webp" srcset="/img/sample.webp">
  <img src="/img/sample.jpg" alt="sample">
</picture>
```

> 서버/클라이언트 지원 상황에 맞게 **우선순위**를 정합니다.  
> 서버는 **Accept 헤더**(`image/avif`, `image/webp`)를 참조해 리라이트/변형 응답 가능.

### 9.2 Nginx(예) — Accept 기반 제공

```nginx
map $http_accept $img_ext {
    default ".jpg";
    "~*image/avif" ".avif";
    "~*image/webp" ".webp";
}

location ~* ^/images/(.*)\.(jpg|jpeg|png)$ {
    try_files /images/$1$img_ext /images/$1.$2 =404;
    add_header Vary Accept;
    expires 30d;
}
```

- 원본이 `/images/foo.jpg`이면, AVIF/WebP 지원 브라우저는 자동으로 `/images/foo.avif` 또는 `.webp` 를 받습니다.

---

## 10) **온디맨드 변환 캐시 서비스**(간단 설계)

- 키: `hash(path + resize + format + quality + color_profile)`  
- 1차 요청: 원본 Fetch → 변환 → `ETag`/`Cache-Control`과 함께 저장  
- 2차 요청: Hit → 바로 서빙  
- 실패 내성: 타임아웃/메모리 상한/폭탄 파일 방지(입력 바이트·픽셀 상한)  
- 모니터링: 히트율/평균 인코드 시간/오류율 메트릭

---

## 11) **마이그레이션 로드맵**

1. **자산 인벤토리**: 유형(사진/UI/애니메), 해상도, 평균 바이트, 메타 정책  
2. **표본 코호트** 1~5% 샘플링 → **AVIF/JXL/WebP 동시 변환** → PSNR/SSIM/주관검수 + 바이트/시간 로깅  
3. **포맷별 승자 선정**:  
   - 사진: AVIF/JXL 간 바이트/품질/디코드 속도 평가  
   - UI: WebP 444 vs AVIF 444  
   - JPEG은 가능한 **JXL 무손실 래핑**으로 “원본 보존 + 용량↓” 라인 병행  
4. **점진 배포**: `<picture>` + Nginx Accept 폴백 → 점유율 추적  
5. **역호환 저장**: 원본 유지, 신포맷 병행(청크스토리지/버전)  
6. **운영 규칙**:  
   - 바이트 목표 초과 시 **quality↓ 재시도**(이진 탐색)  
   - 아티팩트 감지 룰(엣지 대비/밴딩 지표) → 포맷 전환  
7. **리그레션 보호**: 주기적 샘플 재인코드/지표 비교

---

## 12) **배치 변환 스크립트** (디렉터리 일괄)

```bash
#!/usr/bin/env bash
set -euo pipefail
IN=./input
OUT=./out
mkdir -p "$OUT/avif" "$OUT/jxl" "$OUT/webp" "$OUT/jpegsafe"

for f in "$IN"/*.{jpg,jpeg,png}; do
  [ -e "$f" ] || continue
  base=$(basename "$f")
  name=${base%.*}

  # 메타 추출(선택): exiftool로 EXIF/ICC를 파일로 추출 후 재주입 가능
  # exiftool -icc_profile -b "$f" > "$OUT/$name.icc" || true
  # exiftool -exif:all -b "$f" > "$OUT/$name.exif" || true

  # AVIF
  avifenc "$f" "$OUT/avif/$name.avif" \
    --speed 6 --cq-level 28 --depth 10 --yuv 420 --jobs 8

  # JXL 일반
  cjxl "$f" "$OUT/jxl/$name.jxl" -d 1.2 --effort=7 --num_threads=8

  # JPEG → JXL 무손실 래핑(원본이 JPEG일 때만)
  case "$f" in
    *.jpg|*.jpeg) cjxl "$f" "$OUT/jpegsafe/$name.jxl" --lossless_jpeg=1 ;;
  esac

  # WebP
  cwebp -q 80 -m 4 -mt -metadata all "$f" -o "$OUT/webp/$name.webp"
done
```

---

## 13) **실전 튜닝 포인트**

- **색/감마 정합**: 전처리 연산은 **선형광**에서 수행 후, 최종 인코드는 **sRGB 코드**로.  
- **샘플링**: 사진은 420가 일반적, UI/폰트는 444 권장.  
- **비트심도**: 포토/HDR 워크플로우는 10bpc 이상.  
- **스피드 vs 품질**: CI에서는 빠른 프리셋, 야간 배치에서 고품질 프리셋.  
- **애니메**: 포맷마다 플레이어/프레임제어 차이 → 사내 뷰어/앱 테스트 필수.  
- **보안/안정성**: 입력 픽셀 수/마커 길이 상한, 디코드 타임아웃, setjmp 예외복구(라이브러리별).

---

## 14) **문제해결(트러블슈팅)**

| 증상 | 원인 | 해결 |
|---|---|---|
| 색이 과포화/퇴색 | ICC 손실, sRGB/gamma 오해 | ICC 유지/명시, sRGB 청크/프로필 부착 |
| 텍스트/아이콘 번짐 | 420 서브샘플링 | 444 또는 선명 프리셋(샤픈 후 인코드) |
| 용량 수렴 안됨 | 품질 고정 | **바이트 타깃** 이진탐색(AVIF/WebP에 적용), d/q 조정 |
| 디코드 느림 | 고노력·고심도·큰 해상도 | 썸네일/프리뷰 **스케일 디코드**/리사이즈, 타일링 |
| JPEG 보관 필요 | 포맷 교체로 법적/물류 이슈 | **JXL 무손실 래핑** 병행 저장 |

---

## 15) **바이트 타깃 인코딩**(이진 탐색 예시: WebP)

```cpp
bool EncodeWebP_TargetSize(const uint8_t* rgba, int w, int h, int stride,
                           size_t targetBytes, std::vector<uint8_t>& out)
{
    float lo=20.f, hi=95.f; // 검색 범위
    std::vector<uint8_t> tmp;
    for (int it=0; it<8; ++it) {
        float q = (lo+hi)*0.5f;
        if (!EncodeWebP_RGBA(rgba, w, h, stride, q, /*lossless=*/false, tmp)) return false;
        if (tmp.size() > targetBytes) hi = q; else lo = q;
    }
    out.swap(tmp);
    return true;
}
```

> AVIF도 `--max-quantizer`를 조정하며 동일 아이디어로 수렴 가능(시도·계측·재시도).

---

## 16) **성능 계측 로거**(시간/메모리)

```cpp
#include <chrono>
struct Timer { std::chrono::steady_clock::time_point t; void start(){t=std::chrono::steady_clock::now();} double ms()const{auto d=std::chrono::steady_clock::now()-t; return std::chrono::duration<double, std::milli>(d).count();} };

double MeasureAVIF(const uint8_t* rgba, int w,int h,int stride, std::vector<uint8_t>& bytes){
    Timer tm; tm.start(); EncodeAVIF_RGBA(rgba,w,h,stride,10,true,28,6,nullptr,0,nullptr,0,bytes); return tm.ms();
}
```

- 결과를 CSV/SQLite에 누적: `파일,포맷,바이트,인코드ms,PSNR,SSIM,날짜,...`

---

## 17) 라이선스/배포 노트(요지)

- **libavif/libaom/rav1e/SVT-AV1**, **libjxl**, **libwebp**는 일반적으로 **관대한 오픈소스 라이선스**입니다.  
- 제품 배포 시 **라이선스 고지 파일** 포함(OSS NOTICE).  
- 하드웨어/플랫폼별 **특허/규정**은 사내 법무/정책 기준을 따를 것.

---

## 18) 최종 정리

- **AVIF/JXL/WebP** 는 “**한 가지가 만능**”이 아닙니다. **콘텐츠 특성·운영 목표**에 따라 **혼용**이 현실적 최선.  
- **AVIF**: 사진·캡처, 10bit/HDR, 고효율.  
- **JXL**: **JPEG 무손실 래핑**·고화질·유연한 심도.  
- **WebP**: 넓은 실전 호환성·빠른 인코딩, UI/일러스트 강점.  
- **파이프라인**은 전처리(선형), 메타 보존, 포맷별 프리셋, 자동 계측/재시도, 배포 폴백/Accept 네고까지 **엔드투엔드**로 설계하세요.  
- 본문 CLI와 C++ 스니펫을 **ImageTool**에 붙이면, 하루 만에 **실전 운영**까지 갈 수 있습니다.
