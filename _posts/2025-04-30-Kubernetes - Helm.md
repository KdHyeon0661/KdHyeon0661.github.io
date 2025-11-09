---
layout: post
title: Kubernetes - Helm
date: 2025-04-30 19:20:23 +0900
category: Kubernetes
---
# Helm이란? 왜 사용하는가?

Kubernetes는 유연하지만 실제 배포에는 여러 리소스(Deployment, Service, ConfigMap, Secret, Ingress, PVC 등)를 다양한 환경(dev/stage/prod)에 맞춰 관리해야 한다. 수십 개의 YAML을 복사/붙여넣기하며 값만 바꾸는 접근은 **중복·실수·검증 난이도**를 키운다. Helm은 이를 **템플릿(templates) + 값(values)** 구조로 표준화해, **한 번의 명령**으로 설치/업그레이드/롤백 가능한 **패키지 매니저** 역할을 한다.

---

## 개념 요약

- **Chart**: Kubernetes 애플리케이션 패키지(메타데이터 + 템플릿 + 기본값).
- **Release**: Chart가 **클러스터에 설치된 인스턴스**. 동일 차트를 여러 번 설치 가능(이름만 다르면 독립 릴리스).
- **Values**: 템플릿에 주입되는 **환경별 설정값**(파일, CLI, CI/CD 변수로 오버라이드).
- **Repository/OCI**: Chart를 배포·공유하는 저장소(artifacthub, 사내 Helm repo, OCI 레지스트리).

---

## Helm을 쓰면 해결되는 것

| 이슈 | 기존 방식 | Helm 도입 후 |
|---|---|---|
| 환경별 YAML 관리 | dev/prod 파일 따로 복제 | values 파일만 분리(템플릿 재사용) |
| 변경 추적/되돌리기 | kubectl apply 뒤 추적 난해 | `helm history/rollback` |
| 설치 원자성/검증 | 설치 중간 실패 시 중복 리소스 | `--atomic --wait`로 일관성 보장 |
| 배포 품질 | 사람 의존 스텝/체크리스트 | `helm lint/test`로 자동 점검 |
| 공유/재사용 | 문서/복붙 | Chart 저장소/OCI로 패키징 배포 |

---

## 빠른 시작(핵심 명령)

```bash
# 차트 스캐폴드
helm create mychart

# 렌더 결과(적용 전 확인)
helm template myrel ./mychart -f values-prod.yaml

# 규칙/구문 점검
helm lint ./mychart

# 설치(없으면 설치, 있으면 업그레이드)
helm upgrade --install myrel ./mychart -n app --create-namespace \
  -f values.yaml -f values-prod.yaml --atomic --wait --timeout 5m

# 상태/이력/롤백
helm list -n app
helm history myrel -n app
helm rollback myrel 2 -n app

# 제거
helm uninstall myrel -n app
```

---

## Chart 기본 구조와 파일 역할

```text
mychart/
├── Chart.yaml           # 차트 메타(이름, 버전, 의존성)
├── values.yaml          # 기본값(환경별 파일로 오버라이드)
├── values.schema.json   # 값 스키마(JSONSchema, 선택이지만 강력 권장)
├── templates/           # K8s 템플릿(YAML+Go template)
│   ├── _helpers.tpl     # 네이밍/라벨 함수 등 공통 템플릿
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── NOTES.txt        # 설치 후 안내 메시지
│   └── tests/           # helm test 리소스(Job/Pod)
├── charts/              # 서브차트(의존성)
├── crds/                # CRD(별도 수명주기 주의)
└── .helmignore          # 패키징 제외 목록
```

---

## 최소 실전 예제(하나의 웹앱)

### values.yaml

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
  hosts: []  # [{ host: app.example.com, paths: [{ path: "/", pathType: Prefix }] }]
  tls: []    # [{ hosts: [app.example.com], secretName: app-tls }]

resources:
  requests: { cpu: 100m, memory: 128Mi }
  limits:   { cpu: 200m, memory: 256Mi }

env:
  - name: LOG_LEVEL
    value: info

nodeSelector: {}
tolerations: []
affinity: {}
```

### templates/_helpers.tpl

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

### templates/deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "mychart.fullname" . }}
  labels: {{- include "mychart.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount | default 1 }}
  selector:
    matchLabels: {{- include "mychart.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels: {{- include "mychart.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: app
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy | default "IfNotPresent" }}
          ports:
            - containerPort: 8080
          env:
            {{- range $e := .Values.env }}
            - name: {{ $e.name }}
              value: {{ $e.value | quote }}
            {{- end }}
          readinessProbe:
            httpGet: { path: /readyz, port: 8080 }
          livenessProbe:
            httpGet: { path: /livez,  port: 8080 }
          resources: {{- toYaml .Values.resources | nindent 12 }}
```

### templates/service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "mychart.fullname" . }}
  labels: {{- include "mychart.labels" . | nindent 4 }}
spec:
  type: {{ .Values.service.type }}
  selector: {{- include "mychart.selectorLabels" . | nindent 4 }}
  ports:
    - name: http
      port: {{ .Values.service.port }}
      targetPort: 8080
