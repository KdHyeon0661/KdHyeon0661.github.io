---
layout: post
title: Kubernetes - 개요
date: 2025-03-31 19:20:23 +0900
category: Kubernetes
---
# 쿠버네티스 입문 가이드

## 개요

컨테이너와 마이크로서비스 아키텍처가 보편화되면서, 수많은 컨테이너를 효율적으로 관리하고 운영하는 도구의 필요성이 절실해졌습니다. 수동 배포는 확장성과 신뢰성에 한계가 있으며, 장애 대응이 느리고 복잡합니다.

**쿠버네티스(Kubernetes)** 는 이러한 문제를 해결하기 위해 등장한 컨테이너 오케스트레이션 플랫폼입니다. 이 플랫폼은 애플리케이션의 배포, 스케일링, 네트워킹, 관측, 자가 치유 등을 자동화하여 개발자와 운영팀이 복잡한 인프라 관리 부담에서 벗어나 비즈니스 로직에 집중할 수 있도록 돕습니다.

---

## 쿠버네티스란 무엇인가?

- **어원**: 그리스어로 "조타수(helmsman)" 또는 "파일럿"을 의미합니다. 이는 많은 컨테이너라는 배(선박)들을 안전하게 이끄는 역할을 상징합니다.
- **기원**: Google 내부의 대규모 클러스터 관리 시스템인 'Borg'의 경험을 바탕으로 개발되어 2014년 오픈소스로 공개되었으며, 현재는 Cloud Native Computing Foundation(CNCF)의 주도 하에 발전하고 있습니다.
- **핵심 철학**: **"선언형(Declarative) 관리"** 입니다. 사용자가 애플리케이션이 *어떤 상태여야 하는지(YAML 파일로 정의)* 를 선언하면, 쿠버네티스의 컨트롤러가 지속적으로 현재 상태를 확인하며 선언된 원하는 상태(Desired State)로 자동 맞춥니다.

간단히 말해, **"내가 원하는 서비스의 상태를 선언하면, 쿠버네티스가 끊임없이 그 상태를 유지하도록 관리한다"** 는 것이 핵심입니다.

---

## 왜 쿠버네티스가 필요한가?

### 수동 및 전통적 배포의 한계

- **환경 불일치**: 개발, 스테이징, 프로덕션 환경 간의 OS, 라이브러리, 설정 차이로 인해 "내 로컬에서는 되는데" 문제가 빈번했습니다.
- **확장의 어려움**: 트래픽 증가 시 서버를 수동으로 추가하고 구성하는 작업은 시간이 많이 들고 오류가 발생하기 쉽습니다.
- **가용성 관리 부재**: 서버나 프로세스가 다운되면 수동으로 감지하고 재시작해야 했으며, 정교한 헬스 체크가 어려웠습니다.
- **운영 복잡도**: 로그, 모니터링, 배포 상태에 대한 통합된 가시성이 부족하여 문제 해결이 느렸습니다.

### 컨테이너화의 등장과 새로운 과제

도커와 같은 컨테이너 기술은 **이미지 기반의 일관된 배포**를 가능하게 하여 환경 차이 문제를 해결하고, 리소스 효율을 높였습니다. 그러나 수십, 수백 개의 컨테이너가 실행되기 시작하면 새로운 문제들이 생깁니다.
- 컨테이너를 **어디에, 어떻게 배치할지(스케줄링)**?
- 컨테이너가 실패하면 **누가 자동으로 재시작할지**?
- 컨테이너 간에 **어떻게 통신할지(네트워킹)**?
- 새로운 버전을 **어떻게 안전하게 롤아웃하고, 문제가 생기면 롤백할지**?

이러한 복잡한 운영 작업들을 자동화하는 것이 바로 **컨테이너 오케스트레이션**의 역할이며, 쿠버네티스는 이 분야의 사실상 표준(de facto standard)이 되었습니다.

### 쿠버네티스가 제공하는 핵심 가치

