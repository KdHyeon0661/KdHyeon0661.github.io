---
layout: post
title: Kubernetes - Loki로 로그 수집하기
date: 2025-05-05 22:20:23 +0900
category: Kubernetes
---
# Loki로 로그 수집하기 (Kubernetes + Grafana 연동)

**Grafana Loki**는 Prometheus의 설계 철학을 로그에 적용한 **라벨 기반 로그 수집 및 조회 시스템**이다.  
Prometheus가 시계열 메트릭을 스크레이프하여 라벨로 집계하듯, Loki는 로그 스트림을 **라벨(메타데이터)**로 분류하고 **인덱스를 최소화**해 비용 효율적이다. Kubernetes와의 결합성, Grafana에서의 쿼리/대시보드/알림 연계가 강력하다.

---

## 1. 아키텍처 개요

```
[Kubernetes Nodes]                     [Control Plane / Observability NS]
  /var/log/containers/*.log  ──>  [Promtail DaemonSet]  ─push─>  [Loki]
       (stdout/stderr)              |  Kubernetes SD                |
                                    └─ relabel/pipeline            |
                                                            [Grafana]
                                                      (Explore / Dashboards / Alerts)
```

- **Promtail**: 각 노드의 컨테이너 로그를 tail → Kubernetes 메타데이터로 **라벨**화 → Loki로 push
- **Loki**: 라벨 인덱스 + 청크 스토리지(파일/객체 저장소)에 로그 저장
- **Grafana**: Loki를 데이터소스로 조회(Explore), 시각화(Dashboards), 알림(Alerts)

---

## 2. 설치 옵션 비교

| 방식 | 장점 | 단점 | 적합도 |
|------|------|------|--------|
| kube-prometheus-stack와 별도 Loki 스택 | Prometheus/Alertmanager와 분리 운영 | 컴포넌트 분산 관리 | 중·대규모 |
| grafana/loki-stack Helm Chart | 빠른 설치, 일체형 | 세밀한 커스터마이즈는 추가 values 필요 | 학습·소규모·PoC |
| Jsonnet/Json 모듈 | 완전한 제어 | 초기 학습 곡선 높음 | 대규모·운영팀 표준화 |

이 글은 **Helm 기반 빠른 구축**을 중심으로, **운영에서 필요한 커스터마이징 values**를 함께 제시한다.

---

## 3. Helm으로 Loki + Promtail + Grafana 설치

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

기본 설치(학습/PoC):

```bash
helm install loki grafana/loki-stack \
  --namespace loki-stack --create-namespace \
  --set grafana.enabled=true
```

설치 확인:

```bash
kubectl get pods -n loki-stack
```

주요 파드:
- `loki-0`: Loki StatefulSet (단일 레플리카)
- `promtail-*`: DaemonSet (각 노드에서 로그 수집)
- `loki-grafana-*`: Grafana

Grafana 접속:

```bash
kubectl port-forward svc/loki-grafana -n loki-stack 3000:80
# 브라우저: http://localhost:3000
# 암호 확인
kubectl get secret --namespace loki-stack loki-grafana \
  -o jsonpath="{.data.admin-password}" | base64 -d; echo
```

Grafana 데이터소스 확인:  
Settings → Data Sources → Loki (자동 생성)  
미생성 시 **URL**: `http://loki.loki-stack.svc.cluster.local:3100`

---

## 4. 운영에 맞춘 Helm 커스터마이징

학습 설치는 디폴트 설정으로 충분하지만, 실제 운영에서는 **보존 기간, 스토리지, 리소스, 라벨 정책**을 조정해야 한다.

### 4.1 values 예시(단일 Loki, PV, 보존 7d)

