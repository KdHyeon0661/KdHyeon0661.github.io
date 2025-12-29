---
layout: post
title: Kubernetes - ArgoCD를 통한 GitOps 방식 Kubernetes 배포
date: 2025-05-31 20:20:23 +0900
category: Kubernetes
---
# Argo CD를 통한 GitOps 방식 Kubernetes 배포

## GitOps 개념 재정의: 선언형, 감시, 동기화, 자가 치유

GitOps는 인프라스트럭처와 애플리케이션 배포를 관리하는 현대적인 방식으로, 다음과 같은 핵심 원칙을 기반으로 합니다:

- **선언형 관리**: "어떻게 실행할 것인가"가 아닌 **목표 상태(YAML, Helm, Kustomize 매니페스트)**를 Git에 저장합니다.
- **지속적 감시와 동기화**: Argo CD가 Git 저장소를 지속적으로 모니터링하여 변경사항을 감지하고, Kubernetes 클러스터와 동기화합니다.
- **자가 치유(드리프트 복구)**: Git에 선언된 상태와 실제 클러스터 상태 간의 불일치가 발생하면 자동 또는 수동으로 복구합니다.
- **감사 가능성과 재현성**: 모든 변경 이력이 Git 히스토리로 관리되어 누가, 언제, 왜 변경했는지 추적 가능합니다.

> **운영 핵심 원칙**: 모든 변경은 Git Pull Request를 통해 이루어져야 하며, Kubernetes 클러스터 내에서 직접 `kubectl apply`를 실행하는 것은 드리프트의 주요 원인이 되므로 금지해야 합니다.

---

## Argo CD 설치 및 초기 설정

### 기본 설치

```bash
# Argo CD 네임스페이스 생성 및 설치
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 웹 UI 접근을 위한 포트 포워딩
kubectl -n argocd port-forward svc/argocd-server 8080:443

# 초기 관리자 비밀번호 확인
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo
```

### 운영 환경 권장 사항

- `--insecure` 옵션 비활성화 및 Ingress와 TLS(cert-manager 등) 적용
- OIDC를 통한 기업 SSO(Single Sign-On) 통합
- 프로젝트 및 폴더 기반 RBAC(Role-Based Access Control) 제한 설정

---

## Git 저장소 전략: 모노레포 vs 멀티레포

### 모노레포 폴더 구조 예시

```
repo/
├─ apps/
│  ├─ payments/
│  │  ├─ base/           # Kustomize 기본 설정 (공통)
│  │  └─ overlays/
│  │     ├─ dev/         # 개발 환경 설정
│  │     ├─ staging/     # 스테이징 환경 설정
│  │     └─ prod/        # 프로덕션 환경 설정
│  └─ web/
│     ├─ base/
│     └─ overlays/
│        ├─ dev/
│        ├─ staging/
│        └─ prod/
└─ platform/
   ├─ ingress/           # 인그레스 구성
   ├─ monitoring/        # 모니터링 구성
   └─ logging/           # 로깅 구성
```

- **Kustomize 방식**: base/overlays 패턴으로 환경별 설정 관리
- **Helm 방식**: 차트와 values-{env}.yaml 파일로 환경별 파라미터 관리
- 각 환경(dev, staging, prod)별로 다른 설정 값 적용

### App-of-Apps 패턴: 계층적 애플리케이션 관리

App-of-Apps 패턴은 루트 애플리케이션이 하위 애플리케이션들을 관리하는 트리 구조를 만듭니다:

```yaml
# root-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root-app
  namespace: argocd
spec:
  project: platform
  source:
    repoURL: https://github.com/your-org/repo
    targetRevision: main
    path: platform/root
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy: 
    automated: 
      prune: true
      selfHeal: true
```

`platform/root/` 디렉토리에는 플랫폼 및 업무 애플리케이션을 선언하는 하위 Application 매니페스트들이 포함되어 있어, 전체 애플리케이션 생태계를 트리 구조로 관리할 수 있습니다.

---

## Application 정의: 다양한 매니페스트 형식 지원

Argo CD는 Helm, Kustomize, 일반 YAML 파일 등 다양한 형식의 매니페스트를 지원합니다.

### Kustomize 기반 Application 예제

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: web-prod
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/your-org/repo
    targetRevision: main
    path: apps/web/overlays/prod
    kustomize:
      commonLabels:
        app.kubernetes.io/managed-by: argocd
  destination:
    server: https://kubernetes.default.svc
    namespace: web
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
```

### Helm 기반 Application 예제

```yaml
spec:
  source:
    repoURL: https://github.com/your-org/repo
    targetRevision: main
    path: charts/web
    helm:
      valueFiles:
        - values-prod.yaml
      parameters:
        - name: image.tag
          value: "2.1.3"
