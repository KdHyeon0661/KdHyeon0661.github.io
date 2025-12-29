---
layout: post
title: Kubernetes - Pod, ReplicaSet, Deployment
date: 2025-04-04 21:20:23 +0900
category: Kubernetes
---
# Kubernetes 핵심 오브젝트 이해하기: Pod, ReplicaSet, Deployment

## 서론: Kubernetes 워크로드 관리의 기초

Kubernetes에서 애플리케이션을 배포하고 관리하기 위한 핵심 오브젝트인 Pod, ReplicaSet, Deployment를 이해하는 것은 효과적인 컨테이너 오케스트레이션의 시작점입니다. 이 세 가지 오브젝트는 계층적인 관계를 가지며, 각각 특정한 책임과 역할을 담당합니다. 이 가이드는 각 오브젝트의 심층적인 이해부터 실무 적용 패턴까지 체계적으로 설명합니다.

## Pod: Kubernetes의 기본 실행 단위

### Pod의 본질

Pod는 Kubernetes에서 실행되는 가장 작은 배포 가능 단위로, 하나 이상의 컨테이너와 공유된 네트워크 네임스페이스, IPC(프로세스 간 통신) 공간, 그리고 공유된 스토리지 볼륨을 포함합니다. Pod 내부의 컨테이너들은 항상 함께 스케줄링되고, 동일한 노드에서 실행되며, 생명주기를 공유합니다.

**핵심 개념**: 대부분의 경우 하나의 Pod에 하나의 컨테이너가 포함되지만, 밀접하게 결합된 컨테이너들을 함께 실행해야 하는 경우 다중 컨테이너 Pod 패턴을 사용할 수 있습니다.

### 단일 컨테이너 Pod: 기본 패턴

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
  labels:
    app: web
    environment: production
spec:
  containers:
  - name: web-server
    image: nginx:1.27-alpine
    ports:
    - containerPort: 80
    resources:
      requests:
        cpu: "100m"
        memory: "128Mi"
      limits:
        cpu: "500m"
        memory: "256Mi"
    env:
    - name: ENVIRONMENT
      value: "production"
    - name: LOG_LEVEL
      value: "info"
    readinessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 5
      failureThreshold: 3
    livenessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 15
      periodSeconds: 10
      failureThreshold: 3
```

### 다중 컨테이너 Pod: 사이드카 패턴

다중 컨테이너 Pod는 여러 컨테이너가 동일한 네트워크 네임스페이스와 스토리지 볼륨을 공유하는 패턴입니다. 이 패턴은 특정한 사용 사례에 유용합니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-with-logger
  labels:
    app: web
spec:
  volumes:
  - name: shared-logs
    emptyDir: {}
  
  containers:
  # 메인 애플리케이션 컨테이너
  - name: web-app
    image: nginx:1.27-alpine
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log/nginx
    
  # 사이드카 컨테이너: 로그 수집기
  - name: log-collector
    image: fluent/fluentd:latest
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log/nginx
    env:
    - name: FLUENTD_CONF
      value: "fluent.conf"
```

### 다중 컨테이너 패턴 유형

1. **사이드카(Sidecar) 패턴**: 주 애플리케이션을 보조하는 기능을 제공 (로그 수집, 모니터링 에이전트, 설정 리로더 등)
2. **어댑터(Adapter) 패턴**: 애플리케이션의 출력을 표준 형식으로 변환 (로그 포맷 변환, 메트릭 정규화 등)
3. **앰배서더(Ambassador) 패턴**: 외부 서비스에 대한 프록시 역할 (데이터베이스 연결, API 게이트웨이 등)

### 초기화 및 임시 컨테이너

**Init 컨테이너**는 메인 컨테이너가 시작되기 전에 한 번 실행되는 컨테이너로, 데이터베이스 마이그레이션, 권한 설정, 의존성 다운로드 등 초기화 작업에 사용됩니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-init
spec:
  initContainers:
  - name: init-migrate
    image: postgres:15-alpine
    command: ['sh', '-c', 'echo "Running database migrations..." && sleep 5']
    env:
    - name: PGHOST
      value: "postgres-service"
  
  containers:
  - name: main-app
    image: myapp:1.0.0
    ports:
    - containerPort: 8080
