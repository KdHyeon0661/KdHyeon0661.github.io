---
layout: post
title: 데이터 통신 - Bandwidth Utilization (1)
date: 2024-07-24 19:20:23 +0900
category: DataCommunication
---
# Bandwidth Utilization: Multiplexing and Spectrum Spreading

## Multiplexing (다중화)

**정의**: 하나의 물리 링크(매질) 용량을 **여러 논리 채널**로 나누어 다수의 송수신자가 **동시에**(혹은 빠른 시분할로) 쓰게 하는 기술.
- 송신측: **Multiplexer(MUX)**
- 수신측: **Demultiplexer(DEMUX)**
- 논리 단위: **channel** (주파수/시간/파장/코드 등으로 분리)

다중화의 주요 목적은 **링크 활용도 극대화**, **비용 절감**, **유연한 확장성**이다.

---

### Frequency-Division Multiplexing (FDM, 주파수 분할)

아날로그 대역통과 매질에서 각 사용자의 신호를 **서로 다른 반송 주파수**로 올려(변조) **주파수 축**으로 나누어 전송한다.

#### 핵심 아이디어

- 각 사용자 \(i\)의 메시지 대역폭 \(B_i\) → 반송파 \(f_{c,i}\)에 **상향변환**
- 결합된 합성 신호의 총 점유 대역폭은
  $$
  B_{\text{total}} \approx \sum_i \big( B_{\text{RF},i} + G_i \big),
  $$
  여기서 \(G_i\)는 **가드밴드(guard band)**.
- 수신부는 **밴드패스 필터**로 분리 후 각 반송 제거(복조).

#### 가드밴드 설계 감각

- 인접 채널 누설(ACI)·필터 롤오프·주파수 드리프트를 고려하여
  $$
  G_i \gtrsim \alpha\cdot B_{\text{RF},i} \quad (\alpha \text{는 시스템/필터 품질에 의존})
  $$

#### 전화/방송 계층 예시(역사적 구조)

- **Voice channel**: 약 \(4~\text{kHz}\)
- **Group**: 12채널 \(\Rightarrow\) 약 \(48~\text{kHz}\)
- **Super-group**: 5 그룹 \(\Rightarrow\) 약 \(240~\text{kHz}\)
- **Master-group**: 10 슈퍼그룹 \(\Rightarrow\) 약 \(2.4~\text{MHz}\)
- **Jumbo-group**: 6 마스터그룹 \(\Rightarrow\) 약 \(15.12~\text{MHz}\)

> **실무 감각**: 라디오/TV/CATV, 초기 이동통신(좁은 채널폭) 등에서 FDM이 핵심. 동적 주파수 할당이 필요한 경우 기지국이 **가용 채널**을 사용자에게 할당하고 통화 종료 시 회수한다.

#### 미니 계산(예)

- 10개 음성 채널, 각 **AM(DSB-FC)** 로 \(B=4\text{kHz}\)를 운반, 가드밴드 채널당 \(G=1\text{kHz}\)라면
  $$
  B_{\text{ch}} = 2B = 8\text{kHz},\quad
  B_{\text{total}} \approx 10\times(8+1)=90\text{kHz}.
  $$

---

### Wavelength-Division Multiplexing (WDM, 파장 분할)

광섬유에서 **서로 다른 파장(=주파수)**의 **광 캐리어**를 합쳐 한 가닥으로 전송. 프리즘/회절격자 원리로 분해가 가능.

#### 특징

- **FDM의 광 버전**: 파장 \(\lambda\)별 채널.
- **투명성**: IP/ATM/SONET/SDH/Ethernet 등 **다양한 비트율/프로토콜**을 파장별 실어 나름.
- **확장성**: **파장 추가**만으로 용량 증설 (장비/섬유 증설 최소화).
- **장거리**: 1550nm **C-band**에서 **EDFA**(광증폭기)로 중계 증폭 → **리피터 수 감소**.
- **DWDM/CWDM**: 채널 간격(그리드) 차이로 구분(고밀도/저밀도).

#### 간단 파장↔주파수 변환

$$
f = \frac{c}{\lambda}.
$$
예) \(\lambda=1550\,\text{nm}\Rightarrow f\approx 193.5\,\text{THz}\).

