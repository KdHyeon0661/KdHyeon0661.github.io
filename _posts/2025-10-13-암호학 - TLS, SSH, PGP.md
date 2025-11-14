---
layout: post
title: 암호학 - TLS/SSH/PGP
date: 2025-10-13 18:30:23 +0900
category: 암호학
---
# 프로토콜: TLS/SSH/PGP 등 (⚙️)

> 이 장은 **네트워크·전자우편 보안을 실무 관점**에서 압축 정리합니다.
> **TLS 1.3 핸드셰이크(키 스케줄/0-RTT 위험) → SSH(키 교환·호스트키·포워딩) → 전자우편 보안(S/MIME·OpenPGP)** 순서로, 그림/명세/명령 예제/미스유스를 함께 담았습니다.
> 실무 포인트: **표준 구현 사용 + 안전한 기본값 + 로그/키 수명/자동화**.

---

## ✅ 10.1 TLS 1.3 — 핸드셰이크 흐름 & 키 스케줄

### 한눈에 보는 변화(1.2 → 1.3)

- **왕복(RTT) 감소**: 풀 핸드셰이크 1-RTT, 세션 재개 0-RTT(옵션).
- **암호 스위트 단순화**: `TLS_AES_128_GCM_SHA256`, `TLS_AES_256_GCM_SHA384`, `TLS_CHACHA20_POLY1305_SHA256` 등 **AEAD + HKDF** 고정.
- **서버 인증 의무화**, **완전 전진 기밀성(FS)**: (EC)DHE 필수.
- 취약한 구성(예: RSA 키교환, CBC, RC4, static RSA, SHA-1) 제거.

### 풀 핸드셰이크 흐름(개념 그림)

```
Client                         Server
------                         ------
ClientHello
  - KeyShare[g^x]  ---------->      (선택된 그룹 확인)
                           ServerHello
                           - KeyShare[g^y]
                           {EncryptedExtensions}
                           {Certificate}
                           {CertificateVerify}
                           {Finished}  <----  핸드셰이크 키로 암호화
{Finished} --------------->

[Application Data <----> Application Data]   (앱키로 암호화)
```

- 중괄호 `{...}`: **암호화됨(Handshake traffic keys)**.
- 최종 앱 데이터는 **Application traffic keys**로 보호.

### 키 스케줄(요약 직관)

핵심은 **HKDF**. 입력 비밀들을 단계적으로 결합하고, 라벨(label)로 도메인 분리.

- **Early Secret** = HKDF-Extract(0, PSK)
- **Handshake Secret** = HKDF-Extract(DH(g^xy), Derive(Early Secret))
- **Master Secret** = HKDF-Extract(0, Derive(Handshake Secret))

여기서 `Derive(X)`는 라벨링된 HKDF-Expand. 이 비밀들로부터
- **Client/Server Handshake Traffic Secret**
- **Client/Server Application Traffic Secret** (및 업데이트 키)
를 라벨별로 확장해 사용.

**포인트**
- (EC)DHE로 **전진기밀성**.
- PSK(재개) 사용 시도라도 **(EC)DHE + PSK**(하이브리드)가 권장.

### 0-RTT(Zero-RTT) 데이터 — 속도 vs 위험

**장점**: 세션 재개 시 **클라이언트가 0-RTT로 앱 데이터 전송** 가능(왕복 절감).
**위험(핵심 두 가지)**:
1) **리플레이 가능(replayable)**: 엄밀한 의미의 FS가 약화. 중간자가 0-RTT 레코드를 **다시 재전송** 가능.
2) **합의가 바뀌기 전 데이터**: 서버 정책(알고리즘/ALPN/서버 설정)과 **미스매치** 가능.

**대응 가이드**
- **멱등 요청만 0-RTT 허용**(GET/조회성, 부수효과 없는 것).
- 서버는 **Anti-Replay** 창(트래킹)·**짧은 수용 시간**·**토큰 바인딩** 사용.
- 중요한 변경/결제/쓰기 요청은 **1-RTT 이후**에만 허용.

### 실무 설정·점검(예시)

**OpenSSL s_client 디버깅**
```bash
# 서버와 TLS 1.3로 통신, 핸드셰이크·알고리즘 확인

openssl s_client -connect example.com:443 -tls1_3 -msg -state
```

**cURL 강제 스위트(가능한 경우)**
```bash
curl --tlsv1.3 --ciphers TLS_AES_256_GCM_SHA384 https://example.com/
```

