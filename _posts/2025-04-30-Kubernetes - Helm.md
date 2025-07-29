---
layout: post
title: Kubernetes - Helm
date: 2025-04-30 19:20:23 +0900
category: Kubernetes
---
# Helm이란? 왜 사용하는가?

Kubernetes는 유연하고 강력한 오케스트레이션 플랫폼이지만,  
실제로 애플리케이션을 배포하고 관리하려면 수많은 YAML 파일을 직접 작성해야 합니다.

예를 들어, 단순한 웹 애플리케이션을 배포하기 위해 다음과 같은 리소스가 필요합니다:

- Deployment
- Service
- ConfigMap
- Secret
- Ingress
- PVC (PersistentVolumeClaim)

모든 환경(dev, stage, prod)마다 이 YAML들을 복사/붙여넣기하면서 수정을 반복하게 된다면?

→ ❗️**중복, 실수, 유지보수 지옥**이 기다리고 있습니다.

그래서 등장한 것이 **Helm**입니다.

---

## ✅ Helm이란?

**Helm은 Kubernetes의 패키지 매니저**입니다.  
`apt`, `yum`, `brew`와 비슷하게 **Kubernetes 애플리케이션을 템플릿화하고 배포**할 수 있게 도와줍니다.

- 복잡한 YAML 리소스를 **템플릿으로 관리**
- 한 번의 명령으로 여러 리소스를 배포/업데이트
- 설정값만 바꿔서 환경별 배포 가능 (values.yaml)

> 🎯 간단히 말하면, Helm은 **Kubernetes 앱을 배포·버전관리하는 도구**입니다.

---

## ✅ 왜 Helm을 써야 할까?

| 문제점 | Helm 도입 전 | Helm 도입 후 |
|--------|---------------|---------------|
| YAML 파일 너무 많음 | 수십~수백 개 직접 관리 | 템플릿 1세트 + values 파일만 관리 |
| 환경별 설정 반복 | dev/prod YAML 따로 작성 | `values.yaml`로 분리 |
| 배포 실수 | 수동 apply 중 오류 발생 | `helm install`로 한번에 배포 |
| 버전 관리 어려움 | 특정 배포 버전 추적 어려움 | `helm list`, `helm rollback` |
| 앱 공유 어려움 | 수동 복사/붙여넣기 | Helm Chart로 배포 가능 |

---

## ✅ Helm의 핵심 개념

| 용어 | 설명 |
|------|------|
| **Chart** | Helm 애플리케이션 패키지 (템플릿 + values + 메타데이터) |
| **Release** | Chart를 클러스터에 실제로 배포한 인스턴스 |
| **Repository** | Chart를 저장하고 배포하는 저장소 (ex. ArtifactHub) |
| **Values** | 설정값들. 환경마다 다르게 적용 가능 (values.yaml) |

---

## ✅ Helm Chart 구조 예시

```bash
mychart/
├── Chart.yaml         # 메타데이터
├── values.yaml        # 기본 설정값
├── templates/         # 리소스 템플릿 폴더
│   ├── deployment.yaml
│   ├── service.yaml
│   └── ingress.yaml
```

---

## ✅ 간단한 Helm 사용 흐름

```bash
# 1. Chart 생성
helm create mychart

# 2. 설정값 수정
vim mychart/values.yaml

# 3. 배포
helm install my-release ./mychart

# 4. 수정 후 업그레이드
helm upgrade my-release ./mychart

# 5. 배포된 리소스 확인
kubectl get all

# 6. 삭제
helm uninstall my-release
```

---

## ✅ Helm으로 배포된 앱은 어떻게 관리되나?

Helm은 `Release` 단위로 설치/업데이트/롤백이 가능합니다.

```bash
helm list               # 설치된 모든 릴리스 확인
helm rollback [릴리스명] [버전]  # 특정 버전으로 롤백
helm history [릴리스명]         # 배포 히스토리 보기
```

→ Kubernetes 리소스를 Helm이 내부적으로 관리하여  
   **운영자가 배포 이력을 추적하고, 빠르게 롤백할 수 있게** 도와줍니다.

---

## ✅ Helm은 이런 곳에 쓰입니다

- Nginx Ingress, Prometheus, Grafana, Redis, PostgreSQL 등 **오픈소스 설치**  
- 사내 서비스 배포용 **내부 Helm Chart 개발**
- 환경별 YAML 분기 대신 **`values-dev.yaml`, `values-prod.yaml`** 활용
- GitOps (ArgoCD, Flux 등)와 함께 **배포 자동화**

---

## ✅ Helm vs Kustomize 비교

| 항목 | Helm | Kustomize |
|------|------|-----------|
| 템플릿 기능 | ✅ Go 템플릿 | ❌ (오버레이 방식) |
| 복잡한 조건 분기 | 가능 | 제한적 |
| 주입 설정 (values) | 파일/CLI 모두 가능 | 패치 방식 |
| 학습 난이도 | 중간 | 쉬움 |
| 버전 관리 | Release 관리 | 없음 |

> 대부분 Helm이 더 강력하고, 복잡한 앱 배포에 적합  
> 하지만 단순한 설정 분리만 필요하면 Kustomize도 훌륭한 선택

---

## ✅ 결론

Helm은 다음과 같은 상황에서 **필수 도구**입니다:

- 여러 환경(dev/stage/prod)에 하나의 애플리케이션을 배포
- ConfigMap, Secret, PVC 등 리소스가 복잡한 서비스
- 오픈소스 도구(Grafana, Prometheus 등)를 손쉽게 설치
- CI/CD 또는 GitOps 환경에서 선언적으로 배포 관리