---
layout: post
title: AWS - S3 Object Lock
date: 2025-07-18 21:20:23 +0900
category: AWS
---
# AWS S3 Object Lock 완벽 가이드: 데이터 삭제 방지와 규정 준수

## 왜 Object Lock인가? — 랜섬웨어·감사·규제 대응의 기준선

- **불변성(immutability)** 확립: **WORM(Write Once Read Many)** 모델로 **삭제/수정 불가** 보장
- **감사·규정 준수**: 금융/의료/공공 기록의 **법정 보존** 충족
- **백업 무결성**: 스냅샷/증분 백업 체인에 대한 **의도치 않은 삭제 방지**

### 개념 요약(초안 보강)

| 기능 | 설명 |
|---|---|
| **WORM** | 기록 후 **삭제/수정 불가** |
| **Retention(보존 기간)** | 기간 종료 전 삭제 금지 |
| **Mode** | **Governance**(관리자 우회 가능) / **Compliance**(절대 불가) |
| **Legal Hold** | 기간 제한 없이 보존(ON/OFF) |
| **Versioning 필수** | **버전 관리 활성화** 필요 |

---

## Object Lock 구성 요소 (정확 동작 모델)

| 구성 요소 | 설명 | 비고 |
|---|---|---|
| **Bucket-level Enablement** | **버킷 생성 시점**에만 활성화 가능 | 기존 버킷은 불가(새 버킷 필요) |
| **Default Retention** | 버킷 수준 **기본 Mode/기간** | 새 객체에 기본 적용(객체별 override 가능) |
| **Object Retention** | **버전별** 보존(Mode + UntilDate) | Compliance는 **연장만 가능**, 단축/제거 불가 |
| **Legal Hold** | 별도 플래그(ON/OFF) | Retention과 **독립** |
| **Bypass Header** | Governance 우회 삭제/변경 허용 | `s3:BypassGovernanceRetention` 권한 필요 |

---

## Governance vs Compliance (차이를 명확히)

| 항목 | Governance | Compliance |
|---|---|---|
| 삭제/수정 방지 | 기본 방지 | **절대적 방지** |
| 우회 가능 | **가능** (권한 + 헤더 필요) | **불가** (루트 포함) |
| 기간 변경 | 연장/단축 가능(권한 필요) | **연장만 가능** |
| 용도 | 내부 규정/랜섬웨어 대응 | 법규 준수(금융/의료/공공) |

---

## 사전 조건과 제약

- **반드시 새 버킷에서** **Object Lock 활성화** 체크
- **Versioning 자동 활성화**(필수)
- Retention **UTC ISO8601** 시각 사용(예: `2026-08-07T00:00:00Z`)
- Retention 기간과 Legal Hold는 **서로 독립**—둘 다 걸리면 **둘 다** 해제/만료되어야 삭제 가능

---

## 버킷 생성과 기본 설정 (콘솔·CLI·IaC)

### 콘솔 경로(요약)

1. **S3 → 버킷 만들기**
2. 고급 설정 → **Object Lock 활성화** 체크
3. (선택) **기본 보존**(Mode/기간) 설정
4. 생성

### CLI — 버킷 생성 시 Object Lock 활성화

```bash
aws s3api create-bucket \
  --bucket locked-bucket-$(date +%s) \
  --create-bucket-configuration LocationConstraint=ap-northeast-2 \
  --object-lock-enabled-for-bucket
```

> 핵심 플래그: `--object-lock-enabled-for-bucket`

### 버킷 기본 보존 구성(선택)

```bash
aws s3api put-object-lock-configuration \
  --bucket locked-bucket-123 \
  --object-lock-configuration '{
    "ObjectLockEnabled": "Enabled",
    "Rule": {
      "DefaultRetention": {
        "Mode": "GOVERNANCE",
        "Days": 30
      }
    }
  }'
```

> `Days` 또는 `Years` 지정 가능. DefaultRetention은 **새 객체**에 자동 적용.

### CloudFormation (스니펫)

```yaml
Resources:
  LockedBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub "locked-${AWS::AccountId}"
      ObjectLockEnabled: true
      VersioningConfiguration:
        Status: Enabled
      ObjectLockConfiguration:
        ObjectLockEnabled: Enabled
        Rule:
          DefaultRetention:
            Mode: GOVERNANCE
            Days: 30
```

### Terraform (스니펫)

