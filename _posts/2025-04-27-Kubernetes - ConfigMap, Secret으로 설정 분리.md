---
layout: post
title: Kubernetes - ConfigMap, Secret으로 설정 분리
date: 2025-04-27 19:20:23 +0900
category: Kubernetes
---
# Kubernetes에서 ConfigMap / Secret으로 설정 분리하기

## 왜 설정을 분리해야 하나?

| 이유 | 설명 |
|---|---|
| 환경별 분리 | 동일 이미지를 `dev/stage/prod`에 배포하되, 설정은 환경마다 다르게 주입 |
| 보안 경계 | 비밀(자격증명, 토큰)은 코드/이미지 이외 경로로 관리 |
| 운영 민첩성 | 설정만 바꿔도 롤링 없이(또는 최소 롤링) 동작 변경 가능 |
| 선언적 관리 | YAML로 이력 추적, 리뷰/승인, 재현 가능한 배포(GitOps) |

---

## 오브젝트 개요 — ConfigMap vs Secret

- **ConfigMap**: 평문 구성값(비민감). 텍스트 설정, 플래그, 엔드포인트, 기능 토글 등.
- **Secret**: 민감정보. 기본적으로 **base64 인코딩** 저장(암호화 아님). etcd **암호화 설정**으로 저장 시 암호화 가능.

요약 비교:

| 항목 | ConfigMap | Secret |
|---|---|---|
| 용도 | 일반 설정 | 자격증명/키/토큰/인증서 |
| 저장 포맷 | 평문 | base64 인코딩(`data`), 평문 입력(`stringData`) |
| kubectl 출력 | 마스킹 없음 | 기본 마스킹(일부 출력 제한) |
| etcd 암호화 | 별도 설정 필요 | 동일하나 보안상 **반드시** 권장 |
| 주입 방식 | env, volume, args | env, volume, args |
| 크기 제한 | 오브젝트 최대 ~1MiB 권고(에티시디/아피서버 오버헤드 고려) | 동일 |

---

## 기본 예제(보강)

### ConfigMap (일반 설정)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  labels:
    app.kubernetes.io/name: sample
data:
  APP_MODE: "production"
  LOG_LEVEL: "info"
  FEATURE_X_ENABLED: "true"
  # 파일 형태로 투하할 수도 있다(멀티라인)
  application.properties: |
    http.port=8080
    cache.size=1024
```

### Secret (민감 정보)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
stringData:
  DB_USER: admin
  DB_PASSWORD: s3cr3tpass
  # PEM, JSON 등 바이너리는 base64 사용
  # data:
  #   tls.crt: <base64>
  #   tls.key: <base64>
```

> 참고: `stringData`는 커밋 시 평문이므로 **레포 보안**(SOPS/Sealed Secrets 등)과 분리 저장 전략을 반드시 고려한다.

---

## Pod에 주입하는 3가지 경로

### 환경 변수 일괄 주입(envFrom)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: demo-envfrom
spec:
  containers:
  - name: app
    image: ghcr.io/example/app:1.0.0
    envFrom:
    - configMapRef:
        name: app-config
    - secretRef:
        name: db-secret
```

- **충돌 규칙**: 동일 키가 중복되면 **뒤에 나오는 항목이 덮어쓰지 않는다.**(컨테이너 시작 실패) → 키 네이밍 규칙으로 방지.
- 프로세스에서 `APP_MODE`, `DB_USER` 등으로 사용.

### 개별 키 매핑(env)

```yaml
env:
- name: APP_MODE
  valueFrom:
    configMapKeyRef:
      name: app-config
      key: APP_MODE
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: db-secret
      key: DB_PASSWORD
```

- 미존재 키 시 `optional: true`를 활용해 유연성 확보 가능.

### 볼륨으로 파일 마운트

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: demo-files
spec:
  containers:
  - name: app
    image: ghcr.io/example/app:1.0.0
    volumeMounts:
    - name: cfg
      mountPath: /etc/config
    - name: sec
      mountPath: /etc/secret
      readOnly: true
  volumes:
  - name: cfg
    configMap:
      name: app-config
      # 선택 키만 파일로 투하
      items:
      - key: application.properties
        path: app.properties
  - name: sec
    secret:
      secretName: db-secret
```

