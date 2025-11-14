---
layout: post
title: Kubernetes - ConfigMap vs Secret
date: 2025-04-11 19:20:23 +0900
category: Kubernetes
---
# Kubernetes ConfigMap vs Secret

## 공통점(요약)

| 항목 | 설명 |
|---|---|
| 목적 | Pod에 **외부 설정** 주입 |
| 주입 경로 | **환경변수**, **파일 마운트**, 커맨드라인 인자 조합 |
| 선언형 | YAML로 선언, Git 관리 가능(Secret은 전략 필요) |
| 동적 반영 | 파일 마운트 시 **주기적 갱신**(kubelet이 투영 볼륨 파일 갱신). 단, 애플리케이션이 **핫 리로드**를 지원해야 무중단 반영 |

> 환경변수로 넣은 값은 **컨테이너 재시작 전까지 갱신되지 않는다**. 파일 마운트가 “롤링 없이 갱신”에 유리하다.

---

## ConfigMap — 비민감 설정

### 기본 YAML

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-config
data:
  APP_ENV: production
  LOG_LEVEL: info
  MAX_CONNECTIONS: "100"
  app.conf: |
    [server]
    port = 8080
```

### Pod 주입 — envFrom / 개별 키 참조 / 파일 마운트

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cm-demo
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh","-c","env | grep -E 'APP_ENV|LOG_LEVEL|MAX_CONNECTIONS'; sleep 3600"]
    envFrom:                    # 1) 일괄 환경변수
    - configMapRef:
        name: my-config
    env:                        # 2) 개별 키
    - name: SERVER_PORT
      valueFrom:
        configMapKeyRef:
          name: my-config
          key: app.conf        # (예시) 보통은 파일로 적합
    volumeMounts:               # 3) 파일 마운트
    - name: cfg
      mountPath: /etc/config
  volumes:
  - name: cfg
    configMap:
      name: my-config
      items:
      - key: app.conf
        path: app.conf
```

### 특성/옵션

- **`binaryData`**: 바이너리(베이스64 인코딩 텍스트로 기술) 보관.
- **`immutable: true`**: 변경 불가로 만들어 성능 향상(워처/갱신 비용↓).
- 크기 제한(클러스터/버전 의존, 일반적으로 수백 KB 수준)과 키 길이 제한 유의.
- 파일 마운트 시 권한:
```yaml
volumes:
- name: cfg
  configMap:
    name: my-config
    defaultMode: 0444   # 파일 퍼미션(8진수)
```

---

## Secret — 민감정보 저장

### 기본 YAML(베이스64 `data`)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
type: Opaque
data:
  DB_USER: YWRtaW4=          # "admin"
  DB_PASSWORD: cGFzc3dvcmQ=  # "password"
```

### `stringData`로 간단 작성(생성 시 자동 base64 변환)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-secret-sd
type: Opaque
stringData:
  DB_USER: admin
  DB_PASSWORD: password
```

### Pod 주입 — envFrom / 개별 키 / 파일

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-demo
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh","-c","printenv DB_USER; printenv DB_PASSWORD; ls -l /etc/secret; sleep 3600"]
    envFrom:
    - secretRef:
        name: my-secret
    env:
    - name: ONLY_PW
      valueFrom:
        secretKeyRef:
          name: my-secret
          key: DB_PASSWORD
    volumeMounts:
    - name: sec
      mountPath: /etc/secret
      readOnly: true
  volumes:
  - name: sec
    secret:
      secretName: my-secret
      defaultMode: 0400
```

### 특성/보안 포인트

- Secret은 **RBAC**로 접근 통제. `kubectl get secret -o yaml` 출력 시 일부 UI/툴에서 **마스킹**.
- etcd **암호화(at rest)** 활성화 권장(Cluster 설정). Base64는 **암호화가 아니다**.
- 타입:
  - `kubernetes.io/dockerconfigjson`(이미지 풀 시크릿)
  - `kubernetes.io/tls`(TLS cert/key)
  - `Opaque`(기타 임의 키)

---

## ConfigMap vs Secret — 핵심 차이

| 항목 | ConfigMap | Secret |
|---|---|---|
| 주 용도 | 비민감 설정 | 민감정보(비번/토큰/인증서) |
| 저장 형식 | 평문 | base64 텍스트(평문 아님, 단순 인코딩) |
| 접근 제어 | 가능(RBAC) | **강화 필요(RBAC + at-rest 암호화)** |
| 마스킹 | 기본 없음 | **있음**(도구/CLI 출력 시) |
| 파일/ENV 주입 | 모두 가능 | 모두 가능 |
| 권장 모범 | 리로딩/큰 설정 | 최소한으로 보관, 감사/추적 필수 |

---

## 고급 주입 패턴

### projected volume(여러 소스 합쳐 마운트)

```yaml
volumes:
- name: projected
  projected:
    sources:
    - configMap:
        name: my-config
        items:
        - key: app.conf
          path: conf/app.conf
    - secret:
        name: my-secret
        items:
        - key: DB_PASSWORD
          path: secrets/db_password