1.  **자동화된 배포 및 업데이트**: 롤링 업데이트, 블루-그린, 카나리 배포 전략을 지원하여 무중단 배포와 안전한 롤백이 가능합니다.
2.  **서비스 디스커버리와 로드 밸런싱**: 컨테이너 그룹에 대한 안정적인 네트워크 엔드포인트를 제공하고 트래픽을 자동으로 분산합니다.
3.  **자가 치유(Self-healing)**: 컨테이너가 정상적으로 시작하지 않거나 중단되면 자동으로 재시작하며, 불량 노드의 워크로드를 다른 정상 노드로 재배치합니다.
4.  **수평적 확장**: CPU 사용률이나 커스텀 메트릭을 기반으로 애플리케이션 인스턴스(Pod) 수를 자동으로 조정합니다.
5.  **설정 및 비밀 관리**: 애플리케이션 설정과 민감정보(비밀번호, 토큰)를 컨테이너 이미지와 분리하여 안전하게 관리하고 동적으로 주입할 수 있습니다.
6.  **스토리지 오케스트레이션**: 로컬 스토리지, 클라우드 디스크 등 다양한 저장소를 컨테이너에 자동으로 마운트합니다.

---

## 전통 방식, Docker 단독, Kubernetes 비교

| 항목 | 전통 배포 (가상머신/물리서버) | Docker 단독 | Kubernetes |
|---|---|---|---|
| **배포 단위** | 애플리케이션 + 의존성 패키지 | Docker 이미지 | Pod (하나 이상의 컨테이너) |
| **배포 방식** | SSH/스크립트를 통한 수동 복사 및 구성 | `docker run`, `docker compose` 수동 실행 | YAML 선언문 적용 (`kubectl apply`) |
| **스케일링** | 수동 서버 증설/축소 | 컨테이너 복제 수동 관리 | ReplicaSet/Deployment로 복제수 선언, HPA로 자동 확장 |
| **장애 대응** | 모니터링 알람 후 수동 개입 | 컨테이너 다운 시 수동 재시작 | Liveness/Readiness Probe 기반 자동 재시작 및 재스케줄링 |
| **서비스 디스커버리** | 로드 밸런서 설정 또는 DNS 수동 관리 | 링크(--link) 또는 외부 도구 필요 | Service 리소스와 내장 DNS를 통한 자동 발견 |
| **트래픽 제어** | 하드웨어/소프트웨어 LB 별도 구성 | 리버스 프록시(예: Nginx) 수동 구성 | Service (L4), Ingress (L7) 리소스로 관리 |
| **설정 관리** | 설정 파일 또는 환경변수 분산 관리 | Dockerfile 내 ENV 또는 볼륨 마운트 | ConfigMap, Secret으로 중앙 관리 및 주입 |
| **관측성** | 서버별 로그/메트릭 도구 설치 필요 | 컨테이너 로그 확인 가능 | 표준화된 로그, 메트릭, 트레이스 수집 체계 (EFK, Prometheus, OTel) |

---

## 간단한 예시로 비교하기

### Docker로 Nginx 하나 실행하기

```bash
docker run -d -p 80:80 --name web nginx
```
**장점**: 매우 간단하고 빠르게 실행할 수 있습니다.
**한계**:
- 3개로 확장하려면 명령어를 두 번 더 실행하고, 로드 밸런서를 따로 설정해야 합니다.
- 컨테이너가 죽으면 자동으로 재시작되지 않습니다.
- 새 버전으로 업데이트할 때는 기존 컨테이너를 중지하고 새로 띄워야 하므로 다운타임이 발생합니다.

### Kubernetes로 Nginx 배포하기

```yaml
# nginx-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3 # 1. 3개의 복제본을 유지
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx:1.27-alpine
        ports:
        - containerPort: 80
        readinessProbe: # 2. 트래픽을 받을 준비가 됐는지 확인
          httpGet:
            path: /
            port: 80
        resources: # 3. 각 컨테이너에 필요한 자원 보장 및 제한
          requests:
            cpu: "50m"
            memory: "64Mi"
          limits:
            cpu: "200m"
            memory: "128Mi"
---
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
  type: LoadBalancer # 4. 외부 트래픽을 3개의 Pod으로 분산
```

