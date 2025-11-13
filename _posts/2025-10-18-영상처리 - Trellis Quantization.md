---
layout: post
title: 영상처리 - Trellis Quantization
date: 2025-10-18 16:25:23 +0900
category: 영상처리
---
# Trellis Quantization — JPEG **비트수(용량)** vs **화질(왜곡)**의 동적계획법 최적화

> **한 줄 요약**
> 기존의 “품질(Q)”·정수 반올림 양자화 대신, **레이트-디스토션(R-D) 최적화**로 각 8×8 DCT 블록의 계수 값을 **동적계획법(트렐리스, Viterbi)** 로 선택해 **용량을 더 줄이면서 화질 저하를 최소화**하는 기법.
> 실무에선 **mozjpeg(cjpeg)** 가 대표적 구현(옵션으로 활성화). libjpeg-turbo의 TurboJPEG API만으로는 직접 제어가 어렵고(테이블 주입까지만), **trellis는 일부 포크/도구**에서 제공됩니다.

---

## 0. 왜 Trellis Quantization인가?

- **문제점(기본 JPEG)**: 각 DCT 계수는 양자화 후 **독립적으로** 반올림된다. 그러나 실제 JPEG 비트스트림은
  **허프만 부호화 + (AC) 런-길이(RLE)** 로 엮여 있어, 한 계수의 선택이 **이웃 계수들의 런/코드비용**에 영향을 준다.
- **핵심 아이디어**: **전체 블록**의 비트수 \(R\)과 왜곡 \(D\)의 합
  \[
  J = D + \lambda R
  \]
  를 최소화하도록, **지그재그 순서**로 계수를 훑으며 가능한 양자화 레벨 후보들 중 **최적 경로**(트렐리스)를 찾는다.
- **효과**:
  - 용량 5–15% 절감(소스·옵션에 따라)
  - **EOB**(End-of-Block) 위치 최적화로 **블록 리스킹/엣지 링잉** 감소
  - **DC/AC** 각각에 다른 람다 \(\lambda\) / 후보폭을 두어 **자연 이미지 vs UI/문서**에 맞춤 최적화 가능

---

## 1. 수학적 모델(요지)

### 1.1 비용함수
- 각 블록의 원래 DCT 계수 \(C_k\) (지그재그 인덱스 \(k=0..63\))와 양자화 스텝 \(Q_k\).
- 정수 양자화 후보 \(q_k \in \mathbb{Z}\) 를 고르면, 복원 계수는 \(\hat{C}_k = q_k \cdot Q_k\).
- 왜곡(제곱 오차):
  \[
  D = \sum_k w_k \cdot (C_k - \hat{C}_k)^2
  \]
  (가중치 \(w_k\)는 **시각 가중(예: luma 고주파↑)** 도입 가능, 기본은 \(w_k=1\)).
- 레이트 추정 \(R\): 현재 **허프만 테이블**(DC/AC)을 기준으로,
  - DC: 크기 클래스 + 부호 amplitude 비트 길이
  - AC: (런, 크기) 심볼 + amplitude 비트 길이
  - **연속된 0**의 길이, EOB 여부가 비용을 크게 좌우.

> 트렐리스는 상태로 **“현재까지의 0런 길이(run)”**를 기억하며, 후보 \(q_k\) 를 선택할 때의 **추가비용 \(\Delta J_k\)** 를 계산하여 누적 최소비용 경로를 갱신한다.

### 1.2 동적계획(개략)
- 상태 \(s_k = (\text{runlen})\), 천이 \(s_k \to s_{k+1}\)
  - \(q_k = 0\)이면 runlen 증가
  - \(q_k \neq 0\)이면 그 직전 runlen과 합쳐 **(runlen, size)** 허프만 기호 비용 + amplitude비용 지불, runlen=0으로 리셋
- **EOB 최적화**: 끝자락에서 “더 비싼 소수의 작은 비영(非0) 계수” vs “조기 EOB” 중 R-D로 더 좋은 쪽 선택.

---

## 2. 툴/엔진 선택지

- **mozjpeg(cjpeg)**: Trellis Quantization/스캔 최적화/감각 품질 튜닝(예: `-tune-ssim`) 제공.
  - **명령행 사용**이 가장 현실적인 접근. (옵션 이름·유무는 빌드 버전에 따라 상이)
