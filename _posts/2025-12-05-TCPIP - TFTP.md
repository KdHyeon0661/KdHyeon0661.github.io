---
layout: post
title: TCPIP - TFTP
date: 2025-12-05 19:25:23 +0900
category: TCPIP
---
# 단순 파일 전송 프로토콜 (TFTP)

## TFTP 개요, 역사 및 표준

### 단순함의 철학
Trivial File Transfer Protocol (TFTP)은 1980년대 초반 개발된 극도로 단순한 파일 전송 프로토콜입니다. FTP의 복잡성과 대조적으로, TFTP는 가장 기본적인 파일 전송 기능만 제공하도록 의도적으로 설계되었습니다. "단순함이 미덕이다"는 철학을 구현한 TFTP는 정말로 '사소한'(trivial) 수준의 기능만을 제공하며, 이로 인해 구현이 간단하고 리소스 사용이 최소화됩니다.

### 역사적 배경과 필요성
TFTP의 기원은 1981년으로 거슬러 올라갑니다. 당시 네트워크 부팅이나 디스크 없는 워크스테이션(Diskless Workstation)이 새로운 개념으로 등장하면서, 작고 효율적인 파일 전송 프로토콜이 필요했습니다. 이러한 시스템들은 제한된 메모리와 처리 능력을 가졌기 때문에 FTP와 같은 완전한 기능의 프로토콜을 실행할 수 없었습니다.

TFTP의 역사적 발전 타임라인:
- 1981: TFTP 초기 버전 (RFC 783)
  - 최초 표준화
  - 블록 번호 8비트 제한
  - 네트워크 부팅용으로 개발
- 1984: TFTP 개정 (RFC 906)
  - XNS/Xerox 환경에서의 사용 정의
- 1992: TFTP 옵션 확장 (RFC 1350)
  - 블록 번호 16비트로 확장
  - 현재의 표준 버전
- 1995: TFTP 블록 크기 옵션 (RFC 1782)
  - 블록 크기 협상 도입
- 1998: TFTP 타임아웃 간격 옵션 (RFC 2349)
  - 시간 초과 및 전송 크기 옵션
- 2005: TFTP 윈도우 크기 옵션 (제안)
  - 성능 향상을 위한 윈도잉

### 설계 목표와 핵심 원칙
TFTP의 설계자들은 다음과 같은 원칙을 우선시했습니다:
1. **최소주의**: 가능한 한 가장 작은 프로토콜 구현
2. **저자원**: 제한된 메모리와 CPU에서 실행 가능
3. **신뢰성**: UDP 위에서 신뢰할 수 있는 전송 보장
4. **단방향성**: 한 번에 한 방향으로만 전송

### 표준화 과정
TFTP 표준 문서 진화:

| RFC      | 연도 | 주요 내용                              | 상태          |
|----------|------|----------------------------------------|---------------|
| RFC 783  | 1981 | 초기 TFTP 사양                         | 구식          |
| RFC 906  | 1984 | 부트스트랩 로딩을 위한 TFTP            | 구식          |
| RFC 1350 | 1992 | TFTP 개정판 2 (현재 표준)              | 인터넷 표준   |
| RFC 1782 | 1995 | TFTP 블록 크기 옵션                    | 제안 표준     |
| RFC 2347 | 1998 | TFTP 옵션 확장                         | 제안 표준     |
| RFC 2348 | 1998 | TFTP 블록 크기 옵션                    | 제안 표준     |
| RFC 2349 | 1998 | TFTP 타임아웃 간격 및 전송 크기 옵션   | 제안 표준     |

## TFTP 일반 운영, 연결 설정 및 클라이언트/서버 통신

### 연결 없는 통신 모델
TFTP는 TCP를 사용하는 FTP와 달리 UDP(User Datagram Protocol)를 기반으로 합니다. 이는 연결 설정 과정 없이 즉시 데이터 전송을 시작할 수 있음을 의미합니다. 하지만 UDP는 신뢰성 있는 전송을 보장하지 않으므로, TFTP는 자체적인 신뢰성 메커니즘을 구현해야 했습니다.

