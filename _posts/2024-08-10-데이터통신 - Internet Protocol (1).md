---
layout: post
title: 데이터 통신 - Internet Protocol (1)
date: 2024-08-10 20:20:23 +0900
category: DataCommunication
---
# Chapter 19.1 Internet Protocol — IPv4 Datagram Format, Fragmentation, Options, Security

> 앞 장(18장)에서 **IPv4 주소, 포워딩, 라우터 동작**을 봤다면, 이제는 “실제로 전선 위를 흐르는 IPv4 데이터그램이 정확히 어떻게 생겼는지”와 “어디서 잘리고(fragmentation), 어떤 옵션이 있으며, 보안 측면에서 어떤 문제가 있는지”를 파는 단계다.
> 이 장은 RFC 791(Internet Protocol)과 이후 확장(RFC 2474, RFC 3168, RFC 6274 등)을 기본 레퍼런스로 삼는다.

---

## IPv4 Datagram Format

### 전체 IPv4 헤더 구조

IPv4 데이터그램은 크게 **헤더(Header)**와 **데이터(payload)**로 나뉜다.

아주 기본적인(옵션 없는) IPv4 헤더는 **20바이트** 길이를 갖고, 필요하면 옵션 필드를 통해 60바이트까지 늘릴 수 있다. RFC 791에서 정의한 포맷은 아래와 같다.

