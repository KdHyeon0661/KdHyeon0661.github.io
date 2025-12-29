---
layout: post
title: Kubernetes - 블로그 플랫폼을 쿠버네티스로 배포하기
date: 2025-06-14 19:20:23 +0900
category: Kubernetes
---
# Kubernetes를 활용한 블로그 플랫폼 배포 실습

## 환경 준비

```bash
minikube start --driver=docker
minikube addons enable ingress
```

**스토리지 클래스 확인** (선택사항):
```bash
kubectl get storageclass
```

> 고성능이 필요한 경우 SSD 계열 StorageClass(예: `standard-rwo`, `gp3`, `premium`)를 지정할 수 있습니다.

---

## 아키텍처 설계

```
[클라이언트] → [Ingress(nginx)] → [Service/ghost] → [Pod/ghost]
                                      │
                                      └→ [Service/mysql] → [Pod/mysql] ↔ [PVC/mysql-pvc]

[PVC/ghost-content]  ←  /var/lib/ghost/content  (이미지/테마 저장용)
```

이 아키텍처는 Ghost 블로그 플랫폼과 MySQL 데이터베이스를 Kubernetes에서 운영하는 기본 구조를 보여줍니다.

---

## MySQL 구성: 상태 유지와 보안

### MySQL 매니페스트 파일 (mysql-deployment.yaml)

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
spec:
  accessModes: [ "ReadWriteOnce" ]
  resources:
    requests:
      storage: 5Gi
  # storageClassName: "standard"   # 필요 시 명시적 지정
---
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
type: Opaque
data:
  password: bXlzcWw= # 'mysql' (base64 인코딩)
  user: bXl1c2Vy     # 'myuser'
  database: Z2hvc3Q= # 'ghost'
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  replicas: 1
  selector:
    matchLabels: { app: mysql }
  template:
    metadata:
      labels: { app: mysql }
    spec:
      containers:
        - name: mysql
          image: mysql:8.0
          ports:
            - containerPort: 3306
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom: { secretKeyRef: { name: mysql-secret, key: password } }
            - name: MYSQL_DATABASE
              valueFrom: { secretKeyRef: { name: mysql-secret, key: database } }
            - name: MYSQL_USER
              valueFrom: { secretKeyRef: { name: mysql-secret, key: user } }
            - name: MYSQL_PASSWORD
              valueFrom: { secretKeyRef: { name: mysql-secret, key: password } }
          args:
            - --character-set-server=utf8mb4
            - --collation-server=utf8mb4_0900_ai_ci
            - --default-authentication-plugin=mysql_native_password
          volumeMounts:
            - name: mysql-storage
              mountPath: /var/lib/mysql
          readinessProbe:
            exec:
              command: ["bash","-lc","mysqladmin ping -h 127.0.0.1 -u$MYSQL_USER -p$MYSQL_PASSWORD || exit 1"]
            initialDelaySeconds: 10
            periodSeconds: 5
          livenessProbe:
            tcpSocket: { port: 3306 }
            initialDelaySeconds: 30
            periodSeconds: 10
          resources:
            requests: { cpu: "250m", memory: "512Mi" }
            limits: { cpu: "1", memory: "1Gi" }
      volumes:
        - name: mysql-storage
          persistentVolumeClaim:
            claimName: mysql-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  selector: { app: mysql }
  ports:
    - port: 3306
      targetPort: 3306
```

**적용 및 상태 확인**:
```bash
kubectl apply -f mysql-deployment.yaml
kubectl rollout status deploy/mysql
```

> **MySQL 버전 선택**: MySQL 8.0을 권장하며, 보안과 성능 면에서 우수합니다. 사전에 로캘과 인코딩을 지정하여 문자열 검색 및 이모지 지원 문제를 예방할 수 있습니다.

---

## Ghost 블로그 플랫폼 구성

Ghost는 `/var/lib/ghost/content` 디렉토리에 이미지, 테마, 설정 파일 등을 저장합니다. PVC를 통해 이 데이터를 영구적으로 보존할 수 있습니다.

### Ghost 설정 파일 (ghost-config.yaml)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ghost-config
data:
  url: "http://blog.local"  # Ingress 호스트와 일치해야 함
  mail__transport: "Direct" # 필요 시 SMTP로 변경
  # SMTP 설정 예시:
  # mail__transport: "SMTP"
  # mail__options__service: "Mailgun"
  # mail__options__auth__user: "postmaster@domain"
  # mail__options__auth__pass: "********"
---
apiVersion: v1
kind: Secret
metadata:
  name: ghost-secret
type: Opaque
data:
  db_user: bXl1c2Vy     # 'myuser'
  db_password: bXlzcWw= # 'mysql'
  db_name: Z2hvc3Q=     # 'ghost'
```

**적용**:
```bash
kubectl apply -f ghost-config.yaml
```

