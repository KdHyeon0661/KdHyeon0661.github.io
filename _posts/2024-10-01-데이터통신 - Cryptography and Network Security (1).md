---
layout: post
title: 데이터 통신 - Cryptography and Network Security (1)
date: 2024-10-01 20:20:23 +0900
category: DataCommunication
---
# Chapter 31. Cryptography and Network Security

## Introduction

### Security Goals — 무엇을 지키려 하는가

네트워크·시스템 보안에서 가장 많이 쓰는 프레임워크는 **CIA(+AAA)** 다:

- **Confidentiality (기밀성)**
- **Integrity (무결성)**
- **Availability (가용성)**
- **Authentication (인증)**
- **Authorization / Access control (인가·접근통제)**
- **Non-repudiation (부인 방지)**

ISO/IEC 7498-2와 IETF 용어집에서도 위와 비슷한 서비스를 표준적으로 정의한다.

각 목표를 네트워크 예제로 다시 정리해보자.

#### (1) Confidentiality — 누가 봐도 되는가?

- 의미: **데이터를 허가된 사람만 볼 수 있게 만드는 것**
- 예:
  - HTTPS로 접속한 쇼핑몰에서 카드 번호 전송
  - 회사 VPN으로 사내용 메신저를 쓸 때 대화 내용 숨기기
- 전형적인 기법:
  - 대칭키 암호 (AES-GCM, ChaCha20-Poly1305 등)
  - 비대칭키 암호로 대칭키를 교환 (RSA, ECDH 등)
  - 저장 데이터 암호화(디스크, DB, 백업)

#### (2) Integrity — 내용이 변하지 않았는가?

- 의미: 전송·저장되는 동안 **데이터가 몰래 바뀌지 않았음을 보장**
- 예:
  - 소프트웨어 업데이트 패키지가 중간에서 바뀌지 않았는지 확인
  - 전자계약 PDF가 서명된 이후 내용이 수정되지 않았는지 검증
- 주요 기법:
  - 해시(MD5, SHA-1은 사실상 폐기 / SHA-256 이상 권장)
  - **MAC / HMAC** (예: HMAC-SHA-256)
  - AEAD(AES-GCM, ChaCha20-Poly1305)의 “A” (Authenticated) 부분

#### (3) Availability — 필요할 때 쓸 수 있는가?

- 의미: 정당한 사용자가 **원할 때 서비스를 쓸 수 있게 하는 것**
- 예:
  - 대형 쇼핑몰이 블랙프라이데이 때 트래픽 폭주와 DDoS에도 버티는 것
  - 은행의 인터넷 뱅킹 서비스가 업무시간 중에는 항상 응답
- 기법:
  - DDoS 완화(Anycast, CDN, Rate limiting)
  - 리던던시(다중 서버, 다중 링크, 멀티 리전)
  - 우선순위·스케줄링(QoS) 등

#### (4) Authentication — 누구인지 확인하는 것

- **Peer entity authentication**: 통신 상대(클라이언트·서버)가 누구인지 검증
- **Data origin authentication**: 이 **메시지를 누가 보냈는지** 검증

예:

- 로그인 ID/비밀번호
- TLS 서버 인증서(브라우저 주소 표시줄의 자물쇠)
- OAuth2 / OpenID Connect 토큰

기법:

- 패스워드 + 솔팅된 해시
- 공개키 인증서(X.509), TLS 핸드셰이크
- OTP, FIDO2 보안키, SAML 토큰 등

#### (5) Non-repudiation — “안 했다”고 발뺌하지 못하게

- 의미: 어떤 행위를 한 뒤 **나중에 부인하지 못하게 만드는 것**
- 예:
  - 전자 계약서에 전자서명 → 서명자가 본인임과, 나중에 “내가 안 했다” 주장 방지
  - 온라인 뱅킹 고액 이체 시 전자 서명 기록
- 대표 기법:
  - **디지털 서명**(RSA, ECDSA, EdDSA 등)
  - 서명 검증 기록(타임스탬프, 감사 로그)

#### (6) Access Control / Authorization

- 의미: 인증된 사용자에게 **권한에 따른 접근만 허용**
- 예:
  - DB에서 관리자만 전체 고객 정보를 조회, 일반 상담원은 일부 필드만
  - 클라우드 IAM 정책(AWS IAM, GCP IAM 등)
- 기법:
  - ACL, RBAC, ABAC
  - 방화벽·보안 그룹
  - OAuth2 scope, API gateway 정책 등

---

### Attacks — 공격의 분류