```yaml
# file: values-loki.yaml
loki:
  enabled: true
  isDefault: true
  persistence:
    enabled: true
    size: 50Gi
    storageClassName: gp3 # 클러스터 환경에 맞게
  config:
    auth_enabled: false  # 내부 클러스터 한정 또는 Ingress로 보호
    server:
      http_listen_port: 3100
    common:
      compactor_address: http://loki:3100
    storage_config:
      boltdb_shipper:
        active_index_directory: /data/loki/index
        cache_location: /data/loki/cache
        shared_store: filesystem
      filesystem:
        directory: /data/loki/chunks
    schema_config:
      configs:
        - from: "2024-01-01"
          store: boltdb-shipper
          object_store: filesystem
          schema: v13
          index:
            prefix: index_
            period: 24h
    table_manager:
      retention_deletes_enabled: true
      retention_period: 168h # 7d
  resources:
    requests:
      cpu: "500m"
      memory: "1Gi"
    limits:
      cpu: "2"
      memory: "4Gi"

promtail:
  enabled: true
  resources:
    requests:
      cpu: "100m"
      memory: "200Mi"
    limits:
      cpu: "1"
      memory: "500Mi"
  config:
    clients:
      - url: http://loki:3100/loki/api/v1/push
    positions:
      filename: /run/promtail/positions.yaml
    scrape_configs:
      - job_name: kubernetes-pods
        pipeline_stages: []  # 후술: JSON/멀티라인/레벨 매핑 예제 추가
        kubernetes_sd_configs:
          - role: pod
        relabel_configs:
          - source_labels: [__meta_kubernetes_pod_label_app]
            target_label: app
          - source_labels: [__meta_kubernetes_namespace]
            target_label: namespace
          - source_labels: [__meta_kubernetes_pod_name]
            target_label: pod
          - source_labels: [__meta_kubernetes_container_name]
            target_label: container
          # 불필요한 noisy 라벨 제거 예시
          - action: labeldrop
            regex: "controller-revision-hash|pod-template-hash"
```

적용:

```bash
helm upgrade --install loki grafana/loki-stack \
  -n loki-stack -f values-loki.yaml
```

### 4.2 오브젝트 스토리지(S3) 사용(중·대규모 권장)

```yaml
loki:
  config:
    storage_config:
      boltdb_shipper:
        active_index_directory: /data/loki/index
        cache_location: /data/loki/cache
        shared_store: s3
      aws:
        s3: s3://<ACCESS_KEY>:<SECRET_KEY>@s3.<region>.amazonaws.com/<bucket-name>
        s3forcepathstyle: true
    schema_config:
      configs:
        - from: "2024-01-01"
          store: boltdb-shipper
          object_store: s3
          schema: v13
          index:
            prefix: index_
            period: 24h
```

주의: 자격 증명은 Secret로 분리하고 values에서는 참조하도록 구성한다.

---

## 5. Promtail 파이프라인 고급 설정

Kubernetes 로그는 다음이 섞여 있다.
- 단순 텍스트
- JSON 구조화 로그
- 스택트레이스(멀티라인)
- 타임스탬프/레벨 필드 다양한 포맷

Promtail의 `pipeline_stages`로 **정규화**하자.

### 5.1 JSON 로그 파싱 + 레벨 매핑

```yaml
pipeline_stages:
  - json:
      expressions:
        level: level
        msg: message
        ts: timestamp
        user: userId
  - timestamp:
      source: ts
      format: RFC3339
  - labels:
      level:
      user:
  - output:
      source: msg
  - replace:
      expression: "(?i)password=[^ ]+"
      replace: "password=****"
```

- JSON에서 `level`, `message`, `timestamp`를 추출.
- `labels` 단계로 일부 필드를 **라벨**로 승격(필터링/집계 가능).
- `output`로 메시지 본문을 치환.
- 민감 정보 마스킹 예시 포함.

### 5.2 멀티라인(Go/Java 스택트레이스) 묶기

```yaml
pipeline_stages:
  - multiline:
      firstline: '^\d{4}-\d{2}-\d{2}T'  # ISO8601로 시작하면 새로운 라인
      max_wait_time: 3s
```

또는 Java 로그 패턴:

```yaml
  - multiline:
      firstline: '^\d{2}:\d{2}:\d{2}\.\d{3} '  # 12:34:56.789 형태
      max_wait_time: 3s
```

### 5.3 정규 로그 레벨 추출(텍스트)

```yaml
pipeline_stages:
  - regex:
      expression: '(?P<level>INFO|WARN|ERROR|DEBUG)\s+(?P<msg>.*)'
  - labels:
      level:
  - output:
      source: msg
```

### 5.4 Kubernetes 메타데이터 라벨링 정돈

불필요한 변동 라벨을 제거하면 **카디널리티**와 비용이 준다.

```yaml
relabel_configs:
  - source_labels: [__meta_kubernetes_namespace]
    target_label: namespace
  - source_labels: [__meta_kubernetes_pod_label_app_kubernetes_io_name]
    target_label: app
  - action: labeldrop
    regex: "pod-template-hash|controller-revision-hash|annotation_*"
```

---

## 6. LogQL 핵심과 실전 쿼리

LogQL은 **라벨 선택자 + 파이프라인 연산자**로 구성된다.

### 6.1 기본

```logql
{namespace="default", app="nginx"}
{job="kubernetes-pods"} |= "GET"          # 문자열 포함
{app="api"} |~ "error|fail|exception"     # 정규식 포함
```

### 6.2 파싱 후 필드 기반 필터

Promtail에서 `labels`로 승격된 필드는 라벨로, `output`은 본문으로 이용 가능.

