---
layout: post
title: Kubernetes - Jenkins + Helm + Kubernetes 파이프라인 구성
date: 2025-05-31 21:20:23 +0900
category: Kubernetes
---
# Jenkins + Helm + Kubernetes 파이프라인 구성하기

현대적인 Kubernetes 애플리케이션 배포를 위한 Jenkins 파이프라인 구축은 네 가지 핵심 축을 중심으로 설계되어야 합니다:

1. **재현 가능한 이미지 빌드**: 일관된 태깅 전략, 캐시 최적화, 멀티스테이지 Dockerfile
2. **보안 내재화**: 이미지 스캔(Trivy), 서명(Cosign), 정책 검증(Conftest/OPA)
3. **신뢰성 있는 배포**: 구조화된 Helm 차트, 환경별 오버레이, 롤아웃 검증 및 자동 롤백
4. **운영 자동화**: 멀티브랜치 지원, 동적 환경, 통합 알림, Helm diff, 재시도 메커니즘

---

## 아키텍처 개요

```mermaid
graph TD
A[개발자 Git push] --> B[Jenkins 멀티브랜치 파이프라인]
B --> C[Docker Buildx 캐시 활용 빌드]
C --> D[Trivy 이미지 스캔]
D --> E[Cosign 서명]
E --> F[(Container Registry)]
F --> G[Helm upgrade --install]
G --> H[Kubernetes Cluster]
H --> I[롤아웃 검증(readiness/metric)]
I --> J[알림/자동 롤백/체인지로그]
```

---

## 사전 준비 요구사항

| 구성 요소 | 선택/예시 | 비고 |
|---|---|---|
| Jenkins | LTS 또는 JCasC(Configuration as Code) | Kubernetes 플러그인 또는 JNLP 에이전트 |
| Docker | 빌드 노드(권한 분리 권장) | Buildx 및 QEMU 포함 |
| 컨테이너 레지스트리 | Docker Hub/ECR/GCR/ACR | 읽기 전용 토큰 분리 권장 |
| Helm | v3 이상 | `helm diff` 플러그인 추천 |
| Kubernetes 접근 | kubeconfig 또는 클라우드 OIDC | 최소 권한 ServiceAccount + RBAC |
| 비밀 정보 관리 | Jenkins Credentials | 레지스트리, kubeconfig, Cosign 키, Slack 웹훅 등 |
| 보안 스캔 도구 | Trivy | 임계값 기반 실패 게이트 |

---

## Jenkins 필수 플러그인

- **Docker Pipeline**: `docker.build`, `withDockerRegistry` 지원
- **Kubernetes CLI**: 또는 셸 스크립트를 통한 직접 설치
- **Credentials Binding**: 비밀 정보 및 파일 주입
- **Git** / **Pipeline: Multibranch** / **Blue Ocean**(선택사항)
- **AnsiColor**: 로그 가독성 향상
- **Slack Notification**: 알림 통합
- **Pipeline Utility Steps**: YAML/JSON 파싱 유틸리티

---

## 프로젝트 구조 권장안

```
.
├─ Jenkinsfile
├─ Dockerfile
├─ helm-chart/
│  ├─ Chart.yaml
│  ├─ values.yaml           # 기본값
│  ├─ values-stg.yaml       # 스테이징 환경
│  ├─ values-prod.yaml      # 프로덕션 환경
│  └─ templates/
│     ├─ deployment.yaml
│     ├─ service.yaml
│     ├─ hpa.yaml           # 수평 Pod 오토스케일러
│     └─ ingress.yaml
├─ ops/
│  ├─ policies/             # Conftest/OPA 정책 파일
│  ├─ scripts/              # 스모크 테스트, 롤아웃 검증
│  └─ k8s-rbac/             # ServiceAccount, Role, RoleBinding
└─ .jenkins/
   └─ jenkins.yaml          # JCasC 구성(선택사항)
```

---

## Dockerfile 최적화: 멀티스테이지 빌드와 캐시 활용

```dockerfile
# syntax=docker/dockerfile:1.7

FROM --platform=$BUILDPLATFORM node:20-alpine AS deps
WORKDIR /src
COPY package*.json ./
RUN npm ci

FROM deps AS build
COPY . .
RUN npm run build

FROM node:20-alpine AS runtime
ENV NODE_ENV=production
WORKDIR /app
COPY --from=deps /src/node_modules /app/node_modules
COPY --from=build /src/dist /app/dist
USER node
EXPOSE 3000
CMD ["node", "dist/main.js"]
```

