---
layout: post
title: Kubernetes - PodSecurityPolicy와 SecurityContext
date: 2025-05-22 20:20:23 +0900
category: Kubernetes
---
# PodSecurityPolicy와 SecurityContext로 보안 강화하기

Kubernetes는 **유연한 컨테이너 실행 환경**을 제공하지만,  
기본 설정만으로는 **루트 권한**, **호스트 접근**, **권한 상승** 등이 허용될 수 있습니다.

보안 사고를 방지하기 위해, Kubernetes에서는 다음과 같은 메커니즘을 제공합니다:

- `PodSecurityPolicy` (PSP) *(v1.25부터 deprecated)*
- `SecurityContext`
- `PodSecurity Admission` (PSA) *(PSP의 대체)*

---

## ✅ SecurityContext란?

Pod 또는 Container 수준에서 보안 설정을 정의할 수 있는 **Kubernetes 오브젝트의 필드**입니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  containers:
  - name: nginx
    image: nginx
    securityContext:
      runAsUser: 1000
      runAsNonRoot: true
      readOnlyRootFilesystem: true
```

### 주요 필드

| 필드 | 설명 |
|------|------|
| `runAsUser` | 지정된 UID로 실행 |
| `runAsNonRoot` | UID가 0(root)이면 거부 |
| `privileged` | `true`이면 호스트 커널 권한 부여 |
| `allowPrivilegeEscalation` | `sudo`, `setuid` 등 권한 상승 여부 |
| `readOnlyRootFilesystem` | 루트 파일시스템을 읽기 전용으로 설정 |
| `capabilities` | 리눅스 capabilities 부여/제거 (`NET_ADMIN`, `SYS_TIME` 등) |
| `seccompProfile` | Seccomp 프로파일 사용 |

> `securityContext`는 Pod 또는 각 Container에 개별로 적용할 수 있습니다.

---

## ✅ PodSecurityPolicy (PSP)란? *(v1.25에서 제거됨)*

`PodSecurityPolicy`는 Kubernetes에서 Pod이 생성될 때  
**어떤 보안 조건을 만족해야 허용할 것인지**를 정의하는 **정책 오브젝트**입니다.

```yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: restricted
spec:
  privileged: false
  runAsUser:
    rule: MustRunAsNonRoot
  seLinux:
    rule: RunAsAny
  fsGroup:
    rule: MustRunAs
    ranges:
    - min: 1
      max: 65535
  volumes:
  - configMap
  - emptyDir
  - secret
```

→ 이 PSP를 통해 `root 권한`, `특정 볼륨 타입`, `privileged` 여부 등을 **제어**할 수 있습니다.

> ⚠️ PSP는 Kubernetes 1.25에서 공식 **제거**되었습니다.  
> 대신 **PodSecurity Admission(PSA)** 또는 **OPA Gatekeeper**를 사용하세요.

---

## ✅ PSP → PodSecurity Admission (PSA)로 전환하기

Kubernetes 1.23부터는 PodSecurityPolicy 대신 **PodSecurity Admission** 기능이 도입되어  
**기본적인 보안 정책을 네임스페이스 단위로 적용**할 수 있습니다.

### PSA 예시 (Namespace에 라벨 설정)

```bash
kubectl label namespace dev pod-security.kubernetes.io/enforce=restricted
```

| 정책 수준 | 설명 |
|-----------|------|
| `privileged` | 거의 모든 설정 허용 |
| `baseline` | 일반적인 보안 정책 수준 (default) |
| `restricted` | root 방지, 권한 상승 금지 등 가장 엄격 |

→ `restricted`를 적용하면 `runAsNonRoot: true`, `readOnlyRootFilesystem: true` 등이 강제됨

---

## ✅ 실전 예제: SecurityContext로 안전한 컨테이너 실행

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hardened-app
spec:
  containers:
  - name: app
    image: myapp:latest
    securityContext:
      runAsNonRoot: true
      allowPrivilegeEscalation: false
      capabilities:
        drop: ["ALL"]
      readOnlyRootFilesystem: true
```

→ root 사용자 사용 불가, 권한 상승 불가, 쓰기 방지된 루트 파일시스템 설정

---

## ✅ 실전 예제: PSA로 네임스페이스 보호

```bash
kubectl label namespace prod pod-security.kubernetes.io/enforce=restricted
kubectl label namespace prod pod-security.kubernetes.io/enforce-version=v1.26
```

→ `prod` 네임스페이스에서 root 사용자 사용 Pod 배포 시 거부됨

---

## ✅ 보안 수준별 체크리스트

| 항목 | 내용 |
|------|------|
| 사용자 권한 | `runAsNonRoot`, `runAsUser` 지정 |
| 파일시스템 보호 | `readOnlyRootFilesystem: true` |
| 권한 상승 방지 | `allowPrivilegeEscalation: false` |
| 특수 권한 제거 | `capabilities.drop: ["ALL"]` |
| privileged 여부 | `privileged: false` 명시 |

---

## ✅ 정책 강제화 도구들

| 도구 | 설명 |
|------|------|
| PodSecurity Admission | 기본 제공 Admission Controller (K8s 1.23+) |
| OPA Gatekeeper | CRD 기반 보안 정책 구성 |
| Kyverno | YAML 기반 정책 적용 도구 |
| PSP (deprecated) | 1.25 이하에서만 사용 가능 |

---

## ✅ 결론

| 항목 | 설명 |
|------|------|
| `SecurityContext` | Pod/Container 단위 보안 설정 |
| `PodSecurityPolicy` | 세부 정책 정의 (1.25에서 제거됨) |
| `PodSecurity Admission` | 기본 보안 정책 (restricted, baseline 등) |
| 추천 | PSA + SecurityContext 병행 설정 |
| 고급 보안 정책 | OPA Gatekeeper, Kyverno 활용 |

> Kubernetes는 유연한 만큼, 안전하지 않으면 위험합니다.  
> 최소 권한 원칙과 강제 보안 정책으로 **기본 방어선**을 구축해야 합니다.

---

## ✅ 참고 링크

- [SecurityContext 공식 문서](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)
- [PodSecurityPolicy 공식 문서 (deprecated)](https://kubernetes.io/docs/concepts/policy/pod-security-policy/)
- [PodSecurity Admission 소개](https://kubernetes.io/docs/concepts/security/pod-security-admission/)
- [OPA Gatekeeper](https://open-policy-agent.github.io/gatekeeper/)