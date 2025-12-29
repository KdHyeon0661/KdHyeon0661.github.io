---
layout: post
title: Kubernetes - Helm으로 패키지 설치 및 커스터마이징
date: 2025-04-30 21:20:23 +0900
category: Kubernetes
---
# Helm으로 패키지 설치 및 커스터마이징: 완전 가이드

## Helm 개요 및 시작하기

Helm은 Kubernetes 애플리케이션을 패키징, 배포, 관리하기 위한 표준 패키지 관리자입니다. 차트(Chart)라는 패키지 형식을 통해 복잡한 Kubernetes 애플리케이션을 쉽게 배포하고 버전 관리할 수 있습니다.

### Helm 리포지토리 관리

Helm 차트를 검색하고 설치하려면 먼저 리포지토리를 추가해야 합니다:

```bash
# 공식 Bitnami 리포지토리 추가
helm repo add bitnami https://charts.bitnami.com/bitnami

# 리포지토리 목록 업데이트 (최신 차트 정보 동기화)
helm repo update
```

여러 리포지토리를 추가할 수 있으며, 이름이 충돌하는 경우 `리포지토리명/차트명` 형식으로 구분하여 사용합니다.

### 차트 검색 및 탐색

설치 가능한 차트를 검색하려면:

```bash
# 리포지토리 내 NGINX 관련 차트 검색
helm search repo nginx
```

검색 결과 예시:
```
NAME             CHART VERSION  APP VERSION  DESCRIPTION
bitnami/nginx    15.3.2         1.25.2       NGINX Open Source
```

ArtifactHub와 같은 공개 차트 허브에서 검색하려면:
```bash
helm search hub nginx
```

---

## Helm 차트 설치 기본

### 가장 간단한 설치 방법

```bash
# 기본 옵션으로 NGINX 설치
helm install my-nginx bitnami/nginx
```

- `my-nginx`: 릴리스 이름 (동일 차트를 여러 번 설치할 때마다 고유한 이름 사용)
- `bitnami/nginx`: 리포지토리/차트 이름

### 설치 상태 확인

```bash
# Helm 릴리스 목록 확인
helm list

# 특정 릴리스 상세 상태 확인
helm status my-nginx

# 생성된 Kubernetes 리소스 확인
kubectl get deployment,service,configmap,secret,pod
```

---

## Helm 값(Values) 커스터마이징 방법

### 1. 명령줄 인수를 통한 즉석 오버라이드

```bash
helm install my-nginx bitnami/nginx \
  --set service.type=NodePort \
  --set service.nodePorts.http=30080 \
  --set replicaCount=3
```

**장점**: 빠르고 간단하게 테스트 가능
**단점**: 복잡한 구성에서는 가독성과 재현성이 떨어짐. 운영 환경에서는 권장하지 않음

### 2. 값 파일(Values File)을 활용한 구성 (권장)

`custom-values.yaml` 파일 생성:
```yaml
# custom-values.yaml
replicaCount: 2

service:
  type: LoadBalancer
  port: 80

image:
  tag: 1.25.2
  pullPolicy: IfNotPresent

resources:
  requests:
    memory: "256Mi"
    cpu: "250m"
  limits:
    memory: "512Mi"
    cpu: "500m"
```

값 파일을 사용한 설치:
```bash
helm install my-nginx bitnami/nginx -f custom-values.yaml
```

### 값 파일 레이어링

여러 값 파일을 조합하여 사용할 수 있습니다. 나중에 지정된 파일이 이전 값을 덮어씁니다:

```bash
helm install my-nginx bitnami/nginx \
  -f base-values.yaml \
  -f environment-values.yaml \
  -f region-specific-values.yaml
```

### 3. 기본값 추출 및 편집

차트의 모든 기본값을 확인하고 필요한 부분만 수정하려면:

```bash
# 차트의 기본 값 출력
helm show values bitnami/nginx

# 기본값을 파일로 저장
helm show values bitnami/nginx > default-values.yaml

# 파일 편집 후 설치
vim default-values.yaml
helm install my-nginx bitnami/nginx -f default-values.yaml
```

---

## 릴리스 관리 및 업그레이드

### 업그레이드 수행

설치된 릴리스의 구성을 변경하려면:

```bash
helm upgrade my-nginx bitnami/nginx -f updated-values.yaml
```

### 변경 사항 미리보기 (helm-diff 플러그인)

변경 사항을 적용하기 전에 미리 확인하려면 helm-diff 플러그인을 사용합니다:

```bash
# helm-diff 플러그인 설치
helm plugin install https://github.com/databus23/helm-diff

# 업그레이드 전 변경 사항 확인
helm diff upgrade my-nginx bitnami/nginx -f updated-values.yaml
```

### 안전한 업그레이드 옵션

프로덕션 환경에서는 다음과 같은 안전 옵션을 사용하는 것이 좋습니다:

```bash
helm upgrade --install my-nginx bitnami/nginx \
  -f updated-values.yaml \
  --atomic \
  --wait \
  --timeout 5m \
  --create-namespace \
  --namespace production
```

옵션 설명:
- `--atomic`: 업그레이드 실패 시 자동 롤백
- `--wait`: 모든 리소스가 준비 상태가 될 때까지 대기
- `--timeout`: 대기 시간 제한
- `--create-namespace`: 네임스페이스가 없으면 생성
- `--namespace`: 특정 네임스페이스에 설치

### 릴리스 기록 관리

```bash
# 릴리스 변경 이력 확인
helm history my-nginx

# 특정 리비전으로 롤백
helm rollback my-nginx 2

# 현재 적용된 값 확인
helm get values my-nginx --all
```

### 릴리스 삭제

```bash
helm uninstall my-nginx
```

**주의사항**: 기본적으로 Helm은 PVC(PersistentVolumeClaim)를 삭제하지 않습니다. 데이터 손실을 방지하기 위한 의도적인 설계입니다.

---

## 환경별 구성 관리 전략

효과적인 환경별 구성 관리를 위한 파일 구조 예시:

```
charts/
├── my-application/
│   ├── Chart.yaml
│   ├── values.yaml          # 기본값
│   ├── values-dev.yaml      # 개발 환경 오버라이드
│   ├── values-staging.yaml  # 스테이징 환경 오버라이드
│   ├── values-prod.yaml     # 프로덕션 환경 오버라이드
│   └── templates/
```

배포 예시 (프로덕션 환경):
```bash
helm upgrade --install my-app ./charts/my-application \
  -f charts/my-application/values.yaml \
  -f charts/my-application/values-prod.yaml \
  --atomic --wait --timeout 10m \
  --namespace production
```

---

## 실전 예제: 다양한 애플리케이션 설치

### Redis 클러스터 설치 (고가용성 구성)

```bash
helm install redis-cluster bitnami/redis \
  --set architecture=replication \
  --set auth.enabled=true \
  --set auth.password=$(openssl rand -base64 32) \
  --set master.persistence.enabled=true \
  --set master.persistence.size=10Gi \
  --set master.persistence.storageClass=gp2 \
  --set replica.replicaCount=2 \
  --set replica.persistence.enabled=true \
  --namespace database \
  --create-namespace
```

### NGINX Ingress Controller 설치

```bash
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --set controller.service.type=LoadBalancer \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-type"="nlb" \
  --set controller.autoscaling.enabled=true \
  --set controller.autoscaling.minReplicas=2 \
  --set controller.autoscaling.maxReplicas=10 \
  --namespace ingress-nginx \
  --create-namespace
```

### Prometheus 모니터링 스택 설치

```bash
# Prometheus 리포지토리 추가
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# kube-prometheus-stack 설치
helm install monitoring prometheus-community/kube-prometheus-stack \
  -f monitoring-values.yaml \
  --namespace monitoring \
  --create-namespace \
  --atomic --wait --timeout 15m
```

