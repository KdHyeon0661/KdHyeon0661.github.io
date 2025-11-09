---
layout: post
title: Kubernetes - GitHub Actions로 Kubernetes 자동 배포하기
date: 2025-05-31 19:20:23 +0900
category: Kubernetes
---
# GitHub Actions로 Kubernetes 자동 배포하기

## 0) 전체 그림

```mermaid
flowchart LR
  A[Developer push] --> B[GitHub Actions]
  B --> C[Build & Cache with Buildx]
  C --> D[Scan (Trivy)]
  D --> E[Sign (Cosign)]
  E --> F[Push to Registry]
  F --> G[Deploy (kubectl/Helm/Kustomize)]
  G --> H[Rollout verify + notify]
```

---

## 1) 트리거 & 브랜치 전략

- `main` → **프로덕션** 자동 배포
- `develop` → **스테이징** 자동 배포
- PR → **빌드/테스트만** 수행, 배포는 보류 또는 **환경 미리보기(preview)**

```yaml
name: CI/CD

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main, develop ]
```

---

## 2) 컨테이너 빌드: 태깅·캐시·멀티플랫폼

### 2.1 태깅 전략 (가시성 + 재현성)

- `latest`는 **사람용**; **머신/배포에는 Git SHA** 활용
- 환경별 태그: `myapp:prod-<sha>`, `myapp:stg-<sha>`

{% raw %}
```bash
IMAGE_REG=${{ secrets.DOCKER_USERNAME }}/myapp
TAG=${{ github.sha }}
# 결과: <registry>/myapp:<sha>
```
{% endraw %}

### 2.2 Buildx + 캐시 (속도·비용 최적화)

{% raw %}
```yaml
- name: Set up QEMU
  uses: docker/setup-qemu-action@v3

- name: Set up Buildx
  uses: docker/setup-buildx-action@v3

- name: Build & Push
  uses: docker/build-push-action@v6
  with:
    context: .
    platforms: linux/amd64
    push: true
    tags: |
      ${{ env.IMAGE_REG }}:${{ github.sha }}
      ${{ env.IMAGE_REG }}:latest
    cache-from: type=gha
    cache-to: type=gha,mode=max
```
{% endraw %}

---

## 3) 보안 내장: 이미지 스캔 + 서명 + 정책 게이트

### 3.1 Trivy로 취약점 스캔

{% raw %}
```yaml
- name: Scan image (Trivy)
  uses: aquasecurity/trivy-action@0.24.0
  with:
    image-ref: ${{ env.IMAGE_REG }}:${{ github.sha }}
    format: 'table'
    exit-code: '1'
    ignore-unfixed: true
```
{% endraw %}

> 실패 시 파이프라인 중단(기본 임계값을 팀 정책에 맞춰 조정)

### 3.2 Cosign으로 서명 & 검증

{% raw %}
```yaml
- name: Install Cosign
  uses: sigstore/cosign-installer@v3.7.0

- name: Sign image
  run: |
    cosign sign --key env://COSIGN_PRIVATE_KEY \
      $IMAGE_REG:${{ github.sha }}
  env:
    COSIGN_PRIVATE_KEY: ${{ secrets.COSIGN_PRIVATE_KEY }}
    COSIGN_PASSWORD: ${{ secrets.COSIGN_PASSWORD }}
```
{% endraw %}

배포 전 검증(클러스터 쪽 Admission/Policy 연계도 가능):

{% raw %}
```yaml
- name: Verify signature (pre-deploy)
  run: |
    cosign verify --key env://COSIGN_PUBLIC_KEY $IMAGE_REG:${{ github.sha }}
  env:
    COSIGN_PUBLIC_KEY: ${{ secrets.COSIGN_PUBLIC_KEY }}
```
{% endraw %}

### 3.3 정책 게이트(옵션)

- OPA Conftest로 **K8s 매니페스트 정책** 검사(예: `runAsNonRoot` 강제):
```yaml
- name: Conftest policy check
  uses: instrumenta/conftest-action@v0.3.0
  with:
    files: k8s/
```

---

## 4) K8s 인증: kubeconfig vs ServiceAccount(OIDC)

### 4.1 빠른 시작 (kubeconfig Secrets)

