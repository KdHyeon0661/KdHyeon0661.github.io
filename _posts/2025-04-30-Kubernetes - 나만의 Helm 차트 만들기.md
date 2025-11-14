---
layout: post
title: Kubernetes - 나만의 Helm 차트 만들기
date: 2025-04-30 22:20:23 +0900
category: Kubernetes
---
# 나만의 Helm 차트 만들기

## 스캐폴딩: 차트 생성

```bash
helm create mychart
```

생성 구조:

```
mychart/
├── Chart.yaml
├── values.yaml
├── charts/
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── hpa.yaml
│   ├── ingress.yaml
│   ├── _helpers.tpl
│   └── NOTES.txt
└── .helmignore
```

- `Chart.yaml`: 차트 메타데이터(이름, 버전 등)
- `values.yaml`: 기본 설정값(환경별 오버라이드 대상)
- `templates/`: Kubernetes 매니페스트 템플릿(Go 템플릿)
- `charts/`: 서브차트(의존 차트)
- `_helpers.tpl`: 네이밍/라벨 등 공통 템플릿 함수
- `NOTES.txt`: 설치 후 안내 메시지

---

## values.yaml 설계 — “환경 독립 이미지 + 환경 종속 설정” 분리

```yaml
replicaCount: 2

image:
  repository: ghcr.io/acme/myapp
  tag: "1.4.0"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 8080

env:
  - name: LOG_LEVEL
    value: info

resources:
  requests:
    cpu: 200m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi

ingress:
  enabled: false
  className: ""
  hosts: []
  tls: []

podAnnotations: {}
nodeSelector: {}
tolerations: []
affinity: {}
```

핵심 관점:
- **이미지 태그는 명시적으로 고정**(latest 지양) → 재현성
- env/리소스/Ingress 등은 **환경별 파일**로 오버라이드

---

## 템플릿 커스터마이징: Deployment

`templates/deployment.yaml` (핵심 부분만 발췌)

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
      annotations:
        {{- toYaml .Values.podAnnotations | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ required "image.tag must be set" .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy | default "IfNotPresent" }}
          ports:
            - containerPort: {{ .Values.service.port }}
              name: http
          env:
            {{- range .Values.env }}
            - name: {{ .name }}
              value: {{ quote .value }}
            {{- end }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
      {{- end }}
```
{% endraw %}

템플릿 포인트:
- `include`로 헬퍼 호출, `toYaml|nindent`로 들여쓰기·가독성 유지
- `required`로 필수 값 검증 실패 시 렌더 단계에서 에러
- `default`, `quote` 등 함수로 안전한 값 처리

---

## Service 템플릿

{% raw %}
```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "mychart.fullname" . }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
spec:
  type: {{ .Values.service.type | default "ClusterIP" }}
  ports:
    - name: http
      port: {{ .Values.service.port }}
      targetPort: http
  selector:
    {{- include "mychart.selectorLabels" . | nindent 4 }}
```
{% endraw %}

---

## Ingress(옵션)

{% raw %}
```yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "mychart.fullname" . }}
  annotations:
    {{- if .Values.ingress.className }}
    kubernetes.io/ingress.class: {{ .Values.ingress.className }}
    {{- end }}
spec:
  {{- if .Values.ingress.className }}
  ingressClassName: {{ .Values.ingress.className }}
  {{- end }}
  rules:
    {{- range $host := .Values.ingress.hosts }}
    - host: {{ $host.host | quote }}
      http:
        paths:
          {{- range $p := $host.paths }}
          - path: {{ $p.path }}
            pathType: {{ $p.pathType | default "Prefix" }}
            backend:
              service:
                name: {{ include "mychart.fullname" $ }}
                port:
                  number: {{ $.Values.service.port }}
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
```yaml
{{- define "mychart.name" -}}
{{ .Chart.Name }}
{{- end }}

{{- define "mychart.fullname" -}}
{{- if .Values.fullnameOverride -}}
{{ .Values.fullnameOverride }}
{{- else -}}
{{ printf "%s-%s" .Release.Name .Chart.Name | trunc 63 | trimSuffix "-" }}
{{- end -}}
{{- end }}

{{- define "mychart.labels" -}}
app.kubernetes.io/name: {{ include "mychart.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
helm.sh/chart: {{ printf "%s-%s" .Chart.Name .Chart.Version | quote }}
{{- end }}

{{- define "mychart.selectorLabels" -}}
app.kubernetes.io/name: {{ include "mychart.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}
```
{% endraw %}

