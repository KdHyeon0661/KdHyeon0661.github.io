---
layout: post
title: AWS - DynamoDB
date: 2025-07-23 21:20:23 +0900
category: AWS
---
# ☁️ AWS DynamoDB: NoSQL 데이터베이스 완벽 가이드

## 1. DynamoDB란 무엇인가?

DynamoDB는 **완전관리형 NoSQL 키-값 및 문서형 데이터베이스 서비스**로, **높은 가용성과 확장성**을 제공하며 **서버 관리 없이** 사용할 수 있는 AWS의 대표적인 서버리스 데이터베이스입니다.

### 주요 특징

| 특징 | 설명 |
|------|------|
| 서버리스 | 인프라 관리 없이 사용 가능 |
| 고가용성 | AWS 리전 및 AZ에 걸쳐 자동 복제 |
| 확장성 | 수평적 확장 지원 (자동 또는 수동) |
| 빠른 응답속도 | 마이크로초 단위 응답 (Single-digit ms) |
| 이벤트 기반 트리거 | Lambda와 연동 가능 |
| 유연한 데이터 구조 | JSON 기반의 문서형 저장 지원 |

---

## 2. DynamoDB의 데이터 모델

### 테이블 (Table)
- DynamoDB는 데이터를 **테이블 단위**로 관리합니다.
- 테이블에는 **기본 키(primary key)**가 반드시 필요합니다.

### 기본 키 구성

1. **단순 기본 키 (Partition Key only)**
   - 하나의 속성만으로 식별
   - 예: `UserID`

2. **복합 기본 키 (Partition Key + Sort Key)**
   - 두 개의 속성 조합으로 유일성 유지
   - 예: `UserID` + `OrderDate`

### 항목 (Item)
- 테이블 내의 개별 데이터 레코드.
- JSON 객체와 유사한 구조로 속성을 가질 수 있음.
- 각 항목은 서로 다른 속성을 가질 수 있음 (스키마 없음).

### 속성 (Attribute)
- 키-값 쌍으로 표현되는 데이터 요소.

```json
{
  "UserID": "user123",
  "Name": "Alice",
  "Age": 30,
  "Hobbies": ["hiking", "reading"]
}
```

---

## 3. DynamoDB의 읽기/쓰기 방식

### 쓰기 모델

- **PutItem**: 항목 삽입 또는 덮어쓰기
- **UpdateItem**: 항목의 일부 속성 갱신
- **DeleteItem**: 항목 삭제

### 읽기 모델

- **GetItem**: 기본 키로 항목 조회
- **Query**: 파티션 키로 여러 항목 검색 (정렬 키 조건 포함 가능)
- **Scan**: 전체 테이블을 순회하며 조건 검색 (비효율적)

---

## 4. 프로비저닝 모델

DynamoDB는 다음 두 가지 용량 모드 중 하나를 선택하여 사용합니다.

### ▶ 프로비저닝된 용량 모드 (Provisioned Mode)
- 사전에 읽기/쓰기 처리량 설정
- 처리량 초과 시 에러 발생 (또는 버스트 크레딧 사용)
- Auto Scaling 기능 지원

### ▶ 온디맨드 모드 (On-Demand Mode)
- 예측 없이 요청 기반 요금
- 트래픽 예측이 어려운 경우 적합
- 초기 테스트, 스타트업에 유리

---

## 5. 인덱스: GSI와 LSI

### ▶ LSI (Local Secondary Index)
- 테이블 생성 시 정의
- 파티션 키는 동일하고, 다른 정렬 키로 인덱싱
- 최대 5개

### ▶ GSI (Global Secondary Index)
- 테이블 생성 후에도 추가 가능
- **완전히 다른 파티션 키 + 정렬 키** 가능
- 최대 20개 (기본 제한)

---

## 6. TTL (Time To Live)

- 항목 단위로 만료 시간을 지정 가능
- 지정된 시간이 지나면 자동 삭제
- 저장 비용 절감 및 데이터 유지 관리에 유리

---

## 7. DynamoDB Streams

- 테이블의 데이터 변경 이벤트 (삽입, 수정, 삭제)를 캡처
- Lambda와 연동하여 비동기 트리거 구성 가능
- 실시간 분석, 복제 등에 활용

```json
{
  "eventName": "INSERT",
  "dynamodb": {
    "Keys": { "UserID": { "S": "user123" } },
    "NewImage": { "Name": { "S": "Alice" }, "Age": { "N": "30" } }
  }
}
```

---

## 8. 보안 및 접근 제어

### IAM 기반 권한 제어
- 테이블 단위 또는 항목 단위 접근 제어 가능
- 예: `dynamodb:GetItem`, `dynamodb:PutItem` 권한

### 암호화
- 서버 측 암호화 지원 (기본/사용자 키 선택 가능)
- 전송 중 암호화 (HTTPS)도 기본 지원

---

## 9. 서버리스 애플리케이션과 통합

- Lambda, API Gateway, DynamoDB 조합은 대표적인 **서버리스 웹앱 아키텍처** 구성
- DynamoDB Streams와 Lambda를 이용한 **비동기 이벤트 기반 처리**

---

## 10. 사용 예제: Node.js로 CRUD

```js
const AWS = require('aws-sdk');
AWS.config.update({ region: 'ap-northeast-2' });

const docClient = new AWS.DynamoDB.DocumentClient();

// Insert
docClient.put({
  TableName: "Users",
  Item: { UserID: "user123", Name: "Alice", Age: 30 }
}, (err, data) => { if (err) console.error(err); });

// Read
docClient.get({
  TableName: "Users",
  Key: { UserID: "user123" }
}, (err, data) => { if (err) console.error(err); else console.log(data.Item); });
```

---

## 11. DynamoDB 비용

| 항목 | 요금 방식 |
|------|-----------|
| 저장 공간 | GB 단위 월별 과금 |
| 읽기/쓰기 요청 | 프로비저닝 or 온디맨드 모드 |
| Streams | 추가 요금 없음 |
| 백업/복구 | 선택적, 추가 과금 |

### 절감 팁

- TTL 사용으로 오래된 데이터 자동 삭제
- LSI는 삭제 불가 → 신중히 설계
- GSI 남용하지 않기 (요금 증가 요인)

---

## 12. DynamoDB vs RDS

| 비교 항목 | DynamoDB | RDS |
|------------|----------|------|
| 구조 | NoSQL | 관계형 |
| 스키마 | 유연 | 엄격 |
| 확장성 | 자동 수평 확장 | 수직 확장 or Read Replica |
| 사용 사례 | IoT, 게임, 세션, 로그 | 전통적인 비즈니스 앱, ERP 등 |

---

## 🔚 결론

DynamoDB는 현대 웹 애플리케이션에서 빠르고 유연한 데이터베이스 요구사항을 충족하기 위해 탄생한 **강력한 서버리스 NoSQL 서비스**입니다. 복잡한 인프라를 신경 쓰지 않고도 글로벌 확장성과 고성능을 갖춘 DB 시스템을 만들 수 있는 AWS의 핵심 서비스 중 하나입니다.

**✅ 추천 사용 시나리오:**
- 대규모 IoT 또는 게임 트래픽 처리
- 사용자 세션 저장
- 서버리스 웹 애플리케이션의 저장소 역할
- 이벤트 기반 스트림 처리
