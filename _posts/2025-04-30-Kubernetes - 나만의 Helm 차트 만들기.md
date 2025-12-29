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

생성된 차트 구조:

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

각 파일의 역할:
- `Chart.yaml`: 차트의 이름, 버전, 의존성 등 메타데이터 정의
- `values.yaml`: 템플릿에 주입할 기본 구성값으로 환경별로 오버라이드 가능
- `templates/`: Go 템플릿 문법으로 작성된 쿠버네티스 매니페스트
- `charts/`: 서브차트(의존성) 저장 디렉터리
- `_helpers.tpl`: 차트 전반에서 재사용할 네이밍 및 라벨 함수 정의
- `NOTES.txt`: 설치 완료 후 사용자에게 표시할 안내 메시지

---

## values.yaml 설계 — 환경별 구성 분리

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

설계 원칙:
- **이미지 태그는 명시적으로 고정**하여 동일한 버전의 재현 가능한 배포 보장
- 환경별 차이(로그 레벨, 리소스, Ingress 설정 등)는 별도의 values 파일로 관리
- 기본값은 개발 환경에 적합하도록 설정하고, 운영 환경에서는 오버라이드

---

## 템플릿 커스터마이징: Deployment

`templates/deployment.yaml`의 핵심 부분:

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

템플릿 작성 시 유의사항:
- `include` 함수로 헬퍼 템플릿 호출하여 일관된 네이밍 유지
- `toYaml`과 `nindent` 조합으로 가독성 있는 YAML 출력
- `required` 함수로 필수 값 검증 실패 시 조기에 오류 발생
- `default`, `quote` 등의 함수로 안전한 값 처리

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

## Ingress 템플릿 (선택적)

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

## _helpers.tpl — 네이밍 및 라벨 표준화

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

- `app.kubernetes.io/*` 표준 라벨 채택: 쿠버네티스 에코시스템과의 호환성 향상
- 리소스 이름 길이 제한(63자) 준수를 위해 `trunc` 함수 사용
- 일관된 라벀링으로 모니터링, 로깅, 서비스 디스커버리 용이

---

## 유용한 템플릿 기법

### 조건문과 반복문 활용

{% raw %}
```yaml
{{- if .Values.metrics.enabled }}
# 메트릭스 수집 관련 리소스 생성
{{- end }}

env:
  {{- range $k, $v := .Values.extraEnv }}
  - name: {{ $k }}
    value: {{ $v | quote }}
  {{- end }}
```
{% endraw %}

### `tpl` 함수를 이용한 동적 템플릿 렌더링

values에 저장된 템플릿 문자열을 실행 시점에 해석:

{% raw %}
```yaml
{{- $rendered := tpl (.Values.someTemplatedString | default "") . -}}
```
{% endraw %}

### 파일 내용 직접 삽입

{% raw %}
```yaml
data:
  application.yaml: |
    {{- .Files.Get "files/application.yaml" | nindent 4 }}
```
{% endraw %}

### 필수 값 검증과 기본값 처리

{% raw %}
```yaml
- name: DB_HOST
  value: {{ required "db.host is required" .Values.db.host | quote }}
- name: SECRET_B64
  value: {{ .Values.secret | b64enc }}
```
{% endraw %}

---

## values.schema.json — 구성값 검증

차트 루트에 JSON 스키마를 추가하면 `helm install` 또는 `helm upgrade` 명령 실행 시 값의 유효성을 검증할 수 있습니다.

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

스키마 검증의 장점:
- 오타나 형식 오류를 배포 전 조기에 발견
- 런타임 장애 예방 및 안정성 향상

---

## 설치 전 검증 루틴

```bash
# 템플릿 렌더링 결과 미리보기
helm template myrel ./mychart -f values-dev.yaml | tee rendered.yaml

# 차트 문법 및 구조 검사
helm lint ./mychart

# 쿠버네티스 매니페스트 스키마 검증 (kubeconform 사용)
helm template myrel ./mychart -f values-dev.yaml | kubeconform -strict -
```

변경 사항 비교 (helm-diff 플러그인):

```bash
helm plugin install https://github.com/databus23/helm-diff
helm diff upgrade myrel ./mychart -f values-prod.yaml
```

