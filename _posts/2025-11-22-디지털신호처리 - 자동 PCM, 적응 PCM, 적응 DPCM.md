---
layout: post
title: 디지털신호처리 - 자동 PCM, 적응 PCM, 적응 DPCM
date: 2025-11-22 18:25:23 +0900
category: 디지털신호처리
---
# 자동 PCM(APCM), 적응 PCM, 적응 DPCM(ADPCM)

## 예비사항 — PCM 계열 부호화의 공통 목표

PCM 계열의 목표는 간단하다.

1. **샘플링**으로 시간 이산화
2. **양자화**로 진폭 이산화
3. **부호화(비트열)** 로 전송/저장

그런데 현실 신호(특히 음성, 생체/센서, 계측)는

- 시간에 따라 에너지가 크게 변하고(비정상성),
- 인접 샘플끼리 강한 상관(예측 가능성)이 있다.

따라서 “항상 같은 스텝, 항상 같은 비트수로 정밀하게 때려박는” 표준 PCM은 비효율적이다.
이 비효율을 줄이려는 두 축이 생긴다.

- **진폭 통계가 바뀌는 것에 맞춰 양자화기를 적응** → **APCM / 적응 PCM**
- **샘플 간 상관을 예측으로 제거한 뒤 오차를 적응 양자화** → **ADPCM**

---

## 표준 PCM(Uniform / Companded) 복습

### 균일 PCM

샘플 \(x[n]\)을 스텝 \(\Delta\)로 양자화한 출력 \(Q(x[n])\)를 \(B\)비트로 부호화한다.

$$
Q(x)=\Delta\cdot \mathrm{round}\!\left(\frac{x}{\Delta}\right),\quad
\Delta=\frac{2X_{\max}}{2^B}.
$$

이상적 모델에서 양자화 오차는

$$
e \sim \mathcal{U}(-\Delta/2,\Delta/2),\quad
\sigma_e^2=\frac{\Delta^2}{12}.
$$

풀스케일 사인(피크 = \(X_{\max}\))의 SNR 근사:

$$
\mathrm{SNR} \approx 6.02B+1.76~\mathrm{dB}.
$$

### 고정 비균일 PCM(컴팬딩)

\(\mu\)-law/A-law는 “사람 귀/신호 분포가 로그형”이라는 사실을 이용해 **압축 → 균일 양자화 → 역압축**을 한다.
하지만 **압축 곡선이 고정**이라, 시간에 따라 통계가 크게 바뀌면 최적이 아니다.

---

## 자동 PCM = APCM(Adaptive PCM)의 개념

문헌에서 **Automatic PCM**은 보통 **APCM(Adaptive Pulse Code Modulation)** 을 의미한다.
즉, **예측 없이** PCM 양자화기의 **스케일/스텝을 신호 통계에 맞춰 적응**시키는 계열이다.

핵심 아이디어:

- 양자화기는 “입력 범위에 맞게” 동작할 때 가장 효율적이다.
- 그런데 입력 범위(에너지)가 시간에 따라 바뀐다.
- 그래서 **스케일 팩터(scale factor)** 또는 **스텝 \(\Delta\)** 를 계속 조절한다.

APCM은 크게 두 형태가 있다.

1. **즉시(instantaneous) / 음절(syllabic) 적응**
   - 샘플 또는 짧은 IIR 추정으로 스케일을 계속 갱신
2. **블록 적응(block adaptive)**
   - 블록 단위로 에너지/최대치를 추정해 그 블록 동안 같은 스케일을 사용

---

## 블록 자동 PCM(Block APCM) 구조와 수식

### 구조

블록 길이 \(L\)로 입력을 자른다.

- 블록 \(m\)의 샘플: \(x_m[0],\dots,x_m[L-1]\)
- 블록 스케일: \(g_m\)

정규화:

$$
u_m[n]=\frac{x_m[n]}{g_m}.
$$