`monitoring-values.yaml` 주요 구성:
```yaml
grafana:
  adminPassword: ${GRAFANA_ADMIN_PASSWORD}
  persistence:
    enabled: true
    size: 10Gi

prometheus:
  prometheusSpec:
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: gp2
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 50Gi

alertmanager:
  alertmanagerSpec:
    storage:
      volumeClaimTemplate:
        spec:
          storageClassName: gp2
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 10Gi
```

---

## Helm 차트 검증 및 디버깅

### 템플릿 렌더링 미리보기

실제 Kubernetes 매니페스트를 생성하기 전에 템플릿 렌더링 결과를 확인:

```bash
helm template my-release ./my-chart -f values.yaml --debug
```

### 차트 린트(Lint)

차트의 문법과 구조 검증:

```bash
helm lint ./my-chart
```

### JSON Schema를 활용한 값 검증

차트 루트에 `values.schema.json` 파일을 생성하여 값의 유효성을 검증할 수 있습니다:

```json
{
  "$schema": "https://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "replicaCount": {
      "type": "integer",
      "minimum": 1,
      "maximum": 10,
      "description": "Number of application replicas to deploy"
    },
    "image": {
      "type": "object",
      "properties": {
        "repository": { "type": "string" },
        "tag": { "type": "string", "pattern": "^[0-9]+\\.[0-9]+\\.[0-9]+$" },
        "pullPolicy": { "type": "string", "enum": ["Always", "IfNotPresent", "Never"] }
      },
      "required": ["repository", "tag"]
    },
    "service": {
      "type": "object",
      "properties": {
        "type": { 
          "type": "string", 
          "enum": ["ClusterIP", "NodePort", "LoadBalancer", "ExternalName"] 
        },
        "port": { 
          "type": "integer", 
          "minimum": 1, 
          "maximum": 65535 
        }
      },
      "required": ["type", "port"]
    }
  },
  "required": ["replicaCount", "image", "service"]
}
```

---

## 고급 커스터마이징 패턴

### Ingress 구성

```yaml
# values-ingress.yaml
ingress:
  enabled: true
  className: nginx
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - host: app.example.com
      paths:
        - path: /
          pathType: Prefix
        - path: /api
          pathType: Prefix
  tls:
    - secretName: app-tls
      hosts:
        - app.example.com
```

### Horizontal Pod Autoscaler 구성

```yaml
# values-hpa.yaml
autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 80
```

### 리소스 요청 및 제한

```yaml
# values-resources.yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"
```

### Pod 배치 전략

```yaml
# values-affinity.yaml
nodeSelector:
  node-type: compute-optimized

tolerations:
  - key: "spot"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"

affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
            - key: "app.kubernetes.io/name"
              operator: In
              values: ["my-app"]
        topologyKey: "kubernetes.io/hostname"
```

---

## Helm Hooks 및 테스트

### 사전 설치 훅(Pre-install Hook)

데이터베이스 마이그레이션과 같은 사전 작업을 위한 Job 정의:

{% raw %}
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: "{{ .Release.Name }}-db-migrate"
  annotations:
    "helm.sh/hook": pre-install,pre-upgrade
    "helm.sh/hook-weight": "-5"
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  template:
    spec:
      restartPolicy: OnFailure
      containers:
        - name: migrator
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          command: ["/bin/sh", "-c"]
          args:
            - ./scripts/migrate-database.sh
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: connection-string
```
{% endraw %}

### 테스트 훅

애플리케이션 기능 검증을 위한 테스트:

{% raw %}
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: "{{ .Release.Name }}-integration-test"
  annotations:
    "helm.sh/hook": test
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  restartPolicy: Never
  containers:
    - name: test-runner
      image: alpine/curl
      command: ["sh", "-c"]
      args:
        - |
          echo "Testing service connectivity..."
          curl -f http://{{ .Release.Name }}-service:{{ .Values.service.port }}/health
          echo "Testing API endpoint..."
          curl -f http://{{ .Release.Name }}-service:{{ .Values.service.port }}/api/version
```
{% endraw %}

