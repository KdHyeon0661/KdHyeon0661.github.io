---
layout: post
title: 데이터 통신 - Internet Protocol (2)
date: 2024-08-10 21:20:23 +0900
category: DataCommunication
---
# Chapter 19.2 ICMPv4 — Messages, Debugging Tools, Checksum

> 목표: IPv4 위에서 동작하는 **ICMPv4(Internet Control Message Protocol)**의
> - 메시지 구조와 종류,
> - `ping`, `traceroute` 같은 디버깅 도구에서의 활용,
> - ICMP 체크섬 계산 방법
> 을 **이론 + 예제 + 코드**까지 한 번에 정리한다.
> ICMPv4는 RFC 792가 기본이고, 타입·코드는 IANA 레지스트리에서 최신으로 관리된다.

---

## ICMPv4 Messages

### ICMPv4의 위치와 역할

- **계층 위치**
  - IP 헤더의 **Protocol 필드가 1**인 경우, 그 페이로드가 ICMPv4 메시지다.
  - 즉, ICMP는 별도 L4가 아니라 **“IPv4의 헬퍼 프로토콜”**(제어/진단용)로 동작한다.
- **주요 목적**
  1. **에러 보고**: 목적지 도달 불가, TTL 초과, 파라미터 에러, 리다이렉트 등
  2. **상태·진단**: Echo(Echo Request/Reply), Timestamp, Router Advertisement 등
- **규칙(요약)** (RFC 792, RFC 1122 기준)
  - 에러를 알리기 위해 **ICMP 에러를 보내지만**, **에러에 대한 ICMP 에러는 보내지 않는다.**
  - 브로드캐스트/멀티캐스트에 대해선 대부분의 에러 ICMP를 보내지 않는다.
  - 에러 메시지 안에는 항상 **원본 IP 헤더 + L4 헤더 일부(최소 8바이트)**를 실어, 송신 측이 어느 세션에 대한 에러인지 식별할 수 있게 한다.

---

### 공통 ICMP 헤더 포맷

RFC 792에서 정의된 ICMPv4 메시지의 기본 구조:

