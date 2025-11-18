---
layout: post
title: 데이터 통신 - Multimedia (2)
date: 2024-09-19 21:20:23 +0900
category: DataCommunication
---
# Chapter 28.3 인터넷 멀티미디어

2023년 기준 글로벌 분석 업체 Sandvine·Deloitte 자료를 보면, **비디오 트래픽이 전체 인터넷 트래픽의 약 60~66%**를 차지하고, 1년 사이에만 20% 이상 증가한 것으로 보고된다.  
즉, 오늘날 인터넷은 사실상 “멀티미디어 전송 플랫폼”이라고 해도 과언이 아니다.

---

## 1. 세 가지 스트리밍 패턴 한눈에 보기

### 1.1 세 가지 축

인터넷 멀티미디어를 분류할 때 보통 다음 두 축을 본다.

1. **콘텐츠가 미리 저장되어 있는가?**
2. **사용자와의 상호작용 지연 허용 수준은 어느 정도인가?**

이 두 축으로 분류하면 보통 이렇게 나뉜다.

| 유형 | 시간 특성 | 예시 | 목표 지연(글라스-투-글라스) | 핵심 프로토콜(실무) |
|------|-----------|------|------------------------------|----------------------|
| Streaming stored A/V (VoD) | 미리 저장 | 넷플릭스, 디즈니+, 팟캐스트 | 수십 초도 허용 | HTTP Progressive, HLS, DASH |
| Streaming live A/V | 실시간 생성, 단방향 | 스포츠 생중계, 라이브 행사, 게임 중계 | 수 초(보통 5~20초) | RTMP/SRT 인제스트 + HLS/DASH 배포 |
| Real-time interactive A/V | 실시간 생성, 양방향 | 화상회의, 음성통화, 원격 협업, 클라우드 게임 | 수백 ms 이내(보통 150ms 이하) | RTP/SRTP, WebRTC, DTLS, ICE, STUN/TURN |

여기서 **지연(latency)**은 대략 아래처럼 쪼갤 수 있다.

$$
T_{\text{total}} = T_{\text{capture}} + T_{\text{encode}} + T_{\text{pack}} + T_{\text{network}} + T_{\text{buffer}}
$$

- \(T_{\text{capture}}\): 카메라/마이크에서 입력을 캡처하는 데 걸리는 시간
- \(T_{\text{encode}}\): 인코더가 프레임을 압축하는 데 걸리는 시간
- \(T_{\text{pack}}\): 세그먼트/패킷으로 나누고 헤더를 붙이는 시간
- \(T_{\text{network}}\): 네트워크 전송 시간
- \(T_{\text{buffer}}\): 플레이어/단말이 끊김 방지를 위해 쌓아두는 버퍼 시간

VoD는 \(T_{\text{buffer}}\)를 크게 잡고 끊김을 줄이며, 실시간 상호작용은 \(T_{\text{buffer}}\)를 최소화하고 대신 화질/안정성을 양보한다.

---

## 2. Streaming Stored Audio/Video (저장된 콘텐츠 스트리밍)

### 2.1 개념과 예시

**Streaming stored A/V**는 서버에 **이미 저장된 파일(동영상·오디오)**을 필요할 때마다 클라이언트로 전송해 재생하는 방식이다.

- 넷플릭스에서 드라마 1편을 선택해 보는 경우
- 유튜브에서 이미 업로드된 VOD 영상을 보는 경우
- 팟캐스트 앱에서 지난 방송을 듣는 경우

특징은:

1. **시간 제약이 약하다**  
   - 10·20초 정도의 추가 지연은 사용자가 크게 느끼지 않을 수 있다.
   - 대신 **버퍼를 크게 두고 화질/안정성을 높이는 전략**을 쓴다.

2. **시점은 사용자가 조절**  
   - 사용자는 언제든지 재생 시작·일시정지·앞/뒤로 이동 가능.
   - 서버는 전체 파일 중 일부 구간만 빠르게 제공할 수 있어야 한다.

3. **HTTP를 기본 전송층으로 사용**  
   - 표준 웹 인프라(HTTP, HTTPS, CDN, 프록시 캐시)를 그대로 활용한다.
   - 방화벽/프록시 통과성이 뛰어나다.

---

### 2.2 단순 프로그레시브 다운로드 vs HTTP Adaptive Streaming