```bash
kubectl apply -f nginx-deployment.yaml
```
위 YAML 파일 하나로 다음과 같은 이점을 얻습니다:
1.  **자동 복제 및 자가 치유**: 항상 3개의 Nginx Pod을 유지합니다. 하나가 죽으면 새로운 Pod이 자동 생성됩니다.
2.  **안정적인 배포**: `readinessProbe`를 통해 Pod이 트래픽을 처리할 준비가 된 후에만 서비스에 추가됩니다.
3.  **자원 보장 및 격리**: 각 컨테이너에 최소 자원을 보장하고, 한도를 설정하여 다른 워크로드를 방해하지 않습니다.
4.  **로드 밸런싱**: `Service` 리소스가 하나의 안정된 IP/도메인으로 트래픽을 3개의 Pod에 자동으로 분산합니다.

---

## 쿠버네티스 기본 아키텍처

쿠버네티스 클러스터는 크게 **컨트롤 플레인(Control Plane)** 과 **워커 노드(Worker Node)** 로 구성됩니다.

### 컨트롤 플레인 (마스터 노드)
클러스터의 두뇌 역할을 하며, 전반적인 상태를 관리하고 결정을 내립니다.
- **kube-apiserver**: 모든 관리 명령(`kubectl`)과 컴포넌트 간 통신의 진입점입니다.
- **etcd**: 클러스터의 모든 구성 데이터와 상태를 저장하는 고가용성 키-값 저장소입니다.
- **kube-scheduler**: 새로 생성된 Pod을 리소스, 정책, 제약 조건을 고려하여 적합한 워커 노드에 할당합니다.
- **kube-controller-manager**: Node Controller, Replication Controller 등 다양한 컨트롤러를 운영하여 현재 상태를 원하는 상태로 조정하는 제어 루프를 실행합니다.

### 워커 노드
실제 컨테이너(애플리케이션)가 실행되는 곳입니다.
- **kubelet**: 노드 에이전트로, Pod 명세를 받아 컨테이너를 실행하고 상태를 API 서버에 보고합니다.
- **Container Runtime**: Pod 내 컨테이너를 실행하는 소프트웨어입니다 (예: containerd, CRI-O).
- **kube-proxy**: 노드의 네트워크 규칙을 관리하여 Pod에 대한 네트워크 통신과 로드 밸런싱을 가능하게 합니다.
- **CNI 플러그인**: Pod 간 네트워킹을 제공합니다 (예: Calico, Cilium).

---

## 핵심 리소스 오브젝트와 YAML 실습

### 1. Pod, ReplicaSet, Deployment
- **Pod**: 쿠버네티스에서 배포 가능한 최소 단위로, 하나 이상의 밀접한 컨테이너 그룹입니다 (예: 앱 컨테이너 + 로그 수집 사이드카).
- **ReplicaSet**: 지정된 수의 동일한 Pod 복제본이 항상 실행되도록 유지합니다.
- **Deployment**: ReplicaSet을 관리하며, 애플리케이션 업데이트(롤링, 재생성)와 롤백을 위한 선언적 방법을 제공합니다. **가장 일반적으로 사용되는 워크로드 리소스입니다.**

```yaml
# deployment-example.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-api
  template: # 여기서 Pod 템플릿을 정의합니다.
    metadata:
      labels:
        app: my-api
    spec:
      containers:
      - name: api-container
        image: myregistry/api:v1.2.0
        ports:
        - containerPort: 8080
        env: # 환경변수 주입
        - name: DB_HOST
          value: "database-service"
        resources:
          requests: # 최소 보장 자원
            cpu: "100m"
            memory: "128Mi"
          limits: # 사용 상한 자원
            cpu: "500m"
            memory: "256Mi"
        livenessProbe: # 컨테이너가 살아있는지 검사
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
```

### 2. Service & Ingress
- **Service**: Pod 집합에 대한 안정적인 네트워크 엔드포인트(IP/도메인)를 정의하고, 트래픽을 로드 밸런싱합니다. (`ClusterIP`, `NodePort`, `LoadBalancer` 타입)
- **Ingress**: 클러스터 외부에서 내부 서비스로의 HTTP/HTTPS 경로를 관리합니다. 호스트 기반 또는 경로 기반 라우팅, TLS 종료 등을 처리합니다.

```yaml
# service-ingress-example.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-api-service
spec:
  selector:
    app: my-api
  ports:
  - port: 80 # 서비스가 노출하는 포트
    targetPort: 8080 # Pod의 컨테이너 포트
  type: ClusterIP
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-api-ingress
spec:
  rules:
  - host: api.mycompany.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-api-service
            port:
              number: 80
```