TFTP vs FTP 기본 비교:

| 특성         | TFTP               | FTP                |
|--------------|--------------------|--------------------|
| 전송 프로토콜 | UDP (포트 69)      | TCP (포트 20,21)   |
| 인증         | 없음               | 사용자명/암호      |
| 명령 세트    | 읽기/쓰기만        | 다양함             |
| 데이터 연결  | 없음 (단일 포트)   | 별도 데이터 연결   |
| 전송 모드    | 옥텟 모드만        | 여러 모드          |
| 구현 크기    | 매우 작음 (KB)     | 큼 (MB)            |

### 포트 할당 및 통신 모델
TFTP는 잘 알려진 포트 69를 사용하지만, 실제 데이터 전송은 동적으로 할당된 고차원 포트에서 이루어집니다:

TFTP 포트 사용 시퀀스:
1. 클라이언트 → 서버 (포트 69)
   - RRQ 또는 WRQ 메시지 전송
2. 서버 → 클라이언트 (랜덤 고차원 포트)
   - ACK 또는 DATA 메시지 전송
   - 포트: 클라이언트의 요청 포트
3. 클라이언트 ↔ 서버 (모두 고차원 포트)
   - 실제 데이터 전송
   - 서버: 고정 포트 69 사용 중단
   - 새로운 포트에서 통신 계속

### 연결 설정 과정
TFTP 연결은 매우 단순한 요청-응답 모델을 따릅니다:

읽기 요청(RRQ) 시작:
```
클라이언트                            서버
     |                                   |
     |--- RRQ 메시지 (포트 69) -------->|
     | • 파일명: "bootimage.bin"        |
     | • 모드: "octet"                  |
     |                                   |
     |                                   | 파일 존재 확인
     |                                   | 포트 할당
     |<-- DATA 블록 1 (포트 X) ---------|
     | • 블록 번호: 1                   |
     | • 데이터: 512바이트              |
     |                                   |
     |--- ACK 1 (포트 X) -------------->|
     | • 블록 번호: 1                   |
     |                                   |
데이터 전송 계속...
```

쓰기 요청(WRQ) 시작:
```
클라이언트                            서버
     |                                   |
     |--- WRQ 메시지 (포트 69) -------->|
     | • 파일명: "config.cfg"           |
     | • 모드: "octet"                  |
     |                                   |
     |                                   | 파일 생성 준비
     |                                   | 포트 할당
     |<-- ACK 0 (포트 X) ---------------|
     | • 블록 번호: 0                   |
     |                                   |
     |--- DATA 블록 1 (포트 X) ---------|
     | • 블록 번호: 1                   |
     | • 데이터: 512바이트              |
     |                                   |
데이터 전송 계속...
```

### 핸드오프(Handoff) 메커니즘
TFTP의 독특한 특징 중 하나는 초기 요청 후 포트가 변경된다는 점입니다:

```python
def tftp_server_handoff(client_addr, client_port, filename, mode):
    """
    TFTP 서버 포트 핸드오프 처리
    """
    # 1. 초기 요청 수신 (포트 69)
    initial_request = receive_udp(port=69)
    
    # 2. 새로운 임시 포트 할당
    new_port = allocate_ephemeral_port()
    
    # 3. 새 포트로 응답 전송
    if initial_request.type == "RRQ":
        # 파일 존재 확인
        if file_exists(filename):
            first_block = read_file_block(filename, block_num=1)
            send_udp(client_addr, client_port, new_port, 
                     DATA(block_num=1, data=first_block))
        else:
            send_error(client_addr, client_port, "File not found")
    
    elif initial_request.type == "WRQ":
        # 파일 생성 준비
        create_file(filename)
        send_udp(client_addr, client_port, new_port,
                 ACK(block_num=0))
    
    # 4. 포트 69 연결 종료, 새 포트에서 계속
    return new_port
```