- **libjpeg-turbo / TurboJPEG**: 고속·호환성 초점. **trellis 옵션 없음**(2025-10 기준 일반 빌드).
  - 터보 경로만으로는 trellis 적용 불가 → **mozjpeg로 인코딩 단계만 대체**하는 하이브리드 파이프라인 권장.
- **직접 구현**: 연구/학습/특수 요구에 한해, 아래 **블록 단위 trellis 샘플** 참조(교육용 스케치).

---

## 3. 실무 사용(추천): **mozjpeg cjpeg** 파이프라인

> **시나리오**: `IppDib`/`IppImage` → **PPM/PGM** 임시 파일 → **mozjpeg cjpeg**로 trellis 인코딩 → JPEG bytes 회수

### 3.1 예시 커맨드 (버전에 따라 다를 수 있음)
```bash
# 자연사진(4:2:0), trellis+허프만 최적화+프로그레시브(선택)
cjpeg -quality 85 -sample 420 -optimize -progressive \
      -trellis \
      -outfile out.jpg in.ppm

# UI/문서(4:4:4), 선명도 유지(고주파 보호), trellis
cjpeg -quality 92 -sample 444 -optimize \
      -trellis \
      -quant-table 1 \
      -outfile out.jpg in.ppm
```
- `-trellis`: trellis quantization 활성화(일부 빌드에선 다른 플래그/기본 on일 수도 있음)
- `-optimize`: 허프만 최적화(크기↓). **절대적 재현성**이 목표면 off 고려.
- `-progressive`: 네트워크 감상성↑, 그러나 **바이트 재현성**은 스캔 스크립트 영향.
- `-quant-table`: mozjpeg 프리셋 테이블 선택(버전에 따라 의미 다름).
> 정확한 옵션은 `cjpeg -h` 또는 해당 배포의 README를 확인하세요. (빌드/배포에 따라 이름·기능이 상이)

### 3.2 C++에서 외부 인코더 호출(간단 스케치)
```cpp
// 1) IppDib → in.ppm 저장(BGRA → PPM, 또는 그레이면 PGM)
// 2) system() 또는 CreateProcess로 cjpeg 호출
// 3) out.jpg 읽어서 메모리로 가져오기

bool EncodeWithMozJpeg(const std::wstring& cjpegExe,
                       const std::wstring& inPpm,
                       const std::wstring& outJpg,
                       bool isUI)
{
    std::wstring cmd = L"\"" + cjpegExe + L"\" "
        + (isUI ? L"-quality 92 -sample 444 " : L"-quality 85 -sample 420 ")
        + L"-optimize -trellis -outfile \"" + outJpg + L"\" \"" + inPpm + L"\"";
    // CreateProcessW(...) 생략
    return true;
}
```

---

## 4. (고급) **mozjpeg 라이브러리** 직접 사용 스케치

모든 배포가 동일 API를 제공하진 않습니다. 일부 버전에선
`jpeg_c_set_bool_param / jpeg_c_set_int_param / jpeg_c_set_float_param` 과 같은 “확장 파라미터”로 trellis를 켭니다.

```cpp
// 주의: 아래 심볼/파라미터명은 mozjpeg 배포본에 따라 달라질 수 있습니다.
// 실제 사용 전 반드시 mozjpeg의 jpeglib.h / README를 확인하세요.
#ifdef MOZJPEG_EXT_PARAMS
    jpeg_c_set_bool_param(&cinfo, JBOOLEAN_TRELLIS_QUANT, TRUE);
    jpeg_c_set_bool_param(&cinfo, JBOOLEAN_TRELLIS_QUANT_DC, TRUE); // DC도 trellis
    jpeg_c_set_bool_param(&cinfo, JBOOLEAN_TRELLIS_EOB_OPT, TRUE);  // EOB 최적화
    jpeg_c_set_int_param (&cinfo, JINT_TRELLIS_NUM_LOOPS, 1);       // 반복 회수
    jpeg_c_set_float_param(&cinfo, JFLOAT_TRELLIS_LAMBDA, 1.0f);    // 람다 스케일(버전별 상이)
    // 곡선에 맞춘 튜닝(예: -tune-ssim)도 float 파라미터로 조정 가능할 수 있음
#endif
```