```text
0                   1                   2                   3
0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|     Type      |     Code      |          Checksum             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                 Rest of Header (type/code별로 다름)          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         Data ...                             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

- **Type (8비트)**: 어떤 종류의 ICMP 메시지인지
- **Code (8비트)**: 같은 타입 안에서의 세부 코드
- **Checksum (16비트)**: ICMP 전체(헤더+데이터)에 대한 Internet checksum
- **나머지 필드 + 데이터**는 타입/코드마다 구조가 다르다.

예) **Echo Request/Reply** 구조:

```text
Type | Code | Checksum | Identifier | Sequence Number | Data...
```

---

### 에러 메시지 vs 정보 메시지

IANA의 ICMP Type Numbers를 기준으로, 자주 쓰이는 타입만 추려보면:

| 구분 | Type | 대표 Code | 이름 / 용도 (요약) |
|------|------|-----------|---------------------|
| 정보 | 0    | 0         | Echo Reply (`ping` 응답) |
| 에러 | 3    | 여러 개   | Destination Unreachable (네트워크/호스트/포트 등 도달 불가) |
| 에러 | 4    | 0         | Source Quench (혼잡 제어) — **현재는 완전히 Deprecated** |
| 에러 | 5    | 0~3       | Redirect (더 나은 라우터로 경로 갱신) |
| 정보 | 8    | 0         | Echo Request (`ping` 요청) |
| 정보 | 9    | 0         | Router Advertisement (라우터 자신 홍보) |
| 정보 | 10   | 0         | Router Solicitation (라우터 찾기 요청) |
| 에러 | 11   | 0/1       | Time Exceeded (TTL 초과 / 재조립 시간 초과) |
| 에러 | 12   | 0~2       | Parameter Problem (헤더 필드 에러) |
| 정보 | 13   | 0         | Timestamp Request |
| 정보 | 14   | 0         | Timestamp Reply |

IANA는 이 타입·코드를 **공식 레지스트리**에서 지속적으로 갱신하며, 2025년 기준으로 Source Quench(Type 4), Alternate Host Address(Type 6) 등 일부는 “Deprecated”로 명시되어 있다.

---

### 대표 에러 메시지와 시나리오

#### 1) Destination Unreachable (Type 3)

**상황:**

- 라우터가 목적지 네트워크로 가는 라우트가 없거나,
- 목적지 호스트가 다운되어 있거나,
- L4 포트가 열려 있지 않거나,
- DF=1인데 MTU가 부족해 단편화가 필요할 때 등.

**구조(요약):**

```text
Type=3 | Code | Checksum
Unused (0)
+ 원본 IP 헤더 + 원본 데이터의 앞 8바이트
```

**중요한 Code 예시 (일부만):**

- Code 0: Network unreachable
- Code 1: Host unreachable
- Code 2: Protocol unreachable
- Code 3: Port unreachable
- Code 4: Fragmentation needed and DF set (Path MTU Discovery에 사용)

**시나리오 예 — Port Unreachable**

1. 클라이언트 A가 서버 B의 UDP 포트 9999로 패킷을 보냈는데,
2. 서버 B에서 해당 포트를 리슨하는 프로세스가 없음.
3. B는 해당 패킷을 드롭하고, A에게 **ICMP Type 3, Code 3 (Port unreachable)**를 보낸다.
4. A의 애플리케이션/OS는 이 에러를 보고 “연결 불가능” 상태로 처리.

`traceroute`(UNIX 계열)는 마지막 홉에 도달했는지 확인하기 위해 바로 이 **Port Unreachable** 메시지를 활용한다. (뒤에서 자세히 설명)

---

#### 2) Time Exceeded (Type 11)

**상황:**

- 패킷의 TTL이 0이 되어 라우터가 더 이상 포워딩하지 못할 때,
- 조각난(fragmented) 패킷이 재조립 시간 안에 완성되지 못할 때.

**Code:**

- Code 0: TTL exceeded in transit
- Code 1: Fragment reassembly time exceeded

**시나리오 — traceroute의 핵심**

- 송신자가 **TTL=1**인 패킷을 보낸다.
  - 1번째 라우터에서 TTL이 0이 되므로 그 라우터는 **ICMP Time Exceeded**를 송신자에게 보낸다.
- 송신자는 **TTL=2**로 다시 보낸다.
  - 2번째 라우터가 Time Exceeded를 보낸다.
- 이런 식으로 TTL을 늘려 가면서 경로 상의 라우터들을 하나씩 “드러내는” 것이 traceroute의 아이디어다.

---

#### 3) Parameter Problem (Type 12)

**상황:** IPv4 헤더 자체가 잘못되어 있는 경우:

- 잘못된 옵션,
- 필수 옵션 누락,
- 헤더 길이/총 길이 모순 등.

**형식(요약):**

```text
Type=12 | Code | Checksum
Pointer (문제 바이트의 위치)
+ 원본 IP 헤더 + 데이터 일부
```

- 수신 측은 Pointer 값으로 어느 바이트가 문제인지 파악할 수 있다.

---

#### 4) Redirect (Type 5)

**상황:**

- 호스트가 라우팅을 잘못해, 최적이 아닌 라우터로 트래픽을 보내고 있을 때,
- 라우터가 “다음부턴 저 라우터를 통하라”고 알려주는 메커니즘.

**보안·운영 관점:**

- Redirect는 호스트의 라우팅 테이블을 외부에서 수정하는 행위와 비슷하기 때문에,
  많은 운영자들이 **Redirect를 차단**하거나 내부 네트워크 일부에서만 사용하는 방향으로 운영한다.

---

#### 5) Source Quench (Type 4, Deprecated)

- 원래는 혼잡 시 송신자에게 “속도 줄여라”라는 신호를 보내는 용도였으나,
  연구 결과 **혼잡 제어에 비효율적이고, 악용 가능성이 크다**는 이유로,
  RFC 1812와 RFC 6633, RFC 6918을 통해 **발생·처리 모두 공식적으로 폐기**되었다.

현대 스택에서는:

- 라우터는 Source Quench를 **보내지 말아야 하고(MUST NOT)**,
- 호스트도 이를 받으면 **무시해야 한다(MUST silently discard)**.

---

### 대표 정보 메시지

#### 1) Echo Request (Type 8) / Echo Reply (Type 0)

- `ping`의 기반.
- 구조(공통):

```text
Type | Code=0 | Checksum
Identifier | Sequence Number
Data (임의의 바이트들)
```

- Identifier/Sequence Number로 요청·응답 매칭.
- RTT 측정을 위해 송신자가 Data에 타임스탬프를 넣기도 한다.

#### 2) Timestamp Request/Reply (Type 13/14)

- 밀리초 단위 타임스탬프를 교환해 **시간 동기**나 **지연 측정**에 사용하도록 설계.
- NTP 같은 더 세련된 프로토콜이 등장하면서 거의 사용되지 않지만, 구조를 이해해 두면 시간 관련 설계를 할 때 도움이 된다.

---

## Debugging Tools — ping, traceroute, 기타

ICMPv4는 실무에서 **네트워크 디버깅·모니터링**에 필수다. 가장 대표적인 도구가 `ping`, `traceroute`(Windows에선 `tracert`)다.

---

### ping — Echo Request/Reply 기반

#### 동작 개념

1. 클라이언트가 대상 호스트로 **ICMP Echo Request(Type 8)**를 전송.
2. 대상 호스트가 살아 있고 ICMP를 허용한다면, **Echo Reply(Type 0)**를 그대로 되돌려 보냄.
3. 송신자는 왕복 시간을 측정하고, 손실률·지터 등을 통계로 보여준다.

**정리**

- “IP 레벨에서 호스트가 살아 있고, 경로가 있는지”를 확인.
- DNS 문제, 방화벽 문제, L4 문제(TCP/UDP 포트)와는 별개.

---

#### ping 예제 출력 해석 (Linux)

```bash
$ ping -c 4 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=115 time=21.3 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=115 time=21.1 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=115 time=21.5 ms
64 bytes from 8.8.8.8: icmp_seq=4 ttl=115 time=21.0 ms