보안 목표를 정의했다면, 그 목표를 깨려는 공격을 이해해야 한다. 현대 문헌에서는 크게 **수동(passive)**, **능동(active)** 공격으로 나눈다.

#### (1) Passive Attack — 훔쳐보기

- 특징:
  - 시스템 상태를 **변경하지 않고**, **엿듣기/분석만** 한다.
  - 탐지가 어렵다.
- 예:
  - 암호화되지 않은 Wi-Fi 트래픽 도청
  - TLS 사용 전 HTTP 로그인 페이지 스니핑
  - 트래픽 패턴 분석(메타데이터만 보고도 어느 서비스 쓰는지 추론)

수동 공격은 주로 **기밀성**에 대한 위협이다.

#### (2) Active Attack — 건드리기·조작하기

- 특징:
  - 메시지를 **수정 / 삽입 / 삭제 / 재전송 / 지연**하는 식으로 시스템을 변경
- 예:
  - **Man-in-the-Middle(MITM)**: 사이에 끼어 TLS를 가로채고 다른 인증서를 제시
  - **Replay 공격**: 이전의 정상 로그인 패킷을 캡처해 다시 보내 인증을 따내기
  - **DoS / DDoS**: 대량 트래픽을 보내 서비스 마비
  - **세션 하이재킹**: 쿠키 탈취 후 사용자의 세션을 가로채기

능동 공격은 기밀성뿐 아니라 **무결성·가용성·인증·부인방지** 모두에 영향을 준다.

#### (3) 암호 분석 관점의 공격 모델

암호 알고리즘 자체를 깨려는 공격은 보통 입력 정보를 어디까지 아는지에 따라 나눈다:

- **Ciphertext-only**: 암호문만 가지고 원문·키를 추측
- **Known-plaintext**: 일부 평문–암호문 쌍을 알고 있음
- **Chosen-plaintext / chosen-ciphertext**: 공격자가 원하는 평문·암호문에 대한 암·복호 결과를 얻을 수 있음

현대 실무에서는 설계 단계에서부터 **최악의 모델(선택 평문/암호문 공격)** 에서도 안전하도록 요구한다.

#### (4) 기타 실무 공격들

- **키 탈취**: 암호는 튼튼한데, 관리 부주의로 키 파일이 유출
- **사이드 채널**: 시간·전력 소비, 캐시 타이밍 등을 분석해 키 추론
- **구현 취약점**:
  - 패딩 오라클(CBC 모드 구현 오류)
  - 난수 생성기 취약점
  - 잘못된 IV 재사용 등

---

### Security Services and Techniques — 서비스와 메커니즘 매핑

ISO/IEC, IETF 문서에서는 **보안 서비스**(무엇을 제공하는가)와 **보안 메커니즘**(어떻게 제공하는가)을 구분한다.

아래 표처럼 생각하면 정리하기 좋다.

| 보안 목표 | 보안 서비스 | 핵심 기법(메커니즘) | 예시 프로토콜 |
|-----------|-------------|----------------------|---------------|
| 기밀성 | Data Confidentiality | 대칭키 암호(AES, ChaCha20), 키 교환(DH/ECDH), 터널링(IPsec, TLS) | HTTPS, VPN(IPsec, OpenVPN), 디스크 암호화 |
| 무결성 | Data Integrity | MAC/HMAC, 서명, 해시(SHA-2, SHA-3) | TLS 레코드, IPsec AH/ESP, S/MIME |
| 인증 | Entity/Data Origin Authentication | 디지털 서명, MAC, 패스워드+해시, 인증서 | TLS 서버 인증, OAuth2, SSH |
| 부인 방지 | Non-repudiation | 디지털 서명, 타임스탬프 | 전자 서명법 기반 전자계약, 서명된 이메일(S/MIME, OpenPGP) |
| 가용성 | Availability | 중복, 모니터링, Rate limiting, DDoS 방어, QoS | CDN, Anycast, WAF, 트래픽 스크러빙 |
| 접근통제 | Access Control | ACL, RBAC, 방화벽, 암호화 기반 토큰 | VPN, 방화벽 정책, IAM, API gateway |

#### 예제 시나리오: 인터넷 뱅킹

HTTPS 기반 인터넷 뱅킹을 예로 들어 **각 레이어에서 어떤 보안 서비스가 제공되는지** 보면:

1. **TLS**
   - 서버 인증(X.509 인증서) → Authentication
   - 세션 키 교환(DHE/ECDHE) → Confidentiality의 기반
   - AES-GCM/ChaCha20-Poly1305 → Confidentiality + Integrity