> ✅ **실전 팁**
> - **사진**: 4:2:0 + trellis + optimize + (선택) progressive
> - **UI/문서**: 4:4:4 + trellis(강도 약간↓) + optimize, progressive는 상황에 따라
> - **절대 재현성**: progressive OFF, optimize OFF(+고정 허프만), 고정 양자화 테이블과 함께 사용하는 편이 안전

---

## 5. (교육용) **블록 단위 Trellis** 미니 구현

> **목표**: 한 8×8 블록에 대해, **표준 허프만** 길이 근사와 **작은 후보 집합**(예: {0, q0, q0±1})으로
> \(J=D+\lambda R\) 최소화 경로를 찾아 **정수 양자화 계수 q[k]** 를 산출.
> (연구/학습용으로 충분히 동작하는 스케치입니다. 제품화엔 더 정교한 비용/스캔/경계처리 필요)

### 5.1 전제
- 입력: 공간영역 8×8 픽셀(0..255) 또는 이미 DCT된 값
- DCT: 표준 부동소수점 8×8 DCT 사용(예시 포함)
- 양자화표: 8비트 `Q[64]`
- 허프만 길이: **표준 DC/AC 테이블**의 **코드 길이**를 미리 계산해 **룩업**(예시 포함)
- 후보: 각 AC에 대해 `{0, q0, q0±1}`(범위 허용 시), DC는 `{q0, q0±1}`

