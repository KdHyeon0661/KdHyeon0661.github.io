---
layout: post
title: Kubernetes - 데이터베이스 배포
date: 2025-04-15 19:20:23 +0900
category: Kubernetes
---
# Kubernetes에서 상태 저장 데이터베이스 배포하기

## 왜 StatefulSet인가

`Deployment`는 무상태 앱에 적합하다. 반면 DB는 다음을 요구한다.

- **안정적인 네트워크 ID**: Pod 재생성 시에도 예측 가능한 이름/DNS  
- **지속 스토리지**: Pod 재스케줄링·재시작에도 데이터 보존  
- **순차적 롤아웃/롤백**: 데이터 정합성 보장을 위한 시작/종료 순서

이 요구사항을 만족하는 쿠버네티스 리소스가 **StatefulSet**이다.

---

## 구성 요소 개요

| 리소스 | 역할 | 핵심 포인트 |
|---|---|---|
| StatefulSet | 상태 저장 Pod 관리 | 고정 Pod 이름(`name-0`), 순차 롤링, `volumeClaimTemplates` |
| PVC / PV | Pod별 고유 스토리지 | `RWO` 기본, 동적 프로비저닝(StorageClass) 권장 |
| Service(Headless) | 개별 Pod DNS | `clusterIP: None` → `mysql-0.mysql.svc` 등 고정 FQDN |
| Service(ClusterIP) | 내/외부 접근 엔드포인트 | 애플리케이션이 접속할 단일 VIP 제공 |
| ConfigMap / Secret | 설정과 자격분리 | 비민감/민감 값 분리, 마운트 또는 ENV |
| PDB / HPA / VPA | 가용성/자원 자동화 | 유지 가능 복제수, 수평/수직 조정 |
| NetworkPolicy | 네트워크 보안 | DB 접근 허용 범위 최소화 |

---

## 스토리지 설계 원칙

- **StorageClass + 동적 프로비저닝**을 사용한다. 멀티 존/토폴로지에는 `volumeBindingMode: WaitForFirstConsumer`.
- 접근 모드: 단일 인스턴스 DB는 대부분 **RWO**. 다중 라이터 공유는 파일 스토리지(RWX)가 필요하나, DB 데이터 파일에 RWX 공유는 권장하지 않는다.
- **리클레임 정책**: 운영 데이터는 `Retain`을 선호(수동 파기). 테스트는 `Delete`.
- 퍼미션: `securityContext.fsGroup` / `runAsUser`로 마운트 권한 문제를 예방.
- 스냅샷/백업 전략을 초기에 포함(스토리지 스냅샷 + binlog/WAL 아카이브).

---

## 디렉터리 구조(예시)

```
db/
├── mysql/
│   ├── mysql-config.yaml
│   ├── mysql-service.yaml
│   └── mysql-statefulset.yaml
└── postgres/
    ├── pg-config.yaml
    ├── pg-service.yaml
    └── pg-statefulset.yaml
```

---

## 공통 Service 패턴

Headless(개별 Pod 주소) + ClusterIP(애플리케이션이 붙는 VIP) 이중화가 실전에서 편리하다.

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: mysql-headless
  labels: { app: mysql }
spec:
  clusterIP: None                # Headless Service
  selector: { app: mysql }
  ports:
  - name: mysql
    port: 3306
---
apiVersion: v1
kind: Service
metadata:
  name: mysql
  labels: { app: mysql }
spec:
  selector: { app: mysql }
  ports:
  - name: mysql
    port: 3306
```

Headless로 각 Pod가 `mysql-0.mysql-headless.svc` 처럼 고정 FQDN을 가진다. 애플리케이션은 보통 `mysql.svc:3306`(ClusterIP)를 사용한다.

---

## MySQL: StatefulSet 단일 인스턴스(학습/PoC 기준)

### ConfigMap & Secret

```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-config
data:
  MYSQL_DATABASE: mydb
  MYSQL_USER: appuser
  MYSQL_CHARSET: utf8mb4
  MYSQL_COLLATION: utf8mb4_0900_ai_ci
---
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
type: Opaque
stringData:
  MYSQL_ROOT_PASSWORD: "changeme-root"
  MYSQL_PASSWORD: "changeme-app"
