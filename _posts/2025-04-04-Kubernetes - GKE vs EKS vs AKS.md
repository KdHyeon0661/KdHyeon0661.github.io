---
layout: post
title: Kubernetes - GKE vs EKS vs AKS
date: 2025-04-04 20:20:23 +0900
category: Kubernetes
---
# 클라우드에서의 Kubernetes: GKE vs EKS vs AKS 간단 비교

## 들어가며

- 네트워킹(CNI, LB/Ingress, IP 할당 모델)과 **소스 IP 보존**
- 보안(IAM/OIDC, Secret 암호화, PSP→PSA/Pod Security, 정책 엔진)
- 스토리지/CSI, 볼륨 클래스 매핑(PD/EBS/Azure Disk 및 파일 스토리지)
- 오토스케일링(HPA/VPA/Cluster Autoscaler/노드풀 전략)
- 업그레이드/수명주기, 가용성/멀티존, 로깅/모니터링 통합
- **IaC/CLI 예제**(gcloud/eksctl/az, Terraform 스니펫)와 **트러블슈팅 체크리스트**
- PoC→운영 이행 **의사결정 트리**와 **간단 비용 개념식**

---

## 공통점(복습)

- CNCF 인증 **표준 K8s** (kubectl/Helm/Argo CD 등과 호환)
- **노드풀/오토스케일링/롤링 업데이트/롤백**
- IAM 연동, 로깅/모니터링 통합, API/콘솔/CLI 제공
- **Managed Control Plane**(업데이트·가용성 관리는 벤더가 주도)

---

## 핵심 차이 한눈에(확장 개요)

| 영역 | GKE | EKS | AKS |
|---|---|---|---|
| 운영 모드 | Standard / **Autopilot(서버리스 노드)** | Managed CP + Self-managed/Managed Node + **Fargate(Pod 서버리스)** | Managed CP + Nodepool |
| CNI/데이터패스 | VPC-native(Secondary IP/Alias IP), Dataplane V2(eBPF) 옵션 | **AWS VPC CNI**(ENI), Cilium 선택적 | Azure CNI(Overlay/기본/예산형), Cilium dataplane |
| LB/Ingress | GLB + L7 Ingress(GCE/Envoy), 네이티브 통합 우수 | NLB/ALB + **ALB Ingress Controller** 강력 | SLB/ALB + **AGIC(Application Gateway Ingress Controller)** |
| 스토리지(기본 CSI) | **PD(Standard/Balanced/SSD)**, Filestore(CSI) | **EBS gp3/io**, **EFS**(RWX) | **Azure Disk**(Premium/Ultra), **Azure Files**(RWX) |
| 보안/IAM | GSA↔KSA Workload Identity(풀매니지드 OIDC) | **IRSA(OIDC)**, KMS, PrivateLink | AAD Pod Identity(→**Workload Identity**), Key Vault |
| 관측 | Cloud Logging/Monitoring | CloudWatch / OpenSearch / Managed Prometheus | Azure Monitor/Log Analytics |
| 업그레이드 속도 | **빠름**, 채널(Stable/Rapid) | 보통~느림 | 보통 |
| 비용/과금 감각 | Autopilot로 세밀 청구(미사용 절감) | 컨트롤플레인 무료, 네트워킹/ENI 비용 고려 | 노드풀 비용 무난, 저장소/밴드폭 가격 경쟁력 |
| DX | gcloud 강력, UI 깔끔 | IaC/네트워크 유연성·확장성 | **초심자 친화 UI/문서**, GitHub Actions 연계 용이 |

> **요지**: GKE=자동화·데이터패스 품질, EKS=보안·네트워크 통합/생태계, AKS=학습 곡선 완만·DevOps 친화.

---

## 각 서비스 상세

### GKE (Google Kubernetes Engine)

- **모드**: Standard(노드 직접 운영) / **Autopilot**(노드 자동, Pod단 과금)
- **네트워킹**: VPC-native(Secondary Range), Dataplane V2(선택, eBPF 기반)
- **보안**: Workload Identity(권장), Binary Authorization, GKE Sandbox(GVisor)
- **운영**: 릴리스 채널, 자동 수리/자동 업그레이드, 노드 자동수리