정규화된 신호를 **고정 균일 PCM**으로 양자화:

$$
\hat{u}_m[n]=Q(u_m[n]) \quad(\text{B-bit uniform}).
$$

복원:

$$
\hat{x}_m[n]=g_m \hat{u}_m[n].
$$

즉 **“블록마다 자동 게인 조절을 하고 그 결과를 PCM”** 하는 셈이다.

### 스케일 추정(대표적 두 방법)

1) **최대 절대값 기반**

$$
g_m = \max_{0\le n<L} |x_m[n]|.
$$

→ 과부하(overload) 거의 없음, 대신 한 샘플의 피크에 끌려 **분해능이 거칠어질 수 있음**.

2) **RMS/분산 기반**

$$
g_m = c \sqrt{\frac{1}{L}\sum_{n=0}^{L-1}x_m[n]^2}.
$$

→ 평균적 에너지에 맞춰 스케일링. 단, \(c\) 선택이 중요하고 피크 과부하 위험이 있다.

### 스케일의 양자화(부가 비트)

실제 시스템은 \(g_m\)을 실수로 보낼 수 없다.

- 스케일 후보 집합 \(\{G_0,\dots,G_{K-1}\}\)을 미리 정의
- 블록마다 가장 가까운 \(G_k\) 선택
- 그 **인덱스 \(k\)** 만 전송

전송 비트:

- 샘플: \(B\)비트 × \(L\)개
- 스케일 인덱스: \(\log_2 K\)비트

즉, 유효 비트율:

$$
R = B + \frac{\log_2 K}{L} \quad (\text{bits/sample}).
$$

블록이 길수록 스케일 오버헤드는 줄지만 **적응이 느려진다.**

---

## GNU Octave: 블록 APCM 구현/검증

아래 실험은 “에너지가 블록마다 변하는 신호(예: 음성의 유성/무성 구간)”에서
표준 PCM과 APCM의 차이를 보여준다.

```octave
%% block_apcm_demo.m
clear; close all; clc; pkg load signal

%% 1) 테스트 신호 만들기: 블록마다 에너지가 바뀌는 톤
Fs = 8000;
t  = (0:Fs*2-1)/Fs;          % 2초
x  = [0.1*sin(2*pi*300*t(1:Fs)) ...
      0.9*sin(2*pi*300*t(Fs+1:end))];  % 뒤쪽이 더 큰 에너지

%% 2) 블록 APCM 파라미터
B = 6;                 % 샘플당 6비트 PCM
L = 80;                % 블록 길이 (10 ms @ 8kHz)
K = 32;                % 스케일 코드북 크기
Xmax = 1;              % 정규화 가정

% 균일 양자화 스텝(정규화 도메인)
Delta = 2*Xmax / (2^B);

% 스케일 코드북(로그 스페이스 예)
G = logspace(log10(0.02), log10(1.0), K);

%% 3) 함수들
function uq = q_uniform(x, Delta)
  uq = Delta * round(x/Delta);
end

function [xhat, k_idx] = apcm_block_encode_decode(x, B, L, G)
  Xmax = 1; Delta = 2*Xmax/(2^B);
  N = length(x);
  xhat = zeros(size(x));
  k_idx = zeros(1, ceil(N/L));

  blk = 1;
  for s = 1:L:N
    e = min(s+L-1, N);
    xb = x(s:e);

    % 스케일 추정(최대값)
    g_est = max(abs(xb)) + 1e-12;

    % 코드북에서 가장 가까운 스케일 선택
    [~, k] = min(abs(G - g_est));
    g = G(k);
    k_idx(blk) = k;

    % 정규화 + PCM
    u  = xb / g;
    uq = q_uniform(u, Delta);

    % 복원
    xhat(s:e) = g * uq;

    blk = blk + 1;
  end
end

%% 4) 표준 PCM(고정 스텝) vs APCM
% 고정 PCM은 전체 Xmax=1에 맞춰 스텝 고정
x_pcm = q_uniform(x, Delta);

[x_apcm, idx] = apcm_block_encode_decode(x, B, L, G);

%% 5) 성능 비교
e_pcm  = x_pcm  - x;
e_apcm = x_apcm - x;

SNR_pcm  = 10*log10(mean(x.^2)/mean(e_pcm.^2));
SNR_apcm = 10*log10(mean(x.^2)/mean(e_apcm.^2));

fprintf("Fixed PCM SNR = %.2f dB\n", SNR_pcm);
fprintf("Block APCM SNR = %.2f dB\n", SNR_apcm);
fprintf("Scale indices used (first few blocks): ");
disp(idx(1:10));

%% 6) 시각화(선택)
figure; subplot(3,1,1);
plot(x); title("original x[n]");
subplot(3,1,2);
plot(x_pcm); title("fixed PCM output");
subplot(3,1,3);
plot(x_apcm); title("block APCM output");
```

