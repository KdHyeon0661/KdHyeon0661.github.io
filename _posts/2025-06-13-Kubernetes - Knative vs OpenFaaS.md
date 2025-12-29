---
layout: post
title: Kubernetes - Knative vs OpenFaaS
date: 2025-06-13 19:20:23 +0900
category: Kubernetes
---
# K8s 기반 서버리스: Knative vs OpenFaaS

**서버리스(Serverless)** 의 핵심은 코드 작성에만 집중하고, 인프라의 운영과 관리 책임을 플랫폼에 맡기는 것입니다. Kubernetes 생태계에서도 이러한 패러다임을 구현하는 오픈소스 솔루션이 존재하며, 그 대표주자가 바로 **Knative**와 **OpenFaaS**입니다.

---

## 한눈 비교 (의사결정 표)

| 구분 | **Knative** | **OpenFaaS** |
|---|---|---|
| 철학/포지션 | K8s 네이티브(Serving/Eventing), **HTTP 및 이벤트 처리 표준화** | 경량 **Function 프레임워크**, 쉬운 시작과 사용 |
| 설치 난이도 | 중간~높음 (Ingress/Gateway 구성 필요) | 낮음~중간 (Helm 기반 간단 설치) |
| 확장/스케일링 | 기본 **scale-to-zero** 지원, `Activator`/`queue-proxy` 아키텍처 | **scale-to-zero 가능**, KEDA와의 통합성이 뛰어남 |
| 이벤트 처리 | **Knative Eventing(CloudEvents 표준)** | **Connectors** (Kafka, NATS, Cron 등 다양한 트리거) |
| 빌드 방식 | 컨테이너 이미지를 전제 (외부 빌드 시스템/Buildpacks/Tekton 활용) | 핸들러 코드 중심(템플릿, `faas-cli` 도구) |
| 모니터링 | Prometheus/Grafana와 연동 (별도 구성 필요) | 내장된 Prometheus, 대시보드 구성이 용이 |
| 트래픽 라우팅 | **Revision/Traffic Split/Canary 배포** | Gateway 라우팅, Canary는 Helm/K8s 기본 기능으로 구현 |
| 학습 곡선 | **CRD와 개념이 많아** 초기 진입 장벽이 있음 | 낮음 (코드 작성 → `faas-cli up` 명령으로 배포) |
| 적합한 사례 | HTTP 기반 API, CloudEvents, GCR/Cloud Run과 친화적인 환경 | PoC/엣지 컴퓨팅/경량 자동화, 다양한 트리거 필요 시 |

---

## Kubernetes 기반 서버리스의 가치

- **벤더 종속성 회피**: 특정 클라우드의 FaaS 서비스 대신 **이식성 높은 Kubernetes** 플랫폼 위에서 동일한 서버리스 패턴을 구현할 수 있습니다.
- **마이크로서비스와 GitOps**: 선언적 배포, Canary/Blue-Green 전략, 자동 롤백 등 현대적인 배포 방식을 활용할 수 있습니다.
- **이벤트 주도 아키텍처**: 메시지 스트림, 이벤트 버스, 크론잡, 웹훅 등 다양한 소스로부터 워크플로를 자동화할 수 있습니다.

### 오토스케일링의 간단한 원리

서버리스 플랫폼은 일반적으로 다음 근사치를 바탕으로 레플리카 수를 결정합니다.
```
원하는 레플리카 수 ≈ (들어오는 초당 요청 수 × 평균 요청 지연 시간) / 파드당 처리 가능 동시성
```
- **Knative**: 요청 동시성(concurrency)과 응답 시간 기반의 **KPA**(Knative Pod Autoscaler) 또는 표준 K8s HPA를 사용합니다.
- **OpenFaaS**: 내장 Prometheus 메트릭 또는 KEDA를 통한 사용자 정의 메트릭(예: Kafka Lag)을 기반으로 스케일링합니다.

---

