---
layout: post
title: Kubernetes - ConfigMap vs Secret
date: 2025-04-11 19:20:23 +0900
category: Kubernetes
---
# Kubernetes ConfigMap vs Secret

애플리케이션을 컨테이너로 배포할 때, 환경 설정 값(설정 파일, 포트 번호, 외부 서비스 주소 등)이나 민감한 정보(DB 비밀번호, API 키 등)를 외부에서 주입해야 하는 경우가 많습니다.

Kubernetes에서는 이러한 외부 설정을 다음 두 가지 오브젝트로 관리합니다:

- **ConfigMap**: 일반 설정 값 저장  
- **Secret**: 민감한 정보를 인코딩하여 저장

이 글에서는 이 둘의 차이점, 사용법, 예제를 비교해봅니다.

---

## ✅ 1. 공통점

| 항목 | 설명 |
|------|------|
| 용도 | Pod에 환경 설정을 주입 |
| 사용 방식 | 환경 변수, 파일 마운트, 커맨드라인 인자 등 |
| 선언적 관리 | YAML로 선언 및 버전관리 가능 |
| 동적 주입 가능 | Rolling update 없이 변경 가능 (단, 일부는 재시작 필요) |

---

## ✅ 2. ConfigMap이란?

ConfigMap은 **비밀번호나 민감하지 않은 일반 설정 값을 key-value 형태로 저장**하는 오브젝트입니다.

### 🧾 예시: YAML 정의

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-config
data:
  APP_ENV: production
  LOG_LEVEL: info
  MAX_CONNECTIONS: "100"
```

### 🛠 사용 방법

#### 1) 환경 변수로 주입

```yaml
spec:
  containers:
  - name: app
    image: myapp
    envFrom:
    - configMapRef:
        name: my-config
```

#### 2) 파일로 마운트

```yaml
volumeMounts:
  - name: config-volume
    mountPath: /etc/config
volumes:
  - name: config-volume
    configMap:
      name: my-config
```

→ `/etc/config/APP_ENV`, `/etc/config/LOG_LEVEL` 파일이 생성됨

---

## ✅ 3. Secret이란?

Secret은 **비밀번호, 토큰, 인증서 등 민감한 정보를 저장**하는 오브젝트입니다.  
`Base64 인코딩`으로 저장되며, 디코딩 없이 바로 사용 가능하도록 설계되어 있습니다.

### 🧾 예시: YAML 정의

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
type: Opaque
data:
  DB_USER: YWRtaW4=        # "admin"의 base64
  DB_PASSWORD: cGFzc3dvcmQ= # "password"의 base64
```

> ⚠️ `data` 필드는 **base64 인코딩된 값**이어야 합니다.  
> 텍스트로 직접 작성하려면 `stringData`를 사용할 수 있습니다.

### 🛠 사용 방법

#### 1) 환경 변수로 주입

```yaml
envFrom:
  - secretRef:
      name: my-secret
```

#### 2) 파일로 마운트

```yaml
volumes:
  - name: secret-volume
    secret:
      secretName: my-secret
volumeMounts:
  - name: secret-volume
    mountPath: /etc/secret
    readOnly: true
```

---

## ✅ 4. ConfigMap vs Secret: 차이점 비교

| 항목 | ConfigMap | Secret |
|------|-----------|--------|
| 용도 | 설정값, 환경변수 | 비밀번호, 토큰, 인증서 등 |
| 저장 형식 | 일반 텍스트 | base64 인코딩 (보안 우선) |
| 기본 인코딩 | ❌ (평문 저장) | ✅ base64 |
| 보안 수준 | 낮음 | 높음 (RBAC + 암호화 가능) |
| 자동 마스킹 | ❌ | ✅ (kubectl 출력 시) |
| 파일 마운트 가능 여부 | ✅ | ✅ |
| 환경 변수 주입 가능 여부 | ✅ | ✅ |
| 암호화 저장 (etcd 기준) | ❌ | ✅ (기본 설정 시 암호화 지원) |

---

## ✅ 5. 실습 명령어 예시

### 1) ConfigMap 생성

```bash
kubectl create configmap my-config --from-literal=APP_ENV=dev
kubectl get configmap my-config -o yaml
```

### 2) Secret 생성

```bash
kubectl create secret generic my-secret \
  --from-literal=DB_USER=admin \
  --from-literal=DB_PASSWORD=password

kubectl get secret my-secret -o yaml
```

```bash
# base64 디코딩
echo 'YWRtaW4=' | base64 -d
```

---

## ✅ 6. 보안 팁

- Secret은 **RBAC 제어, etcd 암호화, access 제한**으로 보호 가능
- Secret은 기본적으로 `base64`만 적용되므로 **암호화가 아님**
  → etcd 암호화를 통해 보호 필요
- Secret과 ConfigMap을 모두 Mount한 경우, **Secret이 우선 적용**됨

---

## ✅ 결론

| 상황 | 사용 오브젝트 |
|------|----------------|
| 단순 설정, 포트, 환경 | ✅ ConfigMap |
| 민감한 정보, 인증 키 | ✅ Secret |

ConfigMap과 Secret은 쿠버네티스의 선언형 설정의 핵심입니다.  
애플리케이션을 보안성과 유연성을 갖춘 상태로 운영하기 위해서는 이 둘을 **적절히 구분**하여 사용하는 것이 중요합니다.