**최적화 포인트:**
- `npm ci`를 통한 재현 가능한 의존성 설치
- 런타임 이미지 최소화 및 비루트 사용자 실행
- 빌드 캐시 레이어 최적화

---

## Helm 차트 템플릿 구성

### 기본 values.yaml

```yaml
image:
  repository: yourname/yourapp
  tag: latest
  pullPolicy: IfNotPresent

replicaCount: 2

resources:
  requests:
    cpu: 200m
    memory: 256Mi
  limits:
    cpu: "1"
    memory: 512Mi

service:
  type: ClusterIP
  port: 80

containerPort: 3000

probes:
  enabled: true
  readinessPath: /healthz/ready
  livenessPath: /healthz/live

nodeSelector: {}
tolerations: []
affinity: {}
```

### Deployment 템플릿

{% raw %}
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "chart.name" . }}
  labels:
    app: {{ include "chart.name" . }}
spec:
  replicas: {{ .Values.replicaCount }}
  revisionHistoryLimit: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: {{ include "chart.name" . }}
  template:
    metadata:
      labels:
        app: {{ include "chart.name" . }}
    spec:
      serviceAccountName: {{ include "chart.serviceAccountName" . | default "cd-bot" }}
      containers:
      - name: {{ include "chart.name" . }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        ports:
        - containerPort: {{ .Values.containerPort }}
        resources:
{{- toYaml .Values.resources | nindent 10 }}
{{- if .Values.probes.enabled }}
        startupProbe:
          httpGet:
            path: /healthz/startup
            port: {{ .Values.containerPort }}
          failureThreshold: 60
          periodSeconds: 2
        readinessProbe:
          httpGet:
            path: {{ .Values.probes.readinessPath }}
            port: {{ .Values.containerPort }}
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: {{ .Values.probes.livenessPath }}
            port: {{ .Values.containerPort }}
          periodSeconds: 10
{{- end }}
```
{% endraw %}

환경별 값 파일(`values-stg.yaml`, `values-prod.yaml`)을 통해 리소스, 레플리카 수, 도메인 설정 등을 오버라이드할 수 있습니다.

---

## Kubernetes RBAC 구성 예제

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cd-bot
  namespace: default
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: cd-deployer
  namespace: default
rules:
- apiGroups: ["", "apps", "batch", "extensions"]
  resources: ["deployments","services","configmaps","secrets","pods"]
  verbs: ["get","list","watch","create","update","patch","delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: cd-deployer-binding
  namespace: default
subjects:
- kind: ServiceAccount
  name: cd-bot
  namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: cd-deployer
```

---

## Jenkins Credentials 설계

| Credential ID | 타입 | 용도 |
|---|---|---|
| `docker-hub-credentials` | Username/Password | 컨테이너 이미지 푸시 |
| `kubeconfig-id` | Secret file | Kubernetes 클러스터 접근 |
| `COSIGN_PRIVATE_KEY` | Secret text/file | Cosign 이미지 서명 키 |
| `COSIGN_PASSWORD` | Secret text | Cosign 키 비밀번호 |
| `SLACK_WEBHOOK_URL` | Secret text | Slack 알림 웹훅 |

---

## 종합 Jenkinsfile 예제 (Declarative Pipeline)

```groovy
pipeline {
  agent any

  options {
    ansiColor('xterm')
    timestamps()
    buildDiscarder(logRotator(numToKeepStr: '50'))
    timeout(time: 30, unit: 'MINUTES')
  }

  environment {
    REGISTRY_CREDENTIALS   = 'docker-hub-credentials'
    KUBECONFIG_CREDENTIALS = 'kubeconfig-id'
    IMAGE_NAME             = 'yourname/yourapp'
    GIT_SHA                = "${env.GIT_COMMIT ?: env.BUILD_NUMBER}"
    IMAGE_TAG              = "${env.BUILD_NUMBER}"
    COSIGN_PRIV            = credentials('COSIGN_PRIVATE_KEY')
    COSIGN_PASS            = credentials('COSIGN_PASSWORD')
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
        sh 'git rev-parse --short HEAD'
      }
    }

    stage('Docker Build & Push') {
      steps {
        script {
          docker.withRegistry('', REGISTRY_CREDENTIALS) {
            sh """
              docker buildx create --use || true
              docker buildx build \
                --platform linux/amd64 \
                --tag ${IMAGE_NAME}:${IMAGE_TAG} \
                --tag ${IMAGE_NAME}:latest \
                --push .
            """
          }
        }
      }
    }

    stage('Security Scan (Trivy)') {
      steps {
        sh """
          docker run --rm aquasec/trivy:latest \
            image --exit-code 1 --ignore-unfixed \
            ${IMAGE_NAME}:${IMAGE_TAG} || { echo 'Trivy failed'; exit 1; }
        """
      }
    }

    stage('Sign Image (Cosign)') {
      steps {
        sh """
          echo "${COSIGN_PRIV}" > cosign.key
          COSIGN_PASSWORD='${COSIGN_PASS}' cosign sign --key cosign.key ${IMAGE_NAME}:${IMAGE_TAG}
          rm -f cosign.key
        """
      }
    }

    stage('Helm Diff (Preview)') {
      steps {
        withCredentials([file(credentialsId: "${KUBECONFIG_CREDENTIALS}", variable: 'KCFG')]) {
          sh """
            export KUBECONFIG=$KCFG
            helm repo add stable https://charts.helm.sh/stable || true
            helm plugin install https://github.com/databus23/helm-diff || true
            helm diff upgrade myapp ./helm-chart \
              --namespace default --allow-unreleased \
              --set image.repository=${IMAGE_NAME} \
              --set image.tag=${IMAGE_TAG} || true
          """
        }
      }
    }

    stage('Deploy with Helm') {
      steps {
        withCredentials([file(credentialsId: "${KUBECONFIG_CREDENTIALS}", variable: 'KCFG')]) {
          sh """
            export KUBECONFIG=$KCFG
            helm upgrade --install myapp ./helm-chart \
              --namespace default --create-namespace \
              --set image.repository=${IMAGE_NAME} \
              --set image.tag=${IMAGE_TAG}
          """
        }
      }
    }

    stage('Rollout Verification') {
      steps {
        withCredentials([file(credentialsId: "${KUBECONFIG_CREDENTIALS}", variable: 'KCFG')]) {
          sh """
            export KUBECONFIG=$KCFG
            kubectl rollout status deploy/myapp -n default --timeout=240s
            kubectl -n default run curltest --rm -it --image=curlimages/curl --restart=Never -- \
              curl -fsS http://myapp.default.svc.cluster.local/healthz/ready
          """
        }
      }
    }
  }

  post {
    success {
      echo 'Deployment Success'
      script {
        if (env.SLACK_WEBHOOK_URL) {
          sh """
            curl -X POST -H 'Content-type: application/json' \
            --data '{"text":"Deployment succeeded: ${IMAGE_NAME}:${IMAGE_TAG}"}' \
            '${SLACK_WEBHOOK_URL}'
          """
        }
      }
    }
    failure {
      echo 'Deployment Failed'
      script {
        if (env.SLACK_WEBHOOK_URL) {
          sh """
            curl -X POST -H 'Content-type: application/json' \
            --data '{"text":"Deployment failed: ${IMAGE_NAME}:${IMAGE_TAG}"}' \
            '${SLACK_WEBHOOK_URL}'
          """
        }
      }
    }
    always {
      echo 'Pipeline Finished.'
    }
  }
}
```

**파이프라인 핵심 요소:**
- **Trivy 스캔**: 보안 취약점 감지 및 임계값 초과 시 실패 처리
- **Cosign 서명**: 컨테이너 이미지 무결성 보장
- **Helm diff**: 배포 전 변경사항 시각화
- **롤아웃 검증 및 스모크 테스트**: 기능적 정상 동작 확인

---

## 멀티브랜치 및 다중 환경 전략

브랜치 기반 배포 전략을 구현할 수 있습니다:
- `develop` 브랜치 → **스테이징(staging)** 환경
- `main` 브랜치 → **프로덕션(production)** 환경

**Jenkinsfile에서 환경 구분:**

```groovy
def valuesFile = (env.BRANCH_NAME == 'main') ? 'values-prod.yaml' : 'values-stg.yaml'
def namespace = (env.BRANCH_NAME == 'main') ? 'prod' : 'stg'

sh """
  helm upgrade --install myapp ./helm-chart \
    -f helm-chart/${valuesFile} \
    --set image.repository=${IMAGE_NAME} \
    --set image.tag=${IMAGE_TAG} \
    --namespace ${namespace} --create-namespace
"""
```

---

## 카나리 및 블루-그린 배포 전략

Helm을 활용한 간단한 카나리 배포 구현:

```bash
# 카나리 릴리스 배포
helm upgrade --install myapp-canary ./helm-chart \
  --set image.tag=${IMAGE_TAG} \
  --set replicaCount=1 \
  --set service.canary=true

# Ingress 컨트롤러를 통한 트래픽 가중치 조정
# 정상 동작 확인 후 본선 릴리스 업데이트
helm upgrade --install myapp ./helm-chart \
  --set image.tag=${IMAGE_TAG}

# 카나리 릴리스 정리
helm uninstall myapp-canary
```

NGINX 또는 Istio와 같은 Ingress 컨트롤러의 가중치 기반 라우팅 기능을 활용하여 트래픽을 점진적으로 전환할 수 있습니다.

---

## Helmfile을 통한 다중 차트 관리

대규모 환경에서는 Helmfile을 사용하여 여러 Helm 차트를 선언적으로 관리할 수 있습니다:

**`helmfile.yaml` 예시:**

{% raw %}
```yaml
releases:
- name: myapp
  namespace: prod
  chart: ./helm-chart
  values:
  - values-prod.yaml
  set:
  - name: image.repository
    value: yourname/yourapp
  - name: image.tag
    value: {{ requiredEnv "IMAGE_TAG" }}
```
{% endraw %}

**Jenkins 단계에서 실행:**

```groovy
sh """
  export IMAGE_TAG=${IMAGE_TAG}
  helmfile apply
"""
```

---

## 비밀 정보 관리: SealedSecrets 및 SOPS

Git 저장소에는 평문 비밀 정보를 저장해서는 안 됩니다. 다음과 같은 접근 방식을 권장합니다:

### SealedSecrets
암호화된 `SealedSecret` 리소스만 Git에 저장하고, 클러스터 내 컨트롤러가 복호화합니다.

### SOPS
암호화된 `*.enc.yaml` 파일로 저장하고, Jenkins 에이전트에서 복호화 후 적용:

```bash
sops -d k8s/secret.enc.yaml | kubectl apply -f -
```

---

## 성과 지표 모니터링

배포 프로세스의 효율성과 안정성을 측정하기 위해 다음과 같은 지표를 고려할 수 있습니다:

- **배포 성공률 (\(p_s\))**: 성공적인 배포 비율
- **평균 복구 시간(MTTR, \(T_r\))**: 실패 시 복구에 소요된 평균 시간
- **배포 빈도 (\(f_d\))**: 단위 시간당 배포 횟수

운영 효용을 개념적으로 표현하면:

```
운영 효용 = (배포 빈도 × 성공률) - (가중치 × 평균 복구 시간 × 실패률)
```

보안 스캔, 정책 검증, 서명과 같은 보안 게이트는 성공률을 높이고, 자동 롤백 및 롤아웃 검증은 평균 복구 시간을 줄입니다. 빌드 캐시 및 병렬화는 배포 빈도를 높입니다.

---

## 일반적인 문제 해결 가이드

| 증상 | 진단 명령 | 일반적인 원인 | 해결 방안 |
|---|---|---|---|
| `ImagePullBackOff` | `kubectl describe pod` | 레지스트리 인증 실패 또는 태그 오류 | 자격증명 및 태그 일치 확인 |
| `CrashLoopBackOff` | `kubectl logs --previous` | 프로브 실패, 환경변수 오류, 메모리 부족 | `startupProbe` 튜닝, 환경변수 및 리소스 제한 확인 |
| 롤아웃 중단 | `kubectl rollout status` | readiness 프로브 지속적 실패 | 헬스 엔드포인트 및 외부 의존성 확인 |
| Helm 배포 실패 | `helm get`, `helm diff` | 값 오버라이드 충돌 | values 파일 병합 및 스키마 확인 |
| 권한 거부 | `kubectl auth can-i` | RBAC 권한 부족 | Role 및 RoleBinding 보완 |

---

## 고급 운영 팁

### 재시도 메커니즘
일시적인 네트워크 문제에 대비하여 중요한 단계에 재시도 로직을 추가:

```groovy
retry(3) {
  sh 'helm upgrade --install ...'
}
```

### Helm 원자적 배포
`--atomic` 플래그를 사용하여 실패 시 자동 롤백:

```bash
helm upgrade --install --atomic myapp ./helm-chart
```

### 서버사이드 적용
선언형 충돌 최소화를 위해 서버사이드 적용 사용:

```bash
kubectl apply --server-side -f <렌더링된-매니페스트>
```

### 메트릭 기반 게이트
Prometheus API를 활용하여 오류율 및 지연 시간을 모니터링하고, 임계값 초과 시 배포 중단:

```bash
# 배포 전 메트릭 확인
if curl -s http://prometheus/api/v1/query?query=error_rate | grep -q "value.*[0-9]\\.[0-9]{2}"; then
  echo "오류율 임계값 초과, 배포 중단"
  exit 1
fi
```

### 프리뷰 환경
Pull Request 기반 동적 네임스페이스 생성 및 정리:

```groovy
def previewNamespace = "pr-${env.CHANGE_ID}"
sh """
  kubectl create namespace ${previewNamespace}
  helm upgrade --install myapp ./helm-chart --namespace ${previewNamespace}
  # 머지 후 정리
  kubectl delete namespace ${previewNamespace}
"""
```

---

## 최소 구성 예제 (신속한 시작용)

### 간소화된 Jenkinsfile

```groovy
pipeline {
  agent any
  environment {
    REGISTRY_CREDENTIALS = 'docker-hub-credentials'
    KUBECONFIG_CREDENTIALS = 'kubeconfig-id'
    IMAGE_NAME = 'yourname/yourapp'
    TAG = "${env.BUILD_NUMBER}"
  }
  stages {
    stage('Checkout'){ 
      steps { checkout scm } 
    }
    stage('Build & Push'){
      steps {
        script {
          docker.withRegistry('', REGISTRY_CREDENTIALS) {
            sh """
              docker build -t ${IMAGE_NAME}:${TAG} -t ${IMAGE_NAME}:latest .
              docker push ${IMAGE_NAME}:${TAG}
              docker push ${IMAGE_NAME}:latest
            """
          }
        }
      }
    }
    stage('Deploy'){
      steps {
        withCredentials([file(credentialsId: "${KUBECONFIG_CREDENTIALS}", variable: 'KCFG')]) {
          sh """
            export KUBECONFIG=$KCFG
            helm upgrade --install myapp ./helm-chart \
              --namespace default --create-namespace \
              --set image.repository=${IMAGE_NAME} \
              --set image.tag=${TAG}
            kubectl rollout status deploy/myapp -n default --timeout=180s
          """
        }
      }
    }
  }
}
```

---

## 결론

Jenkins, Helm, Kubernetes를 결합한 파이프라인 구축은 기업의 보안, 통제, 온프레미스 요구사항에 적합한 강력한 배포 자동화 솔루션을 제공합니다. 효과적인 파이프라인 설계를 위해 다음과 같은 원칙을 준수해야 합니다:

1. **구조적 표준화**: 환경별 오버레이를 분리한 일관된 레포지토리 구조 채택
2. **보안 내재화**: Trivy 스캔, Cosign 서명, RBAC 최소 권한 원칙 적용
3. **신뢰성 보장**: startupProbe/readinessProbe 정확 설정, Helm diff 검증, 롤아웃 검증 구현
4. **운영 자동화**: 멀티브랜치 지원, 알림 통합, 실패 시 자동 롤백 경로 마련
5. **비밀 정보 안전 관리**: SealedSecrets 또는 SOPS를 통한 Git 기반 안전한 비밀 정보 관리

규모가 확장됨에 따라 빌드 파이프라인은 Jenkins로 관리하고, 최종 배포는 GitOps(Argo CD)로 분리하는 하이브리드 접근 방식도 고려할 수 있습니다. 이러한 구성으로 시작하여 보안, 관측성, 자동화를 점진적으로 강화하면 팀의 배포 신뢰성과 속도를 동시에 향상시킬 수 있습니다.
