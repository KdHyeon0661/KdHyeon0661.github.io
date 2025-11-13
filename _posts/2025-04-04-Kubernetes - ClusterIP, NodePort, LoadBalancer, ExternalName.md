---
layout: post
title: Kubernetes - ClusterIP, NodePort, LoadBalancer, ExternalName
date: 2025-04-04 22:20:23 +0900
category: Kubernetes
---
# K8s 핵심 오브젝트 이해하기 : Service (ClusterIP, NodePort, LoadBalancer, ExternalName)

## 들어가며

- **작동 원리**: selector → Endpoints/EndpointSlice → kube-proxy(iptable/ipvs) 데이터패스
- **Type별 차이와 실무 옵션**: `externalTrafficPolicy`, `sessionAffinity`, `topologyKeys`, `publishNotReadyAddresses`
- **Headless Service / DNS / 서비스 디스커버리 패턴**
- **소스 IP 보존, 헬스체크, readiness와 라우팅의 관계**
- **MetalLB/클라우드 LB, Ingress와의 역할 분리**
- **테스트/관측/트러블슈팅 명령 모음**

---

## Service란? (요약 복습)
- **Pod 집합**(라벨 셀렉터)의 **안정된 접근 지점**(Virtual IP/DNS)을 제공하는 L4 추상화.
- Pod의 수명과 무관하게 **일관된 주소**를 노출하며, kube-proxy(또는 eBPF CNI)가 **부하 분산**한다.
- 내부 DNS 규칙: `서비스명.네임스페이스.svc.cluster.local`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api
spec:
  selector: { app: api }
  ports:
  - name: http
    port: 80
    targetPort: 8080
```

---

## Service 작동 원리 — Selector → EndpointSlice → kube-proxy
1) 사용자가 Service를 생성하면, **label selector** 로 매칭되는 Pod 집합이 **EndpointSlice**로 관리된다.
2) `kube-proxy`(각 노드 DaemonSet)가 EndpointSlice를 **Watch** 하며 **iptable/ipvs 규칙**(혹은 eBPF CNI)을 갱신한다.
3) 클라이언트가 Service VIP로 트래픽을 보내면, 데이터패스가 **실제 Pod IP:Port** 로 DNAT/포워딩한다.

EndpointSlice 조회:
```bash
kubectl get endpointslices -l kubernetes.io/service-name=api
kubectl describe endpointslice <name>
```

---

## 주요 타입 비교 한눈에

| 타입 | VIP | 외부 접근 | LB 필요 | 주용도 |
|---|---|---|---|---|
| **ClusterIP**(기본) | 있음(ClusterIP) | ❌ | ❌ | 내부 마이크로서비스 통신 |
| **NodePort** | 있음 | ✅(노드IP:포트) | ❌ | 간단 외부 노출/테스트 |
| **LoadBalancer** | 있음 | ✅(외부 IP) | ✅(클라우드/MetalLB) | 실서비스 외부 노출 |
| **ExternalName** | 없음(CNAME) | ✅(DNS만) | ❌ | 외부 서비스 프록시(DNS) |
| **Headless**(`clusterIP: None`) | 없음 | 내부만(직결) | ❌ | Stateful/직접 디스커버리 |

---

## ClusterIP — 내부 통신의 표준

### 기본 예시
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-clusterip
spec:
  selector: { app: nginx }
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

### 포인트
- 클러스터 내부에서만 접근 가능.
- `kubectl port-forward` 로 로컬에서 임시 프록시 가능:
```bash
kubectl port-forward svc/nginx-clusterip 8080:80
```

### 세션 어피니티(쿠키 없이 L4 레벨 고정화)
```yaml
spec:
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800  # 최대 24h
```

---

## NodePort — 노드 포트를 열어 외부에서 접근

### 예시
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport
spec:
  type: NodePort
  selector: { app: nginx }
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080   # 30000~32767
```

### 포인트
- 접근: `http://<노드IP>:30080`
- **source IP 보존**: 기본은 `externalTrafficPolicy: Cluster` → DNAT 경유 시 소스 IP가 변경될 수 있다. 필요하면 아래 참조.

---

## LoadBalancer — 외부 L4 LB를 통한 공개

### 클라우드/MetalLB 공통 예시
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-lb
spec:
  type: LoadBalancer
  selector: { app: nginx }
  ports:
  - port: 80
    targetPort: 80
