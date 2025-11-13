---
layout: post
title: 정보통신기사 - OSI vs TCP/IP, 흐름/오류제어
date: 2025-08-22 12:25:23 +0900
category: 정보통신기사
---
# OSI vs TCP/IP, 흐름/오류제어(ARQ/윈도우) 총정리

## 0. 큰 그림 한 장

- **OSI 7계층**: 이론적 모델(**표준화·용어 정리**)
- **TCP/IP 4~5계층**: 실제 인터넷 스택(**구현·배포**)
- **오류제어**는 주로 **링크계층(Ethernet FCS, 무선 FEC/ARQ)**와 **전송계층(TCP 재전송)**에서,
  **흐름제어**는 **수신 측 버퍼 보호**(TCP 윈도우 광고),
  **혼잡제어**는 **네트워크 내부 혼잡** 완화(TCP Slow Start/CA/CUBIC/BBR).
- **ARQ**는 프레임/세그먼트 단위 **재전송**으로 신뢰성 달성. **슬라이딩 윈도우**로 파이프라인 전송.
- **핵심 직감**: 링크 용량을 꽉 채우려면 **윈도우 ≥ BDP**(Bandwidth–Delay Product).

---

## 1. OSI 7계층 vs TCP/IP

### 1.1 계층 매핑 표

| OSI | 키워드 | PDU명 | 대표 프로토콜/예 |
|---|---|---|---|
| L7 응용(Application) | 서비스/데이터 | Data | HTTP, FTP, DNS, SIP, MQTT |
| L6 표현(Presentation) | 인코딩/암호 | Data | TLS, ASN.1/JSON/CBOR, JPEG |
| L5 세션(Session) | 세션/동기 | Data | (현실선 L7/L4가 흡수) |
| L4 전송(Transport) | 종단 신뢰/순서 | **Segment/Datagram** | **TCP**, **UDP**, QUIC(UDP기반) |
| L3 네트워크(Network) | 라우팅/주소 | **Packet/Datagram** | **IP(v4/v6)**, ICMP, ARP(NB) |
| L2 데이터링크(Data Link) | 프레이밍/접근 | **Frame** | **Ethernet**, 802.11 MAC, PPP, HDLC |
| L1 물리(Physical) | 신호/부호/매체 | Bit | UTP/광, 전송선, 변조/라인코딩 |

> **TCP/IP 모델**은 보통 **응용/전송/인터넷/네트워크접근(=L2+L1)**의 4층 또는 **5층(링크/물리 분리)**으로 설명.

### 1.2 캡슐화/디캡슐화와 주소체계

- **캡슐화**: App Data → (L4 헤더)Segment → (L3 헤더)Packet → (L2 헤더/트레일러)Frame → (L1)Bit
- **주소계층**
  - L2: **MAC**(국소 링크)
  - L3: **IP**(종단간 라우팅)
  - L4: **포트**(프로세스 식별)
- **표준 오류검출**
  - L2: **FCS/CRC**(예: Ethernet CRC-32)
  - L3: IPv4 **헤더 체크섬**(IPv6 없음)
  - L4: **TCP/UDP 체크섬**(IPv6에서도 필수; **의사헤더** 포함)

---

## 2. 오류제어 vs 흐름제어 vs 혼잡제어

- **오류제어(Error Control)**: 에러 검출/정정/재전송(ARQ).
  - **링크계층**: 무선(L2 ARQ, Hybrid-ARQ), 유선(Ethernet은 재전송 없음·상위에 맡김).
  - **전송계층**: TCP 재전송(ACK/타임아웃/SACK).
- **흐름제어(Flow Control)**: **수신 버퍼 보호**.
  - TCP **수신창(rwnd)** 광고로 송신 속도를 제한.
- **혼잡제어(Congestion Control)**: **네트워크 혼잡(큐 오버플로)** 방지.
  - TCP **혼잡창(cwnd)**, Slow Start/CA/FR/FR 등, CUBIC/BBR 등 알고리즘.

> **TCP 송신 가능 윈도우** = `min(rwnd, cwnd)` (단, 아직 미확인 데이터에 한해).
> **애드밴스 윈도우(awnd)** 같은 표현도 문헌에 존재하지만, 실무 핵심은 **rwnd vs cwnd**.

