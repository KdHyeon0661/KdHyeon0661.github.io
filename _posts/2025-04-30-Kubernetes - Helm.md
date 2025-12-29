---
layout: post
title: Kubernetes - Helm
date: 2025-04-30 19:20:23 +0900
category: Kubernetes
---
# Helm이란? 왜 사용하는가?

## 개념 요약

Helm은 Kubernetes 애플리케이션 관리를 위한 사실상의 표준 패키지 매니저입니다. Kubernetes 리소스 정의를 "차트(Chart)"라는 단위로 패키징하고, 이를 다양한 환경에 일관되게 배포하고 관리할 수 있도록 돕습니다.

핵심 개념:
- **차트(Chart)**: 애플리케이션을 구성하는 모든 Kubernetes 리소스의 템플릿, 기본 설정값, 메타데이터를 포함하는 패키지입니다.
- **릴리스(Release)**: 클러스터에 설치된 차트의 인스턴스입니다. 동일한 차트로 이름만 다르게 여러 독립적인 배포를 생성할 수 있습니다.
- **값(Values)**: 차트 템플릿에 주입되는 환경별 구성값입니다. 파일, 명령줄 인자, CI/CD 변수를 통해 재정의할 수 있습니다.
- **저장소(Repository/OCI)**: 차트를 배포하고 공유하는 저장소입니다. 전통적인 HTTP Helm 저장소 또는 현대적인 OCI 레지스트리를 사용할 수 있습니다.

---

## Helm의 핵심 가치와 해결과제

### 환경별 구성 관리의 단순화
기존에는 개발(dev), 스테이징(staging), 프로덕션(prod) 환경별로 YAML 파일을 별도로 복제하고 관리해야 했습니다. Helm을 사용하면 단일 템플릿 세트를 유지하면서 `values-dev.yaml`, `values-prod.yaml`과 같은 값 파일만 분리하여 관리할 수 있습니다. 이로 인해 코드 재사용성이 극대화되고 관리 부담이 줄어듭니다.

### 배포 생명주기 관리의 강화
`kubectl apply`로 직접 배포할 경우, 변경 내역 추적과 특정 버전으로의 롤백이 복잡하고 오류가 발생하기 쉽습니다. Helm은 `helm history`로 모든 배포 변경 이력을 명확히 추적하고, `helm rollback`을 통해 원클릭으로 이전 정상 상태로 안전하게 되돌릴 수 있습니다.

### 안정적이고 원자적인 설치
Kubernetes 리소스 설치 중간에 실패하면 일부 리소스만 생성된 불완전한 상태가 될 수 있습니다. Helm의 `--atomic` 플래그를 사용하면 설치나 업그레이드가 실패할 경우 전체 변경 사항을 자동으로 롤백하여 클러스터 상태의 일관성을 보장합니다. `--wait` 플래그와 결합하면 모든 리소스가 정상적으로 준비될 때까지 기다린 후 배포를 완료합니다.

### 배포 품질 보증의 자동화
수동 체크리스트와 사람의 주의에 의존하던 배포 전 검증 과정을 Helm은 `helm lint`(문법 및 규칙 검사)와 `helm test`(설치 후 기능 테스트 실행) 명령으로 자동화합니다. 이를 CI/CD 파이프라인에 통합하여 배포 품질 게이트를 구축할 수 있습니다.

### 패키징과 재사용성
내부 팀 간 또는 공개적으로 애플리케이션을 공유할 때, Helm 차트는 모든 의존성과 설정 옵션을 포함한 표준화된 패키지 형식을 제공합니다. ArtifactHub나 사내 Helm 저장소, OCI 레지스트리를 통해 차트를 쉽게 검색, 배포, 버전 관리할 수 있습니다.

---

## 빠른 시작: 핵심 명령어

