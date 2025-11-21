---
layout: post
title: Kubernetes - Nginx 배포
date: 2025-04-15 20:20:23 +0900
category: Kubernetes
---
# Kubernetes 예제: 간단한 Nginx 배포

## 목표

- Nginx를 **Deployment**로 배포하여 **Pod를 관리**
- Nginx를 **Service(NodePort)**로 외부에 노출
- `kubectl` 명령어로 상태 확인 및 테스트
- (+) **프로브/리소스 한도/보안 컨텍스트/롤링 업데이트** 반영
- (+) **ConfigMap**으로 정적 파일 주입, **Ingress**로 도메인 노출
- (+) **HPA**로 오토스케일, **문제 해결 루틴**까지 학습

---

## 사전 준비

- 클러스터: Minikube/Kind/EKS/GKE/AKS 등 (실습은 Minikube/Kind 가볍게 권장)
- `kubectl` 연결 확인:
```bash
kubectl version --client
kubectl cluster-info
kubectl get nodes -o wide
```

---

## Nginx Deployment 정의(기본형)

### `nginx-deployment.yaml` (가장 단순한 형태)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 2
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

### 설명

| 항목 | 설명 |
|------|------|
| `replicas: 2` | Pod 2개 실행(가용성↑, 롤링 업데이트 안전) |
| `selector.matchLabels` | 관리 대상 Pod 라벨 지정(불변 필드, 템플릿 라벨과 **완전 일치**) |
| `image: nginx:latest` | 최초엔 편하지만, **실전은 명시 버전**(예: `nginx:1.27-alpine`) 권장 |
| `containerPort: 80` | Pod 내부 노출 포트 |

> 실전 팁: `latest`는 재현성/보안 스캔에서 불리하다. **정확 버전 태그** 사용 권장.

---

## Service 정의 (NodePort)

### `nginx-service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
```

### 설명

| 항목 | 설명 |
|------|------|
| `type: NodePort` | 모든 노드의 `nodePort`로 외부 접근 허용 |
| `selector.app=nginx` | 라벨 매칭되는 Pod로 트래픽 전달 |
| `port: 80` | Service 관점의 포트 |
| `targetPort: 80` | 컨테이너 실제 포트 |
| `nodePort: 30080` | 외부 접근 포트(30000~32767) |

> Minikube 기준 `minikube service nginx-service`가 가장 간단.
> 클라우드/온프렘이면 **노드 IP** + `nodePort`로 접근.

---

## 리소스 배포 & 확인

```bash
kubectl apply -f nginx-deployment.yaml
kubectl apply -f nginx-service.yaml
```

상태 확인:
```bash
kubectl get deployments
kubectl get pods -o wide
kubectl get svc
kubectl describe deployment nginx-deployment
kubectl get endpoints nginx-service -o wide
```

예시 출력:
```
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   2/2     2            2           1m

NAME             TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
nginx-service    NodePort   10.96.22.121   <none>        80:30080/TCP   1m
```

---

## 외부에서 접근 테스트

### Minikube

```bash
minikube service nginx-service
```
→ 브라우저에서 Nginx 환영 페이지가 열린다.

### NodePort 직접 접근

```bash
# 조회

kubectl get nodes -o wide
minikube ip   # Minikube일 때

# 접근

curl -I http://<노드 IP>:30080
```

---

## “운영형”으로 보강한 Deployment

기본형은 학습용으로 충분하지만, 운영에선 다음을 추가해야 한다:
- **readinessProbe / livenessProbe**: 정상 판정·자가 치유
- **resources**: 요청/상한으로 스케줄링·격리 안정화
- **securityContext**: 루트 회피 등 최소 권한
- **imagePullPolicy**: 버전에 맞는 정책
- **롤링 업데이트 파라미터**: 무중단 배포

### 📄 `nginx-deployment.ops.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app.kubernetes.io/name: nginx
    env: dev
