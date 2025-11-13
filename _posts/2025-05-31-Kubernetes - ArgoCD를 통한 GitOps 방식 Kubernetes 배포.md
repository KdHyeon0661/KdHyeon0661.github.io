---
layout: post
title: Kubernetes - ArgoCD를 통한 GitOps 방식 Kubernetes 배포
date: 2025-05-31 20:20:23 +0900
category: Kubernetes
---
# Argo CD를 통한 GitOps 방식 Kubernetes 배포

## 1. GitOps 재정의 — 선언형·감시·동기화·드리프트 복구

- **선언형**: “실행 방법”이 아니라 **목표 상태(YAML/Helm/Kustomize)** 만 Git에 저장
- **감시/동기화**: Argo CD가 Git을 폴링/웹훅으로 감지 → 클러스터로 **Sync**
- **드리프트 복구**: Git과 다른 **실제 상태** 발생 시 **자동/수동 Self-Heal**
- **감사/재현성**: 변경 이력 = Git 히스토리

> 운영 포인트: **누가/언제/왜**를 Git PR로 추적. 쿠버네티스 내부에서 직접 `kubectl apply` 금지(드리프트 원인).

---

## 2. 설치(요약) & 초기 로그인

```bash
kubectl create ns argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 포트포워딩(UI)
kubectl -n argocd port-forward svc/argocd-server 8080:443

# 기본 비밀번호
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo
```

운영 시 권장:
- `--insecure` 비활성, Ingress + TLS(예: cert-manager) 적용
- OIDC(회사 SSO) 연동, 프로젝트/폴더 기반 RBAC 제한

---

## 3. 리포지토리 전략 — 모노레포 vs 다중 레포, 환경 분리

### 3.1 폴더 레이아웃 예시(모노레포)

```
repo/
├─ apps/
│  ├─ payments/
│  │  ├─ base/           # Kustomize base (공통)
│  │  └─ overlays/
│  │     ├─ dev/
│  │     ├─ staging/
│  │     └─ prod/
│  └─ web/
│     ├─ base/
│     └─ overlays/{dev,staging,prod}
└─ platform/
   ├─ ingress/
   ├─ monitoring/
   └─ logging/
```

- **base/overlays**(Kustomize) 혹은 **Helm 차트 + values-(env).yaml**
- 환경(dev/staging/prod)별 파라미터 차등

### 3.2 App-of-Apps 패턴(루트 하나로 트리 구성)

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
  syncPolicy: { automated: { prune: true, selfHeal: true } }
```

`platform/root/` 폴더 안에서 **하위 Application 들을 선언**하여 **플랫폼·업무 앱**을 트리로 관리.

---

## 4. Application 선언 — Helm/Kustomize/Plain YAML

### 4.1 Kustomize 예

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

### 4.2 Helm 예

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

## 5. ApplicationSet — 다중 환경·클러스터 자동 생성

**템플릿 + 제너레이터**로 Application 다발 생성 (환경/리전 확장에 탁월)

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
        automated: { prune: true, selfHeal: true }
```
{% endraw %}

**Cluster generator**, **Git generator**, **Matrix generator** 조합으로 대규모 확장 자동화.

---

## 6. Sync 정책·웨이브·훅 — 배포 순서·후크 제어

### 6.1 Sync 옵션

```yaml
syncPolicy:
  automated: { prune: true, selfHeal: true }
  syncOptions:
    - CreateNamespace=true
    - ApplyOutOfSyncOnly=true
    - PruneLast=true
    - RespectIgnoreDifferences=true
```

- **PruneLast**: 삭제는 마지막에
- **ApplyOutOfSyncOnly**: 변경된 리소스만 적용

### 6.2 웨이브(우선순위) — 리소스 주석

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "0"   # 숫자 낮을수록 먼저
```

예: CRD(−1) → 네임스페이스(0) → Config(1) → Deployment(2) → Ingress(3)

### 6.3 훅(Hook) — Job/Workflow를 Sync 시점에 삽입

```yaml
metadata:
  annotations:
    argocd.argoproj.io/hook: PreSync      # PreSync/Sync/SyncFail/PostSync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
