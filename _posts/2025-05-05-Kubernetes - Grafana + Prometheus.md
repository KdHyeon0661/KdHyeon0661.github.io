---
layout: post
title: Kubernetes - Grafana + Prometheus
date: 2025-05-05 21:20:23 +0900
category: Kubernetes
---
# Grafana + Prometheus로 Kubernetes 모니터링 구축하기

## 아키텍처 개요

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

Kubernetes 클러스터의 포괄적인 모니터링을 구축하기 위해 Prometheus와 Grafana 조합이 산업계 표준으로 자리 잡았습니다. 이 아키텍처에서 각 노드의 리소스 메트릭은 kubelet과 cAdvisor를 통해 수집되며, 클러스터 수준의 오브젝트 상태는 kube-state-metrics가 제공합니다. Prometheus는 이 모든 메트릭을 주기적으로 수집(Scrape)하여 시계열 데이터베이스(TSDB)에 저장합니다. 저장된 데이터는 사용자 정의 경보 규칙(PrometheusRule)을 통해 분석되고, 중요한 이벤트는 Alertmanager를 통해 다양한 채널로 통지됩니다. 최종적으로 Grafana는 Prometheus를 데이터 소스로 연결하여 대시보드를 통해 직관적인 시각화를 제공합니다.

**kube-prometheus-stack** Helm 차트는 이 모든 컴포넌트를 통합하여 관리하기 쉽게 패키징했습니다. Prometheus Operator는 ServiceMonitor, PodMonitor, PrometheusRule과 같은 커스텀 리소스 정의(CRD)를 활용해 모니터링 구성을 선언적으로 관리할 수 있게 해줍니다.

---

## 설치: Helm을 통한 빠른 시작

### 저장소 추가 및 설치
가장 일반적이고 권장되는 방법은 Helm을 사용하는 것입니다.

```bash
# 공식 Helm 저장소 추가
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# kube-prometheus-stack 설치
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace
```

이 명령어는 `monitoring` 네임스페이스를 생성하고 다음 주요 컴포넌트를 배포합니다:
- **Prometheus**: 메트릭 수집 및 저장 엔진
- **Alertmanager**: 경보 통합 및 라우팅 관리
- **Grafana**: 메트릭 시각화 대시보드
- **node-exporter**: 노드 수준의 하드웨어 및 OS 메트릭 수집기
- **kube-state-metrics**: Kubernetes 오브젝트 상태를 메트릭으로 변환

배포 상태는 아래 명령어로 확인할 수 있습니다.

```bash
kubectl get pods,svc -n monitoring
```

---

## 초기 접근 및 기본 구성

### Grafana 대시보드 접속
Grafana는 클러스터 내부의 Service로 노출됩니다. 로컬에서 테스트하기 위해 포트 포워딩을 사용할 수 있습니다.

```bash
# Grafana 서비스 포트 포워딩 (로컬 3000포트로 연결)
kubectl port-forward svc/prometheus-grafana -n monitoring 3000:80
```
이제 브라우저에서 `http://localhost:3000`에 접속할 수 있습니다. 초기 로그인 정보는 다음과 같이 확인합니다.

```bash
# 관리자 비밀번호 확인
kubectl get secret prometheus-grafana -n monitoring \
  -o jsonpath="{.data.admin-password}" | base64 -d; echo
```
기본 사용자 이름은 `admin`이며, 위 명령어로 출력된 비밀번호를 사용합니다.

### Prometheus UI 접근
Prometheus 자체의 웹 UI에도 접근할 수 있습니다.

```bash
kubectl port-forward svc/prometheus-kube-prometheus-prometheus -n monitoring 9090:9090
```
`http://localhost:9090`에서 수집 대상 상태, 저장된 메트릭 쿼리(PromQL), 경보 규칙 등을 확인할 수 있습니다.

---

## 실전을 위한 값(Values) 커스터마이징