```

---

## ApplicationSet: 다중 환경 및 클러스터 자동 관리

ApplicationSet은 템플릿과 생성기(Generator)를 조합하여 여러 Application을 자동으로 생성하는 강력한 기능입니다. 환경이나 리전 확장에 특히 효과적입니다.

{% raw %}
```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: web-multi
  namespace: argocd
spec:
  generators:
    - list:
        elements:
          - cluster: prod-apne2
            url: https://prod-apne2.k8s.local
            env: prod
          - cluster: stg-apne2
            url: https://stg-apne2.k8s.local
            env: staging
  template:
    metadata:
      name: 'web-{{env}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/your-org/repo
        targetRevision: main
        path: apps/web/overlays/{{env}}
      destination:
        server: '{{url}}'
        namespace: web
      syncPolicy:
        automated: 
          prune: true
          selfHeal: true
```
{% endraw %}

ApplicationSet은 클러스터 생성기, Git 생성기, 매트릭스 생성기 등을 조합하여 대규모 확장을 자동화할 수 있습니다.

---

## 동기화 정책과 웨이브, 훅: 배포 순서와 라이프사이클 제어

### 동기화 옵션 설정

```yaml
syncPolicy:
  automated: 
    prune: true
    selfHeal: true
  syncOptions:
    - CreateNamespace=true
    - ApplyOutOfSyncOnly=true
    - PruneLast=true
    - RespectIgnoreDifferences=true
```

**주요 옵션 설명:**
- `PruneLast`: 리소스 삭제를 마지막에 수행
- `ApplyOutOfSyncOnly`: 변경된 리소스만 적용하여 성능 최적화

### 리소스 배포 순서 제어 (Sync Wave)

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "0"   # 숫자가 낮을수록 먼저 배포
```

**일반적인 배포 순서 예시:**
1. CRD(Custom Resource Definition): -1
2. 네임스페이스: 0  
3. ConfigMap/Secret: 1
4. Deployment/StatefulSet: 2
5. Service/Ingress: 3

### 배포 훅(Hook)을 통한 라이프사이클 관리

```yaml
metadata:
  annotations:
    argocd.argoproj.io/hook: PreSync      # PreSync, Sync, SyncFail, PostSync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
```

훅을 사용하면 데이터베이스 마이그레이션, 캐시 워밍업, 상태 검증과 같은 작업을 선언적으로 모델링할 수 있습니다.

---

## Progressive Delivery: Argo Rollouts를 통한 점진적 배포

Argo Rollouts는 카나리(Canary)나 블루-그린(Blue-Green) 배포 전략을 구현하여 배포 위험을 최소화합니다.

### Rollout 리소스 예시 (카나리 배포)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: web
spec:
  replicas: 8
  strategy:
    canary:
      steps:
        - setWeight: 10    # 트래픽의 10%로 시작
        - pause: 
            duration: 180  # 3분간 모니터링
        - setWeight: 50    # 트래픽의 50%로 확대
        - pause: 
            duration: 300  # 5분간 모니터링
        - setWeight: 100   # 전체 트래픽으로 전환
  selector:
    matchLabels: 
      app: web
  template:
    metadata: 
      labels: 
        app: web
    spec:
      containers:
        - name: web
          image: your-registry/web:2.1.3
          ports: 
            - name: http
              containerPort: 8080
```

### Argo CD와의 통합

- Rollout CRD(Custom Resource Definition)와 컨트롤러를 플랫폼 레이어로 배포
- 서비스와 인그레스를 통한 가중치 기반 트래픽 라우팅
- Prometheus와 같은 메트릭 시스템과 통합하여 자동 중단 및 롤백 구현

---

## 비밀 정보 관리 전략: SOPS, SealedSecrets, Vault

Git 저장소에는 평문 비밀 정보를 저장해서는 안 됩니다. 다음과 같은 방법으로 비밀 정보를 안전하게 관리할 수 있습니다.

### SOPS를 통한 암호화

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
stringData:
  username: ENC[AES256_GCM,data:...]
  password: ENC[AES256_GCM,data:...]
```

- Argo CD SOPS 플러그인 또는 Kustomize SOPS 통합을 통해 자동 복호화
- 암호화 키는 클러스터 외부에 안전하게 보관

### SealedSecrets 활용

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata: 
  name: registry-auth