- **app.kubernetes.io/\*** 표준 라벨을 일관되게 사용 → 운영·모니터링·셀렉터에 유리
- 길이 제한(63) 고려해 `trunc` 사용

---

## 다양한 템플릿 기법 모음

### 조건/반복/기본값

{% raw %}
```yaml
{{- if .Values.metrics.enabled }}
# ServiceMonitor 혹은 PodMonitor 등 CRD 사용 시

{{- end }}

env:
  {{- range $k, $v := .Values.extraEnv }}
  - name: {{ $k }}
    value: {{ $v | quote }}
  {{- end }}
```
{% endraw %}

### `tpl`로 동적 렌더링

values에 템플릿 문자열이 들어온 경우 해석:

{% raw %}
```yaml
{{- $rendered := tpl (.Values.someTemplatedString | default "") . -}}
```
{% endraw %}

### 파일 삽입

{% raw %}
```yaml
data:
  application.yaml: |
    {{- .Files.Get "files/application.yaml" | nindent 4 }}
```
{% endraw %}

### `required`, `default`, `quote`, `b64enc`

{% raw %}
```yaml
- name: DB_HOST
  value: {{ required "db.host is required" .Values.db.host | quote }}
- name: SECRET_B64
  value: {{ .Values.secret | b64enc }}
```
{% endraw %}

---

## 값 스키마(검증) — `values.schema.json`

차트 루트에 JSONSchema를 두면 `helm install/upgrade` 시 값 검증이 가능하다.

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
        "tag": { "type": "string", "minLength": 1 }
      },
      "required": ["repository","tag"]
    },
    "service": {
      "type": "object",
      "properties": {
        "type": { "type": "string", "enum": ["ClusterIP","NodePort","LoadBalancer"] },
        "port": { "type": "integer", "minimum": 1, "maximum": 65535 }
      },
      "required": ["type","port"]
    }
  },
  "required": ["replicaCount","image","service"]
}
```

장점
- 파이프라인 초기에 **오타·형식 오류** 차단
- 런타임 장애를 릴리즈 이전에 예방

---

## 설치 전 검증 루틴

```bash
# 렌더 결과 미리보기

helm template myrel ./mychart -f values-dev.yaml | tee rendered.yaml

# 차트 린트

helm lint ./mychart

# (권장) kubeconform 등으로 스키마 검증

helm template myrel ./mychart -f values-dev.yaml | kubeconform -strict -
```

변경 전후 비교(helm-diff 플러그인):

```bash
helm plugin install https://github.com/databus23/helm-diff
helm diff upgrade myrel ./mychart -f values-prod.yaml
```

---

## 설치/업그레이드 — 안전 옵션

```bash
helm upgrade --install myapp ./mychart \
  -f values-prod.yaml \
  --atomic --wait --timeout 5m
```

- `--atomic`: 실패 시 자동 롤백
- `--wait`: Ready까지 대기
- `--timeout`: 상한 지정

릴리즈 상태/이력/롤백:

```bash
helm status myapp
helm history myapp
helm rollback myapp 2
```

---

## Hook & Test — 배포 전후 작업/검증 자동화

### 마이그레이션 Hook(Job)

{% raw %}
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: "{{ include "mychart.fullname" . }}-migrate"
  annotations:
    "helm.sh/hook": pre-install,pre-upgrade
    "helm.sh/hook-weight": "10"
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  template:
    spec:
      restartPolicy: OnFailure
      containers:
        - name: migrate
          image: ghcr.io/acme/migrator:1.4.0
          args: ["./migrate.sh"]
```
{% endraw %}

