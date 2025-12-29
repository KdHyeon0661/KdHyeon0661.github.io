---
layout: post
title: Kubernetes - ClusterIP, NodePort, LoadBalancer, ExternalName
date: 2025-04-04 22:20:23 +0900
category: Kubernetes
---
# Kubernetes 핵심 오브젝트 이해하기: Service (ClusterIP, NodePort, LoadBalancer, ExternalName)

## 서론: Kubernetes 서비스의 중요성

Kubernetes 환경에서 서비스(Service)는 애플리케이션의 네트워킹을 관리하는 핵심 추상화 계층입니다. 동적으로 생성되고 삭제되는 파드(Pod)들 사이에 안정적인 통신 경로를 제공하며, 로드 밸런싱과 서비스 디스커버리를 구현합니다. 서비스를 이해하는 것은 Kubernetes 네트워킹을 마스터하는 데 필수적인 첫걸음입니다.

서비스의 주요 목적은 다음과 같습니다:
1. **안정적인 엔드포인트 제공**: 파드의 수명과 무관하게 일관된 접근 지점을 제공합니다.
2. **로드 밸런싱**: 여러 파드 인스턴스에 트래픽을 분산시킵니다.
3. **서비스 디스커버리**: DNS 이름을 통해 서비스를 쉽게 찾고 접근할 수 있게 합니다.
4. **네트워크 추상화**: 애플리케이션이 복잡한 네트워크 세부사항을 알 필요 없이 통신할 수 있게 합니다.

이 가이드는 서비스의 작동 원리부터 각 유형별 특징, 실무 적용 방법까지 체계적으로 설명합니다.

---

## 서비스의 기본 작동 원리

### 서비스 생성과 동작 흐름

서비스가 작동하는 전체 과정을 이해해 봅시다:

1. **서비스 생성**: 사용자가 셀렉터(selector)를 지정하여 서비스를 생성합니다.
2. **엔드포인트 생성**: 셀렉터와 일치하는 파드들을 찾아 엔드포인트(Endpoints)와 엔드포인트슬라이스(EndpointSlices)를 생성합니다.
3. **kube-proxy 감시**: 각 노드에서 실행되는 kube-proxy 데몬셋이 엔드포인트슬라이스 변경을 감시합니다.
4. **네트워크 규칙 업데이트**: kube-proxy가 iptables, ipvs 또는 eBPF를 통해 네트워크 규칙을 업데이트합니다.
5. **트래픽 라우팅**: 클라이언트가 서비스의 가상 IP(VIP)로 트래픽을 보내면 네트워크 규칙에 따라 실제 파드 IP로 트래픽이 전달됩니다.

### 주요 구성 요소 확인하기

```bash
# 서비스 상태 확인
kubectl get services

# 엔드포인트 확인 (서비스와 연결된 파드들)
kubectl get endpoints

# 엔드포인트슬라이스 확인 (더 세분화된 엔드포인트 정보)
kubectl get endpointslices

# 특정 서비스의 엔드포인트슬라이스 확인
kubectl get endpointslices -l kubernetes.io/service-name=<서비스이름>
```

### 내부 DNS 규칙

Kubernetes는 내부 DNS 서비스를 제공하여 서비스 디스커버리를 단순화합니다. 서비스의 DNS 이름은 다음과 같은 패턴을 따릅니다:
```
<서비스이름>.<네임스페이스>.svc.cluster.local
```

동일한 네임스페이스 내에서는 서비스 이름만으로도 접근이 가능합니다.

---

## 서비스 유형 비교: 언제 어떤 유형을 사용해야 할까?

Kubernetes는 다양한 서비스 유형을 제공하며, 각 유형은 특정 사용 사례에 적합합니다:

| 서비스 유형 | 가상 IP | 외부 접근 | 로드밸런서 필요 | 주요 사용 사례 |
|------------|---------|-----------|----------------|---------------|
| **ClusterIP** | 있음 | ❌ 불가능 | ❌ 필요 없음 | 내부 마이크로서비스 통신 |
| **NodePort** | 있음 | ✅ 가능 (노드IP:포트) | ❌ 필요 없음 | 간단한 외부 접근, 테스트 환경 |
| **LoadBalancer** | 있음 | ✅ 가능 (외부 IP) | ✅ 클라우드 또는 MetalLB | 프로덕션 환경 외부 노출 |
| **ExternalName** | 없음 (CNAME) | ✅ DNS를 통한 외부 접근 | ❌ 필요 없음 | 외부 서비스에 대한 DNS 별칭 |
| **Headless**<br>(`clusterIP: None`) | 없음 | 내부 접근만 | ❌ 필요 없음 | 스테이트풀셋, 직접 파드 디스커버리 |

