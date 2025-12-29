---
layout: post
title: Kubernetes - Loki로 로그 수집하기
date: 2025-05-05 22:20:23 +0900
category: Kubernetes
---
# Loki로 로그 수집하기 (Kubernetes + Grafana 연동)

**Grafana Loki**는 Prometheus의 설계 철학을 로그 관리에 적용한 **라벨 기반 로그 수집 및 조회 시스템**입니다. Prometheus가 시계열 메트릭을 스크레이핑하여 라벨로 집계하는 방식과 유사하게, Loki는 로그 스트림을 **라벨(메타데이터)**로 분류하고 **인덱스를 최소화**하여 비용 효율성을 높입니다. Kubernetes와의 긴밀한 통합과 Grafana에서의 쿼리, 대시보드, 알림 연계가 강력한 장점입니다.

---

## 아키텍처 개요

```
[Kubernetes Nodes]                     [Control Plane / Observability NS]
  /var/log/containers/*.log  ──>  [Promtail DaemonSet]  ─push─>  [Loki]
       (stdout/stderr)              |  Kubernetes SD                |
                                    └─ relabel/pipeline            |
                                                            [Grafana]
                                                      (Explore / Dashboards / Alerts)
```

- **Promtail**: 각 노드에서 컨테이너 로그를 수집하여 Kubernetes 메타데이터를 **라벨**로 변환한 후 Loki로 전송합니다.
- **Loki**: 라벨 기반 인덱스와 청크 스토리지(파일 시스템 또는 객체 저장소)에 로그를 저장합니다.
- **Grafana**: Loki를 데이터 소스로 연결하여 로그 조회(Explore), 시각화(Dashboards), 알림(Alerts) 기능을 제공합니다.

---

## 설치 옵션 비교

- **kube-prometheus-stack와 별도 Loki 스택**: Prometheus와 Alertmanager를 별도로 운영할 수 있어 유연성이 높지만, 컴포넌트가 분산되어 관리가 복잡할 수 있습니다. 중·대규모 환경에 적합합니다.
- **grafana/loki-stack Helm Chart**: 빠른 설치가 가능한 일체형 솔루션이지만, 세밀한 커스터마이징에는 추가 구성이 필요합니다. 학습, 소규모 환경, PoC에 적합합니다.
- **Jsonnet/Json 모듈**: 완전한 제어가 가능하지만 초기 학습 곡선이 높습니다. 대규모 환경이나 운영팀의 표준화된 관리를 필요로 하는 경우에 적합합니다.

이 문서에서는 **Helm 기반의 빠른 구축**을 중심으로 설명하며, **운영 환경에서 필요한 커스터마이징 설정**을 함께 제시합니다.

---

## Helm으로 Loki + Promtail + Grafana 설치

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

기본 설치(학습 및 PoC 용도):

```bash
helm install loki grafana/loki-stack \
  --namespace loki-stack --create-namespace \
  --set grafana.enabled=true
```

설치 상태 확인:

```bash
kubectl get pods -n loki-stack
```

주요 파드 구성:
- `loki-0`: Loki StatefulSet (단일 레플리카)
- `promtail-*`: DaemonSet (각 노드에서 로그 수집)
- `loki-grafana-*`: Grafana 웹 인터페이스

Grafana 접속:

```bash
kubectl port-forward svc/loki-grafana -n loki-stack 3000:80
# 브라우저에서 http://localhost:3000 접속
# 관리자 암호 확인
kubectl get secret --namespace loki-stack loki-grafana \
  -o jsonpath="{.data.admin-password}" | base64 -d; echo
```

Grafana 데이터 소스 확인:
Settings → Data Sources → Loki (자동 생성되어 있어야 함)
자동 생성되지 않은 경우 **URL**을 `http://loki.loki-stack.svc.cluster.local:3100`로 설정합니다.

---

## 운영 환경에 맞춘 Helm 커스터마이징

학습용 기본 설치로는 충분하지만, 실제 운영 환경에서는 **보존 기간, 스토리지, 리소스 할당, 라벨 정책** 등을 조정해야 합니다.

### 기본 운영 설정 예시 (단일 Loki, PV 사용, 7일 보존)

```yaml
# 파일: values-loki.yaml

loki:
  enabled: true
  isDefault: true
  persistence:
    enabled: true
    size: 50Gi
    storageClassName: gp3  # 클러스터 환경에 맞게 조정
  config:
    auth_enabled: false    # 내부 클러스터용 또는 Ingress 인증 구성 필요
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
      retention_period: 168h  # 7일
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
        pipeline_stages: []  # 후술할 JSON/멀티라인/레벨 매핑 예제 추가
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
          # 불필요한 노이즈 라벨 제거
          - action: labeldrop
            regex: "controller-revision-hash|pod-template-hash"
```

