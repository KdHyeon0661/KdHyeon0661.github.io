---
layout: post
title: Kubernetes - Namespace, Label, Selector
date: 2025-04-11 19:20:23 +0900
category: Kubernetes
---
# Kubernetes 핵심 개념: Namespace, Label, Selector

- 팀/환경별 **네임스페이스 전략** + ResourceQuota/LimitRange/DNS 범위
- **표준 라벨 체계**(app.kubernetes.io/*)와 라이프사이클·토폴로지 라벨
- **Equality/Set 기반 Selector**, 컨트롤러별 불변성/가변성
- **Service ↔ Pod 라벨 정합성** 검증 루틴, 실전 명령 모음
- **Affinity/Anti-Affinity/Topology Spread**에서 라벨의 역할
- **NetworkPolicy/IngressClass** 등에서의 네임스페이스/라벨 활용
- 트러블슈팅/안전한 운영 체크리스트

---

## ✅ 1. Namespace — 클러스터 리소스의 논리적 격리

### 목적과 효과

- **멀티팀/멀티프로젝트**: 권한·리소스·네트워크 정책을 **네임스페이스 단위**로 분리.
- **리소스 가드레일**: `ResourceQuota`, `LimitRange`로 과소/과대 사용 방지.
- **이름 충돌 방지**: `dev/mypod`와 `prod/mypod`는 **서로 다른 객체**.

### 기본 네임스페이스

| 이름 | 설명 |
|---|---|
| `default` | 기본 작업 공간 |
| `kube-system` | 코어 컴포넌트(kube-dns/CoreDNS 등) |
| `kube-public` | 공개 읽기 가능한 정보(클러스터 정보 등) |
| `kube-node-lease` | 노드 하트비트(lease)용 |

### 생성/사용

```bash
kubectl create namespace dev
kubectl get ns
kubectl config set-context --current --namespace=dev
```

YAML 지정:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
  namespace: dev
spec:
  containers:
  - name: c
    image: nginx:1.27-alpine
```

### 네임스페이스 단위 가드레일

**ResourceQuota**(총량 상한) + **LimitRange**(Pod/컨테이너 기본 request/limit).

```yaml
# ResourceQuota 예: dev NS에서 총 코어/메모리/객체 수 제한

apiVersion: v1
kind: ResourceQuota
metadata:
  name: rq-standard
  namespace: dev
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    pods: "200"
    services: "50"
---
# LimitRange 예: 기본 request/limit 자동 부여

apiVersion: v1
kind: LimitRange
metadata:
  name: lr-defaults
  namespace: dev
spec:
  limits:
  - type: Container
    default:
      cpu: "1"
      memory: 1Gi
    defaultRequest:
      cpu: "250m"
      memory: 256Mi
```

### 네임스페이스와 DNS/Service 검색

- Pod에서 `svc명` → 동일 NS의 Service를 우선 조회.
- FQDN: `svc명.네임스페이스.svc.cluster.local`.
- 다른 NS 접근 시 FQDN을 쓰거나 `-n`을 명시한 `kubectl port-forward/service` 사용.

---

## ✅ 2. Label — key/value 메타데이터로 의미를 부여

### 목적

- **그룹핑/선택**의 최소 단위. Service, ReplicaSet, Deployment, NetworkPolicy 등 **거의 모든 선택 로직의 근간**.
- 운영 메타정보(팀, 환경, 릴리스, 버전, 역할)를 객체에 **가볍게** 부착.

### 문법/제약(요약)

- key는 `prefix(optional)/name` 형태. 일반적으로 `app.kubernetes.io/name`처럼 **도메인 접두사** 권장.
- value는 짧은 ASCII(영숫자, `-`, `_`, `.`), 길이 제한(일반적으로 63자 내).

### 표준 라벨 권장 세트

공식 권고 키(검색/도구 연계에 유리):

| Key | 예시 | 의미 |
|---|---|---|
| `app.kubernetes.io/name` | `payment-api` | 애플리케이션 이름 |
| `app.kubernetes.io/instance` | `payment-api-prod` | 인스턴스/릴리즈 이름(Helm release 등) |
| `app.kubernetes.io/version` | `1.4.2` | 앱 버전 |
| `app.kubernetes.io/component` | `api`/`worker` | 구성 요소 |
| `app.kubernetes.io/part-of` | `billing` | 상위 시스템 |
| `app.kubernetes.io/managed-by` | `helm`/`kustomize`/`argo` | 관리 도구 |

라이프사이클/환경/토폴로지 라벨(예시):
```yaml
metadata:
  labels:
    app.kubernetes.io/name: payment-api
    app.kubernetes.io/instance: payment-api-prod
    app.kubernetes.io/version: "1.4.2"
    env: prod
    tier: backend
    region: ap-northeast-2
    zone: ap-northeast-2a
```

### 명령으로 라벨 조작

```bash
kubectl label pod mypod team=platform
kubectl label pod mypod team-         # 라벨 제거
kubectl get pods -l app=payment-api -o wide
```

> **주의**: 라벨은 **변경 가능**하지만, **컨트롤러의 selector**가 가리키는 집합이 바뀌면 재조정이 일어난다(서비스 디스커넥트, RS 관리 집합 변화 등).

---

## ✅ 3. Selector — 라벨 기반의 정확한 선택

### 두 가지 스타일

| 유형 | 예시 | 설명 |
|---|---|---|
| **Equality** | `app=web`, `tier!=db` | 동등/부정 비교 |
| **Set-based** | `env in (dev,qa)`, `app notin (web,admin)` | 집합 포함/배제 |

CLI에서도 동일:
```bash
kubectl get pods -l 'env in (dev,qa),tier=backend'
```

### 컨트롤러별 Selector와 불변성

| 리소스 | selector 불변성 | 비고 |
|---|---|---|
| **ReplicaSet** | **불변** | 생성 후 변경 불가. Pod 집합 정의의 핵심. |
| **Deployment** | (내부 RS selector는 불변) | `.spec.selector`는 RS와 일치해야 함. |
| **Service** | **변경 가능** | 변경 즉시 엔드포인트 집합 변동. 생산환경에서는 신중히. |
| **NetworkPolicy** | 변경 가능 | Ingress/Egress 규칙 재평가. |
| **HPA/VPA/Spread** | 변경 가능 | 라벨 기준의 대상 집합 재평가. |

> **가장 흔한 실수**: 템플릿 라벨과 selector 불일치 → Service Endpoints가 비어 있거나 RS가 Pod를 관리하지 못함.

---

## ✅ 4. 실습: Namespace + Label + Selector 한 번에

### 라벨이 붙은 Pod/Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: dev
  labels:
    app.kubernetes.io/name: web
    env: dev
spec:
  replicas: 2
  selector:
    matchLabels:
      app.kubernetes.io/name: web   # selector와 template.labels 반드시 일치
  template:
    metadata:
      labels:
        app.kubernetes.io/name: web
        env: dev
        tier: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:1.27-alpine
        ports: [{ containerPort: 80 }]
```

### 해당 라벨을 고르는 Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
  namespace: dev
spec:
  selector:
    app.kubernetes.io/name: web
    tier: frontend
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

### 검증 루틴

```bash
# dev NS로 컨텍스트 고정

kubectl config set-context --current --namespace=dev

# 라벨 확인

kubectl get deploy,pod -l app.kubernetes.io/name=web -o wide

# Service→Endpoints 연결 확인 (가장 중요한 검증)

kubectl get endpoints web -o yaml

# 일시적으로 selector를 바꿔보면(주의: 실제 운영 금지) 즉시 Endpoints 변동

kubectl patch svc web -p '{"spec":{"selector":{"app.kubernetes.io/name":"not-exist"}}}'
kubectl get endpoints web                 # 비어있는지 확인
# 원복

kubectl patch svc web -p '{"spec":{"selector":{"app.kubernetes.io/name":"web","tier":"frontend"}}}'
```

---

## ✅ 5. 라벨이 좌우하는 스케줄링/분산/정책

### Affinity/Anti-Affinity — 라벨로 배치 제어

```yaml
spec:
  template:
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app.kubernetes.io/name
                operator: In
                values: ["web"]
            topologyKey: "kubernetes.io/hostname"
```
- **동일 노드**에 `web`이 몰리지 않게 **분산**(Anti-Affinity).
- `requiredDuring...`는 하드, `preferred...`는 소프트 제약.

### Topology Spread Constraints — 균등 분산

```yaml
spec:
  template:
    spec:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app.kubernetes.io/name: web
```
- 라벨 집합(`web`)의 **존 단위 균등 분산**.

### NetworkPolicy — 네임스페이스/라벨 조합

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: prod
spec:
  podSelector:
    matchLabels:
      tier: backend               # prod/backend 대상
  policyTypes: ["Ingress"]
  ingress:
  - from:
    - namespaceSelector:
        matchLabels: { env: prod }   # prod 라벨이 붙은 NS에서
      podSelector:
        matchLabels: { tier: frontend } # tier=frontend인 Pod만 허용
    ports:
    - protocol: TCP
      port: 8080
```
- **네임스페이스 라벨**(`env=prod`)과 **Pod 라벨**(`tier`)을 함께 써서 **정밀 제어**.

---

## ✅ 6. kubectl 실전 셀렉션/출력 패턴

```bash
# Set-based 다중 조건

kubectl get pods -l 'env in (dev,qa),tier=backend' -o wide

# JSONPath로 이름과 라벨 key만 추리기

kubectl get svc -l app.kubernetes.io/name=web -o=jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.labels.env}{"\n"}{end}'

# 라벨 키만 있는 리소스 찾기(값 무시)

kubectl get pods -l 'team'
```

---

## ✅ 7. 베스트 프랙티스

1) **표준 라벨 체계 고정**
   프로젝트 시작 시 `app.kubernetes.io/*` + `env`/`tier`/`team` 등 **조합을 문서화**하고, 모든 리소스에 적용.

2) **selector ↔ template.labels 정합성 테스트 자동화**
   CI에서 `Service.selector`가 **실제 Pod 템플릿 라벨**과 일치하는지 lint(Schema/OPA/Conftest 등).

3) **Service Endpoints 검증을 배포 파이프라인에 포함**
   `kubectl get ep`가 **비어 있지 않음**을 조건으로 실패 처리.

4) **네임스페이스별 가드레일**
   `ResourceQuota` + `LimitRange`로 폭주 방지. 이미지 풀 정책/PSA(보안레벨)도 NS 단위로.

5) **라벨 변경의 파급효과 인지**
   운영 중 라벨 변경은 **선택 집합**을 바꾼다. 변경 전/후 영향(서비스 연결/RS 관리)을 체크.

6) **토폴로지 라벨 통일**
   클러스터/노드 라벨(`topology.kubernetes.io/zone`, `kubernetes.io/hostname`)을 기준으로 분산 전략 일관성 유지.

---

## ✅ 8. 트러블슈팅 표

| 증상 | 1차 확인 | 원인 | 해결 |
|---|---|---|---|
| Service가 응답 없음 | `kubectl get ep <svc>` | selector-라벨 불일치 | Pod 템플릿 라벨과 Service selector 정합 |
| RS/Deployment가 Pod 관리 안 함 | `kubectl describe rs/deploy` | `.spec.selector`와 `.template.labels` 불일치 | 둘을 1:1 매칭(불변 필드 주의) |
| 특정 존에만 몰림 | `topologySpreadConstraints` 미설정 | 기본 스케줄링으로 편중 | 스프레드 제약 추가 |
| 허용된 트래픽도 차단 | `NetworkPolicy` 네임스페이스/Pod 라벨 확인 | NS 라벨 누락, 조건 불만족 | NS 라벨/Pod 라벨 정합성 수정 |
| 다른 NS 서비스 조회 실패 | 잘못된 DNS 이름 | FQDN 미사용 | `svc.ns.svc.cluster.local` 사용 |

---

## ✅ 9. 패턴 모음(운영 템플릿)

### 팀/환경 매트릭스

```yaml
metadata:
  labels:
    app.kubernetes.io/name: checkout
    app.kubernetes.io/instance: checkout-prod
    env: prod
    team: commerce
    tier: backend
```

### 서비스 라벨과 셀렉터의 교과서적 정합

```yaml
spec:
  selector:
    app.kubernetes.io/name: checkout
    tier: backend
# 아래 Pod 템플릿에도 동일하게!

```

### 네임스페이스 라벨 → 정책에 활용

```bash
kubectl label namespace prod env=prod
kubectl label namespace dev  env=dev
# NetworkPolicy, Gatekeeper/OPA, ArgoCD Project 범위 등에 재사용

```

---

## ✅ 10. 요약

| 개념 | 한 줄 정의 | 핵심 포인트 |
|---|---|---|
| **Namespace** | 리소스의 **논리적 구획/격리** | Quota/LimitRange, DNS 범위, 보안정책의 경계 |
| **Label** | 리소스에 부착하는 **의미 태그** | 표준 라벨 체계, 운영 메타정보, 라벨 변경의 영향 |
| **Selector** | 라벨을 기준으로 하는 **집합 선택** | Equality/Set 기반, 컨트롤러 불변성/가변성 이해 |

**운영 핵심**: “**표준 라벨 체계**”를 정하고, “**selector ↔ template.labels 정합성**”을 배포 파이프라인에서 **자동 검증**하라.
네임스페이스엔 **가드레일**을, 트래픽/배치/분산엔 **라벨 기반 정책**을 입히면, 클러스터는 **예측 가능하고 안전**해진다.