## TFTP 상세 운영 및 메시징

### 전송 상태 머신(State Machine)
TFTP 클라이언트와 서버는 간단하지만 정확한 상태 머신을 구현합니다:

```
TFTP 클라이언트 상태 다이어그램 (읽기 전용 예시)
+----------+     +----------+     +----------+
|   초기   |---->|  RRQ 전송 |---->| 데이터   |
|   상태   |     |          |     |  대기    |
+----------+     +----------+     +----------+
                       |                |
                       |                | 데이터 수신
                       |                |
                       v                v
                  +----------+     +----------+
                  |  오류    |<----|  ACK 전송 |
                  |  처리    |     |          |
                  +----------+     +----------+
                                         |
                                         | 모든 데이터 수신
                                         v
                                    +----------+
                                    |  완료    |
                                    +----------+
```

### 신뢰성 보장 메커니즘
UDP의 비신뢰성을 극복하기 위해 TFTP는 STOP-AND-WAIT ARQ(Automatic Repeat reQuest) 프로토콜을 구현합니다:

STOP-AND-WAIT ARQ 작동 원리:
```
송신자                           수신자
    |                               |
    |--- DATA 블록 N ------------>|
    |                               |
    |   데이터 처리 및 저장        |
    |   ACK 준비                   |
    |<-- ACK N --------------------|
    |                               |
    |   ACK 확인                   |
    |   다음 블록 준비             |
    |--- DATA 블록 N+1 ---------->|
    |                               |

타임아웃 처리:
송신자가 ACK를 기다림 (기본 5초)
    ↓
ACK 수신 안 됨
    ↓
동일 블록 재전송
    ↓
최대 재전송 횟수 초과 시 오류
```

### 데이터 블록 관리
TFTP는 고정 크기 데이터 블록을 사용하며, 마지막 블록은 512바이트 미만으로 파일의 끝을 나타냅니다:

```python
class TftpBlockManager:
    def __init__(self, block_size=512):
        self.block_size = block_size
        self.current_block = 0 if writing else 1
        self.last_block_size = 0
        
    def process_data_block(self, data, block_num):
        """데이터 블록 처리"""
        # 블록 번호 검증
        expected_block = self.current_block
        
        if block_num < expected_block:
            # 이전 블록의 중복 수신
            return "duplicate", None
            
        elif block_num > expected_block:
            # 블록 순서 오류
            return "sequence_error", None
            
        else:  # block_num == expected_block
            # 정상적인 블록 수신
            self.current_block += 1
            self.last_block_size = len(data)
            
            # 마지막 블록 확인 (512바이트 미만)
            if len(data) < self.block_size:
                return "last_block", data
            else:
                return "normal", data
    
    def is_transfer_complete(self):
        """전송 완료 여부 확인"""
        return self.last_block_size < self.block_size
```

### 오류 처리 및 복구
TFTP는 제한적이지만 효과적인 오류 처리 메커니즘을 가지고 있습니다:

TFTP 오류 코드 및 의미:

| 오류 코드 | 의미             | 일반적인 시나리오                     |
|-----------|------------------|----------------------------------------|
| 0         | 정의되지 않음    | 명시되지 않은 오류                     |
| 1         | 파일 없음        | RRQ에 대한 파일이 존재하지 않음        |
| 2         | 접근 위반        | 권한 없이 파일에 접근 시도             |
| 3         | 디스크 가득 참   | 저장 공간 부족                         |
| 4         | 잘못된 작업      | 지원하지 않는 작업 요청                |
| 5         | 알 수 없는 전송 ID | 포트 번호 불일치                      |
| 6         | 파일 이미 존재   | WRQ에 대한 파일이 이미 존재            |
| 7         | 알 수 없는 사용자 | (일반적으로 사용되지 않음)             |