적용:

```bash
helm upgrade --install loki grafana/loki-stack \
  -n loki-stack -f values-loki.yaml
```

### S3 스토리지 구성 예시 (중·대규모 환경 권장)

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

주의: 자격 증명은 Secret으로 분리하고 values에서는 참조하도록 구성해야 합니다.

---

## Promtail 파이프라인 고급 설정

Kubernetes 환경의 로그는 다양한 형식이 혼재되어 있습니다.
- 단순 텍스트 로그
- JSON 구조화 로그
- 스택 트레이스(멀티라인)
- 다양한 포맷의 타임스탬프와 로그 레벨 필드

Promtail의 `pipeline_stages`를 활용하여 이러한 로그를 **정규화**할 수 있습니다.

### JSON 로그 파싱 및 로그 레벨 매핑

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

- JSON 로그에서 `level`, `message`, `timestamp`, `userId` 필드를 추출합니다.
- `labels` 단계에서 `level`과 `user` 필드를 **라벨**로 승격시켜 필터링과 집계가 가능하도록 합니다.
- `output`으로 메시지 본문을 치환합니다.
- 민감한 정보를 마스킹하는 예시가 포함되어 있습니다.

### 멀티라인 로그 묶기

```yaml
pipeline_stages:
  - multiline:
      firstline: '^\d{4}-\d{2}-\d{2}T'  # ISO8601 형식으로 시작하면 새로운 로그 라인
      max_wait_time: 3s
```

Java 애플리케이션 로그 패턴:

```yaml
  - multiline:
      firstline: '^\d{2}:\d{2}:\d{2}\.\d{3} '  # 12:34:56.789 형태
      max_wait_time: 3s
```

### 텍스트 로그에서 로그 레벨 추출

```yaml
pipeline_stages:
  - regex:
      expression: '(?P<level>INFO|WARN|ERROR|DEBUG)\s+(?P<msg>.*)'
  - labels:
      level:
  - output:
      source: msg
```

### Kubernetes 메타데이터 라벨링 정리

불필요하게 변동성이 높은 라벨을 제거하면 **카디널리티**를 낮추고 비용을 절감할 수 있습니다.

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

## LogQL 핵심 구문과 실전 쿼리

LogQL은 **라벨 선택자 + 파이프라인 연산자**로 구성된 Loki의 쿼리 언어입니다.

### 기본 쿼리

```logql
{namespace="default", app="nginx"}
{job="kubernetes-pods"} |= "GET"          # "GET" 문자열이 포함된 로그
{app="api"} |~ "error|fail|exception"     # 정규식 패턴이 매칭되는 로그
```

### 파싱 후 필드 기반 필터링

Promtail에서 `labels` 단계를 통해 라벨로 승격된 필드는 라벨로, `output` 단계를 거친 메시지는 본문으로 활용할 수 있습니다.

```logql
{app="checkout", level="ERROR"} |= "payment"
```

### 집계와 비율 계산

로그 수 집계:

```logql
count_over_time({app="api"} |= "ERROR" [5m])
```

에러 비율 계산:

```logql
sum(rate(({app="api"} |= "ERROR")[5m]))
/
sum(rate(({app="api"}))[5m])
```

메서드별 요청 수 (메서드가 라벨로 승격된 경우):

```logql
sum by (method) (count_over_time({app="web"}[1m]))
```

### 시간 함수와 범위 활용

```logql
{app="nginx"} |= "200" | unwrap duration_ms | avg_over_time( [5m] )
```

`unwrap`은 수치형 필드를 본문에서 추출해야 사용할 수 있습니다. 이를 위해서는 파이프라인 단계에서 해당 필드를 구문 분석해야 합니다.

---

## Grafana 대시보드 구성

### Explore 탭 활용

- 쿼리: `{namespace="default", app="nginx"}`
- 라이브 모드(Streaming)를 활성화하여 실시간 로그 모니터링

### Logs 패널 구성

패널 생성 → Query 입력 → Format: Logs 선택
예시 쿼리:
- `{app="nginx"} |= "GET"`
- `{app="api", level="ERROR"}`

테이블 변환, 필드 강조(Highlighter), 라벨 표시 여부를 조정할 수 있습니다.

