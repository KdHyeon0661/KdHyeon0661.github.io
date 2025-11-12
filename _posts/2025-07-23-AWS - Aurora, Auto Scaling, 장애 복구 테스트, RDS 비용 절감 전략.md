---
layout: post
title: AWS - Aurora, Auto Scaling, 장애 복구 테스트, RDS 비용 절감 전략
date: 2025-07-23 16:20:23 +0900
category: AWS
---
# Amazon Aurora, Auto Scaling, 장애 복구 테스트, RDS 비용 절감 전략

## 0. 한 장 요약

- **Aurora**는 **컴퓨트(인스턴스)**와 **분산 스토리지(6-way, 3AZ)**를 분리한 **관리형 RDB**로, **고성능/고가용/확장성**을 제공한다.  
- **Auto Scaling**은 **Serverless v2(세밀한 ACU 스케일)**, **리드 리플리카 자동 증감**, **글로벌 읽기** 등으로 달성한다.  
- **장애 복구**는 **Multi-AZ + 자동 Failover**가 기본. **게임데이**로 **시나리오형 복구 테스트**를 정례화한다.  
- **비용 최적화**는 **인스턴스/스토리지/백업/데이터 전송** 축으로 수립하고, **RI/Serverless/Lifecycle/관측성 기반 다운사이징**으로 체질화한다.

---

## 1. Amazon Aurora란 무엇인가? (초안 보강)

### 1.1 개요 & 특징

| 항목 | 설명 |
|---|---|
| 아키텍처 | **컴퓨트**(Writer/Reader)와 **다중 AZ 분산 스토리지**(6중 복제, 3AZ) 분리 |
| 성능 | 엔진 최적화로 **MySQL 대비 최대 수배, PostgreSQL 대비 수배** 수준 목표(작업부하 의존) |
| 가용성 | 페이지 단위 복구·쿼럼 쓰기·자동 장애 감지/Failover |
| 확장성 | **Reader 최대 15개**(엔진/버전에 따라 상이), **스토리지 최대 128TB 자동 확장** |
| 백업 | 지속 백업(증분), 포인트인타임 복구(PITR), 스냅샷(수동) |
| 보안 | VPC, KMS 암호화, IAM 인증(토큰), TDE 유사(서버측 암호화) |
| 호환 | **MySQL-Compatible**, **PostgreSQL-Compatible** 에디션 |

> **핵심 관점**: Aurora는 “**스토리지 엔진을 서비스화**”했기 때문에 로그/체크포인트/복구 경로가 전통적인 RDB보다 **더 빠르고 탄력적**이다.

### 1.2 Aurora 클러스터 구성 요소

| 컴포넌트 | 역할 |
|---|---|
| Cluster(엔드포인트) | `Writer`, `Reader`, `Cluster Endpoint`, `Custom Endpoint` |
| DB Instance | Writer(Primary), Reader(Replica) |
| Storage Volume | 10GB 단위 자동 증설(최대 128TB), 6-way 복제 |
| Endpoints | `cluster-endpoint`: 읽기/쓰기, `reader-endpoint`: 읽기 분산, `custom-endpoint`: 특정 서브셋 묶음 |

---

## 2. Auto Scaling (서버리스 v2, 리플리카, 글로벌)

### 2.1 Aurora Serverless v2 (세밀 탄력)

- **초 단위**로 **ACU(Aurora Capacity Unit)**를 조정하여 vCPU/메모리를 **부하에 맞추어 축소/확장**.  
- **프로비저닝 없는** 운영(최소/최대 ACU 범위 지정) → 유휴시 비용 절감.

**핵심 설정 (콘솔/CLI 개념)**  
- 최소 ACU / 최대 ACU 범위 지정  
- 트래픽 급증 시 연속/점진 확장, 감소 시 일정 지연 후 축소

```bash
# (개념) 서버리스 v2 클러스터 파라미터 예시
aws rds create-db-cluster \
  --db-cluster-identifier aurora-slv2-demo \
  --engine aurora-mysql \
  --engine-version <호환 버전> \
  --scaling-configuration MinCapacity=2,MaxCapacity=64,AutoPause=false \
  --master-username admin --master-user-password '***'
```

