---
layout: post
title: Kubernetes - Grafana + Prometheus
date: 2025-05-05 21:20:23 +0900
category: Kubernetes
---
# Grafana + Prometheus로 Kubernetes 모니터링 구축하기

## 1. 아키텍처 개요

```
+-------------------+       +-----------------------+       +----------------+
| Kubernetes Nodes  |  ---> | Prometheus (Operator) |  ---> | Grafana        |
|  kubelet/cAdvisor |  ---> |  TSDB, Rules, Alerts  |  ---> | Dashboards     |
|  kube-state-metrics| ---> | Alertmanager          |  ---> | (Auth/Orgs)    |
+-------------------+       +-----------------------+       +----------------+
         ^                             |
         |  scrape                     | alerts
         |                             v
         |                         Slack/Email/Webhook/...
```

- **kube-prometheus-stack**: Prometheus Operator 기반 통합 차트. Prometheus, Alertmanager, Grafana, node-exporter, kube-state-metrics 등이 함께 배포된다.
- **ServiceMonitor/PodMonitor**: Prometheus의 `scrape_configs`를 YAML CRD로 선언적으로 관리한다.
- **PrometheusRule**: Alerting/Recording 룰을 CRD로 선언한다.
- **Grafana**: Prometheus를 데이터 소스로 사용해 시각화한다.

---

## 2. 설치 방법 — Helm (권장)

### 2.1 리포지토리 추가

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

### 2.2 kube-prometheus-stack 설치

```bash
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace
```

설치되는 주요 컴포넌트:
- Prometheus, Alertmanager, Grafana
- node-exporter, kube-state-metrics
- ServiceMonitor/PodMonitor/PrometheusRule CRDs

상태 확인:

```bash
kubectl get pods -n monitoring
kubectl get svc  -n monitoring
```

---

## 3. 기본 접속

### 3.1 Grafana 접근

```bash
kubectl port-forward svc/prometheus-grafana -n monitoring 3000:80
# 브라우저에서 http://localhost:3000
```

기본 계정:

```bash
# 초기 암호 확인
kubectl get secret prometheus-grafana -n monitoring \
  -o jsonpath="{.data.admin-password}" | base64 -d; echo
```

ID: `admin`  
PW: 위 명령어 결과

### 3.2 Prometheus UI

```bash
kubectl port-forward svc/prometheus-kube-prometheus-prometheus -n monitoring 9090:9090
# http://localhost:9090
```

---

## 4. 값 커스터마이징(설치/업그레이드)

실전에서는 리텐션/퍼시스턴스/리소스/보안 등을 조정해야 한다. 예시 values:

```yaml
# values-monitoring.yaml (발췌)
grafana:
  adminPassword: "change-me"
  service:
    type: ClusterIP
  persistence:
    enabled: true
    size: 10Gi
  # 조직/데이터소스/대시보드 자동 프로비저닝 예시는 뒤 섹션 참조

prometheus:
  prometheusSpec:
    retention: 15d
    retentionSize: "20GB"
    storageSpec:
      volumeClaimTemplate:
        spec:
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 50Gi
    resources:
      requests:
        cpu: 500m
        memory: 2Gi
      limits:
        cpu: "2"
        memory: 4Gi

alertmanager:
  alertmanagerSpec:
    replicas: 2
    # 고가용성 예시

kubeStateMetrics:
  enabled: true

nodeExporter:
  enabled: true
```

적용:

```bash
helm upgrade --install prometheus prometheus-community/kube-prometheus-stack \
  -n monitoring -f values-monitoring.yaml --atomic --wait
```

---

## 5. ServiceMonitor / PodMonitor 로 커스텀 앱 수집

### 5.1 앱이 `/metrics` 노출

예: Flask

```python
# app.py
from flask import Flask, Response
from prometheus_client import Counter, generate_latest

app = Flask(__name__)
req_total = Counter('myapp_requests_total', 'Total HTTP requests')

@app.route("/")
def index():
    req_total.inc()
    return "ok"

@app.route("/metrics")
def metrics():
    return Response(generate_latest(), mimetype="text/plain")
```

K8s Service 예시:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
  labels:
    app.kubernetes.io/name: myapp
spec:
  selector:
    app.kubernetes.io/name: myapp
  ports:
    - name: http
      port: 8080
      targetPort: 8080
