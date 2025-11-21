---
layout: post
title: 데이터 통신 - Standard Client-Server Protocols (4)
date: 2024-09-09 21:20:23 +0900
category: DataCommunication
---
# 원격 로그인과 보안의 진화

## Telnet — Local vs Remote Login

### Telnet의 목적과 역사

**Telnet**은 ARPANET 시절부터 존재했던 **원격 터미널 접속 프로토콜**이다.
RFC 854(1983)에서 정의되며, “터미널 지향 프로세스와 터미널 장치를 표준 방식으로 연결하기 위한 양방향 8비트 바이트 스트림”을 제공하는 것이 목적이라고 설명한다.

핵심 아이디어:

- 사용자는 자신의 단말(키보드+화면)을 통해
- **멀리 떨어진 호스트에 로그인(login)** 하고
- 마치 로컬 터미널처럼 쉘을 사용할 수 있게 하는 것

과거에는 유닉스 서버, 메인프레임 등에 로그인할 때 Telnet이 사실상 표준이었다. 오늘날에는 **암호화가 되지 않아 보안상 심각한 문제**가 있기 때문에
미국 CISA, NSA, 캐나다 사이버 보안 센터 등에서 **Telnet 사용 금지 또는 비활성화, SSH 전환**을 강력히 권고한다.

---

### Telnet 프로토콜 개요

Telnet은 응용 계층 프로토콜로, **TCP 23번 포트**를 기본으로 사용한다.

- **전송 계층**: TCP
- **기본 포트**: 23/TCP
- **데이터 단위**: 8비트 바이트 스트림
- **추상 터미널 모델**: NVT(Network Virtual Terminal)

#### NVT(Network Virtual Terminal)

RFC 854에서 Telnet은 **NVT**라는 추상적인 텍스트 터미널 모델을 정의한다.

- 클라이언트와 서버는 모두 NVT 문자 집합(ASCII 기반)을 이해한다고 가정
- 실제 클라이언트 터미널이 VT100이든 콘솔이든, Telnet은 이를 NVT로 변환해서 보낸다
- 서버는 NVT를 기반으로 애플리케이션(쉘, 텍스트 UI 등)을 제공

#### 명령 네고시에이션

Telnet은 **데이터와 제어 명령이 같은 스트림**으로 흘러간다.

- 특수 바이트 `IAC (Interpret As Command) = 255` 등장 시 뒤에 오는 바이트들을 **Telnet 명령**으로 해석
- 예: `IAC DO <option>`, `IAC WILL <option>` 형식으로 옵션 협상

실제 옵션들(에코 방식, 터미널 타입 등)은 대부분 역사적 중요성이 크지만,
실무에서는 Telnet을 더 이상 쓰지 않기에 깊이 파고들 필요는 적다.

---

### Local Login vs Remote Login (Telnet 관점)

교과서에서 “local versus remote login”을 Telnet 문맥에서 이야기할 때, 핵심은 다음이다.

#### 1) Local Login

**로컬 로그인**은 말 그대로 **사용자와 OS가 같은 머신**에 있는 경우다.

- 예: 데스크톱에서 직접 리눅스 콘솔에 로그인 (`tty1`, `tty2`…)
- Windows에서 직접 로그인(로컬 콘솔)
- 로그인 경로:
  - 키보드 → OS의 TTY/콘솔 드라이버 → 로그인 프로그램(`login`, `getty`) → 사용자 쉘

데이터 흐름(아주 단순화):

```text
[키보드] --(OS 내부)--> [로컬 로그인 프로세스] --> [쉘/bash]
[쉘 출력] --(OS 내부)--> [로컬 디스플레이]
```

네트워크는 전혀 개입하지 않는다.

#### 2) Remote Login (Telnet)

**원격 로그인**은 사용자가 **네트워크를 통해 다른 호스트**에 로그인하는 것:

- 사용자는 Telnet 클라이언트를 실행
- 로컬에서는 “터미널 애플리케이션”이지만, 실제 명령은 **원격 호스트의 쉘**에 전달
- 로그인 경로:
  - 키보드 → Telnet 클라이언트 → TCP/IP → 네트워크 → Telnet 서버 → 로그인 프로그램 → 쉘

데이터 흐름:

```text
[키보드]
   |
   v
[Telnet Client] --TCP/IP--> [Telnet Server] --> [쉘/bash on remote host]
   ^                                               |
   |                                               v
[화면] <-- TCP/IP <------------------------ [쉘 출력]
```

이때, **로컬 로그인과의 핵심 차이**는:

1. **입출력이 네트워크를 통해 전달**된다는 점
2. 입력/출력 모두 **평문(plaintext)** 으로 흐르기 때문에,
   중간의 공격자가 패킷을 캡처하면 **ID/비밀번호, 명령, 결과를 전부 볼 수 있다**는 점

#### 3) 예제: 로컬 vs Telnet 원격 로그인 시나리오 비교

예를 들어, 사무실에서 리눅스 서버에 접속하는 경우를 보자.

- **로컬 로그인**:
  - 서버에 모니터와 키보드를 직접 연결
  - 부팅 후 `login:` 프롬프트에서 계정 입력
  - 물리적으로 서버 앞에 있어야 한다.

- **Telnet 원격 로그인**:
  - 사용자는 자신의 노트북에서 다음과 같이 입력

    ```bash
    telnet server.example.com 23
    ```

  - 네트워크를 통해 서버의 Telnet 데몬(`telnetd`)에 접속
  - 서버는 `login:` 프롬프트를 Telnet 클라이언트에 전송
  - 사용자는 아이디, 비밀번호 입력 → 그대로 네트워크(평문)로 전달

이때, 중간에 스니핑 장비가 있으면 다음과 같이 보인다(요약):

```text
C: l o g i n :   a l i c e
S: P a s s w o r d :
C: s e c r e t 1 2 3
...
```

암호화가 없기 때문에 **와이어샤크(Wireshark) 같은 도구로 쉽게 ID/비밀번호 탈취**가 가능하다.
이 점 때문에 최신 보안 가이드에서는 **Telnet 사용을 취약점으로 간주**하고 있다.

---

### Telnet 세션 실습 예 (교육용)

※ 실제 인터넷에서는 보안상 사용하지 않고, **실습용으로만** 로컬 네트워크에서 사용해야 한다.

#### 1) 간단한 에코 서버에 Telnet으로 접속

리눅스에서 테스트용 TCP 에코 서버를 열고 Telnet 클라이언트로 접속해볼 수 있다.
(예: `nc`를 이용한 간단한 서버)

```bash
# 서버 쪽 (포트 9000에 에코 서버)

nc -l -p 9000

# 클라이언트 쪽

telnet 192.0.2.10 9000
Trying 192.0.2.10...
Connected to 192.0.2.10.
Escape character is '^]'.
Hello over Telnet
Hello over Telnet
```

- 클라이언트에서 입력한 문자열이 그대로 에코 서버에 도달하고, 다시 돌아온다.
- 이 과정에서 **암호화는 전혀 없다**.

#### 2) SMTP 서버에 Telnet으로 접속 (헤더만 관찰)

과거에는 SMTP 서버 디버깅을 위해 Telnet으로 25번 포트에 접속해 직접 명령을 보내기도 했다.

```bash
telnet mail.example.com 25
Trying 203.0.113.5...
Connected to mail.example.com.
220 mail.example.com ESMTP Postfix

HELO client.example.net
250 mail.example.com
QUIT
221 2.0.0 Bye
Connection closed by foreign host.
```

- Telnet은 결국 **“임의의 TCP 서비스에 접속해서 바이트를 던지고 받는 일반적인 터미널 도구”**처럼 사용할 수 있다.
- 그러나 보안상 지금은 대부분 **`openssl s_client`, `nc`, `swaks` 등**을 사용하며, Telnet은 지양한다.

---

### Telnet의 보안 문제와 현대 가이드라인

현대 보안 가이드는 공통적으로 다음을 권고한다:

1. **Telnet 서비스 비활성화**
   - CISA의 BOD 23-02 및 네트워크 인프라 보안 가이드에서,
     인터넷에 노출된 관리 인터페이스에서 Telnet을 제거/비활성화 하고 SSH 등을 사용할 것을 요구한다.
2. **관리용으로 SSH 사용**
   - Cisco, Juniper 등 주요 벤더 구성 예제에서도 Telnet 대신 SSH만 허용하는 설정을 제공한다.
