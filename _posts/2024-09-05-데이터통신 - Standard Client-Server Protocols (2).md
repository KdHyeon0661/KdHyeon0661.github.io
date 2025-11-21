---
layout: post
title: 데이터 통신 - Standard Client-Server Protocols (2)
date: 2024-09-05 23:20:23 +0900
category: DataCommunication
---
# FTP — 두 개의 연결, 제어/데이터 채널, 보안까지 완전 정리

이 절에서는 **FTP(File Transfer Protocol)** 의 고전적인 구조를 그대로 따라가면서도,
2020년대 기준의 **보안·운영 관점**을 반영해서 정리한다.

특히 교과서에서 강조하는 네 가지 키워드:

- **Two Connections**
- **Control Connection**
- **Data Connection**
- **Security for FTP**

을 실제 세션 예제, 코드 예제, 다이어그램과 함께 풀어 본다.

---

## Two Connections — FTP만의 독특한 “이중 연결” 구조

### 1) 왜 굳이 연결을 두 개나 쓸까?

FTP는 RFC 959에서 처음 표준화된, **가장 오래된 응용계층 프로토콜** 중 하나다.

FTP 설계자들이 중요하게 본 것은:

1. **대화(control)와 데이터(file)를 분리**
2. **파일 전송 중에도 추가 명령(중지, 중단 등)을 보내고 싶다**
3. **세션을 유지한 상태에서 파일 여러 개를 연속 전송하고 싶다**

그래서 HTTP처럼 “요청 한 번에 응답 한 번” 구조가 아니라,

- **제어 연결(control connection)**: 명령/응답만 오가는 TCP 연결 (보통 포트 21)
- **데이터 연결(data connection)**: 파일/디렉터리 목록 등 실제 데이터가 오가는 TCP 연결

을 **별도의 TCP 연결**로 분리하는 구조를 택했다.

### 2) FTP 세션의 큰 흐름

가장 기본적인 FTP 세션을 순서대로 적어 보면:

```text
[1] 클라이언트 → 서버: TCP 연결 생성 (포트 21)  → "제어 연결" open
[2] 서버 → 클라이언트: 220 Service ready 응답
[3] 클라이언트 ↔ 서버: USER / PASS / 기타 명령
[4] 클라이언트 → 서버: 파일 전송 명령 (RETR, STOR, LIST 등)
[5] 클라이언트 ↔ 서버: TCP "데이터 연결" 생성 (포트 20 또는 임의 포트)
[6] 데이터 연결에서 실제 파일/목록 전송
[7] 데이터 연결 종료
[8] 필요하면 [4]~[7] 반복 (같은 제어 연결 유지)
[9] 클라이언트 → 서버: QUIT
[10] 제어 연결 종료
```

이를 그림으로 정리하면:

```text
제어 연결 (포트 21, 세션 내내 유지)
   ┌───────────────────────────────────────────────┐
   │ USER / PASS / CWD / LIST / RETR / STOR / ... │
   └───────────────────────────────────────────────┘

데이터 연결 (명령마다 새로 열렸다 닫힘)
   LIST → 데이터 연결 #1 (목록)
   RETR → 데이터 연결 #2 (파일 다운로드)
   STOR → 데이터 연결 #3 (파일 업로드)
```

즉, **제어 연결은 긴 생명주기**, 데이터 연결은 **짧은 생명주기**를 갖는다.

### 3) 실제 텍스트 세션 예제

리눅스/유닉스에서 예전 방식대로 `telnet` 으로 FTP 서버에 직접 붙어보면 이 구조가 그대로 드러난다
(요즘은 보안상 실제로 이렇게 쓰면 안 되고, 학습용으로만 사용).

```text
$ telnet ftp.example.org 21
Trying 203.0.113.10...
Connected to ftp.example.org.
220 ftp.example.org FTP server ready.      ← 서버 인사(제어 연결)

USER alice
331 Password required for alice.
PASS secret123
230 User alice logged in.

PWD
257 "/" is the current directory.

TYPE I                                 ← 바이너리 전송
200 Type set to I.

PASV                                   ← 수동 모드 데이터 포트 요청
227 Entering Passive Mode (203,0,113,10,195,19).
                                       ← IP: 203.0.113.10, 포트: 195*256 + 19 = 49939

LIST                                   ← 디렉터리 목록 요청
150 Opening BINARY mode data connection for file list.
(여기서 별도의 TCP 데이터 연결이 열리고, 목록이 오간다)
226 Transfer complete.

QUIT
221 Goodbye.                            ← 제어 연결 종료
Connection closed by foreign host.
```