## Knative — Serving과 Eventing으로 이해하는 구조

### 아키텍처 핵심

- **Serving**: `Service` 리소스가 `Configuration`을 생성하고, 이를 기반으로 불변의 **`Revision`** 이 만들어집니다. `Route`는 트래픽을 특정 Revision으로 분배합니다.
  외부 트래픽은 Ingress(예: **Kourier**, Istio)를 통해 유입되며, **Activator** 컴포넌트가 scale-to-zero 상태의 함수를 깨우고(wake up), **queue-proxy** 사이드카가 파드별 트래픽 큐와 동시성을 관리합니다.
- **Eventing**: **Broker/Trigger** 모델과 **CloudEvents** 표준을 기반으로, Kafka 등 다양한 이벤트 소스를 통합합니다.

```
[Client] → [Ingress(Kourier/Istio)] → [Activator] → [Service/Revision Pods]
                                          ↘ queue-proxy → User Container
```

### 설치 요약

```bash
# Serving CRDs 및 코어 컴포넌트 설치
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.13.0/serving-crds.yaml
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.13.0/serving-core.yaml

# 경량 게이트웨이인 Kourier 설치 및 설정
kubectl apply -f https://github.com/knative/net-kourier/releases/download/knative-v1.13.0/kourier.yaml
kubectl patch configmap/config-network -n knative-serving \
  -p='{"data":{"ingress.class":"kourier.ingress.networking.knative.dev"}}'
```

> 프로덕션 환경에서는 Ingress에 Load Balancer 연결, DNS 설정, TLS 관리(cert-manager), HPA/KPA 파라미터 튜닝이 필수적입니다.

### 간단한 Knative Service 예제

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
        autoscaling.knative.dev/target: "100"      # 파드당 목표 동시 처리 요청 수
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
kubectl get ksvc hello-knative # URL 필드를 확인하세요.
```

### Revision 기반 Canary/Traffic Split

```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: web
spec:
  template:
    spec:
      containers:
      - image: ghcr.io/you/web:v2 # 새 Revision 생성
  traffic:
  - revisionName: web-00001       # 기존 Revision
    percent: 80                    # 트래픽의 80% 유지
  - latestRevision: true           # 새 Revision (위의 v2)
    percent: 20                    # 트래픽의 20% 전환
```

### Knative Eventing 예제 (Broker/Trigger)

**네임스페이스에 기본 Broker 설치**
```bash
kubectl apply -f - <<'YAML'
apiVersion: eventing.knative.dev/v1
kind: Broker
metadata:
  name: default
  namespace: events
YAML
```

**Trigger: 특정 유형의 이벤트를 서비스로 전달**
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
      type: com.example.hello # 이 유형의 이벤트만 필터링
  subscriber:
    ref:
      apiVersion: serving.knative.dev/v1
      kind: Service
      name: hello-knative
      namespace: default
```

**CloudEvents 형식으로 이벤트 전송**
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

## OpenFaaS — 경량 함수 프레임워크

### 주요 구성 요소

- **Gateway**(UI+API), **faas-netes**(Kubernetes 백엔드 컨트롤러), **Prometheus**(기본 내장), **queue-worker**(비동기 호출 처리)
- **Function Watchdog**: 컨테이너를 함수처럼 실행하는 HTTP 또는 STDIN/STDOUT 어댑터입니다.

### Helm을 이용한 설치

```bash
helm repo add openfaas https://openfaas.github.io/faas-netes/
helm upgrade openfaas --install openfaas/openfaas \
  --namespace openfaas --create-namespace \
  --set basic_auth=true
# 관리자 비밀번호 확인
kubectl -n openfaas get secret basic-auth -o jsonpath='{.data.basic-auth-password}' | base64 -d; echo
```

### 함수 생성부터 배포까지 (Python 예제)

