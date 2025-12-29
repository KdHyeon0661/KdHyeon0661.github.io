---
layout: post
title: Kubernetes - ConfigMap, Secret으로 설정 분리
date: 2025-04-27 19:20:23 +0900
category: Kubernetes
---
# Kubernetes에서 ConfigMap과 Secret으로 설정 분리하기

## 설정 분리의 중요성

애플리케이션 설정을 컨테이너 이미지와 분리하는 것은 현대적인 클라우드 네이티브 운영의 핵심 원칙입니다. 이는 다음과 같은 여러 이점을 제공합니다:

- **환경별 배포**: 동일한 애플리케이션 이미지를 개발, 스테이징, 프로덕션 등 다양한 환경에 배포하면서 각 환경에 맞는 설정값(예: 데이터베이스 엔드포인트, 기능 플래그)만 변경하여 적용할 수 있습니다.
- **보안 강화**: 데이터베이스 비밀번호, API 토큰과 같은 민감 정보를 소스 코드나 이미지 레이어에 포함시키지 않고, 보안이 강화된 별도의 메커니즘을 통해 관리할 수 있습니다.
- **운영 유연성**: 애플리케이션을 재배포하지 않고도 설정값을 변경할 수 있어, 운영 중인 서비스에 대한 변경이 더 빠르고 안전해집니다.
- **선언적 관리**: 모든 설정을 YAML 파일로 정의하고 버전 관리 시스템(Git)에 저장함으로써 변경 이력을 추적하고, 재현 가능한 배포를 구현할 수 있습니다.

쿠버네티스는 이러한 요구사항을 충족시키기 위해 **ConfigMap**과 **Secret**이라는 두 가지 핵심 리소스를 제공합니다.

## ConfigMap vs Secret: 기본 비교

| 항목 | ConfigMap | Secret |
|------|-----------|---------|
| **주요 용도** | 비밀번호, 토큰, 키가 **아닌** 일반 애플리케이션 설정 관리 | 패스워드, OAuth 토큰, SSH 키, TLS 인증서 등 **민감 정보** 관리 |
| **데이터 저장 방식** | 평문 텍스트로 저장 | 기본적으로 Base64로 인코딩된 텍스트로 저장 (`stringData`로 평문 입력 가능) |
| **CLI 출력 시** | 값이 평문으로 표시됨 | 값이 마스킹되거나 `****`로 표시됨 |
| **보안 고려사항** | etcd 암호화 권장 | **etcd 저장소 암호화 필수**, RBAC를 통한 엄격한 접근 제어 권장 |
| **데이터 주입 방법** | 환경변수, 파일 마운트, 커맨드라인 인자 | 환경변수, 파일 마운트, 커맨드라인 인자 (파일 마운트가 더 안전) |
| **크기 제한** | 오브젝트당 대략 1MiB 이내 권장 (성능 고려) | 오브젝트당 대략 1MiB 이내 권장 (성능 고려) |

> **핵심 차이**: ConfigMap은 **설정(configuration)** 을, Secret은 **비밀(secret)** 정보를 위한 것입니다. Secret의 Base64 인코딩은 단순한 인코딩이지 암호화가 아니므로, **etcd 저장소 암호화를 반드시 활성화**해야 합니다.

## 기본 사용법 예제

### ConfigMap 생성 예시

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  # 단순 키-값 쌍
  APP_ENVIRONMENT: "production"
  LOG_LEVEL: "info"
  MAX_CONNECTIONS: "100"
  
  # 파일 형태의 멀티라인 구성 (예: 속성 파일, YAML, JSON)
  application.properties: |
    server.port=8080
    cache.enabled=true
    feature.new-ui=false
```

### Secret 생성 예시

`stringData` 필드를 사용하면 Base64 변환 없이 평문으로 작성할 수 있습니다. 쿠버네티스가 자동으로 인코딩합니다.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: database-credentials
type: Opaque  # 사용자 정의 데이터 타입
stringData:
  DB_USERNAME: "app_user"
  DB_PASSWORD: "VeryS3cur3P@ssw0rd!"
  API_KEY: "sk_live_abc123def456"
```

## Pod에 설정 주입하는 주요 방법

### 1. 환경변수로 일괄 주입 (`envFrom`)

ConfigMap이나 Secret의 모든 키-값 쌍을 환경변수로 변환하여 Pod에 주입합니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app-container
    image: myapp:latest
    envFrom:
    - configMapRef:
        name: app-config      # app-config의 모든 키가 환경변수가 됨
    - secretRef:
        name: database-credentials  # Secret의 모든 키도 환경변수가 됨