저장된 콘텐츠 스트리밍의 구현 방식은 크게 두 가지다.

#### 2.2.1 HTTP 프로그레시브 다운로드

가장 단순한 방식:

1. 브라우저/앱이 `GET /video.mp4`로 요청.
2. 서버는 큰 `.mp4` 파일을 HTTP 응답으로 차례로 내려보낸다.
3. 클라이언트는 **앞부분부터 버퍼링하면서 재생**을 시작한다.

장점:

- 구현이 매우 간단 (웹 서버만 있으면 된다)
- 기존 인프라(캐시, 프록시)를 그대로 활용 가능

단점:

- **적응형 비트레이트(ABR)가 어렵다**  
  큰 파일 하나를 받고 있기 때문에, 도중에 네트워크 상태가 나빠져도 쉽게 화질을 낮출 수 없다.
- **특정 구간부터의 재생(시킹)**에 비효율적  
  HTTP Range 요청으로 해결할 수 있지만 복잡해진다.

#### 2.2.2 HTTP Adaptive Streaming (HLS, DASH 등)

실제 대규모 서비스(넷플릭스, 디즈니+, 유튜브 등)는 거의 다 **HTTP 적응형 스트리밍(HAS: HTTP Adaptive Streaming)**을 사용한다. 대표 예:

- HLS (HTTP Live Streaming, 원래는 Apple이 제안)
- MPEG-DASH (Dynamic Adaptive Streaming over HTTP)  

핵심 아이디어:

1. **미디어를 작은 조각(segment)으로 자르기**  
   - 보통 2~6초짜리 세그먼트들 (`segment_000.ts`, `segment_001.ts`, …).
   - 최신 저지연 모드에서는 1초 이하 또는 부분(chunked) 세그먼트도 사용.

2. **여러 품질(비트레이트)로 인코딩**  
   예:  
   - 240p 500 kbps  
   - 480p 1.2 Mbps  
   - 720p 3 Mbps  
   - 1080p 5 Mbps

3. **매니페스트(playlist) 파일 제공**  
   - HLS: `.m3u8`
   - DASH: `.mpd` (XML 기반)  
   플레이어는 이 리스트를 읽고, **현재 네트워크 상태에 따라 각 세그먼트의 품질을 선택**한다.

4. **클라이언트 쪽 적응 로직(ABR)**  
   - 측정된 대역폭, 버퍼량, 히스토리, QoE 지표 등에 따라 품질을 실시간으로 변경한다.

---

### 2.3 파일 크기와 비트레이트 계산

앞에서 다룬 것처럼, 비트레이트와 재생 시간은 간단히 다음처럼 관계를 가진다.

$$
\text{파일 크기(bit)} = \text{비트레이트(bps)} \times \text{재생 시간(sec)}
$$

예시:

- 1080p 비디오를 H.264로 5 Mbps, 길이 10분(=600초)로 인코딩했다면:

$$
\text{파일 크기} = 5{,}000{,}000 \times 600 = 3{,}000{,}000{,}000 \text{ bit}
$$

이를 바이트로 바꾸면:

$$
\text{파일 크기(Byte)} = \frac{3{,}000{,}000{,}000}{8} \approx 375{,}000{,}000 \text{ Byte} \approx 357 \text{ MiB}
$$

이런 계산을 통해 **네트워크 대역폭, 저장소, CDN 비용**을 설계할 수 있다.

---

### 2.4 버퍼와 지연 – 끊김 없는 재생 조건

스트리밍 클라이언트는 보통 **플레이백 버퍼**를 사용한다.  
버퍼 크기 \(B\)와 스트림 비트레이트 \(R_s\), 버퍼 시간 \(T_b\) 사이에는:

$$
B = R_s \times T_b
$$

예시:

- 비디오 비트레이트: \(R_s = 5\ \text{Mbps}\)
- 초기 버퍼 시간: \(T_b = 5\ \text{s}\)

이면 초기 버퍼량은:

$$
B = 5\,000\,000 \times 5 = 25\,000\,000 \text{ bit} \approx 3.125 \text{ MB}
$$

즉, **3MB 정도만 미리 받아도 5초 분량을 끊김 없이 재생할 수 있다.**

조건:

1. 장기 평균 네트워크 대역폭 \(R_n\)이 \(R_s\)보다 크거나 같아야 한다.  
   - \(R_n < R_s\)이면 버퍼가 점점 줄어들다 결국 끊김(재버퍼링)이 발생.
2. 네트워크 지연/변동(jitter)이 크더라도, 버퍼가 그것을 흡수할 수 있어야 한다.

---

### 2.5 적응형 비트레이트(ABR) 로직 예시

실제 DASH/HLS 플레이어는 **ABR(Adaptive Bitrate) 알고리즘**을 사용한다. 최근 연구와 실무 경험에서는 **지연·버퍼 기반 알고리즘**이 QoE 측면에서 유리하다고 보고된다.  

아주 단순한 예제(파이썬 의사 코드):

```python
# 이용 가능한 프로필 (비트레이트 kbps 기준)
profiles = [
    {"id": "240p", "bitrate": 500},
    {"id": "480p", "bitrate": 1200},
    {"id": "720p", "bitrate": 3000},
    {"id": "1080p", "bitrate": 5000},
]

last_n_throughputs = []  # 최근 N개 세그먼트의 측정 대역폭 (kbps)

def select_profile(throughput_history, buffer_seconds):
    if not throughput_history:
        return profiles[0]  # 첫 세그먼트는 안전하게 최저 화질

    avg_throughput = sum(throughput_history) / len(throughput_history)

    # 버퍼가 충분하면 throughput의 80%까지, 부족하면 50%까지만 사용
    if buffer_seconds > 15:
        safety_factor = 0.8
    elif buffer_seconds > 5:
        safety_factor = 0.6
    else:
        safety_factor = 0.5

    target = avg_throughput * safety_factor

    # target 이하에서 가장 높은 프로필 선택
    chosen = profiles[0]
    for p in profiles:
        if p["bitrate"] <= target and p["bitrate"] >= chosen["bitrate"]:
            chosen = p
    return chosen
```

이 예제는 단지 개념적인 것이고, 실제 상용 플레이어는 **더 복잡한 지표(QoE, 재버퍼링 패널티, 스위칭 빈도 등)**를 사용한다.

---

### 2.6 CDN과 캐싱

스트리밍 VoD에서 **CDN(Content Delivery Network)**은 필수다.

- 전 세계 엣지 서버에 세그먼트 파일을 캐시.
- 사용자가 영상 재생을 시작할 때, 가장 가까운 엣지에서 데이터를 제공해 **지연과 백본 트래픽 비용을 줄임**.
- HTTP 기반이라 **일반 웹 캐시 인프라와 같은 메커니즘**을 사용할 수 있다.

간단한 구조:

```text
[Origin Server] ----[CDN Edge]----[Client]
      ^                   ^             ^
   원본 파일           세그먼트 캐시   플레이어
```

---

## 3. Streaming Live Audio/Video (라이브 스트리밍)

이제 **실시간으로 생성되는 콘텐츠(라이브)**를 다루자.

- 스포츠 라이브, e스포츠, 주식 방송, 라이브 콘서트 스트리밍 등
- 여전히 대체로 **단방향(one-to-many)**이다.
- 지연은 수 초 단위면 괜찮지만, 채팅·도박·배팅과 결합되면 더 낮은 지연이 필요하다.

### 3.1 라이브 스트리밍 파이프라인 개요

전형적인 라이브 스트리밍 파이프라인:

```text
카메라/마이크
    │
    ▼
[인코더] --(RTMP/SRT/…)-> [미디어 서버/클라우드 인제스트]
                                  │
                                  ├─ 트랜스코딩(여러 화질)
                                  └─ 패키징(HLS/DASH, LL-HLS/LL-DASH)
                                  │
                              [CDN/엣지]
                                  │
                                  ▼
                              [클라이언트(플레이어)]
```

#### 3.1.1 인제스트(ingest) 프로토콜

1. **RTMP (Real-Time Messaging Protocol)**  
   - 오랫동안 라이브 인제스트 표준처럼 쓰였던 프로토콜.
   - TCP 기반, 비교적 구현이 쉬움.
   - 하지만 암호화/방화벽 이슈, HTTP/HTTPS 시대와 궁합이 다소 애매해 점점 비중이 줄고 있다.