```

### StatefulSet(프로브·보안·리소스 포함)

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql-headless
  replicas: 1
  selector: { matchLabels: { app: mysql } }
  podManagementPolicy: OrderedReady
  template:
    metadata:
      labels: { app: mysql }
    spec:
      securityContext:
        fsGroup: 999                      # mysql(데몬) 그룹과 정합되도록 환경에 맞추어 조정
      containers:
      - name: mysql
        image: mysql:8.0
        imagePullPolicy: IfNotPresent
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom: { secretKeyRef: { name: mysql-secret, key: MYSQL_ROOT_PASSWORD } }
        - name: MYSQL_DATABASE
          valueFrom: { configMapKeyRef: { name: mysql-config, key: MYSQL_DATABASE } }
        - name: MYSQL_USER
          valueFrom: { configMapKeyRef: { name: mysql-config, key: MYSQL_USER } }
        - name: MYSQL_PASSWORD
          valueFrom: { secretKeyRef: { name: mysql-secret, key: MYSQL_PASSWORD } }
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
        - name: conf
          mountPath: /etc/mysql/conf.d
        readinessProbe:
          exec:
            command: ["sh","-c","mysqladmin ping -uroot -p$MYSQL_ROOT_PASSWORD"]
          initialDelaySeconds: 15
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 10
        livenessProbe:
          exec:
            command: ["sh","-c","mysqladmin ping -uroot -p$MYSQL_ROOT_PASSWORD"]
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
        resources:
          requests:
            cpu: "250m"
            memory: "512Mi"
          limits:
            cpu: "1"
            memory: "1Gi"
      volumes:
      - name: conf
        configMap:
          name: mysql-config
          items:
          - key: MYSQL_CHARSET
            path: charset.ignore           # 예시: 실제 my.cnf 생성은 initContainer에서 처리 가능
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      storageClassName: standard          # 환경의 StorageClass 이름
      accessModes: ["ReadWriteOnce"]
      resources:
        requests: { storage: 10Gi }
```

### MySQL용 my.cnf 커스터마이징(선택)

`initContainer`로 `my.cnf`를 구성해 `/etc/mysql/conf.d/`에 투입하면 일관성이 높다.

```yaml
initContainers:
- name: init-mycnf
  image: busybox:1.36
  command: ["sh","-c","cat >/workdir/my.cnf <<'EOF'\n[mysqld]\ncharacter-set-server=utf8mb4\ncollation-server=utf8mb4_0900_ai_ci\ninnodb_flush_log_at_trx_commit=1\nsync_binlog=1\nEOF\ncp /workdir/my.cnf /etc/mysql/conf.d/my.cnf"]
  volumeMounts:
  - name: conf
    mountPath: /etc/mysql/conf.d
  - name: tmp
    mountPath: /workdir
volumes:
- name: tmp
  emptyDir: {}
```

---

## PostgreSQL: StatefulSet 단일 인스턴스(학습/PoC 기준)

### ConfigMap & Secret

```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: pg-config
data:
  POSTGRES_DB: mydb
  POSTGRES_USER: appuser
  PGDATA_SUBDIR: pgdata
---
apiVersion: v1
kind: Secret
metadata:
  name: pg-secret
type: Opaque
stringData:
  POSTGRES_PASSWORD: "changeme-app"
```

### StatefulSet(프로브·WAL 옵션 예시)

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: pg-headless
  replicas: 1
  selector: { matchLabels: { app: postgres } }
  template:
    metadata:
      labels: { app: postgres }
    spec:
      securityContext:
        fsGroup: 999                       # postgres 사용자/그룹과 정합
      containers:
      - name: postgres
        image: postgres:15
        env:
        - name: POSTGRES_DB
          valueFrom: { configMapKeyRef: { name: pg-config, key: POSTGRES_DB } }
        - name: POSTGRES_USER
          valueFrom: { configMapKeyRef: { name: pg-config, key: POSTGRES_USER } }
        - name: POSTGRES_PASSWORD
          valueFrom: { secretKeyRef: { name: pg-secret, key: POSTGRES_PASSWORD } }
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        ports:
        - containerPort: 5432
          name: postgres
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
        readinessProbe:
          exec:
            command: ["sh","-c","pg_isready -U \"$POSTGRES_USER\" -d \"$POSTGRES_DB\" -h 127.0.0.1 -p 5432"]
          initialDelaySeconds: 15
          periodSeconds: 5
          failureThreshold: 10
        livenessProbe:
          exec:
            command: ["sh","-c","pg_isready -U \"$POSTGRES_USER\" -d \"$POSTGRES_DB\" -h 127.0.0.1 -p 5432"]
          initialDelaySeconds: 30
          periodSeconds: 10
        resources:
          requests:
            cpu: "250m"
            memory: "512Mi"
          limits:
            cpu: "1"
            memory: "1Gi"
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      storageClassName: standard
      accessModes: ["ReadWriteOnce"]
      resources:
        requests: { storage: 10Gi }