> 주: 실제 파라미터 명/지원값은 엔진/버전에 따라 다르므로 배포 전 콘솔/공식 문서로 확인.

### 2.2 Reader Auto Scaling

- **리드 리플리카 개수**를 CloudWatch ALB/탐색 요청 지표 기반으로 **자동 증감**.  
- 엔드포인트 `reader-endpoint`로 **읽기 부하 분산**.

**Terraform 스니펫(개념)**
```hcl
resource "aws_appautoscaling_target" "aurora_readers" {
  max_capacity       = 10
  min_capacity       = 1
  resource_id        = "cluster:aurora-mycluster"   # 형식 유의
  scalable_dimension = "rds:cluster:ReadReplicaCount"
  service_namespace  = "rds"
}
resource "aws_appautoscaling_policy" "aurora_scale_out" {
  name               = "aurora-reader-scale-out"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.aurora_readers.resource_id
  scalable_dimension = aws_appautoscaling_target.aurora_readers.scalable_dimension
  service_namespace  = aws_appautoscaling_target.aurora_readers.service_namespace
  target_tracking_scaling_policy_configuration {
    predefined_metric_specification {
      predefined_metric_type = "RDSReaderAverageCPUUtilization"
    }
    target_value = 60
  }
}
```

### 2.3 Aurora Global Database (전세계 읽기/재해 리전)

- **Primary Region**의 로그 블록을 **비동기**로 서브 리전 **Secondary**에 전파 → **cross-region 읽기 지연 최소화**.  
- **리전 장애 시** Secondary를 **승격(Promote)**하여 쓰기 가능한 독립 클러스터로 전환(수동/자동 전환 시나리오 수립).

---

## 3. 연결/성능 운영 (실전 팁)

### 3.1 RDS Proxy

- Lambda/서버리스/대규모 연결 수 **스파이크** 대응  
- **커넥션 풀/재활용**로 Aurora 보호, 콜드스타트/오버헤드 감소

```bash
# RDS Proxy 생성(개념)
aws rds create-db-proxy \
  --db-proxy-name app-proxy \
  --engine-family MYSQL \
  --auth 'AuthScheme=SECRETS' \
  --role-arn arn:aws:iam::123:role/rdsproxy-role \
  --vpc-subnet-ids subnet-1 subnet-2
```

### 3.2 세션/풀 전략

- **짧은 트랜잭션** 유지(긴 트랜잭션은 Failover 지연/락 확장 리스크)  
- ORM 커넥션 풀 크기를 **작게 시작 → 관측성으로 점증**  
- **읽기/쓰기 분리**: 읽기는 `reader-endpoint`로 라우팅

### 3.3 파라미터 튜닝

- **innodb_buffer_pool_size(유사)**, **work_mem(포스트그레)** 등 **워크로드 프로파일**에 맞게 조정  
- Autovacuum(포스트그레) 설정/모니터링  
- 쿼리 플랜 고정/힌트는 최소화, **지표 기반 인덱스 최적화**

---

## 4. 장애 복구(Failover) 테스트 (게임데이 Runbook 포함)

### 4.1 장애 시 처리 흐름 (초안 정리 + 보강)

| 구성 | 동작 |
|---|---|
| RDS Multi-AZ (기존) | 스탠바이로 자동 승격, DNS 엔드포인트 전환(분 단위) |
| Aurora Cluster | Reader 승격(수초~수십초), **스토리지 독립**으로 빠른 복구 목표 |
| Global Database | 2차 리전 승격(Promote)로 DR 수행 |

### 4.2 테스트 명령(의도적 Failover)

```bash
# Aurora 기본 인스턴스 강제 Failover(Writer 재시작)
aws rds reboot-db-instance \
  --db-instance-identifier my-aurora-writer \
  --force-failover
```

**검증 포인트**
- 애플리케이션 에러율/대기시간 스파이크  
- 커넥션 풀 **자동 재연결** 여부  
- 트랜잭션 재시도/멱등성(Idempotency) 구현 확인

