---
layout: post
title: Kubernetes - ConfigMap vs Secret
date: 2025-04-11 19:20:23 +0900
category: Kubernetes
---
# Kubernetes ConfigMap vs Secret

## 개요

쿠버네티스에서 애플리케이션 설정과 민감정보를 컨테이너 이미지와 분리하여 관리하는 것은 중요한 모범 사례입니다. **ConfigMap**과 **Secret**은 이 목적을 위해 설계된 핵심 리소스 오브젝트입니다. 둘 다 키-값 쌍 형태의 데이터를 저장하고, 이를 Pod에 환경변수나 파일 형태로 주입할 수 있습니다.

## 공통점

ConfigMap과 Secret은 다음과 같은 공통점을 공유합니다:
- **목적**: 애플리케이션에 **외부 구성 데이터를 제공**합니다.
- **주입 방법**: 데이터를 **환경변수**로 설정하거나, **파일(또는 디렉토리)로 마운트**하는 방식으로 Pod에 전달할 수 있습니다.
- **선언적 관리**: YAML 파일로 정의되며, Git과 같은 버전 관리 시스템으로 관리할 수 있습니다.
- **네임스페이스 범위**: 둘 다 네임스페이스(Namespace)에 속하는 리소스입니다.

> **중요한 차이점**: 환경변수로 주입된 값은 **Pod이 재시작되기 전까지 업데이트되지 않습니다.** 반면, 파일로 마운트된 경우, kubelet이 주기적으로 파일 내용을 갱신하여 변경 사항을 반영할 수 있습니다. 이때 애플리케이션이 파일 변경을 감지하고 핫 리로드(Hot Reload)를 지원해야 무중단 업데이트가 가능합니다.

---

## ConfigMap: 비민감 설정 관리

ConfigMap은 애플리케이션 설정, 구성 파일, 환경 플래그 등 **민감하지 않은 일반 데이터**를 저장하는 데 사용됩니다.

### ConfigMap 생성 예시 (YAML)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  # 간단한 키-값 쌍
  APP_ENVIRONMENT: "production"
  LOG_LEVEL: "info"
  MAX_THREADS: "10"
  
  # 전체 파일 내용을 하나의 키 값으로 (멀티라인 문자열)
  application.yaml: |
    server:
      port: 8080
    database:
      host: postgres-service
      name: appdb
```

### Pod에 ConfigMap 데이터 주입하기

ConfigMap 데이터는 여러 방식으로 Pod에 주입될 수 있습니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-demo-pod
spec:
  containers:
  - name: demo-container
    image: nginx:alpine
    # 1. 전체 ConfigMap을 환경변수로 일괄 주입
    envFrom:
    - configMapRef:
        name: app-config
        
    # 2. 특정 키만 개별적으로 환경변수로 주입
    env:
    - name: LOG_LEVEL_FROM_KEY
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: LOG_LEVEL
          
    # 3. ConfigMap의 데이터를 파일로 마운트
    volumeMounts:
    - name: config-volume
      mountPath: /etc/app-config
      
  volumes:
  - name: config-volume
    configMap:
      name: app-config
      # 특정 키만 파일로 생성하거나, 파일명을 지정할 수 있습니다.
      items:
      - key: application.yaml
        path: application.yaml
```

### ConfigMap의 주요 특성

- **바이너리 데이터**: `binaryData` 필드를 사용하여 바이너리 파일(예: 이미지, 컴파일된 설정)을 Base64 인코딩 문자열로 저장할 수 있습니다.
- **불변성(Immutable)**: `immutable: true` 속성을 설정하면 ConfigMap을 수정할 수 없게 만들 수 있습니다. 이는 보안을 강화하고, kubelet이 ConfigMap 변경을 감시하는 부하를 줄여 성능을 향상시킵니다.
- **크기 제한**: 클러스터 버전과 구성에 따라 다르지만, 일반적으로 1MB 미만의 데이터를 저장하는 데 적합합니다.
- **파일 권한**: 파일로 마운트할 때 `defaultMode` 필드로 파일 권한(예: `0444` - 읽기 전용)을 설정할 수 있습니다.

---

## Secret: 민감정보 관리

Secret은 패스워드, OAuth 토큰, SSH 키, TLS 인증서와 같은 **민감한 정보**를 저장하고 관리하는 데 특화되었습니다.

### Secret 생성 예시 (YAML)

Secret 데이터는 Base64로 인코딩된 문자열로 저장됩니다.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque  # 임의의 사용자 정의 데이터를 위한 타입
data:
  # Base64 인코딩된 값 (echo -n 'value' | base64)
  DATABASE_PASSWORD: cGFzc3dvcmQxMjM=  # "password123"
  API_TOKEN: ZXhhbXBsZS10b2tlbg==       # "example-token"
```

보다 편리하게 평문으로 작성하려면 `stringData` 필드를 사용할 수 있습니다. 쿠버네티스가 이를 자동으로 Base64 인코딩하여 저장합니다.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret-plain
type: Opaque
stringData:  # Base64 인코딩 필요 없음
  DATABASE_PASSWORD: password123
  API_TOKEN: example-token
```