---

## 3. ARQ(Automatic Repeat reQuest)와 슬라이딩 윈도우

### 3.1 정지-대기(Stop-and-Wait, SW)

- 1개 프레임 전송 → **ACK 기다림** → 다음 프레임.
- **순환 비트**(0/1)로 중복검출. 단순하지만 **대역지연 제품(BDP)** 큰 링크에서 **효율↓**.

**주기 시간**
$$
T_\text{cycle} = T_\text{tx} + T_\text{prop, fwd} + T_\text{ack} + T_\text{prop, rev}
$$
- $T_\text{tx}=L/R$ (프레임 L비트, 링크율 R), $T_\text{prop}$= 전파지연.

**성능(오류확률 $p$, 성공확률 $1-p$)**
$$
\text{Throughput}_\text{SW} = \frac{(1-p)\,L}{T_\text{tx}+T_\text{ack}+T_\text{prop,f}+T_\text{prop,r}}
$$
> 해석: 성공 1회당 평균 전송 시도 $1/(1-p)$ → 평균 주기 늘어남 → 유효 처리량은 $(1-p)$가 곱해짐.

**이용률(무오류 근사, ack 무시, 왕복전파 $2T_p$)**
$$
U_\text{SW} \approx \frac{T_\text{tx}}{T_\text{tx}+2T_p}
= \frac{1}{1+2a},\quad a=\frac{T_p}{T_\text{tx}}
$$
- $a$가 클수록(=긴 RTT/짧은 프레임) **U ↓**.

### 3.2 Go-Back-N (GBN)

- **윈도우 W**개까지 연속 전송. 누락 검출 시 **해당 프레임 이후 모두 재전송**.
- **누적 ACK**(cumulative ACK). 구현 간단, 손실 시 **연쇄 재전송**(낭비↑).

**무오류 이용률 근사**
$$
U_\text{GBN} \approx
\begin{cases}
\frac{W}{1+2a}, & W<1+2a\\
1, & W\ge 1+2a
\end{cases}
$$
- 파이프라인을 가득 채우려면 **$W \ge 1+2a$** 필요(= **BDP 조건**).

**에러 존재 시 직감**: $p$가 작을수록 SR과 격차↓, $p$가 크면 **GBN 손해** 커짐.

### 3.3 Selective Repeat (SR)

- **손실된 프레임만 선택 재전송**, 수신측 **아웃오브오더 수용/버퍼링**.
- 가장 효율적이지만 수신 버퍼/번호공간 관리 필요(**윈도우 ≤ 시퀀스공간/2** 보통).

**무오류 이용률**: GBN과 동일 조건에서 **1**에 더 빨리 수렴(실제로는 유사).
**손실 시**: 선택 재전송 덕에 **처리량 저하 최소화**.

### 3.4 타임아웃 & 재전송

- **재전송 타이머**는 **RTT 추정** 기반(§5).
- **누락/중복 ACK**: GBN은 누적 ACK n 도착 시 n까지 모두 확인.
- **SR**은 개별 ACK(Selective ACK) 필요. (TCP는 **SACK 옵션**으로 SR에 가깝게 동작)

---

## 4. BDP와 윈도우 크기

### 4.1 BDP(Bandwidth–Delay Product)

$$
\mathrm{BDP}[\mathrm{bits}] = \text{대역폭 }R[\mathrm{bit/s}] \times \text{RTT }T[\mathrm{s}]
$$
- 파이프라인을 채우려면 **송신 중 미확인 데이터 ≥ BDP** 필요.
- 프레임/세그먼트 크기 $L$일 때 **윈도우(세그먼트 수)** 최소:
$$
W_\text{min} \approx \left\lceil \frac{\mathrm{BDP}}{L} \right\rceil
$$

**예)** 100 Mb/s, RTT 50 ms, MSS 1460 B(=11680 b)
$\mathrm{BDP}=100\times10^6 \times 0.05 = 5\times 10^6$ b → $W\approx 5\mathrm{e}6/11680\approx 428$ 세그먼트.
→ **윈도우 스케일** 없으면 불가(기본 65,535B 한계). **윈도우 스케일 옵션** 필수.

