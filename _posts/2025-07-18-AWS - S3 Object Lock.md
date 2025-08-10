---
layout: post
title: AWS - S3 Object Lock
date: 2025-07-18 21:20:23 +0900
category: AWS
---
# 🔐 AWS S3 Object Lock 완벽 가이드: 데이터 삭제 방지와 규정 준수

---

## 🧭 개요

**S3 Object Lock**은 Amazon S3에 저장된 객체에 대해  
**일정 기간 또는 영구적으로 삭제를 방지**하는 기능입니다.

주로 다음과 같은 상황에서 사용됩니다:

- **데이터 변경 불가 요구** (e.g., 법적 규제, 감사 목적)
- **랜섬웨어 대응** (데이터 불변성 확보)
- **백업 보존 전략** (WORM: Write Once Read Many)

---

## 🔎 핵심 기능 요약

| 기능 | 설명 |
|------|------|
| WORM 모델 | 한번 쓰면 변경/삭제 불가 (Write Once, Read Many) |
| 보존 기간 설정 | 시간 단위로 삭제 금지 기간 지정 가능 |
| 삭제 방지 모드 | **Governance** / **Compliance** 모드 |
| Legal Hold | 별도 보존 플래그 설정 (기간 제한 없음) |
| 버전 관리 필요 | 반드시 **버전 관리 활성화된 버킷에서만 사용 가능** |

---

## 📦 Object Lock 구성 요소

| 구성 요소 | 설명 |
|-----------|------|
| **Retention Mode** | 객체에 대한 삭제/수정 제한 모드 설정 |
| **Retention Period** | 보존 기간 (예: 7일, 1년 등) |
| **Legal Hold** | 법적 보존 태그로, 기간 없이 유지 가능 |
| **Version ID** | 각 객체 버전에 대해 별도 정책 적용됨 |

---

## ⚖️ 모드 비교: Governance vs Compliance

| 항목 | Governance | Compliance |
|------|------------|------------|
| 삭제 방지 | 관리자만 무시 가능 (권한 필요) | **절대 삭제 불가** |
| 보존 무시 가능 여부 | O (IAM 권한 필요) | X (아예 차단) |
| 사용 사례 | 내부 규칙, 랜섬웨어 대응 | 법적 규제, 금융/의료 등 |
| 변경 가능 여부 | O (특정 권한) | X (AWS도 불가) |

---

## 🛠️ 설정 절차

### 1️⃣ Object Lock 사용 가능 버킷 생성

기존 버킷에는 설정 불가합니다. 새 버킷을 만들며 체크해야 합니다.

1. S3 → **버킷 만들기**
2. 고급 설정 → ✅ `Object Lock 활성화` 체크
3. 버전 관리 자동 활성화됨

---

### 2️⃣ 객체 업로드 시 보존 설정

```bash
aws s3api put-object \
  --bucket locked-bucket \
  --key backup.tar.gz \
  --body backup.tar.gz \
  --object-lock-mode GOVERNANCE \
  --object-lock-retain-until-date 2025-12-31T00:00:00Z
```

| 옵션 | 설명 |
|------|------|
| `--object-lock-mode` | `GOVERNANCE` or `COMPLIANCE` |
| `--object-lock-retain-until-date` | UTC ISO8601 형식 |

---

### 3️⃣ 기존 객체에 Legal Hold 적용

```bash
aws s3api put-object-legal-hold \
  --bucket locked-bucket \
  --key important.docx \
  --legal-hold Status=ON
```

- `Status=ON` → 객체는 영구적으로 삭제/변경 불가

---

## 📜 JSON 정책 예시 (Governance 모드)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowBypassGovernanceRetention",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:user/admin"
      },
      "Action": [
        "s3:BypassGovernanceRetention",
        "s3:PutObjectRetention"
      ],
      "Resource": "arn:aws:s3:::locked-bucket/*"
    }
  ]
}
```

> 특정 사용자만 Governance 모드를 무시하고 삭제할 수 있도록 허용하는 정책입니다.

---

## 🧪 실습 예시

### 시나리오: 1년간 변경/삭제 불가한 백업 파일 생성

1. 버킷 생성 시 Object Lock 활성화
2. 다음 명령어로 업로드

```bash
aws s3api put-object \
  --bucket backups-secure \
  --key backup-2025-08.tar.gz \
  --body backup-2025-08.tar.gz \
  --object-lock-mode COMPLIANCE \
  --object-lock-retain-until-date 2026-08-07T00:00:00Z
```

3. 확인:

```bash
aws s3api get-object-retention \
  --bucket backups-secure \
  --key backup-2025-08.tar.gz
```

---

## ⚠️ 주의사항

| 항목 | 설명 |
|------|------|
| 기존 버킷에는 설정 불가 | 새로 생성 시에만 활성화 가능 |
| 버전 관리 필수 | 버전 관리가 비활성화되면 Object Lock도 사용 불가 |
| Compliance 모드 신중히 설정 | 삭제 불가능 → 데이터 실수로 고정될 수 있음 |
| UTC 시간 형식 | `RetainUntilDate`는 UTC 기반 ISO 형식 사용 |
| 비용 | 보존 기간 동안 객체 삭제 못하므로, 비용 발생 지속 |

---

## 📂 Object Lock vs S3 버전 관리

| 항목 | S3 Versioning | Object Lock |
|------|---------------|-------------|
| 이전 버전 보관 | ✅ 가능 | ✅ 가능 |
| 삭제 방지 | ❌ 기본 불가 | ✅ 가능 (Retention 설정) |
| 복원 가능성 | ✅ | ✅ (단, 삭제 불가 정책 적용 시 제외) |
| 목적 | 변경 추적, 복구 | **보존**, **불변성 확보**, 규정 준수 |

---

## 💼 실무 활용 시나리오

### ① 랜섬웨어 방지

- 백업 데이터를 Object Lock + Governance 모드로 보호
- 삭제 명령 무시됨 → 감염 피해 최소화

### ② 금융 데이터 보존

- 회계, 고객 데이터 등은 7년 보존 요구
- Compliance 모드로 설정 → 법적 규정 충족

### ③ 로그 증거 확보

- 보안 감사 로그 → 변경/삭제 방지
- S3 + CloudTrail + Object Lock 조합

---

## ✅ 요약

| 항목 | 설명 |
|------|------|
| Object Lock 활성화 | 버킷 생성 시 옵션 필요 |
| 두 가지 모드 | Governance / Compliance |
| 버전 관리 필수 | Object Lock 사용 조건 |
| 삭제 방지 수단 | Retention + Legal Hold |
| 실무 적용 | 보안, 규제, 랜섬웨어 대응에 필수 |
