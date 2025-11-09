---
layout: post
title: Kubernetes - Pod, ReplicaSet, Deployment
date: 2025-04-04 21:20:23 +0900
category: Kubernetes
---
# K8s 핵심 오브젝트 이해하기: Pod, ReplicaSet, Deployment

## 들어가며

- **Pod 심화**: 단일/멀티 컨테이너 패턴(사이드카·어댑터·앰배서더), Init/Ephemeral 컨테이너, 프로브/리소스/보안/스케줄링
- **ReplicaSet 심화**: selector 불변성, 템플릿 해시, 수동 업데이트의 한계
- **Deployment 심화**: 롤링 업데이트 파라미터, Recreate, 블루그린/카나리 구성 패턴, 롤아웃/되돌리기
- **실전 명령/트러블슈팅**: 이벤트/로그 루틴, 흔한 오류 테이블
- **간단 용량·레플리카 산식**: 운영 감각을 돕는 수식

---

## 1) Pod — 쿠버네티스의 최소 실행 단위

### 1.1 개념 확장
- **Pod = 하나 이상의 컨테이너 + 공유 네임스페이스**(네트워크/IPC) + **공유 볼륨**.
- 일반적으로 **1 Pod = 1 컨테이너**가 기본이나, **사이드카 패턴**(로그 수집, 프록시, 설정 리로드 등)으로 다중 컨테이너를 사용.
- **동일 Pod 내 컨테이너는 localhost(127.0.0.1)로 통신** 가능, 스토리지는 동일 볼륨 마운트로 공유.

### 1.2 단일 컨테이너 Pod (기본)
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-pod
  labels: { app: web }
spec:
  containers:
  - name: nginx
    image: nginx:1.27-alpine
    ports: [{ containerPort: 80 }]
    resources:
      requests: { cpu: "100m", memory: "128Mi" }
      limits:   { cpu: "500m", memory: "256Mi" }
    readinessProbe:
      httpGet: { path: "/", port: 80 }
      initialDelaySeconds: 5
      periodSeconds: 5
    livenessProbe:
      httpGet: { path: "/", port: 80 }
      initialDelaySeconds: 15
      periodSeconds: 10
```

### 1.3 다중 컨테이너 Pod(사이드카)
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-with-reloader
  labels: { app: web }
spec:
  volumes:
  - name: conf
    emptyDir: {}
  containers:
  - name: web
    image: nginx:1.27-alpine
    volumeMounts: [{ name: conf, mountPath: /etc/nginx/conf.d }]
  - name: reloader
    image: alpine:3.20
    command: ["/bin/sh","-c"]
    args:
      - |
        while true; do
          # (예시) 외부에서 파일 투입 후 nginx -s reload
          sleep 5
          nginx -s reload || true
        done
    volumeMounts: [{ name: conf, mountPath: /etc/nginx/conf.d }]
```
- **패턴**: 
  - *Sidecar*: 보조 기능(로그/프록시/리로드)
  - *Adapter*: 메트릭/로그 포맷 변환
  - *Ambassador*: 외부 시스템 프록시

### 1.4 Init/Ephemeral 컨테이너
- **InitContainers**: 본 컨테이너 시작 전 1회성 준비(마이그레이션/권한 세팅).
- **Ephemeral Containers**: 실행 중인 Pod에 디버깅 컨테이너를 일시 주입.

```yaml
spec:
  initContainers:
  - name: migrate
    image: bitnami/kubectl
    command: ["sh","-c"]
    args: ["echo 'db migrate here'; sleep 3"]
  containers:
  - name: app
    image: your/app:1.0.0
```

> Ephemeral 컨테이너는 `kubectl debug`로 주입:
```bash
kubectl debug pod/<POD> -it --image=busybox:1.36 --target=app
```

### 1.5 스케줄링·보안·환경설정 핵심
```yaml
spec:
  nodeSelector: { "kubernetes.io/os": linux }
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: "nodepool"
            operator: In
            values: ["frontend"]
  tolerations:
  - key: "workload"
    operator: "Equal"
    value: "batch"
    effect: "NoSchedule"
  securityContext:
    runAsNonRoot: true
    fsGroup: 2000
  serviceAccountName: app-sa
  containers:
  - name: app
    image: your/app:1.0.0
    envFrom:
    - configMapRef: { name: app-config }
    - secretRef:    { name: app-secret }
```

---

## 2) ReplicaSet — Pod “개수”를 보장하는 컨트롤러

### 2.1 개념 확장
- **Replica 개수 유지**가 역할의 전부. 템플릿에 따라 Pod를 생성/추가/복구한다.
- **업데이트 전략은 제공하지 않음** → 사양 변경(이미지/환경) 시 새 RS를 만들어 교체해야 함(수동은 비권장).

