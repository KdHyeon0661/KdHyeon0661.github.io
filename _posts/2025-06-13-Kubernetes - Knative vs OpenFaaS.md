---
layout: post
title: Kubernetes - Knative vs OpenFaaS
date: 2025-06-13 19:20:23 +0900
category: Kubernetes
---
# K8s 기반 서버리스: Knative vs OpenFaaS

서버리스(Serverless)는 애플리케이션 배포와 운영에서 **인프라를 신경 쓰지 않고 코드 실행에만 집중할 수 있도록 해주는** 컴퓨팅 패러다임입니다.  
Amazon Lambda, Azure Functions와 같은 클라우드 서비스들이 대표적이지만, **Kubernetes 환경에서도 서버리스를 구현할 수 있는 오픈소스 플랫폼**들이 존재합니다.

이번 글에서는 **Kubernetes 기반 서버리스 플랫폼인 Knative와 OpenFaaS**를 구조, 구성요소, 차이점, 실무 활용 측면에서 비교합니다.

---

## ✅ 왜 Kubernetes에서 서버리스를 쓸까?

- 기존 서버리스 플랫폼은 벤더 종속적(AWS Lambda 등)
- K8s 환경에서의 **이식성**, **유연성**, **확장성 확보**
- 이벤트 기반 아키텍처 구성 가능
- CI/CD, 마이크로서비스, GitOps 등과 통합 용이

---

## ☁️ Knative: Google 주도의 K8s 기반 서버리스 프레임워크

### 📌 주요 구성 요소

| 컴포넌트 | 설명 |
|----------|------|
| **Knative Serving** | HTTP 기반 Function을 자동 배포, 확장, 스케일 제로 처리 |
| **Knative Eventing** | Kafka, CloudEvents 등 다양한 이벤트 소스 연동 |
| **Autoscaler** | 요청량 기반 Pod 수 자동 조절 (scale-to-zero 포함) |

### 📦 아키텍처

```
Ingress Gateway (Istio / Kourier)
      ↓
   Knative Serving
      ↓
K8s Deployment → Function Pod
```

### ✅ 특징

- HTTP 기반 요청 처리에 특화
- **자동 스케일 → 0**, 콜드 스타트 지원
- 다양한 Ingress(Gateway)와 통합 가능: Istio, Kourier 등
- **CloudEvents 표준 기반 Eventing** 지원
- **CRD(Custom Resource)** 기반 구성

### 🔧 설치

```bash
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.13.0/serving-crds.yaml
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.13.0/serving-core.yaml
```

---

## ⚡ OpenFaaS: 경량화된 서버리스 Function 프레임워크

### 📌 주요 구성 요소

| 컴포넌트 | 설명 |
|----------|------|
| **Gateway** | HTTP API를 통해 함수 배포 및 실행 관리 |
| **faas-netes** | Kubernetes 배포 담당 |
| **Function Watchdog** | 함수 실행 컨트롤러 |
| **Prometheus** | 모니터링 내장 지원 |
| **CLI (faas-cli)** | 함수 배포 자동화 도구 |

### 🏗 아키텍처

```
Client / Event → API Gateway
              ↓
         Function Pod (via faas-netes)
              ↓
           Metrics via Prometheus
```

### ✅ 특징

- HTTP뿐 아니라 **CronJob, Kafka, MQTT, NATS 등 다양한 트리거** 지원
- 설치 및 구성 간단 (`helm install` 기반)
- **Dockerfile 없이 handler 코드만 작성 → 빠른 개발**
- Prometheus 기반 실시간 함수 메트릭 제공
- 서버리스 외에도 **마이크로서비스 라우팅용**으로 활용 가능

### 🔧 설치 예시

```bash
helm repo add openfaas https://openfaas.github.io/faas-netes/
helm upgrade openfaas --install openfaas/openfaas \
  --namespace openfaas --create-namespace \
  --set basic_auth=true
```

---

## 🆚 Knative vs OpenFaaS 비교

| 항목 | Knative | OpenFaaS |
|------|---------|----------|
| 설치 난이도 | 중간~어려움 (Ingress 등 설정 필요) | 간단 (Helm 설치) |
| 트리거 방식 | HTTP + Eventing (CloudEvents) | HTTP + Cron + Kafka 등 다양 |
| 스케일링 | Scale-to-zero 지원 (기본) | Scale-to-zero 가능 (옵션) |
| 모니터링 | 별도 구성 필요 (Prometheus 연동 가능) | Prometheus 기본 내장 |
| 빌드 방식 | 컨테이너 이미지 필요 (Build 단계 외부 구성) | handler 기반 자동 빌드 |
| 학습 난이도 | 높음 (Custom CRD, Eventing 등 이해 필요) | 낮음 (CLI, YAML 기반) |
| 기업 사용 | Google Cloud Run 기반 | 다양한 환경에서 경량 활용 |
| 커뮤니티 | Google 주도, CNCF 산하 | 커뮤니티 중심 오픈소스 |

---

## 🧪 예시: 간단한 HTTP 함수 배포

### ▶ Knative Function 예시

```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: hello-knative
spec:
  template:
    spec:
      containers:
        - image: gcr.io/knative-samples/helloworld-go
          env:
            - name: TARGET
              value: "Knative"
```

```bash
kubectl apply -f hello-knative.yaml
kubectl get ksvc hello-knative
```

---

### ▶ OpenFaaS Function 예시

```bash
faas-cli new hello-openfaas --lang python
# handler.py 수정 후
faas-cli up -f hello-openfaas.yml
```

→ `http://<gateway-ip>:8080/function/hello-openfaas` 호출

---

## 🧭 어떤 플랫폼을 선택해야 할까?

| 상황 | 추천 플랫폼 |
|------|--------------|
| HTTP 기반 API 중심 | Knative |
| 다양한 이벤트 트리거 기반 자동화 | OpenFaaS |
| 서버리스 + 이벤트 스트리밍 | Knative Eventing |
| 간단한 자동화 서버리스 PoC | OpenFaaS |
| Google Cloud와 연계 고려 | Knative |
| 리소스 적은 환경, 경량 배포 | OpenFaaS |

---

## ✅ 결론

Kubernetes 기반 서버리스는 더 이상 복잡한 일이 아닙니다.

- Knative는 **HTTP 기반 요청 처리 + 확장성 + CloudEvents**에 강점
- OpenFaaS는 **경량, 빠른 개발, 다양한 트리거 처리**에 적합

자신의 환경과 요구사항에 맞는 플랫폼을 선택하여, **함수 기반 마이크로서비스** 또는 **자동화 파이프라인**을 손쉽게 구현해보세요.

---

## 📚 참고 자료

- [Knative 공식 문서](https://knative.dev/)
- [OpenFaaS 공식 사이트](https://www.openfaas.com/)
- [CloudEvents 사양](https://cloudevents.io/)
- [CNCF Serverless Landscape](https://landscape.cncf.io/category=serverless)