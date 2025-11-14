---
layout: post
title: Kubernetes - Helm Chart
date: 2025-04-30 20:20:23 +0900
category: Kubernetes
---
# Helm Chart 구조

## 기본 디렉터리 구조와 생성

```bash
helm create mychart
tree mychart
```

```text
mychart/
├── Chart.yaml            # 차트 메타데이터(필수)
├── values.yaml           # 기본 값(환경별 오버라이드 가능)
├── values.schema.json    # 값 스키마(선택: JSONSchema)
├── charts/               # 서브차트(.tgz 또는 디렉터리)
├── templates/            # 리소스 템플릿(YAML+Go 템플릿)
│   ├── _helpers.tpl      # 공통 템플릿 함수/네이밍 규칙
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── hpa.yaml
│   ├── NOTES.txt         # 설치 후 안내 메시지
│   └── tests/            # helm test용 Job
├── crds/                 # CRD 매니페스트(컨트롤러 리소스)
└── .helmignore           # 패키징 제외 목록
```

핵심 파일:

- **Chart.yaml**: 차트 이름/버전/의존성 메타데이터
- **values.yaml**: 템플릿에 주입될 기본 값
- **templates/**: 실제 쿠버네티스 리소스 템플릿(Go 템플릿 문법)
- **_helpers.tpl**: 공통 네이밍/라벨 함수 정의
- **values.schema.json**: 값 검증(미스매치를 설치 전에 잡는다)
- **crds/**: CRD는 **일반 템플릿과 별도**로 처리된다(삭제/업데이트 주의)
- **charts/**: 서브차트(의존성) 보관

---

## Chart.yaml

```yaml
apiVersion: v2
name: mychart
description: A sample Helm chart for Kubernetes
type: application           # or library
version: 1.2.0              # 차트 버전(semver)
appVersion: "1.25.3"        # 앱 버전(정보용)
kubeVersion: ">=1.25.0-0"   # 호환 K8s 버전(선택)
dependencies:
  - name: redis
    version: 18.6.2
    repository: https://charts.bitnami.com/bitnami
    alias: cache            # 하위 차트 접근명 변경(선택)
    condition: cache.enabled
```

- **type: library** 를 사용하면 리소스 생성 없이 템플릿 재사용 전용 차트를 만들 수 있다.
- **dependencies**는 `helm dependency update`로 `charts/`에 가져온다.

---

## values.yaml — 기본 값과 환경 오버라이드

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
  tls: []                   # [{ hosts: [...], secretName: ... }]

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

fullnameOverride: ""         # 강제 이름 지정
nameOverride: ""             # 베이스 이름만 오버라이드

global:                      # 서브차트까지 전파되는 글로벌 키
  imagePullSecrets: []
```

- 환경별 파일(예: `values-prod.yaml`)로 오버라이드:
  ```bash
  helm upgrade --install web ./mychart -f values-prod.yaml
  ```

---

## 템플릿 핵심 문법(Go 템플릿 + Sprig)

템플릿은 **Go 템플릿**이며 Sprig 함수가 포함된다.

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

### Service 템플릿(조건부 생성)

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

### Ingress 템플릿(활성화 시에만)

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

## _helpers.tpl — 네이밍/라벨 표준화

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

- **표준 라벨**(app.kubernetes.io/*)을 통일하면 추적성과 선택자가 쉬워진다.

---

## values.schema.json — 값 검증(권장)

설치 전에 **값의 타입·필수성** 오류를 차단한다.

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

검증 실행은 `helm install/upgrade` 시 자동 수행된다.

---

## NOTES.txt — 설치 후 안내 출력

`templates/NOTES.txt`는 사용자에게 **접속 방법/다음 단계**를 알려준다.

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

## Hooks — 설치/업그레이드 수명주기 제어

Job/ConfigMap 등에 **훅 애노테이션**을 달아 특정 타이밍에 실행할 수 있다.

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

- **hook 순서/가중치**로 여러 훅의 실행 순서를 관리한다.
- **delete-policy**로 훅 리소스 수거 정책을 정한다.

---

## Tests — 설치 검증(helm test)

`templates/tests/` 디렉터리의 Job은 `helm test RELEASE`로 실행된다.

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

## CRDs — 컨트롤러 리소스 배포

- **crds/** 디렉터리에 두면 Helm이 **선/후** 순서로 처리하며 기본적으로 **관리/삭제가 보수적**이다.
- CRD 변경은 **가급적 수동 관리**(호환성/마이그레이션 확인). 운영 차트에서는 CRD를 별도 차트로 분리하기도 한다.

---

## 서브차트·글로벌 값·의존성

### 의존성 설치/업데이트

```bash
helm dependency update mychart
```

### 하위 차트 값 주입

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

- **`.Values.global`** 은 모든 서브차트에 전파된다.
- 하위 차트 이름(또는 `alias`)으로 네임스페이스처럼 값을 구분한다.

---

## 고급 템플릿 패턴

### `tpl` — 값 안의 템플릿도 렌더링

{% raw %}
```yaml
# values.yaml

config:
  welcome: "Hello {{ .Release.Name }}"

# ConfigMap

data:
  message: |-
    {{ tpl .Values.config.welcome . }}
```
{% endraw %}

### `lookup` — 클러스터 조회(주의)

{% raw %}
```yaml
{{- $cm := (lookup "v1" "ConfigMap" .Release.Namespace "shared-config") -}}
{{- if $cm }}
dataFromCluster: {{ $cm.data | toYaml | nindent 2 }}
{{- end }}
```
{% endraw %}

- 재현성/테스트성을 해칠 수 있어 **가급적 선언형**으로 유지하되, 마이그레이션/참조 시 신중히 사용.

### 조건부 리소스 생성

{% raw %}
```yaml
{{- if .Values.hpa.enabled }}
# hpa.yaml …

{{- end }}
```
{% endraw %}

---

## 설치·업그레이드·검증 명령어

```bash
# 드라이런 + 렌더 결과 확인(클러스터에 적용 X)

helm template web ./mychart -f values-prod.yaml

# 문법/스키마/베스트 프랙티스 점검

helm lint ./mychart

# 설치 또는 업그레이드(존재하면 upgrade, 없으면 install)

helm upgrade --install web ./mychart -n app --create-namespace -f values-prod.yaml

# 변경점 비교(diff 플러그인)

helm plugin install https://github.com/databus23/helm-diff
helm diff upgrade web ./mychart -f values-prod.yaml

# 되돌리기

helm history web
helm rollback web 2

# 테스트

helm test web

# 패키징/배포

helm package mychart
helm repo index .
```

---

## 보안/설정 분리 — Secret/문자열 렌더링

Helm 자체는 Secret 암호화를 제공하지 않는다. 일반적으로는:

- **Sealed Secrets**, **External Secrets Operator**, **SOPS+Helm(helm-secrets 플러그인)** 등과 연계
- 값 파일에 민감 정보를 두지 말고, 외부 시크릿 매커니즘으로 주입

예: External Secrets Operator 값 연동(개요)

{% raw %}
```yaml
# templates/external-secret.yaml

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

## 릴리스 전략 — Canary/Blue-Green 예시(간단)

Ingress 애노테이션/두 릴리스 병행으로 Canary를 구현하거나, **네임스페이스/이름 분리**로 Blue-Green을 구성한다.

```bash
helm upgrade --install web-blue  ./mychart -n app -f values-blue.yaml
helm upgrade --install web-green ./mychart -n app -f values-green.yaml
# LB/Ingress 백엔드 전환 후 검증 → Blue 삭제

```

---

## 베스트 프랙티스 요약

- **이름/라벨 규격화**: `_helpers.tpl`에 표준 네이밍/라벨 정의
- **스키마 검증 활성화**: `values.schema.json`으로 타입/필수값 보장
- **조건부 리소스**: `.Values.*.enabled`로 선택적 생성
- **템플릿 가독성**: `toYaml | nindent`로 들여쓰기/멀티라인 정리
- **환경별 values 파일**: `values-dev.yaml`, `values-prod.yaml` 등
- **설치 전 렌더/리트(템플릿/스키마/린트)**: `helm template`, `helm lint`
- **릴리스 자동화**: `--atomic`, `--wait`, `--timeout`으로 실패 시 롤백
  ```bash
  helm upgrade --install web ./mychart --atomic --wait --timeout 5m
  ```
- **CRD 분리 배포**: 차트와 수명주기를 분리해 안전하게 관리
- **비밀 관리 외부화**: Sealed/External Secrets 또는 SOPS

---

## 실전 미니 차트 — 단일 포트 웹앱

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
  requests: { cpu: 100m, memory: 128Mi }
  limits:   { cpu: 200m, memory: 256Mi }
```

### templates/deployment.yaml

{% raw %}
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "mychart.fullname" . }}
  labels: {{- include "mychart.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels: {{- include "mychart.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels: {{- include "mychart.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: app
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet: { path: /readyz, port: 8080 }
          livenessProbe:
            httpGet: { path: /livez,  port: 8080 }
          resources: {{- toYaml .Values.resources | nindent 12 }}
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
  selector: {{- include "mychart.selectorLabels" . | nindent 4 }}
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
                port: { number: {{ $.Values.service.port }} }
        {{- end }}
  {{- end }}
{{- end }}
```
{% endraw %}

설치/검증:

```bash
helm upgrade --install web ./mychart -n app --create-namespace
helm get manifest web -n app | less
kubectl -n app get all
```

---

## 결론

Helm Chart는 **템플릿(templates/)** 과 **값(values.yaml)** 을 조합해 Kubernetes 리소스를 일관되게 생성한다. 여기에 **스키마 검증(values.schema.json)**, **네이밍/라벨 표준(_helpers.tpl)**, **훅/테스트**, **서브차트/글로벌 값**, **CRD 취급**, **운영 명령어와 릴리스 전략**을 더하면 **재현 가능하고 신뢰도 높은 배포 파이프라인**을 구현할 수 있다. 실제 팀/프로덕션에서는 본문 패턴을 기본 골격으로 삼아, **환경별 values, 비밀 관리 외부화, 릴리스 안전장치(atomic/wait/timeout), 관측/롤백 전략**을 체계화하길 권한다.