2. **애플리케이션**
   - 로그인 ID/비밀번호, 2FA → Authentication
   - 계좌별 권한 → Access control
   - 거래 내역 서명·로그 → Non-repudiation 보조
3. **인프라**
   - 방화벽·IDS/IPS·DDoS 방어 → Availability·Access control

---

## Confidentiality (기밀성)

기밀성은 “**메시지를 허가된 당사자만 읽을 수 있게 하는 것**”이다. 핵심 수단이 바로 **암호화(encryption)** 이고, 크게:

- **대칭키 암호(symmetric key cipher)**
- **비대칭키 암호(asymmetric/public-key cipher)**

두 가지를 결합한 **하이브리드 암호**가 실무 표준이다.

### Symmetric Key Ciphers — 대칭키 암호

#### (1) 기본 개념과 수식

동일한 키 $$K$$ 로 암·복호화 모두를 수행하는 방식이다.

- 암호화:
  $$ C = E_K(M) $$
- 복호화:
  $$ M = D_K(C) $$

여기서:

- $$M$$: 평문(plaintext)
- $$C$$: 암호문(ciphertext)
- $$E_K, D_K$$: 키 $$K$$ 를 사용하는 암·복호 함수

**보안의 핵심은 키를 비밀로 지키는 것**이다. 알고리즘은 공개돼도 괜찮다는 것이 현대 암호학의 기본 전제(케르크호프의 원칙).

#### (2) 블록 암호 vs 스트림 암호

현대 대칭 암호는 크게 두 유형으로 나뉜다.

1. **블록 암호 (Block cipher)**
   - 고정 길이 블록(예: 128비트)을 입력으로 받아 같은 길이의 블록을 출력
   - 대표 알고리즘: **AES**, Camellia, (구식: 3DES 등)
   - 블록 모드(ECB, CBC, CTR, GCM 등)를 통해 긴 데이터 암호화

2. **스트림 암호 (Stream cipher)**
   - **키스트림(의사 난수 스트림)** 을 생성하고,
   - 평문과 XOR 연산으로 암·복호
   - 대표 알고리즘: **ChaCha20**, (역사적으로 RC4 – 지금은 안전하지 않은 것으로 간주)

현대 인터넷 프로토콜(TLS 1.3 등)에서는 AES-GCM 같은 **AEAD 블록 암호 모드**와 ChaCha20-Poly1305 같은 스트림 기반 AEAD가 사실상 표준이다.

#### (3) 블록 암호 모드: 왜 ECB는 안 되는가

블록 암호 자체는 단일 블록 변환이므로, 긴 데이터에 쓰려면 **모드(mode of operation)** 가 필요하다.

- **ECB (Electronic Codebook)**:
  각 블록을 독립적으로 암호화. 빠르지만 **같은 평문 블록 → 같은 암호문 블록**이라 패턴이 노출된다. 현대 보안 지침에서는 ECB 사용을 사실상 금지한다.

- **CBC (Cipher Block Chaining)**:
  이전 암호문 블록과 XOR한 뒤 암호화. 동일한 평문이라도 다른 IV를 쓰면 다른 암호문이 된다. 다만 패딩 오라클 공격 등 구현 취약점이 많다.

- **CTR (Counter)**:
  카운터 값에 암호화를 적용해 키스트림을 생성, 평문과 XOR. 스트림 암호처럼 동작.

- **GCM (Galois/Counter Mode)**:
  CTR 기반에 **무결성 인증 태그**까지 제공하는 AEAD 모드. TLS, IPsec 등에서 표준.

#### (4) 현대 실무에서의 추천 알고리즘 & 키 길이

미국 NIST 및 다양한 서구 가이드라인은 **AES-128/192/256**을 기본 대칭키 암호로 채택하며, 128비트 이상 키를 권장한다.

- AES-128: 보통 **128-bit security**
- AES-256: 더 긴 키로 장기 보안을 노릴 수 있으나, 대부분의 웹 서비스에는 AES-128도 충분한 수준
- ChaCha20-Poly1305: AES 하드웨어 가속이 약한 모바일·임베디드 환경에서 선호

#### (5) 예제: AES-GCM으로 JSON 메시지 암호화 (파이썬)

실습 느낌을 살리기 위해 `cryptography` 라이브러리를 사용하는 예시를 보자. (개념 이해용 코드이다.)