기본 설치만으로는 프로덕션 환경에서 부족할 수 있습니다. 데이터 보존 기간, 영구 저장소, 리소스 할당량 등을 맞춤 구성해야 합니다. Helm `values.yaml` 파일을 통해 이러한 설정을 제어할 수 있습니다.

```yaml
# values-monitoring.yaml
grafana:
  # 기본 관리자 비밀번호 변경
  adminPassword: "secure-password-here"
  service:
    type: ClusterIP # 또는 필요시 LoadBalancer/NodePort
  persistence:
    enabled: true   # 데이터 손실 방지를 위한 영구 볼륨
    size: 10Gi

prometheus:
  prometheusSpec:
    retention: 15d          # 데이터 보존 기간
    retentionSize: "30GB"   # 최대 저장 용량
    storageSpec:
      volumeClaimTemplate:
        spec:
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 50Gi # Prometheus TSDB용 충분한 저장 공간
    resources:
      requests:
        cpu: 500m
        memory: 2Gi
      limits:
        cpu: "2"
        memory: 4Gi

alertmanager:
  alertmanagerSpec:
    replicas: 2 # 고가용성을 위한 복제본
```

이 사용자 정의 값 파일을 적용하려면 업그레이드 명령을 사용합니다.

```bash
helm upgrade --install prometheus prometheus-community/kube-prometheus-stack \
  -n monitoring -f values-monitoring.yaml --atomic --wait
```

---

## 커스텀 애플리케이션 모니터링 통합

자체 개발 애플리케이션의 메트릭도 Prometheus로 수집하려면 몇 가지 단계가 필요합니다.

### 1. 애플리케이션에서 메트릭 노출
먼저 애플리케이션이 Prometheus 형식의 메트릭을 HTTP 엔드포인트(일반적으로 `/metrics`)에서 제공해야 합니다. 대부분의 프로그래밍 언어에는 Prometheus 클라이언트 라이브러리가 존재합니다.

### 2. Kubernetes Service 정의
애플리케이션의 Pod를 가리키는 Service가 필요합니다. 이 Service는 Prometheus가 스크랩 대상을 찾는 데 사용됩니다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
  labels:
    app: myapp
spec:
  selector:
    app: myapp
  ports:
    - name: web
      port: 8080
      targetPort: 8080
```

### 3. ServiceMonitor CRD 생성
Prometheus Operator는 기존의 복잡한 `scrape_configs` 대신 선언적인 ServiceMonitor 리소스를 사용합니다.

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: myapp-monitor
  labels:
    release: prometheus # 이 라벨이 중요: Prometheus가 선택하는 기준
spec:
  selector:
    matchLabels:
      app: myapp        # 위 Service의 라벨과 매칭
  endpoints:
    - port: web         # Service에 정의된 포트 이름
      path: /metrics    # 메트릭 엔드포인트 경로
      interval: 30s     # 수집 주기
```

`release: prometheus` 라벨은 kube-prometheus-stack이 배포한 Prometheus 인스턴스가 이 ServiceMonitor를 선택하도록 하는 키입니다. ServiceMonitor가 생성되면 Prometheus는 자동으로 구성을 다시 로드하고 지정된 대상에서 메트릭 수집을 시작합니다.

Pod 자체를 직접 모니터링해야 하는 경우(예: 사이드카 컨테이너)에는 `PodMonitor` 리소스를 사용할 수 있습니다.

---

## 데이터 분석: 실용적인 PromQL 쿼리

Prometheus의 강력한 쿼리 언어인 PromQL을 사용하면 수집된 메트릭에서 인사이트를 추출할 수 있습니다.

**기본 리소스 사용률:**
```promql
# 네임스페이스별 CPU 사용률 (코어 단위)
sum by (namespace) (rate(container_cpu_usage_seconds_total{container!=""}[5m]))

# 네임스페이스별 메모리 사용량 (바이트 단위)
sum by (namespace) (container_memory_working_set_bytes{container!=""})

# Pod 재시작 횟수
sum by (namespace, pod) (kube_pod_container_status_restarts_total)
```