```

### 실무 옵션
- **소스 IP 보존**(직접 노드로 보내게 강제):
```yaml
spec:
  externalTrafficPolicy: Local
```
  - 장점: 클라이언트 **소스 IP 보존**.
  - 단점: 일부 노드에 백엔드가 없으면 그 노드로 유입된 트래픽은 실패(HealthCheck/노드 선택을 고려).
- **헬스체크 노드포트**(클라우드 벤더가 노드 헬스 확인 시 사용):
```yaml
spec:
  externalTrafficPolicy: Local
# status.loadBalancer.ingress 할당 후, kube-proxy가 healthCheckNodePort 자동 생성(환경 의존).
```

> **MetalLB**(베어메탈/로컬) 사용 시 IP 풀/ARP 광고 설정이 필요. Minikube/Kind에서는 별도 설치 후 실습 가능.

---

## ExternalName — 외부 DNS로 프록시(CNAME)

### 예시
```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-google
spec:
  type: ExternalName
  externalName: www.google.com
```

### 포인트
- **포트 지정 불가**, **엔드포인트 생성 없음**.
- `svc DNS → CNAME → 외부 도메인` 으로 해석. 실제 트래픽은 Pod가 외부로 직접 나간다.

---

## Headless Service — clusterIP: None (직접 디스커버리)

### 예시(스테이트풀/직결)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: db
spec:
  clusterIP: None
  selector: { app: db }
  ports:
  - port: 5432
```

### 포인트
- VIP 없이 **Pod별 DNS A 레코드**를 반환 → 클라이언트가 직접 선택/셸빙 가능.
- StatefulSet과 조합하여 `pod-0.db.default.svc` 처럼 예측 가능한 DNS 사용.

---

## DNS와 서비스 디스커버리

### 내부 DNS 패턴
- `svc`: `nginx.default.svc.cluster.local`
- 같은 네임스페이스에서는 `nginx` 만으로도 접근 가능(리졸버 서치도메인).

### 테스트
```bash
kubectl run -it --rm dns-test --image=nicolaka/netshoot -- /bin/bash
# 컨테이너 내부
dig nginx.default.svc.cluster.local
curl -sS http://nginx.default.svc.cluster.local
```

---

## kube-proxy 데이터패스 — iptables vs ipvs (개념)

- **iptables**: 널리 쓰이는 기본. 단순/호환성 우수.
- **ipvs**: 커넥션 수 많을 때 스케일/성능 우수. kube-proxy 설정으로 전환 가능.
- 일부 CNI(예: Cilium)는 eBPF로 **kube-proxy 없는** L4 로드밸런싱 제공(클러스터 설정에 따름).

kube-proxy 모드 확인:
```bash
kubectl -n kube-system get ds kube-proxy -o yaml | grep -i mode -n
```

---

## 실습: 간단 Nginx 배포 + Service + 검증

### Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: nginx, labels: { app: nginx } }
spec:
  replicas: 2
  selector: { matchLabels: { app: nginx } }
  template:
    metadata: { labels: { app: nginx } }
    spec:
      containers:
      - name: nginx
        image: nginx:1.27-alpine
        ports: [{ containerPort: 80 }]
        readinessProbe:
          httpGet: { path: /, port: 80 }
          initialDelaySeconds: 5
          periodSeconds: 5
```

### NodePort Service
```yaml
apiVersion: v1
kind: Service
metadata: { name: nginx-nodeport }
spec:
  type: NodePort
  selector: { app: nginx }
  ports:
  - name: http
    port: 80
    targetPort: 80
    nodePort: 30080
```

### 검증
```bash
kubectl apply -f deploy.yaml
kubectl apply -f svc-nodeport.yaml
kubectl get pods -o wide
kubectl get svc nginx-nodeport
curl -I http://<node-ip>:30080
```

---

## 실습: LoadBalancer(MetalLB or 클라우드) + 소스 IP 보존

### Service (source IP 보존)
```yaml
apiVersion: v1
kind: Service
metadata: { name: web-lb }
spec:
  type: LoadBalancer
  selector: { app: nginx }
  externalTrafficPolicy: Local
  ports:
  - port: 80
    targetPort: 80