### 5.2 코드
```cpp
#include <cmath>
#include <cstdint>
#include <vector>
#include <algorithm>
#include <limits>
#include <array>
using std::vector;

// -------------------- 8x8 DCT (간단 버전) --------------------
static const double PI = 3.14159265358979323846;
static void dct8x8(const double in[64], double out[64]) {
    auto C = [](int u){ return (u==0)? std::sqrt(1.0/2.0) : 1.0; };
    for (int v=0; v<8; ++v) {
        for (int u=0; u<8; ++u) {
            double sum=0.0;
            for (int y=0; y<8; ++y) for (int x=0; x<8; ++x) {
                sum += in[y*8+x] *
                    std::cos(((2*x+1)*u*PI)/16.0) *
                    std::cos(((2*y+1)*v*PI)/16.0);
            }
            out[v*8+u] = 0.25 * C(u) * C(v) * sum;
        }
    }
}
// 지그재그 인덱스
static const int ZZ[64] = {
     0, 1, 8,16, 9, 2, 3,10,
    17,24,32,25,18,11, 4, 5,
    12,19,26,33,40,48,41,34,
    27,20,13, 6, 7,14,21,28,
    35,42,49,56,57,50,43,36,
    29,22,15,23,30,37,44,51,
    58,59,52,45,38,31,39,46,
    53,60,61,54,47,55,62,63
};

// -------------------- 허프만 길이 근사 --------------------
// 표준 DC/AC 허프만 테이블에서 '코드 길이'를 미리 산출했다 가정(예시: size/rle → bitlen)
static uint8_t StdDCLumaSizeCodeBits[12] = {
    // size=0..11 (카테고리 별 코드 길이)
    2,3,3,3,3,3,4,5,6,7,8,9  // 단순 예시(실제 표와 약간 다를 수 있음: 교육용)
};
static uint8_t StdACLumaCodeBits[16][11] = { /* (run, size) → codebits, 교육용 스케치 */ };

// size(진폭)비트 길이: 부호 포함 "size" 비트
static inline int AmplitudeBitsLen(int v){
    int a = std::abs(v);
    int bits = 0; while (a) { a >>= 1; ++bits; }
    return bits; // size
}

// -------------------- Trellis Solver --------------------
struct TrellisResult {
    int16_t q[64];   // 최종 정수 양자화 계수
    double  J;       // 최종 비용
};

struct HuffCtx {
    // 허프만 길이 룩업: DC size→codebits, AC (run,size)→codebits
    const uint8_t* dc_size_bits;    // [0..11]
    const uint8_t (*ac_run_size_bits)[11]; // [run 0..15][size 1..10] 예시
};

// k=0..63(지그재그), 상태: runlen(0..15)
// DP[k][run] = (최소 J, 선택 q_k, 이전 run)
struct Node { double J; int q; uint8_t prev_run; };

TrellisResult TrellisQuantBlock(const double dct[64], const uint8_t Q[64],
                                const HuffCtx& H, double lambda)
{
    // 1) 기준 정수양자화 q0 = round(C/Q)
    int q0[64];
    for (int i=0;i<64;i++){
        int idx = ZZ[i];
        double val = dct[idx] / (double)Q[idx];
        q0[i] = (int)std::round(val);
    }

    // 2) 후보 세트 구성
    auto candidates = [&](int i)->std::array<int,5>{
        int c0=q0[i]; std::array<int,5> cs{c0,0,c0-1,c0+1,(int)std::copysign(1.0, (double)c0)};
        // 고유/정렬 간단화 생략
        return cs;
    };

    // 3) DP 테이블
    Node dp[64][16];
    for (int i=0;i<64;i++) for (int r=0;r<16;r++){ dp[i][r].J=std::numeric_limits<double>::infinity(); dp[i][r].q=0; dp[i][r].prev_run=0; }

    // 4) DC(k=0): run 상태 의미 없음. DC 예외 부호화(이전 블록 DC 예측은 여기선 0으로 가정)
    {
        int i=0, idx=ZZ[i]; // =0
        auto cs = candidates(i);
        for (int t=0;t<(int)cs.size(); ++t){
            int q = cs[t];
            double D = (dct[idx] - q*Q[idx])*(dct[idx] - q*Q[idx]);
            // DC 부호화 비용 근사: size=AmplitudeBitsLen(q) → DC 허프만 코드 길이 + size
            int size = AmplitudeBitsLen(q);
            if (size>11) size=11; // 교육용 클램프
            int R = H.dc_size_bits[size] + size;
            double J = D + lambda * R;
            // run=0으로 채움
            if (J < dp[i][0].J){ dp[i][0]={J,q,0}; }
        }
    }

    // 5) AC(k=1..63): run=0..15
    for (int i=1;i<64;i++){
        int idx = ZZ[i];
        auto cs = candidates(i);
        for (int run=0; run<16; ++run){
            if (dp[i-1][run].J==std::numeric_limits<double>::infinity()) continue;
            for (int t=0;t<(int)cs.size(); ++t){
                int q = cs[t];
                double D = (dct[idx] - q*Q[idx])*(dct[idx] - q*Q[idx]);
                double addR = 0.0;
                int nextRun = run;

                if (q==0){
                    // 0 → run 증가(최대 15로 클램프)
                    nextRun = std::min(15, run+1);
                    // 아직 부호화 안 함(후속 비영 계수에서 함께 부호화)
                    double J = dp[i-1][run].J + D; // R은 0
                    if (J < dp[i][nextRun].J) dp[i][nextRun] = {J, 0, (uint8_t)run};
                }else{
                    // run만큼 0, 다음 비영(크기 size) 심볼 부호화 + amplitude size 비트
                    int size = AmplitudeBitsLen(q); if (size<1) size=1; if (size>10) size=10; // 교육용
                    int codebits = H.ac_run_size_bits[run][size-1];
                    addR = codebits + size;
                    nextRun = 0;
                    double J = dp[i-1][run].J + D + lambda*addR;
                    if (J < dp[i][nextRun].J) dp[i][nextRun] = {J, q, (uint8_t)run};
                }
            }
        }
    }

    // 6) EOB(끝맺음) 비용 처리: 마지막 i에서 남은 run을 EOB로 마감하는 비용을 더해 최소 찾기
    // 교육용: 남은 run을 EOB(0x00)로 부호화한다고 가정하고 코드 길이 근사(표준 테이블에서 길이 상수라고 가정).
    auto EOB_bits = 4; // 교육용 상수(실제 표와 다를 수 있음)
    double bestJ = std::numeric_limits<double>::infinity();
    int bestRun=0;
    for (int run=0; run<16; ++run){
        double J = dp[63][run].J + lambda * EOB_bits;
        if (J < bestJ){ bestJ=J; bestRun=run; }
    }

    // 7) 역추적
    TrellisResult res; res.J = bestJ;
    int run = bestRun;
    for (int i=63;i>=0;--i){
        res.q[ZZ[i]] = (int16_t)dp[i][run].q;
        run = dp[i][run].prev_run;
    }
    return res;
}
```

