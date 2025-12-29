---
layout: post
title: Kubernetes - GKE vs EKS vs AKS
date: 2025-04-04 20:20:23 +0900
category: Kubernetes
---
# 클라우드에서의 Kubernetes: GKE, EKS, AKS 비교 가이드

## 개요

쿠버네티스를 프로덕션에 도입할 때, 인프라를 직접 구축하고 관리하는 것은 상당한 복잡성과 운영 부담을 동반합니다. 주요 클라우드 제공자들은 이를 해결하기 위해 관리형 쿠버네티스 서비스를 제공합니다: Google의 **GKE(Google Kubernetes Engine)**, AWS의 **EKS(Elastic Kubernetes Service)**, 그리고 Microsoft Azure의 **AKS(Azure Kubernetes Service)** 입니다.

이 서비스들은 컨트롤 플레인(마스터 노드)의 관리, 업데이트, 가용성을 담당하여 사용자가 애플리케이션 배포와 운영에 집중할 수 있게 합니다. 본 가이드는 세 서비스의 핵심 차이점, 네트워킹, 보안, 스토리지, 운영 측면을 비교하여 프로젝트에 적합한 선택을 돕고자 합니다.

---

## 공통 장점: 관리형 쿠버네티스의 가치

세 서비스 모두 다음과 같은 공통된 이점을 제공합니다:
- **CNCF 인증 쿠버네티스**: 표준 `kubectl`, Helm, Argo CD 등과 완벽 호환됩니다.
- **관리형 컨트롤 플레인**: 벤더가 API 서버, etcd, 스케줄러 등의 업데이트와 가용성을 관리합니다.
- **통합 관측성**: 클라우드 네이티브 로깅(Cloud Logging, CloudWatch, Azure Monitor) 및 모니터링 도구와 통합됩니다.
- **자동 확장**: Horizontal Pod Autoscaler(HPA) 및 클러스터 오토스케일러를 지원합니다.
- **클라우드 IAM 통합**: 각 플랫폼의 Identity and Access Management 시스템과 연동됩니다.

---

## 핵심 차이점 요약

| 영역 | GKE (Google) | EKS (AWS) | AKS (Azure) |
|---|---|---|---|
| **운영 모델** | Standard 모드 / **Autopilot** (서버리스 노드) | Managed Control Plane / **Fargate** (Pod 서버리스) | Managed Control Plane + 사용자 관리 노드풀 |
| **네트워킹 (CNI)** | VPC-native (별도 Pod IP 범위) / Dataplane V2(eBPF) 옵션 | **AWS VPC CNI** (ENI 기반) / Cilium 선택 설치 | Azure CNI (Overlay/기본) / Cilium Dataplane |
| **로드 밸런싱 & Ingress** | Google Cloud Load Balancer + GCE Ingress (강력한 통합) | NLB/ALB + **ALB Ingress Controller** (풍부한 기능) | Azure Load Balancer + **AGIC** (Application Gateway Ingress Controller) |
| **스토리지 (CSI)** | **Persistent Disk (PD)** / Filestore (NFS) | **EBS (gp3/io2)** / **EFS** (NFS) | **Azure Disk** / **Azure Files** (SMB/NFS) |
| **보안 & IAM** | Workload Identity (GSA ↔ KSA) | **IRSA (IAM Roles for Service Accounts)** | **AAD Workload Identity** |
| **관측성** | Cloud Logging & Monitoring (통합 우수) | CloudWatch / Managed Prometheus | Azure Monitor / Log Analytics |
| **업그레이드 속도** | **가장 빠름** (릴리스 채널) | 보통 (사용자 관리) | 보통 |
| **비용 구조** | Autopilot은 Pod 기준 세밀 과금 | 컨트롤 플레인 무료, 노드 및 네트워킹 비용 | 직관적인 노드풀 비용, 저장소 경쟁력 |
| **개발자 경험** | `gcloud` CLI 강력, UI 깔끔 | IaC 및 네트워크 설정의 유연성 높음 | **초보자 친화적 UI/문서**, GitHub Actions 연계 용이 |

