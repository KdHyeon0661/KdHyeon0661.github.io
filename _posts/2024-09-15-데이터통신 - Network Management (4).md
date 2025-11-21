---
layout: post
title: 데이터 통신 - Network Management (4)
date: 2024-09-15 23:20:23 +0900
category: DataCommunication
---
# — 언어 기초, 데이터 타입, 인코딩(ASN.1, BER/DER/PER/XER/JER)

이 장은 27장 Network Management(특히 SNMP, X.509, 기타 관리 프로토콜)의 “ASN.1” 부분을 블로그용으로 재구성·확장한 것이다.
SNMP MIB, X.509 인증서, 많은 텔코 프로토콜이 **ASN.1로 데이터 구조를 정의하고, X.690 계열 인코딩 규칙**으로 전송한다.

구조는 다음 세 파트로 나눈다.

1. **Language basics**: 모듈 구조, 타입/값 정의, 태그(tag), 제약(constraint)
2. **Data types**: 원시/구성 타입, Universal/Application/Context-specific/Private 클래스
3. **Encoding**: BER/DER/CER(전통 TLV), PER(압축), XER/XML, JER/JSON 인코딩 개념

---

## ASN.1 개요와 역할

### ASN.1이 하는 일

**ASN.1은 “데이터 구조를 기술하는 언어”**다. 프로토콜 메시지의 구조를 문법적으로 정의해 놓으면, 양 끝단(에이전트/매니저, 클라이언트/서버)이 같은 정의를 공유하기만 하면 서로 다른 CPU/언어/OS라도 데이터를 주고받을 수 있다.

예:

- **SNMP**: SMIv2가 ASN.1 매크로를 사용해 MIB 테이블을 정의하고, 전송은 ASN.1 + BER.
- **X.509 인증서(RFC 5280)**: 인증서 구조, 확장 필드(subjectAltName 등)를 ASN.1 타입으로 정의하고 DER로 인코딩.
- **텔코·무선(3GPP/5G)**: 시그널링 메시지 구조를 ASN.1로 정의하고, PER(특히 unaligned)을 사용해 매우 압축된 이진 형식으로 전송.

즉, ASN.1은 **“추상 문법”**이고, X.690/X.691/X.693/X.697 등은 **“전송 문법(인코딩 규칙)”**이다.

---

## Language Basics — 언어 기초

### 모듈 구조

ASN.1 파일은 보통 **모듈(Module)** 단위로 구성된다.

```asn1
My-Example-MODULE DEFINITIONS AUTOMATIC TAGS ::= BEGIN

IMPORTS
    Integer32
        FROM SNMPv2-SMI
    DisplayString
        FROM SNMPv2-TC;

Person ::= SEQUENCE {
    name      UTF8String,
    age       Integer32 (0..150),
    email     IA5String OPTIONAL
}

END
```

구조:

- `모듈명 DEFINITIONS [태깅 모드] ::= BEGIN ... END`
- `IMPORTS` 절로 다른 모듈의 타입을 가져옴
- 각 줄이 **타입 정의(Type assignment)** 혹은 **값 정의(Value assignment)**

실무에서는:

- `SNMPv2-SMI`, `SNMPv2-TC` 등이 IETF MIB 모듈 이름(RFC 3418 등).
- X.509도 `PKIX1Explicit88`, `PKIX1Implicit88` 같은 모듈로 정의.

### 타입 정의와 값 정의

두 가지 핵심 문장:

1. **타입 정의**

```asn1
TypeName ::= SOME-ASN1-TYPE
```

예:

```asn1
Age ::= INTEGER (0..150)

PersonName ::= UTF8String (SIZE (1..64))
```

2. **값 정의**

```asn1
constName TypeName ::= value
```

예:

```asn1
maxAge Age ::= 150

defaultAdminEmail IA5String ::= "admin@example.com"
```

**주석**은 `-- ...` 형식으로 사용한다:

```asn1
-- This is a comment
Person ::= SEQUENCE {
    name UTF8String, -- person name
    ...
}
```

### 태그(Tag)와 태그 클래스

ASN.1 값은 인코딩 시 **Tag-Length-Value(TLV)** 또는 **Identifier-Length-Contents(ILC)** 구조로 표현된다. 태그는 크게 네 클래스.