### 4.3 게임데이 Runbook (샘플)

1) **사전 조건**  
   - IaC 상태 최신화, 최근 스냅샷, 알림 채널(Slack/PagerDuty) 점검  
2) **시나리오**  
   - S1: Writer 인스턴스 장애 → Reader 승격  
   - S2: AZ 장애 모의 → 서브넷/라우팅 격리(비가역 작업은 금지)  
   - S3: 리전 격리 → Global DB Secondary 승격(Promote)  
3) **관측/지표**  
   - DB 연결 실패율, 평균/95p 레이턴시, Error Budget 소비량  
   - CloudWatch: `DatabaseConnections`, `Deadlocks`, `FreeLocalStorage`  
4) **Roll-back**  
   - 원복(Writer 재배치/Global 역할 복귀), 파라미터/커넥션 풀 정상화  
5) **포스트모템**  
   - 타임라인/감지~복구 시간, 인프라/애플리케이션 조치 항목 백로그화

---

## 5. 백업/복구/클로닝/백트래킹

### 5.1 백업/스냅샷

- **지속 백업**: 보존기간 내 **PITR**  
- **스냅샷**: 수동/공유/카탈로그화

```bash
aws rds create-db-snapshot \
  --db-snapshot-identifier snap-2025-11 \
  --db-instance-identifier my-aurora-writer
```

### 5.2 복구

```bash
aws rds restore-db-cluster-from-snapshot \
  --db-cluster-identifier my-restore \
  --snapshot-identifier snap-2025-11 \
  --engine aurora-mysql
```

### 5.3 Fast Cloning / Backtrack(엔진/버전 의존)

- **Fast Database Cloning**: 스토리지 레벨 **Copy-on-Write**로 **대용량 즉시 클론**(테스트/리포팅)  
- **Backtrack(일부 MySQL 호환)**: **시점 되감기**(DDL/엔진 제약 확인)

---

## 6. 보안/컴플라이언스

### 6.1 암호화/네트워크

- **KMS 암호화**(at-rest), **TLS(in-transit)**  
- 전용 **서브넷 그룹**, 보안그룹 최소 허용  
- **프라이빗 엔드포인트** 우선

### 6.2 인증/권한

- **IAM 데이터베이스 인증**(토큰 기반, 유효기간 짧음)  
- 최소 권한 원칙, **비밀은 Secrets Manager**에 저장/로테이션

### 6.3 감사/감사로그

- **CloudTrail**(제어면), DB Audit 로그(데이터면)  
- 장기 보관 시 **S3 + Object Lock(Compliance/Governance)** 조합

---

## 7. 관측성(Observability)

### 7.1 CloudWatch 지표(샘플)

| 지표 | 의미/임계값 아이디어 |
|---|---|
| `CPUUtilization` | 70~80% 지속 시 스케일 아웃/업 고려 |
| `FreeableMemory` | 급강하 시 쿼리 버스트/정렬/캐시 압박 |
| `DatabaseConnections` | 애플리케이션 커넥션 전략 점검 |
| `AuroraReplicaLag` | 리드 일관성/지연 |
| `Deadlocks` | 트랜잭션 경합/락 순서 문제 |
| `DiskQueueDepth` | 스토리지 압박 시 신호 |

### 7.2 퍼포먼스 관점 SQL 점검(예: PostgreSQL)

```sql
-- 느린 쿼리 확인(Extension/pg_stat_statements 가정)
SELECT query, calls, total_time, mean_time
FROM pg_stat_statements
ORDER BY total_time DESC
LIMIT 20;
```

```sql
-- Autovacuum 대상 테이블 후보
SELECT relname, n_live_tup, n_dead_tup
FROM pg_stat_all_tables
ORDER BY n_dead_tup DESC
LIMIT 20;
```

---

## 8. 비용 모델과 최적화 (수식 포함)

### 8.1 근사비용 수식

