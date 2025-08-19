---
layout: post
title: AWS - ECS, EKS
date: 2025-08-03 18:20:23 +0900
category: AWS
---
# 🐳 ECS / EKS: 컨테이너 서비스

AWS에서는 컨테이너 기반 애플리케이션을 배포하고 운영할 수 있도록 두 가지 대표적인 서비스인 **Amazon ECS (Elastic Container Service)**와 **Amazon EKS (Elastic Kubernetes Service)**를 제공합니다. 둘은 비슷한 목적을 가지고 있지만, 사용 방식과 철학이 다릅니다.

---

## ✅ Amazon ECS (Elastic Container Service)

### 🔷 개요

ECS는 AWS에서 제공하는 **완전관리형 컨테이너 오케스트레이션 서비스**로, **Docker 컨테이너를 손쉽게 실행**하고 관리할 수 있도록 지원합니다. 쿠버네티스 없이도 AWS 리소스만으로 컨테이너 클러스터를 운영할 수 있습니다.

### 🔷 주요 구성 요소

- **Task**: 하나 이상의 컨테이너로 구성된 작업 단위 (Pod에 대응)
- **Task Definition**: Task의 정의. 어떤 컨테이너를 쓸지, 환경변수, 리소스 설정 등을 포함.
- **Service**: Task를 지정된 개수로 실행하고 유지하는 오브젝트
- **Cluster**: Task와 Service가 배포되는 논리적 컨테이너
- **Launch Type**: 실행 방식 선택
  - `EC2`: 직접 관리하는 EC2 인스턴스에 컨테이너 배포
  - `Fargate`: 서버리스 방식으로 인프라 관리 없이 컨테이너 실행

### 🔷 ECS 특징

| 항목 | 설명 |
|------|------|
| 관리형 | AWS에서 ECS 클러스터와 인프라를 자동 관리 |
| Fargate 지원 | 서버를 직접 관리하지 않고 컨테이너 실행 가능 |
| IAM 통합 | 컨테이너별로 세분화된 권한 설정 |
| Auto Scaling | Service 단위로 오토스케일링 가능 |
| CloudWatch 통합 | 로그 및 모니터링 기본 제공 |

---

## ✅ Amazon EKS (Elastic Kubernetes Service)

### 🔷 개요

EKS는 AWS에서 제공하는 **완전관리형 Kubernetes 서비스**로, 표준 K8s API를 그대로 사용할 수 있으며 온프레미스 및 멀티클라우드와도 호환됩니다. 쿠버네티스의 복잡한 설치 및 업그레이드 과정을 AWS가 관리합니다.

### 🔷 주요 구성 요소

- **EKS 클러스터**: Kubernetes Control Plane을 AWS에서 관리
- **노드 그룹**: 워커 노드 (EC2 또는 Fargate 기반)
- **Fargate Profile**: 서버리스 방식으로 Pod 단위 자동 실행
- **EKS Add-ons**: CoreDNS, kube-proxy, VPC CNI 플러그인 등의 자동 설치

### 🔷 EKS 특징

| 항목 | 설명 |
|------|------|
| 표준 Kubernetes | 기존 K8s 도구 및 생태계와 100% 호환 |
| 관리형 Control Plane | AWS에서 고가용성으로 Control Plane 운영 |
| IAM 연동 | K8s RBAC과 AWS IAM 통합 |
| Helm, ArgoCD 사용 가능 | DevOps 및 GitOps 툴과 호환 |
| 자체 워커 노드 또는 Fargate 선택 가능 | 유연한 아키텍처 설계 가능 |

---

## ✅ ECS vs EKS 비교

| 항목 | ECS | EKS |
|------|-----|-----|
| 오케스트레이션 엔진 | AWS 자체 | Kubernetes |
| 복잡도 | 낮음 (간단한 구조) | 높음 (K8s 학습 필요) |
| 커뮤니티 도구 호환 | 제한적 | 풍부한 생태계 도구 (Helm, Istio 등) |
| 실행 방식 | EC2 / Fargate | EC2 / Fargate |
| 학습 곡선 | 낮음 | 높음 |
| 커스터마이징 | 제한적 | 매우 유연 |
| 멀티 클러스터 / 멀티 클라우드 | 제한적 | 가능 |
| 사용 사례 | 단순한 마이크로서비스, 서버리스에 가까운 구조 | 복잡한 MSA, 멀티 클러스터 운영 |

---

## ✅ 어떤 것을 선택할까?

| 상황 | 추천 서비스 |
|------|--------------|
| 빠르게 시작하고 싶을 때 | ECS + Fargate |
| K8s 생태계를 적극 활용하고자 할 때 | EKS |
| DevOps 팀이 Kubernetes에 익숙한 경우 | EKS |
| 단순한 워크로드 운영 | ECS |
| 하이브리드 또는 멀티클라우드 전략 | EKS |

---

## ✅ 실습 예시 (ECS + Fargate)

```bash
# Task Definition 생성
aws ecs register-task-definition \
  --family my-task \
  --container-definitions file://container-definition.json

# Cluster 생성
aws ecs create-cluster --cluster-name my-cluster

# Fargate Service 실행
aws ecs create-service \
  --cluster my-cluster \
  --service-name my-service \
  --task-definition my-task \
  --launch-type FARGATE \
  --network-configuration 'awsvpcConfiguration={subnets=[subnet-abc123],securityGroups=[sg-xyz456],assignPublicIp="ENABLED"}'
```

---

## ✅ 마무리

ECS와 EKS는 모두 AWS의 강력한 컨테이너 관리 서비스입니다. ECS는 빠르고 단순한 배포에 적합하며, EKS는 확장성과 유연성이 뛰어난 환경에서의 대규모 운영에 적합합니다. 두 서비스를 적절히 활용하면 다양한 요구사항에 맞는 컨테이너 인프라를 구축할 수 있습니다.