```

**임시(Ephemeral) 컨테이너**는 실행 중인 Pod에 일시적으로 추가하여 디버깅이나 문제 해결을 수행할 수 있게 합니다:

```bash
# 실행 중인 Pod에 디버깅 컨테이너 추가
kubectl debug pod/web-pod -it --image=busybox:1.36 --target=web-app
```

### 고급 Pod 구성 요소

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: advanced-pod
spec:
  # 노드 선택 및 선호도
  nodeSelector:
    kubernetes.io/os: linux
    node-type: high-memory
  
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: node-pool
            operator: In
            values: ["frontend"]
    
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: app
              operator: In
              values: ["web"]
          topologyKey: kubernetes.io/hostname
  
  # 테인트(Taint) 허용
  tolerations:
  - key: "workload-type"
    operator: "Equal"
    value: "batch-job"
    effect: "NoSchedule"
  
  # 보안 컨텍스트
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000
    seccompProfile:
      type: RuntimeDefault
  
  # 서비스 계정
  serviceAccountName: app-service-account
  
  containers:
  - name: app
    image: myapp:latest
    # 환경 변수 구성
    envFrom:
    - configMapRef:
        name: app-config
    - secretRef:
        name: app-secrets
    
    # 리소스 제한
    resources:
      requests:
        cpu: "200m"
        memory: "256Mi"
      limits:
        cpu: "500m"
        memory: "512Mi"
```

---

## ReplicaSet: 복제본 관리 컨트롤러

### ReplicaSet의 역할

ReplicaSet은 지정된 수의 동일한 Pod 복제본이 항상 실행되도록 보장하는 Kubernetes 컨트롤러입니다. Pod의 수명 주기를 관리하며, 노드 장애나 Pod 실패 시 자동으로 새로운 Pod를 생성합니다.

### ReplicaSet 구성 예제

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: web-replicaset
  labels:
    app: web
    tier: frontend
spec:
  # 유지해야 할 Pod 복제본 수
  replicas: 3
  
  # Pod 선택 기준 (변경 불가)
  selector:
    matchLabels:
      app: web
      tier: frontend
  
  # Pod 템플릿
  template:
    metadata:
      labels:
        app: web
        tier: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:1.27-alpine
        ports:
        - containerPort: 80
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
```

### ReplicaSet의 주요 특성

1. **셀렉터 불변성**: ReplicaSet 생성 후 셀렉터를 변경할 수 없습니다. 이는 의도치 않은 Pod 선택을 방지합니다.
2. **템플릿 해시**: ReplicaSet은 Pod 템플릿의 해시를 계산하여 생성된 Pod를 추적합니다.
3. **수동 업데이트의 한계**: 이미지 버전이나 환경 변수를 변경하려면 새 ReplicaSet을 생성해야 합니다.

### ReplicaSet의 실제 사용 시나리오

ReplicaSet은 일반적으로 직접 사용되지 않고 Deployment에 의해 관리됩니다. 그러나 다음과 같은 경우 직접 사용할 수 있습니다:
- 교육 및 실험 목적
- 커스텀 컨트롤러와의 통합
- 특정한 수명 주기 관리 요구사항이 있는 경우

---

## Deployment: 선언적 업데이트 관리자

### Deployment의 중요성

Deployment는 ReplicaSet의 상위 추상화로, 애플리케이션의 선언적 업데이트, 롤링 업데이트, 롤백 기능을 제공합니다. 실무에서 가장 널리 사용되는 워크로드 관리 오브젝트입니다.

### 기본 Deployment 구성

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-deployment
  labels:
    app: web
    environment: production
spec:
  # 원하는 Pod 복제본 수
  replicas: 3
  
  # Pod 선택기
  selector:
    matchLabels:
      app: web
  
  # 배포 전략
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1           # 추가로 생성할 수 있는 Pod 수 (또는 백분율)
      maxUnavailable: 0     # 업데이트 중 사용 불가능할 수 있는 Pod 수
  
  # 리비전 이력 제한
  revisionHistoryLimit: 10
  
  # 진행 데드라인 (초)
  progressDeadlineSeconds: 600
  
  # Pod 템플릿
  template:
    metadata:
      labels:
        app: web
        environment: production
    spec:
      containers:
      - name: web-server
        image: nginx:1.27.0
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "200m"
            memory: "256Mi"
```

### 배포 전략 상세

#### 1. 롤링 업데이트 (기본 전략)
롤링 업데이트는 새로운 Pod를 점진적으로 생성하고 오래된 Pod를 제거하는 방식으로, 서비스 중단 없이 업데이트를 수행합니다.

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%        # 최대 25% 추가 Pod 생성 가능
      maxUnavailable: 25%  # 최대 25% Pod가 동시에 사용 불가능할 수 있음