### 4.2 Stop-and-Wait의 병목

- 사실상 $W=1$. 위 예에서 활용률 $U\approx 1/(1+2a)$로 **극저**. → **슬라이딩 윈도우 필수**.

---

## 5. TCP 핵심 메커니즘 (흐름·오류·혼잡·타이머)

### 5.1 흐름제어: **수신창(rwnd)**

- 수신자는 **Advertised Window**(가용 버퍼)로 송신자 속도 제한.
- **Silly Window Syndrome(SWS)** 회피: 너무 작은 윈도우 광고/전송 금지(Clark’s solution), **Nagle**와 조합.

### 5.2 오류제어: ACK, 재전송, SACK

- **누적 ACK**: 마지막 연속 수신 바이트까지 승인.
- **타임아웃 재전송**, **Fast Retransmit**(중복 ACK 3회) → 빠른 손실 복구.
- **SACK** 옵션: 도착한 구간 범위를 전달 → **선택 재전송**(SR 유사).

### 5.3 혼잡제어: cwnd

- **Slow Start**: `cwnd`를 **지수** 증가(ACK당 MSS 추가).
- **Congestion Avoidance**: 선형 증가.
- **Fast Recovery**: Reno류, **CUBIC**(현대 리눅스 기본), **BBR**(대역폭·RTprop 추정 기반).

> 송신 허용 데이터 ≈ `min(rwnd, cwnd)` 바이트. 링크 꽉 채우려면 **둘 다 BDP 이상**이어야.

### 5.4 RTT 추정과 RTO (Jacobson/Karels + Karn)

- **SRTT/RTTVAR** 갱신:
$$
\begin{aligned}
\text{Err} &= \text{RTT}_\text{sample}-\text{SRTT}\\
\text{SRTT} &\leftarrow \text{SRTT} + \alpha\cdot \text{Err}\quad(\alpha\approx 1/8)\\
\text{RTTVAR} &\leftarrow \text{RTTVAR} + \beta\cdot (|\text{Err}|-\text{RTTVAR}),\ \beta\approx 1/4\\
\text{RTO} &= \text{SRTT} + \max(G, K\cdot\text{RTTVAR}),\ K=4
\end{aligned}
$$
- **Karn’s Algorithm**: 재전송된 세그먼트의 RTT는 **샘플에서 제외**.

### 5.5 Nagle, Delayed ACK

- **Nagle**: 작은 패킷을 모아 전송(소형 세그먼트 폭발 방지). **대화형 지연** 유발 가능 → 지연 민감 앱은 `TCP_NODELAY`.
- **Delayed ACK**: ACK 지연(예 40ms)으로 ACK 수 절감. Nagle와 **상호작용** 주의.

---

## 6. 수치 예제

### 예제 1) Stop-and-Wait 효율
- 링크 10 Mb/s, 프레임 10 kB, 거리 왕복 전파 40 ms(=RTT), ACK 40 B 무시.
- $T_\text{tx} = 10\,\text{kB} \times 8 / (10\,\text{Mb/s}) = 8\,\text{ms}$
- $U \approx \frac{8}{8+40} = \mathbf{0.167}$ (≈16.7%). → **비효율**, 슬라이딩 윈도우 필요.

### 예제 2) GBN 윈도우로 파이프 채우기
- 위 링크에서 $a=T_p/T_{tx}=20/8=2.5$, $1+2a=6$.
- **$W \ge 6$**이면 **U→1**(무오류 근사). 실제는 ACK/헤더/손실 고려해 약간↓.

### 예제 3) BDP로 TCP 윈도우 요구량
- 1 Gb/s, RTT 30 ms, MSS 1448 B
  - BDP = $1\mathrm{e}9\times 0.03 = 30\,\mathrm{Mb} = 3.75\,\mathrm{MB}$
  - 세그먼트 수 ≈ $3.75\text{MB}/1448 \approx 2666$ → **윈도우스케일 필요**(`wscale`로 2^n 배 확장).

### 예제 4) 손실이 있는 SW 처리량
- 손실확률 $p=0.01$, 예제1의 분모 동일:
  $\text{Throughput}=(1-p)\cdot L /(T_\text{cycle})=0.99\cdot 80\,\text{kbit}/48\,\text{ms}\approx \mathbf{1.65\,Mb/s}$.
  (무손실 1.67 Mb/s 대비 소폭 감소)