```text
  0                   1                   2                   3
  0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |Version| IHL   | DSCP | ECN |         Total Length            |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |         Identification        |Flags|      Fragment Offset   |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |  Time to Live |    Protocol   |        Header Checksum       |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |                       Source Address                          |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |                    Destination Address                        |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |                    Options (if any)           |    Padding    |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |                          Data ...                             |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

각 필드의 의미를 정리하면 다음과 같다.

| 필드                 | 크기   | 설명 |
|----------------------|--------|------|
| Version              | 4 bit  | IP 버전 (IPv4는 항상 4) |
| IHL                  | 4 bit  | Internet Header Length (4바이트 단위) |
| DSCP / ECN           | 8 bit  | Differentiated Services Code Point / Explicit Congestion Notification |
| Total Length         | 16 bit | 헤더+데이터 합한 전체 길이 (바이트) |
| Identification       | 16 bit | 단편화(fragmentation) 시 각 조각을 묶어주는 ID |
| Flags                | 3 bit  | 단편화 제어(DF, MF) 등 |
| Fragment Offset      | 13 bit | 원본 데이터그램 내에서의 조각 위치(8바이트 단위) |
| Time to Live (TTL)   | 8 bit  | 홉 수 제한(0이 되면 폐기) |
| Protocol             | 8 bit  | 상위 계층 프로토콜 번호 (TCP=6, UDP=17 등) |
| Header Checksum      | 16 bit | IP 헤더에 대한 오류 검사(비암호학적) |
| Source Address       | 32 bit | 출발지 IPv4 주소 |
| Destination Address  | 32 bit | 목적지 IPv4 주소 |
| Options + Padding    | 0~40 B | 선택적 옵션들, 32bit 정렬을 위한 패딩 |

---

### Version과 IHL

- **Version**
  - 4bit, 현재 IPv4 데이터그램이라면 항상 `4`.
  - IPv6는 별도의 헤더 구조를 가지며, Version=6.

- **IHL (Internet Header Length)**
  - “헤더의 길이(4바이트 단위)”를 나타냄.
  - 최소값: 5 (5 × 4 = 20바이트, 옵션 없음)
  - 최대값: 15 (15 × 4 = 60바이트)

수식으로 쓰면:

$$
\text{HeaderLen(bytes)} = \text{IHL} \times 4
$$

예시:

- IHL = 5 → 헤더 20바이트, 옵션 없음
- IHL = 7 → 헤더 28바이트, 옵션 8바이트 포함

---

### DS 필드(DSCP + ECN)

과거 IPv4 헤더의 8비트는 **Type of Service(ToS)** 로 정의되었지만, 현대 인터넷에서는 **Differentiated Services(DS) 필드**로 재정의되었다.

- 상위 6비트: **DSCP(Differentiated Services Code Point)**
  - 트래픽 클래스(예: EF, AFxy, CSx)를 표현
  - IANA의 DSCP registry에 공식 코드포인트가 정의되어 있음.
- 하위 2비트: **ECN(Explicit Congestion Notification)**
  - 혼잡 발생 시 패킷을 드롭하는 대신 “마킹”하는 용도로 사용
  - `00`: Not-ECT, `10`/`01`: ECT(0/1), `11`: CE(혼잡 경험)

실무 예:

- VoIP나 실시간 게임 트래픽: EF(Expedited Forwarding) DSCP를 설정해 우선 처리
- 백업/대용량 파일 전송: 낮은 우선순위 DSCP로 설정해 혼잡 시 먼저 드롭

---

### Total Length

- 헤더 + 데이터 전체 길이(바이트)
- 16비트 → 최대 값은 \(2^{16}-1 = 65\,535\) 바이트

$$
20 \le \text{Total Length} \le 65,\!535
$$

이 필드 덕분에 수신 측은 **IP 계층에서 데이터그램의 끝을 정확히 알고**,
그 뒤에 오는 다른 프레임/패킷과 구분할 수 있다.

예: MTU 1500 이더넷 환경에서

- 총 길이 1500 이하 → 단편화 없이 전송 가능
- 더 크면 단편화 필요(DF 비트가 0인 경우)

---

### Identification, Flags, Fragment Offset

이 세 필드는 **단편화(fragmentation)/재조립(reassembly)**를 위해 함께 사용된다.

- **Identification(16비트)**
  - 같은 원본 데이터그램에서 나온 조각들이 **같은 값**을 가짐.
  - 수신 측은 `(Source IP, Destination IP, Protocol, Identification)` 조합으로 어떤 조각이 어떤 원본 데이터그램에 속하는지 구분한다.

- **Flags(3비트)** — RFC 791에서 정의된 의미

  - 비트 0: 예약(항상 0이어야 함)
  - 비트 1: DF(Don’t Fragment)
    - 0: 단편화 허용
    - 1: 단편화 금지 (Path MTU Discovery에 사용)
  - 비트 2: MF(More Fragments)
    - 0: 마지막 조각
    - 1: 뒤에 더 많은 조각이 있음

- **Fragment Offset(13비트)**
  - 조각의 위치를 **8바이트 단위**로 나타낸다.
  - 즉, 실제 바이트 offset은 `FragmentOffset × 8`.

수식:

$$
\text{ByteOffset} = \text{FragmentOffset} \times 8
$$

---

### TTL, Protocol, Header Checksum

- **TTL(Time to Live)**
  - 패킷이 라우터를 지날 때마다 1씩 감소.
  - 0이 되면 해당 패킷은 폐기되고, 보통 ICMP Time Exceeded 메시지가 출발지로 간다.
  - 라우팅 루프 방지 기능.

- **Protocol**
  - 상위 계층(transport or others)을 식별하는 번호.
  - 예: TCP=6, UDP=17, ICMP=1, OSPF=89 등
  - IANA의 Protocol Numbers registry에 공식 번호가 정의되어 있다.

- **Header Checksum**
  - IPv4 헤더에 대한 16비트 1의 보수(ones-complement) 체크섬.
  - 라우터가 TTL을 줄일 때마다 헤더가 바뀌므로, 그때마다 checksum을 다시 계산하거나 “보정”해야 한다.
  - 페이로드는 보호하지 않아, **보안용 보호 수단은 전혀 아니다** (단순 오류 검출용).

---

### 소스/목적지 주소, Options

- **Source Address, Destination Address**
  - 32비트 IPv4 주소.
  - 앞 장에서 본 CIDR, DHCP, ARP를 통해 할당·해석되는 값.

- **Options + Padding**
  - 다양한 목적(경로 기록, 시간 측정, source route, router alert 등)으로 사용.
  - 실무 인터넷에서는 **옵션을 포함한 패킷이 성능/보안 이슈 때문에 거의 사용되지 않으며**, 일부 옵션은 사실상 사용 금지 수준으로 취급된다.
  - 뒤의 3장에서 Options를 따로 상세히 다룬다.

---

### 간단한 헤더 파서 예제 (파이썬)

리눅스에서 RAW 소켓으로 읽어온 데이터가 IPv4 패킷이라고 가정하고,
헤더의 기본 필드들을 파싱해 보는 코드:

```python
import struct
import socket