#### 미니 계산(예)

- 80채널 DWDM, 채널 간격 \(50\,\text{GHz}\), 파장대역 중앙 \(193.5\,\text{THz}\):
  총 스팬 \(\approx 80\times 50\,\text{GHz}=4\,\text{THz}\) 범위 내 설계 필요.

---

### Time-Division Multiplexing (TDM, 시분할)

**시간축**으로 링크를 나누어 **타임슬롯**에 사용자 데이터를 **교차 삽입**(interleaving)한다.

#### 공통 개념

- **프레임**: 한 바퀴 도는 주기 내 모든 채널의 슬롯이 들어있는 단위
- **프레이밍 비트**: 수신측 동기/경계 인지를 위한 패턴
- **인터리빙**: 송수신의 고속 스위칭으로 슬롯을 교차 배치

#### 동기식 TDM (Synchronous TDM)

- 각 채널은 **고정 슬롯**을 항상 가짐(데이터 유무 무관) → **빈 슬롯 낭비** 가능
- 장점: **간단/예측 가능/순서 유지 쉬움**
- 단점: **비효율**(저부하 시)

##### 빈 슬롯 완화 기법

1) **Multilevel Multiplexing**: 일부 입력의 데이터율이 정수배 관계일 때 레벨을 맞추어 계층 구성
2) **Multiple-slot Allocation**: 고속 입력에 **복수 슬롯** 할당
3) **Pulse Stuffing(비트 패딩)**: 비슷한 속도로 맞추기 위해 **더미 비트** 삽입

#### 통계적 TDM (Statistical TDM, Asynchronous TDM)

- **데이터가 있을 때만** 프레임에 삽입
- 각 슬롯에 **주소/식별자** 포함 필요
- 장점: 평균 부하가 낮을 때 **대역 효율↑**
- 단점: 피크 부하 시 **버퍼링/대기 지연(queuing)** 발생

##### 주소 비트 수

사용자 수 \(N\)일 때 주소 비트 수
$$
n_{\text{addr}} = \lceil \log_2 N \rceil.
$$

##### 출력 링크 속도 조건

집계 입력 평균율 \(\sum \bar{R_i}\)에 대해
$$
R_{\text{out}} > \sum \bar{R_i} \quad (\text{여유 포함}).
$$

#### 동기/준동기 관점(PDH/SDH 감각)

- **Plesiochronous(준동기)**: 각 입력이 자체 클럭 → **비트 스터핑**으로 정렬
- **Synchronous(완전 동기)**: 단일 기준 클럭에 동기 → 헤더/포인터로 하위 신호 정렬(개념적으로 SONET/SDH)

---

### Frame Synchronizing (프레이밍)

프레임 경계를 알리기 위해 **프레이밍 비트/패턴**을 주기적으로 삽입.
복호기는 이 패턴을 탐색해 **프레임 록**을 얻고 유지한다.

---

### 계층 & T/E-Line 감각

**미국식 DS 계층(역사적 표준)**:
- **DS-0**: 64 kbps (8 kHz 샘플 × 8비트 PCM)
- **DS-1**: 1.544 Mbps = **24 × DS-0** + 프레이밍(193비트/프레임 × 8000프레임/s)
  $$
  193\times8000 = 1{,}544{,}000\ \text{bps}
  $$
- **DS-2**: 6.312 Mbps = 4 × DS-1
- **DS-3**: 44.736 Mbps ≈ 7 × DS-2
- **DS-4**: 274.176 Mbps ≈ 6 × DS-3

**T-line**: DS 레벨을 물리 회선 서비스화(T1, T3 등).
**E-line**: 유럽식 위계(E1=2.048 Mbps 등).

> 아날로그 모뎀 접속(56 kbps급)도 **디지털 백홀**(DS/T/E)에 매핑되어 수송된다.

---

### Interleaving 작동 미니 예제(파이썬, 개념용)

