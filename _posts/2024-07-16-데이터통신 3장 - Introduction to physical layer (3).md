---
layout: post
title: 데이터 통신 3장 - Introduction to Physical Layer (3)
date: 2024-07-16 19:20:23 +0900
category: DataCommunication
---
# 3.5 Data Rate Limits (데이터 전송률 한계)

데이터 전송률을 결정하는 핵심 요소는 **대역폭(B)**, **신호 레벨 수( \(L\) )**, **채널 품질(SNR)** 입니다. 이상/현실 채널 각각에 대해 **나이퀴스트**와 **섀넌**이 주는 상한을 이해하고, **두 식을 함께 써서** 실무적인 목표치를 정하는 것이 기본 절차입니다.

---

## 3.5.1 Noiseless Channel: Nyquist Bit Rate (무잡음 채널)

> **가정**: 잡음이 없고, 기저대역(low-pass) 채널에서 \(L\)개 레벨의 신호를 사용.

**나이퀴스트 비트율**
$$
R_\text{max} \;=\; 2B \,\log_2 L \qquad [\text{bps}]
$$

- \(B\): 채널 대역폭(Hz)  
- \(L\): 구분 가능한 **신호 레벨 수**(= 1심볼당 상태 수)  
- \(\log_2 L\): **심볼당 비트수**(bits/symbol)

**해석 팁**
- \(L\)을 늘리면 이론상 \(R_\text{max}\)가 증가하지만, 현실에서는 **레벨 간 간격**이 좁아져 **오검출/BER 증가** 위험이 커집니다(특히 SNR 낮을수록 불리).

---

## 3.5.2 Noisy Channel: Shannon Capacity (잡음 채널)

> **가정**: 가우시안 잡음 등 현실적 잡음 존재. 채널의 **이론적 용량 상한**.

**섀넌 용량**
$$
C \;=\; B \,\log_2 \!\bigl(1+\text{SNR}\bigr) \qquad [\text{bps}]
$$

- \(\text{SNR}\) 은 **선형비**(전력비)이어야 함. dB → 선형 변환:
  $$
  \text{SNR} \;=\; 10^{\tfrac{\text{SNR}_{\text{dB}}}{10}}
  $$

**해석 팁**
- 섀넌 식은 **최고 한계**일 뿐, 특정 BER/지연/복잡도 제약 하에서의 **실제 달성률**은 더 낮습니다.  
- **코딩 이득(FEC)**, **다이버시티**, **적응 변조/코딩(AMC)** 로 **상한에 근접**시킵니다.

---

## 3.5.3 Using Both Limits (두 한계를 함께 쓰기)

> **절차**: (1) 섀넌으로 **절대 상한** 확인 → (2) 목표 처리율 \(R\) 설정 → (3) 나이퀴스트로 필요한 **레벨 수 \(L\)** / **심볼율** 산출.

### 예제) B = 1 MHz, SNR = 63 (≈ 18 dB)
1) **Shannon 상한**
$$
C \;=\; 10^6 \log_2(1+63) \;=\; 10^6 \log_2 64 \;=\; 6\times 10^6\ \text{bps} \ (\approx 6\ \text{Mbps})
$$
2) **목표 전송률**: 현실 여유를 두고 \(R=4\ \text{Mbps}\) 가정  
3) **Nyquist로 \(L\) 산출**
$$
4\times 10^6 \;=\; 2 \times 10^6 \times \log_2 L \;\;\Rightarrow\;\; \log_2 L = 2 \;\Rightarrow\; L=4
$$

> 결론: **4레벨(2비트/심볼)** 변조로 **4 Mbps**는 이 환경에서 **합리적인 목표**.  
> 참고: 실제 BER 목표(예: \(10^{-6}\))를 만족하려면 **요구 SNR**(혹은 코딩율)을 더 면밀히 검증해야 합니다.

---

### 추가 예제) 주어진 B, SNR에서 필요한 \(L\)/심볼율 구하기

- 목표 \(R\)를 정했다면,
  $$
  \log_2 L \;=\; \frac{R}{2B}, \qquad R_\text{symbol} \;=\; \frac{R}{\log_2 L}
  $$
- **QAM** 계열이라면 \(L = M\)-QAM, \(\log_2 M\) 비트/심볼. \(M\)이 커질수록 **요구 SNR**↑.

---

### 파이썬 미니 계산기

```python
import math

def shannon_capacity(bw_hz, snr_db):
    snr = 10**(snr_db/10.0)
    return bw_hz * math.log2(1.0 + snr)

def nyquist_levels(R_bps, bw_hz):
    bits_per_symbol = R_bps / (2.0 * bw_hz)
    L = 2 ** bits_per_symbol
    return L, bits_per_symbol

B = 1_000_000
SNR_dB = 18.0  # ~63 linear
C = shannon_capacity(B, SNR_dB)
print("Shannon cap ≈", round(C/1e6,2), "Mbps")

R = 4_000_000
L, bpsym = nyquist_levels(R, B)
print("Required levels L ≈", round(L), "(bits/symbol≈", round(bpsym,2), ")")
```