---

## ClusterIP: 내부 통신의 표준

### 기본 개념

ClusterIP는 Kubernetes 클러스터 내부에서만 접근 가능한 서비스 유형입니다. 이는 마이크로서비스 아키텍처에서 서비스 간 통신에 가장 일반적으로 사용됩니다.

### 기본 예시

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api-service
  namespace: default
spec:
  selector:
    app: api
    version: v1
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 8080
  - name: metrics
    protocol: TCP
    port: 9090
    targetPort: 9090
```

### 세션 어피니티 설정

특정 상황에서는 클라이언트의 연결이 동일한 백엔드 파드로 유지되어야 할 수 있습니다. 이를 세션 어피니티(Session Affinity)라고 합니다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: sticky-service
spec:
  selector:
    app: webapp
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 3600  # 1시간 동안 동일한 클라이언트 IP는 동일한 파드로 라우팅
```

### 로컬 테스트를 위한 포트 포워딩

개발 환경에서 ClusterIP 서비스를 로컬에서 테스트하려면 포트 포워딩을 사용할 수 있습니다:

```bash
# 서비스의 80 포트를 로컬 8080 포트로 포워딩
kubectl port-forward svc/api-service 8080:80

# 이제 로컬에서 접근 가능
curl http://localhost:8080
```

---

## NodePort: 노드 포트를 통한 외부 접근

### 기본 개념

NodePort 서비스는 각 노드의 특정 포트를 열어 외부에서 클러스터 내 서비스에 접근할 수 있게 합니다. 이는 개발 및 테스트 환경에 유용하지만, 프로덕션 환경에서는 일반적으로 Ingress나 LoadBalancer와 함께 사용됩니다.

### 기본 예시

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-nodeport
spec:
  type: NodePort
  selector:
    app: web
  ports:
  - name: http
    port: 80          # 서비스 내부 포트
    targetPort: 8080  # 컨테이너 포트
    nodePort: 30080   # 노드 포트 (30000-32767 범위, 생략 시 자동 할당)
```

### 접근 방법

NodePort 서비스는 다음과 같은 방식으로 접근할 수 있습니다:
- 모든 노드의 IP 주소와 지정된 nodePort로 접근
- `http://<노드IP>:30080`
- 여러 노드가 있는 경우 어떤 노드로 접근해도 동일한 서비스로 라우팅됨

### 주의사항

1. **포트 범위**: NodePort는 30000-32767 범위의 포트를 사용합니다.
2. **보안 고려사항**: 모든 노드의 포트가 열리므로 방화벽 설정이 필요할 수 있습니다.
3. **소스 IP 보존**: 기본적으로 소스 IP 정보가 손실될 수 있습니다.

---

## LoadBalancer: 외부 로드밸런서 통합

### 기본 개념

LoadBalancer 서비스는 클라우드 공급자의 로드밸런서를 자동으로 프로비저닝하거나, 온프레미스 환경에서는 MetalLB와 같은 솔루션을 통해 외부 로드밸런서를 제공합니다.

### 기본 예시

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-loadbalancer
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
  - name: http
    port: 80
    targetPort: 8080
  # 외부 트래픽 정책 설정
  externalTrafficPolicy: Local
```

### 소스 IP 보존

LoadBalancer 서비스에서 클라이언트의 실제 IP 주소를 보존하려면 `externalTrafficPolicy: Local`을 설정해야 합니다:

```yaml
spec:
  type: LoadBalancer
  externalTrafficPolicy: Local
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080
```

**장점**:
- 클라이언트의 실제 소스 IP가 애플리케이션에 전달됨
- 로드밸런서와 노드 간의 홉(hop) 수가 줄어듦

**단점**:
- 백엔드 파드가 없는 노드로 트래픽이 전달되면 실패함
- 부하 분산이 고르지 않을 수 있음

### 클라우드별 주석(Annotation)

클라우드 공급자별로 추가 기능을 주석(annotation)을 통해 설정할 수 있습니다:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-lb
  annotations:
    # AWS 예시
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    service.beta.kubernetes.io/aws-load-balancer-backend-protocol: "http"
    # GCP 예시
    networking.gke.io/load-balancer-type: "Internal"
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080
```

### MetalLB (온프레미스 로드밸런서)

온프레미스 환경에서 LoadBalancer 서비스를 사용하려면 MetalLB를 설치해야 합니다:

```bash
# MetalLB 설치 (예시)
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.10/config/manifests/metallb-native.yaml

# IP 풀 구성
cat <<EOF | kubectl apply -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.1.240-192.168.1.250
EOF
```

