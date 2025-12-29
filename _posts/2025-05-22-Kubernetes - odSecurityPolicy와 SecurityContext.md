---
layout: post
title: Kubernetes - PodSecurityPolicy와 SecurityContext
date: 2025-05-22 20:20:23 +0900
category: Kubernetes
---
# PodSecurityPolicy와 SecurityContext로 Kubernetes 보안 강화하기

## 보안 정책과 런타임 설정의 역할 구분

Kubernetes 환경에서의 효과적인 보안 강화는 두 가지 수준에서 이루어집니다:

- **정책(Policy)**: 허용 가능한 보안 경계를 정의하고, 이를 위반할 경우 거부, 경고, 감사하는 규칙입니다. PodSecurity Admission(PSA), Gatekeeper, Kyverno 등이 이에 해당합니다.
- **실행 프로파일(SecurityContext)**: 실제 파드와 컨테이너의 런타임 속성(사용자 ID, 권한, 커널 인터페이스 접근 등)을 명시적으로 설정합니다.

```
[개발/CI] ── 생성된 Pod/Deployment ──► [어드미션 컨트롤러: PSA/Gatekeeper/Kyverno]
                                              │ (enforce/audit/warn 모드 적용)
                                              ▼
                                       [API 서버 저장]
                                              ▼
                                       [Kubelet 실행]
                                              │
                               [SecurityContext / seccomp / AppArmor / SELinux 적용]
```

---

## SecurityContext: 런타임 보안 설정의 핵심

SecurityContext는 파드 수준(`Pod.spec.securityContext`)과 컨테이너 수준(`Pod.spec.containers[*].securityContext`)에서 설정할 수 있습니다. 컨테이너 수준 설정이 더 구체적이며, 파드 수준의 기본값을 오버라이드합니다.

### 컨테이너 수준 SecurityContext 예제 (최소 권한 원칙 적용)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hardened-app
spec:
  containers:
  - name: app
    image: mycorp/app:1.0.0
    securityContext:
      runAsNonRoot: true
      runAsUser: 10001
      runAsGroup: 10001
      allowPrivilegeEscalation: false
      privileged: false
      capabilities:
        drop: ["ALL"]         # 모든 권한 제거 후 필요한 것만 추가
        # add: ["NET_BIND_SERVICE"]  # 특수 권한이 필요한 경우만 추가
      readOnlyRootFilesystem: true
      seccompProfile:
        type: RuntimeDefault  # 또는 Localhost (커스텀 프로파일)
```

### 파드 수준 SecurityContext 설정 (공통 값 관리)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hardened-pod
spec:
  securityContext:
    fsGroup: 2000
    fsGroupChangePolicy: "OnRootMismatch"
    supplementalGroups: [2001, 2002]
    seLinuxOptions:
      type: "spc_t"           # 환경에 맞게 조정 필요
    sysctls:
    - name: net.ipv4.ip_unprivileged_port_start
      value: "0"
  containers:
  - name: app
    image: mycorp/app:2.0.0
    securityContext:
      runAsNonRoot: true
      allowPrivilegeEscalation: false
      capabilities: { drop: ["ALL"] }
      readOnlyRootFilesystem: true
```

### 주요 SecurityContext 필드의 보안 의미

| 필드 | 보안 의미 | 권장값 |
|---|---|---|
| `runAsNonRoot` + `runAsUser` | 루트(UID 0) 권한 실행 방지 | `runAsNonRoot: true`, 비루트 UID 지정 |
| `allowPrivilegeEscalation` | 권한 상승 경로 차단 | `false` |
| `privileged` | 호스트 커널 무제한 접근 방지 | `false` |
| `capabilities.drop` | 리눅스 capability 기본 제거 | `["ALL"]` |
| `readOnlyRootFilesystem` | 루트 파일 시스템 변조 방지 | `true` |
| `seccompProfile` | 시스템 콜 접근 최소화 | `RuntimeDefault` |

### 위험한 볼륨 및 호스트 관련 옵션 관리

- `hostNetwork`, `hostPID`, `hostIPC`는 가능하면 `false`로 유지
- `hostPath` 볼륨 사용은 최소화하고, 필요한 경우 PSA나 Gatekeeper를 통해 엄격히 제한
- `privileged` 컨테이너는 가능한 한 사용하지 않음
- `procMount`는 기본값(`Default`) 유지

---

## PodSecurityPolicy(PSP)의 역사와 교훈

### PSP의 역할과 제거 배경

PSP는 어드미션 컨트롤러로, 파드 생성 시 보안 조건을 검사하고 위반 시 거부하는 역할을 했습니다. 그러나 v1.25에서 완전히 제거되었는데, 그 이유는 다음과 같습니다:

- **사용자 경험의 복잡성**: PSP는 복잡한 권한 모델과 구성으로 인해 운영이 어려웠습니다.
- **권한 모델의 혼란**: 사용자와 PSP 간의 관계가 직관적이지 않았습니다.
- **운영 난이도**: 실제 환경에서의 적용과 관리가 복잡했습니다.

### PSP 대체 솔루션

이제 PSP 대신 다음과 같은 솔루션들을 사용해야 합니다:

- **기본 정책**: PodSecurity Admission(PSA) - Kubernetes 내장 기능
- **고급/세밀한 정책**: OPA Gatekeeper 또는 Kyverno

---

## PodSecurity Admission(PSA): PSP의 내장 대체 솔루션

PSA는 네임스페이스 라벨을 통해 정책 수준을 지정하는 간단하면서도 효과적인 접근 방식을 제공합니다.

### PSA 정책 레벨

| 레벨 | 목적 | 주요 특징 |
|---|---|---|
| `privileged` | 모든 권한 허용 | 테스트 또는 특수 목적용 |
| `baseline` | 일반 워크로드 기본 보호 | 일부 위험 기능 제한 |
| `restricted` | 가장 엄격한 보안 (권장) | 루트 실행 금지, 권한 상승 차단, 호스트 접근 제한 등 |

### PSA 적용 모드

| 모드 | 동작 | 용도 |
|---|---|---|
| **enforce** | 위반 시 **거부** | 프로덕션 환경 보안 강제 |
| **audit** | 승인하되 **감사 로그** 기록 | 보안 위반 모니터링 |
| **warn** | 승인하되 **CLI 경고** 출력 | 개발/테스트 환경 경고 |

### PSA 라벨 설정 예시

```bash
# 프로덕션 네임스페이스에 restricted 정책 강제 적용
kubectl label namespace prod \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=v1.27

# 동일 네임스페이스에서 다양한 정책 레벨 적용
kubectl label namespace prod \
  pod-security.kubernetes.io/warn=baseline \
  pod-security.kubernetes.io/warn-version=v1.27 \
  pod-security.kubernetes.io/audit=privileged \
  pod-security.kubernetes.io/audit-version=v1.27
```

> **버전 라벨 중요성**: `*-version` 라벨을 통해 정책 스키마 버전을 고정하면, 클러스터 업그레이드 시 예상치 못한 정책 변화를 방지할 수 있습니다.

### PSA restricted 레벨에서 차단되는 주요 사항

- `runAsUser: 0` 또는 `runAsNonRoot: false`
- `privileged: true`
- `hostNetwork`, `hostPID`, `hostIPC: true`
- 제한되지 않은 `hostPath` 볼륨 사용
- `allowPrivilegeEscalation: true`
- seccomp 또는 AppArmor 프로파일 누락 (버전에 따라 다름)

---

## PSA와 SecurityContext 실전 예제

### PSA 위반 예제 (restricted 네임스페이스)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: bad-pod
  namespace: prod   # prod는 enforce=restricted
spec:
  containers:
  - name: app
    image: nginx
    securityContext:
      runAsNonRoot: false       # PSA 위반
      allowPrivilegeEscalation: true  # PSA 위반
      privileged: true          # PSA 위반
```

**결과**: 어드미션 단계에서 거부되며, kubectl 명령어에 PSA 위반 항목이 출력됩니다.

### PSA 준수 예제 (최소 권한 원칙 적용)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: good-pod
  namespace: prod
spec:
  securityContext:
    fsGroup: 2000
  containers:
  - name: app
    image: mycorp/app:1.2.3
    securityContext:
      runAsNonRoot: true
      runAsUser: 10001
      runAsGroup: 10001
      allowPrivilegeEscalation: false
      capabilities:
        drop: ["ALL"]
      readOnlyRootFilesystem: true
      seccompProfile:
        type: RuntimeDefault
```

---

## 고급 정책 제어: Kyverno와 Gatekeeper

PSA로 부족한 세밀한 제어가 필요할 때는 Kyverno나 Gatekeeper와 같은 외부 정책 엔진을 사용할 수 있습니다.

### Kyverno 예제: 모든 컨테이너에 runAsNonRoot 강제 적용

Kyverno는 YAML 기반 정책 정의와 변이(mutate) 기능에 강점이 있습니다.

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: enforce-nonroot
spec:
  rules:
  - name: set-runAsNonRoot
    match:
      resources:
        kinds: ["Pod"]
    mutate:
      patchStrategicMerge:
        spec:
          containers:
          - (name): "*"
            securityContext:
              +(runAsNonRoot): true