spec:
  replicas: 2
  revisionHistoryLimit: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # 가용성 우선
      maxUnavailable: 0
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
        env: dev
    spec:
      securityContext:
        runAsNonRoot: true
        seccompProfile: { type: RuntimeDefault }
      containers:
      - name: nginx
        image: nginx:1.27-alpine
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 256Mi
        readinessProbe:
          httpGet: { path: "/", port: 80 }
          initialDelaySeconds: 3
          periodSeconds: 5
          timeoutSeconds: 2
        livenessProbe:
          httpGet: { path: "/", port: 80 }
          initialDelaySeconds: 15
          periodSeconds: 10
          timeoutSeconds: 2
```

적용:
```bash
kubectl apply -f nginx-deployment.ops.yaml
kubectl rollout status deploy/nginx-deployment
kubectl get pods -o wide
```

이미지 업데이트/롤백:
```bash
kubectl set image deploy/nginx-deployment nginx=nginx:1.27.1-alpine
kubectl rollout status deploy/nginx-deployment
kubectl rollout history deploy/nginx-deployment
kubectl rollout undo deploy/nginx-deployment
```

---

## ConfigMap로 컨텐츠/설정 주입

실전에서 Nginx를 “빈 환영 페이지” 대신 **우리 콘텐츠**로 바꾸려면 `ConfigMap`을 **파일 마운트** 한다.

### 정적 파일 ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-static
data:
  index.html: |
    <!DOCTYPE html>
    <html>
    <head><meta charset="utf-8"><title>Nginx on K8s</title></head>
    <body>
      <h1>배포 성공 🎉</h1>
      <p>이 페이지는 ConfigMap에서 주입되었습니다.</p>
    </body>
    </html>
  default.conf: |
    server {
      listen 80;
      server_name _;
      root /usr/share/nginx/html;
      index index.html;
    }
```

### Nginx에 마운트

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 2
  selector: { matchLabels: { app: nginx } }
  template:
    metadata:
      labels: { app: nginx }
    spec:
      containers:
      - name: nginx
        image: nginx:1.27-alpine
        volumeMounts:
        - name: web
          mountPath: /usr/share/nginx/html/index.html
          subPath: index.html
        - name: conf
          mountPath: /etc/nginx/conf.d/default.conf
          subPath: default.conf
        ports: [{ containerPort: 80 }]
      volumes:
      - name: web
        configMap:
          name: nginx-static
          items:
          - key: index.html
            path: index.html
      - name: conf
        configMap:
          name: nginx-static
          items:
          - key: default.conf
            path: default.conf
```

적용 후 검증:
```bash
kubectl rollout restart deploy/nginx-deployment
kubectl get pods -o wide
curl http://<노드 IP>:30080 | head -n 5
```

> **subPath**는 파일 단건 마운트에 편리하지만, 변경 감지가 제한될 수 있다.
> 빈번 변경/핫리로드는 **디렉터리 마운트** + Nginx `-s reload`(사이드카) 패턴 고려.

---

## Ingress로 도메인 노출(선택)

NodePort 대신 **Ingress Controller**(예: NGINX Ingress, Traefik)를 쓰면, 도메인/경로 라우팅·TLS 등 **L7 기능**을 쉽게 적용할 수 있다.
Minikube:
```bash
minikube addons enable ingress
```

### `nginx-ingress.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx   # 배포 환경에 따라 'nginx' 또는 기본 클래스명 확인
  rules:
  - host: example.local     # /etc/hosts에 minikube ip 매핑해서 사용 가능
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-service
            port:
              number: 80
```

적용 후:
```bash
kubectl apply -f nginx-ingress.yaml
kubectl get ingress
# hosts 파일에 <minikube ip> example.local 등록 후:

curl -I http://example.local
```

---

## ✅ 8. HPA로 자동 확장(선택)

**metrics-server**가 필요하다(Minikube 애드온 지원).

```bash
minikube addons enable metrics-server
kubectl top nodes
kubectl top pods
```

### HPA 적용

```bash
kubectl autoscale deployment nginx-deployment --cpu-percent=50 --min=2 --max=10
kubectl get hpa -w
```

부하 생성 예시:
```bash
# 임시 부하 Pod 생성

kubectl run hey --image=rakyll/hey --restart=Never -- \
  -z 5s -c 50 -q 10 http://nginx-service.default.svc.cluster.local