```python
from cryptography.hazmat.primitives.ciphers.aead import AESGCM
import os, json

def encrypt_json(key: bytes, data: dict, aad: bytes = b"") -> tuple[bytes, bytes, bytes]:
    aesgcm = AESGCM(key)             # 16, 24, 32 bytes => AES-128/192/256
    nonce  = os.urandom(12)          # GCM 권장 96-bit nonce
    plaintext = json.dumps(data).encode("utf-8")
    ciphertext = aesgcm.encrypt(nonce, plaintext, aad)
    return nonce, ciphertext, aad

def decrypt_json(key: bytes, nonce: bytes, ciphertext: bytes, aad: bytes = b""):
    aesgcm = AESGCM(key)
    plaintext = aesgcm.decrypt(nonce, ciphertext, aad)
    return json.loads(plaintext.decode("utf-8"))

# 사용 예

key = AESGCM.generate_key(bit_length=128)
order = {"user": "alice", "amount": 120, "currency": "USD"}
aad   = b"order-service-v1"   # 인증만 하고 싶지만 암호화는 안 해도 되는 메타데이터

nonce, c, aad = encrypt_json(key, order, aad)
restored = decrypt_json(key, nonce, c, aad)
print(restored)
```

**상황 예시**

- 마이크로서비스 환경에서 `order-service` 와 `billing-service` 가
  내부 네트워크라도 민감한 주문 정보를 주고받을 때,
- 위와 같이 AES-GCM으로 **기밀성+무결성**을 동시에 확보.

여기서 주의점:

- **nonce(IV)를 중복 사용하면 절대 안 된다** (같은 키에서 동일 nonce 두 번 사용 금지)
- 키 관리는 코드 밖(KMS, HSM, 환경 변수나 컨피그 서버 등)에서 해야 한다.

#### (6) 대칭키 암호의 장단점

- 장점:
  - 매우 빠름 → 대량 데이터 암호화에 적합(디스크·VPN·영상 스트리밍 등)
  - 알고리즘 구조가 상대적으로 단순
- 단점:
  - **키 배포 문제**: 안전한 채널 없이 서로 키를 공유하기 어렵다.
  - 통신자 수가 많으면 $$O(n^2)$$ 개의 키가 필요

이 한계를 해결하기 위해 등장한 것이 바로 **비대칭키 암호**다.

---

### Asymmetric (Public-Key) Ciphers — 비대칭키 암호

#### (1) 기본 개념과 수식

비대칭키 암호에서는 **서로 다른 두 키**를 사용한다.

- 공개키 $$K_{pub}$$: 모두에게 알려도 되는 키
- 개인키 $$K_{priv}$$: 소유자만 알고 있어야 하는 키

암호화/복호화 수식은 다음과 같이 볼 수 있다.

- 기밀성용 암호화:
  - $$ C = E_{K_{pub}}(M) $$
  - $$ M = D_{K_{priv}}(C) $$
- 서명용:
  - 서명: $$ S = \text{Sign}_{K_{priv}}(M) $$
  - 검증: $$ \text{Verify}_{K_{pub}}(M, S) \Rightarrow \text{참/거짓} $$

공개키 암호의 아름다움은 **키 배포 문제를 크게 완화**한다는 점이다.

#### (2) 대표 알고리즘과 현대 권장 키 길이

1. **RSA**
   - 큰 정수의 소인수분해가 어려운 것에 기반
   - 웹·이메일·VPN 등에서 오래 쓰인 고전적인 공개키 암호
   - 미국·유럽 가이드라인은 **오늘날 최소 2048비트, 장기적으로 3072비트 이상**을 권장한다.

2. **ECC(Elliptic-Curve Cryptography)**
   - 타원 곡선 이산 로그 문제(ECDLP)에 기반한 공개키 암호
   - 같은 보안 강도에서 RSA보다 훨씬 작은 키 길이로 가능:
     - ECC-256(P-256) ≈ RSA-3072 수준의 보안 강도
   - 모바일·IoT·TLS 1.3 등에서 널리 사용

3. (참고) 양자 이후(Post-Quantum)
   - 현재 연구·표준화 진행 중이며, NIST PQC 프로젝트가 대표적이다.
   - 여기서는 기본 개념만 이해하면 충분.

#### (3) 하이브리드 암호 시스템

공개키 암호는 연산량이 크고, 처리 가능한 데이터 크기도 제한적이다.
그래서 실무에서는 **항상 “하이브리드” 구조**를 쓴다.

1. 클라이언트가 **무작위 대칭키 $$K$$** 생성
2. 이 $$K$$ 로 실제 데이터(대량)를 AES-GCM 등으로 암호화
3. $$K$$ 를 서버의 공개키 $$K_{pub}$$ 로 암호화:
   $$ C_K = E_{K_{pub}}(K) $$