```

### templates/ingress.yaml

```yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "mychart.fullname" . }}
spec:
  ingressClassName: {{ .Values.ingress.className | default "nginx" }}
  rules:
  {{- range .Values.ingress.hosts }}
    - host: {{ .host | quote }}
      http:
        paths:
        {{- range .paths }}
          - path: {{ .path }}
            pathType: {{ .pathType | default "Prefix" }}
            backend:
              service:
                name: {{ include "mychart.fullname" $ }}
                port: { number: {{ $.Values.service.port }} }
        {{- end }}
  {{- end }}
  {{- if .Values.ingress.tls }}
  tls:
    {{- toYaml .Values.ingress.tls | nindent 4 }}
  {{- end }}
{{- end }}
```

### 설치/업그레이드

```bash
helm upgrade --install web ./mychart -n app --create-namespace \
  -f values.yaml -f values-prod.yaml --atomic --wait
kubectl -n app get all
```

---

## 환경별 값 관리 패턴

- 파일 레이어링: `-f values.yaml -f values-stage.yaml -f values-stage-us-east-1.yaml`
- CLI 오버라이드(드리프트 주의): `--set image.tag=1.1.0 --set service.type=LoadBalancer`
- CI/CD 변수 주입: `--set-string git.sha=$GIT_COMMIT`
- 스키마 검증으로 오탈자·타입 오류 사전 차단

### values.schema.json(강력 권장)

```json
{
  "$schema": "https://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "replicaCount": { "type": "integer", "minimum": 1 },
    "service": {
      "type": "object",
      "properties": {
        "type": { "type": "string", "enum": ["ClusterIP", "NodePort", "LoadBalancer"] },
        "port": { "type": "integer", "minimum": 1, "maximum": 65535 }
      },
      "required": ["type", "port"]
    }
  },
  "required": ["replicaCount", "service"]
}
```

---

## 설치 신뢰도 높이기

- **사전 렌더/검토**: `helm template`, `helm lint`
- **원자적 설치**: `--atomic --wait --timeout 5m`
- **환경 검증**: `--dry-run --debug`로 렌더/훅 흐름 확인
- **변경점 비교**: `helm-diff` 플러그인(업그레이드 전 diff)

```bash
helm plugin install https://github.com/databus23/helm-diff
helm diff upgrade web ./mychart -f values-prod.yaml
```

---

## 훅(Hooks)과 테스트(helm test)

### Hooks(예: 마이그레이션 사전 실행)

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: "{{ include "mychart.fullname" . }}-db-migrate"
  annotations:
    "helm.sh/hook": pre-install,pre-upgrade
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

### Test(설치 후 가용성 확인)

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

```bash
helm test web -n app
```

---

## 차트 의존성·서브차트·글로벌 값

### Chart.yaml

```yaml
dependencies:
  - name: redis
    version: 18.6.2
    repository: https://charts.bitnami.com/bitnami
    alias: cache
    condition: cache.enabled
```

```bash
helm dependency update ./mychart
```

### values.yaml

```yaml
cache:
  enabled: true
  architecture: standalone

global:
  imagePullSecrets:
    - name: regcred
```

- `global.*` 키는 하위 차트까지 전파된다.
- `alias`로 접근 네임을 바꿔 충돌을 피한다.

---

## Helm 저장소와 OCI 레지스트리

### Helm repo(HTTP)

```bash
helm package mychart
helm repo index .
helm repo add acme https://repo.acme.local/helm
helm push mychart-1.2.0.tgz acme
```

### OCI(Helm v3, 권장 추세)

```bash
export HELM_EXPERIMENTAL_OCI=1
helm registry login ghcr.io
helm package mychart
helm push mychart-1.2.0.tgz oci://ghcr.io/acme/helm
helm pull  oci://ghcr.io/acme/helm/mychart --version 1.2.0
```

OCI는 **표준 컨테이너 레지스트리**로 권한·감사·미러링을 통합 관리하기 좋다.

---

## 비밀 관리와 보안(Helm의 한계와 연계 패턴)

- Helm은 값 파일 암호화를 제공하지 않는다.
- 권장 패턴
  - **Sealed Secrets**: 암호화된 Secret을 Git에 보관 → 클러스터에서 복호화
  - **External Secrets Operator**: 클라우드 시크릿 매니저(AWS/GCP/Azure)에서 동기화
  - **SOPS + helm-secrets**: 값 파일을 git에 암호화 상태로 저장

예: External Secrets Operator 연동 템플릿

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: {{ include "mychart.fullname" . }}-db
spec:
  secretStoreRef:
    kind: ClusterSecretStore
    name: {{ .Values.secrets.storeName }}
  target:
    name: {{ include "mychart.fullname" . }}-db
  data:
    - secretKey: DB_PASSWORD
      remoteRef:
        key: {{ .Values.secrets.db.passwordKey }}
```

추가적으로:
- PodSecurity/NetworkPolicy/PSA 등 정책과 함께 사용.
- 이미지 서명/검증(Cosign, policy-controller)로 서플라이 체인 강화.
- `values.schema.json`으로 값 오탈자/형식 오류 차단.