### 2.2 예시와 핵심 속성
```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-rs
spec:
  replicas: 3
  selector:               # 필수: Pod 선택 기준(불변)
    matchLabels:
      app: nginx
  template:               # 이 템플릿으로 Pod 생성
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.27-alpine
        ports: [{ containerPort: 80 }]
```
- **selector는 불변**: 생성 후 바꿀 수 없음. 라벨-셀렉터 정합성은 필수.
- 템플릿이 바뀔 경우, RS를 새로 만들거나(수동) → 보통은 **Deployment에 맡김**.

### 2.3 언제 직접 사용할까?
- 교육/실험 목적, 또는 **커스텀 컨트롤러**와의 결합에서 드물게.  
- 실무에선 거의 항상 **Deployment** 사용.

---

## 3) Deployment — 선언적 업데이트의 표준

### 3.1 개념 확장
- **ReplicaSet의 상위 컨트롤러**. RS를 생성·교체하여 **롤링 업데이트/롤백**을 자동화.
- 배포 전략(rolling vs recreate), 진행 상태 감시, 히스토리 관리 등을 제공.

### 3.2 기본 예시
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
spec:
  replicas: 3
  selector:
    matchLabels: { app: nginx }
  template:
    metadata:
      labels: { app: nginx }
    spec:
      containers:
      - name: nginx
        image: nginx:1.27.0
        ports: [{ containerPort: 80 }]
```

### 3.3 롤링 업데이트 파라미터
```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # 추가 생성 허용 개수(또는 %)
      maxUnavailable: 0    # 동시에 내려갈 수 있는 개수(또는 %)
  revisionHistoryLimit: 10
  progressDeadlineSeconds: 600
```
- **maxSurge**↑, **maxUnavailable**↓: 가용성 최우선.  
- **progressDeadlineSeconds**: 지연/교착 시 실패 처리.

### 3.4 Recreate 전략(다운타임 허용)
```yaml
spec:
  strategy:
    type: Recreate
```
- 특정 워크로드(단일 리더·상호 배타 리소스)에 유용.

### 3.5 카나리/블루그린(서비스 분리 패턴)

**카나리(두 Deployment, 라벨로 트래픽 분배):**
```yaml
# svc는 app=web 라벨을 선택한다고 가정
---
apiVersion: apps/v1
kind: Deployment
metadata: { name: web-stable }
spec:
  selector: { matchLabels: { app: web, track: stable } }
  template:
    metadata: { labels: { app: web, track: stable } }
    spec: { containers: [{ name: app, image: your/app:1.0.0 }] }

---
apiVersion: apps/v1
kind: Deployment
metadata: { name: web-canary }
spec:
  replicas: 1
  selector: { matchLabels: { app: web, track: canary } }
  template:
    metadata: { labels: { app: web, track: canary } }
    spec: { containers: [{ name: app, image: your/app:1.1.0 }] }

---
apiVersion: v1
kind: Service
metadata: { name: web }
spec:
  selector: { app: web }            # track은 선택하지 않아 두 트랙 모두 뒤에 둠
  ports: [{ port: 80, targetPort: 8080 }]
```
- **레플리카 수**로 카나리 비율 조정(예: stable 9, canary 1 → 10%).  
- 정교한 가중치는 **Ingress/Gateway/LB**에서 수행.

**블루그린(서비스 라벨 전환):**
- `web-blue`와 `web-green` 두 개를 준비하고, **Service의 selector만** 전환하여 트래픽 스위치.  
- 되돌리기(rollback)가 빠르다.

### 3.6 롤아웃 명령 습관
```bash
kubectl set image deploy/nginx-deploy nginx=nginx:1.27.1
kubectl rollout status deploy/nginx-deploy
kubectl rollout history deploy/nginx-deploy
kubectl rollout undo deploy/nginx-deploy --to-revision=3
```

---

## 4) 세 오브젝트의 관계(복습 도식)
```
[ Deployment ]   --(관리/교체)-->   [ ReplicaSet ]   --(생성/보장)-->   [ Pod ] -> 컨테이너
```
- Deployment = 업데이트/전략/이력
- ReplicaSet = 개수 보장
- Pod = 실행 단위

---

## 5) 실습 — Nginx 배포·노출·업데이트 루틴

### 5.1 배포 + 서비스(NodePort)
```bash
kubectl create deployment nginx --image=nginx:1.27-alpine --replicas=3
kubectl expose deployment nginx --type=NodePort --port=80
kubectl get deploy,rs,pods,svc -o wide
```

### 5.2 Readiness/Liveness 추가(선언형 패치)
```yaml
# patch-nginx-probes.yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: nginx }
spec:
  template:
    spec:
      containers:
      - name: nginx
        image: nginx:1.27-alpine
        readinessProbe:
          httpGet: { path: /, port: 80 }
          initialDelaySeconds: 5
        livenessProbe:
          httpGet: { path: /, port: 80 }
          initialDelaySeconds: 15
