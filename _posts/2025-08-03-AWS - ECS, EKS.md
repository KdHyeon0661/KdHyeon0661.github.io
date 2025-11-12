---
layout: post
title: AWS - ECS, EKS
date: 2025-08-03 18:20:23 +0900
category: AWS
---
# ECS / EKS: 컨테이너 서비스

## 0. 빠른 개요: 언제 ECS? 언제 EKS?

| 질문 | ECS 권장 | EKS 권장 |
|---|---|---|
| 빠른 시작/낮은 학습 곡선 | Fargate + ECS |  |
| 쿠버네티스 생태계(Helm/Istio/ArgoCD 등) |  | ✔ |
| 멀티클러스터/하이브리드/멀티클라우드 전략 | 제한적 | ✔ |
| 매우 세밀한 네트워크/보안/스케줄링 커스터마이징 | 제한적 | ✔ |
| 단순 MSA/배치 잡/이벤트 처리 | ✔ |  |
| 팀의 K8s 숙련도 | 불필요 | 필요 |

> **핵심**: 빠르게 운영하려면 **ECS**, 생태계/유연성을 극대화하려면 **EKS**.

---

## 1. 공통 토대: 컨테이너 아키텍처 핵심

### 1.1 네트워킹(VPC) 설계 기본
- **퍼블릭 서브넷**: ALB/NLB, NAT GW 등 외부 노출 리소스
- **프라이빗 서브넷**: 서비스(타스크/파드), RDS/ElastiCache 등 데이터 계층
- **보안그룹**: 허용 규칙만(Stateless NACL은 억제적으로)
- **엔드포인트**: S3/DynamoDB/STS 등 **Interface/Gateway Endpoint**로 NAT 비용↓
- **로깅**: VPC Flow Logs, CloudTrail, Config, GuardDuty

### 1.2 컨테이너 레지스트리
- **ECR** 사용: 이미지 스캔, **수명주기(Lifecycle)** 규칙로 이미지 정리  
```json
{
  "rules": [
    { "rulePriority": 1, "description": "Keep last 20", "selection": { "tagStatus": "any", "countType": "imageCountMoreThan", "countNumber": 20 }, "action": { "type": "expire" } }
  ]
}
```

### 1.3 CI/CD 파이프라인(샘플 흐름)
```
Git → Build(gha/CodeBuild) → ECR Push → IaC(CDK/CFn/Terraform) → ECS/EKS 배포
```

---

## 2. Amazon ECS 완전 가이드

### 2.1 핵심 개념 복습
- **Task Definition**: 컨테이너 이미지/리소스/환경/로깅/볼륨 등 정의(JSON)
- **Task**: 실행 단위(1..n 컨테이너) — K8s Pod에 대응
- **Service**: Task의 **지속 실행/DesiredCount 유지** 및 배포 전략 관리
- **Cluster**: 작업 실행 논리 그룹(용량: EC2 or Fargate)
- **Launch Type**: `EC2` vs `FARGATE`
- **Capacity Provider**: Spot/On-Demand/혼합 정책, Auto Scaling 연계

### 2.2 네트워킹 모드
- **awsvpc(권장)**: 컨테이너(ENI) 단위 IP 부여 → 보안그룹/네트워크 제어 용이
- bridge/host(EC2)도 가능하나 신규는 awsvpc 추천

### 2.3 Task Definition 예제(로깅, 헬스체크, ulimits, Secret, FireLens)
```json
{
  "family": "webapp",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "executionRoleArn": "arn:aws:iam::123456789012:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::123456789012:role/appTaskRole",
  "containerDefinitions": [
    {
      "name": "app",
      "image": "123456789012.dkr.ecr.ap-northeast-2.amazonaws.com/webapp:latest",
      "portMappings": [{ "containerPort": 8080, "protocol": "tcp" }],
      "healthCheck": {
        "command": ["CMD-SHELL", "curl -f http://localhost:8080/health || exit 1"],
        "interval": 10, "timeout": 5, "retries": 3, "startPeriod": 10
      },
      "environment": [
        { "name": "ENV", "value": "prod" }
      ],
      "secrets": [
        { "name": "DB_PASSWORD", "valueFrom": "arn:aws:secretsmanager:ap-northeast-2:123456789012:secret:dbpass-xxxx" }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-region": "ap-northeast-2",
          "awslogs-group": "/ecs/webapp",
          "awslogs-stream-prefix": "app"
        }
      },
      "ulimits": [{ "name": "nofile", "softLimit": 65536, "hardLimit": 65536 }]
    }
  ]
}
```

