---
layout: post
title: Kubernetes - Kafka + Kubernetes 연동
date: 2025-06-14 21:20:23 +0900
category: Kubernetes
---
# 실습: Kafka + Kubernetes 연동

## 아키텍처 개요

```text
[Client Pod / kcat / Python]  ──(ClusterIP)──> [Kafka Brokers (StatefulSet)]
                                         └──> [ZooKeeper (StatefulSet)]  # ZK 모드일 때
[Kafka UI]  ────> Kafka bootstrap
[Prometheus JMX Exporter]  ──> Metrics ──> Grafana
```

상태 저장이 필요한 Kafka Broker와 ZooKeeper는 StatefulSet과 PVC(Persistent Volume Claim)을 사용하여 배포합니다. 클러스터 내부 통신에는 ClusterIP 서비스를 활용하며, 외부 접근이 필요할 경우 NodePort나 LoadBalancer 서비스 타입과 함께 `advertised.listeners` 설정을 적절히 조정해야 합니다. 보안 강화를 위해 SASL/TLS를 적용할 수 있으며, 초기화 작업으로 Job을 통해 토픽을 사전 생성하고, JMX Exporter를 통한 메트릭 수집으로 모니터링 체계를 구축할 수 있습니다.

---

## 배포 방식 선택: Helm, Operator, 수동 YAML

Kubernetes에 Kafka를 배포하는 방법은 크게 세 가지가 있으며, 각각의 장단점과 적합한 상황이 다릅니다.

| 방식 | 장점 | 단점 | 추천 상황 |
|---|---|---|---|
| **Bitnami Helm** | 빠른 배포, 잘 구성된 기본값, 풍부한 예시 | 세부적인 커스터마이즈 시 values.yaml 학습 필요 | 빠른 개념 검증(PoC)이나 소규모 운영 환경 |
| **Strimzi Operator** | 토픽, 사용자, 리스너를 CRD 수준에서 선언적 제어 | 운영 패러다임에 대한 학습 곡선 존재 | 팀 표준화나 운영 자동화가 중시되는 환경 |
| **수동 YAML** | 모든 설정을 완전히 제어 가능 | 초기 작성 및 지속적인 유지보수 비용이 큼 | 학습 목적이나 특수한 요구사항이 있는 경우 |

이 실습에서는 가장 보편적이고 시작이 쉬운 **Bitnami Helm 차트**를 중심으로 진행하며, 필요한 경우 Operator나 수동 구성의 핵심 개념을 함께 설명합니다.

---

## Helm으로 Kafka + ZooKeeper 배포 (내부 접속용)

먼저, 클러스터 내부에서만 접근 가능한 기본 PLAINTEXT 프로토콜로 Kafka를 배포해보겠습니다. 복잡한 외부 접속이나 보안 설정은 이후에 단계적으로 추가할 수 있습니다.

### Helm 리포지토리 추가 및 업데이트
```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

### 최소 구성 values 파일 작성
`values-internal.yaml` 파일을 생성하고 다음과 같이 설정합니다.

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

# 내부 통신용 리스너 설정 (암호화 없음)
listeners:
  client:
    protocol: PLAINTEXT
  interbroker:
    protocol: PLAINTEXT

# Kubernetes 내부 DNS를 사용하도록 광고 리스너 설정
advertisedListeners:
  - name: CLIENT
    value: PLAINTEXT://my-kafka.default.svc.cluster.local:9092
  - name: INTERBROKER
    value: PLAINTEXT://my-kafka-0.my-kafka-headless.default.svc.cluster.local:9093
```
> Bitnami 차트는 브로커 수와 헤드리스 서비스에 맞춰 `advertisedListeners`를 자동 생성해주지만, 내부에서 고정된 DNS 이름으로 접근해야 하는 경우에는 위와 같이 명시적으로 지정할 수 있습니다.

### Helm 설치 실행
```bash
helm install my-kafka bitnami/kafka -f values-internal.yaml
kubectl get pods -w  # 파드 생성 상태 확인
kubectl get svc      # 생성된 서비스 확인
```

설치 후 생성되는 주요 서비스는 다음과 같습니다.
- `my-kafka` (ClusterIP): 클라이언트 연결용 서비스 (현재는 내부 전용)
- `my-kafka-headless` (Headless): 개별 브로커 파드의 DNS 레코드를 제공하는 서비스

---

## 파드 내부에서 토픽과 메시지 테스트하기

### 브로커 파드에 접속
먼저 실행 중인 Kafka 브로커 파드 내부로 접속합니다.
```bash
kubectl exec -it my-kafka-0 -- bash
```

### 토픽 생성 및 목록 확인
파드 내부에서 Kafka 명령어 도구를 사용해 토픽을 생성하고 확인합니다.
```bash
kafka-topics.sh --bootstrap-server localhost:9092 --create --topic demo-topic --partitions 3 --replication-factor 1
kafka-topics.sh --bootstrap-server localhost:9092 --list
```