```
```bash
kubectl apply -f patch-nginx-probes.yaml
kubectl rollout status deploy/nginx
```

### 5.3 이미지 업데이트/되돌리기
```bash
kubectl set image deploy/nginx nginx=nginx:1.27.2
kubectl rollout status deploy/nginx
kubectl rollout history deploy/nginx
kubectl rollout undo deploy/nginx
```

---

## 6) 운영 팁 — 라벨·셀렉터·리소스·스케줄링 체크리스트

- **라벨/셀렉터 정합성**: `kubectl get ep <svc>`로 Service↔Pod 연결 검증.
- **리소스 request/limit**: HPA/VPA와 연동, 과대요청은 비용 상승·밀도 저하.
- **스케줄링 제약**: nodeAffinity/taints/tolerations 설정 시 Pending 방지.
- **프로브**: readiness=트래픽 참여, liveness=자가회복; 시작 지연은 `startupProbe`.
- **보안**: PodSecurity(PSA) 레벨, SecurityContext(runAsNonRoot, drop CAPs), 이미지 서명.

---

## 7) 디버깅 루틴(명령 스니펫)
```bash
kubectl get pods -o wide
kubectl describe pod <POD>
kubectl logs -f deploy/<DEPLOY>                 # 디플로이먼트 전체 로그
kubectl get events --sort-by=.lastTimestamp
kubectl exec -it pod/<POD> -- /bin/sh
kubectl top pods -A
# 서비스 연결 검증
kubectl get ep <SVC>
kubectl run -it --rm netshoot --image=nicolaka/netshoot -- /bin/bash
# 내부에서:
curl -I http://<SVC_NAME>.<NS>.svc.cluster.local
```

---

## 8) 흔한 문제와 해결

| 증상 | 1차 확인 | 원인 후보 | 해결 |
|---|---|---|---|
| Pod Pending | `describe pod` 이벤트 | 자원 부족, 노드 taint, affinity 과도 | 요청/제약 조정, 노드풀 확장, tolerations |
| CrashLoopBackOff | `logs -p`, 이벤트 | 잘못된 env/secret, init 실패 | 시동 스크립트/의존성 수정, startupProbe |
| Readiness 실패 | `describe pod` | 프로브 경로/포트 불일치 | 프로브 교정, 초기 지연 증가 |
| Service 접속 불가 | `kubectl get ep <svc>` | selector-라벨 mismatch | 라벨/셀렉터 재정합 |
| 롤링 지연 | `rollout status`, 이벤트 | 프로브 실패/리소스 부족 | maxSurge/Unavailable 조정, 리소스/프로브 교정 |
| Undo 불가 | history 0/재배포 | 이력 보존 미설정 | `revisionHistoryLimit` 충분히 설정 |

---

## 9) 간단 용량·레플리카 산정(학습용)
요청률(QPS)과 평균 처리시간 \(t\) 초, 파드당 안정 처리량 \(\text{cap}_{pod}\) (동시 처리 가능 요청 수)가 있을 때 필요한 **최소 레플리카**는
$$
\text{replicas}_{\min} \approx 
\left\lceil \frac{\text{QPS}\cdot t}{\text{cap}_{pod}} \right\rceil \cdot \text{버퍼}
$$
- **버퍼**: 배포 중 이격, 장애/스파이크 대비(예: 1.3~2.0).  
- HPA로 실제 트래픽에 따라 자동 조정하되, 초기 값은 위 산식으로 추정.

---

## 10) 요약 — 언제 무엇을 쓸까?
| 목적 | 오브젝트 | 설명/메모 |
|---|---|---|
| 임시 실습·단발 테스트 | **Pod** | 직접 생성 가능하나 운영 비권장 |
| 동일 Pod 수 보장 | **ReplicaSet** | 개수 유지만 담당(업데이트는 수동) |
| 배포/업데이트/롤백 표준 | **Deployment** | 실무 기본. 롤링/블루그린/카나리 |

**핵심**: 운영에서는 **Deployment**를 중심으로 생각하고, RS는 내부 구현 디테일로, Pod는 실행 단위로 바라보자.  
프로브/리소스/스케줄링/보안을 함께 설계하면 **안정적 롤아웃과 빠른 복구**가 가능하다.