spec:
  encryptedData:
    .dockerconfigjson: AgAhg...  # 클러스터별 키로 암호화
```

### Vault와 External Secrets Operator 통합

- External Secrets Operator를 사용하여 Vault, AWS Secrets Manager, GCP Secret Manager의 값을 Kubernetes Secret으로 동기화
- Argo CD는 Secret 매니페스트 구조만 관리하고, 실제 값은 런타임 시 주입

> **보안 원칙**: Git 저장소에는 절대 평문 비밀 정보를 저장하지 마세요. 키 수명 주기 관리, 접근 제어, 감사 로깅은 필수입니다.

---

## 프로젝트, RBAC, SSO: 다중 테넌트 환경 구축

### 프로젝트 기반 접근 제어

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: team-web
  namespace: argocd
spec:
  destinations:
    - namespace: web
      server: https://kubernetes.default.svc
  sourceRepos:
    - https://github.com/your-org/repo
  clusterResourceWhitelist:
    - group: '*'
      kind: Namespace
  roles:
    - name: dev
      policies:
        - p, proj:team-web:dev, applications, sync, team-web/*, allow
```

### SSO(Single Sign-On) 통합

- OIDC 또는 SAML을 통한 기업 인증 시스템 연동
- 그룹 기반 프로젝트 역할 매핑으로 팀별 접근 제한
- 필요에 따라 네임스페이스 단위로 Argo CD를 배포하여 강력한 격리 구현

---

## 알림 및 관측성: Notifications, 메트릭, 대시보드

### 알림 설정

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: web-prod
  annotations:
    notifications.argoproj.io/subscribe.on-sync-succeeded.slack: '#deploy'
    notifications.argoproj.io/subscribe.on-health-degraded.slack: '#alerts'
```

### 메트릭 수집

Argo CD는 다양한 메트릭을 제공합니다:
- `argocd_application_sync_status`: 동기화 상태
- `argocd_application_health_status`: 애플리케이션 건강 상태
- `argocd_application_sync_total`: 동기화 횟수

이를 기반으로 다음과 같은 대시보드를 구성할 수 있습니다:
- 동기화 성공률
- 드리프트 발생 빈도
- 평균 동기화 시간

#### 배포 성공률 KPI

배포 시도 \(N\)회 중 성공 \(S\)회일 때 성공률은 다음과 같이 계산합니다:

```
성공률 = (S / N) × 100%
```

서비스 수준 목표(SLO, 예: 99%)를 유지하기 위해 적절한 롤백 및 게이트 메커니즘을 구성해야 합니다.

---

## 종합 배포 시나리오: 신규 서비스 온보딩

### Helm을 사용한 신규 서비스 배포 절차

1. `charts/newsvc/` 디렉토리에 새로운 Helm 차트 추가
2. `values-{env}.yaml` 파일로 환경별 설정 구성
3. `apps/newsvc/overlays/{env}`에 인그레스 및 리소스 설정 추가
4. ApplicationSet에 새로운 환경/클러스터 요소 추가
5. Pull Request 생성 → 코드 리뷰 → 머지
6. Argo CD가 자동으로 애플리케이션 생성 및 동기화
7. 알림 확인 및 드리프트/상태 모니터링

### 마이그레이션 훅이 포함된 배포 예시

1. `PreSync` 훅으로 데이터베이스 마이그레이션 작업 실행
2. 카나리 배포 전략으로 10% → 50% → 100% 점진적 전환
3. `PostSync` 훅으로 캐시 워밍업 및 서킷 브레이커 검증
4. 이상 징후 감지 시 자동 중단 및 `argocd app rollback` 명령 실행

---

## 문제 해결 가이드: 드리프트, 상태 이상, 동기화 실패

| 증상 | 가능한 원인 | 해결 방안 |
|---|---|---|
| **반복적인 OutOfSync 상태** | 클러스터 내 수동 변경 | 클러스터 접근 제한, SelfHeal 활성화 |
| **동기화 실패** | CRD 미적용 또는 배포 순서 문제 | Sync Wave 설정, `PrunePropagationPolicy` 조정 |
| **상태 Degraded** | 커스텀 리소스 상태 검증 규칙 부재 | Resource Health Lua 확장 정의 |
| **권한 오류(403)** | 프로젝트/네임스페이스 제한 | AppProject 및 ServiceAccount 설정 확인 |
| **비밀 정보 복호화 실패** | SOPS 키 누락 | 키 배포 및 플러그인 설정 점검 |

### 유용한 CLI 명령어

```bash
# 애플리케이션 상태 확인
argocd app get web-prod

