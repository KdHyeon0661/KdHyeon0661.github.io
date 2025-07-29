---
layout: post
title: Kubernetes - Grafana + Prometheus
date: 2025-05-05 21:20:23 +0900
category: Kubernetes
---
# Grafana + Prometheus로 Kubernetes 모니터링 구축하기

Kubernetes 클러스터의 상태, 리소스 사용량, 애플리케이션 지표 등을  
시각적으로 모니터링하려면 **Prometheus + Grafana 조합**이 매우 유용합니다.

- **Prometheus**: 메트릭 수집, 저장, 쿼리 기능 제공  
- **Grafana**: Prometheus 데이터를 시각화하는 대시보드 툴

---

## ✅ 1. 구성도 개요

```
[Kubernetes Cluster]
       │
       └─> kubelet, cAdvisor 등에서 메트릭 수집
       ↓
[Prometheus] ← scrape
       ↓
[Grafana] ← 데이터 시각화
```

---

## ✅ 2. 설치 방법 선택

### 방법 1: Helm 사용 (권장)

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

---

## ✅ 3. Prometheus 설치

```bash
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace
```

설치되는 구성:

- Prometheus
- Alertmanager
- kube-state-metrics
- node-exporter
- Grafana (기본 포함됨)

설치 확인:

```bash
kubectl get pods -n monitoring
```

---

## ✅ 4. Grafana 접속

Grafana는 기본적으로 `ClusterIP`로 설치됩니다. 포트 포워딩으로 접근 가능합니다:

```bash
kubectl port-forward svc/prometheus-grafana -n monitoring 3000:80
```

→ 브라우저에서 `http://localhost:3000` 접속

### 기본 로그인 정보

- ID: `admin`
- PW: `prom-operator` (혹은 다음 명령어로 확인 가능)

```bash
kubectl get secret prometheus-grafana -n monitoring -o jsonpath="{.data.admin-password}" | base64 -d
```

---

## ✅ 5. Grafana에서 대시보드 구성

### 📌 기본 내장 대시보드

- Kubernetes Nodes
- Kubernetes Pods
- Cluster Resource Usage
- Prometheus Stats

→ 이미 kube-prometheus-stack에 포함되어 자동 구성됨

---

## ✅ 6. Prometheus 확인

Prometheus UI 접속:

```bash
kubectl port-forward svc/prometheus-kube-prometheus-prometheus -n monitoring 9090:9090
```

→ http://localhost:9090

여기서 직접 PromQL 쿼리 테스트 가능

### 예시 쿼리:

```promql
node_cpu_seconds_total
container_memory_usage_bytes{container!="",container!="POD"}
rate(container_cpu_usage_seconds_total[5m])
```

---

## ✅ 7. Custom Metrics 추가 예시

앱에서 Prometheus 포맷으로 지표 노출 (예: `/metrics` 엔드포인트)

```python
# Flask 예시
from prometheus_client import Counter, generate_latest
from flask import Response

c = Counter('my_requests_total', 'HTTP 요청 수')

@app.route("/metrics")
def metrics():
    return Response(generate_latest(), mimetype="text/plain")
```

→ Prometheus 설정에 scrape config 추가 후 Grafana에서 시각화 가능

---

## ✅ 8. Node Exporter 메트릭 확인

```promql
node_memory_MemAvailable_bytes
node_filesystem_avail_bytes
rate(node_cpu_seconds_total{mode="idle"}[5m])
```

→ CPU, 메모리, 디스크 사용량 등을 확인할 수 있음

---

## ✅ 9. 자주 사용하는 Prometheus 쿼리 (PromQL)

| 메트릭 설명 | PromQL |
|-------------|--------|
| CPU 사용률 | `rate(container_cpu_usage_seconds_total[5m])` |
| 메모리 사용량 | `container_memory_usage_bytes` |
| Pod 재시작 수 | `kube_pod_container_status_restarts_total` |
| Node 사용률 | `100 - (avg by (instance)(irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)` |

---

## ✅ 10. 경고(Alerting) 설정 (옵션)

`kube-prometheus-stack`은 기본 AlertManager를 포함합니다.

- `rules.yaml` 파일을 수정하거나
- Grafana에서 알림 조건 설정 가능

예: CPU가 90% 초과하면 Slack 알림 전송

---

## ✅ 11. 필요 시 삭제

```bash
helm uninstall prometheus -n monitoring
kubectl delete ns monitoring
```

---

## ✅ 결론

Prometheus + Grafana 조합은 Kubernetes 모니터링의 **표준 스택**입니다.

| 구성 요소 | 역할 |
|-----------|------|
| Prometheus | 메트릭 수집 및 저장 |
| AlertManager | 알림 전송 |
| Grafana | 대시보드 시각화 |
| node-exporter | Node 리소스 수집 |
| kube-state-metrics | Kubernetes 상태 정보 수집 |

> 클러스터 상태를 눈으로 확인하고, 병목 현상을 빠르게 찾고,  
> 실시간 지표를 기반으로 문제를 예방할 수 있는 인프라 운영의 핵심 도구입니다.

---

## ✅ 참고

- [kube-prometheus-stack Helm Chart 공식 문서](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)
- [Grafana 공식 대시보드 마켓플레이스](https://grafana.com/grafana/dashboards)
- [PromQL 문법 문서](https://prometheus.io/docs/prometheus/latest/querying/basics/)