> **핵심 특징 요약**: **GKE**는 자동화와 네트워크 데이터플레인 성능에, **EKS**는 AWS 생태계와의 깊은 통합 및 고급 네트워킹에, **AKS**는 완만한 학습 곡선과 마이크로소프트 DevOps 도구 연계에 강점이 있습니다.

---

## 각 서비스 상세 분석

### GKE (Google Kubernetes Engine)

Google은 Borg 시스템의 경험을 바탕으로 쿠버네티스를 처음으로 오픈소스화했으며, GKE는 이에 대한 깊은 이해를 반영한 서비스입니다.

- **주요 특징**:
    - **Autopilot 모드**: 노드 프로비저닝과 크기 조정을 완전히 자동화하며, 사용한 Pod 리소스만큼 비용을 지불하는 서버리스 운영 모델을 제공합니다.
    - **VPC-native 네트워킹**: Pod에 클러스터 VPC의 보조 IP 범위(Secondary IP Range)를 직접 할당하여 네트워크 성능과 규모 확장성을 향상시킵니다.
    - **Dataplane V2**: Cilium 기반의 eBPF 데이터플레인을 선택적으로 활성화할 수 있어, 네트워크 정책 성능과 가시성을 높일 수 있습니다.
    - **Workload Identity**: 쿠버네티스 서비스 계정(KSA)을 Google 서비스 계정(GSA)에 안전하게 연결하여 Pod이 Google Cloud API에 접근할 수 있게 합니다.
- **적합한 사용 사례**: Google Cloud를 주력으로 사용하는 조직, 최대한의 운영 자동화를 원하는 팀, 고성능 네트워킹이 필요한 워크로드.

**빠른 클러스터 생성 (Autopilot 모드)**:
```bash
gcloud container clusters create-auto demo-cluster \
  --region=asia-northeast3 \
  --project=<PROJECT_ID>
gcloud container clusters get-credentials demo-cluster --region=asia-northeast3
```

### EKS (Elastic Kubernetes Service)

AWS의 관리형 쿠버네티스 서비스로, 방대한 AWS 서비스 생태계와의 긴밀한 통합이 최대 강점입니다.

- **주요 특징**:
    - **AWS VPC CNI**: 각 Pod에 Elastic Network Interface(ENI)를 할당하여, Pod이 VPC 내의 평범한 인스턴스처럼 통신할 수 있게 합니다. 이는 보안 그룹을 Pod 단위로 적용할 수 있는 **Security Groups for Pods** 기능을 가능하게 합니다.
    - **IAM Roles for Service Accounts (IRSA)**: 쿠버네티스 서비스 계정과 IAM 역할을 연결하는 안전한 방식으로, Pod이 AWS 서비스(S3, DynamoDB 등)에 접근할 수 있는 권한을 부여합니다.
    - **Fargate 프로파일**: 서버리스 컴퓨팅 엔진인 Fargate를 사용하여 Pod을 실행할 수 있습니다. 노드 관리 부담이 전혀 없으며, Pod 단위로 비용이 청구됩니다.
    - **ALB Ingress Controller**: Application Load Balancer(ALB)를 Ingress 리소스로 사용할 수 있는 컨트롤러로, 고급 라우팅 규칙, 인증/인가 통합을 제공합니다.
- **적합한 사용 사례**: AWS 인프라에 이미 투자된 조직, 복잡한 네트워크 및 보안 요구사항이 있는 엔터프라이즈, AWS 특화 서비스(SQS, SNS, RDS 등)와의 긴밀한 통합이 필요한 애플리케이션.

