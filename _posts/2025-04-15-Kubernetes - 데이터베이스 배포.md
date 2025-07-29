---
layout: post
title: Kubernetes - 데이터베이스 배포
date: 2025-04-15 19:20:23 +0900
category: Kubernetes
---
# Kubernetes에서 상태 저장 데이터베이스 배포하기  
## MySQL / PostgreSQL StatefulSet 예제

Kubernetes는 기본적으로 **무상태(stateless) 애플리케이션**에 최적화되어 있습니다.  
하지만 **데이터베이스처럼 데이터를 유지해야 하는 워크로드**를 실행하려면, 몇 가지 특별한 리소스가 필요합니다.

이 글에서는 Kubernetes에서 **MySQL 또는 PostgreSQL을 안전하게 배포**하는 방법을 소개합니다.

---

## ✅ 왜 StatefulSet이 필요한가?

`Deployment`는 무상태 앱에 적합합니다. Pod가 재시작되거나 스케줄링될 때 이름(IP)이 바뀌고, 고정된 스토리지가 없습니다.

→ 하지만 **DB는 다음 조건을 요구**합니다:

- 고정된 Pod 이름과 네트워크 주소
- 지속적인 스토리지 (재시작해도 데이터 보존)
- 순차적 시작과 안정된 ID

> ✅ 이 조건을 만족하는 Kubernetes 리소스가 바로 **StatefulSet**입니다.

---

## ✅ 구성 요소 요약

| 리소스 | 역할 |
|--------|------|
| StatefulSet | 상태 저장 Pod 관리 |
| PersistentVolumeClaim | Pod마다 고유한 볼륨 요청 |
| PersistentVolume (동적/정적) | 실제 스토리지 |
| ConfigMap / Secret | DB 설정값, 비밀번호 주입 |
| Service (Headless) | 각 Pod를 고유하게 접근 |

---

## ✅ 실습: MySQL StatefulSet 배포 예제

### 📁 디렉터리 구조

```
mysql/
├── mysql-config.yaml       # ConfigMap + Secret
├── mysql-service.yaml      # Headless + ClusterIP 서비스
├── mysql-statefulset.yaml  # StatefulSet
```

---

### ① ConfigMap & Secret 정의

```yaml
# mysql-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-config
data:
  MYSQL_DATABASE: mydb
---
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
type: Opaque
stringData:
  MYSQL_ROOT_PASSWORD: mypass
```

---

### ② Service 정의 (Headless + ClusterIP)

```yaml
# mysql-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  clusterIP: None  # Headless 서비스!
  selector:
    app: mysql
  ports:
    - port: 3306
```

→ Headless 서비스는 `mysql-0.mysql.default.svc.cluster.local` 형태의 **개별 DNS 이름**을 부여합니다.

---

### ③ StatefulSet 정의

```yaml
# mysql-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        envFrom:
          - configMapRef:
              name: mysql-config
          - secretRef:
              name: mysql-secret
        ports:
          - containerPort: 3306
        volumeMounts:
          - name: mysql-pv
            mountPath: /var/lib/mysql
  volumeClaimTemplates:
  - metadata:
      name: mysql-pv
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
```

---

## ✅ 전체 적용

```bash
kubectl apply -f mysql-config.yaml
kubectl apply -f mysql-service.yaml
kubectl apply -f mysql-statefulset.yaml
```

```bash
kubectl get pods
kubectl get pvc
kubectl get svc
```

---

## ✅ PostgreSQL도 거의 유사하게 구성

### PostgreSQL 예시 차이점

- 이미지: `postgres:15`
- 설정 ENV: `POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD`
- 마운트 경로: `/var/lib/postgresql/data`

```yaml
containers:
- name: postgres
  image: postgres:15
  env:
    - name: POSTGRES_DB
      value: mydb
    - name: POSTGRES_USER
      value: admin
    - name: POSTGRES_PASSWORD
      valueFrom:
        secretKeyRef:
          name: pg-secret
          key: password
  volumeMounts:
    - name: pg-pv
      mountPath: /var/lib/postgresql/data
```

---

## ✅ 볼륨 확인 및 데이터 유지 확인

```bash
kubectl delete pod mysql-0
```

→ Pod는 다시 생성되지만, 이전 PVC가 유지되므로 데이터도 그대로 남아 있음

---

## ✅ 볼륨 자동 할당 (StorageClass)

클러스터에 동적 프로비저닝을 위한 `StorageClass`가 구성돼 있다면,  
`PersistentVolume`을 따로 정의하지 않아도 `volumeClaimTemplates`가 자동으로 스토리지를 연결합니다.

```bash
kubectl get sc  # 사용 가능한 스토리지 클래스 확인
```

---

## ✅ 모니터링 및 접근

```bash
# mysql Pod 접속
kubectl exec -it mysql-0 -- mysql -u root -p

# postgresql Pod 접속
kubectl exec -it postgres-0 -- psql -U admin
```

---

## ✅ 정리: Deployment vs StatefulSet

| 항목 | Deployment | StatefulSet |
|------|------------|-------------|
| Pod 이름 | 동적 (랜덤) | 고정 (`name-0`, `name-1`) |
| 스토리지 | 공유 가능 | Pod별 고유 스토리지 |
| 시작 순서 | 무관 | 순차적 (0 → 1 → 2...) |
| 네트워크 ID | 랜덤 | 고정 DNS 이름 부여 |
| 사용처 | 웹 서버, API 등 | DB, Kafka, Redis 등 상태 저장 |

---

## ✅ 결론

Kubernetes에서 데이터베이스를 운영하려면 다음 요소가 필수입니다:

- **StatefulSet**: 고정 ID, 순차적 배포
- **PersistentVolumeClaim**: 스토리지 유지
- **Headless Service**: DNS 기반의 Pod 식별
- **Secret/ConfigMap**: 민감한 정보 분리