### 예제 5) TCP cwnd 성장(개념)
- 초기 `cwnd=1 MSS`, RTT마다 2배(ACK마다 +1 MSS) → 몇 RTT 안에 BDP 근처 도달.
- 도달 후 **선형 증가**(혼잡회피), 손실 시 **감소**.

---

## 7. Python 미니 계산기

```python
import math

# 1. Stop-and-Wait 처리량
def sw_throughput_bps(R_bps, L_bytes, RTT_s, p_loss=0.0, ack_bytes=0):
    Ttx = (L_bytes*8)/R_bps
    Tack = (ack_bytes*8)/R_bps
    Tcycle = Ttx + RTT_s + Tack
    return (1.0 - p_loss) * (L_bytes*8) / Tcycle

# 2. GBN 무오류 이용률 근사
def gbn_utilization(W, R_bps, L_bytes, RTT_s):
    Ttx = (L_bytes*8)/R_bps
    a = (RTT_s/2)/Ttx  # one-way prop/Ttx
    if W < 1 + 2*a:
        return W/(1+2*a)
    return 1.0

# 3. BDP와 윈도우
def window_for_bdp(R_bps, RTT_s, MSS_bytes):
    bdp_bits = R_bps*RTT_s
    return math.ceil(bdp_bits/(MSS_bytes*8))

# 4. Jacobson/Karels RTO 갱신
def rto_series(samples, alpha=1/8, beta=1/4, G=0.001, K=4):
    SRTT = samples[0]
    RTTVAR = samples[0]/2
    out = []
    for x in samples[1:]:
        err = x - SRTT
        SRTT += alpha*err
        RTTVAR += beta*(abs(err)-RTTVAR)
        RTO = SRTT + max(G, K*RTTVAR)
        out.append((SRTT, RTTVAR, RTO))
    return out

# Demo
R=10e6; L=10*1024; RTT=0.040
print("SW throughput (10Mb/s,10KB,40ms):", round(sw_throughput_bps(R,L,RTT)/1e6,3),"Mb/s")
print("GBN U(W=6):", round(gbn_utilization(6,R,L,RTT),3))
print("Window for BDP (1Gb/s,30ms,MSS1448):", window_for_bdp(1e9,0.03,1448))
print("RTO series:", [tuple(round(v,4) for v in t) for t in rto_series([0.05,0.055,0.052,0.07,0.06])])
```

---

## 8. 링크/계층별 오류·흐름제어 배치 감

- **L1/L2**: 비트/프레임 수준 오류 → **FEC/ARQ/CRC**. 무선은 **HARQ(Soft-Combining)**로 지연 낮춤.
- **L3**: IPv6는 헤더 체크섬 없음(상하위에 맡김). ICMP는 진단용.
- **L4(TCP)**: 종단 신뢰, **재전송·순서복원**.
- **애플리케이션/UDP**: 필요시 **애플리케이션 ARQ/FEC**(예: 영상 전송의 RTP + FEC, QUIC은 UDP 위 신뢰성 구현).

---

## 9. TCP 고급 토픽 압축 정리

- **윈도우 스케일**: `WSopt = 2^s`, `AdvertisedWindow << s`로 확장.
- **SACK**: 구간 단위로 도착 범위 알림 → 재전송 효율 ↑.
- **ECN**: 활성화 시 라우터가 **마크**로 혼잡 신호(드롭 없이) 전달.
- **CUBIC**: RTT 독립적 성장 함수로 고속/장거리 친화.
- **BBR**: 대역폭/최저RTT를 추정해 송신율 설정(버퍼블로트 저감).

---

## 10. 자주 틀리는 포인트(정오표)