```bash
# Python 템플릿을 사용해 새로운 함수 스캐폴드 생성
faas-cli new hello-openfaas --lang python
# 생성된 handler.py 파일을 편집한 후

# 빌드, 푸시, 배포를 한번에 실행
faas-cli up -f hello-openfaas.yml
# 함수 호출
curl http://<gateway-external-ip>:8080/function/hello-openfaas
```

**`hello-openfaas.yml` 파일 내용**
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
      com.openfaas.scale.zero: "true" # scale-to-zero 활성화
    labels:
      com.openfaas.scale.max: "10"
      com.openfaas.scale.min: "0"
```

### 이벤트 트리거 (크론, 카프카, 비동기 호출)

- **Cron**: `faas-netes` 커넥터(cron-connector)를 통해 스케줄링.
- **Kafka/NATS/MQTT**: 해당 커넥터를 배포 후, 특정 토픽의 메시지를 함수로 라우팅.
- **비동기 호출**: `POST /async-function/<함수명>` 엔드포인트로 호출하면 queue-worker가 메시지를 큐에 넣고 순차적으로 처리합니다.

---

## 스케일링, 콜드 스타트, 지연 시간 최적화

### Knative 오토스케일링

- **KPA**: 주로 요청 동시성(`autoscaling.knative.dev/target`)을 기반으로 스케일링하며, `window`, `scaleDownDelay` 등의 어노테이션으로 행동을 제어할 수 있습니다.
- 표준 Kubernetes **HPA**를 백엔드로 사용하는 옵션도 있습니다.
- **minScale**을 설정해 콜드스타트를 완전히 방지하거나, **containerConcurrency**로 파드당 동시 처리량을 정밀 제어할 수 있습니다.

콜드스타트가 성능에 치명적인 애플리케이션의 경우:
```yaml
metadata:
  annotations:
    autoscaling.knative.dev/minScale: "1"          # 항상 최소 1개의 파드 유지
    autoscaling.knative.dev/scaleDownDelay: "300s" # 유휴 상태에서 파드 종료를 5분 지연
spec:
  template:
    spec:
      containerConcurrency: 2 # 이 파드는 동시에 최대 2개의 요청만 처리
```

### OpenFaaS 스케일링

- 기본적으로 Prometheus 메트릭(호출 횟수 등)에 기반하며, `com.openfaas.scale.*` 레이블로 제한을 설정합니다.
- **KEDA 연동**을 통해 Kafka 지연 lag, 메시지 큐 길이 등 복잡한 지표를 기반으로 스케일링할 수 있습니다.
- watchdog의 `read_timeout`, `write_timeout`, `exec_timeout` 옵션을 조정해 함수의 타임아웃과 안정성을 관리합니다.

---

## 관측 가능성(Observability)

### Knative 모니터링

- **요청/응답/동시성 메트릭**: queue-proxy와 Activator 컴포넌트가 Prometheus 형식의 메트릭을 노출합니다.
- 대시보드: kube-prometheus-stack과 함께 사용하며, `queue_requests_per_second`, `container_concurrency`와 같은 메트릭을 위한 커스텀 Grafana 패널을 구성할 수 있습니다.

PromQL 예시:
```promql
sum(rate(queue_requests_total{namespace="default", service="hello-knative"}[1m]))
```

### OpenFaaS 모니터링

- **내장된 Prometheus**가 `gateway_function_invocation_total`, `gateway_functions_seconds_sum` 등의 함수별 메트릭을 자동으로 수집합니다.
- 공식적으로 제공되는 풍부한 Grafana 대시보드 템플릿을 쉽게 임포트할 수 있습니다.

---

## 보안, 네트워크, 비용 관점

### 공통 고려사항

- **PSA(Pod Security Admission)**: 네임스페이스에 `restricted` 레벨을 적용하고, Cosign으로 이미지 서명, Trivy로 취약점 스캔을 도입하세요.
- **TLS**: Ingress 레벨에서 cert-manager를 활용한 자동 인증서 관리, 서비스 메시나 SPIFFE/SPIRE를 이용한 내부 **mTLS**를 고려하세요.
- **네트워크 격리**: `NetworkPolicy`를 적용해 함수 간 불필요한 동서 트래픽을 제한하세요.

함수 네임스페이스에서 특정 대상으로만 아웃바운드를 허용하는 예시:
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
          name: kafka # 'kafka' 라벨이 있는 네임스페이스만 허용
    ports:
    - port: 9092
      protocol: TCP
```