kubectl get hpa -w
```

> 실전은 **리소스 요청(requests)** 가 정확해야 CPU% 기준이 신뢰 가능.

---

## 관측 & 로그 & 디버깅 루틴

```bash
# 이벤트/상태 확인

kubectl get events --sort-by=.lastTimestamp | tail
kubectl describe deploy nginx-deployment
kubectl describe svc nginx-service

# 엔드포인트 집합 확인(연결 핵심)

kubectl get endpoints nginx-service -o yaml

# Pod 로그/쉘

kubectl logs -l app=nginx --tail=100
kubectl exec -it deploy/nginx-deployment -- sh -c 'ls -l /usr/share/nginx/html && nginx -T | head -n 40'
```

---

## 트러블슈팅 표

| 증상 | 1차 확인 | 원인 | 해결 |
|---|---|---|---|
| `curl` 30080 접속 실패 | `kubectl get svc` `get ep` | 엔드포인트 없음(라벨 불일치), 노드 보안그룹 | 라벨/셀렉터 정합, 보안그룹/방화벽 포트 개방 |
| Pod가 `CrashLoopBackOff` | `kubectl logs` | 설정 파일 경로/권한, 포트 충돌 | ConfigMap 경로/권한 재확인, 포트 수정 |
| `READY 0/1` 지속 | `describe pod` 프로브 이벤트 | 프로브 경로 불일치/지연 | readiness/liveness 경로/지연 조정 |
| 롤링 멈춤 | `rollout status` | 새 Pod Ready 실패 | `maxSurge/unavailable`·프로브·리소스 조정 |
| Ingress 404/502 | `kubectl get ingress`/컨트롤러 로그 | IngressClass/Service 포트 잘못 | IngressClass·Service 이름/포트 확인 |
| HPA 미동작 | `kubectl top` | metrics-server 미설치/권한 | 애드온/권한 확인, requests 설정 |

---

## YAML 하나로 묶기(기본형 유지)

### `nginx-full.yaml` — Deployment + Service

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels: { app: nginx }
    spec:
      containers:
      - name: nginx
        image: nginx:1.27-alpine
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
```

적용:
```bash
kubectl apply -f nginx-full.yaml
kubectl get all -l app=nginx
```

---

## 리소스 정리

```bash
kubectl delete -f nginx-full.yaml
# 또는 개별 삭제

kubectl delete deployment nginx-deployment
kubectl delete service nginx-service
# Ingress/HPA/ConfigMap 등을 추가했다면 따로 정리

kubectl delete ingress nginx-ingress || true
kubectl delete hpa nginx-deployment || true
kubectl delete configmap nginx-static || true
```

---

## 실전 체크리스트(요약)

- **버전 고정**: `nginx:<정확 태그>` (latest 지양)
- **프로브 설정**: readiness/liveness(필수), startupProbe(초기 무거운 앱)
- **리소스**: requests/limits 지정 → 스케줄 안정성/HPA 근거
- **라벨/셀렉터**: Service ↔ Pod 라벨 정합성 **항상 검증**(`kubectl get ep`)
- **ConfigMap/Secret**: 설정/자격증명 파일 마운트 패턴 숙지
- **Ingress**: NodePort 대신 L7로 운영(도메인/TLS/경로 분기)
- **HPA**: metrics-server + 정확한 requests
- **롤링 파라미터**: 가용성 요구에 맞게 `maxSurge/maxUnavailable` 설계
- **보안**: `runAsNonRoot`, `seccomp`, image 서명/스캔, `PSA` 정책

---

## 결론

이 글에서 **기본 배포(Deployment+NodePort)** 를 시작으로 **운영형 보강(프로브/리소스/보안/롤링)**, **컨텐츠 주입(ConfigMap)**, **도메인 노출(Ingress)**, **오토스케일(HPA)** 까지 실제 운영 흐름을 따라가며 완성했다.
이 구조를 템플릿으로 삼아, 애플리케이션 특성에 맞는 **프로브/리소스/스케일/보안 정책**을 얹으면 곧바로 **신뢰 가능한 서비스 배포**가 가능하다.