1) **흐름제어 vs 혼잡제어** 혼동 금지: **rwnd(수신버퍼)** vs **cwnd(네트워크 상태)**.
2) **Stop-and-Wait**는 RTT 큰 링크에서 **거의 사용 안 함**(교육용). 실제는 **슬라이딩 윈도우**.
3) **GBN vs SR**: 손실 시 **GBN은 꼬리까지 재전송**, **SR은 선택 재전송**.
4) **BDP** 계산에서 **바이트/비트 단위** 혼동 주의.
5) **IPv6에는 IP 헤더 체크섬이 없다**(TCP/UDP가 보장).
6) **Nagle + Delayed ACK**가 합쳐져 **대화 지연**을 만들 수 있다(작은 요청/응답 패턴).
7) **윈도우 스케일** 없이 고BDP 링크에서 성능 안 나옴.

---

## 11. 체크리스트(설계/튜닝)

- [ ] RTT·대역폭으로 **BDP** 계산 → `rwnd/cwnd` 목표치 가늠
- [ ] **윈도우 스케일/SACK/ECN** 활성(가능 시)
- [ ] **MSS/MTU** 조정(경로 MTU 탐색, 분할 방지)
- [ ] 지연 민감 트래픽: **`TCP_NODELAY`**, **Delayed ACK** 타이머/정책 점검
- [ ] 무선/오손 환경: **FEC/HARQ/L2 ARQ**와 TCP 재전송의 상호작용 확인
- [ ] 장거리/고속: **CUBIC/BBR** 선택, 큐 관리(AQM/CoDel, ECN)

---

## 12. 연습문제(풀이 포함)

### Q1. 50 Mb/s 링크, RTT 80 ms, MSS 1460 B. 파이프를 채우는 최소 세그먼트 윈도우?
**풀이**: BDP = $50\mathrm{e}6 \times 0.08 = 4\mathrm{e}6\,b = 500{,}000\,B$.
세그먼트 수 ≈ 500,000 / 1460 ≈ **343**.

### Q2. Stop-and-Wait에서 $L=5$ kB, $R=5$ Mb/s, RTT=60 ms일 때 이용률?
**풀이**: $T_{tx}= (5k\times8)/5M = 8\,\text{ms}$. $U=8/(8+60)=\mathbf{0.117}$ (11.7%).

### Q3. GBN에서 $a=3$일 때 $W=7$이면 무오류 이용률은?
**풀이**: $1+2a=7$. $W=7\Rightarrow U=1$ (파이프 채움).

### Q4. TCP의 송신 가능량을 제한하는 두 창은? 그리고 실제 송신 허용은?
**풀이**: **rwnd(흐름제어)**, **cwnd(혼잡제어)**. 실제 허용 = **min(rwnd, cwnd)**.

### Q5. RTO 추정에서 재전송된 세그먼트의 RTT 샘플을 제외하는 이유는?
**풀이**: 어떤 ACK가 **원전송**을 가리키는지 불명확 → **RTT 왜곡 방지**(Karn의 알고리즘).

### Q6. SR에서 시퀀스 번호 공간이 8이라면 최대 윈도우는?
**풀이**: **번호공간/2 = 4** (혼동 방지 원칙).

---

## 13. 빠른 암기표(Cheat Sheet)

- **OSI→TCP/IP**: (L7~L5)응용 / L4전송 / L3인터넷 / (L2+L1)네트워크접근
- **PDU**: L4=세그먼트, L3=패킷, L2=프레임
- **SW 이용률** $=1/(1+2a)$, $a=T_p/T_{tx}$
- **GBN/SR**: $W \ge 1+2a$ → 파이프 가득
- **BDP** $=R\cdot RTT$; 윈도우(세그먼트) $\approx \lceil\mathrm{BDP}/\mathrm{MSS}\rceil$
- **TCP 송신 허용** $=\min(rwnd, cwnd)$
- **RTO** $=\text{SRTT}+4\cdot\text{RTTVAR}$ (개념)
- **Nagle**: 소패킷 억제, **Delayed ACK**: ACK 지연(상호작용 주의)
- **SACK/윈도우 스케일/ECN**: 현대 필수 옵션

---

## 14. 마무리

- **링크가 빠른데도 느리다?** 거의 항상 **BDP 대비 윈도우/RTT/손실/큐** 문제입니다.
- **학습 순서 추천**: (1) SW 직감 → (2) GBN/SR로 파이프라인 → (3) BDP로 윈도우 산출 → (4) TCP의 rwnd/cwnd/RTO/SACK/ECN 조립.