오류 메시지 처리 흐름:
```python
def handle_tftp_error(error_code, error_msg, remote_host, remote_port):
    """TFTP 오류 처리"""
    error_messages = {
        0: "정의되지 않은 오류",
        1: "파일을 찾을 수 없습니다",
        2: "접근이 거부되었습니다",
        3: "디스크가 가득 찼습니다",
        4: "잘못된 TFTP 작업입니다",
        5: "알 수 없는 전송 ID입니다",
        6: "파일이 이미 존재합니다",
        7: "알 수 없는 사용자입니다"
    }
    
    # 오류 메시지 생성
    error_message = error_messages.get(error_code, "알 수 없는 오류")
    if error_msg:
        error_message += f": {error_msg}"
    
    # 오류 패킷 생성 및 전송
    error_packet = create_error_packet(error_code, error_message)
    send_udp(remote_host, remote_port, error_packet)
    
    # 연결 종료
    cleanup_connection()
    
    return error_message
```

## TFTP 옵션 및 옵션 협상

### 옵션 확장의 진화
원래 TFTP는 매우 단순했지만, 실무적 필요에 의해 옵션 협상 기능이 추가되었습니다. RFC 2347은 TFTP 옵션 협상을 위한 프레임워크를 정의합니다.

### 주요 옵션 타입

#### 1. 블록 크기 옵션 (blksize)
가장 중요한 옵션으로, 기본 512바이트 블록 크기를 변경할 수 있습니다:

블록 크기 옵션 협상:
```
클라이언트                            서버
     |                                   |
     |--- RRQ -------------------------->|
     | • 파일명: "largefile.iso"         |
     | • 모드: "octet"                   |
     | • 옵션: blksize=8192              |
     |                                   |
     |                                   | 옵션 지원 확인
     |<-- OACK --------------------------|
     | • 옵션: blksize=8192              |
     |                                   |
     |--- ACK (블록 0) ----------------->|
     | • 옵션 수락 확인                  |
     |                                   |
     |<-- DATA 블록 1 -------------------|
     | • 블록 번호: 1                    |
     | • 데이터: 8192바이트              |
     |                                   |
```

블록 크기 선택 고려사항:
```python
def calculate_optimal_block_size(mtu, overhead=8):
    """
    최적의 TFTP 블록 크기 계산
    MTU: 최대 전송 단위 (일반적으로 1500)
    overhead: UDP/IP 헤더 오버헤드 (8바이트)
    """
    # IP MTU에서 오버헤드 제외
    max_data_size = mtu - overhead
    
    # TFTP 데이터 패킷 구조 고려
    # 2바이트(오퍼레이션 코드) + 2바이트(블록 번호) + 데이터
    tftp_overhead = 4
    optimal_size = max_data_size - tftp_overhead
    
    # 일반적인 값으로 조정
    common_sizes = [512, 1024, 2048, 4096, 8192]
    
    for size in common_sizes:
        if size <= optimal_size:
            return size
    
    return max(common_sizes)  # 가장 큰 허용 크기
```

#### 2. 타임아웃 간격 옵션 (timeout)
전송 타임아웃을 사용자 정의할 수 있습니다:

```python
class TftpTimeoutManager:
    def __init__(self, default_timeout=5.0):
        self.base_timeout = default_timeout
        self.current_timeout = default_timeout
        self.retry_count = 0
        self.max_retries = 5
        
    def negotiate_timeout(self, requested_timeout):
        """타임아웃 값 협상"""
        # 허용 범위 확인 (1-255초)
        if 1 <= requested_timeout <= 255:
            self.current_timeout = requested_timeout
            return requested_timeout
        else:
            # 범위 밖이면 기본값 사용
            return self.base_timeout
    
    def calculate_retry_timeout(self):
        """재전송 타임아웃 계산 (지수 백오프)"""
        timeout = self.current_timeout * (2 ** self.retry_count)
        
        # 최대값 제한 (60초)
        timeout = min(timeout, 60.0)
        
        self.retry_count += 1
        return timeout
    
    def reset_retry_count(self):
        """재시도 카운트 초기화"""
        self.retry_count = 0
```