$$
\text{Monthly Cost} \approx
\sum_i (H_i \cdot r_i) +
\sum_s (G_s \cdot p_s) +
\sum_b (G_b \cdot p_b) +
\sum_x (E_x \cdot d_x)
$$

- \(H_i\): 인스턴스 \(i\)의 사용 시간(시간), \(r_i\): 시간당 요금  
- \(G_s\): 일반 스토리지(GB·월), \(p_s\): 단가  
- \(G_b\): 백업 스토리지(GB·월), \(p_b\): 단가  
- \(E_x\): egress(GB), \(d_x\): GB당 전송 단가

### 8.2 절감 전략(초안 정리 + 보강)

| 전략 | 요지 | 팁 |
|---|---|---|
| **Reserved Instances** | 1/3년 약정 시 큰 할인 | 트래픽 예측 가능 워크로드 |
| **Serverless v2** | 유휴 시간 **미과금**, 초 단위 탄력 | 최소/최대 ACU 범위 설계 |
| **다운사이징** | CPU < 20~30% 지속이면 등급↓ | 관측성 기반, 성능 회귀 테스트 |
| **스토리지 관리** | 오래된 스냅샷/백업 정리 | 스냅샷 보존 정책/태그 운영 |
| **리드 최적화** | 읽기는 리플리카/캐시로 | API/레포트 읽기 분리 |
| **데이터 전송 절감** | 동일 리전에 붙이기 | VPC 내부 호스팅, 프라이빗 |
| **쿼리 최적화** | I/O/CPU/메모리 낭비 제거 | 인덱스/실행계획/배치작업창 설정 |

**CLI 예시: 스냅샷 정리**
```bash
aws rds describe-db-snapshots --query 'DBSnapshots[?SnapshotCreateTime<`2024-11-10`].[DBSnapshotIdentifier]' --output text |
xargs -n1 -I{} aws rds delete-db-snapshot --db-snapshot-identifier {}
```

**인스턴스 등급 변경(즉시 적용 예)**
```bash
aws rds modify-db-instance \
  --db-instance-identifier mydb \
  --db-instance-class db.r6g.large \
  --apply-immediately
```

---

## 9. IaC & 배포 파이프라인

### 9.1 Terraform (개념 요약: Aurora 클러스터 + 파라미터 그룹)

```hcl
provider "aws" { region = "ap-northeast-2" }

resource "aws_rds_cluster" "aur" {
  cluster_identifier      = "aurora-mysql-demo"
  engine                  = "aurora-mysql"
  master_username         = "admin"
  master_password         = "SuperSecret!"
  backup_retention_period = 7
  preferred_backup_window = "17:00-18:00"
  database_name           = "appdb"
  storage_encrypted       = true
  kms_key_id              = "arn:aws:kms:ap-northeast-2:123:key/abcd"
}

resource "aws_rds_cluster_instance" "writer" {
  identifier         = "aurora-writer-1"
  cluster_identifier = aws_rds_cluster.aur.id
  instance_class     = "db.r6g.large"
  engine             = aws_rds_cluster.aur.engine
}

resource "aws_rds_cluster_instance" "reader" {
  count              = 2
  identifier         = "aurora-reader-${count.index}"
  cluster_identifier = aws_rds_cluster.aur.id
  instance_class     = "db.r6g.large"
  engine             = aws_rds_cluster.aur.engine
}
```

### 9.2 GitHub Actions (개념: 마이그레이션 + 배포)

```yaml
name: DB Migrate & Deploy
on: { push: { branches: [ "main" ] } }
jobs:
  migrate-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123:role/oidc-rds-deploy
          aws-region: ap-northeast-2
      - name: Run migrations
        run: |
          alembic upgrade head   # or flyway/liquibase
      - name: Terraform apply
        run: |
          terraform init && terraform apply -auto-approve
```

---

## 10. 마이그레이션/호환/개발자 경험

- **호환성**: 기존 **MySQL/PostgreSQL 드라이버/ORM** 그대로 사용 가능(버전 제약 확인).  
- **스키마 관리**: Flyway/Liquibase/Alembic으로 **버전형 마이그레이션** 표준화.  
- **Zero-Downtime**: 읽기 복제본 **스위치 오버** + Blue/Green 배포 조합.