### 설치 검증 테스트(helm test)

{% raw %}
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: "{{ include "mychart.fullname" . }}-test"
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

실행:

```bash
helm test myapp
```

### NOTES.txt — 설치 후 안내

`templates/NOTES.txt` 예시:

{% raw %}
```
1) Service:
   kubectl get svc {{ include "mychart.fullname" . }}

2) Port-forward:
   kubectl port-forward svc/{{ include "mychart.fullname" . }} {{ .Values.service.port }}:{{ .Values.service.port }}

3) Ingress:
{{- if .Values.ingress.enabled }}
   https://{{ (index .Values.ingress.hosts 0).host }}
{{- else }}
   (ingress disabled)
{{- end }}
```
{% endraw %}

---

## 서브차트/의존성 — charts/ & 글로벌 값

`Chart.yaml`에 의존성 선언:

```yaml
apiVersion: v2
name: mychart
version: 1.2.0

dependencies:
  - name: redis
    version: 18.1.3
    repository: https://charts.bitnami.com/bitnami
```

의존성 다운로드:

```bash
helm dependency update ./mychart
```

서브차트 값 주입:

```yaml
# values.yaml

redis:
  architecture: replication
  auth:
    enabled: true
    password: "supersecret"
```

모든 서브차트에 공통 적용하고 싶으면 **global** 키 사용:

```yaml
global:
  imagePullSecrets:
    - name: regcred
```

---

## 보안/비밀 관리 — 운영 체크리스트

- Secret은 **base64 인코딩일 뿐 암호화 아님** → **etcd 암호화**, RBAC 최소 권한
- 값 파일에 비밀번호 평문 저장 금지 → 다음 중 택일/혼용
  - SOPS + helm-secrets 플러그인
  - Sealed Secrets(클러스터에서 복호화)
  - External Secrets Operator(클라우드 비밀 관리자 연동)

예: helm-secrets

```bash
helm plugin install https://github.com/jkroepke/helm-secrets
# secrets-prod.yaml 을 sops 로 암호화

helm secrets enc secrets-prod.yaml
helm upgrade --install myapp ./mychart \
  -f values-prod.yaml -f secrets-prod.yaml
```

---

## HPA/PDB/보건검진(Probe) 연계

### HPA 활성화(예시)

```yaml
# values.yaml

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 8
  targetCPUUtilizationPercentage: 70
```

`templates/hpa.yaml`에서 `autoscaling.enabled` 조건으로 렌더.

### PDB(중단 예산)

```yaml
# values.yaml

pdb:
  enabled: true
  minAvailable: "50%"
```

### Probe

```yaml
livenessProbe:
  httpGet: { path: /livez, port: http }
  initialDelaySeconds: 10
readinessProbe:
  httpGet: { path: /readyz, port: http }
  initialDelaySeconds: 5
```

---

## 고급: Library Chart(공통 템플릿) 분리

여러 서비스에서 동일한 템플릿을 재사용하려면 **type=library** 차트를 별도로 만들고, 앱 차트에서 `include`로 호출한다.
대규모 조직에서 네이밍/라벨/리소스/Sidecar 패턴을 통일할 때 유용.

`Chart.yaml`:

```yaml
type: library
```

---

## OCI 레지스트리로 차트 배포

컨테이너 레지스트리에 차트를 저장/배포(감사·권한·캐시 장점)

```bash
export HELM_EXPERIMENTAL_OCI=1

helm package ./mychart
helm registry login ghcr.io
helm push mychart-1.2.0.tgz oci://ghcr.io/acme/helm

helm pull oci://ghcr.io/acme/helm/mychart --version 1.2.0
```