위에서 보는 것처럼:

- 포트 21으로 열린 연결은 끝까지 **명령/응답**만 주고받는다.
- `LIST` 명령을 내리면, **별도의 데이터 연결**이 열렸다가, 전송이 끝나면 `226` 으로 종료.

이 구조가 바로 “Two Connections”의 핵심이다.

---

## Control Connection — 명령과 응답이 오가는 “대화 채널”

### 1) 특성 요약

**제어 연결(control connection)** 은 다음과 같은 특징을 가진다.

- **TCP 기반**, 서버의 **포트 21**(기본값)에 클라이언트가 접속
- 세션 전체 동안 유지 (로그인부터 QUIT까지)
- 명령과 응답은 **ASCII 텍스트 줄 단위**로 오간다.
- 파일/데이터는 이 연결로 전송되지 않는다.

### 2) 명령/응답 구조

각 FTP 명령은 한 줄의 텍스트로 전송된다:

```text
<COMMAND> [인자...]<CRLF>
```

예:

```text
USER alice
PASS secret123
CWD /pub
TYPE I
RETR file.iso
```

서버 응답은 **3자리 숫자 코드 + 메시지** 로 이루어진다:

```text
220 Service ready for new user.
331 User name okay, need password.
230 User logged in, proceed.
550 File not found.
```

대표적인 코드:

| 코드 대역 | 의미 |
|----------|------|
| 1xx | Positive Preliminary (곧 이어질 응답/작업 시작) |
| 2xx | Positive Completion (성공) |
| 3xx | Positive Intermediate (추가 정보/작업 필요) |
| 4xx | Transient Negative Completion (일시적 실패) |
| 5xx | Permanent Negative Completion (영구적 실패) |

### 3) 상태 유지(Stateful)한 프로토콜

HTTP와 달리 FTP는 제어 연결이 **강하게 상태를 유지**한다:

- 현재 작업 디렉터리 (CWD)
- 현재 전송 타입 (ASCII / Binary)
- 현재 사용자 인증 상태
- 일부 구현에서는 세션 옵션들 (예: 전송 모드, 보호 수준 등)

이 때문에, 아래와 같은 시퀀스가 가능하다:

```text
USER alice      (로그인)
CWD /pub        (디렉터리 이동)
TYPE I          (바이너리 전송 지정)
RETR video.mp4  (파일 다운로드)
RETR image.png  (또 다른 파일 다운로드)
QUIT
```

중간의 설정(CWD, TYPE)이 **세션 전체에 유지**된다.

### 4) 예제: 간단한 Python으로 제어 연결만 훔쳐보기

실제 파일 전송까지 하지 않고, **제어 연결의 명령/응답만** 확인해볼 수 있다.

```python
import socket

host = "ftp.debian.org"   # 예시용 공개 FTP 미러 (실제 운영 상태는 수시로 바뀜)
port = 21

with socket.create_connection((host, port), timeout=5) as sock:
    # 서버 인사 메시지 읽기
    print(sock.recv(4096).decode("utf-8", errors="ignore"))

    # USER 명령 보내기
    sock.sendall(b"USER anonymous\r\n")
    print(sock.recv(4096).decode("utf-8", errors="ignore"))

    # PASS 명령 보내기
    sock.sendall(b"PASS guest@example.com\r\n")
    print(sock.recv(4096).decode("utf-8", errors="ignore"))

    # QUIT
    sock.sendall(b"QUIT\r\n")
    print(sock.recv(4096).decode("utf-8", errors="ignore"))
```

- 이 코드는 **제어 연결**만 사용하여 FTP 서버와 “대화”한다.
- `RETR`, `LIST` 등을 보내면 데이터 연결까지 뜨지만, 여기서는 **제어 채널의 동작 구조**를 보는 것이 목적이다.

---

## Data Connection — 실제 파일/목록이 흐르는 채널 (Active/Passive)

### 1) 개요: 왜 별도 데이터 연결인가?

제어 연결은 “대화용”이므로 오버헤드가 작고, 상태를 유지하기 좋다. 반면

- 파일 하나가 수 GB에 달할 수 있고
- 디렉터리 목록도 수만 줄이 될 수 있으며
- 전송 중에 **STOP** 이나 **ABORT** 같은 제어 명령을 보내고 싶다

