---
layout: post
title: 영상처리 - Huffman 최적화 / Restart 마커
date: 2025-10-14 16:25:23 +0900
category: 영상처리
---
# 2. **Huffman 최적화 / Restart 마커**의 기본 정책  
*(네트워크 견고성 & 파일 크기 최적화 — TurboJPEG/jpeglib 실전 가이드)*

> 핵심 요약  
> - **Huffman 최적화(Optimize Huffman tables)**: 인코더가 **실제 데이터 통계**로 허프만 테이블을 새로 만들어 **같은 품질에서 2–10% 용량 절감**. 단, **CPU를 한 번 더** 씀(두 번 스캔).  
> - **Restart 마커(DRI + RSTn)**: 엔트로피 스트림을 **작은 조각(세그먼트)**으로 나눠 **오류 전파를 차단**하고, **하드웨어/멀티스레드 디코드**를 돕는 **구조적 체크포인트**. 대가: **수 바이트/세그먼트의 오버헤드**.

---

## 0. 왜 둘 다 중요할까?

- 저장/배포(**정적 파일**)에는 **최소 용량**이 중요 → **Huffman 최적화 ON**, **Restart OFF(=0)** 가 보통 최적.  
- 전송/스트리밍(**네트워크**)에는 **오류 격리/빠른 복구**가 중요 → **Restart ON**(간격을 짧게), Huffman 최적화도 함께 ON 권장.  
- 대규모 썸네일 파이프라인(갤러리/클라우드): **최적화 ON**, **Restart OFF**가 일반적.  
- **MJPEG(카메라/RTSP/RTP/HTTP multipart)**: 한 프레임에 **Restart ON**을 주면 **부분 손상에도 화면 유지** 가능.

---

## 1. JPEG 허프만 최적화(Optimize Coding)

### 1.1 개념
JPEG 기본 인코드는 **표준 허프만 테이블**이나 **품질 기반의 고정 테이블**을 쓸 수 있습니다.  
하지만 각 이미지의 실제 심볼 분포(DC/AC 계수 run-length, size 등)는 다르므로 **이미지별 최적 테이블**을 만들면 **엔트로피 길이가 짧아져** 파일이 줄어듭니다.

- libjpeg(jpeglib)에서는 `cinfo.optimize_coding = TRUE;` 한 줄로 **두 번 패스(수집→인코드)**를 수행.  
- TurboJPEG에서는 `TJFLAG_OPTIMIZE_CODING` 플래그.

> 체감  
> - 사진·문서 혼합 세트: **평균 3–7% 감소** 사례가 흔함(씬·설정에 따라 1–10%).  
> - CPU 오버헤드는 보통 **수 ms–수십 ms** 수준(이미지/플랫폼 의존).

### 1.2 코드 (TurboJPEG)
```cpp
struct TJEncodeOpts {
    int quality = 90;
    int subsamp = TJSAMP_420;      // 사진 420, 문서 444 권장 (앞 절 참고)
    bool progressive = true;
    bool optimize   = true;        // ★ 허프만 최적화
    bool accurate_dct = false;     // UI 문서는 true 고려
};

bool TJ_Encode_Optimized(const IppDib& img, const TJEncodeOpts& o,
                         std::vector<unsigned char>& out, std::string& err)
{
    out.clear(); err.clear();
    tjhandle tc = tjInitCompress();
    if (!tc) { err = "tjInitCompress failed"; return false; }

    int flags = 0;
    if (o.progressive) flags |= TJFLAG_PROGRESSIVE;
    if (o.optimize)    flags |= TJFLAG_OPTIMIZE_CODING;   // ★
    if (o.accurate_dct) flags|= TJFLAG_ACCURATEDCT; else flags|= TJFLAG_FASTDCT;

    unsigned char* buf=nullptr; unsigned long len=0;
    int rc = tjCompress2(tc,
        (unsigned char*)img.bits(), img.width(), img.stride(), img.height(),
        TJPF_BGRA,
        &buf, &len,
        o.subsamp, o.quality, flags);
    if (rc!=0){ err = tjGetErrorStr2(tc); tjDestroy(tc); return false; }

    out.assign(buf, buf+len);
    tjFree(buf);
    tjDestroy(tc);
    return true;
}
```

