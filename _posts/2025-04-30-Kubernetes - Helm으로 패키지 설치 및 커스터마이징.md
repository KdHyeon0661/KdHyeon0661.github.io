---
layout: post
title: Kubernetes - Helm으로 패키지 설치 및 커스터마이징
date: 2025-04-30 21:20:23 +0900
category: Kubernetes
---
# Helm으로 패키지 설치 및 커스터마이징

## 리포지토리 추가 및 동기화

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

여러 저장소를 추가해 둘 수 있으며, 이름 충돌 시 `repo/name` 접두를 사용해 구분한다.

---

## 설치 가능한 차트 검색

```bash
helm search repo nginx
```

예시 출력

```
NAME             CHART VERSION  APP VERSION  DESCRIPTION
bitnami/nginx    15.3.2         1.25.2       NGINX Open Source
```

원격 전체를 탐색하려면 `helm search hub <keyword>`를 사용할 수 있다(ArtifactHub 연동).

---

## 기본 설치(가장 빠른 길)

```bash
helm install my-nginx bitnami/nginx
kubectl get all
```

- `my-nginx`: 릴리스 이름(동일 차트를 여러 번 설치하려면 이름만 달리하면 됨)
- `bitnami/nginx`: 저장소/차트 이름

설치된 리소스 확인

```bash
helm list
helm status my-nginx
kubectl get deploy,svc,cm,secret,pod
```

---

## 세 가지 방식

### CLI로 즉시 오버라이드

```bash
helm install my-nginx bitnami/nginx \
  --set service.type=NodePort \
  --set service.nodePorts.http=30080
```

장점: 빠르고 간단.
단점: 값이 분산되면 재현성/가독성 저하. 운영에서는 **값 파일 사용** 권장.

---

### 커스텀 values 파일 사용(권장)

```yaml
# custom-values.yaml

replicaCount: 2

service:
  type: LoadBalancer
  port: 80

image:
  tag: 1.25.2
```

```bash
helm install my-nginx bitnami/nginx -f custom-values.yaml
```

여러 파일을 **레이어링**할 수 있다(뒤에 오는 파일이 앞의 값을 덮어씀).

```bash
helm install my-nginx bitnami/nginx \
  -f values.yaml -f values-prod.yaml -f values-prod-apne2.yaml
```

---

### 차트 기본값을 추출해 편집

```bash
helm show values bitnami/nginx > default-values.yaml
vim default-values.yaml
helm install my-nginx bitnami/nginx -f default-values.yaml
```

전체 옵션을 한눈에 파악하고 필요한 부분만 변경하기 좋다.

---

## 설치 후 값 변경(업그레이드)

설치 이후에도 values를 수정하여 업그레이드 가능하다.

```bash
helm upgrade my-nginx bitnami/nginx -f updated-values.yaml
```

변경 전/후 차이를 미리 보고 싶다면 **helm-diff 플러그인**을 사용한다.

```bash
helm plugin install https://github.com/databus23/helm-diff
helm diff upgrade my-nginx bitnami/nginx -f updated-values.yaml
```

**안전 업그레이드 옵션**

```bash
helm upgrade --install my-nginx bitnami/nginx \
  -f updated-values.yaml \
  --atomic --wait --timeout 5m
```

- `--atomic`: 실패 시 자동 롤백
- `--wait`: 리소스가 Ready될 때까지 대기
- `--timeout`: 대기 제한

---

## 릴리스 이력·상태·롤백

```bash
helm history my-nginx
helm status my-nginx
helm rollback my-nginx 2
```

원인 분석 시 `kubectl describe`, 이벤트, 파드 로그를 함께 본다.

---

## 릴리스 삭제

```bash
helm uninstall my-nginx
```

생성된 K8s 리소스가 함께 제거된다(다만 **PVC/외부 스토리지**는 남을 수 있으므로 정책 확인).

---

## 환경별 values 분리 패턴

- `values.yaml`: 공통 기본값
- `values-dev.yaml`, `values-stage.yaml`, `values-prod.yaml`: 환경별 오버라이드
- 지역/존 세분화: `values-prod-apne2.yaml` 등

배포 예시

```bash
helm upgrade --install web ./mychart \
  -f values.yaml \
  -f values-prod.yaml \
  -f values-prod-apne2.yaml \
  --atomic --wait
```

CI/CD에서는 **브랜치/태그/환경 변수**로 파일 조합을 결정한다.

---

## 실전 예제 모음

### Redis 설치 및 커스터마이징(Bitnami)

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install my-redis bitnami/redis \
  --set architecture=replication \
  --set auth.password=supersecretpassword \
  --set master.persistence.enabled=true \
  --set replica.replicaCount=2