#### 3. 전송 크기 옵션 (tsize)
파일 크기를 사전에 알려서 진행 상황 추적을 가능하게 합니다:

전송 크기 옵션 활용 시나리오:
1. 클라이언트 → 서버 (RRQ):
   - 옵션: tsize=0 (크기 요청)
2. 서버 → 클라이언트 (OACK):
   - 옵션: tsize=1048576 (1MB 파일)
3. 클라이언트:
   - 전체 크기 알게 됨
   - 진행률 표시 가능
   - 필요한 저장 공간 확인
4. 전송 중:
   - 수신한 바이트 수로 진행률 계산
   - 예상 완료 시간 추정

### 옵션 협상 프로토콜
옵션 협상은 OACK 메시지를 통해 이루어집니다:

```python
def negotiate_tftp_options(client_request):
    """TFTP 옵션 협상 처리"""
    requested_options = parse_options(client_request)
    negotiated_options = {}
    
    for option_name, option_value in requested_options.items():
        if option_name == "blksize":
            # 블록 크기 협상
            try:
                blksize = int(option_value)
                if 8 <= blksize <= 65464:  # RFC 2348 범위
                    negotiated_options["blksize"] = blksize
                else:
                    # 범위 밖: 기본값 유지
                    pass
            except ValueError:
                # 유효하지 않은 값: 무시
                pass
                
        elif option_name == "timeout":
            # 타임아웃 협상
            try:
                timeout = int(option_value)
                if 1 <= timeout <= 255:
                    negotiated_options["timeout"] = timeout
            except ValueError:
                pass
                
        elif option_name == "tsize":
            # 전송 크기 협상
            if option_value == "0":  # 클라이언트가 크기 요청
                file_size = get_file_size(client_request.filename)
                negotiated_options["tsize"] = str(file_size)
            else:
                # 클라이언트가 크기 제공 (WRQ 시)
                negotiated_options["tsize"] = option_value
    
    # OACK 메시지 생성
    if negotiated_options:
        return create_oack_message(negotiated_options)
    else:
        # 협상된 옵션 없으면 일반 응답
        return None
```

## TFTP 메시지 형식

### 기본 메시지 구조
TFTP는 5가지 기본 메시지 유형을 정의합니다. 모든 메시지는 UDP 데이터그램으로 전송되며, 간단한 바이너리 형식을 따릅니다.

TFTP 메시지 유형 개요:

| 오퍼레이션 | 값   | 이름               | 용도                |
|------------|------|--------------------|---------------------|
| 1          | 0x01 | RRQ (Read Request)  | 파일 읽기 요청      |
| 2          | 0x02 | WRQ (Write Request) | 파일 쓰기 요청      |
| 3          | 0x03 | DATA (Data)         | 데이터 전송         |
| 4          | 0x04 | ACK (Acknowledgment)| 확인 응답           |
| 5          | 0x05 | ERROR (Error)       | 오류 보고           |
| 6          | 0x06 | OACK (Option Ack)   | 옵션 협상 응답      |

### 상세 메시지 형식

#### 1. RRQ/WRQ 메시지 형식
읽기/쓰기 요청 메시지는 다음과 같은 구조를 가집니다:

RRQ/WRQ 메시지 구조 (바이트 단위):
```
오퍼레이션 (2바이트) | 파일명 (가변) | 0 (1바이트) | 모드 (가변) | 0 (1바이트) | 옵션1 (가변) | 옵션값1 (가변)
```

옵션 확장 시:
```
... | 0 (1바이트) | 옵션N (가변) | 옵션값N (가변) | 0 (1바이트)
```

실제 바이너리 예시 (16진수):
```
RRQ 요청: "test.txt" 파일, 옥텟 모드
00 01 74 65 73 74 2E 74 78 74 00 6F 63 74 65 74 00
```