| 클래스 | 설명 | 예 |
|--------|------|----|
| UNIVERSAL | ASN.1 정의에 공통인 타입 | INTEGER(2), OCTET STRING(4), SEQUENCE(16) |
| APPLICATION | 특정 프로토콜·응용 전용 | 예: `APPLICATION 0`에서 메시지 헤더 |
| CONTEXT-SPECIFIC | 구조 내 위치에 따라 의미 결정 | `[0]`, `[1]` 등 |
| PRIVATE | 개인/회사 내부 정의용 | `[PRIVATE 5]` 등 |

태그 표기:

```asn1
-- Context-specific tag 0, IMPLICIT
SimpleString ::= [0] IMPLICIT UTF8String

-- Application class tag 3, EXPLICIT
AppMessage ::= [APPLICATION 3] EXPLICIT SEQUENCE {
    version  INTEGER,
    payload  OCTET STRING
}
```

#### IMPLICIT vs EXPLICIT vs AUTOMATIC

- **IMPLICIT**: 새 태그로 “교체”
- **EXPLICIT**: 새 태그로 “한 번 더 감싸기”
- **AUTOMATIC TAGS**: 모듈 레벨에서 자동 태깅

예:

```asn1
MyModule DEFINITIONS AUTOMATIC TAGS ::= BEGIN

Person ::= SEQUENCE {
    name UTF8String,
    age  INTEGER
}

END
```

`AUTOMATIC TAGS`를 쓰면, `Person` 안의 `name`과 `age`는 자동으로 `[0]`, `[1]` 같은 context-specific 태그를 가진다(정확한 규칙은 X.680 기준).

### 제약(Constraints)

ASN.1은 단순 타입에 **제약**을 붙여 값 범위를 제한할 수 있다.

대표적인 제약:

- **범위 제약**: `(min..max)`
- **SIZE 제약**: `SIZE (min..max)`
- **조합**: `SIZE (1..64) (FROM ("A".."Z" | "a".."z"))` 같은 것까지 가능(X.682).

예:

```asn1
Age ::= INTEGER (0..150)

Username ::= UTF8String (SIZE (1..32))

Ipv4Address ::= OCTET STRING (SIZE (4))
Ipv6Address ::= OCTET STRING (SIZE (16))
```

PER 같은 압축 인코딩 규칙(PER)은 이 제약을 활용해 비트를 줄인다. 범위가 좁을수록 필요한 비트 수가 줄어든다.

### Network Management와 ASN.1 예시(간단한 MIB)

SNMP MIB 예(개념):

```asn1
Example-MIB DEFINITIONS ::= BEGIN

IMPORTS
    OBJECT-TYPE, Integer32, enterprises
        FROM SNMPv2-SMI
    DisplayString
        FROM SNMPv2-TC;

exampleMIB OBJECT IDENTIFIER ::= { enterprises 99999 }

exampleObjects OBJECT IDENTIFIER ::= { exampleMIB 1 }

exampleDeviceName OBJECT-TYPE
    SYNTAX      DisplayString (SIZE (0..255))
    MAX-ACCESS  read-only
    STATUS      current
    DESCRIPTION "Device name."
    ::= { exampleObjects 1 }

END
```

여기서:

- `OBJECT-TYPE` 매크로는 SMI 문법이지만, 내부적으로는 ASN.1 타입·값으로 풀린다.
- `DisplayString`은 `OCTET STRING`에 `SIZE` + 문자셋 제약을 붙인 것.

---

## Data Types — 데이터 타입

X.680은 여러 **원시(primitive)**·**구성(constructed)** 타입과 태그 번호를 정의한다.

### Primitive vs Constructed

- **Primitive 타입**: 값이 하나의 “스칼라”처럼 직접 인코딩
  - BOOLEAN, INTEGER, OCTET STRING, NULL, OBJECT IDENTIFIER 등
- **Constructed 타입**: 다른 ASN.1 값들의 집합으로 구성
  - SEQUENCE, SET, SEQUENCE OF, SET OF, CHOICE 등

BER/DER에서는 이 구분이 인코딩의 P/C 비트에 반영된다(Identifier octet에서).

### 주요 Universal Primitive 타입

#### BOOLEAN