#### 빠른 생성(Autopilot 예)

```bash
gcloud container clusters create-auto demo \
  --region=asia-northeast3 \
  --project=<PROJECT_ID>
gcloud container clusters get-credentials demo --region=asia-northeast3
kubectl get nodes
```

#### NodeLocal DNS, DataplaneV2 스니펫(개념)

```bash
gcloud container clusters update demo \
  --enable-dataplane-v2 \
  --enable-dns-cache
```

---

### EKS (Elastic Kubernetes Service)

- **네트워킹**: AWS VPC CNI(ENI 기반 Pod IP), SecurityGroups for Pods 선택 가능
- **보안**: **IRSA**(IAM Roles for Service Accounts) 표준, KMS, PrivateLink
- **서버리스**: **Fargate**(Pod 단위), ALB/NLB와 깊은 통합

#### 빠른 생성(eksctl)

```bash
eksctl create cluster --name demo --region ap-northeast-2 \
  --with-oidc --nodes 3 --node-type m5.large
aws eks update-kubeconfig --name demo --region ap-northeast-2
kubectl get nodes
```

#### ALB Ingress Controller 설치(요지)

```bash
# OIDC/IRSA 설정 후, 헬름으로 aws-load-balancer-controller 설치

helm repo add eks https://aws.github.io/eks-charts
helm upgrade --install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system --set clusterName=demo \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

---

### AKS (Azure Kubernetes Service)

- **네트워킹**: Azure CNI(Overlay/기본), Cilium dataplane 도입, UDR/VNet 통합 수월
- **보안**: AAD 통합, Azure Workload Identity, Key Vault provider
- **운영**: UI/CLI 친화, 자동 업그레이드/노드풀 관리 간편, GitHub Actions/ADO 연계

#### 빠른 생성(az)

```bash
az group create -n rg-aks -l koreacentral
az aks create -g rg-aks -n demo \
  --node-count 3 --node-vm-size Standard_D4s_v5 \
  --enable-managed-identity --generate-ssh-keys
az aks get-credentials -g rg-aks -n demo
kubectl get nodes
```

#### AGIC(앱 게이트웨이) 인그레스(개념)

```bash
# App Gateway 생성 후 AGIC 애드온 활성화(az aks enable-addons --addons ingress-appgw ...)

```

---

## 네트워킹·LB·Ingress — 실무 논점

- **CNI IP 할당**
  - GKE: Secondary Range(노드/Pod CIDR 분리) → 대규모 IP 관리 용이
  - EKS: ENI 할당 모델 → 서브넷/ENI/Pod 밀도/비용 고려
  - AKS: Azure CNI(Overlay/기본) 옵션 → 오버레이로 IP 절감 가능
- **LB/Ingress**
  - GKE: GCE Ingress(HTTP LB) 또는 Gateway API/Envoy 기반
  - EKS: **ALB Ingress Controller**(다양한 라우팅/인증), 외부 NLB
  - AKS: **AGIC**(L7 WAF/엔터프라이즈), Nginx Ingress도 일반적
- **소스 IP 보존**: `spec.externalTrafficPolicy: Local` + 백엔드 분포 전략

Ingress 예(공통):
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata: { name: web }
spec:
  ingressClassName: nginx  # GKE/GCE, EKS/ALB, AKS/AGIC에 맞게 조정
  rules:
  - host: web.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend: { service: { name: web-svc, port: { number: 80 } } }
```

---

## 스토리지/CSI — 볼륨 클래스 매핑

| 목적 | GKE | EKS | AKS |
|---|---|---|---|
| 블록(PVC RWO) | **PD**(balanced/ssd) | **EBS**(gp3/io2) | **Azure Disk**(Premium/Ultra) |
| 파일(RWX) | **Filestore** | **EFS** | **Azure Files** |
| 암호화 | CMEK 지원 | KMS | Key Vault 통합 |

PVC 예:
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata: { name: data }
spec:
  storageClassName: standard  # gke: standard-rwo, eks: gp3, aks: managed-csi 등
  accessModes: [ReadWriteOnce]
  resources: { requests: { storage: 20Gi } }
