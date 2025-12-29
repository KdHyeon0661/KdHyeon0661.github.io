---
layout: post
title: Kubernetes - GitHub Actions로 Kubernetes 자동 배포하기
date: 2025-05-31 19:20:23 +0900
category: Kubernetes
---
# GitHub Actions를 활용한 Kubernetes 자동 배포 가이드

## 소개: 현대적 CI/CD 파이프라인 구축

GitHub Actions를 활용한 Kubernetes 자동 배포는 개발자가 코드를 푸시하는 것만으로 애플리케이션을 빌드, 테스트, 배포하는 완전한 CI/CD 파이프라인을 구축할 수 있는 강력한 방법입니다. 이 가이드는 GitHub Actions를 사용하여 안전하고 효율적이며 확장 가능한 Kubernetes 배포 파이프라인을 구축하는 종합적인 방법을 제공합니다.

전체적인 배포 프로세스를 다이어그램으로 표현하면 다음과 같습니다:

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

이 아키텍처는 코드 변경사항이 자동으로 컨테이너 이미지로 변환되고, 보안 검사를 거친 후 Kubernetes 클러스터에 배포되는 완전히 자동화된 워크플로우를 보여줍니다.

---

## 브랜치 전략과 트리거 설정

효과적인 배포 파이프라인은 명확한 브랜치 전략과 적절한 트리거 설정으로 시작합니다. 일반적으로 다음과 같은 접근법을 권장합니다:

- **main 브랜치**: 프로덕션 환경으로의 자동 배포
- **develop 브랜치**: 스테이징 환경으로의 자동 배포
- **풀 리퀘스트**: 빌드 및 테스트만 수행, 배포는 보류 또는 환경 미리보기(preview) 제공

GitHub Actions 워크플로우 트리거는 다음과 같이 설정할 수 있습니다:

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main, develop ]
```

이 설정은 특정 브랜치에 코드가 푸시되거나 풀 리퀘스트가 생성될 때 워크플로우가 자동으로 실행되도록 합니다.

---

## 컨테이너 이미지 빌드: 최적화와 효율성

### 이미지 태깅 전략
효과적인 이미지 태깅은 배포의 추적성과 재현성을 보장합니다. 다음과 같은 태깅 전략을 권장합니다:

- **Git SHA 태그**: 정확한 코드 버전 추적을 위해 사용 (예: `myapp:abc123def`)
- **환경별 태그**: 환경 식별을 위해 사용 (예: `myapp:prod-abc123`, `myapp:stg-abc123`)
- **latest 태그**: 개발자 편의를 위해 사용하지만, 자동 배포에는 Git SHA 태그 사용

### Docker Buildx를 활용한 고성능 빌드
Buildx는 멀티플랫폼 빌드와 캐싱을 지원하여 빌드 시간을 크게 단축할 수 있습니다:

```yaml
- name: Set up QEMU for multi-platform builds
  uses: docker/setup-qemu-action@v3

- name: Set up Docker Buildx
  uses: docker/setup-buildx-action@v3

- name: Build and push Docker image
  uses: docker/build-push-action@v6
  with:
    context: .
    platforms: linux/amd64,linux/arm64
    push: true
    tags: |
      ${{ env.REGISTRY }}/myapp:${{ github.sha }}
      ${{ env.REGISTRY }}/myapp:latest
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

이 구성은 GitHub Actions 캐시를 활용하여 이전 빌드 레이어를 재사용하고, 멀티 아키텍처 이미지를 생성하며, 이미지를 컨테이너 레지스트리로 푸시합니다.

---

## 보안 통합: 이미지 스캔과 서명

### Trivy를 활용한 취약점 스캐닝
보안은 CI/CD 파이프라인의 핵심 요소입니다. Trivy를 사용하여 컨테이너 이미지의 취약점을 스캔할 수 있습니다:

```yaml
- name: Scan image for vulnerabilities
  uses: aquasecurity/trivy-action@0.24.0
  with:
    image-ref: ${{ env.REGISTRY }}/myapp:${{ github.sha }}
    format: 'sarif'
    output: 'trivy-results.sarif'
    exit-code: '1'
    ignore-unfixed: true
    severity: 'CRITICAL,HIGH'
```

이 단계는 심각도가 높은 수정되지 않은 취약점이 발견되면 파이프라인을 실패시킵니다. SARIF 형식의 결과는 GitHub의 보안 탭에서 시각화할 수 있습니다.

### Cosign을 활용한 이미지 서명
이미지 서명은 이미지의 무결성과 출처를 보장합니다:

```yaml
- name: Install Cosign
  uses: sigstore/cosign-installer@v3.7.0

- name: Sign container image
  run: |
    cosign sign \
      --key env://COSIGN_PRIVATE_KEY \
      ${{ env.REGISTRY }}/myapp:${{ github.sha }}
  env:
    COSIGN_PRIVATE_KEY: ${{ secrets.COSIGN_PRIVATE_KEY }}
    COSIGN_PASSWORD: ${{ secrets.COSIGN_PASSWORD }}
```

서명된 이미지는 배포 전에 검증되어 변조되지 않았음을 확인할 수 있습니다.

### 정책 검증
Kubernetes 매니페스트에 대한 정책 검증을 통해 보안과 규정 준수 요구사항을 강제할 수 있습니다:

```yaml
- name: Validate Kubernetes manifests with Conftest
  uses: instrumenta/conftest-action@v0.3.0
  with:
    files: k8s/
    policy: policies/
```

이 단계는 OPA(Open Policy Agent) 정책을 사용하여 Kubernetes 매니페스트가 보안 모범 사례를 준수하는지 확인합니다.

---

## Kubernetes 인증: 안전한 접근 방식

### OIDC를 활용한 최신 인증 방식
장기적인 kubeconfig 파일 대신 OIDC(OpenID Connect)를 사용하여 더 안전한 인증 방식을 구현할 수 있습니다. 주요 클라우드 제공업체별 설정:

- **AWS EKS**: IAM Roles for Service Accounts(IRSA) 구성
- **Google GKE**: Workload Identity 구성
- **Azure AKS**: Federated credentials 구성

GitHub Actions 워크플로우에 OIDC 구성:

```yaml
permissions:
  id-token: write   # OIDC 토큰 요청을 위한 권한
  contents: read    # 코드 체크아웃을 위한 권한
```

### 최소 권한 RBAC 구성
배포 서비스 계정에 필요한 최소 권한만 부여하는 것이 중요합니다:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deployment-manager
  namespace: production
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
```

이 역할은 배포 관리만을 위한 최소한의 권한을 제공하며, 시크릿 생성이나 네임스페이스 삭제와 같은 위험한 작업은 허용하지 않습니다.

---

## 배포 전략 선택: kubectl, Kustomize, Helm

### kubectl: 단순성과 직접성
가장 간단한 접근 방식으로, 작은 규모의 프로젝트나 빠른 시작에 적합합니다:

```yaml
- name: Deploy using kubectl
  run: |
    # 환경별 설정 적용
    envsubst < k8s/deployment.yaml | kubectl apply -f -
    
    # 롤아웃 상태 확인
    kubectl rollout status deployment/myapp \
      --namespace production \
      --timeout=300s
```

### Kustomize: 환경별 구성 관리
환경별 구성 변형이 필요한 경우 Kustomize가 이상적인 선택입니다:

```
k8s/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
└── overlays/
    ├── staging/
    │   └── kustomization.yaml
    └── production/
        └── kustomization.yaml
```

워크플로우에서의 사용:
```yaml
- name: Deploy to staging using Kustomize
  if: github.ref == 'refs/heads/develop'
  run: |
    kustomize build k8s/overlays/staging | kubectl apply -f -
```

### Helm: 복잡한 애플리케이션 배포
다양한 구성 옵션과 의존성을 가진 복잡한 애플리케이션에는 Helm이 적합합니다:

```yaml
- name: Deploy using Helm
  run: |
    helm upgrade --install myapp ./charts/myapp \
      --namespace production \
      --create-namespace \
      --values values/production.yaml \
      --set image.tag=${{ github.sha }} \
      --atomic \
      --wait
```

`--atomic` 플래그는 배포 실패 시 자동 롤백을 보장하며, `--wait` 플래그는 모든 리소스가 준비될 때까지 대기합니다.

---

## 종합적인 배포 워크플로우 예제

다음은 GitHub Actions를 사용한 완전한 Kubernetes 배포 워크플로우의 예제입니다:

```yaml
name: Production Deployment

on:
  push:
    branches: [ main ]

