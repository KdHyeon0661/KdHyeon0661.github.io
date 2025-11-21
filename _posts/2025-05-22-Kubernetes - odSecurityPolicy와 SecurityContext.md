---
layout: post
title: Kubernetes - PodSecurityPolicy와 SecurityContext
date: 2025-05-22 20:20:23 +0900
category: Kubernetes
---
# PodSecurityPolicy와 SecurityContext로 보안 강화하기

## 큰 그림: “정책”과 “실행 프로파일”의 역할 분리

- **정책(Policy)**: *허용 경계*를 정의하고, 위반 시 **거부/경고/감사**. (PSA, Gatekeeper, Kyverno)
- **실행 프로파일(SecurityContext)**: 파드/컨테이너 **실제 런타임 속성**(사용자/권한/커널 인터페이스)을 명시.

```
[Dev/CI] ── 생성된 Pod/Deployment ──► [Admission: PSA/Gatekeeper/Kyverno]
                                              │ (enforce/audit/warn)
                                              ▼
                                       [API Server 저장]
                                              ▼
                                       [Kubelet 실행]
                                              │
                               [SecurityContext / seccomp / AppArmor / SELinux]
```

---

## SecurityContext — 런타임 보안 설정의 핵심

`Pod.spec.securityContext` (파드 수준)와 `Pod.spec.containers[*].securityContext` (컨테이너 수준) 두 레벨에서 설정합니다.
컨테이너 수준이 **보다 구체적**이며, 파드 수준 기본값을 **오버라이드**합니다.

### 컨테이너 수준 예제 (권장 최소권한 템플릿)

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
        drop: ["ALL"]         # 필요 시 add에 최소 권한만 추가
        # add: ["NET_BIND_SERVICE"]  # 예: 1024 미만 포트 바인딩 필요 시
      readOnlyRootFilesystem: true
      seccompProfile:
        type: RuntimeDefault  # 또는 Localhost (커스텀 프로파일)
```

### 파드 수준 공통 값 (fsGroup, supplementalGroups, sysctls)

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
      type: "spc_t"           # 환경에 맞게 조정(일반적으로 PSA-restricted에선 RunAsAny가 아님)
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

> **필드별 의의**
> - `runAsNonRoot` + `runAsUser`: 루트(UID 0) 금지.
> - `allowPrivilegeEscalation: false`: `setuid`/`sudo` 경로 차단.
> - `privileged: false`: 호스트 커널에 거의 무제한 접근을 막음.
> - `capabilities.drop: ["ALL"]`: 리눅스 capability 기본 제거, 필요한 것만 `add`로 부여.
> - `readOnlyRootFilesystem: true`: 루트 FS를 읽기 전용으로 하여 파일 변조/랜섬 행위 방지.
> - `seccompProfile`: 시스템 콜 표면 최소화. **`RuntimeDefault`** 권장.

### 볼륨·호스트 관련 위험 옵션

- `hostNetwork: false`, `hostPID: false`, `hostIPC: false` 유지.
- `hostPath` 볼륨 사용 금지(정말 필요한 경우 **경로 제한** 및 PSA/Gatekeeper로 통제).
- `privileged` 컨테이너 금지.
- `procMount: Default` 유지(커스텀은 위험).

---

## — 제거 배경과 교훈

- PSP는 **Admission Controller**로, 파드 생성 시 보안 조건을 검사/거부.
- **v1.25에서 제거(Deprecated→Removed)**: UX 복잡성·권한 모델 혼란·운영 난이도.
- **교훈**: 정책은 *간명한 기본 가드레일* + *고급 시나리오는 외부 정책엔진*으로.

> **대체**:
> - 기본선은 **PodSecurity Admission(PSA)** (Kubernetes 내장)
> - 세밀/커스터마이징은 **OPA Gatekeeper** 또는 **Kyverno**

---

## — PSP의 내장 대체

**네임스페이스 라벨**로 정책 수준을 지정합니다.

### 정책 레벨

| 레벨 | 의도 | 대략적 특성 |
|---|---|---|
| `privileged` | 거의 모든 것 허용 | 테스트/특수 목적 |
| `baseline` | 일반적 워크로드 최소 보호 | 일부 위험 기능 제한 |
| `restricted` | 가장 엄격(권장) | 루트 금지, 권한상승 금지, 호스트 접근 차단 등 |

### 세 가지 모드(라벨 키)

- **enforce**: 위반 시 **거부**
- **audit**: 승인하되 **감사 로그** 기록
- **warn**: 승인하되 **CLI 경고** 출력

```bash
# prod 네임스페이스에 restricted 강제(거부)