---

## 안전한 설치 및 업그레이드

```bash
helm upgrade --install myapp ./mychart \
  -f values-prod.yaml \
  --atomic --wait --timeout 5m
```

- `--atomic`: 실패 시 자동 롤백
- `--wait`: 모든 리소스가 준비 상태가 될 때까지 대기
- `--timeout`: 작업 시간 상한 지정

릴리스 상태 확인 및 관리:

```bash
helm status myapp
helm history myapp
helm rollback myapp 2
```

---

## Hook과 Test를 이용한 배포 자동화

### 데이터베이스 마이그레이션 Hook (Job)

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

### 설치 검증 테스트 (helm test)

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

테스트 실행:

```bash
helm test myapp
```

### NOTES.txt — 설치 후 사용자 안내

{% raw %}
```
1) Service 확인:
   kubectl get svc {{ include "mychart.fullname" . }}

2) 로컬에서 접근하기 (포트 포워딩):
   kubectl port-forward svc/{{ include "mychart.fullname" . }} {{ .Values.service.port }}:{{ .Values.service.port }}

3) Ingress 접근:
{{- if .Values.ingress.enabled }}
   https://{{ (index .Values.ingress.hosts 0).host }}
{{- else }}
   (Ingress 비활성화됨)
{{- end }}
```
{% endraw %}

---

## 서브차트와 의존성 관리

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

서브차트 값 구성:

```yaml
# values.yaml

redis:
  architecture: replication
  auth:
    enabled: true
    password: "supersecret"
```

모든 서브차트에 공통으로 적용할 값은 `global` 키 사용:

```yaml
global:
  imagePullSecrets:
    - name: regcred
```

---

## 보안 및 비밀 정보 관리

Helm은 기본적으로 Secret을 암호화하지 않습니다. 운영 환경에서는 다음 방법 중 하나를 선택하여 비밀 정보를 안전하게 관리하는 것이 좋습니다.

1. **SOPS + helm-secrets 플러그인**: 파일 기반 암호화
2. **Sealed Secrets**: 클러스터 내에서만 복호화 가능
3. **External Secrets Operator**: 클라우드 비밀 관리 서비스와 연동

예: helm-secrets 사용

```bash
helm plugin install https://github.com/jkroepke/helm-secrets
# secrets-prod.yaml 파일 암호화
helm secrets enc secrets-prod.yaml

# 암호화된 파일로 배포
helm upgrade --install myapp ./mychart \
  -f values-prod.yaml -f secrets-prod.yaml
```

---

## 고급 기능 연동

### HPA (Horizontal Pod Autoscaler)

```yaml
# values.yaml
autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 8
  targetCPUUtilizationPercentage: 70
```

`templates/hpa.yaml`에서 `autoscaling.enabled` 조건으로 HPA 리소스 생성.

### PDB (Pod Disruption Budget)

```yaml
# values.yaml
pdb:
  enabled: true
  minAvailable: "50%"
```

### 헬스체크 프로브

```yaml
livenessProbe:
  httpGet:
    path: /livez
    port: http
  initialDelaySeconds: 10
readinessProbe:
  httpGet:
    path: /readyz
    port: http
  initialDelaySeconds: 5
```

---

## 라이브러리 차트를 통한 템플릿 재사용

여러 서비스에서 공통으로 사용하는 템플릿은 별도의 라이브러리 차트로 분리할 수 있습니다.

`Chart.yaml`:

```yaml
apiVersion: v2
name: common-templates
version: 1.0.0
type: library
```

애플리케이션 차트에서는 `include` 함수로 라이브러리 차트의 템플릿을 호출합니다. 이 방식은 대규모 조직에서 네이밍, 라벨, 리소스 패턴을 통일하는 데 유용합니다.

---

## OCI 레지스트리를 이용한 차트 배포

컨테이너 레지스트리에 Helm 차트를 저장하고 배포할 수 있습니다.

```bash
export HELM_EXPERIMENTAL_OCI=1

# 차트 패키징
helm package ./mychart

# 레지스트리 로그인
helm registry login ghcr.io

# 차트 업로드
helm push mychart-1.2.0.tgz oci://ghcr.io/acme/helm

# 차트 다운로드
helm pull oci://ghcr.io/acme/helm/mychart --version 1.2.0
```