> **주의(교육용 근사)**
> - 실제 허프만 길이는 **정확한 테이블**로 계산해야 하며, EOB/0xF0(ZRL)·런 경계·스캔 구조(프로그레시브)·DC 예측 등도 더 정밀히 다룹니다.
> - 후보 집합·람다 스케일링·가중치 \(w_k\) 조정으로 품질/용량 특성이 크게 달라집니다.

---

## 6. 람다 \(\lambda\)와 후보폭 **튜닝 가이드**

- **람다 크면(R↑ 가중)**: 더 작은 파일(하지만 세밀 도려냄↑)
- **람다 작으면(D↑ 가중)**: 화질 유지(파일 커짐)
- 실무는 **품질(Q)** ↔ **\(\lambda\)** 매핑 테이블을 내부적으로 정의.
  예) `Q=85 → λ=0.9`, `Q=92 → λ=0.6` (도메인별 경험값)
- 후보폭: `{0, q0, q0±1}`로 시작 → 아티팩트 보이면 고주파 몇 구간에 한해 `±2`까지 확장
- **Y vs Cb/Cr**: C 채널은 컬러 얼룩 방지를 위해 **람다↑(더 가혹)** 또는 후보폭 축소

---

## 7. **사진 vs UI/문서** 운영 레시피

- **사진(4:2:0)**
  - trellis **ON**, 허프만 최적화 ON, progressive ON(감상성↑)
  - C 채널은 후보폭 더 좁게/람다↑ → 컬러 노이즈 억제
- **UI/문서(4:4:4)**
  - trellis ON, progressive는 케이스별
  - Y 고주파 가중 \(w_k\) 상향, 람다↓ (엣지 살리기)
  - 결과를 **스크린샷·폰트** 패턴으로 A/B 평가(링잉/프린지 여부)

---

## 8. 검증 & 로그

- **품질 지표**: PSNR, **SSIM/MS-SSIM**(mozjpeg `-tune-ssim` 계열과 상성)
- **크기/시간**: 파일 바이트, 인코드 시간(트렐리스 루프 수 × 블록 수)
- **시각 점검**:
  - 하이콘트라스트 엣지 링잉 여부
  - 텍스트 테두리·세리프 흐림/울렁
  - 피부톤/그라디언트 밴딩

---

## 9. Pitfalls / 주의사항

- **바이트 재현성**: trellis + 허프만 최적화 + progressive는 **입력 데이터**에 따라 코드 길이가 바뀌므로
  **강한 재현성**(바이트 동일) 요구 시에는 비권장. (스캔 스크립트/허프만 테이블 고정 등 추가 조치 필요)
- **속도 비용**: 기본 양자화 대비 **추가 CPU**. mozjpeg는 다양한 최적화로 완화했지만 모바일 실시간엔 신중.
- **Restart 마커**: MCU 경계 재동기화는 trellis 자체와 직접 충돌하진 않지만, **비용모델**은 스캔/경계에 민감.
- **터보 파이프라인**: TurboJPEG 단독 사용 프로젝트는 **인코딩 단계만 mozjpeg로 우회**하는 **하이브리드**가 현실적.

---

## 10. “우리 프로젝트에 붙이기” 실전 시나리오

1. **ImageTool 편집 파이프라인**:
   - 편집/미리보기: TurboJPEG(BGRA)로 고속 디코드/표시
   - 최종 저장: **mozjpeg cjpeg**로 trellis 인코드(품질 프로필 Photo/UI)
2. **서버 썸네일러**:
   - 원본 → 다양한 사이즈 썸네일 생성 시 **trellis ON**으로 저장비 절감(트래픽↓)
   - UI 캡처 썸네일은 4:4:4 + 람다 보정으로 글자 품질 유지
3. **문서 스캐너 앱**:
   - 선명도 우선(4:4:4), trellis로 용량 제어, 두 값(Q/λ)만 노출하는 간단 UI

---

## 11. 요약

- Trellis Quantization은 JPEG의 **허프만/RLE 구조**를 고려한 **R-D 최적 양자화**로,
  **용량을 줄이면서** 시각 품질을 **잘 유지**하는 실전 기술.
- 가장 쉬운 도입은 **mozjpeg(cjpeg) 사용**이며, 파이프라인에서 **최종 인코딩 단계**만 치환해도 효과가 크다.
- R-D 람다·후보폭·가중치·샘플링(420/444)을 **도메인별로 표준화**해 일관된 결과를 얻자.