```bash
# 새 차트의 기본 구조 생성
helm create mychart

# 템플릿이 실제로 생성할 YAML을 미리 확인
helm template myrel ./mychart -f values-prod.yaml

# 차트의 문법과 모범 사례 검사
helm lint ./mychart

# 설치 또는 업그레이드 (가장 일반적인 명령)
helm upgrade --install myrel ./mychart -n app --create-namespace \
  -f values.yaml -f values-prod.yaml --atomic --wait --timeout 5m

# 배포 상태 및 이력 확인
helm list -n app
helm history myrel -n app

# 이전 릴리스(예: 2번째 배포)로 롤백
helm rollback myrel 2 -n app

# 릴리스 제거
helm uninstall myrel -n app
```

---

## 차트 기본 구조

```
mychart/
├── Chart.yaml           # 차트 이름, 버전, 설명, 의존성 등 메타데이터
├── values.yaml          # 템플릿에 사용되는 기본 구성값
├── values.schema.json   # 값 파일의 유효성을 검증하는 JSON 스키마 (권장)
├── templates/           # Kubernetes 매니페스트 템플릿
│   ├── _helpers.tpl     # 재사용 가능한 템플릿 함수 정의
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── NOTES.txt        # 설치 후 사용자에게 표시할 안내문
│   └── tests/           # helm test로 실행되는 테스트 리소스
├── charts/              # 다운로드된 서브차트(의존성)가 위치하는 디렉토리
├── crds/                # 사용자 정의 리소스 정의(CRD) 파일
└── .helmignore          # 패키징 시 제외할 파일 패턴
```

---

## 실전 예제: 웹 애플리케이션 차트

### 구성값 정의 (values.yaml)
```yaml
replicaCount: 2
image:
  repository: ghcr.io/acme/webapp
  tag: "1.0.0"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 8080

ingress:
  enabled: false
  className: nginx
  hosts: []  # 예: [{ host: "app.example.com", paths: [{ path: "/", pathType: Prefix }] }]

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 200m
    memory: 256Mi

env:
  - name: LOG_LEVEL
    value: "info"
```

### 템플릿 헬퍼 (_helpers.tpl)
{% raw %}
```
{{- define "mychart.fullname" -}}
{{- printf "%s-%s" .Release.Name .Chart.Name | trunc 63 | trimSuffix "-" -}}
{{- end -}}

{{- define "mychart.labels" -}}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
helm.sh/chart: {{ printf "%s-%s" .Chart.Name .Chart.Version | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end -}}
```
{% endraw %}

### 디플로이먼트 템플릿 (deployment.yaml)
{% raw %}
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "mychart.fullname" . }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app.kubernetes.io/name: {{ .Chart.Name }}
      app.kubernetes.io/instance: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app.kubernetes.io/name: {{ .Chart.Name }}
        app.kubernetes.io/instance: {{ .Release.Name }}
    spec:
      containers:
        - name: app
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
            - containerPort: 8080
          env:
            {{- toYaml .Values.env | nindent 12 }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```
{% endraw %}

---

## 환경별 구성 관리 패턴

Helm의 강력한 기능 중 하나는 여러 소스의 값을 계층적으로 병합하여 최종 구성을 만드는 것입니다.

- **파일 레이어링**: `helm install -f values.yaml -f values-prod.yaml -f values-region-us.yaml`
- **명령줄 오버라이드**: `--set replicaCount=3 --set image.tag=latest` (주의: 지나치게 사용하면 구성이 숨겨질 수 있음)
- **값 스키마 검증**: `values.schema.json` 파일을 정의하여 필수 필드, 데이터 타입, 허용 범위를 검증함으로써 오타나 잘못된 설정으로 인한 배포 실패를 방지합니다.

### 값 스키마 예시 (values.schema.json)
```json
{
  "$schema": "https://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "replicaCount": {
      "type": "integer",
      "minimum": 1,
      "description": "실행할 Pod의 복제본 수"
    },
    "image": {
      "type": "object",
      "properties": {
        "repository": { "type": "string" },
        "tag": { "type": "string" }
      },
      "required": ["repository"]
    }
  },
  "required": ["replicaCount", "image"]
}
```

---

## 고급 활용: 훅(Hooks)과 테스트

### 훅을 이용한 데이터베이스 마이그레이션
{% raw %}
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: "{{ .Release.Name }}-db-migrate"
  annotations:
    "helm.sh/hook": pre-upgrade,pre-install
    "helm.sh/hook-weight": "5"
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  template:
    spec:
      containers:
        - name: migrator
          image: "{{ .Values.image.repository }}-migrate:{{ .Values.image.tag }}"
```
{% endraw %}
이 Job은 애플리케이션 업그레이드 *전에* 실행되며(`pre-upgrade`), 성공적으로 완료된 후에는 삭제됩니다(`hook-succeeded`).