이런 요구 때문에 **데이터는 별도의 TCP 연결에서** 전송한다.

FTP는 여기서 두 가지 모드를 제공한다:

- **Active 모드**
- **Passive 모드**

### 2) Active 모드 (PORT 명령)

Active 모드는 **고전적인 기본 모드**다.

1. 클라이언트는 제어 연결에서 `PORT` 명령으로
   “내가 데이터 받거나 보낼 포트는 XXX야” 라고 서버에게 알린다.
2. 서버는 **자신의 포트 20**(기본값)에서, 클라이언트가 지정한 IP:포트로
   TCP 연결을 거꾸로 건다.
3. 그 연결 위에서 파일/목록 데이터가 전송된다.

텍스트 예:

```text
Client: PORT 192,0,2,5,195,80
Server: 200 PORT command successful.

Client: RETR bigfile.iso
Server: 150 Opening BINARY mode data connection.
(서버가 192.0.2.5:50000 으로 TCP 연결을 생성하고, bigfile.iso 전송)
Server: 226 Transfer complete.
```

여기서 `195,80` 은 포트 번호를 나타내며,
$$\text{포트} = 195 \times 256 + 80 = 50000$$
과 같이 계산된다.

**문제점**:

- 서버→클라이언트 방향으로 새 연결을 거는 구조라,
  **NAT/방화벽 환경**에서 잘 막힌다.
- 클라이언트 측에서 임의 포트 하나를 열고 외부에서의 inbound를 허용해야 한다.

이 때문에, 실제 인터넷 환경에서는 **Passive 모드**가 훨씬 더 많이 쓰인다.

### 3) Passive 모드 (PASV/EXTENDED PASV)

Passive 모드는 제어/데이터 연결 방향을 반대로 돌린다:

1. 클라이언트가 `PASV` 명령을 보낸다.
2. 서버가 “내가 열어놓은 데이터 포트는 XXX야” 라고 응답한다.
3. 클라이언트가 **그 포트로 서버에 연결을 건다.**

예:

```text
Client: PASV
Server: 227 Entering Passive Mode (203,0,113,10,195,19).

Client: RETR report.pdf
Server: 150 Opening BINARY mode data connection for report.pdf.
(클라이언트가 203.0.113.10:49939로 TCP 연결을 만들고, report.pdf 수신)
Server: 226 Transfer complete.
```

- 서버 IP: `203.0.113.10`
- 데이터 포트: $$195 \times 256 + 19 = 49939$$

Passive 모드의 장점:

- **클라이언트가 항상 outbound 연결만 만든다.**
- 대부분의 NAT/방화벽이 이 방향의 연결은 허용한다.

오늘날 브라우저 내장 FTP 클라이언트나 GUI FTP 클라이언트(예: FileZilla 등)는
기본 설정을 Passive 모드로 두는 것이 일반적이다.

### 4) 데이터 연결 위에서는 무엇이 오가는가?

데이터 연결에서는 다음과 같은 “페이로드”가 오갈 수 있다:

- `LIST`, `NLST`: 디렉터리 목록 (텍스트)
- `RETR`: 파일 다운로드 (텍스트/바이너리)
- `STOR`, `APPE`: 파일 업로드
- 일부 확장 명령: `MLSD`, `MLST` 등

RFC 959는 데이터 표현 모드로 **ASCII, EBCDIC, Image(=binary)** 등을 정의한다.
실무에서는 거의 항상 **바이너리(Image) 모드**가 사용된다.

### 5) 예제: Python `ftplib` 으로 데이터 연결 사용

```python
from ftplib import FTP

ftp = FTP("ftp.example.org")
ftp.login("anonymous", "guest@example.org")

# 현재 디렉터리 목록 (LIST -> 데이터 연결 사용)

print("Directory listing:")
ftp.retrlines("LIST")

# 파일 다운로드 (RETR -> 데이터 연결 사용)

with open("README.txt", "wb") as f:
    ftp.retrbinary("RETR README.txt", f.write)

ftp.quit()
```

내부적으로는:

- `FTP(...)` → 포트 21 제어 연결
- `login()` → `USER` / `PASS` 명령 교환
- `retrlines("LIST")` → `PASV` + `LIST` → 데이터 연결 생성 → 목록 수신
- `retrbinary("RETR ...")` → 또 다른 데이터 연결에서 파일 수신