kubectl label namespace prod \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=v1.27

# 동일 네임스페이스에서 baseline 위반은 경고, privileged 위반은 감사만

kubectl label namespace prod \
  pod-security.kubernetes.io/warn=baseline \
  pod-security.kubernetes.io/warn-version=v1.27 \
  pod-security.kubernetes.io/audit=privileged \
  pod-security.kubernetes.io/audit-version=v1.27
```

> **버전 라벨**(`*-version`)은 **정책 스키마** 버전을 고정해,
> 클러스터 업그레이드 시 **예상치 못한 정책 변화**를 방지합니다.

### PSA로 막히는 대표 케이스(Restricted)

- `runAsUser: 0` 또는 `runAsNonRoot: false`
- `privileged: true`
- `hostNetwork/hostPID/hostIPC: true`
- 위험한 `hostPath` 볼륨 (허용 목록 밖)
- `allowPrivilegeEscalation: true`
- seccomp/AppArmor 누락(버전에 따라 요구 수준 상이)

---

## 실패/성공 예제로 이해하는 PSA + SecurityContext

### 실패 예 — restricted 네임스페이스에서 루트/권한상승 시도

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
      runAsNonRoot: false       # ❌
      allowPrivilegeEscalation: true  # ❌
      privileged: true          # ❌
```

**결과:** Admission 단계에서 **거부**. (kubectl에 에러 및 PSA 위반 항목 출력)

### 성공 예 — 최소권한 + 읽기전용 루트

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

## / Kyverno — PSA로 부족한 “세밀 제어” 채우기

### Gatekeeper (ConstraintTemplate + Constraint)

**정책(레고 블록)을 코딩**해 재사용·버전관리. 예: `hostPath` 경로 화이트리스트만 허용.

```yaml
# ConstraintTemplate: hostPath 경로 제한

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

### Kyverno (정책을 YAML로 선언)

예: 모든 컨테이너에 `runAsNonRoot: true` 강제하고 없으면 **자동 패치**(mutate).

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

> Gatekeeper는 **검증 중심**, Kyverno는 **검증+변이**에 강함.
> PSA(기본선) + Kyverno/Gatekeeper(세밀/변이) 조합이 실무에서 흔합니다.

---

## seccomp / AppArmor / SELinux — 커널 인터페이스 최소화

### seccomp

- **`RuntimeDefault`**: 컨테이너 런타임 기본 보안 프로파일 사용(권장).
- `Localhost`로 커스텀 JSON 프로파일을 노드에 배치 후 참조 가능.

```yaml
securityContext:
  seccompProfile:
    type: RuntimeDefault
# 또는
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

### SELinux (플랫폼 따라 상이)

```yaml
securityContext:
  seLinuxOptions:
    user: "system_u"
    role: "system_r"
    type: "spc_t"
    level: "s0"
```

> PSA restricted 레벨에선 특정 SELinux/ seccomp 요건이 있을 수 있으니 **정책 버전**에 맞추세요.

---

## 위험한 볼륨/옵션 식별과 대체 전략

| 항목 | 위험 | 대체/완화 |
|---|---|---|
| `hostPath` | 호스트 파일시스템 직접 접근 | CSI(PVC) 사용, Gatekeeper/Kyverno로 경로 제한 |
| `privileged: true` | 커널 직접 접근 | 필요한 cap만 `add`, eBPF/NET_ADMIN 등 최소화 |
| `hostNetwork/hostPID/hostIPC` | 네임스페이스 격리 우회 | 가능한 `false` 유지, 정말 필요시 별도 네임스페이스/노드풀로 격리 |
| `allowPrivilegeEscalation: true` | setuid 등으로 권한 상승 | `false`로 고정 |
| `capabilities add: ["ALL"]` | 과도한 권한 | `drop: ["ALL"]`, 필요한 소수 cap만 추가 |