### 비용 효율성 (Scale-to-zero의 가치)

- 장시간 유휴 상태인 함수는 **scale-to-zero**가 노드 리소스를 크게 절약합니다.
- 반대로 **콜드스타트 지연 시간**이 서비스 수준 협약(SLA)에 영향을 미치는 핵심 함수라면, `minScale`을 1 이상으로 설정해 항상 웜(warm) 상태를 유지하는 절충안을 선택할 수 있습니다.

---

## CI/CD 및 GitOps 통합

### Knative와 ArgoCD

- `ksvc`(Knative Service), `Broker`, `Trigger` 등의 리소스를 **Git 레포지토리에 선언형으로 저장**하고 ArgoCD가 클러스터 상태를 자동 동기화하게 합니다.
- Canary 배포는 `traffic` 필드의 퍼센티지를 변경하는 PR을 통해 점진적으로 수행할 수 있습니다.

### OpenFaaS와 GitHub Actions

- `faas-cli build/push/deploy` 명령어를 파이프라인에 통합합니다.
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

## 실전 아키텍처 레퍼런스

### HTTP API + Canary 배포 (Knative 중심)

- 아키텍처: Kourier(Ingress LB) + Knative Serving
- 전략: 핵심 경로 API는 `minScale=1`로 웜 상태 유지, 부수적 함수는 scale-to-zero 적용
- 배포: **Revision 기반 Traffic Split**으로 10% → 30% → 100% 점진적 롤아웃

### 이벤트 기반 스트림 처리 파이프라인

- **Knative Eventing + Kafka**: CloudEvents 표준으로 이벤트 라우팅 및 Dead Letter Queue(DLQ) 설정
- **OpenFaaS + KEDA(Kafka Scaler)**: 토픽의 메시지 지연(Lag)을 지표로 자동 스케일링

### 엣지/온프레미스 경량 자동화

- 경량 Kubernetes인 K3s 위에 **OpenFaaS** 배포
- Cron 커넥터나 웹훅을 이용한 로컬 작업 자동화, 내장 Prometheus로 기본 관측 가능

---

## 문제 해결 가이드

### Knative 주요 문제점

- **함수에 접근 불가**: 대상 도메인의 DNS 설정, Ingress Controller의 Load Balancer 상태, 헬스 체크를 확인하세요.
- **콜드스타트 지연 시간이 너무 김**: `minScale` 값을 조정하고, 컨테이너 이미지를 슬림하게 최적화하며, 애플리케이션 초기화 로직을 검토하세요.
- **4xx/5xx 오류 발생**: `queue-proxy`, `activator`, `ingress` 컴포넌트의 로그를 순차적으로 확인하세요. (`kubectl -n knative-serving logs ...`)
- **이벤트 전달 실패**: Broker와 Trigger의 상태(`kubectl get broker,trigger`), 재시도(delivery) 파라미터 설정을 점검하세요.

### OpenFaaS 주요 문제점

- **권한 또는 인증 실패**: `faas-cli login` 명령어로 게이트웨이 인증 상태를 재확인하세요.
- **비동기 호출 지연**: `queue-worker` 파드의 로그와 NATS/Kafka 커넥터의 상태를 확인하세요.
- **함수가 스케일링되지 않음**: Prometheus 타겟이 정상적인지, KEDA를 사용한다면 ScaledObject 리소스의 설정을 검토하세요.

---

## 고급 성능 튜닝 팁

### Knative 튜닝