구성 요소:
- 00 01: RRQ 오퍼레이션 코드
- 74 65 73 74 2E 74 78 74: "test.txt" (ASCII)
- 00: 널 종결자
- 6F 63 74 65 74: "octet"
- 00: 널 종결자

모드 문자열 유형:
- **netascii**: 텍스트 파일 (CR/LF 변환)
- **octet**: 바이너리 파일 (원시 데이터)
- **mail**: 구식 모드 (현재 거의 사용 안 함)

#### 2. DATA 메시지 형식
데이터 전송 메시지는 가장 빈번히 사용되는 TFTP 메시지입니다:

DATA 메시지 구조:
```
오퍼레이션 (2바이트) | 블록 번호 (2바이트) | 데이터 (0-512바이트)
```

마지막 블록 규칙:
- 데이터 크기가 512바이트 미만이면 마지막 블록
- 빈 파일: 데이터 크기 0의 DATA 패킷 하나로 전송
- 블록 번호: 1부터 시작 (WRQ 시 ACK 0 후 블록 1 시작)

바이너리 예시:
```
DATA 블록 3, 512바이트 데이터
00 03 00 03 [512바이트 데이터...]
```

#### 3. ACK 메시지 형식
확인 응답 메시지는 매우 간단한 구조입니다:

ACK 메시지 구조:
```
오퍼레이션 (2바이트) | 블록 번호 (2바이트)
```

특수 케이스:
- WRQ 시작 시: ACK 0 전송 (블록 번호 0)
- 데이터 블록 N에 대한 응답: ACK N
- 옵션 협상 후: ACK 0 (OACK 확인용)

바이너리 예시:
```
ACK 블록 42
00 04 00 2A  (0x2A = 10진수 42)
```

#### 4. ERROR 메시지 형식
오류 보고 메시지는 오류 코드와 설명을 포함합니다:

ERROR 메시지 구조:
```
오퍼레이션 (2바이트) | 오류 코드 (2바이트) | 오류 메시지 (가변 길이) | 0 (1바이트)
```

바이너리 예시:
```
"File not found" 오류 (코드 1)
00 05 00 01 46 69 6C 65 20 6E 6F 74 20 66 6F 75 6E 64 00
```

구성 요소:
- 00 05: ERROR 오퍼레이션
- 00 01: 오류 코드 1
- 46 69 6C 65 ...: "File not found" (ASCII)
- 00: 널 종결자

#### 5. OACK 메시지 형식
옵션 협상 응답 메시지는 RRQ/WRQ와 유사한 형식을 가집니다:

OACK 메시지 구조:
```
오퍼레이션 (2바이트) | 옵션1 (가변) | 옵션값1 (가변) | 0 (1바이트) | 옵션N (가변)
```

바이너리 예시:
```
blksize=8192, timeout=10 협상 응답
00 06 62 6C 6B 73 69 7A 65 00 38 31 39 32 00 74 69 6D 65 6F 75 74 00 31 30 00
```

구성 요소:
- 00 06: OACK 오퍼레이션
- 62 6C 6B 73 69 7A 65: "blksize"
- 00: 널 종결자
- 38 31 39 32: "8192"
- 00: 널 종결자
- 74 69 6D 65 6F 75 74: "timeout"
- 00: 널 종결자
- 31 30: "10"
- 00: 널 종결자

### 메시지 처리 상태 머신