**상대적 사용률 및 효율성:**
```promql
# CPU 요청(Request) 대비 실제 사용률 (%)
100 * sum by (namespace, pod) (rate(container_cpu_usage_seconds_total{container!=""}[5m]))
/
sum by (namespace, pod) (kube_pod_container_resource_requests{resource="cpu"})

# 노드별 평균 CPU 사용률 (%)
100 - (avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

이러한 쿼리는 Grafana 대시보드의 패널에 직접 사용하거나, 더 나은 성능을 위해 Recording Rule로 변환할 수 있습니다.

---

## 성능 최적화: Recording Rule 활용

자주 실행되거나 계산 비용이 높은 쿼리는 Recording Rule로 미리 계산해 두어 Prometheus 서버의 부하를 줄이고 대시보드 응답 속도를 개선할 수 있습니다.

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: precomputed-rules
  labels:
    release: prometheus
spec:
  groups:
    - name: precomputed.rules
      interval: 1m  # 이 주기로 규칙을 평가/저장
      rules:
        - record: namespace:cpu_usage:rate5m
          expr: sum by (namespace) (rate(container_cpu_usage_seconds_total{container!=""}[5m]))
        - record: namespace:memory_working_set:bytes
          expr: sum by (namespace) (container_memory_working_set_bytes{container!=""})
```

이제 대시보드나 다른 경보 규칙에서 복잡한 원본 쿼리 대신 `namespace:cpu_usage:rate5m`과 같은 간단한 메트릭 이름을 직접 참조할 수 있습니다.

---

## 사전 예방적 운영: Alertmanager를 통한 경보

문제가 발생한 후 대응하는 것보다 발생하기 전에 감지하는 것이 더 효과적입니다. Prometheus Rule을 정의하여 특정 조건(예: 높은 CPU 사용량, 빈번한 재시작)을 감시하고, Alertmanager를 통해 적절한 채널로 알림을 보낼 수 있습니다.

### 경보 규칙 정의 예시
{% raw %}
```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: resource-alerts
  labels:
    release: prometheus
spec:
  groups:
    - name: resource.rules
      rules:
        - alert: HighNodeCPU
          expr: 100 - (avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 85
          for: 10m  # 이 조건이 10분간 지속될 때만 경보 발송
          labels:
            severity: warning
          annotations:
            summary: "노드 CPU 사용률이 높습니다 - {{ $labels.instance }}"
            description: "노드 {{ $labels.instance }}의 CPU 사용률이 85%를 10분 이상 초과했습니다."
```
{% endraw %}

### Alertmanager 구성
Alertmanager는 경보를 수신하여 중복 제거, 그룹화, 음소거 처리한 후 최종 수신자(Slack, 이메일, PagerDuty, 웹훅 등)로 라우팅합니다. 구성은 일반적으로 Helm values를 통해 또는 Secret 리소스로 제공됩니다.

---

## 효과적인 시각화: Grafana 대시보드 관리

Grafana의 강점은 풍부한 시각화 옵션과 활발한 커뮤니티입니다.