```hcl
resource "aws_s3_bucket" "locked" {
  bucket = "locked-${var.account_id}"
  object_lock_enabled = true
  versioning {
    enabled = true
  }
}

resource "aws_s3_bucket_object_lock_configuration" "locked_cfg" {
  bucket = aws_s3_bucket.locked.id
  rule {
    default_retention {
      mode = "GOVERNANCE"
      days = 30
    }
  }
}
```

---

## 객체 업로드와 보존 지정 (정교한 예제)

### 업로드 시 개별 Retention 지정

```bash
aws s3api put-object \
  --bucket locked-bucket \
  --key backup.tar.gz \
  --body backup.tar.gz \
  --object-lock-mode GOVERNANCE \
  --object-lock-retain-until-date 2026-01-01T00:00:00Z
```

### 기존 객체에 Retention/Legal Hold 적용

```bash
# Retention 설정/변경(Compliance는 연장만 가능)

aws s3api put-object-retention \
  --bucket locked-bucket \
  --key important.db \
  --retention Mode=GOVERNANCE,RetainUntilDate=2026-08-07T00:00:00Z

# Legal Hold ON

aws s3api put-object-legal-hold \
  --bucket locked-bucket \
  --key important.db \
  --legal-hold Status=ON
```

### Governance 우회 삭제(권한 + 헤더 필요)

```bash
aws s3api delete-object \
  --bucket locked-bucket \
  --key backup.tar.gz \
  --bypass-governance-retention
```

- 필요 권한: `s3:BypassGovernanceRetention`
- 필요 전제: Mode=Governance, 아직 RetainUntilDate 미도래
- Compliance 모드는 **불가**

---

## 권한 설계 (최소 권한 + 우회 권한 분리)