**관찰 포인트**

- 에너지가 작을 때는 APCM이 **스케일을 낮춰 분해능을 확보**한다.
- 에너지가 커지면 스케일이 따라가 **과부하를 막는다.**
- 블록 길이 \(L\)과 코드북 \(K\)를 바꾸면
  **오버헤드 vs 적응 속도 vs 왜곡**의 트레이드오프가 바뀐다.

---

## 적응 PCM의 즉시/음절(Instantaneous/Syllabic) 적응

블록 적응이 “천천히 따라가는” 방식이라면,
즉시/음절 적응은 **스텝을 매 샘플 또는 짧은 시간 상수로 계속 조절**한다.

대표 형태:

1. **스텝 적응 규칙**
   - 최근 양자화 코드의 크기가 크면 \(\Delta\)를 키워 과부하를 방지
   - 작으면 \(\Delta\)를 줄여 분해능 확보

2. **엔벨로프 추정 + 스케일링**
   - 입력의 절대값을 IIR로 추적
     $$a[n]=\alpha a[n-1]+(1-\alpha)|x[n]|$$
   - 정규화 \(u[n]=x[n]/a[n]\) 후 고정 PCM

즉시 적응은 **빠르지만 펌핑(pumping), 높은 구현 민감도**가 생길 수 있다.
그래서 실무에선 **블록 적응(저지연) + 약한 IIR 스무딩**을 혼합하기도 한다.

---

## DPCM 복습 — 예측으로 상관 제거

### 기본 수식

예측기 \( \hat{x}[n] \)로 현재 샘플을 예측하고,
오차 \(d[n]\)만 양자화한다.

$$
d[n]=x[n]-\hat{x}[n],\quad
\hat{d}[n]=Q(d[n]),\quad
\hat{x}[n]=\hat{x}[n]+ \hat{d}[n].
$$

간단한 1차 예측기:

$$
\hat{x}[n]=\beta \hat{x}[n-1].
$$

상관이 강한 신호라면 \(d[n]\)의 분산이 작아져
**같은 비트수로 더 높은 SNR**을 얻는다.

---

## 적응 DPCM = ADPCM의 개념과 표준 구조

ADPCM은

- **예측기(DPCM)**
- **적응 양자화기(APCM)**

를 결합한 구조다.
즉, **오차를 양자화하되 그 양자화기(또는 예측기)를 계속 적응**한다.

### 블록도(개념)

```text
x[n] ──▶( - )──▶ d[n] ─▶ [Adaptive Quantizer] ─▶ c[n]
        ▲                               │
        │                               ▼
     [Predictor] ◀── x̂[n] ◀─ [Inverse Q + Adapt]
```

- `Predictor`: \(\hat{x}[n]\) 생성
- `Adaptive Quantizer`: \(d[n]\)를 적응 스텝으로 양자화
- `Inverse Q + Adapt`: 복원된 \(\hat{d}[n]\)로 \(\hat{x}[n]\) 갱신
  동시에 스텝/예측 계수를 적응 규칙으로 업데이트