---

## GitOps/CI 파이프라인 통합

Argo CD나 Flux와 같은 GitOps 도구를 사용하면 Git 저장소의 차트와 values 변경사항을 클러스터에 자동으로 동기화할 수 있습니다.

CI/CD 파이프라인 예시:

```bash
# 차트 문법 검사
helm lint ./mychart

# 템플릿 렌더링 및 스키마 검증
helm template myrel ./mychart -f values-prod.yaml | kubeconform -strict -

# 변경사항 비교
helm diff upgrade myrel ./mychart -f values-prod.yaml

# 안전한 배포
helm upgrade --install myrel ./mychart -f values-prod.yaml --atomic --wait --timeout 10m

# 테스트 실행
helm test myrel
```

---

## 운영 문제 해결 가이드

1. **렌더링 결과 확인**: `helm template --debug --dry-run`
2. **변경사항 비교**: `helm diff upgrade` 명령으로 예상 변경 확인
3. **대기 및 롤백**: `--wait --atomic --timeout` 옵션으로 안전한 배포
4. **네임스페이스 확인**: `-n` 옵션을 일관되게 사용
5. **이미지 풀 오류**: `imagePullSecrets` 및 레지스트리 권한 확인
6. **CRD 확인**: 차트에서 요구하는 Custom Resource Definition 사전 설치
7. **스토리지 문제**: StorageClass 및 PVC 바인딩 확인
8. **Ingress 설정**: Ingress 클래스, 어노테이션, 호스트, TLS Secret 확인
9. **리소스 제한**: Requests/Limits 및 HPA 임계값 적절성 검토
10. **이벤트 및 로그**: `kubectl describe`와 `kubectl logs`로 문제 진단

---

## 환경별 values 파일 예시

### 개발 환경 (values-dev.yaml)

```yaml
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

### 운영 환경 (values-prod.yaml)

```yaml
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
  requests:
    cpu: 300m
    memory: 512Mi
  limits:
    cpu: 800m
    memory: 1Gi
pdb:
  enabled: true
  minAvailable: "50%"
```

환경별 배포:

```bash
# 개발 환경
helm upgrade --install myapp ./mychart -f values-dev.yaml

# 운영 환경
helm upgrade --install myapp ./mychart -f values-prod.yaml
```

---

## Helm 명령어 참고 사항

- **리포지터리 관리**: `helm repo add <name> <url>`, `helm repo update`
- **차트 검색 및 정보 확인**: `helm search repo <keyword>`, `helm show values <chart>`
- **렌더링 및 검증**: `helm template`, `helm lint`, `helm diff upgrade`
- **배포 및 관리**: `helm upgrade --install`, `helm status`, `helm history`, `helm rollback`
- **테스트**: `helm test <release>`

---

## 결론

자체 Helm 차트를 개발하면 다음과 같은 이점을 얻을 수 있습니다:

1. **표준화 및 재사용성**: 모든 쿠버네티스 매니페스트를 템플릿화하여 중복을 제거하고, 환경별 구성값을 체계적으로 분리할 수 있습니다.

2. **운영 안정성**: 스키마 검증, Hook, 테스트, 이력 관리, 롤백 기능을 통해 배포 프로세스의 안정성을 크게 향상시킬 수 있습니다.

3. **조직 표준화**: 헬퍼 템플릿과 라이브러리 차트를 통해 네이밍 규칙, 라벨 체계, 리소스 패턴을 조직 전체에 일관되게 적용할 수 있습니다.

4. **보안 강화**: 비밀 정보 관리 도구와의 통합을 통해 민감한 데이터를 안전하게 처리할 수 있습니다.

차트 개발은 `helm create`로 생성한 기본 구조에서 시작하여, 점진적으로 values 스키마 검증, 표준화된 헬퍼 함수, Hook 및 테스트, 보안 메커니즘을 추가해 나가는 것이 좋습니다. 이를 통해 팀의 표준이 되고 프로덕션 환경에서도 신뢰할 수 있는 차트로 발전시킬 수 있습니다.