### 2.4 서비스+ALB(Fargate) 배포 CLI
```bash
# 클러스터
aws ecs create-cluster --cluster-name app-cluster

# 태스크 정의 등록
aws ecs register-task-definition --cli-input-json file://taskdef.json

# 서비스 생성(서브넷·보안그룹은 프라이빗/앱SG)
aws ecs create-service \
  --cluster app-cluster \
  --service-name web-service \
  --task-definition webapp \
  --desired-count 2 \
  --launch-type FARGATE \
  --network-configuration 'awsvpcConfiguration={subnets=[subnet-a,subnet-b],securityGroups=[sg-app],assignPublicIp=DISABLED}' \
  --load-balancers 'targetGroupArn=arn:aws:elasticloadbalancing:ap-northeast-2:123456789012:targetgroup/tg/xxx,containerName=app,containerPort=8080' \
  --deployment-controller type=ECS \
  --scheduling-strategy REPLICA
```

### 2.5 배포 전략
- **롤링 기본**, 실패시 **Deployment Circuit Breaker**(자동 롤백)
```bash
aws ecs update-service \
  --cluster app-cluster \
  --service web-service \
  --deployment-configuration 'maximumPercent=200,minimumHealthyPercent=100,deploymentCircuitBreaker={enable=true,rollback=true}'
```
- **Blue/Green**: CodeDeploy와 연동(테스트 트래픽 전환, 단계적 라우팅)

### 2.6 오토스케일링
- **Service Auto Scaling**: CPU/Memory/Request 기반 DesiredCount 자동 조정
```bash
aws application-autoscaling register-scalable-target \
  --service-namespace ecs \
  --resource-id service/app-cluster/web-service \
  --scalable-dimension ecs:service:DesiredCount \
  --min-capacity 2 --max-capacity 10

aws application-autoscaling put-scaling-policy \
  --service-namespace ecs \
  --scalable-dimension ecs:service:DesiredCount \
  --resource-id service/app-cluster/web-service \
  --policy-name cpu70 \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "TargetValue":70.0,"PredefinedMetricSpecification":{"PredefinedMetricType":"ECSServiceAverageCPUUtilization"}
  }'
```

### 2.7 서비스 디스커버리
- Cloud Map로 `app.namespace.local` 이름해결 → 내부 gRPC/HTTP 서비스 연결 용이

### 2.8 로깅·모니터링
- **awslogs** 혹은 **FireLens(Fluent Bit)** → CloudWatch Logs/Elastic/OpenSearch
- X-Ray, CloudWatch ServiceLens로 트레이스
- 메트릭: CPU/Memory/RunningTask/Service Deployment 지표

### 2.9 보안/권한
- **Task Execution Role**: ECR Pull/Logs 권한
- **Task Role**: 애플리케이션이 필요한 AWS API 최소권한(IAM Boundary/ABAC 고려)
- SG: ALB→App 80/TCP 허용, 외부는 ALB만 공개

### 2.10 비용 최적화 팁
- Fargate vCPU/GB-초 과금 → **리소스 할당 상향 주의**
- EC2 Launch + **Spot**(Capacity Provider) 혼합로 비용↓
- NAT GW 절감: VPC Endpoint 적극 활용
- 로그 보존 기간/샘플링 조정

---

## 3. Amazon EKS 완전 가이드