### 1.3 코드 (jpeglib)
```cpp
jpeg_compress_struct c; jpeg_error_mgr jerr;
c.err = jpeg_std_error(&jerr); jpeg_create_compress(&c);
jpeg_mem_dest(&c, &mem, &memSize);

c.image_width  = W; c.image_height = H;
#ifdef JCS_EXTENSIONS
c.in_color_space = JCS_EXT_BGRA; c.input_components = 4;
#else
c.in_color_space = JCS_RGB;      c.input_components = 3;
#endif

jpeg_set_defaults(&c);
// … 서브샘플링(444/422/420) 설정 함수 사용(이미 구현한 SetChromaSampling)
SetChromaSampling(c, JpegChroma::CS420);
jpeg_set_quality(&c, 90, TRUE);

// ★ 허프만 최적화 켜기
c.optimize_coding = TRUE;

jpeg_start_compress(&c, TRUE);
// scanline 쓰기 …
jpeg_finish_compress(&c);
jpeg_destroy_compress(&c);
```

---

## 2. Restart 마커 — **오류 격리와 병렬성**

### 2.1 구조
- **DRI(Define Restart Interval)**: `0xFFDD` 마커 + 길이(4) + **Ri(2바이트)**. 여기서 **Ri = 세그먼트당 MCU 수**.  
- **RSTn**: 세그먼트 경계마다 **2바이트 마커**(`0xFFD0`~`0xFFD7`) 삽입. **n은 0–7 순환**.

**효과**  
- 비트스트림 손상/패킷 유실이 있어도 **다음 RST 경계에서 DC 예측/비트 동기 재설정** → **오류 전파가 차단**.  
- 일부 디코더/하드웨어는 **세그먼트 단위 병렬/파이프라인** 처리(캐시/리셋 유리).

**대가**  
- 각 세그먼트마다 **2바이트** 오버헤드(+DRI 4바이트).  
- 세그먼트가 많을수록 파일이 약간 커짐(보통 0.1–0.6% 수준, 간격 설정에 따라 다름).

### 2.2 MCU 크기와 세그먼트 수
샘플링에 따라 **MCU 크기**가 달라집니다.

- 4:4:4 → MCU = **8×8**  
- 4:2:2 → MCU = **16×8**  
- 4:2:0 → MCU = **16×16**

폭/높이를 MCU 크기로 나눈 몫(올림)이 **MCU 컬럼 수** `MCU_cols`, **MCU 로우 수** `MCU_rows`.  
**Ri = 세그먼트당 MCU 수**, 보통 **“n MCU-rows마다”**를 직관적으로 잡습니다:
\[
Ri = MCU\_cols \times n
\]
세그먼트 개수:
\[
N_{\text{seg}} \approx \left\lceil \frac{MCU\_rows}{n} \right\rceil \times MCU\_cols
\]
오버헤드(바이트):
\[
\text{Overhead} \approx 4\ (\text{DRI}) + 2 \times N_{\text{seg}}
\]

> 예) 4000×3000, 4:2:0 → MCU(16×16), `MCU_cols=250`, `MCU_rows=188`.  
> `n=4`(=4 MCU-rows)라면 `Ri=250×4=1000`,  
> `N_seg ≈ ceil(188/4)×250 = 47×250 = 11750`,  
> **오버헤드 ≈ 4 + 2×11750 = 23504B ≈ 23KB**  
> (10MB 사진이면 0.2% 수준)

### 2.3 언제, 얼마 간격?

- **네트워크 스트리밍/MJPEG/원격 뷰**: **`n = 2~8` MCU-rows** 권장(오류 격리↑).  
- **중요 자료 보관(손상 복구성 우선)**: **`n = 8~16`** 정도.  
- **순수 파일 크기 우선**: **Restart OFF(=0)**.

> **프로그레시브 JPEG**에도 Restart는 유효. 다만 **스캔마다** 적용되므로 마커 수가 더 늘 수 있습니다. 오버헤드 영향은 여전히 수‰–수% 이내인 경우가 많습니다.

---

## 3. 구현 — jpeglib에서 Restart 간격 주기

### 3.1 “row 단위”로 지정하기
- `cinfo.restart_in_rows = n;` : **MCU row 단위**(권장, 직관적).  
- 내부에서 `Ri = n × MCU_cols`로 변환됩니다.