3. **레거시 시스템 처리**
   - 만약 어쩔 수 없이 Telnet을 사용해야 하는 장비가 있다면,
     - 관리 네트워크를 분리(VPN 내부에서만 허용)
     - 방화벽 ACL로 접근 제어
     - 가능한 빨리 SSH 지원 장비로 교체

즉, Telnet은 **네트워크 프로토콜 역사 공부**와 **보안 교육**에는 유용하지만,
현업에서는 “사용하면 안 되는 프로토콜”로 보는 것이 맞다.

---

### Local vs Remote Login, Telnet vs SSH 요약 표

| 구분 | Local Login | Telnet 원격 로그인 | SSH 원격 로그인 |
|------|------------|--------------------|-----------------|
| 위치 | 사용자와 OS가 같은 머신 | 네트워크를 통해 원격 호스트 접속 | 네트워크를 통해 원격 호스트 접속 |
| 프로토콜 | 없음 (OS 내부 콘솔) | Telnet (TCP 23) | SSH (TCP 22) |
| 암호화 | OS 내부, 네트워크 없음 | 없음 (평문) | 강한 암호화 (대칭+MAC) |
| 서버 인증 | 물리적 접근 | 사실상 없음 | 필수(호스트 키) |
| 사용자 인증 | 패스워드, 로컬 정책 | ID/비밀번호 평문 전송 | ID/비밀번호 또는 공개키, 기타 |
| 사용 권고 | 정상 | 사용 금지 (보안 취약) | 적극 권장 |

이제 Telnet의 한계를 보완하기 위해 등장한 **SSH**를 보자.

---

## SSH — Secure Shell

### SSH 개요

**SSH(Secure Shell)** 는 “보안이 없는 원격 로그인 프로토콜(Telnet, rlogin, rsh 등)”을 대체하기 위해 설계된 **보안 원격 로그인 및 보안 네트워크 서비스 프로토콜**이다.

- **설계 목표**:
  - 암호화된 채널(기밀성)
  - 무결성 보호
  - 서버(및 필요시 클라이언트) 인증
  - 여러 논리 채널(Session, TCP 포워딩 등)을 하나의 암호화된 연결 안에서 다중화
- **기본 포트**: 22/TCP

RFC 4251에 따르면, SSH는 **3개의 주요 컴포넌트**로 구성된다.

1. Transport Layer Protocol
2. User Authentication Protocol
3. Connection Protocol

이 세 부분을 이해하면 SSH를 깊이 있게 파악할 수 있다.

---

### SSH Components — 세 가지 계층 구조

#### 1) SSH Transport Layer Protocol

**전송 계층 프로토콜**은 다음을 책임진다.

- 서버 호스트 인증(서버가 진짜인지 확인)
- 키 교환(Key Exchange) 및 세션 키 생성
- 대칭키 암호화(AES 등)를 통한 기밀성 제공
- MAC(Message Authentication Code) 혹은 AEAD로 무결성 보호
- 압축(Optional)

흐름(단순화):

1. 클라이언트→서버: 버전 문자열 교환 (`SSH-2.0-OpenSSH_9.0` 등)
2. 알고리즘 네고시에이션(지원하는 키 교환, 암호, MAC 목록 교환)
3. 키 교환(예: ECDH)
4. 서버 호스트 키 확인(known_hosts와 대조)
5. 세션 키 생성 → 이후의 모든 트래픽은 암호화됨

NSA와 유럽 보안 기관들은 SSH 사용할 때 **SSHv2만 허용**하고,
강력한 암호 스위트(AES-GCM, ChaCha20-Poly1305 등)를 사용할 것을 권고한다.

#### 2) SSH User Authentication Protocol

Transport Layer 위에서 동작하는 **사용자 인증 프로토콜**이다.

지원 방식:

- **패스워드 인증**: 암호는 암호화된 채널로 전달되지만,
  브루트포스·피싱 위험이 있으므로 보안 가이드에서 점차 비권장
- **공개키 인증(public key)**:
  - 사용자는 개인키를 로컬에 보관
  - 서버에는 공개키를 `~/.ssh/authorized_keys`에 등록
  - 로그인 시 클라이언트가 서명 기반 챌린지/응답으로 본인 증명