```

`postgresql.conf`나 `pg_hba.conf`는 ConfigMap + initContainer로 반영하거나, 전용 이미지를 빌드해 관리한다.

---

## 배포 및 확인

```bash
kubectl apply -f mysql/mysql-config.yaml
kubectl apply -f mysql/mysql-service.yaml
kubectl apply -f mysql/mysql-statefulset.yaml

kubectl apply -f postgres/pg-config.yaml
kubectl apply -f postgres/pg-service.yaml
kubectl apply -f postgres/pg-statefulset.yaml

kubectl get pods -l app=mysql
kubectl get pvc -l app=mysql
kubectl get svc | egrep 'mysql|pg'
```

접속 테스트:

```bash
# MySQL
kubectl exec -it sts/mysql -- sh -c 'mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -e "select version();"'

# PostgreSQL
kubectl exec -it sts/postgres -- sh -c 'psql -U "$POSTGRES_USER" -d "$POSTGRES_DB" -c "select version();"'
```

---

## 데이터 영속성 확인

```bash
# 테이블과 데이터 생성
kubectl exec -it mysql-0 -- sh -c 'mysql -uroot -p$MYSQL_ROOT_PASSWORD -e "create table if not exists mydb.t(t int); insert into mydb.t values(1); select * from mydb.t;"'

# Pod 삭제 후 재생성
kubectl delete pod mysql-0

# 데이터 확인
kubectl exec -it mysql-0 -- sh -c 'mysql -uroot -p$MYSQL_ROOT_PASSWORD -e "select * from mydb.t;"'
```

PVC가 유지되므로 데이터가 남아 있어야 한다.

---

## 운영 필수 옵션

### 리소스 요청/제한
- I/O 지연을 방지하기 위해 최소 **메모리 요청**을 충분히 준다.
- DB는 OOM에 민감하므로 `limits.memory`와 엔진 설정(버퍼/워크 메모리)을 함께 조정한다.

### 보안 컨텍스트
- `runAsUser`/`fsGroup`/`runAsNonRoot` 사용. 퍼미션 오류가 잦다.
- Secret에는 DB 비밀번호, 인증서 등 민감 정보를 담고 RBAC로 제한한다.

### 프로브
- `readinessProbe`: 트래픽 수신 가능 상태 확인
- `livenessProbe`: 적절히 보수적으로, 복구 루프 방지
- DB 부팅 시간이 길 수 있으니 초기 지연/실패 임계치를 충분히 높인다.

### PDB(PodDisruptionBudget)
강제 축출 시 최소 가용성을 보장한다.

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: mysql-pdb
spec:
  minAvailable: 1
  selector:
    matchLabels: { app: mysql }
```

### 안티어피니티/토폴로지 분산
복제본 운용 시 노드/존 간 분산을 강제한다.

```yaml
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchLabels: { app: mysql }
      topologyKey: "kubernetes.io/hostname"
```

---

## 백업·복구 전략

### 스토리지 스냅샷
CSI 스냅샷 CRDs가 있다면 정지점 확보가 쉽다.

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata: { name: csi-snapclass }
driver: ebs.csi.aws.com
deletionPolicy: Delete
---
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata: { name: mysql-data-snap }
spec:
  volumeSnapshotClassName: csi-snapclass
  source:
    persistentVolumeClaimName: data-mysql-0   # 실제 PVC 이름
```

스냅샷에서 새 PVC 생성:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-from-snap
spec:
  storageClassName: standard
  dataSource:
    name: mysql-data-snap
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes: ["ReadWriteOnce"]
  resources:
    requests: { storage: 10Gi }
```

### MySQL: binlog + 논리/물리 백업
- `sync_binlog=1`, `innodb_flush_log_at_trx_commit=1`로 크래시 안전성 향상.
- 증분 복구를 위해 **binlog 보관** 또는 Percona XtraBackup 등을 고려.
- 주기적 `mysqldump`는 긴 락/성능 저하 가능성(규모에 따라 물리 백업 권장).

### PostgreSQL: WAL 아카이브 + 베이스백업
- `archive_mode=on`, `archive_command`로 외부 스토리지에 WAL 보관.
- `pg_basebackup`로 베이스백업 + WAL 재생으로 시점 복구.