### 적응 대상

ADPCM의 적응은 보통 두 층으로 나뉜다.

1. **스텝 크기(quantizer step) 적응**
   - 급격한 기울기 구간에서 “slope overload”가 생기지 않도록
     스텝을 빠르게 키움
   - 평탄 구간에서는 스텝을 줄여 잡음을 낮춤

2. **예측기 계수 적응(선택적)**
   - 고정 예측기(단순 1차/2차) 또는
   - LMS류로 계수를 갱신하는 적응 예측기

ITU-T G.726 같은 표준 ADPCM은
**예측기 + 스텝 적응을 모두 포함**한다.

---

## ADPCM 스텝 적응(대표 규칙)

### Jayant형 지수 스텝 적응

양자화 인덱스 \(I[n]\)의 크기가 크면
오차가 크거나 신호 기울기가 급하다는 뜻 → 스텝을 키운다.

$$
\Delta[n+1]=\Delta[n]\cdot M(I[n]).
$$

여기서 \(M(\cdot)\)은 사전에 정해둔 **승수 테이블**이다.

- \(|I[n]|\) 작음 → \(M < 1\)
- \(|I[n]|\) 큼 → \(M > 1\)

이 “승수 테이블 + 지수적 적응” 구조가
ADPCM의 핵심 동작을 만든다.

### 왜 적응이 필요한가?

DPCM에서 오차가 갑자기 커지면

- 고정 스텝이면 포화(overload)되어 큰 왜곡이 나오고
- 스텝을 키우면 과부하는 줄지만 평탄 구간 잡음이 커진다.

적응은 이 둘을 “시간적으로 분리”한다.

- **급한 구간: 스텝↑**
- **잔잔한 구간: 스텝↓**

---

## GNU Octave: 간단 4-bit ADPCM(IMA 스타일) 구현

아래는 널리 쓰이는 “4-bit ADPCM”의 단순 버전이다.
(원리는 G.726과 동일한 **적응 오차 양자화**이며, 실습용으로 최소화했다.)

```octave
%% adpcm_ima_demo.m
clear; close all; clc; pkg load signal

%% 1) 입력 신호
Fs = 8000; t = (0:Fs-1)/Fs;
x = 0.8*sin(2*pi*300*t) + 0.2*sin(2*pi*900*t);
x = x / max(abs(x));     % 정규화

%% 2) IMA ADPCM 계열 테이블(대표)
step_table = [ ...
     7 8 9 10 11 12 13 14 16 17 19 21 23 25 28 31 ...
    34 37 41 45 50 55 60 66 73 80 88 97 107 118 130 143 ...
   157 173 190 209 230 253 279 307 337 371 408 449 494 544 598 658 ...
   724 796 876 963 1060 1166 1282 1411 1552 ];
index_table = [-1 -1 -1 -1 2 4 6 8];   % 코드 크기에 따른 스텝 인덱스 변화

%% 3) ADPCM 인코더/디코더
function [codes, xhat] = adpcm_encode_decode(x, step_table, index_table)
  N = length(x);
  codes = zeros(1,N);
  xhat  = zeros(1,N);

  % 초기 상태
  pred = 0;       % 예측값
  idx  = 1;       % 스텝 테이블 인덱스 (1-based)
  step = step_table(idx);

  for n=1:N
    d = x(n) - pred;

    % 4-bit 양자화 (sign + 3-bit magnitude)
    code = 0;
    if d < 0
      code = 8; d = -d;         % sign bit
    end

    % 크기 추정
    diffq = step/8;             % baseline
    if d >= step, code += 4; d -= step; diffq += step; end
    if d >= step/2, code += 2; d -= step/2; diffq += step/2; end
    if d >= step/4, code += 1; diffq += step/4; end

    % 복원된 오차
    if bitand(code,8)
      diffq = -diffq;
    end

    % 예측 갱신
    pred = pred + diffq;
    pred = max(min(pred,1),-1); % saturation
    xhat(n) = pred;
    codes(n)= code;

    % 스텝 적응
    idx = idx + index_table(bitand(code,7)+1);
    idx = max(min(idx, length(step_table)), 1);
    step = step_table(idx);
  end
end

[codes, x_adpcm] = adpcm_encode_decode(x, step_table, index_table);

%% 4) 성능
e = x_adpcm - x;
SNR = 10*log10(mean(x.^2)/mean(e.^2));
fprintf("4-bit ADPCM SNR ≈ %.2f dB\n", SNR);

figure;
subplot(2,1,1); plot(x); title("original");
subplot(2,1,2); plot(x_adpcm); title("ADPCM reconstructed");
```