### 3.1 핵심 개념 복습
- **Control Plane**: AWS 관리(HA)
- **노드 그룹**: EC2(Managed/Unmanaged) 또는 **Fargate Profile**(서버리스 파드)
- **AWS VPC CNI**: 파드에 VPC IP 할당(서브넷 IP 용량 고려)
- **IRSA(권장)**: IAM Role for Service Account — 파드별 세밀 권한

### 3.2 클러스터 생성(eksctl 예)
```bash
eksctl create cluster \
  --name app-eks \
  --region ap-northeast-2 \
  --nodes 3 --node-type t3.large \
  --with-oidc \
  --managed
```

### 3.3 애플리케이션 배포(Deployment/Service/Ingress)
```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: web, labels: { app: web } }
spec:
  replicas: 3
  selector: { matchLabels: { app: web } }
  template:
    metadata: { labels: { app: web } }
    spec:
      serviceAccountName: web-sa
      containers:
        - name: app
          image: 123456789012.dkr.ecr.ap-northeast-2.amazonaws.com/web:latest
          ports: [{ containerPort: 8080 }]
          readinessProbe: { httpGet: { path: /health, port: 8080 }, initialDelaySeconds: 5, periodSeconds: 10 }
          resources:
            requests: { cpu: "200m", memory: "256Mi" }
            limits:   { cpu: "500m", memory: "512Mi" }
---
apiVersion: v1
kind: Service
metadata: { name: web-svc }
spec:
  type: NodePort
  selector: { app: web }
  ports: [{ port: 80, targetPort: 8080 }]
```

**AWS Load Balancer Controller**(ALB Ingress) 예:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ing
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
spec:
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend: { service: { name: web-svc, port: { number: 80 } } }
```

### 3.4 오토스케일링
- **HPA**: 파드 수준
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: web-hpa }
spec:
  scaleTargetRef: { apiVersion: apps/v1, kind: Deployment, name: web }
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target: { type: Utilization, averageUtilization: 70 }
```
- **Cluster Autoscaler / Karpenter**: 노드 증감(Spot/온디맨드 혼합)
  - Karpenter는 가용성/가격 최적의 개별 노드 프로비저닝

### 3.5 스토리지/시크릿
- **EBS CSI**: 블록 스토리지(PV/PVC)
- **EFS CSI**: 공유 파일시스템
- **Secrets Store CSI**: Secrets Manager/SSM 파드 마운트
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata: { name: web-pvc }
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: gp3
  resources: { requests: { storage: 20Gi } }