env:
  REGISTRY: ghcr.io/${{ github.repository }}
  IMAGE_NAME: myapp

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
      id-token: write
    
    steps:
    - name: Checkout repository
      uses: actions/checkout@v4
      
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3
      
    - name: Log in to container registry
      uses: docker/login-action@v3
      with:
        registry: ghcr.io
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}
    
    - name: Extract metadata for Docker
      id: meta
      uses: docker/metadata-action@v5
      with:
        images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
        tags: |
          type=sha,prefix=
          type=ref,event=branch
    
    - name: Build and push Docker image
      uses: docker/build-push-action@v6
      with:
        context: .
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
        cache-from: type=gha
        cache-to: type=gha,mode=max
    
    - name: Security scan
      uses: aquasecurity/trivy-action@0.24.0
      with:
        image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
        format: 'sarif'
        output: 'trivy-results.sarif'
    
    - name: Configure Kubernetes
      uses: azure/setup-kubectl@v4
      with:
        version: 'latest'
    
    - name: Authenticate to Kubernetes cluster
      uses: azure/k8s-set-context@v3
      with:
        method: service-account
        k8s-url: ${{ secrets.K8S_URL }}
        k8s-secret: ${{ secrets.K8S_SECRET }}
    
    - name: Deploy to Kubernetes
      run: |
        # 환경 변수로 이미지 태그 전달
        export IMAGE_TAG=${{ github.sha }}
        
        # Kustomize로 배포
        kustomize build k8s/overlays/production \
          | envsubst \
          | kubectl apply -f -
        
        # 롤아웃 상태 확인
        kubectl rollout status deployment/myapp \
          --namespace production \
          --timeout=300s
    
    - name: Post-deployment verification
      run: |
        # 스모크 테스트 실행
        kubectl run smoke-test \
          --namespace production \
          --rm \
          -i \
          --restart=Never \
          --image=curlimages/curl \
          -- \
          curl -f http://myapp.production.svc.cluster.local/health
        
        # 메트릭 확인
        kubectl top pods --namespace production
    
    - name: Notify deployment status
      if: always()
      uses: rtCamp/action-slack-notify@v2
      env:
        SLACK_WEBHOOK: ${{ secrets.SLACK_WEBHOOK_URL }}
        SLACK_COLOR: ${{ job.status == 'success' && 'good' || 'danger' }}
        SLACK_TITLE: "Deployment ${{ job.status }}"
        SLACK_MESSAGE: "Deployment of ${{ github.repository }} completed with status: ${{ job.status }}"
```

이 워크플로우는 코드 체크아웃부터 배포 후 검증까지의 완전한 프로세스를 포함하며, 각 단계가 실패할 경우 전체 파이프라인이 중단됩니다.

---

## 보안 비밀 관리

### SealedSecrets를 활용한 안전한 비밀 관리
SealedSecrets는 Kubernetes 시크릿을 안전하게 Git에 저장할 수 있게 해줍니다:

1. **암호화된 비밀 생성**:
```bash
kubectl create secret generic my-secret \
  --from-literal=password=supersecret \
  --dry-run=client \
  -o yaml \
  | kubeseal --format yaml > my-sealed-secret.yaml
```

2. **Git에 암호화된 파일 저장**
3. **Kubernetes에 배포**:
```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: my-secret
spec:
  encryptedData:
    password: AgBy3i4OJSWK+PiTySYZZA9rO43cGDEq...
```

Sealed Secrets 컨트롤러가 이 파일을 자동으로 복호화하여 일반 Kubernetes Secret으로 변환합니다.

### SOPS를 활용한 파일 기반 비밀 관리
SOPS는 YAML, JSON, ENV 파일을 암호화할 수 있는 유연한 도구입니다:

```yaml
- name: Decrypt secrets with SOPS
  run: |
    sops --decrypt secrets/production.enc.yaml > secrets/production.yaml
    
- name: Apply decrypted secrets
  run: |
    kubectl apply -f secrets/production.yaml
```

---

## 롤아웃 전략과 검증

### 블루-그린 배포
블루-그린 배포는 새 버전(그린)을 배포한 후 트래픽을 한 번에 전환하는 방식입니다:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    version: "green"  # 새 버전으로 트래픽 전환
  ports:
  - port: 80
```

### 카나리 배포
카나리 배포는 새 버전을 점진적으로 롤아웃하여 위험을 최소화합니다:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-canary
spec:
  replicas: 2  # 전체 트래픽의 작은 비율
  selector:
    matchLabels:
      app: myapp
      version: canary
```

### 롤아웃 검증 및 자동 롤백
배포 후 검증을 통해 문제를 조기에 발견하고 자동으로 롤백할 수 있습니다:

```yaml
- name: Run post-deployment tests
  run: |
    # 통합 테스트 실행
    kubectl run integration-test \
      --rm \
      -i \
      --restart=Never \
      --image=myapp-test \
      -- \
      npm test
    
    # 성능 테스트 실행
    kubectl run performance-test \
      --rm \
      -i \
      --restart=Never \
      --image=k6 \
      -- \
      run /tests/performance.js