- **키보드-인터랙티브**:
  - 일회성 비밀번호, OTP 등과 연계 가능
- **호스트 기반 인증**:
  - 신뢰된 호스트 간에 노드 자체를 인증(일반 인터넷보다는 클러스터/내부망에 사용)

실무에서는 **공개키 인증 + MFA**(예: SSH 키 + FIDO2) 조합이 권장된다.

#### 3) SSH Connection Protocol

**Connection Protocol**은 암호화된 SSH 연결 안에 **다수의 논리 채널**을 다중화한다.

대표 채널 타입:

- **session** 채널:
  - 원격 쉘(`bash`, `zsh` 등)
  - 원격 명령 실행 (`ssh host "uptime"`)
  - X11 포워딩 등
- **direct-tcpip / forwarded-tcpip**:
  - 로컬 포트 포워딩
  - 원격 포트 포워딩
  - SOCKS 프록시(동적 포워딩)

예를 들어, 하나의 SSH 연결로:

- 터미널 세션
- `scp` 파일 전송
- 포트 포워딩

을 동시에 사용할 수 있다. 이 구조 덕분에 SSH는 단순한 “원격 로그인”을 넘어 **범용 보안 터널링 메커니즘**으로 쓰인다.

---

### SSH 기본 사용 예제

여기서는 **OpenSSH 클라이언트** 기준으로 볼 수 있는 실제 예제를 살펴본다.

#### 1) 기본 원격 로그인

```bash
ssh alice@server.example.com
```

- 처음 접속 시:

  ```text
  The authenticity of host 'server.example.com (203.0.113.10)' can't be established.
  ECDSA key fingerprint is SHA256:abc123....
  Are you sure you want to continue connecting (yes/no/[fingerprint])?
  ```

  - 여기서 **서버 호스트 키**를 확인하고 `yes` 입력 → `~/.ssh/known_hosts`에 저장

- 그다음부터는 호스트 키가 변경되지 않는 한 경고 없이 접속

#### 2) 공개키 생성 및 등록

```bash
# 키 생성 (Ed25519 예시)

ssh-keygen -t ed25519 -C "alice@example.com"
# 기본 위치: ~/.ssh/id_ed25519, ~/.ssh/id_ed25519.pub

# 공개키를 서버로 복사

ssh-copy-id alice@server.example.com
```

이제부터는:

```bash
ssh alice@server.example.com
```

입력 시 패스워드 대신 키 기반 인증으로 로그인할 수 있다.

#### 3) 포트 포워딩 예제

- **로컬 포트 포워딩**: 로컬 8080 → 원격 DB 5432

```bash
ssh -L 8080:127.0.0.1:5432 alice@db.example.com
```

- **원격 포트 포워딩**: 원격 9000 → 로컬 웹 서버 127.0.0.1:3000

```bash
ssh -R 9000:127.0.0.1:3000 alice@bastion.example.com
```

이런 포워딩 기능은 편리하지만, CISA 등은 *“무분별한 SSH 포워딩은 내부로의 백도어”*가 될 수 있다며 주의할 것을 경고한다.

---

### SSH Applications — 주요 활용 사례

SSH는 “보안 원격 로그인”을 넘어 다양한 용도로 쓰인다.

#### 1) 원격 서버 관리(Remote Login)

가장 대표적인 활용:

```bash
ssh admin@router.example.net
ssh dev@web01.internal
```

- 리눅스 서버, 네트워크 장비, 클라우드 VM 등에 접속해:
  - 패키지 설치/업데이트
  - 설정 파일 수정
  - 로그 확인
  - 서비스 재시작 등

NSA, CISA 등은 네트워크 인프라 장비 관리 시 Telnet, HTTP 대신 **SSH와 HTTPS만 허용**하도록 하라고 안내한다.

#### 2) 파일 전송 — SCP, SFTP

SSH 위에서 동작하는 대표적 파일 전송 도구:

- `scp`: 간단한 파일 복사
- `sftp`: 대화형 파일 전송, 디렉터리 탐색 등

예:

```bash
# scp: 로컬→원격

scp backup.tar.gz alice@server.example.com:/data/backup/

# scp: 원격→로컬

scp alice@server.example.com:/var/log/syslog ./syslog

# sftp: 대화형 모드

sftp alice@server.example.com
sftp> ls
sftp> put report.pdf
sftp> get /var/log/auth.log
sftp> exit
```