```

### 5.2 ServiceMonitor 정의

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: myapp
  labels:
    release: prometheus  # Prometheus 선택 라벨(차트 기본값과 맞춰야 함)
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: myapp
  namespaceSelector:
    matchNames: ["default"]
  endpoints:
    - port: http
      path: /metrics
      interval: 30s
      scrapeTimeout: 10s
```

- `release: prometheus` 라벨은 kube-prometheus-stack이 생성한 Prometheus 리소스가 자신이 감시할 ServiceMonitor를 **라벨 셀렉터로 선택**하기 위해 필요하다(차트 기본값 기준).

### 5.3 PodMonitor(사이드카/헤드리스 등 상황)

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: sidecar-metrics
  labels:
    release: prometheus
spec:
  namespaceSelector:
    any: true
  selector:
    matchLabels:
      metrics: "enabled"
  podMetricsEndpoints:
    - port: metrics
      path: /metrics
      interval: 30s
```

---

## 6. PromQL 실전 쿼리

노드/컨테이너/클러스터 기초:

```promql
# 컨테이너 CPU 사용률(초당)
rate(container_cpu_usage_seconds_total{container!=""}[5m])

# 컨테이너 메모리 사용량
container_memory_usage_bytes{container!=""}

# Pod 재시작 수
kube_pod_container_status_restarts_total

# 노드 CPU Idle 비율
avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[5m]))
```

특정 네임스페이스 CPU 사용률(코어):

```promql
sum by (namespace) (
  rate(container_cpu_usage_seconds_total{container!="",namespace!="",pod!=""}[5m])
)
```

컨테이너 메모리 워킹셋(네임스페이스별 합):

```promql
sum by (namespace) (
  container_memory_working_set_bytes{container!="",namespace!=""}
)
```

요청 대비 사용률:

```promql
# CPU 사용률(%): 사용/요청
100 * sum by (namespace) (
  rate(container_cpu_usage_seconds_total{container!="",pod!=""}[5m])
)
/
sum by (namespace) (
  kube_pod_container_resource_requests{resource="cpu"}
)
```

---

## 7. Recording Rules로 비용 절감 및 성능 향상

반복 계산되는 무거운 PromQL은 **Recording Rule**로 미리 계산해 저장한다.

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: recording-rules
  labels:
    release: prometheus
spec:
  groups:
    - name: workloads.rules
      interval: 1m
      rules:
        - record: ns:cpu_usage_seconds:rate5m
          expr: sum by (namespace) (rate(container_cpu_usage_seconds_total{container!="",pod!=""}[5m]))
        - record: ns:mem_workingset_bytes
          expr: sum by (namespace) (container_memory_working_set_bytes{container!="",pod!=""})
```

이제 대시보드/경보에서 `ns:cpu_usage_seconds:rate5m`, `ns:mem_workingset_bytes`를 직접 사용 가능.

---

## 8. Alerting — PrometheusRule + Alertmanager

### 8.1 경보 예시: 노드 CPU 고사용

{% raw %}
```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: node-alerts
  labels:
    release: prometheus
spec:
  groups:
    - name: node.alerts
      rules:
        - alert: NodeHighCpu
          expr: 100 - (avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 90
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "High CPU on {{ $labels.instance }}"
            description: "CPU usage > 90% for 10m"
```
{% endraw %}

### 8.2 Alertmanager 라우팅(예: Slack)

values에서 Alertmanager 설정을 오버라이드하거나 `Secret`로 구성:
- 라우팅 트리, 경보 묶음, 재알림 간격, 수신자(Webhook/Slack/Email) 설정

---

## 9. Grafana — 대시보드 가져오기/프로비저닝

### 9.1 UI에서 임포트
- grafana.com 대시보드 ID로 Import
- Kubernetes/Node/Pod/Cluster/etcd/Network 등 풍부한 카탈로그 활용

### 9.2 YAML 프로비저닝(권장)

데이터소스:

```yaml
# values-monitoring.yaml (발췌)
grafana:
  additionalDataSources:
    - name: Prometheus
      type: prometheus
      url: http://prometheus-kube-prometheus-prometheus.monitoring.svc:9090
      access: proxy
      isDefault: true
```

대시보드 프로비저닝:

```yaml
grafana:
  dashboardsConfigMaps:
    default: "grafana-dashboards"

# ConfigMap에 JSON 대시보드 저장
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-dashboards
  namespace: monitoring
  labels:
    grafana_dashboard: "1"
data:
  k8s-overview.json: |
    { ... Grafana JSON ... }
```

