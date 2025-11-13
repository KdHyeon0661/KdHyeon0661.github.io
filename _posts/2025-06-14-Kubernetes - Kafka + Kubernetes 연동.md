---
layout: post
title: Kubernetes - Kafka + Kubernetes 연동
date: 2025-06-14 21:20:23 +0900
category: Kubernetes
---
# 실습: Kafka + Kubernetes 연동

## 1. 아키텍처 개요

```text
[Client Pod / kcat / Python]  ──(ClusterIP)──> [Kafka Brokers (StatefulSet)]
                                         └──> [ZooKeeper (StatefulSet)]  # ZK 모드일 때
[Kafka UI]  ────> Kafka bootstrap
[Prometheus JMX Exporter]  ──> Metrics ──> Grafana
```

- **상태 저장**: Broker/ZooKeeper는 StatefulSet + PVC
- **내부연결**: ClusterIP로 같은 클러스터 내 접근
- **외부연결**: NodePort/LoadBalancer + `advertised.listeners` 조정
- **보안**: (옵션) SASL/TLS
- **초기화**: Job로 토픽 사전 생성
- **관측성**: JMX Exporter → Prometheus → Grafana

---

## 2. 배포 방식 선택: Helm, Operator, 수동 YAML

| 방식 | 장점 | 단점 | 추천 상황 |
|---|---|---|---|
| **Bitnami Helm** | 빠름, 기본값 잘 구성, 예시 풍부 | 세부 확장 시 values 학습 필요 | 빠른 PoC, 소규모 운영 |
| **Strimzi Operator** | 토픽/유저/리스너 CRD 수준 제어 | 러닝커브 | 팀 표준화/운영 자동화 |
| 수동 YAML | 완전 제어 | 초기 작성/유지 비용 큼 | 학습/특수 요구 |

> 본 실습은 **Bitnami Helm** 중심으로 진행하고, 필요 지점에 Operator/수동 포인트를 짚습니다.

---

## 3. Helm으로 Kafka + ZooKeeper 배포 (내부접속용)

> 로컬/내부 접근만 필요한 **기본 PLAINTEXT**.
> 시작은 단순하게, 이후 외부접속·보안으로 확장합니다.

### 3.1 Helm 리포지토리 & 설치

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

### 3.2 values-internal.yaml (권장) — 최소 구성

```yaml
replicaCount: 1

zookeeper:
  enabled: true
  replicaCount: 1
  persistence:
    enabled: true
    size: 5Gi

persistence:
  enabled: true
  size: 10Gi

service:
  type: ClusterIP

# 내부 리스너(PLAINTEXT)
listeners:
  client:
    protocol: PLAINTEXT
  interbroker:
    protocol: PLAINTEXT

# Kubernetes DNS로 브로커 광고(내부 only)
advertisedListeners:
  - name: CLIENT
    value: PLAINTEXT://my-kafka.default.svc.cluster.local:9092
  - name: INTERBROKER
    value: PLAINTEXT://my-kafka-0.my-kafka-headless.default.svc.cluster.local:9093
```

> Bitnami 차트의 `advertisedListeners`는 브로커 수/헤드리스 서비스에 맞춰 **자동 생성**되지만,
> 내부 DNS 기반 고정이 필요하면 명시적으로 제공할 수 있습니다.

### 3.3 설치

```bash
helm install my-kafka bitnami/kafka -f values-internal.yaml
kubectl get pods -w
kubectl get svc
```

**주요 서비스**

- `my-kafka`(ClusterIP: 외부용이 아님)
- `my-kafka-headless`(Headless: 브로커 DNS 해결)

---

## 4. 파드 내부에서 토픽/메시지 테스트

### 4.1 브로커 파드 접속

```bash
kubectl exec -it my-kafka-0 -- bash
```

### 4.2 토픽 생성/목록

```bash
kafka-topics.sh --bootstrap-server localhost:9092 --create --topic demo-topic --partitions 3 --replication-factor 1
kafka-topics.sh --bootstrap-server localhost:9092 --list
```

### 4.3 메시지 송수신

```bash
# Producer
kafka-console-producer.sh --bootstrap-server localhost:9092 --topic demo-topic
> hello
> kafka

# Consumer
kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic demo-topic --from-beginning --timeout-ms 5000
```

---