```python
def sync_tdm_frame(chunks):
    """
    chunks: [b1, b2, ..., bK] 각 채널의 이번 프레임 페이로드(동일 길이 가정)
    반환: 인터리빙된 프레임 바이트열
    """
    assert len({len(c) for c in chunks}) == 1
    K = len(chunks)
    L = len(chunks[0])
    out = bytearray()
    # 단순 바이트 인터리빙(채널 순서 1..K 반복)
    for i in range(L):
        for k in range(K):
            out.append(chunks[k][i])
    return bytes(out)

def stat_tdm_pack(inputs, max_frame_bytes):
    """
    inputs: [(chan_id:int, payload:bytes), ...] - 유효 데이터만 수집
    max_frame_bytes: 프레임 최대 크기
    간단 주소+길이 헤더를 붙여 채울 수 있을 만큼만 채움
    """
    frame = bytearray()
    i = 0
    while i < len(inputs) and len(frame) < max_frame_bytes:
        cid, payload = inputs[i]
        header = bytes([cid & 0xFF, len(payload) & 0xFF])
        if len(frame) + 2 + len(payload) > max_frame_bytes:
            break
        frame += header + payload
        i += 1
    return bytes(frame), inputs[i:]
```

---

### 실전 시나리오 계산

#### 시나리오 A — 음성 30채널을 FDM으로

- 각 음성 \(B=3.4\,\text{kHz}\) (편의상 \(B\approx 4\,\text{kHz}\)), AM(DSB-SC) 가정
- 채널당 RF 대역폭 \(=2B\approx 8\text{kHz}\)
- 가드밴드 \(G=1\text{kHz}\) → 채널 폭 \(9\text{kHz}\)
- 총 대역폭 \(\approx 30\times 9=270\text{kHz}\)

#### 시나리오 B — 24채널 데이터를 동기식 TDM으로 (DS-1)

- DS-0 = 64 kbps, 24채널 → 24×64=1.536 Mbps
- 프레이밍 오버헤드 포함 1.544 Mbps
- 슬롯 낭비는 있지만 **고정 지연/간단 복구** 장점

#### 시나리오 C — 통계적 TDM

- 사용자 64명, 평균 10%만 활성 → 평균 동시 활성 6.4명
- 주소 비트 \(\lceil\log_2 64\rceil=6\)
- 패킷화/헤더 오버헤드 고려 시, 평균 처리율은 훨씬 **낮은 링크 속도**로도 만족
- 피크 시엔 **대기 지연**에 대비한 버퍼 설계 필요

---

## Spectrum Spreading (스펙트럼 확산)

> 질문 본문에는 다중화가 중심이었지만, 섹션 제목에 포함된 **확산(Spreading)**을 **보완**하여 정리한다. (부족한 내용 충원)

**스펙트럼 확산(SS)**: 원래 메시지 대역폭 \(B\)보다 훨씬 넓은 **확산 대역 \(B_s\)**로 퍼뜨려 송신함으로써 **간섭/도청/페이딩**에 강하고 **다중접속**을 가능케 하는 기술.

- **핵심 파라미터**:
  - **칩율(chip rate)** \(R_c\) (확산 코드 비트율)
  - **스프레딩 팩터(SF)** \(= \frac{R_c}{R_b}\) (데이터율 \(R_b\) 대비)
  - **처리이득(Processing Gain)**:
    $$
    G_p = \frac{B_s}{B} \approx \frac{R_c}{R_b} = \text{SF}.
    $$
  → **간섭/잡음에 대한 이득** 및 **코드 분리** 능력

---

### DSSS (Direct-Sequence Spread Spectrum)

데이터 비트를 **칩 시퀀스**(PN code, 골드코드 등)와 **칩 단위 XOR/곱**하여 **직접 확산**.

#### 송신

- 데이터 비트열 \(b[n]\in\{\pm1\}\)
- 칩 시퀀스 \(c[k]\in\{\pm1\}\) (길이 \(L=\)SF)
- 칩화된 심볼: \(s[k]=b[n]\cdot c[k]\) (한 비트 당 \(L\)칩)

#### 수신

- 동일 코드로 **상관(correlation)** 하여 **디스프레딩** → 원하는 신호만 합쳐지고, **타 신호/잡음은 퍼짐**.

#### 장점

- **협대역 간섭**에 강함(확산 → 간섭이 분산되어 평균화)
- **보안성/도청 난이도 증가**(코드 필요)
- **코드 분할 다중접속(CDMA)** 가능