서버 측 메시지 처리:
```python
class TftpServerStateMachine:
    def __init__(self):
        self.state = "IDLE"
        self.current_transfer = None
        
    def process_message(self, message, client_addr):
        """들어오는 메시지 처리"""
        opcode = message[0:2]
        
        if opcode == b'\x00\x01':  # RRQ
            if self.state == "IDLE":
                return self.handle_rrq(message, client_addr)
                
        elif opcode == b'\x00\x02':  # WRQ
            if self.state == "IDLE":
                return self.handle_wrq(message, client_addr)
                
        elif opcode == b'\x00\x03':  # DATA
            if self.state == "RECEIVING":
                return self.handle_data(message, client_addr)
                
        elif opcode == b'\x00\x04':  # ACK
            if self.state == "SENDING":
                return self.handle_ack(message, client_addr)
                
        elif opcode == b'\x00\x05':  # ERROR
            return self.handle_error(message, client_addr)
            
        elif opcode == b'\x00\x06':  # OACK
            return self.handle_oack(message, client_addr)
            
        else:
            # 알 수 없는 오퍼레이션 코드
            return self.send_error(client_addr, 4, "Illegal TFTP operation")
    
    def handle_rrq(self, message, client_addr):
        """RRQ 처리"""
        # 파일명, 모드, 옵션 파싱
        filename, mode, options = parse_rrq_wrq(message)
        
        # 파일 존재 확인
        if not file_exists(filename):
            return self.send_error(client_addr, 1, "File not found")
        
        # 옵션 협상
        if options:
            negotiated = negotiate_options(options)
            if negotiated:
                # OACK 전송
                self.state = "NEGOTIATING"
                return send_oack(client_addr, negotiated)
        
        # 파일 열기 및 첫 번째 블록 전송
        self.current_transfer = open_file(filename, "rb")
        first_block = read_block(self.current_transfer, block_size=512)
        
        self.state = "SENDING"
        return send_data(client_addr, block_num=1, data=first_block)
```

### 블록 번호 롤오버 처리
블록 번호는 16비트 부호 없는 정수로, 0에서 65535까지의 값을 가질 수 있습니다:

```python
def handle_block_number_rollover(current_block, received_block):
    """
    블록 번호 롤오버 처리
    TFTP 블록 번호는 16비트 (0-65535)
    65535 이후 0으로 돌아감
    """
    # 블록 번호 비교 (롤오버 고려)
    diff = (received_block - current_block) % 65536
    
    if diff == 0:
        return "current"  # 현재 블록
    elif diff == 1:
        return "next"     # 다음 블록
    elif diff > 1 and diff < 32768:
        return "future"   # 미래 블록 (순서 오류)
    else:
        # diff >= 32768: 과거 블록 (재전송)
        return "past"
    
def increment_block_number(block_num):
    """블록 번호 증가 (롤오버 처리)"""
    return (block_num + 1) % 65536
```

## TFTP vs FTP 상세 비교

### 설계 철학 비교

| 측면 | TFTP | FTP |
|------|------|-----|
| 철학 | "단순함이 미덕" | "완전한 기능 제공" |
| 목표 | 최소한의 기능으로 제한된 환경 작동 | 다양한 파일 작업 지원 |
| 사용처 | 네트워크 부팅, 임베디드 시스템 | 일반 파일 전송, 서버 관리 |
| 적합 환경 | 저자원 시스템, 자동화 스크립트 | 인간 상호작용, GUI 클라이언트 |

### 기술적 차이점

#### 1. 프로토콜 계층 비교
TFTP:
- 전송 계층: UDP (비연결형)
- 포트: 69 (초기), 이후 동적 포트
- 신뢰성: 응용 계층에서 구현 (Stop-and-Wait ARQ)

FTP:
- 전송 계층: TCP (연결형)
- 포트: 21(제어), 20(데이터-액티브 모드)
- 신뢰성: 전송 계층에서 보장 (TCP)

#### 2. 연결 모델 비교
TFTP 연결 모델:
- 단일 요청-응답 사이클
- 연결 설정 없음 (UDP)
- 상태 유지: 최소한 (현재 블록 번호만)

FTP 연결 모델:
- 이중 연결 (제어 + 데이터)
- 세션 상태 유지 (로그인, 현재 디렉토리 등)
- 액티브/패시브 모드 지원

#### 3. 보안 및 인증
TFTP 보안:
- 인증: 없음
- 암호화: 없음
- 접근 제어: 파일 시스템 권한에 의존
- 취약점: 스푸핑, 중간자 공격에 취약