### Ghost 배포 매니페스트 (ghost-deployment.yaml)

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ghost-content-pvc
spec:
  accessModes: [ "ReadWriteOnce" ]
  resources:
    requests:
      storage: 5Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ghost
spec:
  replicas: 1
  selector:
    matchLabels: { app: ghost }
  template:
    metadata:
      labels: { app: ghost }
    spec:
      containers:
        - name: ghost
          image: ghost:5
          ports:
            - containerPort: 2368
          env:
            - name: url
              valueFrom: { configMapKeyRef: { name: ghost-config, key: url } }
            - name: database__client
              value: mysql
            - name: database__connection__host
              value: mysql
            - name: database__connection__user
              valueFrom: { secretKeyRef: { name: ghost-secret, key: db_user } }
            - name: database__connection__password
              valueFrom: { secretKeyRef: { name: ghost-secret, key: db_password } }
            - name: database__connection__database
              valueFrom: { secretKeyRef: { name: ghost-secret, key: db_name } }
          volumeMounts:
            - name: ghost-content
              mountPath: /var/lib/ghost/content
          readinessProbe:
            httpGet:
              path: /
              port: 2368
            initialDelaySeconds: 15
            periodSeconds: 5
            timeoutSeconds: 2
          livenessProbe:
            httpGet:
              path: /
              port: 2368
            initialDelaySeconds: 60
            periodSeconds: 10
            timeoutSeconds: 2
          resources:
            requests: { cpu: "200m", memory: "256Mi" }
            limits: { cpu: "1", memory: "1Gi" }
      volumes:
        - name: ghost-content
          persistentVolumeClaim:
            claimName: ghost-content-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: ghost-service
spec:
  selector: { app: ghost }
  ports:
    - port: 80
      targetPort: 2368
      protocol: TCP
```

**적용 및 상태 확인**:
```bash
kubectl apply -f ghost-deployment.yaml
kubectl rollout status deploy/ghost
```

> **URL 설정 중요성**: `url` 환경변수는 Ghost의 Canonical URL로 작동합니다. 이 값은 Ingress 호스트와 정확히 일치해야 이미지, 링크, 세션 관리에 문제가 발생하지 않습니다.

---

## Ingress 구성: 호스트 라우팅 및 보안 헤더

### Ingress 매니페스트 (ghost-ingress.yaml)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ghost-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/proxy-body-size: "20m"
    nginx.ingress.kubernetes.io/server-snippet: |
      add_header X-Frame-Options SAMEORIGIN always;
      add_header X-Content-Type-Options nosniff always;
      add_header Referrer-Policy strict-origin-when-cross-origin always;
spec:
  ingressClassName: nginx
  rules:
    - host: blog.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: ghost-service
                port:
                  number: 80
```

**적용**:
```bash
kubectl apply -f ghost-ingress.yaml
```

**호스트 파일 등록**:
시스템의 호스트 파일(`/etc/hosts` 또는 `C:\Windows\System32\drivers\etc\hosts`)에 다음을 추가합니다:
```
127.0.0.1 blog.local
```

이제 브라우저에서 `http://blog.local`에 접속할 수 있습니다.

---

## 운영 확장 및 고급 구성

### 초기 관리자 설정 및 데이터 관리

- **초기 관리자 계정**: Ghost는 첫 접속 시 관리자 비밀번호 생성 페이지를 표시합니다. 동시성 문제를 방지하기 위해 접근 제어(임시 Basic Auth) 또는 유지보수 시간대를 설정하는 것을 권장합니다.
- **이미지 업로드 관리**: 이미지는 `/var/lib/ghost/content/images`에 저장됩니다. PVC 크기와 백업 주기를 계획하는 것이 중요합니다.
- **테마 배포**: 테마는 `content/themes` 디렉토리에 저장됩니다. CI/CD 파이프라인에서 빌드 후 동기화하거나, InitContainer를 통해 테마 아카이브를 주입할 수 있습니다.

### 테마 배포를 위한 InitContainer 예시

```yaml
spec:
  template:
    spec:
      initContainers:
        - name: theme-init
          image: busybox:1.36
          command: ["sh","-c"]
          args:
            - |
              mkdir -p /var/lib/ghost/content/themes/mytheme
              # ConfigMap이나 emptyDir로 제공된 테마 아카이브 추출
              # tar -xf /extras/mytheme.tar -C /var/lib/ghost/content/themes/mytheme
          volumeMounts:
            - name: ghost-content
              mountPath: /var/lib/ghost/content
      containers:
        - name: ghost
          # 기존 컨테이너 설정
```

---

## 백업 및 복구 전략

### MySQL 데이터베이스 백업 (CronJob 활용)

**mysql-backup-cronjob.yaml**:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-backup-secret
type: Opaque
data:
  password: bXlzcWw=
  user: bXl1c2Vy
  database: Z2hvc3Q=
