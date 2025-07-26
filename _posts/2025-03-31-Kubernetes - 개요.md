---
layout: post
title: Kubernetes - 개요
date: 2025-03-31 19:20:23 +0900
category: Kubernetes
---
# 쿠버네티스란? 왜 필요한가?

쿠버네티스(Kubernetes, 줄여서 K8s)는 **컨테이너화된 애플리케이션의 자동 배포, 확장, 관리**를 위한 **오픈소스 컨테이너 오케스트레이션 플랫폼**입니다. 원래 Google이 개발하였고 현재는 CNCF(Cloud Native Computing Foundation)에서 관리합니다.

## 쿠버네티스의 정의

- **Kubernetes**는 '조타수(helmsman)'라는 뜻의 그리스어에서 유래되었으며, 클러스터 형태로 구성된 인프라에서 컨테이너들을 체계적으로 관리합니다.
- Docker와 같은 컨테이너 런타임 위에서 **다수의 컨테이너를 스케줄링하고 운영할 수 있는 환경**을 제공합니다.

## 왜 필요한가?

### 1. 수작업 배포의 한계

기존에는 서버에 직접 접속하여 애플리케이션을 배포하거나 스크립트로 자동화했지만, 이 방식은 다음과 같은 문제점을 가지고 있습니다:

- 배포 환경마다 설정이 달라 오류 발생 가능성
- 수평 확장이 어려움 (여러 서버에 일일이 배포 필요)
- 장애 복구 및 재시작 수작업 처리
- 모니터링, 로깅, 로드밸런싱 등의 기능 부족

### 2. 컨테이너화의 대두

Docker 같은 컨테이너 기술은 애플리케이션을 패키징하고 이식성 있게 배포할 수 있도록 도와주었지만, **수십~수천 개의 컨테이너를 운영하려면 오케스트레이션이 필요**합니다.

### 3. 쿠버네티스의 역할

쿠버네티스는 이러한 문제를 해결합니다:

- 애플리케이션의 **자동 배포, 업데이트, 롤백**
- **부하 분산** 및 트래픽 관리
- **헬스 체크**와 자동 복구 (Self-Healing)
- **자동 확장/축소 (Auto-Scaling)**
- **비밀 관리(Secret)**, 설정 분리(ConfigMap)

---

# 전통적인 배포 방식 vs Docker vs Kubernetes

| 항목 | 전통적인 배포 방식 | Docker 단독 | Kubernetes |
|------|------------------|--------------|-------------|
| 배포 방법 | 수동 배포 (ssh, scp 등) | 컨테이너 이미지 실행 | Declarative YAML 정의 |
| 스케일링 | 수동 서버 추가 | 컨테이너 수동 복제 | `ReplicaSet`으로 자동 확장 |
| 장애 대응 | 수동 재시작 필요 | 컨테이너 수동 재실행 | 자동 재시작 및 상태 감지 |
| 로드밸런싱 | 외부 장비 또는 수작업 구성 | 없음 또는 외부 구성 | Service 및 Ingress 제공 |
| 설정 관리 | 파일 또는 환경 변수 | Dockerfile 내 삽입 | ConfigMap / Secret |
| 모니터링 | 로그 파일 수집 또는 외부 도구 | 제한적 | 통합된 모니터링 (Prometheus 등 연동) |

---

## 예시: 전통적인 Docker 배포

```bash
# Docker로 Nginx 실행
docker run -d -p 80:80 --name web nginx
```

## 예시: Kubernetes로 Nginx 배포

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```

```bash
# 배포 적용
kubectl apply -f nginx-deployment.yaml
```

---

## 결론

쿠버네티스는 단순히 "많은 컨테이너를 띄우는 도구"가 아니라, **서비스를 신뢰성 있게 운영하기 위한 핵심 인프라 플랫폼**입니다. 자동화된 배포, 확장, 관리를 통해 운영 복잡도를 낮추고, 서비스의 가용성과 안정성을 높여줍니다.

→ **클라우드 네이티브 시대의 핵심 기술**로 자리 잡고 있으며, 마이크로서비스 아키텍처와도 매우 잘 어울립니다.