테스트 실행:
```bash
helm test my-release
```

### 설치 후 메시지 (NOTES.txt)

`templates/NOTES.txt` 파일을 생성하여 사용자에게 유용한 정보를 제공할 수 있습니다:

{% raw %}
```
Thank you for installing {{ .Chart.Name }}.

Your release is named {{ .Release.Name }}.

To access your application:
1. Get the application URL by running:
   echo http://{{ .Values.ingress.host }}

2. Retrieve the admin password:
   kubectl get secret {{ .Release.Name }}-admin-password \
     -o jsonpath="{.data.password}" | base64 --decode

Next steps:
- Review the configuration in values.yaml
- Consider setting up monitoring and alerts
- Configure backup for persistent data
```
{% endraw %}

---

## 보안 및 비밀 관리

### Helm Secrets 플러그인 (SOPS 통합)

민감한 값을 암호화하여 버전 관리 시스템에 안전하게 저장:

```bash
# helm-secrets 플러그인 설치
helm plugin install https://github.com/jkroepke/helm-secrets

# 값 파일 암호화
helm secrets enc values-secrets.yaml

# 암호화된 파일로 설치
helm secrets upgrade --install my-app ./my-chart \
  -f values.yaml \
  -f values-secrets.yaml

# 암호화된 파일 복호화 (필요시)
helm secrets dec values-secrets.yaml
```

### External Secrets Operator 연동

클라우드 제공자의 비밀 관리 서비스와 통합:

```yaml
# values-external-secrets.yaml
externalSecrets:
  enabled: true
  backend: aws-secrets-manager
  region: us-east-1
  secrets:
    - name: database-credentials
      key: /production/database/credentials
    - name: api-keys
      key: /production/api/keys
```

### 보안 모범 사례

1. **민감한 정보 절대 평문 저장 금지**: 비밀번호, API 키, 인증서 등은 항상 암호화
2. **최소 권한 원칙**: 서비스 계정에 필요한 최소한의 권한만 부여
3. **정기적 인증서/키 순환**: 자동 순환 정책 구현
4. **네트워크 정책**: 필요한 통신만 허용하는 네트워크 정책 적용
5. **Pod 보안 컨텍스트**: 비루트 사용자, 읽기 전용 루트 파일 시스템 등 보안 설정 강화

---

## OCI 레지스트리 통합

Helm 차트를 컨테이너 레지스트리에 저장하고 관리:

```bash
# 실험적 OCI 기능 활성화
export HELM_EXPERIMENTAL_OCI=1

# 컨테이너 레지스트리 로그인
helm registry login ghcr.io -u username -p token

# 차트 패키징
helm package ./my-chart

# OCI 레지스트리에 푸시
helm push my-chart-1.0.0.tgz oci://ghcr.io/my-org/charts

# OCI 레지스트리에서 풀
helm pull oci://ghcr.io/my-org/charts/my-chart --version 1.0.0
```

---

## GitOps 통합 (Argo CD 예시)

Helm과 GitOps 도구를 통합하여 선언적 배포 파이프라인 구축:

```yaml
# argocd-application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-application
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/my-org/helm-charts.git
    targetRevision: main
    path: charts/my-application
    helm:
      valueFiles:
        - values.yaml
        - values-production.yaml
      parameters:
        - name: image.tag
          value: v1.2.3
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

---

## 문제 해결 가이드

### 일반적인 문제 및 해결 방법

1. **템플릿 렌더링 오류**
   ```bash
   # 디버그 모드로 템플릿 렌더링
   helm template my-release ./my-chart --debug
   
   # 특정 값으로 테스트
   helm template my-release ./my-chart --set key=value
   ```

2. **업그레이드 실패**
   ```bash
   # 변경 사항 미리 확인
   helm diff upgrade my-release ./my-chart -f new-values.yaml
   
   # 롤백
   helm rollback my-release
   
   # 리소스 상태 확인
   kubectl get all -n namespace
   kubectl describe pod pod-name
   kubectl logs pod-name
   ```

3. **의존성 문제**
   ```bash
   # 차트 의존성 업데이트
   helm dependency update ./my-chart
   
   # 의존성 빌드
   helm dependency build ./my-chart
   ```

4. **네임스페이스 관련 문제**
   ```bash
   # 올바른 네임스페이스 사용 확인
   helm list -n correct-namespace
   
   # 네임스페이스 생성 옵션 사용
   helm upgrade --install ... --create-namespace --namespace target-ns
   ```

### 디버깅 체크리스트

- [ ] `helm lint`로 차트 문법 검증
- [ ] `helm template --debug`로 렌더링 결과 확인
- [ ] `helm diff`로 변경 사항 미리보기
- [ ] Kubernetes 리소스 상태 확인 (`kubectl get all`)
- [ ] Pod 이벤트 및 로그 확인
- [ ] 서비스 엔드포인트 확인
- [ ] 네트워크 정책 및 RBAC 권한 확인
- [ ] 스토리지 클래스 및 PVC 바인딩 상태 확인
- [ ] 리소스 할당량(ResourceQuota) 제한 확인
- [ ] 이미지 풀 정책 및 레지스트리 접근 권한 확인

---

## Helm 명령어 요약

| 명령어 | 설명 | 사용 예시 |
|---|---|---|
| `helm repo add` | Helm 리포지토리 추가 | `helm repo add bitnami https://charts.bitnami.com/bitnami` |
| `helm repo update` | 리포지토리 인덱스 갱신 | `helm repo update` |
| `helm search repo` | 리포지토리 내 차트 검색 | `helm search repo nginx` |
| `helm show values` | 차트 기본값 출력 | `helm show values bitnami/nginx` |
| `helm install` | 차트 설치 | `helm install my-app bitnami/nginx` |
| `helm upgrade` | 릴리스 업그레이드 | `helm upgrade my-app bitnami/nginx` |
| `helm upgrade --install` | 설치 또는 업그레이드 | `helm upgrade --install my-app bitnami/nginx` |
| `helm list` | 설치된 릴리스 목록 | `helm list -A` (모든 네임스페이스) |
| `helm status` | 릴리스 상태 확인 | `helm status my-app` |
| `helm history` | 릴리스 변경 이력 | `helm history my-app` |
| `helm rollback` | 특정 리비전으로 롤백 | `helm rollback my-app 2` |
| `helm uninstall` | 릴리스 삭제 | `helm uninstall my-app` |
| `helm template` | 템플릿 렌더링 | `helm template my-app ./my-chart` |
| `helm lint` | 차트 검증 | `helm lint ./my-chart` |
| `helm test` | 테스트 실행 | `helm test my-app` |
| `helm get values` | 적용된 값 확인 | `helm get values my-app -a` |

---

## 종합 실전 워크플로우 예시

다음은 프로덕션 환경에서 Helm을 사용한 완전한 배포 워크플로우 예시입니다:

```bash
# 1. 리포지토리 설정
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# 2. 차트 검색 및 정보 확인
helm search repo nginx
helm show values bitnami/nginx > nginx-defaults.yaml

# 3. 환경별 값 파일 준비
cat > nginx-prod-values.yaml << EOF
replicaCount: 3

service:
  type: LoadBalancer
  port: 80

ingress:
  enabled: true
  className: nginx
  hosts:
    - host: app.example.com
      paths:
        - path: /
          pathType: Prefix

resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "200m"
    memory: "256Mi"

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 5
  targetCPUUtilizationPercentage: 70
EOF

# 4. 사전 검증
helm lint bitnami/nginx
helm template my-nginx-prod bitnami/nginx -f nginx-prod-values.yaml | kubeconform -strict -

# 5. 안전한 설치
helm upgrade --install my-nginx-prod bitnami/nginx \
  -f nginx-prod-values.yaml \
  --namespace production \
  --create-namespace \
  --atomic \
  --wait \
  --timeout 10m

# 6. 설치 상태 확인
helm status my-nginx-prod -n production
kubectl get all -n production
kubectl get ingress -n production

# 7. 업그레이드 테스트
helm diff upgrade my-nginx-prod bitnami/nginx \
  -f nginx-prod-values.yaml \
  --namespace production

# 8. 롤링 업그레이드
helm upgrade my-nginx-prod bitnami/nginx \
  -f nginx-prod-values.yaml \
  --namespace production \
  --atomic \
  --wait \
  --timeout 10m

# 9. 모니터링 및 유지보수
helm history my-nginx-prod -n production

# 문제 발생 시 롤백
helm rollback my-nginx-prod 1 -n production

# 10. 정리 (필요시)
helm uninstall my-nginx-prod -n production
```

---

## 결론: Helm을 통한 효과적인 Kubernetes 애플리케이션 관리

Helm은 Kubernetes 애플리케이션의 패키징, 배포, 관리를 표준화하는 강력한 도구입니다. 효과적인 Helm 사용을 위한 핵심 원칙을 정리합니다:

### 1. 재현성과 일관성 보장
- **값 파일 버전 관리**: 모든 구성 변경을 Git으로 추적 가능하게 관리
- **환경별 구성**: 개발, 스테이징, 프로덕션 환경에 맞는 값 파일 유지
- **선언적 배포**: Helm 차트와 값 파일로 전체 인프라 상태 정의

### 2. 안전한 배포 파이프라인 구축
- **사전 검증**: 린트, 템플릿 렌더링, 스키마 검증으로 문제 조기 발견
- **안전한 업그레이드**: `--atomic`, `--wait` 옵션으로 롤백 가능한 배포 보장
- **변경 사항 투명성**: `helm diff`로 변경 내용 명확히 확인

### 3. 보안 및 비밀 관리
- **민감 정보 보호**: SOPS, Sealed Secrets, External Secrets로 비밀 안전하게 관리
- **최소 권한 원칙**: 서비스 계정에 필요한 권한만 부여
- **정기적인 감사**: 구성 변경과 접근 권한 정기적으로 검토

### 4. 운영 모범 사례 적용
- **리소스 관리**: 적절한 요청과 제한 설정으로 리소스 효율성 확보
- **고가용성 구성**: 복제본, 어피니티, PDB로 서비스 가용성 보장
- **모니터링 및 알림**: 헬스 체크, 메트릭, 로깅으로 시스템 상태 지속적으로 모니터링

### 5. 지속적인 개선
- **자동화**: CI/CD 파이프라인과 통합하여 배포 프로세스 자동화
- **피드백 루프**: 모니터링 데이터를 기반으로 구성 지속적으로 최적화
- **문서화 및 지식 공유**: 차트 문서화 및 팀 내 모범 사례 공유

Helm은 단순한 설치 도구를 넘어 Kubernetes 애플리케이션의 생명주기 전반을 관리하는 플랫폼입니다. 조직의 특정 요구사항에 맞게 Helm을 적절히 구성하고 활용한다면, 복잡한 분산 시스템을 안정적이고 효율적으로 운영할 수 있는 기반을 마련할 수 있을 것입니다.

차트 설계, 값 관리, 보안 구성, 배포 자동화 등 각 측면에서 신중한 계획과 구현이 필요합니다. 이 가이드에서 제시한 원칙과 모범 사례를 출발점으로 삼아, 조직의 성장과 함께 Helm 기반 배포 파이프라인을 지속적으로 발전시켜 나가시기 바랍니다.