```asn1
isEnabled BOOLEAN
```

- BER에서 `FALSE` → 내용 0x00, `TRUE` → 0xFF(또는 다른 non-zero; DER은 0xFF로 고정).

#### INTEGER

```asn1
Age ::= INTEGER (0..150)
```

- 2의 보수, big-endian, 최소 바이트 수로 표현(DER 기준).

예: 값 300(10진수) 인코딩

- 16진수: 0x012C → 두 바이트 `01 2C`
- BER/DER에서 Tag = 0x02(INTEGER), Length = 0x02, Value = 0x01 0x2C → `02 02 01 2C`

#### ENUMERATED

```asn1
Color ::= ENUMERATED {
    red(0),
    green(1),
    blue(2)
}
```

- 실질적으로는 INTEGER와 같지만, **의미 있는 심볼 이름**을 붙여줌.
- PER, XER, JER 등에서 ENUM 이름 그대로 또는 숫자로 인코딩 규칙이 달라질 수 있다.

#### OCTET STRING / BIT STRING

```asn1
Payload ::= OCTET STRING (SIZE (0..1024))

Flags ::= BIT STRING {
    up(0),
    running(1),
    adminDown(2)
}
```

- `OCTET STRING`: 바이트 배열
- `BIT STRING`: 비트 단위 필드. SNMP의 `TruthValue`, `RowStatus` 같은 것들이 이런 타입과 매핑된다(매크로 레벨에서).

#### OBJECT IDENTIFIER

OID는 ASN.1에서 **전 세계적으로 유일한 네임스페이스**를 만든다.

예:

```asn1
iso               OBJECT IDENTIFIER ::= { 1 }
identified-organization OBJECT IDENTIFIER ::= { iso 3 }
dod              OBJECT IDENTIFIER ::= { identified-organization 6 }
internet         OBJECT IDENTIFIER ::= { dod 1 }
directory        OBJECT IDENTIFIER ::= { internet 1 }
mgmt             OBJECT IDENTIFIER ::= { internet 2 }
private          OBJECT IDENTIFIER ::= { internet 4 }
```

X.690의 규칙에 따라 첫 두 개 `a1.a2`는 다음과 같이 결합된다.

$$
\text{firstOctet} = 40 \times a_1 + a_2, \quad 0 \le a_1 \le 2,\; 0 \le a_2 \le 39
$$

나머지 arc는 7비트씩 끊어서 base-128 continuation 형식으로 인코딩한다.

### Constructed 타입

#### SEQUENCE / SEQUENCE OF

```asn1
Person ::= SEQUENCE {
    name   UTF8String,
    age    INTEGER (0..150),
    email  IA5String OPTIONAL
}

IntList ::= SEQUENCE OF INTEGER
```

- **SEQUENCE**: 필드가 순서 있게 나열
- **OPTIONAL** 필드는 값이 없으면 인코딩되지 않는다.
- **DEFAULT** 값은 실제 값이 default와 같으면 인코딩 생략(DER 조건 하에서).

#### SET / SET OF

- **SET**: 순서의 의미가 없고, DER에서는 필드 인코딩 순서를 태그 값 기준으로 정렬해야 한다.

```asn1
NameParts ::= SET {
    family UTF8String,
    given  UTF8String,
    middle UTF8String OPTIONAL
}
```

#### CHOICE

```asn1
IpAddress ::= CHOICE {
    ipv4  OCTET STRING (SIZE (4)),
    ipv6  OCTET STRING (SIZE (16))
}
```

- 한 시점에 오직 하나의 대안을 가진다.
- 인코딩 시, 선택된 대안의 태그가 사용.

---

## Encoding — BER/DER/CER, PER, XER, JER

X.690/X.691/X.693/X.697은 ASN.1 타입을 실제 바이트 또는 XML/JSON으로 인코딩하는 규칙을 정의한다.

### BER의 기본 구조: TLV

**BER(Basic Encoding Rules)** 는 각 요소를 `Identifier(또는 Tag) + Length + Contents`로 인코딩한다.

1. **Identifier(또는 Tag) 옥텟**
   - 상위 2비트: 클래스(UNIVERSAL=00, APPLICATION=01, CONTEXT-SPECIFIC=10, PRIVATE=11)
   - 그 다음 1비트: Primitive(0)/Constructed(1)
   - 나머지 5비트: 태그 번호(0~30). 31 이상이면 추가 바이트 사용.