폴더/프로바이더 프로비저닝(파일 마운트 기반)도 가능하다. 운영에서는 **대시보드도 Git으로 관리**하고 CI에서 검증 후 반영하도록 하라.

---

## 10. SLO/에러버짓을 위한 수학적 정의

가용성 SLO와 에러버짓:

$$
\text{SLO}_{availability} = 1 - \frac{\text{5xx requests}}{\text{total requests}}
$$

월간 에러버짓:

$$
\text{Error Budget} = 1 - \text{SLO Target}
$$

예를 들어 SLO 99.9%인 경우, 에러버짓은 0.1%이다. 해당 기간 동안의 `rate(http_requests_total{status=~"5.."}[5m])` 와 전체 요청으로 계산해 SLO 위반을 경보로 만들 수 있다.

---

## 11. 보안, RBAC, 네트워크, 비밀 관리

- **네임스페이스 분리**: `monitoring` 전용
- **RBAC 최소화**: kube-state-metrics 권한 범위 확인
- **NetworkPolicy**: Prometheus/Grafana/노드 익스포터 등 간 통신 허용만 열기
- **Secret 관리**: Grafana admin PW, Alertmanager webhook 토큰 등은 Secret/외부 비밀 관리자
- **Ingress + TLS**: 외부 접근 시 Ingress, cert-manager로 TLS 적용
- **Grafana 인증**: OAuth(OIDC/GitHub/Google) 연동, Org/Folder/Team 권한 분리

---

## 12. 퍼시스턴스, 리텐션, 리소스/HPA, 비용 최적화

- Prometheus TSDB는 IOPS 영향이 크다. SSD/PD-SSD/EBS gp3 등 블록 스토리지 권장
- `prometheus.prometheusSpec.retention`(기간), `retentionSize`(용량)로 관리 비용 제어
- 불필요한 고카디널리티 라벨 제거(특히 컨테이너/Pod UID, 고유 요청 ID 등)
- Scrape interval 상향(예 30s→60s) 및 Recording Rule 적극 사용
- Grafana/Prometheus 리소스 Requests/Limits 합리적 설정, HPA로 대시보드 피크 대응

HPA 예시(외부 메트릭 생략, KEDA/Prometheus Adapter로 확장 가능):

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: grafana
  namespace: monitoring
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: prometheus-grafana
  minReplicas: 1
  maxReplicas: 5
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 80
```

---

## 13. 멀티클러스터/장기보관 — Thanos(요약)

- 각 클러스터 Prometheus에 Thanos Sidecar를 붙이고, 오브젝트 스토리지(S3/GCS)에 업로드
- 중앙에 Thanos Querier로 **수년 단위**의 장기 메트릭 집계/조회
- 다운샘플링으로 장기 쿼리 성능 개선
- Alerting은 로컬 Prometheus에서, 리포팅은 Thanos에서

---

## 14. End-to-End 예제 모음

### 14.1 커스텀 애플리케이션 지표 + ServiceMonitor + 대시보드

1) 앱 `/metrics` 노출(언어별 라이브러리)
2) Service/Deployment에 라벨 추가
3) ServiceMonitor로 스크랩 선언
4) Grafana에서 임포트/프로비저닝으로 패널 추가

### 14.2 경보: 네임스페이스별 CPU 사용률 과다

{% raw %}
```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: ns-cpu-alerts
  labels:
    release: prometheus
spec:
  groups:
    - name: ns.cpu.alerts
      rules:
        - alert: NamespaceHighCPU
          expr: 100 * (ns:cpu_usage_seconds:rate5m) /
                sum by (namespace) (kube_pod_container_resource_requests{resource="cpu"}) > 200
          for: 15m
          labels:
            severity: critical
          annotations:
            summary: "High CPU usage in {{ $labels.namespace }}"
            description: "Namespace CPU usage > 200% of requests for 15m"