### Pod에 Secret 데이터 주입하기

Secret 데이터도 ConfigMap과 동일한 방식으로 Pod에 주입할 수 있습니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-demo-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    # 1. 전체 Secret을 환경변수로 일괄 주입
    envFrom:
    - secretRef:
        name: app-secret
    # 2. 특정 키만 환경변수로 주입
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: app-secret
          key: DATABASE_PASSWORD
    # 3. Secret 데이터를 파일로 마운트 (읽기 전용 권장)
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: secret-volume
    secret:
      secretName: app-secret
      defaultMode: 0400  # 소유자만 읽을 수 있도록
```

### Secret의 주요 특성 및 보안 고려사항

- **타입**: Secret은 용도에 따라 여러 타입을 가질 수 있습니다.
    - `Opaque`: 기본 타입으로 임의의 사용자 정의 데이터입니다.
    - `kubernetes.io/dockerconfigjson`: 프라이빗 Docker 레지스트리 인증 정보를 저장합니다.
    - `kubernetes.io/tls`: TLS 통신에 사용되는 인증서와 비밀 키를 저장합니다.
    - `kubernetes.io/service-account-token`: 서비스 어카운트 토큰입니다.
- **Base64 인코딩은 암호화가 아닙니다**: Secret의 데이터는 단순히 Base64로 인코딩된 평문입니다. 보안을 위해서는 반드시 **etcd 암호화(at-rest encryption)** 를 활성화해야 합니다.
- **접근 제어**: RBAC(Role-Based Access Control)를 통해 Secret에 대한 접근 권한을 엄격히 제한해야 합니다. `kubectl get secret` 명령의 출력은 일부 도구에서 마스킹되지만, YAML 출력 시에는 원본 값이 노출될 수 있습니다.
- **권장 사용법**: 가능하면 환경변수보다 **파일로 마운트**하는 방식을 사용하세요. 환경변수는 프로세스 목록, 로그, 또는 컨테이너 내부의 `/proc` 파일 시스템을 통해 쉽게 노출될 수 있습니다.

---

## ConfigMap vs Secret 핵심 비교

| 항목 | ConfigMap | Secret |
|------|-----------|---------|
| **주요 용도** | 애플리케이션 설정, 환경 변수, 구성 파일 등 **민감하지 않은 데이터** | 패스워드, API 토큰, 인증서 등 **민감한 정보** |
| **데이터 저장 형식** | 평문 텍스트 | Base64 인코딩된 텍스트 (단순 인코딩, 암호화 아님) |
| **접근 제어** | RBAC로 관리 가능 | **강화된 RBAC와 etcd 저장소 암호화가 필수** |
| **CLI 출력 시** | 평문으로 표시됨 | 값이 마스킹되거나 `****`로 표시될 수 있음 |
| **Pod 주입 방법** | 환경변수, 파일 마운트 모두 가능 | 환경변수, 파일 마운트 모두 가능 (파일 마운트 권장) |
| **모범 사례** | 파일 마운트를 통한 동적 리로딩 활용 | 최소 권한 원칙, 외부 비밀 관리 도구(Vault 등)와 연동 고려 |

---

## 고급 사용 패턴과 모범 사례

### 1. Projected Volume을 사용한 다중 소스 통합

여러 ConfigMap과 Secret의 데이터를 하나의 디렉토리 트리로 통합하여 마운트할 수 있습니다.

```yaml
volumes:
- name: all-config
  projected:
    sources:
    - configMap:
        name: app-config
        items:
        - key: application.yaml
          path: config/application.yaml
    - secret:
        name: app-secret
        items:
        - key: DATABASE_PASSWORD
          path: secrets/db-password.txt
```

### 2. `subPath`를 이용한 단일 파일 마운트 (주의 필요)

ConfigMap이나 Secret의 특정 키를 컨테이너의 특정 파일 경로에 마운트할 수 있습니다.

```yaml
volumeMounts:
- name: config-volume
  mountPath: /app/config/database.yaml  # 단일 파일로 마운트
  subPath: database.yaml                # 볼륨 내의 파일명
```

> **주의**: `subPath`를 사용하여 마운트된 파일은 ConfigMap/Secret이 업데이트되어도 **자동으로 갱신되지 않습니다**. 동적 업데이트가 필요한 경우, 디렉토리 전체를 마운트하는 방식을 사용하세요.

### 3. 설정 변경 시 자동 롤아웃 유도

Deployment의 Pod 템플릿에 ConfigMap이나 Secret의 내용 해시(checksum)를 주석(annotation)으로 추가하면, 설정이 변경될 때마다 Pod 템플릿이 바뀌어 자동 롤링 업데이트가 트리거됩니다. 이는 Helm이나 Kustomize에서 일반적으로 사용되는 패턴입니다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  template:
    metadata:
      annotations:
        # ConfigMap 내용의 해시를 주석으로 추가
        checksum/config: {{ include "configmap.hash" . }}
    spec:
      containers:
      - name: app
        image: myapp:latest
        volumeMounts:
        - name: config
          mountPath: /etc/config
      volumes:
      - name: config
        configMap:
          name: app-config
```