def parse_ipv4_header(data: bytes):
    # 최소 20바이트 필요
    if len(data) < 20:
        raise ValueError("too short for IPv4 header")

    # !: network byte order, B: 1바이트, H: 2바이트, 4s: 4바이트 문자열
    first_byte, dscp_ecn, total_len, ident, flags_frag, ttl, proto, chksum, \
        src, dst = struct.unpack("!BBHHHBBH4s4s", data[:20])

    version = first_byte >> 4
    ihl = first_byte & 0x0F
    header_len = ihl * 4

    flags = (flags_frag >> 13) & 0x7
    frag_offset = flags_frag & 0x1FFF

    src_ip = socket.inet_ntoa(src)
    dst_ip = socket.inet_ntoa(dst)

    return {
        "version": version,
        "ihl": ihl,
        "header_len": header_len,
        "dscp": dscp_ecn >> 2,
        "ecn": dscp_ecn & 0x3,
        "total_len": total_len,
        "identification": ident,
        "flags": flags,
        "df": (flags & 0x2) != 0,
        "mf": (flags & 0x1) != 0,
        "fragment_offset": frag_offset,
        "ttl": ttl,
        "protocol": proto,
        "header_checksum": chksum,
        "src_ip": src_ip,
        "dst_ip": dst_ip,
    }