## 5. kcat로 테스트 (권장 도구)

> 다양한 옵션/헤더/키/오프셋 제어에 편리.

### 5.1 임시 파드 생성

```bash
kubectl run kcat --image=edenhill/kcat:1.7.1 -it --rm --restart=Never -- /bin/sh
```

### 5.2 송수신

```bash
# 목록
kcat -b my-kafka.default.svc.cluster.local:9092 -L

# Producer
echo "hello-from-kcat" | kcat -b my-kafka.default.svc.cluster.local:9092 -t demo-topic

# Consumer
kcat -b my-kafka.default.svc.cluster.local:9092 -t demo-topic -o beginning -e
```

---

## 6. Python Producer/Consumer를 Kubernetes로 배포

### 6.1 코드

**producer.py**
```python
from kafka import KafkaProducer
import os, time

bootstrap = os.getenv("KAFKA_BOOTSTRAP", "my-kafka.default.svc.cluster.local:9092")
topic = os.getenv("KAFKA_TOPIC", "demo-topic")

producer = KafkaProducer(bootstrap_servers=bootstrap.encode("utf-8"))
for i in range(5):
    msg = f"py-producer msg-{i}".encode("utf-8")
    producer.send(topic, msg)
    print("sent:", msg)
    time.sleep(1)
producer.flush()
```

**consumer.py**
```python
from kafka import KafkaConsumer
import os

bootstrap = os.getenv("KAFKA_BOOTSTRAP", "my-kafka.default.svc.cluster.local:9092")
topic = os.getenv("KAFKA_TOPIC", "demo-topic")

consumer = KafkaConsumer(
    topic,
    bootstrap_servers=[bootstrap],
    auto_offset_reset="earliest",
    enable_auto_commit=True,
)

for msg in consumer:
    print("recv:", msg.value.decode("utf-8"))
```

### 6.2 Dockerfile

```Dockerfile
FROM python:3.12-slim
WORKDIR /app
RUN pip install --no-cache-dir kafka-python
COPY producer.py consumer.py ./
CMD [ "sleep", "infinity" ]
```

빌드/푸시:

```bash
docker build -t <username>/kafka-demo:0.1.0 .
docker push <username>/kafka-demo:0.1.0
```

### 6.3 K8s 배포

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kafka-demo-config
data:
  KAFKA_BOOTSTRAP: "my-kafka.default.svc.cluster.local:9092"
  KAFKA_TOPIC: "demo-topic"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kafka-producer
spec:
  replicas: 1
  selector:
    matchLabels: { app: kafka-producer }
  template:
    metadata:
      labels: { app: kafka-producer }
    spec:
      containers:
        - name: producer
          image: <username>/kafka-demo:0.1.0
          command: ["python","/app/producer.py"]
          envFrom:
            - configMapRef: { name: kafka-demo-config }
          resources:
            requests: { cpu: "50m", memory: "64Mi" }
            limits: { cpu: "200m", memory: "256Mi" }
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kafka-consumer
spec:
  replicas: 1
  selector:
    matchLabels: { app: kafka-consumer }
  template:
    metadata:
      labels: { app: kafka-consumer }
    spec:
      containers:
        - name: consumer
          image: <username>/kafka-demo:0.1.0
          command: ["python","/app/consumer.py"]
          envFrom:
            - configMapRef: { name: kafka-demo-config }
          resources:
            requests: { cpu: "50m", memory: "64Mi" }
            limits: { cpu: "200m", memory: "256Mi" }
```

적용/확인:

```bash
kubectl apply -f kafka-demo.yaml
kubectl logs deploy/kafka-consumer -f
```

---

## 7. 외부 접속(클러스터 밖에서) — NodePort/LoadBalancer

Kafka는 **클라이언트가 브로커에 재접속**하기 때문에 `advertised.listeners`를 **외부 IP/도메인**으로 맞춰야 합니다.

### 7.1 간단 NodePort 예(학습용)

**values-external.yaml**

```yaml
service:
  type: NodePort
  nodePorts:
    client: 30092
  externalTrafficPolicy: Cluster

listeners:
  client:
    protocol: PLAINTEXT
  interbroker:
    protocol: PLAINTEXT