**Nginx (개념 스니펫)**
```nginx
ssl_protocols TLSv1.3;
ssl_ciphers TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256:TLS_AES_128_GCM_SHA256;
# 0-RTT(early data) 비활성화 권장 (필요시 앱 레벨 멱등성 검사)

ssl_early_data off;
```

**실수 체크**
- [ ] TLS 1.3 활성화, 1.0/1.1 비활성.
- [ ] AEAD만(기본), ECDHE만(키 교환).
- [ ] 0-RTT는 기본 **OFF**, 필요 시 **멱등만 허용**.
- [ ] HSTS/OCSP 스테이플링/알고리즘 관측 로깅.

---

## ✅ 10.2 SSH — 키 교환·호스트키·포워딩

### SSH의 빌딩 블록

- **KEX(Key Exchange)**: (EC)DH 또는 PQC 하이브리드 변형 등으로 **세션키 합의**.
- **Host Key**: 서버 신원 검증(Ed25519/RSA/ECDSA…).
- **User Auth**: 공개키/패스워드/키보드-인터랙티브/SSO.
- **채널/포워딩**: 포트/에이전트/X11/프록시.

### 핸드셰이크 흐름(요약)

1) 알고리즘 협상: `kex_algorithms`, `server_host_key_algorithms`, `ciphers`, `MACs`
2) **KEX**: (EC)DH로 공유비밀 \(K\)
3) **서버 HostKey** 로 `H`(트랜스크립트 해시)에 **서명** → **서버 진위** 검증
4) **세션키 파생**: `HKDF/Derive`류로 암복호키/IV 생성
5) 사용자 인증 시작

**실무 스위트 감각**
- **키 교환**: `curve25519-sha256`(X25519), `ecdh-sha2-nistp256` 등
- **호스트키**: **ed25519** 선호(작고 빠름), RSA도 3072+
- **암호**: `chacha20-poly1305@openssh.com` 또는 `aes256-gcm@openssh.com`

### 호스트키 검증 & known_hosts

- 첫 접속 시 서버 키 지문 확인(**TOFU** 모델) 또는 **사전 배포**.
- `~/.ssh/known_hosts` 에 서버의 호스트키(또는 `@cert-authority`) 기록.
- **호스트키 회전** 시 `UpdateHostKeys` 사용(서버가 새 키를 알려주고 클라이언트가 업데이트).

**예시: 첫 연결 시 지문 확인**
```bash
ssh -o StrictHostKeyChecking=ask user@host
# 또는 서버 측: ssh-keygen -l -f /etc/ssh/ssh_host_ed25519_key.pub

```

**호스트 인증서(OpenSSH CA)**
```bash
# CA로 서버 호스트키 서명 → 클라이언트는 CA만 신뢰

@cert-authority *.corp.example ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI...  # known_hosts
```

### 사용자 인증 & 에이전트 포워딩

- **공개키 인증**(권장): `~/.ssh/id_ed25519` + 서버의 `authorized_keys`.
- **에이전트 포워딩**: `ssh -A host` → 원격에서 **로컬 에이전트**로 서명 위임.
  - **주의**: 신뢰할 수 없는 중간 호스트에 에이전트 포워딩 **금지**(키 탈취 위험). 필요 시 **`ProxyJump`**/단일 홉 원칙.

**SSH Config 스니펫**
```sshconfig
Host bastion
  HostName bastion.corp
  User ops
  IdentityFile ~/.ssh/id_ed25519
  PubkeyAuthentication yes
  IdentitiesOnly yes
  ForwardAgent no

Host app-*
  ProxyJump bastion
  User deploy
  IdentityFile ~/.ssh/id_ed25519
  ForwardAgent no
```

### 포트 포워딩 & 터널

- **로컬 포워딩**: `-L [bind:]lport:dst:port` (내 로컬에서 원격으로 터널)
  ```bash
  ssh -L 127.0.0.1:8443:db.internal:443 user@bastion
  curl https://localhost:8443/
  ```
- **원격 포워딩**: `-R rport:dst:port` (원격에서 내 쪽으로)
- **다이내믹(SOCKS) 포워딩**: `-D 1080` (로컬 SOCKS5 프록시)

**보안 주의**
- 바인드 주소 `127.0.0.1` 로 제한(무심코 0.0.0.0로 노출 금지).
- 접근통제(방화벽/SSH `PermitOpen`) 병행.

### 흔한 실수 & 체크리스트

- [ ] `PasswordAuthentication no`(가능하면 비활성) + **공개키만**.
- [ ] `PermitRootLogin prohibit-password` 이상.
- [ ] `KexAlgorithms`, `HostKeyAlgorithms`, `Ciphers`에서 구식 제거.
- [ ] 에이전트 포워딩 최소화, `ProxyJump` 사용.
- [ ] `known_hosts`/CA 기반 호스트 신뢰 체계화.
- [ ] **Fail2ban/로그 감시**, 키 수명/회전 정책.