---

## GitOps/CI 파이프라인 통합

- **Argo CD/Flux**로 리포지터리 선언적 동기화 (values 파일 레이어링)
- PR 기반 변경 → 렌더/린트/스키마 검증 → diff → 자동 배포
- 샘플 CI 단계:

```bash
helm lint ./mychart
helm template myrel ./mychart -f values-prod.yaml | kubeconform -strict -
helm diff upgrade myrel ./mychart -f values-prod.yaml
helm upgrade --install myrel ./mychart -f values-prod.yaml --atomic --wait --timeout 10m
helm test myrel
```

---

## 운영 트러블슈팅 체크리스트

1. 렌더 미리보기: `helm template --debug --dry-run`
2. 차이 보기: `helm diff upgrade ...`
3. 대기/원자성: `--wait --atomic --timeout`
4. 네임스페이스 일치: `-n` 옵션 일관 유지
5. 이미지 풀 실패: `imagePullSecrets`, 레지스트리 권한
6. CRD 존재 여부: 차트 요구 API/CRD 사전 설치
7. 스토리지: StorageClass/PVC 바인딩, 퍼미션
8. Ingress: 클래스/어노테이션/호스트/TLS Secret 확인
9. 자원: Requests/Limits 과소/과대 및 HPA 임계치
10. 파드 이벤트/로그: `kubectl describe`, `kubectl logs`

---

## 환경별 values 예시(개발/운영)

```yaml
# values-dev.yaml

replicaCount: 1
image:
  tag: "dev-2025-11-01"
env:
  - name: LOG_LEVEL
    value: debug
service:
  type: NodePort
  port: 8080
ingress:
  enabled: false
```

```yaml
# values-prod.yaml

replicaCount: 4
image:
  tag: "1.4.0"
env:
  - name: LOG_LEVEL
    value: info
service:
  type: LoadBalancer
  port: 8080
ingress:
  enabled: true
  className: nginx
  hosts:
    - host: app.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - hosts: [app.example.com]
      secretName: app-tls
resources:
  requests: { cpu: 300m, memory: 512Mi }
  limits:   { cpu: 800m, memory: 1Gi }
pdb:
  enabled: true
  minAvailable: "50%"
```

설치:

```bash
helm upgrade --install myapp ./mychart -f values-dev.yaml     # 개발
helm upgrade --install myapp ./mychart -f values-prod.yaml    # 운영
```

---

## 빠른 점검 명령 모음

| 목적 | 명령 |
|---|---|
| 리포 추가/업데이트 | `helm repo add <n> <url>`, `helm repo update` |
| 검색/기본값 보기 | `helm search repo <kw>`, `helm show values <chart>` |
| 렌더/린트/디프 | `helm template`, `helm lint`, `helm diff upgrade` |
| 설치/업그레이드 | `helm upgrade --install <rel> ./mychart -f ... --atomic --wait` |
| 상태/이력/롤백 | `helm status`, `helm history`, `helm rollback` |
| 테스트 | `helm test <rel>` |

---

## 결론

**자체 Helm 차트**를 만들면 다음을 얻는다.

- 모든 매니페스트를 **템플릿화**하여 **중복 제거**·**환경별 값 분리**·**재현 가능한 배포**
- 스키마 검증/Hook/Test/이력·롤백/OCI/GitOps·CI로 **운영 안전성** 대폭 향상
- 조직 표준(라벨/네이밍/리소스/보안)과 패턴(HPA/PDB/Ingress/Secrets)을 **헬퍼·라이브러리 차트**로 일원화

처음에는 `helm create`의 기본 골격에서 시작하되,
**values 스키마, 헬퍼 표준, Hook/Test, 보안(비밀 관리), 검증 파이프라인**을 더해
프로덕션에서도 견고하게 돌아가는 **팀의 표준 차트**로 발전시켜라.