- `KUBECONFIG_DATA`에 base64로 저장 (최소권한 계정 사용)

```bash
cat ~/.kube/config | base64 -w0
```

워크플로우에서:

{% raw %}
```yaml
- name: Set kubeconfig
  run: |
    echo "${{ secrets.KUBECONFIG_DATA }}" | base64 -d > $HOME/.kube/config
```
{% endraw %}

### 4.2 권장(클라우드): **ServiceAccount + RBAC + OIDC**  
- GitHub Actions의 **OIDC 토큰**으로 클러스터에서 **짧은 수명의 자격** 발급  
- 구체 설정(EKS/IAM Role for Service Accounts, GKE Workload Identity, AKS federated credential)은 각 클라우드 가이드에 따릅니다.

> 핵심: **장기 kubeconfig 없이** 안전하게 배포 권한을 위임

---

## 5) RBAC 최소권한 예시

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
  resources: ["deployments","services","configmaps","secrets","jobs","pods"]
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

> 네임스페이스 분리/환경 단위로 역할을 좁히는 것이 안전합니다.

---

## 6) 배포 방식 3종: `kubectl` / Kustomize / Helm

### 6.1 kubectl (가장 단순)

{% raw %}
```yaml
- name: Install kubectl
  uses: azure/setup-kubectl@v4
  with:
    version: 'v1.30.2'

- name: Deploy (kubectl)
  run: |
    kubectl set image deployment/myapp myapp=${{ env.IMAGE_REG }}:${{ github.sha }} -n default
    kubectl apply -f k8s/                # 필요한 리소스 전체 적용
    kubectl rollout status deployment/myapp -n default --timeout=180s
```
{% endraw %}

### 6.2 Kustomize (환경 오버레이)

디렉토리 예시:
```
k8s/
  base/
    deployment.yaml
    service.yaml
    kustomization.yaml
  overlays/
    staging/kustomization.yaml
    prod/kustomization.yaml
```

워크플로우:
```yaml
- name: Install Kustomize
  run: |
    curl -sSfL https://github.com/kubernetes-sigs/kustomize/releases/download/kustomize%2Fv5.4.2/kustomize_v5.4.2_linux_amd64.tar.gz \
    | tar xz -C /usr/local/bin

- name: Build & Apply (staging)
  if: github.ref == 'refs/heads/develop'
  run: |
    kustomize build k8s/overlays/staging | kubectl apply -f -
    kubectl rollout status deploy/myapp -n staging
```

### 6.3 Helm (복잡한 차트/값 관리)

{% raw %}
```yaml
- name: Setup Helm
  uses: azure/setup-helm@v4

- name: Helm upgrade (prod)
  if: github.ref == 'refs/heads/main'
  run: |
    helm upgrade --install myapp ./helm-chart \
      --namespace prod --create-namespace \
      --set image.repository=${{ env.IMAGE_REG }} \
      --set image.tag=${{ github.sha }} \
      --set replicaCount=3
    kubectl rollout status deploy/myapp -n prod
```
{% endraw %}

---

## 7) **실전 워크플로우** (엔드투엔드)

`.github/workflows/deploy.yml`