- `containerConcurrency` 값을 낮추면 개별 요청의 지연 시간 안정성이 높아지지만, 지나치게 낮추면 동일 트래픽을 처리하기 위해 파드 수가 과도하게 늘어날 수 있습니다.
- `autoscaling.knative.dev/targetUtilizationPercentage` 어노테이션을 사용해 목표 사용률을 조정하여 리소스 여유율을 제어할 수 있습니다.
- `readinessProbe`를 정확하게 설정하면 새 Revision 롤아웃 중 발생하는 5xx 오류를 줄일 수 있습니다.

### OpenFaaS 튜닝

- watchdog의 `read_timeout`, `write_timeout`, `exec_timeout` 값을 애플리케이션 특성에 맞게 조정하여 타임아웃 오류를 방지하세요.
- 함수 템플릿을 커스터마이징해 최적화된 베이스 이미지(예: uvicorn-gunicorn 기반 Python, tini init 프로세스, distroless 이미지)를 사용하세요.
- 기본 `basic_auth` 외에 OIDC(OpenID Connect)를 게이트웨이와 연동해 기업 SSO를 적용할 수 있습니다.

---

## 실전 코드 예제 모음

### Knative — 간단한 Go API 핸들러

**main.go**
```go
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

**Knative Service 배포 매니페스트**
```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: hello-go
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/target: "80"
        autoscaling.knative.dev/window: "10s"
    spec:
      containers:
      - image: ghcr.io/you/hello-go:1.0.0
        env:
        - name: TARGET
          value: "Knative"
```

### OpenFaaS — Python 핸들러 (동기/비동기 공용)

**handler.py**
```python
def handle(event, context):
    name = (event.body or "").strip() or "OpenFaaS"
    return {
        "statusCode": 200,
        "body": f"Hello {name}"
    }
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

**비동기 방식으로 함수 호출**
```bash
curl -X POST http://gateway:8080/async-function/hello-ofn -d "Alice"
```

---

## 선택 가이드라인 (실무 시나리오별)

| 시나리오 | 권장 솔루션 |
|---|---|
| HTTP API 중심, Canary/Revision 관리 필요, 표준 이벤트 시스템 통합 | **Knative** |
| 경량 자동화/엣지 컴퓨팅/빠른 개념 검증(PoC)/쉬운 시작 | **OpenFaaS** |
| 대용량 스트림 처리, 토픽 레이턴시 기반 오토스케일링 | Knative(Eventing+Kafka) **또는** OpenFaaS+KEDA |
| 콜드스타트 지연이 허용되지 않는 핵심 API | Knative + `minScale: 1` 설정 |
| 자원 절약이 중요한 대기형 함수 | 둘 다 **scale-to-zero** 지원, 팀의 코드 및 운영 선호도에 따라 선택 |

---

## 결론

- **Knative**는 Kubernetes의 설계 철학을 따르는 **표준화된 Serving/Eventing 플랫폼**으로, 정교한 트래픽 분할, 리비전 관리, CloudEvents 표준 준수에 강점이 있습니다.
- **OpenFaaS**는 **경량성과 사용의 단순함**을 우선시하며, 빠른 개발 사이클과 다양한 트리거, 기본 제공되는 관측 도구를 특징으로 합니다.
- 두 플랫폼 모두 성공적인 운영을 위해 **슬림한 컨테이너 이미지**, 적절한 **동시성 및 타임아웃 설정**, **종합적인 모니터링**, **보안 정책(PSA/NetworkPolicy/TLS)**, **GitOps 워크플로**의 도입이 핵심입니다.

최종 선택은 **“팀의 운영 전문성과 프로젝트의 구체적인 요구사항에 더 잘 부합하는 것”**에 따라 결정해야 합니다. 핵심 비즈니스 경로에는 Knative를, 주변 배치 작업이나 자동화에는 OpenFaaS를 사용하는 **혼합(Multi) 전략** 역시 현실적이고 효과적인 접근법이 될 수 있습니다.