---

## ✅ 10.3 전자우편 보안 — S/MIME & OpenPGP

### 문제 배경

전자우편은 **기본 평문** 전송(서버 간은 STARTTLS 사용 가능하지만 종단 간 보장은 별개).
**종단 간 보안**을 위해 **콘텐츠 암호화·서명**이 필요 → **S/MIME** 또는 **OpenPGP**.

---

### S/MIME (CMS 기반, PKI 신뢰)

- **표준**: CMS(Cryptographic Message Syntax, PKCS#7) 기반.
- **신뢰모델**: **X.509 PKI**(기업/조직 CA) — 인증서 발급·폐기·체인 검증.
- **기능**: `multipart/signed`(서명) / `application/pkcs7-mime`(암호/서명) / `application/pkcs7-signature`(서명 전용).

**워크플로**
1) 수신자 **공개키 인증서** 획득(디렉터리/카드/첨부).
2) 송신자는 **랜덤 세션키**로 메일 본문을 AE로 암호화 → 세션키를 **수신자 공개키**로 래핑(수신자별 하나씩) → **EnvelopedData** 생성.
3) 선택적으로 **SignedData**로 발신자 서명(체인 포함).
4) 수신자는 자신의 **개인키**로 세션키 복호 → 본문 복호/서명 검증.

**OpenSSL 예(개념)**
```bash
# 수신자 인증서로 암호화

openssl smime -encrypt -aes256 -in mail.txt -out mail.p7m -outform DER recipient.crt

# 서명 생성(발신자 측)

openssl smime -sign -in mail.txt -signer sender.crt -inkey sender.key -out mail_signed.eml -text

# 수신자 복호

openssl smime -decrypt -in mail.p7m -inform DER -recip me.crt -inkey me.key -out mail.dec.txt
```

**장점**: 조직/엔터프라이즈 배포 용이(PKI), 클라이언트 광범위 지원(Outlook/Apple Mail 등).
**주의**: PKI 운영(폐지/CRL/OCSP), 인증서 배포/백업(스마트카드/HSM 권장), 메일 게이트웨이와의 상호작용.

---

### OpenPGP (PGP/GnuPG, Web-of-Trust)

- **신뢰모델**: **웹오브트러스트**(서로의 키를 서명하여 신뢰를 전달) 또는 **키 디렉터리/투명성 로그**.
- **포맷**: `multipart/signed` + `application/pgp-encrypted` / `application/pgp-signature`.
- 개념은 S/MIME과 유사(세션키 암호화 + 본문 암호화 + 송신자 서명).
- **키 유형**: 주키 + 서브키(서명/암호/인증 분리) 권장. Revocation certificate 미리 생성.

**GnuPG 예(개념)**
```bash
# 키 생성

gpg --quick-generate-key "Alice <alice@example.com>" ed25519 cert,sign  # 주키
gpg --quick-add-key <Alice-keyid> cv25519 encrypt                        # 암호화 서브키

# 수신자 키 가져오기(키 서버/파일)

gpg --recv-keys <Bob-keyid>

# 메일 암호화(+서명)

gpg --encrypt --sign -r bob@example.com -o mail.asc mail.txt

# 검증/복호

gpg --decrypt mail.asc > mail.dec.txt
```

**장점**: 개인·커뮤니티 친화, 공급망 서명(소스코드·패키지)에서 광범위 채택.
**주의**: 키 분실·백업·폐기 절차, 사용자 교육(지문 확인, 신뢰 설정).

---

## ✅ 10.4 비교·선택 가이드

| 항목 | TLS 1.3 | SSH | S/MIME | OpenPGP |
|---|---|---|---|---|
| 보호 범위 | **전송 중**(세션) | 원격 쉘/포워딩 세션 | **메일 콘텐츠(종단 간)** | **메일/파일(종단 간)** |
| 신뢰 모델 | PKI(서버 인증서) | TOFU/호스트 CA | X.509 PKI | Web-of-Trust/디렉터리 |
| 키 합의 | (EC)DHE(+PSK) | (EC)DH | 하이브리드(세션키+공개키 래핑) | 동일 |
| 기본 무결성 | AEAD 포함 | MAC 포함 | 서명 옵션 | 서명 옵션 |
| 배포 적합 | 웹/서비스 | 관리·DevOps | 엔터프라이즈 메일 | 개인/OSS/커뮤니티 |