```

데이터 마이그레이션, 캐시 워밍, 헬스 체크 등 **배포 파이프라인을 선언형으로** 모델링.

---

## 7. Progressive Delivery — Argo Rollouts 연동

**점진 배포(캔어리/블루그린)** 로 리스크를 낮춤.

### 7.1 Rollout 리소스(캔어리 예)

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
        - setWeight: 10
        - pause: { duration: 180 }   # 모니터링 관찰
        - setWeight: 50
        - pause: { duration: 300 }
        - setWeight: 100
  selector:
    matchLabels: { app: web }
  template:
    metadata: { labels: { app: web } }
    spec:
      containers:
        - name: web
          image: your-registry/web:2.1.3
          ports: [{ name: http, containerPort: 8080 }]
```

### 7.2 Argo CD와 함께 쓰기

- Rollout CRDs/컨트롤러를 **플랫폼 계층**으로 배포
- Service/Ingress와 연계(가중 라우팅), **메트릭 게이트**(Prometheus)로 자동 중단/롤백

---

## 8. Secret 전략 — SOPS·SealedSecrets·Vault

### 8.1 Mozilla SOPS(+age/GPG) 예

Git에는 **암호화된 YAML**만 저장, 컨트롤러/플러그인으로 복호화 적용.

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

- Argo CD **SOPS 플러그인** 또는 **Kustomize SOPS**로 자동 복호화
- age 키는 **클러스터 외부**에 보관(키 관리 원칙)

### 8.2 SealedSecrets(리포지토리에는 암호문만)

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata: { name: registry-auth }
spec:
  encryptedData:
    .dockerconfigjson: AgAhg...  # 컨트롤러 키로 암호화
```

### 8.3 Vault(External Secrets)

- **External Secrets Operator**로 Vault/AWS SM/GCP SM 값을 Secret로 동기화
- Argo CD는 **Secret 껍데기**/연결만 선언 → 런타임 시 주입

> 운영 원칙: **Git에 평문 비밀 금지**. 키 수명/회전, 접근 통제, 감사로그 필수.

---

## 9. 프로젝트(Project)·RBAC·SSO — 멀티테넌시

### 9.1 프로젝트로 경계 설정

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

### 9.2 SSO(OIDC/SAML)과 매핑

- 그룹별 **프로젝트 롤** 매핑 → 팀 경계와 권한 최소화
- **Namespace-scoped Argo CD** 로 강한 격리도 가능

---

## 10. 알림/관측 — Notifications·Metrics·Dashboards

### 10.1 argocd-notifications

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: web-prod
  annotations:
    notifications.argoproj.io/subscribe.on-sync-succeeded.slack: '#deploy'
    notifications.argoproj.io/subscribe.on-health-degraded.slack: '#alerts'
```

### 10.2 메트릭

- `argocd_application_sync_status`, `..._health_status`, `..._sync_total`
- 대시보드: **Sync 성공률**, **드리프트 빈도**, **평균 Sync 시간**

#### 간단 KPI(배포 성공률)
배포 \(N\)회 중 성공 \(S\)회일 때
$$
\text{SuccessRate} = \frac{S}{N} \times 100(\%)
$$
SLO(예: 99%)를 유지하도록 롤백/게이트 조정.

---

## 11. 배포 시나리오(엔드투엔드)

### 11.1 신규 서비스 온보딩(Helm)

1. `charts/newsvc/` 추가, `values-{env}.yaml` 작성
2. `apps/newsvc/overlays/{env}` → Ingress/리소스 설정
3. `ApplicationSet`에 env/cluster 요소 추가
4. PR → 리뷰/머지 → Argo CD가 앱 생성·Sync
5. 알림으로 확인, 드리프트/헬스 모니터링

### 11.2 마이그레이션 훅 포함 배포

- `PreSync` Job로 DB 마이그 실행
- 캔어리로 10% → 50% → 100%
- `PostSync`로 캐시 예열/서킷 검증
- 이상 시 자동 중단, `argocd app rollback`(또는 Rollouts 자동 롤백)

---

## 12. 트러블슈팅 — 드리프트·헬스·동기화 실패

