---
layout: post
title: Kubernetes - Jenkins + Helm + Kubernetes 파이프라인 구성
date: 2025-05-31 21:20:23 +0900
category: Kubernetes
---
# Jenkins + Helm + Kubernetes 파이프라인 구성하기

핵심은 다음 네 축입니다.

1. **재현 가능한 이미지**: 태깅 전략·캐시 최적화·멀티스테이지 Dockerfile
2. **보안 내장**: 이미지 스캔(Trivy) + 서명(Cosign) + 정책 게이트(Conftest/OPA)
3. **신뢰성 배포**: Helm 차트 구조화(환경 오버레이, Helmfile 옵션), rollout 검증/자동 롤백
4. **운영 자동화**: 멀티브랜치, 동적 환경(Preview), Slack/Teams 알림, Helm diff, 재시도

---

## 아키텍처 개요(확장)

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

## 사전 준비 체크리스트

| 항목 | 선택/예시 | 비고 |
|---|---|---|
| Jenkins | LTS 또는 JCasC | JNLP/Inbound agent 또는 Kubernetes plugin 에이전트 |
| Docker | 빌더 노드(권한 분리) | Buildx/QEMU 포함 |
| Registry | Docker Hub/ECR/GCR/ACR | 리드 전용 토큰 분리 권장 |
| Helm | v3 이상 | `helm diff` 플러그인 추천 |
| Kube 접근 | kubeconfig 또는 클라우드 OIDC | 최소 권한 ServiceAccount + RBAC |
| 시크릿 | Jenkins Credentials | Registry, kubeconfig, Cosign key, Slack webhook 등 |
| 보안 스캔 | Trivy(권장) | 임계치에 따른 fail 게이트 |

---

## Jenkins 필수 플러그인

- **Docker Pipeline**: `docker.build`, `withDockerRegistry`
- **Kubernetes CLI** 또는 `sh`로 직접 설치
- **Credentials Binding**: 시크릿/파일 주입
- **Git** / **Pipeline: Multibranch** / **Blue Ocean**(선택)
- **AnsiColor**(로그 가독성), **Slack Notification**(선택)
- **Pipeline Utility Steps**(YAML/JSON 파싱)

---

## 레포 구조(권장)

```
.
├─ Jenkinsfile
├─ Dockerfile
├─ helm-chart/
│  ├─ Chart.yaml
│  ├─ values.yaml
│  ├─ values-stg.yaml
│  ├─ values-prod.yaml
│  └─ templates/
│     ├─ deployment.yaml
│     ├─ service.yaml
│     ├─ hpa.yaml           # 선택
│     └─ ingress.yaml       # 선택
├─ ops/
│  ├─ policies/             # conftest/OPA 정책
│  ├─ scripts/              # smoke test, rollout check 등
│  └─ k8s-rbac/             # SA/Role/RoleBinding
└─ .jenkins/
   └─ jenkins.yaml          # JCasC(선택)
```

---

## Dockerfile 최적화(멀티스테이지 + 캐시)

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

- `npm ci` → 재현성
- 런타임 이미지 최소화, `USER`로 비루트 실행

---

## Helm 차트 핵심 템플릿

**`helm-chart/values.yaml`**

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

**`helm-chart/templates/deployment.yaml`**

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

환경별 값(`values-stg.yaml`, `values-prod.yaml`)로 **리소스·레플리카·도메인**만 오버라이드.

---

## 예시

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

| ID | 타입 | 예시 |
|---|---|---|
| `docker-hub-credentials` | Username/Password | 이미지 푸시 |
| `kubeconfig-id` | Secret file | kubeconfig 최소권한 |
| `COSIGN_PRIVATE_KEY` | Secret text/file | Cosign 서명 키 |
| `COSIGN_PASSWORD` | Secret text | 키 비밀번호 |
| `SLACK_WEBHOOK_URL` | Secret text | 알림 |

---

## Jenkinsfile(Declarative, 엔드투엔드)

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
    IMAGE_TAG              = "${env.BUILD_NUMBER}" // 또는 GIT_SHA
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
            # 간단 스모크 테스트
            kubectl -n default run curltest --rm -it --image=curlimages/curl --restart=Never -- \
              curl -fsS http://myapp.default.svc.cluster.local/healthz/ready
          """
        }
      }
    }
  }

  post {
    success {
      echo '✅ Deployment Success'
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
      echo '❌ Deployment Failed'
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

포인트:
- **Trivy 실패 시 빌드 중단** → 보안 게이트
- **Cosign 서명** → 이미지 무결성
- **Helm diff** → 적용 전 변화량 시각화
- **rollout status + 스모크** → 기능적 검증

---

## 멀티브랜치/환경 전략

- `develop` → **staging**: `values-stg.yaml` 사용
- `main` → **prod**: `values-prod.yaml` 사용

Jenkinsfile에서 **브랜치 조건**:

```groovy
def valuesFile = (env.BRANCH_NAME == 'main') ? 'values-prod.yaml' : 'values-stg.yaml'

sh """
  helm upgrade --install myapp ./helm-chart \
    -f helm-chart/${valuesFile} \
    --set image.repository=${IMAGE_NAME} \
    --set image.tag=${IMAGE_TAG} \
    --namespace ${(env.BRANCH_NAME == 'main') ? 'prod' : 'stg'} --create-namespace