4. 서버는 개인키 $$K_{priv}$$ 로 $$C_K$$ 를 복호화해 $$K$$ 를 얻는다.

이를 **키 캡슐화(key encapsulation)** 라고 부른다.

#### (4) 하이브리드 암호 예제 (파이썬 개념 코드)

아래는 개념 설명용으로 단순화한 코드다. 실제 서비스에서는 검증된 라이브러리를 사용해야 한다.

```python
from cryptography.hazmat.primitives.asymmetric import rsa, padding
from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.primitives.ciphers.aead import AESGCM
import os, json

# 서버 측: RSA 키 쌍 생성 (보통은 미리 만들어 두고 인증서로 배포)

private_key = rsa.generate_private_key(public_exponent=65537, key_size=3072)
public_key  = private_key.public_key()

# 클라이언트: 세션용 대칭키 생성

session_key = AESGCM.generate_key(bit_length=128)

# 데이터 암호화 (AES-GCM)

aesgcm = AESGCM(session_key)
nonce  = os.urandom(12)
payload = json.dumps({"msg": "Hello, confidential world!"}).encode("utf-8")
ciphertext = aesgcm.encrypt(nonce, payload, None)

# 세션키를 서버 공개키로 캡슐화

encrypted_key = public_key.encrypt(
    session_key,
    padding.OAEP(
        mgf=padding.MGF1(algorithm=hashes.SHA256()),
        algorithm=hashes.SHA256(),
        label=None
    )
)

# ---- 서버 측 수신 과정 ----
# 세션키 복호화

dec_session_key = private_key.decrypt(
    encrypted_key,
    padding.OAEP(
        mgf=padding.MGF1(algorithm=hashes.SHA256()),
        algorithm=hashes.SHA256(),
        label=None
    )
)
aesgcm_srv = AESGCM(dec_session_key)
plaintext = aesgcm_srv.decrypt(nonce, ciphertext, None)
print(plaintext.decode("utf-8"))
```

**상황 예시**

- HTTPS/TLS 핸드셰이크에서 서버 공개키(인증서)를 사용해
  세션키(혹은 그에 준하는 키 재료)를 교환하는 과정을 단순화해 표현한 것이다.
- 실제 TLS 1.3에서는 RSA 보다는 **(EC)Diffie-Hellman 키 교환**을 사용하지만,
  “공개키로 대칭키를 안전하게 교환한다”는 개념은 동일하다.

#### (5) 비대칭키 암호의 활용 영역

1. **키 교환(Key Exchange)**
   - 대칭키를 안전하게 합의/전달 → 이후 대칭키로 데이터 암호화

2. **디지털 서명(Digital Signature)**
   - 송신자가 자신의 **개인키**로 메시지를 서명
   - 누구나 공개키로 검증 가능 → 인증 + 무결성 + 부인 방지

3. **인증서 기반 인프라(PKI)**
   - CA가 발급한 인증서를 통해 공개키에 신뢰를 부여
   - HTTPS, 코드 서명, S/MIME 이메일 등

---

## 마무리 정리

지금까지의 내용을 한 번에 정리하면:

1. **31.1 Introduction**
   - 보안 목표: Confidentiality, Integrity, Availability, Authentication, Authorization, Non-repudiation
   - 공격: 수동/능동, 암호 분석 모델, 사이드 채널·키 관리·구현 취약점
   - 서비스와 메커니즘: 각 목표에 대응하는 암호 기법(암호화, MAC, 서명, 키 교환, 접근 통제 등)

2. **31.2 Confidentiality**
   - **대칭키 암호**:
     - AES, ChaCha20 등 현대 알고리즘 / 블록 vs 스트림 / AEAD 모드
     - 키 길이와 보안 강도의 개념, ECB 금지, IV/nonce 재사용 금지
   - **비대칭키 암호**:
     - RSA, ECC(P-256 등)와 키 길이 상관 관계
     - 디지털 서명과 키 교환
   - **하이브리드 암호**:
     - 실제 인터넷(HTTPS, VPN, 메시지 암호화)은 항상
       “공개키로 대칭키를 안전하게 교환 + 대칭키로 데이터 암호화” 패턴을 사용

이 정도를 이해하고 나면, 뒤에서 나올 **IPsec, TLS, 전자메일 보안(S/MIME, PGP), 응용계층 프로토콜의 보안 섹션**들을 볼 때
“어떤 서비스(기밀성/무결성/인증 등)를 어떤 조합의 알고리즘으로 제공하는지”를 구조적으로 읽을 수 있게 된다.