| 증상 | 원인 | 해결 |
|---|---|---|
| **OutOfSync 반복** | 외부 `kubectl` 변경 | 클러스터 접근 제한, SelfHeal=on |
| **Sync 실패** | CRD 미적용/순서 문제 | Sync Wave/`PrunePropagationPolicy`/`Replace=true` |
| **헬스 Degraded** | 커스텀 리소스 헬스 규칙 미정 | **Resource Health** Lua 확장 |
| **권한(403)** | 프로젝트/네임스페이스 제한 | AppProject/ServiceAccount 확인 |
| **Secret 복호화 실패** | SOPS 키 부재 | 키 배포/플러그인 설정 점검 |

CLI 도움말:
```bash
argocd app get web-prod
argocd app diff web-prod
argocd app sync web-prod
argocd app rollback web-prod
```

---

## 13. 보안·거버넌스 모범사례

- **Git 보호**: 변경은 PR만 → 필수 리뷰·승인·CI 검증
- **Argo CD 읽기 전용 원칙**: 운영자는 **Git만 변경**
- **Least Privilege**: AppProject/RBAC으로 소스/대상 제한
- **Secret 금지**: SOPS/SealedSecrets/Vault 사용
- **정책 엔진**: Kyverno/Gatekeeper로 “모든 앱은 PDB·리소스 제한 필수” 등 enforce
- **감사**: Argo CD Audit 로그 + Git 이력 보존

---

## 14. 백업/DR — Git + 상태 스냅샷

- **Git = 선언의 백업**. 여기에 더해:
  - Argo CD **ConfigMap/Secret**(프로젝트/클러스터 크레덴셜) 백업
  - `applications.argoproj.io` 리소스 백업
  - etcd/Velero로 클러스터 레벨 스냅샷
- 복구: 새 클러스터에 Argo CD 설치 → root app 적용 → **전환경 재구성**

---

## 15. 예제 모음

### 15.1 간단 Application (자동 생성/정리/자체 치유)

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
    automated: { prune: true, selfHeal: true }
    syncOptions:
      - CreateNamespace=true
```

### 15.2 Helm + 환경 값

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

### 15.3 Sync 웨이브/훅

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pre-migrate
  annotations:
    argocd.argoproj.io/sync-wave: "-1"
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec: { template: { spec: { containers: [{name: m, image: alpine, args: ["sh","-c","echo migrate"]}], restartPolicy: Never } } }
```

### 15.4 ApplicationSet — 폴더 스캔으로 자동 생성(Git generator)

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
      syncPolicy: { automated: { prune: true, selfHeal: true } }
```
{% endraw %}

---

## 16. 운영 체크리스트(한 장 요약)

- [ ] Git 브랜치 전략/PR 필수/CI 검증
- [ ] AppProject로 소스/대상/권한 제약
- [ ] ApplicationSet으로 환경/클러스터 확장 자동화
- [ ] Sync 옵션(Prune/SelfHeal/OutOfSyncOnly)·웨이브·훅 설계
- [ ] Rollouts로 점진 배포 + SLO/게이트
- [ ] Secret: SOPS/SealedSecrets/Vault 중 택1
- [ ] Notifications·대시보드·SLO/알림
- [ ] DR: root app·컨피그 백업·복구 리허설
- [ ] **클러스터 내 수동 변경 금지**(드리프트 근절)

---

## 결론

- **Argo CD = 선언형 배포 엔진**: Git에서 **상태를 정의**하고, 동기화/치유/정리까지 자동.
- 규모가 커질수록 **App-of-Apps**·**ApplicationSet**·**Rollouts**·**보안·정책 엔진** 결합이 필수.
- **관측/알림/DR**이 갖춰지면 GitOps는 **재현 가능하고 감당 가능한 운영 체계**가 된다.

---

## 참고 링크

- Argo CD 문서: <https://argo-cd.readthedocs.io/>
- Argo Rollouts: <https://argoproj.github.io/argo-rollouts/>
- ApplicationSet: <https://argocd-applicationset.readthedocs.io/>
- GitOps 소개: <https://www.weave.works/technologies/gitops/>
- SOPS: <https://github.com/mozilla/sops> / SealedSecrets: <https://github.com/bitnami-labs/sealed-secrets>
- External Secrets Operator: <https://external-secrets.io/>