```

### 2. 개별 키를 환경변수로 주입 (`env`)

특정 키만 선택적으로 환경변수로 매핑할 수 있습니다.

```yaml
    env:
    - name: DATABASE_PASSWORD  # Pod 내부에서 사용할 환경변수명
      valueFrom:
        secretKeyRef:
          name: database-credentials  # 참조할 Secret 이름
          key: DB_PASSWORD            # Secret 내의 특정 키
          optional: true              # 키가 없어도 Pod 시작 실패하지 않음
    - name: LOG_LEVEL
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: LOG_LEVEL
```

### 3. 파일로 마운트 (가장 유연한 방식)

ConfigMap이나 Secret의 데이터를 컨테이너의 파일 시스템 경로에 파일로 마운트합니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app-container
    image: myapp:latest
    volumeMounts:
    - name: config-volume
      mountPath: /etc/app/config
    - name: secret-volume
      mountPath: /etc/app/secrets
      readOnly: true  # Secret은 읽기 전용으로 마운트 권장
      
  volumes:
  - name: config-volume
    configMap:
      name: app-config
      items:  # 특정 키만 파일로 생성
      - key: application.properties
        path: application.properties
  - name: secret-volume
    secret:
      secretName: database-credentials
      defaultMode: 0400  # 파일 권한 설정 (소유자만 읽기)
```

## 설정 변경과 롤아웃 전략

ConfigMap이나 Secret의 내용이 변경되었을 때, 이를 실행 중인 애플리케이션에 어떻게 적용할지 결정해야 합니다.

### 환경변수 방식의 제한 사항

**중요**: 환경변수로 주입된 값은 **Pod이 재시작되기 전까지 업데이트되지 않습니다.** 설정을 변경하려면 Pod를 재생성해야 하며, 이는 일반적으로 Deployment의 롤링 업데이트를 의미합니다.

### 파일 마운트 방식의 동적 업데이트

파일로 마운트된 ConfigMap/Secret은 **kubelet에 의해 주기적으로 동기화**됩니다. 변경 사항은 수초에서 수십 초 내에 컨테이너의 파일 시스템에 반영됩니다. 그러나 애플리케이션이 이를 인지하고 새 설정을 적용하려면 추가 작업이 필요합니다:

1.  **애플리케이션 자체 리로드 기능**: 애플리케이션이 설정 파일 변경을 감지(`inotify` 등)하거나 주기적으로 파일을 다시 읽어서 설정을 갱신하도록 구현되어 있어야 합니다.
2.  **외부 시그널 전송**: `SIGHUP` 시그널을 보내거나 애플리케이션의 `/reload` HTTP 엔드포인트를 호출하여 설정을 다시 로드하도록 할 수 있습니다. (주의: 구현 복잡성 증가)

### 권장 패턴: 설정 변경 시 자동 롤링 업데이트

실제 운영에서는 설정 변경 시 자동으로 Pod를 재시작하여 새로운 설정을 적용하는 방식을 선호합니다. 이는 **Helm**이나 **Kustomize**와 같은 도구를 사용해 쉽게 구현할 수 있습니다.

**Kustomize 예시 (`kustomization.yaml`)**:
```yaml
resources:
- deployment.yaml

configMapGenerator:
- name: app-config
  literals:
  - LOG_LEVEL=info
  - APP_ENV=production

secretGenerator:
- name: app-secrets
  literals:
  - DB_PASSWORD=secret123

generatorOptions:
  disableNameSuffixHash: false  # ConfigMap/Secret 이름 뒤에 해시를 붙임
```
Kustomize는 ConfigMap/Secret 내용이 변경될 때마다 새로운 해시를 생성하여 오브젝트 이름(예: `app-config-abc123`)을 변경합니다. Deployment가 이 이름을 참조하므로, 설정이 바뀌면 참조하는 오브젝트 이름도 바뀌어 Pod 템플릿이 변경된 것으로 인식되어 자동 롤링 업데이트가 트리거됩니다.

## 고급 보안 관행

### 1. 최소 권한 원칙 (RBAC)
Secret에 접근해야 하는 Pod에만 필요한 최소한의 권한을 부여하세요. 이를 위해 전용 ServiceAccount를 생성하고, 해당 ServiceAccount에 특정 Secret에 대한 `get` 권한만 부여하는 Role과 RoleBinding을 구성합니다.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: read-db-secret
  namespace: production