```
{% endraw %}

전제: 위 `ns:cpu_usage_seconds:rate5m`는 Recording Rule로 정의되어 있다고 가정.

---

## 15. 운영 트러블슈팅

- ServiceMonitor가 안 먹으면:
  - `metadata.labels.release` 가 Prometheus 셀렉터와 일치하는지
  - `selector.matchLabels` 가 타깃 Service와 일치하는지
  - HTTPS/BasicAuth/Bearer/Relabeling 필요 여부
- 메트릭이 안 보이면:
  - `/metrics` 실제 응답 확인(`kubectl port-forward` 후 curl)
  - 포트/경로/네임스페이스 셀렉터 재검토
- 고카디널리티 폭증:
  - 라벨 정리, `metric_relabel_configs`로 드롭
  - 샘플링 간격 상향, 룰 계산 주기 완화
- Grafana 패널 느림:
  - Recording Rule 사용
  - 시간 범위 축소, 레전드 단순화, 변수/정규식 최소화
- 디스크 가득 참:
  - 리텐션/용량 상향 조정, Thanos 도입 검토

---

## 16. 실습 커맨드 모음

```bash
# 설치
helm upgrade --install prometheus prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace -f values-monitoring.yaml --atomic --wait

# 기본 포워딩
kubectl port-forward svc/prometheus-grafana -n monitoring 3000:80
kubectl port-forward svc/prometheus-kube-prometheus-prometheus -n monitoring 9090:9090

# CRDs 리소스 확인
kubectl get servicemonitors.monitoring.coreos.com -A
kubectl get podmonitors.monitoring.coreos.com -A
kubectl get prometheusrules.monitoring.coreos.com -A

# 룰/알람 디버깅
kubectl -n monitoring exec -it deploy/prometheus-kube-prometheus-operator -- \
  sh -c 'curl -s localhost:8080/metrics | head -n 20'
```

---

## 17. 예제 대시보드 패널 PromQL 레시피

- 노드별 CPU 사용률(%):

```promql
100 - (avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

- Pod 재시작 상위 10:

```promql
topk(10, increase(kube_pod_container_status_restarts_total[1h]))
```

- 네임스페이스별 메모리 워킹셋(GB):

```promql
sum by (namespace) (container_memory_working_set_bytes{container!=""}) / 1024^3
```

- 노드 파일시스템 사용률(%):

```promql
100 * (1 - (node_filesystem_avail_bytes{fstype!~"tmpfs|overlay"} /
            node_filesystem_size_bytes{fstype!~"tmpfs|overlay"}))
```

---

## 18. 참고 구조: Blackbox Exporter(HTTP 핑)

외부 엔드포인트 가용성도 Prometheus에 수집:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: blackbox-exporter
  labels:
    app: blackbox
spec:
  ports:
    - name: http
      port: 9115
  selector:
    app: blackbox
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: blackbox
  labels:
    release: prometheus
spec:
  endpoints:
    - port: http
      path: /probe
      params:
        module: [http_2xx]
      interval: 30s
      scrapeTimeout: 10s
      metricRelabelings:
        - sourceLabels: [__address__]
          targetLabel: instance
  namespaceSelector:
    matchNames: ["monitoring"]
  selector:
    matchLabels:
      app: blackbox
```

---

## 19. 결론

- **kube-prometheus-stack**으로 빠르게 통합 모니터링 스택을 올리고,
- **ServiceMonitor/PodMonitor/PrometheusRule**로 지표·경보를 선언적으로 관리하며,
- **Recording Rule**과 적절한 **리텐션/퍼시스턴스**로 성능/비용을 균형 있게 유지하고,
- **Grafana 프로비저닝**으로 대시보드/데이터소스를 코드로 관리하라.
- 멀티클러스터/장기보관이 필요하면 **Thanos**를 검토하라.

이 가이드를 기반으로 **클러스터 상태를 시각적으로 파악**하고, **병목을 조기 탐지**하며, **SLO 중심 운영**을 구현할 수 있다.

---

## 부록 A. 최소 재현 세트(붙여넣기용)

### A.1 설치 커맨드

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

helm install prometheus prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace
```

### A.2 샘플 ServiceMonitor

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: sample
  labels:
    release: prometheus
spec:
  namespaceSelector:
    matchNames: ["default"]
  selector:
    matchLabels:
      app.kubernetes.io/name: myapp
  endpoints:
    - port: http
      path: /metrics
      interval: 30s
```

### A.3 샘플 Alert(노드 CPU)

{% raw %}
```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: node-high-cpu
  labels:
    release: prometheus
spec:
  groups:
    - name: node.alerts
      rules:
        - alert: NodeHighCpu
          expr: 100 - (avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 90
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "High CPU on {{ $labels.instance }}"
```
{% endraw %}

### A.4 Grafana 포트포워드

```bash
kubectl port-forward svc/prometheus-grafana -n monitoring 3000:80
```

이후 http://localhost:3000 접속.