```

### 하위 디렉터리/파일명 제어

```yaml
volumes:
- name: cfg
  configMap:
    name: my-config
    items:
    - key: app.conf
      path: app/app.conf
```

### 서브패스(subPath) 마운트(단일 파일만)

```yaml
volumeMounts:
- name: cfg
  mountPath: /etc/app/app.conf
  subPath: app.conf
```

> subPath는 **갱신 이벤트 반영이 제한**될 수 있어, 리로드 목적에는 **디렉터리 마운트**가 더 낫다.

---

## 동적 갱신 vs 롤아웃 트리거

- **파일 마운트**: kubelet이 주기적으로 투영 볼륨을 **갱신**. 애플리케이션이 SIGHUP/파일 감시로 **핫 리로드** 가능하면 **무중단**.
- **환경변수**: 컨테이너 시작 시 주입 → **재시작해야 변경 반영**.

### 변경 시 롤아웃 자동 트리거(체크섬 애노테이션 패턴)

Helm/Kustomize에서 ConfigMap/Secret 내용을 해시하여 Pod 템플릿 애노테이션에 주입 → 변경 시 자동 롤링.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  template:
    metadata:
      annotations:
        checksum/config: "e3b0c44298fc1c149afbf4c8996fb924..."
    spec:
      containers:
      - name: web
        image: nginx
        volumeMounts:
        - name: cfg
          mountPath: /etc/config
      volumes:
      - name: cfg
        configMap:
          name: my-config
```

> **핫 리로드 가능한 앱**이면 굳이 롤아웃을 유발하지 않고 파일 갱신만으로 처리하는 전략도 좋다.

---

## 실전 예제 — 하나의 앱에 ConfigMap+Secret 동시 주입

### 리소스

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_ENV: "prod"
  LOG_LEVEL: "info"
  app.toml: |
    [db]
    host = "db.prod.svc.cluster.local"
    port = 5432
---
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
stringData:
  DB_USER: "svc_user"
  DB_PASSWORD: "S3cureP@ss"
```

### Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  replicas: 2
  selector:
    matchLabels: { app: app }
  template:
    metadata:
      labels: { app: app }
    spec:
      containers:
      - name: app
        image: ghcr.io/example/app:1.0.0
        envFrom:
        - configMapRef: { name: app-config }
        - secretRef:    { name: app-secret }
        env:
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: app.toml    # 보통 파일, 예시는 의도적
        volumeMounts:
        - name: cfg
          mountPath: /etc/app
        - name: creds
          mountPath: /etc/creds
          readOnly: true
        ports: [{ containerPort: 8080 }]
      volumes:
      - name: cfg
        configMap:
          name: app-config
          items:
          - key: app.toml
            path: app.toml
      - name: creds
        secret:
          secretName: app-secret
          defaultMode: 0400
```

---

## 생성/검증/유틸 명령

```bash
# ConfigMap/Secret 생성(리터럴)

kubectl create configmap my-config --from-literal=APP_ENV=dev
kubectl create secret generic my-secret --from-literal=DB_USER=admin --from-literal=DB_PASSWORD=password

# 디렉터리/파일로 생성

kubectl create configmap web-cm --from-file=./conf/
kubectl create secret generic tls-cert --type=kubernetes.io/tls \
  --from-file=tls.crt=./server.crt --from-file=tls.key=./server.key

# 조회/내용 확인

kubectl get cm,secret
kubectl describe cm my-config
kubectl get secret my-secret -o yaml           # 주의: base64 값 노출
kubectl get cm web-cm -o jsonpath='{.data.app\.conf}'

# 디코딩(로컬)

kubectl get secret my-secret -o jsonpath='{.data.DB_USER}' | base64 -d; echo
```