```

### 서버에서 소스 IP 관찰(예: nginx log에 remote_addr 기록)
- `externalTrafficPolicy: Local` 로 **DNAT 없이** 트래픽이 도착해 **클라이언트 IP** 가 로그에 남는다.
- 단, **백엔드 Pod가 없는 노드** 로 들어온 트래픽은 실패할 수 있다(노드 그룹/스케줄 전략 고려).

---

## 실습: Headless + StatefulSet (간단 패턴)

```yaml
apiVersion: v1
kind: Service
metadata: { name: echo-hs }
spec:
  clusterIP: None
  selector: { app: echo }
  ports:
  - port: 5678
---
apiVersion: apps/v1
kind: StatefulSet
metadata: { name: echo }
spec:
  serviceName: "echo-hs"
  replicas: 3
  selector: { matchLabels: { app: echo } }
  template:
    metadata: { labels: { app: echo } }
    spec:
      containers:
      - name: echo
        image: hashicorp/http-echo
        args: ["-text=$(POD_NAME)"]
        env:
        - name: POD_NAME
          valueFrom: { fieldRef: { fieldPath: metadata.name } }
        ports: [{ containerPort: 5678 }]
```

DNS 예:
```bash
# 각 Pod가 개별 A레코드로 노출
dig echo-0.echo-hs.default.svc.cluster.local
dig echo-hs.default.svc.cluster.local  # 여러 A레코드 반환
```

---

## 자주 쓰는 옵션(실무 체크리스트)

- **`externalTrafficPolicy: Local`**: 소스 IP 보존(노드 로컬 트래픽만 수락).
- **`sessionAffinity: ClientIP`**: L4 레벨 세션 유지.
- **`publishNotReadyAddresses: true`**: 레디가 아니어도 Endpoints에 포함(특수 상황).
- **`topologyKeys`**: 레거시. 현재는 `topologySpreadConstraints` 등으로 트래픽/배치 균형을 잡는 편.
- **어노테이션 기반 벤더 옵션**: 클라우드 LB(HealthCheck, IdleTimeout, ProxyProtocol 등)는 각 벤더 문서 참조.

---

## Ingress와 Service의 역할 분리
- **Service**: L4 가상 IP(LB/NodePort/ClusterIP)와 Pod 집합 연결.
- **Ingress**: L7(HTTP/HTTPS) 라우팅 규칙(호스트/경로) → **반드시** 백엔드로 **Service** 필요.
- 실무에선 **`Ingress + (백엔드) Service`** 가 일반적이며, 외부 L7 게이트웨이(Gateway API)를 쓰는 패턴도 증대.

---

## 관측/디버깅 명령 모음

```bash
# 서비스/엔드포인트/엔드포인트슬라이스
kubectl get svc -A -o wide
kubectl get endpoints -A
kubectl get endpointslices -A

# 데이터 경로 확인
kubectl get svc nginx -o yaml
kubectl describe svc nginx
kubectl get ep nginx -o wide
kubectl get endpointslices -l kubernetes.io/service-name=nginx -o wide

# 네트워크 툴박스
kubectl run -it --rm netshoot --image=nicolaka/netshoot -- /bin/bash
# 내부에서:
curl -I http://nginx.default.svc.cluster.local
dig +short nginx.default.svc.cluster.local