--- 8.8.8.8 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3005ms
rtt min/avg/max/mdev = 21.016/21.247/21.529/0.222 ms
```

핵심 포인트:

- `icmp_seq`: ICMP Sequence Number, 패킷 순서 식별에 사용
- `ttl`: 수신 시점의 TTL (경로 길이 대략 추정 가능)
- `time`: RTT
- 마지막 통계는 **손실률, rtt 최소/평균/최대**를 요약

---

#### ping + ICMP를 직접 만들어 보는 코드 (Python, raw socket)

> 주의: Raw socket 사용은 OS 권한이 필요하고, 실망스러운 네트워크 환경을 만들면 안 되므로
> **자기 실험 환경에서만** 사용해야 한다.

```python
import os
import socket
import struct
import time

def checksum(data: bytes) -> int:
    if len(data) % 2:
        data += b"\x00"
    s = 0
    for i in range(0, len(data), 2):
        w = data[i] << 8 | data[i+1]
        s += w
        s = (s & 0xFFFF) + (s >> 16)
    return ~s & 0xFFFF

def build_echo(identifier: int, seq: int, payload: bytes) -> bytes:
    # Type=8 (Echo Request), Code=0
    header = struct.pack("!BBHHH", 8, 0, 0, identifier, seq)
    chksum = checksum(header + payload)
    header = struct.pack("!BBHHH", 8, 0, chksum, identifier, seq)
    return header + payload