**해석**

- `step_table`, `index_table`가 바로 **스텝 적응의 실체**
- 코드의 크기가 큰 구간에서 `idx`가 빠르게 증가하며
  스텝이 커지고 slope overload가 억제된다.
- 평탄 구간에선 `idx`가 감소하며 스텝이 작아져 잡음이 줄어든다.

---

## PCM vs APCM vs DPCM vs ADPCM 비교

| 구분 | 예측 사용 | 적응 사용 | 장점 | 단점/주의 | 대표 용도 |
|---|---:|---:|---|---|---|
| PCM | X | X | 구현 단순, 안정 | 비정상/상관 신호에 비효율 | 계측/기본 ADC |
| APCM(자동/적응 PCM) | X | O(스케일·스텝) | 비정상 진폭에 강함, 구현 적당 | 스케일 오버헤드, 예측 이득 없음 | 음성/센서 블록 부호화 |
| DPCM | O | X | 상관 신호에 비트 절약 | 고정 스텝의 slope overload | 저지연 예측 부호화 |
| ADPCM | O | O | 상관 + 비정상 모두 대응, 우수한 품질/비트 | 구조 복잡, 고정소수점 민감 | 전화/임베디드 오디오, 표준 코덱 |

---

## 실무 설계 체크리스트

1. **입력 통계 파악**
   - 비정상성(에너지 변동) 크면 APCM/ADPCM 우선
   - 상관(예측 가능성) 크면 DPCM/ADPCM 우선

2. **APCM 설계**
   - 블록 길이 \(L\):
     - 짧음 → 적응 빠름/오버헤드↑
     - 김 → 적응 느림/오버헤드↓
   - 스케일 코드북 \(K\):
     - 작음 → 오버헤드↓/스케일 왜곡↑
     - 큼 → 오버헤드↑/정밀↑

3. **ADPCM 설계**
   - 스텝 적응 테이블의 **공격/감쇠 속도**가 품질 결정
   - 예측기 차수↑ → 이득↑, 하지만 수치 민감↑
   - 복원 경로에서 **포화(saturation)** 정책 명확히

4. **고정소수점**
   - 스케일/스텝 갱신이 overflow/underflow에 취약
   - 내부 누산 폭, 라운딩 모드, 클리핑 규칙을 반드시 명시

---

## 정리

- **Automatic PCM = APCM**:
  예측 없이 **양자화 스케일/스텝을 자동 조절**해 비정상 진폭에 대응한다.
- **Adaptive DPCM = ADPCM**:
  DPCM의 예측 이득 위에 **적응 양자화(스텝/예측)를 결합**해
  상관 + 비정상을 동시에 줄인다.
- **Octave 실습**으로
  “적응이 왜 품질/비트율을 바꾸는지”를 수치로 확인할 수 있다.

다음 단계는

- G.726/G.722 같은 표준 ADPCM의 **정확한 예측/적응 수식**
- APCM에서 **스케일 추정의 최적화(Lloyd–Max형)**
- 고정소수점에서의 limit cycle / 잡음 해석