---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: mysql-backup
spec:
  schedule: "0 3 * * *"  # 매일 오전 3시 실행
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: dump
              image: mysql:8.0
              env:
                - name: MYSQL_USER
                  valueFrom: { secretKeyRef: { name: mysql-backup-secret, key: user } }
                - name: MYSQL_PASSWORD
                  valueFrom: { secretKeyRef: { name: mysql-backup-secret, key: password } }
                - name: MYSQL_DATABASE
                  valueFrom: { secretKeyRef: { name: mysql-backup-secret, key: database } }
              command: ["bash","-lc"]
              args:
                - |
                  set -e
                  TS=$(date +%F-%H%M%S)
                  mysqldump -h mysql -u$MYSQL_USER -p$MYSQL_PASSWORD $MYSQL_DATABASE > /backup/ghost-$TS.sql
                  ls -lh /backup
              volumeMounts:
                - name: backup
                  mountPath: /backup
          volumes:
            - name: backup
              persistentVolumeClaim:
                claimName: mysql-pvc
```

**적용 및 모니터링**:
```bash
kubectl apply -f mysql-backup-cronjob.yaml
kubectl get cronjob
```

**데이터베이스 복구**:
```bash
kubectl exec -it <mysql-pod> -- mysql -u<user> -p<password> ghost < backup-file.sql
```

### Ghost 콘텐츠 백업 방법

1. **PVC 스냅샷**: 스토리지 클래스가 지원하는 경우
2. **kubectl cp 명령어**: `/var/lib/ghost/content` 디렉토리 복사
3. **외부 오브젝트 스토리지 동기화**: rclone 사이드카 컨테이너를 활용한 S3 동기화

---

## 보안 강화

### TLS 인증서 자동 관리 (cert-manager + Let's Encrypt)

**ClusterIssuer 구성**:
```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-http01
spec:
  acme:
    email: you@example.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef: { name: acme-account-key }
    solvers:
      - http01:
          ingress:
            class: nginx
```

**Ingress TLS 설정 추가**:
```yaml
metadata:
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-http01
spec:
  tls:
    - hosts: [ "blog.your-domain.com" ] # 실제 도메인 필요
      secretName: ghost-tls
```

> **도메인 주의**: Minikube의 `blog.local`은 공용 인증서를 발급받을 수 없습니다. 실제 도메인을 사용하거나 자체 CA 또는 mkcert를 활용하세요.

### 관리자 접근 보호

- **기본 인증**: NGINX Ingress의 Basic Auth 기능 활용
- **OIDC 연동**: Keycloak, oauth2-proxy와 같은 리버스 프록시를 통한 `/ghost` 경로 보호
- **보안 헤더**: HSTS, CSP(Content Security Policy) 헤더 추가

---

## 확장성 및 고가용성

### 수평 파드 오토스케일링 (HPA)

```bash
kubectl autoscale deployment ghost --cpu-percent=60 --min=1 --max=5
kubectl get hpa
```

> **주의사항**: Ghost는 PVC를 통해 콘텐츠를 저장하지만, 기본적으로 ReadWriteOnce(RWO) 접근 모드를 사용합니다. 다중 레플리카 구성 시 NFS/RWX 스토리지 클래스 사용 또는 S3와 같은 외부 저장소 연동을 고려해야 합니다.

### Pod 중단 예산 (PDB)

**pdb-ghost.yaml**:
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: ghost-pdb
spec:
  minAvailable: 1
  selector:
    matchLabels: { app: ghost }
```

### 안티 어피니티 (다중 노드 환경)

```yaml
spec:
  template:
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 50
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: app
                      operator: In
                      values: ["ghost"]
                topologyKey: "kubernetes.io/hostname"
```

---

## 모니터링 및 관측성

- **로그 확인**: `kubectl logs deploy/ghost -f`
- **리소스 사용량**: `kubectl top pods`, `kubectl top nodes` (metrics-server 필요)
- **애플리케이션 메트릭**: Ghost 자체 Prometheus 메트릭은 제한적이므로 NGINX Ingress, 노드, MySQL(mysqld-exporter) 지표를 통해 간접적으로 모니터링
- **대시보드**: Lens, K9s 도구 활용 또는 Prometheus/Grafana 스택 추가 설치

---

## GitOps 및 패키징

### Kustomize를 활용한 환경별 구성

**디렉토리 구조**:
```
k8s/
  base/
    mysql-deployment.yaml
    ghost-config.yaml
    ghost-deployment.yaml
    ghost-ingress.yaml
  overlays/prod/...
  overlays/dev/...
```

### Helm 차트 패키징