---

## ✅ 10.5 미스유스 패턴 & 방어 요령

**TLS**
- 0-RTT로 **비멱등** 요청 처리 → **리플레이 취약** ⇒ 0-RTT OFF 또는 멱등만 허용.
- 구식 스위트 활성화 ⇒ 서버 정책으로 TLS 1.2 이하 제한/차단, 관측 로깅.

**SSH**
- 에이전트 포워딩을 다중 홉에 남발 ⇒ `-A` 최소화, `ProxyJump` 사용.
- 호스트키 확인 생략 ⇒ known_hosts/호스트 CA 활용, `StrictHostKeyChecking yes`.

**S/MIME/PGP**
- 개인키 백업 부재·유실 ⇒ 스마트카드/HSM·리커버리 정책.
- 수신자 키 부정확(오탈자·피싱) ⇒ **지문**(fingerprint) 확인 절차.

---

## ✅ 10.6 운영 체크리스트

- **TLS 1.3**
  - [ ] AEAD 스위트만: AES-GCM/ChaCha20-Poly1305
  - [ ] 0-RTT 비활성(또는 멱등만)
  - [ ] HSTS/OCSP 스테이플링/알고리즘 관측
  - [ ] 키/인증서 회전 자동화(ACME 등)

- **SSH**
  - [ ] `PubkeyAuthentication yes`, 패스워드 로그인 제한
  - [ ] Ed25519 호스트키/사용자키, RSA≥3072
  - [ ] `ProxyJump`로 경유, 포워딩 최소화
  - [ ] 키 수명/회전, 감사 로그

- **S/MIME**
  - [ ] CA/발급/폐기/CRL·OCSP 운영
  - [ ] 개인키 보호(HSM/스마트카드), 백업/복구
  - [ ] 게이트웨이/아카이브와 상호운용 테스트

- **OpenPGP**
  - [ ] 주키/서브키 분리, 철회증명 사전 생성
  - [ ] 키 배포/지문 검증 절차
  - [ ] 사용자 교육(서명 검증, 피싱 주의)

---

## ✅ 10.7 미니 실습 묶음

### TLS 핸드셰이크 관찰

```bash
# JA3/알고리즘은 별도 도구 필요. 여기서는 서버 선택된 스위트·ALPN 확인

openssl s_client -connect www.example:443 -tls1_3 -alpn h2 -servername www.example
```

### SSH 에이전트 안전 사용

```bash
# 현재 에이전트에 등록된 키 확인

ssh-add -l
# 세션 한정 키(잠금 프롬프트) 추가

ssh-add -t 3600 ~/.ssh/id_ed25519
```

### S/MIME 서명/검증 라운드트립

```bash
openssl smime -sign -in note.txt -signer me.crt -inkey me.key -out note.signed.eml -text
openssl smime -verify -in note.signed.eml -CAfile ca.crt -out note.verified.txt
```

### PGP 암호화/서명 라운드트립

```bash
gpg --encrypt --sign -r bob@example.com -o msg.asc msg.txt
gpg --verify msg.asc
gpg --decrypt msg.asc > msg.dec.txt
```

---

## ✅ 10.8 요약 카드

- **TLS 1.3**: 1-RTT, HKDF 키 스케줄, AEAD 고정, 0-RTT는 **리플레이 위험**.
- **SSH**: KEX→호스트키→세션키, **known_hosts/CA**로 서버 신뢰, 포워딩 최소화.
- **S/MIME**: **PKI 기반** 메일 종단 보안, 조직 배포에 적합.
- **OpenPGP**: **웹오브트러스트/디렉터리**, 개인·커뮤니티/서명 체계에 강점.
- 공통: **키 관리/회전/로그**와 **사용자 절차**가 절반의 보안.

---

## ✅ 10.9 연습문제

1) TLS 1.3의 **키 스케줄**에서 Early/Handshake/Master Secret의 역할과, 각각에서 파생되는 **Traffic Secret**의 용도를 설명하라.
2) **0-RTT** 데이터가 가져오는 리스크를 **리플레이/정합성 관점**에서 사례와 함께 기술하라.
3) SSH에서 **호스트키 인증서**(OpenSSH CA)를 사용했을 때 **TOFU 대비 장점**과 운영 절차를 요약하라.
4) S/MIME와 OpenPGP의 **신뢰 모델 차이**(PKI vs Web-of-Trust)를 비교하고, 기업/개인 환경에서의 장단점을 나열하라.
5) 메일 종단 암호화에서 **세션키 래핑(enveloped)**의 장점과, 여러 수신자 처리 방식을 설명하라.