- 디렉터리 내 파일 이름=키, 파일 내용=값.

---

## 변경 전파와 롤링/핫 리로드

### env 주입의 특성

- **환경변수는 컨테이너 시작 시에만** 주입됨 → **ConfigMap/Secret 변경 ≠ 자동 적용**.
- 재시작/롤링이 필요.

#### 패턴 A: 애노테이션 해시를 템플릿에 삽입(권장)

Helm/Kustomize에서 ConfigMap/Secret의 해시를 Pod 템플릿 애노테이션에 주입하여 **값이 바뀌면 자동 롤링**.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  selector:
    matchLabels: { app: web }
  template:
    metadata:
      labels: { app: web }
      annotations:
        # 예: Kustomize의 configMapGenerator/secretGenerator가 생성한 해시
        checksum/config: "d2f9e9b4..."
        checksum/secret: "a1b2c3d4..."
    spec:
      containers:
      - name: web
        image: ghcr.io/example/web:2.1.0
        envFrom:
        - configMapRef: { name: app-config }
        - secretRef: { name: db-secret }
```

### 파일 마운트의 특성

- **볼륨으로 마운트된 파일은 값 변경 시 kubelet이 파일을 갱신**(수 초 지연).
- 애플리케이션이 해당 파일을 **감시(inotify)** 하거나 주기적 재로딩 로직이 있으면 **무중단 핫 리로드** 가능.

#### 패턴 B: SIGHUP/HTTP 핫 리로드 엔드포인트

- 컨테이너 내 사이드카(예: reloader) 또는 `ConfigMap` 변경 감지 → 앱에 SIGHUP/HTTP `/reload` 호출.

```yaml
lifecycle:
  postStart:
    exec:
      command: ["sh","-c","inotifywait -m /etc/config -e modify | while read; do curl -sf http://127.0.0.1:8080/reload || true; done"]
```

> 주의: 도구 설치/권한, 안정성 고려. 운영환경에서는 검증된 사이드카(예: stakater/Reloader)나 애플리케이션 자체 리로드 기능이 더 안전하다.

---

## 값 형식과 제약

- 모든 값은 문자열로 간주. 정수/불리언은 앱에서 파싱.
- **크기**: 오브젝트당 수백 KB–1MiB 내를 권장(과대하면 apiserver/etcd 성능 저하).
- **바이너리**: `Secret.data`에 base64. ConfigMap은 `binaryData`도 지원.
- **불변화(immutability)**: 잦은 변경이 필요 없는 설정은 `immutable: true`로 실수 방지 및 APIServer 부하 감소.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: stable-config
immutable: true
data:
  TOGGLE: "off"
```

---

## 보안 심화 — Secret 안전하게 쓰기

### 최소 권한(RBAC)

- Secret 읽기 권한을 **네임스페이스/리소스 단위로 최소화**.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: read-db-secret
  namespace: prod