2. **SRT (Secure Reliable Transport)**  
   - Haivision이 개발 후 오픈소스로 공개한 UDP 기반 저지연 프로토콜.  
   - 특징:
     - 손실 많은 공용 인터넷에서도 ARQ 기반 재전송으로 품질 유지
     - 암호화·인증 지원
     - 튜닝 가능한 지연(몇백 ms~수 초)
   - 요약: “TCP 수준의 안정성과, UDP 수준의 저지연을 동시에 추구”

3. 그 외 RIST, WebRTC 인제스트, DASH-IF Live Media Ingest 등 다양한 신규 표준들이 등장.  

#### 3.1.2 패키징과 배포

인제스트로 들어온 단일(또는 소수) 스트림은 미디어 서버에서:

1. 여러 화질로 **트랜스코딩** (예: 1080p 6 Mbps, 720p 3 Mbps, 480p 1.5 Mbps 등)
2. HLS/LL-HLS, DASH/LL-DASH 형태로 **세그먼트화 + 매니페스트 생성**
3. CDN으로 푸시/풀, 엣지에서 전 세계 사용자에게 배포

---

### 3.2 라이브 스트리밍의 지연 분석

전통적인 HLS/DASH 기반의 라이브 스트리밍 지연은 대략:

- 세그먼트 길이 6초
- 플레이어는 **항상 최소 3개 세그먼트를 버퍼에 쌓아두고 재생** (18초)
- 네트워크 지연 + 인코딩 지연까지 더하면 20~45초 정도 지연이 발생할 수 있다.  

즉, “라이브”라고 해도 실제로는 **수십 초 뒤의 장면**을 보고 있는 셈이다.

#### 3.2.1 저지연 라이브(LL-HLS/LL-DASH, CMAF)

최근에는 CMAF(Common Media Application Format) 기반의 **Low-Latency HLS(LL-HLS)**, **Low-Latency DASH(LL-DASH)**가 표준화·도입되고 있다.  

핵심 아이디어:

1. 긴 세그먼트를 다시 짧은 **chunk**로 나눈다 (예: 1초 세그먼트를 200ms chunk 5개로).
2. HTTP 응답을 **chunked transfer**로 보내, 세그먼트가 완전히 만들어지기 전에 재생을 시작한다.
3. 플레이어는 아주 작은 버퍼만 쌓고 곧바로 재생한다.

이렇게 하면:

- 지연을 2~5초 수준까지 줄이면서
- 여전히 HTTP/CDN 인프라를 사용할 수 있다.

---

### 3.3 라이브 스트리밍 구현 예시 (개념)

여기서는 개념을 설명하는 수준으로, FFmpeg를 사용한 간단한 인제스트 예를 보자.

#### 3.3.1 RTMP 인제스트 예시

```bash
# 로컬 카메라(/dev/video0)와 마이크를 읽어
# H.264/AAC로 인코딩한 뒤 RTMP로 퍼블리시
ffmpeg \
  -f v4l2 -i /dev/video0 \
  -f alsa -i hw:0 \
  -c:v libx264 -preset veryfast -b:v 4000k \
  -c:a aac -b:a 128k \
  -f flv rtmp://live.example.com/app/stream_key
```

서버 측에서는 NGINX-RTMP 모듈 등으로 받아 다시 HLS로 패키징할 수 있다.

#### 3.3.2 SRT 인제스트 예시

```bash
# SRT로 전송
ffmpeg \
  -re -i input.mp4 \
  -c:v libx264 -preset veryfast -b:v 4000k \
  -c:a aac -b:a 128k \
  -f mpegts "srt://ingest.example.com:9000?mode=caller&latency=80"
```

- `latency=80`ms 정도로 설정해 저지연 전송을 시도한다.
- SRT 수신 서버는 이를 다시 HTTP 라이브 스트리밍 포맷으로 변환해 CDN에 배포할 수 있다.

---

### 3.4 저장형 스트리밍과 라이브 스트리밍 비교 정리

| 항목 | Stored (VoD) | Live |
|------|--------------|------|
| 지연 | 수십 초 허용 가능 | 수 초 이내 필요 |
| 타임라인 | 항상 전체가 존재 | 생성 중인 프레임만 존재 |
| 사용자 제어 | 자유로운 시킹, 되감기 | 라이브 기준으로 근처 시킹만 가능 |
| 전송 프로토콜 | HTTP VoD, HLS/DASH | 인제스트(RTMP/SRT) + HLS/DASH(LL) |
| 버퍼 전략 | 큰 버퍼, 끊김 최소화 | 지연과 끊김 사이 타협 |