---

## ExternalName: 외부 서비스에 대한 DNS 별칭

### 기본 개념

ExternalName 서비스는 쿠버네티스 내부에서 외부 서비스에 대한 DNS 별칭(CNAME)을 제공합니다. 이는 쿠버네티스 클러스터 외부에 있는 서비스를 마치 내부 서비스인 것처럼 참조할 수 있게 합니다.

### 기본 예시

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-database
spec:
  type: ExternalName
  externalName: database.company.com
```

### 사용 사례

1. **외부 데이터베이스 참조**: 클러스터 외부의 관리형 데이터베이스 서비스를 참조
2. **마이그레이션 중간 단계**: 외부 서비스를 내부로 마이그레이션하는 동안 사용
3. **개발/프로덕션 환경 분리**: 개발 환경에서는 외부 모의(mock) 서비스를 참조

### 제한사항

- 포트 지정 불가능
- 실제 트래픽 라우팅을 제공하지 않음 (DNS 리디렉션만)
- TLS/SSL 종료 불가능

---

## Headless 서비스: 직접 파드 디스커버리

### 기본 개념

Headless 서비스는 클러스터 IP를 할당하지 않는 서비스로, DNS 쿼리가 서비스의 가상 IP 대신 실제 파드 IP 목록을 반환합니다.

### 기본 예시

```yaml
apiVersion: v1
kind: Service
metadata:
  name: headless-db
spec:
  clusterIP: None  # Headless 서비스로 설정
  selector:
    app: database
  ports:
  - port: 5432
    targetPort: 5432
```

### 사용 사례

1. **스테이트풀셋(StatefulSet)과 함께 사용**: 각 파드가 고유한 네트워크 아이덴티티를 가질 때
2. **사이드카 선택적 통신**: 특정 파드와 직접 통신해야 할 때
3. **서비스 메시 구현**: 서비스 메시 아키텍처에서 사용

### DNS 동작

Headless 서비스에 대한 DNS 쿼리는 다음과 같이 동작합니다:
- 서비스 이름으로 쿼리: 연결된 모든 파드의 IP 주소 목록 반환
- 파드별 DNS: `<파드이름>.<서비스이름>.<네임스페이스>.svc.cluster.local`

---

## 서비스 디스커버리와 DNS

### 내부 DNS 구조

Kubernetes는 CoreDNS (이전에는 kube-dns)를 통해 내부 서비스 디스커버리를 제공합니다:

```
<서비스이름>.<네임스페이스>.svc.cluster.local
```

### DNS 테스트 방법

```bash
# 네트워크 디버깅 도구가 포함된 임시 파드 실행
kubectl run -it --rm dns-test --image=nicolaka/netshoot -- /bin/bash

# 컨테이너 내부에서 DNS 테스트
nslookup web-service.default.svc.cluster.local
dig +short web-service.default.svc.cluster.local

# 간단한 curl 테스트
curl -I http://web-service.default.svc.cluster.local
```

### 환경별 DNS 동작

1. **동일 네임스페이스**: 서비스 이름만으로 접근 가능 (`web-service`)
2. **다른 네임스페이스**: 전체 도메인 이름 필요 (`web-service.other-namespace.svc.cluster.local`)
3. **파드 내부**: `/etc/resolv.conf`에 검색 도메인 자동 구성

---

## kube-proxy 모드: iptables vs ipvs vs eBPF

### iptables 모드 (기본)

가장 널리 사용되는 모드로, 안정적이고 호환성이 뛰어납니다.

**특징**:
- 각 서비스와 엔드포인트에 대한 iptables 규칙 생성
- 규칙이 많아지면 성능 저하 가능성
- 모든 주요 배포판에서 지원

### ipvs 모드

대규모 클러스터에 적합한 고성능 모드입니다.

**특징**:
- 리눅스 커널의 IP 가상 서버(IPVS) 기능 사용
- 해시 테이블 기반으로 더 효율적인 부하 분산
- 수만 개의 서비스에서도 좋은 성능
- 세션 어피니티, 스케줄링 알고리즘 등 고급 기능 지원

설정 방법:
```bash
# kube-proxy 설정에서 mode 변경
kubectl edit configmap kube-proxy -n kube-system
# mode: "ipvs" 로 변경
```

### eBPF 모드 (Cilium 등)

최신 네트워킹 솔루션에서 제공하는 모드입니다.

**특징**:
- 커널 공간에서 네트워크 처리로 성능 향상
- kube-proxy 없이 서비스 구현 가능
- 고급 네트워킹 정책 및 가시성 기능

### 현재 모드 확인

```bash
# kube-proxy 설정 확인
kubectl -n kube-system get configmap kube-proxy -o yaml | grep -i mode

