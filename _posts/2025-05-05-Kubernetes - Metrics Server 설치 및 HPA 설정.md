---
layout: post
title: Kubernetes - Metrics Server 설치 및 HPA 설정
date: 2025-05-05 23:20:23 +0900
category: Kubernetes
---
# Metrics Server 설치 및 HPA(Horizontal Pod Autoscaler) 설정하기

Kubernetes에서 **자동으로 Pod 수를 조절**하려면  
CPU/메모리 사용량을 실시간으로 측정할 수 있어야 합니다.

이를 가능하게 해주는 구성은 다음과 같습니다:

- **Metrics Server**: 클러스터의 리소스 사용량을 수집 (CPU, 메모리)
- **HPA (Horizontal Pod Autoscaler)**: 리소스 사용량에 따라 Pod 수 자동 조정

---

## ✅ 1. Metrics Server란?

Kubernetes에서 Pod, Node의 리소스 사용량을 수집하는 **경량화된 메트릭 수집기**입니다.

- `kubectl top pod`, `kubectl top node` 등의 명령을 가능하게 함
- HPA에서 리소스 사용량 기준으로 스케일링하려면 반드시 필요함
- Prometheus와는 목적이 다름 (Prometheus는 장기 보관/모니터링용)

---

## ✅ 2. Metrics Server 설치

### ① 공식 매니페스트 사용 (K8s v1.27 기준)

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

> 참고: 클러스터가 Minikube, Kind 등 TLS 인증이 없는 환경일 경우 아래 수정이 필요할 수 있습니다.

```yaml
# components.yaml 파일 내 metrics-server Deployment 부분 수정
containers:
- name: metrics-server
  args:
    - --kubelet-insecure-tls
    - --kubelet-preferred-address-types=InternalIP
```

---

### ② 설치 확인

```bash
kubectl get deployment metrics-server -n kube-system
```

```bash
kubectl top nodes
kubectl top pods
```

→ 리소스 사용량이 출력되면 정상 동작 중

---

## ✅ 3. HPA(Horizontal Pod Autoscaler)란?

**CPU 또는 메모리 사용량을 기준으로 Pod 개수를 자동으로 조절하는 기능**입니다.

- `Deployment`, `ReplicaSet`, `StatefulSet` 등에 적용 가능
- `metrics-server`를 통해 수집한 리소스를 기준으로 작동
- YAML로 정의하거나 CLI로 생성 가능

---

## ✅ 4. HPA 실습: 예제 앱 배포

```bash
kubectl create deployment cpu-demo --image=nginx
kubectl expose deployment cpu-demo --port=80
```

---

## ✅ 5. HPA 생성 (CPU 기준)

```bash
kubectl autoscale deployment cpu-demo \
  --cpu-percent=50 \
  --min=1 \
  --max=5
```

→ CPU 사용률이 50%를 초과하면 자동으로 최대 5개까지 확장됨

### 생성된 HPA 확인

```bash
kubectl get hpa
kubectl describe hpa cpu-demo
```

출력 예시:

```
Name:                   cpu-demo
Namespace:              default
Target:                 Deployment/cpu-demo
Min replicas:           1
Max replicas:           5
Target CPU utilization: 50%
Current CPU utilization: 10%
```

---

## ✅ 6. CPU 부하 유발 (인위적 테스트)

### Busybox로 부하 발생시키기

```bash
kubectl run -i --tty load-generator --image=busybox /bin/sh
```

안에서 다음 명령 실행:

```sh
while true; do wget -q -O- http://cpu-demo; done
```

→ CPU 부하가 지속적으로 발생하여 HPA가 작동

### 결과 확인

```bash
kubectl get hpa
```

→ Pod 수가 자동으로 2 → 3 → 4... 증가하는 것을 확인 가능

---

## ✅ 7. HPA YAML 정의 예시

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: cpu-demo
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: cpu-demo
  minReplicas: 1
  maxReplicas: 5
  targetCPUUtilizationPercentage: 50
```

적용:

```bash
kubectl apply -f hpa.yaml
```

---

## ✅ 8. HPA V2 버전 (메모리 기준 포함)

`autoscaling/v2`부터는 메모리나 커스텀 메트릭도 가능해짐:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: mem-demo
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 1
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 60
```

---

## ✅ 9. 주의사항 및 트러블슈팅

| 증상 | 원인 및 해결 |
|------|---------------|
| `kubectl top` 결과 없음 | Metrics Server 미설치 또는 TLS 문제 |
| HPA가 작동 안함 | 실제 CPU 사용량이 낮거나 Metrics 수집 불가 |
| minReplicas만 유지됨 | 부하가 충분하지 않음 (테스트 필요) |
| Custom Metric HPA 불가 | Prometheus Adapter 설치 필요 |

---

## ✅ 결론

| 구성 요소 | 설명 |
|------------|------|
| **Metrics Server** | Kubernetes에서 리소스 사용량 수집 |
| **HPA** | 리소스 사용량에 따라 Pod 자동 확장 |
| **kubectl top** | 메트릭 확인 가능 (`metrics-server` 필요) |

> HPA는 클라우드 비용 최적화와 자동 확장을 위한 핵심 구성 요소입니다.  
> 메트릭 기반 오토스케일링으로 안정성과 탄력성을 확보할 수 있습니다.