프랑스 ANSSI의 OpenSSH 보안 사용 권고에 따르면,
SCP/SFTP는 기존 FTP/FTPS보다 방화벽 설정, 정책 일관성 측면에서 장점이 있으며,
가능하면 SSH 기반 파일 전송을 표준으로 삼으라고 권고한다.

#### 3) 터널링과 프록시 — SSH Tunneling

SSH는 **TCP 터널링**을 제공하여:

- 내부 DB에 안전하게 접속
- 제한된 서비스(내부 대시보드 등)를 일시적으로 외부에서 접근
- SOCKS 프록시로 트래픽 중계

등에 활용할 수 있다. 하지만 잘못 사용하면 **조직 외부로 백도어를 여는 행위**가 될 수 있다는 점에서, CISO 레벨의 가이드에서 엄격한 정책·감사가 요구된다.

예: 로컬에서 내부 네트워크로 트래픽 터널링

```bash
# 회사 내부망에만 열려 있는 10.0.0.10:5601(Kibana)에
# 외부에서 접근하기 위해 bastion을 통해 터널 생성

ssh -L 5601:10.0.0.10:5601 bastion.corp.example
# 이후 브라우저에서 http://localhost:5601 로 접속

```

#### 4) 자동화·배포·CI/CD

SSH는 다음과 같은 자동화 시나리오에서도 핵심 역할:

- CI 서버에서 애플리케이션 빌드 후 서버로 배포
- Ansible, Fabric 같은 **구성 관리 도구**들이 SSH를 활용해 명령 실행
- Git 리포지토리 접근(예: `git@github.com:...`)

권장 보안 패턴:

- 비대화형 계정에 대해 **키 기반 인증 + 제한된 권한**(forced command, `Match` 블록 등)
- `~/.ssh/authorized_keys`에 `command="..."`, `from="..."` 옵션으로 권한 최소화

---

### SSH 보안 설정 모범 사례 (요약)

미국 NSA, CISA, 캐나다·EU 보안 기관의 권고를 종합하면, 실무에서 SSH를 사용할 때 다음을 지키는 것이 좋다.

1. **SSH 버전**
   - SSHv1 비활성화, **SSHv2만 허용**
2. **암호 스위트**
   - 강한 알고리즘(AES-GCM, ChaCha20-Poly1305, modern KEX)만 사용
   - 취약한 알고리즘(DES, RC4, old MAC 등) 비활성화
3. **인증**
   - 가능하면 **패스워드 로그인 비활성화**
   - 공개키 인증 + MFA(FIDO2, OTP 등)
4. **접근 통제**
   - `AllowUsers`, `AllowGroups`로 접근 가능한 계정 제한
   - 방화벽/보안 그룹으로 SSH 포트 접근 최소화
5. **포트 포워딩/터널링 정책**
   - 업무상 필요한 경우만 허용
   - 로깅·모니터링을 통해 비정상 터널 탐지
6. **호스트 키 관리**
   - 서버 재설치 시 호스트 키 변경 알림
   - `known_hosts`를 통한 MITM 감지
7. **로그·감사**
   - 실패/성공 로그인 시도, 비정상 활동 탐지
   - 중앙 로그(예: SIEM)와 연계

---

## 정리

- **Telnet**은 초기 인터넷에서 표준 원격 로그인 프로토콜이었으나,
  암호화·인증 부재로 인해 오늘날에는 **보안상 사용이 금지 수준**인 레거시 기술이다.
  - Local vs Remote login을 이해하면서 “네트워크를 누가 볼 수 있는지” 관점으로 보면 Telnet의 위험성이 직관적으로 보인다.
- **SSH**는 이러한 문제를 해결하기 위해 탄생한 **보안 원격 접속 및 터널링 플랫폼**이다.
  - RFC 4251에서 정의한 **Transport / User Authentication / Connection** 세 계층을 이해하면,
    SSH가 왜 단순 로그인 도구를 넘어 **범용 보안 채널**로 쓰이는지 자연스럽게 보인다.
  - 현대 보안 가이드는 “Telnet 비활성화, SSHv2만 허용, 강한 암호·키·정책 적용”을 공통적으로 권고한다.
