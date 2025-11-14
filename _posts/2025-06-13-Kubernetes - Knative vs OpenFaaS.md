---
layout: post
title: Kubernetes - Knative vs OpenFaaS
date: 2025-06-13 19:20:23 +0900
category: Kubernetes
---
# K8s 기반 서버리스: Knative vs OpenFaaS

**Serverless = 코드에만 집중**하고, 인프라 관리는 플랫폼이 담당.
Kubernetes에서도 오픈소스 기반으로 **서버리스를 구현**할 수 있습니다. 그 대표 주자가 **Knative**와 **OpenFaaS**.

---

## 한눈 비교 (의사결정 표)

| 구분 | **Knative** | **OpenFaaS** |
|---|---|---|
| 철학/포지션 | K8s 네이티브(Serving/Eventing), **HTTP+Event** 표준화 | 경량 **Function 프레임워크**, 쉬운 온보딩 |
| 설치 난이도 | 중~상 (Ingress/Gateway 필요) | 하~중 (Helm 기반 간단) |
| 확장/스케일 | 기본 **scale-to-zero**, `Activator`/`queue-proxy` | **scale-to-zero 가능**, KEDA와 궁합 ↑ |
| 이벤트 | **Knative Eventing(CloudEvents)** | **Connectors** (Kafka, NATS, Cron 등) |
| 빌드 | 이미지 전제(외부 빌드/Buildpacks/tekton) | handler 중심(템플릿, `faas-cli`) |
| 모니터링 | Prometheus/Grafana 연동(별도 구성) | Prometheus 내장, 대시보드 쉬움 |
| 트래픽 전략 | **Revision/Traffic Split/Canary** | Gateway 라우팅, Canary는 Helm/K8s로 |
| 러닝 커브 | **CRD·개념 많음** | 낮음(코드→`faas-cli up`) |
| 적합 사례 | HTTP 기반 API, CloudEvents, GCR/Cloud Run 친화 | PoC/엣지/경량 자동화, 다양한 트리거 |

---

## Serverless on K8s가 주는 가치

- **벤더 종속 회피**: FaaS 클라우드 대신 **이식성 높은 K8s** 위에서 동일 패턴
- **마이크로서비스 + GitOps**: 선언적 배포, Canary/Blue-Green, 자동 롤백
- **이벤트 드리븐**: 스트림/버스/크론/웹훅 등으로 워크플로 자동화

### 간단한 오토스케일 직관식

$$
\text{DesiredReplicas} \approx \left\lceil \frac{\text{Incoming QPS} \times \text{AvgLatency}}{\text{Concurrency per Pod}} \right\rceil
$$
- **Knative**: concurrency/RT 기반 **KPA**(Knative Pod Autoscaler) 또는 HPA로 결정
- **OpenFaaS**: Prometheus/KEDA 메트릭 기반 증감

---

## Knative — Serving & Eventing로 보는 구조

### 아키텍처 핵심

- **Serving**: `Service` → `Configuration` → **`Revision`** 생성 → `Route`로 트래픽 분배
  Ingress(예: **Kourier**, Istio)를 통해 **HTTP** 유입 → **Activator**가 scale-to-zero에서 깨움 → **queue-proxy**가 per-Pod 트래픽 제어
- **Eventing**: **Broker/Trigger** + **CloudEvents** 규격, Kafka 등 다수 소스

```
[Client] → [Ingress(Kourier/Istio)] → [Activator] → [Service/Revision Pods]
                                          ↘ queue-proxy → Container
```

### 설치 (요지)

```bash
# Serving CRDs/Core

kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.13.0/serving-crds.yaml
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.13.0/serving-core.yaml

# Kourier (경량 게이트웨이)

kubectl apply -f https://github.com/knative/net-kourier/releases/download/knative-v1.13.0/kourier.yaml
kubectl patch configmap/config-network -n knative-serving \
  -p='{"data":{"ingress.class":"kourier.ingress.networking.knative.dev"}}'
```

> 프로덕션: Ingress LB, DNS, TLS(cert-manager), HPA/KPA 튜닝 필수

### 가장 작은 Knative Service