- name: Rollback on failure
  if: failure()
  run: |
    kubectl rollout undo deployment/myapp \
      --namespace production
```

---

## 모니터링과 알림

### 배포 상태 모니터링
배포 성공 여부와 애플리케이션 상태를 실시간으로 모니터링합니다:

```yaml
- name: Monitor deployment metrics
  run: |
    # 애플리케이션 메트릭 확인
    kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1 \
      | jq '.items[] | select(.metricName | contains("http_requests"))'
    
    # 오류율 확인
    ERROR_RATE=$(kubectl logs deployment/myapp \
      --since=5m \
      | grep -c "ERROR" \
      | awk '{print $1/300}')  # 5분당 오류율
    
    if (( $(echo "$ERROR_RATE > 0.01" | bc -l) )); then
      echo "High error rate detected: $ERROR_RATE"
      exit 1
    fi
```

### 다양한 채널을 통한 알림
배포 결과를 다양한 채널로 알림:

```yaml
- name: Send Slack notification
  uses: 8398a7/action-slack@v3
  with:
    status: ${{ job.status }}
    fields: repo,message,commit,author,action,eventName,ref,workflow,job,took
  env:
    SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
  
- name: Send Microsoft Teams notification
  uses: aliencube/microsoft-teams-actions@v0.8.0
  with:
    webhook_url: ${{ secrets.MS_TEAMS_WEBHOOK_URL }}
    title: "Deployment Completed"
    summary: "Deployment of ${{ github.repository }} completed"
    text: "Status: ${{ job.status }}, Commit: ${{ github.sha }}"
```

---

## 다중 환경 및 클러스터 배포

여러 환경이나 클러스터에 배포해야 하는 경우 매트릭스 전략을 사용할 수 있습니다:

```yaml
jobs:
  deploy:
    strategy:
      matrix:
        environment: [staging, production]
        cluster: [east, west]
    
    runs-on: ubuntu-latest
    
    steps:
    - name: Configure context for ${{ matrix.environment }} in ${{ matrix.cluster }}
      run: |
        kubectl config use-context ${{ matrix.cluster }}-${{ matrix.environment }}
    
    - name: Deploy to ${{ matrix.environment }} in ${{ matrix.cluster }}
      run: |
        helm upgrade --install myapp ./charts/myapp \
          --namespace ${{ matrix.environment }} \
          --values values/${{ matrix.environment }}.yaml \
          --set cluster=${{ matrix.cluster }}
```

---

## GitOps와의 통합

GitHub Actions CI 파이프라인을 GitOps 도구(Argo CD, Flux)와 통합할 수 있습니다:

```yaml
- name: Update GitOps manifest
  run: |
    # 배포 매니페스트 업데이트
    sed -i "s|image:.*|image: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}|g" \
      gitops/manifests/deployment.yaml
    
    # 변경사항 커밋 및 푸시
    git config user.name "GitHub Actions"
    git config user.email "actions@github.com"
    git add gitops/manifests/deployment.yaml
    git commit -m "Update image to ${{ github.sha }}"
    git push origin main

- name: Sync Argo CD application
  run: |
    argocd app sync myapp \
      --server ${{ secrets.ARGOCD_SERVER }} \
      --auth-token ${{ secrets.ARGOCD_TOKEN }}
```

이 접근 방식은 배포 상태의 "단일 진실 공급원"으로 Git을 사용하는 GitOps 원칙을 따릅니다.

---

## 문제 해결과 장애 복구

### 일반적인 배포 문제와 해결책

**이미지 풀 실패**
```bash
# 이미지 풀 실패 원인 확인
kubectl describe pod <pod-name> | grep -A 10 Events

# 이미지 레지스트리 접근 권한 확인
kubectl create secret docker-registry regcred \
  --docker-server=$REGISTRY \
  --docker-username=$USERNAME \
  --docker-password=$PASSWORD
```

**리소스 제한 문제**
```bash
# 리소스 사용량 확인
kubectl top pods --namespace production

# 리소스 요청/제한 조정
kubectl set resources deployment/myapp \
  --requests=cpu=200m,memory=256Mi \
  --limits=cpu=500m,memory=512Mi
```

**Readiness/Liveness 프로브 실패**
```bash
# 프로브 로그 확인
kubectl logs <pod-name> --previous