```

#### 2. 재생성(Recreate) 전략
모든 기존 Pod를 먼저 종료한 후 새로운 Pod를 생성하는 방식으로, 짧은 다운타임이 발생하지만 단순한 업데이트가 가능합니다.

```yaml
spec:
  strategy:
    type: Recreate
```

### 고급 배포 패턴

#### 블루-그린 배포
두 개의 완전한 환경(블루와 그린)을 유지하면서 트래픽을 한 번에 전환하는 방식입니다.

```yaml
# 블루 환경 (현재 프로덕션)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
      version: blue
  template:
    metadata:
      labels:
        app: web
        version: blue
    spec:
      containers:
      - name: web
        image: myapp:v1.0.0
---
# 그린 환경 (새 버전)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
      version: green
  template:
    metadata:
      labels:
        app: web
        version: green
    spec:
      containers:
      - name: web
        image: myapp:v1.1.0
---
# 서비스 (라벨 선택기로 트래픽 전환)
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: web
    version: blue  # 블루에서 그린으로 변경하여 트래픽 전환
  ports:
  - port: 80
    targetPort: 8080
```

#### 카나리 배포
소수의 사용자에게 새 버전을 먼저 배포하고 점진적으로 확대하는 방식입니다.

```yaml
# 안정 버전 (대부분의 트래픽)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-stable
spec:
  replicas: 9
  selector:
    matchLabels:
      app: web
      track: stable
  template:
    metadata:
      labels:
        app: web
        track: stable
    spec:
      containers:
      - name: web
        image: myapp:v1.0.0
---
# 카나리 버전 (일부 트래픽)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-canary
spec:
  replicas: 1  # 전체 트래픽의 10%
  selector:
    matchLabels:
      app: web
      track: canary
  template:
    metadata:
      labels:
        app: web
        track: canary
    spec:
      containers:
      - name: web
        image: myapp:v1.1.0
---
# 서비스 (두 트랙 모두 선택)
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: web  # stable과 canary 모두 선택
  ports:
  - port: 80
    targetPort: 8080
```

### Deployment 관리 명령어

```bash
# 이미지 업데이트
kubectl set image deployment/web-deployment web-server=nginx:1.27.1

# 롤아웃 상태 확인
kubectl rollout status deployment/web-deployment

# 배포 이력 확인
kubectl rollout history deployment/web-deployment

# 특정 리비전으로 롤백
kubectl rollout undo deployment/web-deployment --to-revision=2

# 배포 일시 중지
kubectl rollout pause deployment/web-deployment

# 배포 재개
kubectl rollout resume deployment/web-deployment

# 배포 확장/축소
kubectl scale deployment/web-deployment --replicas=5
```

---

## 세 오브젝트의 관계 이해

### 계층적 구조
```
Deployment (배포 관리)
    │
    └── ReplicaSet (복제본 관리)
            │
            └── Pod (실행 단위)
                    │
                    └── Container(s) (애플리케이션)
```

### 각 오브젝트의 책임

1. **Deployment**: 배포 전략, 롤링 업데이트, 롤백, 배포 이력 관리
2. **ReplicaSet**: 지정된 수의 Pod 복제본 유지, Pod 생성/삭제 관리
3. **Pod**: 컨테이너 실행 환경 제공, 공유 네트워크/스토리지 관리

### 실무에서의 관계

일반적으로 개발자는 Deployment를 직접 관리하고, ReplicaSet은 Deployment에 의해 자동으로 관리됩니다. Pod는 ReplicaSet에 의해 생성되고 관리됩니다.

---

## 종합 실습: 전체 워크플로우

### 1. 기본 배포 생성
```bash
# Deployment 생성
kubectl create deployment nginx --image=nginx:1.27-alpine --replicas=3

# 상태 확인
kubectl get deployments
kubectl get replicasets
kubectl get pods

# 서비스 노출
kubectl expose deployment nginx --type=NodePort --port=80

# 서비스 확인
kubectl get services
```

### 2. 헬스 체크 추가
```yaml
# nginx-with-probes.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
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
          periodSeconds: 10
```

```bash
# 배포 적용
kubectl apply -f nginx-with-probes.yaml

# 롤아웃 상태 확인
kubectl rollout status deployment/nginx
```

### 3. 애플리케이션 업데이트
```bash
# 이미지 업데이트
kubectl set image deployment/nginx nginx=nginx:1.27.1

# 롤아웃 진행 상황 확인
kubectl rollout status deployment/nginx

# 배포 이력 확인
kubectl rollout history deployment/nginx

# 문제 발생 시 롤백
kubectl rollout undo deployment/nginx