def icmp_ping_once(dst_ip: str, timeout=1.0):
    sock = socket.socket(socket.AF_INET, socket.SOCK_RAW, socket.IPPROTO_ICMP)
    sock.settimeout(timeout)

    ident = os.getpid() & 0xFFFF
    seq = 1
    payload = struct.pack("!d", time.time())  # 전송 시간 기록
    packet = build_echo(ident, seq, payload)

    t0 = time.time()
    sock.sendto(packet, (dst_ip, 0))

    try:
        data, addr = sock.recvfrom(65535)
    except socket.timeout:
        print("Request timed out")
        return

    t1 = time.time()
    rtt_ms = (t1 - t0) * 1000
    # IP 헤더는 최소 20B이므로 그 뒤에 ICMP를 찾는다 (간단화)
    icmp = data[20:28]
    r_type, r_code, r_sum, r_id, r_seq = struct.unpack("!BBHHH", icmp)
    print(f"Reply from {addr[0]}: type={r_type}, code={r_code}, id={r_id}, "
          f"seq={r_seq}, time={rtt_ms:.2f} ms")

if __name__ == "__main__":
    icmp_ping_once("8.8.8.8")
```

이 코드는 ICMP Echo Request 패킷을 직접 만들고, 수신한 응답의 Type/Code를 출력한다.

---

### traceroute / tracert — Time Exceeded 기반

`traceroute`(유닉스)/`tracert`(Windows)는 모두 **ICMP 메시지를 관찰**해서 경로를 추적한다.

#### 두 가지 구현 방식

1. **UNIX 계열 traceroute**
   - UDP 패킷을 매우 높은 포트(예: 33434)로 보냄.
   - TTL을 1, 2, 3 … 늘리면서 전송.
   - 각 중간 라우터는 TTL=0이 되면 **ICMP Time Exceeded(Type 11)**를 보낸다.
   - 마지막 홉에서는 UDP 포트가 열려 있지 않으므로 **ICMP Destination Unreachable (Port Unreachable, Type 3 Code 3)**가 돌아온다.

2. **Windows tracert**
   - 기본적으로 **ICMP Echo Request**를 사용.
   - TTL을 1, 2, 3 … 늘려 가며 보내고,
     중간 라우터의 Time Exceeded와 마지막 홉의 Echo Reply를 이용해 경로를 표시.

#### traceroute 예시 해석

```bash
$ traceroute 8.8.8.8
 1  192.168.0.1   1.123 ms  0.987 ms  1.045 ms
 2  10.0.0.1      3.456 ms  3.322 ms  3.410 ms
 3  203.0.113.1   8.910 ms  8.875 ms  8.902 ms