### 4. 보안 강화를 위한 외부 도구 활용

- **HashiCorp Vault / AWS Secrets Manager / Azure Key Vault / GCP Secret Manager**: 이러한 외부 비밀 관리 도구와 쿠버네티스를 연동하여 Secret의 수명 주기, 로테이션, 감사를 중앙에서 관리할 수 있습니다. **External Secrets Operator (ESO)** 는 이를 자동화하는 인기 있는 도구입니다.
- **Sealed Secrets**: 공개키 암호화를 사용하여 Secret을 안전하게 Git에 저장할 수 있게 해주는 도구입니다. 암호화는 클러스터 외부에서, 복호화는 클러스터 내부의 컨트롤러에서만 수행됩니다.
- **SOPS (Secrets OPerationS)**: YAML, JSON, ENV 등 다양한 형식의 파일을 KMS, PGP, Age 키로 암호화하여 Git에 저장할 수 있는 도구입니다.

### 5. 불변성(Immutable) 활용

변경이 거의 없는 설정의 경우, ConfigMap과 Secret을 불변으로 표시하여 성능과 안정성을 높일 수 있습니다.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: static-config
immutable: true  # 한번 생성되면 수정 불가
data:
  COMPANY_NAME: "MyCorp"
  SUPPORT_EMAIL: "support@mycorp.com"
```

---

## 문제 해결 가이드

| 증상 | 가능한 원인 | 확인 및 해결 방법 |
|------|-------------|-------------------|
| **Pod에서 ConfigMap/Secret 키를 찾을 수 없음** | 키 이름 오타, 네임스페이스 불일치 | `kubectl describe pod <pod-name>`으로 이벤트 및 마운트 상태 확인. 키 이름과 네임스페이스가 정확한지 재확인. |
| **파일 내용이 업데이트되지 않음** | 환경변수로 주입되었거나 `subPath` 사용 | 환경변수 방식은 Pod 재시작 필요. `subPath` 대신 전체 디렉토리를 마운트하여 동적 업데이트 활성화. |
| **파일 권한 오류** | 기본 파일 모드가 애플리케이션 요구사항과 맞지 않음 | ConfigMap/Secret 볼륨 정의에서 `defaultMode`를 조정 (예: `0444`). |
| **Secret 값이 노출될 위험** | 과도한 RBAC 권한, `kubectl get secret -o yaml` 남용 | 최소 권한 원칙으로 RBAC 역할 구성. etcd 암호화 활성화. 외부 비밀 관리 도구 도입 고려. |
| **프라이빗 이미지 풀 실패** | 이미지 풀 Secret이 생성되지 않았거나 Pod에 연결되지 않음 | `kubernetes.io/dockerconfigjson` 타입의 Secret을 생성하고, Pod의 `imagePullSecrets` 필드에 연결. |
| **etcd에 평문 Secret 저장** | 클러스터 수준의 암호화 구성이 활성화되지 않음 | `EncryptionConfiguration` 파일을 구성하여 API 서버의 etcd 암호화를 활성화. |

---

## 결론

ConfigMap과 Secret은 쿠버네티스에서 애플리케이션 구성과 민감정보를 효과적으로 분리하고 관리할 수 있게 해주는 필수 도구입니다. 올바른 사용을 위해서는 다음 원칙을 기억하세요:

1.  **명확한 구분**: **ConfigMap은 비밀번호, 토큰, 키가 아닌 일반 설정용**, **Secret은 모든 민감정보용**으로 엄격히 구분하여 사용하세요.
2.  **보안 강화**: Secret 사용 시 RBAC를 통한 최소 권한 부여, etcd 저장소 암호화, 그리고 가능한 외부 비밀 관리 도구(Vault, AWS Secrets Manager 등)와의 연동을 고려하세요.
3.  **적절한 주입 방법 선택**: 동적 업데이트가 필요하면 **파일 마운트**를, 재시작이 허용되는 간단한 설정은 **환경변수**를 사용하세요. Secret은 보안상 파일 마운트가 더 안전합니다.
4.  **변경 관리 전략 수립**: 애플리케이션이 핫 리로드를 지원한다면 파일 마운트를 통한 무중단 업데이트를, 그렇지 않다면 체크섬 기반의 자동 롤아웃 트리거를 구현하세요.
5.  **도구 적극 활용**: Kustomize의 `secretGenerator`, Helm의 체크섬 패턴, 또는 Sealed Secrets/External Secrets Operator와 같은 도구를 활용하여 GitOps 워크플로우에서 보안과 편의성을 동시에 확보하세요.

올바르게 구성된 ConfigMap과 Secret 관리는 더 안전하고, 유연하며, 재현 가능한 쿠버네티스 애플리케이션 배포의 기반이 됩니다.