이 모든 것이 RFC 959에서 정의한 **two-connection 구조**를 그대로 따른다.

---

## Security for FTP — 왜 평문 FTP는 “더 이상 쓰면 안 되는가?”

이제 가장 중요한 현실적인 부분, **보안(Security)** 을 보자.

### 1) 전통적인 FTP의 보안 문제

전통적인 FTP(RFC 959 기준)는 **암호화를 전혀 사용하지 않는다**.

따라서:

- **계정 정보( USER / PASS )가 평문으로 전송**된다.
- 업로드/다운로드 하는 파일 내용도 모두 평문이다.
- 공격자가 네트워크 스니퍼만 설치해도:
  - 사용자 ID/비밀번호 탈취
  - 민감한 파일 내용 탈취
  - FTP 명령/응답 전체 엿보기

실제로 미국 CISA는 2024년 지침에서
“Telnet, FTP, TFTP 같은 **평문 서비스는 불필요하면 비활성화**하라”고 명시하고 있다.

또한 최근까지도 FTP 연동 장비에서 **평문 전송 취약점**이 반복적으로 보고되고 있다.

- FTP 자체가 잘못된 것이 아니라,
- **암호화·인증을 제공하지 않는 설계**이기 때문에
  오늘날 인터넷 환경에서는 **보안 수준이 사실상 0에 가깝다**는 평가를 받는다.

### 2) FTP Security Extensions — RFC 2228

이 문제를 보완하기 위해 IETF는 **FTP Security Extensions (RFC 2228)** 을 정의했다.

RFC 2228은 다음과 같은 새로운 명령/응답을 추가하여:

- **강한 인증(Strong authentication)**
- **무결성(Integrity)**
- **기밀성(Confidentiality)**

을 **제어 채널과 데이터 채널 모두에 적용**할 수 있도록 했다.

주요 명령:

- `AUTH` : TLS, SSL 등 보안 메커니즘 사용 요청
- `PBSZ`: 보호 버퍼 크기 협상
- `PROT`: 보호 수준(데이터 채널을 암호화할지 여부 등)
- `CCC` : 암호화된 제어 채널을 다시 평문으로 되돌리는 옵션
- `MIC`, `CONF`, `ENC` : 메시지 무결성·기밀성 보장 관련

### 3) FTPS — “FTP over TLS”

RFC 2228의 확장과 TLS(RFC 2246 등)를 조합한 것이 **FTPS** 이다.
이를 정식으로 규정한 문서가 RFC 4217 “Securing FTP with TLS”이다.

FTPS의 핵심 아이디어:

1. **제어 연결**(포트 21 또는 990)을 TLS로 감싼다.
2. **데이터 연결**도 TLS로 감싼다(선택 사항, PROT 명령으로 제어).

전형적인 Explicit FTPS(포트 21) 시퀀스:

```text
Client:  AUTH TLS
Server:  234 Enabling TLS Connection.
(여기서 TLS 핸드셰이크 진행 → 제어 채널 암호화)

Client:  USER alice
Server:  331 Password required.
Client:  PASS secret123
Server:  230 User logged in.

Client:  PBSZ 0
Server:  200 PBSZ=0

Client:  PROT P      ← 데이터 채널도 암호화(Private)하겠다는 의미
Server:  200 Protection level set to P.

Client:  PASV
Server:  227 Entering Passive Mode (...)
(데이터 채널도 TLS로 암호화된 상태에서 파일 전송)
```

- `PROT P`: 데이터 채널을 **암호화 + 무결성 보호**
- `PROT C`: 데이터 채널은 평문(clear), 제어 채널만 암호화
- 전통적인 FTP에 비해, 계정 정보/파일 모두를 보안 터널 안에서 전송할 수 있다.

**주의**: FTPS에서 사용하는 TLS 버전도 최신 표준을 따라야 한다.
IETF는 TLS 1.0/1.1을 2021년 RFC 8996으로 정식 폐기(deprecate)했다.
따라서 오늘날 FTPS 서버는 TLS 1.2 이상(가능하면 1.3)을 사용해야 한다.

### 4) SFTP vs FTPS vs FTP — 오늘날의 권장 사항

실무에서는 “FTP 보안”을 얘기할 때 **두 가지 다른 프로토콜**이 혼용되곤 한다:

1. **FTPS (FTP over TLS)**
   - 현재 이야기하고 있는 FTP 확장.
   - 기본 FTP(포트 21)에 TLS를 입힌 것.
   - 제어/데이터 채널 모두 TLS로 보호 가능.

2. **SFTP (SSH File Transfer Protocol)**
   - 이름은 “FTP”가 들어가지만, **SSH(포트 22) 위에서 돌아가는 완전히 다른 프로토콜**.
   - SSH의 암호화·인증·무결성을 그대로 활용.
   - 단일 포트(22)만 사용해서, 방화벽·NAT 환경에서도 다루기 쉽다.
   - 2020년대에는 기업용 파일 전송의 사실상 **표준**으로 간주되는 경우가 많다.

미국과 유럽의 보안 가이드라인, 업계 모범 사례를 보면 공통적으로:

- 민감한 데이터 전송에 **평문 FTP는 사용하지 말 것**
- 가능하면 **SFTP** 또는 **FTPS** 같은 **암호화된 프로토콜**로 마이그레이션할 것

을 권고한다.

### 5) 예제: Python `ftplib` + TLS (FTPS)

Python의 `ftplib` 는 `FTP_TLS` 클래스를 통해 FTPS를 지원한다.

```python
from ftplib import FTP_TLS

ftps = FTP_TLS("ftps.example.org")
ftps.login("alice", "secret123")   # 제어 채널 인증 (이미 TLS 위일 수도 있다)

# 데이터 채널도 암호화하도록 명령

ftps.prot_p()

# 암호화된 채널로 파일 다운로드

with open("secret.pdf", "wb") as f:
    ftps.retrbinary("RETR secret.pdf", f.write)

ftps.quit()
```

내부적으로는:

- `AUTH TLS` → TLS 핸드셰이크로 제어 채널 암호화
- `PBSZ 0`, `PROT P` → 데이터 채널까지 암호화
- 이후 모든 USER/PASS/RETR/STOR 등 명령/데이터가 TLS로 보호된다.

### 6) 보안 설계 체크리스트 (2025년 기준)

FTP/FTPS 관련 시스템을 설계하거나 리뷰할 때, 최소한 아래 항목은 점검해야 한다:

1. **평문 FTP(21, 암호화 없음)를 외부에 노출하지 않는가?**
2. FTPS 사용 시:
   - TLS 1.2 또는 1.3만 허용하는가? (TLS 1.0/1.1 금지)
   - 강력한 암호 스위트만 허용하는가?
   - 서버/클라이언트 인증서 관리가 적절한가?
3. **데이터 채널까지 PROT P로 보호**하고 있는가?
   (제어 채널만 암호화하고 데이터 채널을 평문으로 두는 실수가 없는지)
4. 계정 관리:
   - 익명 FTP 익스포트(anonymous FTP)가 정말 필요한가?
   - 최소 권한 원칙(읽기 전용 vs 읽기/쓰기)을 지키는가?
5. 로그·감사:
   - FTP/FTPS 성공/실패 시도, 업로드/다운로드 이력을 적절히 기록하는가?
   - 사람/시스템 계정이 구분되어 있는가?

---

## 정리

이 절에서 본 FTP 핵심 포인트를 정리하면:

1. FTP는 **제어 연결 + 데이터 연결**이라는 독특한 구조를 가진다.
   - 제어 연결: 포트 21, 세션 전체 동안 명령/응답만
   - 데이터 연결: 파일/목록이 오가는 별도 TCP 연결 (명령마다 열렸다 닫힘)

2. 제어 연결은 상태를 유지하며, 명령( USER, PASS, RETR, STOR, PASV, PORT …)과
   3자리 상태 코드(220, 230, 550 등)를 주고받는다.

3. 데이터 연결은 Active(서버가 클라이언트에 연결)와 Passive(클라이언트가 서버에 연결) 모드가 있으며,
   실제 파일/디렉터리 목록 데이터가 이 위에서 전송된다.

4. 전통적인 FTP는 **암호화·무결성 보호가 전혀 없기 때문에**,
   오늘날 인터넷에서는 **민감한 데이터에는 절대 사용하면 안 되는 프로토콜**이다.

5. FTP의 보안을 위해:
   - RFC 2228(FTP Security Extensions), RFC 4217(FTP over TLS)에 따라 **FTPS** 를 사용하거나,
   - 아예 SSH 기반의 **SFTP** 로 전환하는 것이 2020년대 기준 모범 사례이다.