### 애플리케이션 건강 상태 테스트
{% raw %}
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: "{{ .Release.Name }}-test-connection"
  annotations:
    "helm.sh/hook": test
spec:
  containers:
    - name: test
      image: curlimages/curl
      command: ['curl', '--fail', 'http://{{ include "mychart.fullname" . }}:{{ .Values.service.port }}/health']
  restartPolicy: Never
```
{% endraw %}
이 Pod는 `helm test <RELEASE_NAME>` 명령으로 실행되어 배포된 애플리케이션의 정상 동작을 확인합니다.

---

## 차트 의존성 관리

복잡한 애플리케이션은 Redis, PostgreSQL 등의 외부 서비스에 의존할 수 있습니다. Helm은 `Chart.yaml`에 의존성을 선언하고 자동으로 가져올 수 있습니다.

### Chart.yaml
```yaml
dependencies:
  - name: postgresql
    version: "~12.0.0"
    repository: "https://charts.bitnami.com/bitnami"
    condition: postgresql.enabled
  - name: redis
    version: "~16.0.0"
    repository: "https://charts.bitnami.com/bitnami"
    alias: cache  # 별칭을 사용하여 이름 충돌 방지
    condition: cache.enabled
```

의존성을 다운로드하려면 `helm dependency update` 명령을 실행합니다. 상위 차트의 `values.yaml`에서 조건을 제어하고 구성값을 전달할 수 있습니다.

---

## 보안과 비밀 관리

Helm 값 파일에 평문 비밀번호를 저장하는 것은 보안상 위험합니다. 다음과 같은 접근 방식을 권장합니다:

1.  **External Secrets Operator**: AWS Secrets Manager, HashiCorp Vault 등 외부 비밀 저장소와 Kubernetes Secret을 동기화합니다.
2.  **Sealed Secrets**: 공개키로 암호화된 Secret을 Git에 저장하고, 클러스터 내의 컨트롤러만이 복호화할 수 있게 합니다.
3.  **SOPS + helm-secrets 플러그인**: 값 파일 자체를 암호화하여 Git에 저장합니다.

---

## 결론

Helm은 Kubernetes 애플리케이션의 배포 복잡성을 해결하는 강력한 도구입니다. 단순한 템플릿 엔진을 넘어, 패키징, 버전 관리, 의존성 해결, 원자적 배포, 롤백, 테스트까지 포괄하는 완전한 배포 수명주기 관리 플랫폼을 제공합니다.

표준화된 차트 형식은 팀 내 및 커뮤니티 전반의 지식과 애플리케이션 구성을 공유하는 데 기여합니다. 값 파일과 스키마 검증을 통한 환경별 구성 분리는 일관성과 안정성을 보장하며, GitOps 워크플로우와의 원활한 통합은 현대적인 지속적 배포 파이프라인의 핵심 요소가 됩니다.

초기 학습 곡선이 존재할 수 있지만, Helm이 제공하는 자동화, 신뢰성, 유지보수성의 혜택은 중대규모 Kubernetes 환경을 운영하는 모든 조직에 있어 필수적인 투자입니다. Kubernetes 생태계에서 Helm은 단순한 도구가 아닌, 안정적이고 반복 가능한 배포 문화를 구축하는 데 기반이 되는 인프라의 핵심 부분입니다.