**빠른 클러스터 생성 (`eksctl` 도구 사용)**:
```bash
eksctl create cluster --name demo-cluster --region ap-northeast-2 \
  --with-oidc \  # IRSA 사용을 위한 OIDC 공급자 생성
  --nodes 3 --node-type m5.large
aws eks update-kubeconfig --name demo-cluster --region ap-northeast-2
```

### AKS (Azure Kubernetes Service)

Microsoft Azure의 관리형 쿠버네티스 서비스로, 사용 편의성과 Microsoft 생태계(Active Directory, GitHub, DevOps)와의 통합에 중점을 둡니다.

- **주요 특징**:
    - **Azure AD 통합**: 클러스터 접근 제어를 Azure Active Directory(AAD)와 통합할 수 있어, 기업의 기존 인증 체계를 쉽게 적용할 수 있습니다.
    - **Azure Workload Identity**: GKE의 Workload Identity, EKS의 IRSA와 유사하게, Pod의 서비스 계정이 Azure 서비스에 접근하기 위한 자격 증명을 안전하게 관리합니다.
    - **Application Gateway Ingress Controller (AGIC)**: Azure의 웹 애플리케이션 방화벽(WAF) 기능을 갖춘 Application Gateway를 Ingress 컨트롤러로 사용할 수 있게 합니다. 엔터프라이즈급 보안과 성능을 제공합니다.
    - **개발자 친화성**: Azure Portal의 직관적인 UI와 상세한 문서는 쿠버네티스 초보자에게 매우 유용합니다. GitHub Actions 및 Azure DevOps와의 원활한 통합도 장점입니다.
- **적합한 사용 사례**: Microsoft 기술 스택(예: .NET, Azure DevOps)을 사용하는 조직, 쿠버네티스 학습 및 초기 도입 단계, 엔터프라이즈급 Ingress/WAF 솔루션이 필요한 경우.

**빠른 클러스터 생성 (`az` CLI 사용)**:
```bash
az group create --name rg-aks-demo --location koreacentral
az aks create --resource-group rg-aks-demo --name demo-cluster \
  --node-count 3 --node-vm-size Standard_D4s_v5 \
  --enable-managed-identity --generate-ssh-keys
az aks get-credentials --resource-group rg-aks-demo --name demo-cluster
```

---

## 주요 운영 영역별 비교

### 네트워킹 및 로드 밸런싱
- **IP 관리**: GKE는 별도의 보조 IP 범위를 사용하는 반면, EKS는 VPC ENI를 소비하므로 큰 규모에서는 서브넷 및 ENI 한도를 고려해야 합니다. AKS는 오버레이 네트워킹으로 IP 공간을 절약할 수 있습니다.
- **Ingress 컨트롤러**: 각 플랫폼은 자체 관리형 솔루션(GCE Ingress, ALB Ingress Controller, AGIC)을 권장하지만, Nginx Ingress Controller와 같은 오픈소스 솔루션도 모두 사용 가능합니다.
- **소스 IP 보존**: 외부 클라이언트의 실제 IP 주소를 애플리케이션에 전달하려면 Service의 `externalTrafficPolicy: Local`을 설정해야 합니다. 이때 로드 밸런서의 헬스 체크가 정상 노드로만 전달되도록 구성해야 합니다.

### 스토리지
애플리케이션에 적합한 스토리지 클래스를 선택하는 것이 중요합니다.

| 필요성 | GKE | EKS | AKS |
|---|---|---|---|
| **표준 블록 스토리지 (RWO)** | `standard-rwo` (PD HDD) | `gp3` (범용 SSD) | `managed-csi` (표준 HDD) |
| **고성능 블록 스토리지 (RWO)** | `premium-rwo` (PD SSD) | `io2`/`io2 Block Express` | `managed-csi-premium` |
| **공유 파일 스토리지 (RWX)** | `filestore-csi` (NFS) | `efs-csi` (NFS) | `azurefile-csi` (SMB/NFS) |

PVC 예시 (EKS의 gp3 사용):
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: gp3  # 플랫폼별 적절한 클래스명 사용
  resources:
    requests:
      storage: 100Gi