```

### 3.6 보안
- **IRSA**: 파드별 IAM(필수)
- **RBAC**: 최소 권한 Role/RoleBinding
- **Pod Security(기본 표준)** 또는 Kyverno/OPA Gatekeeper 정책 준수
- **NetworkPolicy**(Calico/Cilium): 파드 간 트래픽 허용 목록
- **서비스 메시**(App Mesh/Istio): mTLS, 트래픽 분할, 관측성

### 3.7 관측성(Observability)
- **Prometheus/Grafana**(AMP/AMG), **CloudWatch Agent**(컨테이너 인사이트), **OpenTelemetry**
- **X-Ray**(애플리케이션 트레이스) + ServiceLens
- **로그**: Fluent Bit DaemonSet → CloudWatch/OS/OpenSearch

### 3.8 배포 전략
- **RollingUpdate** 기본, **Canary/BlueGreen**: Argo Rollouts, Flagger 등
- GitOps: **ArgoCD**로 선언형 배포 파이프라인

### 3.9 비용·용량 계산 감각
- 파드 요청/리밋 합이 노드 용량을 초과하지 않도록 **Packing** 최적화  
  $$ \text{Node 수} \approx \left\lceil \max\left( \frac{\sum \text{CPU requests}}{\text{Node CPU}}, \frac{\sum \text{Mem requests}}{\text{Node Mem}} \right) \right\rceil $$
- IP 소진: VPC CNI 파드당 IP → **서브넷 여유**/PrefixMode 확인
- Fargate 파드 단가 vs EC2(Spot 혼합) 비교

---

## 4. ECS vs EKS: 운영·배포 전략 비교

| 주제 | ECS | EKS |
|---|---|---|
| 배포 | ECS 롤링/CodeDeploy Blue-Green | K8s Rolling/Canary/Blue-Green(Argo Rollouts) |
| 서비스 연결 | ALB/Service Discovery(Cloud Map) | ALB Ingress/Service Mesh |
| 권한 | Task Role/Execution Role | IRSA + RBAC |
| 오토스케일 | Service Auto Scaling/Capacity Provider | HPA + Cluster Autoscaler/Karpenter |
| 로깅/모니터링 | awslogs/FireLens + CW/X-Ray | Fluent Bit/OTel + Prom/Grafana + CW/X-Ray |
| IaC | CloudFormation/CDK/Terraform | 동일 + Helm/ArgoCD |

---

## 5. 실습 I — ECS Fargate 표준 패턴(End-to-End)

### 5.1 전제
- **프라이빗 서브넷** 2개, **퍼블릭 서브넷** 2개, **ALB** 퍼블릭에 배치
- ECR에 `webapp:latest` 이미지

### 5.2 ALB/TG/SG
```bash
# 보안그룹(예시)
aws ec2 create-security-group --group-name alb-sg --description alb --vpc-id vpc-xxx
aws ec2 authorize-security-group-ingress --group-id sg-alb --protocol tcp --port 80 --cidr 0.0.0.0/0
aws ec2 create-security-group --group-name app-sg --description app --vpc-id vpc-xxx
aws ec2 authorize-security-group-ingress --group-id sg-app --protocol tcp --port 8080 --source-group sg-alb
```

### 5.3 ECS 리소스
- 위 2.3 TaskDef JSON + 2.4 create-service로 배포  
- 헬스체크 패스 → ALB 경유 정상 응답

### 5.4 오토스케일/배포회로차단/로그 보존 기간 설정
- 2.5/2.6/로깅 수명주기 함께 적용

---

## 6. 실습 II — EKS 표준 패턴(End-to-End)

### 6.1 클러스터
```bash
eksctl create cluster --name demo --region ap-northeast-2 --with-oidc --nodes 3 --managed
```

### 6.2 AWS Load Balancer Controller 설치(Helm)
```bash
helm repo add eks https://aws.github.io/eks-charts
helm upgrade -i aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system --set clusterName=demo --set serviceAccount.create=false \
  --set region=ap-northeast-2 --set vpcId=vpc-xxx
```
> 사전에 IRSA 권한 바인딩(ServiceAccount + IAM Policy) 필요

### 6.3 앱/서비스/인그레스 + HPA
- 3.3/3.4 예제 적용 후 `kubectl get ingress,svc,deploy,hpa -n default`로 확인

### 6.4 관측성
- CloudWatch Agent + Fluent Bit DaemonSet 설치, 대시보드 구성

---

## 7. 패턴·참고 설계

### 7.1 멀티테넌시
- **ECS**: 서비스/클러스터 분리 + Task Role/SG/네임스페이스 태깅
- **EKS**: Namespace 격리 + NetworkPolicy + ResourceQuota + IRSA

### 7.2 데이터 계층 접근
- RDS/EKS/ECS: SG로 L4 제어, IAM DB Auth(옵션), 시크릿은 SM/SSM/CSI

### 7.3 메시/고급 트래픽
- **ECS**: App Mesh(Envoy 사이드카)로 서킷브레이커/재시도/관측성
- **EKS**: Istio/App Mesh/Linkerd

### 7.4 배치/이벤트 처리
- **ECS**: Scheduled Tasks(EventBridge)로 크론 잡
- **EKS**: CronJob

---

## 8. 트러블슈팅 체크리스트

- **접속 실패**: ALB 헬스체크 경로/보안그룹/서브넷 라우팅 확인  
- **DNS 불가**: VPC DNS Hostnames/Support 켜기, CoreDNS/조건부포워딩(eks)  
- **이미지 Pull 실패**: ECR 권한(Task Execution Role/IRSA), VPC 엔드포인트  
- **IP 소진**(EKS): 서브넷 사용률/Prefix Mode 설정  
- **스케일 안됨**: HPA 대상 메트릭 노출 확인(리소스/EMF/PromAdapter), Autoscaler 로그  
- **CPU 100%**: 리밋/요청/GC 튜닝, 사이드카 고려, pprof/otel로 병목 추적

---

## 9. 비용·성능 최적화 포인트

- **ECS Fargate**: vCPU/메모리 티어 최소화, **서브 프로세스/스레드 수 조정**으로 오버프로비 방지
- **EKS**: Spot + Karpenter, 바이너리 최적화(distroless/Alpine), **멀티스테이지 빌드**로 이미지 크기↓
- **NAT 게이트웨이**: 요금 민감 → ECR/CloudWatch 등 엔드포인트 구성
- **로그**: 수집 경로 단순화 + 보존 기간(7~14일) + 샘플링

---

## 10. IaC: CDK 스니펫(요약)

### 10.1 CDK로 ECS Fargate 서비스
```ts
import * as cdk from 'aws-cdk-lib';
import { Cluster, FargateService, FargateTaskDefinition, ContainerImage, AwsLogDriver } from 'aws-cdk-lib/aws-ecs';
import { ApplicationLoadBalancedFargateService } from 'aws-cdk-lib/aws-ecs-patterns';
import { Vpc } from 'aws-cdk-lib/aws-ec2';