```

운영 체크포인트
- 보안: `auth.enabled=true`, 비밀번호 Secret 외부화, NetworkPolicy
- 스토리지: `persistence.enabled=true`, `storageClass`/용량 확인
- 가용성: `architecture=replication`, Sentinel/HA 옵션 차트별 문서 확인

---

### NGINX(서비스 타입 커스터마이징)

```bash
helm install my-nginx bitnami/nginx \
  --set service.type=NodePort \
  --set service.nodePorts.http=30080
```

혹은 LoadBalancer

```bash
helm install my-nginx bitnami/nginx \
  --set service.type=LoadBalancer \
  --set service.ports.http=80
```

---

### Prometheus 스택(모니터링 예시, kube-prometheus-stack)

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm upgrade --install mon prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace \
  -f values-monitoring.yaml \
  --atomic --wait
```

`values-monitoring.yaml` 예시 포인트
- Grafana admin 비밀번호/Ingress
- 스토리지 클래스/보존 기간
- 리소스 리퀘스트/리밋, HPA/샤딩

---

## 설치 전 렌더링·검증·스키마

변환 결과 미리 보기(적용하지 않음)

```bash
helm template myrel ./mychart -f values-prod.yaml | tee rendered.yaml
```

린트

```bash
helm lint ./mychart
```

값 스키마(JSONSchema)로 형식/필수 키 검증(차트 루트에 `values.schema.json`)

```json
{
  "$schema": "https://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "replicaCount": { "type": "integer", "minimum": 1 },
    "service": {
      "type": "object",
      "properties": {
        "type": { "type": "string", "enum": ["ClusterIP","NodePort","LoadBalancer"] },
        "port": { "type": "integer", "minimum": 1, "maximum": 65535 }
      },
      "required": ["type","port"]
    }
  },
  "required": ["replicaCount","service"]
}
```

---

## 일반적인 커스터마이징 패턴

### 이미지 태그 고정 및 릴리스에 커밋 주입

```bash
helm upgrade --install web ./mychart \
  --set image.tag=1.2.3 \
  --set-string git.sha=$GIT_COMMIT
```

### Ingress 활성화(클래스/호스트/경로)

```yaml
# values.ingress.yaml

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
```

```bash
helm upgrade --install web ./mychart -f values.ingress.yaml
```

### HPA/리소스/확장

```yaml
hpa:
  enabled: true
  min: 2
  max: 8
  cpu: 70
resources:
  requests: { cpu: 200m, memory: 256Mi }
  limits:   { cpu: 500m, memory: 512Mi }
```

### PodDisruptionBudget

```yaml
pdb:
  enabled: true
  minAvailable: "50%"
```

### NodeSelector/첨두 시간 Tolerations/Affinity

```yaml
nodeSelector: { "nodegroup": "apps" }
tolerations:
  - key: "workload"
    operator: "Equal"
    value: "burst"
    effect: "NoSchedule"
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          topologyKey: "kubernetes.io/hostname"
          labelSelector:
            matchLabels:
              app.kubernetes.io/name: web
```

---

## 설치 전후 작업: Hooks·Test·NOTES

### 사전 마이그레이션 훅(Job)

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
          image: ghcr.io/acme/migrator:1.0.0
          args: ["./migrate.sh"]
```
{% endraw %}

### 배포 검증 테스트

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
      args: ["-sf", "http://{{ include "mychart.fullname" . }}:8080/healthz"]
```
{% endraw %}

실행

```bash
helm test myrel -n app
```

### NOTES.txt(설치 안내)

`templates/NOTES.txt`에 서비스 접근 방법, 기본 크리덴셜, 다음 단계 등을 적어두면 설치 후 `helm status`에서 바로 보인다.

---

## 보안·비밀 관리

Helm 자체는 값 파일 암호화를 제공하지 않는다. 운영에서는 다음 패턴을 사용한다.

- **SOPS + helm-secrets 플러그인**: 값을 암호화해 Git에 저장
- **Sealed Secrets**: 암호화된 Secret을 클러스터에서 복호화
- **External Secrets Operator**: AWS/GCP/Azure Secret Manager에서 동기화

예: helm-secrets

```bash
helm plugin install https://github.com/jkroepke/helm-secrets
# secrets-prod.yaml 를 sops 로 암호화

helm secrets enc secrets-prod.yaml
helm upgrade --install web ./mychart -f values-prod.yaml -f secrets-prod.yaml
```

주의
- Secret은 base64 인코딩일 뿐 암호화가 아니다 → etcd 암호화/권한(RBAC) 병행.
- 값 파일에 비밀을 평문으로 두지 않기(로컬/CI 로그 유출 주의).