### 시계열 그래프 패턴

- `count_over_time`/`rate` 함수로 수치화
- Graph 패널에 Time series 형식으로 표시

예: 5xx 응답 수 시각화

```logql
sum by (pod) (rate({app="nginx"} |~ " 5\\d\\d " [5m]))
```

---

## 알림 패턴

Grafana 알림 기능을 사용하여 **LogQL 표현식의 임계값**을 감시할 수 있습니다.

예: 최근 5분 동안 에러 로그가 10건 이상 발생한 경우

```logql
count_over_time({app="myapp"} |= "error" [5m])
```

알림 조건: `IS ABOVE 10`
연락처: Slack/Email/Webhook 등

주의사항:
- 너무 빈번한 경고는 알림 피로도를 유발하므로 **집계 주기와 지연 시간**을 적절히 조정해야 합니다.
- 불필요한 노이즈 라벨을 제거하여 **조건의 일관성**을 확보해야 합니다.

---

## 보존, 성능, 비용 관리 전략

### 보존 정책

- 개발/스테이징 환경: 3~7일
- 프로덕션 환경: 7~30일 (규제 요구사항 및 내부 정책에 따라)
- 장기 보관이 필요한 경우 객체 저장소와 **압축 청크**를 활용

### 카디널리티 관리

- 라벨 수와 가능한 값의 종류를 최소화
- Pod UID, Revision 해시 등 **고카디널리티 라벨 제거**
- `labeldrop` 설정과 라벨 표준화 가이드 도입

### 수집량 및 쿼리 최적화

- Promtail에서 **멀티라인 묶기**를 통해 라인 수 폭발 방지
- 불필요한 로그 필터링과 민감 정보 마스킹을 수집 단계에서 수행하여 전송량 절감
- Grafana에서 시간 범위와 대상 라벨을 **정확히 제한**하여 쿼리 성능 향상

---

## 보안 및 격리

### 네트워크 보안

- Loki API(3100/TCP) 접근을 **클러스터 내부로 제한**
- 외부 노출이 필요한 경우 Ingress와 인증(BasicAuth 이상, OIDC 권장) 구성

### RBAC 설정

- Promtail: Pod 및 Namespace 메타데이터 조회 권한 필요
- Grafana: 조직/팀 단위의 데이터 소스 접근 권한 분리

### 멀티테넌시 지원

- 다중 테넌시가 필요한 경우 `X-Scope-OrgID` 헤더 기반 테넌트 분리 지원
- Helm values에서 테넌시 모드 활성화 및 프록시/Ingress에서 헤더 주입 전략 설계

---

## 실전 예제: Nginx 배포 및 로그 조회

Nginx 디플로이먼트 생성:

```bash
kubectl create deployment nginx --image=nginx
```

Grafana Explore에서 쿼리 실행:

```logql
{app="nginx"}
```

Nginx Access 로그 중 GET 요청만 필터링:

```logql
{app="nginx"} |= "GET"
```

5xx 응답 비율 모니터링:

```logql
sum(rate({app="nginx"} |~ " 5\\d\\d " [5m]))
/
sum(rate({app="nginx"} |~ " \\d\\d\\d " [5m]))
```

---

## Promtail 고급 파이프라인 통합 예시

다양한 유형의 로그를 동시에 처리하는 **권장 파이프라인 구성**:

```yaml
promtail:
  config:
    scrape_configs:
      - job_name: kubernetes-pods
        kubernetes_sd_configs:
          - role: pod
        pipeline_stages:
          # 1) 멀티라인 처리 (두 가지 패턴 예시 중 선택)
          - multiline:
              firstline: '^\d{4}-\d{2}-\d{2}T'     # ISO8601 형식
              max_wait_time: 3s
          # - multiline:
          #     firstline: '^\d{2}:\d{2}:\d{2}\.\d{3} ' # Java 시간 형식
          # 2) JSON 파싱 (구조화된 로그 처리)
          - json:
              expressions:
                ts: time
                level: level
                msg: message
                method: method
                status: status
                path: path
          # 3) 타임스탬프 변환 및 라벨 승격
          - timestamp:
              source: ts
              format: RFC3339
          - labels:
              level:
              method:
              status:
          # 4) 본문 치환 및 민감정보 마스킹
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

이 파이프라인을 적용하면 Grafana에서 다음과 같은 쿼리를 실행할 수 있습니다:

```logql
{app="web", level="ERROR"}                # 에러 레벨 로그 필터링
{app="web"} | unwrap status | avg_over_time([5m])   # 상태 코드 평균 계산
```

---

## 문제 해결 가이드

**Grafana에서 데이터 소스 오류 발생**
- **원인**: URL, Service 이름, 네임스페이스 오타 또는 네트워크 차단
- **점검**: `kubectl get svc -n loki-stack` 명령으로 서비스 상태 확인
- **해결**: URL 정확히 입력, Port-forward로 연결 테스트

**로그가 비어 있음**
- **원인**: Promtail 권한 부족 또는 로그 경로 문제
- **점검**: Promtail 파드 로그 확인, `/var/log/containers` 접근 가능 여부
- **해결**: DaemonSet 권한 및 볼륨 마운트 점검, RBAC 설정 보완

**멀티라인 로그 분리 문제**
- **원인**: firstline 패턴이 원본 로그와 일치하지 않음
- **점검**: 원본 로그 샘플 확인
- **해결**: 패턴 수정, `max_wait_time` 값 조정

**라벨 폭발 및 비용 증가**
- **원인**: 고카디널리티 라벨 사용
- **점검**: `labeldrop` 설정 적용 여부
- **해결**: 라벨 축소, 라벨 정책 코드 리뷰 도입

**쿼리 성능 저하**
- **원인**: 과도한 시간 범위 또는 라벨 필터링 미적용
- **점검**: 쿼리 시간 범위 및 라벨 제한 확인
- **해결**: 시간 범위 축소, 집계 쿼리는 점진적으로 적용

**보존 정책 미적용**
- **원인**: retention 설정 누락
- **점검**: Loki 구성 파일 확인
- **해결**: `table_manager.retention_*` 설정 또는 Ruler 정책 구성

---

## 시스템 삭제

```bash
helm uninstall loki -n loki-stack
kubectl delete ns loki-stack
```

---

## 결론

Grafana Loki는 Kubernetes 환경에서 로그 관리를 위한 효율적이고 강력한 솔루션입니다. Prometheus와 유사한 라벨 기반 접근 방식을 채택하여 메트릭과 로그를 일관된 방식으로 관리할 수 있습니다.

핵심 구성 요소:
- **Loki**: 라벨 기반 로그 저장소로 인덱스를 최소화하여 비용 효율성 제공
- **Promtail**: 로그 수집, 라벨링, 파이프라인 처리(멀티라인/JSON/마스킹) 담당
- **Grafana**: Explore, 대시보드, 알림을 통한 로그 시각화 및 모니터링

운영 포인트:
- 애플리케이션은 **stdout/stderr**로 로그를 출력하고, Promtail을 통해 구조화 및 정제 후 **라벨 카디널리티를 최소화**해야 합니다.
- 운영 환경에서는 **보존 정책, 스토리지 구성, 카디널리티 관리, 보안 설정** 네 가지 측면을 철저히 관리해야 합니다.
- LogQL을 활용하여 **필터링 → 집계 → 지표화 → 알림**까지 일관된 로그 분석 흐름을 구축할 수 있습니다.

초기에는 빠른 설치를 위해 Helm 차트를 활용하고, 점차 운영 요구사항에 맞게 커스터마이징을 추가하는 접근 방식을 권장합니다. 특히 파이프라인 설정을 통해 로그를 효과적으로 구조화하면, 이후의 모니터링과 문제 해결이 훨씬 수월해집니다.

---

## 부록 A: Grafana 프로비저닝 (선택 사항)

코드형 관리를 위해 ConfigMap으로 데이터 소스와 대시보드를 프로비저닝할 수 있습니다.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-datasources
  namespace: loki-stack
  labels:
    grafana_datasource: "1"
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

## 부록 B: LogQL 연습 예제

Promtail에서 `status`, `method`, `path` 필드를 라벨로 승격했다고 가정한 쿼리 예시:

```logql
# 상태 코드별 비율 계산

sum(rate({app="nginx"} | unwrap status [5m])) by (status)
/ ignoring(status) sum(rate({app="nginx"} [5m]))

# 오류 발생 상위 10개 엔드포인트

topk(10, count_over_time({app="api", status="500"}[10m]) by (path))

# 특정 사용자 관련 이슈 추적 (user 라벨 승격 가정)

{app="api", user="u-123"} |~ "timeout|deadline"
```

라벨화 수준은 수집 단계에서 결정됩니다. **필요한 필드만 신중하게 선택**하여 라벨로 승격시키는 것이 시스템 성능과 비용 관리에 중요합니다.