# 주의: advertised listeners를 외부로
# Minikube라면: minikube ip 로 확인
advertisedListeners:
  - name: CLIENT
    value: PLAINTEXT://$(MINIKUBE_IP):30092
  - name: INTERBROKER
    value: PLAINTEXT://my-kafka-0.my-kafka-headless.default.svc.cluster.local:9093

extraEnvVars:
  - name: MINIKUBE_IP
    valueFrom:
      configMapKeyRef:
        name: cluster-info
        key: minikube_ip
```

> **주의**: 실제 환경에서는 **LoadBalancer + DNS**를 권장. 브로커별 로드밸런서 또는 NLB 구성이 일반적입니다.

적용:

```bash
# minikube ip 저장
kubectl create configmap cluster-info --from-literal=minikube_ip=$(minikube ip)

helm upgrade --install my-kafka bitnami/kafka -f values-external.yaml
```

클라이언트에서:

```bash
kcat -b $(minikube ip):30092 -L
```

---

## 8. 보안(옵션): SASL/PLAIN + TLS 개요

운영환경에서는 **최소 SASL/PLAIN**, 가능하면 **TLS**를 적용합니다. Bitnami Chart는 다음을 지원:

- `auth.clientProtocol: sasl` / `sasl.enabledMechanisms: PLAIN,SCRAM-SHA-512`
- `tls.enabled: true` / 자가서명 or cert-manager와 연동

예시(요약):

```yaml
auth:
  clientProtocol: sasl
  sasl:
    interbrokerMechanism: scram-sha-512
    mechanisms: scram-sha-512
    jaas:
      clientUsers: [ "app" ]
      clientPasswords: [ "app-password" ]
      zookeeperUser: "zkuser"
      zookeeperPassword: "zkpass"

tls:
  enabled: true
  existingSecret: "kafka-tls-secret"  # 미리 TLS 비밀키/인증서 저장
```

> SASL/TLS 구성 후에는 **클라이언트 옵션도 일치**해야 합니다.
> (kcat의 `-X security.protocol=SASL_SSL` 등)

---

## 9. 토픽 자동 생성: Job로 초기화

환경 부팅 시 필요한 **토픽/파티션/리텐션**을 **코드처럼 관리**하는 것이 중요합니다.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: kafka-init-topics
spec:
  backoffLimit: 1
  template:
    spec:
      restartPolicy: OnFailure
      containers:
        - name: init
          image: bitnami/kafka:3.7
          command: ["bash","-c"]
          args:
            - |
              set -e
              BOOT=my-kafka.default.svc.cluster.local:9092
              kafka-topics.sh --bootstrap-server $BOOT --create --if-not-exists --topic orders --partitions 6 --replication-factor 1 --config retention.ms=604800000
              kafka-topics.sh --bootstrap-server $BOOT --create --if-not-exists --topic payments --partitions 3 --replication-factor 1
              echo "topics initialized"
```

적용:

```bash
kubectl apply -f kafka-init-job.yaml
kubectl logs job/kafka-init-topics -f
```

---

## 10. Kafka UI 배포 (Provectus)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kafka-ui
spec:
  replicas: 1
  selector: { matchLabels: { app: kafka-ui } }
  template:
    metadata: { labels: { app: kafka-ui } }
    spec:
      containers:
        - name: ui
          image: provectuslabs/kafka-ui:latest
          env:
            - name: KAFKA_CLUSTERS_0_NAME
              value: local
            - name: KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS
              value: my-kafka.default.svc.cluster.local:9092
          ports: [{ containerPort: 8080 }]
---
apiVersion: v1
kind: Service
metadata:
  name: kafka-ui
spec:
  type: NodePort
  selector: { app: kafka-ui }
  ports:
    - port: 8080
      targetPort: 8080
      nodePort: 30080
```

접속:

```
http://<노드IP>:30080
```

---

## 11. 모니터링: JMX Exporter + Prometheus

Bitnami Kafka는 JMX Exporter 연동을 지원합니다(차트 옵션 참고).

예시(요약):

```yaml
metrics:
  jmx:
    enabled: true
  kafka:
    enabled: true

serviceMonitor:
  enabled: true
  namespace: monitoring