rules:
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["db-secret"]
  verbs: ["get"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-binds-db-secret
  namespace: prod
subjects:
- kind: ServiceAccount
  name: app-sa
  namespace: prod
roleRef:
  kind: Role
  name: read-db-secret
  apiGroup: rbac.authorization.k8s.io
```

- 워크로드에 **전용 ServiceAccount**를 부여하고 필요한 Secret만 허용.

### etcd 저장 암호화(서버사이드)

- kube-apiserver의 `EncryptionConfiguration`으로 Secret을 **저장 시 암호화**.

예시(클러스터 관리 영역):
```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources: ["secrets"]
  providers:
  - aescbc:
      keys:
      - name: key1
        secret: <base64-encoded-32-byte-key>
  - identity: {}
```

> 운영환경에서는 **반드시** 활성화. 키 로테이션/백업 정책 포함.

### 노출면 최소화

- Secret을 **환경변수**로 노출하면 프로세스 크래시 덤프/프로파일에서 유출 가능. 가능하면 **파일 마운트**가 더 안전.
- 컨테이너 로그에 비밀을 출력하지 말 것.
- `kubectl describe`/`get -o yaml` 권한을 제한.

### 외부 비밀 소스 연계(CSI Driver)

- 클라우드 KMS/Secrets Manager(AWS/GCP/Azure)와 동기화하려면 **Secrets Store CSI Driver** 사용.
- 장점: Secret을 etcd에 저장하지 않거나, 동기화 주기를 제어 가능.

간단한 예(개념):
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: csi-secrets-demo
spec:
  serviceAccountName: app-sa
  volumes:
  - name: secrets-store-inline
    csi:
      driver: secrets-store.csi.k8s.io
      readOnly: true
      volumeAttributes:
        secretProviderClass: aws-sm-example
  containers:
  - name: app
    image: ghcr.io/example/app:1.0.0
    volumeMounts:
    - name: secrets-store-inline
      mountPath: "/mnt/secrets-store"
      readOnly: true
```

---

## 실전 운영 패턴

### Kustomize(해시 유도 롤아웃)

`kustomization.yaml`:
```yaml
resources:
- deployment.yaml

configMapGenerator:
- name: app-config
  literals:
  - APP_MODE=prod
  - LOG_LEVEL=info
  behavior: create

secretGenerator:
- name: db-secret
  literals:
  - DB_USER=admin
  - DB_PASSWORD=s3cr3t
  behavior: create

generatorOptions:
  disableNameSuffixHash: false   # 이름 뒤 해시를 붙여 롤아웃 유도
```

- 생성된 이름(`app-config-<hash>`)이 바뀌면 Deployment 템플릿의 참조도 바뀌며 **자동 롤링**.

### Helm(템플릿 값 → CM/Secret)

`values.yaml`:
```yaml
config:
  appMode: prod
  logLevel: info
secret:
  dbUser: admin
  dbPassword: s3cr3t
```

템플릿에서 주입 후, `.Values` 변경 시 checksum 애노테이션을 통해 롤아웃.

{% raw %}
```yaml
metadata:
  annotations:
    checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
    checksum/secret: {{ include (print $.Template.BasePath "/secret.yaml") . | sha256sum }}
```
{% endraw %}

### GitOps(SOPS/Sealed Secrets)

- 레포에 비밀을 평문으로 두지 않기 위해 **SOPS**(KMS/PGP 기반 파일 암호화) 또는 **Sealed Secrets**(클러스터 퍼블릭 키로 암호화) 사용.
- 파이프라인에서 복호화/적용.

---

## 네임스페이스/레이블/셀렉터와의 결합

- 환경별 네임스페이스(`dev`,`stage`,`prod`)로 Secret/ConfigMap을 분리.
- `app`, `env`, `component` 레이블을 통일하여 **정책/검색/감사** 용이화.

```yaml
metadata:
  labels:
    app.kubernetes.io/name: sample
    app.kubernetes.io/instance: sample-prod
    app.kubernetes.io/component: web
    app.kubernetes.io/part-of: sample-suite
    app.kubernetes.io/version: "2.1.0"
```

---

## 트러블슈팅

| 증상 | 원인 | 해결 |
|---|---|---|
| 설정을 바꿨는데 앱이 그대로 | env 주입은 런타임 재주입 안 됨 | 롤링 유도(해시 애노테이션) 또는 파일 마운트 + 핫리로드 |
| 컨테이너 시작 실패 “not found key” | 누락된 키 | `optional: true` 혹은 값 기본치 제공, 배포 순서 검증 |
| 예상치 못한 값 충돌 | envFrom 키 충돌 | 키 네이밍 규칙, env 개별 매핑, 정적 스키마 도입 |
| 성능 저하/에러 “etcd large object” | 과도한 크기 | 설정 분할/압축/외부 저장소(오브젝트 스토리지)로 이전 |
| 비밀 노출 | 권한 과다, 로그 누출 | RBAC 최소화, 로깅 필터링, 감사(audit) 활성화 |

확인 명령:
```bash
kubectl get cm,secret
kubectl describe cm app-config
kubectl describe secret db-secret
kubectl exec -it deploy/web -- env | grep -E 'APP_MODE|DB_'
kubectl exec -it deploy/web -- ls -l /etc/config /etc/secret
```

---

## 고급 주제

### Projected Volume로 한 디렉터리에 합치기

ConfigMap/Secret/DownwardAPI/ServiceAccount 토큰을 한 디렉터리에 투영.

```yaml
volumes:
- name: projected
  projected:
    sources:
    - configMap:
        name: app-config
    - secret:
        name: db-secret
```

### subPath로 파일만 선택 마운트

```yaml
volumeMounts:
- name: cfg
  mountPath: /app/config/app.properties
  subPath: app.properties
```

### 컨피그 스키마 검증

- 애플리케이션 기동 시 스키마 검증 실패 → `readiness=false`로 유지해 **잘못된 설정이 트래픽에 노출되지 않도록**.

---

## 실전 통합 예시(Deployment + CM/Secret + 롤링)

```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: web-config
data:
  APP_MODE: "prod"
  LOG_LEVEL: "info"
  application.yaml: |
    server:
      port: 8080
    feature:
      x: true
---
apiVersion: v1
kind: Secret
metadata:
  name: web-secret
type: Opaque
stringData:
  DB_USER: "admin"
  DB_PASSWORD: "s3cr3t"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  selector:
    matchLabels: { app: web }
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  template:
    metadata:
      labels: { app: web }
      annotations:
        checksum/config: "d2f9e9b4..."   # 템플릿 도구에서 주입
        checksum/secret: "a1b2c3d4..."
    spec:
      serviceAccountName: web-sa
      containers:
      - name: web
        image: ghcr.io/example/web:2.1.0
        ports:
        - containerPort: 8080
        envFrom:
        - configMapRef: { name: web-config }
        - secretRef:    { name: web-secret }
        volumeMounts:
        - name: cfg
          mountPath: /etc/web
        - name: sec
          mountPath: /etc/secret
          readOnly: true
        readinessProbe:
          httpGet: { path: /readyz, port: 8080 }
          initialDelaySeconds: 5
          periodSeconds: 5
          failureThreshold: 3
        livenessProbe:
          httpGet: { path: /livez, port: 8080 }
          initialDelaySeconds: 20
          periodSeconds: 10
          failureThreshold: 3
      volumes:
      - name: cfg
        configMap:
          name: web-config
          items:
          - key: application.yaml
            path: app.yaml
      - name: sec
        secret:
          secretName: web-secret
```

---

## 명령어 실습(요약)

```bash
# 생성

kubectl create configmap app-config \
  --from-literal=APP_MODE=dev \
  --from-literal=LOG_LEVEL=debug

kubectl create secret generic db-secret \
  --from-literal=DB_USER=admin \
  --from-literal=DB_PASSWORD=mysecret

# 조회

kubectl get cm app-config -o yaml
kubectl get secret db-secret -o yaml
echo c2VjcmV0 | base64 -d

# 적용/롤링

kubectl apply -f deployment.yaml
kubectl rollout status deploy/web

# 변경 후 롤링 유도(해시 미사용 시)

kubectl rollout restart deploy/web
```

---

## 결론

- **ConfigMap**은 일반 설정, **Secret**은 민감 데이터를 다룬다.
- **env 주입은 재시작 필요**, **파일 마운트는 실시간 갱신 + 핫리로드 패턴**으로 무중단 적용이 가능하다.
- 대규모/협업 환경에서는 **RBAC 최소권한, etcd 암호화, GitOps(Helm/Kustomize + 해시 기반 롤아웃), 외부 비밀 연계(CSI/SOPS/Sealed)** 를 조합해 **보안과 가용성**을 동시에 달성하라.
- 설정 검증과 Ready 게이트를 통해 **잘못된 설정이 프로덕션 트래픽에 영향을 주지 않도록** 하는 것이 마지막 안전망이다.