# 또는 kube-proxy 파드 로그 확인
kubectl -n kube-system logs ds/kube-proxy | grep -i "Using"
```

---

## 실전 예제: 종합적인 서비스 배포

### 1. 기본 애플리케이션 배포 (Deployment)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-application
  labels:
    app: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
      version: v1
  template:
    metadata:
      labels:
        app: web
        version: v1
    spec:
      containers:
      - name: web
        image: nginx:1.27-alpine
        ports:
        - containerPort: 80
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 15
          periodSeconds: 20
```

### 2. 다양한 서비스 유형 배포

```yaml
# ClusterIP 서비스 (내부 통신용)
apiVersion: v1
kind: Service
metadata:
  name: web-clusterip
spec:
  selector:
    app: web
    version: v1
  ports:
  - name: http
    port: 80
    targetPort: 80
---
# NodePort 서비스 (외부 테스트용)
apiVersion: v1
kind: Service
metadata:
  name: web-nodeport
spec:
  type: NodePort
  selector:
    app: web
    version: v1
  ports:
  - name: http
    port: 80
    targetPort: 80
    nodePort: 30080
---
# LoadBalancer 서비스 (프로덕션 외부 노출용)
apiVersion: v1
kind: Service
metadata:
  name: web-loadbalancer
  annotations:
    # 클라우드별 주석 추가 가능
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
spec:
  type: LoadBalancer
  selector:
    app: web
    version: v1
  externalTrafficPolicy: Local
  ports:
  - name: http
    port: 80
    targetPort: 80
```

### 3. 배포 및 검증

```bash
# 배포
kubectl apply -f deployment.yaml
kubectl apply -f services.yaml

# 상태 확인
kubectl get deployments
kubectl get pods -o wide
kubectl get services -o wide

# 엔드포인트 확인
kubectl get endpoints
kubectl describe service web-clusterip

# 내부 통신 테스트
kubectl run -it --rm test-pod --image=curlimages/curl -- sh
# 컨테이너 내부에서:
curl http://web-clusterip.default.svc.cluster.local

# 외부 접근 테스트 (NodePort)
# 브라우저에서 http://<노드IP>:30080 접속
```

---

## 고급 구성 옵션

### 1. 다중 포트 서비스

```yaml
apiVersion: v1
kind: Service
metadata:
  name: multi-port-service
spec:
  selector:
    app: myapp
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 8080
  - name: https
    protocol: TCP
    port: 443
    targetPort: 8443
  - name: metrics
    protocol: TCP
    port: 9090
    targetPort: 9090
```

### 2. 준비되지 않은 주소 게시

기본적으로 서비스는 준비 프로브(Readiness Probe)를 통과한 파드만 엔드포인트에 포함합니다. 이를 변경하려면:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: publish-not-ready
spec:
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 8080
  publishNotReadyAddresses: true
```

### 3. 클라이언트 IP 기반 세션 어피니티

```yaml
apiVersion: v1
kind: Service
metadata:
  name: session-affinity-service
spec:
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 8080
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800  # 3시간
```

---

## 문제 해결 가이드

### 일반적인 문제와 해결 방법

**문제 1: 서비스에 연결할 수 없음**
```bash
# 서비스 상태 확인
kubectl get service <서비스이름> -o wide

# 엔드포인트 확인 (파드가 연결되어 있는지)
kubectl get endpoints <서비스이름>

# 셀렉터와 파드 라벨 일치 확인
kubectl get pods --show-labels
kubectl describe service <서비스이름> | grep -i selector

# 파드가 정상 상태인지 확인
kubectl get pods -o wide
kubectl describe pod <파드이름>
```

**문제 2: NodePort로 접속 불가**
```bash
# NodePort 값 확인
kubectl get service <서비스이름> -o yaml | grep nodePort

# 노드 IP 확인
kubectl get nodes -o wide

# 방화벽 규칙 확인
# Linux: sudo iptables -L -n | grep <NodePort>
# 또는 방화벽 서비스 상태 확인

# 파드가 실행 중인 노드 확인
kubectl get pods -o wide
```

**문제 3: LoadBalancer 외부 IP가 할당되지 않음**
```bash
# 서비스 상태 확인
kubectl get service <서비스이름> -o wide

# 이벤트 확인
kubectl describe service <서비스이름>

# 클라우드 공급자 로드밸런서 상태 확인
# 또는 MetalLB 로그 확인
kubectl logs -n metallb-system -l app=metallb
```

**문제 4: 소스 IP가 손실됨**
```bash
# externalTrafficPolicy 확인
kubectl get service <서비스이름> -o yaml | grep -i trafficpolicy