```cpp
// 네트워크 견고성 인코드 (예: 4 MCU-rows 간격, 허프만 최적화 동시 ON)
void Encode_NetworkRobust(const IppDib& img, int quality, JpegChroma cs,
                          int restart_rows, std::vector<uint8_t>& out)
{
    jpeg_compress_struct c; jpeg_error_mgr jerr;
    c.err = jpeg_std_error(&jerr); jpeg_create_compress(&c);
    unsigned char* mem=nullptr; unsigned long memSize=0;
    jpeg_mem_dest(&c, &mem, &memSize);

    c.image_width = img.width(); c.image_height= img.height();
#ifdef JCS_EXTENSIONS
    c.in_color_space = JCS_EXT_BGRA; c.input_components = 4;
#else
    c.in_color_space = JCS_RGB;      c.input_components = 3;
#endif
    jpeg_set_defaults(&c);
    SetChromaSampling(c, cs);        // 420/444 등
    jpeg_set_quality(&c, quality, TRUE);

    // ★ 허프만 최적화 + Restart in rows
    c.optimize_coding = TRUE;
    c.restart_in_rows = restart_rows;   // 예: 4

    jpeg_start_compress(&c, TRUE);

    // scanline 쓰기 …
    while (c.next_scanline < c.image_height){
        const uint8_t* s = (const uint8_t*)img.bits() + (size_t)c.next_scanline*img.stride();
#ifdef JCS_EXTENSIONS
        JSAMPROW rp = (JSAMPROW)s;
#else
        static std::vector<uint8_t> row; row.resize(img.width()*3);
        for (int x=0;x<img.width();++x){ row[x*3+0]=s[x*4+2]; row[x*3+1]=s[x*4+1]; row[x*3+2]=s[x*4+0]; }
        JSAMPROW rp = row.data();
#endif
        jpeg_write_scanlines(&c, &rp, 1);
    }

    jpeg_finish_compress(&c);
    out.assign(mem, mem+memSize);
    jpeg_destroy_compress(&c); free(mem);
}
```

### 3.2 (대안) `restart_interval`(MCU 개수 직접 지정)
- 특정 **바이트 목표 세그먼트 길이**를 추정해 직접 MCU 수로 넣고 싶다면:
```cpp
c.restart_interval = Ri; // MCU count per segment (DRI Ri)
```
- 보통은 `restart_in_rows`가 쉬워 유지보수에 유리합니다.

---

## 4. TurboJPEG에서의 Restart 제어

- **클래식 TurboJPEG API**(tjCompress2)는 **Restart 간격을 직접 노출하지 않습니다**.  
- **정밀 제어**(DRI 필요)는 **jpeglib** 경로를 사용하세요.  
- 단, **무손실 변환**(tjTransform) 시에는 기존 스트림의 Restart가 유지됩니다.

> 최신 버전의 “tj3 API”가 있는 환경에선 설정 키가 추가될 수 있으나, **프로젝트 휴대성**을 위해 이 문서에선 **jpeglib 경로**를 기본으로 권장합니다.

---

## 5. **정책 설계** — 실전 권장값

### 5.1 저장/배포(정적 파일)
- **Huffman 최적화**: **ON**  
- **Restart**: **OFF**  
- **Progressive**: 웹/클라우드 미리보기용이라면 ON 권장(하지만 스캔 수가 많아지면 파일 크기↑ 가능, 보통 1~5% 내외).

### 5.2 네트워크 스트리밍(RTP/RTSP/MJPEG/모바일 업로드)
- **Huffman 최적화**: **ON** (네트워크 대역폭 절감)  
- **Restart**: **ON, `restart_in_rows=2~8`**  
- **샘플링**: 사진 420, UI 444(필요 시)  
- **품질**: 네트워크 제약에 맞춰 Q=75~90  
- **프로그레시브**: 실시간 디코더/hw에 따라 상이. **Baseline** 선호하는 장비가 아직 많습니다.

> **현장 팁**  
> - LTE/5G 업로드 오류를 겪는 모바일 카메라 SDK: **`restart_in_rows=4`** 로 바꿔 **줄무늬/깨짐 범위**가 확연히 줄어드는 사례가 흔합니다.  
> - CCTV·차량 DVR: **하드웨어 JPEG 디코더**가 **RST 경계**에서 파이프라인을 재동기화하여 **글리치가 화면 전체로 번지지 않음**.

---

## 6. “세그먼트 크기 목표”로 **Restart 간격 자동 튜닝**

**목표**: 세그먼트(두 Restart 사이) 평균 크기를 **4–16KB** 수준으로 맞추고 싶다(오류 격리/캐시 친화).  
**방법**: **프로브 인코딩**(일단 Restart=0으로 한 번 압축) → **평균 바이트/MCU-row**를 추정 → 그 값으로 `restart_in_rows` 계산 → **재인코딩**.