**대시보드 임포트:** [Grafana 공식 대시보드 사이트](https://grafana.com/grafana/dashboards)에는 Kubernetes, 노드, Prometheus 자체를 모니터링하는 수천 개의 사전 제작된 대시보드가 있습니다. 대시보드 ID(예: `3119` for Kubernetes cluster monitoring)를 사용하여 Grafana UI 내에서 쉽게 가져올 수 있습니다.

**코드로서의 대시보드:** 프로덕션 환경에서는 대시보드 구성을 Git에서 관리하는 것이 좋습니다. 이를 위해 Grafana의 "프로비저닝" 기능을 사용할 수 있습니다. 대시보드 JSON 정의를 ConfigMap에 저장하고, Helm values를 통해 Grafana에 마운트하도록 지시할 수 있습니다.

```yaml
# values-monitoring.yaml 예시 추가
grafana:
  dashboardsConfigMaps:
    default: "grafana-dashboards" # ConfigMap 이름 참조

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-dashboards
  namespace: monitoring
data:
  kubernetes-cluster.json: |  # 파일 이름이 중요
    { "dashboard": { ... JSON 정의 ... }, "folderTitle": "Kubernetes" }
```
이 방식으로 배포 시 대시보드가 자동으로 생성되며, 버전 관리와 코드 리뷰의 혜택을 누릴 수 있습니다.

---

## 고급 주제 및 모범 사례

### 보안 강화
  - `monitoring` 전용 네임스페이스를 사용하여 리소스를 격리합니다.
  - 네트워크 정책(NetworkPolicy)을 적용하여 필요한 포트만 열어둡니다.
  - Grafana에는 Ingress 리소스와 TLS(예: cert-manager 사용)를 구성하여 안전하게 외부에 노출합니다.
  - Grafana 인증을 기본 자격 증명에서 OAuth2(OIDC, GitHub, GitLab 등)로 업그레이드하여 중앙 집중식 관리를 구현합니다.

### 성능 및 비용 최적화
  - **스크랩 간격:** 모든 메트릭에 15초 간격이 필요하지는 않습니다. 중요도에 따라 30초 또는 60초로 조정합니다.
  - **라벨 카디널리티:** 고유한 값이 너무 많은 라벨(예: 전체 요청 ID, 사용자 ID)은 Prometheus 성능을 급격히 저하시킵니다. `metric_relabel_configs`를 사용하여 불필요한 라벨을 제거합니다.
  - **보존 정책:** `retention`과 `retentionSize`를 비용과 요구사항에 맞게 조정합니다. 매우 장기적인 데이터는 Thanos나 Cortex와 같은 솔루션으로 오프클러스터 오브젝트 저장소에 저장하는 것을 고려합니다.

### 확장: 멀티클러스터 및 장기 보관
단일 클러스터를 넘어 여러 클러스터의 메트릭을 중앙에서 집계하고 수년간의 데이터를 유지해야 할 필요가 생길 수 있습니다. **Thanos** 또는 **Cortex** 프로젝트는 이러한 요구사항을 해결합니다. 기본적으로 각 클러스터의 Prometheus에 사이드카를 추가하고, 메트릭을 오브젝트 저장소(S3, GCS)에 지속적으로 업로드하게 합니다. 중앙 쿼리어(Querier) 컴포넌트는 모든 클러스터와 오브젝트 저장소의 데이터를 통합하여 하나의 지점에서 쿼리할 수 있게 해줍니다.

---

## 결론

Grafana와 Prometheus를 기반으로 한 Kubernetes 모니터링 스택은 클러스터와 그 안에서 실행되는 애플리케이션의 건강 상태에 대한 완벽한 가시성을 제공하는 강력한 기반을 구축합니다. Helm을 통해 `kube-prometheus-stack`을 배포하는 것은 빠르게 시작할 수 있는 길을 열어주며, ServiceMonitor와 PrometheusRule과 같은 CRD를 사용하면 모니터링 구성을 선언적이고 GitOps 친화적인 방식으로 관리할 수 있습니다.

핵심은 단순히 도구를 설치하는 데 그치지 않고, 비즈니스와 운영 요구사항에 맞게 조정하는 데 있습니다. 올바른 수집 간격 설정, Recording Rule을 통한 성능 최적화, 의미 있는 경보 정의, 팀의 워크플로우에 통합된 대시보드 생성이 모두 포함됩니다. 보안과 장기적인 유지 관리성을 고려한 설계는 이 시스템이 성장하는 인프라의 신뢰할 수 있는 기둥이 되도록 보장합니다.

이 가이드가 제공하는 패턴과 예제는 견고한 모니터링 관행을 수립하고, 잠재적인 문제를 사전에 식별하며, 궁극적으로 더 안정적이고 이해하기 쉬운 Kubernetes 환경을 조성하는 데 도움이 될 것입니다.