# 특정 리비전으로 롤백
kubectl rollout undo deployment/nginx --to-revision=1
```

---

## 운영 모범 사례

### 라벨과 셀렉터 관리
- **일관된 라벨링 전략** 수립: `app`, `version`, `environment`, `tier` 등 표준 라벨 사용
- **셀렉터 정합성 검증**: 서비스와 Pod의 라벨이 일치하는지 정기적으로 확인
```bash
# 서비스와 연결된 엔드포인트 확인
kubectl get endpoints <service-name>
```

### 리소스 관리
- **요청(Request)과 제한(Limit) 설정**: 메모리 누수와 CPU 스로틀링 방지
- **적절한 복제본 수 설정**: 가용성과 비용 간의 균형 유지
- **HPA(Horizontal Pod Autoscaler) 활용**: 부하에 따른 자동 확장/축소

### 스케줄링 최적화
- **노드 선택기(NodeSelector)**: 특정 노드 풀에 워크로드 배치
- **어피니티(Affinity)/안티어피니티(Anti-Affinity)**: 워크로드 분산 또는 집중
- **테인트(Taints)와 톨러레이션(Tolerations)**: 특수 노드용 워크로드 관리

### 헬스 체크 구성
- **Readiness Probe**: 트래픽 수신 준비 상태 확인
- **Liveness Probe**: 애플리케이션 정상 실행 상태 확인
- **Startup Probe**: 느린 시작 애플리케이션을 위한 초기화 시간 허용

### 보안 강화
- **Pod 보안 표준(Pod Security Standards)**: 기본, 베이직, 제한된 정책 적용
- **보안 컨텍스트(SecurityContext)**: 비루트 실행, 파일 시스템 권한 제한
- **이미지 서명 및 검증**: 신뢰할 수 있는 이미지만 사용

---

## 문제 해결 가이드

### 일반적인 문제와 해결 방법

**문제 1: Pod가 Pending 상태로 유지됨**
```bash
# Pod 상세 정보 확인
kubectl describe pod <pod-name>

# 일반적인 원인과 해결책:
# 1. 리소스 부족: 요청(Request) 값 감소 또는 노드 추가
# 2. 노드 테인트(Taint): 톨러레이션(Toleration) 추가
# 3. 노드 선택기(NodeSelector) 불일치: 라벨 수정
# 4. PVC 바인딩 실패: 스토리지 클래스 확인
```

**문제 2: Pod가 CrashLoopBackOff 상태**
```bash
# 이전 컨테이너 로그 확인
kubectl logs <pod-name> --previous

# Pod 이벤트 확인
kubectl describe pod <pod-name> | grep -A 10 Events

# 일반적인 원인:
# 1. 잘못된 환경 변수 또는 시크릿
# 2. Init 컨테이너 실패
# 3. 컨테이너 시작 스크립트 오류
# 4. 의존성 서비스 연결 실패
```

**문제 3: Readiness Probe 실패**
```bash
# Pod 상태 확인
kubectl get pod <pod-name> -o yaml | grep -A 20 readinessProbe

# 일반적인 해결책:
# 1. 프로브 경로/포트 수정
# 2. 초기 지연 시간(initialDelaySeconds) 증가
# 3. 프로브 간격(periodSeconds) 조정
# 4. 애플리케이션 헬스 엔드포인트 구현
```

**문제 4: 서비스 연결 실패**
```bash
# 서비스와 엔드포인트 확인
kubectl get service <service-name>
kubectl get endpoints <service-name>

# 일반적인 원인:
# 1. 서비스 셀렉터와 Pod 라벨 불일치
# 2. Pod의 readinessProbe 실패
# 3. 네트워크 정책 차단
```

**문제 5: 롤링 업데이트 지연 또는 실패**
```bash
# 롤아웃 상태 상세 확인
kubectl rollout status deployment/<deployment-name>

# 배포 이벤트 확인
kubectl describe deployment/<deployment-name>

# 일반적인 해결책:
# 1. maxSurge/maxUnavailable 값 조정
# 2. readinessProbe 설정 최적화
# 3. 리소스 요청 증가
```

### 디버깅 명령어 모음

```bash
# 기본 상태 확인
kubectl get pods,replicasets,deployments -o wide

# 상세 정보 확인
kubectl describe pod <pod-name>
kubectl describe replicaset <rs-name>
kubectl describe deployment <deployment-name>

# 로그 확인
kubectl logs <pod-name>
kubectl logs -f deployment/<deployment-name>  # 실시간 로그
kubectl logs --previous <pod-name>  # 이전 컨테이너 로그