```

### 보안
- **워크로드 ID**: 세 플랫폼 모두 OIDC 기반의 안전한 자격 증명 관리 방식을 제공합니다(GKE Workload Identity, EKS IRSA, AKS Workload Identity). 이를 사용하면 Pod에 환경변수로 AWS 키/구글 키를 넘기지 않아도 됩니다.
- **Secret 관리**: EKS는 AWS KMS, GKE는 Cloud KMS, AKS는 Azure Key Vault를 사용하여 etcd에 저장된 Secret 데이터를 암호화할 수 있습니다.
- **Pod 보안 정책**: 레거시 PodSecurityPolicy(PSP)는 더 이상 사용되지 않으며, **Pod Security Admission (PSA)** 으로 대체되고 있습니다. 네임스페이스에 `baseline` 또는 `restricted` 보안 표준을 라벨로 적용할 수 있습니다.

### 자동 확장
- **Horizontal Pod Autoscaler (HPA)**: CPU/메모리 사용률이나 커스텀 메트릭을 기반으로 Pod 복제본 수를 조정합니다.
- **Cluster Autoscaler**: 노드풀의 리소스가 부족하면 자동으로 노드를 추가하고, 사용되지 않는 노드는 제거합니다.
- **노드풀 전략**: 비용 절감을 위해 GKE는 선점형(Preemptible) VM, EKS는 스팟 인스턴스(Spot Instances), AKS는 스팟 노드풀(Spot Node Pools)을 활용할 수 있습니다. 중요한 워크로드는 온디맨드 노드풀에, 내결함성이 있는 배치 작업은 스팟 노드풀에 배치하는 혼합 전략이 효과적입니다.

---

## 문제 해결 가이드

초기 설정 시 자주 발생하는 문제들을 다음과 같이 점검할 수 있습니다.

| 증상 | 공통 점검 사항 | 플랫폼별 특이점 |
|---|---|---|
| **로드 밸런서 EXTERNAL-IP가 `<pending>`** | 서비스 `describe` 출력, 컨트롤러 Pod 로그 확인 | **EKS**: IAM 역할(IRSA)이 ALB/NLB 컨트롤러에 부여되었는지 확인.<br>**AKS**: AGIC 애드온이 활성화되고 Application Gateway에 올바른 권한이 설정되었는지 확인. |
| **Pod이 `Pending` 상태** | `kubectl describe pod`로 이벤트 확인 (리소스 부족, 테인트 불일치). | **EKS**: 서브넷에 사용 가능한 ENI/IP가 충분한지 확인 (VPC CNI 한도).<br>**GKE**: 보조 IP 범위(Secondary Range)가 충분한지 확인. |
| **DNS 이름 해석 실패** | CoreDNS Pod 상태 확인 (`kubectl -n kube-system get pods`). | **AKS**: Cilium/오버레이 네트워크를 사용하는 경우 관련 구성 확인. |
| **PVC가 `Pending` 상태** | PVC `describe` 출력, 스토리지 클래스(StorageClass) 존재 여부 확인. | **GKE**: 영구 디스크(PD)가 클러스터와 동일한 리전에 생성되도록 설정.<br>**EKS/AKS**: PVC의 AZ와 노드풀의 AZ가 일치하는지 확인. |

---

## 선택 가이드: 어떤 서비스를 사용해야 할까?

프로젝트와 조직의 상황에 따라 최적의 선택이 달라집니다. 아래 의사결정 트리를 참고하세요.

1.  **주요 클라우드 플랫폼이 이미 결정되었나요?**
    - **예**: 해당 플랫폼의 관리형 서비스(GCP → GKE, AWS → EKS, Azure → AKS)를 선택하는 것이 생태계 통합과 운영 편의성 면에서 유리합니다.
    - **아니오**: 다른 요소들을 고려하세요.

2.  **운영 부담을 최대한 줄이고 싶나요? (서버리스 운영)**
    - **예, 완전 자동화를 원함**: **GKE Autopilot**을 고려하세요. 노드 관리가 전혀 필요 없습니다.
    - **예, 하지만 Pod 단위 컨트롤은 유지함**: **EKS Fargate**를 고려하세요.

3.  **네트워크와 보안 구성에 대한 높은 수준의 제어와 커스터마이징이 필요하나요?**
    - **예**: **EKS**는 VPC, 보안 그룹, IAM 정책과의 세밀한 통합을 통해 가장 유연한 제어를 제공합니다.

4.  **팀의 학습 곡선을 최소화하고 빠르게 시작하는 것이 우선순위인가요?**
    - **예**: **AKS**는 직관적인 포털, 명확한 문서, DevOps 도구 연계로 시작하기 가장 쉬운 경향이 있습니다.

5.  **고성능 네트워크 데이터플레인과 선도적인 자동화 기능에 중점을 두나요?**
    - **예**: **GKE**의 Dataplane V2(eBPF)와 강력한 자동화 생태계가 강점입니다.

**핵심 결론**: 기술적 요구사항만이 아닌, 조직의 **클라우드 전략, 기존 인프라, 팀의 숙련도, 그리고 운영 문화**에 맞는 서비스를 선택하세요. 작은 개념 증명(PoC)으로 시작하여(Autopilot, Fargate, 기본 AKS 등) 운영 복잡도를 점진적으로 높여가는 접근법을 권장합니다.

---

## 실전 예제: 웹 애플리케이션 배포 (HPA + PVC)

다음 YAML은 세 플랫폼 모두에서 동작하는 기본적인 웹 애플리케이션 배포를 보여줍니다. 스토리지 클래스 이름만 플랫폼에 맞게 변경하면 됩니다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  labels:
    app: webapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
        volumeMounts:
        - name: app-data
          mountPath: /usr/share/nginx/html
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
      volumes:
      - name: app-data
        persistentVolumeClaim:
          claimName: webapp-pvc
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: webapp-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: standard-rwo  # 변경 필요: EKS(gp3), AKS(managed-csi)
---
apiVersion: v1
kind: Service
metadata:
  name: webapp-service
spec:
  type: LoadBalancer
  selector:
    app: webapp
  ports:
  - port: 80
    targetPort: 80
  externalTrafficPolicy: Local  # 클라이언트 소스 IP 보존
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: webapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: webapp
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

이 매니페스트를 적용하려면:
```bash
kubectl apply -f webapp-deployment.yaml
```

---

## 결론

GKE, EKS, AKS는 모두 뛰어난 관리형 쿠버네티스 서비스이며, 각각 Google, AWS, Microsoft의 강점과 철학을 반영하고 있습니다. **GKE**는 운영의 단순화와 고급 네트워킹에, **EKS**는 AWS의 방대한 서비스와의 통합 및 정교한 제어에, **AKS**는 사용 편의성과 Microsoft 생태계 연계에 초점을 맞춥니다.

성공적인 도입을 위해서는 단순히 기술적 기능 비교를 넘어, 다음과 같은 요소들을 종합적으로 고려해야 합니다:
- **조직의 클라우드 전략과 기존 투자**
- **팀의 기술적 숙련도와 선호하는 운영 모델**
- **애플리케이션의 특정 요구사항** (예: 특정 데이터베이스 서비스 연계, 특정 네트워크 토폴로지)
- **총소유비용(TCO)과 예산**

어떤 서비스를 선택하든, **인프라를 코드(IaC)로 관리(Terraform, Crossplane 등)**, **강력한 관측성(모니터링, 로깅, 추적)을 구축**, 그리고 **보안 모범 사례(최소 권한, 비밀 관리, Pod 보안)** 를 일찍부터 적용하는 것이 장기적인 운영 성공의 핵심입니다.