```cpp
// 1) 프로브(재시작 없음) → 평균 바이트/MCU-row 추정
struct ProbeStats { double bytesPerMcuRow=0; int mcuRows=0, mcuCols=0; };

static ProbeStats Probe_McuStats(const IppDib& img, int quality, JpegChroma cs){
    // 임시 인코드(허프만 최적화 ON 권장)로 전체 바이트 얻기
    std::vector<uint8_t> tmp;
    {
        jpeg_compress_struct c; jpeg_error_mgr jerr;
        c.err = jpeg_std_error(&jerr); jpeg_create_compress(&c);
        unsigned char* mem=nullptr; unsigned long memSize=0;
        jpeg_mem_dest(&c,&mem,&memSize);
        c.image_width=img.width(); c.image_height=img.height();
#ifdef JCS_EXTENSIONS
        c.in_color_space=JCS_EXT_BGRA; c.input_components=4;
#else
        c.in_color_space=JCS_RGB;      c.input_components=3;
#endif
        jpeg_set_defaults(&c); SetChromaSampling(c, cs); jpeg_set_quality(&c, quality, TRUE);
        c.optimize_coding = TRUE; // 통계도 더 현실적으로
        jpeg_start_compress(&c, TRUE);
        while (c.next_scanline<c.image_height){
            const uint8_t* s=(const uint8_t*)img.bits() + (size_t)c.next_scanline*img.stride();
#ifdef JCS_EXTENSIONS
            JSAMPROW rp=(JSAMPROW)s;
#else
            static std::vector<uint8_t> row; row.resize(img.width()*3);
            for(int x=0;x<img.width();++x){ row[x*3+0]=s[x*4+2]; row[x*3+1]=s[x*4+1]; row[x*3+2]=s[x*4+0]; }
            JSAMPROW rp=row.data();
#endif
            jpeg_write_scanlines(&c,&rp,1);
        }
        jpeg_finish_compress(&c);
        tmp.assign(mem, mem+memSize);
        // MCU 격자 계산
        int hmax=0,vmax=0;
        for (int i=0;i<c.num_components;++i){ hmax=std::max(hmax, c.comp_info[i].h_samp_factor);
                                              vmax=std::max(vmax, c.comp_info[i].v_samp_factor); }
        int mcuW = hmax*8, mcuH = vmax*8;
        int mcuCols = (c.image_width  + mcuW-1)/mcuW;
        int mcuRows = (c.image_height + mcuH-1)/mcuH;
        double perRow = (mcuRows>0)? (double)tmp.size()/mcuRows : 0.0;

        jpeg_destroy_compress(&c); free(mem);
        return { perRow, mcuRows, mcuCols };
    }
}

// 2) 목표 세그먼트 바이트(예: 8KB)로 restart_in_rows 계산 → 재인코딩
bool Encode_WithTargetSegment(const IppDib& img, int quality, JpegChroma cs,
                              size_t targetSegBytes, std::vector<uint8_t>& out)
{
    auto st = Probe_McuStats(img, quality, cs);
    if (st.mcuRows<=0) return false;

    // perRow ≈ 한 MCU-row 당 평균 바이트
    // targetSegBytes / perRow ≈ 세그먼트 당 MCU-rows
    int rows = (int)std::max(1.0, std::floor(targetSegBytes / std::max(1.0, st.bytesPerMcuRow)));
    rows = std::min(rows, 32); // 너무 커지지 않게 클립(오버헤드/효과 균형)
    Encode_NetworkRobust(img, quality, cs, rows, out);
    return true;
}
```

> 실전에서는 **8KB** 전후를 자주 씁니다(패킷 손실 시 화면 ‘깨짐’ 범위를 미세화). 대역폭/품질/장비 특성에 맞춰 4KB~16KB로 튜닝하세요.

---

## 7. 시나리오별 레시피

### 7.1 **모바일 업로드(메신저/클라우드)**
- **사진**: 420, Q=85–92, **Optimize ON**, **Restart OFF**  
  → 전송 중 오류는 상위 전송 계층이 보통 재시도/체크섬으로 방지. 저장 크기가 우선.
- **저품질 네트워크**(MMS/이동체 환경): **Restart ON(`rows=4~8`)** 옵션 제공 → 화면 깨짐 국소화.

### 7.2 **MJPEG 스트리밍(CCTV/로봇)**
- Baseline, 420, Q=70–85, **Optimize ON**, **Restart rows=2~8**  
  → 손상 프레임의 **잔상 폭발 방지**. 하드웨어 디코더와 상성 좋음.

### 7.3 **스캔·문서 캡처(텍스트 중요)**
- 444, Q=90–95, **Accurate DCT**, **Optimize ON**, **Restart OFF**(정적)  
- 실시간 전송이면 **rows=4~8** 켜서 글리치 국소화.

---

## 8. 검증과 측정 포인트