2. **Length 옥텟**
   - 짧은 형식(short): 최상위 비트 0, 나머지 7비트가 길이(0~127).
   - 긴 형식(long): 최상위 비트 1, 나머지 7비트는 “길이를 나타내는 옥텟 수”.
3. **Contents 옥텟**
   - 실제 값.

#### 예: INTEGER 300 인코딩

300(10진) = 0x012C. DER/BER:

- Identifier: `02` (UNIVERSAL, primitive, tag=2)
- Length: `02`
- Contents: `01 2C`

→ 전체 바이트: `02 02 01 2C`

### Length 인코딩 예제

#### 짧은 형식

길이 100(0x64):

- Length 옥텟 = 0x64 (최상위 비트 0, 나머지 7비트가 100)

#### 긴 형식

길이 435(10진) (예: 큰 문자열)

- 435(10) = 0x01B3 → 두 바이트 필요
- 첫 길이 옥텟: 0x82 (1000 0010₂, 상위 비트 1 + “뒤에 2바이트 있음”)
- 다음 두 옥텟: 0x01 0xB3
- 총 Length 필드: `82 01 B3`
X.690에서 이 구조 예시를 그대로 제시한다.

### Constructed 타입 인코딩

SEQUENCE는 Constructed 타입이므로:

- Identifier: 클래스 + Constructed 비트(1) + tag=16(0x10)
- Length: 전체 내용 길이
- Contents: 각 필드의 TLV를 순서대로 붙인 것

예:

```asn1
Person ::= SEQUENCE {
    name UTF8String,
    age  INTEGER
}
```

값:

```asn1
name = "AL"
age  = 30
```

단계:

1. `name` 인코딩
   - Tag: UTF8String(Universal 12 → 0x0C)
   - 값 `"AL"`: 2바이트 `41 4C`
   - Length: 0x02
   - TLV: `0C 02 41 4C`
2. `age` 인코딩
   - INTEGER(2)
   - 30 = 0x1E → `1E`
   - TLV: `02 01 1E`
3. SEQUENCE 인코딩
   - Contents: `0C 02 41 4C 02 01 1E` (총 7바이트)
   - Identifier: SEQUENCE, Constructed → 0x30
   - Length: 0x07
   - 최종: `30 07 0C 02 41 4C 02 01 1E`

### DER와 CER — Canonical 규칙

X.690은 **BER**, **CER(Canonical Encoding Rules)**, **DER(Distinguished Encoding Rules)** 를 규정한다.

- **BER**: 인코딩 선택의 자유(예: BOOLEAN TRUE를 여러 값으로 표현 가능, 길이 short/long 선택 등).
- **DER**:
  - **디지털 서명** 등에 쓰기 위해 **하나의 유일한 인코딩**만 허용.
  - 특징:
    - 길이는 항상 **정의 길이(definite)** + 가능한 한 짧은 형식
    - BOOLEAN TRUE는 반드시 0xFF
    - SET 요소는 태그 오름차순으로 정렬
  - X.509 인증서, CRL에서 필수.
- **CER**:
  - 마찬가지로 canonical이지만, 긴 문자열 등 일부 경우에 **indefinite length**를 허용하는 변종.

### PER — Packed Encoding Rules

X.691/PER는 **"비트를 최대한 아끼는" 인코딩 규칙**이다.

- 길이·태그 정보를 생략하거나 압축.
- 제약 정보(예: `(0..150)` 범위)를 이용해 필요한 비트 수를 최소화.
- 3GPP 이동통신 등에서 표준적으로 사용.

예:

```asn1
Age ::= INTEGER (0..150)
```

- 0..150 → 값 개수 151개
- 필요한 비트 수는 $$\lceil \log_2(151) \rceil = 8$$ 비트
- 따라서 PER는 Age를 1바이트로 표현(0~150 중 하나).

### XER(XML Encoding Rules)와 JER(JSON Encoding Rules)

X.693/X.697은 ASN.1 값을 XML/JSON으로 표현하기 위한 규칙이다.