# 이벤트 확인
kubectl get events --sort-by=.lastTimestamp
kubectl get events --field-selector involvedObject.name=<pod-name>

# 실행 중인 Pod에 접속
kubectl exec -it <pod-name> -- /bin/sh

# 네트워크 연결 테스트
kubectl run -it --rm network-test --image=nicolaka/netshoot -- /bin/bash
# 컨테이너 내부에서:
# curl -I http://<service-name>.<namespace>.svc.cluster.local
# nslookup <service-name>

# 리소스 사용량 확인
kubectl top pods
kubectl top nodes
```

---

## 용량 계획과 복제본 수 결정

효과적인 Kubernetes 운영을 위해서는 적절한 복제본 수 결정이 중요합니다. 다음은 기본적인 계산 방식을 제공합니다:

### 기본 계산 공식

```
필요한 최소 복제본 수 ≈ (초당 요청 수 × 평균 응답 시간(초)) / (파드당 처리 능력) × 버퍼 계수
```

**변수 설명**:
- **초당 요청 수(QPS)**: 애플리케이션이 처리해야 하는 초당 요청 수
- **평균 응답 시간**: 요청당 평균 처리 시간(초)
- **파드당 처리 능력**: 단일 파드가 안정적으로 처리할 수 있는 동시 요청 수
- **버퍼 계수**: 피크 트래픽과 장애 대비를 위한 여유 계수(일반적으로 1.3-2.0)

### 실제 예시

예를 들어, 웹 애플리케이션이 다음과 같은 특성을 가진다고 가정합니다:
- 초당 요청 수: 1000
- 평균 응답 시간: 0.1초
- 파드당 처리 능력: 50 동시 요청
- 버퍼 계수: 1.5

```
필요한 복제본 수 = (1000 × 0.1) / 50 × 1.5 = 3
```

### 운영적 고려사항

1. **최소 복제본 수**: 단일 장애점(SPOF) 방지를 위해 최소 2개 이상 유지
2. **수직적 확장(Vertical Scaling) vs 수평적 확장(Horizontal Scaling)**: 리소스 증가 vs 복제본 증가
3. **HPA(Horizontal Pod Autoscaler) 활용**: CPU, 메모리, 커스텀 메트릭 기반 자동 확장
4. **클러스터 오토스케일러(Cluster Autoscaler)**: 노드 수 자동 조정

---

## 결론

Pod, ReplicaSet, Deployment는 Kubernetes 워크로드 관리의 핵심 삼각형을 형성합니다. 각 오브젝트의 역할과 상호작용을 이해하는 것은 효과적인 Kubernetes 운영의 기초입니다.

### 핵심 요점 정리

1. **Pod는 실행 단위**: 하나 이상의 컨테이너와 공유 리소스를 포함하는 기본 배포 단위
2. **ReplicaSet은 복제본 관리자**: 지정된 수의 동일한 Pod가 항상 실행되도록 보장
3. **Deployment는 배포 관리자**: 롤링 업데이트, 롤백, 배포 전략을 제공하는 고수준 추상화

### 실무 적용 권장사항

1. **기본 워크로드 관리에는 Deployment 사용**: 대부분의 경우 직접 Pod나 ReplicaSet을 관리하지 않고 Deployment를 사용
2. **적절한 헬스 체크 구현**: Readiness, Liveness, Startup 프로브를 상황에 맞게 구성
3. **리소스 요청과 제한 설정**: 메모리 누수와 CPU 경합 방지
4. **배포 전략 수립**: 애플리케이션 특성에 맞는 롤링 업데이트, 블루-그린, 카나리 배포 전략 선택
5. **모니터링과 관측 강화**: 로그, 메트릭, 트레이싱을 통한 가시성 확보

### 다음 단계

이 기본 오브젝트들을 숙달한 후에는 다음 단계로 나아가시기를 권장합니다:
1. **StatefulSet과 DaemonSet 학습**: 상태 유지 워크로드와 노드별 배포 패턴
2. **서비스 메시(Service Mesh) 도입**: Istio, Linkerd 등을 통한 고급 네트워킹
3. **GitOps 워크플로우 구현**: Argo CD, Flux를 통한 선언적 배포 자동화
4. **고급 스케줄링 전략**: Pod 디스럽션 버짓, 토폴로지 스프레드 제약 활용

Kubernetes 워크로드 관리는 지속적인 학습과 실험을 통해 발전시켜 나가는 과정입니다. 기본 개념을 탄탄히 다진 후, 조직과 애플리케이션의 특수한 요구사항에 맞게 최적화해 나가시기 바랍니다.