- **파일 크기**: Optimize ON/OFF 차이(%, ms).  
- **Restart 간격**: rows=0/2/4/8 별 파일 크기(‰), 오류 주입 시 **손상 면적**.  
- **디코드 성능**: HW/NPU/SoC 환경에서 **Restart ON 시 파이프라인 이득**이 있는지.  
- **프로그레시브와의 상호작용**: 스캔 수↑이면 마커 수↑ → 오버헤드 측정.  
- **에러 복구성**: 비트 플립/절단 테스트에서 **다음 RST 경계** 이후 정상 복귀 확인.

---

## 9. MathJax — 오버헤드 근사식

세그먼트 개수 \(N_{\text{seg}}\) 와 오버헤드 \(O\):
\[
N_{\text{seg}} = \left\lceil \frac{MCU\_rows}{n} \right\rceil \times MCU\_cols,\quad
O \approx 4 + 2 \cdot N_{\text{seg}} \ \text{(bytes)}
\]
여기서 \(n\) 은 `restart_in_rows`.  
대형 사진에서도 \(O\) 는 보통 **수 kB ~ 수십 kB** 수준 → **수퍼센트(‰)** 오버헤드.

---

## 10. UI/도구 통합(권장 UX)

- [x] 체크박스 **“허프만 최적화(용량 감소)”** (기본 ON)  
- [x] 체크박스 **“Restart 마커(네트워크 견고성)”** + 콤보 **행 간격(2/4/8/16)**  
- [x] 고급: **“세그먼트 바이트 목표(8KB)”** → 자동 rows 계산  
- [x] **미리보기**: 크기/품질/오버헤드 라벨, 네트워크 모드 시 **손상 시뮬레이터**(가상 패킷 드롭)  
- [x] **프리셋 묶음**  
  - 저장 최적: *Optimize=ON, Restart=OFF*  
  - 네트워크 견고: *Optimize=ON, Restart rows=4, Baseline*

---

## 11. 요약 결론

- **허프만 최적화**: 항상 켜두고(특히 배치 변환/보관/클라우드), **무료 용량 절감**을 챙기자.  
- **Restart 마커**: **네트워크/스트리밍**/불안정 채널에선 **짧은 간격(2~8 rows)** 으로 **오류 격리**를 보장.  
- 파일만 저장할 땐 오버헤드 때문에 보통 **끄는 편**이 유리.  
- 대규모 시스템에선 **프로브→자동 rows 산정**으로 “**복구성/용량** 밸런스”를 일관되게 유지하라.

---

## 12. 간단 실험 코드(크기 비교 & 손상 시뮬)

```cpp
// 1) 허프만 최적화 ON/OFF 크기 비교
std::vector<uint8_t> a,b;
JpegEncodeOptions opt; opt.chroma=JpegChroma::CS420;

// OFF
optimize=false; 
{
    jpeg_compress_struct c; jpeg_error_mgr jerr; 
    c.err=jpeg_std_error(&jerr); jpeg_create_compress(&c);
    unsigned char* mem=nullptr; unsigned long memSize=0; jpeg_mem_dest(&c,&mem,&memSize);
    // … 초기화, SetChromaSampling, jpeg_set_quality(&c, 90, TRUE);
    c.optimize_coding = FALSE;     // OFF
    // … compress …
    a.assign(mem, mem+memSize); free(mem); jpeg_destroy_compress(&c);
}
// ON
{
    jpeg_compress_struct c; jpeg_error_mgr jerr; 
    c.err=jpeg_std_error(&jerr); jpeg_create_compress(&c);
    unsigned char* mem=nullptr; unsigned long memSize=0; jpeg_mem_dest(&c,&mem,&memSize);
    // … 동일 설정 …
    c.optimize_coding = TRUE;      // ON
    // … compress …
    b.assign(mem, mem+memSize); free(mem); jpeg_destroy_compress(&c);
}
printf("size OFF=%zu, ON=%zu, gain=%.2f%%\n", a.size(), b.size(),
       100.0*(1.0 - (double)b.size()/a.size()));

// 2) Restart rows=0/4/8 파일 크기 비교
for (int rows : {0,4,8}){
    std::vector<uint8_t> buf;
    // Encode_NetworkRobust(img, 90, CS420, rows, buf);
    printf("rows=%d -> size=%zu\n", rows, buf.size());
}
// 3) 손상 시뮬: 중간 바이트 1% 난수로 flip → 디코드 가능한 영역(오류 전파 범위) 관찰
```

> 실제 손상 시뮬은 파일을 메모리에 로드한 뒤 **엔트로피 구간 중간** 바이트를 무작위로 바꾸고, 프레임을 디코드해 **깨짐이 어디까지 퍼지는지** 비교하세요. `rows`가 작을수록 **피해가 국소화**됩니다.