---

## 보안/거버넌스 베스트 프랙티스

- **RBAC 최소권한**: `get/list/watch` 권한을 좁힌 역할(Role)과 RoleBinding 사용.
- **etcd 암호화(at-rest)**: 클러스터 단에서 **EncryptionConfiguration** 활성화.
- **이미지 풀 시크릿**: `kubernetes.io/dockerconfigjson` 타입 활용, 네임스페이스 기본 이미지 풀 시크릿 설정.
- **노출 경로 최소화**: 가능하면 **파일 마운트**로 주입(환경변수는 프로세스 리스트/덤프/로그로 새는 경우가 많다).
- **감사/추적**: Secret 접근은 감사 로그/모니터링.
- **서드파티 도구**:
  - **SOPS**(+KMS/GPG)로 깃에 암호화된 Secret 저장 → CI에서 복호화 적용
  - **Sealed Secrets**(컨트롤러로 클러스터에서만 복호화)
  - **External Secrets Operator(ESO)**: AWS Secrets Manager/Azure Key Vault/GCP Secret Manager에서 **자동 동기화**
- **`immutable: true`**: 변하지 않는 설정은 불변으로 고정해 성능/안정성↑.
- **PodSecurity/PSA**: 네임스페이스 수준 보안 레벨(Baseline/Restricted)로 비밀 사용 Pod에 보안 기준 준수.

---

## Kustomize/Helm 실전 패턴

### Kustomize generator

```yaml
# kustomization.yaml

configMapGenerator:
- name: app-config
  literals:
  - APP_ENV=staging
  files:
  - app.toml=conf/app.stg.toml

secretGenerator:
- name: app-secret
  literals:
  - DB_USER=svc_user
  - DB_PASSWORD=S3cureP@ss
generatorOptions:
  disableNameSuffixHash: false   # 해시 붙여 롤아웃 자동 트리거
```

### Helm — 체크섬 애노테이션

{% raw %}
```yaml
metadata:
  annotations:
    checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
```
{% endraw %}
> ConfigMap/Secret 변경 시 Pod 템플릿 애노테이션 값이 달라져 **자동 롤링**.

---

## 트러블슈팅 테이블

| 증상 | 1차 점검 | 원인 후보 | 해결 |
|---|---|---|---|
| Pod에서 키가 안 보임 | `kubectl describe pod` 볼륨/ENV 섹션 | 이름/키 오타, 네임스페이스 상이 | key/name/NS 정합성 수정 |
| 파일이 갱신 안 됨 | 투영 볼륨 여부 | 환경변수로 주입함, subPath 사용 | 파일 마운트로 전환, subPath 대신 디렉터리 마운트 |
| 권한 오류 | 마운트 퍼미션 | 디폴트 0644/0444 | `defaultMode`/UID/GID 조정 |
| Secret 노출 위험 | RBAC/감사 로그 | 과도 권한, `kubectl get secret` 남용 | Role 최소화, 자동화 계정 제한/비사용 |
| 이미지 풀 실패 | events `ImagePullBackOff` | 레지스트리 인증 없이 프라이빗 이미지 | 이미지 풀 시크릿 생성+참조 |
| at-rest 암호화 미적용 | 클러스터 설정 | etcd 평문 저장 | EncryptionConfiguration 적용 |

---

## 요약·선택 가이드

- **ConfigMap**: 비민감 설정(포맷/플래그/엔드포인트). 파일 마운트로 **핫 리로드**를 노려라.
- **Secret**: 민감정보. **RBAC 최소권한 + etcd 암호화 + 감사**는 필수. 가능하면 **파일 마운트** 사용.
- **변경 전파 전략**:
  - 핫 리로드 가능한 애플리케이션 → **파일 마운트 + 갱신 감지**
  - 재시작 필요/동작 보장 우선 → **체크섬 애노테이션으로 롤아웃 트리거**

> 실무 핵심: “**무엇을 Secret에 둘지**”를 엄격히 구분하고, “**어떻게 롤아웃을 유발/억제할지**”를 패턴으로 정착시키자. Git 관리에는 SOPS/SealedSecrets/ESO 같은 **운영 친화 도구**를 활용하면 안전성과 DX가 함께 올라간다.
