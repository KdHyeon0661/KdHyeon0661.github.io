---
layout: post
title: DB - DDL, DML, DCL, TCL
date: 2025-02-15 20:20:23 +0900
category: DB
---
# 🧩 SQL 명령어 분류: DDL, DML, DCL 실무 정리

## 1️⃣ DDL (Data Definition Language) — **데이터 구조 정의**

### 📌 정의
DDL은 데이터베이스의 **스키마(구조)**를 정의하거나 변경하는 명령어입니다.  
즉, **테이블, 인덱스, 제약조건, 뷰, 시퀀스 등 객체를 생성·변경·삭제**할 때 사용합니다.

### 🔧 주요 명령어
| 명령어 | 설명 |
|--------|------|
| `CREATE` | 테이블, 뷰, 인덱스 등 새 객체 생성 |
| `ALTER` | 기존 객체의 구조 수정 (컬럼 추가, 삭제 등) |
| `DROP` | 객체 완전 삭제 |
| `TRUNCATE` | 테이블의 모든 데이터 삭제 (롤백 불가, 구조는 유지) |
| `RENAME` | 객체 이름 변경 |

### ✅ 실무 예시
```sql
-- 테이블 생성
CREATE TABLE Employee (
  EmpID INT PRIMARY KEY,
  Name VARCHAR(100),
  Salary DECIMAL(10,2)
);

-- 컬럼 추가
ALTER TABLE Employee ADD HireDate DATE;

-- 테이블 삭제
DROP TABLE Old_Logs;

-- 모든 데이터 초기화 (빠름, 복구 불가)
TRUNCATE TABLE Temp_Import;
```

### 💡 실무 팁
- `ALTER`는 신중하게 사용: 운영 중 컬럼 구조 변경 시 예기치 않은 장애 발생 가능
- `DROP`은 되돌릴 수 없으므로 항상 **백업** 필수
- `TRUNCATE`는 트랜잭션 로그 최소화 → 대량 초기화 시 유용

---

## 2️⃣ DML (Data Manipulation Language) — **데이터 조작**

### 📌 정의
DML은 테이블에 저장된 데이터를 **삽입, 조회, 수정, 삭제**하는 명령어입니다.  
가장 자주 사용하는 SQL 명령어들이 포함되어 있으며, **트랜잭션 제어의 대상**이 됩니다.

### 🔧 주요 명령어
| 명령어 | 설명 |
|--------|------|
| `SELECT` | 데이터 조회 |
| `INSERT` | 새로운 행 추가 |
| `UPDATE` | 기존 데이터 수정 |
| `DELETE` | 데이터 삭제 |

### ✅ 실무 예시
```sql
-- 직원 정보 삽입
INSERT INTO Employee (EmpID, Name, Salary) VALUES (1001, '홍길동', 4000.00);

-- 직원 급여 인상
UPDATE Employee SET Salary = Salary * 1.1 WHERE EmpID = 1001;

-- 퇴사자 삭제
DELETE FROM Employee WHERE EmpID = 1001;

-- 데이터 조회
SELECT * FROM Employee WHERE Salary > 5000;
```

### 💡 실무 팁
- 항상 `WHERE` 조건 주의: WHERE 없이 `UPDATE`, `DELETE` 하면 전체 테이블 영향
- 트랜잭션(`BEGIN`, `COMMIT`, `ROLLBACK`)으로 작업 안전하게 처리
- `MERGE`는 조건에 따라 `INSERT` 또는 `UPDATE`를 실행하는 실무에서 자주 쓰이는 복합 문법

---

## 3️⃣ DCL (Data Control Language) — **접근 제어 및 권한 관리**

### 📌 정의
DCL은 데이터베이스 사용자에 대한 **접근 권한 제어**를 담당합니다.  
주로 DBA나 시스템 관리자들이 사용하며, 보안과 관련된 정책 적용 시 사용됩니다.

### 🔧 주요 명령어
| 명령어 | 설명 |
|--------|------|
| `GRANT` | 특정 권한을 사용자에게 부여 |
| `REVOKE` | 부여한 권한 회수 |

### ✅ 실무 예시
```sql
-- SELECT 권한 부여
GRANT SELECT ON Employee TO user_dev;

-- INSERT, UPDATE 권한 부여
GRANT INSERT, UPDATE ON Orders TO order_manager;

-- 권한 회수
REVOKE INSERT ON Orders FROM user_temp;
```

### 💡 실무 팁
- 보안 민감 업무일수록 **역할(Role)** 기반 권한 설계 권장 (ex. READ_ONLY, DATA_ENTRY)
- `WITH GRANT OPTION`: 권한을 위임할 수 있으므로 부여 시 주의
- 객체 단위 뿐 아니라 **컬럼 단위**, **스키마 단위** 권한 제어도 가능

---

## 🧠 정리 비교

| 구분 | 목적 | 대표 명령어 | 트랜잭션 포함 여부 | 실무 사용 빈도 |
|------|------|--------------|---------------------|------------------|
| DDL | 구조 정의 | CREATE, ALTER, DROP | ❌ (암묵적 COMMIT) | 개발/운영 중간중간 |
| DML | 데이터 조작 | SELECT, INSERT, UPDATE, DELETE | ✅ | **매우 빈번** |
| DCL | 권한 제어 | GRANT, REVOKE | ❌ | 보안·운영관리 시 |

---

## 📌 보너스: TCL (Transaction Control Language)

DML 작업 후 **트랜잭션을 명시적으로 제어**하는 명령어입니다.

| 명령어 | 설명 |
|--------|------|
| `COMMIT` | 변경사항 영구 반영 |
| `ROLLBACK` | 변경사항 취소 |
| `SAVEPOINT` | 중간 지점 설정 |

```sql
BEGIN;
UPDATE Employee SET Salary = Salary + 100 WHERE EmpID = 1001;
ROLLBACK; -- 다시 원상복구
```

---

## ✅ 마무리

> 실무에서 DML은 가장 많이 사용되고,  
> DDL은 구조 변경 시,  
> DCL은 운영 환경에서 사용자 권한 관리 시 사용됩니다.

SQL 명령어는 단순 암기보다 **실제 흐름과 조합**을 이해하는 것이 중요합니다.  
예를 들어:  
> DDL로 테이블 생성 → DCL로 권한 설정 → DML로 데이터 입력/조회 → TCL로 트랜잭션 관리