```

이 코드는 **헤더 구조 이해**를 돕는 예제일 뿐, 실제로 패킷을 조작하거나 공격에 사용하는 용도는 아니다.

---

## Fragmentation (단편화)

### 왜 단편화가 필요한가

네트워크에는 각 링크마다 **MTU(Maximum Transmission Unit)**가 있다.

- 이더넷(일반): 1500바이트
- PPPoE: 1492바이트
- 일부 VPN 터널: 1400바이트 근처
- 점대점 전용 회선 등: 다양

IPv4 데이터그램의 `Total Length`가 **경로 상 링크의 MTU보다 크면**,
해당 링크에선 그대로 보낼 수 없다. 이때 가능한 전략은 두 가지:

1. **라우터가 데이터그램을 잘게 쪼갬(fragmentation)**
   - DF=0인 경우(단편화 허용)
2. **단편화하지 않고, ICMP “Fragmentation Needed”를 돌려보내 출발지가 Path MTU를 줄이도록 함**
   - DF=1인 경우 (Path MTU Discovery에 사용)

IPv6는 라우터 단편화를 금지하고, **오직 출발지에서만 단편화**할 수 있게 바뀌었다.
이는 IPv4 단편화로 인한 성능·보안 문제를 줄이기 위한 설계다.

---

### 단편화 동작 방식

단편화 발생 시, 라우터는:

1. 원본 데이터그램의 **Identification**을 그대로 복사.
2. 각 조각마다 **Flags/FragmentOffset**을 적절히 설정.
3. 각 fragment마다 **새로운 IP 헤더**를 붙이고, Total Length를 해당 fragment 크기에 맞게 조정.

재조립은 **목적지 호스트에서만** 수행되며, 라우터는 중간에서 재조립하지 않는다.

---

### 예제: 4000바이트 데이터그램을 MTU 1500 링크로 전송

가정:

- 원본 IPv4 데이터그램 총 길이: 4000바이트 (헤더 20 + 데이터 3980)
- 링크 MTU: 1500
- DF=0 (단편화 허용)
- 옵션 없음 → 각 fragment 헤더 20바이트

1) **한 fragment에 실릴 수 있는 최대 데이터(payload)**

$$
\text{MaxData} = \text{MTU} - \text{HeaderLen}
= 1500 - 20 = 1480\ \text{bytes}
$$

IPv4 규칙상 **마지막 fragment를 제외한 나머지 fragment의 데이터 길이는 8바이트의 배수**여야 한다.

- 1480은 \(1480 / 8 = 185\)로 정확히 나누어 떨어짐 → OK

2) **필요한 fragment 수**

원본 데이터(payload)는 3980바이트.

- 첫 fragment: 1480바이트
- 두 번째 fragment: 1480바이트 → 지금까지 2960바이트
- 남은 데이터: \(3980 - 2960 = 1020\)바이트 → 세 번째 fragment

총 세 개의 fragment가 만들어진다.

3) **각 fragment의 Offset 계산**

FragmentOffset은 “데이터 부분의 시작 위치/8”이다.

- Fragment 1
  - 데이터 범위: 0 ~ 1479
  - Offset: \(0 / 8 = 0\)
  - Flags: MF=1 (뒤에 더 있음)
  - Total Length: 20(헤더) + 1480 = 1500

- Fragment 2
  - 데이터 범위: 1480 ~ 2959
  - Offset: \(1480 / 8 = 185\)
  - Flags: MF=1
  - Total Length: 1500

- Fragment 3
  - 데이터 범위: 2960 ~ 3979
  - Offset: \(2960 / 8 = 370\)
  - Flags: MF=0 (마지막 조각)
  - Total Length: 20 + 1020 = 1040

수식으로 일반화하면:

$$
\text{Offset}_i = \frac{\text{데이터 시작 바이트}}{8}
$$

---

### 단편화 시뮬레이션 코드 예제 (파이썬)

```python
def fragment_ipv4(total_len, header_len, mtu):
    data_len = total_len - header_len
    max_data = mtu - header_len

    # 마지막 fragment 제외 데이터 길이는 8의 배수
    max_data_aligned = (max_data // 8) * 8

    fragments = []
    offset = 0
    remaining = data_len

    while remaining > 0:
        if remaining > max_data_aligned:
            dlen = max_data_aligned
            mf = True
        else:
            dlen = remaining
            mf = False

        frag = {
            "offset": offset // 8,
            "data_len": dlen,
            "total_len": dlen + header_len,
            "mf": mf,
        }
        fragments.append(frag)

        offset += dlen
        remaining -= dlen

    return fragments

frags = fragment_ipv4(total_len=4000, header_len=20, mtu=1500)
for i, f in enumerate(frags, 1):
    print(f"Fragment {i}: offset={f['offset']}*8, data={f['data_len']} bytes, "
          f"total_len={f['total_len']}, MF={int(f['mf'])}")
```

출력은 앞에서 손으로 계산한 내용과 같은 구성을 보여준다.

---

### Fragmentation이 가져오는 문제점

1. **성능 문제**
   - 각 fragment마다 헤더가 붙어 오버헤드 증가.
   - 손실 시 재전송 비용 증가: fragment 중 하나만 유실되어도, 상위 계층 입장에선 “원본 세그먼트 전체를 다시 보내야” 할 수 있음.
   - 라우터/호스트의 reassembly 버퍼 부담.

2. **보안 문제 (상세는 4장에서)**
   - **중첩(overlapping) fragment**를 이용해 침입 탐지/방화벽을 우회하는 공격.
   - **tiny fragment**로 헤더를 여러 조각으로 쪼개, 필터링을 우회하려는 시도.
   - RFC 6274는 이러한 공격 벡터와 방어 권고사항을 자세히 정리하고 있다.

3. **Path MTU Discovery와의 상호작용**
   - PMTUD는 DF=1로 보내고, 단편화가 필요하면 라우터가 ICMP “Fragmentation Needed”를 보내도록 하는 메커니즘.
   - 방화벽/중간 장비가 ICMP를 막아버리면 PMTUD가 깨지고, “블랙홀” 문제가 발생.

실무에서는 **가급적 IPv4 단편화를 피하고**,
MSS 조정이나 터널 MTU 조정으로 Path MTU에 맞춰 송신하는 것이 권장된다.

---

## IPv4 Options

### 옵션 필드의 구조

옵션은 “필요한 패킷만” 추가 정보를 실을 수 있도록 만든 확장 메커니즘이다.
하지만 많은 라우터가 **옵션이 있는 패킷을 slow path로 보내거나 아예 드롭**하기 때문에,
실제 인터넷 트래픽에서는 거의 사용되지 않는다.

옵션의 기본 형식은 다음과 같다 (RFC 791):

- **단일 바이트 옵션** (예: EOL, NOP)
  - 8bit: Option Type
- **가변 길이 옵션**
  - Option Type (1바이트)
  - Option Length (1바이트, 전체 길이)
  - Option Data (가변 길이)

옵션 리스트 끝은 항상 **붙임을 맞추기 위한 Padding**을 포함해
헤더 전체가 32bit(4바이트) 배수가 되도록 맞춘다.

---

### 주요 옵션 타입

아래 표는 대표적인 IPv4 옵션들이다. (번호는 RFC에서 정의된 Type 값의 십진수 표현)

| 이름                 | 타입 | 설명 |
|----------------------|------|------|
| End of Option List   | 0    | 옵션 리스트 끝을 표시 |
| No Operation (NOP)   | 1    | 1바이트 패딩, 옵션 정렬에 사용 |
| Record Route         | 7    | 경로 상의 라우터 주소를 차례로 기록 |
| Timestamp            | 68   | 경로 상 라우터가 타임스템프를 기록 |
| Loose Source Route   | 131  | “느슨한” source routing (지나야 할 라우터 리스트) |
| Strict Source Route  | 137  | “엄격한” source routing (정확한 다음 홉 목록) |
| Router Alert         | 148  | 라우터가 이 패킷을 특별히 주목해야 함 (예: IGMP, RSVP) |

특히, **Loose/Strict Source Route**는 보안상 위험성이 커서
대부분의 운영자들이 **완전히 차단**하거나 사용하지 않는다.

---

### Record Route & Timestamp 예

#### Record Route

- 송신자가 “내 패킷이 어떤 라우터들을 거쳐 가는지”를 알고 싶을 때 사용.
- 각 라우터는 자신이 패킷을 포워딩할 때 **자신의 IP 주소를 옵션에 추가**.

제한:

- 옵션 공간이 작기 때문에, 실제로는 **몇 홉 안 되는 짧은 경로**에서만 유의미.
- 경로 상 라우터들이 “옵션 처리”를 모두 지원해야 함.
- 보안·성능 문제로, 많은 ISP 코어 라우터가 이 옵션을 무시하거나 드롭한다.

#### Timestamp

- Record Route와 비슷하지만 “주소 대신 (또는 함께) 타임스템프”를 기록.
- RTT 측정이나, “각 홉에서 처리 시간”을 분석하는 데 사용할 수 있음.
- 마찬가지로 실무 인터넷에서는 거의 쓰이지 않음.

---

### Router Alert 옵션

**Router Alert(옵션 타입 148)**은 “이 패킷은 라우터에서 특별히 처리해야 함”을 표시하는 옵션이다. 예를 들어 IGMP, RSVP 메시지가 이런 옵션을 사용할 수 있다.

- 라우터는 이 옵션이 있으면 패킷을 그냥 포워딩만 하는 것이 아니라
  **제어 플레인으로도 전달**해 추가 처리를 할 수 있다.
- 하지만 옵션 자체가 slow path를 유발하므로,
  대부분의 트래픽은 이 옵션 없이 운영되는 것이 일반적이다.

---

### 옵션 파싱 예제 코드

단순화된 IPv4 옵션 파서(모든 옵션을 다 지원하는 건 아니고, 구조 이해용):

```python
def parse_ipv4_options(header: bytes):
    """header: IPv4 헤더 전체 (옵션 포함), 최소 20바이트"""
    ihl = (header[0] & 0x0F)
    header_len = ihl * 4

    if header_len == 20:
        return []  # 옵션 없음

    options_data = header[20:header_len]
    opts = []
    i = 0
    while i < len(options_data):
        opt_type = options_data[i]
        if opt_type == 0:   # End of Option List
            opts.append({"type": 0, "name": "EOL"})
            break
        elif opt_type == 1: # NOP
            opts.append({"type": 1, "name": "NOP"})
            i += 1
            continue
        else:
            if i + 1 >= len(options_data):
                break
            opt_len = options_data[i+1]
            opt_data = options_data[i+2:i+opt_len]
            opts.append({
                "type": opt_type,
                "length": opt_len,
                "data": opt_data,
            })
            i += opt_len
    return opts
```

이 코드는 옵션이 실제로 무엇을 의미하는지는 해석하지 않지만,
“옵션 헤더가 어떻게 구성되어 있는지”를 보여주는 데 충분하다.

---

## Security of IPv4 Datagram

IPv4는 1981년에 설계되었고, 당시에는 **보안 위협 모델이 매우 단순**했다.
RFC 6274는 “이 오래된 설계가 현대 인터넷에서 어떤 보안 문제를 갖는지”를 종합적으로 분석하고 있다.

여기서는 **데이터그램 자체와 관련된 대표적인 보안 이슈**를 정리한다.

---

### 헤더 체크섬의 한계

- IPv4 **Header Checksum**은 단순히 전송 중 우발적인 비트 오류를 잡기 위한 용도다.
- 공격자는 패킷을 변조한 다음, **체크섬을 새로 계산해 넣으면 끝**이기 때문에
  **무결성·인증 보장은 전혀 제공하지 않는다.**
- 페이로드(데이터)에는 아예 checksum도 걸려 있지 않다(상위 계층이 책임).

따라서:

- IP 레벨에서의 “보안”이 필요하다면, **IPsec(AH/ESP)** 등의 별도 메커니즘이 필요하다.
- 현대의 실제 인터넷 보안은 주로 **TLS(전송 계층)** 또는 **VPN(터널링)**에서 이뤄진다.

---

### 주소 스푸핑(Source Address Spoofing)과 BCP 38

IPv4 헤더의 Source Address는 **송신자가 임의의 값을 넣을 수 있다**.
라우터는 기본적으로 이 값을 검증하지 않기 때문에,

- 공격자가 출발지 주소를 “임의의 다른 사람 주소”로 바꿔 넣는 **스푸핑(spoofing)**이 가능하다.
- 그 결과:
  - **DDoS 공격**에서 공격 원점을 숨길 수 있음.
  - 반사(reflection)·증폭(amplification) 공격에서 출발지 IP를 피해자 주소로 바꾸어,
    응답 트래픽이 피해자에게 폭주하도록 만들 수 있음.

이를 막기 위해 IETF는 **BCP 38 (Network Ingress Filtering)**을 권고한다.

- ISP/조직 경계 라우터는 “자기 네트워크 쪽에서 나오는 패킷의 Source IP가
  **자기 네트워크에 할당된 주소인지** 검사”해야 한다.
- 맞지 않으면 해당 패킷은 드롭 → 스푸핑된 패킷이 인터넷으로 나가지 못하게 함.

실무에서는 여전히 BCP 38을 적용하지 않은 네트워크가 남아 있어,
스푸핑 기반 DDoS가 완전히 사라지지는 않았다. 하지만 **주요 북미·유럽 ISP**들은
이 원칙을 점점 더 적극적으로 적용하고 있다.

---

### Fragmentation 관련 공격

RFC 6274는 IPv4 단편이 **어떻게 악용될 수 있는지**에 대해 자세히 기술한다.

대표적인 공격 아이디어:

1. **Overlapping Fragments**
   - 일부 fragment들이 겹치는 offset 범위를 가지도록 조작.
   - 호스트/방화벽/IDS마다 reassembly 규칙이 다를 경우,
     - 방화벽/IDS는 무해한 것으로 재조립하지만,
     - 실제 호스트는 악의적인 페이로드로 재조립…
   - 결과적으로 필터를 우회하는 효과.

2. **Tiny Fragment Attack**
   - IP 헤더나 TCP 헤더를 **여러 작은 fragment로 쪼갠다.**
   - 방화벽이 “첫 fragment 헤더만 보고 필터링”하는 경우,
     나머지 fragment에 숨겨진 정보를 보고 차단하지 못할 수 있다.
   - 예: TCP 포트 번호를 두 번째 fragment에 넣어 우회.

3. **Fragment Flooding**
   - reassembly 버퍼를 고갈시키기 위해 대량의 fragment를 뿌려 DoS를 유발.

방어 권고(요약):

- overlapping fragment가 들어오면 **재조립 과정에서 오류로 간주하고 폐기**.
- fragment 크기가 너무 작거나, 비정상적인 패턴이면 필터링.
- 방화벽/IDS는 reassembly 로직을 엄격히 RFC 기준에 맞추거나,
  아예 “이상한 fragment는 모두 드롭”하는 정책을 선택.

---

### Options 기반 공격과 정책

IPv4 옵션은 그 자체로 **slow path**를 유발하고,
일부 옵션은 보안상 위험하다.

1. **Source Routing 옵션 (Loose/Strict Source Route)**
   - 송신자가 경로를 직접 지정할 수 있어,
     **보안 장비를 우회하거나 특정 경로를 강제**하는 데 악용될 수 있다.
   - RFC 6274는 이 옵션을 **기본적으로 비활성화하거나 드롭**할 것을 강력 권고한다.

2. **Record Route/Timestamp**
   - 자체가 공격은 아니지만, **네트워크 토폴로지·라우터 주소·지연 정보**를 노출해
     공격자가 정찰(reconnaissance)에 사용할 수 있음.
   - 많은 ISP가 이 옵션들을 무시하거나 필터링한다.

3. **Router Alert**
   - 제어 트래픽(IGMP, RSVP 등)을 위해 필요한 경우가 있으나,
     임의의 패킷이 Router Alert를 사용해 라우터 CPU에 부담을 줄 수 있다.
   - 따라서 특정 프로토콜에 대해서만 허용하고, 나머지는 방화벽으로 차단하는 식의 정책이 사용된다.

현대 운영 가이드라인에서는 대체로:

- **일반 인터넷에서는 IPv4 옵션 사용을 최소화**하고,
- **Source Route 옵션은 반드시 차단**,
- **Record Route/Timestamp는 내부 디버깅용 정도로만 제한**
하는 것을 권장한다.

---

### ICMP, Path MTU, 라우팅 공격

IPv4 데이터그램은 ICMP와 함께 동작한다.

- DF=1 + MTU 초과 시: 라우터가 ICMP “Fragmentation Needed”를 보냄 (PMTUD).
- 라우팅 문제: 라우터는 ICMP Redirect/Unreachable/Time Exceeded 등을 사용.

보안 측면:

- 공격자가 **위조된 ICMP 메시지**를 보내 호스트의 라우팅 캐시를 왜곡하거나,
  Path MTU를 잘못 추정하게 만들어 **MTU blackhole**을 유발할 수 있다.
- 방화벽/라우터는 ICMP를 완전히 막기보다는,
  - “내부에서 외부로 나가는 요청에 대한 정상적인 응답(ICMP error)”은 허용하고,
  - 외부에서 들어오는 의심스러운 ICMP는 필터링하는 형태의 **정교한 정책**을 사용한다.

---

### IPv4 보안 강화를 위한 일반적인 조치

IPv4 데이터그램 자체가 제공하는 보안 기능은 매우 제한적이므로,
실무에서는 다음과 같은 레이어에서 보안을 보강한다.

1. **네트워크 경계에서의 필터링**
   - BCP 38 ingress filtering으로 스푸핑 방지.
   - 비정상적인 fragment, 위험한 옵션, 이상한 TTL 등을 필터링.

2. **IPsec**
   - AH/ESP를 사용해 IP 레벨에서 **무결성(Integrity)·기밀성(Confidentiality)** 제공.
   - 특히 사이트 간 VPN, 기업 내부 트래픽 보호에 사용.

3. **전송 계층/애플리케이션 계층 보안**
   - TLS(HTTPS, SMTPS 등), SSH, QUIC 등의 암호화 프로토콜 사용.
   - 애플리케이션 레벨에서 인증/인가 로직을 구현.

4. **라우팅 보안**
   - BGPsec/RPKI 등으로 라우팅 정보 자체의 위변조를 방지.

IPv4 자체는 “암호화나 인증을 제공하지 않는 단순한 패킷 전달 메커니즘”이라는 점을 명확히 이해하고,
위 레이어에서 적절한 보안 대책을 결합해야 한다.

---

## 정리

이번 19.1에서는 **IPv4 데이터그램의 내부 구조**와 그 위에서 벌어지는 **단편화·옵션 처리·보안 이슈**를 집중적으로 살펴보았다.

- **Datagram format**
  - IPv4 헤더는 최소 20바이트, IHL과 Options에 따라 최대 60바이트까지 확장.
  - DSCP/ECN 필드를 통해 DiffServ, Explicit Congestion Notification을 지원.
  - Identification/Flags/FragmentOffset는 단편화에 사용.
  - Header Checksum은 단순 오류 검출용이지, 보안 기능은 아니다.

- **Fragmentation**
  - 링크 MTU를 넘는 데이터그램은 조각으로 나눠져 전송.
  - FragmentOffset은 8바이트 단위, DF/MF 비트로 단편화 제어.
  - 성능과 보안 관점에서 문제가 많아, 실무에서는 Path MTU Discovery와 MSS 조정으로 단편화를 피하는 것이 일반적.

- **Options**
  - EOL, NOP, Record Route, Timestamp, Source Route, Router Alert 등 다양한 옵션 존재.
  - 하지만 보안·성능상 이유로, 공용 인터넷에서는 거의 사용되지 않고 대부분 필터링 또는 slow path 처리.

- **Security of IPv4 datagram**
  - IPv4 헤더는 스푸핑, 단편화 공격, 옵션 악용, ICMP 악용 등의 공격 표면을 가진다.
  - BCP 38 ingress filtering, fragment/옵션 필터링, IPsec, 상위 계층 보안 프로토콜을 통해 보완해야 한다.
  - RFC 6274는 IPv4 설계/구현의 보안 취약점을 체계적으로 정리한 참고 문서다.