### Governance 우회 권한 부여 샘플(초안 보강)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ObjectLockAdmin",
      "Effect": "Allow",
      "Action": [
        "s3:PutObjectRetention",
        "s3:PutObjectLegalHold",
        "s3:GetObjectRetention",
        "s3:GetObjectLegalHold"
      ],
      "Resource": "arn:aws:s3:::locked-bucket/*"
    },
    {
      "Sid": "BypassGovernance",
      "Effect": "Allow",
      "Action": [
        "s3:BypassGovernanceRetention",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::locked-bucket/*"
    }
  ]
}
```

> 운영에서는 **우회 권한을 별도 역할**로 분리하고, **MFA 조건 / 승인을 거친 임시 세션**만 사용하도록 설계.

---

## 검증·점검 커맨드 (운영 체크리스트)

```bash
# 버킷의 Object Lock 활성화 확인

aws s3api get-object-lock-configuration --bucket locked-bucket

# 객체 Retention 확인

aws s3api get-object-retention --bucket locked-bucket --key backup.tar.gz

# 객체 Legal Hold 확인

aws s3api get-object-legal-hold --bucket locked-bucket --key backup.tar.gz

# 객체 메타데이터(VersionId 포함)

aws s3api head-object --bucket locked-bucket --key backup.tar.gz
```

---

## 실전 랩 ① — “1년 불변 백업 버킷”

### 시나리오

- 요구사항: 1년간 **삭제/수정 절대 금지** 백업
- 해법: **Compliance** 모드 + RetainUntil 1년

```bash
# 버킷 생성(Object Lock)

aws s3api create-bucket \
  --bucket backups-secure \
  --create-bucket-configuration LocationConstraint=ap-northeast-2 \
  --object-lock-enabled-for-bucket

# 기본 보존(권장: Governance X, 명시적 Compliance 사용)
#   Compliance는 실수 위험이 크므로 DefaultRetention 대신 개별 객체에서 지정 권장

# 백업 업로드(Compliance + 1년)

aws s3api put-object \
  --bucket backups-secure \
  --key backup-2025-08.tar.gz \
  --body backup-2025-08.tar.gz \
  --object-lock-mode COMPLIANCE \
  --object-lock-retain-until-date 2026-08-07T00:00:00Z

# 확인

aws s3api get-object-retention \
  --bucket backups-secure \
  --key backup-2025-08.tar.gz
```

**중요**: Compliance는 **AWS 지원/루트** 포함 **누구도** 단축/삭제 불가. 실수 방지 절차(승인/체크리스트) 필수.

---

## 실전 랩 ② — “랜섬웨어 대비 백업” (Governance + 우회 통제)

### 정책

- 기본은 **Governance 30일**(삭제 금지)
- **보안 담당자 역할**만 **우회 권한** 보유(+MFA)

```bash
# 기본 보존 30일

aws s3api put-object-lock-configuration \
  --bucket org-backups \
  --object-lock-configuration '{
    "ObjectLockEnabled": "Enabled",
    "Rule": { "DefaultRetention": { "Mode": "GOVERNANCE", "Days": 30 } }
  }'

# 일일 백업 업로드(기본 30일 적용)

aws s3 cp backup-$(date +%F).tar.gz s3://org-backups/
```

- 사고 시(오탐/오배포) **보안 역할**이 **MFA 세션으로만** 우회 삭제 가능
- 일반 운영자는 **절대 삭제 불가**

---

## 운영·거버넌스 팁 (실무에서 놓치기 쉬운 부분)

1. **단축 금지 규칙**: Compliance는 **연장만 가능**. 거버넌스 문서/Runbook에 **굵게** 명시.
2. **Retention vs Legal Hold**: Legal Hold는 **기간 없음** + **즉시 효력**. 해제 전까지 어떤 RetainUntil보다 **강력**.
3. **DefaultRetention의 영향범위**: 이미 존재하는 객체에는 **소급 적용 안 됨**.
4. **객체 단위**: **Version별**로 보존—새 버전 업로드 시 **각 버전**이 독립 보존.
5. **수명주기(Lifecycle)와의 상호작용**: Retention/Legal Hold가 유효하면 **Transition/Expiration이 지연**될 수 있음.
6. **복제(Replication)**: **S3 Replication**은 Object Lock 메타를 대상 버킷으로 **전파** 가능. 대상도 **Object Lock 활성화+버전 관리**여야 함.
7. **스토리지 클래스**: Standard/IA/Glacier 계열 모두 **삭제 제한**은 동일 원칙. 다만 Glacier 계열은 **조기 삭제 수수료** 고려.
8. **비용 모델**: 보존기간 동안 **영구 저장 비용** 발생.
   총 비용 근사:
   $$
   \text{Cost} \approx \sum_{m=1}^{M} (G_m \cdot c_m) + \text{Requests} + \text{Retrieval}
   $$
   여기서 \(G_m\)은 월별 GB, \(c_m\)은 해당 클래스 단가.
9. **감사 추적**: **CloudTrail**로 `PutObjectRetention`, `PutObjectLegalHold`, `DeleteObject` 이벤트를 **중앙 S3/CloudWatch Logs**에 아카이브.
10. **우회 권한 관리**: `s3:BypassGovernanceRetention`은 **별도 역할 + MFA 조건부**로. 평시 **비활성화**(Deny SCP) 후 **변경창구 승인** 시 일시 해제.

---

## 감시·경보 (CloudWatch + EventBridge)

### 이벤트 패턴(예: Retention 변경 탐지)

```json
{
  "source": ["aws.s3"],
  "detail-type": ["AWS API Call via CloudTrail"],
  "detail": {
    "eventName": ["PutObjectRetention", "PutObjectLegalHold"]
  }
}
```

- 이 룰로 **SNS 알림** 발송 → 보안 채널/메일로 실시간 통지

---

## 대조: Object Lock vs. 단순 Versioning

| 항목 | Versioning | Object Lock |
|---|---|---|
| 삭제 방지 | ❌ (과거 버전 삭제 가능) | ✅ (Retention/Legal Hold) |
| 법적 준수 | 제한적 | ✅ (Compliance) |
| 운영 리스크 | 오작동 시 과거 버전도 제거 가능 | 우발 삭제 방지 강함 |

---

## FAQ (실무 질문 기반)

**Q1. 기존 버킷에 Object Lock을 켤 수 있나?**
A. **아니오.** **새 버킷**을 만들 때만 활성화 가능.

**Q2. Compliance 모드에서 잘못된 날짜를 짧게 고칠 수 있나?**
A. **불가.** **연장만** 가능. 테스트는 **반드시 Governance**로.

**Q3. Legal Hold와 Retention 중 어느 쪽이 더 강한가?**
A. 성격이 다름. **Legal Hold ON**이면 기간과 무관하게 **삭제 금지**. 둘 다 걸리면 **둘 다** 해제/만료되어야 삭제 가능.

**Q4. Glacier Deep Archive로 전환해도 잠금은 유지되나?**
A. **유지.** 단, 복구 지연/수수료 감안.

**Q5. 복제 대상에서도 동일 보존 정책으로 잠기나?**
A. **가능.** 대상 버킷이 **Object Lock 활성화+버전 관리** 상태여야 하며, **레플리케이션 규칙**에서 메타 전파가 이뤄진다.

---

## 미니 실습 — “증분 백업 체인 + 월간 장기 보존”

- **일일 증분**: Governance 30일
- **월간 풀백업**: Compliance 7년

```bash
BUCKET=datalake-backups

# 일일 증분(기본 Governance 30일 가정: DefaultRetention)

aws s3 cp incr-2025-11-10.tar.zst s3://$BUCKET/incr/2025/11/10/

# 월간 풀(명시적 Compliance 7년)

aws s3api put-object \
  --bucket $BUCKET \
  --key full/2025-11.tar.zst \
  --body full-2025-11.tar.zst \
  --object-lock-mode COMPLIANCE \
  --object-lock-retain-until-date 2032-11-01T00:00:00Z
```

운영 팁: **태그/프리픽스 규칙**으로 일일/월간 파일을 구분해 **검색/청구/감사**를 용이하게 한다.

---

## 문제 해결(트러블슈팅)

- **`AccessDenied`로 Retention 수정 실패**
  → 권한(`s3:PutObjectRetention`) 확인, Compliance는 **연장만 가능**. 거버넌스의 경우 우회 권한·헤더 확인.
- **`InvalidRequest` 시간 형식 오류**
  → ISO8601 + UTC(`Z`) 사용.
- **삭제 커맨드가 무시됨**
  → Retention/Legal Hold 상태 확인. Governance 우회는 `--bypass-governance-retention` 필요.
- **레플리케이션 미동작**
  → 대상 버킷 Object Lock/Versioning 활성화 여부, KMS 권한, 필수 버킷 정책 확인.
- **수명주기 정책이 안 먹힘**
  → Retention/Legal Hold가 유효하면 Transition/Expiration이 지연. 기간 만료 후 적용.

---

## 보안 아키텍처 샘플 — “우회는 승인 시에만”

1. **일반 운영 역할**: 읽기/쓰기 가능, **삭제/우회 불가**
2. **보안 관리자 역할**: `s3:BypassGovernanceRetention` 포함, **MFA 필수 조건부**
3. **SCP**: 평상시 **Deny**로 우회 기능 봉인 → **변경창구 승인** 시 일시 해제
4. **감사 로깅**: Retention/Legal Hold 변경/삭제 이벤트 **실시간 알림**(EventBridge→SNS)

---

## 수식으로 보는 리스크 저감(개념적)

랜섬웨어 공격 시 삭제 성공확률 \(p\) 를 가정할 때, Object Lock 도입 후 성공확률 \(p'\) 는

$$
p' \approx p \cdot \mathbf{1}[\text{NoRetention} \land \text{NoLegalHold}]
$$

Governance/Compliance/Legal Hold 중 하나라도 참이면 \(p' \to 0\)에 수렴(우회 권한의 엄격 제어 전제).

---

## 마무리 요약(초안 정리 + 보강 포인트)

| 항목 | 핵심 |
|---|---|
| 활성화 시점 | **버킷 생성 시**만 가능 |
| 필수 조건 | **Versioning** |
| 모드 | Governance(우회 가능) / Compliance(절대 불가) |
| 삭제 방지 수단 | **Retention + Legal Hold** |
| 권장 운영 | 테스트/운영 **분리**, 우회 권한 **MFA+승인 워크플로** |
| 백업 전략 | **일상 Governance**, **장기/규정 Compliance** |
| 감시 | CloudTrail + EventBridge + SNS 알림 |

---

## 명령 모음(치트시트)

```bash
# 버킷 생성(OL Enabled)

aws s3api create-bucket --bucket <name> \
  --create-bucket-configuration LocationConstraint=ap-northeast-2 \
  --object-lock-enabled-for-bucket

# 기본 보존(예: Governance 30일)

aws s3api put-object-lock-configuration --bucket <name> \
  --object-lock-configuration '{"ObjectLockEnabled":"Enabled","Rule":{"DefaultRetention":{"Mode":"GOVERNANCE","Days":30}}}'

# 업로드 시 명시적 보존

aws s3api put-object --bucket <name> --key a.tar \
  --body a.tar --object-lock-mode COMPLIANCE \
  --object-lock-retain-until-date 2026-01-01T00:00:00Z

# Retention/Legal Hold 조회

aws s3api get-object-retention --bucket <name> --key a.tar
aws s3api get-object-legal-hold --bucket <name> --key a.tar

# Governance 우회 삭제

aws s3api delete-object --bucket <name> --key a.tar --bypass-governance-retention
```