---

## 11. 트러블슈팅 & 체크리스트

### 11.1 자주 겪는 이슈

| 증상 | 원인 | 해결 |
|---|---|---|
| Failover 길어짐 | 긴 트랜잭션/락 경합 | 트랜잭션 짧게, 배치 윈도, 적절 격리수준 |
| Replica Lag 증가 | 대형 트랜잭션/인덱스 부재 | 배치 쪼개기, 인덱스/플랜 최적화 |
| 연결 폭증 | 서버리스/함수형 급증 | **RDS Proxy**, 커넥션 풀, KeepAlive |
| 비용 급증 | 스냅샷 누적/서버 과스펙 | 스냅샷 정리, 다운사이징, Serverless |
| 쓰기 병목 | Hot row/PK 경합 | 샤딩키/파티션/쓰기 패턴 개선 |

### 11.2 운용 체크리스트

- [x] Multi-AZ(Aurora는 기본 다중AZ 스토리지) + 리플리카 구성  
- [x] 장애 복구 **게임데이 분기별 1회**  
- [x] RDS Proxy + 커넥션 풀 설정  
- [x] 백업 보존/스냅샷 수명주기 정책  
- [x] KMS 암호화 + TLS 강제  
- [x] 관측성 대시보드(지표/로그/쿼리)  
- [x] 비용 대시보드(스토리지/스냅샷/전송/인스턴스)  
- [x] IaC 일원화 + 리뷰/승인 파이프라인

---

## 12. 실습: 읽기/쓰기 라우팅 & 간단 부하 테스트

### 12.1 애플리케이션 라우팅(Python 예)

```python
import pymysql
import os

WRITER = os.environ["WRITER_ENDPOINT"]      # cluster.endpoint
READER = os.environ["READER_ENDPOINT"]      # reader.endpoint

def get_conn(writer=False):
    host = WRITER if writer else READER
    return pymysql.connect(host=host, user="app", password="***", db="appdb", connect_timeout=3)

def create_order(user_id, amount):
    with get_conn(writer=True) as conn:
        with conn.cursor() as cur:
            cur.execute("INSERT INTO orders(user_id, amount) VALUES(%s,%s)", (user_id, amount))
        conn.commit()

def get_orders(user_id, limit=50):
    with get_conn(writer=False) as conn:
        with conn.cursor() as cur:
            cur.execute("SELECT id, amount FROM orders WHERE user_id=%s ORDER BY id DESC LIMIT %s", (user_id, limit))
            return cur.fetchall()
```

### 12.2 간단 부하(동시 읽기)

```bash
# 100 동시 GET 호출 시뮬레이션(예: wrk)
wrk -t8 -c100 -d30s http://api.example.com/orders?user=42
```

관찰: `reader-endpoint`가 리드 트래픽을 분산하는지, 레이턴시/CPU/연결 지표를 체크.

---

## 13. 마무리 요약 (초안 + 보강)

| 주제 | 핵심 |
|---|---|
| Aurora 아키텍처 | 컴퓨트/스토리지 분리, 3AZ·6중 복제 |
| Auto Scaling | **Serverless v2**(ACU), **Reader Auto Scaling**, **Global** |
| 장애 복구 | Failover 자동화 + **게임데이**로 정기 검증 |
| 연결/성능 | **RDS Proxy**, 짧은 트랜잭션, 읽기/쓰기 분리 |
| 보안/컴플라이언스 | KMS/TLS/IAM/Secrets Manager/Object Lock(로그 보존) |
| 관측/최적화 | CloudWatch/쿼리 관측 + 다운사이징/RI/스냅샷 정리 |
| IaC/배포 | Terraform/SAM + CI/CD, Blue/Green |

이 가이드를 **당신의 리전/엔진 버전/트래픽 패턴**에 맞게 치환하면, **고성능·고가용·비용효율**이 균형 잡힌 Aurora 운영 표준을 곧바로 구축할 수 있다.