### 메시지 송수신 테스트
생성한 토픽으로 메시지를 보내고 받아봅니다.
```bash
# Producer 실행 (메시지 입력 대기)
kafka-console-producer.sh --bootstrap-server localhost:9092 --topic demo-topic
# > hello
# > kafka

# Consumer 실행 (저장된 메시지 조회)
kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic demo-topic --from-beginning --timeout-ms 5000
```

---

## kcat 도구를 활용한 테스트 (권장)

kcat(과거의 kafkacat)은 다양한 고급 기능(헤더, 키, 오프셋 제어 등)을 제공하는 강력한 Kafka 클라이언트 도구입니다.

### 임시 테스트 파드 생성
Kubernetes 클러스터 내에서 kcat을 사용하기 위해 임시 파드를 실행합니다.
```bash
kubectl run kcat --image=edenhill/kcat:1.7.1 -it --rm --restart=Never -- /bin/sh
```

### kcat을 이용한 기본 작업
파드 내부에서 다음 명령어들을 실행합니다.
```bash
# 브로커 및 토픽 메타데이터 조회
kcat -b my-kafka.default.svc.cluster.local:9092 -L

# 메시지 전송 (Producer)
echo "hello-from-kcat" | kcat -b my-kafka.default.svc.cluster.local:9092 -t demo-topic

# 메시지 수신 (Consumer - 처음부터 끝까지)
kcat -b my-kafka.default.svc.cluster.local:9092 -t demo-topic -o beginning -e
```

---

## Python Producer/Consumer 애플리케이션을 Kubernetes에 배포하기

### 애플리케이션 코드 작성
먼저 간단한 Producer와 Consumer 코드를 Python으로 작성합니다.

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

### Docker 이미지 빌드 및 푸시
애플리케이션을 컨테이너화하기 위한 Dockerfile을 작성합니다.

```Dockerfile
FROM python:3.12-slim
WORKDIR /app
RUN pip install --no-cache-dir kafka-python
COPY producer.py consumer.py ./
CMD [ "sleep", "infinity" ]
```

다음 명령어로 이미지를 빌드하고 레지스트리에 푸시합니다.
```bash
docker build -t <username>/kafka-demo:0.1.0 .
docker push <username>/kafka-demo:0.1.0
```

### Kubernetes 매니페스트 작성 및 배포
ConfigMap과 Deployment를 정의하여 애플리케이션을 배포합니다.

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

매니페스트를 적용하고 Consumer 로그를 확인합니다.
```bash
kubectl apply -f kafka-demo.yaml
kubectl logs deploy/kafka-consumer -f
```

---

## 외부 접속 구성: NodePort 또는 LoadBalancer

Kafka 클라이언트는 브로커에 직접 연결하고 재연결하기 때문에, 외부에서 접근하려면 `advertised.listeners` 설정을 **클라이언트가 접근 가능한 외부 IP 또는 도메인**으로 맞춰야 합니다.

### NodePort를 이용한 간단한 외부 접속 구성 (학습용)
`values-external.yaml` 파일을 다음과 같이 작성합니다.

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

# 광고 리스너를 외부 주소로 설정
# Minikube 환경이라면 `minikube ip` 명령어로 확인한 IP를 사용
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
> **중요**: NodePort는 학습이나 테스트용이며, 실제 운영 환경에서는 **LoadBalancer 서비스 타입과 DNS 레코드**를 사용하는 것을 강력히 권장합니다. 프로덕션에서는 브로커별로 로드밸런서를 구성하거나 Network Load Balancer(NLB)를 활용하는 것이 일반적입니다.

설치 또는 업그레이드를 실행합니다.
```bash
# Minikube IP를 ConfigMap에 저장
kubectl create configmap cluster-info --from-literal=minikube_ip=$(minikube ip)

# Helm 설치/업그레이드
helm upgrade --install my-kafka bitnami/kafka -f values-external.yaml
```

외부 클라이언트에서 다음과 같이 연결을 테스트할 수 있습니다.
```bash
kcat -b $(minikube ip):30092 -L
```

---

## 보안 구성: SASL/PLAIN과 TLS (개요)

운영 환경에서는 최소한 **SASL/PLAIN** 인증을, 가능하다면 **TLS 암호화**를 함께 적용해야 합니다. Bitnami Helm 차트는 이 두 가지를 모두 지원합니다.

- **인증**: `auth.clientProtocol: sasl` 및 `sasl.enabledMechanisms` 설정
- **암호화**: `tls.enabled: true` 설정 (자가 서명 인증서 또는 cert-manager를 통한 관리)