FTP 보안:
- 기본 인증: 사용자명/암호 (평문)
- 암호화: FTPS (FTP over SSL/TLS)
- 익명 접근: 지원 (제한적)
- 취약점: 평문 전송, 브루트 포스 공격

### 성능 및 효율성 비교

#### 1. 전송 효율성
TFTP 효율성:
- Stop-and-Wait ARQ: 낮은 대역폭 활용률
- 작은 블록 크기(512바이트): 오버헤드 큼
- UDP 기반: 빠른 시작, 낮은 오버헤드
- 옵션 확장으로 부분적 개선 가능

FTP 효율성:
- TCP 스트림: 높은 대역폭 활용률
- 큰 버퍼 크기: 효율적 대용량 전송
- 이중 연결: 제어/데이터 분리로 효율적
- 재전송: TCP가 처리 (전송 계층)

#### 2. 리소스 사용량
TFTP 리소스:
- 메모리: 수 KB (매우 작음)
- CPU: 최소한의 처리
- 구현 크기: 수백 라인의 코드
- 적합 대상: 부트로더, 라우터, 임베디드

FTP 리소스:
- 메모리: 수 MB
- CPU: 상대적으로 높음
- 구현 크기: 수만 라인의 코드
- 적합 대상: 서버, 데스크톱 시스템

### 기능적 비교 표

| 기능 | TFTP | FTP |
|------|------|-----|
| 파일 작업 | 읽기/쓰기만 | 읽기, 쓰기, 삭제, 이름 변경, 권한 변경 |
| 디렉토리 | 미지원 | 목록 보기, 변경, 생성, 삭제 |
| 인증 | 없음 | 사용자/암호, 익명 |
| 전송 모드 | 옥텟(바이너리)만 | ASCII, 이미지, EBCDIC |
| 재개 기능 | 미지원 | 지원 (REST 명령) |
| 압축 | 없음 | 지원 (MODE C) |
| 대용량 파일 | 블록 번호 제한(65535블록) | 무제한 |
| 동시 전송 | 단일 파일 | 다중 파일 (mget/mput) |

### 실제 사용 사례 비교

#### TFTP의 전형적 사용 사례:
1. **네트워크 부팅 (PXE)**
   - 클라이언트가 DHCP를 통해 IP 주소를 획득합니다.
   - 클라이언트가 TFTP를 사용하여 부트 이미지(부트로더, 커널 등)를 다운로드합니다.
   - 서버 측에서는 간단한 TFTP 서버만 실행하면 됩니다.

2. **라우터/스위치 구성 관리**
   - 구성 파일 백업: 네트워크 장비의 설정을 TFTP 서버로 업로드하여 백업합니다.
   - 펌웨어 업데이트: TFTP를 통해 새로운 펌웨어 이미지를 다운로드하여 장비를 업데이트합니다.
   - 자동화 스크립트와의 통합이 용이하여 대규모 장비 관리에 적합합니다.

3. **임베디드 시스템**
   - 메모리나 처리 능력이 제한된 환경에서 사용됩니다.
   - 단순한 프로토콜 스택만 구현하면 되어 시스템 자원을 적게 사용합니다.
   - 자동화된 배포 프로세스에 통합하기 쉬워 대량 생산 환경에 적합합니다.

#### FTP의 전형적 사용 사례:
1. **웹 호스팅 파일 관리**
   - 웹마스터가 HTML, CSS, JavaScript 파일을 서버에 업로드할 때 사용합니다.
   - 대용량 데이터베이스 백업 파일을 전송할 때 적합합니다.
   - 디렉토리 구조를 유지하면서 파일을 관리해야 할 때 필요합니다.

2. **기업 내부 파일 공유**
   - 부서 간 대용량 파일(설계도, 보고서, 미디어 파일 등)을 교환할 때 사용합니다.
   - 권한 기반 접근 제어가 필요한 내부 자료실 역할을 합니다.
   - 익명 또는 인증된 사용자를 위한 파일 다운로드 센터로