```

---

## 오토스케일링 — HPA/VPA/CA/노드풀

- **HPA**: Pod 수 자동 조정(CPU/메모리/사용자 지표)
- **VPA**: Pod 리소스 요청 상향/권고(실험→운영 점진 적용)
- **Cluster Autoscaler**: 노드풀 증감. Managed Node Group/Node Pool과 연동
- **워크로드 격리**: 온디맨드/스팟(Preemptible/Spot) 혼합, taints/tolerations, PodPriority

HPA 예:
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: api-hpa }
spec:
  scaleTargetRef: { apiVersion: apps/v1, kind: Deployment, name: api }
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target: { type: Utilization, averageUtilization: 70 }
```

---

## 보안 — IAM/OIDC/PSA/비밀/정책

- **ID 연동**
  - GKE: Workload Identity(GSA↔KSA)
  - EKS: **IRSA(OIDC)**
  - AKS: **Workload Identity(AAD)**
- **Secret 암호화**: KMS/Key Vault/CMEK로 etcd 암호화
- **Pod Security Admission(PSA)** 라벨로 Baseline/Restricted 적용
- **정책 엔진**: Gatekeeper/OPA, Kyverno

PSA 예:
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: prod
  labels:
    pod-security.kubernetes.io/enforce: restricted
```

---

## 업그레이드/가용성/관측

- **업그레이드**: CP→노드풀 순, 불가역 변경/API 제거 주의
- **멀티존/멀티리전**: 노드풀 Zonal 분산, 스토리지 클래스/존 한정 유의
- **관측**: 관리형 모니터링 + Prom/Grafana(OSS) 혼용, OpenTelemetry 권장

Prometheus-Operator 예(일부):
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata: { name: api }
spec:
  selector: { matchLabels: { app: api } }
  endpoints:
  - port: http
    interval: 30s
```

---

## IaC/CLI — 가장 짧은 생성 예 모음

### GKE(Standard, Zonal)

```bash
gcloud container clusters create demo \
  --zone=asia-northeast3-a --num-nodes=3 \
  --enable-ip-alias --release-channel=regular
```

### EKS(eksctl)

```bash
eksctl create cluster --name demo --region ap-northeast-2 \
  --nodegroup-name ng --nodes 3 --with-oidc
```

### AKS(az)

```bash
az aks create -g rg-aks -n demo --node-count 3 \
  --network-plugin azure --enable-managed-identity
```

Terraform(개념 스니펫):
```hcl
# 각 provider 블록과 모듈 생략, 가독 목적의 최소화만 표기

resource "google_container_cluster" "gke" {
  name     = "demo"
  location = "asia-northeast3-a"
  release_channel { channel = "REGULAR" }
  ip_allocation_policy {}
}
```

---

## 간단 비용 개념식(학습용)

요청률(QPS), 평균 처리시간 \(t\), 파드당 안정 동시 처리량 \(\text{cap}_{pod}\), 노드당 파드 수 \(\text{pods/node}\), 리소스 단가를 둔다면:

$$
\text{필요 파드 수} \approx
\left\lceil \frac{\text{QPS}\cdot t}{\text{cap}_{pod}} \right\rceil \cdot \text{버퍼}
$$

$$
\text{필요 노드 수} \approx
\left\lceil \frac{\text{필요 파드 수}}{\text{pods/node}} \right\rceil
$$

> GKE Autopilot/ EKS Fargate는 **Pod 단위** 과금 비중이 높아 **빈 파드/과대요청**을 줄이는 것이 관건. EKS/AKS 노드풀은 **스팟/예약/절전** 전략으로 비용 최적화.

---

## 트러블슈팅(공통 체크리스트)