```

> Prometheus Operator가 있으면 `ServiceMonitor`로 자동 스크레이프.
> Grafana에서는 Kafka JMX 대시보드(예: 7589 등)를 사용.

---

## 12. 리소스/가용성/네트워크

- **리소스**: 브로커는 디스크/IO/메모리 영향 큼. `requests/limits` + 적절한 스토리지 클래스(IOPS).
- **PDB**: 점검 시 최소 가용 브로커 유지. 예: 3브로커 구성 시 `minAvailable: 2`.
- **안티어피니티**: 브로커를 서로 다른 노드에 분산.
- **NetworkPolicy**: 허용 리스트 방식(Ingress from 앱 네임스페이스, Kafka UI, Prometheus).
- **HPA**: **상태풀 워크로드에는 일반적으로 부적합**. 브로커 수 증감은 운영 절차 필요(재분배/리밸런스).

**안티어피니티 예시(요약)**

```yaml
podAntiAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchExpressions:
          - key: app.kubernetes.io/name
            operator: In
            values: ["kafka"]
      topologyKey: "kubernetes.io/hostname"
```

**NetworkPolicy 예시(엄격 허용)**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: kafka-allow-apps
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: kafka
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: default
          podSelector:
            matchLabels:
              app: kafka-producer
        - namespaceSelector:
            matchLabels:
              name: default
          podSelector:
            matchLabels:
              app: kafka-consumer
      ports:
        - port: 9092
  policyTypes: ["Ingress"]
```

---

## 13. ZooKeeper vs KRaft(무ZK)

- **ZooKeeper 모드**: 안정적이며 풍부한 레퍼런스. Bitnami 기본값.
- **KRaft 모드**: 최근 Kafka의 **내장 메타데이터 관리**. 구성 단순화, ZK 제거.
  Operator/차트에서 `kraft.enabled: true` 플래그로 전환 가능(차트 버전에 따라 옵션명 상이).

**선택 가이드**

| 항목 | ZooKeeper | KRaft |
|---|---|---|
| 성숙도 | 매우 높음 | 빠르게 성숙 |
| 운영 복잡도 | ZK 운영 부담 | 단순 |
| 신규 구축 | 보수적/레거시 연계 | 신규 권장 추세 |

---

## 14. 트러블슈팅 체크리스트

- `kubectl get events --sort-by=.lastTimestamp`: 스케줄 실패/리소스 부족
- 브로커 로그: `kubectl logs my-kafka-0 -c kafka -f`
- DNS: `my-kafka.default.svc.cluster.local` / `my-kafka-0.my-kafka-headless...` 해석 확인
- 외부접속 실패:
  - NodePort/NLB 포트 오픈?
  - `advertised.listeners` 실제 외부 IP/도메인?
- 컨슈머 레이턴시 증가:
  - 디스크 IOPS, 네트워크, GC, 배치 사이즈, acks 설정 점검
- 재시작 루프:
  - PVC 권한/공간, 환경변수 오타, 리스너 충돌, 포트 점유

---

## 15. 전체 정리 · 삭제

```bash
# UI
kubectl delete deploy/kafka-ui svc/kafka-ui

# Kafka(Helm)
helm uninstall my-kafka

# 토픽 Job
kubectl delete job kafka-init-topics

# 테스트 앱
kubectl delete -f kafka-demo.yaml
```

---

## 결론 요약

| 주제 | 핵심 |
|---|---|
| 배포 | Bitnami Helm으로 빠르게 구성, StatefulSet+PVC |
| 테스트 | 브로커 파드 내부 콘솔 도구, **kcat**로 실전 검증 |
| 앱 연동 | Python Producer/Consumer를 K8s로 배포(ENV/CM/Secret) |
| 외부접속 | NodePort/LoadBalancer + **advertised.listeners** 필수 조정 |
| 보안 | (옵션) SASL/PLAIN/TLS, 클라이언트 설정 일치 |
| 운영 | 토픽 초기화 Job, Kafka UI, JMX 모니터링, PDB/안티어피니티/NP |
| 진화 | ZooKeeper → **KRaft**로 단순화 트렌드 |

---

## 참고

- Bitnami Kafka Helm: https://artifacthub.io/packages/helm/bitnami/kafka
- Kafka 공식 문서: https://kafka.apache.org/documentation/
- kcat(카프카 캐틀): https://github.com/edenhill/kcat
- Provectus Kafka UI: https://github.com/provectus/kafka-ui
- Prometheus JMX Exporter: https://github.com/prometheus/jmx_exporter
