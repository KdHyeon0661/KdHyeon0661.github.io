---
layout: post
title: Kubernetes - Helm Chart
date: 2025-04-30 20:20:23 +0900
category: Kubernetes
---
# Helm Chart 구조

## 기본 디렉터리 구조

```bash
helm create mychart
tree mychart
```

```text
mychart/
├── Chart.yaml
├── values.yaml
├── values.schema.json
├── charts/
├── templates/
│   ├── _helpers.tpl
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── hpa.yaml
│   ├── NOTES.txt
│   └── tests/
├── crds/
└── .helmignore
```

### 핵심 파일 설명

- **Chart.yaml**: 차트의 이름, 버전, 의존성 등 메타데이터를 정의합니다.
- **values.yaml**: 템플릿에 주입될 기본 구성값을 정의합니다. 환경별로 오버라이드 가능합니다.
- **templates/**: 실제 쿠버네티스 리소스를 생성하는 Go 템플릿 파일들이 위치합니다.
- **_helpers.tpl**: 차트 전반에서 사용되는 공통 네이밍 규칙과 라벨 함수를 정의합니다.
- **values.schema.json**: 값의 타입과 필수성을 검증하여 설치 전 오류를 방지합니다.
- **crds/**: Custom Resource Definition 매니페스트를 별도로 관리합니다.
- **charts/**: 서브차트(의존성)를 보관합니다.

---

## Chart.yaml

```yaml
apiVersion: v2
name: mychart
description: A sample Helm chart for Kubernetes
type: application
version: 1.2.0
appVersion: "1.25.3"
kubeVersion: ">=1.25.0-0"
dependencies:
  - name: redis
    version: 18.6.2
    repository: https://charts.bitnami.com/bitnami
    alias: cache
    condition: cache.enabled
```

- `type: library`로 설정하면 리소스를 생성하지 않는 템플릿 재사용 전용 차트를 만들 수 있습니다.
- `dependencies`는 `helm dependency update` 명령어로 charts/ 디렉터리에 다운로드됩니다.

---

## values.yaml — 구성값 관리

```yaml
replicaCount: 3

image:
  repository: nginx
  tag: "1.25.3"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: false
  className: nginx
  hosts:
    - host: app.example.com
      paths:
        - path: /
          pathType: Prefix
  tls: []

resources:
  limits:
    cpu: 200m
    memory: 256Mi
  requests:
    cpu: 100m
    memory: 128Mi

env:
  - name: LOG_LEVEL
    value: info

nodeSelector: {}
tolerations: []
affinity: {}

fullnameOverride: ""
nameOverride: ""

global:
  imagePullSecrets: []
```

환경별 구성 파일(예: `values-prod.yaml`)을 사용하여 기본값을 오버라이드할 수 있습니다.
```bash
helm upgrade --install web ./mychart -f values-prod.yaml
```

---

## 템플릿 핵심 문법 (Go 템플릿 + Sprig)

템플릿은 Go 템플릿을 기반으로 하며 Sprig 유틸리티 함수를 사용할 수 있습니다.

- **내장 객체**: `.Values`, `.Release`, `.Chart`, `.Capabilities`, `.Release.Namespace`
- **흐름 제어**: `if`, `with`, `range`
- **함수**: `default`, `required`, `include`, `tpl`, `toYaml`, `nindent`, `b64enc` 등

### Deployment 템플릿 예시

{% raw %}
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "mychart.fullname" . }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount | default 1 }}
  selector:
    matchLabels:
      {{- include "mychart.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "mychart.selectorLabels" . | nindent 8 }}
    spec:
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      containers:
        - name: app
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy | default "IfNotPresent" }}
          ports:
            - containerPort: 80
          env:
            {{- range $i, $e := .Values.env }}
            - name: {{ $e.name }}
              value: {{ $e.value | quote }}
            {{- end }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```
{% endraw %}

### Service 템플릿 (조건부 생성)

{% raw %}
```yaml
{{- if .Values.service }}
apiVersion: v1
kind: Service
metadata:
  name: {{ include "mychart.fullname" . }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
spec:
  type: {{ .Values.service.type | default "ClusterIP" }}
  selector:
    {{- include "mychart.selectorLabels" . | nindent 4 }}
  ports:
    - name: http
      port: {{ .Values.service.port | default 80 }}
      targetPort: 80
{{- end }}
```
{% endraw %}

### Ingress 템플릿 (활성화 시에만 생성)

{% raw %}
```yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "mychart.fullname" . }}
  annotations:
    kubernetes.io/ingress.class: {{ .Values.ingress.className | default "nginx" | quote }}
spec:
  ingressClassName: {{ .Values.ingress.className | default "nginx" }}
  rules:
  {{- range .Values.ingress.hosts }}
    - host: {{ .host | quote }}
      http:
        paths:
        {{- range .paths }}
          - path: {{ .path | quote }}
            pathType: {{ .pathType | default "Prefix" }}
            backend:
              service:
                name: {{ include "mychart.fullname" $ }}
                port:
                  number: {{ $.Values.service.port | default 80 }}
        {{- end }}
  {{- end }}
  {{- if .Values.ingress.tls }}
  tls:
    {{- toYaml .Values.ingress.tls | nindent 4 }}
  {{- end }}
{{- end }}
```
{% endraw %}

---

## _helpers.tpl — 공통 템플릿 함수

{% raw %}
```tpl
{{- define "mychart.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" -}}
{{- end }}

{{- define "mychart.fullname" -}}
{{- if .Values.fullnameOverride -}}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" -}}
{{- else -}}
{{- printf "%s-%s" .Release.Name (include "mychart.name" .) | trunc 63 | trimSuffix "-" -}}
{{- end -}}
{{- end }}

{{- define "mychart.selectorLabels" -}}
app.kubernetes.io/name: {{ include "mychart.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

{{- define "mychart.labels" -}}
{{ include "mychart.selectorLabels" . }}
helm.sh/chart: {{ printf "%s-%s" .Chart.Name .Chart.Version | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}
```
{% endraw %}

`app.kubernetes.io/*` 형식의 표준 라벨을 사용하면 리소스의 추적성과 선택기 사용이 용이해집니다.

---

## values.schema.json — 값 검증

설치 전에 구성값의 타입과 필수성을 검증하여 오류를 사전에 방지합니다.

```json
{
  "$schema": "https://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "replicaCount": { "type": "integer", "minimum": 1 },
    "image": {
      "type": "object",
      "properties": {
        "repository": { "type": "string", "minLength": 1 },
        "tag": { "type": "string" }
      },
      "required": ["repository"]
    },
    "service": {
      "type": "object",
      "properties": {
        "type": { "type": "string", "enum": ["ClusterIP", "NodePort", "LoadBalancer"] },
        "port": { "type": "integer", "minimum": 1, "maximum": 65535 }
      },
      "required": ["type", "port"]
    }
  },
  "required": ["image", "service"]
}
```

`helm install` 또는 `helm upgrade` 실행 시 자동으로 검증이 수행됩니다.

---

## NOTES.txt — 설치 후 안내 메시지

`templates/NOTES.txt`는 사용자에게 애플리케이션 접근 방법이나 다음 단계를 안내합니다.

{% raw %}
```tpl
{{- if .Values.ingress.enabled -}}
1. Access:
   https://{{ (index .Values.ingress.hosts 0).host | quote }}
{{- else -}}
1. Service (ClusterIP): run port-forward
   kubectl port-forward svc/{{ include "mychart.fullname" . }} 8080:{{ .Values.service.port }}
   curl http://localhost:8080
{{- end -}}
```
{% endraw %}

---

## Hooks — 릴리스 수명주기 제어

Job이나 ConfigMap에 훅 애노테이션을 추가하여 설치, 업그레이드 등의 특정 시점에 작업을 실행할 수 있습니다.

{% raw %}
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: "{{ include "mychart.fullname" . }}-db-migrate"
  annotations:
    "helm.sh/hook": pre-upgrade,pre-install
    "helm.sh/hook-weight": "5"
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  template:
    spec:
      restartPolicy: OnFailure
      containers:
        - name: migrate
          image: "{{ .Values.migrate.image }}"
          args: ["./migrate.sh"]
```
{% endraw %}

- `hook-weight`를 사용하여 여러 훅의 실행 순서를 제어할 수 있습니다.
- `hook-delete-policy`로 훅 리소스의 삭제 시점을 관리합니다.

---

## Tests — 설치 검증

`templates/tests/` 디렉터리의 Job 매니페스트는 `helm test RELEASE` 명령어로 실행되어 릴리스의 정상 동작을 검증합니다.

{% raw %}
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: "{{ include "mychart.fullname" . }}-test-connection"
  annotations:
    "helm.sh/hook": test
spec:
  restartPolicy: Never
  containers:
    - name: curl
      image: curlimages/curl
      args: ["-sf", "http://{{ include "mychart.fullname" . }}:{{ .Values.service.port }}/healthz"]
```
{% endraw %}

---

## CRDs — Custom Resource Definition 관리

- `crds/` 디렉터리에 위치한 CRD 파일은 일반 템플릿과 별도로 선행 설치됩니다.
- CRD의 변경은 호환성 문제를 일으킬 수 있으므로 주의가 필요하며, 운영 환경에서는 차트와 별도로 관리하거나 분리된 차트로 배포하는 것이 일반적입니다.

---

## 서브차트와 의존성 관리

### 의존성 설치

```bash
helm dependency update mychart
```

### 서브차트 값 구성

```yaml
# values.yaml

cache:
  enabled: true
  architecture: standalone
  auth:
    enabled: false

global:
  imagePullSecrets:
    - name: regcred
```

- `global` 키 아래의 값은 모든 서브차트에 전파됩니다.
- 서브차트는 해당 차트의 이름(또는 alias)으로 네임스페이스처럼 값을 구분하여 참조합니다.

---

## 고급 템플릿 패턴

### `tpl` 함수 — 값 내부의 템플릿 렌더링

{% raw %}
```yaml
# values.yaml
config:
  welcome: "Hello {{ .Release.Name }}"

# ConfigMap 템플릿
data:
  message: |-
    {{ tpl .Values.config.welcome . }}
```
{% endraw %}

### `lookup` 함수 — 클러스터 내 기존 리소스 조회

{% raw %}
```yaml
{{- $cm := (lookup "v1" "ConfigMap" .Release.Namespace "shared-config") -}}
{{- if $cm }}
dataFromCluster: {{ $cm.data | toYaml | nindent 2 }}
{{- end }}
```
{% endraw %}

`lookup` 함수는 선언적 특성을 해칠 수 있으므로, 마이그레이션 등의 특별한 경우에만 신중하게 사용하는 것이 좋습니다.

---

## 주요 Helm 명령어

```bash
# 템플릿 렌더링 결과 확인 (실제 적용 없음)
helm template web ./mychart -f values-prod.yaml

# 차트 문법 및 구조 검사
helm lint ./mychart

# 설치 또는 업그레이드
helm upgrade --install web ./mychart -n app --create-namespace -f values-prod.yaml

# 변경점 비교 (helm-diff 플러그인)
helm plugin install https://github.com/databus23/helm-diff
helm diff upgrade web ./mychart -f values-prod.yaml

# 릴리스 이력 확인 및 롤백
helm history web
helm rollback web 2

# 테스트 실행
helm test web

# 차트 패키징
helm package mychart
```

---

## 보안 및 구성 분리

Helm 자체는 Secret 암호화를 제공하지 않습니다. 일반적으로 다음과 같은 외부 도구와 연계하여 비밀 정보를 관리합니다.

- **Sealed Secrets**
- **External Secrets Operator**
- **SOPS** (helm-secrets 플러그인과 함께)

예를 들어, External Secrets Operator를 사용하는 경우:

{% raw %}
```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: {{ include "mychart.fullname" . }}-db
spec:
  secretStoreRef:
    name: {{ .Values.secrets.storeName }}
    kind: ClusterSecretStore
  data:
    - secretKey: DB_PASSWORD
      remoteRef:
        key: {{ .Values.secrets.db.passwordKey }}
```
{% endraw %}

---

## 릴리스 전략 예시 (Blue-Green)

네임스페이스 또는 릴리스 이름을 분리하여 Blue-Green 배포를 구현할 수 있습니다.

```bash
# Blue 버전 배포
helm upgrade --install web-blue ./mychart -n app -f values-blue.yaml
# Green 버전 배포
helm upgrade --install web-green ./mychart -n app -f values-green.yaml
# 트래픽을 Green으로 전환 및 검증 후 Blue 삭제
helm uninstall web-blue -n app
```

---

## 실전 예제: 단일 포트 웹 애플리케이션 차트

### values.yaml

```yaml
replicaCount: 2
image:
  repository: ghcr.io/acme/webapp
  tag: "1.0.0"
service:
  type: ClusterIP
  port: 8080
ingress:
  enabled: true
  className: nginx
  hosts:
    - host: app.example.com
      paths:
        - path: /
          pathType: Prefix
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 200m
    memory: 256Mi
```

### templates/deployment.yaml

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
      {{- include "mychart.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "mychart.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: app
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet:
              path: /readyz
              port: 8080
          livenessProbe:
            httpGet:
              path: /livez
              port: 8080
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```
{% endraw %}

### templates/service.yaml

{% raw %}
```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "mychart.fullname" . }}
spec:
  type: {{ .Values.service.type }}
  selector:
    {{- include "mychart.selectorLabels" . | nindent 4 }}
  ports:
    - name: http
      port: {{ .Values.service.port }}
      targetPort: 8080
```
{% endraw %}

### templates/ingress.yaml

{% raw %}
```yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "mychart.fullname" . }}
spec:
  ingressClassName: {{ .Values.ingress.className }}
  rules:
    {{- range .Values.ingress.hosts }}
    - host: {{ .host }}
      http:
        paths:
          {{- range .paths }}
          - path: {{ .path }}
            pathType: {{ .pathType }}
            backend:
              service:
                name: {{ include "mychart.fullname" $ }}
                port:
                  number: {{ $.Values.service.port }}
          {{- end }}
    {{- end }}
{{- end }}
```
{% endraw %}

---

## 결론

Helm Chart는 Go 템플릿을 통해 쿠버네티스 매니페스트를 생성하는 표준화된 패키징 시스템입니다. 기본적인 `templates/`와 `values.yaml` 조합에서 출발하여, `_helpers.tpl`을 통한 일관된 네이밍, `values.schema.json`을 이용한 구성 검증, 훅과 테스트를 통한 수명주기 관리, 서브차트를 활용한 모듈화 등을 점진적으로 도입할 수 있습니다.

운영 환경에서는 환경별 values 파일을 체계적으로 관리하고, 비밀 정보는 외부 시스템에 위임하며, `--atomic` 및 `--wait` 플래그를 활용한 안전한 배포 자동화를 구축하는 것이 중요합니다. 또한 CRD와 같은 민감한 리소스는 차트의 수명주기와 분리하여 관리함으로써 시스템의 안정성을 높일 수 있습니다.

이 문서에 소개된 구조와 패턴은 재현 가능하고 유지보수성이 높은 배포 파이프라인을 구축하는 데 유용한 기반이 될 것입니다.