간략한 설정 예시는 다음과 같습니다.

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
  existingSecret: "kafka-tls-secret"  # 미리 생성한 TLS 키와 인증서가 들어있는 Secret 이름
```

> **주의**: SASL과 TLS를 적용한 후에는 모든 클라이언트의 연결 설정도 일치시켜야 합니다. 예를 들어 kcat을 사용할 때는 `-X security.protocol=SASL_SSL`과 같은 추가 옵션이 필요합니다.

---

## 토픽 자동 생성: Job을 활용한 초기화

환경을 처음 구성하거나 재구성할 때 필요한 기본 토픽을 자동으로 생성하는 것은 운영상 좋은 관행입니다. 필요한 토픽, 파티션 수, 데이터 보존 정책 등을 코드처럼 관리할 수 있습니다.

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

Job을 적용하고 로그를 확인합니다.
```bash
kubectl apply -f kafka-init-job.yaml
kubectl logs job/kafka-init-topics -f
```

---

## Kafka UI 도구 배포 (Provectus Kafka UI)

Kafka 클러스터의 상태, 토픽, 메시지 흐름 등을 시각적으로 모니터링하기 위해 웹 UI 도구를 배포할 수 있습니다. Provectus Kafka UI는 인기 있는 오픈소스 도구입니다.

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

배포 후 브라우저에서 `http://<노드IP>:30080` 주소로 접속하여 UI를 확인할 수 있습니다.

---

## 모니터링: JMX Exporter와 Prometheus 연동

Bitnami Kafka 차트는 JMX 메트릭 수집을 위한 Exporter를 내장하고 있어, Prometheus와의 연동을 쉽게 구성할 수 있습니다.

다음은 메트릭 수집을 활성화하는 values.yaml 설정의 예시입니다.

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

> Prometheus Operator가 설치된 환경이라면 `ServiceMonitor` 리소스를 생성하여 자동으로 메트릭을 스크레이핑하도록 할 수 있습니다. 수집된 메트릭은 Grafana에서 Kafka용 대시보드(예: 대시보드 ID 7589)를 import하여 시각화할 수 있습니다.

---

## 운영 고려사항: 리소스, 가용성, 네트워크

Kafka 클러스터를 안정적으로 운영하기 위해 고려해야 할 몇 가지 중요한 사항이 있습니다.

- **리소스 관리**: Kafka 브로커는 디스크 I/O, 네트워크, 메모리 사용량이 많은 워크로드입니다. 적절한 `requests`와 `limits` 설정과 함께, 충분한 IOPS를 제공하는 스토리지 클래스를 선택해야 합니다.
- **PodDisruptionBudget (PDB)**: 유지보수 작업 중에도 최소한의 서비스 가용성을 보장하기 위해 PDB를 설정합니다. 예를 들어 3개의 브로커가 있다면 `minAvailable: 2`로 설정할 수 있습니다.
- **안티어피니티**: 고가용성을 위해 브로커 파드가 서로 다른 물리 노드에 스케줄되도록 합니다.
- **네트워크 정책**: NetworkPolicy를 사용하여 필요한 출처(예: 애플리케이션 네임스페이스, Kafka UI, Prometheus)에서만 Kafka 포트(9092)로의 인바운드 트래픽을 허용하는 화이트리스트 방식을 적용합니다.
- **HPA 고려사항**: Kafka 브로커는 상태를 유지하는 스테이트풀 워크로드이므로, 일반적인 Horizontal Pod Autoscaler(HPA)를 사용한 자동 스케일링에는 적합하지 않습니다. 브로커 수의 변경은 파티션 재배치와 리밸런싱을 수반하는 운영 절차가 필요합니다.

**안티어피니티 설정 예시**
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

**네트워크 정책 예시 (엄격한 허용 방식)**
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

## ZooKeeper 모드 vs KRaft 모드 (무ZooKeeper)

Kafka는 기존에 메타데이터 관리를 위해 ZooKeeper에 의존해왔지만, 최신 버전에서는 **KRaft(Kafka Raft)** 모드를 도입하여 내부적으로 메타데이터를 관리할 수 있게 되었습니다.

- **ZooKeeper 모드**: 오랜 기간 검증된 안정적인 방식이며 레퍼런스가 풍부합니다. Bitnami 차트의 기본 설정입니다.
- **KRaft 모드**: ZooKeeper 의존성을 제거하여 아키텍처를 단순화합니다. 운영 복잡도를 줄이고 신규 클러스터 구축 시 점점 더 권장되는 추세입니다. Strimzi Operator나 최신 Bitnami 차트에서 `kraft.enabled: true`와 같은 플래그로 활성화할 수 있습니다.

**선택 가이드**

