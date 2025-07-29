---
layout: post
title: Kubernetes - Loki로 로그 수집하기
date: 2025-05-05 22:20:23 +0900
category: Kubernetes
---
# Loki로 로그 수집하기 (Kubernetes + Grafana 연동)

**Grafana Loki**는 Prometheus처럼 작동하는 **로그 수집 시스템**입니다.  
차이점은 Prometheus가 메트릭을 수집하는 반면, Loki는 **로그(Log)를 수집**합니다.

- Loki는 **라벨 기반 로그 집계**를 제공
- Fluent Bit, Promtail 등과 함께 사용
- Grafana와의 통합이 매우 뛰어남

---

## ✅ Loki 구성 요소

| 구성 요소 | 역할 |
|-----------|------|
| **Loki** | 로그를 저장하는 백엔드 |
| **Promtail** | Pod 로그를 수집하고 Loki에 전송 |
| **Grafana** | Loki 로그를 시각화 |

---

## ✅ 1. Loki + Promtail 설치 (Helm 사용)

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

```bash
helm install loki grafana/loki-stack \
  --namespace loki-stack --create-namespace \
  --set grafana.enabled=true
```

### 설치된 리소스 확인

```bash
kubectl get pods -n loki-stack
```

- `loki-0`: 로그 저장소
- `promtail-*`: 로그 수집기 (각 노드)
- `grafana-*`: Grafana UI

---

## ✅ 2. Grafana 접속 및 로그인

```bash
kubectl port-forward svc/loki-grafana -n loki-stack 3000:80
```

- 접속: [http://localhost:3000](http://localhost:3000)
- 기본 계정: `admin / prom-operator` (또는 Secret에서 조회)

```bash
kubectl get secret --namespace loki-stack loki-grafana \
  -o jsonpath="{.data.admin-password}" | base64 -d
```

---

## ✅ 3. Grafana에서 Loki 데이터소스 연결 확인

Grafana → ⚙️ 설정 → Data Sources → `Loki` 자동 생성됨  
없다면 수동으로 추가:

- Name: `Loki`
- Type: `Loki`
- URL: `http://loki.loki-stack.svc.cluster.local:3100`

"Save & Test" 클릭 → 정상 연결 확인

---

## ✅ 4. 로그 쿼리 기본 예제 (LogQL)

Grafana 대시보드 → **Explore 탭** → `Loki` 선택 후 로그 조회

### 기본 쿼리

```logql
{job="kubernetes-pods"}
```

### 특정 앱 로그만 조회

```logql
{app="nginx"}
```

### 로그 필터링

```logql
{app="nginx"} |= "GET"
{namespace="default"} |~ "error|fail"
```

### 시간 범위 설정

- 우측 상단 시간 선택기에서 `Last 1 hour`, `Last 5m` 등 선택

---

## ✅ 5. Promtail 설정 구조

Promtail은 각 노드에서 `/var/log/containers/*.log` 파일을 tail하여  
Pod 메타데이터(Label, Namespace 등)와 함께 Loki로 전송합니다.

기본 설정 (`values.yaml`) 예:

```yaml
promtail:
  enabled: true
  config:
    clients:
      - url: http://loki:3100/loki/api/v1/push
    positions:
      filename: /tmp/positions.yaml
    scrape_configs:
      - job_name: kubernetes-pods
        kubernetes_sd_configs:
          - role: pod
        relabel_configs:
          - source_labels: [__meta_kubernetes_pod_label_app]
            target_label: app
```

→ Promtail은 자동으로 Pod 메타데이터를 가져와 라벨로 변환해줍니다.

---

## ✅ 6. 실전 예제: Nginx 로그 보기

Nginx 배포 후 로그 확인:

```bash
kubectl create deployment nginx --image=nginx
```

Explore 탭에서 다음 쿼리 입력:

```logql
{app="nginx"}
```

→ 로그가 실시간으로 출력됨

---

## ✅ 7. 로그 대시보드 만들기

Grafana → Dashboards → + New → Add Panel

- Query: `{app="nginx"} |= "GET"`
- Panel Type: Logs
- 제목: "Nginx Access Logs"

---

## ✅ 8. Alerting 연동 (Loki + Alertmanager)

Loki는 기본적으로 메트릭 기반 Alert는 지원하지 않지만,  
LogQL 쿼리를 이용한 log volume 또는 error 탐지를 기준으로 알림 설정 가능

예: 최근 5분 간 `error` 로그 수 ≥ 10

```logql
count_over_time({app="myapp"} |= "error" [5m]) > 10
```

Grafana → Alert → Contact Points → Slack / Email 설정

---

## ✅ 9. 로그 저장 경량화 및 보관 전략

- Loki는 Elasticsearch보다 경량 구조 (인덱스 없음)
- BoltDB 또는 object storage (S3, GCS 등) 지원
- 로그 압축 및 보존 기간 설정 가능 (Helm values에서 설정)

---

## ✅ 10. 삭제 방법

```bash
helm uninstall loki -n loki-stack
kubectl delete namespace loki-stack
```

---

## ✅ 결론

| 구성 | 설명 |
|------|------|
| Loki | 경량 로그 저장소, Prometheus와 유사한 구조 |
| Promtail | 로그 수집기 (Pod에서 로그 수집) |
| Grafana | Loki 로그를 실시간 분석 및 시각화 |

**Loki의 장점**

- Prometheus 방식처럼 라벨 기반 필터링
- Elasticsearch 대비 경량, 유지비용 낮음
- Grafana와 긴밀한 통합
- Helm으로 빠르게 설치 가능