```

### Gatekeeper 예제: hostPath 경로 제한

Gatekeeper는 Rego 언어를 사용하여 복잡한 정책을 코드로 표현할 수 있습니다.

```yaml
# ConstraintTemplate: hostPath 경로 제한 정책 정의
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8sallowedhostpaths
spec:
  crd:
    spec:
      names:
        kind: K8sAllowedHostPaths
      validation:
        openAPIV3Schema:
          properties:
            allowedPaths:
              type: array
              items:
                type: string
  targets:
  - target: admission.k8s.gatekeeper.sh
    rego: |
      package k8sallowedhostpaths
      violation[{"msg": msg}] {
        input.review.kind.kind == "Pod"
        hostpath := input.review.object.spec.volumes[_].hostPath.path
        not allowed(hostpath, input.parameters.allowedPaths)
        msg := sprintf("hostPath %v not allowed", [hostpath])
      }
      allowed(path, allowed_paths) {
        some i
        startswith(path, allowed_paths[i])
      }
---
# Constraint: 실제 정책 적용
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sAllowedHostPaths
metadata:
  name: only-allow-var-log
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
  parameters:
    allowedPaths:
    - "/var/log/"
```

> **선택 가이드**: PSA로 기본 보안을 확보한 후, 추가적인 검증이 필요하면 Gatekeeper, 변이 기능이 필요하면 Kyverno를 고려하세요.

---

## 커널 수준 보안: seccomp, AppArmor, SELinux

컨테이너의 커널 인터페이스를 최소화하는 것은 중요한 보안 전략입니다.

### seccomp (시스템 콜 필터링)

```yaml
securityContext:
  seccompProfile:
    type: RuntimeDefault  # 컨테이너 런타임 기본 프로파일 사용
# 또는 커스텀 프로파일 사용
# seccompProfile:
#   type: Localhost
#   localhostProfile: profiles/myapp-seccomp.json
```

### AppArmor (노드 지원 필요)

```yaml
metadata:
  annotations:
    container.apparmor.security.beta.kubernetes.io/app: runtime/default
```

### SELinux (플랫폼에 따라 다름)

```yaml
securityContext:
  seLinuxOptions:
    user: "system_u"
    role: "system_r"
    type: "spc_t"
    level: "s0"
```

> PSA restricted 레벨에서는 특정 SELinux나 seccomp 요구사항이 있을 수 있으므로 정책 버전에 맞는 설정을 확인하세요.

---

## 위험한 설정 식별과 대체 전략

| 위험 항목 | 보안 위협 | 대체/완화 방안 |
|---|---|---|
| `hostPath` | 호스트 파일 시스템 직접 접근 | CSI/PVC 사용, Gatekeeper/Kyverno로 경로 제한 |
| `privileged: true` | 커널에 대한 무제한 접근 | 필요한 capability만 추가, 최소 권한 원칙 적용 |
| `hostNetwork/hostPID/hostIPC` | 네임스페이스 격리 우회 | 가능하면 `false` 유지, 필요한 경우 별도 네임스페이스/노드 풀로 격리 |
| `allowPrivilegeEscalation: true` | setuid 등을 통한 권한 상승 | `false`로 고정 |
| `capabilities add: ["ALL"]` | 과도한 권한 부여 | `drop: ["ALL"]` 적용 후 필요한 최소한의 capability만 추가 |

---

## PSP에서 PSA로의 전환 전략

1. **현황 분석**: 기존 파드 스펙에서 위험한 옵션 사용 현황 조사
2. **샌드박스 테스트**: `warn`/`audit` 모드로 시작하여 영향 범위 확인
3. **워크로드 수정**: Helm/Kustomize 템플릿에 SecurityContext 보강 적용
4. **단계적 강화**: `baseline` → `restricted`로 점진적 정책 강화
5. **예외 관리**: Gatekeeper/Kyverno로 조직별 예외 규칙 관리
6. **지속적 검증**: PR 단계 정적 분석과 런타임 감사 로그 모니터링

---

## 운영 점검과 디버깅

### 보안 구성 스캔 예제

```bash
# privileged 컨테이너 탐지
kubectl get pods -A -o json \
| jq -r '.items[]
  | select(.spec.containers[]?.securityContext?.privileged==true)
  | [.metadata.namespace,.metadata.name] | @tsv'

# hostPath 볼륨 사용 탐지
kubectl get pods -A -o json \
| jq -r '.items[]
  | select(any(.spec.volumes[]?; has("hostPath")))
  | [.metadata.namespace,.metadata.name] | @tsv'