- **XER**: XML 기반 인코딩. 사람·툴이 보기 쉬운 형태.
- **JER**: JSON 기반 인코딩(2021년 X.697로 표준화). 현대 REST/웹환경과 잘 맞는다.

예를 들어,

```asn1
Person ::= SEQUENCE {
    name UTF8String,
    age  INTEGER
}
```

값 `name="Alice", age=30`을 JER로 표현하면 대략:

```json
{
  "name": "Alice",
  "age": 30
}
```

(정확한 레이아웃은 JER 규칙에 따라 결정된다.)

### 실전 예제: X.509 Subject 인코딩 스케치

X.509 Subject는 대략 다음과 같은 ASN.1 구조(간략화):

```asn1
Name ::= CHOICE {
  rdnSequence  RDNSequence
}

RDNSequence ::= SEQUENCE OF RelativeDistinguishedName

RelativeDistinguishedName ::= SET SIZE (1..MAX) OF AttributeTypeAndValue

AttributeTypeAndValue ::= SEQUENCE {
  type   OBJECT IDENTIFIER,
  value  ANY
}
```

실제 인증서를 DER로 디코딩하면 다음과 같은 TLV 트리가 나온다(개략):

- SEQUENCE (Name)
  - SET
    - SEQUENCE
      - OBJECT IDENTIFIER (예: commonName OID)
      - UTF8String ("example.com")
  - SET
    - SEQUENCE
      - OBJECT IDENTIFIER (countryName)
      - PrintableString ("US")
  - ...

이처럼 ASN.1 타입 정의 → DER 인코딩 → 인증서 바이너리라는 흐름으로 이어진다.

---

## 예제 코드와 실습 아이디어

이 장은 주로 **이론·형식**에 집중하지만, 실제 관리 시스템·보안 코드에서 ASN.1이 어떻게 쓰이는지 감을 잡기 위해 간단한 코드 예시를 들어보자.

### Python(pyasn1)을 이용한 단순 디코딩 예(개념)

```python
from pyasn1.type import univ, char, namedtype
from pyasn1.codec.der import decoder

class Person(univ.Sequence):
    componentType = namedtype.NamedTypes(
        namedtype.NamedType('name', char.UTF8String()),
        namedtype.NamedType('age',  univ.Integer())
    )

der_bytes = bytes.fromhex('30 07 0C 02 41 4C 02 01 1E')  # 앞에서 본 예

person, rest = decoder.decode(der_bytes, asn1Spec=Person())
print(person['name'], int(person['age']))
```

실제로는:

- MIB 모듈을 ASN.1 → 코드로 컴파일(asn1c, OSS Nokalva 등)
- 또는 pyasn1, go-asn1 같은 라이브러리를 이용해 동적으로 디코딩.

### TLV 디코더 의사코드

ASN.1 BER/DER 스트림을 읽는 최소 구조는 거의 항상 **TLV 루프**다.

```pseudo
while not end_of_stream:
    tag = read_identifier_octets()
    length = read_length_octets()
    value_bytes = read_exact(length)
    process(tag, value_bytes)
```

- `read_identifier_octets()`는 class/P/C/type 정보를 추출
- `read_length_octets()`는 short/long/indefinite를 처리
- constructed 타입이면 `value_bytes` 안을 재귀적으로 다시 TLV 파싱

---

## 정리

**ASN.1**은:

- SNMP, X.509, 텔코 시그널링, 많은 네트워크 관리/보안 프로토콜에서 쓰이는 **표준 데이터 정의 언어**이며,
- X.690/691/693/697 계열 인코딩 규칙(BER, DER, PER, XER, JER 등)과 결합되어
  - 효율적인 이진 전송(예: PER, BER/DER)
  - 사람이 읽기 좋은 XML/JSON 표현(XER, JER)을 동시에 제공한다.

이 장에서 정리한 세 가지 축:

1. **Language basics**
   - 모듈 구조, 태그 클래스, IMPLICIT/EXPLICIT/AUTOMATIC, 제약
2. **Data types**
   - BOOLEAN/INTEGER/ENUM/OCTET STRING/OBJECT IDENTIFIER, SEQUENCE/SET/CHOICE 등
3. **Encoding**
   - BER/DER/CER의 TLV 구조, PER의 압축 인코딩, XER/JER 등 현대적인 텍스트 인코딩