| 증상 | 공통 원인 후보 | 1차 점검 | 플랫폼 특이점 |
|---|---|---|---|
| LB EXTERNAL-IP 미할당 | 권한/서브넷/쿼터 | `describe svc`, 컨트롤러 로그 | EKS: ALB Role/IRSA, AKS: AGIC 애드온/권한 |
| Pod Pending | IP/자원 부족/테인트 | `describe pod`, 노드풀 용량 | EKS: ENI 할당 한도, GKE: Secondary IP 범위 |
| DNS 실패 | CoreDNS CrashLoop/CNI | `kubectl -n kube-system get pods` | AKS: Cilium/Overlay 조합 확인 |
| 소스 IP 유실 | DNAT(Cluster 모드) | `externalTrafficPolicy` 확인 | LB 헬스/노드 분포 고려 |
| PVC 바인딩 실패 | SC/존/용량 | `describe pvc` | GKE: PD 지역성, EKS: AZ 매칭, AKS: ZRS 옵션 |

---

## 어떤 서비스를 선택할까? — 의사결정 트리

1) **클라우드 주력**이 명확?
   - GCP 중심 → **GKE**, AWS 중심 → **EKS**, Azure 중심 → **AKS**
2) **서버리스 운영** 선호?
   - 완전 서버리스에 가깝게 → **GKE Autopilot**
   - Pod 서버리스가 필요 → **EKS Fargate**
3) **보안/네트워크** 커스텀 강도?
   - VPC/NLB/ALB/PrivateLink/SG for Pods 등 정교함 → **EKS**
4) **초기 진입/학습 곡선** 완만함?
   - **AKS**(UI/문서/DevOps 연계 용이)
5) **데이터패스/성능/자동화**
   - GKE Dataplane V2, 자동화 생태계 → **GKE**

---

## 데모 매니페스트(공통) — 웹+HPA+PVC

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: web, labels: { app: web } }
spec:
  replicas: 2
  selector: { matchLabels: { app: web } }
  template:
    metadata: { labels: { app: web } }
    spec:
      containers:
      - name: web
        image: nginx:1.27-alpine
        ports: [{ containerPort: 80 }]
        volumeMounts:
        - name: data
          mountPath: /usr/share/nginx/html
      volumes:
      - name: data
        persistentVolumeClaim: { claimName: web-pvc }
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata: { name: web-pvc }
spec:
  accessModes: [ReadWriteOnce]
  resources: { requests: { storage: 10Gi } }
  storageClassName: ""  # 플랫폼별 SC 이름로 교체(gke: standard-rwo, eks: gp3, aks: managed-csi)
---
apiVersion: v1
kind: Service
metadata: { name: web-svc }
spec:
  type: LoadBalancer
  selector: { app: web }
  ports:
  - port: 80
    targetPort: 80
  externalTrafficPolicy: Local  # 소스 IP 보존(노드 분포 주의)
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: web-hpa }
spec:
  scaleTargetRef: { apiVersion: apps/v1, kind: Deployment, name: web }
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource: { name: cpu, target: { type: Utilization, averageUtilization: 60 } }
```

---

## 요약·베스트 프랙티스

- **공통**: 선언형(IaC), 릴리스 계획, 관측/알람, 보안 기준선(RBAC/PSA/Secret 암호화)
- **GKE**: Autopilot로 운영 복잡도↓, Dataplane V2/eBPF, Workload Identity
- **EKS**: **IRSA+ALB** 조합, VPC/ENI 설계, 스팟/온디맨드 혼합 노드풀
- **AKS**: AAD/Workload Identity, AGIC/DevOps 친화, Cilium/Overlay로 IP 효율

> 결론: **주 인프라·보안정책·네트워크 설계·운영 문화**에 맞춰 선택하라.
> PoC는 가볍게 시작(Autopilot/Fargate/기본 AKS) → 관측/보안/스케일을 더하며 **운영 기준선**을 만든다.

부록: 필수 명령 모음
```bash
# GKE

gcloud container clusters create-auto demo --region=asia-northeast3
gcloud container clusters get-credentials demo --region=asia-northeast3

# EKS

eksctl create cluster --name demo --region ap-northeast-2 --with-oidc
aws eks update-kubeconfig --name demo --region ap-northeast-2

# AKS

az aks create -g rg-aks -n demo --node-count 3 --enable-managed-identity
az aks get-credentials -g rg-aks -n demo
kubectl get nodes -o wide
```