# 프로브 설정 조정
kubectl patch deployment myapp \
  --patch='{"spec":{"template":{"spec":{"containers":[{"name":"myapp","livenessProbe":{"initialDelaySeconds":30}}]}}}}'
```

### 자동화된 롤백
파이프라인에 자동 롤백 메커니즘을 구축합니다:

```yaml
- name: Automatic rollback on health check failure
  run: |
    # 건강 상태 확인
    HEALTH_STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
      http://myapp.production.svc.cluster.local/health)
    
    if [ "$HEALTH_STATUS" != "200" ]; then
      echo "Health check failed. Rolling back..."
      kubectl rollout undo deployment/myapp \
        --namespace production
      exit 1
    fi
```

---

## 성능 최적화와 비용 관리

### 빌드 시간 최적화
```yaml
- name: Optimized Docker build
  uses: docker/build-push-action@v6
  with:
    context: .
    # 레이어 캐싱 최적화
    cache-from: |
      type=registry,ref=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:buildcache
      type=gha
    cache-to: |
      type=registry,ref=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:buildcache,mode=max
      type=gha,mode=max
    # 빌드 인수로 종속성 캐싱
    build-args: |
      BUILDKIT_INLINE_CACHE=1
```

### Kubernetes 리소스 최적화
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
      - name: myapp
        resources:
          requests:
            cpu: "100m"    # 실제 사용량에 맞게 조정
            memory: "128Mi"
          limits:
            cpu: "200m"    # 피크 부하 대비 적절한 여유
            memory: "256Mi"
        # 효율적인 스케일링을 위한 준비 시간 설정
        lifecycle:
          preStop:
            exec:
              command: ["sh", "-c", "sleep 5"]
```

---

## 모범 사례와 권장사항

### 보안 모범 사례
1. **최소 권한 원칙**: 배포 서비스 계정에 필요한 최소 권한만 부여
2. **비밀 관리**: SealedSecrets나 외부 비밀 관리자를 사용하여 비밀을 안전하게 관리
3. **이미지 서명**: 모든 프로덕션 이미지에 서명하여 무결성 보장
4. **정기적인 스캔**: 취약점 스캐닝을 정기적으로 수행

### 안정성 모범 사례
1. **점진적 롤아웃**: 카나리 또는 블루-그린 배포로 위험 최소화
2. **자동 롤백**: 건강 상태 검사 실패 시 자동 롤백 구현
3. **모니터링**: 실시간 모니터링과 알림 설정
4. **재해 복구 계획**: 장애 발생 시 신속한 복구 계획 수립

### 효율성 모범 사례
1. **캐싱 전략**: 빌드 캐시를 효과적으로 활용하여 빌드 시간 단축
2. **병렬 실행**: 독립적인 작업을 병렬로 실행하여 전체 실행 시간 단축
3. **리소스 최적화**: 실제 사용량에 맞는 리소스 요청/제한 설정
4. **정리 작업**: 사용하지 않는 이미지와 리소스 정기 정리

---

## 결론

GitHub Actions를 활용한 Kubernetes 자동 배포는 현대적인 소프트웨어 개발과 배포의 핵심 요소입니다. 이 가이드에서 제시한 접근법과 모범 사례를 통해 다음과 같은 이점을 얻을 수 있습니다:

1. **개발자 생산성 향상**: 코드 푸시만으로 자동화된 배포 프로세스 실행
2. **배포 안정성 강화**: 자동화된 테스트, 검증, 롤백 메커니즘으로 배포 실패 위험 최소화
3. **보안 강화**: 이미지 스캔, 서명, 정책 검증을 통한 종합적 보안 관리
4. **운영 효율성 개선**: 일관된 배포 프로세스와 자동화로 운영 부담 감소
5. **비용 최적화**: 효율적인 리소스 사용과 빌드 최적화로 비용 절감

성공적인 배포 파이프라인 구축은 한 번에 완성되는 것이 아니라 지속적인 개선의 과정입니다. 작은 규모로 시작하여 점진적으로 기능을 추가하고, 실제 운영 데이터를 기반으로 지속적으로 최적화하는 접근법을 권장합니다.

이 가이드가 제공하는 패턴과 예제를 활용하여 조직의 특정 요구사항에 맞는 안전하고 효율적이며 확장 가능한 Kubernetes 배포 파이프라인을 구축하시기 바랍니다. 잘 설계된 배포 파이프라인은 단순히 애플리케이션을 배포하는 도구를 넘어서, 조직의 혁신 속도와 품질 수준을 결정하는 핵심 인프라가 될 것입니다.