{% raw %}
```yaml
name: Deploy to Kubernetes

on:
  push:
    branches: [ main, develop ]

env:
  IMAGE_REG: ${{ secrets.DOCKER_USERNAME }}/myapp

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write   # OIDC 사용 시 필요
    steps:
    - name: Checkout
      uses: actions/checkout@v4

    - name: Set short SHA
      run: echo "SHORT_SHA=${GITHUB_SHA::7}" >> $GITHUB_ENV

    - name: Docker login
      uses: docker/login-action@v3
      with:
        username: ${{ secrets.DOCKER_USERNAME }}
        password: ${{ secrets.DOCKER_PASSWORD }}

    - name: QEMU
      uses: docker/setup-qemu-action@v3

    - name: Buildx
      uses: docker/setup-buildx-action@v3

    - name: Build & Push
      uses: docker/build-push-action@v6
      with:
        context: .
        push: true
        platforms: linux/amd64
        tags: |
          ${{ env.IMAGE_REG }}:${{ github.sha }}
          ${{ env.IMAGE_REG }}:latest
        cache-from: type=gha
        cache-to: type=gha,mode=max

    - name: Trivy scan
      uses: aquasecurity/trivy-action@0.24.0
      with:
        image-ref: ${{ env.IMAGE_REG }}:${{ github.sha }}
        format: 'table'
        exit-code: '1'
        ignore-unfixed: true

    - name: Install Cosign
      uses: sigstore/cosign-installer@v3.7.0

    - name: Cosign sign
      run: |
        cosign sign --key env://COSIGN_PRIVATE_KEY \
          $IMAGE_REG:${{ github.sha }}
      env:
        IMAGE_REG: ${{ env.IMAGE_REG }}
        COSIGN_PRIVATE_KEY: ${{ secrets.COSIGN_PRIVATE_KEY }}
        COSIGN_PASSWORD: ${{ secrets.COSIGN_PASSWORD }}

    - name: Set kubeconfig (kubeconfig mode)
      if: ${{ secrets.KUBECONFIG_DATA != '' }}
      run: |
        mkdir -p $HOME/.kube
        echo "${{ secrets.KUBECONFIG_DATA }}" | base64 -d > $HOME/.kube/config

    - name: Install kubectl
      uses: azure/setup-kubectl@v4
      with:
        version: 'v1.30.2'

    - name: Verify signature (pre-deploy)
      run: |
        cosign verify --key env://COSIGN_PUBLIC_KEY \
          $IMAGE_REG:${{ github.sha }}
      env:
        IMAGE_REG: ${{ env.IMAGE_REG }}
        COSIGN_PUBLIC_KEY: ${{ secrets.COSIGN_PUBLIC_KEY }}

    - name: Render image tag (Kustomize)
      run: |
        sed -i "s#image: .*#image: ${IMAGE_REG}:${GITHUB_SHA}#g" k8s/base/deployment.yaml

    - name: Deploy (staging via Kustomize)
      if: github.ref == 'refs/heads/develop'
      run: |
        curl -sSfL https://github.com/kubernetes-sigs/kustomize/releases/download/kustomize%2Fv5.4.2/kustomize_v5.4.2_linux_amd64.tar.gz | tar xz -C /usr/local/bin
        kustomize build k8s/overlays/staging | kubectl apply -f -
        kubectl rollout status deploy/myapp -n staging --timeout=180s

    - name: Deploy (prod via Helm)
      if: github.ref == 'refs/heads/main'
      uses: azure/setup-helm@v4

    - name: Helm upgrade prod
      if: github.ref == 'refs/heads/main'
      run: |
        helm upgrade --install myapp ./helm-chart \
          --namespace prod --create-namespace \
          --set image.repository=${IMAGE_REG} \
          --set image.tag=${GITHUB_SHA} \
          --set replicaCount=4
        kubectl rollout status deploy/myapp -n prod --timeout=240s
```
{% endraw %}

> 필요 시 `argocd app sync` 같은 **GitOps 연동**으로 전환 가능합니다(아래 §12).

---

## 8) 배포 매니페스트 예시

### 8.1 Deployment (기본형)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: default
  labels: { app: myapp }
spec:
  replicas: 3
  revisionHistoryLimit: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels: { app: myapp }
  template:
    metadata:
      labels: { app: myapp }
    spec:
      serviceAccountName: cd-bot
      containers:
      - name: myapp
        image: your-docker-id/myapp:REPLACED_BY_CI
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 3000
        resources:
          requests: { cpu: "200m", memory: "256Mi" }
          limits:   { cpu: "1",    memory: "512Mi" }
        startupProbe:
          httpGet: { path: /healthz/startup, port: 3000 }
          periodSeconds: 2
          failureThreshold: 60
        livenessProbe:
          httpGet: { path: /healthz/live, port: 3000 }
          periodSeconds: 10
        readinessProbe:
          httpGet: { path: /healthz/ready, port: 3000 }
          periodSeconds: 5
        envFrom:
        - configMapRef: { name: myapp-config }
        - secretRef:    { name: myapp-secret }