```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: hello-knative
  namespace: default
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/minScale: "0"      # scale-to-zero 허용
        autoscaling.knative.dev/target: "100"      # pod당 목표 동시성
    spec:
      containers:
      - image: gcr.io/knative-samples/helloworld-go
        env:
        - name: TARGET
          value: "Knative"
```

배포 및 엔드포인트 확인:
```bash
kubectl apply -f hello-knative.yaml
kubectl get ksvc hello-knative
```

### Canary/Traffic Split (Revision 기반)

```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: web
spec:
  template:
    spec:
      containers:
      - image: ghcr.io/you/web:v2
  traffic:
  - revisionName: web-00001
    percent: 80
  - latestRevision: true
    percent: 20
```

### Knative Eventing 미니 예제 (Broker/Trigger)

**Broker 설치(네임스페이스 기본)**
```bash
kubectl apply -f - <<'YAML'
apiVersion: eventing.knative.dev/v1
kind: Broker
metadata:
  name: default
  namespace: events
YAML
```

**Trigger: 특정 타입 이벤트 → 서비스로 전달**
```yaml
apiVersion: eventing.knative.dev/v1
kind: Trigger
metadata:
  name: hello-trigger
  namespace: events
spec:
  broker: default
  filter:
    attributes:
      type: com.example.hello
  subscriber:
    ref:
      apiVersion: serving.knative.dev/v1
      kind: Service
      name: hello-knative
      namespace: default
```

**CloudEvents 전송(예: curl)**
```bash
curl -v "http://broker-ingress.events.svc.cluster.local/events/default" \
  -X POST \
  -H "Ce-Type: com.example.hello" \
  -H "Ce-Source: cli" \
  -H "Ce-Id: 42" \
  -H "Ce-SpecVersion: 1.0" \
  -H "Content-Type: application/json" \
  -d '{"msg":"hi"}'
```

---

## OpenFaaS — 경량 Function 프레임워크

### 구성요소

- **Gateway**(UI+API), **faas-netes**(K8s backend), **Prometheus**(기본), **queue-worker**(비동기)
- **Function Watchdog**: HTTP/STDIO 어댑터(컨테이너를 함수처럼)

### 설치(Helm)

```bash
helm repo add openfaas https://openfaas.github.io/faas-netes/
helm upgrade openfaas --install openfaas/openfaas \
  --namespace openfaas --create-namespace \
  --set basic_auth=true
# 비밀번호 확인

kubectl -n openfaas get secret basic-auth -o jsonpath='{.data.basic-auth-password}' | base64 -d; echo
```

### 함수 스캐폴드–배포(파이썬 예)

```bash
faas-cli new hello-openfaas --lang python
# handler.py 편집 후

faas-cli up -f hello-openfaas.yml
curl http://<gateway>:8080/function/hello-openfaas
```

**`hello-openfaas.yml` (핵심)**
```yaml
provider:
  name: openfaas
  gateway: http://gateway.openfaas:8080
functions:
  hello-openfaas:
    lang: python
    handler: ./hello-openfaas
    image: ghcr.io/you/hello-openfaas:latest
    annotations:
      com.openfaas.scale.zero: "true"
    labels:
      com.openfaas.scale.max: "10"
      com.openfaas.scale.min: "0"
```

### 이벤트 트리거(크론/카프카/비동기)

- **Cron**: `faas-netes` 커넥터(체크아웃: cron-connector)
- **Kafka/NATS/MQTT**: 커넥터 배포 후 topic→function 연결
- **Async**: `POST /async-function/<fn>` → queue-worker 소비

---

## 스케일링·콜드 스타트·지연 튜닝

### Knative Autoscaling(요지)

- **KPA**: 요청 동시성 기반, `autoscaling.knative.dev/target`, `window` 등
- **HPA 백엔드**도 가능
- **minScale**로 콜드스타트 회피, **containerConcurrency**로 Pod당 동시 처리 튜닝

예) 빠른 기동이 어려운 앱:
```yaml
metadata:
  annotations:
    autoscaling.knative.dev/minScale: "1"          # 항상 1개 유지
    autoscaling.knative.dev/scaleDownDelay: "300s" # 꺼짐 지연
spec:
  template:
    spec:
      containerConcurrency: 2
```

