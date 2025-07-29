---
layout: post
title: Kubernetes - ConfigMap, Secret으로 설정 분리
date: 2025-04-27 19:20:23 +0900
category: Kubernetes
---
# Kubernetes에서 ConfigMap / Secret으로 설정 분리하기

현대적인 애플리케이션 배포에서는 **코드와 설정을 분리**하는 것이 핵심입니다.  
Kubernetes는 이를 위해 두 가지 핵심 오브젝트를 제공합니다:

- `ConfigMap`: 일반 설정값 (비밀번호 아님)
- `Secret`: 민감한 정보 (비밀번호, 토큰 등)

---

## ✅ 왜 설정을 분리해야 하나?

| 이유 | 설명 |
|------|------|
| 환경별 설정 구분 | dev/prod에 따라 설정값 분리 가능 |
| 보안 향상 | 민감 정보는 코드 외부에서 관리 |
| 재사용성 | 같은 이미지로 여러 환경 대응 |
| 선언적 관리 | GitOps와 연계 가능 (YAML로 추적) |

---

## ✅ 1. ConfigMap 예제 (일반 설정)

### 📄 configmap.yaml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_MODE: production
  LOG_LEVEL: info
  MAX_CONNECTIONS: "100"
```

---

## ✅ 2. Secret 예제 (민감한 정보)

### 📄 secret.yaml

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
stringData:
  DB_USER: admin
  DB_PASSWORD: s3cr3tpass
```

> ⚠️ `data:`는 base64로 인코딩해야 하고, `stringData:`는 자동으로 처리됨

---

## ✅ 3. Pod에 주입하는 방법

Kubernetes는 ConfigMap/Secret을 다음과 같은 방식으로 Pod에 주입할 수 있습니다:

| 방식 | 설명 |
|------|------|
| 환경 변수 | `envFrom`, `env` |
| 볼륨 마운트 | 파일로 `/etc/config`, `/etc/secret` 등에 주입 |
| 커맨드라인 인자 | 컨테이너 시작 인자에 포함 |

---

### ✅ (1) 환경 변수로 주입하기

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: demo-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    envFrom:
    - configMapRef:
        name: app-config
    - secretRef:
        name: db-secret
```

→ 앱 내에서 `process.env.APP_MODE`, `process.env.DB_USER` 등으로 접근 가능

---

### ✅ (2) 파일로 주입하기 (마운트)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: demo-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
    - name: secret-volume
      mountPath: /etc/secret
      readOnly: true
  volumes:
  - name: config-volume
    configMap:
      name: app-config
  - name: secret-volume
    secret:
      secretName: db-secret
```

→ `/etc/config/APP_MODE`, `/etc/secret/DB_PASSWORD`와 같은 파일로 자동 생성됨

---

## ✅ 4. 명령어 실습

### ConfigMap 생성

```bash
kubectl create configmap app-config \
  --from-literal=APP_MODE=dev \
  --from-literal=LOG_LEVEL=debug
```

### Secret 생성

```bash
kubectl create secret generic db-secret \
  --from-literal=DB_USER=admin \
  --from-literal=DB_PASSWORD=mysecret
```

### 조회

```bash
kubectl get configmap app-config -o yaml
kubectl get secret db-secret -o yaml

# base64 디코딩
echo c2VjcmV0 | base64 -d
```

---

## ✅ 5. ConfigMap vs Secret 요약

| 항목 | ConfigMap | Secret |
|------|-----------|--------|
| 용도 | 일반 설정 | 민감 정보 |
| 저장 방식 | 평문 | base64 인코딩 |
| etcd 암호화 | ❌ 기본 없음 | ✅ 가능 (보안 권장) |
| 출력 시 마스킹 | ❌ | ✅ 자동 마스킹 |
| 파일 마운트 가능 | ✅ | ✅ |
| 환경변수 사용 | ✅ | ✅ |

---

## ✅ 6. 팁 & 보안 주의사항

- Secret은 반드시 RBAC로 접근 제한할 것
- Secret은 base64 인코딩일 뿐 **암호화는 아님**
  → etcd 암호화 사용 권장 (`EncryptionConfiguration`)
- GitOps/Helm 환경에서는 **stringData**와 `.Values`를 활용해 분리 관리

---

## ✅ 결론

ConfigMap과 Secret은 Kubernetes의 핵심 오브젝트로, 다음과 같은 패턴을 만들 수 있습니다:

- 코드와 설정의 완전한 분리
- 동일한 이미지로 다양한 환경 운영 가능
- 보안 정보는 코드 외부에서 안전하게 관리