---

## OCI 레지스트리 활용(권장 추세)

차트를 컨테이너 레지스트리(예: GHCR, ECR, GAR)에 저장

```bash
export HELM_EXPERIMENTAL_OCI=1
helm registry login ghcr.io
helm package ./mychart
helm push mychart-1.0.0.tgz oci://ghcr.io/acme/helm
helm pull oci://ghcr.io/acme/helm/mychart --version 1.0.0
```

OCI는 인증, 감사, 캐시/미러를 레지스트리 수준에서 통합 관리할 수 있어 엔터프라이즈에 적합하다.

---

## GitOps 연계(Argo CD/Flux)

Argo CD 예시(요지)

- Source: Helm
- valueFiles/parameters 지정
- 릴리스는 Git 상태에 의해 동기화
- 차트는 리포 혹은 OCI에서 가져옴

장점
- 선언적 배포, 자동 drift 수정, 리뷰 가능한 PR 기반 변경.

---

## 트러블슈팅 체크리스트

1. 렌더 확인: `helm template --debug --dry-run -f <values>.yaml`
2. 린트: `helm lint`
3. 차이 확인: `helm diff upgrade ...`
4. 대기/원자성: `--wait --atomic --timeout 5m`
5. 파드 상태: `kubectl describe pod`, 이벤트/오브젝트 상태 확인
6. CRD 버전: 차트가 필요로 하는 API 버전/CRD 설치 여부 확인
7. 네임스페이스: `-n <ns>` 일관성 유지
8. 권한: 이미지 풀 권한(imagePullSecrets), PDB/HPA로 인한 스케일 실패 여부
9. 스토리지: StorageClass, PVC 바인딩, 퍼미션 문제
10. Ingress: 클래스 이름/어노테이션/호스트/TLS Secret 이름 확인

---

## 명령어 요약

| 명령 | 설명 |
|---|---|
| `helm repo add <name> <url>` | 리포 추가 |
| `helm repo update` | 인덱스 갱신 |
| `helm search repo <kw>` | 리포 검색 |
| `helm show values <chart>` | 차트 기본값 출력 |
| `helm install <rel> <chart> [-f file] [--set k=v]` | 설치 |
| `helm upgrade <rel> <chart> ...` | 업그레이드 |
| `helm upgrade --install ...` | 없으면 설치/있으면 업그레이드 |
| `helm list [-n ns]` | 릴리스 목록 |
| `helm status <rel>` | 상태 조회 |
| `helm history <rel>` | 이력 조회 |
| `helm rollback <rel> <rev>` | 롤백 |
| `helm uninstall <rel>` | 제거 |
| `helm template <rel> <chart>` | 렌더 미리보기 |
| `helm lint <chart>` | 차트 검증 |
| `helm test <rel>` | 테스트 훅 실행 |
| `helm get values <rel> [-a]` | 실제 적용된 값 조회 |

---

## 완전한 설치 흐름 예시(요약)

```bash
# 리포 등록

helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# 후보 차트 조사

helm search repo nginx
helm show values bitnami/nginx > base.yaml

# 환경별 값 파일 준비

cp base.yaml values-prod.yaml
vim values-prod.yaml  # LB, Ingress, 리소스, HPA 등 수정

# 사전 검증

helm template my-nginx bitnami/nginx -f values-prod.yaml | kubeconform -strict -
helm lint bitnami/nginx

# 안정적 설치

helm upgrade --install my-nginx bitnami/nginx \
  -f values-prod.yaml \
  --atomic --wait --timeout 5m

# 상태/지표 확인

helm status my-nginx
kubectl get all
kubectl logs deploy/my-nginx

# 값 변경/업그레이드

helm diff upgrade my-nginx bitnami/nginx -f values-prod.yaml
helm upgrade my-nginx bitnami/nginx -f values-prod.yaml --atomic --wait

# 문제 시 롤백

helm history my-nginx
helm rollback my-nginx 2
```

---

## 결론

Helm은 복잡한 Kubernetes 애플리케이션을 **패키징**하고, 환경별 **값 오버라이드**로 커스터마이징하며, **이력/롤백**으로 운영 안정성을 보장한다.
설치 전 렌더링/린트/디프, 업그레이드 시 원자성·대기 옵션, 테스트/훅, 값 스키마, 비밀 관리(SOPS/Sealed/ESO), OCI, GitOps까지 결합하면 **재현 가능한 배포 파이프라인**을 구축할 수 있다.
운영에서는 값 파일 레이어링과 차트 검증, 보안 정책, 스토리지/네트워크/리소스 설정을 체계적으로 관리하라. Helm은 그 목적을 달성하기 위한 **표준 도구**다.