```

### PSA 위반 문제 해결

- 거부 메시지에서 구체적인 위반 항목 확인 (예: `hostNetwork` 금지, `runAsNonRoot` 필요)
- `kubectl describe ns <네임스페이스>`로 PSA 라벨 확인
- `warn`/`audit` 모드를 활용하여 사전에 문제 탐지

---

## 안전한 기본 템플릿 모음

### 일반 웹 애플리케이션 (Restricted 호환)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: prod
spec:
  replicas: 2
  selector: { matchLabels: { app: web } }
  template:
    metadata:
      labels: { app: web }
    spec:
      securityContext:
        fsGroup: 2000
      containers:
      - name: web
        image: mycorp/web:1.0.0
        ports: [{ containerPort: 8080 }]
        securityContext:
          runAsNonRoot: true
          runAsUser: 10001
          runAsGroup: 10001
          allowPrivilegeEscalation: false
          capabilities: { drop: ["ALL"] }
          readOnlyRootFilesystem: true
          seccompProfile: { type: RuntimeDefault }
        volumeMounts:
        - name: data
          mountPath: /app/data
      volumes:
      - name: data
        emptyDir: {}   # 영구 저장이 필요한 경우 PVC로 교체
```

### 네트워크 유틸리티 (1024 미만 포트 바인딩 필요)

```yaml
securityContext:
  runAsNonRoot: true
  allowPrivilegeEscalation: false
  capabilities:
    drop: ["ALL"]
    add: ["NET_BIND_SERVICE"]  # 1024 미만 포트 바인딩을 위한 최소 권한
```

### 로그 수집 사이드카 (읽기 전용 루트 파일 시스템)

```yaml
containers:
- name: log-shipper
  image: mycorp/shipper:2.1
  securityContext:
    runAsNonRoot: true
    allowPrivilegeEscalation: false
    capabilities: { drop: ["ALL"] }
    readOnlyRootFilesystem: true
    seccompProfile: { type: RuntimeDefault }
  volumeMounts:
  - name: tmp
    mountPath: /tmp
volumes:
- name: tmp
  emptyDir: { sizeLimit: "64Mi" }
```

---

## 자주 묻는 질문

**Q. PSA만으로 충분한 보안을 확보할 수 있나요?**
A. 기본적인 보안 요구사항은 PSA로 충분히 관리할 수 있습니다. 그러나 조직별 특수 요구사항이나 자동 패치 기능이 필요하다면 Kyverno를, 복잡한 검증 규칙이 필요하다면 Gatekeeper를 추가로 사용하는 것이 좋습니다.

**Q. seccomp나 AppArmor를 꼭 사용해야 하나요?**
A. 네, 강력히 권장합니다. 시스템 콜과 프로세스 격리 수준을 제한함으로써 잠재적인 취약점 악용 가능성을 크게 줄일 수 있습니다.

**Q. 루트 권한이 정말 필요한 워크로드는 어떻게 처리하나요?**
A. 이러한 워크로드는 별도의 네임스페이스나 노드 풀로 격리하고, PSA를 `baseline`으로 설정하거나 Gatekeeper/Kyverno를 통해 명시적인 화이트리스트로 관리하는 것이 좋습니다.

---

## 결론

Kubernetes 환경에서 효과적인 보안 강화를 위해서는 다음과 같은 다층적 접근 방식을 적용해야 합니다:

1. **기본 보안 정책 수립**: PSA를 통해 네임스페이스 수준에서 기본적인 보안 정책을 강제하세요. `restricted` 레벨을 프로덕션 환경의 기본값으로 설정하는 것이 좋습니다.

2. **워크로드별 보안 설정**: 모든 파드와 컨테이너에 적절한 SecurityContext를 적용하여 최소 권한 원칙을 구현하세요. `runAsNonRoot`, `readOnlyRootFilesystem`, `seccompProfile` 같은 핵심 설정을 기본 템플릿에 포함시키세요.

3. **고급 정책 관리**: PSA로 부족한 세밀한 제어가 필요할 경우 Kyverno나 Gatekeeper를 도입하여 조직별 정책을 구현하세요. 특히 변이 기능이 필요하면 Kyverno, 복잡한 검증 로직이 필요하면 Gatekeeper를 선택하세요.

4. **지속적 모니터링과 개선**: 보안 설정을 일회성 작업이 아닌 지속적인 과정으로 관리하세요. 정기적인 보안 스캔, 감사 로그 분석, 위반 사항 모니터링을 통해 보안 상태를 지속적으로 개선하세요.

5. **점진적 적용 전략**: 기존 환경을 보호하면서 새로운 보안 정책을 도입하려면 점진적인 접근이 필요합니다. 먼저 `warn`과 `audit` 모드로 시작하여 영향도를 파악한 후, 단계적으로 `enforce` 모드로 전환하세요.

이러한 종합적인 접근 방식을 통해 Kubernetes 워크로드의 보안성을 크게 향상시키고, 컨테이너 환경에서의 공격 표면을 최소화할 수 있습니다. 보안은 단일 솔루션이 아닌 여러 계층의 방어 체계로 접근할 때 가장 효과적입니다.