### OpenFaaS Scaling

- 기본 Prometheus 메트릭 기반, `com.openfaas.scale.*` 레이블
- **KEDA 연동**으로 Kafka lag/Queue depth 등 지표 기반 스케일
- `read_timeout/write_timeout/exec_timeout` watchdog 옵션으로 안정성 확보

---

## 관측(Observability)

### Knative

- **Req/Resp/Concurrency**: queue-proxy, Activator 메트릭(는 Prometheus로 수집)
- 대시보드: kube-prometheus-stack + 커스텀 패널(릿시트: `queue_requests_per_second`, `container_concurrency`)

PromQL 예:
```promql
sum(rate(queue_requests_total{namespace="default", service="hello-knative"}[1m]))
```

### OpenFaaS

- **Prometheus 내장**, 함수별 `gateway_function_invocation_total`, `*_duration_seconds`
- Grafana 임포트용 대시보드 템플릿 풍부

---

## 보안·네트워크·비용 관점

### 공통

- **PSA restricted** 네임스페이스 라벨, 이미지 서명(Cosign), 취약점 스캔(Trivy)
- Ingress TLS(cert-manager), 내부 **mTLS**(메시 또는 SPIFFE/SPIRE)
- **NetworkPolicy**로 함수 간 동서 트래픽 최소화

예) 특정 네임스페이스 egress 제한:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: fn-egress
  namespace: functions
spec:
  podSelector: {}
  policyTypes: [Egress]
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: kafka
    ports:
    - port: 9092
      protocol: TCP
```

### 비용 직감(Scale-to-zero의 힘)

- 장시간 유휴 트래픽이면 **scale-to-zero**가 노드 자원 절감에 직결
- 반대로 **cold start 비용**이 큰 서비스는 minScale로 절충

---

## CI/CD & GitOps

### Knative + ArgoCD

- `ksvc`/Eventing 리소스를 **Git에 선언**→ ArgoCD 자동 동기화
- Canary는 `traffic` 필드 수정 PR로 점진 전개

### OpenFaaS + GitHub Actions

- `faas-cli build/push/deploy` 파이프라인
{% raw %}
```yaml
name: openfaas-cicd
on: { push: { branches: [main] } }
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - run: curl -sSL https://cli.openfaas.com | sudo sh
    - run: faas-cli login --username admin --password ${{ secrets.OF_PASS }} --gateway ${{ secrets.OF_GATEWAY }}
    - run: faas-cli up -f stack.yml --gateway ${{ secrets.OF_GATEWAY }}
```
{% endraw %}

---

## 실전 레퍼런스 아키텍처

### HTTP API + Canary (Knative)

- Kourier(LB) + Knative Serving
- `minScale=1`(핵심 경로만 warm), 비핵심은 scale-to-zero
- **Revision Traffic** 10%→30%→100% 점진

### 이벤트 파이프라인(스트림 처리)

- **Knative Eventing + Kafka**: CloudEvents 라우팅, DLQ 설정
- **OpenFaaS + KEDA(Kafka scaler)**: topic lag 기반 자동 스케일

### 엣지/온프레 경량 자동화

- K3s 위 **OpenFaaS**: Cron/웹훅 기반 로컬 작업 자동화, Prom 내장 관측

---

## 트러블슈팅 체크리스트

### Knative

- **대상 도메인/Ingress**: DNS/CNAME, LB health
- **콜드스타트가 길다**: `minScale`, 이미지 slim, 초기화 지연 제거
- **4xx/5xx**: queue-proxy/activator/ingress 로그 → `kubectl -n knative-serving logs ...`
- **Eventing**: Broker/Trigger 상태, delivery(재시도) 파라미터

### OpenFaaS

- **권한/게이트웨이 인증**: `faas-cli login` 확인
- **비동기 지연**: queue-worker 로그, NATS/Kafka 커넥터 상태
- **스케일 안됨**: Prometheus targets, KEDA 스케일러 설정

---

## 고급 튜닝 팁

### Knative

- `containerConcurrency` 낮추면 지연 안정↑, 과도하게 낮추면 Pod 폭증
- `autoscaling.knative.dev/targetUtilizationPercentage`로 여유율 제어
- `readinessProbe` 정확화로 rollout 중 5xx 억제

### OpenFaaS

- watchdog `read_timeout`/`write_timeout`/`exec_timeout` 조정
- 이미지 빌드 시 **템플릿 커스터마이즈**(uvicorn-gunicorn 파이썬, tini, distroless)
- `basic_auth` 외 OIDC 게이트웨이 연동(기업 SSO)

---

## 실전 코드 예시 묶음

### Knative — Go 핸들러 (간단 API)

```go
// main.go
package main
import (
  "fmt"
  "log"
  "net/http"
  "os"
)
func handler(w http.ResponseWriter, r *http.Request) {
  target := os.Getenv("TARGET")
  if target == "" { target = "World" }
  fmt.Fprintf(w, "Hello %s\n", target)
}
func main() {
  http.HandleFunc("/", handler)
  log.Fatal(http.ListenAndServe(":8080", nil))
}
```

**Dockerfile**
```dockerfile
FROM golang:1.22 as build
WORKDIR /src
COPY . .
RUN CGO_ENABLED=0 go build -o app