---

## Kustomize와 비교(상황별 선택)

| 항목 | Helm | Kustomize |
|---|---|---|
| 템플릿(조건/반복/함수) | Go 템플릿(강력) | 없음(오버레이 패치) |
| 환경 분리 | values 파일·OCI/Repo 배포 | overlay 디렉터리 |
| 릴리스 관리 | history/rollback | 없음(kubectl 히스토리 의존) |
| 테스트/훅 | helm test, hooks | 없음(별도 스크립트) |
| 학습 난도 | 중간(템플릿/Sprig) | 낮음(패치 개념) |
| 적합 사례 | 복잡한 앱/오픈소스 패키징 | 단순 변형/클러스터 단일 팀 |

실무에서는 **Helm 차트**를 **Argo CD/Flux**와 함께 사용하거나, **Helm 차트를 베이스로 Kustomize 오버레이**를 덧대는 하이브리드도 활용한다.

---

## GitOps/CI-CD 파이프라인 예시

### GitHub Actions(OCI로 푸시·배포)

```yaml
name: helm-release
on: { push: { branches: [ main ], paths: [ "charts/mychart/**" ] } }
jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Setup Helm
        uses: azure/setup-helm@v4
      - name: Helm Lint
        run: helm lint charts/mychart
      - name: Package
        run: helm package charts/mychart -d dist
      - name: Push to OCI
        env:
          HELM_EXPERIMENTAL_OCI: "1"
          CR_PAT: ${{ secrets.GHCR_TOKEN }}
        run: |
          echo $CR_PAT | helm registry login ghcr.io -u ${{ github.actor }} --password-stdin
          helm push dist/mychart-*.tgz oci://ghcr.io/acme/helm
```

Argo CD에서는 `source: helm`을 지정하고 `valueFiles`/`parameters`로 환경 값을 주입한다.

---

## 운영 팁과 안티패턴

- **안정성 옵션**: `--atomic --wait --timeout` 기본 사용.
- **차트 품질**: `helm lint` + `values.schema.json` + `helm test`.
- **드리프트 방지**: 운영중 수동 `kubectl edit/apply` 지양(릴리스와 드리프트).
- **조건 깔끔화**: `.Values.xxx.enabled` 패턴으로 선택적 리소스 생명주기 제어.
- **CRD 관리**: 차트와 분리하거나 CRD 전용 차트로 관리(업그레이드 안전성).
- **tpl/lookup 남용 주의**: 선언형/재현성을 해치지 않도록 최소화.
- **버전 정책**: 차트 `version`(semver)와 앱 `appVersion`를 분리 관리.
- **Diff/Smoke Test**: 승격 전에 스테이징 diff와 `helm test` 자동화.

---

## 확장 예제: ConfigMap/Secret, HPA, PodDisruptionBudget

### ConfigMap + Secret(템플릿)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "mychart.fullname" . }}-cfg
data:
  LOG_LEVEL: {{ (index .Values.env 0).value | default "info" | quote }}
---
apiVersion: v1
kind: Secret
metadata:
  name: {{ include "mychart.fullname" . }}-sec
type: Opaque
stringData:
  DB_USER: {{ .Values.db.user | quote }}
  DB_PASS: {{ .Values.db.pass | quote }}
```

### HPA(조건부)

```yaml
{{- if .Values.hpa.enabled }}
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {{ include "mychart.fullname" . }}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ include "mychart.fullname" . }}
  minReplicas: {{ .Values.hpa.min | default 2 }}
  maxReplicas: {{ .Values.hpa.max | default 5 }}
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: {{ .Values.hpa.cpu | default 70 }}
{{- end }}
```

### PodDisruptionBudget

```yaml
{{- if .Values.pdb.enabled }}
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: {{ include "mychart.fullname" . }}
spec:
  selector:
    matchLabels: {{- include "mychart.selectorLabels" . | nindent 6 }}
  minAvailable: {{ .Values.pdb.minAvailable | default "50%" }}
{{- end }}
```

---

## 마이그레이션 힌트(수동 YAML → Helm)

1. 현재 YAML을 `templates/`로 이동.
2. 반복/환경차이 난 부분을 변수화(values로 승격).
3. `_helpers.tpl`에 네이밍/라벨 공통화.
4. `values.schema.json`으로 타입·필수값 검증.
5. `helm template/lint/test`로 품질 게이트 도입.
6. GitOps로 승격 파이프라인 구축.

---

## 결론

Helm은 Kubernetes 배포를 **패키징·변수화·버전관리**로 끌어올려, **재현 가능한 설치/업그레이드/롤백**과 **환경별 구성 분리**를 실현한다. 스키마 검증, 훅/테스트, 의존성/OCI, 비밀관리 연계, GitOps 자동화까지 결합하면, 팀과 조직은 **일관성 있는 배포 파이프라인**과 **운영 신뢰도**를 확보할 수 있다. 단순 오버레이만 필요하면 Kustomize가 더 간단할 수도 있으나, 복잡한 앱/오픈소스 패키징/릴리스 관리가 요구된다면 Helm이 **사실상 표준 선택**이다.