...
10  8.8.8.8      20.120 ms 19.950 ms 20.010 ms
```

- 각 줄은 “TTL=N” 패킷에 대해 수신된 ICMP의 **소스 주소**와 RTT.
- 마지막 라인은 Destination Unreachable(Port) 또는 Echo Reply에 도달한 것.

---

### 기타 도구 (mtr, pathping, PMTUD 등)

- **mtr**: `ping` + `traceroute`를 합쳐, 각 홉에 대해 RTT와 패킷 손실률을 지속적으로 측정.
- **pathping (Windows)**: 각 홉의 통계를 계산해 “어디에서 손실이 심한지”를 가시화.
- **Path MTU Discovery (PMTUD)**:
  - DF=1인 패킷을 보내고, 중간에서 단편화가 필요하면 **ICMP Dest Unreachable, Fragmentation needed (Type 3, Code 4)**를 받는다.
  - 이 메시지를 이용해 Path MTU를 점차 줄여 나간다.

---

## ICMP Checksum

### Internet Checksum 개요

ICMPv4 체크섬은 **IPv4/UDP/TCP와 동일한 Internet checksum 알고리즘**을 사용한다. RFC 1071과 RFC 1141에 공식적으로 정리되어 있고, Wikipedia 등에도 예제가 잘 정리되어 있다.

정의(요약):

- ICMP 메시지 전체(헤더 + 데이터)를 **16비트 단위로 나눈다.**
- 모든 16비트 단어를 **1의 보수 덧셈(one’s complement addition)**으로 더한다.
- 그 합의 **1의 보수(비트 반전)**를 취한 값이 체크섬.

수식으로 쓰면, ICMP 메시지를 \(n\)개의 16비트 단어 \(w_1,\dots,w_n\)으로 보고
\(S\)를 1의 보수 덧셈으로 계산한 합이라고 할 때:

$$
S = w_1 \oplus w_2 \oplus \cdots \oplus w_n \quad
(\text{여기서 } \oplus \text{는 16비트 1의 보수 덧셈})
$$

$$
\text{Checksum} = \overline{S}
$$

여기서:

- 1의 보수 덧셈은 **16비트 초과 부분(캐리)을 계속 상위에서 잘라내고 다시 더하는** 연산이다.
- 수신자는 **전체(헤더+데이터)를 다시 합산해서 결과가 0xFFFF인지** 확인한다.
  - 다시 말해, 전체를 더해서 **1의 보수가 0**이면 “에러 없음”으로 본다.

---

### 체크섬 계산 절차 (ICMPv4에 적용)

RFC 792에서 ICMP 헤더의 Checksum 필드는 다음과 같이 정의한다:

1. ICMP 메시지 전체(헤더 + 데이터)를 16비트 단위로 나눈다.
2. Checksum 필드는 **0으로 채운 상태**에서 계산을 시작한다.
3. 모든 16비트 단어를 1의 보수 덧셈으로 더한다.
4. 합의 1의 보수를 취한 값을 Checksum 필드에 넣는다.

검증 시:

- 다시 전체(체크섬 포함)를 같은 방식으로 더해, 결과 합이 0xFFFF면 “문제 없음”으로 본다.

---

### 간단한 수치 예제 — Echo Request

다음과 같은 ICMP Echo Request를 가정해보자.

- Type = 8 (Echo Request)
- Code = 0
- Checksum = 0x0000 (계산 전 초기값)
- Identifier = 0x1234
- Sequence = 0x0001
- Data = ASCII `"ABCD"` (0x41, 0x42, 0x43, 0x44)

16비트 단위로 나누면(네트워크 바이트 순서, big-endian):

- Word 1: Type(8) + Code(0) → `0x0800`
- Word 2: Checksum (초기 0) → `0x0000`
- Word 3: Identifier → `0x1234`
- Word 4: Sequence → `0x0001`
- Word 5: Data[0:2] `"AB"` → 0x41, 0x42 → `0x4142`
- Word 6: Data[2:4] `"CD"` → 0x43, 0x44 → `0x4344`

합을 16비트 1의 보수 덧셈으로 계산해 보자.

1.
   $$
   S_1 = 0x0800 + 0x0000 = 0x0800
   $$
2.
   $$
   S_2 = 0x0800 + 0x1234 = 0x1A34
   $$
3.
   $$
   S_3 = 0x1A34 + 0x0001 = 0x1A35
   $$
4.
   $$
   S_4 = 0x1A35 + 0x4142 = 0x5B77
   $$
5.
   $$
   S_5 = 0x5B77 + 0x4344 = 0x9EBB
   $$

16비트 범위를 넘는 캐리가 없으므로, 최종 합 \(S = 0x9EBB\).

체크섬은 이 값의 **1의 보수**:

$$
\text{Checksum} = \sim 0x9EBB = 0x6144
$$

따라서 이 ICMP Echo Request의 최종 헤더는:

```text
Type=8, Code=0, Checksum=0x6144, Identifier=0x1234, Sequence=0x0001
```

수신 측에서 검증할 때는:

- 위 6개의 단어에 **0x6144**를 포함해서 다시 1의 보수 덧셈을 하면,
- 결과가 0xFFFF(1의 보수 0)이 나온다 → 오류 없음.

---

### 체크섬 계산 코드 예제

#### Python 구현

```python
def internet_checksum(data: bytes) -> int:
    # 길이가 홀수면 마지막에 0을 붙여 짝수로 맞춤
    if len(data) % 2:
        data += b"\x00"

    s = 0
    # 16비트 단위로 잘라서 더함
    for i in range(0, len(data), 2):
        w = (data[i] << 8) + data[i+1]
        s += w
        # 16비트 초과 캐리를 다시 더하는 1의 보수 덧셈
        s = (s & 0xFFFF) + (s >> 16)

    # 1의 보수
    return ~s & 0xFFFF