```

> `startupProbe` 도입으로 **프로브 오탐에 의한 CrashLoopBackOff**를 줄입니다.

---

## 9) 시크릿 관리(강화): Sealed Secrets / SOPS

### 9.1 Sealed Secrets (클러스터에서만 복호화)

- 개발자는 **암호화된 SealedSecret**만 Git에 커밋
- 컨트롤러가 SealedSecret → Secret으로 자동 복호화

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: myapp-secret
  namespace: default
spec:
  encryptedData:
    DB_PASSWORD: AgAvZk...   # kubeseal로 암호화된 값
```

### 9.2 SOPS + age/GPG

- 레포에 `*.enc.yaml`로 저장, CI에서 복호화 후 `kubectl apply`
- Git 기록에 평문 비밀값이 남지 않음

---

## 10) 롤아웃 검증 & 알림

### 10.1 롤아웃 상태 실패 시 즉시 실패

```yaml
- name: Verify rollout
  run: |
    kubectl rollout status deploy/myapp -n prod --timeout=240s
```

### 10.2 슬랙/디스코드 알림

{% raw %}
```yaml
- name: Slack notify
  uses: slackapi/slack-github-action@v1.27.0
  with:
    payload: |
      {
        "text": "Deploy succeeded: ${{ github.repository }}@${{ github.ref_name }} (sha: ${{ github.sha }})"
      }
  env:
    SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```
{% endraw %}

---

## 11) 멀티 클러스터/환경 배포 (matrix)

{% raw %}
```yaml
strategy:
  matrix:
    include:
      - env: staging
        namespace: stg
        kubeconfig: ${{ secrets.KCFG_STG }}
      - env: prod
        namespace: prod
        kubeconfig: ${{ secrets.KCFG_PROD }}

steps:
- name: Set kubeconfig
  run: |
    echo "${{ matrix.kubeconfig }}" | base64 -d > $HOME/.kube/config

- name: Deploy each env
  run: |
    kustomize build k8s/overlays/${{ matrix.env }} | kubectl apply -n ${{ matrix.namespace }} -f -
```
{% endraw %}

---

## 12) GitHub Actions ↔ GitOps(Argo CD) 조합

- **CI는 이미지 빌드/스캔/서명**까지
- **CD는 Argo CD**가 **Git 상태를 기준으로** 자동 동기화

실무 패턴:
1) CI에서 `values-prod.yaml`(이미지 태그만) PR 생성  
2) 머지 → Argo CD가 감지하여 동기화  
3) 롤아웃·드리프트 복구·승인 게이트는 Argo 생태계(Argo Rollouts/Notifications)로

> 장점: **상태의 진실 = Git**, 수동 변경은 드리프트로 감지/복구

---

## 13) Blue-Green/Canary (헬스 체크 동반)

### 13.1 간단 Canary (두 Deployment 가중치 전환)

- `svc`는 `selector`로 트래픽 분기(수동 스텝)
- 또는 **Argo Rollouts**로 점진적 비율 전환 + 자동 중단

### 13.2 `kubectl`만으로 안전장치

- 배포 직후 **스모크 테스트 Job** 실행
- 실패 시 `kubectl rollout undo`

```yaml
- name: Smoke test
  run: |
    kubectl -n prod run smoke --rm -it --image=curlimages/curl -- \
      curl -fsS http://myapp.prod.svc.cluster.local/healthz/ready
```

---

## 14) 성능·비용 최적화

- Buildx 캐시 공유(GHA 캐시) → **빌드 시간 단축**
- 레이어 구조 개선: 의존 설치/앱 빌드 → 런타임 분리(멀티스테이지)
- `requests/limits` 실측 기반 설정(§리소스 가이드와 연계)
- `kubectl diff`로 **변경 최소화**(옵션)

```yaml
- name: Server-side apply (idempotent)
  run: |
    kubectl apply --server-side -f k8s/
```

---

## 15) 장애대응/트러블슈팅 체크리스트

| 증상 | 진단 | 해결 |
|---|---|---|
| `ImagePullBackOff` | 이미지 경로/권한 | 레지스트리 로그인/PR 베이스 태그 확인 |
| `CrashLoopBackOff` | `logs --previous`, probe | `startupProbe`/리소스/ENV 재점검 |
| 롤아웃 정지 | `rollout status`, 이벤트 | `kubectl describe deploy/<name>` |
| RBAC 거부 | `kubectl auth can-i ... --as=...` | Role/RoleBinding 수정 |
| 시크릿 반영 안 됨 | 매니페스트/네임스페이스 | 이름/키/네임스페이스 일치 여부 |