### 3. ConfigMap & Secret
- **ConfigMap**: 설정 데이터를 키-값 쌍으로 저장하여 Pod에 환경변수나 구성 파일로 주입할 수 있습니다.
- **Secret**: 비밀번호, OAuth 토큰, SSH 키 같은 민감한 정보를 저장합니다. Base64로 인코딩되지만, etcd 암호화 등을 통해 보안을 강화해야 합니다.

```yaml
# config-secret-example.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  log.level: "INFO"
  feature.enabled: "true"
---
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
stringData: # 자동으로 base64 인코딩됨
  username: admin
  password: very-secret-password
```

Deployment에서 사용 예시:
```yaml
env:
  - name: LOG_LEVEL
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: log.level
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: db-secret
        key: password
```

### 4. Volume, PersistentVolume(PV), PersistentVolumeClaim(PVC)
- **Volume**: Pod 내 컨테이너들이 데이터를 공유하거나 영구 저장할 수 있는 디렉토리를 제공합니다.
- **PersistentVolume (PV)**: 관리자가 프로비저닝하거나 스토리지 클래스(StorageClass)를 통해 동적으로 제공되는 클러스터의 스토리지 조각입니다.
- **PersistentVolumeClaim (PVC)**: 사용자(Pod)가 PV로부터 특정 크기 및 접근 모드의 스토리지를 요청하는 것입니다.

```yaml
# pvc-example.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: fast-ssd # 특정 스토리지 클래스 지정
```

Deployment에서 PVC 사용:
```yaml
spec:
  containers:
  - name: app
    image: myapp:latest
    volumeMounts:
    - mountPath: "/data"
      name: app-storage
  volumes:
  - name: app-storage
    persistentVolumeClaim:
      claimName: app-data-pvc
```

---

## 배포 전략

### 1. 롤링 업데이트 (RollingUpdate) - 기본
새 버전의 Pod을 하나씩 생성하고, 기존 버전의 Pod을 하나씩 제거하며 점진적으로 업데이트합니다. 무중단 배포가 가능합니다.
```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # 원하는 복제본 수를 초과하여 생성할 수 있는 최대 Pod 수
      maxUnavailable: 0  # 업데이트 중 사용 불가능할 수 있는 최대 Pod 수
```
**장점**: 리소스 사용 효율적, 다운타임 없음.
**주의점**: `readinessProbe`가 정확해야 새 Pod이 준비될 때까지 트래픽을 받지 않아 안정적입니다.

### 2. 카나리 배포 (Canary)
새 버전을 소량(예: 10%)의 사용자 트래픽에만 노출하여 오류율이나 성능을 모니터링한 후, 문제가 없으면 점차 비율을 높여가는 전략입니다. Ingress Controller(예: Nginx Ingress)의 어노테이션을 통해 가중치 기반 라우팅으로 구현할 수 있습니다.

### 3. 블루-그린 배포 (Blue-Green)
현재 버전(Blue)과 새 버전(Green)의 완전한 환경을 각각 준비합니다. 모든 테스트가 완료되면, 로드 밸런서의 목표를 한 번에 Green 환경으로 전환합니다.
**장점**: 롤백이 매우 빠름(다시 Blue로 전환).
**단점**: 두 세트의 리소스가 필요하여 비용이 더 듭니다.

---

## 모니터링과 관측성

안정적인 운영을 위해서는 클러스터와 애플리케이션의 상태를 파악할 수 있어야 합니다.

- **로그**: 애플리케이션 로그는 `stdout`/`stderr`로 출력하는 것을 원칙으로 합니다. **Fluentd**나 **Fluent Bit**을 데몬셋으로 배포하여 모든 노드의 컨테이너 로그를 수집한 후, **Elasticsearch**에 저장하고 **Kibana**에서 시각화하는 EFK 스택이 일반적입니다.
- **메트릭**: **Prometheus**는 쿠버네티스 생태계의 표준 메트릭 수집 도구입니다. 노드, Pod, 서비스 등의 메트릭을 수집하고, **Grafana**를 통해 대시보드를 구성합니다. Horizontal Pod Autoscaler(HPA)도 이 메트릭을 활용합니다.
- **트레이싱**: 분산 시스템에서 요청이 흐르는 경로와 각 구간의 소요 시간을 추적합니다. **OpenTelemetry(OTel)** 표준과 **Jaeger**나 **Tempo** 같은 백엔드를 사용합니다.