# 노드/프록시 상태
kubectl -n kube-system get ds kube-proxy -o wide
kubectl -n kube-system logs ds/kube-proxy --tail=200
```

---

## 문제 해결(트러블슈팅) 테이블

| 증상 | 가능 원인 | 점검 | 해결 |
|---|---|---|---|
| `ClusterIP`로 접근 불가 | 셀렉터/레이블 미스매치 | `kubectl get ep <svc>` | Service.selector ↔ Pod.labels 정정 |
| NodePort 접근 404/Timeout | 방화벽/노드IP 혼동/파드 미스케줄 | `kubectl get nodes -o wide`, `get ep`, `describe svc` | 노드IP 확인, 파드 분포/포트 확인 |
| LB 외부 IP 미할당 | 클라우드 권한/MetalLB 미구성 | `kubectl get svc`, 컨트롤러 로그 | 클라우드 IAM/MetalLB IPPool 설정 |
| 소스 IP가 프록시 IP로 보임 | `externalTrafficPolicy: Cluster` | `kubectl get svc -o yaml` | `externalTrafficPolicy: Local` 로 변경(주의사항 고려) |
| 일부 노드만 접속 가능 | Local 정책 + 백엔드 없음 | `kubectl get pods -o wide` | Pod를 모든 노드에 분산/노드 선택 제어 |
| DNS 실패 | CoreDNS CrashLoop/CNI 이슈 | `kubectl -n kube-system get pods`, `logs` | CNI/코어DNS 정상화, 네임서버 확인 |

---

## 성능/용량 개념식(학습용, 간단)
서비스에 도달하는 초당 요청 수(QPS)와 평균 응답 시간 \(t\) 가 주어지면, 요구 동시성은 대략:
$$
\text{동시성} \approx \text{QPS} \cdot t
$$
파드당 안정 동시 처리량 \(\text{cap}_{pod}\) 일 때 필요한 최소 레플리카 수:
$$
\text{replicas}_{\min} =
\left\lceil \frac{\text{동시성}}{\text{cap}_{pod}} \right\rceil \cdot \text{버퍼}
$$
> 라우팅은 Service가 하지만, **실제 처리 능력은 Pod/노드 자원**이 결정한다. HPA로 동적으로 보정하자.

---

## 네임스페이스 간 접근 예시(복습)
```bash
# nginx 서비스가 default에 있을 때
curl http://nginx.default.svc.cluster.local
```

---

## 요약/베스트 프랙티스
- **ClusterIP**: 기본. 내부 통신에 사용.
- **NodePort**: 빠른 외부 테스트. 운영에는 제한적.
- **LoadBalancer**: 퍼블릭 노출 표준. 클라우드/MetalLB 필요.
- **ExternalName**: 내부 DNS alias → 외부 서비스.
- **Headless**: VIP 없이 Pod 개별 디스커버리(StatefulSet 등).
- **추가 옵션**: `externalTrafficPolicy`, `sessionAffinity`, `publishNotReadyAddresses` 를 상황에 맞게.
- **검증 습관**: 항상 `svc ↔ endpoints(혹은 endpointslice) ↔ pod` 연계를 확인.

---

## 부록: 한 번에 실행하는 전체 예시(4종 Service)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: demo, labels: { app: demo } }
spec:
  replicas: 2
  selector: { matchLabels: { app: demo } }
  template:
    metadata: { labels: { app: demo } }
    spec:
      containers:
      - name: app
        image: hashicorp/http-echo
        args: ["-text=hello demo"]
        ports: [{ containerPort: 5678 }]

---
# 1. ClusterIP
apiVersion: v1
kind: Service
metadata: { name: demo-clusterip }
spec:
  selector: { app: demo }
  ports: [{ name: http, port: 80, targetPort: 5678 }]

---
# 2. NodePort
apiVersion: v1
kind: Service
metadata: { name: demo-nodeport }
spec:
  type: NodePort
  selector: { app: demo }
  ports:
  - name: http
    port: 80
    targetPort: 5678
    nodePort: 30081

---
# 3. LoadBalancer (MetalLB/클라우드 필요)
apiVersion: v1
kind: Service
metadata: { name: demo-lb }
spec:
  type: LoadBalancer
  selector: { app: demo }
  externalTrafficPolicy: Local
  ports:
  - name: http
    port: 80
    targetPort: 5678

---
# 4. ExternalName
apiVersion: v1
kind: Service
metadata: { name: demo-external }
spec:
  type: ExternalName
  externalName: example.com
```

검증 명령:
```bash
kubectl apply -f demo.yaml
kubectl get svc -o wide
kubectl get endpoints,endpointSlices -l kubernetes.io/service-name=demo-clusterip
kubectl run -it --rm t --image=radial/busyboxplus:curl -- curl -sS http://demo-clusterip
```

---

## 결론
Service는 쿠버네티스 네트워킹의 **관문**이자 **안정화 장치**다. 타입/옵션/데이터패스를 이해하면, “왜 트래픽이 이 Pod로 갔는가?”, “왜 소스 IP가 사라졌는가?” 같은 질문에 즉시 답할 수 있다.
운영에서는 **Service ↔ Ingress ↔ Pod** 관계를 **관측/테스트/정책**으로 단단히 묶자. 그렇게 하면 확장·리다이렉션·장애 전파를 예측 가능하게 만들 수 있다.