rules:
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["database-credentials"]  # 이 특정 Secret만
  verbs: ["get"]  # 읽기 권한만
```

### 2. etcd 저장 암호화 활성화
민감 정보가 평문으로 etcd에 저장되는 것을 방지하기 위해, 클러스터 수준에서 **etcd 암호화**를 구성해야 합니다. 이는 `EncryptionConfiguration` 파일을 통해 kube-apiserver에 설정합니다.

### 3. 외부 비밀 관리 도구와 연동
대규모 또는 엔터프라이즈 환경에서는 HashiCorp Vault, AWS Secrets Manager, Azure Key Vault, Google Secret Manager와 같은 전문 비밀 관리 도구를 쿠버네티스와 연동하는 것을 고려하세요. **Secrets Store CSI Driver**나 **External Secrets Operator**와 같은 도구가 이 연동을 도와줍니다.

**장점**:
- 중앙 집중식 비밀 관리 및 로테이션
- etcd에 민감 정보 저장 방지
- 세분화된 접근 정책과 감사 로그

## 모범 사례 요약

1.  **명확한 분리**: ConfigMap은 **일반 설정**, Secret은 **모든 민감 정보**용으로 엄격히 구분하여 사용하세요.
2.  **주입 방법 선택**: 동적 업데이트가 필요하거나 보안이 우선이라면 **파일 마운트** 방식을, 간단한 설정은 환경변수 방식을 사용하세요. Secret은 가능하면 파일로 마운트하세요.
3.  **변경 관리 자동화**: 설정 변경 시 자동 롤아웃을 위해 Helm이나 Kustomize의 해시 기반 네이밍 기능을 활용하세요.
4.  **보안 강화**: Secret 사용 시 RBAC로 접근을 제한하고, etcd 암호화를 활성화하며, 가능하다면 외부 비밀 관리 도구를 도입하세요.
5.  **설정 검증**: 애플리케이션 시작 시 설정값의 유효성을 검증하고, 문제가 있을 경우 `readinessProbe`를 실패 처리하여 잘못된 설정의 Pod가 트래픽을 받지 않도록 하세요.

## 문제 해결 명령어

```bash
# ConfigMap과 Secret 목록 조회
kubectl get configmaps
kubectl get secrets

# 특정 ConfigMap/Secret의 상세 내용 확인 (Secret 값은 마스킹됨)
kubectl describe configmap app-config
kubectl describe secret database-credentials

# Secret의 Base64 인코딩된 값 디코딩 (주의: 민감 정보 노출)
kubectl get secret database-credentials -o jsonpath='{.data.DB_PASSWORD}' | base64 -d

# Pod의 환경변수 확인
kubectl exec <pod-name> -- env | grep DB_

# Pod에 마운트된 설정 파일 확인
kubectl exec <pod-name> -- ls -la /etc/app/config/
kubectl exec <pod-name> -- cat /etc/app/config/application.properties

# 설정 변경 후 Deployment 롤링 재시작 (해시 기반 자동화가 없을 때)
kubectl rollout restart deployment/<deployment-name>
```

## 결론

ConfigMap과 Secret은 쿠버네티스 애플리케이션의 구성 관리를 근본적으로 변화시킨 강력한 도구입니다. 올바르게 사용하면 다음과 같은 이점을 얻을 수 있습니다:

- **일관성과 재현성**: 모든 환경에서 동일한 이미지를 사용하면서 환경별 차이는 설정으로 관리합니다.
- **보안 강화**: 민감 정보를 코드베이스에서 분리하고, 암호화와 접근 제어를 통해 보호합니다.
- **운영 효율성**: 설정 변경에 대한 빠르고 안전한 배포 주기를 구현합니다.

이러한 이점을 완전히 실현하기 위해서는 단순히 ConfigMap과 Secret을 사용하는 것을 넘어, **RBAC를 통한 접근 제어, etcd 암호화, GitOps를 통한 선언적 관리, 그리고 외부 비밀 관리 도구와의 연동**까지 종합적인 보안 및 운영 전략을 수립하고 적용하는 것이 중요합니다. 설정 관리가 단순한 기술 선택이 아닌, 애플리케이션의 안정성, 보안성, 그리고 운영 효율성의 핵심 기반이 될 수 있도록 체계적으로 접근하시기 바랍니다.