| 항목 | ZooKeeper 모드 | KRaft 모드 |
|---|---|---|
| **성숙도** | 매우 높음 | 빠르게 성숙 중 |
| **운영 복잡도** | ZooKeeper 클러스터 추가 운영 필요 | 아키텍처 단순, 운영 부담 감소 |
| **신규 구축 권장** | 보수적 접근 또는 기존 레거시 연계 필요 | 신규 클러스터 권장 추세 |

---

## 문제 해결을 위한 핵심 확인 사항

Kafka on Kubernetes 배포 및 운영 중 문제가 발생할 경우 다음 사항들을 순차적으로 점검해보세요.

1.  **Kubernetes 이벤트 확인**: `kubectl get events --sort-by=.lastTimestamp` 명령어로 스케줄 실패, 리소스 부족 등의 근본 원인을 찾을 수 있습니다.
2.  **브로커 로그 분석**: `kubectl logs my-kafka-0 -c kafka -f` 명령어로 브로커의 시작 및 실행 로그를 확인합니다. 리스너 설정 오류, 디스크 공간 부족 등이 여기서 나타납니다.
3.  **내부 DNS 확인**: 클라이언트 애플리케이션이 `my-kafka.default.svc.cluster.local` 또는 `my-kafka-0.my-kafka-headless.default.svc.cluster.local`와 같은 서비스 DNS 이름을 정상적으로 resolving 하는지 확인합니다.
4.  **외부 접속 문제**: NodePort 또는 LoadBalancer를 사용하는 경우, 해당 포트가 실제로 열려 있고 방화벽 규칙에 막히지 않았는지, 그리고 가장 중요한 **`advertised.listeners` 설정이 클라이언트가 연결을 시도하는 정확한 외부 IP 또는 도메인을 가리키고 있는지** 재확인합니다.
5.  **컨슈머 지연 증가**: 컨슈머 레이턴시가 증가한다면 디스크 I/O 병목, 네트워크 지연, GC(Garbage Collection) 과잉, 배치 크기 설정, Producer의 `acks` 설정 등을 종합적으로 점검해야 합니다.
6.  **파드 재시작 루프**: 파드가 계속 재시작된다면 PVC의 권한 문제나 디스크 공간 부족, 환경변수 오타, 리스너 포트 충돌 등을 의심해볼 수 있습니다.

---

## 전체 정리 및 삭제

실습을 마친 후 모든 리소스를 정리하려면 다음 명령어들을 실행합니다.

```bash
# Kafka UI 삭제
kubectl delete deploy/kafka-ui svc/kafka-ui

# Helm으로 설치한 Kafka 릴리즈 삭제
helm uninstall my-kafka

# 토픽 초기화 Job 삭제
kubectl delete job kafka-init-topics

# 테스트용 Python 애플리케이션 삭제
kubectl delete -f kafka-demo.yaml
```

---

## 결론

Kubernetes 환경에 Apache Kafka를 성공적으로 배포하고 운영하는 것은 몇 가지 핵심 원칙을 이해하는 데서 시작합니다. 먼저, Bitnami Helm 차트를 사용하면 StatefulSet과 PVC를 활용한 안정적인 상태 저장 배포를 빠르게 구성할 수 있습니다. 내부 통신 테스트는 브로커 파드 내의 콘솔 도구로, 보다 실전적인 테스트는 kcat 같은 강력한 클라이언트 도구로 수행하는 것이 좋습니다.

애플리케이션 연동 시에는 환경 변수와 ConfigMap을 활용해 연결 정보를 중앙에서 관리해야 합니다. 외부 접속을 구성할 때는 반드시 `advertised.listeners` 설정을 클라이언트의 관점에서 접근 가능한 주소로 맞춰야 한다는 점을 잊지 마세요. 운영 환경을 위해서는 SASL/PLAIN 인증과 TLS 암호화를 적용하는 것이 보안의 기본입니다.

운영 측면에서는 토픽 생성과 같은 초기화 작업을 Job으로 자동화하고, Kafka UI를 통해 가시성을 확보하며, JMX Exporter와 Prometheus를 연동한 모니터링 체계를 구축해야 합니다. 또한 고가용성과 보안을 위해 PodDisruptionBudget, 안티어피니티, NetworkPolicy와 같은 Kubernetes 네이티브 기능을 적극 활용하세요.

마지막으로, Kafka 생태계는 ZooKeeper에 의존하는 기존 모드에서 벗어나 KRaft 모드로 진화하고 있습니다. 신규 클러스터를 구축할 경우 아키텍처 단순화와 운영 효율성 향상을 위해 KRaft 모드를 적극적으로 고려해보는 것을 권장합니다.

이 실습을 통해 Kubernetes와 Kafka의 기본적인 연동 패턴을 익혔다면, 이제 이를 바탕으로 자신의 애플리케이션 요구사항과 운영 환경에 맞춰 구성 요소를 확장하고 세부적으로 튜닝해 나갈 수 있을 것입니다.