---

# 3.6 Performance (네트워크 성능)

성능 평가는 **대역폭(bandwidth)**, **처리율(throughput)**, **지연(latency)**, **대역폭-지연곱(BDP)**, **지터(jitter)** 등 **상호 연관된 지표**로 이뤄집니다.

---

## 3.6.1 Bandwidth (대역폭)

- **Hz 단위 대역폭**: 합성 신호의 **주파수 범위** 혹은 채널이 **통과시키는 스펙트럼 폭**.  
- **bps 단위 “대역폭”**: 관용적으로 **전송 속도/용량**을 지칭(정확히는 **bit rate**/**link rate**).  
  > 혼동 주의: 물리/신호처리 맥락의 “대역폭(Hz)” vs 네트워킹 관용의 “대역폭(bps)”.

---

## 3.6.2 Throughput (처리율)

- **정의**: 관측 지점을 **단위 시간** 동안 **실제로 통과한 비트 수**.
  $$
  \text{Throughput} \;=\; \frac{\text{Total Bits}}{\text{Time}}
  $$

- **총 전송 비트 수**:
  $$
  \text{Total Bits} \;=\; N_\text{frames} \times \text{Avg Bits per Frame}
  $$

- **예제**: 링크의 정격 “10 Mbps”인데, **1분** 동안 평균 **1만 비트**짜리 프레임 **12,000개**가 통과 →  
  $$
  \text{Throughput} = \frac{12{,}000 \times 10{,}000}{60} = 2{,}000{,}000\ \text{bps} = 2\ \text{Mbps}
  $$
  ⇒ **정격 10 Mbps**라도 실제 **처리율은 2 Mbps**일 수 있음(오버헤드/경합/혼잡/재전송/매체공유 등).

**Goodput**(순수 유효 데이터율)도 구분: 헤더·재전송·프로토콜 오버헤드를 제외한 **응용 유효률**.

---

## 3.6.3 Latency (지연)

- **정의**: 송신 측이 **첫 비트**를 내보낸 순간부터, **전체 메시지**가 수신 측에 **도착 완료**될 때까지의 시간.

$$
\text{Latency} \;=\; \text{Propagation} \;+\; \text{Transmission} \;+\; \text{Queuing} \;+\; \text{Processing}
$$

### 구성 항목
- **Propagation time**(전파 지연):
  $$
  t_p \;=\; \frac{\text{Distance}}{\text{Propagation speed}}
  $$
  예) 2,000 km 광 링크, \(v\approx 2.0\times 10^8\ \text{m/s}\) → \(t_p\approx 10\) ms (편도)

- **Transmission time**(전송 시간): 프레임 **비트를 모두 밀어 넣는 시간**
  $$
  t_t \;=\; \frac{\text{Frame size (bits)}}{\text{Link rate (bps)}}
  $$
  예) 1,500바이트(=12,000비트) 프레임, 100 Mbps → \(t_t=120\,\mu s\)

- **Queuing time**(대기): 큐에서 **서비스 기다림**(혼잡 시 ↑)  
- **Processing**(처리): 헤더 파싱, 라우팅 룩업, 방화벽 룰 평가 등

> **RTT**(왕복 지연)는 보통 **라우팅/혼잡** 평가에 사용. **TCP 동작**에도 직접 영향.

---

## 3.6.4 Bandwidth–Delay Product (BDP)

**링크가 가득 차 있을 때, “파이프 안에 동시에 떠 있는 비트 수”**  
$$
\text{BDP} \;=\; \text{Bandwidth} \times \text{RTT} \qquad [\text{bits}]
$$

- **의미**: 최대 스루풋을 내려면 **송신 윈도우(버퍼)** 가 최소한 BDP 이상 필요(특히 고대역폭·고RTT 링크).  
- **예**: 1 Gbps, RTT 40 ms  
  $$
  \text{BDP} = 10^9 \times 0.04 = 40 \times 10^6\ \text{bits} \approx 5\ \text{MB}
  $$
  TCP 윈도우가 5 MB보다 작으면 **링크를 못 채움** → 스루풋 저하.

**파이썬 계산기**
```python
def bdp_bytes(bw_bps, rtt_ms):
    return int(bw_bps * (rtt_ms/1000.0) / 8)

print("BDP @1Gbps, 40ms =", bdp_bytes(1_000_000_000, 40), "bytes")
```

---

## 3.6.5 Jitter (지터, 지연 변동)

- **정의**: 패킷(또는 프레임) **도착 간격/지연**의 **변동성**.  
  실시간 음성/영상·산업 제어 등에서 **지터 억제**가 중요(버퍼링·플레이아웃 제어).

### 간단한 계산 예
- 패킷 \(i\)의 **관측 간격** \(\Delta_i = t^\text{rx}_i - t^\text{rx}_{i-1}\),  
  **이상적 간격**(발신) \(\delta = t^\text{tx}_i - t^\text{tx}_{i-1}\).  
- **지터 표본** \(J_i = |\Delta_i - \delta|\).  
  여러 표본 평균/백분위(95p) 등으로 지표화.

### RTP(Jacobson) 지터 근사(개념 참고)
- 지터를 지수평활로 추정:  
  $$
  J \leftarrow J + \frac{|D(i-1,i)| - J}{16}
  $$
  여기서 \(D\)는 **상대 지연 변화**(왕복이 아닌 **일방향**) 추정치.

**파이썬 지터 측정 스니펫(일방향 근사)**
```python
def simple_jitter(tx_times, rx_times):
    # tx_times, rx_times: 송/수신 타임스탬프 배열(동일 길이, 초 단위)
    import numpy as np
    tx = np.array(tx_times); rx = np.array(rx_times)
    delta_tx = np.diff(tx)
    delta_rx = np.diff(rx)
    jitter_samples = np.abs(delta_rx - delta_tx)  # 간격 변형
    return jitter_samples.mean(), np.percentile(jitter_samples, 95)

# 예시 데이터로 테스트해보면 평균/95퍼센타일 지터를 얻을 수 있음.
```

---

## 종합 예제 — 링크 계획 미니 워크플로

> **문제**: 20 MHz 채널(Hz), SNR = 25 dB(≈ 316), RTT ≈ 30 ms, 목표 **낮은 프레임 손실**로 **영상 스트림** 전송.

1) **Shannon 상한**  
   $$
   C = 20\times10^6 \log_2(1+316) \approx 20\times10^6 \times 8.31 \approx 166.2\ \text{Mbps}
   $$
   → 현실 목표 \(R\): **120 Mbps** (여유 반영)

2) **Nyquist(기저대역 가정)로 \(L\) 감**:  
   \(R=120\) Mbps, \(B=20\) MHz  
   $$
   \log_2 L = \frac{R}{2B} = \frac{120}{40} = 3 \Rightarrow L=8
   $$
   → **8레벨(3bit/심볼)** 이상 필요. (무선은 보통 패스밴드/OFDM/적응 MCS 사용)

3) **BDP**  
   $$
   \text{BDP} = 120\times10^6 \times 0.03 \approx 3.6\times10^6\ \text{bits} \approx 450\ \text{KB}
   $$
   → 송신/수신 버퍼(플레이아웃 포함) 최소 **수백 KB~수 MB** 권장.

4) **지터 예산**  
   - 코덱/플레이어 버퍼가 **수십 ms** 버퍼링 가능하도록 설정.  
   - 무선 레이어 ARQ/스케줄링이 지터에 영향 → **프레이밍/타이밍** 최적화.

---

## 빠른 체크 문제

1) \(B=2\) MHz, **무잡음**, \(L=16\). **나이퀴스트 비트율**은?  
   $$
   R_\text{max}=2B\log_2 L = 4\times10^6 \times 4 = 16\ \text{Mbps}
   $$

2) \(B=5\) MHz, \(\text{SNR}_{\text{dB}}=10\) dB. **섀넌 용량**은?  
   \(\text{SNR}=10\) → \(C=5\times10^6\log_2(11)\approx 17.3\) Mbps

3) **처리율**: 500바이트 프레임을 초당 30,000개 전송. 처리율은?  
   \(500\times8=4000\) 비트/프레임 → \(4000\times30{,}000=120\) Mbps

4) **지연**: 1,200바이트 프레임, 100 Mbps 링크, 1,000 km 광( \(2.0\times10^8\) m/s ), 큐잉/처리 무시.  
   - \(t_t = 9{,}600/10^8=96\,\mu s\)  
   - \(t_p = 1{,}000{,}000/2.0\times10^8 \approx 5\) ms  
   - 총 \(\approx 5.096\) ms

5) **BDP**: 50 Mbps, RTT 80 ms →  
   \(50\times10^6\times 0.08 = 4\times10^6\) bits ≈ **0.5 MB**

---

## 요약

- **Nyquist**: \(R_\text{max}=2B\log_2 L\) — **무잡음/기저대역** 가정하의 상한.  
- **Shannon**: \(C=B\log_2(1+\text{SNR})\) — **잡음 존재** 시 이론 상한.  
- **실무**: 섀넌으로 **상한 확인** → 목표 \(R\) 설정 → 나이퀴스트로 **필요 레벨/심볼율** 산출 → **BER/SNR/코딩**으로 검증.  
- **성능**: 대역폭(Hz/bps) 구분, **처리율≠정격 속도**, 지연=전파+전송+대기+처리, **BDP**로 윈도우/버퍼 산정, **지터**는 실시간 품질의 핵심.