---

## 스케일과 고가용성

### 수평 확장
단일 인스턴스 RDBMS는 일반적으로 **읽기 복제본**으로 수평 확장한다(쓰기 스케일은 샤딩/분할 필요).

- MySQL: `mysqld` Primary/Replica 구성을 스크립트 또는 오퍼레이터로 관리
- PostgreSQL: 스트리밍 리플리케이션(복제 슬롯/타임라인 관리)

### 오퍼레이터 활용
프로비저닝·페일오버 자동화·백업 스케줄링이 필요하면 데이터베이스 오퍼레이터(예: MySQL/PG 전용 오퍼레이터, 또는 헬름 차트)를 검토한다. 자체 구현 대비 운영 복잡도를 크게 줄인다.

---

## 장애 주입과 점검 루틴

```bash
# 노드 축출 시 DB 가용성 관찰
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data

# Pod 강제 재시작
kubectl delete pod mysql-0

# PVC 바인딩 및 이벤트 확인
kubectl describe pvc data-mysql-0 | sed -n '/Events/,$p'
kubectl get events --sort-by=.lastTimestamp | tail -n 30

# 파일시스템/퍼미션 점검
kubectl exec -it mysql-0 -- sh -c 'id; df -h; ls -al /var/lib/mysql'
```

자주 보는 실패 패턴과 처치:

| 증상 | 원인 | 조치 |
|---|---|---|
| ContainerCreating에서 정지 | 볼륨 마운트 실패/CSI 오류 | CSI 컨트롤러 로그, IAM/네트워크, StorageClass 확인 |
| 퍼미션 에러(EACCES) | fsGroup/runAsUser 미설정 | `securityContext` 설정, 이미지 사용자/UID 확인 |
| PVC Pending 지속 | SC 없음/접근 모드 불일치 | `kubectl get sc`, PVC/SC 정합성 확인(RWO/RWX) |
| 프로브 반복 실패 | 초기 지연 값 부족/DB 부팅 지연 | `initialDelaySeconds` 상향, 스토리지 I/O 확인 |
| 디스크 부족 | PVC 용량 과소 | `allowVolumeExpansion` 활성화, PVC 확장 |

---

## 전천후 템플릿: Primary(쓰기) + Replica(읽기) 토폴로지(개요)

상세 구현은 DB 엔진/오퍼레이터별로 상이하므로 개념 템플릿만 요약한다.

- Headless + ClusterIP 2종
  - `db-read`: Replica용 서비스(Selector로 Replica만 포함)
  - `db-write`: Primary용 서비스(Primary Pod 라벨 선택)
- StatefulSet 2개 또는 1개 + 역할 라벨링
- 페일오버: 오퍼레이터/플로팅 라벨/사이드카 스크립트로 역할 전환

애플리케이션은 **쓰기 연결**은 `db-write`, **읽기 연결**은 `db-read`로 분리해 트래픽을 유연하게 분배한다.

---

## 운영 체크리스트

- 스토리지
  - StorageClass, `WaitForFirstConsumer`, 리클레임 정책
  - IOPS/지연 요건과 디스크 클래스 매칭
- 보안
  - Secret/RBAC/NetworkPolicy, `runAsNonRoot`, `fsGroup`
- 가용성
  - PDB, 안티어피니티/토폴로지 분산, 노드 업그레이드 전략
- 관측
  - 메트릭(커넥션, 캐시히트, 슬로우쿼리), 로그, PVC 이벤트 알림
- 백업/복구
  - 스냅샷 주기, 논리/물리 백업, DR 리허설
- 업그레이드
  - 엔진 마이너/메이저 업그레이드 시 호환성 및 롤백 계획
- 비용
  - 스토리지 클래스/크기/스냅샷 보존 정책 최적화

---

## 결론 요약

- **StatefulSet + Headless Service + PVC**는 쿠버네티스에서 DB를 안정적으로 운영하기 위한 기본 3요소다.
- 스토리지 정책(접근 모드·리클레임·토폴로지)과 보안/프로브/리소스 설정은 초기 설계에서 반드시 결정한다.
- 백업/복구(스냅샷 + binlog/WAL)와 가용성 전략(PDB/안티어피니티/오퍼레이터)은 운영 탄력성을 좌우한다.
- 단일 인스턴스에서 출발하되, 읽기 복제본·오퍼레이터 기반 자동화를 단계적으로 도입해 성숙도를 높인다.