FROM gcr.io/distroless/base-debian12
COPY --from=build /src/app /app
USER 65532:65532
EXPOSE 8080
ENTRYPOINT ["/app"]
```

**Knative Service**
```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata: { name: hello-go }
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/target: "80"
        autoscaling.knative.dev/window: "10s"
    spec:
      containers:
      - image: ghcr.io/you/hello-go:1.0.0
        env: [{ name: TARGET, value: "Knative" }]
```

### OpenFaaS — Python 핸들러(동기/비동기 공용)

**handler.py**
```python
def handle(event, context):
    name = (event.body or "").strip() or "OpenFaaS"
    return f"Hello {name}"
```

**stack.yml**
```yaml
provider:
  name: openfaas
  gateway: http://gateway.openfaas:8080
functions:
  hello-ofn:
    lang: python
    handler: ./hello-ofn
    image: ghcr.io/you/hello-ofn:latest
    labels:
      com.openfaas.scale.min: "0"
      com.openfaas.scale.max: "20"
    annotations:
      com.openfaas.health.http.path: "/healthz"
```

**비동기 호출**
```bash
curl -X POST http://gateway:8080/async-function/hello-ofn -d "Alice"
```

---

## 선택 가이드 (실무 시나리오별)

| 시나리오 | 권장 |
|---|---|
| HTTP API 중심, Canary/Revision, 표준 이벤트 | **Knative** |
| 경량 자동화/엣지/빠른 PoC/쉬운 온보딩 | **OpenFaaS** |
| 대용량 스트림 처리, 토픽 레이턴시 기반 스케일 | Knative(Eventing+Kafka) **또는** OpenFaaS+KEDA |
| 콜드스타트 민감 핵심 API | Knative + `minScale` (=1) |
| 자원 절약 대기형 함수 | 둘 다 **scale-to-zero**, 코드·운영 취향대로 |

---

## 마무리

- **Knative**: K8s 원칙에 충실한 **Serving/Eventing 표준 플랫폼**, 트래픽 분할/리비전/CloudEvents에 강함.
- **OpenFaaS**: **경량성과 단순함**, 빠른 개발과 다양한 트리거·관측을 기본 제공.
- 공통의 성공 키: **슬림 컨테이너**, 적정 **concurrency/timeout**, **모니터링**, **보안(PSA/NetworkPolicy/TLS)**, **GitOps**.

**정답은 “둘 중 우리 팀의 운영 역량과 요구에 맞는 것”**입니다.
핵심 경로는 Knative, 주변 자동화는 OpenFaaS처럼 **혼합 전략**도 충분히 실용적입니다.

---

## 참고 링크

- Knative: <https://knative.dev/>
- OpenFaaS: <https://www.openfaas.com/>
- CloudEvents: <https://cloudevents.io/>
- KEDA: <https://keda.sh/>
- Knative Eventing Kafka: <https://knative.dev/docs/eventing/>