# Git과 클러스터 상태 비교
argocd app diff web-prod

# 수동 동기화 실행
argocd app sync web-prod

# 이전 버전으로 롤백
argocd app rollback web-prod
```

---

## 보안 및 거버넌스 모범 사례

- **Git 보안**: 모든 변경은 Protected Branch를 통한 Pull Request로만 허용
- **Argo CD 읽기 전용 원칙**: 운영자는 Git 저장소만 변경하고, Argo CD는 읽기 전용으로 유지
- **최소 권한 원칙**: AppProject와 RBAC을 통해 소스 저장소와 대상 클러스터를 제한
- **비밀 정보 보호**: SOPS, SealedSecrets, Vault 중 하나를 반드시 사용
- **정책 엔진 통합**: Kyverno 또는 Gatekeeper를 통해 "모든 애플리케이션은 PDB와 리소스 제한을 가져야 한다" 같은 규칙 강제
- **감사 로깅**: Argo CD 감사 로그와 Git 이력을 장기간 보존

---

## 백업 및 재해 복구 전략

Git 저장소 자체가 선언적 상태의 백업 역할을 하지만, 다음 추가 조치가 필요합니다:

- Argo CD 구성(ConfigMap, Secret, 프로젝트, 클러스터 인증 정보) 정기 백업
- `applications.argoproj.io` 리소스 백업
- etcd 또는 Velero를 통한 클러스터 수준 스냅샷
- 복구 절차: 새 클러스터에 Argo CD 설치 → 루트 애플리케이션 적용 → 전체 환경 자동 재구성

---

## 실전 예제 모음

### 기본 Application 예제

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/your-repo/k8s-manifests
    targetRevision: main
    path: nginx
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated: 
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### Helm과 환경별 값 파일 사용

```yaml
spec:
  source:
    repoURL: https://github.com/your-repo/infra
    path: charts/api
    targetRevision: main
    helm:
      valueFiles:
        - values-staging.yaml
      parameters:
        - name: resources.requests.cpu
          value: "200m"
```

### 동기화 웨이브와 훅이 포함된 Job

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pre-migrate
  annotations:
    argocd.argoproj.io/sync-wave: "-1"
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec: 
  template: 
    spec: 
      containers:
        - name: migrate
          image: alpine
          args: ["sh", "-c", "echo Running migration..."]
      restartPolicy: Never
```

### ApplicationSet - Git 생성기를 통한 폴더 기반 자동 생성

{% raw %}
```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: apps-by-folder
  namespace: argocd
spec:
  generators:
    - git:
        repoURL: https://github.com/your-org/repo
        revision: main
        directories:
          - path: apps/*/overlays/prod
  template:
    metadata:
      name: '{{path.basename}}-prod'
    spec:
      project: default
      source:
        repoURL: https://github.com/your-org/repo
        targetRevision: main
        path: '{{path}}'
      destination:
        server: https://prod-apne2.k8s.local
        namespace: '{{path.basename}}'
      syncPolicy: 
        automated: 
          prune: true
          selfHeal: true
```
{% endraw %}

---

## 결론

Argo CD를 통한 GitOps 방식은 Kubernetes 애플리케이션 배포와 관리를 혁신적으로 변화시킵니다:

1. **선언적 배포 엔진**: Git에 상태를 정의하고, 동기화, 자가 치유, 정리까지 자동화하는 완전한 배포 생태계를 제공합니다.

2. **확장성과 유연성**: 규모가 커질수록 App-of-Apps 패턴, ApplicationSet, Argo Rollouts와 같은 고급 기능이 필수적이며, 보안 정책 엔진과의 통합으로 거버넌스를 강화할 수 있습니다.

3. **운영 성숙도**: 관측성, 알림, 재해 복구 체계가 갖추어지면 GitOps는 재현 가능하고 관리 가능한 운영 모델로 진화합니다.

4. **문화적 전환**: GitOps는 단순한 기술 솔루션이 아닌 문화적 변화를 요구합니다. 모든 변경의 투명성, 협업 기반 검토, 자동화된 배포 파이프라인은 개발자와 운영팀 간의 장벽을 허물고 더 빠르고 안전한 소프트웨어 제공을 가능하게 합니다.

성공적인 GitOps 구현을 위해서는 기술적 구현뿐만 아니라 조직의 프로세스와 문화도 함께 발전시켜야 합니다. Argo CD는 이 여정에서 강력한 동반자가 될 것입니다.