# Local로 변경
kubectl patch service <서비스이름> -p '{"spec":{"externalTrafficPolicy":"Local"}}'

# 변경 후 파드 재배포 필요 여부 확인
```

**문제 5: DNS 해결 실패**
```bash
# CoreDNS 파드 상태 확인
kubectl get pods -n kube-system -l k8s-app=kube-dns

# CoreDNS 로그 확인
kubectl logs -n kube-system -l k8s-app=kube-dns

# 네임스페이스의 DNS 정책 확인
kubectl get ns <네임스페이스> -o yaml | grep -i dns
```

### 진단 명령어 모음

```bash
# 네트워크 관련 모든 리소스 확인
kubectl get svc,ep,endpointslices,pods -o wide

# 상세 정보 확인
kubectl describe service <서비스이름>
kubectl describe endpoints <서비스이름>
kubectl describe endpointslices <엔드포인트슬라이스이름>

# 네트워크 디버깅을 위한 임시 파드 실행
kubectl run -it --rm network-test --image=nicolaka/netshoot -- /bin/bash

# kube-proxy 상태 확인
kubectl -n kube-system get pods -l k8s-app=kube-proxy
kubectl -n kube-system logs -l k8s-app=kube-proxy --tail=100

# 네트워크 정책 확인
kubectl get networkpolicies -A
```

---

## 모범 사례와 권장사항

### 1. 네이밍 규칙
- 서비스 이름은 소문자, 하이픈으로 구분된 명사 사용 (`user-service`, `payment-gateway`)
- 포트 이름은 프로토콜과 용도를 반영 (`http`, `https`, `grpc`, `metrics`)

### 2. 레이블 관리
- 서비스 셀렉터와 파드 레이블을 명확하고 일관되게 관리
- 버전 레이블을 사용하여 카나리 배포나 A/B 테스트 지원

### 3. 헬스 체크 구현
- 모든 서비스는 readinessProbe와 livenessProbe 구현
- 헬스 체크 엔드포인트는 가벼우면서 의미 있는 검사 수행

### 4. 리소스 관리
- 필요 이상의 NodePort 사용 자제
- LoadBalancer 서비스는 실제로 외부 노출이 필요한 경우만 사용
- Internal LoadBalancer 사용으로 내부 통신 보안 강화

### 5. 보안 고려사항
- NetworkPolicy로 서비스 간 통신 제한
- 필요 없는 포트 노출 방지
- 민감한 서비스는 ClusterIP로 제한

### 6. 모니터링과 관측
- 서비스 가용성 메트릭 수집
- 엔드포인트 변화 모니터링
- 지연 시간과 오류율 추적

---

## 결론

Kubernetes 서비스는 현대적인 마이크로서비스 아키텍처의 핵심 구성 요소입니다. 서비스를 효과적으로 이해하고 사용함으로써 다음과 같은 이점을 얻을 수 있습니다:

1. **탄력적인 애플리케이션 아키텍처**: 동적으로 변하는 파드 환경에서도 안정적인 통신 보장
2. **효율적인 트래픽 관리**: 로드 밸런싱, 세션 관리, 트래픽 정책을 통한 최적화
3. **단순화된 서비스 디스커버리**: DNS 기반의 직관적인 서비스 찾기
4. **다양한 접근 시나리오 지원**: 내부 통신부터 외부 노출까지 다양한 요구사항 충족

서비스 유형별 특징을 정리하면:
- **ClusterIP**: 내부 마이크로서비스 통신의 기본
- **NodePort**: 개발 및 테스트 환경에서의 간편한 외부 접근
- **LoadBalancer**: 프로덕션 환경의 표준적인 외부 노출
- **ExternalName**: 외부 서비스에 대한 통합된 참조점 제공
- **Headless 서비스**: 특수한 디스커버리 패턴과 스테이트풀 워크로드 지원

서비스를 효과적으로 운영하기 위해서는 지속적인 모니터링, 적절한 구성, 그리고 체계적인 문제 해결 프로세스가 필요합니다. 서비스 메시(Service Mesh)나 게이트웨이 API(Gateway API)와 같은 고급 네트워킹 패턴을 탐구하기 전에, 기본 서비스 개념을 확실히 이해하는 것이 중요합니다.

이 가이드가 Kubernetes 서비스의 세계를 탐험하는 데 도움이 되었기를 바랍니다. 실제 프로젝트에 적용하면서 경험을 쌓고, 지속적으로 최신 Kubernetes 네트워킹 기능을 학습해 나가시기 바랍니다.