"""
```

---

## Canary/Blue-Green(Helm 값으로 제어)

단순 **Canary**: 동일 차트 두 릴리스(`myapp`, `myapp-canary`) 운영 → Service 셀렉터/Ingress 경로/가중치로 트래픽 조절.

```bash
helm upgrade --install myapp-canary ./helm-chart \
  --set image.tag=${IMAGE_TAG} \
  --set replicaCount=1 \
  --set service.canary=true
```

Ingress 컨트롤러(NGINX, Istio) 기능으로 **가중치 전환**, 정상 시 **본선 릴리스** 업데이트 후 `canary` 제거.

---

## Helmfile(선택)로 다수 차트/환경 동시 관리

`helmfile.yaml`:

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

Jenkins 단계:

```groovy
sh """
  export IMAGE_TAG=${IMAGE_TAG}
  helmfile apply
"""
```

---

## 시크릿 관리(Sealed Secrets/SOPS 권장)

- **Sealed Secrets**: 암호화된 `SealedSecret`만 Git에 저장 → 클러스터에서 복호화
- **SOPS**: `*.enc.yaml`로 저장, 에이전트에서 복호화 후 `kubectl apply`

Jenkins에서 복호화 후 적용:

```bash
sops -d k8s/secret.enc.yaml | kubectl apply -f -
```

---

## 성과지표(개념적 모델)

배포 성공 확률 \(p_s\), 실패 확률 \(p_f = 1 - p_s\), 평균 복구시간 MTTR \(T_r\), 배포 빈도 \(f_d\)일 때,
운영 효용(개념) \(U\):

$$
U = f_d \cdot p_s - \alpha \cdot T_r \cdot (1 - p_s)
$$

- 보안 스캔/정책/서명 → \(p_s \uparrow\)
- 자동 롤백/롤아웃 검증 → \(T_r \downarrow\)
- 캐시/병렬화 → \(f_d \uparrow\)

---

## 트러블슈팅 빠른표

| 증상 | 진단 명령 | 흔한 원인 | 해결 |
|---|---|---|---|
| `ImagePullBackOff` | `kubectl describe pod` | 레지스트리 인증/태그 오타 | 자격증명, 태그 일치 점검 |
| `CrashLoopBackOff` | `kubectl logs --previous` | 프로브, ENV, 메모리 | `startupProbe` 튜닝, ENV/limits 재설정 |
| 롤아웃 정지 | `kubectl rollout status` | readiness 실패 | 헬스엔드포인트/DB 의존성 확인 |
| Helm 실패 | `helm get`/`helm diff` | 값 오버라이드 충돌 | values 병합/스키마 확인 |
| 권한 거부 | `kubectl auth can-i` | RBAC 부족 | Role/Binding 보완(최소권한) |

---

## 확장 팁

- **재시도**: 일시적 네트워크 문제는 `retry(n)` 블록으로 래핑
- **Helm 작동 건전성**: `--atomic` 옵션으로 실패 시 자동 롤백
- **서버사이드 Apply**: 선언형 충돌 최소화
  `kubectl apply --server-side -f <rendered>`
- **메트릭 기반 게이트**: Prometheus API로 슬로우 에러율·레이턴시 체크 후 승인/중단
- **프리뷰 환경**: PR 번호 기반 네임스페이스 동적 생성 → 머지 시 정리

---

## 최소 동작 예제(빠른 시도용)

### Jenkinsfile(라이트 버전)

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
    stage('Checkout'){ steps { checkout scm } }
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

## 운영 점검 체크리스트

- [ ] 레포·차트 구조: 환경 오버레이 분리
- [ ] Docker 빌드 캐시/멀티스테이지 적용
- [ ] 이미지 태그는 **SHA/빌드번호** 고정값
- [ ] Trivy 스캔, Cosign 서명 게이트
- [ ] RBAC 최소권한 SA 사용
- [ ] `startupProbe/readinessProbe` 정확히 설정
- [ ] `helm diff`로 변경사항 인지
- [ ] 롤아웃 검증 + 스모크테스트
- [ ] 실패 시 알림 및 즉시 롤백 경로 확보
- [ ] 시크릿은 Sealed Secrets/SOPS로 Git 관리

---

## 결론

- Jenkins는 **자유도가 높아** 엔터프라이즈 요구(보안/통제/온프레미스)에 적합합니다.
- Helm으로 **환경별 값을 선언형**으로 관리하고, Jenkins 파이프라인에서 **보안·신뢰성 게이트**(스캔/서명/정책/검증)를 통합하면,
  **반복 가능하고 안전한 배포 자동화**가 완성됩니다.
- 규모가 커지면, 빌드는 Jenkins, 최종 배포는 **GitOps(Argo CD)**로 분리하는 하이브리드도 유효합니다.

---

## 참고 자료

- Jenkins Pipeline 문서
- Helm 공식 문서
- Kubernetes RBAC 가이드
- Trivy / Cosign / Conftest
- Helm Diff 플러그인
- Sealed Secrets / SOPS

이 구성으로 시작해 **보안·관측·자동화**를 점진적으로 추가하면, 팀의 배포 신뢰도와 속도를 동시에 끌어올릴 수 있습니다.