#### 간단 파이썬 예(개념용)

```python
import numpy as np

def dsss_spread(bits, chip_seq):
    """
    bits: {+1,-1} 길이 N
    chip_seq: {+1,-1} 길이 L (한 비트 당 칩 길이)
    """
    L = len(chip_seq)
    out = np.repeat(bits, L) * np.tile(chip_seq, len(bits))
    return out

def dsss_despread(rx, chip_seq):
    L = len(chip_seq)
    rx = rx.reshape(-1, L)
    corr = (rx * chip_seq).sum(axis=1)  # 상관
    # 부호 판정
    return np.sign(corr)
```

---

### FHSS (Frequency-Hopping Spread Spectrum)

시간에 따라 **반송 주파수**를 **의사난수 시퀀스**에 따라 급격히 바꾸며 전송.

- **Slow/ Fast FH**: 한 비트에 여러 홉(빠름) vs 여러 비트에 한 홉(느림)
- **장점**: **내재적 주파수 다양성**, 특정 대역 간섭/도청 회피
- **단점**: 주파수 동기/홉 동기 복잡, 규제/스캔 시간 고려

---

### RAKE 수신기(개념)

다중경로 채널에서 DSSS는 서로 다른 지연 성분들을 **포수신기(핑거)**로 **상관 후 합성**(최대비결합).
→ 페이딩에 강하고, **멀티패스 이득**을 얻는다.

---

### 처리이득/BER 감각

**처리이득** \(G_p\)이 클수록 동일 SNR에서 더 낮은 BER 기대. 단, **코드 상관 특성**, **동기 정밀도**, **멀티유저 간섭(MAI)**가 실제 성능을 좌우.

---

### 다중화 vs 확산 — 관점 정리

- **다중화**: **여러 사용자/스트림을 분리**(주파수/시간/파장/코드 등)
- **확산**: **한 사용자 신호를 넓혀** **간섭/보안/다중접속**에서 이득
- 실제 시스템은 **혼합**: 예) WCDMA는 **코드 확산 + 슬롯/프레임 시간 구조**, OFDM/OFDMA는 **주파수 다중화 + 시간/코드/공간** 등

---

## 간단 체크 문제

1) FDM에서 채널 20개, 각 RF 대역폭 12 kHz, 가드밴드 2 kHz/채널이면 총 대역폭은?
   **해**: \((12+2)\times 20=280\ \text{kHz}\).

2) DWDM 80채널, 간격 50 GHz일 때 총 스펙트럼 스팬은?
   **해**: \(80\times 50\,\text{GHz}=4\,\text{THz}\).

3) 동기식 TDM에서 빈 슬롯 낭비를 줄이는 세 가지 방법을 쓰고, 각 개념을 한 줄로 요약하라.
   **해**: Multilevel(정수배 결합), Multiple-slot(고속 입력에 다중 슬롯), Pulse stuffing(더미 비트로 속도 정렬).

4) 통계적 TDM에서 사용자 128명일 때 주소 비트 수는?
   **해**: \(\lceil\log_2 128\rceil=7\).

5) DSSS에서 칩율 10 Mcps, 데이터율 100 kbps이면 스프레딩 팩터와 처리이득은?
   **해**: \(SF=10{,}000{,}000/100{,}000=100\), \(G_p\approx 100\).

---

## 핵심 요약

- **FDM/WDM/TDM**은 **주파수/파장/시간** 축으로 자원을 나누어 **여러 채널을 동시에** 운반.
- **동기식 TDM**은 단순/예측 가능하나 **빈 슬롯 낭비**, **통계적 TDM**은 평균 부하에서 효율적이나 **피크 지연**.
- **DS 계층**(DS-0/1/3 등)은 역사적 디지털 다중화 체계, 서비스(T/E-line)의 기반.
- **스펙트럼 확산(DSSS/FHSS)**은 **처리이득**으로 간섭·도청·페이딩에 강하고, **코드 분할 접속**을 가능케 함.
- 실제 시스템은 **다중화+확산**을 혼합하여 요구 사항(대역/지연/BER/EMI/보안/복잡도)을 동시에 만족시키는 지점을 찾는다.