export class EcsFargateStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string) {
    super(scope, id);
    const vpc = Vpc.fromLookup(this, 'Vpc', { isDefault: false });
    const cluster = new Cluster(this, 'Cluster', { vpc });

    new ApplicationLoadBalancedFargateService(this, 'Svc', {
      cluster,
      cpu: 256,
      memoryLimitMiB: 512,
      desiredCount: 2,
      taskImageOptions: {
        image: ContainerImage.fromRegistry('public.ecr.aws/nginx/nginx:stable'),
        containerPort: 80,
        logDriver: new AwsLogDriver({ streamPrefix: 'app' })
      },
      publicLoadBalancer: true
    });
  }
}
```

### 10.2 CDK로 EKS 클러스터(요약)
```ts
import * as cdk from 'aws-cdk-lib';
import * as eks from 'aws-cdk-lib/aws-eks';
import { Vpc } from 'aws-cdk-lib/aws-ec2';

export class EksStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string) {
    super(scope, id);
    const vpc = Vpc.fromLookup(this, 'Vpc', { isDefault: false });
    const cluster = new eks.Cluster(this, 'Cluster', {
      vpc,
      version: eks.KubernetesVersion.V1_29,
      defaultCapacity: 2
    });
    cluster.addManifest('App', /* k8s objects */);
  }
}
```

---

## 11. 요약 테이블

| 항목 | ECS | EKS |
|---|---|---|
| 학습 곡선 | 낮음 | 높음 |
| 배포 속도 | 빠름 | 중간 |
| 생태계 | 제한적(필요 충분) | 매우 풍부(Helm/Mesh/GitOps) |
| 커스터마이징 | 제한적 | 매우 유연 |
| 서버리스 | Fargate | Fargate Profile |
| 권한 모델 | Task Role | IRSA + RBAC |
| 오토스케일 | 서비스/용량 제공자 | HPA + (CA/Karpenter) |
| 추천 케이스 | 단순 MSA, 빠른 운영 | 대규모/복잡/멀티클러스터/하이브리드 |

---

## 12. 결론

- **ECS**는 **빠른 가치실현과 운영 단순성**이 강점 — Fargate와 결합하면 서버 관리 부담이 사실상 0에 수렴.  
- **EKS**는 **표준 쿠버네티스 생태계와 유연성**을 최대화 — 대규모/복잡한 조직에 적합.

운영의 핵심은 **보안/관측/스케일/비용**을 모두 코드화해 **일관성** 있게 유지하는 것.  
위 패턴(네트워킹·권한·배포전략·오토스케일·관측·IaC)을 조합하면 **프로덕션급 컨테이너 플랫폼**을 안정적으로 구축·운영할 수 있습니다.