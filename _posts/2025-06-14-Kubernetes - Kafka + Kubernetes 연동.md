---
layout: post
title: Kubernetes - 실습 : Kafka + Kubernetes 연동
date: 2025-06-13 21:20:23 +0900
category: Kubernetes
---
# 🛠️ 실습: Kafka + Kubernetes 연동

Kafka는 대용량의 실시간 스트리밍 데이터를 처리하는 데 널리 사용되는 분산 메시징 시스템입니다. 이번 실습에서는 **Apache Kafka 클러스터를 Kubernetes 위에 배포하고 테스트 애플리케이션을 통해 연동**하는 과정을 다룹니다.

---

## ✅ 실습 목표

- Zookeeper + Kafka 클러스터를 K8s에 배포
- StatefulSet과 PVC로 상태 저장
- `kubectl`로 토픽 생성 및 메시지 송수신 테스트
- Kafka UI 연결 (선택)

---

## 📦 Kafka 구성 방식

Kafka는 안정적인 클러스터링을 위해 보통 다음과 같은 구성 요소를 사용합니다:

| 구성 요소 | 설명 |
|-----------|------|
| **Zookeeper** | Kafka 브로커의 메타데이터, 리더 선출 등을 관리 |
| **Kafka Broker** | 메시지 송수신 및 토픽 저장 담당 |
| **Kafka Client** | 데이터를 송수신하는 애플리케이션 또는 CLI |
| **Kafka UI** (선택) | 웹 기반 토픽 관리 도구 |

---

## 🔧 1. Kafka 배포 방식 선택

Kafka는 Helm Chart를 통해 가장 빠르게 배포할 수 있습니다. 대표적인 Helm Chart:

- [Bitnami Kafka Chart](https://artifacthub.io/packages/helm/bitnami/kafka)

### Helm 설치가 안 되어 있다면:

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

---

## 🚀 2. Kafka + Zookeeper 배포 (Helm)

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

helm install my-kafka bitnami/kafka \
  --set replicaCount=1 \
  --set zookeeper.replicaCount=1 \
  --set persistence.enabled=true \
  --set service.type=ClusterIP
```

### 확인

```bash
kubectl get pods
kubectl get svc
```

---

## 🔌 3. Kafka 접속 도구 설치

Kafka 클러스터와 통신하기 위해 Bitnami에서 제공하는 CLI 도구를 사용할 수 있습니다.

### Pod 안에서 kafka-console 명령 실행

```bash
kubectl exec -it my-kafka-0 -- bash
```

### 토픽 생성

```bash
kafka-topics.sh --create --topic test-topic --bootstrap-server localhost:9092
```

### 메시지 보내기

```bash
kafka-console-producer.sh --topic test-topic --bootstrap-server localhost:9092
> Hello
> World
```

### 메시지 받기

```bash
kafka-console-consumer.sh --topic test-topic --bootstrap-server localhost:9092 --from-beginning
```

---

## 🧪 4. Kafka 연동 테스트 애플리케이션 (Python)

### Kafka Python 의존성 설치

```bash
pip install kafka-python
```

### producer.py

```python
from kafka import KafkaProducer

producer = KafkaProducer(bootstrap_servers='localhost:9092')
producer.send('test-topic', b'Hello from Python!')
producer.flush()
```

### consumer.py

```python
from kafka import KafkaConsumer

consumer = KafkaConsumer('test-topic', bootstrap_servers='localhost:9092')
for msg in consumer:
    print(msg.value.decode())
```

👉 위 코드를 Kubernetes 외부에서 실행하려면 `NodePort` 또는 `LoadBalancer`로 Kafka 포트를 외부에 노출해야 합니다:

```bash
helm upgrade my-kafka bitnami/kafka --set service.type=NodePort
kubectl get svc my-kafka
```

---

## 🌐 5. Kafka UI 배포 (선택)

Kafka 토픽/컨슈머를 시각적으로 관리하려면 Kafka UI를 사용할 수 있습니다. 대표 도구:

- [Kafka UI (provectus)](https://github.com/provectus/kafka-ui)

### 간단한 배포 예

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kafka-ui
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kafka-ui
  template:
    metadata:
      labels:
        app: kafka-ui
    spec:
      containers:
        - name: kafka-ui
          image: provectuslabs/kafka-ui:latest
          ports:
            - containerPort: 8080
          env:
            - name: KAFKA_CLUSTERS_0_NAME
              value: local
            - name: KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS
              value: my-kafka.default.svc.cluster.local:9092
---
apiVersion: v1
kind: Service
metadata:
  name: kafka-ui
spec:
  type: NodePort
  selector:
    app: kafka-ui
  ports:
    - port: 8080
      targetPort: 8080
```

```bash
kubectl apply -f kafka-ui.yaml
```

브라우저에서 `localhost:<노출된 포트>` 로 접속 가능

---

## 📁 정리 구조

```plaintext
- Kafka (StatefulSet)
- Zookeeper (StatefulSet)
- Kafka Service
- Kafka UI (선택)
- Kafka Python Producer / Consumer
```

---

## 💡 실전 적용 아이디어

| 시나리오 | 설명 |
|----------|------|
| 실시간 로그 수집 | 각 마이크로서비스에서 Kafka로 로그 수집 |
| Kafka → Spark → S3 | 데이터 레이크 파이프라인 구축 |
| Kafka 기반 비동기 통신 | 주문/결제/배송 시스템 간 연결 |

---

## 🧼 클러스터 정리

```bash
helm uninstall my-kafka
kubectl delete -f kafka-ui.yaml
```

---

## ✅ 마무리 요약

- Helm으로 Kafka + Zookeeper를 간단히 배포 가능
- kubectl로 토픽 생성 및 메시지 테스트 가능
- Kafka UI로 시각적 토픽 관리 가능
- 외부 애플리케이션 연동 시 `NodePort` 또는 Ingress 필요

---

## 📚 참고 링크

- https://artifacthub.io/packages/helm/bitnami/kafka
- https://github.com/provectus/kafka-ui
- https://kafka.apache.org/documentation/