# 예: 위에서 사용한 Echo Request 헤더+데이터

header = bytes.fromhex("08 00 00 00 12 34 00 01")
payload = b"ABCD"
chk = internet_checksum(header + payload)
print(hex(chk))  # 0x6144
```

실제 ICMP 송신 단계에서는:

1. `Checksum` 필드를 0으로 채운 헤더 + 데이터를 준비하고,
2. 위 함수를 적용해 나온 값을 다시 헤더의 Checksum에 채워 넣는다.

---

#### C 스타일 의사코드

RFC 1071에 나오는 알고리즘을 C 스타일로 요약하면 다음과 같다 (16비트 배열 `addr`, 길이 `len`):

```c
unsigned short in_cksum(unsigned short *addr, int len)
{
    unsigned long sum = 0;

    while (len > 1) {
        sum += *addr++;
        len -= 2;
    }

    // 남는 바이트가 있으면 하위 8비트만 더함
    if (len == 1) {
        unsigned short last = 0;
        *(unsigned char*)(&last) = *(unsigned char*)addr;
        sum += last;
    }

    // 16비트를 넘어가는 캐리를 계속 접어서 더한다
    while (sum >> 16) {
        sum = (sum & 0xFFFF) + (sum >> 16);
    }

    // 1의 보수
    return (unsigned short)(~sum);
}
```

ICMP 헤더 구조체 포인터와 데이터 포인터를 연결해서 이 함수에 넘기면 된다.

---

### ICMP 체크섬과 보안

- **역할**: 전송 중의 우발적인 비트 에러를 검출.
- **한계**:
  - 공격자가 패킷을 조작한 뒤 체크섬을 다시 계산해 넣으면 되므로,
  - **무결성/인증/기밀성 보장은 전혀 없다.**
- 따라서, ICMP를 포함한 IPv4 트래픽을 보호하려면:
  - **IPsec**(AH/ESP),
  - 또는 TLS, VPN 같은 **상위 계층 보안 프로토콜**이 별도로 필요하다.

---

## 정리

이번 19.2 절에서는 **ICMPv4**를 다음 세 관점에서 정리했다.

1. **Messages**
   - ICMPv4는 IPv4(Protocol=1)의 제어/진단용 프로토콜.
   - Type/Code 조합으로 다양한 메시지를 표현하며,
     - 에러: Destination Unreachable, Time Exceeded, Parameter Problem, Redirect, (역사적) Source Quench 등
     - 정보: Echo Request/Reply, Timestamp, Router Solicitation/Advertisement 등
   - 많은 타입이 현대 인터넷에서 비활성화되거나 Deprecated 상태이며, 공식 IANA 레지스트리에서 지속적으로 관리된다.

2. **Debugging Tools**
   - `ping`: Echo Request/Reply를 이용해 **reachability, RTT, 손실률**을 측정.
   - `traceroute`/`tracert`: TTL을 조작해 각 홉에서 오는 Time Exceeded/Unreachable 메시지로 **경로를 추적**.
   - Path MTU Discovery, mtr, pathping 등도 ICMP 에러 메시지를 활용해 네트워크 상태를 진단한다.

3. **ICMP Checksum**
   - 전체 ICMP 메시지(헤더 + 데이터)에 대해 **16비트 Internet checksum**을 계산.
   - 1의 보수 덧셈 + 1의 보수 연산으로 정의되며, RFC 1071/1141에 효율적인 구현 기법이 정리되어 있다.
   - 에러 검출용일 뿐, 보안(무결성/인증)을 위한 것은 아니다.

앞에서 정리한 IPv4 데이터그램 구조(19.1)와 이어서 보면,
“**IPv4 헤더 위에 ICMP가 어떻게 실리고, 어떤 타입/코드로 어떤 상황을 표현하는지,
그리고 실제 도구가 어떤 메시지를 관찰하는지**”가 전체적으로 연결될 것이다.
