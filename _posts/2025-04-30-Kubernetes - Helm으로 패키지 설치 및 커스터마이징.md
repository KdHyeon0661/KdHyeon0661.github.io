---
layout: post
title: Kubernetes - Helm으로 패키지 설치 및 커스터마이징
date: 2025-04-30 21:20:23 +0900
category: Kubernetes
---
# Helm으로 패키지 설치 및 커스터마이징

Helm은 Kubernetes 환경에서 오픈소스 애플리케이션(NGINX, Prometheus, Redis 등)을  
간단한 명령 한 줄로 설치할 수 있게 해주는 **패키지 관리자**입니다.

또한 `values.yaml` 파일을 통해 **설정값을 변경하며 설치 또는 업그레이드**할 수 있어,  
운영 환경에서 매우 강력하고 유용합니다.

---

## ✅ 1. Helm Chart 저장소 추가

Helm 패키지를 설치하려면 먼저 해당 Chart가 있는 **Helm Repository**를 추가해야 합니다.

예: Bitnami 저장소 추가

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

---

## ✅ 2. 설치 가능한 Chart 검색

```bash
helm search repo nginx
```

```bash
NAME                    CHART VERSION  APP VERSION  DESCRIPTION
bitnami/nginx           15.3.2         1.25.2       NGINX Open Source
```

---

## ✅ 3. Helm Chart 기본 설치

예: nginx 설치

```bash
helm install my-nginx bitnami/nginx
```

- `my-nginx`: 릴리스 이름 (사용자가 정함)
- `bitnami/nginx`: Chart 이름 (저장소/이름)

설치 후 생성된 리소스 확인:

```bash
kubectl get all
```

---

## ✅ 4. 커스터마이징 방법

Helm은 Chart를 설치할 때 설정값을 덮어쓸 수 있습니다.

### ✅ 방법 1. CLI로 바로 설정

```bash
helm install my-nginx bitnami/nginx \
  --set service.type=NodePort \
  --set service.nodePorts.http=30080
```

→ values.yaml 안의 설정값을 CLI에서 직접 수정

---

### ✅ 방법 2. 커스텀 values 파일 사용

먼저 값을 담은 YAML 파일을 생성:

📄 `custom-values.yaml`

```yaml
replicaCount: 2

service:
  type: LoadBalancer
  port: 80

image:
  tag: 1.25.2
```

설치 시 적용:

```bash
helm install my-nginx bitnami/nginx -f custom-values.yaml
```

---

### ✅ 방법 3. 설치된 Chart의 기본 values 추출해서 수정

```bash
helm show values bitnami/nginx > default-values.yaml
vim default-values.yaml  # 필요한 부분 수정
helm install my-nginx bitnami/nginx -f default-values.yaml
```

> 이 방법은 **전체 설정 옵션을 확인하면서 수정**할 수 있어 유용합니다.

---

## ✅ 5. 설치 후 값 변경 (Upgrade)

설치 후에도 설정값을 수정하고 업그레이드할 수 있습니다.

```bash
helm upgrade my-nginx bitnami/nginx -f updated-values.yaml
```

---

## ✅ 6. 릴리스 상태 확인

```bash
helm list
helm status my-nginx
```

---

## ✅ 7. 릴리스 삭제

```bash
helm uninstall my-nginx
```

→ 관련 Kubernetes 리소스도 함께 제거됨

---

## ✅ 8. 실전 팁: 환경별 values 분리

`values-dev.yaml`, `values-prod.yaml` 등 환경별 설정을 나눌 수 있습니다.

```bash
helm install webapp ./mychart -f values-dev.yaml
helm install webapp ./mychart -f values-prod.yaml
```

---

## ✅ 예시: Redis 설치 및 커스터마이징

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install my-redis bitnami/redis \
  --set architecture=replication \
  --set auth.password=supersecretpassword \
  --set master.persistence.enabled=true \
  --set replica.replicaCount=2
```

---

## ✅ Helm 명령어 요약

| 명령어 | 설명 |
|--------|------|
| `helm repo add` | Helm 저장소 추가 |
| `helm search repo` | Chart 검색 |
| `helm install` | Chart 설치 |
| `helm upgrade` | 릴리스 업그레이드 |
| `helm list` | 설치된 릴리스 목록 |
| `helm status` | 릴리스 상태 확인 |
| `helm uninstall` | 릴리스 삭제 |
| `helm show values` | 기본 설정값 출력 |
| `helm get values` | 설치된 릴리스의 설정값 조회 |

---

## ✅ 결론

Helm을 이용하면 복잡한 Kubernetes 애플리케이션도  
**간단한 명령어와 설정 파일만으로 설치, 설정, 배포, 롤백**까지 쉽게 수행할 수 있습니다.

Helm은 특히 다음과 같은 상황에서 유용합니다:

- 복잡한 오픈소스 앱 설치 (Redis, Kafka, Prometheus 등)
- 환경별 커스터마이징
- 설정 변경에 따른 빠른 배포
- GitOps 기반 배포 전략에 통합