---

## 4. Real-Time Interactive Audio/Video (실시간 상호작용 A/V)

이제 가장 까다로운 유형인 **실시간 상호작용 스트리밍**이다.

### 4.1 목표와 제약

주요 예시:

- 화상 회의(Zoom, Teams, Webex 등)
- WebRTC 기반 브라우저 화상 채팅
- 온라인 게임 음성 채팅, 클라우드 게임
- 원격 제어/원격 현장 지원

특징:

1. **양방향(two-way)** 통신
2. 인간이 지연을 체감하기 때문에 지연 허용 범위가 매우 작다.
   - 일반적으로 **왕복 지연 250ms 이상**이면 대화가 어색해지고,
   - **왕복 150ms 이하**를 목표로 설계한다.
3. 패킷 손실·지터에 강해야 한다.
4. **TCP는 거의 사용하지 않고, UDP 위에 RTP/SRTP를 사용하는 것이 일반적**이다.

---

### 4.2 WebRTC 아키텍처 개요

오늘날 웹 브라우저 기반 실시간 AV의 사실상의 표준은 **WebRTC**이다. WebRTC는 단일 프로토콜이 아니라, 여러 프로토콜의 조합이다.  

구성 요소:

1. **RTP / SRTP (Real-time Transport Protocol / Secure RTP)**  
   - 음성/영상 프레임을 전송하는 실시간 전송 프로토콜.
   - SRTP는 RTP에 암호화를 추가한 것.

2. **DTLS (Datagram Transport Layer Security)**  
   - UDP 위에서 동작하는 TLS.
   - 키 교환 및 암호 채널 형성에 사용.

3. **ICE (Interactive Connectivity Establishment)**  
   - STUN/TURN 서버를 이용해 NAT/방화벽 환경에서도 피어 간 경로를 찾는 프레임워크.
   - 가능한 모든 후보(candidate) 경로를 시험해 가장 좋은 경로를 선택한다.  

4. **STUN/TURN 서버**
   - STUN: 클라이언트의 공인 IP:포트(“reflexive address”)를 알기 위한 프로토콜.
   - TURN: 직접 연결이 안 될 때, **중계(relay)** 역할을 하는 서버.

5. **시그널링 프로토콜 (별도의 웹소켓/HTTP 등)**
   - WebRTC 자체에는 정의되어 있지 않음.
   - SDP(Session Description Protocol) 교환, ICE candidate 교환, 룸/세션 관리 등은 애플리케이션 개발자가 정의해야 한다.

---

### 4.3 WebRTC 통화 흐름 예시

간단한 1:1 화상 통화 흐름:

1. **시그널링 연결**
   - 두 브라우저 A, B는 WebSocket 등으로 시그널링 서버와 연결.

2. **A가 offer 생성**
   - A: 카메라/마이크 접근 → `RTCPeerConnection` 생성 → `createOffer()` 호출 → SDP offer 생성.
   - SDP에는 코덱, 해상도, 대역폭 제약 등이 들어있다.

3. **시그널링 서버를 통한 offer 전달**
   - A → 서버 → B

4. **B가 answer 생성**
   - B: `setRemoteDescription(offer)` → `createAnswer()` → answer를 A에게 전달.

5. **ICE 후보 교환**
   - A, B는 각각 STUN/TURN 서버에 연결해 가능한 경로 후보(candidate)를 얻는다.
   - 후보를 서로 교환하며 최적 경로를 찾는다 (ICE connectivity checks).

6. **미디어 흐름 시작**
   - 경로가 확정되면, SRTP로 오디오/비디오 프레임이 왕복한다.

이 과정을 통해, **브라우저 간 직접 P2P 또는 서버 중계를 통한 실시간 미디어 흐름**이 만들어진다.

---

### 4.4 WebRTC 코드 스니펫(개념 예제, JavaScript)

아주 간단한 signaling 없는 로컬 loopback 예제 개념:

```javascript
const pc = new RTCPeerConnection();

async function start() {
  const stream = await navigator.mediaDevices.getUserMedia({
    video: true,
    audio: true
  });

  // 로컬 비디오 표시
  document.getElementById("local").srcObject = stream;

  // WebRTC 연결에 트랙 추가
  for (const track of stream.getTracks()) {
    pc.addTrack(track, stream);
  }

  const offer = await pc.createOffer();
  await pc.setLocalDescription(offer);

  // 실제 서비스에서는 여기서 시그널링 서버를 통해
  // offer를 상대에게 보내고, answer를 받아야 한다.
}

pc.ontrack = (event) => {
  // 상대방 비디오/오디오 수신
  document.getElementById("remote").srcObject = event.streams[0];
};

start().catch(console.error);
```

실제 서비스에서는 이 위에 **시그널링, 인증, 룸/채널 관리, SFU/MCU 서버**까지 얹어서 완성한다.

---

### 4.5 SFU vs MCU vs P2P

실시간 멀티 파티 화상 회의에서 중요한 토폴로지:

1. **P2P Mesh**
   - 모든 참가자가 서로 직접 연결.
   - N명이면 연결 수가 \(N(N-1)/2\)라, N이 조금만 커져도 확장성이 나쁘다.

2. **MCU (Multipoint Control Unit)**
   - 중앙 서버가 모든 스트림을 받아서 하나의 믹스된 스트림으로 만들어 각 클라이언트에 전달.
   - 클라이언트는 계산 부담이 적지만, 서버 비용이 크고 지연이 커질 수 있다.

3. **SFU (Selective Forwarding Unit)**
   - 서버가 스트림을 믹스하지 않고, 단지 “선택적 포워딩”만 한다.
   - 각 클라이언트는 여러 스트림을 받아서 레이아웃 렌더링을 담당.
   - 현대 WebRTC 기반 화상 회의의 주류 구조.

---

### 4.6 실시간 스트리밍 vs 라이브 스트리밍 비교

| 항목 | 라이브 스트리밍 | 실시간 상호작용 |
|------|----------------|------------------|
| 지연 목표 | 2~10초 | 150ms 이하(왕복 250ms 이하) |
| 상호작용 | 채팅, 약한 인터랙션 | 음성/영상 대화, 공동 작업 |
| 네트워크 | HTTP 기반(HLS/DASH) | UDP 기반 RTP/SRTP, WebRTC |
| 토폴로지 | 1→N 브로드캐스트 | P2P, SFU, MCU |
| 품질 정책 | 화질 우선, 끊김 조금 허용 | 끊김 최소화, 필요시 화질 희생 |

---

### 4.7 SRT·WebRTC를 결합한 저지연 파이프라인

최근에는 **SRT + WebRTC**를 조합해 다음과 같은 구조를 많이 설계한다.  

1. **SRT**로 스튜디오/카메라에서 클라우드까지 **저지연·고신뢰 전송** (Contribution).
2. 클라우드에서는 이를 여러 화질로 트랜스코딩.
3. **일반 시청자**에게는 HLS/LL-HLS로 배포 (수 초 지연).
4. **패널/해설자·게스트**에게는 WebRTC로 배포 (수백 ms 지연).

이렇게 하면:

- 제작진·출연자 간 인터랙션은 실시간에 가깝게 유지하면서
- 대규모 시청자에게는 안정적인 HTTP 기반 스트리밍을 제공할 수 있다.

---

## 5. 정리

28.3 “Multimedia in the Internet”에서 다룬 세 가지 패턴을 다시 요약하면:

1. **Streaming stored audio/video**  
   - 미리 저장된 콘텐츠를 HTTP 기반 VoD/ABR 방식으로 제공.  
   - 버퍼를 충분히 쌓아 끊김을 줄이고, ABR로 네트워크 상태에 적응.

2. **Streaming live audio/video**  
   - 실시간 생성 콘텐츠를 인제스트(RTMP/SRT 등) → 트랜스코딩 → HLS/ DASH(LL-HLS/LL-DASH)로 배포.  
   - 지연·화질·안정성 사이에서 트레이드오프.

3. **Real-time interactive audio/video**  
   - WebRTC, RTP/SRTP, ICE/STUN/TURN 등의 프로토콜 스택을 이용해 지연 수백 ms 수준의 쌍방향 통신을 구현.  
   - 토폴로지(피어 투 피어, SFU, MCU)에 따라 확장성과 서버 비용이 달라짐.