`values.yaml`에 다음과 같은 파라미터를 정의할 수 있습니다:
* `ghost.url`: 블로그 URL
* `mysql.password`: 데이터베이스 비밀번호
* `resources`: 리소스 요청 및 제한
* `persistence.size`: 저장소 크기

Argo CD 또는 Flux를 통해 Git 저장소와 클러스터를 자동으로 동기화할 수 있습니다.

---

## 문제 해결 가이드

| 증상 | 가능한 원인 | 해결 방안 |
|---|---|---|
| `Pending` 상태 | PVC 바인딩 실패 또는 스토리지 부족 | `kubectl get pvc` 실행, StorageClass 확인 |
| Ghost 502/504 오류 | `url` 설정 불일치 또는 Ingress 구성 오류 | `url` 값을 실제 접속 URL과 일치시킴, Ingress 규칙 점검 |
| 데이터베이스 연결 실패 | Secret 오타 또는 MySQL 준비 전 Ghost 시작 | readinessProbe 확인, initContainers를 통한 데이터베이스 연결 확인 |
| 로그인 루프 | `url`의 스킴 또는 도메인 불일치 | `url`을 실제 접속 URL로 수정 후 재시작 |
| 업로드 실패 | PVC 용량 부족 또는 권한 문제 | PVC 확장, 보안 컨텍스트 점검 |
| 응답 속도 저하 | 리소스 부족 또는 이미지 최적화 필요 | requests/limits 상향, 이미지 압축 및 캐시 최적화 |

### 데이터베이스 준비 대기 initContainers 예시

```yaml
initContainers:
  - name: wait-mysql
    image: busybox:1.36
    command: ["sh","-c"]
    args:
      - |
        until nc -zv mysql 3306; do echo "MySQL 대기 중..."; sleep 2; done
```

---

## 정리 및 삭제

```bash
# 리소스 삭제
kubectl delete -f ghost-ingress.yaml
kubectl delete -f ghost-deployment.yaml
kubectl delete -f ghost-config.yaml
kubectl delete -f mysql-deployment.yaml
kubectl delete -f mysql-backup-cronjob.yaml

# 추가 리소스 삭제 (존재하는 경우)
kubectl delete hpa ghost
kubectl delete pdb ghost-pdb
```

> **데이터 보존 주의**: PVC는 기본적으로 삭제되지 않습니다. 완전한 정리를 원하면 PVC와 PV도 명시적으로 삭제해야 합니다.

---

## 매니페스트 파일 요약

| 파일 | 설명 |
|---|---|
| `mysql-deployment.yaml` | PVC, Secret, Deployment, Service 포함 |
| `ghost-config.yaml` | ConfigMap 및 Secret (애플리케이션 설정) |
| `ghost-deployment.yaml` | Ghost PVC, Deployment, Service (프로브 및 리소스 포함) |
| `ghost-ingress.yaml` | Ingress 구성 (보안 헤더 및 리라이트 설정) |
| `mysql-backup-cronjob.yaml` | 데이터베이스 백업 스케줄 (선택사항) |
| `pdb-ghost.yaml` | Ghost 가용성 정책 (선택사항) |

---

## 결론

이 실습을 통해 Kubernetes 환경에서 Ghost 블로그 플랫폼을 안정적으로 배포하고 운영하는 방법을 체계적으로 학습했습니다. 주요 구현 내용을 요약하면 다음과 같습니다:

1. **상태 유지 아키텍처**: Ghost 콘텐츠와 MySQL 데이터를 PVC를 통해 영구적으로 저장하여 데이터 손실을 방지했습니다.

2. **안정성 강화**: readiness/liveness 프로브, initContainers, PodDisruptionBudget을 통해 서비스의 가용성과 신뢰성을 확보했습니다.

3. **보안 체계 구축**: Secret을 통한 민감 정보 관리, TLS 인증서 자동화, 보안 헤더 추가로 종합적인 보안 환경을 구성했습니다.

4. **확장성 설계**: 수평 파드 오토스케일링(HPA)과 안티 어피니티 규칙을 통해 트래픽 증가와 다중 노드 환경에 대비했습니다.

5. **운영 자동화**: CronJob을 통한 정기 백업, GitOps 접근법을 통한 배포 관리 자동화를 구현했습니다.

이 기본 구성은 프로덕션 수준의 블로그 서비스를 위한 견고한 기반을 제공합니다. 필요에 따라 CDN 통합, 오브젝트 스토리지(S3) 플러그인, 외부 이메일/SMS 인증, 이미지 프록시, 캐시 계층(Cloudflare/NGINX 캐시) 등을 추가로 구현하여 시스템을 더욱 강화할 수 있습니다.

이 실습을 통해 얻은 지식은 단순히 블로그 플랫폼에 국한되지 않고, Kubernetes에서 상태 유지 애플리케이션을 운영하는 일반적인 패턴과 모범 사례를 이해하는 데 도움이 될 것입니다.