---

## 보안 고려사항

쿠버네티스 보안은 다층 방어 원칙을 따릅니다.
1.  **클러스터 보안**: API 서버 인증/인가(RBAC), etcd 암호화, 컨트롤 플레인 구성 요소 간 TLS 통신.
2.  **노드 보안**: 노드 OS 강화, kubelet 인증.
3.  **Pod/컨테이너 보안**:
    - **Pod Security Admission(PSA)**: Pod에 보안 표준(`restricted`, `baseline`)을 강제합니다.
    - **이미지 보안**: 신뢰할 수 있는 레지스트리 사용, 취약점 스캔(Trivy), 서명 검증(Cosign).
    - **런타임 보안**: 비루트 사용자 실행, 불필요한 Linux Capabilities 제거, 읽기 전용 파일 시스템 적용.
4.  **네트워크 보안**: **NetworkPolicy**를 사용하여 Pod 간 통신을 명시적으로 허용/거부합니다.

```yaml
# NetworkPolicy 예시: 'web' Pod으로의 트래픽을 'frontend' 네임스페이스에서만 허용
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-web-from-frontend
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          role: frontend
    ports:
    - protocol: TCP
      port: 80
```

---

## CI/CD와 GitOps

쿠버네티스 환경에서는 애플리케이션 코드와 더불어 인프라 구성(YAML)도 버전 관리되고 자동 배포되어야 합니다.
- **CI (지속적 통합)**: 코드 변경 시 이미지를 빌드하고, 테스트, 보안 스캔, SBOM 생성, 서명을 수행합니다.
- **CD (지속적 배포)**: 쿠버네티스 매니페스트를 관리하고 배포합니다.
    - **Helm**: 패키지 매니저로, 템플릿과 값을 분리하여 환경별 차이를 관리합니다.
    - **Kustomize**: YAML 파일을 패치(Overlay)하는 네이티브 도구입니다.
    - **GitOps**: **Argo CD**나 **Flux**와 같은 도구를 사용하여 Git 저장소에 선언된 상태를 클러스터의 실제 상태와 지속적으로 동기화합니다. 모든 변경 사항은 Pull Request를 통해 Git에 먼저 적용되므로, 감사 추적과 승인 프로세스가 명확해집니다.

---

## 결론: 신뢰할 수 있는 운영 플랫폼으로서의 쿠버네티스

쿠버네티스는 단순히 컨테이너를 실행하는 도구를 넘어, **현대적 애플리케이션을 운영하기 위한 포괄적인 플랫폼**입니다. "선언적 관리"라는 핵심 개념을 통해, 개발자와 운영자는 *원하는 상태*를 YAML로 정의하기만 하면, 복잡한 배포, 확장, 네트워킹, 복구 과정을 쿠버네티스가 자동으로 처리합니다.

이는 단순한 자동화를 넘어 **운영의 예측 가능성과 신뢰성을 근본적으로 높입니다**. 팀은 반복적이고 오류가 발생하기 쉬운 수동 작업에서 벗어나, 더 높은 가치를 창출하는 비즈니스 로직 개발과 시스템 안정성 개선에 집중할 수 있게 됩니다.

물론, 쿠버네티스는 학습 곡선이 가파르고 초기 복잡도가 높습니다. 소규모 서비스라면 관리형 컨테이너 서비스나 서버리스 옵션이 더 적합할 수 있습니다. 그러나 장기적인 관점에서 성장 가능한 아키텍처, 표준화된 운영 프로세스, 그리고 탄력적인 확장성이 필요하다면, 쿠버네티스는 그 기반을 마련하는 데 탁월한 선택입니다.

**다음 단계는 실습입니다.** Minikube나 Kind로 로컬 클러스터를 구성하고, 간단한 웹 애플리케이션을 Deployment와 Service로 배포해 보세요. 그 후, 설정을 ConfigMap으로 분리하고, HPA를 구성하며, GitOps 도구를 도입하는 식으로 점진적으로 쿠버네티스의 강력한 기능을 익혀나가시기 바랍니다.