---

## PSA 전환 전략 (PSP → PSA 마이그레이션)

1. **현황 수집**: 기존 파드 스펙에서 위험 옵션 사용 조사(`jq`/`kubectl`/폴리시 스캐너).
2. **샌드박스**: `warn=`/`audit=`부터 적용하여 영향 범위 확인.
3. **수정**: 워크로드 템플릿(Helm/Kustomize)에서 `securityContext` 보강.
4. **강제(enforce=baseline→restricted)**: 단계적 상향.
5. **세밀 정책**: Gatekeeper/Kyverno로 예외/추가 규칙 관리.
6. **지속 검증**: PR 게이트(정적 스캔) + 런타임 감사(PSA audit, Audit Log, Falco 등).

---

## 운영 점검/디버깅 레시피

### 빠른 스캔(예시)

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

### PSA 위반 원인 파악

- **거부 메시지**에 구체 항목이 나옵니다(예: `hostNetwork` 금지, `runAsNonRoot` 필요 등).
- `kubectl describe ns <ns>`로 PSA 라벨 확인.
- `warn`/`audit` 모드 활성화해 **사전 탐지**.

---

## “안전한 기본 템플릿” 3종 (복붙용)

### API 서버 뒤 단순 웹앱 (Restricted 호환)

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
        emptyDir: {}   # 상태 필요시 PVC로 교체
```

### 네트워크 유틸 필요(1024 미만 포트 바인딩)

```yaml
securityContext:
  runAsNonRoot: true
  allowPrivilegeEscalation: false
  capabilities:
    drop: ["ALL"]
    add: ["NET_BIND_SERVICE"]
```

### 로그 사이드카(읽기전용 루트 + 임시 쓰기 경로만)

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

## 체크리스트 요약

- [ ] 네임스페이스에 **PSA 라벨** 설정: `enforce=restricted`(운영), `warn/audit` 병행.
- [ ] 모든 워크로드에 **`runAsNonRoot: true`** + **비루트 UID/GID** 명시.
- [ ] **`allowPrivilegeEscalation: false`**, **`privileged: false`**.
- [ ] **`capabilities.drop: ["ALL"]`**, 필요한 최소 cap만 `add`.
- [ ] **`readOnlyRootFilesystem: true`**, 쓰기 필요 경로는 **emptyDir/PVC**로 분리.
- [ ] **seccompProfile: RuntimeDefault**(가능하면 전면 적용).
- [ ] `hostNetwork/hostPID/hostIPC` 금지, `hostPath` 금지 또는 엄격 제한.
- [ ] 예외/세밀 규칙은 **Gatekeeper/Kyverno**로 캡슐화(코드 리뷰/PR 게이트).
- [ ] 정기 스캔과 **Audit/경고**로 지속 검증.

---

## FAQ

**Q. PSA만으로 충분한가요?**
A. 기본선엔 충분하지만, 조직별/앱별 예외와 변이(자동 패치)가 필요하면 **Kyverno**, 검증 규칙의 코드화가 필요하면 **Gatekeeper**를 병행하세요.

**Q. seccomp/AppArmor는 꼭 써야 하나요?**
A. **예(권장)**. 시스템 콜/프로세스 격리 표면을 줄여 **취약점 악용 가능성**을 크게 낮춥니다.

**Q. 루트 권한이 필요한 드문 워크로드는요?**
A. 별도 네임스페이스/노드풀로 격리, PSA는 `baseline` 또는 제한적 예외, Gatekeeper/Kyverno로 **명시적 화이트리스트**를 사용하세요.

---

## 참고

- SecurityContext: <https://kubernetes.io/docs/tasks/configure-pod-container/security-context/>
- PodSecurity Admission: <https://kubernetes.io/docs/concepts/security/pod-security-admission/>
- PSP (Deprecated): <https://kubernetes.io/docs/concepts/policy/pod-security-policy/>
- OPA Gatekeeper: <https://open-policy-agent.github.io/gatekeeper/>
- Kyverno: <https://kyverno.io/>