---

## 16) **완성형** 샘플 레포 구조

```
.
├─ .github/workflows/deploy.yml
├─ Dockerfile
├─ helm-chart/ (옵션)
├─ k8s/
│  ├─ base/
│  │  ├─ deployment.yaml
│  │  ├─ service.yaml
│  │  └─ kustomization.yaml
│  └─ overlays/
│     ├─ staging/kustomization.yaml
│     └─ prod/kustomization.yaml
├─ policies/ (Conftest/OPA)
└─ manifests-secrets/ (SealedSecrets/SOPS-enc)
```

---

## 17) 수학적(개념) 메트릭 목표식(옵션)

배포 실패율 \(p_f\), 평균 복구시간 MTTR \(T_r\), 배포 빈도 \(f_d\)일 때 안정성 효용 \(U\) (개념 모델):

$$
U = f_d \cdot (1 - p_f) - \alpha \cdot T_r \cdot p_f
$$

- \(f_d\) ↑, \(p_f\) ↓, \(T_r\) ↓가 목표  
- 자동 롤백/검증/알림이 \(p_f\), \(T_r\)를 낮춥니다.

---

## 18) 요약

- **태그는 SHA**, 캐시/스캔/서명으로 **신뢰 가능한 아티팩트**를 만든다.
- 배포는 **kubectl/Kustomize/Helm** 중 팀 숙련도와 복잡도에 맞춰 선택.
- 시크릿은 **Sealed Secrets/SOPS**로 Git에 안전하게 보관.
- **롤아웃 검증 + 알림 + 롤백**으로 MTTR을 최소화.
- 규모가 커지면 **GitOps(Argo CD)**로 상태 기반 운영을 권장.

---

## 부록: 최소 동작 예제(빠른 적용용)

### A) `.github/workflows/deploy-min.yml`

{% raw %}
```yaml
name: Deploy (min)

on:
  push: { branches: [ main ] }

env:
  IMAGE: ${{ secrets.DOCKER_USERNAME }}/myapp

jobs:
  cd:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - uses: docker/login-action@v3
      with:
        username: ${{ secrets.DOCKER_USERNAME }}
        password: ${{ secrets.DOCKER_PASSWORD }}
    - uses: docker/build-push-action@v6
      with:
        push: true
        tags: |
          ${{ env.IMAGE }}:${{ github.sha }}
          ${{ env.IMAGE }}:latest
    - uses: azure/setup-kubectl@v4
      with: { version: 'v1.30.2' }
    - name: Kubeconfig
      run: |
        mkdir -p $HOME/.kube
        echo "${{ secrets.KUBECONFIG_DATA }}" | base64 -d > $HOME/.kube/config
    - name: Deploy
      run: |
        kubectl set image deploy/myapp myapp=${{ env.IMAGE }}:${{ github.sha }} -n default
        kubectl rollout status deploy/myapp -n default --timeout=180s
```
{% endraw %}

### B) `k8s/base/deployment.yaml` (필수 값만)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: myapp, namespace: default }
spec:
  replicas: 2
  selector: { matchLabels: { app: myapp } }
  template:
    metadata: { labels: { app: myapp } }
    spec:
      containers:
      - name: myapp
        image: your-docker-id/myapp:latest
        ports:
        - containerPort: 3000
        readinessProbe:
          httpGet: { path: /healthz/ready, port: 3000 }
        livenessProbe:
          httpGet: { path: /healthz/live, port: 3000 }
```

---

## 참고 링크

- GitHub Actions 공식 문서  
- kubectl tool installer Action  
- Trivy / Cosign / Conftest  
- Sealed Secrets / SOPS  
- Argo CD / Argo Rollouts / Notifications

> 단계적으로 도입하세요. **최소 파이프라인** → **스캔/서명/정책/알림** → **GitOps 전환** 순으로 확장하면 **안정성과 속도**를 동시에 얻을 수 있습니다.