```logql
{app="checkout", level="ERROR"} |= "payment"
```

### 6.3 집계·비율

로그 수 집계:

```logql
count_over_time({app="api"} |= "ERROR" [5m])
```

에러 비율(간이):

```logql
sum(rate(({app="api"} |= "ERROR")[5m])) 
/
sum(rate(({app="api"}))[5m])
```

메서드별 요청 수(파싱이 라벨로 되어 있을 때):

```logql
sum by (method) (count_over_time({app="web"}[1m]))
```

### 6.4 시간 함수와 범위

```logql
{app="nginx"} |= "200" | unwrap duration_ms | avg_over_time( [5m] )
```

`unwrap`은 수치 필드를 본문에서 추출해야 사용할 수 있다(파이프라인에서 구문 추출 필요).

---

## 7. Grafana 대시보드 구성

### 7.1 Explore 탭

- 쿼리: `{namespace="default", app="nginx"}`
- 라이브 모드(Streaming)로 실시간 모니터링

### 7.2 Logs Panel

패널 생성 → Query 입력 → Format: Logs  
예:
- `{app="nginx"} |= "GET"`
- `{app="api", level="ERROR"}`

테이블 변환, 필드 강조(Highlighter), 라벨 표시 여부를 조정한다.

### 7.3 로그 기반 지표화(Log-to-metrics) 패턴

- `count_over_time`/`rate`로 수치화
- Graph 패널에 Time series로 표시

예: 5xx 응답 수

```logql
sum by (pod) (rate({app="nginx"} |~ " 5\\d\\d " [5m]))
```

---

## 8. Alerting 패턴

Grafana Alert로 **LogQL 표현식의 임계치**를 감시한다.

예: 최근 5분 에러 ≥ 10

```logql
count_over_time({app="myapp"} |= "error" [5m])
```

알림 조건: `IS ABOVE 10`  
Contact points: Slack/Email/Webhook

주의:
- 지나치게 빈도 높은 경고는 알림 피로도를 유발하므로 **집계·지연**을 적절히 조정한다.
- 노이즈 라벨을 줄여 **조건 일관성**을 확보한다.

---

## 9. 보존·성능·비용 전략

### 9.1 보존
- 개발/스테이징: 3~7일
- 프로덕션: 7~30일(규제/내부 정책 기준)
- 장기 보관은 오브젝트 스토리지와 **압축 청크**를 활용

### 9.2 카디널리티 관리
- 라벨 수와 가능한 값의 종류를 최소화
- Pod UID/Revision 해시 등 **고카디널리티 라벨 제거**
- `labeldrop`, 라벨 표준화 가이드 도입

### 9.3 수집량/쿼리 최적화
- Promtail에서 **멀티라인 묶기**로 라인 폭발 방지
- 필터링/마스킹을 수집단에서 수행해 전송량 절감
- Grafana에서 시간 범위·대상 라벨을 **정확히** 제한

---

## 10. 보안·격리

### 10.1 네트워크
- Loki API(3100/TCP) 접근을 **클러스터 내부로 제한**
- 외부 노출 시 Ingress + 인증(최소 BasicAuth, 더 나은 방법은 OIDC)

### 10.2 RBAC
- Promtail: Pod/Namespace 메타데이터를 조회할 수 있는 권한 필요
- Grafana: 오가니제이션/팀 단위의 데이터소스 권한 분리

### 10.3 테넌트
- 멀티테넌시 필요 시 `X-Scope-OrgID` 헤더 기반 테넌트 분리 지원
- Helm values에서 모드 활성화 및 프록시/Ingress에서 헤더 주입 전략 설계

---

## 11. 실전 예제: Nginx 배포 → 로그 조회

Nginx 배포:

```bash
kubectl create deployment nginx --image=nginx
```

Explore → 쿼리:

```logql
{app="nginx"}
```

Nginx Access 로그 GET만:

```logql
{app="nginx"} |= "GET"
```

5xx 비율 모니터:

```logql
sum(rate({app="nginx"} |~ " 5\\d\\d " [5m]))
/
sum(rate({app="nginx"} |~ " \\d\\d\\d " [5m]))
```

---

## 12. Promtail 고급 파이프라인 통합 예시

여러 유형의 로그를 동시에 다루는 **권장 파이프라인**:

```yaml
promtail:
  config:
    scrape_configs:
      - job_name: kubernetes-pods
        kubernetes_sd_configs:
          - role: pod
        pipeline_stages:
          # 1) 멀티라인(자주 쓰는 두 가지 패턴 예시 중 택1)
          - multiline:
              firstline: '^\d{4}-\d{2}-\d{2}T'     # ISO8601
              max_wait_time: 3s
          # - multiline:
          #     firstline: '^\d{2}:\d{2}:\d{2}\.\d{3} ' # Java 시각
          # 2) JSON 파싱(가능하면 구조화)
          - json:
              expressions:
                ts: time
                level: level
                msg: message
                method: method
                status: status
                path: path
          # 3) 타임스탬프/라벨 승격
          - timestamp:
              source: ts
              format: RFC3339
          - labels:
              level:
              method:
              status:
          # 4) 본문 치환 + 민감정보 마스킹
          - output:
              source: msg
          - replace:
              expression: "(?i)authorization: Bearer [A-Za-z0-9\\._-]+"
              replace: "authorization: Bearer ****"
        relabel_configs:
          - source_labels: [__meta_kubernetes_namespace]
            target_label: namespace
          - source_labels: [__meta_kubernetes_pod_label_app]
            target_label: app
          - source_labels: [__meta_kubernetes_pod_name]
            target_label: pod
          - source_labels: [__meta_kubernetes_container_name]
            target_label: container
          - action: labeldrop
            regex: "controller-revision-hash|pod-template-hash|annotation_*"
```

이 파이프라인을 적용하면 Grafana에서 다음과 같이 쿼리할 수 있다:

```logql
{app="web", level="ERROR"}                # 레벨 라벨 필터
{app="web"} | unwrap status | avg_over_time([5m])   # 수치화된 status 활용
```

---

## 13. 문제 해결(트러블슈팅)

| 증상 | 원인 | 점검 | 해결 |
|------|------|------|------|
| Grafana에서 데이터소스 오류 | URL/Service/NS 오타, 네트워크 차단 | `kubectl get svc -n loki-stack` | URL 정확화, Port-forward로 재확인 |
| 로그가 비어 있음 | Promtail 권한/경로 문제 | Promtail 파드 로그, `/var/log/containers` 접근 | DaemonSet 권한/마운트 점검, RBAC 보완 |
| 멀티라인 분리 | firstline 패턴 미스 | 원 로그 샘플 확인 | 패턴 보정, `max_wait_time` 조정 |
| 라벨 폭발/비용 증가 | 고카디널리티 라벨 | `labeldrop`, 표준화 가이드 | 라벨 축소, 라벨 정책 코드리뷰 |
| 느린 쿼리 | 넓은 시간범위/라벨 미제한 | 시간/라벨 선필터 | 대역폭 줄이고 집계는 점진적으로 |
| 보존 안됨 | retention 설정 누락 | Loki config 확인 | `table_manager.retention_*`/Ruler 정책 설정 |

---

## 14. 삭제

```bash
helm uninstall loki -n loki-stack
kubectl delete ns loki-stack
```

---

## 15. 요약

| 구성 | 역할 |
|------|------|
| Loki | 라벨 기반 로그 저장(인덱스 최소화로 비용 효율) |
| Promtail | 로그 수집·라벨링·파이프라인(멀티라인/JSON/마스킹) |
| Grafana | Explore/대시보드/알림(LogQL 기반) |

핵심 포인트:
- **stdout/stderr**로 로그를 내보내고, Promtail로 구조화/정제 후 **라벨 최소화**.
- 운영은 **보존/스토리지/카디널리티/보안** 네 가지 축을 관리.
- LogQL로 **필터→집계→지표화→알림**까지 일관된 흐름을 구축한다.

---

## 부록 A. Grafana 프로비저닝(선택)

코드형 관리가 필요하면 ConfigMap로 데이터소스/대시보드를 프로비저닝한다.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-datasources
  namespace: loki-stack
  labels: { grafana_datasource: "1" }
data:
  loki.yaml: |
    apiVersion: 1
    datasources:
      - name: Loki
        type: loki
        access: proxy
        url: http://loki.loki-stack.svc.cluster.local:3100
        isDefault: true
```

---

## 부록 B. Kubernetes 접근 로그 샘플로 LogQL 연습

가정: Promtail에서 `status`, `method`, `path` 라벨화

```logql
# 5xx 비율
sum(rate({app="nginx"} | unwrap status [5m])) by (status)
/ ignoring(status) sum(rate({app="nginx"} [5m]))

# 엔드포인트 상위 오류
topk(10, count_over_time({app="api", status="500"}[10m]) by (path))

# 특정 사용자 이슈 추적(라벨 user 승격 가정)
{app="api", user="u-123"} |~ "timeout|deadline"
```

라벨화 수준은 수집단에서 결정된다. **필요한 필드만 엄선**해서 라벨로 올리는 것이 핵심이다.
